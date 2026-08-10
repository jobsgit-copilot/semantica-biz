---
title: "智能体记忆"
description: "AgentContext 如何存储、检索并管理持久化的智能体记忆 —— 分层的短期与长期记忆层、基于 FAISS 的语义检索、图谱增强的上下文，以及面向国防、安全、临床与金融场景的跨会话持久化。"
icon: "brain"
---

**[English](agent-memory.md)** · **简体中文（当前）**

`AgentContext` 为 LLM 智能体维护一个持久化的记忆层 —— 将观察内容存储为向量嵌入，按语义相似度进行检索，并可选地将图邻近度融合到排序中。当你的智能体需要在跨会话之间回忆过往发现、而不希望每次重启时重新阅读原始材料时，即可使用它。

<a id="what-is-agent-memory"></a>
## 什么是智能体记忆？

智能体记忆为多个智能体会话之间的信息提供持久化存储与智能检索。`AgentContext` 是编排记忆存储、检索与管理的核心组件，它整合了三个关键系统：

**VectorStore** 使用向量嵌入处理语义搜索。它将文本存储为高维向量，并通过余弦相似度或其他距离度量检索相似内容。

**ContextGraph** 以节点（实体）和边（关系）的形式维护结构化知识。这支持多跳遍历和图谱感知检索，能够沿相关实体之间的连接进行查找。

**AgentContext** 编排这两个组件，提供统一的接口来存储记忆、检索相关上下文以及管理跨会话的对话。

**持久化记忆对比无状态检索：** 传统的 RAG 系统在会话之间会丢失上下文。智能体记忆在重启后依然保留所学信息、对话历史和累积知识，从而支持长期记忆和跨会话回溯。

<a id="why-use-agent-memory"></a>
## 为什么要使用智能体记忆？

**跨会话回溯。** 智能体能够记住之前的交互、发现与决策，无需在重启后重新处理原始材料。

**长期知识累积。** 随着智能体处理更多文档，信息会随时间不断积累，为未来查询构建日益丰富的知识库。

**对话历史。** 智能体在对话过程中保持上下文，可引用长篇交互或调查中较早的内容。

**图谱感知检索。** 除简单的语义相似度之外，检索还能沿实体关系发现关联信息，这是纯向量搜索所遗漏的。

**决策追踪。** 在完整上下文与推理路径中记录决策，从而支持审计追踪，并为未来相似场景提供先例匹配。

<a id="when-to-use--when-not-to-use"></a>
## 何时使用 / 何时不应使用

**在以下场景使用智能体记忆：**
- 需要随时间累积知识的长期运行智能体
- 跨多个会话逐步建立理解的研究助手
- 上下文增量构建的调查工作流
- 必须记住过往交互与决策的系统
- 需要审计追踪与决策先例的场景

**在以下场景请勿使用：**
- 为一次性文档查询构建简单的无状态 RAG 系统
- 执行一次性文档搜索且无需持久化
- 运行无需保留知识的临时实验
- 实体之间关系无关紧要的简单检索任务

<Info>
  本指南介绍记忆层。若需图谱增强的遍历与实体链接，请参阅[上下文图谱](context-graphs.zh-CN.md)。若需决策可问责性 —— 记录、审计并因果追踪智能体所做的选择 —— 请参阅[决策智能](decision-intelligence.zh-CN.md)。
</Info>

<a id="setting-up-a-persistent-memory-context"></a>
## 设置持久化记忆上下文

在启动时同时配置向量库、知识图谱和 `AgentContext`，让这三个组件以相同路径持久化到磁盘。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

# VectorStore 依赖显式的 save()/load() 进行持久化
ti_vs = VectorStore(
    backend="faiss",
    dimension=768,
)

# ContextGraph 持有实体节点及其关系
ti_graph = ContextGraph(advanced_analytics=True)

# AgentContext 统一编排所有组件
ti_agent = AgentContext(
    vector_store=ti_vs,
    knowledge_graph=ti_graph,
    retention_days=365,          # CTI 报告在一年内仍然有效
    max_memories=50000,          # 环形缓冲区上限 —— 最旧的记忆优先淘汰
    graph_expansion=True,        # 在 retrieve() 中启用多跳图遍历
    max_expansion_hops=2,
    hybrid_alpha=0.5,            # 50% 语义得分 / 50% 图结构得分
    decision_tracking=True,      # 启用 record_decision() 与 find_precedents()
    kg_algorithms=True,          # Node2Vec 嵌入、中心性、链接预测
)
```

`hybrid_alpha` 参数控制检索如何融合语义相似度（纯向量搜索）与图结构相似度（知识图谱的拓扑结构）。设为 `0.5` 时，智能体会同等对待这两种信号。对于刚摄取、图谱较为稀疏的语料，你可以从接近 `0.0` 的值开始，随着图谱逐渐丰富再调高。

<a id="storing-what-the-agent-learns"></a>
## 存储智能体所学的内容

<a id="single-observations"></a>
### 单条观察

智能体处理的每一条情报都可以通过一次调用来存储。该字符串会立即被嵌入并索引；可选的元数据会随之一同保存，并在每次检索结果中可用。

```python
# 存储来自 OSINT 情报源的一条发现
memory_id = ti_agent.store(
    "APT29 uses HAMMERTOSS for C2 communication over Twitter and GitHub",
    metadata={
        "source": "mandiant_apt29_report",
        "actor": "APT29",
        "technique": "T1102",   # Web 服务
        "tlp": "WHITE",
    },
)
# memory_id 是一个 UUID 字符串 —— 用于稍后检索或遗忘此条目

# 将观察内容标记到活跃事件，以便后续作为一组进行检索
ti_agent.store(
    "New C2 indicator: c2-upd4te[.]ru resolves to 185.220.101.47, cert hash a3f4b8c1...",
    metadata={"type": "ioc", "confidence": 0.92, "source": "internal_hunt"},
    conversation_id="incident_ir2025_0847",
    user_id="analyst_zhang",
)
```

`conversation_id` 充当命名空间。标记为 `incident_ir2025_0847` 的记忆后续可作为一组被检索 —— 这对于构建按事件的上下文窗口很有用，且不会污染全局搜索索引。

<a id="ingesting-document-corpora"></a>
### 摄取文档语料

当 `store()` 接收一个列表时，它会将每个元素视为一个文档，构建由文本中抽取的实体与关系组成的图谱，并返回所创建内容的统计信息。

```python
stats = ti_agent.store(
    [
        {
            "content": "APT29 infrastructure cluster: 185.220.101.0/24, AS200651",
            "metadata": {"source": "shadowserver", "actor": "APT29", "ioc_type": "network"},
        },
        {
            "content": "SolarWinds supply chain compromise attributed to APT29, 2020",
            "metadata": {"source": "us_cert_aa20-352a", "actor": "APT29", "campaign": "SUNBURST"},
        },
        {
            "content": "NOBELIUM (APT29) leverages OAuth token theft against cloud workloads",
            "metadata": {"source": "msft_blog_2023", "actor": "APT29", "technique": "T1528"},
        },
    ],
    extract_entities=True,       # 抽取攻击者、IP、CVE、技术节点
    extract_relationships=True,  # 链接 攻击者 → 攻击行动 → 技术 → 基础设施
    link_entities=True,          # 合并跨文档的重复实体提及
)

print("Stored: {}, Graph nodes: {}, Graph edges: {}".format(
    stats["stored_count"],   # 3 —— 每个文档一条
    stats["graph_nodes"],    # 抽取并 upsert 到图谱中的实体
    stats["graph_edges"],    # 这些实体之间的关系
))
```

此调用之后，知识图谱中会包含 APT29、HAMMERTOSS、基础设施子网、SUNBURST 攻击行动以及 OAuth 令牌窃取等节点 —— 它们彼此相连。正是这些图谱连接支持多跳检索：当你询问"cloud OAuth 攻击"时，智能体可以沿图谱从技术节点回溯到 APT29，再向前跳转到基础设施指标。

<a id="retrieving-relevant-memory"></a>
## 检索相关记忆

<a id="semantic-retrieval"></a>
### 语义检索

最直接的检索调用按语义相似度进行搜索 —— 无需关键词匹配。你的查询向量会与所有已存储的记忆向量进行比较，并返回带分数的顶部匹配结果。

```python
results = ti_agent.retrieve(
    "cloud OAuth token theft campaigns",
    max_results=8,
    min_score=0.2,
)

for r in results:
    actor = r.get("metadata", {}).get("actor", "unknown")
    print("[{:.3f}]  [{}]  {}".format(r["score"], actor, r["content"][:80]))

# [0.912]  [APT29]  NOBELIUM (APT29) leverages OAuth token theft against cloud workloads
# [0.741]  [APT29]  SolarWinds supply chain compromise attributed to APT29, 2020
# [0.683]  [APT29]  APT29 infrastructure cluster: 185.220.101.0/24, AS200651
```

智能体将 OAuth 发现排在最前面 —— 并非因为查询包含完全相同的短语，而是因为嵌入空间将"cloud OAuth token theft campaigns"与"NOBELIUM leverages OAuth token theft against cloud workloads"放在了相近的位置。

<a id="graph-anchored-retrieval-with-proximity-scoring"></a>
### 基于邻近度评分的图锚定检索

当你以某个特定实体作为调查中心时，可以将检索锚定到该节点，并将语义得分与图邻近度得分融合。

```python
results = ti_agent.retrieve(
    "cloud OAuth token theft campaigns",
    max_results=10,
    use_graph=True,
    anchor_node="APT29",      # 广度优先搜索（BFS）从知识图谱中的此节点开始
    max_hops=3,
    proximity_weight=0.35,    # 65% 语义 + 35% 邻近度 —— 根据你的图谱密度调整
    min_score=0.1,
)

for r in results:
    # combined_score 融合了语义得分与图邻近度
    score = r.get("combined_score", r["score"])
    hop  = r.get("hop_distance", "-")
    band = r.get("distance_band", "-")  # "direct"、"near"、"mid-range"、"distant"
    print("[{:.3f}]  hop={}  band={}  {}".format(score, hop, band, r["content"][:70]))
```

`proximity_weight` 参数是按调用覆盖的 —— 当围绕特定攻击者进行枢轴分析时，可以使用较高的邻近度权重；而在广泛探索时，则可回落到纯语义搜索。

<a id="graph-grounded-reasoning"></a>
### 基于图谱锚定的推理

当你需要一个综合多条记忆项的自然语言答案时，可以使用 `query_with_reasoning()` 从图谱中检索上下文，并让 LLM 基于这些来源锚定其答案。

```python
from semantica.llms import Groq

llm = Groq(model="llama-3.1-8b-instant", api_key="YOUR_GROQ_KEY")

result = ti_agent.query_with_reasoning(
    "Which threat actors are associated with SMB lateral movement in EMEA "
    "and what infrastructure do they share with cloud OAuth campaigns?",
    llm_provider=llm,
    max_results=15,
    max_hops=3,
)

print(result["response"])       # 锚定后的自然语言答案
print(result["confidence"])     # 聚合后的检索置信度得分

# 检查答案所锚定的来源
for src in result["sources"]:
    print("  -", src["content"][:60])
```

结果包含一个 `reasoning_path` 字段，用于精确追踪到达答案所遍历的图谱边 —— 这对分析师评审与审计很有用。

<a id="building-a-working-memory-window"></a>
## 构建工作记忆窗口

使用 `conversation_id` 过滤器将检索限定在当前会话范围内，并将事件范围的历史与全局语义搜索相结合。

```python
incident_id = "ir2025_0847"

# 随着新告警到达逐一存储，并标记到该事件
ti_agent.store(
    "Alert: lateral movement detected from WKSTN-047 to DC01 via SMB (PsExec artifact)",
    metadata={"type": "alert", "severity": "critical", "technique": "T1021.002"},
    conversation_id=incident_id,
    user_id="analyst_zhang",
)

ti_agent.store(
    "Analyst note: WKSTN-047 user jsmith flagged for suspicious login from 10.2.5.40 at 03:14 UTC",
    metadata={"type": "analyst_note"},
    conversation_id=incident_id,
    user_id="analyst_zhang",
)

# 检索完整的事件线索
incident_history = ti_agent.conversation(
    incident_id,
    max_items=50,
    reverse=False,          # 时间顺序
    include_metadata=True,
)

for item in incident_history:
    role = item["metadata"].get("type", "note")
    print("[{}] {}".format(role, item["content"][:80]))

# 将事件范围的历史与跨全局记忆的语义搜索相结合
context_items = ti_agent.retrieve(
    "SMB lateral movement PsExec domain controller",
    max_results=5,
    use_graph=True,
    conversation_id=incident_id,  # 仅过滤该事件的记忆
)
```

这种模式让智能体能够为每个事件构建聚焦的工作记忆窗口，同时全局向量索引随时间累积跨所有事件的知识。

<a id="domain-examples"></a>
## 领域示例

<Tabs>
<Tab title="国防 —— CTI/威胁情报">
一个威胁情报融合单元持续摄取 OSINT 情报源、MISP 事件以及内部猎杀发现。智能体必须将新指标与已知攻击者档案进行关联，并基于累积情报 —— 而不仅仅是最新的报告 —— 产出归因评估。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import Groq

ti_graph = ContextGraph(advanced_analytics=True, node_embeddings=True)
ti_agent = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=ti_graph,
    retention_days=365,
    max_memories=50000,
    hybrid_alpha=0.6,
    decision_tracking=True,
)

# 摄取一份新的 CTI 报告 —— 实体与基础设施流入图谱
ti_agent.store(
    [
        {"content": "APT29 infrastructure cluster: 185.220.101.0/24, AS200651",
         "metadata": {"source": "shadowserver", "actor": "APT29", "tlp": "WHITE"}},
        {"content": "SolarWinds supply chain compromise attributed to APT29, campaign SUNBURST",
         "metadata": {"source": "us_cert_aa20-352a", "actor": "APT29", "campaign": "SUNBURST"}},
        {"content": "NOBELIUM (APT29) leverages OAuth token theft against cloud workloads",
         "metadata": {"source": "msft_blog_2023", "actor": "APT29", "technique": "T1528"}},
    ],
    extract_entities=True,
    extract_relationships=True,
)

# 新的猎杀发现 —— 这个 C2 域名是否与 APT29 有关？
ti_agent.store(
    "New C2 indicator: c2-upd4te[.]ru resolves to 185.220.101.47, cert hash a3f4b8...",
    metadata={"type": "ioc", "confidence": 0.92, "source": "internal_hunt"},
    conversation_id="hunt_2025_q3",
)

# 图锚定的归因查询：从 APT29 出发，遍历 3 跳
llm = Groq(model="llama-3.1-8b-instant", api_key="YOUR_GROQ_KEY")
attribution = ti_agent.query_with_reasoning(
    "Is c2-upd4te[.]ru connected to APT29 based on infrastructure overlap?",
    llm_provider=llm,
    max_results=10,
    max_hops=3,
)
print(attribution["response"])
print("Confidence: {:.0%}".format(attribution["confidence"]))

# 跨分析师班次持久化情报基础
ti_agent.save("ti_state/")
```

</Tab>
<Tab title="安全 —— SOC/事件响应">
一个 SOC 分析师助手在交接班之间承载上下文。一线（Tier 1）记录初始告警与分诊结论；二线（Tier 2）接手时已经加载了完整的事件历史，无需翻阅工单轨迹。智能体会呈现相关的操作手册步骤，并找出相似的过往事件以估算 MTTR。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import Groq

soc_graph = ContextGraph()
soc_agent = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=soc_graph,
    retention_days=180,
    max_memories=100000,
    decision_tracking=True,
)

# 一次性预加载操作手册知识库 —— 跨重启持久化
soc_agent.store([
    "T1021.002 (SMB/Windows Admin Shares): isolate host, reset service accounts, check for credential dumping",
    "T1003.001 (LSASS Memory): collect memory dump, run Mimikatz signatures, notify IR team",
    "T1190 (Exploit Public-Facing Application): check WAF logs, correlate with CVE feed, patch window 4h",
])

incident_id = "ir-2025-0847"

# 一线记录告警
soc_agent.store(
    "Alert: host WKSTN-047 failed 14 Kerberos AS-REQ in 30s from 10.2.5.40",
    metadata={"type": "alert", "severity": "high", "technique": "T1110.003"},
    conversation_id=incident_id,
    user_id="tier1_chen",
)
soc_agent.store(
    "Lateral movement confirmed: 10.2.5.40 connected to DC01 via PsExec",
    metadata={"type": "finding", "severity": "critical", "technique": "T1021.002"},
    conversation_id=incident_id,
    user_id="tier1_chen",
)

# 记录封控决策以供审计与先例匹配
decision_id = soc_agent.record_decision(
    category="containment",
    scenario="Confirmed lateral movement from WKSTN-047 to DC01 via SMB",
    reasoning="PsExec artifact detected; immediate isolation prevents DC compromise",
    outcome="isolated_wkstn047",
    confidence=0.95,
    entities=["WKSTN-047", "DC01"],
    decision_maker="tier1_chen",
)

# 呈现匹配的操作手册步骤
runbook = soc_agent.retrieve(
    "SMB lateral movement with PsExec to domain controller",
    max_results=3,
    use_graph=True,
)
for step in runbook:
    print("[{:.3f}] {}".format(step["score"], step["content"]))

# 查找相似的过往事件 —— 二线据此估算解决时间
precedents = soc_agent.find_precedents(
    "lateral movement SMB domain controller compromise",
    category="containment",
    limit=3,
)
for p in precedents:
    print("Past: {} -> {} ({:.0%})".format(p.scenario[:50], p.outcome, p.confidence))

# 二线无需阅读工单即可加载完整事件上下文
soc_agent.save("soc_state/")
```

</Tab>
<Tab title="生命科学 —— 临床/制药">
一个临床决策支持智能体在多次会诊之间维护患者上下文，并将治疗历史延续下去。智能体在处方决策前呈现指南禁忌，随后记录该决策并附完整因果追踪，供 MDT（多学科团队）审计。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

clinical_graph = ContextGraph(advanced_analytics=True)
clinical_agent = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=clinical_graph,
    retention_days=3650,      # 10 年临床记录留存
    max_memories=500000,
    decision_tracking=True,
)

# 一次性加载指南 —— ADA、BNF、NICE —— 跨会话持久化
clinical_agent.store([
    {"content": "ACE inhibitors are first-line for hypertension in diabetic patients (ADA 2024)",
     "metadata": {"source": "ADA_2024", "category": "guideline", "strength": "A"}},
    {"content": "Metformin contraindicated in eGFR < 30 mL/min/1.73m2 — risk of lactic acidosis",
     "metadata": {"source": "BNF_2024", "category": "contraindication", "strength": "absolute"}},
    {"content": "SGLT2 inhibitors reduce cardiovascular events in T2DM with CKD stage 3a (CREDENCE trial)",
     "metadata": {"source": "NEJM_CREDENCE", "category": "guideline", "strength": "A"}},
], extract_entities=True, extract_relationships=True)

patient_id = "PT-00841"

# 为本次会诊构建患者上下文
clinical_agent.store(
    "Patient PT-00841: T2DM, hypertension, eGFR 28 mL/min/1.73m2, no penicillin allergy",
    metadata={"type": "patient_summary", "patient_id": patient_id},
    conversation_id="consult_2025_07_01",
    user_id="dr_okonkwo",
)
clinical_agent.store(
    "Current medications: lisinopril 10mg, atorvastatin 40mg, aspirin 75mg",
    metadata={"type": "medication_list", "patient_id": patient_id},
    conversation_id="consult_2025_07_01",
)

# 在处方二甲双胍之前 —— 核查指南库
contraindications = clinical_agent.retrieve(
    "metformin prescribing with reduced kidney function eGFR",
    max_results=5,
    use_graph=True,
    conversation_id="consult_2025_07_01",
)
for item in contraindications:
    category = item.get("metadata", {}).get("category", "?")
    print("[{:.3f}]  [{}]  {}".format(item["score"], category, item["content"][:80]))
# [0.947]  [contraindication]  Metformin contraindicated in eGFR < 30 mL/min/1.73m2 ...
# [0.821]  [guideline]         SGLT2 inhibitors reduce cardiovascular events in T2DM with CKD ...

# 记录该决策 —— eGFR 28 低于绝对禁忌阈值
decision_id = clinical_agent.record_decision(
    category="treatment_modification",
    scenario="T2DM patient PT-00841 eGFR 28: metformin dose review required",
    reasoning=(
        "eGFR 28 falls below absolute contraindication threshold of 30 mL/min/1.73m2 "
        "per BNF_2024. Discontinue metformin; initiate dapagliflozin review per CREDENCE."
    ),
    outcome="discontinued_metformin_initiated_dapagliflozin_review",
    confidence=0.97,
    entities=["PT-00841", "metformin", "dapagliflozin", "eGFR"],
    decision_maker="dr_okonkwo",
)

# 供 MDT 审计的可解释性追踪
explanation = clinical_agent.trace_decision_explainability(decision_id)
print("Guideline connections traced: {}".format(explanation.get("total_connections", 0)))

clinical_agent.save("clinical_state/{}/".format(patient_id))
```

</Tab>
<Tab title="银行 —— 风险/合规">
一个抵押贷款承保智能体在整个决策工作流中承载监管知识与申请上下文。每一条信贷决策都会连同锚定它的确切监管指引一同记录，为模型风险治理评审生成可辩护的审计追踪。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

credit_graph = ContextGraph(advanced_analytics=True)
credit_agent = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=credit_graph,
    retention_days=2555,      # 7 年监管留存
    max_memories=1000000,
    decision_tracking=True,
    kg_algorithms=True,
)

# 加载监管知识库 —— Basel III、CRR、EBA 指引
credit_agent.store([
    {"content": "Basel III: CET1 capital ratio minimum 4.5% + 2.5% conservation buffer",
     "metadata": {"source": "BCBS_Basel3", "category": "capital_requirement"}},
    {"content": "PD floor for retail exposures: 0.1% under IRB approach (CRR Art. 160)",
     "metadata": {"source": "CRR_Art160", "category": "risk_parameter"}},
    {"content": "DSTI ratio > 40% requires enhanced creditworthiness assessment per EBA GL 2020/06",
     "metadata": {"source": "EBA_GL_2020_06", "category": "affordability"}},
    {"content": "Adverse action notice required within 30 days of credit denial (ECOA Reg. B)",
     "metadata": {"source": "ECOA_RegB", "category": "regulatory_obligation"}},
], extract_entities=True, extract_relationships=True)

app_id = "APP-2025-994421"

# 加载申请上下文
credit_agent.store(
    "Applicant APP-2025-994421: gross income 82000 GBP, requested 320000 GBP 30yr mortgage, LTV 78%",
    metadata={"type": "application_summary", "app_id": app_id},
    conversation_id=app_id,
)
credit_agent.store(
    "Credit bureau: score 714, 0 defaults in 7yr, 2 hard inquiries last 12mo, DSTI 38%",
    metadata={"type": "bureau_data", "app_id": app_id},
    conversation_id=app_id,
)

# 检索与该申请相关的监管指引
guidance = credit_agent.retrieve(
    "mortgage affordability DSTI 38% regulatory requirements LTV 78%",
    max_results=5,
    use_graph=True,
    conversation_id=app_id,
)
for g in guidance:
    source = g.get("metadata", {}).get("source", "?")
    print("[{:.3f}]  [{}]  {}".format(g["score"], source, g["content"][:80]))
# [0.891]  [EBA_GL_2020_06]  DSTI ratio > 40% requires enhanced creditworthiness ...
# [0.724]  [CRR_Art160]      PD floor for retail exposures: 0.1% under IRB approach ...

# 记录该决策 —— DSTI 38% 低于 EBA 的 40% 阈值
decision_id = credit_agent.record_decision(
    category="mortgage_origination",
    scenario="320k GBP 30yr mortgage, LTV 78%, DSTI 38%, credit score 714",
    reasoning=(
        "Score 714 exceeds 680 floor; DSTI 38% within EBA GL 2020/06 threshold of 40%; "
        "LTV 78% requires standard LMI; no derogatory history in 7yr; "
        "stress test at +300bps passes affordability."
    ),
    outcome="approved_conditional_lmi",
    confidence=0.89,
    entities=[app_id, "LTV_78pct", "DSTI_38pct"],
    decision_maker="underwriting_model_v4",
)

# 为模型治理评审查找相似先例
precedents = credit_agent.find_precedents(
    "mortgage approval borderline DSTI affordability stress test",
    category="mortgage_origination",
    limit=5,
)
for p in precedents:
    print("Precedent: {} -> {} ({:.0%})".format(p.scenario[:50], p.outcome, p.confidence))

credit_agent.save("credit_state/{}/".format(app_id))
```

</Tab>
</Tabs>

<a id="persisting-and-restoring-state"></a>
## 持久化与恢复状态

在分析师换班结束时 —— 或在进程重启之前 —— 调用 `save()` 将完整上下文写入磁盘。下次启动时，调用 `load()` 将其完全恢复。

```python
# save() 将记忆 JSON 以及后端特定的向量库工件
# 写入 agent_state/vector_store/，并将图谱导出到 knowledge_graph.json。
# 使用默认 VectorStore 实现时，这包括：
#   agent_state/agent_memory.json
#   agent_state/vector_store/store_data.pkl
#   agent_state/vector_store/index.bin
#   agent_state/knowledge_graph.json
ti_agent.save("agent_state/")
```

当新进程启动时 —— 或新分析师登录时 —— 从该检查点恢复：

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

# 创建一个配置匹配的全新上下文
ti_agent_restored = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=ContextGraph(advanced_analytics=True),
    retention_days=365,
    decision_tracking=True,
)

# load() 从磁盘恢复所有三个组件
ti_agent_restored.load("agent_state/")

# 所有记忆、图谱边和决策先例现已可用
results = ti_agent_restored.retrieve("APT29 OAuth token theft cloud infrastructure")
```

<Info>
  `AgentMemory` 本身以 JSON 保存，但向量库会单独持久化自己的索引与向量数据。`load()` 恢复的是这些后端工件，而非按需重新嵌入记忆，因此请跨会话保持相同的向量库后端、维度和评分配置。
</Info>

<a id="taking-checkpoints-during-analysis"></a>
## 分析过程中获取检查点

对于长时间运行的分析循环，在关键步骤前后获取命名快照，这样你就可以对比智能体在每个阶段新增了什么。

```python
# 分析循环开始前获取快照
ti_agent.checkpoint("pre_enrichment")

# ... 存储新证据、抽取实体、记录决策 ...

# 富集完成后获取快照
ti_agent.checkpoint("post_enrichment")

# 精确查看发生了哪些变化
diff = ti_agent.diff_checkpoints("pre_enrichment", "post_enrichment")
print("Decisions added:     {}".format(len(diff["decisions_added"])))
print("Relationships added: {}".format(len(diff["relationships_added"])))

# 可选：通过 TemporalVersionManager 持久化（初始化时需要 temporal_version_manager=）
# ti_agent.flush_checkpoint("post_enrichment")
```

<a id="memory-lifecycle-and-housekeeping"></a>
## 记忆生命周期与内务管理

保留策略在每次 `store()` 调用时自动应用 —— 早于 `retention_days` 的条目会被自动清理，无需任何手动干预。你也可以删除特定记忆或清空整个对话命名空间。

```python
# 按 ID 遗忘特定记忆
ti_agent.forget(memory_id="some-uuid-string")

# 清除标记到特定事件的所有记忆
cleared = ti_agent.forget(conversation_id="incident_ir2025_0847")
print("Cleared {} items".format(cleared))

# 清除超过 90 天的所有内容
old_cleared = ti_agent.clear(days_old=90)

# 获取当前记忆统计信息
s = ti_agent.stats()
print("Total memories: {}".format(s.get("total_items", 0)))
```

<a id="common-pitfalls"></a>
## 常见陷阱

**关闭前忘记持久化记忆。** 智能体记忆在执行期间存储于内存中。如果在进程终止前未调用 `save()`，所有累积的记忆、图谱关系和对话都将丢失。

**为无关任务使用相同的对话命名空间。** 对话 ID 应界定相关交互的范围。将单一对话用于多个无关调查会污染检索结果，并使上下文不够聚焦。

**存储过多低价值信息。** 并非每一条观察都需要永久存储。应聚焦于存储洞见、决策和重要发现，而非冗长的原始日志或临时计算。

**在简单检索即可满足时却使用智能体记忆。** 对于一次性文档查询或无状态查询，传统检索比搭建持久化记忆基础设施更简单、更高效。

**检索过多上下文并增加延迟。** 过大的 `max_results`、过高的 `max_hops` 或宽泛的查询会检索过多上下文，增加 LLM 的 token 用量和响应延迟。请从聚焦的检索参数开始。

<a id="related-guides"></a>
## 相关指南

- [上下文图谱](context-graphs.zh-CN.md) —— 底层 `ContextGraph` 如何存储实体节点与决策节点；时间区间推理；节点插入前的去重；由图谱生成本体。
- [决策智能](decision-intelligence.zh-CN.md) —— 将决策记录为图谱节点，附带因果链与策略门控。
- [多智能体系统](multi-agent.zh-CN.md) —— 通过共享的 `AgentContext` 协调多个智能体，并通过 save/load 进行交接。
- [LLM 集成](llm-integrations.zh-CN.md) —— 配置传递给 `query_with_reasoning()` 的 LLM 提供方。
- [去重指南](deduplication.zh-CN.md) —— `DuplicateDetector`、`EntityMerger`、相似度方法与聚类策略的完整参考。
- [本体管理](ontology.zh-CN.md) —— 从知识图谱生成并校验 OWL 本体；导出为 Turtle、OWL/XML、JSON-LD。
- [上下文模块参考](../reference/context.zh-CN.md) —— 完整 API：`AgentContext`、`AgentMemory`、`MemoryItem`、`ContextRetriever`。
- [向量库参考](../reference/vector_store.zh-CN.md) —— FAISS、Qdrant、pgvector、Pinecone 后端。
