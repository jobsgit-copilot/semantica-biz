---
title: "Semantica"
description: "AI 的可问责性与上下文层：上下文图谱 · 决策智能 · 完整溯源"
---

**[English](index.md)** · **简体中文（当前）**

```bash
pip install semantica
```

你的智能体刚刚做出了一个决定。现在有人需要对此做出解释。

*它在那一刻知道什么？哪些事实塑造了最终结果？这些事实来自何处？它以前是否做过同样的判断？结果又如何？*

如果你的技术栈无法用可追溯的记录来回答这些问题，你就存在一道缺口。这不是能力缺口，而是**可问责性缺口**。正因如此，AI 迟迟未能在医疗、金融、法律和政府部门实现规模化落地；也正是出于这个原因，面向这些市场构建产品的团队总是在反复从零开始重建同样的防护机制。

**Semantica 弥合了这道缺口。** 它是位于你现有智能体框架之下的上下文与可问责性层：它并不取代 LangChain 或 LlamaIndex，而是让它们的输出值得信赖。


## 每个生产级 AI 团队都会撞到的问题

强大的智能体并不自动等于值得信赖的智能体。现代 AI 系统存在五个结构性盲点，使其无法部署于受监管的环境中：

**缺少记忆结构** —— 智能体存储的是向量嵌入，而非语义
- 无法追问一条事实*为什么*被召回
- 被召回的事实无法回链到其源文档
- 上下文是一个每次运行都会重置的黑盒

**缺少决策追踪** —— 智能体持续在行动，却什么都不记录
- 没有可提交给监管机构或审计员的历史记录
- 无法重放或复现某次过往决策
- 调试意味着重新跑一遍，而非复盘评审

**缺少溯源** —— 输出无法追溯到源事实
- 在医疗、金融和法律领域：这是硬性的合规阻塞点
- 缺少从推理回溯到原始文档的血缘
- 无法证明智能体实际依赖了哪些信息

**缺少推理透明度** —— 只有黑盒答案，没有解释
- 无法验证推理路径
- 无法对某条具体结论提出异议
- 没有改进或纠正未来行为的依据

**缺少冲突检测** —— 相互矛盾的事实悄悄并存于向量库中
- 当两个来源不一致时无从察觉
- 输出随时间推移变得不一致且不可预测
- 随着知识库增长，静默失败不断累积

<Note>
  这些都不是边缘情况。它们正是企业 AI 试点停滞的原因，也是你的合规团队一直说*还没准备好*的原因。
</Note>


## Semantica 为你的技术栈带来什么

Semantica 为每个智能体提供其可问责所需的基础设施。只需几分钟，即可将它接入你现有的设置：

**上下文图谱** —— 一张结构化、可查询的图谱，涵盖你的智能体所知、所决、所推理的一切
- 在智能体多次运行之间持久化：会话之间不会丢失上下文
- 可使用 SPARQL 和完整的图算法进行查询
- 节点和边上带有 `valid_from` / `valid_until` 的时态模型
- 支持完整知识状态的按时间点快照

**决策智能** —— 每个决策都是系统中的一等对象
- `record_decision()` 捕获完整生命周期与因果链
- 对过往决策进行混合先例检索，以保证一致性
- `analyze_decision_impact()` 展示下游影响
- 从触发到结果的因果链可视化

**完整溯源** —— 每个事实都链接到其源文档与数据摄取事件
- 跨所有模块的 W3C PROV-O 合规血缘
- 从原始输入到最终推理的完整可追溯性
- `recorded_at` 时间戳标记，支持 OWL-Time 导出
- 满足 HIPAA、SOX、GDPR、FDA 21 CFR Part 11 的审计就绪要求

**推理引擎** —— 可解释的推理路径，而非黑盒
- 前向链接、Rete、演绎、归纳
- 基于 SPARQL 查询对 RDF 图谱进行推理
- 使用递归 Horn 子句规则的 Datalog
- 每个结论都有可追溯的推导路径作为支撑

**时态智能** —— 你的图谱不仅知道*是什么*，还知道*何时*
- Allen 区间代数：覆盖全部 13 种时态关系
- 对历史图谱状态进行按时间点查询
- 每个事实都带有时态溯源时间戳
- 支持 OWL-Time 导出，满足标准化归档要求

**本体中心** —— 在浏览器中完成本体全生命周期
- 用于 schema 设计与编辑的可视化编辑器
- 用于约束编写与校验的 SHACL Studio
- 跨多个本体的对齐编写
- 内置健康度仪表盘与版本控制

<Tip>
  可与任何 LLM 供应商以及任何智能体框架协同工作：将其加入现有技术栈，无需改动你的架构。
</Tip>

<img src="/assets/img/diagrams/architecture-overview.svg" alt="Semantica 四层架构：数据摄取 → 处理 → 智能 → 应用" style={{ width: '100%', borderRadius: '12px', margin: '24px 0' }} />


## 亲眼看它运行

一次 pip 安装。几行代码连接你的智能体。其他一切都变得可追溯。

```bash
pip install semantica
```

<CodeGroup>

```python OpenAI
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import OpenAI

context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=1536),
    knowledge_graph=ContextGraph(advanced_analytics=True),
    decision_tracking=True,
    llm=OpenAI(model="gpt-4o"),
)

context.store("GPT-4 outperforms GPT-3.5 on reasoning benchmarks by 40%")

decision_id = context.record_decision(
    category="model_selection",
    scenario="Choose LLM for production reasoning pipeline",
    reasoning="GPT-4 benchmark advantage justifies 3x cost increase",
    outcome="selected_gpt4",
    confidence=0.91,
)

precedents = context.find_precedents("model selection reasoning", limit=5)
influence  = context.analyze_decision_influence(decision_id)
```

```python Anthropic
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import LiteLLM
import os

context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=1024),
    knowledge_graph=ContextGraph(advanced_analytics=True),
    decision_tracking=True,
    llm=LiteLLM(model="anthropic/claude-opus-4-7", api_key=os.getenv("ANTHROPIC_API_KEY")),
)

context.store("Claude excels at long-context reasoning and code generation")

decision_id = context.record_decision(
    category="model_selection",
    scenario="Choose LLM for document analysis pipeline",
    reasoning="Claude's 200k context window eliminates chunking overhead",
    outcome="selected_claude",
    confidence=0.94,
)

precedents = context.find_precedents("document analysis model", limit=5)
```

```python Ollama（本地）
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.llms import LiteLLM

context = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=ContextGraph(advanced_analytics=True),
    decision_tracking=True,
    llm=LiteLLM(model="ollama/llama3.2", base_url="http://localhost:11434"),
)

# 完全本地化：数据不会离开你的基础设施
context.store("Local LLMs enable air-gapped compliance deployments")

decision_id = context.record_decision(
    category="deployment_model",
    scenario="Choose inference strategy for on-prem environment",
    reasoning="Air-gap requirement eliminates cloud API options",
    outcome="local_inference",
    confidence=0.99,
)
```

</CodeGroup>

- [完整快速上手](quickstart.zh-CN.md) —— 分步流水线演练
- [Cookbook](cookbook.zh-CN.md) —— 40+ 真实场景的 Jupyter notebook
- [加入 Discord](https://discord.gg/sV34vps5hH) —— 社区交流与支持


## 为后果攸关的领域而构建

Semantica 专为这类领域而设计：每一个决策都必须可解释，每一个事实都必须可追溯：

**医疗与生命科学**
- 提供完整审计追踪的临床决策支持
- 药物相互作用与禁忌图谱
- 患者安全事件追踪与根因分析
- 开箱即用的 HIPAA 合规溯源链

**金融与风险**
- 欺诈检测知识图谱
- 经得起审计考验的风险评估追踪
- SOX、GDPR 与 MiFID II 合规基础设施
- 用于监管报告的模型决策血缘

**法律与合规**
- 证据支撑的研究，每条引用事实都有溯源链接
- 带有可追溯条款抽取的合同分析
- 跨司法辖区的法规变更追踪
- 可直接用于法庭可采信文档的完整推理路径

**网络安全**
- 关联行为者、TTP 与指标的威胁归因图谱
- 带有完整事件溯源的事件响应时间线
- 覆盖完整杀伤链的安全审计追踪
- 与 MITRE ATT&CK 对齐的知识图谱集成

**政府与国防**
- 从需求简报到最终结果的策略决策追踪
- 带有溯源链的涉密信息处理
- 面向情报报告的保管链审查
- 支持本地 LLM 的气隙部署

**关键基础设施**
- 基于时态智能的电网状态追踪
- 交通安全事件图谱
- 带有决策审计追踪的应急响应协同
- 面向高风险运营决策的后果建模


## 从这里开始

<Steps>
  <Step title="安装 Semantica">
    ```bash
    pip install semantica
    ```
    参阅[安装指南](installation.zh-CN.md)了解可选 extras（`[all]`、`[neo4j]`、`[pinecone]`）以及环境配置。
  </Step>
  <Step title="运行快速上手">
    用 [5 分钟](quickstart.zh-CN.md)构建一条完整的知识图谱流水线：
    - 从任意来源摄取文档
    - 抽取实体与关系
    - 构建并查询图谱
    - 记录并追踪一个决策
  </Step>
  <Step title="理解心智模型">
    [核心概念](concepts.zh-CN.md)涵盖：
    - 知识图谱 vs. 向量库：何时使用哪种
    - GraphRAG 是什么，以及 Semantica 如何实现它
    - 溯源与决策追踪如何协同工作
    - 可问责性层的架构
  </Step>
  <Step title="深入任意模块">
    每个模块都有一份专属的[参考页](reference/context.zh-CN.md)，包含：
    - 完整的类与方法文档
    - 带类型与默认值的参数表
    - 针对每个特性的可运行代码示例
  </Step>
</Steps>

- [安装](installation.zh-CN.md) —— 不到一分钟装好 Semantica
- [快速上手](quickstart.zh-CN.md) —— 5 分钟构建一条完整的知识图谱流水线
- [核心概念](concepts.zh-CN.md) —— API 背后的心智模型
- [API 参考](reference/context.zh-CN.md) —— 精确的模块、类与方法细节
- [Cookbook](cookbook.zh-CN.md) —— 面向真实场景的领域 notebook
- [更新日志](https://github.com/semantica-agi/semantica/releases) —— 版本发布历史


## 完整能力

<AccordionGroup>

<Accordion title="上下文与决策智能" icon="brain">

### 上下文图谱

- 实体、关系与决策的结构化、持久化图谱
- 每个节点和边都带有 `valid_from` / `valid_until` 的时态模型
- 跨历史图谱状态的按时间点查询
- 距离智能：语义邻域与 N×N 距离矩阵

### 决策追踪

- `record_decision()` 配合完整生命周期管理与因果链
- 对过往决策进行混合相似度检索，以强制保证一致性
- `analyze_decision_impact()` 与 `analyze_decision_influence()` 用于后果建模
- Ego 模式探索，用于有针对性的邻域调查

</Accordion>

<Accordion title="知识工程" icon="diagram-project">

### 实体与关系抽取

- 命名实体识别(NER)：基于模式、ML 或 LLM 方法
- 通过 LLM 或基于规则的流水线进行类型化三元组抽取
- 带有时态与因果链接的事件抽取

### 本体与 Schema

- 本体中心：可视化编辑器、SHACL Studio、对齐、健康度仪表盘
- 去重 v2：`blocking_v2`、`hybrid_v2`、`semantic_v2`：速度提升最高达 7 倍
- Datalog 推理：带不动点语义的递归 Horn 子句规则
- SPARQL 推理：基于查询对 RDF 图谱进行推理

</Accordion>

<Accordion title="溯源与可审计性" icon="shield-check">

### 血缘追踪

- 跨所有模块的 W3C PROV-O 血缘：每个事实都有其来源
- `recorded_at` 时间戳标记，支持完整的 OWL-Time 导出
- 基于 SHA-256 校验和与版本控制的变更管理
- 从数据摄取事件到最终推理的完整审计追踪

### 合规基础设施

- HIPAA：面向患者数据、审计就绪的溯源链处理
- SOX / MiFID II：具备完整可追溯性的金融决策记录
- GDPR：用于主体访问与被遗忘权工作流的数据血缘
- FDA 21 CFR Part 11：电子记录与电子签名合规

</Accordion>

<Accordion title="数据摄取与导出" icon="database">

### 摄取格式

- 文档：PDF、DOCX、HTML、PPTX、Docling 版面分析
- 结构化数据：JSON、CSV、Excel、Parquet、XML
- 数据源：网络爬取、SQL、Snowflake、feeds、邮件、代码仓库、MCP

### 向量库

- FAISS、Pinecone、Weaviate、Qdrant、Milvus、PgVector、内存存储

### 图存储

- Neo4j、FalkorDB、Apache AGE、Amazon Neptune

### 导出格式

- RDF：Turtle、JSON-LD、N-Triples、RDF/XML
- 表格：Parquet、CSV、Arrow
- 图：GraphML、GEXF、DOT、ArangoDB AQL
- 本体：OWL、SKOS、SHACL

</Accordion>

</AccordionGroup>


## 模块参考

| 模块 | 提供的能力 |
| :-------- | :----------------- |
| `semantica.context` | 上下文图谱、智能体记忆、决策追踪、因果分析、先例检索 |
| `semantica.kg` | 知识图谱构建、图算法、时态模型、Allen 区间代数 |
| `semantica.semantic_extract` | NER、关系抽取、事件抽取、三元组生成 |
| `semantica.reasoning` | 前向链接、Rete、演绎、归纳、SPARQL、Datalog |
| `semantica.ontology` | SHACL、SKOS、对齐、差异/迁移、自动生成、OWL/RDF |
| `semantica.explorer` | FastAPI Knowledge Explorer(知识探索器)、本体中心、距离智能、SHACL Studio |
| `semantica.mcp_server` | MCP stdio 服务器：面向 Claude Desktop、VS Code、Cursor、Windsurf、Cline 的 12 个工具 |
| `semantica.vector_store` | FAISS、Pinecone、Weaviate、Qdrant、Milvus、PgVector |
| `semantica.graph_store` | Neo4j、FalkorDB、Apache AGE、Amazon Neptune |
| `semantica.triplet_store` | 支持 SPARQL 的内存与持久化 RDF 三元组库 |
| `semantica.ingest` | 文件、网络、feeds、数据库、Snowflake、Parquet、XML、MCP |
| `semantica.parse` | 文档解析：PDF、DOCX、HTML、PPTX、Docling 版面分析 |
| `semantica.split` | 文本分块：句子、段落、token、语义边界等策略 |
| `semantica.normalize` | 文本归一化、实体规范化、空白与编码清理 |
| `semantica.embeddings` | Sentence-Transformers、FastEmbed、OpenAI、BGE、Ollama 本地向量嵌入 |
| `semantica.pipeline` | 流水线 DSL、并行 worker、重试策略、失败处理 |
| `semantica.export` | RDF、Parquet、ArangoDB AQL、CSV、OWL、Arrow、GraphML、GEXF、DOT |
| `semantica.visualization` | 编程式图渲染：力导向、层级、环形、弹簧等布局 |
| `semantica.deduplication` | 实体去重 v1/v2、相似度评分、blocking、合并 |
| `semantica.conflicts` | 跨重叠知识源的冲突检测与冲突解决 |
| `semantica.provenance` | W3C PROV-O 血缘追踪、来源归因、审计追踪 |
| `semantica.change_management` | 基于 SHA-256 校验和的版本控制、差异、回滚 |
| `semantica.llms` | Groq、OpenAI、Anthropic、Gemini、Ollama、DeepSeek、Novita AI、LiteLLM、HuggingFace |
| `semantica.seed` | 从 CSV、JSON、SQL、API 与 RDF 源进行基础图谱种子填充 |
| `semantica.evals` | 评测套件：知识图谱质量、抽取 F1、流水线基准测试、回归追踪 |
| `semantica.core` | 编排、ConfigManager、LifecycleManager、PluginRegistry、MethodRegistry |
| `semantica.utils` | 日志、校验、进度追踪、哈希工具、嵌套字典辅助函数 |


## 为什么选择 Semantica？

**开源、MIT 协议** —— 无厂商锁定。没有付费墙功能。
- 完整源代码在 GitHub 上公开
- 你的安全团队可审计每一行代码
- 可自由 fork、扩展与自托管，无任何限制
- 无遥测、无使用情况上报

**生产就绪** —— 为承受不起意外的团队而构建。
- 1,000+ 通过的测试，具备完整的回归覆盖
- `PipelineValidator` 在启动时即捕获配置错误
- `FailureHandler` 支持指数退避与死信队列
- v0.5.0 修复了 12 项安全漏洞

**设计上的模块化** —— 只导入你需要的内容。
- 可在没有图存储的情况下使用 `NERExtractor`
- 可在没有向量存储的情况下使用 `ContextGraph`
- 每个组件都可独立替换与测试
- 无框架锁定：与任意智能体技术栈协同工作
