---
title: "Databricks 集成"
description: "将 Unity Catalog 元数据和 Delta Lake 表从 Databricks 摄取到 Semantica 的知识图谱流水线。"
icon: "cloud"
---

**[English](databricks.md)** · **简体中文（当前）**

> 通过个人访问令牌或 OAuth M2M 认证，从 Databricks 中抽取 Delta Lake 表和 Unity Catalog 元数据（模式、血缘）到 Semantica。


<a id="installation"></a>
## 安装

```bash
# 安装 Databricks 支持
pip install "semantica[db-databricks]"

# 或分别安装连接器
pip install databricks-sdk databricks-sql-connector
```


<a id="basic-usage"></a>
## 基本用法

```python
from semantica.ingest import DatabricksIngestor
import os

ingestor = DatabricksIngestor(
    host=os.getenv("DATABRICKS_HOST"),           # 例如 https://adb-xxx.azuredatabricks.net
    token=os.getenv("DATABRICKS_TOKEN"),
    http_path=os.getenv("DATABRICKS_HTTP_PATH"),  # SQL warehouse 或集群的 HTTP 路径
    catalog=os.getenv("DATABRICKS_CATALOG", "main"),
    schema=os.getenv("DATABRICKS_SCHEMA", "default"),
)

data = ingestor.ingest_table("customers")
print(f"Retrieved {data.row_count} rows: columns: {data.columns}")
```

<Tip>
使用环境变量（或通过 `python-dotenv` 加载 `.env` 文件）将凭证排除在源代码之外。不带参数的 `DatabricksIngestor()` 会自动从 `DATABRICKS_*` 环境变量读取配置。
</Tip>


<a id="authentication-methods"></a>
## 认证方式

<Tabs>
  <Tab title="个人访问令牌">
    ```python
    ingestor = DatabricksIngestor(
        host="https://adb-xxx.azuredatabricks.net",
        token="dapi-xxxxxxxx",
        http_path="/sql/1.0/warehouses/xxxxxxxx",
    )
    ```
  </Tab>
  <Tab title="OAuth M2M（推荐）">
    ```python
    ingestor = DatabricksIngestor(
        host="https://adb-xxx.azuredatabricks.net",
        client_id="your_service_principal_client_id",
        client_secret="your_service_principal_client_secret",
        http_path="/sql/1.0/warehouses/xxxxxxxx",
    )
    ```
    生产环境首选：配置中不存储长期有效的个人令牌。
  </Tab>
</Tabs>

<Note>
`http_path` 标识用于执行查询的 SQL warehouse 或通用集群。可在 Databricks UI 的 **SQL Warehouses → Connection details** 下找到。Unity Catalog 元数据调用（`list_catalogs`、`get_table_schema`、`get_table_lineage` 等）只需要 `host` 和凭证——这些调用不需要 `http_path`。
</Note>


<a id="querying"></a>
## 查询

<a id="ingest-a-table-with-filters"></a>
### 带过滤条件摄取表

```python
data = ingestor.ingest_table(
    "customers",
    catalog="main",
    schema="default",
    where="country = 'USA' AND created_date > '2024-01-01'",
    order_by="created_date DESC",
    limit=10000,
)
```

<a id="custom-sql"></a>
### 自定义 SQL

```python
data = ingestor.ingest_query("""
    SELECT customer_id, SUM(amount) AS total_amount
    FROM main.default.sales
    WHERE date >= '2024-01-01'
    GROUP BY customer_id
""")
```


<a id="unity-catalog-metadata"></a>
## Unity Catalog 元数据

<a id="schema-introspection"></a>
### 模式内省

```python
schema = ingestor.get_table_schema("customers")
for column in schema["columns"]:
    print(f"{column['name']}: {column['type']}")
```

<a id="catalogs-schemas-and-tables"></a>
### 目录、模式和表

```python
catalogs = ingestor.list_catalogs()
schemas = ingestor.list_schemas(catalog="main")
tables = ingestor.list_tables(catalog="main", schema="default")
```

<a id="table-and-column-lineage"></a>
### 表和列血缘

```python
lineage = ingestor.get_table_lineage("customers", catalog="main", schema="default")
print(lineage["upstream"])    # 汇入 `customers` 的表
print(lineage["downstream"])  # 从 `customers` 派生的表
```

使用 `get_table_lineage` 可以直接从 Unity Catalog 的血缘跟踪中构建 `Table --DEPENDS_ON--> Table` 边，无需从查询日志中重新推导血缘。

<Tip>
传入 `include_column_lineage=True` 可同时解析每列的上游/下游引用（每列会多一次 Unity Catalog 请求，因此是可选的）：

```python
lineage = ingestor.get_table_lineage(
    "customers", catalog="main", schema="default", include_column_lineage=True,
)
print(lineage["columns"]["email"])
# {"upstream": ["main.default.raw_customers.email_address"], "downstream": []}
```

</Tip>


<a id="export-as-semantica-documents"></a>
## 导出为 Semantica 文档

```python
documents = ingestor.export_as_documents(
    data,
    id_field="customer_id",
    text_fields=["name", "email", "notes"],
)
print(f"Created {len(documents)} documents for processing")
```


<a id="batch-processing-large-tables"></a>
## 批量处理大表

```python
PAGE_SIZE = 5000
for page in range(total_pages):
    data = ingestor.ingest_table(
        "large_table",
        limit=PAGE_SIZE,
        offset=page * PAGE_SIZE,
    )
    process_batch(data)
```

或使用内置的 `batch_size` 参数：

```python
data = ingestor.ingest_query(
    "SELECT * FROM main.default.large_table",
    batch_size=5000,
)
```


<a id="troubleshooting"></a>
## 故障排查

```python
from semantica.ingest import DatabricksConnector

connector = DatabricksConnector(
    host="https://adb-xxx.azuredatabricks.net",
    token="dapi-xxxxxxxx",
    http_path="/sql/1.0/warehouses/xxxxxxxx",
)
if not connector.test_connection():
    print("Connection failed: check host, http_path, and credentials")
```


<a id="see-also"></a>
## 另请参阅

- [摄取模块](../reference/ingest.zh-CN.md) — 完整的 DatabricksIngestor 及所有其他摄取器。
- [Snowflake 集成](snowflake.zh-CN.md) — 面向 Snowflake + Databricks 混合数据资产的配套连接器。
- [流水线](../reference/pipeline.zh-CN.md) — 将 Databricks 摄取作为流水线步骤使用。
- [安装](../installation.zh-CN.md) — 所有可选的依赖扩展。
- [知识图谱](../reference/kg.zh-CN.md) — 从摄取的 Databricks 数据构建知识图谱。
