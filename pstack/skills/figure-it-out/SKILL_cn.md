---
name: figure-it-out
description: "当没有更合适的 playbook 时，设计一份可审计的 playbook：一次大型迁移、一个野心勃勃的多部分改动，或者人类离开一段时间后会复查的工作。根据任务调整严谨度，运行假设循环，并记录决策。用于 /figure-it-out 或 'figure it out'。"
disable-model-invocation: true
---

# Figure it out

当任务不匹配任何 playbook 时，就设计一份。在写任何代码之前，交付物是工作流本身：一系列阶段，它们根据任务调整严谨度、运行科学方法，并留下一条人类离开后仍能审计的决策轨迹。

## Start

打开一个 todolist，其第一项是阅读 **poteto-mode** skill 的 Principles 部分。然后把下面的阶段作为待办项加入。

## Phase A: Frame

先摸清底数，再承诺。在启动运行之前，直到你能陈述以下内容才开始：

- 完成的定义，作为一个可证伪的谓词（**prove-it-works** principle skill）。
- 范围，量化：粗略的单元和工作量，加上摸底发现的阻塞项。
- 严谨度级别，偏高。单向门和高影响半径的获得更多。可逆的低风险步骤获得更少。严谨度是门禁和产物，而不是"再努力一点"。

在承诺一次长时间运行之前，呈现框架和权衡。可逆工作照常推进（**never-block-on-the-human** principle skill），但一次数小时的运行值得设置一个检查点。

## Phase B: Design the workflow

分解为原子的、可独立落地的单元。按风险最高的未知项优先排序。脚手架和验证先于功能（**foundational-thinking** principle skill）。

- 在工作之前构建验证工具，并从改动前的状态捕获基线，使检查读作"旧值 vs 新值"。
- 对于单向门的设计决策，运行 **architect** skill（它会运行 **arena**）。对于形态已经明确具体的机械性工作，跳过它。对一个已确定的设计再跑一次 arena 属于过度工程（**laziness-protocol** principle skill）。
- 决定哪些部分并行展开。只在接缝处并行，并给每个 worker 自己的 worktree 或分支（**separate-before-serializing-shared-state** principle skill）。不要过度展开。根据 Subagents section of the **poteto-mode** skill，为每个 subagent 选择适合其工作的推理级别。
- 把设计好的阶段列表写下来。这份列表就是人类要复查的东西。

然后执行该设计。把它的步骤作为具体项目加入 todolist 中，位于 Phase C 和 Phase D 条目之间。每一步都在 Phase C 循环纪律下运行，并把 Phase D 的日志穿插其中——每落地一步就写一行，而不是把整条轨迹留到最后。

## Phase C: Run the loop

每个单元都是一次实验。陈述假设，做最小的改动，在真实产物上对照谓词度量，若推进了就保留，若没有就还原。应用 **sequence-verifiable-units** principle skill，在启动下一个单元之前先验证当前单元，而不是把检查都堆到最后。

- 通过检查产物来验证，绝不用自报告。当某件事过于轻松地通过时，先怀疑观察方法，而不是系统。
- 把委派的工作与一个裁判配对。如果某个 worker 钻了门禁的空子，就重置并加固契约。如果门禁本身是错的，就在它自己的改动中修复门禁，而不是绕着它走。
- 结论是 VERIFIED、NOT VERIFIED 或 INCONCLUSIVE。不确定不算通过。不要隐藏否定结果。

## Phase D: Keep the audit trail

通过 **show-me-your-work** skill 记录运行过程。figure-it-out 的工作通常足够宏大，值得把轨迹提交进去，让复查者能在 PR 中读到。正是这条轨迹加上 diff，让人类能够回头并信任这份工作。

## Phase E: Verify and hand back

在真实产品上，对照 Phase A 的谓词检查整体，而不仅仅是工具。把任何反复出现的修正编码为门禁、lint 规则、检查或脚本（**encode-lessons-in-structure** principle skill）。

**Reply:** 你设计的 playbook、严谨度级别及原因、决策轨迹的路径、对照谓词已验证的内容，以及仍未解决的内容。
