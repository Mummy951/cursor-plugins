# 高级：子代理、恢复、制品与检查

你已经掌握了可用的 `Agent.create` + `send` + `wait` 循环。以下是你接下来会用到的能力。

## 子代理

在 v1 中，子代理**仅支持云端**。`AgentOptions` 上的 `agents:` 字段会通过云端创建调用中的 `customSubagents` 传递；本地执行器会静默地丢弃它。如果你在传入 `agents` 的同时又带上 `local: { ... }`，什么都不会发生。在支持本地对等能力之前，请将子代理设计限定在云端代理范围内。

定义命名子代理，主代理可以通过 `Agent` 工具来派生它们：

```typescript
const agent = Agent.create({
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  cloud: {
    repos: [{ url: "https://github.com/your-org/your-repo", startingRef: "main" }],
  },
  agents: {
    "code-reviewer": {
      description: "Expert code reviewer for quality and security.",
      prompt: "Review code for bugs, security issues, and proven approaches. Be concrete and cite file:line.",
      model: "inherit",
    },
    "test-writer": {
      description: "Writes tests for code changes.",
      prompt: "Write comprehensive unit and integration tests. Use the project's test framework.",
    },
  },
});
```

键名（`"code-reviewer"`）是主代理引用子代理的方式。`description` 告诉它何时调用；`prompt` 是子代理的系统提示词。

### 适合的使用场景

- **专项审查** —— 派生一个审查子代理，来审计主代理刚做的改动。
- **并行研究** —— 主代理同时把"总结 X""总结 Y""总结 Z"分派给三个子代理。
- **风险隔离** —— 用一个受限的提示词，把破坏性操作隔离在命名子代理中。

### 不适合的使用场景

- 替代普通的辅助函数。如果不需要 LLM 推理，就不要把它做成子代理。
- 深层嵌套的"代理调用代理再调用代理"链条。一层子代理几乎总是够用；再往下通常意味着你需要换一种架构。

### 子代理 MCP

云端子代理按**名称**引用父级的 MCP 服务器，而不是内联配置。内联的 `McpServerConfig` 条目在 v1 中会被拒绝，并抛出 `ConfigurationError("cloud custom subagents only support string references in v1.")`。请在父级的 `mcpServers` 映射中配置该服务器，然后在子代理中按键名引用它：

```typescript
const agent = Agent.create({
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  cloud: { repos: [{ url: "https://github.com/your-org/your-repo", startingRef: "main" }] },
  mcpServers: {
    postgres: {
      type: "http",
      url: "https://mcp.example.com/postgres",
      headers: { Authorization: `Bearer ${process.env.PG_RO_TOKEN!}` },
    },
  },
  agents: {
    "db-reader": {
      description: "Answers read-only questions against the database.",
      prompt: "Query with read-only SQL. Never execute writes.",
      mcpServers: ["postgres"],
    },
  },
});
```

父级的 `mcpServers` 是唯一依据；子代理从该映射中按名称挑选。

## 恢复代理

代理 ID 会持久保存。之后可以用相应的选项，从任意进程恢复：

```typescript
const agent = Agent.resume(agentId, {
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  local: { cwd: process.cwd() },
  mcpServers: sameInlineMcpYouUsedBefore, // inline MCP is not persisted!
});
const run = await agent.send("continue where we left off");
```

### 会持久保存的内容

- 对话历史和代理状态。
- 创建时在服务端固化下来的代理级设置。
- 云端：整个代理（克隆、分支，以及可能存在的 PR 状态）。

### 不会持久保存的内容

- **内联 `mcpServers`** —— 恢复时需要再次传入。
- 本地：`cwd` 通过路径标识。如果路径已不存在，恢复时就找不到本地代理。
- **`local.settingSources`** —— 环境配置会根据当前环境重新加载。

### 查找要恢复的代理

```typescript
// Local agents under a cwd:
const local = await Agent.list({ runtime: "local", cwd: process.cwd(), limit: 20 });

// Cloud agents for the caller:
const cloud = await Agent.list({ runtime: "cloud", apiKey });

// Or a specific cloud agent by ID:
const one = await Agent.get("bc-abc123", { apiKey });
```

两个 list 调用都会返回 `{ items, nextCursor }`。用 `nextCursor` 进行分页。

## 检查运行

```typescript
const runs = await Agent.listRuns(agentId, { runtime: "local", cwd: process.cwd() });
const run  = await Agent.getRun(runs.items[0].id, { runtime: "local", cwd: process.cwd() });

// Replay the stream (works when the persisted events are available):
if (run.supports("stream")) {
  for await (const event of run.stream()) { /* ... */ }
}
```

`run.conversation()` 返回累积的 `ConversationTurn[]` —— 适合用来渲染对话记录界面。正在运行的本地运行包含工具调用详情；重放的运行则根据持久化的流重建，如果最初的运行时没有捕获工具参数/结果，重放结果可能会更稀疏。

### 云端运行 ID 与代理 ID

云端代理 ID 以 `bc-` 开头。云端运行 ID 看起来像普通的 UUID。**不要把 `bc-` 开头的 ID 传给 `getRun` 并指望它能工作** —— 你需要的是运行 ID。

要从云端的 `bc-` 代理获取运行 ID：

```typescript
const runs = await Agent.listRuns("bc-abc123", { runtime: "cloud", apiKey });
for (const r of runs.items) console.log(r.id);
```

## 持久化的消息

`run.conversation()` 的作用域限定在一次运行内。要获取某个代理所有运行中完整的持久化对话：

```typescript
const messages = await Agent.messages.list(agentId, { runtime: "local", cwd: process.cwd() });
```

消息是一种原始的、结构稳定的形状 —— 用户轮次、助手轮次、元数据。你通常会在代码中把它们传给类似 `extractReadableMessages(...)` 的辅助函数来进行渲染。

## 制品

在云端代理上，代理可以生成 git diff 之外的制品文件 —— 例如测试结果、覆盖率报告、生成的资源：

```typescript
const artifacts = await agent.listArtifacts();
for (const a of artifacts) console.log(a.path, a.sizeBytes);

const buffer = await agent.downloadArtifact(artifacts[0].path);
```

本地代理目前从 `listArtifacts()` 返回空列表，并从 `downloadArtifact()` 抛出异常。目前请把制品相关流程视为仅限云端。

## 生命周期：归档、取消归档、删除

```typescript
await Agent.archive("bc-abc123",   { apiKey });
await Agent.unarchive("bc-abc123", { apiKey });
await Agent.delete("bc-abc123",    { apiKey });
```

- **归档**会让代理从默认列表中隐藏，但保留历史记录（在 `Agent.list({ runtime: "cloud" })` 上设置 `includeArchived: true` 可将其包含进来）。
- **取消归档**会反转这一操作。
- **删除**是破坏性操作 —— 无法撤销。请谨慎限定作用范围。

传入不存在的 ID 会抛出异常。不要吞掉错误；ID 不存在通常意味着你的记录维护出了问题。

## 账户与目录

```typescript
import { Cursor } from "@cursor/sdk";

const me      = await Cursor.me({ apiKey });               // apiKeyName, userEmail, createdAt
const models  = await Cursor.models.list({ apiKey });       // available model IDs
const repos   = await Cursor.repositories.list({ apiKey }); // GitHub repos the caller has connected
```

- `Cursor.models.list()` —— 如果你不知道有哪些可用模型，请在构造选项之前调用它。不要硬编码冷门的模型 ID。
- `Cursor.repositories.list()` —— 给出调用方实际可用的 `cloud.repos[].url` 条目。如果你在构建一个让用户选择仓库的界面，这就是数据来源。
- `Cursor.me()` —— 确认密钥的身份。在运维工具中很有用。

这三者都仅限云端，且需要 API key。

## `agent.reload()`

在更改本地设置（`.cursor/*`、MCP 配置文件、hook 文件）后调用此方法，如果你想让底层的本地执行器接收这些更改而不必重建代理：

```typescript
await agent.reload();
```

它不适用于正在运行的运行 —— 只影响之后的 `send()` 调用。

## 综合示例 —— 长期运行的服务

```typescript
import { Agent } from "@cursor/sdk";

export class MyAgentOrchestrator {
  async runOrContinue(agentId: string | undefined, prompt: string) {
    const apiKey = process.env.CURSOR_API_KEY!;
    const options = {
      apiKey,
      model: { id: "composer-2" },
      local: { cwd: "/var/app/repo" },
      mcpServers: {
        linear: {
          type: "http" as const,
          url: "https://mcp.linear.app/sse",
          headers: { Authorization: `Bearer ${process.env.LINEAR_API_KEY!}` },
        },
      },
    };
    await using agent = agentId
      ? Agent.resume(agentId, options)
      : Agent.create(options);

    const run = await agent.send(prompt);
    const result = await run.wait();
    return { agentId: agent.agentId, runId: run.id, status: result.status };
  }
}
```

关键选择：

- `await using` 语法意味着你不会忘记释放资源。
- 每次调用都重新传入 MCP 配置，以便恢复能正常工作。
- 将 `agentId` 持久化到调用方的数据库中；后续请求时据此恢复。
