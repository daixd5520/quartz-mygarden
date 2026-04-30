---
tags:
  - 工具
title: Claude Code 指向 MiniMax 的 alias
---

# Claude Code 指向 MiniMax 的 alias

本地常用的一个 alias，把 `cc` 绑到 MiniMax 的 Claude 兼容端点上，省掉每次手动导环境变量。把这段塞进 `~/.zshrc` 就能用。

```bash
alias cc="ANTHROPIC_API_KEY=sk-xxx \
ANTHROPIC_BASE_URL=https://api.minimaxi.com/anthropic \
ANTHROPIC_DEFAULT_OPUS_MODEL=MiniMax-M2.7 \
ANTHROPIC_DEFAULT_SONNET_MODEL=MiniMax-M2.7 \
ANTHROPIC_DEFAULT_HAIKU_MODEL=MiniMax-M2.7 \
CLAUDE_CODE_SUBAGENT_MODEL=MiniMax-M2.5 \
claude"
```

`sk-xxx` 换成自己的 key。三个模型槽位都指到 M2.7，sub-agent 单独走 M2.5——日常调试用 M2.7 体感够，sub-agent 调用量大用 M2.5 控制成本。

真正的 key 放本地 secret，不要 commit。
