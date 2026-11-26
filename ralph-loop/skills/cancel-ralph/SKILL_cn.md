---
name: cancel-ralph
description: 取消一个正在运行的 Ralph Loop。当用户想要停止、取消或中止正在运行的 ralph loop 时使用。
---

# 取消 Ralph

## 触发条件

用户想要取消或停止一个正在运行的 Ralph loop。

## 工作流程

1. 检查 `.cursor/ralph/scratchpad.md` 是否存在。

2. **如果它不存在**：告知用户“未发现正在运行的 Ralph loop。”

3. **如果它存在**：
   - 读取 `.cursor/ralph/scratchpad.md`，从 `iteration:` 字段获取当前迭代次数。
   - 移除状态文件以及任何 done 标记：
     ```bash
     rm -rf .cursor/ralph
     ```
   - 报告：“已取消 Ralph loop（当前处于第 N 次迭代）。”

## 输出

一句简短的确认信息，包含迭代次数；或者一条表示当前没有正在运行的 loop 的消息。
