---
title: "常见问题"
description: "关于 Semantica 的常见问题：安装、功能、集成与故障排查。"
icon: "circle-question"
---

**[English](faq.md)** · **简体中文（当前）**

<Info>
  使用 **Ctrl+F** / **Cmd+F** 在本页搜索。常用跳转：[安装](#安装) · [数据与功能](#数据与功能) · [故障排查](#故障排查)
</Info>

## 快速解答

| 问题 | 回答 |
| :-------- | :------ |
| 许可证？ | MIT：永久免费，无任何付费功能 |
| Python 版本？ | 3.8+（推荐 3.11+） |
| 需要 API key？ | 可选：模式抽取无需任何 key |
| 兼容 LangChain / LlamaIndex？ | 兼容：Semantica 是构建在它们之上的层，而非替代品 |
| 可用于生产？ | 可以：1,000+ 测试，v0.5.0 附带 12 项安全修复 |
| 最新版本？ | **v0.6.0**（2026 年 7 月） |
| 本地 LLM？ | 支持：通过 LiteLLM 接入 Ollama，或使用 HuggingFaceLLM 实现气隙环境 |


## 总体

<AccordionGroup>

<Accordion title="Semantica 是什么？" icon="info-circle">

Semantica 是一个开源框架，用于为 AI 构建上下文图谱与决策智能层。它将非结构化数据（文档、API、数据库）转换为结构化的知识图谱，并提供完整的溯源追踪，从而让 AI 系统具备可解释性和可审计性。

它并非 LangChain 或 LlamaIndex 的替代品。它是架设在它们之上的**可问责性层**：记录决策、将事实追溯到来源，并让推理过程透明可见。

</Accordion>

<Accordion title="使用 Semantica 可以构建什么？" icon="hammer">

- 基于文档和多源数据的知识图谱
- 具备图接地检索与来源归属的 GraphRAG 系统
- 拥有结构化决策历史和语义记忆的 AI 智能体
- 符合 W3C PROV-O 血缘要求的合规流水线（HIPAA、SOX、GDPR、FDA 21 CFR Part 11）
- 追踪事实随时间变化的时态图
- 由本体驱动、支持 SHACL 校验的知识库

</Accordion>

<Accordion title="Semantica 与 LangChain 或 LlamaIndex 有何不同？" icon="scale-balanced">

多数框架止步于检索或生成。Semantica 增加了一个**可问责性层**：每一个决策都被记录，每一个事实都链接到来源，每一个推理步骤都可解释。它面向那些需要审计 AI *为什么* 得出某个结论（而不仅仅是它说了什么）的场景而设计。

Semantica 与这些框架协同工作，而非与之对抗。

</Accordion>

<Accordion title="Semantica 是免费的吗？" icon="tag">

是的：采用 MIT 许可证，没有厂商锁定，没有付费功能。部分能力需要第三方 API key（例如 OpenAI 向量嵌入、Groq 推理），但 Semantica 本身始终免费且开源。

</Accordion>

<Accordion title="最新版本是什么？" icon="star">

**v0.5.0**：2026 年 5 月发布。

亮点：Ontology Hub、Distance Intelligence、Parquet/XML 摄取、12 项安全修复、Graph Explorer 重新设计、NER 网关修复。

```bash
pip install --upgrade semantica
```

</Accordion>

</AccordionGroup>


## 安装

<AccordionGroup>

<Accordion title="如何安装 Semantica？" icon="download">

```bash
pip install semantica
```

请参阅[安装](installation.zh-CN.md)了解虚拟环境配置、可选 extras（`[gpu]`、`[all]`、特定于提供商的 extras）以及针对各平台的故障排查。

</Accordion>

<Accordion title="需要什么 Python 版本？" icon="python">

Python **3.8 或更高版本**。推荐使用 Python 3.11+，以获得最佳性能和兼容性。

</Accordion>

<Accordion title="在 Windows 上 [all] extra 安装失败" icon="windows">

这是一个已知 bug：已在 **v0.5.0** 中修复。请升级：

```bash
pip install --upgrade semantica
```

如果您仍在使用旧版本，请单独安装各项 extras：`pip install "semantica[core]"`，然后再添加 `[llm-openai]`、`[gpu]` 等。

</Accordion>

<Accordion title="系统要求是什么？" icon="server">

| 要求 | 最低 | 推荐 |
| :----------- | :------- | :----------- |
| Python | 3.8 | 3.11+ |
| 内存 | 4 GB | 16 GB+ |
| 存储 | 2 GB | 20 GB+ |
| GPU | 可选 | 用于向量嵌入和 ML 模型的 CUDA |

</Accordion>

</AccordionGroup>


## 数据与功能

<AccordionGroup>

<Accordion title="Semantica 支持哪些数据源？" icon="database">

| 类别 | 来源 |
| :-------- | :------- |
| **文件** | PDF、DOCX、HTML、JSON、CSV、Excel、PPTX、Parquet（v0.5.0）、XML（v0.5.0）、归档文件 |
| **Web** | `WebIngestor` 爬取、RSS 订阅、站点地图 |
| **数据库** | 通过 `DBIngestor` / `SnowflakeIngestor` / `DatabricksIngestor` 接入 PostgreSQL、MySQL、Snowflake、Databricks |
| **NoSQL** | 通过 `MongoIngestor` 接入 MongoDB，通过 `DuckDBIngestor` 接入 DuckDB |
| **流** | 通过 `StreamIngestor` 接入 Kafka、实时摄取 |
| **协议** | 通过 `MCPIngestor` 接入 MCP（Model Context Protocol） |
| **云** | 通过 `GDriveIngestor` 接入 Google Drive，HuggingFace 数据集 |

</Accordion>

<Accordion title="可以使用我自己的模型吗？" icon="robot">

可以。Semantica 支持：

- **自定义 NER 和抽取模型**：通过 `method_registry` 注册
- **自定义向量嵌入模型**：任何具有 `.encode()` 接口的模型
- **自定义 LLM 提供商**：通过 LiteLLM（100+ 模型）或直接对接提供商
- **自定义流水线处理器**：通过 `PluginRegistry` 注册

</Accordion>

<Accordion title="Semantica 支持 GPU 吗？" icon="bolt">

支持。当 GPU 可用时，会自动用于向量嵌入生成、ML 模型推理和向量运算。安装 GPU 支持：

```bash
pip install "semantica[gpu]"
```

其中包含带 CUDA 的 PyTorch、FAISS GPU 和 CuPy。

</Accordion>

<Accordion title="Semantica 如何处理大型数据集？" icon="layer-group">

- **批处理**：以可配置的分块处理文档，以控制内存占用
- **并行处理**：`Pipeline(workers=N)` 并发执行各抽取步骤
- **增量处理**：在新数据到达时增量更新图谱，无需全量重算
- **持久化后端**：将内存中的 NetworkX 替换为 Neo4j、FalkorDB 或 Apache AGE，用于大规模生产图谱

</Accordion>

<Accordion title="什么是 Temporal Intelligence？" icon="clock">

`TemporalKnowledgeGraph` 为节点和边附加 `valid_from` / `valid_until` 时间窗口，支持时间点查询和历史分析。支持全部 13 种 Allen 区间代数关系以及 OWL-Time 导出。

```python
from semantica.kg import TemporalKnowledgeGraph

tkg = TemporalKnowledgeGraph()
tkg.add_temporal_triple("A", "caused", "B", valid_from="2024-01", valid_until="2024-06")
snapshot = tkg.query_at_time("2024-03")
```

自 v0.4.0 起可用。

</Accordion>

<Accordion title="什么是 Ontology Hub？" icon="sitemap">

一个覆盖完整本体生命周期的可视化浏览器 UI：通过 `semantica.explorer` 启动。包含：

- **可视化编辑器**：创建并编辑类、属性和关系
- **SHACL Studio**：编写、校验并导出 SHACL 形状
- **对齐编写**：跨本体映射概念
- **健康仪表盘**：覆盖率、一致性和约束违规指标
- **版本控制**：本体变更的差异比对与历史

自 v0.5.0 起可用。

</Accordion>

<Accordion title="什么是 Distance Intelligence？" icon="compass">

针对图中任意实体的语义邻域探索。返回带距离带分类的结构化邻近数据。

- 跨一组实体的 N×N 距离矩阵
- 以单个节点为中心的 ego 模式可视化
- 距离带：基于向量嵌入阈值的 `near` / `mid` / `far`
- 针对重复查询的向量嵌入缓存优化

自 v0.5.0 起可用。

</Accordion>

<Accordion title="我的 NER 抽取器在自定义网关上会静默回退到 pattern 模式" icon="triangle-exclamation">

已在 **v0.5.0** 中修复。对于不兼容的网关，现在会有条件地省略 `response_format=json_object` 参数，并自动应用普通的 `generate()` 加 JSON 解析作为回退。升级即可修复：

```bash
pip install --upgrade semantica
```

</Accordion>

</AccordionGroup>


## 技术

<AccordionGroup>

<Accordion title="支持哪些图数据库？" icon="diagram-project">

- **Neo4j**：行业标准，Cypher 查询语言
- **FalkorDB**：Redis 协议，超低延迟
- **Apache AGE**：PostgreSQL 扩展，OpenCypher
- **Amazon Neptune**：AWS 托管，支持 SPARQL 和 Gremlin
- **NetworkX**：内存型，用于开发和小规模图

</Accordion>

<Accordion title="支持哪些导出格式？" icon="file-export">

RDF（Turtle、JSON-LD、N-Triples、XML）、Apache Parquet、ArangoDB AQL、Apache Arrow、LPG、CSV、YAML、OWL 本体以及距离矩阵。

</Accordion>

<Accordion title="支持哪些向量库？" icon="server">

FAISS、Pinecone、Weaviate、Qdrant、Milvus、PgVector 以及内存存储。所有后端共享同一套 `VectorStore` API：只需修改一行即可切换。

</Accordion>

<Accordion title="支持哪些 LLM 提供商？" icon="microchip">

Groq、OpenAI、Anthropic、Google Gemini、Ollama（完全本地）、DeepSeek、Novita AI、LiteLLM（通过单一接口接入 100+ 模型），以及任何兼容 OpenAI 的网关。

</Accordion>

<Accordion title="Semantica 可用于生产环境吗？" icon="shield-check">

可以。v0.5.0 附带：

- 覆盖 Python 3.8–3.12 的 1,000+ 通过测试
- `PipelineValidator` 和 `FailureHandler`，支持指数退避和可配置的重试策略
- 跨所有模块的 W3C PROV-O 溯源追踪
- 基于 SHA-256 校验和的变更管理与完整审计追踪
- 12 项安全漏洞修复：涵盖 eval 注入、pickle 反序列化、SQL 注入、XXE、SSRF、ReDoS、路径穿越等

</Accordion>

</AccordionGroup>


## 故障排查

<AccordionGroup>

<Accordion title="ModuleNotFoundError: No module named 'semantica'" icon="xmark-circle">

请确认已激活正确的 Python 环境：

```bash
pip list | grep semantica
pip install --upgrade semantica
```

</Accordion>

<Accordion title="安装时出现依赖错误" icon="xmark-circle">

```bash
pip install --upgrade pip wheel
pip install semantica
```

如果 `[all]` 在 Windows 上失败，请改为单独安装各项 extras。

</Accordion>

<Accordion title="处理过程中出现内存错误" icon="memory">

请减小批大小、启用流式摄取，或切换到持久化图后端：

```python
from semantica.graph_store import FalkorDBStore
store   = FalkorDBStore(host="localhost", port=6379)
builder = GraphBuilder(merge_entities=True, graph_store=store)
```

</Accordion>

<Accordion title="向量嵌入或推理速度缓慢" icon="gauge-high">

请安装 GPU 支持并确认 CUDA 可用：

```bash
pip install "semantica[gpu]"
nvidia-smi  # 确认 GPU 可见
```

</Accordion>

<Accordion title="Windows 上出现 Unicode / cp1252 崩溃" icon="windows">

已在 **v0.5.0** 中修复。请升级，或针对旧版本设置编码环境变量：

```bash
pip install --upgrade semantica
# 对于旧版本：
set PYTHONIOENCODING=utf-8
```

</Accordion>

</AccordionGroup>


## 支持

- [Discord](https://discord.gg/sV34vps5hH) — 社区聊天与实时支持。
- [GitHub Issues](https://github.com/semantica-agi/semantica/issues) — Bug 报告与功能请求。
- [Contributing](contributing-guide.zh-CN.md) — 帮助改进 Semantica。
