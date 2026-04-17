[Github Copilot](https://github.com/copilot) 是 Github 推出的订阅计划，Pro+ 只需 39$ 即可拥有每月 1500 次高级模型调用。相比起 Claude Code 的订阅，Github 账号注册简单，普通银联信用卡即可完成支付。

Copilot 调用次数是按照自然月刷新的，也就是每个月的 1 号。所以如果你时间卡的好，理论上可以拉满双倍的 3000 次请求。

注意，非常不建议你用自己的 Github 大号做转发，虽然目前没有封号案例，但有 Anthropic (畜生) 的前车之鉴，还是谨慎为妙

## 通过 CC Switch 转发

如果你只是需要日常编程使用，最新版的 [CC Switch](https://github.com/farion1231/cc-switch) 直接内置了 Copilot 订阅转发能力，选择 Github Copilot 添加供应商，登录 Github 账号即可完成配置，开箱即用，非常简单。

![](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=ZWIwYTYxZmQwYTEyMjZjYWIwNWY4ZDUzMTMyMjAyOGJfZnI1a3Z4dWE3Vm5KamU5blJZT2xublZuT29DWjU5TkxfVG9rZW46TUQ0SmJtMzg2bzVQZTB4VXk1N2N2c0ZLbkpoXzE3NzYzOTU2NTk6MTc3NjM5OTI1OV9WNA)

![](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=NTAxMGE4YmQ0NTQ2ZTE5YTFjMTE3ZjM5NDI4MTU2OTJfNmpXemxLbU14SXhUZGtnMUNacUF2Z0czUVhGWTRrT1RfVG9rZW46S1BSTWJYRjJ3b1NkTXZ4dUlVdWNsSkN1blJmXzE3NzYzOTU2NTk6MTc3NjM5OTI1OV9WNA)

CC Switch 也支持了 Codex 的转发，在 Claude Code 里用 GPT-5.4 感觉还不错。

如果没生效可以尝试打开 CC Switch 的本地代理

![](https://bytedance.larkoffice.com/space/api/box/stream/download/asynccode/?code=ZWMzNjI1YmQ1YTNiMTM2MmQzMzAxZTM2ZmJmYWIxYzVfalROUEZhSlM5Y2tqZW9wV3NiUUJpaDF1N1pHTnlYU2NfVG9rZW46Rm9uUWJZM1dQb29kRVV4THU0UGM2VUZ5bllrXzE3NzYzOTU2NTk6MTc3NjM5OTI1OV9WNA)

## 自建中继服务

Copilot 是按照请求次数计费的。但并不是严格发起一次请求就扣减一次。研究后发现，Copilot API 通过 请求中的 `X-Initiator` 标头来表示发起人是“用户”还是“Agent”，如果是用户就扣减一次。如果我们自己来做这个中继，**理论上完全可以把所有请求的** **`X-Initiator`** **都标记为 Agent，大幅缩小扣减次数！**

以及如果你希望将 API 分享给团队/接入虾/自动化能力等，也可以自建。

仓库地址：https://code.byted.org/wangshuaiqi.com/copilot-relay

本项目内置了 Skill，可以一键完成安装和日常管理。

**首次安装：** `/install-relay`

该技能会引导你完成完整安装流程：安装依赖、GitHub 登录、选择 initiator 策略、生成密钥、启动 relay、运行自检，并打印包含所有配置值的摘要。

**启动或重启 relay：** `/start-relay`

检查已有账号，确认 relay 是否已在运行，如未运行则启动，然后确认健康状态、用量和近期统计。

**功能特性**

- **多账号池** — 注册多个 GitHub 账号，自动故障转移与冷却
- **Initiator 策略** — 通过控制 `x-initiator` 头来管理高级配额消耗（`off` / `session` / `always`）
- **流式代理** — SSE 响应零缓冲直通转发
- **审计日志** — 每个请求记录模型、token 数、initiator 及耗时，写入 `data/audit.jsonl`
- **用量与统计** — 通过管理端点实时查看配额跟踪与历史统计
- **热登录** — 运行时通过 `/relay/login/start` API 添加账号，无需重启

详细使用方式见 README
