---
title: "Core 模块"
description: "框架编排、生命周期管理、配置和插件系统。"
icon: "gear"
---

**[English](core.md)** · **简体中文（当前）**

**`semantica.core`** 是框架的**协调层**：

- `Semantica` 编排器从 YAML 配置协调整个知识图谱构建流水线
- `ConfigManager` 加载 YAML 配置，支持深度合并、验证和环境变量覆盖
- `PluginRegistry` 支持在运行时动态注册和加载组件
- `LifecycleManager` 管理启动/关闭，包含健康监控和生命周期钩子

<Tip>
  在绝大多数用例中，请直接使用单独的模块。只有在需要应用级生命周期管理、集中化配置或插件系统时，才使用 `semantica.core`。
</Tip>


<a id="what-you-get"></a>
## 你将获得

- **Semantica** — 高层编排器：从单个 `config.yaml` 协调整个知识图谱构建流水线。应用级部署的入口点。
- **ConfigManager** — YAML 配置，支持深度合并、`SEMANTICA_` 环境变量覆盖和点号表示法嵌套键访问。将密钥排除在源文件之外。
- **LifecycleManager** — 有序的启动/关闭钩子、健康监控和 6 状态机。对于 FastAPI 应用等长时间运行的服务至关重要。
- **PluginRegistry** — 注册自定义摄取器、解析器、导出器或任何组件。在运行时按名称加载：无需导入。


<a id="exported-classes"></a>
## 导出的类

| 类 | 角色 |
| :--- | :--- |
| `Semantica` | 编排入口：协调整个知识图谱构建流水线 |
| `ConfigManager` | YAML 配置加载、深度合并、验证和环境变量覆盖 |
| `LifecycleManager` | 启动/关闭状态机，带健康监控和生命周期钩子 |
| `PluginRegistry` | 动态插件发现、注册和加载 |
| `method_registry` | 全局 `MethodRegistry` 实例：注册和分发自定义编排方法 |


<a id="semantica-orchestration"></a>
## Semantica（编排）

**`Semantica`** 是协调**完整知识图谱构建流水线**的高层入口：

```python
from semantica.core import Semantica, ConfigManager

config_manager = ConfigManager()
config = config_manager.load_from_file("config.yaml")

framework = Semantica(config=config)
framework.initialize()

try:
    result = framework.build_knowledge_base(
        sources=["doc1.pdf", "doc2.docx"],
        embeddings=True,
        graph=True,
    )
    status = framework.get_status()
    print(f"State: {status['state']}")
finally:
    framework.shutdown(graceful=True)
```

<a id="core-methods"></a>
### 核心方法

| 方法 | 描述 |
| :------ | :----------- |
| `initialize()` | 初始化所有框架组件 |
| `build_knowledge_base(sources, **kwargs)` | 编排完整的知识图谱构建流水线 |
| `run_pipeline(pipeline, data)` | 执行现有的 `Pipeline` 实例 |
| `get_status()` | 返回系统健康状态和当前状态 |
| `shutdown(graceful=True)` | 优雅关闭：等待进行中的操作 |

<a id="configmanager"></a>
## ConfigManager

集中化配置加载，支持深度合并和环境变量覆盖：

```python
from semantica.core import ConfigManager

manager = ConfigManager()
config = manager.load_from_file("config.yaml")

# 合并基础配置与环境特定的覆盖
merged = manager.merge_configs(
    manager.load_from_file("base.yaml"),
    manager.load_from_file("prod.yaml"),
)

# 使用点号表示法进行嵌套键访问
batch_size = config.get("processing.batch_size", default=16)
config.set("processing.batch_size", 64)
config.validate()
```

<a id="yaml-configuration"></a>
### YAML 配置

```yaml
llm_provider:
  name: openai
  model: gpt-4o
  # 不要将 API 密钥放在 YAML 中：请改用环境变量。
  # ConfigManager 使用 yaml.safe_load() 加载 YAML，该方法不会
  # 插值 ${...} 表达式。通过环境变量设置密钥（见下文）。

processing:
  batch_size: 32
  max_workers: 4

quality:
  min_confidence: 0.7

logging:
  level: INFO
```

环境变量覆盖（前缀 `SEMANTICA_`）：

使用双下划线（`__`）来生成嵌套键访问的点分隔符。实现会去除 `SEMANTICA_` 前缀，将结果转为小写，并将 `__` 替换为 `.`，然后对配置字典调用 `set_nested_value()`。

```bash
# 双下划线映射到嵌套键：
export SEMANTICA_LLM_PROVIDER__MODEL=gpt-4o
export SEMANTICA_LLM_PROVIDER__NAME=openai
export SEMANTICA_PROCESSING__BATCH_SIZE=64
export SEMANTICA_QUALITY__MIN_CONFIDENCE=0.8
export SEMANTICA_LOGGING__LEVEL=DEBUG
```

<a id="lifecyclemanager"></a>
## LifecycleManager

通过定义的状态机和有序的启动/关闭钩子管理框架状态：

**状态机：** `UNINITIALIZED` → `INITIALIZING` → `READY` → `RUNNING` → `STOPPING` → `STOPPED`

```python
from semantica.core import LifecycleManager

manager = LifecycleManager()

def init_db():
    print("Initializing database...")

def cleanup_db():
    print("Closing database connections...")

# 启动时优先级值较低的先运行
# 关闭时优先级值较高的先运行
manager.register_startup_hook(init_db,     priority=10)
manager.register_shutdown_hook(cleanup_db, priority=10)

manager.startup()

# 组件健康监控
class DatabaseComponent:
    def health_check(self):
        return {"healthy": True, "message": "Connected"}

manager.register_component("database", DatabaseComponent())
summary = manager.get_health_summary()
# → {
#     "state": "ready",
#     "is_healthy": True,
#     "total_components": 1,
#     "healthy_components": 1,
#     "unhealthy_components": 0,
#     "last_check": 1234567890.0,
#     "components": {
#         "database": {"healthy": True, "message": "Connected", "timestamp": ...}
#     }
#   }

manager.shutdown(graceful=True)
```

<a id="pluginregistry"></a>
## PluginRegistry

注册参与完整流水线的自定义组件：包含溯源追踪、重试策略和并行执行：

```python
from semantica.core import PluginRegistry

class MyPlugin:
    def initialize(self):
        print("Plugin initialized")

    def execute(self, data):
        return {"processed": True}

registry = PluginRegistry(plugin_paths=["./plugins"])
registry.register_plugin("my_plugin", MyPlugin, version="1.0.0")

plugin = registry.load_plugin("my_plugin", api_key="xxx")
result = plugin.execute("sample data")

for info in registry.list_plugins():
    print(f"{info['name']}: {info['version']}")
```

<a id="methodregistry"></a>
## MethodRegistry

注册自定义编排方法并按名称分发：

```python
from semantica.core import method_registry
from semantica.core.methods import build_knowledge_base

def fast_kb_builder(sources, **kwargs):
    # 自定义逻辑：跳过向量嵌入以提升速度
    ...

method_registry.register("knowledge_base", "fast", fast_kb_builder)

result = build_knowledge_base(sources=["doc.pdf"], method="fast")
```

<a id="when-to-use-core-vs-individual-modules"></a>
## 何时使用 Core 与单独模块

| 场景 | 推荐方案 |
| :-------- | :-------------------- |
| 单次抽取任务 | `from semantica.semantic_extract import NERExtractor` |
| 构建知识图谱 | `from semantica.kg import GraphBuilder` |
| 多步流水线 | `from semantica.pipeline import Pipeline` |
| 应用级生命周期 + 配置 | `from semantica.core import Semantica, ConfigManager` |
| 自定义分发 / 插件 | `from semantica.core import method_registry, PluginRegistry` |

<Tip>
  只有在构建长时间运行的应用（例如 FastAPI 服务）时才使用 `Semantica` 和 `LifecycleManager`，这些应用需要有序启动、健康检查和优雅关闭。对于脚本和笔记本，请直接使用单独的模块。
</Tip>

- [流水线](pipeline.zh-CN.md) — 流水线执行和步骤编排。
- [工具](utils.zh-CN.md) — Core 内部使用的共享工具。
- [快速上手](../getting-started.zh-CN.md) — 在使用 Core 之前学习基础知识。
- [LLMs](llms.zh-CN.md) — 通过 ConfigManager 配置 LLM 提供商。
