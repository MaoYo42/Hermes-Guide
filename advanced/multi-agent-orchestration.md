# 多代理编排与并行执行

> delegate_task · Kanban 工作流 · 多设备协同 · 三设备实战

Hermes 的多代理能力分为两个层面：**轻量级子代理**（`delegate_task`）用于并行任务分解，**Kanban 系统**用于持久化工作流编排。本文从实战角度深入两者，并展示如何在三设备上协同运行。

---

## 目录

- [delegate_task 深入](#delegate_task-深入)
- [Kanban 工作流](#kanban-工作流)
- [多设备模式](#多设备模式)
- [三设备实战架构](#三设备实战架构)
- [模式对比](#模式对比)
- [常见陷阱](#常见陷阱)

---

## delegate_task 深入

`delegate_task` 是 Hermes 最强大的内置工具之一。它生成隔离的子代理实例，每个有独立的对话上下文、终端会话和工具集。只有最终摘要进入父代理的上下文。

### 基础用法

```python
# 单任务
delegate_task(
    goal="分析项目 src/api/handlers.py 中的安全漏洞",
    context="""
        项目路径: /home/user/webapp
        使用 Flask + PyJWT + bcrypt
        关注: SQL注入、JWT验证、密码处理
    """,
    toolsets=["terminal", "file"]
)

# 并行批次（默认最多 3 个并发）
delegate_task(tasks=[
    {"goal": "调研主题 A", "toolsets": ["web"]},
    {"goal": "调研主题 B", "toolsets": ["web"]},
    {"goal": "修复构建", "toolsets": ["terminal", "file"]}
])
```

### 子代理独立终端

**每个子代理获得独立的终端会话**，包括独立的：
- 工作目录和 shell 环境
- 进程树和运行时状态
- 环境变量（继承自主代理）

这意味着它们可以在同一个物理目录上并行工作而不会互相干扰 —— 只要不编辑同一个文件。

### 关键参数参考

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `toolsets` | 全量 | 子代理可用的工具集 |
| `max_iterations` | 50 | 子代理最大工具调用轮数 |
| `role` | `leaf` | `leaf` 不能继续委托，`orchestrator` 可以 |
| `context` | 空 | 传递给子代理的全部上下文信息 |

### 嵌套编排（Nested Orchestration）

默认情况下委托是扁平的（depth=1 —— 父代理生成子代理，子代理不能继续委托）。要启用嵌套：

```yaml
# ~/.hermes/config.yaml
delegation:
  max_spawn_depth: 2           # 允许 2 层嵌套
  orchestrator_enabled: true   # 默认即是 true
```

```python
# 父代理 → orchestrator（depth 1）→ leaf 子代理（depth 2）
delegate_task(
    goal="调查三种代码审查方法并推荐一种",
    role="orchestrator",         # 这个子代理可以继续委托
    context="..."
)
```

**成本警告：** `max_spawn_depth: 3` + `max_concurrent_children: 3` 理论上可达 27 个并发叶子节点。每层都翻倍消耗。

### 模型覆盖（省钱技巧）

可以给子代理单独指定一个更便宜的模型：

```yaml
delegation:
  model: "google/gemini-flash-2.0"
  provider: "openrouter"
```

也可以在 `delegate_task` 调用中透传（由代理自动完成）。这会显著降低并行批次的 token 消耗。

### /agents 监控面板

在 TUI 模式下输入 `/agents`（别名 `/tasks`）可以：

- 查看所有正在运行和已完成子代理的实时树状视图
- 查看每分支的 token/成本消耗汇总
- 在飞行中中止特定子代理而不影响其他
- 回顾每个子代理的逐轮操作历史

经典 CLI 模式下显示文本摘要。

### 生命周期与持久性

```
delegate_task 是同步的，非持久的
```

- 如果父代理被中断（用户发新消息、`/stop`、`/new`），所有活动子代理被取消
- 已取消子代理返回 `status="interrupted"`，未完成的工作丢弃
- 子代理不会在父代理轮次结束后继续运行

**对于需要持久化的工作**（必须跨中断存活），使用：

- `cronjob(action="create")` — 调度独立的代理运行
- `terminal(background=True, notify_on_complete=True)` — 后台 shell 命令

### 实用模式：分批大规模并行

默认并发数为 3。如果确实需要更多并行子代理：

```yaml
delegation:
  max_concurrent_children: 10   # 没有硬上限，只有下限 1
```

但实际效果取决于 API 速率限制。对 DeepSeek / OpenAI 等云模型，建议 3-5 个并发；对本地模型（Ollama/vLLM）可以更高。

---

## Kanban 工作流

Kanban 系统是 Hermes 的持久化任务编排框架，适用于需要跨越多个会话、涉及多个人类审查者或需要完整审计轨迹的场景。

### Kanban vs delegate_task

| 维度 | delegate_task | Kanban |
|------|--------------|--------|
| 生命周期 | 单次代理轮次内 | 跨会话、跨天 |
| 持久性 | 父中断则子取消 | 持久化到 SQLite |
| 并行 | 线程池并发 | Worker lane 调度 |
| 审查 | 无 | 支持 `review-required` 阻塞 |
| 审计 | 无 | 完整的事件日志 |
| 适用场景 | 实时并行子任务 | 长期项目、多人协作 |

### Kanban 核心概念

```text
Hermes Kanban  = 规范化任务生命周期 + 审计轨迹
Worker lane    = 负责执行被指派卡片的实现者
Reviewer       = 把关 "完成" 状态的人
```

### 任务生命周期

```
ready → claimed → running → completed
                           → blocked  → (reviewer 处理) → ready
                           → gave_up / crashed / timed_out
```

### 创建和操作看板

```bash
# 创建看板
hermes kanban create my-project --description "项目规划"

# 添加任务
hermes kanban add my-project "实现用户认证模块" --assignee coder

# 查看任务
hermes kanban show my-project

# 查看特定任务详情
hermes kanban show my-project --task <id>

# 查看运行日志
hermes kanban tail <task_id>

# 诊断
hermes kanban diagnostics my-project
```

### Worker Lane 架构

Worker lane 是一个三类契约：

1. **指派标识符字符串**（assignee）—— 通常是 Hermes profile 名
2. **生成机制**（spawn）—— `hermes -p <profile> chat -q <prompt>`
3. **生命周期终结器** —— `kanban_complete` / `kanban_block` / 进程退出

#### 环境变量注入

当 worker 被调度时，以下环境变量可用：

| 变量 | 用途 |
|------|------|
| `HERMES_KANBAN_TASK` | 当前任务 ID |
| `HERMES_KANBAN_DB` | 看板 SQLite 文件路径 |
| `HERMES_KANBAN_BOARD` | 看板 slug |
| `HERMES_KANBAN_WORKSPACE` | 当前任务的工作目录 |
| `HERMES_KANBAN_RUN_ID` | 当前运行 ID |
| `HERMES_KANBAN_CLAIM_LOCK` | 声明锁字符串 |

#### 完成 vs 阻塞约定

对于代码变更任务，使用 `review-required` 前缀阻塞：

```python
# 工作完成，需要审查
kanban_block(reason="review-required: 添加了用户认证模块，文件变更见注释")

# 先在注释中记录详细信息
kanban_comment(text="变更文件: src/auth/login.py, src/auth/jwt.py\n测试结果: 全部通过")
```

审查者批准后调用 `kanban_unblock` 让任务重新进入 `ready` 状态。

### 多 Profile 看板舰队

```bash
# 创建不同角色的 profile
hermes profile create coder
hermes profile create researcher
hermes profile create reviewer

# 在 kanban 中指派任务给不同 profile
hermes kanban add my-project "研究 OAuth2.0 方案" --assignee researcher
hermes kanban add my-project "实现登录页面" --assignee coder
hermes kanban add my-project "审查登录实现" --assignee reviewer
```

_dispatcher 会自动根据 assignee 字段将任务路由到对应的 profile。_

---

## 多设备模式

当你在多台设备上运行 Hermes 时，有三类协同模式可以选择：

### 模式 A：统一网关，多设备接入

所有设备通过同一个 Telegram/Discord Bot 连接到同一个 Hermes 实例。

```text
┌──────────┐     ┌──────────┐     ┌──────────┐
│ 笔记本    │     │ 台式机   │     │ 手机      │
│ Telegram  │     │ Telegram  │     │ Telegram  │
└─────┬────┘     └─────┬────┘     └─────┬────┘
      └────────────┬───┴────────────┘
                   │
          ┌────────▼────────┐
          │  Hermes Gateway  │
          │  (云端 VPS)      │
          │  统一 API Key    │
          └─────────────────┘
```

**优点：** 记忆统一、技能统一、在哪聊都一样
**缺点：** 单点故障、网关设备必须 24/7 在线

### 模式 B：独立设备，Git 同步 Profile

每台设备运行独立的 Hermes 实例，通过 Git 同步 profile 分布。

```bash
# 在笔记本上创建 profile
hermes profile create my-agent
cd ~/.hermes/profiles/my-agent
git init && git add . && git commit -m "initial"
git remote add origin git@github.com:you/my-agent.git
git push -u origin main

# 在台式机上安装
hermes profile install github.com/you/my-agent --alias

# 笔记本更新后：
git push

# 台式机拉取更新：
hermes profile update my-agent
```

**优点：** 无单点故障、每设备独立记忆、离线和本地执行
**缺点：** 记忆不共享、需手动同步

### 模式 C：API Server 分发（推荐）

一台主力设备运行 API Server，其他设备通过 Open WebUI / curl 接入。

```text
┌────────────┐
│  主力机     │ ←── 运行 Hermes API Server (:8642)
│  Arch Linux │     配置了全部 MCP、Firecrawl、记忆
└─────┬──────┘
      │ localhost:8642
      ├──────────────────┐
      │                  │
┌─────▼──────┐   ┌──────▼───────┐
│  笔记本      │   │  Android 手机  │
│  Open WebUI  │   │  Termux       │
│  → :8642/v1  │   │  → :8642/v1   │
└────────────┘   └──────────────┘
```

**优点：** 统一计算资源、_瘦客户端_访问、记忆集中在主力机
**缺点：** 主力机需要在线、网络延迟、远程执行（工具在主力机端）

---

## 三设备实战架构

以下是我的三设备协同方案，经过长期验证。

### 硬件配置

| 设备 | 系统 | 角色 | 运行组件 |
|------|------|------|----------|
| ThinkPad T480s | Arch Linux | **主力机** | Hermes CLI/TUI、API Server、Telegram Gateway、Firecrawl MCP |
| MacBook | macOS | **工作站** | Open WebUI → 主力机 API Server |
| Android 手机 | Termux | **移动端** | Hermes CLI、Telegram 客户端 |

### 架构图

```text
                         ┌─────────────────────┐
                         │    Telegram Bot      │
                         │ (统一消息网关)        │
                         └──────────┬──────────┘
                                    │
┌───────────────────────────────────┼───────────────────────────────┐
│  ThinkPad T480s (Arch Linux)     │                               │
│                                   ▼                               │
│  ┌─────────────────────────────────────────────┐                 │
│  │          Hermes Gateway                       │                 │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────┐  │                 │
│  │  │CLI / TUI │  │API Server│  │TG Gateway  │  │                 │
│  │  │          │  │:8642     │  │            │  │                 │
│  │  └──────────┘  └──────────┘  └────────────┘  │                 │
│  └─────────────────────┬────────────────────────┘                 │
│                        │                                          │
│  ┌─────────────────────▼────────────────────────┐                 │
│  │     Infra: MCP Firecrawl · 记忆 · 技能 · Cron │                 │
│  └──────────────────────────────────────────────┘                 │
└───────────────────────────────────────────────────────────────────┘
          │                               │
          │ :8642/v1                      │ Telegram
          ▼                               ▼
┌──────────────────┐          ┌──────────────────────┐
│  MacBook          │          │  Android (Termux)    │
│  Open WebUI       │          │  Telegram Client     │
│  → T480s:8642/v1  │          │  Hermes CLI (轻量)   │
│  Web 界面远程操控   │          │  应急 CRON 任务       │
└──────────────────┘          └──────────────────────┘
```

### 配置详解

#### 主力机（Arch Linux）配置

```bash
# ~/.hermes/.env
API_SERVER_ENABLED=true
API_SERVER_KEY=my-secret-key
API_SERVER_PORT=8642
API_SERVER_HOST=0.0.0.0                  # 允许局域网访问
API_SERVER_CORS_ORIGINS=http://macbook.local:3000

# Telegram Bot Token
TELEGRAM_BOT_TOKEN=123456:ABC-DEF...
```

```yaml
# ~/.hermes/config.yaml
model: deepseek/deepseek-chat
terminal:
  backend: local
mcp_servers:
  firecrawl:
    command: npx
    args: ["-y", "firecrawl-mcp"]
    env:
      FIRECRAWL_API_KEY: "fc-xxx"
```

#### macOS 工作站配置

```bash
# Open WebUI Docker 启动
docker run -d -p 3000:8080 \
  -e OPENAI_API_BASE_URL=http://thinkpad.local:8642/v1 \
  -e OPENAI_API_KEY=my-secret-key \
  -e ENABLE_OLLAMA_API=false \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

> 注意：`thinkpad.local` 需要 mDNS 支持（Arch Linux 安装 `avahi` 并启用 `avahi-daemon.service`）。也可以直接用 IP 地址。

#### Android (Termux) 配置

```bash
# Termux 中安装
pkg install python git curl
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# 简单配置（轻量使用，连接到主力机 API）
# 通常在 Termux 上跑 CLI，解决一些简单问题
# 或者在 Telegram 上通过统一网关使用
```

### 多设备协同建议

1. **主力机始终在线** —— 用 `systemd` 确保 Hermes 和 API Server 开机自启
2. **Telegram 作为统一接口** —— 三设备都可以收发消息，网关在主力机
3. **API Server 内网访问** —— 不要暴露到公网，仅在局域网使用
4. **Termux 做应急备用** —— 主力机挂了也能在手机上查看和操作
5. **定期同步技能** —— 用 Git 同步自创 skills 到各设备

### systemd 自启配置

```bash
# /etc/systemd/system/hermes-gateway.service
[Unit]
Description=Hermes Agent Gateway
After=network.target

[Service]
Type=simple
User=maoyo
Environment=HOME=/home/maoyo
ExecStart=/home/maoyo/.local/bin/hermes gateway
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

```bash
sudo systemctl enable --now hermes-gateway
```

---

## 模式对比

| 模式 | delegate_task | Kanban | Cron Job |
|------|-------------|--------|----------|
| **运行时机** | 代理轮次内 | 按任务调度 | 按时间计划 |
| **是否同步** | 同步（阻塞父代理） | 异步（独立进程） | 异步 |
| **并行能力** | 默认 3 并发 | Worker Lane 并发 | 独立运行 |
| **持久性** | 不持久（中断丢失） | 持久（SQLite） | 持久 |
| **审计** | 无 | 完整事件日志 | 有日志 |
| **适合** | 实时子任务 | 长期项目 | 定时任务 |
| **是否可中断** | 是（父中断→子中断） | 是（独立） | 否（后台） |

---

## 常见陷阱

### 陷阱 1：子代理没有上下文！

子代理是**完全空白**地启动的。它们不知道父代理之前说了什么、做了什么。必须把所有需要的上下文通过 `context` 参数传递：

```python
# 错误！子代理不知道 "那个 bug" 是什么
delegate_task(goal="修复那个 bug")

# 正确
delegate_task(
    goal="修复 api/handlers.py 第 47 行的 TypeError",
    context="文件路径: /home/user/project/api/handlers.py\n"
            "错误信息: 'NoneType' object has no attribute 'get'\n"
            "parse_body() 在 Content-Type 缺失时返回 None"
)
```

### 陷阱 2：Kanban Worker 没有 kanban-worker 技能

Worker profile 必须安装了 `kanban-worker` 技能才能响应 kanban 任务：

```bash
hermes -p coder skills install official/devops/kanban-worker
```

### 陷阱 3：API Server 远程执行

通过 API Server 发起的工具调用**运行在 API Server 机器上**，不是在客户端。如果你在 MacBook 的 Open WebUI 中运行 `pwd`，返回的是 Arch Linux 主力机上的路径。这不是 BUG，是架构设计。

### 陷阱 4：多设备共享同一 Telegram Token

如果两个 Hermes 实例使用同一个 Telegram Bot Token，Telegram API 会把消息随机发到其中一个。每个 profile/实例必须使用独立的 Bot Token。

### 陷阱 5：子代理冲突

多个子代理如果编辑同一个文件，最后一个写完的才会生效。确保子代理编辑不同的文件，或者让父代理在并行工作完成后合并冲突。

---

## 参考

- [Hermes 官方子代理委托文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- [委托与并行工作指南](https://hermes-agent.nousresearch.com/docs/guides/delegation-patterns)
- [Kanban Worker Lanes](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes)
- [Profile Distributions](https://hermes-agent.nousresearch.com/docs/user-guide/profile-distributions)
- [Kanban-worker 技能](https://github.com/NousResearch/hermes-agent/blob/main/skills/devops/kanban-worker/SKILL.md)
- [Kanban-orchestrator 技能](https://github.com/NousResearch/hermes-agent/blob/main/skills/devops/kanban-orchestrator/SKILL.md)

---

> ← 返回 [README](../README.md) · 编辑: `advanced/multi-agent-orchestration.md`
