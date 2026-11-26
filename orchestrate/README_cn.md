# Orchestrate（编排）

通过 Cursor SDK 将一个大型任务分发给并行的 Cursor 云端 agent。planner（规划者）发布任务，worker（执行者）向上交接，脚本则从磁盘和 git 中对树进行对账，从而使 spawn / wait / handoff（生成 / 等待 / 交接）循环持续收敛，而无需长期运行的 agent 状态。

该 skill 本身位于 [`skills/orchestrate/SKILL.md`](./skills/orchestrate/SKILL.md)。阅读该文件以获取完整的操作手册；本 README 仅介绍在调用它之前需要配置的内容。

## 前置条件

- PATH 中有 `bun`。
- `CURSOR_API_KEY` 中有一个 Cursor API 密钥。
- 如果你希望有一个 Slack 线程来镜像运行过程，则需要可选的 Slack app 和 bot token。

## Cursor API 密钥

1. 打开 [https://cursor.com/dashboard/integrations](https://cursor.com/dashboard/integrations)。
2. 创建一个个人用户 API 密钥。其值以 `cursor_` 开头。
3. 导出它：`export CURSOR_API_KEY="cursor_..."`。

团队服务账号密钥（Team Settings → Service accounts）同样适用于本地和云端运行。完整的认证模型请参阅 [`cursor-sdk` 插件](https://github.com/cursor/plugins/tree/main/cursor-sdk)。

## Slack app（可选）

Slack 可见性是可选启用的。当 token 未设置时，脚本只记录一次日志，然后在不使用 Slack 的情况下运行；正确性不会改变。要启用它：

1. 在 [https://api.slack.com/apps](https://api.slack.com/apps) 处创建一个 Slack app → **From scratch**。选择一个名称和一个工作区。
2. 在 **OAuth & Permissions** → **Bot Token Scopes** 下，添加：

   | Scope | 原因 |
   | --- | --- |
   | `chat:write` | 发布和编辑消息。 |
   | `chat:write.customize` | 在每次发布时设置 bot 用户名和图标。 |
   | `chat:write.public` | 在公共频道中发布，无需先邀请 bot。 |
   | `files:write` | 将交接产物上传到运行线程。 |
   | `files:read` | 与 `files:write` 配合用于 upload v2 流程。 |
   | `reactions:read` | 监视 kickoff 消息上的 Andon `:rotating_light:` 反应。 |
   | `channels:history` | 读取线程回复。如果你的运行频道是私有的，请改用 `groups:history`。 |

   可选但推荐：

   | Scope | 原因 |
   | --- | --- |
   | `users:read.email` | 根据 `git config user.email` 解析调度者的名字。如果不使用该项，请显式传入 `--dispatcher-name`。 |

3. **Install to Workspace** 并复制 **Bot User OAuth Token**（`xoxb-...`）。
4. 导出它：`export SLACK_BOT_TOKEN="xoxb-..."`。
5. 将 bot 邀请到你希望运行过程以线程形式呈现的频道（`/invite @your-bot`）。带有 `chat:write.public` 的公共频道可跳过此步；私有频道则需要邀请。
6. 获取频道 ID。在 Slack 中：右键点击频道 → **View channel details** → 对话框底部。在 `kickoff` 时通过 `--slack-channel <id>` 传入（或设置 `SLACK_CHANNEL_ID`）。第一次 kickoff 会将该 id 持久化到计划中；子 planner 和之后的 `run` 调用会继承它。

## 安装

```bash
cd skills/orchestrate/scripts
bun install
```

这些脚本有意置于宿主仓库的包管理器工作区之外。

## 调用

```bash
bun skills/orchestrate/scripts/cli.ts kickoff "<goal>" \
  [--repo <url>] [--ref main] [--model claude-opus-4-7] \
  [--slack-channel <id>] [--dispatcher-name "<first name>"]
```

CLI 会打印 `{ agentId, runId, status, url }`；从此处开始，云端根 planner 会自行驱动。有关 `run`、`spawn`、`respawn`、`kill`、`tail`、`comment` 和 `andon` 子命令，请参阅该 skill。

## 许可证

MIT。请参阅 [`LICENSE`](./LICENSE)。
