# 设置 pstack

在本页中，你会安装插件、选择 pstack 使用哪些模型，并运行第一个任务。整个设置过程就是一条命令加一段简短的对话。

## 安装插件

在 Cursor 对话中，运行：

```text
/add-plugin pstack
```

Cursor 会确认插件已安装。

在 Claude Code 中，先运行 `/plugin marketplace add cursor/plugins`，再运行 `/plugin install pstack@cursor-plugins`。Claude Code 会给插件命令加上前缀，因此本指南中的每条命令在那里都要带上 `pstack:` 前缀，例如 `/pstack:setup-pstack`，它会把模型规则写入 `~/.claude/rules/pstack-models.md`。[Claude Code 适配器](../harness/claude-code.md)介绍了其他差异。

## 选择模型

运行：

```text
/setup-pstack
```

[`/setup-pstack`](../../skills/setup-pstack/SKILL.md) 会检测你能使用的模型，询问推理预算，展示每个角色（代码委派、判断、评审组），然后问你想要什么。回答这些问题即可。它会写入 `~/.cursor/rules/pstack-models.mdc`，这是一条很小的规则，每个 pstack 技能都会读取它。

默认设置与 `large` 预算相同。代码、文稿和判断任务使用 Opus 5.5，`xhigh` 推理级别；评审组和 swarm worker 使用 `high` 级别。没有任何角色会高于 `xhigh`。在评审组之外，只有当你为某个任务明确指定 Fable 5.2，或者同一任务说明连续两次由 Opus 5.5 执行都未通过校验时，才会使用 Fable 5.2。`medium` 会把每个角色降低一级，`small` 降低两级，两者消耗的 token 都更少。

你只需覆盖自己关心的部分。规则中没有对应行的角色会保留技能的默认设置。要恢复默认值，删掉该角色所在的行即可。重新运行 `/setup-pstack` 时，会保留那些模型与默认值不同的角色。默认值发生变化后，此前写下的规则仍会锁定旧默认值，直到你再次运行 `/setup-pstack`。这次运行会替换它能识别的旧默认值，并告诉你替换了哪些，同时保留你的其他选择。要解除它无法识别的锁定，先删掉该角色所在的行，或删掉整个文件。

你可能会想，如果用 Auto 会怎样。把某个角色设为 `inherit-parent` 或 `auto`，pstack 就会省略子代理的 `model` 字段，这样子代理会继承你父级对话的模型。这两个值含义相同，都不是模型 slug。对于评审组角色，其值是一个列表，每一项运行一个子代理，因此列表长度决定了评审组的大小。设置过程还会配置 `swarm workers`，即每个 `/swarm` worker 的默认模型，除非某次竞赛为每一方单独指定了模型。

## 是否接受验证提议

设置结束时，`/setup-pstack` 会寻找一种在你的项目中证明应用行为的方式，可能是 `verify-*` 技能，也可能是已有的测试框架。如果两者都没找到，它会提供一次机会，用 [`/create-verification-skill`](../../skills/create-verification-skill/SKILL.md) 生成一个。

答应后，它会写入 `.cursor/skills/verify-<app>/`，这是一个项目本地技能，教会智能体像用户一样驱动你的应用。在交付之前，它会先验证该技能确实能用一次。拒绝后，设置流程会继续。你随时可以自己运行 `/create-verification-skill`。[验证与发布](./06-verify-and-ship.md#create-a-project-verification-skill)会深入介绍这一点。

如果你刚接触 pstack，就答应吧。能检查自身工作的智能体会一直推进，直到校验通过。做不到这一点的智能体，会把每个结果都交回给你手动检查。在本指南的所有内容中，验证技能的回报最大。

接着，除非项目里已经有正式验证技能，否则设置过程会提供一次机会，用 [`/create-formal-verification-skill`](../../skills/create-formal-verification-skill/SKILL.md) 生成一个。测试检查的是有人想到要写的用例。而正式验证技能会检查代码的一项关键属性，例如"一个任务永远不会同时运行两次"，并在规定范围内对所有情况逐一检查。如果你的代码涉及并发、重试、状态机或解析器，在这些地方 bug 代价高昂，而测试又很少覆盖到出问题的用例，那就答应吧。否则就拒绝，等以后情况有变再运行。[验证与发布](./06-verify-and-ship.md#prove-a-property-with-a-formal-verification-skill)介绍了这一点。

设置完成后，开启一段新对话。模型规则对新会话生效。

## 控制成本

pstack 会在子代理和评审组上额外消耗 token。这就是严谨的代价。想少花一些：

- 重新运行 `/setup-pstack`，选择更小的推理预算或更便宜的模型。主对话用强模型，代码角色用更便宜、更快的模型，这是个不错的搭配。
- 把某个角色设为 `auto` 或 `inherit-parent`，让它使用对话自身的模型运行。
- 缩短评审组列表。每一项运行一个子代理。
- 把 `/poteto-mode` 留给需要严谨处理的工作。小而明显的改动不需要。

## 运行第一个任务

选一个真实但规模不大的任务，像向同事描述那样说明它：

```text
/poteto-mode 给这个命令加一个 --json 标志。文本输出保持逐字节一致。两者都要验证。
```

留意待办清单。最前面几项是复制进来的、与本次提示词匹配的行动手册步骤，也就是功能开发行动手册。

接下来你可以正常输入后续指令。想让 `/poteto-mode` 在整个对话中一直生效，就从 `/` 菜单里选中它，并按 Option+Enter（Mac）或 Alt+Enter（Windows）代替 Enter。这会把它变成一种[自定义模式](https://cursor.com/docs/skills#using-a-skill-as-a-custom-mode)，在你退出之前，它每一轮都会保留在上下文中。自定义模式在 Agents 窗口和 CLI 中都可用。直接按 Enter 只会把技能附加到当前这条消息上，随着对话推进它会逐渐失效。

下一步：[通过 `/poteto-mode` 分派工作](./02-poteto-mode.md)。
