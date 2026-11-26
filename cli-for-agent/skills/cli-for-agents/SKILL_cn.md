---
name: cli-for-agents
description: >-
  设计或评审 CLI，使编码 agent 能够可靠地运行它们：非交互式标志、带示例的分层
  --help、stdin/管道、快速且可操作的错误、幂等性、dry-run 以及可预测的结构。当构建
  CLI、添加命令、编写 --help，或用户提及 agent、终端或适合自动化的 CLI 时使用。
---

# 面向 agent 的 CLI

面向人类的 CLI 常常会阻碍 agent：交互式提示、大量的前置文档，以及没有可复制粘贴示例的帮助文本。优先采用那些能在无头（headless）环境下工作、并能在管道中组合的模式。

## 非交互式优先

- 每一项输入都应该能表示为一个标志或标志值。不要要求使用方向键、菜单或带超时的提示。
- 只有在缺少标志时，**才**回退到交互模式——而不是反过来。

**差：** `mycli deploy` → `? Which environment? (use arrow keys)`  
**好：** `mycli deploy --env staging`

## 可发现性，而非倾倒上下文

- agent 是逐步发现子命令的：先 `mycli`，然后是 `mycli deploy --help`。不要在每次运行时都打印整本手册。
- 让每个子命令拥有自己的文档，这样未使用的命令就不会进入上下文。

## 好用的 `--help`

- 每个子命令都有 `--help`。
- 每个 `--help` 都包含带有真实调用示例的 **Examples**。示例比散文更有助于模式匹配。

```text
Options:
  --env     Target environment (staging, production)
  --tag     Image tag (default: latest)
  --force   Skip confirmation

Examples:
  mycli deploy --env staging
  mycli deploy --env production --tag v1.2.3
  mycli deploy --env staging --force
```

## stdin、标志与管道

- 在合理的地方接受 stdin（例如 `cat config.json | mycli config import --stdin`）。
- 避免奇怪的位置参数顺序，避免为缺失的值回退到交互式提示。
- 支持链式调用：`mycli deploy --env staging --tag $(mycli build --output tag-only)`。

## 快速失败并给出可操作的错误

- 当缺少必需标志时：立即退出，给出清晰的消息和一个**正确的示例调用**，而不是卡住不动。

```text
Error: No image tag specified.
  mycli deploy --env staging --tag <image-tag>
  Available tags: mycli build list --output tags
```

## 幂等性

- agent 会频繁重试。同一条成功的命令运行两次应当安全（无操作，或明确提示"已完成"），而不是产生重复的副作用。

## 破坏性操作

- 添加 `--dry-run`（或等价机制），让 agent 在提交之前能预览计划。
- 提供 `--yes` / `--force` 来跳过确认，同时为人类保留安全的默认行为。

## 可预测的结构

- 在任何地方都使用一致的模式，例如 `resource` + `verb`：如果已有 `mycli service list`，那么 `mycli deploy list` 和 `mycli config list` 应当遵循同样的形态。

## 成功时的输出

- 成功时，返回对机器有用的数据：ID、URL、耗时。纯文本就可以；避免只依赖装饰性输出。

```text
deployed v1.2.3 to staging
url: https://staging.myapp.com
deploy_id: dep_abc123
duration: 34s
```

## 评审现有 CLI 时

- 检查：非交互式路径、分层帮助、`--help` 上的示例、stdin/管道的处理、带调用示例的错误消息、幂等性、dry-run、绕过确认的标志、一致的命令结构、结构化的成功输出。
