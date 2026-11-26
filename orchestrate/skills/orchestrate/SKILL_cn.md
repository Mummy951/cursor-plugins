---
name: orchestrate
description: 仅在用户显式输入 `/orchestrate <goal>` 时使用，用于分解一个大任务、通过 Cursor SDK 生成一棵由并行 cloud-agent 工作者/子规划者/验证者组成的树，并收集结构化的交接信息；不要自主调用。
disable-model-invocation: true
---

# Orchestrate

一次显式的 `/orchestrate <goal>` 会将一个大任务分发给并行的 Cursor cloud agent。工作者之间互不通信；它们通过结构化的交接信息向上汇报。生成（spawn）、等待（wait）和交接（handoff）的循环位于 `scripts/cli.ts` 中。规划者编写 `plan.json`，脚本执行它，规划者读取交接信息以决定下一步做什么。长时间运行的 agent 循环会发生偏移；带有 JSON 状态文件的脚本能够保持立场稳固。

**必读：`cursor-sdk` skill（[cursor/plugins/cursor-sdk](https://github.com/cursor/plugins/tree/main/cursor-sdk)）。** 生成、鉴权和错误分类都在那里。不要重新实现该 skill 已经记录的内容。

## 设置

- `CURSOR_API_KEY` 必须是个人/用户级别的 key。从 [Cursor Dashboard > Integrations](https://cursor.com/dashboard/integrations) 创建它，然后在使用前阅读 `cursor-sdk` 的 Auth 部分。
- `SLACK_BOT_TOKEN` 是可选的。设置后，向 `kickoff` 或第一个 `run --root` 传递 `--slack-channel <id>`，或者设置 `SLACK_CHANNEL_ID`。脚本会把 channel 存储在 `plan.slackChannel` 中，在那里发布 kickoff 线程，镜像任务状态，并读取 Andon 反应。当 token 未设置时，脚本只记录一次日志，并在没有 Slack 可见性的情况下运行；正确性不会改变。

## 核心原则

这些规则使整棵树无需全局协调即可自我收敛。

1. **规划者拥有范围并发布任务。它们不编写代码。** 编写 `plan.json`、读取交接信息以及决定下一步是什么是规划者的工作。编辑文件、运行 `git merge` 以及就地修复冲突则不是。如果规划者产生了想要编码的冲动，它应该改为为工作者发布一个任务。
2. **规划者不知道谁会接手它们的任务。** 脚本将每个任务路由给一个 cloud agent。规划者的心智模型保持在任务层面。
3. **工作者是隔离的。** 一个任务、一份仓库克隆、没有任何通往其他 agent 的通道。完成后交接一次。
4. **子规划者是递归的规划者。** 规划者发布一个"为该切片制定子计划"的任务；子规划者完全拥有该切片，并交回一份聚合的交接信息。
5. **通过交接持续运转。** 一个以为自己已经完成的规划者可能会收到迟到的交接信息并重新规划。在规划者决定停止发布任务之前，不存在"完成"状态。
6. **传播，而非同步。** 兄弟节点之间没有交叉通信。层级之间没有共享状态。每一层只能看到其子节点的交接信息。

## 节点类型

| 节点             | 运行循环？ | 范围                       | 输出                                        |
| ---------------- | ---------- | -------------------------- | ------------------------------------------- |
| Planner          | 是         | 整个用户目标               | 面向用户的消息 + 可选的 PR                   |
| Subplanner (↻)   | 是         | 父级范围的一个切片         | 向父级交接                                   |
| Worker           | 否         | 一个具体任务               | 向生成它的规划者交接                         |
| Verifier         | 否         | 一个目标的验收标准         | 向生成它的规划者交接裁决（Verdict）          |
| Git              | 不适用     | 共享媒介                   | 分支（代码） + handoffs/（含义）             |

## 角色

一个 skill，两种角色。阅读你角色的参考文件，跳过另一个。

**Dispatcher（调度者）。** 你处于本地 IDE 会话中，用户输入了 `/orchestrate <goal>`。你的工作是启动一个云端根规划者（root planner）并返回其 URL。参见 `references/dispatcher.md`。这是一次性的；你不是规划者。

**Planner（规划者，根或子）。** 你是通过一个结构化 prompt 被生成的，该 prompt 以"You are the root planner for:"或"You are a subplanner for:"开头。或者用户选择在本地运行规划循环。你拥有一个范围、发布任务、读取交接信息、决定下一步做什么。参见 `references/planner.md`。

`disable-model-invocation: true` 意味着此 skill 仅在显式调用时加载。
