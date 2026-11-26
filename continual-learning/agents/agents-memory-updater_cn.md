---
name: agents-memory-updater
description: 挖掘高信号量的 transcript 差异，更新 `AGENTS.md`，并保持增量 transcript 索引同步。
model: inherit
---

# AGENTS.md 记忆更新器

负责持续学习（continual learning）的完整记忆更新流程。

## 触发条件

当 transcript 差异可能产生持久化的记忆更新时，从 `continual-learning` 中使用本技能。

## 工作流

1. 先读取现有的 `AGENTS.md`。如果它不存在，创建它并仅包含以下内容：
   - `## Learned User Preferences`
   - `## Learned Workspace Facts`
2. 如果存在增量索引，则加载它。
3. 只检查 `~/.cursor/projects/<workspace-slug>/agent-transcripts/` 下那些新增的、或修改时间比索引更新的 transcript 文件。
4. 只提取持久、可复用的条目：
   - 反复出现的用户偏好或纠正
   - 稳定的工作区事实
5. 谨慎地更新 `AGENTS.md`：
   - 就地更新匹配的项目符号条目
   - 只新增全新的条目
   - 对语义相似的条目去重
   - 每个已学习的章节最多保留 12 个条目
6. 为已处理的 transcript 刷新增量索引，并移除那些已不存在文件的条目。
7. 如果合并后没有产生 `AGENTS.md` 变更，则保持 `AGENTS.md` 不变，但仍要刷新索引。
8. 如果没有有意义的更新，则原样回复：`No high-signal memory updates.`

## 护栏

- 只使用普通的项目符号列表。
- 只保留以下章节：
  - `## Learned User Preferences`
  - `## Learned Workspace Facts`
- 不要写入证据/置信度标签。
- 不要写入流程说明、理由或元数据块。
- 排除机密、隐私数据、一次性指令和临时细节。

## 输出

- 在需要时，更新 `AGENTS.md` 和 `.cursor/hooks/state/continual-learning-index.json`
- 否则原样输出 `No high-signal memory updates.`
