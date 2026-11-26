# Docs Canvas

一个用于将文档（架构说明、API 参考、runbook 和代码库导览）渲染为可导航的 Cursor Canvas 的 Cursor 插件，而非一个扁平的 markdown 文件。

## 状态

这个插件是一个**初始脚手架**。skill 结构已经完整，Canvas 欢迎页会在 marketplace 中展示它，但 skill 正文有意只是起步提纲，而非一份经过充分调优的 playbook。随着「在 Canvas 上呈现文档」这一模式逐渐成熟，预计会持续迭代。

## 包含内容

- `docs-canvas`：构建一个 Canvas，以概览卡片、可导航的目录、正文/代码/图表混合的章节以及参考文献区块来呈现结构化文档。

## 何时使用

- 将架构说明、设计文档或 RFC 渲染成可快速浏览的内容，而不只是从头到尾逐行阅读。
- 把一个 markdown 文档目录，或单篇大型文档，转换成带跳转导航的 Canvas。
- 用比单条回复更丰富的布局来回答代码库问题——章节、图表、表格、标注框（callout）。

用「docs canvas」「documentation overview」「architecture walkthrough」「API reference page」或「render this doc as an interactive canvas」这类短语来触发它。

## 组织方式

该 skill 期望一个文档 Canvas 以以下内容开头：

1. **概览（Overview）** — 简短的摘要卡片：目的、范围、受众。
2. **目录（Table of contents）** — 固定/置顶的章节列表，读者可以跳转到其中任意章节。
3. **正文章节（Body sections）** — 每个逻辑单元一节（架构、API、示例、注意事项），混合正文、代码块、图表和标注框。
4. **参考文献（References）** — 指向相关文档、源文件、RFC 和外部资料的链接。

这些只是下限，而非上限——该 skill 鼓励采用任何对特定主题真正有助于读者的表达方式（图表、表格、决策树、详细示例）。

## 要求

- 启用了 Canvas 的 Cursor。
- 源材料：一个 markdown 文件目录、单个文档 URL、一份内联大纲，或一个待回答的代码库问题。

## 许可证

MIT
