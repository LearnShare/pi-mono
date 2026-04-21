# pi-ai 类型系统与 LLM 提供商分析

## 一、核心类型与数据结构

### 1.1 消息协议类型

pi-ai 定义了统一的消息协议类型体系，核心定义在 `packages/ai/src/types.ts`:

```typescript
// 用户消息
interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;
}

// 助手消息
interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: Api;
  provider: Provider;
  model: string;
  responseId?: string;
  usage: Usage;
  stopReason: StopReason;
  errorMessage?: string;
  timestamp: number;
}

// 工具结果消息
interface ToolResultMessage<TDetails = any> {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  details?: TDetails;
  isError: boolean;
  timestamp: number;
}
```

**内容类型**：

```typescript
interface TextContent {
  type: "text";
  text: string;
  textSignature?: string;
}

interface ThinkingContent {
  type: "thinking";
  thinking: string;
  thinkingSignature?: string;
  redacted?: boolean;
}

interface ImageContent {
  type: "image";
  data: string;  // base64
  mimeType: string;
}

interface ToolCall {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, any>;
  thoughtSignature?: string;
}
```

### 1.2 事件流协议

事件流采用发布-订阅模式，定义在 `packages/ai/src/utils/event-stream.ts`:

```typescript
type AssistantMessageEvent =
  | { type: "start"; partial: AssistantMessage }
  | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  | { type: "done"; reason: Extract<StopReason, "stop" | "length" | "toolUse">; message: AssistantMessage }
  | { type: "error"; reason: Extract<StopReason, "aborted" | "error">; error: AssistantMessage };
```

事件流协议特点：
- 支持增量更新（delta 事件）
- 支持 thinking/reasoning 独立流
- 支持工具调用增量构建
- 错误通过事件传播，不抛出异常

### 1.3 流选项定义

```typescript
interface StreamOptions {
  temperature?: number;
  maxTokens?: number;
  signal?: AbortSignal;
  apiKey?: string;
  transport?: "sse" | "websocket" | "auto";
  cacheRetention?: "none" | "short" | "long";
  sessionId?: string;
  onPayload?: (payload: unknown, model: Model<Api>) => unknown | undefined;
  onResponse?: (response: ProviderResponse, model: Model<Api>) => void;
  headers?: Record<string, string>;
  maxRetryDelayMs?: number;
  metadata?: Record<string, unknown>;
}

interface SimpleStreamOptions extends StreamOptions {
  reasoning?: ThinkingLevel;  // "minimal" | "low" | "medium" | "high" | "xhigh"
  thinkingBudgets?: ThinkingBudgets;
}
```

### 1.4 工具定义

```typescript
interface Tool<TParameters extends TSchema = TSchema> {
  name: string;
  description: string;
  parameters: TParameters;
}

interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}
```

## 二、LLM 提供商支持

### 2.1 支持的提供商（24 个）

| 提供商 | API 类型 | 核心模型 |
|--------|---------|----------|
| openai | openai-completions / openai-responses | GPT-5, GPT-4o |
| azure-openai-responses | azure-openai-responses | GPT-4o |
| openai-codex | openai-codex-responses | Codex |
| anthropic | anthropic-messages | Claude 4.6/4.7 |
| amazon-bedrock | bedrock-converse-stream | Nova, Claude |
| google | google-generative-ai | Gemini 2.0 |
| google-gemini-cli | google-gemini-cli | Gemini CLI |
| google-vertex | google-vertex | Gemini on Vertex |
| mistral | mistral-conversations | Mistral Large |
| xai | openai-completions | Grok |
| groq | openai-completions | Llama |
| cerebras | openai-completions | Llama |
| openrouter | openai-completions | 多提供商路由 |
| vercel-ai-gateway | openai-completions | 多提供商路由 |
| zai | openai-completions | Za |
| minimax | openai-completions | MiniMax |
| minimax-cn | openai-completions | MiniMax CN |
| huggingface | openai-completions | 开源模型 |
| opencode | openai-completions | OpenCode |
| opencode-go | openai-completions | OpenCode Go |
| kimi-coding | openai-completions | Kimi |
| github-copilot | openai-completions | GPT-4o |
| google-antigravity | openai-completions | Gemini |

### 2.2 API 标识符

```typescript
type KnownApi =
  | "openai-completions"
  | "mistral-conversations"
  | "openai-responses"
  | "azure-openai-responses"
  | "openai-codex-responses"
  | "anthropic-messages"
  | "bedrock-converse-stream"
  | "google-generative-ai"
  | "google-gemini-cli"
  | "google-vertex";
```

## 三、提供商抽象实现

### 3.1 注册机制

提供商通过 `api-registry.ts` 注册：

```typescript
// packages/ai/src/api-registry.ts
interface ApiProvider {
  api: Api;
  stream: StreamFunction;
  streamSimple: StreamFunction;
}

let apiProviders: Map<Api, ApiProvider> = new Map();

export function registerApiProvider(provider: ApiProvider): void
export function getApiProvider(api: Api): ApiProvider | undefined
export function clearApiProviders(): void
```

### 3.2 内置提供商注册

`packages/ai/src/providers/register-builtins.ts` 实现了内置提供商的懒加载：

```typescript
// 懒加载模式 - 按需导入提供商模块
function loadAnthropicProviderModule(): Promise<...> {
  anthropicProviderModulePromise ||= import("./anthropic.js").then(...);
  return anthropicProviderModulePromise;
}

// 创建延迟流函数
function createLazyStream<TApi>(loadModule: () => Promise<...>): StreamFunction<TApi> {
  return (model, context, options) => {
    const outer = new AssistantMessageEventStream();
    loadModule()
      .then((module) => {
        const inner = module.stream(model, context, options);
        forwardStream(outer, inner);
      })
      .catch((error) => {
        // 错误通过事件流传播
        outer.push({ type: "error", reason: "error", error: message });
        outer.end(message);
      });
    return outer;
  };
}

// 注册所有内置提供商
export function registerBuiltInApiProviders(): void {
  registerApiProvider({ api: "anthropic-messages", stream: streamAnthropic, streamSimple: ... });
  registerApiProvider({ api: "openai-completions", ... });
  // ... 更多提供商
}
```

### 3.3 流调用入口

`packages/ai/src/stream.ts` 提供统一的流调用接口：

```typescript
import "./providers/register-builtins.js";  // 初始化时注册所有提供商

export function stream<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: ProviderStreamOptions,
): AssistantMessageEventStream

export async function complete<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: ProviderStreamOptions,
): Promise<AssistantMessage>

export function streamSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions,
): AssistantMessageEventStream

export async function completeSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions,
): Promise<AssistantMessage>
```

## 四、模型系统

### 4.1 模型元数据

```typescript
interface Model<TApi extends Api> {
  id: string;
  name: string;
  api: TApi;
  provider: Provider;
  baseUrl: string;
  reasoning: boolean;
  input: ("text" | "image")[];
  cost: {
    input: number;      // $/million tokens
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };
  contextWindow: number;
  maxTokens: number;
  headers?: Record<string, string>;
  compat?: OpenAICompletionsCompat | OpenAIResponsesCompat;
}
```

### 4.2 模型注册与发现

`packages/ai/src/models.ts` 提供模型查询接口：

```typescript
export function getModel<TProvider, TModelId>(
  provider: TProvider,
  modelId: TModelId,
): Model<...>

export function getProviders(): KnownProvider[]

export function getModels<TProvider>(provider: TProvider): Model<...>[]

export function calculateCost<TApi>(model: Model<TApi>, usage: Usage): Usage["cost"]

export function supportsXhigh<TApi>(model: Model<TApi>): boolean
```

### 4.3 兼容性配置

针对 OpenAI 兼容 API 的配置：

```typescript
interface OpenAICompletionsCompat {
  supportsStore?: boolean;
  supportsDeveloperRole?: boolean;
  supportsReasoningEffort?: boolean;
  reasoningEffortMap?: Partial<Record<ThinkingLevel, string>>;
  supportsUsageInStreaming?: boolean;
  maxTokensField?: "max_completion_tokens" | "max_tokens";
  requiresToolResultName?: boolean;
  requiresAssistantAfterToolResult?: boolean;
  requiresThinkingAsText?: boolean;
  thinkingFormat?: "openai" | "openrouter" | "zai" | "qwen" | "qwen-chat-template";
  openRouterRouting?: OpenRouterRouting;
  vercelGatewayRouting?: VercelGatewayRouting;
  zaiToolStream?: boolean;
  supportsStrictMode?: boolean;
  cacheControlFormat?: "anthropic";
  sendSessionAffinityHeaders?: boolean;
}
```

## 五、环境变量与凭证

`packages/ai/src/env-api-keys.ts` 统一管理 API 密钥：

```typescript
// 支持的凭证环境变量
const API_KEY_VARS: Record<KnownProvider, string> = {
  "anthropic": "ANTHROPIC_API_KEY",
  "openai": "OPENAI_API_KEY",
  "azure-openai-responses": "AZURE_OPENAI_API_KEY",
  "google": "GOOGLE_API_KEY",
  "mistral": "MISTRAL_API_KEY",
  // ... 更多
};

export function getEnvApiKey(provider: KnownProvider): string | undefined
```

## 六、代码示例

### 6.1 基本流调用

```typescript
import { getModel, stream, completeSimple } from "@anthropic/pi-ai";

const model = getModel("anthropic", "claude-sonnet-4-6");
const context = {
  messages: [{
    role: "user",
    content: "Hello",
    timestamp: Date.now(),
  }],
};

const events = stream(model, context);
for await (const event of events) {
  console.log(event);
}

const result = await completeSimple(model, context);
console.log(result.content);
```

### 6.2 带工具调用

```typescript
import { Type } from "@sinclair/typebox";
import { getModel, completeSimple } from "@anthropic/pi-ai";

const model = getModel("anthropic", "claude-sonnet-4-6");
const tool = {
  name: "calculator",
  description: "Calculate a math expression",
  parameters: Type.Object({
    expression: Type.String(),
  }),
};

const context = {
  messages: [...],
  tools: [tool],
};

const result = await completeSimple(model, context);
// result.content 包含 ToolCall
```

### 6.3 自定义 reasoning 级别

```typescript
import { getModel, completeSimple, ThinkingLevel } from "@anthropic/pi-ai";

const model = getModel("openai", "gpt-5.3");
const result = await completeSimple(model, context, {
  reasoning: "high",  // minimal, low, medium, high, xhigh
});
```

## 七、参考源码

| 模块 | 路径 | 说明 |
|------|------|------|
| 核心类型 | `packages/ai/src/types.ts` | 消息、事件、选项类型定义 |
| 事件流 | `packages/ai/src/utils/event-stream.ts` | AssistantMessageEventStream 实现在此 |
| API 注册 | `packages/ai/src/api-registry.ts` | 提供商注册表 |
| 内置提供商 | `packages/ai/src/providers/register-builtins.ts` | 24 提供商懒加载注册 |
| 模型系统 | `packages/ai/src/models.ts` | 模型查询与定价计算 |
| 模型定义 | `packages/ai/src/models.generated.ts` | 自动生成的模型元数据 |
| 流调用入口 | `packages/ai/src/stream.ts` | stream/complete 函数 |
| 凭证管理 | `packages/ai/src/env-api-keys.ts` | 环境变量读取 |
| Anthropic 提供商 | `packages/ai/src/providers/anthropic.ts` | Claude API 实现 |
| OpenAI 提供商 | `packages/ai/src/providers/openai-completions.ts` | GPT API 实现 |
| Bedrock 提供商 | `packages/ai/src/providers/amazon-bedrock.ts` | AWS Bedrock 实现 |
| Google 提供商 | `packages/ai/src/providers/google.ts` | Gemini API 实现 |
| Azure 提供商 | `packages/ai/src/providers/azure-openai-responses.ts` | Azure OpenAI 实现 |
| 消息转换 | `packages/ai/src/providers/transform-messages.ts` | 消息格式转换 |