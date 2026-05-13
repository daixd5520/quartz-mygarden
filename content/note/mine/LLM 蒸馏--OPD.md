---
tags:
  - 方法论
  - llm
  - 蒸馏
draft: "true"
---

# 动机

部分业务对线上时延要求比较高，要求模型在一定时间内处理完大量的数据，因此通常无法直接利用线上API进行上线。此时比较常规的做法是对线上API进行黑箱蒸馏，得到本地部署的轻量小模型，从而实现推理效率与效果的trade-off。

常见黑箱蒸馏是用闭源模型标注，然后 SFT->RL，但是小规模模型一般难以蒸馏到闭源大模型的性能

例如

利用豆包标注70w条样本，进行SFT和RL两阶段的训练，明显发现，最后的蒸馏效果与模型基座大小成正比，其中大规模Qwen3-14B可以接近Doubao的性能，而小规模基座Qwen3-4B和1.7B则有明显的性能断档，**此时我们便考虑，能否利用效果理想的大规模基座Qwen3-14B去提升小规模基座Qwen3-4B的效果，从而在不提升推理时延的情况下，明显提升小基座模型的性能。**

# OPD

OPD本质上就是用Teacher评估Student采样的轨迹，计算RKL作为稠密的监督信号

核心做法来自Thinking-Machine-Labs探究的Qwen3的On-Policy Distillation，该做法的核心是利用「**同架构**」的大规模Teacher模型对小规模的Student模型进行蒸馏，从而在「少量的训练步骤」下「快速提升」Student模型的效果。具体来说，On-Policy Distillation包含以下两个核心设计：

- **On Policy**：指的是样本**由Student模型采样**，而非Teacher模型采样，因此不会像SFT式的Off-Policy训练一样会存在Exposure Bias的问题，与强化微调RFT一样是On-Policy的。
- **Distillation**：指的是Student模型采样序列后，分别送入Student模型和Teacher模型计算**Per-token Logits**（词表维度），然后**计算Reverse KL Loss**进行训练，具体Loss如下：

$$
\mathcal{L}_{\mathrm{RKL}}(\theta)
=
\mathbb{E}_{x \sim \mathcal{D},\, y \sim \pi_\theta(\cdot \mid x)}
\left[
\log \pi_\theta(y \mid x) - \log \pi_{\text{teacher}}(y \mid x)
\right]
$$

把序列概率展开到 token 级别，就是：

$$
\mathcal{L}_{\mathrm{RKL}}(\theta)
=
\mathbb{E}_{x \sim \mathcal{D},\, y \sim \pi_\theta(\cdot \mid x)}
\left[
\sum_{t=1}^{T}
\left(
\log \pi_\theta(y_t \mid x, y_{<t})
-
\log \pi_{\text{teacher}}(y_t \mid x, y_{<t})
\right)
\right]
$$

如果进一步写成对每个位置词表分布的期望形式，则有：

$$
\mathcal{L}_{\text{token}}(\theta)
=
\mathbb{E}_{x \sim \mathcal{D},\, y_{<t} \sim \pi_\theta}
\left[
\sum_{t=1}^{T}
\mathbb{E}_{\hat{y}_t \sim \pi_\theta(\cdot \mid x, y_{<t})}
\left[
\log \pi_\theta(\hat{y}_t \mid x, y_{<t})
-
\log \pi_{\text{teacher}}(\hat{y}_t \mid x, y_{<t})
\right]
\right]
$$

这个目标写出梯度后，会发现它本质上可以看作一个特殊形式的 Policy Gradient。忽略可以当 baseline 扔掉的常数项后，有：

$$
\nabla_\theta \mathcal{L}_{\text{token}}(\theta)
=
\mathbb{E}_{x \sim \mathcal{D},\, y_{<t} \sim \pi_\theta}
\left[
\sum_{t=1}^{T}
\mathbb{E}_{\hat{y}_t \sim \pi_\theta(\cdot \mid x, y_{<t})}
\left[
\nabla_\theta \log \pi_\theta(\hat{y}_t \mid x, y_{<t})
\left(
\log \pi_\theta(\hat{y}_t \mid x, y_{<t})
-
\log \pi_{\text{teacher}}(\hat{y}_t \mid x, y_{<t})
\right)
\right]
\right]
$$

因此如果把它改写成“做梯度上升”的 Policy Gradient 视角，那么对应的优势函数可以写成：

$$
A_t
=
\log \pi_{\text{teacher}}(\hat{y}_t \mid x, y_{<t})
-
\log \pi_\theta(\hat{y}_t \mid x, y_{<t})
$$

也就是说，Teacher 比 Student 更偏好的 token，会得到正优势；Student 自己过度自信、但 Teacher 不认可的 token，会得到负优势。

- 当 $\pi_\theta(\hat{y}_t \mid x, y_{<t}) < \pi_{\text{teacher}}(\hat{y}_t \mid x, y_{<t})$ 时，$A_t > 0$，梯度上升会增大这个 token 在 Student 下的概率。
- 当 $\pi_\theta(\hat{y}_t \mid x, y_{<t}) > \pi_{\text{teacher}}(\hat{y}_t \mid x, y_{<t})$ 时，$A_t < 0$，梯度上升会降低这个 token 在 Student 下的概率。
- 当 $\pi_\theta(\hat{y}_t \mid x, y_{<t}) = \pi_{\text{teacher}}(\hat{y}_t \mid x, y_{<t})$ 时，$A_t = 0$，这一项没有更新压力。

这样看会更直观：OPD并不是像普通RL那样，只在整条回答结束后给一个稀疏奖励，而是在**每个位置、每个token分布**上都给出一个稠密的“偏好修正信号”。所以它虽然形式上很像 On-Policy RL，但监督信号要密得多，训练效率也通常更高。

因此本质上，OPD就是「**具有 Per-logits 稠密奖励信号的在线强化学习**」。同时也能理解为什么前文会强调「**同架构**」：只有同架构模型，分词器通常才一致，才能在 logits 层面对齐并计算 Reverse KL。这里所谓的 logits 层面对齐，指的是在每个 token 的**整个词表分布**上计算 loss；相对而言，常规的 token-level loss（如 SFT 或很多 RL 目标）通常只关心某个具体采样 token 的 log-prob。

## 为什么同架构和同 tokenizer 几乎是硬约束

这里可以把话说得更精确一点：**同 tokenizer 是更底层、更数学意义上的硬约束；同架构则是在这类 per-logits OPD 里，工程上几乎等价于硬约束。**

先说 tokenizer。OPD做的不是“Teacher 给一句参考答案，Student 去模仿这串 token”，而是：**在 Student 自己采样出来的前缀上，Teacher 和 Student 分别输出下一步的整个词表分布，然后直接做 Reverse KL。**

这件事隐含了一个前提：在时刻 $t$，Teacher 和 Student 所看到的“同一个 prefix”，必须真的对应同一串 token，也必须落在同一个词表坐标系上。否则所谓的 logits 对齐就是假的。

如果 tokenizer 不同，问题会立刻出现，而且不是小误差，是目标函数本身就变了：

- **词表维度对不上**：Student 的第 15231 维未必对应 Teacher 的第 15231 维，两个 logits 向量连坐标轴都不是一套。
- **切词粒度不一样**：Student 可能把一个片段切成 3 个 token，Teacher 可能切成 5 个 token。这样一来，“第 $t$ 步的 next-token distribution” 压根不是在同一个生成状态上比较。
- **特殊 token 语义不一致**：BOS、EOS、换行、角色分隔符、thinking token、工具调用标记如果编码方式不同，前缀语义会从一开始就错位。
- **概率质量无法直接对应**：Teacher 把概率放在一个长 token 上，Student 可能只能把概率分摊到多个短 token 上。这个时候你没法直接说“Teacher 在这个 token 上更偏好”，因为双方讨论的根本不是同一个离散事件。

所以一旦 tokenizer 不同，最直接的 per-logits Reverse KL 基本就没法定义了。你当然可以硬做一些映射，比如把 Teacher 的 token 概率投影到 Student 的 token 空间，或者退化成 sequence-level KD，但那已经不是这里讨论的这套 OPD 了。换句话说，**对这类直接在词表分布上算 RKL 的做法来说，同 tokenizer 不是“更方便”，而是目标函数成立的前提。**

再说为什么“同架构”在实践里也几乎是硬约束。严格从数学上说，如果两个模型**词表完全一致、前缀编码方式一致、输出语义一致**，那么它们不一定非得是同架构，也可以在同一个 token 空间里比较分布。但现实里这几个条件太苛刻了，而“同架构”通常就是拿来一口气保证这些条件都尽量成立的。

它带来的好处至少有四层。

第一层是**输入语义对齐更自然**。同架构模型通常共享：

- tokenizer
- chat template
- special tokens 约定
- position encoding / RoPE 规则
- attention mask 习惯

这意味着同一段 prefix 喂进去之后，Teacher 和 Student 至少是在“同一种语言系统”里理解它，而不是一个按这个规则切、另一个按那个规则切。

第二层是**输出头语义更接近**。OPD学的是 logits 分布，不只是某个 token 的 one-hot 标签。Teacher 给 Student 的不是“标准答案”，而是一整条软分布曲线。只有当两边模型家族足够接近时，这条软分布才更像“高分辨率监督信号”。

如果架构差异很大，经常会出现两种问题：

- **归纳偏置不同**：比如一个模型更擅长长程依赖，另一个模型在局部模式上更强，Teacher 的分布形状对 Student 来说不一定是可学习的。
- **校准方式不同**：两边 logits 的尖锐度、熵、温度感都可能不一样。即使 token 空间相同，Student 也会觉得 Teacher 的分布“太怪”，优化会变得更难。

第三层是**中间格式和行为习惯更接近**。推理模型里这点尤其重要。比如：

- 是否显式输出 think token
- 是否使用特定的分隔符组织 reasoning / answer
- 遇到多轮对话时偏好的回复骨架
- 工具调用、代码块、数学公式的格式习惯

这些东西看上去像“表面风格”，但在自回归模型里它们不是表面现象，而是会真实改变后续 prefix 分布的状态变量。Teacher 和 Student 如果在这些局部格式习惯上差得太大，Student rollout 出来的轨迹对 Teacher 来说就会越来越 out-of-distribution，蒸馏信号会变脏。

第四层是**容量差距更可控**。OPD通常希望 Teacher 比 Student 强，但不是强到完全处在另一个物种。你要的是一个“同一路线上的更强版本”，而不是一个世界观完全不同的裁判。比如 Qwen3-14B 蒸馏 Qwen3-4B，这种 setup 的直觉就很顺：Teacher 学到的很多分布形状、推理节奏、token 偏好，Student 至少有希望在同一家族里继承下来。

这也是为什么业界经常会发现：

- **同家族大蒸小**，收益通常最稳定
- **异家族硬蒸**，就算 teacher 很强，student 也未必真能吃到多少

不是因为异家族一定不能蒸，而是因为你失去了那种“logits 可以直接当软标签”的低摩擦条件，很多时候优化器会先学到 Teacher 的表面分布形状，却学不到真正支撑这些分布的内部结构。

所以更准确的说法是：

- **同 tokenizer**：这是 direct per-logits OPD 的硬约束，不满足的话，RKL 本身就没法在同一个离散空间里定义。
- **同架构**：这是工程上几乎等价于硬约束的强条件。它不一定是数学上唯一必要条件，但它把 tokenizer、prefix 语义、输出分布语义、推理格式习惯这些最容易出问题的点一起锁住了。

如果一定要在异架构之间做蒸馏，通常就得退一步：

- 不再做最直接的 per-logits RKL
- 改做 sequence-level distillation
- 或者引入 token mapping / hidden-state projection / adapter 对齐

但这样做的代价是，原本 OPD 里最值钱的那部分“细粒度、低噪声、逐 token 分布监督”会被削弱。说得直白一点，**OPD 之所以香，很大一部分就是因为 Teacher 和 Student 足够像，像到 logits 可以直接拿来当监督信号。**

## OPD vs RL vs SFT

如果只看训练范式，OPD最值得讲的地方不在“它也是蒸馏”，而在于它刚好卡在 SFT 和 RL 中间，把两边最值钱的部分拼到了一起。

从两个维度看最清楚：

- **采样方式**：样本到底是谁采的，是 Teacher 采，还是 Student 自己采。
- **监督信号**：更新到底有多细，是整条序列一个分数，还是细到每个 token、甚至每个位置的整条词表分布。

### SFT

SFT本质上是 **Off-Policy**。训练数据通常来自人工标注、规则构造，或者 Teacher 模型直接生成的参考答案。Student 学到的是“在 Teacher 常出现的 prefix 上，下一步应该怎么输出”。

如果写成概率视角，SFT最大化的是 Teacher 数据上的对数似然：

$$
\max_\theta \;
\mathbb{E}_{(x, y) \sim \mathcal{D}_{\text{teacher}}}
\left[
\log \pi_\theta(y \mid x)
\right]
$$

从分布角度看，这可以理解成在 Teacher 数据分布上逼近 forward KL，所以它天然带有 distillation 的味道。它的优点很明显：

- 监督信号细到 token 级
- 优化稳定
- 冷启动非常好用

但它有一个老问题：**训练时看到的是 Teacher 轨迹，推理时走的是 Student 自己的轨迹。**

这就是 Exposure Bias。短回答问题不一定很致命，长链条推理里就很麻烦。Student 只要前面一步走偏，后面就会进入训练时几乎没见过的 prefix 区域，误差会越滚越大。

### RL

这里说的 RL，主要指 post-training 阶段常见的 RLVR / GRPO 这一类做法。它的特点刚好和 SFT 反过来：

- **采样方式**是 On-Policy，样本由 Student 自己采
- **监督信号**通常是稀疏的，很多任务最后只给一个正确 / 错误，或者一个序列级分数

这类方法的好处是训练推理一致。模型是在自己的轨迹上被纠偏，不容易出现 SFT 那种“老师带着走时学得挺好，自己一上场就散架”的问题。

但问题也同样明显：**奖励太稀。**

如果 verifier 只在整条回答结束后给一个 0/1 信号，那么反向传播时，整条序列上的 token 基本都被同一个序列级 reward 一起推。这样当然能学，但样本效率会很差，因为模型只知道“这条轨迹整体好 / 不好”，却不知道具体是哪里好、哪里坏。

这也是为什么 RL 在 reasoning 模型上很有效，但训练成本经常很高。你确实能学到“探索自己的轨迹空间”，但为此要交不少算力税。

### OPD

OPD最妙的地方，就是它把这两件事同时拿到了：

- **轨迹来自 Student 自己**，所以它是 On-Policy
- **监督来自 Teacher 的 per-logits 分布**，所以它又是稠密的

也就是说，OPD既不像 SFT 那样完全依赖 Teacher 轨迹，也不像 RL 那样只拿一个序列级稀疏 reward。它是在 Student 自己走出来的 prefix 上，让 Teacher 对每一步、每个 token 分布都给出一个细粒度修正。

如果只用一句话概括三者差别：

- **SFT**：Teacher 采样，token 级监督，但 Off-Policy
- **RL**：Student 采样，On-Policy，但奖励通常稀疏
- **OPD**：Student 采样，On-Policy，同时又保留了 per-logits 的稠密监督

这也是为什么 OPD 看上去像 RL，但训练效率往往比纯 RL 好很多；同时它又有 distillation 的味道，但比纯 SFT 更能缓解 Exposure Bias。

### 为什么 OPD 训练效率更高

这件事本质上不神秘。SFT 和 RL 分别卡在两个不同的瓶颈上：

- SFT 的问题是分布错位，Student 没有在自己的轨迹上学
- RL 的问题是奖励太 sparse，信用分配太粗

OPD 刚好把这两个问题对冲掉了。

对同一条 Student rollout：

- RLVR 可能只会说“这题做对了”或者“这题做错了”
- OPD 会在每一个位置都告诉你：Teacher 比你更偏好哪些 token，你又在哪些 token 上过度自信了

从优化角度看，这相当于把“一个序列级标量 reward”换成了“逐 token、逐词表分布的稠密修正信号”。没人会在训练里嫌监督更密。

所以 OPD 往往会呈现出一个很有意思的经验现象：**步数不需要很多，但起效非常快。**

### 和 Qwen3 结果放在一起看

Qwen3 技术报告里，On-Policy Distillation 的卖点也正是这个“快”和“省”。

报告里给出的结论是，在小模型 post-training 上，OPD 相比四阶段训练流程有明显更好的立即收益，同时 GPU hours 只需要大约十分之一。你给的那组数字也能很好说明这个趋势：

- SFT 冷启后继续走 RL 微调，Qwen3-8B 可以做到约 `67.6%`
- 这一路需要大约 `17920` GPU hours，成本非常重，量级上大致相当于再做一轮大规模 SFT
- 而 On-Policy Distillation 在更低成本下就能达到更好的结果

另一组更有代表性的数字是：

- 在 `400k` 样本上做完 SFT 的 Qwen3-8B
- 再继续做 OPD
- 只要大约 `150` 步，就能把分数拉到 `70%`

如果这个量级成立，那它传递出的信息其实很明确：**OPD 不是靠“训练更久”赢的，而是靠“监督更贴着 Student 当前错误分布”赢的。**

从 FLOPs 角度看，这种做法能省下 `9x` 到 `30x` 的成本，也就不奇怪了。因为它减少的不是某个常数项，而是减少了大量“虽然在训练，但信号没那么对位”的无效更新。

### 一个更工程化的总结

如果让我做很粗的决策，我会这么看：

- **SFT** 适合冷启动、打基础、注入格式和风格
- **RL** 适合最终追求 task reward、探索策略空间、做能力上限突破
- **OPD** 适合在已有不错 Student 的前提下，利用强 Teacher 快速把能力往上拽，尤其适合 strong-to-weak distillation

也就是说，OPD 不一定要替代 SFT 或 RL。更常见、更合理的位置其实是：

- 先用 SFT 把模型拉到一个能说人话、能稳定输出的状态
- 再用 OPD 快速吸收强 Teacher 的分布级知识
- 最后如果任务真的需要 verifier 驱动的目标最优，再补 RL

这也是为什么 OPD 很适合放在“中间那一段”做能力跃迁。它不像 SFT 那样更多是在学老师走过的轨迹，也不像 RL 那样更多是在稀疏 reward 下自己瞎撞。它更像是：**Student 先自己走，再让 Teacher 在每一步上指出“你这里该往哪边偏一点”。**

## 为什么 reverse KL 比 forward KL 更适合 strong-to-weak distillation

这件事如果只背一句“forward KL 是 mode-covering，reverse KL 是 mode-seeking”，其实远远不够。真正到了 strong-to-weak distillation 的场景，关键不是背术语，而是要看：**Teacher 明显比 Student 强，Teacher 的分布就是更复杂，而 Student 的容量根本装不下这整套复杂性。**

这时你逼 Student 学什么，就变成了核心问题。

### forward KL 在逼 Student “全都想要”

先从最常见的 SFT 说起。SFT 本质上更接近 forward KL 那一侧：Teacher 给出参考轨迹，Student 在 Teacher 轨迹上最大化似然。它隐含的目标是：**Teacher 高概率的地方，Student 最好都要覆盖到。**

这在分类任务里通常没什么问题，因为类别就那么几个。但在生成任务里，尤其是 reasoning 场景，Teacher 的下一 token 分布经常是多峰的。

比如在某一步上，Teacher 可能觉得下面几种 continuation 都讲得通：

- 继续走推理链 A
- 改写成更紧的推理链 B
- 先补一句解释，再收束到答案

对一个大模型 Teacher 来说，这几个模式它都能维持，因为它容量够大，知道每条路后面会怎么走。

但对一个小模型 Student 来说，问题在于：**它往往没能力同时把这几种模式都学好。**

如果这时你用更偏 forward KL 的目标去逼它，效果经常会变成：

- Teacher 哪些地方有概率质量，Student 都得去分一点质量
- Student 被迫对多个模式同时表忠心
- 结果是每个模式都学得不够扎实

说得更直接一点，forward KL 对小模型不太友好的地方在于：**它默认 Student 只要足够努力，就能把 Teacher 的高概率区域都罩住。** 但 strong-to-weak distillation 里，这个假设往往一开始就不成立。

于是很容易出现一种局面：Student 为了“别漏掉 Teacher 的模式”，把概率撒得到处都是，最后看起来更平、更虚、更不敢做决定。落到生成上，就是：

- 该果断输出时不够果断
- 该沿着一条推理链走深时，反而在多个可能性之间摇摆
- 生成质量看起来像“什么都学了一点，但没有一个学得特别像样”

### reverse KL 更符合“小模型只能挑重点学”的现实

reverse KL 的视角正好反过来。它不是问“Teacher 哪些模式你都覆盖到了吗”，而是问：

- **你当前自己会生成什么**
- **在这些你自己已经会走到的区域里，Teacher 更偏好什么**

也就是说，reverse KL 不是强迫 Student 先去把 Teacher 的所有模式都兜住，而是让 Student **先在自己的支持集上站稳，再在这个局部空间里往 Teacher 的方向修正。**

这和 strong-to-weak distillation 的现实约束是非常一致的。因为小模型最怕的，不是“没把大模型所有模式都学全”，而是“本来就学不全，还被逼着平均分配精力”。

reverse KL 更像一种资源受限下的合理策略：

- 小模型做不到完整复刻 Teacher
- 那就不要逼它全盘覆盖
- 而是在它当前能到达、能表达、能稳定优化的区域里，优先学 Teacher 最有价值的偏好

这也是为什么 reverse KL 往往会让 Student 更“像一个明确的模型”，而不是“像一个被大模型的多峰分布拉扯得很散的半成品”。

### 一个很具体的直觉例子

假设 Teacher 在某一步的分布是这样的：

- token A：`0.45`
- token B：`0.40`
- 其他 token：`0.15`

这说明 Teacher 眼里，A 和 B 都是合理延续，只是 A 略优一点。

但 Student 容量不够，它没法同时把 A 后面的整条推理链和 B 后面的整条推理链都学完整。对它来说，更现实的情况通常是：

- 要么把 A 这条链学扎实
- 要么把 B 这条链学扎实
- 同时两条都学好，超纲了

如果这时用更偏 forward KL 的目标，优化会不断提醒它：

- A 很重要
- B 也很重要
- 两边你都别丢

于是 Student 可能学出来一个中间态：A 不够稳，B 也不够稳，概率还撒给一堆边缘 token。

而 reverse KL 更像是在说：

- 你现在自己更常走到哪条路？
- 如果你已经走到 A 这边了，那就把 A 这边学得更像 Teacher
- 如果你已经走到 B 这边了，就把 B 这边修正得更好

它不要求小模型先变成一个“什么都会一点的大模型缩略版”，而是允许它先形成一个更窄、但更靠谱的决策边界。

对小模型来说，这通常反而更有价值。因为部署时你真正要的，往往不是“它知道所有可能性都存在”，而是“它在自己会走的那条路上别走歪”。

### 为什么这在 reasoning 蒸馏里尤其明显

reasoning 场景比普通 chat 更容易把这个问题放大。因为 reasoning 不是单步分类，而是长链条生成。某一步如果分布开始发散，后面整条链都会跟着偏。

Teacher 之所以强，不只是因为它知道正确答案，而是因为它能维持很多条中间推理路径，同时在后面把这些路径收回来。小模型通常没有这个本事。

这时如果你还要求 Student 去覆盖 Teacher 的多个潜在推理模式，等于是在它本来就不够用的容量上继续施压。结果往往是：

- 中间推理链发散得更厉害
- token 熵偏高
- 最终答案不一定更准，反而更容易“看起来懂了，其实没走稳”

reverse KL 更适合 reasoning distillation 的地方就在这里：它更像是在**沿着 Student 自己已经走出来的思路做局部纠偏**，而不是要求它同时掌握 Teacher 所有可能的思路分叉。

这和 OPD 的 on-policy 属性又是天然契合的。Student rollout 到哪，Teacher 就在这个 prefix 上给 logits；所以优化目标天然聚焦在“Student 真会去的状态”，而不是“Teacher 理论上可能去的所有状态”。

### 从优化稳定性看，reverse KL 也更顺

在 strong-to-weak 里，Teacher 和 Student 的能力差通常不小。能力差一大，forward KL 很容易把 Student 推向一个它根本无力表达的目标分布。

这会带来两个工程上很烦的问题：

- **过度平滑**：Student 为了覆盖 Teacher 的多个模式，把概率摊得很开
- **无效逼近**：优化器在追一个 Student 根本表示不出来的分布，更新很多，但收益有限

reverse KL 相对顺手的原因，是它天然只惩罚 Student 自己已经放了质量的地方。Student 没去的区域，reverse KL 不会像 forward KL 那样强行要求它补齐。

这对小模型很关键，因为它减少了那种“明明学不会，还被迫为此付梯度”的浪费。

你可以把两者粗暴理解成：

- **forward KL**：Teacher 说“我会的这些你最好都学”
- **reverse KL**：Teacher 说“你已经在做的这些里，哪些该提高，哪些该压低”

前者更像全面课程表，后者更像针对错题本订正。对于 strong-to-weak，这个差别非常大。

### 所以在 strong-to-weak 里，reverse KL 的优势到底是什么

如果压成几句话，我会这么说：

- 小模型容量有限，最怕被逼着覆盖大模型的全部模式
- forward KL 更容易让 Student 去做这件事，于是容易学得散
- reverse KL 更尊重 Student 当前的支持集，只在它自己会到达的区域里做修正
- 这让蒸馏目标更贴近“小模型学得会什么”，而不是“大模型会什么”

这也是为什么 OPD 这种 **Student on-policy rollout + reverse KL** 的组合会很适合 strong-to-weak distillation。不是因为 reverse KL 在任何地方都天然更高级，而是因为在这个具体问题里，它更符合容量受限 Student 的优化现实。

如果再说得更直白一点：

- **forward KL** 更像在问：你为什么没学会老师会的所有东西？
- **reverse KL** 更像在问：你已经会做这些了，那怎么把它做得更像老师？

对小模型来说，后一个问题通常更可答，也更有训练价值。
