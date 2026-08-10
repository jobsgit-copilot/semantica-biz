---
title: "去重模块"
description: "实体去重：相似度评分、分块、合并和基于聚类的批量处理。"
icon: "copy"
---

**[English](deduplication.md)** · **简体中文（当前）**

**`semantica.deduplication`** 检测并合并跨来源的重复实体，以生成**干净、单一事实来源**的知识图谱：

- 四种 v2 策略比 v1 快达 7 倍：`blocking_v2`、`hybrid_v2`、`semantic_v2`
- `ClusterBuilder` 使用 Union-Find 和层次聚类进行大规模批量去重
- `EntityMerger` 在每个合并实体上保留原始来源溯源
- `MergeStrategyManager` 支持属性级规则和冲突解决
- 所有工作流都在普通 Python 字典上操作：无需 ORM 或模式


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `DuplicateDetector` | 成对和批量检测：返回 `DuplicateCandidate` 或 `DuplicateGroup` 列表 |
| `EntityMerger` | 合并重复组：返回 `List[MergeOperation]` |
| `SimilarityCalculator` | 多因子相似度：字符串、属性、关系和向量嵌入 |
| `ClusterBuilder` | 用于大规模批量去重的 Union-Find 和层次聚类 |
| `MergeStrategy` | 合并策略枚举：`KEEP_FIRST`、`KEEP_LAST`、`KEEP_MOST_COMPLETE`、`KEEP_HIGHEST_CONFIDENCE`、`MERGE_ALL` |
| `PropertyMergeRule` | 包含属性级合并规则的数据类：`{property_name, strategy, conflict_resolution, priority}` |
| `MergeStrategyManager` | 管理和应用命名合并策略；接受属性级规则 |
| `detect_duplicates()` | 便捷函数：`detect_duplicates(entities, method="pairwise", similarity_threshold=0.7)` |
| `merge_entities()` | 便捷函数：`merge_entities(entities, method="keep_most_complete")` |
| `calculate_similarity()` | 便捷函数：`calculate_similarity(entity_a, entity_b, method="multi_factor")` |

<a id="what-you-get"></a>
## 你将获得

- **DuplicateDetector** — 成对、批量、增量和分组检测模式。返回带原因的评分候选者。
- **EntityMerger** — 五种合并策略：保留第一个、最后一个、最完整的、最高置信度，或合并所有字段。
- **SimilarityCalculator** — 跨字符串编辑距离、属性重叠、关系重叠和向量嵌入的多因子评分。
- **ClusterBuilder** — 用于大规模批量去重的 Union-Find 和层次聚类：可处理 10 万+ 实体集。
- **MergeStrategyManager** — 带冲突解决优先级的属性级合并规则。对不同字段应用不同策略。
- **v2 策略** — `blocking_v2`、`hybrid_v2`、`semantic_v2`：对大型实体集比 v1 快达 7 倍。


<a id="getting-started"></a>
## 快速上手

```python
from semantica.deduplication import DuplicateDetector, EntityMerger

entities = [
    {"id": "1", "name": "Apple Inc.",   "type": "Company"},
    {"id": "2", "name": "Apple",        "type": "Company"},
    {"id": "3", "name": "Microsoft",    "type": "Company"},
]

# 1. 检测重复项：返回 List[DuplicateCandidate]
detector   = DuplicateDetector(similarity_threshold=0.7)
candidates = detector.detect_duplicates(entities)

for dup in candidates:
    print(
        "{} vs {}: sim: {:.2f}, confidence: {:.2f}".format(
            dup.entity1.get("name"),
            dup.entity2.get("name"),
            dup.similarity_score,
            dup.confidence,
        )
    )

# 2. 合并重复项：返回 List[MergeOperation]
merger     = EntityMerger()
operations = merger.merge_duplicates(entities, strategy="keep_most_complete")

for op in operations:
    print("Merged {} entities → {}".format(
        len(op.source_entities), op.merged_entity.get("name")
    ))
```

<Tip>
  **在去重之前归一化实体名称。** 规范形式如 `"Apple Inc."` 与 `"apple inc"` 可能仅因大小写差异而评分低于阈值。先运行 `EntityNormalizer` 或 `TextNormalizer` 以获得可靠的匹配。
</Tip>

<a id="duplicatedetector"></a>
## DuplicateDetector

查找重复实体对：

```python
from semantica.deduplication import DuplicateDetector

detector = DuplicateDetector(
    similarity_threshold=0.7,    # 默认 0.7：包含候选者的最低评分
    confidence_threshold=0.6,    # 默认 0.6：包含候选者的最低置信度
    max_results=100,             # 可选：返回候选者总数的硬上限
    top_k_per_entity=3,          # 可选：每个实体的最大候选者数
    min_similarity=0.75,         # 可选：排序后应用的额外下限
    sort_by="confidence",        # "confidence"（默认）| "similarity_score"
)

# 返回 List[DuplicateCandidate]
candidates = detector.detect_duplicates(entities)

for c in candidates:
    print(c.entity1.get("name"), "vs", c.entity2.get("name"))
    print("  similarity_score:", c.similarity_score)
    print("  confidence:      ", c.confidence)
    print("  reasons:         ", c.reasons)

# 使用 union-find 检测重复组（返回 List[DuplicateGroup]）
groups = detector.detect_duplicate_groups(entities)
for g in groups:
    print("Group of {}: confidence: {:.2f}: representative: {}".format(
        len(g.entities), g.confidence, g.representative and g.representative.get("name")
    ))

# 增量检测：将新实体与现有实体比较（返回 List[DuplicateCandidate]）
new_entities = [{"id": "4", "name": "Apple Corp.", "type": "Company"}]
candidates   = detector.incremental_detect(new_entities, entities)
```

<Tip>
  **先调整 `similarity_threshold`，再调整 `confidence_threshold`。** 相似度阈值控制哪些实体对会被考虑。置信度阈值根据多因子评分进一步筛选这些对。从 `similarity_threshold=0.7` 开始，提高它以减少误报。
</Tip>

<Tip>
  **需要合并时使用 `detect_duplicate_groups()`。** `"group"` 检测策略使用 union-find 形成传递聚类：如果 A≈B 且 B≈C，三者都会进入同一组。普通的 `detect_duplicates()` 返回单个对，不具备传递性。
</Tip>

<a id="detect_duplicates-detection-methods"></a>
### `detect_duplicates()` 检测方法

`detect_duplicates()` 便捷函数的 `method=` 参数控制比较的执行方式。
这些与内部使用的 `SimilarityCalculator` 字符串方法无关：

| `method=` | 算法 | 返回 |
| :--------- | :--------- | :------- |
| `"pairwise"`（默认） | O(n²) 全对比较 | `List[DuplicateCandidate]` |
| `"batch"` | 批量相似度计算 | `List[DuplicateCandidate]` |
| `"incremental"` | 新实体与现有实体比较 | `List[DuplicateCandidate]` |
| `"group"` | Union-find 分组形成 | `List[DuplicateGroup]` |

<a id="duplicatecandidate-fields"></a>
### DuplicateCandidate 字段

| 字段 | 类型 | 描述 |
| :----- | :---- | :----------- |
| `entity1` | `Dict` | 第一个实体 |
| `entity2` | `Dict` | 第二个实体 |
| `similarity_score` | `float` | 相似度评分（0–1） |
| `confidence` | `float` | 置信度评分（0–1） |
| `reasons` | `List[str]` | 为何被视为重复 |
| `metadata` | `Dict` | 额外元数据 |

<Warning>
  **`DuplicateCandidate` 的字段是 `entity1`、`entity2`、`similarity_score`：不是 `entity_a`、`entity_b`、`similarity`。** 访问错误的字段名会引发 `AttributeError`。
</Warning>

<a id="duplicategroup-fields"></a>
### DuplicateGroup 字段

| 字段 | 类型 | 描述 |
| :----- | :---- | :----------- |
| `entities` | `List[Dict]` | 组中的所有实体 |
| `similarity_scores` | `Dict` | 配对 → 评分映射 |
| `representative` | `Optional[Dict]` | 组中最完整的实体 |
| `confidence` | `float` | 组置信度评分 |
| `metadata` | `Dict` | 额外元数据 |

<a id="entitymerger"></a>
## EntityMerger

将检测到的重复组合并为规范实体：

```python
from semantica.deduplication import EntityMerger, MergeStrategy

# 基本用法
merger     = EntityMerger(preserve_provenance=True)
operations = merger.merge_duplicates(entities, strategy="keep_most_complete")

# operations 是 List[MergeOperation]
for op in operations:
    merged = op.merged_entity         # 合并后的实体字典
    sources = op.source_entities      # 被合并的原始实体列表
    conflicts = op.merge_result.conflicts  # 未解决的冲突（如果有）

# 直接合并已知组（无需重复检测）
pair = [
    {"id": "1", "name": "Apple Inc.", "type": "Company"},
    {"id": "2", "name": "Apple",      "type": "Company"},
]
op = merger.merge_entity_group(pair, strategy="keep_most_complete")
print(op.merged_entity)

# 检索完整合并历史
history = merger.get_merge_history()
print("Total merges performed:", len(history))
```

<Warning>
  **`merge_entities()` 和 `EntityMerger.merge_duplicates()` 返回 `List[MergeOperation]`，不是实体字典列表。** 在每个操作上访问 `.merged_entity` 以获取合并后的字典。
</Warning>

<a id="merge-strategies"></a>
### 合并策略

以字符串形式传入 `merge_duplicates()` 或 `merge_entity_group()` 的 `strategy=`：

| 策略 | 行为 |
| :-------- | :-------- |
| `"keep_first"` | 保留每个重复组中的第一个实体 |
| `"keep_last"` | 保留最近出现的实体 |
| `"keep_most_complete"` | 保留非空属性 + 关系最多的实体 |
| `"keep_highest_confidence"` | 保留 `.confidence` 值最高的实体 |
| `"merge_all"` | 合并所有属性：冲突解决为列表 |

<a id="per-property-merge-rules"></a>
### 属性级合并规则

属性级规则通过 `add_property_rule()` 设置在 `EntityMerger.merge_strategy_manager` 上。
规则接受一个 `MergeStrategy` 枚举值：

```python
from semantica.deduplication import EntityMerger, MergeStrategy

merger = EntityMerger()

# 添加属性级规则
merger.merge_strategy_manager.add_property_rule(
    "name", MergeStrategy.KEEP_FIRST
)
merger.merge_strategy_manager.add_property_rule(
    "aliases", MergeStrategy.MERGE_ALL
)

# 自定义冲突解决函数
def keep_longest(val1, val2):
    return val1 if len(str(val1)) >= len(str(val2)) else val2

merger.merge_strategy_manager.add_property_rule(
    "description", MergeStrategy.KEEP_FIRST, conflict_resolution=keep_longest
)

operations = merger.merge_duplicates(entities)
```

<Warning>
  **`PropertyMergeRule` 是数据类，不是枚举。** 合并策略枚举是 `MergeStrategy`（`KEEP_FIRST`、`KEEP_LAST`、`KEEP_MOST_COMPLETE`、`KEEP_HIGHEST_CONFIDENCE`、`MERGE_ALL`）。属性级规则通过 `merger.merge_strategy_manager.add_property_rule(name, strategy)` 添加。
</Warning>

<a id="mergeoperation-fields"></a>
### MergeOperation 字段

| 字段 | 类型 | 描述 |
| :----- | :---- | :----------- |
| `source_entities` | `List[Dict]` | 被合并的原始实体 |
| `merged_entity` | `Dict` | 合并后的实体 |
| `merge_result` | `MergeResult` | 带冲突的详细结果 |
| `metadata` | `Dict` | 组置信度、相似度评分、使用的策略 |

<a id="similaritycalculator"></a>
## SimilarityCalculator

计算实体对之间的多因子相似度评分：

```python
from semantica.deduplication import SimilarityCalculator

calc = SimilarityCalculator(
    string_weight=0.6,        # 默认 0.6
    property_weight=0.2,      # 默认 0.2
    relationship_weight=0.2,  # 默认 0.2
    embedding_weight=0.0,     # 默认 0.0（仅当 "embedding" 键存在时使用）
    similarity_threshold=0.7,
)

result = calc.calculate_similarity(entity_a, entity_b)
# result 是一个 SimilarityResult
print(result.score)                         # 总体评分 0.0–1.0
print(result.method)                        # 例如 "multi_factor"
print(result.components["string"])          # 字符串相似度组件
print(result.components["property"])        # 属性重叠组件
print(result.components["relationship"])    # 关系 Jaccard 组件
# result.components["embedding"] 仅在提供向量嵌入时存在

# 字符串相似度方法：method= 接受 "levenshtein"、"jaro_winkler"、"cosine"
lev  = calc.calculate_string_similarity("Apple Inc.", "Apple Inc",  method="levenshtein")
jaro = calc.calculate_string_similarity("Steve Jobs", "Steven Jobs", method="jaro_winkler")
cos  = calc.calculate_string_similarity("apple",     "apples",      method="cosine")

# 向量嵌入余弦相似度（向量输入）
emb_score = calc.calculate_embedding_similarity(embedding_a, embedding_b)

# 属性和关系相似度（实体字典输入）
prop_score = calc.calculate_property_similarity(entity_a, entity_b)
rel_score  = calc.calculate_relationship_similarity(entity_a, entity_b)
```

<a id="similarityresult-fields"></a>
### SimilarityResult 字段

| 字段 | 类型 | 描述 |
| :----- | :---- | :----------- |
| `score` | `float` | 总体加权相似度评分（0–1） |
| `method` | `str` | 使用的方法（例如 `"multi_factor"`、`"levenshtein"`） |
| `components` | `Dict[str, float]` | 各组件评分：`"string"`、`"property"`、`"relationship"`、`"embedding"` |
| `metadata` | `Dict` | 使用的权重和可选的评分明细 |

<a id="clusterbuilder"></a>
## ClusterBuilder

为大规模批量去重构建实体聚类：

```python
from semantica.deduplication import ClusterBuilder

builder = ClusterBuilder(
    similarity_threshold=0.8,  # 同一聚类的最低相似度
    min_cluster_size=2,         # 每个有效聚类的最少实体数
    max_cluster_size=100,       # 每个聚类的最大实体数
    use_hierarchical=False,     # True 为层次聚类，False（默认）为 union-find
)
result = builder.build_clusters(entities)

print("Clusters found:", len(result.clusters))
for cluster in result.clusters:
    print("  [{}] {} entities: quality: {:.2f}".format(
        cluster.cluster_id,
        len(cluster.entities),
        cluster.quality_score,
    ))

print("Unclustered:   ", len(result.unclustered))
print("Quality metrics:", result.quality_metrics)
# {"average_size": ..., "average_quality": ..., "total_clusters": ..., "high_quality_clusters": ...}
```

<a id="cluster-fields"></a>
### Cluster 字段

| 字段 | 类型 | 描述 |
| :----- | :---- | :----------- |
| `cluster_id` | `str` | 唯一聚类标识符 |
| `entities` | `List[Dict]` | 聚类中的实体 |
| `centroid` | `Optional[Dict]` | 代表实体（可选） |
| `quality_score` | `float` | 聚类内平均相似度 |
| `metadata` | `Dict` | 相似度评分和其他元数据 |

<a id="convenience-functions"></a>
## 便捷函数

```python
from semantica.deduplication import detect_duplicates, merge_entities, calculate_similarity

# 检测：method= 接受 "pairwise"（默认）、"batch"、"incremental"、"group"
candidates = detect_duplicates(
    entities,
    method="pairwise",
    similarity_threshold=0.8,
    confidence_threshold=0.6,
)

# 合并：method= 接受策略字符串，与 EntityMerger 相同
operations = merge_entities(entities, method="keep_most_complete", preserve_provenance=True)
# 返回 List[MergeOperation]；访问每个的 .merged_entity

# 相似度：method= 接受 "exact"、"levenshtein"、"jaro_winkler"、"cosine"、
#         "property"、"relationship"、"embedding"、"multi_factor"（默认）
result = calculate_similarity(entity_a, entity_b, method="multi_factor")
print(result.score)
```

<a id="custom-similarity-functions"></a>
## 自定义相似度函数

注册领域特定的相似度逻辑并通过方法注册表使用：

```python
from semantica.deduplication import method_registry, SimilarityResult

def drug_name_similarity(entity_a, entity_b, **kwargs):
    """通过活性化合物前缀匹配药品名称。"""
    name_a = entity_a.get("name", "").lower()
    name_b = entity_b.get("name", "").lower()
    score = 1.0 if name_a[:5] == name_b[:5] else 0.0
    return SimilarityResult(score=score, method="drug_name")

method_registry.register("similarity", "drug_name", drug_name_similarity)

# 现在可通过 calculate_similarity 调用
from semantica.deduplication import calculate_similarity
result = calculate_similarity(entity_a, entity_b, method="drug_name")
```

<a id="common-workflows"></a>
## 常见工作流

<Tabs>
  <Tab title="基本去重">
    ```python
    from semantica.deduplication import DuplicateDetector, EntityMerger

    detector   = DuplicateDetector(similarity_threshold=0.8)
    candidates = detector.detect_duplicates(entities)

    print("Duplicate pairs found:", len(candidates))

    merger     = EntityMerger(preserve_provenance=True)
    operations = merger.merge_duplicates(entities, strategy="keep_most_complete")

    merged_entities = [op.merged_entity for op in operations]
    ```
  </Tab>
  <Tab title="基于组的批量处理">
    ```python
    from semantica.deduplication import DuplicateDetector, EntityMerger

    detector = DuplicateDetector(similarity_threshold=0.75)
    # detect_duplicate_groups 内部使用 union-find
    groups   = detector.detect_duplicate_groups(entities)

    merger = EntityMerger()
    for group in groups:
        op = merger.merge_entity_group(group.entities, strategy="keep_most_complete")
        print("Merged into:", op.merged_entity.get("name"))
    ```
  </Tab>
  <Tab title="大规模聚类">
    ```python
    from semantica.deduplication import ClusterBuilder, EntityMerger

    # 先构建聚类：对大型实体集更高效
    builder = ClusterBuilder(similarity_threshold=0.8, min_cluster_size=2)
    result  = builder.build_clusters(entities)

    merger = EntityMerger()
    for cluster in result.clusters:
        op = merger.merge_entity_group(cluster.entities, strategy="keep_most_complete")
        print("Cluster merged into:", op.merged_entity.get("name"))
    ```
  </Tab>
  <Tab title="增量处理">
    ```python
    from semantica.deduplication import EntityMerger

    existing_entities = [...]  # 已经在图中
    new_entities      = [...]  # 在批次中到达

    merger     = EntityMerger()
    operations = merger.incremental_merge(new_entities, existing_entities)

    print("New merges performed:", len(operations))
    ```
  </Tab>
</Tabs>

- [冲突](conflicts.zh-CN.md) — 检测非重复实体之间的值冲突。
- [知识图谱](kg.zh-CN.md) — GraphBuilder 在构建期间使用去重。
- [归一化](normalize.zh-CN.md) — 在去重之前归一化实体名称。
- [溯源](provenance.zh-CN.md) — 追踪合并实体的血缘。
