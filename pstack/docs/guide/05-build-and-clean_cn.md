# 构建改动并清理 diff

构建行动手册都遵循同一套准则：说出你观察到的情况，让行动手册来索要证据。本页展示每种常见构建任务该在提示词里写什么，再介绍让 diff 保持可评审的清理习惯。

## 把你知道的情况写进每个构建行动手册的提示词

bug 提示词先说明症状，并要求先复现：

```text
/poteto-mode 这条命令重试后会吐出两条记录。先复现，再修复并验证。
```

feature 提示词说明要实现的行为，以及哪些东西不能变：

```text
/poteto-mode 加一个 --json 标志。文本输出要保持逐字节一致。两种形式都要验证。
```

重构提示词在改动结构前先锁定行为：

```text
/poteto-mode 把解析逻辑挪到一个模块里，行为零变化。先记录当前输出，再证明改完之后输出没变。
```

性能提示词说的是测量数据，不是主观印象：

```text
/poteto-mode 在这个测试样本上启动要 1.8 秒。追踪一下，修复测量出来的原因，把前后对比给我看。
```

这些提示词各自会路由到对应的行动手册（[Bug 修复](../../skills/poteto-mode/playbooks/bug-fix.md)、[功能](../../skills/poteto-mode/playbooks/feature.md)、[重构](../../skills/poteto-mode/playbooks/refactoring.md)、[性能问题](../../skills/poteto-mode/playbooks/perf-issue.md)），行动手册会补上你没写的步骤：修复前先复现，实现前先说清数据结构，重构前先锁定行为，优化前先做性能分析。

如果想持续改进某一个数字，可以用 [Hillclimb 行动手册](../../skills/poteto-mode/playbooks/hillclimb.md)。给它指标、目标和尝试次数下限，它就会用一套冻结的测量框架，一次循环验证一个假设。它只保留有效的改进，其余的全部回退。

两个性能行动手册都会用 [`/benchmark-checklist`](../../skills/benchmark-checklist/SKILL.md) 审查各自的数字。Perf issue 会审查它的基线以及后续每个数字，Hillclimb 会在冻结测量框架前先审查它。[验证并发布](./06-verify-and-ship.md#vet-a-measured-number-with-benchmark-checklist) 会告诉你什么时候该自己手动敲这个命令。

有时候你想先找到原因，再谈修复。对于正在发生的症状，比如内存泄漏、CPU 空转或界面异常，可以用 [Runtime forensics 行动手册](../../skills/poteto-mode/playbooks/runtime-forensics.md) 对运行中的进程插桩。对于已经抓好的性能快照，可以用 [Trace forensics 行动手册](../../skills/poteto-mode/playbooks/trace-forensics.md) 读取这份产物，把热点帧映射到源码。两者都只给出诊断，不做修复：

```text
/poteto-mode 这是一份启动缓慢的 cpuprofile。告诉我时间都花在哪了，是哪些源码行造成的。先别修复。
```

## 用 `/tdd` 先写失败的测试

当某个 bug 有低成本的本地测试路径时，整个提示词可以只有两个词：

```text
/tdd 实现
```

有上下文时，这两个词就够了。[`/tdd`](../../skills/tdd/SKILL.md) 会先写一个最小的、因预期原因而失败的测试，再写修复，然后重跑测试。如果写测试需要大范围搭建测试框架，或者要依赖脆弱的 mock，技能会如实说明，并改用最接近的可执行校验。当真实命令是更有力的证据时，别硬凑一个测试。

## 按名称加载 TypeScript 规则

[`typescript-best-practices`](../../skills/typescript-best-practices/SKILL.md) 把类型系统的原则转成具体规则：可辨识联合、边界处用 `unknown`、穷尽的变体、由 schema 推导的类型。它不会自动加载，所以当任务涉及 `.ts` 或 `.tsx` 文件时，手动输入 `/typescript-best-practices`。

## 提交前先清理

[Opening a PR 行动手册](../../skills/poteto-mode/playbooks/opening-a-pr.md) 会在每次提交前对 diff 运行 `/deslop`，poteto-mode 则把 [`/unslop`](../../skills/unslop/SKILL.md) 用在 PR 描述和提交正文上。`/deslop` 随 `cursor-team-kit` 插件提供，不在 pstack 里。如果你没有它，就用大白话要求同样的结果：去掉旁白式注释、没有依据的防御性判断、失效的兼容路径，以及无关的改动。

对于文字，`/unslop` 接受一个目标，外加你补充的规则：

```text
/unslop 处理一下 readme 的改动，别用破折号
```

你会慢慢形成自己的简写。像 `unslop 一下，收紧点` 这样简短的提示词，技能也能准确读懂意图。

## 用 `/no-comments` 去掉注释

注释需要单独过一遍，而且不能由写注释的智能体来做。作者会像你维护自己的注释一样，维护它写的注释。所以在评审前，把注释交给陌生的眼睛来看：

```text
/no-comments 处理这个 diff
```

[`/no-comments`](../../skills/no-comments/SKILL.md) 会派生出 [Comment Sicko](../../agents/comment-sicko.md)，一个只读评审者，它的保留清单很短：许可证头、公共 API 上的文档注释、解释代码做不到的地方的链接，以及由你无法改造的外部依赖所决定的行为。其余的一律删掉。你自己代码里冒出来的意外，可没这种待遇。注释会被当成重构标记反馈回来，`/no-comments` 会从根源上修复它认可的标记。当注释声称某种约束——"勿删"——时，技能会提议把这条说法编码成类型、测试或 lint 规则。不管怎样，注释最终都会被删掉。

这套分工值得理清楚：`/deslop` 清理代码里的冗余，`/unslop` 清理文字里的冗余，`/no-comments` 则把注释交给没写过它们的人来评审。

**常见误区：** 清理不是可选的锦上添花。带着旁白式注释和防御性冗余的 diff，在评审者眼里就像是没做完，而多出来的代码正是下一个 bug 藏身的地方。如果 diff 看着注了水，就在提交前说一句 `deslop it`，别等评审指出来才动手。

下一篇：[验证并发布](./06-verify-and-ship.md)。
