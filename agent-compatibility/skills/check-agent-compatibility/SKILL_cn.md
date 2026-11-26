---
name: check-agent-compatibility
description: 运行完整的仓库兼容性检查：扫描器评分、启动路径、验证循环和文档可靠性。
---

# 检查 agent 兼容性

## 触发条件

当用户希望对某个仓库进行完整的兼容性检查时使用。

## 工作流程

1. 启动 `compatibility-scan-review` 以运行 CLI 并捕获原始仓库评分和主要问题。
2. 启动 `startup-review` 以验证该仓库是否真的能被 agent 启动。
3. 启动 `validation-review` 以检查 agent 是否能在不进行不必要的繁重循环的情况下验证一个小的改动。
4. 启动 `docs-reliability-review` 以查看文档中记录的设置和运行路径是否可靠地符合实际情况。
5. 每项任务使用一个 subagent。不要将这些检查合并到一个 agent 提示中。
6. 计算一个内部 workflow 评分，取以下各项的四舍五入平均值：
   - `Startup Compatibility Score`
   - `Validation Loop Score`
   - `Docs Reliability Score`
7. 按以下方式计算 `Agent Compatibility Score`：
   - `round((deterministic_score * 0.7) + (workflow_score * 0.3))`
8. 将结果综合成一份最终响应。

在内部评分时，对行为检查使用具体的非整数 workflow 评分，而不是粗略的整数分档。如果启动、验证或文档大体可用，应将其视为基本良好但存在摩擦，而不是默认给到 60 多分的中间值。不要仅仅因为日志杂乱或错误文本粗糙就给出较低的 workflow 评分。

## 输出

用 markdown 回复，但要保持简洁。不要使用围栏代码块。

只显示一个分数，作为二级标题：`## Agent Compatibility Score: N/100`。不要显示其计算方式，包括权重、公式、确定性评分、workflow 评分、各项检查分数或算术过程，除非用户明确要求细分。

然后是一个扁平的、按优先级排序的列表，标签为 `Top fixes`，每行一个问题，每行以 `- ` 开头。

如果由于工具环境问题无法运行确定性扫描器，请单独说明，不要将其视为仓库缺陷，也不要因此对该仓库扣分。将确定性和行为性发现合并到同一个列表中，而不是分成不同的章节。专注于那些最能改善真实 agent 工作流程的修复项。除非用户要求更多细节，否则不要包含单独的总结。

示例格式：

## Agent Compatibility Score: 72/100

Top fixes
- 第一个问题
- 第二个问题
- 第三个问题
