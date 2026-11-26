---
name: poteto-help
description: 指导用户完成 pstack 设置、/poteto-mode，以及为任务挑选合适的 skill、playbook 或 principle。带上问题输入 /poteto-help。
disable-model-invocation: true
---

# Poteto 帮助

回答用户关于 pstack 的问题，递给他们一条可以发送的提示词，并链接答案所来自的文件。对于帮助类问题，不要开始实际工作。用户问的是怎么做，而一次 pstack 运行会消耗真实 token，所以把提示词交给他们自己发送。

像 "use pstack to fix this bug" 这样要求干活的消息不是帮助类问题。阅读 [`poteto-mode`](../poteto-mode/SKILL.md)，在其下完成工作，并只提一次 Custom Mode 可以让它保持开启。

本文件把问题映射到承载答案的 skill 和指南页面。那些文件才掌握细节。在引用你路由到的文件之前先阅读它；当它与本映射不一致时，以它为准。这里的链接指向已安装的插件，用户可能无法打开，所以把该文件的公开副本给用户：`https://github.com/cursor/plugins/blob/main/pstack/` 后接它的路径。

## 弄清他们需要什么

从消息和对话中推断需求。像"哪个 skill 审查 PR？"这样点名了场景的，直接进入对应小节。如果需求仍不明确，用以下选项问一个选择题，然后只回答他们选择的小节：

- 完成设置
- 用 `/poteto-mode` 开始任务
- 为某个场景挑选 skill
- 修复出错的运行
- 把 pstack 变成自己的

检查会改变答案的状态，并只在它确实有影响时才提及：

- 没有 `~/.cursor/rules/pstack-models.mdc` 意味着 `/setup-pstack` 尚未为该用户运行过，因此每个角色都使用其默认模型。
- 项目中没有 `verify-*` skill 或其他应用测试框架（harness），意味着 agent 没有脚本化的方式来驱动应用。当问题是关于证明某项改动有效时，提及 `/create-verification-skill`。
- 项目中没有形式化验证 skill，意味着除了测试能覆盖的范围之外，没有任何属性被检查。当问题是关于竞态（race）、不变量（invariant）或测试很少触及的其他情形时，提及 `/create-formal-verification-skill`。当项目中已有该 skill，且它覆盖的代码在其记录的最后同步提交之后发生过变更时，若问题涉及那些代码，则提及 `/maintain-formal-verification-skill`。

当模型规则缺失且这一点很重要时，询问用户现在是否想为每个角色挑选模型以及设定推理预算。当用户是新手、问题关于设置或成本，或答案取决于运行哪些模型时，这一点就很重要。每次对话最多问一次。如果需求也不明确，就把两个问题一起问。提供两个选项：

- 现在：把 `/setup-pstack` 给他们输入，同时也回答他们的问题。
- 稍后：回答他们的问题，并加一行说明：在他们运行 `/setup-pstack` 之前，每个角色都保持其默认模型。

## 完成设置

1. 在聊天中输入 `/add-plugin pstack` 安装，或从侧边栏的 Customize 安装。
2. 运行 [`/setup-pstack`](../setup-pstack/SKILL.md)。它会询问推理预算，为每个角色映射模型，并写入一条规则。该规则适用于新对话。然后它会各提供一次机会，为项目生成 verification skill 和 formal verification skill，除非项目已有其中之一。
3. 用 `/poteto-mode`、一个目标，以及一个可成功或失败的检查来开始一个真正的任务。

在用户调用 skill 之前，安装不会改变任何东西。只有 `/setup-pstack` 会从用户的话中加载。[README](../../README.md) 和 [指南第 1 页](../../docs/guide/01-setup.md) 有详细说明。按照 [`references/prompting.md`](references/prompting.md)，主动提出帮他们措辞第一条提示词。

如果担心成本，就说明 token 花在哪里，以及如何少花一些。pstack 会把额外的 token 花在 subagent 和评审 panel 上。重新运行 `/setup-pstack`，选择更小的预算或更便宜的模型。设置为 `auto` 或 `inherit-parent` 的角色会在聊天所使用的模型上运行，当聊天运行在 Auto 或更便宜的模型上时，这能节省 token。更短的 panel 列表会运行更少的 subagent，每个条目对应一个。把 `/poteto-mode` 留给需要严谨性的工作。

pstack 是为 Cursor 构建的，它也可以安装在 Claude Code 中：先用 `/plugin marketplace add cursor/plugins`，再用 `/plugin install pstack@cursor-plugins`。在那里，每个命令都带有 `pstack:` 前缀，例如 `/pstack:poteto-mode`。[Claude Code 适配器](../../docs/harness/claude-code.md) 映射了 Cursor 的名称，而需要 Cursor 独有功能（如 Bugbot 或 Automations）的步骤会被跳过，并附一行说明。

其他工具也能读取这些 skill，因为它们使用 Agent Skills 格式。但大多数工作流 skill，包括 `/poteto-mode`、`/how` 和 `/teach`，都会派生 Cursor subagent，其中一些带有按角色划分的模型，而 Custom Modes 和 `/loop` 是 Cursor 的功能，所以那些部分在那里可能无法工作。

## 用 `/poteto-mode` 开始任务

`/poteto-mode` 会把任务匹配到一个 playbook，将该 playbook 的步骤复制到待办列表，并在步骤需要时运行其他 skill。一条好的提示词会说明目标以及如何判断完成。它不会列举 skill，因为手写的顺序往往会遗漏或重排 playbook 本会保留的步骤。在帮用户措辞之前，先阅读 [`references/prompting.md`](references/prompting.md)。[指南第 2 页](../../docs/guide/02-poteto-mode.md) 有示例。

`/poteto-mode` 是否保持开启取决于用户如何启动它：

- 在 `/poteto-mode` 上按 Enter 会将该 skill 附加到一条消息上。随着对话推进，它会逐渐失效。
- 在 Mac 上按 Option+Enter、在 Windows 上按 Alt+Enter，或从 skill 条目中选择 Use as Mode，会将其变成 Custom Mode。它会在每一轮都保留在上下文中，直到用户退出该模式，并且不会出现在随意的对话轮次中。
- Cursor 的文档在 Agents Window 和 CLI 中列出了 Custom Modes。在其他地方，每个新任务都用 `/poteto-mode` 启动。

当话题涉及此时，链接 [Cursor 的 skills 文档](https://cursor.com/docs/skills)。在对话中途，"new task" 会让该模式匹配一个新的 playbook。`/poteto-mode` 已经为其 playbook 步骤派生的 subagent 使用 `poteto-agent`。要让你自己的 subagent 获得同样的风格，用 `subagent_type: "poteto-agent"` 来派生它。

## 挑选 skill

默认答案是 `/poteto-mode`，它会在其步骤需要时运行大多数其他 skill。当用户想要的比 playbook 提供的多一些或少一些时，直接点名某个 skill。在推荐某个 skill 之前先阅读它，并给出一条示例提示词。

| 用户想要 | Skill |
|---|---|
| 严谨地完成任何非平凡任务 | [`/poteto-mode`](../poteto-mode/SKILL.md) |
| 在 agent 开始之前，确认它理解了用户的目标和问题 | [`/in-your-own-words`](../in-your-own-words/SKILL.md) |
| 在发送前把提示词改写得更深入而非更长：顶尖专业人士会补充什么、他们的原始素材和方法，以及一次外部检查。它返回提示词，而不执行任务 | [`/sharpen`](../sharpen/SKILL.md) |
| 为他们正在构建的东西学习专业术语，包括从邻近领域借来的词，检查每一处引用的来源，并用这些词改写他们的描述 | [`/reframe`](../reframe/SKILL.md) |
| 了解代码现在如何工作，或新代码应该放在哪里 | [`/how`](../how/SKILL.md) |
| 了解代码为何是这种形态，或某个数字从何而来 | [`/why`](../why/SKILL.md) |
| 理解一项改动或子系统，用通俗的话解释 | [`/teach`](../teach/SKILL.md) |
| 回顾他们自己近期在某个主题上的工作 | [`/recall`](../recall/SKILL.md) |
| 回顾自他们上一条消息以来本对话中的变化，用通俗的话和小的可视化呈现，列出每一件等待他们处理的事项，且仅凭回复就能做出决定 | [`/ketchup`](../ketchup/SKILL.md) |
| 像 `/ketchup` 一样回顾，但以带字幕的幻灯片短视频形式，下方以文字呈现待办调用 | [`/ketchup-video`](../ketchup-video/SKILL.md) |
| 了解一个小的 diff 可能在其自身之外破坏什么 | [`/blast-radius`](../blast-radius/SKILL.md) |
| 在编写跨越函数边界的代码之前，确定类型和模块形态 | [`/architect`](../architect/SKILL.md) |
| 针对同一份简报获得多次尝试，并合并成最好的一个 | [`/arena`](../arena/SKILL.md) |
| 以 cloud agent 的形式对切片并行检查，或让多个 worker 竞速 | [`/swarm`](../swarm/SKILL.md) |
| 让不同的模型审查一个 diff 并试图找出问题 | [`/interrogate`](../interrogate/SKILL.md) |
| 当存在廉价的本地测试时，以测试优先的方式修复 bug | [`/tdd`](../tdd/SKILL.md) |
| 对 `.ts` 或 `.tsx` 工作应用 TypeScript 规则 | [`/typescript-best-practices`](../typescript-best-practices/SKILL.md) |
| 在审查前移除注释，并让一个没有写过这些注释的审查者来审查 | [`/no-comments`](../no-comments/SKILL.md) |
| 清除散文中 AI 的痕迹 | [`/unslop`](../unslop/SKILL.md) |
| 按照规范撰写文档、RFC、README、PR 描述或提交信息 | [`/technical-writing`](../technical-writing/SKILL.md) |
| 用通俗的话再听一遍上一条回复 | [`/bro`](../bro/SKILL.md) |
| 为 agent 提供脚本化驱动应用并证明行为的方式 | [`/create-verification-skill`](../create-verification-skill/SKILL.md) |
| 让 verification skill 及其功能地图与应用重新保持一致 | [`/maintain-verification-skill`](../maintain-verification-skill/SKILL.md) |
| 形式化验证竞态、不变量或测试很少触及的其他属性 | [`/create-formal-verification-skill`](../create-formal-verification-skill/SKILL.md) |
| 在其覆盖的代码变更后让形式化验证重新保持一致，并弄清失败是漂移还是 bug | [`/maintain-formal-verification-skill`](../maintain-formal-verification-skill/SKILL.md) |
| 向 PM 或设计师展示用户所见内容的图表，该图表由 TLA+ 模型校验，并给出通往每个违反规则状态的最短路径 | [`/statechart`](../statechart/SKILL.md) |
| 在报告或依据某个性能数字采取行动之前对其进行审查 | [`/benchmark-checklist`](../benchmark-checklist/SKILL.md) |
| 执行大型或跨领域的改动，或需要暂离后再审查的改动 | [`/figure-it-out`](../figure-it-out/SKILL.md) |
| 在一次运行期间维护决策日志，并在之后回顾它 | [`/show-me-your-work`](../show-me-your-work/SKILL.md) |
| 为每个角色挑选模型并设定推理预算 | [`/setup-pstack`](../setup-pstack/SKILL.md) |
| 把他们自己的工作习惯变成个人 mode skill | [`/automate-me`](../automate-me/SKILL.md) |
| 把已完成任务带来的经验转化为 skill 修改 | [`/reflect`](../reflect/SKILL.md) |
| 防止 agent 在本仓库中重复同样的错误 | [`/correct`](../correct/SKILL.md) |
| 构建一个页面，其按钮通过 webhook 唤醒 Grok Bot | [`/make-bot-ui`](../make-bot-ui/SKILL.md) |
| 在 pstack 中找到方向 | `/poteto-help` |

如果表中缺少与本目录相邻的某个 skill 目录，请阅读它的 frontmatter 并根据其 description 进行路由。`principle-*` 目录将在下面的 principles 中介绍。

容易混淆的情况：

- `/how` 解释代码做什么。`/why` 解释原因。`/teach` 运行其中一个或两者，并用通俗的话解释结果。
- `/arena` 给每个 worker 相同的简报并合并最好的部分。`/swarm` 把工作拆成切片或竞速，并返回一份报告。
- `/interrogate` 审查 diff。`/blast-radius` 寻找 diff 之外的破坏，并证明使改动安全的那一个事实。
- `/recall` 在最近的多个对话中重建上下文。`/ketchup` 只覆盖本对话中自用户上一条消息以来的内容。恢复某个特定对话或分支属于 Session pickup playbook。
- `/figure-it-out` 设计一次严谨的运行。Autonomous run playbook 会驱动一个任务直到达成完成条件。

不在 pstack 中：

- `/deslop`、`control-cli` 和 `control-ui` 随 `cursor-team-kit` 插件提供。
- `/loop` 和 `/create-skill` 是 Cursor 内置的。
- pstack 没有 `/orchestrate` skill。如果斜杠菜单中出现 `/orchestrate`，那是由另一个插件提供的。

## Playbooks 和 principles

Playbooks 是 `/poteto-mode` 内部的步骤列表，不是 skill，因此它们没有斜杠命令。在 `/poteto-mode` 内部，描述任务就会选中一个，而以下短语可以直接点名某个 playbook：

- "babysit this pr" 或 "check on pr 123" 会运行 Babysit。它会把 PR 推进到可合并状态并停在那里。除非用户要求 merge、land 或 ship，否则它不会合并。
- "land the stack" 会运行 Shipping。
- "take over this branch" 会运行 Session pickup。
- "pause safely" 会运行 Pause safely。
- "full autopilot on this queue" 会运行 Autopilot-full。"autopilot, don't merge" 会运行 Autopilot-stack。
- "run the eval playbook" 会运行 Eval。

如果没有 `/poteto-mode`，像 "babysit this pr" 这样的短语可能会启动 Cursor 自身用于同类工作的 skill。[`poteto-mode`](../poteto-mode/SKILL.md) 的 Playbooks 小节列出了所有 playbook。[指南第 6 页](../../docs/guide/06-verify-and-ship.md) 介绍了如何创建、照看和落地 PR。

pstack 没有规划 skill。Cursor 的 Plan Mode 可以与之配合使用。对于跨越多个阶段或堆叠 PR 的工作，向 `/poteto-mode` 要求制定计划会运行 [Multi-phase plan playbook](../poteto-mode/playbooks/multi-phase-plan.md)，它只写计划而不实现。对于设计问题，Prototype playbook 或 `/architect` 会先用代码确定方案。

Principles 是单条规则的 skill，`/poteto-mode` 会阅读它们并在回复中引用。用户很少直接调用它们。他们更倾向于用名称来引导，例如 "apply prove it works. show me the real output."。输入 `/principle-<name>` 仍然可以按需加载某一个。[指南第 8 页](../../docs/guide/08-principles.md) 列出了它们。

## 修复出错的运行

| 症状 | 修复 |
|---|---|
| 该模式在几轮之后不再生效 | 它是用 Enter 启动的。请以 Custom Mode 启动，或用 `/poteto-mode` 启动每个任务。 |
| 一个问题被当作上一个任务的下一步 | 说 "new task"，或说明这一轮不需要该模式。 |
| 新选择的模型没有生效 | `/setup-pstack` 生成的规则适用于新对话。开一个新对话。 |
| 运行成本高于预期 | 参见"完成设置"下的成本段落。 |
| 某个 skill 没有自动加载 | 只有 `/setup-pstack` 会从用户的话中加载。其他 skill 在用户输入它们或 `/poteto-mode` 运行它们时才会加载，而 `/poteto-mode` 并不会运行所有 skill。 |
| 并行 agent 互相覆盖了彼此的改动 | 给每个 agent 自己的 worktree，或把它们作为 cloud agent 运行，这样每个 agent 都有自己的机器。 |
| 一次通宵运行有进展但没有完成任何事 | `/loop` 需要一个可成功或失败的检查，而不是时长。参见[指南第 7 页](../../docs/guide/07-overnight.md)。 |
| 回复仅凭一次成功的构建就宣称成功 | 要求提供真实的命令、流程、存储的值或性能分析（profile）。这就是 prove-it-works principle。 |

对于运行偏离方向的情况，[`references/prompting.md`](references/prompting.md) 提供了单行引导语。[指南第 10 页](../../docs/guide/10-recipes-and-pitfalls.md) 有更多常见陷阱和值得照搬的配方。

## 把 pstack 变成自己的

- [`/automate-me`](../automate-me/SKILL.md) 根据用户自己的历史记录起草一个个人 mode skill，与 `/poteto-mode` 配合使用。
- 在一次会话后使用 [`/reflect`](../reflect/SKILL.md)，把其中的经验教训转化为用户认可的 skill 修改。
- `/poteto-mode write a skill for <workflow>` 会运行 authoring playbook。eval playbook 会以盲测方式检验对 skill 的修改。
- 在一个独立的 PR 中修复行为异常的 skill，而不是在它出问题的功能开发工作中顺手修复。

[指南第 9 页](../../docs/guide/09-make-it-yours.md) 逐一介绍了这些内容。

## 回复

先给出答案。最多在代码块中给出一条示例提示词，当合适时根据 [`references/recipes.md`](references/recipes.md) 改编，然后附上该文件的链接。除非用户要求完整映射，否则保持简短。
