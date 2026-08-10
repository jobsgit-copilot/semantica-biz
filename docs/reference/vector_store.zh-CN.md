---
title: "向量库模块"
description: "FAISS、Pinecone、Weaviate、Qdrant、Milvus 和 PgVector 的统一接口，支持混合搜索。"
icon: "database"
---

**[English](vector_store.md)** · **简体中文（当前）**

`semantica.vector_store` 提供跨所有主要后端存储与搜索向量嵌入的统一 API：

- 通过一行修改即可切换后端：无需更改应用代码
- `HybridSearch` 通过 RRF 或加权平均融合稠密向量相似度与元数据过滤
- `NamespaceManager` 用于多租户结构隔离
- `FAISSStore` 支持 flat、ivf、hnsw 和 pq 索引类型
- 使用并行工作进程批量嵌入并存储；无需重新嵌入即可更新元数据


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `VectorStore` | 统一接口：`store_vectors`、`search_vectors`、`update_vectors`、`delete_vectors` |
| `HybridSearch` | 通过 RRF 或加权平均融合稠密向量相似度与元数据过滤 |
| `MetadataFilter` | 可链式调用的过滤器构建器：`.eq("type", "person").gt("year", 2020).in_list("tag", [...])` |
| `NamespaceManager` | 多租户隔离：为每个项目或用户分配独立的索引命名空间 |
| `FAISSStore` | 本地磁盘或内存：flat、ivf、hnsw 和 pq 索引类型 |
| `WeaviateStore` | 云端或自托管，模式感知 |
| `QdrantStore` | 云端或自托管，支持基于 payload 的过滤 |
| `PineconeStore` | 托管式云端向量数据库，支持 serverless 与 pod 模式 |
| `MilvusStore` | 可扩展的自托管向量数据库 |
| `PgVectorStore` | 带 `pgvector` 扩展的 PostgreSQL：无需额外基础设施 |
| `MetadataStore` | 独立的元数据索引与查询 |
| `SearchRanker` | RRF 与加权平均的结果融合 |

<a id="what-you-get"></a>
## 你将获得

- **VectorStore** — 跨 FAISS、Pinecone、Weaviate、Qdrant、Milvus、PgVector 的统一接口
  - 一行切换后端：无需更改应用代码
  - `add_documents()` 自动嵌入；`store_vectors()` 用于预计算的向量嵌入
- **HybridSearch** — 稠密向量相似度与元数据过滤
  - RRF 或加权平均融合策略
  - 跨独立集合的多源融合
- **MetadataStore** — 按字段值进行丰富的元数据索引
  - 无需重新嵌入即可更新元数据字段
  - OR 和 AND 查询运算符
- **NamespaceManager** — 按租户的结构化命名空间隔离
  - 更快的查询（每个租户的搜索空间更小）
  - 比仅靠元数据过滤的分离更安全
- **批量操作** — 批量添加、删除与元数据更新
  - 可配置 `batch_size` 和 `workers` 的并行嵌入
  - 原地更新向量，无需全量重建索引
- **FAISS 索引类型** — flat、ivf、hnsw 和 pq 索引类型
  - 通过 `FAISSStore.create_index()` 完整控制配置
  - `save()` / `load()` 用于磁盘持久化


<a id="getting-started"></a>
## 入门

**`VectorStore`** 是主入口。开发时使用 `"inmemory"`，**本地生产** 使用 `"faiss"`：

```python
from semantica.vector_store import VectorStore

# 内存（开发 / 测试：无持久化）
store = VectorStore(backend="inmemory", dimension=384)

# FAISS（本地，通过 save/load 持久化到磁盘）
store = VectorStore(backend="faiss", dimension=384)

# 添加纯文本文档（自动嵌入）
ids = store.add_documents(
    documents=["Apple was founded by Steve Jobs.", "Microsoft was co-founded by Bill Gates."],
    metadata=[{"source": "wiki"}, {"source": "wiki"}]
)

# 按文本查询搜索（自动嵌入）
results = store.search("technology company founders", limit=5)
for r in results:
    print(f"{r['id']}: score: {r['score']:.3f}")
```

<Warning>
  **使向量维度与你的嵌入模型匹配。** `dimension` 参数必须与你的嵌入模型的输出大小完全匹配：`BAAI/bge-small-en-v1.5` = 384、`all-MiniLM-L6-v2` = 384、`all-mpnet-base-v2` = 768、`bge-large-en-v1.5` = 1024。不匹配会在插入时抛出错误。
</Warning>

<Tip>
  **文本使用 `add_documents()`，预计算的向量嵌入使用 `store_vectors()`。** `add_documents()` 并行批量自动嵌入。如果你的向量嵌入已经计算好（例如来自微调模型），请直接使用 `store_vectors()` 以跳过重新嵌入。
</Tip>

<a id="quick-start"></a>
## 快速上手

<Steps>
  <Step title="创建向量库">
    ```python
    from semantica.vector_store import VectorStore

    # 内存（开发）
    store = VectorStore(backend="inmemory", dimension=384)

    # FAISS（本地生产）
    store = VectorStore(backend="faiss", dimension=384)
    ```
  </Step>
  <Step title="添加向量">
    ```python
    # 添加文本文档（批量自动嵌入）
    ids = store.add_documents(
        documents=["text one", "text two"],
        metadata=[{"title": "Document 1"}, {"title": "Document 2"}]
    )

    # 直接添加预计算的向量
    ids = store.store_vectors(
        vectors=[embedding1, embedding2],
        metadata=[{"title": "Document 1"}, {"title": "Document 2"}]
    )
    ```
  </Step>
  <Step title="按语义相似度搜索">
    ```python
    # 按文本查询搜索（自动嵌入查询）
    results = store.search("machine learning", limit=10)

    # 按预计算向量搜索
    results = store.search_vectors(query_vector, k=10)

    for r in results:
        print(f"{r['id']}: score: {r['score']:.3f}")
    ```
  </Step>
  <Step title="按元数据过滤结果">
    ```python
    from semantica.vector_store import HybridSearch, MetadataFilter

    mf = MetadataFilter().eq("category", "research").gt("year", 2022)

    # 将 vector_store 传递给构造函数：search() 会自动解析向量
    search  = HybridSearch(vector_store=store)
    results = search.search(query=query_vector, k=10, metadata_filter=mf)
    ```
  </Step>
</Steps>

<a id="backends"></a>
## 后端

<Tabs>
  <Tab title="In-memory / FAISS">

```python
# 内存：无持久化，用于开发和测试
store = VectorStore(backend="inmemory", dimension=384)

# FAISS：通过 save() / load() 实现本地磁盘持久化
store = VectorStore(backend="faiss", dimension=384)
store.save("./my_store")   # 保存到目录
store.load("./my_store")   # 从目录恢复
```

无需安装或 API 密钥。FAISS 需要 `pip install faiss-cpu`。

  </Tab>
  <Tab title="Pinecone">

```bash
pip install "semantica[pinecone]"
```

```python
import os
store = VectorStore(
    backend="pinecone",
    dimension=768,
    api_key=os.getenv("PINECONE_API_KEY"),
    index_name="semantica-index",
    environment="us-east-1-aws"
)
```

  </Tab>
  <Tab title="Weaviate">

```bash
pip install "semantica[weaviate]"
```

```python
store = VectorStore(
    backend="weaviate",
    dimension=768,
    url="http://localhost:8080",
    class_name="Document"
)
```

  </Tab>
  <Tab title="Qdrant">

```bash
pip install "semantica[qdrant]"
```

```python
store = VectorStore(
    backend="qdrant",
    dimension=768,
    url="http://localhost:6333",
    collection_name="semantica"
)
```

  </Tab>
  <Tab title="PgVector">

```bash
pip install "semantica[pgvector]"
```

```python
store = VectorStore(
    backend="pgvector",
    dimension=768,
    connection_string="postgresql://user:pass@localhost/db",
    table_name="embeddings"
)
```

`connection_string` 是必填项：如果缺失，该存储在构造时会抛出 `ValueError`。

  </Tab>
  <Tab title="Milvus">

```bash
pip install pymilvus
```

```python
store = VectorStore(
    backend="milvus",
    dimension=768,
    host="localhost",
    port=19530,
    collection_name="semantica"
)
```

  </Tab>
</Tabs>

<a id="backend-selection-guide"></a>
## 后端选择指南

| 后端 | 部署方式 | API 密钥 | 持久化 | 最适合 |
| :------- | :---------- | :------- | :----------- | :-------- |
| `inmemory` | 进程内 | 否 | 否 | 开发、单元测试 |
| `faiss` | 本地 | 否 | 通过 `save()`/`load()` | 本地部署、离线生产 |
| `pinecone` | 云端 | 是 | 托管 | 托管云、serverless |
| `weaviate` | 自托管 / 云端 | 可选 | 托管 | 丰富的元数据过滤 |
| `qdrant` | 自托管 / 云端 | 可选 | 托管 | 高性能过滤 |
| `milvus` | 自托管 | 否 | 托管 | 大规模生产 |
| `pgvector` | PostgreSQL | 否 | 托管 | Postgres 原生集成 |

<a id="hybridsearch"></a>
## HybridSearch

`HybridSearch` 将向量相似度与元数据过滤相结合。在构造时传入 `vector_store` 可避免每次调用都提供原始向量：

```python
from semantica.vector_store import HybridSearch, MetadataFilter

# 使用 vector_store 时：search() 会自动从存储中提取向量
search = HybridSearch(vector_store=store)
mf     = MetadataFilter().eq("category", "research").gt("year", 2022)

results = search.search(
    query=query_vector,   # np.ndarray 或查询字符串（自动嵌入）
    k=10,
    metadata_filter=mf
)

for r in results:
    print(f"{r['id']}: score: {r['score']:.3f}  metadata: {r['metadata']}")
```

如果没有 `vector_store`，请显式传入向量：

```python
search = HybridSearch()
results = search.search(
    query=query_vector,
    vectors=my_vectors,
    metadata=my_metadata,
    vector_ids=my_ids,
    k=10,
    metadata_filter=mf,
)
```

跨独立集合的多源融合：

```python
sources = [
    {"vectors": v1, "metadata": m1, "ids": ids1},
    {"vectors": v2, "metadata": m2, "ids": ids2},
]
fused = search.multi_source_search(query_vector, sources, k=10)
```

<Tip>
  **使用 `HybridSearch(vector_store=store)` 可避免每次调用都传递原始向量。** 当设置了 `vector_store` 时，`search()` 会自动从存储中提取向量和元数据：你只需传递查询和过滤器。
</Tip>

<a id="metadata-filtering"></a>
## 元数据过滤

`MetadataFilter` 支持链式条件：所有条件以 AND 组合：

```python
from semantica.vector_store import MetadataFilter

mf = MetadataFilter().eq("author", "John Smith")          # 相等
mf = MetadataFilter().ne("status", "archived")            # 不相等
mf = MetadataFilter().gt("year", 2022).lte("year", 2024)  # 范围
mf = MetadataFilter().in_list("tag", ["ai", "ml"])        # 集合成员
mf = MetadataFilter().contains("title", "neural")         # 子串 / 列表包含

# 多个条件：必须全部匹配（AND）
mf = (
    MetadataFilter()
    .eq("category", "research")
    .gt("year", 2022)
    .contains("title", "language model")
)
```

<a id="metadatafilter-methods"></a>
### MetadataFilter 方法

| 方法 | 运算符 | 描述 |
| :------ | :-------- | :----------- |
| `.eq(field, value)` | `==` | 精确相等 |
| `.ne(field, value)` | `!=` | 不相等 |
| `.gt(field, value)` | `>` | 大于 |
| `.gte(field, value)` | `>=` | 大于等于 |
| `.lt(field, value)` | `<` | 小于 |
| `.lte(field, value)` | `<=` | 小于等于 |
| `.in_list(field, values)` | `in` | 字段值在列表中 |
| `.contains(field, value)` | 子串 | 字符串包含或列表包含 |

<a id="searchranker"></a>
## SearchRanker

`SearchRanker` 融合来自多个排序列表的结果：

```python
from semantica.vector_store import SearchRanker

# 倒数排名融合（RRF）：对分数尺度差异鲁棒
ranker  = SearchRanker(strategy="reciprocal_rank_fusion")
fused   = ranker.rank([results_list_1, results_list_2])

# 加权平均：要求同一尺度下的归一化分数
ranker  = SearchRanker(strategy="weighted_average")
fused   = ranker.rank([results_list_1, results_list_2], weights=[0.7, 0.3])
```

| 融合策略 | 描述 |
| :--------------- | :----------- |
| `reciprocal_rank_fusion` | 通过 RRF 进行基于排名的组合：对分数尺度差异鲁棒（默认） |
| `weighted_average` | 分数加权和：向 `rank()` 传递 `weights=[...]` |

<a id="namespace-isolation"></a>
## 命名空间隔离

使用 `NamespaceManager` 将向量分配到具名命名空间，实现多租户隔离：

```python
from semantica.vector_store import NamespaceManager, VectorStore

store      = VectorStore(backend="inmemory", dimension=384)
ns_manager = NamespaceManager()

ns_manager.create_namespace("tenant_a", description="Customer A data")
ns_manager.create_namespace("tenant_b", description="Customer B data")

# 存储向量，然后将其分配到命名空间
ids_a = store.store_vectors(embeddings_a, metadata=metadata_a)
for vid in ids_a:
    ns_manager.add_vector_to_namespace(vid, "tenant_a")

# 列出所有命名空间名称
for name in ns_manager.list_namespaces():     # 返回 List[str]
    print(name)

# 获取某个命名空间中的所有向量
vectors_in_a = ns_manager.get_namespace_vectors("tenant_a")

# 查询某个向量所属的命名空间
ns = ns_manager.get_vector_namespace("vec_0")

ns_manager.delete_namespace("tenant_a")
```

<Tip>
  **在多租户应用中使用 `NamespaceManager`。** 将所有租户的向量存储在同一集合中并在查询时通过元数据过滤，速度慢且在意外遗漏过滤器时有数据泄露风险。命名空间隔离既更快（搜索空间更小）也更安全（结构化隔离）。
</Tip>

<a id="batch-operations"></a>
## 批量操作

```python
# 批量添加文本文档：可配置工作进程的并行嵌入
ids = store.add_documents(
    documents=large_doc_list,
    metadata=large_meta_list,
    batch_size=32,     # 每个嵌入批次的文本数
    parallel=True,     # 使用 ThreadPoolExecutor
)

# 批量添加预计算的向量
ids = store.store_vectors(vectors=embeddings_list, metadata=meta_list)

# 按向量 ID 列表删除
store.delete_vectors(vector_ids=["vec_0", "vec_1", "vec_2"])

# 原地更新向量（inmemory 后端会重建索引）
store.update_vectors(
    vector_ids=["vec_0"],
    new_vectors=[new_embedding]
)
```

<a id="persistence-faiss-and-in-memory"></a>
## 持久化（FAISS 与内存）

```python
store = VectorStore(backend="faiss", dimension=384)
store.add_documents(documents=docs, metadata=meta)

# 保存到目录：创建 index.bin 和 store_data.pkl
store.save("./vector_store_backup")

# 在新进程中恢复
store2 = VectorStore(backend="faiss", dimension=384)
store2.load("./vector_store_backup")
```

<Note>
  云端后端（Pinecone、Weaviate、Qdrant、Milvus、PgVector）自行管理持久化。`save()`/`load()` 仅适用于内存和 FAISS 后端。
</Note>

<Warning>
  **inmemory 和 faiss 后端在进程退出时若未调用 `save()` 会丢失数据。** 添加向量后请调用 `store.save(path)`。云端后端（Pinecone、Qdrant、Weaviate、Milvus、PgVector）会自动持久化。
</Warning>

<a id="metadatastore"></a>
## MetadataStore

`MetadataStore` 对结构化元数据进行索引，让你可以按字段值查询而无需向量：

```python
from semantica.vector_store import MetadataStore

meta_store = MetadataStore()

# 存储和检索元数据
meta_store.store_metadata("doc1", {"author": "Alice", "year": 2024, "category": "research"})
meta_store.store_metadata("doc2", {"author": "Bob",   "year": 2023, "category": "review"})

# 查询：返回匹配向量 ID 的 List[str]
ids = meta_store.query_metadata({"category": "research", "year": 2024})

# OR 查询
ids = meta_store.query_metadata({"category": "research"}, operator="OR")

# 获取和更新特定向量的元数据
meta = meta_store.get_metadata("doc1")
meta_store.update_metadata("doc1", {"score": 0.92})

# 获取某个字段的所有唯一值
years = meta_store.get_field_values("year")

# 统计信息
stats = meta_store.get_stats()
# {"total_vectors": 2, "indexed_fields": 3, "field_counts": {...}}
```

<Tip>
  **无需重新嵌入即可更新元数据。** `MetadataStore.update_metadata(id, {...})` 可更改附加字段（状态、标签、评审日期），而无需重新运行嵌入模型。将其用于不影响语义内容的状态变更。
</Tip>

<a id="faiss-index-type-reference"></a>
## FAISS 索引类型参考

FAISS 索引类型通过直接创建 `FAISSStore` 并调用 `create_index()` 来配置。使用小写类型名：

```python
from semantica.vector_store import FAISSStore

store = FAISSStore(dimension=384)

# flat：暴力精确搜索
store.create_index(index_type="flat", metric="L2")

# ivf：倒排文件索引
store.create_index(index_type="ivf", metric="L2", nlist=100)

# hnsw：分层可导航小世界图
store.create_index(index_type="hnsw", metric="L2", M=32)

# pq：用于内存效率的乘积量化
store.create_index(index_type="pq", metric="L2", m=8)
```

| 索引 | 内存 | 速度 | 精度 | 适用场景 |
| :----- | :------ | :----- | :-------- | :----------- |
| `flat` | 高 | 慢 | 精确（100%） | < 10 万向量，正确性至关重要 |
| `ivf` | 中 | 快 | ~95–98% | 10 万–1000 万向量，平衡良好 |
| `hnsw` | 中高 | 很快 | ~97–99% | 低延迟、生产检索 |
| `pq` | 低 | 快 | ~90–95% | 数百万向量、内存受限 |

<Warning>
  **FAISS 索引类型名称为小写。** `FAISSStore.create_index()` 方法接受 `"flat"`、`"ivf"`、`"hnsw"`、`"pq"`：而非 `"Flat"`、`"IVF"`、`"HNSW"`、`"PQ"`。大写值会抛出 `ValidationError`。
</Warning>

<Note>
  使用 `VectorStore(backend="faiss")` 时，底层的 `FAISSStore` 默认以 flat 索引初始化。要使用 ivf/hnsw/pq，请直接构造 `FAISSStore` 并用所需类型调用 `create_index()`。
</Note>

<a id="common-workflows"></a>
## 常见工作流

<Tabs>
  <Tab title="语义搜索流水线">
    ```python
    from semantica.vector_store import VectorStore

    store = VectorStore(backend="faiss", dimension=384)

    # 索引文档
    store.add_documents(
        documents=corpus_texts,
        metadata=[{"source": src} for src in sources],
        batch_size=64,
    )

    # 持久化
    store.save("./corpus_index")

    # 查询
    results = store.search("What is knowledge graph construction?", limit=5)
    for r in results:
        print(f"[{r['score']:.3f}] {r['metadata']['source']}")
    ```
  </Tab>
  <Tab title="过滤检索">
    ```python
    from semantica.vector_store import VectorStore, HybridSearch, MetadataFilter

    store  = VectorStore(backend="inmemory", dimension=384)
    search = HybridSearch(vector_store=store)

    # 仅返回 2023 年以后且 category 为 "research" 的结果
    mf = MetadataFilter().gte("year", 2023).eq("category", "research")
    results = search.search(query=query_vector, k=10, metadata_filter=mf)
    ```
  </Tab>
  <Tab title="多源融合">
    ```python
    from semantica.vector_store import HybridSearch, SearchRanker

    search = HybridSearch()
    sources = [
        {"vectors": v1, "metadata": m1, "ids": ids1},
        {"vectors": v2, "metadata": m2, "ids": ids2},
    ]
    fused = search.multi_source_search(query_vector, sources, k=10)

    # 自定义融合权重
    ranker = SearchRanker(strategy="weighted_average")
    fused  = ranker.rank([results_a, results_b], weights=[0.7, 0.3])
    ```
  </Tab>
  <Tab title="多租户命名空间">
    ```python
    from semantica.vector_store import VectorStore, NamespaceManager

    store = VectorStore(backend="inmemory", dimension=384)
    ns    = NamespaceManager()

    ns.create_namespace("project_a")
    ns.create_namespace("project_b")

    ids = store.add_documents(docs_a, metadata=meta_a)
    for vid in ids:
        ns.add_vector_to_namespace(vid, "project_a")

    # 检查归属
    print(ns.get_vector_namespace(ids[0]))  # "project_a"
    print(ns.get_namespace_stats("project_a"))
    ```
  </Tab>
</Tabs>

- [Embeddings](embeddings.zh-CN.md) — 生成存储于此的向量。
- [Context](context.zh-CN.md) — AgentContext 使用 VectorStore 进行记忆检索。
- [Split](split.zh-CN.md) — 在嵌入和存储之前对文档进行分块。
- [Ingest](ingest.zh-CN.md) — 在嵌入和存储之前摄取文档。
