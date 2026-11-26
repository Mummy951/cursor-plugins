# 持续学习（Continual Learning）

根据会话记录（transcript）的变化，自动且增量地让 `AGENTS.md` 保持最新。

该插件结合了以下部分：

- 一个 `stop` 钩子（hook），用于决定何时触发学习。
- 一个 `continual-learning` skill，用于编排学习流程。
- 一个 `agents-memory-updater` subagent，用于挖掘新增或变化的会话记录并更新 `AGENTS.md`。

它通过以下方式避免产生嘈杂的重写（noisy rewrites）：

- 首先读取现有的 `AGENTS.md`，并就地更新匹配的项目符号（bullet）。
- 仅处理新增或变化的会话记录文件。
- 仅写入普通的项目符号（不包含证据/置信度等元数据）。

## 安装

```bash
/add-plugin continual-learning
```

## 工作原理

在符合条件的 `stop` 事件上，钩子可能会发出一条 `followup_message`，要求 agent 运行 `continual-learning` skill。

该 skill 被标记为 `disable-model-invocation: true`，因此在常规的模型调用期间不会被自动选中。当它确实运行时，它会将整个记忆更新流程委托给 `agents-memory-updater`。

钩子将本地运行时状态保存在：

- `.cursor/hooks/state/continual-learning.json`（节奏状态）

更新器使用一个增量的会话记录索引，位于：

- `.cursor/hooks/state/continual-learning-index.json`

## 触发节奏

默认节奏：

- 至少完成 10 个回合（turn）
- 距上次运行至少 120 分钟
- 会话记录的修改时间（mtime）必须比上一次运行时有所前进

试用模式默认值（在本插件的钩子配置中启用）：

- 至少完成 3 个回合
- 至少 15 分钟
- 24 小时后自动过期，随后回退到默认节奏

## 可选的环境变量覆盖

- `CONTINUAL_LEARNING_MIN_TURNS`（或旧版 `CONTINUOUS_LEARNING_MIN_TURNS`）
- `CONTINUAL_LEARNING_MIN_MINUTES`（或旧版 `CONTINUOUS_LEARNING_MIN_MINUTES`）
- `CONTINUAL_LEARNING_TRIAL_MODE`（或旧版 `CONTINUOUS_LEARNING_TRIAL_MODE`）
- `CONTINUAL_LEARNING_TRIAL_MIN_TURNS`（或旧版 `CONTINUOUS_LEARNING_TRIAL_MIN_TURNS`）
- `CONTINUAL_LEARNING_TRIAL_MIN_MINUTES`（或旧版 `CONTINUOUS_LEARNING_TRIAL_MIN_MINUTES`）
- `CONTINUAL_LEARNING_TRIAL_DURATION_MINUTES`（或旧版 `CONTINUOUS_LEARNING_TRIAL_DURATION_MINUTES`）

## AGENTS.md 中的输出格式

记忆更新器仅写入：

- `## Learned User Preferences`
- `## Learned Workspace Facts`

每一项都是一个普通的项目符号。

## 许可证

MIT
