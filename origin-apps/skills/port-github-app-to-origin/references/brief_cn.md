# 简报与反馈

## 简报

在仓库根目录写一个 Markdown 文件（除非团队另有命名约定，否则为 `ORIGIN-PORTING-BRIEF.md`），并打印其路径。小型应用约 800 词，大型应用约 2,000 词。任何无话可说的部分都略去。表格形状随应用而定。一份好的简报应当：

- 以摘要开篇。给出结论（可直接移植、需变通方案移植，或因 X 受阻），说明决定其余内容的问题（通常是仓库是原生的还是镜像的），以及是否有给 Cursor 的反馈、其中是否有任何一条会阻塞移植。
- 只列出无法原样沿用的能力。对每一项，说明它映射到的 Origin 操作、事件 slug 或文档章节，或注明没有对应项。列表末尾用一行收束其余部分，例如"14 个操作可直接映射；参见 scopes 一行"。仅在 webhook payload 字段缺失或需要额外 API 调用时才列出该字段。以需要申请的 scopes 结尾。
- 在有帮助时，为本应用描述一个端到端测试。包括选择哪些事件、如何检查仓库的镜像状态、应当到达的第一个事件及其内容、第一次写入。跳过通用设置；文档中的"Implementation checklist"已涵盖这部分。
- 说明是什么让这次移植的规模变大或变小，以及如何灰度发布（两个版本并行运行，或直接切换）。不要给出时间估算。
- 只问团队必须决定的问题。不要把一项发现包装成问题。
- 结尾用两行说明简报基于哪个 spec 版本（`info.version` 与获取时间）以及哪个代码提交，然后以不带编号的 Feedback for Cursor 作为最后一节。

关于应用的每一项论断都要用 `file:line` 佐证，或当是某个库替应用完成时用"from `<dependency>`"佐证。关于 Origin 的每一项论断都要有已保存文档中的依据。如果要在表格中用短标签，请使用朴素的措辞：works as is、workaround（说明代价）、not available（询问团队）、gap（撰写反馈）。

## 给 Cursor 的反馈

Cursor 希望听到团队需要什么。凡是会阻塞团队主流程、损害其正确性、安全性或规模，或者团队希望 Origin 提供的能力，都应提出。下面的测试只是把条目归类为反馈（Origin 应当增加的能力）和问题（团队必须做的决定）。它们绝不决定是否要发声。

变通方案是用另一种方式得到相同结果，例如多一次读取、换一个标识符、改一条路径、在客户端过滤，或使用应用自己控制的标记。它往往是正确答案。当它带来以下任一代价时，就变成 gap，并获得一条反馈：

- fan-out，即每个事件额外的调用，且在本应用的量级下随仓库规模或活动规模增长；
- 可能错误的答案，例如猜测哪个 check run 或评论是应用自己的、从多个版本共用的 SHA 反推 pull request，或构造一个文档并未承诺其格式的 URL；
- 更宽的 scope、更长寿命的 token，或在只需安装 token 的地方使用用户凭证；
- 改变团队用户所见或所能做的事；
- 应用的主流程或其首个端到端测试所依赖的能力。

应用存在就是为了对其做出反应的状态变更，若既无事件、也无其他察觉方式，同样是一个 gap。文档从未提及的东西今天尚不可用，会得到一个 question；只有当它阻塞主流程时才成为反馈。

不是反馈、只是问题或备注的情形：代码未使用的字段或过滤器；不同但有机械替代方式的约定；文档化的设计选择，例如 token 寿命或不提供 GraphQL。

每个 gap 写一条条目，用 Origin 术语表达，不透露团队内部信息。当至少有一条条目时，也在简报旁边把该节写入 `ORIGIN-FEEDBACK.md`。当一条都没有时，不写文件，并用一行说明。该文件不含许可证头、仓库名、产品名，也不提及任何其他 forge。如果文档与观察到的行为不一致，把它放入反馈末尾简短的"Docs questions for Cursor"列表，而不是团队的问题中。建议的形状：

```markdown
### Feedback: <capability, in Origin terms>
- **Use case:** the app needs to <do what, for whom>, <how often or at what volume>.
- **Origin today:** <what is missing or costly; cite the closest operationId, slug, or section, or "no operation">.
- **Workaround considered:** <the route and the cost that makes it insufficient, or "none found">.
- **Blocking?** yes / no, for which flow.
- **Spec version checked:** <info.version>, <date>.
```

描述能力。不要提议 scope、字段或路由名称。反馈由团队发送，而不是你；他们会在发送前移除任何泄露其内部信息的内容。

## GitHub 特性在 Origin 上的位置

在把任何东西称为 gap 之前先查这里，然后在已保存的文档中确认。这份列表会过时；以文档为准。

有 Origin 对应项：安装回调参数 → "Installation receipt"；RS256 应用 JWT → "App JWT"；长期安装 token → "Installation access token"；权限 → "Scopes" 与 `x-origin-scopes`；数字 ID 与 `/repositories/{id}` → "IDs"、"Repository paths"；`Link` 分页与总数 → "Pagination"；commit statuses → 带稳定 `key` 的 check runs（"Check runs"）；pull request 上的 `/issues/{n}/comments` 与 `/issues/{n}/labels` → pull request 端点；仓库 webhook 增删改查 → Create App 与 Update App 上的应用 `events` 列表；带 `action` 字段的单个 `pull_request` 事件 → 每个 action 一个 slug（"Events"）；`x-github-*` 头与 HMAC 签名 → "Headers"、"Signature verification"；GitHub 在 payload 中内联的数据（变更文件、before-SHA、URL、用户资料）→ 额外读取（"Resource references"、"Current limitations"）；以 commit SHA 为键的 reviews → `pullRequestVersion`；按作者查找应用自己的 check runs 或评论 → check run 的 `key`，或应用自己控制的标记；用户登录与以用户身份操作 → "Acting on behalf of users"（用户确认回执、installation user tokens）。

GitHub 镜像：安装只能读取元数据与内容，合并 pull request 或更改默认分支不可用。每次写入都需要一个原生仓库，即一个在 Origin 上创建的仓库（"Mirrored repositories"）。

当前 spec 中没有（询问团队；仅当阻塞主流程时才作为反馈）：GraphQL（把每个查询拆成 REST 调用）；Issues（pull request 的评论、讨论串、reviews 与 labels 覆盖了 pull request 这一半）；OAuth 应用 token 铸造；写入任意 blob 或 tree（已有 commit-from-files 与 push）；查找用户、邮箱、团队或成员（reviewer 标识符按 public id、邮箱或 group slug 解析）；读取组成员关系或用户的实际权限。对最后两项，应指名的变通方案是限制到某个仓库与 scope 的用户 token。除非用户持有该权限，否则铸造它会返回 `403`，因此它也可兼作权限检查。
