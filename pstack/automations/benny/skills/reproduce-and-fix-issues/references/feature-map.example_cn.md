# 功能地图示例

映射 Benny 可能复现的每一个面向用户的功能。在驱动应用之前阅读相关章节。始终保持这份地图以用户视角为准。内部机制和当前代码路径应在运行时发现，而不是在此处固化。

将此文件复制到 `.cursor/automations/benny/` 之外，例如复制到 `.cursor/benny/feature-map.md`，并将 `control.feature_map_path` 设置为该副本。Pack 刷新时不得覆盖它。

## 各功能模板

### `<feature name>`

`<one-line user-visible purpose>`

#### 用户如何到达这里

- 点击路径：`<screen> -> <menu, tab, or panel> -> <control>`
- 键盘快捷键：`<shortcut or none>`

#### 控制适配器如何驱动它

- `<adapter action>` 配合 `<inputs>` 应产生 `<visible result>`。
- 重置：`<how the adapter returns to a fresh state>`。

#### 稳定选择器

- `<role and accessible name>`
- `<ARIA relationship>`
- `<data-component or purpose-named data attribute>`

切勿使用生成的 CSS 或 StyleX 类名、动态哈希、子元素索引或脆弱的 DOM 位置。

#### 需要覆盖的状态

- 默认、悬停、focus-visible、激活、禁用
- 加载中、空、错误
- 选中、打开、展开
- `<relevant feature-specific variants>`

标记不适用的状态。

#### 前置条件与设置

- 认证：`<account state>`
- 数据：`<fixture>`
- 权限：`<role>`
- 功能标志：`<flag or none>`
- 服务：`<required availability>`

#### 证据与交叉核对

- 截图：`<app identity, feature, and discriminating state>`
- 视频：`<entry path, interaction, and final state>`
- 交叉核对：`<read-only state or value that confirms the UI>`

#### 注意事项

- `<known dead end or wrong surface>`
- `<safe environment translation>`

## 虚构示例

这些功能属于一个虚构的任务应用。它们只是示例，并非 Benny 必需的功能。

### 登录

让用户进入任务应用。

#### 用户如何到达这里

- 打开应用并选择 `Sign in`。无快捷键。

#### 控制适配器如何驱动它

- `open_app`、`click Sign in`、`fill credentials` 和 `click Continue` 应打开项目列表。
- 通过退出登录并清除一次性会话来重置。

#### 稳定选择器

- 按钮 `Sign in`、文本框 `Email` 和 `Password`、`data-component="sign-in-form"`

#### 需要覆盖的状态

- 默认、focus-visible、提交中、禁用、加载中、错误

#### 前置条件与设置

- 一次性账户和可用的认证服务

#### 证据与交叉核对

- 录制从落地页到项目列表的过程。检查只读的会话状态。

#### 注意事项

- 营销页面是错误的界面。缺少认证服务属于阻塞。

### 项目列表与详情

让用户浏览项目并打开其中一个。

#### 用户如何到达这里

- 打开 `Items` 标签页，然后选择一行。

#### 控制适配器如何驱动它

- `select_tab Items` 和 `click <fixture item>` 应打开其详情。
- 通过关闭详情并清除选择来重置。

#### 稳定选择器

- 名为 `Items` 的标签页和列表、以 fixture 命名的行、`data-component="item-detail"`

#### 需要覆盖的状态

- 加载中、空、错误、选中、打开、展开

#### 前置条件与设置

- 已命名的 fixture 项目、读取权限、可用的项目服务

#### 证据与交叉核对

- 展示选中项及对应的详情标题。检查选中项 ID。

#### 注意事项

- 搜索结果可能看起来相似，但使用不同的路径。

### 项目编辑器

让用户创建或编辑项目。

#### 用户如何到达这里

- 在详情中选择 `Edit`，或在列表中选择 `New item`。

#### 控制适配器如何驱动它

- `click Edit`、`fill <field>` 和 `click Save` 应更新详情。
- 通过恢复 fixture 来重置。

#### 稳定选择器

- 按钮 `Edit`、`New item`、`Save`，表单 `Item editor`，带标签关联的字段

#### 需要覆盖的状态

- 默认、focus-visible、已修改、验证中、禁用、保存中、错误、成功

#### 前置条件与设置

- 可编辑的 fixture、写入权限、可用的保存服务

#### 证据与交叉核对

- 通过更新后的详情展示字段变更。只读检查存储的项目值。

#### 注意事项

- 不要注入表单状态。只读的详情字段不是编辑器。

### 设置

让用户更改个人偏好设置。

#### 用户如何到达这里

- 打开个人资料菜单，然后选择 `Settings`。

#### 控制适配器如何驱动它

- `open_menu Profile`、`click Settings` 和 `toggle <preference>` 应更新该控件。
- 通过恢复初始偏好设置来重置。

#### 稳定选择器

- 按钮 `Profile`、菜单项 `Settings`、区域 `Settings`、按用途命名的偏好属性

#### 需要覆盖的状态

- 关闭、打开、选中、focus-visible、禁用、加载中、错误

#### 前置条件与设置

- 已登录的测试账户、已知偏好设置、可用的偏好服务

#### 证据与交叉核对

- 展示菜单路径和最终控件状态。只读检查偏好值。

#### 注意事项

- 操作系统设置属于不同的界面。

## 完整性检查清单

- 每个可复现的面向用户的功能都有对应章节。
- 每个章节都写明了用户路径、适配器操作和重置方式。
- 选择器使用角色、名称、ARIA、稳定的组件标记或按用途命名的属性。
- 没有选择器使用生成的类名或 DOM 位置。
- 相关的交互、加载中、空、错误、选中和展开状态均已覆盖。
- 认证、fixture、权限、功能标志和服务均已明确写出。
- 截图、视频和底层交叉核对要求均已明确写出。
- 错误界面、死胡同和安全环境转换均已列出。
- 实现细节仍留待运行时发现。
