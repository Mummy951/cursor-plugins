# 更新日志

## 1.0.0

- Advisor 插件首次发布。
- Skill：`advisor`（`/advisor [model|off|status|ask ...|nudge on|off]`），可用作 Custom Mode。
- Agent：`advisor-subagent`，一个只读 subagent，默认固定使用 Grok 4.6、努力程度（effort）为 xhigh（可通过 `/advisor <model>` 指定任意模型），返回裁决结论和指导建议。
- Hooks：跟踪自上次咨询以来的编辑，将每次咨询记录到 `.cursor/advisor/log.md`，并在回合结束时提醒进行完成前的咨询。
