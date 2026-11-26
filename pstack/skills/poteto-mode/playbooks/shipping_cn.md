### 交付

**你负责落地的内容。独立验证每个 PR，只从根部落地经过验证的连续序列，然后不要碰这个 stack。**

1. **确定 forge，然后在每个 PR 落地前独立验证它。** 默认使用 GitHub CLI (`gh`)。如果 `command -v origin` 成功且 Origin 能够解析该仓库，则所有 PR 操作都使用 `origin pr`。否则继续使用 `gh` 并记录这一回退。绝不要依赖 Graphite (`gt`)。每个 PR 配一个验证者，各自是一个 Cursor cloud agent，以用户的方式、使用匹配的 control skill（例如来自 `cursor-team-kit` 的 `control-ui` 或 `control-cli`），以 parent 对 head 运行变更，并在该 PR 上发布 `PASS`、`PASS+NOTES` 或 `FAIL`。只有来自非代码作者的 agent 的结论才算数，因为作者检查的是他原本想构建的东西。绿色的 CI 和通过的 bot review 都不是结论，因为两者都没有以用户的方式运行变更。
2. **只落地以底部为根的连续已验证序列。** 从最低的未合并 PR 向上走，在第一个没有 `PASS` 或 `PASS+NOTES` 的 PR 处停下。它下面的 PR 就是该序列，最上面那个是上限（ceiling）。位于未验证 PR 之上的已验证 PR 要等待，因为没有下面的代码它无法落地。而某个分支既不持有也不需要那段代码的 PR，可以移到 trunk 上独立落地。它需要先获得新的结论，因为旧的结论是在其下方有那段代码时跑出来的，而 trunk 没有那段代码。
3. **重新检查结论是否仍然描述当前代码。** 结论覆盖的是一个 patch，而链式变基会把它上面的每个 PR 移到一个新的 SHA，且不会让任何检查失败。在信任较旧的结论之前，先比较结论 SHA 处与当前 head 的 `git patch-id`。当 patch 仅在测试、文档或 lint 配置上不同时，如果某个 lane 运行的内容在两个 SHA 上构建结果除了噪声之外相同，那么该 lane 的结果仍然有效。噪声是指同样出现在结论 SHA 两次构建之间的差异，或是一个内嵌的 commit SHA。报告每种噪声及其涉及的文件，以便读者核查判断。变更本身仍需要新的检查和 review，因为旧的结论从未见过它。没有构建输出的 lane（例如 dev server）无法体现这一点，所以要重新运行它，并重新验证其他任何发生漂移的内容。
4. **自底向上落地，一次一个 PR。** 始终合并序列中最低的未合并 PR，使用 `origin pr merge <pr>` 或 `gh pr merge <pr> --squash`。在 Origin 上，这个简单命令也会合并你所指定 PR 之下的所有 PR；在 GitHub 上，base 仍是其父分支的 PR 会合并到那个分支，而不是 trunk。回读 PR 以确认结果（`origin pr view <pr>` 或 `gh pr view <pr> --json state,mergedAt,baseRefName`），因为当合并队列只是把 PR 排队时，合并命令也可能报告成功，而请求的 auto-merge 并不是合并。一个已落地的 PR 显示为以 trunk 为 base 的已合并状态，而不是以父分支为 base。只在结论覆盖其当前 head 的 PR 上启用 `--auto`（Origin 的 merge-when-ready 或 GitHub 的 auto-merge），因为它会落地任何就绪的 head。合并落地后，把 base 为被合并分支的那个 PR 重新指向 trunk（`origin pr edit <pr> --base <trunk>` 或 `gh pr edit <pr> --base <trunk>`），把它的 patch 移到合并结果之上，并用 `--force-with-lease` 推送。结论覆盖的是 patch，而不是它的提交，所以第 3 步的 patch-id 规则决定结论是否仍然有效。当其检查通过且结论仍然有效时再合并它。
5. **在 stack 落地期间不要碰它。** 第 4 步中移动下一个 PR 是对分支或 base 的唯一改动，因为合并会落地当时其分支和 base 所持有的任何内容，无论结论如何。在 GitHub 上，用 `/loop` 下的 `scripts/watch-pr/watch-pr --queued-stack` 监视该序列，直到它在上限处报告 `COMPLETE`。通过 `--stack-prs` 以自底向上的顺序传入该序列的 PR 编号，每次重启 watcher 时都传入相同的列表。在 Origin 上，每次 `origin pr checks <pr> --watch` 返回时，用 `origin pr view <pr> --checks` 重新读取当前 PR，直到上限显示为已合并。当父 PR 合并时 base 移到 trunk 是预期行为，不是需要修复的损坏。如果合并停滞，先弄清原因再做任何改动。
6. **在上限处停止。** 扩展该序列需要新的结论，因此这是重新走一遍第 1 步。

**回复：** 已验证的序列及其上限、每个 PR 的结论及产出者、落地了什么以及你如何确认、下一个缺口需要什么。
