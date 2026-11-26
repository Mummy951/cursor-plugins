# 要求

dyl-stack 建立在其他插件之上。在开始任何工作之前，请先检查你的 skill 所需的插件。如果缺少某个插件，请停止操作，指出该插件的名称，并告诉用户运行 `/add-plugin <name>`。不要临时拼凑缺失的部分。

| 插件 | 安装时机 | 访问其文件 |
|---|---|---|
| `pstack` | `setup-pstack` 在你的 skill 列表中 | 大多数 pstack skill 只能通过斜杠命令调用，且不会显示在列表中。它们位于 `setup-pstack` 旁边：`<setup-pstack dir>/../poteto-mode/SKILL.md`、`../poteto-mode/playbooks/<name>.md`、`../principle-<name>/SKILL.md`、`../bro/SKILL.md`、`../unslop/SKILL.md`。 |
| `cursor-team-kit` | `control-ui`、`verify-this` 和 `deslop` 在你的 skill 列表中 | 其列出的路径。 |
| `thermos` | `thermo-nuclear-review-subagent` 和 `thermo-nuclear-code-quality-review-subagent` 是可用的 Task subagent 类型 | 通过 subagent 类型启动它们。 |
| `figma` | `figma-design-to-code` 在你的 skill 列表中，且 Figma MCP 工具调用成功 | 其列出的路径。 |
