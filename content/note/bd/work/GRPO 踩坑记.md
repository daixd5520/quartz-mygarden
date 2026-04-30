---
tags:
  - landing
  - GRPO
  - RLHF
  - llm应用
title: GRPO 踩坑记
draft: "true"
---

# GRPO 踩坑记

基于 Qwen3-4B 的 `struct_plan_v6` SFT 模型，在 swift `rlhf` 上跑 GRPO，外挂自研的 `plan_xml_reward_decoupled` reward plugin。训练本身不复杂，几个陷阱都踩在"日志看起来都对、但模型其实没在学"这种灰色地带上。这里记录两个代价比较大的坑。

## 坑一：Reward 恒为 -1，模型根本没进 judge

### 现象

![[GRPO 踩坑记-2.png]]

开跑没几个 step 就发现 reward 贴着 -1 不动，方差几乎为零。swanlab 训练监控里：

![[GRPO 踩坑记-3.png]]

![[GRPO 踩坑记-4.png]]

### 假设

Reward plugin 的设计是分层的：先跑一道 **hard gate** 做格式检验，不合法直接返回 -1；合法的样本才进入更贵的 **judge** 阶段给细粒度分数。

把监控串起来看，异常信号有两个：

1. `PlanReward` 耗时几乎贴地，只有偶尔一个尖峰——说明绝大部分样本在 hard gate 就被快速毙掉了，根本没进 judge。
2. `train/completions/max_length` 一直顶在 `512`，而这个 SFT 模型在正常推理下不会把输出拉到这么长。

两条信号指向同一个结论：**模型输出大多不合法**，格式检验直接判死。但 `dcd_llm` 直接请求同一份 checkpoint 是能正确输出的，所以问题不在模型本身，而在 swift GRPO 训练时喂给 vLLM 的 prompt 构造方式。

### 验证

在 reward plugin 里加一行控制台打印，把模型的实际输出拉出来看：

![[GRPO 踩坑记-1.png]]

确认模型的确没有按 schema 要求输出。配合 swift 的启动日志，基本锁定原因是数据格式——我喂进去的是裸 `prompt` 字段，没有走 messages 结构也没有带 system prompt：

```json
{"prompt": [{"role": "user", "content": "你是一个"选车语义解析模型"，。。。。\n\n========================\n【用户输入】\n========================\n雷克萨斯RX350h四驱版驾驶体验"}],"query": "雷克萨斯RX350h四驱版驾驶体验", "select_schema": "|xxxx|xx|","where_schema": "xx|xx", "task": "plan_xml"}
```

swift 对 GRPO 场景默认按 chat template 走，但只有当数据是标准 `messages` + system prompt 格式时，chat template 才会正确注入任务指令。裸 `prompt` 被当成单轮 user 消息喂进去，SFT 阶段训练的那一套 system prompt 完全没生效，模型自然退化成乱输出。

### 结论

- 数据换成 `messages` 格式，把任务指令放进 `system` role。
- 改完之后模型输出立刻回到 SFT 时的样子，reward 分布正常，judge 路径走得通。
- **教训：GRPO 对输入数据格式比 SFT 挑剔得多。**SFT 阶段 framework 会帮你兜底很多边界情况，GRPO 只要 rollout 阶段和训练阶段的 chat template 对不上，reward 就会整体塌陷，而且症状只是"reward 很低"——极具误导性。

遇到类似 reward 恒低的情况，排查顺序我现在固定成：

1. 先看 reward plugin 的阶段分布（hard gate vs judge），判断是"走到 judge 但被打低分"还是"根本没进 judge"。
2. 如果是后者，直接打印 rollout 出来的原始文本，和 SFT 推理时做 diff。
3. 再对照 framework 的 chat template 和数据格式，确认 system prompt 有没有丢。

### 附：启动命令与训练日志

```shell fold title:启动命令
export PLAN_REWARD_VERSION=v2
export PLAN_REWARD_PLUGIN_WORKERS=${MLP_WORKER_GPU}
export PLAN_REWARD_PLUGIN_LOG_EVERY=1
pip install swanlab

FORCE_TORCHRUN=1 \
NNODES=${MLP_WORKER_NUM} \
NODE_RANK=${MLP_ROLE_INDEX} \
NPROC_PER_NODE=${MLP_WORKER_GPU} \
MASTER_ADDR=${MLP_WORKER_0_PRIMARY_HOST} \
MASTER_PORT=29500 \
swift rlhf \
    --rlhf_type grpo \
    --model /dcar_ai_vepfs/yuhongjiang/models/qwen3-4b-struct_plan_v6/v2-20260419-173600-iter-2200-hf \
    --external_plugins /dcar_ai_vepfs/daixindi/script/plugins/plan_reward_decoupled/0424/PlanReward_plugin.py \
    --reward_funcs plan_xml_reward_decoupled \
    --dataset /dcar_ai_vepfs/daixindi/data/grpo/FinalPlan500_with_plan_prompt-1grpo.jsonl \
    --train_type lora \
    --torch_dtype bfloat16 \
    --max_completion_length 512 \
    --num_train_epochs 1 \
    --per_device_train_batch_size 2 \
    --per_device_eval_batch_size 2 \
    --learning_rate 1e-6 \
    --gradient_accumulation_steps 2 \
    --eval_steps 10 \
    --save_steps 10 \
    --logging_steps 5 \
    --save_total_limit 5 \
    --warmup_ratio 0.05 \
    --num_generations 4 \
    --steps_per_generation 4 \
    --temperature 1.6 \
    --max_length 8192 \
    --output_dir /dcar_ai_vepfs/daixindi/models/grpo_qwen3-4b-struct_plan_v6_plan_reward_decoupled \
    --dataloader_num_workers 1 \
    --dataset_num_proc 1 \
    --deepspeed zero3 \
    --report_to swanlab \
    --swanlab_token wihfZNZ9BnOuZWeePxDT3 \
    --swanlab_project grpo_qwen3-4b-struct_plan_v6-temp07 \
    --swanlab_exp_name grpo_qwen3-4b-struct_plan_v6
```

```log fold title:问题阶段的 reward 日志样例（全部 fatal_by_judge）
[2026-04-28 14:44:46] [plan_xml_reward_decoupled] call_counter=11 version=v2 cost_time=59.32s rewards={1.000:4, -0.800:4} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8
[2026-04-28 14:44:50] [plan_xml_reward_decoupled] call_counter=11 version=v2 cost_time=63.24s rewards={-0.800:7, 0.970:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8
[2026-04-28 14:44:59] [plan_xml_reward_decoupled] call_counter=11 version=v2 cost_time=71.69s rewards={-0.800:4, 1.000:4} stages={fatal_by_judge:4, scored:4} gate_reasons={} exceptions={} invalid_like=0/8
[2026-04-28 14:45:00] [plan_xml_reward_decoupled] call_counter=11 version=v2 cost_time=72.67s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8
```

### 修复后的观测

改成带 system prompt 的 messages 格式后，输出立刻正常，日志里可以看到 reward 分布拉开：

![[GRPO 踩坑记-5.png]]

当前运行参数：温度 1.2，`max_completion_length` 512，2 worker × 8 卡/机，每张卡显存 3–4 GB。

![[GRPO 踩坑记-6.png]]

## 坑二：Reward -0.8 占比过高，judge 判罚过重

### 现象

修完第一个坑之后，reward 分布虽然不再塌到 -1，但 -0.8 这一档仍然占了很大比例。结合 reward plugin 的阶段分布，多数是在 judge 阶段命中了 fatal 判定——也就是说 judge 把很多本应给连续分的样本直接打成了硬错。

### 假设

Fatal 的判定条件在 judge prompt 里写得比较激进，倾向于"宁可杀错不可放过"。在 RL 训练早期模型波动大，这会让绝大多数探索样本被一刀切，梯度信号几乎全部来自 fatal，连续 reward 的塑形信号被淹没。

### 待验证

- 松掉 judge prompt 里的 fatal 判定，让它只在真正不可修复的结构错误上触发。
- 把边界情况从 fatal 换成连续 reward，让模型能从"接近正确"的样本上拿到梯度。
- 观察 reward 方差和 fatal 占比的变化。

这一条还在复现和验证中，后面补结果。
