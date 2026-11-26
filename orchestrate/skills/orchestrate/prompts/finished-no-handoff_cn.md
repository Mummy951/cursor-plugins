<!-- orchestrate finished-no-handoff
task: {{taskName}}
branch: {{branch}}
agentId: {{agentId}}
runId: {{runId}}
resultStatus: {{resultStatus}}
terminatedAt: {{terminatedAt}}
-->

# {{taskName}} 未进行交接即结束

状态：{{resultStatus}}（云端 agent 正常结束，但从未写入 `## Status` 交接内容）
云端 agent：{{agentId}}
运行：{{runId}}
分支：{{branch}}
终止时间：{{terminatedAt}}

## 建议的后续步骤
- 查看 `handoffs/{{taskName}}.md` 中的原始交接内容，了解 worker 实际输出了什么。
- 如果这看起来像是提示词未命中目标（worker 生成了叙述文字但没有按结构化模板输出），请按原样重试。
- 放弃：如果目标确实没有可接受的产出，则跳过该任务并围绕它重新规划。{{rawSnippetBlock}}
