---
title: "摄取模块"
description: "从文件、Parquet、XML、网页、公共 API、信息流、仓库、邮件和数据库进行通用数据摄取。"
icon: "database"
---

**[English](ingest.md)** · **简体中文（当前）**

**`semantica.ingest`** 是将数据加载到 Semantica 的**通用入口**：

- 15+ 摄取适配器：文件、网页、SQL、Databricks、Snowflake、Kafka、MCP、Git 仓库、邮件
- 基于 PyArrow 的 Parquet，支持列选择与分区数据集
- XXE 安全的 lxml XML，支持可选的 XSD 模式校验
- `ingest()` 统一分发器：根据路径或 URL 自动检测来源类型
- 每个摄取器返回自己的类型化对象（`FileObject`、`WebContent`、`TableData` 等）


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `FileIngestor` | PDF、DOCX、HTML、JSON、CSV、Excel、PPTX、ZIP/TAR：根据扩展名自动检测类型 |
| `CloudStorageIngestor` | AWS S3、Google Cloud Storage 与 Azure Blob Storage 的统一客户端 |
| `WebIngestor` | 网页抓取与爬取，提供 `ingest_url`、`crawl_sitemap`、`crawl_domain` |
| `RESTIngestor` | 通用 REST API 摄取，支持 headers、params、重试与分页 |
| `PublicAPIIngestor` | 无鉴权的公共 API 摄取，内置预配置示例与速率限制 |
| `FeedIngestor` | RSS/Atom 信息流摄取，可通过 `FeedMonitor` 进行实时监控 |
| `StreamIngestor` | 从 Kafka、RabbitMQ、AWS Kinesis 与 Apache Pulsar 进行实时摄取 |
| `RepoIngestor` | Git 仓库：源码文件、提交历史与元数据 |
| `DBIngestor` | 通过 SQLAlchemy 访问 SQL 数据库：表、视图与自定义查询 |
| `SnowflakeIngestor` | Snowflake 数据仓库查询与表导出 |
| `DatabricksIngestor` | Databricks Unity Catalog 元数据、Delta 表查询与血缘 |
| `ParquetIngestor` | Apache Parquet 文件与分区数据集，支持列选择 |
| `ArrowIngestor` | Apache Arrow IPC 与 Feather 文件处理 |
| `XMLIngestor` | XXE 安全的 XML 解析，支持可选的 XSD 模式校验 |
| `EmailIngestor` | IMAP/POP3 邮件摄取，支持附件抽取 |
| `OntologyIngestor` | OWL/RDF/Turtle 本体文件摄取 |
| `MCPIngestor` | Model Context Protocol (MCP) 资源摄取 |
| `ingest()` | 统一分发器：根据路径或 URL 自动检测来源类型 |

<a id="getting-started"></a>
## 快速上手

对本地文件使用 **`FileIngestor`**：它会根据文件扩展名**自动检测格式**，并处理归档文件：

```python
from semantica.ingest import FileIngestor

ingestor = FileIngestor()

# 单个文件 -> FileObject
file_obj = ingestor.ingest_file("data/report.pdf")
print(file_obj.name)       # "report.pdf"
print(file_obj.file_type)  # "pdf"
print(file_obj.text)       # 解码后的文本内容（FileObject 的属性）
print(file_obj.size)       # 字节数

# 目录扫描 -> List[FileObject]
files = ingestor.ingest_directory("data/", recursive=True)
for f in files:
    print(f.name, f.file_type, f.size)
```

<Tip>
  **对于本地文件，`FileIngestor` 始终是最快的路径。** 它根据扩展名自动检测格式，自动处理 ZIP/TAR 归档，并将内容读入 `.content` 字节或 `.text` 属性。当你只需要文件元数据时，使用 `read_content=False`。
</Tip>

对于网页、数据库或流式数据源，每个摄取器都暴露自己类型化的方法：

```python
# 网页
from semantica.ingest import WebIngestor
wc = WebIngestor(delay=1.0, respect_robots=True).ingest_url("https://example.com")
print(wc.title, wc.text)

# 数据库：构造函数无必填参数；将 connection_string 传入各方法
from semantica.ingest import DBIngestor
db = DBIngestor()
result = db.ingest_database("postgresql://user:pass@localhost/db")
# result["tables"]["documents"]["rows"] 包含数据行

# 统一分发器：自动检测来源类型
from semantica.ingest import ingest
result = ingest("data/report.pdf")          # -> {"files": [FileObject]}
result = ingest("https://example.com")      # -> {"content": WebContent}
result = ingest("data/events.parquet")      # -> {"data": ParquetData}
result = ingest("ontology.ttl")             # -> {"ontology": OntologyData}
```

<a id="quick-start"></a>
## 快速启动

<Steps>
  <Step title="摄取本地文件">
    ```python
    from semantica.ingest import FileIngestor

    ingestor = FileIngestor()

    # 单个文件：根据扩展名自动检测类型
    file_obj = ingestor.ingest_file("data/report.pdf")

    # 递归扫描目录
    files = ingestor.ingest_directory("data/", recursive=True)

    # ingest() 也可使用：自动路由到文件或目录
    from semantica.ingest import ingest
    result = ingest("data/report.pdf")   # {"files": [FileObject]}
    ```
  </Step>
  <Step title="连接到数据库">
    ```python
    from semantica.ingest import DBIngestor

    ingestor = DBIngestor()

    # 摄取所有表
    result = ingestor.ingest_database(
        "postgresql://user:pass@localhost/db",
        include_tables=["documents"],
    )
    # result["tables"]["documents"]["rows"] 包含数据行字典

    # 运行自定义查询
    rows = ingestor.execute_query(
        "postgresql://user:pass@localhost/db",
        "SELECT id, content, created_at FROM documents WHERE status = :s",
        s="active",
    )
    ```
  </Step>
  <Step title="接入流水线">
    ```python
    from semantica.ingest import FileIngestor
    from semantica.pipeline import PipelineBuilder, ExecutionEngine
    from semantica.parse import DocumentParser
    from semantica.semantic_extract import NERExtractor

    ingestor  = FileIngestor()
    parser    = DocumentParser()
    extractor = NERExtractor(method="ml")

    builder = PipelineBuilder()
    builder.add_step("ingest",  "file_ingest",    handler=ingestor.ingest_file)
    builder.add_step("parse",   "document_parse", handler=parser.parse)
    builder.add_step("extract", "ner_extract",    handler=extractor.extract)
    builder.connect_steps("ingest", "parse")
    builder.connect_steps("parse",  "extract")

    pipeline = builder.build("my_pipeline")
    result   = ExecutionEngine().execute_pipeline(pipeline, data="data/report.pdf")
    ```
  </Step>
</Steps>

<a id="ingestors"></a>
## 摄取器

<Tabs>
  <Tab title="基于文件">
    <a id="fileingestor"></a>
    ### FileIngestor

    ```python
    from semantica.ingest import FileIngestor

    ingestor = FileIngestor()

    # 单个文件
    file_obj = ingestor.ingest_file("data/report.pdf")

    # 目录：返回 List[FileObject]
    files = ingestor.ingest_directory("data/", recursive=True)

    # ingest() 会自动分发到 ingest_file 或 ingest_directory
    files = ingestor.ingest("data/")
    ```

    支持的格式：PDF、DOCX、TXT、HTML、JSON、CSV、Excel（XLSX/XLS）、PPTX、ZIP/TAR 归档。

    <Note>
      不支持 glob 模式（例如 `"data/**/*.docx"`）。`ingest()` 只接受文件路径或目录路径。要在目录内按扩展名过滤，请使用带 `pattern=` 过滤选项的 `ingest_directory()`。
    </Note>

    <a id="parquetingestor"></a>
    ### ParquetIngestor

    基于 PyArrow 的 Apache Parquet 文件摄取，支持 Hive 风格分区数据集：

    ```python
    from semantica.ingest import ParquetIngestor

    ingestor = ParquetIngestor()

    # 单个 Parquet 文件 -> ParquetData
    data = ingestor.ingest_file("data/events.parquet")

    # 分区目录（year=2024/month=01/...）
    data = ingestor.ingest_directory("data/partitioned/")

    # 仅加载特定列：作为 kwarg 传入
    from semantica.ingest import ingest_parquet
    data = ingest_parquet("data/events.parquet", columns=["id", "text", "timestamp"])

    # 仅抽取模式而不加载数据
    schema = ingest_parquet("data/events.parquet", method="schema")
    ```

    依赖 `pyarrow`：`pip install pyarrow`。

    <Tip>
      **对于结构化分析数据，请使用 `ParquetIngestor` 而非 `FileIngestor`。** Parquet 摄取会保留 CSV 读取会丢失的列类型（int、float、datetime）。使用 `columns=["id", "text"]` 可避免加载未使用的列：这对拥有数百列的宽表至关重要。
    </Tip>

    <a id="xmlingestor"></a>
    ### XMLIngestor

    基于 XXE 安全的 lxml 摄取，支持可选的模式校验：

    ```python
    from semantica.ingest import XMLIngestor

    # 基本摄取
    ingestor = XMLIngestor()
    data = ingestor.ingest_file("data/records.xml")

    # 启用 XSD 校验：将 schema_path 作为 kwarg 传入
    from semantica.ingest import ingest_xml
    data = ingest_xml("data/records.xml", schema_path="schema.xsd")

    # 仅输出校验报告
    report = ingest_xml("data/feed.xml", method="validate", schema_path="schema.xsd")

    # 目录扫描
    results = ingestor.ingest_directory("data/records/")
    ```

    <Note>
      `XMLIngestor` 使用 lxml 并设置 `resolve_entities=False`，以防止 XML External Entity (XXE) 注入攻击。
    </Note>

    <Warning>
      **`XMLIngestor` 默认是 XXE 安全的。** 不要在传入 Semantica 之前使用标准库 `xml.etree.ElementTree` 预解析 XML：它无法阻止 XXE 攻击。`XMLIngestor` 使用 lxml 并设置 `resolve_entities=False`，可安全解析不可信 XML。
    </Warning>
  </Tab>
  <Tab title="网页与信息流">
    <a id="webingestor"></a>
    ### WebIngestor

    ```python
    from semantica.ingest import WebIngestor

    ingestor = WebIngestor(
        delay=1.0,             # 请求间隔秒数
        respect_robots=True,   # 遵守 robots.txt
        timeout=30,
    )

    # 单个 URL -> WebContent
    content = ingestor.ingest_url("https://example.com/about")
    print(content.title)
    print(content.text)
    print(content.links)

    # 站点地图爬取 -> List[WebContent]
    pages = ingestor.crawl_sitemap("https://example.com/sitemap.xml")

    # 域名爬取 -> List[WebContent]
    pages = ingestor.crawl_domain("https://example.com", max_pages=50)
    ```

    依赖 `beautifulsoup4`：`pip install beautifulsoup4`。

    <Tip>
      **对网页爬取做速率限制。** `WebIngestor(delay=1.0, respect_robots=True)` 是负责任的默认配置。不加速率限制，你可能会被目标服务器封禁或违反其服务条款。
    </Tip>

    <a id="publicapiingestor"></a>
    ### PublicAPIIngestor

    用于不需要密钥或令牌的公共 REST 风格 API：

    ```python
    from semantica.ingest import PublicAPIIngestor, PublicAPIExamples, ingest_public_api

    ingestor = PublicAPIIngestor(rate_limit_delay=1.0)

    # 摄取任意公共端点
    data = ingestor.ingest_public_api("https://jsonplaceholder.typicode.com/posts")

    # 按名称使用预配置示例
    data = ingestor.ingest_example("rest_countries_all")

    # 检测端点是否可在无鉴权情况下访问
    detection = ingestor.detect_public_api("https://jsonplaceholder.typicode.com/posts")

    # 列出可用的预配置示例
    examples = PublicAPIExamples.list_examples()

    # 便捷函数
    data = ingest_public_api("https://jsonplaceholder.typicode.com/posts")
    ```

    公共 API 摄取默认会拒绝常见的鉴权头与查询参数。需要鉴权的 API 请使用 `RESTIngestor`。

    <a id="feedingestor-rssatom"></a>
    ### FeedIngestor（RSS/Atom）

    ```python
    from semantica.ingest import FeedIngestor

    ingestor = FeedIngestor()

    # 摄取一个信息流 -> FeedData
    feed = ingestor.ingest_feed("https://feeds.example.com/rss")

    # 从某网站发现信息流
    from semantica.ingest import ingest_feed
    feeds = ingest_feed("https://example.com", method="discover")
    ```

    依赖 `beautifulsoup4`：`pip install beautifulsoup4`。

    <a id="repoingestor"></a>
    ### RepoIngestor

    摄取 Git 仓库：源代码、提交历史与依赖图：

    ```python
    from semantica.ingest import RepoIngestor

    ingestor = RepoIngestor(
        branch="main",
        file_types=[".py", ".md", ".yaml"],
        include_commits=True,
        commit_range="HEAD~100..HEAD",
    )

    result = ingestor.ingest_repository("https://github.com/org/repo")
    result = ingestor.ingest_repository("/path/to/local/repo")
    ```

    依赖 `GitPython`：`pip install gitpython`。

    <a id="emailingestor"></a>
    ### EmailIngestor

    通过 IMAP 或 POP3 摄取邮件，支持附件抽取与会话分析：

    ```python
    from semantica.ingest import EmailIngestor
    import os

    ingestor = EmailIngestor(
        protocol="imap",
        host="imap.gmail.com",
        port=993,
        use_ssl=True,
        username=os.getenv("EMAIL_USER"),
        password=os.getenv("EMAIL_PASS"),
        folder="INBOX",
        attachment_types=[".pdf", ".docx", ".txt"],
        include_thread_analysis=True,
        max_emails=500,
    )
    emails = ingestor.ingest()
    ```

    依赖 `beautifulsoup4`：`pip install beautifulsoup4`。
  </Tab>
  <Tab title="云存储">
    <a id="cloudstorageingestor"></a>
    ### CloudStorageIngestor

    `CloudStorageIngestor` 是 AWS S3、Google Cloud Storage 与 Azure Blob Storage 的统一客户端：

    ```python
    from semantica.ingest import CloudStorageIngestor
    import os

    # AWS S3：列出并下载对象
    ingestor = CloudStorageIngestor(
        provider="s3",
        access_key_id=os.getenv("AWS_ACCESS_KEY_ID"),
        secret_access_key=os.getenv("AWS_SECRET_ACCESS_KEY"),
        region="us-east-1",
    )
    objects = ingestor.list_objects("my-documents-bucket", prefix="reports/2024/")
    content = ingestor.download_object("my-documents-bucket", "reports/2024/report.pdf")

    # FileIngestor.ingest_cloud() 封装了 CloudStorageIngestor
    from semantica.ingest import FileIngestor
    files = FileIngestor().ingest_cloud(
        provider="s3",
        bucket="my-documents-bucket",
        prefix="reports/2024/",
        access_key_id=os.getenv("AWS_ACCESS_KEY_ID"),
        secret_access_key=os.getenv("AWS_SECRET_ACCESS_KEY"),
        region="us-east-1",
    )
    ```
  </Tab>
  <Tab title="数据库">
    <a id="dbingestor-sql"></a>
    ### DBIngestor（SQL）

    `DBIngestor` 的构造函数没有必填参数。将连接字符串传入每个方法：

    ```python
    from semantica.ingest import DBIngestor

    ingestor = DBIngestor()

    # 摄取整个数据库（所有表，或按条件过滤）
    result = ingestor.ingest_database(
        "postgresql://user:pass@localhost/db",
        include_tables=["documents"],
    )
    # result["schema"], result["tables"], result["total_tables"]

    # 运行自定义查询 -> List[Dict]
    rows = ingestor.execute_query(
        "postgresql://user:pass@localhost/db",
        "SELECT id, content FROM documents WHERE status = :s",
        s="active",
    )

    # 导出单张表 -> TableData
    table = ingestor.export_table(
        "postgresql://user:pass@localhost/db",
        table_name="documents",
        limit=1000,
    )
    ```

    依赖 `sqlalchemy`：`pip install sqlalchemy`，再加上你的数据库驱动。

    <Warning>
      **`DBIngestor()` 的构造函数不接收连接字符串。** 将连接字符串作为第一个位置参数传给 `ingest_database()`、`execute_query()` 或 `export_table()`，而不要传给 `DBIngestor()` 本身。
    </Warning>

    <a id="snowflakeingestor"></a>
    ### SnowflakeIngestor

    ```python
    from semantica.ingest import SnowflakeIngestor
    import os

    ingestor = SnowflakeIngestor(
        account=os.getenv("SNOWFLAKE_ACCOUNT"),
        user=os.getenv("SNOWFLAKE_USER"),
        password=os.getenv("SNOWFLAKE_PASSWORD"),
        warehouse="COMPUTE_WH",
        database="ANALYTICS",
        schema="PUBLIC",
    )
    result = ingestor.ingest_query("SELECT * FROM documents")
    result = ingestor.ingest_table("documents")
    ```

    <a id="databricksingestor"></a>
    ### DatabricksIngestor

    ```python
    from semantica.ingest import DatabricksIngestor
    import os

    ingestor = DatabricksIngestor(
        host=os.getenv("DATABRICKS_HOST"),
        token=os.getenv("DATABRICKS_TOKEN"),
        http_path=os.getenv("DATABRICKS_HTTP_PATH"),
        catalog="main",
        schema="default",
    )
    result = ingestor.ingest_query("SELECT * FROM documents")
    result = ingestor.ingest_table("documents")
    lineage = ingestor.get_table_lineage("documents")
    ```
  </Tab>
  <Tab title="流">
    <a id="streamingestor"></a>
    ### StreamIngestor

    从消息 broker 进行实时摄取：每个方法返回一个类型化的处理器：

    ```python
    from semantica.ingest import StreamIngestor

    ingestor = StreamIngestor()

    # Kafka -> KafkaProcessor
    processor = ingestor.ingest_kafka(
        topic="documents",
        bootstrap_servers=["localhost:9092"],
    )
    processor.set_message_handler(lambda msg: print(msg))
    processor.start_consuming()

    # RabbitMQ -> RabbitMQProcessor
    processor = ingestor.ingest_rabbitmq(
        queue="document_queue",
        connection_url="amqp://guest:guest@localhost/",
    )

    # AWS Kinesis -> KinesisProcessor
    processor = ingestor.ingest_kinesis(
        stream_name="documents-stream",
        region="us-east-1",
    )

    # Apache Pulsar -> PulsarProcessor
    processor = ingestor.ingest_pulsar(
        topic="persistent://public/default/documents",
        service_url="pulsar://localhost:6650",
    )

    # 一次性启动所有处理器
    ingestor.start_streaming()
    # 停止所有处理器
    ingestor.stop_streaming()

    # 监控流健康状态
    health = ingestor.monitor.check_health()
    ```

    流处理器需要相应的客户端库（kafka-python、pika、boto3、pulsar-client）。

    <Warning>
      **`StreamIngestor` 的各方法需要安装目标 broker 的客户端库。** `ingest_kafka` 需要 `kafka-python`，`ingest_rabbitmq` 需要 `pika`，`ingest_kinesis` 需要 `boto3`，`ingest_pulsar` 需要 `pulsar-client`。缺失依赖会在调用时（而非导入时）抛出 `ImportError`。
    </Warning>
  </Tab>
</Tabs>

<a id="ingest-unified-dispatcher"></a>
## `ingest()` 统一分发器

`ingest()` 根据路径或 URL 自动检测来源类型，并路由到对应的摄取器。它返回一个 `Dict[str, Any]`，其中键取决于来源类型：

```python
from semantica.ingest import ingest

# 文件
result = ingest("report.pdf")               # {"files": [FileObject]}
result = ingest("data/", source_type="file") # {"files": [FileObject, ...]}

# 网页
result = ingest("https://example.com")       # {"content": WebContent}

# 信息流（根据 URL 模式自动检测）
result = ingest("https://example.com/feed.xml") # {"feeds": FeedData}

# Parquet（根据 .parquet 扩展名自动检测）
result = ingest("events.parquet")            # {"data": ParquetData}

# XML（根据 .xml 扩展名自动检测）
result = ingest("records.xml")               # {"xml": XMLIngestionData}

# 本体（根据 .ttl/.owl/.rdf 自动检测）
result = ingest("ontology.ttl")              # {"ontology": OntologyData}

# 数据库（根据连接字符串前缀自动检测）
result = ingest("postgresql://user:pass@localhost/db") # {"data": ...}

# 公共 API
result = ingest(
    "https://jsonplaceholder.typicode.com/posts",
    source_type="public_api",
)                                            # {"data": APIData}
```

<a id="ingest-parameters"></a>
### `ingest()` 参数

| 参数 | 类型 | 默认值 | 描述 |
| :--------- | :---- | :------- | :----------- |
| `sources` | `str`、`Path` 或 `List` | **必填** | 文件路径、URL、目录或连接字符串 |
| `source_type` | `str` | `None`（自动检测） | `"file"`、`"web"`、`"public_api"`、`"feed"`、`"stream"`、`"repo"`、`"email"`、`"db"`、`"parquet"`、`"xml"`、`"ontology"`、`"mcp"` |
| `method` | `str` | `None` | 可选的方法覆盖，传递给底层摄取器 |
| `**kwargs` | | | 转发给底层摄取器方法的额外选项 |

<a id="fileobject-fields"></a>
## FileObject 字段

`FileIngestor` 返回 `FileObject` 实例：

<Accordion title="FileObject 模式">

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Any, Dict, Optional

@dataclass
class FileObject:
    path:        str                    # 绝对文件路径
    name:        str                    # 文件名（例如 "report.pdf"）
    size:        int                    # 字节大小
    file_type:   str                    # 检测到的类型（无点，例如 "pdf"、"docx"）
    mime_type:   Optional[str]          # 可检测到时的 MIME 类型
    content:     Optional[bytes]        # 原始字节（read_content=False 时为 None）
    metadata:    Dict[str, Any]         # 扩展名、父目录、is_supported 等
    ingested_at: datetime               # 摄取时间戳

    @property
    def text(self) -> str:
        """从 content 字节解码得到的文本（UTF-8，回退到 latin-1）。"""
        ...
```

要从已摄取文件中获取文本，使用 `.text` 属性：

```python
file_obj = FileIngestor().ingest_file("report.pdf")
text = file_obj.text       # 解码后的字符串
raw  = file_obj.content    # 原始字节
```

跳过读取内容（适用于只扫描目录而不加载文件）：

```python
files = FileIngestor().ingest_directory("data/", recursive=True, read_content=False)
```

</Accordion>

<a id="ontologyingestor"></a>
## OntologyIngestor

将已有的 OWL 或 RDF 本体文件作为结构化知识源摄取：

```python
from semantica.ingest import OntologyIngestor

ingestor = OntologyIngestor()

data = ingestor.ingest_ontology("domain_ontology.owl", format="turtle")

# 或使用便捷函数
from semantica.ingest import ingest_ontology
data = ingest_ontology("domain_ontology.ttl")
```

<a id="custom-ingestors"></a>
## 自定义摄取器

注册一个自定义摄取器函数，即可参与完整的注册表：

```python
from semantica.ingest.registry import method_registry
from semantica.ingest import FileObject

def my_ingestor(source, **kwargs):
    # 返回你的格式所产生的任何内容
    return FileObject(
        path=source,
        name=source,
        size=0,
        file_type="custom",
        content=b"...",
        metadata={},
    )

method_registry.register("file", "my_format", my_ingestor)

# 现在可通过便捷函数调用
from semantica.ingest import ingest_file
result = ingest_file("source_path", method="my_format")
```

- [解析](parse.zh-CN.md) —— 将原始来源解析为结构化文本与表格。
- [流水线](pipeline.zh-CN.md) —— 将摄取编排为流水线的第一步。
- [Snowflake 集成](../integrations/snowflake.zh-CN.md) —— Snowflake 专属的配置与鉴权指南。
- [Databricks 集成](../integrations/databricks.zh-CN.md) —— Databricks Unity Catalog 的配置、鉴权与血缘指南。
- [溯源](provenance.zh-CN.md) —— 跟踪从摄取到推理的血缘链路。
