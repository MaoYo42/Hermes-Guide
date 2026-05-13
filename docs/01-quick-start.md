# 01 — 5 分钟快速上手

> 从零安装 → 配置提供商 → 首次对话 → 基本配置

本指南带你从零开始，5 分钟内让 Hermes Agent 在你的机器上跑起来。之后你就能在终端里和它聊天、让它执行命令、帮你写代码或查资料。

---

## 目录

- [1. 安装 Hermes](#1-安装-hermes)
- [2. 刷新 Shell 并验证](#2-刷新-shell-并验证)
- [3. 选择模型提供商](#3-选择模型提供商)
- [4. 首次对话](#4-首次对话)
- [5. 基本配置](#5-基本配置)
- [6. 下一步](#6-下一步)
- [常见问题](#常见问题)

---

## 1. 安装 Hermes

### Linux / macOS / WSL2 / Android Termux

打开终端，执行一键安装命令：

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

这个命令会完成以下操作：
- 克隆 Hermes Agent 仓库
- 安装 Python 和 Node.js 依赖
- 创建虚拟环境
- 安装全局 `hermes` CLI 命令

> **Windows 用户：** 原生 Windows 暂不完全支持。请先安装 [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install)，然后在 WSL2 终端中执行上述命令。
>
> **Android Termux 用户：** 直接用上面的一行命令，安装器会自动识别 Termux 环境。详细指南参考官方 [Termux 文档](https://hermes-agent.nousresearch.com/docs/getting-started/termux)。

### 从 OpenClaw 迁移？

如果你已经在用 OpenClaw，Hermes 可以直接导入现有配置：

```bash
hermes claw migrate --dry-run    # 先预览可迁移的内容
hermes claw migrate               # 执行迁移
```

这会自动转换 OpenClaw 的记忆文件、技能、API 密钥和消息平台设置。

---

## 2. 刷新 Shell 并验证

安装完成后，刷新你的 shell 配置：

```bash
source ~/.bashrc    # 如果用的是 bash
# 或
source ~/.zshrc     # 如果用的是 zsh
```

验证安装是否成功：

```bash
hermes --version
```

如果显示版本号（如 `v0.13.0`），说明安装成功。

---

## 3. 选择模型提供商

Hermes 是模型无关的，支持数十种提供商。首次使用，你需要选择一个模型提供商。

最简单的方式是使用交互式向导：

```bash
hermes model
```

这会启动一个菜单，引导你选择提供商并完成认证。

### 推荐选择（按优先级排序）

| 提供商 | 说明 | 设置方式 |
|--------|------|----------|
| **DeepSeek** | 性价比极高的推理模型，v4 Flash 速度快 | 设置 `DEEPSEEK_API_KEY` 环境变量 |
| **Nous Portal** | Nous Research 自家平台，零配置 | `hermes model` → OAuth 登录 |
| **OpenAI Codex** | ChatGPT OAuth 登录，使用 Codex 模型 | `hermes model` → 设备码认证 |
| **Anthropic** | Claude 系列模型 | OAuth 登录或 API Key |
| **OpenRouter** | 多提供商路由，一台搞定所有模型 | 输入 OpenRouter API Key |
| **自定义端点** | Ollama / vLLM / SGLang 等本地模型 | 设置 base URL + API key |

> **最低上下文要求：** Hermes 需要模型至少 **64K tokens** 上下文。大多数云端模型（Claude、GPT、Qwen、DeepSeek）都满足。如果跑本地模型，确保上下文设置为至少 65536 tokens。

**小贴士：** 之后随时可以用 `hermes model` 切换提供商，没有任何绑定。

### 密钥存储说明

Hermes 将密钥和配置分开存储：

```text
~/.hermes/
├── config.yaml     # 非敏感配置（模型、终端后端等）
├── .env            # API 密钥和敏感令牌
└── auth.json       # OAuth 凭据
```

也可以通过 CLI 直接设置：

```bash
hermes config set model deepseek/deepseek-chat
hermes config set DEEPSEEK_API_KEY sk-xxxxxxxx
```

系统自动将密钥存到 `.env`，非敏感配置存到 `config.yaml`。

---

## 4. 首次对话

配置完模型后，启动 Hermes：

```bash
hermes            # 经典 CLI 模式
```

或推荐使用更现代的 TUI 模式：

```bash
hermes --tui      # TUI 模式（推荐）
```

你会看到欢迎横幅，显示当前的模型、可用工具和技能。

### 试试这些对话

```text
# 试试系统查询
你：我的磁盘使用情况怎么样？显示前 5 个最大的目录。

# 试试代码相关
你：看看当前目录，告诉我主要项目文件是什么。

# 试试信息检索
你：帮我查一下 Hermes Agent 的最新版本和更新内容。
```

**成功标志：**
- 横幅显示你的模型/提供商
- Hermes 正常回复，没有报错
- 它可以使用终端命令、读文件等工具
- 多轮对话持续正常

### 常用斜杠命令

在聊天中输入 `/` 可以看到自动补全列表：

| 命令 | 作用 |
|------|------|
| `/help` | 查看所有可用命令 |
| `/tools` | 列出可用工具 |
| `/model` | 切换模型 |
| `/skills` | 浏览和安装技能 |
| `/save` | 保存当前对话 |

### 小技巧

- **多行输入：** `Alt+Enter` 或 `Shift+Enter` 换行
- **中断代理：** 代理正在执行时，直接输入新消息按回车即可打断
- **恢复会话：** `hermes --continue`（或 `hermes -c`）恢复上一个会话

---

## 5. 基本配置

### 5.1 查看当前配置

```bash
hermes config              # 查看全部配置
hermes config list         # 列出所有设置项
```

### 5.2 常用配置项

```bash
# 切换模型提供商
hermes config set model deepseek/deepseek-chat

# 设置终端后端（默认 local）
hermes config set terminal.backend local     # 本地直接执行
hermes config set terminal.backend docker    # Docker 隔离执行
hermes config set terminal.backend ssh       # 远程服务器执行

# 配置工具集
hermes tools                 # 交互式配置每个平台的工具权限

# 诊断检查
hermes doctor                # 检查配置是否有问题
```

### 5.3 编辑配置文件

直接编辑配置文件：

```bash
hermes config edit           # 用默认编辑器打开 config.yaml
```

文件位于 `~/.hermes/config.yaml`，结构示例：

```yaml
model: deepseek/deepseek-chat
terminal:
  backend: local
tools:
  enabled:
    - terminal
    - web_search
    - file_system
    - code_execution
mcp_servers: {}
gateway:
  enabled: false
```

### 5.4 设置消息网关（可选）

如果你想让 Hermes 在 Telegram / Discord 等平台上可用：

```bash
hermes gateway setup          # 交互式配置消息平台
hermes gateway start          # 启动网关
hermes gateway status         # 查看网关状态
```

详细配置方法见后续文档 [04-messaging.md](./04-messaging.md)（规划中）。

### 5.5 配置 MCP 扩展（可选）

以 Firecrawl MCP 为例：

```yaml
# 编辑 ~/.hermes/config.yaml 添加：
mcp_servers:
  firecrawl:
    command: npx
    args: ["-y", "firecrawl-mcp"]
    env:
      FIRECRAWL_API_KEY: "fc-xxxxxxxx"
```

配置后重启 Hermes 即可使用 Firecrawl 的网页抓取和搜索能力。

---

## 6. 下一步

| 目标 | 操作 |
|------|------|
| 深入学习配置 | 阅读 `docs/03-configuration.md`（规划中） |
| 设置消息网关 | 阅读 `docs/04-messaging.md`（规划中） |
| 配置记忆系统 | 阅读 `docs/05-memory.md`（规划中） |
| 学习技能系统 | 阅读 `docs/06-skills.md`（规划中） |
| 集成 MCP 工具 | 阅读 `docs/07-mcp.md`（规划中） |
| 多设备协同 | 阅读 `docs/11-termux.md` 和 `docs/12-migration.md`（规划中） |

你也可以直接查阅 [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs/) 了解更多。

---

## 常见问题

### Q: `hermes` 命令找不到？
刷新 shell：`source ~/.bashrc`。如果还是找不到，确认安装脚本没有报错。

### Q: 启动后回复为空或报错？
通常是模型提供商认证问题。重新运行 `hermes model` 确认提供商和 API 密钥正确。

### Q: 如何更新 Hermes？
```bash
hermes update
```

### Q: 如何卸载？
```bash
# 删除 ~/.hermes 目录
rm -rf ~/.hermes
# 删除 CLI 命令（路径取决于安装方式，通常在 ~/.local/bin/hermes）
rm ~/.local/bin/hermes
```

### Q: 终端报错说上下文不足？
模型需要至少 64K tokens 上下文。如果用本地模型（如 Ollama），启动时加上 `--ctx-size 65536`。

### Q: 恢复会话提示找不到？
检查 `hermes sessions list`，确认你在正确的 profile 中。

---

> **恢复工具箱：** 出问题时按顺序执行：`hermes doctor` → `hermes model` → `hermes setup` → `hermes sessions list` → `hermes --continue` → `hermes gateway status`

---

<p align="center">
  <sub>← 返回 <a href="../README.md">README</a> · 由 <a href="https://github.com/MaoYo42">MaoYo42/AK</a> 维护</sub>
</p>
