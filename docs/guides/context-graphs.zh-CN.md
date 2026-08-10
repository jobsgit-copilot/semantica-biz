---
title: "上下文图谱"
description: "Semantica 如何以线程安全的内存属性图谱来存储、链接、搜索和遍历知识 —— 具备时间效性窗口、跨图谱导航、邻近度混合检索，以及从对话到图谱的构建能力。"
icon: "diagram-project"
---

**[English](context-graphs.md)** · **简体中文（当前）**

`ContextGraph` 是一个线程安全的内存属性图谱，在每个节点和边上支持时间效性窗口，内置广度优先搜索（BFS）遍历、用于语义搜索的 FAISS 向量索引，以及通过 `AgentContext` 实现的邻近度混合检索。当多个智能体或线程向共享知识库写入数据，同时分析师对其进行实时查询时，即可使用它。

<a id="what-is-a-context-graph"></a>
## 什么是上下文图谱？

上下文图谱是一个属性图谱，将实体存储为**节点**、将关系存储为**边**，并附加元数据与时间效性信息。

**节点**表示你领域中的实体 —— 威胁攻击者、漏洞、公司、人员，或任何你想追踪的概念。每个节点都有一个 ID、一个类型、可选的内容文本以及元数据属性。

**边**表示实体之间的关系 —— "APT29 使用 SUNBURST"、"Alice 就职于 Acme Corp"，或"CVE-2024-3400 影响 SolarWinds"。每条边都有一个类型、权重和可选的元数据。

**元数据**以键值对形式在节点和边上存储额外属性 —— 地理来源、置信度得分、时间戳，或任何领域特定属性。

像 ContextGraph 这样的**属性图谱**与简单网络的区别在于：它在节点和边上都支持丰富的元数据，使其适用于那些关系需要上下文和属性、复杂真实世界领域。

<a id="why-use-a-context-graph"></a>
## 为什么要使用上下文图谱？

**关系分析。** 图谱结构揭示实体之间如何连接 —— 谁攻击谁、什么利用什么、哪些决策导致了哪些结果。从单个文档中不明显的关联，在连接后会变得清晰。

**多跳推理。** 通过遍历图谱而非关键词匹配来回答"APT29 在 3 步之内能到达哪些资产？"或"哪些漏洞影响我们的关键系统？"等问题。

**上下文保留。** 与单纯的向量搜索不同，图谱保留实体之间的关系。当你找到一个相关的威胁攻击者时，可以立即看到其工具、目标和基础设施。

**时态状态追踪。** 追踪关系何时有效、基础设施何时活跃，或决策何时做出。可查询历史状态，或仅过滤到当前信息。

<a id="when-to-use--when-not-to-use"></a>
## 何时使用 / 何时不应使用

**在以下场景使用上下文图谱：**
- 实体之间存在丰富关系的复杂领域
- 多跳推理与遍历查询
- 实体关系和状态的时态追踪
- 与具有清晰实体关系的结构化数据集成
- 多个来源贡献相互连接数据的协作环境

**在以下场景，简单向量搜索可能就足够了：**
- 关系无关紧要的纯文档检索
- 单跳相似度搜索
- 结构尚未明确定义的探索性研究
- 对不含实体关系的非结构化文本进行只读分析

**在以下场景，图谱可能是不必要的：**
- 简单的关键词或语义搜索任务
- 没有演进关系的静态文档集合
- 单用户、短期的分析项目
- 搭建复杂度超过关系复杂度的场景

<Info>
  ContextGraph 是一种**内存数据结构**。所有节点、边和元数据都存储在 Python 字典与列表中。对于独立图谱，请用 `save_to_file()` 持久化状态。使用 `AgentContext` 时，请改调 `AgentContext.save()` —— 它可一步保存图谱、FAISS 向量索引和记忆。若需在已填充图谱之上进行解析操作 —— 中心性排名、社区发现、节点嵌入、链接预测 —— 请参阅[图谱分析指南](graph-analytics.zh-CN.md)。若需记录和查询以节点形式存储的决策，请参阅[决策智能指南](decision-intelligence.zh-CN.md)。
</Info>

<a id="constructing-the-graph"></a>
## 构建图谱

你可以通过两种方式构建 ContextGraph：

**手动构建** —— 使用 `add_node()` 和 `add_edge()` 以编程方式添加节点和边。这让你对图谱结构拥有完全控制，在你拥有结构化数据或希望构建特定关系模式时非常理想。

**自动抽取工作流** —— 将文档列表传给 `AgentContext.store()` 并设置 `extract_entities=True` 或 `extract_relationships=True`。这要求在 `AgentContext` 构造器中设置 `knowledge_graph=`；抽取过程随后会为检测到的实体创建节点，为发现的关系创建边。传入 `store()` 的单个字符串仅作为记忆项存储，不会触发图谱构建。

最简单的图谱无需任何参数：

```python
from semantica.context import ContextGraph

graph = ContextGraph()
```

对于一个还将运行分析的威胁情报工作负载，请在构造时启用子组件 —— 它们会惰性初始化，但必须预先声明：

```python
graph = ContextGraph(
    advanced_analytics  = True,
    centrality_analysis = True,
    community_detection = True,
    node_embeddings     = True,
)
```

该图谱完全由 Python 字典与一个可重入锁（`threading.RLock`）支撑。没有外部服务、没有数据库连接、没有网络调用。你只需一次导入，就能在单元测试中搭建起一个功能完整的情报图谱。

<a id="adding-your-first-entities"></a>
## 添加你的第一批实体

每个实体都作为一个节点加入，带有一个类型、可选的内容字符串，以及任意数量的元数据 kwargs：

```python
# add_node(node_id, node_type, content=None, **properties) -> None
# 所有额外的 kwargs 都进入 ContextNode.metadata

graph.add_node(
    "APT29",
    "ThreatActor",
    "Russian state-sponsored group, also known as COZY BEAR",
    origin="Russia",
    motivation="espionage",
    first_seen="2008",
)

graph.add_node(
    "SUNBURST",
    "Malware",
    "Supply-chain backdoor embedded in SolarWinds Orion updates",
    family="backdoor",
    first_seen="2019-10",
    platforms=["Windows"],
)

graph.add_node(
    "CVE-2020-10148",
    "Vulnerability",
    "SolarWinds Orion API authentication bypass",
    cvss=10.0,
    affected_product="SolarWinds Orion",
)

graph.add_node(
    "45.142.212.100",
    "C2Domain",
    "Command-and-control server observed in SUNBURST campaign",
    asn="AS29550",
    country="Netherlands",
)

graph.add_node(
    "SolarWinds",
    "Victim",
    "SolarWinds Corporation — software supply chain victim",
    sector="Technology",
)
```

<Info>
  不存在 `properties={}` 参数。请将所有元数据字段作为直接的关键字参数传入。调用 `add_node("x", "t", properties={"k": "v"})` 会在元数据中一个字面名为 `properties` 的键下存储该字典 —— 这不是你想要的结果。
</Info>

现在用带类型、带权重的边将它们连接起来：

```python
# add_edge(source_id, target_id, edge_type="related_to", weight=1.0, **properties) -> None

graph.add_edge("APT29",          "SUNBURST",         "uses",       weight=1.0)
graph.add_edge("SUNBURST",       "CVE-2020-10148",   "exploits",   weight=0.95)
graph.add_edge("SUNBURST",       "SolarWinds",       "targets",    weight=1.0)
graph.add_edge("APT29",          "45.142.212.100",   "operates",   weight=0.9)
graph.add_edge("SUNBURST",       "45.142.212.100",   "beacons_to", weight=0.85)
graph.add_edge("CVE-2020-10148", "SolarWinds",       "affects",    weight=1.0)
```

检查你拥有的内容：

```python
s = graph.stats()
print(f"Nodes: {s['node_count']}, Edges: {s['edge_count']}, Density: {s['density']:.4f}")
# Nodes: 5, Edges: 6, Density: 0.3000

print("Node types:", s["node_types"])   # {"ThreatActor": 1, "Malware": 1, ...}
print("Edge types:", s["edge_types"])   # {"uses": 1, "exploits": 1, ...}
```

<a id="temporal-validity-intel-has-an-expiry-date"></a>
## 时间效性 —— 情报有有效期

使用 `valid_from` 和 `valid_until` 为节点和边标记活动窗口，使时态查询排除过时数据：

```python
# 该 C2 域名仅在攻击行动窗口内活跃
graph.add_node(
    "45.142.212.100",
    "C2Domain",
    "SUNBURST C2 — active during campaign",
    asn="AS29550",
    valid_from="2019-10-01T00:00:00",
    valid_until="2020-12-17T00:00:00",   # DarkHalo C2 关停日期
)

# 一条有效期有限的检测规则
graph.add_node(
    "SIGMA-SUNBURST-001",
    "DetectionRule",
    "Sigma rule: SUNBURST beacon pattern",
    rule_type="sigma",
    valid_from="2020-12-13T00:00:00",
    valid_until="2021-06-30T23:59:59",   # 在观察到更新的 TTP 后废弃
)

# 时态边的工作方式相同
graph.add_edge(
    "APT29", "45.142.212.100", "operates",
    weight=0.9,
    valid_from="2019-10-01T00:00:00",
    valid_until="2020-12-17T00:00:00",
)
```

现在提问：2020 年 12 月 1 日（攻击行动期间）哪些节点是活跃的？

```python
from datetime import datetime

# at_time 必须是 datetime 对象 —— 而非 ISO 字符串
active = graph.find_active_nodes(
    node_type="C2Domain",
    at_time=datetime(2020, 12, 1, 0, 0, 0),
)
print(f"Active C2 domains on 2020-12-01: {len(active)}")
# Active C2 domains on 2020-12-01: 1  (45.142.212.100 仍在其窗口内)

# 与今天对比 —— 该 C2 已过期
active_now = graph.find_active_nodes(node_type="C2Domain")  # 默认为 datetime.now()
print(f"Active C2 domains today: {len(active_now)}")
# Active C2 domains today: 0

# 完整时态快照 —— 仅包含在给定时刻有效的节点和边
snapshot = graph.state_at(datetime(2020, 12, 1, 0, 0, 0))
print(f"Active nodes: {len(snapshot['nodes'])}")
print(f"Active edges: {len(snapshot['edges'])}")
```

这就是你防止今天的查询返回"APT29 当前运营 45.142.212.100"的方式 —— 该边处于其有效窗口之外，不会出现在时态查询中。

<a id="finding-nodes"></a>
## 查找节点

`find_node()` 按 ID 检索，`find_nodes()` 按类型或元数据过滤：

```python
# find_node(node_id) -> Optional[Dict]
# 返回的键："id"、"type"、"content"、"metadata" —— 而非 "node_id" 或 "node_type"

actor = graph.find_node("APT29")
if actor:
    print(actor["id"])       # "APT29"
    print(actor["type"])     # "ThreatActor"
    print(actor["content"])  # "Russian state-sponsored group..."
    print(actor["metadata"]) # {"origin": "Russia", "motivation": "espionage", ...}

# find_nodes(node_type=None, skip=0, limit=None) -> List[Dict]
all_actors = graph.find_nodes(node_type="ThreatActor")
all_vulns  = graph.find_nodes(node_type="Vulnerability")
```

<a id="traversing-the-graph"></a>
## 遍历图谱

BFS 遍历直接回答可达性问题：

```python
# get_neighbors(node_id, hops=1, relationship_types=None,
#               min_weight=0.0, include_distance_metadata=False) -> List[Dict]
# 每条结果：{"id"、"type"、"content"、"relationship"、"weight"、"hop"}

neighbors = graph.get_neighbors("APT29", hops=2)
for n in neighbors:
    print(f"  hop={n['hop']}  [{n['relationship']}]  {n['id']}  ({n['type']})")

# hop=1  [uses]       SUNBURST         (Malware)
# hop=1  [operates]   45.142.212.100   (C2Domain)
# hop=2  [exploits]   CVE-2020-10148   (Vulnerability)
# hop=2  [targets]    SolarWinds       (Victim)
# hop=2  [beacons_to] 45.142.212.100   (C2Domain)  —— 也可通过 hop-1 到达
```

过滤为仅跟随特定边类型 —— 当你只想追踪利用链、排除其他关系类型的噪声时很有用：

```python
exploit_chain = graph.get_neighbors(
    "APT29",
    hops=3,
    relationship_types=["uses", "exploits", "affects"],
)
```

当你需要基于图谱距离理解一条连接有多大把握时，请启用距离元数据。每条结果会获得一个 `confidence_decay` 乘数 —— 距离更远的节点会被降权：

```python
neighbors = graph.get_neighbors(
    "APT29",
    hops=3,
    include_distance_metadata=True,
)
for n in neighbors:
    print(f"  {n['id']:30s}  band={n['distance_band']:8s}  decay={n['confidence_decay']:.3f}")

# APT29 的直接 SUNBURST 边：  band=direct   decay=1.000
# 通过 SUNBURST 到达的 CVE：  band=near     decay=0.850
# 通过 CVE 到达的 SolarWinds：band=mid      decay=0.700
```

要追踪从起始节点到特定目标的路径，请使用带 `include_distance_metadata=True` 的 `get_neighbors()`。每条结果包含一个 `path_to_anchor` 列表，显示从源到该邻居的精确节点 ID 序列：

```python
# 带 include_distance_metadata=True 的 get_neighbors 返回 path_to_anchor
neighbors = graph.get_neighbors("APT29", hops=3, include_distance_metadata=True)
for n in neighbors:
    if n["id"] == "SolarWinds":
        print(" → ".join(n["path_to_anchor"]))
        # APT29 → SUNBURST → SolarWinds
```

<a id="handling-concurrent-writes"></a>
## 处理并发写入

`ContextGraph` 通过包裹每一次变更的可重入锁（`threading.RLock`）来处理并发写入 —— 你无需自行添加同步：

```python
import threading
from semantica.context import ContextGraph

graph = ContextGraph()

def misp_ingest_worker(events):
    for event in events:
        graph.add_node(event["id"], event["type"], event["value"])
        for attr in event.get("attributes", []):
            graph.add_edge(event["id"], attr["value"], "has_attribute")

def nvd_ingest_worker(cves):
    for cve in cves:
        graph.add_node(cve["id"], "Vulnerability", cve["description"], cvss=cve["cvss"])
        graph.add_edge(cve["id"], cve["product"], "affects")

# 两个线程安全地写入同一图谱
t1 = threading.Thread(target=misp_ingest_worker, args=(misp_events,))
t2 = threading.Thread(target=nvd_ingest_worker, args=(nvd_batch,))
t1.start(); t2.start()
t1.join(); t2.join()

print(graph.stats())
```

该锁是可重入的，因此自身也会获取锁的内部调用（例如 `add_edge()` 内部调用 `find_node()`）不会死锁。

<a id="semantic-search-via-agentcontext"></a>
## 通过 AgentContext 进行语义搜索

`AgentContext` 用一个 FAISS 向量索引包装图谱，并允许你按语义相似度检索，可选地混合图邻近度：

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

graph = ContextGraph()
# ... （如上所示填充 CTI 节点）

context = AgentContext(
    vector_store    = VectorStore(backend="faiss", dimension=768),
    knowledge_graph = graph,
    hybrid_alpha    = 0.5,       # 50% 语义 / 50% 结构权重
    decision_tracking = True,
)

# 存储情报摘要 —— 这些会变得可搜索
context.store("APT29 operated SUNBURST backdoor via SolarWinds supply chain compromise")
context.store("45.142.212.100 is a C2 server associated with the SUNBURST campaign")
context.store("CVE-2020-10148 allows unauthenticated API access in SolarWinds Orion")

# 以图邻近度混合进行检索
# anchor_node="APT29" 表示在图谱中靠近 APT29 的节点得分更高
results = context.retrieve(
    "APT29 infrastructure and C2 servers",
    max_results      = 10,
    anchor_node      = "APT29",
    max_hops         = 2,
    proximity_weight = 0.3,    # 30% 图邻近度，70% 语义得分
    use_graph        = True,
)

for r in results:
    # "score"          —— 基础语义相似度（始终存在）
    # "combined_score" —— 混合得分（当 proximity_weight > 0 时存在）
    # "distance_band"  —— "direct" / "near" / "mid" / "far"
    score = r.get("combined_score", r.get("score", 0))
    print(f"[{score:.3f}]  {r.get('content', '')[:70]}")
```

`proximity_weight` 是 `retrieve()` 上的一个**按调用参数**，而非构造器设置。这意味着不同的查询可以在同一上下文对象上使用不同的混合比例 —— 宽泛的语义搜索用 `proximity_weight=0.0`，而聚焦邻域的遍历用 `proximity_weight=0.5`。

<a id="cross-graph-navigation"></a>
## 跨图谱导航

`link_graph()` 连接两个独立的图谱，`cross_graph_path()` 查找跨越边界的路径：

```python
from semantica.context import ContextGraph

actor_graph  = ContextGraph()
victim_graph = ContextGraph()

actor_graph.add_node("APT29",    "ThreatActor", "APT29")
actor_graph.add_node("SUNBURST", "Malware",     "SUNBURST backdoor")
actor_graph.add_edge("APT29", "SUNBURST", "uses")

victim_graph.add_node("SolarWinds", "Victim", "SolarWinds Corporation")
victim_graph.add_node("Treasury",   "Victim", "US Department of Treasury")
victim_graph.add_edge("SolarWinds", "Treasury", "supply_chain_compromised")

link_id = actor_graph.link_graph(
    victim_graph,
    "APT29",
    "SolarWinds",
    link_type="targets",
)

other_graph, target_node_id = actor_graph.navigate_to(link_id)

sw = other_graph.find_node(target_node_id)
if sw:
    print("Reached:", sw["id"])

result = actor_graph.cross_graph_path(
    "APT29",
    victim_graph,
    "Treasury",
)

if result.get("reachable"):
    print(f"Reached in {result['hop_count']} hops")
# APT29 → SUNBURST → SolarWinds → Treasury
```

<a id="serialization-and-persistence"></a>
## 序列化与持久化

在每次摄取循环之后，将图谱保存到磁盘。重启时，将其恢复 —— 整个节点和边集合都会被保留：

```python
# 保存
graph.save_to_file("cti_graph.json")

# 恢复
restored = ContextGraph(advanced_analytics=True)
restored.load_from_file("cti_graph.json")

print(restored.stats())

# to_dict() 给你原始的可序列化字典
d = graph.to_dict()
# d["nodes"]      → 节点字典列表
# d["edges"]      → 边字典列表
# d["statistics"] → {"node_count": int, "edge_count": int}
```

如果图谱曾用 `link_graph()` 创建过跨图谱链接，加载之后请调用 `resolve_links()` 以恢复实时导航 —— 对象引用无法被序列化，因此必须手动重新连接：

```python
g1b, g2b = ContextGraph(), ContextGraph()
g1b.load_from_file("actor_graph.json")
g2b.load_from_file("victim_graph.json")
g1b.resolve_links({g2b.graph_id: g2b})
```

要进行完整的会话持久化（图谱 + FAISS 向量索引 + 记忆），请使用 `AgentContext.save()` / `AgentContext.load()`：

```python
context.save("agent_state/")

# 之后，重启时：
context2 = AgentContext(
    vector_store    = VectorStore(backend="faiss", dimension=768),
    knowledge_graph = ContextGraph(),
)
context2.load("agent_state/")
```

<a id="common-pitfalls"></a>
## 常见陷阱

**重复实体。** 将"APT-29"、"APT29"和"Cozy Bear"作为分开的节点添加会在它们本应是同一实体时割裂图谱。请预先使用一致的命名约定，或在摄取之后使用[去重](deduplication.zh-CN.md)指南中的 `detect_duplicates()` 和 `EntityMerger` 合并它们。

**不一致的命名约定。** 将"ThreatActor"、"threat_actor"和"Threat-Actor"混用作节点类型会破坏按类型过滤的查询。请选择一种约定，并在所有数据来源中统一执行。

**过度连接节点。** 在同一文档中提到的每个实体之间都创建边会引入噪声。请聚焦于有意义的关系 —— 直接的因果、成员关系或功能依赖，而非共现。

**存储不必要的信息。** 将源数据的每个字段都作为元数据添加会膨胀内存使用。请只包含查询、过滤或下游分析所需的属性。

**未能持久化重要的图谱状态。** 由于 ContextGraph 在内存中，除非你调用 `save_to_file()` 或 `AgentContext.save()`，否则关闭应用会丢失所有节点和边。在长时间运行的摄取过程中请定期持久化。

<a id="relationship-between-graph-structure-and-vector-search"></a>
## 图谱结构与向量搜索之间的关系

ContextGraph 结构与向量搜索服务于互补的目的：

- **图谱结构**捕获显式关系，并支持遍历、可达性分析和多跳推理
- **向量搜索**支持基于内容的语义相似度查询与模糊匹配

当通过 `AgentContext` 一起使用时，你可以混合这两种方法 —— 在查找语义相似内容的同时，提升那些在图谱中结构上靠近你起点的结果的排名。

<a id="domain-examples"></a>
## 领域示例

<Tabs>
  <Tab title="国防 —— CTI/威胁情报">
    三个独立的摄取工作线程同时向一个共享的 `ContextGraph` 写入（MISP、NVD、机密 STIX）。时间效性可防止过时的攻击行动数据出现在当前威胁查询中。

```python
from semantica.context import ContextGraph, AgentContext
from semantica.vector_store import VectorStore
from datetime import datetime

graph = ContextGraph(advanced_analytics=True, community_detection=True)

# 核心 CTI 实体
graph.add_node("APT29", "ThreatActor", "Russian GRU unit, COZY BEAR",
               origin="Russia", motivation="espionage")
graph.add_node("SUNBURST", "Malware", "SolarWinds supply chain backdoor",
               family="backdoor", platforms=["Windows"])
graph.add_node("CVE-2020-10148", "Vulnerability",
               "SolarWinds Orion API auth bypass", cvss=10.0)

# 时间受限的 C2 基础设施
graph.add_node("avsvmcloud.com", "C2Domain",
               "SUNBURST DNS C2 domain",
               valid_from="2019-10-01T00:00:00",
               valid_until="2020-12-18T00:00:00")

graph.add_edge("APT29",    "SUNBURST",        "deploys",    weight=1.0)
graph.add_edge("SUNBURST", "CVE-2020-10148",  "exploits",   weight=0.95)
graph.add_edge("SUNBURST", "avsvmcloud.com",  "beacons_to", weight=0.9,
               valid_from="2019-10-01T00:00:00",
               valid_until="2020-12-18T00:00:00")

# 当前正在活跃的 C2 基础设施有哪些？
active_c2 = graph.find_active_nodes(node_type="C2Domain")
print(f"Currently active C2 domains: {len(active_c2)}")
# Currently active C2 domains: 0  —— avsvmcloud.com 已于 2020 年过期

# 历史查询：攻击行动期间什么处于活跃？
campaign_c2 = graph.find_active_nodes(
    node_type="C2Domain",
    at_time=datetime(2020, 6, 1),
)
print(f"C2 domains active June 2020: {len(campaign_c2)}")
# C2 domains active June 2020: 1  —— avsvmcloud.com 当时活跃

# 遍历：从 APT29 出发的完整影响半径
blast_radius = graph.get_neighbors("APT29", hops=3,
                                   include_distance_metadata=True)
for n in blast_radius:
    print(f"  hop={n['hop']}  decay={n['confidence_decay']:.2f}  {n['id']}")
```

  </Tab>

  <Tab title="安全 —— SOC/事件响应">
    在一次活跃事件期间，主机是节点，观察到的横向连接是边。图谱可回答哪些主机处于关键路径上，以及从初始据点看攻击者的可达网络是什么样子。

```python
from semantica.context import ContextGraph

graph = ContextGraph(advanced_analytics=True)

# 受影响主机
for host in ["ws-finance-04", "srv-dc-01", "srv-file-02",
             "ws-hr-11", "srv-backup-01"]:
    graph.add_node(host, "Host", f"Windows host: {host}")

# 观察到的植入物
graph.add_node("COBALT-STRIKE-BEACON-01", "Implant",
               "Cobalt Strike beacon, staged from ws-finance-04")
graph.add_node("MIMIKATZ-DUMP-01", "Tool",
               "Credential dump observed on srv-dc-01")

# 横向移动边
graph.add_edge("ws-finance-04", "srv-dc-01",               "lateral_move", weight=0.9)
graph.add_edge("srv-dc-01",     "srv-file-02",             "lateral_move", weight=0.85)
graph.add_edge("srv-dc-01",     "srv-backup-01",           "lateral_move", weight=0.8)
graph.add_edge("ws-finance-04", "COBALT-STRIKE-BEACON-01", "hosts",        weight=1.0)
graph.add_edge("srv-dc-01",     "MIMIKATZ-DUMP-01",        "executes",     weight=1.0)

# 从初始据点的影响半径
reachable = graph.get_neighbors("ws-finance-04", hops=3,
                                 relationship_types=["lateral_move"])
print("Reachable via lateral movement:")
for n in reachable:
    print(f"  hop={n['hop']}  {n['id']}")

# 通过 path_to_anchor 找出从初始据点到备份服务器的路径
all_paths = graph.get_neighbors("ws-finance-04", hops=3,
                                relationship_types=["lateral_move"],
                                include_distance_metadata=True)
for n in all_paths:
    if n["id"] == "srv-backup-01":
        print(" → ".join(n["path_to_anchor"]))
        # ws-finance-04 → srv-dc-01 → srv-backup-01
```

  </Tab>

  <Tab title="生命科学 —— 临床/制药">
    一个临床试验知识图谱追踪药物、生物标志物、患者人群、不良事件和监管里程碑。每个监管里程碑都有一个有效窗口 —— 查询必须尊重这些窗口，以防过时的疗效数据与当前安全发现被并列引用。

```python
from semantica.context import ContextGraph
from datetime import datetime

graph = ContextGraph(advanced_analytics=True)

# 实体
graph.add_node("dapagliflozin",     "Drug",        "SGLT2 inhibitor, AstraZeneca")
graph.add_node("HbA1c-reduction",   "Biomarker",   "Primary endpoint: HbA1c change from baseline")
graph.add_node("T2D-adults-65plus", "Population",  "Type 2 diabetes, adults 65+, DECLARE-TIMI 58")
graph.add_node("DKA",               "AdverseEvent","Diabetic ketoacidosis, known SGLT2 risk")

# III 期数据节点 —— 申报受理后有效
graph.add_node("DECLARE-TIMI58-results", "ClinicalData",
               "Phase III CVOT results: dapagliflozin vs placebo",
               phase="III",
               primary_endpoint_met=True,
               valid_from="2019-01-11T00:00:00")   # NEJM 发表日期

graph.add_edge("dapagliflozin",          "HbA1c-reduction",    "primary_endpoint", weight=1.0)
graph.add_edge("dapagliflozin",          "T2D-adults-65plus",  "studied_in",       weight=1.0)
graph.add_edge("dapagliflozin",          "DKA",                "risk_of",          weight=0.7)
graph.add_edge("DECLARE-TIMI58-results", "dapagliflozin",      "evaluates",        weight=1.0)

# 仅检索截至某个监管评审日期时可得 的试验数据
active_data = graph.find_active_nodes(
    node_type="ClinicalData",
    at_time=datetime(2019, 6, 1),
)
print(f"Published trial data available June 2019: {len(active_data)}")
# Published trial data available June 2019: 1

# 遍历：在 2 跳之内对 dapagliflozin 已知些什么？
drug_neighbors = graph.get_neighbors("dapagliflozin", hops=2)
for n in drug_neighbors:
    print(f"  [{n['relationship']}]  {n['id']}  ({n['type']})")
```

  </Tab>

  <Tab title="银行 —— 风险/合规">
    一个交易对手风险图谱连接银行、SPV、风险敞口工具、担保方和监管实体。实体具有报告期效性 —— 一个交易对手的 CDS 敞口节点仅在其报告的那个季度有效。

```python
from semantica.context import ContextGraph
from datetime import datetime

graph = ContextGraph(advanced_analytics=True, community_detection=True)

# 实体
graph.add_node("BankA",      "Counterparty", "Tier-1 bank, EUR exposure 4.2B")
graph.add_node("SPV-EUR-01", "SPV",          "Structured vehicle, BankA sponsored")
graph.add_node("BankB",      "Counterparty", "Tier-2 bank, USD exposure 0.8B")
graph.add_node("CCP-LME",    "CCP",          "Central counterparty — LME metals")

# 2024 年 Q4 敞口节点 —— 仅对该报告季度有效
graph.add_node("BankA-BankB-CDS-Q42024", "Exposure",
               "CDS notional 400M, BankA writes protection on BankB",
               notional_eur=400_000_000,
               valid_from="2024-10-01T00:00:00",
               valid_until="2024-12-31T23:59:59")

graph.add_edge("BankA",      "SPV-EUR-01",             "sponsors",   weight=1.0)
graph.add_edge("BankA",      "BankB",                  "exposed_to", weight=0.8)
graph.add_edge("BankA",      "BankA-BankB-CDS-Q42024", "holds",      weight=1.0)
graph.add_edge("SPV-EUR-01", "CCP-LME",                "clears_via", weight=0.9)
graph.add_edge("BankB",      "CCP-LME",                "member_of",  weight=1.0)

# 传染路径：如果 BankA 违约，谁在下游？
downstream = graph.get_neighbors("BankA", hops=3, include_distance_metadata=True)
print("Contagion reach from BankA:")
for n in downstream:
    print(f"  hop={n['hop']}  decay={n['confidence_decay']:.2f}  {n['id']}")

# Q4 敞口全貌 —— 仅纳入 2024 年 Q4 有效的节点
q4_exposures = graph.find_active_nodes(
    node_type="Exposure",
    at_time=datetime(2024, 11, 15),
)
print(f"\nActive Q4 2024 exposures: {len(q4_exposures)}")

# 从 BankA 出发的可达性：识别压力测试范围内的所有实体
stress_reach = graph.get_neighbors("BankA", hops=2)
print(f"Stress-test reachable entities: {len(stress_reach)}")
for n in stress_reach:
    print(f"  hop={n['hop']}  {n['id']}")
```

  </Tab>
</Tabs>

<a id="related-guides"></a>
## 相关指南

- [图谱分析](graph-analytics.zh-CN.md) —— 在已填充的 `ContextGraph` 上进行中心性排名、社区发现、节点嵌入和链接预测
- [决策智能](decision-intelligence.zh-CN.md) —— 将决策记录为带类型节点、因果链分析、先例搜索和策略执行
- [摄取](ingest.zh-CN.md) —— 从 PDF、API、数据库、STIX 包和 RSS 源加载数据到图谱
- [去重](deduplication.zh-CN.md) —— 在插入之前检测并合并近似重复节点，以防止图谱割裂
- [推理](reasoning.zh-CN.md) —— 时间区间代数（Allen 关系）、前向/反向链接，以及基于知识图谱的 SPARQL
- [本体管理](ontology.zh-CN.md) —— 从 `graph.to_dict()` 为下游推理引擎导出正式的 OWL 本体
- [上下文模块参考](../reference/context.zh-CN.md) —— `AgentContext`、`ContextGraph`、`ContextNode`、`ContextEdge` 的完整 API
