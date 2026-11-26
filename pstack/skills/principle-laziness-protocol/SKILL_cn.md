---
name: principle-laziness-protocol
description: "在重构、评估 diff 大小，或想添加抽象、层级、信号透传时应用。倾向于删除和能解决问题的最小改动。"
disable-model-invocation: true
---

# 惰性协议

- **合并决策。** 不要在多个地方重复同一个选择。把它放到单一真相来源背后，并把结果作为一个简单的标志传递。
- **最小化 diff。** 做能解决问题的最小改动。更少的代码行胜过“优雅”的样板代码。
