<!-- 编排失败交接
task: {{taskName}}
branch: {{branch}}
agentId: {{agentId}}
runId: {{runId}}
failureMode: {{failureMode}}
terminatedAt: {{terminatedAt}}
-->

# {{taskName}} 失败交接

状态：错误（云端 agent 终止时未写入交接文档）
失败模式：{{failureMode}}
云端 agent：{{agentId}}
开始时间：{{startedAt}}
终止时间：{{terminatedAt}}
持续时间：{{duration}}
最后活动：{{lastActivityLine}}
最后工具调用：{{lastToolCall}}
分支：{{branch}}
SDK 错误：{{sdkError}}

## 建议的后续步骤
{{suggestions}}
