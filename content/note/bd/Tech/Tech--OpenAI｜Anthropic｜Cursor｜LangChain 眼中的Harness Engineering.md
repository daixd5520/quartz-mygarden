# 引言

笔者苦于工作和生活中听到了大量Harness Engineering的讨论，而且对于如何通过Harness Engineering让AI在AI Coding领域发挥更好的作用这件事也不甚清晰，所以准备启动一个系列：

从什么是Harness Engineering聊到AI Coding 方向上我们如何搭建一个工程级别可用的Harness Engineering

这个文档是这个系列的第一章：主要从市面上搜集到的资料结合个人浅薄理解认知讲一下**什么是Harness Engineering？Harness Engineering有哪些设计的哲学？**

# 定义：什么是 Harness Engineering

想象模型（LLM）是一匹马，这匹马可以带你到达任何地方，但人总有目的地与其他的需求

所以，为了舒服，人会准备缰绳，为了更好的到达某个目的地，人会搭桥修路，这些基础设施不是马本身，但是他们可以让马更好更快的到达人的目的地

AI Agent 也是如此。模型（LLM）是那匹马——强大但未被驯化。Harness Engineering 就是搭围栏、做缰绳、铺跑道的工程学科。

Harness Engineering是设计环境、约束、反馈循环和基础设施以使 AI Agent 在规模化场景下可靠运行的工程学科，所以一个agent系统中，所有模型外的内容统称为Harness Engineering

![[Tech--OpenAI｜Anthropic｜Cursor｜LangChain 眼中的Harness Engineering-1.png]]

# Harness ｜Context｜Prompt

> 那么Harness Engineering与Context Engineering和Prompt Engineering 到底是什么关系呢

三者是**同心圆的包含关系**：Prompt Engineering ⊂ Context Engineering ⊂ Harness Engineering。从内到外，作用范围逐层扩大，确定性逐层增强，杠杆点也从"措辞"迁移到"系统"。

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

- **Prompt Engineering**：在系统提示里写 "请不要使用 legacyFetch()，使用 modernFetch()" → 问题：模型可能忽略，尤其在长上下文中
- **Context Engineering**：在工具描述中移除 legacyFetch 的定义，只暴露 modernFetch；在 AGENTS.md 索引中指向 docs/api-migration.md 让模型按需查阅 → 问题：模型可能通过 Bash 自己调用 legacyFetch
- **Harness Engineering**：写一条 ESLint 规则禁止 legacyFetch + CI 管线强制检查 + 沙箱网络层拦截对废弃端点的请求 → 效果：违规代码根本无法通过，不依赖模型是否"听话"

**杠杆点**

|   |   |   |   |
|---|---|---|---|
|层级|杠杆特征|适用场景|天花板|
|Prompt Engineering|成本最低、见效最快，但天花板低|快速调优单个行为，适合早期探索和简单任务|无法增加能力，只能在模型已有能力内调整表现|
|Context Engineering|中等投入，决定模型"看到什么世界"|正确的信息在正确的时机出现，戏剧性改变输出质量（OpenAI："给地图不给说明书"）|信息到位但模型仍可能偏离方向或自我欺骗|
|Harness Engineering|投入最大，但能根本性改变可能性边界|强制验证、安全隔离、多代理协调、长时任务纠偏、可观测性|LangChain 证明：同一模型仅改 Harness 从 Top 30 → Top 5|

**杠杆方向的本质差异：**

- **Prompt Engineering** 的杠杆在于**措辞的精准性**（"禁止 TODO" vs "请记得完成"）
- **Context Engineering** 的杠杆在于**信息的策展**（找到最小的高信号 token 集合，最大化所需结果的可能性）
- **Harness Engineering** 的杠杆在于**确定性的保障**（模型不遵循指令时，系统兜底）

>用文档里的马的比喻来说：Prompt 是你对马说的话，Context 是马能看到的路况，Harness 是缰绳、围栏和跑道本身。你可以对马喊"往左"，但一条修好的路比任何指令都有效。

# 市面上的讨论

>一起的源头：OpenAI、Cursor 和 Anthropic 通过自己在agent- first 软件开发的实践中逐步的总结了对于模型外自己所有的工程方向的探索对于大模型执行任务的效果提升，因此产生了Harness Engineering这个名词，目前我们看到的CodeX，ClaudeCode和Cursor本质上就是目前市面上最优秀的Harness Engineering的范式

OpenAI、Cursor 和 Anthropic三家对于harness engineering探索的方向并不完全一致，他们其实在解决不同的问题

OpenAI：主要探索的是环境的设计，让agent在一个由人提供的精心设计的环境下可靠的生成代码，所以他们主要讲述搭建文件体系，约束体系，测试基建等（有点像搭桥铺路的逻辑）

Cursor：主要探索的是多智能体的编排，如果让多个agent同时工作，所以他们主要讲的是multi-agent的编排，分工，并行，集成的艺术（有点像马车和流动运输线）

Anthropic：最贴近模型的思路，主要讲的是如何最大程度的运用模型，如何提供工具，如何在一个 agent 长时过程中保持方向和质量。（有点像马缰和驯马）

三家的方向不同，所以在演变过程中出现了一些通识和一些自己的独特的 harness engineering 设计艺术

## 案例与理念

### 3.1 Anthropic ：长时任务的方向与质量

> Anthropic 要解决的问题是：一个 agent 在被精心设计的环境里开始工作之后，怎么在几个小时的连续运行中保持方向和质量

首先定义问题，Anthropic 的文章里提出：

1. 上下文焦虑：随着上下文窗口逐渐变满，模型的一致性开始衰减，压缩过的上下文往往会表现为偏离原定的方向
2. 自我欺骗：agent 在工作过程中能发现自己产出里的缺陷，但随后会说服自己这些缺陷可以接受，给出通过判断

Anthropic 的解法是一个受 GAN（生成对抗网络）启发的三角色架构，核心思路是将生成与评估分离，通过独立的角色和信息隔离来解决自评偏差和方向漂移：

- **Planner — 需求翻译器**：输入是用户的 1-4 句话的简短需求，输出是包含 10+ 个 sprint 工作量的详细产品规格。只负责"做什么"（产品层面和高层技术方向），不进入"怎么做"的实现细节。是整个流程的起点，运行一次后退出，不参与后续 sprint 迭代循环。
- **Generator — 实现者**：按 Planner 产出的 spec 逐个 sprint 实现功能，技术栈固定为 React + Vite（前端）、FastAPI + SQLite/PostgreSQL（后端）。每个 sprint 开始前与 Evaluator 协商 Sprint Contract — 双方就"什么算完成"达成可验证的共识。Generator 只需关心 contract 里的验收标准，不需要知道 Evaluator 内部怎么测试。
- **Evaluator — 独立验证者**：不看 Generator 的代码，不共享任何内部状态，通过 Playwright MCP 操作真实运行的应用来验证 — 点击按钮、填表单、检查页面渲染，和真实用户一样。验证依据是事先协商的 Sprint Contract，不是临时发明标准。前端设计场景下使用四个评分维度：Design Quality（美学连贯性）、Originality（自主设计决策 vs AI 通用风格）、Craft（层次/间距/色彩和谐的技术执行）、Functionality（可用性和任务完成度），单个生成周期跑 5-15 轮迭代。
- **信息隔离**：Evaluator 和 Generator 之间没有共享的内部状态。Sprint Contract 是唯一的桥梁 — 正如 GAN 中生成器和判别器通过输出（而非内部状态）耦合。每个角色有独立的上下文窗口，通过产物（spec、contract、运行中的应用）传递信息，避免三个角色挤在同一个上下文里导致的连贯性衰减。Evaluator 走用户路径做黑盒测试，不检查代码实现细节，避免与 Generator 在实现层面产生纠缠。

**交互拓扑：**

![[Tech--OpenAI｜Anthropic｜Cursor｜LangChain 眼中的Harness Engineering-3.png]]

**透过现象看本质，这个例子里提到了两个长时任务的Harness设计思路：**

1. 解决模型的现有能力的边界问题：随着模型的迭代，设计重心需要跟着调整,从模型已经擅长的地方移走，放到模型仍然薄弱的地方。
    1. 例如：Sprint 机制的移除：Opus 4.5 上：模型在长任务中容易因为context焦虑问题导致偏离方向，所以需要把工作切成 10+ 个 sprint，每个 sprint有明确的 contract，Generator 和 Evaluator 逐个 sprint协商、实现、验证。这本质上是用外部结构弥补模型"长程规划和执行"的不足。而在Opus 4.6 时代：模型的规划能力和长任务执行能力显著增强，Sprint 不再是必要的内容，所以作者直接移除了整个 Sprint 构造。模型自己能管理好工作节奏了。
    2. 例如：多轮CR变为单轮：Opus 4.5 上：代码质量需要多轮Loop来保证，在4.6上简化为单轮
2. 基于GAN（生成对抗网络）的"生成器-评估器"对抗模式：Generator和Evaluator信息隔离，执行者和审查者保持独立性是长时任务纠偏的关键。

### 3.2 Cursor：如何让 Multi-Agent 协同工作

Cursor 要回答的核心问题是：能否通过投入N倍算力来获得N倍有效的吞吐量？ 从 2026 年 1 月到 3 月，他们通过一系列文章记录了这一探索的完整过程——包括观点的演进和自我修正。

|   |   |   |
|---|---|---|
|文章|核心尝试|核心结论/原则|
|Scaling Long-Running Autonomous Coding — Wilson Lin<br><br>Towards Self-Driving Codebases — Wilson Lin|- **Planner-Worker 层级架构**：Planner持续探索代码库并创建任务，支持递归子 Planner；Worker专注执行，无需跨 Worker 协调；评审 Agent评估是否需要继续迭代<br>    <br>- **Planner引入 Handoff 机制**：Worker 完成后写一份交接文档（做了什么、发现了什么、有什么担忧），提交给上级 Planner。这使 Planner能持续重新规划而无需全局同步——异步文件交换替代实时消息协调。|- "简洁优于复杂"——移除了造成瓶颈的集成者（Integrator）角色<br>    <br>- "提示工程（prompt）主导系统行为"<br>    <br>    - 提示工程的具体经验：约束优于指令（"禁止TODO"比"请记得完成所有实现"更有效<br>        <br>    - 具体数值优于模糊表述<br>        <br>- 三大设计原则：<br>    <br>    - 多agent并行时容忍个体失败<br>        <br>    - 数据驱动而非假设驱动、<br>        <br>    - 接受稳定低错误率能显著优化吞吐量<br>        <br>- 吞吐量的瓶颈不在代码，而是在Harness|
|Building a Better Bugbot — Jon Kaplan|- 在代码审查场景中，Cursor 探索了不同于 Planner-Worker的并行策略。Bugbot 每月审查超 200 万个PR，采用八路并行检测：对 diff 随机排序后并行执行八轮 bug查找，通过多数投票来确定是否需要修复。|- **放弃固定流水线、转向全 Agent架构是最大优化点，我的理解是放开流水线式的流程与决策，让多个agent充分发散后自主决策**|
|Implementing a Secure Sandbox for Local Agents — Ani, Yash &Alex<br><br>Fast Regex Search — Vicent Marti|- 沙盒环境替代软约束管理模型工作边界<br>    <br>- 信息检索搜索提速|- 工具决定 Agent上线<br>    <br>    - 特定任务提供沙盒来约束边界能更好的提升模型自主运行的效果<br>        <br>    - 模型不可见内容等于不存在，让模型能更快的拿到信息能显著提效|

### 3.3 OpenAI：Agent-First 时代人和agent如何用Harness Engineering协作

OpenAI是最直接的指明：文档是产品，linter是杠杆，架构是前置条件，人类的工作是设计让 Agent可靠运作的世界。在OpenAI的视角

为此他们做了大量的尝试

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

从这些尝试里透露出OpenAI工程团队在agent协作上逐步完善的设计理念

|   |   |
|---|---|
|理念|解读|
|"Code becomes inexpensive (agents produce it at scale), while environmental quality becomes expensive."|工程师的杠杆点从"把想法变成代码"转向"设计让 Agent 能可靠工作的条件"。|
|"Documentation has always been good hygiene. Now it's the raw material making agents effective."|文档是让 Agent 有效工作的原材料，精准+渐进式披露的知识文档体系能很好的帮助agent完成复杂工作|
|"From the agent's point of view, anything it can't access in-context while running effectively doesn't exist."|Google Docs 里的决策、Slack 线程中的讨论、人脑中的隐含知识——对Agent 全部不可访问，等于从未发生。所有重要知识必须物化为仓库中的可访问文件。|
|"Enforce invariants rather than micromanaging implementation details."|不是告诉 Agent "做 A、不做B"，而是在环境中嵌入结构性约束（linter、CI 测试、分层架构），使违规代码根本无法通过。约束是确定性的、可行的；指令是模糊的、可被忽略的。|
|"If you can articulate what it is about the code you don't like, the next step is to write that down."|人类的核心价值在于审美判断力——识别什么是好的代码模式。但这种判断力必须被显式编码（lint规则、文档、示例），否则它只存在于人脑中，Agent 无法受益。|
|更好的 Harness → Agent 产出更好 → 暴露新的模式偏移→ 编码新的约束 → Harness 再次改善 → ..|Harness不是一成不变的，他会跟着模型能力的发展重心发生迁移|

OpenAI也给出了自己的Harness的架构：

> Harness = 三大支柱

![[Tech--OpenAI｜Anthropic｜Cursor｜LangChain 眼中的Harness Engineering-4.png]]

**支柱一：上下文工程（Context Engineering）**核心原则：「Give Codex a map, not a 1,000-page instruction manual. Context is a scarce resource.」（给 Codex 一张地图，而非一本一千页的说明书。上下文是稀缺资源。）

- **AGENTS.md 作为索引**：~100行精简文件，作为地图指向 docs/ 中的深层文档，实现渐进式披露（Progressive Disclosure）——Agent 从小入口点开始，按需深入查找
- **docs/ 结构化目录**：架构、设计、决策记录（ADR）、标准、参考资料分目录存放，每个目录有明确职责
- **决策回流仓库**：架构决策从 Slack 讨论转移到版本化的仓库产物中。Ryan 的案例：安全库选择决策留在 Slack 导致新工程师引错包，团队通过 `@codex please add guardrails` 15分钟生成4个PR将决策编码为护栏
- **选择无聊技术栈**：训练数据中覆盖度高的成熟技术，Agent 理解更好；必要时重新实现功能而非集成不透明的黑盒库

**支柱二：架构约束强制执行（Architectural Constraint Enforcement）**核心原则：「Enforce invariants rather than micromanaging implementation details.」（强制不变量，而非微管理实现细节。）

- **严格分层架构**：固定层级顺序 Types→Config→Repo→Service→Runtime→UI，依赖方向不可逆转。如同仓库需要清晰标记的货架——人类靠经验找路，Agent 需要明确的结构
- **自定义 Lint 规则**：将品味编码为自动化约束。案例：Codex 反复创建重复的并发辅助函数，只有一个版本正确连接了 OpenTelemetry，团队编写 ESLint 规则禁止在批准位置之外定义该函数（规则本身由 Codex 编写，100% 测试覆盖）
- **结构化 CI 测试**：机械式强制执行架构规范，违规代码根本无法通过——约束是确定性的、可执行的，指令是模糊的、可忽略的

**支柱三：持续垃圾回收（Continuous Garbage Collection）**核心原则：不用人工清理冲刺，用自动化持续清理（团队最初每周五手动清理 AI slop，发现不可扩展后转为此方案）

- **后台 Agent 定时扫描**：自动检测模式偏移（违反黄金原则的代码），开启针对性重构 PR
- **PR 设计为 <1min 审查**：快速合并清理，替代不可扩展的人工清理冲刺
- **质量评分持续跟踪**：更新代码健康度，防止技术债在 Agent 高速产出中悄然积累

**侧栏：复利循环与人类角色**

- **复利循环**：Agent 卡住 → 人类诊断缺失（缺文档？缺约束？缺工具？）→ 补充到 Harness → 所有 Agent 受益。每位工程师编码的专长都乘以整个团队的 Agent 舰队效能
- **人类角色转变**：从代码编写者变为环境设计师。核心工作是设计环境、编码品味、增加站会频率（反直觉：代码越快 → 人类沟通越关键）

### 3.4 LangChain：**模型提供智能，Harness 让智能变得有用**

LangChain 在探索的更多的是：不换模型，仅通过优化 Harness 能带来多大的性能提升？

LangChain 通过两篇文章系统性地阐述了 Harness Engineering 的理论框架和实践方法。第一篇《The Anatomy of an Agent Harness》给出了 Harness 的解剖学定义和六大组件架构；第二篇《Improving Deep Agents with Harness Engineering》通过 Terminal Bench 2.0 的实战验证了 Harness 优化的具体策略和效果。

对于Harness Engineering,LangChain 给出了最直接的定义：Harness 是"除模型本身以外的所有代码、配置和执行逻辑"**。模型提供智能，Harness 让智能变得有用

**Harness 的六大组件架构**

|           |             |                                                                                               |
| --------- | ----------- | --------------------------------------------------------------------------------------------- |
| 组件        | 核心作用        | 关键机制                                                                                          |
| 文件系统      | 持久化存储与协作表面  | Agent 工作区、跨会话状态持久化、Git 集成实现版本控制与回滚、多代理和人机协作的共享表面                                              |
| Bash/代码执行 | 动态工具创建      | 给 Agent 通用代码执行能力而非固定工具集，让 Agent "通过代码即时设计自己的工具"，不受预配置工具的约束                                    |
| 沙箱        | 安全隔离执行      | OS 级隔离、命令白名单、网络隔离、按需环境供应、自验证循环（写代码→跑测试→查日志→修错误）                                               |
| 记忆与搜索     | 突破知识时效性边界   | AGENTS.md 等文件系统记忆标准实现持续学习、Web 搜索和 MCP 工具补充训练截止后的知识、上下文注入让 Agent 在未来会话中可访问更新的知识                |
| 上下文腐化管理   | 对抗长上下文推理退化  | 压缩（Compaction）：智能卸载并总结上下文；工具调用卸载：保留头尾 token，完整输出存文件系统；技能渐进式加载（Skills）：防止启动时加载过多工具导致性能退化       |
| 长周期自主执行   | 支撑跨多窗口的复杂任务 | Git 跟踪捕获百万 token 级的工作进度；Ralph Loop 拦截模型退出尝试，在干净上下文窗口中重新注入原始提示强制继续；规划与自验证让 Agent 分解目标并通过测试套件校验 |

**实战验证：四大优化策略**

LangChain 团队在 Terminal Bench 2.0（89 个跨 ML、调试、生物学的任务）上，使用 Harbor 框架 + Daytona 沙箱 + LangSmith 可观测性平台，仅调整三个变量（系统提示、工具、中间件），实现了 13.7% 的性能提升。

|   |   |   |   |
|---|---|---|---|
|策略|问题|解法|关键机制|
|构建与自验证循环|Agent 写完代码就停止，不运行测试就宣布完成|四阶段强制流程 + 退出拦截|Planning→Build→Verify→Fix 的结构化引导；PreCompletionChecklistMiddleware 在 Agent 退出前强制插入验证提醒（类似 Ralph Loop 模式）|
|环境上下文主动交付|Agent 在陌生环境中浪费大量时间探索|启动时主动注入环境信息|LocalContextMiddleware 在启动时映射目录结构和可用工具；注入时间预算提醒引导 Agent 合理分配精力；强调代码将面临程序化测试，需严格遵循规范|
|防止短视行为（Doom Loop）|Agent 陷入同一方案的 10+ 次无效重试|编辑频次监控 + 策略切换提醒|LoopDetectionMiddleware 跟踪每个文件的编辑次数，超过 N 次后建议"重新考虑你的方法"。属于临时性工程方案，随模型进步将逐步废弃|
|推理计算分配（Reasoning Sandwich）|全程高推理导致超时（得分降至 53.9%），全程低推理质量不足|分阶段差异化分配推理强度|规划阶段 XHigh → 实现阶段 High → 验证阶段 XHigh，形成"三明治"结构。未来方向：模型自适应推理（Claude 和 Gemini 已实现），由模型自行决定计算分配|

**Trace 驱动的迭代优化方法论**

LangChain 提出了一个类似机器学习 boosting 算法的 Harness 优化循环：通过 LangSmith 捕获每次 Agent 执行的完整 trace（延迟、token、成本、每步动作），然后用自动化 Trace Analyzer 并行分析失败案例，聚合反馈后针对性修改 Harness。这个方法的核心洞察是：**不是凭直觉优化，而是让数据和失败模式驱动改进**。

**模型-Harness 协同进化的双刃剑**

LangChain 观察到一个重要现象：模型会对特定 Harness 实现产生过拟合。例如改变工具逻辑可能导致性能下降，因为模型在后训练中已经适应了特定的 Harness 实现。但反直觉的是，**最优 Harness 不一定是模型训练时使用的那个**——Claude Opus 4.6 在非 Claude Code 的 Harness 中通过优化反而表现更好（从 Top 30 到 Top 5）。这意味着 Harness 工程存在独立于模型训练的优化空间。

**透过现象看本质，LangChain 的视角补充了前面三家的盲区：**

1. **Harness 是独立于模型的性能杠杆**：与 Anthropic 关注长时任务质量、Cursor 关注多代理编排、OpenAI 关注环境设计不同，LangChain 用实验数据证明了 Harness 本身作为独立变量的价值——同一模型，仅改 Harness 就能从 Top 30 进入 Top 5
2. **中间件是 Harness 的核心抽象**：LangChain 将 Harness 的可调节点抽象为三个"旋钮"（系统提示、工具、中间件），其中中间件（Middleware/Hooks）是最具工程价值的部分——它在模型调用前后插入确定性逻辑（如 PreCompletionChecklist、LoopDetection、LocalContext），将模型的"指令遵循"问题转化为"代码强制执行"问题，这与 OpenAI 的"约束优于指令"理念一脉相承
3. **Harness 的护栏应设计为可废弃的**：LoopDetectionMiddleware 等机制是为当前模型缺陷设计的临时方案。LangChain 明确指出这些护栏会随模型进步而废弃，但现在它们是构建可靠系统的必要条件。这呼应了 Anthropic 在 Opus 4.5→4.6 迭代中移除 Sprint 机制的实践——**好的 Harness 设计师会跟着模型能力的边界移动自己的重心**
4. ## 通识
5. **人的核心工作从写代码转向了设计 agent 的工作环境。**

> 个人觉得这个是最大共识"Code becomes inexpensive"，我们的工作也应该发生转变，从把想法变成代码 -> 设计agent的工作环境，让agent更快更好更稳定的把想法变成代码，人类的杠杆点在于创造让 agent 能可靠工作的条件，代码本身由 agent 产出。

6. **模型看不到的等于不存在**

> 减少工作中模型看不到的部分（复杂的流程｜Docs 里的讨论｜团队成员脑子里的隐含知识），让模型看到**精准，正确的信息**是关键

7. **Context not Control**

> 约束应该是可执行的、确定性的，指令是可解释的、模糊的。"约束优于指令"——"禁止TODO"比"请记得完成所有实现"更有效，具体数值比模糊表述更有效。约束是确定性的边界，指令是模糊的建议。在 agent 的工作方式里，这个区别比在人类团队里更关键。

8. **纠错比等待廉价**

> 在当前AI能快速交付的场景下，快速试错的成本远低于完美规划，四个团队的实践中都在强调不纠结于首次执行的完美，而是通过多轮的快速试错让结果趋向完美

9. **验证必须是强制的，不能是建议的**

> 四家都独立发现了同一个问题：Agent 天然倾向于写完就停、自我说服"可以接受"。四家的解法形式不同但本质一致——用确定性机制强制验证：Anthropic 用独立 Evaluator 角色 + 信息隔离，执行者不能给自己打分；Cursor 用 Review Agent 独立评估 Worker 产出；OpenAI 用 CI 测试 + Lint 规则，违规代码根本无法通过；LangChain 用 PreCompletionChecklistMiddleware 拦截退出，强制运行验证。**"请记得测试"是无效的指令，"退出前必须通过测试"是有效的约束。**

10. **Harness 应设计为可进化、随着Agent迭代而迭代**

> Harness 不是一成不变的静态产物，而是必须随模型能力迁移的动态系统：Anthropic 从 Opus 4.5→4.6 直接移除了 Sprint 机制，多轮 CR 简化为单轮；Cursor 持续演进架构，移除了造成瓶颈的 Integrator 角色；OpenAI 通过复利循环不断编码新约束；LangChain 将 LoopDetectionMiddleware 明确标注为"临时方案，随模型进步废弃"。**好的 Harness 设计师把护栏设计为可拆卸的脚手架，而非永久性的承重墙。重心永远跟着模型能力的边界走。**

11. **可观测性是 Harness 优化的前提**

> 你无法优化你看不见的东西。LangChain 通过 LangSmith 捕获完整 trace 并用 Trace Analyzer 自动分析失败模式（最系统化）；Cursor 将"数据驱动而非假设驱动"作为核心设计原则；OpenAI 用质量评分持续跟踪代码健康度；Anthropic 用 Sprint Contract 作为可观测的检查点，Evaluator 通过真实应用交互产生可审计的验证记录。**Trace/可观测性是 Harness 工程从"凭直觉调参"进化到"数据驱动改进"的基础设施。**

12. **文件系统是 Agent 世界的共享记忆**

> 四家不约而同地将文件系统作为 Agent 持久化和协作的核心原语：LangChain 将文件系统列为六大组件之首，是跨会话状态和多代理协作的基础；Anthropic 用进度日志 + Git 记录贯穿长时任务的多个上下文窗口；OpenAI 用 AGENTS.md + docs/ 目录，将决策从 Slack 回流到仓库成为可访问文件；Cursor 用 Handoff 文档实现异步协调，Worker 完成后写交接文档而非实时消息。**文件系统不只是存储，它是 Agent 的工作记忆、协作协议和时间机器（Git）。所有需要跨会话、跨 Agent 传递的信息，最终都会落回文件系统。**

# 参考文献：

> 一切讨论的来源： OpenAI：
> 
> https://openai.com/index/harness-engineering/
> 
> Cusor：
> 
> https://cursor.com/cn/blog/self-driving-codebases
> 
> https://cursor.com/cn/blog/scaling-agents
> 
> Anthropic：
> 
> https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
> 
> https://www.anthropic.com/engineering/harness-design-long-running-apps
> 
> Langchain
> 
> https://blog.langchain.com/the-anatomy-of-an-agent-harness/
> 
> https://blog.langchain.com/improving-deep-agents-with-harness-engineering/
> 
> 其他：
> 
> https://mp.weixin.qq.com/s/Q5oK_VtunXlouachrj1agQ
> 
> https://yage.ai/share/harness-engineering-scalability-20260330.html
