# Profile 深度管理：多代理运行的艺术

> 多 Profile 管理 · 配置克隆 · 网关隔离 · Profile Distribution · 实战场景

Profile 系统让一台机器上运行多个完全独立的 Hermes 代理成为可能 —— 每个有自己的配置、API 密钥、记忆、会话、技能和网关状态。本文从高级管理到实战场景，全面覆盖。

---

## 目录

- [Profile 是什么](#profile-是什么)
- [Profile 管理命令速查](#profile-管理命令速查)
- [创建策略：空白 / 克隆 / 全量克隆](#创建策略空白--克隆--全量克隆)
- [Profile 与工作目录/沙箱的关系](#profile-与工作目录沙箱的关系)
- [网关隔离：每个 Profile 独立 Bot](#网关隔离每个-profile-独立-bot)
- [Profile Distribution：分享你的代理](#profile-distribution分享你的代理)
- [实战场景](#实战场景)
- [常见陷阱](#常见陷阱)

---

## Profile 是什么

Profile 本质上是一个独立的 `HERMES_HOME` 目录。每个 profile 有自己的：

```text
~/.hermes/profiles/<name>/
├── config.yaml      # 模型、提供商、工具配置
├── .env             # API 密钥、Bot Token
├── SOUL.md          # 人格与指令
├── memories/        # 持久记忆
├── sessions/        # 会话历史
├── skills/          # 独立技能集合
├── cron/            # 定时任务
├── state.db         # 状态数据库
└── logs/            # 日志
```

创建 profile 后，会自动生成一个同名 CLI 别名：

```bash
hermes profile create coder
# 你现在可以：
coder chat        # 等价于 hermes -p coder chat
coder setup       # 配置 coder 的 API key
coder doctor      # 检查 coder 的配置健康
```

---

## Profile 管理命令速查

```bash
# 创建
hermes profile create <name>               # 空白 profile
hermes profile create <name> --clone       # 克隆配置文件
hermes profile create <name> --clone-all   # 克隆全部（含记忆/会话）

# 使用
hermes -p <name> chat                      # 指定 profile
hermes --profile=<name> doctor
coder chat                                 # 别名（自动生成）

# 设置默认
hermes profile use <name>                  # 设置 sticky 默认

# 信息查看
hermes profile list                        # 列出所有 profile
hermes profile show <name>                 # 详细查看
hermes profile info <name>                 # 查看 distribution 信息

# 管理
hermes profile rename <old> <new>          # 重命名
hermes profile export <name>               # 导出为 tar.gz
hermes profile import <file>.tar.gz        # 导入
hermes profile delete <name>               # 删除（销毁全部数据）

# 分布（分享）
hermes profile install <url>               # 安装 profile distribution
hermes profile update <name>               # 更新 distribution
```

---

## 创建策略：空白 / 克隆 / 全量克隆

### 空白 Profile

```bash
hermes profile create my-bot
```

创建一个全新的 profile，仅包含 bundled 技能。需要运行 `my-bot setup` 配置 API key 和模型。

### 克隆配置 (`--clone`)

```bash
hermes profile create work --clone --clone-from default
```

从指定 profile 复制 `config.yaml`、`.env` 和 `SOUL.md` 到新 profile。**不会**复制记忆、会话历史或技能。

**最佳时机：** 想创建"同配置不同人格"的代理。比如两个代理用同一个 DeepSeek API Key 但不同的 SOUL 和技能集。

### 全量克隆 (`--clone-all`)

```bash
hermes profile create backup --clone-all
```

复制**一切** —— 配置、密钥、人格、所有记忆、会话历史、技能、Cron 任务、插件。完整的快照。

**最佳时机：** 备份、复制一个已经有完整上下文的代理。

### 克隆对比

| 特性 | `--clone` | `--clone-all` |
|------|-----------|---------------|
| config.yaml | ✓ | ✓ |
| .env | ✓ | ✓ |
| SOUL.md | ✓ | ✓ |
| skills/ | ✗ | ✓ |
| memories/ | ✗ | ✓ |
| sessions/ | ✗ | ✓ |
| cron/ | ✗ | ✓ |
| state.db | ✗ | ✓ |

---

## Profile 与工作目录/沙箱的关系

这是一个常见的混淆点，官方文档有明确说明：

> Profile 不等于工作目录，也不等于沙箱。

```text
Profile     → 代理的状态目录（config, memory, skills...）
Working Dir → terminal.cwd 指定的工作目录
Sandbox     → 限制文件系统访问的机制
```

### 关键区别

- Profile **不限制**代理的文件系统访问。`local` 后端的代理仍然可以访问你用户账户能访问的任何目录。
- 工作目录用 `terminal.cwd` 独立设置：

```yaml
# 这个 profile 的工作目录
terminal:
  backend: local
  cwd: /home/user/my-project
```

- 只有 Docker/SSH 等远程后端才能真正限制代理能做什么。

### 目录识别技巧

当前活跃的 profile 会在 CLI 中显示：

- **提示符：** `coder ❯` 而不是 `❯`
- **启动横幅：** 显示 `Profile: coder`
- **`hermes profile`：** 显示当前名称、路径、模型、网关状态

---

## 网关隔离：每个 Profile 独立 Bot

每个 profile 可以运行独立的网关进程，使用不同的 Bot Token：

```bash
# 启动不同 profile 的网关
coder gateway start           # coder 的 Telegram Bot
assistant gateway start       # assistant 的 Telegram Bot（独立进程）
persist gateway start         # 第三个 Bot
```

### 不同 Bot Token

```bash
# 编辑 coder 的 Bot Token
nano ~/.hermes/profiles/coder/.env
# TELEGRAM_BOT_TOKEN=123456:ABC-DEF...

# 编辑 assistant 的 Bot Token（不同的 Bot）
nano ~/.hermes/profiles/assistant/.env
# TELEGRAM_BOT_TOKEN=789012:GHI-JKL...
```

### Token 锁（安全机制）

如果两个 profile 意外使用了同一个 Bot Token，第二个启动的网关会明确报错：

```text
[ERROR] Token already in use by profile 'coder'. Stop that gateway first.
```

支持 Telegram、Discord、Slack、WhatsApp、Signal 的 Token 锁检测。

### 持久化服务

```bash
coder gateway install          # 创建 systemd 服务
assistant gateway install      # 创建另一个 systemd 服务
```

这样即使重启，每个 Bot 都会自动启动：

```text
hermes-gateway-coder.service
hermes-gateway-assistant.service
```

---

## Profile Distribution：分享你的代理

Profile Distribution 把完整的代理打包成 Git 仓库，别人一条命令就能安装。

### 什么时候用

- **分享专用代理** —— 合规监控、代码审查、研究助手
- **多设备同步** —— 笔记本和工作站运行同一代理
- **团队协作** —— 发布经过审核的内部代理
- **产品化** —— 以代理为产品的分发方式

### 仓库结构

```text
my-research-agent/
├── distribution.yaml    # 清单：名称、版本、环境变量要求
├── SOUL.md              # 人格
├── config.yaml          # 配置
├── skills/              # 捆绑技能
│   ├── arxiv-search/
│   │   └── SKILL.md
│   └── paper-summary/
│       └── SKILL.md
├── cron/                # 定时任务
├── mcp.json             # MCP 连接
└── README.md
```

### distribution.yaml 清单

```yaml
name: research-bot
version: 1.0.0
description: "自主研究助手，集成 arXiv 和 Web 搜索"
hermes_requires: ">=0.12.0"
author: "Your Name"
license: "MIT"

env_requires:
  - name: OPENAI_API_KEY
    description: "OpenAI API Key (模型访问)"
    required: true
  - name: SERPAPI_KEY
    description: "SerpAPI Key for web search"
    required: false
    default: ""
```

### 安装与更新

```bash
# 安装
hermes profile install github.com/you/research-bot --alias

# 更新
hermes profile update research-bot
```

### 安装者不泄露的目录

以下路径**永远不**进入 distribution，即使作者意外提交也会被安装器硬性排除：

- `auth.json` / `.env` — 密钥和凭据
- `memories/` — 对话记忆
- `sessions/` — 会话历史
- `state.db*` — 状态数据库
- `logs/` — 日志
- `workspace/` — 工作文件
- `plans/` — 计划

### Distribution Owned vs User Owned

更新时：

| 类别 | 文件 | 更新时 |
|------|------|--------|
| 分布拥有 | SOUL.md, skills/, cron/, mcp.json, distribution.yaml | 被替换 |
| 配置保留 | config.yaml | 保留（除非 `--force-config`） |
| 用户拥有 | memories/, sessions/, .env, logs/ | 从不触及 |

---

## 实战场景

### 场景 1：个人工作流分离

```bash
# 编码助手 —— Cursor 级别的 AI 配对编程
hermes profile create coder
coder config set model.default anthropic/claude-sonnet-4
echo "你是资深的 Python 全栈工程师。专注代码质量。" > ~/.hermes/profiles/coder/SOUL.md

# 研究助手 —— arXiv Web 搜索
hermes profile create researcher
researcher config set model.default openrouter/google/gemini-2.0-flash-001

# 生活助手 —— 日程、购物、提醒
hermes profile create life
life config set model.default deepseek/deepseek-chat
```

每个 profile 有自己的 Telegram Bot，在聊天中互不干扰：

```bash
coder gateway start       # Telegram 上的 @coder_bot
researcher gateway start  # Telegram 上的 @researcher_bot
life gateway start        # Telegram 上的 @life_bot
```

### 场景 2：多用户家庭共享

```bash
# 每个家庭成员一个隔离的 profile
hermes profile create alice
hermes profile create bob
hermes profile create charlie

# 各自独立的网关端口
echo "API_SERVER_ENABLED=true
API_SERVER_PORT=8650
API_SERVER_KEY=alice-秘密" >> ~/.hermes/profiles/alice/.env

echo "API_SERVER_ENABLED=true
API_SERVER_PORT=8651
API_SERVER_KEY=bob-秘密" >> ~/.hermes/profiles/bob/.env

# 各自启动
hermes -p alice gateway &
hermes -p bob gateway &
```

在 Open WebUI 中添加三个不同的连接（Alice/Bob/Charlie），每个用户只能看到自己的模型。

### 场景 3：多环境部署

```bash
# 开发环境
hermes profile create dev
dev config set model.default deepseek/deepseek-chat
dev config set terminal.backend local

# 生产环境（Docker 沙箱）
hermes profile create prod
prod config set model.default anthropic/claude-sonnet-4
prod config set terminal.backend docker

# 开发环境的小模型不限制 CORS
echo "API_SERVER_ENABLED=true
API_SERVER_CORS_ORIGINS=*" >> ~/.hermes/profiles/dev/.env

# 生产环境严格限制
echo "API_SERVER_ENABLED=true
API_SERVER_KEY=prod-复杂密钥
API_SERVER_CORS_ORIGINS=http://dashboard.internal:3000" >> ~/.hermes/profiles/prod/.env
```

### 场景 4：Kanban Worker Fleet

```bash
# 编排器 —— 分配任务，不执行
hermes profile create orchestrator
orchestrator skills install official/devops/kanban-orchestrator

# 编码 Worker
hermes profile create code-worker
code-worker config set model.default anthropic/claude-sonnet-4
code-worker skills install official/devops/kanban-worker

# 安全审查 Worker
hermes profile create security-worker
security-worker config set model.default openrouter/google/gemini-2.0-flash-001
security-worker skills install official/devops/kanban-worker
```

然后创建看板，指派任务：

```bash
hermes kanban create sprint-23
hermes kanban add sprint-23 "实现登录功能" --assignee code-worker
hermes kanban add sprint-23 "安全审计登录功能" --assignee security-worker
```

### 场景 5：多设备同步

笔记本上创建的代理同步到工作站：

```bash
# 笔记本
cd ~/.hermes/profiles/my-agent
git init && git add . && git commit -m "v1"
git remote add origin git@github.com:you/my-agent.git
git push -u origin main

# 工作站
hermes profile install github.com/you/my-agent --alias
```

记忆和会话按设备独立，配置和技能保持同步。

---

## 常见陷阱

### 陷阱 1：Profile 不等于安全隔离

Profile 不限制文件系统访问。如果需要在 Docker/SSH 后端中隔离，设置 `terminal.backend` 为 `docker` 或 `ssh`。

### 陷阱 2：克隆后记得改 API Key

`--clone` 会复制 `.env`。如果两个 profile 使用同一个 API Key，它们计费到同一个账户。要在新 profile 中改成不同的 key：

```bash
nano ~/.hermes/profiles/<name>/.env
```

### 陷阱 3：忘记启动网关

创建了 profile 但没启动网关，Telegram/Discord Bot 不会响应：

```bash
# 启动并持久化
<name> gateway start
<name> gateway install    # systemd 自启
```

### 陷阱 4：Distribution 更新覆盖自定义 SOUL

如果修改了 distribution 的 `SOUL.md`，更新时会**被替换**。把自定义内容放到 `local/` 目录下并在 SOUL 中引用，或者 fork 后用自己的 repo。

### 陷阱 5：删除 Profile 不可逆

```bash
hermes profile delete <name>
```

这会停止网关、删除 systemd 服务、移除别名、删除**所有**数据（包括记忆和会话）。`--yes` 跳过确认但同样不可逆。

---

## 参考

- [Hermes 官方 Profile 文档](https://hermes-agent.nousresearch.com/docs/user-guide/profiles)
- [Profile Distribution 文档](https://hermes-agent.nousresearch.com/docs/user-guide/profile-distributions)
- [Profile 命令参考](https://hermes-agent.nousresearch.com/docs/reference/profile-commands)
- [Kanban Worker Lanes](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes)
- [API Server 多用户设置](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server#multi-user-setup-with-profiles)

---

> ← 返回 [README](../README.md) · 编辑: `advanced/profiling.md`
