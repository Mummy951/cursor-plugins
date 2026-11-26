# 值得复制的提示词

替换成真实的路径、技能和完成检查项。非正式措辞即可。

## 理解

- `/poteto-mode read <thread>. 用你自己的话、以简单直白的英语重述根本问题。`
- `/in-your-own-words <long or messy ask>`
- `/sharpen <the prompt you're about to send>`
- `/poteto-mode investigate why <symptom>. 告诉我我们所知道的、你使用的数据，以及你最好的假设。暂时不要改动任何代码。`
- `用 /how 理解 <subsystem>。然后用 /why 找出它最近为什么会坏。`
- `/recall 我上周在 <topic> 上的工作，然后阅读 <issue>。`
- `/teach me 为什么你用这种方式实现，而不是 <other way>。你做了什么取舍？`
- `/poteto-mode 接管这个分支。阅读决策日志，找出已完成的部分，然后继续。不要重做已经完成的工作。`

## 构建

- Bug：`/poteto-mode <symptom>. 先复现，再修复并验证。`
- 应用中的 Bug：`/poteto-mode 用 /verify-<app> 复现这个问题。如果在 main 上能复现，就修复它，并给我一段视频作为证据。`
- 有廉价测试路径的 Bug：`/poteto-mode 先复现 <bug>。如果有廉价的测试路径，就用 /tdd 处理。然后修复并重跑。`
- 功能：`/poteto-mode 添加 <behavior>。<current output> 保持逐字节一致。两者都要验证。`
- 重构：`/poteto-mode 把 <code> 移到一个模块中，行为零变化。先记录当前输出，并证明之后它没有变化。`
- 性能：`/poteto-mode <operation> 在 <fixture> 上耗时 <time>。追踪它，修复测得的成因，给我看前后对比。`

## 设计与规划

- `/poteto-mode 为 <feature> 做几个原型方案。截图或录视频给我对比。`
- `/poteto-mode 我们需要 <feature>。先用 /architect 设计，并用原型回答悬而未决的问题。在继续之前让我审阅。`
- `/poteto-mode 先写一篇关于如何使用 <new package> 的教程。然后用 /teach me 讲讲为什么它比当前的更好。`
- `请 /arena 对这个线程和我们的方法给出第二意见。`
- `/poteto-mode 把这个设计变成计划。拆成小的、可验证的 PR，每个都有自己的验证步骤。`
- `/poteto-mode 规划将 <library> 迁移到 <target>。拆成小的、可验证的 PR。结果必须与原始版本完全一致，包括 bug。`

## 审查与发布

- `/interrogate 整个分支，但要持怀疑态度。暂时不要改动任何东西。除非是真正的 bug 或回归，否则不要吹毛求疵。` 也请阅读 Dropped 列表。
- `/create-formal-verification-skill 证明 <property> 在 <module> 中成立，即使 <steps> 交错或重试。` 在该代码合并或重构之后：`/maintain-formal-verification-skill`。
- `/swarm 检查 <dir> 下的每个包是否符合其检查脚本。每个包一个 worker。一份报告。`
- `/poteto-mode 打开 pr。提交要小而有序，描述中要有证据。`
- `/poteto-mode 照看这个 pr。让它变绿。` 仅查看状态时用：`/poteto-mode check on pr <number>. 还有未完成的事项吗？`
- `/poteto-mode 合并整个 PR 堆栈。`

## 离开与返回

- `/poteto-mode 我要睡觉了。在基于 <base> 的全新 worktree 中完成 <goal>。完成的标准是 <checks>。保留决策日志。提交前不要问我。/loop 直到完成。如果几个小时后确实卡住了，就停下来写下原因。`
- `/show-me-your-work 跟我讲讲你昨晚做了什么。` 先阅读它的 Attention 章节。
- `/ketchup` 会汇总这次聊天中自你上条消息以来的内容，并列出所有正在等你的调用。
- `/poteto-mode 在这个队列上完全自动驾驶。每一项都是独立的。`
- `/poteto-mode 自动处理这些改动，但不要合并。我自己来合并。`
- `/reflect 记录我们学到的东西，让下一次运行不再重复。` 只批准那些会改变未来决策的编辑。
- `/bro` 用大白话复述上一条回复。
