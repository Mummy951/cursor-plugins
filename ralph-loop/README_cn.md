# Ralph Loop

Ralph Loop 让 Cursor 在一个自引用循环中运行，在每一轮结束后把同一个提示词重新喂回去，直到任务完成。它实现了由 Geoffrey Huntley 首创的 [Ralph Wiggum 技术](https://ghuntley.com/ralph/)。

## 工作原理

两个 hook 驱动这个循环。`afterAgentResponse` hook 会检查每次响应中是否出现与完成短语匹配的 `<promise>` 标签。`stop` hook 在 Cursor 结束一轮时触发。如果尚未检测到 promise，且未达到迭代上限，stop hook 就把原始提示词作为 `followup_message` 回传，启动下一轮迭代。Cursor 会在工作区和 git 历史中看到自己之前的修改，并在此基础上继续迭代、不断重复。提示词始终不变，变的是代码。

## 安装

```
/add-plugin ralph-loop
```

## 快速开始

> Start a ralph loop: "Build a REST API for todos. CRUD operations, input validation, tests. Output COMPLETE when done." --completion-promise "COMPLETE" --max-iterations 50

Cursor 会实现这个 API、运行测试、发现失败、修复失败，并不断重复，直到满足所有要求。

## Skills

**ralph-loop** 启动循环。需要提供提示词和选项：

> Start a ralph loop: "Refactor the cache layer" --max-iterations 20 --completion-promise "DONE"

- `--max-iterations <N>` 在 N 轮迭代后停止（默认：不限制）
- `--completion-promise <text>` 设置表示完成的短语

**cancel-ralph** 删除状态文件并停止循环。

**ralph-loop-help** 详细讲解该技术及用法。

## 编写好的提示词

定义明确的完成标准。像 "make it good" 这类模糊目标无法给 Cursor 提供可验证的依据。

```markdown
为 todos 构建一个 REST API。

完成时：
- 所有 CRUD 端点正常工作
- 已加入输入校验
- 测试全部通过（覆盖率 > 80%）
- 输出：<promise>COMPLETE</promise>
```

把大任务拆成多个阶段。在提示词中纳入测试/修复循环，以鼓励自我修正。始终传入 `--max-iterations` 以防止循环失控。

## 何时使用 Ralph Loop

它适用于有清晰、可验证成功标准的任务：让测试通过、完成迁移、依据规格构建功能。对于需要人类判断或目标模糊的任务，它并不合适。

## 了解更多

- [Geoffrey Huntley 的原始技术](https://ghuntley.com/ralph/)
- [Ralph Orchestrator](https://github.com/mikeyobrien/ralph-orchestrator)

## 许可证

MIT
