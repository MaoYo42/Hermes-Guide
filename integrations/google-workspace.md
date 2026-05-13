# Google Workspace 集成指南

## 概述

`gws`（Google Workspace CLI）是 Google 官方推出的统一命令行工具，用于管理 Drive、Gmail、Calendar、Sheets、Docs、Chat 等所有 Google Workspace 服务。它从 Google Discovery Service 动态构建命令，当 Google Workspace 新增 API 时自动适配。

GitHub: https://github.com/googleworkspace/cli

**注意：** 这不是 Google 官方支持的正式产品。

## 安装

### 方式 1：下载预编译二进制（推荐）

从 GitHub Releases 页面下载对应系统的二进制文件：

```bash
# 以 Linux amd64 为例
curl -LO https://github.com/googleworkspace/cli/releases/latest/download/gws-linux-amd64.tar.gz
tar xzf gws-linux-amd64.tar.gz
sudo mv gws /usr/local/bin/
gws --version
```

### 方式 2：npm 安装

```bash
npm install -g @googleworkspace/cli
```

### 方式 3：Cargo 编译

```bash
cargo install --git https://github.com/googleworkspace/cli --locked
```

### 方式 4：Homebrew（macOS / Linux）

```bash
brew install googleworkspace-cli
```

### 方式 5：Nix Flake

```bash
nix run github:googleworkspace/cli
```

## 认证设置

`gws` 支持多种认证方式：

### 快速设置（推荐 — 需 gcloud CLI）

```bash
gws auth setup
```

此命令会自动：
1. 创建或选择 Google Cloud 项目
2. 启用必要的 API
3. 创建 OAuth 凭据
4. 打开浏览器完成 OAuth 登录

### 手动 OAuth 设置（无 gcloud）

**第一步：创建 OAuth 客户端**

1. 打开 Google Cloud Console → API 和服务 → 凭据
2. 创建 OAuth 客户端 ID → 选择"桌面应用"
3. 下载客户端 JSON 保存到 `~/.config/gws/client_secret.json`

**第二步：添加测试用户**

1. 打开 OAuth 同意屏幕
2. 在"测试用户"中添加你的 Google 账号邮箱
3. 否则登录会显示"Access blocked"

**第三步：登录**

```bash
gws auth login
```

也可指定所需 scope（推荐，避免测试模式 scope 超限）：

```bash
gws auth login -s drive,gmail,sheets,calendar
```

### 服务账号认证（服务器/CI）

```bash
export GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE=/path/to/service-account.json
gws drive files list
```

### 预获取的 Token

```bash
export GOOGLE_WORKSPACE_CLI_TOKEN=$(gcloud auth print-access-token)
gws drive files list
```

### 无头环境（导出凭据）

在可浏览器登录的机器上完成认证后导出：

```bash
gws auth export --unmasked > credentials.json
```

在无头机器上使用：

```bash
export GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE=/path/to/credentials.json
gws drive files list
```

## Hermes Agent 集成

### 方式 1：直接在 Agent 中调用 gws

在技能或工作流中直接调用 `gws` 命令，输出为 JSON：

```bash
gws drive files list --params '{"pageSize": 5}'
gws gmail users messages list --params '{"userId": "me", "maxResults": 3}'
gws calendar events list --params '{"calendarId": "primary", "maxResults": 5}'
```

### 方式 2：安装 gws Agent Skills

`gws` 仓库内置 100+ Agent Skill，直接安装到 Hermes Agent：

```bash
# 安装所有 gws skills
npx skills add https://github.com/googleworkspace/cli

# 或只安装需要的部分
npx skills add https://github.com/googleworkspace/cli/tree/main/skills/gws-gmail
npx skills add https://github.com/googleworkspace/cli/tree/main/skills/gws-drive
npx skills add https://github.com/googleworkspace/cli/tree/main/skills/gws-calendar
npx skills add https://github.com/googleworkspace/cli/tree/main/skills/gws-sheets
```

对于 OpenClaw 或支持 symlink 的 Agent：

```bash
ln -s $(pwd)/skills/gws-* ~/.hermes/skills/
```

### 方式 3：创建自定义 gws 技能

创建 `skills/gws-workflow/SKILL.md`：

```markdown
---
name: gws-workflow
description: 通过 gws CLI 操作 Google Workspace
version: 1.0.0
author: Hermes Agent
tags: [google, workspace, gmail, drive, calendar]
---

# Google Workspace Workflow

## 前置条件
- `gws` CLI 已安装并认证
- `GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE` 已设置或 `gws auth login` 已完成

## 常用操作

### Gmail
- `gws gmail +triage` - 查看未读邮件摘要
- `gws gmail +send --to user@example.com --subject "主题" --body "内容"` - 发送邮件
- `gws gmail users messages list --params '{"userId": "me", "maxResults": 10}'` - 列出最近邮件

### Drive
- `gws drive files list --params '{"pageSize": 10}'` - 列出最近文件
- `gws drive +upload ./doc.pdf --name "文档"` - 上传文件
- `gws drive files get --params '{"fileId": "xxxx"}'` - 获取文件信息

### Calendar
- `gws calendar +agenda` - 查看今日日程
- `gws calendar +insert --summary "会议" --start "2026-05-15T10:00:00"` - 创建事件

### Sheets
- `gws sheets +append --spreadsheet SPREADSHEET_ID --values "数据1,数据2"` - 追加行
- `gws sheets +read --spreadsheet SPREADSHEET_ID` - 读取表格
```

## 常用命令参考

### Drive 文件管理

```bash
# 列出文件
gws drive files list --params '{"pageSize": 10}'

# 创建文件（上传）
gws drive files create --json '{"name": "report.pdf"}' --upload ./report.pdf

# 获取文件内容
gws drive files get --params '{"fileId": "FILE_ID"}'

# 搜索文件
gws drive files list --params '{"q": "name contains \"report\"", "pageSize": 5}'
```

### Gmail

```bash
# 查看未读邮件摘要（helper 命令）
gws gmail +triage

# 发送邮件
gws gmail +send --to alice@example.com --subject "Hello" --body "Hi there"

# 回复邮件
gws gmail +reply --message-id MESSAGE_ID --body "Thanks!"

# 列出消息
gws gmail users messages list --params '{"userId": "me", "maxResults": 5}'
```

### Calendar

```bash
# 查看今日日程（使用账号时区）
gws calendar +agenda

# 创建事件
gws calendar +insert --summary "项目会议" --start "2026-05-15T10:00:00" --end "2026-05-15T11:00:00"

# 列出事件
gws calendar events list --params '{"calendarId": "primary", "maxResults": 5}'
```

### Sheets

```bash
# 追加数据
gws sheets +append --spreadsheet SPREADSHEET_ID --values "张三,95"

# 读取数据
gws sheets +read --spreadsheet SPREADSHEET_ID

# 读取指定范围
gws sheets spreadsheets values get \
  --params '{"spreadsheetId": "ID", "range": "Sheet1!A1:C10"}'
```

### Chat

```bash
# 发送消息到空间
gws chat +send --space "spaces/xxx" --text "部署完成！"

# 列出空间
gws chat spaces list
```

## 环境变量参考

| 变量 | 说明 |
|------|------|
| `GOOGLE_WORKSPACE_CLI_TOKEN` | 预获取的 OAuth Token（最高优先级） |
| `GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE` | OAuth 凭据文件路径 |
| `GOOGLE_WORKSPACE_CLI_CLIENT_ID` | OAuth 客户端 ID |
| `GOOGLE_WORKSPACE_CLI_CLIENT_SECRET` | OAuth 客户端密钥 |
| `GOOGLE_WORKSPACE_CLI_CONFIG_DIR` | 配置目录（默认 `~/.config/gws`） |
| `GOOGLE_WORKSPACE_PROJECT_ID` | GCP 项目 ID |

也支持 `.env` 文件加载。

## 高级用法

### 分页

```bash
# 自动分页（NDJSON 输出）
gws drive files list --params '{"pageSize": 100}' --page-all

# 限制页数
gws drive files list --params '{"pageSize": 100}' --page-limit 5

# 页间延迟（避免限流）
gws drive files list --params '{"pageSize": 100}' --page-delay 200
```

### 预览请求（Dry Run）

```bash
gws chat spaces messages create \
  --params '{"parent": "spaces/xyz"}' \
  --json '{"text": "Deploy complete."}' \
  --dry-run
```

### 查看 Schema

```bash
gws schema drive.files.list
```

### 工作流辅助命令

```bash
# 今日日程摘要
gws workflow +standup-report

# 会议准备
gws workflow +meeting-prep

# 邮件转任务
gws workflow +email-to-task

# 周报摘要
gws workflow +weekly-digest

# 文件公告（Drive 文件分享到 Chat）
gws workflow +file-announce
```

## 故障排除

| 问题 | 解决方案 |
|------|----------|
| `Access blocked` / 403 | OAuth 测试模式，需在同意屏添加测试用户 |
| `redirect_uri_mismatch` | OAuth 客户端类型不是"桌面应用" |
| `accessNotConfigured` | API 未启用，点击错误中的链接启用 |
| scope 超限 | 使用 `-s` 参数限制 scope 数量 |
| `gcloud not found` | `gws auth setup` 需要 gcloud CLI，或手动设置 OAuth |

---

**参考资料：**
- 官方仓库：https://github.com/googleworkspace/cli
- 技能索引：https://github.com/googleworkspace/cli/blob/main/docs/skills.md
- Google Cloud Console：https://console.cloud.google.com
