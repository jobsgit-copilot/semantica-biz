---
title: "溯源与审计追踪"
description: "Semantica 如何通过 W3C PROV-O 合规记录追踪每个实体、关系、分块和属性的来源与血缘——包含 SHA-256 完整性校验和、SQLite 持久化以及面向国防、制药、银行和安全合规的跨模块审计追踪。"
icon: "file-certificate"
---

**[English](provenance.md)** · **简体中文（当前）**

<a id="what-is-provenance"></a>
## 什么是溯源？

溯源是对数据来源、转换方式以及生命周期中每个步骤的责任人的系统性记录。与仅仅描述实体的普通图谱元数据不同，溯源创建了一条不可变的审计追踪，记录系统中每条信息的完整历史。

**核心溯源概念：**

**血缘**追踪从原始来源到当前状态的监管链，经过所有转换，精确展示数据随时间的演变过程。

**来源归因**记录产生每个数据元素的具体文档、数据库、API 调用或人工输入，实现精确引用和验证。

**完整性验证**使用加密校验和来检测溯源记录创建后的任何未授权更改。

**审计追踪**通过维护所有数据操作、转换和决策的防篡改日志来满足监管合规要求。

溯源与简单元数据的区别在于，它创建了法律上可辩护的、加密可验证的记录，能回答关键问题："这从哪里来？"、"谁处理了它？"、"何时发生了变更？"以及"是否被篡改过？"

<a id="why-use-provenance"></a>
## 为何使用溯源？

**满足监管要求。**满足 FDA 21 CFR Part 11、ICH E6(R2) GCP、Basel III BCBS 239 以及国防情报共享协议的要求，这些法规要求完整的数据可追溯性和电子记录完整性。

**来源归因与引用。**将每个实体、关系和属性值追溯到其确切的源文档、API 响应或人工输入，以支持科学可重复性和法律可辩护性。

**可审计性与透明度。**为审计师、监管机构和利益相关者提供对数据处理工作流的完整可见性，包括谁执行了每项操作以及变更何时发生。

**冲突解决与数据质量。**当多个来源为同一属性提供不同值时，溯源记录通过比较来源可信度、时效性和置信度来实现基于证据的冲突解决。

**篡改检测与取证。**加密完整性验证可检测对数据记录的未授权修改，支持安全敏感环境中的事件响应和取证分析。

**数据血缘可追溯性。**回答关于数据谱系的复杂问题，尤其是在实体经历抽取、富化、融合和分析转换的多阶段处理流水线中。

<a id="when-to-use-when-not-to-use"></a>
## 适用与不适用场景

**使用溯源追踪的场景：**
- 需要审计追踪的受监管环境（医疗、金融、国防、制药）
- 必须以证据解决冲突信息的多源数据融合
- 数据质量和来源可信度至关重要的长期知识图谱
- 数据完整性和篡改检测至关重要的生产系统
- 实体经历多次转换的复杂处理流水线
- 需要为基于抽取数据的决策提供法律可辩护性的场景

**溯源可能不必要的场景：**
- 不需要合规的简单原型和概念验证演示
- 处理一次数据后立即丢弃结果的临时工作流
- 不在会话间持久化数据的无状态应用
- 使用可信单一来源数据的内部研究项目
- 溯源开销影响性能的高频、低延迟操作
- 所有数据来自单一、高度可信且从不变化的来源的场景

**以下情况可考虑更简单的替代方案：**
- 基本元数据（创建时间戳、源文件名）提供了足够的可追溯性
- 仅通过版本控制即可实现数据处理的可透明再现
- 监管合规不要求加密完整性验证

`ProvenanceManager` 为每个实体、关系、文档分块和属性值记录一条 W3C PROV-O 合规条目——带有用于篡改检测的 SHA-256 校验和，以及每次 `track_entity()` 调用时自动进行的版本链接。当你需要回答关于某个值从哪里来、谁写入的、以及自首次摄取以来是否发生变化的监管问题时使用它。

<Info>
KG 流水线在抽取的所有内容上自动调用 `track_entity()` 和 `track_relationship()`，因此通过标准流水线进入的实体已被自动追踪。当你需要自定义审计集成、跨模块血缘链或跨多个来源的细粒度属性级归因时，使用此处介绍的手动 API。
</Info>

<a id="setting-up-the-provenance-store"></a>
## 设置溯源存储

`ProvenanceManager` 支持两种存储后端。内存存储零依赖，适合测试。SQLite 存储可跨重启持久化，支持并发读取，并给你的合规团队一个可以直接查询的标准数据库。

```python
from semantica.provenance import ProvenanceManager

# 内存——仅当前会话，无持久化
prov = ProvenanceManager()

# SQLite——持久化到磁盘，自由并发读取
prov = ProvenanceManager(storage_path="provenance.db")

# 自定义后端——传入任何 ProvenanceStorage 实现
from semantica.provenance.storage import SQLiteStorage
prov = ProvenanceManager(storage=SQLiteStorage("audit.db"))
```

对于任何受监管的部署——安全运营、临床数据、金融风险——使用 `storage_path`。SQLite 文件可以备份、版本化，并使用标准工具查询，无需服务器。

<Note>
  `SQLiteStorage` 会自动配置 Write-Ahead Logging（`WAL`）、`busy_timeout=5000` 和 `synchronous=NORMAL`，并以原子即时事务（`BEGIN IMMEDIATE`）执行读-改-写操作（如 `track_entity()`）；纯读操作（`retrieve()`、`trace_lineage()`）使用单独的连接，没有显式写锁，因此不会在写者之后串行化。此外，`ProvenanceManager` 自动支持自定义存储后端仅重写 `trace_lineage(self, entity_id)` 而无需在其签名中要求 `max_depth`。
</Note>

<a id="recording-provenance-when-ingesting-data"></a>
## 摄取数据时记录溯源

数据进入图谱的那一刻就是必须记录溯源的时刻。`track_entity()` 捕获源文档、时间戳、运行抽取的操作员或流水线、来自来源的逐字引用以及置信度分数。它返回一个 `Optional[ProvenanceEntry]`（成功时为 `ProvenanceEntry`，如果在新实体上存储失败则为 `None`），并自动计算 SHA-256 校验和。

```python
# 从 NVD 和商业订阅源摄取 CVE-2024-3400
# 两者分别追踪，以保留完整的多来源图景

entry_nvd = prov.track_entity(
    entity_id="cve-2024-3400",
    source="NVD_feed_2024-04-12",
    metadata={
        "cvss_score": 10.0,
        "vector": "AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H",
        "exploit_status": "unconfirmed",
    },
    confidence=0.98,
    entity_type="vulnerability",
    activity_id="nvd_feed_ingestion",
    source_location="CVE-2024-3400 JSON record",
    source_quote='{"cvssMetricV31":[{"cvssData":{"baseScore":10.0}}]}',
)

print(f"Entity tracked : {entry_nvd.entity_id}")
print(f"Source         : {entry_nvd.source_document}")
print(f"Timestamp      : {entry_nvd.timestamp}")
print(f"Checksum       : {entry_nvd.checksum}")       # SHA-256 hex digest
print(f"First seen     : {entry_nvd.first_seen}")     # 仅在首次调用时设置
```

```text
Entity tracked : cve-2024-3400
Source         : NVD_feed_2024-04-12
Timestamp      : 2024-04-12T14:22:07.881Z
Checksum       : 3f7a9c2d...                          # tamper-detectable
First seen     : 2024-04-12T14:22:07.881Z
```

现在追踪一小时后从商业订阅源到达的同一实体。在同一 `entity_id` 上再次调用 `track_entity()` 会自动将 NVD 条目归档为历史记录，并创建一个通过 `parent_entity_id` 链接到它的新当前条目：

```python
entry_commercial = prov.track_entity(
    entity_id="cve-2024-3400",
    source="commercial_feed_2024-04-12",
    metadata={
        "cvss_score": 9.8,
        "exploit_status": "in_wild",
        "observed_exploitation": True,
    },
    confidence=0.91,
    entity_type="vulnerability",
    activity_id="commercial_feed_ingestion",
)

# NVD 条目现被归档为 cve-2024-3400:v:2024-04-12T14:22:07
# 商业订阅源条目是新的当前状态
# entry_commercial.parent_entity_id == "cve-2024-3400:v:2024-04-12T14:22:07"
```

此版本链接自动完成。你无需手动管理历史条目。

<a id="tracking-multi-source-property-values"></a>
## 追踪多来源属性值

当同一属性出现在多个来源中且有不同值时——正是 CVE 评分的情况——使用 `track_property_source()` 分别记录每个归因。这直接为下游的冲突检测提供数据：冲突模块可以比较属性的所有已追踪值，并附带完整的来源元数据来呈现分歧。

**SourceReference** 是一个结构化元数据容器，精确捕获一条信息在文档中的来源位置。它包含文档标识符、具体位置（页、节、字节范围）、置信度级别以及用于特定领域归因需求的自定义元数据字段。

```python
from semantica.provenance.schemas import SourceReference

nvd_ref = SourceReference(
    document="NVD_feed_2024-04-12",
    section="cvssMetricV31",
    confidence=0.98,
    metadata={"publisher": "NIST", "feed_type": "NVD"},
)

commercial_ref = SourceReference(
    document="commercial_feed_2024-04-12",
    section="cvss_assessment",
    confidence=0.91,
    metadata={"publisher": "ThreatFeed-Co", "observed": True},
)

# 在同一实体 + 属性键下分别追踪每个来源的值
prov.track_property_source("cve-2024-3400", "cvss_score", 10.0, nvd_ref)
prov.track_property_source("cve-2024-3400", "cvss_score", 9.8,  commercial_ref)

# 属性来源存储在 "<entity_id>_<property_name>" 下
# 之后：检索此属性的所有来源来回答"9.8 从哪里来？"
sources = prov.get_all_sources("cve-2024-3400_cvss_score")
for s in sources:
    print(f"{s['source']:<35}  confidence={s['confidence']:.2f}  loc={s['location'] or '—'}")
```

```text
NVD_feed_2024-04-12                   confidence=0.98  loc=cvssMetricV31
commercial_feed_2024-04-12            confidence=0.91  loc=cvss_assessment
```

当监管机构问"9.8 从哪里来？"时，这就是答案：`commercial_feed_2024-04-12`，节 `cvss_assessment`，置信度 0.91，完整元数据表明这是一个报告了观测利用的商业发布者。

<a id="tracing-the-lineage-of-a-node"></a>
## 追踪节点的血缘

一旦实体有了多个溯源条目，你就可以追踪其完整历史来了解它随时间的演变。摄取六个月后，运行血缘追踪。`get_lineage()` 返回完整的版本链——实体经过的每个状态，从最旧到最新——以及摘要元数据：

```python
lineage = prov.get_lineage("cve-2024-3400")

print(f"Entity      : {lineage['entity_id']}")
print(f"First seen  : {lineage['first_seen']}")
print(f"Last updated: {lineage['last_updated']}")
print(f"History depth: {lineage['entity_count']} entries")
print(f"Sources seen : {lineage['source_documents']}")
print()
print("Full version chain (oldest → newest):")
for entry in lineage["lineage_chain"]:
    print(f"  [{entry['timestamp'][:19]}]  agent={entry['agent_id']}")
    print(f"    source={entry['source_document']}")
    print(f"    activity={entry['activity_id']}")
```

```text
Entity      : cve-2024-3400
First seen  : 2024-04-12T14:22:07.881Z
Last updated: 2024-10-08T09:11:44.302Z
History depth: 4 entries
Sources seen : ['NVD_feed_2024-04-12', 'commercial_feed_2024-04-12',
                'NVD_feed_2024-07-18', 'commercial_feed_2024-10-08']

Full version chain (oldest → newest):
  [2024-04-12T14:22:07]  agent=semantica
    source=NVD_feed_2024-04-12
    activity=nvd_feed_ingestion
  [2024-04-12T15:18:33]  agent=semantica
    source=commercial_feed_2024-04-12
    activity=commercial_feed_ingestion
  [2024-07-18T08:04:11]  agent=semantica
    source=NVD_feed_2024-07-18
    activity=nvd_feed_ingestion       # NVD 更新了他们的评分
  [2024-10-08T09:11:44]  agent=semantica
    source=commercial_feed_2024-10-08
    activity=commercial_feed_ingestion
```

这条链回答了监管机构的全部三个问题。9.8 来自 `commercial_feed_2024-04-12`。操作员是 `threat_ingest_pipeline_v2`。评分已发生变化——NVD 在 7 月 18 日更新了其记录——而链精确显示了时间。

<a id="verifying-integrity"></a>
## 验证完整性

每个 `ProvenanceEntry` 都带有写入时计算的 SHA-256 校验和。如果任何字段在事后被修改——无论是由配置错误的流水线、数据库迁移还是蓄意篡改——重新计算的校验和将不匹配。

完整性验证对监管合规和取证分析至关重要。在任何合规审计中运行完整性检查：

```python
from semantica.provenance.integrity import compute_checksum

raw_entries = prov.trace_lineage("cve-2024-3400")

print("Integrity check:")
for e in raw_entries:
    stored   = e.checksum
    computed = compute_checksum(e)
    status   = "OK" if stored == computed else "TAMPERED"
    print(f"  [{status}] {e.entity_id[:50]}  {(stored or '')[:16]}...")
```

```text
Integrity check:
  [OK] cve-2024-3400                                 3f7a9c2d...
  [OK] cve-2024-3400:v:2024-04-12T14:22:07           a1b2c3d4...
  [OK] cve-2024-3400:v:2024-04-12T15:18:33           e5f6a7b8...
  [OK] cve-2024-3400:v:2024-07-18T08:04:11           c9d0e1f2...
```

`TAMPERED` 状态意味着存储的哈希与从当前字段值计算的哈希不匹配——这是写入后修改的证据，必须在该记录用于合规目的之前进行调查。

<a id="tracking-document-chunks-and-their-children"></a>
## 追踪文档分块及其子块

溯源不仅限于实体。当文档被切分为分块用于检索增强生成（RAG）或自然语言处理工作流时，每个分块都需要自己的溯源记录，将其链接到源文件和字节范围。

子分块（来自递归分块）通过 `parent_chunk_id` 链接到其父块，这映射到 W3C PROV-O 标准中的 `prov:wasDerivedFrom`：

```python
# 追踪父分块（公告 PDF 的一个章节）
prov.track_chunk(
    chunk_id="advisory_section_3",
    source_document="CISA_advisory_AA24-099A.pdf",
    source_path="/feeds/cisa/advisories/AA24-099A.pdf",
    start_index=4096,
    end_index=8192,
)

# 追踪从递归分块派生的子分块
prov.track_chunk(
    chunk_id="advisory_section_3a",
    source_document="CISA_advisory_AA24-099A.pdf",
    source_path="/feeds/cisa/advisories/AA24-099A.pdf",
    start_index=4096,
    end_index=6144,
    parent_chunk_id="advisory_section_3",   # prov:wasDerivedFrom
)

# 检索分块的溯源记录
record = prov.get_provenance("advisory_section_3a")
if record:
    print(f"Source   : {record['source_document']}")
    print(f"Range    : bytes {record['start_index']}–{record['end_index']}")
    print(f"Parent   : {record['parent_entity_id']}")  # advisory_section_3
    print(f"Checksum : {record['checksum']}")
```

对于 GDPR 删除权工作流，每个分块溯源记录中的字节范围确切告诉你当数据主体提出删除请求时需要删除哪个文档的哪一部分。

<a id="statistics-across-the-provenance-store"></a>
## 溯源存储的统计信息

在大规模摄取运行之后，`get_statistics()` 提供已追踪所有内容的摘要：

```python
stats = prov.get_statistics()

print(f"Total tracked    : {stats['total_entries']}")
print(f"By entity type   : {stats['entity_types']}")
print(f"Unique sources   : {stats['unique_sources']}")
```

```text
Total tracked    : 14,822
By entity type   : {'vulnerability': 3041, 'chunk': 8204, 'relationship': 2891,
                    'property': 686}
Unique sources   : 12
```

此摘要是合规证明的起点：你可以陈述已追踪记录的总数、不同数据来源的数量以及按记录类型的明细。

<a id="common-pitfalls"></a>
## 常见陷阱

**溯源不保证真实性。**溯源记录忠实地追踪信息来自何处以及如何被处理，但它无法验证原始来源是否准确。从有缺陷或恶意来源出发的完美文档化链仍然会产生不可靠的数据。

**重复使用通用来源标识符。**使用非特定的来源 ID 如"daily_feed"或"batch_001"使得无法将单个记录追溯到其确切来源。始终在来源文档名称中包含时间戳、版本号或唯一批次标识符。

**绕过溯源工作流。**手动插入数据或使用跳过 `track_entity()` 调用的临时脚本会在审计追踪中造成缺口。确保所有数据入口——自动化流水线、手动更正和管理操作——都记录适当的溯源。

**忽视血缘验证。**溯源链在多阶段处理流水线中可能变得复杂。定期验证 `get_lineage()` 和 `trace_lineage()` 返回完整的、逻辑连贯的链，没有缺失链接或循环引用。

**在低价值场景中过度使用溯源。**为每个中间计算或临时变量记录溯源会造成存储开销而无合规收益。将溯源追踪集中在具有法律、监管或业务意义的实体、关系和属性上。

**未能验证完整性校验和。**加密完整性验证只有在实际检查时才有效。在审计工作流和事件响应流程中包含定期的 `compute_checksum()` 验证。

**混合溯源粒度。**在文档级别追踪某些实体而在句子级别追踪其他实体会导致不一致的审计追踪。为每种数据类型和处理工作流建立一致的粒度标准。

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防 — CTI/威胁">

信号情报融合单元追踪每个情报实体从原始采集、经分析处理到成品的监管链。链的每个层级——原始采集、NER 抽取、融合和成品情报——必须分别记录，附带适当的密级处理和操作员身份。溯源链就是监管链：它证明成品情报产品在每一步都可追溯到授权采集和授权分析。

根据 ITAR 和情报共享协议，溯源记录必须显示哪种采集方法产生了原始数据、哪位分析师处理了它、以及哪种融合活动在实体到达成品之前将其与其他情报进行了组合。`track_chunk()`、`track_entity()` 和 `track_relationship()` 分别对应该链的一个层级。

```python
from semantica.provenance import ProvenanceManager
from semantica.provenance.schemas import SourceReference

prov = ProvenanceManager(storage_path="intel_provenance.db")

# 第 1 层：原始采集
prov.track_chunk(
    chunk_id="osint_collection_20260621_0442Z",
    source_document="COLLECTION_TASKING_TK-2026-0192",
    source_path="/osint/raw/20260621_0442Z.txt",
    start_index=0,
    end_index=2048,
    classification="UNCLASSIFIED//FOUO",
    collection_method="OSINT",
    collector_id="STATION_ECHO",
)

# 第 2 层：从采集中抽取的实体
prov.track_entity(
    entity_id="threat_actor_DELTA9",
    source="osint_collection_20260621_0442Z",
    metadata={"label": "THREAT_ACTOR", "confidence_level": "C2"},
    confidence=0.87,
    entity_type="threat_actor",
    activity_id="ner_extraction",
    source_location="paragraph_3",
)

# 第 3 层：来自全源融合的行动关系
prov.track_relationship(
    relationship_id="DELTA9_operates_CAMPAIGN_IRON",
    source="FUSION_REPORT_FP-2026-0447",
    metadata={"type": "operates", "confidence": 0.81},
    confidence=0.81,
    activity_id="all_source_fusion",
)

# 第 4 层：来自两个独立情报来源的属性
humint_src = SourceReference(
    document="HUMINT_REPORT_HR-2026-0821",
    confidence=0.91,
    metadata={"classification": "SECRET", "source_country": "PARTNER_5EYES"},
)
imint_src = SourceReference(
    document="IMINT_PRODUCT_IP-2026-1104",
    page=3,
    section="Ground Truth Assessment",
    confidence=0.87,
    metadata={"sensor": "OPIR", "resolution_m": 0.3},
)
prov.track_property_source("DELTA9", "location_country", "COUNTRY_X", humint_src)
prov.track_property_source("DELTA9", "location_country", "COUNTRY_X", imint_src)

# 成品审计——完整监管链
lineage = prov.get_lineage("threat_actor_DELTA9")
print("Chain of custody:")
for entry in lineage["lineage_chain"]:
    print(f"  [{entry['timestamp'][:19]}]  {entry['activity_id']}  agent={entry['agent_id']}")

# 位置评估的佐证情报来源
sources = prov.get_all_sources("DELTA9_location_country")
for s in sources:
    print(f"  INT source: {s['source']}  (conf={s['confidence']:.2f})")
```

</Tab>

<Tab title="安全 — SOC/事件">

SOC 威胁情报平台追踪每个 CVE 从首次 NVD 摄取到富化运行、CVSS 更新和分析师标注的全过程。溯源链回答了事件响应中最重要的问题："这条漏洞记录是否是最新的？自上次检查以来 CVSS 评分是否被修订过？"

通过对同一 `entity_id` 重复调用 `track_entity()` 创建的版本链为你提供完整的更新历史。如果 NVD 在概念验证发布后将评分从 7.5 修订为 9.8，链会显示该修订的确切时间戳以及写入更新值的流水线。

```python
from semantica.provenance import ProvenanceManager
from semantica.provenance.schemas import SourceReference
from semantica.provenance.integrity import compute_checksum

prov = ProvenanceManager(storage_path="soc_provenance.db")

# 初始 NVD 摄取
prov.track_entity(
    entity_id="cve-2023-44487",
    source="NVD_feed_2023-10-10",
    metadata={"cvss_score": 7.5, "description": "HTTP/2 Rapid Reset DDoS"},
    confidence=0.98,
    entity_type="vulnerability",
    activity_id="nvd_feed_ingestion",
)

# 六周后：NVD 在 PoC 发布后修订了评分
prov.track_entity(
    entity_id="cve-2023-44487",
    source="NVD_feed_2023-11-22",
    metadata={"cvss_score": 7.5, "kev_added": True, "known_exploited": True},
    confidence=0.98,
    entity_type="vulnerability",
    activity_id="nvd_feed_update",
)

# 将 CISA KEV 添加作为单独的属性来源追踪
kev_ref = SourceReference(
    document="CISA_KEV_catalog_2023-11-22",
    section="Known Exploited Vulnerabilities",
    confidence=1.0,
    metadata={"publisher": "CISA", "mandatory_remediation": True},
)
prov.track_property_source("cve-2023-44487", "known_exploited", True, kev_ref)

# 事件响应查询：此 CVE 的完整历史
lineage = prov.get_lineage("cve-2023-44487")
print(f"CVE first seen  : {lineage['first_seen'][:10]}")
print(f"CVE last updated: {lineage['last_updated'][:10]}")
print(f"Update count    : {lineage['entity_count']} records")

# 在依赖该记录做出补丁决策之前的完整性检查
entries = prov.trace_lineage("cve-2023-44487")
for e in entries:
    ok = compute_checksum(e) == e.checksum
    print(f"  [{('OK' if ok else 'TAMPERED')}] {e.timestamp[:19]}  source={e.source_document}")
```

</Tab>

<Tab title="生命科学 — 临床/制药">

临床证据图谱追踪从来源出版物到结构化抽取再到监管提交的疗效数据。W3C PROV-O 链满足 ICH E6(R2) GCP 对数据可追溯性的要求以及 21 CFR Part 11 对电子记录完整性的要求。提交包中的每个实体都必须可追溯到源文档、抽取活动和操作员。

`track_property_source()` 在此处尤为重要：当两项独立研究对同一终点报告不同的疗效值时，每项都必须单独追踪并附完整引用元数据。由此产生的多来源记录是荟萃分析的证据基础，而溯源条目成为监管档案中的支持文档。

```python
from semantica.provenance import ProvenanceManager
from semantica.provenance.schemas import SourceReference
from semantica.provenance.integrity import compute_checksum

prov = ProvenanceManager(storage_path="clinical_provenance.db")

# 追踪源文档分块（III 期研究报告章节）
prov.track_chunk(
    chunk_id="phase3_primary_endpoint_C4591001",
    source_document="STUDY_REPORT_BNT162b2_C4591001_MOD2",
    source_path="/submissions/EMA_rolling_review/mod2_clinical_overview.pdf",
    start_index=4096,
    end_index=6144,
    study_phase="Phase_III",
    ctgov="NCT04368728",
    sponsor="BioNTech_Pfizer",
)

# 追踪抽取的疗效实体，附逐字来源引用
prov.track_entity(
    entity_id="VE_primary_BNT162b2_C4591001",
    source="phase3_primary_endpoint_C4591001",
    metadata={"value": "95.0%", "CI_95": "[90.3–97.6]", "label": "EFFICACY_MEASURE"},
    confidence=0.99,
    entity_type="clinical_endpoint",
    activity_id="structured_data_extraction",
    source_quote="Vaccine efficacy against COVID-19 was 95.0% (95% CI, 90.3–97.6)",
)

# 用于荟萃分析的多研究属性追踪
study1 = SourceReference(
    document="NEJM_doi_10.1056_NEJMoa2034577",
    page=9, section="Table 2",
    confidence=0.99,
    metadata={"study_id": "C4591001", "n_participants": 43448},
)
study2 = SourceReference(
    document="Lancet_doi_10.1016_S0140-6736_21_00448-7",
    page=6, section="Results",
    confidence=0.97,
    metadata={"study_id": "EXT_COHORT", "n_participants": 9119},
)
prov.track_property_source("BNT162b2", "vaccine_efficacy_symptomatic_covid19", "95.0", study1)
prov.track_property_source("BNT162b2", "vaccine_efficacy_symptomatic_covid19", "94.1", study2)

# 监管提交审计——证据链
lineage = prov.get_lineage("VE_primary_BNT162b2_C4591001")
print("Evidence chain for regulatory submission:")
for entry in lineage["lineage_chain"]:
    print(f"  {entry['timestamp'][:10]} | {entry['source_document'][:45]} | "
          f"agent={entry['agent_id']}")

# 完整性验证（21 CFR Part 11 要求）
entries = prov.trace_lineage("VE_primary_BNT162b2_C4591001")
for e in entries:
    status = "OK" if compute_checksum(e) == e.checksum else "TAMPERED"
    print(f"  Integrity [{status}]: {e.entity_id}")

stats = prov.get_statistics()
print(f"\nTotal evidence records in dossier: {stats['total_entries']}")
```

</Tab>

<Tab title="银行 — 风险/合规">

按揭发放系统为每个信贷决策记录完整溯源，满足 SR 11-7 模型风险管理指引和 EBA 模型文档要求。承保模型中使用的每个特征——信用分、DTI 比率、房产估值——都必须可追溯到其源数据拉取、运行模型的操作员以及决策的时间戳。

房产估值使用 `track_property_source()` 从两个来源（RICS 评估和 AVM 估计）追踪，为合规团队提供 LTV 如何计算的完整图景，以及模型最终依赖了哪种估值方法。

```python
from semantica.provenance import ProvenanceManager
from semantica.provenance.schemas import SourceReference

prov = ProvenanceManager(storage_path="credit_provenance.db")
app_id = "APP-2026-994421"

# 追踪征信局数据拉取
prov.track_chunk(
    chunk_id=f"{app_id}_bureau_pull",
    source_document=f"EXPERIAN_CREDITEXPERT_{app_id}_2026-06-21",
    source_path=f"/bureau/experian/{app_id}/2026-06-21.json",
    start_index=0,
    end_index=4096,
    bureau="Experian",
    pull_timestamp="2026-06-21T09:14:32Z",
    consent_ref=f"CONSENT-{app_id}",
)

# 批量追踪抽取的信用特征
features = [
    {"id": f"{app_id}_credit_score",         "confidence": 1.0, "value": 714},
    {"id": f"{app_id}_dti_ratio",             "confidence": 1.0, "value": 0.38},
    {"id": f"{app_id}_derogatory_count_7yr",  "confidence": 1.0, "value": 0},
]
prov.track_entities_batch(
    features,
    source=f"{app_id}_bureau_pull",
    entity_type="credit_feature",
    activity_id="bureau_parsing",
    agent_id="credit_data_service_v2",
)

# 追踪竞争性房产估值
rics_val = SourceReference(
    document=f"RICS_VALUATION_{app_id}_2026-06-18",
    page=3, section="Market Value Assessment",
    confidence=0.96,
    metadata={"valuer": "JLL_Residential", "method": "comparable_sales"},
)
avm_val = SourceReference(
    document=f"AVM_ESTIMATE_{app_id}_2026-06-21",
    confidence=0.84,
    metadata={"provider": "Hometrack_AVM", "model_version": "v8.3"},
)
prov.track_property_source(app_id, "property_value_GBP", "410000", rics_val)
prov.track_property_source(app_id, "property_value_GBP", "403500", avm_val)

# 追踪承保决策本身
prov.track_entity(
    entity_id=f"DECISION_{app_id}",
    source=f"{app_id}_bureau_pull",
    metadata={"outcome": "approved_conditional_lmi", "model": "underwriting_model_v4"},
    confidence=0.89,
    entity_type="credit_decision",
    activity_id="automated_underwriting",
)

# SR 11-7 审计输出
print("=== MODEL AUDIT TRAIL ===")
lineage = prov.get_lineage(f"DECISION_{app_id}")
for entry in lineage["lineage_chain"]:
    print(f"  [{entry['timestamp'][:19]}]  {entry['activity_id']}  agent={entry['agent_id']}")

print("\n=== PROPERTY VALUATION SOURCES ===")
for s in prov.get_all_sources(f"{app_id}_property_value_GBP"):
    print(f"  {s['source'][:55]}  conf={s['confidence']:.2f}")

stats = prov.get_statistics()
print(f"\nTotal audit records: {stats['total_entries']}  |  Sources: {stats['unique_sources']}")
```

</Tab>

</Tabs>

<a id="the-w3c-prov-o-mapping"></a>
## W3C PROV-O 映射

每个 `ProvenanceEntry` 直接映射到 W3C PROV-O 术语。如果你的合规团队或监管机构需要 PROV-O 导出，字段映射是一对一的：

| PROV-O 术语 | `ProvenanceEntry` 字段 | 记录的内容 |
| :--- | :--- | :--- |
| `prov:Entity` | `entity_id` | 被追踪的对象——实体、分块、关系或属性 |
| `prov:Activity` | `activity_id` | 产生它的过程——`"ner_extraction"`、`"bureau_parsing"` |
| `prov:Agent` / `prov:Person` / `prov:SoftwareAgent` / `prov:Organization` | `agent_id`, `agent_type`, `is_automated` | 谁——或什么——运行了该活动，以及是否有人类直接承担责任 |
| `prov:qualifiedAssociation` + `prov:hadRole` | `role` | 智能体在此特定实体中的角色——`"generator"`（默认）、`"approver"`、`"reviewer"`——用于签批/四人眼工作流 |
| `prov:wasDerivedFrom` | `parent_entity_id`（旧版组合字段） | 此实体的先前版本或来源 |
| — | `previous_version_id` | 此条目更正/替换同一事实的先前版本 |
| `prov:wasDerivedFrom` | `derived_from_id` | 此条目派生自不同的来源实体 |
| `prov:used` | `used_entities` | 为产生此实体而消费的实体 ID |
| `prov:generatedAtTime` | `timestamp` | ISO 日期时间，写入时自动设为 `datetime.utcnow()` |
| `prov:qualifiedInvalidation` | `invalidated`, `invalidated_at_time`, `invalidated_by`, `invalidation_reason` | 通过 `ProvenanceManager.invalidate()` 记录为墓碑的撤回/更正，从不硬删除 |
| `prov:startedAtTime` / `prov:endedAtTime` | `activity_started_at_time`, `activity_ended_at_time` | 类型化活动时间——通过 `activity=` 关键字参数传入 `ActivityRecord` 来连同 `activity_id` 一起设置 |
| `prov:qualifiedGeneration`/`Generation`, `qualifiedUsage`/`Usage`, `qualifiedDerivation`/`Derivation` | （从上述字段派生） | `wasGeneratedBy`/`used`/`wasDerivedFrom` 的附加限定形式，随普通三元组自动输出 |
| `prov:wasAssociatedWith` | （从 `agent_id` 派生） | 直接的 Activity→Agent 链接，区别于 Entity→Agent 的 `wasAttributedTo` |
| `prov:actedOnBehalfOf` | `acted_on_behalf_of` | Agent→Agent 委托——例如，自动智能体代表授权它的人类/组织行事 |
| `prov:wasInformedBy` | `informed_by_activities`（作为 `informed_by=[...]` 传入） | 将此条目的活动链接到它所依据的先前活动（例如，一个流水线阶段依据其前一阶段） |
| `prov:Bundle` + `prov:hadMember` | `bundle_id` | 按来源/数据集/摄取运行分组条目（成员三元组，非真正的 RDF 命名图分区） |
| — | `valid_from`, `valid_until`, `revision_type`, `supersedes` | 从已弃用的 `kg.ProvenanceTracker` 合并的双时态字段——始终由调用方提供（从不自动计算），通过 `ProvenanceManager.revision_history()` 呈现，对未显式设置的条目回退到基于时间戳的推导 |

`previous_version_id` 和 `derived_from_id` 与 `parent_entity_id` 并列叠加——读取 `parent_entity_id` 的现有代码保持不变地继续工作，而新代码则获得消歧后的两种关系。

`checksum` 字段不是 PROV-O 标准的一部分——它是 Semantica 的篡改检测扩展。每个条目的 SHA-256 现在还包含 `previous_checksum`（先前条目的校验和，按 `sequence_id` 的插入顺序），将每个条目链接到前一个。`ProvenanceManager.verify_chain()` 遍历整个链并报告任何中断——包括从底层表中硬删除的行，这是单行校验和无法自行检测的。

注意：上面的银行示例将 `agent_id="credit_data_service_v2"` 传给 `track_entities_batch()`——现在这确实会填充条目的 `agent_id` 字段（此前的一个 bug 导致批级别的类型化关键字参数如 `agent_id`/`entity_type`/`activity_id` 被静默吸收到不透明的 `metadata` 块中）。

`export_prov()` 在 `ProvenanceManager.DEFAULT_BASE_URI` 下铸造实体/智能体/活动 URI（默认为 `https://semantica.dev/ns#`——`RDFExporter` 的 `NamespaceManager` 为其 `"semantica"` 前缀使用的同一命名空间，因此 KG 导出和 PROV 导出中同一 `entity_id` 的 URI 可互解析），除非通过 `export_prov(base_uri=...)` 或 CLI 的 `--base-uri` 选项覆盖。

<a id="related-guides"></a>
## 相关指南

- [语义抽取](semantic-extraction.zh-CN.md) —— 为每个抽取实体自动生成溯源条目的 NER 和关系抽取流水线
- [冲突解决](conflict-resolution.zh-CN.md) —— 溯源属性来源直接馈入冲突检测；每个解析值可追溯到其来源
- [去重](deduplication.zh-CN.md) —— 合并操作记录在合并历史中；与溯源配对实现从来源到规范实体的完整血缘
- [溯源参考](../reference/provenance.zh-CN.md) —— 完整的存储后端 API、`InMemoryStorage`、`SQLiteStorage` 和 `ProvenanceEntry` 模式
