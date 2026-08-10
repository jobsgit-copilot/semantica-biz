---
title: "知识图谱模块"
description: "图构建、时序模型、图分析、相似度评分与结构化嵌入。"
icon: "diagram-project"
---

**[English](kg.md)** · **简体中文（当前）**

`semantica.kg` 将抽取出的实体与关系转化为结构化、可查询的知识图谱：

- 时序节点与边，带有 `valid_from` / `valid_until` 时间窗以及全部 13 种 Allen 区间关系
- 完整的图分析套件：中心性、社区发现、路径查找、链路预测
- 用于下游 ML 与相似度评分的 Node2Vec 结构化向量嵌入
- 通过 `TemporalVersionManager` 实现 OWL-Time 导出与版本化快照
- 在持久化之前进行模式与约束校验


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `KnowledgeGraph` | 核心图数据结构：节点、边、属性、时序有效性 |
| `GraphBuilder` | 从实体 + 关系构建；传入 `merge_entities=True` 启用去重 |
| `GraphBuilderWithProvenance` | 封装 `GraphBuilder` 并附带可选的溯源跟踪；传入 `provenance=True` 启用 |
| `EntityResolver` | 图构建期间的实体去重与合并 |
| `GraphAnalyzer` | 统一的分析封装：一次调用即可运行中心性、社区发现与连通性分析 |
| `ConnectivityAnalyzer` | 连通分量检测、桥识别、密度与度数统计 |
| `TemporalGraphQuery` | 时间点快照、范围查询、演化分析、时序路径查找 |
| `TemporalPatternDetector` | 在时序边上进行序列与环模式检测 |
| `TemporalReasoningEngine` | 基于 `TemporalInterval` 对象的全部 13 种 Allen 区间代数关系 |
| `TemporalInterval` | 不可变 dataclass `(start: datetime, end: datetime \| TemporalBound, label?)` |
| `IntervalRelation` | 全部 13 种 Allen 关系标签的枚举（`BEFORE`、`AFTER`、`MEETS`、…） |
| `BiTemporalFact` | 包装 `valid_from`、`valid_until`、`recorded_at`、`superseded_at` 的 dataclass。工厂方法：`BiTemporalFact.from_relationship(rel_dict)` |
| `TemporalBound` | 用于开放式区间的哨兵枚举 —— 单一取值：`TemporalBound.OPEN` |
| `TemporalNormalizer` | 将自然语言时序表达式解析为 `(datetime, datetime)` 元组 —— 零 LLM 调用 |
| `TemporalQueryRewriter` | 从自由文本查询中抽取时序意图；返回 `TemporalQueryResult` |
| `TemporalQueryResult` | `TemporalQueryRewriter.rewrite()` 的 dataclass 输出 |
| `TemporalVersionManager` | 带 SHA-256 完整性的版本化快照，基于 SQLite 的持久化存储 |
| `CentralityCalculator` | PageRank、度数、中介、接近、特征向量中心性 |
| `CommunityDetector` | Louvain、Leiden、Label Propagation 与 K-Clique 社区发现 |
| `PathFinder` | Dijkstra、A*、BFS 与 K-最短路径算法 |
| `LinkPredictor` | 优先连接、Jaccard、Adamic-Adar 链路预测 |
| `NodeEmbedder` | 用于下游 ML 的 Node2Vec 结构化嵌入 |
| `SimilarityCalculator` | 余弦、欧几里得、曼哈顿与相关性相似度评分 |
| `GraphValidator` | 在持久化之前进行模式与约束校验 |
| `AlgorithmTrackerWithProvenance` | 带溯源元数据的算法执行跟踪 |
| `AlgorithmRegistry` / `algorithm_registry` | 已注册算法的注册表；`algorithm_registry` 是共享单例 |
| `ProvenanceTracker` | 针对图操作的 W3C PROV-O 溯源跟踪 |
| `SeedManager` | 跨算法的可复现随机种子管理 |
| `KGConfig` / `kg_config` | 模块级配置；`kg_config` 是共享单例 |


<Tip>
  若需要冲突检测与高级实体解析，请将 `semantica.conflicts` 和 `semantica.deduplication` 与本模块结合使用。
</Tip>

<img src="/assets/img/diagrams/kg-structure.svg" alt="知识图谱实体与关系结构：Person、Organization、Location、Date 节点与带类型的标签边" style={{ width: '100%', borderRadius: '12px', margin: '0 0 24px' }} />

<a id="graphbuilder"></a>
## GraphBuilder

**`GraphBuilder`** 从抽取出的实体与关系构建知识图谱。`merge_entities` 默认为 `False`：传入 **`True`** 可在构建期间启用实体去重：

```python
from semantica.kg import GraphBuilder

# 传入一个包含 "entities" 与 "relationships" 键的字典
builder = GraphBuilder(merge_entities=True)
kg = builder.build({"entities": entities, "relationships": relationships})
```

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `build(sources)` | `dict` | 从字典、字典列表或实体/关系对象列表构建图 |
| `build_single_source(data)` | `dict` | 从单个数据源字典构建图 |


<a id="temporal-knowledge-graphs-v040"></a>
## 时序知识图谱（v0.4.0+）

<Info>
  完整的时序参考（包括 `BiTemporalFact`、`TemporalReasoningEngine`、Allen 区间代数与 `TemporalNormalizer`）在专门的[时序智能](temporal.zh-CN.md)页面中介绍。本节记录 KG 层的时序 API。
</Info>

时序栈 —— 完整参考请见[时序智能](temporal.zh-CN.md)页面。

<a id="building-a-temporal-graph"></a>
### 构建时序图

```python
from semantica.kg import GraphBuilder, TemporalGraphQuery, TemporalVersionManager

builder = GraphBuilder()
kg = builder.build(sources=[
    {
        "entities": [
            {"id": "alice",     "type": "Person"},
            {"id": "acme_corp", "type": "Organization"},
            {"id": "beta_ltd",  "type": "Organization"},
        ],
        "relationships": [
            {
                "source": "alice", "target": "acme_corp", "type": "ceo_of",
                "valid_from":  "2018-01-01",
                "valid_until": "2022-06-01",
            },
            {
                "source": "alice", "target": "beta_ltd", "type": "ceo_of",
                "valid_from":  "2022-06-01",
                # 没有 valid_until -> 开放式（TemporalBound.OPEN）
            },
        ],
    }
])
```

<a id="point-in-time-queries"></a>
### 时间点查询

`TemporalGraphQuery` 接受可选的构造参数；将图传入每次查询调用：

```python
from semantica.kg import TemporalGraphQuery

query = TemporalGraphQuery(
    temporal_granularity="day",        # second|minute|hour|day|week|month|year
    enable_temporal_reasoning=True,
)

# 主 API：query_at_time 返回计数 + 过滤后的数据
result_2020 = query.query_at_time(kg, "", at_time="2020-06-15")
result_2023 = query.query_at_time(kg, "", at_time="2023-01-01")
print(f"Rels in 2020: {result_2020['num_relationships']}")

# 底层：reconstruct_at_time 返回深拷贝的子图字典
snapshot = query.reconstruct_at_time(kg, "2020-06-15")

# 范围查询：在 2021 年任意时段活跃的所有关系
range_result = query.query_time_range(kg, "", "2021-01-01", "2021-12-31")

# 比较两个快照：使用 TemporalVersionManager.compare_versions()
# （temporal_diff() 不存在 —— 见下文的 TemporalVersionManager）
```

<a id="bi-temporal-facts"></a>
### 双时态事实

`BiTemporalFact` 是一个 **dataclass** —— 使用 `from_relationship()` 工厂方法，而非位置构造函数：

```python
from semantica.kg import BiTemporalFact, TemporalBound

rel = {
    "source": "alice", "target": "acme_corp", "type": "ceo_of",
    "valid_from":    "2018-01-01",
    "valid_until":   "2022-06-01",
    "recorded_at":   "2018-01-05T09:32:00Z",
    "superseded_at": None,   # None -> TemporalBound.OPEN（仍然有效）
}
fact = BiTemporalFact.from_relationship(rel)

print(fact.valid_from)      # datetime(2018, 1, 1, tzinfo=utc)
print(fact.valid_until)     # datetime(2022, 6, 1, tzinfo=utc)
print(fact.superseded_at)   # TemporalBound.OPEN

# 开放式事实（无 valid_until -> TemporalBound.OPEN）
open_rel = {"source": "alice", "target": "beta_ltd", "type": "ceo_of",
            "valid_from": "2022-06-01"}
open_fact = BiTemporalFact.from_relationship(open_rel)
print(open_fact.valid_until)   # TemporalBound.OPEN

# 序列化回字典字段以便存储
fields = fact.to_relationship_fields()
```

<a id="allen-interval-algebra"></a>
### Allen 区间代数

`TemporalReasoningEngine` 确定性地实现了**全部 13 种 Allen 关系** —— 无 LLM、无概率。它操作 `TemporalInterval` 对象（而非普通字典）：

```python
from semantica.kg import (
    TemporalReasoningEngine, TemporalInterval, IntervalRelation
)
from datetime import datetime, timezone

def dt(y, m, d): return datetime(y, m, d, tzinfo=timezone.utc)

engine = TemporalReasoningEngine()

h1_2020 = TemporalInterval(start=dt(2020, 1, 1), end=dt(2020, 6, 30))
q2_q4   = TemporalInterval(start=dt(2020, 4, 1), end=dt(2020, 12, 31))

relation = engine.relation(h1_2020, q2_q4)   # 主方法
print(relation)          # IntervalRelation.OVERLAPS
print(relation.value)    # "overlaps"

print(engine.overlaps(h1_2020, q2_q4))  # True
print(engine.contains(q2_q4, h1_2020))  # False
print(engine.active_at(h1_2020, dt(2020, 3, 15)))  # True
```

| `IntervalRelation` | `.value` | 描述 |
| :--- | :--- | :--- |
| `BEFORE` | `"before"` | A 在 B 开始之前严格结束 |
| `MEETS` | `"meets"` | A 在 B 开始时恰好结束 |
| `OVERLAPS` | `"overlaps"` | A 与 B 共享一段时间段；A 先开始且先结束 |
| `STARTS` | `"starts"` | 起点相同；A 比 B 先结束 |
| `DURING` | `"during"` | A 完全包含在 B 中 |
| `FINISHES` | `"finishes"` | 终点相同；B 更早开始 |
| `EQUALS` | `"equals"` | 完全相同的区间 |
| `AFTER`、`MET_BY`、`OVERLAPPED_BY`、`STARTED_BY`、`CONTAINS`、`FINISHED_BY` | *（反向关系）* | 镜像关系 |

<a id="natural-language-temporal-parsing"></a>
### 自然语言时序解析

```python
from semantica.kg import TemporalNormalizer, TemporalQueryRewriter
from datetime import datetime, timezone

# reference_date 在构造时设置（相对短语所必需）
norm = TemporalNormalizer(reference_date=datetime(2024, 6, 15, tzinfo=timezone.utc))

# 返回 Optional[Tuple[datetime, datetime]] —— 不是字典
result = norm.normalize("last quarter")
start, end = result
print(start)   # datetime(2024, 1, 1, tzinfo=utc)
print(end)     # datetime(2024, 3, 31, tzinfo=utc)

result = norm.normalize("2022")
# (datetime(2022, 1, 1, tzinfo=utc), datetime(2022, 12, 31, tzinfo=utc))

result = norm.normalize("unparseable phrase")
print(result)  # None

# TemporalQueryRewriter：主方法是 rewrite()，返回 TemporalQueryResult
rewriter = TemporalQueryRewriter()
result = rewriter.rewrite("Who was CEO before the 2022 restructuring?")
print(result.temporal_intent)    # "before"
print(result.at_time.year)       # 2022
print(result.rewritten_query)    # "Who was CEO"
print(result.confidence)         # 0.85
print(result.has_temporal_context())  # True
```

<a id="versioned-snapshots"></a>
### 版本化快照

```python
from semantica.kg import TemporalVersionManager

# 默认内存中；传入 storage_path="versions.db" 可持久化到 SQLite
versioner = TemporalVersionManager()

# create_snapshot 需要 author 和 description
versioner.create_snapshot(kg, version_label="2024-Q1",
                          author="user@example.com",
                          description="Q1 2024 baseline")

# 列出版本（不是 list_snapshots）
for v in versioner.list_versions():
    print(f"{v['label']:12s}  {v['author']}")

# 比较两个版本（不是 diff_versions）
diff = versioner.compare_versions("2023-Q4", "2024-Q1")
print(f"Entities added:      {diff['summary']['entities_added']}")
print(f"Relationships added: {diff['summary']['relationships_added']}")

# 检索某个版本（不是 restore_snapshot）
past_kg = versioner.get_version("2023-Q4")

# SHA-256 完整性校验
versioner.verify_checksum(past_kg)
```

<Tip>
  完整的类 API、领域示例（人事变动、政策演化、财务时间线）与配置选项请参见[时序智能](temporal.zh-CN.md)参考。
</Tip>


<a id="similarity-scoring"></a>
## 相似度评分

`SimilarityCalculator` 计算节点向量嵌入之间的余弦、欧几里得、曼哈顿与相关性相似度：

```python
from semantica.kg import SimilarityCalculator, NodeEmbedder

# 先计算结构化嵌入
embedder   = NodeEmbedder(method="node2vec", embedding_dimension=128)
embeddings = embedder.compute_embeddings(kg, ["Person", "Organization"], ["RELATED_TO"])

# 再按嵌入相似度比较节点
calc  = SimilarityCalculator()
score = calc.cosine_similarity(embeddings["Apple Inc."], embeddings["Google"])
print(f"Apple–Google structural similarity: {score:.3f}")

# 查找结构上相似的节点：返回节点 ID 的 List[str]
similar = embedder.find_similar_nodes(kg, "Apple Inc.", top_k=5)
for node_id in similar:
    print(node_id)
```


<a id="graph-analytics"></a>
## 图分析

<Tabs>
  <Tab title="中心性">
    跨五种算法衡量节点重要性。使用 `calculate_all_centrality()` 一次性运行全部度量。

    ```python
    from semantica.kg import CentralityCalculator

    calculator = CentralityCalculator()

    # 一次性运行所有中心性度量
    all_metrics = calculator.calculate_all_centrality(graph)

    # 或单独运行
    pagerank    = calculator.calculate_pagerank(graph, damping_factor=0.85)
    betweenness = calculator.calculate_betweenness_centrality(graph)
    closeness   = calculator.calculate_closeness_centrality(graph)

    # 取出最重要的前 10 个节点
    top_nodes = calculator.get_top_nodes(pagerank, top_k=10)
    ```

    | 方法 | 最适用于 |
    | :------ | :-------- |
    | `calculate_degree_centrality()` | 连接数最多的节点 |
    | `calculate_pagerank()` | 基于链接的影响力（类似 Google PageRank） |
    | `calculate_betweenness_centrality()` | 瓶颈 / 桥接节点 |
    | `calculate_closeness_centrality()` | 距所有其他节点最近的节点 |
    | `calculate_eigenvector_centrality()` | 与其他高影响力节点相连的节点 |
  </Tab>
  <Tab title="社区发现">
    发现图中的簇与社区。Louvain 最快；Leiden 产出更高质量的划分。

    ```python
    from semantica.kg import CommunityDetector

    detector = CommunityDetector()

    # Louvain：快速、高质量（默认）
    communities = detector.detect_communities(graph, algorithm="louvain")

    # Leiden：质量更高，速度更慢
    communities = detector.detect_communities_leiden(graph, resolution=1.2)

    # 评估社区质量
    metrics = detector.calculate_community_metrics(graph, communities)
    print(f"Modularity: {metrics['modularity']:.3f}")
    print(f"Communities found: {metrics['num_communities']}")
    ```

    | 算法 | 优势 |
    | :--------- | :-------- |
    | Louvain | 快速、良好的模块度：适用于大图 |
    | Leiden | 最佳模块度：当质量比速度更重要时使用 |
    | Label Propagation | 近似线性时间：适用于超大图 |
    | K-Clique | 重叠社区：节点可属于多个群组 |
  </Tab>
  <Tab title="路径查找">
    在任意两节点之间查找最短路径与备选路由。

    ```python
    from semantica.kg import PathFinder

    finder = PathFinder()

    # Dijkstra 最短路径
    path = finder.dijkstra_shortest_path(graph, "Alice", "Bob")
    print(" → ".join(path["path"]))

    # 两节点间的所有最短路径
    paths = finder.all_shortest_paths(graph, "source", "target")

    # K-最短路径（备选路由）
    k_paths = finder.find_k_shortest_paths(graph, "source", "target", k=3)
    ```

    | 算法 | 用例 |
    | :--------- | :-------- |
    | Dijkstra | 加权最短路径：标准路由 |
    | A\* | 启发式引导搜索：在大型稀疏图上更快 |
    | BFS | 无权最短路径：仅按跳数 |
    | K-最短 | 多条备选路由 |
  </Tab>
  <Tab title="链路预测">
    预测缺失或未来的边。用于补全知识图谱或发现隐含关系。

    ```python
    from semantica.kg import LinkPredictor

    predictor = LinkPredictor(method="preferential_attachment")

    # 预测最可能缺失的前 20 条边
    predicted = predictor.predict_links(graph, top_k=20)
    for link in predicted:
        print(f"{link['source']} → {link['target']}  (score: {link['score']:.3f})")

    # 给某一对节点评分
    score = predictor.score_link(graph, "Alice", "CompanyX")
    ```

    | 算法 | 最适用于 |
    | :--------- | :-------- |
    | 优先连接 | 高度数节点的连接预测 |
    | 共同邻居 | 拥有共同连接的节点 |
    | Jaccard | 归一化的共同邻居重叠 |
    | Adamic-Adar | 加权的共同邻居（惩罚高度数中介节点） |
    | 资源分配 | 保守策略：忽略高度数中介节点 |
  </Tab>
  <Tab title="节点嵌入">
    使用 Node2Vec 计算结构化嵌入，再查找相似节点或喂给下游 ML。

    ```python
    from semantica.kg import NodeEmbedder, SimilarityCalculator

    # 计算 Node2Vec 嵌入
    embedder = NodeEmbedder(method="node2vec", embedding_dimension=128)
    embeddings = embedder.compute_embeddings(
        graph, ["Person", "Organization"], ["RELATED_TO"]
    )

    # 查找结构上相似的节点
    similar = embedder.find_similar_nodes(graph, "Apple Inc.", top_k=5)
    for node_id in similar:
        print(node_id)

    # 按嵌入相似度比较两个具体节点
    calc  = SimilarityCalculator()
    score = calc.cosine_similarity(embeddings["Apple Inc."], embeddings["Google"])
    print(f"Structural similarity: {score:.3f}")
    ```

    <Note>
      `find_similar_nodes` 返回 `List[str]`：一个节点 ID 列表，而非节点对象。请通过 `graph["nodes"]` 查找完整节点数据。
    </Note>
  </Tab>
</Tabs>


<a id="algorithm-summary"></a>
## 算法概览

| 类别 | 算法 | 用例 |
| :-------- | :---------- | :--------- |
| 节点嵌入 | Node2Vec | 结构相似性、节点表示 |
| 相似度 | 余弦、欧几里得、曼哈顿、相关性 | 节点匹配、推荐 |
| 路径查找 | Dijkstra、A\*、BFS、K-最短 | 路由规划、网络分析 |
| 链路预测 | 优先连接、Jaccard、Adamic-Adar | 网络补全 |
| 中心性 | 度数、中介、接近、PageRank | 影响力分析 |
| 社区发现 | Louvain、Leiden、Label Propagation | 社交聚类 |
| 连通性 | 分量、桥、密度 | 网络健壮性 |


<a id="graphvalidator"></a>
## GraphValidator

校验图结构：检查必填字段、重复 ID、悬空边，并可选地检测环与孤立节点：

```python
from semantica.kg import GraphValidator

validator = GraphValidator()
result    = validator.validate(kg)   # 接收 GraphBuilder.build() 返回的字典

if result.is_valid:
    print("Graph is valid")
else:
    for issue in result.issues:
        print(f"{issue.severity.value}: {issue.message}")
```

传入 `strict=True` 可将警告视为错误。传入一个包含 `"entity_types"` 与 `"relationship_types"` 键的 `schema` 字典，可针对已知类型词表进行校验。

<a id="configuration"></a>
## 配置

```yaml
kg:
  resolution:
    threshold: 0.9
    strategy: semantic

  temporal:
    enabled: true
    default_validity: infinite
```

- [图存储](graph_store.zh-CN.md) —— 在 Neo4j、FalkorDB 或 Apache AGE 中持久化图。
- [语义抽取](semantic_extract.zh-CN.md) —— 喂给 GraphBuilder 的实体与关系来源。
- [可视化](visualization.zh-CN.md) —— 交互式可视化知识图谱。
- [冲突](conflicts.zh-CN.md) —— 冲突检测与解决。

<a id="cookbooks"></a>
### 实战手册

- [构建知识图谱](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/07_Building_Knowledge_Graphs.ipynb)：KG 构建基础 · 入门
- [你的第一个知识图谱](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/08_Your_First_Knowledge_Graph.ipynb)：从实体抽取到可视化 · 入门
- [图分析](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/10_Graph_Analytics.ipynb)：中心性与社区发现 · 中级
- [高级图分析](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/02_Advanced_Graph_Analytics.ipynb)：PageRank、Louvain、最短路径 · 高级
- [时序知识图谱](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/10_Temporal_Knowledge_Graphs.ipynb)：时序逻辑与图演化 · 高级
