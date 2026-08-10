---
title: "图存储模块"
description: "为 Neo4j、FalkorDB、Apache AGE 和 Amazon Neptune 图数据库提供统一接口。"
icon: "server"
---

**[English](graph_store.md)** · **简体中文（当前）**

**`semantica.graph_store`** 提供**单一统一 API**，用于在生产图数据库中持久化和查询知识图谱：

- 一行代码即可切换后端：Neo4j、FalkorDB、Apache AGE、Amazon Neptune
- 通过 `QueryEngine` 实现参数化 Cypher 执行与可选的结果缓存
- 批量节点与边加载：比逐条写入更快
- `GraphAnalytics` 提供度中心性、连通分量、最短路径、邻居遍历
- 支持上下文管理器：`with GraphStore(...) as store:` 会自动关闭连接


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `GraphStore` | 统一接口：`create_node`、`create_relationship`、`query`、`get_neighbors`、`shortest_path` |
| `QueryEngine` | 带结果缓存的参数化 Cypher 执行 |
| `GraphAnalytics` | `degree_centrality`、`connected_components`、`shortest_path`、`get_neighbors` |
| `Neo4jStore` | 通过 Bolt 承载生产工作负载：支持 APOC 与 GDS 插件 |
| `ApacheAgeStore` | PostgreSQL + AGE 扩展：无需单独部署图服务器 |
| `AmazonNeptuneStore` | AWS Neptune：通过 Bolt 协议使用 OpenCypher |
| `FalkorDBStore` | 基于 Redis：为实时应用提供亚毫秒级延迟 |


<a id="what-you-get"></a>
## 你将获得

- **GraphStore** —— 跨 Neo4j、FalkorDB、Apache AGE、Amazon Neptune 的统一 API
  - 支持上下文管理器，可自动清理连接
  - `create_nodes()` 用于批量加载：比逐条调用更快
- **QueryEngine** —— 参数化 Cypher 构造可防止注入攻击
  - 可选的进程内结果缓存，通过 `use_cache=True` 开启
  - 写入时调用 `clear_cache()`，通过 `enable_cache()` / `disable_cache()` 切换
- **GraphAnalytics** —— 按度数降序排列的度中心性
  - 连通分量分配
  - 节点间最短路径、最多 N 跳的邻居遍历
- **批量操作** —— `create_nodes(list)`：一次往返加载多个节点
  - 支持类型化属性的 `create_relationship()`
  - `delete_node(detach=True)` 会删除所有相连的关系
- **模式管理** —— `create_index(label, property_name=)`：让 MATCH 查询快上数个数量级
  - `get_stats()`：节点数、边数、类型分布
  - 在批量加载前创建索引可获得最佳性能
- **路径遍历** —— `shortest_path()` 返回 `length`、`nodes`、`relationships`
  - 支持方向与深度控制的 `get_neighbors()`
  - 通过统一 API 实现跨后端路径遍历


<a id="getting-started"></a>
## 快速上手

`GraphStore` 将你选择的后端封装在单一 API 之后。在运行任何查询之前，请调用 `connect()`（或使用上下文管理器）：

```python
from semantica.graph_store import GraphStore

store = GraphStore(
    backend="neo4j",
    uri="bolt://localhost:7687",
    user="neo4j",
    password="password",
)
store.connect()

# 创建一个节点
node = store.create_node(
    labels=["Person"],
    properties={"name": "Alice", "role": "Engineer"},
)
print(node["id"])   # Neo4j 内部整数 ID

# 创建一个关系
store.create_relationship(
    start_node_id=node1_id,
    end_node_id=node2_id,
    rel_type="WORKS_FOR",
    properties={"since": 2022},
)

# 执行 Cypher 查询
results = store.query(
    "MATCH (p:Person)-[:WORKS_FOR]->(o:Organization) WHERE o.name = $org RETURN p",
    parameters={"org": "Acme Corp"},
)

store.close()
```

使用上下文管理器可自动关闭连接：

```python
with GraphStore(backend="neo4j", uri="bolt://localhost:7687", user="neo4j", password="password") as store:
    store.create_node(labels=["Person"], properties={"name": "Bob"})
```

<Warning>
  **在任何操作之前调用 `connect()`。** `GraphStore` 在构造时不会自动连接。请显式调用 `store.connect()`，或使用上下文管理器形式 `with GraphStore(...) as store:`。
</Warning>

<a id="quick-start"></a>
## 快速启动

<Steps>
  <Step title="连接到图数据库">
    ```python
    from semantica.graph_store import GraphStore

    store = GraphStore(
        backend="neo4j",
        uri="bolt://localhost:7687",
        user="neo4j",
        password="password",
    )
    store.connect()
    ```
  </Step>
  <Step title="在加载数据前创建索引">
    ```python
    store.create_index(label="Person",       property_name="name")
    store.create_index(label="Organization", property_name="name")
    ```
  </Step>
  <Step title="加载节点与关系">
    ```python
    # 批量创建：包含 "labels" 与 "properties" 键的字典列表
    store.create_nodes([
        {"labels": ["Person"],       "properties": {"name": "Alice"}},
        {"labels": ["Organization"], "properties": {"name": "Acme Corp"}},
    ])
    ```
  </Step>
  <Step title="查询图">
    ```python
    results = store.query(
        "MATCH (p:Person)-[:WORKS_FOR]->(o:Organization) WHERE o.name = $org RETURN p",
        parameters={"org": "Acme Corp"},
    )
    ```
  </Step>
</Steps>


<a id="graphstore-methods"></a>
## GraphStore 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `create_node(labels, properties)` | `dict` | 创建单个节点，返回包含 `"id"` 的节点字典 |
| `create_nodes(nodes)` | `List[dict]` | 从 `{"labels", "properties"}` 字典列表批量创建节点 |
| `get_node(node_id)` | `dict \| None` | 通过后端 ID 检索节点 |
| `get_nodes(labels, properties, limit)` | `List[dict]` | 查询匹配标签/属性条件的节点 |
| `update_node(node_id, properties, merge)` | `dict` | 更新节点属性；`merge=True`（默认）合并，`merge=False` 替换 |
| `delete_node(node_id, detach)` | `bool` | 删除节点；`detach=True`（默认）同时删除相连关系 |
| `create_relationship(start_node_id, end_node_id, rel_type, properties)` | `dict` | 创建有向关系 |
| `get_relationships(node_id, rel_type, direction, limit)` | `List[dict]` | 获取某节点的关系 |
| `delete_relationship(rel_id)` | `bool` | 按 ID 删除关系 |
| `execute_query(query, parameters)` | `dict` | 执行原始 Cypher，返回 `{"success", "records", "keys", "metadata"}` |
| `query(query, parameters)` | `List[dict]` | 执行 Cypher 并直接返回记录列表 |
| `create_index(label, property_name)` | `bool` | 创建索引以加速查找 |
| `get_neighbors(node_id, rel_type, direction, depth)` | `List[dict]` | 获取邻居节点 |
| `shortest_path(start_node_id, end_node_id, rel_type, max_depth)` | `dict \| None` | 查找两节点间的最短路径 |
| `get_stats()` | `dict` | 从后端获取图统计信息 |


<a id="backends"></a>
## 后端

<Tabs>
  <Tab title="Neo4j（推荐）">
    ```bash
    pip install neo4j
    ```

    ```python
    from semantica.graph_store import GraphStore

    store = GraphStore(
        backend="neo4j",
        uri="bolt://localhost:7687",
        user="neo4j",
        password="password",
        database="neo4j",    # 可选：指定默认数据库
    )
    store.connect()
    ```

    **最适用于：** 生产工作负载、复杂 Cypher 查询、Bloom 可视化。
  </Tab>
  <Tab title="FalkorDB">
    ```bash
    pip install falkordb
    ```

    ```python
    store = GraphStore(
        backend="falkordb",
        host="localhost",
        port=6379,
        graph_name="semantica",
    )
    store.connect()
    ```

    **最适用于：** 通过 Redis 协议实现超低延迟查询、边缘部署。
  </Tab>
  <Tab title="Apache AGE">
    ```bash
    pip install psycopg2-binary
    ```

    ```python
    store = GraphStore(
        backend="age",   # 或 "apache_age"
        connection_string="host=localhost dbname=agedb user=postgres password=secret",
        graph_name="semantica",
    )
    store.connect()
    ```

    **最适用于：** 已在使用 PostgreSQL、希望无需独立服务即可进行图查询的团队。

    <Warning>
      **Apache AGE 需要安装 PostgreSQL 扩展。** `backend="age"` 会调用 AGE 扩展函数。如果你的 PostgreSQL 实例中未安装 AGE，将会得到 `ProgrammingError`。参见 [Apache AGE 文档](https://age.apache.org/age-manual/master/intro/setup.html)了解安装步骤。
    </Warning>
  </Tab>
  <Tab title="Amazon Neptune">
    ```bash
    pip install neo4j boto3
    ```

    ```python
    store = GraphStore(
        backend="neptune",   # 或 "amazon_neptune"
        endpoint="your-cluster.cluster-xxxx.us-east-1.neptune.amazonaws.com",
        port=8182,
        region="us-east-1",
        iam_auth=True,    # 使用 boto3 默认凭证链
    )
    store.connect()

    # 通过 Bolt 协议使用 OpenCypher 查询
    results = store.query(
        "MATCH (p:Person)-[:WORKS_FOR]->(o:Organization) RETURN p, o"
    )
    ```

    **最适用于：** 托管式 AWS 部署。Neptune 使用 Bolt 协议进行 OpenCypher 查询：与 Neo4j 使用相同的查询 API。

    <Warning>
      **Amazon Neptune 使用 `iam_auth=`，而非 `use_iam_auth=`。** `AmazonNeptuneStore` 与 `GraphStore` 的 Neptune 后端都使用 `iam_auth: bool = True` 作为参数名。
    </Warning>
  </Tab>
  <Tab title="后端对比">

    | 后端 | 查询语言 | 部署方式 | IAM 认证 | 最适用于 |
    | :------- | :-------------- | :---------- | :-------- | :-------- |
    | Neo4j | Cypher | 自托管 / Aura | 否 | 生产、复杂遍历、Bloom UI |
    | FalkorDB | OpenCypher | 基于 Redis | 否 | 超低延迟、边缘部署 |
    | Apache AGE | OpenCypher | PostgreSQL 扩展 | 否 | 已使用 Postgres 的团队 |
    | Amazon Neptune | OpenCypher | AWS 托管 | 是 | 云原生、托管、合规 |

  </Tab>
</Tabs>

<a id="graph-operations"></a>
## 图操作

```python
# 创建单个节点
node = store.create_node(
    labels=["Organization"],
    properties={"name": "Apple Inc.", "founded": 1976},
)

# 创建有向关系（需要两个节点 ID）
rel = store.create_relationship(
    start_node_id=jobs_id,
    end_node_id=node["id"],
    rel_type="FOUNDED",
    properties={"year": 1976},
)

# 批量创建节点
store.create_nodes([
    {"labels": ["Person"],       "properties": {"name": "Steve Jobs"}},
    {"labels": ["Organization"], "properties": {"name": "NeXT"}},
])

# 更新节点属性（merge=True 合并，merge=False 替换）
store.update_node(node["id"], {"employees": 164000}, merge=True)

# 删除（detach=True 同时删除相连的关系）
store.delete_node(node["id"], detach=True)
store.delete_relationship(rel["id"])

# 获取邻居
neighbors = store.get_neighbors(
    node["id"],
    rel_type="HAS_EMPLOYEE",
    direction="in",    # "in" | "out" | "both"
    depth=1,
)

# 最短路径：返回 {"length", "nodes", "relationships"} 或 None
path = store.shortest_path(
    start_node_id=jobs_id,
    end_node_id=cook_id,
    rel_type="WORKS_WITH",
    max_depth=5,
)
if path:
    print(f"Hops: {path['length']}")
```

<Tip>
  **使用 `create_nodes()` 进行批量加载。** 逐条 `create_node()` 调用每次都会发起一次网络往返。在初始构建图时，`create_nodes(list)` 更快。
</Tip>

<a id="queryengine"></a>
## QueryEngine

`QueryEngine` 负责查询执行与可选缓存。通过 `store.query_engine` 访问：

```python
from semantica.graph_store import GraphStore

store  = GraphStore(backend="neo4j", uri="bolt://localhost:7687", user="neo4j", password="password")
store.connect()

# 从 store 中获取查询引擎
engine = store.query_engine

# 执行参数化查询
result = engine.execute(
    "MATCH (p:Person) WHERE p.department = $dept RETURN p",
    parameters={"dept": "Engineering"},
)
# result 为 {"success": True, "records": [...], "keys": [...], "metadata": {...}}

# 启用缓存：重复的相同调用会返回缓存结果
result = engine.execute(
    "MATCH (p:Person) RETURN count(p) as total",
    use_cache=True,
)

# 清除缓存结果
engine.clear_cache()

# 切换缓存
engine.disable_cache()
engine.enable_cache()
```

<a id="queryengine-methods"></a>
### QueryEngine 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `execute(query, parameters, use_cache)` | `dict` | 执行 Cypher，返回 `{success, records, keys, metadata}` |
| `clear_cache()` | `None` | 清空所有缓存的查询结果 |
| `enable_cache()` | `None` | 打开缓存（默认开启） |
| `disable_cache()` | `None` | 关闭缓存 |

<Tip>
  **在读密集型工作负载中使用 `QueryEngine` 缓存。** 通过 `store.query_engine` 访问引擎。调用 `engine.execute(query, use_cache=True)` 可在进程内缓存相同查询。在写入导致结果失效后，调用 `engine.clear_cache()`。
</Tip>

<Warning>
  **使用参数化查询，切勿使用字符串拼接。** `store.query("WHERE n.name = $name", parameters={"name": user_input})` 可防止 Cypher 注入攻击。绝不要使用 `f"WHERE n.name = '{user_input}'"`。
</Warning>


<a id="graphanalytics"></a>
## GraphAnalytics

通过 `store._manager.analytics` 访问分析功能，或直接用后端 store 实例构造：

```python
from semantica.graph_store import GraphStore, GraphAnalytics

store = GraphStore(backend="neo4j", uri="bolt://localhost:7687", user="neo4j", password="password")
store.connect()

# GraphAnalytics 接收的是后端 store，而非 GraphStore 门面
analytics = GraphAnalytics(store._store_backend)

# 度中心性：返回按度数降序排列的 {"id", "degree"} 字典列表
scores = analytics.degree_centrality(
    labels=["Person"],
    rel_type="KNOWS",
    direction="both",    # "in" | "out" | "both"
)
for entry in scores[:5]:
    print(f"Node {entry['id']}: degree {entry['degree']}")

# 连通分量（Neo4j 需要 GDS，进程内则使用 NetworkX）
components = analytics.connected_components(labels=["Person"])

# 最短路径：返回 {"length", "nodes", "relationships"} 或 None
path = analytics.shortest_path(
    start_node_id=alice_id,
    end_node_id=charlie_id,
    rel_type="KNOWS",
    max_depth=4,
)

# 邻居遍历
neighbors = analytics.get_neighbors(
    node_id=alice_id,
    rel_type="KNOWS",
    direction="out",
    depth=2,
)
```

<a id="graphanalytics-methods"></a>
### GraphAnalytics 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `degree_centrality(labels, rel_type, direction)` | `List[dict]` | 基于度数的节点重要性：记录按度数降序排列 |
| `connected_components(labels)` | `List[dict]` | 连通分量分配（Neo4j 需要 GDS） |
| `shortest_path(start_node_id, end_node_id, rel_type, max_depth)` | `dict \| None` | 返回含 `length`、`nodes`、`relationships` 的路径，或 `None` |
| `get_neighbors(node_id, rel_type, direction, depth)` | `List[dict]` | 最多 `depth` 跳的邻居节点 |

<Note>
  `betweenness_centrality()`、`pagerank()`、`detect_communities()` 和 `all_paths()` 尚未实现。如需这些算法，请通过 `store.execute_query()` 直接调用 Neo4j GDS 过程。
</Note>


<a id="schema-management"></a>
## 模式管理

```python
# 为快速的标签-属性查找创建索引：使用 property_name= 而非 property=
store.create_index(label="Person",       property_name="name")
store.create_index(label="Organization", property_name="id")

# 图统计信息
stats = store.get_stats()
```

<Warning>
  **在批量加载之前创建索引。** `store.create_index(label="Person", property_name="name")` 可让基于 `name` 的 `MATCH` 查询快上数个数量级。没有索引时，每次查询都会做全表扫描。请先创建索引，再加载数据。
</Warning>

<Warning>
  **`create_index` 的参数是 `property_name=`，而非 `property=`。** `store.create_index(label="Person", property_name="name")`：使用 `property=` 会被静默忽略。
</Warning>

<a id="common-workflows"></a>
## 常见工作流

<Tabs>
  <Tab title="基于 KG 数据构建">
    ```python
    from semantica.graph_store import GraphStore

    store = GraphStore(backend="neo4j", uri="bolt://localhost:7687", user="neo4j", password="password")
    store.connect()

    # 先创建索引以提升速度
    store.create_index("Person",       property_name="name")
    store.create_index("Organization", property_name="name")

    # 将实体作为节点加载
    created = store.create_nodes([
        {"labels": [e["type"]], "properties": {"name": e["text"], "id": e["id"]}}
        for e in entities
    ])

    # 将实体 ID 映射到后端节点 ID
    id_map = {e["id"]: node["id"] for e, node in zip(entities, created)}

    # 加载关系
    for rel in relationships:
        if rel["source_id"] in id_map and rel["target_id"] in id_map:
            store.create_relationship(
                start_node_id=id_map[rel["source_id"]],
                end_node_id=id_map[rel["target_id"]],
                rel_type=rel["type"],
            )

    store.close()
    ```
  </Tab>
  <Tab title="参数化查询">
    ```python
    # 始终使用参数，切勿使用字符串拼接
    results = store.query(
        "MATCH (p:Person)-[:WORKS_FOR]->(o:Organization) "
        "WHERE o.name = $org AND p.role = $role RETURN p.name",
        parameters={"org": user_input_org, "role": user_input_role},
    )
    ```
  </Tab>
  <Tab title="邻居遍历">
    ```python
    # 沿 KNOWS 关系走 2 跳
    neighbors = store.get_neighbors(
        node_id=alice_id,
        rel_type="KNOWS",
        direction="out",
        depth=2,
    )
    for n in neighbors:
        print(n["properties"]["name"])
    ```
  </Tab>
  <Tab title="Apache AGE 说明">
    AGE 对每个顶点只支持一个主标签。如果你传入多个标签，第一个会用作 AGE 标签，其余存储在 `labels` 属性数组中。参数化查询在内部使用字面量内联（AGE 不支持在 `cypher()` 调用中绑定 `$param`）：store 会自动处理转义。
  </Tab>
</Tabs>

- [KG 模块](kg.zh-CN.md) —— 在持久化之前先构建图。
- [三元组库](triplet_store.zh-CN.md) —— 用于语义 Web 和 SPARQL 查询的 RDF 三元组库。
- [可视化](visualization.zh-CN.md) —— 可视化任意后端中存储的图。
- [上下文](context.zh-CN.md) —— AgentContext 使用 GraphStore 进行记忆检索。
