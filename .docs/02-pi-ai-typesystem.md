# pi-ai 类型系统与 API 提供商

## 概述

pi-ai 是统一的 LLM API 层，抽象了 20+ LLM 提供商的差异，提供统一的类型系统、流式接口和错误处理。

**核心目标**:
- 统一不同提供商的 API 差异
- 标准化消息协议和事件流
- 支持 thinking/reasoning
- 多模态输入/输出

## 核心类型

### 文件位置

`packages/ai/src/types.ts` - 核心类型定义 (~412 行)

### API 类型标识符

```typescript
// 定义的 API 类型
export type KnownApi =
  | "openai-completions"      // OpenAI Completions API (GPT-4o 等)
  | "mistral-conversations"    // Mistral Conversations API
  | "openai-responses"       // OpenAI Responses API (带工具调用)
  | "azure-openai-responses" // Azure OpenAI Responses
  | "openai-codex-responses" // OpenAI Codex Responses (GitHub Copilot)
  | "anthropic-messages"    // Anthropic Messages API
  | "bedrock-converse-stream" // Amazon Bedrock Converse Stream
  | "google-generative-ai"   // Google Generative AI (Gemini)
  | "google-gemini-cli"      // Google Gemini CLI
  | "google-vertex";        // Google Vertex AI

// 支持扩展
export type Api = KnownApi | (string & {});
```

### 提供商标识符

```typescript
export type KnownProvider =
  // 官方支持
  | "amazon-bedrock"
  | "anthropic"
  | "google"
  | "google-gemini-cli"
  | "google-antigravity"
  | "google-vertex"
  | "openai"
  | "azure-openai-responses"
  | "openai-codex"
  | "github-copilot"
  // 第三方
  | "xai"
  | "groq"
  | "cerebras"
  | "openrouter"
  | "vercel-ai-gateway"
  | "zai"
  | "mistral"
  | "minimax"
  | "minimax-cn"
  | "huggingface"
  | "opencode"
  | "opencode-go"
  | "kimi-coding";

// 支持扩展
export type Provider = KnownProvider | string;
```

### 模型接口

```typescript
export interface Model<TApi extends Api> {
  // 标识
  id: string;           // 模型 ID (如 "claude-sonnet-4-20250514")
  name: string;        // 显示名称
  api: TApi;          // API 类型
  provider: Provider; // 提供商
  
  // 端点
  baseUrl: string;      // API 基础 URL
  
  // 能力
  reasoning: boolean; // 是否支持 reasoning/thinking
  input: ("text" | "image")[];  // 支持的输入模式
  
  // 定价 ($/million tokens)
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };
  
  // 限制
  contextWindow: number;  // 上下文窗口 (tokens)
  maxTokens: number;      // 最大输出 tokens
  
  // 可选
  headers?: Record<string, string>;  // 自定义请求头
  compat?: OpenAICompletionsCompat; // 兼容性配置
}
```

## 消息协议

### 用户消息

```typescript
export interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;  // Unix 时间戳 (毫秒)
}
```

内容可以是纯文本或多媒体数组。

### 助手消息

```typescript
export interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  
  // 元数据
  api: Api;
  provider: Provider;
  model: string;
  responseId?: string;        // 提供商返回的消息 ID
  usage: Usage;
  stopReason: StopReason;
  errorMessage?: string;
  timestamp: number;
}
```

### 工具结果消息

```typescript
export interface ToolResultMessage<TDetails = any> {
  role: "toolResult";
  toolCallId: string;     // 关联的工具调用 ID
  toolName: string;     // 工具名称
  content: (TextContent | ImageContent)[];  // 结果内容
  details?: TDetails;   // 扩展元数据
  isError: boolean;
  timestamp: number;
}
```

### 内容类型

```typescript
// 文本内容
export interface TextContent {
  type: "text";
  text: string;
  textSignature?: string;  // 元数据 (如 OpenAI message ID)
}

// 思考内容 (reasoning/thinking)
export interface ThinkingContent {
  type: "thinking";
  thinking: string;
  thinkingSignature?: string;
  redacted?: boolean;  // 是否被安全过滤器编辑
}

// 图片内容
export interface ImageContent {
  type: "image";
  data: string;    // Base64 编码
  mimeType: string;  // "image/jpeg", "image/png", etc.
}

// 工具调用
export interface ToolCall {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, any>;
  thoughtSignature?: string;  // Google 特有
}
```

## 使用量与成本

```typescript
export interface Usage {
  input: number;       // 输入 tokens
  output: number;     // 输出 tokens
  cacheRead: number;  // 缓存读取 tokens
  cacheWrite: number; // 缓存写入 tokens
  totalTokens: number;
  
  // 成本 ($)
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
    total: number;
  };
}
```

### 停止原因

```typescript
export type StopReason =
  | "stop"       // 正常完成
  | "length"    // 达到 maxTokens 限制
  | "toolUse"   // 工具调用
  | "error"     // 错误
  | "aborted"; // 被中止
```

## 流选项

### 基础流选项

```typescript
export interface StreamOptions {
  temperature?: number;       // 0-2，控制创造性
  maxTokens?: number;         // 最大输出 tokens
  signal?: AbortSignal;       //中止信号
  apiKey?: string;           // API key (覆盖环境变量)
  
  // 传输方式
  transport?: Transport;     // "sse" | "websocket" | "auto"
  
  // 缓存
  cacheRetention?: CacheRetention;  // "none" | "short" | "long"
  sessionId?: string;              // 会话 ID (用于缓存)
  
  // 回调
  onPayload?: (payload: unknown, model: Model) => unknown;
  onResponse?: (response: ProviderResponse, model: Model) => void;
  
  // 头和元数据
  headers?: Record<string, string>;
  metadata?: Record<string, unknown>;
  
  // 重试
  maxRetryDelayMs?: number;
}

export type CacheRetention = "none" | "short" | "long";
export type Transport = "sse" | "websocket" | "auto";
```

### 简化流选项 (推荐)

```typescript
export interface SimpleStreamOptions extends StreamOptions {
  reasoning?: ThinkingLevel;         // 推理级别
  thinkingBudgets?: ThinkingBudgets;  // token 预算
}

// Thinking 级别
export type ThinkingLevel = "minimal" | "low" | "medium" | "high" | "xhigh";

export interface ThinkingBudgets {
  minimal?: number;
  low?: number;
  medium?: number;
  high?: number;
}
```

## 上下文

```typescript
export interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}

// 工具定义
export interface Tool<TParameters extends TSchema = TSchema> {
  name: string;
  description: string;
  parameters: TParameters;
}
```

## API 提供商注册

### 文件位置

`packages/ai/src/api-registry.ts`

### 注册接口

```typescript
export interface ApiProvider<TApi extends Api, TOptions extends StreamOptions> {
  api: TApi;
  stream: StreamFunction<TApi, TOptions>;
  streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}

type RegisteredApiProvider = {
  provider: ApiProviderInternal;
  sourceId?: string;
};

// 注册表
const apiProviderRegistry = new Map<string, RegisteredApiProvider>();

// 注册提供商
export function registerApiProvider<TApi, TOptions>(
  provider: ApiProvider<TApi, TOptions>,
  sourceId?: string,
): void;

// 获取提供商
export function getApiProvider(api: Api): ApiProviderInternal | undefined;
```

### 注册示例

```typescript
// register-builtins.ts 中的注册
registerApiProvider({
  api: "anthropic-messages",
  stream: streamAnthropic,
  streamSimple: streamSimpleAnthropic,
});

registerApiProvider({
  api: "openai-responses",
  stream: streamOpenAIResponses,
  streamSimple: streamSimpleOpenAIResponses,
});

// etc...
```

## 内置提供商的延迟加载

### 文件位置

`packages/ai/src/providers/register-builtins.ts` (~433 行)

### 延迟加载模式

```typescript
// 1. 定义模块加载器
function loadAnthropicProviderModule() {
  anthropicProviderModulePromise ||= import("./anthropic.js").then((module) => {
    const provider = module as AnthropicProviderModule;
    return {
      stream: provider.streamAnthropic,
      streamSimple: provider.streamSimpleAnthropic,
    };
  });
  return anthropicProviderModulePromise;
}

// 2. 创建延迟流函数
function createLazyStream<TApi, TOptions, TSimpleOptions>(
  loadModule: () => Promise<LazyProviderModule<TApi, TOptions, TSimpleOptions>>,
): StreamFunction<TApi, TOptions> {
  return (model, context, options) => {
    const outer = new AssistantMessageEventStream();
    
    loadModule()
      .then((module) => {
        const inner = module.stream(model, context, options);
        // 转发事件
        forwardStream(outer, inner);
      })
      .catch((error) => {
        const message = createLazyLoadErrorMessage(model, error);
        outer.push({ type: "error", reason: "error", error: message });
        outer.end(message);
      });
    
    return outer;
  };
}

// 3. 导出延迟流函数
export const streamAnthropic = createLazyStream(loadAnthropicProviderModule);
export const streamSimpleAnthropic = createLazySimpleStream(loadAnthropicProviderModule);

// 4. 注册内置提供商
export function registerBuiltInApiProviders(): void {
  registerApiProvider({
    api: "anthropic-messages",
    stream: streamAnthropic,
    streamSimple: streamSimpleAnthropic,
  });
  // ... 更多注册
}
```

### 转发流事件

```typescript
function forwardStream(target: AssistantMessageEventStream, source: AsyncIterable<AssistantMessageEvent>) {
  (async () => {
    for await (const event of source) {
      target.push(event);
    }
    target.end();
  })();
}
```

### 错误处理

```typescript
function createLazyLoadErrorMessage<TApi>(model: Model<TApi>, error: unknown): AssistantMessage {
  return {
    role: "assistant",
    content: [],
    api: model.api,
    provider: model.provider,
    model: model.id,
    usage: { /* empty */ },
    stopReason: "error",
    errorMessage: error instanceof Error ? error.message : String(error),
    timestamp: Date.now(),
  };
}
```

## 事件流协议

### 文件位置

`packages/ai/src/utils/event-stream.ts`

### 事件类型

```typescript
export type AssistantMessageEvent =
  // 开始
  | { type: "start"; partial: AssistantMessage }
  
  // 文本
  | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
  
  // 思考 (reasoning/thinking)
  | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
  
  // 工具调用
  | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  
  // 完成
  | { type: "done"; reason: Extract<StopReason, "stop" | "length" | "toolUse">; message: AssistantMessage }
  | { type: "error"; reason: Extract<StopReason, "aborted" | "error">; error: AssistantMessage };
```

### 事件流实现

```typescript
export class AssistantMessageEventStream {
  private subscribers: ((event: AssistantMessageEvent) => void)[] = [];
  private buffer: AssistantMessageEvent[] = [];
  
  constructor() {
    // 初始化
  }
  
  // 推送事件
  push(event: AssistantMessageEvent): void {
    this.buffer.push(event);
    this.subscribers.forEach(fn => fn(event));
  }
  
  // 订阅事件
  subscribe(fn: (event: AssistantMessageEvent) => void): void {
    this.subscribers.push(fn);
  }
  
  // 获取最终结果
  async result(): Promise<AssistantMessage> {
    // 等待 done 或 error 事件
    return new Promise((resolve) => {
      this.subscribe((event) => {
        if (event.type === "done") {
          resolve(event.message);
        } else if (event.type === "error") {
          resolve(event.error);
        }
      });
    });
  }
  
  // 结束流
  end(message?: AssistantMessage): void {
    // 完成处理
  }
}
```

### 事件流使用示例

```typescript
// 客户端使用
const stream = streamSimple(model, context, options);

stream.subscribe((event) => {
  switch (event.type) {
    case "start":
      // 开始
      break;
    case "text_delta":
      // 新增文本 delta
      process.stdout.write(event.delta);
      break;
    case "thinking_delta":
      // 新增思考 delta
      break;
    case "toolcall_delta":
      // 工具调用参数增量
      break;
    case "done":
      // 完成
      console.log("\n[Done]");
      break;
    case "error":
      // 错误
      console.error("\n[Error]:", event.error.errorMessage);
      break;
  }
});
```

## 环境变量 API Key 解析

### 文件位置

`packages/ai/src/env-api-keys.ts` (~133 行)

### 提供商环境变量映射

```typescript
const envMap: Record<string, string> = {
  // 官方
  openai: "OPENAI_API_KEY",
  "azure-openai-responses": "AZURE_OPENAI_API_KEY",
  google: "GEMINI_API_KEY",
  groq: "GROQ_API_KEY",
  cerebras: "CEREBRAS_API_KEY",
  xai: "XAI_API_KEY",
  openrouter: "OPENROUTER_API_KEY",
  "vercel-ai-gateway": "AI_GATEWAY_API_KEY",
  zai: "ZAI_API_KEY",
  mistral: "MISTRAL_API_KEY",
  minimax: "MINIMAX_API_KEY",
  "minimax-cn": "MINIMAX_CN_API_KEY",
  huggingface: "HF_TOKEN",
  opencode: "OPENCODE_API_KEY",
  "opencode-go": "OPENCODE_API_KEY",
  "kimi-coding": "KIMI_API_KEY",
};

// 特殊处理
if (provider === "github-copilot") {
  return process.env.COPILOT_GITHUB_TOKEN || process.env.GH_TOKEN || process.env.GITHUB_TOKEN;
}

if (provider === "anthropic") {
  return process.env.ANTHROPIC_OAUTH_TOKEN || process.env.ANTHROPIC_API_KEY;
}
```

### OAuth 提供商

某些提供商需要 OAuth 而非简单 API key:

```typescript
// Google Vertex - 支持 ADC
if (provider === "google-vertex") {
  if (process.env.GOOGLE_CLOUD_API_KEY) {
    return process.env.GOOGLE_CLOUD_API_KEY;
  }
  // 检查 ADC 凭证
  const hasCredentials = hasVertexAdcCredentials();
  if (hasCredentials && hasProject && hasLocation) {
    return "<authenticated>";
  }
}

// Amazon Bedrock - 多种认证方式
if (provider === "amazon-bedrock") {
  // AWS_PROFILE
  // AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY
  // AWS_BEARER_TOKEN_BEDROCK
  // AWS_CONTAINER_CREDENTIALS_RELATIVE_URI
  // ... 等
  return "<authenticated>";
}
```

## 模型发现

### 文件位置

`packages/ai/src/models.ts`, `packages/ai/scripts/generate-models.ts`

### 发现来源

1. **models.dev API** - 工具支持模型
2. **OpenRouter API** - 路由模型
3. **Vercel AI Gateway** - 网关模型

### 生成统计

```
Model Statistics:
  Total tool-capable models: 839
  Reasoning-capable models: 535
  
  amazon-bedrock: 91 models
  anthropic: 23 models
  google: 27 models
  openai: 40 models
  groq: 18 models
  cerebras: 4 models
  xai: 24 models
  mistral: 26 models
  huggingface: 20 models
  opencode: 34 models
  github-copilot: 25 models
  openrouter: 248 models
  vercel-ai-gateway: 154 models
```

## 支持的 API 类型详情

### Anthropic

**文件**: `packages/ai/src/providers/anthropic.ts` (~954 行)

```typescript
export interface AnthropicOptions extends StreamOptions {
  // Thinking/Reasoning
  thinkingEnabled?: boolean;
  thinkingBudgetTokens?: number;
  effort?: "low" | "medium" | "high" | "xhigh" | "max";
  thinkingDisplay?: "summarized" | "omitted";
  
  // 工具
  interleavedThinking?: boolean;
  toolChoice?: "auto" | "any" | "none" | { type: "tool"; name: string };
  
  // 客户端
  client?: Anthropic;
}
```

### OpenAI Responses

**文件**: `packages/ai/src/providers/openai-responses.ts` (~264 行)

```typescript
export interface OpenAIResponsesOptions extends StreamOptions {
  reasoningEffort?: "minimal" | "low" | "medium" | "high" | "xhigh";
  reasoningSummary?: "auto" | "detailed" | "concise" | null;
  serviceTier?: ResponseCreateParamsStreaming["service_tier"];
}
```

### OpenAI Completions (兼容模式)

**文件**: `packages/ai/src/providers/openai-completions.ts`

```typescript
export interface OpenAICompletionsOptions extends StreamOptions {
  // 兼容性配置
  compat?: OpenAICompletionsCompat;
}
```

### Google

**文件**: `packages/ai/src/providers/google.ts`

```typescript
export interface GoogleOptions extends StreamOptions {
  // Thinking
  thinkingEnabled?: boolean;
  thinkingBudget?: number;
}
```

### Google Gemini CLI

**文件**: `packages/ai/src/providers/google-gemini-cli.ts`

```typescript
export interface GoogleGeminiCliOptions extends StreamOptions {
  thinkingLevel?: GoogleThinkingLevel;
}
```

### Mistral

**文件**: `packages/ai/src/providers/mistral.ts`

```typescript
export interface MistralOptions extends StreamOptions {
  // Thinking
  thinking?: boolean;
}
```

### Amazon Bedrock

**文件**: `packages/ai/src/providers/amazon-bedrock.ts` (~1029 行)

```typescript
export interface BedrockOptions extends StreamOptions {
  // Thinking
  thinkingEnabled?: boolean;
  thinkingBudgetTokens?: number;
  
  // 显示模式
  thinkingDisplay?: BedrockThinkingDisplay;
}
```

### Azure OpenAI

**文件**: `packages/ai/src/providers/azure-openai-responses.ts`

```typescript
export interface AzureOpenAIResponsesOptions extends StreamOptions {
  // Azure-specific options
}
```

## 总结

pi-ai 提供了：

1. **统一的类型系统** - 标准化 Api, Provider, Model, Message
2. **事件流协议** - 增量更新，支持 thinking, tool calls
3. **提供商注册** - 延迟加载，优化启动时间
4. **环境变量解析** - 统一的 API key 获取
5. **模型发现** - 自动从多个来源发现模型

核��是 `AssistantMessageEventStream` 提供流式事件，`ApiProviderRegistry` 管理提供商实现。