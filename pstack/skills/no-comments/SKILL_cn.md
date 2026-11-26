---
name: no-comments
description: "派生 Comment Sicko，修复已接受的发现，并为声称的约束提供编码方案。"
disable-model-invocation: true
---

# No comments

派生 Comment Sicko。对已接受的发现采取行动。

以 Comment Sicko 的全新视角为准。

## 范围

使用调用方的文件或 diff。否则使用当前相对于基础分支（默认 `main`）的 diff，包括工作树。

## 步骤

1. 使用 `subagent_type: "comment-sicko"` 派生 `Task`。传入范围。按照 **poteto-mode** skill 的 Subagents 章节，为每个子代理挑选适合该任务的推理级别。不要重述其规则。
2. 检查其报告和 diff。拒绝应用代码编辑、范围越界、受异常保护的删除、错误陈述的 `MUST KILL` 理由，以及把保留的有意代码视为有罪的标记。对涉及我们自己代码意外情况的标记进行重塑，使其保持可操作性。不要恢复那些注释。只有在有证据表明某个 keep 针对的是我们无法更改的内容时，该 keep 才得以存续。审计遗漏的范围内 lint 和 TypeScript 抑制（suppressions）。正确性或安全性方面的抑制仍为可操作的 `MUST KILL`。仅在有确切例外和范围内证据时才恢复删除。在接受单薄的 `IMPORTANT` 或 `do not remove` 的 kill 或 keep 之前，追踪其符号或对其运行 `/why`。如果某个 kill 含糊不清，不要恢复。如果某个 keep 被反驳或仍然含糊，删除它。还原并重新运行一份被拒绝的报告，明确指出失败之处。若第二份也被拒绝，将其报告为未决（open），并使 `/no-comments` 失败。
3. 直接修复琐碎的已接受标记：删除死路径、去掉参数或使用真正的 API。如果任何修复需要结构（shape），对被接受的集合及周围代码运行一次 `/architect`。停在草图阶段。Architect 负责设计结构。第 4 步负责实现。
4. 在范围内实现最小的根因修复。移除每一个被指名的权宜之计（workaround）。如果根因在范围之外，落地最小的范围内修复，并将其余部分报告为未决。**principle-fix-root-causes** 和 **principle-redesign-from-first-principles** skill 仅用于指导意图。二者都不授权扩大围栏（fence）或修复围栏之外的实例。绝不要附加症状防护（symptom guards）。
5. 约束性注释写着 `do not remove`、`do not change wording` 或 `talk to X before changing`。保留那些关于我们无法更改内容的 keep。提供成本最低的范围内类型、运行时、测试或 CI lint。等待交互式批准。无人值守和 eval 需要调用方预先批准。如获批准，先编码再删除。否则删除，将约束报告为未决，并草拟范围外的工作。
6. 报告删除数量、恢复的注释、重新运行次数、architect 草图、修复、编码提议、编码结果、未强制执行的约束，以及其他未决工作。
