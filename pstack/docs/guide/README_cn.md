# pstack 指南

当你不再对 agent 事无巨细地微观管理时，pstack 的效果最好。你描述你想要什么，以及你怎么知道它已完成。`/poteto-mode` 会挑选 playbook（行动手册），在步骤需要时运行其他 skill，并向你展示证据。本指南用贴近实际的提示词教你养成这种习惯。

你将学到：

1. [安装 pstack](./01-setup.md)。安装插件并选择你的模型。
2. [通过 `/poteto-mode` 路由工作](./02-poteto-mode.md)。给它一个目标，看它挑选 playbook。
3. [理解代码](./03-understand.md)。一次只读的调查，然后在你编辑任何东西之前使用 `/how`、`/why`、`/teach` 和 `/recall`。
4. [设计变更](./04-design.md)。在代码定形之前，使用 `/architect`、`/arena`、`/swarm`、`/interrogate`、原型和计划。
5. [构建并清理变更](./05-build-and-clean.md)。构建 playbook、`/tdd`、`/unslop` 和 `/no-comments`。
6. [验证并交付](./06-verify-and-ship.md)。在真实应用上证明行为，对测试很少触及的部分做形式化验证，用 `/benchmark-checklist` 审查数字，然后开一个聚焦的 PR 并推动它合并。
7. [在你睡觉时运行工作](./07-overnight.md)。循环之前的信任、夜间契约、可审计的决策日志，以及能扩展超越单个 agent 的 Projects 和 automations。
8. [用原则名称来引导](./08-principles.md)。24 个能在任务中途重定向 agent 的名称。
9. [把它变成你自己的](./09-make-it-yours.md)。你自己的 mode、用于反复出错的 `/correct`，以及如何测试 skill 变更。
10. [方法与陷阱](./10-recipes-and-pitfalls.md)。可以复制的提示词和应当避免的错误。

第一次请按顺序阅读各页。之后，每一页都可以独立阅读。

当你卡住了，或者不确定该用哪个 skill 时，输入 [`/poteto-help`](../../skills/poteto-help/SKILL.md) 并附上你的问题：

```text
/poteto-help which skill should i use to review this branch?
```

它会给出回答，递给你一条可发送的提示词，并链接答案来源的 skill 或指南页面。它不会开始工作，因为一次 pstack 运行会消耗真实的 token，所以你在准备好时再发送那条提示词。它只在你输入时才会运行。

## 如果你只记住一件事

用你自己的话，给 agent 一个目标和一个检查它的方式：

```text
/poteto-mode the export writes duplicate rows when a retry lands mid-run. repro first, then fix and verify.
```

你不需要说出 playbook 的名字或列出 skills。"repro first"（先复现）和一个可检查的结果，就是 `/poteto-mode` 所需的全部路由信号。它会匹配 Bug fix playbook，把步骤复制到待办列表，并在每个步骤触发时调用相应的 skills。

下一步：[安装 pstack](./01-setup.md)。
