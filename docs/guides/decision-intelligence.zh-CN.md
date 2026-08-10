---
title: "决策智能"
description: "Semantica 如何将 AI 智能体的决策作为一等知识图谱对象进行记录、存储、追踪和查询——包括因果链、先例搜索、策略执行和完整的可解释性。"
icon: "scale-balanced"
---

**[English](decision-intelligence.md)** · **简体中文（当前）**

`AgentContext.record_decision()` 将每一条 AI 决策作为知识图谱中的一个节点进行存储，并通过因果边将其与先前的决策和随后的结果连接起来。利用它来构建可审计的推理轨迹——让你能够在六个月之后准确重建：究竟是哪一次分类导致了哪一次升级，以及在记录之前检查了哪一条策略。

<a id="what-is-decision-intelligence"></a>
## 什么是决策智能？

决策智能将智能体自身的决策作为结构化数据进行记录和分析，这些数据可被查询、分析和复用。决策不再在执行后消失，而是成为带有可搜索元数据、推理链和因果关系的持久化图谱节点。

**决策智能记录决策**的方式是捕获智能体所做的每一个选择的场景、推理、结果、置信度和决策者。这些决策成为知识图谱中可查询的节点。

**决策成为图谱节点**，可以建立因果关系（决策 A 导致了决策 B），按相似度搜索（查找与此场景类似的决策），并进行统计分析（置信度趋势、常见结果）。

**目标是可审计性、可解释性、先例搜索和因果追踪。** 你可以追踪决策为何被做出，查找相似的过往决策以保持一致性，并理解从初始检测到最终行动的完整因果链。

**决策智能与智能体记忆（Agent Memory）的区别：** 智能体记忆存储外部知识（文档、事实、观察）。决策智能存储内部决策（分类、审批、智能体自身做出的行动）。

**决策智能与推理（Reasoning）的区别：** 推理使用逻辑规则从现有数据中推导出新的事实。决策智能记录的是智能体在问题求解过程中所做的选择和判断。

**决策智能与图谱分析（Graph Analytics）的区别：** 图谱分析分析知识图谱的结构属性。决策智能专门关注决策过程及其审计轨迹。

<a id="why-use-decision-intelligence"></a>
## 为什么使用决策智能？

**可审计的 AI 行动。** 每条决策都记录了推理、置信度和时间戳，为生产系统中的 AI 行为创建了完整的审计轨迹。

**可解释性。** 当利益相关者询问"系统为什么做了 X？"时，你可以追踪导致该行动的确切决策链，包括中间推理步骤。

**先例复用。** 在做出新决策之前，智能体可以搜索相似的过往场景及其结果，促进一致性并从先前的经验中学习。

**因果分析。** 通过跟踪关联决策节点之间的因果关系，理解早期决策如何级联影响到后续结果。

**治理与合规。** 策略引擎可以根据合规规则对决策进行门控，并且所有策略应用都会被记录以供监管审计。

<a id="when-to-use-when-not-to-use"></a>
## 何时使用 / 何时不使用

**在以下情况使用决策智能：**
- 构建会做出重大选择的自主智能体
- 实现需要审计轨迹的决策工作流
- 在合规要求下运营（金融服务、医疗保健、国防）
- 构建具有多个决策点的审批系统
- 在风险敏感的环境中工作，决策必须可解释

**在以下情况不要使用：**
- 构建仅检索信息的无状态聊天机器人
- 实现没有决策能力的简单 RAG 系统
- 创建只读的信息检索应用
- 构建从不做出需要审计轨迹的可行动决策的应用

<a id="api-architecture-overview"></a>
## API 架构概览

决策智能协调三个主要组件：

**AgentContext** 作为高层编排层。它提供 `record_decision()`、`find_precedents()` 和因果链方法，同时管理底层的存储和检索系统。

**PolicyEngine** 负责策略评估和合规检查。它将策略规则存储为图谱节点，并在决策被记录之前根据这些规则验证决策。

**DecisionRecorder** 专门用于记录结构化决策数据、管理审批链，以及在决策需要绕过正常规则时处理策略例外。

<Info>
  决策追踪同时需要 `VectorStore`（用于基于向量嵌入的先例搜索）和 `ContextGraph`（用于因果图谱存储）。在 `AgentContext` 上设置 `decision_tracking=True`——如果省略 `ContextGraph`，在调用时会引发 `RuntimeError`。`VectorStore` 是 `AgentContext` 本身所必需的：省略该参数会引发 Python 参数绑定的 `TypeError`，而传入 `vector_store=None` 则会在初始化期间引发 `ValueError`。
</Info>

<a id="recording-the-first-decision"></a>
## 记录第一条决策

最常见的入口是 `AgentContext.record_decision()`。它将一个 `Decision` 节点写入图谱，为混合相似度搜索生成向量嵌入，并返回一个 UUID，你用它来因果地链接后续决策。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    decision_tracking=True,
)

# 系统刚刚对一个新的威胁集群完成了分类。
# 在采取任何行动之前，先记录该分类决策。
classification_id = context.record_decision(
    category       = "threat_classification",
    scenario       = "Unattributed C2 cluster using HAMMERTOSS-like Twitter dead-drop pattern",
    reasoning      = "Infrastructure overlaps known APT29 hosting ASN; TTP T1102 matches NOBELIUM playbook with 0.88 cosine similarity",
    outcome        = "classified_as_apt29_cluster",
    confidence     = 0.88,
    decision_maker = "cti_pipeline_v2",
    entities       = ["apt29", "hammertoss", "twitter_c2"],
)

print("Decision recorded:", classification_id)
# → "Decision recorded: dec_a3f2b1c4-..."
```

`decision_maker` 字段标识产生此决策的组件、工作流、智能体或系统。使用一致的标识符（如 `"cti_pipeline_v2"`、`"analyst_chen"` 或 `"risk_model_v3"`），以便按决策来源进行过滤和分析。

支撑该节点的 `Decision` 数据类具有以下字段——这些就是被存储和搜索的内容：

```python
from semantica.context import Decision
from datetime import datetime

# 显式构造一个 Decision（record_decision 的替代方式）
d = Decision(
    decision_id    = "dec_001",           # UUID——如果通过 record_decision 省略则自动生成
    category       = "threat_classification",
    scenario       = "Unattributed C2 cluster",
    reasoning      = "Infrastructure overlaps APT29 ASN",
    outcome        = "classified_as_apt29_cluster",
    confidence     = 0.88,               # 浮点数 0.0–1.0
    timestamp      = datetime.now(),
    decision_maker = "cti_pipeline_v2",
    # 可选字段：
    valid_from     = "2025-07-01T00:00:00",   # ISO 日期时间
    valid_until    = "2025-09-30T23:59:59",   # ISO 日期时间
    metadata       = {"source_feed": "isac_partner_b"},
)
graph.add_decision(d)
```

<a id="searching-precedents-before-deciding"></a>
## 在决策之前搜索先例

在做出重大判断之前，系统应搜索过往决策中的相似场景。这就是你防止同一个集群在两次智能体运行中被不同分类的方式——第二个智能体找到第一个智能体的决策并将其用作先验。

```python
# 在对一个新的未归因集群进行分类之前先搜索
precedents = context.find_precedents(
    "unattributed C2 cluster Twitter dead-drop infrastructure",
    limit=5,
)

for p in precedents:
    print("[{:.2f} confidence] {} → {}".format(p.confidence, p.category, p.outcome))
    print("  Reasoning: {}".format(p.reasoning[:80]))
    print("  Similarity: {:.3f}".format(p.metadata.get("similarity_score", 0)))
```

混合搜索融合了两种信号：基于 `scenario` 和 `reasoning` 文本的语义相似度（权重 0.7），以及通过 Node2Vec 向量嵌入的结构性图谱邻近度（权重 0.3）。结果是一个排序后的 `Decision` 对象列表——无论表述差异多大，最相似的过往决策都会浮到顶部。

<a id="building-a-causal-chain"></a>
## 构建因果链

决策很少孤立存在。一个分类决策导致一个升级决策，升级决策又导致一个遏制决策。用因果边将它们链接起来，让你可以双向遍历这条链——上游理解什么导致了某个结果，下游查看某个早期决策触发了什么。

```python
# 上面的分类导致了一次升级
escalation_id = context.record_decision(
    category       = "escalation",
    scenario       = "APT29 cluster confirmed — active C2 beaconing to NATO contractor subnet",
    reasoning      = "Classification confidence 0.88 exceeds 0.80 escalation threshold; active C2 requires immediate SOC notification",
    outcome        = "escalated_to_soc_tier2",
    confidence     = 0.95,
    decision_maker = "escalation_engine",
)

# 链接两条决策：分类导致了升级
graph.add_causal_relationship(classification_id, escalation_id, "CAUSED")

# 该升级影响了一个补丁优先级决策
patch_id = context.record_decision(
    category       = "patch_priority",
    scenario       = "CVE-2024-3400 present in two NATO contractor VPN appliances",
    reasoning      = "Active exploitation by classified APT29 cluster elevates CVE-2024-3400 to P0 regardless of base CVSS",
    outcome        = "prioritized_cve_2024_3400_p0",
    confidence     = 0.97,
    decision_maker = "patch_engine",
)

graph.add_causal_relationship(escalation_id, patch_id, "INFLUENCED")
```

现在从补丁决策回溯追踪到其根因：

```python
upstream = context.get_causal_chain(
    patch_id,
    direction = "upstream",
    max_depth = 5,
)

print("Causal chain upstream from patch prioritization:")
for d in upstream:
    depth = d.metadata.get("causal_distance", "?")
    print("  [depth {}] {} → {}  (confidence={:.2f})".format(
        depth, d.category, d.outcome, d.confidence
    ))
# [depth 1] escalation → escalated_to_soc_tier2  (confidence=0.95)
# [depth 2] threat_classification → classified_as_apt29_cluster  (confidence=0.88)
```

并从原始分类向下游追踪，查看它触发的一切：

```python
downstream = context.get_causal_chain(
    classification_id,
    direction = "downstream",
    max_depth = 5,
)
print("Downstream decisions triggered:", len(downstream))
for d in downstream:
    print("  → {} [{}]".format(d.outcome, d.category))
```

<a id="generating-an-explainability-report"></a>
## 生成可解释性报告

`trace_decision_explainability` 一次调用就能给你全貌：上游原因、下游影响和总连接数。这就是你附加到事后复盘或审计报告中的内容。

```python
explanation = context.trace_decision_explainability(patch_id)

print("Decision:", patch_id)
print("Total graph connections :", explanation["total_connections"])
print("Upstream causes         :", len(explanation.get("upstream_decisions", [])))
print("Downstream effects      :", len(explanation.get("downstream_decisions", [])))
```

如需更深层的因果分析（包含置信度衰减和距离区间），直接在图谱上使用 `trace_decision_causality`：

```python
chains = graph.trace_decision_causality(patch_id, max_depth=5)

for chain in chains:
    print("Chain: {} hops | band={} | decay={:.3f}".format(
        chain["hop_count"], chain["distance_band"], chain["confidence_decay"]
    ))
    print("  Interpretation:", chain["interpretation"])
    # 例如："决策链跨越 2 跳，位于 'near' 区间，置信度为 84%
    #        ——因果归因可靠。"
```

<a id="gating-decisions-against-policy"></a>
## 用策略对决策进行门控

在记录高风险决策之前，先根据版本化策略对其进行检查。`PolicyEngine` 将 `Policy` 节点存储在图谱中，并根据其规则对 `Decision` 对象进行门控。

```python
from semantica.context import PolicyEngine, Policy, Decision
from datetime import datetime

engine = PolicyEngine(graph_store=graph)

engine.add_policy(Policy(
    policy_id   = "cti_confidence_gate",
    name        = "CTI Minimum Confidence Policy",
    description = "All threat classifications must have confidence >= 0.80",
    rules       = {"min_confidence": 0.80, "requires_reasoning": True},
    category    = "threat_classification",
    version     = "1.0",
    created_at  = datetime.now(),
    updated_at  = datetime.now(),
))

d = Decision(
    decision_id    = "dec_low_conf",
    category       = "threat_classification",
    scenario       = "Possible APT29 activity — weak signals only",
    reasoning      = "Single IP overlap, no TTP match",
    outcome        = "classified_as_apt29_tentative",
    confidence     = 0.62,   # 低于 0.80 阈值
    timestamp      = datetime.now(),
    decision_maker = "cti_pipeline_v2",
)

if engine.check_compliance(d, "cti_confidence_gate"):
    graph.add_decision(d)
    engine.record_policy_application(d.decision_id, "cti_confidence_gate", "1.0")
    print("Decision recorded — policy compliant.")
else:
    print("Decision blocked — confidence 0.62 below policy minimum 0.80.")
    # → "Decision blocked — confidence 0.62 below policy minimum 0.80."
```

当高紧迫性情况需要绕过策略门控时，记录该例外，包含审批人身份和理由：

```python
from semantica.context import DecisionRecorder

recorder = DecisionRecorder(graph_store=graph)

exception_id = recorder.record_exception(
    decision_id     = "dec_low_conf",
    policy_id       = "cti_confidence_gate",
    reason          = "Active exploitation in progress — cannot wait for higher-confidence attribution",
    approver        = "ciso_director",
    approval_method = "slack_dm",
    justification   = "Time-critical incident response; manual CISO sign-off obtained at 03:14 UTC",
)
print("Policy exception recorded:", exception_id)
```

对于多级审批工作流，请将 `DecisionRecorder.record_approval_chain()` 与图数据库后端（例如 Neo4j/FalkorDB）一起使用。本指南中使用的内存 `ContextGraph` 示例不支持通过 `execute_query()` 进行审批链持久化。



<a id="generating-a-decision-audit-report"></a>
## 生成决策审计报告

在班次或事件结束时，`get_decision_insights` 生成图谱中每条决策的统计摘要——适用于交接班记录和合规报告。

```python
insights = graph.get_decision_insights()

print("Total decisions today :", insights["total_decisions"])
print("Confidence — mean={:.2f}  min={:.2f}  max={:.2f}".format(
    insights["confidence_stats"]["mean"],
    insights["confidence_stats"]["min"],
    insights["confidence_stats"]["max"],
))
print("\nDecisions by category:")
for cat, count in sorted(insights["categories"].items(), key=lambda x: -x[1]):
    print("  {:35s} {}".format(cat, count))
print("\nOutcomes:")
for outcome, count in sorted(insights["outcomes"].items(), key=lambda x: -x[1]):
    print("  {:35s} {}".format(outcome, count))
```

示例输出：

```text
Total decisions today : 47
Confidence — mean=0.87  min=0.62  max=0.99

Decisions by category:
  threat_classification                 18
  patch_priority                        12
  escalation                             9
  containment                            8

Outcomes:
  classified_as_apt29_cluster           11
  prioritized_p0_patch                  12
  escalated_to_soc_tier2                 9
  isolated_host                          8
```

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防——CTI/威胁">

CTI 流水线对威胁集群进行分类，记录每个分类的置信度和推理，将分类决策与升级决策因果地链接，并为威胁情报负责人生成每日审计报告。

```python
from semantica.context import AgentContext, ContextGraph, PolicyEngine, Policy
from semantica.vector_store import VectorStore
from datetime import datetime

graph   = ContextGraph(advanced_analytics=True)
engine  = PolicyEngine(graph_store=graph)
context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    decision_tracking=True,
)

engine.add_policy(Policy(
    policy_id   = "cti_gate",
    name        = "CTI Attribution Confidence Gate",
    description = "Attributions require confidence >= 0.80",
    rules       = {"min_confidence": 0.80},
    category    = "threat_classification",
    version     = "1.0",
    created_at  = datetime.now(),
    updated_at  = datetime.now(),
))

# 在分类之前检查先例
precedents = context.find_precedents(
    "Twitter dead-drop C2 pattern overlapping APT29 infrastructure",
    limit=3,
)
for p in precedents:
    print("Prior: {} → {} ({:.0%})".format(p.scenario[:40], p.outcome, p.confidence))

# 记录分类
class_id = context.record_decision(
    category       = "threat_classification",
    scenario       = "New C2 cluster: Twitter dead-drop, AS200651 hosting, TTP T1102",
    reasoning      = "IP block overlaps APT29 cluster; T1102 matches HAMMERTOSS playbook",
    outcome        = "classified_apt29_march_cluster",
    confidence     = 0.88,
    decision_maker = "cti_pipeline_v2",
    entities       = ["apt29", "hammertoss"],
)

# 链接到下游升级
esc_id = context.record_decision(
    category       = "escalation",
    scenario       = "APT29 cluster active — beaconing to NATO subnet 10.30.0.0/16",
    reasoning      = "Active C2 with high-confidence attribution requires immediate SOC notification",
    outcome        = "escalated_tier2_soc",
    confidence     = 0.97,
    decision_maker = "escalation_engine",
)
graph.add_causal_relationship(class_id, esc_id, "CAUSED")

# 班次后审计报告
insights = graph.get_decision_insights()
print("Decisions recorded:", insights["total_decisions"])
print("Mean confidence   :", round(insights["confidence_stats"]["mean"], 2))
```

</Tab>

<Tab title="安全——SOC/事件">

在事件期间，SOC 记录遏制决策，并将其与触发该决策的检测决策建立因果链接。六小时后，事后复盘可以重放从首个告警到最终遏制的确切决策序列。

```python
from semantica.context import AgentContext, ContextGraph, DecisionRecorder
from semantica.vector_store import VectorStore

graph    = ContextGraph()
recorder = DecisionRecorder(graph_store=graph)
context  = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    decision_tracking=True,
)

# T+0：检测决策
detect_id = context.record_decision(
    category       = "detection",
    scenario       = "WKSTN-047: wmiprvse.exe spawned scheduled task — T1053.005",
    reasoning      = "Scheduled task creation by WMI provider host is high-fidelity lateral movement indicator",
    outcome        = "flagged_wkstn047_suspicious",
    confidence     = 0.93,
    decision_maker = "edr_engine",
)

# T+8分钟：由检测引发的遏制决策
contain_id = context.record_decision(
    category       = "containment",
    scenario       = "WKSTN-047 confirmed compromised — lateral movement to DC01 via SMB",
    reasoning      = "PsExec artefact on DC01; isolate before domain-wide credential compromise",
    outcome        = "isolated_wkstn047",
    confidence     = 0.95,
    decision_maker = "analyst_chen",
)
graph.add_causal_relationship(detect_id, contain_id, "CAUSED")

# 事后复盘：追踪完整链路
chain = context.get_causal_chain(contain_id, direction="upstream", max_depth=5)
print("Post-mortem — causal chain for isolation decision:")
for d in chain:
    print("  [depth {}] {} → {}  (confidence={:.2f}, maker={})".format(
        d.metadata.get("causal_distance", "?"),
        d.category, d.outcome, d.confidence, d.decision_maker,
    ))

# 遏制决策的完整可解释性
explanation = context.trace_decision_explainability(contain_id)
print("Upstream causes   :", len(explanation.get("upstream_decisions", [])))
print("Total connections :", explanation["total_connections"])
```

</Tab>

<Tab title="生命科学——临床/制药">

临床 AI 助手记录治疗调整决策及其指南先例，将其与先前的诊断决策建立因果链接，并生成结构化的决策记录供 MDT 审查和监管审计。

```python
from semantica.context import AgentContext, ContextGraph, PolicyEngine, Policy
from semantica.context import Decision, DecisionRecorder
from semantica.vector_store import VectorStore
from datetime import datetime

graph   = ContextGraph()
engine  = PolicyEngine(graph_store=graph)
context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    decision_tracking=True,
)

engine.add_policy(Policy(
    policy_id   = "clinical_confidence_gate",
    name        = "Clinical Decision Confidence Gate",
    description = "Treatment decisions require confidence >= 0.90",
    rules       = {"min_confidence": 0.90},
    category    = "treatment_modification",
    version     = "1.0",
    created_at  = datetime.now(),
    updated_at  = datetime.now(),
))

# 诊断决策先于治疗决策
diag_id = context.record_decision(
    category       = "diagnosis_assessment",
    scenario       = "Patient eGFR 28 mL/min/1.73m2, CKD Stage 4, current metformin 1000mg BD",
    reasoning      = "eGFR 28 confirms CKD Stage 4; below 30 threshold for metformin contraindication",
    outcome        = "confirmed_ckd_stage4_metformin_contraindicated",
    confidence     = 0.99,
    decision_maker = "clinical_ai_v3",
)

# 由诊断评估引发的治疗调整
treat_id = context.record_decision(
    category       = "treatment_modification",
    scenario       = "Metformin discontinuation required — eGFR 28 below contraindication threshold",
    reasoning      = "NICE NG28 and BNF both contraindicate metformin at eGFR < 30; switch to gliclazide MR 30mg OD",
    outcome        = "discontinue_metformin_initiate_gliclazide",
    confidence     = 0.97,
    decision_maker = "clinical_ai_v3",
)
graph.add_causal_relationship(diag_id, treat_id, "CAUSED")

# MDT 审计报告
chain = context.get_causal_chain(treat_id, direction="upstream", max_depth=3)
print("MDT Decision Audit — Treatment Modification")
print("=" * 50)
for d in chain:
    print("Caused by: [{}] {} (confidence={:.0%})".format(
        d.category, d.outcome, d.confidence
    ))

insights = graph.get_decision_insights()
cs = insights["confidence_stats"]
print("\nSession decisions: {}  |  mean confidence: {:.2f}".format(
    insights["total_decisions"], cs["mean"]
))
```

</Tab>

<Tab title="银行——风险/合规">

信贷决策系统根据版本化的贷款策略记录每笔贷款决策，将临界审批与压力测试决策建立因果链接，并导出完整的决策审计轨迹用于 SR 11-7 模型治理审查。

```python
from semantica.context import AgentContext, ContextGraph, PolicyEngine, Policy
from semantica.context import Decision, DecisionRecorder
from semantica.vector_store import VectorStore
from datetime import datetime

graph   = ContextGraph()
engine  = PolicyEngine(graph_store=graph)
context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    decision_tracking=True,
)

engine.add_policy(Policy(
    policy_id   = "lending_policy_v3",
    name        = "Lending Compliance Policy v3",
    description = "Credit decisions require confidence >= 0.85 and documented reasoning",
    rules       = {"min_confidence": 0.85, "requires_reasoning": True},
    category    = "loan_approval",
    version     = "3.0",
    created_at  = datetime.now(),
    updated_at  = datetime.now(),
))

# 在审批之前检查先例
precedents = context.find_precedents(
    "first-time buyer mortgage borderline DSTI stressed rate scenario",
    limit=3,
)
for p in precedents:
    print("Prior: {} → {} ({:.0%})".format(p.scenario[:40], p.outcome, p.confidence))

# 压力测试决策先于审批决策
stress_id = context.record_decision(
    category       = "stress_test",
    scenario       = "APP-2025-994421: LTV 78%, DSTI 38% at current rate; DSTI rises to 44% at +300bps",
    reasoning      = "DSTI 44% under stress exceeds 35% guideline threshold — requires LMI and income verification",
    outcome        = "stress_test_conditional_pass",
    confidence     = 0.88,
    decision_maker = "risk_model_v3",
)

# 审批决策受压力测试影响
d = Decision(
    decision_id    = "loan_dec_994421",
    category       = "loan_approval",
    scenario       = "APP-2025-994421: first-time buyer, LTV 78%, credit score 714, 30yr fixed",
    reasoning      = "Credit score 714 exceeds 700 minimum; LTV within 80% cap; conditional on LMI given stress-test DSTI",
    outcome        = "approved_conditional_lmi_required",
    confidence     = 0.89,
    timestamp      = datetime.now(),
    decision_maker = "credit_model_v3",
)

if engine.check_compliance(d, "lending_policy_v3"):
    loan_id = context.record_decision(
        category=d.category, scenario=d.scenario,
        reasoning=d.reasoning, outcome=d.outcome, confidence=d.confidence,
        decision_maker=d.decision_maker,
    )
    graph.add_causal_relationship(stress_id, loan_id, "INFLUENCED")
    engine.record_policy_application(d.decision_id, "lending_policy_v3", "3.0")
    print("Loan decision recorded — policy compliant.")

    # SR 11-7 可解释性报告
    explanation = context.trace_decision_explainability(loan_id)
    print("Upstream influences:", len(explanation.get("upstream_decisions", [])))
    print("Total connections  :", explanation["total_connections"])
```

</Tab>

</Tabs>

<a id="persisting-decisions-across-restarts"></a>
## 跨重启持久化决策

使用本地 `ContextGraph` 时，在每次会话结束时保存，在下一次会话开始时加载。所有决策节点、因果边和 FAISS 索引都会被恢复。

```python
# 会话结束时
context.save("agent_state/")
# 写入：agent_state/knowledge_graph.json + FAISS 索引

# 下一次会话开始时
context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=ContextGraph(),
    decision_tracking=True,
)
context.load("agent_state/")

# 所有过往决策立即可搜索
results = context.find_precedents("APT29 infrastructure attribution", limit=5)
```

<a id="common-pitfalls"></a>
## 常见陷阱

**记录决策时不链接因果关系。** 孤立的决策节点提供的洞察不如相连的决策链。使用 `add_causal_relationship()` 来链接相关决策并启用因果追踪。

**创建孤立的决策节点。** 决策在与图谱中的实体、其他决策或结果相连时才获得价值。使用 `entities` 参数将决策链接到相关实体。

**记录过多低价值决策。** 并非每个微小选择都需要永久记录。专注于影响结果、需要审计轨迹或受益于先例搜索的重大决策。

**将先例相似度视为证明。** 高相似度分数表明场景相关，而非情况完全相同。将先例作为指导，同时考虑每个新决策的具体上下文。

**在简单检索就足够时使用决策智能。** 如果你的系统仅检索信息而不做出可行动的选择，传统搜索或智能体记忆可能比决策追踪更合适。

<a id="related-guides"></a>
## 相关指南

- [上下文图谱](context-graphs.zh-CN.md) — `ContextGraph` 如何存储决策节点和因果边
- [距离智能](distance-intelligence.zh-CN.md) — `trace_decision_causality()` 用置信度衰减和距离区间标注因果链
- [溯源](provenance.zh-CN.md) — W3C PROV-O 审计轨迹，以标准合规的溯源方式封装决策记录
- [MCP Server](mcp-server.zh-CN.md) — 通过 `record_decision` 和 `find_precedents` 工具向 LLM 智能体暴露决策记录和先例搜索
- [变更管理](change-management.zh-CN.md) — 使用 `flush_checkpoint()` 对决策状态进行检查点以创建版本化快照
