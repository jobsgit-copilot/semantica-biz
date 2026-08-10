---
title: "评估模块"
description: "用于度量知识图谱质量、抽取准确率和流水线性能的评估框架：即将推出。"
icon: "chart-line"
---

**[English](evals.md)** · **简体中文（当前）**

**`semantica.evals`** 计划成为一个综合评估框架，用于度量**抽取准确率、图谱质量和流水线性能**。

<Warning>
  **`semantica.evals` 尚未实现。** 该模块是一个占位符，`__all__ = []`。没有任何类或函数可供导入。本页面仅描述规划中的 API。
</Warning>

<a id="planned-features"></a>
## 规划中的特性

发布后，`semantica.evals` 将提供：

| 规划中的类 | 角色 |
| :--- | :--- |
| `KGEvaluator` | 完整性、一致性、模式合规性、覆盖率以及孤儿节点检测 |
| `ExtractionEvaluator` | 针对金标准数据集的 NER 精确率 / 召回率 / F1 与关系抽取指标 |
| `PipelineBenchmark` | 吞吐量（文档/秒）、每步延迟、峰值内存和错误率 |
| `RegressionTracker` | 记录运行结果，并跨提交或配置变更比较指标 |
| `EvalReport` | 结构化报告：`{scores, regressions, recommendations}` |
| `DeduplicationEvaluator` | 合并精确率、假阳性 / 假阴性率 |
| `ReasoningEvaluator` | 推理准确率、规则覆盖率和推导深度 |

<a id="current-workaround"></a>
## 当前的临时方案

在 `semantica.evals` 发布之前，可使用 `semantica.ontology.OntologyEvaluator` 获取本体质量指标：

```python
from semantica.ontology import OntologyEvaluator

evaluator = OntologyEvaluator()

# evaluate_ontology 只接受本体字典
result = evaluator.evaluate_ontology(ontology)

print("Coverage:    ", result.coverage_score)
print("Completeness:", result.completeness_score)
print("Gaps:        ", result.gaps)
print("Suggestions: ", result.suggestions)

# 带类粒度和关系完整性的完整报告
report = evaluator.generate_report(ontology)
print("Coverage score:    ", report["evaluation"]["coverage_score"])
print("Completeness score:", report["evaluation"]["completeness_score"])
print("Relation coverage: ", report["relation_completeness"]["relation_coverage"])
```

由 `evaluate_ontology()` 返回的 `EvaluationResult` 字段：

| 字段 | 类型 | 描述 |
| :----- | :---- | :----------- |
| `coverage_score` | `float` | 本体可解答的能力问题占比 |
| `completeness_score` | `float` | 类完整性与属性完整性得分的平均值 |
| `gaps` | `List[str]` | 已识别出的覆盖缺口 |
| `suggestions` | `List[str]` | 改进建议 |
| `metrics` | `dict` | 详细的子指标 |

- [语义抽取](semantic_extract.zh-CN.md) —— 抽取模块。
- [知识图谱](kg.zh-CN.md) —— 图谱质量评估。
- [流水线](pipeline.zh-CN.md) —— 流水线性能指标。
- [本体评估器](ontology.zh-CN.md) —— 现在即可用于获取本体质量指标。
