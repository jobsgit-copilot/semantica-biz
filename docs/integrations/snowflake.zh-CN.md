---
title: "Snowflake 集成"
description: "将 Snowflake 表和查询中的结构化数据摄取到 Semantica 的知识图谱流水线。"
icon: "snowflake"
---

**[English](snowflake.md)** · **简体中文（当前）**

> 通过密码、密钥对、OAuth 和 SSO 认证，从 Snowflake 中抽取数据到 Semantica。


<a id="installation"></a>
## 安装

```bash
# 安装 Snowflake 支持
pip install "semantica[db-snowflake]"

# 或单独安装连接器
pip install snowflake-connector-python
```


<a id="basic-usage"></a>
## 基本用法

```python
from semantica.ingest import SnowflakeIngestor
import os

ingestor = SnowflakeIngestor(
    account=os.getenv("SNOWFLAKE_ACCOUNT"),
    user=os.getenv("SNOWFLAKE_USER"),
    password=os.getenv("SNOWFLAKE_PASSWORD"),
    warehouse=os.getenv("SNOWFLAKE_WAREHOUSE"),
    database=os.getenv("SNOWFLAKE_DATABASE"),
    schema=os.getenv("SNOWFLAKE_SCHEMA"),
)

data = ingestor.ingest_table("CUSTOMERS")
print(f"Retrieved {data.row_count} rows: columns: {data.columns}")
```

<Tip>
使用环境变量（或通过 `python-dotenv` 加载 `.env` 文件）将凭证排除在源代码之外。不带参数的 `SnowflakeIngestor()` 会自动从 `SNOWFLAKE_*` 环境变量读取配置。
</Tip>


<a id="authentication-methods"></a>
## 认证方式

<Tabs>
  <Tab title="密码">
    ```python
    ingestor = SnowflakeIngestor(
        account="myaccount",
        user="myuser",
        password="mypassword",
        warehouse="COMPUTE_WH",
    )
    ```
  </Tab>
  <Tab title="密钥对（推荐）">
    ```python
    ingestor = SnowflakeIngestor(
        account="myaccount",
        user="myuser",
        private_key_path="/path/to/rsa_key.p8",
        warehouse="COMPUTE_WH",
    )
    ```
    生产环境首选：配置中不存储密码。
  </Tab>
  <Tab title="OAuth">
    ```python
    ingestor = SnowflakeIngestor(
        account="myaccount",
        user="myuser",
        authenticator="oauth",
        token="your_oauth_token",
        warehouse="COMPUTE_WH",
    )
    ```
  </Tab>
  <Tab title="SSO">
    ```python
    ingestor = SnowflakeIngestor(
        account="myaccount",
        user="myuser",
        authenticator="externalbrowser",
        warehouse="COMPUTE_WH",
    )
    ```
  </Tab>
</Tabs>


<a id="querying"></a>
## 查询

<a id="ingest-a-table-with-filters"></a>
### 带过滤条件摄取表

```python
data = ingestor.ingest_table(
    "CUSTOMERS",
    where="COUNTRY = 'USA' AND CREATED_DATE > '2024-01-01'",
    order_by="CREATED_DATE DESC",
    limit=10000,
)
```

<a id="custom-sql"></a>
### 自定义 SQL

```python
data = ingestor.ingest_query("""
    SELECT CUSTOMER_ID, SUM(AMOUNT) AS TOTAL_AMOUNT
    FROM SALES
    WHERE DATE >= '2024-01-01'
    GROUP BY CUSTOMER_ID
""")
```

<a id="schema-introspection"></a>
### 模式内省

```python
schema = ingestor.get_table_schema("CUSTOMERS")
for column in schema["columns"]:
    print(f"{column['name']}: {column['type']}")
```


<a id="export-as-semantica-documents"></a>
## 导出为 Semantica 文档

```python
documents = ingestor.export_as_documents(
    data,
    id_field="CUSTOMER_ID",
    text_fields=["NAME", "EMAIL", "NOTES"],
)
print(f"Created {len(documents)} documents for processing")
```


<a id="batch-processing-large-tables"></a>
## 批量处理大表

```python
PAGE_SIZE = 5000
for page in range(total_pages):
    data = ingestor.ingest_table(
        "LARGE_TABLE",
        limit=PAGE_SIZE,
        offset=page * PAGE_SIZE,
    )
    process_batch(data)
```

或使用内置的 `batch_size` 参数：

```python
data = ingestor.ingest_query(
    "SELECT * FROM LARGE_TABLE",
    batch_size=5000,
)
```


<a id="troubleshooting"></a>
## 故障排查

```python
from semantica.ingest import SnowflakeConnector

connector = SnowflakeConnector(account="myaccount", user="myuser", password="mypassword")
if not connector.test_connection():
    print("Connection failed: check credentials and account identifier")
```


<a id="see-also"></a>
## 另请参阅

- [摄取模块](../reference/ingest.zh-CN.md) — 完整的 SnowflakeIngestor 及所有其他摄取器。
- [Databricks 集成](databricks.zh-CN.md) — 面向 Snowflake + Databricks 混合数据资产的配套连接器。
- [流水线](../reference/pipeline.zh-CN.md) — 将 Snowflake 摄取作为流水线步骤使用。
- [安装](../installation.zh-CN.md) — 所有可选的依赖扩展。
- [知识图谱](../reference/kg.zh-CN.md) — 从摄取的 Snowflake 数据构建知识图谱。
