---
title: "上下文模块"
description: "智能体上下文图谱、决策追踪、因果链、先例搜索、策略执行和多跳 GraphRAG。"
icon: "brain"
---

**[English](context.md)** · **简体中文（当前）**

`semantica.context` 是 AI 智能体的记忆与决策层：

- 存储带有溯源和向量嵌入检索的事实
- 将决策记录为一等图对象，包含完整因果链
- 让智能体搜索自身历史以在跨运行中保持一致
- 通过多跳 GraphRAG 遍历回答复杂查询
- 执行版本化策略并追踪合规例外


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `AgentContext` | 主入口：记忆、检索、决策、图遍历、检查点 |
| `ContextGraph` | 内存知识图谱，支持中心性、社区检测和决策追踪 |
| `AgentMemory` | 向量支持的持久记忆：`store(text)`、`retrieve(query, max_results)` |
| `EntityLinker` | 将实体提及链接到 URI；在实体 ID 之间创建类型化边 |
| `ContextRetriever` | 混合向量 + 图检索，支持最低评分和图扩展选项 |
| `DecisionRecorder` | 记录带有向量嵌入、因果链和元数据的决策 |
| `PolicyEngine` | 策略管理：`add_policy()`、`check_compliance()`、`get_applicable_policies()` |
| `CausalChainAnalyzer` | 追踪决策如何相互影响：`get_causal_chain(decision_id)` |


<a id="what-you-get"></a>
## 你将获得

- **AgentContext** — 通过一个 API 提供记忆、决策追踪和图支持的检索
  - 对话历史和检查点差异对比
  - 将完整上下文状态持久化和恢复到磁盘
- **ContextGraph** — 线程安全的内存知识图谱
  - PageRank、中心性、社区检测、时间有效性
  - 跨图导航和链接遍历
- **AgentMemory** — 向量嵌入支持的记忆，带保留策略
  - 在可配置的 `max_memory_size` 下进行 LRU 淘汰
  - 每对话历史隔离
- **DecisionRecorder** — 记录带有因果链和置信度评分的决策
  - 时间有效期窗口（`valid_from` / `valid_until`）
  - 每次决策捕获跨系统上下文
- **PolicyEngine** — 知识图谱中的版本化策略存储
  - 针对已记录决策的合规检查
  - 带审批者审计追踪的策略例外追踪
- **EntityLinker** — 将实体文本映射到稳定 URI
  - 在实体 ID 之间创建类型化链接
  - 防止 "Apple"、"Apple Inc."、"AAPL" 成为独立节点
- **ContextRetriever** — 融合向量相似度、图遍历和智能体记忆
  - 比纯向量搜索更丰富的上下文
  - 可配置的 `hybrid_alpha` 和扩展跳数
- **CausalChainAnalyzer** — 追踪任何决策的上游原因和下游影响
  - 带关系类型的可解释性路径
  - 可配置的深度和方向


<a id="quick-start"></a>
## 快速上手

<Steps>
  <Step title="初始化智能体上下文">
    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(advanced_analytics=True),
        decision_tracking=True,
        retention_days=90,
        max_memories=50000,
    )
    ```
  </Step>
  <Step title="存储事实并按语义相似度检索">
    ```python
    memory_id = context.store(
        "GPT-4 outperforms GPT-3.5 on reasoning benchmarks by 40%",
        metadata={"source": "openai_blog", "date": "2024-01"}
    )

    results = context.retrieve("LLM benchmark comparisons", max_results=5)
    for r in results:
        print("{} (score: {:.3f})".format(r["content"], r["score"]))
    ```
  </Step>
  <Step title="记录带有完整溯源的决策">
    ```python
    decision_id = context.record_decision(
        category="model_selection",
        scenario="Choose LLM for production reasoning pipeline",
        reasoning="GPT-4 benchmark advantage justifies 3x cost increase",
        outcome="selected_gpt4",
        confidence=0.91,
        entities=["gpt-4", "gpt-3.5"],
        decision_maker="pipeline_agent",
    )
    ```
  </Step>
  <Step title="查找先例并追踪因果链">
    ```python
    # 搜索过往决策：防止跨运行做出矛盾的选择
    precedents = context.find_precedents("model selection reasoning", limit=5)
    for p in precedents:
        print("[{}] {}  (confidence: {:.2f})".format(p.category, p.outcome, p.confidence))
        print("  Reasoning: {}".format(p.reasoning))

    # 追踪受此决策影响的下游决策
    chain = context.get_causal_chain(decision_id, direction="downstream", max_depth=5)
    print("Downstream decisions: {}".format(len(chain)))

    # 完整的可解释性
    explanation = context.trace_decision_explainability(decision_id)
    print("Total connections: {}".format(explanation["total_connections"]))
    ```
  </Step>
</Steps>


<a id="usage-patterns"></a>
## 使用模式

<Tabs>
  <Tab title="仅向量记忆">
    最快的配置：无知识图谱。最适合需要对事实进行语义搜索而无需图遍历开销的智能体。

    ```python
    from semantica.context import AgentContext
    from semantica.vector_store import VectorStore

    # 零图配置：仅向量记忆
    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
    )

    context.store("User prefers concise responses with code examples")
    context.store("Project uses Python 3.11 with FastAPI and PostgreSQL")

    results = context.retrieve("user coding preferences", max_results=5)
    for r in results:
        print("{:.3f}  {}".format(r["score"], r["content"]))
    ```

    <Check>
      将 `backend="faiss"` 改为 `backend="inmemory"` 以进行零依赖的本地开发。
    </Check>
  </Tab>
  <Tab title="完整智能体上下文">
    生产配置：图 + 决策 + 分析。当你需要可解释性和无矛盾的决策历史时使用。

    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(
            advanced_analytics=True,  # PageRank、中心性、社区检测
            kg_algorithms=True,       # 寻路、链接预测
        ),
        decision_tracking=True,       # 需要 knowledge_graph
        retention_days=90,
        max_memories=50000,
    )

    decision_id = context.record_decision(
        category="model_selection",
        scenario="Choose LLM for production reasoning pipeline",
        reasoning="GPT-4 benchmark advantage justifies 3x cost",
        outcome="selected_gpt4",
        confidence=0.91,
        entities=["gpt-4", "gpt-3.5"],
    )

    # 防止跨运行的矛盾
    precedents = context.find_precedents("model selection", limit=5)
    ```

  </Tab>
  <Tab title="GraphRAG 查询">
    加载预构建的知识图谱，通过多跳图遍历回答复杂问题。

    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(advanced_analytics=True),
        hybrid_alpha=0.4,        # 0.0 = 纯向量  →  1.0 = 纯图
        max_expansion_hops=3,
    )

    # 加载预构建的知识图谱
    context.load_graph("company_kg.json")

    # 多跳 GraphRAG 检索
    results = context.retrieve(
        "companies founded by Apple alumni",
        use_graph=True,
        max_results=10,
    )
    for r in results:
        print("[{:.3f}] {}".format(r["score"], r["content"]))
    ```

    <Tip>
      增加 `max_expansion_hops` 可获得更深的遍历，但会增加延迟。从 2 开始并向上调整。
    </Tip>
  </Tab>
  <Tab title="策略执行">
    添加版本化合规策略，并在记录之前根据策略检查每个决策。

    ```python
    from semantica.context import AgentContext, ContextGraph, PolicyEngine
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(),
        decision_tracking=True,
    )

    engine = PolicyEngine(knowledge_graph=context.knowledge_graph)

    engine.add_policy(
        name="data_privacy",
        description="No PII stored without user consent flag",
        version="1.2",
        effective_date="2024-01-01",
        category="privacy",
        rules={"requires_consent": True, "max_retention_days": 90},
    )

    decision_data = {"action": "store_user_email", "user_consent": True}
    result = engine.check_compliance(decision_data, policy_names=["data_privacy"])

    if result["compliant"]:
        context.record_decision(
            category="data_storage",
            scenario="Store user profile",
            outcome="stored",
            confidence=1.0,
        )
    else:
        print("Blocked by policy:", result["violations"])
    ```
  </Tab>
</Tabs>


<a id="agentcontext"></a>
## AgentContext

**`AgentContext`** 是主入口。在**单一统一 API** 背后封装记忆、图和决策追踪。

<a id="constructor-parameters"></a>
### 构造函数参数

| 参数 | 类型 | 默认值 | 描述 |
| :--------- | :---- | :------- | :----------- |
| `vector_store` | `VectorStore` | **必填** | 基于向量嵌入的记忆检索后端 |
| `knowledge_graph` | `ContextGraph` | `None` | 启用图支持的关系和 GraphRAG |
| `decision_tracking` | `bool` | `False` | 激活 `DecisionRecorder`：需要同时设置 `knowledge_graph` |
| `retention_days` | `Optional[int]` | `30` | 自动过期超过 N 天的记忆；`None` = 永久保留 |
| `max_memories` | `int` | `10000` | LRU 淘汰前的硬上限 |
| `graph_expansion` | `bool` | `True` | 从存储的记忆自动扩展图 |
| `max_expansion_hops` | `int` | `2` | 检索时图扩展的最大跳数 |
| `hybrid_alpha` | `float` | `0.5` | 向量（`0.0`）和图（`1.0`）检索之间的平衡 |
| `advanced_analytics` | `bool` | `True` | 启用 PageRank、中心性和社区分析 |
| `kg_algorithms` | `bool` | `True` | 添加寻路和链接预测 |

<Tip>
  **设置 `retention_days` 以避免记忆膨胀。** 默认值 `30` 会自动修剪。合规关键的智能体可能需要 `retention_days=None` 并通过 `export()` 进行显式归档。
</Tip>

<Tip>
  **在运行之间持久化你的上下文。** `VectorStore` 不会自动持久化——向其构造函数传入 `index_path=` 是无效操作。调用 `context.save("agent_state/")` 将记忆、向量索引和图写入磁盘，并在下一个进程中调用 `context.load("agent_state/")` 来恢复它们。请参阅下方[真实场景模式](#real-world-patterns)中的"持久化与恢复"标签页。
</Tip>

<a id="memory-methods"></a>
### 记忆方法

| 方法 | 返回 | 描述 |
| :------ | :------- | :----------- |
| `store(content, metadata, conversation_id, user_id)` | `str` 或 `Dict` | 存储事实（str → 记忆 ID）或文档列表（list → 统计字典） |
| `batch_store(items)` | `List[str]` | 一次存储多个项目：返回记忆 ID 列表 |
| `retrieve(query, max_results, min_score, use_graph, conversation_id)` | `List[Dict]` | 语义检索；如果设置了 `knowledge_graph` 则自动选择 GraphRAG |
| `forget(memory_id, conversation_id, days_old)` | `int` | 按 ID、对话或时效删除记忆 |
| `update(memory_id, content, metadata)` | `bool` | 更新已存储记忆的内容或元数据 |
| `get_memory(memory_id)` | `Optional[Dict]` | 按 ID 获取特定记忆 |
| `stats()` | `Dict` | 记忆计数、向量库状态、图统计 |
| `health()` | `Dict` | 系统健康状态：所有后端、状态标志 |
| `save(path)` | `None` | 将完整上下文状态（记忆 + 图）持久化到磁盘 |
| `load(path)` | `None` | 从磁盘恢复上下文状态 |
| `export(conversation_id, format)` | `str \| Dict` | 将记忆导出为 JSON 或字典 |
| `import_data(data, format)` | `int` | 从 JSON 或字典导入记忆 |

<Tip>
  **`retrieve()` 使用 `max_results=`，不是 `top_k=`。** 参数是 `max_results`（默认 `5`）。传入 `use_graph=True` 强制使用 GraphRAG，或 `use_graph=False` 强制仅向量检索，无论是否配置了 `knowledge_graph`。
</Tip>

<a id="conversation-methods"></a>
### 对话方法

```python
# 在对话线程中存储轮次
context.store("User asked about deployment options", conversation_id="conv_001")
context.store("Agent recommended Docker + Kubernetes", conversation_id="conv_001")

# 检索完整对话历史
history = context.conversation("conv_001", max_items=50)
for turn in history:
    print("[{}] {}".format(turn["timestamp"], turn["content"]))

# 跨所有对话通过查询检索
results = context.retrieve(
    "deployment recommendations",
    conversation_id="conv_001",
    max_results=10,
)
```

<a id="multi-hop-graphrag"></a>
### 多跳 GraphRAG

**需要在构造时设置 `knowledge_graph`**：为 LLM 锚定的多跳遍历启用 `query_with_reasoning()`：

```python
import os
from semantica.llms import Groq

llm    = Groq(model="llama-3.3-70b-versatile", api_key=os.getenv("GROQ_API_KEY"))
result = context.query_with_reasoning(
    query="What technologies have we chosen and why?",
    llm_provider=llm,
    max_hops=2,
    max_results=10,
)

print(result["response"])
print("Confidence: {:.2f}".format(result["confidence"]))
print("Sources used: {}".format(result["num_sources"]))
```

<a id="decision-methods"></a>
### 决策方法

| 方法 | 返回 | 描述 |
| :------ | :------- | :----------- |
| `record_decision(category, scenario, reasoning, outcome, confidence, entities, decision_maker, valid_from, valid_until)` | `str` | 记录决策；如果 `decision_tracking=False` 或没有 `knowledge_graph` 则抛出 `RuntimeError` |
| `find_precedents(scenario, category, limit, use_hybrid_search, max_hops, as_of)` | `List[Decision]` | 通过语义 + 结构相似度查找相似的过往决策 |
| `query_decisions(query, max_hops, use_hybrid_search)` | `List[Decision]` | 广泛的上下文感知决策搜索 |
| `get_causal_chain(decision_id, direction, max_depth)` | `List[Decision]` | 追踪 `"upstream"` 原因或 `"downstream"` 影响 |
| `trace_decision_explainability(decision_id)` | `Dict` | 完整可解释性：原因、影响、关系路径 |
| `get_policy_engine()` | `PolicyEngine` | 访问活动的 `PolicyEngine` 实例 |

<Warning>
  `decision_tracking=True` 需要同时设置 `knowledge_graph`。没有它，`record_decision()` 会抛出 `RuntimeError`。
</Warning>

<Tip>
  **在每次重要决策之前使用 `find_precedents()`。** 这是上下文模块防止智能体在跨运行中做出矛盾选择的方式。将先例作为上下文呈现给 LLM："我们之前因为类似原因选择了 X。"
</Tip>

<a id="checkpoint-methods"></a>
### 检查点方法

**非常适合审计推理循环**：在每次通过前后创建快照，以准确查看变更内容：

```python
# 对当前图状态创建命名快照
context.checkpoint("before_inference")

# ... 运行推理、记录决策 ...

context.checkpoint("after_inference")

# 准确查看添加/删除了什么
diff = context.diff_checkpoints("before_inference", "after_inference")
print("Decisions added: {}".format(len(diff["decisions_added"])))
print("Relationships added: {}".format(len(diff["relationships_added"])))

# 通过 TemporalVersionManager 将检查点持久化到磁盘
context.flush_checkpoint("after_inference")
```


<a id="contextgraph"></a>
## ContextGraph

**`ContextGraph`** 是支撑 `AgentContext` 的知识图谱。也可以**独立使用**，在没有完整上下文层的情况下进行关系建模。

```python
from semantica.context import ContextGraph

graph = ContextGraph(advanced_analytics=True)

# 构建图
graph.add_node("Python",  "language",  properties={"paradigm": "multi-paradigm"})
graph.add_node("FastAPI", "framework", properties={"language": "Python"})
graph.add_edge("Python", "FastAPI", "enables")

# 直接在图上记录和查询决策
decision_id = graph.record_decision(
    category="technology_choice",
    scenario="Web API framework selection",
    reasoning="FastAPI's async support and auto-docs match our requirements",
    outcome="selected_fastapi",
    confidence=0.92,
    entities=["Python", "FastAPI"],
)

similar = graph.find_precedents_by_scenario("web framework", limit=3)
stats   = graph.stats()
print("Nodes: {}, Edges: {}".format(stats["node_count"], stats["edge_count"]))
```

<a id="constructor-options"></a>
### 构造函数选项

| 参数 | 类型 | 默认值 | 描述 |
| :--------- | :---- | :------- | :----------- |
| `advanced_analytics` | `bool` | `True` | PageRank、介数中心性 |
| `centrality_analysis` | `bool` | `True` | 完整中心性套件 |
| `community_detection` | `bool` | `True` | Louvain 社区聚类 |
| `node_embeddings` | `bool` | `True` | Node2Vec 向量嵌入用于结构相似度 |

<a id="contextgraph-full-method-reference"></a>
### ContextGraph：完整方法参考

| 方法 | 返回 | 描述 |
| :------ | :------- | :----------- |
| `add_node(node_id, node_type, properties, valid_from, valid_until)` | `None` | 添加节点；支持时间有效期窗口 |
| `add_edge(source_id, target_id, edge_type, weight, properties)` | `None` | 添加带可选权重的有向边 |
| `add_nodes(nodes)` | `int` | 从字典列表批量添加；返回添加数量 |
| `add_edges(edges)` | `int` | 批量添加边；返回添加数量 |
| `get_neighbors(node_id, hops)` | `List[Dict]` | 到给定深度的 BFS 邻居 |
| `get_neighbor_distances(node_id, hops)` | `List[Dict]` | 带置信度衰减评分的邻居 |
| `find_node(node_id)` | `Optional[Dict]` | 按 ID 查找单个节点 |
| `find_nodes(node_type, skip, limit)` | `List[Dict]` | 按类型筛选节点并分页 |
| `find_active_nodes(node_type, at_time)` | `List[Dict]` | 在给定时间戳有效的节点 |
| `find_edges(edge_type, skip, limit)` | `List[Dict]` | 按类型筛选边并分页 |
| `record_decision(category, scenario, reasoning, outcome, confidence, entities, decision_maker)` | `str` | 添加带因果边的决策节点 |
| `find_precedents_by_scenario(scenario, category, limit, use_semantic_search, as_of)` | `List[Dict]` | 语义相似的过往场景 |
| `query(query, skip, limit)` | `List[Dict]` | 对节点内容的全文搜索 |
| `stats()` | `Dict` | 节点/边计数、类型分布、图密度 |
| `density()` | `float` | 图密度评分 |
| `save_to_file(path)` | `None` | 将图持久化为 JSON |
| `load_from_file(path)` | `None` | 从 JSON 加载图 |
| `build_from_conversations(conversations, link_entities)` | `Dict` | 从对话数据构建图 |
| `link_graph(other_graph, source_node_id, target_node_id, link_type)` | `str` | 创建跨图导航链接；返回 `link_id` |
| `navigate_to(link_id)` | `Tuple` | 跟随跨图链接到 `(target_graph, target_node_id)` |
| `cross_graph_path(source_node_id, target_graph, target_node_id, max_hops)` | `Dict` | 跨链接图的最短路径 |
| `clear()` | `None` | 重置图状态和所有索引 |

<a id="distance-intelligence-v050"></a>
### 距离智能（v0.5.0）

`ContextGraph` 暴露了完整的距离智能 API，用于探索语义邻域并将邻近度融入检索。

<Info>
  完整的距离智能参考——距离矩阵、API 端点、向量嵌入缓存、Explorer UI——在专门的[距离智能](distance.zh-CN.md)页面中有详细介绍。本节记录上下文层 API。
</Info>

<a id="neighbors-with-distance-metadata"></a>
### 带距离元数据的邻居

向 `get_neighbors()` 传入 `include_distance_metadata=True`，以在每个邻居旁边接收距离带、置信度衰减和路径信息：

```python
graph = ContextGraph(advanced_analytics=True)

# ... 填充图 ...

neighbors = graph.get_neighbors(
    "python",
    hops=3,
    include_distance_metadata=True,
    min_weight=0.3,   # exclude low-confidence edges
)

for n in neighbors:
    print(
        f"{n['node_id']:15s}  "
        f"band={n['distance_band']:10s}  "
        f"decay={n['confidence_decay']:.3f}  "
        f"hops={n['hop_count']}"
    )
```

| 新增字段 | 类型 | 描述 |
| :---------- | :---- | :----------- |
| `distance_band` | `str` | `"direct"`（1 跳）/ `"near"`（2）/ `"mid-range"`（3–4）/ `"distant"`（5+） |
| `confidence_decay` | `float` | `edge_weight ^ hop_count`——随每跳衰减 |
| `path_to_anchor` | `List[str]` | 从锚节点到此邻居的最短路径 |
| `hop_count` | `int` | 从锚节点的 BFS 深度 |

<a id="proximity-blended-retrieval"></a>
### 近邻混合检索

在 `AgentContext` 上设置 `proximity_weight`，将图邻近度融入每次 `retrieve()` 和 `find_precedents()` 调用：

```python
context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=ContextGraph(advanced_analytics=True),
    proximity_weight=0.3,   # 0.7×semantic + 0.3×proximity
)

# combined_score 与 semantic_score 和 proximity_score 一起返回
results = context.retrieve("web API frameworks", max_results=10)
for r in results:
    print(
        f"[{r['combined_score']:.3f}]  "
        f"semantic={r['semantic_score']:.3f}  "
        f"proximity={r['proximity_score']:.3f}  "
        f"{r['content'][:60]}"
    )

# 按调用覆盖权重
precedents = context.find_precedents(
    "infrastructure scaling decisions",
    proximity_weight=0.5,
    limit=5,
)
```

<Tip>
  `proximity_weight=0.0` 完全禁用近邻混合（纯语义）。`proximity_weight=1.0` 返回纯粹按图邻近度排名的结果。`0.2`–`0.4` 之间的值适用于大多数生产场景。
</Tip>


<a id="cross-graph-navigation"></a>
## 跨图导航

链接多个独立的 `ContextGraph` 实例，让智能体可以跨问题空间遍历：

```python
domain_graph    = ContextGraph()
decision_graph  = ContextGraph()

domain_graph.add_node("microservices", "architecture", properties={"style": "distributed"})
decision_graph.add_node("deploy_k8s",  "decision",     properties={"outcome": "approved"})

link_id = domain_graph.link_graph(
    other_graph=decision_graph,
    source_node_id="microservices",
    target_node_id="deploy_k8s",
    link_type="INFORMED_BY",
)

# 在遍历时跟随链接
target_graph, entry_node = domain_graph.navigate_to(link_id)

# 跨图寻路
path = domain_graph.cross_graph_path(
    source_node_id="microservices",
    target_graph=decision_graph,
    target_node_id="deploy_k8s",
    max_hops=5,
)
print("Reachable: {}, hops: {}".format(path["reachable"], path["hop_count"]))
```


<a id="agentmemory"></a>
## AgentMemory

用于对记忆存储和检索进行细粒度控制：

```python
from semantica.context import AgentMemory
from semantica.vector_store import VectorStore

memory = AgentMemory(
    vector_store=VectorStore(backend="faiss", dimension=768),
    max_memory_size=10000,
    retention_policy="90_days",   # 或 "unlimited"
)

memory_id = memory.store(
    "Critical compliance rule: all trades must be pre-approved",
    metadata={"type": "compliance"},
)

results = memory.retrieve(
    query="trade approval requirements",
    max_results=5,
    min_score=0.0,
)

memory.delete_memory(memory_id)
memory.clear_memory(conversation_id="conv_001")

history = memory.get_conversation_history(conversation_id="conv_001", max_items=100)
```

| 参数 | 类型 | 默认值 | 描述 |
| :--------- | :---- | :------- | :----------- |
| `vector_store` | `VectorStore` | **必填** | 语义检索的向量嵌入后端 |
| `max_memory_size` | `int` | `10000` | LRU 淘汰前的最大项目数 |
| `retention_policy` | `str` | `"unlimited"` | `"N_days"`（例如 `"30_days"`）或 `"unlimited"` |

<a id="markdown-round-trips"></a>
### Markdown 往返

`AgentMemory` 可以导出人类可编辑的 Markdown 并将编辑后的文件导入回来。
每个文件包含一条记忆项，YAML frontmatter 中包含必需的元数据，
Markdown 正文中包含记忆内容：

```markdown
---
id: mem_compliance_rule
created_at: '2026-07-22T09:00:00+00:00'
updated_at: '2026-07-22T10:30:00+00:00'
type: compliance
tags:
- trading
- approval
---

All trades must be pre-approved.
```

```python
from pathlib import Path

# A single selected memory can be returned as Markdown text.
document = memory.export(format="markdown", type="compliance")

# Export a memory set as one stable Markdown file per item.
memory.export(format="markdown", destination="memory_export/")

# New IDs create memories; existing IDs are updated in place.
count = memory.import_data(Path("memory_export/"), format="markdown")
```

必需的 frontmatter 字段是 `id`、`created_at`、`updated_at`，以及
`type` 或 `kind` 之一。可选元数据可以在顶层编辑。导入操作在更改记忆之前会拒绝
格式错误或重复的字段，重新导入未更改的
文件是幂等的。记忆本地的 `entities` 和 `relationships` 作为
溯源保留，但不会通过 Markdown 导入应用到 `ContextGraph`。使用专用
导出目录：匹配的文件会被覆盖，但不相关或过时的 Markdown
文件不会被自动删除。导出拒绝覆盖符号链接并
使用原子文件替换。时间戳偏移在 Markdown 中保留，
仅在比较时归一化为 UTC，因此带时区和本地朴素记录可以
安全地一起查询。向量库写入会延迟到内存导入
提交之后；适配器同步保持尽力而为并记录失败。


<a id="policyengine"></a>
## PolicyEngine

`PolicyEngine` 管理存储在知识图谱中的版本化策略。策略作为节点存储，可以链接到决策：

```python
from semantica.context import PolicyEngine
from semantica.context import ContextGraph
from semantica.context.decision_models import Policy, Decision
from datetime import datetime

graph  = ContextGraph()
policy = PolicyEngine(graph_store=graph)

# 创建并存储策略
p = Policy(
    policy_id="policy_001",
    name="Confidence Threshold Policy",
    description="All decisions must have confidence >= 0.7",
    rules={"min_confidence": 0.7, "requires_reasoning": True},
    category="decision_quality",
    version="1.0",
    created_at=datetime.now(),
    updated_at=datetime.now(),
)
policy.add_policy(p)

# 检查特定决策的合规性
decision = Decision(
    decision_id="dec_001",
    category="loan_approval",
    scenario="First-time homebuyer",
    reasoning="Good credit score and stable employment",
    outcome="approved",
    confidence=0.94,
    timestamp=datetime.now(),
    decision_maker="loan_agent",
)
compliant = policy.check_compliance(decision, "policy_001")
print("Compliant:", compliant)

# 获取某个类别的适用策略
policies = policy.get_applicable_policies(category="decision_quality")
for p in policies:
    print("{} v{}".format(p.name, p.version))
```


<a id="entitylinker"></a>
## EntityLinker

将实体文本映射到 URI，并在实体 ID 之间创建类型化链接：

```python
from semantica.context import EntityLinker

linker = EntityLinker(similarity_threshold=0.8)

# 为实体分配 URI
uri = linker.assign_uri("apple_inc", "Apple Inc.", "ORGANIZATION")
print(uri)  # "https://semantica.dev/entity/apple_inc.#organization"

# 从抽取文本中链接实体
entities = [
    {"id": "e1", "text": "Apple Inc.", "type": "ORGANIZATION"},
    {"id": "e2", "text": "Apple",      "type": "ORGANIZATION"},
]
linked = linker.link(text="Apple Inc. was founded by Steve Jobs.", entities=entities)
for e in linked:
    print("{} → {}  (confidence: {:.2f})".format(e.text, e.uri, e.confidence))

# 显式链接两个实体 ID（不是列表：接收两个 ID）
linker.link_entities(
    entity1_id="apple_inc",
    entity2_id="aapl",
    link_type="same_as",
    confidence=0.99,
)

# 构建完整实体网络
web = linker.build_entity_web()
print("Entities:", web["statistics"]["total_entities"])
print("Links:   ", web["statistics"]["total_links"])
```

<Warning>
  **`EntityLinker.link_entities()` 链接两个实体 ID，不是列表。** 调用 `link_entities(entity1_id, entity2_id, link_type)` 在两个已知 ID 之间创建类型化边。要从文本中链接抽取的实体，请使用 `link(text, entities=[...])`。
</Warning>

`link()` 返回的 `LinkedEntity` 字段：

| 字段 | 类型 | 描述 |
| :----- | :---- | :----------- |
| `entity_id` | `str` | 实体标识符 |
| `uri` | `str` | 生成的 URI（例如 `"https://semantica.dev/entity/apple_inc."`) |
| `text` | `str` | 表层形式文本 |
| `type` | `str` | 实体类型 |
| `linked_entities` | `List[EntityLink]` | 相关实体链接，包含 `source_entity_id`、`target_entity_id`、`link_type`、`confidence` |
| `context` | `Dict` | 实体元数据 |
| `confidence` | `float` | 总体置信度评分 |


<a id="contextretriever"></a>
## ContextRetriever

混合检索，结合向量相似度、图遍历和记忆：

```python
from semantica.context import ContextRetriever

retriever = ContextRetriever(
    memory_store=memory,
    knowledge_graph=context_graph,
    vector_store=vector_store,
    use_graph_expansion=True,
    max_expansion_hops=2,
    hybrid_alpha=0.5,
)

results = retriever.retrieve(
    query="What decisions were made about cloud infrastructure?",
    max_results=10,
    use_graph_expansion=True,
    min_relevance_score=0.3,
)

for r in results:
    print("[{}] score={:.3f}: {}".format(r.source, r.score, r.content[:80]))
```


<a id="data-structures"></a>
## 数据结构

<AccordionGroup>
  <Accordion title="Decision">

```python
@dataclass
class Decision:
    decision_id:          str
    category:             str
    scenario:             str
    reasoning:            str
    outcome:              str
    confidence:           float               # 0.0 - 1.0
    timestamp:            datetime
    decision_maker:       str
    reasoning_embedding:  Optional[List[float]]  # 生成的向量嵌入
    node2vec_embedding:   Optional[List[float]]  # 结构嵌入
    valid_from:           Optional[str]       # ISO 日期时间
    valid_until:          Optional[str]       # ISO 日期时间
    metadata:             Dict[str, Any]
```

  </Accordion>
  <Accordion title="Precedent">

```python
@dataclass
class Precedent:
    precedent_id:        str
    source_decision_id:  str
    similarity_score:    float               # 0-1 匹配评分
    relationship_type:   str                 # "similar_scenario" | "same_policy" | "exception_precedent"
    metadata:            Dict[str, Any]
```

  </Accordion>
  <Accordion title="Policy">

```python
@dataclass
class Policy:
    policy_id:    str
    name:         str
    description:  str
    rules:        Dict[str, Any]    # 规则定义
    category:     str
    version:      str               # 例如 "1.0"、"2.1"
    created_at:   datetime
    updated_at:   datetime
    metadata:     Dict[str, Any]
```

  </Accordion>
  <Accordion title="PolicyException">

```python
@dataclass
class PolicyException:
    exception_id:        str
    decision_id:         str
    policy_id:           str
    reason:              str
    approver:            str
    approval_timestamp:  datetime
    justification:       str
    metadata:            Dict[str, Any]
```

  </Accordion>
  <Accordion title="ApprovalChain">

```python
@dataclass
class ApprovalChain:
    approval_id:       str
    decision_id:       str
    approver:          str
    approval_method:   str          # "slack_dm" | "zoom_call" | "email" | "system"
    approval_context:  str
    timestamp:         datetime
    metadata:          Dict[str, Any]
```

  </Accordion>
  <Accordion title="LinkedEntity">

```python
@dataclass
class LinkedEntity:
    entity_id:       str
    uri:             str
    text:            str
    type:            str
    linked_entities: List[EntityLink]
    context:         Dict[str, Any]
    confidence:      float

@dataclass
class EntityLink:
    source_entity_id:  str
    target_entity_id:  str
    link_type:         str          # "same_as" | "related_to" | "part_of"
    confidence:        float
    source:            Optional[str]
    metadata:          Dict[str, Any]
```

  </Accordion>
</AccordionGroup>


<a id="real-world-patterns"></a>
## 真实场景模式

<Tabs>
  <Tab title="医疗保健：治疗决策">
    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    health_agent = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(),
        decision_tracking=True,
    )

    health_agent.store("Patient has hypertension, type 2 diabetes")
    health_agent.store("Patient allergic to penicillin: verified 2024-01")

    decision_id = health_agent.record_decision(
        category="treatment_plan",
        scenario="Hypertension with comorbid diabetes",
        reasoning="ACE inhibitors are renoprotective in diabetic patients",
        outcome="prescribed_lisinopril",
        confidence=0.91,
    )

    precedents = health_agent.find_precedents("hypertension diabetes", limit=5)
    for p in precedents:
        print("Past: {}  (confidence: {:.2f})".format(p.outcome, p.confidence))

    chain = health_agent.get_causal_chain(decision_id, direction="downstream")
    print("Follow-up decisions triggered: {}".format(len(chain)))
    ```
  </Tab>
  <Tab title="金融：贷款决策">
    ```python
    from semantica.context import AgentContext, ContextGraph, PolicyEngine
    from semantica.context.decision_models import Policy, Decision
    from semantica.vector_store import VectorStore
    from datetime import datetime

    graph  = ContextGraph()
    policy = PolicyEngine(graph_store=graph)

    # 添加合规策略
    p = Policy(
        policy_id="lending_policy",
        name="Lending Policy",
        description="Min confidence 0.8 for loan decisions",
        rules={"min_confidence": 0.8},
        category="loan_approval",
        version="1.0",
        created_at=datetime.now(),
        updated_at=datetime.now(),
    )
    policy.add_policy(p)

    loan_agent = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=graph,
        decision_tracking=True,
    )
    loan_agent.store("Applicant: credit score 750, DTI 28%, stable employment 4yr")

    # 在记录之前检查合规性
    d = Decision(
        decision_id="dec_loan_001",
        category="loan_approval",
        scenario="First-time homebuyer: 30yr fixed, 20% down",
        reasoning="Credit score above threshold, DTI within limits",
        outcome="approved_300k",
        confidence=0.94,
        timestamp=datetime.now(),
        decision_maker="loan_agent",
    )
    compliant = policy.check_compliance(d, "lending_policy")
    if compliant:
        loan_agent.record_decision(
            category=d.category,
            scenario=d.scenario,
            reasoning=d.reasoning,
            outcome=d.outcome,
            confidence=d.confidence,
        )
    ```
  </Tab>
  <Tab title="持久化与恢复">
    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(),
        decision_tracking=True,
    )

    context.store("Important fact learned during session")
    context.record_decision(
        category="ops", scenario="Scale up", reasoning="Load > 80%",
        outcome="scaled_to_10_replicas", confidence=0.97,
    )

    # 持久化所有内容
    context.save("agent_state/")

    # 稍后：恢复并继续
    restored = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(),
        decision_tracking=True,
    )
    restored.load("agent_state/")

    results = restored.retrieve("load scaling decisions", max_results=3)
    ```
  </Tab>
</Tabs>

- [向量库](vector_store.zh-CN.md) — 用于记忆检索的向量嵌入存储后端。
- [知识图谱](kg.zh-CN.md) — ContextGraph 内部使用的图算法和分析。
- [推理](reasoning.zh-CN.md) — 在上下文之上的逻辑推断层。
- [溯源](provenance.zh-CN.md) — 每个存储事实的 W3C PROV-O 血缘。

- [Context Module](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/19_Context_Module.ipynb) — 记忆与决策追踪 · 中级
- [Advanced Context Engineering](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/11_Advanced_Context_Engineering.ipynb) — 生产级 FAISS + Neo4j 配置 · 高级
