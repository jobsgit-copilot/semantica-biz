---
title: "可视化"
description: "将知识图谱、本体层次结构、向量嵌入投影、图分析和时间序列渲染为交互式 HTML 或静态图像文件。"
icon: "chart-network"
---

**[English](visualization.md)** · **简体中文（当前）**

`KGVisualizer`、`AnalyticsVisualizer`、`TemporalVisualizer` 和 `OntologyVisualizer` 在一次方法调用中即可将图谱字典、分析结果和本体转换为交互式 HTML 仪表板或静态图像。使用它们无需编写任何渲染代码，即可向利益相关者展示中心性排名、社区聚类、事件时间线以及前后快照差异。

<a id="what-is-visualization"></a>
## 什么是可视化？

可视化将图数据转换为人类可解读的交互式图表、网络图、时间线和其他视觉格式。它将抽象的图结构和分析结果转化为视觉表示，揭示模式、关系和洞见。

**可视化与分析：**分析计算数值度量，如中心性分数和社区归属。可视化将这些度量渲染为按重要性调整大小、按社区分组的彩色节点。

**可视化与推理：**推理从现有数据中推导出新的逻辑事实。可视化以视觉形式呈现既有事实和分析结果，以支持人类的解读和决策。

可视化帮助人类理解图结构、分析结果和时间模式，而这些仅从原始数据是很难解读的。

<a id="why-use-visualization"></a>
## 为什么使用可视化？

**视觉探索：**交互式图谱让你可以平移、缩放、悬停和筛选，从而探索那些以文本或表格形式会让人不知所措的大型网络。

**调查支持：**高亮实体之间的路径、按实体类型进行颜色编码以及按重要性调整节点大小，有助于分析师识别模式并集中调查精力。

**沟通：**视觉呈现让那些不直接处理数据的利益相关者也能理解复杂的图谱关系。

**报告：**静态可视化为书面报告、演示和监管提交提供证据和支持。

<a id="when-to-use-when-not-to-use"></a>
## 何时使用 / 何时不使用

**可视化适用于：**
- 向人类展示图结构和分析结果
- 在中型图谱（10-1000 个节点）中探索关系和模式
- 为利益相关者创建报告和演示
- 调查图谱中的特定路径或邻域
- 沟通来自分析或推理工作流的发现

**图遍历可能足以应对：**
- 程序化地探索关系
- 关于特定路径或连接的简单查询
- 不需要人类解读的自动化工作流

**分析可能更有用于：**
- 计算数值度量和排名
- 以程序化方式发现社区或中心性分数
- 不需要可视化的定量比较

**推理可能更有用于：**
- 通过逻辑推断派生新事实
- 基于规则的决策
- 自动化的策略执行

**可视化在以下情况变得不切实际：**
- 图谱超过约 1000 个节点（浏览器性能下降）
- 网络过于密集，难以视觉解读
- 你需要程序化分析而非人类解读

<a id="typical-visualization-workflow"></a>
## 典型可视化工作流

**图谱 → 筛选 → 可视化 → 解读 → 调查**

最有效的可视化遵循以下模式：

1. **从你的知识图谱开始**，来自 `ContextGraph` 或分析结果
2. **筛选到有意义的子图** — 避免可视化整个企业图谱
3. **选择合适的可视化形式** — 网络、时间线、热力图或排名
4. **解读视觉模式** — 聚类、中心节点、时间趋势
5. **调查有趣的发现** — 深入探索异常模式或离群点

始终在可视化之前进行筛选。一个 10,000 节点的企业图谱在被筛选为 50 个最中心的节点或某个感兴趣的特定实体周围的子图时，才会变得有意义。

<Info>
  **性能警告：**大型图谱（>1000 个节点）会导致浏览器性能问题，并在视觉上令人难以承受。交互式网络可视化在 10-1000 个节点时效果最佳。对于更大的图谱，请使用分析来识别最重要的子图，然后可视化这些筛选后的结果。
</Info>

<Info>
  所有可视化器都接受 `output="interactive"`（Plotly/pyvis HTML，在 Jupyter 中显示或保存到文件）或 `output="static"`（通过 Matplotlib 生成 PNG/SVG）。省略 `file_path` 可取回图形对象以便进一步操作。
</Info>

<a id="rendering-the-full-knowledge-graph"></a>
## 渲染完整的知识图谱

首先要展示给利益相关者的是完整网络 — 节点按实体类型着色，按度中心性调整大小，悬停时显示内容的工具提示。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.visualization import KGVisualizer

graph = ContextGraph(advanced_analytics=True)
ctx   = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    graph_expansion=True,
)
ctx.store([
    "APT29 exploited CVE-2024-3400 targeting NATO defense contractors.",
    "CVE-2024-3400 is a critical vulnerability in PAN-OS by Palo Alto Networks.",
    "HAMMERTOSS is APT29's C2 backdoor operating over Twitter and GitHub.",
    "APT29 conducted the SUNBURST supply chain attack against SolarWinds in 2020.",
], extract_entities=True, extract_relationships=True)

viz = KGVisualizer()

# 交互式网络 — 保存为 HTML，可在任何浏览器中打开
viz.visualize_network(
    graph         = graph.to_dict(),
    output        = "interactive",
    file_path     = "reports/threat_graph.html",
    node_color_by = "type",      # 按实体类型为每个节点着色
    node_size_by  = "degree",    # 更大的节点 = 更多连接
    hover_data    = ["type", "content"],
)
```

生成的 HTML 是完全自包含的 — 不需要服务器。将其作为文件附件共享，它可以在任何浏览器中渲染，支持平移、缩放和悬停。

对于适合 PDF 报告或幻灯片演示的静态 PNG：

```python
viz.visualize_network(
    graph     = graph.to_dict(),
    output    = "static",
    file_path = "reports/threat_graph.png",
    node_color_by = "type",
)
```

要高亮图谱中的某个特定归因路径 — 例如从 APT29 经由 SUNBURST 到 SolarWinds 的链条 — 请将节点 ID 作为 `highlight_path` 传入：

```python
viz.visualize_network(
    graph          = graph.to_dict(),
    output         = "static",
    file_path      = "reports/apt29_path.png",
    highlight_path = ["APT29", "SUNBURST", "SolarWinds"],
)
```

<a id="showing-community-structure"></a>
## 展示社区结构

社区发现之后，你会得到一个将社区标签映射到节点 ID 列表的字典。`KGVisualizer` 上的 `visualize_communities` 会将这些聚类叠加到网络上；`AnalyticsVisualizer.visualize_community_structure` 会转发到同一个社区图谱视图。

```python
from semantica.visualization import AnalyticsVisualizer

# 你从图分析中获得的社区字典
communities = {
    "node_assignments": {
        "apt29": 0,
        "hammertoss": 0,
        "nobelium": 0,
        "sunburst": 0,
        "cve-2024-3400": 1,
        "pan-os": 1,
        "globalprotect": 1,
        "solarwinds": 2,
        "orion-platform": 2,
        "cve-2020-10148": 2,
    },
    "num_communities": 3,
}

# 带社区着色的网络视图
viz.visualize_communities(
    graph       = graph.to_dict(),
    communities = communities,
    output      = "interactive",
    file_path   = "reports/communities_network.html",
)

# 独立的分解图表 — 用于"12 个聚类是什么？"的幻灯片
av = AnalyticsVisualizer()
av.visualize_community_structure(
    graph      = graph.to_dict(),
    communities = communities,
    output     = "interactive",
    file_path  = "reports/communities_breakdown.html",
)
```

<a id="plotting-centrality-rankings"></a>
## 绘制中心性排名

中心性字典将节点 ID 映射到分数。两次调用即可覆盖两种用例：节点大小反映中心性的网络视图，以及用于"连接数最多的前 10 个节点"幻灯片的独立排名条形图。

```python
centrality = {
    "centrality": {
        "apt29": 0.14,
        "cve-2024-3400": 0.11,
        "pan-os": 0.07,
        "hammertoss": 0.06,
        "nobelium": 0.05,
    }
}

# 按中心性分数着色并调整大小的网络
viz.visualize_centrality(
    graph          = graph.to_dict(),
    centrality     = centrality,
    centrality_type= "pagerank",
    output         = "interactive",
    file_path      = "reports/centrality_network.html",
)

# 独立的排名条形图 — 一目了然地展示最具影响力的节点
av.visualize_centrality_rankings(
    centrality      = centrality,
    centrality_type = "pagerank",
    output          = "interactive",
    file_path       = "reports/centrality_rankings.html",
)
```

<a id="analytics-charts-connectivity-and-degree-distribution"></a>
## 分析图表：连通性与度分布

运行图分析之后，另外两张图表可以补全全貌。连通性图表显示存在多少个不连通的组件以及每个组件有多大。度分布显示图谱的幂律形状 — 有助于确认你的图谱是无标度的（少数高度连接的枢纽，大量叶节点）。

```python
# 连通性 — 直接传递分析结果字典
# 可视化器读取的键："is_connected"、"num_components"、"component_sizes"
connectivity = {
    "is_connected":    False,
    "num_components":  3,
    "component_sizes": [42, 8, 2],
}

av.visualize_connectivity(
    connectivity = connectivity,
    output       = "interactive",
    file_path    = "reports/connectivity.html",
)

# 度分布 — 直接传递图谱字典
av.visualize_degree_distribution(
    graph     = graph.to_dict(),
    output    = "interactive",
    file_path = "reports/degree_distribution.html",
)
```

<Info>
  `visualize_connectivity` 接收的是连通性分析结果字典 — 而不是 `graph.to_dict()`。该字典必须包含 `"is_connected"`、`"num_components"` 和 `"component_sizes"`。从你的图分析输出中计算它并传递结果。
</Info>

<a id="drawing-a-timeline-of-events"></a>
## 绘制事件时间线

当你讲述的故事是时间维度的 — CVE 生命周期、事件时间线、攻击活动进展 — `TemporalVisualizer.visualize_timeline` 会将带时间戳的事件列表转换为可滚动的交互式图表。

```python
from semantica.visualization import TemporalVisualizer

tv = TemporalVisualizer()

# 事件在 "events" 键下的字典中传递
tv.visualize_timeline(
    temporal_data = {"events": [
        {"id": "pub",   "label": "CVE-2024-3400 published",      "timestamp": "2024-03-14T00:00:00"},
        {"id": "exp",   "label": "Zero-day exploitation begins",  "timestamp": "2024-03-26T00:00:00"},
        {"id": "patch", "label": "PAN-OS hotfix released",        "timestamp": "2024-04-14T00:00:00"},
        {"id": "rem",   "label": "Contractor remediation confirmed","timestamp": "2024-04-30T00:00:00"},
    ]},
    output    = "interactive",
    file_path = "reports/cve_timeline.html",
)
```

<a id="comparing-two-graph-snapshots-side-by-side"></a>
## 并排比较两个图谱快照

当问题是"3 月 14 日到 4 月 14 日之间发生了什么变化？"时，`visualize_snapshot_comparison` 从 `TemporalVersionManager` 获取两个命名快照，并渲染一个折线图，比较所提供快照之间的图谱指标（实体、关系、密度）。

```python
from semantica.change_management import TemporalVersionManager

vm    = TemporalVersionManager(storage_path="versions.db")
snap1 = vm.get_version("pre_patch_march_14")
snap2 = vm.get_version("post_patch_april_14")

# 将快照作为映射 标签 → 快照字典 的字典传入
tv.visualize_snapshot_comparison(
    snapshots = {
        "pre_patch_march_14":   snap1,
        "post_patch_april_14":  snap2,
    },
    output    = "interactive",
    file_path = "reports/snapshot_diff.html",
)
```

<a id="tracking-graph-growth-over-time"></a>
## 跟踪图谱随时间的增长

利益相关者评审的最终图表是增长曲线 — 过去一年图谱累积了多少节点和边？`visualize_metrics_evolution` 接收一个历史字典和一个并行的时间戳列表。

```python
# 从 TemporalVersionManager 快照构建
versions = vm.list_versions()
versions.sort(key=lambda v: v["timestamp"])

timestamps      = [v["timestamp"][:10] for v in versions]
metrics_history = {
    "node_count": [len(v.get("nodes", [])) for v in versions],
    "edge_count": [len(v.get("edges", [])) for v in versions],
}

tv.visualize_metrics_evolution(
    metrics_history = metrics_history,
    timestamps      = timestamps,
    output          = "interactive",
    file_path       = "reports/graph_growth.html",
)
```

或者直接从已知的季度里程碑填充历史字典：

```python
tv.visualize_metrics_evolution(
    metrics_history = {
        "node_count": [50, 142, 309, 481],
        "edge_count": [88, 387, 821, 1340],
    },
    timestamps = ["2025-01-01", "2025-04-01", "2025-07-01", "2025-10-01"],
    output     = "interactive",
    file_path  = "reports/graph_growth.html",
)
```

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防 — CTI/威胁">

一份完整的分析师简报包：交互式威胁网络、社区分解、CVE 时间线和图谱增长曲线 — 全部在晨会前从一个实时 CTI 图谱生成。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.visualization import KGVisualizer, AnalyticsVisualizer, TemporalVisualizer
import os

graph = ContextGraph(advanced_analytics=True)
ctx   = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    graph_expansion=True,
)
ctx.store([
    "APT29 used HAMMERTOSS for C2 via Twitter and GitHub in 2020.",
    "APT29 infrastructure cluster: 185.220.101.0/24, AS200651.",
    "SolarWinds supply chain compromise attributed to APT29, campaign SUNBURST.",
    "APT29 leveraged OAuth token theft against cloud workloads in 2023.",
], extract_entities=True, extract_relationships=True)

os.makedirs("reports", exist_ok=True)

kg_viz = KGVisualizer()

# 完整网络 — 用于分析师门户的交互式版本
kg_viz.visualize_network(
    graph         = graph.to_dict(),
    output        = "interactive",
    file_path     = "reports/cti_network.html",
    node_color_by = "type",
    node_size_by  = "degree",
    hover_data    = ["type", "content"],
)

# 用于幻灯片演示的归因路径 PNG
kg_viz.visualize_network(
    graph          = graph.to_dict(),
    output         = "static",
    file_path      = "reports/apt29_sunburst_path.png",
    highlight_path = ["APT29", "SUNBURST", "SolarWinds"],
)

# 连通性概览
av = AnalyticsVisualizer()
av.visualize_connectivity(
    connectivity = {"is_connected": True, "num_components": 1, "component_sizes": [24]},
    output       = "interactive",
    file_path    = "reports/connectivity.html",
)

# CVE-2024-3400 事件时间线
tv = TemporalVisualizer()
tv.visualize_timeline(
    temporal_data = {"events": [
        {"id": "pub",   "label": "CVE-2024-3400 published",     "timestamp": "2024-03-14"},
        {"id": "exp",   "label": "Zero-day exploitation",        "timestamp": "2024-03-26"},
        {"id": "patch", "label": "PAN-OS hotfix released",       "timestamp": "2024-04-14"},
        {"id": "rem",   "label": "Remediation confirmed",        "timestamp": "2024-04-30"},
    ]},
    output    = "interactive",
    file_path = "reports/cve_timeline.html",
)
```

</Tab>

<Tab title="安全 — SOC/事件">

在活跃事件期间，SOC 生成一个横向移动网络（高亮攻击路径）、一个用于识别哪些主机最关键的中心性排名，以及一个展示谁连接到谁的关系矩阵。

```python
from semantica.context import ContextGraph
from semantica.visualization import KGVisualizer, AnalyticsVisualizer

graph = ContextGraph(advanced_analytics=True)

for node_id, ntype, content in [
    ("wkstn-047",  "Host",   "Compromised workstation WKSTN-047"),
    ("dc01",       "Host",   "Domain controller DC01"),
    ("jsmith",     "User",   "Compromised user jsmith"),
    ("psexec",     "Tool",   "PsExec lateral movement tool"),
    ("t1021",      "MITRE",  "T1021.002 SMB/Admin Shares"),
]:
    graph.add_node(node_id, ntype, content)

graph.add_edge("wkstn-047", "dc01",    "lateral_movement", weight=1.0)
graph.add_edge("jsmith",    "wkstn-047","session_on",      weight=0.9)
graph.add_edge("psexec",    "wkstn-047","executed_on",     weight=1.0)
graph.add_edge("psexec",    "t1021",   "implements",       weight=0.95)

viz = KGVisualizer()

# 高亮横向移动路径的事件网络
viz.visualize_network(
    graph          = graph.to_dict(),
    output         = "interactive",
    file_path      = "soc/incident_graph.html",
    node_color_by  = "type",
    highlight_path = ["wkstn-047", "dc01"],
    hover_data     = ["type", "content"],
)

# 关系矩阵 — 谁连接到什么
viz.visualize_relationship_matrix(
    graph     = graph.to_dict(),
    output    = "interactive",
    file_path = "soc/rel_matrix.html",
)

# 中心性排名 — 哪台主机最关键？
av = AnalyticsVisualizer()
av.visualize_centrality_rankings(
    centrality      = {"wkstn-047": 0.35, "dc01": 0.28, "psexec": 0.22, "jsmith": 0.15},
    centrality_type = "degree",
    output          = "interactive",
    file_path       = "soc/centrality.html",
)
```

</Tab>

<Tab title="生命科学 — 临床/制药">

一次药物重定位探索：交互式药物-靶点-疾病网络、OWL 类层次结构、UMAP 向量嵌入投影，以及用于发现结构等效化合物的相似度热力图。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.visualization import KGVisualizer, EmbeddingVisualizer, OntologyVisualizer
from semantica.ontology import OntologyGenerator
import numpy as np

graph = ContextGraph(advanced_analytics=True)
ctx   = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    graph_expansion=True,
)
ctx.store([
    "Metformin activates AMPK and reduces hepatic glucose production in Type 2 Diabetes.",
    "Dapagliflozin inhibits SGLT2 and reduces cardiovascular mortality in HFrEF.",
    "Semaglutide agonises GLP-1R and reduces HbA1c in obesity and Type 2 Diabetes.",
], extract_entities=True, extract_relationships=True)

kg_viz = KGVisualizer()
kg_viz.visualize_network(
    graph         = graph.to_dict(),
    output        = "interactive",
    file_path     = "drug_kg.html",
    node_color_by = "type",
    node_size_by  = "degree",
)

# OWL 类层次结构
ontology = OntologyGenerator(
    base_uri="https://purl.obolibrary.org/obo/DRUG_",
    min_occurrences=1,
).generate_from_graph(graph.to_dict())

ov = OntologyVisualizer()
ov.visualize_hierarchy(ontology, output="interactive", file_path="drug_hierarchy.html")
ov.visualize_structure(ontology, output="interactive", file_path="drug_ontology.html")

# 药物向量嵌入的 UMAP 投影和相似度热力图
embeddings = np.array([[0.1, 0.2, 0.3], [0.15, 0.22, 0.31], [0.8, 0.7, 0.6]])
labels     = ["Metformin", "Dapagliflozin", "Semaglutide"]

ev = EmbeddingVisualizer()
# UMAP（Uniform Manifold Approximation and Projection，统一流形近似与投影）将高维
# 向量嵌入降维到 2D，同时保留局部邻域结构
ev.visualize_2d_projection(
    embeddings, labels, method="umap",
    output="interactive", file_path="drug_embeddings.html",
)
ev.visualize_similarity_heatmap(
    embeddings, labels,
    output="interactive", file_path="drug_similarity.html",
)
```

</Tab>

<Tab title="银行 — 风险/合规">

用于模型治理的监管知识图谱：实体网络、用于文档记录的 OWL 类层次结构，以及跨季度 Basel III 更新的图谱增长曲线 — 全部为静态 PNG，用于模型治理委员会材料包。

```python
from semantica.context import ContextGraph
from semantica.change_management import TemporalVersionManager
from semantica.visualization import KGVisualizer, TemporalVisualizer, OntologyVisualizer
from semantica.ontology import OntologyGenerator

graph = ContextGraph(advanced_analytics=True)
vm    = TemporalVersionManager(storage_path="regulatory_versions.db")

for node, ntype, content in [
    ("bcbs-cre20",  "Regulation", "Basel III CRE20 — CRE capital requirements"),
    ("metric-ltv",  "Metric",     "Loan-to-Value Ratio"),
    ("metric-dscr", "Metric",     "Debt Service Coverage Ratio"),
    ("metric-pd",   "Metric",     "Probability of Default"),
    ("metric-lgd",  "Metric",     "Loss Given Default"),
]:
    graph.add_node(node, ntype, content)

graph.add_edge("bcbs-cre20", "metric-ltv",  "requires", weight=1.0)
graph.add_edge("bcbs-cre20", "metric-dscr", "requires", weight=1.0)
graph.add_edge("bcbs-cre20", "metric-pd",   "requires", weight=0.9)
graph.add_edge("bcbs-cre20", "metric-lgd",  "requires", weight=0.9)

kg_viz = KGVisualizer()
kg_viz.visualize_network(
    graph         = graph.to_dict(),
    output        = "static",
    file_path     = "regulatory/regulatory_kg.png",
    node_color_by = "type",
    node_size_by  = "degree",
)

# OWL 层次结构 — 用于文档附录的静态 PNG
ontology = OntologyGenerator(
    base_uri="https://basel.eba.eu/ontology/",
    min_occurrences=1,
).generate_from_graph(graph.to_dict())

ov = OntologyVisualizer()
ov.visualize_hierarchy(ontology, output="static", file_path="regulatory/class_hierarchy.png")

# 跨季度快照的图谱增长曲线
tv = TemporalVisualizer()
tv.visualize_metrics_evolution(
    metrics_history = {
        "node_count": [12, 18, 25, 31],
        "edge_count": [8,  15, 24, 32],
    },
    timestamps = ["2025-01-01", "2025-04-01", "2025-07-01", "2025-10-01"],
    output     = "interactive",
    file_path  = "regulatory/graph_growth.html",
)

# 快照比较 — Q2 和 Q3 Basel 更新之间有什么变化？
snap1 = vm.get_version("basel_q2_2025")
snap2 = vm.get_version("basel_q3_2025")
if snap1 and snap2:
    tv.visualize_snapshot_comparison(
        snapshots = {"basel_q2_2025": snap1, "basel_q3_2025": snap2},
        output    = "interactive",
        file_path = "regulatory/snapshot_diff.html",
    )
```

</Tab>

</Tabs>

<a id="common-pitfalls"></a>
## 常见陷阱

**渲染超大图谱。**试图可视化数千节点的图谱会使浏览器崩溃，并产生无法解读的"毛球"。始终在可视化之前将大型图谱筛选为有意义的子集。

**将视觉上的接近视为关系证明。**在可视化中看起来接近的节点在图结构中未必紧密相关。视觉布局算法优化的是可读性，而非语义准确性。

**可视化重复/未清洗的数据。**重复实体、不一致的命名和数据质量问题在可视化中会被放大。在为利益相关者创建视觉呈现之前，请先清洗你的图谱数据。

**用大量文本字段过载工具提示。**悬停在节点上不应显示整个文档内容。在悬停工具提示中只包含必要的元数据 — 实体类型、名称和关键属性。

**在图谱清理之前运行可视化。**可视化直接反映数据质量问题。命名不一致的实体、重复节点和缺失的关系会造成令人困惑且具有误导性的视觉表示。

<a id="output-modes"></a>
## 输出模式

每个可视化器方法都接受相同的两种输出模式：

| `output` 值 | 格式 | 适用于 |
| :------------- | :----- | :------- |
| `"interactive"` | 自包含 HTML（Plotly / pyvis） | Jupyter 笔记本、分析师门户、邮件附件 |
| `"static"` | PNG / SVG（Matplotlib） | PDF 报告、幻灯片演示、监管提交 |

要取回图形对象而非写入磁盘，请省略 `file_path`：

```python
fig = viz.visualize_network(graph.to_dict(), output="interactive")
fig.show()              # 在 Jupyter 中内联渲染
fig.write_html("out.html")   # 手动导出
```

<a id="related-guides"></a>
## 相关指南

- [上下文图谱](context-graphs.zh-CN.md) — `graph.to_dict()` 是 `KGVisualizer` 的主要输入
- [本体管理](ontology.zh-CN.md) — `OntologyVisualizer` 渲染由 `OntologyGenerator` 生成的本体
- [变更管理](change-management.zh-CN.md) — `TemporalVersionManager` 的快照可供给 `visualize_metrics_evolution()` 和 `visualize_snapshot_comparison()`
- [图分析](graph-analytics.zh-CN.md) — 中心性分数、社区字典和连通性结果，这些是 `AnalyticsVisualizer` 的输入
- [导出与序列化](export.zh-CN.md) — 将同一图谱导出为 GraphML、GEXF 或 DOT，以用于 Gephi 和 Graphviz
