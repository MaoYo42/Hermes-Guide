# Firecrawl MCP 集成指南

## 概述

Firecrawl MCP Server 是 Firecrawl 官方的 MCP（Model Context Protocol）服务端实现，为 AI Agent 提供网页抓取、搜索、交互和深度研究能力。通过本指南，你可以为 Hermes Agent 或其他 MCP 兼容客户端配置 Firecrawl。

## 前置条件

- Node.js 18+（用于 npx 执行）
- Firecrawl API Key（免费获取：https://www.firecrawl.dev/app/api-keys）

## 快速安装（npx）

最简单的运行方式，无需安装：

```bash
env FIRECRAWL_API_KEY=fc-YOUR_API_KEY npx -y firecrawl-mcp
```

## 全局安装

```bash
npm install -g firecrawl-mcp
firecrawl-mcp  # 直接运行
```

## 环境变量配置

### 必要变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `FIRECRAWL_API_KEY` | Firecrawl API 密钥 | `fc-xxxxx` |
| `FIRECRAWL_API_URL` | 自托管实例地址（可选） | `https://firecrawl.your-domain.com` |

### 可选变量（重试策略）

```bash
export FIRECRAWL_RETRY_MAX_ATTEMPTS=5        # 最大重试次数（默认 3）
export FIRECRAWL_RETRY_INITIAL_DELAY=2000    # 初始延迟毫秒（默认 1000）
export FIRECRAWL_RETRY_MAX_DELAY=30000       # 最大延迟毫秒（默认 10000）
export FIRECRAWL_RETRY_BACKOFF_FACTOR=3      # 退避因子（默认 2）
```

### 可选变量（信用监控）

```bash
export FIRECRAWL_CREDIT_WARNING_THRESHOLD=2000    # 警告阈值（默认 1000）
export FIRECRAWL_CREDIT_CRITICAL_THRESHOLD=500    # 临界阈值（默认 100）
```

## Hermes Agent MCP 配置

在 Hermes Agent 的 `~/.hermes/config.json` 中添加 MCP 服务端配置：

```json
{
  "mcpServers": {
    "firecrawl": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"],
      "env": {
        "FIRECRAWL_API_KEY": "fc-你的API密钥",
        "FIRECRAWL_RETRY_MAX_ATTEMPTS": "5",
        "FIRECRAWL_RETRY_INITIAL_DELAY": "2000",
        "FIRECRAWL_RETRY_MAX_DELAY": "30000",
        "FIRECRAWL_RETRY_BACKOFF_FACTOR": "3",
        "FIRECRAWL_CREDIT_WARNING_THRESHOLD": "2000",
        "FIRECRAWL_CREDIT_CRITICAL_THRESHOLD": "500"
      }
    }
  }
}
```

配置完成后重启 Hermes Agent，Agent 即可自动调用 Firecrawl 工具。

## Claude Desktop 配置

编辑 `claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "mcp-server-firecrawl": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"],
      "env": {
        "FIRECRAWL_API_KEY": "YOUR_API_KEY_HERE"
      }
    }
  }
}
```

## Cursor 配置

### Cursor v0.48.6+

1. 打开 Cursor Settings
2. 进入 Features > MCP Servers
3. 点击 "+ Add new global MCP server"
4. 粘贴以下配置：

```json
{
  "mcpServers": {
    "firecrawl-mcp": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"],
      "env": {
        "FIRECRAWL_API_KEY": "YOUR-API-KEY"
      }
    }
  }
}
```

### Cursor v0.45.6

1. 打开 Cursor Settings
2. 进入 Features > MCP Servers
3. 点击 "+ Add New MCP Server"
4. 填写：
   - Name: `firecrawl-mcp`
   - Type: `command`
   - Command: `env FIRECRAWL_API_KEY=your-api-key npx -y firecrawl-mcp`

Windows 用户请使用：`cmd /c "set FIRECRAWL_API_KEY=your-api-key && npx -y firecrawl-mcp"`

## VS Code 配置

将以下内容添加到 User Settings (JSON)（按 `Ctrl+Shift+P` → Preferences: Open User Settings (JSON)）：

```json
{
  "mcp": {
    "inputs": [
      {
        "type": "promptString",
        "id": "apiKey",
        "description": "Firecrawl API Key",
        "password": true
      }
    ],
    "servers": {
      "firecrawl": {
        "command": "npx",
        "args": ["-y", "firecrawl-mcp"],
        "env": {
          "FIRECRAWL_API_KEY": "${input:apiKey}"
        }
      }
    }
  }
}
```

也可在工作区 `.vscode/mcp.json` 中配置以共享给团队。

## Windsurf 配置

编辑 `./codeium/windsurf/model_config.json`：

```json
{
  "mcpServers": {
    "mcp-server-firecrawl": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"],
      "env": {
        "FIRECRAWL_API_KEY": "YOUR_API_KEY"
      }
    }
  }
}
```

## 可用工具一览

| 工具 | 适用场景 | 返回格式 |
|------|----------|----------|
| `firecrawl_scrape` | 单个页面提取 | JSON / Markdown |
| `firecrawl_batch_scrape` | 多个已知 URL 批量提取 | JSON[] / Markdown[] |
| `firecrawl_map` | 发现网站 URL 结构 | URL[] |
| `firecrawl_crawl` | 多页面全站抓取 | Markdown[] |
| `firecrawl_search` | 搜索引擎查询 | 搜索结果[] |
| `firecrawl_extract` | LLM 结构化提取 | 自定义 Schema |
| `firecrawl_agent` | 自主多源调研（异步） | JSON |

## 使用示例

### 抓取单个页面

```json
{
  "name": "firecrawl_scrape",
  "arguments": {
    "url": "https://example.com",
    "formats": ["markdown"],
    "onlyMainContent": true
  }
}
```

### 结构化数据提取

```json
{
  "name": "firecrawl_extract",
  "arguments": {
    "urls": ["https://example.com/product"],
    "prompt": "提取产品名称、价格和描述",
    "schema": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "price": { "type": "number" },
        "description": { "type": "string" }
      }
    }
  }
}
```

### 搜索并抓取

```json
{
  "name": "firecrawl_search",
  "arguments": {
    "query": "最新 AI 研究论文 2026",
    "limit": 5,
    "lang": "zh",
    "scrapeOptions": {
      "formats": ["markdown"],
      "onlyMainContent": true
    }
  }
}
```

### 使用 Agent 进行深度调研

```json
{
  "name": "firecrawl_agent",
  "arguments": {
    "prompt": "找出 Firecrawl 的创始人和他们的背景信息"
  }
}
```

Agent 任务异步执行，通过 `firecrawl_agent_status` 轮询结果。

## 自托管 Firecrawl

如果你使用自托管的 Firecrawl 实例：

```bash
export FIRECRAWL_API_URL=https://firecrawl.your-domain.com
export FIRECRAWL_API_KEY=fc-your-api-key  # 如果实例需要认证
npx -y firecrawl-mcp
```

## 流式 HTTP 模式

使用 Streamable HTTP 替代默认的 stdio 传输协议：

```bash
env HTTP_STREAMABLE_SERVER=true FIRECRAWL_API_KEY=fc-YOUR_API_KEY npx -y firecrawl-mcp
```

访问地址：`http://localhost:3000/mcp`

## 故障排除

| 问题 | 解决方案 |
|------|----------|
| API key 无效 | 在 https://www.firecrawl.dev/app/api-keys 重新生成 |
| 请求超限 | 检查信用额度，或配置重试参数 |
| 页面 JS 渲染内容为空 | 添加 `waitFor: 5000` 参数等待 JS 加载 |
| 自托管连接失败 | 确认 `FIRECRAWL_API_URL` 地址可达 |

---

**参考资料：**
- 官方仓库：https://github.com/firecrawl/firecrawl-mcp-server
- Firecrawl 官网：https://www.firecrawl.dev
- MCP 协议规范：https://modelcontextprotocol.io
