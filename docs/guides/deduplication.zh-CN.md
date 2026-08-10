---
title: "去重与实体合并"
description: "使用多因子相似度检测重复实体，通过可配置的策略合并它们，并在大规模场景下保持知识图谱的整洁。"
---

**[English](deduplication.md)** · **简体中文（当前）**

<a id="what-is-deduplication"></a>
## 什么是去重？

去重是识别指向同一现实世界对象但在数据中表现为不同记录的实体，然后将它们合并为单一规范表示的过程。此过程解决当数据来自多个来源时出现的别名、拼写差异和格式差异。

**关键去重概念：**

**规范实体（Canonical entities）** 是合并所有重复记录后某一现实世界对象的单一权威表示。规范实体成为知识图谱中所有关系指向的节点。

**别名（Aliases）** 是同一实体的替代名称或标识符。例如，"APT29"、"Cozy Bear" 和 "Midnight Blizzard" 都是同一威胁行为者的别名。

**实体解析（Entity resolution）** 是确定不同记录何时指向同一实体的更广泛过程，包括相似度计算、重复检测和合并步骤。

**相似度算法：**
- **Jaro-Winkler** 衡量字符串相似度，对共享前缀给予更高分数，非常适合具有相同开头的名称
- **Levenshtein** 距离计算将一个字符串转换为另一个字符串所需的字符编辑次数，适合捕捉拼写错误和变体

**聚类（Clustering）** 使用 Union-Find 等算法将相关的重复项组合在一起，确保如果 A 匹配 B 且 B 匹配 C，即使 A 和 C 不直接匹配，三者也被组合在一起。

<a id="why-use-deduplication"></a>
## 为什么使用去重？

**数据质量和一致性。** 消除因同一实体不同名称而碎片化关系并导致查询结果不一致的重复节点。

**准确的分析和指标。** 当实体不会因命名差异而被人为地分散到多个节点时，可获得正确的计数、中心性度量和关系分析。

**关系整合。** 将分散的关系合并到单一规范实体上，使得能够完整分析在碎片化数据中会被遗漏的连接和模式。

**来源整合。** 无缝合并来自多个订阅源、系统和数据库的数据，其中相同实体以不同标识符和命名约定出现。

**图谱效率。** 通过消除冗余节点同时通过适当的合并策略保留所有信息，减少图谱大小并提高查询性能。

**溯源保留。** 维护完整的审计轨迹，显示哪个来源为最终规范实体贡献了每条信息。

<a id="when-to-use-when-not-to-use"></a>
## 何时使用 / 何时不使用

**在以下情况使用去重：**
- 多来源数据整合，实体以不同名称或标识符出现
- 易出现别名和变体的实体类型（组织、人员、产品、地理位置）
- 关系准确性依赖于实体整合的知识图谱
- 需要规范实体管理的数据质量工作流
- 需要准确实体计数和关系指标的分析
- 同一现实世界对象出现在多个系统或数据库中的场景

**在以下情况不要使用去重：**
- 具有一致实体标识符和命名约定的单来源数据
- 去重延迟不可接受的高吞吐量流式场景
- 具有可靠主键且设计上不可能出现重复的数据
- 实体变体应保留为单独节点的情况（不同产品版本、基于时间的实体状态）
- 基本数据库约束已处理唯一性的简单精确匹配场景

**在以下情况需谨慎：**
- O(n²) 成对比较在计算上变得昂贵的大型数据集
- 当存在确定性主键（LEI、CVE-ID、ISIN）时使用模糊匹配
- 可能合并真正不同实体的极低相似度阈值

<a id="typical-workflow"></a>
## 典型工作流

去重工作流遵循从检测到合并的系统化过程：

**1. 检测（Detect）** → 使用 `detect_duplicates()` 或 `DuplicateDetector` 通过多因子相似度评分识别潜在匹配

**2. 分组（Group）** → 应用聚类算法将传递相关的重复项收集到组中（A 匹配 B，B 匹配 C → 分组 A、B、C）

**3. 选择规范实体（Select Canonical）** → 根据完整性、来源权威性或置信度分数为每个组选择代表性实体

**4. 合并（Merge）** → 使用 `keep_most_complete` 或 `merge_all` 等策略合并重复实体，同时保留溯源

**5. 验证（Validate）** → 审查合并结果，并根据精确率/召回率分析调整阈值或策略

**6. 更新图谱（Update Graph）** → 用规范实体替换重复节点并转移所有关系

此流水线将碎片化的多源数据转化为整洁、整合的知识图谱，准备好进行分析和推理。

<a id="api-patterns-functional-vs-class-based"></a>
## API 模式：函数式 vs 基于类

Semantica 为不同的使用场景提供了简单的函数式封装和全面的类 API：

**用于简单工作流的函数式封装：**
- `detect_duplicates()` — 以最小配置进行一次性重复检测
- `calculate_similarity()` — 比较两个实体并提供详细的相似度分解
- `merge_entities()` — 围绕 merge_duplicates() 的便捷封装，用于快速合并

**用于复杂工作流的类 API：**
- `DuplicateDetector` — 可配置的重复检测，支持聚类、增量处理和高级相似度选项
- `EntityMerger` — 复杂的合并，支持多种策略、溯源追踪和合并历史

**使用指南：**
- 当你有一组原始实体并需要自动重复检测时使用 `merge_duplicates()`
- 当你已经知道哪些实体是重复项并只需合并一个预先确定的组时使用 `merge_entity_group()`
- 不要在同一工作流中混用函数式封装和类 API——选择一种方法并在整个工作流中保持一致

去重模块使用六种互补的相似度算法——精确匹配、Levenshtein、Jaro-Winkler、余弦相似度、属性比较和向量嵌入——跨多源知识图谱检测重复实体，然后在保留完整溯源的同时将它们合并为单一规范实体。利用它在运行图谱分析或冲突解决之前折叠别名集群（例如 "APT29"、"Cozy Bear"、"Midnight Blizzard"）。

<Info>
在摄取之后、冲突解决之前运行去重。去重将重复节点合并为一个规范实体。冲突解决随后调和该规范实体上不一致的属性值。流水线顺序很重要：先去重，再解决冲突，最后用 SHACL 验证。
</Info>

<a id="finding-your-duplicates-the-first-scan"></a>
## 查找重复项：首次扫描

对于较小数据集上的直接重复检测，从 `detect_duplicates()` 开始。将其指向你的实体，让成对算法使用多种相似度信号比较每一对。

**扩展性考量：** 对于几千个节点的数据集，这在几秒内即可完成。O(n²) 的成对比较成本仅在一万个以上实体时才会变得棘手——对于更大的集合，请参阅下文的聚类部分。

```python
from semantica.deduplication import detect_duplicates

# 从四个不同 CTI 订阅源摄取的威胁行为者——相同的行动者，不同的名称
threat_actors = [
    {"id": "ta-nvd-001",  "name": "APT29",            "type": "ThreatActor",
     "country": "Russia",  "source": "MISP"},
    {"id": "ta-of-002",   "name": "APT-29",            "type": "ThreatActor",
     "country": "Russia",  "source": "OpenCTI"},
    {"id": "ta-rf-003",   "name": "Cozy Bear",         "type": "ThreatActor",
     "aliases": ["APT29"], "country": "Russia", "source": "RecordedFuture"},
    {"id": "ta-sx-004",   "name": "The Dukes",         "type": "ThreatActor",
     "aliases": ["APT29", "Cozy Bear"], "country": "Russia", "source": "STIX-partner"},
    {"id": "ta-ms-005",   "name": "Midnight Blizzard", "type": "ThreatActor",
     "aliases": ["NOBELIUM", "APT29"], "country": "Russia", "source": "Microsoft"},
    {"id": "ta-ap-006",   "name": "APT28",             "type": "ThreatActor",
     "country": "Russia",  "source": "MISP"},  # 不同的行为者——不应匹配
]

candidates = detect_duplicates(
    threat_actors,
    method="pairwise",          # 比较每一对
    similarity_threshold=0.6,   # 标记为候选的最低分数
    confidence_threshold=0.5,   # 匹配的最低置信度
)

for c in candidates:
    print(f"{c.entity1['name']!r}  ~  {c.entity2['name']!r}")
    print(f"  similarity={c.similarity_score:.2f}  confidence={c.confidence:.2f}")
    print(f"  signals: {c.reasons}")
    print()
```

```text
'APT29'  ~  'APT-29'
  similarity=0.89  confidence=0.81
  signals: ['levenshtein', 'jaro_winkler']

'APT29'  ~  'Cozy Bear'
  similarity=0.63  confidence=0.71
  signals: ['property', 'relationship']   # alias field matched

'Cozy Bear'  ~  'The Dukes'
  similarity=0.61  confidence=0.68
  signals: ['property']                   # shared alias "APT29"

'APT29'  ~  'Midnight Blizzard'
  similarity=0.65  confidence=0.73
  signals: ['property']                   # alias "APT29" in Midnight Blizzard record
```

分数讲述了一个清晰的故事。"APT29" 和 "APT-29" 得分 0.89——唯一的区别是连字符，产生了强烈的字符串相似度信号。"Cozy Bear" 和 "The Dukes" 得分较低（0.61），因为名称完全不同，但属性信号触发了，因为两条记录的别名列表中都包含 `"APT29"`。"APT28" 从未出现在结果中，因为它只共享国家字段——不足以跨过 0.6 的阈值。

<a id="understanding-the-candidate-object"></a>
## 理解候选对象

每个 `DuplicateCandidate` 携带两个实体、它们的相似度分数，以及哪些相似度算法促成了匹配的详细分解。这为审计和阈值调优提供了完全透明度：

```python
from semantica.deduplication import calculate_similarity

# 在执行合并之前详细检查单个对
e1 = threat_actors[0]   # APT29
e2 = threat_actors[2]   # Cozy Bear

result = calculate_similarity(e1, e2, method="multi_factor")

print(f"Overall score : {result.score:.2f}")
print(f"Method        : {result.method}")
print(f"Components    :")
for algo, score in result.components.items():
    print(f"  {algo:<20} {score:.2f}")
```

```text
Overall score : 0.63
Method        : multi_factor
Components    :
  exact                0.00   # names are completely different strings
  levenshtein          0.12   # high edit distance between "APT29" and "Cozy Bear"
  jaro_winkler         0.21   # no shared prefix
  property             0.94   # alias match is nearly definitive
  relationship         0.71   # share relationships to overlapping malware families
  embedding            0.78   # semantic vectors land in the same cluster
```

属性组件（0.94）在这里承担了大部分工作。"Cozy Bear" 的记录包含 `aliases: ["APT29"]`，这创建了几乎确定的信号，表明这些实体指向同一威胁行为者。当你看到这样的模式——弱的名称相似度但强的属性匹配——你通常看到的是真正的别名关系，而非误报。

<a id="grouping-duplicates-before-merging"></a>
## 合并前对重复项分组

对于小型数据集，你可以直接合并候选对。对于较大的图谱，同一实体可能在十二个订阅源中以六个不同的名称出现，请使用 Union-Find 聚类的重复分组。这确保如果 A 匹配 B 且 B 匹配 C，即使 A 和 C 不直接满足相似度阈值，三个实体也被组合在一起：

```python
from semantica.deduplication import DuplicateDetector, EntityMerger

detector = DuplicateDetector(
    similarity_threshold=0.6,
    confidence_threshold=0.5,
    use_clustering=True,      # 启用 Union-Find 分组
)

groups = detector.detect_duplicate_groups(threat_actors)

print(f"Found {len(groups)} duplicate groups:")
for group in groups:
    names = [e["name"] for e in group.entities]
    print(f"  Group (confidence={group.confidence:.2f}): {names}")
    if group.representative:
        print(f"  Representative: {group.representative['name']!r}")
```

```text
Found 2 duplicate groups:
  Group (confidence=0.74): ['APT29', 'APT-29', 'Cozy Bear', 'The Dukes', 'Midnight Blizzard']
  Representative: 'APT29'   # highest completeness score in the group

  Group (confidence=0.81): ['APT28', 'Fancy Bear']
  Representative: 'APT28'
```

组结果清晰地展示了整合：五个应为一个规范实体的独立节点。`representative` 字段标识合并器将用作基础的实体——通常是属性集最完整的那个，在此例中是来自 MISP 订阅源的 "APT29"。

<a id="merging-collapsing-the-group-without-losing-data"></a>
## 合并：在不丢失数据的情况下折叠组

一旦你识别了重复组，合并过程将它们整合为规范实体。`keep_most_complete` 策略选择属性数最多的实体作为规范节点，并用其他来源的缺失字段来丰富它：

```python
merger = EntityMerger(preserve_provenance=True)

for group in groups:
    if len(group.entities) < 2:
        continue

    # merge_entity_group() 跳过重复检测，因为 `group.entities`
    # 已经是 detect_duplicate_groups() 确认的组
    op = merger.merge_entity_group(group.entities, strategy="keep_most_complete")

    canonical = op.merged_entity
    source_ids = [e["id"] for e in op.source_entities]
    print(f"Merged {len(op.source_entities)} entities → canonical: {canonical['name']!r}")
    print(f"  Source IDs retired : {source_ids}")
    print(f"  Merge strategy     : {op.merge_result.metadata.get('strategy')}")
```

```text
Merged 5 entities → canonical: 'APT29'
  Source IDs retired : ['ta-nvd-001', 'ta-of-002', 'ta-rf-003', 'ta-sx-004', 'ta-ms-005']
  Merge strategy     : keep_most_complete
```

五个源实体被一个规范表示替换。这五个节点携带的每条关系——到活动、恶意软件家族、TTP、基础设施——现在都附加到规范 "APT29" 节点上。合并在消除冗余的同时保留了所有信息，溯源记录准确显示了哪个订阅源贡献了每个属性。

<a id="reviewing-merge-history-for-audit"></a>
## 审查合并历史以供审计

在批量合并操作之后，你可以检索完整历史来审查所做的每一个决策。此审计轨迹对于理解合并决策和向利益相关者解释至关重要：

```python
history = merger.get_merge_history()

print(f"Total merge operations: {len(history)}")
for op in history:
    print(f"  {op.merged_entity['name']!r} ← {len(op.source_entities)} sources")
    print(f"    strategy: {op.merge_result.metadata.get('strategy')}")
```

此历史提供了合并决策的完全透明度。当订阅源所有者询问为什么他们的实体被合并到另一个实体时，你有该决策的文档化证据和理由。

<a id="streaming-ingestion-incremental-deduplication"></a>
## 流式摄取：增量去重

当你的流水线处理连续的数据流——新的威胁情报每小时到达——你不想在每个批次上对整个图谱重新运行成对比较。使用增量检测仅将新实体与现有规范集进行比较：

```python
# 现有图谱实体（已去重）
existing_actors = [
    {"id": "ta-nvd-001", "name": "APT29", "type": "ThreatActor", "country": "Russia"},
]

# 从新的订阅源到达的新批次
new_batch = [
    {"id": "ta-new-007", "name": "NOBELIUM",      "type": "ThreatActor",
     "aliases": ["APT29"], "country": "Russia", "source": "MSTIC-2026-06"},
    {"id": "ta-new-008", "name": "Scattered Spider", "type": "ThreatActor",
     "country": "Unknown",  "source": "CrowdStrike"},
]

incremental = detector.incremental_detect(new_batch, existing_actors)

print(f"New duplicates found in this batch: {len(incremental)}")
for c in incremental:
    print(f"  {c.entity1['name']!r} matches existing {c.entity2['name']!r}")
    print(f"  score={c.similarity_score:.2f}")
```

```text
New duplicates found in this batch: 1
  'NOBELIUM' matches existing 'APT29'
  score=0.67        # alias field carries "APT29" — property signal fires
```

NOBELIUM 被排队等待与现有 APT29 规范实体合并。Scattered Spider 对每个现有行为者的得分都低于阈值，并作为新的唯一节点添加到图谱中。

<a id="scaling-to-large-entity-sets"></a>
## 扩展到大型实体集

对于一万个以上节点的图谱，由于其 O(n²) 复杂度，成对比较在计算上变得昂贵。使用 `build_clusters()` 运行更高效的向量化批量比较，然后合并每个生成的聚类：

**性能警告：** 始终在代表性数据规模上对相似度操作进行性能分析。在没有适当扩展策略的情况下，适用于 1,000 个实体的方法在 10,000+ 实体时可能变得不可接受地缓慢。

```python
from semantica.deduplication import build_clusters

# 来自 NVD、Vulhub 和供应商订阅源的一万个 CVE 实体
# （未展示——假设 `all_vulns` 是一个包含 1 万+ 字典的列表）

cluster_result = build_clusters(
    all_vulns,
    method="graph_based",        # 基于相似度图谱的 Union-Find
    similarity_threshold=0.75,
)

print(f"Clusters found   : {len(cluster_result.clusters)}")
print(f"Unclustered      : {len(cluster_result.unclustered)}")
print(f"Quality metrics  : {cluster_result.quality_metrics}")

# 合并每个聚类
merger = EntityMerger(preserve_provenance=True)
for cluster in cluster_result.clusters:
    if len(cluster.entities) > 1:
        # 使用 merge_entity_group()，因为聚类已确定这些是重复项
        merger.merge_entity_group(cluster.entities, strategy="keep_most_complete")
```

对于更大的集合，切换到 `method="hierarchical"`，它使用自底向上的凝聚聚类，可扩展到数十万实体，代价是一些精度损失。

<a id="a-simple-example-customer-deduplication"></a>
## 一个简单示例：客户去重

在探索特定领域的案例之前，让我们走查一个简单的客户去重场景。一家公司的 CRM 系统从网络注册、销售团队录入和支持工单中积累了重复的客户记录：

```python
from semantica.deduplication import detect_duplicates, merge_entities

customers = [
    {"id": "cust-001", "name": "John Smith", "email": "john.smith@email.com", 
     "company": "Acme Corp", "source": "web_signup"},
    {"id": "cust-002", "name": "J. Smith", "email": "john.smith@email.com",
     "company": "Acme Corporation", "source": "sales_team"},
    {"id": "cust-003", "name": "John Smith", "phone": "+1-555-0123",
     "company": "Acme Corp", "source": "support_ticket"},
    {"id": "cust-004", "name": "Jane Doe", "email": "jane.doe@email.com",
     "company": "Beta Inc", "source": "web_signup"},
]

# 步骤 1：查找潜在重复项
candidates = detect_duplicates(
    customers,
    method="pairwise",
    similarity_threshold=0.6,  # 需要 60% 相似度
    confidence_threshold=0.5,
)

print("Potential duplicates found:")
for c in candidates:
    print(f"  {c.entity1['name']} ~ {c.entity2['name']} (score: {c.similarity_score:.2f})")
    print(f"    Matching signals: {c.reasons}")

# 预期输出：
# John Smith ~ J. Smith (score: 0.82)
#   Matching signals: ['exact', 'property']  # 相同的邮箱
# John Smith ~ John Smith (score: 0.78)  
#   Matching signals: ['exact', 'property']  # 相同的姓名和公司

# 步骤 2：合并重复项
john_smith_records = [customers[0], customers[1], customers[2]]  # 所有 John Smith 变体
merged_ops = merge_entities(john_smith_records, method="keep_most_complete")

for op in merged_ops:
    canonical = op.merged_entity
    print(f"\nCanonical customer: {canonical['name']}")
    print(f"  Email: {canonical.get('email', 'N/A')}")
    print(f"  Phone: {canonical.get('phone', 'N/A')}")
    print(f"  Company: {canonical['company']}")
    print(f"  Merged from {len(op.source_entities)} records")
    
# 结果：一条 John Smith 记录，包含来自所有三条原始记录的
# 邮箱、电话和公司信息，并具有完整的溯源追踪
```

此示例展示了核心概念：相似度检测找到相关记录，合并将它们整合为保留所有可用信息的规范实体。

<a id="common-pitfalls"></a>
## 常见陷阱

**不做验证就调阈值。** 阈值设置过低会在真正不同的实体之间创建误报合并。在大规模合并操作之前，始终手动审查检测到的重复项样本。

**成对扩展问题。** 比较每个实体对的 O(n²) 成本在 10,000 个实体以上时变得令人望而却步。对于大型数据集，使用聚类方法（`build_clusters`）或切换到向量化相似度。

**当主键存在时使用模糊匹配。** 如果你的实体具有可靠的唯一标识符（LEI 代码、CVE ID、ISBN 号），请对这些字段使用精确匹配，而非计算昂贵的相似度算法。

**不一致地混用封装和类 API。** 不要调用 `detect_duplicates()` 然后手动实例化 `EntityMerger`——选择函数式方法或基于类的方法，并在整个工作流中一致使用。

**忽视合并策略的影响。** `keep_first` 会完全覆盖后续记录，`merge_all` 可能引入冲突值，`keep_most_complete` 可能不尊重来源权威性。选择与你数据质量要求相匹配的策略。

**跳过溯源追踪。** 如果不设置 `preserve_provenance=True`，你将失去对规范实体中每个字段来源的可见性，使审计轨迹成为不可能。

**相似度算法选择不当。** 纯字符串相似度无法处理别名关系（"APT29" vs "Cozy Bear"），而属性匹配对于具有共享属性但身份不同的实体可能过于激进。

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防——CTI/威胁">

一个威胁情报平台从 Mandiant、CrowdStrike、MITRE ATT&CK 和合作伙伴 ISAC 订阅源摄取行为者档案。相同的行为者以供应商特定的名称出现："Cozy Bear"（CrowdStrike）、"APT29"（MITRE）、"Midnight Blizzard"（Microsoft）、"The Dukes"（F-Secure）。在任何分析师查询运行之前，这些别名必须折叠为单一规范节点，使得关系——使用的恶意软件、运营的基础设施、归因的活动——都附加到一处。

别名是这里的关键信号。当任何记录在其 `aliases` 列表中包含规范名称时，基于属性的相似度将强烈触发。设置适中的阈值（0.6）可以捕获仅靠名称相似度会遗漏的基于别名的匹配。

```python
from semantica.deduplication import DuplicateDetector, EntityMerger

actors = [
    {"id": "ta-cs-001",  "name": "Cozy Bear",         "type": "ThreatActor",
     "aliases": ["APT29", "The Dukes"],  "country": "Russia", "source": "CrowdStrike"},
    {"id": "ta-mt-002",  "name": "APT29",              "type": "ThreatActor",
     "aliases": [],                       "country": "Russia", "source": "MITRE"},
    {"id": "ta-ms-003",  "name": "Midnight Blizzard",  "type": "ThreatActor",
     "aliases": ["NOBELIUM", "APT29"],   "country": "Russia", "source": "Microsoft"},
    {"id": "ta-fs-004",  "name": "The Dukes",          "type": "ThreatActor",
     "aliases": ["APT29", "Cozy Bear"],  "country": "Russia", "source": "F-Secure"},
    {"id": "ta-mt-005",  "name": "APT28",              "type": "ThreatActor",
     "aliases": ["Fancy Bear"],           "country": "Russia", "source": "MITRE"},
]

detector = DuplicateDetector(similarity_threshold=0.6, confidence_threshold=0.5)
groups = detector.detect_duplicate_groups(actors)

merger = EntityMerger(preserve_provenance=True)
for group in groups:
    if len(group.entities) < 2:
        continue
    ops = merger.merge_duplicates(group.entities, strategy="keep_most_complete")
    for op in ops:
        feeds = [e["source"] for e in op.source_entities]
        print(f"Canonical: {op.merged_entity['name']!r}  (merged from: {feeds})")
        # 规范实体：'APT29'（合并自：['CrowdStrike', 'MITRE', 'Microsoft', 'F-Secure']）
        # 所有四个行为者节点合并为一个——它们持有的每条关系现在
        # 都附加到规范 APT29 节点上。
```

</Tab>

<Tab title="安全——SOC/事件">

一个 SOC 漏洞管理数据库从 NVD、Vulhub、CISA KEV 和供应商公告中摄取 CVE。同一个 CVE 通常以 "CVE-2024-3400"（来自 NVD）、"PAN-OS GlobalProtect RCE"（来自 Tenable 插件）和 "Critical PAN-OS 0day"（来自博客聚合器）的形式到达。仅靠名称相似度无法匹配这些——`cve_id` 字段中的 CVE ID 才是确定的信号。

```python
from semantica.deduplication import detect_duplicates, merge_entities

vulns = [
    {"id": "v-nvd-001",  "name": "CVE-2024-3400",
     "description": "PAN-OS GlobalProtect command injection RCE",
     "cvss": 10.0, "source": "NVD",      "type": "Vulnerability"},
    {"id": "v-cisa-002", "name": "CVE-2024-3400",
     "description": "Critical PAN-OS vulnerability in active exploitation",
     "cvss": 10.0, "source": "CISA-KEV", "type": "Vulnerability"},
    {"id": "v-ten-003",  "name": "PAN-OS GlobalProtect RCE",
     "cve_id": "CVE-2024-3400",
     "cvss": 10.0, "source": "Tenable",  "type": "Vulnerability"},
    {"id": "v-nvd-004",  "name": "CVE-2023-44487",
     "description": "HTTP/2 Rapid Reset DDoS amplification",
     "cvss": 7.5,  "source": "NVD",      "type": "Vulnerability"},
]

# 低阈值，因为 "CVE-2024-3400" 和
# "PAN-OS GlobalProtect RCE" 之间的名称相似度接近零——我们需要属性信号触发
candidates = detect_duplicates(
    vulns,
    method="pairwise",
    similarity_threshold=0.5,
    confidence_threshold=0.4,
)

for c in candidates:
    print(f"{c.entity1['name']!r}  ~  {c.entity2['name']!r}")
    print(f"  score={c.similarity_score:.2f}  signals={c.reasons}")
    # 'CVE-2024-3400'  ~  'PAN-OS GlobalProtect RCE'
    #   score=0.61  signals=['property', 'exact']   # cve_id 字段匹配

# 合并三个 CVE-2024-3400 变体
cve_group = [v for v in vulns if "3400" in v["name"] or v.get("cve_id") == "CVE-2024-3400"]
ops = merge_entities(cve_group, method="keep_most_complete", preserve_provenance=True)
for op in ops:
    print(f"Canonical CVE: {op.merged_entity['name']}  CVSS={op.merged_entity.get('cvss')}")
    # 规范 CVE：CVE-2024-3400  CVSS=10.0
    # 一个节点现在携带所有三个源记录的溯源。
```

</Tab>

<Tab title="生命科学——临床/制药">

一个临床试验知识图谱从 WHO INN 列表、PubChem、品牌名称数据库和试验注册中心摄取药物实体。华法林以 "warfarin"（INN）、"Coumadin"（品牌，Bristol-Myers Squibb）、"warfarin sodium"（PubChem 化学形式）和 "81-81-2"（CAS 号）的形式出现。CAS 号是基准真相——如果两条记录共享一个 CAS 号，它们就是同一化合物，无论它们携带什么名称。

```python
from semantica.deduplication import DuplicateDetector, EntityMerger, calculate_similarity

compounds = [
    {"id": "c-inn-001", "name": "warfarin",         "type": "Drug",
     "cas": "81-81-2",   "source": "WHO-INN"},
    {"id": "c-bms-002", "name": "Coumadin",         "type": "Drug",
     "cas": "81-81-2",   "source": "BMS-brand"},
    {"id": "c-pc-003",  "name": "warfarin sodium",  "type": "Drug",
     "source": "PubChem"},                            # 无 CAS——较不完整的记录
    {"id": "c-inn-004", "name": "metoprolol",       "type": "Drug",
     "cas": "37350-58-6","source": "WHO-INN"},
    {"id": "c-nov-005", "name": "Lopressor",        "type": "Drug",
     "cas": "37350-58-6","source": "Novartis-brand"},
]

# property 方法对 CAS 号匹配给予极高的评分
sim = calculate_similarity(compounds[0], compounds[1], method="property")
print(f"warfarin vs Coumadin: {sim.score:.2f}")
# warfarin vs Coumadin: 0.91  — CAS 匹配占主导

detector = DuplicateDetector(similarity_threshold=0.6, confidence_threshold=0.5)
groups = detector.detect_duplicate_groups(compounds)

merger = EntityMerger(preserve_provenance=True)
for group in groups:
    if len(group.entities) < 2:
        continue
    ops = merger.merge_duplicates(group.entities, strategy="keep_most_complete")
    for op in ops:
        print(f"Canonical: {op.merged_entity['name']!r}  CAS={op.merged_entity.get('cas')}")
        print(f"  Sources: {[e['source'] for e in op.source_entities]}")
        # 规范实体：'warfarin'  CAS=81-81-2
        #   来源：['WHO-INN', 'BMS-brand', 'PubChem']
        # 所有三条 warfarin 记录合并；WHO-INN 记录作为最完整的记录胜出。
```

</Tab>

<Tab title="银行——风险/合规">

一个风险管理系统在构建交易对手敞口图谱之前，跨 CRM、KYC 和交易系统解析企业客户。法律实体标识符（LEI）是确定的标识符——一个 20 字符的 ISO 17442 代码，在全球范围内唯一标识每个法律实体。如果两条记录共享一个 LEI，它们就是同一法律实体，无论名称格式如何。

三条 BlackRock 记录——"BlackRock Inc."（CRM）、"BlackRock, Inc."（KYC，带逗号）、"Blackrock"（交易系统，小写 b）——必须在信用敞口汇总之前折叠为一个规范交易对手。属性相似度方法对 LEI 匹配的评分接近 1.0，即使在名称相似度中等的情况下也能处理。

```python
from semantica.deduplication import build_clusters, EntityMerger

clients = [
    {"id": "crm-001", "name": "BlackRock Inc.",           "type": "Client",
     "lei": "549300HLPTRASHS0E726", "source": "CRM"},
    {"id": "kyc-001", "name": "BlackRock, Inc.",          "type": "Client",
     "lei": "549300HLPTRASHS0E726", "source": "KYC"},
    {"id": "trd-001", "name": "Blackrock",                "type": "Client",
     "lei": "549300HLPTRASHS0E726", "source": "TradeOps"},
    {"id": "crm-002", "name": "Vanguard Group",           "type": "Client",
     "lei": "549300IH7BVXP9VN3J07", "source": "CRM"},
    {"id": "kyc-002", "name": "The Vanguard Group, Inc.", "type": "Client",
     "lei": "549300IH7BVXP9VN3J07", "source": "KYC"},
]

cluster_result = build_clusters(
    clients,
    method="graph_based",
    similarity_threshold=0.65,   # 仅 LEI 匹配的得分就高于此值
)

merger = EntityMerger(preserve_provenance=True)
for cluster in cluster_result.clusters:
    if len(cluster.entities) < 2:
        continue
    ops = merger.merge_duplicates(cluster.entities, strategy="keep_most_complete")
    for op in ops:
        sources = [e["source"] for e in op.source_entities]
        print(f"Canonical: {op.merged_entity['name']!r}  (from {sources})")
        # 规范实体：'BlackRock Inc.'（来自 ['CRM', 'KYC', 'TradeOps']）
        # 规范实体：'Vanguard Group'（来自 ['CRM', 'KYC']）

print(f"\nBefore: {len(clients)} client records")
print(f"After : {len(cluster_result.clusters)} canonical counterparties")
print(f"Quality: {cluster_result.quality_metrics}")
```

</Tab>

</Tabs>

<a id="choosing-the-right-threshold"></a>
## 选择正确的阈值

相似度阈值控制敏感度。从 0.7 开始，在调整之前检查误报：

- **0.95 及以上** — 仅近乎精确的字符串匹配。适用于名称格式在各订阅源间一致的代码和 ID（LEI、CAS、CVE-ID）。
- **0.80–0.95** — 捕捉排版变体："Apple Inc." vs "Apple, Inc."、"BlackRock" vs "BlackRock Inc."
- **0.65–0.80** — 捕捉缩写和简写形式。对于以长格式和短格式出现的组织名称是必需的。
- **0.50–0.65** — 语义相似度领域。需要属性或向量嵌入信号来弥补弱的名称相似度。当名称可能是完全不同的字符串但指向同一实体时，使用此范围进行基于别名的匹配。

<a id="choosing-a-merge-strategy"></a>
## 选择合并策略

| 策略 | 适用场景 |
| :--- | :--- |
| `keep_most_complete` | 你没有指定的权威来源；最大化信息密度。默认选择。 |
| `keep_first` | 你的第一个来源是黄金记录系统（MDM、LEI 注册表），其数据永远不应被覆盖。 |
| `keep_highest_confidence` | 你的实体带有明确的置信度分数，并且你信任它们作为质量信号。 |
| `merge_all` | 你想要所有属性的超集；当你之后会运行冲突解决来调和分歧时可接受。 |

<a id="related-guides"></a>
## 相关指南

- [摄取任何内容](ingest.zh-CN.md) — 多源摄取创建此模块解决的重复项
- [上下文图谱](context-graphs.zh-CN.md) — 将去重后的实体直接存储在知识图谱中
- [冲突解决](conflict-resolution.zh-CN.md) — 合并后，调和规范实体上不一致的属性值
- [溯源](provenance.zh-CN.md) — 追踪合并谱系，使每个规范实体可追溯到其原始来源
- [流水线](pipeline.zh-CN.md) — 将摄取、去重和存储链接为 `PipelineBuilder` 工作流
