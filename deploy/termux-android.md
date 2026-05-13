# Termux Android 部署指南 (移动端)

> 适用环境：Android / Termux / 代号「姐妹」/ 电池友好 / 端口转发

## 1. 系统要求

- **设备**: Android 手机/平板
- **系统**: Android 10+
- **Termux**: F-Droid 版本（推荐）或 GitHub Release
- **存储**: 建议至少 2GB 可用空间

## 2. 安装 Termux

### 2.1 从 F-Droid 安装（推荐）

F-Droid 版 Termux 维护最活跃，后台保活更好：

1. 下载 F-Droid: https://f-droid.org
2. 搜索并安装 **Termux**、**Termux:API**、**Termux:Boot**
3. 授予 Termux 通知权限（关键！防杀后台）

### 2.2 从 GitHub 安装

备用方案：从 Termux 官方 GitHub Releases 下载 APK：
https://github.com/termux/termux-app/releases

## 3. 初始配置

### 更新包管理器并安装基础工具

```bash
pkg update && pkg upgrade -y

pkg install -y git curl wget openssh termux-api \
  termux-services tmux nodejs python \
  which proot
```

### 存储权限

```bash
termux-setup-storage
# 在 Android 弹窗中点击「允许」
```

### 安装 Hermes Agent

```bash
cd ~
git clone https://github.com/NousResearch/hermes-agent
cd hermes-agent
# 按官方文档完成安装
```

## 4. Shell 环境配置

编辑 `~/.bashrc`（或 `~/.zshrc` 如使用 Zsh）：

```bash
# Hermes 别名
alias h="hermes"
alias hc="hermes --config"
alias hl="hermes logs"
alias hst="hermes status"

# 数据目录
export HERMES_CONFIG_DIR="$HOME/.config/hermes"
export HERMES_DATA_DIR="$HOME/.local/share/hermes"
```

## 5. 后台保活方案

### 5.1 Termux:Boot（推荐）

Termux:Boot 可在 Android 开机时自动启动 Termux 并执行脚本。

1. 安装 Termux:Boot
2. 创建启动脚本：

```bash
mkdir -p ~/.termux/boot
cat > ~/.termux/boot/hermes-start.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
# Hermes Agent 自动启动
termux-wake-lock
cd ~/hermes-agent
hermes --serve &
EOF

chmod +x ~/.termux/boot/hermes-start.sh
```

### 5.2 tmux 会话（手动保活）

```bash
# 创建 tmux 会话
tmux new -s hermes

# 在 tmux 中启动 Hermes
cd ~/hermes-agent
hermes --serve

# 分离会话: Ctrl+B, D
# 重新接入: tmux attach -t hermes
```

### 5.3 使用 termux-services

```bash
# 安装后可使用类似 systemd 的服务管理
# 创建服务脚本（根据 termux-services 语法）
```

## 6. 端口转发方案

### 6.1 本地端口转发 (ssh -L)

从 Linux 笔记本或 Mac 转发到 Android 端：

```bash
# 在 Android Termux 中启动 SSH
sshd

# 查看 Android IP（在同一 WiFi 下）
ifconfig
# 或
ip addr show
```

在 Linux/Mac 端执行端口转发：

```bash
# 将 Android 端 Hermes 端口转发到本地
ssh -L 8080:localhost:8080 -N -p 8022 user@android-ip
```

### 6.2 使用 Termux:API 获取网络信息

```bash
# 获取 WiFi IP
termux-wifi-connectioninfo

# 获取本机信息
termux-info
```

### 6.3 反向隧道（从 Android 访问远程服务）

```bash
# 如果 Android 在 NAT 后，可通过有公网 IP 的服务器建立反向隧道
ssh -R 9090:localhost:8080 user@public-server
```

## 7. 电池友好优化

移动端电源管理是核心考量。

### 7.1 省电配置

```bash
# 1. 不启动自动推理 —— 仅按需运行
# 2. 减少日志写入频率
# 3. 降低模型推理并发数
```

在 Hermes 配置中建议：

```yaml
# ~/.config/hermes/config.yaml（参考示例）
battery:
  power_save: true
  idle_timeout_seconds: 300  # 5分钟无操作自动暂停
  max_concurrent_requests: 1
  disable_auto_download: true
```

### 7.2 使用 termux-wake-lock

仅在需要时持有唤醒锁：

```bash
# 获取唤醒锁（防止息屏后进程被挂起）
termux-wake-lock

# Hermes 运行完毕后释放
termux-wake-unlock
```

### 7.3 Android 系统级省电设置

1. **设置 → 应用 → Termux → 电池 → 不受限制**（防止系统杀进程）
2. **设置 → 应用 → Termux → 通知 → 允许**（保持存活）
3. **关闭「电池优化」** 对 Termux
4. **关闭「后台限制」** 对 Termux

### 7.4 按需运行脚本

```bash
# 辅助脚本：按需启动/停止 Hermes
cat > ~/hermes-ctl.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash

case "$1" in
  start)
    termux-wake-lock
    cd ~/hermes-agent
    hermes --serve &
    echo "Hermes 已启动"
    ;;
  stop)
    pkill -f "hermes --serve"
    termux-wake-unlock
    echo "Hermes 已停止"
    ;;
  status)
    if pgrep -f "hermes --serve" > /dev/null; then
      echo "Hermes 运行中"
    else
      echo "Hermes 未运行"
    fi
    ;;
  *)
    echo "用法: ~/hermes-ctl.sh {start|stop|status}"
    ;;
esac
EOF

chmod +x ~/hermes-ctl.sh
```

## 8. 三端 Git 同步

```bash
cd ~/.config/hermes
git init
git remote add origin <your-repo-url>
git config user.email "android@termux"
git config user.name "姐妹"
git pull origin main  # 拉取 Linux/Mac 端配置
git add .
git commit -m "Android 端同步"
git push -u origin main
```

## 9. 与主设备协同工作流

```
Linux (T480s)  ←→  MacBook (赫妹)  ←→  Android (姐妹)
     │                   │                    │
     └───────────────────┴────────────────────┘
                         │
                   Git 同步 (Obsidian vault + Hermes 配置)
```

典型场景：
- **在家**: 主力 T480s + 外接显示器
- **外出（学术会议）**: MacBook
- **通勤/碎片时间**: Android 手机快速查阅、语音输入

## 10. 故障排查

| 问题 | 可能原因 | 解决办法 |
|------|---------|---------|
| Termux 被系统杀死 | 电池优化未关闭 | 设置中关闭 Termux 电池优化 |
| SSH 连接拒绝 | SSH 未启动或端口未开 | `sshd` 检查，`pkg reinstall openssh` |
| 唤醒锁不生效 | Termux:API 未安装 | `pkg install termux-api` |
| Git 同步冲突 | 多端同时修改 | `git mergetool` 解决冲突 |
| 存储无法访问 | 未运行 termux-setup-storage | 运行后授予权限 |
| 代理连接失败 | 移动网络限制 | 尝试切换 WiFi/移动数据 |

---

*最后更新: 2026-05-14*
