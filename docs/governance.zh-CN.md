---
title: "治理"
description: "项目治理模型：角色、决策流程、发布节奏和代码审查指南。"
icon: "scale-balanced"
---

**[English](governance.md)** · **简体中文（当前）**

> Semantica 由 Hawksight AI 维护，并在开放的治理模型下接受社区贡献。


<a id="roles"></a>
## 角色

- **维护者** — Hawksight AI 团队：审查并合并 PR、管理发布和代码质量、设定项目方向和社区标准。
- **贡献者** — 提交代码、文档和 bug 报告。协助处理 issue 和审查。在 [CONTRIBUTORS.md](https://github.com/semantica-agi/semantica/blob/main/CONTRIBUTORS.md) 中获得认可。
- **社区成员** — 使用 Semantica、提供反馈、分享用例，并参与 GitHub Discussions 和 Discord。


<a id="decision-process"></a>
## 决策流程

<a id="code-changes"></a>
### 代码变更

<Steps>
  <Step title="提案">提交一个描述该变更的 GitHub Issue。</Step>
  <Step title="讨论">就方法和范围进行社区讨论。</Step>
  <Step title="实现">提交包含该变更的 pull request。</Step>
  <Step title="审查">至少一名维护者审查该 PR。</Step>
  <Step title="合并">在 CI 通过并获得维护者批准后合并。</Step>
</Steps>

<a id="major-decisions"></a>
### 重大决策

- 在 GitHub Issues 中发布 RFC
- 至少 1 周的社区讨论期
- 维护者根据社区反馈和技术可行性做出决定


<a id="releases"></a>
## 发布

Semantica 遵循**语义化版本**（`MAJOR.MINOR.PATCH`）：

| 级别 | 触发条件 | 节奏 |
| :------- | :--------- | :--------- |
| MAJOR | 破坏性变更 | 每季度或按需 |
| MINOR | 新功能（向后兼容） | 每月或准备就绪时 |
| PATCH | Bug 修复（向后兼容） | 修复 bug 时 |


<a id="code-review"></a>
## 代码审查

**审查标准：** 功能、代码质量、测试、文档、性能、安全性。

**时间线：** 48 小时内进行初步审查；7 天内进行后续跟进。

**审查者指南：** 保持建设性，解释理由，提供替代方案。

**贡献者指南：** 及时处理评论，不清楚时提问，对反馈保持开放态度。


<a id="communication"></a>
## 沟通渠道

- **GitHub Issues**：bug 报告、功能请求、问题
- **GitHub PRs**：代码贡献
- **GitHub Discussions**：社区交流
- **Security Advisories**：[私下报告安全问题](https://github.com/semantica-agi/semantica/security/advisories/new)


<a id="project-goals"></a>
## 项目目标

- **易用性** — 易于使用和理解：合理的默认值、清晰的文档、最少的繁文缛节。
- **可靠性** — 生产级质量：跨 Python 版本、平台和实际工作负载进行测试。
- **性能** — 高效且可扩展：从单机笔记本到企业级图数据库。
- **可扩展性** — 通过 `PluginRegistry` 模式轻松使用插件和自定义模块进行扩展。
- **社区** — 热情包容：所有背景和经验水平的人都能参与贡献并获得认可。


<a id="license"></a>
## 许可证

MIT 许可证：参见 [LICENSE](https://github.com/semantica-agi/semantica/blob/main/LICENSE) 和[许可证页面](project-license.zh-CN.md)。


<a id="see-also"></a>
## 另请参阅

- [贡献指南](contributing-guide.zh-CN.md) — 如何提交变更。
- [社区](community.zh-CN.md) — 社区准则和渠道。
