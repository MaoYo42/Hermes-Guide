# 消息网关（Gateway）

Hermes Agent 的网关（Gateway）是连接消息平台的桥梁，让 Agent 能通过 Telegram、Discord、Slack、WhatsApp 等平台与你通信，且在不同平台上享有完整工具访问权限——不只是聊天机器人。

---

## Telegram 网关（生产环境运行中）

当前系统上 Telegram 网关已通过 systemd 用户服务运行，状态活跃。

### 实时配置

- **平台状态**：Telegram 已上线
- **允许的用户 ID**：`1419480065`（AK）
- **聊天白名单**：通过 `allowed_chats` / `allow_from` 配置
- **运行方式**：systemd 用户服务

### Telegram 配置参考（`~/.hermes/config.yaml`）

```yaml
telegram:
  bot_token: "${TELEGRAM_BOT_TOKEN}"       # 从 .env 读取
  user_id: "1419480065"                     # 你的 Telegram 用户 ID
  allow_from: ["1419480065"]                # 允许的用户白名单
  allowed_chats: ["-1001234567890"]         # 允许的群组白名单
  require_mention: true                      # 群组中需要 @提及才回复
  guest_mode: false                         # 关闭白名单之外群组的 @提及响应
  free_response_chats: ["-1001234567890"]   # 不需要 @提及的群组
  reply_to_mode: "on"                       # 回复模式（on/off/quoted）
  reactions: true                           # 是否添加反应表情
```

关键配置字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `bot_token` | string | Telegram Bot Token（建议存 .env） |
| `allow_from` | list | 白名单用户 ID，仅这些用户可与 Agent 对话 |
| `allowed_chats` | list | 白名单群组 ID，仅在这些群组中响应 |
| `guest_mode` | bool | 非白名单群组中是否通过 @提及触发 |
| `require_mention` | bool | 群组中是否必须 @提及才响应 |
| `free_response_chats` | list | 不需要 @提及即可响应的群组 |
| `reply_to_mode` | string | 回复模式（on = 回复原消息） |

### 网关 CLI 命令

```bash
hermes gateway run              # 前台启动网关
hermes gateway install          # 安装为后台服务
hermes gateway start            # 启动服务
hermes gateway stop             # 停止服务
hermes gateway restart          # 重启服务
hermes gateway status           # 查看状态
hermes gateway setup            # 交互式配置平台
```

### Systemd 用户服务

当前网关运行配置：

```
~/.config/systemd/user/hermes-gateway.service
```

```ini
[Unit]
Description=Hermes Agent Gateway - Messaging Platform Integration
After=network-online.target

[Service]
Type=simple
ExecStart=/home/maoyo/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run --replace
WorkingDirectory=/home/maoyo/.hermes/hermes-agent
Environment="HERMES_HOME=/home/maoyo/.hermes"
Restart=always
RestartSec=60

[Install]
WantedBy=default.target
```

服务管理方式：

```bash
# 启用开机自启
systemctl --user enable hermes-gateway

# 启动/停止/重启
systemctl --user start hermes-gateway
systemctl --user stop hermes-gateway
systemctl --user restart hermes-gateway

# 查看日志
journalctl --user -u hermes-gateway -f

# 重置失败状态（崩溃循环后）
systemctl --user reset-failed hermes-gateway
```

### 网关设置前提条件

1. **创建 Bot**：通过 [@BotFather](https://t.me/BotFather) 获取 Bot Token
2. **获取 User ID**：向 [@userinfobot](https://t.me/userinfobot) 发送任意消息
3. **安装依赖**：`pip install python-telegram-bot`
4. **配置网关**：`hermes gateway setup` 交互式向导

### 网关会话命令

网关中可使用以下斜杠命令：

```
/approve          批准待执行的命令
/deny             拒绝待执行的命令
/restart          重启网关
/sethome          设置当前会话为主频道
/topic            管理 Telegram DM 话题
/platforms        查看平台连接状态
/commands         浏览所有可用命令
/status           查看会话信息
/update           更新 Hermes
```

---

## 网关支持的平台对比

| 平台 | 状态 | 复杂度 | 前置条件 | 适合场景 |
|------|------|--------|----------|----------|
| **Telegram** | 已上线 | 低 | Bot Token + User ID | 日常对话、快速查询 |
| **Discord** | 可配置 | 中 | Bot Token, Message Content Intent 启用 | 社区频道、群组协作 |
| **Slack** | 可配置 | 中 | Bot Token, `message.channels` 事件订阅 | 工作团队集成 |
| **WhatsApp** | 可配置 | 高 | Business API / 第三方桥 | 移动端优先场景 |
| **Signal** | 可配置 | 高 | Signal CLI | 隐私优先场景 |
| **Email** | 可配置 | 低 | SMTP/IMAP 配置 | 异步任务、报告推送 |
| **Home Assistant** | 可配置 | 中 | Home Assistant 地址 + Token | 智能家居控制 |
| **Matrix** | 可配置 | 中 | Matrix 账号 + Homeserver | 去中心化通信 |
| **飞书 (Feishu)** | 可配置 | 中 | 飞书开放平台配置 | 国内企业集成 |
| **企业微信 (WeCom)** | 可配置 | 中 | 企业微信配置 | 国内企业集成 |
| **微信 (Weixin)** | 可配置 | 高 | 微信公众平台 | 微信生态集成 |
| **iMessage (BlueBubbles)** | 可配置 | 高 | BlueBubbles 服务 | Apple 生态 |
| **API Server** | 生产就绪 | 低 | 无（默认 :8642） | 自定义前端、Open WebUI |

### 平台选择建议

- **个人日常** → Telegram（最简单、最稳定）
- **开发团队** → Discord 或 Slack（支持频道、线程、@提及）
- **企业办公** → 飞书或企微（国内生态）
- **多设备协同** → API Server（任意 OpenAI 兼容前端）

### 多平台支持情况

根据当前 `channel_directory.json`，Hermes 支持以下平台适配器：

```
telegram, discord, whatsapp, slack, signal, matrix, mattermost,
homeassistant, email, sms, dingtalk, feishu, wecom, wecom_callback,
weixin, bluebubbles, qqbot, yuanbao, google_chat, irc, line, teams
```

---

## 网关故障排除

### 检查日志

```bash
# 查看最近错误
grep -i "failed to send\|error" ~/.hermes/logs/gateway.log | tail -20

# 实时监控
journalctl --user -u hermes-gateway -f -n 50
```

### 常见问题

| 症状 | 原因 | 解决方法 |
|------|------|----------|
| SSH 登出后网关退出 | 未启用 linger | `sudo loginctl enable-linger $USER` |
| WSL2 关闭后网关退出 | 无 systemd | 设置 `systemd=true` 到 `/etc/wsl.conf` |
| 网关崩溃循环 | 持续失败 | `systemctl --user reset-failed hermes-gateway` 重置计数 |
| Discord Bot 沉默 | 缺少 Message Content Intent | 在 Discord Developer Portal 启用 |
| Slack Bot 仅回复私聊 | 未订阅 `message.channels` | Bot Events 中添加该事件 |
| 配置变更不生效 | 未重启 | 执行 `/restart` 或重启 systemd 服务 |
