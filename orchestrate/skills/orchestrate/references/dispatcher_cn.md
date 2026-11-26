调度器（dispatcher）的操作手册。如果你是以 "You are the root planner for:" 或 "You are a subplanner for:" 开头的提示词被启动的，请改读 `planner.md`。

# 调度器（Dispatcher）

调度器是一次性的。接收用户的目标，通过 CLI 启动一个云端 root planner，返回 URL，然后停止。

## 职责

1. 接收用户的目标。仅在目标缺失或含糊不清时才要求澄清。用户是刻意选择了并行云端编排；只有当任务确实微不足道时才应劝退。
2. 用该目标以及用户指定的任何约束（模型覆盖、仓库覆盖）运行启动 CLI。
3. 将 CLI 输出中的 URL 返回给用户。停止。planner 会自行驱动。

一次性设置：如果 `node_modules/` 缺失，请在此 skill 的 `scripts/` 目录中运行 `bun install`。这些脚本有意放置在宿主仓库的包管理器工作区之外。

```bash
bun cli.ts kickoff "<goal>" [--repo <url>] [--ref main] [--model claude-opus-4-8] [--slack-channel C123] [--dispatcher-name "Alex"]
```

CLI 会读取 `CURSOR_API_KEY`，从 `git config --get remote.origin.url` 自动检测仓库，构建 spawn 提示词，通过 `cursor-sdk` 启动，并打印 `{ agentId, runId, status, url, dispatcherFirstName }` JSON。Slack 是可选的。如果设置了 `SLACK_BOT_TOKEN`，还要传入 `--slack-channel <id>` 或设置 `SLACK_CHANNEL_ID`；否则启动会在 spawn 之前失败。如果未设置该 token，Slack 保持禁用状态。

## 调度器身份

当名字解析成功时，启动机器人用户名为 `<firstName>'s bot`，否则为 `orchestrate`。解析顺序：

1. `--dispatcher-name "Alex"` 标志。
2. 针对 `git config user.email` 的 Slack `users.lookupByEmail`（尽力而为；缺少权限范围或无匹配时保持未设置）。

CLI 通过启动提示词将解析出的名字传给 root planner；planner 将 `plan.dispatcher = { firstName: "<name>" }` 写入 `plan.json`。子任务保留各自的任务名作为机器人身份。

## 运行摘要

root planner 将 `plan.summary` 写成供 Slack 线程中人类阅读的一行简介。启动会发布 `<rootSlug>: <summary> <agent-link>`；没有 `summary` 时会将 `goal` 截断到约 200 个字符。`summary` 面向人类；`goal` 保留为面向 agent 的完整文本。

## 最小化目标原则

原样传递用户的目标，不要扩展它。不要添加规划启发式、子 planner 数量或结构性规定。planner 会阅读 orchestrate skill 并自行决定如何分解。过度规定会将调度器的上下文泄漏到 planner 的窗口中，并使该 skill 作为对 planner 判断力的真实测试失效。

## 认证

`CURSOR_API_KEY` 必须是用户 API key，而不是团队 key。认证来源的优先级记录在 `cursor-sdk` skill（https://github.com/cursor/plugins/tree/main/cursor-sdk）中。不要把 keychain 查找硬编码进启动 CLI 本身；云端 agent 虚拟机没有 keychain。

## 启动后的可观测性

派发之后可以观测进度：

- `bun cli.ts crawl <repo-path> <branch> <root-slug>` 用于查看深层树状视图。
- `bun cli.ts status` 用于查看顶层状态。
- 当设置了 `SLACK_BOT_TOKEN` 时，查看 `plan.slackChannel` 中的 Slack 启动线程。

`syncStateToGit` 默认为 true。当目标或交接内容不应提交时，在 root plan 上设置 `syncStateToGit: false`。
