# 错误处理

集成 bug 最常见的一个来源：把"agent 未能启动"和"agent 确实执行了工作，但该工作失败了"当作同一种错误来处理。它们并不是一回事。

## 两种失败维度

```
               未启动                         已启动，但未干净地完成
               ────────────                 ──────────────────────────────
               抛出 CursorAgentError         返回 RunResult { status: "error" | "cancelled" }
               .isRetryable                 .id  （在 dashboard 中查找）
               .code / .protoErrorCode      .durationMs   .git   .result
```

始终同时处理这两种情况。仅靠 try/catch 无法捕获一次失败的运行；仅检查 `result.status` 也无法捕获认证失败。

```typescript
import { Agent, CursorAgentError } from "@cursor/sdk";

async function runOnce(): Promise<void> {
  await using agent = Agent.create({ /* ... */ });
  try {
    const run = await agent.send(prompt);
    const result = await run.wait();

    switch (result.status) {
      case "finished":
        console.log(`ok: ${result.id}`);
        return;
      case "cancelled":
        console.warn(`cancelled: ${result.id}`);
        return;
      case "error":
        throw new Error(`run ${result.id} failed after executing; inspect run state`);
      default: {
        const _exhaustive: never = result.status;
        throw new Error(`unexpected status: ${_exhaustive}`);
      }
    }
  } catch (err) {
    if (err instanceof CursorAgentError) {
      console.error(`startup error (${err.constructor.name}): ${err.message}`);
      if (err.isRetryable) {
        // 退避并重试路径
      }
      throw err;
    }
    throw err;
  }
}
```

## `CursorAgentError` 子类型

所有由 SDK 抛出的错误都继承自 `CursorAgentError`。检查具体的子类，以决定该如何处理。


| 类                     | 典型 HTTP | 含义                                                   | 修复方法                                                        |
| --------------------- | ------------ | --------------------------------------------------- | ---------------------------------------------------------- |
| `AuthenticationError` | 401          | 密钥无效/已过期/缺失，或权限错误      | 修复 `CURSOR_API_KEY`（参见 `[auth.md](auth.md)`）            |
| `RateLimitError`      | 429          | 达到请求或用量上限                            | 退避；该错误带有 `isRetryable`                   |
| `ConfigurationError`  | 400/404      | 模型 ID 错误、请求格式错误、资源未找到 | 不要重试。修复调用。                                 |
| `NetworkError`        | 503/504      | 上游超时、临时性基础设施故障                   | 若 `isRetryable` 则带抖动重试                         |
| `UnknownAgentError`   | —            | proto 代码和 HTTP 代码都无法分类      | 记录并上报；检查 `.cause` 以获取原始的 `ConnectError` |


它们都带有：

- `message` —— 面向用户的描述（已去除 Connect 的 `[unknown]` 前缀）
- `isRetryable` —— 来自后端的权威判断，而非启发式推测
- `code` —— 相关时的底层 Connect/gRPC `Code`
- `protoErrorCode` —— 细粒度的后端错误码；稳定的枚举值
- `cause` —— 用于深度调试的原始 `ConnectError`；不要泄露给最终用户

### `UnsupportedRunOperationError`

这是一个独立的基类——它关乎 SDK 本身，而非后端。当你在一个不支持相应操作的 `Run` 上调用 `run.stream()`、`run.wait()`、`run.cancel()` 或 `run.conversation()` 时，就会抛出该错误。常见触发场景：在实时事件存储关闭之后，对通过 `Agent.getRun(...)` 获取的分离句柄调用 `cancel()`/`stream()`。（云端的 `conversation()` 是受支持的——它会尽力从流中累积数据。）

始终优先使用 `run.supports(...)`，而不是 `try/catch`：

```typescript
if (run.supports("cancel")) {
  await run.cancel();
} else {
  console.warn(`cancel not supported: ${run.unsupportedReason("cancel")}`);
}
```

## 重试模式

**应当重试**（配合退避 + 抖动）：

- 带有 `isRetryable === true` 的 `NetworkError`
- 带有 `isRetryable === true` 的 `RateLimitError`（少见；通常后端希望你等待更长的时间，而不是紧凑地重试）
- 带有 `isRetryable === true` 的 `UnknownAgentError` —— 后端在告诉你这是临时性问题

**不要重试**：

- `AuthenticationError` —— 密钥不会自行变好
- `ConfigurationError` —— 错误的输入不会自行变好
- 任何带有 `isRetryable === false` 的错误 —— 后端在告诉你这是终止性错误

对 agent 启动的重试次数保持较少（≤3）；重新启动 agent 的成本很高。如果第一次尝试因带有 `isRetryable === true` 的 `RateLimitError` 而失败，至少退避 30 秒。

```typescript
import { Agent, CursorAgentError, RateLimitError, NetworkError } from "@cursor/sdk";

async function createWithRetry(options: Parameters<typeof Agent.create>[0]) {
  const maxAttempts = 3;
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return Agent.create(options);
    } catch (err) {
      const retryable =
        err instanceof CursorAgentError &&
        err.isRetryable &&
        (err instanceof NetworkError || err instanceof RateLimitError);
      if (!retryable || attempt === maxAttempts) throw err;
      const backoffMs = 2 ** attempt * 1000 + Math.random() * 500;
      await new Promise(r => setTimeout(r, backoffMs));
    }
  }
  throw new Error("unreachable");
}
```

`Agent.create` 是惰性的——在调用 `send()` 之前它不会访问后端。大多数"启动"错误都会在那里浮现。要用重试包裹 `agent.send(...)`，而不只是 `Agent.create(...)`。

## `RunResult.status === "error"` —— 如何处理

该运行至少执行了一部分，遇到了 agent 无法恢复的情况，并报告了错误。`result` 中没有堆栈跟踪；可用的信号是：

- `result.id` —— 运行 ID。用 `Agent.getRun(result.id, { runtime: "cloud", agentId, apiKey })`（云端）或 `Agent.getRun(result.id, { runtime: "local", cwd })`（本地）来获取它，然后读取 `run.conversation()`，查看 agent 尝试做了什么。
- `result.durationMs` —— 如果为 0 或很小，说明失败发生得非常早（不太可能是运行时问题）。
- `result.git` —— 在云端，它告诉你失败之前是否创建了分支。
- `result.model` —— 确认运行的是哪个模型；在你测试多个模型时很有用。

你通常*不*会自动重试 `status: "error"`。agent 已经消耗了 token，并已选定一个方向；盲目重试很可能会做同样的事。要按"人工分流"来设计：记录 ID、给出 dashboard 链接并上报升级。

在以下情况下，重试是合理的：

- 你在做批量/扇出工作，且可以接受较小的错误率。
- prompt 是纯只读且幂等的。
- 你还检查了 conversation，并确认失败是环境性的（例如某个不稳定的 MCP server）。

## `status: "cancelled"` —— 如何处理

运行会在 `run.cancel()` 成功之后报告此状态（本地或云端）。把取消视为非致命情况：记录、清理、继续。对于在服务端被取消的云端运行（例如通过 dashboard 或另一个并行的调用方），当你最终 `wait()` 时，也会看到 `"cancelled"`。

## 生产环境调试

至少始终记录以下内容：

- `agent.agentId` —— 紧接在 create/resume 之后、任何 `send()` 之前
- `run.id` —— 紧接在 `send()` 之后、流之前
- `result.status`、`result.durationMs`、`result.git` —— 在 `wait()` 之后
- 出错时：完整的 `err.message`、`err.constructor.name`、`err.isRetryable`，以及（仅用于内部日志）`err.protoErrorCode`

这五项足以把用户报告的任何问题，与 Cursor dashboard 中的某一次具体运行关联起来。

## 禁忌

- **不要对每个错误都调用 `process.exit(1)`**。带有 `isRetryable: true` 的 `RateLimitError` 需要的是退避循环，而不是退出。
- **不要把 `err.cause` 记录给最终用户**——它是带有内部字段的原始 `ConnectError`。把 `err.message` 记录给人类看，把 `err.cause` 仅记录到内部可观测性系统中。
- **不要把 `CursorAgentError` 悄悄吞掉**，笼统地归到 `console.warn` 里。子类本身就是信号。
- **即使 `isRetryable: true`，也不要对 `AuthenticationError` 重试超过一次**——这几乎总是意味着密钥仍然无效。
