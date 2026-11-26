### Autopilot-stack

运行 Autopilot-full playbook，但不合并。

1. **取代第 5 步，停在已验证。** 当每个 PR 在其当前 head 已验证且绿色时停止，然后将 PR 交给操作者。交接之后，没有操作者的放行任何人都不得推送。每次 loop tick 提示都点名本文件，以便每次 tick 都能看到不合并规则。
2. **仅当某个 PR 建立在另一个 PR 之上时才堆叠它。** 这是第 3 步的唯一例外。按 `playbooks/opening-a-pr.md`，将该 PR 基于另一个 PR 的分支而非主干。取代第 2 步中 rebase 到主干的做法，每个 PR rebase 到其父 PR 的分支，而堆叠的根 rebase 到主干。较低 PR 上的新 head 会重堆叠其上方的 PR，而重堆叠后 `git patch-id` 未变的 PR 保留其裁决。

**回复：** 列出每个 PR 及其裁决，取代已合并的内容。
