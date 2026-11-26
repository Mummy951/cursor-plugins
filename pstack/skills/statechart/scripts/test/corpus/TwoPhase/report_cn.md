# TwoPhase

TLC 到达了 76 个状态和 192 个步骤。图表绘制了 10 个状态和 16 条箭头。
图表未绘制停留在同一图表状态中的 59 个步骤。
步骤 RMPrepare 每次只改变 r1 和 r2 中的一个。
步骤 RMChooseToAbort 每次只改变 r1 和 r2 中的一个。
步骤 RMRcvAbortMsg 每次只改变 r1 和 r2 中的一个。
步骤 RMRcvCommitMsg 每次只改变 r1 和 r2 中的一个。

## TCConsistent 不成立
最短路径有 5 个步骤：
0. Manager init, r1 working, r2 working
1. RMChooseToAbort: Manager init, r1 working, r2 aborted
2. RMPrepare: Manager init, r1 prepared, r2 aborted
3. TMRcvPrepared: Manager init, r1 prepared, r2 aborted
4. TMCommit: Manager done, r1 prepared, r2 aborted
5. RMRcvCommitMsg: Manager done, r1 committed, r2 aborted

## 成立的规则
TPTypeOK, NoDeadlock
