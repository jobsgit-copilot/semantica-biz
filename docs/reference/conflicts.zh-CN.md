---
title: "冲突模块"
description: "多来源冲突检测与解决：值冲突、类型冲突、时间冲突和逻辑冲突，附带调查指南。"
icon: "triangle-exclamation"
---

**[English](conflicts.md)** · **简体中文（当前）**

**`semantica.conflicts`** 检测并解决**多个来源对同一事实存在分歧时的矛盾**：

- 五种冲突类型：值冲突、类型冲突、时间冲突、逻辑冲突和关系冲突
- 七种解决策略：投票、可信度加权、最新优先、首次出现优先、最高置信度、人工审查、专家审查
- `InvestigationGuideGenerator` 为人工解决生成逐步调查指南
- `SourceTracker` 将每个属性值映射到其贡献来源，实现完整的归因
- 冲突被显式呈现：绝不会静默破坏知识图谱


<a id="why-detect-conflicts"></a>
## 为何要检测冲突？

当你从多个来源摄取数据时，矛盾是不可避免的。一份年度报告称 Apple 的营收为 3910 亿美元；一家财经新闻社说是 3830 亿美元。如果没有冲突检测，两个值都会进入你的图，查询会静默地返回不一致的答案。

Semantica 的冲突检测使分歧变得显式且可操作：

- **值冲突**：SEC 称营收为 3910 亿美元；Reuters 称为 3830 亿美元
- **类型冲突**："Python" 在一个来源中是 `ProgrammingLanguage`，在另一个中是 `Snake` 物种
- **时间冲突**：一位 CEO 在重叠的日期范围内有两个不同的雇主
- **逻辑冲突**：一个实体同时持有两个互斥的属性
- **关系冲突**：同一关系在不同来源中具有不一致的基数或属性

<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `ConflictDetector` | 跨实体列表检测值、类型和关系冲突 |
| `ConflictResolver` | 使用可配置策略解决冲突：`voting`、`credibility_weighted`、`most_recent`、`first_seen`、`highest_confidence`、`manual_review`、`expert_review` |
| `ConflictType` | 枚举：`VALUE_CONFLICT`、`TYPE_CONFLICT`、`TEMPORAL_CONFLICT`、`LOGICAL_CONFLICT`、`RELATIONSHIP_CONFLICT` |
| `ResolutionStrategy` | 传递给 `ConflictResolver` 的可用解决策略枚举 |
| `ResolutionResult` | 由 `resolve_conflict` / `resolve_conflicts` 返回的数据类 |
| `SourceTracker` | 追踪每个实体上每个属性值由哪个来源贡献 |
| `SourceReference` | 来源文档引用，包含文档、页码、章节、置信度 |
| `PropertySource` | 聚合的属性级溯源：值 + `SourceReference` 对象列表 |
| `ConflictAnalyzer` | 分析冲突模式、严重性分布和来源统计 |
| `ConflictPattern` | 描述检测到的冲突模式的数据类 |
| `InvestigationGuideGenerator` | 为需要人工审查的冲突生成逐步调查指南 |
| `InvestigationGuide` | 指南数据类：`conflict_id`、`conflict_summary`、`severity`、`investigation_steps`、`recommended_actions` |
| `InvestigationStep` | 步骤数据类：`step_number`、`description`、`action`、`expected_outcome` |

<a id="what-you-get"></a>
## 你将获得

- **ConflictDetector** — 跨实体和关系列表的值、类型和关系冲突检测。
- **ConflictResolver** — 7 种解决策略，包括投票、可信度加权和时间偏好。
- **SourceTracker** — 追踪每个冲突事实来自哪个来源，附带逐来源可信度评分。
- **ConflictAnalyzer** — 模式分析、严重性分组、来源级统计和趋势识别。
- **InvestigationGuideGenerator** — 自动为人工和专家审查生成逐步调查清单。
- **便捷函数** — `detect_conflicts()` 和 `resolve_conflicts()` 用于单次调用工作流。

<a id="quick-start"></a>
## 快速上手

<Steps>
  <Step title="在摄取前设置可信度评分">
    ```python
    from semantica.conflicts import SourceTracker

    tracker = SourceTracker()
    tracker.set_source_credibility("sec_filings",   0.95)
    tracker.set_source_credibility("pubmed",        0.92)
    tracker.set_source_credibility("wikipedia",     0.80)
    tracker.set_source_credibility("news_articles", 0.65)
    ```
  </Step>
  <Step title="构建图后检测冲突">
    ```python
    from semantica.conflicts import ConflictDetector

    detector = ConflictDetector()

    # 检测特定属性的值冲突
    conflicts = detector.detect_value_conflicts(entities, "revenue")
    print("Found %d conflicts" % len(conflicts))

    for conflict in conflicts:
        print("[%s] entity='%s'  attr='%s'" % (
            conflict.conflict_type, conflict.entity_id, conflict.property_name))
        print("  Values: %s  Severity: %s" % (
            conflict.conflicting_values, conflict.severity))
    ```
  </Step>
  <Step title="按严重性分级">
    ```python
    from semantica.conflicts import ConflictAnalyzer

    analyzer  = ConflictAnalyzer()
    analysis  = analyzer.analyze_conflicts(conflicts)
    severity_counts = analysis["by_severity"]["counts"]
    severity_details = analysis["by_severity"]["details"]
    print("Critical: %d" % severity_counts.get("critical", 0))
    print("High:     %d" % severity_counts.get("high", 0))
    print("Low:      %d" % severity_counts.get("low", 0))
    ```
  </Step>
  <Step title="自动解决低严重性冲突，上报关键冲突">
    ```python
    from semantica.conflicts import ConflictResolver, InvestigationGuideGenerator, ResolutionStrategy

    resolver = ConflictResolver(source_tracker=tracker)

    # 自动解决低严重性冲突
    low_conflicts = severity_details.get("low", [])
    # 如需可重新获取完整 Conflict 对象：severity_details 包含字典
    auto_resolved = resolver.resolve_conflicts(
        conflicts,
        strategy=ResolutionStrategy.CREDIBILITY_WEIGHTED,
    )

    # 为关键冲突生成调查指南
    critical_ids = {d["conflict_id"] for d in severity_details.get("critical", [])}
    critical_conflicts = [c for c in conflicts if c.conflict_id in critical_ids]

    generator = InvestigationGuideGenerator()
    for conflict in critical_conflicts:
        guide = generator.generate_guide(conflict)
        print("\n%s" % guide.title)
        for step in guide.investigation_steps:
            print("  [%d] %s" % (step.step_number, step.description))
            print("       Action: %s" % step.action)
    ```
  </Step>
</Steps>

<Warning>
  **在合并之前检测，而不是之后。** 在去重和图构建之前对原始实体数据运行冲突检测。在已经包含合并实体的活动图中检测冲突更加困难：你会丢失原始来源归因。
</Warning>

<a id="conflictdetector"></a>
## ConflictDetector

```python
from semantica.conflicts import ConflictDetector

detector = ConflictDetector()

# 检测特定属性的值冲突
conflicts = detector.detect_value_conflicts(entities, "revenue")
```

<a id="detection-types"></a>
### 检测类型

| 类型 | 检测内容 | 示例 |
| :---- | :--------------- | :------- |
| `VALUE` | 同一实体、同一属性、跨来源值不同 | 营收 $391B vs $383B |
| `TYPE` | 同一实体被分类为不同类型 | "Python" 作为 Language vs Snake |
| `TEMPORAL` | 冲突的时间戳或有效期窗口 | CEO 同时在两家公司任职 |
| `LOGICAL` | 逻辑上不一致的属性组合 | `is_alive=True` 但设置了 `death_date` |
| `RELATIONSHIP` | 跨来源的关系属性不一致 | 两个来源的边权重 0.9 vs 0.3 |

<Warning>
  **`TEMPORAL` 和 `LOGICAL` 冲突检测未在 `ConflictDetector` 上直接实现。** `ConflictType` 枚举包含这些类型供自定义流水线使用，但检测器类仅实现了 `detect_value_conflicts`、`detect_type_conflicts`、`detect_relationship_conflicts` 和 `detect_entity_conflicts`。
</Warning>

按类型运行定向检测：

```python
# 检测特定属性的值冲突
value_conflicts = detector.detect_value_conflicts(entities, "revenue")

# 检测类型分类冲突
type_conflicts = detector.detect_type_conflicts(entities)

# 检测关系属性冲突（接收关系字典列表）
relation_conflicts = detector.detect_relationship_conflicts(relationships)

# 检测一组实体所有属性上的冲突
all_conflicts = detector.detect_entity_conflicts(entities)
```

<a id="conflictdetector-methods"></a>
### ConflictDetector 方法

| 方法 | 返回 | 描述 |
| :------ | :------- | :----------- |
| `detect_value_conflicts(entities, property_name, entity_type=None)` | `List[Conflict]` | 检测跨实体实例在特定属性上的值分歧 |
| `detect_type_conflicts(entities)` | `List[Conflict]` | 检测类型分类冲突 |
| `detect_relationship_conflicts(relationships)` | `List[Conflict]` | 检测关系属性冲突（接收关系字典列表） |
| `detect_entity_conflicts(entities, entity_type=None)` | `List[Conflict]` | 检测一组实体所有受监控属性上的冲突 |
| `get_conflict_report()` | `Dict[str, Any]` | 生成所有检测到的冲突的汇总报告 |

<a id="conflictresolver"></a>
## ConflictResolver

```python
from semantica.conflicts import ConflictResolver, ResolutionStrategy

resolver = ConflictResolver()
results  = resolver.resolve_conflicts(conflicts, strategy=ResolutionStrategy.VOTING)

for result in results:
    print("Resolved '%s' -> %s" % (result.conflict_id, result.resolved_value))
    print("  Strategy: %s  Confidence: %.2f" % (result.resolution_strategy, result.confidence))
```

<Tip>
  **不要自动解决所有冲突。** 对 `severity == "critical"` 或 `severity == "high"` 的冲突使用 `MANUAL_REVIEW`：高严重性意味着分歧很大，出错的风险很高。
</Tip>

<a id="choosing-a-resolution-strategy"></a>
### 选择解决策略

<Tabs>
  <Tab title="CREDIBILITY_WEIGHTED（推荐）">
    按每个来源分配的可信度评分对其值加权：自动偏向权威来源：

    ```python
    from semantica.conflicts import ConflictResolver, SourceTracker, ResolutionStrategy

    tracker = SourceTracker()
    tracker.set_source_credibility("sec_filings",   0.92)
    tracker.set_source_credibility("wikipedia",     0.80)
    tracker.set_source_credibility("news_articles", 0.65)

    resolver = ConflictResolver(source_tracker=tracker)
    results  = resolver.resolve_conflicts(
        conflicts,
        strategy=ResolutionStrategy.CREDIBILITY_WEIGHTED,
    )
    ```

    **最适用于：** 具有已知可靠性排名的来源（SEC > 博客）。
  </Tab>
  <Tab title="VOTING">
    多数投票：跨来源最常见的值胜出：

    ```python
    results = resolver.resolve_conflicts(conflicts, strategy=ResolutionStrategy.VOTING)
    ```

    **最适用于：** 3 个以上可信度大致相同的来源。当所有来源具有相同的可信度评分时，`CREDIBILITY_WEIGHTED` 的行为与 `VOTING` 相同。
  </Tab>
  <Tab title="MOST_RECENT / FIRST_SEEN">
    ```python
    # 最新来源优先：用于快速变化的事实
    results = resolver.resolve_conflicts(conflicts, strategy=ResolutionStrategy.MOST_RECENT)

    # 首次出现优先：用于稳定事实（成立日期、原始名称）
    results = resolver.resolve_conflicts(conflicts, strategy=ResolutionStrategy.FIRST_SEEN)
    ```
  </Tab>
  <Tab title="MANUAL_REVIEW / EXPERT_REVIEW">
    ```python
    # 标记供人工审查：与 InvestigationGuideGenerator 配合使用
    results   = resolver.resolve_conflicts(conflicts, strategy=ResolutionStrategy.MANUAL_REVIEW)
    generator = InvestigationGuideGenerator()

    for conflict in conflicts:
        guide = generator.generate_guide(conflict)
        print("%s" % guide.title)
        for step in guide.investigation_steps:
            print("  [%d] %s" % (step.step_number, step.description))
    ```

    **最适用于：** 高风险决策（`severity == "critical"`）、受监管数据（HIPAA/SOX）和领域特定的歧义。
  </Tab>
  <Tab title="策略对比">

    | 策略 | 枚举值 | 适用场景 |
    | :-------- | :---- | :----------- |
    | 多数投票 | `VOTING` | 3 个以上可信度大致相同的来源 |
    | 可信度加权 | `CREDIBILITY_WEIGHTED` | 来源具有不同权威级别 |
    | 最新优先 | `MOST_RECENT` | 快速变化的事实：股价、员工数、状态 |
    | 首次出现优先 | `FIRST_SEEN` | 稳定事实：成立日期、原始名称 |
    | 最高置信度 | `HIGHEST_CONFIDENCE` | 抽取流水线输出置信度评分 |
    | 人工审查 | `MANUAL_REVIEW` | 高风险决策、受监管数据 |
    | 专家审查 | `EXPERT_REVIEW` | 领域特定歧义：上报给专家 |
  </Tab>
</Tabs>

使用便捷别名以缩短代码：

```python
from semantica.conflicts import voting, credibility_weighted, most_recent, highest_confidence

results = resolver.resolve_conflicts(conflicts, strategy=voting)
```

<a id="sourcetracker"></a>
## SourceTracker

```python
from semantica.conflicts import SourceTracker, SourceReference

tracker = SourceTracker()
tracker.set_source_credibility("sec_10k",   0.92)
tracker.set_source_credibility("wikipedia", 0.80)

source_ref = SourceReference(
    document="sec_10k_2023",
    page=12,
    confidence=0.95,
)
tracker.track_property_source(
    entity_id="apple_inc",
    property_name="revenue",
    value="$391B",
    source=source_ref,
)

# 返回一个 PropertySource 对象，包含 .value 和 .sources（List[SourceReference]）
prop_source = tracker.get_property_sources("apple_inc", "revenue")
if prop_source:
    print("Value: %s" % prop_source.value)
    for s in prop_source.sources:
        credibility = tracker.get_source_credibility(s.document)
        print("  %s (confidence: %.2f, credibility: %.2f)" % (
            s.document, s.confidence, credibility))

chain = tracker.get_traceability_chain("apple_inc")
```

**关键行为：**
- 未显式设置的任何来源，可信度评分默认为 0.50
- `SourceTracker` 存储属性级溯源：因此你可以精确追踪每个值由哪个来源贡献

<Warning>
  **务必设置可信度评分。** 所有来源的默认可信度为 0.50。如果不设置显式评分，`CREDIBILITY_WEIGHTED` 的行为与 `VOTING` 相同。此策略的威力在于差异化。
</Warning>

<Tip>
  **与溯源结合使用。** `SourceTracker` 直接接入[溯源](provenance.zh-CN.md)模块的审计追踪。如果你需要解释某个已解决的值是如何被选中的，溯源记录为你提供完整的链路。
</Tip>

<a id="conflictanalyzer"></a>
## ConflictAnalyzer

```python
from semantica.conflicts import ConflictAnalyzer

analyzer = ConflictAnalyzer()

analysis     = analyzer.analyze_conflicts(conflicts)
patterns     = analysis["patterns"]
severity_counts = analysis["by_severity"]["counts"]
source_stats = analysis["by_source"]
trends       = analyzer.analyze_trends(conflicts)

# analyze_trends 返回字典列表，每个时间段一个
for t in trends:
    print("Period: %s  Count: %d  Trend: %s" % (
        t["period"], t["conflict_count"], t["trend"]))
```

**关键行为：**
- `analyze_conflicts()["patterns"]` 返回 `ConflictPattern` 对象列表：使用 `pattern.pattern_type` 和 `pattern.frequency` 发现系统性数据质量问题
- `analyze_conflicts()["by_source"]` 包含 `counts` 和 `top_sources`：出现在许多冲突中的来源可能存在上游数据质量问题
- `analyze_trends()` 返回每个时间段的字典列表（`period`、`conflict_count`、`trend`、`trend_direction`）：`trend` 为 `"increasing"`、`"decreasing"` 或 `"stable"`

<Tip>
  **使用 `analyze_conflicts()["by_source"]["top_sources"]` 来识别不良数据源。** 单个来源出现在许多冲突中，说明上游存在数据质量问题，而不是逐条记录需要解决的冲突。标记它并调查来源流水线。
</Tip>

<Tip>
  **严重性是字符串标签，不是评分。** `ConflictDetector` 根据属性重要性和值差异分配 `"critical"`、`"high"` 或 `"medium"`。关键字段（`id`、`name`、`type`、`revenue`）始终产生 `"critical"`。领域上下文决定应优先处理什么。
</Tip>

<a id="investigationguidegenerator"></a>
## InvestigationGuideGenerator

为需要人工或专家审查的冲突自动生成人类可读的调查清单：

```python
from semantica.conflicts import InvestigationGuideGenerator

generator = InvestigationGuideGenerator()
guide     = generator.generate_guide(conflict)

print("Title:   %s" % guide.title)
print("Summary: %s" % guide.conflict_summary)

for step in guide.investigation_steps:
    print("  [%d] %s" % (step.step_number, step.description))
    print("       Action: %s" % step.action)
    if step.expected_outcome:
        print("       Expected: %s" % step.expected_outcome)
```

<a id="schemas"></a>
## 数据模式

<AccordionGroup>
  <Accordion title="Conflict 模式">

```python
@dataclass
class Conflict:
    conflict_id:        str
    conflict_type:      ConflictType        # VALUE_CONFLICT | TYPE_CONFLICT | ...
    entity_id:          Optional[str]       # 涉及的实体（关系冲突时为 None）
    property_name:      Optional[str]       # 冲突的属性名
    relationship_id:    Optional[str]       # 涉及的关系（用于 RELATIONSHIP_CONFLICT）
    conflicting_values: List[Any]           # 冲突值（每个来源一个）
    sources:            List[Dict[str, Any]]# 每个值的来源字典
    confidence:         float               # 检测置信度 0–1（默认：1.0）
    severity:           str                 # "low" | "medium" | "high" | "critical"
    recommended_action: Optional[str]
    metadata:           Dict[str, Any]
```

  </Accordion>
  <Accordion title="ResolutionResult 模式">

```python
@dataclass
class ResolutionResult:
    conflict_id:        str
    resolved:           bool
    resolved_value:     Any                 # 未解决或标记审查时为 None
    resolution_strategy: Optional[str]      # 例如 "voting"、"credibility_weighted"
    confidence:         float               # 0.0–1.0
    sources_used:       List[str]           # 贡献的文档 ID
    resolution_notes:   Optional[str]
    metadata:           Dict[str, Any]
```

  </Accordion>

  <Accordion title="ConflictType 枚举">

```python
from semantica.conflicts import ConflictType

ConflictType.VALUE_CONFLICT         # revenue 在来源 A 中为 $391B，在来源 B 中为 $383B
ConflictType.TYPE_CONFLICT          # "Apple" 在一个来源中是 ORGANIZATION，在另一个中是 PRODUCT
ConflictType.TEMPORAL_CONFLICT      # 重叠的有效期窗口且状态矛盾
ConflictType.LOGICAL_CONFLICT       # 事实违反本体公理或 SHACL 约束
ConflictType.RELATIONSHIP_CONFLICT  # 跨来源的关系属性不一致
```

  </Accordion>
  <Accordion title="InvestigationGuide 和 InvestigationStep 模式">

```python
@dataclass
class InvestigationGuide:
    conflict_id:         str
    conflict_summary:    str                      # 生成的分歧摘要
    severity:            str                      # "low" | "medium" | "high" | "critical"
    conflicting_sources: List[Dict[str, Any]]
    investigation_steps: List[InvestigationStep]
    recommended_actions: List[str]
    context:             Dict[str, Any]
    generated_at:        str                      # ISO 时间戳
    # title 是一个 @property："Investigation: <conflict_id>"

@dataclass
class InvestigationStep:
    step_number:      int
    description:      str   # 要做什么
    action:           str   # 要采取的具体行动
    expected_outcome: Optional[str]
```

  </Accordion>
</AccordionGroup>

- [去重](deduplication.zh-CN.md) — 在冲突检测之前解决重复实体。
- [本体](ontology.zh-CN.md) — 逻辑冲突使用 SHACL 形状和本体公理。
- [溯源](provenance.zh-CN.md) — 追踪每个冲突事实来自哪个来源。
- [知识图谱](kg.zh-CN.md) — 被检查冲突的图。
