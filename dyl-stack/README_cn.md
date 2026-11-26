# dyl-stack

我的 agent 风格，叠加在 [pstack](../pstack/) 之上。pstack 承担主要工作：原则、playbook、subagent 路由。dyl-stack 增加了一些我一直在纠正 agent 的关卡、一个能放进粘贴内容的 PR review，以及一个在视觉正确之前绝不会宣称完成的 Figma-to-UI 流程。

## 安装

```bash
/add-plugin pstack
/add-plugin cursor-team-kit
/add-plugin thermos
/add-plugin dyl-stack
```

`/build-figma` 还需要连接 Figma 插件及其 MCP。

## 技能

| 技能 | 何时使用 |
|---|---|
| [`/dyl-mode`](./skills/dyl-mode/SKILL.md) | 处理非平凡工作的默认入口。通过 pstack 的 `poteto-mode` playbook 路由，并在其上叠加我的关卡。 |
| [`/dyl-review`](./skills/dyl-review/SKILL.md) | 你想要最多 7 条可直接粘贴的 review 评论和一个 🟢/🟡/🔴 判定。说 "deep" 可加入 thermos 和 Bugbot。从不发布。 |
| [`/dyl-ready-pr`](./skills/dyl-ready-pr/SKILL.md) | "让 PR 变绿。" 深度 review 直到 🟢，标记为 ready，修复冲突，照看 CI 直到可合并。从不合并。 |
| [`/build-figma`](./skills/build-figma/SKILL.md) | 你有一个带 `node-id` 的 `figma.com/design` URL。先接收需求，映射到你的仓库设计系统，再对照实时 UI 做视觉判定。也会在出现 Figma URL 时自动触发。 |
| [`principle-the-algorithm`](./skills/principle-the-algorithm/SKILL.md) | 由 `dyl-mode` 引用。质疑需求，删除，然后优化、加速、自动化。 |

## dyl-mode 相比 poteto-mode 新增了什么

- **根本原因，而非表象。** 任何失败在修复前都要先写一个 `Root cause: X because Y` 的 todo。空值保护、重试、`.skip` 和快照更新都属于治标，除非那行说明为之提供了依据。
- 在设计任何比一眼可见的改动更大的东西之前，先执行 **The Algorithm**。
- **简洁回复。** 默认语气采用 pstack 的 `/bro`。
- **合并关卡。** 未经当前回合的许可绝不合并。更新已有的 PR，绝不新开重复的。
- 通过 `control-ui` 提供**实时 UI 证据**，并在编写布局 bug 的修复代码前先测量。
- **品味否决具有约束力。** "Roll that back" 就是要回滚。

## 子代理

[`dyl-agent`](./agents/dyl-agent.md) 端到端地运行这套风格。用 `subagent_type: "dyl-agent"` 来启动它。

## 不在此发布

- 原则、playbook、`/bro`、`/unslop` 以及 babysit 监视器随 `pstack` 发布。
- `deslop`、`control-ui`、`control-cli` 和 `verify-this` 随 `cursor-team-kit` 发布。
- thermos review 子代理随 `thermos` 发布。
- `/review-bugbot` 和 `/create-skill` 是 Cursor 内置的。

## 许可证

MIT
