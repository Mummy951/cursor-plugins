# benny 自动化意图

## 我想要自动化什么

我想要两个 Cursor 自动化在同一个 Slack issue 频道中协作运行。

### 自动化 1：分诊 issue 报告

- 触发器：当有人在我配置的主 Slack 频道中发布新的顶层报告时，我希望此自动化在该报告上启动，并保留其原始 thread 坐标。
- 行为：我希望它读取 thread 和附件，将报告分类为 bug 或性能问题、功能请求、问题或反馈，或重新路由，并在路由前追踪可能的归属层。
- 跟踪器：我希望在我配置的跟踪器中搜索重复项，更新有把握的重复项，并且仅在明确是全新 bug 时才创建工单。
- 工具：我希望具备 Slack thread 读取和回复权限、我配置的跟踪器集成，以及我可选的路由映射。
- 结果：我希望在主 thread 中恰好发布一条回复，包含简短的判定以及 `[benny:bug]`、`[benny:performance]` 或 `[benny:other]`。bug 或性能标记可以包含跟踪器 url。
- 边界：我不希望此自动化在主频道中发布根消息。

### 自动化 2：复现并修复已确认的 bug

- 触发器：我希望此自动化从同一个新的顶层报告启动，或从设置期间选择的另一个受支持的触发器启动，然后在原始 thread 中等待可信的分诊标记。
- 门禁：当有人明确负责修复时，我希望它停止。如果现有的 pull request 或已合并的 commit 可能修复该报告，我希望进行验证，而不是发起竞争性变更。
- 行为：我希望它使用我配置的控制适配器和功能映射，通过真实 ui 复现完全相同的症状两次，并捕获截图、视频和只读状态交叉检查。
- 修复：我希望它验证现有的 pull request，而不在其之上编写代码。在确认复现后，它可以尝试一次有边界的根本原因修复，在测试成本低时使用 tdd，对影响范围进行冒烟测试，并且仅在前后证据通过时才创建草稿 pull request。
- 工具：我希望具备 Slack thread 读取和回复权限、仓库和历史记录访问权限、草稿 pull request 创建权限、我配置的跟踪器，以及我的控制适配器。
- 结果：我希望在主 thread 或可选的运维 thread 中获得证据和已验证的结果，以及可选的草稿 pull request。更新应当简洁。
- 边界：我不希望此自动化在主频道中发布根消息。

### 共享规则

- 我希望主频道和根 thread 坐标在整个运行过程中保持不变。
- 我将工具和调试机器人视为证据，而非委托对象或修复责任人。
- 我允许子代理提供帮助，但它们不能向 Slack 发帖，也不能接收 Slack 凭据。
- 我希望将整个包提交到目标仓库的 `.cursor/automations/benny/`。其中的 `SKILL.md` 文件是直接的自动化指令，不是已注册的插件技能。
- 我希望仅通过目标仓库已提交的 `.cursor/settings.json` 启用 pstack，且仅用于共享依赖，例如 `why`、`tdd`、`unslop` 以及所需的原则技能。
- 我希望每个实际运行的自动化提示词直接读取其已提交的操作文件。我不希望使用插件缓存路径、复制的摘录或 slash-skill 发现机制。
- 我将用户自有的配置、功能映射、路由映射和密钥保存在 `.cursor/automations/benny/` 之外，以便包刷新时不会覆盖它们。
- 当频道坐标、跟踪器访问、控制适配器或功能映射缺失或不确定时，我希望两个自动化都采取失败即关闭（fail closed）策略。
- 我只想要草稿 pull request。不要合并或部署。

### 我的配置

- 主 Slack 频道：`<channel>`
- 可选运维频道：`<channel or none>`
- 仓库和默认分支：`<repo>`, `<branch>`
- 跟踪器：`<type, team, project, labels, intake status>`
- 路由映射：`<path or none>`
- 分诊身份：`<slack identity>`
- 控制技能：`<configured skill or adapter>`
- 功能映射：`<committed same-repo path outside the copied pack, or behavior to paraphrase>`
- 模型：`<triage, reproduce, code, media review>`
- 状态 emoji 字符串：`<seen, reproducing, reproduced, blocked, fixing, failed, pull request opened>`
- 预算：`<polling, verdict wait, follow-up, repro, rejection, fix>`
- 可选 bot token 能力：`<none, file download, or editable operations status>`

从 [`configuration.example.yaml`](./templates/configuration.example.yaml) 和 [`feature-map.example.md`](./skills/reproduce-and-fix-issues/references/feature-map.example.md) 开始。在此外部复制并填写它们，例如在 `.cursor/benny/` 下。将密钥值保存在密钥管理器或环境变量中。

## 给代理的说明

人类通过将 Cursor 指向此文件来进入设置。不要查找或调用已发现的 benny slash 技能。

1. 询问将由哪个仓库运行这些自动化。
2. 将包含此 `FOR_AGENTS.md` 的目录视为源包。
3. 将整个源包合并到 `<target-repository>/.cursor/automations/benny/`。
4. 保留每个仅存在于目标位置的文件。永远不要删除无关文件，或覆盖用户自有的配置、功能映射或路由映射。
5. 当源管理路径上已存在的目标文件存在差异时，检查 diff 并进行合并，不要丢弃本地编辑。如果归属不明确，请停止并在替换前询问。
6. 确认复制的 `FOR_AGENTS.md` 和 `skills/setup-benny/SKILL.md` 在目标仓库中存在。
7. 直接从目标仓库读取并遵循 `.cursor/automations/benny/skills/setup-benny/SKILL.md`。

我希望你将此条目合并到目标仓库的 `.cursor/settings.json` 中：

```json
{
	"plugins": {
		"pstack": { "enabled": true }
	}
}
```

保留每个无关的设置和插件。当现有文件使用 jsonc 时，保留注释和有效的 jsonc 语法。

我希望由扎根于目标仓库的全新代理进行验证。确认 pstack 的 `why`、`tdd`、`unslop` 以及 benny 使用的原则技能在项目范围内可解析。不要将当前会话加载的技能或用户级安装中的技能计入。

如果项目级插件不可用或任何共享依赖无法解析，请停止并说明失败原因。不要将 `.cursor/automations/benny/skills/` 添加到插件清单中，也不要期望其文件出现在 slash-skill 列表中。

告诉我在启用任一自动化之前，必须提交 `.cursor/settings.json`、`.cursor/automations/benny/` 以及任何引用的不含密钥的配置。在我明确要求之前，不要创建或更新自动化。

对于首次创建，使用内置的 `/automate` 分别为分诊和复现修复各创建一次。在启动第二个自动化之前，先完成第一个自动化的草稿审查、批准、就绪检查和 Automations 编辑器交接。

将此意图和最终配置转述到每个草稿中。分诊提示词必须读取并遵循 `.cursor/automations/benny/skills/triage-issue-reports/SKILL.md`。复现提示词必须读取并遵循 `.cursor/automations/benny/skills/reproduce-and-fix-issues/SKILL.md`。仅在 `/automate` 确认这些路径已提交到将运行自动化的仓库之后，才使用这些仓库相对路径。

对于现有自动化，不要使用 `/automate` 来检查或更新它们。验证配置，然后使用复制的设置文件中的简明字段清单，以便我在其编辑器中直接编辑每个自动化。不要创建重复项。
