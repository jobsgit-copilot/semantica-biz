---
title: "选择合适的模块"
description: "30 秒内将你的目标映射到对应的 Semantica 模块。"
icon: "compass"
---

**[English](choose-your-module.md)** · **简体中文（当前）**

<Info>
  每个模块都可独立工作——按需导入即可。本页将开发者的目标映射到对应的起点。[模块参考](modules.zh-CN.md) 对每个模块有深入介绍。
</Info>

## 快速参考

在下方找到你的目标。**模块**列即你的导入路径；**关键类**是你首先实例化的类。

| 我想要…… | 模块 | 关键类 |
| :------------ | :------ | :--------- |
| 加载 PDF、DOCX、HTML、CSV 或压缩包 | `ingest` | `FileIngestor` |
| 爬取网站 | `ingest` | `WebIngestor` |
| 加载 Parquet 文件或分区数据集 | `ingest` | `ParquetIngestor` |
| 摄取带 schema 校验的 XML | `ingest` | `XMLIngestor` |
| 从 SQL、Snowflake、Databricks、Kafka 或邮件摄取 | `ingest` | `DBIngestor`、`SnowflakeIngestor`、`DatabricksIngestor`、`StreamIngestor` |
| 从文档中抽取干净的文本和表格 | `parse` | `DocumentParser` |
| 解析带 OCR 或多栏布局的复杂 PDF | `parse` | `DoclingParser` |
| 为向量嵌入或 RAG 对文本分块 | `split` | `TextSplitter` |
| 归一化文本、日期、实体或编码 | `normalize` | `TextNormalizer`、`EntityNormalizer` |
| 在文本中识别命名实体（人物、组织、地点） | `semantic_extract` | `NERExtractor` |
| 从文本中抽取带类型的关系 | `semantic_extract` | `RelationExtractor` |
| 抽取 RDF 主-谓-宾三元组 | `semantic_extract` | `TripletExtractor` |
| 构建可查询的知识图谱 | `kg` | `GraphBuilder` |
| 为事实添加时间有效性（`valid_from` / `valid_until`） | `kg` | `TemporalGraphQuery` |
| 运行图算法（中心性、社区发现、路径） | `kg` | `GraphAnalyzer`、`CentralityCalculator` |
| 生成向量嵌入 | `embeddings` | `EmbeddingGenerator` |
| 存储并检索向量 | `vector_store` | `VectorStore` |
| 将图谱持久化到 Neo4j 或 FalkorDB | `graph_store` | `Neo4jStore`、`FalkorDBStore` |
| 存储 RDF 三元组并用 SPARQL 查询 | `triplet_store` | `TripletStore` |
| 跨数据源对实体去重 | `deduplication` | `DuplicateDetector`、`EntityMerger` |
| 检测并解决相互矛盾的事实 | `conflicts` | `ConflictDetector`、`ConflictResolver` |
| 为 AI 智能体赋予持久记忆 | `context` | `AgentContext` |
| 用知识图谱接地 LLM 响应（GraphRAG） | `context` | `AgentContext.query_with_reasoning()` |
| 记录 AI 决策并保留完整审计追踪 | `context` | `AgentContext.record_decision()` |
| 在做出新决策前检索过往决策 | `context` | `AgentContext.find_precedents()` |
| 追溯某次决策的因果链 | `context` | `AgentContext.get_causal_chain()` |
| 追踪每个事实的来源（W3C PROV-O） | `provenance` | `ProvenanceManager` |
| 对图谱进行版本管理（带校验和与回滚） | `change_management` | `TemporalVersionManager` |
| 从图谱自动生成 OWL schema | `ontology` | `OntologyGenerator` |
| 用 SHACL 约束校验图谱 | `ontology` | `SHACLGenerator`、`OntologyValidator` |
| 从已有知识推导出新事实 | `reasoning` | `Reasoner`、`GraphReasoner` |
| 导出为 RDF Turtle、JSON-LD 或 N-Triples | `export` | `RDFExporter` |
| 导出为 Parquet 以供 Spark / BigQuery 使用 | `export` | `ParquetExporter` |
| 导出到 ArangoDB | `export` | `ArangoAQLExporter` |
| 通过 Cypher 导出到 Neo4j 或 Memgraph | `export` | `LPGExporter` |
| 交互式可视化知识图谱 | `visualization` | `KGVisualizer` |
| 运行可复现的多步骤流水线 | `pipeline` | `PipelineBuilder` |
| 在 Claude Desktop 或 Cursor 中使用 Semantica | `mcp_server` | `semantica-mcp` |
| 从已校验的种子数据引导生成图谱 | `seed` | `SeedDataManager` |
| 用自定义组件扩展 Semantica | `core` | `PluginRegistry` |


## 按目标查找起点

选择你的目标，查看最小导入量和一个可运行的骨架代码。

<Tabs>
  <Tab title="构建知识图谱">
    将文档、网页或数据库转化为结构化、可查询的图谱。

    **流水线：** `ingest` → `parse` → `semantic_extract` → `kg`

    ```python
    from semantica.ingest import FileIngestor
    from semantica.parse import DocumentParser
    from semantica.semantic_extract import NERExtractor, RelationExtractor
    from semantica.kg import GraphBuilder

    sources       = FileIngestor().ingest("report.pdf")
    parsed        = DocumentParser().parse_document("report.pdf")

    # 无需 API key——基于模式的抽取
    entities      = NERExtractor(method="pattern").extract(parsed)
    relationships = RelationExtractor(method="rule").extract(parsed, entities=entities)

    graph = GraphBuilder(merge_entities=True).build(
        sources=[{"entities": entities, "relationships": relationships}]
    )
    print(f"{len(graph.nodes)} nodes, {len(graph.edges)} edges")
    ```

    <Tip>
      向 `NERExtractor` 传入 `method="pattern"` 可实现零成本、无需 API key 的抽取。切换为 `method="llm"` 并搭配任意受支持的提供商可获得更高的召回率。
    </Tip>

    **下一步：** [快速上手 →](quickstart.zh-CN.md) — 含可视化与导出的完整流水线。
  </Tab>

  <Tab title="构建 GraphRAG">
    用结构化知识图谱接地每一次 LLM 响应。每一条声明都可回溯到某个源节点。

    **模块：** `context`

    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore
    from semantica.llms import Groq

    llm = Groq(model="llama-3.3-70b-versatile")

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(advanced_analytics=True),
    )

    # 存储事实——检索同时使用向量与图谱结构
    context.store("Apple Inc. was co-founded by Steve Jobs in 1976 in Cupertino.")

    # GraphRAG 查询，带多跳推理轨迹
    result = context.query_with_reasoning(
        "Who co-founded Apple?",
        llm_provider=llm,
        max_hops=2,
    )
    print(result["response"])        # 接地的答案
    print(result["reasoning_path"])  # 多跳轨迹
    ```

    **下一步：** [Context 模块参考 →](reference/context.zh-CN.md)
  </Tab>

  <Tab title="添加智能体记忆">
    为 AI 智能体赋予跨会话的持久记忆、决策追踪与先例检索。

    **模块：** `context`

    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(advanced_analytics=True),
        decision_tracking=True,   # 使用 record_decision() 必须开启
    )

    # 存储一条记忆
    context.store("GPT-4 outperforms GPT-3.5 on reasoning benchmarks by 40%.")

    # 记录一个带完整因果上下文的决策
    decision_id = context.record_decision(
        category="model_selection",
        scenario="Choose LLM for production reasoning pipeline",
        reasoning="GPT-4 benchmark advantage justifies cost increase",
        outcome="selected_gpt4",
        confidence=0.91,
    )

    # 在做出新决策前检索过往决策
    precedents = context.find_precedents("model selection", limit=5)

    # 追溯该决策的下游影响
    chain = context.get_causal_chain(decision_id, direction="downstream")
    ```

    <Note>
      必须设置 `decision_tracking=True`。否则 `record_decision()` 会抛出 `RuntimeError`。
    </Note>

    **下一步：** [Context 模块参考 →](reference/context.zh-CN.md)
  </Tab>

  <Tab title="追踪溯源">
    对每个事实记录 W3C PROV-O 血缘：来源文档、抽取方法、时间戳与校验和。

    **模块：** `provenance`、`change_management`

    ```python
    from semantica.provenance import ProvenanceManager

    prov = ProvenanceManager()

    # 追踪一个实体，记录完整的来源细节
    prov.track_entity(
        entity_id="entity_1",
        source="DOI:10.1371/journal.pone.0023601",
        source_location="Figure 2",
        confidence=0.92,
    )

    # 获取该实体的完整血缘
    lineage = prov.get_lineage("entity_1")

    # 使用 SHA-256 校验和对图谱做版本管理
    from semantica.change_management import TemporalVersionManager

    manager  = TemporalVersionManager()
    snapshot = manager.create_snapshot(kg, "v1.0", "user@example.com", "Initial build")
    diff     = manager.diff("v1.0", "v1.1")
    ```

    **下一步：** [Provenance 参考 →](reference/provenance.zh-CN.md) · [Change Management 参考 →](reference/change_management.zh-CN.md)
  </Tab>

  <Tab title="导出">
    将你的知识图谱序列化，用于语义网、分析平台或图数据库。

    **模块：** `export`

    ```python
    from semantica.export import RDFExporter, ParquetExporter, LPGExporter, ArangoAQLExporter

    # RDF——多种序列化格式
    RDFExporter().export(graph, "graph.ttl",    format="turtle")
    RDFExporter().export(graph, "graph.jsonld", format="jsonld")

    # Parquet——用于 Spark、BigQuery、Databricks、Snowflake
    ParquetExporter().export(graph, "output/graph.parquet")

    # 通过 Cypher 写入 Neo4j / Memgraph
    LPGExporter().export(graph, "graph.cypher")

    # ArangoDB AQL 插入语句
    ArangoAQLExporter().export(graph, "graph.aql")
    ```

    **格式：** Turtle · JSON-LD · N-Triples · RDF/XML · Parquet · Cypher · Arrow · OWL · CSV · ArangoDB AQL

    **下一步：** [Export 模块参考 →](reference/export.zh-CN.md)
  </Tab>

  <Tab title="MCP — Claude / Cursor">
    在 Claude Desktop、Cursor、VS Code 或任何支持 MCP 的工具中使用 Semantica——配置完成后无需编写 Python 代码。即刻可用 12 个工具。

    **第 1 步——安装：**
    ```bash
    pip install semantica
    ```

    **第 2 步——添加到你的 MCP 客户端配置：**

    <CodeGroup>

    ```json Claude Desktop / Windsurf / Cline
    {
      "mcpServers": {
        "semantica": {
          "command": "semantica-mcp"
        }
      }
    }
    ```

    ```json Cursor / VS Code / Continue
    {
      "mcpServers": {
        "semantica": {
          "command": "semantica-mcp",
          "env": {
            "SEMANTICA_KG_PATH": "/path/to/my_graph.json"
          }
        }
      }
    }
    ```

    </CodeGroup>

    **可用工具：** `extract_entities` · `extract_relations` · `add_entity` · `add_relationship` · `record_decision` · `query_decisions` · `find_precedents` · `get_causal_chain` · `run_reasoning` · `get_graph_analytics` · `export_graph` · `get_graph_summary`

    <Warning>
      设置 `SEMANTICA_KG_PATH` 以在重启后持久化你的图谱。否则当服务进程退出时，所有数据都会丢失。
    </Warning>

    **下一步：** [MCP Server 参考 →](reference/mcp_server.zh-CN.md)
  </Tab>
</Tabs>


## 仍然不确定？

<AccordionGroup>
  <Accordion title="知识图谱 vs. 向量库——我需要哪个？" icon="scale-balanced">
    当你需要结构化推理、多跳遍历、溯源或合规审计追踪时，使用**知识图谱**（`kg`）。

    当你需要在大型文本语料上做快速模糊相似度检索、且不关心条目之间的关系时，使用**向量库**（`vector_store`）。

    通过 `AgentContext`（GraphRAG）**两者结合使用**，可获得接地的 LLM 响应，每条声明都能追溯到某个源节点。

    另请参阅：[核心概念](concepts.zh-CN.md)
  </Accordion>

  <Accordion title="我只想快速跑个示例。" icon="rocket">
    从 [快速上手](quickstart.zh-CN.md) 开始。它会构建一条完整的流水线（摄取 → 解析 → 抽取 → 图谱 → 可视化 → 导出），且无需 API key。
  </Accordion>

  <Accordion title="我想把 Semantica 加到现有智能体中——最小改动是什么？" icon="plug">
    添加 `AgentContext`。它为你的现有智能体包裹上记忆、决策追踪和先例检索——无需更改你的 LLM 提供商或智能体框架。

    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(advanced_analytics=True),
        decision_tracking=True,
    )
    ```

    [Context 模块参考 →](reference/context.zh-CN.md)
  </Accordion>

  <Accordion title="我需要一条合规就绪的流水线——最小技术栈是什么？" icon="shield-check">
    | 层 | 模块 | 关键类 |
    | :---- | :------ | :--------- |
    | 数据摄取 | `ingest` | `FileIngestor` |
    | 抽取 | `semantic_extract` | `NERExtractor` |
    | 图谱 | `kg` | `GraphBuilder` |
    | 血缘 | `provenance` | `ProvenanceManager` |
    | 版本管理 | `change_management` | `TemporalVersionManager` |
    | 审计导出 | `export` | `RDFExporter` |
    支持 HIPAA、SOX、GDPR 以及 FDA 21 CFR Part 11 审计要求。
  </Accordion>
</AccordionGroup>

---

- [快速上手](quickstart.zh-CN.md) — 5 分钟跑通完整流水线。
- [模块参考](modules.zh-CN.md) — 每个模块的示例与常见链路。
- [API 参考](reference/context.zh-CN.md) — 完整的类与方法文档。
