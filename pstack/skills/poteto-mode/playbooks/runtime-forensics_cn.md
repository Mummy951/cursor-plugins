### 运行时取证

**你负责诊断。对活进程进行插桩，不要仅凭源码推论。** 交付物是带引用的诊断，而不是修复。

1. 通过 control skill 在匹配的界面上捕获实时信号：对空转进程抓取 CPU profile，对泄漏抓取 heap snapshot，对视觉故障抓取 CDP trace。要真实产物，而非猜测。
2. 将产物精简到确凿证据：热点路径上的函数、从泄漏对象到 GC root 的持有链、在没有输入时触发的循环。在 subagent 中解析大型产物（**guard-the-context-window** principle skill），把精简后的发现留在主线程。根据 Subagents 章节为每个 subagent 选择合适的推理级别。
3. 先证明机制，再相信它。通过 CDP eval 在运行中的进程上注入插桩，或者在不重新加载的情况下热修复线上代码，以低成本确认假设。
4. 把发现映射回源码：文件、符号、进行分配或调度的那一行。
5. 吞吐量检查点保持一行：`throughput checkpoint: n/a, read-only forensics`。

**回复：** 捕获的信号、精简后的发现、你如何证明机制、源码位置、产物路径。除非被要求，否则不做修复。原因确定后交回 Bug fix 或 Perf。
