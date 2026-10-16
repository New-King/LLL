
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

> 共同套路：状态放 session 级、大内容赶出 prompt、不确定的动作做成可恢复的

### 3.1、todo_list（任务清单）

- 一个工具两用：传 todos 就写，不传就读；merge=true 按 id 增量改 → 省掉读 / 写两个 schema
- 带 revision 版本号：每次真改 +1，前端能拒收过期更新
- 压缩后重注入：只注回 pending / in_progress（已完成的不注，免得模型重做）
- 刻意不改 system prompt（保前缀缓存）

### 3.2、session_search（跨会话检索）

- 一个工具靠参数切模式：query=搜索 / session_id+around_message_id=滚动 / session_id=读整段 / 无参=浏览
- 排序去偏：cron 会话降权，kanban / 子 agent 会话隐藏，否则定时任务的词会淹没你真实对话
- 每条消息有内容上限，避免一条归档的工具结果把 token 顶爆

### 3.3、clarify（停下来问用户）

- 让模型主动问选择题；最多 4 个选项，UI 永远追加"其他（自己填）"一行
- 第一项自动加 "(Recommended)"，但回传前把展示文字剥掉 → 呈现层和答案分离

### 3.4、execute_code（在脚本里批量调工具）

- 模型写一段 Python，脚本里通过 RPC 调 Hermes 工具，只有 stdout 回到模型 → N 次工具往返压成 1 轮
- 持久 kernel；stdout 超限按头尾截断并落盘，返回文件路径（重跑不如去读）

### 3.5、tool_search / tool_describe / tool_call（工具懒加载）

- 不把所有工具都塞进 schema，用 BM25 在"可延迟工具目录"里搜（做了词干，"issues" 能匹配 create_issue）
- 先搜到 → 再拿 schema → 再执行，工具再多也不挤爆上下文

### 3.6、delegate_task（子 agent）

- 角色 leaf（剥掉 delegate / clarify / memory 等，防无限递归）/ orchestrator（保留 delegate，深度 ≤ 2）
- background=true 只返回一个 id，结果稍后回灌成一条消息
- output_schema 校验失败也不丢结果（标 completed + 原文）

### 3.7、文件工具（防盲改）

- write_file 覆盖已有文件前，本次任务必须先读过整份内容，否则拒绝
- 同文件同版本重复读返回 stub 省 token；压缩后清空，保证压缩后重读能拿到全文

### 3.8、通用套路：长输出一律"落盘 + 头尾预览"

- hook 注入的内容每次 API 调用都会重发，超 10K 字符就落盘给路径，保护前缀缓存
