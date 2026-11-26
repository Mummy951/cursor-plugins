---
name: control-ui
description: 构建或改造本地浏览器/CDP 测试工具，用于驱动和检查 Web、IDE 或 Electron 界面。适用于本地界面验证、截图、无障碍快照、性能分析、视觉差异对比或复现界面缺陷。
---

# 控制界面

使用本地浏览器自动化，以证据验证界面行为。如果仓库自带 Playwright、浏览器或 Electron 测试工具，则优先复用；否则围绕应用的开发服务器或 Chromium 调试端口搭建一个临时本地测试工具。

## 用途

- 复现依赖真实浏览器焦点、键盘输入、滚动、缩放或渲染的界面缺陷。
- 通过截图和快照验证视觉或无障碍方面的改动。
- 在发布前检查本地 Web、IDE 或 Electron 行为。
- 捕获控制台日志、网络日志、CPU 性能分析、跟踪记录或堆快照。
- 为 `verify-this` 创建前后对比证据。

## 设置模式

1. 使用仓库中记录的开发命令在本地启动应用。
2. 查找已有的本地测试工具：Playwright 测试、Cypress 规格、Storybook、浏览器脚本、Electron 启动脚本或快照工具。
3. 对于 Web 应用，使用现有浏览器工具连接到本地 URL。
4. 对于 Electron/Chromium，在支持时启用远程调试端口。
5. 通过稳定的应用标记选择正确页面，而不是仅依赖标签页顺序。
6. 优先使用无障碍角色、标签和稳定的 `data-*` 选择器，而非坐标。

## 通用 Web 测试工具

尽可能使用仓库已安装的浏览器工具。如果仓库已有 Playwright，一个最小的一次性探测示例如下：

```javascript
import { chromium } from "playwright";

const browser = await chromium.launch();
const page = await browser.newPage({ viewport: { width: 1280, height: 800 } });
await page.goto("http://127.0.0.1:<port>");
await page.getByRole("button", { name: /submit/i }).click();
await page.screenshot({ path: "/tmp/ui-harness-after.png", fullPage: true });
await browser.close();
```

除非用户要求，否则不要仅为这次探测就把 Playwright 添加为项目依赖。优先使用现有的开发依赖或环境中已有的外部浏览器工具。

## 通用 CDP 测试工具

对于使用 `--remote-debugging-port=<port>` 启动的 Electron 或 Chromium 应用，通过 CDP 连接：

```javascript
import { chromium } from "playwright";

const browser = await chromium.connectOverCDP("http://127.0.0.1:<debug-port>");
const pages = browser.contexts().flatMap((context) => context.pages());
let page;
for (const candidate of pages) {
  if (await candidate.locator("<app-root-selector>").count()) {
    page = candidate;
    break;
  }
}

if (!page) {
  console.log(await Promise.all(pages.map(async (p) => ({
    title: await p.title(),
    url: p.url(),
  }))));
  throw new Error("No matching app page found");
}

await page.screenshot({ path: "/tmp/ui-harness-cdp.png", fullPage: true });
await browser.close();
```

将 `<app-root-selector>` 替换为当前仓库中的稳定标记，例如应用根节点、地标或特定于产品的 `data-*` 属性。

## 交互循环

1. 在操作前捕获页面快照或截图。
2. 根据最新的页面结构选择目标。
3. 只执行一个结构性操作：点击、输入、按键、拖拽、滚动、导航或调整尺寸。
4. 捕获新的快照/截图。
5. 验证预期的状态变化。
6. 当用户需要证据时，保存用于前后对比的产物。

## CDP 能力

仅当更高级的浏览器 API 不足时才使用原始 CDP：

- 性能：CPU 性能分析、跟踪记录、绘制闪烁、FPS 计数器、布局偏移检查。
- 内存：用于排查内存泄漏的堆快照和强制 GC。
- 网络：请求拦截、限速、禁用缓存、请求/响应日志。
- 渲染：视口变更、配色方案模拟、减少动态效果、无障碍检查。
- 调试：控制台实时输出、异常捕获、DOM 快照。

## 页面选择

当多个应用窗口/标签页共享同一调试端口时：

- 优先为被测界面使用正向标记，例如应用根选择器。
- 必要时使用反向标记以避开错误界面。
- 如果没有匹配的页面，则列出可用的页面标题和 URL，而不要猜测。

## 防护准则

- 导航或结构变化后，不要依赖已失效的元素引用。
- 除非在点击前刚捕获了新截图，否则避免使用坐标点击。
- 将测试数据保持在本地且可随时丢弃。
- 除非用户明确同意，否则不要存储来自隐私敏感工作区的截图或堆快照。
- 不要硬编码来自其他仓库的选择器、端口或脚本路径。应发现当前仓库的本地应用标记。
- 完成后清理开发服务器、调试会话和临时配置文件。
