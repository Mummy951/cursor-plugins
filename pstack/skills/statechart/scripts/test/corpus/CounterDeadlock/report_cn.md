# CounterDeadlock

TLC 到达了 4 个状态和 3 步。图表绘制了 4 个状态和 4 条箭头。
有 1 步被约束排除，它终止了某条规则的路径，但图表仍包含这一步。TLC 会在这样的步骤之后检查规则，但它本身并不执行这一步。
图表只标记 TLC 报告出的死锁。在 CONSTRAINT 或 ACTION_CONSTRAINT 下，一个在图表中没有后续步骤的状态，仍可能存在被约束排除在外的步骤，TLC 不会把它算作死锁。

## NotThree 被打破
最短路径有 1 步：
0. 0
1. Jump: 3

## NoDeadlock 被打破
最短路径有 3 步：
0. 0
1. Inc: 1
2. Inc: 2
3. Inc: 3
