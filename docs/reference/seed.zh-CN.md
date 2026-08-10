---
title: "种子数据模块"
description: "从经过验证的结构化来源引导知识图谱：分类法、参考表、产品目录和领域锚点。"
icon: "database"
---

**[English](seed.md)** · **简体中文（当前）**

**`semantica.seed`** 为你的知识图谱提供一个**可靠、经过验证的起点**：

- 首先加载经过验证的参考数据：ISO 代码、员工名册、产品目录、领域分类法
- `SeedDataManager` 将新抽取的数据合并到基础节点上，不会创建重复项
- 支持 JSON、CSV 和编程式注册的种子数据来源
- 从结构化种子数据生成确定性测试图谱
- 将实体抽取锚定到已知实体，减少幻觉和重复节点


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `SeedDataManager` | 协调器：`register_source`、`load_source`、`create_foundation_graph`、`integrate_with_extracted` |
| `SeedDataSource` | 配置数据类：`{name, format, location, entity_type, verified, version, metadata}` |
| `SeedData` | 容器数据类：`{entities, relationships, properties, metadata}` |

<a id="what-you-get"></a>
## 你将获得

- **SeedDataManager** — 注册来源、构建基础图谱、验证质量，并与抽取的数据合并。
- **SeedDataSource** — 类型化的来源定义，支持 CSV、JSON、SQL 和 API，带有格式特定的配置。
- **基础图谱** — 一次性从所有已注册来源构建基础图谱，随时可与抽取的数据合并。
- **合并策略** — `seed_first`、`extracted_first` 和 `merge`，支持属性级别的冲突检测。
- **验证** — 在加载前检查必填字段、ID 唯一性、类型一致性、引用完整性和编码验证。
- **版本管理** — 跨流水线运行跟踪种子数据版本，并对比版本间的变化。

<Tip>
  **何时使用种子数据模块：** 使用结构化参考数据（分类法、用户列表、产品目录）进行引导，加载抽取数据不应覆盖的不可变事实（ISO 国家代码、标准本体术语），使用确定性数据集确保测试可复现性，以及使用规范形式锚定实体消歧。
</Tip>

<a id="quick-start"></a>
## 快速上手

<Steps>
  <Step title="注册你的种子数据来源">
    ```python
    from semantica.seed import SeedDataManager

    manager = SeedDataManager()

    manager.register_source("countries",  "csv",  "data/countries.csv")
    manager.register_source("taxonomy",   "json", "data/taxonomy.json")
    manager.register_source("employees",  "csv",  "data/employees.csv")
    ```

    <Tip>
      **在调用 `create_foundation_graph()` 之前注册所有来源。** `create_foundation_graph()` 一次性处理所有已注册的来源。在调用之后再注册来源意味着该来源会被静默排除。在脚本开头注册所有来源，然后调用一次 `create_foundation_graph()`。
    </Tip>
  </Step>
  <Step title="构建基础图谱">
    ```python
    foundation_kg = manager.create_foundation_graph()
    print(f"Foundation nodes: {len(foundation_kg['entities'])}")
    print(f"Foundation edges: {len(foundation_kg['relationships'])}")
    ```
  </Step>
  <Step title="加载前验证">
    ```python
    # 注意：validate_quality 期望传入图谱字典，但 load_source 返回列表。
    # 为演示起见，这里验证基础图谱。
    report = manager.validate_quality(foundation_kg)
    if not report["valid"]:
        for error in report["errors"]:
            print(f"Error: {error}")
        for warning in report["warnings"]:
            print(f"Warning: {warning}")
    else:
        print(f"Validated {report['metrics']['entity_count']} entities: no issues found")
    ```

    <Warning>
      **加载前验证。** `manager.validate_quality(seed_data)` 会在缺少必填字段、类型不一致和重复 ID 损坏你的图谱之前捕获它们。加载后再运行验证意味着你需要回滚。验证很快：始终先运行它。
    </Warning>
  </Step>
  <Step title="与抽取的数据合并">
    ```python
    from semantica.semantic_extract import NERExtractor

    extractor = NERExtractor(method="ml")
    new_entities = extractor.extract("Apple Inc. partners with Microsoft Corp.")
    
    # 与种子数据合并 - 注意正确的参数名称
    final_kg = manager.integrate_with_extracted(
        seed_data=foundation_kg,
        extracted_data={"entities": new_entities, "relationships": []},
        merge_strategy="merge"
    )
    ```

    <Warning>
      **先加载种子数据，再加载抽取数据。** 种子数据是你的真值基础：已归一化、已策展、已去重。先用 `create_foundation_graph()` 加载它，然后在上面合并抽取的实体。错误的合并顺序会让有噪声的抽取数据覆盖可信的参考值。
    </Warning>
  </Step>
</Steps>

<a id="seeddatasource-types"></a>
## SeedDataSource 类型

<Tabs>
  <Tab title="CSV">
    ```python
    from semantica.seed import SeedDataSource, SeedDataManager

    csv_source = SeedDataSource(
        name="employees",
        format="csv",
        location="data/employees.csv",
        entity_type="Person",
        verified=True,
        metadata={"description": "Company employee list with titles and departments"}
    )

    manager = SeedDataManager()
    manager.register_source(
        "employees", "csv", "data/employees.csv", entity_type="Person", verified=True
    )
    ```
  </Tab>
  <Tab title="JSON">
    ```python
    json_source = SeedDataSource(
        name="taxonomy",
        format="json",
        location="knowledge/taxonomy.json",
        entity_type="Concept",
        relationship_type="subclass_of",
        verified=True,
        metadata={"version": "2.1", "source": "domain_expert"}
    )

    manager.register_source(
        "taxonomy", "json", "knowledge/taxonomy.json",
        entity_type="Concept", relationship_type="subclass_of"
    )
    ```
  </Tab>
  <Tab title="Database">
    ```python
    db_source = SeedDataSource(
        name="geographic",
        format="database",
        location="postgresql://user:pass@host/geonames",
        entity_type="Location",
        verified=False,  # 需要验证
        metadata={"table": "countries", "last_sync": "2024-01-15"}
    )

    manager.register_source(
        "geographic", "database", "postgresql://user:pass@host/geonames",
        entity_type="Location", verified=False
    )
    ```
  </Tab>
  <Tab title="API">
    ```python
    api_source = SeedDataSource(
        name="wikidata",
        format="api",
        location="https://wikidata.org/sparql",
        entity_type="Entity",
        metadata={"auth_required": False, "rate_limit": 60}
    )

    manager.register_source(
        "wikidata", "api", "https://wikidata.org/sparql",
        entity_type="Entity"
    )
    ```
  </Tab>
</Tabs>

<a id="seeddatamanager-reference"></a>
## SeedDataManager 参考

| 方法 | 描述 |
| :------ | :----------- |
| `register_source(name, format, location, **config)` | 向管理器添加新的数据来源 |
| `load_source(source_name)` | 加载并返回已注册来源的原始数据 |
| `create_foundation_graph()` | 从所有已注册来源构建初始图谱 |
| `integrate_with_extracted(seed_data, extracted_data, merge_strategy)` | 将种子数据与新抽取的实体/关系合并 |
| `validate_quality(seed_data)` | 检查数据完整性并返回验证报告 |
| `export_seed_data(path, format)` | 将处理后的种子数据保存到文件 |

`integrate_with_extracted()` 中解决冲突的不同策略：

<Tabs>
  <Tab title="seed_first">
    **种子数据在冲突中胜出**：保留策展的关系而非抽取的关系。

    ```python
    final_kg = manager.integrate_with_extracted(
        seed_data=foundation_kg,
        extracted_data=new_data,
        merge_strategy="seed_first"
    )
    ```

    当种子数据高置信度且抽取属于探索性质时使用。
  </Tab>
  <Tab title="extracted_first">
    **抽取数据在冲突中胜出**：用新信息覆盖种子数据。

    ```python
    final_kg = manager.integrate_with_extracted(
        seed_data=foundation_kg,
        extracted_data=new_data,
        merge_strategy="extracted_first"
    )
    ```

    当抽取质量已知良好时，用于快速原型设计。
  </Tab>
  <Tab title="merge">
    **智能冲突解决**：合并互补属性，对实体去重。

    ```python
    final_kg = manager.integrate_with_extracted(
        seed_data=foundation_kg,
        extracted_data=new_data,
        merge_strategy="merge"
    )
    ```

    当种子数据和抽取数据都有价值时，用于生产流水线。
  </Tab>
</Tabs>

<Tip>
  **对参考数据使用 `seed_first` 合并策略。** 当种子数据编码了权威事实（官方公司名称、规范分类法 ID、员工记录）时，`merge_strategy="seed_first"` 确保这些值胜过抽取的值。仅当抽取数据可能比种子数据更新时才使用 `merge`。
</Tip>

<a id="full-pipeline-example"></a>
## 完整流水线示例

```python
from semantica.seed import SeedDataManager
from semantica.parse import DocumentParser
from semantica.split import TextSplitter
from semantica.semantic_extract import NERExtractor, RelationExtractor

# 初始化各组件
manager   = SeedDataManager()
parser    = DocumentParser()
splitter  = TextSplitter(method="sentence", chunk_size=200)
ner       = NERExtractor(method="ml")
rel_ext   = RelationExtractor(method="ml")

# 注册种子来源
manager.register_source("taxonomy", "json", "seeds/domain_taxonomy.json")
manager.register_source("entities", "csv",  "seeds/known_entities.csv")

# 创建基础图谱
foundation_kg = manager.create_foundation_graph()
print(f"Foundation: {len(foundation_kg['entities'])} entities, {len(foundation_kg['relationships'])} relationships")

# 处理新文档
parsed = parser.parse("research_paper.pdf")
chunks = splitter.split(parsed["full_text"])

# 从每个块中抽取
all_entities = []
all_relations = []
for chunk in chunks:
    entities = ner.extract(chunk.text)
    relations = rel_ext.extract(chunk.text)
    all_entities.extend(entities)
    all_relations.extend(relations)

# 与基础图谱合并
extracted_data = {"entities": all_entities, "relationships": all_relations}
final_kg = manager.integrate_with_extracted(
    seed_data=foundation_kg,
    extracted_data=extracted_data,
    merge_strategy="merge"
)

print(f"Final graph: {len(final_kg['entities'])} entities, {len(final_kg['relationships'])} relationships")

# 导出供下游使用
manager.export_seed_data("output/enriched_kg.json", format="json")
```

<a id="yaml-configuration"></a>
## YAML 配置

在生产部署中用 YAML 定义来源：切换环境无需修改代码：

```yaml
seed:
  sources:
    - name: "employees"
      format: "csv"
      location: "./data/employees.csv"
      config:
        id_column: "employee_id"
        type: "Person"
    - name: "taxonomy"
      format: "json"
      location: "./data/taxonomy.json"
    - name: "products"
      format: "sql"
      location: "${DATABASE_URL}"
      config:
        query: "SELECT id, name, category FROM products WHERE active = true"
  merge:
    strategy: "merge"
  validation:
    strict: true
    required_fields: ["id", "type"]
```

环境变量覆盖：

```bash
export SEMANTICA_SEED_DATA_DIR=./data/seed
export SEMANTICA_SEED_MERGE_STRATEGY=seed_first
```

<Tip>
  **生产部署使用 YAML 配置。** 在 Python 脚本中硬编码来源路径会使环境切换（开发 → 预发布 → 生产）变得脆弱。在 `config.yaml` 的 `seed:` 键下声明来源，并用 `SEMANTICA_SEED_DATA_DIR` 覆盖路径。这样，相同的代码可以在每个环境中运行。
</Tip>

- [摄取](ingest.zh-CN.md) — 将非结构化数据与种子数据一起加载。
- [知识图谱](kg.zh-CN.md) — 种子数据填充的目标图谱。
- [去重](deduplication.zh-CN.md) — 在种子数据与抽取数据合并期间处理重复项。
- [流水线](pipeline.zh-CN.md) — 将种子数据加载作为命名的流水线步骤纳入。
