---
name: swarm
description: "分派 N 个并行 worker，等待它们全部完成，然后返回一份报告。用于 /swarm、'swarm this'，或并行覆盖、竞速、连环考验（gauntlet）以及探索。"
disable-model-invocation: true
---

# Swarm

## 开始

在启动任何任务之前，先打开一个待办清单，每个阶段一条。

1. 框定（Frame）
2. 分派（Fan out）
3. 汇总（Aggregate）
4. 报告（Report）

## 阶段 A：框定（Frame）

1. 说明完成判定标准，以及 swarm 必须返回的产物或报告。
2. 选择形态。可以切成若干分片（slice）、让 N 个 worker 用相同任务书竞速，或两者混合。对于竞速或混合形态，在派生之前先声明 `first pass`、`rank all` 或 `best-of`。
3. 从 `~/.cursor/rules/pstack-models.mdc` 中的 `swarm workers` 一行挑选 worker 模型。如果规则或那一行缺失，使用 `claude-opus-5-5-high-fast`。对于 `auto` 或 `inherit-parent`，省略 `model`，让 worker 运行在父模型上。如果 Task 工具拒绝某个 slug，使用默认值并说明。如果它连默认值也拒绝，就从它的错误消息里选取同一系列中最接近的有效 slug。对于模型竞速，事先说出每一路（arm）的模型。按照 **poteto-mode** skill 的 Subagents 一节，为每个子代理选择适合该工作的推理级别。
4. 当 worker 需要写入时，给它自己可写的输出。当 worker 验证或测量提交（commit）时，每个任务书都要写出确切的 SHA。测量任务书还要写出方法（采样数量、一次采样是什么、顺序）。worker 在结果中记录这两项。

## 阶段 B：分派（Fan out）

在一条消息里派生全部 N 个 worker，带上 `subagent_type: generalPurpose`、`environment: "cloud"`、`run_in_background: true`，以及第 3 步选定的模型；对 `auto` 或 `inherit-parent` 则不设置 `model`。只有当 worker 需要访问用户电脑上的东西时，才使用 `environment: "local"`。

当某个 worker 必须从一个非默认的、已推送的分支开始时，传入 `cloud_base_branch`。

每份任务书都要独立成立。包含目标、范围、确切的分片或竞速路、如何验证，以及要报告什么。报告使用 `PASS`、`ISSUES` 或 `BLOCKED`，并附证据。能够证明某个缺陷的 worker 应报告 `ISSUES`，并列出它能证明的每一个问题，而不只是第一个。

如果某个 worker 退出，用 N-1 继续，并注明这一点。

## 阶段 C：汇总（Aggregate）

读取终端结果。丢弃任何没有记录其任务书所指定 SHA 和方法的结果，并重新派生那个 worker 一次。第二次仍缺失时，记录一个缺口（gap）。缺口不算通过。对于覆盖，每个必需的分片都需要一个结果。对于竞速，应用事先声明的选择规则：使用 first pass、rank all 或 best-of。不要粘贴原始 worker 输出内容。

## 阶段 D：报告（Report）

返回一份合并后的会话内报告，包含结果表、问题的一句话摘要、缺口或退出情况，以及使用竞速时的规则。
