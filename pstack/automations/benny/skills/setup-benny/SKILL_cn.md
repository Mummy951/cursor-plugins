---
name: setup-benny
description: 配置 Benny 并准备其分流与复现自动化。在安装 Benny 或修改其 Slack、跟踪器、仓库、路由、控制、模型或预算设置时使用。
disable-model-invocation: true
---

# 设置 Benny

Benny 作为休眠的自动化包（automation pack）随 pstack 一起提供。插件清单仅暴露 pstack 的常规 skill 根目录；本文件与两个操作文件都不是 slash skill。

用户通过将 Cursor 指向该包的 `FOR_AGENTS.md` 进入设置。引导流程会将整个包复制到目标仓库，然后直接从 `.cursor/automations/benny/skills/setup-benny/SKILL.md` 读取本文件。

Benny 需要外部配置以及两个运行中的 Cursor 自动化。

在用户明确要求之前，不要创建或更新自动化。切勿将密钥值放入插件文件、提示词或已提交的配置中。

## 1. 复制包并启用共享 pstack skill

在询问 Benny 配置之前以及调用内置 `/automate` skill 之前，先执行此步骤。

询问将由哪个仓库运行这些自动化。源包即包含 `FOR_AGENTS.md` 的目录。目标位置为 `<target-repository>/.cursor/automations/benny/`。

将整个源包合并到目标位置：

1. 若目标位置不存在，则创建它。
2. 将每个源文件复制到相同的相对路径。
3. 保留仅存在于目标中的文件。安装或刷新期间切勿删除无关文件。
4. 将用户拥有的配置、功能映射和路由映射保留在目标位置之外。切勿覆盖它们。
5. 当现有受源文件管理的文件存在差异时，检查 diff 并在不丢弃本地修改的情况下合并。如果归属不明确，请先停止并询问，然后再替换。
6. 验证目标位置包含 `FOR_AGENTS.md`、本设置文件、两个操作文件、它们的引用以及模板。

如果本文件已经是从目标位置读取的，则将复制视为已完成，并在继续之前运行相同的验证。

将 pstack 添加到目标仓库的 `.cursor/settings.json`。如果文件或 `.cursor` 目录不存在，请创建它。

将此条目合并到现有的 JSON 或 JSONC 中：

```json
{
	"plugins": {
		"pstack": { "enabled": true }
	}
}
```

保留每个无关的顶层设置以及每个其他插件条目。如果 `plugins.pstack` 已存在，仅更改其 `enabled` 值。当文件使用 JSONC 时，保留注释和合法的 JSONC 语法。编辑后校验文件。

重新加载目标项目，或从该目录启动一个全新的 agent。验证以下共享 pstack skill 能从项目作用域解析：

- `why`
- `tdd`
- `unslop`
- `principle-separate-before-serializing-shared-state`
- `principle-minimize-reader-load`
- `principle-guard-the-context-window`
- `principle-sequence-verifiable-units`
- `principle-fix-root-causes`
- `principle-prove-it-works`

不要将从当前会话或用户作用域插件加载的 skill 计入其中。检查必须证明目标仓库中的全新 agent 能通过项目设置接收到 pstack。

如果项目作用域的插件安装不可用，或任何共享依赖无法解析，请停止并说明失败原因。

Benny 文件直接从 `.cursor/automations/benny/` 读取。不要将该目录添加到插件清单，也不要期望其 `SKILL.md` 文件出现在 slash-skill 列表中。

告知用户，在启用任一自动化之前，必须提交 `.cursor/settings.json`、`.cursor/automations/benny/` 以及任何被引用的无密钥配置。除非用户要求，否则不要提交。

一旦此检查通过，运行中的自动化提示词即可按其稳定的仓库相对路径读取已提交的操作文件。它们不得嵌入插件缓存路径或复制文件内容。

## 2. 调整配置

打开这些复制的示例：

- `../../templates/configuration.example.yaml`
- `../reproduce-and-fix-issues/references/feature-map.example.md`

在 `.cursor/automations/benny/` 之外创建用户拥有的副本。这些是配置文件，而非包文件。示例位置：

- 项目配置，例如 `.cursor/benny/configuration.yaml`
- 项目功能映射，例如 `.cursor/benny/feature-map.md`
- 项目路由映射，例如 `.cursor/benny/routing.md`
- 用户配置，例如 `~/.config/benny/configuration.yaml`
- 用户功能映射，例如 `~/.config/benny/feature-map.md`

为自动化可能复现的每个面向用户的功能填写一个功能映射（feature-map）小节。保持以用户视角描述。不要在映射中固化实现细节或当前代码路径。

不要编辑复制的示例。包刷新可能会在冲突审查后更新受源文件管理的文件，但绝不能触碰用户拥有的副本。

当全新的自动化检出需要读取文件时，优先使用目标仓库中已提交的无密钥文件。否则，将所需值转述到运行中的提示词里。只有在内置 `/automate` skill 确认某仓库文件已在自动化运行所在的仓库中提交后，才引用该文件。

对已提交的包和配置文件使用稳定的仓库相对路径。切勿在运行中的自动化里引用插件源目录或插件缓存路径。

## 3. 填写必填选项

询问或确认：

- 源 Slack 频道 ID
- 可选的操作或状态频道 ID
- 仓库 URL 与默认分支
- 分流身份或 Slack 用户 ID
- 问题跟踪器类型、团队、项目、标签以及接收状态
- 跟踪器适配器 skill 或 MCP 操作
- 可选的路由映射路径
- 必需的控制 skill 名称
- 必需的面向用户功能映射路径
- 状态 emoji 字符串
- Pull request URL 格式
- 轮询与工作量预算
- 用于分流、复现、代码工作和媒体审查的 model slug

仅使用用户 Cursor 模型选择器或受支持模型列表中显示为可用的 model slug。不要猜测 slug，也不要沿用私有默认值。

源频道、分流身份、仓库、跟踪器适配器、控制 skill 和功能映射必须明确。如果任何必填值仍然模糊，则设置失败。

在保存最终自动化名称、描述和提示词垫片之前，使用 pstack 的 `unslop` skill 处理它们。

## 4. 检查集成能力

分流自动化需要：

- 对配置的源 Slack 频道及其话题（thread）的读取权限
- 在该频道中的话题回复权限
- 当报告包含媒体时，附件元数据和文件下载权限
- 通过配置的问题跟踪器适配器进行搜索、读取、创建和更新的权限

复现自动化需要：

- 对源话题的读取权限
- 在源频道中的话题回复权限
- 在配置的操作频道中可选的发帖和编辑权限
- 仓库读取和历史记录权限
- 能够创建草稿 pull request 的 pull request 操作
- 配置的控制适配器 skill

优先使用配置好的 Cursor Slack 操作进行读取和发帖。可选的 `BENNY_SLACK_BOT_TOKEN` 可填补狭窄的缺口，例如编辑一条操作状态消息或下载附件。将值存储在密钥管理器或环境变量中，而不是 YAML 中。

不要使用未记录的集成端点。

## 5. 准备路由映射

如果用户需要重新路由或提醒负责人（owner ping）：

1. 将 `../triage-issue-reports/references/routing.example.md` 复制到 `.cursor/automations/benny/` 之外。
2. 用公开或组织内部的值替换每个占位符。
3. 默认关闭 owner ping。
4. 仅允许对已配置的功能负责人或已确认的可能引入回归的作者发出 ping。

如果未配置路由映射，分流可以对报告进行分类，但不得猜测目标位置或负责人。

## 6. 验证控制适配器

阅读 `../reproduce-and-fix-issues/references/control-adapter.md` 以及用户填好的功能映射。

确认所命名的 skill 能够：

- 启动目标应用
- 通过真实 UI 导航到每个已映射的功能
- 通过声明的适配器操作执行已映射的状态
- 检查状态而不强制结果
- 截取屏幕截图
- 开始和停止录制
- 清理其进程和临时数据

如果缺少任何能力，请保持复现自动化处于禁用状态。它必须失败关闭（fail closed），而不是声称完成了并未执行的复现。

## 7. 准备运行中的自动化

询问这是首次创建还是配置现有自动化。

阅读复制包中的 `../../FOR_AGENTS.md`，作为两条路径下主要的用户意图来源。用它来理解两个触发器、工具、指令、结果和共享规则。

### 首次创建

一次创建一个自动化。

对于每个自动化：

1. 阅读匹配的复制提示词模板，作为辅助的内部素材。
2. 将 `FOR_AGENTS.md`、完成的 Benny 配置以及模板意图转化为完整的自然语言请求。
3. 告知运行中的提示词去读取并遵循其在 `.cursor/automations/benny/` 下确切的已提交操作文件。
4. 使用稳定的仓库相对路径，而非插件源路径或缓存路径。不要将操作文件内容复制到运行中的提示词里。
5. 阅读并遵循内置的 `automate` skill。
6. 让 `automate` 发现 Slack 频道、仓库和已连接的集成。
7. 让 `automate` 确认复制的包和任何被引用的配置文件都已提交在自动化将要运行的同一仓库中。
8. 让 `automate` 展示其草稿表格、获取批准、询问就绪情况，并打开 Automations 编辑器。
9. 在开始下一个自动化之前，完成当前自动化的编辑器交接。

向 `automate` 提供以下完整的分流意图，并根据配置填写：

- 名称 `benny-triage`。
- 每次运行时读取并遵循 `.cursor/automations/benny/skills/triage-issue-reports/SKILL.md`。
- 在配置的源 Slack 频道中每条新的顶层报告时触发。
- 读取触发话题，并且仅在该话题内回复。
- 使用配置的问题跟踪器集成。
- 分类、检查证据、追溯原因、去重，并仅创建清晰的新 bug。
- 以一条仅限话题的裁决结束，附带配置的 `[benny:bug]`、`[benny:performance]` 或 `[benny:other]` 标记以及可选的跟踪器 URL。
- 切勿在源频道发布根消息。

分流编辑器交接完成后，向 `automate` 提供以下完整的复现与修复意图：

- 名称 `benny-reproduce`。
- 每次运行时读取并遵循 `.cursor/automations/benny/skills/reproduce-and-fix-issues/SKILL.md`。
- 在配置的源 Slack 频道中对同样的新顶层报告触发。
- 使用配置的仓库和默认分支。
- 读取源话题，并且仅在该话题内回复。
- 包含 pull request 创建以及配置的跟踪器、控制适配器、功能映射要求。转述已映射的用户路径和状态，除非 `automate` 确认同一仓库中存在合格的已提交文件。
- 在行动前等待可信的分流标记。
- 通过映射的真实 UI 精确复现症状两次并捕获证据。
- 验证现有修复，不要在其之上重写。
- 仅在确认复现后尝试可选的有界修复，然后在证据和检查通过时创建草稿 pull request。
- 切勿在源频道发布根消息。

不要重复 `automate` 的 Slack、仓库、集成、完整性、认证、草稿审查、批准、就绪或编辑器交接工作。

### 现有自动化

内置的 `automate` skill 仅用于创建。不要用其搜索、检查或更新现有自动化。

完成配置、路由、控制适配器和功能映射验证。然后给用户这份简洁的编辑器清单。

对于现有的分流自动化，更新：

- 名称和描述
- 直接指令，读取 `.cursor/automations/benny/skills/triage-issue-reports/SKILL.md`
- 新的顶层 Slack 报告触发器和源频道
- Slack 话题读取和回复能力
- 问题跟踪器集成
- 转述后的分流指令、仅限话题规则以及 Benny 裁决标记

对于现有的复现自动化，更新：

- 名称和描述
- 直接指令，读取 `.cursor/automations/benny/skills/reproduce-and-fix-issues/SKILL.md`
- 匹配的 Slack 触发器和源频道
- 仓库和默认分支
- Slack 话题读取和回复能力
- pull request 操作
- 跟踪器、控制适配器和功能映射要求
- 转述后的标记等待、证据、验证和有界修复指令

请用户在其 Automations 编辑器中直接更新每个现有自动化。不要创建替代或重复的自动化。

### 创建边界

切勿调用直接的自动化后端服务或后端自动化工具。切勿使用携带草稿字段的浏览器 URL。切勿构建或打开 Cursor 协议深层链接。对于新自动化，唯一的完成路径是内置 `automate` skill 经过审查的 Automations 编辑器交接。

在编辑器保存后线程安全测试通过之前，不要启用任一自动化。

## 8. 测试线程安全

使用测试频道或无害的测试报告。

测试前，确认目标仓库的 `.cursor/settings.json`、`.cursor/automations/benny/` 以及每个被引用的无密钥配置文件都已提交在自动化检出所用的分支上。确认两条运行中的提示词都指向其确切的已提交操作文件。如果任何检查失败，请停止。告知用户目前尚不能启用该自动化。

验证：

1. 分流存储根 `thread_ts`，并且恰好发布一条裁决作为回复。
2. 裁决包含恰好一个配置的标记。
3. 复现仅接受来自配置的分流身份的标记。
4. 复现保持相同的不可变源坐标。
5. 没有源频道根消息出现。
6. 被委托的 worker 不能使用任何 Slack 写操作。
7. 缺少坐标、父话题被删除或预检失败时，不产生任何发帖，也不创建跟踪器问题。

仅在全部七项检查通过后，才启用正常流量。
