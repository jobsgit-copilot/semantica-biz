---
title: "模块"
description: "每个 Semantica 模块独立工作：按需使用。"
icon: "puzzle-piece"
---

**[English](modules.md)** · **简体中文（当前）**

<Info>
  想要快速参考？跳转到底部的[模块索引](#模块索引)。
</Info>

<Tip>
  不确定该用哪个模块？[选择合适的模块](choose-your-module.zh-CN.md)指南将 35+ 个开发者目标映射到对应模块，并附带代码示例——如果是首次了解，请从这里开始。
</Tip>

Semantica 按六个逻辑层组织为 **27 个模块**。每个模块都可以独立导入：你无需为不使用的功能付出任何开销。

<a id="architecture-overview"></a>
## 架构概览

- **输入层** — 数据摄取与准备。模块：`ingest`、`parse`、`split`、`normalize`
- **核心处理** — 智能与理解。模块：`semantic_extract`、`kg`、`ontology`、`reasoning`
- **存储** — 持久化数据存储。模块：`embeddings`、`vector_store`、`graph_store`、`triplet_store`
- **质量保障** — 数据质量与一致性。模块：`deduplication`、`conflicts`
- **上下文与记忆** — 智能体记忆与决策追踪。模块：`context`、`provenance`、`change_management`
- **输出与编排** — 导出、可视化与工作流。模块：`export`、`visualization`、`pipeline`、`explorer`


<a id="input-layer"></a>
## 输入层

<a id="ingest"></a>
### 摄取

将来自文件、网页、数据库和流的数据加载到统一的 `SourceDocument` 格式中。

```python
from semantica.ingest import FileIngestor, WebIngestor, ParquetIngestor, XMLIngestor, DatabricksIngestor

# 文件：PDF、DOCX、CSV、Excel、PPTX、JSON、HTML、归档文件
ingestor = FileIngestor()
documents = ingestor.ingest_directory("data/")

# 网页爬取
web_ingestor = WebIngestor()
page = web_ingestor.ingest_url("https://example.com")

# Parquet：单文件、分区目录、Hive 风格（v0.5.0）
parquet = ParquetIngestor()
sources = parquet.ingest("data/events.parquet")

# XML：支持 XSD/DTD 校验和命名空间处理（v0.5.0）
xml = XMLIngestor()
sources = xml.ingest("data/records/", schema_path="schema.xsd")

# 企业级湖仓/数仓 — Unity Catalog + Delta Lake，或 Snowflake 数仓
databricks = DatabricksIngestor(host="...", token="...", http_path="...")
customers   = databricks.ingest_table("customers")
```

**可用的摄取器：** `FileIngestor`、`WebIngestor`、`ParquetIngestor`、`XMLIngestor`、`RESTIngestor`、`PublicAPIIngestor`、`DBIngestor`、`DatabricksIngestor`、`SnowflakeIngestor`、`EmailIngestor`、`FeedIngestor`、`MCPIngestor`、`OntologyIngestor`、`RepoIngestor`、`StreamIngestor`、`ArrowIngestor`、`CloudStorageIngestor`

<Note>
  `DuckDBIngestor`、`ElasticIngestor`、`GDriveIngestor`、`HuggingFaceIngestor`、`MongoIngestor` 和 `PandasIngestor` 也已发布，但尚未从顶层 `semantica.ingest` 命名空间重新导出——请直接导入，例如 `from semantica.ingest.duckdb_ingestor import DuckDBIngestor`。
</Note>

<a id="parse"></a>
### 解析

从原始文档中抽取结构化文本和版面元数据。

```python
from semantica.parse import DocumentParser, DoclingParser

# 标准解析器：支持所有常见格式
parser = DocumentParser()
parsed = parser.parse_document("document.pdf")

# 高级解析器：多列 PDF、合并单元格表格、OCR
parser = DoclingParser(extract_tables=True, extract_images=True, output_format="markdown")
parsed = parser.parse("data/annual_report.pdf")
```

**可用的解析器：** `DocumentParser`、`DoclingParser`、`CodeParser`、`CSVParser`、`DocxParser`、`EmailParser`、`ExcelParser`、`HTMLParser`、`ImageParser`、`JSONParser`、`MCPParser`、`MediaParser`、`PDFParser`、`PPTXParser`、`StructuredDataParser`、`WebParser`、`XMLParser`

<a id="split"></a>
### 分块

按照语义边界感知的方式对文本进行分块，用于向量嵌入和 RAG 流水线。

```python
from semantica.split import TextSplitter

splitter = TextSplitter(method="semantic_transformer")
chunks = splitter.split(text, chunk_size=1000, chunk_overlap=200)
```

**分块策略：** `recursive`、`semantic_transformer`、`entity_aware`、`relation_aware`、`sliding_window`、`structural`

<a id="normalize"></a>
### 归一化

在语义处理之前清洗并标准化文本。

```python
from semantica.normalize import TextNormalizer, normalize_text, normalize_date

normalizer = TextNormalizer()
clean_text        = normalizer.normalize_text(text)
standardized_date = normalize_date("Jan 1st, 2020")
```

**可用的归一化器：** 文本清洗、实体规范化、日期归一化、数字归一化、编码处理、语言检测


<a id="core-processing"></a>
## 核心处理

<a id="semantic-extract"></a>
### 语义抽取

命名实体识别(NER)、关系抽取和三元组生成。

```python
from semantica.semantic_extract import NERExtractor, RelationExtractor, TripletExtractor

ner = NERExtractor(method="llm", llm_provider=llm)
entities = ner.extract("Apple Inc. was founded by Steve Jobs.")

rel = RelationExtractor(method="llm", llm_provider=llm)
relationships = rel.extract(text, entities=entities)

trip = TripletExtractor(method="llm", llm_provider=llm)
triplets = trip.extract(text)
```

**抽取方法：** `"pattern"`（无需 API 密钥）、`"ml"`（本地模型）、`"llm"`（8 种支持的提供商中的任意一种）

**其他抽取器：** `CoreferenceResolver`、`EventDetector`、`SemanticAnalyzer`、`SemanticNetworkExtractor`

<a id="knowledge-graph"></a>
### 知识图谱

图谱构建、图算法、时态模型和距离智能。

```python
from semantica.kg import GraphBuilder, GraphAnalyzer, TemporalGraphQuery, SimilarityCalculator
from datetime import datetime

# 构建
builder = GraphBuilder(merge_entities=True)
kg = builder.build(entities=entities, relationships=relationships)

# 时态图谱（v0.4.0）
query_engine = TemporalGraphQuery(enable_temporal_reasoning=True)
snapshot = query_engine.query_at_time(kg, query="", at_time=datetime(2021, 6, 15))

# 语义相似度（v0.5.0）
calc = SimilarityCalculator()
scores = calc.calculate_similarity(entity_a, entity_b)
```

**可用的图算法：** 中心性计算、社区发现、连通性分析、实体解析、链接预测、路径查找、相似度计算

<a id="ontology"></a>
### 本体

模式管理，包括 SHACL、SKOS、对齐、差异/迁移、自动生成，以及可视化的 Ontology Hub（v0.5.0）。

```python
from semantica.ontology import OntologyGenerator, SHACLGenerator

generator = OntologyGenerator()
ontology  = generator.generate_from_graph(kg)

shacl  = SHACLGenerator()
shapes = shacl.generate(ontology)
```

**组件：** `OntologyGenerator`、`SHACLGenerator`、`OntologyValidator`、`OntologyEvaluator`、`LLMOntologyGenerator`、`OWLGenerator`、`PropertyGenerator`、`DomainOntologies`、`NamespaceManager`

<a id="reasoning"></a>
### 推理

使用多种推断策略从已有知识中推导出新事实。

```python
from semantica.reasoning import Reasoner, DatalogReasoner

# 基于规则的推理
engine = Reasoner()
engine.apply_transitivity("located_in")
engine.apply_symmetry("knows")
result = engine.infer()

# Datalog：递归 Horn 子句规则（v0.4.0）
datalog = DatalogEngine()
datalog.add_rule("ancestor(X, Z) :- parent(X, Y), ancestor(Y, Z).")
results = datalog.query("ancestor(alice, ?)")
```

**引擎：** 前向链接、Rete 网络、演绎、溯因、SPARQL、Datalog：所有引擎都生成可解释的推断路径


<a id="storage"></a>
## 存储

<a id="embeddings"></a>
### 向量嵌入

生成并管理用于语义相似度的向量嵌入。

```python
from semantica.embeddings import EmbeddingGenerator

generator  = EmbeddingGenerator(model="sentence-transformers")
embeddings = generator.generate(["text1", "text2"])
similarity = generator.similarity(embeddings[0], embeddings[1])
```

**支持的模型：** Sentence-Transformers、FastEmbed、OpenAI、BGE

**组件：** `EmbeddingGenerator`、`TextEmbedder`、`VectorEmbeddingManager`、`GraphEmbeddingManager`、`PoolingStrategies`

<a id="vector-store"></a>
### 向量库

支持混合搜索的多后端向量数据库。

```python
from semantica.vector_store import VectorStore

store   = VectorStore(backend="faiss", dimension=768)
store.add_vectors(embeddings, ids)
results = store.search(query_vector, top_k=10)
```

**后端：** FAISS、Pinecone、Weaviate、Qdrant、Milvus、PgVector、内存

**搜索模式：** 语义 top-k、混合（向量 + 关键词）、元数据过滤

<a id="graph-store"></a>
### 图存储

连接图数据库以实现持久化、可查询的存储。

```python
from semantica.graph_store import GraphStore

store = GraphStore(backend="neo4j")
store.add_nodes(entities)
store.add_edges(relationships)
results = store.query("MATCH (n)-[r]->(m) RETURN n, r, m")
```

**后端：** Neo4j、FalkorDB、Apache AGE、Amazon Neptune

<a id="triplet-store"></a>
### 三元组库

基于 RDF 三元组的存储，支持 SPARQL 查询。

```python
from semantica.triplet_store import TripletStore

store = TripletStore(backend="blazegraph")
store.add_triplets(subject, predicate, obj)
results = store.sparql("SELECT ?s ?p ?o WHERE { ?s ?p ?o }")
```

**后端：** Oxigraph（嵌入式）、Blazegraph、Apache Jena、RDF4J


<a id="quality-assurance"></a>
## 质量保障

<a id="deduplication"></a>
### 去重

跨来源检测、评分并合并重复实体。

```python
from semantica.deduplication import EntityResolver

resolver = EntityResolver()
merged   = resolver.resolve(entities, strategy="semantic_v2")
```

**v2 策略**（`blocking_v2`、`hybrid_v2`、`semantic_v2`）比 v1 快最高 7 倍。

**组件：** `EntityResolver`、`DuplicateDetector`、`EntityMerger`、`SimilarityCalculator`、`ClusterBuilder`

**`DuplicateDetector` 选项：** `max_results`、`top_k_per_entity`、`min_similarity`、`sort_by`

<a id="conflicts"></a>
### 冲突

跨重叠的知识来源检测并解决事实冲突。

```python
from semantica.conflicts import ConflictDetector

detector  = ConflictDetector()
conflicts = detector.detect_conflicts(kg)
resolved  = detector.resolve(conflicts, strategy="most_recent")
```

**检测类型：** 取值冲突、类型冲突、时态冲突、逻辑冲突

**解决策略：** 优先最新、优先最可靠来源、多数投票、标记为人工审查


<a id="context-memory"></a>
## 上下文与记忆

<a id="context"></a>
### 上下文

智能体上下文图谱、决策追踪、因果链和先例搜索。

```python
from semantica.context import AgentContext, ContextGraph

context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=ContextGraph(advanced_analytics=True),
    decision_tracking=True,
)

context.store("GPT-4 outperforms GPT-3.5 on reasoning benchmarks by 40%")

decision_id = context.record_decision(
    category="model_selection",
    scenario="...",
    reasoning="...",
    outcome="...",
    confidence=0.9,
)

precedents = context.find_precedents("model selection", limit=5)
```

**组件：** `AgentContext`、`ContextGraph`、`AgentMemory`、`DecisionRecorder`、`CausalAnalyzer`、`EntityLinker`、`PolicyEngine`

<a id="provenance"></a>
### 溯源

跨所有模块的、符合 W3C PROV-O 的血缘追踪。

```python
from semantica.provenance import ProvenanceManager

manager = ProvenanceManager()
manager.track_entity("entity_1", "document.pdf", "person")
lineage = manager.get_lineage("entity_1")
```

**组件：** `ProvenanceManager`、`IntegrityChecker`、`BridgeAxiom`、`ProvenanceStorage`

<a id="change-management"></a>
### 变更管理

支持 SHA-256 校验和、差异对比和回滚的版本控制。

```python
from semantica.change_management import TemporalVersionManager

manager  = TemporalVersionManager(storage_path="versions.db")
snapshot = manager.create_snapshot(kg, "v1.0", "user@example.com", "Initial version")
diff     = manager.diff("v1.0", "v1.1")
```

**组件：** `TemporalVersionManager`、`ChangeLog`、`OntologyVersionManager`、`VersionStorage`


<a id="output-orchestration"></a>
## 输出与编排

<a id="export"></a>
### 导出

将图谱序列化为下游格式，用于分析、语义 Web 或图数据库。

```python
from semantica.export import RDFExporter, ParquetExporter, ArangoAQLExporter

# RDF 格式
RDFExporter().export(graph, file_path="graph.ttl", format="turtle")

# 分析
ParquetExporter().export(graph, file_path="output/graph.parquet")

# ArangoDB
aql = ArangoAQLExporter().export(graph)
```

**导出格式：** RDF（Turtle、JSON-LD、N-Triples、XML）、Parquet、ArangoDB AQL、CSV、OWL、Arrow、LPG、YAML、距离矩阵

<a id="visualization"></a>
### 可视化

渲染交互式和静态的知识图谱可视化。

```python
from semantica.visualization import KGVisualizer

viz = KGVisualizer()
viz.visualize_network(graph, output="html", file_path="graph.html")
```

**可视化器：** `KGVisualizer`、`OntologyVisualizer`、`EmbeddingVisualizer`、`SemanticNetworkVisualizer`、`TemporalVisualizer`、`AnalyticsVisualizer`

**布局算法：** 力导向、层次、环形

<a id="pipeline"></a>
### 流水线

支持并行 worker、重试策略和失败处理的流水线 DSL。

```python
from semantica.pipeline import Pipeline

pipeline = Pipeline()
pipeline.add_step("ingest",   FileIngestor())
pipeline.add_step("extract",  NERExtractor())
pipeline.add_step("build",    GraphBuilder())
result = pipeline.run("data/")
```

**组件：** `Pipeline`、`PipelineBuilder`、`ExecutionEngine`、`FailureHandler`、`PipelineValidator`、`ParallelismManager`、`ResourceScheduler`

<a id="explorer"></a>
### Explorer

基于 FastAPI 的知识浏览器，具备 Ontology Hub、WebSocket 进度、双向路径查找和索引搜索（在 118k 节点上 0.004ms）。

```bash
semantica-explorer --graph my_graph.json
```

**路由：** graph、ontology、provenance、decisions、analytics、SPARQL、temporal、annotations、export/import、vocabulary


<a id="utilities"></a>
## 实用工具

<a id="llm-providers"></a>
### LLM 提供商

通往所有受支持 LLM 提供商的统一接口。

```python
from semantica.llms import Groq, OpenAI, LiteLLM
import os

llm = Groq(model="llama-3.3-70b-versatile", api_key=os.getenv("GROQ_API_KEY"))
llm = OpenAI(model="gpt-4o", api_key=os.getenv("OPENAI_API_KEY"))
# 通过 LiteLLM 使用 Anthropic、Gemini、Ollama、DeepSeek：
llm = LiteLLM(model="anthropic/claude-opus-4-7", api_key=os.getenv("ANTHROPIC_API_KEY"))
```

**支持的提供商：** OpenAI、Anthropic、Google Gemini、Groq、Ollama、DeepSeek、Novita AI、LiteLLM（通过一个接口支持 20+ 模型）

<a id="mcp-server"></a>
### MCP 服务器

将 Semantica 作为 MCP stdio 服务器暴露，用于 IDE 和智能体集成。

```bash
python -m semantica.mcp_server
```

**集成：** Claude Desktop、VS Code、Cursor、Windsurf、Cline：暴露 12 个 MCP 工具

<a id="seed"></a>
### 种子数据

从经过校验的结构化来源引导构建知识图谱：固定点参考数据、受控词表和领域锚点。

```python
from semantica.seed import SeedManager

seed = SeedManager()
seed.populate(kg, dataset="companies", count=100)

# 从文件或内置数据集加载领域种子数据
seed.load_from_file("seed_data/industries.json")
seed.inject(kg)   # 合并种子节点，不重复已有实体
```

**用例：** 用已知实体锚定抽取、预填充本体类、确定性测试图谱生成。

<a id="evals"></a>
### 评估

用于衡量知识图谱质量、抽取准确率和流水线性能的评估框架。

```python
from semantica.evals import KGEvaluator, ExtractionEvaluator, PipelineEvaluator, RegressionTracker

# 知识图谱质量
report = KGEvaluator().evaluate(kg, ontology=ontology)
print(f"Completeness: {report.completeness:.2%}  Consistency: {report.consistency:.2%}")

# 抽取准确率
report = ExtractionEvaluator().evaluate_ner(predictions=extracted, gold_standard=annotated)
print(f"Precision: {report.precision:.3f}  Recall: {report.recall:.3f}  F1: {report.f1:.3f}")

# 流水线吞吐量和延迟
metrics = PipelineEvaluator().benchmark(pipeline, data="data/", bench_runs=5)
print(f"Throughput: {metrics.docs_per_second:.1f} docs/sec")

# 跨运行的回归追踪
tracker = RegressionTracker(db_path="eval_history.db")
run_id  = tracker.record_run(pipeline_version="v1.2.0", metrics=metrics)
diff    = tracker.compare(run_id, baseline_run_id="run_abc123")
```

**组件：** `KGEvaluator`、`ExtractionEvaluator`、`PipelineEvaluator`、`RegressionTracker`

<a id="core"></a>
### 核心

跨所有模块使用的基类、共享数据模型和插件注册表。

```python
from semantica.core import Semantica, PluginRegistry, ConfigManager

# 顶层编排器
sem = Semantica(config_path="config.yaml")
sem.initialize()

# 插件注册表：注册自定义组件
registry = PluginRegistry()
registry.register("my_ingestor", MyCustomIngestor)

# 配置管理
config  = ConfigManager(config_path="config.yaml")
batch   = config.get("processing.batch_size", default=32)
```

**组件：** `Semantica`、`PluginRegistry`、`ConfigManager`、`LifecycleManager`、`HealthMonitor`、`Config`

<a id="utils"></a>
### 工具

用于 ID 生成、日期解析、校验和日志记录的共享工具。

```python
from semantica.utils import helpers, validators, logging
```

**组件：** `helpers`、`validators`、`constants`、`types`、`exceptions`、`logging`、`ProgressTracker`


<a id="common-module-chains"></a>
## 常见模块组合

<Tabs>
  <Tab title="文档 → 知识图谱">
    从任意来源加载文档，并将其转换为可查询的知识图谱。

    **流水线：** `Ingest` → `Parse` → `Normalize` → `Semantic Extract` → `GraphBuilder` → `KG`

```python
from semantica.ingest import FileIngestor
from semantica.parse import DocumentParser
from semantica.semantic_extract import NERExtractor, RelationExtractor
from semantica.kg import GraphBuilder

sources       = FileIngestor().ingest("data/")
parsed        = DocumentParser().parse(sources[0])
entities      = NERExtractor(method="llm", llm_provider=llm).extract(parsed)
relationships = RelationExtractor(method="llm", llm_provider=llm).extract(parsed, entities=entities)
graph         = GraphBuilder(merge_entities=True).build(
                    entities=entities, relationships=relationships
                )
```

    **最适用于：** 研究流水线、企业数据抽取、文档智能
  </Tab>

  <Tab title="GraphRAG">
    将每个 LLM 响应建立在知识图谱之上：带有来源归属的结构化检索。

    **流水线：** `KG` + `VectorStore` → `AgentContext` → GraphRAG 查询 → 有依据的回答

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=ContextGraph(advanced_analytics=True),
)
context.load_graph("company_kg.json")

result = context.query(
    "What companies did Apple alumni found?",
    mode="graphrag",
    reasoning=True,
)
for claim in result.claims:
    print(f"{claim.text}  →  {claim.source_node}")
```

    **最适用于：** 问答系统、带来源归属的 RAG、研究助手
  </Tab>

  <Tab title="AI 智能体">
    为你的智能体提供持久化记忆、决策追踪和策略执行。

    **流水线：** `AgentContext` → 决策记录 → 先例搜索 → 策略检查 → 因果分析

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=ContextGraph(advanced_analytics=True),
    decision_tracking=True,
)
context.store("GPT-4 outperforms GPT-3.5 on reasoning by 40%")

decision_id = context.record_decision(
    category="model_selection",
    scenario="Choose LLM for production",
    reasoning="Benchmark advantage justifies cost",
    outcome="selected_gpt4",
    confidence=0.91,
)
precedents = context.find_precedents("model selection", limit=5)
```

    **最适用于：** 自主智能体、AI 副驾驶、决策支持系统
  </Tab>

  <Tab title="合规流水线">
    从原始数据到最终推断的完整溯源：W3C PROV-O、SHA-256 校验和、审计追踪。

    **流水线：** `Ingest` → `Parse` → `Extract` → `KG` → `Provenance` → `ChangeManagement` → `Export`

```python
from semantica.ingest import FileIngestor
from semantica.semantic_extract import NERExtractor
from semantica.kg import GraphBuilder
from semantica.provenance import ProvenanceManager
from semantica.export import RDFExporter

sources  = FileIngestor().ingest("records/")
entities = NERExtractor(method="llm", llm_provider=llm).extract(sources)
graph    = GraphBuilder(merge_entities=True).build(entities=entities, relationships=[])
prov     = ProvenanceManager()
lineage  = prov.get_entity_lineage("entity_id")

RDFExporter(include_provenance=True).export(graph, file_path="audit.ttl", format="turtle")
```

    **最适用于：** HIPAA、SOX、GDPR、FDA 21 CFR Part 11 部署场景
  </Tab>

  <Tab title="网页抓取 → 图谱">
    抓取网站、归一化文本，并直接从网页中抽取知识。

    **流水线：** `WebIngestor` → `Normalize` → `Semantic Extract` → `GraphStore`

```python
from semantica.ingest import WebIngestor
from semantica.normalize import TextNormalizer
from semantica.semantic_extract import NERExtractor, RelationExtractor
from semantica.graph_store import Neo4jStore

pages      = WebIngestor(max_depth=2).ingest("https://example.com")
normalizer = TextNormalizer()
store      = Neo4jStore(uri="bolt://localhost:7687", user="neo4j", password="password")

for page in pages:
    text          = normalizer.normalize_text(page.text)
    entities      = NERExtractor().extract(text)
    relationships = RelationExtractor().extract(text, entities=entities)
    store.add_nodes(entities)
    store.add_edges(relationships)
```

    **最适用于：** 竞争情报、新闻监测、研究聚合
  </Tab>

  <Tab title="时态分析">
    追踪事实如何随时间变化：时间点查询、快照和版本管理。

    **流水线：** `KG (Temporal)` → `TemporalGraphQuery` → `VersionManager` → `ChangeManagement`

```python
from semantica.kg import GraphBuilder, TemporalGraphQuery, TemporalVersionManager

builder = GraphBuilder()
kg      = builder.build(sources=[{
    "entities": [{"id": "alice", "type": "Person"}],
    "relationships": [{"source": "alice", "target": "acme", "type": "ceo_of",
                       "valid_from": "2020-01-01", "valid_until": "2023-06-01"}]
}])

query         = TemporalGraphQuery()
snapshot_2021 = query.reconstruct_at_time(kg, "2021-06-15")

versioner = TemporalVersionManager()
versioner.create_snapshot(kg, "2024-Q1", author="user@example.com", description="Q1 snapshot")
```

    **最适用于：** 金融历史、监管时间线、组织变更追踪
  </Tab>
</Tabs>


<a id="module-index"></a>
## 模块索引

| 模块 | 用途 | 关键类 |
| :------ | :------- | :----------- |
| [ingest](reference/ingest.zh-CN.md) | 数据摄取 | `FileIngestor`、`WebIngestor`、`ParquetIngestor`、`XMLIngestor` |
| [parse](reference/parse.zh-CN.md) | 文档解析 | `DocumentParser`、`DoclingParser` |
| [split](reference/split.zh-CN.md) | 文本分块 | `TextSplitter` |
| [normalize](reference/normalize.zh-CN.md) | 数据清洗 | `TextNormalizer`、`EntityNormalizer`、`LanguageDetector` |
| [semantic_extract](reference/semantic_extract.zh-CN.md) | 命名实体识别(NER)与关系抽取 | `NERExtractor`、`RelationExtractor`、`TripletExtractor`、`SemanticAnalyzer`、`SemanticNetworkExtractor`、`ExtractionValidator` |
| [kg](reference/kg.zh-CN.md) | 图谱构建 | `GraphBuilder`、`TemporalGraphQuery`、`SimilarityCalculator` |
| [ontology](reference/ontology.zh-CN.md) | 模式管理 | `OntologyGenerator`、`SHACLGenerator` |
| [reasoning](reference/reasoning.zh-CN.md) | 逻辑推断 | `Reasoner`、`DatalogReasoner` |
| [embeddings](reference/embeddings.zh-CN.md) | 向量嵌入 | `EmbeddingGenerator` |
| [vector_store](reference/vector_store.zh-CN.md) | 向量数据库 | `VectorStore` |
| [graph_store](reference/graph_store.zh-CN.md) | 图数据库 | `GraphStore` |
| [triplet_store](reference/triplet_store.zh-CN.md) | RDF 三元组库 | `TripletStore` |
| [deduplication](reference/deduplication.zh-CN.md) | 实体解析 | `EntityResolver`、`DuplicateDetector`、`ClusterBuilder`、`MergeStrategyManager` |
| [conflicts](reference/conflicts.zh-CN.md) | 冲突解决 | `ConflictDetector` |
| [context](reference/context.zh-CN.md) | 智能体上下文与决策 | `AgentContext`、`ContextGraph` |
| [provenance](reference/provenance.zh-CN.md) | W3C PROV-O 血缘 | `ProvenanceManager` |
| [change_management](reference/change_management.zh-CN.md) | 版本控制 | `TemporalVersionManager` |
| [export](reference/export.zh-CN.md) | 数据导出 | `RDFExporter`、`ParquetExporter` |
| [visualization](reference/visualization.zh-CN.md) | 图谱可视化 | `KGVisualizer` |
| [pipeline](reference/pipeline.zh-CN.md) | 工作流编排 | `Pipeline`、`PipelineBuilder` |
| [explorer](reference/explorer.zh-CN.md) | 知识浏览器界面 | `semantica-explorer --graph <file>` |
| [llms](reference/llms.zh-CN.md) | LLM 提供商 | `Groq`、`OpenAI`、`create_provider` |
| [mcp_server](reference/mcp_server.zh-CN.md) | MCP stdio 服务器 | `python -m semantica.mcp_server` |
| [seed](reference/seed.zh-CN.md) | 从结构化源引导构建知识图谱 | `SeedManager` |
| [evals](reference/evals.zh-CN.md) | 质量评估 | `KGEvaluator`、`ExtractionEvaluator`、`PipelineEvaluator`、`RegressionTracker` |
| [core](reference/core.zh-CN.md) | 基类与注册表 | `Semantica`、`ConfigManager`、`PluginRegistry`、`LifecycleManager` |
| [utils](reference/utils.zh-CN.md) | 共享工具 | `helpers`、`validators` |

- [入门指南](getting-started.zh-CN.md) — 5 分钟构建你的第一个知识图谱。
- [Cookbook](cookbook.zh-CN.md) — 40+ 个领域的 notebook，附带真实场景示例。
- [API 参考](reference/context.zh-CN.md) — 完整的技术文档。
