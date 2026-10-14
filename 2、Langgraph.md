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
   
8、记忆：compile({ checkpointer }) + { configurable: { thread_id } }；
      #「比手动拼-messages-多做两件事」 
      - 不用自己拼：只发新消息，历史按 thread_id 从 checkpointer 里读回来。
      - 存的是「每一步的完整状态快照」，不只是 messages —— 所以才多出「挂起后恢复」「从某一步重跑」这些能力

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
    与 checkpointer 的区别（两个维度）：
    - 范围：checkpointer 只限一条 thread；Store 跨 thread
    - 性质：checkpointer 存图的执行快照（框架自动写，用于恢复/续跑）；
            Store 存要长期记住的数据（自己 put / search，用到时才读）
    #一句话：前者是运行存档，后者是长期资料
    用法：
    - 挂载：compile({ checkpointer, store: new MemoryStore() })
    - 传运行时数据：invoke(..., { context: { userId } })
    - 写：runtime.store.put([userId, "memories"], key, value)
    - 读：runtime.store.search([userId, "memories"])   # 前缀匹配，默认最多 10 条
          search(..., { query: "..." }) 
    
