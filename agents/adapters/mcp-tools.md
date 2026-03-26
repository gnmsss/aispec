# agents/adapters/mcp-tools.md — 跨平台 MCP 工具方案

## 目的
定义体系中各 Agent 依赖的外部 MCP 工具，提供统一的安装配置指南，确保在任何 AI 平台上都能使用。

## 为什么使用 MCP 而非平台内建能力

| 对比 | 平台内建能力 | MCP 工具 |
|------|------------|---------|
| 可用性 | 仅特定平台支持 | 所有支持 MCP 的平台通用 |
| 一致性 | 各平台实现不同 | 同一个 MCP 在所有平台行为一致 |
| 可控性 | 由平台方控制 | 自行选择、配置、升级 |
| 成本 | 部分免费 | 部分需要 API Key（多有免费额度） |

## MCP 工具清单

本体系涉及 3 类 MCP 工具，按用途分类：

### 1. 网页搜索 MCP — 用于市场调研和信息搜集

推荐方案（二选一）：

#### 方案 A：Brave Search MCP（推荐）
- **仓库**：`brave/brave-search-mcp-server`
- **能力**：网页搜索、新闻搜索、本地搜索、图片搜索、视频搜索、AI 摘要
- **免费额度**：约 1000 次/月
- **需要**：Brave Search API Key（[brave.com/search/api](https://brave.com/search/api)）
- **运行依赖**：Node.js 22+

配置：
```json
{
  "mcpServers": {
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@brave/brave-search-mcp-server"],
      "env": {
        "BRAVE_API_KEY": "<your-api-key>"
      }
    }
  }
}
```

主要工具：
| 工具 | 用途 |
|------|------|
| `brave_web_search` | 通用网页搜索（竞品信息、行业报告） |
| `brave_news_search` | 新闻搜索（行业动态、竞品新闻） |
| `brave_image_search` | 图片搜索（竞品截图、设计参考） |
| `brave_summarizer` | AI 摘要（快速了解搜索结果要点） |

#### 方案 B：Tavily MCP
- **仓库**：官方 `mcp-tavily`（PyPI）
- **能力**：网页搜索、答案搜索、新闻搜索
- **特点**：专为 AI Agent 设计，返回结果已做 AI 友好的内容提取
- **需要**：Tavily API Key（[tavily.com](https://tavily.com)）
- **运行依赖**：Python 3.11+

配置：
```json
{
  "mcpServers": {
    "tavily": {
      "command": "uvx",
      "args": ["mcp-tavily"],
      "env": {
        "TAVILY_API_KEY": "<your-api-key>"
      }
    }
  }
}
```

远程 MCP（无需本地安装）：
```json
{
  "mcpServers": {
    "tavily": {
      "url": "https://mcp.tavily.com/mcp/?tavilyApiKey=<your-api-key>"
    }
  }
}
```

主要工具：
| 工具 | 用途 |
|------|------|
| `tavily_web_search` | 网页搜索（带 AI 内容提取） |
| `tavily_answer_search` | 答案搜索（直接生成答案+证据） |
| `tavily_news_search` | 新闻搜索（带发布日期） |

#### 选择建议
- **Brave Search**：免费额度大、工具种类多（含图片/视频搜索），适合需要全面调研的场景。
- **Tavily**：返回结果更精准（AI 内容提取），Token 消耗更少，适合快速获取答案的场景。

---

### 2. 浏览器自动化 MCP — 用于竞品体验和设计走查

#### Playwright MCP（推荐）
- **仓库**：`microsoft/playwright-mcp`
- **维护方**：Microsoft（Playwright 团队）
- **能力**：网页导航、元素交互、表单填写、截图、无障碍快照
- **特点**：基于无障碍树（非截图），快速轻量，不需要视觉模型
- **免费**：完全免费，无需 API Key
- **运行依赖**：Node.js 18+

配置：
```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

常用配置变体：
```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": [
        "@playwright/mcp@latest",
        "--browser", "chromium",
        "--viewport-size", "1280x720"
      ]
    }
  }
}
```

主要工具：
| 工具 | 用途 |
|------|------|
| `browser_navigate` | 导航到指定 URL（访问竞品网站） |
| `browser_snapshot` | 获取页面无障碍快照（分析页面结构） |
| `browser_click` | 点击页面元素（体验竞品交互） |
| `browser_type` | 在输入框中输入文字 |
| `browser_screenshot` | 页面截图（记录竞品界面） |
| `browser_tab_list` | 列出打开的标签页 |
| `browser_tab_new` | 打开新标签页 |
| `browser_tab_close` | 关闭标签页 |

---

### 3. 设计工具 MCP — 用于 UI 设计原型输出

#### Pencil MCP
- **标识**：`extension-pencil`（Pencil by High Agency）
- **能力**：创建/编辑 `.pen` 设计文件、AI 图片生成、设计变量管理
- **特点**：专为 AI 设计工作流打造，支持组件、变量、主题
- **安装**：通过 VS Code / Cursor 扩展市场安装 Pencil 扩展
- **免费**：扩展本身免费

> Pencil MCP 与 Brave/Playwright 不同，它是通过编辑器扩展提供的 MCP 服务，不是独立的命令行工具。在 Cursor 中安装 Pencil 扩展后自动注册为 MCP 服务。

主要工具见 `agents/design/agent.md` 中的详细列表。

---

## 各 Agent 的 MCP 依赖

| Agent | 必需 MCP | 可选 MCP | 用途 |
|-------|---------|---------|------|
| Product | 搜索 MCP（Brave/Tavily） | Playwright MCP | 竞品调研、市场分析 |
| Design | Pencil MCP | 搜索 MCP + Playwright MCP | UI/UX 设计、竞品界面分析 |
| QA | — | Playwright MCP | 探索性测试、UI 验收 |
| Security | — | — | 安全审计不依赖外部 MCP |
| DevOps | — | — | 不依赖外部 MCP |
| Spec | — | — | 不依赖外部 MCP |
| 各域 Agent | — | — | 不依赖外部 MCP |

## 各平台配置汇总

### Cursor
在 Cursor Settings → MCP 中添加：
```json
{
  "mcpServers": {
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@brave/brave-search-mcp-server"],
      "env": { "BRAVE_API_KEY": "<key>" }
    },
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```
Pencil MCP 通过安装 Pencil 扩展自动启用。

### Claude Code
在 `claude_desktop_config.json` 中添加：
```json
{
  "mcpServers": {
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@brave/brave-search-mcp-server"],
      "env": { "BRAVE_API_KEY": "<key>" }
    },
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

### OpenAI Codex
在 `.codex/config.toml` 中添加：
```toml
[mcp.brave-search]
command = "npx"
args = ["-y", "@brave/brave-search-mcp-server"]

[mcp.brave-search.env]
BRAVE_API_KEY = "<key>"

[mcp.playwright]
command = "npx"
args = ["@playwright/mcp@latest"]
```

### Gemini ADK
```python
from google.adk.tools.mcp_tool import MCPToolset

brave_search = MCPToolset(
    command="npx",
    args=["-y", "@brave/brave-search-mcp-server"],
    env={"BRAVE_API_KEY": "<key>"},
)

playwright = MCPToolset(
    command="npx",
    args=["@playwright/mcp@latest"],
)
```

## MCP 可用性检查与安装引导协议

当 Agent 启动任务时，**必须**先检查所需 MCP 是否可用。检查不通过时，按以下标准流程处理。

### 检查流程

```
Agent 启动任务
  → 检查本任务所需的 MCP 是否已安装可用
    → 已安装 → 正常执行
    → 未安装 → 进入【安装引导流程】
```

### 安装引导流程（必须执行的 3 步）

#### 第 1 步：告知用户缺少什么、为什么需要

**必须**向用户清楚说明：
- 当前任务需要哪个 MCP。
- 这个 MCP 的作用是什么（用一句话说明价值）。
- 如果不安装会有什么影响（功能受限的具体表现）。

话术参考：
```
当前任务需要【{MCP 名称}】来{完成什么能力}。

{MCP 名称}的作用：{一句话说明}。
- 如果安装：{能获得的能力}
- 如果不安装：{受限的部分}，我会改为{退化方案}。
```

各 MCP 的说明内容：

| MCP | 作用说明 | 不安装的影响 |
|-----|---------|------------|
| 搜索 MCP（Brave/Tavily） | 让 AI 能直接搜索互联网，自动调研竞品信息、行业数据和市场趋势 | 无法自动搜索，需要你手动提供竞品信息和市场数据 |
| Playwright MCP | 让 AI 能打开浏览器访问网页，自动浏览竞品网站、截图、分析界面交互 | 无法自动访问网站，需要你手动提供竞品截图和体验记录 |
| Pencil MCP | 让 AI 能直接创建可视化的设计原型（.pen 文件），包含页面布局、组件、配色 | 无法生成可视化设计稿，改为输出文字描述+CSS 变量的设计方案 |

#### 第 2 步：询问用户是否需要安装

**必须**向用户确认，提供明确的选项：

```
请选择：
1. 安装 {MCP 名称}（推荐）— 我可以尝试自动安装
2. 我自己手动安装 — 我会提供安装步骤
3. 不安装，继续任务 — 我会使用退化方案继续
```

#### 第 3 步：根据用户选择执行

**选择 1 — AI 自动安装**：
- 先检查当前环境是否具备自动安装条件（如是否有终端权限、是否有 npm/npx）。
- 如果可以自动安装：执行安装命令，安装后验证是否成功。
- 如果无法自动安装（如没有终端权限、平台限制）：告知用户原因，转为提供手动安装步骤。

各 MCP 的自动安装方式：

| MCP | 自动安装命令 | 前置条件 | 额外配置 |
|-----|------------|---------|---------|
| Brave Search | `npx -y @brave/brave-search-mcp-server` | Node.js 22+ | 需要用户提供 Brave API Key |
| Tavily | `uvx mcp-tavily` 或 `pip install mcp-tavily` | Python 3.11+ | 需要用户提供 Tavily API Key |
| Playwright | `npx @playwright/mcp@latest` | Node.js 18+ | 无需 API Key |
| Pencil | 编辑器扩展市场安装 | Cursor / VS Code | 无法通过命令行安装 |

> 注意：Brave Search 和 Tavily 需要 API Key，自动安装后还需引导用户配置 Key。Pencil 是编辑器扩展，只能由用户在扩展市场中手动安装。

**选择 2 — 用户手动安装**：
- 提供完整的安装步骤（参考本文档"各平台配置汇总"章节）。
- 提供 API Key 的获取方式（如有）。
- 等待用户确认安装完成后再继续任务。

**选择 3 — 不安装，使用退化方案**：
- 明确告知退化后的工作方式。
- 按退化方案继续任务，不再重复询问。

### 退化方案

| MCP 缺失 | 退化方式 | 具体影响 |
|---------|---------|---------|
| 搜索 MCP | Agent 通过提问引导用户手动提供信息 | 竞品分析依赖用户输入，无法自动发现未知竞品 |
| Playwright MCP | Agent 请用户手动截图或描述竞品界面 | 无法自动浏览和截图，竞品体验分析受限 |
| Pencil MCP | Design Agent 输出文本化设计方案 + CSS 变量定义 | 无可视化原型，但设计 Token 和组件规范仍可产出 |

### 安装后验证

自动安装完成后，**必须**验证 MCP 是否正常工作：

| MCP | 验证方式 |
|-----|---------|
| 搜索 MCP | 尝试调用一次搜索工具（如搜索 "test"），确认返回结果 |
| Playwright MCP | 尝试调用 `browser_navigate` 打开一个测试页面 |
| Pencil MCP | 尝试调用 `get_guidelines`（topic: "design-system"），确认返回设计规范 |

验证失败时，向用户报告错误信息，提供排查建议。
