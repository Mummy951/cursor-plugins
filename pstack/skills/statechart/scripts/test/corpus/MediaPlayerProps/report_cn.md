# MediaPlayerProps

TLC 到达了 12 个状态和 64 个步骤。图表绘制了 7 个状态和 13 条箭头。
步骤 a call comes in 同时改变了 Playback 和 Volume。
步骤 the track ends 同时改变了 Playback 和 Volume。
步骤 you press stop 同时改变了 Playback 和 Volume。

## EventuallyPlays 不成立
TLC 的反例在 tlc-EventuallyPlays.out 中。它有 0 个步骤：
0. Playback Stopped, Volume Normal volume
该运行永远停留在 "Playback Stopped, Volume Normal volume" 状态中。

## QuietWhenStopped 不成立
TLC 的反例在 tlc-QuietWhenStopped.out 中。它有 1 个步骤：
0. Playback Stopped, Volume Normal volume
1. you turn it up: Playback Stopped, Volume Loud

## 成立的规则
TypeOK, NoDeadlock, EventuallyStops, StaysTyped
