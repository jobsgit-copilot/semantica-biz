---
title: "语义抽取"
description: "Semantica 如何使用模式、ML 和 LLM 方法从非结构化文本中抽取实体、关系、事件和 RDF 三元组——包含回退链、批量处理以及面向国防情报、网络安全、生命科学和金融合规的真实流水线。"
icon: "magnifying-glass"
---

**[English](semantic-extraction.md)** · **简体中文（当前）**

<a id="what-is-semantic-extraction"></a>
## 什么是语义抽取？

语义抽取是从非结构化文本中自动识别有意义的信息并将其转换为结构化、机器可读格式的过程。与简单的关键词搜索或模式匹配不同，语义抽取理解自然语言中的上下文、关系和概念之间的隐含联系。

**与基本文本处理的关键区别：**
- **正则匹配**能找到精确模式但遗漏上下文含义
- **关键词搜索**能定位术语但忽略它们之间的关系
- **人工标注**能捕获语义含义但无法扩展
- **语义抽取**自动识别实体、关系和事件，同时保留上下文理解

当你从情报文本中抽取"APT29"和"NATO"等实体时，语义抽取还会捕获 APT29"攻击"NATO 网络这一关系，创建直接馈入图数据库、推理系统和检索工作流的结构化知识。

<a id="why-use-semantic-extraction"></a>
## 为何使用语义抽取？

**知识图谱填充。**将非结构化文档转换为互联的知识图谱，其中实体成为节点，关系成为边，实现复杂的图遍历和推理。

**GraphRAG 准备。**从原始文本中抽取结构化事实，使基于图谱的检索能够找到精确、上下文相关的信息，而不仅仅是相似的文档分块。

**将非结构化文本转化为结构化数据。**将情报报告、临床笔记、法律文件和监管文件转化为数据库、RDF 三元组和 JSON 模式，供下游系统查询和处理。

**下游检索和推理收益。**实现精确的基于实体的搜索、关系发现、因果分析和多跳推理——这些仅靠文档级检索是不可能实现的。

**自动化知识发现。**在大规模文档集合中发现人工分析师因数据量和复杂性而遗漏的隐藏联系和模式。

<a id="when-to-use-when-not-to-use"></a>
## 适用与不适用场景

**使用语义抽取的场景：**
- 将情报报告、临床笔记和监管文件转化为结构化知识
- 从非结构化文本语料库构建知识图谱
- 为基于图谱的推理和 GraphRAG 工作流准备文本
- 跨文档集合发现关系和联系
- 为下游分析和报告创建结构化数据集

**确定性解析可能更适合的场景：**
- 高度结构化的标识符如电子邮件地址、UUID、哈希值和日志 ID，正则模式即可满足
- 从标准化格式（CSV、JSON、XML）中简单抽取数据
- 固定格式的已知模式，不需要上下文理解
- 抽取速度至关重要且不需要语义理解的高频操作

**以下情况可考虑更简单的替代方案：**
- 文档已经是结构化的，不需要自然语言理解
- 简单的关键词搜索或文档检索即可满足需求
- 文本质量太差，无法进行可靠的语义分析（严重损坏的 OCR、碎片化数据）

<a id="typical-workflow"></a>
## 典型工作流

语义抽取工作流遵循一个结构化序列，将原始文本转化为图谱就绪的知识：

**摄取** → 从各种来源（文件、数据库、API）加载文档并为处理准备文本

**抽取** → 应用命名实体识别（NER）、关系抽取、事件检测和共指消解来识别有意义的信息

**解析** → 将实体提及（"APT29"、"该组织"、"他们"）合并为规范引用，并消歧重叠实体

**关联** → 通过关系连接抽取的实体，在概念之间创建结构化连接的网络

**序列化** → 将抽取的知识转换为 RDF 三元组、JSON-LD 或其他结构化格式

**存储** → 将结构化输出加载到知识图谱、向量数据库或智能体记忆系统中

**检索** → 通过图遍历、语义搜索和推理工作流查询结构化知识

此流水线将"APT29 部署 HAMMERTOSS 恶意软件攻击北约网络"等文档转换为 `(APT29, deployed, HAMMERTOSS)` 和 `(HAMMERTOSS, targets, NATO_networks)` 等结构化三元组，从而实现复杂的下游分析。

`semantica.semantic_extract` 将非结构化文本转化为结构化的图谱就绪输出：它识别命名实体、抽取实体间的关系、检测以时间为锚点的事件、解析共指，并将所有内容序列化为 RDF 三元组。使用它从原始文档——情报报告、临床笔记、监管文件或任何自由文本语料库——填充 `ContextGraph`。

<Info>
  抽取的实体和关系通过 `AgentContext.store()` 馈入 `ContextGraph`。有关它们如何归因回源文档，请参阅[溯源指南](provenance.zh-CN.md)。有关填充后的图谱如何被查询和遍历，请参阅[上下文图谱](context-graphs.zh-CN.md)。
</Info>

<a id="step-1-named-entity-recognition-who-and-what-is-in-the-text"></a>
## 步骤 1 —— 命名实体识别：文本中有谁和什么

**命名实体识别（NER）**识别并分类文本中有意义的名词和名词短语，如人物、组织、地点、产品以及领域特定实体如威胁行为体或药物名称。NER 通过识别文档中的关键参与者和对象来构成语义抽取的基础。

`NamedEntityRecognizer` 从文档中抽取有意义的名词，让你根据延迟预算和领域需求选择底层方法：

```python
from semantica.semantic_extract import NamedEntityRecognizer

# 一份完成的情报报告摘录
report = """
ASSESSMENT (SECRET//NOFORN): GAMMA-7 threat actor, assessed with HIGH confidence
to operate from COUNTRY_X, conducted OPERATION NIGHTFALL between Jan–Mar 2025.
The group deployed HAMMERTOSS malware targeting NATO logistics networks via
spear-phishing lures referencing Exercise STEADFAST DEFENDER 2025.
Infrastructure cluster 185.220.101.0/24 was active throughout the campaign.
A second actor, DELTA-3, provided GAMMA-7 with zero-day exploits (CVE-2025-1234,
CVE-2025-5678) for Ivanti Connect Secure VPN appliances.
"""

# LLM 支持的 NER 能处理 spaCy 无法识别的自定义情报实体类型。
# 回退链：如果 llm 返回空结果，尝试 ml（spaCy），然后是模式匹配。
ner = NamedEntityRecognizer(
    methods=["llm", "ml", "pattern"],
    confidence_threshold=0.75,
    provider="anthropic",
    llm_model="claude-sonnet-4-6",
)
entities = ner.extract_entities(report)

for e in entities:
    print("[{:>5.2f}]  {:15s}  {}".format(e.confidence, e.label, e.text))

# Expected output (abbreviated):
# [ 0.94]  THREAT_ACTOR    GAMMA-7
# [ 0.91]  THREAT_ACTOR    DELTA-3
# [ 0.97]  MALWARE         HAMMERTOSS
# [ 0.99]  CVE             CVE-2025-1234
# [ 0.99]  CVE             CVE-2025-5678
# [ 0.88]  NETWORK         185.220.101.0/24
# [ 0.85]  ORG             NATO
```

`confidence` 字段告诉你抽取器的确定程度。低于你阈值（此处为 0.75）的值在到达你之前已被过滤。`label` 字段使用标准 CoNLL 类型（PERSON、ORG、GPE）或 LLM 从上下文推断的领域特定类型（THREAT_ACTOR、MALWARE、CVE、NETWORK）。

一旦获得扁平的实体列表，按类型分组以便后续步骤更易处理：

```python
# classify_entities 将扁平列表按标签分组为字典
grouped = ner.classify_entities(entities)

print("Threat actors:", [e.text for e in grouped.get("THREAT_ACTOR", [])])
# Threat actors: ['GAMMA-7', 'DELTA-3']

print("Malware:      ", [e.text for e in grouped.get("MALWARE", [])])
# Malware:       ['HAMMERTOSS']

print("CVEs:         ", [e.text for e in grouped.get("CVE", [])])
# CVEs:          ['CVE-2025-1234', 'CVE-2025-5678']

# score_confidence 使用统计模型重新评分——当你的主要方法
# 不返回校准概率时很有用（例如模式匹配）
rescored = ner.score_confidence(entities)
high_conf = [e for e in rescored if e.confidence >= 0.85]
print("High-confidence entities: {}".format(len(high_conf)))
```

<a id="step-2-relation-extraction-how-the-entities-connect"></a>
## 步骤 2 —— 关系抽取：实体如何连接

**关系抽取**识别实体之间的语义关系，捕获的不仅仅是文本中存在哪些实体，还有它们如何交互、影响或连接彼此。这创建了链接知识图谱中实体节点的边。

`RelationExtractor` 产出实体之间的连接网络——谁部署了什么、谁向谁提供了什么、哪个 CVE 攻击哪个产品：

```python
from semantica.semantic_extract import RelationExtractor

# 依存解析捕获句法关系（"DELTA-3 provided GAMMA-7 with..."）
# 模式匹配作为回退捕获显式动词短语
rel = RelationExtractor(
    method=["dependency", "pattern"],
    relation_types=["operates_from", "deployed", "targets", "provided_to", "exploits"],
    bidirectional=True,      # 也尝试 object→subject 方向
    confidence_threshold=0.65,
)

# extract_relations 接受原始文本加上你已经找到的实体。
# 传入实体避免重新运行 NER 并约束搜索空间。
relations = rel.extract_relations(report, entities)

for r in relations:
    print("[{:.2f}]  {} --[{}]--> {}".format(
        r.confidence, r.subject.text, r.predicate, r.object.text
    ))

# Example output:
# [0.88]  GAMMA-7 --[deployed]--> HAMMERTOSS
# [0.82]  DELTA-3 --[provided_to]--> GAMMA-7
# [0.79]  CVE-2025-1234 --[exploits]--> Ivanti Connect Secure
# [0.85]  GAMMA-7 --[targets]--> NATO
```

每个 `Relation` 上的 `context` 字段存储周围的句子。这让你能够审计抽取器为何做出某个连接——当分析师需要在基于某个关联采取行动之前验证它在源文本中有据可查时，这至关重要。

<a id="step-3-event-detection-what-happened-when-and-to-whom"></a>
## 步骤 3 —— 事件检测：发生了什么、何时以及对谁

**事件检测**识别文本中描述的离散事件或行动，捕获的不仅是静态关系，还有随时间展开的动态过程。事件包含参与者、时间边界、地点和结果。

`EventDetector` 呈现结构化的时间锚点事件——具有参与者、时间窗口和地点的离散事件：

```python
from semantica.semantic_extract import EventDetector

# method="llm" 是默认值，对隐含事件提供最佳召回率。
# method="pattern" 适用于离线、高吞吐流水线。
detector = EventDetector(method="llm", provider="anthropic")

events = detector.detect_events(report)

for ev in events:
    print("[{}]  time={}  location={}".format(
        ev.event_type, ev.time or "unknown", ev.location or "n/a"
    ))
    print("   participants: {}".format(ev.participants))
    print("   text: {}".format(ev.text[:80]))

# Example output:
# [campaign]  time=Jan–Mar 2025  location=n/a
#    participants: ['GAMMA-7', 'NATO']
#    text: GAMMA-7 conducted OPERATION NIGHTFALL between Jan–Mar 2025...
# [supply_chain]  time=unknown  location=n/a
#    participants: ['DELTA-3', 'GAMMA-7']
#    text: DELTA-3 provided GAMMA-7 with zero-day exploits...
```

对于批量处理，将字典列表传给 `extract()`。每个字典携带一个 `content` 键和可选的用于溯源追踪的 `id`：

```python
# extract() 内部处理并发处理
batch_events = detector.extract([
    {"content": report, "id": "FINTEL_2025_0192"},
    {"content": advisory_2, "id": "FINTEL_2025_0193"},
    # ... 最多 200 个文档
])

# batch_events 是 List[List[Event]]——每个文档一个内部列表
for doc_idx, doc_events in enumerate(batch_events):
    print("Doc {}: {} events".format(doc_idx, len(doc_events)))
```

<a id="step-4-coreference-resolution-one-entity-many-names"></a>
## 步骤 4 —— 共指消解：一个实体，多个名称

**共指消解**识别不同文本范围指向同一现实世界实体的情况，将"APT29"、"该组织"、"他们"和"该威胁行为体"等提及合并为统一引用。这防止下游处理将同一实体视为多个独立对象。

`CoreferenceResolver` 将"GAMMA-7"、"该组织"、"他们"和"该威胁行为体"合并为规范链，使下游抽取不会将它们视为独立实体：

```python
from semantica.semantic_extract import CoreferenceResolver

resolver = CoreferenceResolver(method="llm", provider="anthropic")

# 传入已抽取的实体，使解析器能将代词锚定
# 到正确的命名实体，而非从头猜测
chains = resolver.resolve_coreferences(report, entities=entities)

for chain in chains:
    print("Representative: {}".format(chain.representative.text))
    mentions = [m.text for m in chain.mentions if m.text != chain.representative.text]
    print("   Also called: {}".format(mentions))

# Example output:
# Representative: GAMMA-7
#    Also called: ['the group', 'they', 'the threat actor']
```

共指解析完成后，你现在可以在将文本传给关系抽取之前用规范名称替换代词和别名——从而大幅提升关系图谱的质量。

<a id="step-5-triplet-extraction-and-rdf-serialisation-graph-ready-output"></a>
## 步骤 5 —— 三元组抽取与 RDF 序列化：图谱就绪输出

**三元组抽取**将语义知识转换为主-谓-宾三元组，这是知识图谱和 RDF 数据库的基本构建块。这种结构化表示实现了图查询、推理以及与语义网技术的集成。

`TripletExtractor` 将所有内容转换为主-谓-宾三元组并序列化为 RDF，随时可用于图谱摄取和 SPARQL 查询：

```python
from semantica.semantic_extract import TripletExtractor

tri = TripletExtractor(
    method="llm",
    provider="anthropic",
    llm_model="claude-sonnet-4-6",
    include_temporal=True,       # 在可用时将时间上下文附加到三元组
    include_provenance=True,     # 在每个三元组中嵌入源文档引用
)

# 传入你已经抽取的实体和关系——抽取器
# 使用它们来约束和验证其产出
triplets = tri.extract_triplets(report, entities, relations)

# 在序列化之前过滤格式错误的三元组
valid = tri.validate_triplets(triplets)
print("Valid: {}/{}".format(len(valid), len(triplets)))

for t in valid:
    print("({}, {}, {})  conf={:.2f}".format(
        t.subject, t.predicate, t.object, t.confidence
    ))

# 序列化为 Turtle RDF 用于图谱摄取或 STIX 导出
turtle_rdf = tri.serialize_triplets(valid, format="turtle")
print(turtle_rdf[:600])

# 格式："turtle" | "ntriples" | "jsonld" | "xml"
```

一条经验证的三元组如 `(GAMMA-7, deployed, HAMMERTOSS)`，在 `include_temporal=True` 时会携带你在步骤 3 中检测到的时间区间——使图谱不仅可按发生了什么查询，还可按何时发生查询。

<a id="putting-it-together-a-reusable-extraction-pipeline"></a>
## 综合应用：可复用的抽取流水线

将全部五个步骤串联为一个可对每个传入文档调用的单一函数：

```python
from semantica.semantic_extract import (
    NamedEntityRecognizer, RelationExtractor,
    EventDetector, CoreferenceResolver, TripletExtractor,
)
from semantica.provenance import ProvenanceManager
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore


def ingest_intel_report(
    text: str,
    doc_id: str,
    agent: AgentContext,
    prov: ProvenanceManager,
    method: str = "llm",
) -> dict:
    """
    Full extraction pipeline: NER → Coref → Relations → Events → Triplets → Graph.
    Returns a summary dict with counts for monitoring dashboards.
    """

    # 1. 命名实体识别
    ner = NamedEntityRecognizer(
        methods=[method, "pattern"],
        confidence_threshold=0.70,
        provider="anthropic",
        llm_model="claude-sonnet-4-6",
    )
    entities = ner.extract_entities(text)
    classified = ner.classify_entities(entities)

    # 2. 共指消解——在关系抽取之前
    resolver = CoreferenceResolver(method=method, provider="anthropic")
    chains = resolver.resolve_coreferences(text, entities=entities)

    # 3. 关系抽取
    rel = RelationExtractor(
        method=[method, "pattern"],
        relation_types=["deployed", "targets", "exploits", "operates_from", "provided_to"],
        confidence_threshold=0.65,
        provider="anthropic",
        llm_model="claude-sonnet-4-6",
    )
    relations = rel.extract_relations(text, entities)

    # 4. 事件检测
    detector = EventDetector(method=method, provider="anthropic")
    events = detector.detect_events(text)

    # 5. 三元组抽取和 RDF 序列化
    tri = TripletExtractor(
        method=method,
        provider="anthropic",
        llm_model="claude-sonnet-4-6",
        include_temporal=True,
        include_provenance=True,
    )
    triplets = tri.extract_triplets(text, entities, relations)
    valid    = tri.validate_triplets(triplets)
    turtle   = tri.serialize_triplets(valid, format="turtle")

    # 6. 存储到智能体记忆 + 知识图谱
    graph_stats = agent.store(
        [{"content": text, "metadata": {"source": doc_id, "classification": "SECRET//NOFORN"}}],
        extract_entities=True,
        extract_relationships=True,
        link_entities=True,
    )

    # 7. 溯源追踪——每个实体链接回其源文档
    prov.track_entities_batch(
        [{"id": e.text.lower().replace(" ", "_"), "confidence": e.confidence}
         for e in entities],
        source=doc_id,
        activity_id="intel_extraction_pipeline",
    )

    return {
        "entities":       len(entities),
        "entity_types":   {k: len(v) for k, v in classified.items()},
        "coref_chains":   len(chains),
        "relations":      len(relations),
        "events":         len(events),
        "triplets_valid": len(valid),
        "graph_nodes":    graph_stats.get("graph_nodes", 0),
        "graph_edges":    graph_stats.get("graph_edges", 0),
        "rdf_turtle":     turtle,
    }


# 处理全部 200 份报告
intel_graph = ContextGraph(advanced_analytics=True)
intel_agent = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=intel_graph,
    decision_tracking=True,
)
prov = ProvenanceManager(storage_path="intel_provenance.db")

# reports 是从你的 200 个文档加载的 (text, doc_id) 元组列表
for text, doc_id in reports:
    summary = ingest_intel_report(text, doc_id, intel_agent, prov)
    print("{}: {} entities, {} relations, {} events, {}/{} triplets valid".format(
        doc_id,
        summary["entities"],
        summary["relations"],
        summary["events"],
        summary["triplets_valid"],
        len(summary["rdf_turtle"]),
    ))
```

<a id="domain-examples"></a>
## 领域示例

<Tabs>
  <Tab title="国防 — CTI/威胁">
    完成的情报报告包含威胁行为体、恶意软件、CVE、基础设施集群和行动时间线。LLM 支持的 NER 能处理 spaCy 现成模型遗漏的自定义实体标签（THREAT_ACTOR、OPERATION），而 RDF 序列化产生与 STIX 2.1 对象类型兼容的 Turtle 输出。

```python
from semantica.semantic_extract import (
    NamedEntityRecognizer, RelationExtractor, TripletExtractor,
)

ner = NamedEntityRecognizer(
    methods=["llm", "pattern"],
    confidence_threshold=0.75,
    provider="anthropic",
    llm_model="claude-sonnet-4-6",
)
entities = ner.extract_entities(fintel_text)
grouped  = ner.classify_entities(entities)

print("Threat actors:", [e.text for e in grouped.get("THREAT_ACTOR", [])])
# ['APT29', 'COZY BEAR']
print("Malware:      ", [e.text for e in grouped.get("MALWARE", [])])
# ['HAMMERTOSS', 'SUNBURST']
print("CVEs:         ", [e.text for e in grouped.get("CVE", [])])
# ['CVE-2025-1234', 'CVE-2025-5678']

rel = RelationExtractor(
    method=["llm", "dependency"],
    relation_types=["operates_from", "deployed", "targets", "exploits"],
    confidence_threshold=0.70,
    provider="anthropic",
    llm_model="claude-sonnet-4-6",
)
relations = rel.extract_relations(fintel_text, entities)

tri = TripletExtractor(
    method="llm",
    provider="anthropic",
    llm_model="claude-sonnet-4-6",
    include_temporal=True,
    include_provenance=True,
)
triplets = tri.extract_triplets(fintel_text, entities, relations)
valid    = tri.validate_triplets(triplets)
turtle   = tri.serialize_triplets(valid, format="turtle")
# turtle 现在包含 (APT29, deployed, HAMMERTOSS)、
# (APT29, targets, NATO_logistics) 等的 RDF 三元组——随时可用于 STIX 图谱摄取
```

  </Tab>

  <Tab title="安全 — SOC/事件">
    漏洞公告和 SIEM 告警包含 CVE、IP 地址、域名指标、恶意软件家族和受影响的软件版本。使用 `["ml", "pattern"]` 链的离线 NER 提供低于 200ms 的延迟，这对于 LLM 延迟不可接受的实时分流流水线至关重要。

```python
from semantica.semantic_extract import (
    NamedEntityRecognizer, RelationExtractor, TripletExtractor,
)

advisory = """
CVE-2025-44228 (CVSS 10.0): Critical RCE in Apache Log4j2 versions 2.0–2.14.1.
Threat actor WIZARD SPIDER exploiting via Cobalt Strike beacons since 2025-03-14.
Indicators: 203.0.113.45, c2-update[.]ru. Mitigate: upgrade to Log4j2 >= 2.17.1.
"""

# 离线技术栈——无需 API 密钥，延迟 < 200 ms
ner = NamedEntityRecognizer(
    methods=["ml", "pattern"],
    confidence_threshold=0.60,
)
entities = ner.extract_entities(advisory)
grouped  = ner.classify_entities(entities)

print("CVEs:    ", [e.text for e in grouped.get("CVE", [])])
# ['CVE-2025-44228']
print("IPs:     ", [e.text for e in grouped.get("IP", [])])
# ['203.0.113.45']
print("Domains: ", [e.text for e in grouped.get("DOMAIN", [])])
# ['c2-update[.]ru']
print("Actors:  ", [e.text for e in grouped.get("THREAT_ACTOR", [])])
# ['WIZARD SPIDER']

rel = RelationExtractor(
    method=["dependency", "pattern"],
    relation_types=["exploits", "delivers", "targets", "mitigated_by"],
    confidence_threshold=0.60,
)
relations = rel.extract_relations(advisory, entities)

tri = TripletExtractor(method="pattern", include_temporal=True)
triplets = tri.extract_triplets(advisory, entities, relations)
valid    = tri.validate_triplets(triplets)
ntriples = tri.serialize_triplets(valid, format="ntriples")
print("Valid triplets: {}".format(len(valid)))
# Valid triplets: 7
# 每个三元组编码一个结构化事实：(CVE-2025-44228, exploited_by, WIZARD_SPIDER) 等。
```

  </Tab>

  <Tab title="生命科学 — 临床/制药">
    临床试验报告包含药物、疾病、疗效结果、不良事件和试验标识符。`d4data/biomedical-ner-all` HuggingFace 模型在 PubMed 上训练，返回通用模型遗漏的生物医学专用标签（DRUG、DISEASE、GENE、OUTCOME）。Turtle 格式的 RDF 序列化可以以兼容 EMA XEVMPD 和 FDA SPL 的格式提交。

```python
from semantica.semantic_extract import (
    NamedEntityRecognizer, RelationExtractor, TripletExtractor,
)
from semantica.provenance import ProvenanceManager
from semantica.provenance.schemas import SourceReference

paper = """
Phase III trial NCT04368728: BNT162b2 (Pfizer-BioNTech) evaluated in 43,448
participants aged >= 16 years. Vaccine efficacy: 95.0% (95% CI 90.3–97.6;
p<0.0001) against symptomatic COVID-19. Bell's palsy observed in 4 vaccine
vs 0 placebo participants (not statistically significant).
"""

ner = NamedEntityRecognizer(
    method="huggingface",
    huggingface_model="d4data/biomedical-ner-all",
    confidence_threshold=0.75,
)
entities = ner.extract_entities(paper)
grouped  = ner.classify_entities(entities)

print("Drugs:    ", [e.text for e in grouped.get("DRUG", [])])
# ['BNT162b2']
print("Diseases: ", [e.text for e in grouped.get("DISEASE", [])])
# ['COVID-19', "Bell's palsy"]
print("Outcomes: ", [e.text for e in grouped.get("OUTCOME", [])])
# ['vaccine efficacy 95.0%']

rel = RelationExtractor(
    method=["llm", "dependency"],
    relation_types=["treats", "causes_adverse_event", "has_efficacy", "evaluated_in"],
    confidence_threshold=0.65,
    provider="anthropic",
    llm_model="claude-sonnet-4-6",
)
relations = rel.extract_relations(paper, entities)

tri = TripletExtractor(
    method="llm",
    provider="anthropic",
    llm_model="claude-sonnet-4-6",
    triplet_types=["treats", "has_efficacy", "causes_adverse_event"],
    include_temporal=True,
    include_provenance=True,
)
triplets = tri.extract_triplets(paper, entities, relations)
turtle   = tri.serialize_triplets(tri.validate_triplets(triplets), format="turtle")

# 溯源：将疗效数据追踪到其确切来源，以满足 ICH E6(R2) 合规
prov = ProvenanceManager(storage_path="pharma_provenance.db")
prov.track_property_source(
    "BNT162b2", "vaccine_efficacy_pct", "95.0",
    SourceReference(
        document="DOI:10.1056/NEJMoa2034577",
        page=9,
        section="Table 2",
        confidence=0.99,
        metadata={"study_id": "C4591001", "n_participants": 43448},
    )
)
```

  </Tab>

  <Tab title="银行 — 风险/合规">
    贷款协议、监管文件和信用备忘录包含法律实体、金融工具、交易对手关系和风险分类。基于模式的 NER 足够快，适用于实时贷款发起工作流，而 LLM 抽取能处理依存解析经常失败的监管文件中密集的法律术语。

```python
from semantica.semantic_extract import (
    NamedEntityRecognizer, RelationExtractor, TripletExtractor,
)

credit_memo = """
Borrower: ACME Manufacturing Ltd (registered UK). Guarantor: ACME Group PLC.
Facility: GBP 25M revolving credit at SONIA + 175bps, maturing 2028-03-31.
Collateral: first-ranking charge over ACME Manufacturing's UK fixed assets.
LTV: 67%. PD: 1.8%. LGD: 42%. RWA bucket: Standard (CRE20).
Exposure to counterparty sector: industrial manufacturing, risk tier 2.
"""

ner = NamedEntityRecognizer(
    methods=["llm", "ml", "pattern"],
    confidence_threshold=0.70,
    provider="anthropic",
    llm_model="claude-sonnet-4-6",
)
entities = ner.extract_entities(credit_memo)
grouped  = ner.classify_entities(entities)

print("Orgs:        ", [e.text for e in grouped.get("ORG", [])])
# ['ACME Manufacturing Ltd', 'ACME Group PLC']
print("Money:       ", [e.text for e in grouped.get("MONEY", [])])
# ['GBP 25M']
print("Dates:       ", [e.text for e in grouped.get("DATE", [])])
# ['2028-03-31']

rel = RelationExtractor(
    method=["llm", "dependency"],
    relation_types=["guaranteed_by", "secured_by", "classified_as", "exposed_to"],
    confidence_threshold=0.65,
    provider="anthropic",
    llm_model="claude-sonnet-4-6",
)
relations = rel.extract_relations(credit_memo, entities)

tri = TripletExtractor(
    method="llm",
    provider="anthropic",
    llm_model="claude-sonnet-4-6",
    include_temporal=True,
    include_provenance=True,
)
triplets = tri.extract_triplets(credit_memo, entities, relations)
valid    = tri.validate_triplets(triplets)
jsonld   = tri.serialize_triplets(valid, format="jsonld")
# JSON-LD 输出可直接加载到合规图谱中
# 用于 Basel III RWA 报告和 BCBS 239 数据血缘要求
```

  </Tab>
</Tabs>

<a id="common-pitfalls"></a>
## 常见陷阱

**将抽取视为保证真实的。**语义抽取产生置信度分数是有原因的——即使高置信度抽取也可能不正确。始终校验关键抽取，特别是在安全、临床或金融场景中的高风险决策。

**忽视置信度阈值。**低置信度抽取通常表示文本模糊、模型适配差或输入噪声大。设置适当的阈值（通常 0.65-0.85）可在不可靠结果污染下游处理之前将其过滤。

**跳过实体解析。**同一实体的不同提及（"NATO"、"North Atlantic Treaty Organization"、"the alliance"）会在你的知识图谱中创建重复节点。始终运行共指消解和实体去重。

**OCR 质量差或输入质量差。**语义抽取依赖于可读文本。有 OCR 错误、编码问题或大量遮蔽的文档会产生不可靠的抽取。在抽取之前清理和校验输入文本。

**在正则表达式足够时使用 LLM 抽取。**对于高度结构化的模式如 CVE 标识符（CVE-YYYY-NNNN）、IP 地址、电子邮件地址或 UUID，正则表达式比语义抽取更快、更便宜、更可靠。

**一次处理过多文本。**非常长的文档（>10,000 词）可能超出抽取模型的处理能力并产生不一致的结果。将长文档分割为逻辑分块（章节、段落）并分别处理。

**混合不兼容的抽取方法。**不同方法产生不同的实体标签模式。LLM 抽取可能返回"THREAT_ACTOR"，而 spaCy 对同一实体返回"PERSON"。跨方法归一化标签或使用一致的方法链。

<a id="choosing-your-extraction-method"></a>
## 选择你的抽取方法

六种抽取方法在速度、准确性和基础设施之间进行权衡：

- `"pattern"` 和 `"regex"` —— 无依赖，低于 5 ms，适合作为任何方法链中的最后回退。对于 CVE 标识符或 IP 地址等狭窄、可预测的领域可靠。
- `"rules"` —— 基于语言规则的检测，同样离线，低于 10 ms。
- `"ml"` / `"spacy"` —— 通用英文 NER，50-200 ms，无需 API 调用。使用 `pip install spacy && python -m spacy download en_core_web_sm` 安装。LLM 成本是考量因素时生产流水线的最佳默认选择。
- `"huggingface"` —— 领域特定微调模型，200 ms-2 s。制药使用 `d4data/biomedical-ner-all`，通用高精度 NER 使用 `dslim/bert-base-NER`。使用 `pip install "semantica[huggingface]"` 安装。
- `"llm"` —— 对隐含实体和自定义标签模式召回率最高，每文档 1-10 s。始终搭配回退使用：`methods=["llm", "ml", "pattern"]`。

回退行为是自动的：如果主要方法返回空列表，框架沿链向下查找直到找到结果或穷尽列表。模式匹配始终是隐含的最后手段。

<a id="related-guides"></a>
## 相关指南

- [溯源指南](provenance.zh-CN.md) —— 将每个抽取的实体和分块追踪回其源文档
- [智能体记忆指南](agent-memory.zh-CN.md) —— 将抽取的知识存储为可搜索的智能体记忆，并附图谱富化
- [上下文图谱指南](context-graphs.zh-CN.md) —— 抽取的实体如何填充 `ContextGraph` 节点和边
- [GraphRAG 指南](graphrag.zh-CN.md) —— 从填充后的图谱中检索事实以接地 LLM 响应
- [推理指南](reasoning.zh-CN.md) —— 在抽取的图谱上推导新事实、运行 SPARQL 查询和应用推断规则
- [语义抽取参考](../reference/semantic_extract.zh-CN.md) —— 所有抽取器类、提供者和校验器的完整 API
