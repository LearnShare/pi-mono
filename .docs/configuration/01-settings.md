# 配置详解

> 基于 pi.dev/docs/latest 官方文档整理

---

## 1. 设置文件

### 配置文件位置

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/settings.json` | 全局（所有项目） |
| `.pi/settings.json` | 项目级 |

项目设置覆盖全局设置，嵌套对象合并。

---

## 2. 核心设置项

### 模型与思考

```json
{
  "defaultProvider": "anthropic",
  "defaultModel": "claude-sonnet-4-20250514",
  "defaultThinkingLevel": "medium",
  "hideThinkingBlock": false
}
```

### UI 与显示

```json
{
  "theme": "dark",
  "quietStartup": false,
  "doubleEscapeAction": "tree",
  "editorPaddingX": 0,
  "autocompleteMaxVisible": 5,
  "showHardwareCursor": false
}
```

### 会话压缩

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

### 重试

```json
{
  "retry": {
    "enabled": true,
    "maxRetries": 3,
    "baseDelayMs": 2000
  }
}
```

### 终端与图片

```json
{
  "terminal.showImages": true,
  "terminal.imageWidthCells": 60,
  "images.autoResize": true,
  "images.blockImages": false
}
```

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

支持的 API 类型：`openai-completions`、`openai-responses`、`anthropic-messages`、`google-generative-ai`。