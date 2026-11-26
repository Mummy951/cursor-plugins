### 开启 PR（Opening a PR）

在每个其他 playbook 结束时调用。

从基于 trunk 的 git worktree 中工作，并把无关工作排除在外。Rebase 成小型、可落地的提交。在提交前，对 diff 运行来自 `cursor-team-kit` 的 `/deslop`。

标题：`type(scope): subject`，当某个真实符号承载了该改动时就以它命名。正文：一份一分钟内可读完、少于 40 行的简报，按以下顺序使用这些 `##` 标题，省略为空的标题：`## Why`、`## What changed`、`## Scope`、`## Tradeoffs`（只写那些否则审查者会追问的被否决备选方案）、`## Blast Radius`、`## Verification`。

按基础分支堆叠后续 PR。通过该次运行的 PR 工具以 `draft: false` 将 PR 开启为就绪状态。如果没有该工具，就使用不带 `--draft` 的 `gh pr create`，或者当 Origin 的 CLI 能解析该仓库时使用 `origin pr create --status open`。开启 PR 不会启动 babysit。
