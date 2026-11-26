
### Andon
在整个任务树中停止新的派生。仅当有具体证据表明继续派生会产生垃圾时才提出：上游输出错误且下游任务将因此失败、验证器级联表明验收有误、认证或基础设施无法恢复。某个任务遇到自身障碍属于 `Status: blocked` 交接，而非 Andon。

  bun <path-to-orchestrate>/scripts/cli.ts andon raise --reason "<why>"{{agentIdFlag}} --workspace <workspace>

`--reason` 为必填项，会发布到运行线程，以便整个任务树能看到编排为何暂停。相关反应是子级轮询的低成本门控。`--agent-id` 会添加一个页脚链接，指回提出该请求的 agent。
