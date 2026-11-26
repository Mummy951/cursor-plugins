# DieHard

TLC 到达了 16 个状态和 96 个步骤。该图绘制了 16 个状态和 58 条箭头。
该图未绘制停留在同一图状态中的 38 个步骤。

## NotSolved 被打破
最短路径有 6 个步骤：
0. big=0 small=0
1. FillBig: big=5 small=0
2. BigToSmall: big=2 small=3
3. EmptySmall: big=2 small=0
4. BigToSmall: big=0 small=2
5. FillBig: big=5 small=2
6. BigToSmall: big=4 small=3

## 成立的规则
TypeOK, NoDeadlock
