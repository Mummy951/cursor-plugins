---
name: ralph-loop
description: 启动 Ralph Loop 进行迭代式自引用开发。当用户要求运行 ralph loop、启动迭代循环，或希望对一个任务反复自主迭代直至完成时使用。
---

# Ralph Loop

## 触发条件

用户想要启动一个 Ralph loop。这是一种迭代式开发循环：每一轮结束后都会重新喂入相同的提示，agent 每次迭代都能看到自己上一轮的工作成果。

## 工作流程

1. 收集用户的任务提示和可选参数：
   - `max_iterations`（数字，默认为 0 表示不限制）
   - `completion_promise`（文本，若未设置则为 "null"）

2. 若目录 `.cursor/ralph/` 不存在则创建它，然后在 `.cursor/ralph/scratchpad.md` 写入状态文件，格式必须完全如下：

   ```markdown
   ---
   iteration: 1
   max_iterations: <N or 0>
   completion_promise: "<TEXT>" or null
   ---

   <the user's task prompt goes here>
   ```

   示例：
   ```markdown
   ---
   iteration: 1
   max_iterations: 20
   completion_promise: "COMPLETE"
   ---

   Build a REST API for todos with CRUD operations, input validation, and tests.
   ```

3. 向用户确认 Ralph loop 已激活，然后开始处理任务。

4. stop hook 会自动拦截每一轮的结束，并将相同的提示作为后续消息重新喂入。你会看到它的前缀为 `[Ralph loop iteration N.]`。

## 护栏

- 若设置了完成承诺（completion promise），只有当该表述完全且确实为真时，你才可以输出 `<promise>TEXT</promise>`。
- 不要输出虚假的承诺来逃离循环。
- 始终建议设置 `max_iterations` 作为安全兜底。
- 若 `completion_promise` 的值包含特殊字符，请在 YAML frontmatter 中为其加上引号。

## 输出

确认循环已激活（提示、迭代上限、若已设置的承诺），然后立即开始处理任务。
