---
title: OpenCode + CC Switch 协同部署：第三方中转站多设备同步方案
tags: [OpenCode, CC-Switch, AI编程, 工具, 配置管理]
created: 2026-06-18
---

# OpenCode + CC Switch 协同部署：第三方中转站多设备同步方案

在使用第三方 AI API 中转站配合 OpenCode 开发时，我们经常面临三个痛点：**OpenCode 的 `/connect` 命令无法填写中转站 Base URL**（只能填供应商 ID 和 API Key），官方建议手动编辑 `opencode.json` 但维护麻烦且容易泄露密钥，以及多台设备间配置难以同步。

**CC Switch** 是一款 Claude/Codex 多账号管理工具，通过 GUI 配置界面 + 云端同步机制，完美解决了上述问题。它既能作为 OpenCode 的配置中枢，也能为 Codex、Claude Code 等其他 AI 工具提供统一的配置管理。

CC Switch 的核心优势：

- **GUI 配置**：可视化填写 Base URL、API Key 等字段，告别手动编辑 JSON。
- **多设备同步**：通过 Cloudflare R2 等 S3 兼容存储，实时同步供应商配置、Skills、MCP 服务器。
- **多工具复用**：一份配置同时供 OpenCode、Codex、Claude Code 使用，避免重复维护。
- **Skill 市场**：一键下载和更新社区 Skills，自动同步到所有设备。

本文将从工具定位、安装配置、中转站接入、多设备同步等方面，帮助你快速搭建基于第三方 API 的多设备 AI 编程环境。

---

## 目录

- [1. 工具定位](#1-工具定位)
- [2. 安装](#2-安装)
  - [2.1 OpenCode](#21-opencode)
  - [2.2 CC Switch](#22-cc-switch)
- [3. 配置第三方中转站](#3-配置第三方中转站)
  - [3.1 进入添加供应商界面](#31-进入添加供应商界面)
  - [3.2 字段填写规范](#32-字段填写规范)
  - [3.3 关键坑点：供应商标识必须用内置标准名](#33-关键坑点供应商标识必须用内置标准名)
  - [3.4 切换并验证](#34-切换并验证)
- [4. 多设备同步（Cloudflare R2）](#4-多设备同步cloudflare-r2)
  - [4.1 R2 准备](#41-r2-准备)
  - [4.2 CC Switch 配置同步](#42-cc-switch-配置同步)
  - [4.3 同步范围](#43-同步范围)
  - [4.4 多设备工作流](#44-多设备工作流)
- [5. Skill 市场](#5-skill-市场)
  - [5.1 发现技能](#51-发现技能)
  - [5.2 检查更新](#52-检查更新)
  - [5.3 多工具复用](#53-多工具复用)
- [6. 注意事项](#6-注意事项)
- [7. 参考资料](#7-参考资料)

---

## 1. 工具定位

### OpenCode

OpenCode 是一款开源的 CLI AI 编程 Agent，运行在终端中，支持多种 LLM 提供商。通过 `/init` 分析项目结构后，可以回答代码问题、添加功能、重构代码等。

### CC Switch

CC Switch 是 Claude Code 和 Codex 的多账号管理工具，提供可视化配置界面。它不仅能管理多个 API 账号，还能通过云端同步配置到多台设备，并支持 Skill 市场一键安装。

### 协作关系

CC Switch 作为配置中枢，将供应商配置写入本地配置文件（如 `~/.opencode/` 下的配置）。OpenCode 启动时读取这些配置，自动识别当前激活的供应商和模型。用户在 CC Switch 中切换账号后，OpenCode 下次启动即自动生效，无需手动编辑配置文件。

---

## 2. 安装

### 2.1 OpenCode

使用 npm 全局安装：

```powershell
npm install -g opencode-ai
```

验证安装：

```powershell
opencode --version  # 输出：1.17.8 或更高版本
```

### 2.2 CC Switch

从 GitHub Releases 下载 Windows 安装包：

**推荐直接下载 MSI**：
```
https://github.com/farion1231/cc-switch/releases/download/v3.16.3/CC-Switch-v3.16.3-Windows.msi
```

双击安装后，应用会自动启动并显示在系统托盘（默认开机自启）。

**仓库地址**：https://github.com/farion1231/cc-switch

---

## 3. 配置第三方中转站

### 3.1 进入添加供应商界面

打开 CC Switch 主窗口，点击右上角橙色 **+** 按钮，在弹出的工具图标中选择 **OpenCode** 图标（蓝色方块），进入"编辑供应商"界面。

### 3.2 字段填写规范

| 字段 | 用途 | 示例 |
|------|------|------|
| **供应商标识** * | models.dev 查表键，必须用内置标准名 | `anthropic` |
| **供应商名称** | GUI 显示用，可自定义 | `ergouzi-claude` |
| **备注** | 可选说明 | 公司专用账号 |
| **官网链接** | 中转站后台地址 | `https://ergouzi.life/console/token` |
| **接口格式** | 按中转站文档选择 | `OpenAI Compatible` |
| **API Key** | 中转站发放的密钥 | `sk-J4oTRl77K6fXVlVs...` |
| **Base URL** | 中转站 API 端点地址 | `https://ergouzi.life/v1` |
| **额外选项** | SDK 扩展参数（可选） | `setCacheKey: true` |
| **模型配置** | 可用模型列表 | `claude-opus-4-7` |

**示例配置**：

- 供应商标识：`anthropic`（⚠️ 必须用标准名）
- 供应商名称：`ergouzi-claude`（显示名，随意起）
- Base URL：`https://ergouzi.life/v1`
- API Key：`sk-xxx`
- 模型 ID：`claude-opus-4-7`、`claude-opus-4-8`

填写完成后点击右下角"保存"按钮。

---

### 3.3 关键坑点：供应商标识必须用内置标准名

**原理**：OpenCode 启动后，通过 `供应商标识 + 模型 ID` 在 models.dev 查询模型能力清单（如 `attachment`、`modalities.input/output` 等）。查到后自动注入，否则退化为纯文本模式，发图片等多模态功能失效。

**规则**：供应商标识必须写成 OpenCode 内置的标准名，常见的有：

- `anthropic` - Claude 系列
- `openai` - GPT 系列
- `google` - Gemini 系列
- `deepseek` - DeepSeek 系列
- `zhipu` - 智谱 GLM 系列

**反例**：使用中转站分组名作为供应商标识

如果填写 `ergouzi-0.05折-claude组`，models.dev 无法识别该标识，OpenCode 将无法获取模型能力，导致：
- 发送图片时识别失败
- PDF 附件功能不可用
- 多模态输入/输出能力丢失

**正确做法**：

- **供应商标识**：`anthropic`（标准名，给 OpenCode 查表用）
- **供应商名称**：`ergouzi-claude`（GUI 显示用，随便起）
- **Base URL**：`https://ergouzi.life/v1`（中转站真实地址）
- **模型 ID**：`claude-opus-4-7`（标准模型名）

这样 OpenCode 会：
1. 用 `anthropic` 在 models.dev 查到 Claude 的能力清单
2. 自动注入 `attachment: true` 和 `modalities`
3. 实际请求发往 `https://ergouzi.life/v1`（中转站）

**例外情况**：如果确实需要用非标准标识（如自建 API），则必须手动在"配置 JSON"区域添加 models 属性：

```json
"models": {
  "claude-opus-4-7": {
    "name": "claude-opus-4-7",
    "attachment": true,
    "modalities": {
      "input": ["text", "image", "pdf"],
      "output": ["text"]
    }
  }
}
```

但这种方式维护成本高，每个模型都要手动写，且容易出错。**强烈推荐用标准供应商标识 + Base URL 的方式**，让 OpenCode 自动处理能力注入。

> **提示**：访问 [models.dev](https://models.dev) 查询完整的内置供应商标识列表与对应的模型能力清单。

---

### 3.4 切换并验证

回到 CC Switch 主界面，点击刚刚添加的供应商卡片（如 `ergouzi-claude`），被选中的卡片会高亮显示或显示"移除"按钮。

启动 OpenCode 验证：

```powershell
cd /path/to/project
opencode
```

在 OpenCode TUI 中检查：
1. 右上角是否显示正确的模型名（如 `claude-opus-4-7`）
2. 发送一张图片测试多模态识别是否正常

如果图片识别失败，检查供应商标识是否使用了标准名。

---

## 4. 多设备同步（Cloudflare R2）

### 4.1 R2 准备

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 左侧菜单选择 **R2** → 点击"创建存储桶"
3. 填写存储桶名称（如 `my-cc-switch-sync`），选择位置后创建
4. 进入存储桶 → **设置** → **R2 API 令牌** → 创建 API 令牌
5. 记录以下信息：
   - **Endpoint**：`https://xxx.r2.cloudflarestorage.com`
   - **Access Key ID**：`8c062dea722ca5cb...`
   - **Secret Access Key**：`9fa3807aa24a332d...`
6. 确保存储桶设为**私有**（保护 API Key 不泄露）

---

### 4.2 CC Switch 配置同步

打开 CC Switch，点击左上角齿轮图标 → **高级** → **云同步** → **同步方式** → 选择 **S3 兼容存储**。

填写以下字段：

- **Endpoint**：R2 提供的端点地址（如 `https://xxx.r2.cloudflarestorage.com`）
- **Bucket**：存储桶名称（如 `my-cc-switch-sync`）
- **Region**：填 `auto`
- **Access Key ID**：R2 API 令牌的 Access Key
- **Secret Access Key**：R2 API 令牌的 Secret Key
- **远程根目录**：`cc-switch-sync`（可选，用于隔离数据）

保存后，点击"立即同步"测试连接。

---

### 4.3 同步范围

CC Switch 通过云同步，会自动同步以下内容：

- **供应商配置**：所有添加的 Provider（包括 Base URL、API Key 等）
- **Skills**：已安装的 Skills 及其配置
- **MCP 服务器**：Model Context Protocol 服务器配置

同步后，这些配置会被多个工具共享使用：
- OpenCode
- Codex
- Claude Code
- Gemini
- Hermes

---

### 4.4 多设备工作流

**首次设置**：

1. 在公司电脑上配置好 CC Switch（供应商、R2 同步）
2. 打开云同步功能，确保首次同步成功
3. 在家里电脑上安装 CC Switch
4. 配置相同的 R2 凭证（Endpoint、Bucket、Access Key）
5. 点击"立即同步"拉取公司电脑的配置

**日常使用**：

- 任意一台设备修改配置（添加供应商、安装 Skill）
- 配置自动/手动同步到 R2
- 其他设备打开 CC Switch 时自动拉取最新配置
- 或手动点击"立即同步"强制更新

**冲突处理**：

云同步以最后写入端为准。如果两台设备同时修改配置，后同步的会覆盖先同步的。建议：
- 重要修改后立即同步
- 避免在多台设备同时修改配置

---

## 5. Skill 市场

### 5.1 发现技能

CC Switch 内置 Skill 市场，连接社区 Skill 仓库（如 skills.sh、Vercel Labs 等）。

**操作步骤**：

1. 主界面右上角点击小扳手图标（工具图标）
2. 选择 **Skills 管理**
3. 点击右上角 **发现技能** 按钮
4. 选择仓库源（如 `skills.sh`）
5. 搜索需要的 Skill（如 `find-skills`、`karpathy-guidelines`）
6. 点击"安装"按钮，自动下载并配置

---

### 5.2 检查更新

已安装的 Skills 可以统一检查更新：

1. **Skills 管理** 界面右上角点击 **检查更新**
2. 系统自动比对本地与远程版本
3. 有更新的 Skill 会显示"更新"按钮
4. 点击一键升级到最新版

---

### 5.3 多工具复用

已安装的 Skill 会在 **Skills 管理** 界面显示对应的工具图标：

- Claude（红色图标）
- Codex（黑色图标）
- Gemini（彩色图标）
- OpenCode（蓝色图标）
- Hermes（紫色图标）

这表示该 Skill 可以被对应的工具识别和使用。通过 R2 云同步，Skills 会自动同步到所有设备，无需在每台机器上重复安装。

---

## 6. 注意事项

### 供应商标识不可改

添加供应商后，**供应商标识**字段会变为只读（灰色）。如果填错，只能删除重建，因此命名时务必确认使用标准名。

### 供应商标识必须 models.dev 收录

这是整篇笔记最关键的坑点。如果使用非标准标识，OpenCode 会退化为纯文本模式，多模态功能（图片、PDF、语音）全部失效。详见 [§3.3](#33-关键坑点供应商标识必须用内置标准名)。

### R2 Bucket 必须私有

API Key 通过云同步存储在 R2 中。如果 Bucket 设为公开，等同于将 API Key 公开泄露。务必确保 Bucket 权限设为**私有**。

### 同步冲突以最后写入为准

多设备同时修改配置时，后同步的会覆盖先同步的。重要修改后立即同步，避免被其他设备覆盖。

### 切换供应商后需重启 OpenCode

在 CC Switch 中切换供应商或更新 Skills 后，已运行的 OpenCode 不会立即生效。需要退出 OpenCode（`Ctrl+C` 或 `/exit`）并重新启动。

---

## 7. 参考资料

- [OpenCode 官方文档](https://opencode.ai/docs)
- [CC Switch GitHub 仓库](https://github.com/farion1231/cc-switch)
- [models.dev](https://models.dev) - 查询合法供应商标识与模型能力清单
- [Cloudflare R2 文档](https://developers.cloudflare.com/r2/)
