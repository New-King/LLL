## 是什么
一句话：**它把杂乱的数据源，变成模型能用的、可检索的上下文。（RAG的整条链路都拆成更细致的零件了，和langchain的设计挺像的，可以在某个环节前先测试结果）**

接入 — 各种格式和来源（文件、网页、数据库、S3）统一成 Document，不用自己写一堆 loader
结构化 — 切块、抽字段、写 metadata、算向量，落进任意向量库
  - 抽字段：让模型读一遍正文，输出结构化的几个值：
```json
 {"title": "pgvector 的 HNSW 索引怎么建", "summary": "讲两种索引的区别", "tags": ["pgvector", "HNSW"]}
```

检索编排 — 召回、过滤、重排、融合、合成、引用，每步都是可替换组件
企业用它的实际理由，主要是省掉接入和胶水代码：这些活儿没什么技术含量，但每个项目都要重写一遍。加上组件可插拔（换向量库、换 reranker 只改配置，不被锁死），以及自带评测与回调钩子，能拿数字证明改动有效。

对。差别就在这里：它把整条链拆成可单独替换的零件，每一步都能单独调整，也能单独评测。

## 待学习

- **父子块检索（small-to-big）** —— 用小块召回，返回时补上所在的大块（用在：答案要靠前后段落才读得懂，切大了召回不准、切小了上下文又碎）
- **查询改写 / HyDE** —— 先把问题改写，或让模型生成一段假答案再拿去检索（用在：用户问得口语化、跟文档措辞差距大，怎么调 topK 都召不回来）
- **Agentic RAG** —— 把检索包成工具交给 agent 自己决定调不调（用在：一个问题要查好几轮，或要在多个数据源之间自己挑）
- **RAGAS 四指标** —— faithfulness / answer relevancy / context precision / context recall（用在：改完切块或重排，需要证明是变好还是变差）
- **评测集扩充** —— 从真实提问取样，标注「该命中哪块 / 该拒答」（用在：手上只有一两条测试问题，改什么都心里没底）
- **图索引** —— 抽实体关系建图，再对社区做摘要（用在：答案要跨多篇文档拼起来，或要回答「这批文档都在讲什么」）

## 常见检索器

- **向量检索器**（`VectorIndexRetriever`）—— 按「意思像不像」找。问「怎么切块」，能搜到写「分块粒度」的段落，字不同也能中；缺点是型号、编号、专有名词容易漏
   ``index.as_query_engine() 或 as_retriever() 不传 retriever_mode 时，默认就走它；similarity_top_k 默认是 2，这通常偏少，实际使用时一般会自己调到 5–10``
   
- **关键词检索器**（`BM25Retriever`）—— 按「字面有没有」找，要自己把 nodes 传进去建倒排表。专有名词、数字、缩写很准，但同义词完全不认

- **融合检索器**（`QueryFusionRetriever`）—— 两路一起跑，结果按排名合并（RRF）；也能让模型把问题改写几个版本分别查，哪路漏了另一路补

- **自定义检索器**（继承 `BaseRetriever` 实现 `_retrieve`）—— 查 SQL、调内部接口都行，只要返回 `list[NodeWithScore]`

怎么选：默认用向量检索；出现「专有名词搜不到」就加 BM25 一起融合；数据本来在数据库里，才自己写一个。

---

- 一条链：Index → 检索器 → 引擎 → `query()`。课里看着乱，只是每课造引擎的入口不同。

## 核心对象

- `Document` / `Node` —— 原文 / 切好的块，都带 metadata
- `VectorStoreIndex` —— 存向量的索引，`from_vector_store` 可从已有库直接建
- `QueryEngine` —— 对外提问的入口，`as_query_engine` 或 `xxx.from_args` 产出
- 检索器 —— 有 `retrieve()` 的对象，只找不答，返回带分数的 `NodeWithScore`

## 三种造引擎的入口

- `index.as_query_engine(...)` —— 检索器由 index 出，最常用
- `RetrieverQueryEngine.from_args(retriever)` —— 检索器自己造的（融合等）
- `CitationQueryEngine.from_args(index, ...)` —— 要带编号引用

## 两个容易混的方法

- `retrieve(QueryBundle(q))` —— 只检索 + 后处理，返回 `list[NodeWithScore]`；传字符串会报 `'str' object has no attribute 'query_str'`
- `query(q)` —— 检索 + 后处理 + 合成，返回 `Response`，`.source_nodes` 是引用的块

## 配置是叠加的

- 换引擎类时，`similarity_top_k` / `filters` / `node_postprocessors` 照搬，不用重写
- 典型叠加：先 `SimilarityPostprocessor(similarity_cutoff=...)` 砍分，再 `LLMRerank(top_n=...)` 精排

## 几个坑

- 过滤是硬排除，不是加权：只留 `type=notes` 去问 blog 的内容，模型只能拒答
- 引用编号对应 `citation_chunk_size` 重切后的块，不是原始 256 的块
- `chunk_size` 上限由嵌入模型定，bge-large-zh 是 512 token
- `response_mode` 在 4 块的语料上看不出差别，块多了才有区别

## 定位

- 1–6 课 = 单次问答这条链（读 → 切 → 建索引 → 检索 → 重排 → 生成）
- 7–9 课 = 工程化（`IngestionPipeline` / `Workflow` / 评测）
- 面试不问参数，问取舍：为什么先阈值再重排、RRF 为什么按排名、召回没中怎么排查、改完怎么证明变好了
