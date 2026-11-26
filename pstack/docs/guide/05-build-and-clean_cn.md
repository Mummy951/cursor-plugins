# 构建改动并清理 diff

各个构建 playbook 遵循同一条准则：说出你观察到的现象，让 playbook 来要求证据。本页展示每种常见构建任务该在提示词里写什么，以及让 diff 保持可审查的清理习惯。

## 用你所掌握的信息提示每个构建 playbook

修复 bug 的提示词先陈述症状，并要求先复现：

```text
/poteto-mode this command emits two records after a retry. repro first, then fix and verify.
```

功能提示词陈述行为，以及哪些东西不能改变：

```text
/poteto-mode add a --json flag. text output stays byte-identical. verify both forms.
```

重构提示词在结构变动前先锁定行为：

```text
/poteto-mode move parsing into one module, zero behavior change. record the current output first and prove it's unchanged after.
```

性能提示词陈述测量数据，而非主观印象：

```text
/poteto-mode startup takes 1.8s on this fixture. trace it, fix the measured cause, show me before and after.
```

这些提示词各自路由到对应的 playbook（[Bug fix](../../skills/poteto-mode/playbooks/bug-fix.md)、[Feature](../../skills/poteto-mode/playbooks/feature.md)、[Refactoring](../../skills/poteto-mode/playbooks/refactoring.md)、[Perf issue](../../skills/poteto-mode/playbooks/perf-issue.md)），playbook 会补上你没写的步骤：修复前先复现、实现前先命名数据结构、重构前先锁定行为、优化前先做性能分析。

若要持续改进某一项数值，可以使用 [Hillclimb playbook](../../skills/poteto-mode/playbooks/hillclimb.md)。给它指标、目标，以及尝试次数下限，它会在冻结的测量装置下每次循环一个假设。它保留有效的改进，其余全部回退。

两个性能 playbook 都会对各自的数值运行 [`/benchmark-checklist`](../../skills/benchmark-checklist/SKILL.md)。Perf issue 会审查它的基线及其后的每个数值，Hillclimb 会在冻结测量装置前先审查该装置。[Verify and ship](./06-verify-and-ship.md#vet-a-measured-number-with-benchmark-checklist) 说明何时该自己输入它。

有时你希望在动手修复前先找到根因。对于仍在进行中的症状，例如内存泄漏、CPU 空转或视觉故障，[Runtime forensics playbook](../../skills/poteto-mode/playbooks/runtime-forensics.md) 会对运行中的进程做插桩。对于你已经采集好的 profile，[Trace forensics playbook](../../skills/poteto-mode/playbooks/trace-forensics.md) 会读取产物并把热点帧映射到源码。两者都只给出诊断，而不是修复：

```text
/poteto-mode here's a cpuprofile from the slow startup. tell me where the time goes and which source lines own it. no fix yet.
```

## 用 `/tdd` 先写失败的测试

当某个 bug 有低成本的本地测试路径时，整段提示词可以只有两个词：

```text
/tdd implement
```

在上下文里，这就够了。[`/tdd`](../../skills/tdd/SKILL.md) 会写出因预期原因而失败的最小测试，然后是修复，再重跑测试。如果某个测试需要大范围的测试装置搭建或脆弱的 mock，该 skill 会说明情况，并改用最接近的可执行检查。不要在真实命令能提供更有力证据的地方强行写测试。

## 按名称加载 TypeScript 规则

[`typescript-best-practices`](../../skills/typescript-best-practices/SKILL.md) 把类型系统原则转化为具体规则：可辨识联合类型、边界处的 `unknown`、穷尽的变体、由 schema 派生的类型。它不会自行加载，所以当任务涉及 `.ts` 或 `.tsx` 文件时，输入 `/typescript-best-practices`。

## 提交前先清理

[Opening a PR playbook](../../skills/poteto-mode/playbooks/opening-a-pr.md) 会在每次提交前对 diff 运行 `/deslop`，而 poteto-mode 会把 [`/unslop`](../../skills/unslop/SKILL.md) 应用于 PR 描述和提交正文。`/deslop` 随 `cursor-team-kit` 插件提供，而不在 pstack 中。如果你没有它，就用大白话要求同样的结果：删除叙述性注释、无依据的防御性检查、失效的兼容路径以及无关的改动。

对于散文文本，`/unslop` 接受一个目标以及你附加的任何规则：

```text
/unslop the readme changes, no emdashes
```

你会逐渐形成自己的简写。该 skill 能很好地从诸如 `unslop that, tighten it` 这样的简洁提示词中读懂意图。

## 用 `/no-comments` 去除注释

注释需要单独过一遍，而且不能由写它们的 agent 来做。作者对注释的维护会像你维护自己的注释一样。所以在审查前，把它们交给一双全新的眼睛：

```text
/no-comments the diff
```

[`/no-comments`](../../skills/no-comments/SKILL.md) 会派生出 [Comment Sicko](../../agents/comment-sicko.md)，一个只读的审查者，它有一份简短的保留清单：许可证头、公共 API 上的文档注释、解释代码本身无法解释之事的链接、由你无法重塑的外部依赖所强制的行为。其余一律删除。你自己代码里的意外之处可享受不到这种待遇。注释会以一个重构标记的形式返回，而 `/no-comments` 会从根源上修复它接受的标记。当某条注释声称某种约束——"do not remove"——该 skill 会提议把这一主张编码为类型、测试或 lint。无论哪种方式，注释最终都会被移除。

这种分工值得理清。`/deslop` 清理代码中的冗余，`/unslop` 清理散文文本中的冗余，而 `/no-comments` 把注释交给并非撰写它们的人来审查。

**陷阱：** 清理不是可选的润色。带着叙述性注释和防御性累赘的 diff，在审查者看来是未完成品，而多出来的代码正是下一个 bug 藏身之处。如果觉得 diff 显得注了水，就在提交前说 `deslop it`，而不是等审查指出之后。

下一步：[Verify and ship](./06-verify-and-ship.md)。
