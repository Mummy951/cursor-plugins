# Agent 兼容性

用于检查仓库在 agent 工作流下表现如何的 Cursor 插件。它将已发布的 `agent-compatibility` CLI 与针对启动、验证和文档可靠性的专项审查相结合。

默认情况下，完整检查会返回一个总体分数和一份简短的高杠杆修复建议列表。如果用户想要完整的细分结果，agent 可以展示各组件分数及其背后的推理依据。

## 包含内容

- `check-agent-compatibility`：完整兼容性检查
- `compatibility-scan-review`：基于 CLI 的原始扫描
- `startup-review`：冷启动与引导审查
- `validation-review`：小改动验证审查
- `docs-reliability-review`：文档可靠性审查

## 评分模型

- `Agent Compatibility Score`：展示给用户的最终混合分数
- `Deterministic Compatibility Score`：来自已发布 CLI 的原始分数
- `Startup Compatibility Score`：启动仓库需要多少猜测工作
- `Validation Loop Score`：验证一个小改动的实际可行性
- `Docs Reliability Score`：文档与实际配置路径的吻合程度

最终分数将确定性扫描与工作流检查相结合：

```text
Agent Compatibility Score = round((deterministic * 0.7) + (workflow * 0.3))
```

CLI 还会针对已提交的 agent 工具报告一个加速器层。这些额外上下文会为建议提供依据，但不会抬高确定性兼容性分数本身。

## 使用方法

当你需要完整检查时，使用 `check-agent-compatibility`。该 skill 会分派给上面四个审查 agent，然后返回一份精简结果：

```md
## Agent Compatibility Score: 72/100

Top fixes
- First issue
- Second issue
```

如果你想要各组件分数或权重，可以请求细分结果。

## CLI 说明

该插件不捆绑扫描器。它会在需要时运行已发布的 npm 包。

默认扫描（精简的终端仪表盘）：

```bash
npx -y agent-compatibility@latest .
```

JSON 输出：

```bash
npx -y agent-compatibility@latest --json .
```

Markdown 输出：

```bash
npx -y agent-compatibility@latest --md .
```

纯文本输出：

```bash
npx -y agent-compatibility@latest --text .
```

用于覆盖忽略路径或权重覆盖项的配置：

```bash
npx -y agent-compatibility@latest . --config ./agent-compatibility.config.json
```

该扫描器是启发式的。它对仓库信号进行评分并揭示可能的摩擦点，但它并非对代码库的完整质量评判。

## 本地安装

如果你想直接使用该插件，请将此目录符号链接到：

```bash
~/.cursor/plugins/local/agent-compatibility
```
