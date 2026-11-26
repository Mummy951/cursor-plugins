---
name: cursor-sdk
description: 指导用户在 Cursor TypeScript SDK（`@cursor/sdk`）之上构建应用、脚本、CI 流水线或自动化。当用户提到集成、安装或编写针对 Cursor SDK 的代码时使用此 skill；当他们说 `Agent.create`、`Agent.prompt`、`Agent.resume`、`agent.send`、`run.stream`、`CursorAgentError` 或 `@cursor/sdk` 时使用；当他们要求从脚本、CI/CD 流水线、GitHub Action、后端服务或任何非 Cursor IDE 本身的代码中以编程方式运行 Cursor agent 时使用；以及当他们想要在本地和云端运行时之间选择、为 SDK agent 配置 MCP 服务器，或处理来自 SDK agent 的流式输出、取消或错误时使用。当用户正在把 Cursor 接入自动化、编写运行 Cursor 的机器人，或把 REST `/v1/agents` 调用迁移到 SDK 时也要触发，即使他们没有明确说出包名。要主动使用此 skill，而不是凭记忆回答；SDK 的接口在不断变化，本 skill 及其 references 才是该外部包的权威来源。
---

# Cursor SDK

Cursor TypeScript SDK（`@cursor/sdk`）以编程方式运行 Cursor agent。同一套接口既驱动本地运行时（agent 在你的机器上针对你的文件运行），也驱动云端运行时（agent 在 Cursor 托管或自托管的基线设施上针对克隆的仓库运行并创建 PR）。

使用此 skill 帮助他人**快速搭建一个可用的集成**，并**避开少数会坑到新用户的陷阱**。权威文档位于 [https://cursor.com/docs/api/sdk/typescript](https://cursor.com/docs/api/sdk/typescript)；本 skill 只补充决策制定、失败模式预防，以及随时可扩展的模式。

## 语气与姿态

本 skill 帮助用户**构建**集成。它不是用来验证、祝贺或推销 SDK 作为一个选项的地方。用户的意图就是输入；你的职责是执行。

- **当用户明确点名 SDK 时**（说 "Cursor SDK"、`@cursor/sdk`、`Agent.create`、`Agent.prompt` 等）：假定他们知道 SDK 是什么并已决定使用它。跳过铺垫，跳过鼓劲的话，直接产出集成。不要说"好消息"，不要说"SDK 非常适合这个"，也不要说"这几乎正是 X 被设计出来的模式"。
- **当用户描述了一个 SDK 适合但没点名它的问题时**（"我想要一个审查我 PR 的机器人"、"我想要一个就我的仓库向 Cursor 提问的脚本"）：SDK 还不是确定的选项。简要地把它作为一个问题提出来，然后等待：*"这里我会用 Cursor SDK——你想让我按这个方向来设计吗，还是你心里有别的运行时？"* 如果他们确认，就继续。如果他们反对或想要多个选项，就给出选项。
- **无论哪种情况，都不要把用户的意图复述给他们。** 他们知道自己想要什么。直接进入设计。

避免这些特定的开场白（及其近似变体）：

- "好消息：这正是……的模式"
- "SDK 就是为这种形态而构建的。"
- "太好了，你来对地方了。"
- "这几乎正是 SDK 为 X 设计的模式。"
- 任何恭维用户选择或以溢美之词复述其目标的开场。

优先采用：

- 以设计决策或他们首先需要知道的事情开场。
- 如果你确实有一个设计选择要指出（本地 vs 云端、prompt vs send、同步 vs 流式），用一句话点明并解释原因；不要用验证性的话做铺垫。

## 何时打开 reference 文件

保持本页简短。仅当用户的任务明确落在某个 reference 范围内时才阅读它：


| 如果用户正在……                                                              | 阅读                                                             |
| ---------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| 在本地和云端运行时之间选择，或不确定该用哪个                                 | [`references/runtime-choice.md`](references/runtime-choice.md)   |
| 调试认证（401、"Missing CURSOR_API_KEY"、团队 key vs 用户 key、本地 vs 生产） | [`references/auth.md`](references/auth.md)                       |
| 处理错误、重试、速率限制、`CursorAgentError`、`result.status === error`      | [`references/error-handling.md`](references/error-handling.md)   |
| 消费流式输出、选择事件类型、取消，或决定用流式还是等待                       | [`references/streaming.md`](references/streaming.md)             |
| 配置 MCP 服务器（HTTP、stdio、云端 vs 本地传输、注入认证）                   | [`references/mcp.md`](references/mcp.md)                         |
| 使用子 agent、resume、artifacts、列出/检查 agent、`Agent.messages`          | [`references/advanced.md`](references/advanced.md)               |
| 构建特定集成（CI 审查机器人、定时分诊、聊天、webhook）                       | [`references/patterns.md`](references/patterns.md)               |


下面所有内容是应对 80% 任务所需的最低限度知识。

## 三种调用模式

几乎每一种 SDK 集成都归结为三种形态之一。选择适合工作的那种，不要混用。

### 1. `Agent.prompt(...)` — 一次性

```typescript
import { Agent } from "@cursor/sdk";

const result = await Agent.prompt("Refactor src/utils.ts for readability", {
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  local: { cwd: process.cwd() },
});
console.log(result.status, result.result);
```

用于即发即忘的脚本、GitHub Actions 步骤，或任何"发出这个 prompt、拿到结果、退出"的流程。没有流式输出，没有后续交互，没有需要记住的清理操作。如果你打算用这个然后立刻 resume，那你其实想要的是模式 2。

### 2. `Agent.create(...)` + `agent.send(...)` — 持久且可后续交互

```typescript
import { Agent } from "@cursor/sdk";

const agent = Agent.create({
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  local: { cwd: process.cwd() },
});

try {
  const run = await agent.send("Find the bug in src/auth.ts");
  for await (const event of run.stream()) {
    if (event.type === "assistant") {
      for (const block of event.message.content) {
        if (block.type === "text") process.stdout.write(block.text);
      }
    }
  }
  const result = await run.wait();

  // Follow-up keeps full conversation context.
  const run2 = await agent.send("Now write a regression test for it");
  await run2.wait();
} finally {
  await agent[Symbol.asyncDispose]();
}
```

当你需要流式输出、多轮对话或生命周期操作（取消、状态监听器）时使用。这是大多数非平凡集成的形态。

### 3. `Agent.resume(...)` — 稍后接续已有 agent

```typescript
const agent = Agent.resume(previousAgentId, {
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  local: { cwd: process.cwd() },
});
const run = await agent.send("Also update the changelog");
await run.wait();
```

跨进程边界使用：一个接续昨晚清理任务的 cron、一个扩展用户 agent 的 webhook、一个重新加载对话状态的交互式 CLI。**内联的 `mcpServers` 不会在 resume 时保留** —— 在 resume 调用时再次传入它们。

## 五大陷阱（写代码前先读这些）

这些几乎会让每一个新集成栽跟头。一旦知道，它们都很容易预防。

### 1. 缺少 `cloud: { repos }` 会静默地默认走本地

`AgentOptions` 并不要求 `local` 或 `cloud`；如果两者都省略，SDK 会选择本地运行时。陷阱在于：如果你本想要云端 agent 却忘了 `cloud:` 字段，你会静默地得到一个本地 agent——没有报错，只是一个本地 agent ID 和一个本地执行器。想要云端时始终显式传入 `cloud: { repos }`，本地时也显式传入 `local: { cwd }`，尽管它是默认值。选择正确的运行时：参见 [`references/runtime-choice.md`](references/runtime-choice.md)。

### 2. 两种不同类型的失败，却有把它们混为一谈的本能

```typescript
try {
  const run = await agent.send(prompt);
  const result = await run.wait();
  if (result.status === "error") {
    // Agent started but failed mid-run. Inspect transcript, git state, tool outputs.
    console.error(`run failed: ${result.id}`);
    process.exit(2);
  }
} catch (err) {
  if (err instanceof CursorAgentError) {
    // Didn't start. Auth, config, network. Fix environment, retry.
    console.error(`startup failed: ${err.message}, retryable=${err.isRetryable}`);
    process.exit(1);
  }
  throw err;
}
```

抛出 `CursorAgentError` → 这次运行从未执行（认证、配置、网络问题）。`result.status === "error"` → agent 确实干了活，而那些活失败了。不同的修复方式、不同的退出码、不同的可观测性。完整分类见 [`references/error-handling.md`](references/error-handling.md)。

### 3. 忘记 `await agent[Symbol.asyncDispose]()` 会造成资源泄漏

SDK 持有本地执行器、持久化运行存储和云端 API 客户端的句柄。不释放意味着泄漏子进程、打开的数据库，以及在长时间运行的服务中内存增长。始终在 `finally` 中释放，或使用 `Agent.prompt()`（它会替你释放），或者在你的 tsconfig 支持时使用 `await using` 语法：

```typescript
await using agent = Agent.create({ /* ... */ });
```

### 4. 流式输出是可选的，但 `wait()`（几乎）是必需的

`run.stream()` 是你观察运行的方式；`run.wait()` 是你获取终态结果的方式。你可以跳过流式输出，但跳过 `wait()` 意味着你无法判断这次运行是完成了、出错了还是被取消了，而且你会泄漏这次运行的内部 watcher。始终调用 `wait()`。如果你不想要实时输出，只调用 `wait()` 即可。事件类型参考见 [`references/streaming.md`](references/streaming.md)。

### 5. 并非每个 `run` 操作都在每种运行时上受支持

`Run` 暴露四个操作——`stream`、`wait`、`cancel`、`conversation`——运行时可能支持也可能不支持每一个。在调用前始终用 `run.supports("...")` 做守卫，而不是想当然：

```typescript
if (run.supports("cancel")) await run.cancel();
if (run.supports("conversation")) console.log(await run.conversation());
```

当前值得注意的缺口：分离/重新水化的运行（你是在实时事件存储关闭后通过 `Agent.getRun(...)` 拿到句柄的）可能不支持 `stream()`，且 `conversation()` 可能为空。`run.unsupportedReason(op)` 会告诉你原因。云端 `run.conversation()` 是受支持的——它尽力从流中累积。

## 本地 vs 云端，各用一句话

- **本地** —— 在调用者机器上针对 `cwd` 运行，复用其环境和凭据，适合开发循环和已经有仓库检出的 CI。
- **云端** —— 在 Cursor 托管的 VM 上针对新克隆的 `repos[].url` 运行，适合长时间任务、即发即忘的自动化，以及创建真正的 PR（`autoCreatePR: true`）。

决策树、能力差异和能力缺口（artifacts、cancel、MCP 传输）：[`references/runtime-choice.md`](references/runtime-choice.md)。

## 认证，最低可行方案

```bash
export CURSOR_API_KEY="cursor_..."  # user API key or team service-account key
```

如果没有传入 `apiKey`，SDK 会读取 `CURSOR_API_KEY`。用户 key（来自 [https://cursor.com/dashboard/cloud-agents](https://cursor.com/dashboard/cloud-agents)）和团队服务账号 key（Team Settings → Service accounts）在本地和云端运行中都可用。

如果你看到 401，常见嫌疑是：key 粘贴时带了周围空白、key 是针对不同环境签发的，或者该 key 所属用户对云端运行所需的仓库没有访问权限。完整排障见 [`references/auth.md`](references/auth.md)。

## 模型选择

```typescript
import { Cursor } from "@cursor/sdk";

const models = await Cursor.models.list({ apiKey: process.env.CURSOR_API_KEY! });
```

`composer-2` 是当前大多数集成的默认选择。`{ id: "auto" }` 让服务端来选。模型 ID 会变化；不要硬编码冷门的 ID，先调用 `Cursor.models.list()` 确认调用者有权访问。

模型对**本地是必需项**，对**云端是可选项**（服务端会从调用者账号解析一个默认值）。

## 生产环境最佳实践

把这些应用到任何无人值守运行的集成中：

1. **把每个 `Agent.create` / `Agent.prompt` / `Agent.resume` 都包在带 `[Symbol.asyncDispose]()` 的 try/finally 中。** 不可商量。
2. **区分启动失败与运行失败** —— `CursorAgentError` 用退出码 1，`result.status === "error"` 用退出码 2，只有 `finished` 才用退出码 0。让 CI 失败真正可读。
3. **在 `send()` 之后立刻记录 `run.id` 和 `agent.agentId`**，要在流式输出之前。如果流挂住了，正是这些 ID 是你在 dashboard 或通过 `Agent.getRun(...)` 排查时需要的。
4. **尊重 `error.isRetryable`** —— 这是后端在告诉你该特定失败可以安全重试。盲目重试可能造成重复的云端运行；尊重这个标志则不会。
5. **使用 `local: { settingSources: [] }`（默认），除非你需要环境配置。** 选择 `"all"` 会从调用者环境加载项目/用户/团队/MDM 设置，这很少是服务所需要的。注意：`settingSources` 位于 `local` 之下，而不是顶层；它对云端 agent 没有影响（云端始终遵循团队/项目/plugins）。
6. **对 CI 中的云端 agent，设置 `skipReviewerRequest: true`**，除非应该传呼人工处理——它会抑制 reviewer 请求步骤，让 PR 通知保持安静。
7. **在共享基线设施的代码中始终显式传入 `apiKey`**，而不要依赖环境变量。这让凭据依赖一目了然，并防止跨租户错误。
8. **对真正的单次任务优先使用 `Agent.prompt(...)`** —— 它会替你释放资源，更难泄漏。

带示例的更长版本：[`references/patterns.md`](references/patterns.md)。

## 观察一个不是你启动的运行

你可以在之后按 ID 检查任何 agent/运行：

```typescript
// Cloud: IDs that start with "bc-" auto-route to the cloud API
const info = await Agent.get("bc-abc123", { apiKey });
const run = await Agent.getRun(runId, { runtime: "cloud", agentId: "bc-abc123", apiKey });

// Local: you need the cwd where the agent was created
const localInfo = await Agent.list({ runtime: "local", cwd: process.cwd() });
```

一个以 `bc-` 开头的云端 agent ID **不是**运行 ID。如果你只有运行 ID（来自日志或 webhook），把它连同运行时提示一起传给 `Agent.getRun`；不要把两者混淆。

## 提供 Canvas

如果用户集成会监控、列出或可视化 agent——活跃运行的 dashboard、对话回放、工具调用时间线——提供一个 Cursor Canvas 来渲染它。如果他们接受，完全交由 `canvas` skill 处理。

## 本 skill 不涵盖的内容

- Cloud Agents REST API（`/v1/agents/*`）。如果用户需要非 TS 客户端，REST API 单独记录在 <https://cursor.com/docs/cloud-agent/api>；在假定它与 SDK 功能对等之前，先到那里确认当前能力。
- `.cursor/hooks.json` hooks。云端 agent 会执行它们，但 SDK 不管理它们；参见 Cursor 的 Hooks 文档。
- 私有 worker / 自托管云端。把用户导向 Private Workers 文档。
- Python / 非 TS SDK。撰写本文时还没有其他语言的一方 SDK；REST 是可移植的选项。

