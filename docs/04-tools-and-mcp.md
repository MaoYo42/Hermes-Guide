# 工具系统与 MCP 集成

## 内置工具集（Toolsets）

Hermes Agent 提供丰富的内置工具集，涵盖代码、终端、文件、网络、媒体等各种场景。每个工具集可独立开关。

### 工具集一览

| 工具集 | 功能 | 默认启用 |
|--------|------|----------|
| `web` | Web 搜索与内容抓取（完整版） | 是 |
| `search` | Web 搜索（web 的子集） | 是 |
| `browser` | 浏览器自动化（Browserbase / Camofox / 本地 Chromium） | 是 |
| `terminal` | Shell 命令执行与进程管理 | 是 |
| `file` | 文件读写、搜索、编辑（read_file / write_file / patch / search_files） | 是 |
| `code_execution` | 沙箱化 Python 代码执行 | 是 |
| `vision` | 图像分析与理解 | 是 |
| `image_gen` | AI 图像生成 | 是 |
| `video` | 视频分析与生成 | 是 |
| `tts` | 文本转语音 | 是 |
| `skills` | Skill 浏览与管理 | 是 |
| `memory` | 跨会话持久记忆（MEMORY.md / USER.md） | 是 |
| `session_search` | 搜索历史对话记录 | 是 |
| `delegation` | 子 Agent 任务委派 | 是 |
| `cronjob` | 定时任务管理 | 是 |
| `clarify` | 向用户主动提问澄清 | 是 |
| `messaging` | 跨平台消息发送（Telegram / Discord / Slack 等） | 是 |
| `todo` | 会话内任务规划和追踪 | 是 |
| `kanban` | 多 Agent 工作队列（worker 专用） | 是 |
| `debugging` | 额外调试/内省工具 | 否 |
| `safe` | 最低权限工具集（锁定会话） | 否 |
| `spotify` | Spotify 播放与控制 | 否 |
| `homeassistant` | 智能家居控制 | 否 |
| `discord` | Discord 集成工具 | 否 |
| `discord_admin` | Discord 管理/审核工具 | 否 |
| `feishu_doc` | 飞书文档工具 | 否 |
| `feishu_drive` | 飞书云盘工具 | 否 |
| `yuanbao` | 元宝集成 | 否 |
| `rl` | 强化学习工具 | 否 |
| `moa` | Mixture of Agents | 否 |

### 工具管理命令

```bash
# 交互式开关工具（curses 界面）
hermes tools

# 查看所有工具及状态
hermes tools list

# 启用/禁用特定工具集
hermes tools enable web
hermes tools disable web

# 按平台配置工具（不同平台可启用不同工具集）
hermes tools config
```

> 工具变更需要重启会话（`/reset` 或退出重进）才能生效，这保证了 prompt caching 的稳定性。

### 工具集与平台

Hermes 在每个平台上加载不同的默认工具集。CLI 使用 `_HERMES_CORE_TOOLS` 完整集；网关平台（Telegram / Discord 等）使用经过裁剪的版本。工具定义位于 `toolsets.py` 的 `TOOLSETS` 字典中。

---

## MCP 服务器集成

Hermes Agent 内置原生 MCP 客户端，可在启动时自动连接 MCP 服务器，发现其工具并注册为 Agent 可直接调用的第一类工具。

### 工作原理

启动时，`discover_mcp_tools()` 被调用：

1. 从 `config.yaml` 读取 `mcp_servers` 配置
2. 为每个服务器在后台事件循环中建立连接
3. 调用 `list_tools()` 发现可用工具
4. 将工具注册到 Hermes 工具注册表

### 配置方式

编辑 `~/.hermes/config.yaml`：

```yaml
mcp_servers:
  time:
    command: "uvx"
    args: ["mcp-server-time"]

  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/documents"]

  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_xxx"

  remote_api:
    url: "https://mcp.example.com/mcp"
    headers:
      Authorization: "Bearer sk-xxx"
```

### 配置选项

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `command` | string | -- | 可执行文件（stdio 传输，必填） |
| `args` | list | `[]` | 命令行参数 |
| `env` | dict | `{}` | 子进程环境变量 |
| `url` | string | -- | 服务器 URL（HTTP 传输，必填） |
| `headers` | dict | `{}` | HTTP 请求头 |
| `timeout` | int | `120` | 每次工具调用超时（秒） |
| `connect_timeout` | int | `60` | 初始连接超时（秒） |

> 注意：一个服务器配置只能选 `command`（stdio）或 `url`（HTTP），不能同时使用。

### 工具命名规则

MCP 工具注册格式：

```
mcp_{server_name}_{tool_name}
```

名称中的连字符和下划线会被转为下划线以兼容 LLM API。

示例：
- 服务器 `filesystem`，工具 `read_file` → `mcp_filesystem_read_file`
- 服务器 `github`，工具 `list-issues` → `mcp_github_list_issues`

### 连接生命周期

- 每个服务器作为一个长期存在的 asyncio Task 运行在后台线程
- 连接在整个 Agent 进程生命周期内保持
- 断连时自动重试（指数退避，最多 5 次，最长间隔 60 秒）
- 进程关闭时优雅断开

### 安全性

**环境变量过滤：** stdio 服务器不会继承完整的 shell 环境，只传递安全的基线变量（`PATH`、`HOME`、`USER` 等）。API 密钥等敏感变量必须通过 `env` 显式设置。

**凭据清理：** MCP 工具调用出错时，错误信息中的凭据模式（`ghp_*`、`sk-*`、`Bearer` 等）会被自动脱敏。

### 前置条件

```bash
# 安装 MCP Python SDK
pip install mcp

# Arch Linux 等 externally managed 环境
~/.hermes/hermes-agent/venv/bin/python3 -m pip install mcp

# Node.js（用于 npx 服务器）
# uv（用于 uvx 服务器）
```

---

## Firecrawl MCP 实战示例

Firecrawl MCP 是最常用的 MCP 集成之一，提供完整的 Web 抓取和搜索能力。

### 配置

```yaml
# ~/.hermes/config.yaml
mcp_servers:
  firecrawl:
    command: npx
    args: ["-y", "firecrawl-mcp"]
    env:
      FIRECRAWL_API_KEY: "fc_your_key_here"
```

> 注意 npm 包名是 `firecrawl-mcp`，不是 `@firecrawl/mcp`。

### 可用工具（共 11 个）

| 工具 | 用途 |
|------|------|
| `mcp_firecrawl_scrape` | 抓取单页内容（支持 markdown / JSON schema / screenshot / 品牌分析） |
| `mcp_firecrawl_search` | 网页搜索（站点限定、搜索运算符、图片/新闻源） |
| `mcp_firecrawl_map` | 发现站内所有 URL，支持 search 参数定位 |
| `mcp_firecrawl_crawl` | 深度爬站（异步，返回 job ID） |
| `mcp_firecrawl_check_crawl_status` | 查询爬站进度 |
| `mcp_firecrawl_extract` | 从多个 URL 提取结构化数据（JSON schema + prompt） |
| `mcp_firecrawl_agent` | 自主研究 Agent（异步，复杂调研） |
| `mcp_firecrawl_agent_status` | 查询 Agent 调研进度 |
| `mcp_firecrawl_interact` | 在已抓取的页面上交互（点击、填表、导航） |
| `mcp_firecrawl_interact_stop` | 结束交互会话 |
| `mcp_firecrawl_parse` | 解析本地文件（PDF / Word / Excel / HTML） |

### 常见限制

- **Reddit 不支持**：Firecrawl 会返回 451
- **X/Twitter 支持良好**：专门的处理器可结构化输出推文、回复、点赞
- **免费额度**：500 credits/月，个人日常使用足够
- **search 优先**：先用 search 获取结果摘要，再对需要的页面单独 scrape，比 search+scrapeOptions 更高效

### MCP CLI 命令

```bash
hermes mcp list              # 列出已配置的 MCP 服务器
hermes mcp add NAME          # 添加 MCP 服务器
hermes mcp remove NAME       # 移除
hermes mcp test NAME         # 测试连接
hermes mcp configure NAME    # 切换工具选择
hermes mcp serve             # 将 Hermes 自身作为 MCP 服务器运行
```
