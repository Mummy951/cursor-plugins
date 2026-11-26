---
name: port-github-app-to-origin
description: >-
  规划将现有 GitHub App 移植为 Cursor Origin App 的方案。当任务是把 GitHub App
  迁到 Origin，或对比它所使用的能力与 Origin API 时使用。从应用代码中读取其需求，
  映射到实时的 Origin 规范，并撰写一份移植简报，附上给 Cursor 的反馈。仅做规划。
license: MIT
compatibility: >-
  运行时需要访问 https://cursor.com/docs/api/origin/* 的网络权限。
---

# 将 GitHub App 移植为 Origin App

在应用的代码库内运行此 skill。产出是为团队准备的移植简报，外加一节他们可以原样发送的 Feedback for Cursor
(`references/brief.md`)。本 skill 只做规划。除非用户在读完简报后明确要求，否则不编写或改动代码。先遵循
`origin-api` skill 中的文档与检查规则。另有两条规则：

1. **在代码中找，不要问。** 从源码中读出应用使用了什么。找不到的任何东西都变成向团队提出的问题。
2. **反馈描述用例，而非团队的代码。** 简报中属于团队的部分可以引用 `file:line`。Feedback for Cursor
   只用 Origin 的术语说明应用需要做什么、以及 Origin 为此缺了什么，不含文件路径、模块名、框架细节或仓库名。

## 需要查找什么

为每一项记录 `file:line`，并注明你找过但没有找到的内容。

- 声明的权限与事件，来自 manifest 或基础设施代码（若有签入）。否则从调用中推导。
- 应用处理的每个 webhook 事件，以及每个处理器读取的每个 payload 字段，包括仅用于记录日志的字段。
- 每个 REST 与 GraphQL 调用，包含代码传入的参数与过滤条件、读取的响应字段、它是在每个 webhook 上运行还是在循环中运行、以及它如何分页。
- 认证：JWT 算法、应用在安装后如何获知 installation ID、如何处理 token 过期、任何用户登录及其用途、以及应用是否克隆或推送 git。
- webhook 接收端：如何验证签名、验证时是否持有原始 body、以及如何对投递去重。
- 框架或库为应用发起的调用。Probot 的 receiver、token 缓存与配置加载器；Octokit `App` 的安装与仓库列表；app-auth 库。阅读该依赖的文档，并列出这些调用，标记为“from `<dependency>`”。

## 如何映射

简报需要完整的规范，但 `openapi.yaml` 和 `llms-full.txt` 各有数百 KB，大到无法整体读入上下文。如果可能，把它们保存到本地，放在应用仓库之外的临时位置，以免工作树中的任何内容被覆盖或残留，然后搜索它们并只读取匹配的部分：

- `openapi.yaml` 中的 `x-origin-scopes:` 标记出每个操作的 scope；`operationId` 在其上一行。
- `openapi.yaml` 中的 `x-origin-webhook-events:` 标记出每个 webhook payload schema 及其对应的事件 slug。
- `llms-full.txt` 中的 `### <Endpoint name>` 和 `### <Payload type>` 标题开始一节内容，用点号路径写出该端点或 payload 的字段；从该标题读到下一个 `###`。

若之后只有一个问题，从 `llms.txt` 开始，只抓取那一节。

- 名称匹配只是一个候选，不是答案。对照代码传入和读取的内容，阅读该操作的描述、参数与响应字段。若代码依赖的某个参数或字段缺失，那是变通方案或缺口，不是匹配。
- GitHub 中位于 `/issues/{n}/…` 下的 pull request 调用（评论、标签），在 Origin 上位于 pull request 端点下。若代码在真实 issue 上使用它们，参见 `references/brief.md` 中的 "Where GitHub features live on Origin"。
- Origin 没有 GraphQL。把每个查询拆成 REST 调用，并记录扇出。
- 需要请求的 scope 是你所列操作上 `x-origin-scopes.scopes` 的并集。不要直接照搬 GitHub manifest。
- 每个 GitHub 事件与 action 组合最多映射到 "Events" 中的一个 slug。action 是 slug 的一部分。没有对应 slug 的组合在 Origin 上不是事件。
- 对代码读取的每个 payload 字段，记录它是在 payload 中、在 payload 外围的投递信封中（`event.type` 携带 action）、需要额外读取（说明是哪个操作以及每个事件的调用次数）、可由其他字段计算得出，还是缺失。payload 是快照。REST 资源有但 payload 缺的字段需要额外读取。
- 文档从未提及的能力目前不可用。就此询问团队。文档既未确认也未否认的行为，得到一个提问，外加端到端测试中检查该项的一个步骤。不要假定它按 GitHub 上的方式工作。

## 完成之前

关于 Origin 的每一项论断都指向已保存文档中的某处。每个缺口都有一条反馈条目，说明其代价。反馈不泄露团队内部的任何信息。总结中要提出“原生还是镜像”这一问题。
