# 适配器（Adapters）

适配器是外部 IO 外壳。它们不拥有编排状态。

## SlackAdapter

`SlackAdapter` 是 orchestrate 唯一自带的适配器。Slack 是面向人可见的一层：

- 发布运行启动消息
- 在启动消息所在的线程中，为每个任务发布或编辑一条任务消息
- 当操作员明确要求时上传文件
- 为 Andon 读取表情回应（reactions）
- 在运行线程中发布自由格式的评论

`postRunKickoff` 是唯一一个写入频道根部的适配器方法。CLI 会一次性解析频道，然后用该频道构造适配器。之后每一次 Slack 写入都会带上 `threadTs`，并停留在同一个运行线程中。

Orchestrate 负责维护 Slack 状态镜像、Andon 以及评论重试队列。Agent 仍然可以直接调用 MCP 来处理 Linear、GitHub、Slack、Notion 以及其他临时的外部工作。这些系统并不是适配器的目标端。

## 评论重试队列

`comment-retry-queue.json` 按目标字符串存储必需的评论。合法的 Slack 格式为 `slack:<channel>:<thread_ts>`。工作区调用在发布或清空队列之前，会根据 `plan.slackKickoffRef` 校验目标。

常规生命周期镜像属于尽力而为（best effort）。必需的评论则按队列的退避策略进行重试。
