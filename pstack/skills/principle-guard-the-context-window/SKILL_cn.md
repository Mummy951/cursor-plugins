---
name: principle-guard-the-context-window
description: "当上下文即将填满时应用：大量输出、长文件、重复读取、扇出式规划。把大批量任务路由给子代理；在主线程中保留摘要，而不是原始负载。"
disable-model-invocation: true
---

# 守护上下文窗口

**原因：** 上下文溢出会降低推理质量、产生压缩假象（compression artifacts），并让进展停滞。

**模式：**
- **给阶段定规模、给范围设上限。** 限制每个阶段的文件数，设定轮次预算，把机制成本计算在内。
