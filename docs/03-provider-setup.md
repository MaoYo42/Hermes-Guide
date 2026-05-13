# 03 — Provider 设置指南 (Provider Setup)

Hermes Agent 支持 **35+ 种推理提供商 (Provider)**，覆盖国际主流 API、国内中文提供商、OAuth 认证、本地自部署等多种场景。本文从 DeepSeek 开始，逐步覆盖所有提供商及高级路由策略。

---

## 目录

1. [快速开始 — DeepSeek（MaoYo42 实战）](#1-快速开始--deepseekmaoyo42-实战)
2. [国际主流 API 提供商](#2-国际主流-api-提供商)
3. [中国区 API 提供商](#3-中国区-api-提供商)
4. [OAuth 认证提供商](#4-oauth-认证提供商)
5. [本地自部署与自定义端点](#5-本地自部署与自定义端点)
6. [高级：凭据池 (Credential Pools)](#6-高级凭据池-credential-pools)
7. [高级：降级提供商 (Fallback Providers)](#7-高级降级提供商-fallback-providers)
8. [高级：Provider 路由 (Provider Routing)](#8-高级provider-路由-provider-routing)
9. [Provider 切换命令](#9-provider-切换命令)
10. [完整 Provider 速查表](#10-完整-provider-速查表)

---

## 1. 快速开始 — DeepSeek（MaoYo42 实战）

### 1.1 配置步骤

**第一步：设置 API Key**

```bash
# 编辑 ~/.hermes/.env
echo "DEEPSEEK_API_KEY=sk-你的Key" >> ~/.hermes/.env
```

DeepSeek 官网：[platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)

**第二步：配置 Provider**

```bash
# 两种方式等效：
# 方式 A：交互式
hermes model
# → 选择 "DeepSeek"

# 方式 B：直接编辑 config.yaml
```

**第三步：验证**

```bash
hermes chat --provider deepseek --model deepseek-v4-flash

# 或者直接启动默认
hermes chat
```

### 1.2 DeepSeek 配置详解

```yaml
# ~/.hermes/config.yaml
model:
  provider: deepseek
  default: deepseek-v4-flash      # 主力模型
  base_url: https://api.deepseek.com/v1
```

| 模型 ID | 类型 | 说明 |
|---------|------|------|
| `deepseek-v4-flash` | 快模型 | 高速推理，适合日常对话和编码 |
| `deepseek-v4` | 标准 | 更强的推理能力，适合复杂任务 |
| `deepseek-r1` | 推理模型 | 深度推理，适合数学/逻辑 |

**配置优化：**

```yaml
# 带 fallback 的完整 DeepSeek 配置
model:
  provider: deepseek
  default: deepseek-v4-flash
  base_url: https://api.deepseek.com/v1

fallback_model:
  provider: openrouter
  model: deepseek/deepseek-v4-flash

# 或者如果你的 DeepSeek Key 也适用于 OpenRouter 上的 DeepSeek
fallback_providers:
  - provider: openrouter
    model: deepseek/deepseek-v4-flash
  - provider: anthropic
    model: claude-sonnet-4-6
```

### 1.3 环境变量参考

```
DEEPSEEK_API_KEY=sk-xxx            # API 密钥（必填）
DEEPSEEK_BASE_URL=                 # 端点覆盖（可选，默认 api.deepseek.com/v1）
```

---

## 2. 国际主流 API 提供商

### 2.1 OpenRouter（推荐作为路由层）

**OpenRouter** 是模型统一网关，通过单一 API Key 访问 200+ 模型，自动路由到最低价/最低延迟的后端。

```bash
# 配置
echo "OPENROUTER_API_KEY=sk-or-xxx" >> ~/.hermes/.env
```

```yaml
model:
  provider: openrouter
  default: anthropic/claude-sonnet-4
```

| 模式 | 模型名格式 | 示例 |
|------|-----------|------|
| Claude | `anthropic/<model>` | `anthropic/claude-sonnet-4` |
| OpenAI | `openai/<model>` | `openai/gpt-5.4` |
| Google | `google/<model>` | `google/gemini-3-flash` |
| DeepSeek | `deepseek/<model>` | `deepseek/deepseek-v4-flash` |
| Meta | `meta-llama/<model>` | `meta-llama/llama-4-maverick` |

**OpenRouter 专属配置：**
```yaml
openrouter:
  response_cache: true          # 启用响应缓存（降低成本）
  response_cache_ttl: 300       # 缓存 TTL：5 分钟
  min_coding_score: 0.65        # 最低编码质量评分

provider_routing:
  sort: "price"                 # price | throughput | latency
  ignore:
    - "Together"                # 排除特定后端
  data_collection: "deny"       # 禁止数据收集
```

> **建议：** OpenRouter 适合需要灵活切换多模型、或作为 fallback 路由层的场景。主用单一模型时，建议直连以获得更低延迟。

### 2.2 Anthropic（Claude 直连）

三种认证方式：

```bash
# 方式 A：API Key（推荐，现收现付）
echo "ANTHROPIC_API_KEY=sk-ant-xxx" >> ~/.hermes/.env

# 方式 B：OAuth（需要 Claude Max + 额外用量积分）
hermes model
# → 选择 Anthropic → OAuth
# 浏览器登录，自动保存凭据

# 方式 C：Claude Code 凭据复用（已安装 Claude Code 的用户自动检测）
hermes chat --provider anthropic
```

配置：
```yaml
model:
  provider: anthropic
  default: claude-sonnet-4-6

# alias: --provider claude 或 --provider claude-code
```

**Alias：** `--provider claude` 和 `--provider claude-code` 都等效于 `--provider anthropic`。

> **注意：** API Key 模式按 token 计费，OAuth 模式需要 Claude Max 计划并购买额外用量积分。Pro 用户不能使用 OAuth 路径。

### 2.3 OpenAI（通过 Codex OAuth）

使用 ChatGPT/OAI 账号 OAuth 认证，获取 Codex 系列模型的访问权限。

```bash
hermes model
# → 选择 "OpenAI Codex"
# → 终端显示 Device Code 和 URL
# → 浏览器打开  https://chatgpt.com/activate
# → 输入设备码 → 授权
# → 完成！
```

配置：
```yaml
model:
  provider: openai-codex
  default: gpt-5.3-codex
```

> **提示：** Hermes 可以自动复用已有的 Codex CLI 凭据（`~/.codex/auth.json`），无需重新认证。

### 2.4 GitHub Copilot

支持两种模式：

**`copilot` — 直接 Copilot API（推荐）：**

```bash
# 认证方式（按优先级）：
# 1. COPILOT_GITHUB_TOKEN 环境变量
# 2. GH_TOKEN 环境变量
# 3. GITHUB_TOKEN 环境变量
# 4. gh auth token

# 或通过 OAuth：
hermes model
# → 选择 "GitHub Copilot" → "Login with GitHub"
```

配置：
```yaml
model:
  provider: copilot
  default: gpt-5.4
```

**`copilot-acp` — Copilot CLI 子进程模式：**
```yaml
model:
  provider: copilot-acp
  default: copilot-acp
# 需要：已安装 GitHub CLI (gh) 并登录 copilot login
```

> **Token 限制：** `ghp_*` 类 PAT 不支持。支持 `gho_*`(OAuth)、`github_pat_*`(Fine-grained PAT) 和 `ghu_*`(GitHub App token)。

### 2.5 Google Gemini

**API Key 模式（推荐用于生产）：**
```bash
echo "GOOGLE_API_KEY=AIza..." >> ~/.hermes/.env
# 或：GEMINI_API_KEY=AIza...
```

```yaml
model:
  provider: gemini
  default: gemini-3-flash
```

**OAuth 模式（含免费额度）：**
```bash
hermes model
# → 选择 "Google Gemini (OAuth)"
# → 浏览器登录 Google 账号
# → 自动使用免费额度
```

```yaml
model:
  provider: google-gemini-cli
  default: gemini-3-pro
```

> **OAuth 模式政策风险：** Google 认为第三方软件使用 Gemini CLI OAuth 客户端违反政策。如需最低风险，请使用 API Key 模式。可设 `HERMES_GEMINI_PROJECT_ID` 使用自己的 GCP 项目。

### 2.6 AWS Bedrock

使用 AWS SDK 认证链 — 无需 API Key，通过 IAM/SSO 认证。

```bash
# 通过 AWS 配置文件
hermes chat --provider bedrock --model us.anthropic.claude-sonnet-4-6

# 或通过环境变量
AWS_PROFILE=myprod AWS_REGION=us-east-1 hermes chat --provider bedrock
```

```yaml
model:
  provider: bedrock
  default: us.anthropic.claude-sonnet-4-6
bedrock:
  region: us-east-1
  # profile: myprofile
  guardrail:
    guardrail_identifier: ''
    guardrail_version: ''
```

> **支持模型：** Claude（Anthropic）、Nova（Amazon）、DeepSeek v3.2、Llama 4（Meta）等。

---

## 3. 中国区 API 提供商

以下提供商在中国大陆有优化的网络连接，部分提供中文优化的模型。

### 3.1 Z.AI / 智谱 GLM

```bash
echo "GLM_API_KEY=xxx" >> ~/.hermes/.env
```

```yaml
model:
  provider: zai
  default: glm-5
  # base_url 可省略，Hermes 自动探测可用端点
```

**自动端点探测：** Hermes 会依次尝试全球、中国、编码变体端点，找到与你的 API Key 兼容的端点后自动缓存。

| 别名 | 说明 |
|------|------|
| `zai` | 标准 provider ID |
| `glm` | 别名 |
| `zhipu` | 别名 |

### 3.2 Kimi / Moonshot

```bash
# 国际端点 (api.moonshot.ai)
echo "KIMI_API_KEY=xxx" >> ~/.hermes/.env

# 中国端点 (api.moonshot.cn)
echo "KIMI_CN_API_KEY=xxx" >> ~/.hermes/.env
```

```yaml
model:
  provider: kimi-coding
  default: kimi-k2.5

# 或中国端点：
# provider: kimi-coding-cn
```

| Provider | 端点 | 环境变量 |
|----------|------|---------|
| `kimi-coding` | api.moonshot.ai | `KIMI_API_KEY` |
| `kimi-coding-cn` | api.moonshot.cn | `KIMI_CN_API_KEY` |

### 3.3 阿里云 DashScope / Qwen

```bash
echo "DASHSCOPE_API_KEY=sk-xxx" >> ~/.hermes/.env
```

```yaml
model:
  provider: alibaba
  default: qwen3.5-plus

# 或 Coding Plan 版本：
# provider: alibaba-coding-plan
```

| Provider | 端点 | 说明 |
|----------|------|------|
| `alibaba` | dashscope.aliyuncs.com | 标准 DashScope API |
| `alibaba-coding-plan` | coding-intl.dashscope.aliyuncs.com | 编码计划专用端点（独立计费） |

### 3.4 MiniMax

```bash
# 全球端点
echo "MINIMAX_API_KEY=xxx" >> ~/.hermes/.env

# 中国端点
echo "MINIMAX_CN_API_KEY=xxx" >> ~/.hermes/.env
```

```yaml
model:
  provider: minimax
  default: MiniMax-M2.7

# 或中国端点：
# provider: minimax-cn
```

| Provider | 环境变量 | 说明 |
|----------|---------|------|
| `minimax` | `MINIMAX_API_KEY` | 全球端点 |
| `minimax-cn` | `MINIMAX_CN_API_KEY` | 中国端点 |
| `minimax-oauth` | (OAuth) | 浏览器登录，无需 API Key |

### 3.5 小米 MiMo

```bash
echo "XIAOMI_API_KEY=xxx" >> ~/.hermes/.env
```

```yaml
model:
  provider: xiaomi
  default: mimo-v2-pro
```

| 别名 | 说明 |
|------|------|
| `xiaomi` | 标准 provider ID |
| `mimo` | 快捷别名 |
| `xiaomi-mimo` | 全称别名 |

### 3.6 其他中国提供商速查

| Provider | 环境变量 | Provider ID | 默认模型 |
|----------|---------|------------|---------|
| **Tencent TokenHub** | `TOKENHUB_API_KEY` | `tencent-tokenhub` | `hy3-preview` |
| **StepFun（阶跃星辰）** | `STEPFUN_API_KEY` | `stepfun` | `step-3-mini` |
| **OpenCode Zen** | `OPENCODE_ZEN_API_KEY` | `opencode-zen` | — |
| **OpenCode Go** | `OPENCODE_GO_API_KEY` | `opencode-go` | — |
| **Kilo Code** | `KILOCODE_API_KEY` | `kilocode` | — |
| **Qwen OAuth** | (OAuth) | `qwen-oauth` | `qwen3-coder-plus` |

---

## 4. OAuth 认证提供商

以下提供商通过浏览器 OAuth 登录，无需手动管理 API Key。

| Provider | 设置命令 | 说明 |
|----------|---------|------|
| **Nous Portal** | `hermes model` → Nous Portal | Nous Research 官方门户，订阅制 |
| **OpenAI Codex** | `hermes model` → OpenAI Codex | 使用 ChatGPT 账号 |
| **Anthropic OAuth** | `hermes model` → Anthropic | 需 Claude Max + 额外用量 |
| **Google Gemini OAuth** | `hermes model` → Google Gemini (OAuth) | 含免费额度 |
| **GitHub Copilot** | `hermes model` → GitHub Copilot | 使用 GitHub 账号 |
| **Qwen Portal** | `hermes model` → Qwen OAuth (Portal) | 阿里通义千问门户 |
| **MiniMax OAuth** | `hermes model` → MiniMax (OAuth) | MiniMax 消费者门户 |

**OAuth 凭据存储位置：** `~/.hermes/auth.json`

**自定义 OAuth 客户端（Gemini 为例）：**
```bash
HERMES_GEMINI_CLIENT_ID=your-client.apps.googleusercontent.com
HERMES_GEMINI_CLIENT_SECRET=xxx
```

---

## 5. 本地自部署与自定义端点

### 5.1 Ollama（本地模型）

```bash
# 安装并启动
ollama pull qwen2.5-coder:32b
ollama serve   # 默认端口 11434

# Hermes 配置
hermes model
# → 选择 "Custom endpoint (self-hosted)"
# → URL: http://localhost:11434/v1
# → API Key: (留空)
# → 模型名: qwen2.5-coder:32b
```

```yaml
model:
  provider: custom
  default: qwen2.5-coder:32b
  base_url: http://localhost:11434/v1
  context_length: 32768   # 重要：Ollama 默认上下文很低
```

> **上下文警告：** Ollama 默认上下文仅 4096 tokens，远低于代理工具的需求。需设环境变量 `OLLAMA_CONTEXT_LENGTH=32768 ollama serve`。

### 5.2 vLLM（高性能 GPU 推理）

```bash
vllm serve meta-llama/Llama-3.1-70B-Instruct \
  --port 8000 \
  --enable-auto-tool-choice --tool-call-parser hermes
```

```yaml
model:
  provider: custom
  default: meta-llama/Llama-3.1-70B-Instruct
  base_url: http://localhost:8000/v1
```

### 5.3 llama.cpp（CPU/Metal）

```bash
./build/bin/llama-server \
  --jinja -fa -c 32768 -ngl 99 \
  -m models/qwen2.5-coder-32b-instruct-Q4_K_M.gguf
```

> `--jinja` 是工具调用的必需参数。没有它，模型不会执行工具调用。

### 5.4 LM Studio（桌面应用）

```bash
hermes model
# → "LM Studio"
# → Enter (默认 http://localhost:1234/v1)
# → 选择发现的模型
```

### 5.5 Hugging Face Inference Providers

通过单一 API Key 路由到 20+ 开源模型后端：

```bash
echo "HF_TOKEN=hf_xxx" >> ~/.hermes/.env
```

```yaml
model:
  provider: huggingface    # 别名: hf
  default: Qwen/Qwen3-235B-A22B-Thinking-2507
```

免费额度：每月 $0.10 积分。模型名可加路由后缀：`:fastest`（默认）、`:cheapest`、`:provider_name`。

---

## 6. 高级：凭据池 (Credential Pools)

**凭据池**让你为同一个 Provider 注册多个 API Key 或 OAuth Token。当一个 Key 遇到速率限制或配额耗尽，Hermes 自动切换到下一个健康 Key。

### 6.1 配置方法

```bash
# 添加第二个 OpenRouter Key
hermes auth add openrouter --api-key sk-or-...-key2

# 添加 Anthropic OAuth（与 API Key 并存）
hermes auth add anthropic --type oauth

# 查看池状态
hermes auth list
```

输出示例：
```
openrouter (2 credentials):
  #1  OPENROUTER_API_KEY   api_key env:OPENROUTER_API_KEY ←
  #2  backup-key           api_key manual

anthropic (3 credentials):
  #1  hermes_pkce          oauth   hermes_pkce ←
  #2  claude_code          oauth   claude_code
  #3  ANTHROPIC_API_KEY    api_key env:ANTHROPIC_API_KEY
```

### 6.2 轮换策略

```yaml
credential_pool_strategies:
  openrouter: round_robin
  anthropic: least_used
```

| 策略 | 行为 |
|------|------|
| `fill_first`（默认） | 使用第一个健康 Key 直到耗尽 |
| `round_robin` | 轮流循环使用 |
| `least_used` | 选请求次数最少的 Key |
| `random` | 随机选择健康 Key |

### 6.3 故障恢复流程

```
请求 → 从池中选 Key → 发送请求
  → 429 限流？
    → 重试同 Key 一次
    → 再次 429 → 切换到下一个 Key
    → 全部 Key 耗尽 → 触发 fallback_model
  → 402 计费不足？ → 立即切换到下个 Key（24h 冷却）
  → 成功 → 继续
```

---

## 7. 高级：降级提供商 (Fallback Providers)

当主力提供商不可用时，Hermes 自动切换到备用提供商，会话不会中断。

### 7.1 配置方法

**交互式添加（推荐）：**
```bash
hermes fallback add
# 选择 provider + 模型
```

**或直接编辑 config.yaml：**
```yaml
# 单个降级（兼容旧版）
fallback_model:
  provider: openrouter
  model: anthropic/claude-sonnet-4

# 多个降级链
fallback_providers:
  - provider: openrouter
    model: deepseek/deepseek-v4-flash
  - provider: anthropic
    model: claude-sonnet-4-6
  - provider: nous
    model: nous-hermes-3
```

### 7.2 触发条件

| HTTP 状态码 | 触发动作 |
|------------|---------|
| 429 限流 | 重试 → 耗尽后触发 fallback |
| 500/502/503 服务错误 | 重试后触发 |
| 401/403 认证失败 | 立即触发（不重试） |
| 404 模型未找到 | 立即触发 |

### 7.3 支持降级的 Provider

| Provider | config.yaml 值 | 凭据 |
|----------|---------------|------|
| OpenRouter | `openrouter` | `OPENROUTER_API_KEY` |
| Anthropic | `anthropic` | `ANTHROPIC_API_KEY` |
| Nous Portal | `nous` | hermes auth (OAuth) |
| OpenAI Codex | `openai-codex` | hermes model (ChatGPT OAuth) |
| GitHub Copilot | `copilot` | `COPILOT_GITHUB_TOKEN` |
| DeepSeek | `deepseek` | `DEEPSEEK_API_KEY` |
| Z.AI/GLM | `zai` | `GLM_API_KEY` |
| Kimi | `kimi-coding` | `KIMI_API_KEY` |
| MiniMax | `minimax` | `MINIMAX_API_KEY` |
| Google Gemini | `gemini` | `GOOGLE_API_KEY` |
| AWS Bedrock | `bedrock` | boto3 认证链 |
| 自定义端点 | `custom` | `base_url` + `key_env` |

### 7.4 自定义端点 fallback

```yaml
fallback_model:
  provider: custom
  model: my-local-model
  base_url: http://localhost:8000/v1
  key_env: MY_LOCAL_KEY     # 环境变量名
```

### 7.5 注意

- Fallback 是**按轮次（per-turn）** 的：每次用户新消息都优先尝试主模型
- **子代理不继承** fallback 配置
- **Cron 任务** 使用固定 provider，不支持 fallback
- Fallback 只能用 **config.yaml** 配置，不支持环境变量

---

## 8. 高级：Provider 路由 (Provider Routing)

仅适用于 **OpenRouter** 提供商，控制 OpenRouter 内部的路由策略。

```yaml
provider_routing:
  sort: "price"                    # price | throughput | latency
  only:                            # 白名单
    - "Anthropic"
    - "Google"
  ignore:                          # 黑名单
    - "Together"
    - "DeepInfra"
  order:                           # 优先级顺序
    - "Anthropic"
    - "Google"
    - "AWS Bedrock"
  require_parameters: true         # 仅路由支持所有参数的提供商
  data_collection: "deny"          # allow | deny
```

**实战场景：**

| 目标 | 配置 |
|------|------|
| 最便宜 | `sort: "price"` |
| 最低延迟 | `sort: "latency"` |
| 最高吞吐量 | `sort: "throughput"` |
| 锁定到 Anthropic | `only: ["Anthropic"]` |
| 避开特定提供商 | `ignore: ["Together"] + data_collection: "deny"` |

---

## 9. Provider 切换命令

### `hermes model` — 全量设置向导

在终端执行（**不在会话内**），用于：
- 添加新 Provider
- 运行 OAuth 登录
- 输入 API Key
- 配置自定义端点

```bash
hermes model
# 显示交互式菜单，选择提供商并设置
```

### `/model` — 会话内快速切换

在聊天会话内使用，**只能切换到已配置好的 Provider**：

```
/model deepseek:deepseek-v4-flash
/model openrouter:anthropic/claude-sonnet-4
/model custom:qwen2.5-coder:32b
/model custom:local:llama3          # 命名自定义端点
/model                                 # 不带参数查看当前模型
```

### CLI 参数覆盖

```bash
hermes chat --provider openrouter --model anthropic/claude-sonnet-4
hermes chat --provider deepseek --model deepseek-v4-flash
hermes chat --provider anthropic --model claude-sonnet-4-6
```

---

## 10. 完整 Provider 速查表

### 国际 API 提供商

| 提供商 | Provider ID | 环境变量 | 默认端点 |
|--------|------------|---------|---------|
| DeepSeek | `deepseek` | `DEEPSEEK_API_KEY` | api.deepseek.com/v1 |
| OpenRouter | `openrouter` | `OPENROUTER_API_KEY` | openrouter.ai/api/v1 |
| Anthropic | `anthropic` | `ANTHROPIC_API_KEY` | api.anthropic.com |
| OpenAI Codex | `openai-codex` | OAuth | chatgpt.com/backend-api/codex |
| GitHub Copilot | `copilot` | `COPILOT_GITHUB_TOKEN` | api.githubcopilot.com |
| Google Gemini | `gemini` | `GOOGLE_API_KEY` | generativelanguage.googleapis.com |
| AWS Bedrock | `bedrock` | boto3 链 | — |
| xAI Grok | `xai` | `XAI_API_KEY` | api.x.ai |
| NVIDIA NIM | `nvidia` | `NVIDIA_API_KEY` | integrate.api.nvidia.com/v1 |
| Hugging Face | `huggingface` | `HF_TOKEN` | router.huggingface.co/v1 |
| Ollama Cloud | `ollama-cloud` | `OLLAMA_API_KEY` | ollama.com/v1 |
| GMI Cloud | `gmi` | `GMI_API_KEY` | api.gmi-serving.com/v1 |
| Arcee AI | `arcee` | `ARCEEAI_API_KEY` | api.arcee.ai |
| AI Gateway | `ai-gateway` | `AI_GATEWAY_API_KEY` | — |
| Azure AI Foundry | `azure-foundry` | `AZURE_FOUNDRY_API_KEY` | — |
| LM Studio | `lmstudio` | (本地) | localhost:1234/v1 |
| Nous Portal | `nous` | OAuth | portal.nousresearch.com |
| Kilo Code | `kilocode` | `KILOCODE_API_KEY` | — |

### 中国区提供商

| 提供商 | Provider ID | 环境变量 | 说明 |
|--------|------------|---------|------|
| 智谱 GLM | `zai` | `GLM_API_KEY` | 端点自动探测 |
| Kimi/Moonshot | `kimi-coding` | `KIMI_API_KEY` | 国际端点 |
| Kimi/Moonshot CN | `kimi-coding-cn` | `KIMI_CN_API_KEY` | 中国端点 |
| 阿里 DashScope | `alibaba` | `DASHSCOPE_API_KEY` | Qwen 系列 |
| 阿里 Coding Plan | `alibaba-coding-plan` | `DASHSCOPE_API_KEY` | 独立计费 |
| MiniMax | `minimax` | `MINIMAX_API_KEY` | 全球端点 |
| MiniMax CN | `minimax-cn` | `MINIMAX_CN_API_KEY` | 中国端点 |
| 小米 MiMo | `xiaomi` | `XIAOMI_API_KEY` | 小米自研 |
| 腾讯 TokenHub | `tencent-tokenhub` | `TOKENHUB_API_KEY` | 混元 |
| 阶跃星辰 | `stepfun` | `STEPFUN_API_KEY` | Step 系列 |
| OpenCode Zen | `opencode-zen` | `OPENCODE_ZEN_API_KEY` | — |
| OpenCode Go | `opencode-go` | `OPENCODE_GO_API_KEY` | — |
| Qwen Portal | `qwen-oauth` | OAuth | 阿里消费者入口 |

### OAuth 提供商

| 提供商 | 设置方式 | 备注 |
|--------|---------|------|
| Nous Portal | `hermes model` → Nous Portal | 需要订阅 |
| OpenAI Codex | `hermes model` → OpenAI Codex | 需 ChatGPT 账号 |
| Anthropic | `hermes model` → Anthropic | 需 Claude Max + 额外用量 |
| Google Gemini | `hermes model` → Google Gemini (OAuth) | 含免费额度 |
| GitHub Copilot | `hermes model` → GitHub Copilot | 需 GitHub Copilot 订阅 |
| Qwen Portal | `hermes model` → Qwen OAuth (Portal) | 阿里账号 |
| MiniMax | `hermes model` → MiniMax (OAuth) | MiniMax 消费者入口 |

### 自定义/自部署

| 场景 | Provider ID | 说明 |
|------|------------|------|
| Ollama 本地 | `custom` | localhost:11434/v1 |
| vLLM 本地 | `custom` | localhost:8000/v1 |
| llama.cpp | `custom` | localhost:8080/v1 |
| SGLang 本地 | `custom` | localhost:30000/v1 |
| LM Studio | `lmstudio` | localhost:1234/v1 |
| 任何 OpenAI 兼容 API | `custom` | 任意外部端点 |
| 命名自定义端点 | `custom:<name>` | 多个端点区分 |

---

## 附：MaoYo42 实际配置参考

MaoYo42 的 provider 配置总结：

```yaml
# 主力：DeepSeek 直连
model:
  provider: deepseek
  default: deepseek-v4-flash
  base_url: https://api.deepseek.com/v1

# 凭据池
# auth.json 中有 DeepSeek API Key + Codex OAuth
# .env 中有 DEEPSEEK_API_KEY + TELEGRAM_BOT_TOKEN + FIRECRAWL_API_KEY

# 备选：OpenRouter（注释掉的 fallback）
# fallback_model:
#   provider: openrouter
#   model: anthropic/claude-sonnet-4
```

**选择 DeepSeek 的理由：**
1. 国内直连，延迟最低
2. DeepSeek-v4-Flash 编码能力强
3. 价格合理（比 OpenAI GPT-4 便宜 10 倍+）
4. 保留 OpenRouter fallback 备用

---

上一个文档：[01 — 配置文件详解](02-configuration.md)
