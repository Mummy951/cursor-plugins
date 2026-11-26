---
name: advisor
description: >-
  Advisor 模式。在关键检查点咨询一个更强大（或不同）的模型：
  重大决策前、遇到错误卡住时，以及宣布任务完成前。
  当用户输入 /advisor、要求开启或关闭 advisor、选择 advisor 模型，
  或明确要求从更强大或不同的模型那里获得第二意见时使用。
  绝不要自行启用它。
icon: lightbulb
color: purple
---

# Advisor

你是主模型。Advisor 模式增加了一个更强大的第二模型，你会在几个关键点咨询它。Advisor 会收到你的简报（如果可用，还有完整对话记录），深入思考，并返回带有指导的裁决。它不编辑文件。工作仍由你完成，最终决定也由你做出。

状态保存在项目根目录的 `.cursor/advisor/state.json` 中。当该文件存在且包含 `"enabled": true` 时，该项目就启用了 Advisor 模式。

## 命令

`/advisor` 后面的文本选择要执行的操作。

| 输入 | 操作 |
| --- | --- |
| `/advisor` | 使用默认 Advisor 为本次对话启用：最新的 Grok，采用其最高推理强度。如果状态文件已存在（无论来自本次还是其他对话），将其重新绑定到此处，保留其 `model` 和 `nudge` 设置。 |
| `/advisor <model>` | 同上，但使用指定的模型。任何可供 subagent 使用的模型都可以：`/advisor cursor-grok-4.6-xhigh-fast`、`/advisor composer-2.5`。 |
| `/advisor off` | 禁用：删除 `.cursor/advisor/`。 |
| `/advisor status` | 报告模型、咨询次数，以及回合结束提醒是否开启。不做任何更改。 |
| `/advisor ask <question>` | 立即就当前工作进行咨询，不受检查点限制。 |
| `/advisor nudge on` / `off` | 切换由插件的 stop hook 发布的回合结束提醒（默认开启）。 |

如果消息中还包含一个任务（`/advisor, then refactor the cache layer`），先启用，然后在 Advisor 模式下执行该任务。

### 选择模型

默认是最新 Grok 模型的最高推理强度，当前为 `cursor-grok-4.6-xhigh`。如果你可用的 subagent 模型列表中出现了更新的 Grok 或更高的推理强度层级，优先使用它并说明。

对于 `/advisor <model>`，根据你可用的 subagent 模型 slug 来解析请求：

- 完全匹配的 slug：直接使用。
- 系列名或版本名（`grok fast`、`composer`）：该系列最新模型的最高推理层级。
- 无匹配：说明情况，列出两三个相近选项，并保留当前模型。

如果 Task 工具拒绝了某个 slug，从其错误消息中读取有效 slug，选择最接近的一个（同一系列、最高推理层级），保存到 `state.json`，并用一行告知用户。不要因为 slug 问题而阻塞咨询。

## 启用

1. 按上述方式解析模型。
2. 使用文件写入工具（而非 shell 重定向）写入 `.cursor/advisor/state.json`，必要时创建目录。如果状态文件已存在，沿用其 `model`（除非本命令指定了模型）和 `nudge`，并将其他所有字段重置为下面的值。你无法看出已有文件属于哪次对话，因此始终重写它：这会将模式重新绑定到本次对话，hooks 会重新填充 `conversation_id` 和 `transcript_path`，下一次咨询会启动一个全新的 Advisor，而非恢复另一聊天的 Advisor。如果 `.cursor/advisor/pending` 和 `.cursor/advisor/last-response.txt` 存在，也要删除，以免其他对话留下的标记在此触发回合结束提醒。保留 `log.md`。

   ```json
   {
     "enabled": true,
     "model": "cursor-grok-4.6-xhigh",
     "nudge": true,
     "advisor_agent_id": null,
     "conversation_id": null,
     "transcript_path": null,
     "consults": 0,
     "last_consult_at": null,
     "enabled_at": "<current UTC time, ISO 8601>"
   }
   ```

   插件的 hooks 会填充 `conversation_id`、`transcript_path`、`consults` 和 `last_consult_at`。不要改动它们。
3. 用一行确认：`Advisor on: <slug>. I'll consult it before major decisions, when I'm stuck, and before I call the task done.` 然后在同一条消息中继续处理任何任务。

切勿暂存或提交 `.cursor/advisor/`。

## 检查点

只在这些点咨询，别无其他。每次咨询都是一次强模型调用；其价值来自有选择地使用。

1. **重大决策前。** 在架构或方案之间选择；难以逆转或影响范围广的变更（schema 或数据迁移、删除或重写模块、公共 API 或配置格式变更、依赖替换、认证、支付或其他安全敏感代码）；或者一个含糊到会实质改变工作的请求。等你有了具体方案和备选方案后再咨询，而不是在理解问题之前。一次咨询覆盖整个方案；不要按文件逐一咨询。
2. **卡住时。** 经过两次真正的修复尝试后仍出现相同错误或失败的测试；无法从代码中解释的行为；或者你正准备采用权宜之计：重试循环、`sleep`、宽泛的 `try/except`、跳过测试，或禁用某项检查以绕过问题。
3. **宣布完成前。** 任何改变了逻辑或涉及多个文件的任务：在写最终总结前咨询一次，并说明你验证了什么以及如何验证。对于琐碎的改动（拼写错误、注释、单行配置更改）可跳过，并用一行说明。
4. **应请求。** `/advisor ask ...`，或用户询问 Advisor 的看法。

不要为常规步骤、你可以自己验证的事情（运行测试、阅读代码）咨询，也不要在同一检查点咨询超过一次。如果一项任务中你要咨询超过大约四次，该任务可能应该拆分，或者你应该询问用户。

## 如何咨询

1. 读取 `.cursor/advisor/state.json`，判断你处于哪种情况：
   - **此处已开启模式**：文件存在且 `enabled: true`，并且你在本次对话中执行了启用步骤。检查点咨询和 `/advisor ask` 都完整遵循以下步骤，包括 `resume` 和状态写入。
   - **此处未开启**：文件缺失、`enabled` 为 false，或该文件不是你写的。不是你写的文件属于其他聊天；只有 `/advisor` 会重新绑定它，因此不要改动它，也绝不要 `resume` 一个不是你自己保存的 `advisor_agent_id`。不要在检查点咨询。如果用户在本条消息中明确要求咨询（`/advisor ask ...`，或请求第二意见），则作为一次性咨询执行：如果文件中存在 `model` 就用它全新启动一个 Advisor，否则用默认模型；不 `resume`、不写状态、也不使用 `transcript_path`（其他聊天的对话记录不属于你来分享；写 "not available"）。并提及 `/advisor` 会为本次对话开启该模式。
2. 根据 `references/briefing-template.md` 构建简报。给 Advisor 提供它反驳你所需的一切：
   - 用户的原始请求，逐字照录，以及他们后来补充的约束或更正。
   - 迄今为止发生的事情，按时间顺序：你调查了什么、决定了什么、更改了什么、尝试了什么，以及排除了什么。
   - 相关的工具结果，逐字照录：错误消息、堆栈跟踪、测试输出、diff。删去无关噪声并用 `[...]` 标记删减处，但绝不要转述证据。
   - 当前状态：`git status --short`、`git diff --stat`、你触碰过的文件、任何未完成的事项。
   - 你的具体问题、你看到的选项，以及你目前的倾向及理由。
   - 当 `state.json` 中设置了 `transcript_path` 且本次对话已开启模式时，提供它，以便 Advisor 自行阅读完整对话。
   - 不含机密。遮盖 token、密钥和 `.env` 值。
3. 在前台启动 Advisor 并等待它：
   - `subagent_type: "advisor-subagent"`
   - `model: <state.model>`
   - `description: "Advisor: <checkpoint>"`，例如 `Advisor: pre-completion review`
   - `run_in_background: false`
   - 当 `state.advisor_agent_id` 已设置时，将其作为 `resume` 传入并省略 `model`；Advisor 会保留其模型和之前咨询的上下文，因此简报可以是增量式的：自上次以来的变化加上新问题。如果 resume 失败，则全新启动。
   - 将返回的 agent id 保存到 `state.json` 的 `advisor_agent_id`。当模型更改时清除它。
4. 根据裁决采取行动：
   - `proceed`：继续。
   - `proceed with changes`：执行建议的更改，除非它们与用户指令或你已验证的事实冲突。说明你跳过了哪些以及原因。
   - `stop`：不要继续执行该方案。重新思考；如果是产品或范围问题，则将分歧提交给用户。
   - 如果 Advisor 索要它所需的东西，通过 `resume` 提供一次。不要来回拉锯。
   - 你要对结果负责。Advisor 是一个有力的第二意见，而非权威。如果它对代码库判断有误，向其展示一次证据，或否决它并告知用户原因。
5. 用一两行向用户报告每次咨询：`Advisor (<model>): <verdict>. <One-line summary>. <What you did about it.>` 除非用户要求，否则不要把 Advisor 的完整回复放在对话中；hooks 也会将其追加到 `.cursor/advisor/log.md`。

## 回合结束提醒

当自上次咨询以来文件发生了更改，而回合结束时却没有咨询，插件的 `stop` hook 会发布一条以 `[Advisor]` 开头的后续提醒。将其视为"宣布完成前"检查点：执行完成前咨询，或用一行回答该更改是琐碎的，或你正在等待用户，然后停止。每批编辑最多触发一次。`/advisor nudge off` 可禁用它。

## 禁用

`/advisor off`：删除 `.cursor/advisor/` 并用一行确认。除非用户重新启用该模式或明确要求一次性咨询，否则本次对话中不要再次咨询。

## 防护栏

- 绝不要在未经要求时启用 Advisor 模式。"就此听取第二意见"是一次性咨询，而非模式更改。
- Advisor 是只读的。绝不要要求它编辑文件或执行任务。
- 绝不要围绕 Advisor 循环：每个检查点最多一次后续咨询。
- 如果 `advisor-subagent` subagent 不可用（没有 Task 工具，或 hook 拒绝），告知用户一次，然后继续而不使用它。
- 若要在整个会话（而非单条消息）中保持此 skill 的上下文，用户可以将 `/advisor` 作为 Custom Mode 调用（Option+Enter / Alt+Enter）。无论哪种方式，状态文件都有效。
