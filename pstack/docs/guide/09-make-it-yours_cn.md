# 让它成为你的

poteto-mode 是一个人的风格。它底下的 playbook、路由和模型角色，用在你自己的风格上一样好用。这一页讲如何生成个人 mode、从一次会话中提炼经验、修复仓库让 agent 不再重复犯错、编写一个专注的 skill，以及在信任一次 skill 改动之前先测试它。

起步要比你想象的更小。第一天你不需要很多 skill，甚至不需要这整个插件。用直白的提示词，观察 agent 在哪里失败，当同样的失败出现两次时，再加一个 skill 或一项检查。

## 用 `/automate-me` 生成你自己的 mode

```text
/automate-me
```

你不需要描述自己的风格，因为 [`/automate-me`](../../skills/automate-me/SKILL.md) 会从你的历史记录里读出来。它在当前工作区中挖掘你最近的对话记录，找出反复出现的偏好——包括你喜欢的回复方式、委派、验证、代码、行文和流程，然后问你哪些模式才是真正的你。它通过 Cursor 内置的 `create-skill` 流程起草 `.cursor/skills/<your-name>-mode/SKILL.md`，把草稿跑一遍 [`/unslop`](../../skills/unslop/SKILL.md)，然后从一个 worktree 开一个 PR，让你像审查其他改动一样审查它。

当你的习惯发生变化时，再跑一次：

```text
/automate-me update my mode skill with everything since its last edit
```

更新模式只挖掘该 skill 上次改动以来的历史。它保留你还没有推翻的规则，根据新证据修订那些规则，并且只在出现新模式时才添加章节。

## 用 `/reflect` 捕捉一次会话的经验教训

在一个让你有所收获的任务结束后，立刻运行：

```text
/reflect that took way too long. capture what we learned so the next run doesn't repeat it.
```

[`/reflect`](../../skills/reflect/SKILL.md) 读取对话记录，只保留那些会改变未来 agent 工作的经验教训，并把提议归类到 `Accepted`、`Rejected` 和 `Backlog` 中。在任何 skill 改动之前，它都会等你的批准。只有当一条提议会改变未来的决策时才批准它。一次奇怪的会话只是个轶事，不是一条规则。

## 用 `/correct` 修复环境

当你一遍又一遍地纠正 agent 的同一个错误时，修复应该放在仓库里，而不是你的下一个提示词里。按这些方案的牢固程度排序：

1. 用架构或更好的数据结构让这个错误根本不可能发生。
2. 用类型，或者用一条错误信息直接点明修复方法的 lint 或 CI 检查来阻拦它。
3. 用一个测试捕获它。
4. 把它写成文档或 agent 规则。agent 跳过一条规则不会有任何失败，所以这个排最后。

人工审查不在列表上。一个必须在每个 PR 上都抓出同一个错误的审查者，正是这个方案要解决的问题。[`/correct`](../../skills/correct/SKILL.md) 来完成这项工作：

```text
/correct agents keep calling the database client directly instead of going through the repository layer
```

它读取最近的提交、回滚、审查意见，以及解释变通方法的注释，然后把错误归类。一类错误出现两次就算成立。它以每个提交修复一类的频率，在最高有效层级上修复最常见的那些类别，并证明每一条新检查都能在一个真实的过去错误上失败。它还在 agent 指令文件里维护一张表，把每条规则和强制执行它的东西配对，这样一条无人执行的规则就会作为重复项显现出来。回复会列出每一类及其证据、所选层级，以及为什么更高层级行不通。

不带参数运行它，它会自己从历史中找出这些类别。`/reflect` 和 `/correct` 分工不同。`/reflect` 从一次会话中改进 skill。`/correct` 改动仓库，让某一类错误无法再回来。当修复是一个新的边界时，把它和 `/architect` 搭配使用。[Run many projects in parallel](./07-overnight.md#run-many-projects-in-parallel) 里有一个提示词，可以在整个仓库范围内同时做这两件事。

## 编写一个专注的 skill

当你已经清楚想要捕捉的工作流时：

```text
/poteto-mode write a skill for verifying database migrations in this repo
```

编写 skill 会遵循 [Authoring or modifying a skill playbook](../../skills/poteto-mode/playbooks/authoring-a-skill.md)，它经由 Cursor 内置的 `create-skill` 路由，校验 frontmatter 和链接，并通过 Opening a PR playbook 交付结果。面向 agent 的行文比面向人的行文标准更高，因为一句无用的话会变成未来某个 agent 遵循的指令。让 playbook 来守住这个标准，而不是凭空手写 `SKILL.md`。

有两类特殊情况有各自的生成器。必须驱动你的应用并证明行为的 skill 是验证 skill，所以改用 [`/create-verification-skill`](../../skills/create-verification-skill/SKILL.md) 和 [`/maintain-verification-skill`](../../skills/maintain-verification-skill/SKILL.md)。用模型检查器、证明或其他验证器来检查代码某项属性的 skill 是形式化验证 skill，所以使用 [`/create-formal-verification-skill`](../../skills/create-formal-verification-skill/SKILL.md) 和 [`/maintain-formal-verification-skill`](../../skills/maintain-formal-verification-skill/SKILL.md)。[Verify and ship](./06-verify-and-ship.md#prove-a-property-with-a-formal-verification-skill) 涵盖这两类。

## 用 `/technical-writing` 按标准写文档

skill 不是你交付的唯一文字。对于文档、RFC、readme、PR 描述和提交信息：

```text
/technical-writing review the readme changes
```

[`/technical-writing`](../../skills/technical-writing/SKILL.md) 应用一套分层标准。它的目标是让疲惫的工程师第一遍就能读懂的文字。它先确定文档的模式（教程、操作指南、参考或解释），然后逐句处理：谁做什么，一句一个想法，不让任何内容有两种读法。用它来审查你或 agent 刚写的东西，或者在你要文档时提前点名它。

## 盲测 skill 改动

一次 skill 编辑会影响未来每一次会话，所以要像对待实验一样测试它。采用别人的 skill 也一样。在保留它之前，先确认它让你工作里的 agent 变得更好。

```text
/poteto-mode run the eval playbook on this skill change. same task for both variants, candidates stay blind.
```

当一个 skill 老是达不到目标，而你知道它应该做什么时，在一个任务里同时改动并测试它：

```text
/poteto-mode update the review skill so it flags missing migrations, and eval the change.
```

提前要求 eval 能让改动保持诚实。根据一次糟糕会话写出的修复往往过度拟合那次会话，经过多次编辑后 skill 就会漂移。eval 在它交付之前就能捕捉到这种漂移。

[Eval playbook](../../skills/poteto-mode/playbooks/eval.md) 围绕一种失败模式构建：观察者效应。知道自己正被评估的 agent 表现会不一样。所以候选 agent 会在经过清理的目录里拿到一个看起来自然的任务，永远不会出现 "eval" 或 "candidate" 这样的字眼，也永远不会知道彼此的存在。一位裁判在中性标签下给所有输出打分，而链式遵循的评分依据是每个候选实际读取了哪些文件，而不是它声称读了什么。

在接受裁决之前，自己读完每一份输出。如果你不同意裁判，先怀疑评分标准，而不是怀疑你的判断。

## 用 `/make-bot-ui` 构建 bot UI

有一个情境专属的 skill 面向 Grok Bot 用户。[`/make-bot-ui`](../../skills/make-bot-ui/SKILL.md) 构建一个小页面，其按钮通过一个 webhook 例程唤醒 bot。例如，你可以在审查队列里滑动，每一次滑动都让 bot 对该条目采取行动。你机器上的一台服务器持有 webhook 的发送方密钥，所以这个密钥永远不会到达浏览器或聊天窗口。该 skill 还涵盖如何在 Tailscale 上暴露这个页面。

**陷阱：** 不要因为一个 skill 表现异常就在任务中途编辑它。在它自己的 PR 里修好它，让任务继续推进。一份纠缠在功能开发里的 skill 改动对审查不可见，也无法评估。

下一篇：[Recipes and pitfalls](./10-recipes-and-pitfalls.md)。
