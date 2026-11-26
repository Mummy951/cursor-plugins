### Worktree 与模拟器清理

1. 快照并审计。记录 `df -h /`，然后运行 `scripts/worktree-audit.sh [-b <pr-base-ref>] [repo-path]`（principle-build-the-lever）。不带 `-b` 时它针对 `origin/HEAD` 检查 `MERGED`，所以当 PR 面向另一个分支时传入 `-b origin/<branch>`。它从 `git worktree list` 读取路径，绝不手工输入，因为手工输入的 `myrepo-worktrees/x` 会漏掉位于 `.cursor/worktrees/myrepo/x` 的 worktree（principle-encode-lessons-in-structure）。它按大小、存在时间、合并状态、未提交工作、PR 状态以及最近触碰它的聊天对每个 worktree 分类，然后建议一个分类桶（bucket）。transcript 扫描很慢，放到后台运行。
2. 分类桶只是建议，不是许可。被固定的（pinned）和活跃的聊天才是真实产物（principle-prove-it-works）。从用户或侧边栏获取该集合，并交叉核对每个候选。这个杠杆曾把用户固定的 worktree 标记为 `safe`，所以以固定集合为准。
3. 删除前先验证使用情况。对每一行 `verify-recent-chat` 或任何你存疑的对象，派出一批 subagent 去阅读 transcript，报告该聊天是否被固定或仍在进行，以及它涉及哪些 worktree（principle-guard-the-context-window，transcript 数据量很大）。一个被固定的聊天会通过后台 subagent 把 arena 和 repro 树派生到兄弟 worktree 中，即使它们的名字从未出现在侧边栏，这些树也在使用中。根据 Subagents 章节为每个 subagent 选择合适的推理级别。
4. 遇到不可逆损失时暂停。`wip:N` 表示 N 处已跟踪的未提交修改。先展示 diff 并获得决定，因为删除一个干净的 worktree 可以从其分支恢复，但未提交的工作就没了。`scratch:N` 是未跟踪的临时文件，可以安全丢弃，但要列出文件名。按照 Autonomy，干净、已合并且未在使用中的可以继续。`wip` 和使用中的要暂停。
5. 清理已确认的集合。对每个路径执行 `git worktree remove --force <path>`。如果目录因被忽略的构建产物而残留，就用 `rm -rf` 删除它，然后运行 `git worktree prune`。分支引用会保留，所以不会丢失提交。用 `df -h /` 确认并重新列出。
6. 模拟器及其他回收手段。模拟器通常是下一个最大的收益点。`xcrun simctl --set testing delete all`（XCTestDevices 克隆）、`xcrun simctl delete unavailable`，以及 `xcrun simctl runtime list` 然后对旧运行时执行 `runtime delete <id>`。需要时还可以：Xcode 的 `DerivedData` 和 `iOS DeviceSupport`、`~/Library/Application Support/Cursor`（`state.vscdb.backup`，以及 `snapshots/roots/<root>`——当某个 `<root>` 以你作为工作区打开的文件夹命名时会急剧膨胀）、包缓存（pnpm、uv、brew、yarn）。只清理用户没有说要保留的缓存。

这是唯一一个会删除用户状态、却没有 code review 来捕捉失误的 playbook，所以上面的关卡就是 review。

**回复：** 前后的 `df -h /` 及回收的空间、被清理的 worktree，以及每个保留项的一行原因（被哪个聊天使用，或有未提交的工作）。
