# 多设备部署

## 概述

Hermes Agent 可以在多台设备上同时运行，并通过 Telegram 网关统一管理。MaoYo42 正在运行三台设备的 Hermes 环境，每台设备扮演不同角色：

| 设备 | 角色 | 操作系统 | 用途 |
|------|------|----------|------|
| **T480s** | 主服务器 | Arch Linux | 7×24 运行，Firecrawl，Cron Job，主力工作 |
| **MacBook Air** | 学术工作站 | macOS | 论文研究，轻量开发，课表/作业管理 |
| **Android 手机** | 移动入口 | Termux | 随时随地问答，移动端管理 |

所有三个 Agent 在同一个 Telegram 群组中互相可见，形成"Agent 团队"。

---

## 一、三设备分工

### 1. T480s — 主力服务器

**硬件：** Lenovo ThinkPad T480s (i7-8550U, 24GB RAM, 512GB SSD)

**职责：**
- 7×24 长期运行（从不关机）
- Firecrawl MCP 始终在线（网页抓取、变化检测）
- 定时任务调度（股票分析、系统巡检）
- Telegram 网关 + systemd 服务
- Obsidian 仓库管理（Git 同步）
- 三设备 Agent 中的"大脑"——最完整的 Skill 库和 MEMORY 数据

**运行方式：** 通过 systemd 用户服务自动启动：

```ini
# ~/.config/systemd/user/hermes.service
[Unit]
Description=Hermes Agent (T480s Master)
After=network.target

[Service]
ExecStart=/usr/bin/hermes run
Restart=on-failure
RestartSec=30
Environment=HOME=$HOME

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable hermes
systemctl --user start hermes
```

### 2. MacBook Air — 学术工作站

**角色：** 学习和研究专用 Agent

**特点：**
- 搭载 Apple Silicon (M芯片)，本地推理效率高
- 存放学术论文、课程材料、研究笔记
- 与 T480s 共享 Telegram 网关
- 通过 Git-sync 共享 Obsidian 知识库
- 不运行 Firecrawl（节省 API 额度）
- 定时任务较少，按需启动

**运行方式：** 手动启动，按需：

```bash
hermes run
```

### 3. Android 手机 — 移动入口

**环境：** Termux + Hermes Agent

**特点：**
- 纯 CLI 界面，通过 Termux 在手机端运行
- 主要用于轻量问答和快速记录
- 不运行 Firecrawl、Dashboard 等后台服务
- 共享同一个 Telegram 群组
- 用于外出时的信息查漏补缺

**运行方式：** Termux 启动后直接：

```bash
cd ~/.hermes
hermes run
```

---

## 二、数据同步

### Git-sync Obsidian

三台设备的知识库（Obsidian Vault）通过 Git 保持同步：

```bash
# 仓库结构
~/obsidian-vault/
├── hermes/         # Agent 生成的笔记和 MEMORY
├── daily/          # 每日记录
├── projects/       # 项目文档
└── .git/
```

同步策略：

```bash
# 每台设备配置 Git 远程后一键同步
cd ~/obsidian-vault
git pull --rebase
# ... 修改文件 ...
git add -A
git commit -m "sync: $(date +%Y-%m-%d %H:%M)"
git push
```

MaoYo42 通过 Hermes Cron Job 每天自动同步一次：

```bash
hermes cron add \
  --name "git-sync" \
  --schedule "0 */4 * * *" \
  --script "执行 Obsidian Vault 的 git pull --rebase && git push，输出同步结果" \
  --notify telegram
```

### 共享 Kanban

Hermes 的 Kanban 功能让三台设备的 Agent 共享同一个任务看板。任一台设备上的 Agent 都可以：

```bash
/kanban add "发布 v0.14 版本" --assignee m1
/kanban move PROJ-42 "进行中"
/kanban done "搭建 CI/CD 流水线"
```

Kanban 数据存储在 Git-sync 的 Vault 中，三台设备共享同一份。

---

## 三、通信方式

### Telegram 群组

三个 Agent 加入同一个 Telegram 群组，群组成员包括：

- **MaoYo42**（你——人类用户）
- **Hermes-T480s**（主 Agent）
- **Hermes-Mac**（Mac Agent）
- **Hermes-Mobile**（手机 Agent）

群组工作方式：

```
MaoYo42: @all 谁有空帮我查一下今天的天气？
Hermes-T480s: 我来。当前北京天气 22°C，晴...
```

或者定向呼叫：

```
MaoYo42: @T480s 帮我跑一下股票分析任务
MaoYo42: @Mac 搜索那篇关于 MLX 的论文
```

每个 Agent 的名字前缀不同，Telegram 网关配置中通过 `agent_name_prefix` 区分。

### SSH 隧道

Mac 和手机端通过 SSH 隧道访问 T480s 上的 Dashboard 和 API：

```bash
# Mac → T480s 隧道（访问 Dashboard）
ssh -N -L 9119:localhost:9119 user@t480s.local

# 手机 → T480s 隧道（Termux + SSH）
ssh -N -L 8648:localhost:8648 user@t480s-ip
```

---

## 四、配置差异

三台设备的 `~/.hermes/config.yaml` 各有侧重：

### T480s（完整版）

```yaml
model:
  provider: deepseek
  default: deepseek-v4-flash

cron:
  enabled: true   # 定时任务全开

tools:
  enabled:
    - web
    - browser
    - terminal
    - file
    - cronjob
    # ... 全部工具集

plugins:
  firecrawl:
    enabled: true   # Firecrawl 常开
```

### Mac（精简版）

```yaml
model:
  provider: deepseek
  default: deepseek-v4-flash

cron:
  enabled: false   # 不运行定时任务

tools:
  enabled:
    - web
    - file
    - memory    # 需要读写共享知识库

plugins:
  firecrawl:
    enabled: false  # 省 API 额度
```

### 手机（极简版）

```yaml
model:
  provider: deepseek
  default: deepseek-v4-flash

cron:
  enabled: false

tools:
  enabled:
    - search   # 只需要搜索能力
    - memory
```

---

## 五、部署检查清单

| 项目 | T480s | Mac | 手机 |
|------|-------|-----|------|
| Hermes Agent | 7×24 systemd | 按需启动 | 按需启动 |
| Telegram 网关 | 7×24 | 按需 | 按需 |
| Firecrawl MCP | 始终运行 | 不运行 | 不运行 |
| Cron Jobs | 全部激活 | 无 | 无 |
| Dashboard | :9119 常开 | 按需 | 不启动 |
| Obsidian Git-sync | 每 4h 自动同步 | 手动 | 手动 |
| Kanban | 共享 | 共享 | 共享 |
| SSH 隧道 | 被连接方 | 主动连接 | 主动连接 |

---

## 六、常见问题

### Q: 三台设备会不会互相冲突？

A: 不会。每个 Agent 有独立的 session 和 MEMORY，仅在共享 Kanban 和 Git Vault 时有交集。Telegram 群组中通过 Agent 名字区分。

### Q: 手机 Termux 运行 Hermes 真的流畅吗？

A: 取决于任务复杂度。简单问答（搜索、查天气、笔记）完全可用。复杂任务（Firecrawl 爬取、长代码生成）建议转给 T480s。

### Q: 如何确保 T480s 7×24 稳定？

A: 详见 [12-tips-and-tricks.md](12-tips-and-tricks.md) 中关于 T480s 电源管理的部分。关键点：限制电池充电 75-80% 以延长寿命，设置 systemd 自动重启。
