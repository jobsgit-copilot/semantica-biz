---
title: "距离智能"
description: "语义邻域、N×N 距离矩阵、自我模式探索、邻近度混合检索，以及向量嵌入缓存优化。"
icon: "radar"
---

**[English](distance.md)** · **简体中文（当前）**

距离智能为知识图谱中的每个节点赋予一个**语义邻域**——使你不仅能回答"A 是否连接到 B？"，还能回答"A 在语义上距离 B 有多近，中间又存在什么？"

距离智能于 **v0.5.0** 引入，跨越三个层级运作：

<div style={{display:"flex",flexWrap:"wrap",gap:"1.5rem",margin:"1.5rem 0"}}>
  <div style={{flex:"1 1 200px",padding:"1.25rem 1.5rem",borderRadius:"10px",border:"1px solid rgba(16,185,129,0.25)",background:"rgba(16,185,129,0.04)"}}>
    <div style={{fontSize:"1.1rem",fontWeight:700,color:"#10B981",marginBottom:"6px"}}>距离矩阵</div>
    <div style={{fontSize:"0.82rem",color:"rgba(255,255,255,0.6)",lineHeight:1.5}}>任意节点集合之间的 N×N 上三角语义距离</div>
  </div>
  <div style={{flex:"1 1 200px",padding:"1.25rem 1.5rem",borderRadius:"10px",border:"1px solid rgba(16,185,129,0.25)",background:"rgba(16,185,129,0.04)"}}>
    <div style={{fontSize:"1.1rem",fontWeight:700,color:"#10B981",marginBottom:"6px"}}>语义邻域</div>
    <div style={{fontSize:"0.82rem",color:"rgba(255,255,255,0.6)",lineHeight:1.5}}>带置信度衰减和距离分级分类的 BFS 自我图</div>
  </div>
  <div style={{flex:"1 1 200px",padding:"1.25rem 1.5rem",borderRadius:"10px",border:"1px solid rgba(16,185,129,0.25)",background:"rgba(16,185,129,0.04)"}}>
    <div style={{fontSize:"1.1rem",fontWeight:700,color:"#10B981",marginBottom:"6px"}}>邻近度混合</div>
    <div style={{fontSize:"0.82rem",color:"rgba(255,255,255,0.6)",lineHeight:1.5}}>在检索中将语义相似度与图邻近度相结合</div>
  </div>
  <div style={{flex:"1 1 200px",padding:"1.25rem 1.5rem",borderRadius:"10px",border:"1px solid rgba(16,185,129,0.25)",background:"rgba(16,185,129,0.04)"}}>
    <div style={{fontSize:"1.1rem",fontWeight:700,color:"#10B981",marginBottom:"6px"}}>10× 缓存</div>
    <div style={{fontSize:"0.82rem",color:"rgba(255,255,255,0.6)",lineHeight:1.5}}>基于图谱修订号的向量嵌入缓存避免冗余重复计算</div>
  </div>
</div>


<a id="distance-bands"></a>
## 距离分级

每个邻居结果都会根据跳数和语义相似度被归入四个距离分级之一：

| 分级 | 跳数 | 含义 | Explorer 颜色 |
| :---- | :-------- | :------- | :------------- |
| `direct` | 1 | 直接邻居——语义重叠度高 | 绿色 |
| `near` | 2 | 相隔一跳——紧密相关概念 | 青色 |
| `mid-range` | 3–4 | 概念相关但有一定分离 | 黄色 |
| `distant` | 5+ | 结构连接弱 | 红色 |

距离分级贯穿整个系统：检索结果、路径响应、API 端点，以及 Explorer 自我模式可视化，全部使用同一套四档分类。


<a id="quick-start"></a>
## 快速上手

<Steps>
  <Step title="获取带距离元数据的邻居">
    最简单的入口：调用 `get_neighbors()` 并传入 `include_distance_metadata=True`：

    ```python
    from semantica.context import ContextGraph

    graph = ContextGraph(advanced_analytics=True)

    graph.add_node("python",   "language",   properties={"paradigm": "multi"})
    graph.add_node("fastapi",  "framework",  properties={"language": "Python"})
    graph.add_node("django",   "framework",  properties={"language": "Python"})
    graph.add_node("sqlmodel", "library",    properties={"orm": True})

    graph.add_edge("python",   "fastapi",  "enables")
    graph.add_edge("python",   "django",   "enables")
    graph.add_edge("fastapi",  "sqlmodel", "uses")

    neighbors = graph.get_neighbors(
        "python",
        hops=3,
        include_distance_metadata=True,
    )

    for n in neighbors:
        print(f"{n['node_id']:12s}  band={n['distance_band']:10s}  "
              f"decay={n['confidence_decay']:.3f}  "
              f"path={n['path_to_anchor']}")
    ```
    ```
    fastapi       band=direct     decay=1.000  path=['python', 'fastapi']
    django        band=direct     decay=1.000  path=['python', 'django']
    sqlmodel      band=near       decay=0.750  path=['python', 'fastapi', 'sqlmodel']
    ```
  </Step>
  <Step title="计算语义距离矩阵">
    ```python
    from semantica.kg import SimilarityCalculator, NodeEmbedder

    # 先生成结构向量嵌入
    embedder   = NodeEmbedder(method="node2vec", embedding_dimension=128)
    embeddings = embedder.compute_embeddings(kg, ["language", "framework", "library"], ["enables", "uses"])

    # N×N 上三角距离矩阵
    calc   = SimilarityCalculator()
    matrix = calc.compute_distance_matrix(embeddings)

    # matrix["distances"] 是一个上三角字典：{(node_a, node_b): distance}
    for (a, b), dist in sorted(matrix["distances"].items(), key=lambda x: x[1]):
        print(f"{a:15s} ↔ {b:15s}  distance={dist:.4f}")
    ```
  </Step>
  <Step title="将邻近度混合进检索">
    在 `AgentContext` 上设置 `proximity_weight`，即可将图邻近度混合进每一次语义检索调用：

    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(advanced_analytics=True),
        decision_tracking=True,
        proximity_weight=0.3,   # combined = 0.7×语义 + 0.3×邻近度
    )

    # retrieve() 和 find_precedents() 都使用混合得分
    results = context.retrieve("web API frameworks", max_results=10)
    for r in results:
        print(f"[{r['combined_score']:.3f}]  semantic={r['semantic_score']:.3f}  "
              f"proximity={r['proximity_score']:.3f}  {r['content'][:60]}")
    ```
  </Step>
</Steps>


<a id="contextgraph-distance-api"></a>
## ContextGraph 距离 API

<a id="get-neighbors"></a>
### `get_neighbors()`

当传入 `include_distance_metadata=True` 时，返回带有距离元数据增强的 BFS 邻居：

```python
neighbors = graph.get_neighbors(
    node_id="python",
    hops=4,
    include_distance_metadata=True,
    min_weight=0.3,               # 排除低置信度边
)
```

| 字段 | 类型 | 描述 |
| :---- | :---- | :----------- |
| `node_id` | `str` | 节点标识符 |
| `node_type` | `str` | 节点类型标签 |
| `properties` | `Dict` | 节点属性字典 |
| `hop_count` | `int` | 距锚点的 BFS 跳数 |
| `distance_band` | `str` | `"direct"` / `"near"` / `"mid-range"` / `"distant"` |
| `confidence_decay` | `float` | 经过跳数衰减后的置信度得分：`weight^hop_count` |
| `path_to_anchor` | `List[str]` | 从锚点到该节点的最短路径 |
| `edge_weight` | `float` | 直接边的权重（当 hop=1 时） |

<a id="get-neighbor-distances"></a>
### `get_neighbor_distances()`

返回按组合置信度衰减距离得分排序的邻居列表：

```python
distances = graph.get_neighbor_distances("fastapi", hops=3)

for d in distances:
    print(f"{d['node_id']:15s}  score={d['combined_distance_score']:.4f}  "
          f"band={d['distance_band']}")
```


<a id="similaritycalculator-pairwise-similarity"></a>
## SimilarityCalculator —— 成对相似度

`SimilarityCalculator` 使用四种指标计算节点向量嵌入之间的相似度。

```python
from semantica.kg import SimilarityCalculator

calc = SimilarityCalculator(method="cosine", normalize=True)
# method: "cosine" | "euclidean" | "manhattan" | "correlation"
```

<a id="constructor"></a>
### 构造函数

| 参数 | 类型 | 默认值 | 描述 |
| :--------- | :---- | :------- | :----------- |
| `method` | `str` | `"cosine"` | 默认指标：`"cosine"`、`"euclidean"`、`"manhattan"`、`"correlation"` |
| `normalize` | `bool` | `True` | 在计算前对向量进行归一化 |

<a id="methods"></a>
### 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `cosine_similarity(vector1, vector2)` | `float` | 两个向量之间的余弦相似度 `[-1, 1]` |
| `euclidean_distance(embedding1, embedding2)` | `float` | 两个向量之间的 L2 距离（非负） |
| `manhattan_distance(embedding1, embedding2)` | `float` | 两个向量之间的 L1 距离（非负） |
| `correlation_similarity(embedding1, embedding2)` | `float` | 两个向量之间的皮尔逊相关系数 `[-1, 1]` |
| `batch_similarity(embeddings, query_embedding, method=None, top_k=None, chunk_size=1000)` | `Dict[str, float]` | 所有节点相对于一个查询向量的相似度。返回 `{node_id: score}` |
| `pairwise_similarity(embeddings, method=None)` | `Dict[Tuple[str,str], float]` | 所有节点对的上三角 N×N 成对相似度矩阵 |
| `find_most_similar(embeddings, query_embedding, top_k=10, method=None)` | `List[Tuple[str, float]]` | 按相似度排序的 top-k `(node_id, score)` 对 |

<a id="pairwise-similarity-matrix"></a>
### 成对相似度矩阵

`pairwise_similarity()` 返回 N×N 矩阵的上三角——每个键都是一个 `(node_id_a, node_id_b)` 元组：

```python
from semantica.kg import NodeEmbedder, SimilarityCalculator

embedder   = NodeEmbedder(method="node2vec", embedding_dimension=128)
embeddings = embedder.compute_embeddings(kg, ["language", "framework"], ["enables", "uses"])

calc = SimilarityCalculator(method="cosine")

# N×N 上三角：Dict[(node_a, node_b), similarity_score]
matrix = calc.pairwise_similarity(embeddings)

# 按相似度排序（最相似的在前）
for (a, b), score in sorted(matrix.items(), key=lambda x: x[1], reverse=True)[:5]:
    print(f"{a:15s} ↔ {b:15s}  similarity={score:.4f}")

# 查找最相似的一对
best_pair = max(matrix.items(), key=lambda x: x[1])
print(f"Most similar: {best_pair[0]}  score={best_pair[1]:.4f}")

# 查找最不相似的一对
worst_pair = min(matrix.items(), key=lambda x: x[1])
print(f"Most distant:  {worst_pair[0]}  score={worst_pair[1]:.4f}")
```

<Note>
  该矩阵仅包含上三角——会存储 `(a, b)` 但不会存储 `(b, a)`。要双向查找：`matrix.get((a, b)) or matrix.get((b, a))`。
</Note>

<a id="batch-similarity"></a>
### 批量相似度

使用分块的向量化运算高效地将一个查询向量与所有节点进行比较：

```python
# 查询向量与所有节点比较
scores = calc.batch_similarity(
    embeddings,
    query_embedding=my_query_vec,
    method="cosine",    # 覆盖默认值
    top_k=10,           # 只返回前 10 个（None = 全部）
    chunk_size=1000,    # 为提升内存效率的分块大小
)

for node_id, score in sorted(scores.items(), key=lambda x: x[1], reverse=True):
    print(f"{node_id:15s}  {score:.4f}")
```

<a id="find-most-similar"></a>
### 查找最相似节点

```python
# 按降序排序的 top-k (node_id, score) 元组
similar = calc.find_most_similar(
    embeddings,
    query_embedding=embeddings["python"],
    top_k=5,
    method="cosine",
)

for node_id, score in similar:
    print(f"{node_id:15s}  similarity={score:.4f}")
```

<a id="individual-metrics"></a>
### 单项指标

```python
vec_a = embeddings["fastapi"]
vec_b = embeddings["django"]

cosine  = calc.cosine_similarity(vec_a, vec_b)
l2      = calc.euclidean_distance(vec_a, vec_b)
l1      = calc.manhattan_distance(vec_a, vec_b)
pearson = calc.correlation_similarity(vec_a, vec_b)

print(f"Cosine:      {cosine:.4f}")
print(f"Euclidean:   {l2:.4f}")
print(f"Manhattan:   {l1:.4f}")
print(f"Correlation: {pearson:.4f}")
```


<a id="proximity-blended-retrieval"></a>
## 邻近度混合检索

`AgentContext.retrieve()` 和 `find_precedents()` 都支持一个 `proximity_weight` 参数，将图邻近度混合进语义相似度得分：

```
combined_score = (1 − proximity_weight) × semantic_score
              + proximity_weight × proximity_score
```

其中 `proximity_score` 由查询锚点节点的跳数和边权重推导而来。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=ContextGraph(advanced_analytics=True),
    proximity_weight=0.3,
)

# 标准检索——自动混合邻近度
results = context.retrieve("model deployment strategies", max_results=10)

# 按调用覆盖权重
results = context.retrieve(
    "model deployment strategies",
    max_results=10,
    proximity_weight=0.5,   # 该查询使用更强的邻近度权重
)

# find_precedents 同样混合邻近度
precedents = context.find_precedents(
    "infrastructure scaling decisions",
    proximity_weight=0.4,
    limit=5,
)

for p in precedents:
    print(f"[{p.combined_score:.3f}]  {p.outcome}  (confidence: {p.confidence:.2f})")
```


<a id="embedding-cache"></a>
## 向量嵌入缓存

向量嵌入缓存避免了为自上次调用以来未发生变化的节点重新计算向量嵌入——在大型图上可带来高达 **10× 的吞吐量提升**。

<a id="how-it-works"></a>
### 工作原理

每个 `GraphSession` 都会跟踪一个从当前节点和边状态派生出的**图谱修订号哈希**。当一个距离矩阵或邻域请求到达时：

1. 将修订号哈希与缓存的哈希进行比较
2. 如果未变化：直接返回缓存的向量嵌入
3. 如果已变化（添加或修改了节点/边）：缓存失效并重新计算向量嵌入

```python
from semantica.explorer import GraphSession

session = GraphSession(graph=kg)

# 第一次调用：计算向量嵌入并存入缓存
embeddings = session.get_cached_embeddings()

# 第二次调用（图未变化）：立即返回缓存
embeddings = session.get_cached_embeddings()

# 图被修改后：缓存自动失效
session.graph.add_node("new_node", "concept", properties={})
embeddings = session.get_cached_embeddings()  # 重新计算
```

| 参数 | 类型 | 默认值 | 描述 |
| :--------- | :---- | :------- | :----------- |
| `force_refresh` | `bool` | `False` | 即使图未变化也强制使缓存失效 |
| 缓存失效 | 自动 | — | 由 `add_nodes()`、`add_edges()` 或任何变更操作触发 |
| 缓存作用域 | 每会话 | — | 每个 `GraphSession` 各自维护独立的缓存 |

<Tip>
  该缓存在同一张图被反复查询距离矩阵和自我模式邻域的 Explorer 部署中最有效。在批处理流水线场景中，请设置 `force_refresh=True`，以确保始终使用最新的图状态。
</Tip>


<a id="rest-api-endpoints"></a>
## REST API 端点

v0.5.0 新增了五个用于以编程方式访问距离智能的端点：

<a id="post-apigraphdistance-matrix"></a>
### `POST /api/graph/distance-matrix`

为一组节点 ID 计算 N×N 语义距离矩阵：

```bash
curl -X POST http://localhost:8000/api/graph/distance-matrix \
  -H "Content-Type: application/json" \
  -d '{
    "node_ids": ["alice", "bob", "acme_corp", "beta_ltd"],
    "embedding_model": "all-MiniLM-L6-v2",
    "include_band_classification": true
  }'
```

```json
{
  "matrix": {
    "alice,bob": 0.312,
    "alice,acme_corp": 0.087,
    "alice,beta_ltd": 0.154,
    "bob,acme_corp": 0.401,
    "bob,beta_ltd": 0.233,
    "acme_corp,beta_ltd": 0.198
  },
  "most_similar": ["alice", "acme_corp"],
  "most_distant": ["bob", "acme_corp"],
  "mean_distance": 0.231
}
```

<a id="get-apigraphnodeidsemantic-neighborhood"></a>
### `GET /api/graph/node/{id}/semantic-neighborhood`

获取一个节点的自我图（BFS 邻域）及其距离元数据：

```bash
curl "http://localhost:8000/api/graph/node/alice/semantic-neighborhood?depth=3&include_distance_metadata=true"
```

```json
{
  "anchor_node": "alice",
  "neighbors": [
    {"node_id": "acme_corp", "distance_band": "direct",   "confidence_decay": 1.0,  "hop_count": 1},
    {"node_id": "ceo_role",  "distance_band": "direct",   "confidence_decay": 1.0,  "hop_count": 1},
    {"node_id": "beta_ltd",  "distance_band": "near",     "confidence_decay": 0.75, "hop_count": 2},
    {"node_id": "london_hq", "distance_band": "mid-range","confidence_decay": 0.56, "hop_count": 3}
  ],
  "total_neighbors": 4,
  "depth": 3
}
```

<a id="get-apidecisionscausal-distance"></a>
### `GET /api/decisions/causal-distance`

返回两个决策节点之间的因果距离（穿过因果边的跳数）：

```bash
curl "http://localhost:8000/api/decisions/causal-distance?source=dec_001&target=dec_005"
```

```json
{
  "source": "dec_001",
  "target": "dec_005",
  "causal_hops": 3,
  "causal_path": ["dec_001", "dec_002", "dec_004", "dec_005"],
  "distance_band": "near"
}
```

<a id="get-apitemporaldistance-history"></a>
### `GET /api/temporal/distance-history`

追踪两个节点之间的语义距离如何随时间演变：

```bash
curl "http://localhost:8000/api/temporal/distance-history?node_a=alice&node_b=acme_corp&snapshots=2021-01-01,2022-01-01,2023-01-01"
```

```json
{
  "node_a": "alice",
  "node_b": "acme_corp",
  "history": [
    {"timestamp": "2021-01-01", "distance": 0.08, "band": "direct"},
    {"timestamp": "2022-01-01", "distance": 0.09, "band": "direct"},
    {"timestamp": "2023-01-01", "distance": 0.54, "band": "mid-range"}
  ]
}
```

<a id="post-apiexportdistance-enriched"></a>
### `POST /api/export/distance-enriched`

导出经距离元数据增强的图数据（CSV 或 JSONL，上限 200 个节点）：

```bash
curl -X POST http://localhost:8000/api/export/distance-enriched \
  -H "Content-Type: application/json" \
  -d '{"anchor_node": "alice", "depth": 4, "format": "csv"}'
```


<a id="explorer-distance-intelligence-ui"></a>
## Explorer 距离智能 UI

Knowledge Explorer 将距离智能直接内嵌于浏览器仪表板中：

<AccordionGroup>

<Accordion title="自我模式（Ego Mode）" icon="circle-nodes">
  自我模式以一个选定节点为中心进行可视化，并用 **BFS 景深衰减**渲染其语义邻域——离锚点越远的节点越暗，从而揭示概念邻近度的"形状"。

  - **深度滑块（1–8）**：控制邻域的 BFS 半径
  - **置信度衰减可视化**：边的不透明度映射到 `confidence_decay` 得分
  - **距离分级颜色编码**：绿色（direct）→ 青色（near）→ 黄色（mid-range）→ 红色（distant）
  - **瓶颈节点高亮**：连接了原本分离簇群的桥接节点会在路径检查器中高亮显示

  通过 Explorer 工具栏激活：**View → Ego Mode**，然后点击任意节点将其设为锚点。
</Accordion>

<Accordion title="距离热力图" icon="table-cells">
  热力图将一个 N×N 距离矩阵渲染为颜色编码的网格——可即时揭示哪些节点簇在语义上具有内聚性，哪些是孤立的。

  - **颜色刻度**：绿色（近，距离 → 0）经黄色到红色（远，距离 → 1）
  - **悬停**：显示每个单元格的精确距离值和距离分级
  - **排序选项**：按节点类型、社区归属或字母顺序对行/列排序

  通过 Explorer 侧边栏的 **View → Distance Heatmap** 访问。
</Accordion>

<Accordion title="语义叠加" icon="layer-group">
  在标准的力导向图布局上叠加语义相似度，而无需切换模式：

  - **语义叠加**：边的粗细按语义相似度得分缩放
  - **结构叠加**：边的粗细按图中心性缩放
  - 两种叠加可独立切换

  通过 Explorer 工具栏中的 **Overlay** 开关访问。
</Accordion>

<Accordion title="路径检查器" icon="route">
  点击任意两个节点以检查它们之间的最短路径。路径检查器会显示：

  - **距离分级标签**：将整条路径归类为 direct / near / mid-range / distant
  - **指标卡片**：跳数、平均边权重、路径置信度衰减
  - **瓶颈节点高亮**：移除后会使路径断开的那个单一节点
  - **距离历史**：两个节点之间的距离在各图快照间如何变化的时间线

  在任意两个选中的节点上通过 **右键 → Inspect Path** 访问。
</Accordion>

</AccordionGroup>


<a id="real-world-patterns"></a>
## 实战模式

<Tabs>
  <Tab title="知识簇发现">
    无需运行社区发现，即可在大型知识图谱中找出语义内聚的主题簇：

    ```python
    from semantica.kg import NodeEmbedder, SimilarityCalculator

    embedder = NodeEmbedder(method="node2vec", embedding_dimension=128)
    embeddings = embedder.compute_embeddings(kg, node_types=["Concept", "Topic"])

    calc = SimilarityCalculator()

    # 将成对距离 < 0.2 的节点聚为一簇
    clusters = calc.cluster_by_distance(embeddings, threshold=0.2)

    for i, cluster in enumerate(clusters):
        print(f"Cluster {i+1} ({len(cluster)} nodes): {cluster[:5]}")
    ```
  </Tab>
  <Tab title="异常检测">
    标记在其结构邻居中意外地遥远的节点——可能是数据质量问题，也可能是真正的异常：

    ```python
    from semantica.context import ContextGraph
    from semantica.kg import NodeEmbedder, SimilarityCalculator

    graph   = ContextGraph(advanced_analytics=True)
    # ... 构建图 ...

    embedder = NodeEmbedder(method="node2vec", embedding_dimension=128)
    embeddings = embedder.compute_embeddings(graph._graph, ["entity"], ["RELATED_TO"])

    calc = SimilarityCalculator()

    for node_id in graph._graph.nodes():
        neighbors = graph.get_neighbors(node_id, hops=1, include_distance_metadata=True)
        for n in neighbors:
            # 由边连接但在语义上非常遥远 → 异常候选
            structural_dist = 1.0 - n["edge_weight"]
            semantic_dist   = calc.euclidean_distance(
                embeddings[node_id], embeddings[n["node_id"]]
            )
            if semantic_dist > 0.7 and structural_dist < 0.3:
                print(f"Anomaly: {node_id} → {n['node_id']}  "
                      f"(structural={structural_dist:.2f}, semantic={semantic_dist:.2f})")
    ```
  </Tab>
  <Tab title="决策一致性审计">
    验证相似的决策（语义距离低）是否达成了相似的结果——将不一致之处标记以供复核：

    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(advanced_analytics=True),
        decision_tracking=True,
        proximity_weight=0.4,
    )

    # ... 填入历史决策 ...

    # 查找语义相近但结果不同的决策对
    all_decisions = context.query_decisions("", max_hops=0)
    for i, d1 in enumerate(all_decisions):
        for d2 in all_decisions[i+1:]:
            precedents = context.find_precedents(
                d1.scenario, limit=5, proximity_weight=0.4
            )
            for p in precedents:
                if p.source_decision_id == d2.decision_id:
                    if p.similarity_score > 0.85 and d1.outcome != d2.outcome:
                        print(f"INCONSISTENCY: {d1.scenario}")
                        print(f"  Decision A: {d1.outcome}  (confidence {d1.confidence:.2f})")
                        print(f"  Decision B: {d2.outcome}  (confidence {d2.confidence:.2f})")
                        print(f"  Similarity: {p.similarity_score:.3f}")
    ```
  </Tab>
</Tabs>


<a id="performance"></a>
## 性能

| 操作 | 无缓存 | 有缓存 | 提升 |
| :--------- | :------------ | :--------- | :---------- |
| 距离矩阵（118k 节点） | ~48s | ~4.8s | **10×** |
| 语义邻域（深度 4） | ~2.1s | ~0.21s | **10×** |
| 节点搜索（已索引） | 24 ms | 0.004 ms | **6,000×** |
| 语义去重 | 基线 | — | **6.98×**（v2 算法） |

<Note>
  10× 的缓存提升适用于请求之间图未发生变化的场景。在持续添加节点的写密集型流水线中，缓存命中率会较低。读密集的 Explorer 使用场景请使用 `force_refresh=False`（默认），批处理流水线场景请使用 `force_refresh=True`。
</Note>

- [Context 模块](context.zh-CN.md) —— `ContextGraph.get_neighbors()` 与邻近度混合检索。
- [Knowledge Graph 模块](kg.zh-CN.md) —— `NodeEmbedder`、`SimilarityCalculator` 以及图分析。
- [可视化](visualization.zh-CN.md) —— 以编程方式生成距离热力图和自我模式图渲染。
- [Explorer](explorer.zh-CN.md) —— 内置距离智能仪表板的 Knowledge Explorer。

- [距离智能](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/12_Distance_Intelligence.ipynb) —— 语义邻域与距离矩阵 · 进阶
