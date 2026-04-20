# pi-agent-core Agent 运行时

## 概述

pi-agent-core 是通用的 Agent 运行时，封装了 LLM 调用和工具执行逻辑。

**核心目标**:
- 消息队列管理 (steering/followUp)
- 流式 LLM 调用
- 工具执行 (sequential/parallel)
- 生命周期事件
- 前后钩子

## 文件结构

```
packages/agent/
├── src/
│   ├── index.ts           # 入口
│   ├── agent.ts           # Agent 类 (~543 行)
│   ├── agent-loop.ts     # Agent 循环 (~639 行)
│   ├── types.ts          # 类型定义 (~349 行)
│   └── proxy.ts          # 代理工具
```

## Agent 类

### 文件位置

`packages/agent/src/agent.ts`

### 类定义

```typescript
export class Agent {
  // 状态
  private _state: MutableAgentState;
  
  // 事件监听
  private readonly listeners = new Set<(event: AgentEvent, signal: AbortSignal) => Promise<void>>();
  
  // 消息队列
  private readonly steeringQueue: PendingMessageQueue;
  private readonly followUpQueue: PendingMessageQueue;
  
  // 配置
  public convertToLlm: (messages: AgentMessage[]) => Message[] | Promise<Message[]>;
  public transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;
  public streamFn: StreamFn;
  public getApiKey?: (provider: string) => string | undefined;
  public onPayload?: SimpleStreamOptions["onPayload"];
  public onResponse?: SimpleStreamOptions["onResponse"];
  public beforeToolCall?: (context: BeforeToolCallContext, signal?: AbortSignal) => Promise<BeforeToolCallResult>;
  public afterToolCall?: (context: AfterToolCallContext, signal?: AbortSignal) => Promise<AfterToolCallResult>;
  
  // 运行状态
  private activeRun?: ActiveRun;
  
  // 配置
  public sessionId?: string;
  public thinkingBudgets?: ThinkingBudgets;
  public transport: Transport;
  public maxRetryDelayMs?: number;
  public toolExecution: ToolExecutionMode;
  
  constructor(options: AgentOptions = {}) { ... }
}
```

### Agent 选项

```typescript
export interface AgentOptions {
  // 初始状态
  initialState?: Partial<Omit<AgentState, "pendingToolCalls" | "isStreaming" | "streamingMessage" | "errorMessage">>;
  
  // 消息转换
  convertToLlm?: (messages: AgentMessage[]) => Message[] | Promise<Message[]>;
  transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;
  
  // LLM 调用
  streamFn?: StreamFn;
  getApiKey?: (provider: string) => string | undefined;
  onPayload?: SimpleStreamOptions["onPayload"];
  onResponse?: SimpleStreamOptions["onResponse"];
  
  // 工具执行钩子
  beforeToolCall?: (context: BeforeToolCallContext, signal?: AbortSignal) => Promise<BeforeToolCallResult>;
  afterToolCall?: (context: AfterToolCallContext, signal?: AbortSignal) => Promise<AfterToolCallResult>;
  
  // 队列模式
  steeringMode?: QueueMode;  // "all" | "one-at-a-time"
  followUpMode?: QueueMode;
  
  // 配置
  sessionId?: string;
  thinkingBudgets?: ThinkingBudgets;
  transport?: Transport;
  maxRetryDelayMs?: number;
  toolExecution?: ToolExecutionMode;
}
```

### Agent 状态

```typescript
export interface AgentState {
  // 基本状态
  systemPrompt?: string;
  model?: Model<any>;
  thinkingLevel?: ThinkingLevel;
  
  // 消息和工具
  messages: AgentMessage[];
  tools: AgentTool<any>[];
  
  // 流式状态
  isStreaming: boolean;
  streamingMessage?: AgentMessage;
  
  // 工具调用状态
  pendingToolCalls: Set<string>;
  
  // 错误
  errorMessage?: string;
}
```

## 消息队列

### 文件位置

`packages/agent/src/agent.ts` - `PendingMessageQueue` 类

### 队列模式

```typescript
type QueueMode = "all" | "one-at-a-time";

class PendingMessageQueue {
  private messages: AgentMessage[] = [];
  
  constructor(public mode: QueueMode) {}
  
  enqueue(message: AgentMessage): void {
    this.messages.push(message);
  }
  
  hasItems(): boolean {
    return this.messages.length > 0;
  }
  
  drain(): AgentMessage[] {
    if (this.mode === "all") {
      // 取出所有消息
      const drained = this.messages.slice();
      this.messages = [];
      return drained;
    }
    
    // 取出一个消息
    const first = this.messages[0];
    if (!first) return [];
    this.messages = this.messages.slice(1);
    return [first];
  }
  
  clear(): void {
    this.messages = [];
  }
}
```

### 队列用途

1. **Steering Queue** - 注入正在进行的对话
   - 在 AI 响应之前插入消息
   - 用例: 用户中断、优先级调整

2. **Follow-up Queue** - 在 AI 停止后处理
   - 在 AI 停止后执行
   - 用例: 自动后续问题

## Agent 公共 API

### 订阅事件

```typescript
subscribe(listener: (event: AgentEvent, signal: AbortSignal) => Promise<void> | void): () => void {
  this.listeners.add(listener);
  return () => this.listeners.delete(listener);
}
```

### 获取/设置状态

```typescript
get state(): AgentState {
  return this._state;
}
```

### 队列操作

```typescript
// 注入消息 (在当前 turn 后执行)
steer(message: AgentMessage): void {
  this.steeringQueue.enqueue(message);
}

// 队列消息 (在 AI 停止后执行)
followUp(message: AgentMessage): void {
  this.followUpQueue.enqueue(message);
}

// 清空队列
clearSteeringQueue(): void;
clearFollowUpQueue(): void;
clearAllQueues(): void;

// 检查队列
hasQueuedMessages(): boolean;
```

### 主方法

```typescript
// 启动新提示
async prompt(message: AgentMessage | AgentMessage[]): Promise<void>;
async prompt(input: string, images?: ImageContent[]): Promise<void>;

// 继续当前会话
async continue(): Promise<void>;

// 中止
abort(): void;

// 等待空闲
waitForIdle(): Promise<void>;

// 重置
reset(): void;
```

### 使用示例

```typescript
const agent = new Agent({
  streamFn: streamSimple,
  convertToLlm: (messages) => messages.filter(
    (m) => m.role === "user" || m.role === "assistant" || m.role === "toolResult"
  ),
});

// 订阅事件
agent.subscribe((event, signal) => {
  if (event.type === "message_start") {
    console.log("Message:", event.message.role);
  }
  if (event.type === "toolcall_end") {
    console.log("Tool result:", event.result);
  }
});

// 启动提示
await agent.prompt("Hello, how are you?");

// 使用 steering 注入消息
agent.steer({ role: "user", content: "Actually, also tell me the weather." });

// 使用 followup 添加后续
agent.followUp({ role: "user", content: "Thanks! Here's a picture of my cat." });

// 继续
await agent.continue();

// 等待完成
await agent.waitForIdle();
```

## Agent 循环

### 文件位置

`packages/agent/src/agent-loop.ts` (~639 行)

### 循环函数

```typescript
// 从新提示开始
export function agentLoop(
  prompts: AgentMessage[],
  context: AgentContext,
  config: AgentLoopConfig,
  signal?: AbortSignal,
  streamFn?: StreamFn,
): EventStream<AgentEvent, AgentMessage[]>;

// 继续当前会话
export function agentLoopContinue(
  context: AgentContext,
  config: AgentLoopConfig,
  signal?: AbortSignal,
  streamFn?: StreamFn,
): EventStream<AgentEvent, AgentMessage[]>;

// 内部运行
export async function runAgentLoop(
  prompts: AgentMessage[],
  context: AgentContext,
  config: AgentLoopConfig,
  emit: AgentEventSink,
  signal?: AbortSignal,
  streamFn?: StreamFn,
): Promise<AgentMessage[]>;
```

### 主循环实现

```typescript
async function runLoop(
  currentContext: AgentContext,
  newMessages: AgentMessage[],
  config: AgentLoopConfig,
  signal: AbortSignal | undefined,
  emit: AgentEventSink,
  streamFn?: StreamFn,
): Promise<void> {
  let firstTurn = true;
  
  // 检查 steering 消息
  let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) || [];
  
  // 外层循环: 当有 follow-up 消息时继续
  while (true) {
    let hasMoreToolCalls = true;
    
    // 内层循环: 处理工具调用和 steering 消息
    while (hasMoreToolCalls || pendingMessages.length > 0) {
      if (!firstTurn) {
        await emit({ type: "turn_start" });
      } else {
        firstTurn = false;
      }
      
      // 处理 pending messages
      if (pendingMessages.length > 0) {
        for (const message of pendingMessages) {
          await emit({ type: "message_start", message });
          await emit({ type: "message_end", message });
          currentContext.messages.push(message);
          newMessages.push(message);
        }
        pendingMessages = [];
      }
      
      // 流式获取助手响应
      const message = await streamAssistantResponse(
        currentContext, config, signal, emit, streamFn
      );
      newMessages.push(message);
      
      // 检查停止原因
      if (message.stopReason === "error" || message.stopReason === "aborted") {
        await emit({ type: "turn_end", message, toolResults: [] });
        await emit({ type: "agent_end", messages: newMessages });
        return;
      }
      
      // 执行工具调用
      if (message.content.some(c => c.type === "toolCall")) {
        const toolResults = await executeToolCalls(message, currentContext, config, signal);
        
        // 添加工具结果到上下文
        for (const result of toolResults) {
          currentContext.messages.push(result);
          newMessages.push(result);
        }
        
        await emit({
          type: "turn_end",
          message,
          toolResults,
        });
      } else {
        // 没有工具调用，停止
        hasMoreToolCalls = false;
      }
    }
    
    // 获取 follow-up 消息
    const followUps = await config.getFollowUpMessages?.() || [];
    if (followUps.length > 0) {
      // 添加入上下文并继续
      for (const message of followUps) {
        currentContext.messages.push(message);
        newMessages.push(message);
      }
      continue; // 继续外层循环
    }
    
    // 没有更多消息，退出
    await emit({ type: "agent_end", messages: newMessages });
    return;
  }
}
```

### 获取助手响应

```typescript
async function streamAssistantResponse(
  currentContext: AgentContext,
  config: AgentLoopConfig,
  signal: AbortSignal | undefined,
  emit: AgentEventSink,
  streamFn?: StreamFn,
): Promise<AssistantMessage> {
  // 1. 应用 transformContext
  let context = currentContext;
  if (config.transformContext) {
    const transformed = await config.transformContext(currentContext.messages, signal);
    context = { ...currentContext, messages: transformed };
  }
  
  // 2. 应用 convertToLlm
  const llmMessages = await config.convertToLlm(context.messages);
  const llmContext: Context = {
    systemPrompt: context.systemPrompt,
    messages: llmMessages,
    tools: context.tools,
  };
  
  // 3. 调用 LLM
  const stream = streamFn!(context.model, llmContext, {
    signal,
    sessionId: config.sessionId,
    thinkingBudgets: config.thinkingBudgets,
    transport: config.transport,
    maxRetryDelayMs: config.maxRetryDelayMs,
  });
  
  // 4. 处理流事件 (略)
  // ...
  
  // 返回最终消息
  return finalMessage;
}
```

## 工具执行

### 执行模式

```typescript
export type ToolExecutionMode = "sequential" | "parallel";
```

- **Sequential**: 顺序执行
- **Parallel**: 预检顺序执行，然后并发执行

### 执行模式区别

```typescript
// Sequential - 一个一个执行
for (const toolCall of toolCalls) {
  const result = await executeTool(toolCall);
  results.push(result);
}

// Parallel - 预检后并发
const preflightResults = [];
for (const toolCall of toolCalls) {
  const preflight = await preflightTool(toolCall);
  preflightResults.push(preflight);
}

// 并发执行
const executionPromises = toolCalls.map((toolCall, i) => {
  if (!preflightResults[i].blocked) {
    return executeTool(toolCall);
  }
  return createBlockedResult(toolCall, preflightResults[i].reason);
});

results = await Promise.all(executionPromises);
```

## 工具执行钩子

### Before 钩子

```typescript
interface BeforeToolCallContext {
  assistantMessage: AssistantMessage;  // 请求工具调用的助手消息
  toolCall: AgentToolCall;              // 工具调用块
  args: unknown;                    // 验证后的参数
  context: AgentContext;            // 当前上下文
}

interface BeforeToolCallResult {
  block?: boolean;   // 是否阻止执行
  reason?: string;   // 阻止原因 (可选)
}

// 使用
beforeToolCall: async (context, signal) => {
  // 检查权限
  const hasPermission = await checkPermission(context.toolCall.name);
  if (!hasPermission) {
    return { block: true, reason: "Not authorized" };
  }
}
```

### After 钩子

```typescript
interface AfterToolCallContext {
  assistantMessage: AssistantMessage;
  toolCall: AgentToolCall;
  args: unknown;
  result: AgentToolResult<any>;   // 执行结果
  isError: boolean;
  context: AgentContext;
}

interface AfterToolCallResult {
  content?: (TextContent | ImageContent)[];  // 替换结果内容
  details?: unknown;                         // 替换 details
  isError?: boolean;                       // 替换错误标志
}

// 使用
afterToolCall: async (context, signal) => {
  // 添加日志
  await logToolExecution(context.toolCall.name, context.result);
  
  // 替换错误结果
  if (context.isError && context.result.content?.[0]?.type === "text") {
    return {
      content: [{
        type: "text",
        text: sanitize(context.result.content[0].text),
      }],
    };
  }
}
```

## 事件协议

### 文件位置

`packages/agent/src/types.ts`

### 事件类型

```typescript
export type AgentEvent =
  // Agent 级别
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  
  // Turn 级别
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: AgentToolResult[] }
  
  // Message 级别
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_end"; message: AgentMessage }
  
  // Tool Call 级别
  | { type: "toolcall_start"; toolCall: AgentToolCall; tool: AgentTool }
  | { type: "toolcall_delta"; toolCall: AgentToolCall; delta: string }
  | { type: "toolcall_end"; toolCall: AgentToolCall; result: AgentToolResult }
  | { type: "toolcall_error"; toolCall: AgentToolCall; error: Error };
```

### 事件流使用

```typescript
agent.subscribe(async (event, signal) => {
  switch (event.type) {
    case "agent_start":
      console.log("Agent started");
      break;
      
    case "turn_start":
      console.log("Turn started");
      break;
      
    case "message_start":
      console.log("Message:", event.message.role);
      break;
      
    case "toolcall_start":
      console.log("Tool call:", event.toolCall.name);
      break;
      
    case "toolcall_delta":
      // 增量参数
      break;
      
    case "toolcall_end":
      console.log("Tool result:", event.result);
      break;
      
    case "turn_end":
      console.log("Turn ended. Tool results:", event.toolResults.length);
      break;
      
    case "agent_end":
      console.log("Agent ended. Total messages:", event.messages.length);
      break;
  }
});
```

## Agent 上下文

```typescript
export interface AgentContext {
  // 系统提示
  systemPrompt?: string;
  
  // 消息历史
  messages: AgentMessage[];
  
  // 可用工具
  tools?: AgentTool<any>[];
  
  // 当前模型
  model: Model<any>;
  
  // thinking 级别
  thinkingLevel?: ThinkingLevel;
}
```

## 消息类型

```typescript
// AgentMessage 是扩展的消息类型
export interface AgentMessage {
  role: "user" | "assistant" | "toolResult" | "custom";
  content: any;
  timestamp: number;
  
  // 扩展字段
  images?: ImageContent[];
  toolCallId?: string;
  toolName?: string;
}
```

## 配置接口

```typescript
export interface AgentLoopConfig extends SimpleStreamOptions {
  // 模型
  model: Model<any>;
  
  // 消息转换
  convertToLlm: (messages: AgentMessage[]) => Message[];
  transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;
  
  // API Key
  getApiKey?: (provider: string) => string | undefined;
  
  // 消息队列
  getSteeringMessages?: () => Promise<AgentMessage[]>;
  getFollowUpMessages?: () => Promise<AgentMessage[]>;
  
  // 工具执行
  toolExecution?: ToolExecutionMode;
  beforeToolCall?: (context: BeforeToolCallContext, signal?: AbortSignal) => Promise<BeforeToolCallResult>;
  afterToolCall?: (context: AfterToolCallContext, signal?: AbortSignal) => Promise<AfterToolCallResult>;
  
  // 更多配置...
}
```

## 总结

pi-agent-core 核心设计：

1. **Agent 类** - 状态管理和公共 API
2. **消息队列** - steering/followUp 模式
3. **Agent 循环** - 处理 LLM 调用和工具执行
4. **工具执行钩子** - before/after 拦截
5. **事件协议** - 生命周期事件

关键特性：
- 流式响应处理
- Sequential/Parallel 工具执行模式
- 前后钩子拦截
- 完整的生命周期事件