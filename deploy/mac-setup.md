# macOS 部署指南 (学术用 · MacBook)

> 适用环境：MacBook / macOS (Ventura+) / 学术研究使用 / 代号「赫妹」

## 1. 系统要求

- **设备**: MacBook (Apple Silicon 或 Intel)
- **系统**: macOS Ventura (13.0) 或更高
- **Shell**: Zsh (默认) 或 Bash
- **网络**: 校园网 / VPN / 学术代理

## 2. 安装 Homebrew

macOS 推荐使用 Homebrew 管理包依赖：

```bash
# 安装 Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 添加到 PATH（Apple Silicon Mac）
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

安装必要工具：

```bash
brew install git curl wget
```

## 3. 安装 Hermes Agent

```bash
cd ~
git clone https://github.com/NousResearch/hermes-agent
cd hermes-agent
# 按官方文档完成安装
```

## 4. Shell 环境配置 (Zsh)

编辑 `~/.zshrc`：

```zsh
# Hermes Agent 别名
alias h="hermes"
alias hc="hermes --config"
alias hl="hermes logs"
alias hst="hermes status"

# 学术网络代理（如使用校园网 HTTP 代理）
# 取消注释并根据实际情况修改
# export HTTP_PROXY=http://proxy.xxx.edu.cn:8080
# export HTTPS_PROXY=http://proxy.xxx.edu.cn:8080
# export NO_PROXY=localhost,127.0.0.1,.edu.cn

# 数据目录
export HERMES_CONFIG_DIR="$HOME/.config/hermes"
export HERMES_DATA_DIR="$HOME/.local/share/hermes"
```

重载配置：

```bash
source ~/.zshrc
```

## 5. 学术使用场景配置

### 5.1 目录结构建议

```
~/academic/
├── papers/          # PDF 论文
├── notes/           # 研究笔记（可与 Obsidian 联动）
├── data/            # 数据集
├── experiments/     # 实验代码
└── hermes-workspace/ # Hermes 工作区
```

创建并配置工作区：

```bash
mkdir -p ~/academic/hermes-workspace
```

### 5.2 与 Obsidian 联动 (Git 同步)

Hermes 可与 Obsidian 知识库配合使用。典型工作流：

1. 在 Obsidian 中记笔记，存储在 `~/obsidian-vault/`
2. Hermes 读取 vault 内容辅助思考
3. 通过 Git 在 MacBook、Linux、Android 三端同步

配置建议：

```bash
# 设置 Obsidian vault 指向
export HERMES_OBSIDIAN_VAULT="$HOME/obsidian-vault"
```

### 5.3 校园网代理配置

如果所在学术机构的网络需要代理：

```bash
# 在 ~/.zshrc 中添加
export HERMES_NETWORK_PROXY="auto"  # 自动检测
# 或手动指定
# export HTTP_PROXY="http://127.0.0.1:7890"
# export HTTPS_PROXY="http://127.0.0.1:7890"
```

## 6. 后台运行 (LaunchAgent)

macOS 推荐使用 LaunchAgent 实现后台驻留。

### 创建 plist 文件

`~/Library/LaunchAgents/com.nousresearch.hermes.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.nousresearch.hermes</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/hermes</string>
        <string>--serve</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>

    <key>WorkingDirectory</key>
    <string>/Users/用户名/hermes-agent</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key>
        <string>/Users/用户名</string>
    </dict>

    <key>StandardOutPath</key>
    <string>/Users/用户名/Library/Logs/hermes-stdout.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/用户名/Library/Logs/hermes-stderr.log</string>
</dict>
</plist>
```

> 注意：将 `用户名` 替换为实际的 macOS 用户名。

### 加载 LaunchAgent

```bash
# 加载（启动 + 开机自启）
launchctl load ~/Library/LaunchAgents/com.nousresearch.hermes.plist

# 查看状态
launchctl list | grep hermes

# 卸载（停止 + 取消自启）
launchctl unload ~/Library/LaunchAgents/com.nousresearch.hermes.plist
```

## 7. 电源管理 (笔记本优化)

MacBook 在电池供电时的优化建议：

```bash
# 查看能量影响
top -o power

# 在电池模式下暂停 Hermes（手动）
launchctl unload ~/Library/LaunchAgents/com.nousresearch.hermes.plist
# 插电后重新加载
launchctl load ~/Library/LaunchAgents/com.nousresearch.hermes.plist
```

使用 `pmset` 查看电源状态：

```bash
pmset -g batt
```

## 8. 三端 Git 同步

Hermes 配置文件可通过 Git 在 Linux/Mac/Android 三端同步：

```bash
cd ~/.config/hermes
git init
git remote add origin <your-repo-url>
git add .
git commit -m "初始 Hermes 配置"
git push -u origin main
```

建议同步的内容：
- `~/.config/hermes/` — Hermes 配置
- `~/obsidian-vault/` — 知识库（独立仓库）
- Hermes 工作区文件

## 9. 故障排查

| 问题 | 可能原因 | 解决办法 |
|------|---------|---------|
| LaunchAgent 未启动 | plist 路径错误 | 检查 `~/Library/LaunchAgents/` 是否存在 |
| 权限错误 | SIP / 沙箱限制 | 确保路径在允许范围内 |
| 代理无法连接 | 校园网需要认证 | 检查 HTTP_PROXY 设置 |
| 高能耗 | 持续推理 | 考虑只在需要时启动服务 |
| 日志丢失 | 日志目录未创建 | `mkdir -p ~/Library/Logs` |

---

*最后更新: 2026-05-14*
