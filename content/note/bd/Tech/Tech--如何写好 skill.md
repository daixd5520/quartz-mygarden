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
