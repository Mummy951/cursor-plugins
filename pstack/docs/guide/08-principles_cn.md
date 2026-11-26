# 用原则名称来引导

pstack 附带 24 条原则，每条都作为独立技能发布。`/poteto-mode` 会在每个多步骤任务开始时读取它们的索引，应用任务触发的那些原则，并在回复中说出每条被应用的原则，以及它所改变的决策。

你不需要调用原则。你用它们的名称来引导。每个名称都指向一条智能体已经读过的完整规则，因此一句话比一整段指令更精准地改变工作方向。

## 实践中的引导

假设智能体正要给三个现有适配器再硬塞一个新适配器：

```text
use subtract before you add. delete the obsolete adapters first, then design what's left.
```

假设它因为构建通过就宣称成功：

```text
apply prove it works. run the real import flow and show me the written records.
```

假设两个并行的尝试即将写入同一个分支：

```text
separate before serializing shared state. give each attempt its own worktree, no locks.
```

每句话之所以奏效，是因为其背后的规则很具体。智能体仍然必须在回复中说明这条规则改变了哪个决策。引用了一条原则却没有任何决策支撑，意味着智能体只是说出了规则名称，并没有真正应用它。

## 简述这 24 条

核心原则决定要构建多少，以及何时重新思考设计：

- [Laziness Protocol（惰性协议）](../../skills/principle-laziness-protocol/SKILL.md) 倾向于删除，以及采用解决问题所需的最小改动。
- [Foundational Thinking（基础性思考）](../../skills/principle-foundational-thinking/SKILL.md) 在编写逻辑之前先选定核心数据结构。
- [Redesign from First Principles（从第一性原理重新设计）](../../skills/principle-redesign-from-first-principles/SKILL.md) 把新需求当作从第一天起就存在那样来整合。
- [Attack the Premise（攻击前提）](../../skills/principle-attack-the-premise/SKILL.md) 写下两次或多次失败修复所共有的前提，并在下一次修复前直接检验它。
- [Subtract Before You Add（先减后加）](../../skills/principle-subtract-before-you-add/SKILL.md) 在其之上构建之前，先移除无用的负担。
- [Minimize Reader Load（最小化读者负担）](../../skills/principle-minimize-reader-load/SKILL.md) 压缩读者必须在脑中记住的层级和隐藏状态。
- [Outcome-Oriented Execution（面向结果的执行）](../../skills/principle-outcome-oriented-execution/SKILL.md) 让重写收敛到目标设计，而不是保留一次性的兼容状态。
- [Experience First（体验优先）](../../skills/principle-experience-first/SKILL.md) 选择用户的结果，而非实现的便利。
- [Exhaust the Design Space（穷尽设计空间）](../../skills/principle-exhaust-the-design-space/SKILL.md) 在没有先例时，构建两到三个相互竞争的原型。
- [Build the Lever（打造杠杆）](../../skills/principle-build-the-lever/SKILL.md) 构建能完成或验证工作的脚本，让评审者可以重新运行它。当智能体不断手工重复同一件事时，让它写出它希望拥有的工具或技能。如果某个脚本能以相同方式每次都完成某个步骤，就用脚本，把智能体留给需要判断的决策。

架构原则决定状态、校验和兼容性位于何处：

- [Model the Domain（建模领域）](../../skills/principle-model-the-domain/SKILL.md) 把重复出现的规则编码到一个结构中，而不是散落的条件判断里。
- [Boundary Discipline（边界纪律）](../../skills/principle-boundary-discipline/SKILL.md) 在边界处校验，信任内部类型。
- [Type System Discipline（类型系统纪律）](../../skills/principle-type-system-discipline/SKILL.md) 让非法的状态无法被表示。
- [Make Operations Idempotent（让操作幂等）](../../skills/principle-make-operations-idempotent/SKILL.md) 让重试收敛到相同的最终状态。
- [Migrate Callers Then Delete Legacy APIs（先迁移调用方再删除遗留 API）](../../skills/principle-migrate-callers-then-delete-legacy-apis/SKILL.md) 一次性完成迁移与删除。
- [Separate Before Serializing Shared State（序列化共享状态前先分离）](../../skills/principle-separate-before-serializing-shared-state/SKILL.md) 在引入协调之前先消除共享。

验证原则定义什么才算证据：

- [Prove It Works（证明它有效）](../../skills/principle-prove-it-works/SKILL.md) 验证真实的产物，而非代理指标。
- [Fix Root Causes（修复根本原因）](../../skills/principle-fix-root-causes/SKILL.md) 在修改代码之前先复现并追溯到原因。
- [Sequence Work into Verifiable Units（把工作切分为可验证的单元）](../../skills/principle-sequence-verifiable-units/SKILL.md) 每个小单元都以一次检查结束，再开始下一个。
- [Test Behavior, Not Implementation（测试行为，而非实现）](../../skills/principle-test-behavior-not-implementation/SKILL.md) 断言代码使用者能观察到的东西，这样当该行为被破坏时测试就会失败，而内部实现改变时测试仍能通过。
- [Explain the Number（解释数字）](../../skills/principle-explain-the-number/SKILL.md) 在任何人信任或报告某个测量数字之前，说明是什么限制了它，并排除它测量的是别的东西。 [`/benchmark-checklist`](../../skills/benchmark-checklist/SKILL.md) 把它变成你要根据真实运行回答的七个问题。

委派原则让并行工作保持理智：

- [Guard the Context Window（守护上下文窗口）](../../skills/principle-guard-the-context-window/SKILL.md) 把批量读取路由给子智能体，把结论保留在主对话中。
- [Never Block on the Human（绝不被人类阻塞）](../../skills/principle-never-block-on-the-human/SKILL.md) 对可逆的工作继续推进，并呈现结果。

还有一条元原则：

- [Encode Lessons in Structure（把经验编码进结构）](../../skills/principle-encode-lessons-in-structure/SKILL.md) 把你已经重复过两次的建议变成一条 lint、检查或脚本。 [`/correct`](../../skills/correct/SKILL.md) 把它应用到整个仓库，如 [Make it yours](./09-make-it-yours.md#fix-the-environment-with-correct) 所示。

不要死记这个列表。现在快速浏览一遍，然后当你发现智能体正在做某个这里的名称本可避免的事情时，再回来看。词汇就是这样扎根的。

下一篇：[Make it yours](./09-make-it-yours.md)。
