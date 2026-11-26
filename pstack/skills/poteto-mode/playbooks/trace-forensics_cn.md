### Trace 取证

**你负责基于产物做出诊断。**

与对活进程插桩的 **Runtime forensics** 不同。这里的捕获已经存在。产物是一个固定的数据集，读取它，不要重新运行它。保持工具通用：对 cpuprofile 和 `.json.gz` 使用 DevTools 或 trace 解析器，对 spindump 使用文本编辑器，对 heapsnapshot 使用你的 heap 工具。

1. 识别格式并用正确的工具加载它。在 subagent 中解析大型产物（**principle-guard-the-context-window** skill），把精简后的发现留在主线程。根据 Subagents 章节为每个 subagent 选择合适的推理级别。
2. 把原始产物转换成可查询的形式。将 trace 或 heap snapshot 导入 sqlite，每个 sample、frame 或 node 一行。先达到可查询的形态再阅读。
3. 缩小到原因。查询占用时间最多的 frame，沿调用树走到热点路径。对于泄漏，沿持有链从泄漏对象追到 GC root。对于 spindump，找到卡在 CPU 上或被阻塞的线程及其等待原因。
4. 归因到源码。通过产物自带的符号把热点 frame 映射到文件、符号和行。没有源码映射的 frame 还不算诊断。要么解析出符号，要么直说产物没有携带符号。
5. 如果有成对的捕获，对比前后产物以确认。否则，把该发现标记为产物所支持的最强假设，而不是已确认的原因。
6. 交回带引用的诊断，除非被要求否则不做修复。原因确定后路由到 Bug fix 或 Perf issue。吞吐量检查点保持一行：`throughput checkpoint: n/a, read-only forensics`。

**回复：** 产物及其格式、精简后的发现、源码位置、产物路径，以及是否有成对捕获对其进行了确认。
