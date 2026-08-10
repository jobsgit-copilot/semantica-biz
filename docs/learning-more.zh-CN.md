---
title: "深入学习"
description: "结构化的学习路径、配置参考、故障排查和性能指南。"
icon: "graduation-cap"
---

**[English](learning-more.md)** · **简体中文（当前）**

无论你是在运行第一个流水线，还是在生产环境中部署 Semantica，本页都为你提供一条结构化的前进路径：从初学者到企业级使用。


<a id="learning-paths"></a>
## 学习路径

- **初学者（1–2 小时）** — 刚接触 Semantica 和知识图谱。[从安装开始 →](installation.zh-CN.md)
- **中级（4–6 小时）** — 已掌握基础知识，正在构建真实应用。[从模块开始 →](modules.zh-CN.md)
- **高级（8+ 小时）** — 企业部署、定制化和扩展。[从架构开始 →](architecture.zh-CN.md)

<Tabs>
  <Tab title="初学者（1–2 小时）">
    刚接触 Semantica 和知识图谱。无需具备任何图数据库经验。

    <Steps>
      <Step title="配置你的环境">
        [安装指南](installation.zh-CN.md)：虚拟环境、可选扩展依赖、平台相关修复。
      </Step>
      <Step title="理解核心概念">
        [核心概念](concepts.zh-CN.md)：知识图谱是什么、向量嵌入如何工作、抽取起什么作用。
      </Step>
      <Step title="运行你的第一个示例">
        [入门指南](getting-started.zh-CN.md)：5 分钟代码演示，使用基于模式的抽取（无需 API 密钥）。
      </Step>
      <Step title="构建你的第一个知识图谱">
        [快速上手教程](quickstart.zh-CN.md)：从摄取到可视化的完整 6 步流水线。
      </Step>
      <Step title="交互式探索">
        [Welcome to Semantica notebook](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/01_Welcome_to_Semantica.ipynb)：每个模块的 Jupyter 演示。
      </Step>
    </Steps>
  </Tab>
  <Tab title="中级（4–6 小时）">
    已掌握基础知识，正在构建真实应用。假设你已完成初学者路径。

    <Steps>
      <Step title="学习每个模块">
        [模块指南](modules.zh-CN.md)：全部 27 个模块，附带代码示例和常见流水线组合。
      </Step>
      <Step title="构建生产级知识图谱">
        [Building Knowledge Graphs notebook](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/07_Building_Knowledge_Graphs.ipynb)：多源、去重、冲突解决。
      </Step>
      <Step title="添加语义搜索">
        [Embeddings notebook](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/09_Embeddings.ipynb)：提供商、池化策略、向量库。
      </Step>
      <Step title="多源集成">
        [Multi-Source Data Integration notebook](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/06_Multi_Source_Data_Integration.ipynb) 介绍多源模式。
      </Step>
    </Steps>
  </Tab>
  <Tab title="高级（8+ 小时）">
    企业部署、定制化和扩展。假设你具备生产环境使用经验。

    <Steps>
      <Step title="理解架构">
        [架构指南](architecture.zh-CN.md)：四层设计、扩展点以及设计决策。
      </Step>
      <Step title="时态智能">
        [Temporal Graphs notebook](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/10_Temporal_Knowledge_Graphs.ipynb)：`valid_from`/`valid_until`、Allen 区间代数、时间点查询。
      </Step>
      <Step title="本体驱动的知识库">
        [Ontology notebook](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/14_Ontology.ipynb)：自动生成、SHACL 校验、Ontology Hub（v0.5.0）。
      </Step>
      <Step title="高级可视化">
        [Complete Visualization Suite notebook](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/03_Complete_Visualization_Suite.ipynb)：UMAP、t-SNE、社区布局、向量嵌入投影。
      </Step>
      <Step title="企业级导出">
        [Multi-Format Export notebook](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/05_Multi_Format_Export.ipynb)：带 PROV-O 的 RDF、Parquet、Neo4j Cypher、Arrow、OWL。
      </Step>
    </Steps>
  </Tab>
</Tabs>


<a id="configuration-reference"></a>
## 配置参考

所有设置都可通过环境变量覆盖：无需修改代码。

| 设置 | 环境变量 | 默认值 |
| :------- | :-------------------- | :------- |
| OpenAI API 密钥 | `OPENAI_API_KEY` | `None` |
| Groq API 密钥 | `GROQ_API_KEY` | `None` |
| Anthropic API 密钥 | `ANTHROPIC_API_KEY` | `None` |
| 向量嵌入提供商 | `SEMANTICA_EMBEDDING_PROVIDER` | `"openai"` |
| 图后端 | `SEMANTICA_GRAPH_BACKEND` | `"networkx"` |
| 日志级别 | `SEMANTICA_LOG_LEVEL` | `"INFO"` |
| 日志格式 | `SEMANTICA_LOG_FORMAT` | `"text"` |


<a id="troubleshooting"></a>
## 故障排查

<AccordionGroup>

<Accordion title="ModuleNotFoundError: No module named 'semantica'" icon="circle-xmark">

验证安装，并确认激活了正确的 Python 环境：

```bash
pip list | grep semantica
pip install --upgrade semantica
```

如需可选功能，请安装相应的扩展依赖：

```bash
pip install "semantica[llm-openai]"   # OpenAI 提供商
pip install "semantica[gpu]"          # GPU 加速
```

</Accordion>

<Accordion title="AuthenticationError" icon="lock">

将你的 API 密钥设置为环境变量——切勿在源文件中硬编码密钥：

```bash
export OPENAI_API_KEY="sk-..."
export GROQ_API_KEY="gsk_..."
```

</Accordion>

<Accordion title="MemoryError 或 OOM 崩溃" icon="memory">

从默认的内存型 NetworkX 后端切换到持久化的图数据库：

```python
from semantica.graph_store import FalkorDBStore
from semantica.kg import GraphBuilder

store   = FalkorDBStore(host="localhost", port=6379)
builder = GraphBuilder(merge_entities=True, graph_store=store)
```

对于大型语料，还应减小批大小并启用流式摄取。

</Accordion>

<Accordion title="在大型数据集上处理缓慢" icon="gauge">

启用并行执行和 GPU 加速：

```python
from semantica.pipeline import Pipeline

pipeline = Pipeline(workers=8, batch_size=32)
pipeline.run(sources)
```

```bash
pip install "semantica[gpu]"  # 基于 CUDA 的向量嵌入
```

</Accordion>

<Accordion title="Windows [all] 安装失败" icon="windows">

已在 **v0.5.0** 中修复。请升级：

```bash
pip install --upgrade semantica
```

或者单独安装各个扩展依赖：先 `pip install "semantica[core]"`，再根据需要添加 `[llm-openai]`、`[gpu]` 等。

</Accordion>

<Accordion title="Windows 上 cp1252 编码崩溃" icon="windows">

已在 **v0.5.0** 中修复。对于更早的版本，请设置编码环境变量：

```bash
set PYTHONIOENCODING=utf-8
```

</Accordion>

</AccordionGroup>


<a id="performance-optimization"></a>
## 性能优化

<AccordionGroup>

<Accordion title="后端选择：开发 vs. 生产" icon="server">

| 操作 | NetworkX（默认） | Neo4j / FalkorDB |
| :--------- | :------------------ | :---------------- |
| 图谱构建 | 快 | 中等 |
| 查询性能 | 中等 | 快 |
| 可扩展性 | 仅内存 | 持久化、生产规模 |
| 推荐用途 | 开发、小型图谱 | 生产、大型语料 |

在本地开发和原型设计时使用 NetworkX。在部署到生产环境之前切换到持久化后端。

</Accordion>

<Accordion title="大型语料的批处理" icon="layer-group">

分批处理文档，而不是一次处理一个。根据可用内存配置 `chunk_size`：一个不错的起点是在 16 GB 内存的机器上每批处理 1,000 个文档。

```python
from semantica.pipeline import Pipeline

pipeline = Pipeline(workers=8, batch_size=32)
pipeline.run(sources)
```

</Accordion>

<Accordion title="去重 v2：快最高 7 倍" icon="bolt">

如果去重成为瓶颈，请从 v1 策略切换到 v2 引擎：

```python
resolver = EntityResolver()
merged   = resolver.resolve(entities, strategy="semantic_v2")  # 快最高 7 倍
```

`blocking_v2`、`hybrid_v2` 和 `semantic_v2` 策略通过在相似度评分之前进行候选块划分，来减少 O(n²) 的比较次数。

</Accordion>

</AccordionGroup>


<a id="security-best-practices"></a>
## 安全最佳实践

- **API 密钥**：存储在环境变量或密钥管理器中；切勿提交到版本控制；定期轮换
- **敏感数据**：对于 PII 或涉密内容，使用本地嵌入模型（Ollama、HuggingFace）；在没有数据处理协议的情况下，避免将敏感数据发送到外部 API
- **图谱导出**：对敏感导出进行静态加密；在配置自定义 LLM 网关时使用 v0.5.0 的 SSRF 安全 `base_url` 校验
- **XML 摄取**：始终使用 `XMLIngestor`（v0.5.0），它使用对 XXE 安全的 lxml 后端；切勿用标准库解析器解析不可信的 XML

- [Cookbook](cookbook.zh-CN.md) — 从初学者到高级的交互式 Jupyter notebook。
- [常见问题](faq.zh-CN.md) — 常见问题解答。
- [API 参考](reference/core.zh-CN.md) — 完整的技术文档。
