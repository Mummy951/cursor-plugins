# 运行时选择：本地 vs 云端

SDK 在两个截然不同的运行时之上暴露了同一套接口。选错运行时不会造成致命错误，但会浪费数小时。请谨慎选择。

## 决策树

从这里开始：

1. **agent 是否需要在 GitHub 上开 PR？** → 云端。本地 agent 只会修改 `cwd` 中的文件；它们不会创建分支或推送。
2. **agent 的运行时间是否会超过调用方进程的存活时间？**（触发即不管的定时任务、派生出隔夜作业运行的 webhook）→ 云端。agent 的存活时间会超过你的脚本。
3. **agent 是否需要调用方机器所不具备的算力、隔离环境或凭证？**（沙箱化求值、受控的环境变量、固定的运行时）→ 云端。
4. **这是一个开发循环脚本、已经检出仓库的 CI 步骤，还是针对用户当前项目的 CLI？** → 本地。仓库已经在磁盘上；云端会毫无必要地重新克隆它。
5. **用户是否希望在不向 GitHub 发起网络调用的情况下运行？**（气隙环境、非 GitHub 仓库、对未提交代码的实验）→ 本地。云端需要一个 GitHub 仓库 URL。

如果两点指向相反的方向，对于生产环境集成，云端通常是更安全的选择；对于开发工具，本地通常是更安全的选择。

## 能力矩阵

| 能力                               | 本地                                  | 云端（Cursor 托管）                                             |
| ---------------------------------- | ------------------------------------- | ------------------------------------------------------------- |
| 创建真正的 PR                      | 否                                    | 是（`cloud.autoCreatePR: true`）                              |
| 处理未提交的本地更改               | 是                                   | 否 —— 从 `startingRef` 克隆                                   |
| 存活时间超过调用方进程             | 否                                   | 是 —— 可通过 `agentId` 恢复                                   |
| 运行中途可取消                     | 是                                   | 是（服务端取消；以防万一请检查 `run.supports("cancel")`）      |
| 产物下载                           | 尚未实现                              | 是                                                           |
| MCP stdio 传输                     | 是                                   | 是（命令在云端 VM 内运行）                                    |
| MCP HTTP 传输                      | 是                                   | 是                                                           |
| 环境 Cursor 设置                   | 通过 `settingSources` 选择性启用      | 始终遵循企业/团队 hooks                                       |
| 需要 GitHub 仓库                   | 否                                   | 是（`cloud.repos[].url`）                                      |
| 需要 API key                       | 用于远程模型调用（大多数 prompt）     | 始终需要                                                      |

## 本地运行时 —— 实际工作原理

```typescript
const agent = Agent.create({
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  local: { cwd: "/absolute/or/relative/path/to/repo" },
});
```

- `cwd` 是 agent 读写文件的位置。该类型接受字符串数组，但本地执行器目前只使用第一个条目（`platform.ts` 的 `getCwd` 返回 `cwd[0]`）；在多根支持发布之前，请传入单个路径。
- agent 会在进程内派生辅助程序（工具执行、shell 运行器、MCP stdio 进程）。请彻底释放它们以回收这些资源。
- 持久化状态位于 `cwd` 下的 Cursor 数据目录中。`Agent.list({ runtime: "local", cwd })` 会在那里列出此前创建的 agent。
- 环境设置（项目规则、团队策略、团队配置的 MCP 服务器）默认**不会**加载。请在 **`local` 内部**传入 `settingSources` 来选择启用（例如 `local: { cwd, settingSources: ["project"] }`，或使用 `"all"` 加载全部）。
- 本地运行会以调用方的权限在调用方机器上执行工具。请像对待 `rm -rf` 一样对待 agent —— 限定好 `cwd` 的作用范围，不要在环境变量中放入你不想暴露的秘密。

何时优先选择本地：

- CLI 工具、编辑器集成、开发脚本。
- 仓库已经检出、且你希望直接检查目录树的 CI 步骤。
- 快速迭代：无需网络克隆、无需 PR、无需通知审查者。

## 云端运行时 —— 实际工作原理

```typescript
const agent = Agent.create({
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  cloud: {
    repos: [{ url: "https://github.com/your-org/your-repo", startingRef: "main" }],
    autoCreatePR: true,
    skipReviewerRequest: true, // Keep CI quiet; flip to false for review-worthy changes
  },
});
```

- Cursor 会配置一台 VM，在 `startingRef` 处克隆 `repos[].url`，运行 agent，推送一个分支，并（在 `autoCreatePR` 时）开启一个 PR。
- agent ID 以 `bc-` 为前缀（background composer）。SDK 辅助方法（`Agent.get`、`Agent.archive` 等）会根据该前缀自动路由。
- 调用方（`CURSOR_API_KEY` 背后的用户）必须拥有到目标仓库的 GitHub 连接。如果没有，云端会返回 `ERROR_GITHUB_NO_USER_CREDENTIALS` —— 这是环境配置问题，不是代码 bug。
- `run.cancel()` 在云端受支持（服务端取消）；仍请用 `run.supports("cancel")` 做保护，以保证可移植性。
- 仅当你希望 agent 推送到某个已有分支时才设置 `workOnCurrentBranch: true` —— 这种情况很少见，通常意味着你试图模拟本地运行；请改用本地运行时。

何时优先选择云端：

- 任何需要开 PR 供人工审查的工作。
- 不应阻塞本地进程的定时/自动化工作。
- 跨多个仓库或分支的并行扇出。
- 针对调用方尚未检出的仓库运行。

## 常见的"我本意是另一个"症状

- **"agent 创建成功了，但 GitHub 上什么都没发生"** —— 你本想传 `cloud:` 却传了 `local:`。本地不会推送。
- **"云端 agent 看不到我未提交的更改"** —— 这是设计如此。请提交更改，或改用本地。
- **"云端 agent 说找不到我的 GitHub 仓库"** —— 调用方的 Cursor 账户没有到该仓库的 GitHub 连接。这不是代码 bug；请在控制面板中处理。
- **"`run.cancel()` 在我的 run 上抛异常"** —— 通常是一个脱离的 run 句柄（对实时通道已消失的 run 调用 `Agent.getRun(...)`）。请在调用前用 `run.supports("cancel")` 做保护。
- **"我试图跨机器复用一个 agent"** —— 云端 agent 可以在任何地方恢复（`Agent.resume(bcId, { ... })`）。本地 agent 的作用范围限定在其 `cwd` 的数据目录；从另一台机器恢复会得到一个全新的 agent。

## 混合：在同一个脚本中同时运行本地和云端 agent

这完全没问题，也是很常见的模式（例如本地检查 + 云端 PR）。每个 agent 都是独立的；请释放二者。

```typescript
const localAgent = Agent.create({ /* ... local */ });
const cloudAgent = Agent.create({ /* ... cloud */ });
try {
  const summary = await Agent.prompt("Summarize the diff on HEAD", {
    apiKey,
    model: { id: "composer-2" },
    local: { cwd: process.cwd() },
  });
  const cloudRun = await cloudAgent.send(`Follow up on: ${summary.result}`);
  await cloudRun.wait();
} finally {
  await localAgent[Symbol.asyncDispose]();
  await cloudAgent[Symbol.asyncDispose]();
}
```
