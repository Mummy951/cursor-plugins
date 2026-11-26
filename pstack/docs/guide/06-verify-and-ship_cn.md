# 验证结果并开启 PR

"能编译"并不是证据。[Prove It Works 原则](../../skills/principle-prove-it-works/SKILL.md)要求智能体在报告成功之前检查真实的产物，而你的职责就是让"真实产物"变得可检查。本页涵盖如何陈述完成条件、如何审查一个测得的数字、如何为你的应用生成验证技能、如何正式验证一个测试很少触及的属性、如何开启 PR，以及如何推动它直到合并。

在大多数智能体工作中，验证是最慢的一步，因为它通常是需要等待人类的一步。让智能体能够自己完成验证，你就不再是瓶颈。跳过它，运行更多的智能体只会带来更多未经检查、等待你审阅的工作。

![一架原型飞机飞过真实的测试航线，她用秒表计时，机器人拍摄并记录运行过程；终端显示 verify: pass, evidence: captured。](./images/verification.jpg)

## 预先陈述完成条件

把"完成"意味着什么写进第一个提示词，用任何合适的措辞：

```text
/poteto-mode add json output to this command. text output stays byte-identical, the json parses, both run against the sample project. show me the evidence.
```

这样智能体就有了三项可以运行的检查，而不是一种需要迎合的情绪。当回复回来时，它应该带着确切的命令和输出。如果某项检查无法运行，一个好的回复会说"inconclusive"（无定论）。一个自信却没有证据的回复是一个警示信号。

让检查与变更相匹配：

- CLI 变更要运行真实的命令。
- UI 变更要在运行中的应用里走一遍变更过的流程。当它必须与参照物逐像素一致时，[Visual parity playbook](../../skills/poteto-mode/playbooks/visual-parity.md) 会把截图与冻结的基线做对比，而不是凭肉眼判断。
- 解析器或迁移要重放一份保存的输入。
- 性能变更要对比前后 profile。
- 存储变更要回读写入的值。

把证明当作一份你可以亲自检查的产物来索取：失败的测试然后是成功的测试、前后对比视频、trace、截图。如果修复已经合并，就在 main 上再要求同样的检查。一份产物胜过一套听起来合理的解释，因为你可以质疑它，而不必重放整个运行过程。

对于一处你并不完全信任的小改动，[`/blast-radius`](../../skills/blast-radius/SKILL.md) 会找出它可能在别处破坏什么。它找出那个使变更安全的关键事实，并通过运行代码来证明这个事实，而不是靠争论。

## 用 `/benchmark-checklist` 审查测得的数字

前后数字是最容易在无意中出错的证据。一个热缓存、一侧是 debug 构建，或者根本没有在计时区间内运行的工作，都可能各自产生一个令人信服的加速效果。在报告一个数字或据此行动之前，输入：

```text
/benchmark-checklist vet the export speedup before it goes in the pr
```

[`/benchmark-checklist`](../../skills/benchmark-checklist/SKILL.md) 会问七个问题，并要求每一个都有来自一次运行的证据：

1. 是什么限制了这个数字，为什么它不是两倍？
2. 每一侧的调优方式是否都与生产环境一致？
3. 结果是否突破了物理限制，比如磁盘带宽或核心数？
4. 有没有任何东西报错或返回了错误的输出？
5. 它能否在交替运行中复现，并给出中位数和区间？
6. 它在端到端、在用户等待的那条路径上是否重要？
7. 这项工作是否真的发生在计时区间之内？

结论会以更快、更慢、无可测量差异或无定论返回，并附上运行次数、区间和限制因素。当它无法指出限制因素，或某一侧未经调优时，它会说无定论。`/poteto-mode` 已经在 Perf issue 和 Hillclimb playbook 内部运行了这个检查清单，所以当你在这两者之外测量了某样东西，或者别人的数字看起来好得不真实时，就自己输入它。它是 [Explain the Number 原则](../../skills/principle-explain-the-number/SKILL.md)的实操形式。

## 创建项目验证技能

上面 UI 那条里藏着一个真实的需求。智能体需要一种脚本化的方式来驱动你的应用。如果你的项目已经有，很好。如果没有，运行：

```text
/create-verification-skill
```

[`/create-verification-skill`](../../skills/create-verification-skill/SKILL.md) 会审视代码仓库，而不是审视你。它会弄清楚用户会触碰什么、应用如何在本地启动、什么可以驱动它（优先使用现有的 harness，否则用浏览器和 CDP、PTY，或纯 HTTP）、什么证据能证明行为，以及两个实例能否并行运行。它只问你代码无法回答的问题。

它会写入 `.cursor/skills/verify-<app>/`，其中包含面向智能体的说明，带有确切的 Launch、Doctor、Drive、Evidence 和 Cleanup 章节，一个 Helpers 章节用于它附带的任何脚本，此外还有一个位于 `features/` 下的功能地图，用于索引应用做什么，以及什么结果能证明每个功能正常。该技能附带一个[功能地图的完整示例](../../skills/create-verification-skill/references/feature-map-example/)，其中带有一个 README 索引，每个功能一个文件，使用四个必需的 H2。在交付之前，生成器会端到端地证明这个技能一次：启动、doctor 检查、驱动一个功能、捕获证据、清理，然后确认证据在清理后依然存在。如果这次证明失败，就不要使用其产物。

从那时起，"在应用中验证它"就成了任何智能体都能执行的步骤，在这个仓库里，无需任何设置对话。当你想让证明采用某种特定形式时，就在提示词中点名它：

```text
/poteto-mode build the bulk-archive action. use /verify-<app> to verify your changes and show me a video and screenshots as proof.
```

```text
/poteto-mode repro this with /verify-<app>. if it repros on main, fix it and show me a video as proof.
```

一旦验证技能可用，[`/swarm`](../../skills/swarm/SKILL.md) 就可以按功能地图条目拆分一整轮检查并聚合结果。一群验证器还能在足够大的样本上确认一次性能提升，或者在 PR 发布前对应用做模糊测试以发现回归。

把验证技能当作基础设施，而不是一次性用品。把它提交进仓库，让团队里的每个人、每个智能体都用同样的方式驱动应用。然后[打造杠杆](../../skills/principle-build-the-lever/SKILL.md)。当智能体不断编写一次性脚本来点击浏览应用时，就要求提供一个小型控制 CLI 供技能调用。智能体花费更少的 token，每次运行也变得可重复。一个被智能体良好使用的 CLI 具备这些特征：

- 少量可组合的命令，每条都做实事，而不是许多单薄的命令。
- 对任何破坏性操作都有 `--dry-run` 选项。
- 子命令逐步揭示功能，而不是一次性全部暴露。
- 错误信息告诉用户该怎么做，而不只是说哪里错了。
- 丰富的 `--help` 文本。
- 机器可读的输出，比如 JSON。

在你还在那里时，也让开发环境设置变得可重复：种子数据、测试用户，以及一条每次都以相同方式把环境带起来的命令。

## 保持验证技能诚实可信

应用会变化，功能地图会过时。至少每天运行一次，理想情况下通过定时任务自动化来运行，这样没人需要记着去做：

```text
/maintain-verification-skill
```

[`/maintain-verification-skill`](../../skills/maintain-verification-skill/SKILL.md) 会审查生成的技能：每个功能并行一个只读的源码读取器，然后进行一次实时的遍历，驱动每一个已映射的功能。它最终只会以三种结果之一结束。`clean` 表示覆盖完整、无需发布任何东西。`changed` 表示有一个经过验证的修正 PR，且仅限于验证技能自身的目录内。`blocked` 会指出阻塞项。它从不编辑产品代码。如果实时遍历捕捉到产品回归，它会报告该回归，而不是用一次文档修改把它掩盖掉。

## 用正式验证技能证明一个属性

验证技能证明的是你驱动的那些路径。有些 bug 存在于没人驱动的路径上：两次重叠的重试、在写入中途过期的锁、乱序到达的状态变更。测试很少触及这些情况。对于这类 bug 代价高昂的代码，运行：

```text
/create-formal-verification-skill
```

[`/create-formal-verification-skill`](../../skills/create-formal-verification-skill/SKILL.md) 会从代码中挑选目标：代码做出的一个承诺，破坏它会伤害用户，而测试很少触及破坏它的那种情况。它挑选最强的目标，而不是最容易验证的目标。你可以点名一个领域或一个 bug，让它从那里开始：

```text
/create-formal-verification-skill the job lease and retry logic. a job must never run twice at once.
```

问题决定工具。事件是否可能以坏的顺序发生，适合用模型检查器对一个小型设计模型进行建模，无论何种语言，TLA+ 都是这方面的默认选择。代码是否对所有输入都表现正确，适合用直接运行在代码上的验证器。如果你的语言没有可用的验证器，智能体会如实说明，并用生成的输入测试真实代码，而不是一个手写的心智模型。一个必须在任何规模下都成立的属性可能需要一个证明。智能体会告诉你它选择了哪种工具以及原因。

它会验证一个属性，并展示该属性在一个被破坏的代码变体上失败，因为你从未见过失败的检查证明不了任何东西。当当前代码已经破坏了该属性时，你会得到一个简短的反例，以你的代码术语表述为一系列步骤。该属性保持原样陈述，因此这个发现始终是一个发现。除非你要求，否则产品代码不会改变。

它会写入一个项目本地技能，告诉下一个智能体某次变更是否触及被验证的代码、如何用一条命令运行检查，以及如何区分真正的 bug 和建模错误。在交付之前，生成器会针对一个破坏该属性的临时改动，亲自按该技能走一遍，检查必须能捕捉到它。交付说明会写清楚验证了什么、用了什么工具和边界、检查发现了什么，以及它不覆盖什么。一次通过覆盖的是模型或边界，而不是整个系统。

该技能还会记录每条检查覆盖哪些代码，以及该检查上次与之匹配的提交。跳过该技能的合并和重构，可能让检查停留在对已经消失的代码描述的通过状态上。在这些之后，运行 [`/maintain-formal-verification-skill`](../../skills/maintain-formal-verification-skill/SKILL.md)。它会从代码更新验证、重新运行、展示每条更新后的检查仍然可以失败，并告诉你每次失败到底是验证中的漂移还是代码中的 bug。

## 用状态图展示行为

用模型术语表述的反例能说服你。而拥有该行为的 PM 或设计师需要用他们的术语来看它。运行：

```text
/statechart what the user sees when they edit a note offline
```

[`/statechart`](../../skills/statechart/SKILL.md) 会绘制一张用户所见内容的 Mermaid 状态图，由 TLA+ 模型生成，因此图与被检查的模型不会彼此漂移。读者的规则（"离线时你绝不会看到 Synced"）会变成不变量。图会标出每一个破坏规则的状态，报告会用读者的语言给出到达它的最短路径。在交付之前，智能体会在真实代码中复现每一条 bug 路径，这样建模错误就不会作为一个 bug 到达读者手中。

## 开启 PR

```text
/poteto-mode open the pr. small ordered commits, evidence in the description.
```

[Opening a PR playbook](../../skills/poteto-mode/playbooks/opening-a-pr.md) 从 worktree 开始工作，把工作 rebase 成小的、可落地的提交，清理 diff，写一份一分钟简报作为正文，并让 PR 准备就绪。堆叠的后续 PR 胜过不断膨胀的分支。

## 用 Babysit 推动 PR 达到可合并状态

一个打开的 PR 会立刻开始积累阻塞项。检查失败、审阅者留言、主干前进。把这些杂事交给 [Babysit playbook](../../skills/poteto-mode/playbooks/babysit.md)：

```text
/poteto-mode babysit this pr. get it green.
```

Babysit 会监视 PR，在可用时使用内置的观察器对接 GitHub 或 Origin CLI，并按顺序处理阻塞项：先冲突，再审阅讨论串，最后 CI。它从不 rebase 或强制推送。冲突或过期的基线会连同需要 rebase 的分支一起返回给你。评论分类持怀疑态度，因为人类和机器人会把真实的发现和噪音列在同一个清单里。真实的发现会得到修复，噪音则被驳回，并在讨论串上贴出反证。当你只想要状态时，问得更小一点，Babysit 会直接回答而不启动循环：

```text
/poteto-mode check on pr 123. anything outstanding?
```

Babysit 会在可合并状态时停下。它从不合并，即使一切都是绿的，因为合并是一个不同的决定。

## 用 Shipping 落地整个 PR 栈

绿并不等于安全。当你准备落地时，明确说出来：

```text
/poteto-mode land the stack.
```

[Shipping playbook](../../skills/poteto-mode/playbooks/shipping.md) 在布防任何东西之前，会独立验证每个 PR。每个 PR 由一个全新的智能体实时证明行为，判断变更的智能体绝不是写它的那个。然后 Shipping 只从底部开始落地连续且已验证的那一段，默认通过 GitHub 逐个 PR 进行，在 Origin CLI 可用时使用它，并报告第一个破坏链条的 PR。一个位于未验证 PR 之上的已验证 PR 会等待，因为合并它会把那个缺口拉到自己下面。

下一步：[Run work while you sleep](./07-overnight.md)。
