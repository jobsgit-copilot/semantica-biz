---
title: "导出与序列化"
description: "将知识图谱导出为 RDF（Turtle、JSON-LD、N-Triples）、GraphML、Cypher（Neo4j）、ArangoDB AQL、CSV、Parquet、OWL 等格式。"
---

**[English](export.md)** · **简体中文（当前）**

<a id="what-is-export"></a>
## 什么是导出？

导出将 Semantica 图谱数据转换为外部工具和系统使用的格式。与将数据保留在 Semantica 内部的持久化机制不同，导出专门为与外部消费者的互操作性而设计。

**导出与内部持久化：**
- **`AgentContext.store()`** 和图谱持久化将数据保留在 Semantica 内部，用于持续处理、检索和推理
- **导出函数** 将图谱数据序列化为外部系统可直接使用的标准化格式

导出支持与分析平台、图数据库、RDF 三元组库、语义网系统、数据仓库、商业智能工具以及需要以原生格式访问你知识图谱数据的下游消费者进行集成。

<a id="why-use-export"></a>
## 为什么使用导出？

**一次构建，多次导出。** 通过 Semantica 的抽取和推理工作流创建知识图谱，然后将相同的图谱数据导出为多种格式供不同消费者使用，无需重建或重新处理。

**与现有生态系统的互操作性。** 将 Semantica 图谱连接到你组织中已有的工具和工作流，从 Neo4j 图数据库到 Gephi 可视化再到 pandas 数据分析流水线。

**分析和报告工作流。** 将图谱数据输入到商业智能工具、统计分析平台和机器学习流水线中，这些工具需要 CSV、Parquet 或 RDF 等特定数据格式。

**图数据库迁移和部署。** 将图谱从 Semantica 的内存表示迁移到 Neo4j、ArangoDB 或三元组库等生产图数据库，以实现可扩展的查询性能。

**RDF 和语义网集成。** 导出到语义网标准（Turtle、JSON-LD、N-Triples），以与本体工具、SPARQL 端点和语义推理系统集成。

**数据湖和数据仓库集成。** 导出为 Parquet 等列式格式，以与 DuckDB、Apache Spark 和云数据仓库等现代数据栈工具集成。

**合规和归档工作流。** 生成标准化导出，用于监管提交、长期归档和要求数据格式的审计轨迹需求。

<a id="when-to-use-when-not-to-use"></a>
## 何时使用 / 何时不使用

**在以下情况使用导出：**
- 将 Semantica 图谱与外部系统和工具集成
- 与使用不同技术栈的团队共享图谱数据
- 构建在下游处理中使用图谱数据的分析流水线
- 使用需要语义网标准的 RDF 和本体工作流
- 创建报告、可视化和商业智能仪表板
- 将图谱迁移到生产数据库以实现可扩展的查询性能
- 满足特定数据格式提交的合规要求

**在以下情况不要使用导出：**
- 你只是想保存和重新加载 Semantica 状态——请使用内置持久化机制
- 智能体持久化和记忆连续性是你的主要目标
- 内部检索、推理和图谱操作足以满足你的用例
- 导出会为完全在 Semantica 内运行的工作流增加不必要的复杂性
- 你需要实时访问不断演变的图谱数据——导出创建的是静态快照

**在以下情况考虑使用内部持久化：**
- 你的工作流涉及在 Semantica 内迭代构建、查询和推理图谱
- 你需要维护智能体记忆、对话历史和决策追踪
- 图谱数据将在 Semantica 工作流中继续被处理和丰富

`export_rdf`、`export_graph`、`export_lpg` 及相关函数在单次调用中将 `ContextGraph` 序列化为十种格式中的任意一种，忠实地保留节点类型、边权重和元数据。当下游消费者——三元组库、图数据库、可视化工具、ML 流水线或电子表格审计员——各自期望从同一个内存图谱获取不同格式时使用它们。

<Info>
  所有导出函数都接受 `graph.to_dict()` 作为第一个参数——与 `ContextGraph.to_dict()` 生成的字典相同。构建一次图谱，即可将其导出为所需的所有格式而无需重新序列化。请注意，`graph.to_dict()` 会将整个图谱物化到内存中，因此非常大的图谱可能需要额外的内存规划。
</Info>

<a id="building-the-graph-to-export"></a>
## 构建要导出的图谱

在首次导出之前，先填充一个图谱。以下每个示例都从相同的设置开始：

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph()
ctx   = AgentContext(vector_store=vs, knowledge_graph=graph, graph_expansion=True)

ctx.store(
    [
        "APT29 exploits CVE-2024-3400 in PAN-OS to target NATO defense contractors.",
        "CVE-2024-3400 is a critical remote code execution vulnerability in GlobalProtect.",
        "HAMMERTOSS is APT29's C2 backdoor using Twitter and GitHub as covert channels.",
    ],
    extract_entities=True,
    extract_relationships=True,
)

graph_data = graph.to_dict()   # 单个字典，在以下所有导出中复用
```

<a id="rdf-formats-for-triple-stores-and-semantic-reasoners"></a>
## RDF 格式——用于三元组库和语义推理器

**RDF（Resource Description Framework）** 是语义网的基础数据模型，将信息表示为主-谓-宾三元组。RDF 格式对于与语义网技术、本体工具和需要形式化知识表示的系统集成至关重要。

当你的消费者是三元组库（GraphDB、Stardog、Apache Jena）或 OWL 推理器（HermiT、Pellet）时，你需要 RDF。Semantica 通过单次 `export_rdf` 调用导出所有五种标准 RDF 序列化格式。

```python
from semantica.export import export_rdf

# Turtle——人类可读的默认格式；最适合审查和 Git 存储
export_rdf(graph_data, "threat_graph.ttl", format="turtle")

# N-Triples——每行一个三元组，无缩进；向任何三元组库批量加载最快
export_rdf(graph_data, "threat_graph.nt", format="ntriples")

# JSON-LD——嵌入 @context；当下游消费者原生使用 JSON 时最佳
export_rdf(graph_data, "threat_graph.jsonld", format="jsonld")

# RDF/XML——最大的遗留互操作性；某些较旧的 OWL 工具所需
export_rdf(graph_data, "threat_graph.rdf", format="rdfxml")
```

选择哪种格式取决于你的消费者。Turtle 非常适合人工审查和提交到 Git——它紧凑且可读。N-Triples 是批量加载到 SPARQL 端点的最快选择，因为解析器可以逐行流式处理而无需缓冲整个文件。当下游系统已经使用 JSON 并且你希望语义上下文嵌入在同一载荷中时，JSON-LD 是正确的选择。RDF/XML 的存在是为了与早于其他格式的旧工具兼容。

<a id="graph-formats-for-gephi-maltego-and-network-analysis"></a>
## 图谱格式——用于 Gephi、Maltego 和网络分析

**标签属性图（LPG）** 格式表示具有类型化节点和边的网络，这些节点和边携带属性和元数据。这些格式针对专注于探索关系和结构模式的图谱可视化工具和网络分析平台进行了优化。

GraphML、GEXF 和 DOT 是图谱分析和可视化工具的原生格式。它们保留节点属性、边权重和类型标签，因此你在 Semantica 中构建的图谱可在 Gephi 或 NetworkX 中立即渲染并带有完整属性数据。

```python
from semantica.export import export_graph

# GraphML——最广泛的工具支持：Gephi、yEd、NetworkX、Maltego
export_graph(graph_data, "threat_graph.graphml", format="graphml")

# GEXF——在 Gephi 中支持更丰富的属性；更适合大型属性图谱
export_graph(graph_data, "threat_graph.gexf", format="gexf")

# DOT (Graphviz)——用于报告中图表的自动布局和渲染
export_graph(graph_data, "threat_graph.dot", format="dot")
```

如果你使用 Gephi 进行分析师简报，GEXF 格式值得了解——它携带 GraphML 无法表达的动态属性和时间数据。当你需要 Graphviz 自动布局图谱以嵌入 PDF 报告或文档站点时，DOT 是正确的选择。

<a id="neo4j-cypher-for-graph-pattern-threat-hunting"></a>
## Neo4j Cypher——用于图模式威胁狩猎

**Cypher** 是 Neo4j 的声明式图谱查询语言，使用模式匹配来查找和操作图谱数据。Cypher 导出使团队能够使用 Neo4j 优化的查询引擎运行复杂的图谱查询、模式检测和图谱分析。

当 SOC 团队想要对图谱运行 Cypher 查询——查找共享基础设施的威胁行为者，或追踪多跳攻击路径——你导出为 Cypher 并通过单条命令将结果加载到 Neo4j Desktop 或 Memgraph 中。

```python
from semantica.export import export_lpg

export_lpg(graph_data, "threat_graph.cypher", method="cypher")
```

输出文件包含可直接运行的 `CREATE` 和 `MATCH` 语句：

```text
CREATE (:ThreatActor {id: "apt29", name: "APT29", nation_state: "RU"})
CREATE (:Vulnerability {id: "cve-2024-3400", cvss_score: 10.0})
MATCH (a {id: "apt29"}), (b {id: "cve-2024-3400"}) CREATE (a)-[:EXPLOITS {confidence: 0.97}]->(b)
```

使用 `cypher-shell < threat_graph.cypher` 加载到 Neo4j，或将文件拖入 Neo4j Desktop 的导入向导。从那时起，SOC 团队可以编写 Cypher 查询而无需接触 Python。

<a id="arangodb-aql-for-multi-model-queries"></a>
## ArangoDB AQL——用于多模型查询

ArangoDB 在单一查询语言中结合了图遍历、文档查询和全文搜索。当你的合规团队需要将图谱与结构化监管文档进行连接时，ArangoDB 是正确的后端。

```python
from semantica.export import export_arango

export_arango(
    graph_data,
    "regulatory.aql",
    vertex_collection         = "regulatory_nodes",
    edge_collection           = "regulatory_edges",
    include_collection_creation = True,    # 生成 CREATE COLLECTION 语句
    batch_size                = 200,       # INSERT 语句分批以避免内存峰值
)
```

`include_collection_creation=True` 标志意味着 AQL 文件是自包含的——它在插入数据之前创建集合，因此你可以在全新的 ArangoDB 实例上运行它而无需任何预先设置。

<a id="csv-for-spreadsheet-audits-and-statistical-analysis"></a>
## CSV——用于电子表格审计和统计分析

**CSV（逗号分隔值）** 是一种简单的表格格式，受到电子表格应用、统计工具和数据分析平台的普遍支持。CSV 导出将图谱数据扁平化为行列形式，适用于主要处理表格数据的团队。

合规团队在 Excel 中工作。数据科学团队在 pandas 中工作。他们都需要 CSV。`export_csv` 将图谱写入扁平行——当你传入基础路径时，实体和关系作为单独的文件。

```python
from semantica.export import export_csv

# 单个 CSV——节点和边通过 "record_type" 列交错
export_csv(graph_data, "threat_graph.csv")

# 拆分 CSV——写入 threat_graph_entities.csv 和 threat_graph_relationships.csv
export_csv(
    {"entities": graph_data.get("nodes", []),
     "relationships": graph_data.get("edges", [])},
    "threat_graph",
)
```

拆分形式对下游工具更有用：实体 CSV 提供实体类型的数据透视表；关系 CSV 在 pandas 或 R 中提供网络分析。

<a id="parquet-for-data-lakes-and-ml-pipelines"></a>
## Parquet——用于数据湖和 ML 流水线

**Parquet** 是一种针对分析工作负载优化的列式存储格式，提供高效的压缩和快速的查询性能。Parquet 文件与现代数据栈工具和机器学习框架无缝集成。

当数据科学团队在 DuckDB、Spark 或湖仓中对图谱属性运行特征工程时，Parquet 是他们想要的格式。它是列式的、压缩的，可被每个主要 ML 框架读取。

```python
from semantica.export import export_parquet

export_parquet(graph_data, "threat_graph_entities.parquet")
```

一旦转为 Parquet，图谱实体就成为一个 DataFrame，可以与遥测数据连接、用外部特征丰富，并输入分类模型——数据科学方面无需任何自定义序列化代码。

<a id="owl-for-ontology-based-reasoning"></a>
## OWL——用于基于本体的推理

**OWL（Web Ontology Language）** 是一种语义网标准，用于表示具有类、属性和逻辑约束的丰富本体。OWL 支持在形式化知识模型上进行自动推理、一致性检查和推断。

**OntologyGenerator** 通过分析实体类型、关系和模式来生成类层次结构、属性定义和逻辑约束，从而从图谱数据创建形式化本体。这支持模式验证、自动推理以及与语义网工具的集成。

当你使用 `OntologyGenerator` 从图谱生成了 OWL 本体后，你可以将其导出用于 Protégé、HermiT 推理或监管提交。

```python
from semantica.export import export_owl
from semantica.ontology import OntologyGenerator

ontology = OntologyGenerator(base_uri="https://example.org/cti/") \
               .generate_from_graph(graph_data)

export_owl(ontology, "cti_ontology.owl", format="owl-xml")
```

<a id="common-pitfalls"></a>
## 常见陷阱

**混淆导出与持久化。** 导出创建用于互操作性的外部快照，而持久化维护 Semantica 的内部状态。当你需要保存和重新加载智能体记忆或继续基于图谱的工作流时，不要使用导出——请使用内置持久化机制。

**在图谱更改后导出过时的图谱数据。** 始终在最终图谱修改之后调用 `graph.to_dict()`。如果你在工作流早期存储了 `graph_data` 然后修改了图谱，导出将反映过时状态而非最新更改。

**重新运行昂贵的抽取而非复用现有图谱数据。** 通过实体抽取和关系推断构建一次图谱，然后使用相同的 `graph_data` 字典导出为多种格式。不要为每种导出格式重建图谱。

**在 CSV 就足够时选择过于复杂的格式。** 如果下游消费者处理表格数据且不需要图谱结构保留，CSV 比 RDF 或 GraphML 格式更简单、更快速且更普遍受支持。

**假设溯源和历史自动出现在导出中。** 标准导出格式捕获当前图谱状态但不包含溯源链、版本历史或审计轨迹。如果你需要完整的谱系信息，请使用专用的溯源导出机制。

**忽视下游模式要求。** 不同系统期望不同的标识符格式、属性模式和关系表示。在部署到生产工作流之前，验证你导出的数据是否与消费系统的期望匹配。

**在没有内存规划的情况下导出极大的图谱。** `graph.to_dict()` 操作将整个图谱物化到内存中。对于非常大的图谱，请监控内存使用量，并考虑在资源受限的环境中使用分块或流式方法。

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防——CTI/威胁">

CTI 团队需要同时在四个位置使用相同的威胁图谱：用于跨团队查询的 SPARQL 端点、用于分析师简报的 Gephi、用于图模式威胁狩猎的 Neo4j，以及用于 SIEM 摄取流水线的 JSON-LD 订阅源。四次导出，一个图谱字典。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.ingest import ingest_file
from semantica.export import export_rdf, export_graph, export_lpg
import os

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph()
ctx   = AgentContext(vector_store=vs, knowledge_graph=graph, graph_expansion=True)

apt_report = ingest_file("apt29_2024_campaign.pdf", method="file")
ctx.store(apt_report.text, extract_entities=True, extract_relationships=True)

graph_data = graph.to_dict()
os.makedirs("./exports/", exist_ok=True)

# 1. Turtle → SPARQL 端点（Apache Jena、Stardog、GraphDB）
export_rdf(graph_data, "./exports/threat_graph.ttl", format="turtle")

# 2. JSON-LD → SIEM 摄取（Splunk、Elastic）——JSON 原生流水线
export_rdf(graph_data, "./exports/threat_graph.jsonld", format="jsonld")

# 3. GraphML → Gephi 用于分析师简报可视化
export_graph(graph_data, "./exports/threat_graph.graphml", format="graphml")

# 4. Cypher → Neo4j 用于图模式威胁狩猎
export_lpg(graph_data, "./exports/threat_graph.cypher", method="cypher")

print("Threat graph exported to 4 formats.")
```

</Tab>

<Tab title="安全——SOC/事件">

在活跃事件期间，SOC 需要将图谱放入 Neo4j 进行狩猎、放入 GEXF 进行 Gephi 时间线可视化、放入 GraphML 导入 Maltego。三者都来自同一个内存图谱。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.export import export_lpg, export_graph
import os

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph()
ctx   = AgentContext(
    vector_store=vs,
    knowledge_graph=graph,
    graph_expansion=True,
    decision_tracking=True,
)

incidents = [
    "Host ws-finance-03 (10.10.1.5): scheduled task created via wmiprvse.exe — T1053.005",
    "User jsmith logged in from anomalous IP 185.220.101.7 (Tor exit node)",
    "EDR alert on dc01: LSASS memory access by procdump.exe — T1003.001",
]
ctx.store(incidents, extract_entities=True, extract_relationships=True)

graph_data = graph.to_dict()
os.makedirs("./soc_exports/", exist_ok=True)

# Cypher → Neo4j 用于基于 Cypher 的威胁狩猎
export_lpg(graph_data, "./soc_exports/incident_graph.cypher", method="cypher")

# GEXF → Gephi 用于时间线可视化
export_graph(graph_data, "./soc_exports/incident_graph.gexf", format="gexf")

# GraphML → Maltego 用于链接分析
export_graph(graph_data, "./soc_exports/incident_graph.graphml", format="graphml")

print("SOC graph exported — load incident_graph.cypher into Neo4j Desktop.")
```

</Tab>

<Tab title="生命科学——临床/制药">

临床试验数据需要到达 SPARQL 端点用于跨试验联合查询、CSV 用于 R 中的统计分析，以及 Turtle 文件用于监管提交。图谱编码了从试验方案中抽取的化合物-靶点-疾病关系。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.ingest import DBIngestor
from semantica.export import export_rdf, export_csv

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph(advanced_analytics=True)
ctx   = AgentContext(
    vector_store=vs,
    knowledge_graph=graph,
    graph_expansion=True,
    retention_days=None,
)

db = DBIngestor()
trial_rows = db.execute_query(
    "postgresql://readonly@clindb:5432/trials",
    "SELECT compound, target_protein, disease, mechanism FROM trial_protocols",
)
trial_texts = [
    "{} targets {} in {} via {}.".format(
        r["compound"], r["target_protein"], r["disease"], r["mechanism"]
    )
    for r in trial_rows
]
ctx.store(trial_texts, extract_entities=True, extract_relationships=True)

graph_data = graph.to_dict()

# N-Triples 用于快速批量加载到 GraphDB 或 Stardog
export_rdf(graph_data, "./exports/clinical_graph.nt",  format="ntriples")

# Turtle 用于人工审查和监管档案附件
export_rdf(graph_data, "./exports/clinical_graph.ttl", format="turtle")

# 拆分 CSV 用于 R / SAS 中的统计分析
export_csv(
    {"entities": graph_data.get("nodes", []),
     "relationships": graph_data.get("edges", [])},
    "./exports/clinical_graph",
)

print("Clinical graph exported — ready for SPARQL endpoint and statistical review.")
```

</Tab>

<Tab title="银行——风险/合规">

一个监管知识图谱必须到达 ArangoDB 用于多模型合规查询、RDF/XML 用于长期监管归档，以及 JSON-LD 用于合规仪表板 API。巴塞尔 III 法规、风险参数及其关系都作为图谱节点被捕获。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.ingest import ingest_file
from semantica.export import export_arango, export_rdf
import os

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph()
ctx   = AgentContext(
    vector_store=vs,
    knowledge_graph=graph,
    graph_expansion=True,
    retention_days=2555,    # 7 年监管留存
)

regs = [
    ingest_file("basel3_cre20.pdf",       method="file"),
    ingest_file("sr_11_7_model_risk.pdf", method="file"),
    ingest_file("bcbs239.pdf",            method="file"),
]
ctx.store(
    [r.text for r in regs],
    extract_entities=True,
    extract_relationships=True,
)

graph_data = graph.to_dict()
os.makedirs("./compliance_exports/", exist_ok=True)

# ArangoDB AQL 用于多模型监管查询（图谱 + 文档连接）
export_arango(
    graph_data,
    "./compliance_exports/regulatory.aql",
    vertex_collection           = "regulatory_nodes",
    edge_collection             = "regulatory_edges",
    include_collection_creation = True,
    batch_size                  = 200,
)

# RDF/XML 用于长期监管归档（最大遗留互操作性）
export_rdf(graph_data, "./compliance_exports/regulatory_audit.rdf", format="rdfxml")

# JSON-LD 用于合规仪表板 REST API
export_rdf(graph_data, "./compliance_exports/regulatory.jsonld", format="jsonld")

print("Compliance graph exported in 3 formats.")
```

</Tab>

</Tabs>

<a id="choosing-the-right-format"></a>
## 选择正确的格式

格式决策通常归结为谁在消费输出以及他们已经在使用什么工具。

如果你的消费者使用 SPARQL 或使用三元组库，请选择 Turtle（人工审查）、N-Triples（批量加载）或 JSON-LD（JSON 原生流水线）。如果他们使用属性图数据库，Cypher 用于 Neo4j 或 Memgraph；当他们还需要在同一系统中进行文档和搜索查询时，AQL 用于 ArangoDB。如果他们使用图谱可视化工具，GraphML 是具有最广泛工具支持的最安全默认选择，GEXF 专门在 Gephi 中提供更丰富的属性处理，当你需要 Graphviz 自动渲染静态图表时 DOT 是正确的选择。如果他们在电子表格或统计工具中工作，CSV 是阻力最小的路径。如果他们在 DuckDB、Spark 或湖仓中运行 ML 流水线，Parquet 是他们想要的格式。

对于语义推理和本体工作，OWL/XML 是首选格式——它是唯一为 Protégé 和 HermiT 保留完整类层次结构的输出。

<a id="related-guides"></a>
## 相关指南

- [上下文图谱](context-graphs.zh-CN.md) — `ContextGraph` 对象的 `to_dict()` 为所有导出提供数据
- [本体管理](ontology.zh-CN.md) — 导出从图谱生成的 OWL 本体
- [推理与规则](reasoning.zh-CN.md) — 推理结果可以导出为 RDF 三元组
- [变更管理](change-management.zh-CN.md) — 在导出之前对图谱进行快照，以证明导出来自已验证的状态
- [流水线](pipeline.zh-CN.md) — 在单个 `PipelineBuilder` 中链接摄取、抽取和导出
