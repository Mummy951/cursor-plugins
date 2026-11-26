---
name: continual-learning
description: 通过将对话记录挖掘和 `AGENTS.md` 更新委托给 `agents-memory-updater` 来编排持续学习。
disable-model-invocation: true
---

# 持续学习

通过将记忆更新流程委托给一个 subagent，使 `AGENTS.md` 保持最新。

## 触发条件

当用户要求挖掘此前的对话记录、维护 `AGENTS.md`，或运行持续学习循环时使用。

## 工作流程

1. 调用 `agents-memory-updater`。
2. 返回该 updater 的结果。

## 防护栏

- 父级 skill 仅负责编排。
- 不要在父级流程中挖掘对话记录或编辑文件。
- 不要绕过该 subagent。
