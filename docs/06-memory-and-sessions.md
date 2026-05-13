# 记忆系统与会话管理

Hermes Agent 拥有持久化记忆系统，让 Agent 在不同会话之间记住用户信息、环境特征和学到的知识。

---

## 持久记忆（Persistent Memory）

### 双存储结构

记忆系统由两个文件组成，存放在 `~/.hermes/memories/` 目录：

| 文件 | 字符限制 | 用途 |
|------|----------|------|
| `MEMORY.md` | 2200 字符 | Agent 的笔记：环境事实、项目约定、工具特性、学到的东西 |
| `USER.md` | 1375 字符 | 关于用户的信息：偏好、沟通风格、期望、工作习惯 |

> 限制是**字符数**而非 token 数，因为字符计数是模型无关的。

### 记忆注入机制

记忆以**冻结快照**形式注入到 system prompt 中：

```
会话启动时 → load_from_disk() 读取记忆文件
           → 生成快照（_system_prompt_snapshot）
           → 注入到 system prompt
           → 整个会话期间快照不变

会话中写记忆 → 立即写入磁盘（持久化）
             → System prompt 不变（保持 prefix cache）
             → 下一会话加载新快照
```

这种"冻结快照 + 即时持久化"模式兼顾了 Prompt Caching 效率和数据持久性。

### 记忆工具

Agent 使用单个 `memory` 工具管理记忆，支持四个操作：

- **add** — 添加新条目（自动去重）
- **replace** — 替换现有条目（短唯一子串匹配）
- **remove** — 删除条目
- **read** — 读取当前记忆状态

条目分隔符为 `§`（section sign），条目可跨多行。

### 记忆编写准则

从工具描述中提取的最佳实践：

1. **声明式事实，而非指令** — "用户偏好简洁回复" ✓ vs "你必须简洁回复" ✗
2. **持久性判断** — 只保存 7 天后仍有效的事实
3. **减少纠正** — 最有价值的记忆是能避免用户再次纠正你的那一条
4. **不保存任务进度** — PR 号、commit SHA、阶段性完成状态会过期
5. **复杂流程用 Skill** — 新学的工作流应保存为 Skill，不是记忆

### 安全扫描

记忆注入到 system prompt，因此有严格的安全检查：

- **Unicode 不可见字符**检测（零宽空格、双向文本控制符等）
- **Prompt 注入模式**检测（"ignore previous instructions"、"you are now" 等）
- **凭据外泄模式**检测（curl/wget 带 API key、cat .env 等）
- **后门代码**检测（authorized_keys、SSH 访问等）

命中任一种模式都会拒写并返回错误信息。

---

## 会话管理（Session Management）

每次与 Hermes Agent 的交互都是一个会话（session），包含完整的对话历史和工具调用记录。

### 会话存储

会话以 JSON 文件形式保存在 `~/.hermes/sessions/`，使用 SQLite 数据库（`hermes_state.db`）索引，支持 FTS5 全文搜索。

### CLI 命令

```bash
hermes sessions list              # 列出最近的会话
hermes sessions browse            # 交互式会话选择器
hermes sessions export OUTPUT     # 导出为 JSONL
hermes sessions rename ID TITLE   # 重命名会话
hermes sessions delete ID         # 删除会话
hermes sessions prune             # 清理旧会话（--older-than N days）
hermes sessions stats             # 会话存储统计
```

### 会话恢复

```bash
# 恢复最近的会话
hermes --continue
hermes -c

# 按 ID 或标题恢复特定会话
hermes --resume SESSION_ID
hermes -r SESSION_ID

# 恢复带名称的会话
hermes -c my-session-name
```

### 会话中的记忆注入

会话 system prompt 中，记忆以以下结构注入：

```
<Memory>
§ 条目 1
§ 条目 2
§ ...

<You and the User>
§ 用户偏好 1
§ 用户偏好 2
§ ...
```

Memory 区块来自 MEMORY.md，User 区块来自 USER.md。

### 会话生命周期

```
hermes (启动) → 加载记忆 → 构建 system prompt
  → 用户输入 → Agent 循环（工具调用 + 回复）
  → 会话结束（保存 transcript）
  → 外部记忆提供者可选地提取摘要
```

### 会话搜索

```bash
# 在会话中搜索历史内容
使用 /history 命令

# 通过 session_search 工具搜索
Agent 可使用 session_search 工具搜索过去的对话
```

### 记忆提供者插件

除内置记忆外，Hermes 支持外部记忆提供者插件：

| 提供者 | 特点 |
|--------|------|
| Honcho | AI 原生跨会话用户建模，语义搜索 |
| OpenViking | 轻量级替代方案 |
| Mem0 | 零配置语义记忆 |
| Supermemory | 超大规模记忆管理 |

```bash
hermes memory setup      # 交互式配置外部提供者
hermes memory status     # 查看当前提供者
hermes memory off        # 关闭外部提供者
```

---

## 常见问题

**Q: 为什么我的记忆没有在当前会话中生效？**
A: 记忆快照在会话启动时加载，写操作即时持久化但不会影响当前会话的 system prompt。新记忆在下一会话生效。

**Q: 记忆字符超限了怎么办？**
A: `memory` 工具会自动拒绝超出限制的写入。需要清理旧条目以释放空间。

**Q: 会话文件会占用多少空间？**
A: 每个会话约数个 KB 到数百 KB，取决于对话长度。使用 `hermes sessions prune` 定期清理。

**Q: 如何搜索很久以前的对话？**
A: 使用 `hermes sessions browse` 交互式浏览，或通过 session_search 工具让 Agent 帮你搜索。
