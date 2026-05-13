# 02 — 配置文件详解 (Configuration)

本文以 **MaoYo42** 的真实配置为例，逐段解读 `~/.hermes/config.yaml` 的全部核心配置项。

> 配置文件位置：`~/.hermes/config.yaml`
> 密钥存放：`~/.hermes/.env`
> OAuth 凭据：`~/.hermes/auth.json`

---

## 1. 模型与 Provider 配置

```yaml
model:
  provider: deepseek
  default: deepseek-v4-flash
  base_url: https://api.deepseek.com/v1

providers: {}
fallback_providers: []
credential_pool_strategies: {}
```

**model** 是配置的核心入口：

| 字段 | 说明 | 实例如 |
|------|------|--------|
| `provider` | 推理后端类型 | `deepseek` 使用 DeepSeek 官方 API |
| `default` | 默认模型 ID | `deepseek-v4-flash` |
| `base_url` | OpenAI 兼容端点 | DeepSeek 官方端点 |

**MaoYo42 的选择理由：** DeepSeek-v4-Flash 是 DeepSeek 最新一代快模型，推理速度快、编码能力强，适合作为日常主力模型。base_url 使用国内直连的官方端点，延迟极低。

**替代方案：**
- 使用 `openrouter` 作为 provider，可切换 200+ 模型
- 使用 `anthropic` 直接连 Anthropic API，绕过路由层
- 使用 `ollama-cloud` 托管开源模型
- 使用 `custom` 加 base_url 连本地 vLLM/Ollama

`providers: {}` 表示不定义额外 provider 级别的覆盖（如超时等）。如果需要：
```yaml
providers:
  deepseek:
    request_timeout_seconds: 600
    models:
      deepseek-v4-flash:
        timeout_seconds: 300
```

---

## 2. Toolsets 与代理行为

```yaml
toolsets:
  - hermes-cli

agent:
  max_turns: 90
  gateway_timeout: 1800
  restart_drain_timeout: 180
  api_max_retries: 3
  service_tier: ''
  tool_use_enforcement: auto
  gateway_timeout_warning: 900
  gateway_notify_interval: 180
  gateway_auto_continue_freshness: 3600
  image_input_mode: auto
  disabled_toolsets: []
```

**toolsets** 控制代理可用的工具集合。`hermes-cli` 是 Hermes Agent 的标准工具集（终端、文件读写、搜索、MCP 等）。

**agent** 段关键参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `max_turns` | 90 | 单次会话最大工具调用轮数 |
| `gateway_timeout` | 1800s | 网关会话超时（30分钟） |
| `api_max_retries` | 3 | API 调用失败重试次数 |
| `image_input_mode` | auto | 图片输入模式（auto/disable） |

> **实战建议：** `max_turns` 不宜过低（<20），复杂任务（如编码重构）需要多轮工具调用。`gateway_timeout` 在 Telegram 等异步场景可适当延长。

---

## 3. 终端后端配置

```yaml
terminal:
  backend: local
  modal_mode: auto
  cwd: .
  timeout: 180
  env_passthrough: []
  shell_init_files: []
  auto_source_bashrc: true
  docker_image: nikolaik/python-nodejs:python3.11-nodejs20
  docker_forward_env: []
  docker_env: {}
  singularity_image: docker://nikolaik/python-nodejs:python3.11-nodejs20
  modal_image: nikolaik/python-nodejs:python3.11-nodejs20
  daytona_image: nikolaik/python-nodejs:python3.11-nodejs20
  vercel_runtime: node24
  container_cpu: 1
  container_memory: 5120
  container_disk: 51200
  container_persistent: true
  docker_volumes: []
  docker_mount_cwd_to_workspace: false
  docker_extra_args: []
  docker_run_as_host_user: false
  persistent_shell: true
  lifetime_seconds: 300
```

**MaoYo42 使用 `local` 后端**，命令直接在宿主机执行。这是最简单的模式，适合个人使用。

**终端后端对比：**

| 后端 | 隔离性 | 适用场景 |
|------|--------|----------|
| `local` | 无 | 开发、个人使用 |
| `docker` | 完整容器隔离 | 安全沙箱、CI/CD |
| `ssh` | 网络隔离 | 远程服务器 |
| `modal` | 云端 VM | 弹性云计算 |
| `singularity` | 容器隔离 | HPC 集群 |

**container_persistent: true** 在容器后端下保持文件系统状态跨会话持久化。

> **实战建议：** 生产环境推荐 `docker` 后端。如需本地安全，注意 `terminal.backend: local` 下代理拥有你用户的所有文件系统权限。

---

## 4. 显示与个性

```yaml
display:
  compact: false
  personality: kawaii
  resume_display: full
  busy_input_mode: interrupt
  tui_auto_resume_recent: false
  bell_on_complete: false
  show_reasoning: false
  streaming: false
  timestamps: false
  final_response_markdown: strip
  persistent_output: true
  persistent_output_max_lines: 200
  inline_diffs: true
  show_cost: false
  skin: default
  language: en
  tui_status_indicator: kaomoji
  user_message_preview:
    first_lines: 2
    last_lines: 2
  interim_assistant_messages: true
  tool_progress_command: false
  tool_progress_overrides: {}
  tool_preview_length: 0
  ephemeral_system_ttl: 0
  platforms: {}
  runtime_footer:
    enabled: false
    fields:
      - model
      - context_pct
      - cwd
  copy_shortcut: auto
  tool_progress: all
```

**MaoYo42 的配置亮点：**

| 设置 | 值 | 效果 |
|------|----|------|
| `personality: kawaii` | kawaii | 萌系表情符号（(｡>ω<｡)） |
| `tui_status_indicator: kaomoji` | kaomoji | 颜文字状态指示 |
| `inline_diffs: true` | 开启 | 工具输出差异内联显示 |
| `final_response_markdown: strip` | strip | 移除最终响应的 Markdown 格式 |
| `language: en` | 英文 | 界面语言（不影响代理回复语言） |

**kawaii 个性**让 Hermes 使用颜文字和萌系语素，适合追求趣味交互风格的用户。可用值：`default`、`kawaii`、`professional`、`minimal`。

> **实战建议：** `show_cost: true` 可显示每次调用的 token 消耗；`show_reasoning: true` 可查看模型的推理过程（DeepSeek-R1 等支持）。

---

## 5. 隐私与安全

```yaml
privacy:
  redact_pii: false

security:
  allow_private_urls: false
  redact_secrets: true
  tirith_enabled: true
  tirith_path: tirith
  tirith_timeout: 5
  tirith_fail_open: true
  website_blocklist:
    enabled: false
    domains: []
    shared_files: []
  acked_advisories: []
  allow_lazy_installs: true
```

**核心安全特性：**

| 设置 | 值 | 说明 |
|------|----|------|
| `redact_secrets: true` | 启用 | **自动从日志和输出中脱敏 API Key、Token 等敏感信息** |
| `tirith_enabled: true` | 启用 | 命令安全策略引擎，阻止潜在危险命令 |
| `tirith_fail_open: true` | 开放 | Tirit 引擎不可用时允许命令通过 |

**redact_secrets: true 是强烈推荐的设置**。Hermes 会自动识别并脱敏：
- API Key（`sk-*`, `fc-*` 等模式）
- OAuth Token
- 环境变量中的密钥
- 日志中的敏感信息

**网站黑名单**（`website_blocklist.enabled: false` 默认关闭）用于防止代理访问特定域名。

> **实战建议：** 开启 `redact_secrets` 是必要的安全基线。`privacy.redact_pii` 是更严格的 PII 脱敏，默认为关闭。

---

## 6. 记忆与收藏家 (Memory & Curator)

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  provider: ''

curator:
  enabled: true
  interval_hours: 168
  min_idle_hours: 2
  stale_after_days: 30
  archive_after_days: 90
  backup:
    enabled: true
    keep: 5
```

**记忆系统** 让代理跨会话记住用户偏好和关键信息：

| 参数 | MaoYo42 设置 | 说明 |
|------|-------------|------|
| `memory_enabled` | true | 启用代理记忆 |
| `user_profile_enabled` | true | 启用用户画像 |
| `memory_char_limit` | 2200 | 代理记忆最大字符数 |
| `user_char_limit` | 1375 | 用户画像最大字符数 |

**Curator（收藏家）** 是会话管理的自动调度器：

| 参数 | 值 | 说明 |
|------|----|------|
| `enabled: true` | 启用 | 自动清理归档旧会话 |
| `interval_hours: 168` | 每周 | 每周运行一次清理 |
| `stale_after_days: 30` | 30天 | 30天未活动的会话标记为过时 |
| `archive_after_days: 90` | 90天 | 90天后归档 |
| `backup.enabled: true` | 备份 | 归档前备份 |

> **实战建议：** `memory_char_limit` 取决于你的模型上下文窗口。大型模型（如 DeepSeek-v4）可设置更高（4000+）。Curator 的 `min_idle_hours: 2` 确保代理空闲 2 小时以上才进行归档，避免打断工作中的会话。

---

## 7. 语音与 TTS/STT

```yaml
tts:
  provider: edge
  edge:
    voice: en-US-AriaNeural
  elevenlabs:
    voice_id: pNInz6obpgDQGcFmaJgB
    model_id: eleven_multilingual_v2
  openai:
    model: gpt-4o-mini-tts
    voice: alloy
  xai:
    voice_id: eve
    language: en
    sample_rate: 24000
    bit_rate: 128000
  mistral:
    model: voxtral-mini-tts-2603
    voice_id: c69964a6-ab8b-4f8a-9465-ec0925096ec8
  neutts:
    ref_audio: ''
    ref_text: ''
    model: neuphonic/neutts-air-q4-gguf
    device: cpu
  piper:
    voice: en_US-lessac-medium

stt:
  enabled: true
  provider: local
  local:
    model: base
    language: ''
  openai:
    model: whisper-1
  mistral:
    model: voxtral-mini-latest

voice:
  record_key: ctrl+b
  max_recording_seconds: 120
  auto_tts: false
  beep_enabled: true
  silence_threshold: 200
  silence_duration: 3
```

**MaoYo42 配置解读：**

- **TTS：`provider: edge`** — 使用微软 Edge 的在线 TTS，`en-US-AriaNeural` 是自然女声
- **STT：`provider: local` + `model: base`** — 使用本地 Whisper 模型（base 参数=~1.5GB，速度与精度的平衡点）
- 注入了多个备选 TTS 提供商（ElevenLabs、OpenAI、xAI、Mistral、Neuphonic、Piper），可通过 `/voice` 命令切换

**voice 配置：**
- `record_key: ctrl+b` — 录音快捷键
- `silence_threshold: 200` — 静音检测阈值（200ms）
- `beep_enabled: true` — 开始/停止录音时有提示音

> **实战建议：** TTS 选择 edge 适合需要免费高质量语音的场景。ElevenLabs 质量更高但需要 API Key。STT 的 `local` 模式完全离线，保护隐私。

---

## 8. 网关 (Gateway) 与平台

```yaml
telegram:
  reactions: false
  channel_prompts: {}
  allowed_chats: '1419480065'
  allow_from: '1419480065'

platforms:
  api_server:
    extra:
      port: 8642
      host: 127.0.0.1
    enabled: true
    key: ''
    cors_origins: '*'
```

**Telegram 网关** — MaoYo42 使用 Telegram Bot：
- `allowed_chats: '1419480065'` — 仅允许特定用户 ID 访问
- `allow_from: '1419480065'` — 来源限制，防止他人使用
- `reactions: false` — 不启用 Telegram 消息反应

**API Server** — 在 `127.0.0.1:8642` 上启用 HTTP API：
- 本地 REST API 端点
- CORS 允许任意来源（开发环境）
- 可用于集成第三方工具或 Web UI

> **实战建议：** 生产环境应设置 `api_server.key` 添加 API Key 认证，并将 `cors_origins` 限制为具体域名。

其他平台配置（slack、discord、whatsapp、mattermost、matrix 等）均为默认空值，可根据需要启用。

---

## 9. 时区与调度

```yaml
timezone: Asia/Shanghai

cron:
  wrap_response: true
  max_parallel_jobs: null

session_reset:
  mode: both
  idle_minutes: 1440
  at_hour: 4

checkpoints:
  enabled: false
  max_snapshots: 20
  max_total_size_mb: 500
  max_file_size_mb: 10
  auto_prune: true
  retention_days: 7
  delete_orphans: true
  min_interval_hours: 24
```

**MaoYo42 在上海时区操作**，配置亮点：

- `timezone: Asia/Shanghai` — 所有时间戳、cron 任务使用中国时区
- `session_reset.mode: both` — 综合策略（空闲 24 小时 + 每日 4AM 自动重置）
- `checkpoints` 默认关闭 — 可改为 `enabled: true` 启用会话检查点回滚

> **实战建议：** Cron 任务的时区跟随 `timezone` 设置。`at_hour: 4` 选在凌晨低使用时段重置会话，减少影响。

---

## 10. MCP 服务器配置

```yaml
mcp_servers:
  firecrawl:
    command: npx
    args: ["-y", "firecrawl-mcp"]
    env:
      FIRECRAWL_API_KEY: "fc-2fd...064c"
```

**MaoYo42 集成了 Firecrawl MCP 服务器**：
- 使用 `npx -y firecrawl-mcp` 自动安装并运行
- Firecrawl 提供深度网络爬取、搜索和内容提取能力
- API Key 从 `.env` 加载（`FIRECRAWL_API_KEY`）

**MCP 服务器配置格式：**
```yaml
mcp_servers:
  <server-name>:
    command: <可执行命令>      # npx, uvx, python, node, docker
    args: [<参数列表>]
    env:
      KEY: value               # 环境变量（支持 ${VAR} 引用）
```

> **实战建议：** MCP 服务器在代理启动时自动运行。建议将 API Key 放在 `.env` 中，config.yaml 中只写键名引用或留空通过环境变量注入。

---

## 11. 压缩与上下文管理

```yaml
compression:
  enabled: true
  threshold: 0.5
  target_ratio: 0.2
  protect_last_n: 20
  hygiene_hard_message_limit: 400

prompt_caching:
  cache_ttl: 5m
  long_lived_prefix: true
  long_lived_ttl: 1h

context:
  engine: compressor
```

**上下文压缩**是处理长会话的关键：

| 参数 | 值 | 说明 |
|------|----|------|
| `threshold: 0.5` | 50% | 上下文使用超过 50% 时触发压缩 |
| `target_ratio: 0.2` | 20% | 压缩后将上下文缩减到 20% |
| `protect_last_n: 20` | 20条 | 保留最近的 20 条消息不被压缩 |
| `hygiene_hard_message_limit: 400` | 400条 | 消息数达到 400 条强制截断 |

**AI 辅助压缩：** `context.engine: compressor` 使用 LLM 对中间轮次生成摘要，而非简单丢弃。

> **实战建议：** `threshold` 不要设太低（如 0.1），否则每次对话都在压缩，增加 API 开销和延迟。`protect_last_n` 保留最近的决策上下文。

---

## 12. 子代理 (Delegation)

```yaml
delegation:
  model: ''
  provider: ''
  base_url: ''
  api_key: ''
  inherit_mcp_toolsets: true
  max_iterations: 50
  child_timeout_seconds: 600
  reasoning_effort: ''
  max_concurrent_children: 3
  max_spawn_depth: 1
  orchestrator_enabled: true
  subagent_auto_approve: false
```

**子代理系统**让主代理派生子任务并行执行：

| 参数 | 值 | 说明 |
|------|----|------|
| `inherit_mcp_toolsets: true` | 继承 | 子代理继承主代理的 MCP 工具 |
| `max_concurrent_children: 3` | 3个 | 最多同时运行 3 个子代理 |
| `max_spawn_depth: 1` | 1层 | 子代理不能再派生子代理 |
| `orchestrator_enabled: true` | 启用 | 主代理可编排多子任务 |

> **实战建议：** 如果主模型是 DeepSeek-v4-Flash 这类高性能模型，子代理可设不同模型（如 Gemini Flash）来降低成本。设置 `subagent_auto_approve: true` 允许子代理自动批准操作（谨慎使用）。

---

## 13. 显示隐藏的配置节

MaoYo42 的配置中还包含以下重要节：

### 开放路由 (OpenRouter)
```yaml
openrouter:
  response_cache: true
  response_cache_ttl: 300
  min_coding_score: 0.65
```
（虽然在用 DeepSeek 直连，但仍保留了 OpenRouter 配置作为备选）

### 辅助模型 (Auxiliary)
```yaml
auxiliary:
  vision: { provider: auto, model: '', timeout: 120 }
  web_extract: { provider: auto, model: '', timeout: 360 }
  compression: { provider: auto, model: '', timeout: 120 }
  session_search: { provider: auto, model: '', timeout: 30, max_concurrency: 3 }
  skills_hub: { provider: auto, model: '', timeout: 30 }
  mcp: { provider: auto, model: '', timeout: 30 }
  title_generation: { provider: auto, model: '', timeout: 30 }
  triage_specifier: { provider: auto, model: '', timeout: 120 }
  curator: { provider: auto, model: '', timeout: 600 }
```
`provider: auto` 让 Hermes 自动选择可用的辅助推理后端（优先 OpenRouter/Nous Portal，然后依次尝试各 API-Key 提供商）。

### 日志 (Logging)
```yaml
logging:
  level: INFO
  max_size_mb: 5
  backup_count: 3
```
日志级别可选：`DEBUG`、`INFO`、`WARNING`、`ERROR`。调试问题时可临时设为 `DEBUG`。

### 模型目录 (Model Catalog)
```yaml
model_catalog:
  enabled: true
  url: https://hermes-agent.nousresearch.com/docs/api/model-catalog.json
  ttl_hours: 24
```
自动从官方获取最新的模型列表，每日刷新一次。`enabled: false` 可禁用，使用本地手动配置。

### Fallback 模型（注释掉的备选）
```yaml
# fallback_model:
#   provider: openrouter
#   model: anthropic/claude-sonnet-4
```
DeepSeek 不可用时，自动降级到 OpenRouter 上的 Claude Sonnet。

---

## 14. 配置文件核查清单

新用户安装 Hermes Agent 后，建议按以下顺序配置：

1. **`model`** — 设置 provider + default model
2. **`.env`** — 填写 API Key（`DEEPSEEK_API_KEY` 等）
3. **`terminal.backend`** — 选择终端后端（推荐 local 或 docker）
4. **`display.personality`** — 设置交互风格
5. **`memory`** — 启用记忆系统
6. **`curator`** — 启用会话自动管理
7. **`security.redact_secrets`** — 设为 true
8. **`timezone`** — 设置时区
9. **`mcp_servers`** — 添加需要的 MCP 工具
10. **网关平台** — Telegram/Discord 等选配
11. **`fallback_model`** — 配置降级方案
12. **`auxiliary`** — 辅助模型路由

---

## 15. 常见问题

**Q: 修改 config.yaml 后需要重启吗？**
A: 大部分配置（model、display、memory 等）在下次会话启动时生效。`hermes config set` 修改实时生效。

**Q: 配置错误导致代理无法启动怎么办？**
A: 运行 `hermes config check` 检测配置问题，或 `hermes config reset` 恢复默认。

**Q: 如何在多个配置文件间切换？**
A: 使用多个 `~/.hermes/` profile 目录，通过 `HERMES_HOME` 环境变量切换。

---

下一个文档：[03 — Provider 设置指南](03-provider-setup.md)
