---
tags:
  - agent
  - llm应用
  - 项目
---

# 什么是nanobot？

> https://github.com/HKUDS/nanobot

源于OpenClaw，一个极致轻量化的AI个人助理。

和大家所熟知的OpenClaw、Manus、Claude code是一个东西，没错，和OpenClaw已经160w行的代码数量相比，作者用python对OpenClaw进行了重构，nanobot只用了大约 1% 代码复刻了 OpenClaw 同等量级的功能。

阅读文本你可以了解到：

1. 如何从零实现一个AI个人助理（agent框架）
2. nanobot在openclaw的基础上做了哪些优化抽象

极致精简的代码量背后是良好的抽象与设计，同时也更适合于研究学习。首先来看下nanobot的能力：

1. **ReAct Loop：Agent推理循环，目前主流的Agent范式
2. **上下文管理：自动的上下文压缩策略
3. **双层memory系统：**长期记忆+历史记录
4. **支持10+聊天渠道：**Telegram、Discord、飞书、钉钉、微信、Slack、QQ、Email、WhatsApp、Matrix
5. **支持20+LLM供应商：**OpenAI、Claude、DeepSeek、Gemini、通义千问、Kimi……
6. **支持MCP协议**

# 整体架构

## 架构设计

下面是github上给的架构图，整体比较简略但是包含了nanobot核心的模块

![[Tech--Nanobot 学习-1.png]]
**详细架构图**

nanobot整体架构上可以分为4层：

- Access Layer：命令行工具CLI、聊天工具
- Message Bus Layer：封装的消息总线，通过两个Queue抽象了消息的发送接收，屏蔽了不同 Channel 聊天平台的差异
- Agent Core Layer：核心部分，包含ReAct Loop、Context上下文管理、Memory管理、Skill、LLM Providers
- Infrastructure Layer：包括Cron定时任务、心跳检测、session会话管理、安全、配置加载等

![[Tech--Nanobot 学习-2.png]]
## **项目结构**

> 比较清晰，一目了然

```Python
nanobot/
├── nanobot/                    # 🧠 【核心 Python 包】系统的大脑和躯干
│   ├── agent/                 # 🤖 核心代理层 (Agent Core)
│   │   ├── loop.py            #   ReAct 主循环 (思考->行动->观察)
│   │   ├── context.py         #   Prompt 组装车间
│   │   ├── memory.py          #   记忆系统 (长短期记忆合并与持久化)
│   │   ├── skills.py          #   技能加载与解析器
│   │   ├── subagent.py        #   子任务/子代理管理器
│   │   └── tools/             #   🛠️ 内置工具库
│   ├── bus/                   # 🚌 消息总线 (Message Bus)
│   │   ├── events.py          #   定义标准消息结构 (Inbound/Outbound)
│   │   └── queue.py           #   异步队列，解耦 Channels 和 Agent
│   ├── channels/              # 🔌 渠道适配层 (Channel Adapters)
│   │   ├── base.py            #   渠道基类 (定义公共接口如接收、发送、权限校验)
│   │   ├── telegram.py        #   Telegram 机器人接入
│   │   ├── feishu.py          #   飞书接入
│   │   └── ... (Slack, Email, DingTalk 等)
│   ├── providers/             # 🧠 模型适配层 (LLM Providers)
│   ├── skills/                # 🎯 内置技能包 (Built-in Skills)
│   ├── session/               # 💬 会话管理 (Session Management)
│   ├── cron/                  # ⏰ 定时任务 (Cron Jobs)
│   ├── heartbeat/             # 💓 心跳服务 (Heartbeat)
│   ├── config/                # ⚙️ 配置管理 (Configuration)
│   ├── cli/                   # 🖥️ 命令行界面 (Command Line Interface)
│   └── utils/                 # 🧰 通用工具类
└── docs/                      # 开发者文档（如插件开发指南）
```

# 深度解析

## Agent的核心 - ReAct Loop

ReAct（Reasoning + Acting）是目前最主流的 Agent 范式。将COT（Chain of Thought，思维链）和工具调用相结合，模拟人类思考与行动的过程，直白点说，就是让LLM进行「推理 - 工具调用 - 推理」不断循环，并根据行动反馈调整策略，最终解决问题。

```Python
async def _run_agent_loop(self, initial_messages, on_progress) -> tuple[str | None, list[str], list[dict]]:
    """Run the agent iteration loop."""
    messages = initial_messages
    iteration = 0
    final_content = None
    while iteration < self.max_iterations:
        iteration += 1
        tool_defs = self.tools.get_definitions()
        # 第一步 💡调用LLM推理
        response = await self.provider.chat_with_retry(
            messages=messages,
            tools=tool_defs,
            model=self.model,
        )
        if response.has_tool_calls:
            tool_call_dicts = [
                tc.to_openai_tool_call()
                for tc in response.tool_calls
            ]
            # 🤖 LLM推理结果加入上下文
            messages = self.context.add_assistant_message(messages, response.content, tool_call_dicts)
            # 第二步 🔧 工具调用
            for tool_call in response.tool_calls:
                result = await self.tools.execute(tool_call.name, tool_call.arguments)
                # 📃 工具调用结果加入上下文
                messages = self.context.add_tool_result(
                    messages, tool_call.id, tool_call.name, result
                )
        else:
            clean = self._strip_think(response.content)
            # Don't persist error responses to session history — they can
            # poison the context and cause permanent 400 loops
            if response.finish_reason == "error":
                ...
            messages = self.context.add_assistant_message(...)
            final_content = clean
            break
            
    if final_content is None and iteration >= self.max_iterations:
        logger.warning("Max iterations ({}) reached", self.max_iterations)
        final_content = (
            f"I reached the maximum number of tool call iterations ({self.max_iterations}) "
            "without completing the task. You can try breaking the task into smaller steps."
        )
        
    return final_content, tools_used, messages
```

## 上下文管理 - ContextBuilder

ContextBuilder负责Agent的上下文管理，上下文包含以下部分：

- runtime_ctx：运行时信息包括channel、chatID
- user_content：用户输入，支持多模态，_build_user_content里面封装了多模态数据的解析，调用外部工具
- system_prompt：
    - _get_identity：nanobot身份信息

      `# nanobot 🐈 You are nanobot, a helpful AI assistant.`

    - _load_bootstrap_files：`["AGENTS.md", "SOUL.md", "USER.md", "TOOLS.md"]`
    - 短期记忆：从memory加载
    - skill：
        - 常驻skill：整个`skill.md`加载到context
        - 其他skill：（**渐进式披露**）只加载summary信息`name、description`

后续和nanobot对话中的信息和工具调用都会加入到context当中

```Python
class ContextBuilder:
    """Builds the context (system prompt + messages) for the agent."""

    BOOTSTRAP_FILES = ["AGENTS.md", "SOUL.md", "USER.md", "TOOLS.md"]

    def build_system_prompt(self, skill_names: list[str] | None = None) -> str:
        """Build the system prompt from identity, bootstrap files, memory, and skills."""
        parts = [self._get_identity()]
        # 加载引导文件 ["AGENTS.md", "SOUL.md", "USER.md", "TOOLS.md"]
        bootstrap = self._load_bootstrap_files()
       
        # 🧠 加载短期记忆
        memory = self.memory.get_memory_context()
        
        # 加载常驻技能
        always_skills = self.skills.get_always_skills()
       
        # 加载技能摘要
        skills_summary = self.skills.build_skills_summary()
        if skills_summary:
            parts.append(f"""# Skills

The following skills extend your capabilities. To use a skill, read its SKILL.md file using the read_file tool.
Skills with available="false" need dependencies installed first - you can try installing them with apt/brew.

{skills_summary}""")

        return "\n\n---\n\n".join(parts)

    def build_messages(self, history, current_message, skill_names, media, channel,chat_id,current_role) -> list[dict[str, Any]]:
        """Build the complete message list for an LLM call."""
        # 运行时
        runtime_ctx = self._build_runtime_context(channel, chat_id)
        # 用户输入 支持多模态 _build_user_content里面封装了多模态数据的解析，调用外部工具
        user_content = self._build_user_content(current_message, media)

        # Merge runtime context and user content into a single user message
        # to avoid consecutive same-role messages that some providers reject.
        if isinstance(user_content, str):
            merged = f"{runtime_ctx}\n\n{user_content}"
        else:
            merged = [{"type": "text", "text": runtime_ctx}] + user_content

        return [
            {"role": "system", "content": self.build_system_prompt(skill_names)},
            *history,
            {"role": current_role, "content": merged},
        ]
        
    def add_tool_result()
    def add_assistant_message()
```

## Agent记忆系统 - Memory
### 双层记忆系统

nanobot的记忆系统由长期事实记忆MEMORY.md 、可搜索的历史记录HISTORY.md组成

```Python
class MemoryStore:
    """Two-layer memory: MEMORY.md (long-term facts) + HISTORY.md (grep-searchable log)."""
```

|   |   |   |
|---|---|---|
||MEMORY.md|HISTORY.md|
|介绍|长期事实记忆，大脑中的认知|按时间戳记录的grep友好的历史日志|
|读取时间|每轮对话都会读取，加载进system_prompt|按需搜索|
|写入方式|全量覆盖写|尾部追加append|

**MEMORY.md为啥是全量覆盖写？**

MEMORY.md是「长期事实记忆」相当于nanobot中的大脑认知，这份认知可能被更新（更新旧的内容），而不是追加。

> 更新**MEMORY.md prompt：** full updated long-term memory as markdown. Include all existing facts plus new ones. Return unchanged if nothing new.

### 归档机制 - consolidate

LLM的上下文窗口是有限制的，当对话越来越长，会触发上下文窗口的限制，需要有一种机制来处理这个问题。

> nanobot上下文窗口限制 64K

1. **触发时机**

- 收到用户消息同步触发
- 后台协程空闲时异步执行
- 强制归档，当用户输入 /new 清空当前会话时，旧消息会被强制进行归档

2. **加锁**

用session.key进行加锁，避免和后台异步任务并发执行

3. **Chunk选取** 一旦判断当前消息的 Token 超出最大安全限制，系统会在**自然对话轮次**处切一刀（`pick_consolidation_boundary`），将最老的一批未归档消息（Chunk）提取出来
4. **记忆的更新与合并 (MEMORY.md 逻辑)**

提取出来的消息不会直接简单追加，而是连同现有的 `MEMORY.md` 整体喂给大模型，大模型通过工具调用 (`save_memory`) 将新老知识融合成一份**全新的长期记忆 (****`memory_update`****)，然后完全覆盖（Overwrite）** 现存的 `MEMORY.md`

4. **历史记录的追加 (HISTORY.md 逻辑)**

大模型同时会输出一份关于这段切片发生的摘要动作 (`history_entry`)，以追加（Append）形式写入 `HISTORY.md`，构建出一个纯线性增长的历史记录

5. **滑动窗口**

一轮归档完成后，游标 `last_consolidated` 前移。如果剩余未归档部分消息的 Token 量依然很大（context_window_tokens / 2，默认64k / 2），则不断循环这一过程

一个很实用的稳定性设计：如果连续多次 consolidate 失败，nanobot 会降级为 raw archive（把原始消息直接 dump 到 HISTORY.md），宁可“记得粗糙”，也不要阻塞主流程

![[Tech--Nanobot 学习-4.png]]

## 渐进式加载 - Skill

```Python
def build_system_prompt(self, skill_names: list[str] | None = None) -> str:
    """Build the system prompt from identity, bootstrap files, memory, and skills."""
    parts = [self._get_identity()]
    # 加载引导文件 ["AGENTS.md", "渐进式披露", "USER.md", "TOOLS.md"]
    bootstrap = self._load_bootstrap_files()
    
    # 🧠 加载短期记忆
    memory = self.memory.get_memory_context()
    if memory:
        parts.append(f"# Memory\n\n{memory}")
    
    # 加载常驻技能
    always_skills = self.skills.get_always_skills()
    if always_skills:
        always_content = self.skills.load_bootstrap_files(always_skills)
        if always_content:
            parts.append(f"# Active Skills\n\n{always_content}")
    
    # 加载技能摘要
    skills_summary = self.skills.build_skills_summary()
    if skills_summary:
        parts.append(f"""# Skills
The following skills extend your capabilities. To use a skill, read its SKILL.md file using the read_file tool.
Skills with available="false" need dependencies installed first - you can try installing them with available="false" need dependencies installed first - you can try installing them with apt/brew.{skills_summary}
""")
    
    return "\n\n---\n\n".join(parts)
```

## 消息总线 - MessageBus

MessageBus定义两个队列，一进一出用于发送消息和接收消息

```Python
class MessageBus:
    """
    Async message bus that decouples chat channels from the agent core.

    Channels push messages to the inbound queue, and the agent processes
    them and pushes responses to the outbound queue.
    """

    def __init__(self):
        self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()

    async def publish_inbound(self, msg: InboundMessage) -> None:
        """Publish a message from a channel to the agent."""
        await self.inbound.put(msg)

    async def consume_inbound(self) -> InboundMessage:
        """Consume the next inbound message (blocks until available)."""
        return await self.inbound.get()

    async def publish_outbound(self, msg: OutboundMessage) -> None:
        """Publish a response from the agent to channels."""
        await self.outbound.put(msg)

    async def consume_outbound(self) -> OutboundMessage:
        """Consume the next outbound message (blocks until available)."""
        return await self.outbound.get()
        
 
```

BaseChannel是channel的基类，每一个channel子类都会调用基类的`_handle_message`方法，实现消息的统一处理和解耦

```Python
class BaseChannel(ABC):
     async def _handle_message(self,sender_id,chat_id,content,media,metadata,session_key) -> None:
        """
        Handle an incoming message from the chat platform.
        """
        
        await self.bus.publish_inbound(msg)
```

# 快速上手/开发指南

安装

```Bash
# 安装
uv tool install nanobot-ai
```

初始化

```Bash
# 初始化配置
nanobot onboard
```

需要配置API Key `vim ~/.nanobot/config.json`

```JSON
{
  "agents": {
    "defaults": {
      "workspace": "~/.nanobot/workspace",
      "model": "ep-20260303165437-6tjxw",
      "provider": "openrouter",
      "maxTokens": 8192,
      "contextWindowTokens": 65536,
      "temperature": 0.1,
      "maxToolIterations": 40,
      "reasoningEffort": null
    }
  },
  ...
  "providers": {
    "openrouter": {
      "apiKey": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxx",
      "apiBase": "https://ark-cn-beijing.bytedance.net/api/v3", // 使用方舟上的模型需要配置这个
      "extraHeaders": null
    },
  }
}  
```

开始对话

```Bash
# 开始对话
nanobot agent
```

![[Tech--Nanobot 学习-3.png]]
## 对接飞书机器人

修改配置`vim ~/.nanobot/config.json`

```JSON
{
    "feishu": {
        "enabled": true,
        "appId": "cli_a94f94d0cd79dcc9",
        "appSecret": "GiTsoeePmZSUBiwi2l46reeUdJZXMlFR",
        "encryptKey": "",
        "verificationToken": "",
        "allowFrom": [
            "*"
        ],
        "reactEmoji": "THUMBSUP",
        "groupPolicy": "mention",
        "replyToMessage": false
    }
}
```


配置完就可以nanobot愉快的对话啦！

![](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=YmEzZmE5ZGE1MmEyNjViZWFhZjU3ZGQzNzI4NjhlMDBfd0o5OHNEUDVzMkd5YTROQ1hJUFJnYUczd1RMTTYyMHRfVG9rZW46REtRRmIzcmZ3b0U0Vnl4T0QwUmNRV0RzbnFoXzE3NzU3MDYwMDQ6MTc3NTcwOTYwNF9WNA)

# 总结

Agent的核心在于围绕LLM构建了
