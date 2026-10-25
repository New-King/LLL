- 定位：LlamaIndex 提问只有一条链 —— Index → 检索器 → 引擎 → `query`。课里看着乱，是因为每课换了一种造引擎的方式。

## 三个核心对象

| 对象 | 是什么 | 谁产生 |
|---|---|---|
| `Document` / `Node` | 原文 / 切好的块（带 metadata） | `SimpleDirectoryReader` → `SentenceSplitter` |
| `Index` | 存向量的地方 | `VectorStoreIndex(nodes)` |
| `QueryEngine` | 对外提问的入口 | `as_query_engine` / `xxx.from_args` |

## 检索器：只找，不答

- 输入一个问题，返回一批候选块（`NodeWithScore`，带分数），不写答案
- 有 `retrieve()` 方法的对象就是检索器
- 三种来源：
  - `VectorIndexRetriever`：`index.as_retriever(...)` 出来的，按向量相似找
  - `BM25Retriever`：按字面命中找，要自己从 `nodes` 建倒排表
  - `QueryFusionRetriever`：包住上面两种，按排名融合

## 引擎 = 检索器 + 后处理 + 合成器

造引擎只有三种入口，产物都是 `BaseQueryEngine`，都有 `.query()` / `.retrieve()`：

| 场景 | 写法 |
|---|---|
| 检索器由 index 出 | `index.as_query_engine(similarity_top_k=..., filters=..., node_postprocessors=[...])` |
| 检索器自己造的（融合等） | `RetrieverQueryEngine.from_args(fusion)` |
| 要带编号引用 | `CitationQueryEngine.from_args(index, retriever=..., node_postprocessors=[...], citation_chunk_size=...)` |

`as_query_engine(**kwargs)` 内部就是这两句：

```python
retriever = index.as_retriever(**kwargs)
engine = RetrieverQueryEngine.from_args(retriever)
```

所以它只认自己造的检索器，接不了外部传来的对象。

## 能力是叠加的

换引擎类时，前面课的配置照搬，不用重写：

```python
# 阈值 + 重排，叠到引用引擎上
CitationQueryEngine.from_args(
    index,
    similarity_top_k=8,
    citation_chunk_size=256,
    node_postprocessors=[
        SimilarityPostprocessor(similarity_cutoff=0.6),
        LLMRerank(top_n=3),
    ],
)

# 融合检索器，叠到引用引擎上
CitationQueryEngine.from_args(index, retriever=fusion, citation_chunk_size=256)
```

## retrieve 和 query 的区别

| | 做什么 | 返回 |
|---|---|---|
| `retrieve(QueryBundle(q))` | 检索 + 后处理，不合成 | `list[NodeWithScore]` |
| `query(q)` | 检索 + 后处理 + 合成 | `Response`（`.source_nodes` 是引用的块） |

`retrieve` 必须传 `QueryBundle`，传字符串会在重排那步报 `'str' object has no attribute 'query_str'`（实测）。

## 几个坑

- **过滤会把答案一起排掉**：只留 `type=notes` 去问 blog 里的内容，模型只能拒答 —— 过滤是硬排除，不是加权
- **引用编号对应的是 `citation_chunk_size` 重切后的块**，不是原始的 256 块
- **`chunk_size` 上限由嵌入模型定**：bge-large-zh 输入上限 512 token
- **`response_mode` 在 4 块的语料上看不出差别**：compact 一次拼完、refine 逐块改、tree_summarize 分组汇总，块多了才有区别

## 框架 vs 业务

- 1–6 课 = 单次问答这条链（读 → 切 → 建索引 → 检索 → 重排 → 生成）
- 7–9 课 = 工程化（`IngestionPipeline` / `Workflow` / 评测）
- 面试不问 API 参数，问的是取舍：为什么先阈值再重排、RRF 为什么按排名、召回没中怎么排查、改完怎么证明变好了
