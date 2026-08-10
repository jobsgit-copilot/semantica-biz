---
title: "SHACL 验证"
description: "从 OWL 本体生成 W3C SHACL 形状，针对结构与语义约束验证 RDF 知识图谱，并输出结构化的违规报告。"
icon: "shield-check"
---

**[English](shacl-validation.md)** · **简体中文（当前）**

<a id="what-is-shacl-validation"></a>
## 什么是 SHACL 验证？

SHACL（Shapes Constraint Language，形状约束语言）是一种用于验证图数据的标准。本体定义的是概念上的*模式*（即你的领域中"存在什么"），而 SHACL 定义的是结构性的*规则与约束*（即"应当如何组织"）。

在 Semantica 中，`SHACLGenerator` 基于你的本体生成约束规则（形状），而 `_run_pyshacl` 根据这些规则评估你的实际数据。如果某个节点违反了规则（例如缺少必需属性或使用了错误的数据类型），系统会生成一份详细的违规报告。

<a id="why-use-shacl-validation"></a>
## 为什么使用 SHACL 验证？

在运行分析、导出数据或将其输入生产模型之前，数据验证至关重要。SHACL 充当**数据质量闸门**，确保你的图数据在结构上是健全的。使用它可以发现：

- 缺少必需属性（例如没有邮箱地址的客户）。
- 数据类型不匹配（例如本应是数字却使用了字符串）。
- 基数违规（例如一个人有三个主地址）。

<a id="when-to-use-when-not-to-use"></a>
## 何时使用 / 何时不使用

- **何时使用**：你拥有一个复杂、相互关联的知识图谱，需要验证跨图谱的节点*关系*和结构完整性。SHACL 非常适合确保合并的、高度连接的数据符合你的业务规则。
- **何时不使用**：如果你只是验证扁平的 JSON 负载或单个传入的 API 请求。对于扁平数据或单条记录，应使用更简单、更快的库，例如 Pydantic 或 JSONSchema。

---

<a id="key-terms-explained"></a>
## 关键术语解释

在深入之前，以下是你将会遇到的几个概念：

- **RDF（Resource Description Framework，资源描述框架）**：一种将数据表示为图谱的标准方式。它把信息视为相互连接的"三元组"（主语 → 谓语 → 宾语）。
- **OWL（Web Ontology Language，Web 本体语言）**：一种用于构建本体的语言。它定义了你领域中存在的类和属性。
- **SHACL 形状（Shapes）**：实际的验证规则。一个"形状"针对你数据中的某个特定类（如 `Person`），并定义它必须遵循的约束（如"必须有一个出生日期"）。
- **Turtle（.ttl）**：一种流行的、人类可读的文件格式，用于存储 RDF 图数据和 SHACL 形状。

---

<a id="typical-workflow"></a>
## 典型工作流

典型的 SHACL 验证流水线遵循以下生命周期：

1. **本体**：构建一个表示你的领域的本体。
2. **SHACL 形状**：从该本体生成形状。
3. **数据图**：准备你的知识图谱。
4. **验证**：根据 SHACL 形状验证知识图谱。
5. **违规报告**：分析报告中的错误。
6. **修复**：修复数据或流水线并重新验证。

---

<a id="universal-example-employee-department"></a>
## 通用示例：员工与部门

我们来看一个简单、普遍理解的示例：确保每位 `Employee` 都属于某个 `Department`，并拥有一个 `employee_id`。

```python
from semantica.context import ContextGraph
from semantica.ontology import OntologyGenerator, SHACLGenerator, PropertyShape
from semantica.ontology.ontology_validator import _run_pyshacl

# 1. 准备你的数据图
graph = ContextGraph()
graph.add_node("emp-1", "Employee", "Alice", employee_id="E001")
graph.add_node("emp-2", "Employee", "Bob") # 缺少 employee_id，将导致违规！

# 2. 构建本体
ontology = (
    OntologyGenerator(base_uri="https://company.example.com/ontology/", min_occurrences=1)
    .generate_from_graph(graph.to_dict(), name="CompanyOntology")
)

# 3. 生成 SHACL 形状
shacl_gen = SHACLGenerator(base_uri="https://company.example.com/shapes/", severity="Violation")
shacl_graph = shacl_gen.generate(ontology)

# 注入强制约束
BASE = "https://company.example.com/ontology/"
for ns in shacl_graph.node_shapes:
    if "Employee" in ns.target_class:
        ns.property_shapes.append(
            PropertyShape(path=f"{BASE}employee_id", min_count=1, severity="Violation")
        )

# 将形状序列化为 Turtle
shacl_ttl = shacl_gen.serialize(shacl_graph, format="turtle")

# 4. 准备你的 RDF 数据图
# （用于验证时，请将你的图实例序列化为 RDF。此处使用 Turtle 字符串。）
data_ttl = """
@prefix ex: <https://company.example.com/ontology/> .

<http://example.org/emp-1> a ex:Employee ;
    ex:employee_id "E001" .

<http://example.org/emp-2> a ex:Employee .
"""

# 5. 运行验证
report = _run_pyshacl(data_ttl, shacl_ttl)

# 6. 分析报告
print(f"Graph conforms: {report.conforms}")
if not report.conforms:
    report.explain_violations() # 填充人类可读的说明
    for v in report.violations:
        print(f"Violation: {v.explanation}")
```

---

现在，让我们更深入地探讨该工作流。

<a id="step-1-build-the-ontology-from-your-merged-graph"></a>
## 第 1 步 — 从合并后的图谱构建本体

SHACL 形状派生自本体。如果你在之前的运行中已经有了本体，可以跳过此步骤。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.ontology import OntologyGenerator

graph = ContextGraph()
ctx   = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    graph_expansion=True,
)

# 加载合并后的 CTI 数据 — 在生产环境中这将是你的完整 12,000 节点图谱
ctx.store(
    [
        "APT29 is a Russian state sponsored threat actor targeting NATO governments.",
        "CVE-2024-3400 is a critical vulnerability in PAN-OS exploited by APT29.",
        "HAMMERTOSS is a backdoor malware family used by APT29 for C2 over Twitter.",
        "PAN-OS is a network operating system developed by Palo Alto Networks.",
    ],
    extract_entities=True,
    extract_relationships=True,
)

ontology = (
    OntologyGenerator(base_uri="https://cti.example.org/ontology/", min_occurrences=1)
    .generate_from_graph(graph.to_dict(), name="CyberOntology")
)

print(f"Classes inferred: {len(ontology.get('classes', []))}")
# 推断出的类：4  → ThreatActor、Vulnerability、Malware、Platform
```

---

<a id="step-2-generate-shacl-shapes-from-the-ontology"></a>
## 第 2 步 — 从本体生成 SHACL 形状

`SHACLGenerator` 生成一个 `SHACLGraph`，其中每个 OWL 类对应一个 `NodeShape`。

```python
from semantica.ontology import SHACLGenerator

shacl_gen = SHACLGenerator(
    base_uri="https://cti.example.org/shapes/",
    include_inherited=True,   # 将父类约束传播到子类
    severity="Violation",     # 所有生成形状的默认严重级别
    quality_tier="standard",  # 约束严格程度："minimal" | "standard" | "strict"
)

shacl_graph = shacl_gen.generate(ontology)

print(f"Node shapes generated: {len(shacl_graph.node_shapes)}")
# 生成的节点形状：4  — 每个类一个

for ns in shacl_graph.node_shapes:
    print(f"  {ns.target_class}  ({len(ns.property_shapes)} property constraints)")
# https://cti.example.org/ontology/ThreatActor   (2 property constraints)
# https://cti.example.org/ontology/Vulnerability  (3 property constraints)
# https://cti.example.org/ontology/Malware        (2 property constraints)
# https://cti.example.org/ontology/Platform       (1 property constraint)
```

生成的形状告诉你流水线观察到了什么。它们尚未编码你的领域*所要求的*内容。下一节展示如何注入领域特定的强制约束。

---

<a id="step-3-inject-domain-constraints"></a>
## 第 3 步 — 注入领域约束

添加流水线无法仅从数据中推断的强制 `PropertyShape` 约束。

```python
from semantica.ontology import PropertyShape

BASE = "https://cti.example.org/ontology/"

for node_shape in shacl_graph.node_shapes:

    if "Malware" in node_shape.target_class:
        # family 是必需的 — 缺少它会导致 Violation
        node_shape.property_shapes.append(
            PropertyShape(
                path=f"{BASE}family",
                min_count=1,
                severity="Violation",
            )
        )
        # attribution_confidence 是建议项 — 缺少它会导致 Warning
        node_shape.property_shapes.append(
            PropertyShape(
                path=f"{BASE}attribution_confidence",
                min_count=1,
                datatype="http://www.w3.org/2001/XMLSchema#float",
                severity="Warning",
            )
        )

    if "Vulnerability" in node_shape.target_class:
        # cvss_score 是你的检测规则所要求的
        node_shape.property_shapes.append(
            PropertyShape(
                path=f"{BASE}cvss_score",
                min_count=1,
                datatype="http://www.w3.org/2001/XMLSchema#float",
                severity="Violation",
            )
        )

    if "ThreatActor" in node_shape.target_class:
        # name 是必需的；nation_state 分类是建议项
        node_shape.property_shapes.append(
            PropertyShape(path=f"{BASE}name",         min_count=1, severity="Violation")
        )
        node_shape.property_shapes.append(
            PropertyShape(path=f"{BASE}nation_state",  min_count=1, severity="Warning")
        )

# 将最终的形状图序列化为 Turtle 以便重用和版本控制
shacl_ttl = shacl_gen.serialize(shacl_graph, format="turtle")

with open("cti_shapes.ttl", "w") as f:
    f.write(shacl_ttl)

print("Shapes written to cti_shapes.ttl")
```

你也可以从零开始手动构造形状 — 当你需要表达生成器永远不会推断的约束时，这很有用，例如对 CVE ID 字段的正则模式：

```python
from semantica.ontology import NodeShape, PropertyShape, SHACLGraph

# 要求 CVE ID 匹配规范的 NIST 格式
cve_id_shape = NodeShape(
    target_class="https://cti.example.org/ontology/Vulnerability",
    name="VulnerabilityShape",
    closed=False,
    severity="Violation",
    property_shapes=[
        PropertyShape(
            path="https://cti.example.org/ontology/cve_id",
            min_count=1,
            pattern=r"^CVE-\d{4}-\d{4,}$",   # 例如 CVE-2024-3400
            severity="Violation",
        ),
    ],
)
# 注入到现有的 shacl_graph 或构建独立的 SHACLGraph
```

---

<a id="step-4-run-validation-and-read-the-report"></a>
## 第 4 步 — 运行验证并阅读报告

将图谱序列化为 RDF，然后针对形状运行 `_run_pyshacl`。

```python
from semantica.ontology.ontology_validator import _run_pyshacl

# 准备你的 RDF 数据字符串（由于 export_rdf 主要导出结构化元数据，
# 你通常使用 rdflib 或类似工具将自定义数据图序列化为 Turtle）。
data_ttl = """
@prefix ex: <https://cti.example.org/ontology/> .

<http://example.org/malware-002> a ex:Malware .
<http://example.org/vuln-003> a ex:Vulnerability ;
    ex:cve_id "CVE24-3400" .
"""

# 运行 SHACL 验证
report = _run_pyshacl(
    data_ttl,
    shacl_ttl,
    data_graph_format="turtle",
    shacl_format="turtle",
)

# 高层级摘要
print(f"Conforms   : {report.conforms}")
# Conforms   : False   ← 发现至少一处违规

print(f"Violations : {report.violation_count}")
# Violations : 3

print(f"Warnings   : {report.warning_count}")
# Warnings   : 2

print(report.summary())
# 图谱不符合：3 处违规。
```

摘要告诉你出了问题。现在深入查看细节。

---

<a id="step-5-understand-the-violations"></a>
## 第 5 步 — 理解违规

每个 `SHACLViolation` 标识了节点、属性路径以及所需的修复。

```python
if not report.conforms:
    # 为每条违规填充通俗英文说明
    report.explain_violations()
    
    # 遍历并打印说明
    for v in report.violations:
        print(v.explanation)
    # 节点 <http://example.org/malware-002> 缺少必需属性
    #   <https://cti.example.org/ontology/family>。至少需要 1 个值。
    # 节点 <http://example.org/vuln-003> 缺少必需属性
    #   <https://cti.example.org/ontology/cvss_score>。至少需要 1 个值。
    # 节点 <http://example.org/vuln-003> 的
    #   <https://cti.example.org/ontology/cve_id> 值为 'CVE24-3400'，与要求的模式不匹配。

    # 遍历以进行程序化分诊
    for v in report.violations:
        print(f"VIOLATION  node={v.focus_node}")
        print(f"           path={v.result_path}")
        print(f"           rule={v.constraint}")
        print(f"           msg ={v.message}")
        if v.value:
            print(f"           val ={v.value}")
        if v.explanation:
            print(f"           fix ={v.explanation}")
        print()

    # 警告严重级别较低 — 评审但不阻断
    for w in report.warnings:
        print(f"WARNING  {w.focus_node}  {w.result_path}  {w.message}")
```

输出直接对应于修复任务：`malware-002` 需要添加 `family` 属性；`vuln-003` 需要 `cvss_score`，并且其 `cve_id` 需要更正为规范格式。

---

<a id="step-6-auto-remediate-common-violations"></a>
## 第 6 步 — 自动修复常见违规

标记或修补缺少必需属性的节点，然后重新验证以确认。

```python
# 将报告解析为 dict 以进行程序化处理
report_dict = report.to_dict()

# 收集缺少 'family' 属性的节点
missing_family = [
    v["focus_node"]
    for v in report_dict.get("violations", [])
    if "family" in (v.get("result_path") or "")
]

print(f"Malware nodes missing 'family': {len(missing_family)}")
# 在生产环境中：将这些排队等待分析师补充或应用默认值
# 例如 graph.update_node(node_id, {"family": "UNKNOWN — 需要分诊"})

# 修复后，重新运行验证以确认修复
# （先将修补后的图谱重新导出为 Turtle，然后再次调用 _run_pyshacl）
report2 = _run_pyshacl(patched_data_ttl, shacl_ttl)
print(f"Violations after remediation: {report2.violation_count}")
# 修复后的违规：0
```

---

<a id="common-pitfalls"></a>
## 常见陷阱

- **假设本体会自动保证数据质量**：`SHACLGenerator` 根据它在数据中观察到的内容生成形状。如果你的数据缺少某个字段，生成器不会知道它是必需的，除非你显式注入该约束（如第 3 步所示）。
- **将 `ContextGraph` 直接传递给 SHACL 验证器**：`_run_pyshacl` 函数期望的是 RDF 字符串（如 Turtle 格式），而不是原始 Python 字典或 `ContextGraph` 对象。
- **忘记 RDF 序列化**：在验证之前，你必须将图谱序列化（通常通过 `export_rdf` 使用临时文件）。
- **将验证视为一次性步骤**：验证应作为自动化步骤集成到你的 CI/CD 流水线或数据接入流程中，充当反复出现的守门员，而不是一次性脚本。
- **忽略验证报告**：不符合的图谱必须进行修复。未能评审 `violation_count` 并解决这些问题，就失去了 SHACL 验证的意义。

---

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防 — CTI/威胁">

某 DoD（美国国防部）CTI 团队在将威胁图谱共享给 ISAC 合作伙伴之前，对其强制执行 STIX 兼容的约束。每个 `ThreatActor` 必须声明 `name`，每个 `Vulnerability` 必须携带 `cvss_score`。验证闸门在每晚同步时自动运行。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.ontology import OntologyGenerator, SHACLGenerator, PropertyShape
from semantica.ontology.ontology_validator import _run_pyshacl

graph = ContextGraph()
ctx   = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=graph,
    graph_expansion=True,
)

ctx.store([
    "APT29 is a Russian state-sponsored threat actor targeting NATO governments.",
    "CVE-2024-3400 is a critical PAN-OS vulnerability with CVSS 10.0, exploited by APT29.",
    "HAMMERTOSS is a backdoor malware family used by APT29 for C2 over Twitter and GitHub.",
], extract_entities=True, extract_relationships=True)

ontology  = (
    OntologyGenerator(base_uri="https://cti.dod.mil/ontology/", min_occurrences=1)
    .generate_from_graph(graph.to_dict(), name="CTIOntology")
)

shacl_gen   = SHACLGenerator(
    base_uri="https://cti.dod.mil/shapes/",
    include_inherited=True,
    severity="Violation",
)
shacl_graph = shacl_gen.generate(ontology)

# STIX 对齐的强制字段
for ns in shacl_graph.node_shapes:
    if "ThreatActor" in ns.target_class:
        ns.property_shapes.append(
            PropertyShape(path="https://cti.dod.mil/ontology/name",         min_count=1, severity="Violation")
        )
        ns.property_shapes.append(
            PropertyShape(path="https://cti.dod.mil/ontology/nation_state",  min_count=1, severity="Warning")
        )
    if "Vulnerability" in ns.target_class:
        ns.property_shapes.append(
            PropertyShape(path="https://cti.dod.mil/ontology/cvss_score",   min_count=1, severity="Violation")
        )

shacl_ttl = shacl_gen.serialize(shacl_graph, format="turtle")

# 准备 RDF 数据字符串
data_ttl = """
@prefix ex: <https://cti.dod.mil/ontology/> .

<http://example.org/apt29> a ex:ThreatActor .
<http://example.org/cve-2024-3400> a ex:Vulnerability .
<http://example.org/hammertoss> a ex:Malware .
"""

report = _run_pyshacl(data_ttl, shacl_ttl)
print(f"CTI graph conforms : {report.conforms}")
print(f"Violations         : {report.violation_count}")
print(f"Warnings           : {report.warning_count}")

if not report.conforms:
    report.explain_violations()
    for v in report.violations:
        print(v.explanation)
    # 在违规解决之前阻断每晚的 ISAC 共享
```

</Tab>

<Tab title="安全 — SOC/事件">

某 SOC 团队在将零信任策略节点发布到策略执行点之前对其进行验证。每个 `Policy` 节点必须携带 `version`（semver 格式）和 `effective_date`。缺少任一字段的节点都是违规，会阻断发布。

```python
from semantica.context import ContextGraph
from semantica.ontology import OntologyGenerator, SHACLGenerator, PropertyShape
from semantica.ontology.ontology_validator import _run_pyshacl

graph = ContextGraph()
graph.add_node("policy-001", "Policy", "MFA Required for Tier-1 Resources",
               version="1.0.0", effective_date="2025-01-01", owner="security_team")
graph.add_node("policy-002", "Policy", "Admin Access Requires PAM Checkout")
# policy-002 没有 version 或 effective_date — 预期会违规

ontology = (
    OntologyGenerator(base_uri="https://zerotrust.corp/ontology/", min_occurrences=1)
    .generate_from_graph(graph.to_dict(), name="ZeroTrustOntology")
)

shacl_gen   = SHACLGenerator(base_uri="https://zerotrust.corp/shapes/", severity="Violation")
shacl_graph = shacl_gen.generate(ontology)

BASE = "https://zerotrust.corp/ontology/"
for ns in shacl_graph.node_shapes:
    if "Policy" in ns.target_class:
        ns.property_shapes += [
            PropertyShape(
                path=f"{BASE}version",
                min_count=1,
                pattern=r"^\d+\.\d+\.\d+$",   # semver
                severity="Violation",
            ),
            PropertyShape(
                path=f"{BASE}effective_date",
                min_count=1,
                datatype="http://www.w3.org/2001/XMLSchema#date",
                severity="Violation",
            ),
        ]

shacl_ttl = shacl_gen.serialize(shacl_graph, format="turtle")

# 准备 RDF 数据字符串
data_ttl = """
@prefix ex: <https://zerotrust.corp/ontology/> .

<http://example.org/policy-001> a ex:Policy ;
    ex:version "1.0.0" ;
    ex:effective_date "2025-01-01"^^<http://www.w3.org/2001/XMLSchema#date> .

<http://example.org/policy-002> a ex:Policy .
"""

report = _run_pyshacl(data_ttl, shacl_ttl)
print(f"Policy graph conforms: {report.conforms}")
# 策略图谱是否符合：False

for v in report.violations:
    print(f"  VIOLATION: {v.focus_node}  —  {v.result_path}  —  {v.message}")
# 违规：...policy-002 — ...version       — ...version 上的值少于 1
# 违规：...policy-002 — ...effective_date — ...effective_date 上的值少于 1
```

</Tab>

<Tab title="生命科学 — 临床/制药">

某临床信息学团队在将试验本体节点加载到试验注册系统之前对其进行验证。每个 `ClinicalTrial` 节点必须声明 `phase`（Phase I–IV 之一）、`primary_endpoint` 和 `principal_investigator`。缺少其中任何一项都会阻断注册提交。

```python
from semantica.ontology import LLMOntologyGenerator, SHACLGenerator, PropertyShape
from semantica.ontology.ontology_validator import _run_pyshacl
from semantica.export import export_rdf
import tempfile, os

llm_gen  = LLMOntologyGenerator(provider="openai", model="gpt-4o")
ontology = llm_gen.generate_ontology_from_text(
    """
    A phase II oncology trial studies the efficacy of Compound XR-401 in NSCLC patients.
    The trial is led by Principal Investigator Dr. Sarah Chen at Memorial Sloan Kettering.
    Primary endpoint: overall response rate at 24 weeks.
    Secondary endpoint: progression-free survival.
    """
)

shacl_gen   = SHACLGenerator(base_uri="https://purl.obolibrary.org/obo/TRIAL_shapes/")
shacl_graph = shacl_gen.generate(ontology)

TRIAL = "https://purl.obolibrary.org/obo/TRIAL_"
for ns in shacl_graph.node_shapes:
    if "ClinicalTrial" in ns.target_class or "Trial" in ns.target_class:
        ns.property_shapes += [
            PropertyShape(
                path=f"{TRIAL}phase",
                min_count=1,
                in_values=["Phase I", "Phase II", "Phase III", "Phase IV"],
                severity="Violation",
            ),
            PropertyShape(
                path=f"{TRIAL}primary_endpoint",
                min_count=1,
                severity="Violation",
            ),
            PropertyShape(
                path=f"{TRIAL}principal_investigator",
                min_count=1,
                severity="Warning",
            ),
        ]

shacl_ttl = shacl_gen.serialize(shacl_graph, format="turtle")
print(f"SHACL shapes generated — {len(shacl_graph.node_shapes)} node shapes")
# 已生成 SHACL 形状 — 5 个节点形状

# 验证试验数据
# 将本体序列化为数据以根据形状进行验证
tmp = tempfile.NamedTemporaryFile(suffix=".ttl", delete=False)
tmp.close()
export_rdf(ontology, tmp.name, format="turtle")
with open(tmp.name) as f:
    data_ttl = f.read()
os.unlink(tmp.name)

report = _run_pyshacl(data_ttl, shacl_ttl)
print(f"Trial data conforms: {report.conforms}")
print(f"Warnings           : {report.warning_count}")
```

</Tab>

<Tab title="银行 — 风险/合规">

某信用风险团队在每份申请进入信用模型之前，根据 Basel III CRE20 强制字段（`ltv`、`pd`、`lgd`、`asset_class`）验证每个 `LoanApplication` 节点。任何缺失的字段都是违规，会导致记录被拒绝。

```python
from semantica.context import ContextGraph
from semantica.ontology import OntologyGenerator, SHACLGenerator, PropertyShape
from semantica.ontology.ontology_validator import _run_pyshacl

graph = ContextGraph()
graph.add_node("loan-001", "LoanApplication", "Prime mortgage APP-2025-88421",
               ltv=0.78, pd=0.023, lgd=0.45, asset_class="CRE")
graph.add_node("loan-002", "LoanApplication", "SME working capital facility",
               ltv=0.65)
# loan-002 缺少 pd、lgd、asset_class — 预期三处违规

ontology = (
    OntologyGenerator(base_uri="https://basel.eba.eu/ontology/", min_occurrences=1)
    .generate_from_graph(graph.to_dict(), name="BaselRiskOntology")
)

shacl_gen   = SHACLGenerator(base_uri="https://basel.eba.eu/shapes/", severity="Violation")
shacl_graph = shacl_gen.generate(ontology)

BASE = "https://basel.eba.eu/ontology/"
for ns in shacl_graph.node_shapes:
    if "LoanApplication" in ns.target_class:
        for field in ["ltv", "pd", "lgd", "asset_class"]:
            ns.property_shapes.append(
                PropertyShape(
                    path=f"{BASE}{field}",
                    min_count=1,
                    severity="Violation",
                )
            )

shacl_ttl = shacl_gen.serialize(shacl_graph, format="turtle")

# 准备 RDF 数据字符串
data_ttl = """
@prefix ex: <https://basel.eba.eu/ontology/> .

<http://example.org/loan-001> a ex:LoanApplication ;
    ex:ltv "0.78" ;
    ex:pd "0.023" ;
    ex:lgd "0.45" ;
    ex:asset_class "CRE" .

<http://example.org/loan-002> a ex:LoanApplication ;
    ex:ltv "0.65" .
"""

report = _run_pyshacl(data_ttl, shacl_ttl)
print(f"Loan portfolio conforms: {report.conforms}")
# 贷款组合是否符合：False

print(f"Violations             : {report.violation_count}")
# 违规             : 3

for v in report.violations:
    print(f"  [{v.severity}]  {v.focus_node.split('/')[-1]}  —  {v.result_path.split('/')[-1]}")
# [Violation]  loan-002 — pd
# [Violation]  loan-002 — lgd
# [Violation]  loan-002 — asset_class

# 导出违规报告以用于监管审计追踪
report_dict = report.to_dict()
```

</Tab>

</Tabs>

---

<a id="using-shacl-validation-as-a-cicd-gate"></a>
## 将 SHACL 验证用作 CI/CD 闸门

将此函数作为发布前的闸门调用；退出码 1 会阻断流水线。

```python
import sys
from semantica.ontology import OntologyGenerator, SHACLGenerator
from semantica.ontology.ontology_validator import _run_pyshacl

def validate_before_publish(data_graph_str: str, ontology: dict) -> None:
    shacl_gen   = SHACLGenerator(base_uri="https://example.org/shapes/")
    shacl_graph = shacl_gen.generate(ontology)
    shacl_ttl   = shacl_gen.serialize(shacl_graph, format="turtle")

    report = _run_pyshacl(data_graph_str, shacl_ttl)

    if not report.conforms:
        print(f"Graph validation FAILED — {report.violation_count} violation(s)")
        report.explain_violations()
        for v in report.violations:
            print(v.explanation)
        sys.exit(1)

    print(f"Graph validation PASSED ({report.warning_count} warning(s))")
```

---

<a id="related-guides"></a>
## 相关指南

- [本体管理](ontology.zh-CN.md) — 生成派生 SHACL 形状的 OWL 本体
- [推理与规则](reasoning.zh-CN.md) — 用逻辑推断规则补充 SHACL 结构约束
- [导出与序列化](export.zh-CN.md) — 将图谱数据序列化为 Turtle/RDF/XML 以作为 `_run_pyshacl` 的输入
- [冲突解决](conflict-resolution.zh-CN.md) — 在 SHACL 验证之前检测并解决数据冲突
- [变更管理](change-management.zh-CN.md) — 在本体版本变更时对 SHACL 形状进行版本闸门控制
