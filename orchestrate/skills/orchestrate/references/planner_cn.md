根规划器与子规划器的操作手册。Dispatcher 请阅读 `dispatcher.md`。

# 规划器（Planner）

根规划器与子规划器的行为方式相同。根规划器向用户汇报；子规划器向其父级汇报。

## 前置条件

在行动之前先加载：

1. 加载 [cursor-sdk plugin](https://github.com/cursor/plugins/tree/main/cursor-sdk)，用于鉴权、派生（spawn），以及区分 `CursorAgentError` 与 `RunResult.status === "error"`。

脚本要求 `bun` 在 PATH 中。在该 skill 的 `scripts/` 目录内用 `bun install` 安装依赖。

在计划或状态结构发生变化后，在 `scripts/` 中用 `bun run generate-schemas` 从 `scripts/schemas.ts` 重新生成 `schemas/*.json`。

Slack 可见性使用 `SLACK_BOT_TOKEN`。所需权限范围（scopes）：

- `chat:write` —— 发布和编辑消息。
- `chat:write.customize` —— 为机器人消息设置自定义用户名和图标。
- `chat:write.public` —— 无需先加入频道即可在公开频道发帖。
- `files:write` —— 上传交接产物。
- `files:read` —— 与 `files:write` 搭配，用于上传 v2 流程。
- `reactions:read` —— 监视 kickoff 消息上的 Andon `:rotating_light:` 表情回应。
- `channels:history` —— 通过 `conversations.replies` 读取线程回复。如果运行线程位于私有频道，则改用 `groups:history`。

可选：

- `users:read.email` —— 尽力根据 dispatcher 的 git 邮箱查找名字（first-name）。没有它时，需显式传入 `--dispatcher-name`。

在这些权限范围生效之前，Slack 调用会失败，并在 `attention.log` 中出现 Slack 的 `missing_scope` 错误。运行仍会继续，因为 git 和磁盘才是权威来源。

## 事实来源（Source of truth）

Git 和磁盘是底层基础。

- `plan.json` 承载任务图、Slack 配置、仓库 URL 和模型选择。
- `state.json` 承载任务状态、agent/run id、分支名和 Slack 消息时间戳。
- `handoffs/*.md` 承载 worker 和 verifier 的输出。
- `attention.log` 承载操作员可见的失败和决策。

Slack 用于人工可见性，而非任务状态。脚本会发布一条 kickoff 消息，在该线程中镜像任务状态，并读取 kickoff 消息上的 `:rotating_light:` 作为 Andon。kickoff 之后，Slack 的写入都保留在运行线程内；适配器对这些写入要求提供 `threadTs`。如果 Slack 宕机，编排的正确性不会改变。

Orchestrate 负责 Slack 状态镜像、Andon 和评论重试队列。Agent 仍然可以直接调用 MCP 来处理 Linear、GitHub、Slack、Notion 及其他临时的外部工作。这些系统不是 orchestrate 的目的地。

## 阶段 1：发布任务

在 `<workspace>` 写入 `plan.json`；默认 workspace 是 `.orchestrate/<rootSlug>/`。

```json
{
  "$schema": "<path-to-orchestrate>/schemas/plan.schema.json",
  "goal": "<verbatim user goal>",
  "summary": "ship the dark-mode toggle end to end",
  "rootSlug": "dark-mode",
  "baseBranch": "main",
  "repoUrl": "https://github.com/example-org/example-repo",
  "tasks": [
    {
      "name": "frontend-toggle",
      "type": "worker",
      "scopedGoal": "Add a Settings UI toggle that flips `useDarkMode` in localStorage.",
      "pathsAllowed": ["packages/ui/src/settings/**"],
      "acceptance": ["Toggle renders in Settings > Appearance"]
    }
  ]
}
```

`summary` 面向 Slack 线程中的人；`goal` 是 agent 的完整上下文。当 `summary` 未设置时，kickoff 会回退到截断的 `goal`。

在首次 `run --root` 时，脚本使用 `plan.slackChannel` 进行 Slack kickoff，并写入 `plan.slackKickoffRef`。根计划通过 `kickoff --slack-channel`、`run --root --slack-channel` 或 `SLACK_CHANNEL_ID` 获得 `slackChannel`。子规划器继承这两个字段，以便整棵树都镜像到同一个线程中。

规划规则：

- 合并也是任务。发布一个 worker，其 `scopedGoal` 要说明要合并哪些分支、冲突处理意图和验证方式。
- 优先使用 worker，除非你能说清某个子规划器会做的分解方式。
- 一个 worker 可以承担很多工作。Worker 和 verifier 是拥有数小时运行时间的完整云端 agent：多文件切片、多步骤重构、完整的复现/修复/测试循环都能放进一次 spawn 中。每次 spawn 都会消耗云端 agent 运行时间、制造 Slack 噪音，并增加你自己的协调开销。默认使用更少、更宽的 worker；只有当某个切片确实独立或存在真实争用风险时，才采用更细的粒度。
- 默认使用 verifier。使用 `type: "verifier"` 和 `verifies: "<target-task-name>"`。
- 用 `verify` 描述具体的检查配方。Worker 将其读作目标行为；verifier 从其目标继承它。
- 仅对那些你希望作为独立 draft PR 交付的独立 worker 任务设置 `openPR: true`。
- 对定量断言添加 `measurements[]`。脚本会在交接后在 worker 分支上重新运行每条命令，并记录偏移（drift）。
- 保持扇入（fan-in）小。如果某个任务需要许多上游交接，先发布一个聚合 worker。
- 尽量减少路径重叠。当兄弟任务之间的归属很重要时，列出禁止路径。
- 将任务规格放在 `plan.tasks[]` 中。将共享产物放在 git 中，并通过路径引用它们。
- 对经由重试队列路由的 Slack 备注使用 `comment` CLI。仅对必须送达的消息使用 `--criticality required`。对于非 Slack 目的地，agent 直接调用相应的 MCP。

## 阶段 2：驱动 workspace

所有操作员操作都通过 `scripts/cli.ts` 进行。

```bash
bun <path-to-orchestrate>/scripts/cli.ts <subcommand> <workspace> [...]
```

`run` 会派生依赖已满足的待处理任务，等待交接，写入交接，并重复该过程，直到无法再取得进展。退出码 `0` 表示干净完成。退出码 `100` 表示一次计划好的检查点重启；重新运行同一命令。其他非零码意味着需要阅读 `state.json` 和 `attention.log`。

不要 detach `run`。该脚本是状态、交接、Slack 镜像、重试队列排空和 Andon 轮询的心跳。当它退出时，调用 `tree`。如果任何任务仍是 `pending` 或 `running`，再次运行该循环。

`state.json` 是事实来源。用 `tree`、`list` 和 `status` 检查它。

## 评论（Comments）

`comment` CLI 仅用于 Slack，且从不发布 kickoff。传入 `--task <name>` 以校验任务上下文并解析运行线程，或显式传入 `--thread-ts <ts>`。

示例：

```bash
bun cli.ts comment "worker-one is blocked on auth" --task worker-one --workspace .orchestrate/root
bun cli.ts comment "no-repro on the upstream report; need a Linear ticket filed before retrying" --thread-ts 1714500000.000100 --criticality required --workspace .orchestrate/root
```

`--workspace` 在使用 `--task` 时以及对于非操作员文件上传都是必需的。运行之外的操作员通过一个由当前用户拥有的、权限设为 `0600` 的 `~/.orchestrate/operator-mode` 文件来启用操作员模式。假定 worker 无法写入操作员的 OS 主目录。

必需的评论使用 `comment-retry-queue.json` 及现有的退避策略。

对于外部追踪系统（Linear、GitHub、呼叫值班寻呼），agent 直接调用相应的 MCP。Orchestrate 不路由这些系统。

## 失败恢复

脚本处理机械层面的存活性（liveness）。规划器处理语义层面的问题。

- 瞬时的 spawn 失败会在 `spawnTask` 内重试。
- 重启后的循环通过 `recoverRunning` 重新挂接到正在运行的任务上。
- `RunResult.status === "error"` 或交接受阻属于规划器的决策：重新派生、拆分、升级或放弃。
- 当上游失败时，下游任务保持 `pending`。修复上游后重新运行，或 `kill` 掉被放弃的下游工作。
- 子规划器在第一次尝试后从自身分支克隆，以便已提交的子状态和交接得以保留。
- `maxAttempts` 限制自动派生次数。只有当再次尝试是有意为之，才在任务定义中提高它。
- 计划好的检查点重启会在以 `100` 退出前提交状态和交接；重新运行同一命令。

### Andon

Andon 会暂停整棵树中的新派生。根规划器轮询 Slack kickoff 消息中的 `:rotating_light:`。子级通过 git 经由 `plan.andonStateRef` 和 `plan.andonStatePath` 读取缓存的根状态。

```bash
bun <path-to-orchestrate>/scripts/cli.ts andon raise --reason "<why>" --workspace <workspace>
bun <path-to-orchestrate>/scripts/cli.ts andon clear --workspace <workspace> [--note "<what changed>"]
```

`raise` 时必须提供 `--reason`。根规划器轮询 Slack kickoff 消息中的 `:rotating_light:`，并扫描最近一条匹配的 `🚨 ANDON RAISED ...: <reason>` 线程回复，然后将截断后的原因写入 `state.andon.reason`。子级通过 git 读取该缓存状态，因此无需自己调用 Slack 就能看到编排*为何*暂停。Andon 状态由操作员输入，上限 500 字符，并与 `state.json` 的其余部分处于同一信任圈内。

仅当继续派生会给整棵树带来垃圾结果时才触发 Andon：错误的上游输出、损坏的验收条件，或不可恢复的鉴权/基础设施问题。某个任务自身的卡点应写在其交接中，而非 Andon。

## 查找 agent

使用 `bun cli.ts tree <workspace>` 和 `bun cli.ts list <workspace>` 查看血缘、状态和 agent ID。不要依赖云端 agent 的显示标题。

`syncStateToGit` 默认开启，以便远程观察者可以从 git 读取 `state.json` 和交接。当这些产物应保持本地时，将其设为 `false`。

```bash
bun cli.ts crawl <local-repo-path> <root-branch> <root-slug>
bun cli.ts kill-tree <local-repo-path> <root-branch> <root-slug> [-y] [--agent-id <id>]
```

这两个命令都会遍历 `.orchestrate/<rootSlug>/state.json`；每个子规划器行都会递归进入 `orch/<rootSlug>/<subplanner-name>`。
