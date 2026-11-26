# 交接

交接是信息在节点之间流动的唯一方式。worker 产出交接；planner 阅读它们并做出决策。没有共享分支，没有 status API，没有跨兄弟节点的闲聊。正是这种统一性让整棵树在没有全局协调的情况下保持运转。

脚本会指示每个生成的 agent 以一条结构化的最终消息结尾（status、branch、summary、notes、follow-ups）。确切格式见 `prompts/*.md` 中的模板。脚本会附上可追溯性头信息，将最终消息逐字保存到 `<workspace>/handoffs/<task-name>.md`。不要丰富或净化内容；planner 需要 worker 未经过滤的原话。

## 度量（worker 交接）

worker 在 `## Measurements` 下自我报告；格式见 `prompts/worker.md`。当某个任务声明了 `measurements[]` 时，脚本会在 worker 的分支上重新运行每条命令，并将数值偏差 >10% 或单位不匹配标记到 `attention.log`；worker 仍会进行交接，由 planner 决定是否重新生成。编写细节见 `references/planner.md` → `measurements[]`。

## 阅读交接

对于每一份新的 `handoffs/*.md`：

1. 非 `success` 的 **Status**：决定是重试、修复，还是通过后续任务进行澄清。
2. **Branch**：记录下来；如果另一个任务需要在其基础上继续构建，就引用它。
3. **What I did**：当作事实看待，但快速浏览一下是否有与你预期不符的说法。
4. **Notes / concerns / deviations / findings / thoughts / feedback**：内容最丰富的一节。每一条要点都可能成为新任务。worker 关于范围划分或任务清晰度的反馈尤其有价值：它告诉你计划中的 prompt 是否物尽其用。
5. **Suggested follow-ups**：候选任务。接受、拒绝或合并。

`Status: blocked` 是单个任务的死胡同；planner 会重试、修复或澄清，整棵树继续运转。只有当在整个树上继续生成只会浪费精力时，才改用 Andon（见 `references/planner.md` → Failure recovery）。

## 合成失败交接

当 worker 死亡却没有写下自己的交接（cap-hit、OOM、tool-error、网络中断、未捕获的 SDK 错误）时，脚本会写入 `handoffs/<task>-failure.md`，让 planner 看到一份事后分析而非一片沉默。随后循环会带着 checkpoint 同步返回退出码 1，以便 planner 立即做出反应。

```markdown
<!-- orchestrate failure handoff
task: <name>
branch: <actual or placeholder branch>
agentId: bc-...
runId: run-...
failureMode: cap-hit | oom | network-drop | tool-error | unknown
terminatedAt: <iso>
-->

# <name> — failure handoff

Status: error (cloud agent terminated without writing a handoff)
Failure mode: cap-hit | oom | network-drop | tool-error | unknown
Cloud agent: bc-...
Started: <iso>
Terminated: <iso>
Duration: <ms>
Last activity: <iso> — <status text from state.json lastUpdate>
Last tool call: <name from SDK stream tail, or (unknown)>
Branch: <branch>
SDK error: <truncated error text>

## Suggested next steps
- <one bullet per option, planner picks>
```

分类器启发式规则：
- 当持续时间为 70–80 分钟且运行处于 terminal-error 时为 `cap-hit`
- 当输出或 SDK 错误包含 `out of memory` / `OOMKilled` / `exit code 137` 时为 `oom`
- 当 SDK 错误匹配 `fetch failed` / `ETIMEDOUT` / `ECONN` / `socket` / `dns` / `disconnect` 时为 `network-drop`
- 当 SDK 错误提到 `tool_use_failed` 或 `tool-error` 时为 `tool-error`
- 其他情况为 `unknown`

按模式划分的默认重试策略：
- `cap-hit` / `oom`：以更小的范围重试
- `network-drop`：原样重试（视为瞬时故障）
- `tool-error`：换一个 `model` 重试
- `unknown`：原样重试一次，然后放弃

对同一任务重试 2 次后，除非你有具体证据表明下一次重试会成功，否则宁愿放弃（从 `plan.json` 中移除，围绕它重新规划）而不是进行第 3 次尝试。

## 完成但无交接的旁路文件

当某次运行以 `status=finished` 结束但正文没有 `## Status` 标题时，脚本会在原始 `<task>.md` 旁边写入 `handoffs/<task>-finished-no-handoff.md`。将原始正文视为 worker 的意图；看起来可恢复就重试，不可恢复就放弃。

## 下游 prompt 中的上游交接

worker 位于兄弟分支上，**在运行时无法读取彼此的分支**。如果下游任务依赖上游任务的产出，planner 必须代为转达。

脚本负责转达。当生成一个 `dependsOn` 包含已交接任务的任务时，它会将每份上游交接正文粘贴到下游 prompt 中。在生成前用 `bun cli.ts prompt <workspace> <task>` 预览。

带来的影响：

- `dependsOn` 具有语义意义，而不只是一个调度闸门。只要下游任务需要上游的发现，就使用它，即使 Git 层面的排序并不严格要求如此。
- 未声明 `dependsOn` 却又需要上游上下文 = worker 只能靠猜。
- 扇入过多会膨胀 prompt 体积。如果变得难以处理，就把摘要下推到上游交接中，而不是让下游 prompt 变得臃肿。
- 交接是逐字呈现的：马虎的 `## What I did` 章节会污染每一个下游任务。这种格式是共享上下文的公地；请尊重它。

## 产出你自己的交接（subplanner）

subplanner 的最终消息就是它交给父级的交接。将子任务的结果向上聚合；不要直接转发原始的子交接。父级拥有更全局的上下文，但局部细节更少。

| 字段 | 规则 |
|-------|------|
| `Status` | 仅当每条验收标准都满足时才为 `success`。有任何子任务为 partial 则为 `partial`。仍有硬性阻塞则为 `blocked`。 |
| `Branch` | 你要向上交接的实际交付分支，而不是你的记账分支（`orch/<parent-rootSlug>/<your-name>`）。通常是子树中最后一个 merge 任务的输出。在孤儿恢复之后，或者某个集成 worker 合并进了子任务的分支，交付物可能是那个子分支。下游任务会在你这里命名的任何东西之上构建，所以要明确写出真正的交付物。 |
| `What my subtree did` | 每个有意义的工作切片一条要点，而不是每个子任务一条。父级关心的是工作，不是你的组织架构。 |
| `Notes / concerns` | 把兄弟子树或根节点可能冲突的任何事情都摆出来。对真实风险保持沉默比多余的琐事更糟。 |
| `Suggested follow-ups` | 属于你父级范围的任务，不是你自己的。 |

## 验证者交接

verifier 的最终消息是对某一目标任务验收标准的裁决，而不是实现总结。

```markdown
## Verification
<one of: live-ui-verified | unit-test-verified | type-check-only | verifier-blocked | verifier-failed>

## Target
`<target-name>` on branch `<target-branch>`

## Branch
`<verifier-branch>` (or "(no branch)" if you committed nothing)

## Execution
- <command run> → <outcome>
- <test suite> → <pass/fail counts>
- <manual repro step> → <observed behavior>
(list every meaningful thing you actually ran; this section is what distinguishes a real verification from pattern-matching)

## Findings
Per acceptance criterion:
- [x] <criterion text>: <evidence> (met | not met | n/a)
Other findings (severity-ordered):
- (high) <finding>: evidence
- (med) <finding>: evidence
- (low) <finding>: evidence

## Notes & suggestions
- <anything the planner should know: flaky tests, adjacent issues noticed, suggestions for follow-up tasks>
```

`## Verification` 会被脚本解析并持久化到*目标*任务的状态行（`state.json` 中的 `tasks[].verification`），这样运行后的分类器就能按质量对“已修复并验证”进行分桶，而不是把所有非失败都等同看待。权威定义见 `prompts/verifier.md`。简版如下：

| 值 | 含义 | planner 的响应 |
|---|---|---|
| `live-ui-verified` | verifier 现场复现了 bug，并确认修复消除了它。 | 当作已交付予以信任；除非出现其他发现，否则无需后续任务。 |
| `unit-test-verified` | 针对性测试覆盖了改动的代码路径并通过。 | 对非 UI bug 可以接受。对 UI bug，待环境允许时用 `live-ui-verified` 补一轮。 |
| `type-check-only` | 仅类型检查 / 构建通过。 | 较弱；只足以应对纯类型改动。任何涉及行为的改动都需要更强的验证。 |
| `verifier-blocked` | verifier 遇到环境故障（Docker 限流、端口、缺失凭据）。 | 修复可能正确但未经证实。环境恢复后重新生成 verifier，或升级处理。不算作已验证。 |
| `verifier-failed` | verifier 运行了，但修复没有解决 bug。 | 用后续修复任务处理，不要自动重新生成。 |

worker 和 subplanner 也可以写一行 `## Verification` 来自我报告自己的证据。之后的 verifier 会在同一目标行上覆盖这份自我报告。

脚本也接受旧式的 `## Verdict pass | fail | inconclusive` 形式，并将其迁移为最保守的新值：`pass` → `type-check-only`、`fail` → `verifier-failed`、`inconclusive` → `verifier-blocked`。新的 verifier prompt 直接输出 `## Verification`。

在 `plan.json` 中显式发布 verifier：

```json
{
  "name": "frontend-toggle",
  "type": "worker",
  "scopedGoal": "Add a Settings → Appearance toggle that persists `editor.experimentalDarkMode` through the existing settings service.",
  "pathsAllowed": ["packages/ui/src/settings/**"],
  "acceptance": [
    "Toggle renders in Settings → Appearance",
    "Toggling on persists `editor.experimentalDarkMode=true`",
    "Reloading Settings shows the persisted toggle state"
  ],
  "verify": "## Setup\n- Start the Settings UI dev environment with the existing repo workflow.\n\n## Automated\n- Run the focused Settings UI test that covers Appearance settings persistence.\n\n## Manual\n- Open Settings → Appearance, toggle dark mode on, reload Settings, and confirm the toggle remains on.\n\n## Gotchas\n- Make sure the test account starts with no existing `editor.experimentalDarkMode` override."
},
{
  "name": "verify-frontend-toggle",
  "type": "verifier",
  "verifies": "frontend-toggle",
  "scopedGoal": "Verify the Settings → Appearance toggle works against every acceptance criterion by running the UI test or manually exercising the screen.",
  "acceptance": ["Verification section includes execution evidence for all frontend-toggle acceptance criteria"]
}
```

## 合并也是任务

因为 planner 不写代码，合并通过任务来完成。发布一个 worker，其 `scopedGoal` 写明两个分支和冲突解决策略，并用 `dependsOn` 同时卡住两个兄弟任务：

```json
{
  "name": "merge-frontend-and-theme",
  "type": "worker",
  "scopedGoal": "Merge `orch/dark-mode/frontend-toggle` into the current branch. On conflict in `packages/ui/src/settings/Settings.tsx`, prefer frontend-toggle's hook wiring. After merge, verify `pnpm -w typecheck` passes.",
  "startingRef": "orch/dark-mode/theme-system",
  "dependsOn": ["frontend-toggle", "theme-system"],
  "acceptance": ["Merge committed with both parents in history", "pnpm -w typecheck passes"]
}
```

它的交接会告诉你合并是否成功、冲突是否不直观、验收是否成立。像对待其他任何交接一样对待它。

## 持续运转

只要子任务还可能产出交接，planner 就不算真正“完成”。如果某份交接在你已经总结或向上发送交接之后才到达：

- 阅读它。
- 如果它改变了你的结论，就明说：“又有一个 worker 带着 X 回来了，需要修订”。
- 如有需要，发布后续任务。
- 产出一份新的交接 / 总结。

只有当你决定停止发布任务，且每一个进行中的任务都已终结时，才硬性停止。
