---
title: "架构"
description: "四层模块化架构，专为组件独立使用、清晰的关注点分离和完全的可扩展性而设计。"
icon: "building"
---

**[English](architecture.md)** · **简体中文（当前）**

Semantica 围绕四层模块化架构构建。只导入你所需的内容：框架从不强制使用完整技术栈。每个组件都可独立替换，每一层都通过干净的接口通信，没有任何隐藏的耦合。


## 四层架构

<img src="/assets/img/diagrams/architecture-overview.svg" alt="Semantica 四层架构" style={{ width: '100%', borderRadius: '12px', margin: '16px 0 24px' }} />

<Tabs>

<Tab title="第 1 层：数据摄取">

将来自任意数据源的数据作为统一的 `SourceDocument` 加载到流水线中。

| 数据源 | 模块 | 说明 |
| :------ | :------ | :----- |
| PDF、DOCX、PPTX、HTML、JSON、CSV | `ingest.FileIngestor` | 支持压缩包、递归目录扫描 |
| Parquet | `ingest.ParquetIngestor` | PyArrow、Hive 风格分区（v0.5.0） |
| XML | `ingest.XMLIngestor` | 防 XXE 的 lxml、XSD/DTD 校验（v0.5.0） |
| 网页 | `ingest.WebIngestor` | 可配置抓取深度、链接过滤 |
| SQL / Snowflake / Databricks | `ingest.DBIngestor` / `ingest.SnowflakeIngestor` / `ingest.DatabricksIngestor` | 自定义 SQL、schema 内省、Unity Catalog 血缘 |
| Kafka / 流 | `ingest.StreamIngestor` | 实时数据流摄取 |
| 邮件 | `ingest.EmailIngestor` | IMAP/SMTP，支持附件抽取 |
| 代码仓库 | `ingest.RepoIngestor` | Git 仓库、代码结构 |
| MCP | `ingest.MCPIngestor` | Model Context Protocol 数据源 |

</Tab>

<Tab title="第 2 层：处理">

将原始文本转换为结构化、富信息的文档，以便写入知识库。

| 步骤 | 模块 | 作用 |
| :---- | :------ | :------------ |
| 解析 | `parse.DocumentParser` / `parse.DoclingParser` | 文本与版式抽取、表格检测 |
| 归一化 | `normalize` | 规范形式、日期/名称标准化、编码修复 |
| 抽取 | `semantic_extract` | 命名实体识别(NER)、关系抽取、事件检测、三元组 |
| 构建 | `kg.GraphBuilder` | 实体合并、边构建、图谱组装 |
| QA | `deduplication`、`conflicts` | 去重检测、冲突解决、校验 |

</Tab>

<Tab title="第 3 层：智能">

持久的知识存储与向量嵌入基础设施，为检索与推理提供动力。

| 组件 | 模块 | 说明 |
| :--------- | :------ | :----------- |
| 知识图谱 | `kg` | 图谱构建、时态模型、图分析、距离智能 |
| 向量库 | `vector_store` | pgvector、Qdrant、Weaviate、Pinecone：语义相似度检索 |
| 本体 | `ontology` | OWL/RDFS 建模、SHACL 校验、本体对齐 |
| 三元组库 | `triplet_store` | RDF 三元组存储与 SPARQL 查询 |
| 向量嵌入 | `embeddings` | Sentence-Transformers、FastEmbed、OpenAI、BGE |
| 时态 | `kg.TemporalKnowledgeGraph` | `valid_from` / `valid_until`、Allen 区间代数（v0.4.0） |

</Tab>

<Tab title="第 4 层：应用">

消费知识图谱和向量库以服务于下游用例。

| 用例 | 模块 | 说明 |
| :-------- | :------ | :----------- |
| GraphRAG | `context.AgentContext` | 面向 LLM 的图谱接地检索 |
| 智能体记忆 | `context.ContextGraph` | 跨智能体运行的持久语义记忆 |
| 决策追踪 | `context.AgentContext` | 记录、追踪并审计每一次智能体决策 |
| 本体中心 | `explorer` | 可视化编辑器、SHACL Studio、对齐界面（v0.5.0） |
| 多智能体 | `integrations.agno` | 共享上下文、团队级记忆、知识图谱工具集 |
| 可视化 | `visualization` | 交互式 HTML 图谱、向量嵌入图、时态视图 |
| 导出 | `export` | RDF、Parquet、ArangoDB AQL、OWL、CSV、Arrow |
| 推理 | `reasoning` | 前向链接、Rete、Datalog、SPARQL、溯因推理 |

</Tab>

</Tabs>


## 数据流

每条流水线都遵循同一条从原始数据源到最终交付产物的线性路径：

<img src="/assets/img/diagrams/pipeline-flow.svg" alt="Semantica 8 步流水线：摄取 → 解析 → 归一化 → 抽取 → 构建知识图谱 → QA → 存储 → 交付" style={{ width: '100%', borderRadius: '10px', margin: '16px 0 24px' }} />


## 模块地图

| 层 | 类别 | 模块 |
| :----- | :-------- | :------- |
| **第 1 层：数据摄取** | 数据源 | `ingest`、`split` |
| **第 2 层：处理** | 转换 | `parse`、`normalize`、`semantic_extract`、`deduplication`、`conflicts` |
| **第 3 层：智能** | 存储 | `kg`、`vector_store`、`graph_store`、`triplet_store`、`embeddings`、`ontology` |
| **第 4 层：应用** | 交付 | `context`、`reasoning`、`export`、`visualization`、`explorer`、`pipeline` |
|: | 横切关注点 | `provenance`、`change_management`、`llms`、`mcp_server`、`seed`、`evals`、`core`、`utils` |


## 扩展点

每一层都暴露基于注册表的扩展点。注册自定义实现后，它们将参与完整流水线，无需对核心代码做任何改动。

<CodeGroup>

```python 自定义摄取器
from semantica.ingest.registry import method_registry

def custom_file_ingestor(source):
    # 返回一个文档字典列表，包含 'text'、'metadata'、'source'
    return [{"text": "...", "metadata": {}, "source": source}]

# 在 "file" 任务类别下以唯一名称注册
method_registry.register("file", "my_custom_format", custom_file_ingestor)

available = method_registry.list_all("file")
```

```python 自定义抽取器
from semantica.semantic_extract.registry import method_registry

def custom_entity_extractor(text, config=None):
    # 返回一个实体字典列表，包含 'text'、'type'、'confidence'
    return [{"text": "...", "type": "CUSTOM_TYPE", "confidence": 0.9}]

# 在 "entity" 抽取任务下注册
method_registry.register("entity", "my_extractor", custom_entity_extractor)
```

```python 自定义插件
from semantica.core import PluginRegistry

class MyPlugin:
    def process(self, graph, config):
        # 原地修改图谱并返回
        return graph

registry = PluginRegistry()
registry.register_plugin("my_plugin", MyPlugin, version="1.0.0")
```

</CodeGroup>


## 设计决策

<AccordionGroup>

<Accordion title="模块化：按需使用" icon="puzzle-piece">

每个组件都可独立工作。`NERExtractor` 无需图存储即可运行。`VectorStore` 无需决策追踪即可运行。框架从不强制实例化完整技术栈：你只为导入的部分买单。

</Accordion>

<Accordion title="可插拔：无需修改核心即可扩展" icon="plug">

自定义的摄取器、抽取器、校验器和导出器都遵循相同的基类模式。通过 `PluginRegistry` 注册后，它们就会加入完整流水线：溯源追踪、重试策略和并行执行一应俱全——无需改动核心代码。

</Accordion>

<Accordion title="默认开启溯源" icon="link">

血缘追踪在最底层就已内建于图谱构建之中。每个节点和每条边都携带一个 `source_id`，指向来源文档、抽取方法和时间戳。无需手动开启：溯源始终启用。

</Accordion>

<Accordion title="配置优于约定" icon="sliders">

集中式的 `ConfigManager`，支持环境变量覆盖。没有魔法默认值：所有行为都是显式且可覆盖的。适用于开发、预发和生产环境需要不同后端的多环境部署。

</Accordion>

</AccordionGroup>


## 性能特征

| 特性 | 机制 |
| :-------------- | :--------- |
| **并行执行** | `Pipeline(workers=N)`，每个阶段可配置工作线程数 |
| **增量处理** | 图谱增量更新：新数据无需全量重算 |
| **流式摄取** | 处理大规模语料时无需全部载入内存 |
| **后端灵活性** | 将内存版 NetworkX 替换为 Neo4j / FalkorDB，API 无需改动 |
| **去重 v2** | `blocking_v2`、`hybrid_v2`、`semantic_v2`：比 v1 快最高 7 倍 |
| **索引检索** | Explorer 在 118k 节点上的检索仅需 0.004ms（v0.5.0） |

- [模块文档](modules.zh-CN.md) — 完整的模块文档及代码示例。
- [了解更多](learning-more.zh-CN.md) — 配置参考、性能指南与故障排查。
- [流水线参考](reference/pipeline.zh-CN.md) — 流水线编排、工作线程与重试策略。
- [核心参考](reference/core.zh-CN.md) — 框架生命周期、插件注册表与配置。
