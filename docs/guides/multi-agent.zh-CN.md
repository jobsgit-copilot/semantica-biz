---
title: "多智能体系统"
description: "通过共享内存、知识图谱和决策历史协调多个 AI 智能体 —— 无需消息代理。"
---

**[English](multi-agent.md)** · **简体中文（当前）**

<a id="what-is-multi-agent-coordination"></a>
## 什么是多智能体协作？

多智能体系统是一种软件架构，其中多个自主智能体协同工作，完成单个智能体难以有效完成的复杂任务。开发者不再构建一个试图包揽一切的单体智能体，而是将工作拆分给若干专注于特定职责的专职智能体。

**为什么要将工作拆分给多个智能体：**
- **关注点分离** —— 每个智能体专精于一个领域（摄取、分析、报告），而不是试图精通一切
- **独立的推理** —— 不同智能体可以使用针对其特定任务优化的不同模型、提示词和推理策略
- **并行处理** —— 多个智能体可以同时对同一问题的不同方面进行处理
- **类人的工作流分解** —— 模拟人类团队自然地拆分复杂分析工作的方式

**Semantica 的协作方式：**
Semantica 通过共享上下文（内存和知识图谱）来协调智能体，而不是通过消息代理或服务之间的 API 调用。智能体读写相同的基础数据结构，从而无需复杂的中间件即可无缝共享信息。

**单智能体与多智能体架构：**
- **单智能体** —— 一个 `AgentContext` 处理从摄取到最终输出的所有任务
- **多智能体** —— 多个 `AgentContext` 实例或命名空间化的工作流，各自负责特定的流水线阶段或分析角色

<a id="why-use-multi-agent-systems"></a>
## 为什么使用多智能体系统？

**职责分离。** 将复杂工作流拆分为专注、可控的阶段，每个智能体在各自的专业领域表现出色，而不会被旁支事务淹没。

**复杂工作流的可扩展性。** 处理需要不同专业领域、处理速度和推理方式的复杂分析流水线，而不会产生臃肿的单体智能体。

**独立的推理阶段。** 让不同智能体使用针对各自任务优化的不同 LLM、提示词、置信度阈值和推理策略，而不是在一套"放之四海而皆准"的方案上妥协。

**专职的智能体角色。** 创建为摄取、富化、分析、综合和报告量身定制的智能体，每个智能体都有与其角色匹配的配置和能力。

**共享知识与证据。** 多个智能体向同一个知识图谱和内存存储贡献并从中受益，形成一个随着更多智能体贡献发现而不断改进的累积证据库。

**类人的工作流分解。** 映射自然的人类团队结构 —— 分析师、研究员和决策者各自为协作分析过程贡献专业能力。

<a id="when-to-use-when-not-to-use"></a>
## 适用场景 / 不适用场景

**在以下场景使用多智能体系统：**
- 需要多个阶段的复杂分析工作流（研究 → 分析 → 综合 → 报告）
- 具有不同阶段、能从专职处理中受益的多阶段处理流水线
- 不同智能体处理不同信息源或分析方法的研究与调查工作流
- 担任不同角色的专职智能体团队（OSINT 采集员、富化分析师、情报融合官）
- 不同智能体可能在不同时间或按不同计划运行的长期运行工作流
- 不同分析阶段需要不同 LLM、推理方式或置信度阈值的场景

**不要在以下场景使用多智能体系统：**
- 简单的文档摘要或单步信息检索任务
- 一个智能体就能有效处理所有步骤而无需专职化的线性工作流
- 协调开销超过核心工作复杂度的小型、直接的任务
- 单个智能体加上适当配置就能高效处理整个工作流的场景

**重要考量：** 多智能体系统会引入额外的架构复杂性，包括状态管理、协作模式和调试挑战。只有当专职化和关注点分离带来的收益超过这些额外复杂性时，才选择多智能体方案。

Semantica 通过共享的 `ContextGraph` 协调多个智能体 —— 智能体读写同一个图谱，或通过 `save()` 和 `load()` 交接序列化的状态，无需消息代理。当需要将工作拆分到摄取、富化、推理和报告等角色，且这些角色必须共享同一个证据库时，请使用这种模式。

<Info>
  本指南介绍多智能体协作。关于每个智能体内部使用的内存层，请参见[智能体内存](agent-memory.zh-CN.md)。关于图遍历和实体链接，请参见[上下文图谱](context-graphs.zh-CN.md)。关于决策记录和先例匹配，请参见[决策智能](decision-intelligence.zh-CN.md)。
</Info>

<a id="the-three-coordination-patterns"></a>
## 三种协作模式

在编写任何代码之前，请为你的流水线选择合适的协作模式。

**共享图谱模式：** 多个智能体在单个进程内共享对同一个 `ContextGraph` 和 `VectorStore` 对象的引用。这提供了最低的延迟，因为所有智能体能立即看到变更，并具有内置的线程安全以支持并发访问。当智能体在同一应用中同时运行，并且需要实时访问彼此的贡献时，请选择此模式。

**保存/加载交接模式：** 智能体运行在不同的进程、容器或不同时间。第一个智能体完成工作后调用 `context.save(path)` 序列化其完整状态。下一个智能体调用 `context.load(path)`，在上一个智能体停止的确切位置恢复，包括完整的内存、图谱数据和向量索引。当用于分布式系统、定时工作流，或智能体运行在需要共享存储访问的不同机器上时，请选择此模式。

**命名空间内存模式：** 单个 `AgentContext` 服务于多个逻辑智能体，每个智能体使用唯一的 `conversation_id` 值来限定自己的读写范围。智能体通过命名空间而非独立上下文实例进行隔离。当需要轻量级的角色分离，而不想承担维护多个完整上下文的资源开销时，请选择此模式。

本指南中的流水线同时使用了这三种模式。

<a id="pattern-1-shared-graph-for-concurrent-ingestion"></a>
## 模式 1 —— 用于并发摄取的共享图谱

OSINT（**开源情报** —— 公开可获取的信息）采集器和富化智能体并发运行。它们共享一个 `ContextGraph` 和一个 `VectorStore` —— 图谱内部的 `RLock` 使并发写入安全。

```python
import threading
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

# 一个图谱、一个向量库 —— 两个智能体都向它们写入
shared_graph = ContextGraph(advanced_analytics=True)
shared_vs    = VectorStore(backend="faiss", dimension=768)

def make_agent() -> AgentContext:
    """Factory: each agent gets its own AgentContext wrapping the shared backing stores."""
    return AgentContext(
        vector_store=shared_vs,            # same VectorStore instance
        knowledge_graph=shared_graph,      # same ContextGraph instance
        graph_expansion=True,
        max_expansion_hops=2,
        decision_tracking=True,
    )

osint_agent      = make_agent()
enrichment_agent = make_agent()
reasoning_agent  = make_agent()
```

OSINT 采集器的工作是摄取原始情报源并抽取实体。它不做推理 —— 只是摄取，让图谱累积结构。

```python
def osint_collection():
    """Agent 1: ingest raw threat feeds and CVE data."""
    osint_agent.store(
        [
            {
                "content": "APT29 exploits CVE-2024-3400 in PAN-OS GlobalProtect — unauthenticated RCE, CVSS 10.0",
                "metadata": {"source": "nvd_feed", "cve": "CVE-2024-3400", "actor": "APT29"},
            },
            {
                "content": "Volexity confirms active exploitation of CVE-2024-3400 against NATO member networks",
                "metadata": {"source": "volexity_blog", "actor": "APT29", "target": "NATO"},
            },
            {
                "content": "PAN-OS GlobalProtect affected versions: < 10.2.9-h1, < 11.0.4-h1, < 11.1.2-h3",
                "metadata": {"source": "paloalto_advisory", "cve": "CVE-2024-3400", "product": "GlobalProtect"},
            },
        ],
        extract_entities=True,
        extract_relationships=True,
        conversation_id="osint-pipeline",    # 命名空间作为智能体标识
    )
```

当 OSINT 采集器运行时，富化智能体独立地拉取攻击者画像数据，并将其链接到同一个图谱中。

```python
def enrichment():
    """Agent 2: enrich the graph with actor profile and TTP context."""
    enrichment_agent.store(
        [
            {
                "content": "APT29 TTP profile: T1190 (Exploit Public-Facing Application), T1071.001 (Web Protocols C2), T1078 (Valid Accounts)",
                "metadata": {"source": "mitre_attck", "actor": "APT29", "type": "ttp_profile"},
            },
            {
                "content": "APT29 infrastructure fingerprint: use of Cloudflare Workers for C2 relay, certificate reuse across campaigns",
                "metadata": {"source": "recorded_future", "actor": "APT29", "type": "infrastructure"},
            },
        ],
        extract_entities=True,
        extract_relationships=True,
        conversation_id="enrichment-pipeline",    # 与 OSINT 智能体分离的命名空间
    )
```

并发运行两者 —— 图谱会处理加锁。

```python
t1 = threading.Thread(target=osint_collection)
t2 = threading.Thread(target=enrichment)
t1.start(); t2.start()
t1.join();  t2.join()

# 共享图谱现在包含来自两个智能体的实体和关系。
# 推理智能体可以跨两个智能体存储的所有内容进行查询。
```

<a id="pattern-2-save-load-handoff-to-a-reasoning-agent"></a>
## 模式 2 —— 通过保存/加载交接给推理智能体

推理智能体在摄取完成后运行。在生产流水线中，它可能是一个独立的进程、一个不同的容器或一个定时任务。摄取智能体保存它们的共享状态；推理智能体加载这些状态。

**重要的部署说明：** 当智能体运行在不同的容器或不同的机器上时，它们必须通过共享存储（网络文件系统、云存储或共享卷）访问同一个已保存的状态位置。

```python
# 摄取完成后：保存合并的图谱和向量索引
osint_agent.save("./pipeline/enriched_intel/")
# 写入：
#   pipeline/enriched_intel/agent_memory.json     — 所有 MemoryItem
#   pipeline/enriched_intel/vector_store/         — FAISS 索引
#   pipeline/enriched_intel/knowledge_graph.json  — 所有节点和边

print("Ingestion complete. State saved for reasoning agent.")
```

推理智能体全新启动，加载状态，并拥有对两个摄取智能体所构建一切的完整访问权限。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import LiteLLM

# 创建一个上下文用于加载检查点 —— load() 会覆盖现有状态
reasoning_vs    = VectorStore(backend="faiss", dimension=768)
reasoning_graph = ContextGraph(advanced_analytics=True)
reasoning_agent = AgentContext(
    vector_store=reasoning_vs,
    knowledge_graph=reasoning_graph,
    graph_expansion=True,
    max_expansion_hops=3,
    decision_tracking=True,
)

reasoning_agent.load("./pipeline/enriched_intel/")
# 来自两个摄取智能体的所有记忆、图谱节点和向量嵌入现在都可用了。

# 在综合步骤中使用一个高能力模型
llm = LiteLLM(model="anthropic/claude-sonnet-4-20250514")

synthesis = reasoning_agent.query_with_reasoning(
    "Summarize the APT29 exploitation of CVE-2024-3400: affected products, "
    "observed TTPs, targeted sectors, and recommended mitigations.",
    llm_provider=llm,
    max_results=15,
    max_hops=3,
)

print(synthesis["response"])
print("Confidence: {:.0%}".format(synthesis["confidence"]))

# 将综合结果存回图谱 —— 报告智能体会检索它
reasoning_agent.store(
    "SYNTHESIS: " + synthesis["response"],
    metadata={"type": "synthesis", "agent": "reasoning", "confidence": synthesis["confidence"]},
    conversation_id="synthesis-output",
)

# 将该分析判断记录为可追溯的决策
reasoning_agent.record_decision(
    category="threat_assessment",
    scenario="APT29 active exploitation of CVE-2024-3400 in PAN-OS",
    reasoning=synthesis["reasoning_path"],
    outcome="high_priority_patch_advisory",
    confidence=synthesis["confidence"],
    entities=["APT29", "CVE-2024-3400", "GlobalProtect", "NATO"],
    decision_maker="reasoning_agent_v2",
)

# 交接给报告智能体
reasoning_agent.save("./pipeline/synthesis_output/")
```

<Info>
  `load()` 会覆盖现有上下文 —— 它会在加载之前清除当前内存、图谱和向量状态。在调用 `load()` 之前，上下文中任何未保存的数据都会丢失。
</Info>

<a id="pattern-3-namespaced-memories-for-role-separation"></a>
## 模式 3 —— 用于角色分离的命名空间记忆

报告智能体不需要自己的图谱实例。它共享推理智能体的上下文，但将写入限定在自己的命名空间内 —— `conversation_id` 作为智能体标识，用于分离记忆流并防止不同逻辑智能体之间的污染。

**使用 conversation_id 的命名空间隔离：**
- `conversation_id` 在同一个 `AgentContext` 内创建独立的记忆命名空间
- 每个智能体的记忆保持隔离，除非显式跨命名空间查询
- 当不同逻辑智能体处理相关但不同的任务时，防止意外的记忆污染

```python
# 报告智能体加载综合输出
reporting_vs    = VectorStore(backend="faiss", dimension=768)
reporting_graph = ContextGraph()
reporting_agent = AgentContext(
    vector_store=reporting_vs,
    knowledge_graph=reporting_graph,
    graph_expansion=True,
)
reporting_agent.load("./pipeline/synthesis_output/")

# 检索推理智能体产生的所有内容
synthesis_items = reporting_agent.retrieve(
    "APT29 CVE-2024-3400 threat assessment synthesis",
    max_results=10,
    conversation_id="synthesis-output",   # 限定在推理智能体的输出范围内
)

# 构建最终简报
brief_sections = []
for item in synthesis_items:
    brief_sections.append(item["content"])

# 将最终报告存储在报告智能体自己的命名空间下
reporting_agent.store(
    "\n\n".join(brief_sections),
    metadata={"type": "finished_report", "classification": "TLP:GREEN"},  # TLP（交通灯协议）—— 信息共享准则
    conversation_id="reporting-output",    # 报告智能体的命名空间
    user_id="reporting_agent",
)

# 完整的流水线审计追踪：跨所有命名空间检索
full_trail = reporting_agent.retrieve("APT29 CVE-2024-3400", max_results=25)
print("Pipeline produced {} traceable context items".format(len(full_trail)))
```

通过按 `conversation_id` 过滤，可以单独检索每个智能体的贡献；通过不加过滤地查询，则可以集体检索。

<a id="common-pitfalls"></a>
## 常见陷阱

**忘记使用 conversation_id 命名空间。** 如果没有唯一的 `conversation_id` 值，不同智能体的记忆会混在一起，使得无法追溯哪个智能体贡献了哪些见解。请始终为每个逻辑智能体使用不同且有意义的 conversation ID。

**使用 load() 导致意外状态丢失。** `load()` 函数会覆盖现有上下文而不是合并它。如果你的 `AgentContext` 中有未保存的状态，调用 `load()` 会将其清除。在加载检查点之前，请始终保存当前状态或使用一个全新的上下文。

**跨独立进程使用共享图谱。** 共享图谱模式只在智能体共享对象引用的单个进程内有效。对于运行在不同容器或机器上的分布式智能体，请改用保存/加载交接模式。

**假设 save/load 无需共享存储即可工作。** 在不同进程、容器或机器中的智能体必须能访问同一个文件系统位置才能进行 save/load 交接。请确保共享存储（NFS、云存储、共享卷）已正确配置。

**用多个智能体过度工程化简单工作流。** 多智能体系统会增加协调复杂性和潜在故障点。对于直接的单步任务，简单的单智能体方案通常更可靠、更易调试。

**过度混淆智能体职责。** 每个智能体都应该有清晰、聚焦的角色。试图承担太多不同任务的智能体会失去专职化的好处，变得更难优化、调试和维护。

**忽视内存隔离边界。** 使用命名空间内存时，要小心跨多个 `conversation_id` 值的查询。未限定范围的查询可能会意外检索到其他智能体的记忆，破坏逻辑隔离。

<a id="domain-examples"></a>
## 领域示例

<Tabs>
<Tab title="国防 — CTI/威胁">
一个三智能体情报融合小组：OSINT 采集器摄取公开情报源，HUMINT（**人力情报** —— 从人类来源收集的信息）分析师加载机密摘要，融合官将两路情报综合成 PIR（**优先情报需求** —— 决策所需的关键信息）答案。OSINT 和 HUMINT 智能体在共享图谱上并发运行；融合官在一个**隔离网络环境**（没有互联网连接的隔离网络，用于安全目的）的独立进程中加载合并后的状态。

```python
import threading
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import HuggingFaceLLM

# 用于并发多源情报采集的共享图谱
shared_graph = ContextGraph(advanced_analytics=True)
shared_vs    = VectorStore(backend="faiss", dimension=768)

osint_agent  = AgentContext(vector_store=shared_vs, knowledge_graph=shared_graph, graph_expansion=True)
humint_agent = AgentContext(vector_store=shared_vs, knowledge_graph=shared_graph, graph_expansion=True)

def osint_collection():
    osint_agent.store(
        [
            {"content": "CVE-2024-3400 confirmed exploited by APT29 against NATO member VPN gateways",
             "metadata": {"source": "NVD", "classification": "UNCLASSIFIED", "actor": "APT29"}},
            {"content": "Palo Alto PSIRT: GlobalProtect OS command injection via crafted SESSID cookie",
             "metadata": {"source": "PAN-SA-2024-0006", "classification": "UNCLASSIFIED"}},
        ],
        extract_entities=True,
        extract_relationships=True,
        conversation_id="osint-collector",
    )

def humint_analysis():
    # 在已授权的环境中，HUMINT 文档来自本地的机密库
    humint_agent.store(
        [
            {"content": "[S//NF] APT29 operator tradecraft: deploy WARPWIRE credential harvester post-exploitation of perimeter VPNs",
             "metadata": {"source": "HUMINT_Q4_2024", "classification": "SECRET//NOFORN", "actor": "APT29"}},
            {"content": "[S//NF] Target selection pattern: APT29 prioritizes Foreign Ministry and Defense Attache networks within NATO",
             "metadata": {"source": "HUMINT_Q4_2024", "classification": "SECRET//NOFORN", "actor": "APT29"}},
        ],
        extract_entities=True,
        extract_relationships=True,
        conversation_id="humint-analyst",
    )

# 并发的多源情报采集 —— 图谱处理线程安全
t1 = threading.Thread(target=osint_collection)
t2 = threading.Thread(target=humint_analysis)
t1.start(); t2.start()
t1.join();  t2.join()

# 为隔离网络环境的融合官保存合并后的情报库
osint_agent.save("./fusion/combined_intel/")

# --- 融合官（隔离网络段，独立进程） ---
fusion_vs    = VectorStore(backend="faiss", dimension=768)
fusion_graph = ContextGraph(advanced_analytics=True)
fusion_officer = AgentContext(
    vector_store=fusion_vs,
    knowledge_graph=fusion_graph,
    graph_expansion=True,
    decision_tracking=True,
)
fusion_officer.load("./fusion/combined_intel/")

# 隔离网络环境下的推理：NFS 共享上的本地模型
llm = HuggingFaceLLM(model="/opt/models/llama-3.1-70b-instruct")

pir_answer = fusion_officer.query_with_reasoning(
    "PIR: What is APT29's current exploitation methodology against NATO perimeter VPNs "
    "and what post-exploitation capabilities have they deployed in Q4 2024?",
    llm_provider=llm,
    max_results=20,
    max_hops=3,
)
print(pir_answer["response"])
fusion_officer.save("./fusion/pir_report/")
```

</Tab>

<Tab title="安全 — SOC/事件">
一个三层 SOC 流水线：Tier 1 使用快速的 Groq 模型对告警进行分诊，当 Tier 1 置信度较低时 Tier 2 用深度的 Claude 分析进行升级，而一个管理者智能体读取跨所有层级的完整事件线程。三个层级共享一个 `ContextGraph` —— 每个层级用按层级限定的 `conversation_id` 标记各自的发现。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import Groq, LiteLLM

shared_graph = ContextGraph()
shared_vs    = VectorStore(backend="faiss", dimension=768)

def make_soc_agent() -> AgentContext:
    return AgentContext(
        vector_store=shared_vs,
        knowledge_graph=shared_graph,
        graph_expansion=True,
        decision_tracking=True,
    )

tier1   = make_soc_agent()
tier2   = make_soc_agent()
manager = make_soc_agent()

incident_id = "INC-2025-110342"

# --- Tier 1：使用 Groq 快速分诊（目标 < 500ms） ---
fast_llm = Groq(model="llama-3.1-8b-instant", api_key="YOUR_GROQ_KEY")

alert = (
    "Host: ws-finance-03  User: jsmith  Event: Scheduled task — base64-encoded PowerShell\n"
    "Sigma: T1053.005  Parent: wmiprvse.exe  Time: 2025-06-21T09:14:32Z"
)
tier1.store(alert, metadata={"tier": 1, "incident": incident_id})

triage = tier1.query_with_reasoning(
    "Is this a true positive? One-line verdict.",
    llm_provider=fast_llm,
    max_results=5,
)
tier1.store(
    "TIER1 VERDICT: " + triage["response"],
    metadata={"tier": 1, "incident": incident_id, "confidence": triage["confidence"]},
    conversation_id="{}-tier1".format(incident_id),
)

# --- Tier 2：当 Tier 1 置信度较低时进行深度调查 ---
if triage["confidence"] < 0.90:
    deep_llm = LiteLLM(model="anthropic/claude-sonnet-4-20250514")

    investigation = tier2.query_with_reasoning(
        "Full MITRE ATT&CK analysis of incident {}. "
        "Identify attack chain, affected systems, blast radius, and recommended containment.".format(incident_id),
        llm_provider=deep_llm,
        max_results=15,
        max_hops=3,
    )
    tier2.store(
        "TIER2 ANALYSIS: " + investigation["response"],
        metadata={"tier": 2, "incident": incident_id},
        conversation_id="{}-tier2".format(incident_id),
    )
    tier2.record_decision(
        category="escalation",
        scenario="Escalate {} — Tier 1 confidence {:.0%}".format(incident_id, triage["confidence"]),
        reasoning=investigation["reasoning_path"],
        outcome="escalated_to_tier3",
        confidence=investigation["confidence"],
        entities=["ws-finance-03", "jsmith", "wmiprvse.exe"],
        decision_maker="tier2_analyst",
    )

# --- 管理者：读取跨所有层级的完整事件线程 ---
full_thread = manager.retrieve(
    "incident {}".format(incident_id),
    use_graph=True,
    max_results=25,
)
print("Full incident thread ({} items):".format(len(full_thread)))
for item in full_thread:
    tier = item.get("metadata", {}).get("tier", "?")
    print("  [Tier {}] {}".format(tier, item["content"][:100]))

# 用于事后复盘的各层级历史
t1_history = manager.conversation("{}-tier1".format(incident_id))
t2_history = manager.conversation("{}-tier2".format(incident_id))
```

</Tab>

<Tab title="生命科学 — 临床/制药">
一个三智能体药物发现流水线：文献智能体摄取 PubMed 论文，实验智能体加载已验证的实验结果，首席智能体综合两者以识别先导化合物。文献和实验智能体在共享图谱上并行运行；首席智能体在摄取完成后跨两者进行查询。

```python
import threading
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import LiteLLM

shared_graph = ContextGraph(advanced_analytics=True)
shared_vs    = VectorStore(backend="faiss", dimension=768)

def make_agent() -> AgentContext:
    return AgentContext(
        vector_store=shared_vs,
        knowledge_graph=shared_graph,
        graph_expansion=True,
        max_expansion_hops=3,
        retention_days=None,      # 研究数据无限期保留
    )

lit_agent = make_agent()
exp_agent = make_agent()
chief     = make_agent()

def literature_review():
    # 摄取 KRAS G12C 抑制剂的 PubMed 摘要
    lit_agent.store(
        [
            {"content": "Sotorasib (AMG-510) achieves 37.1% ORR in KRAS G12C NSCLC (CodeBreaK 100, NEJM 2021)",
             "metadata": {"source": "CodeBreaK100", "target": "KRAS_G12C", "compound": "sotorasib"}},
            {"content": "Adagrasib (MRTX849) ORR 42.9% in KRAS G12C NSCLC with CNS activity (KRYSTAL-1, NEJM 2022)",
             "metadata": {"source": "KRYSTAL-1", "target": "KRAS_G12C", "compound": "adagrasib"}},
            {"content": "Resistance to KRAS G12C inhibitors frequently driven by Y96D mutation in switch-II pocket",
             "metadata": {"source": "Tanaka_CancerCell_2021", "target": "KRAS_G12C", "mechanism": "resistance"}},
        ],
        extract_entities=True,
        extract_relationships=True,
        conversation_id="literature-agent",
    )

def experimental_results():
    # 加载已验证的体外和体内实验数据
    exp_agent.store(
        [
            {"content": "Compound RMC-6291: IC50=0.6nM KRAS G12C, selectivity index 480, in-vivo efficacy 82%, toxicity grade 1",
             "metadata": {"source": "internal_assay", "compound": "RMC-6291", "source_type": "experimental"}},
            {"content": "Compound BI-7273: IC50=1.4nM KRAS G12C, selectivity index 310, in-vivo efficacy 71%, toxicity grade 2",
             "metadata": {"source": "internal_assay", "compound": "BI-7273", "source_type": "experimental"}},
            {"content": "Compound GDC-6036: IC50=0.9nM KRAS G12C, active against Y96D resistance mutation, toxicity grade 1",
             "metadata": {"source": "internal_assay", "compound": "GDC-6036", "source_type": "experimental"}},
        ],
        extract_entities=True,
        conversation_id="experimental-agent",
    )

# 并行摄取 —— 图谱处理并发
t1 = threading.Thread(target=literature_review)
t2 = threading.Thread(target=experimental_results)
t1.start(); t2.start()
t1.join();  t2.join()

# 首席智能体跨文献和实验数据进行综合
llm = LiteLLM(model="anthropic/claude-sonnet-4-20250514")

synthesis = chief.query_with_reasoning(
    "Identify the top two candidate compounds for KRAS G12C NSCLC that show "
    "both strong experimental IC50 selectivity and clinical/literature support "
    "for the target pathway, including any coverage of resistance mechanisms.",
    llm_provider=llm,
    max_results=20,
    max_hops=3,
)
print(synthesis["response"])
chief.save("./drug_discovery/kras_g12c_checkpoint/")
```

</Tab>

<Tab title="银行业 — 风险/合规">
一个四智能体信贷委员会：风险台智能体计算 PD/LGD，合规台智能体检查 Basel III 和 EBA 要求，信贷官智能体应用策略，委员会主席智能体跨三个台读取以产出带完整审计追踪的最终决策。每个台用命名空间标记各自发现；主席跨所有命名空间读取。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import LiteLLM

shared_graph = ContextGraph(advanced_analytics=True)
shared_vs    = VectorStore(backend="faiss", dimension=768)

def make_desk_agent() -> AgentContext:
    return AgentContext(
        vector_store=shared_vs,
        knowledge_graph=shared_graph,
        graph_expansion=True,
        decision_tracking=True,
        retention_days=2555,    # Basel III 七年保留期
        kg_algorithms=True,
    )

risk_desk       = make_desk_agent()
compliance_desk = make_desk_agent()
credit_officer  = make_desk_agent()
committee_chair = make_desk_agent()

app_id = "LOAN-2025-88421"
llm    = LiteLLM(model="anthropic/claude-sonnet-4-20250514")

# --- 风险台：PD/LGD/EL 分析 ---
risk_desk.store(
    "Risk analysis {}: PD=2.3%, LGD=45%, EL=89k GBP, DSCR=1.12, LTV=78%. "
    "Stress test +300bps: DSCR falls to 0.98 — marginal but within policy floor of 0.95.".format(app_id),
    metadata={"desk": "risk", "application": app_id},
    conversation_id="{}-risk".format(app_id),
)
risk_desk.record_decision(
    category="credit_risk",
    scenario="Risk assessment {}".format(app_id),
    reasoning="PD 2.3% within 3% internal threshold; LTV 78% marginal — LMI required; stress test passes",
    outcome="conditional_approval_lmi_required",
    confidence=0.78,
    entities=[app_id, "LTV_78pct", "PD_2pct"],
    decision_maker="risk_model_v6",
)

# --- 合规台：Basel III / EBA GL 2020/06 检查 ---
compliance_desk.store(
    [
        {"content": "Basel III CRR2 Art. 92: total capital ratio minimum 8% + 2.5% conservation buffer",
         "metadata": {"source": "CRR2_Art92", "category": "capital_requirement"}},
        {"content": "EBA GL 2020/06: DSTI > 40% requires enhanced creditworthiness assessment and senior credit officer sign-off",
         "metadata": {"source": "EBA_GL_2020_06", "category": "affordability"}},
        {"content": "CRE20: LTV > 80% for residential mortgages requires LMI or equivalent credit enhancement",
         "metadata": {"source": "Basel_CRE20", "category": "collateral"}},
    ],
    extract_entities=True,
    extract_relationships=True,
)

compliance_check = compliance_desk.query_with_reasoning(
    "Does application {} (residential mortgage, LTV 78%, DSTI 35%) satisfy "
    "Basel III CRE20, EBA GL 2020/06 affordability requirements, and CRR2 Art. 92?".format(app_id),
    llm_provider=llm,
    max_results=10,
)
compliance_desk.store(
    "COMPLIANCE FINDING: " + compliance_check["response"],
    metadata={"desk": "compliance", "application": app_id, "confidence": compliance_check["confidence"]},
    conversation_id="{}-compliance".format(app_id),
)

# --- 委员会主席：跨台综合和最终决策 ---
committee_decision = committee_chair.query_with_reasoning(
    "Summarize all desk findings for application {} and produce the final "
    "credit committee decision with all conditions stated explicitly.".format(app_id),
    llm_provider=llm,
    max_results=25,
    max_hops=3,
)
print(committee_decision["response"])

# 记录最终决策 —— 这是监管机构看到的内容
committee_chair.record_decision(
    category="credit_committee",
    scenario="Final committee decision {}".format(app_id),
    reasoning=committee_decision["reasoning_path"],
    outcome="approved_with_conditions_lmi",
    confidence=committee_decision["confidence"],
    entities=[app_id, "LTV_78pct", "DSTI_35pct"],
    decision_maker="credit_committee_2025",
)

# 各台审计检索
risk_thread       = committee_chair.conversation("{}-risk".format(app_id))
compliance_thread = committee_chair.conversation("{}-compliance".format(app_id))

committee_chair.save("./credit_files/{}/context/".format(app_id))
```

</Tab>
</Tabs>

<a id="memory-isolation-reference"></a>
## 记忆隔离参考

当多个智能体写入一个共享上下文时，使用 `conversation_id` 来隔离它们的流，并单独检索。

```python
# 将一条记忆标记到某个智能体的命名空间
context.store("Finding: lateral movement confirmed", conversation_id="tier2-ir")

# 仅检索该智能体的记忆
tier2_history = context.retrieve("lateral movement", conversation_id="tier2-ir")

# 获取某个命名空间的完整有序历史
full_history = context.conversation("tier2-ir", max_items=100)

# 删除某个智能体的整个命名空间
context.forget(conversation_id="tier2-ir")
```

同样的模式也适用于按用户范围的隔离 —— 将 `conversation_id` 替换为 `user_id`：

```python
context.store("...", user_id="analyst-jsmith")
context.retrieve("...", user_id="analyst-jsmith")
```

<a id="related-guides"></a>
## 相关指南

- [智能体内存](agent-memory.zh-CN.md) —— 每个智能体内部使用的记忆存储、检索、持久化和工作记忆窗口
- [上下文图谱](context-graphs.zh-CN.md) —— 直接构建和遍历共享的 `ContextGraph`；时间区间推理；节点插入前的实体去重
- [决策智能](decision-intelligence.zh-CN.md) —— 跨智能体交接记录和追踪决策，包含因果链分析
- [LLM 集成](llm-integrations.zh-CN.md) —— 配置传给每个智能体 `query_with_reasoning()` 的 LLM 提供者
