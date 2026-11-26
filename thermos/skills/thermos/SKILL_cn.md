---
name: thermos
description: "并行启动两个 thermo-nuclear 评审子代理，然后综合它们的发现。适用于 thermos、双重 thermo 评审，或组合式的缺陷/安全与代码质量分支审计。"
disable-model-invocation: true
---

# Thermos

将两轮 thermo 评审作为异步后台子代理并行运行，然后综合它们的结果。

## 工作流程

1. 根据用户请求、PR、当前分支或相关变更文件确定评审范围。
2. 收集 diff，以及评审者在不猜测的情况下评估变更所需的文件和上下文片段。
3. 在同一条消息中以 `run_in_background: true` 启动两个子代理：
   - `subagent_type: "thermo-nuclear-review-subagent"`，用于检查缺陷、破坏性变更、安全问题、开发体验回归、功能开关泄漏以及其他分支审计风险。
   - `subagent_type: "thermo-nuclear-code-quality-review-subagent"`，用于检查可维护性、结构、文件体积增长、面条式代码、抽象以及代码库健康度风险。
4. 向每个子代理传入相同的限定范围 diff/文件上下文，并要求它返回按优先级排序的发现，附带文件引用和证据。
5. 两者完成后，综合结果，将发现置于首位，并在各评审者之间去重。对相互重叠的发现赋予更高权重，用自己的判断解决分歧，并保持总结简短。

如果各个后台总结已经对用户可见，不要将其整体复述。应呈现统一结论、信号最强的发现以及任何仍存在的不确定性。
