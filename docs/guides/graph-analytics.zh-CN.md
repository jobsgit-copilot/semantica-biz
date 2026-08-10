---
title: "图分析"
description: "如何在知识图谱中找到最重要的节点、发现隐藏的社区、度量结构相似性并预测缺失的连接——附真实场景示例。"
icon: "chart-network"
---

**[English](graph-analytics.md)** · **简体中文（当前）**

`ContextGraph` 图分析——中心性、社区发现、Node2Vec 向量嵌入、链接预测与结构相似性——回答的是你的图谱*意味着*什么，而不仅仅是它包含什么。通过 `advanced_analytics=True` 一次性启用，即可对结构上最重要的节点进行排名、浮现隐藏的运营簇，并在隐含连接被正式观测到之前将其标记出来。

<a id="what-is-graph-analytics"></a>
## 图分析是什么？

图分析运用数学算法来发现知识图谱结构中的模式和属性。它超越了数据的存储与检索，转而分析关系本身。

**图分析 vs. 图遍历：** 遍历沿着已有的边找到相连的节点；而分析审视整张图的结构来发现模式——哪些节点最具影响力，哪些节点群构成社区，哪些连接是缺失的。

**图分析 vs. 推理：** 推理运用逻辑规则推导出新的事实；而分析运用统计和拓扑算法度量结构属性，如中心性、聚类和相似性。

图分析帮助你理解数据的*形态*和*重要性模式*，揭示那些从单个节点或边上看不出来的洞察。

<a id="why-use-graph-analytics"></a>
## 为什么使用图分析？

**发现具有影响力的实体。** 并非所有节点都同等重要。图分析能识别哪些实体在网络结构中最居中、连接最多，或战略位置最关键。

**社区发现。** 图分析通过分析连接密度模式揭示隐藏的簇和运营分组。聚集到一起的实体往往共享那些从元数据上看不出来的目的、来源或行为。

**关系发现。** 链接预测识别那些尚未被明确观测到但很可能存在的连接，帮助调查人员聚焦于最可能缺失的关系。

**调查支撑。** 图分析提供关于重要性和关联性的客观度量，帮助分析师优先决定先调查哪些实体、哪些关系值得深入审查。

**风险识别。** 中心性度量能识别出一旦被移除将最大程度瓦解网络的实体——这对理解单点故障、关键基础设施或高影响目标非常有用。

<a id="when-to-use-when-not-to-use"></a>
## 何时使用 / 何时不使用

**适合使用图分析的场景：**
- 大型图谱（100+ 节点），其模式从肉眼检查不明显
- 基于结构重要性来确定调查工作的优先级
- 发现隐藏的社区和运营簇
- 理解网络韧性和脆弱点
- 通过链接预测识别缺失的关系

**图遍历可能就足够的场景：**
- 跟踪特定实体之间已知的关系
- 探索某些节点周围的邻域
- 在已知实体之间寻找路径

**推理可能就足够的场景：**
- 应用已知的逻辑规则推导新事实
- 策略执行和基于规则的决策
- 关系遵循清晰逻辑模式的情况

**简单查询可能就足够的场景：**
- 模式在视觉上一目了然的小型图谱
- 直接查找特定实体或关系
- 你确切知道自己在找什么的场景

**下列情况下分析能提供价值：**
- 你需要理解整体结构和模式
- 人工检查会遗漏重要的结构属性
- 你想发现意料之外的关系或社区
- 关于重要性或相似性的客观度量能够指导决策

<a id="key-analytics-concepts"></a>
## 关键分析概念

**模块度（Modularity）** 度量图谱中社区划分的清晰程度。高模块度（>0.4）意味着检测到的社区内部连接很多、外部连接很少——表明存在真实的组织结构。

**Louvain 社区发现** 用于找到彼此之间连接比与图谱其余部分更密集的节点组。它对发现运营簇、组织单元或功能分组很有用。

**Node2Vec** 通过在图中模拟随机游走来为节点生成向量嵌入。在这些游走中出现于相似上下文的节点最终会得到相似的向量，即使它们之间没有直接相连。

**中心性度量** 衡量节点重要性的不同方面：
- 度（Degree）：直接连接有多少
- 中介（Betweenness）：一个节点出现在其他节点之间最短路径上的频率
- 特征向量（Eigenvector）：重要性基于其邻居本身重要
- PageRank：在图中随机游走时的重要程度

<Info>
  所有分析都要求在构造时设置 `advanced_analytics=True`。否则本指南中的每个方法都会抛出 `RuntimeError: advanced_analytics not enabled`。该标志会惰性初始化五个子组件：`CentralityCalculator`、`CommunityDetector`、`NodeEmbedder`、`LinkPredictor` 和 `SimilarityCalculator`。
</Info>

<a id="setting-up-the-analytical-graph"></a>
## 设置分析图谱

```python
from semantica.context import ContextGraph

graph = ContextGraph(
    advanced_analytics  = True,
    community_detection = True,
    node_embeddings     = True,
)
```

如果你正在从存储加载已有图谱，请传入同样的标志——子组件会从已加载的图谱状态初始化，而不是从一个全新的空图初始化。

<a id="the-one-call-snapshot"></a>
## 一次调用得到全景

在深入各项分析之前，先了解全貌。`analyze_graph_with_kg()` 会按顺序运行每一轮分析，并返回一份统一的报告：

```python
report = graph.analyze_graph_with_kg()

print(f"Nodes:       {report['graph_metrics']['node_count']}")
print(f"Edges:       {report['graph_metrics']['edge_count']}")
print(f"Density:     {report['graph_metrics']['density']:.4f}")
print(f"Communities: {len(report['community_analysis'])}")
```

对于一张有 2400 个节点、8700 条边的图谱，密度为 0.003 是完全正常的——知识图谱天然稀疏；大多数节点只连接到一个小邻域，而不是与所有节点相连。如果你看到密度高于 0.1，很可能是过度连接的枢纽节点把一切都拉到一起，这会扭曲中心性得分。

在每一批摄取结束时运行一次。结果给你一个基线，可与下一次运行对比——如果社区数量在两次摄取周期之间从 12 降到 4，说明新数据中有内容正在桥接此前相互独立的簇。这是一个在下次分析师简报之前值得调查的信号。

完整的报告结构：

```text
report
├── "graph_metrics"         → node_count, edge_count, density
├── "centrality_analysis"   → all five measures + per-node rankings
├── "community_analysis"    → detected communities + modularity score
├── "connectivity_analysis" → connected components, diameter, avg path length
├── "node_embeddings"       → node_id → List[float] (Node2Vec vectors)
└── "timestamp"             → ISO datetime of this analysis run
```

<a id="finding-the-kingpins-centrality-analysis"></a>
## 发现核心人物——中心性分析

节点并非生而平等。中心性分析为每个节点给出五个不同的得分，每个得分衡量一种不同的重要性。

关键直觉：一个节点可能*度*很低（直接连接很少），但*中介中心性*极高（一切都得经过它）。这就是一个**中介节点**——一旦被移除就会把图谱切割成互不相连碎片的节点。在威胁情报中，中介节点通常是基础设施节点：抗举报的托管商、共享的 C2 框架，或是每个攻击行动都要经过的中间加载器。

<a id="scoring-a-single-node"></a>
### 为单个节点打分

当你怀疑某个特定实体很重要时，先直接给它打分：

```python
scores = graph.get_node_centrality("APT29")

# degree      → 0.0420  (4.2% 的节点是直接邻居)
# betweenness → 0.1837  (APT29 位于 18% 的最短路径上)
# closeness   → 0.6123  (到任何其他节点的平均跳数为 1.6)
# eigenvector → 0.8941  (APT29 的大多数邻居本身也高度互联)
# pagerank    → 0.0089  (0.89% 的随机游走概率质量落在这里)

for measure, score in scores.items():
    print(f"{measure:12s}: {score:.4f}")
```

这里 0.18 的中介中心性得分非常显眼——它意味着 APT29 充当了图中近五分之一信息流的中介。如果一位分析师想理解"是什么把我们观测到的 TTP 与已知基础设施连接起来"，他们必须经过 APT29。这不仅仅是名声；这是结构上的权力。

<a id="bulk-rankings-across-the-full-graph"></a>
### 在整张图上进行批量排名

要一次性按全部五项度量对每个节点排名：

```python
from semantica.kg import CentralityCalculator

calc       = CentralityCalculator()
graph_dict = graph.to_dict()

all_measures = calc.calculate_all_centrality(graph_dict)

# 取中介中心性前 5 名——这些就是你的中介节点
betweenness_rankings = all_measures["centrality_measures"]["betweenness"]["rankings"]
print("Top brokers (betweenness):")
for entry in betweenness_rankings[:5]:
    print(f"  [{entry['score']:.4f}]  {entry['node']}")
```

如果你只关心其中两项度量，可以传入子集以避免全部计算：

```python
focused = calc.calculate_all_centrality(
    graph_dict,
    centrality_types = ["degree", "pagerank"],
)
```

<a id="which-measure-to-trust"></a>
### 该信任哪一项度量？

| 如果你想知道…… | 使用 |
| :--- | :--- |
| 谁的直接连接最多？ | `degree` |
| 谁一旦被移除会最大程度瓦解网络？ | `betweenness` |
| 谁能最快触达网络其余部分？ | `closeness` |
| 谁因为邻居重要而重要？ | `eigenvector` |
| 在图的随机游走中谁更重要？ | `pagerank` |

对于归因分析，**特征向量**中心性往往会浮现出最有意义的节点——高特征向量意味着你与其他高特征向量节点相连，这在威胁情报中对应于"公认重要"的实体聚集在一起的现象。

<a id="mapping-threat-actor-clusters-community-detection"></a>
## 勾勒威胁行为者簇——社区发现

中心性告诉你单个节点的信息。社区发现告诉你关于*群体*的信息——哪些节点构成了内部连接多于外部连接的紧密簇。

设想你的图谱里有二十个不同的威胁行为者节点。中心性可以分别给它们排名，但无法告诉你其中十二个实际上共享基础设施、工具模式和目标行业，从而构成一个单一的运营簇，而剩下八个分成两个独立的群体。社区发现能自动找出这一点。

```python
from semantica.kg import CommunityDetector

detector   = CommunityDetector()
result     = detector.detect_communities(graph.to_dict(), algorithm="louvain")

communities = result["communities"]       # List[List[str]] — node groups
assignments = result["node_assignments"]  # node_id → community index
modularity  = result["modularity"]        # 0.0–1.0 — partition quality

print(f"Found {len(communities)} communities  (modularity: {modularity:.3f})")

for i, members in enumerate(communities):
    print(f"\n  Community {i} — {len(members)} members")
    print(f"  Sample: {', '.join(members[:4])}")
```

**模块度高于 0.4** 通常被认为是有意义的——所发现的社区并非随机。如果你的图谱返回 0.71，这是一个强烈信号，表明聚类是真实的：你的威胁行为者确实构成了运营簇，而不仅仅是随机关联。

当你看到一个社区混合了你认为互不相关的威胁行为者时，这是一个值得调查的假设。也许 `COZY BEAR`、`VENOMOUS BEAR` 和 `FANCY BEAR` 都出现在同一个社区里，是因为它们共享 C2 基础设施——尽管传统上它们被归因到不同的 GRU 单位。

立即用以下代码可视化：

```python
from semantica.visualization import KGVisualizer

viz = KGVisualizer()
viz.visualize_communities(
    graph       = graph,
    communities = communities,
    output      = "interactive",
    file_path   = "threat_clusters.html",
)
```

<a id="teaching-the-graph-to-measure-distance-node2vec-embeddings"></a>
## 让图谱学会度量距离——Node2Vec 向量嵌入

中心性和社区发现是拓扑层面的——它们把边当作二元连接处理。Node2Vec 向量嵌入则更深入：它通过在图中模拟成千上万次随机游走来为每个节点生成一个稠密向量，学习哪些节点倾向于出现在相似的邻域里。

实际结果是：扮演相似*结构角色*的节点最终在嵌入空间中彼此靠近——即使它们之间没有直接的边。两个都位于威胁行为者和受害基础设施之间的 C2 域名会有相似的向量嵌入，即使它们处于图谱中完全不同的部分。

```python
from semantica.kg import NodeEmbedder
import numpy as np

embedder   = NodeEmbedder()     # requires: pip install semantica[embeddings]
embeddings = embedder.compute_embeddings(
    graph_store        = graph,   # ContextGraph instance — not a dict
    node_labels        = None,    # None = embed all node types
    relationship_types = None,    # None = traverse all edge types
)
# Returns: Dict[str, List[float]] — node_id → embedding vector

for node_id, vec in list(embeddings.items())[:3]:
    print(f"{node_id}: dim={len(vec)}, norm={np.linalg.norm(vec):.4f}")
```

这些向量在 Semantica 中成为一等公民——它们以 `node2vec_embedding` 的形式存储在 `Decision` 节点上，并被 `find_precedents_by_scenario()` 自动用作结构相似性分量。

<a id="what-does-this-attack-pattern-remind-me-of"></a>
## 这个攻击模式让我想起了什么？

一旦有了向量嵌入，你就可以问："我的图谱中哪些节点在结构上与 APT29 最相似？"这不是关于共享的边——而是关于共享的*角色*。还有哪些其他节点在图谱中扮演与 APT29 相同的位置？

```python
from semantica.kg import SimilarityCalculator

calc = SimilarityCalculator()

top5 = calc.find_most_similar(
    embeddings      = embeddings,
    query_embedding = embeddings["APT29"],
    top_k           = 5,
)

print("Nodes most similar to APT29 (by structural role):")
for node_id, score in top5:
    print(f"  [{score:.4f}]  {node_id}")
```

如果 `LAZARUS` 和 `APT38` 都以高于 0.85 的相似度出现在前列，这值得在分析师报告中注明：这些群体在图谱中扮演着结构上相同的角色，这可能支撑一个统一追踪假设，即便传统归因把它们分开。

要直接比较两个节点：

```python
score = calc.cosine_similarity(embeddings["APT29"], embeddings["LAZARUS"])
print(f"Structural similarity APT29 ↔ LAZARUS: {score:.4f}")
```

如果想要一条不需要预先计算全部嵌入的快捷路径：

```python
similar = graph.find_similar_nodes(
    "APT29",
    similarity_type = "structural",
    top_k           = 5,
)
for r in similar:
    print(f"  [{r['score']:.4f}]  {r['type']:20s}  {r['id']}")
```

<a id="anticipating-the-next-move-link-prediction"></a>
## 预判下一步——链接预测

链接预测回答的是另一个问题：不是"哪些节点相似"，而是"哪些边是*缺失*的？"图中两个节点之间缺少直接连接，可能不是因为连接不存在，而是因为你还没有观测到它。

实践中：你有 `APT29 → uses → SUNBURST` 和 `SUNBURST → targets → Windows Server 2019`。链接预测可能会以高分浮现出 `APT29 → targets → Windows Server 2019`——这条连接由传递性隐含，但尚未被明确画出。

```python
from semantica.kg import LinkPredictor

predictor   = LinkPredictor()
predictions = predictor.predict_links(graph.to_dict(), top_k=10)

print("Predicted missing links:")
for node1, node2, score in predictions:
    print(f"  [{score:.4f}]  {node1}  →  {node2}")
```

得分高于 0.8 值得分析师复审——这些不是随机的；它们是现有图谱拓扑强烈暗示的边。得分低于 0.5 是噪声。适合人工复审的最佳区间是 0.6–0.8：合理但尚未确认。

<Info>
  链接预测也可用于 `Decision` 节点，通过 `DecisionQuery.predict_decision_relationships(decision_id, top_k)`。参见[决策智能指南](decision-intelligence.zh-CN.md)，了解如何浮现过往决策之间的因果关系。
</Info>

<a id="understanding-your-decision-history"></a>
## 理解你的决策历史

当决策作为节点存储在图谱中时，`get_decision_insights()` 为你提供横跨所有决策的分析视图：

```python
insights = graph.get_decision_insights()

print(f"Total decisions: {insights['total_decisions']}")
print(f"Mean confidence: {insights['confidence_stats']['mean']:.2f}")

print("\nBy category:")
for category, count in sorted(insights["categories"].items(), key=lambda x: -x[1]):
    print(f"  {category:<30} {count}")

print("\nBy outcome:")
for outcome, count in sorted(insights["outcomes"].items(), key=lambda x: -x[1]):
    print(f"  {outcome:<20} {count}")
```

置信度统计很说明问题：如果平均置信度是 0.91，但最低值是 0.34，说明有人在以低置信度做决策，而该决策仍被记录为最终决策。这一差距值得在治理评审中标记出来。

`insights` 内部的 `"advanced_analytics"` 键包含了完整的 `analyze_graph_with_kg()` 结果——因此调用 `get_decision_insights()` 无需额外调用就能给你完整的分析图景。

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防——CTI 威胁网络">

你的 SOC 刚把三个威胁情报源合并成一张单一的 CTI 图谱。在本周分析师简报之前，你需要给最危险的行动者排名、识别隐藏的运营簇，并标记出尚未被正式跟踪的隐含连接。

```python
from semantica.context import ContextGraph
from semantica.kg import CentralityCalculator, CommunityDetector, LinkPredictor
from semantica.visualization import AnalyticsVisualizer

graph = ContextGraph(
    advanced_analytics  = True,
    community_detection = True,
    node_embeddings     = True,
)
# (Populated from MISP/OTX/OSINT ingest)

# Full snapshot
report = graph.analyze_graph_with_kg()
print(f"Graph: {report['graph_metrics']['node_count']} nodes, "
      f"{report['graph_metrics']['edge_count']} edges")

# Rank brokers — who, if taken down, fragments the network?
calc     = CentralityCalculator()
measures = calc.calculate_all_centrality(graph.to_dict())
brokers  = measures["centrality_measures"]["betweenness"]["rankings"][:5]

print("\nTop 5 broker nodes (betweenness):")
for entry in brokers:
    print(f"  [{entry['score']:.4f}]  {entry['node']}")

# Find operational clusters
detector = CommunityDetector()
result   = detector.detect_communities(graph.to_dict(), algorithm="louvain")
print(f"\n{len(result['communities'])} clusters  "
      f"(modularity {result['modularity']:.3f})")

# Flag implied connections
predictor   = LinkPredictor()
predictions = predictor.predict_links(graph.to_dict(), top_k=5)
print("\nHigh-confidence implied connections:")
for n1, n2, score in predictions:
    if score > 0.7:
        print(f"  [{score:.3f}]  {n1}  →  {n2}")
```

</Tab>

<Tab title="安全——活跃事件">

在一次活跃事件中，你的图谱已经扩展到包含告警节点、受影响主机、横向移动边和已识别的恶意软件。你需要理解爆炸半径：哪些主机是横向移动链中的结构性中介节点，以及哪些连接尚未被正式记录？

```python
from semantica.context import ContextGraph
from semantica.kg import CentralityCalculator, LinkPredictor

graph = ContextGraph(advanced_analytics=True)
# (Populated from SIEM alert correlation and EDR telemetry)

calc     = CentralityCalculator()
measures = calc.calculate_all_centrality(graph.to_dict())

# Betweenness finds the pivot hosts — the ones lateral movement flows through
pivot_hosts = measures["centrality_measures"]["betweenness"]["rankings"][:5]
print("Pivot hosts (lateral movement brokers):")
for entry in pivot_hosts:
    print(f"  [{entry['score']:.4f}]  {entry['node']}")

# Identify the attacker's likely next targets
predictor   = LinkPredictor()
predictions = predictor.predict_links(graph.to_dict(), top_k=10)

print("\nLikely next lateral moves (predicted):")
for src, dst, score in predictions:
    if score > 0.65:
        print(f"  [{score:.3f}]  {src}  →  {dst}")

# Score a specific high-value target
dc_scores = graph.get_node_centrality("DC-PROD-01")
print(f"\nDomain Controller centrality:")
print(f"  Betweenness: {dc_scores['betweenness']:.4f}")
print(f"  PageRank:    {dc_scores['pagerank']:.4f}")
# High betweenness + high pagerank on a DC = confirmed pivot point
```

</Tab>

<Tab title="生命科学——临床试验图谱">

你的临床试验平台维护着一张包含药物、生物标志物、患者群体、不良事件类型和监管决策的图谱。在每个试验阶段结束时，你需要了解代谢综合征药物簇是否真的与心血管簇相互独立——这种分离对联合用药风险评估很重要。

```python
from semantica.context import ContextGraph
from semantica.kg import CommunityDetector, NodeEmbedder, SimilarityCalculator

graph = ContextGraph(
    advanced_analytics  = True,
    community_detection = True,
    node_embeddings     = True,
)
# (Populated from trial records, PubMed, and FDA adverse event database)

# Find drug-disease clusters
detector = CommunityDetector()
result   = detector.detect_communities(graph.to_dict(), algorithm="louvain")
print(f"Clinical clusters: {len(result['communities'])}  "
      f"(modularity {result['modularity']:.3f})")

# Inspect the cluster containing dapagliflozin
drug_cluster_idx = result["node_assignments"].get("dapagliflozin")
if drug_cluster_idx is not None:
    cluster_members = result["communities"][drug_cluster_idx]
    print(f"\nCluster containing dapagliflozin ({len(cluster_members)} members):")
    for m in cluster_members[:8]:
        print(f"  {m}")

# Find drugs that play the same structural role as dapagliflozin
embedder   = NodeEmbedder()
embeddings = embedder.compute_embeddings(graph_store=graph)
calc       = SimilarityCalculator()

similar_drugs = calc.find_most_similar(
    embeddings      = embeddings,
    query_embedding = embeddings["dapagliflozin"],
    top_k           = 5,
)
print("\nStructurally similar to dapagliflozin:")
for drug, score in similar_drugs:
    print(f"  [{score:.4f}]  {drug}")
```

</Tab>

<Tab title="银行——交易对手风险">

你的风险管理团队把交易对手关系建模为一张图谱：银行、SPV、风险敞口工具和监管实体通过风险敞口、担保和所有权边相连。在季度压力测试之前，你需要识别哪些实体是系统性的——它们的失败会引发最广泛的级联。

```python
from semantica.context import ContextGraph
from semantica.kg import CentralityCalculator, CommunityDetector

graph = ContextGraph(advanced_analytics=True, community_detection=True)
# (Populated from regulatory filings, trade repositories, and internal exposure data)

calc     = CentralityCalculator()
measures = calc.calculate_all_centrality(graph.to_dict())

# Eigenvector finds "too connected to fail" — entities connected to other
# high-centrality entities
systemic = measures["centrality_measures"]["eigenvector"]["rankings"][:5]
print("Systemically connected entities:")
for entry in systemic:
    print(f"  [{entry['score']:.4f}]  {entry['node']}")

# Betweenness finds contagion brokers — entities that, if they defaulted,
# would disconnect otherwise-separate parts of the exposure graph
brokers = measures["centrality_measures"]["betweenness"]["rankings"][:5]
print("\nContagion brokers:")
for entry in brokers:
    print(f"  [{entry['score']:.4f}]  {entry['node']}")

# Find exposure clusters — groups with dense mutual exposure
detector = CommunityDetector()
result   = detector.detect_communities(graph.to_dict(), algorithm="louvain")
print(f"\n{len(result['communities'])} exposure clusters  "
      f"(modularity {result['modularity']:.3f})")
```

</Tab>

</Tabs>

<a id="common-pitfalls"></a>
## 常见陷阱

**把预测当成事实。** 链接预测和相似度得分是概率估计，不是已确认的关系。高链接预测得分暗示一个可能的连接，但在据此行动之前需要人工核验。

**重复实体。** 把 "APT-29"、"APT29" 和 "Cozy Bear" 作为分开的节点会人为降低它们的中心性得分并碎片化社区。在运行分析之前先对实体去重以获得准确结果。

**命名不一致。** 把 "ThreatActor"、"threat_actor" 和 "Actor" 混用作节点类型会破坏按节点类型分组的分析。请在所有数据源中使用一致的命名约定。

**过度解读分析结果。** 一个中介中心性高的节点在你当前图谱中结构上重要——但现实世界未必重要。分析揭示的是数据中的模式，而非关于领域的普遍真理。

**在过小的图谱上运行分析。** 社区发现和中心性度量在 100+ 节点且连接密度足够的图谱上最有意义。在该阈值以下图谱上的结果可能无法提供可靠的洞察。

**图谱质量差。** 重复实体、命名不一致和缺失的关系会直接影响分析准确性。在运行分析之前先对数据去重和归一化——算法会放大图谱中存在的任何结构，包括噪声。

<a id="what-the-numbers-mean"></a>
## 数字意味着什么

| 得分 | 它告诉你什么 |
| :--- | :--- |
| 中介中心性 > 0.15 | 关键中介节点——在任何归因或根因分析中优先调查此节点 |
| 模块度 > 0.4 | 社区是真实的，不是统计噪声——可以信任聚类结果 |
| 模块度 < 0.2 | 图谱过于稠密，在这一层级上无法形成有意义的社区结构 |
| 向量嵌入余弦相似度 > 0.85 | 两个节点在结构上近乎相同——有强力假设支撑统一追踪 |
| 链接预测得分 > 0.7 | 这条边几乎肯定存在——排队等待分析师确认 |
| 链接预测得分 0.5–0.7 | 合理但不确定——当作假设而非事实对待 |

<a id="related-guides"></a>
## 相关指南

- [上下文图谱](context-graphs.zh-CN.md)——构建并查询底层 `ContextGraph`
- [可视化](visualization.zh-CN.md)——把中心性排名和社区簇渲染成交互式仪表板
- [决策智能](decision-intelligence.zh-CN.md)——把链接预测和结构相似性应用于决策节点
- [GraphRAG](graphrag.zh-CN.md)——使用分析结果将 LLM 生成锚定在上下文最相关的子图上
