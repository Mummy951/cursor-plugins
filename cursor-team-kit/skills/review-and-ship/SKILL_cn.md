---
name: review-and-ship
description: 审查当前分支的 bug、意图契合度和测试覆盖率；运行或编写测试；提交聚焦的改动；创建或更新 PR。
---

# 审查与发布

## 触发条件

在发布前审查改动。关闭关键问题、验证行为，并创建或更新 PR。

## 工作流程

1. 收集上下文：与基线分支对比 diff、未提交的改动、最近的提交、变更的文件，以及在有用时从近期相关对话中获取的用户意图。
2. 针对变更的行为运行有针对性的测试。如果不存在聚焦的测试，则决定是添加测试还是记录这一缺口。
3. 审查正确性、回归问题、安全性和意图契合度。对于较大的 diff，使用并行 subagent。
4. 在定稿前修复关键问题，并重新运行受影响的测试。
5. 提交选定的文件，并附上简洁的提交信息。
6. 推送分支并创建或更新 PR。

## 建议的检查

```bash
git fetch origin main
git diff origin/main...HEAD
git status
gh pr checks --json name,bucket,state,workflow,link
```

## 防护栏

- 优先考虑正确性、安全性和回归问题，而非纯风格方面的意见。
- 保持提交聚焦，避免无关的文件改动。
- 如果 pre-commit 检查失败，应修复问题而不是绕过 hooks。
- 在判断 PR 是否就绪时，使用 `gh pr checks` 而非仅限 GitHub Actions 的命令。

## 输出

- 发现摘要（critical、warning、note）
- 运行的测试及其结果
- PR URL
