---
name: review-plugin-submission
description: 审查 Cursor 插件是否已具备 marketplace 上架条件。在发布前验证清单（manifest）、组件元数据、发现路径和提交质量时使用。
---

# 审查插件提交

## 触发条件

插件已实现，在提交或发布前需要进行最终质量检查。

## 工作流程

1. 验证清单（manifest）有效性：
   - `.cursor-plugin/plugin.json` 存在
   - `name` 为有效的小写 kebab-case 格式
   - 元数据字段一致且合理（`description`、`version`、`author`、`license`）
2. 验证组件可发现性：
   - Skills 位于 `skills/*/SKILL.md`
   - Rules 位于 `rules/`，为 `.mdc` 或 markdown 变体格式
   - Agents 位于 `agents/` 的 markdown 文件中
   - Commands 位于 `commands/` 的 markdown 或文本文件中
   - Hooks 位于 `hooks/hooks.json`
   - MCP 配置位于 `mcp.json` 中（或通过 `mcpServers` 覆盖）
3. 验证组件元数据：
   - Skills 包含 `name` 和 `description` frontmatter
   - Rules 包含有效的 frontmatter 和清晰的指引
   - Agents 和 commands 包含 `name` 和 `description`
4. 验证仓库集成：
   - 对于 marketplace 仓库，插件条目存在于 `.cursor-plugin/marketplace.json` 中
   - `source` 解析到插件目录，且名称唯一
5. 验证文档质量：
   - `README.md` 说明了用途、安装方式和组件覆盖范围
   - 可选的 logo 路径有效且托管在仓库中

## 检查清单

- Manifest 存在且能解析为有效 JSON
- 所有声明的路径均存在且为相对路径
- 无损坏的文件引用
- skills/rules/agents/commands 均无缺失的 frontmatter
- 插件范围清晰且聚焦
- Marketplace 注册已完成（若为多插件仓库）

## 输出

- 按章节划分的通过/失败报告
- 按优先级排序的修复列表
- 最终提交建议
