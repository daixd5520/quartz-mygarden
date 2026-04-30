# 引言

最近工作和生活里听到大量 Harness Engineering 的讨论，对这个概念怎么真正落到 AI Coding 上一直没完全理清楚，所以打算做一个系列——从"什么是 Harness Engineering"一路聊到"AI Coding 方向上怎么搭一个工程级可用的 Harness"。

这是第一篇，主要做概念和哲学的梳理：市面上已有的资料加上一些个人的整理，说清楚 Harness Engineering 是什么、主流家都在做什么取舍。

# 什么是 Harness Engineering

**在 agent 系统里，除模型本身之外的所有代码、配置、执行逻辑统称为 Harness Engineering。** 它是一门工程学科，研究怎么设计环境、约束、反馈回路和基础设施，让 AI Agent 在规模化场景里可靠运行。

有个直观的比喻：把模型比作马，它本身很有力量，但光有马到不了目的地——人还要准备缰绳、搭桥、铺跑道。Agent 里模型是那匹马，Harness Engineering 就是围栏、缰绳、跑道的集合。本文后面多处会借用这个比喻。

![[Tech--OpenAI｜Anthropic｜Cursor｜LangChain 眼中的Harness Engineering-1.png]]

# Harness ｜Context｜Prompt

Harness Engineering、Context Engineering、Prompt Engineering 三者是**同心圆的包含关系**：Prompt Engineering ⊂ Context Engineering ⊂ Harness Engineering。从内到外作用范围逐层扩大、确定性逐层增强、杠杆点也从"措辞"迁移到"系统"。

![[Tech--OpenAI｜Anthropic｜Cursor｜LangChain 眼中的Harness Engineering-2.png]]

**三者的本质区别**

|       |                       |                                         |                                              |
| ----- | --------------------- | --------------------------------------- | -------------------------------------------- |
| 维度    | Prompt Engineering    | Context Engineering                     | Harness Engineering                          |
| 一句话定义 | 你对模型**说的话**           | 模型在推理时**看到的一切**                         | 模型之外的**整个系统**                                |
| 包含什么  | 系统提示、用户指令、Few-shot 示例 | Prompt + 工具描述 + 检索文档 + 对话历史 + 记忆 + 压缩策略 | Context + 沙箱 + CI + 中间件 + 可观测性 + 编排逻辑 + 文件系统 |
| 核心问题  | 怎么**措辞**让模型理解意图       | 在有限窗口里放**哪些 token**                     | 怎么让模型的智能**可靠地产出价值**                          |
| 作用范围  | 单次模型调用                | 单次推理的完整输入                               | 跨会话、跨 Agent 的完整系统                            |
| 确定性   | 低——模型可能不遵循指令          | 中——信息到位但模型仍可能偏离                         | 高——包含确定性执行逻辑（CI 不过就不合并、中间件拦截退出）              |
| 模型可见性 | 完全可见                  | 完全可见                                    | **部分不可见**（沙箱隔离、中间件拦截、CI 管线对模型透明）             |

**一个例子贯穿三层**

以"禁止 Agent 使用已废弃的 API"为例：

- **Prompt Engineering**：在系统提示里写 "请不要使用 legacyFetch()，使用 modernFetch()"。问题是模型可能忽略，尤其长上下文中
- **Context Engineering**：在工具描述里移除 legacyFetch 的定义，只暴露 modernFetch；在 AGENTS.md 索引中指向 `docs/api-migration.md`，让模型按需查阅。问题是模型可能通过 Bash 自己调 legacyFetch
- **Harness Engineering**：一条 ESLint 规则禁止 legacyFetch + CI 管线强制检查 + 沙箱网络层拦截对废弃端点的请求。违规代码根本无法通过，不依赖模型是否"听话"

**杠杆点**

|   |   |   |   |
|---|---|---|---|
|层级|杠杆特征|适用场景|天花板|
|Prompt Engineering|成本最低、见效最快，但天花板低|快速调优单个行为，适合早期探索和简单任务|无法增加能力，只能在模型已有能力内调整表现|
|Context Engineering|中等投入，决定模型"看到什么世界"|正确的信息在正确的时机出现，戏剧性改变输出质量（OpenAI："给地图不给说明书"）|信息到位但模型仍可能偏离方向或自我欺骗|
|Harness Engineering|投入最大，但能根本性改变可能性边界|强制验证、安全隔离、多代理协调、长时任务纠偏、可观测性|LangChain 证明：同一模型仅改 Harness 从 Top 30 → Top 5|

**杠杆方向上的本质差异：** Prompt Engineering 的杠杆在于**措辞的精准性**（"禁止 TODO" vs "请记得完成"）；Context Engineering 的杠杆在于**信息的策展**——用最小的高信号 token 集合最大化所需结果的可能性；Harness Engineering 的杠杆在于**确定性的保障**——模型不遵循指令时系统兜底。

用文档里马的比喻来说：Prompt 是你对马说的话，Context 是马能看到的路况，Harness 是缰绳、围栏和跑道本身。你可以对马喊"往左"，但一条修好的路比任何指令都有效。

# 市面上的讨论

真正把 Harness Engineering 这个词推到台面上的是 OpenAI、Cursor、Anthropic——它们在 agent-first 的软件开发实践里系统性地摸索了"模型之外"所有工程方向的价值。我们今天看到的 Codex、Claude Code、Cursor 本质上就是当下最优秀的 Harness Engineering 范式。

但三家关心的具体问题并不一致：

- **OpenAI**：侧重**环境设计**——让 agent 在一个由人精心设计的环境下可靠产出代码。重点是搭文件体系、约束体系、测试基建（相当于搭桥铺路）
- **Cursor**：侧重**多智能体编排**——让多个 agent 同时工作，关心分工、并行、集成的艺术（相当于马车和流动运输线）
- **Anthropic**：最贴近模型——关心怎么把模型用到极致，怎么提供工具、怎么在 agent 长时过程中保持方向和质量（相当于马缰和驯马）

方向不同，三家演变出了一些共通的共识和一些各自独特的 Harness 设计艺术。

## 案例与理念

### 3.1 Anthropic：长时任务的方向与质量

Anthropic 要解决的核心问题是：一个 agent 在被精心设计的环境里开始工作之后，怎么在几个小时的连续运行中保持方向和质量。

它们在文章里先定义了两个问题：

- **上下文焦虑**：随着上下文窗口逐渐变满，模型的一致性开始衰减，压缩过的上下文往往偏离原定方向
- **自我欺骗**：agent 能发现自己产出里的缺陷，但随后会说服自己这些缺陷可以接受，给出通过判断

解法是一个受 GAN 启发的三角色架构，核心思路是**将生成与评估分离**，通过独立的角色和信息隔离解决自评偏差和方向漂移：

- **Planner — 需求翻译器**：输入是用户的 1-4 句话的简短需求，输出是包含 10+ 个 sprint 工作量的详细产品规格。只负责"做什么"（产品层面和高层技术方向），不进入"怎么做"的实现细节。是整个流程的起点，运行一次后退出，不参与后续 sprint 迭代循环。
- **Generator — 实现者**：按 Planner 产出的 spec 逐个 sprint 实现功能，技术栈固定为 React + Vite（前端）、FastAPI + SQLite/PostgreSQL（后端）。每个 sprint 开始前与 Evaluator 协商 Sprint Contract — 双方就"什么算完成"达成可验证的共识。Generator 只需关心 contract 里的验收标准，不需要知道 Evaluator 内部怎么测试。
- **Evaluator — 独立验证者**：不看 Generator 的代码，不共享任何内部状态，通过 Playwright MCP 操作真实运行的应用来验证 — 点击按钮、填表单、检查页面渲染，和真实用户一样。验证依据是事先协商的 Sprint Contract，不是临时发明标准。前端设计场景下使用四个评分维度：Design Quality（美学连贯性）、Originality（自主设计决策 vs AI 通用风格）、Craft（层次/间距/色彩和谐的技术执行）、Functionality（可用性和任务完成度），单个生成周期跑 5-15 轮迭代。
- **信息隔离**：Evaluator 和 Generator 之间没有共享的内部状态。Sprint Contract 是唯一的桥梁 — 正如 GAN 中生成器和判别器通过输出（而非内部状态）耦合。每个角色有独立的上下文窗口，通过产物（spec、contract、运行中的应用）传递信息，避免三个角色挤在同一个上下文里导致的连贯性衰减。Evaluator 走用户路径做黑盒测试，不检查代码实现细节，避免与 Generator 在实现层面产生纠缠。

**交互拓扑：**

![[Tech--OpenAI｜Anthropic｜Cursor｜LangChain 眼中的Harness Engineering-3.png]]

**从这个案例可以抽出两条长时任务的 Harness 设计思路：**

1. **跟着模型能力边界走**：设计重心要从模型已经擅长的地方移走，放到模型仍然薄弱的地方。
   - **Sprint 机制的移除**：Opus 4.5 时代，模型在长任务中容易因为 context 焦虑偏离方向，所以需要把工作切成 10+ 个 sprint，每个 sprint 有明确的 contract，Generator 和 Evaluator 逐个 sprint 协商、实现、验证。本质上是用外部结构弥补模型"长程规划和执行"的不足。Opus 4.6 规划能力和长任务执行能力显著增强，Sprint 就不再必要，作者直接移除了整个 Sprint 构造——模型自己能管节奏了。
   - **多轮 CR 变为单轮**：4.5 上代码质量需要多轮 Loop 保证，4.6 直接简化为单轮。
2. **基于 GAN 的"生成器-评估器"对抗模式**：Generator 和 Evaluator 信息隔离，执行者和审查者保持独立，是长时任务纠偏的关键。

### 3.2 Cursor：如何让 Multi-Agent 协同工作

Cursor 要回答的核心问题是：能否通过投入 N 倍算力换 N 倍有效吞吐量？从 2026 年 1 月到 3 月，他们通过一系列文章记录了这一探索的完整过程，包括观点的演进和自我修正。

|   |   |   |
|---|---|---|
|文章|核心尝试|核心结论/原则|
|Scaling Long-Running Autonomous Coding — Wilson Lin<br><br>Towards Self-Driving Codebases — Wilson Lin|- **Planner-Worker 层级架构**：Planner持续探索代码库并创建任务，支持递归子 Planner；Worker专注执行，无需跨 Worker 协调；评审 Agent评估是否需要继续迭代<br>    <br>- **Planner引入 Handoff 机制**：Worker 完成后写一份交接文档（做了什么、发现了什么、有什么担忧），提交给上级 Planner。这使 Planner能持续重新规划而无需全局同步——异步文件交换替代实时消息协调。|- "简洁优于复杂"——移除了造成瓶颈的集成者（Integrator）角色<br>    <br>- "提示工程（prompt）主导系统行为"<br>    <br>    - 提示工程的具体经验：约束优于指令（"禁止TODO"比"请记得完成所有实现"更有效<br>        <br>    - 具体数值优于模糊表述<br>        <br>- 三大设计原则：<br>    <br>    - 多agent并行时容忍个体失败<br>        <br>    - 数据驱动而非假设驱动、<br>        <br>    - 接受稳定低错误率能显著优化吞吐量<br>        <br>- 吞吐量的瓶颈不在代码，而是在Harness|
|Building a Better Bugbot — Jon Kaplan|- 在代码审查场景中，Cursor 探索了不同于 Planner-Worker的并行策略。Bugbot 每月审查超 200 万个PR，采用八路并行检测：对 diff 随机排序后并行执行八轮 bug查找，通过多数投票来确定是否需要修复。|- **放弃固定流水线、转向全 Agent架构是最大优化点，我的理解是放开流水线式的流程与决策，让多个agent充分发散后自主决策**|
|Implementing a Secure Sandbox for Local Agents — Ani, Yash &Alex<br><br>Fast Regex Search — Vicent Marti|- 沙盒环境替代软约束管理模型工作边界<br>    <br>- 信息检索搜索提速|- 工具决定 Agent上线<br>    <br>    - 特定任务提供沙盒来约束边界能更好的提升模型自主运行的效果<br>        <br>    - 模型不可见内容等于不存在，让模型能更快的拿到信息能显著提效|

### 3.3 OpenAI：Agent-First 时代里人和 Agent 怎么协作

OpenAI 的立场最直接：**文档是产品、linter 是杠杆、架构是前置条件，人类的工作是设计让 Agent 可靠运作的世界**。他们围绕这个判断做了一批尝试，有成有败：

|   |   |   |   |
|---|---|---|---|
|尝试|做法|结论|经验|
|巨型AGENTS.md|把所有规则、标准、约定塞进一个文件|失败|上下文是稀缺资源，过大的指令文件挤占了任务、代码和文档的空间，Agent 要么遗漏关键约束，要么开始优化错误的目标|
|每周五人工清理 AI slop|每周安排时间手动清理 Agent 生成的低质量代码|失败|不能扩展——代码产出速度远超人工清理速度|
|依赖人工审查维护架构一致性|靠 code review 人工把关架构规范|失败|Agent 速度太快，人工审查成为瓶颈；且同一类错误反复出现成功的尝试|
|AGENTS.md 渐进式披露|~100 行的精简索引，指向 docs/ 中的深层文档|成功|Agent 从小入口点开始，按需深入查找|
|自定义 Lint 规范编码"品味"|发现 Codex 重复创建并发辅助函数（只有一个版本正确连接OpenTelemetry），写了一条 ESLint 规则禁止在批准位置之外定义该函数|成功|一次编码，永久生效；规则本身也由 Codex 编写并有 100%测试覆盖|
|Slack 决策反映回代码库|安全库选择决策留在 Slack 导致新工程师引错包；团队在Slack中 @codex please add guardrails，15 分钟生成 4 个 PR|成功|决策从不可见变为可见且可强制执行|
|自动化垃圾回收|后台 Agent 任务定期扫描模式偏移，开启针对性清理PR（设计为 1 分钟内可审查）|成功|自动的代码清理解决了agent Coding质量监管问题|
|人的能力的复利|前端专家入职后编码 React 组件架构知识效果: 所有 Agent 立即开始分hooks、创建小型可测试组件|成功|人的经验和知识对于所有agent可复利|
|重写而非复用|用模型去覆盖一些成熟能力，不引入非透明的库而是重写需要的原子能力|成功|agent自己实现的原子能力，Agent 理解更好，生成质量更高|
|拥抱AI的快速变更|增加站会频率，每天 30 分钟站会|成功|反直觉但必要——代码变更速度极快，"可能几周后才意识到核心架构模式已经变了"|

从这一堆尝试里可以抽出 OpenAI 工程团队关于 agent 协作的几条核心判断：

|   |   |
|---|---|
|理念|解读|
|"Code becomes inexpensive (agents produce it at scale), while environmental quality becomes expensive."|工程师的杠杆点从"把想法变成代码"转向"设计让 Agent 能可靠工作的条件"。|
|"Documentation has always been good hygiene. Now it's the raw material making agents effective."|文档是让 Agent 有效工作的原材料，精准+渐进式披露的知识文档体系能很好的帮助agent完成复杂工作|
|"From the agent's point of view, anything it can't access in-context while running effectively doesn't exist."|Google Docs 里的决策、Slack 线程中的讨论、人脑中的隐含知识——对Agent 全部不可访问，等于从未发生。所有重要知识必须物化为仓库中的可访问文件。|
|"Enforce invariants rather than micromanaging implementation details."|不是告诉 Agent "做 A、不做B"，而是在环境中嵌入结构性约束（linter、CI 测试、分层架构），使违规代码根本无法通过。约束是确定性的、可行的；指令是模糊的、可被忽略的。|
|"If you can articulate what it is about the code you don't like, the next step is to write that down."|人类的核心价值在于审美判断力——识别什么是好的代码模式。但这种判断力必须被显式编码（lint规则、文档、示例），否则它只存在于人脑中，Agent 无法受益。|
|更好的 Harness → Agent 产出更好 → 暴露新的模式偏移→ 编码新的约束 → Harness 再次改善 → ..|Harness不是一成不变的，他会跟着模型能力的发展重心发生迁移|

OpenAI 在文章里画出了自己的 Harness 架构，概括成三条主轴：

![[Tech--OpenAI｜Anthropic｜Cursor｜LangChain 眼中的Harness Engineering-4.png]]

**上下文工程（Context Engineering）——给地图，不给说明书。** 上下文是稀缺资源，塞得越满越容易失焦。

- **AGENTS.md 作为索引**：~100行精简文件，作为地图指向 docs/ 中的深层文档，实现渐进式披露（Progressive Disclosure）——Agent 从小入口点开始，按需深入查找
- **docs/ 结构化目录**：架构、设计、决策记录（ADR）、标准、参考资料分目录存放，每个目录有明确职责
- **决策回流仓库**：架构决策从 Slack 讨论转移到版本化的仓库产物中。Ryan 的案例：安全库选择决策留在 Slack 导致新工程师引错包，团队通过 `@codex please add guardrails` 15分钟生成4个PR将决策编码为护栏
- **选择无聊技术栈**：训练数据中覆盖度高的成熟技术，Agent 理解更好；必要时重新实现功能而非集成不透明的黑盒库

**架构约束强制（Architectural Constraint Enforcement）——强制不变量，不是微管理实现。** 不要靠指令告诉 agent 做 A 不做 B，要靠结构让违规代码根本过不去。

- **严格分层架构**：固定层级顺序 Types→Config→Repo→Service→Runtime→UI，依赖方向不可逆转。如同仓库需要清晰标记的货架——人类靠经验找路，Agent 需要明确的结构
- **自定义 Lint 规则**：将品味编码为自动化约束。案例：Codex 反复创建重复的并发辅助函数，只有一个版本正确连接了 OpenTelemetry，团队编写 ESLint 规则禁止在批准位置之外定义该函数（规则本身由 Codex 编写，100% 测试覆盖）
- **结构化 CI 测试**：机械式强制执行架构规范，违规代码根本无法通过——约束是确定性的、可执行的，指令是模糊的、可忽略的

**持续垃圾回收（Continuous Garbage Collection）——用自动化取代人工清理。** 团队最早搞过每周五手动清 AI slop 的冲刺，结果很快发现清理速度追不上产出速度，只能转成常态化自动化：

- **后台 Agent 定时扫描**：自动检测模式偏移（违反黄金原则的代码），开启针对性重构 PR
- **PR 设计为 <1min 审查**：快速合并清理，替代不可扩展的人工清理冲刺
- **质量评分持续跟踪**：更新代码健康度，防止技术债在 Agent 高速产出中悄然积累

这三条的背后有一个复利循环：**agent 卡住 → 人诊断到底是缺文档、缺约束还是缺工具 → 补进 Harness → 整个团队所有 agent 受益。** 一个人编码的专长被整支 agent 舰队复用。这也解释了为什么 OpenAI 反复强调"代码越快、人类沟通越关键"——每天 30 分钟站会不是形式主义，是代码变更速度推着组织节奏走。

### 3.4 LangChain：模型提供智能，Harness 让智能变得有用

LangChain 想回答的是一个更"实验性"的问题：**模型不换，只动 Harness，能带来多大的性能提升？** 他们在两篇博文里给出了答案——《The Anatomy of an Agent Harness》做解剖学定义和六大组件架构，《Improving Deep Agents with Harness Engineering》在 Terminal Bench 2.0 上做实战验证。

LangChain 给 Harness 的定义是这四家里最干脆的：**除模型本身以外的所有代码、配置和执行逻辑**。模型出智能，Harness 让这种智能变得可用。

**六大组件架构**

|           |             |                                                                                               |
| --------- | ----------- | --------------------------------------------------------------------------------------------- |
| 组件        | 核心作用        | 关键机制                                                                                          |
| 文件系统      | 持久化存储与协作表面  | Agent 工作区、跨会话状态持久化、Git 集成实现版本控制与回滚、多代理和人机协作的共享表面                                              |
| Bash/代码执行 | 动态工具创建      | 给 Agent 通用代码执行能力而非固定工具集，让 Agent "通过代码即时设计自己的工具"，不受预配置工具的约束                                    |
| 沙箱        | 安全隔离执行      | OS 级隔离、命令白名单、网络隔离、按需环境供应、自验证循环（写代码→跑测试→查日志→修错误）                                               |
| 记忆与搜索     | 突破知识时效性边界   | AGENTS.md 等文件系统记忆标准实现持续学习、Web 搜索和 MCP 工具补充训练截止后的知识、上下文注入让 Agent 在未来会话中可访问更新的知识                |
| 上下文腐化管理   | 对抗长上下文推理退化  | 压缩（Compaction）：智能卸载并总结上下文；工具调用卸载：保留头尾 token，完整输出存文件系统；技能渐进式加载（Skills）：防止启动时加载过多工具导致性能退化       |
| 长周期自主执行   | 支撑跨多窗口的复杂任务 | Git 跟踪捕获百万 token 级的工作进度；Ralph Loop 拦截模型退出尝试，在干净上下文窗口中重新注入原始提示强制继续；规划与自验证让 Agent 分解目标并通过测试套件校验 |

**Terminal Bench 2.0 上的四条优化策略**

LangChain 在 Terminal Bench 2.0（89 个跨 ML、调试、生物学的任务）上，用 Harbor 框架 + Daytona 沙箱 + LangSmith 可观测平台，只调了三个变量（系统提示、工具、中间件），就拿到了 13.7% 的提升。四条策略分别对应四类常见失败模式：

|   |   |   |   |
|---|---|---|---|
|策略|问题|解法|关键机制|
|构建与自验证循环|Agent 写完代码就停止，不运行测试就宣布完成|四阶段强制流程 + 退出拦截|Planning→Build→Verify→Fix 的结构化引导；PreCompletionChecklistMiddleware 在 Agent 退出前强制插入验证提醒（类似 Ralph Loop 模式）|
|环境上下文主动交付|Agent 在陌生环境中浪费大量时间探索|启动时主动注入环境信息|LocalContextMiddleware 在启动时映射目录结构和可用工具；注入时间预算提醒引导 Agent 合理分配精力；强调代码将面临程序化测试，需严格遵循规范|
|防止短视行为（Doom Loop）|Agent 陷入同一方案的 10+ 次无效重试|编辑频次监控 + 策略切换提醒|LoopDetectionMiddleware 跟踪每个文件的编辑次数，超过 N 次后建议"重新考虑你的方法"。属于临时性工程方案，随模型进步将逐步废弃|
|推理计算分配（Reasoning Sandwich）|全程高推理导致超时（得分降至 53.9%），全程低推理质量不足|分阶段差异化分配推理强度|规划阶段 XHigh → 实现阶段 High → 验证阶段 XHigh，形成"三明治"结构。未来方向：模型自适应推理（Claude 和 Gemini 已实现），由模型自行决定计算分配|

**Trace 驱动的迭代优化**

LangChain 把 Harness 优化类比成 ML 里的 boosting：LangSmith 捕获每次 agent 执行的完整 trace（延迟、token、成本、每步动作），再用自动化 Trace Analyzer 并行分析失败案例、聚合反馈、针对性改 Harness。核心立场是**不靠直觉，靠数据和失败模式驱动改进**。

**模型-Harness 协同进化是双刃剑**

LangChain 观察到一个反直觉的现象：模型会对特定 Harness 实现产生过拟合——随便改一下工具逻辑，性能可能下降，因为模型后训练里已经适应了当初的 Harness 形状。但反过来，**最优 Harness 不一定是模型训练时用的那个**——Claude Opus 4.6 换到非 Claude Code 的 Harness 里，经过调优反而从 Top 30 冲进了 Top 5。**Harness 工程有独立于模型训练的优化空间**，这是 LangChain 给整个讨论最重的一块砝码。

**LangChain 补上了前三家的盲区：**

1. **Harness 是独立于模型的性能杠杆**：与 Anthropic 关注长时任务质量、Cursor 关注多代理编排、OpenAI 关注环境设计不同，LangChain 用实验数据证明了 Harness 本身作为独立变量的价值——同一模型，仅改 Harness 就能从 Top 30 进入 Top 5
2. **中间件是 Harness 的核心抽象**：LangChain 将 Harness 的可调节点抽象为三个"旋钮"（系统提示、工具、中间件），其中中间件（Middleware/Hooks）是最具工程价值的部分——它在模型调用前后插入确定性逻辑（如 PreCompletionChecklist、LoopDetection、LocalContext），将模型的"指令遵循"问题转化为"代码强制执行"问题，这与 OpenAI 的"约束优于指令"理念一脉相承
3. **Harness 的护栏应设计为可废弃的**：LoopDetectionMiddleware 等机制是为当前模型缺陷设计的临时方案。LangChain 明确指出这些护栏会随模型进步而废弃，但现在它们是构建可靠系统的必要条件。这呼应了 Anthropic 在 Opus 4.5→4.6 迭代中移除 Sprint 机制的实践——**好的 Harness 设计师会跟着模型能力的边界移动自己的重心**
## 通识

四家方向不同，但有一套高度重合的共识浮出水面：

**人的核心工作从写代码转向设计 agent 的工作环境。** 这是最大的共识，OpenAI 的原话是"Code becomes inexpensive"。人类的杠杆点在于创造让 agent 能可靠工作的条件，代码本身由 agent 产出。我们的工作也应该从"把想法变成代码"迁移到"设计 agent 的工作环境，让它更快更好更稳定地把想法变成代码"。

**模型看不到的等于不存在。** 要减少工作里模型看不到的部分（复杂流程、Docs 里的讨论、团队成员脑子里的隐含知识），让模型看到精准、正确的信息。

**Context not Control。** 约束应该是可执行的、确定性的；指令是可解释的、模糊的。"禁止 TODO"比"请记得完成所有实现"更有效，具体数值比模糊表述更有效。约束是边界，指令是建议。在 agent 的工作方式里，这个区别比在人类团队里更关键。

**纠错比等待廉价。** AI 能快速交付的场景下，快速试错的成本远低于完美规划。四家都在强调不纠结首次执行的完美，而是通过多轮快速试错让结果趋近完美。

**验证必须是强制的，不能是建议的。** 四家独立发现了同一个问题：agent 天然倾向于写完就停、自我说服"可以接受"。解法形式不同但本质一致——用确定性机制强制验证。Anthropic 用独立 Evaluator 角色 + 信息隔离，执行者不能给自己打分；Cursor 用 Review Agent 独立评估 Worker 产出；OpenAI 用 CI 测试 + Lint 规则，违规代码根本无法通过；LangChain 用 PreCompletionChecklistMiddleware 拦截退出，强制运行验证。"请记得测试"是无效的指令，"退出前必须通过测试"才是有效的约束。

**Harness 应设计为可进化、可废弃的。** Harness 不是一成不变的静态产物，而是必须随模型能力迁移的动态系统：Anthropic 从 Opus 4.5→4.6 直接移除 Sprint 机制、多轮 CR 简化为单轮；Cursor 持续演进架构、移除造成瓶颈的 Integrator 角色；OpenAI 通过复利循环不断编码新约束；LangChain 把 LoopDetectionMiddleware 明确标注为"临时方案，随模型进步废弃"。**好的 Harness 设计师把护栏设计为可拆卸的脚手架，而非永久性的承重墙——重心永远跟着模型能力的边界走。**

**可观测性是 Harness 优化的前提。** 你无法优化你看不见的东西。LangChain 通过 LangSmith 捕获完整 trace 并用 Trace Analyzer 自动分析失败模式（最系统化）；Cursor 把"数据驱动而非假设驱动"作为核心设计原则；OpenAI 用质量评分持续跟踪代码健康度；Anthropic 用 Sprint Contract 作为可观测的检查点，Evaluator 通过真实应用交互产生可审计的验证记录。Trace 和可观测性是 Harness 工程从"凭直觉调参"进化到"数据驱动改进"的基础设施。

**文件系统是 agent 世界的共享记忆。** 四家不约而同地把文件系统作为 agent 持久化和协作的核心原语：LangChain 把文件系统列为六大组件之首，作为跨会话状态和多代理协作的基础；Anthropic 用进度日志 + Git 记录贯穿长时任务的多个上下文窗口；OpenAI 用 AGENTS.md + `docs/` 把决策从 Slack 回流成可访问文件；Cursor 用 Handoff 文档实现异步协调，Worker 完成后写交接文档而不是实时消息。**文件系统不只是存储，它是 agent 的工作记忆、协作协议和时间机器（Git）**。所有需要跨会话、跨 agent 传递的信息，最终都会落回文件系统。

# 参考文献

OpenAI：

- https://openai.com/index/harness-engineering/

Cursor：

- https://cursor.com/cn/blog/self-driving-codebases
- https://cursor.com/cn/blog/scaling-agents

Anthropic：

- https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- https://www.anthropic.com/engineering/harness-design-long-running-apps

LangChain：

- https://blog.langchain.com/the-anatomy-of-an-agent-harness/
- https://blog.langchain.com/improving-deep-agents-with-harness-engineering/

其他：

- https://mp.weixin.qq.com/s/Q5oK_VtunXlouachrj1agQ
- https://yage.ai/share/harness-engineering-scalability-20260330.html
