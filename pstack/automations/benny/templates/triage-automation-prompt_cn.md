# 问题分流自动化提示词

> 供复制设置流程使用的源材料。在 `automate` 确认复制的包已提交到自动化将要运行的仓库后，将此意图改写成内置的 `automate` 草稿。

本次运行请阅读并遵循 `.cursor/automations/benny/skills/triage-issue-reports/SKILL.md`。

配置来源。仅当此仓库相对路径已提交到同一目标仓库时，才包含该路径。否则请改写已配置的值。切勿使用插件源路径或缓存路径：

```text
{{BENNY_CONFIG_PATH}}
```

触发器：

```json
{
	"source_channel_id": "{{SLACK_CHANNEL_ID}}",
	"message_ts": "{{SLACK_MESSAGE_TS}}",
	"thread_ts": "{{SLACK_THREAD_TS_OR_EMPTY}}"
}
```

创建意图应将其描述为在已配置的源 Slack 频道中的一条新的顶层报告。

将源频道和根线程时间戳视为不可更改。如果其中任一缺失或与配置不匹配，请停止，不要发布消息，也不要写入问题跟踪器。

已提交的操作文件负责分类、附件审查、原因追溯、路由、去重、跟踪器写入以及最终裁定。不要发布进度消息。切勿在源频道中发布根消息。

协调者是唯一的 Slack 发布者。任何被委派的工作者都必须为只读、仅返回结果，并被明确禁止一切 Slack 写入操作。

以恰好一个已配置的标记结束这唯一的裁定：

```text
[benny:bug]
[benny:performance]
[benny:other]
```

bug 或 performance 标记可以添加 `tracker=<URL>`。
