---
title: "CLI 配置"
description: "五个 Semantica 可执行文件：各自的作用、适用场景以及如何确认其正常工作。"
icon: "terminal"
---

**[English](cli-setup.md)** · **简体中文（当前）**

安装基础包后会在你的 `PATH` 上注册五个可执行文件。每个都有其独特用途。本页说明它们是什么、如何验证它们可用，以及在每种情况下该使用哪一个。


<a id="installed-commands"></a>
## 已安装的命令

```bash
pip install semantica
```

安装后，以下命令可用：

| 命令 | 入口点 | 作用 |
| :------- | :----------- | :------------ |
| `semantica` | `semantica.cli:main` | 用于流水线运行、抽取和图谱操作的通用 CLI |
| `semantica-server` | `semantica.server:main` | 绑定到 `0.0.0.0:8000` 的 FastAPI/uvicorn REST API 服务器 |
| `semantica-worker` | `semantica.worker:main` | Semantica 部署的后台 worker 进程入口点 |
| `semantica-explorer` | `semantica.explorer:main` | 用于知识图谱探索的交互式浏览器仪表盘 |
| `semantica-mcp` | `semantica.mcp_server:main` | 面向 Claude Desktop、Cursor、Windsurf 及其他 MCP 客户端的 MCP 服务器（stdio） |

<Note>
  `semantica-explorer` 需要 `pip install semantica[explorer]`。在没有该扩展依赖的情况下运行它会立即打印错误并退出。完整操作流程请参阅 [Explorer 配置](explorer-setup.zh-CN.md)。
</Note>


<a id="verify-the-installation"></a>
## 验证安装

确认每个命令都可用并打印其用法：

```bash
semantica --help
semantica-server --help
semantica-worker --help
semantica-explorer --help
semantica-mcp --help
```

确认包版本：

```bash
python -c "import semantica; print(semantica.__version__)"
```


<a id="when-to-use-each-command"></a>
## 各命令的适用场景

- **semantica** — 通用 CLI。在 shell 脚本或 CI 作业中用于一次性的流水线运行、实体抽取和图谱操作。
- **semantica-server** — 启动 REST API 服务器。绑定到 `0.0.0.0:8000`。当其他服务或应用需要通过 HTTP 以编程方式访问 Semantica 时使用。
- **semantica-worker** — 后台任务处理器。当你需要在请求周期之外进行异步流水线执行时，与 `semantica-server` 一起运行。先启动服务器，再启动一个或多个指向同一后端的 worker。
- **semantica-explorer** — 启动浏览器仪表盘。需要 `pip install semantica[explorer]`。用于交互式地探索已保存的知识图谱。请参阅 [Explorer 配置](explorer-setup.zh-CN.md)。
- **semantica-mcp** — 通过 stdio 运行 MCP 服务器。在你的 MCP 客户端配置文件中配置它，以将全部 12 个工具和 3 个资源暴露给 Claude Desktop、Cursor、Windsurf 或任何兼容 MCP 的客户端。请参阅 [MCP 服务器](reference/mcp_server.zh-CN.md)。


<a id="usage-examples"></a>
## 用法示例

<Tabs>
  <Tab title="REST 服务器">
    ```bash
    # 在 0.0.0.0:8000 上启动 FastAPI + uvicorn
    semantica-server
    ```

    运行后，用以下命令检查：

    ```bash
    curl http://localhost:8000/health
    # {"status": "ok"}

    curl http://localhost:8000/api/info
    # {"name": "Semantica API", "version": "...", "status": "active"}
    ```

    交互式 API 文档位于 `http://localhost:8000/docs`。
  </Tab>
  <Tab title="Worker">
    ```bash
    semantica-worker
    ```

    该 worker 在收到 `SIGINT`（Ctrl-C）或 `SIGTERM` 时会干净地退出。
  </Tab>
  <Tab title="MCP 客户端配置">
    添加到你的 MCP 客户端配置文件中：

    ```json
    {
      "mcpServers": {
        "semantica": {
          "command": "semantica-mcp"
        }
      }
    }
    ```

    如果命令不在 `PATH` 上，可以使用 Python 模块形式：

    ```json
    {
      "mcpServers": {
        "semantica": {
          "command": "python",
          "args": ["-m", "semantica.mcp_server"]
        }
      }
    }
    ```

    在配置客户端之前，先直接测试它：

    ```bash
    echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | semantica-mcp
    ```

    你应该收到一个 JSON-RPC 响应。完整的工具和资源列表请参阅 [MCP 服务器](reference/mcp_server.zh-CN.md)。
  </Tab>
  <Tab title="Explorer">
    ```bash
    pip install semantica[explorer]
    semantica-explorer --graph my_graph.json
    ```

    完整操作流程（包括如何构建和保存图谱文件）请参阅 [Explorer 配置](explorer-setup.zh-CN.md)。
  </Tab>
  <Tab title="Python 模块形式">
    每个命令也都可以作为 Python 模块运行：当脚本目录不在 `PATH` 上时很有用：

    ```bash
    python -m semantica.mcp_server
    python -m semantica.explorer --graph my_graph.json
    ```
  </Tab>
</Tabs>


<a id="environment-variables"></a>
## 环境变量

`semantica-mcp` 读取两个环境变量：

| 变量 | 默认值 | 说明 |
| :-------- | :------- | :----------- |
| `SEMANTICA_KG_PATH` | *(无)* | 启动时要加载的已保存图谱文件路径 |
| `SEMANTICA_LOG_LEVEL` | `WARNING` | 日志详细程度：`DEBUG`、`INFO`、`WARNING` |

`semantica-server` 读取一个：

| 变量 | 默认值 | 说明 |
| :-------- | :------- | :----------- |
| `SEMANTICA_CORS_ORIGINS` | `http://localhost:5173,http://127.0.0.1:5173` | 允许的 CORS 来源的逗号分隔列表 |

这些命令不会读取其他环境变量。


<a id="troubleshooting"></a>
## 故障排查

<AccordionGroup>

<Accordion title="找不到命令" icon="terminal">

可执行文件位于当前活动 Python 环境的 `bin/`（Linux/Mac）或 `Scripts/`（Windows）目录中。如果找不到命令，该目录很可能不在 `PATH` 上。

先激活你的虚拟环境：

```bash
source venv/bin/activate   # Linux / Mac
venv\Scripts\activate      # Windows
semantica --help
```

查找 pip 放置脚本的位置：

```bash
python -m site --user-scripts   # 用户级安装
pip show -f semantica           # 显示所有已安装文件
```

</Accordion>

<Accordion title="找到命令但在导入时崩溃" icon="triangle-exclamation">

```bash
pip install --upgrade semantica
python -c "import semantica; print(semantica.__version__)"
```

如果你有多个 Python 环境，请安装到 shell 解析到的那个：

```bash
python -m pip install semantica
```

</Accordion>

<Accordion title="semantica-explorer：需要 uvicorn" icon="map">

Explorer 扩展依赖不包含在基础安装中：

```bash
pip install semantica[explorer]
```

</Accordion>

<Accordion title="在 MCP 客户端内 semantica-mcp 静默失败" icon="plug">

MCP 服务器通过 stdio 通信。先直接从 shell 测试它：

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"ping","params":{}}' | semantica-mcp
```

收到 `{"jsonrpc":"2.0","id":1,"result":{}}` 响应即表示服务器工作正常。如果没有看到任何输出，请检查该命令是否在 `PATH` 上以及基础包是否已安装。

</Accordion>

<Accordion title="Windows：启动时出现 DLL 错误" icon="windows">

安装 [Microsoft Visual C++ Redistributable](https://aka.ms/vs/17/release/vc_redist.x64.exe)。这是 PyTorch 及相关包所需的 Windows 系统依赖，不是 Semantica 的 bug。

</Accordion>

</AccordionGroup>


<a id="next-steps"></a>
## 后续步骤

- [Explorer 配置](explorer-setup.zh-CN.md) — 构建图谱、保存并启动浏览器仪表盘。
- [MCP 服务器](reference/mcp_server.zh-CN.md) — 通过 MCP 协议暴露的全部 12 个工具和 3 个资源。
- [安装](installation.zh-CN.md) — 虚拟环境、可选扩展依赖以及平台相关说明。
- [快速上手](quickstart.zh-CN.md) — 附带可运行代码的端到端流水线演示。
