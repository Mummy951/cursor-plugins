### 调查（Investigation）

**你负责给出答案。规划、路由、撰写。**

调查请求是只读的。它们产出的是带引用的解释或建议，而不是代码改动。

1. 追踪问题所涉及的代码。对于涉及动机的问题，还要经过 **why** 技能路由。
2. 吞吐量检查点保持一行：`throughput checkpoint: n/a, read-only investigation`。
3. 产出输出（Overview / Key Concepts / How It Works / Where Things Live / Gotchas），或者如果请求是在多个备选方案之间做决策，则给出带权衡表的建议。
4. 对答复应用 **unslop** 技能。

不开 PR、不做 babysit、不用 `architect`，除非该调查是代码改动的前置步骤。如果是，就交回给用户并重新路由到 Bug fix 或 Feature。

**答复：** 调查输出。对于"我们确定吗？"这类问题，给出你真实的判断并附上理由。如果前提有误，就提出反对意见（见 Autonomy）。
