---
title: "MCP 服务器"
description: "将 Semantica 的知识图谱、决策智能和推理能力连接到 Claude Desktop、Windsurf、VS Code、Cline 以及任何兼容 MCP 的 AI 客户端。"
icon: "plug"
---

**[English](mcp-server.md)** · **简体中文（当前）**

<a id="what-is-mcp"></a>
## 什么是 MCP？

MCP 是 Model Context Protocol（模型上下文协议）的缩写。它是一个开放标准，允许外部 AI 助手（如 Claude Desktop、Cursor 或 Windsurf）安全地访问本地工具和数据源。

Semantica MCP 服务器将你的知识图谱暴露为 12 个可调用的工具。通过连接它，任何兼容的 AI 客户端都可以在对话过程中实时遍历图谱、记录决策、运行分析并导出结果，而你无需编写自定义的工具封装。

<Info>
  Semantica MCP 服务器暴露了 12 个工具和 3 个只读资源。所有工具都接受并返回 JSON。除了一个用于图谱持久化的可选环境变量外，无需任何其他配置。
</Info>

<a id="architecture-communication"></a>
## 架构与通信

理解 MCP 的底层工作原理很重要。**Semantica MCP 服务器不是 REST API。** 它没有网络端口、没有 HTTP 端点，也不需要 API 密钥。

相反，AI 客户端会在本地将 `semantica-mcp` 作为子进程启动。AI 与 Semantica 之间的所有通信都通过标准输入输出（`stdio`）安全地进行。由于服务器在你的用户账户下本地运行，它天然拥有你的本地文件权限。

<a id="why-use-mcp-with-semantica"></a>
## 为什么在 Semantica 中使用 MCP？

- **零代码集成**：无需编写任何胶水代码，即可将 Semantica 的图谱能力连接到你最喜欢的 AI IDE 或桌面聊天应用。
- **实时图谱更新**：与 AI 对话，从文档中抽取实体，并看着它们即时填充到你实时的知识图谱中。
- **可审计的 AI**：让 AI 做出决策，并通过 Semantica 的决策智能工具自动将推理过程和因果链直接记录到图谱中。

<a id="when-to-use-when-not-to-use"></a>
## 适用场景 / 不适用场景

- **适用场景**：你希望使用第三方 AI 界面（如 Claude Desktop 或 Windsurf）来操作、查询并对本地机器上的 Semantica 知识图谱进行推理。
- **不适用场景**：你正在构建自主的 Python 脚本或后端服务。如果你在编写 Python 代码来构建智能体，请原生地使用 `semantica.context.AgentContext`，而不是启动 MCP 服务器。MCP 服务器不支持通过 HTTP/SSE 进行远程托管。

---

<a id="typical-workflow"></a>
## 典型工作流

连接 AI 客户端遵循一个标准的流程：

1. **安装**：在你的 Python 环境中安装 Semantica。
2. **配置客户端**：将 `semantica-mcp` 命令和图谱的绝对路径添加到 AI 客户端的 JSON 配置中。
3. **启动客户端**：启动 Claude Desktop 或 Windsurf，它们会自动拉起 MCP 服务器。
4. **工具调用**：用自然语言向 AI 提问。AI 会自主地将 12 个可用工具串联起来。
5. **图谱更新**：AI 直接修改你的本地图谱，添加实体、边和决策。

---

<a id="starting-the-server"></a>
## 启动服务器

安装 Semantica，然后配置你的客户端以启动 MCP 服务器。服务器使用 `stdio` 传输方式运行。

```bash
pip install semantica
```

```bash
# 通过 CLI 入口启动（推荐）
semantica-mcp

# 或者直接通过 Python 模块启动
python -m semantica.mcp_server
```

默认情况下，服务器以 `WARNING` 级别记录日志，并且不产生任何启动输出。设置 `SEMANTICA_LOG_LEVEL=INFO`（或 `DEBUG`）可以在 stderr 上看到启动消息。如果没有设置 `SEMANTICA_KG_PATH`，服务器会初始化一个空的内存图谱，这对于测试来说足够了。要让图谱在重启后依然存在，请设置该路径：

```bash
SEMANTICA_KG_PATH=/data/threat_graph.json semantica-mcp
```

<Info>
  如果没有设置 `SEMANTICA_KG_PATH`，图谱会在服务器进程退出时重置。对于任何希望数据在重启后仍然保留的会话，请始终使用绝对文件路径来设置此路径。
</Info>

<a id="connecting-to-claude-desktop"></a>
## 连接到 Claude Desktop

编辑 Claude Desktop 的配置文件 —— macOS 上位于 `~/Library/Application Support/Claude/claude_desktop_config.json`，Windows 上位于 `%APPDATA%\Claude\claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "semantica": {
      "command": "semantica-mcp",
      "env": {
        "SEMANTICA_KG_PATH": "/absolute/path/to/knowledge_graph.json",
        "SEMANTICA_LOG_LEVEL": "INFO"
      }
    }
  }
}
```

保存后重启 Claude Desktop。Semantica 工具会自动出现在工具面板中 —— Claude 现在可以在任何对话中调用它们。

如果 `semantica-mcp` 不在你的系统 PATH 中（例如，它安装在 virtualenv 中），请在 `"command"` 中使用完整的二进制文件绝对路径：`"/path/to/venv/bin/semantica-mcp"`。

<a id="connecting-to-other-clients"></a>
## 连接到其他客户端

**Windsurf** —— 在 Settings → MCP Servers → Add Server 中，或编辑 `~/.windsurf/mcp_servers.json`：

```json
{
  "semantica": {
    "command": "semantica-mcp",
    "env": { "SEMANTICA_KG_PATH": "/absolute/path/to/knowledge_graph.json" }
  }
}
```

**VS Code（Cline / Roo Code / Continue）** —— 添加到项目根目录下的 `.mcp.json` 中：

```json
{
  "servers": {
    "semantica": {
      "type": "stdio",
      "command": "semantica-mcp",
      "env": {
        "SEMANTICA_KG_PATH": "${workspaceFolder}/knowledge_graph.json",
        "SEMANTICA_LOG_LEVEL": "DEBUG"
      }
    }
  }
}
```

**Docker**：

```bash
docker run --rm -i \
  -e SEMANTICA_KG_PATH=/data/kg.json \
  -v /local/absolute/path:/data \
  ghcr.io/semantica-agi/semantica-mcp:latest
```

<a id="what-the-agent-can-do-the-12-tools"></a>
## 智能体能做什么：12 个工具

连接后，LLM 可以在对话过程中调用这些工具中的任何一个。智能体会自动把它们串联起来 —— 你不需要编排调用顺序，只需描述你想要什么。

**实体与关系抽取** —— `extract_entities` 从自由文本中提取命名实体；`extract_relations` 抽取语义关系和 RDF 三元组。这两个工具无需任何预处理，就能把一份原始的 OSINT 报告转化为结构化的图谱输入。

**知识图谱操作** —— `add_entity` 添加一个节点，`add_relationship` 添加一条有向边。抽取之后，智能体调用这些工具将发现的内容持久化到实时图谱中。

**决策智能** —— `record_decision` 将一个决策写成一个溯源节点，包含置信度分数、推理过程和决策者身份。`query_decisions` 通过查询或类别检索过去的决策。`find_precedents` 通过语义相似度找到最相似的过往决策。`get_causal_chain` 向上游或下游追踪决策的因果关系。

**推理** —— `run_reasoning` 对一组事实应用前向链接的 IF/THEN 规则，并返回推导出的结论。

**分析与导出** —— `get_graph_analytics` 计算 PageRank 中心性和社区检测。`get_graph_summary` 返回节点数、决策数和服务器状态。`export_graph` 将当前图谱序列化为 Turtle（`"turtle"` / `"ttl"`）、RDF/XML（`"xml"`）、N-Triples（`"nt"`）、JSON-LD（`"json-ld"`）或纯 JSON（`"json"`）。

<a id="universal-example-employee-directory"></a>
## 通用示例：员工目录

在深入复杂的领域示例之前，这里有一个简单、大家都容易理解的会话。一位 HR 经理在 Claude Desktop 中输入了一个提示：

> "从这份关于 Alice 调入工程部的会议记录中抽取实体，将它们添加到图谱中，并记录一项晋升决策。"

Claude 自动串联了四次工具调用：

```text
1. extract_entities(text="Alice is transferring to Engineering...")
   → { "entities": [{"label": "Alice", "type": "Employee"}, {"label": "Engineering", "type": "Department"}] }

2. add_entity(id="emp-alice", label="Alice", type="Employee")
   add_entity(id="dept-eng", label="Engineering", type="Department")

3. add_relationship(source="emp-alice", target="dept-eng", type="WORKS_IN")

4. record_decision(
       category="promotion",
       scenario="Alice transferring to Engineering",
       reasoning="Approved by Engineering Director",
       outcome="transfer_approved",
       confidence=1.0
   )
```
图谱会立即更新，包含新的组织结构以及一条完全可审计的决策追踪。

<a id="watching-a-real-agent-session"></a>
## 观看真实的智能体会话

下面是当一名网络安全分析师在 Claude Desktop 中输入提示且图谱处于实时状态时所发生的情况。提示内容是：

> "从这份 OSINT 报告中抽取实体和关系，将它们添加到知识图谱中，然后以 0.88 的置信度记录一项关于 APT29 的归因决策，并将完整图谱导出为 Turtle。"

Claude 自动串联了六次工具调用：

```text
1. extract_entities(text="<report text>")
   → { "entities": [{"label": "APT29", "type": "ThreatActor"}, ...] }

2. extract_relations(text="<report text>")
   → { "relations": [{"source": "APT29", "type": "EXPLOITS", "target": "CVE-2024-3400"}] }

3. add_entity(id="apt29", label="APT29", type="ThreatActor", metadata={"alias": "NOBELIUM"})
   → { "status": "added", "id": "apt29" }
   (repeated for each extracted entity)

4. add_relationship(source="apt29", target="cve-2024-3400", type="EXPLOITS",
                    metadata={"confidence": 0.97})
   → { "status": "added" }
   (repeated for each extracted relation)

5. record_decision(
       category="threat_attribution",
       scenario="C2 beacon from 185.220.101.47, TTP T1566.001 observed",
       reasoning="IP overlaps APT29 infrastructure cluster; TTPs match NOBELIUM phishing playbook",
       outcome="attributed_to_apt29",
       confidence=0.88,
       decision_maker="analyst_zhang"
   )
   → { "decision_id": "dec_a3f2b1", "status": "recorded" }

6. export_graph(format="turtle")
   → { "format": "turtle", "data": "@prefix ... <apt29> a :ThreatActor ..." }
```

分析师只用自然语言问了一个问题。六次结构化的工具调用针对实时图谱数据执行了。最终结果是一个填充好的图谱、一条记录了完整溯源的归因决策，以及一份准备好供 SPARQL 端点使用的 Turtle 导出 —— 所有这一切都在单轮对话中完成。

<a id="the-three-read-only-resources"></a>
## 三个只读资源

资源无需工具调用即可暴露图谱状态 —— 客户端可以随时读取它们：

| URI | 描述 |
| :-- | :---------- |
| `semantica://graph/summary` | 节点数、决策数、服务器状态 |
| `semantica://decisions/list` | 最多 50 条最近记录的决策 |
| `semantica://schema/info` | 服务器版本、能力、可用工具列表 |

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防 — CTI/威胁">

CTI 团队使用 Claude Desktop 将新的 OSINT 报告与现有的威胁图谱进行关联、记录归因决策，并查询因果链 —— 全部通过自然语言完成，并且图谱会实时更新。

**分析师提示：**
> "从这份关于 APT40 的 Mandiant 报告中抽取实体，将它们添加到知识图谱中，查找过往任何关于 APT40 归因的决策，然后以 0.84 的置信度记录一项新的归因决策。"

Claude 自动串联：

1. 对报告文本执行 `extract_entities` + `extract_relations`
2. 对每个抽取出的实体执行 `add_entity`（APT40、CVE、基础设施节点）
3. 对每个抽取出的关系执行 `add_relationship`
4. `query_decisions(query="APT40 attribution", category="threat_attribution")`
5. `find_precedents(scenario="APT40 targeting maritime sector", max_results=3)`
6. `record_decision(category="threat_attribution", outcome="attributed_to_apt40", confidence=0.84, decision_maker="analyst_zhang")`

该归因现在是一个与 OSINT 证据相连的图谱节点，可被未来的智能体检索。

```json
{
  "category": "threat_attribution",
  "scenario": "C2 infrastructure overlaps APT40 cluster; TEMP.Periscope TTPs confirmed",
  "reasoning": "Three C2 IPs match known APT40 hosting ASN; T1190 exploit chain identical to 2023 campaign",
  "outcome": "attributed_to_apt40",
  "confidence": 0.84,
  "decision_maker": "analyst_zhang"
}
```

</Tab>

<Tab title="安全 — SOC/事件">

在一次实时事件中，SOC 使用 Claude 对图谱进行推理、应用零信任策略规则，并记录遏制决策及其因果链 —— 从而创建一条实时审计追踪。

**SOC 分析师提示：**
> "将 WKSTN-047 和 DC01 添加为主机，添加它们之间的横向移动关系，运行推理以对严重性进行分类，并记录一项遏制决策。"

Claude 串联：

1. `add_entity(id="wkstn-047", type="Host", label="WKSTN-047")`
2. `add_entity(id="dc01", type="Host", label="DC01")`
3. `add_relationship(source="wkstn-047", target="dc01", type="lateral_movement")`
4. `run_reasoning(facts=["Host(WKSTN-047)", "LateralMove(WKSTN-047, DC01)", "DC(DC01)"], rules=["IF LateralMove(X, Y) AND DC(Y) THEN CriticalIncident(X)"])`
5. `record_decision(category="containment", scenario="Lateral movement to DC detected", outcome="isolate_wkstn047", confidence=0.95)`
6. `get_causal_chain(decision_id="...", direction="downstream", max_depth=3)`

遏制决策及其下游影响被捕获到图谱中，用于事后复盘。

</Tab>

<Tab title="生命科学 — 临床/制药">

临床 AI 助手使用 MCP 服务器记录带有溯源的治疗决策、检索指南先例，并导出决策图谱以供监管提交和 MDT（多学科诊疗）评审。

**临床提示：**
> "患者 eGFR 为 28，正在服用二甲双胍。查找在肾功能严重受损时修改二甲双胍剂量的先例，然后记录一项治疗调整决策。"

Claude 调用：

1. `find_precedents(scenario="metformin with eGFR below 30", max_results=5)`
2. `record_decision(category="treatment_modification", scenario="eGFR 28, current metformin 1000mg BD", reasoning="eGFR 28 is below the 30 mL/min/1.73m2 absolute contraindication threshold per BNF and NICE NG28", outcome="discontinue_metformin_switch_to_gliclazide", confidence=0.97, decision_maker="clinical_ai_v2")`
3. `get_causal_chain(direction="upstream")` —— 呈现驱动该决策的指南节点
4. `export_graph(format="json-ld")` —— 生成用于 MDT 评审的决策图谱

图谱捕获了该决策、其指南依据以及因果链 —— 全部可检索用于监管审计。

</Tab>

<Tab title="银行业 — 风险/合规">

信用风险团队使用 MCP 服务器记录每一项借贷决策及其推理链，呈现监管先例，并导出合规图谱用于 Basel III 模型治理评审。

**信贷分析师提示：**
> "记录 APP-2025-994421（LTV 78%、DSTI 38%、信用评分 714）的一项有条件抵押贷款批准，查找三个最相似的过往批准，并返回该决策的因果链。"

Claude 调用：

1. `record_decision(category="mortgage_origination", scenario="LTV 78%, DSTI 38%, credit score 714, first-time buyer", reasoning="LTV within 80% cap; DSTI 38% under stressed rate scenario breaches 35% guideline — conditional approval with LMI requirement", outcome="approved_conditional_lmi", confidence=0.89, decision_maker="credit_model_v3")`
2. `find_precedents(scenario="mortgage approval borderline DSTI stress test", max_results=3)`
3. `get_causal_chain(decision_id="...", direction="upstream", max_depth=5)`
4. `export_graph(format="turtle")` —— 生成供模型治理委员会使用的决策溯源图谱

最终结果是一条带有先例链接、完全可审计的信贷决策追踪，可随时用于 SR 11-7 模型风险治理评审。

</Tab>

</Tabs>

---

<a id="common-pitfalls"></a>
## 常见陷阱

- **把 MCP 当作 HTTP 服务器**：不要尝试对 MCP 服务器进行 `curl` 或寻找端口号。它通过 `stdin/stdout` 通信，并等待来自父 AI 客户端的 JSON-RPC 消息。
- **为 `SEMANTICA_KG_PATH` 使用相对路径**：由于 AI 客户端将服务器作为子进程启动，工作目录可能不可预测。请始终使用绝对路径（例如 `C:\Users\Name\graph.json` 或 `/Users/name/graph.json`），以避免数据丢失。
- **虚拟环境 PATH 问题**：如果你将 Semantica 安装在 Python 虚拟环境中，Claude Desktop 不会自动在全局系统 PATH 中找到 `semantica-mcp`。你必须在 `"command"` 字段中提供该二进制文件的绝对路径。
- **期望支持远程托管**：基于 stdio 的 MCP 服务器必须与 AI 客户端运行在同一台本地机器上。不支持通过网络进行远程执行。
- **将 MCP 集成与 `AgentContext` 混淆**：如果你正在编写自己的 Python 代码来编排 LLM，请不要使用 MCP 服务器。请在代码中原生使用 `AgentContext` 类。

---

<a id="troubleshooting"></a>
## 故障排查

**服务器未在 Claude Desktop 中出现** —— 编辑配置后，请完全退出并重新打开 Claude Desktop（仅关闭窗口是不够的）。验证二进制文件是否在 PATH 中：在 Unix 上使用 `which semantica-mcp`，在 Windows 上使用 `where semantica-mcp`。如果使用的是 virtualenv，请在 `"command"` 中使用二进制文件的绝对路径。设置 `SEMANTICA_LOG_LEVEL=DEBUG` 并检查 stderr 以查看启动错误。

**图谱数据在会话之间未持久化** —— 将 `SEMANTICA_KG_PATH` 设置为绝对文件路径。如果没有设置，图谱仅在内存中，并且会在每次服务器重启时重置。

**工具调用返回空结果** —— `get_graph_summary` 返回 `"node_count": 0` 表示图谱为空。请通过 `add_entity` 和 `add_relationship` 填充它，或者先对文本运行 `extract_entities`，然后对每个结果执行 `add_entity`。

**`SEMANTICA_KG_PATH` 上的权限错误** —— 服务器进程需要对该文件及其父目录的读写权限。如果在 Docker 中运行，请验证卷挂载和文件所有权。

<a id="related-guides"></a>
## 相关指南

- [推理与规则](reasoning.zh-CN.md) —— `run_reasoning` 工具背后的引擎
- [决策智能](decision-intelligence.zh-CN.md) —— 决策如何作为因果图谱节点存储
- [上下文图谱](context-graphs.zh-CN.md) —— `add_entity` 和 `add_relationship` 写入的图谱
- [导出与序列化](export.zh-CN.md) —— 通过 `export_graph` 可用的所有导出格式
- [本体管理](ontology.zh-CN.md) —— 从通过 MCP 构建的图谱生成 OWL 本体
