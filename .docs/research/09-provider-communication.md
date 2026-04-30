# Provider 通信机制研究

> 基于 pi-mono 源码深度分析
> 研究主题：02-detailed-research-plan.md 主题2

---

## 执行摘要

本文档深入分析 pi-coding-agent 与 20+ LLM 提供商之间的通信机制，涵盖请求构建、流式响应处理、Provider 特定功能等核心机制。

---

## 1. Provider 架构概览

### 1.1 支持的 Provider

| Provider | API 类型 | 说明 |
|----------|----------|------|
| Anthropic | anthropic-messages | Claude 系列模型 |
| OpenAI | openai-responses / openai-completions | GPT 系列模型 |
| Google | google-generative-ai | Gemini 系列模型 |
| Azure OpenAI | azure-openai-responses | Azure 托管 OpenAI |
| Amazon Bedrock | bedrock-converse-stream | AWS Bedrock 模型 |
| DeepSeek | openai-responses | DeepSeek 模型 |
| Mistral | mistral-conversations | Mistral 模型 |
| Groq | openai-responses | Groq 加速模型 |
| xAI | openai-responses | Grok 模型 |
| OpenRouter | openai-responses | 聚合 API |
| Hugging Face | openai-responses | Hugging Face 推理端点 |
| Cloudflare Workers AI | cloudflare | Workers AI |
| Google Vertex AI | google-vertex | GCP Vertex AI |
| + 更多... | | |

### 1.2 核心类型定义

```typescript
// packages/ai/src/types.ts
export type KnownApi =
  | "openai-completions"
  | "openai-responses"
  | "anthropic-messages"
  | "bedrock-converse-stream"
  | "google-generative-ai"
  | "google-vertex"
  | "mistral-conversations"
  // ...
```

---

## 2. 请求构建流程

### 2.1 消息转换

**入口：** `transformMessages()`

```typescript
// packages/ai/src/providers/transform-messages.ts
export function transformMessages<TApi extends Api>(
  messages: Message[],
  model: Model<TApi>,
  normalizeToolCallId?: ...
): Message[]
```

**转换步骤：**

1. **图片降级**：不支持视觉的模型，将图片替换为占位符
   ```typescript
   NON_VISION_USER_IMAGE_PLACEHOLDER = "(image omitted: model does not support images)"
   ```

2. **Thinking 块处理**：跨模型时丢弃思考内容
   ```typescript
   // Redacted thinking is opaque encrypted content, only valid for the same model.
   // Drop it for cross-model to avoid API errors.
   ```

3. **工具调用 ID 归一化**：不同 Provider 工具调用 ID 格式不同
   ```typescript
   // OpenAI: 450+ chars with special characters like `|`
   // Anthropic: ^[a-zA-Z0-9_-]+$ (max 64 chars)
   ```

### 2.2 角色映射

| pi 消息角色 | Anthropic | OpenAI | Google |
|------------|-----------|--------|--------|
| `user` | `user` | `user` | `user` |
| `assistant` | `assistant` | `assistant` | `model` |
| `toolResult` | `tool` | `tool` | `tool` |
| `system` | 独立字段 | 消息数组中的 `system` 角色 | `system_instruction` 字段 |

### 2.3 内容序列化

**Content 类型：**

```typescript
type Content = TextContent | ImageContent | ThinkingContent | ToolCall;

interface TextContent {
  type: "text";
  text: string;
}

interface ImageContent {
  type: "image";
  source: "base64" | "url";
  mediaType: string;
  data: string;
}

interface ThinkingContent {
  type: "thinking";
  thinking: string;  // Base64 编码的思考内容
}
```

---

## 3. 流式响应处理

### 3.1 事件流架构

**核心类：** `AssistantMessageEventStream`

```typescript
// packages/ai/src/utils/event-stream.ts
export class AssistantMessageEventStream extends EventStream<AssistantMessageEvent, AssistantMessage> {
  constructor() {
    super(
      (event) => event.type === "done" || event.type === "error",
      (event) => event.type === "done" ? event.message : event.error
    );
  }
}
```

### 3.2 事件类型

| 事件 | 说明 |
|------|------|
| `start` | 流开始，包含初始 AssistantMessage |
| `text_start` | 文本块开始 |
| `text_delta` | 文本增量 |
| `text_end` | 文本块结束 |
| `thinking_start` | 思考块开始 |
| `thinking_delta` | 思考增量 |
| `thinking_end` | 思考块结束 |
| `toolcall_start` | 工具调用开始 |
| `toolcall_delta` | 工具调用 JSON 增量 |
| `toolcall_end` | 工具调用结束 |
| `done` | 流成功结束 |
| `error` | 流错误结束 |

### 3.3 使用示例

```typescript
const stream = await streamAnthropic(model, context, options);

for await (const event of stream) {
  switch (event.type) {
    case "start":
      // 初始化 UI
      break;
    case "text_delta":
      // 增量显示文本
      break;
    case "toolcall_delta":
      // 增量构建工具调用参数
      break;
    case "done":
      // 完成处理
      break;
  }
}

const finalMessage = await stream.result();
```

---

## 4. Provider 特定处理

### 4.1 Anthropic 特有事件

```typescript
// Anthropic SDK 事件映射到 pi 事件
contentBlock_delta        → text_delta / thinking_delta
message_delta             → usage 更新
message_stop              → done
content_block_stop        → text_end / thinking_end
```

**思考块处理：**
- Anthropic 的思考内容作为独立的 `thinking` content block
- 存储为 Base64 编码的加密内容
- 仅对同一模型有效，跨模型时丢弃

**缓存控制：**
```typescript
function getCacheControl(model, cacheRetention?: CacheRetention) {
  const retention = resolveCacheRetention(cacheRetention);
  return {
    retention,
    cacheControl: {
      type: "ephemeral",
      ttl: retention === "long" ? "1h" : undefined
    }
  };
}
```

### 4.2 OpenAI 特有处理

**工具调用格式：**
- `function_call` (旧版)
- `tool_calls` (新版 Responses API)

**Stop Reasons：**
- `stop`: 正常完成
- `length`: 达到 maxTokens
- `tool_use`: 需要工具调用

### 4.3 Google 特有处理

**系统提示位置：** `system_instruction` 字段

**思考格式：**
- OpenAI o1 系列：`reasoning` 字段
- Google：透明的思考过程

**特殊字段：**
- `thoughtSignature`: 用于复用思考上下文

---

## 5. 工具定义序列化

### 5.1 工具 Schema 转换

pi 使用 TypeBox 定义工具参数：

```typescript
import { Type } from "typebox";

const tool = {
  name: "my_tool",
  description: "Does something",
  parameters: Type.Object({
    input: Type.String(),
    count: Type.Optional(Type.Integer()),
  })
};
```

转换为 Provider 格式：

| Provider | 格式 |
|----------|------|
| Anthropic | JSON Schema |
| OpenAI | JSON Schema / function 定义 |
| Google | JSON Schema（不支持 `anyOf`） |

### 5.2 枚举处理重要约定

```typescript
import { StringEnum } from "@mariozechner/pi-ai";

// 正确 - 与 Google 兼容
action: StringEnum(["list", "add"] as const)

// 错误 - 不能与 Google 兼容
action: Type.Union([Type.Literal("list"), Type.Literal("add")])
```

---

## 6. Usage 追踪

### 6.1 Usage 结构

```typescript
interface Usage {
  input: number;        // 输入 token 数
  output: number;       // 输出 token 数
  cacheRead: number;    // 缓存读取 token
  cacheWrite: number;   // 缓存写入 token
  totalTokens: number;  // 总 token 数
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
    total: number;
  };
}
```

### 6.2 Provider 差异

| Provider | cacheRead/cacheWrite | 说明 |
|----------|---------------------|------|
| Anthropic | 支持 | 完整缓存 API |
| OpenAI | 部分 | 仅部分模型支持 |
| Google | 不支持 | 无缓存功能 |

---

## 7. Stop Reason 映射

| Provider | pi StopReason | 说明 |
|----------|---------------|------|
| Anthropic | `stop`, `length`, `tool_use` | 完整支持 |
| OpenAI | `stop`, `length`, `tool_use` | 完整支持 |
| Google | `stop`, `length`, `RECITATION`, `SAFETY` | 特殊原因 |

---

## 8. 关键源码位置

| 文件 | 作用 |
|------|------|
| `packages/ai/src/types.ts` | 通用类型定义 |
| `packages/ai/src/providers/*.ts` | 20+ Provider 实现 |
| `packages/ai/src/context.ts` | 请求上下文构建 |
| `packages/ai/src/utils/event-stream.ts` | 流式事件发射 |
| `packages/ai/src/providers/transform-messages.ts` | 消息转换 |

---

## 9. 研究问题解答

### Q1: 如何将 AgentMessage 转换为 Provider Payload？

通过 `transformMessages()` 函数：
1. 图片降级（不支持视觉的模型）
2. Thinking 块处理（跨模型丢弃）
3. 工具调用 ID 归一化

### Q2: 流式响应的完整事件顺序？

```
start → [text_start → text_delta* → text_end]*
       → [thinking_start → thinking_delta* → thinking_end]*
       → [toolcall_start → toolcall_delta* → toolcall_end]*
       → done / error
```

### Q3: 各 Provider 如何处理 system prompt？

- Anthropic: 独立 `system` 字段
- OpenAI: 消息数组中的 `role: "system"` 消息
- Google: `system_instruction` 字段

---

## 10. 流程图

### 10.1 请求构建流程

```
AgentSession
    │
    ▼
┌─────────────────────┐
│ 构建 Context        │
│ - systemPrompt      │
│ - messages          │
│ - tools             │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 转换 Messages       │
│ - 图片降级          │
│ - Thinking 处理     │
│ - 工具调用 ID 归一化 │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 调用 Provider       │
│ streamAnthropic()   │
│ streamOpenAI()      │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 处理流式事件        │
│ - text_delta        │
│ - thinking_delta    │
│ - toolcall_delta    │
└────────┬────────────┘
         ▼
    ┌────┴────┐
    │ done    │
    └─────────┘
```

---

*本文档为研究产出，后续可根据需要进一步补充和修正。*