# 仪表盘与 API

## 概述

Hermes Agent 提供了两种 Web 界面和一套 REST API，方便用户以图形化方式管理 Agent 或集成到外部系统。

MaoYo42 在两台设备上分别运行了不同的 Web 界面：
- **T480s 桌面** — 运行内置 Dashboard（:9119），用于配置管理
- **Mac 笔记本** — 运行 hermes-web-ui（:8648），用于群组聊天

---

## 一、内置 Dashboard

内置 Dashboard 是 Hermes Agent 自带的 Web 管理界面，基于 **FastAPI** 后端 + **React** 前端。

### 启动

```bash
hermes dashboard
```

默认监听 `http://localhost:9119`。

MaoYo42 通过 systemd 用户服务确保 Dashboard 随系统自动启动：

```ini
# ~/.config/systemd/user/hermes-dashboard.service
[Unit]
Description=Hermes Agent Dashboard
After=network.target

[Service]
ExecStart=/usr/bin/hermes dashboard
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

### 三大模块

#### 1. 配置编辑器（Config Editor）

图形化编辑 `~/.hermes/config.yaml`：

- 模型选择：切换 provider 和默认模型
- 工具集开关：可视化启用/禁用工具
- Gateway 设置：配置 Telegram/Discord/Slack 网关
- Cron Job 管理：查看和调整定时任务

> 编辑后即时生效，无需重启 Agent。

#### 2. 环境变量管理（Env Manager）

管理 `~/.hermes/.env` 中的密钥和敏感配置：

- API Key 管理（DeepSeek、OpenAI、Firecrawl 等密钥）
- 网关 Token 配置
- MCP 服务认证凭据

> Dashboard 中的 Env Manager 比直接编辑文件更安全——它不会在页面源码或日志中明文暴露密钥。

#### 3. 会话浏览器（Session Browser）

查看所有历史会话记录：

- 按时间、状态、平台筛选
- 查看单个会话的完整对话内容
- 导出会话为 Markdown 或 JSON

---

## 二、hermes-web-ui（第三方界面）

hermes-web-ui 是一个由社区开发的 Hermes Agent Web 前端，运行在 :8648 端口。

### 特点

- **群组聊天**（Group Chat）— 这是最常用的功能
  - 创建多个 Agent 同时在线
  - 多个 Agent 共享一个对话窗口
  - 适合团队协作场景

MaoYo42 在 Mac 笔记本上通过 hermes-web-ui 创建了群组聊天，将 T480s 上的主 Agent 和 Mac 上的 Agent 拉到同一对话中协同工作。

### 对比

| 特性 | 内置 Dashboard (:9119) | hermes-web-ui (:8648) |
|------|----------------------|----------------------|
| 配置编辑 | 支持 | 不支持 |
| 环境变量管理 | 支持 | 不支持 |
| Agent 聊天 | 基本（单 Agent） | 高级（群组聊天） |
| 多 Agent 同时在线 | 不支持 | 支持 |
| 部署方式 | 内置于 Hermes | 独立安装 |
| 端口 | 9119 | 8648 |

---

## 三、API Server

Hermes Agent 提供 REST API，用于外部程序集成。

### 启用 API Server

```bash
hermes api-server
```

默认监听 `http://localhost:9119/api`（与 Dashboard 共享端口）。

### 核心端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/api/session` | POST | 创建新会话 |
| `/api/session/{id}` | GET | 获取会话信息 |
| `/api/session/{id}/chat` | POST | 发送消息（Blocking） |
| `/api/session/{id}/chat/stream` | POST | 发送消息（Stream） |
| `/api/session/{id}/messages` | GET | 获取历史消息 |
| `/api/session/list` | GET | 列出所有会话 |
| `/api/config` | GET/PUT | 读取/更新配置 |
| `/api/config/env` | GET/PUT | 读取/更新环境变量 |
| `/api/tools` | GET | 列出可用工具 |
| `/api/health` | GET | 健康检查 |

### 使用示例

```bash
# 创建一个新会话
curl -X POST http://localhost:9119/api/session \
  -H "Content-Type: application/json" \
  -d '{"session_name": "api-test"}'

# 发送消息（阻塞）
curl -X POST http://localhost:9119/api/session/{id}/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "你好，请用一句话介绍 Hermes Agent"}'

# 流式聊天
curl -N -X POST http://localhost:9119/api/session/{id}/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"message": "写一首关于 AI 的诗"}'
```

### 远程 API 访问

MaoYo42 通过 SSH 隧道安全暴露 API：在 T480s 上运行 API Server，Mac 笔记本通过 SSH 反向隧道访问：

```bash
# Mac 端：创建 SSH 隧道到 T480s
ssh -L 9119:localhost:9119 user@t480s-ip
```

之后 Mac 上就可以通过 `http://localhost:9119` 访问 T480s 的 Hermes API。

---

## 四、配置参考

```yaml
# ~/.hermes/config.yaml 中关于 Dashboard/API 的配置
dashboard:
  host: "127.0.0.1"       # 监听地址（默认本地）
  port: 9119              # 端口
  allowed_origins:        # CORS 控制
    - "http://localhost:8648"

api_server:
  enabled: true
  require_auth: false     # 生产环境建议设为 true
  api_key: ""             # 设置后需在 Header 中传递 X-API-Key
```

---

## 五、安全建议

1. **不要暴露 Dashboard 到公网** — 监听在 `127.0.0.1`，通过 SSH 隧道访问
2. **生产环境启用认证** — 设置 `require_auth: true` 和 `api_key`
3. **反向代理保护** — 使用 Nginx/Caddy 添加 HTTPS 和 IP 白名单
4. **Dashboard != WebUI** — 内置 Dashboard 偏管理，hermes-web-ui 偏聊天，按需启动
