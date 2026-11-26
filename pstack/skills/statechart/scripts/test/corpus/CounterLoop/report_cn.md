# CounterLoop

TLC 到达了 3 个状态和 3 步。图中绘制了 3 个状态和 3 条箭头。

## ReachesThree 被打破
TLC 的反例在 tlc-ReachesThree.out 中。它有 3 步：
0. 0
1. Up: 1
2. Up: 2
3. Back: 1
该运行永远在 "1" 和 "2" 之间循环。

## 成立的规则
NoDeadlock
