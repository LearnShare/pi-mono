# 提供商设置

> 基于 pi.dev/docs/latest 官方文档整理

---

## 1. 认证方式

### 订阅登录（/login）

在交互模式下运行 `/login`，选择提供商进行 OAuth 认证。

| 提供商 | 说明 |
|--------|------|
| Claude Pro/Max | Anthropic 订阅 |
| ChatGPT Plus/Pro (Codex) | OpenAI 订阅 |
| GitHub Copilot | GitHub Copilot 订阅（支持企业版） |
| Google Gemini CLI | Google 免费/付费账户 |
| Google Antigravity | Google 沙盒环境 |

令牌存储在 `~/.pi/agent/auth.json`，过期自动刷新。

### API Key 认证

设置环境变量：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

或者在 `/login` 中选择 API Key 提供商，将密钥存入 `auth.json`。

---

## 2. 支持的提供商

### API Key 提供商

| 提供商 | 环境变量 | auth.json key |
|--------|---------|---------------|
| Anthropic | `ANTHROPIC_API_KEY` | `anthropic` |
| OpenAI | `OPENAI_API_KEY` | `openai` |
| DeepSeek | `DEEPSEEK_API_KEY` | `deepseek` |
| Google Gemini | `GEMINI_API_KEY` | `google` |
| Mistral | `MISTRAL_API_KEY` | `mistral` |
| Groq | `GROQ_API_KEY` | `groq` |
| xAI | `XAI_API_KEY` | `xai` |
| OpenRouter | `OPENROUTER_API_KEY` | `openrouter` |
| Hugging Face | `HF_TOKEN` | `huggingface` |

### 云提供商

| 提供商 | 所需环境变量 |
|--------|-------------|
| Azure OpenAI | `AZURE_OPENAI_API_KEY` + `AZURE_OPENAI_BASE_URL` |
| Amazon Bedrock | AWS Profile / IAM Keys / Bearer Token |
| Cloudflare Workers AI | `CLOUDFLARE_API_KEY` + `CLOUDFLARE_ACCOUNT_ID` |
| Google Vertex AI | Application Default Credentials |

---

## 3. 自定义模型

通过 `~/.pi/agent/models.json` 添加本地模型（Ollama、vLLM、LM Studio）：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        { "id": "llama3.1:8b" },
        { "id": "qwen2.5-coder:7b" }
      ]
    }
  }
}
```

### 支持的 API 类型

- `openai-completions`
- `openai-responses`
- `anthropic-messages`
- `google-generative-ai`