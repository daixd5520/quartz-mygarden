# Skill 的本质

```Markdown
---
name: frontend-design
description: Create distinctive, production-grade frontend interfaces with high design quality. Use this skill when the user asks to build web components, pages, artifacts, posters, or applications (examples include websites, landing pages, dashboards, React components, HTML/CSS layouts, or when styling/beautifying any web UI). Generates creative, polished code and UI design that avoids generic AI aesthetics.
---

This skill guides creation of distinctive, production-grade frontend interfaces that avoid generic "AI slop" aesthetics. Implement real working code with exceptional attention to aesthetic details and creative choices.

...
```

众所周知，Skill 本质上是一个**按需加载的 prompt** **、模板****和脚本包**：一个带 YAML Frontmatter 的 `SKILL.md`，加上可选的脚本、参考文档、模板。Claude 启动时只加载 `name + description`（几十 tokens），只有当任务匹配 `description` 时才会把完整内容读进上下文。所以它解决的是“我有一堆领域知识/流程，但不想常驻 context”的问题。

**为什么 Skill 不是靠向量召回的？**

你可能会好奇，为什么 Skills 不是通过语义相似度或 BM25 召回的，而是将 Skills 的 Frontmatter 信息常驻在上下文里呢？这是因为我们希望 Skill 是原子的、可二次组合的。因此，希望由大模型来决定为了完成用户提出的任务，究竟是使用单一 Skill 还是多个 Skills 组合。那么如果 Skills 过多，占用太多上下文空间怎么办呢？单独用一个 Flash / Lite 的小尺寸大语言模型来做 Skills 挑选即可。

### 关键机制

- **渐进披露 (progressive disclosure)**：SKILL.md 是入口，里面再 `Read` 其他文件。不要把所有东西塞进 SKILL.md。在 [DeerFlow 的官网](https://deerflow.tech/)上，我用一个动画形象的演示了该过程。
- **description 是唯一的触发器**：写不好就永远不会被调用。不仅要包含**“什么时候用 + 用来干啥”**两个维度，有时候还需包含**“什么时候不能用”**的逻辑。
- **Skill ≠ Prompt ≠ Agent ≠ Hook**：Skill 是知识/流程，Agent 是独立上下文的子进程，Hook 是确定性触发的 shell 命令。自动化行为（“每次 X 都要 Y”）应该用 Hook，不是 Skill。

# 2 写skill

## 2.1 用什么写

用 AI 写，给 AI 看。

不仅如此，你需要用最贵的模型编写（如 Claude Opus），这样你才能有机会用更便宜的模型（如 Claude Sonnet）执行这个 Skill。至于使用英文还是中文写，也是由 AI 自己来决定的，通常是英文为主。至于编辑器，最好的工具就是 `skill-creator` 这个 Skill 本身，一般的 AI 工具都内置了这个 Skill，你可通过 `/skill-creator` 来执行。理解了这个层面后，接下来我们就来介绍一下 `/skill-creator` 里应该怎么和 AI 沟通你的需求。

**示例：**

```Markdown
/skill-creator 帮我实现一个技能，根据当前仓库的现有代码，提取出典型的团队代码风格：
1. 首先，通过文件树摸清这个仓库的主要编程语言，然后 propose 几个你认为最重要的文件。这些被挑选出的文件个数不能低于 3 个或多于 10 个，这些文件应该足以让你将了解团队代码的基本风格，同时尽可能的涵盖了项目中不同的编程语言、不同的架构层（配置、数据访问、API 暴露、Thrift 定义等）
2. 提取出团队代码的文件夹、文件、方法、成员、类型命名的规律（允许一个类别下有不同的命名方法），细致到例如分页器的参数是如何命名的
3. 作为 Coding Agent，你应该比团队还要更加了解自己的代码风格，而不是事无巨细的列出大家都知道的 rules（token-saving），才足以体现出你的价值
4. 总结为简明扼要的、方便 AI 理解而不是人类理解的 markdown 文档：docs/code-convention.md
```

## 2.2 什么时候做渐进式披露

通常我们会把参考资料（reference）、示例（example）、模板（template）和脚本（Node.js / Python script）作为 Skill 渐进式披露的零件。就像我们前面提到的那样，最好的 Skill 编辑器就是 AI 自己，因此在通过 `/skill-creator` 生成了一个庞大的 Skill 后，你只需要对 Agent 说：

> 当前 SKILL.md 太长了，缺乏 progressive-disclosure 机制，但是也请不要滥用。

剩下来的事情就交给 AI 吧，它会帮你拆分成若干文件。例如，如果你的 Skill 里有模板和示例，并且内容很长，它会帮你拆分到对应目录的 markdown 文件里；再比如你的 Skill 里有类似 `switch...case` 的逻辑分支，并且每一个分支的逻辑都很庞大，它就会帮你把这些逻辑拆分成独立的 Markdown 文件，在运行时只有命中条件的一个或多个逻辑分支才会被加载。

当然，如果你的 Skill 里需要执行“机械式”的逻辑，skill-creator 也会帮你用 Node.js 或 Python 代码来实现，这样就不会在这些环节出现幻觉或漏执行（你也可以主动要求要用脚本执行某些环节），你可以直接对 Agent 说：

> Skill 里可以“机械式”执行的部分（如果有）请帮我用 Python 实现。

## 2.3 保留 Human-in-the-Loop 的交互

一个好的 Skill 应该适当的引入 Human-in-the-Loop。你可以在 Skill 的 prompt 里明确要求它使用 `AskUserQuestion` 来与用户进行多轮交互。`AskUserQuestion` 工具支持单选、多选和预览单选等交互，同时还支持 Step-by-step 式的向导。

# 3 持续迭代--Agent RL

如果你使用的工具正好是 Claude Code，那么恭喜你，在 Claude Code 里跑 `/skill-creator` 其实就是：**生成 → 你在真实任务里用它 → 失败/别扭的地方反馈 → 让 skill-creator 改 SKILL.md**。

**每一轮都是一次人工 RLHF。skill-creator 自带了** **eval（评估）能力**，生成完 Skill 后，主动跟它说：

Run evals on this skill with your mocked test cases, and I'll return you with my feedback via the Eval Review web page you provided

Claude Code 会帮你自动生成测试用例，并用多个 Sub-agent 并行运行评估：

![[Tech--如何写好 skill-1.png]]

在评测完成后，它会生成一个 `Eval Reviewer` 网站用于查看评测结果，它会给你查看用了这个 Skill 和没有用的区别。

![[Tech--如何写好 skill-2.png]]

# 4 来自`claude-api` 的启示

看看 `[claude-api](https://github.com/anthropics/skills/blob/main/skills/claude-api/SKILL.md)` 这条 skill 的写法，就是本文说的模式。歧义越大的场景，反例越重要。

1. **先写“一个能跑的例子”，再抽象成 skill** 反直觉但好用：先让 Claude 手动完成一次任务，把过程记下来，然后对 skill-creator 说"把这段流程固化成 skill"。从具体到抽象，比从抽象凭空写效果好得多。
2. **把大知识拆文件，SKILL.md 只放索引** SKILL.md 里写 "详细 API 参考见 `reference/api.md`，模板见 `templates/`"。Claude 需要时会自己去读。SKILL.md 本身控制在 100 行内最好，因为它是每次都要过一遍的。
3. **脚本优先于 prompt** 如果某个子步骤是确定性的（格式转换、校验、调 API），写成 `scripts/xxx.py` 让 skill 去执行，比用自然语言描述可靠一个数量级。skill-creator 默认偏好 prompt，你要主动说"这步用 Python 脚本实现"。
4. **用** **`allowed-tools`** **frontmatter 收窄权限** 在 SKILL.md frontmatter 里限定 `allowed-tools: Read, Grep, Bash(git log:*)`，避免 skill 在不该动手的场景乱改文件。skill-creator 不会主动加这个，需要你提。
5. **测试驱动：先写 eval，再写 skill** 让 skill-creator 先生成 5-10 个测试 prompt（带预期行为），再让它写 SKILL.md，最后跑 eval 对齐。这就是前面说的 RL 循环的自动化版本。

# 5 并行多任务

## 5.1 前言

搞清楚一件事：**什么样的任务适合并行，什么样的不适合**。

笔者最近在为一个通用 **EDA（Exploratory Data Analysis）Skill** 设计执行架构时，深入思考了这个问题。EDA 就是给 Agent 一个 CSV 或 Excel，让 AI 不断自己命题、自己编写并执行 Python 代码进行查询统计，在认为自己足够了解数据后，撰写最终的报告，是一个标准的 Code-act 过程。本文把并行子任务的价值拆解为三个维度——**性能**、**隔离**、**多样性**——逐一展开，最后讨论何时不该并行以及工程上的取舍。

**题外话：什么是 EDA？**

EDA（Exploratory Data Analysis，探索性数据分析）是数据分析的"第一眼"——在你建模、下结论之前，先让数据"说话"。它没有固定流程，核心是用统计 + 可视化快速摸清数据的长相：有多少行列、分布是否正常、哪些字段高度相关、哪里有缺失或异常。EDA 的价值不在于给出答案，而在于帮你问出**正确的问题**。

以下是我对 EDA 这个 Skill 的两种架构设计对比，一种是`串行架构`，另一种则是`并行架构`：

![[Tech--如何写好 skill-3.png]]

## 5.2 优势

### 5.2.1 性能

单 Agent 串行执行时，每一步推理都要等上一步完成。对一个典型的 EDA 任务来说，单变量分析、相关性计算、异常检测、趣味发现挖掘，这四块工作之间几乎没有数据依赖。串行执行意味着总时间是四块之和；并行执行则取决于最慢的那块——理论上接近 4 倍加速。

> 这不是理论推演。M1-Parallel 论文（2025 年发表于 arXiv）在复杂推理任务上实测了并行多团队执行的效果，报告了高达 2.2 倍的端到端加速，同时保持了准确率不下降。而 Anthropic 在其 Research 系统的内部评测中发现，多 Agent 架构（Claude Opus 4 领衔 + Claude Sonnet 4 子 Agent）相比单 Agent Claude Opus 4，在研究任务上的表现提升了 90.2%。

但“快”还只是表层。更深层的性能优势在于**智力密度**（intelligence density）。Anthropic 的分析揭示了一个惊人的数字：

> 在 BrowseComp 评测中，token 使用量单独就能解释 **80%** 的性能差异。换句话说，Research 任务的质量几乎正比于你能投入多少计算量。多 Agent 并行执行的真正价值，是让你能在相同的墙钟时间内，投入数倍的 token——也就是**数倍的“思考量”**。
> 
> **笔者按**：但也很耗费金钱

### 5.2.2 上下文隔离

LLM 的上下文窗口是一种稀缺资源。往里面塞的东西越多，模型的注意力就越分散，推理质量就越退化。LangChain 的官方文档对此有一个非常精辟的量化分析：在一个需要比较 Python、JavaScript 和 Rust 的任务中，使用 Subagent 模式（每个子 Agent 只加载自己需要的 2000 token 文档）总共消耗约 9K token；而使用 Skills 模式（所有文档都塞进同一个上下文）则膨胀到 15K token。子 Agent 模式的 token 消耗减少了 67%。

上下文隔离带来三个层次的好处：

**第一，注意力聚焦：**当一个子 Agent 只负责“异常检测”这一件事时，它的整个上下文窗口都被异常检测相关的数据、代码输出和推理过程所填满。它不需要“记得”相关性分析的中间结果，也不需要“忽略”Fun Facts 挖掘过程中产生的噪音。它拥有一间干净的房间，可以心无旁骛地工作。

**第二，降低"上下文污染"风险：**在长序列推理中，前面步骤的错误会像传染病一样扩散到后续步骤。一个统计计算中的小错误可能让后续的异常检测产生误判，误判又可能污染最终的 Fun Facts。并行执行天然切断了这条错误传播链——每个子 Agent 从干净的状态出发，互不干扰。

**第三，延长有效工作时间：**Claude Code 的子 Agent 设计就是一个经典案例。主 Agent 把需要大量阅读和搜索的"调研类"任务交给子 Agent 完成。子 Agent 做完后只返回结论，而不是把整个调研过程的 trace 灌回主 Agent 的上下文。这样主 Agent 的上下文保持精简，能持续工作更长时间而不触及上下文窗口上限。

### 5.2.3 多样性

多条路径同时展开，总有一条能找到最优解。

![[Tech--如何写好 skill-4.png]]

这正是 M1-Parallel 框架的核心思路：并行运行多个 Agent 团队，每个团队独立制定计划、独立执行，然后通过"早停"（early termination）或"聚合"（aggregation）策略选出最佳结果。实验表明，即使不刻意引导各团队走不同的路径，单纯依靠 LLM 采样的天然多样性，并行执行就已经能显著提升任务完成率。

在 EDA 场景中，这种多样性体现得尤为具体。比如“Fun Facts 挖掘”这个子任务，不同的子 Agent 可能会从完全不同的角度切入数据：一个发现了时间维度上的异常分布，另一个发现了某个分类变量中的 Zipf 定律偏离，第三个注意到了两列之间出人意料的负相关。如果用单 Agent 串行执行，你大概率只会得到最"主流"的那一两个发现——因为模型的注意力被前序步骤的结果所锚定（anchoring effect），很难跳出既有的思维框架。

![[Tech--如何写好 skill-5.png]]

**Sub-agent vs Agent Team**

聪明的你可能已经想到了，既然可以综合四个 Sub-agent 的视角，是否还可以让不同的 Agent 辩论呢？是的，这时候你需要[在 Claude Code 中开启 Agent Team（实验功能）](https://code.claude.com/docs/zh-CN/agent-teams)。他们可以先“吵架”再“复合”。事实上，让背后运行不同的模型的 Agent 以及不同 Prompt 的 Agent 在一起讨论，还真的可以得到更全面、更值得推敲的结果，唯一的缺点就是——**Agent Team 真的好耗 Token 啊**！

多样性的价值在**“阅读型”**任务中远大于**“写作型”**任务。这也是 Anthropic 和 Cognition 看似矛盾却实则一致的原因——Anthropic 做的是 Research（阅读、搜索、综合），Cognition 做的是 Coding（写代码、修 bug、集成）。Research 天然适合并行探索，因为搜索空间大且各方向互不干扰；Coding 则对一致性要求极高，并行写出的代码片段很容易互相冲突。【不同场景需要不同工具】

## 5.3 何时不应并行

三种应该谨慎使用并行的场景。

**场景一：子任务之间存在强依赖。** 如果子任务 B 的输入依赖子任务 A 的输出，那并行就失去了意义。在 EDA 中，"数据加载"和"数据质量审计"就是这种关系——你必须先成功加载数据，才能审计数据质量。笔者在 EDA Skill 中把 Phase 0（加载）和 Phase 1（质量审计）设计为串行，只在 Phase 2（深度分析）才展开并行。

**场景二：子 Agent 之间需要"商量"。** Cognition 的核心批评就在这里。当多个子 Agent 需要在执行过程中互相协调——比如两个 Agent 写的代码要合并到同一个代码库——并行执行就会产生冲突。今天的 LLM 还不擅长这种跨 Agent 的实时协商。Cognition 的原话很有画面感：这就像让两个从未见过面的工程师各自写一半代码，然后指望第三个人把它们拼在一起。

**场景三：上下文共享不充分。** 如果 Lead Agent 给子 Agent 的任务描述过于笼统——比如只说"研究半导体短缺"而不指定应该关注哪个时期、哪个细分市场——那多个子 Agent 很可能做出大量重复工作。Anthropic 在早期开发中就踩过这个坑。解法是 Lead Agent 在 spawn 子 Agent 时必须提供非常具体的任务描述，包括调查目标、输出格式、使用的工具和明确的任务边界。

## 5.4 如何在 Skill 中提示

通过 `skill-creator` 创建 Skill 时，你可以这么说来启发 Agent 在 Skill 里用并行：

为提高效率，应显式使用 **spawn** 这个单词来启发模型启动**并行子任务**（如单变量分析、双变量分析、数据质量检查、Fun Facts 挖掘等同步推进）。在 SKILL.md 的 Frontmatter 里显式的添加 allowed-tools: Agent Task。 使用多任务运行时，可以遵循“先并行，再串行，再并行，再串行”的模式。注注意并行的任务之间应当尽量的独立不依赖。如果用户开启了`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`，则 Agent 之间可以协作。

`spawn` 这个单词程序员们一定不会陌生，通常是指 **“生成”、“创建”** 或 **“启动”** 一个**新的独立执行单元**，。而Frontmatter 指的是 SKILL.md 的文件头部中夹在两个“---”横线之间的内容，而 `allow-tools` 属性是提示 Agent 可以使用哪些工具（用空格分隔），其中 `Agent` 工具分别是 Claude 用于 Sub-agent 多任务的工具，而 `Task` 则是老版本中的名称，不少其他的 Agent 因为历史原因也叫 `Task`。

## 5.5 skill示例--EDA

```Markdown
/skill-creator 我想设计并实现一组通用的 CSV/Excel 数据源的 Exploratory Data Analysis Skill。Skill 用英文撰写。

# Steps
- 通过 Code-Act Loop（短期 Planning → 执行 → 观察 → 迭代）驱动分析过程，而不是固定的 Workflow
- 使用 Pandas / NumPy 做数据处理和统计计算
- 使用 Mermaid 做数据可视化（嵌入 Markdown）
- 最终产出一份 支持 Mermaid 的 Markdown 富文本报告，包含：
  - 数据基本面（规模、类型、缺失值等）
  - 发挥 EDA 和 Code-Act Loop 的特长，根据数据灵活决定内容，而不是固定的版块
  - Fun Facts（数据集中有趣或反直觉的发现）
  - 推荐下一步探查的方向

# 强调多任务
- 为提高效率，分析阶段应显式使用 spawn 这个单词来启发模型启动并行子任务（如单变量分析、双变量分析、数据质量检查、Fun Facts 挖掘等同步推进）。在 SKILL.md 的 Frontmatter 里显式的添加 allowed-tools: Agent Task。 
- 其中，第 2 步是可以用多任务进行的，可以“先并行，再串行，再并行，再串行”的模式。注意并行的任务之间应当尽量的独立不依赖。
- 如果用户开启 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`，则 Agent 之间可以协作。
```

## 5.6 子skill

当某个子任务本身足够复杂——复杂到它有自己的工作流、自己的参考文件、甚至自己的 Code-Act 循环——把它硬塞在主 Skill 中就开始显得拥挤了。

更好的做法是：**把复杂子任务独立成一个子 Skill，然后在主 Skill 中通过 spawn 子 Agent 时指定使用哪个 Skill 来执行。**

这本质上是 Skill 层面的**“分而治之（Conquer and Divide）”**。主 Skill 扮演 Orchestrator 的角色，只负责任务分解和结果汇总；每个子 Skill 是一个自包含的专家模块，拥有自己的 SKILL.md、references 目录、甚至 bundled scripts。子 Agent 被 spawn 出来后，加载对应的子 Skill，在自己的上下文里独立完成工作，最后把结果交回主 Agent。

这样做的好处是三重的：

1. 每个子 Skill 可以**独立迭代**——修改异常检测的逻辑不需要动主 Skill 的任何一行；
2. 子 Skill 可以**跨场景复用**——同一个 `financial-analyzer` Skill 既能被 EDA Skill 调用，也能被用户直接触发；
3. 它天然实现了 Progressive-Disclosure——主 Skill 只需要知道子 Skill 的名字和职责，不需要把子 Skill 的完整指令加载进自己的上下文。

以笔者的 EDA Skill 为例。Phase 2 的四个子任务中，“Fun Facts Mining”其实可以做得非常深——它可能需要检测 Zipf 分布、计算 Benford 定律偏离度、搜索历史上的今天与数据中日期的巧合、甚至用一些统计假设检验来验证"反直觉"程度。这些逻辑写在 EDA Skill 里会让主文件膨胀，也会让其他三个子任务的指令被 Fun Facts 的细节所"淹没"。

解法是把它独立成 `/mnt/skills/user/fun-facts-miner/SKILL.md`：

```Plain
fun-facts-miner/
├── SKILL.md                     # 主指令：Fun Facts 挖掘工作流
└── references/
    └── statistical-tests.md     # 参考：可用的统计检验方法
```

主 Skill 里关于 Fun Facts 的指令就只有这几行：

```Markdown
#### Sub-task D: Fun Facts Mining
Spawn a new sub-agent, and use `/fun-facts-miner` to execute.  Pass the following context to the sub-agent:
- The loaded DataFrame (saved as a parquet file at a known path) 
- The data quality summary from Phase 1 
- The target: produce 5–8 fun facts, each citing a specific number

The sub-agent will return a structured Markdown section that can be directly embedded into the final report.
```

而其他的细节——如用哪些统计检验、怎么判断“反直觉”、输出格式的完整规范——都封装在 `fun-facts-miner` 的 `SKILL.md` 中了。

这个模式的关键语法是 `use /`_`{skill-name}`_——它告诉 Agent 运行时在 spawn 子 Agent 时去加载指定的 Skill 文件，而不是使用内联指令。这就像人类团队中的一种分工约定：主管说“这个子项目交给张工，按他自己的 SOP 来做”，而不是“这个子项目交给张工，具体步骤是第一步……第二步……”。

当然，并非所有子任务都值得独立成 Skill。笔者的判断标准是：**如果一个子任务的指令超过 100 行，或者它在其他场景中也有独立使用的价值，就值得拆出去。**

不仅如此，如果你使用了 Agent Team，也可以使用上述方法，来提前设计需要哪些 Teammates。当然，你也可以将分工的事情留给 Agent 临时决定，不过这可能需要在运行时始终使用更高级的模型（如 Claude Opus 4.6）。

>**第一，在 Skill 的元数据中显式声明并行能力。** 在 `SKILL.md` 的 Frontmatter 里，把 `Task`、`Agent` 或你的框架中对应的并行工具列入 `allowed-tools`。这是给模型的**“许可信号”**——如果你不显式声明，一些模型有可能不会主动使用并行。在 `SKILL.md` 中，也要使用**“spawn”**这样的动作词来触发模型的并行意识。
    第二，为每个子任务提供足够具体的指令。** Anthropic 踩过的坑值得所有人引以为戒：模糊的子任务描述会导致重复工作。好的子任务描述应包含四要素——**目标**（做什么）、**边界**（不做什么）、**输出格式**（返回什么）、**工具指导**（用什么工具）。
    **第三，设计好 fan-out / fan-in 的接口。** **fan-out** 是 Lead Agent 如何把任务分发给子 Agent；**fan-in** 则是子 Agent 如何把结果返回给 Lead Agent。两者都需要明确的**数据契约**。在笔者的 EDA Skill 中，每个子任务的输出规范是“至少 2 个 Mermaid 图表 + 文字洞察”，Lead Agent 在 Phase 3 会按照固定的报告模板把这些素材组装成最终报告。
