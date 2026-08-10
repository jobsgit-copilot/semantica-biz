---
title: "策略引擎"
description: "定义、版本化并对知识图谱决策执行治理策略——包含合规检查、违规追踪、影响分析和多级审批工作流。"
icon: "scale-balanced"
---

**[English](policy-engine.md)** · **简体中文（当前）**

<a id="what-is-policy-engine"></a>
## 什么是策略引擎？

策略评估是对照预定义的治理规则和约束系统地检查决策的过程。与自动阻止不合规操作的应用层执行不同，策略评估提供合规状态，可触发不同的工作流——审批流程、异常处理或审计要求。

**核心策略概念：**

**策略评估**检查决策是否满足定义的标准，但不会自动阻止操作，从而实现灵活的治理工作流。

**治理与合规工作流**使用策略评估结果，将决策路由到适当的审批链、异常流程或审计追踪中。

**审批流程**可由策略违规触发，创建带有理由和审批人责任记录的文档化异常路径。

**与执行的区别：**策略评估返回合规状态（`True`/`False`），但不会自动阻止操作。由你的工作流决定下一步发生什么——立即批准、升级、异常处理或拒绝。

<a id="why-use-policy-engine"></a>
## 为何使用策略引擎？

**治理与可问责性。**创建可审计的决策工作流，其中每一次策略评估、异常和批准都永久记录在知识图谱中，并带有完整的溯源追踪。

**合规验证。**在决策最终确定或执行之前，系统地对照监管要求、内部策略和风险管理规则进行检查。

**审批工作流编排。**通过结构化的审批流程路由不合规决策，附带文档化的理由和多级签批。

**监管合规。**通过维护完整的策略版本历史、异常记录和合规检查轨迹来满足审计要求，供监管机构检查。

**风险管理。**将高风险决策标记为需要额外审查，同时允许常规合规决策以最小的摩擦继续进行。

**策略演进追踪。**维护策略变更的版本历史并进行影响分析，实现基于证据的策略完善和监管报告。

<a id="when-to-use-when-not-to-use"></a>
## 适用与不适用场景

**策略引擎适用于：**
- 需要结构化审批流程和审计追踪的治理工作流
- 策略遵循必须可文档化和可验证的监管合规场景
- 高风险决策的多级审批工作流（财务审批、安全例外、临床治疗）
- 策略违规会触发特定升级程序的受监管环境
- 不合规决策需要额外监督的风险管理工作流
- 要求完整策略应用和异常追踪的审计需求

**策略引擎不适用于：**
- 简单的表单校验或基本输入检查——请使用标准校验库
- 不需要审计追踪或治理工作流的基本业务规则
- 策略评估开销会影响性能的低风险、高吞吐检查
- 不需要版本追踪和审批流程的确定性规则检查
- 策略评估延迟不可接受的实时运营决策

**警告：**策略引擎增加了治理开销，需要仔细的工作流设计。仅在结构化策略管理的收益大于额外复杂性时使用。

`PolicyEngine` 对已记录的决策执行命名策略，如果决策满足所有策略规则则返回 `True`。使用它在运行时对 AI 决策进行把关——需要双重来源确认的归因、需要高级审批的升级，或任何在记录结果之前必须验证合规性的决策类别。策略是版本化的图谱节点，因此每次检查、异常和审批链都是永久审计追踪的一部分。

<Info>
策略引擎位于 `AgentContext` 和 `ContextGraph` 之上。策略以节点形式存储在与决策相同的图谱中，使其具有与任何其他知识图谱实体相同的因果追踪、溯源和时间有效性。

**关键对象：**`PolicyEngine` 和 `Policy` 从 `semantica.context` 导入。`Decision` 是一个数据类，包含 `decision_id`、`category`、`scenario`、`reasoning`、`outcome`、`confidence`、`timestamp`、`decision_maker` 和 `metadata` 等字段。`DecisionRecorder` 从 `semantica.context.decision_recorder` 导入，用于审批工作流追踪。
</Info>

<a id="supported-rule-types"></a>
## 支持的规则类型

PolicyEngine 实现支持特定的规则模式，用于评估决策属性和元数据：

**置信度规则：**
- `min_confidence: 0.85` —— `decision.confidence >= 0.85`

**结果校验：**
- `allowed_outcomes: ["approved", "approved_with_conditions"]` —— `decision.outcome` 必须在列表中

**类别校验：**
- `required_categories: ["credit_risk", "operational_risk"]` —— `decision.category` 必须在列表中

**元数据字段规则：**
- `min_*: value` —— 元数据字段必须 `>= value`（例如 `min_credit_score: 680`）
- `max_*: value` —— 元数据字段必须 `<= value`（例如 `max_ltv: 0.85`）
- `required_*: value` —— 元数据字段必须等于 `value`（字符串）或包含所有项（列表）

**字段查找行为：**对于规则 `min_credit_score`，引擎会依次检查 `metadata["credit_score"]`，然后是 `metadata["*_credit_score"]`（后缀匹配），然后是 `decision.credit_score` 属性。

**重要提示：**以下规则类型不受支持，会导致意外行为：
- `disallowed_outcomes`（改用 `allowed_outcomes`）
- `mandatory_fields`（改用 `required_*` 针对特定字段）
- `requires_mfa`（改用元数据字段检查，如 `required_mfa_verified`）
- 复杂的嵌套条件或运算符

---

<a id="defining-the-policy"></a>
## 定义策略

`Policy` 是一个带有自由格式 `rules` 字典的数据类——使用支持的规则模式编码你的领域所需的任何内容。

```python
from semantica.context import ContextGraph, PolicyEngine, Policy
from datetime import datetime

graph  = ContextGraph()
engine = PolicyEngine(graph_store=graph)

attribution_policy = Policy(
    policy_id   = "pol-attr-001",
    name        = "Nation-State Attribution — Dual Source + Senior Approval",
    description = (
        "Attributions to nation-state actors require corroboration from two independent "
        "intelligence sources and explicit senior analyst approval before being recorded "
        "in the authoritative graph."
    ),
    rules = {
        "min_independent_sources":  2,
        "required_approver_role":   "senior_analyst",
        "allowed_outcomes":         ["nation_state_attributed_dual_source"],
        "min_confidence":           0.85,
        "required_source_a":        True,
        "required_source_b":        True,
        "required_approver":        True,
    },
    category   = "threat_attribution",
    version    = "1.0.0",
    created_at = datetime.utcnow(),
    updated_at = datetime.utcnow(),
)

policy_id = engine.add_policy(attribution_policy)
print(f"Policy registered: {policy_id}")
# Policy registered: pol-attr-001
```

策略现在是图谱中的一个节点。它有版本字符串、创建时间戳和一个 `rules` 字典，合规检查器在根据策略评估决策时会读取该字典。

---

<a id="checking-a-decision-for-compliance"></a>
## 检查决策的合规性

`check_compliance` 接受一个 `Decision` 对象和策略 ID，返回布尔值。

```python
from semantica.context import Decision

# AI 分析师的 APT29 归因——仅引用了一个来源，尚无高级审批
decision = Decision(
    decision_id   = "",                        # 如果留空则自动生成
    category      = "threat_attribution",
    scenario      = "APT29 activity cluster in NATO network telemetry Q2 2025",
    reasoning     = (
        "Observed TTPs match APT29 historical patterns: HAMMERTOSS C2, "
        "spear-phishing via OneDrive lure, targeting foreign ministry staff. "
        "Single source: internal SIEM telemetry."
    ),
    outcome       = "nation_state_attributed_single_source",
    confidence    = 0.91,
    timestamp     = datetime.utcnow(),
    decision_maker= "ai_threat_analyst_v3",
    metadata      = {
        "independent_sources": 1,  # 低于 min_independent_sources 要求
        "approver_role": "analyst",  # 低于 required_approver_role
        "source_a": True,  # 有第一个来源
        # 缺少 source_b 和 approver 字段
    }
)

is_compliant = engine.check_compliance(decision, policy_id)
print(f"Compliant: {is_compliant}")
# Compliant: False
#
# Multiple rule violations:
# - outcome "nation_state_attributed_single_source" not in allowed_outcomes
# - independent_sources (1) < min_independent_sources (2)
# - approver_role "analyst" != required_approver_role "senior_analyst"
# - missing required_source_b and required_approver fields
```

引擎返回 `False`。决策并未被拒绝——它只是被标记了。接下来发生什么取决于你的工作流。在某些组织中，不合规的结果会直接阻止向权威图谱写入。在其他组织中，它会触发一个异常流程，由人工审批人审查证据并签批。

---

<a id="recording-a-policy-exception"></a>
## 记录策略例外

`record_exception` 将决策、违反的策略、审批人身份和理由永久关联在一起。

```python
exception_id = engine.record_exception(
    decision_id  = decision.decision_id,
    policy_id    = policy_id,
    reason       = "Time-sensitive attribution — protective action required before second source available",
    approver     = "sr_analyst_chen",
    justification= (
        "Senior analyst reviewed SIEM telemetry and concurs with TTP matching. "
        "Exception approved under Emergency Attribution Procedure §3.2. "
        "Second source corroboration to be completed within 72 hours."
    ),
)

print(f"Exception recorded: {exception_id}")
# Exception recorded: exc-pol-attr-001-20250621-001
#
# 异常节点同时关联到决策和图谱中的策略，
# 形成永久的三角溯源链：decision → exception → policy。
```

---

<a id="building-a-multi-level-approval-chain"></a>
## 构建多级审批链

对于最高风险的决策——将与政府合作伙伴共享的正式归因报告——单个审批人是不够的。需要三个人签批：团队负责人、部门主管和 CISO。`DecisionRecorder.record_approval_chain` 在一次调用中捕获所有三个审批人，将每个审批人与沟通方式和签批上下文关联。

```python
from semantica.context.decision_recorder import DecisionRecorder

recorder = DecisionRecorder(graph_store=graph)

# approvers、methods 和 contexts 必须是等长的平行列表
recorder.record_approval_chain(
    decision_id = decision.decision_id,
    approvers   = ["team_lead_okonkwo",  "dept_head_zhang",    "ciso_miller"],
    methods     = ["slack_dm",            "zoom_call",           "email"],
    contexts    = [
        "Team lead reviewed TTPs and SIEM evidence",
        "Dept head approved sharing with Five Eyes partners",
        "CISO authorised formal nation-state attribution report",
    ],
)

print("Three-level approval chain recorded")
# 图谱现在承载一条有向审批链：
#   decision → team_lead_okonkwo → dept_head_zhang → ciso_miller
# 每个链接都标记了沟通方式和上下文，供监察长审查。
```

---

<a id="what-if-impact-analysis-before-changing-a-policy"></a>
## 更改策略前的影响假设分析

六个月后，你的威胁情报主管想要收紧策略：将最低置信度阈值从 0.85 提高到 0.92，以减少误报归因。在她更新策略之前，她想知道在更严格的规则下会有多少过去的决策被阻止。

`analyze_policy_impact` 对历史决策记录进行假设模拟——不会产生永久性更改。

```python
current_policy = engine.get_policy(policy_id)

impact = engine.analyze_policy_impact(
    policy_id      = policy_id,
    proposed_rules = {**current_policy.rules, "min_confidence": 0.92},
)

print(f"Decisions affected by raising confidence floor to 0.92: {impact.get('affected_decisions', 0)}")
# Decisions affected by raising confidence floor to 0.92: 4
#
# 四个过去的归因置信度在 0.85 到 0.92 之间。
# 在新规则下，这四个都需要例外处理。
# 主管现在可以做出基于证据的选择：这种权衡是否可以接受？
```

影响字典包含每个决策的详细信息，而不仅仅是数量。你可以检查哪些具体的归因决策会受到影响，审查它们的推理，并决定是否值得收紧阈值。

---

<a id="updating-the-policy-and-finding-affected-decisions"></a>
## 更新策略并查找受影响的决策

主管决定继续提高阈值。她将策略更新到版本 1.1.0，并记录了原因。旧版本保留在历史记录中。

```python
engine.update_policy(
    policy_id     = policy_id,
    rules         = {**current_policy.rules, "min_confidence": 0.92},
    change_reason = "Q3 attribution quality review — raise confidence floor from 0.85 to 0.92 "
                    "to reduce false-positive nation-state attributions",
    new_version   = "1.1.0",
)

print(f"Policy updated: {policy_id} to version 1.1.0")
# Policy updated: pol-attr-001 to version 1.1.0

# 查找在 v1.0.0 下评估的所有决策——
# 这些需要重新审查以确认它们仍然满足新标准。
affected = engine.get_affected_decisions(
    policy_id    = policy_id,
    from_version = "1.0.0",
    to_version   = "1.1.0",
)

print(f"Decisions to re-audit: {len(affected)}")
for dec in affected:
    print(f"  {dec.get('decision_id')}  confidence={dec.get('confidence')}  outcome={dec.get('outcome')}")
# decision-a3f1  confidence=0.87  outcome=nation_state_attributed
# decision-b22c  confidence=0.89  outcome=nation_state_attributed
# ...
```

这就是重新审计工作流：每个在旧策略下做出的决策都会被呈现出来，对照新标准进行审查，然后要么重新确认，要么标记为需要修正。图谱保留了每个决策受哪个策略版本管辖的完整历史。

---

<a id="reviewing-the-full-audit-trail"></a>
## 查看完整审计追踪

在任何时候——为了监察长审查、董事会报告或事件调查——你都可以检索策略的完整版本历史。

```python
history = engine.get_policy_history(policy_id)

print(f"Policy versions on record: {len(history)}")
for version in history:
    print(f"  v{version.version}  updated={version.updated_at.date()}  rules_keys={list(version.rules.keys())}")
# v1.0.0  updated=2025-06-21  rules_keys=[min_independent_sources, required_approver_role, ...]
# v1.1.0  updated=2025-09-14  rules_keys=[min_independent_sources, required_approver_role, ...]
#
# 每个版本的完整 rules 字典都被保留——你可以精确重放
# 对于任何过去的决策，在任何过去的策略版本下，
# 合规检查会返回什么结果。
```

---

<a id="common-pitfalls"></a>
## 常见陷阱

**误以为合规失败会自动阻止操作。**PolicyEngine 返回合规状态，但不会自动阻止操作。你的工作流必须检查返回的布尔值，并决定接下来发生什么——批准、拒绝、异常处理或升级。

**使用不受支持的规则键。**该实现仅支持特定模式：`min_*`、`max_*`、`required_*`、`min_confidence`、`allowed_outcomes` 和 `required_categories`。任何其他规则键会回退到键存在性检查：仅当该确切键存在于 `decision.metadata` 中时才通过，无论其值如何。这意味着像 `disallowed_outcomes` 这样的键在该字面键不存在于元数据中时（常见情况）会静默**失败**合规检查，而在 `disallowed_outcomes` 键恰好以任何值存在于元数据中时——无论实际结果如何——会静默**通过**。这两种行为都不符合预期的"结果不得在此列表中"语义——请改用 `allowed_outcomes`。

**将例外视为批准。**使用 `record_exception()` 记录策略例外不会自动使不合规的决策变为合规。例外是审计追踪条目——你的工作流仍需决定是否继续执行该不合规决策。

**误以为 PolicyEngine 会自动修改图谱状态。**PolicyEngine 仅评估合规性并记录策略应用、例外和审批链。它不修改决策结果、元数据或阻止操作——那是你的工作流的职责。

**使用复杂的嵌套规则结构。**该实现不支持复杂条件逻辑、嵌套运算符或任意表达式。保持规则简单：仅使用单字段比较、列表成员检查和阈值校验。

**规则评估缺失元数据。**像 `min_credit_score` 这样的规则要求相应的元数据字段（`credit_score`）存在于 `decision.metadata` 中。缺失的元数据字段会导致规则评估失败，使决策变为不合规。

**忘记检查规则评估结果。**始终显式处理合规和不合规两种情况。未经过适当异常处理就继续执行的不合规决策会造成审计缺口和治理风险。

---

<a id="domain-examples"></a>
## 领域示例

<Tabs>

<Tab title="国防 — CTI/威胁">

TLP:RED 情报在未经指挥官级批准的情况下绝不可在原始组织之外共享。该策略在涉及涉密威胁报告的每个信息共享决策上执行。违规被路由到 J2 官员进行例外审查，而不是静默记录。

```python
from semantica.context import ContextGraph, PolicyEngine, Policy, Decision
from semantica.context.decision_recorder import DecisionRecorder
from datetime import datetime

graph    = ContextGraph()
engine   = PolicyEngine(graph_store=graph)
recorder = DecisionRecorder(graph_store=graph)

opsec_policy = Policy(
    policy_id   = "pol-opsec-001",
    name        = "TLP:RED — Restricted Dissemination",
    description = "TLP:RED intelligence must not be shared outside the originating organisation",
    rules = {
        "required_classification":      "TLP:RED",
        "allowed_outcomes":             ["retained_internal", "escalated_internal"],
        "min_confidence":               0.95,
        "required_tlp":                 True,
        "required_authorised_recipients": True,
    },
    category   = "information_sharing",
    version    = "2.1.0",
    created_at = datetime.utcnow(),
    updated_at = datetime.utcnow(),
)
engine.add_policy(opsec_policy)

decision = Decision(
    decision_id   = "",
    category      = "information_sharing",
    scenario      = "APT29 SIGINT report TLP:RED — share with Five Eyes partners?",
    reasoning     = "Tactical intelligence — partner request via UKIC liaison",
    outcome       = "shared_with_partner",   # 违反 allowed_outcomes 策略
    confidence    = 0.88,                    # 低于 min_confidence 阈值
    timestamp     = datetime.utcnow(),
    decision_maker= "analyst_rodriguez",
    metadata      = {
        "classification": "TLP:RED",
        "tlp": True,
        "authorised_recipients": True,
    }
)

is_compliant = engine.check_compliance(decision, "pol-opsec-001")
print(f"Compliant: {is_compliant}")
# Compliant: False — outcome 'shared_with_partner' not in allowed_outcomes; confidence below 0.95

if not is_compliant:
    # 路由到 J2 进行例外审查——需要双重指挥官批准
    exception_id = engine.record_exception(
        decision_id  = decision.decision_id,
        policy_id    = "pol-opsec-001",
        reason       = "Five Eyes partner urgent request — time-critical tactical intelligence",
        approver     = "j2_officer_hayes",
        justification= "Commander approved limited dissemination under UKUSA Article 4 emergency clause",
    )
    recorder.record_approval_chain(
        decision_id = decision.decision_id,
        approvers   = ["j2_officer_hayes", "unit_commander_brooks"],
        methods     = ["email",             "zoom_call"],
        contexts    = ["J2 tactical review", "Commander emergency approval"],
    )
    print(f"Exception recorded with dual-commander approval: {exception_id}")

# 供监察长审查的版本历史
history = engine.get_policy_history("pol-opsec-001")
print(f"Policy versions on record: {len(history)}")
```

</Tab>

<Tab title="安全 — SOC/事件">

零信任访问策略对一级系统强制执行 MFA，对特权账户强制执行 PAM 会话签出。每个自动化访问决策都根据两项策略进行检查。当 PAM 门户在紧急补丁窗口期间不可用时，SOC 主管在工作开始之前——而非之后——记录一个有时间限制的例外并签批。

```python
from semantica.context import ContextGraph, PolicyEngine, Policy, Decision
from datetime import datetime

graph  = ContextGraph()
engine = PolicyEngine(graph_store=graph)

for pol in [
    Policy(
        policy_id   = "pol-zt-mfa",
        name        = "MFA Required — All Tier-1",
        description = "Every Tier-1 access decision must verify MFA",
        rules       = {
            "required_mfa_verified":       True,
            "allowed_outcomes":            ["access_granted_with_mfa"],
            "min_confidence":              0.90,
        },
        category   = "access_control",
        version    = "1.0.0",
        created_at = datetime.utcnow(),
        updated_at = datetime.utcnow(),
    ),
    Policy(
        policy_id   = "pol-zt-pam",
        name        = "PAM Checkout — Privileged Accounts",
        description = "Privileged account use requires PAM session checkout",
        rules       = {
            "required_pam_session":        True,
            "required_session_recording":  True,
            "max_session_hours":           4,
            "allowed_outcomes":            ["privileged_access_granted_with_pam"],
        },
        category   = "privileged_access",
        version    = "1.0.0",
        created_at = datetime.utcnow(),
        updated_at = datetime.utcnow(),
    ),
]:
    engine.add_policy(pol)

# 紧急补丁窗口——PAM 门户不可用
decision = Decision(
    decision_id   = "",
    category      = "privileged_access",
    scenario      = "sysadmin_kim patching DC-04 — PAM portal unreachable",
    reasoning     = "Critical patch CVE-2025-1234 — 4-hour RTO — PAM portal offline",
    outcome       = "privileged_access_granted_no_pam",
    confidence    = 0.78,
    timestamp     = datetime.utcnow(),
    decision_maker= "soc_automation",
    metadata      = {
        "pam_session": False,      # PAM 签出失败
        "session_recording": True, # 手动记录已就位
        "session_hours": 3,        # 计划会话时长
    }
)

pam_compliant = engine.check_compliance(decision, "pol-zt-pam")
print(f"PAM policy compliant: {pam_compliant}")
# PAM policy compliant: False

# SOC 主管在工作开始前记录例外
exception_id = engine.record_exception(
    decision_id  = decision.decision_id,
    policy_id    = "pol-zt-pam",
    reason       = "PAM portal offline during critical patch window",
    approver     = "soc_lead_okafor",
    justification= "Emergency patch approved under BCP §7.3 — manual session logging in place",
)

# 假设分析：将最大会话时长从 4 小时缩减到 2 小时对董事会风险审查的影响
pam_policy = engine.get_policy("pol-zt-pam")
impact = engine.analyze_policy_impact(
    policy_id      = "pol-zt-pam",
    proposed_rules = {**pam_policy.rules, "max_session_hours": 2},
)
print(f"Decisions affected by tighter session cap: {impact.get('affected_decisions', 0)}")
```

</Tab>

<Tab title="生命科学 — 临床/制药">

一项绝对药物禁忌策略阻止在 eGFR 低于 30 时处方二甲双胍。AI 临床决策支持系统在每次处方决策到达 EHR 之前都根据此策略进行检查。当边缘病例需要临床例外时，在处方签发之前记录一条三人 MDT 审批链——顾问医生、肾脏专科医生和药剂师。

```python
from semantica.context import ContextGraph, PolicyEngine, Policy, Decision
from semantica.context.decision_recorder import DecisionRecorder
from datetime import datetime

graph    = ContextGraph()
engine   = PolicyEngine(graph_store=graph)
recorder = DecisionRecorder(graph_store=graph)

safety_policy = Policy(
    policy_id   = "pol-clin-001",
    name        = "Metformin Absolute Contraindication — eGFR < 30",
    description = "Metformin must not be prescribed when eGFR is below 30 ml/min/1.73m²",
    rules = {
        "min_egfr":                    30,      # eGFR 必须 >= 30
        "allowed_outcomes":            ["metformin_discontinued", "metformin_contraindicated", "alternative_prescribed"],
        "required_clinician_sign_off": True,
        "required_egfr_check":         True,
    },
    category   = "clinical_safety",
    version    = "3.0.0",   # 对齐 BNF 2024
    created_at = datetime.utcnow(),
    updated_at = datetime.utcnow(),
    metadata   = {"source": "BNF_2024", "strength": "absolute"},
)
engine.add_policy(safety_policy)

# AI 决策支持建议
decision = Decision(
    decision_id   = "",
    category      = "treatment_modification",
    scenario      = "PT-00841: T2DM, eGFR 28 — metformin review",
    reasoning     = "eGFR 28 is below the 30 threshold; discontinue metformin and initiate SGLT2i",
    outcome       = "metformin_discontinued_dapagliflozin_initiated",
    confidence    = 0.97,
    timestamp     = datetime.utcnow(),
    decision_maker= "cdss_v4",
    metadata      = {
        "egfr": 28,                    # 低于最低阈值
        "clinician_sign_off": True,
        "egfr_check": True,
        "drug": "metformin",
    }
)

is_compliant = engine.check_compliance(decision, "pol-clin-001")
print(f"Compliant: {is_compliant}")
# Compliant: True — outcome is metformin_discontinued, which is allowed

# 处方记录的 MDT 审批链
recorder.record_approval_chain(
    decision_id = decision.decision_id,
    approvers   = ["dr_okonkwo",  "consultant_renal_patel", "pharmacist_kwon"],
    methods     = ["zoom_call",    "zoom_call",               "email"],
    contexts    = [
        "Prescribing physician MDT review",
        "Renal consultant confirmed eGFR 28 — supports discontinuation",
        "Clinical pharmacist approved SGLT2i initiation protocol",
    ],
)
print("MDT approval chain recorded — prescription safe to issue")

# 供 CQC 检查的策略版本历史
history = engine.get_policy_history("pol-clin-001")
print(f"Policy versions on record: {len(history)}")
```

</Tab>

<Tab title="银行 — 风险/合规">

一项 Basel III 按揭承保策略将 LTV 限制在 85%，DSTI 限制在 40%。信贷模型在每次发起决策入账之前都根据此策略进行检查。当信贷委员会想在下季度之前提高最低信用分底线时，影响分析显示当前账簿中有多少已批准贷款在新规则下不会通过——策略变更的审计追踪会提交给董事会风险委员会。

```python
from semantica.context import ContextGraph, PolicyEngine, Policy, Decision
from datetime import datetime

graph  = ContextGraph()
engine = PolicyEngine(graph_store=graph)

mortgage_policy = Policy(
    policy_id   = "pol-credit-001",
    name        = "Retail Mortgage Underwriting — Basel III CRE20",
    description = "Standard residential mortgage policy aligned to Basel III CRE20",
    rules = {
        "max_ltv":                         0.85,
        "max_dsti":                        0.40,
        "min_credit_score":                680,
        "min_stress_test_bps":             300,
        "allowed_outcomes":                ["approved", "approved_with_conditions"],
    },
    category   = "credit_risk",
    version    = "2.3.0",
    created_at = datetime.utcnow(),
    updated_at = datetime.utcnow(),
    metadata   = {"regulatory_basis": "Basel_III_CRE20", "effective_date": "2025-01-01"},
)
engine.add_policy(mortgage_policy)

# 边缘决策——LTV 86%，超限一个百分点
decision = Decision(
    decision_id   = "",
    category      = "mortgage_origination",
    scenario      = "APP-2025-9921: £320k mortgage, LTV 86%, DSTI 38%, credit score 710",
    reasoning     = (
        "LTV 86% exceeds 85% cap. Stress test at +300bps passes. "
        "Credit score 710 above 680 floor. DSTI 38% within 40% limit."
    ),
    outcome       = "approved_ltv_exception",   # 不在 allowed_outcomes 中——标记为不合规
    confidence    = 0.72,
    timestamp     = datetime.utcnow(),
    decision_maker= "underwriting_model_v4",
    metadata      = {
        "ltv": 0.86,           # 超过 max_ltv 0.85
        "dsti": 0.38,          # 在 max_dsti 0.40 之内
        "credit_score": 710,   # 高于 min_credit_score 680
        "pd": 0.023,           # 记录用于审计——此策略中无阈值规则
        "lgd": 0.45,           # 记录用于审计——此策略中无阈值规则
        "stress_test_bps": 300,
    }
)

is_compliant = engine.check_compliance(decision, "pol-credit-001")
print(f"Compliant: {is_compliant}")
# Compliant: False — ltv (0.86) > max_ltv (0.85) and outcome not in allowed_outcomes

if not is_compliant:
    exception_id = engine.record_exception(
        decision_id  = decision.decision_id,
        policy_id    = "pol-credit-001",
        reason       = "LTV 86% — one point over cap; stress test and all other metrics pass",
        approver     = "sr_underwriter_walsh",
        justification= "Credit committee approved exception under high-quality borrower criteria §4.1",
    )
    print(f"Exception recorded: {exception_id}")

# 提高信用分底线前的假设分析——提交董事会风险委员会
impact = engine.analyze_policy_impact(
    policy_id      = "pol-credit-001",
    proposed_rules = {**mortgage_policy.rules, "min_credit_score": 700},
)
print(f"Decisions affected by raising credit score floor to 700: {impact.get('affected_decisions', 0)}")

# 查找在 v2.3.0 下做出的所有决策——策略更新后重新审计
affected = engine.get_affected_decisions("pol-credit-001", "2.3.0", "2.4.0")
print(f"Decisions to re-audit: {len(affected)}")

# 提交更新——版本升级附带完整变更归因
engine.update_policy(
    policy_id     = "pol-credit-001",
    rules         = {**mortgage_policy.rules, "min_credit_score": 700},
    change_reason = "Credit committee Q3 review — raise score floor from 680 to 700",
    new_version   = "2.4.0",
)
print("Policy updated to v2.4.0")
```

</Tab>

</Tabs>

---

<a id="related-guides"></a>
## 相关指南

- [决策智能](decision-intelligence.zh-CN.md) —— `record_decision()`、因果链和先例搜索——`check_compliance()` 所评估的决策
- [推理与规则](reasoning.zh-CN.md) —— 用形式化推断补充策略规则，用于逻辑冲突检测
- [SHACL 校验](shacl-validation.zh-CN.md) —— 对策略节点本身强制执行结构约束
- [变更管理](change-management.zh-CN.md) —— 与知识图谱一起对策略图谱进行版本快照
- [溯源](provenance.zh-CN.md) —— 每个策略决策和例外的 W3C PROV-O 血缘
- [MCP 服务端](mcp-server.zh-CN.md) —— 将 `record_decision` 和 `find_precedents` 作为 MCP 工具暴露给 AI 智能体
