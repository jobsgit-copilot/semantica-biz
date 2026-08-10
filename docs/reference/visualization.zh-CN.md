---
title: "可视化模块"
description: "交互式与静态的知识图谱、本体、向量嵌入与时序可视化。"
icon: "chart-bar"
---

**[English](visualization.md)** · **简体中文（当前）**

**`semantica.visualization`** 将知识图谱、本体、向量嵌入空间和时序数据渲染为 **交互式 HTML 或静态图像**：无需启动完整的 Explorer 服务器：

- `KGVisualizer`：带力导向、层次和环形布局的交互式网络
- `EmbeddingVisualizer`：带聚类标签的 2D/3D UMAP 或 t-SNE 投影
- `TemporalVisualizer`：时间线视图与跨快照的图谱演化
- `AnalyticsVisualizer`：中心性分数、社区结构、度分布图表

需要 `plotly`：`pip install plotly`。某些导出器还需要 `matplotlib` 或 `graphviz`。


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `KGVisualizer` | 带 force/hierarchical/circular 布局的交互式网络、社区与子图渲染 |
| `OntologyVisualizer` | 从任意本体生成类层次结构与属性关系图 |
| `EmbeddingVisualizer` | 带聚类标签的向量嵌入空间 2D/3D UMAP 或 t-SNE 投影 |
| `SemanticNetworkVisualizer` | 加权语义网络渲染 |
| `AnalyticsVisualizer` | 中心性分数、社区结构、连通性与度分布图表 |
| `TemporalVisualizer` | 时间线视图与跨快照的图谱演化 |

<a id="quick-start"></a>
## 快速上手

<Steps>
  <Step title="渲染知识图谱">
    ```python
    from semantica.visualization import KGVisualizer

    viz = KGVisualizer(layout="force", color_scheme="default")

    # 交互式：在浏览器中打开，支持悬停和点击
    viz.visualize_network(graph, output="interactive")
    ```
  </Step>
  <Step title="应用布局与颜色选项">
    ```python
    viz = KGVisualizer(layout="force", color_scheme="vibrant")

    viz.visualize_network(
        graph,
        output="html",
        file_path="graph.html",
        node_color_by="type",      # 按实体类型属性为节点着色
    )
    ```
  </Step>
  <Step title="导出为静态格式">
    ```python
    # 静态 PNG：用于报告和嵌入文档
    viz.visualize_network(graph, output="png", file_path="graph.png")

    # 矢量 SVG：用于出版物和可缩放图表
    viz.visualize_network(graph, output="svg", file_path="graph.svg")
    ```
  </Step>
</Steps>

<Warning>
  **所有可视化器都需要 `plotly`。** 使用前安装：`pip install plotly`。如果未安装 Plotly，所有可视化器方法都会抛出 `ProcessingError`。
</Warning>

<a id="visualizers"></a>
## 可视化器

<Tabs>
  <Tab title="KGVisualizer">
    交互式与静态的知识图谱渲染：

    ```python
    from semantica.visualization import KGVisualizer

    viz = KGVisualizer(layout="force", color_scheme="default")

    # 交互式：在浏览器中打开
    viz.visualize_network(graph, output="interactive")

    # 保存为 HTML 文件
    viz.visualize_network(graph, output="html", file_path="graph.html")

    # 静态 PNG
    viz.visualize_network(graph, output="png", file_path="graph.png")

    # 按社区着色的图
    viz.visualize_communities(graph, communities, file_path="communities.html")

    # 按中心性调整大小的节点
    viz.visualize_centrality(graph, centrality, centrality_type="degree")

    # 实体类型分布柱状图
    viz.visualize_entity_types(graph, output="interactive")

    # 关系频率热力图
    viz.visualize_relationship_matrix(graph, output="interactive")
    ```

    <Warning>
      **对大型图使用 `max_nodes`。** 力导向布局在约 1,000 个节点以上会变得难以阅读且缓慢。可视化大型图之前先过滤到子图。
    </Warning>

    <Tip>
      **HTML 输出始终是最佳起点。** 交互式 HTML 让你可以缩放、平移和悬停查看详情。仅在嵌入报告时才导出为 PNG/SVG/PDF。
    </Tip>

    <Tip>
      **对于交互式仪表板，优先使用 Explorer。** `KGVisualizer.visualize_network()` 生成自包含的 HTML 文件。Explorer CLI（`semantica-explorer`）提供完整的实时 Web 应用，支持搜索、过滤、路径查找与 REST API。
    </Tip>

    **布局选项（`layout=`）：**

    | 布局 | 描述 | 最适合 |
    | :------ | :----------- | :-------- |
    | `force` | 物理模拟：聚类自然涌现 | 通用图 |
    | `hierarchical` | 自上而下的树形布局 | 分类体系、组织结构图 |
    | `circular` | 节点排列在圆上，边作为弦 | 小型稠密图 |
  </Tab>
  <Tab title="OntologyVisualizer">
    可视化类层次结构与属性关系：

    ```python
    from semantica.visualization import OntologyVisualizer

    viz = OntologyVisualizer()

    # 类层次结构树
    viz.visualize_hierarchy(ontology, output="interactive")

    # 属性 domain/range 图
    viz.visualize_properties(ontology, output="html", file_path="properties.html")

    # 完整结构网络（类 + 属性）
    viz.visualize_structure(ontology, output="interactive")

    # 类-属性矩阵热力图
    viz.visualize_class_property_matrix(ontology, output="html", file_path="matrix.html")

    # 本体指标仪表板
    viz.visualize_metrics(ontology, output="interactive")
    ```
  </Tab>
  <Tab title="EmbeddingVisualizer">
    将高维向量嵌入投影到 2D 以进行聚类分析：

    ```python
    from semantica.visualization import EmbeddingVisualizer

    viz = EmbeddingVisualizer()

    viz.visualize_2d_projection(
        embeddings=embeddings,
        labels=labels,
        output="interactive",
        file_path="embeddings.html",
        method="umap",    # "umap" | "tsne" | "pca"
    )
    ```

    | 方法 | 速度 | 保持 | 最适合 |
    | :------ | :----- | :--------- | :-------- |
    | `umap` | 快 | 全局 + 局部结构 | 大型数据集、聚类发现 |
    | `tsne` | 中 | 局部结构 | 紧密聚类分离 |
    | `pca` | 很快 | 方差 | 快速概览、线性结构 |

    <Tip>
      **在规模化数据上 UMAP 比 t-SNE 更快。** 对于超过 5,000 个点的向量嵌入空间，UMAP 在几秒内完成；t-SNE 可能需要几分钟。两者都能产生良好的聚类分离。
    </Tip>
  </Tab>
  <Tab title="TemporalVisualizer">
    可视化知识图谱如何随时间变化：

    ```python
    from semantica.visualization import TemporalVisualizer

    viz = TemporalVisualizer()

    # 实体/关系变更时间线
    viz.visualize_timeline(temporal_data, output="interactive")

    # 动画网络演化：每个时间步一帧
    viz.visualize_network_evolution(temporal_kg, output="html", file_path="evolution.html")

    # 并排快照对比
    # snapshots：将时间戳字符串映射到图字典的 dict
    snapshots = {
        "2023-01": graph_v1,
        "2024-01": graph_v2,
    }
    viz.visualize_snapshot_comparison(snapshots, output="html", file_path="diff.html")

    # 时序模式：传递 pattern 字典列表
    viz.visualize_temporal_patterns(patterns, output="html", file_path="patterns.html")

    # 指标随时间的演化
    viz.visualize_metrics_evolution(metrics_history, timestamps, output="interactive")
    ```
  </Tab>
  <Tab title="AnalyticsVisualizer">
    可视化图分析结果：中心性、社区与度分布：

    ```python
    from semantica.visualization import AnalyticsVisualizer

    viz = AnalyticsVisualizer()

    # 按中心性度量排列的前 N 个节点的柱状图
    # 参数为 centrality_type=（不是 metric=）和 top_n=（不是 top_k=）
    viz.visualize_centrality_rankings(
        centrality,
        centrality_type="pagerank",
        top_n=20,
        output="html",
        file_path="centrality.html",
    )

    # 按社区着色的网络图
    viz.visualize_community_structure(kg, communities, output="html", file_path="communities.html")

    # 度分布直方图
    viz.visualize_degree_distribution(kg, output="html", file_path="degree_dist.html")

    # 连通性分析（连通/不连通、组件大小）
    viz.visualize_connectivity(connectivity, output="interactive")

    # 完整指标仪表板（节点、边、密度、直径）
    viz.visualize_metrics_dashboard(metrics, output="interactive")

    # 并排比较多个中心性度量
    viz.visualize_centrality_comparison(centrality_results, top_n=10)
    ```
  </Tab>
</Tabs>

<a id="color-schemes"></a>
## 配色方案

所有可视化器都接受 `color_scheme=` 构造函数参数：

```python
viz = KGVisualizer(color_scheme="vibrant")
```

| 方案 | 描述 | 最适合 |
| :------ | :----------- | :-------- |
| `default` | 蓝灰色调色板 | 通用 |
| `vibrant` | 高对比度、高饱和度颜色 | 演示 |
| `pastel` | 柔和、低饱和色调 | 浅色背景 |
| `dark` | 深色背景配明亮节点 | 深色模式仪表板 |
| `light` | 白色背景，细边 | 出版物、印刷 |
| `colorblind` | Okabe-Ito 安全调色板 | 无障碍 |

<Tip>
  **在出版物和仪表板中使用 `color_scheme="colorblind"`。** Okabe-Ito 调色板对所有人都可读，包括约 8% 的红绿色盲读者。
</Tip>

<a id="export-formats"></a>
## 导出格式

| 格式 | 交互式 | 可缩放 | 最适合 |
| :------ | :----------- | :-------- | :-------- |
| `.html` | 是 | N/A | Web 仪表板、探索性分析 |
| `.png` | 否 | 否 | 报告、Jupyter notebook |
| `.svg` | 否 | 是 | 出版物、幻灯片 |
| `.pdf` | 否 | 是 | 印刷、合规导出 |

<a id="convenience-functions"></a>
## 便捷函数

```python
from semantica.visualization import (
    visualize_kg, visualize_ontology, visualize_embeddings,
    visualize_semantic_network, visualize_analytics, visualize_temporal,
)

# 返回 Plotly figure 或 None
fig = visualize_kg(graph, output="interactive", method="default")
fig = visualize_ontology(ontology, output="interactive", method="hierarchy")
fig = visualize_embeddings(embeddings, labels, output="interactive", method="2d_projection")
fig = visualize_analytics(analytics_data, output="interactive", method="centrality")
fig = visualize_temporal(temporal_data, output="interactive", method="timeline")
```

<a id="graph-explorer-full-dashboard"></a>
## 图谱浏览器（完整仪表板）

要使用带搜索、路径查找和 Ontology Hub 的完整浏览器 UI，启动 Explorer CLI：

```bash
semantica-explorer --graph my_graph.json
```

有关完整功能集和 REST API，请参见 [Explorer 参考](explorer.zh-CN.md)。

- [Knowledge Graph](kg.zh-CN.md) — 正在被可视化的图。
- [Ontology](ontology.zh-CN.md) — 可视化本体类结构。
- [Embeddings](embeddings.zh-CN.md) — 生成本处可视化的向量嵌入。
- [Explorer](explorer.zh-CN.md) — 完整的交互式知识浏览器 UI。
