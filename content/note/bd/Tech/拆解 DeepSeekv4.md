
> [!abstract]+
> DeepSeek-V4-Pro（总参数 **1.6T**，激活 **49B**）与 DeepSeek-V4-Flash（总参数 **284B**，激活 **13B**），二者原生支持 **1M** token 上下文。相较 DeepSeek-V3.2，DeepSeek-V4-Pro 在 1M context 下 单 token 推理 FLOPs 只需 27%、KV cache 只需 10%；V4-Flash 更激进，只需 10% FLOPs 和 7% KV cache。在公开基准上，最大推理模式 DeepSeek-V4-Pro-Max 在开源模型中全面登顶 SimpleQA-Verified（57.9 vs. Kimi-K2.6 的 36.9），Codeforces Rating 达到 3206，与 GPT-5.4-xHigh 基本持平，在 CodeForces 人类选手榜上排名第 23 位----**长上下文基础设施重构**

> [!hint]- 技术速览
> ### Infra
> - Agent 训练依赖可执行轨迹：**DSec** 沙箱支持 Function Call / Container / microVM / fullVM，并记录全序 trajectory log；agent 数据的关键是**可执行、可评分、可复现**。
>  - 大规模 RL 需要**可恢复 rollout**：Token 级 WAL、preemptible rollout service、deterministic kernels 共同解决**抢占、重跑**和 batch 变化带来的训练偏差。
>  - 长上下文瓶颈在 KV cache 管理：分层 KV cache 和 on-disk SWA 策略说明，长上下文部署不只是 attention 算法问题，还包括 **cache layout、prefix reuse 和外存管理**。
>  - **工具调用链路**也要工程化：DSML XML tool-call schema 降低 JSON escaping 错误；Quick Instruction 复用 KV cache 执行搜索判断、query 生成等前置任务，降低 TTFT。
> 
> ### 算法
> - **CSA + HCA** 是长上下文注意力折中方案：CSA 负责压缩后 top-k 稀疏检索，HCA 负责重压缩后的全局 dense 记忆，**SWA 补局部细节**。
>  - mHC 用约束残差提升**深层**稳定性：将残差变换矩阵约束到 doubly stochastic 流形，通过 Sinkhorn-Knopp 投影控制谱范数。
>  - Muon 仍需稳定性技巧配合：**Muon 是主优化器**，但 trillion-scale MoE 仍依赖 Anticipatory Routing 和 **SwiGLU** Clamping 控制 loss spike。
>  - Specialist + **OPD** 替代 mixed RL
> - Actor-as-GRM 面向**难验证**任务：用 rubric-guided data 和 Generative Reward Model 替代传统 scalar reward model，适合开放式写作、办公和 agent 任务。

# RL层面

后训练分两个阶段

- 分别训各领域专家
	- SFT
	- 在对应任务上做 GRPO
- 多专家 OPD（On Policy Distillation）
> 和 V3.2 的区别：用 OPD 代替了 Mixed RL
> 先分别训练 **math/code/agent/InstructionFollowing(IF)** 专家，再用 full-vocabulary OPD 蒸馏回统一 student，降低**多能力混训**干扰。

# 架构层面

- 残差连接：`Residual` → **Manifold-Constrained Hyper-Connections (mHC)** 
	- 把HC的残差矩阵 $B_l$ 约束到双重随机矩阵流形（Birkhoff polytope），相比原版 ResidualConnection、HC，解决了表达能力受限、数值不稳定的问题。
- 注意力层：`MLA` → **CSA（Compressed Sparse Attention）+ HCA（Heavily Compressed Attention）混合注意力**
	- 交错分布（除了前两层连续HCA），CSA **压缩率设置为 4** 并且使用 Lightning Indexer 做top-K 稀疏KV 选择。HCA设置**压缩率128**、不做KV稀疏直接DenseMQA。这是DSV4实现1M上下文下，低FLOPs/KV cache的<font color="#f79646">核心技术</font>。
- 优化器：`AdamW` → **Muon**（仅部分模块仍用 AdamW）
![[拆解 DeepSeekv4-1.png|433]]
