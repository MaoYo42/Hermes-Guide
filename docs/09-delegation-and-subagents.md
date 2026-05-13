# 任务委派与子 Agent

## 概述

Hermes Agent 的核心能力之一是 **任务委派（Delegation）**——当收到复杂或可并行化的请求时，Agent 可以创建子 Agent（Sub-agent）来分担工作。子 Agent 拥有与主 Agent 相同的模型能力，但专注于执行被指派的子任务。

MaoYo42 在日常工作流中频繁使用委派机制，尤其是批量数据处理和并行调研场景。

---

## 一、delegate_task 工具

委派功能通过内置工具 `delegate_task` 实现。Agent 在推理过程中会自动决定是否需要委派。

### 基本用法

```bash
# 一个典型的委派请求（Agent 内部工具调用）
delegate_task(task="分析这个GitHub仓库的架构设计")
```

Agent 会在以下场景自动触发委派：
- 任务包含多个可并行执行的子任务
- 任务需要同时调研多个方向的信息
- 当前任务比较耗时，子 Agent 可以独立处理一部分

### 三种调用方式

#### 1. 单任务委派

最简单的形式，将单一子任务交给子 Agent：

```
delegate_task(task="使用 Firecrawl 搜索 Rust 2025 最新版本特性")
```

主 Agent 等待子 Agent 完成后，将结果整合到最终回复中。

#### 2. 批量/并行委派

并行启动多个子 Agent 同时工作，大幅缩短总体响应时间：

```
delegate_task(tasks=[
    "搜索最新AI芯片发布",
    "搜索大模型开源许可证变化",
    "搜索端侧AI部署框架进展",
    "搜索AI安全监管新规"
])
```

**并行限制：** 默认最多 **3 个并发子 Agent**。这是合理平衡速度与 API 费用的上限。

MaoYo42 实战：同时搜索 3 个不同方向的科技新闻，Agent 汇总后输出"今日 AI 要闻简报"。

#### 3. 内联参数传值

可以为每个子任务指定不同参数或上下文：

```
delegate_task(tasks=[
    {"task": "分析A股半导体板块", "context": "近日美国芯片出口管制升级"},
    {"task": "分析A股新能源板块", "context": ""}
])
```

---

## 二、Leaf vs Orchestrator

Hermes 的委派系统区分两种 Agent 角色：

### Leaf（叶子 Agent）

**Leaf** 是只执行任务、不进一步委派别人的子 Agent。大多数委派任务都属于 Leaf 模式。

特性：
- 只能完成被指派的任务
- 不能再次调用 delegate_task
- 用完即销毁，不保留状态

### Orchestrator（编排 Agent）

**Orchestrator** 是主 Agent 的角色——它接收用户请求，拆解任务，委派给 Leaf，收集结果，整合回复。

MaoYo42 的工作流通常是这样：

```
用户请求 → Orchestrator(你) → 拆解为3项子任务
                           → Leaf Agent 1: 搜索数据
                           → Leaf Agent 2: 分析趋势
                           → Leaf Agent 3: 生成图表
                           → Orchestrator 整合结果 → 最终回复
```

---

## 三、委派限制

### max_spawn_depth

`max_spawn_depth` 控制委派的递归深度，默认值为 **1**。

```yaml
# ~/.hermes/config.yaml
delegation:
  max_spawn_depth: 1   # Leaf 不能再委派
```

含义：
- `max_spawn_depth=1` — 只有 Orchestrator 可以委派，子 Agent 不行
- `max_spawn_depth=2` — 子 Agent 也可以委派孙 Agent（一般不推荐）
- `max_spawn_depth=0` — 禁用委派

**MaoYo42 的建议：** 保持默认值 `1`。更深层的递归委派在多数场景下收益不大，反而增加 API 消耗和延迟。

### 其他限制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| 最大并发子 Agent | 3 | 同时运行的最多子 Agent 数 |
| 子 Agent 超时 | 120s | 超时后自动终止 |
| 最大结果长度 | 10000 tokens | 子 Agent 返回内容上限 |

---

## 四、何时使用委派

### 推荐委派的场景

1. **多源调研** — 同时搜索多个信息源，汇总对比
2. **批量处理** — 处理多个独立的文件、URL、数据条目
3. **并行分析** — 对同一数据的不同维度同时分析
4. **相互独立的子任务** — 子任务之间没有依赖关系

### 不适合委派的场景

1. **简单查询** — 一句话就能回答的问题，委派反而增加延迟
2. **需要连续上下文的任务** — 子 Agent 没有前文对话历史
3. **对实时性要求极高的任务** — 串行执行可能更快
4. **需要调用私有/本地敏感数据的任务** — 子 Agent 共享配置和工具

### 判断原则

> 如果一个任务拆开之后，各子任务之间不需要相互通信，且单个子任务的输出足够明确——那就适合委派。

---

## 五、实际用例

### 用例 1：多语言文档翻译

```
用户：帮我翻译这三篇英文文档
Agent 内部：
  delegate_task(tasks=[
    "翻译 doc1.md 为中文",
    "翻译 doc2.md 为中文",
    "翻译 doc3.md 为中文"
  ])
  # 3个子Agent并行翻译
  # 汇总3篇译文输出
```

### 用例 2：竞品分析

```
用户：分析Notion、Obsidian、Logseq的优劣势
Agent 内部：
  delegate_task(tasks=[
    "搜索 Notion 2025 最新功能和用户评价",
    "搜索 Obsidian 2025 最新功能和用户评价",
    "搜索 Logseq 2025 最新功能和用户评价"
  ])
  # 3个子Agent并行搜索
  # 汇总对比表格输出
```

### 用例 3：代码审查

```
用户：帮我 review 这三个 PR
Agent 内部：
  delegate_task(tasks=[
    "review PR #42 的变更",
    "review PR #43 的变更",
    "review PR #44 的变更"
  ])
  # 同时审查3个PR
```

---

## 六、注意事项

1. **API 费用翻倍** — 3 个子 Agent 并行 = 3 倍 token 消耗，注意预算
2. **结果整合需要手工** — Orchestrator 需要处理各子 Agent 的输出格式差异
3. **Leaf 不能持久化** — 子 Agent 会话在返回结果后即销毁，不会写入 MEMORY.md
4. **慎用深度委派** — `max_spawn_depth=2` 可能导致子 Agent 数量指数增长
5. **Telegram 网关兼容** — 在 Telegram 会话中同样支持委派，用户体验一致
