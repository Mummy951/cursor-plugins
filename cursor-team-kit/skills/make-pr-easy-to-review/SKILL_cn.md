---
name: make-pr-easy-to-review
description: 通过清理杂乱的历史记录、改进 PR 描述以及添加审阅者指引来让 PR 更易于审阅，且不改变代码行为。适用于“让这个更容易审阅”“整理这个 PR”“清理提交记录”或“标注差异（diff）”等场景。
---

# 让 PR 更易于审阅

准备一个 PR，使审阅者能够快速理解其意图、重要文件和风险。默认目标是提升可审阅性，而不改变代码行为。

## 工作流程

1. 根据用户提供的 URL 或当前分支确定目标 PR。
2. 检查提交记录、差异（diff）大小、变更路径、生成文件和 PR 描述。
3. 识别可审阅性问题：杂乱的提交、过时的描述、无关的变更、机械性变更与逻辑性变更混杂、缺少测试，或审阅者入口不清晰。
4. 在重写历史记录或强制推送之前，先提出一个计划。
5. 应用安全的改进，然后验证代码树或差异仍然与预期的代码一致。

## 历史记录清理

仅在用户要求或同意该计划时才重写历史记录。在重写之前：

```bash
gh pr view <PR> --json title,headRefName,baseRefName,state,commits
git fetch origin <headRefName> <baseRefName>
ORIGINAL_TREE=$(git rev-parse origin/<headRefName>^{tree})
```

良好的提交分组通常遵循依赖顺序：

1. Schema/存储或生成的 API 定义。
2. 核心逻辑。
3. 接线与集成。
4. UI 或表层行为。
5. 测试。

重写之后，验证内容一致性：

```bash
echo "Original tree: $ORIGINAL_TREE"
echo "Current tree:  $(git rev-parse HEAD^{tree})"
git diff origin/<headRefName> --stat
```

如果代码树发生了非预期的变化，请勿推送。

## 审阅者指引

当代码行为应保持不变时，优先使用 PR 描述和审阅备注：

- 添加与实际差异（diff）相符的 TL;DR（摘要）。
- 将核心文件与生成的文件或机械性变更的文件区分开。
- 明确指出有风险的行为变更、迁移顺序、发布计划和测试覆盖率。
- 当问题跟踪器、仪表盘或设计文档能够解释意图时，附上相关链接。

## 防护准则

- 绝不要将有意义的行为变更隐藏在“清理”之中。
- 除非用户明确要求，否则不要绕过钩子（hooks）。
- 如果 PR 过大，无法仅通过备注就变得易于审阅，建议拆分，而不是在问题外围做表面修饰。
