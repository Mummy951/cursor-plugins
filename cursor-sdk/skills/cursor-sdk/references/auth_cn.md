# 身份验证

简而言之：`export CURSOR_API_KEY=cursor_...` 和 `Agent.create({ apiKey: process.env.CURSOR_API_KEY! })`。本页其余内容都是在这不起作用时你才需要阅读的部分。

## 密钥类型

SDK 接受两种密钥；两者都适用于本地和云端。


| 密钥类型               | 签发位置                                                                           | 使用时机                                                       |
| ---------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| 用户 API 密钥          | [https://cursor.com/dashboard/cloud-agents](https://cursor.com/dashboard/cloud-agents) | 开发工具、个人脚本、以特定用户身份运行                         |
| 团队服务账号密钥       | Team Settings → Service accounts                                                   | 共享 CI、后端服务、任何不应由真人持有密钥的场景                 |


两者都放在同一个 `apiKey` / `CURSOR_API_KEY` 位置——没有二等公民之分。令牌格式相同；无法通过检查来区分它们。

## SDK 如何查找密钥

优先级顺序：

1. 传递给 options 对象的 `apiKey`（`Agent.create`、`Agent.prompt`、`Agent.resume`、`Agent.get` 等）
2. `process.env.CURSOR_API_KEY`

仅此而已——SDK 本身没有配置文件，也没有钥匙串集成。对于共享的基础设施代码，**始终显式传递 `apiKey`**，而不要依赖环境变量，这样凭据依赖关系在调用点就一目了然。

一些仅限云端的辅助方法（`Agent.archive`、`Agent.delete`、`Cursor.me`、`Cursor.models.list`、`Cursor.repositories.list`）也接受 `apiKey` 作为命名选项。当它们未收到该选项时，会回退到 `CURSOR_API_KEY`。

## 最小可行配置

```bash
export CURSOR_API_KEY="cursor_..."
```

```typescript
import { Agent } from "@cursor/sdk";

const agent = Agent.create({
  apiKey: process.env.CURSOR_API_KEY!,
  model: { id: "composer-2" },
  local: { cwd: process.cwd() },
});
```

非空断言（`!`）是一种可读的表达方式，意思是"如果环境变量缺失就大声失败"——否则 SDK 稍后会抛出身份验证错误，而堆栈跟踪不会指向该环境变量。

一个稍微更礼貌的模式：

```typescript
const apiKey = process.env.CURSOR_API_KEY;
if (!apiKey) {
  console.error("Missing CURSOR_API_KEY. Mint one at https://cursor.com/dashboard/cloud-agents.");
  process.exit(1);
}
```

## 身份验证失败的症状


| 症状                                                     | 诊断                                                                       |
| -------------------------------------------------------- | -------------------------------------------------------------------------- |
| 首次 `send()` 时出现 `AuthenticationError: ...`          | 密钥缺失、过期或格式错误（包括空白字符）                                   |
| `ConfigurationError: BAD_USER_API_KEY`                   | 密钥在语法上无效                                                           |
| 仅云端出现 `AuthenticationError`，本地正常               | 密钥无法访问云端 agents 接口——可能是权限问题                               |
| 云端运行报错 `ERROR_GITHUB_NO_USER_CREDENTIALS`          | 调用方没有与目标仓库的 GitHub 连接                                         |
| 本地正常，CI 中返回 401                                  | 环境变量没有进入 CI 作业（引号、secret 作用域、scope 等原因）              |
| 间歇性 401                                               | 几乎总是密钥轮换问题，或两个相互冲突的 `CURSOR_API_KEY` 值                  |


### `ERROR_GITHUB_NO_USER_CREDENTIALS` 不是代码 bug

云端运行会使用调用方的 GitHub 凭据来克隆目标仓库。如果 `CURSOR_API_KEY` 背后的用户从未在 Cursor 仪表盘中连接过 GitHub，云端 agent 就无法访问该仓库，运行会立即失败。修复方法：调用方在仪表盘中关联 GitHub。无需修改代码。

对于服务账号密钥，该服务账号需要像普通用户一样配置 GitHub 访问权限。

## 轮换密钥

- 生产工作负载应按计划轮换；不要把密钥烘焙进镜像中。
- 密钥泄漏时，先在仪表盘中吊销，再签发新密钥。SDK 会在下次进程启动时获取新的环境变量值。
- 优先使用按环境划分的密钥（staging、prod），而不是一个全局密钥，这样吊销操作可以限定范围。

## 一个进程中使用多个密钥

完全支持——每次 `Agent.create` / `Agent.prompt` 调用都是独立的。显式传递 `apiKey`，以免意外回退到调用方并未考虑到的环境变量：

```typescript
const userAgent  = Agent.create({ apiKey: userKey,  /* ... */ });
const botAgent   = Agent.create({ apiKey: botKey,   /* ... */ });
```

## 本地开发

如果你正在开发集成并快速迭代，将密钥保存在被 gitignore 忽略的 `.env` 文件中，并通过类似 `dotenv` 的工具加载：

```typescript
import "dotenv/config";
// Now process.env.CURSOR_API_KEY is populated.
```

不要提交 `.env`。SDK 本身不会读取 `.env`。

## CI 检查清单

- 将密钥放入 secrets 存储，而不是工作流文件中。
- 将 secret 限定在需要它的特定作业/工作流范围内。
- 对于 GitHub Actions，使用仓库级或环境级 secret（环境级还提供审批关卡）。
- 如果需要确认正在使用哪个密钥，只向日志打印密钥前缀（前 6 个字符），绝不要打印完整值。
- 密钥缺失时快速失败——不要让作业运行 10 分钟才在最后返回 401。

## 用于生产集成的服务账号

对于任何不是个人开发脚本的场景：

1. 在团队设置中创建一个团队服务账号。
2. 只授予它所需的权限。
3. 为它签发一个密钥，并将密钥存储在你的 secrets 管理器中。
4. 如果该集成会派生云端 agents，请为该服务账号（而非操作员个人账户）关联 GitHub。
5. 针对该服务账号监控使用情况——如果一切都用同一个密钥运行，你就会失去按调用方归因的能力。
