# MCP 服务器

MCP（Model Context Protocol，模型上下文协议）服务器为 agent 提供额外的工具。SDK 在本地和云端运行时都支持它们，但传输语义和凭证处理规则有所不同，弄错这些通常不会报错，而是悄无声息地出问题（"为什么 agent 没有我的 Linear 工具？"）。

## 两种传输方式，三种部署形态

```typescript
type McpServerConfig =
  | { type?: "stdio"; command: string; args?: string[]; env?: Record<string,string>; cwd?: string }
  | { type?: "http" | "sse"; url: string; headers?: Record<string,string>; auth?: { CLIENT_ID: string; CLIENT_SECRET?: string; scopes?: string[] } };
```

你实际会用到的三种模式：

1. **本地 stdio** —— 在调用方的机器上生成一个子进程。这是大多数 `@modelcontextprotocol/server-*` 软件包的默认方式。
2. **本地 HTTP / 远程 HTTP** —— 由 SDK（本地）或后端（云端）代表 agent 调用的一个 URL。
3. **云端 stdio** —— 子进程生成在**云端 VM 内部**，而不是调用方的机器上。

在两种运行时中，配置形态看起来相似；变化的是执行位置。一个值得了解的具体差异：云端 stdio 服务器拒绝 `cwd` —— 由 VM 控制工作目录，如果你尝试设置它，`cloud-mcp-utils.ts` 会抛出 `ConfigurationError("Cloud MCP server cannot include cwd.")`。对于云端 stdio，省略 `cwd`。

## 本地运行时 —— MCP 配置

```typescript
const agent = Agent.create({
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  local: { cwd: process.cwd() },
  mcpServers: {
    filesystem: {
      type: "stdio",
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-filesystem", process.cwd()],
      cwd: process.cwd(),
      env: { NODE_OPTIONS: "--max-old-space-size=4096" },
    },
    docs: {
      type: "http",
      url: "https://example.com/mcp",
      headers: { Authorization: `Bearer ${process.env.DOCS_TOKEN!}` },
    },
  },
});
```

- Stdio 服务器作为 Node 进程的子进程运行。确保 `command` 在 PATH 上，或使用绝对路径。干净地释放（dispose）agent 以回收这些进程。
- HTTP 服务器由本地 SDK 直接调用。`headers` 会出现在每一次请求中。
- `auth: { CLIENT_ID, CLIENT_SECRET?, scopes? }` 会触发 OAuth 流程 —— 适用于能发放正规 OAuth 令牌的第一方集成。大多数自行管理的 MCP 服务器只需要 `headers`。

## 云端运行时 —— MCP 配置

```typescript
const agent = Agent.create({
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  cloud: {
    repos: [{ url: "https://github.com/your-org/your-repo", startingRef: "main" }],
  },
  mcpServers: {
    linear: {
      type: "http",
      url: "https://mcp.linear.app/sse",
      headers: { Authorization: `Bearer ${process.env.LINEAR_API_KEY!}` },
    },
    github: {
      type: "stdio",
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-github"],
      env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN! },
    },
  },
});
```

与本地相比的关键差异：

- **HTTP 的 `headers` 和 `auth` 由 Cursor 后端代理。** 敏感的 header 值会在服务端被脱敏，不会到达云端 VM。这是存放 OAuth 令牌、API 密钥等的安全位置。
- **Stdio 的 `env` 值会被注入云端 VM。** 像对待任何生产环境密钥一样对待它们 —— VM 内运行的进程可以看到它们。不要用这种方式传递最终用户的凭证。
- **`command`/`args` 必须能在云端 VM 内解析**，该 VM 使用标准 Linux 镜像。`npx`、`node` 以及常见二进制文件都可用；其他工具通常需要一个能在 VM 内解析的 `command`，或由容器提供的二进制文件。

通过 Dashboard 配置的 MCP 服务器在云端同样会被识别。用户在 [https://cursor.com/agents](https://cursor.com/agents) 一次性配置它们；`Agent.create` 上的内联配置会叠加在其之上。

## 跨 `Agent.resume(...)` 的持久性

**内联的 `mcpServers` 不会被持久化。** 如果你恢复（resume）一个 agent 并期望拥有相同的 MCP 工具访问权限，需要再次传入配置：

```typescript
const baseMcp = { /* ... */ };
const agent = Agent.resume(agentId, {
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  local: { cwd: process.cwd() },
  mcpServers: baseMcp,
});
```

通过 Dashboard 配置的服务器确实会持久化（它们以用户为键，而不是以 agent 为键），所以常见的错误是贪图内联配置的便利，然后又疑惑为什么 resume 后丢失了它。

## 来自设置（Settings）的 MCP 服务器

默认情况下，本地 SDK 的 agent **不会**加载环境中存在的用户/团队/项目 MCP 配置。可通过 `settingSources` 选择启用：

```typescript
const agent = Agent.create({
  apiKey,
  model: { id: "composer-2" },
  local: {
    cwd: process.cwd(),
    settingSources: ["project", "user"], // or "all"
  },
});
```

`settingSources` 位于 **`local` 内部** —— 把它放在顶层会导致 TypeScript 报错，并在 JavaScript 中静默地不生效。它对云端 agent 没有影响；云端始终遵循 `project` / `team` / `plugins`。

有效的来源有：`"project"`、`"user"`、`"team"`、`"mdm"`、`"plugins"`，或用 `"all"` 包含全部。使用能满足你需求的最窄集合；`"all"` 可能会引入团队策略和插件，让生产代码感到意外。

内联的 `mcpServers` 始终优先：它们是显式输入，而非环境配置。你可以将两者结合使用。

## 在 HTTP 与 stdio 之间选择

对于远程服务（Linear、GitHub、Jira、Figma，或你自己的内部工具）：

- **优先选择 HTTP**，当服务拥有稳定的 MCP 端点时。在云端，Cursor 的后端负责处理 auth 代理；你不必把密钥带进 VM。
- **使用 stdio**，当你的集成是本地优先的辅助工具（文件系统、某个 CLI、某个开发工具），或当你需要进程级隔离时。在云端，stdio 服务器运行在 Cursor VM 内部 —— 对无状态的辅助工具没问题，但对任何持有长期凭证的东西都很危险。

对于支持 OAuth 的第一方 MCP 服务器，使用 `auth: { CLIENT_ID, CLIENT_SECRET?, scopes? }`，而不是硬编码 `Authorization` header。

## 文件变更后重新加载 MCP

如果你在 agent 运行时修改了本地磁盘上的 `.cursor/mcp.json` 或其他设置来源，并希望 agent 在不重新创建的情况下识别它：

```typescript
await agent.reload();
```

这只影响之后的 `send()` 调用；正在进行的运行使用它们启动时的配置。

## 调试 MCP 故障

MCP 配置问题往往表现为"agent 没有这个工具"，而不是抛出异常。排查清单：

1. **服务器真的注册了吗？** 观察流开始时的 `system` 事件 —— 它有时会包含工具目录。如果没有，说明服务器没有接上。
2. **云端上的 stdio** —— `command` 在云端 VM 镜像中可用吗？`npx` 可用。你自定义的 `/usr/local/bin/…` 则不行。
3. **HTTP headers 缺失？** 在云端，敏感 header 会被脱敏但仍会被转发；缺失的 header 意味着 SDK 配置没有包含它们。仔细核对拼写（某些服务器要求是 `Authorization`，而非 `authorization`）。
4. **Resume 时没有重新传入 MCP？** 你丢失了内联服务器。在 resume 的选项中重新传入它们。
5. **`settingSources` 没设置？** agent 没有加载你项目的 `.cursor/mcp.json`。加上 `"project"`。

## 可直接复制的入门配置

### 本地文件系统辅助工具

```typescript
mcpServers: {
  filesystem: {
    type: "stdio",
    command: "npx",
    args: ["-y", "@modelcontextprotocol/server-filesystem", process.cwd()],
  },
},
```

### 云端 + Linear（HTTP，由云端代理 auth）

```typescript
mcpServers: {
  linear: {
    type: "http",
    url: "https://mcp.linear.app/sse",
    headers: { Authorization: `Bearer ${process.env.LINEAR_API_KEY!}` },
  },
},
```

### 云端 + GitHub（stdio，密钥注入 VM）

```typescript
mcpServers: {
  github: {
    type: "stdio",
    command: "npx",
    args: ["-y", "@modelcontextprotocol/server-github"],
    env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN! },
  },
},
```

### 通过 Cursor Dashboard 配置的服务器（无需内联配置）

只要云端 agent 被允许使用团队 MCP，通过 Dashboard 配置服务器的用户就能免费获得它。无需内联配置；只有当你还想要一个额外的服务器时，才在其之上叠加。
