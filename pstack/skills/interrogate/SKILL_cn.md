---
name: interrogate
description: "用于 \"interrogate\"、\"adversarial review\"、\"multi-model review\"、\"challenge this\"、\"stress test this code\"、\"find blind spots\" 或 \"tear this apart\"。对抗者试图破坏某项变更或设计，只有经运行证明的发现才会保留。"
disable-model-invocation: true
---

# Interrogate

对抗者试图破坏某个变更或设计。只有当一次运行证明了某个发现，它才会保留下来。报告这些发现。不要应用修复。

## 框架

范围即用户所指定的内容。否则，它是该分支与其基线分支的合并基点（merge base）之间、直至工作区（working tree）的分支差异（diff）。用 **in-your-own-words** skill 根据用户消息、提交记录和 PR 描述来陈述意图。当意图不明确时询问用户。

## 展开

运行 **arena** skill 的 Phase A 第 3、4 步以及 Phase B，并做四处改动。用 `interrogate reviewers` 这一行替代 `arena runners`。将范围中未提交的变更复制到每个 worktree。将每个对抗者以 `generalPurpose` 方式启动。用范围、意图和以下简报替代 arena 的任务和理由，提供给每个对抗者：

> 破坏这个变更。用针对真实代码的一次运行来证明每个发现，例如失败的测试、复现脚本，或显示错误行为的命令输出。对于设计或计划，运行一个原型或探查当前工作树。将复现保留在你自己的 worktree 中。报告每个发现时，附上标题、确切的命令、它打印出的输出，以及意图所要求的输出。省略任何你没有运行过的内容。这包括风格、命名、结构，以及需要未来变更才有意义的风险。没有发现也是一个有效的结果。

## 验证

通过重新运行其命令来按 arena Phase F 验证每个发现。当输出显示失败且该失败破坏了意图时，保留该发现。丢弃其余的。合并具有相同根因的发现。

跳过 arena 的 Phase C、D 和 E。对抗者会发现不同的失败，因此判定保留每一个经过验证的发现，而不是某一个对抗者的清单。

## 输出

- **意图**：重述。
- **发现**：对于每个经过验证的发现，给出命令、其输出、它如何破坏意图，以及发现它的对抗者。当没有发现保留下来时，说明"没有经过验证的发现"。
- **丢弃**：对于每个对抗者，给出被丢弃发现的数量及原因。
