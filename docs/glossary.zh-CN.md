---
title: "词汇表"
description: "Semantica 文档与代码库中使用的术语和概念的参考定义。"
icon: "book"
---

**[English](glossary.md)** · **简体中文（当前）**

<Info>
  使用 Ctrl+F / Cmd+F 在本页搜索特定术语。
</Info>

这是 Semantica 文档与代码库中所涉及的每一个概念、数据结构、算法和标准的快速参考词典。


## 核心概念

**Agent（智能体）**
一种自主的 AI 系统，能够感知环境、对信息进行推理，并采取行动以实现目标。在 Semantica 中，智能体使用知识图谱作为结构化记忆和上下文，且每一个决策都被记录为一等对象。

**Context Graph（上下文图谱）**
一个持久化、可查询的图谱，记录智能体所知道、所决定和所推理的一切：实体、关系、决策及其因果联系。它是 `semantica.context` 的核心数据结构。

**Decision（决策）**
Semantica 中的一等对象：一条被记录的智能体选择，包含类别、场景、推理过程、结果、置信度分数、因果链和来源溯源。通过 `context.record_decision()` 进行存储和检索。

**Entity（实体）**
现实世界中一个独立的事物或概念：人、组织、地点、事件或抽象概念。实体是知识图谱中的节点，每个节点都带有类型化属性和来源溯源记录。

**Knowledge Graph（知识图谱，KG）**
使用实体（节点）和关系（边）来表示知识的结构化形式。知识图谱支持推理、查询、语义搜索和可追踪的推断：这是扁平向量库所不具备的能力。

**Relationship（关系）**
两个实体之间有向的、类型化的连接：例如 `works_for`、`located_in`、`founded_by`。关系带有置信度分数，并可溯源回原始文档。

**Semantic（语义）**
与语言或逻辑中的含义相关。语义理解能够捕捉上下文和意图：超越关键词匹配，理解文本的真正含义。


## 数据处理

**Chunking（分块）**
将大文档拆分为较小的片段，同时保留语义上下文。Semantica 支持递归、语义边界、实体感知、关系感知、滑动窗口、结构化和表格感知等分块策略。

**Ingestion（数据摄取）**
从外部来源（文件、数据库、API、流）加载数据，作为统一的 `SourceDocument` 进入流水线。这是每一条 Semantica 流水线的第一阶段。

**Normalization（归一化）**
将数据标准化为一致的规范形式：将日期转换为 ISO 格式、统一实体名称、修复编码问题、清除噪声。确保下游抽取基于干净、一致的文本进行。

**Parsing（解析）**
从非结构化或半结构化文档（PDF、Word 文件、HTML、PPTX）中抽取结构化文本、版面和元数据。`DoclingParser` 还能处理多栏布局、合并单元格表格和 OCR。


## 人工智能

**Abductive Reasoning（溯因推理）**
对观察到的现象推断出最合理解释的推理方式。它是 `semantica.reasoning` 中六种推理引擎之一：根据现有证据返回最有可能的假设。

**Datalog**
一种用于知识库查询的声明式逻辑编程语言。Semantica 的 `DatalogEngine` 支持递归 Horn 子句规则，采用自底向上的半朴素不动点语义。在 v0.4.0 中引入。

**GraphRAG（图增强检索增强生成）**
一种先进的 RAG 方法，将向量相似性搜索与知识图谱遍历相结合。每一个 LLM 响应都基于结构化的图上下文，每条断言都可追溯到源节点。从而消除缺乏来源归属的幻觉。

**Inference（推理）**
使用逻辑规则从已有知识中推导出新的事实或结论：而无需这些被推导出的事实显式存在于源数据中。

**LLM（大语言模型）**
在海量文本语料上训练的 AI 模型，能够理解和生成自然语言。Semantica 集成了 8 家以上的 LLM 提供商，用于实体抽取、关系抽取和推理。

**RAG（检索增强生成）**
一种在生成响应之前从知识库中检索相关上下文以增强 LLM 输出的技术。GraphRAG 在此基础上引入了图遍历，实现更精确、更结构化的检索。


## 知识图谱组件

**Allen Interval Algebra（Allen 区间代数）**
一个由 13 种关系构成的系统，用于描述两个时间区间如何相关：before、after、meets、overlaps、during、starts、finishes、equals 及其逆关系。自 v0.4.0 起在 `TemporalKnowledgeGraph` 中得到支持。

**BiTemporalFact（双时态事实）**
一种具有两个独立时间维度的事实：*valid time*（有效时间，即其在现实世界中成立的时间）和 *transaction time*（事务时间，即其被记录到系统中的时间）。支持对缓慢变化的数据进行完整的审计追踪。

**Edge（边）**
图中两个节点之间的有向连接，表示一种类型化的关系。边携带类型、置信度分数和溯源元数据。

**Node（节点）**
知识图谱中的一个顶点，表示一个实体或概念。节点携带类型化属性、置信度分数，以及可溯源回原始文档的溯源信息。

**Property（属性）**
实体或关系的属性或特征：姓名、日期、URI、置信度分数、来源 URL。

**Temporal Graph（时态图）**
一种知识图谱，其中节点和边携带 `valid_from` / `valid_until` 时间窗口，支持时间点查询和历史状态重建。

**Triplet（三元组）**
知识的原子单位：一个 `(subject, predicate, object)` 三元组：例如 `(Apple_Inc, founded_by, Steve_Jobs)`。它是 RDF 和基于 SPARQL 的存储的基础构件。


## 实体识别与抽取

**Coreference Resolution（共指解析）**
判定文本中多个表述是否指向同一实体：例如 "Apple" 和 "the company" 都指代 Apple Inc.。由 `semantica.semantic_extract` 中的 `CoreferenceResolver` 处理。

**Entity Resolution（实体解析）**
判定不同文档中出现的两个实体指称是否指向同一个现实世界实体。也称为实体链接或去重。使用相似度评分、阻塞和语义向量嵌入来实现。

**Event Detection（事件检测）**
在文本中识别并分类事件：并购、合作、产品发布、监管决策。由 `semantica.semantic_extract` 中的 `EventDetector` 处理。

**Named Entity Recognition（命名实体识别，NER）**
在文本中识别命名实体并将其归类到预定义类别：人物、组织、地点、日期、产品以及自定义类型。提供三种模式：基于模式、基于 ML、基于 LLM。

**Relationship Extraction（关系抽取）**
识别并抽取实体之间类型化的语义关系：例如从原始文本中抽取 `(Google, acquired, DeepMind)`。


## 本体与模式

**Axiom（公理）**
本体中被接受为真的陈述，用于定义逻辑约束：例如"每个 Person 必须有一个 name"、"一个 Organization 在同一时刻最多只能有一位 CEO"。

**Class（类）**
本体中实体的一个类别或类型：`Person`、`Organization`、`Location`。类构成层次结构，并带有由 SHACL 校验的约束。

**Ontology（本体）**
对领域概念、关系和约束的形式化规约：通常以 OWL 表达。Semantica 可以从知识图谱自动生成本体，或导入已有的 OWL/RDF/Turtle 文件。

**Ontology Hub（本体中心）**
Semantica v0.5.0 推出的可视化浏览器 UI，覆盖完整的本体生命周期：可视化类编辑器、SHACL Studio、对齐编写、健康仪表盘以及版本化差异比对。

**OWL（Web Ontology Language，Web 本体语言）**
用于定义和实例化本体的 W3C 标准语言。Semantica 可以生成、导入和导出 OWL 本体。

**SHACL（Shapes Constraint Language，形状约束语言）**
W3C 标准，用于依据一组形状约束对 RDF 图进行校验。Semantica 从本体自动生成 SHACL 形状，并据此对图进行校验。

**SKOS（Simple Knowledge Organization System，简单知识组织系统）**
W3C 标准，用于表示受控词表、分类法和叙词表。在 Semantica 中用于领域词汇管理。


## 存储与检索

**Embedding（向量嵌入）**
一种稠密的数值向量，将文本、图像或其他数据表示在一个连续的语义空间中。含义相近的实体会产生距离接近的向量：从而支持相似性搜索和语义匹配。

**Graph Database（图数据库）**
一种针对使用节点和边原语存储、查询图结构数据进行优化的数据库。Semantica 支持 Neo4j、FalkorDB、Apache AGE 和 Amazon Neptune。

**Hybrid Search（混合搜索）**
一种将向量相似性搜索与关键词或元数据过滤相结合的检索策略：比单独使用任何一种方式都具有更高的准确率。

**Triplet Store（三元组库）**
一种专门用于存储和查询 RDF `(subject, predicate, object)` 三元组的数据库。Semantica 支持嵌入式 Oxigraph，以及 Blazegraph、Apache Jena 和 RDF4J。

**Vector Store（向量库）**
一种针对高维向量嵌入进行相似性存储和搜索而优化的数据库。Semantica 支持 FAISS、Pinecone、Weaviate、Qdrant、Milvus 和 PgVector。


## 图分析

**Centrality（中心性）**
衡量一个节点在图中重要性的指标。常见度量：PageRank（基于链接的重要性）、betweenness centrality（中介中心性，桥接节点）、closeness centrality（接近中心性，到所有其他节点的平均距离）。

**Community Detection（社区发现）**
识别由密集连接的节点构成的群组：内部链接多于外部链接的簇。用于发现主题社区、欺诈团伙和组织集群。

**Distance Band（距离带）**
对节点相对于某一目标的语义邻近度的分类：`near`、`mid` 或 `far`，基于向量嵌入距离阈值。属于 Distance Intelligence（v0.5.0）的一部分。

**Distance Intelligence（距离智能）**
Semantica v0.5.0 的功能，用于语义邻域探索：N×N 距离矩阵、以单个实体为中心的 ego 模式可视化，以及全图的距离带分类。

**PageRank**
一种基于入边结构衡量节点重要性的算法：最初为网页设计，可应用于任何有向图。


## 查询语言与标准

**Cypher**
Neo4j 和 FalkorDB 使用的声明式图查询语言。类似于关系数据库中的 SQL。

**Datalog**
Prolog 的一个子集，用于演绎数据库查询。Semantica 的 `DatalogEngine` 支持递归 Horn 子句规则，通过自底向上的半朴素不动点求值保证可终止。

**RDF（Resource Description Framework，资源描述框架）**
W3C 标准，以主-谓-宾三元组的形式表示信息。它是语义网以及 Semantica 三元组库的基础。

**SPARQL**
W3C 的 RDF 数据查询语言。Semantica 的 `SparqlReasoner` 使用 SPARQL 对 RDF 图进行基于查询的推理。


## 数据质量

**Conflict Resolution（冲突解决）**
处理同一知识图谱中来自多个来源的相互矛盾的事实。Semantica 的 `ConflictDetector` 负责暴露冲突；解决策略包括 prefer-most-recent（优先最新）、prefer-most-reliable（优先最可信）、majority-vote（多数表决）和 flag-for-review（标记待审）。

**Data Provenance（数据溯源）**
关于每条事实的来源、历史和血缘的完整信息：源文档、抽取方法、时间戳、置信度分数。Semantica 中符合 W3C PROV-O 规范。

**Deduplication（去重）**
识别并合并重复的实体记录。Semantica v2 策略（`blocking_v2`、`hybrid_v2`、`semantic_v2`）比 v1 快达 7 倍。

**W3C PROV-O**
W3C 的溯源本体标准。Semantica 在所有模块中以符合 PROV-O 的格式追踪血缘：满足 HIPAA、SOX、GDPR 和 FDA 21 CFR Part 11 合规要求。


## 安全术语

**SSRF（服务端请求伪造，Server-Side Request Forgery）**
一种漏洞：服务器被诱导向非预期目标发起请求。Semantica 在构造时校验 `base_url`，以防止 LLM 网关配置中出现 SSRF。

**XXE（XML 外部实体，XML External Entity）**
XML 解析器中的一种漏洞，允许攻击者读取任意文件或触发 SSRF。Semantica 的 `XMLIngestor`（v0.5.0）使用 XXE 安全的 lxml 后端。


## 另请参阅

- [核心概念](concepts.zh-CN.md) — 关键思想的深入讲解，附带代码示例。
- [入门](getting-started.zh-CN.md) — 首个可运行示例：无需任何图相关经验。
- [Modules 指南](modules.zh-CN.md) — 全部 27 个模块的说明，附带代码和流水线链。
- [API 参考](reference/context.zh-CN.md) — 每个类和方法的完整技术参考。
