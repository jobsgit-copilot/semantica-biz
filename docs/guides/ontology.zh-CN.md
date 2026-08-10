---
title: "本体管理"
description: "从你的知识图谱生成 OWL 本体，验证类层次结构，推断属性，并导出为 Turtle、RDF/XML 或 JSON-LD。"
---

**[English](ontology.md)** · **简体中文（当前）**

`OntologyGenerator` 直接从你知识图谱中已有的实体和关系推导出正式的 OWL 本体 —— 无需预先设计模式。用它为你的图谱的类和属性生成一份机器可读的契约，然后导出为 Turtle、OWL/XML 或 JSON-LD，用于 SHACL 验证、推理引擎和 STIX/TAXII 工具链。

<a id="what-is-an-ontology"></a>
## 什么是本体？

本体是对某个领域中概念及其关系的形式化规约。它通过声明以下内容为你的知识图谱定义一套共享词汇：

**类** —— 你领域中实体的类型。例如网络安全领域的 `ThreatActor`、`Vulnerability` 或 `Software`。类描述了存在哪些种类的事物。

**对象属性（Object Properties）** —— 实体之间的关系。例如 `exploits`（威胁行为者 exploits 漏洞）、`targets`（恶意软件 targets 组织）或 `uses`（行为者 uses 工具）。

**数据属性（Datatype Properties）** —— 具有字面值的属性。例如 `name`（文本）、`severity_score`（小数）、`published_date`（日期）或 `ip_address`（字符串）。

本体通过指定哪些关系是有效的、每个属性可以持有哪些类型的值，以及概念之间如何按层次结构互相关联，使你的知识图谱变得机器可读。

<a id="why-use-ontologies"></a>
## 为什么使用本体？

**一致性。** 没有本体时，一个团队可能对同一概念使用"Threat_Actor"，而另一个团队使用"ThreatActor"。本体会强制执行单一的命名约定。

**验证。** 本体支持自动检查 —— 这个关系是否合理？一个 Vulnerability 的 CVSS 评分应该是文本还是数字？

**推理。** 导出到 OWL/Turtle 后，可以让外部推理引擎（HermiT、Pellet、ELK）推断出新事实。如果 Malware 是 Software 的子类，且 HAMMERTOSS 是 Malware，那么 OWL 推理器会得出 HAMMERTOSS 也是 Software 的结论。Semantica 负责导出本体；推理本身在外部工具中运行。

**知识图谱质量。** 结构化的模式能尽早捕获错误，并确保新数据能与现有实体干净地整合。

<a id="when-to-use-when-not-to-use"></a>
## 适用场景 / 不适用场景

**在以下场景使用本体：**
- 具有复杂实体关系的正式知识图谱
- 跨多个团队或组织的数据整合
- 自动化推理和基于规则的系统
- 需要长期保持一致性的知识库
- 与期望 OWL/RDF 模式的外部工具集成

**不要在以下场景使用本体：**
- 对文本文档进行简单的语义搜索
- 向量相似度就足够的轻量级 RAG
- 模式频繁变更的原型开发
- 不需要可重用性的一次性数据分析
- 开销超过领域复杂度的场景

<Info>
Semantica 的本体模块直接从知识图谱中已有的实体和关系推导出正式的 OWL 本体 —— 无需预先设计模式。一个 6 阶段的流水线会推断类、构建层次结构、映射 OWL 类型，并序列化为 Turtle。该流水线在内存中运行；你不需要运行中的三元组库。
</Info>

---

<a id="a-simple-example"></a>
## 一个简单示例

从一个熟悉的业务领域开始，先理解其机制，再深入网络安全：

```python
from semantica.ontology import OntologyGenerator

# 直接构建一个简单的组织图谱
data = {
    "entities": [
        {"id": "e-1", "name": "Alice",            "type": "Person"},
        {"id": "e-2", "name": "Bob",              "type": "Person"},
        {"id": "e-3", "name": "Carol",            "type": "Person"},
        {"id": "e-4", "name": "Acme Corporation", "type": "Company"},
        {"id": "e-5", "name": "San Francisco",    "type": "Location"},
    ],
    "relationships": [
        {"source_id": "e-1", "target_id": "e-4", "type": "works_for"},
        {"source_id": "e-2", "target_id": "e-4", "type": "works_for"},
        {"source_id": "e-1", "target_id": "e-3", "type": "reports_to"},
        {"source_id": "e-4", "target_id": "e-5", "type": "headquartered_in"},
    ],
}

generator = OntologyGenerator(
    base_uri="https://company.example.org/ontology/",
    min_occurrences=1,
)

ontology = generator.generate_ontology(
    data,
    name="OrganizationOntology",
    build_hierarchy=True,
)

# 检查生成的内容
print(f"Classes: {len(ontology.get('classes', []))}")
# Classes: 3

print("Object Properties:")
for prop in ontology.get('properties', []):
    if prop.get('type') == 'object':
        domain = ', '.join(prop.get('domain', []))
        range_ = ', '.join(prop.get('range', []))
        print(f"  {prop['name']} ({domain} → {range_})")
# works_for (Person → Company)
# reports_to (Person → Person)
# headquartered_in (Company → Location)

print("Datatype Properties:")
for prop in ontology.get('properties', []):
    if prop.get('type') == 'data':
        print(f"  {prop['name']} ({prop.get('range')})")
# name (string)
```

**base_uri**（`https://company.example.org/ontology/`）会成为所有类和属性的命名空间前缀。`Person` 在导出的 RDF 中会变成 `<https://company.example.org/ontology/Person>`。

**对象属性**将实体连接到其他实体（`works_for`、`reports_to`）。**数据属性**将实体连接到字面值（`name`）。

---

<a id="the-graph-that-has-no-schema"></a>
## 没有模式的图谱

现在有了对模式的理解，用 CTI 数据填充一个知识图谱，让流水线的机制变得可见。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph()
ctx   = AgentContext(vector_store=vs, knowledge_graph=graph, graph_expansion=True)

ctx.store(
    [
        "CVE-2024-3400 is a critical vulnerability in PAN-OS exploited by APT29.",
        "APT29 is a Russian state-sponsored threat actor targeting NATO governments.",
        "PAN-OS is a network operating system developed by Palo Alto Networks.",
        "HAMMERTOSS is a backdoor malware used by APT29 for command-and-control.",
    ],
    extract_entities=True,
    extract_relationships=True,
)

# 此时我们有约 8 个节点和若干条边，但没有正式的模式。
# "APT29" 和一个假设的 "Lazarus Group" 都是 ThreatActor ——
# 但没有任何东西强制要求两者都必须有 attribution_confidence 属性。
print(f"Graph nodes: {len(graph.to_dict().get('nodes', []))}")
```

---

<a id="generating-the-ontology"></a>
## 生成本体

`OntologyGenerator` 读取你的图谱字典并运行 6 阶段流水线。

```python
from semantica.ontology import OntologyGenerator

generator = OntologyGenerator(
    base_uri="https://cti.example.org/ontology/",
    min_occurrences=1,   # 每个至少出现一次的实体类型都会成为一个类
)

ontology = generator.generate_from_graph(
    graph.to_dict(),
    name="CyberThreatOntology",
    build_hierarchy=True,  # 推断父子类关系
)

# 流水线产出的内容：
print(f"Classes   : {len(ontology.get('classes', []))}")
# Classes   : 4  → ThreatActor, Vulnerability, Software, Malware

print(f"Properties: {len(ontology.get('properties', []))}")
# Properties: 3  → exploits (object), targets (object), name (datatype)

# 检查一个类：
for cls in ontology.get("classes", []):
    print(f"  {cls['name']}  parent={cls.get('parent')}")
# ThreatActor   parent=None
# Vulnerability parent=None
# Software      parent=None
# Malware       parent=Software   ← hierarchy inferred because HAMMERTOSS was linked to PAN-OS (Software)
```

`Malware` 的层次结构条目表明，流水线根据关系图谱中的共现模式检测到恶意软件是软件的一个子类型。你可以在导出之前手动覆盖这些推断。

---

<a id="validating-the-schema"></a>
## 验证模式

在导出之前运行结构性验证。

```python
from semantica.ontology import validate_ontology

result = validate_ontology(ontology)

print(f"Valid: {result.get('valid', False)}")
# Valid: True

for w in result.get("warnings", []):
    print(f"  WARN : {w}")
# WARN : Class 'Malware' has no declared datatype properties

for e in result.get("errors", []):
    print(f"  ERROR: {e}")
# （无 —— 本体在结构上是健全的）
```

此时出现关于缺少数据属性的警告很常见。它意味着推断流水线在你的图谱中找到了该类，但没有任何节点带有显式的属性值。你可以在下一次导出周期之前，使用 `ClassInferrer` 和 `PropertyGenerator` 手动添加属性。

---

<a id="growing-the-ontology-as-new-node-types-emerge"></a>
## 随着新节点类型出现而增长本体

使用 `ClassInferrer` 可以增量地添加新的实体类型，而无需重新生成整个本体。

```python
from semantica.ontology import ClassInferrer, PropertyGenerator

# 细粒度控制：从新的一批实体中推断类
new_entities = [
    {"id": "kev-001", "name": "KEV-CVE-2024-3400", "type": "KEVEntry",
     "due_date": "2024-04-19", "ransomware_use": "Known"},
    {"id": "kev-002", "name": "KEV-CVE-2023-4966", "type": "KEVEntry",
     "due_date": "2023-11-14", "ransomware_use": "Unknown"},
]

inferrer = ClassInferrer()
new_classes = inferrer.infer_classes(new_entities)
# [{"id": "KEVEntry", "name": "KEVEntry", "parent": None, ...}]

# 在合并之前手动将父类设置为 Vulnerability
for cls in new_classes:
    if cls["name"] == "KEVEntry":
        cls["parent"] = "https://cti.example.org/ontology/Vulnerability"

# 检查层次结构
hierarchy = inferrer.build_class_hierarchy(new_classes)
print(hierarchy)
# {"KEVEntry": {"parent": "...Vulnerability", "children": []}}

# 合并到现有的本体中
ontology["classes"].extend(new_classes)

# 从新节点推断属性
prop_gen = PropertyGenerator()
# PropertyGenerator 读取实体属性和关系模式，
# 以生成数据属性（due_date: xsd:date）和对象属性
```

这里的模式是增量式的：对每一批新数据运行 `infer_classes`，检查结果，在流水线猜错的地方调整父类赋值，然后合并。本体会随着图谱一起增长，而不是落在后面。

---

<a id="generating-an-ontology-from-unstructured-text"></a>
## 从非结构化文本生成本体

`LLMOntologyGenerator` 使用 LLM 从散文中抽取类和属性 —— 在还没有结构化图谱时用于引导很有用。

```python
from semantica.ontology import LLMOntologyGenerator

llm_gen = LLMOntologyGenerator(provider="groq", model="llama-3.1-8b-instant")

ontology_from_text = llm_gen.generate_ontology_from_text(
    """
    APT29 (also known as Cozy Bear) is a Russian state-sponsored threat actor.
    They use spear-phishing emails to deliver HAMMERTOSS malware.
    HAMMERTOSS communicates over Twitter and GitHub to evade detection.
    The group has been observed exploiting CVE-2024-3400 in PAN-OS appliances.
    """
)

# LLM 识别出：ThreatActor、Malware、Vulnerability、Platform、CommunicationChannel
print(f"Classes extracted: {len(ontology_from_text.get('classes', []))}")

# 支持的提供者："groq"、"openai"、"anthropic"、"novita"
```

<Info>
`LLMOntologyGenerator` 最适合在还没有结构化图谱时引导一个新领域。一旦你有了图谱，请优先使用 `OntologyGenerator.generate_from_graph()` —— 它是确定性的、可复现的，并且不会在每次运行时消耗 LLM token。
</Info>

---

<a id="exporting-for-downstream-systems"></a>
## 为下游系统导出

以你的下游工具期望的格式导出。

```python
from semantica.export import export_owl, export_rdf

# OWL/XML —— 用于 Protégé、OWL API、HermiT、Pellet
export_owl(ontology, "cyber_threat.owl", format="owl-xml")

# Turtle —— 紧凑、人类可读；SHACL 工具链的首选
export_rdf(ontology, "cyber_threat.ttl", format="turtle")

# JSON-LD —— 用于 Web API 和关联数据应用
export_rdf(ontology, "cyber_threat.jsonld", format="jsonld")

# N-Triples —— 用于向三元组库（GraphDB、Stardog、Oxigraph）批量加载
export_rdf(ontology, "cyber_threat.nt", format="ntriples")
```

导出的 Turtle 文件是 Semantica SHACL 验证流水线的输入。关于如何从该本体生成约束形状并针对实时图谱数据运行这些形状，请参见 [SHACL 验证](shacl-validation.zh-CN.md)指南。

---

<a id="common-pitfalls"></a>
## 常见陷阱

**过度建模。** 10 个类就够用时不要创建 50 个。从简单开始，只有当为了推理或验证需要正式区分时才增加复杂性。为 `MaliciousEmail` 和 `PhishingEmail` 分别建立类，只有在它们具有不同的属性或关系时才有意义。

**本体漂移。** 当图谱中出现新的实体类型时，除非你重新生成或增量更新本体，否则本体会变得陈旧。请设置监控来检测当前本体未覆盖的新实体类型。

**不一致的类命名。** 选定一种约定（CamelCase、snake_case 或 kebab-case）并坚持使用。在同一个本体中混用 `ThreatActor`、`threat_actor` 和 `threat-actor` 会造成混乱，并破坏期望一致命名模式的工具链。

---

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防 — CTI/威胁">

一个国防 CTI 团队每天早上摄取原始的 OSINT 报告。本体必须与 STIX 2.1 和 NATO MISP 分类法保持互操作，因此 IRI 遵循 DoD 命名空间，并且本体导出为 OWL/XML 供该组织的 SIEM 推理插件使用。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.ingest import ingest_file, ingest_web
from semantica.ontology import OntologyGenerator, validate_ontology
from semantica.export import export_owl, export_rdf

vs    = VectorStore(backend="faiss", dimension=768)
graph = ContextGraph()
ctx   = AgentContext(vector_store=vs, knowledge_graph=graph, graph_expansion=True)

# 摄取一份行动 PDF 和 NVD 公告页面
cti_report = ingest_file("apt29_cozycar_2024.pdf", method="file")
nvd_entry  = ingest_web("https://nvd.nist.gov/vuln/detail/CVE-2024-3400", method="url")

ctx.store(
    [cti_report.text, nvd_entry.text],
    extract_entities=True,
    extract_relationships=True,
)

generator = OntologyGenerator(
    base_uri="https://ontology.dod.mil/cyber/",
    min_occurrences=1,
)
ontology = generator.generate_from_graph(
    graph.to_dict(),
    name="CyberThreatOntology",
    build_hierarchy=True,
)

result = validate_ontology(ontology)
print(f"Ontology valid: {result.get('valid')}")
# Ontology valid: True

# 用于 SIEM 推理插件的 OWL/XML；用于 SHACL 流水线的 Turtle
export_owl(ontology, "./ontologies/cyber_threat.owl", format="owl-xml")
export_rdf(ontology, "./ontologies/cyber_threat.ttl", format="turtle")

print(f"Classes    : {len(ontology.get('classes', []))}")
print(f"Properties : {len(ontology.get('properties', []))}")
# Classes    : 7  (ThreatActor, Vulnerability, Malware, Platform, Campaign, ...)
# Properties : 9  (exploits, targets, uses, name, cvss_score, ...)
```

</Tab>

<Tab title="安全 — SOC/事件">

一个 SOC 团队将零信任身份实体 —— 用户、服务账户、资源和策略 —— 建模为 OWL 本体，这样他们的策略评估引擎就可以使用共享的正式词汇，而不是硬编码的字符串。

```python
from semantica.ontology import OntologyGenerator, ClassInferrer
from semantica.export import export_owl

# 手工指定身份图谱 —— 在生产环境中这来自你的 IAM 导出
data = {
    "entities": [
        {"id": "u-1",  "name": "alice",      "type": "User"},
        {"id": "u-2",  "name": "svc-scanner","type": "ServiceAccount"},
        {"id": "r-1",  "name": "kube-api",   "type": "Resource"},
        {"id": "r-2",  "name": "s3-prod",    "type": "Resource"},
        {"id": "p-1",  "name": "ReadOnly",   "type": "Policy"},
        {"id": "p-2",  "name": "AdminAccess","type": "Policy"},
    ],
    "relationships": [
        {"source_id": "u-1", "target_id": "p-1", "type": "BOUND_TO"},
        {"source_id": "u-2", "target_id": "p-2", "type": "BOUND_TO"},
        {"source_id": "p-1", "target_id": "r-1", "type": "ALLOWS_ACCESS"},
        {"source_id": "p-2", "target_id": "r-2", "type": "ALLOWS_ACCESS"},
    ],
}

generator = OntologyGenerator(
    base_uri="https://zerotrust.corp/ontology/",
    min_occurrences=1,
)
ontology = generator.generate_ontology(data, name="ZeroTrustOntology")

# 检查流水线推断出的内容
inferrer = ClassInferrer()
classes  = inferrer.infer_classes(data["entities"])
for cls in classes:
    print(f"  Class: {cls.get('name')}")
# Class: User
# Class: ServiceAccount
# Class: Resource
# Class: Policy

export_owl(ontology, "./ontologies/zero_trust.owl", format="owl-xml")
print("Ontology exported for policy evaluation engine")
```

</Tab>

<Tab title="生命科学 — 临床/制药">

一个制药团队需要一个符合 OBO Foundry 约定（GO、CHEBI、HP）的本体，用于描述 II/III 期肿瘤试验方案。由于源数据位于 PostgreSQL 试验数据库中 —— 而不是知识图谱中 —— 他们使用 `LLMOntologyGenerator` 从散文描述中引导生成。

```python
from semantica.ontology import LLMOntologyGenerator, OntologyGenerator, validate_ontology
from semantica.export import export_owl, export_rdf
from semantica.ingest import DBIngestor

# 从临床数据库加载试验记录
db = DBIngestor()
trial_rows = db.execute_query(
    "postgresql://readonly@clindb:5432/trials",
    """
        SELECT compound, target_protein, disease_indication,
               mechanism_of_action, primary_endpoint
        FROM trial_protocols WHERE phase IN ('II','III')
    """,
)

# 为 LLM 构造一份自然语言摘要
protocol_text = "\n".join(
    f"Compound {r['compound']} targets {r['target_protein']} "
    f"in {r['disease_indication']} via {r['mechanism_of_action']}. "
    f"Primary endpoint: {r['primary_endpoint']}."
    for r in trial_rows
)

# LLM 抽取出这些类：Compound、TargetProtein、DiseaseIndication、
#   MechanismOfAction、ClinicalEndpoint、ClinicalTrial
llm_gen  = LLMOntologyGenerator(provider="openai", model="gpt-4o")
ontology = llm_gen.generate_ontology_from_text(protocol_text)

result = validate_ontology(ontology)
print(f"Valid: {result.get('valid')}")
for w in result.get("warnings", []):
    print(f"  WARN: {w}")

# 按照 OBO Foundry URI 约定导出
export_owl(ontology, "./ontologies/clinical_trial.owl", format="owl-xml")
export_rdf(ontology, "./ontologies/clinical_trial.ttl", format="turtle")
print("Ontology ready for Protégé review and OBO alignment check")
```

</Tab>

<Tab title="银行业 — 风险/合规">

一个风险团队将 Basel III / BCBS 239 的概念形式化为 OWL 本体，这样自动化合规规则就可以用共享词汇对信用风险实体进行推理 —— 替代 Python 脚本中硬编码的字段名检查。

```python
from semantica.ontology import LLMOntologyGenerator, validate_ontology
from semantica.export import export_owl, export_rdf
from semantica.ingest import ingest_file

# 摄取监管源文档
regs = [
    ingest_file("basel3_cre20.pdf", method="file"),
    ingest_file("sr_11_7.pdf",      method="file"),
    ingest_file("bcbs239.pdf",      method="file"),
]

# 使用 LLM 从监管散文中抽取概念模型
llm_gen  = LLMOntologyGenerator(provider="anthropic", model="claude-sonnet-4-20250514")
ontology = llm_gen.generate_ontology_from_text(
    "\n\n".join(r.text[:8000] for r in regs)  # 每份文档取 token 安全的节选
)

result = validate_ontology(ontology)
if not result.get("valid"):
    for err in result.get("errors", []):
        print(f"ERROR: {err}")
    # 在发布到合规规则引擎之前修复错误
else:
    print("Ontology valid — publishing to compliance registry")
    # 用于 SHACL 形状的 Turtle；用于 HermiT 推理的 OWL/XML；用于 API 的 JSON-LD
    export_owl(ontology, "./ontologies/regulatory.owl",    format="owl-xml")
    export_rdf(ontology, "./ontologies/regulatory.ttl",    format="turtle")
    export_rdf(ontology, "./ontologies/regulatory.jsonld", format="jsonld")
```

</Tab>

</Tabs>

---

<a id="related-guides"></a>
## 相关指南

- [SHACL 验证](shacl-validation.zh-CN.md) —— 从你的本体生成 W3C SHACL 约束形状，并针对实时图谱数据验证它们
- [推理与规则](reasoning.zh-CN.md) —— 在你的本体上应用前向/反向链接规则以推导新事实
- [导出与序列化](export.zh-CN.md) —— 将图谱导出为 RDF、GraphML、CSV 和 Neo4j Cypher
- [语义抽取](semantic-extraction.zh-CN.md) —— 抽取为本体生成提供输入的实体和关系
- [上下文图谱](context-graphs.zh-CN.md) —— 本体生成读取的知识图谱
