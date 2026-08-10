---
title: "Cookbook（实战手册）"
description: "交互式 Jupyter notebook，涵盖从你的第一个知识图谱到生产级 GraphRAG 系统的方方面面。"
icon: "flask"
---

**[English](cookbook.md)** · **简体中文（当前）**

> **这本手册是什么：** 一组可运行的 Jupyter notebook，从你的第一个知识图谱一路讲到生产级 GraphRAG 系统。每本 notebook 都自包含、可在 5 分钟内跑通。
>
> **学习路径：** 新手从[核心教程](#核心教程)按顺序学（数据摄取 → 解析 → 抽取 → 图谱构建）；有基础后进入[进阶概念](#进阶概念)做图分析、推理、多源集成与可视化。找不到方向？先跑[精选配方——你的第一个知识图谱](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/08_Your_First_Knowledge_Graph.ipynb)。

<Tip>
  **从哪里开始：**
  - **Semantica 新手**：从 [核心教程](#核心教程) 开始
  - **正在构建应用**：参见 [进阶概念](#进阶概念)
  - **需要安装帮助**：参见[安装指南](installation.zh-CN.md)
</Tip>

<Note>
  前置条件：Python 3.8+、Jupyter，以及你所偏好的 LLM 提供商的 API 密钥。
</Note>


## 精选配方

- **[你的第一个知识图谱](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/08_Your_First_Knowledge_Graph.ipynb)** — 在 20 分钟内从原始文本构建出一个可查询的知识图谱。主题：抽取、图谱构建、可视化 · *入门*


## 核心教程

掌握 Semantica 框架的必备指南。

- **[欢迎来到 Semantica](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/01_Welcome_to_Semantica.ipynb)** — 对框架核心理念与全部模块的交互式介绍。主题：框架概览、架构 · *入门*
- **[数据摄取](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/02_Data_Ingestion.ipynb)** — 从文件、网页、数据库、流、订阅源、仓库、邮件和 MCP 加载数据。主题：FileIngestor、WebIngestor、DBIngestor · *入门*
- **[文档解析](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/03_Document_Parsing.ipynb)** — 从 PDF、DOCX、HTML 等复杂格式中提取干净的文本。主题：OCR、PDF 解析、文本抽取 · *入门*
- **[数据归一化](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/04_Data_Normalization.ipynb)** — 用于清洗、归一化和准备文本的流水线。主题：文本清洗、Unicode、格式化 · *入门*
- **[实体抽取](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/05_Entity_Extraction.ipynb)** — 使用 NER 识别人物、组织和自定义实体。主题：NER、spaCy、LLM 抽取 · *入门*
- **[关系抽取](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/06_Relation_Extraction.ipynb)** — 发现并对实体之间的关系进行分类。主题：关系分类、依存句法分析 · *入门*
- **[向量嵌入生成](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/12_Embedding_Generation.ipynb)** — 为语义搜索创建并管理向量嵌入。主题：嵌入、OpenAI、HuggingFace · *进阶*
- **[向量库](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/13_Vector_Store.ipynb)** — 搭建用于相似度搜索与检索的向量库。*进阶*
- **[图存储](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/09_Graph_Store.ipynb)** — 把知识图谱持久化到 Neo4j 或 FalkorDB。主题：Neo4j、Cypher、持久化 · *进阶*
- **[本体](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/14_Ontology.ipynb)** — 定义领域模式与本体来结构化你的数据。主题：OWL、RDF、模式设计 · *进阶*


## 进阶概念

深入探讨高级特性、定制化与复杂工作流。

- **[高级抽取](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/01_Advanced_Extraction.ipynb)** — 自定义抽取器、基于 LLM 的抽取和复杂模式匹配。主题：自定义模型、正则、LLM · *高级*
- **[高级图分析](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/02_Advanced_Graph_Analytics.ipynb)** — 中心性、社区发现和寻路算法。主题：PageRank、Louvain、最短路径 · *高级*
- **[高级上下文工程](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/11_Advanced_Context_Engineering.ipynb)** — 使用 FAISS 和 Neo4j 为 AI 智能体打造生产级记忆系统。主题：智能体记忆、GraphRAG、实体注入 · *高级*
- **[完整可视化套件](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/03_Complete_Visualization_Suite.ipynb)** — 你的图谱的交互式、出版级可视化。主题：PyVis、NetworkX、D3.js · *进阶*
- **[冲突解决](https://github.com/semantica-agi/semantica/blob/main/cookbook/introduction/17_Conflict_Detection_and_Resolution.ipynb)** — 处理来自多个来源的矛盾信息的策略。主题：真相发现、投票、置信度 · *高级*
- **[多格式导出](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/05_Multi_Format_Export.ipynb)** — 导出为 RDF、OWL、JSON-LD 和 NetworkX 格式。主题：序列化、互操作 · *进阶*
- **[多源集成](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/06_Multi_Source_Data_Integration.ipynb)** — 把来自不同来源的数据合并到一张统一的图中。主题：实体解析、合并、融合 · *高级*
- **[推理与推断](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/08_Reasoning_and_Inference.ipynb)** — 用逻辑推理从已有事实中推断新知识。主题：逻辑规则、推理引擎 · *高级*
- **[时态知识图谱](https://github.com/semantica-agi/semantica/blob/main/cookbook/advanced/10_Temporal_Knowledge_Graphs.ipynb)** — 对随时间变化的数据进行建模与查询。主题：时间序列、时态逻辑、Allen 代数 · *高级*


## 如何运行

<Steps>
  <Step title="安装 Semantica">
    ```bash
    pip install semantica[all]
    pip install jupyter
    ```
  </Step>
  <Step title="克隆仓库（可选，用于从源码安装）">
    ```bash
    git clone https://github.com/semantica-agi/semantica.git
    cd semantica
    pip install -e ".[all]"
    pip install jupyter
    ```
  </Step>
  <Step title="启动 Jupyter">
    ```bash
    jupyter notebook
    ```
  </Step>
</Steps>

<Tip>
  你也可以使用 Docker 来运行 cookbook：

  ```bash
  docker run -p 8888:8888 hawksight/semantica-cookbook
  ```
</Tip>
