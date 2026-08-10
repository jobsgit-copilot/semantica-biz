---
title: "Docling 集成"
description: "原生 Docling 集成，支持高保真 PDF、DOCX 和 PPTX 解析，具备表格抽取和 OCR 功能。"
icon: "file-lines"
---

**[English](docling.md)** · **简体中文（当前）**

> 解析复杂文档：PDF、DOCX、PPTX、HTML：具备高保真表格抽取和内置 OCR。


<a id="overview"></a>
## 概览

Docling 通过 **`DoclingParser`** 集成到 Semantica 的 `parse` 模块。文档流经 Docling 的**版面分析引擎**，然后直接馈入 Semantica 的抽取和知识图谱流水线。

- **多格式** — PDF、DOCX、PPTX、HTML 等。
- **表格抽取** — 高保真表格解析，支持表头检测。
- **OCR 支持** — 针对扫描文档的内置 OCR。
- **Markdown 导出** — 针对 LLM 消费优化的简洁 Markdown 输出。


<a id="installation"></a>
## 安装

```bash
pip install semantica
# Docling 作为可选依赖包含在内
# 或单独安装：
pip install docling
```


<a id="basic-usage"></a>
## 基本用法

```python
from semantica.parse import DoclingParser

parser = DoclingParser(enable_ocr=True)
result = parser.parse("financial_report.pdf")

print(result["full_text"][:200])
print(f"Found {len(result['tables'])} tables")
```


<a id="full-example"></a>
## 完整示例

```python
from semantica.parse import DoclingParser

parser = DoclingParser(
    enable_ocr=True,
    export_format="markdown",
)

result = parser.parse("complex_invoice.pdf")

# 完整文本内容
print(result["full_text"])

# 抽取的表格
for i, table in enumerate(result["tables"]):
    print(f"Table {i+1} headers: {table.get('headers', [])}")
    for row in table.get("rows", [])[:3]:
        print(f"  Row: {row}")

# 元数据
metadata = result["metadata"]
print(f"Title: {metadata.get('title')}")
print(f"Pages: {result.get('total_pages')}")
```


<a id="doclingparser-parameters"></a>
## DoclingParser 参数

| 参数 | 默认值 | 说明 |
| :----------- | :--------- | :------------- |
| `enable_ocr` | `False` | 为扫描页面启用 OCR |
| `export_format` | `"markdown"` | 输出格式：`"markdown"` 或 `"text"` |


<a id="parsed-result-structure"></a>
## 解析结果结构

```python
{
    "full_text":    str,         # 干净的文档文本
    "tables":       List[dict],  # 抽取的表格（表头 + 行）
    "metadata":     dict,        # 标题、作者、创建日期等
    "total_pages":  int,
}
```


<a id="see-also"></a>
## 另请参阅

- [解析模块](../reference/parse.zh-CN.md) — 完整的 DocumentParser 和 DoclingParser 参考。
- [摄取模块](../reference/ingest.zh-CN.md) — 在解析之前加载文档。
- [语义抽取](../reference/semantic_extract.zh-CN.md) — 对解析后的文本进行命名实体识别（NER）和关系抽取。
- [流水线](../reference/pipeline.zh-CN.md) — 在完整流水线中使用 DoclingParser。
