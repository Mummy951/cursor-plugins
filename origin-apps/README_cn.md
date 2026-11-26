# Origin Apps

两个用于在 [Cursor Origin](https://cursor.com/docs/api/origin)（Cursor 的代码锻造平台）之上构建的 skill。它们涵盖创建 Origin App、调用 API、接收 webhook，以及迁移现有的 GitHub App。该插件仅包含 skill，因此可在 Cursor、Claude Code、Codex 以及任何读取 [Agent Skills](https://agentskills.io) 的 agent 中运行。

## 包含内容

`origin-api` 会引导 agent 阅读 Origin 文档中能回答其问题的章节，并列出需要优先检查的规则：原生仓库与镜像仓库、事件订阅、webhook 验证、规范中的 scope、不透明的 token 和 ID。任何 Origin 相关工作都可以使用它。

`port-github-app-to-origin` 会规划迁移现有 GitHub App 的方案。在该 App 的仓库内部运行它。它会从代码中读取该 App 使用了哪些能力，将其映射到现行的 Origin 规范，并撰写一份迁移简报，内容包括：哪些可以沿用、哪些不行、你的处理程序读取的 webhook 字段以及每个字段在 Origin 上的来源、需要申请的权限（scope）、针对你的 App 的端到端测试、给 Cursor 的反馈，以及你的团队必须决定的问题。它只做规划。除非你要求，否则它不会编写任何代码。

这两个 skill 都会在运行时读取现行规范，绝不凭记忆说出任何端点。

## 何时使用

询问 Origin API 相关的问题，或要求迁移某个 GitHub App 时，相应的 skill 就会加载。在 Cursor 中，你也可以运行 `/origin-api` 或 `/port-github-app-to-origin`。

## 在 Cursor 中安装

在 Cursor Marketplace 中搜索 Origin Apps
（[cursor.com/marketplace/origin-apps](https://cursor.com/marketplace/origin-apps)），
或打开 Customize，找到该插件，并在用户级别或项目级别安装它。

## 在 Cursor 之外使用

这些 skill 仅使用可移植的 [Agent Skills](https://agentskills.io) frontmatter。在 Claude Code 中：先运行 `/plugin marketplace add cursor/plugins`，再运行 `/plugin install origin-apps@cursor-plugins`。对于任何其他 agent：将 `origin-apps/skills/*` 复制到其 skills 文件夹（两个都要复制；迁移用的 skill 引用了 `origin-api`）。

## 要求

- 运行期间需要能够访问 `https://cursor.com/docs/api/origin/*` 的网络。
- 对于迁移用的 skill，需要有该 App 源代码的读取权限。撰写简报不需要 Origin 凭证。端到端测试由你之后运行简报时自行执行。

## 简报的存放位置

迁移用的 skill 会在仓库根目录写入 `ORIGIN-PORTING-BRIEF.md` 并打印其路径。当有给 Cursor 的反馈时，它还会将该部分写入 `ORIGIN-FEEDBACK.md`，可直接原样发送；简报的其余部分供你的团队使用。

## 许可证

MIT
