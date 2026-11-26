---
name: debug-voice
description: >-
  当用户运行 /debug-voice、说语音模式有问题、要求查看或捕获 Grok 实时语音会话中发生的情况，
  或者某个语音集成还没有调试日志时使用。先提出计划，然后以应用自身的语言和约定安装一套仅限开发环境的日志
  管道（客户端记录器 → 本地 NDJSON），接着运行修复循环：把用户的报告与日志特征相匹配，修复一件事，重新测试。
---

# 调试语音（Debug Voice）

让语音会话在事后变得可读，然后依据证据进行修复。agent 听不到应用的声音；日志是它的耳朵，用户是它的裁判。不录制音频、不记录 token、绝不在生产环境中启用。

适用于任何技术栈。这套管道是一个小小的契约（见下文）；用应用已有的任何技术来实现它。

## 工作流

1. **摸清**应用（只读）。
2. **计划**：写出变更清单，展示出来，然后**停下**。在用户认可之前不做任何编辑。
3. **安装**商定的部分，使用应用的语言、框架和约定。
4. **验证**接收端（sink），把应用交回给用户。
5. 根据日志运行**修复循环**。

## 1. 摸清

找到并记录以下路径：

- 语音客户端：实时事件的接收和发送位置、麦克风采集、音频播放、token 获取。
- 服务端：框架、路由如何声明、共享服务端代码放在哪里、如何读取环境变量、这里“生产环境”意味着什么。
- 约定：语言、模块系统、格式化工具、脚本或任务放在哪里（`package.json`、`Makefile`、`pyproject`、`justfile`）、`.gitignore`。
- 客户端类型：浏览器、移动端、桌面端、CLI。非浏览器客户端仍然 POST 相同的 JSON；单进程应用可以跳过 HTTP，直接追加到文件。
- 处理音频 delta 的位置。必须对其计数，但绝不记录。

## 2. 先计划，然后停下

用真实路径和应用的语言填写以下内容，发布出去，并等待确认或精简后的清单。在此之前不要编辑文件。

```markdown
## 调试语音：计划

新增
- <path>：客户端记录器（批处理、脱敏、刷新），使用 <language>
- <path>：仅限开发环境的接收端 `POST /api/voice/log` → `.voice-logs/<sessionId>.ndjson`
- <path>（可选）：汇总命令 `<cmd>`；否则用 jq 读取 NDJSON

修改
- <语音客户端文件>：挂钩点 start、token、mic、env、ws.*、客户端/服务端事件、
  audio.in（2 秒窗口）、audio.out.first、audio.out、play.stop、stop
- <token 路由>：追加 `server.token { ok, status, ms, upstream }`（绝不记录 token）
- <UI 文件>：在语音状态行和语音错误消息中显示会话 id
- .gitignore：`/.voice-logs`
- <scripts 文件>：一个 `voice:logs` 任务（仅在需要汇总命令时）

记录：事件名称和非音频字段、时间、字节数、麦克风 RMS。
绝不记录：token、API 密钥、原始音频、超过 400 字符的字符串。
除非 `VOICE_LOG=1`，否则在生产环境中关闭。

回复“go”，或删掉你不想要的行。
```

## 3. 安装：契约

匹配应用。使用与周围代码相同的语言、相同的路由风格、相同的格式化工具。按照下面的契约编写各个部分；不要为日志引入第二套语言或工具链。

**会话 id**：来自 UUID 的 8 个小写十六进制字符。接收端接受 `^[a-z0-9]{4,64}$`；它会成为文件名。

**条目**（每行一个 JSON 对象）：

| 字段 | 客户端 | 服务端 |
| --- | --- | --- |
| `t` | 自记录器启动以来的毫秒数 | 无；读取方按 `ts` 对齐 |
| `ts` | epoch 毫秒 | epoch 毫秒 |
| `kind` | `start`、`server`、`client`、`error`、`audio.in`、…… | `server.token`、…… |
| `src` | 无 | `"server"` |
| 其余 | 挂钩的字段，已脱敏 | 挂钩的字段 |

**脱敏，在缓冲之前于客户端应用**：对音频事件类型（`response.output_audio.delta`、`response.audio.delta`、`input_audio_buffer.append`）将 `delta` / `audio` 替换为 `bytes` = 解码后的 base64 长度；超过 400 字符的字符串截断为 400 个字符 + `…[N chars]`；深度超过 4 层的对象 → `"[depth]"`；超过 50 项的数组截断。

**客户端记录器**：缓冲条目；每 1 秒或每 200 条以 `POST <sink> {"sessionId","entries":[…]}` 刷新；停止时带 keepalive 刷新（或平台对应的“页面跳转后仍存活”机制）；吞掉所有传输错误，日志绝不能向语音路径抛出异常。开发环境下还要把条目镜像输出到控制台。

**接收端（sink）**：`POST /api/voice/log`，JSON 请求体。非开发环境或 `VOICE_LOG=1` 之外返回 `404`。如果 `sessionId` 未通过正则校验或 `entries` 不是数组，返回 `400`。每次请求最多追加 500 条，丢弃超过 16,000 字符的任何行，写入 `.voice-logs/<sessionId>.ndjson`，必要时创建目录。返回 `204`。

任意服务端的伪代码：

```
handle POST /api/voice/log:
  if production and VOICE_LOG != "1": return 404
  body = parse json or return 400
  if not regex(body.sessionId) or not list(body.entries): return 400
  mkdir .voice-logs; append join(json(e) for e in body.entries[:500] if len < 16000) to .voice-logs/{sessionId}.ndjson
  return 204
```

客户端记录器的伪代码：

```
logger(sessionId, sink):
  buffer = []; started = now()
  log(kind, data): buffer.push({ ...redact(data), t: now() - started, ts: epoch_ms(), kind }); schedule flush (1 s timer, or immediately at 200 entries)
  server(event, extra): log("server", { ...redact_event(event), ...extra })   # never per audio delta
  client(event):        log("client", redact_event(event))                    # never per audio chunk
  error(where, err, extra): log("error", { where, name, message, ...extra })
  flush(final=false): POST sink {"sessionId","entries": buffer}; buffer = []; ignore all errors; keepalive when final
  close(): flush(final=true)
```

## 挂钩点

`kind` 和字段；其结构在每种语言中都相同。

| 时机 | `kind` 和字段 |
| --- | --- |
| 会话开始 | `start { url, target_rate }` |
| token 获取成功 / 失败 | `token.ok { ms }` / `error { where: "token", name, message, ms }` |
| 麦克风授权 / 拒绝 | `mic.ok { ms, label, settings }` / `error { where: "mic", … }` |
| 音频图就绪 | `env { ua, mic_rate, mic_state, play_rate, play_state, capture_frames, target_rate }` |
| Socket | `ws.connecting`、`ws.open { ms }`、`ws.error`、`ws.close { code, reason, wasClean, by }` |
| 发送的每个事件，音频块除外 | `client { …redacted event }` |
| 接收的每个事件，音频 delta 除外 | `server { …redacted event, phase }` |
| 阶段变化（去重） | `phase { phase }` |
| 麦克风块，每 2 秒聚合 | `audio.in { chunks, bytes, rms_max, rms_avg, pending, mic_state, phase }` |
| 打开时发送的预打开缓冲 | `audio.flush { chunks }` |
| 某个响应的第一个音频 delta | `audio.out.first { response_id, bytes, since_response_created_ms, since_speech_stopped_ms, play_state }` |
| `response.done` | `audio.out { response_id, status, deltas, bytes, audio_ms, wall_ms, max_gap_ms, queued_ms, underruns, drain_ms_max }` |
| 打断（barge-in）停止 | `play.stop { reason, dropped_ms }` |
| 用户停止 | `stop { by: "client", phase }`，然后刷新 |
| token 路由（服务端） | `server.token { ok, status, ms, upstream }` |

在消息处理器中：如果事件是音频 delta，就统计字节数和间隔并播放它；否则将其作为带当前阶段的 `server` 记录下来，然后执行原有处理。保留来自 `input_audio_buffer.speech_stopped` 的 `speechStoppedT` 和来自 `response.created` 的 `createdT`；在第一个 delta 时报告这两个时间距离。在播放器中，当下一个预定时间在本轮响应进行中已经过去时，计为一次 underrun，跟踪最大的 drain，在 `response.created` 时重置。

UI：在语音状态行中显示 id（`Listening · session ab12cd34`），并在语音错误消息末尾追加 `(voice session <id>)`，这样用户就能指明是哪一次运行。

## 4. 验证

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST localhost:<port>/api/voice/log \
  -H 'Content-Type: application/json' \
  -d '{"sessionId":"smoke001","entries":[{"t":0,"ts":0,"kind":"start"}]}'   # 204
curl -s -o /dev/null -w "%{http_code}\n" -X POST localhost:<port>/api/voice/log \
  -H 'Content-Type: application/json' -d '{"sessionId":"../x","entries":[]}' # 400
cat .voice-logs/smoke001.ndjson && rm .voice-logs/smoke001.ndjson
```

然后交接：用户在真实应用中测试，并报告他说了什么、听到了什么、什么时候出了问题，以及会话 id。

## 读取日志

以下任意方式均可；都不需要应用的工具链：

```bash
f=.voice-logs/<id>.ndjson
jq -r 'select(.kind|IN("start","token.ok","mic.ok","env","ws.open","ws.close","server.token","stop","error")) | "\(.t // .ts)ms \(.kind) \(.type // "") \(.message // "")"' $f   # milestones and errors
jq -r 'select(.kind=="server") | .type' $f | sort | uniq -c | sort -rn                      # server event counts
jq -c 'select(.kind|IN("audio.out.first","audio.out"))' $f                                  # per-turn latency, gaps, underruns
jq -c 'select(.kind=="audio.in")' $f                                                        # mic windows, rms
```

或者直接读取文件；它按时间顺序每行一个对象。服务端条目只有 `ts`；用 `start` 条目的 `ts` 将它们对齐到客户端时钟。如果用户想要一个汇总命令，就用应用的语言写一个，打印：里程碑、服务端事件计数、每个 `audio.out` 轮次一行及其 `audio.out.first` 延迟、麦克风窗口总计、错误和关闭，以及最后 25 条条目。

## 5. 修复循环

1. 如果应用还没有 `.voice-logs` 管道，先**埋点**（步骤 1–4）。
2. **用户测试**真实应用并描述这次运行。
3. 在出错的 `t` 附近**读取日志**。
4. **匹配**症状 → 特征 → 修复（见下表）。没有匹配的特征：先增加日志，重新测试，然后再修复。
5. **只修一件事**，重新测试，确认特征在新日志中消失。
6. **记录下来**：在“已通过会话确认”下追加一行已验证的记录。如果修复改变了语音的构建方式，也更新 `/add-voice`。

## 症状 → 日志特征 → 修复（起始行）

| 症状（用户） | 特征（日志） | 修复 |
| --- | --- | --- |
| 静音，但出现转录文本 | `env.play_state` 或 `audio.out.first.play_state` = `suspended` | 在用户手势内创建并恢复播放音频上下文；每个会话一个上下文，而不是每轮一个 |
| 助手打断自己 | 在 `phase=speaking` 时出现 `speech_started`；`audio.in.rms_max` 仅在播放期间升高 | 回声。用耳机确认（如果停止，就是回声）。保持回声消除开启、降低扬声器音量，或在 `speaking` 时屏蔽麦克风发送 |
| 卡顿、断断续续 | `audio.out.underruns` > 0、`drain_ms_max` 很高、`max_gap_ms` 远高于块长度 | 在第一个块播放前安排一小段提前量（150–250 ms）；不要每轮重建音频上下文 |
| 爆音、音调或速度错误 | `session.update` 的 rate ≠ 缓冲 rate；`bytes` 为奇数 | 各处统一一个采样率（`audio.input/output.format.rate`、播放器缓冲）；偶数字节对齐 |
| 从不连接，或立即断开 | 在 `session.updated` 之前出现 `ws.close`；`server.token.ok=false` | 每次点击铸造一个 token（300 秒），协议 `xai-client-secret.<token>`，URL 中带 `model`；读取 `server.token.status` 和 `upstream` |
| 麦克风没有反应 | 每个窗口中 `audio.in.rms_max` ≈ 0；`mic.ok.label` 出乎意料 | 设备或操作系统权限错误；检查 `mic.ok.settings`、`label`、`mic_state` |
| 没有用户转录文本 | 没有 `conversation.item.input_audio_transcription.updated` | 在 `session.update` 中设置 `audio.input.transcription.model: "grok-transcribe"` |
| 首个字很慢 | `audio.out.first.since_speech_stopped_ms` 很高 | 尝试 `reasoning.effort: "none"`；缩短 `instructions`；检查 `token.ok.ms` 和 `ws.open.ms` 的连接开销 |
| 用户文本出现在回复之后 | `...transcription.updated` 的 `t` > `response.created` 的 `t` | 在 `input_audio_buffer.committed` 时创建用户行（`item_id`），在 `updated` 时填充 |
| 开头的字被截断 | `ws.open.ms` 很大且 `audio.flush.chunks` 处于缓冲上限 | 在 socket 之前启动麦克风，缓冲早期音频，提高预打开上限 |

## 已通过会话确认

在修复经过新日志验证后追加。格式：`YYYY-MM-DD · 症状 · 特征 · 修复 · 文件`。

## 规则

- 先计划；在用户认可变更清单之前不做任何编辑。
- 绝不记录 token、密钥或原始音频。音频要转换为字节数。
- 仅限开发环境。除非 `VOICE_LOG=1`，否则接收端在生产环境中返回 404。`.voice-logs/` 已被 gitignore。
- 日志绝不向语音路径抛出异常。
- 聚合计音频；绝不按块记录。
- 每次重新测试只改一处，这样日志才能告诉你哪个修复起了作用。
- 不要编造 xAI 事件名称；在添加特征之前，先在 https://docs.x.ai/developers/model-capabilities/audio/speech-to-speech 中确认。
