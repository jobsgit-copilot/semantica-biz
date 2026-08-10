---
title: "分块模块"
description: "递归、语义、实体感知、关系感知、结构化和滑动窗口的文本分块。"
icon: "scissors"
---

**[English](split.md)** · **简体中文（当前）**

**`semantica.split`** 将文档拆分为**保留语义上下文**的块：

- 六种分块策略：递归、语义、实体感知、关系感知、滑动窗口、结构化
- `SemanticChunker` 使用基于向量嵌入的主题转移检测，仅在内容变化时拆分
- `EntityAwareChunker` 确保实体提及在块边界处保持完整
- `RelationAwareChunker` 将主语-谓语-宾语三元组保留在单个块内
- 分块质量直接决定下游向量嵌入精度和实体抽取质量


<a id="why-chunking-matters"></a>
## 为什么分块很重要

大多数 LLM 和向量嵌入模型有固定的上下文窗口。超过该窗口的文档必须拆分。但朴素的拆分（每 500 个字符，不管结构）会破坏语义上下文：

- 像 "Apple Inc." 这样的实体提及被拆分到两个块中，在两个块中都丢失了上下文
- 像 "Steve Jobs founded Apple" 这样的关系三元组在 "Steve Jobs" 处被拆分，留下悬空的主语
- 对混合了两个不相关主题的块进行向量嵌入会产生一个与两者都不匹配的中心向量

Semantica 的分块方法旨在避免这些失败模式。

<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `TextSplitter` | 统一入口：切换 `method=` 无需更改下游代码 |
| `Chunk` | `{text, start_index, end_index, metadata, id}` |
| `SemanticChunker` | 基于向量嵌入的主题转移检测：仅在内容实际变化时拆分 |
| `StructuralChunker` | 使用结构化文本分析的标题/章节拆分 |
| `EntityAwareChunker` | 防止命名实体提及被拆分到块边界两侧 |
| `RelationAwareChunker` | 将主语-谓语-宾语三元组保持在单个块内完整 |
| `HierarchicalChunker` | 产生父/子块关系的多级分块 |

**`TextSplitter` 可用的 `method=` 值：**

| 方法 | 最适合 |
| :--- | :--- |
| `recursive` | 通用文本：依次按段落、句子、单词拆分 |
| `sentence` | 对话文本、问答 |
| `paragraph` | 段落完整性重要的长文本 |
| `token` | LLM 上下文窗口强制限制 |
| `semantic_transformer` | 有主题转移的长文档 |
| `entity_aware` | 知识图谱抽取流水线 |
| `relation_aware` | 三元组完整性重要的知识图谱流水线 |
| `structural` | 有标题/段落结构的文本 |
| `sliding_window` | 用于双编码器检索的密集重叠 |

<a id="what-you-get"></a>
## 你将获得

- **TextSplitter** — 11 种分块策略的统一接口：切换方法无需更改下游代码。
- **语义分块** — 基于向量嵌入的主题转移检测：仅在主题实际变化时拆分。
- **实体感知分块** — 实体跨度永远不会跨越块边界：由边界调整保证。
- **关系感知分块** — 主语-谓语-宾语三元组保留在单个块内，适用于知识图谱流水线。
- **Chunk 对象** — 包含文本、字符偏移量、可选 id 和方法特定元数据的输出数据类。

<a id="quick-start"></a>
## 快速上手

<Steps>
  <Step title="选择分块方法">
    ```python
    from semantica.split import TextSplitter

    splitter = TextSplitter(
        method="recursive",   # 参见分块方法表
        chunk_size=1000,
        chunk_overlap=200,
    )
    ```
  </Step>
  <Step title="拆分原始文本">
    ```python
    chunks = splitter.split(text)

    for chunk in chunks:
        print(f"  Start: {chunk.start_index}, End: {chunk.end_index}")
        print(f"  Method: {chunk.metadata.get('method')}")
        print(f"  Preview: {chunk.text[:80]}...")
    ```
  </Step>
  <Step title="或拆分文档对象">
    ```python
    # split_documents() 接受任何具有 .text 属性的对象，
    # 或纯字符串：不需要特定的文档类。
    class Doc:
        def __init__(self, text, metadata=None):
            self.text = text
            self.metadata = metadata or {}

    doc = Doc(text="Annual report content...", metadata={"source": "annual_report.pdf"})

    splitter = TextSplitter(method="structural")
    chunks   = splitter.split_documents([doc])

    for chunk in chunks:
        print(f"  {chunk.text[:80]}...")
    ```
  </Step>
  <Step title="批量拆分文档列表">
    ```python
    # split_documents() 返回所有输入的扁平 List[Chunk]
    all_chunks = splitter.split_documents(docs)

    for chunk in all_chunks:
        print(chunk.text[:80])
    ```
  </Step>
</Steps>

<a id="splitting-methods"></a>
## 分块方法

| 方法 | 如何拆分 | 最适合 |
| :------ | :------------- | :-------- |
| `recursive` | 段落 → 句子 → 单词（级联回退） | 通用默认选择 |
| `semantic_transformer` | 对句子进行向量嵌入，在余弦相似度下降处拆分 | RAG：主题连贯性重要时 |
| `entity_aware` | 调整边界使实体跨度不被截断 | NER 流水线 |
| `relation_aware` | 将主语-谓语-宾语三元组保持在单个块内 | 知识图谱构建 |
| `sentence` | 句子边界检测（正则表达式、NLTK、spaCy） | 短文档、问答 |
| `paragraph` | 段落边界拆分 | 长篇文章、报告 |
| `token` | 通过 tiktoken 或 transformers 计 token；硬截断 | LLM 上下文窗口准备 |
| `word` | 计单词数带重叠 | 简单的 token 近似拆分 |
| `character` | 固定字符数带重叠；最快，无 NLP | 简单批处理任务 |
| `sliding_window` | 固定大小窗口按步进推进；可配置重叠 | 密集检索（ColBERT、DPR） |
| `structural` | 标题/段落结构检测 | 有显式标题层级的文本 |
| `embedding_semantic` | 向量嵌入相似度边界（`semantic_transformer` 的别名） | 基于向量嵌入连贯性的 RAG |
| `hierarchical` | 多级章节 → 段落 → 句子分块 | 多粒度检索 |

<a id="choosing-a-strategy"></a>
## 选择策略

在选择方法之前使用此决策树：

- **构建知识图谱？** → `relation_aware`（保持三元组完整），然后对纯 NER 用 `entity_aware`
- **检索质量最重要的 RAG 系统？** → `semantic_transformer`
- **用于双编码器检索（ColBERT、DPR）的密集重叠？** → `sliding_window`
- **为固定窗口 LLM 准备提示？** → `token`
- **有标题的结构化文本？** → `structural`
- **段落级连贯性？** → `paragraph` 或 `sentence`
- **无 NLP 开销的快速拆分？** → `recursive` 或 `character`

<a id="textsplitter-constructor"></a>
## TextSplitter 构造函数

```python
from semantica.split import TextSplitter

splitter = TextSplitter(
    method="semantic_transformer",   # 分块策略：参见分块方法表
    chunk_size=1000,                 # 以字符为单位的目标大小
    chunk_overlap=200,               # 相邻块之间的字符重叠
    similarity_threshold=0.7,        # 余弦相似度阈值（仅 semantic_transformer）
    model="all-MiniLM-L6-v2",        # sentence-transformers 模型名称（仅 semantic_transformer）
    ner_method="ml",                 # NER 方法（仅 entity_aware）
    relation_method="ml",            # 关系抽取方法（仅 relation_aware）
)
```

| 参数 | 类型 | 默认值 | 描述 |
| :--------- | :---- | :------- | :----------- |
| `method` | `str \| list[str]` | `"recursive"` | 分块策略，或作为回退链的方法列表 |
| `chunk_size` | `int` | `1000` | 以**字符**为单位的目标大小（不是 token：如果之前使用基于 token 的大小，乘以约 4 以近似相同的边界） |
| `chunk_overlap` | `int` | `200` | 相邻块之间的字符重叠 |
| `similarity_threshold` | `float` | `0.7` | `semantic_transformer` 的余弦相似度阈值：更低 = 更多拆分 |
| `model` | `str` | `"all-MiniLM-L6-v2"` | `semantic_transformer` 的 sentence-transformers 模型名称 |
| `ner_method` | `str` | `"ml"` | `entity_aware` 的 NER 方法：`"pattern"` \| `"regex"` \| `"ml"` \| `"huggingface"` \| `"llm"` |
| `relation_method` | `str` | `"ml"` | `relation_aware` 的关系抽取方法：`"ml"` \| `"llm"` \| `"huggingface"` |
| `tokenizer` | `str` | `"gpt-4"` | `token` 方法的 tiktoken 模型名称：无法识别的名称回退到 `cl100k_base` |

<Warning>
  **`chunk_overlap` 过小。** 没有重叠，跨越块边界的事实将在两个块中都不可见。相对于 `chunk_size` 的 10–20% 重叠是安全的最低值：对于 `chunk_size=1000`，设置 `chunk_overlap=100` 到 `200`。
</Warning>

<a id="splitting-method-details"></a>
## 分块方法详情

<Tabs>
  <Tab title="Recursive（默认）">
    先尝试段落分隔，然后句子边界，然后单词边界：仅在块超过 `chunk_size` 时回退：

    ```python
    splitter = TextSplitter(method="recursive", chunk_size=1000, chunk_overlap=200)
    chunks   = splitter.split(text)
    ```

    **关键行为：**
    - 尽可能保留段落和句子结构
    - 优雅回退：从不产生超过 `chunk_size` 的块
    - 重叠确保跨块边界的上下文连续性
    - 当你不确定使用哪种方法时的良好起点
  </Tab>
  <Tab title="Semantic（语义）">
    使用 sentence-transformers 模型对每个句子进行向量嵌入，然后当连续句子之间的余弦相似度降到 `similarity_threshold` 以下时拆分。每个块覆盖一个连贯的主题：

    ```python
    from semantica.split import TextSplitter

    splitter = TextSplitter(
        method="semantic_transformer",
        model="all-MiniLM-L6-v2",      # 任何 sentence-transformers 模型名称
        similarity_threshold=0.7,       # 0.6 = 更多拆分，0.8 = 更少拆分
        chunk_size=800,
        chunk_overlap=0,                # 不需要：块已经是连贯的
    )
    chunks = splitter.split(text)
    ```

    **关键行为：**
    - 需要 `sentence-transformers` 包：默认使用 `all-MiniLM-L6-v2`，可通过 `model=` 配置
    - 产生可变长度的块：有些主题短，有些长
    - 如果未安装 `sentence-transformers` 则回退到句子拆分
    - 由于向量嵌入计算，比 `recursive` 慢；对重复拆分缓存嵌入

    <Tip>
      **语义拆分需要足够的句子。** `semantic_transformer` 需要多个句子来检测主题转移。在少于约 300 个词的文档上，它的行为类似于 `sentence` 拆分：请改用 `recursive`。
    </Tip>
  </Tab>
  <Tab title="Entity-Aware（实体感知）">
    内部运行 NER，然后调整块边界使没有实体提及被拆分到两个块：

    ```python
    from semantica.split import TextSplitter
    import os

    # ner_method 被传递给内部的 NERExtractor。
    # 使用 "llm" 获得最高精度，"ml"（默认）获得速度。
    splitter = TextSplitter(
        method="entity_aware",
        chunk_size=512,
        chunk_overlap=50,
        ner_method="ml",   # "pattern" | "regex" | "ml" | "huggingface" | "llm"
    )
    chunks = splitter.split(text)

    for chunk in chunks:
        print(f"  entities in chunk: {chunk.metadata.get('entity_count', 0)}")
        print(f"  preview: {chunk.text[:80]}...")
    ```

    **关键行为：**
    - NER 内部运行：实体抽取在拆分器内部自动进行
    - 每个块的实体对象可在 `chunk.metadata["entities"]` 中获取
    - 块大小与 `chunk_size` 略有不同：边界调整 ≤ 一个句子
    - 适用于所有实体类型：PERSON、ORGANIZATION、LOCATION、DATE、自定义类型
  </Tab>
  <Tab title="Relation-Aware（关系感知）">
    将主语-谓语-宾语三元组保持在同一个块内：对知识图谱流水线至关重要：

    ```python
    from semantica.split import TextSplitter

    # relation_method 被传递给内部的 RelationExtractor。
    # 使用 "llm" 获得最高精度，"ml"（默认）获得速度。
    splitter = TextSplitter(
        method="relation_aware",
        chunk_size=512,
        relation_method="ml",   # "ml" | "llm" | "huggingface"
    )
    chunks = splitter.split(text)

    for chunk in chunks:
        print(f"  relations in chunk: {chunk.metadata.get('relation_count', 0)}")
        for rel in chunk.metadata.get("relationships", []):
            print(f"  {rel}")
    ```

    **关键行为：**
    - 关系抽取内部运行：无需预计算的实体或三元组
    - 每个块的关系对象可在 `chunk.metadata["relationships"]` 中获取
    - 隐含实体感知行为：三元组中的两个实体也保持完整
    - 最适合用作 `Parse → Split → Extract → Build KG` 流水线中的拆分步骤
  </Tab>
  <Tab title="Structural（结构化）">
    基于标题和段落边界的结构分析拆分文本。每个标题或段落组成为块边界：

    ```python
    from semantica.split import TextSplitter

    splitter = TextSplitter(method="structural")
    chunks   = splitter.split(text)

    for chunk in chunks:
        print(f"  {chunk.text[:80]}...")
        print(f"  start: {chunk.start_index}, end: {chunk.end_index}")
    ```

    **关键行为：**
    - 对纯文本操作：不需要结构化文档格式
    - 尊重标题层级（以 `#` 或全大写标题开头的行）和段落分隔
    - 使用 `max_chunk_size=` 参数代替标准的 `chunk_size=` 来控制最大大小
    - 如果 `StructuralChunker` 不可用则回退到 `recursive`
  </Tab>
</Tabs>

<a id="chunk-schema"></a>
## Chunk 模式

<AccordionGroup>
  <Accordion title="Chunk 数据类">

```python
@dataclass
class Chunk:
    text:        str                    # 块的文本内容
    start_index: int                    # 在源文本中起始的字符偏移
    end_index:   int                    # 在源文本中结束的字符偏移
    metadata:    Dict[str, Any]         # 方法特定的字段：参见下表
    id:          Optional[str] = None   # 可选的块标识符
```

  </Accordion>
  <Accordion title="Chunk 元数据字段">

元数据键因方法而异。仅列出实际由实现设置的键。

| 字段 | 类型 | 设置者 | 描述 |
| :----- | :---- | :------ | :----------- |
| `method` | `str` | 所有方法 | 产生此块的分块方法 |
| `chunk_size` | `int` | 大多数方法 | 此块的字符长度 |
| `sentence_count` | `int` | `sentence`、`semantic_transformer`、spaCy 路径 | 此块中的句子数 |
| `paragraph_count` | `int` | `paragraph` | 此块中的段落数 |
| `word_count` | `int` | `word` | 此块中的单词数 |
| `token_count` | `int` | `token`；spaCy 可用时 `sentence`/`semantic_transformer` | token 计数：不总是存在 |
| `entity_count` | `int` | `entity_aware` | 边界落在此块中的实体数 |
| `entities` | `list` | `entity_aware` | 边界落在此块中的实体对象 |
| `relation_count` | `int` | `relation_aware` | 此块中的关系三元组数 |
| `relationships` | `list` | `relation_aware` | 此块中的关系对象 |
| `element_count` | `int` | `structural` | 分组到此块中的结构元素数 |
| `element_types` | `list[str]` | `structural` | 元素类型：`"heading"`、`"paragraph"`、`"list"` 等 |

  </Accordion>
</AccordionGroup>

<a id="tokenizer-options"></a>
## 分词器选项

`token` 方法接受 `tokenizer=` kwarg，传递给 `tiktoken.encoding_for_model()`。值应为 tiktoken 模型名称。无法识别的名称自动回退到 `cl100k_base`。

| 值 | 使用的编码 |
| :----- | :------------- |
| `"gpt-4"`（默认） | `cl100k_base` |
| `"gpt-3.5-turbo"` | `cl100k_base` |
| `"text-embedding-ada-002"` | `cl100k_base` |
| 任何无法识别的字符串 | 回退到 `cl100k_base` |

如果未安装 `tiktoken`，`token` 方法回退到按空格分隔的单词拆分。

<Warning>
  **错误的分词器。** `token` 方法将 `tokenizer=` 值传递给 `tiktoken.encoding_for_model()`。如果模型名称未被 tiktoken 识别，它会静默回退到 `cl100k_base`。传入有效的 tiktoken 模型名称（例如 `"gpt-4"`、`"gpt-3.5-turbo"`）以获得确定性行为。
</Warning>

<a id="pipeline-integration"></a>
## 流水线集成

`TextSplitter` 可以独立使用或与其他 Semantica 模块手动组合。下面的示例展示了一个顺序模式：解析文件、拆分文本，然后从每个块中抽取实体：

```python
from semantica.parse import DocumentParser
from semantica.split import TextSplitter
from semantica.semantic_extract import NERExtractor

# 解析
parser = DocumentParser()
parsed = parser.parse("data/report.pdf")   # 返回带 "full_text" 键的字典

# 拆分
splitter = TextSplitter(method="semantic_transformer", chunk_size=512)
chunks   = splitter.split(parsed["full_text"])

# 从每个块中抽取
ner = NERExtractor(method="ml")

for chunk in chunks:
    entities = ner.extract(chunk.text)
    print(f"  {len(entities)} entities in chunk starting at {chunk.start_index}")
```

有关完整的流水线编排 API，请参见[流水线参考](pipeline.zh-CN.md)。

- [解析](parse.zh-CN.md) — 在分块前解析文档：产生章节和元数据。
- [向量嵌入](embeddings.zh-CN.md) — 对块进行向量嵌入以用于向量搜索和语义分块。
- [语义抽取](semantic_extract.zh-CN.md) — 从各个块中抽取实体和关系。
- [流水线](pipeline.zh-CN.md) — 将分块集成为命名的流水线步骤。
