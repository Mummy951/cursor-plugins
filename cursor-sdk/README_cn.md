# Cursor SDK

一个 Cursor 插件，包含一个 skill，帮助用户基于 Cursor TypeScript SDK（`@cursor/sdk`）进行构建。该 skill 涵盖三种调用模式（`Agent.prompt`、`Agent.create` + `agent.send`、`Agent.resume`）、新集成方案中最常见的陷阱、运行时选择（本地 vs 云端）、认证、流式处理、MCP、错误处理，以及可直接扩展的 CI、定时任务、聊天和 webhook 模式。

该 skill 有意保持简短——仅当用户的需求明确属于某个专注的参考文件时，才会指向该文件。

## 包含内容

- `cursor-sdk`：使用 `@cursor/sdk` 进行构建的设计与集成指南，以及关于运行时选择、认证、错误处理、流式处理、MCP、高级功能和集成模式的参考文件。

## 何时使用

当用户集成、安装或编写针对 Cursor SDK 的代码时使用；提及 `Agent.create`、`Agent.prompt`、`Agent.resume`、`agent.send`、`run.stream`、`CursorAgentError` 或 `@cursor/sdk`；希望从脚本、CI/CD 流水线、GitHub Action、后端服务、机器人或 webhook 中运行 Cursor agent；正在本地与云端运行时之间做选择；正在为 SDK agent 配置 MCP server；或者正在将 REST `/v1/agents` 调用迁移到该 SDK 时使用。

该 skill 是关于外部 `@cursor/sdk` 包的权威信息来源，应当主动加载，而不是凭借记忆作答。

## 参考文件

该 skill 保持主 `SKILL.md` 简短，仅当用户的需求明确属于某个参考文件时才会读取它：

| 如果用户正在……                                                                      | 参考文件                               |
| ------------------------------------------------------------------------------------ | -------------------------------------- |
| 在本地与云端运行时之间做选择                                                         | `references/runtime-choice.md`         |
| 调试认证问题（401、缺少 key、团队 key 与用户 key）                                   | `references/auth.md`                   |
| 处理错误、重试、速率限制、`CursorAgentError`、`result.status === error`             | `references/error-handling.md`         |
| 消费流、选择事件类型、取消，或流与等待的取舍                                         | `references/streaming.md`              |
| 配置 MCP server（HTTP、stdio、transport、认证注入）                                  | `references/mcp.md`                    |
| 子 agent、resume、artifacts、列出/检视 agent、`Agent.messages`                       | `references/advanced.md`               |
| 构建特定集成（CI 审查机器人、分类、聊天、webhook）                                   | `references/patterns.md`               |

## 许可证

MIT
