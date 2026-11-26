循环卫生：
- 在前台运行 `bun cli.ts run{{rootFlag}} <workspace>`。Shell 默认会将循环放到后台运行，并会在你的回合结束时中断心跳（heartbeat）。
- 退出码 100 是计划内的检查点重启，不是错误。立即重新运行同一命令；它会从已提交的 `state.json` 恢复。
- 非空错误集上的退出码 1 表示轮到你了。循环因某个任务崩溃而退出；脚本已经为每个死亡的 worker 写入了合成的 `handoffs/<task>-failure.md`，并为那些在没有结构化交接的情况下结束的 worker 写入了相应的 `handoffs/<task>-finished-no-handoff.md`。仍在运行中的 worker 会继续执行；下一次 `run` 会通过 `recoverRunning` 重新挂载。
- `run` 返回后，调用 `tree`。如果任何任务仍处于 `pending` 或 `running` 状态，请再次循环。
- 当此工作区中仍存在非终态任务时，不要结束你的回合。

处理失败交接：
对于每个 `status: "error"` 且具有匹配的 `handoffs/<task>-failure.md` 的任务，读取 `Failure mode` 一行并做出决定：
- `cap-hit` 或 `oom`：以更小的范围重试（拆分为更窄的任务、更严格的 `pathsAllowed`、更精简的 `scopedGoal`）。
- `network-drop`：原样重试；视为暂时性故障。
- `tool-error`：使用不同的 `model` 重试。
- `unknown`：阅读 `Last activity` 和 `SDK error` 行；如果没有任何线索，则视为暂时性故障并原样重试；如果再次失败则放弃。
对于 `<task>-finished-no-handoff.md`，读取 `handoffs/<task>.md` 处的原始片段，判断该 worker 的意图是否可恢复；选择重试或放弃。
每次重试都会消耗一次额外的 cloud-agent 运行；请慎重决策。对同一任务重试 2 次后，除非你有具体证据表明下一次重试会成功，否则优先选择放弃（将该任务从 `plan.json` 中移除，围绕它重新规划），而不是进行第 3 次尝试。更新 `plan.json`，然后重新运行 `bun cli.ts run{{rootFlag}} <workspace>` 以继续。
