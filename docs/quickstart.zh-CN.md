---
title: "快速上手"
description: "5 分钟构建你的第一个知识图谱，无需任何配置。"
icon: "rocket"
---

**[English](quickstart.md)** · **简体中文（当前）**

<Info>
  **v0.5.0** — Ontology Hub、距离智能、Parquet 与 XML 摄取、12 项安全修复。<a href="https://github.com/semantica-agi/semantica/releases" style={{color:"#10B981",fontWeight:600,textDecoration:"none"}}>查看新特性 →</a>
</Info>

本指南将带你完整走一遍构建首个知识图谱的端到端流水线。安装完成后请从这里开始。LLM API key 为可选项：基于模式的抽取开箱即用。


## 安装

<CodeGroup>

```bash pip（推荐）
pip install semantica
```

```bash 安装全部附加组件
pip install semantica[all]
```

```bash 从源码安装
git clone https://github.com/semantica-agi/semantica.git
cd semantica
pip install -e ".[dev]"
```

</CodeGroup>

验证：

```bash
python -c "import semantica; print(semantica.__version__)"
# 0.5.0
```


## 完整流水线

<img src="/assets/img/diagrams/pipeline-flow.svg" alt="Semantica 端到端流水线：摄取 → 解析 → 归一化 → 抽取 → 构建 KG → 问答 → 存储 → 交付" style={{ width: '100%', borderRadius: '10px', margin: '0 0 24px' }} />

<Steps>

<Step title="摄取">

从文件、目录、URL 或数据库加载文档。

<CodeGroup>

```python 文件
from semantica.ingest import FileIngestor

ingestor = FileIngestor()
sources  = ingestor.ingest("data/report.pdf")
# 还支持：.docx, .html, .json, .csv, .xlsx, .pptx, .parquet, .xml
```

```python 网页
from semantica.ingest import WebIngestor

ingestor = WebIngestor(max_depth=2)
sources  = ingestor.ingest("https://example.com/article")
```

```python Parquet / XML (v0.5.0)
from semantica.ingest import ParquetIngestor, XMLIngestor

# 单个文件或 Hive 分区目录
sources = ParquetIngestor().ingest("data/events.parquet")

# 带有 XSD schema 校验的 XML
sources = XMLIngestor(validate_xsd="schema.xsd").ingest("data/records/")
```

</CodeGroup>

</Step>

<Step title="解析">

从原始文档中抽取结构化文本与版面信息。

```python
from semantica.parse import DocumentParser

parser = DocumentParser()
parsed = parser.parse(sources[0])

print(parsed.text[:200])  # 提取的文本
print(parsed.metadata)    # 标题、作者、日期、来源
```

<Tip>
  对于包含表格、图表或多栏版面的 PDF，请使用 `DoclingParser`：它会执行高级版面分析，并在文本之外返回结构化的表格数据。
</Tip>

```python
from semantica.parse import DoclingParser

parser = DoclingParser()
parsed = parser.parse(sources[0])
print(parsed.tables)  # 结构化表格对象
```

</Step>

<Step title="抽取实体与关系">

识别命名实体并抽取它们之间带类型的关系。

<CodeGroup>

```python 基于模式（快速，无需 API key）
from semantica.semantic_extract import NERExtractor, RelationExtractor

ner      = NERExtractor(method="pattern")
entities = ner.extract(parsed)
# 返回：[{"text": "Apple Inc.", "type": "ORGANIZATION", "confidence": 0.98}, ...]

rel           = RelationExtractor(method="rule")
relationships = rel.extract(parsed, entities=entities)
# 返回：[{"subject": "Steve Jobs", "predicate": "founded", "object": "Apple Inc."}, ...]
```

```python 基于 LLM（精度更高）
from semantica.semantic_extract import NERExtractor, RelationExtractor
from semantica.llms import Groq

llm = Groq(model="llama-3.3-70b-versatile")

ner           = NERExtractor(method="llm", llm_provider=llm)
entities      = ner.extract(parsed)

rel           = RelationExtractor(method="llm", llm_provider=llm)
relationships = rel.extract(parsed, entities=entities)
```

</CodeGroup>

</Step>

<Step title="构建知识图谱">

将抽取到的实体与关系组装成一个可查询的知识图谱。

```python
from semantica.kg import GraphBuilder

builder = GraphBuilder(merge_entities=True)
graph   = builder.build({"entities": entities, "relationships": relationships})

print(f"Graph: {len(graph['entities'])} nodes, {len(graph['relationships'])} edges")
```

<Note>
  `merge_entities=True` 会基于语义相似度自动解析重复的实体引用："Apple"、"Apple Inc."、"AAPL"。无需手动去重。
</Note>

</Step>

<Step title="可视化">

在浏览器中渲染可交互、可缩放的知识图谱。

```python
from semantica.visualization import KGVisualizer

viz = KGVisualizer(
    layout="force",        # "force" | "hierarchical" | "circular"
)
viz.visualize_network(graph, output="html", file_path="graph.html", node_color_by="type")
```

在任意浏览器中打开 `graph.html`：平移、缩放、点击节点查看详情、按实体类型过滤。

</Step>

<Step title="导出">

导出为任意下游格式。

<CodeGroup>

```python RDF / 语义 Web
from semantica.export import RDFExporter

exporter = RDFExporter()
exporter.export(graph, file_path="graph.ttl",    format="turtle")
exporter.export(graph, file_path="graph.jsonld", format="json-ld")
exporter.export(graph, file_path="graph.nt",     format="nt")
```

```python Parquet / 分析
from semantica.export import ParquetExporter

exporter = ParquetExporter()
exporter.export(graph, file_path="output/graph.parquet")
# 写入 nodes.parquet + edges.parquet：可直接用于 Spark、BigQuery、Databricks
```

```python ArangoDB
from semantica.export import ArangoAQLExporter

exporter = ArangoAQLExporter()
aql      = exporter.export(graph)
# 返回可直接运行的 AQL INSERT 语句
```

</CodeGroup>

</Step>

</Steps>


## 添加决策智能

用完整的因果链与溯源追踪智能体的每一次决策：只需额外导入一个模块：

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=ContextGraph(advanced_analytics=True),
    decision_tracking=True,
)

# 存储一个带有溯源的事实
context.store("GPT-4 outperforms GPT-3.5 on reasoning benchmarks by 40%")

# 记录一次决策
decision_id = context.record_decision(
    category="model_selection",
    scenario="Choose LLM for production reasoning pipeline",
    reasoning="GPT-4 benchmark advantage justifies 3x cost increase",
    outcome="selected_gpt4",
    confidence=0.91,
)

# 检索相似的过往决策：避免不一致的选择
precedents = context.find_precedents("model selection reasoning", limit=5)
influence  = context.analyze_decision_influence(decision_id)
```


## 常见模式

<AccordionGroup>

<Accordion title="直接处理原始文本：无需文件" icon="text">

```python
from semantica.semantic_extract import NERExtractor, RelationExtractor

text = "Apple Inc. was founded by Steve Jobs, Steve Wozniak, and Ronald Wayne in 1976 in Cupertino, California."

ner           = NERExtractor()
entities      = ner.extract(text)

rel           = RelationExtractor()
relationships = rel.extract(text, entities=entities)
```

</Accordion>

<Accordion title="多源增量式图谱构建" icon="layer-group">

```python
from semantica.kg import GraphBuilder

builder     = GraphBuilder(merge_entities=True)
all_entities, all_rels = [], []

for doc in parsed_docs:
    entities = ner.extract(doc)
    rels     = rel.extract(doc, entities=entities)
    all_entities.extend(entities)
    all_rels.extend(rels)

graph = builder.build({"entities": all_entities, "relationships": all_rels})
```

</Accordion>

<Accordion title="支持时间点查询的时态知识图谱" icon="clock">

```python
from semantica.kg import GraphBuilder, TemporalGraphQuery

builder = GraphBuilder()
kg = builder.build({
    "entities": [
        {"id": "alice",     "type": "Person"},
        {"id": "acme_corp", "type": "Organization"},
        {"id": "beta_ltd",  "type": "Organization"},
    ],
    "relationships": [
        {
            "source": "alice", "target": "acme_corp", "type": "ceo_of",
            "valid_from": "2018-01-01", "valid_until": "2022-06-01",
        },
        {
            "source": "alice", "target": "beta_ltd", "type": "ceo_of",
            "valid_from": "2022-06-01",
        },
    ],
})

tq = TemporalGraphQuery(temporal_granularity="day")

result_2020 = tq.query_at_time(kg, query="",  # query 为未来用途保留
                               at_time="2020-06-15")
result_2023 = tq.query_at_time(kg, query="", at_time="2023-01-01")

print(f"Relationships active in 2020: {result_2020['num_relationships']}")
print(f"Relationships active in 2023: {result_2023['num_relationships']}")
```

</Accordion>

<Accordion title="持久化图存储：Neo4j、FalkorDB、Apache AGE" icon="database">

```python
from semantica.graph_store import Neo4jStore
from semantica.kg import GraphBuilder

store = Neo4jStore(
    uri="bolt://localhost:7687",
    user="neo4j",
    password="password",
)

builder = GraphBuilder(merge_entities=True, graph_store=store)
graph   = builder.build({"entities": entities, "relationships": relationships})
# 图谱已持久化到 Neo4j：在进程重启后依然存在
```

</Accordion>

<Accordion title="完整溯源流水线：W3C PROV-O" icon="link">

```python
from semantica.provenance import ProvenanceManager
from semantica.kg import GraphBuilder

prov    = ProvenanceManager()
prov.track_entity("Apple Inc.", "data/report.pdf", metadata={"confidence": 0.98})

builder = GraphBuilder(merge_entities=True)
graph   = builder.build({"entities": entities, "relationships": relationships})

# 检索任意实体的完整血缘
sources = prov.get_all_sources("Apple Inc.")
print(sources[0])
# {"source": "data/report.pdf", "location": None, "timestamp": "...", "confidence": 0.98}
```

</Accordion>

</AccordionGroup>


## 故障排查

<AccordionGroup>

<Accordion title="未抽取到任何实体" icon="magnifying-glass">

该文档可能包含的是扫描图像，而非可机读文本。请启用 OCR：

```python
from semantica.parse import DocumentParser

parser = DocumentParser(ocr=True)  # 启用 Tesseract OCR
parsed = parser.parse(sources[0])
```

</Accordion>

<Accordion title="大型语料处理缓慢" icon="gauge">

启用并行处理与 GPU 加速：

```bash
pip install semantica[gpu]
```

```python
from semantica.pipeline import Pipeline

pipeline = Pipeline(workers=8, batch_size=32)
pipeline.run(sources)
```

</Accordion>

<Accordion title="大型图谱出现内存错误" icon="memory">

从内存型 NetworkX 切换到持久化后端：

```python
from semantica.graph_store import FalkorDBStore

store   = FalkorDBStore(host="localhost", port=6379)
builder = GraphBuilder(merge_entities=True, graph_store=store)
```

</Accordion>

<Accordion title="NER 在企业网关上回退到模式模式" icon="triangle-exclamation">

已在 **v0.5.0** 中修复。请升级：

```bash
pip install --upgrade semantica
```

</Accordion>

</AccordionGroup>


## 后续步骤

- [核心概念](concepts.zh-CN.md) — 知识图谱、本体、推理引擎：Semantica 背后的心智模型。
- [模块参考](modules.zh-CN.md) — 每个模块的关键类与常见链路详解。
- [API 参考](reference/context.zh-CN.md) — 每个模块、类与参数的完整文档。
- [Cookbook](cookbook.zh-CN.md) — 40+ 基于真实数据集的交互式 Jupyter notebook。
