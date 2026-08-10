---
title: "工具模块"
description: "用于日志、校验、错误处理、进度跟踪与常见操作的共享工具。"
icon: "wrench"
---

**[English](utils.md)** · **简体中文（当前）**

**`semantica.utils`** 提供 Semantica 各处使用的 **共享基础设施**：

- 结构化日志：`setup_logging()`、`get_logger()`、`log_execution_time` 装饰器
- 校验辅助方法：`validate_entity()` 和 `validate_config()` 返回 `(bool, Optional[str])` 且不会抛出异常
- 进度跟踪：`ProgressTracker` 类与带 ETA 的 `track_progress()` 可迭代对象包装器
- 类型化异常：`SemanticaError`、`ValidationError`、`ProcessingError`、`ConfigurationError`、`QualityError`

大多数用户不会直接调用 utils：它是所有模块的 **共享基础**。


<a id="exported-classes"></a>
## 导出的类

| 名称 | 类型 | 角色 |
| :--- | :--- | :--- |
| `setup_logging` | function | 配置 `semantica` 根日志记录器：接受 `level`、`file`、`console`、`rotation` 关键字参数 |
| `get_logger` | function | 获取具名日志记录器实例（`semantica.<name>`） |
| `log_execution_time` | 装饰器 | 包装函数：记录名称、执行时间与成功/失败 |
| `log_performance` | function | 记录预先采集的性能指标：`log_performance(func_name, execution_time, **metrics)` |
| `validate_entity` | function | 校验实体字典：返回 `(bool, Optional[str])`；不抛出异常 |
| `validate_config` | function | 校验配置字典：返回 `(bool, Optional[str])`；不抛出异常 |
| `ProgressTracker` | 类 | 基于类的进度跟踪器，带 ETA 和步骤回调 |
| `track_progress` | function | 用实时进度条包装任意可迭代对象 |
| `clean_text` | function | 归一化空白字符并去除零宽控制字符 |
| `hash_data` | function | 对 `str`、`bytes` 或 `dict` 进行确定性 SHA-256 哈希 |
| `SemanticaError` | 异常 | 所有 Semantica 错误的基类异常 |
| `ValidationError` | 异常 | 当输入未通过校验时抛出；包含 `.field`、`.value`、`.message` |
| `ProcessingError` | 异常 | 在抽取、图构建或流水线步骤期间抛出；包含 `.stage` |
| `ConfigurationError` | 异常 | 配置校验失败时抛出 |
| `QualityError` | 异常 | 数据质量低于阈值时抛出 |

<a id="what-you-get"></a>
## 你将获得

- **日志** — 通过 `@log_execution_time` 装饰器实现结构化日志，并支持通过环境变量配置质量指标。
- **校验** — `validate_entity` 和 `validate_config`，配以携带字段和值上下文的类型化 `ValidationError`。
- **进度跟踪** — `track_progress` 包装任意可迭代对象：自动检测控制台与 Jupyter 环境以选择正确的渲染器。
- **辅助函数** — `clean_text`、`hash_data`、`safe_filename` 以及整个框架中使用的嵌套字典工具。
- **异常层次结构** — `SemanticaError` → `ValidationError`、`ProcessingError`：用于针对性恢复的类型化异常。
- **文件工具** — `read_json_file` 在失败时抛出 `FileNotFoundError` 或 `json.JSONDecodeError`：JSON I/O 周围无需样板 try/except。

<a id="logging"></a>
## 日志

<Steps>
  <Step title="在应用启动时初始化日志">
    ```python
    from semantica.utils import setup_logging, get_logger

    setup_logging(level="INFO")   # "DEBUG" | "INFO" | "WARNING" | "ERROR"
    logger = get_logger(__name__)
    ```

    <Warning>
      **在应用启动时调用一次 `setup_logging(level="INFO")`。** 如果不调用，Semantica 会回退到 Python 的根日志记录器，它可能是静默的或配置不当。在其他 Semantica 模块导入之前调用它，以捕获初始化消息。
    </Warning>
  </Step>
  <Step title="用性能装饰器监测耗时函数">
    ```python
    from semantica.utils import log_execution_time

    @log_execution_time
    def expensive_step(data):
        ...
    # 日志记录："expensive_step completed in 2.34s"
    ```

    <Tip>
      **`@log_execution_time` 是性能装饰器。** 将其应用于任意函数即可自动记录其名称、执行时间与成功/失败。`log_performance` 是一个较低层级的函数，用于记录你已采集的指标：它不是装饰器。
    </Tip>
  </Step>
  <Step title="通过环境变量配置">
    ```bash
    export SEMANTICA_LOG_LEVEL=DEBUG
    export SEMANTICA_LOG_FORMAT=json     # "json" | "text"
    export SEMANTICA_DISABLE_PROGRESS=true
    ```
  </Step>
</Steps>

<a id="validation"></a>
## 校验

```python
from semantica.utils import validate_entity, validate_config, ValidationError

# validate_entity 返回 (is_valid, error_message)
is_valid, error = validate_entity({"id": "1", "type": "PERSON", "text": "Alice"})
if not is_valid:
    raise ValidationError(error)

# validate_config 返回 (is_valid, error_message)
is_valid, error = validate_config(config, required_keys=["model", "provider"])
if not is_valid:
    print(f"Invalid config: {error}")
```

| 函数 | 描述 | 返回值 |
| :-------- | :----------- | :------- |
| `validate_entity(data)` | 检查实体字典是否含 **必填** 字段（`id`、`text`、`type`）且类型正确 | `Tuple[bool, Optional[str]]` |
| `validate_config(cfg, required_keys=None)` | 检查配置字典；可选地强制要求 **必填** 键 | `Tuple[bool, Optional[str]]` |

<a id="progress-tracking"></a>
## 进度跟踪

```python
from semantica.utils import track_progress

# 包装任意可迭代对象：自动检测控制台与 Jupyter
for item in track_progress(items, desc="Processing documents"):
    process(item)
```

支持：

- **控制台**：带 ETA 的 tqdm 进度条
- **Jupyter**：notebook 兼容的组件（自动检测）
- **文件**：将进度写入日志文件

<Tip>
  **`track_progress` 会自动检测 Jupyter。** 在终端中它渲染 tqdm 进度条；在 Jupyter notebook 中它渲染交互式组件。你无需检查环境：同一次调用在两者中都有效。
</Tip>

<a id="helper-functions"></a>
## 辅助函数

```python
from semantica.utils import clean_text, hash_data, safe_filename

# 归一化空白字符并去除控制字符
clean = clean_text("  Hello   World  ")     # -> "Hello World"

# 对 string、bytes 或 dict 进行确定性 SHA-256 哈希
uid   = hash_data({"key": "value"})         # -> hex digest string

# 将字符串净化为可用作文件名的形式
fname = safe_filename("My File?.txt")       # -> "My_File.txt"
```

<Tip>
  **`hash_data()` 在多次运行间是确定性的。** 给定相同的输入字典（任何 JSON 可序列化对象），`hash_data()` 始终返回相同的 SHA-256 十六进制字符串：适合作为流水线步骤中的缓存键或幂等令牌。
</Tip>

<a id="nested-dict-utilities"></a>
## 嵌套字典工具

用于深度配置访问的辅助函数：在 `Config` 和 `ConfigManager` 内部被广泛使用：

```python
from semantica.utils import get_nested_value, set_nested_value, merge_dicts

config = {
    "processing": {"batch_size": 32, "max_workers": 4},
    "llm":        {"provider": "groq", "model": "llama-3.3-70b-versatile"},
}

# 点号表示法读取：键路径不存在时返回默认值
batch = get_nested_value(config, "processing.batch_size", default=16)
# -> 32

# 点号表示法写入
set_nested_value(config, "processing.batch_size", 64)

# 深度合并：嵌套键递归合并（默认 deep=True）
base      = {"a": {"x": 1, "y": 2}, "b": 3}
overrides = {"a": {"y": 99, "z": 4}, "c": 5}
merged    = merge_dicts(base, overrides)
# -> {"a": {"x": 1, "y": 99, "z": 4}, "b": 3, "c": 5}
```

<a id="exception-hierarchy"></a>
## 异常层次结构

<AccordionGroup>
  <Accordion title="异常类型及其抛出时机">

```python
from semantica.utils import SemanticaError, ValidationError, ProcessingError

try:
    run_pipeline(data)
except ValidationError as e:
    # 输入数据未通过模式校验
    logger.error("Validation failed: %s", e.message)
except ProcessingError as e:
    # 抽取或图构建期间失败
    logger.error("Processing failed at stage %s: %s", e.stage, e)
except SemanticaError as e:
    # 捕获所有 Semantica 框架错误的兜底异常
    logger.error("Framework error: %s", e)
```

| 异常 | 抛出时机 | 关键属性 |
| :--------- | :----------- | :-------------- |
| `SemanticaError` | 基类：所有框架错误继承自它 | `.message`、`.context`、`.error_code` |
| `ValidationError` | 输入数据未通过模式或类型校验 | `.field`、`.value`、`.constraint` |
| `ProcessingError` | 在抽取、图构建或流水线步骤期间失败 | `.stage`、`.input_data`、`.output_data` |
| `ConfigurationError` | 配置键缺失或类型错误 | `.config_key`、`.config_value`、`.expected_type` |
| `QualityError` | 数据质量分数低于阈值 | `.quality_score`、`.threshold`、`.metrics` |

  </Accordion>
</AccordionGroup>

<Tip>
  **将 `SemanticaError` 作为最宽泛的异常捕获网。** 所有框架错误都继承自 `SemanticaError`，因此 `except SemanticaError` 可以捕获校验失败、处理错误以及介于其间的一切。使用具体子类来实现针对性的恢复逻辑。
</Tip>

<a id="file-utilities"></a>
## 文件工具

```python
from semantica.utils import read_json_file

# 读取并解析 JSON 文件：失败时抛出 FileNotFoundError 或 json.JSONDecodeError
config = read_json_file("config.json")
```

- [Core](core.zh-CN.md) — 在内部使用 Utils 的框架编排。
- [Pipeline](pipeline.zh-CN.md) — 使用 ProgressTracker 进行每步骤跟踪。
