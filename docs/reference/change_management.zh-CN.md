---
title: "变更管理模块"
description: "知识图谱和本体的版本控制、SHA-256 校验和、差异分析、回滚与审计追踪。"
icon: "clock-rotate-left"
---

**[English](change_management.md)** · **简体中文（当前）**

**`semantica.change_management`** 为知识图谱和本体提供**企业级的版本管理与审计追踪**：

- 每个快照上都有 SHA-256 校验和：无需外部基础设施即可检测篡改
- 任意两个版本之间的结构化差异：节点的添加、删除或修改
- 完整回滚至任意命名快照
- 用于审计追踪查询的逐实体变更历史
- 支持的合规框架：HIPAA、SOX、GDPR、FDA 21 CFR Part 11

<Note>
  开箱即支持的合规框架：**HIPAA**、**SOX**、**GDPR** 和 **FDA 21 CFR Part 11**。
</Note>


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `TemporalVersionManager` | 知识图谱的快照、差异、回滚和逐节点变更历史 |
| `OntologyVersionManager` | 支持结构化差异的模式版本管理 |
| `InMemoryVersionStorage` | 用于开发和测试的快速内存存储：无持久化 |
| `SQLiteVersionStorage` | 生产环境存储：持久化到本地 SQLite 文件 |
| `compute_checksum()` | 返回任意字典（图快照、本体快照）的 SHA-256 指纹 |
| `verify_checksum()` | 通过重新计算并比较快照字典内部存储的校验和来检测篡改 |

<a id="what-you-get"></a>
## 你将获得

- **TemporalVersionManager** — 知识图谱的快照、差异、回滚和逐实体审计追踪。
- **OntologyVersionManager** — OWL 本体的版本控制，支持差异和模式迁移。
- **VersionStorage** — 可插拔后端：测试用 `InMemoryVersionStorage`，生产环境用 `SQLiteVersionStorage`。
- **完整性校验** — 每个快照上的 SHA-256 校验和，用于检测任何未授权的修改。
- **ChangeLogEntry** — 每次快照时验证的内部元数据：ISO 8601 时间戳、邮箱作者和描述（最多 500 字符）。
- **版本历史** — 通过 `list_versions()` 和 `diff()` 提供完整的防篡改版本历史，供监管审查。

<a id="typical-workflow"></a>
## 典型工作流

<Steps>
  <Step title="初始化版本管理器">
    ```python
    from semantica.change_management import TemporalVersionManager

    manager = TemporalVersionManager(storage_path="versions.db")
    ```
  </Step>
  <Step title="在每次破坏性操作前创建快照">
    ```python
    snapshot = manager.create_snapshot(
        graph=kg,
        version_label="v1.0",
        author="user@example.com",
        description="Before deduplication run"
    )
    print("Snapshot label:", snapshot["label"])
    print("Checksum:", snapshot["checksum"])
    ```
  </Step>
  <Step title="执行你的变更">
    运行去重、冲突解决、合并或任何图修改。版本管理器不会自动追踪任何内容：由你控制何时创建快照。
  </Step>
  <Step title="为结果创建快照">
    ```python
    snapshot_v2 = manager.create_snapshot(
        graph=kg,
        version_label="v2.0",
        author="user@example.com",
        description="After deduplication: 1 342 duplicates merged"
    )
    ```
  </Step>
  <Step title="通过差异分析审查变更">
    ```python
    diff = manager.diff("v1.0", "v2.0")
    summary = diff["summary"]
    print("Entities added:   ", summary["entities_added"])
    print("Entities removed: ", summary["entities_removed"])
    print("Entities modified:", summary["entities_modified"])
    ```
  </Step>
</Steps>

<Warning>
  **在每次破坏性操作前创建快照。** 在运行去重、冲突解决或合并操作之前调用 `manager.create_snapshot()`。只有在变更前已存在快照时，`restore_snapshot()` 才能执行回滚。
</Warning>

<a id="temporalversionmanager"></a>
## TemporalVersionManager

知识图谱的版本控制：快照、差异和回滚。

<a id="constructor-parameters"></a>
### 构造函数参数

| 参数 | 类型 | 默认值 | 描述 |
| :--------- | :---- | :------- | :----------- |
| `storage_path` | `str` | `None` | SQLite 数据库路径；省略则使用内存存储 |

<a id="list-and-retrieve"></a>
### 列出与检索

```python
# 列出所有版本：返回 List[Dict]，包含 label、author、timestamp、checksum、entity_count
versions = manager.list_versions()
for v in versions:
    print(v["label"], "|", v["author"], "|", v["timestamp"], "|", v["checksum"][:8], "...")

# 检索特定版本（返回完整快照字典）
snapshot = manager.get_version("v1.0")
```

<a id="temporalversionmanager-methods"></a>
### TemporalVersionManager 方法

| 方法 | 返回 | 描述 |
| :------ | :------- | :----------- |
| `create_snapshot(graph, version_label, author, description)` | `Dict[str, Any]` | 创建版本快照；返回包含 `checksum` 的完整快照字典 |
| `get_version(label)` | `Optional[Dict[str, Any]]` | 检索特定版本标签的快照字典 |
| `list_versions()` | `List[Dict[str, Any]]` | 列出所有版本元数据字典 |
| `diff(version_a, version_b)` | `Dict[str, Any]` | 比较两个快照；`compare_versions` 的别名 |
| `compare_versions(v1, v2)` | `Dict[str, Any]` | 两个快照之间详细的实体/关系差异 |
| `restore_snapshot(graph, target_version, require_confirmation=True)` | `bool` | 将活动图恢复到先前版本；默认情况下会抛出异常，除非 `require_confirmation=False` |
| `get_node_history(node_id)` | `List[Dict[str, Any]]` | 返回特定节点的按时间排序的变更历史 |
| `tag_version(version_label, tag_name)` | `None` | 创建指向版本标签的命名标签 |
| `list_tags()` | `Dict[str, str]` | 返回标签名 → 版本标签的映射 |
| `prune_versions(keep_last_n)` | `Dict[str, Any]` | 删除旧快照，仅保留最近的 N 个 |
| `verify_checksum(snapshot)` | `bool` | 根据快照存储的校验和验证其完整性 |

<a id="diff-analysis"></a>
## 差异分析

比较任意两个快照以准确查看变更内容：适用于代码审查、事件调查和监管审计：

```python
diff = manager.diff("v1.0", "v2.0")

summary = diff["summary"]
print("Entities added:      ", summary["entities_added"])
print("Entities removed:    ", summary["entities_removed"])
print("Entities modified:   ", summary["entities_modified"])
print("Relationships added: ", summary["relationships_added"])
print("Relationships removed:", summary["relationships_removed"])

# 检查各个被修改的实体
for item in diff["entities_modified"]:
    print("Modified:", item["id"])
    for field, change in item["changes"].items():
        print("  %s: %s -> %s" % (field, change["from"], change["to"]))
```

<Tip>
  **使用 `diff()` 进行代码审查和事件调查。** `manager.diff("v1.0", "v2.0")` 返回一个普通字典，包含 `"summary"`、`"entities_added"`、`"entities_removed"` 和 `"entities_modified"`：使用 `"summary"` 子字典获取计数，使用 `"entities_modified"` 检查属性级别的变更。
</Tip>

<Accordion title="diff() 返回模式">

```python
# diff() / compare_versions() 返回一个普通字典：
{
    "version1": str,                  # 第一个版本标签
    "version2": str,                  # 第二个版本标签
    "summary": {
        "entities_added":          int,
        "entities_removed":        int,
        "entities_modified":       int,
        "relationships_added":     int,
        "relationships_removed":   int,
        "relationships_modified":  int,
        # 也以 nodes_*/edges_* 别名存在
    },
    "entities_added":    List[Dict],  # 完整实体字典
    "entities_removed":  List[Dict],
    "entities_modified": List[Dict],  # {id, before, after, changes}
    "relationships_added":    List[Dict],
    "relationships_removed":  List[Dict],
    "relationships_modified": List[Dict],  # {key, before, after, changes}
    # node_*/edge_* 别名指向相同的列表
}
```

</Accordion>

<a id="ontologyversionmanager"></a>
## OntologyVersionManager

本体的版本控制：保存、差异分析和追踪模式变更：

```python
from semantica.change_management import OntologyVersionManager

manager = OntologyVersionManager()

# 保存一个版本
snapshot = manager.create_snapshot(
    ontology_data=ontology,
    version_label="1.2.0",
    author="ontology-team@example.com",
    description="Added FHIR alignment mappings"
)

# 对比两个本体版本：返回一个普通字典
diff = manager.compare_versions("1.1.0", "1.2.0")
print("Classes added:    ", diff["classes_added"])
print("Classes removed:  ", diff["classes_removed"])
print("Properties added: ", diff["properties_added"])
```

<a id="versionstorage-backends"></a>
## VersionStorage 后端

<Tabs>
  <Tab title="SQLite（生产环境）">
    ```python
    from semantica.change_management import SQLiteVersionStorage, TemporalVersionManager

    # 直接将路径传给管理器（推荐）
    manager = TemporalVersionManager(storage_path="versions.db")
    ```

    将所有版本历史持久化到磁盘。在进程重启后仍然保留。推荐用于任何需要保留审计追踪的环境。
  </Tab>
  <Tab title="内存存储（测试）">
    ```python
    from semantica.change_management import InMemoryVersionStorage, TemporalVersionManager

    # 默认（不传 storage_path）自动使用内存存储
    manager = TemporalVersionManager()
    ```

    快速且零配置。数据**不会持久化**：进程退出时所有版本历史都会丢失。仅用于单元测试和开发。
  </Tab>
</Tabs>

<Warning>
  不带参数的默认 `TemporalVersionManager()` 使用内存存储。在生产环境中务必传入 `storage_path="versions.db"` 或显式的 `SQLiteVersionStorage`：否则整个版本历史会在重启时消失。
</Warning>

<Tip>
  **在生产环境中使用 `SQLiteVersionStorage`。** 默认的内存存储会在进程退出时丢失所有版本历史。向 `TemporalVersionManager` 传入 `storage_path="versions.db"`，或显式创建 `SQLiteVersionStorage(db_path="versions.db")`。
</Tip>

<a id="integrity-verification"></a>
## 完整性校验

SHA-256 校验和可检测快照之间对图的任何未授权修改：

```python
from semantica.change_management import compute_checksum, verify_checksum

# 为任意字典计算校验和
checksum = compute_checksum({"nodes": [], "edges": []})

# verify_checksum 直接接收快照字典。
# 它从快照中读取 "checksum" 键并重新计算以进行比较。
snapshot = manager.get_version("v1.0")
is_valid = verify_checksum(snapshot)

if not is_valid:
    raise RuntimeError("Snapshot has been tampered with")
```

<Tip>
  `verify_checksum` 接收完整的快照字典（其中包含存储的 `"checksum"` 键）。直接传入 `create_snapshot` 或 `get_version` 返回的字典：无需单独的 `expected_checksum` 参数。
</Tip>

<a id="changelogentry"></a>
## ChangeLogEntry

`ChangeLogEntry` 是在 `create_snapshot` 内部创建的内部元数据对象。它在快照存储之前验证作者（必须是有效邮箱地址）和描述（非空，最多 500 字符）。

```python
from semantica.change_management.change_log import ChangeLogEntry

# 使用当前时间戳创建
entry = ChangeLogEntry.create_now(
    author="user@example.com",     # 必须是有效的邮箱地址
    description="Initial snapshot" # 最多 500 字符，非空
)
print(entry.timestamp)   # ISO 8601 时间戳
print(entry.author)      # "user@example.com"
print(entry.description) # "Initial snapshot"
```

<Accordion title="ChangeLogEntry 模式">

```python
@dataclass
class ChangeLogEntry:
    timestamp:       str            # ISO 8601 时间戳
    author:          str            # 有效邮箱地址（初始化时验证）
    description:     str            # 变更描述，最多 500 字符
    change_id:       Optional[str]  # 可选的唯一标识符
    related_changes: List[str]      # 相关变更 ID 的可选列表
```

</Accordion>

<a id="compliance-and-version-history"></a>
## 合规与版本历史

所有版本快照构成一条防篡改的审计追踪。使用 `list_versions()` 和 `diff()` 来重建和审查变更，以满足监管需求：

```python
from semantica.change_management import TemporalVersionManager

manager = TemporalVersionManager(storage_path="versions.db")

# 枚举完整的版本历史
for v in manager.list_versions():
    print(v["timestamp"], "|", v["author"], "|", v["label"], "|", v["description"])

# 对比任意两个快照生成变更报告
diff = manager.diff("v1.0", "v2.0")
s = diff["summary"]
print("Added: %d | Removed: %d | Modified: %d" % (
    s["entities_added"], s["entities_removed"], s["entities_modified"]))
```

<Tip>
  **使用 `list_versions()` 和 `diff()` 进行合规审查。** `manager.list_versions()` 返回元数据字典列表（包含 `label`、`author`、`timestamp`、`checksum`）。对 `get_version()` 返回的字典运行 `verify_checksum(snapshot)`，以在任何导出之前确认完整性。
</Tip>

在任何合规导出之前使用 `verify_checksum()` 确认快照完整性：

```python
from semantica.change_management import verify_checksum

snapshot = manager.get_version("v1.0")
is_valid = verify_checksum(snapshot)
if not is_valid:
    raise RuntimeError("Snapshot has been modified since it was recorded")
```

逐节点的变更历史可用于 HIPAA 主体访问和 SOX 审计工作流：

```python
# 获取特定节点的完整变更历史
history = manager.get_node_history("patient_001")
for record in history:
    print(record["timestamp"], record["operation"], record["version_label"])
```

<a id="compliance-coverage"></a>
### 合规覆盖范围

<AccordionGroup>
  <Accordion title="HIPAA：主体访问请求">
    使用 `manager.get_node_history("patient_001")` 检索患者实体上每条已记录的变更。每个 `MutationRecord` 包含 `timestamp`、`operation`、`entity_id`、`payload` 和 `version_label`。每个快照上的 SHA-256 校验和证明记录未被篡改。
  </Accordion>
  <Accordion title="SOX：季度审查">
    使用 `manager.list_versions()` 枚举所有快照，使用 `manager.diff(v1, v2)` 将变更报告限定在相关季度范围内。不可变的快照链提供了 SOX 第 404 节所需的监管链。
  </Accordion>
  <Accordion title="GDPR：被遗忘权验证">
    在删除数据主体的实体后，对图创建快照并与删除前的快照进行差异对比。`diff["entities_removed"]` 提供了一份机器可读的记录，准确记录了被删除的内容和时间，满足第 17 条的文档要求。
  </Accordion>
  <Accordion title="FDA 21 CFR Part 11：电子记录">
    每个快照字典都包含 `author`、`timestamp` 和 `checksum`：这是合规电子记录所需的三个字段。`verify_checksum(snapshot)` 提供 21 CFR § 11.10(e) 所要求的防篡改证据。
  </Accordion>
</AccordionGroup>

- [溯源](provenance.zh-CN.md) — W3C PROV-O 血缘追踪。
- [知识图谱](kg.zh-CN.md) — 被版本管理的图。
- [导出](export.zh-CN.md) — 导出版本化快照。
- [冲突](conflicts.zh-CN.md) — 检测版本之间引入的冲突。
