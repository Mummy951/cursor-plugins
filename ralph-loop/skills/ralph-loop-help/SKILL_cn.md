---
name: ralph-loop-help
description: 讲解 Ralph Loop 插件、其工作原理以及可用的 skills。当用户请求 ralph loop 的帮助、想了解该技术或需要用法示例时使用。
---

# Ralph Loop 帮助

## 触发条件

用户询问 Ralph Loop 是什么、它如何工作，或需要用法指导。

## 需要讲解的内容

### 什么是 Ralph Loop？

Ralph Loop 实现了 Ralph Wiggum 技术——一种基于持续 AI 循环的迭代式开发方法论，由 Geoffrey Huntley 首创。

核心概念：同一个 prompt 被反复送入 agent。所谓“自引用”体现在 agent 能从文件和 git 历史中看到自己此前的工作，而不是把输出作为输入回灌。

每次迭代：
1. agent 收到完全相同的 prompt
2. 处理任务，修改文件
3. 尝试退出
4. Stop hook 拦截退出，并再次送入同一个 prompt
5. agent 从文件中看到自己此前的工作
6. 迭代改进直至完成

### 启动 Ralph Loop

告诉 agent 你的任务以及选项：

```
Start a ralph loop: "Build a REST API for todos" --max-iterations 20 --completion-promise "COMPLETE"
```

选项：
- `--max-iterations N` —— 自动停止前的最大迭代次数
- `--completion-promise "TEXT"` —— 用于表示完成的短语

工作原理：
1. 创建 `.cursor/ralph/scratchpad.md` 状态文件
2. agent 处理任务
3. Stop hook 拦截退出并回灌同一个 prompt
4. agent 看到自己此前的工作并进行迭代
5. 持续进行，直到检测到 promise 或达到最大迭代次数

### 取消 Ralph Loop

请 agent 取消该 ralph loop。它会删除状态文件并报告迭代次数。

### 完成承诺

为表示完成，agent 输出一个 `<promise>` 标签：

```
<promise>TASK COMPLETE</promise>
```

Stop hook 会查找这个特定的标签。若没有它（也没有 `--max-iterations`），Ralph 将无限运行。

### 何时使用 Ralph

**适合：**
- 成功标准清晰、定义明确的任务
- 需要迭代与细化的任务
- 带自我纠正的迭代式开发
- 全新项目（greenfield 项目）

**不适合：**
- 需要人工判断或设计决策的任务
- 一次性操作
- 成功标准不清晰的任务

### 了解更多

- 原始技术：https://ghuntley.com/ralph/
- Ralph Orchestrator：https://github.com/mikeyobrien/ralph-orchestrator

## 输出

清晰地向用户呈现上述信息，并针对其具体问题进行调整。
