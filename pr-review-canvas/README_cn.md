# PR Review Canvas

一个 Cursor 插件，用于将 pull request 的 diff 渲染为交互式的 Cursor Canvas，其组织方式以评审者的理解为导向，而非按文件树顺序排列。

## 包含内容

- `pr-review-canvas`：构建一个 Canvas，把 PR diff 中的核心逻辑放在顶部，接线/集成部分精简地放在中间，样板代码在底部汇总；对真正棘手的部分添加伪代码、示例轨迹和行内标注。

## 何时使用

- 正在评审一个 pull request，并且你想要一份能凸显真正风险的导览，而不是逐个文件的 diff。
- 为人工评审者汇总一组堆叠的 PR。
- 想要对 diff 做导览、查看变更集概览，或者查看「这里到底改了什么」。

用类似「PR review canvas」「review this PR on a canvas」「walk me through this diff on a canvas」这样的短语来触发它，或者让 agent 指向一个 PR URL / 分支 ref。

## 组织方式

该技能按对评审者的价值对变更进行分组：

1. **核心逻辑** —— 新行为、算法变更、状态转换、API 接口变更。带上下文的完整 diff。
2. **接线与集成** —— 路由注册、DI、配置串联。精简呈现。
3. **样板与机械性变更** —— 导入、重命名、生成的代码、格式化。以列表形式汇总，除非直接相关，否则不展示行内 diff。

然后，它会为密集的逻辑补充伪代码，为难以从 diff 预测的行为变更补充具体的前后示例轨迹，并为真正出人意料或有风险的变更块添加行内标注（`Subtle`、`Breaking`、`Race condition`、`Perf`）。

## 要求

- 已启用 Canvas 的 Cursor。
- 能够访问 diff 来源：本地分支/ref（`git diff`）、GitHub PR URL 或编号（`gh pr diff`），或者一个 Graphite 堆叠（`gt` CLI / `gh`）。

## 许可证

MIT
