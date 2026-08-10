---
title: "导出模块"
description: "将知识图谱导出为 RDF、Parquet、LPG、ArangoDB AQL、CSV、GraphML、OWL、JSON-LD、Arrow 和向量格式。"
icon: "file-export"
---

**[English](export.md)** · **简体中文（当前）**

**`semantica.export`** 将知识图谱序列化为**所有下游格式**：

- RDF：Turtle、JSON-LD、N-Triples、RDF/XML：可选内联 W3C PROV-O 溯源
- 分析：面向 Spark、BigQuery、Databricks 的 Apache Parquet 和 Arrow
- 图数据库：用于 Neo4j 的 Cypher `CREATE` 语句；用于 ArangoDB 的 AQL `INSERT`
- 标准格式：GraphML、GEXF、Graphviz DOT、CSV、OWL 2.0
- 向量导出：用于向量嵌入流水线的 NumPy `.npz`、FAISS 索引、二进制


<a id="exported-classes"></a>
## 导出的类

| 类 | 输出格式 | 说明 |
| :--- | :--- | :--- |
| `RDFExporter` | Turtle、JSON-LD、N-Triples、RDF/XML | `export_to_rdf()` → 字符串；`export()` → 文件 |
| `ParquetExporter` | `.parquet` | 需要 `pyarrow`；显式类型化模式 |
| `LPGExporter` | Cypher `CREATE` | 兼容 Neo4j 和 Memgraph |
| `ArangoAQLExporter` | AQL `INSERT` | 顶点和边集合 |
| `GraphExporter` | GraphML、GEXF、Graphviz DOT | 标准图交换格式 |
| `OWLExporter` | Turtle/XML 格式的 OWL 2.0 | 本体序列化 |
| `CSVExporter` | `.csv` | `export_entities()` 与 `export_relationships()` |
| `VectorExporter` | JSON、NumPy `.npz`、FAISS 索引、二进制 | 向量嵌入导出 |
| `ArrowExporter` | Apache Arrow IPC | 需要 `pyarrow`；零拷贝传输 |
| `DistanceExporter` | CSV、JSONL | 成对距离指标；接受一个 `graph` 参数 |
| `ReportGenerator` | HTML、Markdown、JSON、纯文本 | 分析报告 |
| `NamespaceManager` |: | RDF 命名空间抽取与声明生成 |

<a id="getting-started"></a>
## 入门

```python
from semantica.export import RDFExporter

# 将知识图谱字典导出为 Turtle
exporter = RDFExporter()
rdf_str = exporter.export_to_rdf(graph, format="turtle")
with open("output.ttl", "w") as f:
    f.write(rdf_str)
```

或使用单行的便捷函数：

```python
from semantica.export import export_rdf, export_csv, export_lpg

export_rdf(graph,  "output.ttl",    format="turtle")
export_csv(graph,  "output_base")   # 将实体和关系分别写为 CSV
export_lpg(graph,  "import.cypher", method="cypher")
```

<a id="quick-export"></a>
## 快速导出

<Steps>
  <Step title="导出为 RDF 字符串">
    ```python
    from semantica.export import RDFExporter

    exporter = RDFExporter()
    rdf_str = exporter.export_to_rdf(graph, format="turtle")
    ```
  </Step>
  <Step title="将 RDF 直接写入文件">
    ```python
    exporter.export(graph, "output.ttl", format="turtle")
    ```
  </Step>
  <Step title="用于分析的列式导出">
    ```python
    from semantica.export import ParquetExporter

    exporter = ParquetExporter(compression="snappy")
    exporter.export_entities(entities, "nodes.parquet")
    exporter.export_relationships(relationships, "edges.parquet")
    ```
  </Step>
  <Step title="图数据库导入">
    ```python
    from semantica.export import LPGExporter

    exporter = LPGExporter()
    exporter.export(graph, "import.cypher")   # Cypher CREATE 语句
    ```
  </Step>
</Steps>

<a id="exporters"></a>
## 各导出器

<Tabs>
  <Tab title="RDF">
    导出为 W3C RDF 格式：Turtle、JSON-LD、N-Triples 和 RDF/XML。

    **`export_to_rdf()` 返回字符串；`export()` 写入文件：**

    ```python
    from semantica.export import RDFExporter

    exporter = RDFExporter()

    # 返回 RDF 字符串
    turtle_str  = exporter.export_to_rdf(graph, format="turtle")   # Turtle
    jsonld_str  = exporter.export_to_rdf(graph, format="jsonld")   # JSON-LD
    nt_str      = exporter.export_to_rdf(graph, format="ntriples") # N-Triples
    xml_str     = exporter.export_to_rdf(graph, format="rdfxml")   # RDF/XML

    # 接受的格式别名："ttl" -> turtle, "nt" -> ntriples, "xml" -> rdfxml,
    # "json-ld" -> jsonld, "rdf" -> rdfxml

    # 直接写入文件
    exporter.export(graph, "output.ttl", format="turtle")

    # 同样可用
    exporter.export_knowledge_graph(graph, "output.ttl", format="turtle")
    ```

    <Warning>
      **`export_to_rdf()` 返回字符串：它不会写文件。** 请调用 `export()` 或 `export_knowledge_graph()` 以直接写入磁盘。
    </Warning>

    <Tip>
      **检查时使用 `export_to_rdf()` + 字符串，生产环境使用 `export()`。** 在 notebook 或调试会话中，`export_to_rdf()` 便于快速检查。对于写文件的 CI 流水线和常规流水线，`export()` 一次调用即可。
    </Tip>

    <Tip>
      **人类阅读用 `turtle`，流式处理用 `ntriples`。** Turtle 紧凑且易读，适合调试与分享。N-Triples（`.nt`）是面向行的：每行一个三元组，便于安全地流式处理、拼接，并用标准 Unix 工具处理。
    </Tip>

    **命名空间管理：**

    ```python
    from semantica.export import NamespaceManager, RDFExporter

    ns_manager = NamespaceManager()
    # ns_manager.namespaces 包含内置前缀字典（rdf、rdfs、owl、xsd、semantica）
    # 通过直接更新字典来添加自定义命名空间
    ns_manager.namespaces["ex"]     = "http://example.org/"
    ns_manager.namespaces["schema"] = "https://schema.org/"

    # 生成 Turtle 前缀声明
    decls = ns_manager.generate_namespace_declarations(
        ns_manager.namespaces, format="turtle"
    )
    print(decls)   # @prefix ex: <http://example.org/> .   等
    ```

    **时序导出（OWL-Time）：**

    ```python
    # 传入 include_temporal=True 以嵌入 OWL-Time 区间三元组
    turtle_str = exporter.export_to_rdf(
        graph,
        format="turtle",
        include_temporal=True,
        time_axis="valid",  # "valid" | "transaction" | "both"
    )
    ```
  </Tab>
  <Tab title="列式与分析">
    ```python
    from semantica.export import ParquetExporter

    exporter = ParquetExporter(compression="snappy")
    # compression: snappy | gzip | brotli | zstd | lz4 | none

    # 将实体和关系导出为独立的 Parquet 文件
    exporter.export_entities(entities,           "nodes.parquet")
    exporter.export_relationships(relationships, "edges.parquet")

    # 导出完整的知识图谱（写入 entities.parquet 和 relationships.parquet）
    exporter.export_knowledge_graph(graph, "output_base")
    # → output_base_entities.parquet, output_base_relationships.parquet

    # 从列表或字典进行通用导出
    exporter.export(entities, "entities.parquet")
    exporter.export(graph,    "output_base")
    ```

    <Warning>
      **`ParquetExporter` 和 `ArrowExporter` 需要 `pyarrow`。** 两者在未安装 `pyarrow` 时都会回退为一个空操作的占位类。使用这些导出器前请用 `pip install pyarrow` 安装。
    </Warning>

    <Tip>
      **将 `ParquetExporter` 用于下游分析。** Parquet 保留了 CSV 会丢失的列类型（int、float、datetime），并被 Spark、BigQuery、Databricks 和 Snowflake 原生支持。使用 `compression="snappy"` 可在速度和压缩率之间取得良好平衡。
    </Tip>

    需要 `pyarrow`：`pip install pyarrow`。模式为显式类型化。

    ```python
    from semantica.export import CSVExporter

    exporter = CSVExporter(delimiter=",")
    exporter.export_entities(entities,           "nodes.csv")
    exporter.export_relationships(relationships, "edges.csv")
    exporter.export_knowledge_graph(graph,       "output_base")
    ```

    ```python
    from semantica.export import SemanticNetworkYAMLExporter

    exporter = SemanticNetworkYAMLExporter()
    exporter.export(graph, "graph.yaml")
    ```
  </Tab>
  <Tab title="图数据库导入">
    **LPGExporter** 为 Neo4j 和 Memgraph 写入 Cypher `CREATE` 语句：

    ```python
    from semantica.export import LPGExporter

    exporter = LPGExporter()

    # 将 Cypher CREATE 语句写入文件
    exporter.export(graph, "import.cypher")

    # 同样可用
    exporter.export_knowledge_graph(graph, "import.cypher")
    ```

    **ArangoAQLExporter** 为 ArangoDB 写入 `INSERT` 语句：

    ```python
    from semantica.export import ArangoAQLExporter

    exporter = ArangoAQLExporter(
        vertex_collection="entities",
        edge_collection="relationships"
    )

    # 将 AQL INSERT 语句写入文件
    exporter.export(graph, "import.aql")
    exporter.export_knowledge_graph(graph, "import.aql")
    ```

    两个导出器都写入文件并返回 `None`。

    <Warning>
      **`ArangoAQLExporter.export()` 和 `LPGExporter.export()` 写入文件并返回 `None`。** 它们不会返回 AQL/Cypher 字符串。如果需要字符串，请先写入文件再读回。
    </Warning>
  </Tab>
  <Tab title="可视化与 OWL">
    ```python
    from semantica.export import GraphExporter

    exporter = GraphExporter()
    exporter.export(graph, "graph.graphml", format="graphml")  # Gephi、yEd
    exporter.export(graph, "graph.gexf",    format="gexf")     # Gephi 流式格式
    exporter.export(graph, "graph.dot",     format="dot")      # Graphviz
    ```

    ```python
    from semantica.export import OWLExporter

    exporter = OWLExporter()
    exporter.export(ontology, path="ontology.owl", format="owl-xml")
    exporter.export(ontology, path="ontology.ttl", format="turtle")
    ```
  </Tab>
  <Tab title="向量、Arrow 与报告">
    **VectorExporter**：接受 `(vectors, file_path, format=)`：

    ```python
    from semantica.export import VectorExporter

    exporter = VectorExporter()
    # vectors：包含 'id'、'vector'、'text'、'metadata' 键的字典列表
    exporter.export(vectors, "vectors.json",  format="json")
    exporter.export(vectors, "vectors.npz",   format="numpy")   # NumPy .npz
    exporter.export(vectors, "vectors.bin",   format="binary")
    exporter.export(vectors, "vectors.faiss", format="faiss")
    ```

    **ArrowExporter**：需要 `pyarrow`：

    ```python
    from semantica.export import ArrowExporter

    exporter = ArrowExporter()
    exporter.export(graph, "graph.arrow")
    ```

    **DistanceExporter**：在构造时接受一个 `graph` 参数：

    ```python
    from semantica.export import DistanceExporter

    exporter = DistanceExporter(graph)   # graph 为必填

    # 计算所有成对距离并写入文件
    exporter.to_csv("distances.csv")
    exporter.to_jsonl("distances.jsonl")

    # 带列选择和可选节点子集进行计算
    exporter.to_csv(
        "distances.csv",
        include=["source_id", "target_id", "hop_count", "distance_band"],
        node_subset=["node_a", "node_b", "node_c"],
    )

    # 以 pandas DataFrame 返回（需要 pandas）
    df = exporter.to_dataframe(include=["hop_count", "semantic_similarity"])

    # 以字符串返回（适用于 API 响应）
    csv_str  = exporter.to_csv_string(node_subset=["node_a", "node_b"])
    jsonl_str = exporter.to_jsonl_string()
    ```

    可用的 `include` 列：`source_id`、`source_type`、`target_id`、`target_type`、`hop_count`、`weighted_distance`、`semantic_similarity`、`distance_band`、`source_betweenness`、`target_betweenness`。

    <Warning>
      **`DistanceExporter` 在构造时需要一个 graph。** 请用 `DistanceExporter(graph)` 实例化，而不是 `DistanceExporter()`。语义相似度列（`semantic_similarity`）要求图节点在其属性中存有向量嵌入。
    </Warning>

    **ReportGenerator：**

    ```python
    from semantica.export import ReportGenerator

    generator = ReportGenerator()
    generator.generate_report(data, "report.html",  format="html")
    generator.generate_report(data, "report.md",    format="markdown")
    generator.generate_report(data, "report.json",  format="json")
    generator.generate_report(data, "report.txt",   format="text")
    ```
  </Tab>
</Tabs>

<a id="convenience-functions"></a>
## 便捷函数

```python
from semantica.export import (
    export_rdf, export_json, export_parquet, export_csv,
    export_lpg, export_arango, export_graph, export_owl,
    export_vector, export_arrow, export_yaml, generate_report,
)

export_rdf(graph,     "output.ttl",    format="turtle")
export_rdf(graph,     "output.nt",     format="ntriples")
export_json(graph,    "output.json",   format="json")
export_parquet(graph, "output_base",   compression="snappy")
export_csv(graph,     "output_base")   # 使用 CSVExporter.export()
export_lpg(graph,     "import.cypher", method="cypher")
export_arango(graph,  "import.aql")
export_graph(graph,   "graph.graphml", format="graphml")
export_owl(ontology,  "ontology.owl",  format="owl-xml")
export_vector(vectors,"vectors.json",  format="json")
export_arrow(graph,   "graph.arrow")
export_yaml(graph,    "graph.yaml",    method="semantic_network")
generate_report(data, "report.html",   format="html")
```

`export_csv` 便捷函数委托给 `CSVExporter.export()`。如需按类型导出，请直接使用该类（`exporter.export_entities()`、`exporter.export_relationships()`）。

<a id="format-reference"></a>
## 格式参考

| 格式字符串 | 规范名称 | 导出器 | 文件扩展名 | 最适用途 |
| :--- | :--- | :--- | :--- | :--- |
| `"turtle"` / `"ttl"` | `turtle` | `RDFExporter` | `.ttl` | 可读的 RDF、本体分享 |
| `"jsonld"` / `"json-ld"` | `jsonld` | `RDFExporter` | `.jsonld` | API、Linked Data、JSON 流水线 |
| `"ntriples"` / `"nt"` | `ntriples` | `RDFExporter` | `.nt` | 流式 RDF、逐行处理 |
| `"rdfxml"` / `"xml"` / `"rdf"` | `rdfxml` | `RDFExporter` | `.rdf` | W3C RDF/XML，兼容性最广 |
| `"parquet"` | `parquet` | `ParquetExporter` | `.parquet` | Spark、BigQuery、Databricks、Snowflake |
| `"cypher"` | `cypher` | `LPGExporter` | `.cypher` | Neo4j、Memgraph 导入 |
| `"aql"` | `aql` | `ArangoAQLExporter` | `.aql` | ArangoDB 顶点 + 边集合 |
| `"graphml"` | `graphml` | `GraphExporter` | `.graphml` | Gephi、yEd 可视化 |
| `"gexf"` | `gexf` | `GraphExporter` | `.gexf` | Gephi 流式格式 |
| `"dot"` | `dot` | `GraphExporter` | `.dot` | Graphviz 渲染 |
| `"owl-xml"` | `owl-xml` | `OWLExporter` | `.owl` | OWL 2.0 本体分发 |
| `"csv"` | `csv` | `CSVExporter` | `.csv` | 电子表格、简单流水线 |
| `"yaml"` | `yaml` | `SemanticNetworkYAMLExporter` | `.yaml` | 人类可读的配置驱动用途 |
| `"arrow"` | `arrow` | `ArrowExporter` | `.arrow` | 零拷贝进程间传输 |
| `"json"` | `json` | `VectorExporter` | `.json` | 向量嵌入 |
| `"numpy"` | `numpy` | `VectorExporter` | `.npz` | 来自向量嵌入的 NumPy 数组 |
| `"binary"` | `binary` | `VectorExporter` | `.bin` | 原始 float32 二进制 |
| `"faiss"` | `faiss` | `VectorExporter` | `.faiss` | 直接的 FAISS 索引文件 |
| `"html"` / `"markdown"` / `"json"` / `"text"` |: | `ReportGenerator` | `.html` / `.md` / `.json` / `.txt` | 分析报告 |

<Tip>
  **根据消费方匹配导出格式。** Neo4j → `cypher`；ArangoDB → `aql`；Gephi/yEd → `graphml` 或 `gexf`；语义网工具 → `turtle` 或 `json-ld`；分析流水线 → `parquet`；零拷贝 IPC → `arrow`。
</Tip>

- [三元组库](triplet_store.zh-CN.md) —— 将 RDF 导出结果存储到支持 SPARQL 查询的后端。
- [本体](ontology.zh-CN.md) —— 导出 OWL 本体。
- [溯源](provenance.zh-CN.md) —— 在 RDF 导出中包含溯源元数据。
- [流水线](pipeline.zh-CN.md) —— 将导出作为流水线的最终步骤。
