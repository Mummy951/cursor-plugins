# 用原则名称来引导

pstack 将 24 条原则作为独立技能发布。`/poteto-mode` 在每项多步骤任务开始时读取这些原则的索引，应用任务触发的那些原则，并在回复中说出每条应用的原则，以及它改变了哪个决策。

你不用调用原则，而是用它们的名字来引导。每个名字都指向一条智能体已经读过的完整规则，所以一个短语比一整段指令更能精准地调整工作方向。

## 实际引导

假设智能体正准备往三个现有适配器上接一个新适配器：

```text
用「先减后加」：先删掉过时的适配器，再设计剩下的部分。
```

假设它因为构建通过就宣称成功：

```text
用「证明它可行」：跑一遍真实的导入流程，把写好的记录给我看。
```

假设两个并行的尝试就要写入同一个分支：

```text
用「共享状态先分离再串行化」：给每次尝试各自的工作树，不要加锁。
```

每个短语之所以管用，是因为它背后的规则很具体。智能体仍然必须在回复中说明，这条规则改变了哪个决策。引用了原则却没有任何决策跟着改变，就说明智能体只是说出了规则的名字，并没有真正应用它。

## 24 条原则速览

核心原则决定要构建多少，以及什么时候该重新审视设计：

- [惰性协议](../../skills/principle-laziness-protocol/SKILL.md)：优先删除，并采用能解决问题的最小改动。
- [基础思维](../../skills/principle-foundational-thinking/SKILL.md)：先确定核心数据结构，再写逻辑。
- [从第一性原理重新设计](../../skills/principle-redesign-from-first-principles/SKILL.md)：把新需求当作从一开始就存在那样整合进来。
- [质疑前提](../../skills/principle-attack-the-premise/SKILL.md)：写下两次或更多次失败修复所共有的前提，在下次修复前直接验证它。
- [先减后加](../../skills/principle-subtract-before-you-add/SKILL.md)：先去掉累赘，再在其之上构建。
- [减少读者负担](../../skills/principle-minimize-reader-load/SKILL.md)：压缩读者必须在脑中记住的层次和隐藏状态。
- [以结果导向执行](../../skills/principle-outcome-oriented-execution/SKILL.md)：让重写收敛到目标设计，而不是保留临时性的兼容状态。
- [体验优先](../../skills/principle-experience-first/SKILL.md)：把用户的结果放在实现的便利之上。
- [穷尽设计空间](../../skills/principle-exhaust-the-design-space/SKILL.md)：没有先例时，构建两三个相互竞争的原型。
- [打造杠杆](../../skills/principle-build-the-lever/SKILL.md)：构建能完成或证明工作的脚本，让评审者可以重新运行它。当智能体一直在手工做同一件事时，让它写出它想要的那个工具或技能。如果某个步骤脚本每次都能以同样的方式完成，就用脚本，把智能体留给需要判断的场合。

架构原则决定状态、校验和兼容性放在哪里：

- [为领域建模](../../skills/principle-model-the-domain/SKILL.md)：把反复出现的规则编码到一个结构中，而不是散落在各处的条件判断。
- [边界纪律](../../skills/principle-boundary-discipline/SKILL.md)：在边界处校验，信任内部类型。
- [类型系统纪律](../../skills/principle-type-system-discipline/SKILL.md)：让非法状态无法表示。
- [让操作幂等](../../skills/principle-make-operations-idempotent/SKILL.md)：让重试收敛到相同的最终状态。
- [先迁移调用方再删除遗留 API](../../skills/principle-migrate-callers-then-delete-legacy-apis/SKILL.md)：一次性完成迁移和删除。
- [共享状态先分离再串行化](../../skills/principle-separate-before-serializing-shared-state/SKILL.md)：在加入协调机制之前，先消除共享。

验证原则定义什么才算证据：

- [证明它可行](../../skills/principle-prove-it-works/SKILL.md)：验证真实的产物，而不是替代品。
- [修复根本原因](../../skills/principle-fix-root-causes/SKILL.md)：改代码之前先复现问题，并追踪到根因。
- [把工作切分为可校验的小单元](../../skills/principle-sequence-verifiable-units/SKILL.md)：每个小单元都以一次校验收尾，再开始下一个。
- [测试行为，而非实现](../../skills/principle-test-behavior-not-implementation/SKILL.md)：断言代码使用者能观察到的东西，这样当该行为被破坏时测试就会失败，而改动内部实现时测试仍能通过。
- [解释数字](../../skills/principle-explain-the-number/SKILL.md)：在任何人相信或报告某个测量数字之前，先说明是什么限制了它，并排除它测的是别的东西。[`/benchmark-checklist`](../../skills/benchmark-checklist/SKILL.md) 把它变成七个问题，你要用真实运行的结果来回答。

委派原则让并行工作保持条理：

- [守护上下文窗口](../../skills/principle-guard-the-context-window/SKILL.md)：把批量阅读交给子代理，把结论留在主对话中。
- [绝不阻塞人类](../../skills/principle-never-block-on-the-human/SKILL.md)：对可逆的工作直接推进，然后把结果呈现出来。

还有一条元原则：

- [把经验固化到结构中](../../skills/principle-encode-lessons-in-structure/SKILL.md)：把你说过两遍的建议变成 lint、校验或脚本。[`/correct`](../../skills/correct/SKILL.md) 会把它应用到整个仓库，见[把它变成你的](./09-make-it-yours.md#fix-the-environment-with-correct)。

不用背这份清单。现在先快速浏览一遍，之后一旦发现智能体做了某个名字本可以避免的事情，再回来查。这套说法就是这样记住的。

下一篇：[把它变成你的](./09-make-it-yours.md)。
