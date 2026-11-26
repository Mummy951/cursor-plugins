# Cursor Team Kit 插件

面向 CI、代码审查、发布和测试可靠性的内部式工作流。该工具包设计为开箱即用，无需集成第三方服务。

## 安装

```bash
/add-plugin cursor-team-kit
```

## 组件

### Skills

| Skill | 描述 |
|:------|:------------|
| `loop-on-ci` | 监控 CI 运行，并针对失败持续迭代直到检查通过 |
| `review-and-ship` | 执行结构化审查、提交更改并创建 PR |
| `pr-review-canvas` | 生成交互式 HTML PR 演示，带有标注和分类的差异对比 |
| `verify-this` | 用基线/处理产物以及明确的结论来证实或推翻论断 |
| `control-cli` | 构建或改造本地工具，以驱动并分析交互式 CLI 或 TUI |
| `control-ui` | 构建或改造本地浏览器/CDP 工具，用于 Web 或 Electron 界面 |
| `make-pr-easy-to-review` | 清理杂乱的 PR 历史、改进描述并添加审查者指南 |
| `run-smoke-tests` | 运行 Playwright 冒烟测试并对失败进行分类 |
| `fix-ci` | 查找失败的 CI 任务、检查日志并应用针对性的修复 |
| `new-branch-and-pr` | 创建新分支、完成工作并发起拉取请求 |
| `get-pr-comments` | 获取并总结当前拉取请求中的审查意见 |
| `check-compiler-errors` | 运行编译和类型检查命令并报告失败 |
| `what-did-i-get-done` | 将指定时间段内自己提交的 commit 总结为简明的状态更新 |
| `weekly-review` | 生成已发布工作的每周回顾，重点突出 bug 修复/技术债务/净新增内容 |
| `fix-merge-conflicts` | 解决合并冲突、验证构建/测试并总结决策 |
| `deslop` | 移除 AI 生成代码中的冗余糟粕并清理代码风格 |
| `workflow-from-chats` | 从聊天中提取持久的工作偏好，转化为 skill、规则或文档 |
| `thermo-nuclear-code-quality-review` | 执行异常严格的可维护性审查（code-judo、千行规则、意大利面条式代码、边界） |

### Agents

| Agent | 描述 |
|:------|:------------|
| `ci-watcher` | 监控 GitHub Actions 运行并返回简明的通过/失败摘要 |
| `thermo-nuclear-code-quality-review` | 对差异运行热核代码质量评分标准任务的子 agent |

### Rules

| Rule | 描述 |
|:-----|:------------|
| `typescript-exhaustive-switch` | 要求对联合类型/枚举进行穷尽的 switch 处理 |
| `no-inline-imports` | 将导入保持在模块顶层，以保证可读性和一致性 |

## 许可证

MIT
