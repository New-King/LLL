1、定义 State（共享状态，定义各种字段，每个字段可以有默认值和自己的合并规则 reducer）

2、创建 Graph（传入 State 对象）

3、定义 Node 节点（普通函数），每个节点必须接收 State 作为第一个入参，
   返回值是部分 State 对象，不需要更新 State 就返回 {}；
   LangGraph 也提供预置节点，如 new ToolNode，返回 { messages: [...] }
   
4、如何编排：图对象 + node + edge

5、连边示例，使用函数addEdge创建边，START为默认开始节点，END为终点，无法改变。
   addEdge(START, "第一个节点") 是入口
   addEdge("a", "b") 是顺序边
   addEdge("x", END) 是出口
   【start->a->b->end】
   
6、分支与循环：addConditionalEdges("节点", 路由函数) 
   路由函数读 state，返回「下一个节点名」或 END；返回自己就是循环
   #addConditionalEdges 可以理解为特殊的addEdge函数，它决定下一步是到下一个节点还是返回上一个节点

7、状态怎么更新：节点返回的字段按各自的 reducer 处理
   （默认覆盖 / MessagesValue 追加 / ReducedValue 自定义）
   
8、检查点checkpointer：
       graph.ts：compile({ checkpointer })   script.ts:  { configurable: { thread_id } }；
      #「比手动拼-messages-多做两件事」 
      - 不用自己拼：只发新消息，历史按 thread_id 从 checkpointer 里读回来。
      - 存的是「每一步的完整状态快照State」，不只是 messages —— 所以才多出「挂起后恢复」「从某一步重跑」这些能力
      - 某一时刻的完整状态快照 + 它是从哪开始的 + 下一步是哪个节点。

9、人工介入（Human-in-the-Loop）：interrupt + Command
   在节点或工具里调 interrupt(payload) 让图停下来等人：
  **const config = { configurable: { thread_id: "stream-1" } }**
   - 停下 = invoke 立刻返回，结果里多一个 __interrupt__（payload 就在里面）
   - 前提：图必须有 checkpointer，且恢复时用同一个 thread_id
   - 恢复：agent.invoke(new Command({ resume: "approve" }), config)
     resume 的值就是 interrupt() 的返回值
     #resume 不能传 false / 0 / ""：JS 版会当成「没传 resume」直接报错
   - 挂起不是卡住：终端不会自动弹出问题，要自己接（readline 问人，或网页上的批准按钮）
   - 恢复时节点会从头重跑 → interrupt() 之前的副作用必须幂等
   #中断时最后一条 AI 消息 content 通常是空的（模型只给了 tool_calls），
    真正的最终回答要恢复后再跑一轮才有
   #Command 也能从节点返回：update（改状态）+ goto（换路由），
    用它必须在 addNode 的 ends 里声明可能跳到的节点；
    只换路由、不改状态时，用条件边更合适


10、长期记忆（Store）：跨 thread 记住用户
    存要长期记住的数据（自己 put / search，自己要做记忆策略存什么自己定）
    用法：
    - 挂载：compile({ checkpointer, store: new InMemoryStore() })
    - 传运行时数据：invoke(..., { context: { userId } })
    - 写：runtime.store.put([userId, "memories"], key, value)

	① 不带 query —— 就是"列出"：按 namespace 前缀把条目拿出来
	runtime.store.search([userId, "memories"])  //该namespace下的前10条
	runtime.store.search([userId])              //取全部的前10条

	② 带 query —— 语义检索：按相似度排序(需要配置embedding模型)
	runtime.store.search([userId, "memories"], { query: "用户喜欢什么风格" })
    new InMemoryStore({ index: { embeddings, dims: 1536 } })

 
**官方两种推荐策略

① 维护一份载体（如文本或json，每次都传并更新，key传同一个）

② 集合（每次存新的关键信息，用的时候只检索出相关的几条附带，key每次都是新的）
    例：
	[user-1, memories] / m1 → "用户叫小王"
	[user-1, memories] / m2 → "偏好：回答简短"
	[user-1, memories] / m3 → "正在学 LangGraph"
	[user-1, memories] / m4 → "上周问过时间旅行"
    ...

|     | Profile（一份）                 | Collection（一堆）             |
| --- | --------------------------- | -------------------------- |
| 写入  | 每次改整份 → **越大越容易改坏**         | 加一条就行，简单                   |
| 读取  | **整份塞进 prompt**（越大越占 token） | 按需检索，只取几条                  |
| 难点  | 更新时漏掉/改错                    | **删除和更新**哪条（模型容易只加不改）、检索要准 |
| 召回  | 信息都在一起，不会漏                  | 检索不好就取不到                   |

