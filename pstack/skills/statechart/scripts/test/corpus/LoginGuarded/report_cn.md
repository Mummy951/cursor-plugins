# LoginGuarded

TLC 到达了 8 个状态和 11 个步骤。该图绘制了 7 个状态和 10 条箭头。
步骤 ThePasswordIsRight 同时改变 Session 和 Tries。
步骤 ThePasswordIsWrong 同时改变 Session 和 Tries。

## 成立的规则
TypeOK, SignedInWhileLocked, NoDeadlock
