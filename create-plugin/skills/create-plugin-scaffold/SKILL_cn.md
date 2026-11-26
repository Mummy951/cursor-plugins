---
name: create-plugin-scaffold
description: 创建一个新的 Cursor 插件脚手架，包含有效的 manifest、组件目录和 marketplace 接线。在开始新插件或将插件添加到多插件仓库时使用。
---

# 创建插件脚手架

## 触发条件

你需要从零开始创建一个新的 Cursor 插件，并使其可用于本地使用或提交到 marketplace。

## 必需输入

- 插件名称（小写 kebab-case）
- 插件用途和目标用户
- 要包含的组件集（`rules`、`skills`、`agents`、`commands`、`hooks`、`mcpServers`）
- 仓库风格（`single-plugin` 或 `multi-plugin marketplace`）

## 输出位置

默认情况下，在用户的本地插件目录内创建插件：

```
~/.cursor/plugins/local/<plugin-name>/
```

此路径使插件无需任何安装步骤即可立即被 Cursor 使用。如果用户明确要求在别处创建插件（例如在现有仓库内或特定目录中），则遵循该选择。

## 工作流

1. 验证插件名称格式：小写 kebab-case，以字母数字字符开头和结尾。
2. 确定目标目录：
   - 默认：`~/.cursor/plugins/local/<plugin-name>/`
   - 覆盖：如果用户指定了路径，则使用用户指定的路径。
   - 如果目录不存在，则创建该目录（及其父目录）。
3. 在目标目录内创建基础文件：
   - `.cursor-plugin/plugin.json`
   - `README.md`
   - `LICENSE`
   - 可选的 `CHANGELOG.md`
4. 填充 `plugin.json`：
   - 必需：`name`
   - 推荐：`version`、`description`、`author`、`license`、`keywords`
   - 仅当需要非默认发现时，才添加显式的组件路径。
5. 创建带有有效 frontmatter 的组件文件：
   - 规则：`.mdc`，包含 `description`、`alwaysApply`、可选的 `globs`
   - 技能：`skills/<skill-name>/SKILL.md`，包含 `name`、`description`
   - 智能体：`agents/*.md`，包含 `name`、`description`
   - 命令：`commands/*.(md|txt)`，包含 `name`、`description`
6. 如果仓库使用 `.cursor-plugin/marketplace.json`，添加插件条目：
   - `name`
   - `source`
   - 可选的元数据（`description`、`keywords`、`category`、`tags`）
7. 确保所有 manifest 路径都是相对的、有效的，并且不使用绝对路径或父目录遍历。

## 防护栏

- 保持插件专注于单一用例。
- 优先使用简洁、可操作的技能和规则文本，而非冗长的散文。
- 不要引用不存在的文件。
- 除非需要自定义路径，否则使用文件夹发现默认值。
- 始终保存到 `~/.cursor/plugins/local/<plugin-name>/`，除非用户提供了其他路径。

## 输出

- 为插件创建的文件树（包含输出目录的完整路径）
- 最终的 `plugin.json`
- Marketplace 条目（如适用）
- 关于必需字段和组件元数据的简短验证报告
- 确认插件已保存在 `~/.cursor/plugins/local/` 下并可使用
