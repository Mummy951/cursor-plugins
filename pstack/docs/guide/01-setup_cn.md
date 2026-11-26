# 设置 pstack

在本页中，你将安装插件、选择 pstack 使用的模型，并运行你的第一个任务。设置过程只需一条命令和一段简短的对话。

## 安装插件

在 Cursor 聊天中，运行：

```text
/add-plugin pstack
```

Cursor 会确认插件已安装。

在 Claude Code 中，运行 `/plugin marketplace add cursor/plugins`，然后运行 `/plugin install pstack@cursor-plugins`。Claude Code 会为插件命令添加前缀，因此本指南中的每条命令在那里都需要加上 `pstack:` 前缀，例如 `/pstack:setup-pstack`，它会将模型规则写入 `~/.claude/rules/pstack-models.md`。[Claude Code 适配器](../harness/claude-code.md)介绍了其他差异。

## 选择你的模型

运行：

```text
/setup-pstack
```

[`/setup-pstack`](../../skills/setup-pstack/SKILL.md) 会检测你有权使用的模型，询问推理预算，向你展示每个角色（代码委派、判断、评审小组），并询问你想要什么。回答问题即可。它会写入 `~/.cursor/rules/pstack-models.mdc`，这是一条每个 pstack skill 都会读取的小规则。

默认值与 `large` 预算相同。代码、散文和判断在 Opus 5.5 上以 `xhigh` 推理运行，评审小组和 swarm workers 以 `high` 运行。没有任何角色会高于 `xhigh` 运行。在评审小组之外，Fable 5.2 仅当你为某个任务点名它，或同一份简报的两次 Opus 5.5 运行都未通过其检查时才会运行。`medium` 会将每个角色降低一级，`small` 降低两级，两者都消耗更少的 token。

你只需覆盖你关心的内容。规则中某一角色没有对应行，就保留该 skill 的默认值。要恢复默认值，删除该角色的行即可。重新运行 `/setup-pstack` 会保留任何模型与默认值不同的角色。当默认值发生变化时，更改前写入的规则仍会固定旧默认值，直到你再次运行 `/setup-pstack`。该运行会替换它识别出的旧默认值，告诉你哪些被替换，并保留你的其他选择。要取消固定某个它无法识别的默认值，先删除该角色的行或文件。

你可能在想如果使用 Auto 会发生什么。将某个角色设为 `inherit-parent` 或 `auto`，pstack 会省略子代理的 `model` 字段，这样子代理就继承你父聊天的模型。这两个值含义相同，且都不是模型 slug。对于小组角色，该值是一个列表，每个条目运行一个子代理，因此列表长度决定小组规模。设置还会配置 `swarm workers`，即每个 `/swarm` worker 的默认模型，除非某场比赛为每个分支指定了模型。

## 接受验证提议，或不接受

在设置结束时，`/setup-pstack` 会在你的项目中寻找一种证明应用行为的方法，可能是一个 `verify-*` skill 或一个现有的 harness。如果两者都找不到，它会提供一次机会，用 [`/create-verification-skill`](../../skills/create-verification-skill/SKILL.md) 生成一个。

同意的话，它会写入 `.cursor/skills/verify-<app>/`，这是一个项目本地 skill，教代理像用户一样驱动你的应用。在交付之前，它会验证该 skill 可用一次。拒绝的话，设置会继续。你可以随时自己运行 `/create-verification-skill`。[验证并交付](./06-verify-and-ship.md#create-a-project-verification-skill)对此有深入介绍。

如果你是 pstack 新手，选择同意。一个能检查自己工作的代理会持续运行，直到检查通过。而不能检查的代理会把每个结果都交回给你手动检查。在本指南的所有内容中，验证 skill 的回报最大。

然后，设置会提供一次机会，用 [`/create-formal-verification-skill`](../../skills/create-formal-verification-skill/SKILL.md) 生成一个正式的验证 skill，除非项目已经有一个。测试检查的是有人想到要写的用例。正式验证 skill 会检查代码的一个关键属性，例如"一个任务不会同时运行两次"，在既定范围内的所有情况下都检查。当你的代码涉及并发、重试、状态机或解析器时选择同意，因为这类情况下 bug 代价高昂且测试很少触及出问题的用例。否则选择拒绝，如果情况有变再稍后运行。[验证并交付](./06-verify-and-ship.md#prove-a-property-with-a-formal-verification-skill)对此有介绍。

设置完成后，开启一个新聊天。模型规则适用于新会话。

## 控制成本

pstack 会在子代理和评审小组上花费额外的 token。这是严谨的代价。要减少花费：

- 重新运行 `/setup-pstack`，选择更小的推理预算或更便宜的模型。在主聊天中使用强模型，在代码角色中使用更便宜、更快的模型，这是一个不错的组合。
- 将某个角色设为 `auto` 或 `inherit-parent`，使其运行在聊天自身的模型上。
- 缩短小组列表。每个条目运行一个子代理。
- 把 `/poteto-mode` 留给需要严谨的工作。小而明显的改动不需要。

## 运行你的第一个任务

挑选一个真实但很小的任务，像你向同事描述它那样描述它：

```text
/poteto-mode add a --json flag to this command. text output stays byte-identical. verify both.
```

观察待办事项列表。它的前几项是复制进来的匹配 playbook 的步骤，对于这条提示来说是 Feature playbook。

从这里你可以输入正常的后续内容。要在整个聊天中保持 `/poteto-mode` 开启，请从 `/` 菜单中用 Option+Enter（Mac）或 Alt+Enter（Windows）选择它，而不是按 Enter。这会使其成为 [Custom Mode](https://cursor.com/docs/skills#using-a-skill-as-a-custom-mode)，它在每一轮都保留在上下文中，直到你退出它。Custom Modes 在 Agents Window 和 CLI 中可用。直接按 Enter 会将该 skill 附加到一条消息上，并随着聊天推进而淡出。

下一步：[通过 `/poteto-mode` 分派工作](./02-poteto-mode.md)。
