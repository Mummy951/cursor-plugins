---
name: arena
description: "在同一任务上并行派出 N 个候选，选出一个作为基底，再把落选者中最强的部分嫁接进去。用于 /arena、"arena this"、"throw it in the arena"，或当对某个非平凡产物的单次尝试会锁定错误形态时。"
disable-model-invocation: true
---

# Arena

在启动任何事情之前，打开一个 todolist，每个阶段一个条目（Frame、Fan out、Cross-judge、Pick、Graft、Verify）。

## Phase A: Frame

每个候选都收到相同的提示词，因此提示词就是契约。

1. 说明每个候选正在产出的产物。
2. 推导出 rubric，即针对*本*任务而言成功是什么样子的 3-6 条具体、可评分的标准。它是 Phase D 中挑选者的工具。候选只能看到任务。
3. 从 `~/.cursor/rules/pstack-models.mdc` 中的 `arena runners` 行挑选赛手，默认是 `claude-fable-5-2-high`、`claude-opus-5-5-high-fast` 和 `gemini-4-neon-xhigh`。`auto` 或 `inherit-parent` 条目表示父模型，因此省略 `model` 并派出 `generalPurpose`。如果 Task 工具拒绝某个 slug，就使用同一系列中最接近的有效 slug，或舍弃该席位，保留至少两名赛手，并说明这一点。系列按前缀划分，最长匹配优先：去掉开头的 `cursor-` 后，依次为 `claude-fable-*`、`claude-*`、`gemini-*`。任何其他 slug 的系列是其第一个连字符之前的文本。若要覆盖多个设计方向则派出更多赛手；当工作受生成而非判断约束时，重复同一个模型。根据 **poteto-mode** skill 的 Subagents 部分为每个子代理选择推理级别。
4. 为每个候选指定各自的输出位置（尽可能使用 git worktree，否则用 `/tmp/arena-<slug>/candidate-<n>/`），遵循 **separate-before-serializing-shared-state** 原则 skill。

## Phase B: Fan out

在一条消息中派出全部 N 个子代理，设置 `run_in_background: true`，每个都带上任务、共享 grounding 的路径、各自的输出路径，以及产出产物和一份简短理由（说明其考虑过的备选方案以及所舍弃的内容）的指令。

如果某个候选没有产出，就以 N-1 继续，并记下该退出者。

## Phase C: Cross-judge

在 Phase B 的所有候选完成后，从 `~/.cursor/rules/pstack-models.mdc` 中的 `arena cross-judge pool` 行派出一个只读裁判，默认是 `claude-opus-5-5-high-fast`、`gemini-4-neon-xhigh`。裁判绝不来自父模型的供应商；就该检查而言 Fable 算作 Claude，因此使用来自另一供应商的第一个条目。`auto` 或 `inherit-parent` 条目表示父模型。如果你不知道父模型的型号，就使用第一个真实 slug，并说明供应商检查未运行。如果 Task 工具拒绝某个 slug，就使用同一系列中最接近的有效 slug，然后是下一个此类条目，或跳过裁判，并说明这一点。当你在 Phase D 中阅读时，裁判按路径标签、针对每条 rubric 标准为每个候选打分，并推荐一个基底。

## Phase D: Pick a base

在挑选之前，从头到尾读完每个候选。逐条对照 rubric 标准为每个候选打分，而不是凭整体感觉，并在 Phase C 运行了裁判时与其比较。对基底的一致意见确认该选择。分歧意味着你们中有一方存在偏见，或者 rubric 含糊不清。在决定之前读完两份理由。

挑选一个未来的维护者最容易扩展、而不破坏不变量的基底。当两者感觉不分上下时，根据 Laziness Protocol 选择更干净的边界或更小的 API。

## Phase E: Graft

再走一遍每个落选候选，找出值得移植到基底中的内容，通常每个候选一到两处，而不是它的大部分。按照 **redesign-from-first-principles** 原则 skill 手工折叠每一处嫁接，使结果在单一心智模型下保持一致。不要机械地粘贴。

如果候选们收敛到同一种形态，那是强一致。记下它并发布共识形态。无需嫁接。如果它们严重分歧，说明 Phase A 定义不足。重新定义并重跑，而不是对分歧取平均。

## Phase F: Verify

像验证任何其他输出一样验证综合后的产物，遵循 **prove-it-works** 原则 skill。如果验证发现了 arena 遗漏的问题，要么 Phase A 错了（重新定义并重跑），要么某个候选发现了它而你遗漏了嫁接（回到 Phase E）。不要掩饰它。

## Outputs

一个综合后的产物，并附带一份简短的综合说明，指明基底及原因、裁判的裁决或 Phase C 跳过了裁判、每一处嫁接及其来源候选、被舍弃的内容及原因、任何收敛或退出者，以及验证结果。
