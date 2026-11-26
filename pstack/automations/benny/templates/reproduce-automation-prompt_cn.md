# 复现自动化提示词

> 用于复制的设置工作流的源材料。在 `automate` 确认复制的包已提交到将要运行自动化的仓库后，将此意图转述为内置的 `automate` 草稿。

本次运行请阅读并遵循 `.cursor/automations/benny/skills/reproduce-and-fix-issues/SKILL.md`。

配置来源。仅当此相对于仓库的路径已提交到同一目标仓库时，才包含该路径。否则请转述配置的值。切勿使用插件源路径或缓存路径：

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

创建意图应将其描述为在配置的源 Slack 频道中新建的一条顶层报告。它应包含配置的仓库、默认分支、问题跟踪器、控制适配器、功能地图以及草稿拉取请求能力。

将源频道和根线程时间戳视为不可变。如果其中任一缺失或与配置不匹配，请停止且不要发布。

在此确切线程中等待来自配置的分诊身份的已配置分诊标记。仅在 `[benny:bug]` 或 `[benny:performance]` 时才继续。

在尝试复现之前，要求使用配置的控制适配器技能。通过真实 UI 精确复现该区分性症状两次。核验现有的拉取请求或提交，不要在其之上编写。仅在确认复现且操作文件的修复门禁通过后，才尝试有边界的修复。

协调者是唯一的 Slack 发布者。每个子提示词都必须禁止 `SendSlackMessage`、`PostToSlack`、`chat.postMessage` 以及所有其他 Slack 写入操作。子任务只返回结果。

切勿在源频道中发布根消息。
