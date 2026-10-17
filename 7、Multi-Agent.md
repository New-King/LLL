- 是什么：多个 agent 分工干活，不是一个全干（multi-agent / MAS）

- 两种协作形态
  主从：主 agent 拆任务 → 派给多个 worker → 汇总
  对等：agent 平级，互相协作

- 按跨度分三层
  同进程：临时子 agent（一次性）
  同机多身份：持久 agent（有名字、有记忆、能恢复）
  跨机器 / 框架：标准协议互通（A2A）

- 六个设计问题
  上下文怎么传 / 结果怎么汇合 / 怎么防循环 / 挂了怎么办 / 成本与并发 / 状态放内存还是放库

- 与子 agent 的关系：子 agent = 最简单的多 agent（主从）

- 什么时候值得：任务可拆且互相独立 / 一个上下文装不下 / 要并行

```mermaid
flowchart TB
    U["用户 / 外部系统"]

    subgraph Coord["协调层（成熟系统的核心）"]
        SUP["主 agent / Supervisor<br/>规划 · 拆解 · 汇总"]
        KB[("任务板 / 队列<br/>持久化状态")]
        DISP["调度器 Dispatcher<br/>依赖 · 优先级 · 并发上限 · 重试"]
    end

    subgraph Exec["执行层：worker 池"]
        W1["worker 1<br/>独立上下文 + 独立预算"]
        W2["worker 2"]
        W3["worker N"]
    end

    ENV["执行环境<br/>文件 / 终端 / 沙箱"]

    U --> SUP
    SUP -->|写任务| KB
    KB -->|取可执行任务| DISP
    DISP -->|派发| W1
    DISP --> W2
    DISP --> W3
    W1 --> ENV
    W2 --> ENV
    W3 --> ENV

    W1 -.->|结果 / 评论 / 附件 / 阻塞| KB
    W2 -.-> KB
    W3 -.-> KB

    KB -->|汇总| AGG["汇总 / 评审 agent"]
    AGG --> SUP
    SUP -->|最终结果| U

    HUM["人（HITL）<br/>评论 · 审批 · 解除阻塞"] <-.-> KB
    W2 <-.->|跨机器 / 框架：A2A| PEER["外部 agent"]
```

```mermaid
stateDiagram-v2
    [*] --> pending
    pending --> running: 调度器派发
    running --> done: 完成
    running --> blocked: 等依赖 / 等人
    blocked --> running: unblock
    running --> failed: 失败
    failed --> pending: 重试
    running --> pending: 进程挂了 → reclaim
```



