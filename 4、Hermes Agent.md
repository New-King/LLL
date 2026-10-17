
## 1、基本介绍

- 是什么？一个类似 openclaw 这种个人助理的东西
- 内置了很多 Geteway（链接两种不同网络协议之间的节点）和各种常用的插件

---

## 2、循环步骤

```mermaid
flowchart TD
    U["1. 用户发送消息"] --> BC["2. Build Context"]
    BC --> LLM["3. Context 发给 LLM"]
    LLM --> DEC{"4. 是否调用 Tool?"}
    DEC -->|是| EXEC["5. 执行 Tool"]
    EXEC --> FEED["6. Result 回灌 LLM"]
    FEED --> LLM
    DEC -->|否| RESP["7. 最终回复"]
    RESP --> MU["8. Memory Update"]
    MU --> DONE["本轮结束"]
```

### 2.1、如何 Build Context（分层构建 + 拼接）

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 24, "rankSpacing": 36}, "themeVariables": {"fontSize": "15px"}}}%%
flowchart TB
    subgraph Cached["Cached System Prompt（Session 启动时冻结，同 Session 内尽量不变）"]
        ST["stable: SOUL / Tools·Skills 指引 / 环境·平台 hint"]
        CT["context: 可选 system_message + Project Context 一种"]
        VOL["volatile: memory.md + user.md 快照<br/>+ 可选 External 块 + 时间/session 行"]
        ST --> CT --> VOL
    end

    subgraph PerCall["本轮 API 调用层（可变）"]
        HIST["消息历史（可被 Compression 摘要替换）"]
        CUR["本轮 user 消息"]
        EPH["可选: ephemeral overlay / 外部召回挂在本轮 user 上"]
        HINT["可选: 子目录 AGENTS.md 挂在 tool result 后"]
    end

    Cached --> LLM["发给 LLM"]
    PerCall --> LLM
```

#### 部分含义解释

**冻结快照模式：** 系统 prompt 注入在会话开始时捕获一次，会话中途不会改变。这是有意为之——目的是保留 LLM 的前缀缓存以提升性能。当 Agent 在会话期间添加或删除记忆条目时，更改会立即持久化到磁盘，但要到下一次会话开始时才会出现在系统 prompt 中。工具响应始终显示实时状态。

何时恢复：会话恢复、新会话、上下文压缩、模型切换（provider 的缓存空间 / 机制都不同）

- **soul**：你自己设定的 agent 身份 / 人格 / 行为准则
- **system_message**：调用方传入的基本信息和行为准则（cli / 网页前端、外部程序调用 - python 脚本、外部 agent、父调用子 agent、切换模型）
- **project content**：/hermes.md、Agents.md...
- **externa**：外部的长期记忆
- **ephemer overlay**：(临时指令，本质是在 system 尾部添加一段非冻结的提示词)
    - 改人格（不影响 soul，内置了 /personality 指令 14 种人格）
    - 改环境变量
    - 子 agnet 委派（父 agent 委派了什么任务，和冻结的 sp 是分开的）
- **外部召回**：搜索 / api 数据 / 程序获取
- **tool_reult**：上轮工具调用结果，openai 有独立的 role=tool，其他的拼接到 role=user 后
- **子目录 Agents.md**：调用什么工具，该工具的 md 文档会拼接到 tool result 后
- **知识库获取方式**：请求前自动获取（归属外部召回）、走 tool 工具（归属 tool_reult）

---

### 2.2、压缩机制

1. 如何预测？用 API 返回的真实 token 用量（不是估算）
2. prompt_tokens（本次请求的总输入 token）≥ 50% × 模型 context_length → 触发
3. 保护前 3 轮（system prompt、最初的请求、第一次回答）
4. 保护最后 N 轮（默认 20 条消息 ≈ 10 轮）
5. 中间的轮次用便宜快的模型摘要
6. 摘要作为一条 user 消息插入，对话无缝继续

#### 摘要长什么样？

一段结构化 markdown，开头声明"仅作背景，只响应之后的新消息"。区块：目标 / 约束 / 已完成（带日期）/ 进行中 / 阻塞 / 关键决策 / 相关文件 / 下一步。每条一两行，只保留有价值的结论。二次压缩是更新上一份，不是重写。

---

### 2.3、[[Memory]]

https://hermes-agent.nousresearch.com/docs/zh-Hans/user-guide/features/memory

**特点**

- 有效信息：只记忆紧凑、信息密度高的信息
- 重复防护：记忆系统会自动拒绝完全重复的条目
- 安全扫描：记忆被接受之前会扫描注入和数据外泄模式，防止被注入系统 prompt
- 外部记忆：对于超出 MEMORY.md 和 USER.md 范围的更深层持久化记忆，Hermes 内置了 8 个外部记忆提供商插件，外部提供商与内置记忆**并行**运行（而非替代）

**[[Memory由三部分组成]]**

1. **内置记忆文件**（用于 system prompt，存储时机：触发 memory 工具 / turn 结尾）

   每次对话结束（Turn），会进行 memory update，Agent 会自动更新以下信息：

    - memory.md（环境、工具、项目约束...）
    - user.md（用户信息与偏好）
    - soul.md（用户对 agnet 的设定，行为准则规范）

2. **SQLite**（关系型数据库，存对话数据，存储时机：多个时机，增量存储：游标 + 增量，用于会话恢复和跨会话检索）

    **存什么？**

    - 聊天记录
    - 会话元数据（session_id / session_key / 标题、时间戳、profile、token 用量）
    - 本 turn 对话的完整 system prompt
    - **FTS5 全文索引**（作用：让历史对话变成"可搜索"，同时不占常驻上下文）
        - 非向量检索，FTS5 = SQLite 自带的全文检索扩展（Full-Text Search 5）
        - "虚表"：查询时看起来像一张表，能 `WHERE content MATCH '关键词'`，但底层是倒排索引（词 → 命中哪些行），所以查关键词快，不用逐行扫
        - Hermes 把每轮消息文本写进去 → session_search 就靠它（使用二元分词）

    **触发时机？**

    - 消息落库（用户输入 / 每次工具轮 / turn 收尾 flush）
    - 消息被删 / 被压缩归档 → 触发器同步删除对应索引行
    - 启动时：索引是否 stale（比如分词器能力变化）

    **二元分词**

    - **Hermes 专门加载 cjk_unicode61（SQLite FTS5 的 unicode61 分词器 + 中文二元切分）**
    - 二元分词（bigram）就是绕开词典的做法：不判断词，机械地每两个字切一刀：
        - 今天天气不错 → 今天 / 天天 / 天气 / 气不 / 不错
        - 搜索时查询也切一遍：查 天气 → 切成 天气 → 能对上
    - 代价：索引膨胀，精度略差但召回好

3. **External Memory**（外部的记忆，与内置记忆叠加，官方有提供 / 也可自己实现 provider）

---

## 3、部分内置工具

### 3.1、todo_list（任务清单）

**任务复杂或者多步的时候，先列一个 todo 清单，再一步步做。**

- 这只是"建议"，不是强制：靠系统提示 + 工具说明劝模型先规划，列不列由模型自己决定。
- 一个工具两用：给 todos 就是写 / 更新清单，不给就是看当前清单。
- 清单存在会话内存里，也不塞进 system prompt，所以不会拖慢响应（保住缓存）。
- 上下文被压缩时，没做完的 todo 会被重新放回上下文，模型不会忘了要干嘛。

### 3.2、session_search（跨会话检索）

**能搜索你以前所有的对话，不只当前这一次。**

- 一个工具靠参数自动分模式：给关键词=搜，给 session_id=读那一段，什么都不给=列会话。
- 排序会把定时任务、子 agent 产生的对话降权或隐藏，免得搜出来全是机器人的废话，你真正聊的被埋掉。
- 每条消息有长度上限，防止一条超长的工具结果把 token 撑爆。

### 3.3、clarify（中途问用户）

**模型拿不准时，可以停下来给你弹选择题，而不是瞎猜。**

- 最多 4 个选项，界面永远多给一个"其他（自己填）"。
- 第一个选项自动标 "(Recommended)"；但你选完回传给模型的答案里会把这个标签去掉（它只是给你看的）。

### 3.4、execute_code（写脚本一次调多个工具）

**让模型写一段 Python脚本（本工具参数），在脚本里连续调好几个工具，最后只把 print 出来的结果拿回来。**

- 什么时候用：需要连调 3 个以上工具，中间还要做判断 / 循环 / 过滤的时候。
- 好处：本来要来回好几轮（调工具 → 看结果 → 再调），现在一轮搞定，而且中间结果不进上下文。
- 脚本跑在一个常驻的解释器里；输出太长会自动存成文件，只给路径 + 头尾预览，需要时再去读。
- 场景：比如要读10个文件内容，不可能每次都把文件内容又塞回tool_result然后继续下一轮

### 3.5、tool_search / tool_describe / tool_call（三个"桥接工具"：按需加载工具）

**工具太多时，不把所有工具的说明一股脑塞给模型，而是先"搜工具"，用到哪个再加载哪个。**

- 平时不出现；只有挂了很多 MCP / 插件工具时，Hermes 才把它们收起来，换成这三个工具按需查。
- 用法：tool_search 搜 → tool_describe 拿用法 → tool_call 执行。
- 搜索是关键词匹配（BM25），做了英文词干，所以 issues 能匹配到 create_issue。
```
	模型调 tool_search("我想干的事") → 
	拿回可能的工具名 tool_describe(那些名字) → 
	拿回这个工具怎么传参 tool_call(名字, 参数) → 
	Hermes 按这个名字去真正执行
```

### 3.6、delegate_task（子 agent）

**把一个任务丢给另一个独立上下文的子 agent 去做，只有它的最终总结会回到主 agent。**

- 怎么触发：主 agent 自己调这个工具——Hermes 里"派子 agent"就这一个入口。
- 派出去后立刻拿回一个 id，你和主 agent 都能接着干别的，不用干等。
- 派出去时就已经有一条状态 running 的**记录**（内存 + 落库），所以"还在不在跑"是查记录，不是去问子 agent。
- 跑完时 Hermes **新造一个"完成事件"**：先落库，再塞进一个**全局共享队列**；CLI / 网关在**每轮 turn 收尾时**清这个队列，把"结果 + 当初派它的目标"作为一条**合成的 user 消息**注入会话 → 开新 turn。所以看起来像模型"自己又开口了"，**不是界面提示，也不是模型去轮询**。
- 注入的那条消息里还会提醒模型："你可能已经不需要了，自己判断用不用 / 重派"。
- 子agent下派，默认一层，多层要修改参数 max_spawn_depth
- 如何调用？
  **派发**：传 `tasks=[{goal, context}]`，不写 `action`（数组一个元素 = 一个子 agent）。
  **其他操作**：`action="list"` 看进度、`action="steer"` 追加纠偏、`action="stop"` 停掉；`/new` 或关会话会把整棵子 agent 杀掉。

### 3.7、文件工具（防止乱改）

**模型改一个已存在的文件之前，必须先读过它，否则不许覆盖。**

- 就是防模型凭印象瞎改。
- 同一个文件、同一版本重复读，第二次只返回一句提示（省 token）；但上下文压缩后会清空，保证压缩后重读还能拿到全文。

---

## 4、不是工具，但同样关键的设计

### 4.1、长内容一律"落盘 + 头尾预览"

**任何特别长的内容都不整个塞进上下文，而是存成文件，只给头尾预览和路径。**

- 比如 hook 注入的内容每次请求都会重发，超过 10K 字符就落盘，免得每轮重复烧 token。

