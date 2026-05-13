# Telegram 网关集成指南

## 概述

Telegram 网关允许你的 AI Agent 通过 Telegram Bot 接收消息和发送通知，实现人机交互的即时通讯通道。

## 架构说明

```
User ---> Telegram Bot ---> Telegram API ---> Agent (Polling)
User <--- Telegram Bot <--- Telegram API <--- Agent (Send)
```

Agent 使用**轮询（Polling）**模式从 Telegram 服务器获取新消息，而不是 Webhook 模式。这适合运行在无公网 IP 的环境（如开发机、内网服务器）。

## 前置条件

- 一个 Telegram 账号
- 能够访问 https://t.me/botfather（创建 Bot）

## 第一步：通过 BotFather 创建 Bot

1. 在 Telegram 中搜索 **@BotFather** 或访问 https://t.me/botfather
2. 发送 `/newbot` 命令
3. 按照提示设置 bot 名称（显示名，如 `My Hermes Agent`）
4. 设置 bot 用户名（必须以 `bot` 结尾，如 `my_hermes_agent_bot`）
5. 创建成功后，BotFather 会返回一个 **API Token**，格式如：
   ```
   1234567890:ABCdefGHIjklmNOPqrstUVwxyz-1234567
   ```
6. **立即保存这个 Token** — 关闭对话框后将无法再次查看

### 可选：配置 Bot 信息

```
/setdescription  - 设置 Bot 描述
/setabouttext    - 设置 Bot 简介
/setuserpic      - 设置头像
/setcommands     - 设置命令列表
```

### 推荐的命令列表

```
start - 开始使用
help - 获取帮助
status - 查看系统状态
```

设置方法：发送 `/setcommands` 给 BotFather，然后粘贴以上内容。

## 第二步：获取你的用户 ID

Telegram Bot 无法直接知道谁在发消息——你需要获取自己的 Telegram User ID 才能让 Bot 识别你。

### 方法 A：使用 @userinfobot（最简单）

1. 在 Telegram 中搜索 **@userinfobot**
2. 点击 Start 或发送任意消息
3. Bot 会回复你的信息，其中包含：
   ```
   Id: 123456789
   First: Your Name
   ```
4. 记下这个 **Id** 数字

### 方法 B：使用 @getidsbot

1. 搜索 **@getidsbot**
2. 发送 `/start`
3. 返回信息中包含 `Your user ID: 123456789`

### 方法 C：通过 API 自查

启动你的 Bot 后，给自己 Bot 发一条消息，然后访问：

```
https://api.telegram.org/bot<你的TOKEN>/getUpdates
```

返回 JSON 中的 `message.from.id` 就是你的用户 ID。

## 第三步：配置环境变量

在 Hermes Agent 的环境配置文件（如 `~/.hermes/.env`）中添加：

```bash
# Telegram Bot 配置
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklmNOPqrstUVwxyz-1234567
TELEGRAM_ALLOWED_USERS=123456789           # 允许使用的用户 ID，逗号分隔
TELEGRAM_POLLING_INTERVAL=2                # 轮询间隔（秒），默认 2
TELEGRAM_POLLING_TIMEOUT=30                # 长轮询超时（秒），默认 30
```

### 参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `TELEGRAM_BOT_TOKEN` | BotFather 颁发的 API Token | 必填 |
| `TELEGRAM_ALLOWED_USERS` | 允许交互的用户 ID 列表 | 必填（安全考虑） |
| `TELEGRAM_POLLING_INTERVAL` | 两次轮询之间的间隔 | 2 秒 |
| `TELEGRAM_POLLING_TIMEOUT` | 长轮询超时，减少无效请求 | 30 秒 |

## 第四步：集成轮询到 Hermes Agent

### 方式 1：使用 Hermes Agent Skills

创建一个 Telegram 网关技能 `skills/telegram-gateway/SKILL.md`：

```markdown
---
name: telegram-gateway
description: Telegram Bot 消息网关，支持接收用户消息并回复
version: 1.0.0
author: Hermes Agent
tags: [telegram, messaging, gateway]
---

# Telegram Gateway Skill

## 环境变量
- TELEGRAM_BOT_TOKEN: Bot API token
- TELEGRAM_ALLOWED_USERS: 逗号分隔的允许用户 ID
- TELEGRAM_POLLING_INTERVAL: 轮询间隔（秒，默认 2）

## 轮询逻辑
1. 使用长轮询（offset 机制）获取新消息
2. 过滤不在 ALLOWED_USERS 列表中的用户
3. 调用 on_message 处理消息
4. 通过 sendMessage API 回复

## 示例：手动测试
curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d "chat_id=123456789" \
  -d "text=Hello from Hermes Agent!"
```

### 方式 2：独立轮询脚本

创建一个 Python 轮询脚本 `telegram_poll.py`：

```python
#!/usr/bin/env python3
"""Telegram Bot 轮询网关 - 连接 Hermes Agent"""
import requests
import time
import os
import json

BOT_TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN")
ALLOWED_USERS = os.environ.get("TELEGRAM_ALLOWED_USERS", "").split(",")
POLL_INTERVAL = int(os.environ.get("TELEGRAM_POLLING_INTERVAL", "2"))
BASE_URL = f"https://api.telegram.org/bot{BOT_TOKEN}"

def get_updates(offset=None):
    params = {"timeout": 30}
    if offset:
        params["offset"] = offset
    resp = requests.get(f"{BASE_URL}/getUpdates", params=params, timeout=35)
    return resp.json()

def send_message(chat_id, text):
    requests.post(f"{BASE_URL}/sendMessage", json={
        "chat_id": chat_id,
        "text": text,
        "parse_mode": "Markdown"
    })

def on_message(user_id, chat_id, text):
    """处理收到的消息 - 可分发给Agent"""
    print(f"[MSG] User {user_id}: {text}")
    # 此处将消息发送给 Agent 处理
    send_message(chat_id, f"收到: {text}")

def main():
    print(f"[INFO] Telegram Polling Bot Started")
    print(f"[INFO] Allowed users: {ALLOWED_USERS}")
    last_update_id = 0

    while True:
        try:
            updates = get_updates(offset=last_update_id + 1)
            if updates.get("ok") and updates.get("result"):
                for update in updates["result"]:
                    update_id = update["update_id"]
                    if update_id <= last_update_id:
                        continue
                    last_update_id = update_id

                    if "message" in update:
                        msg = update["message"]
                        user_id = str(msg["from"]["id"])
                        chat_id = msg["chat"]["id"]
                        text = msg.get("text", "")

                        if user_id in ALLOWED_USERS:
                            on_message(user_id, chat_id, text)
                        else:
                            print(f"[WARN] Unauthorized user: {user_id}")
                            send_message(chat_id, "抱歉，你无权使用此 Bot。")

            time.sleep(POLL_INTERVAL)
        except KeyboardInterrupt:
            print("\n[INFO] Shutting down...")
            break
        except Exception as e:
            print(f"[ERROR] {e}")
            time.sleep(5)

if __name__ == "__main__":
    main()
```

运行：`python3 telegram_poll.py`

## 第五步：安全加固

### 用户白名单

务必设置 `TELEGRAM_ALLOWED_USERS`，只允许特定用户 ID 与 Bot 交互。建议格式：

```bash
TELEGRAM_ALLOWED_USERS=123456789,987654321
```

### 消息限制

- 设置消息长度上限（如 4096 字符——Telegram 单条消息上限）
- 设置速率限制（如每分钟最多处理 20 条消息）
- 记录所有消息用于审计

### 隐私保护

- 不要在日志中记录完整的消息内容（可截断或 hash）
- Bot Token 视为敏感凭据，不要硬编码在源码中

## 进阶用法

### 发送富文本消息

```bash
curl -s "https://api.telegram.org/bot${TOKEN}/sendMessage" \
  -d "chat_id=123456789" \
  -d "text=*加粗* _斜体_ \`代码\`" \
  -d "parse_mode=MarkdownV2"
```

### 发送图片

```bash
curl -s "https://api.telegram.org/bot${TOKEN}/sendPhoto" \
  -F "chat_id=123456789" \
  -F "photo=@/path/to/image.jpg" \
  -F "caption=图片说明"
```

### 键盘回复

```python
send_message(chat_id, "请选择操作：")
requests.post(f"{BASE_URL}/sendMessage", json={
    "chat_id": chat_id,
    "text": "请选择操作：",
    "reply_markup": {
        "keyboard": [["查看状态", "执行任务"], ["帮助"]],
        "resize_keyboard": True
    }
})
```

## API 参考

Telegram Bot API 核心端点：

| 端点 | 用途 |
|------|------|
| `getUpdates` | 获取新消息（轮询） |
| `sendMessage` | 发送消息 |
| `sendPhoto` | 发送图片 |
| `sendDocument` | 发送文件 |
| `editMessageText` | 编辑已发送消息 |
| `deleteMessage` | 删除消息 |
| `getMe` | 测试 Token 是否有效 |

## 故障排除

| 问题 | 解决方案 |
|------|----------|
| `getUpdates` 返回空 | 先向 Bot 发送一条消息 |
| `403 Forbidden` | Bot 被用户屏蔽，或 Token 无效 |
| `chat not found` | 用户必须先与 Bot 对话一次 |
| 消息过长 | Telegram 限制单条 4096 字符，需分块发送 |
| 轮询无响应 | 检查网络是否可访问 api.telegram.org |

---

**参考资料：**
- Telegram Bot API 文档：https://core.telegram.org/bots/api
- BotFather 帮助：https://t.me/botfather
- @userinfobot：https://t.me/userinfobot
