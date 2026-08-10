---
title: "向量嵌入模块"
description: "文本与图向量嵌入生成：FastEmbed、Sentence-Transformers、OpenAI、BGE：附带池化策略与提供商无关的 API。"
icon: "vector-square"
---

**[English](embeddings.md)** · **简体中文（当前）**

**`semantica.embeddings`** 将文本与图结构转换为**稠密向量表示**：

- 提供商无关的 API：FastEmbed（默认，ONNX，无需 GPU）、Sentence-Transformers、OpenAI、BGE
- 为语义搜索、实体解析、GraphRAG 检索和去重提供动力
- `GraphEmbeddingManager` 为图数据库后端嵌入 KG 节点与边
- 五种池化策略：Mean（默认）、Max、CLS、Attention、Hierarchical
- `check_available_providers()` 显示你的环境中安装了哪些后端


<a id="why-embeddings-matter"></a>
## 为什么向量嵌入很重要

原始文本无法进行数学比较。向量嵌入将含义转化为几何：两个语义相似的句子会产生在高维空间中彼此接近的向量，即使它们没有一个共同单词。

Semantica 将向量嵌入用于：

- **语义搜索**：按含义而非仅仅是关键词查找知识图谱节点
- **实体解析**：检测出 "Apple Inc." 与 "Apple Computer" 指向同一实体
- **去重**：`semantic_v2` 策略通过向量嵌入距离度量实体相似度
- **GraphRAG 检索**：向量 + 图遍历的混合方式，为 LLM 提供有据可依的回答
- **语义分块**：在 `TextSplitter(method="semantic_transformer")` 中检测主题切换边界

<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `EmbeddingGenerator` | 提供商无关的入口：处理批处理与提供商选择 |
| `TextEmbedder` | 文本向量嵌入，自动批处理；默认使用 FastEmbed |
| `GraphEmbeddingManager` | 为 GraphRAG 和图数据库嵌入 KG 节点与边 |
| `VectorEmbeddingManager` | 为向量数据库后端准备并格式化向量嵌入 |
| `OpenAIStore` | OpenAI `text-embedding-3-small` / `text-embedding-3-large` 提供商 |
| `BGEStore` | 通过 `sentence-transformers` 使用 BAAI/bge 模型 |
| `FastEmbedStore` | ONNX 加速的本地向量嵌入：不**要求** CUDA |
| `LlamaStore` | 占位存储：未达到生产可用；请勿用于向量嵌入 |
| `MeanPooling` | 默认池化策略：最适合检索与聚类 |

<a id="what-you-get"></a>
## 你会得到什么

- **EmbeddingGenerator** —— 主入口：提供商无关，自动处理跨所有后端的批处理。
- **TextEmbedder** —— 面向文本，带自动批处理和进度跟踪。默认方法是 FastEmbed。
- **GraphEmbeddingManager** —— 面向图数据库的节点和边向量嵌入：Neo4j、NetworkX、FalkorDB。
- **VectorEmbeddingManager** —— 为 FAISS、Weaviate、Qdrant 和 Milvus 准备、归一化和格式化向量嵌入。
- **提供商存储** —— `OpenAIStore`、`BGEStore`、`FastEmbedStore` 以及 `ProviderStoreFactory`。
- **池化策略** —— Mean、Max、CLS、Attention 和 Hierarchical：控制从 token 到向量的聚合方式。

<a id="provider-setup"></a>
## 提供商设置

<Tabs>
  <Tab title="FastEmbed（默认）">
    ONNX 加速的本地向量嵌入。无需 GPU，无需 API 密钥。最佳入门之选。

    ```bash
    pip install "semantica[fastembed]"
    ```

    ```python
    from semantica.embeddings import EmbeddingGenerator

    # FastEmbed 是默认值：无需配置
    generator = EmbeddingGenerator()
    embedding = generator.generate_embeddings("Text about AI")
    ```

    <Check>
      默认模型是 `BAAI/bge-small-en-v1.5`。零成本，零 GPU，可在任何机器上运行。
    </Check>

    <Warning>
      **FastEmbed 会忽略 `device` 参数。** FastEmbed 使用 ONNX Runtime 并自行管理执行提供商：传入 `device="cuda"` 不起作用。如需显式控制 GPU，请改用 `method="sentence_transformers"`。
    </Warning>
  </Tab>
  <Tab title="Sentence-Transformers">
    通过 HuggingFace 提供广泛的模型选择。本地运行，无需 API 密钥。

    ```bash
    pip install semantica  # 已包含 sentence-transformers
    ```

    ```python
    from semantica.embeddings import EmbeddingGenerator

    generator = EmbeddingGenerator(config={
        "text": {
            "method": "sentence_transformers",
            "model_name": "all-MiniLM-L6-v2",
        }
    })
    ```

    常用模型：`all-MiniLM-L6-v2`（快速、小巧）、`all-mpnet-base-v2`（均衡）、`BAAI/bge-large-en-v1.5`（高精度）。

    <Warning>
      **序列长度限制。** 多数 sentence-transformers 模型有 512-token 上限。超出此长度的文本会被静默截断。对于长文档，请使用 `TextSplitter(method="hierarchical")` + `HierarchicalPooling`。
    </Warning>
  </Tab>
  <Tab title="BGE">
    通过 sentence-transformers 使用 BAAI/bge 模型。业界领先的检索性能，本地运行。

    ```bash
    pip install semantica
    ```

    ```python
    from semantica.embeddings import BGEStore, EmbeddingGenerator

    store     = BGEStore(model="BAAI/bge-large-en-v1.5")
    embedding = store.embed("Text about AI")

    # 或者在已有的 EmbeddingGenerator 上切换模型
    generator = EmbeddingGenerator()
    generator.set_text_model("sentence_transformers", "BAAI/bge-large-en-v1.5")
    ```
  </Tab>
  <Tab title="OpenAI">
    通过 OpenAI API 的云端向量嵌入。质量最高，需要 API 密钥。

    ```bash
    pip install "semantica[llm-openai]"
    export OPENAI_API_KEY="sk-..."
    ```

    ```python
    import os
    from semantica.embeddings import OpenAIStore

    store = OpenAIStore(
        api_key=os.getenv("OPENAI_API_KEY"),
        model="text-embedding-3-small",   # 或 text-embedding-3-large
    )
    embedding = store.embed("Text about AI")
    ```

    | 模型 | 维度 | 最适用途 |
    | :---- | :--------- | :-------- |
    | `text-embedding-3-small` | 1536 | 成本高效的检索 |
    | `text-embedding-3-large` | 3072 | 最高精度的工作负载 |
  </Tab>
</Tabs>

检查你的环境中安装了哪些提供商：

```python
from semantica.embeddings import check_available_providers

providers = check_available_providers()
# → {"sentence_transformers": True, "fastembed": True, "openai": False}
```

<a id="getting-started"></a>
## 入门

`EmbeddingGenerator` 是通向向量嵌入的最快路径：默认方法是 FastEmbed（ONNX，无需 GPU）：

```python
from semantica.embeddings import EmbeddingGenerator

# 默认：FastEmbed 与 BAAI/bge-small-en-v1.5
generator = EmbeddingGenerator()

# 嵌入单段文本
embedding = generator.generate_embeddings("Text about AI")

# 嵌入一个批次
embeddings = generator.generate_embeddings(["Text about AI", "Machine learning concepts"])

# 比较两个向量嵌入（余弦相似度：0.0 到 1.0）
score = generator.compare_embeddings(embeddings[0], embeddings[1], method="cosine")
print(f"Similarity: {score:.3f}")
```

<Tip>
  **在索引和查询时始终使用同一个模型。** 来自不同模型的向量不可比较：它们位于不同的向量空间中。切换模型需要对整个语料库重新嵌入。
</Tip>

在构造之后切换提供商：

```python
# 切换到一个 sentence-transformers 模型
generator.set_text_model("sentence_transformers", "all-MiniLM-L6-v2")

# 切换到 BGE large
generator.set_text_model("sentence_transformers", "BAAI/bge-large-en-v1.5")
```

<a id="quick-start"></a>
## 快速上手

<Steps>
  <Step title="安装并初始化一个提供商">
    ```python
    from semantica.embeddings import EmbeddingGenerator

    # 默认：FastEmbed，免费，本地运行且无需 GPU
    generator = EmbeddingGenerator()

    # 改用 sentence-transformers
    generator = EmbeddingGenerator(config={"text": {"method": "sentence_transformers", "model_name": "all-MiniLM-L6-v2"}})
    ```
  </Step>
  <Step title="生成向量嵌入">
    ```python
    # 单段文本 → 1D 数组
    embedding = generator.generate_embeddings("Text about AI")

    # 批次 → 2D 数组 (n_texts, dim)
    embeddings = generator.generate_embeddings(["Text about AI", "Machine learning concepts"])
    ```
  </Step>
  <Step title="计算相似度">
    ```python
    # 余弦相似度：0.0（无关）到 1.0（含义相同）
    score = generator.compare_embeddings(embeddings[0], embeddings[1], method="cosine")
    print(f"Similarity: {score:.3f}")
    ```
  </Step>
  <Step title="为向量数据库做准备">
    ```python
    from semantica.embeddings import VectorEmbeddingManager
    import numpy as np

    manager = VectorEmbeddingManager()

    embeddings = np.array([...], dtype=np.float32)
    metadata   = [{"text": "doc 1"}, {"text": "doc 2"}]

    result = manager.prepare_for_vector_db(embeddings, metadata=metadata, backend="faiss")
    # result["vectors"]  → 归一化的 float32 数组
    # result["ids"]      → ["vec_0", "vec_1", ...]
    # result["metadata"] → 格式化后的元数据列表
    ```
  </Step>
</Steps>

<a id="supported-models"></a>
## 支持的模型

| 提供商 | 模型 | 维度 | 速度 | 最适用途 |
| :-------- | :----- | :--------- | :----- | :-------- |
| `fastembed` | `BAAI/bge-small-en-v1.5` | 384 | 非常快 | **默认**：针对 CPU 优化，不**要求** GPU |
| `sentence_transformers` | `all-MiniLM-L6-v2` | 384 | 快 | 速度与质量的良好平衡 |
| `sentence_transformers` | `all-mpnet-base-v2` | 768 | 中 | 更高的检索质量 |
| `sentence_transformers` | `BAAI/bge-large-en-v1.5` | 1024 | 中 | 业界领先的检索精度 |
| `openai` | `text-embedding-3-small` | 1536 | API | 高性价比的 OpenAI 向量嵌入 |
| `openai` | `text-embedding-3-large` | 3072 | API | 通过 OpenAI API 获得最高质量 |

<a id="embeddinggenerator"></a>
## EmbeddingGenerator

<Tabs>
  <Tab title="FastEmbed（默认）">
    ```python
    from semantica.embeddings import EmbeddingGenerator

    # 默认：FastEmbed 与 BAAI/bge-small-en-v1.5
    generator = EmbeddingGenerator()
    embeddings = generator.generate_embeddings(texts)
    similarity = generator.compare_embeddings(embeddings[0], embeddings[1])
    ```

    **最适用于：** 仅 CPU 的生产环境，无 GPU 下的最低延迟。默认值：开箱即用。
  </Tab>
  <Tab title="Sentence-Transformers">
    ```python
    from semantica.embeddings import EmbeddingGenerator

    generator = EmbeddingGenerator()
    generator.set_text_model("sentence_transformers", "all-MiniLM-L6-v2")
    embeddings = generator.generate_embeddings(texts)
    ```

    **最适用于：** GPU 可用时需要更高质量检索的场景，或需要微调模型时。
  </Tab>
  <Tab title="OpenAI">
    ```python
    from semantica.embeddings import OpenAIStore
    import os

    store     = OpenAIStore(api_key=os.getenv("OPENAI_API_KEY"), model="text-embedding-3-small")
    embedding = store.embed("Hello world")
    ```

    **最适用于：** 最高质量（`text-embedding-3-large`），或与既有 OpenAI 流水线对齐。
  </Tab>
  <Tab title="GPU 加速">
    ```python
    from semantica.embeddings import EmbeddingGenerator

    # 通过 sentence-transformers 使用 CUDA
    generator = EmbeddingGenerator(config={"text": {"method": "sentence_transformers", "device": "cuda"}})

    # Apple Silicon (M1/M2/M3)
    generator = EmbeddingGenerator(config={"text": {"method": "sentence_transformers", "device": "mps"}})
    ```

    GPU 仅适用于 sentence-transformers。FastEmbed 使用 ONNX，不使用 `device`。
  </Tab>
</Tabs>

<a id="constructor-parameters"></a>
### 构造函数参数

| 参数 | 类型 | 默认值 | 描述 |
| :--------- | :---- | :------- | :----------- |
| `config` | `dict` | `None` | 配置字典；`config["text"]` 会传给 `TextEmbedder` |
| `**kwargs` | | | 合并进 `config` 的额外键/值配置 |

使用 `generator.set_text_model(method, model_name)` 可在构造之后切换向量嵌入模型。

<a id="textembedder"></a>
## TextEmbedder

带批处理的直接文本向量嵌入：

```python
from semantica.embeddings import TextEmbedder

# 默认：FastEmbed 与 BAAI/bge-small-en-v1.5
embedder = TextEmbedder()

# 单段文本 → 1D 数组
embedding = embedder.embed_text("A knowledge graph connects entities with typed relationships.")

# 批次 → 2D 数组 (n_texts, dim)
embeddings = embedder.embed_batch(["First text", "Second text", "Third text"])

# 逐句向量嵌入
sentence_embeddings = embedder.embed_sentences("First sentence. Second sentence.")

# 获取向量嵌入维度
dim = embedder.get_embedding_dimension()
```

<a id="textembedder-constructor-parameters"></a>
### TextEmbedder 构造函数参数

| 参数 | 类型 | 默认值 | 描述 |
| :--------- | :---- | :------- | :----------- |
| `model_name` | `str` | `"BAAI/bge-small-en-v1.5"` | 要加载的模型名称 |
| `method` | `str` | `"fastembed"` | 向量嵌入方法：`"fastembed"` 或 `"sentence_transformers"` |
| `device` | `str` | `"cpu"` | sentence-transformers 使用的设备：`"cpu"`、`"cuda"`、`"mps"`。FastEmbed 会忽略此参数。 |
| `normalize` | `bool` | `True` | 对输出向量进行 L2 归一化 |

**关键行为：**
- 如果 FastEmbed 或 sentence-transformers 不可用，会回退到 128 维的基于哈希的向量嵌入。哈希向量嵌入是确定性的但不具备语义：请勿在生产中使用。
- 大批量会由底层库在内部进行分块，以避免 OOM。

<Warning>
  **维度不匹配。** 你传给向量库的维度必须与向量嵌入模型的输出完全一致。`BAAI/bge-small-en-v1.5` → 384，`all-MiniLM-L6-v2` → 384，`all-mpnet-base-v2` → 768，`BAAI/bge-large-en-v1.5` → 1024。在创建存储之前用 `embedder.get_embedding_dimension()` 检查。
</Warning>

<Tip>
  **回退向量嵌入不具备语义。** 如果 FastEmbed 和 sentence-transformers 都未能成功加载，TextEmbedder 会静默回退到 128 维的 SHA-256 哈希向量嵌入。这些向量是确定性的的但不携带任何语义。请检查 `embedder.get_method()`：如果返回 `"fallback"`，请安装你期望的提供商。
</Tip>

<a id="provider-stores"></a>
## 提供商存储

当你需要对单个后端进行精细控制时，可直接使用提供商存储：

```python
from semantica.embeddings import (
    OpenAIStore, BGEStore, FastEmbedStore,
    ProviderStoreFactory,
)
import os

# OpenAI
store     = OpenAIStore(api_key=os.getenv("OPENAI_API_KEY"), model="text-embedding-3-small")
embedding = store.embed("Hello world")

# BGE（Sentence-Transformers 封装）：传入 model_name= 而非 model=
store     = BGEStore(model_name="BAAI/bge-large-en-v1.5")
embedding = store.embed("Hello world")

# FastEmbed：ONNX 运行时，不要求 CUDA
store     = FastEmbedStore(model_name="BAAI/bge-small-en-v1.5")
embedding = store.embed("Hello world")
# FastEmbedStore 还有一个高效的批处理方法
embeddings = store.embed_batch(["text1", "text2", "text3"])

# 从名称字符串自动选择：适用于配置驱动的流水线
# 支持的提供商："openai"、"bge"、"fastembed"
store = ProviderStoreFactory.create(provider="bge", model_name="BAAI/bge-large-en-v1.5")
```

<Note>
  `LlamaStore` 存在于该模块中，但只是一个占位符：它不会连接到 Ollama，并在 embed 时始终抛出 `ProcessingError`。请勿在生产中使用。
</Note>

<Warning>
  **LlamaStore 不可用。** `LlamaStore` 存在于该模块中，但不会连接到 Ollama。它在 embed 时始终抛出 `ProcessingError`。如需本地基于 ONNX 的向量嵌入，请改用 `FastEmbedStore`；如需基于 sentence-transformers 的本地向量嵌入，请改用 `BGEStore`。
</Warning>

<a id="pooling-strategies"></a>
## 池化策略

池化将一组向量嵌入聚合为单个向量：当你有多块分块向量嵌入需要合并时非常有用：

<Tabs>
  <Tab title="MeanPooling（默认）">
    ```python
    from semantica.embeddings import MeanPooling

    pooler = MeanPooling()
    pooled = pooler.pool(token_embeddings)   # 形状：(hidden_dim,)
    ```

    **最适用于：** 检索、语义搜索和聚类：平均化所有贡献。
  </Tab>
  <Tab title="MaxPooling">
    ```python
    from semantica.embeddings import MaxPooling

    pooler = MaxPooling()
    pooled = pooler.pool(token_embeddings)
    ```

    **最适用于：** 捕获任意特征的存在：取每个维度上的最大激活值。
  </Tab>
  <Tab title="CLSPooling">
    ```python
    from semantica.embeddings import CLSPooling

    pooler = CLSPooling()
    pooled = pooler.pool(token_embeddings)
    ```

    **最适用于：** 分类式任务；显式使用 CLS 池化训练的模型（BERT）。
  </Tab>
  <Tab title="HierarchicalPooling">
    ```python
    from semantica.embeddings import HierarchicalPooling

    pooler = HierarchicalPooling()
    # chunk_size 在 pool 时传入，而非构造时
    pooled = pooler.pool(token_embeddings, chunk_size=10)
    ```

    **最适用于：** 长文档：分块级的均值池化，然后跨分块进行全局均值池化。
  </Tab>
  <Tab title="策略对比">

    | 策略 | 适用场景 |
    | :-------- | :----------- |
    | `mean` | 检索、语义搜索和聚类的默认选择 |
    | `max` | 当你想捕获任意特征的存在，而非平均存在时 |
    | `cls` | 分类式任务；显式使用 CLS 池化训练的模型（BERT） |
    | `attention` | 当 token 重要性差异显著时；更慢但更准确 |
    | `hierarchical` | 包含许多分块的长文档；先分块级再全局池化 |

    ```python
    from semantica.embeddings import PoolingStrategyFactory

    pooler = PoolingStrategyFactory.create(strategy="mean")
    ```

  </Tab>
</Tabs>

<a id="graphembeddingmanager"></a>
## GraphEmbeddingManager

为存储到图数据库而对图节点和边进行向量嵌入：

```python
from semantica.embeddings import GraphEmbeddingManager

manager = GraphEmbeddingManager()

entities = [
    {"id": "e1", "text": "Apple Inc.", "type": "Organization"},
    {"id": "e2", "text": "Tim Cook",   "type": "Person"},
]
relationships = [
    {"source": "e2", "target": "e1", "type": "CEO_OF"}
]

# 嵌入实体 → {id: np.ndarray} 的字典
node_embeddings = manager.embed_entities(entities)

# 嵌入关系 → {id: np.ndarray} 的字典
edge_embeddings = manager.embed_relationships(relationships)

# 或者为图数据库后端一次性准备好一切
result = manager.prepare_for_graph_db(entities, relationships, backend="neo4j")
# result["node_embeddings"] → {id: np.ndarray}
# result["edge_embeddings"] → {id: np.ndarray}
# result["nodes"]           → 添加了 "embedding" 字段的实体
# result["edges"]           → 添加了 "embedding" 字段的关系
```

**支持的后端：** `"neo4j"`、`"networkx"`、`"falkordb"`

<a id="vectorembeddingmanager"></a>
## VectorEmbeddingManager

为向量数据库存储准备并校验向量嵌入：

```python
from semantica.embeddings import VectorEmbeddingManager
import numpy as np

manager    = VectorEmbeddingManager()
embeddings = np.random.rand(5, 384).astype(np.float32)
metadata   = [{"text": f"doc_{i}", "category": "science"} for i in range(5)]

# 为 FAISS 准备
result = manager.prepare_for_vector_db(embeddings, metadata=metadata, backend="faiss")
# result["vectors"]  → L2 归一化的 float32 数组
# result["ids"]      → ["vec_0", "vec_1", ...]
# result["metadata"] → 格式化后的元数据列表

# 在插入前校验维度
is_valid = manager.validate_dimensions(embeddings, backend="milvus")

# 一次性准备多个批次
combined = manager.batch_prepare([embeddings_a, embeddings_b], backend="qdrant")
```

**支持的后端：** `"faiss"`、`"weaviate"`、`"qdrant"`、`"milvus"`

<a id="common-workflows"></a>
## 常见工作流

<Tabs>
  <Tab title="批量文本向量嵌入">
    ```python
    from semantica.embeddings import TextEmbedder

    embedder = TextEmbedder()   # 默认：FastEmbed

    texts = [
        "Apple Inc. was founded by Steve Jobs.",
        "Microsoft was co-founded by Bill Gates.",
        "Amazon was started by Jeff Bezos.",
    ]

    # 一次性完成：比对每条逐个调用 embed_text() 更高效
    embeddings = embedder.embed_batch(texts)
    print(f"Shape: {embeddings.shape}")   # (3, 384)
    ```
  </Tab>
  <Tab title="提供商对比">
    ```python
    from semantica.embeddings import check_available_providers, EmbeddingGenerator

    # 检查已安装的提供商
    available = check_available_providers()
    # → {"sentence_transformers": True, "fastembed": True, "openai": False}

    # 使用可用的最快提供商
    generator = EmbeddingGenerator()
    if available["fastembed"]:
        generator.set_text_model("fastembed", "BAAI/bge-small-en-v1.5")
    elif available["sentence_transformers"]:
        generator.set_text_model("sentence_transformers", "all-MiniLM-L6-v2")

    embeddings = generator.generate_embeddings(texts)
    ```
  </Tab>
  <Tab title="图节点向量嵌入">
    ```python
    from semantica.embeddings import GraphEmbeddingManager

    manager  = GraphEmbeddingManager()
    entities = [{"id": "n1", "text": "Python"}, {"id": "n2", "text": "Django"}]

    node_embeddings = manager.embed_entities(entities)
    # {"n1": array([...]), "n2": array([...])}
    ```
  </Tab>
  <Tab title="相似度搜索">
    ```python
    from semantica.embeddings import EmbeddingGenerator, calculate_similarity
    import numpy as np

    generator = EmbeddingGenerator()
    query     = generator.generate_embeddings("knowledge graph databases")
    corpus    = generator.generate_embeddings([
        "graph databases store relationships",
        "relational databases use tables",
        "knowledge graphs model entity relationships",
    ])

    scores = [calculate_similarity(query, doc, method="cosine") for doc in corpus]
    ranked = sorted(zip(scores, range(len(scores))), reverse=True)

    for score, idx in ranked:
        print(f"{score:.3f}  {['graph databases store...', 'relational databases...', 'knowledge graphs...'][idx]}")
    ```
  </Tab>
</Tabs>

<a id="similarity-computation"></a>
## 相似度计算

```python
from semantica.embeddings import calculate_similarity

# 余弦相似度：只比较方向，不比较幅度；文本场景最常用
score = calculate_similarity(embedding_a, embedding_b, method="cosine")
# → 0.0（正交 / 无关）到 1.0（方向一致）

# 将欧氏距离转换为相似度
score = calculate_similarity(embedding_a, embedding_b, method="euclidean")
```

<a id="convenience-functions"></a>
## 便捷函数

```python
from semantica.embeddings import (
    embed_text, generate_embeddings, calculate_similarity,
    pool_embeddings, check_available_providers,
)

# 单段文本：最快路径
emb = embed_text("Hello world", method="sentence_transformers")

# 批次
embs = generate_embeddings(["text1", "text2"], method="default")

# 将多个向量嵌入池化为一个
pooled = pool_embeddings(embs, method="mean")

# 检查已安装的提供商
providers = check_available_providers()
# → {"sentence_transformers": True, "fastembed": True, "openai": False}
```

- [向量库](vector_store.zh-CN.md) —— 存储并搜索生成的向量嵌入。
- [Split](split.zh-CN.md) —— 在向量嵌入前对文本分块以获得更好的检索质量。
- [KG 模块](kg.zh-CN.md) —— 距离智能使用图向量嵌入来构建语义邻域。
- [去重](deduplication.zh-CN.md) —— 语义去重使用向量嵌入距离进行实体解析。
