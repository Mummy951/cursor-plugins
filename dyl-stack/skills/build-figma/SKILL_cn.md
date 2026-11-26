---
name: build-figma
description: "用于 \"/build-figma\"、带有 node-id 的 figma.com/design URL，或\"实现这个 Figma\"并应用到 Web、桌面或共享 UI。编排 Figma MCP 的 intake，将节点映射到仓库自身的设计系统，然后由 verify-this 视觉裁判进行判定。不要粘贴 Figma Tailwind。"
icon: paintbrush
color: magenta
# 有意设置为可由模型调用，以便 agents 从 Figma URL 中发现它。
---

# 构建 Figma

Figma → 生产环境 UI 的编排器。它**不会**替代下面的 skills。在每个阶段开始时阅读相应的 skill；不要在此重复它们。

**需要** `figma`（plugin 及已连接的 MCP）和 `cursor-team-kit`。首先通过 [../dyl-mode/references/requirements.md](../dyl-mode/references/requirements.md) 检查。缺失 → 停止并告诉用户运行 `/add-plugin <name>`。

促成这些关卡的失败模式：[references/failure-lessons.md](references/failure-lessons.md)。

## 委托给（不要复制）

| 阶段 | 阅读 |
| --- | --- |
| Figma MCP 机制 | 来自 Figma plugin 的 `figma-design-to-code`，在每次 `get_design_context` 之前 |
| 设计系统清单 | [references/design-system-discovery.md](references/design-system-discovery.md)，然后是它找到的任何仓库本地设计系统 skill 或规则 |
| 视觉声明 / 判定格式 | 来自 `cursor-team-kit` 的 `verify-this` + [references/visual-judge.md](references/visual-judge.md) |
| 驱动实时 UI | 来自 `cursor-team-kit` 的 `control-ui`，或该界面仓库自身的 control skill |

## 关卡

```
Build-figma 进度：
- [ ] 0. 解析 fileKey + nodeId（拒绝仅包含文件的 URL）
- [ ] 1. 在任何产品 UI 编辑之前完成 intake（见下文）
- [ ] 2. 仅映射到仓库自身的 primitives/tokens
- [ ] 3. 使用该界面自身的样式系统进行样式设计
- [ ] 4. 按照 visual-judge.md 由视觉裁判判定为 VERIFIED（verify-this 声明格式）
```

关卡 1 阻止编码。“功能优先，视觉润色靠后”是主要的失败模式。

## 工作流（精简）

### 0. 解析 URL

`figma.com/design/:fileKey/...?node-id=1-2` → `fileKey`、`nodeId`（`1-2` → `1:2`）。分支 URL 使用 `branchKey` 作为 `fileKey`。没有 `node-id` → 停止。

### 1. Intake（阻塞）

遵循 `figma-design-to-code`。调用 `get_design_context` 和 `get_screenshot`。如果响应内容稀疏或过大，拆分为可实现的**子**节点。从 MCP URL 下载品牌素材；绝不重绘 logo。

运行[设计系统发现](references/design-system-discovery.md)。然后编写 `/tmp/build-figma/<slug>/intake.md`：

- 目标界面、代码路径，以及你将如何实时驱动它。
- 节点树。
- **度量表。** Figma px → token，若无匹配的 token 则使用固定值（pinned value）。
- **节点 → primitive 映射。** 将每个 Figma 节点映射到一个现有组件。将任何没有匹配项的节点标记为新建，并说明原因。

### 2. 实现

使用该界面的样式系统和已映射的组件。当 Figma px 与 primitive 的默认尺寸不一致时，在该样式系统中固定（pin）Figma 的度量值。不要在全局范围内改变 primitive 的默认值。硬性禁令如下。

### 3. 视觉裁判（阻塞）

用 [references/visual-judge.md](references/visual-judge.md) 对 `verify-this` 进行专门化。将 Figma 截图视为基线（baseline），将实时截图视为处理（treatment）。先匹配主题极性，然后应用评分标准并写出判定。在 NOT VERIFIED 之后，修复失败的行，重新截图，并再次应用评分标准。重复直到 VERIFIED 或 INCONCLUSIVE。在 INCONCLUSIVE 之后停止并报告阻碍；不要声称关卡 4 已完成。

### 4. 报告

intake 路径、映射要点、判定 + 证据路径、剩余缺口。

## 硬性禁令

1. 将 Figma MCP 的 Tailwind/React 粘贴为产品代码。
2. 当 Figma 或仓库中已有品牌素材时仍自行编造。
3. 为结构明显不同的 Figma 设计使用“差不多”的 primitive。
4. 不检查 Figma px 就信任 text/button/dialog 的尺寸 token。
5. 未经裁判判定就声称视觉部分已完成。
6. 在 `intake.md` 存在之前编辑产品 UI。
