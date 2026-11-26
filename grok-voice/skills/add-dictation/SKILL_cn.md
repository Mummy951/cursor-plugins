---
name: add-dictation
description: >-
  当用户运行 /add-dictation，或希望使用 Grok 语音转文字（speech-to-text）时
  使用：一个麦克风按钮，可将语音听写输入到输入框、实时字幕，或转录已录制的
  音频（文件、上传、URL），并提供词级时间戳、说话人分离、字幕、会议记录。
  STT、transcribe、transcription。若需要能对话的语音 agent，请使用 /add-voice。
---

# 添加听写功能

为现有应用添加 Grok 语音转文字功能：一个可将语音听写输入到输入框的麦克风按钮、实时字幕，或已录制音频的转录文本。在 `/add-dictation`、输入 **Dictate**，或明确的“transcribe”意图时运行。Cursor 没有麦克风；应连接到**应用**，而非 IDE。

## 文档

- https://docs.x.ai/developers/model-capabilities/audio/speech-to-text
- 模型选择（先阅读）：https://docs.x.ai/developers/model-capabilities/audio/speech-to-text#model-selection
- 定价（仅引用文档）：https://docs.x.ai/developers/pricing

## 模型

在任何 STT 调用之前，先阅读模型选择章节，并将**文档中列出的最新**模型作为 `model` 传入。不要编造 id 或复用本 skill 中的过时模型。REST：表单字段在 `file` 之前。流式传输：查询参数。官方示例当前发送 `grok-voice-transcribe-2.0`。当前列出的模型：`grok-voice-transcribe-2.0`（最佳）、`grok-voice-transcribe-1.0`（原始版本；省略 `model` 时的 API 默认值）。

## 选择路径

| 需求 | 路径 |
| --- | --- |
| 点击、说话、再点击，文字出现。上传文件。URL。 | **批量** `POST https://api.x.ai/v1/stt`（默认） |
| 说话时文字即出现：字幕、长篇听写、按键说话 | **流式** `wss://api.x.ai/v1/stt`，经由后端中继 |

对于输入框的麦克风按钮，批量是默认方案：一次请求、无 socket、密钥永不离开服务端。仅当用户体验需要中间文本时才使用流式传输。

## 鉴权

- Bearer `XAI_API_KEY`，仅限服务端。STT 文档未说明临时令牌流程，且浏览器无法设置 WebSocket 请求头，因此浏览器流式传输需经由你的后端中继。不要编造令牌流程。
- 切勿将密钥放入客户端 bundle。不要在聊天中粘贴密钥。

## 步骤

1. **梳理应用**
   - 输入框或输入组件、文本应落在何处（在光标处插入还是替换）、服务端框架、包管理器。
   - **麦克风图标属于听写功能**。如果已安装 `/add-voice`，其波形主按钮保持不变；将麦克风作为次要的幽灵按钮添加在其旁边。
   - 已有麦克风采集？如果运行过 `/add-voice`，其 PCM 采集可供流式 STT 使用；将其采样率作为 `sample_rate` 传入。16 kHz 是模型的原生采样率；其他支持的采样率（8000、16000、22050、24000、44100、48000）会在服务端重采样。

2. **批量路径（默认）**
   - 客户端：`MediaRecorder` → `Blob` → `POST` 到你自己的路由。该端点会自动检测容器格式（WAV、MP3、OGG、Opus、FLAC、AAC、MP4、M4A、MKV、WebM），因此直接发送 `MediaRecorder` 产生的任意格式即可。
   - 服务端：以 `multipart/form-data` 转发。选项字段在前，**`file` 放在最后**；`file` 之后的字段可能被忽略。`file` 或 `url`，最大 500 MB。`model` 取自文档（列出的最新版本）。

```ts
// server (any runtime with fetch + FormData)
export async function transcribe(blob: Blob, filename: string) {
  const form = new FormData();
  form.append("model", "grok-voice-transcribe-2.0"); // latest from docs; re-read model-selection
  form.append("format", "true");     // written-form numbers/currency; requires language
  form.append("language", "en");
  // form.append("keyterm", "Acme"); // repeat per term, ≤100 terms × 50 chars
  form.append("file", blob, filename); // last
  const res = await fetch("https://api.x.ai/v1/stt", {
    method: "POST",
    headers: { Authorization: `Bearer ${process.env.XAI_API_KEY}` },
    body: form,
  });
  if (!res.ok) throw new Error(`STT ${res.status}`); // 400 bad input, 413 >500 MB, 429 back off, 502 url fetch failed, 503 retry
  return (await res.json()) as {
    text: string; language: string; duration: number;
    words?: { text: string; start: number; end: number; speaker?: number }[];
    channels?: { index: number; text: string; words: unknown[] }[];
  };
}
```

```ts
// client
const mime = MediaRecorder.isTypeSupported("audio/webm;codecs=opus") ? "audio/webm;codecs=opus" : "audio/mp4";
const rec = new MediaRecorder(stream, { mimeType: mime });
const parts: BlobPart[] = [];
rec.ondataavailable = (e) => parts.push(e.data);
rec.onstop = async () => {
  const fd = new FormData();
  fd.append("file", new Blob(parts, { type: mime }), "dictation");
  const { text } = await (await fetch("/api/dictation", { method: "POST", body: fd })).json();
  insertAtCursor(text);
};
rec.start(); // second tap: rec.stop()
```

3. **流式路径**
   - 中继：服务端持有密钥，升级浏览器的 socket，向上转发二进制帧和客户端控制消息，向下转发 JSON 事件。查询字符串在服务端构建。`model` 取自文档（列出的最新版本）。

```ts
import { WebSocketServer, WebSocket } from "ws";

new WebSocketServer({ port: 8788 }).on("connection", (client) => {
  const q = new URLSearchParams({ model: "grok-voice-transcribe-2.0", sample_rate: "16000", encoding: "pcm", interim_results: "true", language: "en" });
  const up = new WebSocket(`wss://api.x.ai/v1/stt?${q}`, { headers: { Authorization: `Bearer ${process.env.XAI_API_KEY}` } });
  up.on("message", (d) => client.send(d.toString()));                       // transcript.* and error events
  client.on("message", (d, isBinary) => up.readyState === WebSocket.OPEN && up.send(d, { binary: isBinary })); // audio + finalize/audio.done
  const end = () => { client.close(); up.close(); };
  up.on("close", end); up.on("error", end); client.on("close", end);
});
```

   - 浏览器采集：PCM16 小端、单声道、16 kHz、**100 ms 帧 = 3,200 字节**、原始二进制、非 base64。发送前先等待 `transcript.created`。`MediaRecorder` 输出是容器格式，而非原始帧；不要将其流式发送。

```ts
const ws = new WebSocket(relayUrl); ws.binaryType = "arraybuffer";
const ctx = new AudioContext({ sampleRate: 16000 }); // if ctx.sampleRate !== 16000, downsample in the worklet
await ctx.audioWorklet.addModule("/pcm16-worklet.js"); // Float32 → Int16LE, posts one 3,200-byte frame per 100 ms
const node = new AudioWorkletNode(ctx, "pcm16");
ctx.createMediaStreamSource(stream).connect(node);
let ready = false;
node.port.onmessage = (e) => ready && ws.readyState === WebSocket.OPEN && ws.send(e.data);

let committed = "", locked = "", live = "";
ws.addEventListener("message", (e) => {
  const ev = JSON.parse(e.data);
  if (ev.type === "transcript.created") ready = true;
  else if (ev.type === "transcript.partial") {
    if (ev.speech_final) { committed += ev.text + " "; locked = ""; live = ""; } // complete stitched utterance
    else if (ev.is_final) { locked += ev.text + " "; live = ""; }                // chunk final: text will not change
    else live = ev.text;                                                          // interim: may change
    render(committed + locked + live);
  } else if (ev.type === "transcript.done") ws.close();                          // after audio.done
  else if (ev.type === "error") showError(ev.message);                           // most errors close the socket
});
// stop: ws.send(JSON.stringify({ type: "audio.done" }))
// push-to-talk release: ws.send(JSON.stringify({ type: "Finalize" })) then keep streaming (docs show both `finalize` and `Finalize`; the examples use `Finalize`)
```

4. **选项**（流式传输用查询参数，批量用表单字段）

| 想要 | 设置 |
| --- | --- |
| 最新 STT | 阅读文档 #model-selection，传入该 `model`。快照：`grok-voice-transcribe-2.0`。省略 → `grok-voice-transcribe-1.0`。批量：表单字段。流式：查询参数。 |
| 说话时显示文字 | `interim_results=true` |
| “one hundred dollars” → `$100` | 流式：`language=en`；批量：`format=true` + `language=en` |
| 产品名、专业术语 | `keyterm=` 重复传入 |
| 听写数字时句中不被截断 | `smart_turn=0.7&smart_turn_timeout=3000` |
| 更快或更慢的语句结束判定 | `endpointing=` 毫秒，默认 400 |
| 谁说了什么（会议） | `diarize=true` → `words[].speaker` |
| 将客服与客户分到不同声道 | `multichannel=true&channels=2`（仅 PCM，不支持 Opus） |
| 保留“um”、“uh” | `filler_words=true`（默认会移除） |
| 低带宽或移动端 | `encoding=opus`，每帧恰好一个原始 Opus 包，省略 `sample_rate` |
| 将原始音频传给批量接口 | `audio_format=pcm|mulaw|alaw` + `sample_rate` |
| 安静或电话音频 | 降低 `vad_threshold`（流式默认 0.08，批量 0.5） |

5. **Python 版本（仅当服务端是 Python 时）**

```python
import os, requests
r = requests.post(
    "https://api.x.ai/v1/stt",
    headers={"Authorization": f"Bearer {os.environ['XAI_API_KEY']}"},
    data=[("model", "grok-voice-transcribe-2.0"), ("format", "true"), ("language", "en")],
    files={"file": ("dictation.webm", blob, "audio/webm")},  # requests sends data fields before files
)
r.raise_for_status(); text = r.json()["text"]
# streaming: websockets.connect(url, additional_headers={"Authorization": f"Bearer {key}"}); await ws.send(pcm_bytes)
```

6. **冒烟测试**
   - 批量：取自文档的最新 `model`（当前为 `grok-voice-transcribe-2.0`）：`curl -X POST https://api.x.ai/v1/stt -H "Authorization: Bearer $XAI_API_KEY" -F model=grok-voice-transcribe-2.0 -F language=en -F file=@short.wav` → 200 且返回 `text`。同一调用加上 `-F format=true` 但不带 `language` → 400。
   - 流式：听写两个句子，中间停顿。预期先出现中间文本，然后是最终结果；语句边界处不应有重复或消失的词（若词消失，说明拼接后的 `speech_final` 文本未包含各分块最终结果：应追加而非替换 `locked`）。`audio.done` → `transcript.done`，socket 关闭。
   - 在客户端 bundle 中搜索 `XAI_API_KEY`；其中不得出现该密钥。
   - 使用 `/debug-voice` 从日志中调试；将其 hook 点改为 `transcript.*` 事件。

## 不在范围内

- 会回话的语音（`/add-voice`）、朗读文本（`/add-read-aloud`）
- 编造文档中未出现的 STT 令牌流程、端点、模型 id 或事件名称
