---
title: "推理模块"
description: "前向链接、Rete、演绎、归结、SPARQL、Datalog 和时态推理，提供可解释的推断路径。"
icon: "microchip"
---

**[English](reasoning.md)** · **简体中文（当前）**

`semantica.reasoning` 使用逻辑规则从已有事实中派生新知识：

- 六种推理引擎：前向链接、Rete、SPARQL、Datalog、时态推理以及基于 LLM 的 GraphReasoner
- 每个引擎都生成可解释的推断路径：可追溯的规则与事实链
- `DatalogReasoner` 通过半朴素不动点求值保证终止性
- `TemporalReasoningEngine` 实现全部 13 种 Allen 区间代数关系
- `ExplanationGenerator` 生成逐步的自然语言解释


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `Reasoner` | 前向链接推断：`add_fact`、`add_rule`、`forward_chain`、`backward_chain`、`infer_facts` |
| `GraphReasoner` | 基于 LLM 对知识图谱字典进行推理：通过 `reason(graph, query)` 回答自然语言查询 |
| `ReteEngine` | Rete 模式匹配：`build_network`、`add_fact`、`match_patterns`、`execute_matches` |
| `SPARQLReasoner` | 基于规则的 SPARQL 查询扩展：`execute_query`、`expand_query`、`infer_results` |
| `DatalogReasoner` | 递归 Horn 子句规则，采用半朴素不动点求值：`add_fact`、`add_rule`、`derive_all`、`query` |
| `TemporalReasoningEngine` | 全部 13 种 Allen 区间代数关系：`relation(a, b)`、`overlaps`、`contains`、`active_at` |
| `ExplanationGenerator` | 通过 `generate_explanation(inference_result)` 生成逐步解释 |
| `Rule` | IF/THEN 规则：`{rule_id, name, conditions, conclusion, rule_type, confidence, priority}` |
| `Fact` | 工作记忆事实：`{fact_id, predicate, arguments}` |
| `InferenceResult` | 单个派生的结论：`{conclusion, rule_used, premises, confidence}` |


<a id="which-engine-should-i-use"></a>
## 我应该使用哪个引擎？

- [Reasoner](#reasoner-forwardbackward-chaining) — IF/THEN 规则、前向链接和反向链接。**从这里开始**：覆盖 90% 的使用场景，无需查询语言。
- [GraphReasoner](#graphreasoner) — 通过 LLM 对知识图谱进行自然语言查询。无需 SPARQL 或规则：直接提问即可。
- [DatalogReasoner](#datalogreasoner) — 保证终止的递归 Horn 子句规则。适用于复杂的多跳传递规则。
- [ReteEngine](#reteengine) — 用于高频推断的 Rete 模式匹配。当需要同时将大量事实与大量规则匹配时使用。
- [SPARQLReasoner](#sparqlreasoner) — SPARQL 查询扩展和基于规则的推断。在处理 RDF/OWL 数据时使用。
- [TemporalReasoningEngine](#temporalreasoningengine) — 全部 13 种 Allen 区间代数关系。用于时态感知推理：重叠、之前/之后、期间、包含。


<a id="getting-started"></a>
## 快速上手

最常见的模式是用于 IF/THEN 前向链接的 `Reasoner`：

```python
from semantica.reasoning import Reasoner, Rule, RuleType

reasoner = Reasoner()

# 以 predicate(args) 形式的字符串添加事实
reasoner.add_fact("Manager(Alice)")
reasoner.add_fact("Employee(Alice)")

# 使用字符串形式添加 IF-THEN 规则
reasoner.add_rule("IF Manager(?x) THEN HasAuthority(?x)")

# 运行前向链接：返回 List[InferenceResult]
results = reasoner.forward_chain()
for r in results:
    print(r.conclusion)         # "HasAuthority(Alice)"
    print(r.confidence)         # 1.0
    if r.rule_used:
        print(r.rule_used.name) # 所应用的规则名称
```

或者使用 `Rule` 数据类以编程方式构建规则：

```python
from semantica.reasoning import Rule, RuleType

rule = Rule(
    rule_id="rule_001",
    name="manager_authority",
    conditions=["Manager(?x)"],
    conclusion="HasAuthority(?x)",
    rule_type=RuleType.IMPLICATION,
    confidence=0.9,
)
reasoner.add_rule(rule)
```


<a id="reasoner-forwardbackward-chaining"></a>
## Reasoner（前向/反向链接）

**`Reasoner`** 是基于规则的推断的统一入口：迭代事实和规则直至**不动点**，然后可选择通过反向链接证明特定目标：

```python
from semantica.reasoning import Reasoner, Rule, RuleType, InferenceResult

reasoner = Reasoner()

# 事实可以是字符串、KG 实体字典或 KG 关系字典
reasoner.add_fact("Manager(John)")
reasoner.add_fact("Employee(John)")

# IF-THEN 字符串形式
reasoner.add_rule("IF Manager(?x) AND Employee(?x) THEN SeniorStaff(?x)")

# 前向链接：迭代直至不动点
results = reasoner.forward_chain()
for r in results:
    print(r.conclusion)   # 例如 "SeniorStaff(John)"
    print(r.premises)     # 匹配到的前提字符串列表
    print(r.confidence)   # float

# 反向链接：证明特定目标
result = reasoner.backward_chain("SeniorStaff(John)", max_depth=10)
if result:
    print(f"Proven: {result.conclusion}")
    print(f"Premises: {result.premises}")

# infer_facts() 在一次调用中加载事实和规则，返回结论字符串
conclusions = reasoner.infer_facts(
    facts=["Manager(Alice)", "Employee(Alice)"],
    rules=["IF Manager(?x) THEN HasAuthority(?x)"],
)
# → ["HasAuthority(Alice)"]
```

<a id="reasoner-methods"></a>
### Reasoner 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `add_fact(fact)` | `None` | 向工作记忆添加字符串、实体字典或关系字典 |
| `add_rule(rule)` | `Rule` | 添加 `Rule` 对象或 IF-THEN 字符串；规则按 `priority` 降序排列 |
| `forward_chain()` | `List[InferenceResult]` | 迭代推导所有可能的结论直至不动点 |
| `backward_chain(goal, max_depth)` | `InferenceResult \| None` | 证明特定目标字符串，无法证明时返回 `None` |
| `infer_facts(facts, rules)` | `List[str]` | 加载事实和规则后运行 `forward_chain()`，返回结论字符串 |
| `clear()` | `None` | 清除所有事实和规则 |
| `reset()` | `None` | `clear()` 的别名 |

<a id="rule-and-fact-dataclass-fields"></a>
### Rule 和 Fact 数据类字段

```python
from semantica.reasoning import Rule, Fact, RuleType

# Rule：所有字段
rule = Rule(
    rule_id="rule_001",            # 必需：唯一标识符
    name="manager_authority",      # 必需：显示名称
    conditions=["Manager(?x)"],    # 条件字符串列表
    conclusion="HasAuthority(?x)", # 结论字符串
    rule_type=RuleType.IMPLICATION, # IMPLICATION | EQUIVALENCE | CONSTRAINT | TRANSFORMATION
    confidence=1.0,                 # 默认 1.0
    priority=0,                     # 优先级更高的规则先执行
)

# Fact：用于直接操作 Rete 引擎
from semantica.reasoning import Fact
fact = Fact(
    fact_id="f001",                # 必需：唯一标识符
    predicate="Manager",
    arguments=["John"],
    metadata={},
)
```


<a id="graphreasoner"></a>
## GraphReasoner

**`GraphReasoner`** 使用 LLM 回答对知识图谱字典的**自然语言查询**：无需编写 SPARQL 或规则：

```python
from semantica.reasoning import GraphReasoner

# 初始化：默认使用 openai；通过 kwargs 覆盖
reasoner = GraphReasoner(provider="openai", model="gpt-4o-mini")

kg = {
    "entities": [
        {"id": "alice",   "name": "Alice",   "type": "Person",       "properties": {"role": "CEO"}},
        {"id": "acme",    "name": "Acme Inc", "type": "Organization"},
    ],
    "relationships": [
        {"source": "alice", "target": "acme", "type": "leads"}
    ],
}

answer: str = reasoner.reason(
    graph=kg,
    query="Who leads Acme Inc. and what is their role?"
)
print(answer)
```

`reason()` 将图谱转换为文本上下文，并使用结构化提示调用 LLM。返回纯字符串答案。


<a id="reteengine"></a>
## ReteEngine

用于大型规则集的高性能 Rete 模式匹配：

```python
from semantica.reasoning import ReteEngine, Rule, Fact, RuleType

engine = ReteEngine()

# 从 Rule 对象列表构建 Rete 网络
rules = [
    Rule(
        rule_id="r1",
        name="manager_authority",
        conditions=["Manager(?x)"],
        conclusion="HasAuthority(?x)",
    )
]
engine.build_network(rules)

# 向工作记忆添加事实
engine.add_fact(Fact(fact_id="f1", predicate="Manager", arguments=["Alice"]))

# 匹配模式并执行
matches = engine.match_patterns()
results = engine.execute_matches(matches)
# results 是匹配规则产生的结论值列表

# 网络统计信息
stats = engine.get_network_stats()
# → {"total_nodes": N, "alpha_nodes": A, "beta_nodes": B, "terminal_nodes": T, "facts": F}

engine.reset()
```

<a id="reteengine-methods"></a>
### ReteEngine 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `build_network(rules)` | `None` | 从 `Rule` 对象列表构建 Rete 网络 |
| `add_fact(fact)` | `None` | 向工作记忆添加 `Fact` 并在网络中传播 |
| `match_patterns(facts)` | `List[Match]` | 匹配所有模式；可在匹配前可选地添加事实 |
| `execute_matches(matches)` | `List[Any]` | 执行匹配的规则并返回其结论值 |
| `reset()` | `None` | 清除事实和所有节点激活状态 |
| `get_network_stats()` | `dict` | 返回 alpha、beta、终端节点和事实的计数 |


<a id="sparqlreasoner"></a>
## SPARQLReasoner

**`SPARQLReasoner`** 通过**推断规则扩展**增强 SPARQL：添加 IF-THEN 规则后，它们会在执行前自动编织到查询中：

```python
from semantica.reasoning import SPARQLReasoner

reasoner = SPARQLReasoner()

# 添加推断规则（IF-THEN 字符串形式）
reasoner.add_inference_rule("IF is_a(?x, Manager) THEN has_authority(?x)")

# 执行查询：返回 SPARQLQueryResult
result = reasoner.execute_query("""
    PREFIX ex: <http://example.org/>
    SELECT ?person ?company WHERE {
        ?person ex:founded ?company .
        ?company ex:located_in ex:SiliconValley .
    }
""")

for row in result.bindings:
    print(row)   # 每行是一个 variable → {"value": ..., "type": ...} 的字典

# 用推断规则扩展查询（返回修改后的查询字符串）
expanded = reasoner.expand_query("SELECT ?x WHERE { ?x a :Manager }")

# 从现有结果推断额外的绑定
enriched = reasoner.infer_results(result)
```

<a id="sparqlreasoner-constructor"></a>
### SPARQLReasoner 构造函数

```python
SPARQLReasoner(
    config=None,         # 可选的配置字典
    triplet_store=None,  # 可选的 TripletStore 实例，用于实时查询执行
    enable_inference=True,
)
```

<Note>
  当未配置 `triplet_store` 时，`execute_query()` 返回空绑定。通过 `triplet_store=` kwarg 传入 `TripletStore` 实例以对实时后端执行查询。
</Note>


<a id="datalogreasoner"></a>
## DatalogReasoner

纯 Python 自底向上的半朴素不动点求值，用于递归 Horn 子句规则。终止性**有保证**：引擎检测到不动点收敛即停止：

```python
from semantica.reasoning import DatalogReasoner, DatalogFact

datalog = DatalogReasoner()

# 添加基础事实：字符串形式最简单
datalog.add_fact("parent(alice, bob)")
datalog.add_fact("parent(bob, charlie)")

# 或直接使用 DatalogFact（args 是字符串元组）
datalog.add_fact(DatalogFact(predicate="parent", args=("charlie", "dave")))

# 使用 Horn 子句语法添加递归规则
datalog.add_rule("ancestor(X, Y) :- parent(X, Y).")
datalog.add_rule("ancestor(X, Z) :- parent(X, Y), ancestor(Y, Z).")

# 求值至不动点：返回所有派生的事实字符串
all_facts = datalog.derive_all()
# 例如 ["parent(alice, bob)", "parent(bob, charlie)", ..., "ancestor(alice, bob)", ...]

# 使用变量模式查询：变量以大写字母或 ? 开头
results = datalog.query("ancestor(alice, ?Z)")
# → [{"Z": "bob"}, {"Z": "charlie"}, {"Z": "dave"}]

# 清除并重新开始
datalog.clear()
```

<a id="datalogfact-datalogrule-fields"></a>
### DatalogFact、DatalogRule 字段

```python
from semantica.reasoning import DatalogFact, DatalogRule

# DatalogFact：基础事实；args 必须全部为常量（小写字母开头）
fact = DatalogFact(predicate="parent", args=("alice", "bob"))

# DatalogRule：从字符串解析；head 和 body 由解析器设置
# 使用 add_rule("head(X, Y) :- body(X, Z), body2(Z, Y).")：不要直接构造
```

<a id="datalogreasoner-methods"></a>
### DatalogReasoner 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `add_fact(fact)` | `None` | 添加字符串、字典或 `DatalogFact`；常量必须小写字母开头 |
| `add_rule(rule_str)` | `None` | 解析并添加 Horn 子句字符串，如 `"ancestor(X,Y) :- parent(X,Y)."` |
| `derive_all()` | `List[str]` | 运行半朴素不动点求值；以字符串形式返回所有事实 |
| `query(pattern)` | `List[dict]` | 查询派生的事实：如有需要会自动运行 `derive_all()` |
| `load_from_graph(graph)` | `int` | 将 ContextGraph 的节点/边加载为 Datalog 事实；返回添加计数 |
| `clear()` | `None` | 清除所有事实和规则 |


<a id="temporalreasoningengine"></a>
## TemporalReasoningEngine

纯 Python 的 Allen 区间代数：全部 13 种关系，无 LLM 调用：

```python
from datetime import datetime
from semantica.reasoning import TemporalReasoningEngine, TemporalInterval, IntervalRelation

engine = TemporalReasoningEngine()

ceo_tenure   = TemporalInterval(start=datetime(1997, 9, 16), end=datetime(2011, 8, 24))
board_member = TemporalInterval(start=datetime(2000, 1,  1), end=datetime(2012, 6,  1))

# 计算 Allen 关系：方法名是 relation()，而不是 get_relation()
rel = engine.relation(ceo_tenure, board_member)
# → IntervalRelation.DURING  (ceo_tenure 完全在 board_member 之内)

# 其他辅助方法
engine.overlaps(ceo_tenure, board_member)   # bool
engine.contains(board_member, ceo_tenure)   # bool

# 给定时间点是否在某个区间内？
engine.active_at(ceo_tenure, datetime(2005, 6, 1))   # True
```

全部 13 种 Allen 区间代数关系：

| 关系 | 含义 |
| :-------- | :------- |
| `BEFORE` | A 在 B 开始之前结束 |
| `MEETS` | A 恰好在 B 开始时结束 |
| `OVERLAPS` | A 在 B 之前开始，在 B 内部结束 |
| `DURING` | A 完全在 B 内部 |
| `STARTS` | A 和 B 同时开始，A 先结束 |
| `FINISHES` | A 和 B 同时结束，A 较晚开始 |
| `EQUALS` | 相同的区间 |
| `AFTER` | BEFORE 的逆关系 |
| `MET_BY` | MEETS 的逆关系 |
| `OVERLAPPED_BY` | OVERLAPS 的逆关系 |
| `CONTAINS` | DURING 的逆关系 |
| `STARTED_BY` | STARTS 的逆关系 |
| `FINISHED_BY` | FINISHES 的逆关系 |

<Note>
  `TemporalInterval.start` 需要的是 `datetime` 对象，而不是字符串。从标准库导入 `datetime`，并使用 `datetime(year, month, day)` 构造区间。
</Note>


<a id="explanationgenerator"></a>
## ExplanationGenerator

为任何 `InferenceResult` 生成结构化解释：

```python
from semantica.reasoning import ExplanationGenerator, Reasoner, Rule

reasoner = Reasoner()
reasoner.add_fact("Manager(John)")
reasoner.add_rule("IF Manager(?x) THEN HasAuthority(?x)")
results = reasoner.forward_chain()

# ExplanationGenerator 不接受位置参数
generator = ExplanationGenerator()

# 传入 InferenceResult 对象：不是字典
explanation = generator.generate_explanation(results[0])

print(f"Type:       {explanation.explanation_type}")   # "inference"
print(f"Conclusion: {explanation.conclusion}")
print(f"NL:         {explanation.natural_language}")

if explanation.reasoning_path:
    for step in explanation.reasoning_path.steps:
        print(f"  Step {step.step_id}: {step.description}")
        if step.rule_applied:
            print(f"    Rule: {step.rule_applied.name}")

# 用推理路径论证结论
path = generator.show_reasoning_path(results[0])
justification = generator.justify_conclusion(results[0].conclusion, path)
print(justification.explanation_text)
```

<a id="explanationgenerator-methods"></a>
### ExplanationGenerator 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `generate_explanation(reasoning)` | `Explanation` | 为 `InferenceResult`、`Proof` 或归结结果生成结构化解释 |
| `show_reasoning_path(reasoning)` | `ReasoningPath` | 从任何结果中提取并返回推理路径 |
| `justify_conclusion(conclusion, path)` | `Justification` | 为结论构建包含证据和自然语言文本的 `Justification` |

<a id="key-dataclass-fields"></a>
### 关键数据类字段

```python
# Explanation
explanation.explanation_id    # str
explanation.explanation_type  # "inference" | "proof" | "abductive" | "generic"
explanation.conclusion        # 结论值
explanation.reasoning_path    # ReasoningPath | None
explanation.natural_language  # 自然语言字符串（当 generate_nl=True 时，默认值）

# ReasoningStep
step.step_id        # str
step.description    # str
step.rule_applied   # Rule | None  （不是 rule_name）
step.input_facts    # List[Any]
step.output_fact    # Any
step.confidence     # float
```


<a id="engine-selection-guide"></a>
## 引擎选择指南

| 引擎 | 最适合 | 终止性 | 关键方法 |
| :------ | :-------- | :----------- | :---------- |
| `Reasoner` | 简单的 IF/THEN 规则 | 总是终止（有 `max_iterations` 上限） | `forward_chain()` |
| `GraphReasoner` | 通过 LLM 对知识图谱进行自然语言查询 | 总是终止 | `reason(graph, query)` |
| `ReteEngine` | 包含大量事实的大型规则集 | 总是终止 | `match_patterns()` |
| `SPARQLReasoner` | 规则增强的 SPARQL 查询 | 总是终止 | `execute_query()` |
| `DatalogReasoner` | 递归规则（祖先关系、可达性） | 保证不动点终止 | `derive_all()` / `query()` |
| `TemporalReasoningEngine` | 时间区间关系 | 总是终止 | `relation(a, b)` |

<Tip>
  对于递归规则（例如祖先关系、可达性、传递性），请使用 `DatalogReasoner`：它通过半朴素自底向上不动点求值保证终止性。`Reasoner.forward_chain()` 有 `max_iterations` 上限（默认 50），在深度递归时会静默提前停止。
</Tip>

<Warning>
  `GraphReasoner` 需要已配置的 LLM 提供方。如果提供方初始化失败，`reason()` 会返回错误字符串而不是抛出异常。如果需要显式暴露失败，请在调用前检查 `reasoner.provider is not None`。
</Warning>

- [知识图谱](kg.zh-CN.md) — 被推理的知识图谱。
- [本体](ontology.zh-CN.md) — 用于逻辑推理的本体公理和 SHACL 约束。
- [三元组存储](triplet_store.zh-CN.md) — 用于基于 SPARQL 推理的 RDF 后端。
- [上下文](context.zh-CN.md) — 集成到智能体决策智能中的推理。
