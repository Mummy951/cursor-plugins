# 集成模式

五种起始模板，对应人们实际构建的场景形态。复制其中一个，删除你不需要的部分。

每种模式都应用了本 skill 其余部分的[错误处理](error-handling.md)和[流式传输](streaming.md)最佳实践——适配时不要把它们去掉。

---

## 1. GitHub Action：对 PR 进行自动化代码审查

目标：当一个 PR 打开时，对其运行一个云端 agent，并发布审查评论。

```typescript
import { Agent, CursorAgentError } from "@cursor/sdk";

async function main() {
  const {
    PR_URL,
    REPO_URL,
    HEAD_REF,
    BASE_REF,
    CURSOR_API_KEY,
    GITHUB_TOKEN,
  } = process.env;

  if (!CURSOR_API_KEY || !REPO_URL || !HEAD_REF) {
    console.error("Missing required env: CURSOR_API_KEY, REPO_URL, HEAD_REF");
    process.exit(1);
  }

  await using agent = Agent.create({
    apiKey: CURSOR_API_KEY,
    model: { id: "composer-2" },
    cloud: {
      repos: [{ url: REPO_URL, startingRef: HEAD_REF }],
      workOnCurrentBranch: true,
      skipReviewerRequest: true, // 不重复呼叫审查者
    },
    mcpServers: GITHUB_TOKEN
      ? {
          github: {
            type: "stdio",
            command: "npx",
            args: ["-y", "@modelcontextprotocol/server-github"],
            env: { GITHUB_TOKEN },
          },
        }
      : undefined,
  });

  const prompt = `Review the changes on ${HEAD_REF} vs ${BASE_REF} for ${PR_URL}.
Focus on: correctness, security, readability. Post GitHub review comments inline
for concrete issues. No praise-only comments. If nothing to flag, say so.`;

  try {
    const run = await agent.send(prompt);
    console.log(`[review] agent=${agent.agentId} run=${run.id}`);

    for await (const event of run.stream()) {
      if (event.type === "status") console.log(`[review] ${event.status}`);
      if (event.type === "tool_call" && event.status !== "running") {
        console.log(`[review] tool: ${event.name} -> ${event.status}`);
      }
    }

    const result = await run.wait();
    if (result.status !== "finished") {
      console.error(`[review] run ${result.id} ended as ${result.status}`);
      process.exit(2);
    }
    console.log(`[review] done: ${result.durationMs}ms`);
  } catch (err) {
    if (err instanceof CursorAgentError) {
      console.error(`[review] startup failed: ${err.message}`);
      process.exit(err.isRetryable ? 75 : 1); // 瞬时错误返回 EX_TEMPFAIL
    }
    throw err;
  }
}

main();
```

为什么采用这种形态：

- 云端运行时（需要发布 GitHub 评论；独立于 runner 工作）。
- `skipReviewerRequest: true` 让该 action 在 CI 中保持安静。
- 退出码：`0` 表示完成，`1` 表示永久性启动失败，`2` 表示运行结束但状态为 `error`，`75` 表示瞬时可重试失败。

---

## 2. 定时分类：带续跑的 cron 驱动云端运行

目标：每天早上，恢复昨天的分类 agent，并让它对今天新增的 Linear 工单进行分类。

```typescript
import { Agent, CursorAgentError } from "@cursor/sdk";
import { readFile, writeFile } from "node:fs/promises";

const STATE_PATH = "/var/lib/triage/state.json";

const mcpServers = {
  linear: {
    type: "http" as const,
    url: "https://mcp.linear.app/sse",
    headers: { Authorization: `Bearer ${process.env.LINEAR_API_KEY!}` },
  },
};

async function main() {
  const state = await readState();

  await using agent = state.agentId
    ? Agent.resume(state.agentId, {
        apiKey: process.env.CURSOR_API_KEY!,
        model: { id: "composer-2" },
        cloud: { repos: [{ url: process.env.REPO_URL!, startingRef: "main" }] },
        mcpServers, // 续跑时必须重新传入
      })
    : Agent.create({
        apiKey: process.env.CURSOR_API_KEY!,
        model: { id: "composer-2" },
        cloud: { repos: [{ url: process.env.REPO_URL!, startingRef: "main" }] },
        mcpServers,
      });

  console.log(`[triage] agent=${agent.agentId}`);

  try {
    const run = await agent.send(
      "Triage new Linear tickets opened in the last 24h. Label, assign, comment with next steps."
    );
    const result = await run.wait();
    if (result.status === "error") {
      console.error(`[triage] run ${result.id} errored`);
    }
    await writeState({ agentId: agent.agentId, lastRunId: result.id });
  } catch (err) {
    if (err instanceof CursorAgentError && err.isRetryable) {
      console.error(`[triage] transient: ${err.message}, will retry next tick`);
      return;
    }
    throw err;
  }
}

async function readState(): Promise<{ agentId?: string; lastRunId?: string }> {
  try { return JSON.parse(await readFile(STATE_PATH, "utf-8")); }
  catch { return {}; }
}
async function writeState(s: { agentId: string; lastRunId: string }) {
  await writeFile(STATE_PATH, JSON.stringify(s));
}

main();
```

为什么采用这种形态：

- 在多次 cron 调用之间持久化 `agentId` 可以保留对话记忆（例如"记住我们昨天已经分类过哪些工单"）。
- 每次续跑都要重新传入 MCP。
- 对可重试错误优雅处理：跳过本次，下次再试。

---

## 3. 一次性分析脚本

目标：开发者针对本地仓库运行一条命令，得到一份书面分析。

```typescript
#!/usr/bin/env node
import { Agent } from "@cursor/sdk";

const prompt = process.argv.slice(2).join(" ").trim();
if (!prompt) {
  console.error("Usage: analyze.ts <question>");
  process.exit(1);
}

const result = await Agent.prompt(prompt, {
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  local: { cwd: process.cwd() },
});

console.log(result.result ?? "(no output)");
process.exit(result.status === "finished" ? 0 : 2);
```

为什么采用这种形态：

- `Agent.prompt` 会自动为你释放资源。非常适合一次性 CLI。
- 无流式传输——这是一次性的。
- 退出码携带状态；可在 shell 管道中使用（`&&`、`|| fallback`）。

---

## 4. 后端服务：HTTP API 之后的用户级 agent

目标：你的后端服务暴露一个端点，代表用户运行一个 Cursor agent。每个用户都有自己持久化的 agent。

```typescript
import express from "express";
import { Agent, CursorAgentError } from "@cursor/sdk";

const app = express();
app.use(express.json());

// 假设这存在你的数据库中
const userAgents = new Map<string, string>();

app.post("/agents/:userId/send", async (req, res) => {
  const { userId } = req.params;
  const { prompt } = req.body;
  const apiKey = process.env.CURSOR_SERVICE_ACCOUNT_KEY!;

  const existing = userAgents.get(userId);
  const options = {
    apiKey,
    model: { id: "composer-2" as const },
    cloud: { repos: [{ url: "https://github.com/your-org/workspace", startingRef: "main" }] },
  };

  try {
    await using agent = existing
      ? Agent.resume(existing, options)
      : Agent.create(options);

    const run = await agent.send(prompt);
    const result = await run.wait();

    userAgents.set(userId, agent.agentId);
    res.json({
      agentId: agent.agentId,
      runId: result.id,
      status: result.status,
      durationMs: result.durationMs,
    });
  } catch (err) {
    if (err instanceof CursorAgentError) {
      res.status(err.code === undefined ? 500 : 502).json({
        error: err.constructor.name,
        message: err.message,
        retryable: err.isRetryable,
      });
      return;
    }
    res.status(500).json({ error: "internal" });
  }
});

app.listen(3000);
```

为什么采用这种形态：

- 使用服务账号密钥，而非用户密钥——这是共享基础设施。
- 每个请求使用 `await using`；请求之间不留存留的 agent 句柄。
- `userAgents` 是你数据库的一个替身；按用户持久化 `agentId` 以便续跑。
- 错误响应会将 `isRetryable` 传递出去，以便调用方智能退避。

如果你需要向客户端流式返回响应，则不要这样做——改用 `run.stream()` 接入 server-sent-events 端点。

---

## 5. 扇出：并行对多个仓库运行 agent

目标：为列表中的每个仓库派生一个云端 agent，并收集结果。

```typescript
import { Agent, CursorAgentError } from "@cursor/sdk";

async function dispatchOne(repoUrl: string, apiKey: string, prompt: string) {
  await using agent = Agent.create({
    apiKey,
    model: { id: "composer-2" },
    cloud: {
      repos: [{ url: repoUrl, startingRef: "main" }],
      autoCreatePR: false,
      skipReviewerRequest: true,
    },
  });
  try {
    const run = await agent.send(prompt);
    const result = await run.wait();
    return { repoUrl, agentId: agent.agentId, status: result.status, runId: result.id };
  } catch (err) {
    if (err instanceof CursorAgentError) {
      return { repoUrl, error: err.constructor.name, message: err.message };
    }
    throw err;
  }
}

const repos = [
  "https://github.com/your-org/service-a",
  "https://github.com/your-org/service-b",
  "https://github.com/your-org/service-c",
];

const prompt = "Audit Dockerfile for outdated base images. Propose an update.";
const apiKey = process.env.CURSOR_API_KEY!;

const results = await Promise.allSettled(repos.map(r => dispatchOne(r, apiKey, prompt)));
console.log(JSON.stringify(results, null, 2));
```

为什么采用这种形态：

- `Promise.allSettled`——一个仓库失败不会拖垮其他仓库。
- 使用云端运行时，因为云端 agent 实际上是独立的虚拟机；本地扇出会通过调用方的机器串行化。
- 每个 agent 都使用 `await using`，这样即使其他 agent 仍在运行，每个 agent 也能完成清理。
- `autoCreatePR: false`，因为我们要先审查结果；在挑选出通过审计的仓库之后，再运行后续操作来创建 PR。

**速率限制意识**：一次扇出 100 个 agent 会触发后端限制。当 N 较大时，进行批处理：

```typescript
async function runInBatches<T, R>(items: T[], size: number, fn: (item: T) => Promise<R>) {
  const results: R[] = [];
  for (let i = 0; i < items.length; i += size) {
    const batch = items.slice(i, i + size);
    results.push(...await Promise.all(batch.map(fn)));
  }
  return results;
}
```

---

## 跨领域最佳实践

适用于所有五种模式：

- **在流式传输之前记录 `agent.agentId` 和 `run.id`。** 每一次故障排查都从这两个 ID 开始。
- **按失败类型区分退出码/响应码。** 你的运维团队应该一眼就能分辨"无法通过身份验证"和"agent 做了工作但出错"。
- **尊重 `isRetryable`。** 对瞬时错误退避，对终止性错误不要重试。对失败的云端运行盲目重试会产生重复的 PR。
- **`await using` 或显式 `finally`。** 每个 `Agent.create` / `Agent.resume` 都需要一条明确的资源释放路径。
- **显式传入 `apiKey`。** 不要在共享基础设施代码中依赖环境里的隐式变量。
- **不要提交 `CURSOR_API_KEY`。** 不要记录它。当需要确认正在使用哪个密钥时，只显示前缀（`cursor_01ab...`）。
