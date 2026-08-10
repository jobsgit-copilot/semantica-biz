---
title: "Explorer 配置"
description: "安装 Explorer 扩展依赖，将 ContextGraph 保存为 JSON，并启动交互式浏览器仪表盘。"
icon: "map"
---

**[English](explorer-setup.md)** · **简体中文（当前）**

**`semantica-explorer`** 是一个用于知识图谱探索的**交互式浏览器仪表盘**。你给它一个图谱文件，它会启动一个本地服务器，并打开一个浏览器标签页，在其中你可以搜索节点、查找路径、检查溯源并运行图分析：启动后无需编写任何代码。

本页涵盖了从零开始到运行 Explorer 所需的一切。如需完整的 REST API 参考和端点目录，请参阅 [Explorer 参考](reference/explorer.zh-CN.md)。


<a id="prerequisites"></a>
## 前置条件

Explorer 依赖 FastAPI 和 uvicorn，它们不包含在基础安装中：

```bash
pip install semantica[explorer]
```

<Note>
  仅 `pip install semantica` 是不够的。在没有安装 `[explorer]` 扩展依赖的情况下运行 `semantica-explorer` 会立即打印错误并以退出码 1 退出。
</Note>

验证：

```bash
semantica-explorer --help
```

你应该看到带有四个可用标志的用法消息。如果看到 `command not found`，请先激活你的虚拟环境。如需 PATH 帮助，请参阅 [CLI 配置](cli-setup.zh-CN.md#故障排查)。


<a id="minimal-end-to-end-example"></a>
## 最小化端到端示例

以下四步就是让 Explorer 运行起来所需的全部操作：

```python
from semantica.context import ContextGraph

# 1. 创建图谱
graph = ContextGraph()

# 2. 添加节点
graph.add_node("python", "language", content="Python programming language")

# 3. 保存到磁盘
graph.save_to_file("my_graph.json")
```

```bash
# 4. 启动 Explorer
semantica-explorer --graph my_graph.json
```

浏览器会在 `http://127.0.0.1:8000` 打开。健康检查端点可确认服务器已启动：

```bash
curl http://127.0.0.1:8000/api/health
# {"status": "ok"}
```


<a id="step-1-build-and-save-a-contextgraph"></a>
## 第 1 步：构建并保存 ContextGraph

Explorer 从磁盘上的 JSON 文件加载图谱。你需要先创建该文件。

<Steps>
  <Step title="构建图谱">
    ```python
    from semantica.context import ContextGraph

    graph = ContextGraph()

    # add_node(node_id, node_type, content=None, **properties)
    graph.add_node("python",  "language",  content="Python programming language")
    graph.add_node("fastapi", "framework", content="FastAPI web framework")
    graph.add_node("guido",   "person",    content="Guido van Rossum")

    # add_edge(source_id, target_id, edge_type="related_to", weight=1.0, **properties)
    graph.add_edge("python",  "fastapi", "enables")
    graph.add_edge("guido",   "python",  "created")
    ```
  </Step>
  <Step title="保存为 JSON 文件">
    ```python
    graph.save_to_file("my_graph.json")
    ```

    `save_to_file` 将一个包含 `graph_id`、`nodes`、`edges` 和 `links` 的 JSON 对象写入指定路径。
  </Step>
  <Step title="验证文件可加载（可选的健全性检查）">
    ```python
    from semantica.context import ContextGraph

    check = ContextGraph()
    check.load_from_file("my_graph.json")
    print(check.stats())
    # {
    #   "node_count": 3,
    #   "edge_count": 2,
    #   "node_types": {"language": 1, "framework": 1, "person": 1},
    #   "edge_types": {"enables": 1, "created": 1},
    #   "density": ...
    # }
    ```

    如果这段代码运行无错误，Explorer 就能成功加载该文件。
  </Step>
</Steps>

<Tip>
  已经有流水线运行产出的图谱？直接跳到第 2 步。唯一的要求是该文件是通过 `ContextGraph.save_to_file()` 保存的。
</Tip>


<a id="step-2-launch-explorer"></a>
## 第 2 步：启动 Explorer

```bash
semantica-explorer --graph my_graph.json
```

启动过程会打印：

```
✓ Graph loaded: 3 nodes, 2 edges
╭─ Semantica Explorer · http://127.0.0.1:8000 ─╮
│  API docs  http://127.0.0.1:8000/docs         │
│  Health    http://127.0.0.1:8000/api/health   │
╰───────────────────────────────────────────────╯
```

服务器启动后不久，浏览器会自动在 `http://127.0.0.1:8000` 打开。


<a id="cli-flags"></a>
## CLI 标志

`semantica-explorer` 恰好接受四个标志：

| 标志 | 简写 | 默认值 | 说明 |
| :---- | :----- | :------- | :----------- |
| `--graph` | `-g` | *(**必需**)* | ContextGraph JSON 文件路径 |
| `--port` | `-p` | `8000` | 服务器绑定的端口 |
| `--host` |: | `127.0.0.1` | 服务器绑定的主机 |
| `--no-browser` |: | 关闭 | 不自动打开浏览器标签页 |

没有用于身份认证、日志级别或 TLS 的标志。CLI 中未实现这些功能。

<a id="examples"></a>
### 示例

```bash
# 最小化：仅本地，端口 8000，自动打开浏览器
semantica-explorer --graph my_graph.json

# 简写标志
semantica-explorer -g my_graph.json -p 8080

# 在网络上暴露，以便其他机器连接
semantica-explorer --graph my_graph.json --host 0.0.0.0 --port 8080

# 无头模式：跳过自动打开，手动导航
semantica-explorer --graph my_graph.json --no-browser
```

<Warning>
  `--host 0.0.0.0` 会让 Explorer 在每个网络接口上都可访问。该服务器没有内置身份认证。请仅在受信任的私有网络中使用。
</Warning>


<a id="browser-access"></a>
## 浏览器访问

服务器运行后：

| URL | 你会得到什么 |
| :--- | :------------ |
| `http://127.0.0.1:8000` | 交互式仪表盘 |
| `http://127.0.0.1:8000/docs` | Swagger UI：每个 REST 端点，可交互 |
| `http://127.0.0.1:8000/api/health` | 健康检查：`{"status": "ok"}` |

浏览器标签页会在启动后不久打开。如果没有打开，请手动导航到该 URL，或者传递 `--no-browser` 自己打开。


<a id="running-as-a-python-module"></a>
## 以 Python 模块方式运行

如果 `semantica-explorer` 不在 `PATH` 上，请使用模块形式：

```bash
python -m semantica.explorer --graph my_graph.json --port 8080
```


<a id="common-startup-errors"></a>
## 常见启动错误

<AccordionGroup>

<Accordion title="错误：找不到图谱文件" icon="file-circle-xmark">

传递给 `--graph` 的路径必须指向一个已存在的文件。CLI 会在加载任何内容之前用 `os.path.isfile()` 进行检查。

```bash
# 确认文件存在
ls my_graph.json                                          # Linux / Mac
dir my_graph.json                                         # Windows

# 如有需要，使用完整路径
semantica-explorer --graph /absolute/path/to/my_graph.json
```

</Accordion>

<Accordion title="错误：需要 uvicorn" icon="triangle-exclamation">

未随基础包一起安装 `[explorer]` 扩展依赖：

```bash
pip install semantica[explorer]
```

</Accordion>

<Accordion title="Explorer 启动了但显示零个节点" icon="circle-xmark">

文件已加载但不包含任何节点。用 Python 验证：

```python
from semantica.context import ContextGraph
g = ContextGraph()
g.load_from_file("my_graph.json")
print(g.stats())  # 检查 node_count
```

`node_count` 为 `0` 表示图谱是在添加任何节点之前保存的。请确保在 `save_to_file()` 之前调用了 `add_node()`。

</Accordion>

<Accordion title="从其他机器连接被拒绝" icon="network-wired">

默认的 `--host 127.0.0.1` 仅接受来自同一台机器的连接。要允许远程访问：

```bash
semantica-explorer --graph my_graph.json --host 0.0.0.0
```

</Accordion>

<Accordion title="浏览器标签页未打开" icon="browser">

在无头、SSH 和容器环境中属于预期行为。传递 `--no-browser` 以抑制该警告，然后在能够访问该服务器的浏览器中打开 `http://127.0.0.1:8000`。

</Accordion>

</AccordionGroup>


<a id="what-explorer-gives-you"></a>
## Explorer 提供的能力

运行后，Explorer 暴露一个 REST API 和仪表盘，用于：

- **节点和边搜索**：按 ID、类型和内容对所有节点进行索引搜索
- **邻域扩展**：检查可达可配置跳数深度的邻居
- **路径查找**：任意两个节点之间的 BFS 最短路径
- **图分析**：中心性、社区发现、连通性
- **决策与溯源**：查询已记录的决策及其因果链
- **导入 / 导出**：上传 JSON 或 CSV 以扩展图谱；下载当前状态

完整的端点目录记录在 `/docs` 的 Swagger UI 以及下面的参考页中。

- [Explorer 参考](reference/explorer.zh-CN.md) — 每个 REST 端点、WebSocket 事件、图分析以及所有支持的标志。
- [CLI 配置](cli-setup.zh-CN.md) — 全部五个 Semantica 可执行文件及其各自适用场景。
- [Context 模块](reference/context.zh-CN.md) — ContextGraph 的完整文档：构建、查询、保存和加载。
- [快速上手](quickstart.zh-CN.md) — 端到端流水线：摄取 → 抽取 → 构建图谱 → 导出。
