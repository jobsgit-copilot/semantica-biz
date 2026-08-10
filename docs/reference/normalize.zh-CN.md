---
title: "归一化模块"
description: "在抽取运行之前进行文本清洗、实体规范化、日期归一化、数字转换、语言检测和编码修复。"
icon: "broom"
---

**[English](normalize.md)** · **简体中文（当前）**

**`semantica.normalize`** 在抽取和图构建**之前**对原始数据进行标准化处理：

- 文本清洗：Unicode NFC/NFKC、空白字符折叠、智能引号和连字符归一化
- 实体规范化：通过可配置的别名映射进行别名解析和消歧
- 日期归一化：任意格式 → ISO 8601，包括相对日期
- 数字转换：`"$1.2B"` → `1200000000.0`，支持单位和货币处理
- 针对不一致的源数据进行语言检测和编码修复

所有归一化器都提供便捷函数（单行调用）和有状态的类实例（完全控制）。


<a id="why-normalize-before-extraction"></a>
## 为什么在抽取之前进行归一化

非结构化数据本质上是不一致的。如果不进行归一化，同一个现实世界实体会在你的图中表现为几十种变体：

- `"Apple Inc."`、`"Apple Computer Inc."`、`"APPLE INC."`：多个节点，一家公司
- `"Jan 1st, 2020"`、`"01/01/2020"`、`"2020-01-01"`：三种格式，一个日期
- `"$1.2B"`、`"1,200,000,000"`、`"1.2 billion USD"`：三个字符串，一个数字
- `"Hello World"` 与 `"Hello World"`：一个不换行空格会破坏字符串匹配

归一化在任何抽取器、去重器或图构建器看到数据之前，将这些变体合并。

<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `TextNormalizer` | Unicode 形式（NFC/NFKC）、空白字符折叠、智能引号和连字符归一化 |
| `EntityNormalizer` | 使用可配置的别名映射进行别名解析和实体消歧 |
| `DateNormalizer` | 将任意日期字符串格式解析为 ISO 8601；处理相对日期 |
| `NumberNormalizer` | `"$1.2B"` → `1200000000.0`；单位转换；货币解析 |
| `DataCleaner` | 检测并删除重复项、处理缺失值、验证记录 |
| `LanguageDetector` | `detect(text)` → 语言代码 `str`；`detect_with_confidence(text)` → `(code, score)` 元组 |
| `EncodingHandler` | `detect(bytes)` → `(encoding, confidence)` 元组；`convert_to_utf8(bytes)` → `str` |

<a id="getting-started"></a>
## 快速上手

```python
from semantica.normalize import (
    TextNormalizer,
    DateNormalizer,
    NumberNormalizer,
    LanguageDetector,
    EncodingHandler,
)

# 文本：归一化 unicode、折叠空白字符、替换智能引号
normalizer = TextNormalizer()
clean = normalizer.normalize_text("  Hello,  World…  ")
# → "Hello, World..."

# 日期
date_norm = DateNormalizer()
date = date_norm.normalize_date("Jan 1st, 2020")
# → "2020-01-01T00:00:00+00:00"

# 数字
num_norm = NumberNormalizer()
num = num_norm.normalize_number("$1.2B")
# → 1200000000.0

# 语言：返回语言代码字符串
detector = LanguageDetector()
lang = detector.detect("Bonjour le monde")
# → "fr"

# 编码：返回 (encoding_name, confidence) 元组
handler = EncodingHandler()
encoding, confidence = handler.detect(raw_bytes)
utf8_text = handler.convert_to_utf8(raw_bytes)
```

<a id="recommended-processing-order"></a>
## 推荐的处理顺序

<Steps>
  <Step title="EncodingHandler：先修复编码">
    损坏的字节会破坏后续的一切。务必在其他处理之前运行此步骤。

    ```python
    from semantica.normalize import EncodingHandler

    handler = EncodingHandler()
    # detect 返回 (encoding_name, confidence_score)
    encoding, confidence = handler.detect(raw_bytes)
    # convert_to_utf8 返回 str
    utf8_text = handler.convert_to_utf8(raw_bytes)
    ```

    <Warning>
      **务必先运行编码修复。** UTF-8 流中的单个 cp1252 字符会悄无声息地破坏周围文本。请先调用 `handler.convert_to_utf8(raw_bytes)`，再让任何其他归一化器处理数据。
    </Warning>
  </Step>
  <Step title="TextNormalizer：unicode、空白字符、特殊字符">
    ```python
    from semantica.normalize import TextNormalizer

    normalizer = TextNormalizer()
    # normalize_text 接受按调用传入的选项，而不是构造函数参数
    clean_text = normalizer.normalize_text(
        utf8_text,
        unicode_form="NFC",
        case="preserve",
    )
    ```

    <Warning>
      **不要在 NER 之前转为小写。** 在实体抽取之前调用 `normalize_text(text, case="lower")` 会破坏 NER 所依赖的大小写信号。如有需要，仅在抽取之后再应用大小写归一化。
    </Warning>
  </Step>
  <Step title="EntityNormalizer：规范化实体名称">
    ```python
    from semantica.normalize import EntityNormalizer

    # 在 config 中提供别名，以便解析器映射变体
    normalizer = EntityNormalizer(alias_map={
        "apple computer inc.": "Apple Inc.",
        "apple computer, inc.": "Apple Inc.",
    })
    canonical = normalizer.normalize_entity(
        "Apple Computer, Inc.", entity_type="Organization"
    )
    # → "Apple Inc."（如果 alias_map 包含该项；否则返回标题大小写输入）
    ```

    <Warning>
      **`EntityNormalizer` 没有内置的公司后缀展开。** 没有自动将 `"Apple Computer Inc."` 映射为 `"Apple Inc."` 的机制。要规范化公司名称，请提供显式的 `alias_map`（键为小写）：`EntityNormalizer(alias_map={"apple computer inc.": "Apple Inc."})`。
    </Warning>
  </Step>
  <Step title="DateNormalizer 和 NumberNormalizer：解析结构化值">
    ```python
    from semantica.normalize import DateNormalizer, NumberNormalizer

    date_norm = DateNormalizer()
    num_norm  = NumberNormalizer()

    # format 和 timezone 传给 normalize_date()，而不是构造函数
    date = date_norm.normalize_date("Jan 1st, 2020", format="ISO8601", timezone="UTC")
    # → "2020-01-01T00:00:00+00:00"

    num = num_norm.normalize_number("$1.2B")
    # → 1200000000.0
    ```
  </Step>
  <Step title="LanguageDetector：在干净的文本上检测语言">
    ```python
    from semantica.normalize import LanguageDetector

    detector = LanguageDetector()

    # detect() 返回语言代码字符串
    lang = detector.detect("Bonjour le monde")
    # → "fr"

    # detect_with_confidence() 返回 (code, score) 元组
    lang, confidence = detector.detect_with_confidence("Bonjour le monde")
    # → ("fr", 0.98)
    ```
  </Step>
</Steps>

<a id="convenience-functions"></a>
## 便捷函数

最快的路径：一个导入，一次调用：

```python
from semantica.normalize import (
    normalize_text, normalize_entity, normalize_date,
    normalize_number, clean_data, detect_language, handle_encoding,
)

clean  = normalize_text("  Hello,   World  ")
# → "Hello, World"

entity = normalize_entity("John Doe", entity_type="Person")
# → "John Doe"（标题大小写；别名解析需要 alias_map）

date   = normalize_date("Jan 1st, 2020")
# → "2020-01-01T00:00:00+00:00"

num    = normalize_number("$1.2B")
# → 1200000000.0

# detect_language 默认返回语言代码字符串
lang   = detect_language("Bonjour le monde")
# → "fr"

# handle_encoding 使用 operation="detect" 返回 (encoding, confidence) 元组
encoding, confidence = handle_encoding(raw_bytes, operation="detect")

# handle_encoding 使用 operation="convert" 返回 UTF-8 字符串
utf8_text = handle_encoding(raw_bytes, operation="convert")
```

<a id="normalizers"></a>
## 归一化器

<Tabs>
  <Tab title="TextNormalizer">
    `TextNormalizer` 在其构造函数中接受 `config=None, **kwargs`。归一化选项按调用传入 `normalize_text()`：

    ```python
    from semantica.normalize import TextNormalizer

    normalizer = TextNormalizer()

    # normalize_text 选项
    normalized = normalizer.normalize_text(
        raw_text,
        unicode_form="NFC",       # "NFC" | "NFD" | "NFKC" | "NFKD"
        case="preserve",          # "preserve" | "lower" | "upper" | "title"
        normalize_diacritics=False,
        line_break_type="unix",   # "unix" | "windows"
    )

    # HTML 剥离和文本清洗：单独的 clean_text() 方法
    cleaned = normalizer.clean_text(html_text, remove_html=True)

    # 批量归一化
    results = normalizer.process_batch(
        ["  hello  ", "WORLD", "café"],
        unicode_form="NFKC",
        case="lower",
    )

    # normalize() 接受 str 或 List[Dict]（来自 DocumentParser 的已解析文档）
    docs = [{"content": "Hello world"}, {"content": "test text"}]
    normalized_docs = normalizer.normalize(docs)
    ```

    | `normalize_text()` 参数 | 类型 | 默认值 | 描述 |
    | :----------------------------- | :---- | :------- | :----------- |
    | `unicode_form` | `str` | `"NFC"` | Unicode 形式：`"NFC"` / `"NFD"` / `"NFKC"` / `"NFKD"` |
    | `case` | `str` | `"preserve"` | `"preserve"` / `"lower"` / `"upper"` / `"title"` |
    | `normalize_diacritics` | `bool` | `False` | 去除附加符号 |
    | `line_break_type` | `str` | `"unix"` | `"unix"`（`\n`）或 `"windows"`（`\r\n`） |

    **Unicode 形式指南：**

    | 形式 | 适用场景 |
    | :---- | :-------- |
    | `NFC` | 默认：最适合存储和显示 |
    | `NFKC` | 搜索索引：归一化连字、全角字符和分数 |
    | `NFD` | 去除附加符号：将 é 拆分为 e + 组合重音，然后去除重音 |
    | `NFKD` | 与 NFD 相同，但还会分解兼容性字符 |

    **用于精细控制的子归一化器：**

    ```python
    from semantica.normalize import (
        UnicodeNormalizer, WhitespaceNormalizer, SpecialCharacterProcessor
    )

    unicode_norm = UnicodeNormalizer()
    text = unicode_norm.normalize_unicode("café", form="NFC")

    ws_norm = WhitespaceNormalizer()
    text    = ws_norm.normalize_whitespace("Hello\t\t World\n\n")
    # → "Hello  World\n\n"

    processor = SpecialCharacterProcessor()
    text      = processor.normalize_punctuation("‘Hello’")
    # → "'Hello'"
    ```
  </Tab>
  <Tab title="EntityNormalizer">
    `EntityNormalizer` 执行：空白字符清理、可选的别名解析（需要显式 `alias_map`）以及名称格式归一化。

    ```python
    from semantica.normalize import EntityNormalizer

    # 使用别名映射：解析精确匹配（小写键查找）
    normalizer = EntityNormalizer(alias_map={
        "apple computer inc.": "Apple Inc.",
        "ms": "Microsoft",
        "ml":  "Machine Learning",
    })

    normalizer.normalize_entity("Apple Computer Inc.", entity_type="Organization")
    # → "Apple Inc."

    # 不使用别名映射：仅进行空白字符/格式清理
    normalizer2 = EntityNormalizer()
    normalizer2.normalize_entity("apple inc", entity_type="Organization")
    # → "apple inc"（没有内置后缀展开）

    # Person：标题大小写
    normalizer2.normalize_entity("john doe", entity_type="Person")
    # → "John Doe"
    ```

    **关键行为：**
    - 别名映射使用**小写键查找**：以小写形式注册别名
    - `entity_type="Person"` 会激活对名称的 `title()` 大小写处理
    - 没有内置的公司后缀归一化（Inc → Incorporated 等）
     ：如有需要，请手动将这些映射添加到 `alias_map`

    **子归一化器：**

    ```python
    from semantica.normalize import AliasResolver, EntityDisambiguator, NameVariantHandler

    # alias_map 键必须为小写
    resolver = AliasResolver(alias_map={
        "ml":  "Machine Learning",
        "nlp": "Natural Language Processing",
    })
    resolved = resolver.resolve_aliases("ml")
    # → "Machine Learning"；如果不在映射中则为 None

    disambiguator = EntityDisambiguator()
    result = disambiguator.disambiguate(
        "Apple",
        entity_type="Organization",
        context="Steve Jobs founded Apple in Cupertino in 1976",
    )
    # → {"entity_name": "Apple", "entity_type": "Organization", "confidence": 0.8, "candidates": ["Apple"]}

    handler   = NameVariantHandler()
    canonical = handler.normalize_name_format("Dr. JOHN P. SMITH Jr.")
    # → "John P. Smith Jr."（去除前导称谓）
    ```

    <Tip>
      **`AliasResolver` 使用小写键查找。** 即使规范形式为标题大小写，也要以小写键注册别名。解析器在查找之前会将输入转换为小写。
    </Tip>
  </Tab>
  <Tab title="DateNormalizer">
    `DateNormalizer` 接受 `config=None, **kwargs`。`format` 和 `timezone` 选项传给 `normalize_date()`，而不是构造函数：

    ```python
    from semantica.normalize import DateNormalizer

    normalizer = DateNormalizer()

    dates = [
        "January 1st, 2020",
        "01/01/2020",
        "2020-01-01T00:00:00Z",
        "yesterday",
        "3 weeks ago",
    ]

    for d in dates:
        print(normalizer.normalize_date(d, format="ISO8601", timezone="UTC"))
    ```

    需要 `python-dateutil`：`pip install python-dateutil`。如果未安装，则回退到
    `datetime.fromisoformat()`。

    **子归一化器：**

    ```python
    from semantica.normalize import TimeZoneNormalizer, RelativeDateProcessor, TemporalExpressionParser
    from datetime import datetime

    # TimeZoneNormalizer 接受 datetime 对象，而不是字符串
    tz_norm = TimeZoneNormalizer()
    dt_naive = datetime(2024, 1, 1, 9, 0)
    utc_dt = tz_norm.convert_to_utc(dt_naive)
    tz_dt  = tz_norm.normalize_timezone(dt_naive, target_timezone="America/New_York")

    # RelativeDateProcessor：reference_date 传给 process_relative_expression()，
    # 而不是构造函数
    processor = RelativeDateProcessor()
    ref = datetime(2025, 1, 15)
    result = processor.process_relative_expression("3 days ago", reference_date=ref)
    # → datetime(2025, 1, 12)

    parser = TemporalExpressionParser()
    result = parser.parse_temporal_expression("from January 2020 to March 2021")
    # → {"date": ..., "time": ..., "range": {"start": ..., "end": ...}, "relative": False}
    ```
  </Tab>
  <Tab title="NumberNormalizer">
    将带有单位、货币和缩写的数字字符串转换为 `int` 或 `float`：

    ```python
    from semantica.normalize import NumberNormalizer

    normalizer = NumberNormalizer()

    normalizer.normalize_number("$1,234.56")  # → 1234.56
    normalizer.normalize_number("42K")         # → 42000.0
    normalizer.normalize_number("$1.2B")       # → 1200000000.0
    normalizer.normalize_number("3.14e-2")     # → 0.0314
    normalizer.normalize_number("42%")         # → 0.42
    ```

    **单位和货币转换：**

    ```python
    from semantica.normalize import UnitConverter, CurrencyNormalizer

    converter = UnitConverter()
    result    = converter.convert_units(100, from_unit="kg", to_unit="pound")
    # → 220.46...

    normalized_unit = converter.normalize_unit("km")
    # → "kilometer"

    currency_norm = CurrencyNormalizer()
    result = currency_norm.normalize_currency("$42.50")
    # → {"amount": 42.50, "currency": "USD", "original": "$42.50"}
    ```
  </Tab>
  <Tab title="语言和编码">
    ### LanguageDetector

    识别文本字符串的语言。需要 `langdetect`：`pip install langdetect`。

    ```python
    from semantica.normalize import LanguageDetector

    detector = LanguageDetector()

    # detect() 返回语言代码字符串
    lang = detector.detect("Bonjour le monde")
    # → "fr"

    # detect_with_confidence() 返回 (code, confidence) 元组
    lang, confidence = detector.detect_with_confidence("Bonjour le monde")
    # → ("fr", 0.98)

    # detect_multiple() 返回 List[(code, confidence)]
    results = detector.detect_multiple("This might be mixed", top_n=3)
    # → [("en", 0.85), ...]

    # 批量：返回 List[str]
    codes = detector.detect_batch(["Hello", "Hola", "Bonjour", "Ciao"])

    # 检查特定语言
    is_english = detector.is_language(text, "en", min_confidence=0.8)
    ```

    <Note>
      `detect()` 至少需要 10 个字符才能可靠检测。对于较短文本，它返回 `default_language`（默认值：`"en"`）。
    </Note>

    <Warning>
      **`LanguageDetector.detect()` 返回 `str`，而不是字典。** 使用 `detect_with_confidence()` 获取 `(language_code, confidence)` 元组，或使用 `detect_multiple()` 获取 `List[(code, confidence)]`。
    </Warning>

    ### EncodingHandler

    检测和修复字符编码问题。需要 `chardet`：`pip install chardet`。

    ```python
    from semantica.normalize import EncodingHandler

    handler = EncodingHandler()

    # detect() 返回 (encoding_name, confidence_score) 元组
    encoding, confidence = handler.detect(raw_bytes)
    # → ("windows-1252", 0.73)

    # convert_to_utf8() 返回 str
    utf8_text = handler.convert_to_utf8(raw_bytes)
    utf8_text = handler.convert_to_utf8(raw_bytes, source_encoding="cp1252")

    # remove_bom() 返回与输入相同的类型（str 或 bytes），并去除 BOM
    clean = handler.remove_bom(text_with_bom)

    # 检测并转换磁盘上的文件
    utf8_content = handler.convert_file_to_utf8("input.txt", output_path="output.txt")
    ```

    **关键行为：**
    - `detect()` 内部使用 `chardet`：输入越长，准确性越高
    - 如果未提供 `source_encoding`，`convert_to_utf8()` 会自动检测编码，
      然后依次回退到 `latin-1`、`cp1252`、`iso-8859-1`
    - 始终先运行 `EncodingHandler`：损坏的字节会导致每个下游归一化器
      出现级联失败

    <Warning>
      **`EncodingHandler.detect()` 返回 `(str, float)` 元组，而不是字典。** 使用 `encoding, confidence = handler.detect(data)` 解包。
    </Warning>
  </Tab>
</Tabs>

<a id="datacleaner"></a>
## DataCleaner

清洗结构化记录集：在加载到向量库或图之前很有用：

```python
from semantica.normalize import DataCleaner, DataValidator, DuplicateDetector

cleaner = DataCleaner()

# clean_data：在一次遍历中完成 remove_duplicates、validate 和 handle_missing
cleaned = cleaner.clean_data(
    records,
    remove_duplicates=True,
    validate=True,
    handle_missing=True,
    missing_strategy="remove",  # "remove" | "fill" | "impute"
)

# detect_duplicates：返回 List[DuplicateGroup]
groups = cleaner.detect_duplicates(records, threshold=0.9)
for group in groups:
    print(f"Duplicate group: {len(group.records)} records, similarity={group.similarity_score:.2f}")
    print(f"  Canonical: {group.canonical_record}")

# handle_missing_values：独立的缺失值处理
processed = cleaner.handle_missing_values(records, strategy="fill", fill_value="")

# validate_data：根据模式字典验证
result = cleaner.validate_data(
    records,
    schema={"fields": {
        "name":   {"type": str,  "required": True},
        "age":    {"type": int,  "required": False},
        "active": {"type": bool, "required": False},
    }},
)
# ValidationResult 包含 .valid (bool)、.errors (list)、.warnings (list)
print(f"Valid:    {result.valid}")
print(f"Errors:   {len(result.errors)}")
print(f"Warnings: {len(result.warnings)}")
```

<a id="datacleaner-methods"></a>
### DataCleaner 方法

| 方法 | 返回值 | 描述 |
| :------ | :------- | :----------- |
| `clean_data(dataset, remove_duplicates, validate, handle_missing, **options)` | `List[Dict]` | 组合清洗流水线 |
| `detect_duplicates(dataset, threshold, key_fields)` | `List[DuplicateGroup]` | 返回相似度高于阈值的重复组 |
| `validate_data(dataset, schema)` | `ValidationResult` | 根据模式字典验证记录 |
| `handle_missing_values(dataset, strategy)` | `List[Dict]` | 删除、填充或插补缺失值 |

<Tip>
  **`DataCleaner.remove_duplicates()` 不作为独立方法存在。** 使用 `detect_duplicates()` 获取 `DuplicateGroup` 对象，或调用 `clean_data(records, remove_duplicates=True)` 来就地删除它们。
</Tip>

<Tip>
  **`DataCleaner` 对扁平记录进行操作，而不是图实体。** 对于实体级别的语义去重，请改用去重模块中的 `DuplicateDetector`。
</Tip>

<a id="pipeline-integration"></a>
## 流水线集成

```python
from semantica.pipeline import PipelineBuilder, ExecutionEngine
from semantica.ingest import FileIngestor
from semantica.normalize import TextNormalizer
from semantica.semantic_extract import NERExtractor

ingestor   = FileIngestor()
normalizer = TextNormalizer()
extractor  = NERExtractor(method="ml")

builder = PipelineBuilder()
# TextNormalizer.normalize() 同时接受来自 DocumentParser 的 str 和 List[Dict]
builder.add_step("ingest",    "file_ingest",    handler=ingestor.ingest_file)
builder.add_step("normalize", "text_normalize", handler=normalizer.normalize)
builder.add_step("extract",   "ner_extract",    handler=extractor.extract)
builder.connect_steps("ingest",    "normalize")
builder.connect_steps("normalize", "extract")

pipeline = builder.build("normalize_pipeline")
result   = ExecutionEngine().execute_pipeline(pipeline, data="data/documents/")
```

<a id="custom-normalizers"></a>
## 自定义归一化器

在方法注册表中注册自定义归一化器：

```python
from semantica.normalize.registry import method_registry

def my_normalizer(text, **kwargs):
    return text.replace("Inc.", "Incorporated")

method_registry.register("text", "expand_suffixes", my_normalizer)

from semantica.normalize import normalize_text
normalized = normalize_text("Apple Inc.", method="expand_suffixes")
# → "Apple Incorporated"
```

- [解析](parse.zh-CN.md) — 在归一化之前解析文档。
- [分块](split.zh-CN.md) — 对归一化后的文本进行分块以便嵌入。
- [去重](deduplication.zh-CN.md) — 在归一化之后解析重复实体。
- [流水线](pipeline.zh-CN.md) — 将归一化作为命名的流水线步骤。
