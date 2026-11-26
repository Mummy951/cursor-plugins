# pstack

我是 [poteto](https://x.com/poteto)。我不是总裁也不是 CEO，但我在 Meta、Netflix 和 Cursor 处理过数百万行代码。我也是 react 核心团队的一员，在那里帮助构建和维护 react compiler。

越来越多的人觉得 ai 写出了太多劣质代码（slop code）。我同意。我不想让交付成果像一个由二十个「劣质代码艺术家」组成的团队产出的东西。没有质量的吞吐量不是我追求的目标。如果你想快，先深入。

**pstack 就是我的答案。** 这些正是我每天在 Cursor 用来交付高质量代码的技能。它把 cursor 变成一个真正的工程团队。目标不是最大化代码行数（loc），事实上恰恰相反。pstack 帮助你写更少、但更高质量的代码。

**pstack 让你无所畏惧地并行。** 当你能在一个 agent 上深入下去，并相信它能写出优秀、可验证的代码时，你才能真正有信心地并行。用 `poteto-mode` 启动多个 agent，相信它们会把严格的工程原则应用到工作中。

**cursor 让你集各家之所长。** 每个前沿模型都有长处和短处。用 pstack 你可以使用任何模型。事实上，我的许多技能都使用多模型工作流，以利用每个模型的独特优势。

fork 它。改进它。把它变成你的。欢迎 PR！

## 安装

```bash
/add-plugin pstack
```

在 claude code 中：

```bash
/plugin marketplace add cursor/plugins
/plugin install pstack@cursor-plugins
```

claude code 默认对第三方 marketplace 关闭自动更新，所以 pstack 会保持在你安装时的版本。要打开它，运行 `/plugin`，打开 **marketplaces** 标签页，选择 `cursor-plugins`，然后选择 **enable auto-update**。要手动更新一次，运行 `claude plugin update pstack@cursor-plugins`。

claude code 会给插件命令和 agent 加前缀，所以 `/poteto-mode` 变成了 `/pstack:poteto-mode`。[claude code adapter](./docs/harness/claude-code.md) 映射了其他所有 cursor 名称。在 claude code 中，`/pstack:setup-pstack` 会把你的模型选择写入 `~/.claude/rules/pstack-models.md`，而需要 cursor 专属功能（如 bugbot 或 automations）的步骤会被跳过，并附上一行说明。

claude code 会从仓库根目录的 [`harness/pstack/`](https://github.com/cursor/plugins/tree/main/harness/pstack) 安装一份生成的 pstack 副本，包括这份 readme 和指南。修改 pstack 后，在仓库根目录运行 `node scripts/generate-pstack-harness-trees.mjs` 并提交结果。当生成的副本过期时，ci 会失败。

## 开始使用

两步：

1. 运行 [`/setup-pstack`](./skills/setup-pstack/SKILL.md)，选择一个推理预算，并挑选你想要的模型。
2. 当你做任何需要严谨性的事情时，使用 [`/poteto-mode`](./skills/poteto-mode/SKILL.md)。

第一次来？[pstack guide](./docs/guide/README.md) 会带你完成第一个真实任务，从设置和提示到验证和隔夜运行。卡住了，或者不确定哪个技能合适？问问 [`/poteto-help`](./skills/poteto-help/SKILL.md)。

就这样。其他技能是按需使用的，模式技能会在需要时替你调用它们。开箱即用时，代码委派（feature、refactoring、bug fix、perf、hillclimb）、文案和判断都交给 xhigh 级别的 opus 5.5，没有任何角色运行在 xhigh 之上。在评审组之外，fable 5.2 只在你点名它、或同一任务的两次 opus 5.5 运行失败后才运行。默认评审组是 fable 5.2 / opus 5.5 / gemini 4 neon。[`/setup-pstack`](./skills/setup-pstack/SKILL.md) 可以更改其中任何一项。

## 用法

在任务开始时使用 [`/poteto-mode`](./skills/poteto-mode/SKILL.md)。它读取你的请求，从一组 playbook 中挑选，并在步骤需要时运行其他技能。

在输入任务之前，从 `/` 菜单中选择 `/poteto-mode`，这样 cursor 会把该技能附加到消息上。

### 直接使用 [`/poteto-mode`](./skills/poteto-mode/SKILL.md)

这个技能是主要的快捷入口。每当我需要 agent 做严格的工程工作时就会用它。它带有二十二个 playbook：

```
/poteto-mode this pr has a subtle bug where the scroll drifts every 750ms even when idle. repro
first, then fix and verify.
```

```
/poteto-mode i'm going to bed. land the stack even if ci flakes. i want everything merged by
morning.
```

<details>
<summary>二十二个 playbook</summary>

| playbook | 适用场景 |
|---|---|
| [investigation](./skills/poteto-mode/playbooks/investigation.md) | 一个只读问题。x 是如何工作的，y 为什么要这样构建，我们确定吗。 |
| [bug fix](./skills/poteto-mode/playbooks/bug-fix.md) | 复现一个缺陷，找到根本原因，并用运行时证据修复它。 |
| [perf](./skills/poteto-mode/playbooks/perf-issue.md) | 追踪测量到的性能下降，并对照基线改进它。 |
| [hillclimb](./skills/poteto-mode/playbooks/hillclimb.md) | 针对某个目标，对一个指标进行持续、科学的改进，循环提出假设并进行前后测量，每接受一项成果就提交一次。 |
| [runtime forensics](./skills/poteto-mode/playbooks/runtime-forensics.md) | 根据插桩数据诊断线上症状（内存泄漏、空闲 CPU 空转、卡顿）。 |
| [trace forensics](./skills/poteto-mode/playbooks/trace-forensics.md) | 诊断捕获的性能分析产物（cpuprofile、trace、spindump、heap snapshot）。 |
| [feature](./skills/poteto-mode/playbooks/feature.md) | 新的或改变的行为，基于一个命名的数据结构构建。 |
| [refactoring](./skills/poteto-mode/playbooks/refactoring.md) | 保持行为不变的结构或形态变更。 |
| [prototype](./skills/poteto-mode/playbooks/prototype.md) | 一次性的草图，用于低成本地做出设计或行为决策，或通过观察来解决经验性的分歧。 |
| [visual parity](./skills/poteto-mode/playbooks/visual-parity.md) | 两种实现之间的像素级 UI 一致性。 |
| [authoring a skill](./skills/poteto-mode/playbooks/authoring-a-skill.md) | 编写或编辑 SKILL.md。 |
| [eval](./skills/poteto-mode/playbooks/eval.md) | 以盲测方式检验技能或提示词的改动如何影响 agent 行为。 |
| [babysit](./skills/poteto-mode/playbooks/babysit.md) | 推动一个 pr 或一组堆叠 pr 达到可合并状态：冲突、评审线程、ci。 |
| [shipping](./skills/poteto-mode/playbooks/shipping.md) | 独立验证一组全绿的堆叠 pr，然后通过 github（默认）或可用时的 origin 自下而上地落地连续验证过的部分。 |
| [autonomous run](./skills/poteto-mode/playbooks/autonomous-run.md) | 驱动一个长任务直到完成，中途不停下。 |
| [autopilot-full](./skills/poteto-mode/playbooks/autopilot-full.md) | 让独立的 pr 一路运行到合并，每个 pr 一个负责人。swarm 会验证每个代码就绪的头部，补丁的每一次后续改动都会得到一次新的检查。 |
| [autopilot-stack](./skills/poteto-mode/playbooks/autopilot-stack.md) | 不含合并的 autopilot-full。构建并验证 pr，供操作者审阅和落地，只把 pr 堆叠在它所依赖的 pr 之上。 |
| [session pickup](./skills/poteto-mode/playbooks/session-pickup.md) | 恢复或接管先前 agent 正在进行的工作。 |
| [pause safely](./skills/poteto-mode/playbooks/pause-safely.md) | 干净地挂起正在进行的工作，以便稍后恢复。 |
| [multi-phase plan](./skills/poteto-mode/playbooks/multi-phase-plan.md) | 跨阶段或跨堆叠 PR 的工作。 |
| [worktree cleanup](./skills/poteto-mode/playbooks/worktree-cleanup.md) | 通过清理已合并或废弃的 worktree 以及过时的 ios 模拟器来回收磁盘空间，带有安全门控。 |
| [opening a pr](./skills/poteto-mode/playbooks/opening-a-pr.md) | 从一个准备好、有序的小型提交中开启 pr，使用 conventional commits 风格的标题和简报风格的正文。在每个其他 playbook 结束时调用。 |

</details>



调用时，它会：

1. 将你的任务匹配到一个 [playbook](./skills/poteto-mode/playbooks/)，并打开一个待办列表，其开头几项就是它的步骤，逐字复制。
2. 在步骤触发时路由到其他技能。
3. 写出没有 slop 的回复，并分别面向使用者和维护者来组织。

完整规则和 playbook 位于 [`skills/poteto-mode/SKILL.md`](./skills/poteto-mode/SKILL.md)。

要让 [`/poteto-mode`](./skills/poteto-mode/SKILL.md) 跨回合保持开启，从 `/` 菜单中选择它，然后按 option+enter（mac）或 alt+enter（windows）而不是 enter。这会把它变成一个 [custom mode](https://cursor.com/docs/skills#using-a-skill-as-a-custom-mode)，cursor 会在 agents 窗口和 cli 中提供该模式。它会一直留在上下文中，每一回合都有效，直到你退出该模式。普通 enter 只把它附加到一条消息上。

[`/poteto-mode`](./skills/poteto-mode/SKILL.md) 与 cursor 的 `/loop` 命令配合得非常好。你可以让 cursor 工作好几个小时而不牺牲严谨性。

## 技能

当某个步骤需要时，[`/poteto-mode`](./skills/poteto-mode/SKILL.md) 会替你运行其中大多数（`why`、`architect`、`arena`、`swarm`、`interrogate`、`unslop`、`no-comments`、`technical-writing`、`tdd` 以及各项原则）。下面的表格适用于你想直接调用某个技能时：

```
/how do we cancel runs? do we have an n+1 when we look up every run to cancel?
```

```
/interrogate review this pr.
```

<details>
<summary>全部技能</summary>

| 技能 | 何时使用 |
|---|---|
| [`/poteto-mode`](./skills/poteto-mode/SKILL.md) | 任何非平凡任务的默认入口。 |
| [`/poteto-help`](./skills/poteto-help/SKILL.md) | 你是 pstack 新手，或不确定哪个技能、playbook 或原则合适。它会弄清你想做什么，回答那部分问题，并给你一个可以输入的提示词。只在你输入 `/poteto-help` 时运行。 |
| [`/in-your-own-words`](./skills/in-your-own-words/SKILL.md) | 你希望 agent 在开始之前复述你的目标和问题。只在你输入 `/in-your-own-words` 时运行。 |
| [`/sharpen`](./skills/sharpen/SKILL.md) | 你希望在发送提示词之前把它变得更深、而不是更大：一位顶尖专业人士会做而没有要求你做的那些事、他们起步的原始材料、他们的方法和工具，以及一次确认结果正确的外部检查。它会交回改写后的提示词，要求 agent 对任何可以通过实验定夺的选择做原型，并把产品和偏好方面的选择留给你。它不执行任务。只在你输入 `/sharpen` 时运行。 |
| [`/reframe`](./skills/reframe/SKILL.md) | 你说不清自己在构建什么，因此无法清楚地向 agent 提出要求。它从你的文件和你自己的话开始。如果这些可能描述不止一个问题，它会列出可能的两个或三个问题，并给出区分各自的标志，然后挑出最可能的一个。接着它给出 8 到 12 个专家术语，按它们对你所构建内容的改变程度排序，每个都关联到你描述中它所指名的部分，以及一个它已核对过链接的真实项目。它从 2 到 3 个相邻领域借用术语——这些领域用别的名字解决过同样的问题，说明每个类比在哪里失效，然后用新词重写你的描述。只在你输入 `/reframe` 时运行。 |
| [`/how`](./skills/how/SKILL.md) | 你想了解某个子系统如何工作的讲解。 |
| [`/why`](./skills/why/SKILL.md) | 你想知道某样东西为什么是这样构建的。它从记录而非代码中寻找答案：引入它的提交和 pr 讨论、之后的每一次改动，以及当线索不足时的其他来源。它引用作者写下的内容，标注自己的推断，并说明记录没有回答的部分。 |
| [`/recall`](./skills/recall/SKILL.md) | 你正在开始或恢复工作，希望从自己的聊天记录和共享记录中重建某个主题的最近上下文，并以一份紧凑的当前状态简报交回给你。 |
| [`/ketchup`](./skills/ketchup/SKILL.md) | 你想知道自上次消息以来这次聊天中发生了什么变化，用朴素的语言和小的可视化呈现，并把每一件等着你处理的事都列出来，让你仅凭回复就能做决定——即使这会让回复变长。 |
| [`/ketchup-video`](./skills/ketchup-video/SKILL.md) | 你想把 `/ketchup` 的进度同步变成一个带字幕的幻灯片短视频，下面用文字列出等着你处理的事项。 |
| [`/blast-radius`](./skills/blast-radius/SKILL.md) | 你有一个看起来很小的改动，想知道它还可能破坏什么，并且那个证明它安全的关键事实是通过运行代码证明的，而不是口头断言。 |
| [`/architect`](./skills/architect/SKILL.md) | 你即将编写跨越函数边界的代码，希望先确定调用方的使用方式、类型和模块形态。 |
| [`/arena`](./skills/arena/SKILL.md) | 你想对同一件事做 N 次并行尝试，然后取各自最好的部分。 |
| [`/swarm`](./skills/swarm/SKILL.md) | 你想让 N 个并行 worker 处理不同的切片或竞速方案，然后得到一份汇总报告。 |
| [`/interrogate`](./skills/interrogate/SKILL.md) | 你有一个 diff，想让不同的模型试着攻破它，只保留经过某次运行证实的发现。 |
| [`/automate-me`](./skills/automate-me/SKILL.md) | 你想要一个自己的 `-mode` 技能，根据你的实际工作方式起草。 |
| [`/make-bot-ui`](./skills/make-bot-ui/SKILL.md) | 你想要一个页面或仪表盘，其按钮通过 webhook 唤醒 Grok Bot，包括 sender-key 的交接和 Tailscale。 |
| [`/setup-pstack`](./skills/setup-pstack/SKILL.md) | 你想选择 pstack 每个角色使用哪个模型。它会检测你的模型并写入一条配置规则。 |
| [`/reflect`](./skills/reflect/SKILL.md) | 一个长任务落地了，你想把其中的做法记录为一次技能编辑。 |
| [`/correct`](./skills/correct/SKILL.md) | 你一直在为同样的错误纠正 agent。它从历史中挖掘错误类别，在最高有效层级修复每一类（先是架构，然后是类型、lint 和 ci，再是测试，文档最后），并维护一张表格，把每条规则与强制执行它的手段配对。 |
| [`/teach`](./skills/teach/SKILL.md) | 你想真正理解一个改动或子系统，而不只是要一份摘要。它运行 how + why，并编织出一份朴素的解释，用图表一步步构建起来。 |
| [`/tdd`](./skills/tdd/SKILL.md) | 你在修复一个 bug，并且有一条低成本的本地测试路径。先写失败的测试，再写修复。 |
| [`/benchmark-checklist`](./skills/benchmark-checklist/SKILL.md) | 你跑了一个基准测试，或测量到一次加速或性能回退。在你报告或据此行动之前，它会审查这个数字（限速器、调优、错误、重复运行、端到端相关性）。 |
| [`/no-comments`](./skills/no-comments/SKILL.md) | 在评审前去除注释；派生 Comment Sicko，修复被接受的发现，并为声称的约束提供编码方案。 |
| [`/typescript-best-practices`](./skills/typescript-best-practices/SKILL.md) | 你在阅读或编辑 typescript。它把 type-system-discipline 原则落实到语法层面。 |
| [`/figure-it-out`](./skills/figure-it-out/SKILL.md) | 没有现成的 playbook 适用。它为该任务设计一个严格、可审计的 playbook。 |
| [`/show-me-your-work`](./skills/show-me-your-work/SKILL.md) | 你想要一条可审阅的决策轨迹。它把决策记录到一个你可以提交的 tsv 中。 |
| [`/create-verification-skill`](./skills/create-verification-skill/SKILL.md) | 你的项目没有脚本化的方式来证明应用行为。它生成一个项目本地的验证技能，带有功能映射，适用于任何语言或平台。 |
| [`/maintain-verification-skill`](./skills/maintain-verification-skill/SKILL.md) | 你的验证技能的功能映射已经与应用脱节。源头扫描 + 一次实际运行，最多提交一个经过证实的修正 PR。 |
| [`/create-formal-verification-skill`](./skills/create-formal-verification-skill/SKILL.md) | 测试无法触及会破坏竞态、重试、状态机、解析器或 unsafe 代码的情况。它用合适的工具验证一个属性，并生成一个项目本地的技能，在每次改动时重新检查。 |
| [`/maintain-formal-verification-skill`](./skills/maintain-formal-verification-skill/SKILL.md) | 自上次检查以来，你的形式化验证所覆盖的代码发生了变化。它更新模型、测试框架或属性以匹配代码，重新运行，并说明每次失败是验证本身的偏移还是代码中的 bug。 |
| [`/statechart`](./skills/statechart/SKILL.md) | 负责该行为的人需要看到它，而不是读规格说明。它检查一个 tla+ 模型，把用户看到的内容画成 mermaid 状态图，标出每个违反规则的状态，并给出到达每个状态的最短路径。 |
| [`/unslop`](./skills/unslop/SKILL.md) | 你在清理文字。它会去除 AI 痕迹。 |
| [`/bro`](./skills/bro/SKILL.md) | 你想用朴素的人类语言重述上一条消息，不要术语。 |
| [`/technical-writing`](./skills/technical-writing/SKILL.md) | 用于文档、RFC、readme、PR 描述、提交信息的分层文档标准（Diátaxis + Google developer style + STE + Global English）。 |

</details>



### 示例

大多数时候，我在任务开始时就输入 [`/poteto-mode`](./skills/poteto-mode/SKILL.md)，让它路由到一个 playbook。其他技能会在步骤需要时触发。有少数几个我会直接调用。


<details>
<summary>全部示例</summary>

```
bug fix:           /poteto-mode this pr has a subtle bug where the scroll drifts every 750ms even
                   when idle. repro first, then fix and verify.
perf:              /poteto-mode a big list takes a second or two to load even though we virtualize.
                   run a cpu trace and tell me why.
feature:           /poteto-mode build a small feature behind a feature flag. verify it really works.
prototype:         /poteto-mode build two prototypes of the markdown renderer so we can compare.
                   spawn an agent for each.
multi-phase:       /poteto-mode open source these skills as a plugin. nothing internal leaks, work
                   in a temp dir, show me the dependency graph first.
overnight run:     /poteto-mode i'm going to bed. land the stack even if ci flakes. i want
                   everything merged by morning.
babysit:           /poteto-mode check on pr 123. anything outstanding?
visual parity:     /poteto-mode the row spacing is too tall when this flag is on. the second image
                   is correct. repro and fix until it matches.
figure it out:     /poteto-mode i'm stepping away. migrate every caller from the synchronous store
                   to the new async one, keeping behavior identical. i want to trust it was done
                   right when i'm back.
how:               /how do we cancel runs? do we have an n+1 when we look up every run to cancel?
why:               /why is this feature flag not on yet?
architect:         design this instrumentation to be high signal with no false positives. /architect
                   this first.
arena:             /arena take my prompt to the arena verbatim. i want to compare their proposals
                   with yours.
swarm:             /swarm check every package under packages/ against its check.sh. one worker per
                   package. one report.
interrogate:       /interrogate review this pr.
tdd:               /tdd implement
unslop:            can we unslop and tighten the new changes?
reflect:           /reflect that took too long. capture what we learned so the next run doesn't
                   repeat it.
correct:           /correct
show-me-your-work: /show-me-your-work keep a decision trail i can review when i'm back.
automate-me:       /automate-me
help:              /poteto-help which skill should i use to review this branch?
```

</details>

## `poteto-agent` 和 Comment Sicko 子代理

[`/poteto-mode`](./skills/poteto-mode/SKILL.md) 在接收它的聊天中运行。该聊天拥有 playbook、待办列表和每一次子代理派生。

pstack 还附带 [`subagent_type: "poteto-agent"`](./agents/poteto-agent.md)，作为运行期间派生的 delegate 的包装器。它在做任何工作之前会完整阅读 `poteto-mode`，包括其内联的原则索引。替换成 `generalPurpose` 会跳过这次阅读并导致偏离。不要派生 `poteto-agent` 去运行 playbook 本身。

[`/poteto-mode`](./skills/poteto-mode/SKILL.md) 和 [`subagent_type: "poteto-agent"`](./agents/poteto-agent.md) 都通过同一个包装器路由。

pstack 还附带 [Comment Sicko](./agents/comment-sicko.md)，一个只读的注释审阅者，可通过 `subagent_type: "comment-sicko"` 使用。通常通过 [`/no-comments`](./skills/no-comments/SKILL.md) 调用它，而不是直接调用。

## 原则

二十四个短技能，每个对应一条原则。`poteto-mode` 将它们内联索引，并在任务开始时读取该索引。独立文件的存在是为了让其他技能可以按名称引用某条原则，并让索引指向每条原则的完整规则。

<details>
<summary>全部二十四条原则</summary>

| 原则 | 分组 | 规则 |
|---|---|---|
| [laziness-protocol](./skills/principle-laziness-protocol/SKILL.md) | 核心 | 倾向于删除，以及用最小的改动解决问题。 |
| [foundational-thinking](./skills/principle-foundational-thinking/SKILL.md) | 核心 | 在编写逻辑之前应用：选择核心类型和数据结构，安排脚手架与功能工作的先后顺序，追问并发参与者共享什么。把数据结构弄对，下游代码自然就清晰了。 |
| [redesign-from-first-principles](./skills/principle-redesign-from-first-principles/SKILL.md) | 核心 | 像这个需求从第一天起就是一项基础假设那样重新设计，而不是事后把它硬加上去。 |
| [attack-the-premise](./skills/principle-attack-the-premise/SKILL.md) | 核心 | 当两个或更多共享同一前提的修复方案都在同一道关卡上失败时应用。在写下一个假定该前提成立的修复之前，先把前提写下来并直接测试它。 |
| [subtract-before-you-add](./skills/principle-subtract-before-you-add/SKILL.md) | 核心 | 先移除累赘、冗余的校验器和占位引用，然后在这个更简单的基础上构建。 |
| [minimize-reader-load](./skills/principle-minimize-reader-load/SKILL.md) | 核心 | 数一数问题与答案之间的层数，以及读者脑中隐藏的状态；合并只有一个调用方的包装层，缩小可变作用域。 |
| [outcome-oriented-execution](./skills/principle-outcome-oriented-execution/SKILL.md) | 核心 | 在有明确阶段边界的计划内重写和迁移中应用。收敛到目标架构；不要用一次性的兼容代码去维持平滑的中间状态。 |
| [experience-first](./skills/principle-experience-first/SKILL.md) | 核心 | 选择让用户愉悦，而不是实现上的便利；交付更少但更精致的功能，而不是更多粗糙的功能。 |
| [exhaust-the-design-space](./skills/principle-exhaust-the-design-space/SKILL.md) | 核心 | 在定下来之前，构建 2 到 3 个相互竞争的原型并并排比较。 |
| [build-the-lever](./skills/principle-build-the-lever/SKILL.md) | 核心 | 适用于任何非平凡的工作，而不只是批量工作：编辑、迁移、分析、检查。构建能完成它或证明它的工具（codemod、脚本、生成器，或你的子代理遵循的技能），而不是手工操作。这个工具就是审阅者可以重新运行的产出物。 |
| [model-the-domain](./skills/principle-model-the-domain/SKILL.md) | 架构 | 把领域编码到结构中，而不是散落的条件判断里。 |
| [boundary-discipline](./skills/principle-boundary-discipline/SKILL.md) | 架构 | 把防护集中在系统边界（CLI、配置、网络、外部 API）；信任内部类型，把业务逻辑放在纯函数中。 |
| [type-system-discipline](./skills/principle-type-system-discipline/SKILL.md) | 架构 | 让非法状态无法表示，给语义基元打上品牌标记，在边界解析外部数据，穷尽所有变体，从权威 schema 派生。 |
| [make-operations-idempotent](./skills/principle-make-operations-idempotent/SKILL.md) | 架构 | 无论之前是否有部分运行，都收敛到相同的最终状态。 |
| [migrate-callers-then-delete-legacy-apis](./skills/principle-migrate-callers-then-delete-legacy-apis/SKILL.md) | 架构 | 在同一波中迁移调用方并删除旧 API，而不是保留兼容层。 |
| [separate-before-serializing-shared-state](./skills/principle-separate-before-serializing-shared-state/SKILL.md) | 架构 | 先消除共享；只有当单一共享写入者是一项真正的不变量时，才从结构上做串行化。 |
| [prove-it-works](./skills/principle-prove-it-works/SKILL.md) | 验证 | 在完成任务之后、宣布完成之前应用。对照真实产物进行验证（运行该功能、读取实际值、检查 diff），而不是代理指标、自我报告或「能编译」。 |
| [fix-root-causes](./skills/principle-fix-root-causes/SKILL.md) | 验证 | 把每个症状追溯到系统第一次出错的地方，并在那里修复。只有当防护所处理的状态是合法的时候，防护才算一种修复。 |
| [sequence-verifiable-units](./skills/principle-sequence-verifiable-units/SKILL.md) | 验证 | 适用于多步骤工作（批量扫描、迁移、一连串相似编辑），也适用于你如何堆叠提交和 PR。把工作拆成小单元，每个单元都以可验证的状态结束，在下一个之前检查每一个，并安排交付顺序，让整个序列向审阅者自证。 |
| [test-behavior-not-implementation](./skills/principle-test-behavior-not-implementation/SKILL.md) | 验证 | 在你编写、修改或保留测试时应用。断言代码使用者能观察到的东西，这样当该行为被破坏时测试会失败，而只有内部实现改变时测试仍会通过。 |
| [explain-the-number](./skills/principle-explain-the-number/SKILL.md) | 验证 | 在你信任、报告或根据一个测量出的数字采取行动之前应用：加速比、性能回退、吞吐量、延迟或 eval 结果。找出限制它的因素，并排除它测量的其实不是你以为的那项工作。 |
| [guard-the-context-window](./skills/principle-guard-the-context-window/SKILL.md) | 委派 | 把批量工作路由给子代理；在主线程中保留摘要，而不是原始负载。 |
| [never-block-on-the-human](./skills/principle-never-block-on-the-human/SKILL.md) | 委派 | 继续推进、呈现结果，让人类事后纠正方向；把确认留给不可逆的操作。 |
| [encode-lessons-in-structure](./skills/principle-encode-lessons-in-structure/SKILL.md) | 元 | 把规则编码为 lint、元数据标记、运行时检查或脚本，而不是更多文字。 |

</details>

## 未在此附带

有少量 `poteto-mode` 引用但未打包的内容：

- `/deslop` 和 `deslop` 技能随 `cursor-team-kit` 插件提供。
- `control-cli`（用于 CLI 和 TUI）和 `control-ui`（用于浏览器、Electron、web）也随 `cursor-team-kit` 提供。
- `/create-skill` 是 cursor 内置技能。cursor 还附带一个内置的 `/autopilot`，用于监视 pr。在 `poteto-mode` 内部，pr 状态请求会转到 [babysit playbook](./skills/poteto-mode/playbooks/babysit.md)。

如果你想要完整的一套，就把 `cursor-team-kit` 和 pstack 一起安装。

## 为什么没有规划技能？

cursor 已经有一个很棒的 plan mode，它与 pstack 配合得很好。但就我个人而言，我不相信规划。最好的规格就是代码。如果你确实想制定计划，[`/poteto-mode`](./skills/poteto-mode/SKILL.md) 可以涵盖，但这不是默认做法。

## 把它变成你的

`poteto-mode` 是我的风格。你可能并不想要完全一样的。

输入 [`/automate-me`](./skills/automate-me/SKILL.md)。它挖掘你最近的对话记录，根据你的实际工作方式起草一个 `<your-name>-mode` 技能，并在底层通过 pstack 路由。你保留 pstack 作为基础，最终得到自己拥有的路由技能，与 `poteto-mode` 并肩存在。

模型也是可配置的。输入 [`/setup-pstack`](./skills/setup-pstack/SKILL.md)。它检测你能使用的模型，并写入一条始终生效的小规则，把每个角色（代码、判断、评审组）映射到一个模型。每个技能都会读取它，当规则不存在时回退到合理的默认值，所以你只需覆盖你想要的部分。

当某个默认值发生变化时，在变化之前写入的规则仍会固定旧的默认值，直到你再次运行 `/setup-pstack`。这次运行会替换它认识的旧默认值，并保留模型与默认值不同的其他角色。要丢弃一个它不认识的固定项，先删除该角色的那一行或整个文件。

## 自动化

pstack 还附带一个休眠的 [benny automation pack](./automations/benny/)。benny 会对 slack 上报告的问题进行分流，然后用真实的 UI 证据复现并修复确认的 bug。它的文件没有注册为 slash 技能。

要设置它，让 cursor 指向 [`FOR_AGENTS.md`](./automations/benny/FOR_AGENTS.md)。设置过程会把这个包复制到目标仓库的 `.cursor/automations/benny/`，在那里启用 pstack 以共享技能，并把用户配置放在复制的包之外。benny 运行在 cursor automations 上，因此仅适用于 cursor。

## 许可证

MIT
