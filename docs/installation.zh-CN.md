---
title: "安装"
description: "一分钟内完成 Semantica 安装。"
icon: "download"
---

**[English](installation.md)** · **简体中文（当前）**

<Check>
  **PyPI 已提供**：`pip install semantica` 即可开始使用。
</Check>

<Note>
  需要 Python 3.8 或更高版本。推荐使用 Python 3.11+。
</Note>

## 系统要求

| 组件 | 最低 | 推荐 |
| :--------- | :------- | :----------- |
| Python | 3.8 | 3.11+ |
| 操作系统 | Windows / Linux / Mac | Linux / Mac |
| 内存 | 4 GB | 16 GB+ |
| 存储 | 2 GB | 20 GB+（模型和数据） |


## 基础安装

```bash
pip install semantica
```

安装全部可选依赖：

```bash
pip install semantica[all]
```

### 验证安装

```bash
python -c "import semantica; print(semantica.__version__)"
```


## 虚拟环境（推荐）

<Tabs>
  <Tab title="venv">
    ```bash
    python -m venv venv
    source venv/bin/activate   # Linux / Mac
    venv\Scripts\activate      # Windows
    pip install semantica
    ```
  </Tab>
  <Tab title="conda">
    ```bash
    conda create -n semantica python=3.11
    conda activate semantica
    pip install semantica
    ```
  </Tab>
</Tabs>


## 可选依赖

按需安装：

<Tabs>
  <Tab title="GPU">
    ```bash
    pip install semantica[gpu]
    ```
    包含带 CUDA 的 PyTorch、FAISS GPU 和 CuPy。
  </Tab>
  <Tab title="可视化">
    ```bash
    pip install semantica[viz]
    ```
    包含 PyVis、Graphviz 和 UMAP。
  </Tab>
  <Tab title="LLM 提供商">
    ```bash
    pip install semantica[llm-all]       # 所有提供商
    pip install semantica[llm-openai]    # OpenAI
    pip install semantica[llm-anthropic] # Anthropic
    pip install semantica[llm-gemini]    # Google Gemini
    pip install semantica[llm-groq]      # Groq
    pip install semantica[llm-ollama]    # Ollama（本地）
    ```
  </Tab>
  <Tab title="云端">
    ```bash
    pip install semantica[cloud]
    ```
    包含 AWS S3、Azure Blob 和 Google Cloud Storage。
  </Tab>
</Tabs>


## 从源码安装

如需安装最新开发版本或参与贡献：

```bash
git clone https://github.com/semantica-agi/semantica.git
cd semantica

pip install -e .         # 仅核心
pip install -e ".[all]"  # 全部附加依赖
pip install -e ".[dev]"  # 开发工具（pytest、black 等）
```

如果 PyPI 发布版本存在问题，可直接从 main 分支安装：

```bash
pip install git+https://github.com/semantica-agi/semantica.git@main
```


## 故障排查

<AccordionGroup>

<Accordion title="ModuleNotFoundError: No module named 'semantica'" icon="circle-xmark">

请确认你处于正确的虚拟环境中：

```bash
pip list | grep semantica
pip install --upgrade semantica
```

</Accordion>

<Accordion title="安装因依赖错误失败" icon="triangle-exclamation">

```bash
pip install --upgrade pip
pip install build wheel
pip install semantica --no-deps  # 先安装核心，再添加附加依赖
```

</Accordion>

<Accordion title="GPU 依赖安装失败" icon="bolt">

先安装 CPU 版本，再叠加 GPU 支持：

```bash
pip install semantica
pip install semantica[gpu]
```

</Accordion>

<Accordion title="权限被拒绝" icon="lock">

```bash
pip install --user semantica  # 或者使用虚拟环境
```

</Accordion>

<Accordion title="Windows 上 [all] 安装失败" icon="windows">

已在 **v0.5.0** 中修复。请升级到最新版本：

```bash
pip install --upgrade semantica
```

</Accordion>

<Accordion title="Windows 启动时 PyTorch DLL 错误" icon="windows">

请安装 [Microsoft Visual C++ Redistributable](https://aka.ms/vs/17/release/vc_redist.x64.exe)。这是 Windows 系统依赖，并非 Semantica 的 bug。

</Accordion>

</AccordionGroup>


## 后续步骤

- [入门](getting-started.zh-CN.md) — 在构建之前先了解 Semantica 能做什么。
- [快速上手](quickstart.zh-CN.md) — 跟随代码走完端到端工作流。
- [浏览示例](cookbook.zh-CN.md) — 按用例查看 notebook 示例。
