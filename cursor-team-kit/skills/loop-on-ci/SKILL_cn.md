---
name: loop-on-ci
description: 监控 PR 检查并修复失败项，直到全部通过。使用 gh pr checks 作为 PR 附加检查的唯一依据。
---

# CI 循环

## 触发条件

需要观察某个分支或拉取请求，并针对 CI 失败项不断迭代，直到所有必需的检查都通过。

使用 `gh pr checks` 作为依据。它包含所有 PR 附加的检查，而 `gh run list` 只涵盖 GitHub Actions。

## 工作流程

1. 解析当前分支对应的 PR。
2. 在等待之前先检查当前 PR 的各项检查。
3. 如果检查已经失败，先诊断这些失败项。
4. 如果检查仍在进行中，使用 `gh pr checks --watch --fail-fast` 进行观察。
5. 每次推送后，重新检查完整的 PR 检查集，并重复此过程直到全部通过。

## 命令

```bash
# 解析当前的 PR
gh pr view --json number,url,headRefName

# 检查所有附加的检查项
gh pr checks --json name,bucket,state,workflow,link

# 观察进行中的检查并在首次失败时中止
gh pr checks --watch --fail-fast

# 当失败的检查链接到某个 GHA 运行时，查看 GitHub Actions 日志
gh run view <run-id> --log-failed
```

## 防护栏

- 尽可能让每次修复只针对单一的失败原因。
- 不要绕过 hooks（`--no-verify`）来强行推进。
- 如果失败明显与该 PR 无关，并且在 main 分支上已修复，则合并最新的 main，而不是用无关的修复使 PR 变得臃肿。
- 如果失败是间歇性的，重试一次并报告抖动的证据。
- 每次推送后重新运行 `gh pr checks --json name,bucket,state,workflow,link`；检查集可能会发生变化。

## 输出

- 当前 CI 状态
- 失败摘要及已应用的修复
- 检查全部通过后的 PR URL
