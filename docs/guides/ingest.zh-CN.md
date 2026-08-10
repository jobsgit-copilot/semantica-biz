---
title: "摄取一切"
description: "从文件、网站、Git 仓库、数据库、Kafka 流、RSS 源等加载数据——统一摄取 API。"
---

**[English](ingest.md)** · **简体中文（当前）**

Semantica 的摄取模块为每种源类型提供单一一致的函数调用。文件、网页和订阅源返回带 `.text` 属性的对象。API 和数据库源返回结构化对象（`APIData.data`、`List[Dict]`），你需要在存储之前把它们转换成文本。Git 仓库源返回一个包含 `code_files` 和 `commits` 键的字典；流源返回带 `.content` 字段的 `StreamMessage` 对象。它们全部都能干净地组合进一个图谱摄取脚本——具体访问模式见各源章节。

<Info>
  所有摄取函数都位于 `semantica.ingest`。可选依赖会被惰性加载——你只需要为网页/订阅源摄取 `pip install beautifulsoup4`，为仓库摄取 `pip install gitpython`，为 Parquet 安装 `pip install pyarrow`。缺少依赖时会抛出清晰的 `ImportError` 消息，指明具体的包名。
</Info>

<a id="why-ingestion-matters"></a>
## 为什么摄取很重要

在 Semantica 能够分析、搜索、推理或连接你的数据之前，它需要先触达数据。摄取就是这第一步——从数据所在之处拉取内容，并将其转换成 `AgentContext` 可以存储和索引的形式。

一旦摄取完成，你的内容会流入两个地方：一个用于语义搜索的向量索引，以及一个可选的用于实体关系的 `ContextGraph`。从那里开始，每一个下游模块——语义抽取、推理、GraphRAG、决策智能——都基于同一份统一数据运作，无论数据最初来自 PDF、数据库行、API 响应，还是实时流。

<a id="typical-workflow"></a>
## 典型工作流

大多数摄取流水线无论源类型如何，都遵循同样的四个步骤：

1. **连接到源**——用合适的摄取函数和你的连接详情进行调用。
2. **检索数据**——函数返回带文本的对象（文件、网页、订阅源）或需要再处理一步的结构化数据（API、数据库）。
3. **必要时转换**——对于结构化源，从每条记录构建一个纯文本字符串，以便 `AgentContext.store()` 能够嵌入和索引它。
4. **存入 AgentContext**——把字符串、字符串列表或字典列表传给 `context.store()`。可选择启用实体和关系抽取，以同时填充 `ContextGraph`。

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

# AgentContext is the entry point for storage.
# ContextGraph is optional — attach it to build a searchable entity graph.
graph   = ContextGraph()
context = AgentContext(
    vector_store    = VectorStore(backend="faiss", dimension=768),
    knowledge_graph = graph,   # omit if you only need vector search
)

# store() accepts a string, a list of strings, or a list of dicts
context.store("A single observation or document text.")
context.store(["Doc one text.", "Doc two text."])
context.store([{"content": "Doc text.", "metadata": {"source": "wiki"}}])
```

<a id="when-to-use-ingestion"></a>
## 何时使用摄取

当你的数据位于 Semantica 之外且需要把它接入时，使用摄取模块：

- **文件和文档**——PDF、Word 文档、CSV、JSON、XML，以及磁盘上或云存储中的整个目录。
- **网页内容**——公开文档站点、监管出版物页面、新闻源，或任何你可以抓取的 URL。
- **REST API**——内部平台（SIEM、EDR、ITSM、CRM）、威胁情报源，或任何分页 HTTP 端点。
- **数据库**——现有的 SQL 数据库，其中相关记录可以用一条有针对性的查询取回。
- **企业数据平台**——已经存在于 Databricks 湖仓（Unity Catalog + Delta Lake）或 Snowflake 仓库中的表，无需先导出为 CSV。
- **实时流**——Kafka 或其他消息代理，你需要在事件到达时进行处理。
- **Git 仓库**——在版本控制中跟踪的源代码、文档或配置文件。

如果你的数据已经是内存中的 Python 字符串或字典，跳过摄取并直接调用 `context.store()`。

<a id="source-1-pdf-vendor-reports-and-internal-documents"></a>
## 源 1——PDF 厂商报告与内部文档

把文件路径或目录传给 `ingest_file()`，以从 PDF 和其他文档格式中提取纯文本。这对厂商威胁报告、内部策略文档、产品手册或磁盘上任何带文本的文件同样有效：

```python
from semantica.ingest import ingest_file

# Single file
report = ingest_file("apt29_q4_2024.pdf", method="file")
print(report.text[:500])       # extracted plain text
print(report.metadata)         # {"file_type": "pdf", "size": 1843200, ...}
print(report.name)             # "apt29_q4_2024.pdf"
print(report.file_type)        # "pdf"

# Whole directory at once — recursive by default
reports = ingest_file("./vendor_reports/", method="directory", recursive=True)
for r in reports:
    print(f"{r.name}: {len(r.text)} chars extracted")

# apt29_q4_2024.pdf:    42317 chars extracted
# lazarus_group_h2.pdf: 38901 chars extracted
# fin7_campaign.pdf:    51204 chars extracted
# cozy_bear_ttps.pdf:   29876 chars extracted
```

`ingest_file()` 返回一个 `FileObject`（单文件）或 `List[FileObject]`（目录）。`.text` 属性始终是已解码的字符串——你永远不需要手动处理字节或编码。对于 DOCX、XLSX、CSV、TXT、JSON 和 XML 文件，同样的调用也适用；文件类型会从扩展名和 MIME 类型自动检测。

同样的方法适用于内部知识库。如果你的团队把运行手册、产品文档或面向客户的指南存放为 PDF 或 Word 文档：

```python
from semantica.ingest import ingest_file
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store    = VectorStore(backend="faiss"),
    knowledge_graph = graph,
)

# Internal documentation — same API as vendor reports
docs = ingest_file("./internal_docs/", method="directory", recursive=True)
doc_texts = [d.text for d in docs]
context.store(doc_texts, extract_entities=True)
```

如果你的厂商也把报告投递到 S3 存储桶，可以切换到云摄取而无需更改流水线的其余部分：

```python
s3_reports = ingest_file(
    "s3://intel-vendor-bucket/weekly-reports/",
    method="cloud",
    provider="s3",
    aws_access_key_id="AKIA...",
    aws_secret_access_key="...",
)
# Returns List[FileObject] — same shape as the local directory call
```

<a id="source-2-rest-apis"></a>
## 源 2——REST API

`RESTIngestor` 处理认证、重试逻辑和分页。与文件源不同，REST API 返回结构化 JSON——`APIData.data` 是 `List[Dict]`，而不是文本字符串。你需要在把它传给 `AgentContext.store()` 之前从每条记录构建一个文本表示。

```python
from semantica.ingest import RESTIngestor

ingestor = RESTIngestor()

# Single paginated endpoint — returns APIData
# APIData.data is List[Dict] — one dict per record, not a .text string
api_data = ingestor.ingest_endpoint(
    "https://misp.internal/events/restSearch",
    headers={"Authorization": "YOUR_MISP_API_KEY"},
    params={"limit": 100, "page": 1, "threat_level_id": "1"},
)

# api_data.data is List[Dict] — one dict per event
events = api_data.data
print(f"Retrieved {len(events)} MISP events")
print(f"Endpoint: {api_data.endpoint}")
print(f"Status:   {api_data.response_status}")

# Convert to text blobs for the knowledge graph
event_texts = []
for event in events:
    attrs = ", ".join(
        a.get("value", "") for a in event.get("Event", {}).get("Attribute", [])
    )
    text = (
        f"MISP Event {event['Event']['id']}: "
        f"{event['Event'].get('info', '')} "
        f"[TLP: {event['Event'].get('distribution', '')}] "
        f"Attributes: {attrs[:300]}"
    )
    event_texts.append(text)
```

同样的模式适用于任何 REST API——一个工单系统、一个 CRM，或一个内部服务目录。获取结构化记录，然后为每条记录构建一两句话，捕捉用于搜索和抽取的关键事实。

对于跨多页返回数千条记录的端点，`paginated_fetch()` 会自动遍历所有页面，并每页返回一个 `APIData` 对象：

```python
all_events = ingestor.paginated_fetch(
    "https://misp.internal/events/restSearch",
    headers={"Authorization": "YOUR_MISP_API_KEY"},
    page_size=100,
)
# all_events is List[APIData] across all pages
total_events = sum(len(page.data) for page in all_events if isinstance(page.data, list))
print(f"Pages fetched: {len(all_events)}")
print(f"Total events fetched: {total_events}")
```

<a id="source-3-sql-databases"></a>
## 源 3——SQL 数据库

`DBIngestor.execute_query()` 返回 `List[Dict]`——每行一个字典。与 REST API 一样，数据库结果是结构化数据，需要一步文本转换才能存入 `AgentContext`。保持查询聚焦：只取你实际需要的行和列，而不是整表转储。

对于临时 SQL 查询，使用 `DBIngestor.execute_query()`：

```python
from semantica.ingest import DBIngestor

db = DBIngestor()

# Returns List[Dict] — one dict per row
cves = db.execute_query(
    "postgresql://readonly:pass@cvedb.internal:5432/nvd",
    """
        SELECT
            cve_id,
            description,
            cvss_v3_score,
            affected_products,
            published_date,
            last_modified
        FROM cve_records
        WHERE cvss_v3_score >= 7.0
          AND published_date >= NOW() - INTERVAL '30 days'
        ORDER BY cvss_v3_score DESC
    """,
)

print(f"High-severity CVEs (last 30 days): {len(cves)}")

# Convert rows to text for the knowledge graph
cve_texts = [
    f"{r['cve_id']} (CVSS {r['cvss_v3_score']}): "
    f"{r['description']} "
    f"Affects: {r['affected_products']}"
    for r in cves
]
```

同一个 `DBIngestor.execute_query()` 模式适用于 MySQL、SQLite、Oracle 和 SQL Server——只需更换连接字符串。要在编写查询之前做模式发现：

```python
schema = db.execute_query(
    "postgresql://readonly:pass@cvedb.internal:5432/nvd",
    """
        SELECT table_name, column_name, data_type
        FROM information_schema.columns
        WHERE table_schema = 'public'
        ORDER BY table_name, ordinal_position
    """,
)
for col in schema[:10]:
    print(f"{col['table_name']}.{col['column_name']} ({col['data_type']})")
```

<a id="source-4-rss-and-atom-feeds"></a>
## 源 4——RSS 和 Atom 订阅源

`ingest_feed()` 拉取 RSS 和 Atom 订阅源，并返回一个包含 `FeedItem` 对象列表的 `FeedData` 对象。每个条目以字符串形式暴露 `.title`、`.description` 和 `.published`——它们随时可以拼接成文本，无需进一步转换：

```python
from semantica.ingest import ingest_feed

# RSS feed — returns FeedData; iterate .items for FeedItem objects
feed = ingest_feed(
    "https://github.com/advisories.atom",
    method="atom",
)

print(f"Feed title:   {feed.title}")
print(f"Total items:  {len(feed.items)}")

# Convert feed items to text blobs
advisory_texts = []
for item in feed.items:
    text = f"Advisory: {item.title}\n{item.description}"
    advisory_texts.append(text)
    print(f"  {item.title[:80]}  [{item.published}]")

# For NVD's own RSS feed of new CVEs:
nvd_feed = ingest_feed(
    "https://nvd.nist.gov/feeds/xml/cve/misc/nvd-rss.xml",
    method="rss",
)
nvd_texts = [
    f"{item.title}: {item.description}"
    for item in nvd_feed.items
]
```

如果你不知道某个站点用的是 RSS 还是 Atom，`method="discover"` 会从页面 HTML 的 link 标签中找到所有可用的订阅源 URL：

```python
feeds = ingest_feed("https://github.com/advisories", method="discover")
# feeds is a list of validated feed URLs
for f in feeds:
    print(f)
```

<a id="source-5-filesystem-stix-bundles"></a>
## 源 5——文件系统 STIX 包

对目录使用 `ingest_file()`，以从夜间守护进程存放的每个 STIX JSON 文件中提取文本：

```python
from semantica.ingest import ingest_file

# Ingest all STIX JSON files from the directory
stix_files = ingest_file("./stix_bundles/", method="directory", recursive=True)

stix_texts = []
for f in stix_files:
    if f.file_type == "json":
        stix_texts.append(f.text)
        print(f"{f.name}: {f.size:,} bytes")

# apt29_stix_bundle_2024-12-01.json:  184,320 bytes
# lazarus_stix_bundle_2024-12-01.json: 97,408 bytes
# fin7_stix_bundle_2024-12-01.json:   211,968 bytes
```

对于大型基于 XML 的 STIX 1.x 包，使用 `ingest_xml()` 获取结构化解析结果而非原始文本：

```python
from semantica.ingest import ingest_xml

stix_xml_files = ingest_xml("./stix_xml_bundles/", method="directory")
for bundle in stix_xml_files:
    print(f"{bundle.source_path}: {len(bundle.elements)} elements parsed")
```

<a id="source-6-enterprise-data-platforms-databricks-snowflake"></a>
## 源 6——企业数据平台（Databricks 与 Snowflake）

`DatabricksIngestor` 和 `SnowflakeIngestor` 返回包装器对象（`DatabricksData` / `SnowflakeData`），其 `.data` 字段是 `List[Dict]`——与 `DBIngestor.execute_query()` 直接返回的列表字典行形态相同，只是多了一层包装。源 3 中"先转文本，再存储"的同样模式在此适用：用一条有针对性的查询只拉取你需要的表和列，然后为每条记录构建一句话，再交给 `AgentContext.store()`。

```python
from semantica.ingest import DatabricksIngestor

# Unity Catalog + Delta Lake — PAT or OAuth M2M auth
databricks = DatabricksIngestor(
    host="https://adb-xxx.azuredatabricks.net",
    token="dapi-xxxxxxxx",
    http_path="/sql/1.0/warehouses/xxxxxxxx",
    catalog="main",
)

# .data is List[Dict] — one dict per row, same shape as DBIngestor.execute_query()
customers = databricks.ingest_query(
    "SELECT customer_id, name, industry, arr FROM main.default.customers "
    "WHERE churn_risk_score > 0.7"
)
customer_texts = [
    f"Customer {r['customer_id']} ({r['name']}, {r['industry']}): "
    f"ARR ${r['arr']:,}, flagged high churn risk"
    for r in customers.data
]

# Unity Catalog lineage — build Table --DEPENDS_ON--> Table edges directly from
# Unity Catalog's own lineage tracking, instead of re-deriving them from query logs
lineage = databricks.get_table_lineage("customers", catalog="main", schema="default")
lineage_texts = [
    f"Table main.default.customers depends on {upstream}"
    for upstream in lineage["upstream"]
]
```

```python
from semantica.ingest import SnowflakeIngestor

snowflake = SnowflakeIngestor(
    account="myaccount",
    user="myuser",
    password="mypassword",         # or private_key=... for key-pair; use authenticator="oauth", token=... for OAuth
    warehouse="COMPUTE_WH",
    database="ANALYTICS",
    schema="PUBLIC",
)

# Snowflake uppercases unquoted identifiers, so unquoted columns come back
# as ORDER_ID, PRODUCT, etc. unless the source table quotes them lowercase
orders = snowflake.ingest_query(
    "SELECT order_id, product, region, amount FROM orders "
    "WHERE order_date >= DATEADD(day, -30, CURRENT_DATE())"
)
order_texts = [
    f"Order {r['ORDER_ID']}: {r['PRODUCT']} in {r['REGION']}, ${r['AMOUNT']}"
    for r in orders.data
]
```

把生成的文本列表喂给 `AgentContext.store()`，与任何其他结构化源完全一样：

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store    = VectorStore(backend="faiss"),
    knowledge_graph = graph,
)

context.store(
    customer_texts + lineage_texts + order_texts,
    extract_entities=True,
    extract_relationships=True,
)
print(f"Enterprise data graph: {graph.stats()['node_count']} nodes")
```

关于认证详情（Databricks 的 PAT vs. OAuth M2M；Snowflake 的密码 vs. 密钥对 vs. OAuth）、模式/目录内省和故障排查，请参见专门的 [Databricks 集成](../integrations/databricks.zh-CN.md)和 [Snowflake 集成](../integrations/snowflake.zh-CN.md)指南。

> **安全提示：** 切勿在生产代码中硬编码凭据（`token`、`password`、`private_key`）；请通过环境变量（例如 `DATABRICKS_TOKEN`、`SNOWFLAKE_PASSWORD`）或密钥管理器传递它们。

<a id="combining-all-five-sources"></a>
## 合并全部五个源

一旦从每个源得到文本，`AgentContext.store()` 接受一个扁平的字符串列表。Semantica 会把它们一起嵌入和索引——除非你显式添加元数据，否则上下文图谱并不知道哪个字符串来自哪个源。

每个源都返回文本后，把所有内容一次性传入 `AgentContext.store()`：

```python
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.ingest import ingest_file, ingest_feed, RESTIngestor, DBIngestor

# --- Infrastructure ---
graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store    = VectorStore(backend="faiss", dimension=768),
    knowledge_graph = graph,
    graph_expansion = True,
)

# --- Source 1: PDF reports ---
reports      = ingest_file("./vendor_reports/", method="directory")
report_texts = [r.text for r in reports]

# --- Source 2: MISP API ---
ingestor   = RESTIngestor()
api_data   = ingestor.paginated_fetch(
    "https://misp.internal/events/restSearch",
    headers={"Authorization": "YOUR_MISP_API_KEY"},
    page_size=100,
)
misp_texts = [
    f"MISP {e['Event']['id']}: {e['Event'].get('info', '')} "
    f"attrs={len(e['Event'].get('Attribute', []))}"
    for page in api_data
    for e in page.data
]

# --- Source 3: PostgreSQL CVEs ---
db        = DBIngestor()
cves      = db.execute_query(
    "postgresql://readonly:pass@cvedb.internal:5432/nvd",
    "SELECT cve_id, description, cvss_v3_score FROM cve_records "
    "WHERE cvss_v3_score >= 7.0 AND published_date >= NOW() - INTERVAL '30 days'",
)
cve_texts = [f"{r['cve_id']} (CVSS {r['cvss_v3_score']}): {r['description']}" for r in cves]

# --- Source 4: GitHub Advisory feed ---
feed           = ingest_feed("https://github.com/advisories.atom", method="atom")
advisory_texts = [f"{item.title}: {item.description}" for item in feed.items]

# --- Source 5: STIX bundles ---
stix_files = ingest_file("./stix_bundles/", method="directory")
stix_texts = [f.text for f in stix_files if f.file_type == "json"]

# --- Combine and store ---
all_texts = report_texts + misp_texts + cve_texts + advisory_texts + stix_texts

context.store(
    all_texts,
    extract_entities      = True,
    extract_relationships = True,
)

s = graph.stats()
print(f"Graph: {s['node_count']} nodes, {s['edge_count']} edges")
print(f"Total documents ingested: {len(all_texts)}")
```

<a id="common-pitfalls"></a>
## 常见陷阱

**摄取超过所需的数据。** 转储整表或抓取无限页面会用噪声填满你的向量索引并拖慢检索。使用 `WHERE` 子句、日期过滤器和 `page_size` 限制，只取与你用例相关的记录。

**结构化数据的文本质量差。** 一行数据库记录或 API 响应包含字段名、ID 和原始值——而不是句子。像 `"2025-06-21|CVE-2024-3400|10.0"` 这样的字符串嵌入效果差，会产生薄弱的搜索结果。把每条记录格式化成自然句子：`"CVE-2024-3400 (CVSS 10.0): critical RCE in PAN-OS, published 2025-06-21."` 这点额外功夫会在检索质量上得到回报。

**不处理分页。** `RESTIngestor.ingest_endpoint()` 只取单页。如果你的端点有数千条记录，请使用 `paginated_fetch()`——否则你会默默地只摄取第一页。

**API 速率限制。** `RESTIngestor` 在收到 HTTP 429 响应时会自动以指数退避重试（由 `RESTIngestor(config={"backoff_factor": 2})` 中的 `backoff_factor` 控制）。这会响应式地处理突发速率限制，但不会在成功调用之间主动放慢请求。对于有严格每秒配额的 API，请减小 `page_size` 以降低请求频率，或在你自己的循环中在 `paginated_fetch()` 调用之间添加延迟。

**大规模数据库导出。** 把成千上万行导出到 `AgentContext` 很少是正确做法。编写一条只选择与领域相关记录、按日期范围过滤、并只投影你格式化文本所需列的查询。

<a id="handling-errors-gracefully"></a>
## 优雅地处理错误

把每个源包在 try/except 中，这样流水线可以继续运行并在最后报告失败，而不是在第一个坏源上崩溃。这在定时任务中尤其重要——部分数据胜过无数据：

```python
from semantica.ingest import ingest_file, ingest_feed, RESTIngestor, DBIngestor

all_texts = []
errors    = []

# Source 1: PDFs
try:
    reports = ingest_file("./vendor_reports/", method="directory")
    all_texts.extend(r.text for r in reports)
    print(f"PDFs:       {len(reports)} files ingested")
except Exception as e:
    errors.append(f"PDF ingest failed: {e}")

# Source 2: MISP
try:
    ingestor = RESTIngestor()
    events   = ingestor.paginated_fetch(
        "https://misp.internal/events/restSearch",
        headers={"Authorization": "YOUR_MISP_API_KEY"},
        page_size=100,
    )
    all_texts.extend(
        f"MISP {e['Event']['id']}: {e['Event'].get('info', '')}"
        for page in events
        for e in page.data
    )
    print(f"MISP:       {sum(len(page.data) for page in events if isinstance(page.data, list))} events ingested")
except Exception as e:
    errors.append(f"MISP API failed: {e}")

# Source 3: PostgreSQL
try:
    db = DBIngestor()
    cves = db.execute_query(
        "postgresql://readonly:pass@cvedb.internal:5432/nvd",
        "SELECT cve_id, description, cvss_v3_score FROM cve_records "
        "WHERE cvss_v3_score >= 7.0 AND published_date >= NOW() - INTERVAL '30 days'",
    )
    all_texts.extend(f"{r['cve_id']}: {r['description']}" for r in cves)
    print(f"CVEs:       {len(cves)} records ingested")
except Exception as e:
    errors.append(f"PostgreSQL failed: {e}")

# Source 4: GitHub feed
try:
    feed = ingest_feed("https://github.com/advisories.atom", method="atom")
    all_texts.extend(f"{item.title}: {item.description}" for item in feed.items)
    print(f"Advisories: {len(feed.items)} items ingested")
except Exception as e:
    errors.append(f"GitHub feed failed: {e}")

# Source 5: STIX files
try:
    stix_files = ingest_file("./stix_bundles/", method="directory")
    stix_texts = [f.text for f in stix_files if f.file_type == "json"]
    all_texts.extend(stix_texts)
    print(f"STIX:       {len(stix_texts)} bundles ingested")
except Exception as e:
    errors.append(f"STIX directory failed: {e}")

# Report any failures — don't silently swallow them
if errors:
    print(f"\n{len(errors)} source(s) failed:")
    for err in errors:
        print(f"  - {err}")

print(f"\nTotal documents for graph: {len(all_texts)}")
```

<a id="scheduling-recurring-ingestion"></a>
## 调度定期摄取

把合并后的摄取包到一个函数中，并从你选择的调度器（cron、Airflow、云调度器）调用它：

```python
from datetime import datetime, timedelta
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore
from semantica.ingest import ingest_file, ingest_feed, RESTIngestor, DBIngestor

def run_daily_ingest(since: datetime = None):
    """Run the full five-source ingest pipeline. Returns graph stats dict."""
    since = since or datetime.now() - timedelta(days=1)

    graph   = ContextGraph(advanced_analytics=True)
    context = AgentContext(
        vector_store    = VectorStore(backend="faiss", dimension=768),
        knowledge_graph = graph,
        graph_expansion = True,
    )

    all_texts = []
    errors    = []

    # PDFs deposited since last run
    try:
        reports = ingest_file("./vendor_reports/", method="directory")
        # Filter by modification time in production — simplified here
        all_texts.extend(r.text for r in reports)
    except Exception as e:
        errors.append(f"PDF: {e}")

    # MISP events since last run
    try:
        ingestor = RESTIngestor()
        events   = ingestor.paginated_fetch(
            "https://misp.internal/events/restSearch",
            headers={"Authorization": "YOUR_MISP_API_KEY"},
            params={"timestamp": int(since.timestamp())},
            page_size=100,
        )
        all_texts.extend(
            f"MISP {e['Event']['id']}: {e['Event'].get('info', '')}"
            for page in events
            for e in page.data
        )
    except Exception as e:
        errors.append(f"MISP: {e}")

    # CVEs published since last run
    try:
        db = DBIngestor()
        cves = db.execute_query(
            "postgresql://readonly:pass@cvedb.internal:5432/nvd",
            f"SELECT cve_id, description, cvss_v3_score FROM cve_records "
            f"WHERE published_date >= '{since.isoformat()}' AND cvss_v3_score >= 7.0",
        )
        all_texts.extend(f"{r['cve_id']}: {r['description']}" for r in cves)
    except Exception as e:
        errors.append(f"PostgreSQL: {e}")

    # GitHub advisory feed (always latest)
    try:
        feed = ingest_feed("https://github.com/advisories.atom", method="atom")
        all_texts.extend(f"{item.title}: {item.description}" for item in feed.items)
    except Exception as e:
        errors.append(f"GitHub feed: {e}")

    # STIX bundles deposited since last run
    try:
        stix_files = ingest_file("./stix_bundles/", method="directory")
        all_texts.extend(f.text for f in stix_files if f.file_type == "json")
    except Exception as e:
        errors.append(f"STIX: {e}")

    if all_texts:
        context.store(all_texts, extract_entities=True, extract_relationships=True)
        context.save("./cti_state/")

    return {
        "run_at":     datetime.now().isoformat(),
        "documents":  len(all_texts),
        "errors":     errors,
        "graph":      graph.stats(),
    }

if __name__ == "__main__":
    result = run_daily_ingest()
    print(f"Ingested {result['documents']} documents")
    print(f"Graph: {result['graph']['node_count']} nodes, {result['graph']['edge_count']} edges")
    if result["errors"]:
        print("Errors:", result["errors"])
```

<a id="business-examples"></a>
## 业务示例

以下两种模式在安全和研究场景之外也很常见。

**内部产品文档。** 如果你的团队把产品文档、运行手册或入职指南维护为共享盘上的 Markdown 或 PDF 文件，一次性摄取它们，让智能体针对完整语料回答问题，而不是关键词搜索。

```python
from semantica.ingest import ingest_file
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store    = VectorStore(backend="faiss"),
    knowledge_graph = graph,
)

docs = ingest_file("./product_docs/", method="directory", recursive=True)
context.store(
    [d.text for d in docs],
    extract_entities=True,
)
print(f"Indexed {len(docs)} documentation pages")
```

**来自数据库的客户支持工单。** 存储在 SQL 数据库中的支持工单用自然语言捕捉了真实世界的产品问题。摄取它们让你能够浮现模式、找到相似的历史问题，并构建检索增强的支持工具。

```python
from semantica.ingest import DBIngestor
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store    = VectorStore(backend="faiss"),
    knowledge_graph = graph,
)

db = DBIngestor()
# Fetch only resolved tickets from the last 90 days — avoid full table dumps
tickets = db.execute_query(
    "postgresql://readonly:YOUR_DB_PASSWORD@support-db:5432/helpdesk",
    """
        SELECT ticket_id, subject, description, resolution, product_area
        FROM support_tickets
        WHERE status = 'resolved'
          AND created_at >= NOW() - INTERVAL '90 days'
        ORDER BY created_at DESC
        LIMIT 5000
    """,
)

# Transform each row into a readable text string
# Guard against NULL description/resolution — common in real helpdesk schemas
ticket_texts = [
    f"Ticket {r['ticket_id']} [{r['product_area']}]: {r['subject']}. "
    f"Description: {(r['description'] or '')[:300]}. "
    f"Resolution: {(r['resolution'] or '')[:200]}"
    for r in tickets
]

context.store(ticket_texts, extract_entities=True)
print(f"Indexed {len(ticket_texts)} support tickets")
```

<a id="domain-examples"></a>
## 领域示例

以下示例展示了常见部署场景下完整的多源流水线。每个都遵循相同的工作流：从多个源摄取，把结构化数据转换成文本，然后存入共享上下文。

<Tabs>
  <Tab title="国防——CTI/威胁">
    一个联合情报单元每六小时融合三个实时源：NVD CVE RSS、合作机构投放的涉密 PDF，以及一个内部 MISP 实例。

```python
from semantica.ingest import ingest_file, ingest_feed, RESTIngestor
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

graph   = ContextGraph(advanced_analytics=True, community_detection=True)
context = AgentContext(
    vector_store    = VectorStore(backend="faiss", dimension=768),
    knowledge_graph = graph,
    graph_expansion = True,
)

# NVD CVE feed — new CVEs in last 6 hours
nvd_feed  = ingest_feed(
    "https://nvd.nist.gov/feeds/xml/cve/misc/nvd-rss.xml",
    method="rss",
)
nvd_texts = [f"{item.title}: {item.description}" for item in nvd_feed.items]

# Classified PDF drop from partner agency
partner_docs  = ingest_file("//partner-share/intel-drops/", method="directory")
partner_texts = [doc.text for doc in partner_docs]

# MISP events tagged TLP:AMBER or higher
ingestor   = RESTIngestor()
misp_data  = ingestor.paginated_fetch(
    "https://misp.internal/events/restSearch",
    headers={"Authorization": "YOUR_MISP_API_KEY"},
    params={"tags": "tlp:amber||tlp:red", "threat_level_id": "1"},
    page_size=100,
)
misp_texts = [
    f"MISP {e['Event']['id']}: {e['Event'].get('info', '')}"
    for page in misp_data
    for e in page.data
]

# Fuse all three into the graph
context.store(
    nvd_texts + partner_texts + misp_texts,
    extract_entities=True, extract_relationships=True,
)
print(f"Fused graph: {graph.stats()['node_count']} nodes")
```

  </Tab>

  <Tab title="安全——SOC/事件">
    在一次活跃事件中，SOC 以 15 分钟为周期从三个源丰富时间线：SIEM 告警 CSV 导出、EDR REST API，以及内部 CVE 数据库。

```python
from semantica.ingest import ingest_file, RESTIngestor, DBIngestor
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store    = VectorStore(backend="faiss", dimension=768),
    knowledge_graph = graph,
)

# SIEM alert CSV exports (written by SIEM every 15 minutes)
alert_files  = ingest_file("./siem_exports/", method="directory")
alert_texts  = [
    f.text for f in alert_files if f.file_type == "csv"
]

# EDR platform REST API — host telemetry for the affected segment
ingestor  = RESTIngestor()
edr_data  = ingestor.ingest_endpoint(
    "https://edr.internal/api/v1/telemetry",
    headers={"X-API-Key": "EDR_KEY"},
    params={"segment": "finance", "severity": "high", "limit": 500},
)
edr_texts = [
    f"Host {e.get('hostname')}: {e.get('event_type')} — {e.get('description')}"
    for e in edr_data.data
]

# CVE cross-reference for any CVE IDs observed in alerts
db = DBIngestor()
exploited_cves = db.execute_query(
    "postgresql://readonly:pass@cvedb.internal:5432/nvd",
    "SELECT cve_id, description, cvss_v3_score FROM cve_records "
    "WHERE cve_id = ANY(ARRAY['CVE-2024-3400', 'CVE-2024-21762'])",
)
cve_texts = [f"{r['cve_id']} (CVSS {r['cvss_v3_score']}): {r['description']}"
             for r in exploited_cves]

context.store(
    alert_texts + edr_texts + cve_texts,
    extract_entities=True, extract_relationships=True,
)
print(f"Incident graph: {graph.stats()['node_count']} nodes enriched")
```

  </Tab>

  <Tab title="生命科学——临床/制药">
    一个药物警戒平台在每个试验阶段结束后摄取三个数据源：FDA 提交 PDF、一个 PostgreSQL 临床试验数据库，以及通过网页抓取获取的 PubMed 文献。

```python
from semantica.ingest import ingest_file, ingest_web, DBIngestor
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store    = VectorStore(backend="faiss", dimension=768),
    knowledge_graph = graph,
    retention_days  = None,   # regulatory data — unlimited retention
)

# FDA submissions for Phase III oncology trials
submissions  = ingest_file("./fda_submissions/phase3_oncology/", method="directory")
sub_texts    = [s.text for s in submissions]

# Clinical trials database — protocol and AE records
db = DBIngestor()
trial_records = db.execute_query(
    "postgresql://readonly:pass@clindb.internal:5432/trials",
    """
        SELECT t.protocol_id, t.title, t.primary_endpoint,
               ae.event_type, ae.severity, ae.frequency_pct
        FROM clinical_trials t
        JOIN adverse_events ae ON t.protocol_id = ae.protocol_id
        WHERE t.phase = 'III' AND t.therapeutic_area = 'oncology'
        ORDER BY ae.severity DESC
    """,
)
trial_texts = [
    f"Protocol {r['protocol_id']}: {r['title']} "
    f"AE: {r['event_type']} (severity={r['severity']}, freq={r['frequency_pct']}%)"
    for r in trial_records
]

# PubMed literature for the drug compound
pubmed_pages = ingest_web(
    "https://pubmed.ncbi.nlm.nih.gov/?term=dapagliflozin+cardiovascular",
    method="url",
)
pub_texts = [pubmed_pages.text]

context.store(
    sub_texts + trial_texts + pub_texts,
    extract_entities=True, extract_relationships=True,
)
print(f"Pharmacovigilance graph: {graph.stats()['node_count']} nodes")
```

  </Tab>

  <Tab title="银行——风险/合规">
    一个合规团队每季度从三个源拉取数据：BIS 和 Basel 出版物（网页站点地图抓取）、内部策略 PDF，以及一个监管规则数据库。

```python
from semantica.ingest import ingest_file, ingest_web, DBIngestor
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore

graph   = ContextGraph(advanced_analytics=True)
context = AgentContext(
    vector_store    = VectorStore(backend="faiss", dimension=768),
    knowledge_graph = graph,
    retention_days  = 2555,   # 7-year regulatory retention requirement
)

# BIS/Basel publications via sitemap crawl — filter to capital/liquidity pages
bis_pages    = ingest_web("https://www.bis.org/sitemap.xml", method="sitemap")
basel_pages  = [p for p in bis_pages if any(
    kw in p.url.lower() for kw in ["bcbs", "capital", "liquidity", "leverage"]
)]
bis_texts    = [p.text for p in basel_pages]
print(f"BIS pages matched: {len(bis_texts)}")

# Internal policy library
policies     = ingest_file("./regulatory_library/", method="directory", recursive=True)
policy_texts = [p.text for p in policies]

# Regulatory rules database — active rules only
db = DBIngestor()
rules = db.execute_query(
    "postgresql://compliance_ro:pass@regdb.internal:5432/compliance",
    """
        SELECT rule_id, title, requirement_text, effective_date, jurisdiction
        FROM regulations
        WHERE active = true AND jurisdiction IN ('EU', 'US', 'UK')
        ORDER BY effective_date DESC
    """,
)
rule_texts = [
    f"{r['rule_id']} [{r['jurisdiction']}] {r['title']}: {r['requirement_text']}"
    for r in rules
]

context.store(
    bis_texts + policy_texts + rule_texts,
    extract_entities=True, extract_relationships=True,
)
print(f"Compliance graph: {graph.stats()['node_count']} nodes, "
      f"{graph.stats()['edge_count']} edges")
```

  </Tab>
</Tabs>

<a id="related-guides"></a>
## 相关指南

- [流水线](pipeline.zh-CN.md)——用 `PipelineBuilder` 把摄取步骤串联起来，实现自动化、可重试、可并行的工作流
- [上下文图谱](context-graphs.zh-CN.md)——把你摄取的实体作为带类型的属性图存储和查询
- [语义抽取](semantic-extraction.zh-CN.md)——从摄取的文本中做 NER、关系抽取和三元组抽取
- [溯源](provenance.zh-CN.md)——为每个抽取的实体跟踪来源文档、置信度得分和摄取时间戳
- [Databricks 集成](../integrations/databricks.zh-CN.md)——Unity Catalog 设置、PAT/OAuth M2M 认证、血缘内省
- [Snowflake 集成](../integrations/snowflake.zh-CN.md)——仓库设置与密码/密钥对/OAuth 认证
