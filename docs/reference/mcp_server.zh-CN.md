---
title: "MCP 服务器"
description: "Model Context Protocol 服务器：将 Semantica 的完整能力集暴露给 Claude Desktop、VS Code、Cursor 以及任何支持 MCP 的工具。"
icon: "plug"
---

**[English](mcp_server.md)** · **简体中文（当前）**

**`semantica.mcp_server`** 将 Semantica 的知识图谱、决策智能、语义抽取与推理能力作为基于 stdio 的 [MCP（Model Context Protocol）](https://modelcontextprotocol.io) **服务器** 暴露出来：

- 暴露 12 个 MCP 工具：抽取实体、查询图、记录决策、运行推理、导出结果
- 启动后无需编写 Python 代码：配置一次，即可从任意支持 MCP 的客户端使用
- 兼容 Claude Desktop、Windsurf、Cline、Continue、VS Code、Roo Code、Cursor

<a id="server-interface"></a>
## 服务器接口

```json
// 在你的 MCP 客户端中配置（Claude Desktop、Windsurf、Cursor、VS Code 等）
{
  "mcpServers": {
    "semantica": {
      "command": "semantica-mcp"
    }
  }
}
```

```bash
# 或直接运行
semantica-mcp
# 或
python -m semantica.mcp_server
```

<Tip>
  `semantica.mcp_server` 是一个 **stdio 服务器进程**，而非 Python 库。它不暴露可导入的类：所有交互都通过已连接 AI 客户端的 MCP 工具调用进行。
</Tip>

<Warning>
  **服务器通过 stdio 通信：不要向 stdout 写入日志。** 任何输出到 stdout 的 `print()` 或 logger 内容都会破坏 JSON-RPC 消息流。所有日志只写入 `stderr`。通过 `SEMANTICA_LOG_LEVEL` 环境变量配置日志详细级别。
</Warning>

<a id="what-you-get"></a>
## 你将获得

- **12 个 MCP 工具** —— 抽取实体、抽取关系、记录决策、查询决策、查找先例、追踪因果链、添加实体、添加关系、运行分析、汇总图、运行推理、导出图。
- **3 个可读资源** —— 实时图 JSON（`semantica://graph/summary`）、决策列表、模式/版本信息：任意 MCP 客户端均可读取。
- **零基础设施** —— 通过 stdio 运行：无需服务器、无需端口、无需 Docker。在任意 MCP 客户端中一个配置块即可激活。
- **持久化图** —— 将 `SEMANTICA_KG_PATH` 指向已保存的图文件，即可在每次服务器启动时自动重新加载。
- **决策智能** —— 记录决策，通过混合相似度搜索查找先例，并跨智能体运行追踪因果链。
- **REST 替代方案** —— 如果你偏好编程式访问，[Explorer](explorer.zh-CN.md) 模块提供完整的 HTTP API 与浏览器仪表板。

<a id="installation"></a>
## 安装

```bash
pip install semantica
```

MCP 服务器包含在基础安装中：无需额外 extras。

<a id="configuration"></a>
## 配置

<Steps>
  <Step title="找到你的 MCP 客户端配置文件">

    | 客户端 | 配置文件 |
    | :------ | :------------- |
    | Claude Desktop（macOS） | `~/Library/Application Support/Claude/claude_desktop_config.json` |
    | Claude Desktop（Windows） | `%APPDATA%\Claude\claude_desktop_config.json` |
    | Cursor | 项目内 `.cursor/mcp.json`，或全局 `~/.cursor/mcp.json` |
    | VS Code / Continue | `.vscode/mcp.json` 或用户设置 |
    | Windsurf / Cline / Roo Code | 应用专属设置 → MCP Servers |

  </Step>
  <Step title="添加 Semantica MCP 服务器配置">

    <CodeGroup>

    ```json Claude Desktop / Windsurf / Cline
    {
      "mcpServers": {
        "semantica": {
          "command": "semantica-mcp"
        }
      }
    }
    ```

    ```json Cursor
    {
      "mcpServers": {
        "semantica": {
          "command": "semantica-mcp",
          "env": {
            "SEMANTICA_KG_PATH": "/path/to/my_graph.json"
          }
        }
      }
    }
    ```

    ```json VS Code / Continue / Roo Code
    {
      "mcpServers": {
        "semantica": {
          "command": "python",
          "args": ["-m", "semantica.mcp_server"]
        }
      }
    }
    ```

    ```json 启用持久化图
    {
      "mcpServers": {
        "semantica": {
          "command": "semantica-mcp",
          "env": {
            "SEMANTICA_KG_PATH": "/path/to/my_graph.json",
            "SEMANTICA_LOG_LEVEL": "INFO"
          }
        }
      }
    }
    ```

    </CodeGroup>

    <Warning>
      **精确配置 MCP 客户端的 `command` 字段。** `command` 字段必须指向确切的执行文件路径（在 macOS/Linux 上使用 `which semantica-mcp` 查找）。错误路径会静默失败：服务器根本不会出现在工具列表中。请先用原始的 `echo | semantica-mcp` 命令测试，以确认二进制可正常运行。
    </Warning>

  </Step>
  <Step title="在配置客户端前先本地测试">
    ```bash
    # 直接运行服务器（从 stdin 读取，写入 stdout）
    semantica-mcp

    # 或通过 Python 模块运行
    python -m semantica.mcp_server

    # 发送一条 JSON-RPC initialize 消息以确认其工作正常
    echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | semantica-mcp
    ```
  </Step>
</Steps>

<a id="environment-variables"></a>
## 环境变量

| 变量 | 默认值 | 描述 |
| :-------- | :------- | :----------- |
| `SEMANTICA_KG_PATH` | *（无：内存图）* | 要在启动时加载的已持久化图文件路径 |
| `SEMANTICA_LOG_LEVEL` | `WARNING` | 日志详细级别：`DEBUG`、`INFO`、`WARNING` |

<Warning>
  **除非设置了 `SEMANTICA_KG_PATH`，否则图是空的。** MCP 服务器在首次使用时创建全新的内存 `ContextGraph`。将 `SEMANTICA_KG_PATH` 指向之前保存的图文件即可在服务器重启之间恢复状态。若不设置，进程退出时所有数据都会丢失。
</Warning>

<Tip>
  **排查问题时启用调试日志。** 在 MCP 客户端的 `env` 块中设置 `SEMANTICA_LOG_LEVEL=DEBUG`，或直接运行 `python -m semantica.mcp_server` 并查看 stderr 输出。
</Tip>

<a id="tools"></a>
## 工具

MCP 服务器暴露 12 个工具，任意已连接的 AI 助手均可调用：

| 工具 | 类别 | 描述 |
| :---- | :-------- | :----------- |
| `extract_entities` | 抽取 | NER：发现人物、地点、组织、概念 |
| `extract_relations` | 抽取 | 类型化关系与三元组抽取 |
| `record_decision` | 决策智能 | 保存带推理与结果的决策 |
| `query_decisions` | 决策智能 | 按自然语言或类别搜索已记录的决策 |
| `find_precedents` | 决策智能 | 对过往决策进行混合相似度搜索 |
| `get_causal_chain` | 决策智能 | 追踪上游 / 下游因果链 |
| `add_entity` | 图操作 | 向实时图中添加节点 |
| `add_relationship` | 图操作 | 在两节点之间添加有向边 |
| `get_graph_summary` | 图操作 | 节点数、决策数、图状态 |
| `get_graph_analytics` | 图操作 | PageRank 中心性与社区发现 |
| `run_reasoning` | 推理 | 对事实进行 IF/THEN 规则前向链接 |
| `export_graph` | 推理与导出 | 序列化图（`turtle`/`ttl`：RDF Turtle 别名，`nt`、`xml`、`json-ld`、`json`） |

<a id="knowledge-extraction"></a>
### 知识抽取

<AccordionGroup>

<Accordion title="extract_entities" icon="tag">

使用 Semantica NER 从文本中抽取命名实体（人物、地点、组织、概念）。

**输入：**

```json
{ "text": "Apple Inc. was founded by Steve Jobs in Cupertino in 1976." }
```

**输出：**

```json
{
  "entities": [
    { "label": "Apple Inc.", "type": "ORGANIZATION", "start": 0,  "end": 10,  "confidence": 0.98 },
    { "label": "Steve Jobs", "type": "PERSON",       "start": 26, "end": 36,  "confidence": 0.99 },
    { "label": "Cupertino",  "type": "LOCATION",     "start": 40, "end": 49,  "confidence": 0.97 },
    { "label": "1976",       "type": "DATE",          "start": 53, "end": 57,  "confidence": 0.95 }
  ],
  "count": 4
}
```

</Accordion>

<Accordion title="extract_relations" icon="arrows-left-right">

从文本中抽取类型化关系与 `(subject, predicate, object)` 三元组。

**输入：**

```json
{ "text": "Steve Jobs founded Apple Inc. and led it until 2011." }
```

**输出：**

```json
{
  "relations": [
    { "source": "Steve Jobs", "type": "founded", "target": "Apple Inc.", "confidence": 0.96 }
  ],
  "triplets": [
    { "subject": "Steve Jobs", "predicate": "founded", "object": "Apple Inc." }
  ],
  "relation_count": 1,
  "triplet_count": 1
}
```

</Accordion>

</AccordionGroup>

<a id="decision-intelligence"></a>
### 决策智能

<AccordionGroup>

<Accordion title="record_decision" icon="check-circle">

将带完整上下文、推理与元数据的决策记录到知识图谱中。

**输入：**

```json
{
  "category": "model_selection",
  "scenario": "Choose LLM for production reasoning pipeline",
  "reasoning": "GPT-4 benchmark advantage justifies 3x cost increase",
  "outcome": "selected_gpt4",
  "confidence": 0.91,
  "decision_maker": "product_team",
  "valid_from": "2024-01-01",
  "valid_until": "2024-12-31"
}
```

必填字段：`category`、`scenario`、`reasoning`、`outcome`、`confidence`。
可选：`decision_maker`（默认为 `"mcp_client"`）、`valid_from`、`valid_until`。

**输出：**

```json
{ "decision_id": "dec_a1b2c3", "status": "recorded" }
```

</Accordion>

<Accordion title="query_decisions" icon="magnifying-glass">

按自然语言或类别过滤查询已记录的决策。

**输入：**

```json
{ "query": "model selection", "category": "model_selection", "limit": 5 }
```

所有字段均为可选。`limit` 默认为 `10`。提供 `query` 时使用相似度搜索；省略时按 `category` 过滤。

</Accordion>

<Accordion title="find_precedents" icon="clock-rotate-left">

使用混合相似度搜索查找与给定场景相似的过往决策。

**输入：**

```json
{ "scenario": "Choose cloud provider for HIPAA workload", "max_results": 5 }
```

`max_results` 默认为 `5`，最大 `50`。

<Tip>
  **在重大决策之前使用 `find_precedents`。** 该工具会对所有已记录决策执行混合相似度搜索。在任意重要决策路径开始时调用它：可呈现出可能直接适用的过往推理，减少重复工作并提升跨智能体运行的一致性。
</Tip>

</Accordion>

<Accordion title="get_causal_chain" icon="diagram-project">

从某个决策追踪上游或下游的因果链。

**输入：**

```json
{ "decision_id": "dec_a1b2c3", "direction": "downstream", "max_depth": 5 }
```

`direction` 接受 `"upstream"` 或 `"downstream"`（默认：`"downstream"`）。
`max_depth` 默认为 `5`，最大 `20`。

</Accordion>

</AccordionGroup>

<a id="graph-operations"></a>
### 图操作

<AccordionGroup>

<Accordion title="add_entity" icon="circle-plus">

向实时知识图谱添加节点/实体。

**输入：**

```json
{
  "id": "apple_inc",
  "label": "Apple Inc.",
  "type": "Organization",
  "metadata": { "founded": 1976, "hq": "Cupertino" }
}
```

只有 `id` 是必填的。`label` 默认为 `id` 的值。`type` 默认为 `"Entity"`。

</Accordion>

<Accordion title="add_relationship" icon="arrow-right">

在两个已存在的实体之间添加有向关系（边）。

**输入：**

```json
{
  "source": "steve_jobs",
  "target": "apple_inc",
  "type": "FOUNDED",
  "metadata": { "year": 1976 }
}
```

`source` 与 `target` 为必填。`type` 默认为 `"RELATED_TO"`。

</Accordion>

<Accordion title="get_graph_summary" icon="info-circle">

返回当前知识图谱的高层概览。

**输出：**

```json
{
  "node_count": 42,
  "decision_count": 5,
  "graph_ready": true
}
```

不接收任何输入参数。

</Accordion>

<Accordion title="get_graph_analytics" icon="chart-bar">

对当前图计算 PageRank 中心性与社区发现。返回按 PageRank 排序的顶级节点、社区数量以及整体的节点/边计数。

不接收任何输入参数。

</Accordion>

</AccordionGroup>

<a id="reasoning"></a>
### 推理

<AccordionGroup>

<Accordion title="run_reasoning" icon="brain">

在一组事实上运行前向链接 IF/THEN 规则以推导新事实。

**输入：**

```json
{
  "facts": ["Employee(John)", "Manager(John)"],
  "rules": ["IF Manager(?x) THEN HasAuthority(?x)"]
}
```

**输出：**

```json
{ "derived_facts": ["HasAuthority(John)"] }
```

</Accordion>

</AccordionGroup>

<a id="export"></a>
### 导出

<AccordionGroup>

<Accordion title="export_graph" icon="file-export">

将当前知识图谱导出为某种序列化格式。

**输入：**

```json
{ "format": "json-ld" }
```

支持的格式：`turtle`、`ttl`、`nt`、`xml`、`json-ld`、`json`。默认为 `json-ld`。

</Accordion>

</AccordionGroup>

<a id="resources"></a>
## 资源

MCP 服务器暴露三个可读资源：

| URI | 描述 |
| :--- | :----------- |
| `semantica://graph/summary` | 高层图统计信息 |
| `semantica://decisions/list` | 所有已记录的决策（最多 50 条） |
| `semantica://schema/info` | 服务器版本与可用工具 |

- [上下文](context.zh-CN.md) —— MCP 服务器所操作的 ContextGraph。
- [语义抽取](semantic_extract.zh-CN.md) —— 支撑 MCP 工具的 NER 与关系抽取。
- [推理](reasoning.zh-CN.md) —— run_reasoning 背后的前向链接引擎。
- [Agno 集成](../integrations/agno.zh-CN.md) —— 在 Agno 多智能体团队中使用 Semantica。
