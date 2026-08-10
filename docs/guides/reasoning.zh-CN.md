---
title: "推理与规则"
description: "在你的知识图谱上应用前向链接、反向链接、Datalog、SPARQL、RETE、时态区间和基于 LLM 的推理——推导新事实、检查约束并解释推断。"
---

**[English](reasoning.md)** · **简体中文（当前）**

Semantica 的推理层将领域逻辑编码为规则，并将其应用于你的知识图谱，从而推导出任何单一文档都未明确陈述的结论。八种互补的推理模式——从符号规则的前向链接到递归 Datalog 和 LLM 支持的自由格式查询——让你无需切换框架即可为每个推断问题选择合适的工具。

<a id="what-is-reasoning"></a>
## 什么是推理？

推理使用逻辑规则从显式事实中推导隐含知识。与检索、搜索或图遍历不同，推理生成原始数据中未直接陈述的新结论。

**推理与检索：**检索查找与查询匹配的现有信息。推理应用逻辑规则推导未显式存储的新事实。

**推理与搜索：**搜索匹配关键词或语义相似性。推理使用逻辑推断得出事实，如"如果 A 蕴含 B，且 A 为真，则 B 必然为真"。

**推理与图遍历：**遍历沿节点间的现有边行进。推理可根据规则推断新关系——例如，即使没有直接边，也能推断两个实体相关。

<a id="why-use-reasoning"></a>
## 为何使用推理？

**推导隐含知识。**文档可能分别陈述"APT29 使用 SUNBURST"和"SUNBURST 利用 CVE-2020-10148"，但推理会自动得出"APT29 利用 CVE-2020-10148"。

**识别模式。**规则可以检测知识图谱中的复杂模式——共享 TTP 的威胁行为体、传递关系中的供应商，或由条件组合产生的合规违规。

**支持调查。**推理通过突出非显而易见的关联、标记满足风险标准的实体以及解释结论的得出方式来帮助分析师。

**决策支持。**将策略、法规或业务规则编码为逻辑陈述。推理引擎一致地评估它们并为决策提供审计追踪。

<a id="when-to-use-when-not-to-use"></a>
## 适用与不适用场景

**使用推理的场景：**
- 具有明确逻辑关系的复杂领域
- 策略执行和合规检查
- 结论依赖于事实链的多步推断
- 需要可解释决策和审计追踪的场景
- 识别未明确陈述的隐含关系

**检索可能足够的场景：**
- 查找直接回答问题的文档或信息
- 不确定要寻找什么模式的探索性研究
- 简单的关键词或语义搜索

**图遍历可能足够的场景：**
- 遵循实体间的显式关系
- 围绕已知起点的邻域分析
- 在直接连接的实体间寻找路径

**推理提供额外价值的场景：**
- 你的领域有可以推断新事实的逻辑规则
- 你需要检测需要多个条件的模式
- 决策必须可解释和可审计
- 隐含关系与显式关系同等重要

<a id="key-reasoning-concepts"></a>
## 关键推理概念

**Datalog** 是一种基于规则和事实的逻辑编程语言。事实是简单陈述，如"parent(tom, bob)"。规则推导新事实："grandparent(X, Z) :- parent(X, Y), parent(Y, Z)"。Datalog 擅长递归查询和传递关系。

**SPARQL** 是一种用于 RDF 数据的查询语言，可以纳入推断规则。它使用三元组模式匹配图谱数据，并可通过推理扩展以在查询前推导隐含三元组。

**RETE** 是一种高效地针对事实工作内存评估大量规则的算法。它构建一个避免重复评估未改变条件的网络，使其适用于具有数百条规则或流数据的系统。

**基于规则的推断**应用"如果-那么"规则来推导新结论。前向链接应用所有适用规则以推导尽可能多的事实。反向链接从目标倒推以找到最小证明。

<a id="how-graph-data-becomes-reasoning-facts"></a>
## 图谱数据如何成为推理事实

使用 `DatalogReasoner.load_from_graph()` 时，你的知识图谱的节点和边被转换为 Datalog 事实。所有谓词和参数都被转为小写：

- 类型为 "ThreatActor"、ID 为 "APT29" 的节点变为 `threatactor(apt29)`
- 从 APT29 到 SUNBURST、类型为 "uses" 的边变为 `uses(apt29, sunburst)`

推理引擎将这些事实视为推断的起点，应用规则推导新结论并添加到工作内存中。

<Info>
推理模块在你直接提供或从 `ContextGraph` 加载的事实上运行。推导出的事实被添加到工作内存中，并立即可用于同一会话中的进一步推断。要将推导的事实持久化回图谱，请将它们传递给 `AgentContext.store()`。
</Info>

<a id="choosing-a-reasoning-mode"></a>
## 选择推理模式

| 模式 | 最佳用途 | 类 |
|---|---|---|
| 前向链接 | 从基础事实物化所有隐含事实 | `Reasoner.forward_chain()` |
| 反向链接 | 证明特定目标；获取最小证据链 | `Reasoner.backward_chain()` |
| Datalog | 任意深度的递归遍历（供应链、组织图谱） | `DatalogReasoner` |
| SPARQL | 对富化后工作内存的模式匹配查询 | `SPARQLReasoner` |
| RETE | 100+ 规则集——通过 alpha/beta 网络增量传播事实 | `ReteEngine` |
| 时态 | 时间窗口之间的 Allen 区间关系 | `TemporalReasoningEngine` |
| 自然语言 | 基于图谱上下文的 LLM 自由格式查询 | `GraphReasoner` |
| 解释 | 将推断结果转化为人类可读的理由 | `ExplanationGenerator` |

<a id="step-1-ground-facts-and-working-memory"></a>
## 步骤 1 —— 基础事实与工作内存

`Reasoner` 类维护一组基础事实和一系列规则。事实可以以谓词字符串形式添加，也可以从 `ContextGraph` 加载。从抽取流水线产生的显式知识开始：

```python
from semantica.reasoning import Reasoner, Rule, RuleType

reasoner = Reasoner()

# 基础事实：Predicate(arg) 或 Predicate(arg1, arg2)
reasoner.add_fact("ThreatActor(APT29)")
reasoner.add_fact("ThreatActor(GAMMA-7)")
reasoner.add_fact("ThreatActor(DELTA-3)")
reasoner.add_fact("Exploits(APT29, CVE-2025-3400)")
reasoner.add_fact("Exploits(GAMMA-7, CVE-2025-1234)")
reasoner.add_fact("Exploits(GAMMA-7, CVE-2025-5678)")
reasoner.add_fact("CriticalVuln(CVE-2025-3400)")
reasoner.add_fact("CriticalVuln(CVE-2025-1234)")
reasoner.add_fact("CriticalVuln(CVE-2025-5678)")
reasoner.add_fact("Targets(APT29, NATOLogistics)")
reasoner.add_fact("Targets(GAMMA-7, NATOLogistics)")
reasoner.add_fact("SuppliedExploits(DELTA-3, GAMMA-7)")
reasoner.add_fact("SectorOverlap(NATOLogistics, CriticalInfrastructure)")
```

这些基础事实代表文档明确陈述的内容。你接下来添加的规则告诉系统这些事实*意味着什么*。

<a id="step-2-forward-chaining-materialising-derived-facts"></a>
## 步骤 2 —— 前向链接：物化推导事实

前向链接从基础事实出发，应用每条匹配规则直到无法得出新结论——达到不动点：

```python
# 字符串格式的规则会自动解析
# 变量是单个大写字母或多字符大写单词
reasoner.add_rule(
    "IF ThreatActor(X) AND Exploits(X, Y) AND CriticalVuln(Y) THEN HighRiskActor(X)"
)
reasoner.add_rule(
    "IF HighRiskActor(X) AND Targets(X, Z) THEN CriticalTarget(Z)"
)
reasoner.add_rule(
    "IF SuppliedExploits(A, B) AND HighRiskActor(B) THEN HighRiskSupplier(A)"
)

# forward_chain() 应用所有规则直到不动点
derived = reasoner.forward_chain()

for result in derived:
    print("{:<40s}  conf={:.0%}  rule={}".format(
        result.conclusion,
        result.confidence,
        result.rule_used.name if result.rule_used else "n/a",
    ))
```

```text
HighRiskActor(APT29)                      conf=100%  rule=Rule 1
HighRiskActor(GAMMA-7)                    conf=100%  rule=Rule 1
CriticalTarget(NATOLogistics)             conf=100%  rule=Rule 2
HighRiskSupplier(DELTA-3)                 conf=100%  rule=Rule 3
```

DELTA-3 被标记，尽管没有文档以这种方式描述它——系统追踪到：DELTA-3 向 GAMMA-7 提供了利用工具，而 GAMMA-7 利用关键 CVE。对于需要优先级排序或分级置信度的规则，使用 `Rule` 数据类：

```python
# 优先级更高的规则先触发；置信度传播到 InferenceResult.confidence
reasoner.add_rule(Rule(
    rule_id="attr-1",
    name="ttp_match_attribution",
    conditions=["ThreatActor(X)", "Exploits(X, CVE)", "CriticalVuln(CVE)"],
    conclusion="HighRiskActor(X)",
    rule_type=RuleType.IMPLICATION,
    confidence=0.92,
    priority=10,
))

reasoner.add_rule(Rule(
    rule_id="attr-2",
    name="supplier_elevation",
    conditions=["SuppliedExploits(A, B)", "HighRiskActor(B)"],
    conclusion="HighRiskSupplier(A)",
    rule_type=RuleType.IMPLICATION,
    confidence=0.85,
    priority=5,
))
```

<a id="step-3-backward-chaining-proving-a-specific-goal"></a>
## 步骤 3 —— 反向链接：证明特定目标

反向链接通过沿规则倒推来测试单个假设——当你需要是/否答案和最小证据链，而无需先推导所有其他可能事实时的正确工具：

```python
# backward_chain() 在目标可证明时返回 InferenceResult，否则返回 None
result = reasoner.backward_chain("HighRiskSupplier(DELTA-3)", max_depth=5)

if result:
    print("Proved: {}".format(result.conclusion))
    print("Via premises:")
    for p in result.premises:
        print("  - {}".format(p))
    print("Confidence: {:.0%}".format(result.confidence))
else:
    print("Goal not provable — DELTA-3 is not classified as a high-risk supplier "
          "given current facts and rules")
```

```text
Proved: HighRiskSupplier(DELTA-3)
Via premises:
  - SuppliedExploits(DELTA-3, GAMMA-7)
  - HighRiskActor(GAMMA-7)
Confidence: 85%
```

前提列表就是解释链——每一项都是达成结论所必需的事实。当分析师问"为什么 DELTA-3 被归类为高风险？"时，展示这个列表。

<a id="step-4-recursive-inference-with-datalog"></a>
## 步骤 4 —— 使用 Datalog 进行递归推断

`DatalogReasoner` 处理需要任意深度遍历的问题——"哪些行为体可以传递性地触及关键基础设施？"——使用带有半朴素自底向上不动点求值的递归 Horn 子句规则：

```python
from semantica.reasoning import DatalogReasoner

dl = DatalogReasoner()

# EDB（外延数据库）——基础事实；参数是常量（小写）
dl.add_fact("supplied(delta3, gamma7)")
dl.add_fact("supplied(gamma7, apt29_affiliate)")
dl.add_fact("supplied(apt29_affiliate, apt29)")
dl.add_fact("targets(apt29, nato_logistics)")
dl.add_fact("targets(gamma7, nato_logistics)")
dl.add_fact("sector(nato_logistics, critical_infrastructure)")

# IDB（内涵数据库）——递归规则；大写 = 变量
# 基本情形：直接供应链接
dl.add_rule("reaches(X, Y) :- supplied(X, Y).")
# 递归情形：如果 X 供应 Z 且 Z 可达 Y，则 X 可达 Y
dl.add_rule("reaches(X, Y) :- supplied(X, Z), reaches(Z, Y).")

# 结合可达性和行业归属的派生谓词
dl.add_rule("sector_exposure(Actor, Sector) :- reaches(Actor, Target), sector(Target, Sector).")
dl.add_rule("sector_exposure(Actor, Sector) :- targets(Actor, Target), sector(Target, Sector).")

# derive_all() 运行半朴素不动点直到没有新事实出现
dl.derive_all()

# 使用自由变量查询——返回绑定字典列表
exposures = dl.query("sector_exposure(?actor, critical_infrastructure)")
for row in exposures:
    print("CI-exposed actor: {}".format(row["actor"]))
```

```text
CI-exposed actor: delta3
CI-exposed actor: gamma7
CI-exposed actor: apt29_affiliate
CI-exposed actor: apt29
```

DELTA-3 出现，尽管没有文档将其直接连接到关键基础设施。Datalog 追踪了完整链路：delta3 → gamma7 → apt29_affiliate → apt29 → nato_logistics → critical_infrastructure。

绑定变量以提出有向问题：

```python
# delta3 对哪些行业有暴露？
rows = dl.query("sector_exposure(delta3, ?sector)")
for row in rows:
    print("delta3 → sector: {}".format(row["sector"]))
```

通过直接加载 `ContextGraph` 来跳过手动 `add_fact()` 调用：

```python
from semantica.context import ContextGraph

graph = ContextGraph()
# ... 图谱由 AgentContext.store() 或抽取流水线填充 ...
count = dl.load_from_graph(graph)
print("Loaded {} facts from graph".format(count))
```

<a id="step-5-sparql-queries-over-enriched-working-memory"></a>
## 步骤 5 —— 对富化后工作内存的 SPARQL 查询

在前向链接推导出新事实之后，`SPARQLReasoner` 让你使用 SPARQL 三元组模式匹配和可选的推断扩展来查询富化后的工作内存：

```python
from semantica.reasoning import SPARQLReasoner

sparql = SPARQLReasoner()

# 推断规则在执行前扩展 SPARQL 查询
sparql.add_inference_rule(
    "IF ThreatActor(X) AND Exploits(X, Y) AND CriticalVuln(Y) THEN HighRiskActor(X)"
)

query = """
    SELECT ?actor ?cve WHERE {
        ?actor <Exploits> ?cve .
        ?cve   <CriticalVuln> true .
    }
"""

# execute_query() 运行：扩展 → 推断 → 去重
result = sparql.execute_query(query)

for binding in result.bindings:
    print("Actor: {:15s}  CVE: {}".format(
        binding.get("actor", "?"),
        binding.get("cve", "?"),
    ))

# 元数据显示多少结果来自推断 vs 基础事实
print("Original: {}  Inferred: {}".format(
    result.metadata.get("original_count", 0),
    result.metadata.get("inferred_count", 0),
))
```

在运行之前检查扩展后的查询：

```python
# 查看应用推断规则后的查询
expanded = sparql.expand_query(query)
print(expanded)
```

<a id="step-6-rete-engine-for-large-rule-sets"></a>
## 步骤 6 —— 用于大型规则集的 RETE 引擎

`ReteEngine` 实现了 RETE 算法——由 alpha 节点（单条件匹配）和 beta 节点（连接操作）组成的网络，在每条新事实上避免重复评估未改变的条件。当你有 100+ 规则或需要在流式或事件驱动设置中进行增量事实传播时使用它：

```python
from semantica.reasoning import ReteEngine, Rule, RuleType, Fact

# 将规则定义为 Rule 对象——与 Reasoner 使用的 Rule 类相同
rules = [
    Rule(
        rule_id="r1", name="port_scan_detected",
        conditions=["PortScan(Source)", "HighFrequency(Source)"],
        conclusion="Scanning(Source)",
        confidence=0.90, priority=10,
    ),
    Rule(
        rule_id="r2", name="c2_beacon_identified",
        conditions=["Scanning(Source)", "BeaconPattern(Source, Dest)"],
        conclusion="C2Channel(Source, Dest)",
        confidence=0.85, priority=8,
    ),
    Rule(
        rule_id="r3", name="lateral_movement_detected",
        conditions=["C2Channel(Source, Dest)", "InternalHost(Dest)"],
        conclusion="LateralMovement(Source, Dest)",
        confidence=0.80, priority=5,
    ),
]

engine = ReteEngine()
engine.build_network(rules)   # 将规则编译为 alpha/beta/终端节点网络

# 事实是结构化的：Fact(fact_id, predicate, [arguments])
engine.add_fact(Fact("f1", "PortScan",      ["192.168.1.50"]))
engine.add_fact(Fact("f2", "HighFrequency", ["192.168.1.50"]))
engine.add_fact(Fact("f3", "BeaconPattern", ["192.168.1.50", "10.0.0.5"]))
engine.add_fact(Fact("f4", "InternalHost",  ["10.0.0.5"]))

# match_patterns() 返回 Match 对象：规则 + 匹配事实 + 置信度
matches = engine.match_patterns()
print("Rule activations: {}".format(len(matches)))

# execute_matches() 触发每条匹配规则并返回推导结论
conclusions = engine.execute_matches(matches)
for c in conclusions:
    print("Derived:", c)

# 网络诊断——用于验证规则编译
stats = engine.get_network_stats()
print("Nodes: {total_nodes}  Alpha: {alpha_nodes}  Beta: {beta_nodes}  Facts: {facts}".format(**stats))

# 清除工作内存而不重新编译规则网络
engine.reset()
```

规则网络由 `build_network()` 编译一次。后续每次 `add_fact()` 调用仅增量传播到其条件满足的节点——而非整个规则集——这使得评估成本与新激活的数量成正比，而非总规则数。

<a id="step-7-temporal-interval-reasoning"></a>
## 步骤 7 —— 时态区间推理

`TemporalReasoningEngine` 计算时间窗口之间的 Allen 区间关系，让你识别两个事件是否重叠、一个包含另一个、它们在边界处相接等，覆盖你的整个图谱：

```python
from datetime import datetime, timezone
from semantica.reasoning import TemporalReasoningEngine, TemporalInterval, IntervalRelation

engine = TemporalReasoningEngine()

def dt(value: str) -> datetime:
    return datetime.fromisoformat(value.replace("Z", "+00:00")).astimezone(timezone.utc)

# 将时间窗口编码为 datetime
nightfall  = TemporalInterval(start=dt("2025-01-01T00:00:00Z"), end=dt("2025-03-31T23:59:59Z"))
sandstorm  = TemporalInterval(start=dt("2025-03-15T00:00:00Z"), end=dt("2025-06-30T23:59:59Z"))
frostbite  = TemporalInterval(start=dt("2025-07-01T00:00:00Z"), end=dt("2025-09-30T23:59:59Z"))

relation_ns = engine.relation(nightfall, sandstorm)
relation_nf = engine.relation(nightfall, frostbite)

print("NIGHTFALL vs SANDSTORM:", relation_ns)
# IntervalRelation.OVERLAPS——两者在 2025 年 3 月中旬同时活跃
# 值得调查是否存在共享 C2 基础设施或协调行动

print("NIGHTFALL vs FROSTBITE:", relation_nf)
# IntervalRelation.BEFORE——无时间重叠；可能是独立行动
```

13 种 Allen 关系涵盖所有可能的时态关系：

| 关系 | 含义 |
|---|---|
| `BEFORE` | A 在 B 开始之前结束 |
| `MEETS` | A 恰好在 B 开始处结束（无间隙，无重叠） |
| `OVERLAPS` | A 在 B 之前开始并在 B 期间结束 |
| `STARTS` | A 和 B 同时开始；A 先结束 |
| `DURING` | A 完全包含在 B 中 |
| `FINISHES` | A 和 B 同时结束；A 较晚开始 |
| `EQUALS` | A 和 B 是相同的区间 |
| `AFTER`, `MET_BY`, `OVERLAPPED_BY`, `STARTED_BY`, `CONTAINS`, `FINISHED_BY` | 上述关系的逆关系 |

归因于不同行为体的两个行动之间的 `OVERLAPS` 或 `EQUALS` 结果是一个值得标记给分析师审查的信号——时间上的巧合是一种假设，而非结论。

<a id="step-8-llm-based-graph-reasoning"></a>
## 步骤 8 —— 基于 LLM 的图谱推理

`GraphReasoner` 将自由格式的自然语言查询通过 LLM 提供者路由，以图谱作为接地上下文。用于不能清晰映射到预定义规则集的探索性问题：

```python
from semantica.reasoning import GraphReasoner
from semantica.context import ContextGraph

# 使用任何受支持的 LLM 提供者初始化
gr = GraphReasoner(provider="openai", model="gpt-4o-mini")

# 构建知识图谱
graph = ContextGraph()
graph.add_node("apt29",          "ThreatActor",   "APT29 / NOBELIUM", country="Russia")
graph.add_node("cve-2025-3400",  "Vulnerability", "PAN-OS RCE",       cvss=9.8)
graph.add_node("nato_logistics", "Target",        "NATO Logistics Network")
graph.add_edge("apt29", "cve-2025-3400",  "exploits", weight=0.97)
graph.add_edge("apt29", "nato_logistics", "targets",  weight=0.88)

# GraphReasoner 期望 {"entities": [...], "relationships": [...]}
raw = graph.to_dict()
graph_data = {
    "entities":      raw.get("nodes", []),
    "relationships": raw.get("edges", []),
}

answer = gr.reason(
    graph=graph_data,
    query="Which threat actors pose the highest risk to NATO infrastructure, "
          "and what evidence in the graph supports that assessment?"
)
print(answer)
```

`GraphReasoner` 适合早期阶段的调查——当问题是探索性的且你尚未形式化推断规则时。对于可重复的、可审计的决策，请改用 `Reasoner` 或 `DatalogReasoner`。

<a id="step-9-explaining-inferences-in-natural-language"></a>
## 步骤 9 —— 用自然语言解释推断

`ExplanationGenerator` 将任何 `InferenceResult`（来自前向或反向链接）转化为人类可读的解释、逐步的 `ReasoningPath` 以及带有支持证据的 `Justification`：

```python
from semantica.reasoning import Reasoner, Rule, RuleType, ExplanationGenerator

# 先运行推断
reasoner = Reasoner()
reasoner.add_fact("ThreatActor(APT29)")
reasoner.add_fact("Exploits(APT29, CVE-2025-3400)")
reasoner.add_fact("CriticalVuln(CVE-2025-3400)")

reasoner.add_rule(Rule(
    rule_id="r1", name="high_risk_actor",
    conditions=["ThreatActor(X)", "Exploits(X, Y)", "CriticalVuln(Y)"],
    conclusion="HighRiskActor(X)",
    confidence=0.92,
))

derived = reasoner.forward_chain()

# detail_level 选项："simple"、"detailed"、"verbose"
gen = ExplanationGenerator(generate_nl=True, detail_level="detailed")

for result in derived:
    exp = gen.generate_explanation(result)
    print("Conclusion:  {}".format(exp.conclusion))
    print("Explanation: {}".format(exp.natural_language))
    print()

    # 逐步推理路径
    path = gen.show_reasoning_path(result)
    print("Reasoning path ({} steps, confidence {:.0%}):".format(
        len(path.steps), path.total_confidence
    ))
    for step in path.steps:
        print("  [{}] {}".format(step.step_id, step.description))

    # 带有完整证据列表的理由
    just = gen.justify_conclusion(result.conclusion, path)
    print("Justification: {}".format(just.explanation_text))
    print("Supporting evidence: {}".format(just.supporting_evidence))
```

```text
Conclusion:  HighRiskActor(APT29)
Explanation: Given the premises: ThreatActor(APT29), Exploits(APT29, CVE-2025-3400),
             CriticalVuln(CVE-2025-3400), we conclude: HighRiskActor(APT29)
             using rule 'high_risk_actor'.
```

三个详细级别控制解释的详尽程度：`"simple"` 给出单行摘要，`"detailed"` 列出前提和规则名称，`"verbose"` 产生完整的置信度注释叙述。

<a id="putting-it-together-a-complete-reasoning-pipeline"></a>
## 综合应用：完整的推理流水线

结合前向链接、Datalog 可达性和自然语言解释的威胁情报图谱流水线：

```python
from semantica.reasoning import (
    Reasoner, Rule, RuleType,
    DatalogReasoner, ExplanationGenerator,
)
from semantica.context import AgentContext, ContextGraph
from semantica.vector_store import VectorStore


def run_threat_reasoning(graph: ContextGraph) -> dict:
    """Apply inference rules to a threat intelligence graph."""

    # --- 前向链接：推导行为体分类 ---
    reasoner = Reasoner()

    for edge in graph.find_edges():
        src = edge.get("source", "")
        dst = edge.get("target", "")
        rel = edge.get("type", "related_to")
        if src and dst:
            reasoner.add_fact("{}({}, {})".format(rel.replace(" ", "_"), src, dst))

    for node in graph.find_nodes():
        name  = node.get("name", node.get("id", ""))
        ntype = node.get("type", "Entity")
        if name:
            reasoner.add_fact("{}({})".format(ntype.replace(" ", "_"), name))

    reasoner.add_rule(Rule(
        rule_id="r1", name="high_risk_actor",
        conditions=["ThreatActor(X)", "Exploits(X, CVE)", "CriticalVuln(CVE)"],
        conclusion="HighRiskActor(X)", confidence=0.92, priority=10,
    ))
    reasoner.add_rule(Rule(
        rule_id="r2", name="critical_target",
        conditions=["HighRiskActor(X)", "Targets(X, Z)"],
        conclusion="CriticalTarget(Z)", confidence=0.88, priority=8,
    ))
    reasoner.add_rule(Rule(
        rule_id="r3", name="supplier_elevation",
        conditions=["SuppliedExploits(A, B)", "HighRiskActor(B)"],
        conclusion="HighRiskSupplier(A)", confidence=0.85, priority=5,
    ))

    derived = reasoner.forward_chain()

    high_risk_actors    = [r for r in derived if "HighRiskActor"    in r.conclusion]
    critical_targets    = [r for r in derived if "CriticalTarget"   in r.conclusion]
    high_risk_suppliers = [r for r in derived if "HighRiskSupplier" in r.conclusion]

    # --- 反向链接：验证特定供应商假设 ---
    supplier_result = reasoner.backward_chain("HighRiskSupplier(DELTA-3)", max_depth=5)

    # --- Datalog：传递性供应链可达性 ---
    dl = DatalogReasoner()
    dl.load_from_graph(graph)
    dl.add_rule("reaches(X, Y) :- supplied(X, Y).")
    dl.add_rule("reaches(X, Y) :- supplied(X, Z), reaches(Z, Y).")
    dl.add_rule("sector_exposure(Actor, Sector) :- reaches(Actor, T), sector(T, Sector).")
    dl.add_rule("sector_exposure(Actor, Sector) :- targets(Actor, T), sector(T, Sector).")
    dl.derive_all()
    ci_exposures = dl.query("sector_exposure(?actor, critical_infrastructure)")

    # --- ExplanationGenerator：分析师可读的理由 ---
    gen = ExplanationGenerator(generate_nl=True, detail_level="detailed")
    explanations = {}
    for result in high_risk_actors:
        exp = gen.generate_explanation(result)
        explanations[result.conclusion] = exp.natural_language

    return {
        "high_risk_actors":    [r.conclusion for r in high_risk_actors],
        "critical_targets":    [r.conclusion for r in critical_targets],
        "high_risk_suppliers": [r.conclusion for r in high_risk_suppliers],
        "delta3_flagged":      supplier_result is not None,
        "ci_exposed_actors":   [row["actor"] for row in ci_exposures],
        "total_derived":       len(derived),
        "explanations":        explanations,
    }


intel_graph = ContextGraph(advanced_analytics=True)
agent = AgentContext(
    vector_store=VectorStore(backend="faiss", dimension=768),
    knowledge_graph=intel_graph,
)
# ... 通过 agent.store() 或抽取流水线填充 ...

report = run_threat_reasoning(intel_graph)
print("Derived {} new facts".format(report["total_derived"]))
print("High-risk actors:  {}".format(report["high_risk_actors"]))
print("CI-exposed actors: {}".format(report["ci_exposed_actors"]))
print("DELTA-3 flagged:   {}".format(report["delta3_flagged"]))

for conclusion, text in report["explanations"].items():
    print("\n[{}]\n  {}".format(conclusion, text))
```

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防 — CTI/威胁">

威胁情报中的归因链需要多跳置信度传播：TTP 匹配提高行为体归因的概率，ASN 地理位置佐证进一步提高它，而已归因行业的已知攻击模式将其提升到可操作的置信度。每一跳都是带有自身置信权重的独立规则，`InferenceResult` 通过链传递传播值。

```python
from semantica.reasoning import Reasoner, Rule, RuleType

reasoner = Reasoner()

# SIGINT 派生的事实
reasoner.add_fact("C2Beacon(10.0.0.5, AS59796)")
reasoner.add_fact("ASN_Country(AS59796, Russia)")
reasoner.add_fact("TTP(T1566.001, APT29)")           # MITRE ATT&CK 映射
reasoner.add_fact("ObservedTTP(10.0.0.5, T1566.001)")
reasoner.add_fact("TargetSector(10.0.0.5, Aerospace)")

# 带有置信度阶梯的三阶段归因规则集
reasoner.add_rule(Rule(
    rule_id="attr-1", name="ttp_match",
    conditions=["ObservedTTP(IP, TTP)", "TTP(TTP, Actor)"],
    conclusion="SuspectedActor(IP, Actor)",
    confidence=0.75, priority=10,
))
reasoner.add_rule(Rule(
    rule_id="attr-2", name="asn_corroboration",
    conditions=["SuspectedActor(IP, Actor)", "C2Beacon(IP, ASN)", "ASN_Country(ASN, Country)"],
    conclusion="CorroboratedActor(IP, Actor, Country)",
    confidence=0.90, priority=5,
))
reasoner.add_rule(Rule(
    rule_id="attr-3", name="sector_confirmation",
    conditions=["CorroboratedActor(IP, Actor, Russia)", "TargetSector(IP, Aerospace)"],
    conclusion="HighConfidenceAttribution(IP, Actor)",
    confidence=0.95, priority=1,
))

derived = reasoner.forward_chain()
attributions = [r for r in derived if "HighConfidenceAttribution" in r.conclusion]

for a in attributions:
    print("{:50s}  conf={:.0%}".format(a.conclusion, a.confidence))
    print("  Premises: {}".format(a.premises))

# HighConfidenceAttribution(10.0.0.5, APT29)    conf=95%
#   Premises: ['SuspectedActor(10.0.0.5, APT29)', 'CorroboratedActor(10.0.0.5, APT29, Russia)', ...]
```

</Tab>

<Tab title="安全 — SOC/事件">

零信任访问控制决策可以通过将策略编码为规则并针对特定访问请求进行反向链接在查询时评估。证明——或其缺失——就是可解释的决策记录，随时可供审计。

如果反向链接失败，部分结果中的前提列表确切告诉分析师哪个策略条件未被满足——这比通用的"访问被拒绝"消息有用得多。

```python
from semantica.reasoning import Reasoner, Rule, RuleType

reasoner = Reasoner()

# 当前会话的身份和资源事实
reasoner.add_fact("User(alice)")
reasoner.add_fact("HasMFA(alice)")
reasoner.add_fact("Role(alice, analyst)")
reasoner.add_fact("Clearance(alice, SECRET)")
reasoner.add_fact("Resource(kube-api, tier1)")
reasoner.add_fact("RequiresClearance(kube-api, SECRET)")
reasoner.add_fact("RequiresMFA(tier1)")

# 零信任访问策略作为推断规则
reasoner.add_rule(
    "IF User(U) AND HasMFA(U) AND RequiresMFA(T) AND Resource(R, T) THEN MFASatisfied(U, R)"
)
reasoner.add_rule(
    "IF User(U) AND Clearance(U, C) AND RequiresClearance(R, C) THEN ClearanceSatisfied(U, R)"
)
reasoner.add_rule(
    "IF MFASatisfied(U, R) AND ClearanceSatisfied(U, R) THEN AccessGranted(U, R)"
)

# 反向链接访问决策
result = reasoner.backward_chain("AccessGranted(alice, kube-api)", max_depth=5)

if result:
    print("ACCESS GRANTED: {}".format(result.conclusion))
    print("Justified by:")
    for p in result.premises:
        print("  - {}".format(p))
else:
    print("ACCESS DENIED — one or more policy conditions not satisfied")
    partial = reasoner.forward_chain()
    for r in partial:
        print("  Partial: {}".format(r.conclusion))
```

</Tab>

<Tab title="生命科学 — 临床/制药">

药物-药物相互作用检测是前向链接的天然契合：关于酶抑制和代谢途径的事实是静态的，关于相互作用风险的规则是标准化药理学，而输出——一条 `ClinicallySignificantInteraction` 推导事实——需要在处方确认之前可靠地触发。

Datalog 处理传递性情情况：如果药物 A 诱导酶 E1，而 E1 代谢药物 B，则可以在不预先知道深度的情况下递归追踪完整的暴露链。

```python
from semantica.reasoning import Reasoner, DatalogReasoner

# 前向链接：检测临床显著相互作用
reasoner = Reasoner()

reasoner.add_fact("Metabolizes(CYP2C9, Warfarin)")
reasoner.add_fact("Inhibits(Amiodarone, CYP2C9)")
reasoner.add_fact("DrugInPatient(Warfarin)")
reasoner.add_fact("DrugInPatient(Amiodarone)")
reasoner.add_fact("TherapeuticWindow(Warfarin, narrow)")

reasoner.add_rule(
    "IF Inhibits(DrugA, Enzyme) AND Metabolizes(Enzyme, DrugB) "
    "AND DrugInPatient(DrugA) AND DrugInPatient(DrugB) "
    "THEN PotentialInteraction(DrugA, DrugB)"
)
reasoner.add_rule(
    "IF PotentialInteraction(DrugA, DrugB) AND TherapeuticWindow(DrugB, narrow) "
    "THEN ClinicallySignificantInteraction(DrugA, DrugB)"
)

derived = reasoner.forward_chain()
for r in derived:
    if "ClinicallySignificant" in r.conclusion:
        print("ALERT: {}  (conf={:.0%})".format(r.conclusion, r.confidence))
        print("  Rule: {}".format(r.rule_used.name if r.rule_used else "n/a"))

# ALERT: ClinicallySignificantInteraction(Amiodarone, Warfarin)  (conf=100%)

# Datalog：传递性酶诱导链
dl = DatalogReasoner()
dl.add_fact("metabolises(CYP3A4, midazolam)")
dl.add_fact("metabolises(CYP3A4, cyclosporin)")
dl.add_fact("induces(rifampicin, CYP3A4)")
dl.add_rule("reduces_exposure(X, Drug) :- induces(X, Enzyme), metabolises(Enzyme, Drug).")
dl.add_rule("reduces_exposure(X, Drug) :- induces(X, Z), reduces_exposure(Z, Drug).")

dl.derive_all()
reductions = dl.query("reduces_exposure(rifampicin, ?drug)")
for row in reductions:
    print("Rifampicin reduces exposure to: {}".format(row["drug"]))

# Rifampicin reduces exposure to: midazolam
# Rifampicin reduces exposure to: cyclosporin
```

</Tab>

<Tab title="银行 — 风险/合规">

Basel III 资本充足率规则可以自然地转化为前向链接推断规则：每个监管条件是一个事实，每条条款是一条规则，而资本决策是推导结论。规则链就是审计追踪——审查员可以检查确切的哪些条件以何种顺序触发。

反向链接适用于压力测试："在什么条件下这笔贷款会获得 `ConditionalApproval`？"通过证明目标并呈现最小必需事实集来回答。

```python
from semantica.reasoning import Reasoner, Rule, RuleType

reasoner = Reasoner()

# 贷款申请事实
reasoner.add_fact("Loan(LOAN-2025-88421)")
reasoner.add_fact("LTV(LOAN-2025-88421, 0.78)")
reasoner.add_fact("PD(LOAN-2025-88421, 0.023)")
reasoner.add_fact("LGD(LOAN-2025-88421, 0.45)")
reasoner.add_fact("AssetClass(LOAN-2025-88421, CRE)")
reasoner.add_fact("DSCR(LOAN-2025-88421, 1.12)")

# Basel III CRE20 资本规则
reasoner.add_rule(Rule(
    rule_id="cre20-1", name="ltv_rwa_bucket",
    conditions=["Loan(L)", "LTV(L, V)", "AssetClass(L, CRE)"],
    conclusion="RWABucket(L, high)",
    confidence=0.98, priority=10,
))
reasoner.add_rule(Rule(
    rule_id="cre20-2", name="dscr_adequate",
    conditions=["Loan(L)", "DSCR(L, D)"],
    conclusion="DSCRAdequate(L)",
    confidence=0.95, priority=8,
))
reasoner.add_rule(Rule(
    rule_id="cre20-3", name="conditional_approval",
    conditions=["Loan(L)", "RWABucket(L, high)", "DSCRAdequate(L)"],
    conclusion="ConditionalApproval(L)",
    confidence=0.87, priority=5,
))

derived = reasoner.forward_chain()
for r in derived:
    print("{:45s}  [{:.0%}]".format(r.conclusion, r.confidence))

# RWABucket(LOAN-2025-88421, high)                  [98%]
# DSCRAdequate(LOAN-2025-88421)                      [95%]
# ConditionalApproval(LOAN-2025-88421)               [87%]

# 审计：证明批准决策并展示其完整理由
proof = reasoner.backward_chain("ConditionalApproval(LOAN-2025-88421)", max_depth=5)
if proof:
    print("\nAudit trail for {}:".format(proof.conclusion))
    for p in proof.premises:
        print("  - {}".format(p))
```

</Tab>

</Tabs>

<a id="common-pitfalls"></a>
## 常见陷阱

**过度使用推理。**并非每个查询都需要推断。如果你可以通过简单的检索或图遍历回答问题，推理只会增加不必要的复杂性。在需要推导未明确陈述的事实时使用推理。

**图谱质量差。**推理会放大数据质量问题。如果你的图谱有不一致的实体名称、缺失的关系或不正确的事实，推理会传播这些错误。在应用推断规则之前清理图谱数据。

**将推导事实视为已验证事实。**推理结论的可靠性仅取决于其所基于的规则和事实。像"HighRiskActor(APT29)"这样的推导事实反映的是你的规则逻辑，而非基本事实。始终区分观察事实和推导结论。

**规则复杂度过高。**条件过多的复杂规则难以调试和维护。从简单规则开始，逐步增加复杂性。一条有 10 个条件的规则可能应该拆分为更小的、更聚焦的规则。

**大型数据集上的递归推理。**递归 Datalog 规则可以在大型图谱上生成指数级的推导事实。监控工作内存大小，并添加深度限制或终止条件以防止失控推断。

<a id="related-guides"></a>
## 相关指南

- [语义抽取](semantic-extraction.zh-CN.md) —— 抽取填充你所推理的图谱事实的实体和关系
- [GraphRAG](graphrag.zh-CN.md) —— 检索以图谱为基础的上下文用于 LLM 响应
- [本体管理](ontology.zh-CN.md) —— 生成 OWL 本体为你的规则赋予形式语义
- [决策智能](decision-intelligence.zh-CN.md) —— 通过完整因果链记录和追踪推导决策
- [上下文图谱](context-graphs.zh-CN.md) —— 推理所操作的底层知识图谱
- [MCP 服务端](mcp-server.zh-CN.md) —— 将 `run_reasoning` 作为工具暴露给 Claude 和其他智能体
