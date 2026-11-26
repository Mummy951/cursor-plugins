---
name: origin-api
description: >-
  把关于 Cursor Origin API 的问题引导到 Origin 文档中正确的章节，并指出应先检查的
  几条规则。当任务中提到 Origin、Origin API、Origin Apps 或 Origin webhooks 时使用，
  包括创建 Origin App、以 Origin App 身份进行认证、调用 Origin 端点，或处理
  Origin webhook 投递。
license: MIT
compatibility: >-
  运行时需要能访问 https://cursor.com/docs/api/origin/* 的网络。
---

# Origin API

文档是唯一的事实来源。不要凭记忆说出端点、scope、事件 slug、请求头或限制值。
当本文件与文档不一致时，以文档为准。

文档位于 `https://cursor.com/docs/api/origin/`。`llms.txt` 是索引，链接到每个
章节、端点和 webhook 负载。`openapi.yaml` 是契约，其中「Endpoint reference」
解释了它的 `x-origin-*` 扩展。`llms-full.txt` 是完整的参考文档，合并为一个文件。
`changelog` 说明有哪些变更。面对单个问题，先看 `llms.txt`，只抓取能回答该问题的
那一节。`llms-full.txt` 和 `openapi.yaml` 各有几百 KB，大到无法整体读入上下文。
当你需要全部内容时（例如编写移植说明），尽可能把它们保存到本地，放在你正在操作的
任何仓库之外，再在文件中搜索所需的章节标题或注解，只阅读匹配的部分。引用时请给出
`operationId` 和章节名称。

## 应先检查的规则

1. **原生仓库还是镜像仓库。** 一个仓库要么是原生仓库（在 Origin 上创建），
   要么是 GitHub 镜像。在原生仓库上，安装拥有其全部 scope 和所有写权限。
   在 GitHub 镜像上，除 `repository.pushed` 之外的每个事件仍会到达，
   除元数据和内容读取之外的所有调用都会返回 `403`，并且无法合并拉取请求或更改
   默认分支（见「Mirrored repositories」「Events」）。
2. **订阅。** 只有 `installation.*` 事件无需订阅即可到达。缺少订阅只会造成
   静默无事件，而不会报错（见「Events」）。
3. **校验、去重、确认。** 在解析之前先基于原始请求体验证签名，按投递 ID 去重，
   返回 `2xx`，然后再处理（见「Signature verification」「Retries」
   「Automatic disable」）。Origin 签名的是摘要（digest），而 Standard Webhooks
   不会这样做，因此通用的校验器会失败。
4. **scope 来自规范。** 请求应用所调用各操作中 `x-origin-scopes.scopes` 的并集
   （见「Scopes」）。
5. **不透明的令牌和 ID。** 不要构造或解析分页令牌或 ID（见「Pagination」「IDs」）。

索引没有明确写出的一节名称：乱序到达的 check-run post 会怎样，这部分内容在
「Ordering writes」下。

移植已有的 GitHub App：使用本 plugin 中的 `port-github-app-to-origin`。
