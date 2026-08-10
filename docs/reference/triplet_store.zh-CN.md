---
title: "三元组库模块"
description: "嵌入式与服务端支持的 RDF 存储，支持 SPARQL 查询与批量加载。"
icon: "table"
---

**[English](triplet_store.md)** · **简体中文（当前）**

`semantica.triplet_store` 提供 **W3C 标准的 RDF 存储**，支持 **SPARQL 1.1** 查询。当你需要 **语义 Web 兼容性**、OWL 风格的推理、基于 SPARQL 的查询，或符合标准的 RDF 序列化时使用它。

<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :---- | :--- |
| `TripletStore` | 统一接口：`add_triplet`、`add_triplets`、`get_triplets`、`delete_triplet`、`execute_query` |
| `QueryEngine` | **SPARQL 1.1** 执行，支持查询优化与结果缓存 |
| `BulkLoader` | 大批量 RDF 加载，支持批处理、重试与进度跟踪 |
| `BlazegraphStore` | Blazegraph REST API：SPARQL 1.1 Update、命名空间管理 |
| `JenaStore` | Apache Jena：基于 rdflib，通过远程端点支持 SPARQL 读取 |
| `RDF4JStore` | Eclipse RDF4J：REST API、事务支持 |
| `Oxigraph store` | 嵌入式 SPARQL 1.1 存储，支持内存模式与磁盘模式 |

<a id="what-you-get"></a>
## 你将获得

- **TripletStore** — 跨嵌入式 Oxigraph、Blazegraph、Apache Jena 和 RDF4J 的统一接口：通过一个参数即可切换后端。
- **SPARQL** — 通过 `execute_query()` 全面支持 SPARQL SELECT、ASK、CONSTRUCT 和 UPDATE 查询。
- **批量加载** — `add_triplets()` 批量写入，支持可配置的批次大小、重试逻辑与进度跟踪。
- **SKOS 词汇表** — 内置辅助方法：`add_skos_concept()` 和 `get_skos_concepts()`，用于受控词汇表管理。
- **命名图** — Oxigraph、Blazegraph 和 RDF4J 支持通过 `execute_query()` 的 `graph=` 参数进行命名图作用域限定。
- **差异计算** — `compute_delta(old_graph_uri, new_graph_uri)` 返回两个命名图快照之间新增和删除的三元组。

<a id="getting-started"></a>
## 入门

**`TripletStore`** 包装你选择的后端。构造一个 `Triplet` 对象（来自 `semantica.semantic_extract.types`）并调用 `add_triplet()`：

```python
from semantica.triplet_store import TripletStore
from semantica.semantic_extract.types import Triplet

store = TripletStore(
    backend="blazegraph",
    endpoint="http://localhost:9999/blazegraph/sparql"
)

# 创建并存储单个三元组
t = Triplet(
    subject="http://example.org/apple_inc",
    predicate="http://example.org/founded_by",
    object="http://example.org/steve_jobs",
)
store.add_triplet(t)

# 使用 SPARQL 查询：返回包含 .bindings 列表的 QueryResult
result = store.execute_query("""
    PREFIX ex: <http://example.org/>
    SELECT ?person ?company WHERE {
        ?person ex:founded ?company .
    }
""")

for row in result.bindings:
    person  = row.get("person",  {}).get("value")
    company = row.get("company", {}).get("value")
    print(person, company)
```

<a id="quick-start"></a>
## 快速上手

<Steps>
  <Step title="连接到后端">
    ```python
    from semantica.triplet_store import TripletStore

    store = TripletStore(
        backend="blazegraph",
        endpoint="http://localhost:9999/blazegraph/sparql"
    )
    ```
  </Step>
  <Step title="添加三元组">
    ```python
    from semantica.semantic_extract.types import Triplet

    # 添加单个三元组
    store.add_triplet(Triplet(
        subject="http://example.org/apple_inc",
        predicate="http://example.org/founded_by",
        object="http://example.org/steve_jobs",
    ))

    # 批量添加 Triplet 对象列表
    store.add_triplets(triplets, batch_size=500)
    ```
  </Step>
  <Step title="使用 SPARQL 查询">
    ```python
    # execute_query 返回 QueryResult：遍历 result.bindings
    result = store.execute_query("""
        PREFIX ex: <http://example.org/>
        SELECT ?person ?company WHERE {
            ?person ex:founded ?company .
            ?company ex:located_in ex:SiliconValley .
        }
    """)

    for row in result.bindings:
        print(row.get("person",  {}).get("value"))
        print(row.get("company", {}).get("value"))
    ```
  </Step>
  <Step title="存储整个知识图谱">
    ```python
    # store(knowledge_graph, ontology) 将实体/关系
    # 转换为 RDF 三元组并在一次调用中批量加载
    store.store(knowledge_graph=kg_dict, ontology=ontology_dict)
    ```
  </Step>
</Steps>

<a id="backends"></a>
## 后端

<Tabs>
  <Tab title="Oxigraph">
    ```bash
    pip install "semantica[tripletstore-oxigraph]"
    ```

    ```python
    # 内存模式：无需服务器进程或文件
    store = TripletStore(backend="oxigraph")

    # 持久模式：重新打开同一目录以复用数据
    persistent_store = TripletStore(
        backend="oxigraph",
        path="./data/knowledge-graph",
    )
    ```

    **最适合：** 本地开发、CI、桌面应用程序，以及无需外部基础设施的持久单进程工作负载。
  </Tab>
  <Tab title="Blazegraph">
    ```bash
    pip install requests
    ```

    ```python
    from semantica.triplet_store import TripletStore

    store = TripletStore(
        backend="blazegraph",
        endpoint="http://localhost:9999/blazegraph/sparql",
        namespace="kb",      # 默认值："kb"
        timeout=30,          # 请求超时时间（秒）
    )
    ```

    **最适合：** Wikidata 风格工作负载、大量三元组、命名图支持、SPARQL 1.1 Update。
  </Tab>
  <Tab title="Apache Jena">
    ```bash
    pip install rdflib
    ```

    ```python
    store = TripletStore(
        backend="jena",
        endpoint="http://localhost:3030/ds",   # rdflib SPARQLStore 的 SPARQL 读取端点
    )
    ```

    **最适合：** 使用 rdflib 的本地开发、针对 Fuseki 端点的 SPARQL 读取查询。

    <Warning>
      **`backend="jena"` 的 OWL 推理是占位实现。** `enable_inference=True` 会被接受，但推理调用返回 0 个推断三元组。对于生产环境的 OWL 推理，请直接使用 Jena Fuseki 及其内置的推理器配置。
    </Warning>
  </Tab>
  <Tab title="RDF4J">
    ```bash
    pip install requests
    ```

    ```python
    store = TripletStore(
        backend="rdf4j",
        endpoint="http://localhost:8080/rdf4j-server",
        repository_id="semantica",   # 通过 **config 传递
    )
    ```

    **最适合：** Eclipse Foundation 部署、通过 REST API 进行基于事务的加载。
  </Tab>
  <Tab title="后端对比">

    | 后端 | 许可证 | 命名图 | 写入方式 | 最适合 |
    | :------- | :------- | :------------ | :--------- | :-------- |
    | Oxigraph | Apache 2.0 / MIT | 是 | 嵌入式原生 API | 本地、CI、磁盘存储 |
    | Blazegraph | 开源 | 是 | SPARQL Update REST | 大量三元组、SPARQL 1.1 |
    | Apache Jena | Apache 2.0 | 否（rdflib 后端） | rdflib 进程内 | 本地开发、读取查询 |
    | RDF4J | Eclipse 1.0 | 是 | REST API N-Triples | 企业级 Java、事务 |

  </Tab>
</Tabs>

<Tip>
  **使用 Oxigraph 进行零基础设施开发与本地持久化。**
  在分布式生产部署中，通过更改 `backend=` 即可切换到服务端存储。
</Tip>

<a id="triplet-object"></a>
## Triplet 对象

所有存储操作使用来自 `semantica.semantic_extract.types` 的 `Triplet` 数据类：

```python
from semantica.semantic_extract.types import Triplet

t = Triplet(
    subject="http://example.org/apple_inc",     # 必填：完整 URI 字符串
    predicate="http://example.org/founded_by",  # 必填：完整 URI 字符串
    object="http://example.org/steve_jobs",     # 必填：URI 或字面量字符串
    confidence=0.95,                            # 可选：float 0.0–1.0，默认 1.0
    metadata={"source": "wikipedia"},           # 可选：dict
)
```

| 字段 | 类型 | 默认值 | 描述 |
| :----- | :---- | :------- | :----------- |
| `subject` | `str` | **必填** | 主体 URI |
| `predicate` | `str` | **必填** | 谓词 URI |
| `object` | `str` | **必填** | 客体 URI 或字面量 |
| `confidence` | `float` | `1.0` | 置信度分数（0–1） |
| `metadata` | `dict` | `{}` | 任意元数据 |

<Warning>
  **`add_triplet()` 接受 `Triplet` 对象，而非关键字参数。** 使用来自 `semantica.semantic_extract.types` 的 `Triplet(subject=..., predicate=..., object=...)` 并传递该对象：不要向 `add_triplet` 传递 `subject=`、`predicate=`、`obj=`。
</Warning>

<a id="tripletstore-methods"></a>
## TripletStore 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `add_triplet(triplet)` | `dict` | 添加单个 `Triplet` 对象 |
| `add_triplets(triplets, batch_size)` | `dict` | 批量添加 `Triplet` 对象列表；返回 `{"success", "total", "processed", "failed", "batches"}` |
| `get_triplets(subject, predicate, object)` | `List[Triplet]` | 检索匹配 subject/predicate/object 过滤器的三元组 |
| `delete_triplet(triplet)` | `dict` | 从存储中删除一个 `Triplet` |
| `update_triplet(old_triplet, new_triplet)` | `dict` | 原子化删除 + 添加 |
| `execute_query(query, parameters, graph, graphs)` | `QueryResult` | 执行 SPARQL 查询：返回 `QueryResult`，包含 `.bindings`、`.variables`、`.execution_time` |
| `store(knowledge_graph, ontology)` | `dict` | 将 KG + 本体字典转换为 RDF 三元组并批量加载 |
| `add_skos_concept(concept_uri, scheme_uri, pref_label, ...)` | `dict` | 添加 SKOS 概念，支持可选的替代标签、broader/narrower/related |
| `get_skos_concepts(scheme_uri)` | `List[dict]` | 检索所有 SKOS 概念，可按 scheme URI 过滤 |
| `compute_delta(old_graph_uri, new_graph_uri)` | `dict` | 返回两个命名图快照之间的 `{"added_triples", "removed_triples", "added_count", "removed_count"}` |
| `get_stats()` | `dict` | 获取后端统计信息 |

<a id="sparql-queries"></a>
## SPARQL 查询

`execute_query()` 是所有 SPARQL 操作的唯一入口。它返回 `QueryResult`：通过 `.bindings` 访问结果：

```python
from semantica.triplet_store import TripletStore

store = TripletStore(backend="blazegraph", endpoint="http://localhost:9999/blazegraph/sparql")

# SELECT：遍历 result.bindings
result = store.execute_query("""
    PREFIX ex: <http://example.org/>
    SELECT ?person ?company WHERE {
        ?person ex:founded ?company .
    }
""")

for row in result.bindings:
    print(row.get("person",  {}).get("value"))
    print(row.get("company", {}).get("value"))

# ASK、CONSTRUCT、UPDATE：相同方法，不同 SPARQL 形式
result = store.execute_query("""
    PREFIX ex: <http://example.org/>
    ASK { ex:apple_inc ex:founded_by ex:steve_jobs . }
""")
print(result.bindings)   # ASK 在 bindings 中返回布尔结果

# SPARQL UPDATE（INSERT/DELETE）
store.execute_query("""
    PREFIX ex: <http://example.org/>
    INSERT DATA {
        ex:apple_inc ex:listed_on ex:NASDAQ .
    }
""")
```

<a id="queryresult-fields"></a>
### QueryResult 字段

| 字段 | 类型 | 描述 |
| :----- | :---- | :----------- |
| `bindings` | `List[dict]` | 每个 dict 将变量名映射为 `{"value": ..., "type": ...}` |
| `variables` | `List[str]` | SPARQL 结果变量名 |
| `execution_time` | `float` | 耗时（秒） |
| `metadata` | `dict` | 查询、图作用域、缓存命中标志 |

<Warning>
  **`execute_query()` 返回 `QueryResult`，而非列表。** 遍历 `result.bindings`，不要直接遍历 `result`。每个 binding 是一个将变量名映射为 `{"value": ..., "type": ...}` 的 dict。
</Warning>

<a id="sparql-construct-templates"></a>
## SPARQL CONSTRUCT 模板

`semantica.triplet_store.construct_templates` 提供参数化的 SPARQL `CONSTRUCT` 查询模板：一次定义可复用的查询，安全地替换类型化参数，并在一次调用中持久化生成的三元组。这仅适用于 **Blazegraph 后端**（参见上文的[后端](#后端)）—— `BlazegraphStore.execute_sparql()` 是唯一具备 CONSTRUCT 感知 RDF 解析能力的后端。

```python
from semantica.triplet_store.construct_templates import (
    ConstructTemplate,
    ParameterDescriptor,
    ConstructTemplateRegistry,
    render_construct_template,
    execute_construct_template,
)
from semantica.triplet_store import BlazegraphStore

# 定义并注册模板
template = ConstructTemplate(
    name="person_to_foaf",
    description="Maps a person record subject to a foaf:name triple",
    construct_query="""
        PREFIX foaf: <http://xmlns.com/foaf/0.1/>
        CONSTRUCT { {{subject}} foaf:name {{name}} ; foaf:age {{age}} }
        WHERE { {{subject}} a <http://ex.org/Person> }
    """,
    parameters=[
        ParameterDescriptor(name="subject", type="uri", required=True),
        ParameterDescriptor(name="name", type="literal", required=True),
        ParameterDescriptor(
            name="age", type="typed-literal", required=False, default=0,
            datatype="xsd:integer",
        ),
    ],
)

registry = ConstructTemplateRegistry()
registry.register(template)

# 仅渲染：检查替换后的 SPARQL 字符串，无网络调用
sparql = render_construct_template(
    registry.get("person_to_foaf"),
    params={"subject": "http://ex.org/p1", "name": "Alice", "age": 30},
)

# 渲染 + 执行 + 持久化一次完成
store = BlazegraphStore(endpoint="http://localhost:9999/blazegraph", namespace="kb")
triplets = execute_construct_template(
    template=registry.get("person_to_foaf"),
    params={"subject": "http://ex.org/p1", "name": "Alice", "age": 30},
    store_backend=store,
    target_graph="http://ex.org/graphs/people",
)
# triplets: List[Triplet]，已通过 store.add_triplets 持久化
```

每个 `ParameterDescriptor.type` 控制其值的渲染方式：`"uri"` 值会根据允许列表进行验证并包裹在 `<...>` 中，`"literal"` 值会被转义并加引号，`"typed-literal"` 值需要 `datatype`（例如 `"xsd:integer"`），并在数值/布尔 XSD 类型下不加引号渲染。占位符使用 `{{param}}` 而非 SPARQL 自身的 `?param` 语法，这样模板占位符永远不会与查询体中真正的 SPARQL 变量混淆。

<Note>
  CONSTRUCT 模板仅适用于 Blazegraph。如果 `store_backend` 未同时实现 `execute_sparql()` 和 `add_triplets()`，`execute_construct_template()` 会抛出 `ProcessingError`。
</Note>

<a id="sparql-result-pagination"></a>
## SPARQL 结果分页

对于大型结果集，使用 LIMIT 和 OFFSET 进行分页：

```python
page_size = 1000
offset    = 0

while True:
    result = store.execute_query(f"""
        SELECT ?s ?p ?o WHERE {{
            ?s ?p ?o .
        }}
        ORDER BY ?s
        LIMIT {page_size} OFFSET {offset}
    """)
    if not result.bindings:
        break
    process_batch(result.bindings)
    offset += page_size
```

<Warning>
  **对大型 SPARQL 结果集进行分页。** 对大型存储执行 `SELECT * WHERE { ?s ?p ?o }` 会返回所有三元组。在探索性查询中务必包含 `LIMIT` 和 `OFFSET`。除非你指定了 LIMIT，否则 `QueryEngine` 会自动添加 `LIMIT 1000`。
</Warning>

<a id="named-graph-scoping"></a>
## 命名图作用域

Oxigraph、Blazegraph 和 RDF4J 支持命名图。使用 `graph=` 参数将 `execute_query()` 限定到某个命名图：

```python
# 向命名图添加三元组
from semantica.semantic_extract.types import Triplet

t = Triplet(
    subject="http://example.org/a",
    predicate="http://example.org/p",
    object="http://example.org/b",
)
store.add_triplet(t, graph="http://example.org/graph1")

# 通过 SPARQL 中的 FROM 子句查询命名图
result = store.execute_query("""
    SELECT ?s ?p ?o WHERE {
        ?s ?p ?o .
    }
""", graph="http://example.org/graph1")   # 在 WHERE 之前注入 FROM <graph>

# 或者在查询字符串中使用 FROM 内联限定作用域
result = store.execute_query("""
    SELECT ?s ?p ?o FROM <http://example.org/graph1> WHERE {
        ?s ?p ?o .
    }
""")
```

<Note>
  命名图查询作用域限定适用于 Oxigraph、Blazegraph 和 RDF4J。
  `graph=` 查询参数在 Jena 后端上会被静默忽略。
</Note>

<Tip>
  **使用命名图隔离数据源。** 在写入和 `execute_query()` 时传递 `graph="http://example.org/source_A"`
  可同时限定存储与检索的作用域。Oxigraph、Blazegraph 和 RDF4J 支持命名图查询作用域限定。
</Tip>

<a id="bulk-loading"></a>
## 批量加载

`add_triplets()` 通过内部的 `BulkLoader` 批量写入。访问 `store.bulk_loader` 进行配置：

```python
from semantica.triplet_store import TripletStore
from semantica.semantic_extract.types import Triplet

store = TripletStore(backend="blazegraph", endpoint="http://localhost:9999/blazegraph/sparql")

# 默认值：batch_size=1000, max_retries=3
result = store.add_triplets(triplets)
# 返回：{"success": True, "total": N, "processed": N, "failed": 0, "batches": B}

# 本次调用的自定义批次大小
result = store.add_triplets(triplets, batch_size=500)

# 加载前验证
validation = store.bulk_loader.validate_before_load(triplets)
if not validation["valid"]:
    print(validation["errors"])
```

`BulkLoader` 也可以通过 `progress_callback` 直接使用：

```python
from semantica.triplet_store import BulkLoader

loader = BulkLoader(batch_size=2000, max_retries=5, retry_delay=2.0)

def on_progress(p):
    print(f"{p.progress_percentage:.1f}%  ({p.loaded_triplets}/{p.total_triplets})")

progress = loader.load_triplets(triplets, store._store_backend, progress_callback=on_progress)
print(f"Loaded {progress.loaded_triplets} in {progress.elapsed_time:.2f}s")
```

<a id="storing-a-knowledge-graph"></a>
## 存储知识图谱

`store(knowledge_graph, ontology)` 将 KG+本体字典结构转换为 RDF，并在一次调用中批量加载所有内容：

```python
kg = {
    "entities": [
        {"id": "apple_inc",  "type": "Organization", "properties": {"name": "Apple Inc."}},
        {"id": "steve_jobs", "type": "Person",        "properties": {"name": "Steve Jobs"}},
    ],
    "relationships": [
        {"source": "steve_jobs", "target": "apple_inc", "type": "founded"}
    ],
}
ontology = {
    "uri": "https://example.org/ontology/",
    "classes": [
        {"name": "Organization"},
        {"name": "Person"},
    ],
    "properties": [
        {"name": "founded", "type": "object", "domain": ["Person"], "range": ["Organization"]},
    ],
}

result = store.store(knowledge_graph=kg, ontology=ontology)
# 返回 add_triplets 的结果字典
```

<a id="skos-vocabulary-management"></a>
## SKOS 词汇表管理

```python
store.add_skos_concept(
    concept_uri="http://example.org/skos/MachineLearning",
    scheme_uri="http://example.org/skos/AIScheme",
    pref_label="Machine Learning",
    alt_labels=["ML", "Statistical Learning"],
    broader=["http://example.org/skos/AI"],
    definition="A field of artificial intelligence...",
)

# 检索某个 scheme 中的所有概念
concepts = store.get_skos_concepts(scheme_uri="http://example.org/skos/AIScheme")
for c in concepts:
    print(c["uri"], c["pref_label"], c["alt_labels"])
```

<a id="delta-computation"></a>
## 差异计算

```python
# 比较两个命名图快照，返回新增/删除的三元组
delta = store.compute_delta(
    old_graph_uri="http://example.org/graph/v1",
    new_graph_uri="http://example.org/graph/v2",
)

print(f"Added:   {delta['added_count']} triples")
print(f"Removed: {delta['removed_count']} triples")

for t in delta["added_triples"]:
    print(f"+  {t.subject}  {t.predicate}  {t.object}")
for t in delta["removed_triples"]:
    print(f"-  {t.subject}  {t.predicate}  {t.object}")
```

<a id="integration-with-export-module"></a>
## 与 Export 模块的集成

Export 模块写入的 RDF 可以随后通过 `add_triplets()` 由三元组库接收：

```python
from semantica.export import RDFExporter
from semantica.triplet_store import TripletStore
from semantica.semantic_extract.types import Triplet

# 将 KG 导出为 Turtle
exporter = RDFExporter()
exporter.export_to_file(kg, "output.ttl", format="turtle")

# 解析文件并将三元组加载到存储中
# (TripletStore 没有内置的 import_file() 方法 ——
#  使用 rdflib 解析并转换为 Triplet 对象)
import rdflib
g = rdflib.Graph()
g.parse("output.ttl", format="turtle")

store = TripletStore(backend="jena", endpoint="http://localhost:3030/ds")
triplets = [
    Triplet(subject=str(s), predicate=str(p), object=str(o))
    for s, p, o in g
]
store.add_triplets(triplets)

# 使用 SPARQL 查询
result = store.execute_query("SELECT * WHERE { ?s ?p ?o } LIMIT 10")
for row in result.bindings:
    print(row)
```

- [Export](export.zh-CN.md) — 将知识图谱导出为 RDF 格式。
- [Ontology](ontology.zh-CN.md) — 加载 OWL 本体并存储为 RDF 三元组。
- [Reasoning](reasoning.zh-CN.md) — 基于 SPARQL 的属性链推理。
- [Graph Store](graph_store.zh-CN.md) — 用于 Cypher 查询的属性图替代方案。
