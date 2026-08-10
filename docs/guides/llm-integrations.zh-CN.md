---
title: "LLM 集成"
description: "通过统一接口将 Semantica 连接到 Groq、OpenAI、Anthropic、HuggingFace、Novita AI 以及 100+ LLM 提供商。"
---

**[English](llm-integrations.md)** · **简体中文（当前）**

Semantica 暴露了一个统一的提供商接口——一个 `.generate()` 方法——覆盖 Groq、OpenAI、Anthropic Claude、HuggingFace、Novita AI，以及通过 LiteLLM 接入的 100+ 提供商。当你需要出于延迟、准确性、成本或数据驻留原因而切换提供商，且不想改动应用代码时，就使用它。

<a id="what-are-llm-integrations"></a>
## LLM 集成是什么？

`semantica.llms` 模块提供了一个统一接口，用于连接大语言模型提供商。你无需为每个提供商学习不同的 API，无论调用的是 Groq、OpenAI、Anthropic，还是本地 HuggingFace 模型，使用的都是相同的方法（`.generate()`、`.generate_structured()`）。

**跨提供商的统一接口：** Semantica 中所有 LLM 提供商都暴露相同的方法，因此从 OpenAI 切换到 Anthropic 只需更改提供商构造函数，而无需更改应用代码。

**提供商包装器 vs. 语义抽取提供商字符串：** `semantica.llms` 类（`Groq`、`OpenAI`、`LiteLLM`、`HuggingFaceLLM`）是用于文本生成的 Python 对象。`semantica.semantic_extract` 模块接受以字符串形式给出的提供商名称，用于实体和关系抽取。本指南会同时介绍两种方式。

<a id="why-use-llm-integrations"></a>
## 为什么使用 LLM 集成？

**提供商可移植性。** 用一个提供商测试，用另一个部署。从 Groq 原型切换到 Anthropic 生产环境，无需改代码。

**降低供应商锁定。** 避免把应用绑定到单一 LLM 提供商的 API。当定价变化或服务可用性出现问题时，切换提供商很直接。

**一致的 API。** 在所有提供商上使用相同的 `.generate()` 和 `.generate_structured()` 方法，而无需学习各提供商专有的接口。

**多提供商工作流。** 在同一条流水线中用快速模型做初始分类，用昂贵的前沿模型做复杂推理。

**本地 vs. 云端部署灵活性。** 在开发期间使用云提供商，在气隔离的生产环境中切换到本地 HuggingFace 模型。

<a id="when-to-use-when-not-to-use"></a>
## 何时使用 / 何时不使用

**适合使用 LLM 集成的场景：**
- 文本生成、摘要和问答任务
- 需要自然语言理解的复杂推理
- 从非结构化文本中抽取结构化数据
- 需要解读和综合的多步分析
- 上下文、歧义或领域知识起关键作用的任务

**确定性工具可能更好的场景：**
- 正则表达式即可处理的模式匹配
- 标准清晰的简单基于规则的分类
- 数学计算或统计分析
- 图遍历和关系查询
- 逻辑已知的确定数据转换

**完整 LLM 可能不必要的场景：**
- 简单关键词搜索或精确字符串匹配
- 具有预定义决策树的确定性工作流
- 推理开销敏感的高频、低延迟操作
- 可解释性要求透明、基于规则逻辑的任务

<Info>
  `semantica.llms` 中的提供商（`Groq`、`OpenAI`、`LiteLLM`、`HuggingFaceLLM`）用于文本生成和 `query_with_reasoning()`。对于结构化实体和关系抽取，`semantica.semantic_extract` 接受以字符串形式给出的提供商名称。两种模式都会在此介绍。
</Info>

<a id="choosing-a-provider"></a>
## 选择提供商

四个因素驱动提供商选择，每个都针对不同用例做了优化：

**延迟** 在实时 SOC 分诊回路中最重要——分析师正等待分诊结论。Groq 的推理基础设施通常能在 300ms 内返回 8B 模型的响应，使其成为交互式工作流的理想之选。

**准确性** 在高风险决策中最重要：临床禁忌核查、信贷委员会推理、法律文档分析。通过 `LiteLLM` 可用的 Claude 或 GPT-4 等前沿模型提供最强的推理能力。

**数据驻留** 约束会为涉密或受 HIPAA 监管的工作负载排除云提供商。带本地模型路径的 `HuggingFaceLLM` 支持完全气隔离、无网络调用的部署。

**规模化成本** 倾向于像 Novita AI 这样的高吞吐量提供商，适用于每小时处理数千份文档、按 token 累计成本的大批量抽取流水线。

统一接口意味着你可以用 Groq 做速度原型、用 Claude 验证准确性、并部署到 Azure OpenAI 以满足合规——而无需更改应用代码。

<a id="the-shared-interface"></a>
## 共享接口

每个提供商都暴露相同的两个方法：

```python
provider.generate(prompt: str, **kwargs) -> str
provider.generate_structured(prompt: str, **kwargs) -> dict
provider.is_available() -> bool
```

`generate()` 返回纯字符串。`generate_structured()` 指示模型以 JSON 响应并返回解析后的 `dict`。`is_available()` 让你在提交调用之前对提供商做健康检查——在重试逻辑和预热检查中很有用。

这意味着 Semantica 中所有接受 LLM 的位置——`query_with_reasoning()`、语义抽取、自定义推理循环——都可以互换地接受这些提供商中的任意一个。

<a id="groq-fast-inference-for-real-time-agents"></a>
## Groq——面向实时智能体的快速推理

**Groq** 是一家云提供商，专注于使用名为语言处理单元（LPU）的定制硬件进行超快语言模型推理。其基础设施为较小模型提供低于 300ms 的响应时间，使其非常适合速度比最大推理能力更重要的实时应用。

Groq Cloud 在专用构建的语言处理单元上运行开放模型，为 8B 参数模型提供低于 300ms 的延迟。这使得 Groq 成为任何 LLM 处于关键路径上的智能体回路的正确默认选择——SOC 分诊、实时告警分类、对话式智能体。

```python
from semantica.llms import Groq

# api_key falls back to the GROQ_API_KEY environment variable
groq = Groq(model="llama-3.1-8b-instant", api_key="YOUR_GROQ_KEY")

# Always health-check before the first call in a long-running process
if not groq.is_available():
    raise RuntimeError("Groq provider unreachable — check GROQ_API_KEY")

# Plain generation
verdict = groq.generate(
    "Alert: 4200 LDAP objects enumerated in 8s from ws-finance-03. "
    "True positive or false positive? One sentence.",
    temperature=0.1,    # low temperature for deterministic triage verdicts
)
print(verdict)
# "True positive — volume and speed are consistent with T1087.002 domain enumeration."

# Structured extraction — returns a parsed dict
entities = groq.generate_structured(
    "Extract threat actors and CVEs from: "
    "APT29 exploited CVE-2024-3400 in PAN-OS GlobalProtect."
)
# {"threat_actors": ["APT29"], "cves": ["CVE-2024-3400"], "products": ["PAN-OS GlobalProtect"]}
```

Groq 的模型选择归结为速度与能力的权衡：关键路径上的任何任务用 `llama-3.1-8b-instant`，需要更强推理但能接受略高延迟时用 `llama-3.3-70b-versatile`，长上下文摘要任务用 `mixtral-8x7b-32768`。

<a id="openai-function-calling-and-vision"></a>
## OpenAI——函数调用与视觉

**OpenAI** 提供对 GPT 模型家族的访问，包括具备函数调用（结构化工具使用）和处理图像与文档的视觉能力等高级能力的 GPT-4o。OpenAI 模型适合需要强语言理解和生成能力的复杂推理任务。

`OpenAI` 提供商包装了 OpenAI API。当你需要 GPT-4o 函数调用的精确性、文档截图的视觉能力，或者你的团队已有 OpenAI 合同并希望继续使用时，就用它。

```python
from semantica.llms import OpenAI

oai = OpenAI(model="gpt-4o", api_key="YOUR_OAI_KEY")
# api_key falls back to OPENAI_API_KEY environment variable

response = oai.generate(
    "Under CRR2 Article 92, what is the minimum total capital ratio "
    "for a G-SIB subject to a 2% GSIB buffer surcharge?"
)
print(response)

# Structured output — useful for deterministic data extraction
risk_data = oai.generate_structured(
    "Extract all counterparty names and exposure amounts from: "
    "Counterparty A: 45M EUR notional, Counterparty B: 12M EUR notional, "
    "Counterparty C: 89M EUR notional."
)
# {"counterparties": [{"name": "Counterparty A", "exposure_eur": 45000000}, ...]}
```

默认模型 `gpt-3.5-turbo` 适用于分类和轻度抽取。对于复杂的多步监管推理或文档理解，请切换到 `gpt-4o`。

<a id="litellm-one-interface-100-providers"></a>
## LiteLLM——一个接口，100+ 提供商

**LiteLLM** 是一个通用适配器，为包括 Anthropic Claude、Azure OpenAI、AWS Bedrock、Google Vertex AI 和本地 Ollama 实例在内的 100 多个不同 LLM 提供商提供单一接口。它充当翻译层，把你的统一 API 调用转换为各提供商专有的请求，使你无需改代码即可在提供商之间切换。

`LiteLLM` 是瑞士军刀。它包装了 `litellm` 库，后者使用统一的补全 API 与所有主流提供商通信。模型字符串同时编码了提供商和模型名称：`"anthropic/claude-sonnet-4-20250514"`、`"azure/gpt-4o"`、`"bedrock/anthropic.claude-3-5-sonnet-20241022-v2:0"`、`"ollama/llama3.2"`。改字符串即换提供商——无需其他代码改动。

```python
from semantica.llms import LiteLLM

# Anthropic Claude — highest accuracy for complex reasoning
llm = LiteLLM(model="anthropic/claude-sonnet-4-20250514")
# Reads ANTHROPIC_API_KEY from environment

# Azure OpenAI — compliance and data-residency requirements
llm = LiteLLM(model="azure/gpt-4o", api_key="YOUR_AZURE_KEY")

# AWS Bedrock — existing cloud agreement, no new vendor
llm = LiteLLM(model="bedrock/anthropic.claude-3-5-sonnet-20241022-v2:0")

# Google Vertex AI
llm = LiteLLM(model="vertex_ai/gemini-1.5-pro")

# Ollama — fully local, no network calls
llm = LiteLLM(model="ollama/llama3.2")

# All of them: same call
response = llm.generate("Summarise the ICH E6(R2) GCP guideline key requirements.")
```

每个提供商的环境变量约定：`ANTHROPIC_API_KEY`、`AZURE_API_KEY`、`OPENAI_API_KEY`、`GOOGLE_APPLICATION_CREDENTIALS` 等。LiteLLM 会自动识别它们。对于由部署环境驱动的提供商切换，你可以把提供商选择放在配置字典中并在启动时注入——应用代码中不需要任何 `if/else` 分支：

```python
import os

PROVIDER_MAP = {
    "prod":    "anthropic/claude-sonnet-4-20250514",
    "staging": "openai/gpt-4o-mini",
    "local":   "ollama/llama3.2",
    "azure":   "azure/gpt-4o",
}

env = os.getenv("DEPLOY_ENV", "local")
llm = LiteLLM(model=PROVIDER_MAP[env])
# The rest of the application never touches provider names
```

<a id="huggingfacellm-air-gapped-and-on-premise"></a>
## HuggingFaceLLM——气隔离与本地部署

**HuggingFaceLLM** 提供对 HuggingFace 生态中开源模型的访问，既可以从 HuggingFace Hub 下载，也可以从本地文件路径加载。这是在推理期间完全没有任何网络访问的离线部署的唯一选择，例如涉密环境或气隔离系统。

`HuggingFaceLLM` 从 HuggingFace Hub 或本地目录路径加载模型。推理期间没有网络调用。这是涉密环境、受 HIPAA 约束的临床部署以及任何没有出站互联网访问的网络区段的唯一选择。

```python
from semantica.llms import HuggingFaceLLM

# HuggingFace Hub — authentication via HF_TOKEN environment variable
hf = HuggingFaceLLM(model="mistralai/Mistral-7B-Instruct-v0.3")

# Biomedical fine-tuned model for clinical entity extraction
bio_llm = HuggingFaceLLM(model="aaditya/Llama3-OpenBioLLM-70B")

# Air-gapped deployment — model on local NFS share or mounted volume
# Both `model` and `model_name` are accepted as constructor parameters
air_gapped_llm = HuggingFaceLLM(model="/opt/models/llama-3.1-70b-instruct")

response = air_gapped_llm.generate(
    "Summarise SIGINT collection window 2024-Q3 for APT29 C2 infrastructure.",
    max_length=512,
)
```

在你的环境中设置 `HF_TOKEN` 以便 Hub 访问受限或私有模型。对于本地路径，不需要 token——模型目录必须包含标准的 HuggingFace checkpoint 文件。

<a id="swapping-providers-without-changing-application-code"></a>
## 在不更改应用代码的情况下切换提供商

统一接口真正的回报体现在 `query_with_reasoning()` 上。这是在每个 `AgentContext` 中驱动基于图谱推理的调用。因为它接受任何提供商对象，你可以在流水线的任何层级插入一个不同的 LLM，而周围代码零改动。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import Groq, LiteLLM

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph(advanced_analytics=True)
context = AgentContext(vector_store=vs, knowledge_graph=graph, graph_expansion=True)

# Load your knowledge base once
context.store(
    [
        {"content": "APT29 exploits CVE-2024-3400 in PAN-OS GlobalProtect — CVSS 10.0",
         "metadata": {"source": "nvd", "actor": "APT29"}},
        {"content": "NOBELIUM (APT29) leverages OAuth token theft against Azure AD tenants",
         "metadata": {"source": "msft_blog_2023", "actor": "APT29", "technique": "T1528"}},
    ],
    extract_entities=True,
    extract_relationships=True,
)

query = "What is APT29's current exploitation methodology and what cloud services are targeted?"

# Tier 1: fast answer with Groq (< 300ms)
fast_llm = Groq(model="llama-3.1-8b-instant", api_key="YOUR_GROQ_KEY")
fast_result = context.query_with_reasoning(query, llm_provider=fast_llm, max_results=5)
print("FAST: {}  (conf={:.0%})".format(fast_result["response"], fast_result["confidence"]))

# Tier 2: deep answer with Claude if confidence is below threshold
if fast_result["confidence"] < 0.85:
    deep_llm = LiteLLM(model="anthropic/claude-sonnet-4-20250514")
    deep_result = context.query_with_reasoning(
        query, llm_provider=deep_llm, max_results=15, max_hops=3
    )
    print("DEEP: {}  (conf={:.0%})".format(deep_result["response"], deep_result["confidence"]))
```

`context`、图谱、检索逻辑——这些都没有变。两个层级之间只有 `llm_provider` 参数不同。

<a id="using-providers-for-semantic-extraction"></a>
## 将提供商用于语义抽取

对于 NER、关系抽取和三元组抽取，Semantica 的 `semantica.semantic_extract` 模块接受以字符串形式给出的提供商名称，而不是类实例。该模块在内部处理提供商实例化。

```python
from semantica.semantic_extract import NamedEntityRecognizer, EventDetector
from semantica.semantic_extract.methods import (
    extract_entities_llm,
    extract_relations_llm,
    extract_triplets_llm,
)

# NER with Groq — provider name as a string
ner = NamedEntityRecognizer(
    methods=["llm"],
    provider="groq",
    llm_model="llama-3.1-8b-instant",
)
entities = ner.extract_entities(
    "APT29 exploited CVE-2024-3400 in PAN-OS GlobalProtect to compromise NATO member networks."
)
for e in entities:
    # Entity fields: .text, .label, .confidence
    print("{} ({}) — conf={:.2f}".format(e.text, e.label, e.confidence))
# APT29 (THREAT_ACTOR) — conf=0.96
# CVE-2024-3400 (CVE) — conf=0.99
# PAN-OS GlobalProtect (PRODUCT) — conf=0.94
# NATO (ORGANIZATION) — conf=0.91

# Event detection with the same provider pattern
detector = EventDetector(method="llm", provider="groq")
events = detector.detect_events(
    "ENISA published the Threat Landscape 2024 report on October 22nd, "
    "covering 11 primary threat categories including ransomware and supply-chain attacks."
)

# Triplet extraction — returns (subject, predicate, object) triples
text = "Warfarin inhibits VKORC1 enzyme activity, reducing vitamin K-dependent clotting factor synthesis."
triplets = extract_triplets_llm(text, provider="groq", model="llama-3.1-8b-instant")
for t in triplets:
    print("{} -> {} -> {}".format(t.subject, t.predicate, t.object))
# warfarin -> inhibits -> VKORC1 enzyme activity
# warfarin -> reduces -> vitamin K-dependent clotting factor synthesis
```

<a id="novita-ai-cost-efficient-bulk-extraction"></a>
## Novita AI——高性价比批量抽取

Novita AI 暴露了一个与 OpenAI 兼容的 API，并作为抽取层的内置提供商可用。它的访问方式与 `semantica.llms` 类不同——通过 `semantica.semantic_extract.providers` 中的 `create_provider`——这使它成为按调用成本敏感的大批量 NER 流水线的正确选择。

```python
from semantica.semantic_extract.providers import create_provider
from semantica.semantic_extract import NamedEntityRecognizer

# create_provider pools instances — same key reuses the same object
provider = create_provider(
    "novita",
    api_key="YOUR_NOVITA_KEY",       # or set NOVITA_API_KEY env var
    model="deepseek/deepseek-v3.2",  # default model
)

if provider.is_available():
    # Plain generation
    response = provider.generate("Summarise the Basel III leverage ratio requirement.")

    # Structured extraction — returns parsed dict
    data = provider.generate_structured(
        "Extract drug names and dosages from: "
        "Patient received warfarin 5mg daily, aspirin 75mg daily, metformin 500mg twice daily."
    )

# Use Novita through the NER interface — provider name as string
ner = NamedEntityRecognizer(
    methods=["llm"],
    provider="novita",
    llm_model="deepseek/deepseek-v3.2",
)
entities = ner.extract_entities(
    "CVE-2024-3400 is exploited by UNC3886 targeting PAN-OS GlobalProtect."
)
for e in entities:
    print("{} ({}) — conf={:.2f}".format(e.text, e.label, e.confidence))
```

Novita 在底层需要 `openai` Python 客户端——使用 `pip install "semantica[llm-openai]"` 或 `pip install openai` 安装。

<a id="domain-examples"></a>
## 领域示例

<Tabs>
<Tab title="国防——CTI/威胁">
一个涉密的威胁情报分析单元需要完全气隔离运行——不能有任何出站网络流量。抽取模型和推理模型都从本地 NFS 共享加载。知识图谱在分析会话期间累积；所有推理都在本地进行。

```python
from semantica.llms import HuggingFaceLLM
from semantica.semantic_extract import NamedEntityRecognizer
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

# Both models load from the air-gapped NFS share — no Hub calls
extraction_llm = HuggingFaceLLM(model="/opt/models/mistral-7b-instruct")
reasoning_llm  = HuggingFaceLLM(model="/opt/models/llama-3.1-70b-instruct")

# The llms module wrappers can also be used directly for raw prompt generation
# when you want to bypass the semantic extraction layer entirely

sigint_text = (
    "[S//NF] APT29 operator observed deploying WARPWIRE credential harvester "
    "via CVE-2024-3400 on perimeter VPN gateways of target BRAVO-7."
)

# For fully local models, call the provider's generate method directly
raw_entities = extraction_llm.generate(
    "Extract all threat actors, CVEs, malware names, and target identifiers "
    "from the following text as a JSON list:\n\n" + sigint_text
)

# Build the graph and run reasoning
vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store=vs,
    knowledge_graph=graph,
    graph_expansion=True,
    decision_tracking=True,
)

context.store(
    sigint_text,
    metadata={"source": "SIGINT_Q4_2024", "classification": "SECRET//NOFORN"},
    conversation_id="op-analysis-q4",
)

result = context.query_with_reasoning(
    "What credential-harvesting capabilities has APT29 deployed against "
    "perimeter VPN gateways and what CVEs enable initial access?",
    llm_provider=reasoning_llm,   # fully local — no network calls
    max_results=10,
    max_hops=3,
)
print(result["response"])
print("Confidence: {:.0%}".format(result["confidence"]))

context.save("./classified_output/q4_analysis/")
```

</Tab>

<Tab title="安全——SOC/事件">
一条 SOC 流水线在不同层级使用两个提供商：Groq 用于让分析师保持心流的 500ms 以内初始分诊，Anthropic Claude 用于 Tier 1 置信度低于升级阈值时的深入 ATT&CK 分析。提供商切换由程序决定——无需人工交接。

```python
from semantica.llms import Groq, LiteLLM
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph()
context = AgentContext(
    vector_store=vs,
    knowledge_graph=graph,
    graph_expansion=True,
    decision_tracking=True,
)

# Preload MITRE ATT&CK runbook knowledge
context.store([
    "T1087.002 (Domain Account Discovery): anomalous LDAP enumeration — isolate source host, reset service account passwords",
    "T1053.005 (Scheduled Task/Job): encoded PowerShell via wmiprvse.exe — collect task XML, check persistence keys, notify IR",
    "T1021.002 (SMB/Windows Admin Shares): PsExec lateral movement to DC — immediate host isolation, reset service accounts",
])

alert = (
    "SIEM Alert: host ws-finance-03, user jsmith — scheduled task with base64-encoded PowerShell. "
    "Parent process: wmiprvse.exe. Sigma: T1053.005. Time: 2025-06-21T09:14:32Z."
)
context.store(alert, metadata={"type": "alert", "severity": "high"})

# Tier 1: fast triage with Groq — target < 500ms end-to-end
fast_llm = Groq(model="llama-3.1-8b-instant", api_key="YOUR_GROQ_KEY")
triage = context.query_with_reasoning(
    "Is this alert a true positive? One sentence verdict and confidence.",
    llm_provider=fast_llm,
    max_results=5,
)
print("TRIAGE: {} (conf={:.0%})".format(triage["response"], triage["confidence"]))

# Tier 2: escalate to Claude for deep analysis if Tier 1 is uncertain
if triage["confidence"] < 0.88:
    deep_llm = LiteLLM(model="anthropic/claude-sonnet-4-20250514")
    deep = context.query_with_reasoning(
        "Full MITRE ATT&CK analysis of this alert: identify the attack chain, "
        "blast radius, affected systems, and recommended containment steps.",
        llm_provider=deep_llm,
        max_results=15,
        max_hops=3,
    )
    print("DEEP ANALYSIS: {}".format(deep["response"]))

    context.record_decision(
        category="escalation",
        scenario="Scheduled task T1053.005 on ws-finance-03 — Tier 1 conf {:.0%}".format(triage["confidence"]),
        reasoning=deep["reasoning_path"],
        outcome="escalated_tier2",
        confidence=deep["confidence"],
        entities=["ws-finance-03", "jsmith", "T1053.005"],
        decision_maker="soc_pipeline_v3",
    )
```

</Tab>

<Tab title="生命科学——临床/制药">
一条临床 NLP 流水线使用 HuggingFace 上一个领域专用的生物医学 NER 模型做实体抽取，然后切换到 Anthropic Claude 做结构化肿瘤报告综合。同一个 `LiteLLM` 包装器处理 Claude 调用——为满足 HIPAA 合规切换到 Azure OpenAI 只需改一行。

```python
from semantica.llms import LiteLLM
from semantica.semantic_extract import NamedEntityRecognizer

# Biomedical NER — HuggingFace extractor for clinical entities
# (Note: HuggingFace NER uses method="huggingface", not the LLM generation interface)
ner = NamedEntityRecognizer(
    methods=["huggingface"],
    huggingface_model="d4data/biomedical-ner-all",
    confidence_threshold=0.75,
)

clinical_note = (
    "Patient presents with HER2+ breast cancer (T2N1M0, Stage IIB). "
    "Recommended: trastuzumab 8mg/kg loading then 6mg/kg q3w + pertuzumab 840mg loading "
    "then 420mg q3w + docetaxel 75mg/m2 q3w (THP regimen) for 6 cycles. "
    "eGFR 78 mL/min/1.73m2, LVEF 62%. Monitor for cardiotoxicity."
)

entities = ner.extract_entities(clinical_note)
# Entity fields: .text, .label (not .type), .confidence
drugs     = [e for e in entities if e.label == "DRUG"]
diagnoses = [e for e in entities if e.label in ("DISEASE", "CANCER")]

print("Drugs identified:")
for d in drugs:
    print("  {} (conf={:.2f})".format(d.text, d.confidence))
# trastuzumab (conf=0.98), pertuzumab (conf=0.97), docetaxel (conf=0.96)

# Report synthesis with Claude — switch to azure/gpt-4o for HIPAA by changing one string
report_llm = LiteLLM(model="anthropic/claude-sonnet-4-20250514")
# For HIPAA-constrained Azure deployment:
# report_llm = LiteLLM(model="azure/gpt-4o", api_key="YOUR_AZURE_KEY")

oncology_summary = report_llm.generate(
    "Write a structured oncology treatment summary for the following note. "
    "Include: diagnosis, staging, treatment regimen, monitoring parameters, "
    "and key safety considerations.\n\n" + clinical_note
)
print(oncology_summary)
```

</Tab>

<Tab title="银行——风险/合规">
一个监管问答系统把同一个 Basel III 问题跑过两个提供商，并挑选置信度更高的答案——这是高风险监管解读中一种简单的共识模式，因为错误答案带有法律风险。

```python
from semantica.llms import OpenAI, LiteLLM
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store=vs,
    knowledge_graph=graph,
    graph_expansion=True,
    retention_days=2555,
)

# Load Basel III / CRR2 regulatory corpus
context.store(
    [
        {"content": "CRR2 Art. 92: minimum total capital ratio 8% + 2.5% conservation buffer + GSIB surcharge",
         "metadata": {"source": "CRR2_Art92", "category": "capital_requirement"}},
        {"content": "Basel III leverage ratio: Tier 1 capital / total exposure >= 3% (Art. 429 CRR2)",
         "metadata": {"source": "CRR2_Art429", "category": "leverage"}},
        {"content": "GSIB buffer surcharges: bucket 1=1%, bucket 2=1.5%, bucket 3=2%, bucket 4=2.5%, bucket 5=3.5%",
         "metadata": {"source": "BCBS_GSIB_2022", "category": "gsib_buffer"}},
    ],
    extract_entities=True,
    extract_relationships=True,
)

question = (
    "Under CRR2 Article 92 and the BCBS GSIB framework, what is the minimum "
    "total capital ratio for a bucket-2 G-SIB? Show the component breakdown."
)

# Two-provider consensus — same query, same graph, different LLMs
gpt4o  = OpenAI(model="gpt-4o", api_key="YOUR_OAI_KEY")
claude = LiteLLM(model="anthropic/claude-sonnet-4-20250514")

answer_a = context.query_with_reasoning(question, llm_provider=gpt4o,  max_results=10)
answer_b = context.query_with_reasoning(question, llm_provider=claude, max_results=10)

# Pick higher-confidence answer for the audit trail
best = answer_a if answer_a["confidence"] >= answer_b["confidence"] else answer_b
winner = "GPT-4o" if best is answer_a else "Claude"

print("Selected: {} (conf={:.0%})".format(winner, best["confidence"]))
print(best["response"])
# Expected: 8% (minimum) + 2.5% (conservation) + 1.5% (bucket-2 GSIB) = 12.0% total

# Sources the answer is grounded in
for src in best["sources"]:
    print("  - [{}] {}".format(src.get("source", "?"), src["content"][:60]))
```

</Tab>
</Tabs>

<a id="common-pitfalls"></a>
## 常见陷阱

**为简单的抽取任务选择昂贵的前沿模型。** 用 GPT-4o 或 Claude Sonnet 做基础实体抽取是大材小用——Groq 的 Llama 模型能以更低的成本和延迟处理直接的 NER 和分类。把前沿模型留给需要细致解读的复杂推理。

**忽视提供商之间的延迟差异。** Groq 通常在 300ms 内响应，而 Anthropic Claude 对同一查询可能需要 2-3 秒。对于实时智能体或交互式工作流，延迟差异会在多次 LLM 调用中累积。请在真实负载下剖析你的提供商性能。

**用 LLM 做正则表达式即可处理的确定性模式匹配。** 如果你的任务是提取电子邮件地址、电话号码或其他基于模式的实体，正则表达式比 LLM 抽取更快、更便宜、更可靠。当上下文、歧义或领域知识对正确解读很重要时才使用 LLM。

**不校验结构化输出。** `generate_structured()` 方法返回解析后的 JSON，但 LLM 仍可能生成格式错误或不完整的结构。在下游使用数据之前，请始终根据你预期的模式校验返回的字典。

**切换提供商而不测试提示行为。** 不同模型对同一提示的响应不同。为 GPT-4 优化的提示在 Llama 或 Claude 上可能产生糟糕的结果。切换提供商时，请测试你的提示，并按需调整温度、指令或示例。

**过度使用本地 HuggingFace 模型处理需要最新知识的任务。** 本地模型的知识截止于其训练日期，无法访问当前信息。对于需要最新知识（最近的 CVE、现行法规、最新威胁情报）的任务，可能需要训练数据更及时的云提供商。

<a id="related-guides"></a>
## 相关指南

- [智能体记忆](agent-memory.zh-CN.md)——使用任意 LLM 提供商的 `query_with_reasoning()` 进行基于图谱的检索
- [多智能体系统](multi-agent.zh-CN.md)——在共享图谱流水线中把不同 LLM 提供商接到不同智能体层级
- [语义抽取](semantic-extraction.zh-CN.md)——由 LLM 驱动的 NER、关系抽取、事件检测和三元组抽取
- [GraphRAG](graphrag.zh-CN.md)——使用 `query_with_reasoning()` 进行多跳图推理
