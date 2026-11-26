---
name: automate-me
description: "用于「automate me」「创建/更新/刷新我的 -mode skill」「把我的偏好或工作方式变成 skill」，或希望 agent 遵循用户的工作方式。通过 create-skill + unslop 起草或修订一个个人 -mode skill，可选择从最近的对话记录中提取新证据。"
disable-model-invocation: true
---

# Automate me

一个引导式流程，用于把用户的工作惯例转化为 agent 会遵循的 skill。产出的是一个为他们量身定制的 `-mode` skill（例如 `jay-mode`、`priya-mode`）。

本 skill 编排另外三个：一次内联的挖掘流程（见第 1 步）、Cursor 内置的 `create-skill`（编写），以及 **unslop** skill（文字风格约束）。它对三者进行排序，而不是取代它们。

## Flow

### 0. 检查是否已有 skill

递归查找 `.cursor/skills/**/*-mode/SKILL.md` 和 `~/.cursor/skills/*-mode/SKILL.md` 中与用户 handle 匹配的项。Mode skill 可以放在个人分类目录下（`.cursor/skills/<handle>/`），而不仅仅在顶层。如果存在，用 `AskQuestion` 确认意图（除非他们已经说过「update my skill」或类似表述）：

- 更新已有 skill（重复运行时默认）
- 从头开始（很少见，先问清原因再做）

更新模式会改变流程的其余部分：
- 第 1 步只挖掘该 skill 上次编辑之后的历史记录（`git log -1 --format=%cI <path>`）。
- 第 2 步询问发生了什么变化或缺了什么，而不是从零开始询问要捕捉什么。
- 第 4 步就地编辑已有文件。保留用户没有否定过的章节。修订有新证据的章节。只在确有全新规则时才添加新章节。

### 1. 挖掘他们的历史记录

在展开之前，先定位当前工作区的对话记录。系统提示中会给出该工作区的 `agent-transcripts/` 目录。只使用该路径。不要跨 `~/.cursor/projects/*/` 做 glob。那会跨越工作区边界，读取无关项目的私人聊天。

在该范围内梳理近期 agent 对话中的重复模式。针对历史记录的不同片段并行运行多个子 agent（例如最近 2-4 周，拆成 3 片，让每片都有足够材料）。按 **poteto-mode** skill 的 Subagents 一节，为每个子 agent 选择合适推理级别。每个切片挖掘子 agent 读取父级提供的工作区范围内路径下的对话记录，寻找下面的信号，并返回一个简短的结构化列表，列出所看到的模式及证据指针。默认值得搜寻的信号：

- 回复偏好（长度、语气、格式、「dumb it down」式纠正）
- 委派习惯（子 agent、模型、专门工作流、并行度）
- 验证姿态（「完成」意味着什么、单元测试 vs 实际复现、reviewer）
- 代码与文字风格纪律（风格、引用的原则、lint/format 工具）
- 流程惯例（worktree、commit、PR、review/merge 工具）
- 元偏好（任务中途修正 skill、提议新 skill）

在提升某个信号之前，先跨切片交叉核对。出现在 2 个以上切片中的模式为高置信度。孤立信号较弱，通常丢弃。

### 2. 直接询问用户

挖掘会漏掉尚未出现过的意图。使用 `AskQuestion` 工具（结构化多选），而不是让用户从零开始打字。

形式：一到两个问题，每个 4-6 个选项，类别问题用 `allow_multiple: true`。先从宽处问（「哪些方面最重要？」），再针对所选方面用具体选项追问。结构化轮次之后，再用一个自由格式的聊天问题捕捉选项遗漏的内容。

不要一口气抛出 20 个问题。

### 3. 聚类发现

把合并后的信号归组为章节。常见的如下（只使用适用的）：

- **Response style（回复风格）**：长度、语气、格式。
- **Autonomy（自主性）**：不问就做多少、MCP 工具使用。
- **Understand first（先理解）**：界定或调查变更时使用哪些 skill。
- **Subagents（子 agent）**：默认值、并行度、模型与任务的匹配、专门工作流。
- **Prose / code discipline（文字/代码纪律）**：原则、lint 工具、风格指南。
- **Review and verify（评审与验证）**：复现姿态、验证 skill、实测工具。
- **Process（流程）**：git worktree、commit、PR、review/merge 工具。
- **Skills（skill）**：skill 编写习惯、优先修正 skill、提议新 skill。

**poteto-mode** skill 展示了形式。阅读它以了解颗粒度。不要照搬其内容。用户的规则与 poteto-mode 的并不相同。

### 4. 起草 skill

使用 Cursor 内置的 `create-skill` skill 来编写该 skill。放置位置：

- 路径：保留已有 mode skill 的分类。对于新 mode，当仓库已有该 handle 的个人分类时使用 `.cursor/skills/<handle>/<handle>-mode/SKILL.md`。否则默认放在项目中的 `.cursor/skills/<handle>-mode/SKILL.md`（如果用户偏好个人 skill，则放在 `~/.cursor/skills/<handle>-mode/`）。
- Handle：用户的名字或选定的标识符。
- Frontmatter `description`：以他们的名字 + `/<handle>-mode` + 「按他们的方式工作」作为触发，而不是「write code」「review PR」这类通用关键词。
- Frontmatter 格式：遵循 `create-skill` 的 YAML 规则。将 `description` 保持为单个 YAML 标量。当标点或换行需要时，给它加引号，或使用 `description: >-` 配合缩进的续行。
- Frontmatter `disable-model-invocation: true` 为默认值。仅当用户明确希望其 mode 在每一轮都生效时才退出该设置。

### 5. 打磨文字

把 **unslop** skill 和 `create-skill` 的写作规范应用到每一行。

把草稿展示给用户并征求意见。预计会多轮迭代。果断删减。mode skill 不是说明书。

### 6. 落地

在基于 main 的 worktree 中工作。提交并开一个 PR。不要直接 push 到 main。

## Guardrails

- **不要过度拟合某一次对话。** 只说过一次、之后又被否定的偏好是噪音。编入规则前要求多次出现。
- **不要耍小聪明。** 复述其他 skill 的内容、发明隐喻，或为 agent 读者写「诗意」的文字，都是只有成本没有收益。保持可操作性。
- **引用，而非内联。** 用户依赖的其他 skill 应以路径引用出现，而不是粘贴摘录。他们维护在别处的任何原则文档也是如此。
- **章节保持精简。** 只有在用户对该处有明确的、非默认规则时才添加章节。「沟通清晰」不算一个章节。「短段落。比较选项时用表格。仅当条目确实并列时才用列表。」才算。
- **命名惯例通用化。** 在祈使句中使用「the user」或「the human」，而不是作者的名字。
- **不要强求对称。** 如果用户没有值得写下来的流程规则，就整个跳过 Process 章节。

## Evaluation

`-mode` skill 是主观产出。`create-skill` 风格的测试/迭代基准循环在这里没用。与用户做一次氛围检查：读起来像他们吗？漏掉了什么？然后发布。

只有当该 skill 的触发准确性在实践中确实成为问题时，才运行 description 优化循环。

## When not to use

- 用户想要一个针对具体任务的 skill（而非工作惯例）：单独用 `create-skill`，无需挖掘。
- 用户想捕捉一个很窄的工作流（例如「我怎么写 commit message」）。那是普通 skill，不是 mode skill。
