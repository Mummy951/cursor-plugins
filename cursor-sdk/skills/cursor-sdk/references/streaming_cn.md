# 流式传输与运行生命周期

`run.stream()` 是一个 `SDKMessage` 事件的异步生成器。本地和云端运行时的事件形态相同——只需编写一个消费者，即可在所有环境中工作。

## 何时使用流式传输，何时只用 `wait()`

| 场景                                             | 流式传输？      | `wait()`？ |
| ------------------------------------------------ | --------------- | --------- |
| 向用户渲染实时输出（CLI、聊天、Web UI）          | 是              | 是        |
| 只关心成功/失败的一次性脚本                       | 否              | 是        |
| 希望记录工具调用以便调试的 CI 步骤                | 是，输出到 stderr | 是       |
| 可观测性：记录每个事件以便日后回放                | 是，将其持久化   | 是        |
| 轮询另一个并非由你启动的运行                      | 流式传输也可以   | 是        |

你几乎总是需要 `wait()`。有时你不需要 `stream()`。不存在“只流式传输而不 wait”的正确模式——流式传输告诉你发生了什么，`wait()` 告诉你它是否成功。

## 标准消费者

```typescript
for await (const event of run.stream()) {
  switch (event.type) {
    case "assistant":
      for (const block of event.message.content) {
        if (block.type === "text") process.stdout.write(block.text);
        // block.type === "tool_use" means the assistant announced a tool call;
        // the actual execution will follow via tool_call events.
      }
      break;
    case "thinking":
      // Reasoning content. Usually hidden from end users, kept for logs.
      process.stderr.write(`[thinking] ${event.text}\n`);
      break;
    case "tool_call":
      console.error(`[tool] ${event.name} ${event.status} (${event.call_id})`);
      if (event.args !== undefined) console.error(`  args: ${JSON.stringify(event.args)}`);
      if (event.result !== undefined) console.error(`  result: ${JSON.stringify(event.result)}`);
      break;
    case "status":
      console.error(`[status] ${event.status}`);
      break;
    case "task":
      if (event.text) console.error(`[task] ${event.text}`);
      break;
    case "user":
      // Echo of the prompt. Usually ignorable.
      break;
    case "system":
      // Init metadata (model, tool list). Useful for logs.
      break;
    case "request":
      // Request tracking. Log event.request_id for correlation.
      break;
  }
}

const result = await run.wait();
```

## 事件参考

每个事件都带有 `agent_id` 和 `run_id`。`type` 用于区分其他一切内容。

### `"assistant"`

模型文本或工具调用声明。

```typescript
{
  type: "assistant",
  message: { role: "assistant", content: Array<TextBlock | ToolUseBlock> }
}
```

- `TextBlock` = `{ type: "text", text: string }` —— 渲染此内容。
- `ToolUseBlock` = `{ type: "tool_use", id, name, input }` —— 助手正在请求调用工具。你无需对此采取行动；运行时会执行它并发出 `tool_call` 事件。对于那些希望在 LLM 提出请求的瞬间就显示“正在调用 `grep`……”的 UI 来说，这很有用。

### `"thinking"`

推理内容。不要将其放在主 UI 中（用户不需要它），但要保留在日志中（调试时它非常宝贵）。

```typescript
{ type: "thinking", text: string, thinking_duration_ms?: number }
```

### `"tool_call"`

实际的工具执行生命周期。

```typescript
{
  type: "tool_call",
  call_id: string,
  name: string,
  status: "running" | "completed" | "error",
  args?: unknown,
  result?: unknown,
  truncated?: { args?: boolean; result?: boolean }
}
```

首次发出时 `status: "running"`（args 可用，result 未定义），随后再次发出 `status: "completed"` 或 `"error"`（result 可用）。`truncated` 标志表示载荷已在服务端被裁剪——不要尝试完整解析它。

### `"status"`

运行生命周期转换。与 `SDKStatusMessage` 匹配：

| `status` 值     | 含义                                   |
| -------------- | --------------------------------------- |
| `"CREATING"`   | 云端运行正在搭建（克隆、启动）          |
| `"RUNNING"`    | 正在积极执行                            |
| `"FINISHED"`   | 已成功完成                              |
| `"ERROR"`      | 运行中途失败                            |
| `"CANCELLED"`  | 运行已被取消                            |
| `"EXPIRED"`    | 运行已过期                              |

**不要将 `FINISHED` 状态事件当作“我可以跳过 `wait()`”** —— 它只是一个预告，而非最终结果。`wait()` 返回一个 `RunResult`，其中包含你无法从流式传输中获取的使用量/时长/git 信息。

### `"task"`

更高层级的任务状态消息（例如“Planning”、“Editing files”）。可选；对于汇总式进度 UI 很有用。

### `"user"`

运行开始时用户提示的回显。通常会被消费者忽略。

### `"system"`

初始化元数据：实际使用的模型、工具目录。适合在流式传输开始时记录一次。

### `"request"`

请求 ID 跟踪，用于与服务端的可观测性关联。如果你有内部追踪系统，请记录 `event.request_id`。

## 回调与流式传输的对比

`agent.send(...)` 也接受回调：

```typescript
await agent.send(prompt, {
  onDelta: ({ update }) => { /* raw executor delta */ },
  onStep:  ({ step })   => { /* batched step after text/thinking/tool settle */ },
});
```

- `onDelta` 在每个原始执行器增量上触发——粒度比 `SDKMessage` 流细得多。适用于希望获得子块更新的本地 UI。在集成中很少使用。
- `onStep` 在一个逻辑步骤完成时触发（文本 + 思考 + 工具打包在一起）。类似于从流中遍历 `assistant` + `tool_call`，但已预先组装好。

两个回调都会在下一个更新被流水线处理之前等待完成——你可以通过返回一个 Promise 来施加背压。不要在 `onDelta` 中不加考虑地放入缓慢的 I/O；它可能会拖慢运行。

对于大多数消费者，优先使用 `run.stream()`。只有在构建本地 UI 且需要更精细的形态时，才使用 `onDelta` / `onStep`。

## 取消

```typescript
if (run.supports("cancel")) {
  setTimeout(() => run.cancel(), 30_000);
}
```

`run.cancel()` 在本地和云端运行上都受支持。对于云端，它会向服务端的取消端点发送 POST 请求，并根据服务端的权威响应协调本地状态。无论如何都要用 `run.supports("cancel")` 做保护——分离/回放的运行句柄（`Agent.getRun(...)`）可能没有实时的取消通道，而做保护是正确的防御姿态。

取消后，继续消费流直到其结束；你会看到一个终态 `status` 事件，并且 `run.wait()` 会以 `status: "cancelled"` 解析。

## 状态监听器（无流式传输的观察方式）

如果你只想知道运行何时发生状态转换，而不关心内容：

```typescript
const unsubscribe = run.onDidChangeStatus(status => {
  console.error(`[status-listener] ${status}`);
});

await run.wait();
unsubscribe();
```

这不需要 `stream()`，并且会在每次状态转换时触发。适用于不渲染 agent 输出的进度 UI。

## 观察并非由你启动的运行

```typescript
const existing = await Agent.getRun(runId, { runtime: "cloud", agentId: "bc-abc123", apiKey });
if (existing.supports("stream")) {
  for await (const event of existing.stream()) {
    // same loop
  }
}
const result = await existing.wait();
```

回放的流会从持久化的状态重建事件。如果原始运行时捕获了工具载荷（args/result），它们就会存在；较旧的运行的载荷可能会更稀疏。

## 背压与长流

异步迭代器天然地施加背压——运行时生成事件的速度不会超过你的 `for await` 消耗它们的速度。但如果你的消费者对每个事件执行繁重的处理（数据库写入、网络调用），可能会在整个运行期间拖慢它。当这种情况很重要时，请将任务排队并在带外处理。

## 常见错误

- **不耗尽流** —— 会使资源保持打开状态。如果你打开了 `run.stream()`，就必须完整地消费它，或调用 `run.cancel()`。
- **假设工具的 `args`/`result` 总是存在** —— 它们是可选的；在解构之前先检查。
- **在未检查 `name` 的情况下将 `tool_call.result` 解析为特定形态** —— 每个工具都有自己的形态。如果需要强类型，请先按 `event.name` 分支。
- **将 `"status": "FINISHED"` 当作结束来响应** —— 它是终态，但 `wait()` 仍需解析以向你提供使用量/git/时长信息。务必也执行 `await run.wait()`。
