---
title: "GraphRAG——图增强检索"
description: "超越向量搜索：检索事实、追踪推理路径，并将 LLM 响应锚定在你的知识图谱上。"
---

**[English](graphrag.md)** · **简体中文（当前）**

GraphRAG 把向量相似度与知识图谱遍历结合起来，使检索能够找到结构上相连的事实，而不仅仅是听起来相关的文本。当 `ContextGraph` 附加到 `AgentContext` 时，每一次检索调用都会自动将语义搜索与多跳图扩展融合——而 `query_with_reasoning()` 会在返回 LLM 答案的同时返回一条可审计的推理路径。

<a id="what-is-graphrag"></a>
## GraphRAG 是什么？

GraphRAG（图增强的检索增强生成，Graph-Augmented Retrieval-Augmented Generation）通过将向量相似度搜索与知识图谱遍历相结合来增强传统 RAG。GraphRAG 不再只检索语义上相似的文本，而是沿着实体之间的关系去寻找跨多份文档的相连证据。

**GraphRAG vs. 传统纯向量 RAG：** 向量 RAG 找到与查询文本相似的文档。GraphRAG 既找到与查询相似的文档，又找到通过实体关系与这些文档相连的文档——即便后者根本没有提到你的查询词。

**图遍历的作用：** 从向量相似文档中发现的实体出发，GraphRAG 通过关系边向外扩展以发现相关事实。这揭示了纯文本相似度会遗漏的连接——例如通过沿着 Actor → Tool → Victim Organization → Industry Sector 这条路径，发现某个威胁行为者针对医疗行业。

<a id="why-use-graphrag"></a>
## 为什么使用 GraphRAG？

**多跳发现。** 找到距离查询 2-3 个关系步骤的事实。一个关于"APT29 针对医疗行业"的问题可以通过遍历 APT29 → HAMMERTOSS → LifeCare → Healthcare Sector 来浮现关于具体医院的证据。

**相连的证据。** 检索到的是相关实体及其关系的连贯链条，而不是孤立的文档片段。这为 LLM 响应和人工分析提供了更丰富的上下文。

**调查工作流。** 通过从已知实体沿着其连接扩展来跟踪证据链。从一个可疑 IP 出发发现完整的基础设施链，或者追踪某种药物相互作用穿越代谢通路。

**更丰富的检索上下文。** 图扩展会浮现那些关键词或语义搜索单独会遗漏的相关上下文，从而带来更完整、更准确的 LLM 响应。

**可解释性。** GraphRAG 提供审计轨迹，准确展示是哪些实体和关系导向了每一条检索到的证据，使检索过程透明且可核验。

<a id="when-to-use-when-not-to-use"></a>
## 何时使用 / 何时不使用

**下列情况下 GraphRAG 能增加价值：**
- 你的领域有丰富的实体关系（威胁情报、临床数据、监管文档）
- 问题需要跨多份文档连接事实
- 调查工作流受益于沿实体连接追踪
- 可解释性和审计轨迹很重要
- 你拥有结构良好、关系有意义的知识图谱

**下列情况下简单向量搜索可能就足够了：**
- 基于主题相似度的文档检索
- 单文档问答
- 你还不知道自己在找什么的探索性搜索
- 实体关系几乎没有额外价值的领域

**延迟和复杂度考量：**
- GraphRAG 因图遍历而增加计算开销
- 多跳扩展增加检索时间和 token 用量
- 图谱质量直接影响检索质量
- 搭建需要实体抽取和关系构建

**下列情况下 GraphRAG 可能大材小用：**
- 答案已知存在于特定文档中的简单查找查询
- 延迟至关重要的实时应用
- 实体关系无法提供额外价值的领域

<a id="typical-graphrag-workflow"></a>
## 典型 GraphRAG 工作流

**摄取 → 构建图谱 → 检索 → 扩展上下文 → 推理 → 回答**

1. 使用 `AgentContext.store()` **摄取**文档，并启用实体抽取
2. 通过命名实体识别（NER）和关系抽取**构建图谱**，以填充 `ContextGraph`
3. **检索**语义相似的文档，并识别种子实体用于图扩展
4. 在你指定的跳数限制内沿实体关系**扩展上下文**
5. （可选）使用推理引擎基于扩展后的上下文进行**推理**
6. 通过 `query_with_reasoning()` 把丰富的上下文提供给 LLM 来**回答**

<Info>
  **对图谱质量的依赖：** GraphRAG 的检索质量严重依赖于图谱质量、一致的实体链接和有意义的关系。糟糕的实体抽取、重复实体或薄弱的关系会直接影响检索效果。
</Info>

<Info>
  **上下文扩展警告：** 更大的跳数会指数级增加检索到的上下文量，可能显著增加 LLM token 用量和处理时间。从 2-3 跳开始，并根据你的用例监控上下文大小。
</Info>

<Info>
  当你向 `AgentContext` 传入 `knowledge_graph=` 时，GraphRAG 会自动激活。没有单独需要开启的模式。`hybrid_alpha` 参数和 `proximity_weight` 参数控制图结构相对于向量相似度的影响力大小。
</Info>

<a id="building-the-graph-and-loading-your-intelligence"></a>
## 构建图谱并加载情报

在查询图谱之前，你需要先构建它。搭建需要三个对象：一个用于基于嵌入检索的向量库、一个用于结构遍历的 `ContextGraph`，以及一个把二者装配起来的 `AgentContext`。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

# FAISS runs locally with no external dependencies
vs = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph(advanced_analytics=True)

context = AgentContext(
    vector_store=vs,
    knowledge_graph=graph,
    graph_expansion=True,     # enable multi-hop traversal from seed nodes
    max_expansion_hops=3,     # APT29 → infrastructure → victim → sector is 3 hops
    hybrid_alpha=0.6,         # 60% graph influence, 40% vector similarity
    decision_tracking=True,   # record analyst queries as auditable decisions
)
```

现在摄取你的文档。带 `extract_entities=True` 的 `store()` 会在内部运行完整的抽取流水线——命名实体识别（NER）、关系抽取和实体链接——并同时填充向量索引和图谱：

```python
intel_documents = [
    {
        "content": "APT29 deployed HAMMERTOSS malware against NATO logistics networks in Jan–Mar 2025. "
                   "C2 infrastructure used Tor exit nodes in AS59796.",
        "metadata": {"source": "FINTEL_2025_0192", "classification": "SECRET//NOFORN"},
    },
    {
        "content": "HAMMERTOSS was subsequently observed on hosts in the LifeCare hospital network "
                   "(AS64496), suggesting lateral movement beyond the initial NATO targets.",
        "metadata": {"source": "FINTEL_2025_0211"},
    },
    {
        "content": "LifeCare operates 47 acute-care hospitals and is classified as Tier-1 "
                   "healthcare critical infrastructure under CISA Sector 6.",
        "metadata": {"source": "CISA_CI_REGISTRY_2025"},
    },
    {
        "content": "Healthcare critical infrastructure has been a high-priority targeting class "
                   "for Russian state-sponsored threat actors since 2022.",
        "metadata": {"source": "NCSC_ADVISORY_2024_12"},
    },
]

stats = context.store(
    intel_documents,
    extract_entities=True,
    extract_relationships=True,
    link_entities=True,    # merge duplicate entity mentions across documents
)

print("Graph built: {} nodes, {} edges".format(
    stats["graph_nodes"], stats["graph_edges"]
))
# Graph built: 18 nodes, 14 edges
# Nodes: APT29, HAMMERTOSS, NATO, LifeCare, AS59796, CISA Sector 6, ...
# Edges: deployed, observed_on, classified_as, targets, operates_in, ...
```

现在图谱包含了一个连通子图，跨四份文档边界把 APT29 与医疗基础设施连接起来——这是纯向量搜索看不到的东西。

<a id="retrieving-the-relevant-subgraph"></a>
## 检索相关子图

有了填充好的图谱，一次普通的 `retrieve()` 调用已经能做比向量搜索更多的事。当 `use_graph=True` 时，检索器会从 top-k 向量匹配项播种图遍历，并沿着边向外扩展，收集 `max_hops` 范围内的相连事实：

```python
results = context.retrieve(
    "APT29 tactics against healthcare",
    use_graph=True,
    max_results=10,
    expand_graph=True,
    max_hops=3,
)

for r in results:
    print("[score={:.3f}]  {}".format(
        r["score"],
        r["content"][:90],
    ))

# [score=0.921]  APT29 deployed HAMMERTOSS malware against NATO...
# [score=0.887]  HAMMERTOSS was subsequently observed on hosts in the LifeCare...
# [score=0.841]  LifeCare operates 47 acute-care hospitals...
# [score=0.798]  Healthcare critical infrastructure has been a high-priority...
```

注意顶部结果：尽管纯向量搜索可能因为缺乏关键词重叠而把相连事实排得更低，GraphRAG 因为它们在图谱中结构上与种子节点相邻而提升了它们的最终 `score`。返回的 `score` 是向量相关性与图连通性的透明融合。

当你确切知道要把遍历锚定到哪个实体时，传入 `anchor_node`：

```python
# Anchor on APT29 explicitly — proximity scores are calculated from this node
apt29_intel = context.retrieve(
    "C2 infrastructure beaconing patterns",
    use_graph=True,
    anchor_node="APT29",
    proximity_weight=0.7,   # strongly favour nodes close to APT29
    max_hops=3,
    max_results=8,
)
```

<a id="getting-a-grounded-llm-answer-with-a-reasoning-path"></a>
## 获得带推理路径的、有依据的 LLM 答案

`retrieve()` 给你有依据的上下文。`query_with_reasoning()` 则更进一步：它把该子图上下文传给 LLM，并返回答案以及检索系统在图谱中追踪的多跳路径。那条路径就是你的审计轨迹。

```python
from semantica.llms import LiteLLM

llm = LiteLLM(model="anthropic/claude-sonnet-4-20250514")

result = context.query_with_reasoning(
    "What are APT29's known TTPs against healthcare infrastructure, "
    "and what is the evidence chain connecting them?",
    llm_provider=llm,
    max_results=12,
    max_hops=3,
)

# The LLM answer — grounded in graph-retrieved context, not training memory
print(result["response"])

# The multi-hop trace: APT29 → deployed → HAMMERTOSS → observed_on → LifeCare → ...
print("\n--- Reasoning Path ---")
print(result["reasoning_path"])

# Confidence reflects how well the retrieved context supports the answer
print("\nConfidence: {:.1%}".format(result["confidence"]))

# Inspect every source the LLM was given
print("\nSources ({} total):".format(result["num_sources"]))
for src in result["sources"]:
    print("  [{:.3f}] {}".format(src["score"], src["content"][:80]))
```

`reasoning_path` 字段正是 GraphRAG 区别于黑盒 LLM 调用的地方。当分析师问"你怎么知道 APT29 针对医疗行业？"，你可以给他们看系统在你自己的文档上做出的确切遍历——而不是模型从训练数据中生成的说法。

`query_with_reasoning()` 的完整返回结构：

```python
{
    "response":             str,   # LLM-generated answer, grounded in retrieved subgraph
    "reasoning_path":       str,   # multi-hop traversal narrative
    "sources":              list,  # list of retrieved context dicts with scores
    "confidence":           float, # 0–1 aggregate confidence
    "num_sources":          int,
    "num_reasoning_paths":  int,
}
```

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防——CTI/威胁">

多源情报融合：把 OSINT 威胁源、NVD CVE 数据和 HUMINT 摘要摄取进一张单一图谱，然后用多跳推理查询以追踪 C2 基础设施链，并把攻击行动归因到具体行为者。

在涉密环境中，图谱可以按数据处理警示语分区——每个 `AgentContext` 只在经过该用户许可的文档子集上运作。`reasoning_path` 输出还可作为可净化审计轨迹，用于降级报告。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import LiteLLM

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph()

context = AgentContext(
    vector_store=vs,
    knowledge_graph=graph,
    graph_expansion=True,
    max_expansion_hops=3,  # actor → infra → victim → attribution chain
    hybrid_alpha=0.6,      # graph-heavy: structured intel benefits from topology
    decision_tracking=True,
)

# Ingest multi-INT corpus
humint_summary = """
HUMINT-2025-Q1-007: Source BRAVO-9 confirms APT29 operating from
infrastructure in AS59796. C2 beacons use Tor exit nodes in DE/NL.
Targets: ITAR-controlled defense contractors in aerospace sector.
"""
cti_report_text = "APT29 exploited CVE-2025-3400 in PAN-OS GlobalProtect to gain initial access..."

context.store(
    [
        {"content": humint_summary,    "metadata": {"source": "HUMINT-2025-Q1-007"}},
        {"content": cti_report_text,   "metadata": {"source": "CTI_RPT_APT29_2025"}},
    ],
    extract_entities=True,
    extract_relationships=True,
    link_entities=True,
)

llm    = LiteLLM(model="anthropic/claude-sonnet-4-20250514")
result = context.query_with_reasoning(
    "Trace the C2 infrastructure chain for APT29 operations targeting "
    "ITAR-controlled contractors in 2025. Include IP ranges, ASNs, and TTPs.",
    llm_provider=llm,
    max_results=15,
    max_hops=3,
)

print(result["response"])
print("\n--- Reasoning Path ---")
print(result["reasoning_path"])
print("Confidence: {:.1%}".format(result["confidence"]))

# Anchor retrieval on APT29 for a proximity-weighted follow-up
proximate = context.retrieve(
    "C2 beaconing patterns Tor exit nodes",
    use_graph=True,
    anchor_node="APT29",
    proximity_weight=0.7,
    max_hops=3,
    max_results=10,
)
```

</Tab>

<Tab title="安全——SOC/事件">

安全运营：针对一张包含主机、CVE、用户账户、预案和历史事件的图谱进行实时告警分诊。GraphRAG 在一次调用中检索相关预案和相似的历史事件，缩短平均响应时间。

`decision_tracking=True` 标志把每一次分诊查询记录为一个可审计决策，连同提供给 LLM 的完整上下文——这对事后复盘和 SOC 指标至关重要。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import LiteLLM

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph()

soc_context = AgentContext(
    vector_store=vs,
    knowledge_graph=graph,
    graph_expansion=True,
    max_expansion_hops=2,
    hybrid_alpha=0.5,
    decision_tracking=True,
    retention_days=365,
)

runbooks = [
    "RB-001: Lateral movement — isolate source host, collect memory dump, "
    "escalate if EDR alert on LSASS access.",
    "RB-002: Ransomware precursor — block C2 range, snapshot affected volumes, "
    "engage IR team within 15 minutes.",
    "RB-003: Scheduled task persistence — review parent process, check Sigma "
    "T1053.005, quarantine if encoded payload confirmed.",
]
soc_context.store(runbooks, extract_entities=True)

alert_text = """
ALERT-2025-110342 [CRITICAL]
Host: dc01.corp.internal (10.10.1.5)
User: svc_backup (DOMAIN\\svc_backup)
Event: Scheduled task created — cmd.exe /c powershell -enc <base64>
Parent: wmiprvse.exe
Sigma match: T1053.005 Scheduled Task/Job
"""

llm    = LiteLLM(model="anthropic/claude-sonnet-4-20250514")
triage = soc_context.query_with_reasoning(
    "Triage this SIEM alert and identify the correct response runbook:\n{}".format(alert_text),
    llm_provider=llm,
    max_results=8,
    max_hops=2,
)

print("TRIAGE: {}".format(triage["response"]))
# TRIAGE: Based on the wmiprvse.exe parent spawning an encoded PowerShell scheduled task,
# this matches the persistence pattern in RB-003. Recommended action: review parent process
# chain, confirm encoded payload, quarantine dc01.corp.internal if confirmed...
print("Confidence: {:.1%}".format(triage["confidence"]))

# Also pull similar historical incidents for analyst context
similar = soc_context.retrieve(
    "wmiprvse.exe encoded powershell scheduled task persistence",
    use_graph=True,
    max_results=5,
)
for inc in similar:
    print("[{:.3f}] {}".format(inc["score"], inc["content"][:100]))
```

</Tab>

<Tab title="生命科学——临床/制药">

临床决策支持：把 FDA 药品说明书、临床指南和试验摘要摄取进一张图谱，其中药物-酶-代谢物-相互作用链成为可遍历的路径。一次三跳查询（药物 → 酶 → 代谢物 → 禁忌）会浮现任何单一文档都不会明示的相互作用风险。

把 `max_expansion_hops=3` 设为 3 是有意为之：从胺碘酮到升高华法林血浆水平的药代动力学链是 药物 → CYP2C9 抑制 → 华法林代谢降低 → 出血风险，正好是三个结构跳数。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import LiteLLM

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph()

clinical_context = AgentContext(
    vector_store=vs,
    knowledge_graph=graph,
    graph_expansion=True,
    max_expansion_hops=3,  # drug → enzyme → metabolite → interaction
    hybrid_alpha=0.55,
    retention_days=None,   # clinical records: no expiry
)

fda_label_text = (
    "Warfarin sodium: narrow therapeutic index anticoagulant. CYP2C9 is the "
    "primary metabolic pathway. Amiodarone is a potent CYP2C9 inhibitor..."
)

guideline_text = (
    "ESC 2023 AF Guidelines: bridging therapy with heparin is not recommended "
    "for most patients with AF undergoing elective procedures..."
)

clinical_context.store(
    [
        {"content": fda_label_text,  "metadata": {"source": "FDA_WARFARIN_LABEL_2024"}},
        {"content": guideline_text,  "metadata": {"source": "ESC_AF_GUIDELINE_2023"}},
    ],
    extract_entities=True,
    extract_relationships=True,
    link_entities=True,
)

patient_context = """
Patient: 68F, AF, CKD stage 3b (eGFR 32). On warfarin (INR target 2.0–3.0).
Presenting for elective hip replacement. Concurrent: amiodarone 200mg, atorvastatin 40mg.
"""

llm    = LiteLLM(model="anthropic/claude-sonnet-4-20250514")
answer = clinical_context.query_with_reasoning(
    "What is the evidence-based warfarin bridging protocol for this patient "
    "given CKD and amiodarone interaction risk?\n\n{}".format(patient_context),
    llm_provider=llm,
    max_results=12,
    max_hops=3,
)

print(answer["response"])
print("Evidence sources: {}".format(answer["num_sources"]))
print("Reasoning hops:   {}".format(answer["num_reasoning_paths"]))

# Pull the contraindication chain explicitly
contra_chain = clinical_context.retrieve(
    "CYP2C9 inhibition amiodarone warfarin bleeding risk",
    use_graph=True,
    anchor_node="warfarin",
    proximity_weight=0.65,
    max_hops=3,
    max_results=6,
)
```

</Tab>

<Tab title="银行——风险/合规">

监管合规：把 Basel III（CRE20）、BCBS 239、SR 11-7 和 EBA IRRBB 指南摄取为一张图谱，其中监管条款之间互相交叉引用作为边。多跳查询会自动遍历这些交叉引用，因此一个关于商业地产 RWA 的查询能在一次调用中拉取相关的 CRE20 段落以及管辖其计算的 BCBS 239 数据质量要求。

`reasoning_path` 输出可直接作为监管机构要求的审计轨迹，证明某项资本计算有引用的监管文本作为依据。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import LiteLLM

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph()

compliance_context = AgentContext(
    vector_store=vs,
    knowledge_graph=graph,
    graph_expansion=True,
    max_expansion_hops=2,
    hybrid_alpha=0.5,
    retention_days=2555,   # 7-year regulatory retention
)

# In production these come from ingest_file() — shown as strings here for brevity
basel_cre20_text  = "CRE20.32: For income-producing real estate where repayment depends on "
                    "property cash flows, RWA = exposure × risk weight, where risk weight "
                    "is determined by LTV bucket per Table CRE20.3..."
bcbs239_text      = "Principle 3: Risk data should be accurate and have a single authoritative source. "
                    "Where data is aggregated across systems, reconciliation must be documented..."

compliance_context.store(
    [
        {"content": basel_cre20_text, "metadata": {"source": "BCBS_CRE20_2024"}},
        {"content": bcbs239_text,     "metadata": {"source": "BCBS239_2013"}},
    ],
    extract_entities=True,
    extract_relationships=True,
)

llm    = LiteLLM(model="anthropic/claude-sonnet-4-20250514")
answer = compliance_context.query_with_reasoning(
    "Under Basel III CRE20, what are the RWA calculation requirements for "
    "commercial real estate exposures with LTV > 80%? "
    "Cross-reference with BCBS 239 data quality requirements.",
    llm_provider=llm,
    max_results=12,
    max_hops=2,
)

print(answer["response"])
print("Regulatory sources cited: {}".format(answer["num_sources"]))
print("Confidence: {:.1%}".format(answer["confidence"]))

# The reasoning path is the audit log — show it to the regulator
print("\n--- Reasoning Path (audit log) ---")
print(answer["reasoning_path"])
```

</Tab>

</Tabs>

<a id="common-pitfalls"></a>
## 常见陷阱

**跳数过大。** 把 `max_expansion_hops` 设得太高（>4）会产生指数级增长的上下文，淹没 LLM 并增加成本。从 2-3 跳开始，仅在需要时再增加。

**图谱质量差。** GraphRAG 会放大图谱质量问题。重复实体、命名不一致和薄弱的关系会产生糟糕的检索结果。在依赖 GraphRAG 处理重要查询之前，先清理你的图谱数据。

**重复实体。** 把 "APT-29"、"APT29" 和 "Cozy Bear" 作为分开的节点会破坏关系遍历。摄取期间的实体链接有所帮助，但可能仍需人工去重。

**对简单查找查询使用 GraphRAG。** 如果你已知答案存在于某份特定文档中且只需检索它，传统向量搜索比 GraphRAG 更快、更简单。

**假定图扩展总是有益的。** 更多上下文并不总是更好。有时精确、聚焦的检索胜过宽泛的图扩展。请针对你的具体用例测试两种方式。

<a id="tuning-the-vector-graph-balance"></a>
## 调节向量-图谱平衡

在 `AgentContext` 构造函数中设置的 `hybrid_alpha` 参数确立了向量相似度与图影响力的默认混合比例。`0.0` 是纯向量检索；`1.0` 是纯图遍历。推荐的起点是 `0.5`。

当针对某个具体 `anchor_node` 时，你可以在 `retrieve()` 中应用 `proximity_weight`，把与锚点的结构距离动态融合进最终得分：

```python
# Anchor node provided — let vector semantics lead, graph proximity only slightly boosts
results = context.retrieve(
    query, use_graph=True, anchor_node="APT29", proximity_weight=0.2
)

# Known-entity tracing — topology drives the retrieval
results = context.retrieve(
    query, use_graph=True, anchor_node="APT29", proximity_weight=0.8
)
```

`max_hops` 中每多一跳都会指数级增大子图规模。按领域的实用默认值：

```text
General Q&A             max_expansion_hops=2  (95% of useful facts within 2 hops)
Threat intel (APT)      max_expansion_hops=3  (actor → infra → victim → attribution)
Drug interactions       max_expansion_hops=3  (drug → enzyme → metabolite → interaction)
Regulatory cross-ref    max_expansion_hops=2  (rule → article → article)
```

在构造函数中全局设置；通过向 `retrieve()` 传入 `max_hops` 参数可按调用覆盖。

<a id="how-graphrag-works-internally"></a>
## GraphRAG 的内部工作原理

```text
Query text
    |
    v
Vector embedding  ─────────────────────────────────────┐
    |                                                   |
    v                                                   v
Semantic search                         Graph traversal (BFS)
(FAISS / Qdrant)                        from anchor / top-k seeds
    |                                                   |
    └──────────┐                ┌──────────────────────┘
               v                v
         Score fusion (proximity_weight blend)
               |
               v
          Ranked subgraph
               |
               v
         LLM grounding  <── query_with_reasoning()
               |
               v
     {response, reasoning_path, sources, confidence}
```

向量搜索和图遍历独立运行，然后融合各自得分。图遍历使用从向量搜索识别出的种子节点进行广度优先扩展，因此图部分始终锚定在语义相关性上，而不是盲目探索整张图。

<a id="related-guides"></a>
## 相关指南

- [语义抽取](semantic-extraction.zh-CN.md)——从原始非结构化文本构建图谱
- [智能体记忆](agent-memory.zh-CN.md)——存储、检索并持久化智能体记忆
- [上下文图谱](context-graphs.zh-CN.md)——直接构建并遍历知识图谱
- [推理](reasoning.zh-CN.md)——在图谱上推导新事实并运行推理规则
- [决策智能](decision-intelligence.zh-CN.md)——因果链、策略执行、决策追踪
- [LLM 集成](llm-integrations.zh-CN.md)——连接 Groq、OpenAI、Anthropic、HuggingFace 等 100+ 提供商
