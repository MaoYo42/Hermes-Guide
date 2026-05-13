# 进阶技能编写：SKILL.md 深度剖析

> YAML 前置元数据 · 条件激活 · 安全配置 · 发布到 Skills Hub

所有技能均通过 `~/.hermes/skills/` 下的 `SKILL.md` 文件定义。本文面向技能作者，覆盖官方文档中不会展开的编写技巧、发布流程和自动化管理。

---

## 目录

- [SKILL.md 完整字段参考](#skillmd-完整字段参考)
- [渐进式加载与 Tokens 优化](#渐进式加载与-tokens-优化)
- [条件激活 —— Fallback 技能实战](#条件激活--fallback-技能实战)
- [安全配置：环境变量与 Config 设置](#安全配置环境变量与-config-设置)
- [多文件技能结构](#多文件技能结构)
- [skil_manage — 让代理自己写技能](#skill_manage--让代理自己写技能)
- [发布技能到 Skills Hub](#发布技能到-skills-hub)
- [编写高质量技能的原则](#编写高质量技能的原则)
- [完整示例：从零创建一个技能](#完整示例从零创建一个技能)

---

## SKILL.md 完整字段参考

```markdown
---
name: my-awesome-skill
description: 一两句话说明这个技能做什么
version: 1.0.0
author: Your Name
platforms: [macos, linux]            # 可选，限制操作系统
required_environment_variables:
  - name: MY_API_KEY
    prompt: 你的 API 密钥
    help: 从 https://example.com/settings 获取
    required_for: 基本功能
metadata:
  hermes:
    tags: [python, automation, devops]
    category: devops
    fallback_for_toolsets: [web]       # 可选条件激活
    requires_toolsets: [terminal]      # 可选条件激活
    fallback_for_tools: [web_search]   # 可选：特定工具回退
    requires_tools: [terminal]         # 可选：需要特定工具
    config:
      - key: myskill.output_dir
        description: 输出目录路径
        default: "~/output"
        prompt: 输出文件存到哪里？
      - key: myskill.max_results
        description: 最大结果数
        default: "10"
        prompt: 最多返回多少结果？
---

# My Awesome Skill

## 什么时候用
当用户需要 [...] 时触发。适合 [...] 场景。

## 操作步骤
1. 第一步：...
2. 第二步：...
3. ...

## 常见陷阱
- 如果 X 失败，试试 Y
- 注意 Z 场景下的边界情况

## 验证方法
如何确认技能执行成功。
```

### 字段详解

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | **是** | 技能标识符，小写字母+连字符，同时也是斜杠命令名 |
| `description` | 推荐 | 出现在 `skills_list()` 和 `/skills` 命令中 |
| `version` | 推荐 | 语义化版本，Hub 更新时用于比较 |
| `platforms` | 否 | 限 `[macos]` / `[linux]` / `[windows]` 之一或组合，省略则全平台 |
| `required_environment_variables` | 否 | 技能需要的环境变量，安全提示用户在何时设置 |
| `metadata.hermes.tags` | 否 | 标签数组，用于 `skills search` 过滤 |
| `metadata.hermes.category` | 否 | 分类目录，影响在 `skills/` 下的存放路径 |
| `metadata.hermes.fallback_for_toolsets` | 否 | 当某工具集不可用时自动显示 |
| `metadata.hermes.requires_toolsets` | 否 | 仅当某工具集可用时才显示 |
| `metadata.hermes.config` | 否 | 非敏感配置项，存在 `config.yaml` 中 |

---

## 渐进式加载与 Tokens 优化

Hermes 不会在每次对话都把全部技能内容塞进上下文。它使用三级加载策略：

```
Level 0: skills_list()       → [{name, description, category}, ...]   (~3K tokens)
Level 1: skill_view(name)    → 完整的 SKILL.md 内容 + 元数据         (大小不定)
Level 2: skill_view(path)    → 引用目录下的特定文件                   (大小不定)
```

代理只在实际需要时才加载完整的技能内容。编写技能时应当注意：

- **description 足够清晰**：帮助代理在 Level 0 就判断是否该触发这个技能
- **步骤式结构**：每个步骤简短明确，代理能在上下文中保持
- **引用文件分离**：大块参考文档放到 `references/`，通过 Level 2 按需加载

### 示例：合理的渐进设计

```markdown
---
name: deploy-k8s
description: Deploy a containerized app to a Kubernetes cluster using kubectl and Helm
---

# Deploy to Kubernetes

## 快速参考
`kubectl apply -f deploy.yaml` → 部署到当前 context 的集群

## 完整步骤
详见 [references/deployment-checklist.md](references/deployment-checklist.md)

## 回滚
详见 [references/rollback.md](references/rollback.md)
```

---

## 条件激活 —— Fallback 技能实战

条件激活让技能根据当前会话的工具可用性自动显示或隐藏。最典型场景是 **Fallback 技能** —— 当付费工具不可用时，自动切换到免费替代方案。

### 内置示例：DuckDuckGo 搜索回退

```yaml
# DuckDuckGo Search 技能的 frontmatter
metadata:
  hermes:
    fallback_for_toolsets: [web]
```

**行为逻辑：**
- 当你有 `FIRECRAWL_API_KEY` 设置时 → web 工具集可用 → DuckDuckGo 技能隐藏
- 当 API Key 缺失时 → web 工具集不可用 → DuckDuckGo 技能自动出现

### 条件激活行为矩阵

| 字段 | 当条件为真 | 当条件为假 |
|------|-----------|-----------|
| `fallback_for_toolsets: [web]` | 技能隐藏 | 技能显示 |
| `requires_toolsets: [terminal]` | 技能显示 | 技能隐藏 |
| `fallback_for_tools: [web_search]` | 技能隐藏 | 技能显示 |
| `requires_tools: [terminal]` | 技能显示 | 技能隐藏 |

### 高级组合场景

```yaml
metadata:
  hermes:
    # 当没有 web 工具集但有 terminal 时显示
    fallback_for_toolsets: [web]
    requires_toolsets: [terminal]
```

这个技能只在"有终端但没网页搜索"的情况下出现 —— 例如一个用 `curl` + `jq` 替代 `web_search` 的 CLI 版搜索技能。

### 平台限制 + 条件激活

```yaml
platforms: [macos]
metadata:
  hermes:
    fallback_for_toolsets: [web]
```

macOS 专用回退技能 —— 只在 macOS 且无 web 工具时显示。

---

## 安全配置：环境变量与 Config 设置

技能可以声明需要的环境变量和配置项，而无需在 SKILL.md 正文中硬编码密钥。

### 环境变量声明

```yaml
required_environment_variables:
  - name: TENOR_API_KEY
    prompt: Tenor API Key
    help: 从 https://developers.google.com/tenor 获取
    required_for: GIF 搜索功能
```

**行为：**
- 在 CLI 加载技能时，如果变量缺失，Hermes 会安全地提示输入
- 在消息平台（Telegram/Discord）上不会在聊天中问密钥，而是提示用户在本地设置
- 声明的环境变量会自动传递到 `execute_code` 和 `terminal` 沙箱中 —— 技能脚本可以直接使用 `$TENOR_API_KEY`

### 非敏感配置项

```yaml
metadata:
  hermes:
    config:
      - key: gif_search.default_limit
        description: 每次搜索返回的 GIF 数量
        default: "10"
        prompt: 默认返回多少个结果？
      - key: gif_search.rating
        description: 内容分级过滤 (g/pg/pg-13/r)
        default: "pg-13"
        prompt: 内容分级？
```

这些配置值存储在 `~/.hermes/config.yaml` 的 `skills.config` 下。当技能加载时，解析后的配置值会自动注入上下文。`hermes config migrate` 会提示设置未配置项。

### 完整的安全技能示例

```markdown
---
name: gif-search
description: Search Tenor for GIFs
required_environment_variables:
  - name: TENOR_API_KEY
    prompt: Tenor API Key
    help: https://developers.google.com/tenor
    required_for: 全部功能
metadata:
  hermes:
    tags: [fun, gif, search]
    category: productivity
    fallback_for_toolsets: [web]
    config:
      - key: gif_search.default_limit
        description: 默认返回 GIF 数量
        default: "5"
        prompt: 每次搜几个？
      - key: gif_search.rating
        description: 内容分级
        default: "pg-13"
        prompt: 内容分级 (g/pg/pg-13/r)
---

# GIF Search

## 什么时候用
用户想搜索 GIF 图片时。

## 步骤
1. 使用 `TENOR_API_KEY` 调用 Tenor API
2. 搜索关键词来自用户输入
3. 返回结果中前 N 个（由 `gif_search.default_limit` 控制）
```

---

## 多文件技能结构

技能不只是单个 `SKILL.md`。完整的技能目录可以包含：

```
~/.hermes/skills/<category>/<skill-name>/
├── SKILL.md               # 主指令（必需）
├── references/            # 附加参考文档
│   ├── api-reference.md
│   └── troubleshooting.md
├── templates/             # 输出模板
│   ├── report-template.md
│   └── email-template.txt
├── scripts/               # 辅助脚本
│   ├── setup.sh
│   └── validate.py
└── assets/                # 补充文件
    └── logo.png
```

### 代理如何加载引用文件

在 SKILL.md 中直接引用即可：

```markdown
## 部署检查清单

参见 [references/deployment-checklist.md](references/deployment-checklist.md)

## 邮件模板

使用 [templates/release-email.md](templates/release-email.md) 生成通知邮件
```

代理会在需要时通过 `skill_view(name, path)` 按需加载这些文件（Level 2 加载），不会浪费上下文。

---

## skill_manage — 让代理自己写技能

Hermes 的 `skill_manage` 工具让代理可以在运行时创建、更新和删除技能。这是 Hermes 的**程序性记忆** —— 当代理完成一个复杂的任务后，会把它固化为一个可复用的技能。

### 代理何时自动创建技能

- 成功完成复杂任务（5+ 工具调用）后
- 在执行中遇到并解决了错误或死胡同时
- 用户纠正了代理的方法时
- 发现了一个非平凡的工作流时

### skill_manage 支持的操作

| 操作 | 用途 | 关键参数 |
|------|------|----------|
| `create` | 从零创建新技能 | `name`, `content`, 可选 `category` |
| `patch` | 针对性修复（推荐） | `name`, `old_string`, `new_string` |
| `edit` | 大规模结构改写 | `name`, `content`（替换整个 SKILL.md） |
| `delete` | 完全删除技能 | `name` |
| `write_file` | 添加/更新支持文件 | `name`, `file_path`, `file_content` |
| `remove_file` | 删除支持文件 | `name`, `file_path` |

### 实战：手动让代理创建技能

```text
你：把我刚才部署服务的整个流程保存为一个技能，叫 deploy-microservice。
```

Hermes 内部会调用：
```python
skill_manage(
    action="create",
    name="deploy-microservice",
    category="devops",
    content="""---
name: deploy-microservice
description: 部署微服务到 K8s 集群的完整流程
...

# Deploy Microservice

## 步骤
1. 构建 Docker 镜像...
"""
)
```

---

## 发布技能到 Skills Hub

### 方式一：发布为 GitHub Tap

创建一个 GitHub 仓库，按以下结构组织：

```
my-org/hermes-skills/
├── skills/                     # 默认路径
│   ├── deploy-runbook/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── security-audit/
│   │   └── SKILL.md
│   └── db-migration/
│       └── SKILL.md
└── README.md
```

每个目录名就是安装标识符。`SKILL.md` 必须有合法的 frontmatter（`name`, `description`）。

用户添加 tap 并安装：

```bash
# 添加源
hermes skills tap add my-org/hermes-skills

# 搜索并安装
hermes skills search deploy
hermes skills install my-org/hermes-skills/deploy-runbook
```

### 方式二：直接发布到 skills.sh

[skills.sh](https://skills.sh/) 是 Vercel 维护的公开技能目录。你可以把你的技能 PR 到 [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills) 仓库。

用户安装：

```bash
hermes skills search deploy --source skills-sh
hermes skills install skills-sh/your-org/your-skill
```

### 方式三：部署 Well-Known 端点

如果维护一个文档网站（如 Mintlify），可以在 `/.well-known/skills/index.json` 发布技能索引：

```json
{
  "skills": [
    {
      "id": "mintlify",
      "name": "Mintlify Docs Writer",
      "description": "帮助编写和优化 Mintlify 文档",
      "path": "/.well-known/skills/mintlify"
    }
  ]
}
```

用户通过 `well-known` 源发现和安装：

```bash
hermes skills search https://yourdocs.com --source well-known
hermes skills install well-known:https://yourdocs.com/.well-known/skills/mintlify
```

### 方式四：直接 URL

直接把 `SKILL.md` 放在任意 HTTP 服务器上：

```bash
hermes skills install https://example.com/my-skill/SKILL.md
```

### 安装来源对比

| 来源 | 安装标识符示例 | 适用场景 |
|------|---------------|----------|
| `official` | `official/security/1password` | Hermes 官方可选技能 |
| `skills-sh` | `skills-sh/vercel-labs/...` | 公开社区技能目录 |
| `well-known` | `well-known:https://...` | 网站托管自己的技能 |
| `github` | `openai/skills/k8s` | GitHub 仓库直接安装 |
| `url` | `https://.../SKILL.md` | 单文件直接发布 |
| `tap` | `my-org/my-skill` | 自定义私有源 |

### 安全扫描与 Trust 等级

所有 Hub 安装的技能都经过安全扫描，检查数据外泄、提示注入、破坏性命令等。

| 等级 | 来源 | 策略 |
|------|------|------|
| `builtin` | Hermes 内置 | 始终信任 |
| `official` | `optional-skills/` | 内置信任，无第三方警告 |
| `trusted` | 可信注册表（如 openai/skills） | 策略比 community 宽松 |
| `community` | 其余所有来源 | 非危险发现可用 `--force` 覆盖 |

---

## 编写高质量技能的原则

### 原则 1：描述即触发条件

`description` 是代理决定是否加载技能的关键。用**动作+场景**的格式：

```yaml
description: Deploy a containerized Python microservice to Kubernetes using Helm
# 而不是
description: K8s deployment utility
```

### 原则 2：步骤可执行

每个步骤应该是一个可验证的动作：

```markdown
## 步骤
1. 运行 `kubectl get pods` 检查集群状态
2. 更新 `deploy.yaml` 中的镜像标签为 `$VERSION`
3. 运行 `kubectl apply -f deploy.yaml`
4. 运行 `kubectl rollout status deployment/$NAME` 确认部署成功
```

### 原则 3：包含验证方法

```markdown
## 验证
- `curl https://$SERVICE_URL/health` 返回 200
- `kubectl logs deployment/$NAME --tail=50` 无错误日志
```

### 原则 4：记录陷阱

```markdown
## 已知陷阱
- 如果 rollout 卡住，检查 `kubectl describe pod` 中的 Events 段
- ImagePullBackOff 通常意味着镜像标签拼写错误或私有仓库未认证
- CrashLoopBackOff 请检查应用启动命令和环境变量
```

### 原则 5：平台限制

```yaml
platforms: [macos]          # 只在 macOS 上可用
platforms: [linux, macos]   # Linux + macOS
```

省略则所有平台可用。Windows 技能需要特别标注。

---

## 完整示例：从零创建一个技能

### 需求

创建一个"自动给 Git 仓库打标签"的技能。基于 Conventional Commits 自动推断版本号并创建 Git Tag。

### 步骤 1：创建技能目录

```bash
mkdir -p ~/.hermes/skills/devops/auto-tag
```

### 步骤 2：编写 SKILL.md

```markdown
---
name: auto-tag
description: 基于 Conventional Commits 自动推断版本号并创建 Git Tag
version: 1.0.0
author: Your Name
platforms: [macos, linux]
metadata:
  hermes:
    tags: [git, versioning, devops]
    category: devops
    requires_toolsets: [terminal]
---

# Auto Git Tag

## 什么时候用
- 项目准备发布新版本时
- 需要自动计算下一个版本号时
- 项目遵循 Conventional Commits 规范时

## 步骤
1. 切换到目标分支：`git checkout main && git pull`
2. 获取最近一个 tag：`git describe --tags --abbrev=0`
3. 分析从上一个 tag 以来的 commit 信息：
   - 有 `BREAKING CHANGE` 或 `!` 后缀 → major 版本 +1
   - 有 `feat:` → minor 版本 +1
   - 其余（fix, docs, refactor 等）→ patch 版本 +1
4. 创建新 tag：`git tag v$NEW_VERSION`
5. 推送 tag：`git push origin v$NEW_VERSION`

## 验证
- `git tag -l 'v*' | tail -5` 确认新 tag 出现在列表中
- `git log $NEW_TAG --oneline | head -5` 确认 commit 正确

## 陷阱
- 确保本地分支没有未推送的 commit
- 如果仓库没有历史 tag，从 `v0.1.0` 开始
- 仅在 main/master 分支上执行
```

### 步骤 3：使用技能

```bash
# 在聊天中
/auto-tag

# 或直接在命令行
hermes chat -q "帮我给当前仓库自动打 tag"
```

### 步骤 4：发布到 GitHub Tap

```bash
cd ~/.hermes/skills/devops/auto-tag
# 如果你有自己的 skills tap 仓库
# 把 auto-tag 目录复制到 tap 仓库的 skills/ 下
cp -r ~/.hermes/skills/devops/auto-tag /path/to/tap-repo/skills/
cd /path/to/tap-repo
git add skills/auto-tag
git commit -m "feat: add auto-tag skill"
git push
```

用户安装：

```bash
hermes skills install your-org/your-tap/auto-tag
```

---

## 外部技能目录

如果维护了多个 AI 工具共享的技能目录，可以用 `external_dirs`：

```yaml
# ~/.hermes/config.yaml
skills:
  external_dirs:
    - ~/.agents/skills
    - /home/shared/team-skills
    - ${SKILLS_REPO}/skills
```

**注意事项：**
- 外部目录只读 —— 代理创建/编辑技能时始终写回 `~/.hermes/skills/`
- 同名技能本地优先于外部
- 不存在的路径静默忽略
- 外部技能同样出现在 `skills_list`、`skill_view` 和斜杠命令中

---

## 参考

- [Hermes 官方技能系统文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)
- [Skills Hub 命令参考](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills#skills-hub)
- [创建技能 —— 配置设置](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills)
- [Bundled 技能目录](https://hermes-agent.nousresearch.com/docs/reference/skills-catalog)
- [可选技能目录](https://hermes-agent.nousresearch.com/docs/reference/optional-skills-catalog)

---

> ← 返回 [README](../README.md) · 编辑: `advanced/skill-authoring.md`
