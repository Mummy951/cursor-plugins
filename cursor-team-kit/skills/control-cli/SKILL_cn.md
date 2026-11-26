---
name: control-cli
description: 构建或改造一个本地测试框架（harness），用于在无外部服务的情况下驱动、检查和剖析交互式 CLI 或 TUI。适用于 CLI 用户体验检查、启动回归、内存泄漏、卡死、提示流程或终端演示。
---

# Control CLI

使用可重复的本地测试框架（harness）来驱动交互式 CLI，而不是手动反复尝试。如果仓库自带测试/演示框架，优先复用它；否则，使用标准本地工具临时搭建一个测试框架。

## 用途

- 使用确定性的输入复现 CLI/TUI 的 bug。
- 验证键盘操作流程、提示、中断、窗口大小调整行为和终端布局。
- 为 bug 修复捕获修复前/修复后的终端记录（transcript）。
- 剖析启动时间、慢操作、卡死或内存增长。
- 当输出更便于展示而非解释时，录制一段简短的终端演示。

## 测试框架循环

1. 确定被测命令以及最小的可复现工作区。
2. 查找现有的本地测试框架：包脚本、端到端（e2e）测试、演示录制器、expect 脚本或 PTY 辅助工具。
3. 如果没有测试框架，请在隔离的终端会话中使用确定性的环境变量启动 CLI。
4. 在交互之前先捕获当前屏幕。
5. 一次发送一个动作：文本、Enter、方向键、Escape、Ctrl-C、调整窗口大小。
6. 在执行下一个动作之前，等待出现具体的屏幕模式或提示。
7. 保存终端记录（transcript）以及任何性能剖析产物。
8. 干净地终止该会话。

## 测试框架选项

- 仓库原生框架：优先使用已检入（checked-in）的脚本，因为它们了解应用的启动方式、环境变量和提示。
- `tmux`：托管会话、`capture-pane`、`send-keys`、附加/分离（attach/detach）。
- PTY 探针：当 tmux 不可用时使用简短的 Python、Node 或 Expect 脚本。
- 运行时检查器：使用 Node 或 Bun inspector 进行 CPU 剖析、堆快照和实时求值。
- 终端录制器：当用户要求演示时，使用仓库本地的演示工具或兼容 asciinema 的工具。

## 最小 tmux 测试框架

```bash
SESSION="cli-harness-$(date +%s)"
tmux new-session -d -s "$SESSION" -- <command-under-test>
tmux capture-pane -pt "$SESSION"
tmux send-keys -t "$SESSION" "help" Enter
tmux capture-pane -pt "$SESSION"
tmux kill-session -t "$SESSION"
```

对于 Node CLI：

```bash
NODE_OPTIONS="--inspect=127.0.0.1:0" tmux new-session -d -s "$SESSION" -- <node-cli-command>
```

读取终端输出以找到 inspector URL，然后在需要剖析时使用兼容 Chrome DevTools 的工具。

## 最小 PTY 测试框架

当在没有 tmux 或演示框架的仓库中需要确定性等待时，使用 PTY 脚本。除非用户要求添加可复用的测试，否则保持其为临时脚本。

```python
import os
import pty
import select
import subprocess
import time

master_fd, slave_fd = pty.openpty()
proc = subprocess.Popen(
    ["<command>", "<arg>"],
    stdin=slave_fd,
    stdout=slave_fd,
    stderr=slave_fd,
    close_fds=True,
)
os.close(slave_fd)

deadline = time.time() + 30
buffer = b""
while time.time() < deadline:
    ready, _, _ = select.select([master_fd], [], [], 0.25)
    if not ready:
        continue
    chunk = os.read(master_fd, 4096)
    buffer += chunk
    if b"<ready text>" in buffer:
        os.write(master_fd, b"help\n")
        break

print(buffer.decode(errors="replace"))
proc.terminate()
os.close(master_fd)
```

如果 CLI 需要更丰富的终端控制，请使用 `pty.fork()` 或现有的 PTY 库。

## 剖析技巧

- 启动回归：在同一台机器、相同环境和命令下，捕获基准（baseline）和处理后（treatment）的启动耗时。
- 慢操作：启动 CPU 剖析，执行该操作，停止剖析，然后比较自身耗时（self-time）最高的函数。
- 内存泄漏：如可用则强制 GC，拍摄堆快照，重复执行该操作，再次强制 GC，并拍摄另一张快照。
- 卡死：在中断之前，捕获屏幕、活动句柄/资源以及栈/CPU 采样。

## 防护原则

- 优先使用确定性等待，而不是 sleep。如果必须使用 sleep，请说明原因。
- 不要向受控会话发送凭据或破坏性命令。
- 除非仓库已有测试/演示框架，否则将测试框架保持在 `/tmp` 中。
- 不要硬编码来自其他仓库的路径。根据当前仓库的脚本和运行时调整命令。
- 除非用户要求保留，否则清理 tmux 会话、临时目录、inspector 进程和演示产物。
