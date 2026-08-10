---
title: "Explorer"
description: "用于知识图谱探索、本体管理和图分析的交互式 FastAPI 仪表板。"
icon: "map"
---

**[English](explorer.md)** · **简体中文（当前）**

**`semantica.explorer`** 是一个**基于浏览器的仪表板**，用于探索知识图谱、管理本体并运行可视化分析：

- 索引搜索：在 118k 节点上为 0.004ms：无需全表扫描
- 本体中心：可视化编辑器、SHACL Studio、对齐编写，以及健康仪表板
- 任意两个节点之间的双向路径查找
- 用于实时流水线监控的 WebSocket 进度流
- 启动后无需写代码：在浏览器中完成全部图探索


<a id="getting-started"></a>
## 入门

安装、将图导出为 JSON，然后启动：

```bash
pip install "semantica[explorer]"
```

```python
# 1. 将你的图导出为 JSON 文件
import json
from semantica.context import ContextGraph

graph = ContextGraph()
graph.add_node("Python",  "language",  properties={"paradigm": "multi-paradigm"})
graph.add_node("FastAPI", "framework", properties={"language": "Python"})
graph.add_edge("Python", "FastAPI", "enables")
graph.save_to_file("my_graph.json")
```

```bash
# 2. 启动 Explorer
semantica-explorer --graph my_graph.json
# → Loading graph...
# → Graph loaded: 2 nodes, 1 edges
# → Semantica Explorer · http://127.0.0.1:8000
#     API docs  http://127.0.0.1:8000/docs
#     Health    http://127.0.0.1:8000/api/health
```

浏览器会自动打开 `http://127.0.0.1:8000`。交互式 API 文档位于 `/docs`。

<Tip>
  `semantica.explorer` 是一个**服务端进程**，而不是可导入的 Python 库。请使用 CLI 或 `python -m semantica.explorer` 来启动。`app.py` 模块对外暴露了一个模块级的 `app` 实例，可与 uvicorn 或 Docker 配合使用。
</Tip>

<a id="launch"></a>
## 启动

<Steps>
  <Step title="保存你的图并启动 Explorer">
    ```python
    from semantica.context import ContextGraph

    graph = ContextGraph()
    graph.load_from_file("my_graph.json")   # 校验图能否加载
    ```

    ```bash
    semantica-explorer --graph my_graph.json
    # 服务于 http://127.0.0.1:8000
    ```
  </Step>
  <Step title="自定义主机和端口">
    ```bash
    # 在网络上暴露
    semantica-explorer --graph my_graph.json --host 0.0.0.0 --port 8080

    # 跳过自动打开浏览器
    semantica-explorer --graph my_graph.json --no-browser
    ```
  </Step>
  <Step title="无需重启即可导入新数据">
    ```bash
    # 将 JSON 或 CSV 文件导入到运行中的会话
    curl -X POST http://localhost:8000/api/import \
      -F "file=@updated_graph.json"
    ```
  </Step>
  <Step title="通过 Python 模块使用">
    ```bash
    python -m semantica.explorer --graph my_graph.json --port 8080
    ```
  </Step>
</Steps>

<a id="cli-reference"></a>
## CLI 参考

`semantica-explorer` 命令接受正好四个标志：

| 标志 | 简写 | 默认值 | 描述 |
| :---- | :----- | :------- | :----------- |
| `--graph` | `-g` | *（**必填**）* | 要加载的 ContextGraph JSON 文件路径 |
| `--port` | `-p` | `8000` | 服务绑定端口 |
| `--host` |: | `127.0.0.1` | 服务绑定的主机：使用 `0.0.0.0` 在网络上暴露 |
| `--no-browser` |: | off | 跳过自动打开浏览器标签页 |

<Note>
  CLI 中没有用于身份认证、CORS 或日志级别的标志。CORS 允许来源通过 `EXPLORER_CORS_ORIGINS` 环境变量配置（逗号分隔，默认：`http://localhost:5173,http://127.0.0.1:5173`）。
</Note>

<Tip>
  **CORS 来源通过环境变量配置。** 在启动前将 `EXPLORER_CORS_ORIGINS` 设置为允许来源的逗号分隔列表（例如 `EXPLORER_CORS_ORIGINS="http://myapp.example.com"`）。
</Tip>

```bash
# 完整示例
EXPLORER_CORS_ORIGINS="http://myapp.example.com" \
  semantica-explorer --graph my_graph.json --host 0.0.0.0 --port 8080 --no-browser
```

<a id="what-you-get"></a>
## 你会得到什么

- **图浏览器（Graph Explorer）** —— 交互式节点/边搜索、路径查找和邻域扩展。在 118k 节点图上索引搜索达 0.004ms。
- **本体中心（Ontology Hub）** —— SKOS 词表管理、SHACL 形状生成与校验、本体对齐、健康仪表板，以及版本管理。
- **分析（Analytics）** —— 度中心性、社区发现、连通性分析、图校验，以及距离矩阵。
- **REST API** —— 所有功能均以 REST API 提供：完整文档位于 `/docs`。
- **WebSocket 更新** —— 通过 WebSocket 在 `/ws/graph-updates` 实时流式推送图变更事件。
- **CLI 启动器** —— `semantica-explorer --graph my_graph.json` 一键本地启动。

<a id="features"></a>
## 功能特性

<Tabs>
  <Tab title="图浏览器">
    用于导航知识图谱的核心仪表板：

    - **索引搜索**：POST 到 `/api/graph/search` 并附带查询；在 118k 节点图上为 0.004ms
    - **路径查找**：在任意两个节点之间进行 BFS 或 Dijkstra，通过 `GET /api/graph/path?source=&target=`
    - **邻居扩展**：`GET /api/graph/node/{id}/neighbors?depth=2`
    - **按实体类型过滤**：`GET /api/graph/nodes?type=Person`
    - **语义邻域**：`GET /api/graph/semantic-neighborhood?node_id=&top_k=20`
    - **距离矩阵**：`POST /api/graph/distance-matrix`

    <Warning>
      **保存为 JSON 前先过滤大型图。** CLI 会将整个 JSON 文件加载到内存中。对于超过 10k 节点的图，请在导出前过滤到相关子图：力导向布局在超大图上会变得不可用。
    </Warning>
  </Tab>
  <Tab title="本体中心">
    在浏览器中完成本体的生命周期管理：

    - **注册表**：`GET /api/ontology/registry`：列出已加载的本体
    - **SKOS 词表**：`GET /api/ontology/skos/schemes`、`GET /api/ontology/skos/concept/{uri}`
    - **SHACL**：`POST /api/ontology/shacl/generate`、`POST /api/ontology/shacl/validate`
    - **对齐**：`GET/POST /api/ontology/alignments`、`POST /api/ontology/suggest-alignments`
    - **提案与版本管理**：`POST /api/ontology/propose`、`GET /api/ontology/versions/{uri}`
    - **健康**：`GET /api/ontology/health`
  </Tab>
  <Tab title="分析">
    针对已加载图运行的图指标：

    - **组合指标**：`GET /api/analytics?metrics=centrality,community,connectivity`
    - **图校验**：`GET /api/analytics/validation`
    - **增强：链接预测**：`POST /api/enrich/links`
    - **增强：去重**：`POST /api/enrich/dedup`
    - **增强：实体抽取**：`POST /api/enrich/extract`
    - **时序**：`GET /api/temporal/snapshot`、`GET /api/temporal/diff`、`GET /api/temporal/bounds`

    <Tip>
      **使用 `/api/analytics/validation` 检查图质量。** 校验器会在你将图暴露给下游流水线之前检测出孤儿节点、缺失类型以及其他结构性问题。
    </Tip>
  </Tab>
  <Tab title="决策与溯源">
    决策追踪与溯源查询：

    - **决策**：`GET /api/decisions`、`GET /api/decisions/{id}`、`GET /api/decisions/{id}/chain`
    - **先例**：`GET /api/decisions/{id}/precedents`
    - **因果距离**：`GET /api/decisions/causal-distance?source=&target=`
    - **合规**：`GET /api/decisions/{id}/compliance`
    - **溯源**：`GET /api/provenance?node_id=`、`GET /api/provenance/report?node_id=`
    - **标注**：`GET/POST /api/annotations`、`DELETE /api/annotations/{id}`

    <Tip>
      **将 REST API 用于自动化，将 Explorer UI 用于探索。** Explorer 的 REST 端点是稳定的可编程 API：可将它们接入脚本来完成批量标注、SPARQL 查询或导出的自动化。
    </Tip>
  </Tab>
</Tabs>

<a id="api-endpoints"></a>
## API 端点

完整交互式文档位于 `http://localhost:8000/docs`。所有端点均接收和返回 JSON。

<AccordionGroup>
  <Accordion title="图端点">

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/graph/stats` | `GET` | 节点数、边数、实体类型分布 |
    | `/api/graph/nodes` | `GET` | 列出节点：`?type=&search=&skip=&limit=&cursor=&bbox=` |
    | `/api/graph/node/{id}` | `GET` | 获取单个节点的全部属性 |
    | `/api/graph/node/{id}/neighbors` | `GET` | 某节点的邻居：`?depth=1`（1–5） |
    | `/api/graph/edges` | `GET` | 列出边：`?type=&source=&target=&skip=&limit=&cursor=` |
    | `/api/graph/path` | `GET` | 最短路径：`?source=&target=&algorithm=bfs&directed=true` |
    | `/api/graph/search` | `POST` | 索引搜索：请求体：`{query, limit, filters, anchor_node}` |
    | `/api/graph/distance-matrix` | `POST` | 成对距离：请求体：`{node_ids, metric}`（最多 50 个节点） |
    | `/api/graph/semantic-neighborhood` | `GET` | 语义邻居：`?node_id=&top_k=20&min_similarity=0.0` |

  </Accordion>
  <Accordion title="分析、增强与时序">

    **分析：**

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/analytics` | `GET` | 图指标：`?metrics=centrality,community,connectivity` |
    | `/api/analytics/validation` | `GET` | 图校验报告 |

    **增强：**

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/enrich/extract` | `POST` | 从文本抽取实体 |
    | `/api/enrich/links` | `POST` | 节点的链接预测 |
    | `/api/enrich/dedup` | `POST` | 重复检测 |
    | `/api/enrich/merge` | `POST` | 合并重复节点 |
    | `/api/reason` | `POST` | 在图上运行推理 |

    **时序：**

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/temporal/snapshot` | `GET` | `?at=ISO8601` 时刻的图快照（默认为当前时刻） |
    | `/api/temporal/diff` | `GET` | 两个时间之间的差异：`?from_time=&to_time=` |
    | `/api/temporal/patterns` | `GET` | 时序活动模式 |
    | `/api/temporal/bounds` | `GET` | 图中最早和最晚的时序边界 |
    | `/api/temporal/distance-history` | `GET` | 距离历史：`?source=&target=` |

  </Accordion>
  <Accordion title="本体、词表与 SPARQL">

    **本体：**

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/ontology/registry` | `GET` | 列出已加载的本体 |
    | `/api/ontology/load` | `POST` | 从 URL 或内容加载本体 |
    | `/api/ontology/create` | `POST` | 创建新本体 |
    | `/api/ontology/search` | `GET` | 搜索本体实体：`?q=term` |
    | `/api/ontology/health` | `GET` | 本体健康与覆盖指标 |
    | `/api/ontology/alignments` | `GET/POST` | 列出或创建本体对齐 |
    | `/api/ontology/suggest-alignments` | `POST` | AI 建议的对齐 |
    | `/api/ontology/shacl/generate` | `POST` | 生成 SHACL 形状 |
    | `/api/ontology/shacl/validate` | `POST` | 用 SHACL 校验 RDF |
    | `/api/ontology/skos/schemes` | `GET` | 列出 SKOS 概念方案 |
    | `/api/ontology/skos/concept/{uri}` | `GET` | 获取一个 SKOS 概念 |
    | `/api/ontology/proposals` | `GET/POST` | 管理本体变更提案 |
    | `/api/ontology/versions/{uri}` | `GET` | 版本历史 |

    **词表：**

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/vocabulary/schemes` | `GET` | 通过 TripletStore 提供的 SKOS 方案 |
    | `/api/vocabulary/concepts` | `GET` | 某方案中的概念：`?scheme=URI` |
    | `/api/vocabulary/hierarchy` | `GET` | 概念层次树 |
    | `/api/vocabulary/import` | `POST` | 导入 SKOS/RDF 词表文件 |

    SKOS 层次写入会拒绝 `skos:broader` 与
    `skos:narrower` 关系中出现的环。词表导入在添加节点前会校验
    整个批次，而直接的图/会话边写入则在图存储边界处
    应用同一不变量。

    **SPARQL：**

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/sparql` | `POST` | 执行只读 SPARQL 查询（`SELECT`、`ASK`、`CONSTRUCT` 或 `DESCRIBE`）；`CONSTRUCT`/`DESCRIBE` 以 `subject`、`predicate`、`object` 列返回三元组，`ASK` 返回一个 `result` 布尔列 |

  </Accordion>
  <Accordion title="决策、溯源、标注与导出">

    **决策：**

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/decisions` | `GET` | 分页列出已记录的决策 |
    | `/api/decisions/{id}` | `GET` | 单个决策详情 |
    | `/api/decisions/{id}/chain` | `GET` | 某决策的因果链 |
    | `/api/decisions/{id}/precedents` | `GET` | 相似的过往决策 |
    | `/api/decisions/{id}/compliance` | `GET` | 策略合规检查 |
    | `/api/decisions/causal-distance` | `GET` | 因果距离：`?source=&target=` |

    **溯源：**

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/provenance` | `GET` | 实体溯源谱系：`?node_id=` |
    | `/api/provenance/report` | `GET` | 溯源导出报告：`?node_id=` |

    **标注：**

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/annotations` | `GET` | 列出标注：`?node_id=`（可选） |
    | `/api/annotations` | `POST` | 创建标注（返回 201） |
    | `/api/annotations/{id}` | `DELETE` | 删除标注（返回 204） |

    **导出 / 导入：**

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/export` | `POST` | 导出图，格式为 JSON 或 CSV：请求体：`{format, node_ids}` |
    | `/api/export/distance-enriched` | `POST` | 导出成对距离，格式为 CSV 或 JSONL |
    | `/api/import` | `POST` | 从 `.json` 或 `.csv` 文件导入节点/边（最大 50 MB） |

  </Accordion>
  <Accordion title="健康与信息">

    | 端点 | 方法 | 描述 |
    | :-------- | :------ | :----------- |
    | `/api/health` | `GET` | 返回 `{"status": "ok"}` |
    | `/api/info` | `GET` | 服务器名称、版本、状态 |
    | `/docs` | `GET` | 交互式 Swagger UI：全部端点 |

  </Accordion>
</AccordionGroup>

<a id="websocket-graph-updates"></a>
## WebSocket 图更新

实时的图变更事件会通过 WebSocket 在 `ws://localhost:8000/ws/graph-updates` 上流式推送：

```python
import asyncio
import json
import websockets

async def watch_updates():
    async with websockets.connect("ws://localhost:8000/ws/graph-updates") as ws:
        # 服务端在连接建立时发送一个 ack
        ack = json.loads(await ws.recv())
        print("Connected:", ack)

        # 发送一个 ping 以确认连接仍然存活
        await ws.send("ping")

        async for message in ws:
            event = json.loads(message)
            print("[{}] {}".format(event["event"], event.get("data")))

asyncio.run(watch_updates())
```

WebSocket 消息模式：

```json
{
  "event":     "graph_mutation",
  "data": {
    "event_type":  "ADD_NODE",
    "entity_id":   "node_123",
    "payload":     {}
  },
  "timestamp": "2024-01-15T10:30:00+00:00"
}
```

通过 WebSocket 广播的事件类型包括：`connection_ack`、`pong`，以及 `graph_mutation`（当节点或边通过导入或增强被添加/更新/移除时触发）。发送文本 `"ping"` 即可收到 `pong` 响应。

<Warning>
  **服务端重启后会丢失会话状态。** 没有自动保存。在关闭前请调用 `POST /api/export`（请求体 `{"format": "json"}`）下载当前状态。
</Warning>

<a id="performance"></a>
## 性能

| 场景 | 延迟 |
| :-------- | :------- |
| 节点搜索（118k 节点，已索引） | 0.004ms |
| 邻居扩展（深度 2） | < 5ms |
| BFS 路径（118k 节点） | < 50ms |
| SPARQL SELECT（简单模式） | < 20ms |
| 距离矩阵（50 个节点，语义） | ~2s（带向量嵌入缓存） |

节点搜索索引在启动时构建。对于超过 500k 节点的图，请在连接前预留额外的启动时间。

距离矩阵每次请求上限为 50 对节点。语义距离要求节点在其属性中存有向量嵌入。

<a id="troubleshooting"></a>
## 故障排查

**浏览器标签页未打开**
浏览器会在服务端启动 1.5 秒后打开。如果自动打开失败，请使用 `--no-browser` 并手动打开 `http://127.0.0.1:8000`。

**`Error: graph file not found`**
`--graph` 路径必须是已存在的文件。启动前请检查路径并确保文件存在。

**`Error: uvicorn is required`**
请安装 explorer 扩展：`pip install "semantica[explorer]"`。

**API 调用出现 `Connection refused`**
服务端默认只绑定到 `127.0.0.1`。要从另一台机器或容器访问 Explorer，请使用 `--host 0.0.0.0` 启动。

**导入后图为空**
导入端点（`/api/import`）只解析 `.json` 和 `.csv` 文件。其他格式返回 HTTP 422。JSON 文件必须包含顶层 `entities`/`nodes` 数组或 `relationships`/`edges` 数组。

**`/api/graph/path` 报 `PathFinder not available` 错误**
路径查找需要 `semantica[kg]` 扩展。请用 `pip install "semantica[all]"` 安装。

**语义邻域返回 503**
语义邻域要求节点属性中存有节点向量嵌入（键名为 `embedding`、`vector` 或 `node2vec_embedding`）。不含向量嵌入的图会返回 503。

**重启后会话状态丢失**
会话状态仅存于内存中。请在关闭前使用 `POST /api/export` 保存 JSON 快照。

- [Context](context.zh-CN.md) —— 构建并保存 Explorer 加载的 ContextGraph。
- [本体](ontology.zh-CN.md) —— 以编程方式管理本体并生成 SHACL。
- [可视化](visualization.zh-CN.md) —— 无需 Explorer 服务端即可进行编程式图渲染。
- [导出](export.zh-CN.md) —— 无需启动服务端即可导出为 RDF、Parquet 等格式。
