---
name: setup-pstack
description: 配置 pstack 每个角色使用哪些模型以及相应的推理预算。检测你可用的模型，并写入一条始终应用的规则，覆盖 skill 的默认值。用于 /setup-pstack、"configure pstack models"、"pstack budget"，或更改 pstack 的模型选择。
---

# 配置 pstack（Setup pstack）

写入 `~/.cursor/rules/pstack-models.mdc`，这是一条始终应用的规则，用于设置 pstack 每个角色使用的模型。

## 步骤

### 1. 检测可用模型

枚举本会话中可以传给 `Task` 子代理的模型 slug。这是可靠的来源。如果 Cursor 还暴露了列出用户有权使用的模型的模型 API 或 CLI，优先使用它以求完整。如果你检测不到任何模型，请用户粘贴他们有权访问的 slug。绝不要写入你尚未确认可用的真实 slug。别名 `inherit-parent` 和 `auto` 始终有效，即使它们不是检测到的 slug。

### 2. 加载当前状态

默认的角色到模型映射就是步骤 5 中所示的规则形态。如果 `~/.cursor/rules/pstack-models.mdc` 已经存在，读取它，并把它的 `# budget` 行和角色值当作当前选择。否则从那些默认值开始。某个角色的值如果不在步骤 5 中，例如 `hardest tasks`、`architect runners`、`reflect tooling` 或 `why investigators`，则来自已退役的角色。丢弃它。

较早的默认值曾把另一个 Gemini 模型、然后是 `gemini-4-neon-high` 放在 panel 席位上。因此在任何规则中，位于 panel 角色（`arena runners`、`arena cross-judge pool`、`interrogate reviewers`）中的 `gemini-4-neon-high` 条目，或除 `gemini-4-neon-*` 之外的 `gemini-4-*` 条目，都是旧默认值。每一个都改为 `gemini-4-neon-xhigh`。位于其他任何层级的 Gemini 4 Neon 条目是用户的选择，保持不变。告诉用户你替换了哪些条目。

没有 `# defaults: 3` 行的规则来自较早的设置。那套设置为每个角色写一行，所以其中一些值是旧默认值，而非选择。在步骤 3 之前，替换这些值：

- 一个 `arena cross-judge pool` 列表，如果恰好由一个 `claude-fable-*` 条目、一个 `claude-opus-*` 条目和一个 `gemini-*` 条目组成，顺序任意，则采用该角色步骤 5 的列表。Fable 和 Opus 属于同一厂商，所以 Fable 条目能评判的地方，Opus 条目也能评判。
- 一行 `# budget: unlimited` 改为 `large`，因为没有角色运行在 `xhigh` 之上。
- 如果规则也没有 `# defaults: 2` 行，那么旧 slug 是匹配 `grok-*` 的 slug。位于 `feature, refactoring`、`bug-fix`、`perf-issue`、`hillclimb` 或 `swarm workers` 中的旧 slug，采用该角色步骤 5 的默认值。在 panel 角色（`arena runners`、`arena cross-judge pool`、`interrogate reviewers`）中，一个恰好由一个 `claude-opus-*` 条目和一个旧 slug 组成的列表，顺序任意，采用该角色步骤 5 的列表。在任何其他列表中，每个旧 slug 改为 `claude-opus-5-5-high-fast`。

其他每个值都是用户的选择，保持不变，包括其他任何角色中的旧 slug，以及每个 `inherit-parent` 或 `auto` 值。告诉用户你替换了哪些值。

### 3. 预算、映射并确认

**(a) 询问预算。** 优先使用 AskQuestion 而非自由文本。提供以下三个选项，标签必须完全一致，并在规则记录了当前预算时说出当前预算。没有规则时，说明 `large` 与 skill 默认值一致。

- `large — xhigh reasoning`
- `medium — high reasoning`
- `small — medium reasoning`

**(b) 应用它。** 从 skill 默认值构建工作表，在重新运行时保留你按系列、列表或别名（`inherit-parent`、`auto`）更改过的任何角色。系列是 **arena** skill 的 Phase A 所定义的那些，因此 Fable 自成一个系列。每个角色都有一个目标努力度，即其默认值的努力度按预算移动。`large` 保持不变，`medium` 在阶梯 `xhigh` > `high` > `medium` > `low` 上降低一级，`small` 降低两级。没有角色运行在 `xhigh` 之上，处于阶梯底部的目标就停在底部。大多数默认值都在 `xhigh`，每个预算标签都说明了它们会落到哪里。把角色中每个真实 slug 的努力度 token 设为该角色的目标，包括 panel 条目。努力度 token 是最后一个 token，或尾随 `fast` 之前的那个 token。如果结果不是检测到的 slug，则使用同一系列中努力度不高于目标且最高的那个检测到的 slug，否则把该角色标记为需要选择。`inherit-parent` 和 `auto` 不改变。因此 `medium` 会把 `claude-opus-5-5-xhigh-fast` 变成 `claude-opus-5-5-high-fast`，把 `claude-opus-5-5-high-fast` 变成 `claude-opus-5-5-medium-fast`，而 `small` 会把它们变成 `claude-opus-5-5-medium-fast` 和 `claude-opus-5-5-low-fast`。Opus 5.5 始终运行其 `-fast` slug，所以每个 Opus 5.5 值都以 `-fast` 结尾。Fable 5.2 和 Gemini 没有 fast slug，所以绝不要写一个。Gemini 4 Neon 的 slug 从 `low` 到 `xhigh`，没有 `max` 或 `fast` slug，所以它的席位运行在 `gemini-4-neon-xhigh`，即其最高 slug。panel 角色中的 Gemini 4 Neon 条目采用其自己的目标，即经预算移动后的 `xhigh`，所以 `large` 保留 `gemini-4-neon-xhigh`，`medium` 写入 `gemini-4-neon-high`，`small` 写入 `gemini-4-neon-medium`。

**(c) 展示角色并确认。** 展示每个角色及其模型，把任何不在检测到的集合中的真实 slug 标记为需要选择。同时列出步骤 2 丢弃的每一行。询问是按原样接受还是更改特定角色，提供检测到的 `xhigh` 或以下的模型，加上 `inherit-parent` 和 `auto`（两者含义均为：该角色运行在父聊天模型上，Auto 用户正是靠此保持在 Auto 上）作为选项。优先使用 AskQuestion 而非自由文本。对于 panel 角色（arena runners、interrogate reviewers），值是一个列表，且每个条目运行一个子代理，别名条目也算在内，所以列表长度决定数量。`arena cross-judge pool` 也是一个列表，但 Arena 只运行一个评判。它只尝试来自与父代理不同厂商的值，按列表顺序，所以要按偏好排序。Fable 和其他 Claude 模型属于同一厂商。`inherit-parent` 或 `auto` 值绝不担任评判。除非某场比赛或比较为每个分支指定了另一个模型，否则 `swarm workers` 是每个 worker 的默认模型。

### 4. 校验

写入的每个真实 slug 都必须在检测到的集合中，且处于 `xhigh` 或以下。`inherit-parent` 和 `auto` 始终通过。如果所选的真实 slug 不可用，停下来再次询问。

### 5. 写入规则

写入 `~/.cursor/rules/pstack-models.mdc`，包含 `alwaysApply: true`、`# defaults: 3` 行、一行带所选标签及该标签所命名的努力度的 `# budget`，以及每个角色一行，使用 poteto-mode 所用的相同标签。覆盖整个文件，使重新运行保持幂等。形态如下：

```
---
description: pstack per-role model choices (overrides skill defaults)
alwaysApply: true
---
# pstack model configuration. One line per role. Delete a line to fall back to the skill default.
# `inherit-parent` or `auto` as a value: the role runs on the parent chat model (omit Task `model` and spawn `generalPurpose`). Alias entries in a panel list still count toward its fan-out.
# defaults: 3
# budget: large (xhigh)
feature, refactoring: claude-opus-5-5-xhigh-fast
bug-fix: claude-opus-5-5-xhigh-fast
perf-issue: claude-opus-5-5-xhigh-fast
hillclimb: claude-opus-5-5-xhigh-fast
judgment and prose: claude-opus-5-5-xhigh-fast
arena runners: claude-fable-5-2-high, claude-opus-5-5-high-fast, gemini-4-neon-xhigh
arena cross-judge pool: claude-opus-5-5-high-fast, gemini-4-neon-xhigh
swarm workers: claude-opus-5-5-high-fast
interrogate reviewers: claude-fable-5-2-high, claude-opus-5-5-high-fast, gemini-4-neon-xhigh
```

### 6. 确认

告诉用户规则已写入，并且它适用于新会话。重新运行本 skill 会更新它。

### 7. 提供一个验证 skill（可选）

检查项目是否有办法驱动真实应用来提供证明（一个 `verify-*` skill，或现有的测试框架）。如果没有，提供一次："是否需要一个项目本地的验证 skill，让 agent 能像用户一样驱动应用并证明改动有效？我可以用 /create-verification-skill 生成一个。" 回答"是"时，调用 `/create-verification-skill`（解析 pstack 安装的位置：工作区、用户或插件）。回答"否"时，继续，不要强推。

### 8. 提供一个形式化验证 skill（可选）

检查项目是否已有形式化验证 skill（一个项目 skill，对代码运行模型检查器、证明或其他验证器）。如果没有，提供一次："是否需要一个形式化验证 skill，让 agent 能证明你的代码的一个关键属性在范围内的所有情况下都成立，而不仅仅是测试覆盖的情况？我可以用 /create-formal-verification-skill 生成一个。" 回答"是"时，调用 `/create-formal-verification-skill`（解析 pstack 安装的位置：工作区、用户或插件）。回答"否"时，继续，不要强推。
