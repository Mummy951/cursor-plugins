# 派生任务

`plan.json` 条目与脚本派生的云端 agent 之间的契约。机制（认证、`CursorAgentError` 与 `RunResult.status === "error"` 的区别）见 `cursor-sdk/SKILL.md`。先读那个。

## 分支命名

云端 agent 拥有自己的工作分支。`state.json` 以确定性的占位符（`orch/<rootSlug>/<task-name>`）起始，使派生前的状态可读，然后在交接后将其替换为 `Run.git.branches[].branch` 上报的分支。强制使用 kebab-case（`orchestrate.ts` 中的 `TASK_NAME_RE`），使任务名仍可直接用于文件系统路径和提示文本而无需转义。没有自动管理的集成分支。各分支独立存在，直到某个合并任务将它们整合（见 `handoffs.md` → "合并也是任务"）。

不要要求 worker 创建或重命名分支以匹配占位符。如果下游任务需要上游任务的代码，就依赖那个上游任务，以便脚本等待交接并使用记录下来的实际分支。

## Agent 命名

云端 agent 在 `Agent.create` 时会被赋予一个 `name`，这样 Cursor agent 列表（`cursor.com/agents`、IDE agent 列表）就能将同一次 orchestrate 运行归为一组，并在数十个并发子任务中保持可读。

| 派生位置 | Agent 名称 |
|------------|------------|
| 根 planner（`cli.ts kickoff`） | `<目标的第一行，最多 100 个字符>` |
| Worker / subplanner / verifier（`spawnTask`） | `<rootSlug>/<taskName>` —— 回显任务的分支，但不含 `orch/` 前缀 |
| 模型目录探测（`probe-models`） | `probe: <modelSlug>` |

服务端将名称限制为 100 个字符，并拒绝空值或纯空白值；辅助函数会处理这两种情况。当省略 `name` 时，云端后端会根据第一条提示自动生成名称，因此这纯粹是为了提升可读性 —— 去掉名称不会改变行为。

## 起始引用与依赖

| 字段 | 控制内容 | 默认值 | 搭配使用 |
|-------|----------|---------|-----------|
| `startingRef` | 云端 agent 从哪个分支克隆 | `plan.baseBranch` | 依赖另一个任务的工作时搭配 `dependsOn` |
| `dependsOn` | 任务何时被允许派生 | `[]` | 脚本在交接后从 `Run.git` 记录上游分支 |

单独使用 `startingRef` 而不搭配 `dependsOn` 会得到一个时间点快照：即派生时该分支上存在的任何提交，也可能什么都没有。除非你确实想"从当前顶端开始，即使是空的"，否则应将两者搭配使用。

Verifier 默认将 `startingRef` 设为目标任务的分支，并自动将目标加入 `dependsOn`；planner 无需显式设置这两者。

任何任务都可以设置 `verify`：一个 Markdown 格式的计划（设置、自动化、手动、注意事项）。Worker 将其视为目标规格；verifier 继承目标的 `verify` 作为自己的方案。

## 派生设计决策

脚本调用 `Agent.create` 时带有两个刻意设置的默认值：

- **PR 按任务选择性开启。** 云端 agent 创建调用中的 `autoCreatePR` 与任务的 `openPR` 标志一致（默认 false）。服务端 `cloud_agent_pr_control` 开关目前使该标志成为空操作，因此真正的机制在 worker 提示中：当 `openPR: true` 时，worker 会被指示在推送后通过 ManagePullRequest 工具针对 `plan.prBase ?? plan.baseBranch` 开启一个草稿 PR。Subplanner 和 verifier 从不开启 PR；它们交接给父任务。由任务驱动的 PR 给 planner 一个明确的保证：该任务以独立的 pull request 交付。
- **PR 基线 vs. worker 起始引用。** 对于未指定自身起始引用的 worker，`plan.baseBranch` 就是其起始引用。`plan.prBase`（可选，默认为 `baseBranch`）是开启 PR 的 worker 提交 PR 的目标。当你希望 worker 从 planner 的分支继承 planner 侧的设置、但仍针对 `main` 开启 PR 时（这样每个 worker PR 无需 planner 分支先落地即可合并），就将它们分开。对于 worker PR 堆叠在 planner 分支上的经典模式，不设置 `prBase`。
- **没有共享的集成分支。** 在合并任务整合之前，每个任务都是一座孤岛。自动管理的集成分支会把 planner 层级的编码决策偷偷塞进基础设施，违反核心原则 #1。

## 任务提示契约

脚本根据 `scopedGoal`、`pathsAllowed`、`pathsForbidden`、`acceptance`、`startingRef`、`type` 和 `openPR` 渲染提示。如果提示与你的意图不符，就修正 plan 条目。不要修改提示模板。每条派生提示都会告诉 agent：

- 它是隔离的。不与其他 agent 通信。
- 提交到当前云端 agent 分支并推送。不重命名分支、不合并、不 rebase。仅当任务设置了 `openPR: true` 时才开启草稿 PR（仅限 worker；subplanner 和 verifier 从不开启 PR）。
- 最终消息是按 `handoffs.md` 中的结构写的交接信息。这是 planner 唯一会读取的内容。

Subplanner 的提示还以 `/orchestrate` 开头，以便 agent 启动时自动加载该 skill。

Worker 无法在运行中提出澄清问题。`scopedGoal` 描述不足会导致无声的偏移。编写每个任务时都要假设你再也没有机会去纠正它。

云端 agent 的虚拟机可能会屏蔽环境变量值作为提示注入防御，因此不要依赖环境变量来共享多个 agent 都必须使用的数据。改用 planner 编写的产物模式：将文件提交到基础分支，并在 `scopedGoal` 中通过路径引用它，使克隆能获取到该文件。切勿将凭据粘贴到 `scopedGoal` 中；它会被发送给模型提供方，并且在状态同步开启时可能最终进入 git 历史。

## Slack 可见性

当设置了 `SLACK_BOT_TOKEN` 时，Slack 流量由脚本负责。Agent 不驱动生命周期。脚本将启动帖发布到 `plan.slackChannel`，将结果记录在 `plan.slackKickoffRef` 中，在该帖中同步任务状态消息，并读取启动消息上的 `:rotating_light:` 以触发 Andon。

派生提示仍包含 Slack 部分，因为 worker 可能需要留下备注：

- 当沉默会隐藏有用上下文时，`comment "<note>" --thread-ts <run-thread-ts> --workspace <workspace>` 会通过重试队列发布一条备注。`--task <name>` 搭配 `--workspace` 也可接受；CLI 会校验任务并在运行帖中发布。
- 对于 Linear、GitHub 或其他外部系统，直接调用相应的 MCP。Orchestrate 的 `comment` CLI 仅适用于 Slack。

如果 Slack 备注发布失败，继续工作，并在交接中说明发生了什么。对于下游提示的组装，磁盘交接仍是权威来源。

## 跟踪与恢复

每次成功派生后，脚本会将 `agentId`、`runId` 和 `parentAgentId` 持久化到 `state.json`，以便后续重新运行时可通过 `Agent.getRun` 重新挂载并从磁盘读取血缘关系。重启时，身份信息不完整的行（恰好只有 `agentId` / `runId` 中的一个）会被标记为 `error` 并附上说明性备注。重命名并重新派生，或将其清除。

每次交接后，脚本都会核对依赖它的 verifier 的 `startingRef`。实际 worker 分支来自交接正文中的 `## Branch` 行 —— 对于 worker 运行，SDK 会将 `Run.git.branches[].branch` 留空，因此正文才是权威来源。任何 `verifies` 指向刚交接完成的任务、且 `startingRef` 仍为 `orch/<rootSlug>/<task>` 占位符的 verifier，都会更新为那个真实分支。planner 编写的 `startingRef` 覆盖优先；每次传播都会记录到 `attention.log`。加载时还会扫过已交接的行，使从磁盘恢复的状态在下一次派生前收敛。

## 血缘关系

每个 `state.tasks[]` 行的 `parentAgentId` 是派生时来自 `plan.selfAgentId` 的、发起派生的 planner 的云端 agent id。`kill-tree --agent-id` 会沿这些父级链接向下遍历。如果 planner 从未设置 `selfAgentId`，子任务将获得 `parentAgentId: null`，在限定范围的 kill 中会跳过该子树；省略 `--agent-id` 则取消整棵树。

派生模板使用子任务的 id 作为 `{{selfAgentId}}`，父任务的 id 作为 `{{parentAgentId}}`。`Agent.create` 返回的 id 在 `send()` 之前就有效；云端客户端在创建时发送该 id，并拒绝不匹配的服务器响应。
