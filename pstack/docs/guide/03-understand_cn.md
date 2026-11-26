# 在改动代码之前先理解它

编辑你并不理解的代码，正是细微回归被发布出去的原因，这对 agent 和对你的影响都一样。agent 通常以两种方式之一失败。它们误读了你的意图，或者它们没有足够上下文把工作做对。[What goes in a prompt](./02-poteto-mode.md#what-goes-in-a-prompt) 解决第一种情况。本页解决第二种情况。

pstack 为你提供了四种入手方式。`/how` 解释代码现在做了什么。`/why` 挖掘它为何被写成这样。`/teach` 把两者融合成一份解释。`/recall` 重建你自己关于某个主题的最近上下文。每一种方式还会让 agent 用你能核对的语言解释它自己。这就是你监督一个可能比你更懂代码的 agent 的方法。

![A detective studies a machine blueprint with a magnifying glass while robots fetch case files; the evidence board behind her links clues under /how and /why.](./images/understanding.jpg)

## 从只读调查开始

当原因还不清楚时，要的是调查结论，而不是修复方案：

```text
/poteto-mode investigate why background jobs time out every few hours. give me what we know, what data you used, and your best hypotheses. don't change any code yet.
```

"don't change any code yet" 会把此请求路由到 [Investigation playbook](../../skills/poteto-mode/playbooks/investigation.md)。它会追踪代码，针对动机方面的问题加上 `/why`，并返回一份带引用的解释。如果是在多个选项之间做选择，它会返回一份带权衡表的推荐。询问 "what data you used" 会让 agent 把它的证据和猜测分开。当调查结论指向某个修复时，就把这个修复作为一个新任务启动。

## 用 `/how` 追踪行为

```text
/how do we dedupe notifications? is there an n+1 when we look up subscribers?
```

问你真正想问的问题。[`/how`](../../skills/how/SKILL.md) 会阅读代码，并以一位资深工程师带你熟悉该子系统的水平来回答，给出运行时流程、关键类型以及不明显的地方。它会启动一个只读子 agent 去探索代码并写出解释。

## 用 `/why` 挖掘历史

```text
/why was the retry limit set to five? does the reason still hold?
```

[`/why`](../../skills/why/SKILL.md) 从记录而非代码中寻找答案。它阅读引入该代码的提交和 PR 讨论，以及之后的每一次改动，并在线索单薄时查看其他来源。答案会引用作者们写下的内容，把 agent 自己的推断标注为推断，并说明记录没有回答的部分，因为"没人写下原因"本身就是一个答案。

这两者可以自然地组合。当你怀疑历史能解释这团乱麻时，`do why first then how` 是个不错的提示。

## 用 `/teach` 真正搞懂它

```text
/teach me how this PR changes retries. convince me it fixes the cause and not the symptom.
```

当一份摘要还不够时，就用 [`/teach`](../../skills/teach/SKILL.md)。它会运行 `/how` 和 `/why`（对于很小的改动，也许只运行其中之一），把调查结论编织成一份平实的解释，并通过一张张图逐步展开。"convince me" 这个说法值得复用。它把解释变成一场你可以检验的论证，而不是一次导览。

它同样适用于 agent 自己的选择：

```text
/teach me why you implemented it this way and not with a queue. what did you trade off, and why?
```

讲解对 agent 的帮助不亚于对你的帮助。一个必须解释自己工作的 agent，不得不去读代码，并用证据支撑每一处论断，而不是自信地断言然后继续往下走。

## 用 `/recall` 重建你自己的上下文

```text
/recall catch me up on the export work from last week
```

[`/recall`](../../skills/recall/SKILL.md) 会挖掘你自己的最近聊天记录以及共享记录（issue、此前的修复、仍在触发的错误），并交回一份关于事情进展和下一步的简报。你的旧聊天记录里保存着新 agent 所缺少的上下文，所以在一个旧主题上开展新工作时，先用它加载上下文，再交付新的输入：

```text
/recall my work on the virtualized list from yesterday, then read this bug report.
```

如果你想恢复某一个具体的聊天，那是下面的 Session pickup playbook，而不是 `/recall`。

## 用 Session pickup 接手先前的工作

当另一个 agent（或者上周的你）把一个分支做到一半就搁下时：

```text
/poteto-mode take over this branch. read the decision log, figure out what's done, and continue from there. don't redo finished work.
```

[Session pickup playbook](../../skills/poteto-mode/playbooks/session-pickup.md) 把先前的轨迹视为权威。它重建分支状态和各项决策，指出恢复点，并对照原始目标核验继承而来的论断，而不是从头重新推导一切。

**陷阱：** 不要因为"agent 反正会去读代码"就跳过本页的这些技能。一个没有追踪过的模型就开始编辑的 agent，往往会在第一个看起来合理的位置去修复症状。先做 `/how` 比修第二个 bug 更划算。

下一步：[Design the change](./04-design.md)。
