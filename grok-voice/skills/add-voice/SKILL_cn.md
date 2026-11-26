---
name: add-voice
description: >-
  当用户运行 /add-voice、输入 Voice Mode，或要求为应用添加 Grok
  实时语音时使用，包括替换 STT-LLM-TTS 级联或 OpenAI Realtime。连接 speech-to-speech、安全的 auth 以及应用麦克风。Composer：波形按钮，
  麦克风图标保留给听写使用。仅将麦克风转文字使用 /add-dictation；朗读文本回复使用
  /add-read-aloud。添加 debug 日志并从日志修复使用 /debug-voice。
---

# 添加语音

为现有应用添加 Grok Speech to Speech。在 `/add-voice`、输入 **Voice Mode**，或明确的“添加 Grok 语音”意图时运行。

## 目标

可用的双工链路：用户应用的麦克风输入、音频输出、`wss://api.x.ai/v1/realtime?model=grok-voice-latest`、安全的 auth。Cursor 没有原生麦克风；应连接**应用**（或示例客户端），而不是 IDE。

## 协议优先

与语言无关的事件循环。**默认使用 TypeScript 示例。** 仅在客户端 API 不同的地方提供简短的 Python 对照版本（例如 `ws` 与 `websockets`）。

## 文档

- https://docs.x.ai/developers/model-capabilities/audio/speech-to-speech
- https://docs.x.ai/developers/model-capabilities/audio/ephemeral-tokens
- 定价（仅引用文档）：https://docs.x.ai/developers/pricing （约 $0.08/分钟 STS + $0.004/文本项；单次会话最长 120 分钟）

## 步骤

1. **梳理应用**
   - 技术栈：无、OpenAI Realtime、STT→LLM→TTS 级联、仅 TTS/STT。
   - 客户端：web / Node / iOS / Android / server。
   - 如果存在级联或 OpenAI Realtime：用下面这个单一的双工循环替换它（URL、model、voice、事件差异）；仅在产品仍需要 agent 之外的一次性听或说时，保留独立的 `/v1/stt` 或 `/v1/tts`。

2. **Auth**
   - 服务端：Bearer `XAI_API_KEY`。
   - 浏览器/移动端：后端 `POST https://api.x.ai/v1/realtime/client_secrets`，客户端使用临时 token（Bearer 或浏览器 `sec-websocket-protocol`：`xai-client-secret.<token>`）。
   - 绝不要把长期有效的 key 放进客户端包中。不要在聊天中粘贴 key。

3. **连接 + 会话**
   - URL：`wss://api.x.ai/v1/realtime?model=grok-voice-latest`
   - 连接打开时：发送 `session.update`，包含 `voice`（默认 `eve`）、`instructions`、`turn_detection: { type: "server_vad" }`（或对 push-to-talk 用 `null`），PCM 24 kHz，除非应用已在别处统一标准。
   - 设置 `audio.input.transcription.model: "grok-transcribe"`，否则不会收到用户转录（`conversation.item.input_audio_transcription.updated` 是累积的，不是增量）。
   - 如需要工具：`web_search`、`x_search`、`file_search`、`mcp`、自定义 `function`。

4. **音频 I/O（应用侧）**
   - 每个会话一个 `AudioContext` 用于采集和播放，在用户手势内创建（自动播放策略）。申请 24 kHz；如果浏览器给出其他采样率，发送前重采样。
   - 麦克风 → AudioWorklet，约 100 ms 一段 → `input_audio_buffer.append`（或二进制传输）。并行启动 WS 和麦克风；缓冲早期音频，连接打开后冲刷。
   - 立即播放 `response.output_audio.delta`；提前约 150 ms 调度，使各段首尾相接。在 `input_audio_buffer.speech_started` 时，停止所有已排队内容（barge-in 打断）。
   - 转录行：在 `input_audio_buffer.committed` 时创建用户行（`item_id`），在 `…transcription.updated` 时填充；助手文本来自 `response.output_audio_transcript.delta` / `.done`，在 `response.done` 时结束该轮。
   - 对于 function 工具：`function_call_output`，完成播放，然后 `response.create`。

5. **Composer UI 约定**
   - 一个主按钮，位于 composer 右侧。composer 为空 → **波形**图标（描边，例如 Phosphor `WaveformIcon weight="bold"`；绝不要用 `fill` 字重，在 16 px 下会渲染成一团）；点击启动语音模式。有任意文本 → 经典的**发送**箭头；在语音模式下该文本进入实时会话（`conversation.item.create` + `response.create`）。文本回复流式输出 → 停止方块。
   - 语音进行中时，同一按钮显示**动画波形**（4 根柱，每根约 3 px 宽、2 px 间隔、约 16 px 高，最小 scale 0.4 以在 28 px 按钮中保持清晰），点击结束会话。阶段驱动动画：聆听时慢，说话时快，连接/思考时更慢且略暗（opacity ≥ 0.75）。尊重 `prefers-reduced-motion`。不要用 X 按钮，不要用脉冲光环。
   - 状态位于 **composer 内部，而非其周围**：占位符显示 `Connecting…` / `Listening…` / `Thinking…` / `Speaking…`，另加一个 `sr-only` 的 `role="status"`。不要单独的状态行。
   - **麦克风图标保留给听写**（`/add-dictation`）。绝不要将其用于语音模式。

6. **TS 骨架（默认）**

```ts
const url = "wss://api.x.ai/v1/realtime?model=grok-voice-latest";
// Node: pass Authorization header. Browser: use xai-client-secret.<token> protocol.
const ws = new WebSocket(url /* , { headers: { Authorization: `Bearer ${token}` } } */);

ws.addEventListener("open", () => {
  ws.send(JSON.stringify({
    type: "session.update",
    session: {
      voice: "eve",
      instructions: "You are a helpful voice agent.",
      turn_detection: { type: "server_vad" },
    },
  }));
});

ws.addEventListener("message", (ev) => {
  const event = JSON.parse(String(ev.data));
  if (event.type === "response.output_audio.delta") {
    // decode base64 PCM and play
  }
});
```

7. **Python 对照版（仅当应用是 Python）**

```python
import json, os, websockets

url = "wss://api.x.ai/v1/realtime?model=grok-voice-latest"
headers = {"Authorization": f"Bearer {os.environ['XAI_API_KEY']}"}

async with websockets.connect(url, additional_headers=headers) as ws:
    await ws.send(json.dumps({
        "type": "session.update",
        "session": {
            "voice": "eve",
            "instructions": "You are a helpful voice agent.",
            "turn_detection": {"type": "server_vad"},
        },
    }))
    async for raw in ws:
        event = json.loads(raw)
        if event.get("type") == "response.output_audio.delta":
            pass  # decode and play
```

8. **埋点（在首次人工测试之前）**
   - 运行 `/debug-voice`：它会提出一个计划，然后用应用自身的语言安装仅开发环境的日志 sink（`POST /api/voice/log` → `.voice-logs/<sessionId>.ndjson`，已 gitignore）、一个将音频缩减为字节数的客户端 logger，以及在 UI 中显示 session id。
   - 同一 skill 带有修复循环以及症状 → 日志特征 → 修复的对照表。

9. **冒烟测试**
   - 通过 `conversation.item.create` + `response.create` 进行文本轮次；确认音频或转录事件。
   - 确认客户端中没有长期有效的 key（在构建后的客户端包中 grep 环境变量名和 `client_secrets`）。
   - 让用户戴上**耳机**使用应用。用扬声器时麦克风会听到回复，模型会自我应答；那是回声，不是循环中的 bug。
   - 用 `/debug-voice` 迭代。

## 范围之外

- 仅语音转文字（`/add-dictation`）、朗读文本（`/add-read-aloud`）
- 图像生成和纯文本推理
- 臆造的端点、事件或 CLI 标志
