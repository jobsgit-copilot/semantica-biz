---
title: "Agno 集成"
description: "通过五个专注的组件，将 Semantica 的语义智能栈接入 Agno 多智能体团队。"
icon: "robot"
---

**[English](agno.md)** · **简体中文（当前）**

> 五个即插即用组件，将 Semantica 的知识图谱、向量记忆和决策智能带入任何 Agno 智能体或团队。


<a id="installation"></a>
## 安装

```bash
# 核心集成
pip install "semantica[agno]"

# 带图存储后端
pip install "semantica[agno,graph-neo4j]"
pip install "semantica[agno,graph-falkordb]"

# 完整技术栈
pip install "semantica[agno,graph-neo4j,vectorstore-pgvector]"
```


<a id="components-at-a-glance"></a>
## 组件概览

- **AgnoContextStore** — `AgentMemory(db=…)`：用混合向量 + 上下文图谱记忆替换 Agno 的扁平存储。为任何智能体添加决策跟踪和先例搜索。
- **AgnoKnowledgeGraph** — `Agent(knowledge=…)`：文档流经完整的 Semantica 抽取流水线，进入一个可查询的 `ContextGraph`，并支持多跳 GraphRAG。
- **AgnoDecisionKit** — `Agent(tools=[…])`：6 个决策智能工具：记录决策、查找先例、追踪因果链、分析影响、检查策略、汇总历史。
- **AgnoKGToolkit** — `Agent(tools=[…])`：7 个知识图谱构建工具：抽取实体、抽取关系、添加到图谱、查询图谱、查找关联、推断事实、导出子图。
- **AgnoSharedContext** — 团队级：一个共享的 `ContextGraph`，所有智能体共用。每个智能体通过 `bind_agent()` 获得按角色限定的视图。写入操作会按角色打标签。


<a id="component-details"></a>
## 组件详情

<Tabs>
  <Tab title="AgnoContextStore">
    用混合**向量 + 上下文图谱**记忆存储替换 Agno 的扁平会话存储。实现 `agno.memory.db.base.MemoryDb`。

    ```python
    from agno.agent import Agent
    from agno.memory import AgentMemory
    from agno.models.openai import OpenAIChat
    from semantica.context import ContextGraph
    from semantica.vector_store import VectorStore
    from integrations.agno import AgnoContextStore

    store = AgnoContextStore(
        vector_store=VectorStore(backend="faiss"),
        knowledge_graph=ContextGraph(advanced_analytics=True),
        decision_tracking=True,
        graph_expansion=True,
        session_id="user_session_42",
    )

    agent = Agent(
        model=OpenAIChat(id="gpt-4o"),
        memory=AgentMemory(db=store),
        description="A financially aware assistant with persistent decision intelligence.",
    )
    ```

    | 方法 | 说明 |
    | :-------- | :------------- |
    | `upsert_memory()` | 将文本存入 `AgentContext`（向量索引 + 图谱节点） |
    | `read_memories()` | 混合检索：向量相似度 + 图谱跳转扩展 |
    | `record_decision()` | 记录带有推理和结果的结构化决策 |
    | `find_precedents()` | 返回语义相似的历史决策 |
  </Tab>
  <Tab title="AgnoKnowledgeGraph">
    为 Agno 智能体提供一个可查询的 `ContextGraph`，而非扁平的文档存储。摄取的文档会流经完整的 Semantica 抽取流水线。

    ```python
    from agno.agent import Agent
    from agno.models.openai import OpenAIChat
    from semantica.kg import GraphBuilder
    from semantica.semantic_extract import NERExtractor, RelationExtractor
    from integrations.agno import AgnoKnowledgeGraph

    kg = AgnoKnowledgeGraph(
        graph_builder=GraphBuilder(),
        ner_extractor=NERExtractor(),
        relation_extractor=RelationExtractor(),
    )

    kg.load("regulatory_docs/", recursive=True)
    kg.load(texts=["Basel IV capital requirements apply from January 2026."])

    agent = Agent(model=OpenAIChat(id="gpt-4o"), knowledge=kg, search_knowledge=True)
    ```

    **摄取：** `解析 → NER → 关系抽取 → 图谱构建 → 向量索引`

    **搜索：** `向量检索 → 实体查找 → 图谱跳转扩展 → 上下文注入`

    ```python
    ctx = kg.get_graph_context("Basel IV")
    # 返回该实体直接邻域的文本摘要
    ```
  </Tab>
  <Tab title="AgnoDecisionKit">
    将 Semantica 的决策智能作为原生 Agno 工具暴露出来。

    ```python
    from agno.agent import Agent
    from agno.models.openai import OpenAIChat
    from semantica.context import AgentContext
    from integrations.agno import AgnoDecisionKit

    ctx   = AgentContext(decision_tracking=True)
    agent = Agent(
        model=OpenAIChat(id="gpt-4o"),
        tools=[AgnoDecisionKit(context=ctx)],
        show_tool_calls=True,
    )
    agent.print_response("Should we approve this mortgage application?")
    ```

    | 工具 | 说明 |
    | :------ | :------------- |
    | `record_decision` | 记录带有推理、结果和置信度的决策 |
    | `find_precedents` | 搜索类似的过往决策 |
    | `trace_causal_chain` | 追踪某个决策的因果链 |
    | `analyze_impact` | 评估某个决策的下游影响 |
    | `check_policy` | 根据策略规则验证决策 |
    | `get_decision_summary` | 按类别汇总决策历史 |
  </Tab>
  <Tab title="AgnoKGToolkit">
    让智能体在推理过程中主动构建和查询上下文图谱。

    ```python
    from agno.agent import Agent
    from agno.models.openai import OpenAIChat
    from integrations.agno import AgnoKGToolkit

    agent = Agent(
        model=OpenAIChat(id="gpt-4o"),
        tools=[AgnoKGToolkit()],
        show_tool_calls=True,
    )
    ```

    | 工具 | 说明 |
    | :------ | :------------- |
    | `extract_entities` | 从文本中抽取命名实体 |
    | `extract_relations` | 抽取实体之间的关系 |
    | `add_to_graph` | 将实体/关系添加到上下文图谱 |
    | `query_graph` | 查询图谱（自然语言或 Cypher） |
    | `find_related` | 查找与给定实体相关的概念 |
    | `infer_facts` | 应用规则从图谱中推断新事实 |
    | `export_subgraph` | 将子图导出为 RDF / JSON-LD |
  </Tab>
  <Tab title="AgnoSharedContext">
    一个共享的 `ContextGraph`，供整个 Agno `Team` 共用。每个智能体通过 `bind_agent()` 获得按角色限定的视图。写入操作会按角色打标签。

    ```python
    from agno.agent import Agent
    from agno.team import Team
    from agno.models.openai import OpenAIChat
    from semantica.context import ContextGraph
    from semantica.vector_store import VectorStore
    from integrations.agno import AgnoSharedContext, AgnoDecisionKit, AgnoKGToolkit

    shared = AgnoSharedContext(
        vector_store=VectorStore(backend="faiss"),
        knowledge_graph=ContextGraph(advanced_analytics=True),
        decision_tracking=True,
    )

    research_agent = Agent(
        name="Researcher",
        model=OpenAIChat(id="gpt-4o"),
        memory=shared.bind_agent("researcher"),
        tools=[AgnoKGToolkit(context=shared)],
    )
    decision_agent = Agent(
        name="Analyst",
        model=OpenAIChat(id="gpt-4o"),
        memory=shared.bind_agent("analyst"),
        tools=[AgnoDecisionKit(context=shared)],
    )

    team = Team(
        name="Research & Decision Team",
        agents=[research_agent, decision_agent],
        mode="coordinate",
    )
    ```

    ```python
    decision_id = shared.record_decision(
        category="strategy",
        scenario="Expand to EU market",
        reasoning="Strong demand signals from Q1 survey",
        outcome="approved",
        confidence=0.87,
        agent_role="cfo",
    )
    precedents = shared.find_precedents("market expansion")
    insights   = shared.get_shared_insights()
    ```
  </Tab>
</Tabs>


<a id="api-reference"></a>
## API 参考

```python
from integrations.agno import (
    AgnoContextStore,    # MemoryDb 实现
    AgnoKnowledgeGraph,  # AgentKnowledge 实现
    AgnoDecisionKit,     # 决策智能 Toolkit
    AgnoKGToolkit,       # 知识图谱 Toolkit
    AgnoSharedContext,   # 团队级共享上下文
    AGNO_AVAILABLE,      # 布尔值：若已安装 agno 则为 True
)
```

这五个类在未安装 `agno` 的情况下也可使用：它们携带完整的 Semantica API 并能优雅降级。


<a id="see-also"></a>
## 另请参阅

- [Context 模块](../reference/context.zh-CN.md) — 支撑该集成的 AgentContext 和 ContextGraph。
- [知识图谱](../reference/kg.zh-CN.md) — AgnoKnowledgeGraph 使用的知识图谱构建。
- [LLMs](../reference/llms.zh-CN.md) — 为 Agno 智能体配置 LLM 提供商。
- [向量库](../reference/vector_store.zh-CN.md) — AgnoContextStore 的向量后端。
