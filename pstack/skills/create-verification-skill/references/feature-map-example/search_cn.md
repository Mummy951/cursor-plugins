# 搜索笔记

搜索让用户按标题或正文文本查找笔记、查看匹配的笔记，并区分"无匹配"与"搜索不可用"两种情况。

## 子功能

- `search-open` 从每个受支持的浏览器入口点打开搜索。
- `search-match` 返回标题和正文匹配项，且不更改笔记数据。
- `search-open-result` 在笔记编辑器中打开某个结果。
- `search-empty` 针对无匹配项的查询显示完整的空状态。
- `search-clear` 清除查询并恢复"最近笔记"视图。
- `search-cli` 从终端返回相同的匹配笔记。

## 如何进入（用户视角）

- 选择浏览器工具栏中的 `Search` 按钮。
- 当焦点位于可编辑字段之外时，在浏览器中按 `/`。
- 在终端中运行 `notes search <query>`。

## 使用 control-notes 驱动

前置条件：

- Notes 在 `http://127.0.0.1:4173` 上运行正常。
- 一次性数据目录中包含正文为 `Draft budget` 的 `Quarterly plan`。
- `control-notes doctor` 报告预期的 URL 和数据目录。

- **工具栏入口。** 选择 `Search` 按钮。运行 `control-notes browser click --role button --name "Search"`。会出现一个名为 `Search notes` 的对话框，焦点位于其搜索框中。
- **键盘入口。** 关闭对话框，将焦点置于页面上，然后按 `/`。运行 `control-notes browser press --key "/"`。会出现同一个对话框，且页面不会插入斜杠。
- **标题匹配。** 输入 `quarterly`。运行 `control-notes browser fill --role searchbox --name "Search notes" --value "quarterly"`。`Search results` 列表包含 `Quarterly plan`，且不包含 `Grocery list`。
- **正文匹配。** 将查询替换为 `budget`。运行 `control-notes browser fill --role searchbox --name "Search notes" --value "budget"`。结果 `Quarterly plan` 仍然可见，并带有正文匹配摘录。
- **打开结果。** 选择 `Quarterly plan`。运行 `control-notes browser click --role link --name "Quarterly plan"`。对话框关闭，编辑器标题显示为 `Quarterly plan`。
- **空状态。** 重新打开搜索并输入 `volcano`。运行 `control-notes browser fill --role searchbox --name "Search notes" --value "volcano"`。搜索完成后会出现一个名为 `No matching notes` 的状态提示。
- **清除查询。** 选择 `Clear search`。运行 `control-notes browser click --role button --name "Clear search"`。搜索框为空，`Recent notes` 区域取代了结果列表。
- **CLI 匹配。** 从终端搜索。运行 `control-notes cli -- notes search "quarterly" --format json`。退出码为 `0`，stdout 包含一个标题为 `Quarterly plan` 的对象。
- **CLI 未命中。** 搜索一个不存在的值。运行 `control-notes cli -- notes search "volcano" --format json`。退出码为 `0`，stdout 为 `[]`。
- **证明。** 捕获填充后的结果状态。运行 `control-notes browser snapshot --aria --path artifacts/search/results.aria.txt` 和 `control-notes browser screenshot --path artifacts/search/results.png`。这两个产物都能标识出 Notes、查询内容和 `Quarterly plan`。

## 注意事项

- 当编辑器或搜索框具有焦点时按 `/` 会插入文本，而不是打开搜索。
- 结果会在短暂的防抖后更新。应等待结果列表或空状态出现，而不是使用固定时长的 sleep。
- 除非用户启用 `Include archived`，否则已归档的笔记会被排除。
- CLI 默认输出便于人阅读的格式。使用 `--format json` 可获得稳定的断言结果。
- 打开结果会改变浏览器状态。在验证另一个查询之前，请重新打开搜索。
