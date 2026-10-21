## 一句话定位

- **agent**： 模型 + harness + 工具**，跑在 runtime 上。
- **harness**：你设计的驱动层，管编排、调模型、写提示词、处理工具结果、裁剪上下文、出错回退。
- **runtime**：框架提供的运行基础设施。执行调度、状态存储、通信通道、上下文供给。
- **trace**：可观测系统里的记录数据，来自 runtime 发出的事件，但采集、存储、展示由外部系统负责（LangSmith、OpenTelemetry、Phoenix）。

## 关系

- harness **依赖** runtime：节点里 `runtime.store.put(...)`、`config.writer(...)` 用的都是 runtime 的能力。
- runtime **不包含** harness：runtime 不决定何时调模型、提示词写什么。
- runtime **只发出事件**，不存储也不展示；配了 `LANGSMITH_TRACING`，LangSmith 才把这些事件收集成 trace，不配照跑、只是没有记录。

## 汽车比喻

- 模型 = 引擎
- harness = 底盘、传动、方向盘（决定往哪开、怎么开）
- runtime = 道路、加油站、车库（保证能跑、能停、能再启动）
- 工具 = 挂在底盘上的执行部件
- trace = 行车数据；LangSmith = 外挂修理厂，负责收集和查看这些数据，不在 runtime 里


