---
title: "贡献指南"
description: "如何为 Semantica 贡献代码、文档、测试和社区支持。"
icon: "code-pull-request"
---

**[English](contributing-guide.md)** · **简体中文（当前）**

欢迎各种形式的贡献：代码、文档、测试和社区支持。每一份贡献都会在发布说明和 GitHub 贡献者列表中得到认可。


<a id="quick-start"></a>
## 快速上手

```bash
# 在 GitHub 上 Fork 本仓库，然后：
git clone https://github.com/your-username/semantica.git
cd semantica
pip install -e ".[dev]"
pytest
```

刚接触本项目？请从标记为 [`good-first-issue`](https://github.com/semantica-agi/semantica/labels/good-first-issue) 的工单开始：它们的范围设定为在几个小时内即可完成，无需深入了解代码库。


<a id="ways-to-contribute"></a>
## 贡献方式

- **代码** — 修复 bug、实现功能、优化性能，或使用插件注册表添加新的摄取器、解析器和导出器。
- **文档** — 修复拼写错误、提升清晰度、补充缺失的示例、编写教程，或在模块演进时保持 API 参考文档的准确性。
- **测试** — 为未测试的模块或边界情况添加测试覆盖，用最小复现重现已报告的 bug，或提升跨平台的可靠性。
- **社区** — 在 GitHub Issues 和 Discussions 中回答问题，以建设性的反馈审查 pull request，或在博客文章和演讲中分享 Semantica。


<a id="development-setup"></a>
## 开发环境搭建

```bash
git clone https://github.com/your-username/semantica.git
cd semantica
pip install -e ".[dev]"
```

**代码风格工具：**

```bash
pytest                      # 完整测试套件
black semantica/ tests/     # 自动格式化
isort semantica/ tests/     # 排序导入
flake8 semantica/           # 代码检查
```

风格约定：使用 **Black** 进行格式化，**isort** 排序导入，**flake8** 进行代码检查。这三者都会在 CI 中运行。


<a id="reporting-issues"></a>
## 报告问题

**Bug 报告**应包含：

- 实际发生了什么与预期是什么
- 最小复现步骤
- 你的环境：Python 版本、操作系统、Semantica 版本（`python -c "import semantica; print(semantica.__version__)"`）

**功能请求**应包含：

- 你的具体使用场景
- 你希望 Semantica 做什么
- 为什么它能让广泛的用户受益，而不仅仅是你的特定工作流


<a id="pull-request-checklist"></a>
## Pull Request 检查清单

提交 PR 之前，请确认：

<Check>本地测试通过：`pytest`</Check>
<Check>新功能包含带有可运行代码示例的文档</Check>
<Check>代码遵循项目风格：Black、isort、flake8</Check>
<Check>提交信息清晰，并描述"*为什么*"而不仅仅是"*做了什么*"</Check>
<Check>没有未解决的合并冲突</Check>


<a id="code-of-conduct"></a>
## 行为准则

所有贡献者都应遵循 [Contributor Covenant 行为准则](https://github.com/semantica-agi/semantica/blob/main/CODE_OF_CONDUCT.md)。请保持尊重、耐心和建设性：尤其是对待新人。如要举报违规行为，请提交一个带有 `[CoC]` 前缀的 issue。


<a id="help"></a>
## 获取帮助

- [GitHub Issues](https://github.com/semantica-agi/semantica/issues)
- [GitHub Discussions](https://github.com/semantica-agi/semantica/discussions)
- [Discord](https://discord.gg/sV34vps5hH)

- [社区](community.zh-CN.md) — 社区准则和价值观。
- [治理](governance.zh-CN.md) — 决策制定和项目运作方式。
