---
title: "入门"
description: "AI 的上下文与智能层：将原始数据转化为可解释、可审计的知识图谱。"
icon: "rocket"
---

**[English](getting-started.md)** · **简体中文（当前）**

<Tip>
  已经安装好了？直接前往[快速上手](quickstart.zh-CN.md)。需要先了解安装？查看[安装指南](installation.zh-CN.md)。
</Tip>

## 你能构建什么

- **GraphRAG 系统** — 将 LLM 的回答锚定在可追溯、结构化的知识上。每一条声明都可回溯到某个源节点。
- **可问责的 AI 智能体** — 拥有结构化决策历史、因果链与先例检索的智能体。每一次选择都被记录且可审计。
- **生产级知识图谱** — 从多源数据构建、校验并维护企业级的语义知识库。
- **合规就绪的 AI** — 每一个事实都带有 W3C PROV-O 溯源。内置 HIPAA、SOX、GDPR、FDA 21 CFR Part 11 基础设施。


## 3 步完成设置

<Steps>
  <Step title="安装 Semantica">
    <CodeGroup>

    ```bash pip（推荐）
    pip install semantica
    ```

    ```bash 安装全部附加组件
    pip install semantica[all]
    ```

    ```bash 从源码安装
    git clone https://github.com/semantica-agi/semantica.git
    cd semantica
    pip install -e ".[dev]"
    ```

    </CodeGroup>

    <Check>
      验证安装：
      ```python
      import semantica
      print(semantica.__version__)  # 0.6.0
      ```
    </Check>
  </Step>

  <Step title="选择你的路径">
    选择与你构建目标相匹配的路径：每条路径都从一个聚焦的 5 分钟示例开始。

    | 路径 | 你希望…… | 起步入口 |
    | :----- | :-------------- | :--------- |
    | **知识图谱** | 将文档转化为结构化、可查询的图谱 | [快速上手 → 第 1 步](quickstart.zh-CN.md) |
    | **智能体上下文** | 为你的 AI 智能体提供持久记忆与决策追踪 | [Context 参考](reference/context.zh-CN.md) |
    | **GraphRAG** | 将 LLM 的回答锚定在结构化知识上 | [核心概念 → GraphRAG](concepts.zh-CN.md#graphrag) |
    | **MCP 集成** | 在 Claude Desktop 或 VS Code 中使用 Semantica | [MCP Server](reference/mcp_server.zh-CN.md) |

  </Step>

  <Step title="运行流水线">
    完整的 6 步流水线：摄取、解析、抽取、构建、可视化、导出，详见[快速上手](quickstart.zh-CN.md)。使用基于模式的抽取（无需 API key）不到 5 分钟即可完成。

    <Note>
      快速上手对 LLM API key 是**可选的**。基于模式的抽取开箱即用：当你准备好时再升级到 LLM 抽取以获得更高精度。
    </Note>
  </Step>
</Steps>


## 选择你的路径

<Tabs>
  <Tab title="知识图谱">
    从任意文档或数据源构建一个结构化知识图谱。

    ```python
    from semantica.ingest import FileIngestor
    from semantica.parse import DocumentParser
    from semantica.semantic_extract import NERExtractor, RelationExtractor
    from semantica.kg import GraphBuilder

    # 1. 摄取
    sources = FileIngestor().ingest("data/report.pdf")

    # 2. 解析
    parsed = DocumentParser().parse(sources[0])

    # 3. 抽取
    ner           = NERExtractor(method="pattern")  # 无需 API key
    entities      = ner.extract(parsed)
    relationships = RelationExtractor().extract(parsed, entities=entities)

    # 4. 构建
    graph = GraphBuilder(merge_entities=True).build(
        {"entities": entities, "relationships": relationships}
    )
    print(f"{len(graph['entities'])} nodes, {len(graph['relationships'])} edges")
    ```

    **下一步：**[完整流水线实操 →](quickstart.zh-CN.md)
  </Tab>

  <Tab title="智能体上下文">
    为你的智能体提供持久记忆、决策追踪与先例检索。

    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(advanced_analytics=True),
        decision_tracking=True,
    )

    # 存储一个带有溯源的事实
    context.store("GPT-4 outperforms GPT-3.5 on reasoning by 40%")

    # 记录一次带有完整因果链的决策
    decision_id = context.record_decision(
        category="model_selection",
        scenario="Choose LLM for production pipeline",
        reasoning="GPT-4 benchmark advantage justifies cost",
        outcome="selected_gpt4",
        confidence=0.91,
    )

    # 在做新决策前检索过往决策
    precedents = context.find_precedents("model selection", limit=5)
    ```

    **下一步：**[Context 模块参考 →](reference/context.zh-CN.md)
  </Tab>

  <Tab title="GraphRAG">
    将每一条 LLM 回答都锚定在你的知识图谱上：不再有无根的断言。

    ```python
    from semantica.context import AgentContext, ContextGraph
    from semantica.vector_store import VectorStore

    context = AgentContext(
        vector_store=VectorStore(backend="faiss", dimension=768),
        knowledge_graph=ContextGraph(advanced_analytics=True),
    )

    # 加载你的知识图谱
    context.load_graph("company_kg.json")

    # 多跳 GraphRAG 查询
    result = context.query(
        "What companies were founded by people who worked at Apple?",
        mode="graphrag",
        reasoning=True,
    )

    # 每个声明都可回溯到源节点
    for claim in result.claims:
        print(f"{claim.text}  →  source: {claim.source_node}")
    ```

    **下一步：**[GraphRAG 核心概念 →](concepts.zh-CN.md#graphrag)
  </Tab>

  <Tab title="MCP 集成">
    在 Claude Desktop、VS Code、Cursor 或任意 MCP 客户端中使用 Semantica：设置完成后无需编写 Python 代码。

    ```bash
    pip install semantica
    ```

    添加到你的 MCP 客户端配置中：

    ```json
    {
      "mcpServers": {
        "semantica": {
          "command": "semantica-mcp"
        }
      }
    }
    ```

    立即可用的 12 个工具：抽取实体、查询图谱、记录决策、执行推理、导出结果。

    **下一步：**[MCP Server 参考 →](reference/mcp_server.zh-CN.md)
  </Tab>
</Tabs>


## 核心架构

Semantica 采用模块化、分层的架构：只导入你需要的部分。

- **[输入层](reference/ingest.zh-CN.md)** — 从任意来源加载并准备数据。模块：`ingest`、`parse`、`split`、`normalize`
- **[语义层](reference/semantic_extract.zh-CN.md)** — 从原始文本中抽取含义。模块：`semantic_extract`、`kg`、`ontology`、`reasoning`
- **[存储层](reference/vector_store.zh-CN.md)** — 持久化知识以供检索。模块：`embeddings`、`vector_store`、`graph_store`、`triplet_store`
- **[质量层](reference/deduplication.zh-CN.md)** — 校验并去重。模块：`deduplication`、`conflicts`
- **[上下文层](reference/context.zh-CN.md)** — 追踪决策与血缘。模块：`context`、`provenance`、`change_management`
- **[输出层](reference/export.zh-CN.md)** — 将结果交付到下游。模块：`export`、`visualization`、`pipeline`、`explorer`


## 我需要哪个模块？

请参阅[选择合适的模块](choose-your-module.zh-CN.md)指南——它将 35+ 项开发者目标映射到全部 27 个模块中合适的起步点，并为最常见的路径提供可运行的代码。


## 后续步骤

- [核心概念](concepts.zh-CN.md) — 深入讲解知识图谱、本体与推理。
- [快速上手教程](quickstart.zh-CN.md) — 包含可运行代码的完整 6 步流水线实操。
- [模块参考](modules.zh-CN.md) — 每个模块、类与常见链路的详解。
- [API 参考](reference/context.zh-CN.md) — 每个类与方法的完整模块文档。


## 帮助

- [Discord](https://discord.gg/sV34vps5hH) — 提问、分享项目、获取社区支持。
- [GitHub Issues](https://github.com/semantica-agi/semantica/issues) — 报告 bug 或申请功能。
- [常见问题](faq.zh-CN.md) — 常见问题的解答。
