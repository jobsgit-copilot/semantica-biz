---
title: "变更管理与版本管理"
description: "随时间对知识图谱和本体进行快照、版本管理、差异对比与迁移 —— 具备 SQLite 持久化、校验和验证以及结构化变更日志。"
icon: "clock-rotate-left"
---

**[English](change-management.md)** · **简体中文（当前）**

<a id="what-is-change-management-versioning"></a>
## 什么是变更管理与版本管理？

知识图谱在不断变化。`TemporalVersionManager` 通过在特定时间点捕获完整的状态快照，为你的图谱提供可验证的历史记录。它允许你在重大变更之前创建命名快照、在任意两个状态之间生成详细的差异、通过一次调用回滚到先前版本，并在向下游发布数据之前校验 SHA-256 校验和。

<a id="storage-behavior"></a>
## 存储行为

传入 `storage_path`，例如 `TemporalVersionManager(storage_path="versions.db")`，即可将快照持久化到磁盘上的 SQLite 数据库中。省略 `storage_path` 时，它会默认使用脚本结束后即消失的内存存储。

<a id="why-use-change-management"></a>
## 为什么要使用变更管理？

变更管理充当你的安全网和审计追踪。用它来：
- **保护摄取**：在大型批量摄取之前创建快照，这样当数据损坏时你可以瞬间回滚。
- **审计追踪**：维护一份可验证的日志，记录变更发生的时间、谁授权了变更，以及具体修改了哪些节点/边。
- **发布门控**：在签发发布之前，对比暂存环境与生产环境的图谱并校验校验和。

<a id="which-tool-do-i-need"></a>
## 我需要哪个工具？

Semantica 提供了多个追踪特性。选择正确的工具至关重要：
- **变更管理**（本指南）：用于**整图快照**、状态差异和完整回滚。
- **溯源（Provenance）**：用于细粒度的**来源与血缘追踪**。它回答的是*"这个特定节点来自哪份文档？"*
- **智能体记忆**：用于**对话与会话上下文状态**。它回答的是*"AI 智能体在本次会话期间做了哪些决策？"*

<a id="when-to-use--when-not-to-use"></a>
## 何时使用 / 何时不应使用

- **何时使用**：你拥有关键的检查点（如每日数据馈送、合作方合并或监管报送），需要冻结图谱的完整状态并可能将其回滚。
- **何时不应使用**：你拥有一个数百万节点的庞大图谱，并希望追踪每一次微小编辑。由于 `TemporalVersionManager` 对整张图谱字典进行快照，对巨型图谱进行过于频繁的快照会导致严重的存储膨胀。请改用溯源进行细粒度追踪。

<Info>
  `TemporalVersionManager` 与 `AgentContext.flush_checkpoint()` 直接集成 —— 智能体检查点和手动快照共享相同的存储格式，支持在自动化与手动工作流之间进行差异对比。
</Info>

---

<a id="typical-workflow"></a>
## 典型工作流

一个标准的变更管理周期遵循以下进展：

1. **快照**：捕获基线图谱状态。
2. **修改**：运行你的摄取、变更或分析。
3. **对比**：生成差异以查看发生了哪些变化。
4. **校验**：检查 SHA-256 哈希以确保数据完整性。
5. **打标签**：应用人类可读的标签（例如 `approved`）。
6. **回滚**：如果修改不正确，将图谱状态恢复原状。

---

<a id="universal-example-employee-profile-update"></a>
## 通用示例：员工档案更新

我们来看一个广为人知的示例：追踪员工的部门调转。

```python
from semantica.change_management import TemporalVersionManager
from semantica.context import ContextGraph

# 1. 设置图谱与版本管理器
graph = ContextGraph()
graph.add_node("emp-101", "Employee", "Alice")
graph.add_node("dept-hr", "Department", "Human Resources")
graph.add_edge("emp-101", "dept-hr", "works_in")

# 因为我们提供了 storage_path，所以启用了 SQLite 持久化
vm = TemporalVersionManager(storage_path="hr_versions.db")

# 2. 对基线进行快照
snap_v1 = vm.create_snapshot(
    graph         = graph.to_dict(),
    version_label = "v1_baseline",
    author        = "hr_system@example.com",
    description   = "Initial employee graph",
)

# 3. 修改图谱（将 Alice 调转到工程部）
graph.add_node("dept-eng", "Department", "Engineering")
graph.add_edge("emp-101", "dept-eng", "works_in")

# 4. 对变更后的状态进行快照
snap_v2 = vm.create_snapshot(
    graph         = graph.to_dict(),
    version_label = "v2_transfer",
    author        = "hr_admin@example.com",
    description   = "Alice transferred to Engineering",
)

# 5. 对比版本
diff = vm.compare_versions("v1_baseline", "v2_transfer")
print("Nodes added:", diff["summary"]["nodes_added"])  # 1 (Engineering)
print("Edges added:", diff["summary"]["edges_added"])  # 1 (works_in Eng)
```

接下来，让我们使用领域特定场景更深入地探讨这些能力。

---

<a id="creating-snapshots"></a>
## 创建快照

在任何重大变更之前创建快照：一次摄取扫描、一次合作方数据合并，或一次自动化富集运行。

```python
from semantica.change_management import TemporalVersionManager
from semantica.context import ContextGraph

graph = ContextGraph()
graph.add_node("apt29",         "ThreatActor",   "APT29 / NOBELIUM")
graph.add_node("cve-2024-3400", "Vulnerability", "CVE-2024-3400 PAN-OS RCE")
graph.add_edge("apt29", "cve-2024-3400", "exploits", weight=0.97)

vm = TemporalVersionManager(storage_path="cti_versions.db")

snap_pre = vm.create_snapshot(
    graph         = graph.to_dict(),
    version_label = "q3_2025_baseline",
    author        = "analyst_zhang@example.com",
    description   = "CTI baseline before Q3 OSINT sweep",
)

print(snap_pre["label"])        # "q3_2025_baseline"
print(snap_pre["checksum"])     # 序列化图谱的 SHA-256
print(snap_pre["timestamp"])    # ISO 日期时间
print(snap_pre["author"])       # "analyst_zhang"
print(len(snap_pre["nodes"]))   # 快照时的节点数量
print(len(snap_pre["edges"]))   # 快照时的边数量
```

运行摄取之后，再次快照以标记变更后的状态：

```python
graph.add_node("cve-2024-21412", "Vulnerability", "CVE-2024-21412 Windows SmartScreen bypass")
graph.add_node("apt40",          "ThreatActor",   "APT40 / BRONZE MOHAWK")
graph.add_edge("apt40", "cve-2024-21412", "exploits", weight=0.88)

snap_post = vm.create_snapshot(
    graph         = graph.to_dict(),
    version_label = "q3_2025_post_nvd_sweep",
    author        = "osint_pipeline@example.com",
    description   = "After NVD weekly sweep — 2025-07-14",
)
```

<a id="comparing-two-snapshots"></a>
## 对比两个快照

`compare_versions` 返回任意两个命名快照之间精确的差异 —— 新增、删除或修改的节点和边。

```python
diff = vm.compare_versions("q3_2025_baseline", "q3_2025_post_nvd_sweep")

s = diff["summary"]
print("Nodes added   :", s["nodes_added"])    # 2
print("Nodes removed :", s["nodes_removed"])  # 0
print("Edges added   :", s["edges_added"])    # 1

for node in diff["nodes_added"]:
    print(" +", node.get("id"), "/", node.get("content"))

for edge in diff["edges_added"]:
    print(" +", edge.get("source"), "→", edge.get("target"), "[" + edge.get("type", "") + "]")
```

你也可以直接传入快照字典而非标签：

```python
diff = vm.compare_versions(snap_pre, snap_post)
```

<a id="verifying-integrity"></a>
## 校验完整性

在将快照发布到下游系统 —— SIEM、合作方数据馈送、监管报送 —— 之前，校验 SHA-256 校验和以确认快照写入后没有任何篡改。

```python
snap = vm.get_version("q3_2025_post_nvd_sweep")

if not vm.verify_checksum(snap):
    raise RuntimeError("Checksum mismatch on q3_2025_post_nvd_sweep — aborting publish.")

print("Integrity verified — safe to publish.")
```

<a id="rolling-back"></a>
## 回滚

传入目标标签和 `require_confirmation=False`（一道显式的安全门）即可将图谱恢复到任何先前的快照。

```python
vm.restore_snapshot(
    graph                = graph,
    target_version       = "q3_2025_baseline",
    require_confirmation = False,
)

# 将回滚事件记录为一个新的快照
vm.create_snapshot(
    graph         = graph.to_dict(),
    version_label = "q3_2025_rollback",
    author        = "analyst_zhang@example.com",
    description   = "Rolled back to baseline after corrupted OSINT batch",
)
```

<Info>
  如果 `require_confirmation` 未显式设置为 `False`，`restore_snapshot` 会抛出 `ProcessingError`，以防止自动化脚本意外回滚。
</Info>

<a id="building-an-audit-changelog"></a>
## 构建审计变更日志

按时间顺序列出所有快照，并将每个快照与其前一个进行差异对比，从而产出人类可读的变更日志。

```python
versions = sorted(vm.list_versions(), key=lambda v: v["timestamp"])

print("Graph Change Log")
print("=" * 60)

for i, v in enumerate(versions):
    print("\n[{}] {}  (by {})".format(v["timestamp"][:10], v["label"], v["author"]))
    print("  " + v["description"])

    if i > 0:
        diff = vm.compare_versions(versions[i - 1]["label"], v["label"])
        s    = diff["summary"]
        print("  Changes: +{} nodes  -{} nodes  +{} edges  -{} edges".format(
            s["nodes_added"], s["nodes_removed"],
            s["edges_added"], s["edges_removed"],
        ))
```

示例输出：

```text
Graph Change Log
============================================================

[2025-07-01] q3_2025_baseline  (by analyst_zhang@example.com)
  CTI baseline before Q3 OSINT sweep

[2025-07-14] q3_2025_post_nvd_sweep  (by osint_pipeline@example.com)
  After NVD weekly sweep — 2025-07-14
  Changes: +2 nodes  -0 nodes  +1 edges  -0 edges

[2025-07-14] q3_2025_rollback  (by analyst_zhang@example.com)
  Rolled back to baseline after corrupted OSINT batch
  Changes: -2 nodes  +0 nodes  -1 edges  +0 edges
```

<a id="tagging-milestones"></a>
## 标记里程碑

为任意快照附加一个命名标签，以标记评审门控、已批准状态或监管报送。

```python
vm.tag_version("q3_2025_post_nvd_sweep", "q3-approved")

for tag_name, version_label in vm.list_tags().items():
    print(f"{tag_name:20s} → {version_label}")
# q3-approved          → q3_2025_post_nvd_sweep
```

<a id="node-history"></a>
## 节点历史

将 `TemporalVersionManager` 附加到活跃的 `ContextGraph`，可自动记录每一次单独的变更 —— 每一次 `add_node`、`add_edge` 和更新调用 —— 而不仅仅是快照级的差异。

```python
vm.attach_to_graph(graph)

graph.add_node("apt29-alias", "ThreatActor", "NOBELIUM (rebranding 2021)")
graph.add_edge("apt29", "apt29-alias", "alias_of", weight=1.0)

for record in vm.get_node_history("apt29"):
    print("[{}] {} on {}  payload={}".format(
        record["timestamp"], record["operation"], record["entity_id"],
        str(record["payload"])[:60],
    ))
```

<a id="agentcontext-integration"></a>
## AgentContext 集成

在构造 `AgentContext` 时传入版本管理器。智能体检查点和手动快照共享同一存储，为你提供一份统一的历史记录。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.change_management import TemporalVersionManager

graph = ContextGraph()
vm    = TemporalVersionManager(storage_path="agent_versions.db")

context = AgentContext(
    vector_store             = VectorStore(backend="faiss", dimension=768),
    knowledge_graph          = graph,
    temporal_version_manager = vm,
)

context.store("APT29 targeting NATO infrastructure in Q3 2025")
context.checkpoint("pre_analysis")

# ... 智能体推理循环 ...

context.checkpoint("post_analysis")
context.flush_checkpoint("post_analysis")   # 通过 vm 持久化

diff = context.diff_checkpoints("pre_analysis", "post_analysis")
print("Decisions added    :", len(diff["decisions_added"]))
print("Relationships added:", len(diff["relationships_added"]))
```

---

<a id="common-pitfalls"></a>
## 常见陷阱

- **对巨型图谱进行过于频繁的快照**：`TemporalVersionManager` 对整张图谱结构进行快照。在每次微小编辑时都对庞大图谱这样做会导致严重的存储膨胀。请将其用于里程碑门控，而非事件溯源。
- **忘记在变更追踪之前调用 `attach_to_graph`**：如果你想使用 `get_node_history()`，必须在任何变更发生*之前*调用 `vm.attach_to_graph(graph)`。否则事件将不会被捕获。
- **混淆溯源与版本管理**：不要使用版本快照来回答"这个特定节点的数据来自哪里？"。那是溯源模块的职责。版本管理追踪的是某一时间点*整个*图谱的状态。
- **忘记回滚确认要求**：在自动化脚本中调用 `restore_snapshot` 会抛出 `ProcessingError` 并导致流水线崩溃，除非你显式传入 `require_confirmation=False`。
- **过多快照导致的存储增长**：如果你从不清理旧快照或不必要地进行快照，SQLite 数据库会随时间变得很大。

---

<a id="domain-examples"></a>
## 领域示例

<Tabs>
  <Tab title="国防 —— CTI 流水线">
    每日 NVD 与 ISAC 数据馈送的摄取，附带前/后快照、一份 SOC 变更简报，以及在发布到 SIEM 之前的一道完整性门控。

```python
from semantica.change_management import TemporalVersionManager
from semantica.context import ContextGraph
import datetime

graph = ContextGraph()
vm    = TemporalVersionManager(storage_path="cti_versions.db")
today = datetime.date.today().isoformat()

snap_pre = vm.create_snapshot(
    graph         = graph.to_dict(),
    version_label = f"pre_nvd_{today}",
    author        = "osint_pipeline@example.com",
    description   = "CTI baseline before NVD sweep",
)

# 摄取
graph.add_node("cve-2025-1337",    "Vulnerability", "CVE-2025-1337 critical RCE")
graph.add_node("apt29-q3-cluster", "ThreatActor",   "APT29 Q3 2025 campaign cluster")
graph.add_edge("apt29-q3-cluster", "cve-2025-1337", "weaponizes", weight=0.91)

snap_post = vm.create_snapshot(
    graph         = graph.to_dict(),
    version_label = f"post_nvd_{today}",
    author        = "osint_pipeline@example.com",
    description   = "After NVD sweep",
)

diff = vm.compare_versions(snap_pre["label"], snap_post["label"])
s    = diff["summary"]
print(f"SOC Bulletin: +{s['nodes_added']} threat nodes, +{s['edges_added']} relationships")
for n in diff["nodes_added"]:
    print(f"  + [{n.get('type')}] {n.get('content')}")

if not vm.verify_checksum(snap_post):
    raise RuntimeError("Checksum mismatch — aborting SIEM publish")
print("Snapshot verified — publishing to SIEM feed.")
```

  </Tab>

  <Tab title="安全 —— 事件响应">
    在一次实时 IR 介入期间，对事件图谱在每次重大发现时进行快照，这样事后复盘就能精确复盘调查是如何演进的。

```python
from semantica.change_management import TemporalVersionManager
from semantica.context import ContextGraph

graph = ContextGraph()
vm    = TemporalVersionManager(storage_path="incident_ir042.db")

# T+0：初始分诊
graph.add_node("wkstn-047",   "Host",    "Compromised workstation WKSTN-047")
graph.add_node("attacker-ip", "Network", "Attacker ingress 185.220.101.47")
graph.add_edge("attacker-ip", "wkstn-047", "initial_access", weight=0.95)

vm.create_snapshot(
    graph         = graph.to_dict(),
    version_label = "ir042_t0_triage",
    author        = "analyst_chen@example.com",
    description   = "T+0 — one compromised host identified",
)

# T+2h：横向移动已确认
graph.add_node("dc01",       "Host",    "Domain controller DC01")
graph.add_node("svc-backup", "Account", "Stolen service account SVC-BACKUP")
graph.add_edge("wkstn-047",  "svc-backup", "credential_theft", weight=0.88)
graph.add_edge("svc-backup", "dc01",       "lateral_move",     weight=0.82)

vm.create_snapshot(
    graph         = graph.to_dict(),
    version_label = "ir042_t2h_lateral",
    author        = "analyst_chen@example.com",
    description   = "T+2h — lateral movement to DC01 via stolen SVC-BACKUP",
)

diff = vm.compare_versions("ir042_t0_triage", "ir042_t2h_lateral")
s    = diff["summary"]
print(f"Post-mortem T+0→T+2h: +{s['nodes_added']} hosts/accounts, +{s['edges_added']} attack paths")
for edge in diff["edges_added"]:
    print(f"  + {edge.get('source')} → {edge.get('target')} [{edge.get('type')}]")
```

  </Tab>

  <Tab title="生命科学 —— 临床试验">
    跨各阶段对临床试验知识图谱进行版本管理，以在 II 期与 III 期申报之间为监管评审产出机器可验证的差异。

```python
from semantica.change_management import TemporalVersionManager
from semantica.context import ContextGraph

graph_ph2 = ContextGraph()
graph_ph2.add_node("compound-xr401", "Compound", "XR-401")
graph_ph2.add_node("endpoint-orr",   "Endpoint", "Overall Response Rate")
graph_ph2.add_node("disease-nsclc",  "Disease",  "NSCLC")

graph_ph3 = ContextGraph()
graph_ph3.add_node("compound-xr401",  "Compound",   "XR-401")
graph_ph3.add_node("endpoint-orr",    "Endpoint",   "Overall Response Rate")
graph_ph3.add_node("endpoint-pfs",    "Endpoint",   "Progression-Free Survival")
graph_ph3.add_node("disease-nsclc",   "Disease",    "NSCLC")
graph_ph3.add_node("comparator-doce", "Comparator", "Docetaxel")

vm = TemporalVersionManager(storage_path="trial_xr401.db")

vm.create_snapshot(
    graph=graph_ph2.to_dict(), version_label="phase_ii_v1.0",
    author="clinical_data_team@example.com", description="Phase II — ORR primary, NSCLC",
)
vm.create_snapshot(
    graph=graph_ph3.to_dict(), version_label="phase_iii_v2.0",
    author="clinical_data_team@example.com", description="Phase III — PFS co-primary, Docetaxel added",
)

diff = vm.compare_versions("phase_ii_v1.0", "phase_iii_v2.0")
print("Regulatory diff Phase II → Phase III:")
for n in diff["nodes_added"]:
    print(f"  + [{n.get('type')}] {n.get('content')}")

snap = vm.get_version("phase_iii_v2.0")
assert vm.verify_checksum(snap), "Checksum failed — cannot attach to dossier"
print("Phase III snapshot verified — safe for regulatory submission.")
```

  </Tab>

  <Tab title="银行 —— 模型治理">
    在每次监管更新时对 Basel III 信用风险模型图谱进行版本管理，并在生产部署之前为模型治理委员会产出机器可验证的差异。

```python
from semantica.change_management import TemporalVersionManager
from semantica.context import ContextGraph

graph = ContextGraph()
graph.add_node("metric-cet1",      "CapitalMetric", "CET1 Capital Ratio")
graph.add_node("metric-ltv",       "RiskParameter", "LTV Ratio")
graph.add_node("metric-pd",        "RiskParameter", "Probability of Default")
graph.add_node("metric-lgd",       "RiskParameter", "Loss Given Default")
graph.add_node("regulation-cre20", "Regulation",    "Basel III CRE20")

vm = TemporalVersionManager(storage_path="credit_risk_versions.db")

vm.create_snapshot(
    graph=graph.to_dict(), version_label="basel_v1.0",
    author="risk_model_team@example.com", description="Basel III CRE20 initial graph",
)

# 监管更新 —— DSCR 成为强制要求
graph.add_node("metric-dscr", "RiskParameter", "Debt Service Coverage Ratio")
graph.add_edge("regulation-cre20", "metric-dscr", "requires", weight=1.0)

vm.create_snapshot(
    graph=graph.to_dict(), version_label="basel_v1.1",
    author="risk_model_team@example.com", description="DSCR added per EBA GL 2020/06",
)

diff = vm.compare_versions("basel_v1.0", "basel_v1.1")
s    = diff["summary"]
print(f"Model governance diff v1.0→v1.1: +{s['nodes_added']} params, +{s['edges_added']} rules")
for n in diff["nodes_added"]:
    print(f"  + [{n.get('type')}] {n.get('content')}")

snap = vm.get_version("basel_v1.1")
assert vm.verify_checksum(snap), "Checksum failed — aborting model deployment"
print("Model v1.1 verified and approved for production.")
```

  </Tab>
</Tabs>

<a id="related-guides"></a>
## 相关指南

- [上下文图谱](context-graphs.zh-CN.md) —— `ContextGraph.to_dict()` 为 `create_snapshot()` 提供输入
- [本体管理](ontology.zh-CN.md) —— 将本体版本管理与图谱版本管理相结合，获得完整的模式 + 数据审计追踪
- [SHACL 校验](shacl-validation.zh-CN.md) —— 在每个版本门控进行快照之前校验图谱数据
- [溯源](provenance.zh-CN.md) —— 将变更管理与 W3C PROV-O 血缘相结合，获得完整审计追踪
- [可视化](visualization.zh-CN.md) —— `TemporalVisualizer.visualize_snapshot_comparison()` 与 `visualize_metrics_evolution()` 将版本差异渲染为交互式图表
