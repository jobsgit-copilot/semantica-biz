---
title: "溯源模块"
description: "W3C PROV-O 血缘跟踪、来源归属、防篡改校验和以及跨所有模块的审计追踪。"
icon: "link"
---

**[English](provenance.md)** · **简体中文（当前）**

`semantica.provenance` 跟踪每个事实的完整血缘：从原始摄取到抽取、分块和关系构建：

- 符合 W3C PROV-O：适用于 HIPAA、SOX、GDPR、FDA 21 CFR Part 11 审计追踪
- 每个存储的 `ProvenanceEntry` 上使用 SHA-256 校验和进行篡改检测
- `SQLiteStorage` 用于跨重启持久化；`InMemoryStorage` 用于开发
- `ProvenanceManager` 提供 `track_entity`、`track_relationship`、`track_chunk` 和 `get_lineage`
- 通过 `BridgeAxiom` 桥接到 W3C PROV-O 本体，以进行语义 Web 导出


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `ProvenanceManager` | 中央跟踪器：`track_entity`、`track_relationship`、`track_chunk`、`get_lineage`、`get_statistics` |
| `ProvenanceEntry` | 单条血缘记录：`{entity_id, entity_type, activity_id, source_document, confidence, checksum, ...}` |
| `SourceReference` | 富来源指针：`{document, page, section, line, confidence, metadata}` |
| `ProvenanceStorage` | 抽象存储接口 |
| `InMemoryStorage` | 默认后端：快速，不在重启间持久化 |
| `SQLiteStorage` | 持久化后端：持久化到本地 SQLite 文件 |
| `compute_checksum` | 返回 `ProvenanceEntry` 的 SHA-256 指纹 |
| `verify_checksum` | 通过比较存储的与重新计算的哈希来检测篡改 |

<a id="getting-started"></a>
## 快速上手

<Tabs>
  <Tab title="内存中（默认）">
    零配置：快速，无磁盘写入。用于笔记本、测试和单次运行的脚本。

    ```python
    from semantica.provenance import ProvenanceManager, compute_checksum, verify_checksum

    manager = ProvenanceManager()   # 默认使用 InMemoryStorage

    entry = manager.track_entity(
        entity_id="apple_inc",
        source="annual_report_2023.pdf",
        source_location="Page 12, Section 3.1",
        source_quote="Apple Inc. was incorporated on January 3, 1977.",
        confidence=0.98,
    )

    print(entry.checksum)        # 自动计算的 SHA-256 十六进制
    print(verify_checksum(entry))  # True：篡改检测
    ```

    <Note>
      进程退出时内存存储会丢失。对于需要跨重启保留的内容，请使用 `SQLiteStorage`。
    </Note>
  </Tab>
  <Tab title="SQLite（持久化）">
    将溯源持久化到本地 SQLite 文件。用于生产流水线和审计追踪。

    ```python
    from semantica.provenance import ProvenanceManager, SQLiteStorage

    # 选项 1：显式存储实例
    manager = ProvenanceManager(storage=SQLiteStorage("provenance.db"))

    # 选项 2：简写：与上面等效
    manager = ProvenanceManager(storage_path="provenance.db")

    entry = manager.track_entity(
        entity_id="apple_inc",
        source="annual_report_2023.pdf",
        source_location="Page 12, Section 3.1",
        confidence=0.98,
    )

    # 重启后检索血缘：条目持久化在 provenance.db 中
    lineage = manager.get_lineage("apple_inc")
    print(f"{len(lineage)} provenance entries for apple_inc")
    ```

    <Check>
      SQLite 文件在首次写入时自动创建。无需模式设置。
    </Check>
  </Tab>
</Tabs>

<a id="provenancemanager"></a>
## ProvenanceManager

**`ProvenanceManager`** 是所有血缘数据的中央跟踪器。每次调用 `track_entity`、`track_relationship` 或 `track_chunk` 都会自动计算并存储 **SHA-256 校验和**，用于篡改检测。

<a id="constructor"></a>
### 构造函数

```python
ProvenanceManager(
    storage=None,        # ProvenanceStorage 实例；默认为 InMemoryStorage
    storage_path=None,   # str 路径：如果提供则创建 SQLiteStorage
)
```

如果 `storage` 和 `storage_path` 都被省略，则使用 `InMemoryStorage`。

<Warning>
  **`InMemoryStorage` 不会跨重启持久化。** 在审计追踪必须跨进程退出保留的任何环境中，传入 `storage_path="provenance.db"` 或显式的 `SQLiteStorage` 实例。
</Warning>

<a id="tracking-methods"></a>
### 跟踪方法

```python
from semantica.provenance import ProvenanceManager, SourceReference

manager = ProvenanceManager()

# 跟踪实体
entry = manager.track_entity(
    entity_id="apple_inc",        # 必需
    source="annual_report.pdf",   # 必需：文档 ID、DOI、文件路径
    source_location="Page 12",    # 可选关键字参数
    source_quote="Incorporated on January 3, 1977.",  # 可选关键字参数
    confidence=0.98,              # 可选关键字参数，默认 1.0
    entity_type="organization",   # 可选关键字参数，默认 "entity"
    metadata={"sector": "tech"},  # 可选的元数据字典
)

# 跟踪关系
rel_entry = manager.track_relationship(
    relationship_id="jobs_founded_apple",
    source="annual_report.pdf",
    confidence=0.95,
    metadata={"type": "founded"},
)

# 跟踪文档分块（分块后）
chunk_entry = manager.track_chunk(
    chunk_id="chunk_001",
    source_document="report.pdf",
    source_path="/docs/report.pdf",
    start_index=0,
    end_index=500,
    parent_chunk_id=None,
)

# 使用 SourceReference 跟踪属性
source_ref = SourceReference(
    document="DOI:10.1038/s41586-021-03371-z",
    page=4,
    section="Table S4",
    confidence=0.92,
)
prop_entry = manager.track_property_source(
    entity_id="cabo_pulmo",
    property_name="biomass_increase",
    value="463%",
    source=source_ref,
)
```

<a id="batch-tracking"></a>
### 批量跟踪

批量跟踪方法在共享事务中按块（默认 `batch_size=1000`）处理条目。只有成功提交到存储的实体或分块才会计入返回的计数，防止回滚的条目夸大成功计数。

```python
entities = [
    {"id": "entity_1", "confidence": 0.9},
    {"id": "entity_2", "confidence": 0.85},
]
count = manager.track_entities_batch(entities, source="doc_1")
# 返回成功跟踪并提交的实体数量

chunks = [
    {"id": "chunk_0", "start_index": 0, "end_index": 500},
    {"id": "chunk_1", "start_index": 500, "end_index": 1000},
]
count = manager.track_chunks_batch(chunks, source_document="doc_1")
```

<a id="retrieving-lineage"></a>
### 检索血缘

```python
# get_lineage 返回字典：而不是 ProvenanceEntry
lineage = manager.get_lineage("apple_inc")

print(lineage["entity_id"])        # "apple_inc"
print(lineage["source_documents"]) # ["annual_report.pdf"]
print(lineage["first_seen"])       # ISO 时间戳字符串
print(lineage["last_updated"])     # ISO 时间戳字符串
print(lineage["entity_count"])     # 链中的条目数
print(lineage["lineage_chain"])    # 条目字典列表（完整历史）
print(lineage["metadata"])         # 合并的元数据字典

# trace_lineage 返回原始 ProvenanceEntry 对象
entries = manager.trace_lineage("apple_inc")
for entry in entries:
    print(entry.entity_id, entry.source_document, entry.confidence)

# get_all_sources 返回来源字典列表
sources = manager.get_all_sources("apple_inc")
for s in sources:
    print(s["source"], s["location"], s["confidence"])

# get_provenance 以字典形式返回最近的条目（或 None）
prov = manager.get_provenance("apple_inc")
if prov:
    print(prov["source_document"])
```

<Note>
  `get_lineage()` 返回聚合的 **字典**，而不是 `ProvenanceEntry`。当你需要字段级别的访问（例如 `entry.checksum`）时，使用 `trace_lineage()` 获取原始 `ProvenanceEntry` 对象。
</Note>

<a id="utility-methods"></a>
### 实用方法

```python
# 关于所有被跟踪条目的统计信息
stats = manager.get_statistics()
# {"total_entries": 42, "entity_types": {"entity": 30, "chunk": 12}, "unique_sources": 5}

# 清除所有溯源数据；返回已清除的条目计数
cleared = manager.clear()
```

<a id="provenancemanager-methods-reference"></a>
### ProvenanceManager 方法参考

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `track_entity(entity_id, source, metadata, **kwargs)` | `Optional[ProvenanceEntry]` | 原子地记录实体溯源；成功时返回 `ProvenanceEntry`，存储失败时返回 `None`/现有条目 |
| `track_relationship(relationship_id, source, metadata, **kwargs)` | `Optional[ProvenanceEntry]` | 记录关系溯源；成功时返回 `ProvenanceEntry`，存储失败时返回 `None` |
| `track_chunk(chunk_id, source_document, ...)` | `Optional[ProvenanceEntry]` | 使用字符偏移量记录分块溯源；成功时返回 `ProvenanceEntry`，存储失败时返回 `None` |
| `track_property_source(entity_id, property_name, value, source)` | `Optional[ProvenanceEntry]` | 记录属性级别的来源归属；成功时返回 `ProvenanceEntry`，存储失败时返回 `None` |
| `track_entities_batch(entities, source)` | `int` | 批量跟踪实体；返回成功计数 |
| `track_chunks_batch(chunks, source_document)` | `int` | 批量跟踪分块；返回成功计数 |
| `get_lineage(entity_id)` | `Dict[str, Any]` | 作为聚合字典的完整血缘 |
| `trace_lineage(entity_id)` | `List[ProvenanceEntry]` | 作为原始 `ProvenanceEntry` 对象的完整血缘 |
| `get_all_sources(entity_id)` | `List[Dict]` | 实体的所有来源文档 |
| `get_provenance(entity_id)` | `Dict \| None` | 以字典形式返回最近的溯源条目 |
| `get_statistics()` | `Dict[str, Any]` | 按类型统计的条目数和唯一来源数 |
| `clear()` | `int` | 清除所有记录；返回已清除计数 |

<a id="provenanceentry-fields"></a>
## ProvenanceEntry 字段

`ProvenanceEntry` 是核心数据类。每个跟踪方法在成功时返回一个（存储失败时返回 `None`）：

```python
from semantica.provenance import ProvenanceEntry

# 所有字段及其类型和默认值
entry = ProvenanceEntry(
    entity_id="entity_001",           # str：必需
    entity_type="entity",             # str：必需（entity、chunk、relationship、property）
    activity_id="ner_extraction",     # str：必需
    agent_id="semantica",             # str：默认 "semantica"
    source_document="report.pdf",     # str：默认 ""
    source_location="Page 4",         # Optional[str]：默认 None
    source_quote="Relevant text...",  # Optional[str]：默认 None
    timestamp="2024-01-01T12:00:00",  # str：自动设置为 utcnow()
    first_seen=None,                  # Optional[str]：ISO 时间戳
    last_updated=None,                # Optional[str]：ISO 时间戳
    confidence=0.9,                   # float：默认 1.0
    checksum=None,                    # Optional[str]：由 compute_checksum() 设置
    parent_entity_id=None,            # Optional[str]：prov:wasDerivedFrom
    used_entities=[],                 # List[str]：prov:used
    start_index=None,                 # Optional[int]：用于分块
    end_index=None,                   # Optional[int]：用于分块
    credibility=None,                 # Optional[float]：来源可信度
    metadata={},                      # Dict[str, Any]
    version="1.0",                    # str
)

# 转换为字典
d = entry.to_dict()

# 从字典重建
entry2 = ProvenanceEntry.from_dict(d)
```

<a id="sourcereference-fields"></a>
## SourceReference 字段

`SourceReference` 提供一个指向来源文档中某个位置的可引用指针：

```python
from semantica.provenance import SourceReference

ref = SourceReference(
    document="DOI:10.1038/s41586-021-03371-z",  # str：必需（DOI、URL、文件路径）
    page=4,                                       # Optional[int]
    section="Table S4",                           # Optional[str]
    line=None,                                    # Optional[int]
    timestamp=None,                               # Optional[datetime]
    confidence=0.92,                              # float：默认 1.0
    metadata={"credibility": "peer-reviewed"},    # Dict[str, Any]
)

# 与 track_property_source 一起使用
manager.track_property_source(
    entity_id="cabo_pulmo",
    property_name="biomass_increase",
    value="463%",
    source=ref,
)
```

<a id="storage-backends"></a>
## 存储后端

<a id="inmemorystorage"></a>
### InMemoryStorage

快速，无持久化。适用于开发、测试和短生命周期的进程：

```python
from semantica.provenance import InMemoryStorage, ProvenanceManager

manager = ProvenanceManager(storage=InMemoryStorage())
```

<a id="sqlitestorage"></a>
### SQLiteStorage

持久化到磁盘。适用于生产、审计追踪和法规合规：

```python
from semantica.provenance import SQLiteStorage, ProvenanceManager

manager = ProvenanceManager(storage=SQLiteStorage("provenance.db"))

# 或使用简写
manager = ProvenanceManager(storage_path="provenance.db")
```

`SQLiteStorage` 在首次使用时自动创建数据库和索引。

- **原子性和并发性**：配置 Write-Ahead Logging（`PRAGMA journal_mode=WAL`）、`PRAGMA busy_timeout=5000` 和 `PRAGMA synchronous=NORMAL`。读-改-写方法（`track_entity()`、`store()`）打开单个连接并在立即写事务（`BEGIN IMMEDIATE`）中执行，确保这些序列在并发连接间被序列化，同时不会跨调用留下打开的文件句柄。普通读取（`retrieve()`、`trace_lineage()`）使用不带显式写锁的单独连接，因此并发读取不会在写者或彼此之后序列化。
- **向后兼容性**：覆盖 `trace_lineage(self, entity_id)` 的自定义存储子类保持向后兼容；`ProvenanceManager` 会检查覆盖签名，如果 `max_depth` 不受支持，则自动使用单个参数调用它。

<a id="tamper-evident-checksums"></a>
## 防篡改校验和

`compute_checksum` 和 `verify_checksum` 由 `track_entity` 和所有其他跟踪方法自动使用。你也可以直接调用它们：

```python
from semantica.provenance import compute_checksum, verify_checksum

entry = manager.trace_lineage("apple_inc")[0]

# 从条目字段重新计算校验和
checksum = compute_checksum(entry)

# 使用存储的校验和验证（entry.checksum）
is_valid = verify_checksum(entry)

# 或对照单独存储的预期校验和验证
is_valid = verify_checksum(entry, expected_checksum=checksum)

if not is_valid:
    raise RuntimeError("Provenance record has been tampered with.")
```

校验和覆盖 `entity_id`、`entity_type`、`activity_id`、`source_document`、`timestamp` 和 `confidence`。

<Tip>
  **在任何合规导出之前运行 `verify_checksum(entry)`。** 直接传入 `trace_lineage()` 返回的 `ProvenanceEntry` 对象。如果存储的校验和不再匹配，在导出继续之前抛出错误。
</Tip>

<a id="bridge-axiom-translation-chains"></a>
## 桥接公理转换链

`BridgeAxiom` 和 `TranslationChain` 可在 `semantica.provenance.bridge_axiom` 中获得，用于跟踪具有完整系数归属的多层领域转换：

```python
from semantica.provenance.bridge_axiom import BridgeAxiom, create_translation_chain
from semantica.provenance import ProvenanceManager

manager = ProvenanceManager()

# 使用 DOI 支持的系数定义桥接公理
axiom = BridgeAxiom(
    axiom_id="BA-001",
    name="biomass_tourism_elasticity",
    rule="1% biomass increase -> 0.346% tourism revenue increase",
    coefficient=0.346,
    source_doi="10.1038/s41586-021-03371-z",
    source_page="Table S4",
    confidence=0.92,
    input_domain="ecological",
    output_domain="financial",
)

# 应用于一个值并跟踪溯源
result = axiom.apply(
    input_entity="cabo_pulmo_biomass",
    input_value=463.0,
    prov_manager=manager,
)
print(result["output_value"])  # 463.0 * 0.346 = 160.098

# 构建多步转换链
input_data = {"entity_id": "cabo_pulmo", "value": 463.0, "source": "DOI:10.1371/..."}
chain = create_translation_chain(input_data, [axiom], prov_manager=manager)
print(chain.confidence)  # 0.92
```

<a id="integration-with-graphbuilder"></a>
## 与 GraphBuilder 集成

`GraphBuilderWithProvenance`（来自 `semantica.kg`）自动为每个节点和边记录溯源：

```python
from semantica.kg import GraphBuilderWithProvenance
from semantica.provenance import ProvenanceManager, SQLiteStorage

prov_manager = ProvenanceManager(storage=SQLiteStorage("provenance.db"))
builder = GraphBuilderWithProvenance(provenance_manager=prov_manager)
kg = builder.build_single_source(graph_data)

# 检索血缘：get_lineage 返回字典
lineage = prov_manager.get_lineage("apple_inc")
print(lineage["source_documents"])  # 来源文档 ID 列表
print(lineage["first_seen"])        # ISO 时间戳
```

<a id="integration-with-nerextractor"></a>
## 与 NERExtractor 集成

`NERExtractor` 和其他抽取器接受 `provenance=True` 以在每个抽取的实体上嵌入溯源元数据。你必须使用 `ProvenanceManager` 手动跟踪结果：

```python
from semantica.semantic_extract import NERExtractor
from semantica.provenance import ProvenanceManager

manager = ProvenanceManager()

ner = NERExtractor(method="ml", provenance=True)
entities = ner.extract("Steve Jobs founded Apple Inc.")

# 手动跟踪每个抽取的实体
for entity in entities:
    manager.track_entity(
        entity_id=entity.id,
        source="source_document.txt",
        confidence=entity.confidence,
        entity_type=entity.type,
    )

# 现在检索血缘
lineage = manager.get_lineage(entities[0].id)
print(lineage["source_documents"])
```

<Note>
  在 `NERExtractor` 上设置 `provenance=True` 会在抽取的实体对象上嵌入元数据 — 它不会自动调用 `ProvenanceManager.track_entity()`。你必须在抽取后自己调用 `track_entity()`。
</Note>

<a id="common-workflows"></a>
## 常见工作流

<Tabs>
  <Tab title="实体跟踪">
    ```python
    from semantica.provenance import ProvenanceManager

    manager = ProvenanceManager(storage_path="provenance.db")

    # 跟踪从文档中抽取的实体
    entry = manager.track_entity(
        entity_id="entity_001",
        source="report_2024.pdf",
        source_location="Page 5",
        source_quote="Revenue grew 12% year-over-year.",
        confidence=0.95,
    )
    # entry.checksum 自动设置

    # 检索完整血缘
    lineage = manager.get_lineage("entity_001")
    print(lineage["source_documents"])
    ```
  </Tab>
  <Tab title="分块跟踪">
    ```python
    from semantica.provenance import ProvenanceManager

    manager = ProvenanceManager()

    # 跟踪由 split 模块生成的分块
    manager.track_chunk(
        chunk_id="chunk_0001",
        source_document="report.pdf",
        start_index=0,
        end_index=512,
    )

    # 一次性批量跟踪所有分块
    chunks = [
        {"id": "c0", "start_index": 0,   "end_index": 512},
        {"id": "c1", "start_index": 512, "end_index": 1024},
    ]
    count = manager.track_chunks_batch(chunks, source_document="report.pdf")
    ```
  </Tab>
  <Tab title="完整性检查">
    ```python
    from semantica.provenance import (
        ProvenanceManager, compute_checksum, verify_checksum
    )

    manager = ProvenanceManager(storage_path="provenance.db")
    manager.track_entity("e1", source="doc.pdf", confidence=0.9)

    entries = manager.trace_lineage("e1")
    entry = entries[0]

    # 验证存储的校验和是否仍然有效
    if not verify_checksum(entry):
        raise RuntimeError("Provenance tampered: " + entry.entity_id)
    ```
  </Tab>
</Tabs>

<a id="compliance-notes"></a>
## 合规说明

Semantica 中的溯源跟踪产生以下审计工件：

| 标准 | 可用 |
| :-------- | :--------- |
| **W3C PROV-O** | 合规数据模型；`to_dict()` 和 `from_dict()` 用于序列化 |
| **HIPAA** | 审计追踪：实体 → 来源文档 → 时间戳 → 置信度 |
| **SOX** | 防篡改校验和；每个条目上的时间戳 |
| **GDPR** | 血缘图谱支持数据擦除影响分析 |
| **FDA 21 CFR Part 11** | 带有 `timestamp`、`agent_id`、`activity_id`、`checksum` 的电子记录 |

<Note>
  `ProvenanceManager` 不包含内置的 Turtle 或 JSON-LD 序列化。使用 `entry.to_dict()` 和 `get_lineage()` 检索溯源数据，然后如果需要 W3C PROV-O RDF 输出，使用你偏好的 RDF 库进行序列化。
</Note>

- [变更管理](change_management.zh-CN.md) — 版本控制和快照审计追踪。
- [摄取](ingest.zh-CN.md) — 溯源从摄取阶段开始。
- [导出](export.zh-CN.md) — 在 RDF 导出中包含溯源元数据。
- [上下文](context.zh-CN.md) — 通过 AgentContext 的决策溯源。
