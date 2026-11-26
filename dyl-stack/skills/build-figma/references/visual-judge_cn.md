# 视觉判定器

将 `cursor-team-kit` 中的 `verify-this`  specialization 用于 Figma → 实际 UI 的场景。沿用其可证伪的声明、工件树，以及 VERIFIED / NOT VERIFIED / INCONCLUSIVE 三种判定结论。将声明定义为视觉等价。treatment 必须在每一项评分维度上与 baseline 匹配。当二者匹配时，本 specialization 返回 VERIFIED；当任意一项不通过时返回 NOT VERIFIED；当无法捕获可比较的 treatment 时返回 INCONCLUSIVE。像素级机器 diff 是可选的；结构化的并排对比是必需的。

## 工件

```
/tmp/build-figma/<slug>/
├── intake.md
├── claim.md
├── baseline/        # Figma 截图 + 素材
├── treatment/       # 实际 UI 截图
├── diff/            # 可选的组合图
├── timeline.md
└── verdict.md
```

## 评分表（任意一项 Fail → NOT VERIFIED）

0. **主题极性。** 在其他维度之前，先确认实际 UI 与 Figma 的 light/dark 一致。极性错误时：切换应用主题并重新截图。不要把反色的按钮当作原语 bug。
1. **结构。** 区域与 Figma 相同；没有缺失或多余的框架元素（chrome）。
2. **原语。** 组件与 intake 中的映射一致；没有臆造的品牌标识；图标与 Figma 一致且存在于仓库的图标集中。
3. **字体。** 字号和行高在 intake 指标表的 ~1px 误差范围内（或已固定）。
4. **间距。** 内边距、间距和控件高度与 intake 指标一致。
5. **素材。** 使用 Figma 或仓库导出的素材，通过布局设定尺寸；不许出现自然尺寸放大或预先缩小的图片。
6. **控件。** 复选框、按钮和关闭操作元素匹配；没有猜测的图标名称。

Figma 占位符数量与实际数据不一致不算 Fail，除非要求使用 fixtures。

使用 `control-ui` 或仓库自身的 control skill 驱动界面。判定为 VERIFIED 后，如果仓库有 design-polish 或 QA skill，则运行它。

判定为 NOT VERIFIED 时，修复失败的维度，重新截图，并再次应用评分表。重复此过程直到 VERIFIED 或 INCONCLUSIVE。如果按照 control skill 文档中的设置后仍无法捕获实际 UI，则以 INCONCLUSIVE 停止。在 `verdict.md` 中记录尝试的截图操作和阻碍因素；不要声称 Gate 4 已完成。
