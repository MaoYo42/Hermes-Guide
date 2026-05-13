# Obsidian 知识库集成指南

## 概述

将 Obsidian 知识库通过 Git 同步与 Hermes Agent 集成，让 Agent 能读取、搜索和更新你的个人知识库。通过 git-synced vault，Agent 可以将新的学习成果、笔记和见解持久化存入你的 Obsidian 知识库。

## 架构

```
Obsidian (Desktop/App)
    ↕ 编辑 ✓
Git Repository (GitHub/GitLab)
    ↕ 同步
Hermes Agent (CLI)
    ↕ 读写
知识库文件 (.md)
```

## 第一步：创建 Obsidian 知识库（Vault）

### 新 Vault

1. 打开 Obsidian 桌面应用
2. 点击 **Create new vault**
3. 命名（如 `my-knowledge`）
4. 选择本地存储位置（如 `~/Documents/my-knowledge`）

### 现有 Vault

如果已有 Obsidian 知识库，直接使用现有目录。

## 第二步：初始化 Git 仓库

### 创建远程仓库

在 GitHub（或 GitLab）创建一个新的私有仓库：

```bash
# 使用 gh CLI
gh repo create my-knowledge-vault --private --description "我的 Obsidian 知识库"

# 或手动在 GitHub 网页端创建（推荐设为 Private）
```

### 本地初始化和推送

```bash
cd ~/Documents/my-knowledge

# 初始化 git
git init

# 创建 .gitignore（排除 Obsidian 配置和缓存）
cat > .gitignore << 'EOF'
# Obsidian
.obsidian/workspace.json
.obsidian/cache/
.obsidian/plugins/
.obsidian/themes/
.obsidian/community-plugins.json
.obsidian/graph.json
.obsidian/hotkeys.json
.obsidian/app.json
.obsidian/command-palette.json
.obsidian/bookmarks.json
.obsidian/switcher.json
.obsidian/templates.json
.obsidian/starred.json
.obsidian/global-search.json
.trash/

# OS files
.DS_Store
Thumbs.db

# Temp
*.tmp
EOF

# 首次提交
git add .
git commit -m "初始化 Obsidian 知识库"

# 关联远程仓库
git remote add origin git@github.com:你的用户名/my-knowledge-vault.git

# 推送
git push -u origin main
```

### .gitignore 说明

推荐的 `.gitignore` 保留 `.obsidian/` 中的核心配置但排除动态生成的文件：

- **排除** `workspace.json`（窗口布局，每台设备不同）
- **排除** `cache/`（本地缓存，不需同步）
- **排除** `plugins/` 和 `themes/`（可通过插件管理器重建）
- **保留** `core-plugins.json`（核心配置，建议同步）

## 第三步：安装 Obsidian Git 插件

Obsidian Git 社区插件实现自动提交和推送：

1. 打开 Obsidian → 设置 → 社区插件
2. 关闭"安全模式"（如果开启）
3. 点击 "浏览" → 搜索 **Obsidian Git**
4. 点击安装 → 启用

### 配置自动同步

在插件设置中配置：

```
# 自动备份（推荐）
Vault backup interval (minutes): 10

# 自动提交
Auto commit after file changes: ✓
Commit message: "auto: {{date}} 知识库更新"

# 推送策略
Auto pull interval (minutes): 10
Push on backup: ✓

# 拉取策略
Pull changes on startup: ✓
```

这样每次编辑后 Obsidian 会自动提交并推送，保持仓库最新。

## 第四步：Agent 读取知识库

### 配置环境变量

在 `~/.hermes/.env` 中添加：

```bash
# Obsidian 知识库路径
OBSIDIAN_VAULT_PATH=~/Documents/my-knowledge
```

### 创建知识库技能

创建 `skills/obsidian-vault/SKILL.md`：

```markdown
---
name: obsidian-vault
description: 读取和操作 Obsidian 知识库文件
version: 1.0.0
author: Hermes Agent
tags: [obsidian, knowledge, vault, notes]
---

# Obsidian Vault Skill

## 环境变量
- OBSIDIAN_VAULT_PATH: 知识库本地路径

## 操作

### 搜索笔记
使用 `grep -r` 或 `rg` 搜索知识库中的内容：
```bash
rg "关键词" "$OBSIDIAN_VAULT_PATH" --type md
```

### 读取笔记
```bash
cat "$OBSIDIAN_VAULT_PATH/笔记路径.md"
```

### 列出最近修改的笔记
```bash
find "$OBSIDIAN_VAULT_PATH" -name "*.md" -not -path "*/.obsidian/*" \
  -printf '%T@ %p\n' | sort -rn | head -10
```

### 创建新笔记
```bash
mkdir -p "$OBSIDIAN_VAULT_PATH/分类"
cat > "$OBSIDIAN_VAULT_PATH/分类/新笔记.md" << 'NOTE'
---
title: 新笔记
date: $(date +%Y-%m-%d)
tags: []
---

# 标题

内容...
NOTE
```

### 提交和推送
```bash
cd "$OBSIDIAN_VAULT_PATH"
git add -A
git commit -m "agent: 添加新笔记 - $(date +%Y-%m-%d)"
git push
```

## 规则
1. 所有笔记使用 Markdown 格式
2. 每篇笔记应包含 YAML frontmatter（标题、日期、标签）
3. 使用双链 [[Wikilinks]] 连接相关笔记
4. 提交信息以 "agent:" 开头以区分人类和 AI 编辑
```

## 第五步：Agent 写入知识库

### 基本写入流程

```bash
# 1. 确定笔记位置
VAULT=$OBSIDIAN_VAULT_PATH

# 2. 创建分类目录（如果不存在）
mkdir -p "$VAULT/编程/Python"

# 3. 写入笔记
cat > "$VAULT/编程/Python/Python异步编程.md" << 'EOF'
---
title: Python 异步编程
created: 2026-05-14
tags: [python, async, programming]
related: [编程/Python/协程]
---

# Python 异步编程

## 核心概念

- `async/await` 语法
- 事件循环 (`asyncio.run()`)
- 协程任务 (`asyncio.create_task()`)

## 示例

```python
import asyncio

async def main():
    await asyncio.sleep(1)
    print("Hello, async world!")

asyncio.run(main())
```
EOF

# 4. 提交并推送
cd "$VAULT"
git add -A
git commit -m "agent: 添加 Python 异步编程笔记"
git push
```

### 更新现有笔记

```bash
# 使用 sed 或 patch 更新 frontmatter 或内容
VAULT=$OBSIDIAN_VAULT_PATH
NOTE="$VAULT/编程/Python/Python异步编程.md"

# 更新 frontmatter 中的 tags
sed -i 's/tags: \[python, async\]/tags: [python, async, programming, coroutine]/' "$NOTE"

git -C "$VAULT" add -A
git -C "$VAULT" commit -m "agent: 更新 Python 异步编程标签"
git -C "$VAULT" push
```

## 第六步：知识库结构建议

### 推荐目录结构

```
my-knowledge/
├── 01-编程语言/
│   ├── Python/
│   ├── JavaScript/
│   └── Rust/
├── 02-技术架构/
│   ├── 微服务/
│   └── 系统设计/
├── 03-AI-ML/
│   ├── 提示词工程/
│   ├── Agent/
│   └── 模型/
├── 04-项目管理/
├── 05-个人成长/
├── 06-日记/
│   ├── 2026-01.md
│   └── 2026-05.md
├── 07-项目/
│   └── hermes-agent/
├── 资源/
│   └── 模板/
│       ├── 笔记模板.md
│       └── 日记模板.md
├── .gitignore
└── README.md
```

### 笔记模板

```markdown
---
title: "{{title}}"
created: {{date}}
updated: {{date}}
tags: []
status: seedling  # seedling | growing | evergreen
related: []
---

# {{title}}

## 概述

## 要点

## 链接
```

## 第七步：多设备同步

### 拉取更新

```bash
# 在另一台设备上
git clone git@github.com:你的用户名/my-knowledge-vault.git

# 在 Obsidian 中打开为 Vault（Open folder as vault）
```

### 解决冲突

```bash
# 如果发生同步冲突
cd ~/Documents/my-knowledge
git pull --rebase  # 变基拉取

# 解决冲突后
git add .
git rebase --continue
git push
```

建议开启 Obsidian Git 插件的自动拉取功能，减少冲突概率。

## 第八步：进阶 — 知识库搜索

### 使用 ripgrep (rg) 快速搜索

```bash
# 安装 ripgrep
sudo apt install ripgrep  # Linux
brew install ripgrep      # macOS

# 搜索内容
rg "关键词" "$OBSIDIAN_VAULT_PATH" --type md -l

# 搜索包含特定标签的笔记
rg "tags: .*python" "$OBSIDIAN_VAULT_PATH" --type md

# 按修改时间搜索最近笔记
rg "" "$OBSIDIAN_VAULT_PATH" --type md -l --sort modified
```

### Agent 自动学习写入

创建一个 Agent 自动化流程：

```bash
# 1. Agent 完成任务后总结收获
# 2. 检查知识库中是否已有相关内容
if rg -l "相似概念" "$OBSIDIAN_VAULT_PATH" --type md; then
  # 3. 如果存在，更新或追加
  echo "更新已有笔记..."
else
  # 4. 如果不存在，创建新笔记
  echo "创建新笔记..."
fi
# 5. 提交推送
```

## 安全注意事项

1. **私有仓库**：知识库包含个人信息，务必使用 `--private`
2. **Token 安全**：使用 SSH 密钥而非 HTTPS Token 进行 git 操作
3. **敏感内容**：不要在笔记中存储密码、密钥等敏感信息
4. **.gitignore**：确保排除 `.obsidian/workspace.json` 等本地配置文件
5. **提交审核**：Agent 的自动提交可能包含意外内容，建议定期审查

## 故障排除

| 问题 | 解决方案 |
|------|----------|
| 推送被拒绝 | 先 `git pull --rebase` 拉取最新内容 |
| 冲突标记出现在笔记中 | 手动解决冲突或使用 `git mergetool` |
| Obsidian Git 插件报错 | 检查 `.gitignore` 是否误排除了关键目录 |
| 文件太多推送慢 | 将大文件排除或使用 Git LFS |
| Agent 写入权限不足 | 确保 Agent 运行用户有目录写权限 |

---

**参考资料：**
- Obsidian 官网：https://obsidian.md
- Obsidian Git 插件：https://github.com/denolehov/obsidian-git
- 了解 Agent Skills 规范：https://agentskills.io
