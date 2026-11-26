---
name: pr-review-canvas
disable-model-invocation: true
description: 生成一个交互式 PR 审阅导览 HTML 页面。通过 gh API 获取 PR 数据，将文件分类为核心变更与机械性变更，添加审阅者批注，并在渲染 diff 时检测代码移动。当用户粘贴 GitHub PR URL 并要求进行审阅、导览或总结，或说出“review this PR”时使用。
---

# PR 审阅画布

生成对 GitHub PR 的交互式 HTML 审阅，读起来就像一位同事在带你梳理重点内容。

## 工作流

### 1. 获取 PR 数据

并行运行以下 `gh api` 调用：

```bash
gh api repos/{owner}/{repo}/pulls/{number} --jq '{title, body, user: .user.login, state, additions, deletions, changed_files, base: .base.ref, head: .head.ref}'
gh api repos/{owner}/{repo}/pulls/{number}/files --paginate --jq '.[] | {filename, status, additions, deletions, patch}'
gh api repos/{owner}/{repo}/pulls/{number}/comments --jq '.[] | {user: .user.login, body, path, line}'
```

### 2. 分析 PR 并编写正文 HTML

阅读 diff、理解 PR，并直接将 `<body>` 内容编写为 HTML。你拥有充分的创作自由——目标是向审阅者清晰地解释这个 PR。使用最适合该 PR 的任何结构。

**典型结构**（按需调整）：
- 包含标题、PR 编号、作者和统计信息的页头
- 用通俗语言解释该 PR 做了什么的总结框
- 带批注和 diff 的核心文件区块
- 默认折叠的机械性/样板文件
- 底部的审阅清单

**但你也可以添加：**
- **伪代码总结**，用于冗长的代码——用通俗语言或简短伪代码展示算法，并将真实 diff 折叠在下方（使用一个标记为“Show full implementation”的 `.bp-section` 卡片）。当 150 行重试/退避/错误处理代码实际上只是“使用指数退避和熔断器获取数据”时，这种方式非常有效。
- 图表（内联 SVG、通过 CDN 引入的 mermaid、`<pre>` 中的 ASCII 艺术图）
- 展示前后控制流的流程图
- 对比新旧行为的表格
- 用于警告、问题或陷阱的提示框
- 有帮助的交互式组件
- 任何其他能让审阅更清晰的内容

**伪代码模式示例：**
```html
<div class="file-card">
  <div class="file-hdr" onclick="toggle(this)">
    <span class="fname">retryClient.ts</span>
    <div class="fstats"><span class="pill add">+173</span><span class="pill del">&minus;11</span><span class="chev open">&#9654;</span></div>
  </div>
  <div class="file-body open">
    <div class="file-note">
      <strong>What this does in plain English:</strong>
      <pre style="margin-top:8px;color:var(--text);font-size:12px;line-height:1.6;">
fetch(url):
  if circuit breaker is open → fail fast
  retry up to N times:
    try fetch with timeout
    on success → close circuit breaker, return
    on retryable error → wait (exponential backoff + jitter)
    on non-retryable error → throw
  circuit breaker records failure</pre>
    </div>
    <div class="bp-section" style="margin:0;border:0;border-radius:0;">
      <div class="bp-hdr" onclick="toggleBP(this)">
        <span>Show full implementation (+173 lines)</span><span class="chev">&#9654;</span>
      </div>
      <div class="bp-body"><div data-diff="retryClient"></div></div>
    </div>
  </div>
</div>
```

### 3. 可用的 CSS 类和 JS 工具函数

从此 skill 目录读取 [styles.css](styles.css) 和 [renderer.js](renderer.js)。它们为你提供了一套预构建的深色主题工具包。将它们原样注入 [template.html](template.html)。

**可用的 CSS 类：**

| 类 | 用途 |
|-------|---------|
| `.header`, `.header h1`, `.header-meta` | 页面页头 |
| `.pill.add`, `.pill.del`, `.pill.files` | 统计徽章（+N、-N、N 个文件） |
| `.content` | 居中内容容器（最大 900px） |
| `.summary` | 总结/TL;DR 框 |
| `.section-title` | 带下边框的区块标题 |
| `.ic` | 行内代码引用（等宽、蓝色、深色背景） |
| `.file-card`, `.file-hdr`, `.file-body` | 可折叠文件卡片（在 `.file-hdr` 上使用 `onclick="toggle(this)"`） |
| `.file-note` | 文件卡片内置的置顶审阅者批注 |
| `.bp-section`, `.bp-hdr`, `.bp-body` | 折叠的样板文件卡片（使用 `onclick="toggleBP(this)"`） |
| `.bp-note` | 样板文件卡片内的批注 |
| `.verdict` | 审阅清单框 |

**可用的 JS 函数：**

| 函数 | 用法 |
|----------|-------|
| `toggle(hdrElement)` | 切换 `.file-body` 的展开/折叠 |
| `toggleBP(hdrElement)` | 切换 `.bp-body` 的展开/折叠 |
| `renderDiff(target, diffInput)` | 渲染统一 diff。`target` 可以是 DOM 元素、字符串 ID 或 CSS 选择器。`diffInput` 可以是原始 patch 字符串或行数组——两者都可用。自动过滤 import、折叠仅空白变更、检测移动的代码（蓝色/紫色着色）。 |
| `esc(string)` | 对字符串进行 HTML 转义 |

**渲染 diff——使用带自动发现的 `data-diff` 属性。**
在你希望渲染 diff 的任何位置，将 `<div data-diff="KEY"></div>` 占位符放入正文 HTML。渲染器会在 DOM 加载后自动找到它们，并从 [template.html](template.html) 中的 `<script id="pr-diffs-json" type="application/json">` 元素填充内容。

**关键：patch 字符串除了换行符、反斜杠和引号外，还可能包含 `</script>`。** 即使 `json.dumps(...)` 也不够，因为如果你将原始输出粘贴到可执行的 `<script>` 中，HTML 解析可能会提前终止标签。切勿手动将 patch 字符串嵌入 JS/JSON。请改用这种安全的方式：

1. 在获取阶段，使用 `jq` 将 patch 保存到 JSON 文件（它能正确处理转义）：
```bash
gh api repos/{owner}/{repo}/pulls/{number}/files --paginate \
  --jq '[.[] | {key: (.filename | gsub("[^a-zA-Z0-9]"; "_")), value: (.patch // "")}] | from_entries' \
  > /tmp/pr-patches-{number}.json
```

2. 在组装阶段，使用 Python 将 JSON 安全地注入 `template.html`：
```bash
python3 <<'PY'
import json
from pathlib import Path

patches = json.loads(Path('/tmp/pr-patches-{number}.json').read_text())
html = Path('/tmp/pr-review-{number}-body.html').read_text()
css = Path('styles.css').read_text()
js = Path('renderer.js').read_text()
tmpl = Path('template.html').read_text()

# Prevent literal </script> from terminating HTML script tags early.
safe_json = json.dumps(patches).replace('<', '\\u003c').replace('>', '\\u003e').replace('&', '\\u0026')

out = (
  tmpl.replace('/* INJECT_CSS */', css)
      .replace('/* INJECT_JS */', js)
      .replace('<!-- INJECT_BODY -->', html)
      .replace('{"__PR_DIFFS_PLACEHOLDER__":true}', safe_json)
)

Path('/tmp/pr-review-{number}.html').write_text(out)
PY
```

这保证了有效的 JSON 和脚本安全的 HTML 嵌入。agent 将正文 HTML 写入临时文件，然后由 Python 安全地组装所有内容。

diff 数据的键应与 HTML 中的 `data-diff` 属性值匹配：
```html
<div data-diff="path_to_file_ts"></div>
```

由于 renderer.js 在 `<head>` 中加载，如果需要用于自定义场景，你也可以直接从内联 `<script>` 标签调用 `renderDiff(target, lines)`。该函数接受 DOM 元素、ID 字符串或 CSS 选择器作为 `target`，并接受字符串或数组作为 `lines`。

**你不限于这些。** 添加你自己的内联 `<style>` 块、`<script>` 块、SVG、图表或任何其他内容。预构建的组件可以节省时间，但不会限制你。

### 4. 组装并提供服务

1. 将你的正文 HTML（`<body>` 内的所有内容）写入 `/tmp/pr-review-{number}-body.html`
2. 使用上述第 3 步中的 `jq` 命令将 patch 保存到 `/tmp/pr-patches-{number}.json`
3. 运行上述第 3 步中的 Python 组装脚本（从此 skill 目录读取 styles.css、renderer.js、template.html，安全地注入正文和 patch，并写入最终 HTML）
4. 在固定端口上启动本地服务器：
   ```bash
   cd /tmp && python3 -m http.server 8432 --bind 127.0.0.1
   ```
   在后台运行它，然后在应用内浏览器中导航到 `http://127.0.0.1:8432/pr-review-{number}.html`。

   **为什么使用固定端口和 `cd /tmp`：** 后台 shell 没有 TTY，因此 Python 会无限期缓冲其启动消息（“Serving HTTP on...”）——使用端口 0 意味着你永远无法得知选择了哪个端口。而 `--directory /tmp` 虽然可用，但 `cd /tmp` 在不同 Python 版本间更稳健。如果端口 8432 被占用，请尝试 8433、8434 等。

### diff 功能（由 renderer.js 自动处理）

- 过滤掉仅含 import 的行
- 将仅空白变更折叠为上下文行
- 检测移动的代码块（在一处删除 3 行以上连续代码，并在别处完全相同地添加）——以蓝色/紫色渲染，而非红/绿色
- 近似匹配（移动 + 小幅编辑）使用不同的紫色着色

### 样式说明

- 深色主题：`#1a1a1a` 背景，正文使用 Inter 字体，代码使用 IBM Plex Mono 字体
- 橙色使用 `var(--warning)`，绿色使用 `var(--success)`，红色使用 `var(--danger)`，蓝色使用 `var(--accent)`
- 置顶的文件头（`position: sticky; top: 0`）和批注（`top: 35px`）在滚动时保持固定
- 核心文件默认展开（`.file-body.open`），机械性文件折叠
