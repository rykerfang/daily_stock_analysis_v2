# 腾讯云 TokenHub DeepSeek 接入指南

本文档说明如何在 `daily_stock_analysis_v2` 项目中通过 GitHub Actions 使用腾讯云 TokenHub 的 DeepSeek 模型进行每日 AI 分析。

## 架构概览

```
GitHub Actions (cron) → main.py → LLM Channels → TokenHub API → DeepSeek-V4-Pro
                                    ↓
                              LiteLLM Router (OpenAI 兼容协议)
```

项目已有的 LLM 渠道系统（`LLM_CHANNELS`）原生支持 OpenAI 兼容协议，TokenHub 正好是 OpenAI 兼容的，因此**无需修改任何项目代码**，只需配置环境变量即可。

## 前置条件

1. 腾讯云账号已开通 TokenHub 服务
2. 已获取 TokenHub API Key（在 [腾讯云控制台](https://console.cloud.tencent.com/tcr/tokenhub) 获取）

## GitHub 仓库配置步骤

### Step 1: 配置 Secrets

进入 GitHub 仓库 → Settings → Secrets and variables → Actions → **Repository secrets**，添加：

| Secret 名称 | 值 | 说明 |
|---|---|---|
| `TOKENHUB_API_KEY` | `你的TokenHub API Key` | 主 API Key |

> 如果有多个 Key（用于轮询负载均衡），可以配置 `TOKENHUB_API_KEYS`（逗号分隔多个 Key），优先于单个 `TOKENHUB_API_KEY`。

### Step 2: 配置 Variables

进入 GitHub 仓库 → Settings → Secrets and variables → Actions → **Variables**，添加：

| Variable 名称 | 值 | 说明 |
|---|---|---|
| `LLM_CHANNELS` | `tokenhub` | 激活 TokenHub 渠道 |
| `LLM_TOKENHUB_MODELS` | `deepseek-v4-pro` | 使用的模型（默认已在 workflow 中设置，可覆盖） |

### 可选 Variables（一般不需要修改）

| Variable 名称 | 默认值 | 说明 |
|---|---|---|
| `LLM_TOKENHUB_PROTOCOL` | `openai` | 协议类型，TokenHub 兼容 OpenAI |
| `LLM_TOKENHUB_BASE_URL` | `https://tokenhub.tencentmaas.com/v1` | API 地址（广州区域） |
| `LLM_TOKENHUB_ENABLED` | `true` | 是否启用 |

### Step 3: 配置自选股（必须）

| Variable 名称 | 示例值 | 说明 |
|---|---|---|
| `STOCK_LIST` | `sh600519,sz000001,hk00700,usAAPL` | 你的自选股列表 |

代码格式：
- 沪市: `sh` + 6位代码（如 `sh600519`）
- 深市: `sz` + 6位代码（如 `sz000001`）
- 港股: `hk` + 5位代码（如 `hk00700`）
- 美股: `us` + 代码（如 `usAAPL`）

### Step 4: 配置通知渠道（至少选一个）

选择你要接收报告的渠道，配置对应的 Secret/Variable：

| 渠道 | 关键配置 |
|---|---|
| 企业微信 | `WECHAT_WEBHOOK_URL` |
| 飞书 | `FEISHU_WEBHOOK_URL` |
| Telegram | `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` |
| 邮件 | `EMAIL_SENDER` + `EMAIL_PASSWORD` + `EMAIL_RECEIVERS` |
| PushPlus | `PUSHPLUS_TOKEN` |
| Discord | `DISCORD_WEBHOOK_URL` |

## 高级配置

### 多模型 Fallback

如果想配置 TokenHub 为主力模型，其他渠道作为降级备选：

```
LLM_CHANNELS=tokenhub,deepseek
LITELLM_MODEL=openai/deepseek-v4-pro
LITELLM_FALLBACK_MODELS=deepseek/deepseek-chat
```

这样当 TokenHub 不可用时，会自动切换到 DeepSeek 官方 API。

### 新加坡区域

如果使用新加坡区域的 TokenHub：

| Variable 名称 | 值 |
|---|---|
| `LLM_TOKENHUB_BASE_URL` | `https://tokenhub-intl.tencentmaas.com/v1` |

> 注意：不支持跨地域调用，请确保使用与服务开通地域一致的接口地址。

### 备用域名

如果默认域名不可用，可切换备用域名：

| 区域 | 默认地址 | 备用地址 |
|---|---|---|
| 广州 | `https://tokenhub.tencentmaas.com/v1` | `https://tokenhub.tencentmaas.cn/v1` |
| 新加坡 | `https://tokenhub-intl.tencentmaas.com/v1` | `https://tokenhub-intl.tencentmaas.cn/v1` |

### 使用其他 TokenHub 模型

TokenHub 支持多种模型，可按需配置 `LLM_TOKENHUB_MODELS`：

| 模型名称 | model 参数值 | 特点 |
|---|---|---|
| DeepSeek-V4-Pro | `deepseek-v4-pro` | 深度推理，推荐用于分析 |
| DeepSeek-V4-Flash | `deepseek-v4-flash` | 快速响应，成本更低 |
| DeepSeek-V3.2 | `deepseek-v3.2` | 上一代模型 |
| GLM-5.1 | `glm-5.1` | 智谱模型 |
| Kimi-K2.6 | `kimi-k2.6` | 月之暗面模型 |
| MiniMax-M3 | `minimax-m3` | MiniMax 模型 |

配置多个模型（逗号分隔）：

```
LLM_TOKENHUB_MODELS=deepseek-v4-pro,deepseek-v4-flash
```

## 验证配置

### 手动触发测试

1. 进入 GitHub 仓库 → Actions → 每日股票分析
2. 点击 "Run workflow"
3. 选择模式 `full`，勾选 `force_run`（跳过交易日检查）
4. 运行完成后查看日志，确认：
   - `TokenHub Key: ✅ 已配置`
   - 分析正常完成，报告已生成

### 常见问题

**Q: 报错 `No LLM configured`**
A: 检查 `LLM_CHANNELS` 变量是否设置为 `tokenhub`，以及 `TOKENHUB_API_KEY` Secret 是否已配置。

**Q: 报错 401 Unauthorized**
A: API Key 错误或过期。确认 Key 来自正确的地域（广州/新加坡），且 Key 有效。

**Q: 报错跨地域调用失败**
A: `LLM_TOKENHUB_BASE_URL` 必须与服务开通地域一致。广州用户用 `tokenhub.tencentmaas.com`，新加坡用户用 `tokenhub-intl.tencentmaas.com`。

**Q: 分析结果为空或 JSON 解析失败**
A: DeepSeek-V4-Pro 输出较长，建议检查 `max_tokens` 是否足够。项目已内置 JSON 修复机制。

## 定时调度

Workflow 默认在每周一至周五北京时间 18:00 自动运行。修改 cron 表达式可调整时间：

```yaml
schedule:
  - cron: '0 10 * * 1-5'  # UTC 10:00 = 北京时间 18:00
```

如需改为北京时间 8:00 运行：

```yaml
schedule:
  - cron: '0 0 * * 1-5'  # UTC 0:00 = 北京时间 8:00
```

## 技术原理

### LLM Channels 工作机制

项目使用 LiteLLM 作为统一的 LLM 调用层，通过 `LLM_CHANNELS` 环境变量配置渠道：

1. `LLM_CHANNELS=tokenhub` → 激活 tokenhub 渠道
2. 系统读取 `LLM_TOKENHUB_*` 系列环境变量
3. 构建 LiteLLM Router model_list：
   ```python
   {
       "model_name": "openai/deepseek-v4-pro",
       "litellm_params": {
           "model": "openai/deepseek-v4-pro",
           "api_key": "<your_key>",
           "api_base": "https://tokenhub.tencentmaas.com/v1"
       }
   }
   ```
4. LiteLLM 以 OpenAI 兼容协议调用 TokenHub API

### API 调用流程

```
main.py → GeminiAnalyzer → LiteLLM Router → TokenHub API
                                              ↓
                              POST https://tokenhub.tencentmaas.com/v1/chat/completions
                              Headers: Authorization: Bearer <api_key>
                              Body: { "model": "deepseek-v4-pro", "messages": [...] }
```

## 参考资料

- [腾讯云 TokenHub DeepSeek 模型接入文档](https://cloud.tencent.com/document/product/1823/130078)
- [TokenHub 语言模型调用概览](https://cloud.tencent.com/document/product/1823/130079)
- [项目 README](https://github.com/rykerfang/daily_stock_analysis_v2#readme)
