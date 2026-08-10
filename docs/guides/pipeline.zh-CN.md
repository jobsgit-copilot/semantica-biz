---
title: "流水线构建器"
description: "使用 PipelineBuilder DSL 构建端到端数据处理工作流 —— 在一条声明式流水线中完成摄取、抽取、归一化、向量嵌入和存储。"
---

**[English](pipeline.md)** · **简体中文（当前）**

`PipelineBuilder` 解决了处理步骤之间的胶水问题。声明你的步骤、注册处理函数、连接依赖关系，然后把控制权交给 `ExecutionEngine` —— 它会处理拓扑排序、在步骤之间传递输出、在失败时按可配置的退避策略重试，并返回一个结构化的 `ExecutionResult` 供你记录日志或告警。

<a id="why-use-pipelines"></a>
## 为什么使用流水线？

流水线解决了多步骤数据处理中的协调问题。你不必编写每个函数都调用下一个函数的单体脚本，而是定义独立的步骤并声明它们的依赖关系。流水线引擎会自行算出执行顺序、并行运行相互独立的步骤，并在步骤之间自动传递数据。

当遇到以下情况时，这就变得至关重要：
- 多个需要不同处理的数据源
- 可以并发运行的步骤
- 复杂的重试或错误恢复需求
- 频繁变更的工作流
- 不同的人负责不同步骤的团队协作环境

一条带两个并行分支的五步流水线，执行速度会比线性脚本更快；而要增加第六个步骤，只需声明它在 DAG 中的位置即可。

<a id="when-to-use-when-not-to-use"></a>
## 适用场景 / 不适用场景

**在以下场景使用流水线：**
- 具有 3 个以上处理阶段的多步 ETL 工作流
- 每天或每小时运行的定时生产作业
- 某些步骤可以并行运行的工作流
- 能从模块化设计中受益的复杂数据转换
- 不同的人维护不同步骤的团队协作环境

**不要在以下场景使用流水线：**
- 一次性脚本或原型（改用简单的函数）
- 只有 1-2 个步骤的线性工作流
- 在 Jupyter notebook 中进行的探索性数据分析
- 定义步骤的开销超过工作流复杂度的场景

<a id="typical-pipeline-workflow"></a>
## 典型流水线工作流

无论哪个领域，大多数数据流水线都遵循一个四阶段模式：

1. **摄取** —— 从文件、API、数据库或流中拉取数据
2. **转换** —— 对原始数据进行清洗、验证、归一化和富化
3. **抽取** —— 运行实体识别、关系抽取或其他 NLP 任务
4. **存储** —— 将结果写入向量库、知识图谱或输出文件

每个阶段都可以有多个并行步骤。例如，摄取阶段可能同时从三个不同的 REST API 拉取数据，而转换阶段则对每个来源应用不同的清洗规则。

<Info>
  `PipelineBuilder` 和 `ExecutionEngine` 位于 `semantica.pipeline` 中。失败处理、重试策略和并行管理都是独立的类，你可以单独导入以进行细粒度控制。自定义步骤处理函数就是普通的 Python 函数 —— 无需子类化。
</Info>

<a id="your-first-pipeline"></a>
## 你的第一条流水线

最小可用的流水线有三个步骤：摄取、抽取、存储。定义它们、连接它们、构建、执行。

每个步骤都是一个以 `data` 为第一个位置参数的函数，并通过关键字参数接收步骤配置。流水线引擎会用上游数据（来自依赖项）调用你的步骤处理函数，并通过 `**kwargs` 传递任何步骤配置：

```python
def handler(data, **config):
    # data 包含来自所有上游依赖的输出
    # config 包含传给 add_step() 的所有参数
    processed_data = transform_logic(data, config.get("param1", "default"))
    return processed_data  # 始终为下游步骤返回数据
```

根步骤（没有依赖项的步骤）的 `data` 参数会收到 `None` —— 它们从零生成数据，而不是转换上游输出。

将处理函数直接通过 `handler` 关键字参数传给 `add_step()`，同时传入任何步骤配置：

```python
builder.add_step("extract_step", "ner_extract", handler=extract_entities_handler)
builder.connect_steps("ingest_step", "extract_step")
```

```python
from semantica.pipeline import PipelineBuilder, ExecutionEngine
from semantica.ingest import ingest_file
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

# --- 定义处理函数 ---
# 引擎调用 handler(data, **step_config)，所以第一个位置参数
# 始终是上游数据；步骤配置以 kwargs 传入。

def ingest_stix_bundles(data, **config):
    # 根步骤 —— data 为 None，从 config 生成数据
    files = ingest_file(config["path"], method="directory")
    return [f.text for f in files if f.file_type == "json"]

def extract_entities(data, **config):
    # data 是上一步的输出
    threshold = config.get("confidence_threshold", 0.7)
    results = []
    for text in data:
        # 你的 NER 逻辑放在这里 —— 此处为便于说明已简化
        results.append({"text": text, "entities": [], "threshold": threshold})
    return results

def build_graph(data, **config):
    graph   = ContextGraph(advanced_analytics=True)
    context = AgentContext(
        vector_store    = VectorStore(backend="faiss", dimension=768),
        knowledge_graph = graph,
    )
    texts = [d["text"] for d in data]
    context.store(texts, extract_entities=True, extract_relationships=True)
    context.save(config["output_path"])
    return {"node_count": graph.stats()["node_count"],
            "edge_count": graph.stats()["edge_count"]}

# --- 构建流水线 ---
# 在 add_step() 中直接传 handler=，这样每个 PipelineStep 都携带其可调用对象。

builder = PipelineBuilder()

builder.add_step("ingest",  "stix_ingest",  handler=ingest_stix_bundles, path="./stix_bundles/")
builder.add_step("extract", "ner_extract",  handler=extract_entities,    confidence_threshold=0.75)
builder.add_step("store",   "kg_build",     handler=build_graph,         output_path="./cti_output/")

builder.connect_steps("ingest",  "extract")
builder.connect_steps("extract", "store")

pipeline = builder.build("cti_pipeline")

# --- 执行 ---

engine = ExecutionEngine(max_workers=4, retry_on_failure=True)
result = engine.execute_pipeline(pipeline)

print(f"Success:  {result.success}")
print(f"Output:   {result.output}")   # {"node_count": 312, "edge_count": 847}
print(f"Duration: {result.metrics['execution_time']:.2f}s")
print(f"Steps completed: {result.metrics['steps_executed']}")
```

`ExecutionEngine` 在执行之前会对步骤图做拓扑排序，所以即使你以错误的顺序声明步骤，执行顺序也总是正确的。每个步骤会收到上一步的返回值作为它的 `data` 参数。

<a id="reading-the-executionresult"></a>
## 读取 ExecutionResult

每次 `engine.execute_pipeline()` 调用都会返回一个 `ExecutionResult` 数据类。在假定成功之前先检查它：

```python
result = engine.execute_pipeline(pipeline)

if not result.success:
    print("Pipeline failed. Errors:")
    for err in result.errors:
        print(f"  {err}")
else:
    print(f"Pipeline completed in {result.metrics['execution_time']:.1f}s")
    print(f"Steps run:    {result.metrics['steps_executed']}")
    print(f"Steps failed: {result.metrics['steps_failed']}")
    # result.output  — the return value of the final step
    # result.metadata — {"pipeline_id": "...", "execution_time": float}
```

`result.errors` 是一个 `List[str]` —— 每个失败的步骤对应一条，每条包含异常消息。一条设置了 `retry_on_failure=True` 的流水线在将某个失败步骤记录为失败并继续之前，最多会尝试 `max_retries` 次（默认：3 次）。

<a id="handling-failures-and-configuring-retry-policy"></a>
## 处理失败和配置重试策略

默认情况下，`ExecutionEngine(retry_on_failure=True)` 使用指数退避策略：重试三次，从 1 秒开始，每次翻倍，上限为 60 秒。对于调用外部 API 或数据库的步骤 —— 这些场景下瞬时失败是预期内的 —— 你可以通过 `FailureHandler` 设置按步骤类型的策略：

```python
from semantica.pipeline import ExecutionEngine, FailureHandler, RetryPolicy, RetryStrategy

# 构建一个自定义的失败处理器
handler = FailureHandler()

# Web/API 步骤：使用指数退避最多重试 5 次
handler.set_retry_policy(
    "misp_fetch",
    RetryPolicy(
        max_retries    = 5,
        strategy       = RetryStrategy.EXPONENTIAL,
        backoff_factor = 2.0,
        initial_delay  = 2.0,
        max_delay      = 120.0,
    ),
)

# 数据库步骤：固定延迟，较少重试次数（连接池通常恢复很快）
handler.set_retry_policy(
    "db_ingest",
    RetryPolicy(
        max_retries   = 3,
        strategy      = RetryStrategy.FIXED,
        initial_delay = 5.0,
    ),
)

# NER 步骤：不重试 —— 如果模型崩溃则需要人工介入
handler.set_retry_policy(
    "ner_extract",
    RetryPolicy(max_retries=0),
)

engine = ExecutionEngine(
    max_workers      = 4,
    retry_on_failure = True,
)
# 当某个步骤失败时，引擎使用 handler.get_retry_policy(step.step_type)
```

`handler.classify_error()` 会区分 `ValidationError`（低严重性，通常不重试）、`ProcessingError`（高严重性）以及超时/连接错误（中等严重性，总是重试）。你可以查看分类结果：

```python
try:
    result = engine.execute_pipeline(pipeline)
except Exception as e:
    classification = handler.classify_error(e)
    print(f"Severity: {classification['severity'].value}")   # "high" / "medium" / "low"
    print(f"Message: {classification['message']}")
```

<a id="running-steps-in-parallel"></a>
## 并行运行步骤

当两个步骤互不依赖时 —— 例如，NER 抽取和三元组抽取都读取同一个摄取输出 —— 通过将两者都连接到同一个上游步骤，把它们声明为并行分支：

```python
builder = PipelineBuilder()
builder.register_step_handler("file_ingest",     ingest_stix_bundles)
builder.register_step_handler("ner_extract",     run_ner)
builder.register_step_handler("triplet_extract", run_triplets)
builder.register_step_handler("kg_merge",        merge_into_graph)

builder.add_step("ingest",   "file_ingest",     handler=ingest_stix_bundles, path="./stix_bundles/")
builder.add_step("ner",      "ner_extract",     handler=run_ner,             confidence_threshold=0.75)
builder.add_step("triplets", "triplet_extract", handler=run_triplets,        include_temporal=True)
builder.add_step("store",    "kg_merge",        handler=merge_into_graph,    output_path="./cti_output/")

# ingest 同时喂给 ner 和 triplets（并行）
builder.connect_steps("ingest",   "ner")
builder.connect_steps("ingest",   "triplets")
# 两者都汇聚到 store
builder.connect_steps("ner",      "store")
builder.connect_steps("triplets", "store")

builder.set_parallelism(2)   # 并发运行 ner 和 triplets

pipeline = builder.build("parallel_extraction")
engine   = ExecutionEngine(max_workers=2, retry_on_failure=True)
result   = engine.execute_pipeline(pipeline)
```

`set_parallelism(n)` 告诉引擎它可以同时运行多少个步骤。拓扑排序保证只有依赖项全部完成的步骤才有资格并发执行 —— 你不会在某个步骤的输入就绪之前意外运行它。

<a id="common-pitfalls"></a>
## 常见陷阱

**忘记返回数据。** 如果一个步骤处理函数不返回任何内容，下游步骤的 `data` 参数会收到 `None`。这通常会导致崩溃或静默失败。每个非终止步骤都应该为下一个阶段返回数据。

**过度的并行。** 在一台 8 核机器上设置 `max_workers=50` 带来的开销会超过加速效果。从 `max_workers=4` 开始，在监控资源使用的同时逐步增加。大多数 I/O 密集型步骤在适度并行下表现良好。

**有状态的处理函数。** 步骤处理函数应该是纯函数 —— 给定相同的输入数据和配置，应该产生相同的输出。避免使用在调用之间持久存在的全局变量、文件句柄或数据库连接。每次步骤执行都应该是独立的。

**调试大型流水线。** 当一条 10 步的流水线在第 7 步失败时，不要为了调试而重新运行整条流水线。把失败的步骤提取到一个独立脚本中，用实际的中间数据作为输入，并在隔离环境中修复它。

<a id="development-and-debugging"></a>
## 开发与调试

从小处开始，在构建完整流水线之前逐个验证每个步骤：

```python
# 在开发期间于处理函数中添加调试输出
def extract_entities(data, **config):
    print(f"Processing {len(data)} items with threshold {config.get('confidence_threshold')}")
    results = []
    for item in data:
        # 你的处理逻辑
        results.append({"text": item["text"], "entities": []})
    print(f"Generated {len(results)} results")
    return results

# 用已知数据测试单个处理函数
test_data = [{"text": "sample data", "entities": []}]
result = extract_entities(test_data, confidence_threshold=0.5)
print(f"Processed {len(result)} items")
```

对于复杂流水线，可以添加检查点来保存中间输出：

```python
def save_checkpoint(data, **config):
    # 保存中间数据用于调试
    checkpoint_path = config.get("checkpoint_path", "checkpoint.json")
    with open(checkpoint_path, "w") as f:
        json.dump(data, f)
    return data  # 原样传递数据

builder.add_step(
    "checkpoint_after_transform", "checkpoint",
    handler=save_checkpoint,
    checkpoint_path="transform_output.json"
)
builder.connect_steps("transform", "checkpoint_after_transform")
```

<a id="delta-incremental-processing"></a>
## 增量处理

增量流水线只处理你的数据两个版本之间的变更，而不是从头重新处理所有内容。这对于全量重新处理太慢或太昂贵的大型数据集至关重要。

你的 STIX 包目录每天晚上会增加 20-30 个新文件。每天早上重新处理全部 4000 个历史文件会浪费时间和算力。在摄取步骤上设置 `delta_mode=True` 会告诉流水线只处理自上次版本快照以来发生变更的文件：

```python
builder = PipelineBuilder()
builder.register_step_handler("stix_ingest", ingest_stix_bundles)
builder.register_step_handler("ner_extract", run_ner)
builder.register_step_handler("kg_append",   append_to_graph)

builder.add_step(
    "ingest", "stix_ingest",
    handler           = ingest_stix_bundles,
    path              = "./stix_bundles/",
    delta_mode        = True,
    base_version_id   = "2024-11-30",   # 上次成功的运行
    target_version_id = "2024-12-01",   # 今天的快照
)
builder.add_step("extract", "ner_extract", handler=run_ner)
builder.add_step("store",   "kg_append",   handler=append_to_graph, output_path="./cti_output/")

builder.connect_steps("ingest",  "extract")
builder.connect_steps("extract", "store")

pipeline = builder.build("delta_pipeline")
result   = ExecutionEngine(max_workers=4, retry_on_failure=True).execute_pipeline(pipeline)

print(f"Delta run: {result.output}")
```

`base_version_id` 和 `target_version_id` 存储在 `PipelineStep` 数据类上，并通过 `config` 传递给你的处理函数 —— 你的处理函数负责使用它们来过滤输入。一种典型的模式是将文件修改时间戳与基础版本日期进行比较。

<a id="building-from-a-config-dict"></a>
## 从配置字典构建

对于定义在配置文件中的流水线 —— 当不同环境（开发、预发布、生产）以不同的路径和阈值运行同一条流水线时很有用 —— 可以向 `build_pipeline()` 传入一个字典，而不是手动调用 `add_step()`：

```python
from semantica.pipeline import PipelineBuilder, ExecutionEngine

pipeline_config = {
    "name": "cti_pipeline",
    "parallelism": 4,
    "steps": [
        {
            "name": "ingest",
            "type": "stix_ingest",
            "config": {"path": "./stix_bundles/"},
        },
        {
            "name": "extract",
            "type": "ner_extract",
            "config": {"confidence_threshold": 0.8},
        },
        {
            "name": "store",
            "type": "kg_build",
            "config": {"output_path": "./cti_output/"},
        },
    ],
}

builder = PipelineBuilder()
# 像之前一样注册处理函数，然后：
pipeline = builder.build_pipeline(pipeline_config)

engine = ExecutionEngine(max_workers=4, retry_on_failure=True)
result = engine.execute_pipeline(pipeline)
```

注意，`build_pipeline()` 从每个步骤配置字典中的 `"dependencies"` 键读取步骤连接（而不是通过 `connect_steps()` 调用）。如果你使用这条路径，请显式添加依赖项：

```python
{
    "name": "extract",
    "type": "ner_extract",
    "config": {"confidence_threshold": 0.8, "dependencies": ["ingest"]},
},
```

<a id="monitoring-progress"></a>
## 监控进度

`ExecutionEngine` 会自动与 Semantica 的进度跟踪器集成 —— 每次步骤的启动、更新和完成都会被记录。要观察长时间运行的流水线的进度，可以在执行之后检查 `Pipeline` 对象上的步骤状态：

```python
from semantica.pipeline import StepStatus

result   = engine.execute_pipeline(pipeline)
for step in pipeline.steps:
    status_str = step.status.value   # "completed" / "failed" / "skipped"
    print(f"  {step.name:20s}  {status_str}")
    if step.status == StepStatus.FAILED and step.error:
        print(f"    Error: {step.error}")
```

`result.metrics` 给出聚合视图：

```python
print(f"Total time:      {result.metrics['execution_time']:.2f}s")
print(f"Steps completed: {result.metrics['steps_executed']}")
print(f"Steps failed:    {result.metrics['steps_failed']}")
```

<a id="domain-examples"></a>
## 领域示例

<Tabs>
  <Tab title="国防 — CTI/威胁">
    一个 SOC 威胁情报团队需要一条端到端的流水线，它从机密目录中摄取 STIX 包，用自定义的威胁行为者标签运行实体抽取，并构建一个可供分析师查询的 `ContextGraph`。该流水线每六小时运行一次；失败的步骤会自动重试，这样一次瞬时的文件系统错误不会丢掉一个摄取周期。

```python
from semantica.pipeline import PipelineBuilder, ExecutionEngine, RetryPolicy, RetryStrategy
from semantica.ingest import ingest_file
from semantica.semantic_extract import NamedEntityRecognizer
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

def ingest_classified_stix(data, **config):
    files = ingest_file(config["path"], method="directory")
    return [
        {"text": f.text, "source": f.name, "classification": config["classification"]}
        for f in files if f.file_type == "json"
    ]

def extract_cti_entities(data, **config):
    ner = NamedEntityRecognizer(
        methods=["pattern", "ml"],
        custom_labels=["THREAT_ACTOR", "MALWARE", "CVE", "C2_DOMAIN", "CAMPAIGN"],
        confidence_threshold=config.get("confidence_threshold", 0.80),
    )
    results = []
    for doc in data:
        entities = ner.extract_entities(doc["text"])
        results.append({**doc, "entities": [e.__dict__ for e in entities]})
    return results

def build_cti_graph(data, **config):
    graph   = ContextGraph(advanced_analytics=True, community_detection=True)
    context = AgentContext(
        vector_store    = VectorStore(backend="faiss", dimension=768),
        knowledge_graph = graph,
        graph_expansion = True,
    )
    texts = [d["text"] for d in data]
    context.store(texts, extract_entities=True, extract_relationships=True)
    context.save(config["output_path"])
    return graph.stats()

builder = PipelineBuilder()
builder.register_step_handler("classified_ingest", ingest_classified_stix)
builder.register_step_handler("cti_ner",           extract_cti_entities)
builder.register_step_handler("cti_graph",         build_cti_graph)

builder.add_step("ingest",  "classified_ingest",
                 handler=ingest_classified_stix,
                 path="./classified/stix/",
                 classification="SECRET//REL TO USA FVEY")
builder.add_step("extract", "cti_ner", handler=extract_cti_entities, confidence_threshold=0.85)
builder.add_step("store",   "cti_graph", handler=build_cti_graph, output_path="./cti_state/")

builder.connect_steps("ingest",  "extract")
builder.connect_steps("extract", "store")
builder.set_parallelism(2)

pipeline = builder.build("cti_pipeline")
engine   = ExecutionEngine(max_workers=2, retry_on_failure=True)
result   = engine.execute_pipeline(pipeline)

print(f"CTI pipeline: success={result.success}, "
      f"nodes={result.output.get('node_count')}, "
      f"time={result.metrics['execution_time']:.1f}s")
```

  </Tab>

  <Tab title="安全 — SOC/事件">
    在一次活跃的 P1 事件期间，SOC 需要一条 15 分钟周期的流水线，拉取最新的 SIEM 告警 CSV，将 CVE 与内部数据库进行交叉引用，并更新事件知识图谱。该流水线运行在一个紧凑的周期上 —— NER 失败不得阻塞图谱更新，因此步骤配置为在 NER 错误上不重试，但在数据库超时时激进重试。

```python
from semantica.pipeline import (
    PipelineBuilder, ExecutionEngine, FailureHandler,
    RetryPolicy, RetryStrategy,
)
from semantica.ingest import ingest_file, DBIngestor
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

def ingest_siem_alerts(data, **config):
    files = ingest_file(config["path"], method="directory")
    return [{"text": f.text, "source": f.name}
            for f in files if f.file_type == "csv"]

def enrich_with_cves(data, **config):
    db = DBIngestor()
    cve_rows = db.execute_query(config["db_url"], config["query"])
    cve_lookup = {r["cve_id"]: r for r in cve_rows}
    for doc in data:
        doc["cve_enrichment"] = cve_lookup
    return data

def update_incident_graph(data, **config):
    graph   = ContextGraph(advanced_analytics=True)
    context = AgentContext(
        vector_store    = VectorStore(backend="faiss", dimension=768),
        knowledge_graph = graph,
    )
    texts = [d["text"] for d in data]
    context.store(texts, extract_entities=True, extract_relationships=True)
    context.save(config["output_path"])
    return graph.stats()

handler = FailureHandler()
handler.set_retry_policy("db_enrich",
    RetryPolicy(max_retries=5, strategy=RetryStrategy.EXPONENTIAL,
                initial_delay=2.0, max_delay=30.0))
handler.set_retry_policy("cti_ner", RetryPolicy(max_retries=0))

builder = PipelineBuilder()
builder.register_step_handler("siem_ingest",  ingest_siem_alerts)
builder.register_step_handler("db_enrich",    enrich_with_cves)
builder.register_step_handler("graph_update", update_incident_graph)

builder.add_step("ingest",  "siem_ingest",  handler=ingest_siem_alerts, path="./siem_exports/")
builder.add_step("enrich",  "db_enrich",
                 handler=enrich_with_cves,
                 db_url="postgresql://ro:pass@cvedb:5432/nvd",
                 query="SELECT cve_id, description, cvss_v3_score FROM cve_records "
                       "WHERE cve_id = ANY(ARRAY['CVE-2024-3400','CVE-2024-21762'])")
builder.add_step("store",   "graph_update", handler=update_incident_graph, output_path="./incident_state/")

builder.connect_steps("ingest",  "enrich")
builder.connect_steps("enrich",  "store")

pipeline = builder.build("incident_pipeline")
engine   = ExecutionEngine(max_workers=4, retry_on_failure=True)
result   = engine.execute_pipeline(pipeline)

print(f"Incident update: {result.success}, "
      f"nodes={result.output.get('node_count')}, "
      f"errors={result.errors}")
```

  </Tab>

  <Tab title="生命科学 — 临床/制药">
    一条临床 NLP 流水线在每个试验月份结束后处理 EHR 导出数据：从共享目录摄取患者笔记，使用 HuggingFace 模型运行生物医学 NER 以抽取药物/疾病/剂量实体，记录符合 ICH E6(R2) 的溯源，并追加到试验图谱中。该流水线使用增量模式，因此只处理自上次运行以来新增的笔记。

```python
from semantica.pipeline import PipelineBuilder, ExecutionEngine
from semantica.ingest import ingest_file
from semantica.semantic_extract import NamedEntityRecognizer
from semantica.provenance import ProvenanceManager, SourceReference
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

def ingest_ehr_notes(data, **config):
    files = ingest_file(config["path"], method="directory")
    return [
        {"text": f.text, "patient_id": f.name.split("_")[0], "source": f.name}
        for f in files
    ]

def run_biomedical_ner(data, **config):
    ner = NamedEntityRecognizer(
        methods=["huggingface"],
        huggingface_model="d4data/biomedical-ner-all",
        confidence_threshold=config.get("confidence_threshold", 0.80),
        custom_labels=["DRUG", "DISEASE", "DOSAGE", "GENE", "BIOMARKER"],
    )
    results = []
    for doc in data:
        entities = ner.extract_entities(doc["text"])
        results.append({**doc, "entities": entities})
    return results

def track_provenance_and_store(data, **config):
    manager = ProvenanceManager(storage_path=config["provenance_db"])
    graph   = ContextGraph(advanced_analytics=True)
    context = AgentContext(
        vector_store    = VectorStore(backend="faiss", dimension=768),
        knowledge_graph = graph,
        retention_days  = None,
    )
    for doc in data:
        for entity in doc.get("entities", []):
            source = SourceReference(
                document=doc["patient_id"],
                section="clinical_note",
                confidence=getattr(entity, "confidence", 1.0),
            )
            manager.track_entity(
                entity_id=f"{doc['patient_id']}_{getattr(entity, 'text', '')}",
                source=source.document,
                metadata={"entity_type": getattr(entity, "label", ""),
                           "confidence": getattr(entity, "confidence", 1.0)},
            )
    context.store([d["text"] for d in data], extract_entities=True)
    context.save(config["output_path"])
    return graph.stats()

builder = PipelineBuilder()
builder.register_step_handler("ehr_ingest",    ingest_ehr_notes)
builder.register_step_handler("bio_ner",       run_biomedical_ner)
builder.register_step_handler("prov_store",    track_provenance_and_store)

builder.add_step("ingest",  "ehr_ingest",
                 handler=ingest_ehr_notes,
                 path="./ehr_exports/month_12/",
                 delta_mode=True,
                 base_version_id="2024-11",
                 target_version_id="2024-12")
builder.add_step("extract", "bio_ner", handler=run_biomedical_ner, confidence_threshold=0.82)
builder.add_step("store",   "prov_store",
                 handler=track_provenance_and_store,
                 provenance_db="./provenance/trial_Q4.db",
                 output_path="./trial_state/")

builder.connect_steps("ingest",  "extract")
builder.connect_steps("extract", "store")
builder.set_parallelism(4)

pipeline = builder.build("clinical_nlp_pipeline")
engine   = ExecutionEngine(max_workers=4, retry_on_failure=True)
result   = engine.execute_pipeline(pipeline)

print(f"Clinical pipeline: {result.success}, "
      f"nodes={result.output.get('node_count')}")
```

  </Tab>

  <Tab title="银行业 — 风险/合规">
    一个合规团队每月运行一条增量流水线：从站点地图摄取新的 BIS 出版物，抽取资本比率和阈值实体，并将它们追加到已经持有三年监管历史的现有合规知识图谱中。增量模式确保只处理自上次运行以来发布的页面。

```python
from semantica.pipeline import PipelineBuilder, ExecutionEngine
from semantica.ingest import ingest_web
from semantica.semantic_extract import NamedEntityRecognizer, TripletExtractor
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

def ingest_bis_pages(data, **config):
    pages = ingest_web(config["sitemap"], method="sitemap")
    return [
        {"text": p.text, "url": p.url}
        for p in pages
        if p.text.strip() and any(
            kw in p.url.lower() for kw in ["bcbs", "capital", "liquidity"]
        )
    ]

def extract_regulatory_entities(data, **config):
    ner = NamedEntityRecognizer(
        methods=["pattern", "ml"],
        custom_labels=["REGULATION", "CAPITAL_RATIO", "RISK_WEIGHT", "THRESHOLD"],
        confidence_threshold=config.get("confidence_threshold", 0.75),
    )
    extractor = TripletExtractor(
        method="pattern", include_temporal=True, include_provenance=True,
    )
    results = []
    for doc in data:
        entities = ner.extract_entities(doc["text"])
        triplets = extractor.extract_triplets(doc["text"], entities=entities)
        results.append({**doc, "entities": entities, "triplets": triplets})
    return results

def append_to_compliance_graph(data, **config):
    graph = ContextGraph(advanced_analytics=True)
    graph.load_from_file(config["existing_graph"])
    context = AgentContext(
        vector_store    = VectorStore(backend="faiss", dimension=768),
        knowledge_graph = graph,
        retention_days  = 2555,   # 7-year regulatory requirement
    )
    context.store([d["text"] for d in data],
                  extract_entities=True, extract_relationships=True)
    context.save(config["output_path"])
    return {"added_docs": len(data), "graph": graph.stats()}

builder = PipelineBuilder()
builder.register_step_handler("bis_ingest",   ingest_bis_pages)
builder.register_step_handler("rule_extract", extract_regulatory_entities)
builder.register_step_handler("graph_append", append_to_compliance_graph)

builder.add_step("ingest",  "bis_ingest",
                 handler=ingest_bis_pages,
                 sitemap="https://www.bis.org/sitemap.xml",
                 delta_mode=True,
                 base_version_id="2024-11",
                 target_version_id="2024-12")
builder.add_step("extract", "rule_extract", handler=extract_regulatory_entities, confidence_threshold=0.75)
builder.add_step("store",   "graph_append",
                 handler=append_to_compliance_graph,
                 existing_graph="./compliance_graph/knowledge_graph.json",
                 output_path="./compliance_graph/")

builder.connect_steps("ingest",  "extract")
builder.connect_steps("extract", "store")
builder.set_parallelism(4)

pipeline = builder.build("regulatory_pipeline")
engine   = ExecutionEngine(max_workers=4, retry_on_failure=True)
result   = engine.execute_pipeline(pipeline)

print(f"Compliance delta update: {result.output}")
```

  </Tab>
</Tabs>

<a id="related-guides"></a>
## 相关指南

- [摄取](ingest.zh-CN.md) —— 摄取步骤支持的所有源类型：PDF、API、数据库、RSS 源、STIX 目录和流
- [语义抽取](semantic-extraction.zh-CN.md) —— 用于抽取步骤的 NER、关系抽取、三元组抽取和事件检测
- [上下文图谱](context-graphs.zh-CN.md) —— 构建和查询存储步骤所填充的 `ContextGraph`
- [溯源](provenance.zh-CN.md) —— 为每个抽取的实体跟踪来源文档、置信度分数和流水线运行 ID
