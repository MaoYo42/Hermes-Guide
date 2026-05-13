# Linux 服务器部署指南 (Arch Linux · ThinkPad T480s)

> 适用环境：Arch Linux / ThinkPad T480s / systemd 用户服务 / Fish shell / FlClash 代理

## 1. 系统要求

- **发行版**: Arch Linux（已更新至最新）
- **硬件**: ThinkPad T480s (Intel 8th Gen)
- **Shell**: Fish (>= 3.6)
- **服务管理**: systemd --user
- **代理**: FlClash (TUN 模式 / HTTP 混合代理)

## 2. 安装依赖

```bash
sudo pacman -Syu
sudo pacman -S --needed git curl wget base-devel fish
```

### 安装 Hermes Agent

从 Nous Research 官方仓库获取：

```bash
cd ~
git clone https://github.com/NousResearch/hermes-agent
cd hermes-agent
# 根据官方 README 执行安装
# 例如使用 pip 或 npm，以实际文档为准
```

## 3. Fish Shell 配置

添加 Hermes 别名和环境变量到 `~/.config/fish/config.fish`：

```fish
# Hermes Agent 配置
set -x HERMES_CONFIG_DIR ~/.config/hermes
set -x HERMES_DATA_DIR ~/.local/share/hermes

# 快捷命令
alias h='hermes'
alias hc='hermes --config'
alias hl='hermes logs'
alias hst='hermes status'

# 如果使用 FlClash HTTP 代理
set -x HTTP_PROXY http://127.0.0.1:7890
set -x HTTPS_PROXY http://127.0.0.1:7890
set -x NO_PROXY localhost,127.0.0.1,.local

# 提示符增强
function fish_greeting
    echo "🐚 Hermes Agent 就绪 — $(date '+%Y-%m-%d %H:%M')"
end
```

重载配置：

```bash
source ~/.config/fish/config.fish
```

## 4. FlClash 代理配置

FlClash 运行于 TUN 模式，提供 HTTP 混合代理端口 `7890`。

> 注意：Hermes Agent 如需联网下载模型或更新，建议在启动前确保 FlClash 已运行。

验证代理连通性：

```bash
curl -x http://127.0.0.1:7890 -s -o /dev/null -w "%{http_code}" https://www.google.com
# 应返回 200
```

## 5. systemd 用户服务

创建用户级 systemd 服务，使 Hermes Agent 开机自启、自动保活。

### 服务单元文件

创建 `~/.config/systemd/user/hermes-agent.service`：

```ini
[Unit]
Description=Hermes Agent — AI Assistant Service
Documentation=https://hermes-agent.nousresearch.com/docs
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=%h/hermes-agent/hermes --serve
Restart=on-failure
RestartSec=5
Environment="PATH=%h/.local/bin:/usr/local/bin:/usr/bin"
Environment="HOME=%h"
# 如果使用 FlClash 代理，取消注释以下三行
# Environment="HTTP_PROXY=http://127.0.0.1:7890"
# Environment="HTTPS_PROXY=http://127.0.0.1:7890"
# Environment="NO_PROXY=localhost,127.0.0.1,.local"

# 日志管理
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```

### 启用并启动服务

```bash
# 重新加载 systemd 用户实例
systemctl --user daemon-reload

# 启用开机自启
systemctl --user enable hermes-agent.service

# 立即启动
systemctl --user start hermes-agent.service

# 查看状态
systemctl --user status hermes-agent.service

# 查看实时日志
journalctl --user -u hermes-agent.service -f
```

### 常用 systemd 命令

```bash
# 重启服务
systemctl --user restart hermes-agent.service

# 停止服务
systemctl --user stop hermes-agent.service

# 禁用开机自启
systemctl --user disable hermes-agent.service
```

## 6. 日常运维

### 查看日志

```bash
# 最近 50 条日志
journalctl --user -u hermes-agent.service -n 50 --no-pager

# 追踪实时日志
journalctl --user -u hermes-agent.service -f

# 按时间过滤（最近 1 小时）
journalctl --user -u hermes-agent.service --since "1 hour ago"
```

### 更新 Hermes Agent

```bash
cd ~/hermes-agent
git pull
# 根据官方文档重新安装/构建
# systemctl --user restart hermes-agent.service
```

### 资源监控 (ThinkPad T480s)

```bash
# 查看内存使用
ps aux | grep hermes

# 查看 CPU 占用
top -p $(pgrep -f hermes)

# 磁盘占用（数据目录）
du -sh ~/.local/share/hermes
```

## 7. 故障排查

| 问题 | 可能原因 | 解决办法 |
|------|---------|---------|
| 服务启动失败 | 端口被占用 | `ss -tlnp \| grep <port>` 检查端口 |
| 代理连接错误 | FlClash 未运行 | `systemctl --user status flclash`（如适用） |
| 日志无输出 | systemd 日志限制 | `journalctl --vacuum-size=100M` |
| 开机不自启 | user service 未 enable | `systemctl --user enable hermes-agent.service` |
| 高 CPU 占用 | 后台推理任务 | `htop` 查看，考虑限制并发 |

## 8. ThinkPad T480s 电池优化提示

由于 T480s 是笔记本，如需省电：

```bash
# 安装 TLP 电源管理
sudo pacman -S tlp
sudo systemctl enable tlp.service
sudo systemctl start tlp.service

# 在不需要时暂停 Hermes 服务
systemctl --user stop hermes-agent.service
# 需要时再启动
systemctl --user start hermes-agent.service
```

---

*最后更新: 2026-05-14*
