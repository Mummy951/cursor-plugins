---
name: run-smoke-tests
description: 运行 Playwright 冒烟测试、调试失败并验证修复
---

# 运行冒烟测试

## 触发条件

需要在变更之前或之后进行端到端的冒烟验证。

## 工作流程

1. 构建目标应用的先决条件。
2. 运行相关的冒烟测试套件或某个聚焦的测试文件。
3. 如果失败，检查 trace/日志并隔离根本原因。
4. 应用最小修复并重新运行，直到稳定。

## 示例命令

```bash
# Run full smoke suite
npm run smoketest

# Run a specific smoke test file
npm run smoketest -- path/to/test.spec.ts

# Faster iteration when build artifacts are ready
npm run smoketest-no-compile -- path/to/test.spec.ts
```

## 防护栏

- 优先使用确定性的等待和断言，而不是脆弱的超时。
- 重新运行已通过的修复，以减少偶发的误报。
- 仅在明确要求并记录在案时才隔离测试。

## 输出

- 测试结果摘要
- 根本原因与修复
- 剩余的偶发性风险（如有）
