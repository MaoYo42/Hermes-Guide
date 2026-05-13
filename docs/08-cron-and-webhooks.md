# 定时任务与 Webhook

## 概述

Hermes Agent 内置了一套实用的定时任务系统（Cron Job System），同时支持 Webhook 订阅机制。两者结合，可以让 Agent 在无人值守的情况下周期性地执行任务、监控变化、推送通知。

MaoYo42 的 Cron 使用策略聚焦于关键业务场景：

---

## 一、Cron Job 系统

### 触发方式：hermes cron CLI

所有定时任务通过 `hermes cron` 命令行管理：

```bash
# 列出所有任务
hermes cron list

# 添加一个任务
hermes cron add --name "daily-stock" --schedule "30 9 * * 1-5" --script "分析今日A股热点板块"

# 暂停/恢复
hermes cron pause daily-stock
hermes cron resume daily-stock

# 删除
hermes cron remove daily-stock
```

### 计划格式（Schedule）

| 格式 | 含义 | 示例 |
|------|------|------|
| `30m` | 每 30 分钟 | 短周期监控 |
| `1h` | 每小时 | 常规巡检 |
| `2h` | 每 2 小时 | 低频检查 |
| `0 9 * * *` | 标准 cron：每日 9:00 | 日报类任务 |
| `30 9 * * 1-5` | 工作日 9:30 | 交易日分析 |
| `0 0 1 * *` | 每月 1 日 0:00 | 月度汇总 |

MaoYo42 最常用的格式是 `30m`（高频变化检测）和 `0 9 * * *`（每日推送）。

### 执行模式

Hermes Cron Job 支持两种执行模式：

#### 1. Script 模式（推荐）

直接指定一条 Agent 指令，每次触发时 Agent 将该指令当作新会话的初始消息执行：

```bash
hermes cron add \
  --name "daily-stock" \
  --schedule "30 9 * * 1-5" \
  --script "查看今天A股涨幅前5的板块，分析资金流向，输出摘要"
```

特点：
- 每次触发是独立的 "新对话"，Agent 从零开始执行
- 不保留历史上下文，适合无状态任务
- 执行结果可通过 notify 选项推送到消息平台

#### 2. Agent 模式（不推荐）

```bash
hermes cron add --name "monitor" --schedule "30m" --agent
```

Agent 模式下，每次触发会让 Agent 在当前活跃会话中插入一条消息。容易打断正在进行的对话，不建议在个人工作流中使用，除非你明确需要 Agent 在当前上下文中继续。

**MaoYo42 的建议：始终使用 Script 模式。** 独立干净，互不干扰。

### 执行结果通知

通过 `--notify` 将任务执行结果推送到消息平台：

```bash
hermes cron add \
  --name "daily-risk" \
  --schedule "0 8 * * *" \
  --script "检查系统安全更新和异常登录" \
  --notify telegram
```

支持的 notify 目标：`telegram`、`discord`、`slack` 等。

---

## 二、Job 链式调用（context_from）

Cron Job 支持链式执行：一个 Job 可以引用前一个 Job 的上下文，形成处理流水线。

```bash
hermes cron add \
  --name "data-fetch" \
  --schedule "0 9 * * *" \
  --script "获取今日最新经济数据"

hermes cron add \
  --name "data-analyze" \
  --schedule "15 9 * * *" \
  --script "分析数据并输出趋势" \
  --context_from data-fetch
```

`context_from` 会让第二个 Job 在触发时自动加载前一个 Job 的最近一次执行结果作为上下文。这在多步骤数据处理中非常有用。

**注意事项：**
- 两个 Job 的间隔时间要足够（如上面间隔 15 分钟）
- 如果前一个 Job 失败，后一个 Job 仍会执行，但上下文可能为空
- 链不能形成循环依赖

---

## 三、Webhook 订阅

Webhook 系统让外部系统可以在特定事件发生时通知 Hermes Agent。

### 添加 Webhook

```bash
# 添加一个 webhook 端点
hermes cron webhook add --path /alert --script "分析收到的告警并给出处理建议"
```

这会生成一个独特的 URL 端点（如 `https://your-host/webhook/alert/xxxxx`），外部系统可以向这个 URL 发送 POST 请求。

### 使用场景

MaoYo42 的实际用法：

1. **系统监控告警** — Prometheus Alertmanager 推送告警到 Hermes webhook，Agent 自动分析严重性并给出处理建议
2. **Git 事件订阅** — GitHub/GitLab Webhook 推送 PR/MR 事件，Agent 自动 Review 变更
3. **价格变化跟踪** — 爬虫检测到商品价格变化后推送 webhook，Agent 判断是否值得购入

### Webhook 安全

每个 webhook 端点自动生成一个不可预测的 token 作为路径的一部分，天然防猜解。如果需要额外认证，可在 `~/.hermes/config.yaml` 中配置：

```yaml
cron:
  webhook_secret: "your-shared-secret"
```

客户端在 HTTP Header 中带上 `X-Webhook-Secret: your-shared-secret` 即可。

---

## 四、实际用例

### 用例 1：每日 A 股分析

```bash
hermes cron add \
  --name "a-share-daily" \
  --schedule "30 9 * * 1-5" \
  --script "查看今天A股涨幅前5的板块，分析资金净流入前3个股，100字摘要输出" \
  --notify telegram
```

### 用例 2：变化检测（Change Detection）

结合 Firecrawl 的 changeTracking 功能，监控网页变化：

```bash
hermes cron add \
  --name "watch-pricing" \
  --schedule "30m" \
  --script "使用 Firecrawl 检查 https://example.com/pricing 是否有变化，如有变化则抓取新内容并对比差异" \
  --notify telegram
```

### 用例 3：系统健康巡检

```bash
hermes cron add \
  --name "sys-health" \
  --schedule "0 */2 * * *" \
  --script "检查系统磁盘使用率、内存、CPU负载，如有异常输出告警" \
  --notify telegram
```

---

## 五、管理与调试

| 命令 | 用途 |
|------|------|
| `hermes cron list` | 查看所有任务及状态 |
| `hermes cron logs NAME` | 查看指定任务的执行历史 |
| `hermes cron pause NAME` | 暂停任务 |
| `hermes cron resume NAME` | 恢复任务 |
| `hermes cron remove NAME` | 删除任务 |
| `hermes cron trigger NAME` | 手动触发一次执行（测试用） |
| `hermes cron webhook list` | 列出所有 webhook 端点 |

### 日志查看

```bash
# 查看最近 10 次执行记录
hermes cron logs daily-stock

# 查看带详细输出的日志
hermes cron logs daily-stock --verbose
```

---

## 六、注意事项

1. **时间周期不要太密** — `30m` 是最短推荐周期，更密的计划可能导致速率限制或 API 费用激增
2. **Script 模式稳定可靠** — 每个 Job 独立运行，一个 Job 失败不影响其他 Job
3. **Webhook 端点不要暴露** — token 参数不可预测，建议额外加一层反向代理 IP 白名单
4. **Firecrawl 变化检测** — 会消耗 Firecrawl 额度，监控高频变化页面时注意控制频率
5. **context_from 注意间隔** — 给链式 Job 留足时间差
