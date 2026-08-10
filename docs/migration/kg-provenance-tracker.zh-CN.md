---
title: "从 kg.ProvenanceTracker 迁移"
description: "如何从已弃用的 semantica.kg.ProvenanceTracker 迁移到统一的 semantica.provenance.ProvenanceManager。"
---

**[English](kg-provenance-tracker.md)** · **简体中文（当前）**

<a id="why-migrate"></a>
## 为什么迁移

`semantica.kg.ProvenanceTracker` 已弃用，并将在未来的主要版本中移除。它是一个独立的内存实现，从未委托给统一的溯源后端——`semantica.provenance.ProvenanceManager` 就是该后端，现在是跨所有 Semantica 模块跟踪实体和关系溯源的受支持方式（参见[溯源与审计追踪指南](/guides/provenance.zh-CN.md)）。

`kg.ProvenanceTracker` 上的每个方法现在在使用时都会发出 `DeprecationWarning`，但现有代码在该类被移除之前仍会保持不变地工作——目前没有强制的迁移截止日期。

<a id="method-mapping"></a>
## 方法映射

| `kg.ProvenanceTracker` | `ProvenanceManager` 等价方法 | 说明 |
| --- | --- | --- |
| `ProvenanceTracker()` | `ProvenanceManager()` | `ProvenanceManager` 还接受 `storage_path=` 用于 SQLite 持久化，而不仅仅是内存。 |
| `track_entity(entity_id, source, metadata)` | `track_entity(entity_id, source, metadata)` | 调用形式相同。`ProvenanceManager` 还会通过 `parent_entity_id` 自动将每次更新链接到其先前版本。 |
| `get_all_sources(entity_id)` | `get_all_sources(entity_id)` | 字段名不同：`kg` 跟踪器在 `"recorded_at"` 下返回每条记录的时间；`ProvenanceManager` 返回 `"timestamp"`。 |
| `clear(entity_id=None)` | `clear()` | `ProvenanceManager.clear()` 会清除所有溯源数据；目前还没有按实体清除的功能。 |
| `query_recorded_between(start, end)` | `query_recorded_between(start, end)` | 调用形式相同；按 `timestamp`（ISO 8601 字符串比较）筛选所有跟踪条目，而不仅仅是一个实体。 |
| `revision_history(fact_id)` | `revision_history(fact_id)` | 调用形式和返回形式相同（`version`、`valid_from`、`valid_until`、`recorded_at`、`author`、可选的 `revision_type`/`supersedes`）——它会遍历实体的 `previous_version_id` 链，而不是平铺的按实体字典。 |
| `export_audit_log(fact_ids, format)` | *暂无直接等价方法* | 从 `get_lineage()` 的输出来构建导出，或序列化 `get_statistics()` 以获得摘要视图。 |

没有直接等价的方法不计划在 `kg.ProvenanceTracker` 上重新实现——它们需要在调用方代码中编写一个小适配器，或者如果你大量依赖它们，可以向 `ProvenanceManager` 提交功能请求。

<a id="example"></a>
## 示例

```python
# 迁移前
from semantica.kg import ProvenanceTracker

tracker = ProvenanceTracker()
tracker.track_entity("entity_1", source="doc_1", metadata={"confidence": 0.9})
sources = tracker.get_all_sources("entity_1")  # [{"source": ..., "recorded_at": ..., "confidence": 0.9}]

# 迁移后
from semantica.provenance import ProvenanceManager

prov = ProvenanceManager()
prov.track_entity("entity_1", source="doc_1", metadata={"confidence": 0.9})
sources = prov.get_all_sources("entity_1")  # [{"source": ..., "timestamp": ..., "metadata": {...}, ...}]
```

<a id="suppressing-the-warning-during-migration"></a>
## 在迁移期间屏蔽警告

如果你需要暂时继续使用 `kg.ProvenanceTracker`，并希望在规划切换期间屏蔽警告：

```python
import warnings

with warnings.catch_warnings():
    warnings.simplefilter("ignore", DeprecationWarning)
    tracker = ProvenanceTracker()
```

这只是权宜之计，并非真正的修复——请计划在 `kg.ProvenanceTracker` 被移除之前迁移到 `ProvenanceManager`。
