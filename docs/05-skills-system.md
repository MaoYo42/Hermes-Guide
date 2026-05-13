# Skill 系统

## 概述

Hermes Agent 的 Skill 系统是一个自进化的知识库。Skill 是 Markdown 格式的结构化文档，包含特定领域的知识、工作流和最佳实践。Agent 在学习到新知识或发现可复用的解决方案时，可以将它们持久化为 Skill，供未来会话使用。

Skill 不是指令或提示——它们是可加载的上下文知识，让 Agent 在需要时能即时调用。

---

## 安装状态

截至目前，Hermes 已安装 **107 个 Skill**，分为两类：

- **78 个内置 Skill** — 随 Hermes Agent 源码一起打包
- **29 个本地 Skill** — 从社区 Hub 安装或 Agent 自主创建

完整清单可通过 `hermes skills list` 查看。

---

## Skill 生命周期

Skill 经历四个层级，从通用到个性化：

```
内置 (builtin) → Hub (社区) → 本地安装 → Agent 自主创建
```

### 1. 内置 Skill（builtin）

随 Hermes Agent 源码一起发布的 Skill，位于 `hermes-agent/skills/` 目录。不可删除，始终可用。

典型的内置 Skill：
- `hermes-agent` — Hermes Agent 自身的使用指南和配置参考
- `native-mcp` — MCP 客户端配置和故障排除
- `kanban-orchestrator` / `kanban-worker` — 多 Agent 工作队列

### 2. Hub Skill（社区）

从官方 Skill Hub 或自建 Tap 源安装的 Skill。Hub 中的 Skill 经过审核，覆盖各种场景：

```
hermes skills browse           # 浏览 Hub 所有 Skill
hermes skills search QUERY     # 搜索 Hub
hermes skills install ID       # 安装（ID 为 Hub 标识符或 SKILL.md URL）
hermes skills inspect ID       # 预览但不安装
hermes skills check            # 检查更新
hermes skills update           # 更新过时 Skill
hermes skills uninstall NAME   # 卸载
hermes skills publish PATH     # 发布到注册表
```

Hub 支持 Tap 源——添加 GitHub 仓库作为额外 Skill 源：

```bash
hermes skills tap add REPO     # 添加 GitHub repo 作为 Skill 源
```

### 3. 本地安装 Skill

通过 Hub 安装后保存在 `~/.hermes/skills/` 目录。它们是活跃的知识模块，可在会话中按需加载。

### 4. Agent 自主创建 Skill（最强大的特性）

当 Agent 解决复杂问题、发现工作流或得到纠正时，可以将知识保存为 Skill：

```
使用 skill 工具创建新 Skill →
Skill 文件写入 ~/.hermes/skills/ → 
Curator 自动管理其生命周期
```

这是 Hermes 区别于其他 Agent 框架的核心特性——Agent 在实践中自我进化。

---

## Curator：Skill 自动看护系统

Curator 是一个后台维护系统，专门管理 Agent 自主创建的 Skill。

### 职责范围

- **只处理** `created_by: "agent"` 的 Skill（内置和 Hub Skill 不受影响）
- **跟踪使用情况** — 记录每个 Skill 的 use_count、view_count、patch_count
- **标记闲置 Skill** — 长时间未使用的标记为 stale
- **归档僵尸 Skill** — stale 超过阈值的自动归档
- **备份保障** — 归档前创建 tar.gz 备份，永不丢失
- **永不删除** — 最大破坏性操作是归档，不会删除任何 Skill

### CLI 命令

```bash
hermes curator status          # 查看 Curator 状态
hermes curator run             # 立即执行一次维护
hermes curator pause           # 暂停自动维护
hermes curator resume          # 恢复自动维护
hermes curator pin NAME        # 固定 Skill（豁免自动管理）
hermes curator unpin NAME      # 取消固定
hermes curator archive NAME    # 手动归档
hermes curator restore NAME    # 恢复归档的 Skill
hermes curator prune           # 清理归档
hermes curator backup          # 手动备份
hermes curator rollback        # 回滚
```

### 配置

```yaml
curator:
  enabled: true
  interval_hours: 24
  min_idle_hours: 1
  stale_after_days: 30
  archive_after_days: 60
  backup:
    enabled: true
    max_backups: 10
```

### 遥测数据

使用数据保存在 `~/.hermes/skills/.usage.json`，包含每个 Skill 的：

- `use_count` — 使用次数
- `view_count` — 查看次数
- `patch_count` — 更新/修正次数
- `last_used_at` — 最后使用时间
- `state` — 状态（active / stale / archived）
- `pinned` — 是否固定
- `archived_at` — 归档时间
- `created_by` — 创建者（agent / 用户 / hub）

---

## 在会话中使用 Skill

```bash
# 列出所有已安装 Skill
hermes skills list

# 查看 Skill 详情
hermes skills inspect NAME

# 按平台启用/禁用
hermes skills config

# 会话中加载 Skill（/skill 命令）
/skill hermes-agent

# 启动时预加载 Skill
hermes -s hermes-agent,obsidian
```

---

## Skill 结构示例

```markdown
---
name: example-skill
description: "技能描述"
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [tag1, tag2]
    homepage: https://example.com
    related_skills: [skill-a, skill-b]
---

# Skill 标题

## 核心内容

...
```

### 关键元数据字段

| 字段 | 说明 |
|------|------|
| `name` | 唯一标识符，也是加载时的名称 |
| `description` | 简短描述，显示在列表中 |
| `version` | 语义化版本号 |
| `platforms` | 支持的平台（linux / macos / windows） |
| `tags` | 分类标签 |
| `related_skills` | 相关 Skill 引用 |
| `metadata.hermes.created_by` | 标记创建者（agent / hub） |

---

## 与 Curator 相关的会话命令

在会话内使用 `/curator` 命令可直接操作：

```
/curator status       # 查看维护状态
/curator run          # 立即执行维护
/curator pin NAME     # 固定 Skill
/curator archive NAME # 归档
/curator restore NAME # 恢复

/reload-skills        # 重新扫描 skill 目录（新增/删除后使用）
/skill NAME           # 加载 Skill 到当前会话
```
