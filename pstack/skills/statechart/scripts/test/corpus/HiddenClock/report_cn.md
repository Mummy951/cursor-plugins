# HiddenClock

TLC 到达了 2 个状态和 2 个步骤。该图绘制了 2 个状态和 1 条箭头。
该图未绘制停留在同一图状态中的 1 个步骤。
使用 VIEW 时，TLC 会为其视为相同的每组状态保留一个代表。箭头可能指向该代表，而不是该步骤实际到达的状态。

## EndsAtZero 被打破
TLC 的反例在 tlc-EndsAtZero.out 中。它有 1 个步骤：
0. x=0 clock=0
1. Tick: x=1 clock=1
运行永远停留在 "x=1 clock=1"。
