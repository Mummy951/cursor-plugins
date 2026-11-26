# benny

benny 为你提供两个用于处理 Slack 问题报告的 cursor 自动化。一个对每份报告进行分流（triage）。另一个复现已确认的 bug，并可能准备一份小型的修复草稿。

此目录中的文件是处于休眠状态的配置和自动化源文件。它们不会作为 slash skills 出现。

## 设置

1. 让 cursor 指向 [`FOR_AGENTS.md`](./FOR_AGENTS.md) 并指定目标仓库名称。
2. 让配置流程将整个目录合并到目标的 `.cursor/automations/benny/` 中。它必须保留目标端独有的文件，并审查冲突，而不是覆盖本地修改。
3. 让配置流程在目标仓库的 `.cursor/settings.json` 中启用 pstack，以共享依赖项：

```json
{
	"plugins": {
		"pstack": { "enabled": true }
	}
}
```

4. 将用户自有的配置保留在复制的包之外，例如放在 `.cursor/benny/` 中。调整 [`configuration.example.yaml`](./templates/configuration.example.yaml) 和 [`feature-map.example.md`](./skills/reproduce-and-fix-issues/references/feature-map.example.md)。
5. 在启用任一自动化之前，先提交 `.cursor/settings.json`、`.cursor/automations/benny/` 以及任何不含密钥的配置。
6. 在各自的编辑器中审查每个新的自动化草稿或更新现有自动化。然后发送一份无害的测试报告，并验证每个源频道的帖子都留在原始线程中。
