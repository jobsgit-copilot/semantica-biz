---
title: "时态智能"
description: "双时态事实、时间点快照、Allen 区间代数、时态模式检测和自然语言时态解析，用于时态感知的知识图谱。"
icon: "clock"
---

**[English](temporal.md)** · **简体中文（当前）**

时态智能让你的知识图谱全面理解*何时* — 不仅仅是什么是真的，还有它在现实世界中何时为真、何时被记录，以及事实如何随时间演化。

跨 **v0.3.0**（上下文时态有效性）和 **v0.4.0**（完整时态栈）交付，该系统涵盖五个层次：

<div style={{display:"flex",flexWrap:"wrap",gap:"1.5rem",margin:"1.5rem 0"}}>
  <div style={{flex:"1 1 180px",padding:"1.25rem 1.5rem",borderRadius:"10px",border:"1px solid rgba(16,185,129,0.25)",background:"rgba(16,185,129,0.04)"}}>
    <div style={{fontSize:"1.1rem",fontWeight:700,color:"#10B981",marginBottom:"6px"}}>双时态模型</div>
    <div style={{fontSize:"0.82rem",color:"rgba(255,255,255,0.6)",lineHeight:1.5}}>每条事实都有有效时间 + 事务时间</div>
  </div>
  <div style={{flex:"1 1 180px",padding:"1.25rem 1.5rem",borderRadius:"10px",border:"1px solid rgba(16,185,129,0.25)",background:"rgba(16,185,129,0.04)"}}>
    <div style={{fontSize:"1.1rem",fontWeight:700,color:"#10B981",marginBottom:"6px"}}>时间点查询</div>
    <div style={{fontSize:"0.82rem",color:"rgba(255,255,255,0.6)",lineHeight:1.5}}>一次调用重建任何历史图谱状态</div>
  </div>
  <div style={{flex:"1 1 180px",padding:"1.25rem 1.5rem",borderRadius:"10px",border:"1px solid rgba(16,185,129,0.25)",background:"rgba(16,185,129,0.04)"}}>
    <div style={{fontSize:"1.1rem",fontWeight:700,color:"#10B981",marginBottom:"6px"}}>Allen 区间代数</div>
    <div style={{fontSize:"0.82rem",color:"rgba(255,255,255,0.6)",lineHeight:1.5}}>全部 13 种时态关系，确定性推理</div>
  </div>
  <div style={{flex:"1 1 180px",padding:"1.25rem 1.5rem",borderRadius:"10px",border:"1px solid rgba(16,185,129,0.25)",background:"rgba(16,185,129,0.04)"}}>
    <div style={{fontSize:"1.1rem",fontWeight:700,color:"#10B981",marginBottom:"6px"}}>自然语言时态解析</div>
    <div style={{fontSize:"0.82rem",color:"rgba(255,255,255,0.6)",lineHeight:1.5}}>零 LLM 调用 — 纯正则表达式 + dateutil</div>
  </div>
</div>


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :---- | :---- |
| `BiTemporalFact` | 封装 `valid_from`、`valid_until`、`recorded_at`、`superseded_at` 的数据类。工厂方法：`BiTemporalFact.from_relationship(rel_dict)` |
| `TemporalBound` | 用于开放式区间的枚举哨兵。单值：`TemporalBound.OPEN` |
| `TemporalInterval` | `TemporalReasoningEngine` 使用的冻结数据类 `(start: datetime, end: datetime \| TemporalBound, label?)` |
| `IntervalRelation` | 全部 13 种 Allen 关系标签的枚举（`BEFORE`、`AFTER`、`MEETS` 等） |
| `TemporalGraphQuery` | 时间点快照、范围查询、模式检测、演化分析、时态路径查找 |
| `TemporalPatternDetector` | 在时态边上进行序列和循环模式检测 |
| `TemporalReasoningEngine` | 基于 `TemporalInterval` 对象的 Allen 区间代数 — 纯 Python，确定性 |
| `TemporalNormalizer` | 将自然语言时态表达式解析为 `(datetime, datetime)` 元组 — 零 LLM 调用 |
| `TemporalQueryRewriter` | 从自由文本查询中提取时态意图；返回 `TemporalQueryResult` |
| `TemporalQueryResult` | `TemporalQueryRewriter.rewrite()` 的数据类输出 |
| `TemporalVersionManager` | 创建、列出、比较和应用版本化图谱快照的修订 |


<a id="quick-start"></a>
## 快速上手

<Steps>
  <Step title="构建时态感知图谱">
    在构建时为任何关系附加 `valid_from` / `valid_until`：

    ```python
    from semantica.kg import GraphBuilder

    builder = GraphBuilder()
    kg = builder.build(sources=[{
        "entities": [
            {"id": "alice",     "type": "Person"},
            {"id": "acme_corp", "type": "Organization"},
            {"id": "beta_ltd",  "type": "Organization"},
        ],
        "relationships": [
            {
                "source": "alice", "target": "acme_corp", "type": "ceo_of",
                "valid_from":  "2018-01-01",
                "valid_until": "2022-06-01",
            },
            {
                "source": "alice", "target": "beta_ltd", "type": "ceo_of",
                "valid_from":  "2022-06-01",
                # 没有 valid_until → 开放式（TemporalBound.OPEN）
            },
        ],
    }])
    ```
  </Step>
  <Step title="查询特定时间点的图谱">
    `TemporalGraphQuery` 接受构造函数参数；将图谱传入每次查询调用：

    ```python
    from semantica.kg import TemporalGraphQuery

    query = TemporalGraphQuery(temporal_granularity="day")

    # query_at_time 是主要的公共 API
    result_2020 = query.query_at_time(kg, query="", at_time="2020-06-15")
    result_2023 = query.query_at_time(kg, query="", at_time="2023-01-01")

    print(f"Rels active in 2020: {result_2020['num_relationships']}")
    print(f"Rels active in 2023: {result_2023['num_relationships']}")
    ```
  </Step>
  <Step title="重建特定时间戳的子图谱">
    `reconstruct_at_time()` 是底层原语 — 返回完整的图谱字典，
    仅包含在给定时刻有效的节点和边：

    ```python
    snapshot = query.reconstruct_at_time(kg, "2021-06-15")
    # snapshot 有 "entities" 和 "relationships" 键
    # 可用于所有 GraphAnalyzer、PathFinder、CommunityDetector 调用
    ```
  </Step>
  <Step title="创建版本化快照">
    ```python
    from semantica.kg import TemporalVersionManager

    versioner = TemporalVersionManager()          # 内存存储
    # versioner = TemporalVersionManager(storage_path="versions.db")  # SQLite

    versioner.create_snapshot(
        kg,
        version_label="2024-Q1",
        author="user@example.com",
        description="Q1 2024 snapshot after board restructure",
    )

    for v in versioner.list_versions():
        print(f"{v['label']:12s}  {v['author']}  {v['timestamp']}")
    ```
  </Step>
</Steps>


<a id="the-bi-temporal-model"></a>
## 双时态模型

大多数系统只跟踪一条时间线：某事当前何时为真。双时态图谱同时跟踪**两条独立的时间线**：

<Tabs>
  <Tab title="有效时间">
    *该事实在现实世界中何时为真？*

    - `valid_from` — 事实变为真实的日期
    - `valid_until` — 事实不再真实的日期。对于当前活跃的事实，省略（或使用 `TemporalBound.OPEN`）

    ```python
    from semantica.kg import BiTemporalFact, TemporalBound

    # 从现有关系字典创建
    rel = {
        "source": "alice", "target": "acme_corp", "type": "ceo_of",
        "valid_from":  "2018-01-01",
        "valid_until": "2022-06-01",
    }
    fact = BiTemporalFact.from_relationship(rel)

    print(fact.valid_from)   # datetime(2018, 1, 1, tzinfo=utc)
    print(fact.valid_until)  # datetime(2022, 6, 1, tzinfo=utc)

    # 序列化回字典字段
    fields = fact.to_relationship_fields()
    print(fields["valid_from"])   # "2018-01-01T00:00:00Z"
    print(fields["valid_until"])  # "2022-06-01T00:00:00Z"
    ```
  </Tab>
  <Tab title="事务时间">
    *我们何时在系统中记录了该事实？*

    - `recorded_at` — 在摄取时自动打戳（默认为 `datetime.now(utc)`）
    - `superseded_at` — 当更高版本替换此记录时设置。`TemporalBound.OPEN` 表示仍然当前

    ```python
    rel = {
        "source": "alice", "target": "acme_corp", "type": "ceo_of",
        "valid_from":    "2018-01-01",
        "valid_until":   "2022-06-01",
        "recorded_at":   "2018-01-05T09:32:00Z",
        "superseded_at": None,   # 仍然是当前记录
    }
    fact = BiTemporalFact.from_relationship(rel)

    print(fact.recorded_at)    # datetime(2018, 1, 5, 9, 32, tzinfo=utc)
    print(fact.superseded_at)  # TemporalBound.OPEN
    ```
  </Tab>
  <Tab title="TemporalBound.OPEN">
    `TemporalBound.OPEN` 是表示开放式区间的唯一哨兵 — 没有定义结束日期的事实：

    ```python
    from semantica.kg import TemporalBound

    print(TemporalBound.OPEN)          # TemporalBound.OPEN
    print(TemporalBound.OPEN.value)    # "OPEN"

    # 没有 valid_until 的关系会自动获得 TemporalBound.OPEN
    rel = {"source": "alice", "target": "beta_ltd", "type": "ceo_of",
           "valid_from": "2022-06-01"}
    fact = BiTemporalFact.from_relationship(rel)
    print(fact.valid_until)            # TemporalBound.OPEN
    ```

    <Note>
      `TemporalBound.OPEN` 同时替换起始和结束哨兵 — 只有一个值。推理引擎在比较结束边界时将 `OPEN` 视为 `datetime.max`（遥远未来），在用于 `superseded_at` 时视为 `datetime.min`（遥远过去）。
    </Note>
  </Tab>
</Tabs>


<a id="temporalgraphquery-reference"></a>
## TemporalGraphQuery — 参考

构造一次；图谱传入每个方法调用：

```python
from semantica.kg import TemporalGraphQuery

query = TemporalGraphQuery(
    enable_temporal_reasoning=True,   # 默认值
    temporal_granularity="day",       # second|minute|hour|day|week|month|year
    max_temporal_depth=None,          # 可选的最大深度
)
```

<a id="core-methods"></a>
### 核心方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `query_at_time(graph, query, at_time, include_history=False, time_axis="valid")` | `Dict` | 主要 API — 将图谱过滤到 `at_time` 时有效的事实。返回 `entities`、`relationships`、`num_entities`、`num_relationships` |
| `reconstruct_at_time(graph, at_time, *, time_axis="valid")` | `Dict` | 底层 — 返回 `at_time` 时有效的深拷贝子图谱。可用于所有分析工具 |
| `query_time_range(graph, query, start_time, end_time, temporal_aggregation="union", include_intervals=True, time_axis="valid")` | `Dict` | 在 `[start, end]` 期间活跃的所有关系。`temporal_aggregation`：`"union"` / `"intersection"` / `"evolution"` |
| `validate_temporal_consistency(graph)` | `TemporalConsistencyReport` | 检测倒置区间、同边重叠事实和实体生命周期违规 |
| `query_temporal_pattern(graph, pattern, time_window=None, min_support=1)` | `Dict` | 检测 `"sequence"` 或 `"cycle"` 模式。委托给 `TemporalPatternDetector` |
| `analyze_evolution(graph, entity=None, relationship=None, start_time=None, end_time=None, metrics=None)` | `Dict` | 随时间跟踪演化指标（`"count"`、`"diversity"`、`"stability"`） |
| `find_temporal_paths(graph, source, target, start_time=None, end_time=None, max_path_length=None, enforce_causal_ordering=True, ordering_strategy="strict")` | `Dict` | 尊重时态有效性的 BFS 路径。`ordering_strategy`：`"strict"` / `"overlap"` / `"loose"` |

<a id="time_axis-parameter"></a>
### `time_axis` 参数

所有查询方法接受 `time_axis` 参数，控制使用哪些时间戳进行过滤：

| 值 | 效果 |
| :---- | :----- |
| `"valid"`（默认） | 按 `valid_from` / `valid_until` 过滤 — 事实何时为真 |
| `"transaction"` | 按 `recorded_at` / `superseded_at` 过滤 — 我们何时记录它 |
| `"both"` | 事实必须同时在两条轴上活跃 |

<a id="range-query-example"></a>
### 范围查询示例

```python
# 2021 年中任意时刻活跃的所有关系
result = query.query_time_range(kg, "", "2021-01-01", "2021-12-31")
for rel in result["relationships"]:
    print(f"  {rel['source']} --[{rel['type']}]--> {rel['target']}")

# 仅在整个范围内有效的关系（更严格）
result = query.query_time_range(
    kg, "", "2021-01-01", "2021-12-31",
    temporal_aggregation="intersection",
)

# 按日历周期分组
result = query.query_time_range(
    kg, "", "2021-01-01", "2021-12-31",
    temporal_aggregation="evolution",
)
for period, rels in result["relationship_buckets"].items():
    print(f"  {period}: {len(rels)} relationships active")
```

<a id="evolution-analysis"></a>
### 演化分析

```python
evolution = query.analyze_evolution(
    kg,
    entity="alice",            # 跟踪特定实体（None = 整个图谱）
    relationship="ceo_of",     # 跟踪特定边类型（None = 全部）
    start_time="2018-01-01",
    end_time="2024-12-31",
    metrics=["count", "diversity", "stability"],
)
print(f"Relationship count:    {evolution['count']}")
print(f"Relationship types:    {evolution['diversity']}")
```

<a id="temporal-path-finding"></a>
### 时态路径查找

```python
paths = query.find_temporal_paths(
    kg,
    source="alice",
    target="beta_ltd",
    start_time="2022-01-01",
    end_time="2024-12-31",
    max_path_length=5,
    enforce_causal_ordering=True,
    ordering_strategy="strict",  # strict|overlap|loose
)
for p in paths["paths"]:
    print(f"  {' → '.join(p['path'])}  (length={p['length']})")
```

<a id="consistency-validation"></a>
### 一致性验证

```python
from semantica.kg import TemporalGraphQuery

report = TemporalGraphQuery().validate_temporal_consistency(kg)

print(f"Errors:   {len(report.errors)}")
print(f"Warnings: {len(report.warnings)}")

for err in report.errors:
    print(f"  [{err['issue_type']}] fact_id={err['fact_id']}: {err['message']}")
```

报告的错误类型：`inverted_interval`、`invalid_temporal_fields`、`missing_source_entity`、`missing_target_entity`、`source_lifetime_mismatch`、`target_lifetime_mismatch`。
警告类型：`overlapping_same_edge`、`gap_after_restart`。


<a id="temporalpatterndetector"></a>
## TemporalPatternDetector

检测跨图谱边的循环时态模式。直接访问或通过 `TemporalGraphQuery.query_temporal_pattern()`：

```python
from semantica.kg import TemporalPatternDetector

detector = TemporalPatternDetector()

# 查找顺序边模式（A→B→C，边首尾相接）
sequences = detector.detect_temporal_patterns(
    kg,
    pattern_type="sequence",
    min_frequency=2,
    time_window=None,
)

for seq in sequences:
    print(f"Sequence: {seq['signature']}  (occurs {seq['frequency']} times)")
    for occ in seq["occurrences"]:
        print(f"  nodes={occ['nodes']}  {occ['start_time']} → {occ['end_time']}")

# 查找循环模式（A→B→C→A）
cycles = detector.detect_temporal_patterns(
    kg,
    pattern_type="cycle",
    min_frequency=1,
)
```

| 参数 | 类型 | 默认值 | 描述 |
| :--------- | :---- | :------- | :----------- |
| `pattern_type` | `str` | `"sequence"` | `"sequence"` 或 `"cycle"` |
| `min_frequency` | `int` | `2` | 模式返回的最小出现次数 |
| `time_window` | `Any` | `None` | 对模式窗口的可选时间约束 |

每个模式字典包含：`pattern_type`、`signature`（节点 ID 元组）、`frequency`、`occurrences`（包含 `nodes`、`edges`、`start_time`、`end_time` 的列表）。


<a id="allen-interval-algebra"></a>
## Allen 区间代数

`TemporalReasoningEngine` 对 `TemporalInterval` 对象进行操作 — 具有 `start: datetime` 和 `end: datetime | TemporalBound` 的冻结数据类：

```python
from semantica.kg import (
    TemporalReasoningEngine, TemporalInterval, IntervalRelation, TemporalBound
)
from datetime import datetime, timezone

def dt(year, month, day):
    return datetime(year, month, day, tzinfo=timezone.utc)

engine = TemporalReasoningEngine()

h1_2020 = TemporalInterval(start=dt(2020, 1, 1), end=dt(2020, 6, 30))
q2_q4   = TemporalInterval(start=dt(2020, 4, 1), end=dt(2020, 12, 31))

relation = engine.relation(h1_2020, q2_q4)
print(relation)                          # IntervalRelation.OVERLAPS
print(relation.value)                    # "overlaps"

print(engine.overlaps(h1_2020, q2_q4))  # True
print(engine.contains(q2_q4, h1_2020))  # False
```

<a id="all-13-relations"></a>
### 全部 13 种关系

| `IntervalRelation` | `.value` | 逆关系 | 描述 |
| :--- | :--- | :--- | :--- |
| `BEFORE` | `"before"` | `AFTER` | A 严格在 B 开始之前结束 |
| `AFTER` | `"after"` | `BEFORE` | A 严格在 B 结束之后开始 |
| `MEETS` | `"meets"` | `MET_BY` | A 恰好在 B 开始时结束 |
| `MET_BY` | `"met_by"` | `MEETS` | A 恰好在 B 结束时开始 |
| `OVERLAPS` | `"overlaps"` | `OVERLAPPED_BY` | A 和 B 共享一段时间；A 先开始且先结束 |
| `OVERLAPPED_BY` | `"overlapped_by"` | `OVERLAPS` | B 在 A 之前开始和结束，它们共享一段时间 |
| `STARTS` | `"starts"` | `STARTED_BY` | 相同的开始时间；A 先结束 |
| `STARTED_BY` | `"started_by"` | `STARTS` | 相同的开始时间；B 先结束 |
| `DURING` | `"during"` | `CONTAINS` | A 完全在 B 内部 |
| `CONTAINS` | `"contains"` | `DURING` | B 完全在 A 内部 |
| `FINISHES` | `"finishes"` | `FINISHED_BY` | 相同的结束时间；A 较晚开始 |
| `FINISHED_BY` | `"finished_by"` | `FINISHES` | 相同的结束时间；B 较晚开始 |
| `EQUALS` | `"equals"` | *（自逆）* | 完全相同的区间 |

<a id="additional-engine-methods"></a>
### 其他引擎方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `active_at(interval, timestamp, granularity=None)` | `bool` | `timestamp` 是否在 `interval` 内？ |
| `merge_intervals(intervals)` | `List[TemporalInterval]` | 合并重叠/相邻的区间 |
| `gap_analysis(intervals, domain_start, domain_end)` | `List[TemporalInterval]` | 查找域内未覆盖的间隙 |
| `coverage_percentage(intervals, domain_start, domain_end)` | `float` | 区间覆盖域的比例 |
| `timeline_of(entity_id, graph)` | `List[Dict]` | 实体的排序事件时间线 |
| `retroactive_coverage(revision, original_facts)` | `Dict` | 将事实分类为受修订 `affected`、`partial` 或 `unaffected` |
| `normalize_timestamp(timestamp, granularity)` | `datetime` | 将时间戳截断到粒度 |
| `normalize_interval(start, end, granularity)` | `TemporalInterval` | 解析并将区间扩展到粒度边界 |

<a id="advanced-interval-operations"></a>
### 进阶：区间操作

```python
from datetime import datetime, timezone

def dt(y, m, d): return datetime(y, m, d, tzinfo=timezone.utc)

intervals = [
    TemporalInterval(start=dt(2020, 1, 1), end=dt(2020, 6, 30)),
    TemporalInterval(start=dt(2020, 4, 1), end=dt(2020, 12, 31)),
    TemporalInterval(start=dt(2021, 3, 1), end=TemporalBound.OPEN),
]

# 合并重叠区间
merged = engine.merge_intervals(intervals)
print(f"Merged into {len(merged)} intervals")

# 查找 2020 年覆盖中的间隙
gaps = engine.gap_analysis(intervals, dt(2020, 1, 1), dt(2020, 12, 31))
print(f"Uncovered gaps: {len(gaps)}")

# 覆盖比例
pct = engine.coverage_percentage(intervals, dt(2020, 1, 1), dt(2021, 12, 31))
print(f"Coverage: {pct:.1%}")

# 实体时间线（所有添加/修改/删除事件按时间排序）
timeline = engine.timeline_of("alice", kg)
for event in timeline:
    print(f"  {event['timestamp'].date()}  {event['change_type']}")
```


<a id="temporalnormalizer-nl-temporal-parsing"></a>
## TemporalNormalizer — 自然语言时态解析

将自然语言时态短语转换为 `(valid_from, valid_until)` 日期时间元组。**零 LLM 调用。** 纯正则表达式 + `dateutil.relativedelta`。

```python
from semantica.kg import TemporalNormalizer
from datetime import datetime, timezone

norm = TemporalNormalizer(
    reference_date=datetime(2024, 6, 15, tzinfo=timezone.utc)
)
```

<a id="normalizevalue-optionaltupledatetime-datetime"></a>
### `normalize(value)` → `Optional[Tuple[datetime, datetime]]`

```python
# ISO 8601 → 点区间
result = norm.normalize("2022-03-15")
print(result)
# (datetime(2022, 3, 15, tzinfo=utc), datetime(2022, 3, 15, tzinfo=utc))

# 年份 → 全年跨度
result = norm.normalize("2022")
print(result)
# (datetime(2022, 1, 1, tzinfo=utc), datetime(2022, 12, 31, tzinfo=utc))

# 季度 → 季度跨度
result = norm.normalize("Q2 2021")
print(result)
# (datetime(2021, 4, 1, tzinfo=utc), datetime(2021, 6, 30, tzinfo=utc))

# 月 + 年
result = norm.normalize("January 2022")
print(result)
# (datetime(2022, 1, 1, tzinfo=utc), datetime(2022, 1, 31, tzinfo=utc))

# YYYY-MM（ISO 部分）
result = norm.normalize("2022-03")
print(result)
# (datetime(2022, 3, 1, tzinfo=utc), datetime(2022, 3, 31, tzinfo=utc))

# 相对短语（需要 reference_date）
result = norm.normalize("last quarter")
print(result)
# (datetime(2024, 1, 1, tzinfo=utc), datetime(2024, 3, 31, tzinfo=utc))

result = norm.normalize("last year")
# (datetime(2023, 1, 1, tzinfo=utc), datetime(2023, 12, 31, tzinfo=utc))

# 无法解析 → None（从不抛出异常，记录 debug 日志）
result = norm.normalize("recently")
print(result)   # None
```

<Warning>
  `normalize()` 对无法解析的输入返回 `None` — 它**从不抛出**异常。对于相对短语（`"last quarter"`、`"this year"` 等），`reference_date` **必须**在构造时设置，否则在调用时会抛出 `ValueError`。
</Warning>

<a id="normalizephrasephrase-optionaldict"></a>
### `normalize_phrase(phrase)` → `Optional[Dict]`

在短语映射中查找领域特定的时态短语：

```python
meta = norm.normalize_phrase("expiry date")
print(meta)
# {"maps_to": "valid_until", "type": "end", "domain": ["Healthcare", "Supply Chain"]}

meta = norm.normalize_phrase("retroactive to")
print(meta)
# {"maps_to": "valid_from", "type": "start", "retroactive": True, "domain": ["Regulatory", "Finance"]}

meta = norm.normalize_phrase("unknown phrase")
print(meta)   # None
```

内置领域短语覆盖：通用/政策、医疗、网络安全、供应链、金融和能源。

<a id="custom-phrase-map"></a>
### 自定义短语映射

```python
from datetime import datetime, timezone

def my_grant_window(ref: datetime):
    return (
        datetime(ref.year, 10, 1, tzinfo=timezone.utc),
        datetime(ref.year, 10, 31, tzinfo=timezone.utc),
    )

norm = TemporalNormalizer(
    reference_date=datetime(2024, 1, 1, tzinfo=timezone.utc),
    phrase_map={"grant application window": my_grant_window},
)
start, end = norm.normalize("grant application window")
```

<a id="supported-expressions"></a>
### 支持的表达式

| 模式 | 示例 | 返回类型 |
| :------- | :------- | :---------- |
| ISO 8601 完整日期/日期时间 | `"2022-03-15"`、`"2022-03-15T10:00:00Z"` | 点区间 |
| 仅年份 | `"2022"` | 全年跨度 |
| 月 + 年（单词） | `"January 2022"`、`"Jan 2022"` | 全月跨度 |
| YYYY-MM（ISO 部分） | `"2022-03"` | 全月跨度 |
| 季度 + 年份 | `"Q2 2021"` | 季度跨度 |
| 相对（内置） | `"last year"`、`"last quarter"`、`"this month"`、`"three months ago"`、`"six months ago"`、`"two years ago"` | 计算的跨度 |
| 有歧义的斜杠日期 | `"03/04/2022"` | `None` + `TemporalAmbiguityWarning` |
| 领域短语 | `"expiry date"`、`"retroactive to"` | 仅通过 `normalize_phrase()` |


<a id="temporalqueryrewriter"></a>
## TemporalQueryRewriter

从自然语言查询中提取时态意图，以便下游检索可以应用确定性的时态过滤。

**两种模式：** 仅正则表达式（无 LLM）或 LLM 辅助用于自由表达。

```python
from semantica.kg import TemporalQueryRewriter

# 仅正则表达式（默认 — 标准库之外无依赖）
rewriter = TemporalQueryRewriter()

# LLM 辅助用于更复杂的表达
from semantica.llms import Groq
rewriter = TemporalQueryRewriter(
    llm_provider=Groq(model="llama-3.1-8b-instant"),
    reference_date=datetime.now(timezone.utc),
)
```

<a id="rewritequery-contextnone-temporalqueryresult"></a>
### `rewrite(query, context=None)` → `TemporalQueryResult`

```python
# "before" 意图
r = rewriter.rewrite("which suppliers were certified before 2021?")
print(r.temporal_intent)    # "before"
print(r.at_time.year)       # 2021
print(r.rewritten_query)    # "which suppliers were certified?"
print(r.confidence)         # 0.85

# "between" 意图
r = rewriter.rewrite("revenue between Q1 2022 and Q3 2022")
print(r.temporal_intent)    # "between"
print(r.start_time)         # datetime(2022, 1, 1, tzinfo=utc)
print(r.end_time)           # datetime(2022, 9, 30, tzinfo=utc)

# "during" 意图
r = rewriter.rewrite("what decisions were made during Q2 2023?")
print(r.temporal_intent)    # "during"
print(r.at_time)            # datetime(2023, 4, 1, tzinfo=utc)

# 无时态短语
r = rewriter.rewrite("list all active suppliers")
print(r.temporal_intent)    # None
print(r.rewritten_query)    # "list all active suppliers"
print(r.has_temporal_context())  # False
```

<a id="temporalqueryresult-fields"></a>
### `TemporalQueryResult` 字段

| 字段 | 类型 | 描述 |
| :---- | :---- | :----------- |
| `rewritten_query` | `str` | 去除时态短语并规范化空格后的原始查询 |
| `at_time` | `Optional[datetime]` | `before`、`after`、`at`、`during` 意图的时间点边界 |
| `start_time` | `Optional[datetime]` | `between` 查询的下界 |
| `end_time` | `Optional[datetime]` | `between` 查询的上界 |
| `temporal_intent` | `Optional[str]` | `"before"`、`"after"`、`"at"`、`"during"`、`"between"` 之一，或 `None` |
| `confidence` | `float` | 正则表达式抽取为 `0.85`；LLM 传播的置信度或 `0.75` 回退 |

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `has_temporal_context()` | `bool` | 如果提取了任何时态参数则为 `True` |

支持的意图关键词：`before` / `prior to` / `until` / `up to`，`after` / `since` / `following`，`during` / `in` / `within`，`as of` / `at` / `on`，`between … and …`。


<a id="temporalversionmanager"></a>
## TemporalVersionManager

创建和管理带有 SHA-256 完整性检查的版本化图谱快照。支持**内存**（默认）和 **SQLite 持久化**存储。

```python
from semantica.kg import TemporalVersionManager

# 内存（默认）
versioner = TemporalVersionManager()

# SQLite 支持（跨进程重启持久化）
versioner = TemporalVersionManager(
    storage_path="graph_versions.db",
    version_strategy="timestamp",   # timestamp | incremental | semantic
)
```

<a id="methods"></a>
### 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `create_snapshot(graph, version_label, author, description)` | `Dict` | 创建带 SHA-256 校验和的快照。`author` 和 `description` 为必填 |
| `create_version(graph, version_label=None, timestamp=None, metadata=None)` | `Dict` | 轻量版本，无校验和或强制 author |
| `list_versions()` | `List[Dict]` | 列出所有存储的快照 |
| `get_version(label)` | `Optional[Dict]` | 按标签检索快照 |
| `compare_versions(v1, v2, comparison_metrics=None)` | `Dict` | 两个版本或标签之间的详细实体 + 关系差异 |
| `apply_revision(snapshot, revision)` | `Dict` | 时态修订：取代匹配的事实而不删除原始记录 |
| `validate_snapshot(snapshot)` | `bool` | 按 v1.0 模式验证（必填字段 + 类型） |
| `migrate_snapshot(snapshot)` | `Dict` | 将旧格式快照升级到 v1.0 |
| `verify_checksum(snapshot)` | `bool` | 通过 SHA-256 进行完整性检查 |

<a id="snapshot-diff-example"></a>
### 快照与差异对比示例

```python
# 创建快照（author 和 description 为必填）
snap = versioner.create_snapshot(
    kg,
    version_label="v1.0",
    author="analyst@example.com",
    description="Initial baseline",
)
print(snap["checksum"])  # SHA-256 hex string

# 列出版本
for v in versioner.list_versions():
    print(f"{v['label']:12s}  {v['author']}  {v['timestamp']}")

# 获取特定版本
past = versioner.get_version("v1.0")

# 差异：比较两个版本（传入标签或快照字典）
diff = versioner.compare_versions("v1.0", "v2.0")
print(f"Entities added:          {diff['summary']['entities_added']}")
print(f"Entities removed:        {diff['summary']['entities_removed']}")
print(f"Relationships added:     {diff['summary']['relationships_added']}")
print(f"Relationships removed:   {diff['summary']['relationships_removed']}")

# 每个修改实体上的字段级变化
for change in diff["entities_modified"]:
    print(f"  {change['id']}: {change['changes']}")
```

<a id="temporal-revision"></a>
### 时态修订

对特定事实 ID 应用修订 — 原始记录被**取代**（而非删除），保留完整的审计追踪：

```python
revision = {
    "fact_ids":       ["alice|ceo_of|acme_corp"],   # 关系键：src|type|target
    "new_valid_from": "2018-03-01",
    "new_valid_until": None,    # None = TemporalBound.OPEN
    "revision_type":  "correction",                  # correction | retroactive
    "author":         "analyst@example.com",
    "reason":         "Original start date was incorrect",
}

revised_snapshot = versioner.apply_revision(snap, revision)
# 原始事实保留，superseded_at 被设置
# 替代事实具有 new_valid_from，superseded_at = OPEN
```

<a id="integrity-migration"></a>
### 完整性与迁移

```python
# 验证快照模式
is_valid = versioner.validate_snapshot(snap)

# 验证校验和完整性
is_intact = versioner.verify_checksum(snap)

# 升级旧格式快照（无 format_version 字段）
upgraded = versioner.migrate_snapshot(old_snap)
```


<a id="context-graph-temporal-features-v030"></a>
## 上下文图谱时态特性（v0.3.0）

`ContextGraph` 直接在图谱节点和决策上公开时态感知，自 v0.3.0 起可用：

```python
from semantica.context import ContextGraph
from datetime import datetime, timezone

graph = ContextGraph(advanced_analytics=True)

# 添加有时间边界的节点
graph.add_node("policy_v1", "policy",
               properties={"text": "All transactions require dual approval"},
               valid_from="2021-01-01",
               valid_until="2023-06-30")

graph.add_node("policy_v2", "policy",
               properties={"text": "Transactions > $50k require dual approval"},
               valid_from="2023-07-01")

# 查找在特定时间戳活跃的节点
current_policies = graph.find_active_nodes(
    node_type="policy",
    at_time=datetime.now(timezone.utc),
)
for p in current_policies:
    print(p["properties"]["text"])
# → "Transactions > $50k require dual approval"

# 历史查询
past_policies = graph.find_active_nodes(
    node_type="policy",
    at_time=datetime(2022, 6, 1, tzinfo=timezone.utc),
)
for p in past_policies:
    print(p["properties"]["text"])
# → "All transactions require dual approval"
```

<a id="temporal-decision-windows"></a>
### 时态决策窗口

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=ContextGraph(),
    decision_tracking=True,
)

# 策略变更后被取代的决策
old_id = context.record_decision(
    category="data_retention", scenario="Set retention window for user PII",
    reasoning="GDPR Article 5(1)(e) limits storage",
    outcome="retain_90_days", confidence=0.98,
    valid_from="2023-01-01", valid_until="2023-06-30",
)

new_id = context.record_decision(
    category="data_retention", scenario="Set retention window for user PII",
    reasoning="Legal confirmed 60-day window after new DPA amendment",
    outcome="retain_60_days", confidence=0.99,
    valid_from="2023-07-01",
)

# 时态先例搜索
old_prec = context.find_precedents("data retention PII", as_of="2023-03-01", limit=3)
new_prec = context.find_precedents("data retention PII", as_of="2024-01-01", limit=3)
```


<a id="real-world-patterns"></a>
## 实际应用模式

<Tabs>
  <Tab title="人员与组织结构">
    ```python
    from semantica.kg import GraphBuilder, TemporalGraphQuery

    builder = GraphBuilder()
    kg = builder.build(sources=[{
        "entities": [
            {"id": "alice",   "type": "Person"},
            {"id": "finteam", "type": "Team"},
        ],
        "relationships": [
            {"source": "alice", "target": "finteam", "type": "leads",
             "valid_from": "2020-01-01", "valid_until": "2022-12-31"},
        ],
    }])

    query = TemporalGraphQuery()

    # 2022 年 11 月的事件 → 谁负责？
    result = query.query_at_time(kg, "", "2022-11-15")
    leads = [r for r in result["relationships"] if r["type"] == "leads"]
    print(f"Team lead at incident: {leads[0]['source']}")
    ```
  </Tab>
  <Tab title="策略演化">
    ```python
    from semantica.kg import TemporalVersionManager, TemporalGraphQuery

    versioner = TemporalVersionManager(storage_path="policy_history.db")
    versioner.create_snapshot(kg_before, version_label="2023-H1",
                              author="compliance@org.com",
                              description="Pre-July policy baseline")
    versioner.create_snapshot(kg_after, version_label="2023-H2",
                              author="compliance@org.com",
                              description="Post-July amendment")

    diff = versioner.compare_versions("2023-H1", "2023-H2")
    print(f"Policy changes: {diff['summary']['relationships_modified']}")
    ```
  </Tab>
  <Tab title="一致性审计">
    ```python
    from semantica.kg import TemporalGraphQuery

    report = TemporalGraphQuery().validate_temporal_consistency(kg)

    if report.errors:
        print("ERRORS (must fix):")
        for e in report.errors:
            print(f"  [{e['issue_type']}] {e['message']} (fact: {e['fact_id']})")

    if report.warnings:
        print("WARNINGS (review):")
        for w in report.warnings:
            print(f"  [{w['issue_type']}] {w['message']} (fact: {w['fact_id']})")
    ```
  </Tab>
  <Tab title="自然语言查询重写">
    ```python
    from semantica.kg import TemporalQueryRewriter, TemporalGraphQuery

    rewriter = TemporalQueryRewriter()
    query    = TemporalGraphQuery()

    user_query = "Who was responsible for compliance before the 2022 audit?"
    result = rewriter.rewrite(user_query)

    if result.has_temporal_context():
        # 使用时间点过滤
        snapshot = query.reconstruct_at_time(kg, result.at_time)
    else:
        snapshot = kg

    # 现在用 result.rewritten_query 在 snapshot 上运行你的检索
    print(f"Intent: {result.temporal_intent}")
    print(f"Query:  {result.rewritten_query}")
    ```
  </Tab>
</Tabs>


<a id="configuration"></a>
## 配置

```yaml
kg:
  temporal:
    enabled: true
    default_validity: infinite        # 省略 valid_until 时为 OPEN
    recorded_at_auto_stamp: true      # 对每条摄取的事实自动填充 recorded_at
    reasoning:
      enabled: true
      granularity: day                # second|minute|hour|day|week|month|year
      engine: allen                   # allen | point_in_time_only
```

- [知识图谱模块](kg.zh-CN.md) — 核心图谱构建、`GraphBuilder`、分析。
- [上下文模块](context.zh-CN.md) — 决策时态窗口和 `find_active_nodes()`。
- [溯源](provenance.zh-CN.md) — 与时态元数据一起打戳的 W3C PROV-O 血缘。
- [导出](export.zh-CN.md) — 带时态注释的 OWL、Turtle、JSON-LD 和 Parquet 导出。

- [时态知识图谱](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/10_Temporal_Knowledge_Graphs.ipynb) — 时态推理和 Allen 代数 · 进阶
- [上下文模块](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/19_Context_Module.ipynb) — 包含时态决策窗口 · 中级
