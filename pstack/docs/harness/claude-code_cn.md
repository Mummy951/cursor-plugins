# pstack 在 Claude Code 中

pstack 的 skill 使用 Cursor 的名称。在 Claude Code 中，每个会话读取本文件一次，然后用下面的表格把 skill 中的每个 Cursor 名称映射过来。`<pstack>` 指的是存放 `skills/` 和 `docs/` 的文件夹。如果某个 Cursor 字段在表格中没有对应行，就丢弃它并说明这一点。绝不要传递工具 schema 会拒绝的字段或值。

## 工具

| Cursor | Claude Code |
| --- | --- |
| `Task` | `Agent` |
| `AskQuestion` | `AskUserQuestion`。在 `-p` 模式下它不存在，所以用纯文本提问。 |
| 打开一个 todolist，或把步骤复制到 todolist 工具中 | 如果会话提供 `TodoWrite`，就使用它。否则把列表保留在你的回复中，并在每个阶段更新它。 |
| 运行另一个 pstack skill（`/name` 或 "the **name** skill"） | `Read` `<pstack>/skills/<name>/SKILL.md` 并遵循它。`Skill` 工具会拒绝大多数 pstack skill。当你告诉用户输入它时，写成 `/pstack:<name>`。 |
| `Grep`, `Glob` | 当会话没有 `Grep` 或 `Glob` 时，使用带 `rg` 的 `Bash`。 |

## 子代理字段

| Cursor | Claude Code |
| --- | --- |
| `subagent_type: generalPurpose` | `subagent_type: "general-purpose"` |
| `poteto-agent`, `comment-sicko` | `pstack:poteto-agent`, `pstack:comment-sicko` |
| `poteto-agent` 文件中的 `model:` 行 | 它包含的是一个 Cursor slug。在每一次派生 `pstack:poteto-agent` 的 `Agent` 调用中都要传递 `model` 和 `effort`，按上面的模型表格映射，这样那个 slug 就不会到达 API。 |
| `readonly: true` | `subagent_type: "Explore"`。它会保留 `Bash`，所以任务说明里也要写"不要编辑文件"，并且之后要检查 `git status`。 |
| `readonly: false` | `general-purpose`。它会保留会话的 MCP 服务器。 |
| `environment: "cloud"` | `isolation: "worktree"`。要指明 worker 检出的分支或提交。 |
| `environment: "local"` | 省略。 |
| `cloud_base_branch: <b>` | 告诉 worker 先检出 `<b>`。 |
| `run_in_background: true` | 省略。子代理默认在后台运行，完成时通知你。 |
| `resume` | 派生一个新的 agent，并把之前的结果粘贴到它的任务说明中。 |
| `file_attachments` | 在任务说明中列出这些路径。 |

## 模型

| Cursor | Claude Code |
| --- | --- |
| `claude-<family>-<version>-<effort>[-fast]`，例如 `claude-opus-5-5-xhigh` | `model: "opus"` 和 `effort: "xhigh"`。`model` 只接受 `opus`、`sonnet`、`haiku`、`fable`。去掉 `-fast`。 |
| 一个 `gemini-*` 或其他非 Claude 的 slug | 省略 `model`，让 agent 使用会话模型运行，并说明这一点。 |
| `auto` 或 `inherit-parent` | 省略 `model`。 |
| `pstack-models.mdc` 规则，`~/.cursor/rules/pstack-models.mdc` | `~/.claude/rules/pstack-models.md`。Claude Code 会把它加载到每个新会话中，所以它的角色行已经在上下文里了。如果没有，就读取该文件。如果文件或角色行缺失，就使用 skill 的默认 slug。 |
| 一个角色值 `<model> <effort>`，例如 `opus high` | `model: "opus"` 和 `effort: "high"`。 |

## setup-pstack

Claude Code 会把 `~/.claude/rules/` 中的每个 `.md` 文件加载到每个新会话中，所以 `setup-pstack` 把它的规则写到那里。遵循它的步骤，并做以下改动：

- 检测。检测到的模型就是 `Agent` 工具 schema 列出的 `model` 值，每个都配有一个来自其 `effort` 取值的努力级别。把真实模型写成 `<model> <effort>`，例如 `opus xhigh`。努力级别是最后一个 token，这样预算步骤就能按原样工作。
- 默认值。用上面的行映射每个默认 slug：`claude-opus-5-5-xhigh-fast` 是 `opus xhigh`，`claude-opus-5-5-high-fast` 是 `opus high`，`claude-fable-5-2-high` 是 `fable high`，`gemini-4-neon-xhigh` 是 `inherit-parent`。
- 写入。写入 `~/.claude/rules/pstack-models.md`，不带 frontmatter，然后是注释行、`# budget` 行，以及来自第 5 步的每个角色一行。覆盖整个文件。绝不要把角色行放进 `CLAUDE.md` 或 `AGENTS.md`。

## 路径

| Cursor | Claude Code |
| --- | --- |
| `.cursor/skills/`, `~/.cursor/skills/` | `.claude/skills/`, `~/.claude/skills/` |
| `~/.cursor/plugins/`, `.cursor-plugin/plugin.json` | `~/.claude/plugins/`, `.claude-plugin/plugin.json` |
| `.cursor/rules/*.mdc`，始终应用的规则 | `CLAUDE.md`，如果没有 `CLAUDE.md` 则用 `AGENTS.md`。Claude Code 从不读取 `.mdc` 文件。模型规则是例外。参见"模型"。 |
| `https://github.com/cursor/plugins/blob/main/pstack/<path>` | 当该文件存在时用 `<pstack>/<path>`。插件附带了 skills、`docs/guide/` 和 `README.md`。读取本地副本，绝不要去抓取它。当用户需要链接时，把 URL 给他们。 |
| `~/.cursor/projects/<slug>/agent-transcripts/` 中的会话记录 | `~/.claude/projects/<slug>/<session>.jsonl`。slug 是工作区路径，把每个 `/` 和 `.` 都改成 `-`，所以 `/Users/you/proj` 就变成 `-Users-you-proj`。该文件夹中直接存放的最新 `.jsonl` 就是当前会话。它的 `<session>/subagents/` 文件夹存放子代理的会话记录。只读取这个工作区的文件夹。 |

## 仅 Cursor 特性

Claude Code 没有 Cursor Automations、Bugbot、Cursor cloud agents 或内置 PR 工具。当某个步骤需要其中某个时，在回复中写一行 `Skipped in Claude Code: <step>. <reason>.`，然后继续下一步。不要停下、询问或重试。当后面的步骤需要被跳过步骤所产出的东西（例如 URL 或 key）时，留一个占位符并在那一行中点明它。

| Cursor | Claude Code |
| --- | --- |
| Cursor Automations：例程（routines）、`update_state` 例程调用、`api2.cursor.sh/automations` webhook、例程唤醒（routine wakes）以及 `SendToUser` 机密请求 | 用那一行跳过。 |
| Bugbot | Bugbot 在 GitHub PR 上的评论就是普通的评审线程（review thread），所以用该步骤给出的 forge 命令对它们进行分类处理。当 PR 没有 Bugbot 检查或评论时，跳过启动、等待或配置 Bugbot 的步骤。 |
| Cursor cloud agents | `environment: "cloud"` 映射到 worktree，如"子代理字段"中所述。跳过需要 cloud agent 本身的步骤，例如它的 dashboard 状态或它自己的机器。 |
| 内置 PR 工具 | 使用该步骤给出的 forge 命令，例如 `gh pr create`。这不属于跳过，所以不要写那一行。 |
