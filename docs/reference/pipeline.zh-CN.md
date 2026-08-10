---
title: "流水线模块"
description: "带有并行工作进程、重试策略、故障处理和进度跟踪的流水线 DSL。"
icon: "gear"
---

**[English](pipeline.md)** · **简体中文（当前）**

**`semantica.pipeline`** 让你将 Semantica 组件串联成**可复现、容错的工作流**：

- 按步骤的故障策略：`skip`、`retry`、`abort` 或 `fallback`
- 通过 `ParallelismManager` 实现并行工作进程：线程池或进程池
- `PipelineValidator` 在运行前捕获循环、缺失的处理程序和配置错误
- 预置模板：`"document_processing"`、`"rag_pipeline"`、`"kg_construction"`、`"ontology_generation"`
- 流水线可序列化为 YAML：可在任何环境中保存和重新加载


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `PipelineBuilder` | 用于串联步骤的 DSL：`add_step`、`connect_steps`、`set_parallel`、`build` |
| `ExecutionEngine` | 运行已构建的流水线：`execute_pipeline(pipeline, data)` → `ExecutionResult` |
| `ExecutionResult` | `{success, output, metadata, metrics, errors}`：完整的运行摘要 |
| `FailureHandler` | 按步骤的策略：失败时 `skip`、`retry`、`abort` 或 `fallback` |
| `ParallelismManager` | 用于并发执行步骤的线程池或进程池，工作进程数量可配置 |
| `PipelineValidator` | 在运行前捕获依赖循环、缺失的处理程序和配置错误 |
| `PipelineTemplateManager` | 预置模板：`"document_processing"`、`"rag_pipeline"`、`"kg_construction"`、`"ontology_generation"` |

<a id="why-use-a-pipeline"></a>
## 为什么要使用流水线？

你可以用纯 Python 代码将 Semantica 模块串联起来。流水线增加了：

- **重试和故障处理** — 一个损坏的文档不会让 10,000 个文档的运行崩溃。
- **并行性** — 用一个参数在多个工作进程中运行抽取。
- **进度跟踪** — tqdm 控制台进度条或通过 WebSocket 流式传输到 Explorer。
- **可复现性** — 将确切的流水线配置保存为 YAML，并在任何机器上重放。
- **增量模式** — 重新运行时，仅处理自上次运行以来发生变化的文档。
- **验证** — 在配置错误的步骤和依赖循环导致运行中失败之前捕获它们。

<Note>
  对于快速脚本和笔记本，使用普通的模块调用。对于任何需要重复运行、大规模运行或在生产环境中运行的任务，请使用流水线。
</Note>

<img src="/assets/img/diagrams/pipeline-flow.svg" alt="流水线步骤顺序：摄取 → 解析 → 归一化 → 抽取 → 构建 KG → 问答 → 存储 → 交付" style={{ width: '100%', borderRadius: '10px', margin: '0 0 24px' }} />

<a id="quick-start"></a>
## 快速上手

<Steps>
  <Step title="构建流水线">
    ```python
    from semantica.pipeline import PipelineBuilder
    from semantica.ingest import FileIngestor
    from semantica.parse import DocumentParser
    from semantica.semantic_extract import NERExtractor
    from semantica.kg import GraphBuilder

    ingestor   = FileIngestor()
    parser     = DocumentParser()
    extractor  = NERExtractor(method="ml")
    kg_builder = GraphBuilder(merge_entities=True)

    builder = PipelineBuilder()
    builder.add_step("ingest",   "file_ingest",    handler=ingestor.ingest_file)
    builder.add_step("parse",    "document_parse", handler=parser.parse)
    builder.add_step("extract",  "ner_extract",    handler=extractor.extract)
    builder.add_step("build_kg", "graph_build",    handler=kg_builder.build)
    builder.connect_steps("ingest", "parse")
    builder.connect_steps("parse",  "extract")
    builder.connect_steps("extract","build_kg")

    pipeline = builder.build("my_pipeline")
    ```
  </Step>
  <Step title="运行前验证">
    ```python
    from semantica.pipeline import PipelineValidator

    validator = PipelineValidator()
    result = validator.validate_pipeline(pipeline)

    if not result.valid:
        for error in result.errors:   # errors 是 List[str]
            print(f"Error: {error}")
        for warning in result.warnings:
            print(f"Warning: {warning}")
    ```

    <Tip>
      **在生产环境运行前使用 `PipelineValidator`。** 它能捕获依赖循环、缺失的步骤名称以及会在运行中才暴露为错误的配置错误的连接。验证是即时的；在 30 分钟的抽取作业后发现这些问题就不然了。
    </Tip>
  </Step>
  <Step title="执行并检查结果">
    ```python
    from semantica.pipeline import ExecutionEngine

    engine = ExecutionEngine()
    result = engine.execute_pipeline(pipeline, data="data/")

    kg = result.output
    print(f"Success:        {result.success}")
    print(f"Steps executed: {result.metrics['steps_executed']}")
    print(f"Steps failed:   {result.metrics['steps_failed']}")
    print(f"Duration:       {result.metrics['execution_time']:.1f}s")
    ```

    <Tip>
      **检查 `result.metrics` 以查找瓶颈。** `result.metrics['steps_executed']` 和 `result.metrics['execution_time']` 提供对整体流水线健康状况的快速了解。如需按步骤的计时，请在运行后检查每个 `PipelineStep` 上的 `step.result`。
    </Tip>
  </Step>
</Steps>

<a id="parallel-processing"></a>
## 并行处理

在构建器上设置并行性，并将 `max_workers` 传给 `ExecutionEngine`：

```python
from semantica.pipeline import PipelineBuilder, ExecutionEngine

builder = PipelineBuilder()
builder.add_step("ingest",  "file_ingest",    handler=ingestor.ingest_file)
builder.add_step("parse",   "document_parse", handler=parser.parse)
builder.add_step("extract", "ner_extract",    handler=extractor.extract)
builder.add_step("build",   "graph_build",    handler=kg_builder.build)
builder.set_parallelism(4)

pipeline = builder.build("parallel_pipeline")
engine   = ExecutionEngine(max_workers=4)
result   = engine.execute_pipeline(pipeline, data="data/")
```

<Tip>
  **根据工作负载类型设置 `workers=`。** I/O 密集型步骤（Web 抓取、数据库查询）使用线程工作进程，CPU 密集型步骤（嵌入、OCR、大型 NER 批处理）使用进程工作进程。在错误的步骤类型上混用池类型会浪费资源而不会带来速度提升。
</Tip>

<a id="retry-and-error-handling"></a>
## 重试和错误处理

<Tabs>
  <Tab title="指数退避（推荐）">
    ```python
    from semantica.pipeline import RetryPolicy, RetryStrategy, FailureHandler, ExecutionEngine

    policy = RetryPolicy(
        max_retries=3,
        strategy=RetryStrategy.EXPONENTIAL,
        initial_delay=1.0,    # 1s → 2s → 4s
        backoff_factor=2.0
    )

    handler = FailureHandler()
    handler.set_retry_policy("ner_extract", policy)   # 以 step_type 为键

    engine = ExecutionEngine(default_max_retries=3, default_backoff_factor=2.0)
    result = engine.execute_pipeline(pipeline, data="data/")
    ```

    最适合瞬时 API 错误和速率限制：每次重试等待时间更长，给上游服务恢复时间。
  </Tab>
  <Tab title="线性退避">
    ```python
    from semantica.pipeline import RetryPolicy, RetryStrategy

    policy = RetryPolicy(
        max_retries=3,
        strategy=RetryStrategy.LINEAR,
        initial_delay=2.0    # 2s → 4s → 6s
    )
    ```

    当重试之间的延迟应可预测增长时使用：例如，等待数据库锁释放。
  </Tab>
  <Tab title="固定退避">
    ```python
    from semantica.pipeline import RetryPolicy, RetryStrategy

    policy = RetryPolicy(
        max_retries=5,
        strategy=RetryStrategy.FIXED,
        initial_delay=1.0    # 每次尝试 1s
    )
    ```

    当对具有固定冷却窗口的服务进行重试时使用。
  </Tab>
</Tabs>

<a id="failure-strategies"></a>
### 故障策略

| 策略 | 行为 | 适用场景 |
| :-------- | :--------- | :----------- |
| `"skip"` | 记录失败，继续下一个文档 | 生产环境：一个坏文档不应阻止 10k 个 |
| `"stop"` | 立即抛出异常 | 开发环境：快速暴露错误 |
| `"retry"` | 通过 `RetryPolicy` 重试，然后跳过 | 当故障可能是瞬时的时候 |

<Warning>
  在生产环境中，配置具有有限重试次数的 `RetryPolicy`，以便单个失败步骤不会停止整个运行。执行后，检查 `result.errors` 以查找并重新处理失败的文档。
</Warning>

<Warning>
  **配置重试策略以在生产环境中控制故障。** 使用 `handler.set_retry_policy("step_type", RetryPolicy(max_retries=3))`，这样瞬时错误会被重试而不会停止流水线。运行后，检查 `result.errors` 以查找并重新处理任何耗尽重试次数的文档。
</Warning>

<a id="progress-tracking"></a>
## 进度跟踪

<Tabs>
  <Tab title="控制台（tqdm）">
    ```python
    from semantica.pipeline import ExecutionEngine

    engine = ExecutionEngine()
    result = engine.execute_pipeline(pipeline, data="data/")
    # 进度跟踪器在执行期间向控制台输出 tqdm 进度条
    ```

    通过 Semantica 内置的进度跟踪器在终端显示实时进度条。最适合脚本和 CLI 工具。
  </Tab>
  <Tab title="实时状态检查">
    ```python
    from semantica.pipeline import ExecutionEngine
    import threading, time

    engine = ExecutionEngine()

    # 在后台线程中运行，从主线程轮询进度
    def run():
        engine.execute_pipeline(pipeline, data="data/")

    t = threading.Thread(target=run, daemon=True)
    t.start()

    while t.is_alive():
        progress = engine.get_progress(pipeline.name)
        if progress:
            print(f"  {progress['completed_steps']}/{progress['total_steps']} steps: {progress['status']}")
        time.sleep(2)
    ```

    在执行期间轮询 `get_progress()` 获取实时状态。
  </Tab>
</Tabs>

<a id="pipeline-dsl"></a>
## 流水线 DSL

`PipelineBuilder` 使用 `add_step(name, type, **config)` 和 `connect_steps(from, to)` 来定义 DAG：

```python
from semantica.pipeline import PipelineBuilder, ExecutionEngine

builder = PipelineBuilder()

# 添加步骤：step_type 是字符串标签，handler 是运行时调用的可调用对象
builder.add_step("ingest",      "file_ingest",    handler=ingestor.ingest_file)
builder.add_step("parse",       "document_parse", handler=parser.parse)
builder.add_step("normalize",   "text_normalize", handler=normalizer.normalize)
builder.add_step("extract",     "ner_extract",    handler=extractor.extract)
builder.add_step("rel_extract", "rel_extract",    handler=rel_extractor.extract)
builder.add_step("build_kg",    "graph_build",    handler=kg_builder.build)
builder.add_step("deduplicate", "dedup",          handler=deduplicator.deduplicate)
builder.add_step("export",      "rdf_export",     handler=exporter.export, format="turtle", path="output.ttl")

# 串联数据流
builder.connect_steps("ingest",      "parse")
builder.connect_steps("parse",       "normalize")
builder.connect_steps("normalize",   "extract")
builder.connect_steps("extract",     "rel_extract")
builder.connect_steps("rel_extract", "build_kg")
builder.connect_steps("build_kg",    "deduplicate")
builder.connect_steps("deduplicate", "export")

pipeline = builder.build("full_pipeline")
result   = ExecutionEngine().execute_pipeline(pipeline, data="data/")
```

<a id="serialize-and-restore-pipelines"></a>
## 序列化和恢复流水线

`PipelineSerializer` 将流水线转换为 JSON 或字典以进行存储，稍后可重新加载：

```python
from semantica.pipeline import PipelineSerializer

serializer = PipelineSerializer()

# 序列化为 JSON 字符串
json_str = serializer.serialize_pipeline(pipeline, format="json")

# 保存到文件
with open("pipeline_config.json", "w") as f:
    f.write(json_str)

# 在任何机器上恢复并执行
with open("pipeline_config.json") as f:
    restored = serializer.deserialize_pipeline(f.read())

result = ExecutionEngine().execute_pipeline(restored, data="data/")
```

<Tip>
  序列化的流水线捕获步骤名称、类型和配置：但不包括处理程序函数（可调用对象无法序列化）。在执行前在恢复的步骤上重新注册处理程序。
</Tip>

<a id="pre-built-templates"></a>
## 预置模板

`PipelineTemplateManager` 以正确的步骤顺序串联常见工作流：无需手动串联：

```python
from semantica.pipeline import PipelineTemplateManager

manager = PipelineTemplateManager()
```

`create_pipeline_from_template(name)` 方法返回一个已配置的 `PipelineBuilder`。对其调用 `.build(pipeline_name)` 以生成可运行的 `Pipeline`。

- **document_processing** — **摄取 → 解析 → 归一化 → 抽取 → 嵌入 → 构建 KG** — 从摄取到知识图谱的完整文档处理。

  ```python
  builder  = manager.create_pipeline_from_template("document_processing")
  pipeline = builder.build("doc_pipeline")
  ```

- **rag_pipeline** — **摄取 → 分块 → 嵌入 → 存储向量** — 用于问答的 RAG 流水线：构建向量索引的存储。

  ```python
  builder  = manager.create_pipeline_from_template("rag_pipeline")
  pipeline = builder.build("rag_pipeline")
  ```

- **kg_construction** — **摄取 → 抽取实体 → 抽取关系 → 去重 → 解析 → 构建图** — 从多个来源构建知识图谱。

  ```python
  builder  = manager.create_pipeline_from_template("kg_construction")
  pipeline = builder.build("kg_pipeline")
  ```

- **ontology_generation** — **抽取概念 → 推断类 → 推断属性 → 生成 OWL → 验证** — 从抽取数据生成本体。

  ```python
  builder  = manager.create_pipeline_from_template("ontology_generation")
  pipeline = builder.build("ontology_pipeline")
  ```

<Tip>
  **对常见模式使用 `PipelineTemplateManager` 的模板。** `create_pipeline_from_template("kg_construction")` 以正确顺序串联归一化、去重、冲突检测和图构建：避免常见错误，如先去重再归一化。
</Tip>

<a id="executionengine"></a>
## ExecutionEngine

对流水线执行的精细控制：暂停、恢复、取消和检查实时进度：

```python
from semantica.pipeline import ExecutionEngine

engine = ExecutionEngine(max_workers=4)

# pipeline.name 是用于所有控制操作的流水线 ID
result = engine.execute_pipeline(pipeline, data="data/")

pipeline_id = pipeline.name   # 例如 "my_pipeline"

# 在当前步骤完成后暂停
engine.pause_pipeline(pipeline_id)

progress = engine.get_progress(pipeline_id)
print(f"Completed: {progress['completed_steps']}/{progress['total_steps']}")
print(f"Status: {progress['status']}")

engine.resume_pipeline(pipeline_id)
engine.stop_pipeline(pipeline_id)
```

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `execute_pipeline(pipeline, data)` | `ExecutionResult` | 从头到尾执行流水线 |
| `get_pipeline_status(pipeline_id)` | `PipelineStatus` | 当前状态（RUNNING、PAUSED、STOPPED） |
| `get_progress(pipeline_id)` | `Dict` | `completed_steps`、`total_steps`、`progress_percentage`、`status` |
| `pause_pipeline(pipeline_id)` | `None` | 在当前步骤完成后挂起 |
| `resume_pipeline(pipeline_id)` | `None` | 从暂停状态恢复 |
| `stop_pipeline(pipeline_id)` | `None` | 立即取消并清理 |

<a id="pipelinevalidator"></a>
## PipelineValidator

在问题表现为运行中失败之前捕获它们：

```python
from semantica.pipeline import PipelineValidator

validator = PipelineValidator()
result    = validator.validate_pipeline(pipeline)

if result.valid:
    print("Pipeline is valid: safe to run")
else:
    for error in result.errors:     # errors 是 List[str]
        print(f"Error: {error}")
    for warning in result.warnings: # warnings 是 List[str]
        print(f"Warning: {warning}")
```

执行的检查：
- **依赖循环检测**：A 依赖于 B，B 依赖于 A
- **步骤类型验证**：每个步骤类型必须已注册
- **连接完整性**：引用的步骤名称必须存在
- **配置完整性**：必须存在必需参数

<a id="parallelismmanager"></a>
## ParallelismManager

<Tabs>
  <Tab title="线程池（I/O 密集型）">
    ```python
    from semantica.pipeline import ParallelismManager, Task

    # use_processes=False（默认）→ 用于 I/O 密集型任务的线程池
    manager = ParallelismManager(max_workers=8, use_processes=False)

    tasks   = [Task(task_id=f"t{i}", handler=ner.extract, args=(text,)) for i, text in enumerate(texts)]
    results = manager.execute_parallel(tasks)
    # 返回 List[ParallelExecutionResult]

    successes = [r for r in results if r.success]
    failures  = [r for r in results if not r.success]
    ```

    对 **I/O 密集型** 步骤使用线程池：Web 抓取、数据库查询、API 调用。
  </Tab>
  <Tab title="进程池（CPU 密集型）">
    ```python
    from semantica.pipeline import ParallelismManager, Task

    # use_processes=True → 进程池，绕过 Python GIL
    manager = ParallelismManager(max_workers=4, use_processes=True)

    tasks   = [Task(task_id=f"t{i}", handler=embedder.generate_embeddings, args=(chunk,)) for i, chunk in enumerate(chunks)]
    results = manager.execute_parallel(tasks)
    ```

    对 **CPU 密集型** 步骤使用进程池：嵌入、OCR、大型 NER 批处理。
  </Tab>
</Tabs>

<a id="resourcescheduler"></a>
## ResourceScheduler

防止大型运行时的内存过度订阅：

```python
from semantica.pipeline import ResourceScheduler, ExecutionEngine

scheduler = ResourceScheduler()
engine    = ExecutionEngine()

resources = scheduler.allocate_resources(pipeline)

try:
    result = engine.execute_pipeline(pipeline, data="data/")
finally:
    scheduler.release_resources(resources)
```

<a id="delta-mode"></a>
## 增量模式

仅重新处理自上次运行以来已更改的数据：

```python
from semantica.pipeline import PipelineBuilder, ExecutionEngine

builder = PipelineBuilder()

# delta_mode=True 告诉 ExecutionEngine 计算两个快照之间的差异，
# 并仅将更改的三元组传给此步骤的处理程序
builder.add_step(
    "ingest",  "file_ingest",
    handler=ingestor.ingest_file,
    delta_mode=True, base_version_id="v1", target_version_id="v2"
)
builder.add_step(
    "extract", "ner_extract",
    handler=extractor.extract,
    delta_mode=True, base_version_id="v1", target_version_id="v2"
)
builder.add_step(
    "build", "graph_build",
    handler=kg_builder.build,
    delta_mode=False  # 始终重建合并后的图
)
builder.connect_steps("ingest", "extract")
builder.connect_steps("extract", "build")

pipeline = builder.build("delta_pipeline")
engine   = ExecutionEngine()
result   = engine.execute_pipeline(
    pipeline,
    data="data/",
    version_manager=version_manager,   # 增量模式必需
    triplet_store=triplet_store        # 增量模式必需
)
```

<Note>
  增量检测对源内容使用 SHA-256 校验和。仅校验和与 `base_version_id` 不同的来源才会传给下游步骤。对于每小时或每天针对不断增长的语料库运行的流水线，增量模式消除了冗余的重新嵌入和重新抽取。
</Note>

<a id="sparql-construct-template-steps"></a>
## SPARQL CONSTRUCT 模板步骤

使用 `"construct_template"` 步骤类型来渲染并执行一个 [SPARQL CONSTRUCT 模板](triplet_store.zh-CN.md#sparql-construct-templates)，作为流水线的一部分。`store_backend` 和 `construct_template_registry` 是执行时资源，而不是步骤配置 — 将它们传给 `execute_pipeline()`，与 `delta_mode` 步骤接收 `version_manager` 和 `triplet_store` 的方式相同：

```python
from semantica.pipeline import PipelineBuilder, ExecutionEngine
from semantica.triplet_store.construct_templates import construct_template_step_handler

builder = PipelineBuilder()
builder.add_step(
    "apply_person_template",
    "construct_template",
    handler=construct_template_step_handler,
    template_name="person_to_foaf",
    params={"subject": "http://ex.org/p1", "name": "Alice", "age": 30},
    target_graph="http://ex.org/graphs/people",
)
pipeline = builder.build("person_pipeline")

engine = ExecutionEngine()
result = engine.execute_pipeline(
    pipeline,
    data=None,
    store_backend=store,                   # 必需：一个 BlazegraphStore 实例
    construct_template_registry=registry,  # 必需：持有已注册的模板
)

triplets = result.output   # List[Triplet]，已通过 store.add_triplets 持久化
```

<Note>
  如果 `store_backend` 或 `construct_template_registry` 缺失于 `execute_pipeline()` 的选项中，`construct_template` 步骤会抛出 `ProcessingError`；如果 `template_name` 未注册，则抛出 `ValidationError`。
</Note>

<a id="schemas"></a>
## 模式

<AccordionGroup>
  <Accordion title="ExecutionResult 模式">

```python
@dataclass
class ExecutionResult:
    success:  bool            # 如果所有步骤完成且无故障则为 True
    output:   Any             # 来自最终流水线步骤的输出
    metadata: Dict[str, Any]  # {"pipeline_id": "...", "execution_time": 1.23}
    metrics:  Dict[str, Any]  # {"steps_executed": 4, "steps_failed": 0, "execution_time": 1.23}
    errors:   List[str]       # 来自失败步骤的错误消息（完全成功时为空）

# 访问模式
result.success                       # bool
result.output                        # 最终步骤输出
result.metadata["pipeline_id"]       # 用作 ID 的流水线名称
result.metadata["execution_time"]    # 总挂钟时间秒数
result.metrics["steps_executed"]     # 成功完成的步骤计数
result.metrics["steps_failed"]       # 失败的步骤计数
result.errors                        # List[str] 错误消息
```

  </Accordion>
  <Accordion title="PipelineStep 模式">

```python
@dataclass
class PipelineStep:
    name:              str
    step_type:         str
    config:            Dict[str, Any]
    dependencies:      List[str]          # 此步骤等待的步骤名称
    handler:           Optional[Callable]
    status:            StepStatus
    result:            Any
    error:             Optional[Exception]
    delta_mode:        bool               # True = 仅处理已更改的数据
    base_version_id:   Optional[str]     # 要与之比较的快照 ID
    target_version_id: Optional[str]     # 正在生成的快照 ID
```

  </Accordion>
  <Accordion title="StepStatus 枚举">

```python
from semantica.pipeline import StepStatus

StepStatus.PENDING    # 尚未开始
StepStatus.RUNNING    # 正在执行
StepStatus.COMPLETED  # 成功完成
StepStatus.FAILED     # 发生错误：检查 step.error
StepStatus.SKIPPED    # 由于 FailureHandler 的 "skip" 策略而跳过
```

  </Accordion>
</AccordionGroup>

- [摄取](ingest.zh-CN.md) — 大多数流水线的第一步。
- [语义抽取](semantic_extract.zh-CN.md) — 核心抽取步骤。
- [知识图谱](kg.zh-CN.md) — 图构建步骤。
- [导出](export.zh-CN.md) — 最终输出步骤。
