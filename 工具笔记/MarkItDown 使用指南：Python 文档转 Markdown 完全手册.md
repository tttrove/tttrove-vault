---
title: MarkItDown 使用指南：Python 文档转 Markdown 完全手册
tags: [Python, 工具, Markdown, 文档转换]
created: 2026-06-18
---

# MarkItDown 使用指南：Python 文档转 Markdown 完全手册

在处理各类文档（PDF、Word、Excel、PowerPoint）时，我们经常需要将其转换为纯文本或 Markdown 格式，以便进行文本分析、索引建立或 LLM 处理。**传统工具要么转换质量低，要么无法保留文档结构**（标题、列表、表格等）。

**MarkItDown** 是微软开源的轻量级 Python 工具，专为 LLM 和文本分析场景设计，能将多种文档格式转换为结构化 Markdown，保留标题层级、表格、链接等关键信息。

MarkItDown 的核心优势：

- **多格式支持**：PDF、DOCX、PPTX、XLSX、图片、音频、HTML、EPUB 等 10+ 种格式。
- **结构保留**：转换后的 Markdown 保留原文档的标题、列表、表格、链接等结构。
- **LLM 友好**：输出格式接近 LLM 训练数据格式，token 效率高。
- **零配置安装**：通过 `uv` 工具链一键安装，自动管理 Python 版本和依赖。
- **CLI + API**：既可命令行批量转换，也可 Python 脚本集成。

本文将从环境准备、安装配置、基础用法、高级功能、实战场景及故障排除等方面，帮助你快速掌握 MarkItDown 的使用。

---

## 目录

- [1. 引言：什么是 MarkItDown？](#1-引言什么是-markitdown)
- [2. 环境要求与安装](#2-环境要求与安装)
  - [2.1 Python 版本要求](#21-python-版本要求)
  - [2.2 使用 uv 安装（推荐）](#22-使用-uv-安装推荐)
  - [2.3 使用 pip 安装（传统方式）](#23-使用-pip-安装传统方式)
- [3. 基础用法](#3-基础用法)
  - [3.1 CLI 命令行转换](#31-cli-命令行转换)
  - [3.2 Python API 调用](#32-python-api-调用)
- [4. 可选功能与依赖](#4-可选功能与依赖)
- [5. 常见场景与最佳实践](#5-常见场景与最佳实践)
  - [5.1 常见使用场景](#51-常见使用场景)
  - [5.2 最佳实践](#52-最佳实践)
- [6. 高级功能](#6-高级功能)
  - [6.1 使用 LLM 生成图片描述](#61-使用-llm-生成图片描述)
  - [6.2 Azure Document Intelligence 集成](#62-azure-document-intelligence-集成)
  - [6.3 插件系统](#63-插件系统)
- [7. 故障排除](#7-故障排除)
- [8. 长期维护](#8-长期维护)
- [9. 总结](#9-总结)
- [10. 参考资料](#10-参考资料)

---

## 1. 引言：什么是 MarkItDown？

MarkItDown 是由微软 AutoGen 团队开发的文档转换工具，专注于将各类文档转换为 Markdown 格式。与 `textract` 等纯文本提取工具不同，MarkItDown 强调**保留文档结构**（标题、列表、表格、链接），输出的 Markdown 既适合人类阅读，更适合作为 LLM 的输入。

**核心设计理念**：
- Markdown 是最接近纯文本的结构化标记语言，LLM（如 GPT-4）天然理解 Markdown 语法。
- 主流 LLM 训练数据中包含大量 Markdown 格式文档，对其支持更好。
- Token 效率高，同样内容的 Markdown 比 HTML 或 JSON 更节省 token。

---

## 2. 环境要求与安装

### 2.1 Python 版本要求

MarkItDown 要求：
- **最低版本**：Python 3.10
- **官方支持**：3.10、3.11、3.12、3.13
- **实现**：CPython 和 PyPy 均可

检查当前 Python 版本：

```powershell
python --version  # 输出如：Python 3.14.5
```

---

### 2.2 使用 uv 安装（推荐）

`uv` 是现代 Python 包管理工具，速度快、自动管理 Python 版本、完全隔离环境。

#### 步骤 1：安装 uv

```powershell
# PowerShell 7（Windows）
irm https://astral.sh/uv/install.ps1 | iex

# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
```

安装后 `uv` 会自动添加到 PATH，重启终端生效。

#### 步骤 2：使用 uv tool 安装 markitdown

```powershell
# 安装常用格式支持（PDF、Word、Excel、PowerPoint）
uv tool install --python 3.12 "markitdown[pdf,docx,pptx,xlsx]"

# 或安装全部功能（包括音频转录、YouTube 字幕等）
uv tool install --python 3.12 "markitdown[all]"
```

**说明**：
- `--python 3.12` 指定使用 Python 3.12（uv 会自动下载，与系统 Python 隔离）
- `[pdf,docx,pptx,xlsx]` 是可选依赖组，按需选择
- 安装后 `markitdown` 命令全局可用

#### 步骤 3：验证安装

```powershell
markitdown --version  # 输出：markitdown 0.1.6
uv tool list          # 查看已安装工具
```

**路径说明**（Windows）：
- `markitdown.exe` → `C:\Users\<用户名>\.local\bin\markitdown.exe`
- Python 3.12 → `%LOCALAPPDATA%\uv\python\cpython-3.12.x\`
- 虚拟环境 → `%LOCALAPPDATA%\uv\tools\markitdown\`

---

### 2.3 使用 pip 安装（传统方式）

如果不想使用 uv，可以直接用 pip：

```bash
# 安装到全局（不推荐，可能污染系统 Python）
pip install "markitdown[pdf,docx,pptx,xlsx]"

# 推荐：先创建虚拟环境
python -m venv .venv
source .venv/bin/activate  # Windows 用 .venv\Scripts\Activate.ps1
pip install "markitdown[pdf,docx,pptx,xlsx]"
```

---

## 3. 基础用法

### 3.1 CLI 命令行转换

#### 单文件转换

```powershell
# 输出到标准输出
markitdown 报告.pdf

# 重定向到文件
markitdown 报告.pdf > 报告.md

# 使用 -o 参数指定输出文件
markitdown 报告.pdf -o 报告.md
```

#### 管道输入

```powershell
# 从标准输入读取
cat 文件.docx | markitdown > 输出.md

# PowerShell 写法
Get-Content 文件.docx -Raw | markitdown
```

#### 批量转换

```powershell
# 转换当前目录所有 PDF
Get-ChildItem *.pdf | ForEach-Object {
    markitdown $_.FullName -o "$($_.BaseName).md"
}

# 转换多种格式
Get-ChildItem *.pdf,*.docx,*.pptx | ForEach-Object {
    $outFile = [System.IO.Path]::ChangeExtension($_.Name, ".md")
    markitdown $_.FullName -o $outFile
}
```

---

### 3.2 Python API 调用

适合需要后处理（清洗、入库、嵌入向量）的场景。

#### 基础用法

```python
from markitdown import MarkItDown

# 初始化转换器
md = MarkItDown()

# 转换文件
result = md.convert("报告.pdf")
print(result.text_content)  # 获取 Markdown 文本

# 保存到文件
with open("报告.md", "w", encoding="utf-8") as f:
    f.write(result.text_content)
```

#### 批量转换目录

```python
from pathlib import Path
from markitdown import MarkItDown

md = MarkItDown()
input_dir = Path("./documents")
output_dir = Path("./markdown")
output_dir.mkdir(exist_ok=True)

for file in input_dir.glob("*.{pdf,docx,pptx,xlsx}"):
    try:
        result = md.convert(str(file))
        out_file = output_dir / f"{file.stem}.md"
        out_file.write_text(result.text_content, encoding="utf-8")
        print(f"✓ 转换完成: {file.name}")
    except Exception as e:
        print(f"✗ 转换失败 {file.name}: {e}")
```

---

## 4. 可选功能与依赖

MarkItDown 默认只包含基础功能（HTML、CSV、JSON、XML、EPUB、ZIP），其他格式需要安装可选依赖。

| 可选依赖 | 支持格式 | 安装方式 |
|---------|---------|---------|
| `[pdf]` | PDF 文档 | `uv tool install "markitdown[pdf]"` |
| `[docx]` | Word 文档（.docx） | `uv tool install "markitdown[docx]"` |
| `[pptx]` | PowerPoint（.pptx） | `uv tool install "markitdown[pptx]"` |
| `[xlsx]` | Excel 2007+（.xlsx） | `uv tool install "markitdown[xlsx]"` |
| `[xls]` | Excel 97-2003（.xls） | `uv tool install "markitdown[xls]"` |
| `[audio-transcription]` | WAV、MP3 音频转录 | `uv tool install "markitdown[audio-transcription]"` |
| `[youtube-transcription]` | YouTube 视频字幕 | `uv tool install "markitdown[youtube-transcription]"` |
| `[az-doc-intel]` | Azure Document Intelligence | `uv tool install "markitdown[az-doc-intel]"` |
| `[all]` | 所有功能 | `uv tool install "markitdown[all]"` |

**按需组合安装**：

```powershell
# 只装 Office 三件套 + PDF
uv tool install "markitdown[pdf,docx,pptx,xlsx]"

# 只装 PDF 和音频
uv tool install "markitdown[pdf,audio-transcription]"
```

---

## 5. 常见场景与最佳实践

### 5.1 常见使用场景

#### 场景 1：为 LLM 准备训练数据

将技术文档、手册、论文批量转换为 Markdown，作为 RAG（检索增强生成）系统的知识库。

```python
from markitdown import MarkItDown
import chromadb

md = MarkItDown()
client = chromadb.Client()
collection = client.create_collection("docs")

for pdf in Path("./papers").glob("*.pdf"):
    result = md.convert(str(pdf))
    collection.add(
        documents=[result.text_content],
        ids=[pdf.stem]
    )
```

#### 场景 2：将 Word 文档发布为静态博客

```powershell
# 批量转换 Word 文档到 Hugo/Jekyll 的 content 目录
Get-ChildItem ./drafts/*.docx | ForEach-Object {
    $frontmatter = "---`ntitle: $($_.BaseName)`ndate: $(Get-Date -Format 'yyyy-MM-dd')`n---`n`n"
    $markdown = markitdown $_.FullName
    $frontmatter + $markdown | Out-File "./content/posts/$($_.BaseName).md" -Encoding utf8
}
```

#### 场景 3：Excel 数据表转 Markdown 表格

适合将数据分析结果快速生成可读报告。

```python
result = md.convert("销售数据.xlsx")
# 输出包含 Markdown 表格格式的文本
```

#### 场景 4：提取 PDF 合同中的文本用于审查

```python
from markitdown import MarkItDown

md = MarkItDown()
contract = md.convert("合同.pdf")

# 检查关键条款
if "违约责任" in contract.text_content:
    print("发现违约责任条款")
```

---

### 5.2 最佳实践

#### 1. 优先使用 uv tool 管理

与 pip 全局安装相比，`uv tool` 完全隔离环境，升级和卸载更干净，不会污染系统 Python。

```powershell
# 升级
uv tool upgrade markitdown

# 卸载
uv tool uninstall markitdown
```

#### 2. 按需安装可选依赖

`[all]` 会安装 40+ 依赖包（包括 `pydub`、`SpeechRecognition`、`onnxruntime` 等），如果只转换 Office 文档，用精简组合：

```powershell
uv tool install "markitdown[pdf,docx,pptx,xlsx]"
```

#### 3. 处理大文件时使用 Python API

CLI 模式会将整个文件加载到内存，处理大型 PDF（100+ 页）时，用 Python API 可以流式处理或分页转换。

#### 4. 对扫描版 PDF 使用 Azure Document Intelligence

内置 PDF 解析器（pdfminer-six）无法处理扫描版或图片型 PDF，需要 OCR。推荐使用 Azure Document Intelligence 或插件：

```python
md = MarkItDown(docintel_endpoint="<azure_endpoint>")
result = md.convert("扫描版.pdf")
```

#### 5. 版本锁定

在生产环境或脚本中锁定版本，避免破坏性更新：

```bash
pip install "markitdown[pdf,docx,pptx,xlsx]==0.1.6"
```

---

## 6. 高级功能

### 6.1 使用 LLM 生成图片描述

默认情况下，MarkItDown 遇到图片只会输出 `![image](path)`。通过集成 OpenAI API，可以自动生成图片描述。

```python
from markitdown import MarkItDown
from openai import OpenAI

client = OpenAI(api_key="your-api-key")

md = MarkItDown(
    llm_client=client,
    llm_model="gpt-4o",
    llm_prompt="请用一句话描述这张图片的内容"  # 可选自定义提示词
)

result = md.convert("演示文稿.pptx")  # 图片会被替换为描述文字
print(result.text_content)
```

**适用场景**：
- PowerPoint 包含大量图表和截图
- 图片型 PDF 需要文字化
- 为视障用户生成无障碍文档

---

### 6.2 Azure Document Intelligence 集成

Azure Document Intelligence（原 Form Recognizer）提供高质量的 OCR 和版面分析，适合复杂 PDF。

#### CLI 用法

```powershell
markitdown 扫描版.pdf -d -e "<document_intelligence_endpoint>" -o 输出.md
```

#### Python API 用法

```python
md = MarkItDown(docintel_endpoint="<azure_endpoint>")
result = md.convert("复杂表格.pdf")
```

**注意**：
- 每次调用会产生 Azure API 费用
- 需要先创建 Azure Document Intelligence 资源
- [创建资源文档](https://learn.microsoft.com/azure/ai-services/document-intelligence/)

---

### 6.3 插件系统

MarkItDown 支持第三方插件扩展功能（默认禁用）。

#### 查看已安装插件

```powershell
markitdown --list-plugins
```

#### 启用插件

```powershell
markitdown --use-plugins 文件.pdf
```

#### 推荐插件：markitdown-ocr

为 PDF、DOCX、PPTX 中的嵌入图片添加 OCR 支持（使用 LLM Vision）。

```bash
pip install markitdown-ocr
```

```python
from markitdown import MarkItDown
from openai import OpenAI

md = MarkItDown(
    enable_plugins=True,
    llm_client=OpenAI(),
    llm_model="gpt-4o"
)

result = md.convert("包含图片的文档.pdf")  # 自动提取图片中的文字
```

---

## 7. 故障排除

### 7.1 命令找不到：`markitdown: command not found`

**原因**：PATH 未包含安装目录。

**解决**：

```powershell
# 检查 PATH
$env:PATH -split ';' | Select-String ".local"

# 如果没有，手动添加（PowerShell）
$env:PATH += ";$env:USERPROFILE\.local\bin"

# 永久生效：重启终端或刷新环境变量
refreshenv  # 需要 chocolatey
```

---

### 7.2 PDF 转换乱码或缺失内容

**原因**：
- PDF 使用了非标准字体编码
- 扫描版 PDF 无文字层

**解决**：

```python
# 方法 1：使用 Azure Document Intelligence
md = MarkItDown(docintel_endpoint="<endpoint>")

# 方法 2：使用 OCR 插件
pip install markitdown-ocr
md = MarkItDown(enable_plugins=True, llm_client=OpenAI(), llm_model="gpt-4o")
```

---

### 7.3 Excel 转换丢失公式计算结果

**原因**：MarkItDown 只提取单元格的**显示值**，不执行公式。

**解决**：在 Excel 中先"复制→粘贴为值"，再转换。或使用 `pandas` 读取后计算：

```python
import pandas as pd

df = pd.read_excel("数据.xlsx")
df['总计'] = df['数量'] * df['单价']  # 手动计算
df.to_markdown("输出.md")
```

---

### 7.4 音频转录失败：`ffmpeg not found`

**原因**：音频转录功能依赖 `ffmpeg`。

**解决**：

```powershell
# Windows（chocolatey）
choco install ffmpeg

# macOS
brew install ffmpeg

# Linux（Debian/Ubuntu）
sudo apt install ffmpeg
```

---

## 8. 长期维护

### 查看已安装工具

```powershell
uv tool list
```

### 升级到最新版

```powershell
uv tool upgrade markitdown
```

### 查看可用 Python 版本

```powershell
uv python list
```

### 卸载工具

```powershell
uv tool uninstall markitdown
```

### 清理未使用的 Python 版本

```powershell
uv python uninstall 3.11
```

---

## 9. 总结

MarkItDown 是一款轻量、高效的文档转 Markdown 工具，通过保留文档结构、优化 LLM token 效率，成为文本分析和知识库构建的理想选择。其多格式支持（PDF、Office、图片、音频）和灵活的 CLI + API 双模式，使其适用于批量处理、脚本集成、RAG 系统等多种场景。

掌握 MarkItDown 后，你可以：

- 快速将技术文档、论文、报告转换为 Markdown，用于知识库或 RAG 系统。
- 批量处理 Office 文档，发布为静态博客或文档站点。
- 结合 LLM API 自动生成图片描述，提升文档无障碍性。
- 通过 Azure Document Intelligence 处理扫描版 PDF 和复杂表格。

建议从基础 CLI 转换开始实践，逐步探索 Python API 集成和 LLM 增强功能，结合最佳实践持续优化工作流。

---

## 10. 参考资料

- [MarkItDown GitHub 仓库](https://github.com/microsoft/markitdown)
- [MarkItDown PyPI 页面](https://pypi.org/project/markitdown/)
- [uv 工具文档](https://docs.astral.sh/uv/)
- [Azure Document Intelligence 文档](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
- [OpenAI Vision API 文档](https://platform.openai.com/docs/guides/vision)
