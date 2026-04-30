---
tags:
  - landing
  - inference
  - 推理优化
draft: "true"
---

![[关于 tp 开不开-1.png]]

一个和直觉反的发现：A100 + SGLang 部署 1.7B 和 4B 模型，QPS=1 的时候，tp=2 比 tp=1 更快。按过去的经验——只要一张卡塞得下，tp=1 一定比 tp>1 快，因为省了 all-reduce——这结论是反的。问 GPT 加上自己翻 profile，能拼出一个比较干净的解释：**瓶颈没到通信上，而在单卡算力/带宽利用率；tp=2 恰好把模型从 memory-bound 的低效区推到更均衡的工作点上**。下面把机制拆开写。

## 负载抽象：decode 阶段是"小 batch、低并行度"的重复小算子

QPS=1、单条自回归 decode，每一步本质是：

$$
h_{t+1} = \mathrm{Transformer}(h_t)
$$

每一步的 GEMM 规模是 hidden × hidden（4096 × 4096 级别），batch=1，序列在 decode 阶段实际参与 attention 计算的只有新 token。这种形状对 Tensor Core 很不友好，kernel launch overhead 和 HBM latency 在总时间里占比很高。

## tp=1：算力没吃满，带宽却已经紧张

单卡放 4B 参数，GEMM 形状固定但小，几乎没有并发空间。拿 profile 看：

- SM occupancy 不高，Tensor Core tile 对不齐，FLOPs utilization 低。
- 每步 decode 都要把权重从 HBM 拉一遍、再把 KV cache 拉一遍，实测 dram utilization 很高。

这是典型的 **memory-bound + kernel launch 重**场景：GPU 在等数据，不在等算。

## tp=2：看似多通信，实际是把瓶颈拆掉

tp=2 把权重沿 hidden 维切两半分到两张卡，每步加一次 all-reduce / all-gather。表面上是"多了一步通信会变慢"，实际能变快的原因：

1. **HBM 带宽近似翻倍**。权重和 KV cache 一半在卡 0、一半在卡 1，单卡带宽压力对半砍，原本被带宽卡住的 decode 能跑得更快。
2. **单卡 GEMM 变小，kernel 更 friendly**。切完之后 sub-matrix 的尺寸反而更容易对齐 Tensor Core tile（128/256），Tensor Core 利用率提升。
3. **compute 和 communication 能 overlap**。SGLang / vLLM / TensorRT-LLM 的 stream pipeline 会把下一步的 GEMM 和当前的 NVLink 通信并行起来，通信不是串行加到总延迟上。
4. **两个中等利用率的卡**比一个低利用率的卡总吞吐更高。

简单写成 latency 分解：

$$
T = T_{\text{compute}} + T_{\text{memory}} + T_{\text{launch}} + T_{\text{comm}}
$$

- tp=1：$T_{\text{memory}}$ 大，$T_{\text{compute}}$ 没吃满，$T_{\text{comm}}=0$。
- tp=2：$T_{\text{memory}} \downarrow$，$T_{\text{compute}} \uparrow$（利用率提升），$T_{\text{comm}} \uparrow$ 但被 overlap 吃掉。

两张卡的 NVLink 带宽在 A100 上是很充足的（600 GB/s 级别），all-reduce 单次的 payload 对 4B 模型每层只有几个 MB，所以 `T_comm` 很难成为瓶颈。

## 什么时候 tp=2 反而输

这个结论不是普适的，以下三种情况会反转：

1. **batch 拉大**。一旦 batch ≥ 8 或 16，单卡已经到 compute-bound 区间，通信就是纯开销。
2. **模型更大**。13B+ 的模型在单卡已经 compute heavy，tp=2 只能帮你解决显存装不下的问题，不会再快。
3. **跨 PCIe 或跨机通信**。NVLink 不在了，带宽骤降，tp 引入的 all-reduce 直接变成主要延迟。

## 想确认上面这套说法，直接看 profile

Nsight Systems / DCGM 里盯四个指标：

- SM occupancy
- DRAM utilization
- NVLink utilization
- kernel time vs memcpy time

tp=1 大概率是 DRAM 很高、SM 不满；tp=2 会看到 DRAM 降、SM 升、NVLink 有占用但不是瓶颈。这一组指标对上了，就可以相信"带宽分摊 + Tensor Core friendly + overlap"这个解释。

## 一句话收束

tp 开不开不是"通信是否昂贵"的问题，而是"你当前的负载处在哪条瓶颈曲线上"。decode 阶段、小 batch 这一档，tp=2 是在用一点通信换一大段 HBM 带宽和 SM 利用率；把 batch 拉起来或者换更大的模型，结论会立刻反转。

还能再往下挖的一个问题是：**为什么 decode 阶段几乎永远是 memory-bound，而 prefill 更偏 compute-bound？** 这个留到下一篇。
