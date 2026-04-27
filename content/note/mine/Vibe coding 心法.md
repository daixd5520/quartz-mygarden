---
tags:
  - 方法论
  - 工具
---

动手写新代码前，先 rg 搜一下项目里是否已有类似实现。重复代码是技术债的最大来源之一。

-----

## cc 插件：

### 1 superpowers

- 与spec-kit的区别
	![[Vibe coding 心法-1.png|255]]
- 一个问题一个确认，苏格拉底式提问
	![[Vibe coding 心法-2.png|428]]
- 多分支，每个任务单开一分支，
	- 1.主分支始终干净；2.多任务互不干扰并行；3.失败直接删 work tree，零成本回滚（相当于一个沙盒了）
- TDD（测试驱动开发）。给一条反馈通路（相当于 ReAct？）
- ==AI不友好的架构==
	• 一个文件几千行，逻辑交织模块之间循环依赖
	• 业务逻辑散落在各处
	• 改一个地方要看十个件
- ==AI友好的架构==
	•**单职责**，每个模块做件事
	• 依赖关系清晰，单向流动
	• 业务逻辑内聚，改动范围可控 一个需求涉及的文件数可预期
架构债务会放大AI的问题。如果人都理不清的代码，AI更理不清。

- 上下文工程最好精准少量而有效
	![[Vibe coding 心法-3.png|387]]

### 2 wizard

[link](https://dev.to/_vjk/i-made-claude-code-think-before-it-codes-heres-the-prompt-bf)

>这个插件说的几步正是我使用 Trae 写 trustworthy-assistant 遇到大部分的问题。做这部分笔记的时候深有同感。

``` bash
curl -sL https://raw.githubusercontent.com/vlad-ko/claude-wizard/main/install.sh | bash
```

- 初级开发者（claudecode 默认）：阅读需求-打开文件-开始写代码
	- 容易出现 bug--缺乏**流程**
- senior（cc with wizard）：阅读需求-查看周围代码-阅读测试用例-检查 git 历史-写代码
	- 开始慢，完成快
- wizard--8 阶段方法论
	- 需要准备：一个 `CLAUDE.md` 定义你的项目规范，一个在开始工作前创建的 GitHub issue（`/wizard` 可以帮助你写一个），每个任务一个干净的特性分支，对 TDD（test driven development）的重视使用。
><mark style="background:rgba(240, 200, 0, 0.2)"> ### Phase 1: Plan before you touch anything</mark>
> 	Claude 读取你的 `CLAUDE.md` ，找到链接的 GitHub 问题，并在编写任何代码之前构建一个结构化的待办事项列表。它评估复杂性：可能受影响的文件数量、是否存在架构影响、可能出错的程度。然后相应地确定工作规模。
><mark style="background:rgba(240, 200, 0, 0.2)"> ### Phase 2: Explore before you assume</mark>
> 	搜索它打算使用的每个模型、方法、关系和常量，并在代码中引用它们之前验证它们的存在。Turns out "does this actually exist"
> <mark style="background:rgba(240, 200, 0, 0.2)">### Phase 3: Write the tests first</mark>
> 	强制执行 TDD. Claude 先写失败的测试，运行它们（它们必须失败），实现使它们通过的最小代码，然后验证。每次都按这个顺序，没有捷径。
> <mark style="background:rgba(240, 200, 0, 0.2)">### Phase 4: Implement the minimum</mark>
> <mark style="background:rgba(240, 200, 0, 0.2)">### Phase 5: Verify nothing regressed</mark>
> 	第 5 阶段运行更广泛的测试套件，而不仅仅是新测试。目标是零回归。如果无关紧要的事情出了问题，最好现在发现，而不是在 PR 评审评论中说“呃，为什么计费模块会失败？”
> <mark style="background:rgba(240, 200, 0, 0.2)">### Phase 6: Document while the context is fresh</mark>
> 	行内注释、变更日志条目、任何需要更新的内容。小步骤，容易跳过，但在背景信息消失之前总是值得做的。三个月后阅读这段代码的人可能就是你，完全记不起当初为什么做出那个决定。
><mark style="background:rgba(240, 200, 0, 0.2)"> ### Phase 7: The adversarial review</mark>
> 	在每次提交之前，Claude 都会以攻击者的身份而不是作者的身份来审查自己的工作。
> 		如果这个程序同时运行两次会怎样？
> 		如果输入是空的？空的？负的？
> 		我在做哪些假设可能是不正确的？
> 		如果这个在生产环境中崩溃了，后果如何
> 	以不同的模式思考代码：像==攻击者==一样，而不是像==作者==一样。
><mark style="background:rgba(240, 200, 0, 0.2)"> ### Phase 8: The quality gate cycle</mark>
> 	`/wizard` 不仅打开 PR 就认为完成了工作。它监控自动化审查机器人状态（虫子机器人、代码兔子，无论你有什么），阅读每个发现，修复有效问题，回复误报，并重复直到状态变为干净。
