---
title: "解析模块"
description: "文档解析和文本提取：用于标准格式的 DocumentParser 和用于复杂布局的 DoclingParser。"
icon: "file-lines"
---

**[English](parse.md)** · **简体中文（当前）**

**`semantica.parse`** 从非结构化文档中提取**结构化文本、布局、表格和元数据**：

- `DocumentParser`：广泛支持多种格式（PDF、DOCX、HTML、JSON、CSV、PPTX、XLSX），无额外依赖
- `DoclingParser`：复杂布局、合并单元格表格、多列 PDF、OCR（`pip install docling`）
- 两者都返回一致的 `dict`，包含 `full_text`、`metadata`、`pages` 和 `tables` 键
- `parse_batch()` 并行处理多个文件，并支持可配置的错误处理


<a id="getting-started"></a>
## 快速上手

<a id="installation"></a>
### 安装

解析模块对于标准格式开箱即用：

```python
from semantica.parse import DocumentParser

parser = DocumentParser()
result = parser.parse("document.pdf")
print(result["full_text"])  # 提取的文本内容
```

如需增强的表格提取和复杂布局支持，请安装 Docling 依赖：

```bash
pip install docling
```

```python
from semantica.parse import DoclingParser

parser = DoclingParser(export_format="markdown")
result = parser.parse("document.pdf", extract_tables=True)
print(result["tables"])  # 增强的表格提取
```

<a id="first-document-parsing"></a>
### 首次文档解析

```python
from semantica.parse import DocumentParser

# 解析任意支持的格式
parser = DocumentParser()
result = parser.parse("annual_report.pdf")

# 访问提取的内容
text = result["full_text"]           # 完整的文档文本
metadata = result["metadata"]        # 文档属性
pages = result.get("pages", [])      # 页面级内容

print(f"Extracted {len(text)} characters from {metadata.get('page_count', 0)} pages")
```

<a id="parser-selection-guide"></a>
## 解析器选择指南

<Tabs>
  <Tab title="DocumentParser：标准">
    零额外依赖。用于清晰的 PDF、Word 文档、HTML 和结构化格式。

    | | |
    | :-- | :-- |
    | **格式** | PDF、DOCX、HTML、TXT、JSON、CSV、PPTX、XLSX |
    | **速度** | 快 |
    | **安装** | 无：包含在基础安装中 |
    | **最适合** | 清晰的文档、广泛的格式支持、生产流水线 |

    ```python
    from semantica.parse import DocumentParser

    parser = DocumentParser()
    result = parser.parse("contract.pdf")

    print(result["full_text"])        # 提取的文本
    print(result["metadata"])         # 标题、作者、页数……
    print(len(result.get("pages", [])))  # 按页拆分
    ```
  </Tab>
  <Tab title="DoclingParser：复杂布局">
    卓越的表格提取、OCR、多列 PDF。需要 `pip install docling`。

    | | |
    | :-- | :-- |
    | **格式** | PDF、DOCX、PPTX、XLSX、HTML、图像 |
    | **速度** | 较慢（深度布局分析） |
    | **安装** | `pip install docling` |
    | **最适合** | 合并单元格表格、扫描文档、多列布局 |

    ```python
    from semantica.parse import DoclingParser

    parser = DoclingParser(export_format="markdown")
    result = parser.parse(
        "financial_report.pdf",
        extract_tables=True,
        extract_text=True,
    )

    for i, table in enumerate(result["tables"]):
        print(f"Table {i+1}: {table['row_count']} rows × {table['col_count']} columns")
        print(f"  Page: {table['page_number']}")
        for row in table["rows"][:3]:
            print(" | ".join(row))
    ```

    <Tip>
      从 `DocumentParser` 开始。仅在你需要更好的表格提取或遇到复杂 PDF 布局时切换到 `DoclingParser`。
    </Tip>
  </Tab>
  <Tab title="批处理">
    并行处理多个文件，每个文件单独隔离错误。

    ```python
    from semantica.parse import DocumentParser

    parser  = DocumentParser()
    results = parser.parse_batch(
        ["doc1.pdf", "doc2.docx", "doc3.html"],
        continue_on_error=True,   # 跳过失败的文件而不是抛出异常
    )

    print(f"Parsed: {results['success_count']}/{results['total']}")

    for item in results["successful"]:
        print(f"{item['file_path']}: {len(item['result']['full_text'])} chars")

    for item in results["failed"]:
        print(f"FAILED: {item['file_path']}: {item['error']}")
    ```

    <Note>
      对于可能存在个别文件损坏或不受支持的生产批处理任务，建议使用 `continue_on_error=True`。
    </Note>
  </Tab>
</Tabs>

<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `DocumentParser` | 自动检测格式：委托给特定格式的解析器（PDF、DOCX、HTML、JSON、CSV 等） |
| `DoclingParser` | 复杂布局、合并单元格表格、多列 PDF 和 OCR（`pip install docling`） |
| `DoclingMetadata` | 来自 Docling 解析的文档元数据 |
| `PDFParser` | PDF 文本和元数据提取 |
| `WebParser` | URL 抓取 + HTML 解析 |
| `EmailParser` | `.eml` / `.msg` 邮件文件，支持附件提取 |
| `CodeParser` | 源代码文件，支持语法感知的代码块检测 |

<a id="documentparser"></a>
## DocumentParser

用于清晰、机器可读文档的标准解析器：

```python
from semantica.parse import DocumentParser

parser = DocumentParser()
result = parser.parse("data/report.pdf")

print(result["full_text"])      # 完整提取的文本
print(result["metadata"])       # 文档属性（标题、作者、页数等）
if "pages" in result:           # 页面级内容（可用时）
    print(f"Pages: {len(result['pages'])}")
```

支持的格式：PDF、DOCX、HTML、TXT、JSON、CSV、PPTX、XLSX。

<a id="doclingparser"></a>
## DoclingParser

使用 Docling 后端的高级解析器：处理 `DocumentParser` 无法处理的布局：

```bash
pip install docling
```

```python
from semantica.parse import DoclingParser

parser = DoclingParser(
    export_format="markdown",      # 导出格式："markdown" | "html" | "json"
    enable_ocr=False               # 为扫描文档启用 OCR
)

result = parser.parse(
    "data/annual_report.pdf",
    extract_tables=True,           # 提取结构化表格
    extract_images=False,          # 提取图像区域
    extract_text=True              # 提取文本内容
)

print(result["full_text"])    # 完整提取的文本
print(result["tables"])       # 结构化表格数据
if "pages" in result:         # 页面级内容
    print(f"Pages: {len(result['pages'])}")
```

在以下场景使用 `DoclingParser`：

- 多列 PDF 布局
- 带有合并单元格或复杂表头的表格
- 带有嵌入图表的 PPTX 幻灯片
- 带有公式的 XLSX 电子表格
- 通过 OCR 处理的扫描文档
- 学术论文和技术报告

<a id="ocr-support"></a>
## OCR 支持

```python
parser = DoclingParser(
    enable_ocr=True,           # 通过 PdfPipelineOptions 启用 OCR
    export_format="markdown"
)

result = parser.parse("data/scanned_contract.pdf")
print(result["full_text"])     # OCR 提取的文本
```

<a id="supported-formats"></a>
## 支持的格式

| 格式 | 扩展名 | 使用的解析器 | 说明 |
| :------ | :--------- | :----------- | :----- |
| PDF | `.pdf` | `PDFParser` / `DoclingParser` | 文本、表格、元数据；Docling 增加 OCR |
| Word | `.docx` | 内置 | 文本、标题、表格、元数据 |
| HTML | `.html`, `.htm` | `HTMLParser` / `WebParser` | `WebParser` 抓取远程 URL |
| Markdown | `.md` | 内置 | 保留标题层次结构 |
| 纯文本 | `.txt` | `TXTParser` | 最少元数据 |
| JSON | `.json` | `JSONParser` | 每行一个对象或数组 |
| CSV / TSV | `.csv`, `.tsv` | `CSVParser` | 自动检测表头 |
| Excel | `.xlsx`, `.xls` | 内置 | 支持工作表选择 |
| PowerPoint | `.pptx` | 内置 | 嵌入图表时使用 `DoclingParser` |
| 邮件 | `.eml`, `.msg` | `EmailParser` | 提取附件 |
| XML | `.xml` | `XMLIngestor` | XXE 安全，可选 XSD 验证 |
| 归档 | `.zip`, `.tar` | `FileIngestor` | 递归提取 |
| 源代码 | `.py`, `.js`, `.java`, ... | `CodeParser` | AST 感知的代码块检测 |

<a id="parser-output-structure"></a>
## 解析器输出结构

两种解析器都返回具有以下结构的字典：

```python
result = {
    "full_text": str,              # 完整提取的文本
    "metadata": dict,              # 文档属性和统计信息
    "pages": List[dict],           # 页面级内容（可用时）
    "tables": List[dict],          # 结构化表格数据（DoclingParser）
    "images": List[dict],          # 图像区域（DoclingParser）
    "total_pages": int,            # 总页数
    "export_format": str           # 用于文本提取的格式（DoclingParser）
}
```

<a id="metadata-structure"></a>
### 元数据结构

```python
metadata = {
    "file_path": str,              # 源文件路径
    "page_count": int,             # 页数
    "format": str,                 # 文件格式（"pdf"、"docx" 等）
    # 额外字段因解析器和文档类型而异
}
```

<a id="documentparser-methods"></a>
## DocumentParser 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `parse(source)` | `dict` | 自动检测格式并提取文本、元数据、表格 |
| `parse_batch(sources)` | `dict` | 并行处理多个来源 |
| `extract_text(path)` | `str` | 仅从文档中提取文本内容 |
| `extract_metadata(path)` | `dict` | 仅从文档中提取元数据 |

<a id="integration-with-fileingestor"></a>
## 与 FileIngestor 集成

最常见的模式：摄取一个目录，然后解析每个来源：

```python
from semantica.ingest import FileIngestor
from semantica.parse import DoclingParser

ingestor = FileIngestor()
parser   = DoclingParser(export_format="markdown")

sources = ingestor.ingest("data/reports/")
for source in sources:
    result = parser.parse(source)
    # 访问提取的内容
    text = result["full_text"]
    tables = result["tables"] 
    metadata = result["metadata"]
```

<Note>
  Docling 是可选依赖。如果未安装 `docling`，`DoclingParser` 会抛出 `ImportError` 并给出安装说明：`pip install docling`。`DocumentParser` 始终可用，无需任何额外依赖。
</Note>

- [摄取](ingest.zh-CN.md) — 在解析之前加载文件。
- [分块](split.zh-CN.md) — 对解析后的文本进行分块以便嵌入和抽取。
- [Docling 集成](../integrations/docling.zh-CN.md) — 完整的 Docling 集成设置指南。
- [语义抽取](semantic_extract.zh-CN.md) — 从解析后的文本中抽取实体和关系。
