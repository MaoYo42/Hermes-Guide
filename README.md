# Hermes-Guide

> 🏛️ Hermes Agent 中文实战指南 | 从入门到多设备协同，社区驱动 · 实战优先
>
> Chinese practical guide for Hermes Agent by Nous Research — 取代过时的 OpenClaw-Guide

[![Hermes v0.13.0](https://img.shields.io/badge/Hermes-v0.13.0-blue)](https://github.com/NousResearch/hermes-agent)
[![License](https://img.shields.io/badge/License-MIT-green)](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE)
[![Nous Research](https://img.shields.io/badge/by-Nous%20Research-purple)](https://nousresearch.com/)

---

## 概述

**Hermes Agent** 是由 [Nous Research](https://nousresearch.com/) 开发的开源自改进 AI 代理。它不是 IDE 里的编码副驾驶，也不是套壳聊天机器人——它是一个**自主代理**，运行越久越强大：自动从经验中创建技能、跨会话持久记忆、支持 20+ 消息平台接入，并有完整的沙箱执行环境。

本指南作者在 **Arch Linux ThinkPad T480s、macOS、Android Termux** 三设备上日常使用 Hermes，配合 DeepSeek v4 Flash 模型和 Firecrawl MCP，是 OpenClaw-Guide 的正式继任者。

### 核心特性

| 特性 | 说明 |
|------|------|
| **自我进化** | 自动从成功工作流创建技能，执行中持续优化，跨会话积累能力 |
| **分层记忆** | 持久笔记 (MEMORY.md/USER.md) + FTS5 全文检索会话历史 + Honcho 用户建模 |
| **消息网关** | Telegram / Discord / Slack / WhatsApp / Signal / Email / 飞书 / 钉钉 / 企微等 20+ 平台 |
| **调度自动化** | 内置 Cron，自然语言定义定时任务，结果推送到任意平台 |
| **子代理并行** | 隔离子代理独立执行任务，零上下文开销的并行管线 |
| **沙箱执行** | 支持 Local / Docker / SSH / Daytona / Modal / Singularity 五种后端 |
| **MCP 集成** | 无缝接入全部 MCP 服务器，扩展数据库、GitHub、文件系统等能力 |
| **模型无关** | 支持 OpenAI / Anthropic / DeepSeek / OpenRouter / 本地模型等数十种提供商 |
| **从 OpenClaw 迁移** | `hermes claw migrate` 一键导入配置、记忆、技能和 API 密钥 |

---

## 特性对比：Hermes vs OpenClaw vs Claude Code vs Codex

| 维度 | Hermes Agent | OpenClaw | Claude Code | Codex CLI |
|------|-------------|----------|-------------|-----------|
| **开源协议** | MIT ✅ | MIT ✅ | 专有 ❌ | 专有 ❌ |
| **模型提供商** | 数十种，模型无关 ✅ | 多种 ✅ | 仅 Anthropic ❌ | 仅 OpenAI ❌ |
| **自我进化** | 自动创建/优化技能 ✅ | 人工编写技能 ⚠️ | 无 ❌ | 无 ❌ |
| **跨会话记忆** | 分层记忆 + FTS5 检索 ✅ | 文件式记忆 ✅ | 有限上下文 ❌ | 有限上下文 ❌ |
| **消息网关** | 20+ 平台 ✅ | 多平台 ✅ | 仅 CLI ❌ | 仅 CLI ❌ |
| **定时任务** | 内置 Cron ✅ | 需外部工具 ❌ | 无 ❌ | 无 ❌ |
| **子代理并行** | 内置 ✅ | 无 ❌ | 有限 ⚠️ | 有限 ⚠️ |
| **执行沙箱** | 5 种后端 ✅ | Docker 等 ✅ | 无 ❌ | 无 ❌ |
| **MCP 支持** | 完整支持 ✅ | 部分支持 ⚠️ | 支持 ✅ | 支持 ✅ |
| **编辑器集成** | ACP 协议 ✅ | 有限 ⚠️ | VS Code 插件 ✅ | VS Code 插件 ✅ |
| **安全防护** | 5 层纵深防御 ✅ | 基础防护 ⚠️ | 基础 ⚠️ | 基础 ⚠️ |
| **移动端运行** | Termux 支持 ✅ | 支持 ✅ | 不支持 ❌ | 不支持 ❌ |
| **从 OpenClaw 迁移** | `hermes claw migrate` ✅ | 无 ❌ | 无 ❌ | 无 ❌ |
| **定价** | 完全免费，仅需 API 费用 | 完全免费，仅需 API 费用 | $20/月 + API 费用 | $20/月 + API 费用 |
| **适用场景** | 通用代理 + 编码 + 自动化 | 个人 AI 助手平台 | 专业编码 | 专业编码 |

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户交互层                                │
│   CLI/TUI    Telegram    Discord    Slack    WhatsApp   ...      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                     Hermes Gateway (消息网关)                     │
│              路由、会话管理、平台适配、认证授权                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    AIAgent 核心执行循环                           │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────────────────┐  │
│  │  感知    │→│  推理    │→│  行动    │→│ 学习与自我改进     │  │
│  │ 输入处理 │  │ 模型调用 │  │ 工具执行 │  │ 技能创建/优化/记忆 │  │
│  └─────────┘  └─────────┘  └─────────┘  └───────────────────┘  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                       基础设施层                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│  │ 记忆系统  │ │ 技能系统  │ │ MCP 集成  │ │ 执行后端          │  │
│  │ MEMORY.md│ │ SQLite   │ │ Firecrawl│ │ Local/Docker/SSH  │  │
│  │ USER.md  │ │ skills/  │ │ GitHub   │ │ Daytona/Modal    │  │
│  │ SQLite DB│ │ Hub      │ │ ...更多  │ │ Singularity       │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│  │ Cron调度  │ │ 安全模块  │ │ 会话管理  │ │ 配置与日志        │  │
│  │ jobs.json│ │ 5层防御  │ │ sessions/│ │ config.yaml/.env  │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**核心工作流：** 用户的请求经 Gateway 进入 AIAgent 执行循环 → 模型推理 → 调用 70+ 内置工具或 MCP 工具 → 执行完成后自我评估 → 将成功的流程抽象为技能 → 存入记忆系统 → 下次可直接复用。

---

## 快速安装

### Linux / macOS / WSL2 / Termux

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc    # 或 source ~/.zshrc
```

### Windows (原生, PowerShell) — 早期 Beta

```powershell
irm https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.ps1 | iex
```

### 初始化配置

```bash
hermes setup          # 交互式配置向导
hermes model          # 选择模型提供商
hermes                # 开始聊天
```

> 完整安装指南见 [docs/01-quick-start.md](./docs/01-quick-start.md)

---

## 文档目录

| 文档 | 说明 |
|------|------|
| 📖 **[docs/01-quick-start.md](./docs/01-quick-start.md)** | 5 分钟快速上手 — 安装 → 配置 → 首次对话 |
| 📚 **docs/02-installation.md** (规划中) | 多平台安装详解（Linux / macOS / Termux / Docker） |
| ⚙️ **docs/03-configuration.md** (规划中) | 配置文件详解：`config.yaml`、`.env`、`SOUL.md` |
| 💬 **docs/04-messaging.md** (规划中) | 消息网关配置：Telegram / Discord / 微信 / 飞书等 |
| 🧠 **docs/05-memory.md** (规划中) | 记忆系统实战：MEMORY.md、USER.md、FTS5 检索 |
| 🔧 **docs/06-skills.md** (规划中) | 技能系统：自动创建、手动编写、Skills Hub |
| 🔌 **docs/07-mcp.md** (规划中) | MCP 集成：Firecrawl、GitHub、数据库等 |
| 🏗️ **docs/08-subagents.md** (规划中) | 子代理与并行执行 |
| ⏰ **docs/09-cron.md** (规划中) | 定时任务自动化 |
| 🔒 **docs/10-security.md** (规划中) | 安全配置与最佳实践 |
| 📱 **docs/11-termux.md** (规划中) | Android Termux 部署指南 |
| 🔄 **docs/12-migration.md** (规划中) | 从 OpenClaw 迁移 |
| 🐛 **docs/13-troubleshooting.md** (规划中) | 常见问题与故障排查 |
| 💡 **docs/14-tips.md** (规划中) | 实战技巧与最佳实践 |

---

## 技术栈

| 组件 | 版本/选择 | 说明 |
|------|----------|------|
| Hermes Agent | v0.13.0 | 核心代理框架 |
| 模型 | DeepSeek v4 Flash | 主力推理模型 |
| MCP | Firecrawl | 网页爬取与搜索 |
| 网关 | Telegram | 主力消息通道 |
| 运行设备 | Arch Linux + macOS + Termux | 三设备协同 |

---

## 相关链接

- [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs/)
- [Hermes Agent GitHub](https://github.com/NousResearch/hermes-agent)
- [Nous Research](https://nousresearch.com/)
- [OpenClaw（前身项目）](https://github.com/openclaw/openclaw)

---

## 许可证

本指南采用 MIT 许可证。Hermes Agent 本身同样采用 MIT 许可证。

---

<p align="center">
  <sub>由 <a href="https://github.com/MaoYo42">MaoYo42/AK</a> 维护 · 社区驱动 · 实战优先</sub>
</p>
