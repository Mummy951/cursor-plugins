# CounterGuard

TLC 到达了 2 个状态和 1 步。图表绘制了 2 个状态和 1 条箭头。
TLC 以退出码 75 停止了绘图运行，因此图表只绘制了它在停止之前到达的那些状态。

## TypeOK 被打破
TLC 找到了一个反例，位于 tlc-TypeOK.out 中。

## 未检查的规则
Lookup, NoDeadlock
TLC 在检查这些规则之前就以退出码 75 停止了：
```
Error: Evaluating invariant Lookup failed.
Attempted to apply tuple
<<"zero", "one">>
to integer 3 which is out of domain.
Error: The behavior up to this point is:
State 1: <Initial predicate>
x = 0

State 2: <Next line 6, col 9 to line 6, col 18 of module CounterGuard>
x = 1

State 3: <Next line 6, col 9 to line 6, col 18 of module CounterGuard>
x = 2

Error: The error occurred when TLC was evaluating the nested
expressions at the following positions:
0. Line 12, column 11 to line 12, column 30 in CounterGuard
1. Line 12, column 11 to line 12, column 21 in CounterGuard
```
