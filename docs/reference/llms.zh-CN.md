---
title: "LLM 模块"
description: "为 Groq、OpenAI、LiteLLM（Anthropic、Gemini、Ollama、DeepSeek、Azure、Bedrock、100+ 模型）与 HuggingFace 提供统一接口。"
icon: "microchip"
---

**[English](llms.md)** · **简体中文（当前）**

**`semantica.llms`** 在所有主流 LLM 提供商之上提供**单一一致的 API**：

- 每个提供商都可作为抽取器、推理器与智能体中 `llm_provider=` 参数的即插即用替代
- `LiteLLM` 用单个类和模型字符串前缀即可路由到 100+ 提供商
- `HuggingFaceLLM` 完全在本地运行：无需 API 密钥，不发起网络调用
- 通过 `generate_with_schema()` 实现结构化输出，可从任意提供商抽取 JSON
- 支持流式、工具使用，以及用于批量推理的 `generate_batch()`


<a id="exported-classes"></a>
## 导出的类

```python
from semantica.llms import Groq, OpenAI, LiteLLM, HuggingFaceLLM
```

| 类 | 提供商 | 是否需要 API 密钥 |
| :----- | :-------- | :---------------- |
| `Groq` | Groq Cloud | `GROQ_API_KEY` |
| `OpenAI` | OpenAI / 任何 OpenAI 兼容网关 | `OPENAI_API_KEY` |
| `LiteLLM` | 通过 LiteLLM 路由访问 100+ 提供商 | 取决于模型 |
| `HuggingFaceLLM` | 本地 HuggingFace Transformers | 无需（本地） |

<Tip>
  **Anthropic、Gemini、Ollama、DeepSeek、Azure、Bedrock、Cohere 等 90+ 其他提供商** 都可通过 `LiteLLM` 使用其模型字符串前缀访问。参见下文的 [LiteLLM 章节](#litellm-100-providers)。
</Tip>

<a id="what-you-get"></a>
## 你将获得

- **统一的 `LLMProvider` 接口**：一行代码即可切换提供商，无需改动应用代码
- **`LiteLLM`**：通过模型字符串路由用单个类访问 100+ 提供商
- **本地模型**：`HuggingFaceLLM` 完全在本地运行，无需 API 密钥
- **流式**：逐 token 输出，打造低延迟体验
- **自定义网关**：通过 `base_url` 将 `OpenAI` 指向任意 OpenAI 兼容端点

<a id="choosing-a-provider"></a>
## 选择提供商

<Tabs>
  <Tab title="Groq：入门之选">
    提供免费额度，推理速度最快，零配置门槛。最适用于开发与高吞吐抽取流水线。

    | | |
    | :-- | :-- |
    | **速度** | 非常快：100+ tok/s |
    | **成本** | 提供免费额度 |
    | **上下文** | 128k |
    | **最适用于** | 开发、高吞吐抽取 |

    ```python
    import os
    from semantica.llms import Groq

    llm = Groq(
        model="llama-3.1-8b-instant",
        api_key=os.getenv("GROQ_API_KEY"),
        temperature=0.0,
    )
    ```

    在 [console.groq.com](https://console.groq.com) 获取免费密钥。
  </Tab>
  <Tab title="OpenAI：生产之选">
    准确率最高，JSON 模式与函数调用体验最佳。适用于对抽取质量有要求的流水线。

    | | |
    | :-- | :-- |
    | **速度** | 快 |
    | **成本** | 中 |
    | **上下文** | 128k |
    | **最适用于** | 生产质量、JSON 抽取、函数调用 |

    ```python
    import os
    from semantica.llms import OpenAI

    llm = OpenAI(
        model="gpt-4o",
        api_key=os.getenv("OPENAI_API_KEY"),
        temperature=0.0,
        max_tokens=4096,
    )
    ```
  </Tab>
  <Tab title="Ollama：本地 / 离线">
    完全在本地运行：无需 API 密钥，数据不离开你的基础设施。离线部署所必需。

    | | |
    | :-- | :-- |
    | **速度** | 中（取决于硬件） |
    | **成本** | 免费（仅本地算力） |
    | **上下文** | 因模型而异 |
    | **最适用于** | 隐私、离线、自定义微调 |

    ```bash
    # 先安装 Ollama 并拉取一个模型
    ollama pull llama3.2:3b
    ```

    ```python
    from semantica.llms import LiteLLM

    llm = LiteLLM(
        model="ollama/llama3.2:3b",
        api_base="http://localhost:11434",  # Ollama 默认端口
    )
    ```

    <Note>
      无需 API 密钥。在创建 `LiteLLM` 实例之前，请确保 Ollama 服务正在运行（`ollama serve`）。
    </Note>
  </Tab>
  <Tab title="Claude：推理之选">
    上下文窗口最大，多跳推理最佳，安全标准最高。适用于复杂分析与长文档抽取。

    | | |
    | :-- | :-- |
    | **速度** | 快 |
    | **成本** | 中 |
    | **上下文** | 200k |
    | **最适用于** | 复杂推理、长文档、安全关键型输出 |

    ```python
    import os
    from semantica.llms import LiteLLM

    llm = LiteLLM(
        model="anthropic/claude-sonnet-4-20250514",
        api_key=os.getenv("ANTHROPIC_API_KEY"),
        temperature=0.0,
    )
    ```
  </Tab>
  <Tab title="DeepSeek：成本优化">
    海量工作负载下每 token 成本最低。在代码与结构化数据抽取方面表现强劲。

    | | |
    | :-- | :-- |
    | **速度** | 快 |
    | **成本** | 极低 |
    | **上下文** | 64k |
    | **最适用于** | 高吞吐流水线、代码任务、成本敏感型工作负载 |

    ```python
    import os
    from semantica.llms import LiteLLM

    llm = LiteLLM(
        model="deepseek/deepseek-chat",
        api_key=os.getenv("DEEPSEEK_API_KEY"),
        temperature=0.0,
    )
    ```
  </Tab>
</Tabs>

<a id="api-key-setup"></a>
## API 密钥配置

<a id="environment-variables-recommended"></a>
### 环境变量（推荐）

```bash
# 添加到你的 shell 配置文件（.bashrc、.zshrc 等）
export GROQ_API_KEY="your_groq_api_key_here"
export OPENAI_API_KEY="your_openai_api_key_here" 
export ANTHROPIC_API_KEY="your_anthropic_api_key_here"

# 重新加载 shell
source ~/.bashrc
```

<a id="configuration-file-method"></a>
### 配置文件方式

```yaml
# config.yaml
llm_provider:
  name: groq
  model: llama-3.1-8b-instant
  temperature: 0.0
# 设置 GROQ_API_KEY 环境变量并传给构造函数
```

<a id="programmatic-setup"></a>
### 编程式配置

```python
import os
from semantica.llms import Groq, LiteLLM

# 方法 1：直接传入 API 密钥
llm = Groq(api_key="your-api-key-here", model="llama-3.1-8b-instant")

# 方法 2：环境变量（首选）
llm = Groq(api_key=os.getenv("GROQ_API_KEY"), model="llama-3.1-8b-instant")

# 方法 3：通过 LiteLLM 使用多个提供商
providers = {
    "fast": LiteLLM(model="groq/llama-3.1-8b-instant", api_key=os.getenv("GROQ_API_KEY")),
    "smart": LiteLLM(model="anthropic/claude-sonnet-4-20250514", api_key=os.getenv("ANTHROPIC_API_KEY"))
}
```

<a id="security-best-practices"></a>
### 安全最佳实践

<Warning>
切勿将 API 密钥提交到版本控制。请使用环境变量或安全的密钥管理方案。
</Warning>

```python
# 错误 —— API 密钥写在代码里
llm = Groq(api_key="gsk_abc123...", model="llama-3.1-8b-instant")

# 正确 —— 使用环境变量
llm = Groq(api_key=os.getenv("GROQ_API_KEY"), model="llama-3.1-8b-instant")
```

<a id="providers"></a>
## 提供商

<CodeGroup>

```python Groq
import os
from semantica.llms import Groq

llm = Groq(
    model="llama-3.3-70b-versatile",   # 推荐；实现默认值：llama-3.1-8b-instant
    api_key=os.getenv("GROQ_API_KEY"),
    max_tokens=64000,
    temperature=0.0,
)
# **最适用于：** 高吞吐抽取、低成本的快速推理
```

```python OpenAI
import os
from semantica.llms import OpenAI

llm = OpenAI(
    model="gpt-4o",                     # 推荐；实现默认值：gpt-3.5-turbo
    api_key=os.getenv("OPENAI_API_KEY"),
    temperature=0.0,
)
# **最适用于：** 通用任务、函数调用、JSON 模式
```

```python LiteLLM (100+ providers)
import os
from semantica.llms import LiteLLM

# pip install "semantica[llm-litellm]"

# Anthropic Claude
llm = LiteLLM(model="anthropic/claude-opus-4-5",         api_key=os.getenv("ANTHROPIC_API_KEY"))

# Google Gemini
llm = LiteLLM(model="gemini/gemini-1.5-pro",             api_key=os.getenv("GOOGLE_API_KEY"))

# Ollama（本地：无需 API 密钥）
llm = LiteLLM(model="ollama/llama3.2:3b",                api_base="http://localhost:11434")

# DeepSeek
llm = LiteLLM(model="deepseek/deepseek-chat",            api_key=os.getenv("DEEPSEEK_API_KEY"))

# Azure OpenAI
llm = LiteLLM(model="azure/gpt-4o",                      api_key=os.getenv("AZURE_API_KEY"))

# AWS Bedrock
llm = LiteLLM(model="bedrock/anthropic.claude-3-5-sonnet-20241022-v2:0")

# Novita AI
llm = LiteLLM(model="novita/deepseek/deepseek-v3.2",     api_key=os.getenv("NOVITA_API_KEY"))
```

```python HuggingFaceLLM (Local)
from semantica.llms import HuggingFaceLLM

llm = HuggingFaceLLM(
    model="mistralai/Mistral-7B-Instruct-v0.3",
    device="cuda",           # "cpu" | "cuda" | "mps"
    max_new_tokens=512,
    temperature=0.1,
)
# 自带模型：完全本地可控，无需 API 密钥
```

</CodeGroup>

<a id="litellm-100-providers"></a>
## LiteLLM：100+ 提供商

`LiteLLM` 是访问未被 `semantica.llms` 直接导出的任意提供商的推荐方式。使用 `provider/model` 字符串格式：

```python
import os
from semantica.llms import LiteLLM

# 模式：LiteLLM(model="<provider>/<model-name>")
providers = {
    "Anthropic":  LiteLLM(model="anthropic/claude-opus-4-5",       api_key=os.getenv("ANTHROPIC_API_KEY")),
    "Gemini":     LiteLLM(model="gemini/gemini-1.5-pro",            api_key=os.getenv("GOOGLE_API_KEY")),
    "Ollama":     LiteLLM(model="ollama/llama3.2:3b",               api_base="http://localhost:11434"),
    "DeepSeek":   LiteLLM(model="deepseek/deepseek-chat",           api_key=os.getenv("DEEPSEEK_API_KEY")),
    "Azure":      LiteLLM(model="azure/gpt-4o",                     api_key=os.getenv("AZURE_API_KEY")),
    "Bedrock":    LiteLLM(model="bedrock/anthropic.claude-3-5-sonnet-20241022-v2:0"),
    "Cohere":     LiteLLM(model="cohere/command-r-plus",            api_key=os.getenv("COHERE_API_KEY")),
    "Novita AI":  LiteLLM(model="novita/deepseek/deepseek-v3.2",    api_key=os.getenv("NOVITA_API_KEY")),
}

# 每个 LiteLLM 实例都实现相同的 .generate() 接口
response = providers["Anthropic"].generate("Explain GraphRAG in one paragraph.")
```

<Note>
  受支持的 LiteLLM 模型字符串完整列表见 [docs.litellm.ai/docs/providers](https://docs.litellm.ai/docs/providers)。请使用上面所示的 `provider/model` 格式。
</Note>

<a id="custom--enterprise-gateways"></a>
## 自定义 / 企业网关

任意 OpenAI 兼容端点：内部路由层、Qwen 代理或私有 LLaMA 部署：

```python
import os
from semantica.llms import OpenAI

llm = OpenAI(
    model="qwen2.5-72b",
    api_key=os.getenv("GATEWAY_API_KEY"),
    base_url="https://my-internal-gateway.company.com/v1",
)
```

<Note>
  `base_url` 会在构造时校验。非 HTTP(S) 协议会抛出 `ValueError`，以防止 SSRF 攻击（已在 v0.5.0 中修复）。
</Note>

<a id="using-in-extractors"></a>
## 在抽取器中使用

所有抽取器都接受任意提供商作为 `llm_provider=`：

```python
import os
from semantica.semantic_extract import NERExtractor, RelationExtractor, TripletExtractor
from semantica.llms import Groq

llm = Groq(model="llama-3.3-70b-versatile", api_key=os.getenv("GROQ_API_KEY"))

ner  = NERExtractor(method="llm",      llm_provider=llm, max_retries=3)
rel  = RelationExtractor(method="llm", llm_provider=llm)
trip = TripletExtractor(method="llm",  llm_provider=llm)
```

<a id="provider-comparison"></a>
## 提供商对比

| 提供商 | 导入 | 速度 | 成本 | 本地 | 上下文 | 最适用于 |
| :-------- | :------ | :----- | :---- | :----- | :------- | :-------- |
| Groq | `Groq` | 非常快 | 低 | 否 | 128k | 高吞吐抽取 |
| OpenAI | `OpenAI` | 快 | 中 | 否 | 128k | 通用、函数调用 |
| Anthropic | `LiteLLM(model="anthropic/...")` | 快 | 中 | 否 | 200k | 复杂推理、安全 |
| Gemini | `LiteLLM(model="gemini/...")` | 快 | 低 | 否 | 1M | 长上下文、多模态 |
| Ollama | `LiteLLM(model="ollama/...")` | 中 | 免费 | 是 | 视情况 | 隐私、离线 |
| DeepSeek | `LiteLLM(model="deepseek/...")` | 快 | 极低 | 否 | 64k | 代码、分析 |
| Azure OpenAI | `LiteLLM(model="azure/...")` | 快 | 中 | 否 | 128k | 企业、合规 |
| AWS Bedrock | `LiteLLM(model="bedrock/...")` | 快 | 视情况 | 否 | 视情况 | AWS 原生工作负载 |
| HuggingFace | `HuggingFaceLLM` | 慢 | 免费 | 是 | 视情况 | 自定义模型、BYOM |

<Tip>
  对于生产抽取流水线，Groq 提供最佳的吞吐成本比。对于复杂的多跳推理，Claude Opus 或 GPT-4o 提供最高准确率。
</Tip>

<a id="defaults-and-reproducibility"></a>
## 默认值与可复现性

文档示例可能展示更强的模型以提供更好的开发体验，而实现默认值则优先考虑可靠性与成本效率。了解实际默认值有助于获得可复现的结果与一致的基准测试。

**已核实的实现默认值：**

| 提供商 | 默认模型 | 说明 |
| :---------- | :--------------- | :------- |
| `Groq` | `llama-3.1-8b-instant` | 实现默认值；示例使用 `llama-3.3-70b-versatile` 作展示 |
| `OpenAI` | `gpt-3.5-turbo` | 实现默认值；示例使用 `gpt-4o` 作展示 |
| `HuggingFaceLLM` | `gpt2` | 轻量、广泛兼容 |

这些是在不指定 `model=` 构造提供商时所用的模型。本文档中的示例使用更强的展示模型。在生产环境中请始终显式传入 `model=` 以获得可复现的结果。

**为什么这很重要：**
- 在不同环境中获得可复现的抽取结果
- 为基准测试提供一致的基线性能
- 在扩展生产工作负载时成本可预测

<a id="performance-and-reliability-tips"></a>
## 性能与可靠性建议

<a id="extraction-with-retries"></a>
### 带重试的抽取

```python
import os
from semantica.semantic_extract import NERExtractor
from semantica.llms import Groq

llm = Groq(model="llama-3.3-70b-versatile", api_key=os.getenv("GROQ_API_KEY"))
ner = NERExtractor(method="llm", llm_provider=llm, max_retries=3)

# 处理多段文本，自动重试
texts = ["Document 1 text...", "Document 2 text...", "Document 3 text..."]
all_entities = []

for text in texts:
    entities = ner.extract(text)
    all_entities.extend(entities)
    
# 速率限制由提供商自动处理
```

<a id="model-selection-by-use-case"></a>
### 按用例选择模型

| 用例 | 推荐的提供商/模型 | 理由 |
| :---------- | :--------------------------- | :----------- |
| **实体抽取** | `Groq("llama-3.3-70b-versatile")` | 快速，对结构化任务准确度良好 |
| **关系抽取** | `OpenAI("gpt-4o")` | 最擅长复杂关系推理 |
| **复杂分析** | `LiteLLM("anthropic/claude-sonnet-4-20250514")` | 推理能力最强 |
| **高吞吐/低成本** | `LiteLLM("deepseek/deepseek-chat")` | 每 token 成本最低 |

<a id="error-handling"></a>
### 错误处理

```python
import os
from semantica.llms import Groq
from semantica.semantic_extract import NERExtractor

llm = Groq(
    model="llama-3.3-70b-versatile",
    api_key=os.getenv("GROQ_API_KEY")
)

# 针对速率限制与瞬时错误自动重试
extractor = NERExtractor(
    method="llm", 
    llm_provider=llm,
    max_retries=3      # 自动重试失败的请求
)
```

- [语义抽取](semantic_extract.zh-CN.md) —— 使用 LLM 进行 NER 与关系抽取。
- [Agno 集成](../integrations/agno.zh-CN.md) —— 在 Agno 多智能体团队中使用 LLM 提供商。
- [推理](reasoning.zh-CN.md) —— 由 LLM 支撑的演绎与归纳推理。
- [上下文](context.zh-CN.md) —— GraphRAG 使用 LLM 在知识图谱上进行推理。
