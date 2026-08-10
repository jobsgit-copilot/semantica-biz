---
title: "本体模块"
description: "自动化本体生成、SHACL 验证、OWL/RDF 导出、命名空间管理以及由 LLM 驱动的本体生成。"
icon: "sitemap"
---

**[English](ontology.md)** · **简体中文（当前）**

`semantica.ontology` 提供知识图谱模式的完整生命周期管理：

- 通过 5 阶段流水线从 KG 数据自动生成本体（语义网络 → YAML → 类型 → 层次结构 → TTL）
- 通过 `LLMOntologyGenerator` 为复杂领域进行 LLM 驱动的本体生成
- SHACL 验证：生成形状、验证图谱并获取违规报告
- 以 Turtle、RDF/XML 和 JSON-LD 格式导出 OWL/RDF
- 本体中心可视化编辑器可在 `semantica.explorer` 中使用（v0.5.0）


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `OntologyEngine` | 编排完整本体生命周期的统一门面 |
| `OntologyGenerator` | 从 KG 数据自动生成本体（5 阶段流水线） |
| `LLMOntologyGenerator` | 为复杂领域进行 LLM 驱动的本体生成 |
| `SHACLGenerator` | 从本体或 KG 模式生成 SHACL 形状 |
| `OntologyValidator` | 根据 SHACL 形状验证任意图谱：返回 `SHACLValidationReport` |
| `OWLGenerator` | 将本体序列化为 Turtle、RDF/XML、JSON-LD |
| `NamespaceManager` | IRI 生成、前缀管理和命名空间绑定 |
| `OntologyEvaluator` | 覆盖率、完整性和粒度质量指标 |
| `ClassInferrer` | 从实体类型模式推断类 |
| `PropertyGenerator` | 从实体属性和关系生成属性 |
| `AssociativeClassBuilder` | 将 N 元关系建模为中间 OWL 类 |


<a id="getting-started"></a>
## 快速上手

**`OntologyEngine`** 是你完成完整本体工作流的主要入口：

```python
from semantica.ontology import OntologyEngine

# 为你的领域初始化 base URI
engine = OntologyEngine(base_uri="https://example.org/ontology/")

# 从你的知识图谱数据生成本体
ontology = engine.from_data({"entities": entities, "relationships": relationships})

# 根据生成的 SHACL 形状验证图谱
report = engine.validate_graph(kg, ontology=ontology)
if not report.conforms:
    for v in report.violations:
        print(f"{v.severity}: {v.message} on {v.focus_node}")

# 导出为 OWL Turtle
engine.export_owl(ontology, "ontology.ttl", format="turtle")
```

<a id="ontologyengine-unified-facade"></a>
## OntologyEngine（统一门面）

**`OntologyEngine`** 编排完整的本体生命周期：**生成、验证、导出和评估**：

```python
from semantica.ontology import OntologyEngine

engine = OntologyEngine(base_uri="https://example.org/ontology/")

# 从 KG 数据生成本体
ontology = engine.from_data({"entities": entities, "relationships": relationships})

# 根据生成的 SHACL 形状验证图谱
report = engine.validate_graph(kg, ontology=ontology)
if not report.conforms:
    for v in report.violations:
        print(f"{v.severity}: {v.message} on {v.focus_node}")

# 导出为 OWL Turtle
engine.export_owl(ontology, "ontology.ttl", format="turtle")
```

<a id="ontologyengine-methods"></a>
### OntologyEngine 方法

| 方法 | 描述 |
| :------ | :----------- |
| `from_data(data)` | 对实体/关系数据运行 5 阶段流水线 |
| `validate_graph(kg, ontology=...)` | 根据生成的 SHACL 形状检查知识图谱 |
| `export_owl(ontology, path, format)` | 序列化为 `"turtle"`、`"xml"` 或 `"json-ld"` |
| `evaluate(ontology, kg)` | 计算覆盖率、完整性和粒度指标 |

<a id="ontologygenerator-5-stage-pipeline"></a>
## OntologyGenerator（5 阶段流水线）

**`OntologyGenerator`** 从你的知识图谱实体和关系自动生成形式化本体：

```python
from semantica.ontology import OntologyGenerator

generator = OntologyGenerator(base_uri="https://example.org/ontology/")
ontology  = generator.generate_ontology({
    "entities":      entities,
    "relationships": relationships,
})
```

<Steps>
  <Step title="语义网络解析">
    从源数据中的实体类型和关系结构中提取概念和模式。
  </Step>
  <Step title="YAML 转定义">
    将提取的模式转换为中间的类和属性定义。
  </Step>
  <Step title="定义转类型">
    将定义映射到 OWL 构造：`owl:Class`、`owl:ObjectProperty`、`owl:DatatypeProperty`。
  </Step>
  <Step title="层次结构生成">
    使用传递闭包和循环检测构建分类树：生成 `rdfs:subClassOf` 链。
  </Step>
  <Step title="TTL 生成">
    使用 `rdflib` 将最终本体序列化为 Turtle 格式。还提供：RDF/XML 和 JSON-LD。
  </Step>
</Steps>

<a id="shacl-validation"></a>
## SHACL 验证

从本体生成 SHACL 形状，并根据它们验证任意图谱：

```python
from semantica.ontology import SHACLGenerator, OntologyValidator, SHACLValidationReport, SHACLViolation

# 从本体生成形状
generator  = SHACLGenerator()
shapes     = generator.generate(ontology)
shapes_ttl = shapes.serialize(format="turtle")

# 根据形状验证图谱
validator = OntologyValidator()
report: SHACLValidationReport = validator.validate_graph(kg, ontology=ontology)

if not report.conforms:
    violation: SHACLViolation
    for violation in report.violations:
        print(f"{violation.severity}: {violation.message}")
        print(f"  Node: {violation.focus_node}")
        print(f"  Path: {violation.result_path}")
```

<a id="validation-report-fields"></a>
### 验证报告字段

| 字段 | 类型 | 描述 |
| :----- | :---- | :----------- |
| `conforms` | `bool` | 如果图谱通过所有 SHACL 约束则为 `True` |
| `violations` | `List[SHACLViolation]` | 详细的失败记录 |
| `focus_node` | `str` | 违规图谱节点的 IRI |
| `result_path` | `str` | 违规属性路径的 IRI |
| `severity` | `str` | `"Violation"`、`"Warning"` 或 `"Info"` |
| `message` | `str` | 人类可读的约束失败描述 |

<a id="llm-powered-ontology-generation"></a>
## LLM 驱动的本体生成

对于难以通过统计推断模式模式的复杂或新颖领域：

```python
from semantica.ontology import LLMOntologyGenerator

# 使用你偏好的 LLM 提供商初始化
generator = LLMOntologyGenerator(provider="openai")  # 或 "anthropic"、"groq" 等
ontology  = generator.generate_ontology_from_text(
    text="A biomedical ontology for clinical trial protocols involving patients, trials, interventions, and outcomes."
)
```

<a id="owl-rdf-export"></a>
## OWL / RDF 导出

```python
from semantica.ontology import OWLGenerator

generator = OWLGenerator()
generator.export_owl(ontology, path="ontology.ttl",  format="turtle")
generator.export_owl(ontology, path="ontology.owl",  format="xml")
generator.export_owl(ontology, path="ontology.json", format="json-ld")
```

<a id="namespace-management"></a>
## 命名空间管理

```python
from semantica.ontology import NamespaceManager

ns = NamespaceManager(base_uri="https://example.org/")
ns.register("ex",     "https://example.org/")
ns.register("schema", "https://schema.org/")
ns.register("owl",    "http://www.w3.org/2002/07/owl#")

# 为类和属性生成 IRI
class_iri    = ns.generate_class_iri("Person")
property_iri = ns.generate_property_iri("worksFor")
```

<a id="ontology-evaluation"></a>
## 本体评估

衡量生成本体的覆盖率、完整性和粒度：

```python
from semantica.ontology import OntologyEvaluator

evaluator = OntologyEvaluator()
result    = evaluator.evaluate_ontology(ontology, kg)

print(f"Class coverage:    {result.class_coverage:.2f}")
print(f"Property coverage: {result.property_coverage:.2f}")
print(f"Completeness:      {result.completeness:.2f}")
print(f"Granularity:       {result.granularity:.2f}")

for gap in result.gaps:
    print(f"Gap: {gap.description}")
```

<a id="common-workflows"></a>
## 常见工作流

<Tabs>
  <Tab title="快速开始">
    **用 3 步生成并验证一个本体：**

    ```python
    from semantica.ontology import OntologyEngine

    # 1. 初始化引擎
    engine = OntologyEngine(base_uri="https://yourcompany.com/ontology/")

    # 2. 从你的数据生成
    ontology = engine.from_data({"entities": entities, "relationships": relationships})

    # 3. 根据知识图谱验证
    report = engine.validate_graph(kg, ontology=ontology)
    if report.conforms:
        print("✓ Graph conforms to ontology")
    else:
        print(f"✗ Found {len(report.violations)} violations")
    ```
  </Tab>
  <Tab title="LLM 驱动的生成">
    **从文本描述生成本体：**

    ```python
    from semantica.ontology import LLMOntologyGenerator

    generator = LLMOntologyGenerator(provider="openai")
    ontology = generator.generate_ontology_from_text("""
        Create an e-commerce ontology with products, customers, orders, 
        categories, reviews, and payment methods.
    """)

    # 用额外约束进行精炼
    engine = OntologyEngine()
    validated = engine.validate(ontology)
    ```
  </Tab>
  <Tab title="导出和集成">
    **以多种格式导出本体：**

    ```python
    from semantica.ontology import OntologyEngine

    engine = OntologyEngine()

    # 导出为 OWL/Turtle 以用于 Protégé
    engine.export_owl(ontology, "schema.ttl", format="turtle")

    # 导出为 JSON-LD 以用于 Web 应用
    engine.export_owl(ontology, "schema.jsonld", format="json-ld")

    # 生成 SHACL 形状以用于验证
    engine.export_shacl(ontology, "shapes.ttl")
    ```
  </Tab>
</Tabs>

<a id="ingest-an-existing-ontology"></a>
## 摄取现有本体

加载并解析本体文件以供下游使用：

```python
from semantica.ontology import ingest_ontology

ontology_data = ingest_ontology("schema.ttl")     # Turtle
ontology_data = ingest_ontology("schema.owl")     # OWL/XML
ontology_data = ingest_ontology("schema.jsonld")  # JSON-LD
```

<Note>
  本体版本管理（`VersionManager`、`OntologyVersion`）已迁移到 `semantica.change_management`。请从那里导入：`from semantica.change_management import VersionManager`。
</Note>

- [推理](reasoning.zh-CN.md) — 在本体公理上应用推断规则。
- [知识图谱](kg.zh-CN.md) — 本体所建模的图谱。
- [导出](export.zh-CN.md) — 将本体导出为 RDF、OWL 或 JSON-LD。
- [冲突](conflicts.zh-CN.md) — 检测本体约束违规。
