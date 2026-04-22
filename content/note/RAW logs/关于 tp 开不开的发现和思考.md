![[关于 tp 开不开-1.png]]

事情是这样的：我以前一直以为 一张卡放得下的话，tp1 肯定比 tp>1 快，因为卡间通讯需要时间

可是 A100, sglang 部署 1.7B 和 4B 的模型，QPS=1 的情况下都是 tp2 比较快

问了 gpt 发现通讯没打满的情况下，tp 还可能增大 tensor core 的利用率

这个现象本身是**正常的**，但前提是：你现在的瓶颈**不在通信，而在单卡算力/带宽利用率**。否则 tp=2 理论上只会更慢。

我直接把机制拆开讲清楚。

---

# 一、先把问题抽象成计算模型

4B 模型，单 query，并发=1，本质是：

> **单条自回归 decode：逐 token 推理**

每一步是：

[  
h_{t+1} = \text{Transformer}(h_t)  
]

关键在于：

> **每一步都是小 batch、低并行度计算**

---

# 二、tp=1 时发生了什么

单卡跑 4B：

- 参数全在一张卡
- 每层 GEMM 规模是固定的（比如 hidden 4096 × 4096）

问题在于：

### 👉 GPU 并没有被吃满

原因是：

### **1. GEMM 尺寸不够“理想”**

对于 GPU（尤其是 Tensor Core）来说：

- 最优是大矩阵（比如 batch 大）
- 但你现在是：
    - batch = 1
    - seq 很短（decode阶段）

👉 导致：

> **算子 launch overhead + memory latency 占比很高**

---

### **2. memory-bound 明显**

Transformer decode 很多操作是：

- KV cache 读取
- 小矩阵乘

这些是典型：

> **memory bandwidth bound，而不是 compute bound**

---

# 三、tp=2 发生了什么（关键）

tp=2 本质是：

> **把权重按 hidden 维切成两半，分布到两张卡**

每一步变成：

- 每张卡算一半 GEMM
- 然后 **all-reduce / all-gather**

---

## 表面看：多了通信，应该更慢

但你现在变快，说明一个事实：

> **通信开销 < 单卡未利用的算力浪费**

---

## 真正的加速来源是这几个点

### **1. 提高了 SM 利用率**

原来：

- 一个大 kernel 没吃满 SM

现在：

- 拆成两个更“适配硬件”的 kernel
- 两张卡各自跑

👉 等价于：

> **把低利用率 → 两个中等利用率**

---

### **2. L2 / HBM 带宽压力被分摊**

原来：

- KV cache / 权重 全压在一张卡
- bandwidth saturation

现在：

- 一半数据在卡1
- 一半在卡2

👉 实际效果：

> **memory bandwidth ×2（近似）**

这点在 decode 阶段非常关键。

---

### **3. kernel launch / pipeline overlap 更好**

现代推理框架（比如 vLLM / TensorRT-LLM）会：

- overlap compute 和通信
- stream pipeline

所以实际不是：

> compute → 通信 → compute

而是：

> **compute 和 communication overlap**

---

### **4. hidden size 切分后更“Tensor Core friendly”**

有些维度在 tp=1 时：

- 不能很好对齐 Tensor Core tile（比如 128/256）

tp=2 后：

- 子矩阵尺寸更规整

👉 提高了：

> **FLOPs utilization**

---

# 四、什么时候 tp=2 会变慢？

这个现象不是普适的，它成立有前提。

如果出现下面情况，tp=2 会输：

---

### **1. batch size 提高**

当：

- batch > 8 或 16

👉 单卡已经吃满：

> **compute-bound**

这时：

> 通信 = 纯负担

---

### **2. 模型更大（比如 13B+）**

这时：

- 单卡已经 compute heavy

tp=2：

- 通信比例变大

---

### **3. 跨机通信（不是 NVLink）**

如果是：

- PCIe
- 或者跨节点

👉 带宽骤降

tp=2 会直接掉速

---

# 五、你这个 case 的本质一句话

> **你现在在“单卡低利用率区间”，tp=2 帮你把计算从 memory-bound 拉向更均衡状态。**

---

# 六、一个更底层的理解（很关键）

把 latency 拆开：

[  
T = T_{compute} + T_{memory} + T_{launch}  
]

tp=1：

- T_memory 很大
- T_compute 没吃满

tp=2：

- T_memory ↓（分摊）
- T_compute ↑（利用率提升）
- T_comm ↑（但被 overlap）

最终：

[  
T_{total} ↓  
]

---

# 七、工程上你可以验证的点

如果你想确认我说的是对的，可以直接看 profile：

- **SM occupancy**
- **dram utilization**
- **nvlink utilization**
- **kernel time vs memcpy time**

你会看到：

👉 tp=1：

- dram 很高
- sm 不满

👉 tp=2：

- dram 降
- sm 升
- nvlink 有占用但没成为瓶颈

---

# 收束

这个现象不是“tp=2更强”，而是：

> **你的当前负载让单卡处在低效区，而 tp=2 恰好把它推回高效区。**

如果把 batch 拉起来，你会看到结论反转。

---

如果继续往下挖，这里其实有个更有意思的问题：

> **为什么 decode 阶段几乎永远是 memory-bound，而 prefill 更偏 compute-bound？**
