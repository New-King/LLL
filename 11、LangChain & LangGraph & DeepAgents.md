## 一句话定位

| 包                      | 是哪一层        | 决定什么                                                   |
| ---------------------- | ----------- | ------------------------------------------------------ |
| `@langchain/core`      | 公共底座        | 消息长什么样、模型与工具接口长什么样（`invoke` / `stream` / `batch` 统一约定） |
| `@langchain/langgraph` | 运行时引擎       | 图怎么跑、状态怎么存、跑一半怎么停、thread 怎么续                           |
| `langchain`            | agent 的默认实现 | 循环怎么转、每轮前后插什么钩子、工具怎么定义                                 |
| `deepagents`           | 预置架构        | 开箱给哪些工具、什么系统提示、文件写到哪（backend）                          |
|                        |             |                                                        |

## 时间线

- 2022：LangChain 先出，用「链 + `AgentExecutor`」做 agent
- 2024：LangGraph 独立，用图表达循环 / 状态 / 人工介入（链式结构做不了）
- 2025–2026：LangChain 1.x 重构，agent 部分改成「建一张图交给 LangGraph 跑」，`AgentExecutor` 淘汰 → `createAgent`
- 结论：**先出现的框架，后来换了内核**。这在软件里很常见，不矛盾

两层关系要分开看：

- **时间上**：LangChain 先有
- **结构上**：LangChain 的 agent 部分建在 LangGraph 上

## 依赖方向（是调用，不是转译）

```
deepagents              createDeepAgent：配默认工具 / 中间件 / backend
     ↓ 调用
langchain               createAgent：把「模型 ↔ 工具」循环建成图 + 中间件
     ↓ 调用
@langchain/langgraph    图、状态、Pregel 引擎、checkpoint、interrupt
     ↓ 使用
@langchain/core         消息类型、模型接口、Runnable、tool
```

## 没有「转译」这回事

- 图是**解释执行**：模型调完，边指向哪个节点就执行哪个节点的函数
- `compile()` 不生成新代码，只做校验 + 挂 checkpointer，返回可执行对象
- 「LangGraph 转译成 LangChain 能处理的代码」不成立 —— 是层层调用，不是代码转换

## 谁在跑

`invoke` 不是 `langchain` 主包提供的，是 **`@langchain/core` 的 `Runnable` 接口**定的。Runnable 规定三个方法：

| 方法 | 语义 | 返回 |
|---|---|---|
| `invoke(input)` | 跑一次，等完整结果 | 结果对象 |
| `stream(input)` | 跑一次，边产出边返回 | 异步迭代器 |
| `batch([a, b, c])` | 一次传多个输入 | 结果数组 |

名称和行为统一，输入输出类型各不相同，所以到处都是 invoke：

| 对象 | 提供者 | 背后的 invoke 做什么 |
|---|---|---|
| `ChatDeepSeek` | `@langchain/deepseek` | 发一次 HTTP 请求 |
| `createAgent` 返回的 agent | `langchain` | 跑 agent 循环（本质是图） |
| 编译后的图 | `@langchain/langgraph` | 跑 Pregel 循环 |
| `createDeepAgent` 返回的 agent | `deepagents` | 同上（本质是图） |

于是：**core 定规矩，各层各自实现**。只用 `@langchain/langgraph` + `@langchain/core`（不装 `langchain` 主包），编译图照样有 `invoke`。

判谁在跑看**类型**，不看方法名：`createAgent` 与 `createDeepAgent` 返回的都是编译好的图。

## 三个包各自的入口

```ts
// langgraph：自己画图
const graph = new StateGraph(State)
  .addNode("llm", llmCall)
  .addEdge(START, "llm")
  .compile({ checkpointer });

// langchain：不画图，要循环
const agent = createAgent({
  model,
  tools: [getWeather],
  middleware: [todoListMiddleware()],
});

// deepagents：连循环带默认配置
const agent = createDeepAgent({
  model,
  backend: new CompositeBackend(new StateBackend(), {}),
  subagents: [researcher],
});
```

## 常见误解

| 说法 | 实际 |
|---|---|
| LangGraph 是 LangChain 的一个模块 | 曾是子项目，2024 独立；现在 LangChain 反过来用它的引擎 |
| DeepAgents 是新框架，替代 LangGraph | 它不实现循环，只是把 LangGraph + langchain 配好默认值 |
| 层与层之间会转译 | 没有转译，只有调用与运行时调度 |
| `invoke` 是 langchain 主包的 | 是 `@langchain/core` 的 `Runnable` 接口；主包只是它的使用者 |
| 待办是 DeepAgents 的功能 | `todoListMiddleware` 来自 langchain；Python 版默认挂上，TS 版要自己挂 |
| 加了子 agent 更安全 | 子 agent 只收窄工具集；命令在哪跑由 backend 决定 |

## 常说的「三件套」是另外三个

| 产品 | 形态 | 管什么 |
|---|---|---|
| LangChain | 包 | 写 agent、接模型、工具与中间件 |
| LangGraph | 包 | 编排运行时：图、状态、持久化、中断 |
| LangSmith | 平台（不是包） | 观测、评测、数据集 |

`deepagents` 是第四层（预置架构），不属于这个三元组。所以三者不是「平起平坐的三剑客」，而是叠层，有依赖顺序。

## 比喻

- core = 标准零件（螺丝、接口规格）
- langgraph = 底盘 + 传动（把零件组装成能跑的机器）
- langchain = 装配线（常用配置：模型 + 工具循环 + 中间件）
- deepagents = 出厂车型（导航、行车记录仪都装好了）

装车用的是零件，不是把整台车翻译成零件清单。

## 判断口诀

- 问「循环怎么转、钩子插哪」→ langchain
- 问「状态存哪、怎么续跑、怎么挂起」→ langgraph
- 问「默认给了哪些工具、文件写哪」→ deepagents
- 问「消息长什么样、为什么到处都有 invoke」→ core
