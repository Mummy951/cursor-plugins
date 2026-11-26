### 重构

**你负责契约。结构会变，行为不变。** 这与 Feature（新增行为）和 Bug fix（修正行为）不同。

如果清理过程中发现了缺失的功能或真实的 bug，把它拆分出来，先针对已固定的契约交付结构变更。允许重新设计，但要明确指出并将其路由到 Feature。大型或跨领域的结构性工作属于 **figure-it-out** skill。本 playbook 适用于聚焦到中等规模的变更。

1. 先固定行为契约。追踪受影响的子系统以了解契约，然后在任何结构变动之前，编写一个特征测试（characterization test）、快照或等价性测试框架，捕获当前行为。如果该区域没有覆盖率，在触碰结构之前先写好固定测试。类型检查和 lint 不算固定测试。
2. 按照 **principle-model-the-domain** 指出代码所缺失的结构。当形态已经清晰且局限在本地时，保持平淡的代码不变。重构必须删除分支或无效状态，而不是增加间接层。
3. 命名目标形态。说明如果今天重新构建，模块布局、类型和调用图应该是什么样（**principle-foundational-thinking**、**principle-redesign-from-first-principles**）。如果目标跨越函数边界，在动手之前运行 **architect** skill 来设计形态。
4. 先减后加。在引入新形态之前，删除死代码、折叠只有一个调用者的包装器、去掉冗余的验证器、移除孤立的引用（**principle-subtract-before-you-add**）。能达到目标形态的最小变更即可交付（**principle-laziness-protocol**）。任何"可能有帮助"的推测性清理都要回退。
5. 以保持行为不变的小步骤推进，每一步都保持固定测试通过。对于 API 重构，在同一波中迁移所有调用者并删除旧 API（**principle-migrate-callers-then-delete-legacy-apis**）。不要兼容性 shim，不要新旧并行的路径。对照实际文件逐一抽查每次重命名。重命名会悄悄遗漏字符串、散文和反向引用中的用法。把机械性的编辑委托给一个 subagent，使用你配置的重构模型（默认 `claude-opus-5-5-xhigh-fast`），并给出明确的范围（文件路径、要移动的名称、要保持的行为）。根据 Subagents 章节为每个 subagent 选择合适的推理级别。
6. 在真实产物上证明行为未变，而不是"能编译"（**principle-prove-it-works**）。对于较大的重构，运行等价性检查：一个对比新旧输出的脚本、将记录的基线在新代码上重放，或通过相应的 control skill 在匹配界面上做一次冒烟测试。
7. 确认这个变更值得保留。衡量成功的标准是降低读者的认知负担（**principle-minimize-reader-load**）。如果 diff 没有在某处降低读者负担，就回退它。
8. 变基为有序的小提交。先是一个删减提交，然后是重构，再是任何后续清理。用 **sequence-verifiable-units** principle skill 来组织它们，使每个保持行为的切片在下一个之前都保持通过。运行 **Opening a PR**。

**回复：** 变更的结构、用来约束它的固定测试、等价性证明、读者负担的变化、交付了什么以及回退了什么。不含新行为。
