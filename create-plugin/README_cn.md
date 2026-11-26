# 创建 plugin

用于创建可直接上架 marketplace 的 Cursor plugin 的元工作流。

## 安装

```bash
/add-plugin create-plugin
```

## 组件

### 技能（Skills）

| 技能 | 描述 |
|:------|:------------|
| `create-plugin-scaffold` | 搭建一个新的 plugin 目录，包含 manifest、组件和仓库配置 |
| `review-plugin-submission` | 根据 marketplace 的期望运行提交前质量检查 |

### 规则（Rules）

| 规则 | 描述 |
|:-----|:------------|
| `plugin-quality-gates` | 保持 plugin manifest、组件元数据和路径有效且一致 |

### 代理（Agents）

| 代理 | 描述 |
|:------|:------------|
| `plugin-architect` | 根据具体用例设计 plugin 结构和组件组合 |

### 命令（Commands）

| 命令 | 描述 |
|:--------|:------------|
| `create-plugin` | 使用正确的文件和元数据构建新的 plugin 脚手架 |

## 典型流程

1. 使用 `/create-plugin`，并提供 plugin 名称、用途和目标组件类型。
2. 生成或更新 `plugin.json`，然后根据需要添加 rules/skills/agents/commands。
3. 在发布或提交到 marketplace 之前运行 `review-plugin-submission`。

## 许可证

MIT
