# GitHub 工作流集成指南

## 概述

本指南涵盖 Agent 与 GitHub 集成的两种方式：
1. **gh CLI**（GitHub 官方命令行工具）— 完整的 API 访问
2. **Agent Skills**（`gh skill` 命令）— 可复用的 AI Agent 技能管理

Hermes Agent 内置了多个 GitHub 相关技能，配合 `gh` CLI 可以实现完整的 PR、Issue、代码审查和仓库管理工作流。

## 前置条件

- 安装 Git
- GitHub 账号
- （推荐）安装 gh CLI v2.90.0+

## 第一步：安装 gh CLI

### macOS

```bash
brew install gh
```

### Linux

```bash
# Debian/Ubuntu
sudo apt install gh

# 或从 GitHub Releases 下载
curl -LO https://github.com/cli/cli/releases/latest/download/gh_*_linux_amd64.tar.gz
tar xzf gh_*_linux_amd64.tar.gz
sudo mv gh_*/bin/gh /usr/local/bin/
```

### Windows

```bash
winget install --id GitHub.cli
```

### 验证安装

```bash
gh --version
# 要求 v2.90.0+ 以支持 gh skill 命令
```

## 第二步：认证

### 交互式浏览器登录（桌面环境）

```bash
gh auth login
# 选择: GitHub.com
# 选择: HTTPS
# 选择: 通过浏览器登录
```

### Token 登录（无头服务器/SSH）

```bash
# 在 https://github.com/settings/tokens 创建 Personal Access Token
# 需要的 scope: repo, workflow, read:org
echo "your_token_here" | gh auth login --with-token

# 让 gh 管理 git 凭据
gh auth setup-git
```

### 使用 Hermes Agent github-auth 技能

Hermes Agent 内置了 `github-auth` 技能，自动检测认证方式：

```bash
# 检测流程
gh auth status 2>/dev/null || echo "gh not authenticated"
```

技能会自动选择：
1. `gh` 已认证 → 直接使用
2. `gh` 未认证 → 引导 Token 或浏览器登录
3. `gh` 未安装 → 使用 git + curl 方式

## 第三步：GitHub 工作流技能

Hermes Agent 自带多个 GitHub 技能，位于 `skills/github/` 目录下：

| 技能 | 用途 |
|------|------|
| `github-auth` | 认证设置（HTTPS Token / SSH / gh CLI） |
| `github-pr-workflow` | PR 创建、更新、合并 |
| `github-code-review` | 代码审查与评论 |
| `github-issues` | Issue 管理 |
| `github-repo-management` | 仓库管理（分支、发布、设置） |

### 使用 PR 工作流

技能 `github-pr-workflow` 支持：

```bash
# 创建 PR
gh pr create --title "功能: 添加新模块" --body "描述..." --base main

# 审查 PR
gh pr review 42 --approve --body "LGTM!"

# 合并 PR
gh pr merge 42 --squash
```

### 使用代码审查技能

技能 `github-code-review` 支持：

```bash
# 获取 PR diff
gh pr diff 42

# 添加行内评论
gh pr review 42 --comment --body "这里需要修改"

# 列出审查意见
gh pr checks 42
```

### 使用 Issue 管理技能

技能 `github-issues` 支持：

```bash
# 创建 Issue
gh issue create --title "Bug: 登录失败" --body "重现步骤..."

# 列出 Issue
gh issue list --assignee @me

# 关闭 Issue
gh issue close 123 --comment "已在 PR #42 修复"
```

## 第四步：gh skill 命令（Agent Skills 管理）

GitHub CLI v2.90.0 新增了 `gh skill` 命令，用于发现、安装、管理和发布 Agent Skills。

### 发现技能

```bash
# 浏览仓库中的技能并交互安装
gh skill install github/awesome-copilot

# 搜索技能
gh skill search mcp-apps
```

### 安装技能

```bash
# 安装指定技能
gh skill install github/awesome-copilot documentation-writer

# 安装指定版本
gh skill install github/awesome-copilot documentation-writer@v1.2.0

# 安装到特定 Agent
gh skill install github/awesome-copilot documentation-writer --agent claude-code --scope user
```

### 锁定版本（供应链安全）

```bash
# 锁定到发布标签
gh skill install github/awesome-copilot documentation-writer --pin v1.2.0

# 锁定到特定 commit（最大可复现性）
gh skill install github/awesome-copilot documentation-writer --pin abc123def
```

### 更新技能

```bash
# 检查更新
gh skill update

# 更新特定技能
gh skill update documentation-writer

# 全部更新（不提示）
gh skill update --all
```

### 发布自己的技能

```bash
# 验证仓库中的所有技能
gh skill publish

# 自动修复元数据问题
gh skill publish --fix
```

### 支持的 Agent 主机

| 主机 | 安装示例 |
|------|----------|
| GitHub Copilot | `gh skill install OWNER/REPO SKILL` |
| Claude Code | `gh skill install OWNER/REPO SKILL --agent claude-code` |
| Cursor | `gh skill install OWNER/REPO SKILL --agent cursor` |
| Codex | `gh skill install OWNER/REPO SKILL --agent codex` |
| Gemini CLI | `gh skill install OWNER/REPO SKILL --agent gemini` |

## 第五步：Git-Only 方式（无 gh CLI）

如果环境没有安装 `gh` CLI，可以使用纯 git + curl 方式：

### HTTPS Token 方式

```bash
# 设置凭据持久存储
git config --global credential.helper store

# 触发一次认证（用户名填 GitHub 账号，密码填 Token）
git ls-remote https://github.com/your-username/any-repo.git

# 设置 git 用户信息
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

### SSH 方式

```bash
# 生成 SSH 密钥
ssh-keygen -t ed25519 -C "your-email@example.com"

# 复制公钥添加到 GitHub
cat ~/.ssh/id_ed25519.pub
# 前往 https://github.com/settings/keys → New SSH key

# 测试连接
ssh -T git@github.com

# 配置 git 自动将 HTTPS 转为 SSH
git config --global url."git@github.com:".insteadOf "https://github.com/"

# 设置 git 用户信息
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

### 无 gh 时的 API 调用

```bash
# 使用 GITHUB_TOKEN 环境变量
export GITHUB_TOKEN="your_token"

# REST API
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/owner/repo/pulls

# 或从 git 凭据中提取 Token
export GITHUB_TOKEN=$(grep "github.com" ~/.git-credentials | head -1 | sed 's|https://[^:]*:\([^@]*\)@.*|\1|')
```

## 常用 gh CLI 命令速查

### PR 操作

```bash
gh pr list                    # 列出 PR
gh pr create                  # 创建 PR
gh pr view 42                 # 查看 PR
gh pr checkout 42             # 检出 PR 分支
gh pr diff 42                 # 查看 diff
gh pr review 42 --approve     # 批准
gh pr review 42 --comment     # 评论
gh pr merge 42                # 合并
gh pr close 42                # 关闭
```

### Issue 操作

```bash
gh issue list                 # 列出 Issue
gh issue create               # 创建 Issue
gh issue view 123             # 查看 Issue
gh issue close 123            # 关闭
gh issue reopen 123           # 重新打开
gh issue comment 123 -b "说明" # 添加评论
```

### 仓库操作

```bash
gh repo view                  # 查看仓库信息
gh repo clone owner/repo      # 克隆仓库
gh repo create my-repo        # 创建仓库
gh repo fork owner/repo       # Fork 仓库
gh repo sync                  # 同步 fork
```

### Actions 操作

```bash
gh run list                   # 列出运行
gh run view                   # 查看运行详情
gh run watch                  # 实时查看输出
gh run rerun                  # 重新运行
gh workflow list              # 列出工作流
gh workflow run build.yml     # 触发工作流
```

### 发布操作

```bash
gh release list               # 列出发布
gh release create v1.0.0     # 创建发布
gh release view v1.0.0       # 查看发布
gh release download v1.0.0   # 下载发布资产
```

## Hermes Agent GitHub Skill 自动检测模式

Hermes Agent 的技能在初始化时自动执行以下检测：

```bash
if command -v gh &>/dev/null && gh auth status &>/dev/null; then
  echo "AUTH_METHOD=gh"       # 使用 gh CLI
elif [ -n "$GITHUB_TOKEN" ]; then
  echo "AUTH_METHOD=curl"     # 使用 curl + Token
elif [ -f ~/.hermes/.env ]; then
  source ~/.hermes/.env       # 从 .env 加载
  echo "AUTH_METHOD=env"
else
  echo "AUTH_METHOD=none"     # 需要先设置认证
fi
```

## 故障排除

| 问题 | 解决方案 |
|------|----------|
| `git push` 要求输入密码 | GitHub 已禁用密码认证，使用 Token 代替 |
| `Permission to X denied` | Token 缺少 `repo` scope，重新生成 |
| `Authentication failed` | 凭据过期，运行 `git credential reject` 重新认证 |
| `gh: command not found` | 使用 git-only 方式（见第五步） |
| SSH 连接被拒绝 | 尝试 SSH over HTTPS：在 `~/.ssh/config` 添加 `Host github.com` + `Port 443` |
| 多账号管理 | 使用 SSH 不同密钥或 per-repo 凭据 URL |

---

**参考资料：**
- gh CLI 文档：https://cli.github.com/manual
- gh skill 文档：https://cli.github.com/manual/gh_skill
- Agent Skills 规范：https://agentskills.io
- Hermes GitHub 技能文档：https://hermes-agent.nousresearch.com/docs/user-guide/skills/bundled/github
