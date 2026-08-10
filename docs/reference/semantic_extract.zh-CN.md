---
title: "语义抽取模块"
description: "命名实体识别、关系抽取、事件检测和三元组生成。"
icon: "magnifying-glass-chart"
---

**[English](semantic_extract.md)** · **简体中文（当前）**

`semantica.semantic_extract` 从非结构化文本中抽取结构化信息：这是 Semantica 中每个知识图谱的基础：

- `NERExtractor`：带置信度分数和来源归因的命名实体识别(NER)
- `RelationExtractor`：类型化关系抽取（`founded_by`、`located_in` 以及自定义类型）
- `TripletExtractor`：直接生成 `(subject, predicate, object)` 三元组，用于 RDF 输出
- `EventDetector`：带参与者、时态上下文和置信度的事件检测
- 每个抽取器三种抽取模式：`"pattern"`（无需 API 密钥）、`"huggingface"`、`"llm"`


<a id="getting-started"></a>
## 快速上手

<a id="prerequisites-setup"></a>
### 前提条件与设置

**第 1 步：安装依赖**

```bash
# 基础抽取（仅 pattern 方法）
pip install semantica

# 用于高级 NER 的 HuggingFace 模型
pip install semantica[models-huggingface]

# 基于 LLM 的抽取（精度最高）
pip install semantica[llm-groq]    # 或 llm-openai
```

**第 2 步：设置 API 密钥**（仅 LLM 方法需要）

```bash
export GROQ_API_KEY="your_groq_key_here"
export OPENAI_API_KEY="your_openai_key_here"
```

**第 3 步：首次抽取**

```python
from semantica.semantic_extract import NERExtractor

# 从 pattern 方法开始（无需设置）
ner = NERExtractor(method="pattern")
entities = ner.extract("Apple Inc. was founded by Steve Jobs.")
print(f"Found {len(entities)} entities")
# Output: Found 2 entities

# 升级到 LLM 以获得更好的精度
from semantica.llms import Groq
import os

llm = Groq(api_key=os.getenv("GROQ_API_KEY"))
ner = NERExtractor(method="llm", llm_provider=llm)
entities = ner.extract("Apple Inc. was founded by Steve Jobs.")
```


<a id="exported-classes"></a>
## 导出的类

<Tip>
  **`NamedEntityRecognizer`** 是带有置信度阈值和重叠合并的高级协调器。**`NERExtractor`** 是较低层的实现。对于大多数用例，为简单起见从 `NERExtractor` 开始，或需要细粒度控制时使用 `NamedEntityRecognizer`。
</Tip>

| 类 | 角色 |
| :--- | :--- |
| `NamedEntityRecognizer` | 带置信度阈值和重叠合并的高级 NER |
| `NERExtractor` | 核心 NER 实现：直接使用更简单 |
| `RelationExtractor` | 类型化关系抽取（`founded_by`、`located_in`、...） |
| `TripletExtractor` | 直接生成 `(subject, predicate, object)` 三元组，用于 RDF 输出 |
| `EventDetector` | 带参与者、时态上下文和置信度分数的事件检测 |
| `CoreferenceResolver` | 将 "Apple" 和 "the company" 解析为同一个规范实体 |
| `Entity` | `{id, text, type, confidence, start, end}` |
| `Relation` | `{subject, predicate, object, confidence}` |
| `Event` | `{type, participants, temporal, location, confidence}` |

<a id="method-selection-guide"></a>
## 方法选择指南

<Tabs>
  <Tab title="Pattern：无需设置">
    零依赖，无需 API 密钥。使用 spaCy 规则和正则表达式匹配标准实体类型。

    | | |
    | :-- | :-- |
    | **设置** | 无需设置：开箱即用 |
    | **成本** | 免费 |
    | **精度** | 对标准实体类型表现良好 |
    | **最适合** | 快速原型设计、批处理、气隙系统 |

    ```python
    from semantica.semantic_extract import NERExtractor, RelationExtractor

    ner = NERExtractor(method="pattern")
    entities = ner.extract("Apple Inc. was founded by Steve Jobs in Cupertino.")

    rel = RelationExtractor(method="pattern")
    relationships = rel.extract(text, entities=entities)
    ```
  </Tab>
  <Tab title="HuggingFace：自定义模型">
    使用任何预训练或微调的 transformer 模型。免费推断，本地运行。

    | | |
    | :-- | :-- |
    | **设置** | `pip install semantica[models-huggingface]` |
    | **成本** | 免费（本地计算） |
    | **精度** | 对领域特定的 NER 表现优异 |
    | **最适合** | 医疗 NER、自定义微调、无 API 成本 |

    ```python
    from semantica.semantic_extract import NERExtractor

    ner = NERExtractor(method="huggingface")

    # 每次调用传入模型
    entities = ner.extract(text, model="dslim/bert-base-NER", device="cpu")

    # 生物医学 NER
    entities = ner.extract(text, model="d4data/biomedical-ner-all")
    ```
  </Tab>
  <Tab title="LLM：最佳精度">
    对复杂模式和自定义实体类型精度最高。需要 LLM API 密钥。

    | | |
    | :-- | :-- |
    | **设置** | `pip install semantica[llm-groq]` + API 密钥 |
    | **成本** | 取决于提供方 |
    | **精度** | 最高：处理复杂类型和上下文 |
    | **最适合** | 生产环境、自定义实体类型、复杂关系模式 |

    ```python
    import os
    from semantica.llms import Groq
    from semantica.semantic_extract import NERExtractor, RelationExtractor, TripletExtractor

    llm = Groq(model="llama-3.3-70b-versatile", api_key=os.getenv("GROQ_API_KEY"))

    ner  = NERExtractor(method="llm",  llm_provider=llm, max_retries=3)
    rel  = RelationExtractor(method="llm", llm_provider=llm)
    trip = TripletExtractor(method="llm", llm_provider=llm)

    entities      = ner.extract(text)
    relationships = rel.extract(text, entities=entities)
    triplets      = trip.extract(text)
    ```
  </Tab>
  <Tab title="回退链">
    按优先级顺序尝试方法：即使首选方法不可用也能保证非空结果。

    ```python
    from semantica.semantic_extract import NERExtractor, RelationExtractor

    # 先尝试 LLM，出错时回退到 pattern
    ner = NERExtractor(method=["llm", "pattern"])
    rel = RelationExtractor(method=["llm", "pattern"])

    # 始终返回结果：对生产流水线安全
    entities      = ner.extract(text)
    relationships = rel.extract(text, entities=entities)
    ```

    <Tip>
      在 API 可用性不保证的流水线中（速率限制、网络问题）使用回退链。列表中的第一个方法总是最先尝试。
    </Tip>
  </Tab>
</Tabs>

<a id="method-availability-by-extractor"></a>
### 各抽取器的方法可用性

| 抽取器 | `pattern` | `huggingface` | `llm` | 备注 |
| :----------- | :----------- | :--------------- | :------- | :------- |
| `NERExtractor` | ✅ | ✅ | ✅ | 完整的方法支持 |
| `RelationExtractor` | ✅ | ✅ | ✅ | 还支持 `dependency`、`cooccurrence` |
| `TripletExtractor` | ✅ | ✅ | ✅ | 还支持 `rules` 方法 |
| `EventDetector` | ✅ | ❌ | ✅ | 仅 pattern 和 LLM |

<a id="method-fallback-chains"></a>
### 方法回退链

为提高可靠性，抽取器支持回退链，按顺序尝试方法直到成功：

```python
# 先尝试 LLM，失败时回退到 pattern
ner = NERExtractor(method=["llm", "pattern"])
rel = RelationExtractor(method=["llm", "pattern"]) 
trip = TripletExtractor(method=["llm", "pattern"])

# 始终返回结果 - 保证非空抽取
entities = ner.extract(text)
```


<a id="quick-start"></a>
## 快速开始

```python
from semantica.semantic_extract import NERExtractor, RelationExtractor, TripletExtractor
from semantica.llms import Groq
import os

text = "Apple Inc. was founded by Steve Jobs in Cupertino in 1976."
llm  = Groq(model="llama-3.3-70b-versatile", api_key=os.getenv("GROQ_API_KEY"))

entities      = NERExtractor(method="llm", llm_provider=llm).extract(text)
relationships = RelationExtractor(method="llm", llm_provider=llm).extract(text, entities=entities)
triplets      = TripletExtractor(method="llm", llm_provider=llm).extract(text)
```

<img src="/assets/img/diagrams/extraction-pipeline.svg" alt="语义抽取流水线：原始文本分发到 NER、关系和共指抽取器，然后合并到三元组生成器" style={{ width: '100%', borderRadius: '12px', margin: '0 0 24px' }} />


<a id="extractor-methods"></a>
## 抽取器方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `extract(text)` | `List[Entity]` / `List[Relation]` / `List[Triplet]` / `List[Event]` | 从单个文本输入抽取 |
| `extract(texts)` | `List[List[...]]` | 处理多个文本（自动检测批处理） |


<a id="nerextractor"></a>
## NERExtractor

```python
from semantica.semantic_extract import NERExtractor
from semantica.llms import Groq
import os

# 基于 pattern：快速、无需 API 密钥，对标准实体类型表现良好
ner = NERExtractor(method="pattern")
entities = ner.extract("Apple Inc. was founded by Steve Jobs in Cupertino.")

# 基于 HuggingFace：自定义模型，无 API 成本
ner = NERExtractor(method="huggingface")
entities = ner.extract(text, model="dslim/bert-base-NER", device="cpu")

# 基于 LLM：精度最高，处理复杂模式和自定义类型
llm = Groq(model="llama-3.3-70b-versatile", api_key=os.getenv("GROQ_API_KEY"))
ner = NERExtractor(method="llm", llm_provider=llm, max_retries=3)
entities = ner.extract(text)
```

输出格式：

```python
[
    {"text": "Apple Inc.",  "type": "ORGANIZATION", "confidence": 0.98, "start": 0,  "end": 10},
    {"text": "Steve Jobs",  "type": "PERSON",       "confidence": 0.99, "start": 27, "end": 37},
    {"text": "Cupertino",   "type": "LOCATION",     "confidence": 0.97, "start": 41, "end": 50}
]
```

<a id="custom-entity-types"></a>
### 自定义实体类型

```python
ner = NERExtractor(
    method="pattern",
    custom_entities={
        "DRUG": ["aspirin", "ibuprofen", "metformin"],
        "GENE": ["BRCA1", "TP53", "EGFR"]
    }
)
```

<Note>
  **v0.5.0 修复：** `NERExtractor(method="llm")` 在自定义网关上不再静默回退到 pattern 抽取。`response_format=json_object` 参数现在在不兼容的网关上会被有条件地省略，并自动应用普通 `generate()` + JSON 解析回退。
</Note>

<a id="relationextractor"></a>
## RelationExtractor

```python
from semantica.semantic_extract import RelationExtractor

rel = RelationExtractor(method="llm", llm_provider=llm, max_retries=3)
relationships = rel.extract(text, entities=entities)
```

输出格式：

```python
[
    {"subject": "Steve Jobs", "predicate": "founded",    "object": "Apple Inc.", "confidence": 0.92},
    {"subject": "Apple Inc.", "predicate": "located_in", "object": "Cupertino",  "confidence": 0.89}
]
```

可用方法：

- `"pattern"`：基于规则的模式匹配
- `"dependency"`：spaCy 依存解析
- `"cooccurrence"`：基于邻近性的共现
- `"huggingface"`：自定义模型
- `"llm"`：精度最高，需要 API 密钥


<a id="tripletextractor"></a>
## TripletExtractor

直接从文本生成 RDF 就绪的 `(subject, predicate, object)` 三元组：

```python
from semantica.semantic_extract import TripletExtractor

trip = TripletExtractor(method="llm", llm_provider=llm)
triplets = trip.extract(text)
# → [{"subject": "Steve Jobs", "predicate": "founded", "object": "Apple Inc.", ...}]
```

三元组适合直接加载到三元组存储或知识图谱中。


<a id="eventdetector"></a>
## EventDetector

检测带参与者和时态上下文的事件：

```python
from typing import List
from semantica.semantic_extract import EventDetector, Event

extractor = EventDetector(method="llm", llm_provider=llm)
events: List[Event] = extractor.extract(text)

for event in events:
    print(f"Event type:   {event.type}")
    print(f"Participants: {event.participants}")
    print(f"Temporal:     {event.temporal}")
    print(f"Confidence:   {event.confidence:.2f}")
```

每个事件的输出字段：

- `type`：事件类别（例如 `"founding"`、`"acquisition"`）
- `participants`：带角色的实体列表
- `temporal`：日期或时间引用
- `location`：位置实体（存在时）
- `confidence`：抽取置信度分数


<a id="coreferenceresolver"></a>
## CoreferenceResolver

在抽取之前将代词和别名引用解析为规范实体：

```python
from semantica.semantic_extract import CoreferenceResolver

resolver = CoreferenceResolver()
resolved_text = resolver.resolve(
    "Apple Inc. was founded in 1976. The company is headquartered in Cupertino."
)
# "Apple Inc." 替换 "The company"，确保下游抽取的一致性
```


<a id="batch-processing"></a>
## 批处理

所有抽取器自动检测批处理输入并高效处理多个文本：

```python
# 使用列表输入进行批处理
texts = ["Apple Inc. was founded by Steve Jobs.", "Google was founded by Larry Page.", "Microsoft was founded by Bill Gates."]

ner = NERExtractor(method="llm", llm_provider=llm)
batch_results = ner.extract(texts)  # 返回 List[List[Entity]]

# 处理结果
for i, doc_entities in enumerate(batch_results):
    print(f"Document {i}: {len(doc_entities)} entities")
    for entity in doc_entities:
        print(f"  - {entity.text} ({entity.label})")
```

**批处理输入选项：**

```python
# 选项 1：字符串列表
texts = ["Text 1...", "Text 2...", "Text 3..."]
results = ner.extract(texts)

# 选项 2：带 ID 的文档列表（添加溯源元数据）
documents = [
    {"id": "doc_1", "content": "Apple Inc. was founded by Steve Jobs."},
    {"id": "doc_2", "content": "Google was founded by Larry Page."}
]
results = ner.extract(documents)  # 实体在元数据中包含 document_id
```

<a id="using-all-extractors-together"></a>
## 组合使用所有抽取器

标准抽取流水线：实体 → 关系 → 三元组：

```python
from semantica.semantic_extract import NERExtractor, RelationExtractor, TripletExtractor
from semantica.llms import Groq
import os

llm = Groq(model="llama-3.3-70b-versatile", api_key=os.getenv("GROQ_API_KEY"))

ner  = NERExtractor(method="llm",      llm_provider=llm, max_retries=3)
rel  = RelationExtractor(method="llm", llm_provider=llm, max_retries=3)
trip = TripletExtractor(method="llm",  llm_provider=llm, max_retries=3)

entities      = ner.extract(text)
relationships = rel.extract(text, entities=entities)
triplets      = trip.extract(text)
```

<a id="extraction-method-comparison"></a>
## 抽取方法对比

| 方法 | 速度 | 成本 | 精度 | 自定义类型 |
| :------ | :----- | :---- | :-------- | :------------ |
| `pattern` | 非常快 | 免费 | 中等 | 是（词典） |
| `ml` | 快 | 免费 | 高 | 有限 |
| `llm` | 中等 | API 成本 | 最高 | 是（模式） |

- [LLM 提供方](llms.zh-CN.md) — 配置用于抽取的 LLM。
- [知识图谱](kg.zh-CN.md) — 从抽取的实体和关系构建图谱。
- [解析模块](parse.zh-CN.md) — 在抽取前解析文档。
- [去重](deduplication.zh-CN.md) — 在抽取后解析重复实体。
