---
title: "核心概念"
description: "Semantica 背后的基本理念：知识图谱、推理、溯源与时态智能详解。"
icon: "book-open"
---

**[English](concepts.md)** · **简体中文（当前）**

<Info>
  初次接触？请先从[入门](getting-started.zh-CN.md)开始，通过实操示例上手，随后再回到此处深入了解。
</Info>

Semantica 将非结构化数据——文档、网页、报告、数据库——转化为**知识图谱**：AI 系统可查询、可推理、可回溯至来源的结构化表示。

其核心在于，Semantica 在你现有的 AI 技术栈之上构建了一层**上下文与可问责性层**。它并不替代 LangChain、LlamaIndex 或你的 LLM 提供方：而是让它们的输出变得**有据可依**、**可追溯**且**可审计**。

- **上下文层（Context Layer）** —— 知识图谱、GraphRAG 检索、语义向量嵌入以及时态智能，将每一个 LLM 响应建立在结构化、可查询的事实之上。
- **可问责性层（Accountability Layer）** —— 溯源追踪、决策智能、冲突检测以及 W3C PROV-O 合规，使你 AI 技术栈中的每一条声明都可审计、可解释。
- **扩展层（Extension Layer）** —— `PluginRegistry` 与 `MethodRegistry` 让你无需修改框架代码，即可替换或增强任何组件：摄取器、抽取器、推理引擎、后端。


## 知识图谱

<img src="/assets/img/diagrams/kg-structure.svg" alt="知识图谱节点与边的结构，展示了实体（Person、Organization、Location、Date）及其类型化关系" style={{ width: '100%', borderRadius: '12px', margin: '0 0 20px' }} />

这是 Semantica 中一切的基石。知识图谱以三种构建块来存储信息：

- **节点（实体）**：人物、公司、地点、事件、概念
- **边（关系）**：`works_for`、`located_in`、`founded_by`
- **属性**：名称、日期、置信度分数、来源 URL

这种结构使知识变得**可搜索**、**可关联**、**可查询**，并且——至关重要地——**可解释**：每一个答案都能追溯到生成它的事实与关系。


## 实体抽取（NER）

扫描文本以发现并对现实世界中的实体进行分类：

```python
# 输入："Apple Inc. was founded by Steve Jobs in 1976 in Cupertino."
{
    "entities": [
        {"text": "Apple Inc.",  "type": "ORGANIZATION", "confidence": 0.98},
        {"text": "Steve Jobs",  "type": "PERSON",       "confidence": 0.99},
        {"text": "1976",        "type": "DATE",         "confidence": 0.95},
        {"text": "Cupertino",   "type": "LOCATION",     "confidence": 0.97}
    ]
}
```

每个实体都会获得一个类型、置信度分数以及指向其源文档的链接。共有三种抽取方法可用：

| 方法 | 速度 | 准确度 | 要求 |
| :------ | :----- | :-------- | :------------ |
| `"pattern"` | ⚡ 极快 | 中等 | 无需 API 密钥：基于正则表达式 |
| `"ml"` | 快 | 高 | 本地 ML 模型 |
| `"llm"` | 中等 | 最高 | LLM 提供方：支持全部 9 种 |

## 关系抽取

发现实体之间如何相互关联：

```python
{
    "relationships": [
        {"subject": "Steve Jobs", "predicate": "founded",    "object": "Apple Inc.", "confidence": 0.92},
        {"subject": "Apple Inc.", "predicate": "located_in", "object": "Cupertino",  "confidence": 0.89}
    ]
}
```

关系可以通过基于规则的方法、ML 模型或 LLM 进行抽取：每一种都会产生带有置信度分数和来源归属的类型化三元组。


## 知识图谱 vs. 向量库

两者都为 AI 检索存储信息：但它们的构建目标各不相同。

<Tabs>
  <Tab title="知识图谱">
    以类型化的节点和带标签的边存储**结构化事实**。用于回答需要理解实体间关系的问题。

    | 优势 | 为何重要 |
    | :-------- | :------------- |
    | **遍历** | 多跳查询："谁创立了 Apple 校友后来加入的公司？" |
    | **可解释性** | 每个答案都能追溯到具体的节点和边：没有黑盒检索 |
    | **时态推理** | 时间点查询、`valid_from`/`valid_until` 时间窗口、历史快照 |
    | **冲突检测** | 当两个来源对同一事实存在分歧时会被发现并予以解决 |
    | **模式约束** | SHACL 校验会在约束违反破坏结果之前捕获它们 |

    **适用场景：**你需要结构化推理、溯源、合规或可解释性时。

    ```python
    from semantica.kg import GraphBuilder, PathFinder

    graph   = GraphBuilder(merge_entities=True).build(entities=entities, relationships=rels)
    finder  = PathFinder()
    path    = finder.dijkstra_shortest_path(graph, "Steve Jobs", "Tim Cook")
    ```
  </Tab>

  <Tab title="向量库">
    存储文本分块的**稠密向量嵌入**。通过寻找语义相似的段落来回答问题：在答案结构事先未知时非常有用。

    | 优势 | 为何重要 |
    | :-------- | :------------- |
    | **模糊相似性** | 即使精确词语不匹配也能找到相关内容 |
    | **速度** | 大规模下的亚毫秒级近似最近邻搜索 |
    | **非结构化文本** | 直接处理段落、句子和原始文档 |
    | **简洁性** | 无需模式设计：嵌入并索引即可 |

    **适用场景：**你需要在大型文本语料上进行快速语义搜索时。

    ```python
    from semantica.vector_store import VectorStore

    store   = VectorStore(backend="faiss", dimension=768)
    store.add_documents(["Apple was founded in 1976.", "Google was founded in 1998."])
    results = store.search("tech company founding dates", limit=5)
    ```
  </Tab>

  <Tab title="GraphRAG（两者结合）">
    Semantica 将两者结合：向量搜索为图遍历提供种子，而图则提供向量库无法提供的结构与溯源。

    | 步骤 | 发生的过程 |
    | :---- | :----------- |
    | **查询嵌入** | 用户查询被嵌入，并通过向量相似度用于发现锚点节点 |
    | **图遍历** | 从锚点节点进行多跳遍历，检索相关实体和关系 |
    | **上下文组装** | 将事实和关系连同每条声明的来源归属一起组装 |
    | **LLM 生成** | LLM 基于检索到的结构化上下文生成有据可依的答案 |

    **结果：**响应中的每条声明都能回溯到具体的图节点：不会产生来自训练数据的幻觉，具备完整的审计追踪。

    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(advanced_analytics=True),
    )
    result = context.query("Who founded Apple?", mode="graphrag")
    ```
  </Tab>
</Tabs>


## 向量嵌入

向量嵌入将文本转换为数值向量，使 AI 系统能够度量语义相似性：即使精确用词不同，也能发现相关概念。

Semantica 将向量嵌入用于：

- **语义搜索**：按含义检索，而非仅凭关键词
- **实体解析**：在不同来源中匹配同一实体
- **先例搜索**：寻找相似的过往决策
- **GraphRAG 检索**：向量 + 图遍历的混合检索
- **距离智能（Distance Intelligence）**：任意节点集之间的 N×N 语义距离矩阵

**支持的模型：**Sentence-Transformers、FastEmbed、OpenAI、BGE、Ollama 本地嵌入。


## GraphRAG

GraphRAG（图增强检索增强生成，Graph-Augmented Retrieval Augmented Generation）通过将 LLM 响应建立在结构化知识图谱之上——而非仅凭原始文本分块——来增强其表现。

<img src="/assets/img/diagrams/graphrag-flow.svg" alt="GraphRAG 流程：用户查询 → 向量搜索 + 图遍历 → 上下文构建器 → LLM → 有据可依的答案" style={{ width: '100%', borderRadius: '12px', margin: '16px 0 20px' }} />

<Steps>
  <Step title="用户提交查询">
    查询被嵌入，并同时用于为向量搜索和图遍历提供种子。
  </Step>
  <Step title="混合上下文检索">
    Semantica 检索相关的图上下文——实体、类型化关系以及多跳推理路径——同时检索向量相似的文本分块。
  </Step>
  <Step title="上下文构建">
    将检索到的事实和推理路径组装成结构化的提示上下文，每个事实都标注其来源节点和置信度。
  </Step>
  <Step title="LLM 生成有据可依的响应">
    LLM 产生的答案中，每条声明都能回溯到图中的某个源节点：没有悬空断言，没有来自训练数据的幻觉。
  </Step>
</Steps>

<Tip>
  **GraphRAG 消除了标准 RAG 的幻觉与可追溯性问题。**标准 RAG 检索的是文本分块；而 GraphRAG 检索的是带有类型化关系的结构化事实。LLM 无法臆造图中从未存在过的结构。
</Tip>


## 本体

本体为你的知识定义了模式和规则：存在哪些实体类型、哪些关系是有效的、以及适用哪些约束。

```python
ontology = {
    "classes": ["Person", "Organization", "Location"],
    "relationships": ["works_for", "located_in", "founded_by"],
    "rules": {
        "Person":       ["must_have_name"],
        "Organization": ["must_have_name", "can_have_founding_date"]
    }
}
```

Semantica 可以从你的知识图谱自动生成本体，或导入现有的 OWL/RDF/Turtle 本体。**Ontology Hub**（v0.5.0）新增了可视化编辑器、SHACL Studio、对齐创作以及实时健康仪表盘。完整的 6 阶段生成流水线请参见 [Ontology 参考](reference/ontology.zh-CN.md)。


## 推理与推断

Semantica 包含多种推理引擎，可从已有事实中推导出新知识。

```text
已知：    Steve Jobs 创立了 Apple Inc.
已知：    Apple Inc. 的总部位于 Cupertino
推断：    Steve Jobs 与 Cupertino 存在关联
```

<Tabs>
  <Tab title="前向链接（Forward Chaining）">
    反复应用 IF/THEN 规则，直到无法推导出新事实为止。最适用于告警系统、合规检查以及基于触发器的工作流。

    ```python
    from semantica.reasoning import Reasoner, Rule, Fact, RuleType

    engine = Reasoner()
    engine.add_fact(Fact(subject="Alice", predicate="is_a", obj="Manager"))
    engine.add_rule(Rule(
        rule_type=RuleType.FORWARD_CHAIN,
        conditions=[{"subject": "?x", "predicate": "is_a", "object": "Manager"}],
        conclusion={"subject": "?x", "predicate": "has_authority", "object": "true"}
    ))
    result = engine.infer()
    ```
  </Tab>
  <Tab title="Rete 网络">
    针对大规模规则集的高效模式匹配：Rete 算法会避免重新评估前提条件未发生变化的规则。最适用于在数百万事实之上运行数千条规则。

    ```python
    from semantica.reasoning import ReteEngine

    engine = ReteEngine()
    engine.load_rules("rules/domain_rules.json")
    results = engine.run(kg)
    ```
  </Tab>
  <Tab title="演绎与溯因">
    **演绎（Deductive）**：从前提到必然结论的经典三段论推理。

    **溯因（Abductive）**：为观察到的证据推断出最可能的解释。最适用于诊断和调查类用例。

    ```python
    from semantica.reasoning import GraphReasoner

    graph_reasoner = GraphReasoner(kg)
    graph_reasoner.add_rule({"if": [{"subject": "?a", "predicate": "parent_of", "object": "?b"}], "then": {"subject": "?a", "predicate": "ancestor_of", "object": "?b"}})
    inferences = graph_reasoner.infer(kg)
    ```
  </Tab>
  <Tab title="Datalog（v0.4.0）">
    带有不动点语义的递归 Horn 子句规则：可处理前向链接无法表达的事务闭包和递归关系。

    ```python
    from semantica.reasoning import DatalogReasoner, DatalogFact, DatalogRule

    reasoner = DatalogReasoner()
    reasoner.add_fact(DatalogFact("parent", ("alice", "bob")))
    reasoner.add_rule(DatalogRule("ancestor(?X, ?Y) :- parent(?X, ?Y)."))
    reasoner.evaluate()
    results = reasoner.query("ancestor(alice, ?Z)")
    ```
  </Tab>
  <Tab title="引擎对比">

    | 引擎 | 描述 | 最适用于 |
    | :------ | :----------- | :-------- |
    | 前向链接 | 反复应用规则直到不动点 | 告警系统、合规检查 |
    | Rete 网络 | 高效模式匹配 | 大规模规则集、高事实吞吐 |
    | 演绎 | 经典三段论推理 | 数学与逻辑推断 |
    | 溯因 | 最可能的解释 | 诊断、调查 |
    | SPARQL | 基于 RDF 的查询式推断 | 语义 Web、本体推理 |
    | Datalog（v0.4.0） | 递归 Horn 子句规则 | 事务闭包、图可达性 |

  </Tab>
</Tabs>

所有引擎都产生**可解释的推理路径**：而非黑盒结论。每个推导出的事实都包含产生它的规则和前提。


## 时态智能

知识会随时间变化。时态图为节点和边附加了 `valid_from` / `valid_until` 时间窗口，支持时间点查询和历史分析。

```python
from semantica.kg import TemporalGraphQuery
from datetime import datetime

query_engine = TemporalGraphQuery(enable_temporal_reasoning=True)

# 查询图在某个特定日期的状态
snapshot = query_engine.query_at_time(kg, query="", at_time=datetime(2021, 6, 15))
```

**支持的功能：**Allen 区间代数（全部 13 种时态关系）、OWL-Time 导出、`recorded_at` 时间戳标记、时态溯源。

**常见用途：**追踪公司领导层变动、政策演变、研究时间线、金融工具历史、监管合规时间窗口。


## 距离智能

探索图中任意实体的语义邻域：用于理解概念上的邻近性、检测聚类以及可视化知识拓扑。

```python
from semantica.kg import SimilarityCalculator

calc   = SimilarityCalculator()
scores = calc.calculate_similarity(entity_a, entity_b)
```

**功能：**N×N 语义距离矩阵、自我模式（ego-mode）可视化、距离带分类（`near` / `mid` / `far`）、针对大型图的向量嵌入缓存优化。

[可视化模块](reference/visualization.zh-CN.md)将距离矩阵渲染为交互式热力图和自我模式邻域图。[Explorer](reference/explorer.zh-CN.md) 则将距离智能直接嵌入浏览器仪表盘。


## 去重与实体解析

现实世界的数据中，同一实体往往以众多别名出现："Apple"、"Apple Inc."、"Apple Computer Inc."。Semantica 的去重流水线能够检测这些重复项、合并属性、解决冲突，并保留原始来源的溯源信息。

<Tabs>
  <Tab title="策略">

    | 策略 | 算法 | 最适用于 |
    | :-------- | :--------- | :-------- |
    | `v1` | Jaro-Winkler 字符串相似度 | 小型数据集、快速基线 |
    | `blocking_v2` | 候选分块 + 相似度 | 大型语料：降低 O(n²) 比较次数 |
    | `hybrid_v2` | 分块 + 语义向量嵌入匹配 | 结构化/非结构化混合的实体名称 |
    | `semantic_v2` | 纯基于向量嵌入的解析 | 比 v1 快达 7 倍；可处理缩写与别名 |

  </Tab>
  <Tab title="配置">
    ```python
    from semantica.deduplication import DuplicateDetector, EntityMerger

    detector = DuplicateDetector(similarity_threshold=0.85)
    duplicates = detector.detect_duplicates(entities)

    merger = EntityMerger()
    deduplicated_entities = merger.merge_duplicates(entities)
    ```
  </Tab>
</Tabs>


## 溯源与可审计性

Semantica 中的每条事实都能回溯到：

- 它所来自的**源文档**
- 所使用的**抽取方法**（pattern / ML / LLM）
- 图构建期间应用的**本体规则**
- 产生任何推断事实的**推理步骤**

<Note>
  这是符合 W3C PROV-O 的血缘信息：适用于需要审计追踪的受监管行业（HIPAA、SOX、GDPR、FDA 21 CFR Part 11）。使用 `RDFExporter(include_provenance=True)` 可将溯源以内联方式嵌入任何 RDF 导出。
</Note>

```python
from semantica.provenance import ProvenanceManager

prov    = ProvenanceManager()
lineage = prov.get_entity_lineage("apple_inc")

print(f"Source:    {lineage.source_document}")
print(f"Method:    {lineage.extraction_method}")
print(f"Extracted: {lineage.timestamp}")
print(f"Checksum:  {lineage.checksum}")
```


## 决策智能

每一个智能体决策在 Semantica 中都是一等对象：被记录、存在因果关联、且可按先例检索。这是 AI 流水线的**可问责性层**：决策不再是稍纵即逝的日志消息，而是可查询的知识图谱节点。

```python
decision_id = context.record_decision(
    category="model_selection",
    scenario="Choose LLM for production pipeline",
    reasoning="GPT-4 benchmark advantage justifies 3x cost increase",
    outcome="selected_gpt4",
    confidence=0.91,
)

# 在做出新决策前，查找相似的过往决策
precedents = context.find_precedents("model selection reasoning", limit=5)

# 追溯某个过往决策的下游影响
influence  = context.analyze_decision_influence(decision_id)
```

<Tip>
  **在每一次高风险决策之前使用 `find_precedents()`。**基于所有已记录决策的混合相似度搜索能够呈现可能适用的过往推理：减少智能体多次运行之间的不一致性，并使组织能够从 AI 决策历史中进行真正的学习。
</Tip>


## 冲突检测

当多个来源对同一事实存在分歧时，Semantica 会标记并解决冲突，而非默默选取某一个值。

**解决策略：**

- **时效性（Recency）**：优先采用最新的来源
- **来源可信度（Source credibility）**：优先采用最可靠的来源（可配置可信度分数）
- **多数表决（Majority vote）**：聚合所有来源，需 ≥ 2 个一致
- **人工审查（Manual review）**：标记供人工仲裁；流水线继续运行而不阻塞

关于 `ConflictResolver`、`SourceTracker` 和 `InvestigationGuideGenerator`，请参见[冲突参考](reference/conflicts.zh-CN.md)。


## 自定义插件开发

Semantica 为扩展而设计。任何组件——摄取器、抽取器、图构建器、推理引擎——都可以通过在运行时注册的自定义实现来替换或增强。

<AccordionGroup>
  <Accordion title="PluginRegistry：按名称替换任意组件">

    `PluginRegistry` 提供跨所有模块的动态插件发现、注册和加载。将你自己的类注册到一个字符串键下；Semantica 就会在配置或流水线步骤中引用该键的任何地方使用它。

    ```python
    from semantica.core import PluginRegistry

    registry = PluginRegistry()

    # 注册一个自定义摄取器
    registry.register_plugin(
        "my_sql_ingestor", MySQLIngestor,
        version="1.0.0",
        description="PostgreSQL ingestor for internal warehouse",
        capabilities=["ingest"],
    )

    # 加载并使用
    plugin = registry.load_plugin("my_sql_ingestor", connection_string="postgresql://...")
    result = plugin.execute("SELECT * FROM documents")

    # 在流水线 YAML 中按名称引用：无需修改代码
    ```

    ```yaml
    steps:
      - name: ingest
        plugin: my_sql_ingestor
        config:
          connection_string: "${DB_URL}"
    ```

    **可用的扩展点：**摄取器、解析器、归一化器、抽取器、推理引擎、导出格式、向量库后端、图存储后端、可视化渲染器。

  </Accordion>
  <Accordion title="MethodRegistry：添加领域特定的图操作">

    `MethodRegistry` 让你按名称在知识图谱对象上注册自定义方法：适用于在不子类化的情况下添加领域特定的图操作。

    ```python
    from semantica.kg import MethodRegistry

    registry = MethodRegistry()

    def find_supply_chain_hops(graph, source_node, max_hops=3):
        """Custom BFS traversal for supply chain graphs."""
        ...

    # 注册到一个字符串键下
    registry.register("supply_chain_hops", find_supply_chain_hops)

    # 在任意图对象上按名称调用
    result = registry.call("supply_chain_hops", kg, source_node="Supplier_A", max_hops=5)

    # 列出所有已注册的方法
    print(registry.list_methods())   # ["supply_chain_hops", ...]
    ```

  </Accordion>
</AccordionGroup>

- [快速上手教程](quickstart.zh-CN.md) —— 用代码构建完整流水线。
- [模块指南](modules.zh-CN.md) —— 每个模块均附带示例讲解。
- [API 参考](reference/context.zh-CN.md) —— 完整的技术参考。
