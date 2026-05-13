# API Server + Open WebUI：Web 界面操控 Hermes

> OpenAI 兼容 API · 端口 8642 · Web 前端的画布交互 · 群聊协作

Hermes 内置了 OpenAI 兼容的 API 服务器。这意味着任何支持 OpenAI 格式的前端 —— Open WebUI、LobeChat、LibreChat 甚至 `curl` —— 都能连接到 Hermes 并使用它的全部工具集。本文深入 API Server 配置、Open WebUI 集成和 `hermes-web-ui` 群聊协作。

---

## 目录

- [API Server 架构](#api-server-架构)
- [快速启动](#快速启动)
- [API 端点详解](#api-端点详解)
- [Open WebUI 集成](#open-webui-集成)
- [群聊协作：hermes-web-ui Group Chat](#群聊协作hermes-web-ui-group-chat)
- [多用户设置](#多用户设置)
- [安全配置](#安全配置)
- [常见问题与排错](#常见问题与排错)

---

## API Server 架构

```
┌─────────────────────────────────────────┐
│          任何 OpenAI 兼容客户端           │
│  Open WebUI / LobeChat / curl / Python  │
└────────────────────┬────────────────────┘
                     │ POST /v1/chat/completions
                     ▼
┌─────────────────────────────────────────┐
│         Hermes API Server (:8642)        │
│  认证层 (Bearer Token) → AIAgent 实例    │
│  System Prompt 叠加层                     │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│       Hermes 核心执行循环                 │
│  工具调用 · 终端 · MCP · 记忆 · 技能      │
│    (运行在 API Server 所在主机)            │
└─────────────────────────────────────────┘
```

**重要架构约束：** API Server 是 **Hermes Agent 运行时**，不是纯 LLM 代理。工具调用在 API Server 所在的主机上执行。如果 MacBook 把 Open WebUI 指向 Arch Linux 主力机的 API Server，`pwd`、文件操作、MCP 查询都在 Arch Linux 上执行。

---

## 快速启动

### 步骤 1：启用 API Server

```bash
# 写入 ~/.hermes/.env
echo "API_SERVER_ENABLED=true" >> ~/.hermes/.env
echo "API_SERVER_KEY=your-local-key" >> ~/.hermes/.env
```

### 步骤 2：启动 Gateway

```bash
hermes gateway
```

成功会看到：
```text
[API Server] API server listening on http://127.0.0.1:8642
```

### 步骤 3：验证

```bash
# 健康检查
curl http://127.0.0.1:8642/health
# {"status": "ok"}

# 模型列表
curl -H "Authorization: Bearer your-local-key" \
  http://127.0.0.1:8642/v1/models
# {"object":"list","data":[{"id":"hermes-agent",...}]}

# 测试对话
curl http://127.0.0.1:8642/v1/chat/completions \
  -H "Authorization: Bearer your-local-key" \
  -H "Content-Type: application/json" \
  -d '{"model":"hermes-agent","messages":[{"role":"user","content":"你好！"}]}'
```

### 一键启动脚本

```bash
# Hermes 自带的 Open WebUI 引导脚本（macOS/Linux）
cd ~/.hermes/hermes-agent
bash scripts/setup_open_webui.sh
```

此脚本会：
1. 确保 `.env` 包含 API Server 配置
2. 重启 Gateway
3. 安装 Open WebUI 到 `~/.local/open-webui-venv`
4. 创建启动脚本 `~/.local/bin/start-open-webui-hermes.sh`
5. 在 macOS 上安装 `launchd` 服务，在 Linux 上安装 `systemd --user` 服务

---

## API 端点详解

### POST /v1/chat/completions

标准的 OpenAI Chat Completions 格式。**无状态** —— 完整的对话历史通过 `messages` 数组传入：

```json
{
  "model": "hermes-agent",
  "messages": [
    {"role": "system", "content": "你是一个 Python 专家。"},
    {"role": "user", "content": "写一个斐波那契函数"}
  ],
  "stream": false
}
```

**流式输出** (`"stream": true`)：返回 SSE 事件流，包含 `chat.completion.chunk` 和 Hermes 自定义的 `hermes.tool.progress` 事件（代理执行工具时实时显示）。

**图片输入**：支持 `image_url`（远程 URL 和 `data:image/...` base64）：

```json
{
  "messages": [{
    "role": "user",
    "content": [
      {"type": "text", "text": "这个图片里有什么？"},
      {"type": "image_url", "image_url": {"url": "data:image/png;base64,..."}}
    ]
  }]
}
```

### POST /v1/responses

OpenAI Responses API 格式。**有状态** —— 通过 `previous_response_id` 保持多轮上下文：

```json
{
  "model": "hermes-agent",
  "input": "我项目里有哪些文件？",
  "instructions": "你是一个有帮助的编码助手。",
  "store": true
}
```

返回的响应包含工具调用和结果的结构化列表：

```json
{
  "id": "resp_abc123",
  "status": "completed",
  "output": [
    {"type": "function_call", "name": "terminal", "arguments": "{\"command\":\"ls\"}", "call_id": "call_1"},
    {"type": "function_call_output", "call_id": "call_1", "output": "README.md src/ tests/"},
    {"type": "message", "role": "assistant", "content": [{"type": "output_text", "text": "你的项目包含..."}]}
  ]
}
```

**多轮对话示例：**

```json
// 第一轮
{"input": "列出项目文件", "store": true}
// 返回响应包含工具调用历史。获取 response_id
// 第二轮（携带上下文）
{"input": "显示 README 内容", "previous_response_id": "resp_abc123"}
```

**命名会话** —— 使用 `conversation` 字段代替跟踪多个 response_id：

```json
{"input": "你好", "conversation": "my-project"}
{"input": "src/ 里有什么？", "conversation": "my-project"}
{"input": "运行测试", "conversation": "my-project"}
```

### GET /v1/models

列出可用模型。模型名称默认为 profile 名称（default profile 为 `hermes-agent`）。

### GET /health 和 GET /health/detailed

```bash
curl http://127.0.0.1:8642/health
# {"status": "ok", ...}

curl http://127.0.0.1:8642/health/detailed
# {"status": "ok", "active_sessions": 2, "running_agents": 1, ...}
```

### Runs API（流式友好替代）

```bash
# 创建 run
curl -X POST http://localhost:8642/v1/runs \
  -H "Authorization: Bearer your-key" \
  -H "Content-Type: application/json" \
  -d '{"input": "搜索最新的 AI 新闻", "session_id": "my-session"}'
# {"run_id": "run_abc123", "status": "started"}

# 轮询状态
curl http://localhost:8642/v1/runs/run_abc123 \
  -H "Authorization: Bearer your-key"

# SSE 事件流
curl http://localhost:8642/v1/runs/run_abc123/events \
  -H "Authorization: Bearer your-key"

# 中止
curl -X POST http://localhost:8642/v1/runs/run_abc123/stop \
  -H "Authorization: Bearer your-key"
```

### Jobs API（后台任务管理）

```bash
# 创建定时任务
curl -X POST http://localhost:8642/api/jobs \
  -H "Authorization: Bearer your-key" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "每天总结一次 Hacker News 热门",
    "schedule": "0 9 * * *",
    "delivery": {"platform": "telegram", "chat_id": "-100123456"}
  }'

# 列出任务
curl http://localhost:8642/api/jobs \
  -H "Authorization: Bearer your-key"

# 暂停
curl -X POST http://localhost:8642/api/jobs/<id>/pause \
  -H "Authorization: Bearer your-key"

# 立即执行
curl -X POST http://localhost:8642/api/jobs/<id>/run \
  -H "Authorization: Bearer your-key"
```

---

## Open WebUI 集成

### Docker 安装（推荐）

```bash
docker run -d -p 3000:8080 \
  -e OPENAI_API_BASE_URL=http://host.docker.internal:8642/v1 \
  -e OPENAI_API_KEY=your-local-key \
  -e ENABLE_OLLAMA_API=false \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

> 注意：`ENABLE_OLLAMA_API=false` 很重要 —— 否则 Open WebUI 会显示一个空的 Ollama 模型列表，干扰选择。

### Docker Compose

```yaml
version: "3"
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "3000:8080"
    volumes:
      - open-webui:/app/backend/data
    environment:
      - OPENAI_API_BASE_URL=http://host.docker.internal:8642/v1
      - OPENAI_API_KEY=your-local-key
      - ENABLE_OLLAMA_API=false
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: always

volumes:
  open-webui:
```

### 通过 Admin UI 配置

1. 打开 `http://localhost:3000`
2. 注册管理员账户（第一个注册用户自动成为管理员）
3. 点击头像 → **管理员面板** → **连接**
4. 找到 **OpenAI API**，点击**齿轮图标**（管理）
5. 点击 **+ 添加新连接**
6. 填写：
   - **URL**: `http://host.docker.internal:8642/v1`
   - **API Key**: 与 Hermes 的 `API_SERVER_KEY` 相同
7. 点击**对勾**验证连接
8. **保存**

### Chat Completions vs Responses API

在 Open WebUI 的管理面板中，编辑 Hermes 连接时可以切换 API 类型：

| 模式 | 端点 | 适用场景 |
|------|------|----------|
| **Chat Completions** (默认) | `/v1/chat/completions` | 推荐，即开即用 |
| **Responses** (实验性) | `/v1/responses` | 需要 `previous_response_id` 的场景 |

目前 Open WebUI 在两种模式下都管理客户端侧对话历史，即使 Responses 模式也是发送全部历史。Responses 的主要优势是 SSE 事件流包含结构化 `function_call` 和 `function_call_output` 事件。

### 系统提示处理

Open WebUI 发送的 `system` 消息叠加在 Hermes 核心提示词之上，而不是替换：

- Hermes 保留所有工具、记忆和技能
- Open WebUI 的 system prompt 作为额外指令

```text
# 在 Open WebUI 的设置中填入：
"You are a Python expert. Always include type hints."

# Hermes 仍然有终端、文件工具、Web 搜索等能力
# 只是额外遵守 "always include type hints" 的指令
```

---

## 群聊协作：hermes-web-ui Group Chat

> **hermes-web-ui** 是 Hermes 的一个消息平台适配器，让 Hermes 出现在 Open WebUI 中作为一个可以群聊的 AI 成员。这不是 Hermes 内置功能，而是通过 webhook 实现的集成模式。

### 架构

```text
┌──────────────────┐     ┌─────────────────┐     ┌──────────────────┐
│ 用户 A            │     │ 用户 B           │     │ 用户 C            │
│ Open WebUI Chat   │     │ Open WebUI Chat  │     │ Open WebUI Chat  │
└────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
         └───────────────┬────────┴───────────────┘
                         │
                ┌────────▼────────┐
                │  Open WebUI      │
                │ Webhook → Hermes │
                └────────┬────────┘
                         │ POST /chat/completions
                         ▼
                ┌─────────────────┐
                │ Hermes API      │
                │ Server (:8642)  │
                │ + GateWay       │
                └─────────────────┘
```

### 配置方式

要让 Hermes 以群聊方式在 Open WebUI 中协作：

1. **启动 Hermes API Server**（如上文所述）
2. **在 Open WebUI 中创建群组** —— Open WebUI 支持 workspace 和多用户协作
3. **添加 Hermes 作为 AI 成员** —— 每个用户可以邀请 AI 进入聊天

### 多用户协作流

```text
1. 用户 A 发送: "@hermes 帮我审查这段代码"
2. Hermes API Server 处理请求
3. 用户 B 可以继续: "@hermes 按用户 A 的建议改一下"
4. 所有用户都能看到 Hermes 的完整回复和工具调用
```

Open WebUI 原生支持 workspace 协作和 AI 模型切换，所以多个用户可以共享同一个 Hermes 实例。

### 注意事项

- Open WebUI 的群聊功能需要管理员启用
- 每个请求都携带完整对话历史，较大的会话会增加延迟
- 建议设置 `stream: true` 以获得更好的交互体验
- 如果要区分用户身份，需要通过 profiles 实现（见下文多用户设置）

---

## 多用户设置

要为不同用户提供隔离的 Hermes 实例，使用 profiles + 不同端口：

```bash
# 创建用户 profile
hermes profile create alice
hermes profile create bob

# 给 Alice 配独立 API Server
cat >> ~/.hermes/profiles/alice/.env <<EOF
API_SERVER_ENABLED=true
API_SERVER_PORT=8650
API_SERVER_KEY=alice-secret
EOF

# 给 Bob 配独立 API Server
cat >> ~/.hermes/profiles/bob/.env <<EOF
API_SERVER_ENABLED=true
API_SERVER_PORT=8651
API_SERVER_KEY=bob-secret
EOF

# 分别启动
hermes -p alice gateway &
hermes -p bob gateway &
```

在 Open WebUI 中添加两个连接：

| 连接 | URL | API Key | 模型名 |
|------|-----|---------|--------|
| Alice | `http://host.docker.internal:8650/v1` | `alice-secret` | `alice` |
| Bob | `http://host.docker.internal:8651/v1` | `bob-secret` | `bob` |

模型下拉列表会显示 `alice` 和 `bob`，分配到不同用户即可隔离。

**端口选择建议：** 避开默认占用：
- `8644` — webhook adapter
- `8645` — wecom-callback
- `8646` — msgraph-webhook
- 推荐 `8650+` 范围

---

## 安全配置

### 环境变量参考

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `API_SERVER_ENABLED` | `false` | 启用 API Server |
| `API_SERVER_PORT` | `8642` | 端口 |
| `API_SERVER_HOST` | `127.0.0.1` | 绑定地址（默认仅 localhost） |
| `API_SERVER_KEY` | 无 | Bearer Token |
| `API_SERVER_CORS_ORIGINS` | 无 | 允许的浏览器来源（逗号分隔） |
| `API_SERVER_MODEL_NAME` | profile 名 | `/v1/models` 显示的模型名 |

### 安全最佳实践

**生产环境严格配置：**

```bash
# 绑定 localhost（默认），不暴露到公网
API_SERVER_HOST=127.0.0.1

# 高强度密钥
API_SERVER_KEY=$(openssl rand -hex 32)

# Open WebUI 通过 Docker 内部网络访问，不需要 CORS
# 如果从浏览器直连，设置严格 CORS
API_SERVER_CORS_ORIGINS=http://dashboard.internal:3000
```

**安全警告：**

- API Server 暴露了 Hermes 的**完整工具集，包括终端命令**
- 绑定到 `0.0.0.0` 时，`API_SERVER_KEY` **必须设置**
- `API_SERVER_CORS_ORIGINS` 保持狭窄
- 默认 `127.0.0.1` 仅限本地使用，浏览器默认不可访问

### Linux Docker 注意事项

```bash
# 选项 1：host.docker.internal（需要 --add-host）
docker run --add-host=host.docker.internal:host-gateway ...

# 选项 2：host 网络模式
docker run --network=host -e OPENAI_API_BASE_URL=http://localhost:8642/v1 ...

# 选项 3：Docker 桥接 IP
docker run -e OPENAI_API_BASE_URL=http://172.17.0.1:8642/v1 ...
```

---

## 常见问题与排错

### 模型下拉列表是空的

```text
# 检查步骤
1. curl http://localhost:8642/health              # → {"status":"ok"}?
2. curl -H "Authorization: Bearer your-key" \
     http://localhost:8642/v1/models               # → 返回模型列表？
3. Open WebUI URL 是否包含 /v1 后缀？               # http://host.docker.internal:8642/v1
4. ENABLE_OLLAMA_API=false 是否设置？               # 否则空 Ollama 列表遮盖模型
```

### 连接测试通过，但没有加载模型

最常见原因：Open WebUI 的 URL 缺少 `/v1` 后缀。连接测试只检查网络可达性，不验证模型列表。

### 响应很慢

这是正常的。Hermes 可能执行了多个工具调用（读文件、跑命令、搜索网页）后才返回最终响应。复杂查询确实需要更长时间。

### "Invalid API Key" 错误

检查 `OPENAI_API_KEY`（Open WebUI）与 `API_SERVER_KEY`（Hermes）是否一致。如果之前已经在 Admin UI 中保存了错误密钥，需要：

1. 在 Admin UI → 连接中删除已保存的连接
2. 重新添加正确的密钥
3. 或者重置 Open WebUI 数据目录

> Open WebUI 在首次启动后将连接设置持久化到内部数据库。修改环境变量后需要删除已有连接或清除数据库。

---

## 参考

- [Hermes 官方 API Server 文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server)
- [Hermes 官方 Open WebUI 集成指南](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/open-webui)
- [Open WebUI GitHub](https://github.com/open-webui/open-webui)
- [Hermes 环境变量参考](https://hermes-agent.nousresearch.com/docs/reference/environment-variables)
- [Gateway 代理模式（Matrix E2EE）](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/matrix#proxy-mode-e2ee-on-macos)

---

> ← 返回 [README](../README.md) · 编辑: `advanced/apiserver-and-openwebui.md`
