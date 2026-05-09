# 强化学习方法

### 概述

见 [link](https://github.com/wdndev/llm_interview_note/blob/main/07.%E5%BC%BA%E5%8C%96%E5%AD%A6%E4%B9%A0/%E5%A4%A7%E6%A8%A1%E5%9E%8BRLHF%EF%BC%9APPO%E5%8E%9F%E7%90%86%E4%B8%8E%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/%E5%A4%A7%E6%A8%A1%E5%9E%8BRLHF%EF%BC%9APPO%E5%8E%9F%E7%90%86%E4%B8%8E%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB.md)

![382](https://dcar.feishu.cn/space/api/box/stream/download/asynccode/?code=NGEyYmM3ZjIxMTI4ZjNlYWYwZjMwNzE1NGQ5ZWVkZmJfRU9IYW9BbFYya0xLTkZYakJxcnlYVU9zNUlrcGU0NkRfVG9rZW46SEZHa2JXNjE1b0tLT0t4Z0FtMGN5QkJibkpiXzE3NzgzMTUxNDc6MTc3ODMxODc0N19WNA)

强化学习的两个实体：智能体（Agent）与环境（Environment）

强化学习中两个实体的交互：

- **状态空间****S**：S即为State，指环境中所有可能状态的集合
- **动作空间A**：A即为Action，指智能体所有可能动作的集合
- **奖励R**： R即为Reward，指智能体在环境的某一状态下所获得的奖励。

#### Loss的直观设计

  $$SFT\_Loss=-logP(A_t|S_t)$$

  $$RLHF\_Loss=-V_t * logP(A_t|S_t)$$

$$(A_t|S_t)$$

表示基于当前状态（前缀Tokens）输出下一个Token（A_t）的prob，V_t代表奖励

- <font color="#f79646">当 Vt>0 时，</font>意味着Critic对Actor当前采取的动作给了正向反馈，因此就需要在训练迭代中提高 $ P(A_{t} | S_{t}) $，这样就能达到减小loss的作用。
- **当 Vt<0 时**，意味着Critic对Actor当前采取的动作给了负向反馈，因此就需要在训练迭代中降低 P(At|St) ，这样就能到达到减小loss的作用。

#### 引入优势（Advantage）

对NLP任务来说，如果Critic对 At 的**总收益预测**为 Vt ，但**实际执行** At 后的总收益是 Rt+γ∗Vt+1 ，我们就定义优势为：

$Adv_t = R_t+\gamma * V_{t+1} - V_t$

Adv衡量的是我们执行At后，也就是生成新的token之后的收益值

$$RLHF\_Loss=-Adv_t * logP(A_t|S_t)$$

### PPO

![](https://dcar.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTllMjhkN2YxODU5ZTQwNDk1ZTM2MDVmOGMzZjY2OTlfTjVQbjNKSmlmenJxV2FkbERYQ3pXT3hERWdDazBYMGxfVG9rZW46RE5IeGJBb0pwb1gwWGt4MThCY2NzMnQxbm1jXzE3NzgzMTUxNDc6MTc3ODMxODc0N19WNA)

### REINFORCE Leave-One-Out

- **留一法****（****Leave-One-Out****）** 构造基线，即第 个样本的基线为除自己外的其他 个样本的均值

![[Tech--强化学习-1.png]]

### **KL-in-loss → KL-in-reward：同一个 log‑prob 的梯度同时对 task reward 和 KL 做平衡，而不是拆成两部分 loss 再相加。

![[Tech--强化学习-2.png|304]]

### REINFORCE++

- **Global Advantage Normalization（GAN）:** 分母用Batch STD Advantage，**adv 分布更稳定，训练曲线更平滑**

![[Tech--强化学习-3.png|556]]

- 与DAPO相同的token-level loss，与RLOO相同的KL-in-reward 等，全局归一化更好，scale震荡的鲁棒性更好

Reinfoce++存在的缺点：会引入偏置，不同的prompt可能天然reward就会更高一些，reward更高的prompt任务会主导更新，所以Rinforce++更适合相似、相同评判标准的任务，不适合跨域、多任务混合训练。

### GSPO

#### GSPO对MOE模型的改进点，MOE的强化无脑GSPO

- MOE模型GRPO痛点：同一条 rollout 样本，在做了一次或几次梯度更新后，新旧策略下激活的专家集合会明显变化；GRPO 用的是 **token-level importance ratio** 
![[Tech--强化学习-4.png|262]]
但在 MoE 里， 

$π_θ(⋅)$ 的数值不仅取决于输出 token，还强依赖当时路由到的专家子网络，当路由变化时，分子分母实际上“走的不是同一套子网络”，就会导致 $w_{i,t}$ **剧烈波动、进一步失效**，训练难以正常收敛。

- 改进

GSPO（**Group Sequence Policy Optimization**）对 MoE（Mixture-of-Experts）模型最“特别”的优化点，核心不是改路由器结构本身，而是**把** **RL** **的 off-policy 校正与裁剪（****clipping****）从 token 级切换到序列级（sequence-level）**，从而**天然规避 MoE 的“路由/专家激活波动”对训练稳定性的破坏**，并**彻底消除对 Routing Replay 这类 MoE 专用补丁的依赖**。

#### Sequence-Level

![[Tech--强化学习-5.png]]

GRPO公式序列化得到GSPO的公式，Adv本质上跟token无关，仅仅是将w的token-level weight聚合成sequence-level的s，而s本质也是通过token-level的分布差聚合而成。

Sequence-level 的IS weights相对于token-level更加平滑，避免了极端token的影响（相比于token-level的clip，sequence-level clip明显更易充分学习），稳定性更好。

#### Token-Level

这个方法结合了sequence_level和token_level，融合了两者的重要性

 ![[Tech--强化学习-6.png]]

#### 为什么GSPO更加平滑

原理类似于 $$1/N *\sum_{i=1}^{N} X_i$$

的方差为 $$D(X)/N$$

，求和之后方差更小，自然更平滑

# 强化学习问题引入及解决方案

## 🐞 Reward Hacking解决方案

1. 从reward本身下手，优化reward设计，或者使用multi_reward
2. 过程信号，矫正过程错误。我们的answer应该算是只有过程？没有明确的答案
3. 可疑高分样本采样标注
4. 守门人LLM judge（safeguard / critic）
5. KL散度加权

# 📊 RLHF指标

### pg_loss（policy gradient loss）

“策略更新的主要项”。在 PPO/GRPO 里常形如：

![[Tech--强化学习-7.png|441]]

**pg_loss趋势与Adv相似，由于Adv是归一化、中心化的，所以pg_loss与Adv同样，是在某个值（0）附近波动；GRPO的更新目标在于** $$\pi_{\theta}/\pi_{\theta_{old}}$$

，所以loss恒定不影响更新

![[Tech--强化学习-8.png]]

### pg_clipfrac（policy gradient clip fraction）

统计有多少比例的样本 **触发了** **clipping**

**怎么看**

- **太高**：说明大部分样本都被裁剪，更新被严重限制（学不动或效率低）。
- **太低**：说明几乎没触发裁剪，更新温和；但如果同时 KL 很高，说明不是温和，是 ratio 在别的地方失控或实现统计口径不同。

### Grad norm

**是什么**

- 当前 step 的梯度大小（通常是全参数 L2 norm，可能是裁剪前或裁剪后，取决于日志点）。
- 它是训练稳定性的“地震仪”。

**怎么看**

- 重点看：是否**频繁尖峰**、是否**持续变大**、是否突然**接近 0**。

**常见态势**

- 正常情况下会抖动，但分布相对稳定；偶尔 spike 可以接受，关键是是否伴随 loss/KL 的异常。

**异常信号**

- **grad_norm 经常爆到很大**：学习率过高、advantage/奖励尺度过大、batch 太小噪声大、混精溢出风险。
- **grad_norm 长期很小**：学习率太低、clip/β 太强、梯度被裁剪得太狠、或者模型已进入平台期。

### entropy（熵）

![[Tech--强化学习-9.png|622]]

GRPO、PPO中⬇️：

![[Tech--强化学习-10.png|553]]

**是什么**

- 输出 token 分布的不确定性（随机性）。高=更分散、更探索；低=更确定、更“模板化”。
- 常作为正则项：鼓励探索、避免策略过早塌缩。

**怎么看**

- 看趋势：**缓慢下降**常见；**断崖式下降**危险（塌缩/骗分/更新过猛）。
- 也要结合质量指标：entropy 降低但 reward 提升且 KL 可控，通常 OK。

**异常信号**

- **entropy 暴跌 +** **KL** **上升 + grad_norm 尖峰**：更新过猛，策略可能“锁死”到某种模式。
- **entropy 很高但 reward 不涨**：探索多但没学到；可能奖励信号弱/噪声大/优势估计不稳定。
