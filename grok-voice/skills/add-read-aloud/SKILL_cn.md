---
name: add-read-aloud
description: >-
  当用户运行 /add-read-aloud 或希望应用使用 Grok 文本转语音朗读文本时使用：
  助手回复上的朗读按钮、自动朗读、TTS、语音输出、旁白、IVR 提示、语音标签、voice_id。
  对于双向语音 agent 使用 /add-voice。对于语音转文本使用 /add-dictation。
---

# 添加朗读功能

将 Grok 文本转语音添加到现有应用：助手回复上的扬声器按钮、自动朗读，或任意文本的旁白。在运行 `/add-read-aloud`、输入 **Read aloud**（朗读），或存在明确的"朗读这段" / "TTS"意图时执行。Cursor 没有扬声器；应连接**应用**，而非 IDE。

## 文档

- https://docs.x.ai/developers/model-capabilities/audio/text-to-speech
- API 参考：https://docs.x.ai/developers/rest-api-reference/inference/voice
- 自定义语音：https://docs.x.ai/developers/model-capabilities/audio/custom-voices
- 定价（仅引用文档）：https://docs.x.ai/developers/pricing

## 选择路径

| 需求 | 路径 |
| --- | --- |
| 点击扬声器，听到完成的回复。为页面朗读旁白。生成文件。 | **批量** `POST https://api.x.ai/v1/tts`（默认） |
| 在 LLM 仍在流式输出时就开始播放音频；打断（barge-in）；超过 15,000 字符的文本 | **流式** `wss://api.x.ai/v1/tts`，通过后端中继 |

批量是朗读按钮的默认方式：一次请求、一个 MP3、可缓存，密钥永不离开服务器。只有当 UX 需要在文本完成前播放音频时才使用流式。`POST /v1/tts` 没有文档记载的流式标志；不要臆造一个。

## 认证

- Bearer `XAI_API_KEY`，仅限服务端。TTS 文档没有记载临时令牌流程，且浏览器无法设置 WebSocket 头，因此浏览器流式播放需通过你的后端中继。
- 切勿将密钥放入客户端 bundle。不要在聊天中粘贴密钥。

## 步骤

1. **梳理应用**
   - 助手消息在哪里渲染、每条消息的操作在哪里（复制、重新生成）、回复流如何结束、服务端框架、包管理器。
   - **扬声器图标属于朗读功能**。波形图标是语音模式（`/add-voice`），麦克风图标是听写（`/add-dictation`）。在消息操作行中放置一个幽灵（ghost）扬声器按钮；加载时显示旋转指示器，播放时显示停止方块。一次只播放一段语音：开始新的会停止当前的。
   - 将操作行与回复的**文本边缘**对齐，而非按钮的边框：图标按钮会将其字形居中，因此如果助手气泡没有内边距，请将该行向左移动相应的内缩量（例如对于 24 px 按钮中的 12–14 px 图标，使用 `-ml-[5px]`）。在浏览器中测量；`<p>` 和 `<svg>` 上的 `getBoundingClientRect` 应具有相同的左边缘。
   - 仅在回复**完成流式输出后**才渲染按钮；对于正在进行中的消息，它会朗读不完整的回复。
   - 当屏幕上有多条消息时，将播放器状态（当前消息 id、`loading | playing`、最近的错误）保存在一个共享 store 中（`useSyncExternalStore`、signal，或应用使用的任何方式），以便每个按钮都能反映该状态，并且错误可以在应用现有的状态区域中显示。每个按钮各自的 `let current` 是不够的。
   - 自动朗读：选择加入（opt-in）的开关，默认关闭，并且仅在页面上发生用户手势之后（自动播放策略）。切勿在加载时自动朗读。
   - 如果已安装 `/add-voice`，其 `AudioContext` 和 PCM 播放器可以播放流式 TTS；不要添加第二套音频图。

2. **准备文本**
   - 朗读散文，而非标记。剥离 markdown：标题 → 文本，`**bold**` → 文本，链接 → 链接文本，行内代码 → 代码本身，围栏代码块 → `[pause] Code block omitted.`，表格 → 每行一句话或省略。保留标点符号；它决定朗读节奏。
   - 中和回复中出现的语音标签（`[laugh]`、`<whisper>`……），使模型的文本无法控制朗读表现。仅剥离**文档记载的标签名**（见第 5 步列表），而非所有方括号：`[1]` 引用和 `[note]` 必须保留。
   - 批量限制为**每次请求 15,000 字符**。对更长的文本按段落、然后句子、再单词边界进行拆分，并依次播放各部分；在播放第 N 部分时获取第 N+1 部分，否则每个边界处都会出现静默间隔。或者使用流式。
   - 按 `hash(text + voice_id + language + speed)` 进行缓存；同一回复经常被重复播放。

3. **批量路径（默认）**
   - 服务端：你的路由接收 `{ text, voice_id?, language? }`，校验每个字段的格式（`voice_id` 为 `^[a-z0-9-]{1,64}$`，`language` 为 BCP-47 或 `auto`），转发 JSON，并以来自上游的 `Content-Type` 和 `Cache-Control: no-store` 将响应体流式返回。将上游 404 映射为"未知语音"，以便客户端获得可读的错误。默认输出为 24 kHz / 128 kbps 的 MP3，在浏览器中随处可播放。
   - `language`：对于聊天应用默认为 `"auto"`，此时回复跟随用户的语言；仅为固定语言的产品固定为 `"en"` 等。

```ts
// server (any runtime with fetch)
export async function speak(text: string, voice_id = "eve", language = "auto") {
  if (!text.trim() || text.length > 15_000) throw new Error("TTS text must be 1–15,000 chars");
  const res = await fetch("https://api.x.ai/v1/tts", {
    method: "POST",
    headers: { Authorization: `Bearer ${process.env.XAI_API_KEY}`, "Content-Type": "application/json" },
    body: JSON.stringify({
      text,
      voice_id,
      language,                       // required: `auto` or BCP-47 (`en`, `pt-BR`); omitting it → 422
      // output_format: { codec: "mp3", sample_rate: 24000, bit_rate: 128000 }, // default
      // speed: 1.0,                  // 0.7–1.5
      // text_normalization: true,    // "$5" → "five dollars"
      // replace: { nginx: "/ˈɛndʒɪn ˈɛks/" },
    }),
  });
  if (!res.ok) throw new Error(`TTS ${res.status}`); // 400 bad text/format, 401 key, 404 unknown voice_id, 422 missing required field (e.g. language), 429/500/503 back off and retry
  return new Response(res.body, { headers: { "Content-Type": res.headers.get("content-type") ?? "audio/mpeg" } });
}
```

```ts
// client
let current: HTMLAudioElement | null = null;
async function readAloud(text: string, voiceId = "eve") {
  current?.pause(); current = null;                       // one utterance at a time
  const res = await fetch("/api/tts", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ text, voice_id: voiceId }) });
  if (!res.ok) throw new Error("TTS request failed");
  const url = URL.createObjectURL(await res.blob());
  const audio = new Audio(url);
  audio.addEventListener("ended", () => URL.revokeObjectURL(url)); // avoid blob leaks
  current = audio;
  await audio.play();                                      // call from the click handler's promise chain
}
function stop() { current?.pause(); current = null; }
```

   - Safari：在 blob URL 上 `audio.duration` 为 `Infinity`。如果需要进度条，可用 `AudioContext.decodeAudioData(buf.slice(0))` 解码，或请求 `with_timestamps: true` 并从 JSON 信封中读取 `duration`（此时音频以 base64 形式位于 `audio` 中）。
   - 对于在手势之外创建的 `AudioContext`，Safari 会永久将其挂起。请在点击处理程序中同步创建它，且在任何 `await` 之前。

4. **流式路径**
   - 中继：服务端持有密钥、升级浏览器 socket、构建查询字符串、向上转发客户端 JSON 并向下转发服务端 JSON。为浏览器请求 `codec=pcm`：原始 PCM16 数据块可以在到达时即被调度播放，而 MP3 数据块若没有 `MediaSource` 则无法逐块解码。

```ts
import { WebSocketServer, WebSocket } from "ws";

new WebSocketServer({ port: 8789 }).on("connection", (client) => {
  const q = new URLSearchParams({ language: "en", voice: "eve", codec: "pcm", sample_rate: "24000" /* optimize_streaming_latency: "1" */ });
  const up = new WebSocket(`wss://api.x.ai/v1/tts?${q}`, { headers: { Authorization: `Bearer ${process.env.XAI_API_KEY}` } });
  up.on("message", (d) => client.send(d.toString()));                                   // audio.delta, audio.done, audio.clear, session.updated, error
  client.on("message", (d) => up.readyState === WebSocket.OPEN && up.send(d.toString())); // text.delta, text.done, text.clear, session.update
  const end = () => { client.close(); up.close(); };
  up.on("close", end); up.on("error", end); client.on("close", end);
});
```

   - 客户端：每个聊天会话一个 socket（它在多段语音之间保持打开；每个团队 50 个并发会话，许可 TTL 600 秒，因此关闭时重连）。将 LLM token 作为 `text.delta` 转发（每个 ≤ 15,000 字符），回复完成时发送 `text.done`，停止时或新回复开始时发送 `text.clear`；收到 `audio.clear` 时丢弃排队的音频。

```ts
const ws = new WebSocket(relayUrl);
const ctx = new AudioContext({ sampleRate: 24000 }); // create in the click handler; resume if suspended
let playhead = 0, sources: AudioBufferSourceNode[] = [];

ws.addEventListener("message", (e) => {
  const ev = JSON.parse(e.data);
  if (ev.type === "audio.delta") {
    const bytes = Uint8Array.from(atob(ev.delta), (c) => c.charCodeAt(0));
    const pcm = new Int16Array(bytes.buffer, 0, bytes.byteLength >> 1);
    const buf = ctx.createBuffer(1, pcm.length, 24000);
    const ch = buf.getChannelData(0);
    for (let i = 0; i < pcm.length; i++) ch[i] = pcm[i] / 32768;
    const src = ctx.createBufferSource(); src.buffer = buf; src.connect(ctx.destination);
    playhead = Math.max(playhead, ctx.currentTime + 0.15);   // ~150 ms lead so chunks butt together
    src.start(playhead); playhead += buf.duration; sources.push(src);
  } else if (ev.type === "audio.done") { /* utterance finished; socket stays open */ }
  else if (ev.type === "audio.clear") { sources.forEach((s) => s.stop()); sources = []; playhead = 0; }
  else if (ev.type === "error") showError(ev.message);
});
// on each LLM token:  ws.send(JSON.stringify({ type: "text.delta", delta: token }))
// on reply finished:  ws.send(JSON.stringify({ type: "text.done" }))
// on stop / barge-in: ws.send(JSON.stringify({ type: "text.clear" }))  → wait for audio.clear before the next text.delta
```

   - 跨 `text.delta` 边界拆分的单词没有问题；匹配和合成会跨 delta 进行。

5. **选项**（批量的 JSON 字段，流式的查询参数）

| 需求 | 设置 |
| --- | --- |
| 不同的语音 | `voice_id`（批量）/ `voice`（流式）。内置语音来自 `GET /v1/tts/voices`：`eve`（默认）、`ara`、`rex`、`leo`、`luna`、`atlas`、`aurora`、`orion`……共 28 个，均支持多语言、不区分大小写。自定义语音：来自控制台的 8 字符 id 或 `GET /v1/custom-voices` |
| 非英语或混合语言 | `language`：`en`、`ar-EG`、`ar-SA`、`ar-AE`、`bn`、`zh`、`fr`、`de`、`hi`、`id`、`it`、`ja`、`ko`、`pt-BR`、`pt-PT`、`ru`、`es-MX`、`es-ES`、`tr`、`vi`，或 `auto` |
| 更快或更慢 | `speed` 0.7–1.5 |
| 将"$5"、"Dr."、"3/4"读成文字 | `text_normalization: true` |
| 品牌名、首字母缩略词、行话 | `replace: { "Acme Mobile": "Acme Mobull", "nginx": "/ˈɛndʒɪn ˈɛks/" }`；≤200 条，键 ≤100 字符（字母、数字、撇号、空格），值 ≤128；全词匹配、不区分大小写、最长匹配优先。流式：在第一个 `text.delta` 之前使用 `session.update { replace }` |
| 富有表现力的朗读 | 行内 `[pause]`、`[long-pause]`、`[laugh]`、`[chuckle]`、`[giggle]`、`[cry]`、`[sigh]`、`[breath]`、`[inhale]`、`[exhale]`、`[tsk]`、`[tongue-click]`、`[lip-smack]`、`[hum-tune]`。将 `<whisper>`、`<soft>`、`<loud>`、`<emphasis>`、`<build-intensity>`、`<decrease-intensity>`、`<slow>`、`<fast>`、`<higher-pitch>`、`<lower-pitch>`、`<singing>`、`<sing-song>` 包裹在整段短语周围 |
| 字幕、卡拉OK、唇形同步 | `with_timestamps: true` → JSON `{ audio (base64), content_type, duration, audio_timestamps: { graph_chars[], graph_times[][start,end] } }`；按顺序遍历 `graph_chars`，切勿按索引切片输入 |
| 更快获得首个音频（流式） | `optimize_streaming_latency=1`（文档还列出 `2`；API 参考列出 `0`/`1`） |
| 电话 / IVR | `output_format: { codec: "mulaw" \| "alaw", sample_rate: 8000 }`；无法在浏览器中播放 |
| 编辑、后期制作 | `codec: "wav"`，`sample_rate: 44100` 或 `48000` |
| 更小的文件 | `codec: "mp3"`，`bit_rate: 64000` |

6. **Python 对应实现（仅当服务端是 Python 时）**

```python
import os, requests
r = requests.post(
    "https://api.x.ai/v1/tts",
    headers={"Authorization": f"Bearer {os.environ['XAI_API_KEY']}"},
    json={"text": text, "voice_id": "eve", "language": "en"},
)
r.raise_for_status(); audio_bytes = r.content  # audio/mpeg
# streaming: websockets.connect(url, additional_headers={"Authorization": f"Bearer {key}"}); send {"type":"text.delta",...}, {"type":"text.done"}; read audio.delta / audio.done
```

7. **冒烟测试**
   - 批量：`curl -X POST https://api.x.ai/v1/tts -H "Authorization: Bearer $XAI_API_KEY" -H "Content-Type: application/json" -d '{"text":"Hello from read aloud.","voice_id":"eve","language":"en"}' --output /tmp/hello.mp3` → 200 `audio/mpeg`（MP3、24 kHz、128 kbps、单声道），可播放。省略 `language` → 422（实测；文档表格仅列出 400）。`voice_id: "nope"` → 404。
   - 在应用中：回复流式输出时没有扬声器；回复结束时出现。点击 → 旋转指示器 → 停止方块 → 音频结束时回到扬声器；播放中再次点击 → 立即停止；在第一条播放时点击第二条回复 → 第一条停止，第二条播放。Safari：首次播放可从点击触发，`URL.revokeObjectURL` 在 `ended` 时触发。
   - 文本准备：一条包含围栏代码块、表格、`[1]` 引用和一个游离的 `[laugh]` 的回复 → 读作"Code block omitted"（代码块已省略）、每行一句、引用保持完整、标签被去除。对此进行单元测试；它是纯函数。
   - 流式：逐 token 发送一个包含两句话的回复；音频应在 `text.done` 之前开始。在语音播放中发送 `text.clear` → `audio.clear`，播放停止且没有残留。在同一 socket 上发送第二段语音 → 全新的 `audio.delta`，没有来自第一段的声音串入。
   - 在客户端 bundle 中搜索 `XAI_API_KEY`；它绝不应出现在其中。针对正在运行的 dev server，获取 `/`，收集其引用的 `/_next/static/chunks/*.js`（或等效）URL，并逐个 grep；不要依赖你尚未构建的生产版本。
   - 使用 `/debug-voice` 从日志中调试；将其钩子点替换为 `audio.delta`（字节数）、`audio.done`、`audio.clear`、`error`。

## 范围之外

- 能听并回答的语音 agent（`/add-voice`）、语音转文本（`/add-dictation`）
- 超出传递现有自定义 `voice_id` 范围的语音克隆
- 臆造 TTS 令牌流程、`POST /v1/tts` 上的 `stream` 标志，或文档中未记载的事件名称
