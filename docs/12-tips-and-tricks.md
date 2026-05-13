# 技巧与最佳实践

## 概述

本章汇总来自 MaoYo42 实际使用 Hermes Agent 的经验技巧，涵盖效率优化、省钱策略、设备维护等方面。这些技巧并非官方文档内容，而是来自实战踩坑后的积累。

---

## 一、键盘工作流

Hermes Agent 的设计哲学是 **键盘优先**。以下技巧可大幅提升操作效率：

### 1. 在会话中使用 / 命令

```
/skill hermes-agent    # 加载指定 Skill
/tool                  # 查看当前可用工具
/memory                # 查看/编辑 MEMORY.md
/kanban                # 查看看板
/todo                  # 查看任务列表
/clear                 # 清屏
/help                  # 查看所有命令
```

### 2. 善用 tab 补全

Hermes 的 CLI 支持命令和参数 tab 补全（基于 `argcomplete`）。如果未生效：

```bash
# 为当前 Shell 启用
eval "$(register-python-argcomplete hermes)"

# 或加入 ~/.zshrc / ~/.bashrc
eval "$(register-python-argcomplete hermes)"
```

### 3. 快速跳转

```bash
# 打开配置目录
cd ~/.hermes

# 查看日志
less ~/.hermes/logs/hermes.log

# 快速编辑配置
hermes config edit   # 打开 $EDITOR
```

### 4. 终端复用

推荐在 tmux 中运行 Hermes，即使 SSH 断开也能保持会话：

```bash
tmux new -s hermes     # 创建新 session
tmux attach -t hermes  # 重新连接
```

---

## 二、T480s 电源管理

T480s 作为 7×24 服务器，电池健康是关键。

### 电池充电阈值

ThinkPad T480s 支持硬件级充电限制，通过 `tlp` 控制：

```bash
# 安装 tlp
sudo pacman -S tlp

# 设置充电阈值（延长电池寿命的关键）
sudo tlp setcharge 75 80
```

解释：
- `75` — 电量低于 75% 时充电
- `80` — 充到 80% 停止

保持电池在 75-80% 区间，可以显著减缓锂电池老化。MaoYo42 的 T480s 电池已持续运行 2 年以上，容量衰减不到 10%。

### 自动启动检查

```bash
# 在 systemd 服务中加入 tlp 检查
systemctl enable --now tlp

# 验证充电阈值
sudo tlp-stat -b
```

### 其他省电措施

```bash
# 关闭不必要的硬件
sudo systemctl mask bluetooth.service
sudo systemctl mask cups.service  # 如果不需要打印

# 合盖不睡眠（持续运行）
# 编辑 /etc/systemd/logind.conf
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
```

---

## 三、Firecrawl 始终在线

MaoYo42 在 T480s 上通过 systemd 服务保持 Firecrawl MCP 始终运行：

```bash
# firecrawl.service
[Unit]
Description=Firecrawl MCP service
After=network.target

[Service]
ExecStart=/usr/bin/firecrawl-mcp --port 3002
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

Firecrawl 常开的好处：
- Cron Job 中的变化检测任务随时可用
- Agent 使用搜索和抓取时零延迟启动
- 不占用额外启动时间

---

## 四、Top 5 公式

MaoYo42 总结了一套"Top 5 公式"，用于大多数信息获取场景——不追求全面，只求快速获得最有价值的洞察。

### 指令模板

```
搜索 [主题]，分别给出 Top 5：
1. Top 5 关键发现
2. Top 5 关注点（风险/注意事项）
3. Top 5 推荐行动
```

### 实际用例

```
搜索 2025 年大模型开源社区动态，分别给出 Top 5 关键趋势和 Top 5 值得关注的项目
```

```
分析当前 A 股市场，给出 Top 5 热门板块和 Top 5 风险提示
```

这个公式简洁、好记，Agent 每次输出都结构清晰，便于快速浏览。

---

## 五、省钱与 API 管理

### Composio vs MCP

MaoYo42 的实践结论：

| 方案 | 费用 | 适用场景 |
|------|------|----------|
| **MCP**（Model Context Protocol） | 免费（自托管） | **优先使用** |
| **Composio** | 付费（按 API 调用计费） | 仅当 MCP 没有对应工具时 |

**原则：先用 MCP 搞定，不行再上 Composio。**

自托管 MCP Server（如 Firecrawl MCP）完全免费，没有每千次调用计费的问题。

### npm 镜像加速

在安装 hermes-web-ui 等 Node.js 项目时，使用中国 npm 镜像：

```bash
# 淘宝镜像（最快）
npm config set registry https://registry.npmmirror.com

# 或单次安装使用
npm install --registry=https://registry.npmmirror.com
```

### 控制 API 消耗

1. **委派任务控制在 3 并发** — 更多并发不会提高质量，只会烧钱
2. **Cron Job 不要太密** — 最短 30 分钟，再短意义不大
3. **Firecrawl 变化检测** — 只监控真正关心的页面，不要什么都监控
4. **在 Mac/手机端关掉 Firecrawl** — 三台设备同时跑 Firecrawl 毫无意义

---

## 六、预算优先，开源优先

MaoYo42 的软件选择原则：

```
开源自托管 > 付费 SaaS
一次性买断 > 订阅制
社区版 > 企业版
```

| 场景 | 开源方案 | 避开的付费方案 |
|------|----------|---------------|
| Web 搜索 | Firecrawl（自托管） | Composio 搜索 API |
| 代码执行 | 内置沙箱 | Composio 代码执行 |
| 文件工具 | Hermes 内置 | Composio 文件操作 |
| 浏览器自动化 | Hermes 内置 Browser | Browserbase（付费） |
| AI 模型 | DeepSeek API（按量，$0.15/M tokens） | Claude Pro / GPT Plus 订阅 |
| Skill 托管 | GitHub Tap 源 | 自有 Hub 付费空间 |

---

## 七、其他实用技巧

### Agent 直呼其名

在 Telegram 群组中，可以用 `@AgentName` 定向呼叫特定设备上的 Agent。名字在网关配置中设置：

```yaml
gateways:
  telegram:
    agent_name: "hermes-t480s"
```

### Session 快速切换

```bash
# 列出所有活跃 session
hermes sessions list

# 切换到某个 session
hermes sessions select <id>
```

### 用好 MEMORY.md

MaoYo42 在 `MEMORY.md` 中保存了个人偏好信息，让 Agent 不需要每次都问：

- 使用 DeepSeek v4 Flash 作为主力模型
- 工作在 UTC+8 时区
- 优先用中文回答
- Obsidian Vault 路径：`/home/maoyo/obsidian-vault`

Agent 每次启动时会自动读取 MEMORY.md，这些偏好就变成了"默认设置"。

---

## 八、一行诊断

当你怀疑 Hermes 配置有问题时，运行：

```bash
hermes doctor
```

这个命令会检查：
- 配置文件语法
- 环境变量是否齐全
- 模型 API 是否可达
- MCP 服务是否在线
- 文件权限是否正确

如果一切正常，输出类似：

```
✓ config.yaml 语法正确
✓ .env 密钥完整 (5/5)
✓ DeepSeek API 连通性正常 (latency: 230ms)
✓ Firecrawl MCP 正常运行 (端口 3002)
✓ 日志目录可写
```

---

## 九、快速参考

```bash
# 日常启动
hermes run

# 遇到问题
hermes doctor            # 诊断
hermes config validate   # 校验配置
hermes --help             # 帮助

# 查看版本
hermes --version

# 升级
pip install -U hermes-agent

# 清理日志（日志文件可能很大）
hermes logs trim --keep 7   # 只保留 7 天
```
