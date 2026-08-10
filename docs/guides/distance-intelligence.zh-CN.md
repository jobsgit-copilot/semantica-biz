---
title: "距离智能"
description: "将图谱邻近度分类为距离区间，沿路径计算置信度衰减，运行多算法路径查找，并在检索中融合语义相似度与结构邻近度。"
icon: "route"
---

**[English](distance-intelligence.md)** · **简体中文（当前）**

`ContextGraph` 距离智能回答了纯语义相似度无法回答的结构性问题：给定两个节点，它们在图谱拓扑、路径权重和推断置信度方面的精确关系是什么？利用它来用跳数和置信度衰减标注归因链，按到锚点的结构邻近度对检索结果进行排名，并浮现隐含连接供分析师审查。

<a id="what-is-distance-intelligence"></a>
## 什么是距离智能？

距离智能量化和分析知识图谱中节点之间的结构关系。它提供关于图谱路径的详细元数据，包括跳数、距离区间、置信度衰减和路径分析。

**距离元数据** 包括跳数（节点之间的边数）、距离区间（如 "direct"、"near"、"distant" 等语义类别）、置信度衰减（沿路径累积的信任度）和路径分析（在节点之间寻找最优路由）。

**跳数** 衡量你从一个节点到达另一个节点必须遍历的边数。跳数为 1 表示直接连接；3 表示你通过 2 个中间节点遍历。

**距离区间** 将原始跳数转换为有意义的类别："direct"（0-1 跳）、"near"（2-3 跳）、"mid-range"（4-6 跳）和 "distant"（7+ 跳）。这些类别有助于解释图谱距离的语义含义。

**置信度衰减** 沿路径乘以边权重来计算累积信任。如果每条边的权重为 0.8，则 3 跳路径的置信度衰减为 0.8³ = 0.512，表示对连接的中等置信度。

**路径分析** 使用 Dijkstra 最短路径或 Yen k-最短路径等算法在节点之间寻找最优路由。

**距离智能与图谱分析：** 图谱分析计算整个图谱的统计度量（如中心性和社区）。距离智能专注于特定节点之间的具体路径和关系。

**距离智能与图遍历：** 简单遍历沿边查找邻居。距离智能使用权重、路径和衰减度量来量化这些连接的质量和置信度。

<a id="why-use-distance-intelligence"></a>
## 为什么使用距离智能？

**置信度感知检索。** 距离智能不再平等对待所有图谱连接，而是按路径置信度对结果加权，使通过更强、更直接关系连接的节点获得更高排名。

**关系发现。** 不仅发现两个实体是否相连，还发现它们如何连接、通过哪些中介、以及跨完整路径的置信度水平。

**因果分析。** 在知识图谱中追踪因果链，每一步都有量化的置信度，这对决策追踪和审计轨迹至关重要。

**先例搜索。** 通过分析结构相似性和路径模式（而非仅内容相似度）查找相似的过往案例。

**图谱感知排名。** 融合语义相似度和图谱邻近度，以浮现纯向量搜索会遗漏的上下文相关结果。

<a id="when-to-use-when-not-to-use"></a>
## 何时使用 / 何时不使用

**在以下情况使用距离智能：**
- 路径质量很重要的多跳推理
- 需要置信度评估的归因分析
- 因果链分析和决策追踪
- 从特定锚点进行的邻近度加权检索
- 查找替代连接路由以进行验证
- 按内容相关性和结构邻近度双重排名结果

**简单图遍历可能足够的情况：**
- 查找节点的直接邻居
- 无置信度加权的基本图谱探索
- 所有边具有相同重要性的情况
- 简单的可达性查询（A 能到达 B 吗？）

**距离智能可能不必要的情况：**
- 单跳邻居查找
- 边权重不代表有意义置信度的图谱
- 简单的存在性查询而非质量评估
- 路径分析增加不必要复杂性的场景

<Info>
  距离智能馈入邻近度混合检索（`retrieve()` 上的 `proximity_weight`）、因果链分析（`trace_decision_causality()`）和高级先例搜索（`find_precedents_hybrid()`）。通过在邻居查询中传入 `include_distance_metadata=True` 或在检索调用中传入 `proximity_weight > 0` 来启用它。
</Info>

<a id="distance-bands-turning-hop-counts-into-meaning"></a>
## 距离区间：将跳数转化为含义

距离智能中的第一个工具是 `classify_path_distance`——它将任何广度优先搜索（BFS）深度映射到具有语义含义的人类可读区间。

```python
from semantica.utils.helpers import classify_path_distance

print(classify_path_distance(0))   # "direct"  — 同一节点
print(classify_path_distance(1))   # "direct"  — 单条边
print(classify_path_distance(2))   # "near"    — 两跳邻域
print(classify_path_distance(3))   # "near"
print(classify_path_distance(5))   # "mid-range"
print(classify_path_distance(9))   # "distant" — 需谨慎处理
```

| 区间 | 跳数范围 | 实际含义 |
| :--- | :-------- | :------------------------ |
| `"direct"` | 0–1 | 直接关系——高置信度推断 |
| `"near"` | 2–3 | 两跳邻域——密切相关，可靠 |
| `"mid-range"` | 4–6 | 可达但语义上分离 |
| `"distant"` | 7+ | 弱耦合——谨慎处理推断 |

这些区间自动出现在每个使用 `include_distance_metadata=True`、`proximity_weight > 0` 或 `trace_decision_causality()` 的结果上。你无需手动计算它们——它们附加在结果上。

<a id="confidence-decay-how-trust-erodes-along-a-path"></a>
## 置信度衰减：信任如何沿路径递减

沿路径的每一跳将累积置信度乘以边权重。乘积——`confidence_decay`——是判断多跳推断是否可信的唯一最有用信号。

<Info>
  **置信度衰减与边权重：** 置信度衰减直接取决于图谱中的边权重。权重应代表置信度、信任度、相关性或类似的领域特定信号，其中较高的值表示更强的关系。未加权的图谱（所有边权重为 1.0）不会产生有意义的衰减分析。
</Info>

<Info>
  **稠密图谱警告：** 非常稠密的图谱会使路径分析在计算上变得昂贵，且结果更难解释。稠密连接创建了许多具有相似权重的可能路径，使基于距离的排名区分度降低。
</Info>

```python
from semantica.context import ContextGraph

graph = ContextGraph()
graph.add_node("apt29",       "ThreatActor",   "APT29 / NOBELIUM")
graph.add_node("hammertoss",  "Malware",       "HAMMERTOSS C2 tool")
graph.add_node("twitter_c2",  "Infrastructure","APT29 Twitter C2 channel")
graph.add_node("nato_target", "Target",        "NATO defense contractor")

graph.add_edge("apt29",      "hammertoss",  "deploys", weight=0.95)
graph.add_edge("hammertoss", "twitter_c2",  "uses",    weight=0.88)
graph.add_edge("twitter_c2", "nato_target", "reaches", weight=0.80)

# 请求附带完整距离元数据的邻居
neighbors = graph.get_neighbors(
    "apt29",
    hops=3,
    include_distance_metadata=True,
)

for n in neighbors:
    print("{:20s}  band={:10s}  decay={:.3f}  hop={}".format(
        n["id"],
        n["distance_band"],
        n["confidence_decay"],
        n["hop"],
    ))
```

输出：

```text
hammertoss            band=direct     decay=0.950  hop=1
twitter_c2            band=near       decay=0.836  hop=2
nato_target           band=near       decay=0.669  hop=3
```

`nato_target` 节点可达——但置信度衰减仅为 0.669。这意味着从 APT29 与 NATO 承包商之间的连接得出的任何推断都携带跨三跳累积的 33% 不确定性预算。在 "near" 区间，推断仍然可用；在具有类似衰减的 "distant" 区间，你会将其标记为需要人工审查。

<a id="getting-all-neighbors-with-distance-metadata"></a>
## 获取带距离元数据的所有邻居

`get_neighbor_distances` 返回给定跳数深度内每个可达节点，按最低置信度阈值过滤，按最近跳数优先、每跳内最强衰减优先排序。

```python
neighbors = graph.get_neighbor_distances(
    "apt29",
    hops=4,
    relationship_types=["deploys", "uses", "reaches"],
    min_confidence=0.60,   # 丢弃 confidence_decay < 0.60 的节点
)

# 每个结果字典包含：
# "id"、"type"、"content"    — 节点身份
# "relationship"             — 最后一跳的边类型
# "weight"                   — 最后一跳的边权重
# "hop"                      — 从锚点的 BFS 深度
# "distance_band"            — "direct" / "near" / "mid-range" / "distant"
# "confidence_decay"         — 路径上所有边权重的乘积
# "path_to_anchor"           — 从锚点到此节点的完整节点 ID 列表

for n in neighbors:
    print("[{:10s}] {:20s}  decay={:.3f}  path={}".format(
        n["distance_band"],
        n["id"],
        n["confidence_decay"],
        " → ".join(n["path_to_anchor"]),
    ))
```

<a id="finding-the-shortest-path-between-two-nodes"></a>
## 查找两个节点之间的最短路径

`PathFinder` 暴露了五种路径算法。选择哪种取决于你需要单一最廉价路径、多条替代路径，还是从源节点出发的所有路径。

```python
from semantica.kg import PathFinder

pf = PathFinder()
```

**Dijkstra——加权最短路径。** 将此作为默认选择。它找到边权重总和最小的路径。

```python
path = pf.dijkstra_shortest_path(
    graph  = graph,
    source = "apt29",
    target = "nato_target",
)
length = pf.path_length(graph, path)
print("Shortest path:", " → ".join(path))
print("Path length  :", round(length, 3))
```

**BFS——无权重最短路径。** 当你想要最少跳数而不考虑边权重时使用。

```python
path = pf.bfs_shortest_path(graph, "apt29", "nato_target")
print("Hop count:", len(path) - 1)
```

**K-最短路径——Yen 算法。** Yen 算法在两个节点之间找到多条替代路径，按总路径成本排名。当你需要替代归因链、冗余分析或佐证路由时使用。找到三条最短路径并展示它们都汇聚到同一目标，比单条路径是更强的证据。

```python
k_paths = pf.find_k_shortest_paths(graph, "apt29", "nato_target", k=3)

for i, path in enumerate(k_paths, 1):
    length = pf.path_length(graph, path)
    band   = classify_path_distance(len(path) - 1)
    print("Path {} [{}] length={:.3f}: {}".format(
        i, band, length, " → ".join(path)
    ))
```

**从源节点出发的所有最短路径。** 当你想要映射从锚点可达的所有内容并理解结构布局时使用。

```python
all_paths = pf.all_shortest_paths(graph, source="apt29")

for target, paths in all_paths.items():
    path = paths[0]
    print("{:20s}  hops={}  path={}".format(
        target, len(path) - 1, " → ".join(path)
    ))
```

<a id="proximity-blended-retrieval"></a>
## 邻近度混合检索

标准语义检索按与查询的文本相似度对结果进行排名。邻近度混合检索添加了第二个信号：每个结果在图谱中与锚点的结构距离有多近？`proximity_weight` 参数控制混合比例。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    hybrid_alpha=0.5,
)

context.store([
    "APT29 exploited CVE-2024-3400 in PAN-OS targeting NATO governments.",
    "HAMMERTOSS is APT29's C2 tool using Twitter as a covert channel.",
    "SUNBURST was a supply chain implant targeting SolarWinds Orion.",
], extract_entities=True, extract_relationships=True)

# 70% 语义 + 30% 图谱邻近度，锚定在 APT29
results = context.retrieve(
    "nation-state C2 infrastructure",
    max_results      = 8,
    use_graph        = True,
    anchor_node      = "APT29",
    max_hops         = 3,
    proximity_weight = 0.30,
    min_score        = 0.20,
)

for r in results:
    print("[{:.3f}]  band={:10s}  hop={}  decay={:.3f}  {}".format(
        r.get("combined_score", r["score"]),
        r.get("distance_band",   "-"),
        r.get("hop_distance",    "-"),
        r.get("confidence_decay", 0),
        r["content"][:70],
    ))
```

当 `proximity_weight > 0` 时，每个结果获得 `proximity_score`、`combined_score`、`hop_distance`、`distance_band`、`confidence_decay` 和 `path_to_anchor`——让你完整了解每个结果为何排名在其所在位置。

<a id="finding-structurally-similar-nodes"></a>
## 查找结构相似的节点

当你想知道图谱中哪些其他节点与给定节点行为相似——相同的连接模式或相似的文本——`find_similar_nodes` 暴露了当前在 `ContextGraph` 上实现的模式。

```python
# 内容相似度——节点内容字段上的文本重叠
content_similar = graph.find_similar_nodes(
    "CVE-2024-3400",
    similarity_type = "content",
    top_k           = 5,
)

# 结构相似度——具有相似邻域拓扑的节点
struct_similar = graph.find_similar_nodes(
    "CVE-2024-3400",
    similarity_type = "structural",
    top_k           = 5,
)

for n in content_similar:
    print("[{:.3f}] {}  {}".format(n["score"], n["type"], n["id"]))
```

`similarity_type="content"` 比较节点文本/内容，而 `similarity_type="structural"` 比较邻域拓扑。其他值目前回退到内容相似度，因此请将 `"embedding"` 保留给底层 KG API，而非 `ContextGraph.find_similar_nodes()`。

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防——CTI/威胁">

查找从 C2 IP 到威胁行为者的主要归因路径，然后查找所有替代佐证路径，以在归因结论进入情报产品之前加强归因论证。

```python
from semantica.context import ContextGraph
from semantica.kg import PathFinder
from semantica.utils.helpers import classify_path_distance

graph = ContextGraph(advanced_analytics=True)

for node_id, ntype, content in [
    ("apt29",       "ThreatActor",   "APT29 / NOBELIUM / Cozy Bear"),
    ("hammertoss",  "Malware",       "HAMMERTOSS C2 backdoor"),
    ("twitter_c2",  "Infrastructure","APT29 Twitter C2 (steganography)"),
    ("github_c2",   "Infrastructure","APT29 GitHub dead-drop resolver"),
    ("as200651",    "Network",       "APT29 hosting cluster AS200651"),
    ("nato_gov",    "Target",        "NATO government agency"),
]:
    graph.add_node(node_id, ntype, content)

graph.add_edge("apt29",     "hammertoss",  "deploys",   weight=0.95)
graph.add_edge("hammertoss","twitter_c2",  "c2_via",    weight=0.88)
graph.add_edge("hammertoss","github_c2",   "c2_via",    weight=0.82)
graph.add_edge("twitter_c2","as200651",    "hosted_on", weight=0.90)
graph.add_edge("github_c2", "as200651",    "resolves",  weight=0.76)
graph.add_edge("as200651",  "nato_gov",    "targets",   weight=0.85)

pf = PathFinder()

# 主要归因路径
primary = pf.dijkstra_shortest_path(graph, "apt29", "nato_gov")
length  = pf.path_length(graph, primary)
band    = classify_path_distance(len(primary) - 1)
print("Primary [{}] length={:.3f}: {}".format(band, length, " → ".join(primary)))

# 三条佐证路径
k_paths = pf.find_k_shortest_paths(graph, "apt29", "nato_gov", k=3)
for i, path in enumerate(k_paths, 1):
    l = pf.path_length(graph, path)
    b = classify_path_distance(len(path) - 1)
    print("Alt {}: {} [{}, length={:.3f}]".format(i, " → ".join(path), b, l))

# 从 APT29 可达且置信度 >= 60% 的所有节点
neighbors = graph.get_neighbor_distances("apt29", hops=4, min_confidence=0.60)
print("\nReachable from APT29 (confidence >= 60%):")
for n in neighbors:
    print("  [{:10s}]  decay={:.3f}  {}".format(
        n["distance_band"], n["confidence_decay"], n["id"]
    ))
```

</Tab>

<Tab title="安全——SOC/事件">

映射横向移动攻击图谱，查找攻击者可能到达域控制器的所有替代路由，并按置信度衰减对每条路由评分，以确定优先阻断哪些路径。

```python
from semantica.context import ContextGraph
from semantica.kg import PathFinder

graph = ContextGraph()

for node_id, ntype, content in [
    ("attacker_ip", "ExternalHost", "Attacker 185.220.101.47"),
    ("wkstn047",    "Host",         "Compromised workstation WKSTN-047"),
    ("svc_backup",  "Account",      "Stolen service account SVC-BACKUP"),
    ("jump_server", "Host",         "Jump server JUMP-01"),
    ("dc01",        "Host",         "Domain controller DC01"),
    ("ad_forest",   "Asset",        "Active Directory forest root"),
]:
    graph.add_node(node_id, ntype, content)

graph.add_edge("attacker_ip","wkstn047",    "initial_access",   weight=0.90)
graph.add_edge("wkstn047",   "svc_backup",  "credential_theft", weight=0.85)
graph.add_edge("svc_backup", "jump_server", "lateral_move",     weight=0.78)
graph.add_edge("jump_server","dc01",        "lateral_move",     weight=0.88)
graph.add_edge("wkstn047",   "dc01",        "direct_smb",       weight=0.60)
graph.add_edge("dc01",       "ad_forest",   "controls",         weight=0.98)

pf = PathFinder()

# 到 DC01 的所有路由——先阻断哪条？
routes = pf.find_k_shortest_paths(graph, "attacker_ip", "dc01", k=3)
for i, route in enumerate(routes, 1):
    length = pf.path_length(graph, route)
    print("Route {} (length={:.3f}): {}".format(i, length, " → ".join(route)))

# 如果 DC01 被攻陷，什么会暴露？
exposed = graph.get_neighbor_distances("dc01", hops=2, min_confidence=0.70)
for n in exposed:
    print("Exposed: {:15s}  [{:8s}]  decay={:.3f}".format(
        n["id"], n["distance_band"], n["confidence_decay"]
    ))
```

</Tab>

<Tab title="生命科学——临床/制药">

映射药物-酶抑制链，计算多步代谢途径的置信度衰减，并使用邻近度混合检索来浮现结构上最相关的药物相互作用证据。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.kg import PathFinder

graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    graph_expansion=True,
)

for node_id, ntype, content in [
    ("warfarin",    "Drug",   "Warfarin — anticoagulant, narrow therapeutic window"),
    ("amiodarone",  "Drug",   "Amiodarone — antiarrhythmic"),
    ("cyp2c9",      "Enzyme", "CYP2C9 — hepatic cytochrome P450"),
    ("bleeding",    "ADE",    "Major bleeding adverse event"),
    ("cyp3a4",      "Enzyme", "CYP3A4 — major metabolising enzyme"),
    ("simvastatin", "Drug",   "Simvastatin — HMG-CoA reductase inhibitor"),
]:
    graph.add_node(node_id, ntype, content)

graph.add_edge("amiodarone", "cyp2c9",     "inhibits",    weight=0.92)
graph.add_edge("cyp2c9",     "warfarin",   "metabolises", weight=0.95)
graph.add_edge("warfarin",   "bleeding",   "risk_of",     weight=0.88)
graph.add_edge("amiodarone", "cyp3a4",     "inhibits",    weight=0.80)
graph.add_edge("cyp3a4",     "simvastatin","metabolises", weight=0.90)

pf = PathFinder()

# 相互作用链：amiodarone → bleeding（3 跳，衰减 = 0.92 × 0.95 × 0.88 = 0.769）
path   = pf.dijkstra_shortest_path(graph, "amiodarone", "bleeding")
length = pf.path_length(graph, path)
print("Interaction chain: {} (length: {:.3f})".format(" → ".join(path), length))

# 从 amiodarone 出发的所有路径——它能到达什么？
all_paths = pf.all_shortest_paths(graph, "amiodarone")
for target, paths in all_paths.items():
    p = paths[0]
    print("  {:15s}  hops={} path={}".format(target, len(p)-1, " → ".join(p)))

# 锚定在 amiodarone 的邻近度混合检索
context.store([
    "CYP2C9 inhibition by amiodarone reduces warfarin metabolism, raising INR.",
    "Patients on warfarin and amiodarone have 4-fold increased major bleeding risk.",
], extract_entities=True, extract_relationships=True)

results = context.retrieve(
    "anticoagulant enzyme inhibition",
    anchor_node      = "amiodarone",
    max_hops         = 3,
    proximity_weight = 0.35,
    max_results      = 5,
)
for r in results:
    print("[{:.3f}]  {:10s}  {}".format(
        r.get("combined_score", r["score"]),
        r.get("distance_band", "-"),
        r["content"][:80],
    ))
```

</Tab>

<Tab title="银行——风险/合规">

对从宏观经济冲击到特定贷款组合的因果距离进行评分，查找所有敞口路由，并记录带有完整因果链注释的压力测试决策。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.kg import PathFinder
from semantica.utils.helpers import classify_path_distance

graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    graph_expansion=True,
    decision_tracking=True,
)

for node_id, ntype, content in [
    ("rate_hike",      "MacroEvent", "Central bank +300bps rate cycle"),
    ("real_estate",    "Sector",     "UK residential real estate"),
    ("cre_sector",     "Sector",     "UK commercial real estate"),
    ("ltv_stress",     "RiskFactor", "LTV ratios deteriorate under higher rates"),
    ("dscr_stress",    "RiskFactor", "DSCR falls below 1.0 at +300bps"),
    ("resi_portfolio", "Portfolio",  "Retail mortgage book £4.2bn"),
    ("cre_portfolio",  "Portfolio",  "CRE lending book £1.8bn"),
    ("provision",      "Impact",     "Expected credit loss provision increase"),
]:
    graph.add_node(node_id, ntype, content)

graph.add_edge("rate_hike",    "real_estate",    "depresses", weight=0.88)
graph.add_edge("rate_hike",    "cre_sector",     "depresses", weight=0.85)
graph.add_edge("real_estate",  "ltv_stress",     "causes",    weight=0.78)
graph.add_edge("cre_sector",   "dscr_stress",    "causes",    weight=0.82)
graph.add_edge("ltv_stress",   "resi_portfolio", "exposes",   weight=0.90)
graph.add_edge("dscr_stress",  "cre_portfolio",  "exposes",   weight=0.88)
graph.add_edge("resi_portfolio","provision",      "increases", weight=0.75)
graph.add_edge("cre_portfolio", "provision",      "increases", weight=0.80)

pf = PathFinder()

# 加息如何到达 ECL 拨备？
k_routes = pf.find_k_shortest_paths(graph, "rate_hike", "provision", k=3)
for i, route in enumerate(k_routes, 1):
    length = pf.path_length(graph, route)
    hops   = len(route) - 1
    band   = classify_path_distance(hops)
    print("Route {} [{:10s}] length={:.3f}: {}".format(
        i, band, length, " → ".join(route)
    ))

# 完整风险敞口图
exposed = graph.get_neighbor_distances("rate_hike", hops=5, min_confidence=0.65)
print("\nRisk exposure map from rate hike:")
for n in exposed:
    print("  [{:10s}]  decay={:.3f}  {}".format(
        n["distance_band"], n["confidence_decay"], n["id"]
    ))

# 记录并追踪压力测试决策
dec_id = context.record_decision(
    category       = "stress_test",
    scenario       = "+300bps rate shock — portfolio stress test Q3 2025",
    reasoning      = "LTV deterioration on resi book within tolerance; CRE DSCR breach requires provisioning",
    outcome        = "increase_provision_cre_book",
    confidence     = 0.87,
    decision_maker = "risk_model_v3",
)

chains = graph.trace_decision_causality(dec_id, max_depth=5)
for chain in chains:
    print("\nCausal chain ({} hops, {}, decay={:.3f}):".format(
        chain["hop_count"], chain["distance_band"], chain["confidence_decay"]
    ))
    print("  Interpretation:", chain["interpretation"])
```

</Tab>

</Tabs>

<a id="common-pitfalls"></a>
## 常见陷阱

**将置信度衰减视为统计概率。** 置信度衰减是基于边权重的启发式度量，而非统计概率。衰减值 0.6 并不意味着"60% 概率"——它意味着基于你领域特定权重分配的路径强度。

**使用未加权图谱却期望有意义的衰减。** 如果所有边的权重为 1.0，无论路径长度如何，置信度衰减始终为 1.0，无法在路径之间提供有用的区分。请分配反映关系强度的有意义权重。

**在稠密图谱上过度探索路径。** 具有许多互连节点的稠密图谱可能生成指数级大数量的路径。限制 `max_hops`，使用 `min_confidence` 阈值，并考虑简单的邻居查找是否足够。

**在简单邻居查找就足够时过度使用距离分析。** 如果你只需要直接邻居或一跳连接，基本的图遍历比完整的距离智能分析更简单、更快。

**检索过多的图谱邻域。** 大的 `max_hops` 值可能检索到淹没下游处理的庞大子图。从 2-3 跳开始，仅在特定用例需要时增加。

<a id="related-guides"></a>
## 相关指南

- [上下文图谱](context-graphs.zh-CN.md) — `ContextGraph` 节点和边模型；`add_edge(weight=...)` 为置信度衰减提供输入
- [图谱分析](graph-analytics.zh-CN.md) — 中心性、社区发现、Node2Vec 向量嵌入、链接预测
- [智能体记忆](agent-memory.zh-CN.md) — 邻近度混合检索（`proximity_weight`）将距离智能集成到记忆搜索中
- [决策智能](decision-intelligence.zh-CN.md) — `trace_decision_causality()` 用于带距离注释的因果链
- [推理与规则](reasoning.zh-CN.md) — `TemporalReasoningEngine` 用于在时间受限的图谱节点上进行 Allen 区间代数运算
