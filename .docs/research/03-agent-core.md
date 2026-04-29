# pi-agent-core 分析

pi-agent-core 是 PI MONO 的 Agent 运行时核心包，负责状态管理、工具执行和事件流处理。

## 一、核心组件

### 1.1 文件结构

```
packages/agent/src/
├── agent.ts        # Agent 类：状态封装和高层 API
├── agent-loop.ts   # 循环逻辑：LLM 调用和工具执行
├── types.ts       # 类型定义
├── proxy.ts      # 代理流函数
└── index.ts     # 导出入口
```

### 1.2 Agent 类

`Agent` 是状态封装类，提供高层 API：

```typescript
// packages/agent/src/agent.ts
export class Agent {
  private _state: MutableAgentState;
  private readonly listeners = new Set<(event: AgentEvent, signal: AbortSignal) => Promise<void>>();
  private readonly steeringQueue: PendingMessageQueue;
  private readonly followUpQueue: PendingMessageQueue;
}
```

**核心职责**：
- 维护 `AgentState`：systemPrompt、model、thinkingLevel、tools、messages
- 管理 `pendingToolCalls`：正在执行的工具调用 ID 集合
- 事件订阅：`subscribe(listener)` 返回取消订阅函数
- 消息队列：`steer()` 和 `followUp()` 用于消息注入

**主要方法**：

| 方法 | 说明 |
|------|------|
| `prompt(message)` | 从文本或消息启动新对话 |
| `continue()` | 从当前上下文继续对话 |
| `steer(message)` | 队列注入消息，在当前 assistant turn 后执行 |
| `followUp(message)` | 队列注入消息，在 agent 停止后执行 |
| `subscribe(listener)` | 订阅生命周期事件 |
| `abort()` | 中止当前运行 |
| `waitForIdle()` | 等待当前运行和事件监听器完成 |
| `reset()` | 重置所有状态和队列 |

### 1.3 AgentLoop 函数

`runAgentLoop` 和 `runAgentLoopContinue` 是低层循环函数：

```typescript
// packages/agent/src/agent-loop.ts
export async function runAgentLoop(
  prompts: AgentMessage[],
  context: AgentContext,
  config: AgentLoopConfig,
  emit: AgentEventSink,
  signal?: AbortSignal,
  streamFn?: StreamFn,
): Promise<AgentMessage[]>
```

**与 Agent 类的区别**：
- Agent 类处理状态序列化和订阅
- AgentLoop 直接操作 context，不维护状态
- AgentLoop 在 LLM 边界处转换 `AgentMessage[]` → `Message[]`

### 1.4 代理流函数

`streamProxy` 用于需要通过中间服务器路由 LLM 调用的场景：

```typescript
// packages/agent/src/proxy.ts
export function streamProxy(
  model: Model<any>,
  context: Context,
  options: ProxyStreamOptions,
): ProxyMessageEventStream
```

服务器端 stripping `partial` 字段减少带宽，客户端重构 partial message。

## 二、状态管理

### 2.1 状态接口

```typescript
// packages/agent/src/types.ts
export interface AgentState {
  systemPrompt: string;
  model: Model<any>;
  thinkingLevel: ThinkingLevel;
  set tools(tools: AgentTool<any>[]);
  get tools(): AgentTool<any>[];
  set messages(messages: AgentMessage[]);
  get messages(): AgentMessage[];
  readonly isStreaming: boolean;
  readonly streamingMessage?: AgentMessage;
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage?: string;
}
```

### 2.2 可变状态

```typescript
// packages/agent/src/agent.ts
type MutableAgentState = Omit<AgentState, "isStreaming" | "streamingMessage" | "pendingToolCalls" | "errorMessage"> & {
  isStreaming: boolean;
  streamingMessage?: AgentMessage;
  pendingToolCalls: Set<string>;
  errorMessage?: string;
};
```

`tools` 和 `messages` 使用 accessor 属性，赋值时复制数组：

```typescript
// packages/agent/src/agent.ts
function createMutableAgentState(
  initialState?: Partial<Omit<AgentState, "pendingToolCalls" | "isStreaming" | "streamingMessage" | "errorMessage">>,
): MutableAgentState {
  let tools = initialState?.tools?.slice() ?? [];
  let messages = initialState?.messages?.slice() ?? [];

  return {
    // ...
    get tools() { return tools; },
    set tools(nextTools: AgentTool<any>[]) { tools = nextTools.slice(); },
    get messages() { return messages; },
    set messages(nextMessages: AgentMessage[]) { messages = nextMessages.slice(); },
    // ...
  };
}
```

### 2.3 消息队列

`PendingMessageQueue` 管理 steering 和 follow-up 消息：

```typescript
// packages/agent/src/agent.ts
class PendingMessageQueue {
  constructor(public mode: QueueMode) {}

  enqueue(message: AgentMessage): void;
  hasItems(): boolean: boolean;
  drain(): AgentMessage[];
  clear(): void;
}

type QueueMode = "all" | "one-at-a-time";
```

- `all`：一次取出所有消息
- `one-at-a-time`：一次取出一个消息

### 2.4 AgentMessage

```typescript
// packages/agent/src/types.ts
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];

export interface CustomAgentMessages {
  // Apps extend via declaration merging
}
```

允许应用通过声明合并添加自定义消息类型。

## 三、工具执行流程

### 3.1 执行模式

```typescript
// packages/agent/src/types.ts
export type ToolExecutionMode = "sequential" | "parallel";
```

| 模式 | 说明 |
|------|------|
| `sequential` | 每个工具调用完成后才执行下一个 |
| `parallel` | 预检查后并发执行，完成顺序发射结果 |

### 3.2 执行 pipeline

```
Assistant Message
    │
    ▼
┌─────────────────────────────────────────┐
│ executeToolCalls()                       │
│   - 过滤 toolCall 内容块                 │
│   - 根据模式选择执行路径                 │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ prepareToolCall()                         │
│   - 查找工具定义                     │
│   - prepareArguments()                 │
│   - validateToolArguments()           │
│   - beforeToolCall hook            │
└─────────────────────────────────────────┘
    │                              │
    ├─ immediate ────────────────►│ emitToolCallOutcome()
    │                            │
    ▼                            │
┌─────────────────────────────────────────┐
│ executePreparedToolCall()              │
│   - 调用 tool.execute()              │
│   - 发射 tool_execution_update       │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ finalizeExecutedToolCall()                │
│   - afterToolCall hook              │
│   - emitToolCallOutcome()             │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ ToolResultMessage                      │
│   - role: "toolResult"               │
│   - toolCallId                     │
│   - toolName                      │
│   - content                      │
│   - isError                     │
└─────────────────────────────────────────┘
```

### 3.3 工具定义

```typescript
// packages/agent/src/types.ts
export interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
  label: string;
  prepareArguments?: (args: unknown) => Static<TParameters>;
  execute: (
    toolCallId: string,
    params: Static<TParameters>,
    signal?: AbortSignal,
    onUpdate?: AgentToolUpdateCallback<TDetails>,
  ) => Promise<AgentToolResult<TDetails>>;
  executionMode?: ToolExecutionMode;
}
```

### 3.4 Hooks

**beforeToolCall**：

```typescript
// packages/agent/src/types.ts
export interface BeforeToolCallResult {
  block?: boolean;
  reason?: string;
}

export interface BeforeToolCallContext {
  assistantMessage: AssistantMessage;
  toolCall: AgentToolCall;
  args: unknown;
  context: AgentContext;
}
```

返回 `{ block: true }` 阻止工具执行，发射错误结果。

**afterToolCall**：

```typescript
// packages/agent/src/types.ts
export interface AfterToolCallResult {
  content?: (TextContent | ImageContent)[];
  details?: unknown;
  isError?: boolean;
}

export interface AfterToolCallContext {
  assistantMessage: AssistantMessage;
  toolCall: AgentToolCall;
  args: unknown;
  result: AgentToolResult<any>;
  isError: boolean;
  context: AgentContext;
}
```

部分覆盖执行结果，按字段合并。

## 四、事件流

### 4.1 事件类型

```typescript
// packages/agent/src/types.ts
export type AgentEvent =
  // Agent lifecycle
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  // Turn lifecycle
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  // Message lifecycle
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  // Tool execution
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

### 4.2 事件序列

```
agent_start
  turn_start
    message_start (user)
    message_end (user)
    [stream assistant response with message_update events]
    message_end (assistant)
    [tool execution events if any]
  turn_end
  [check steering messages → loop if any]
agent_end
```

### 4.3 Agent 类的事件处理

`Agent.processEvents()` 更新内部状态后调用监听器：

```typescript
// packages/agent/src/agent.ts
private async processEvents(event: AgentEvent): Promise<void> {
  switch (event.type) {
    case "message_start":
      this._state.streamingMessage = event.message;
      break;
    case "message_update":
      this._state.streamingMessage = event.message;
      break;
    case "message_end":
      this._state.streamingMessage = undefined;
      this._state.messages.push(event.message);
      break;
    case "tool_execution_start":
      this._state.pendingToolCalls.add(event.toolCallId);
      break;
    case "tool_execution_end":
      this._state.pendingToolCalls.delete(event.toolCallId);
      break;
    // ...
  }

  for (const listener of this.listeners) {
    await listener(event, signal);
  }
}
```

**注意**：`agent_end` 是最后一个发射的事件，但 `waitForIdle()` 在 `agent_end` 监听器完成后才 resolve。

## 五、配置与扩展

### 5.1 AgentOptions

```typescript
// packages/agent/src/agent.ts
export interface AgentOptions {
  initialState?: Partial<AgentState>;
  convertToLlm?: (messages: AgentMessage[]) => Message[];
  transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;
  streamFn?: StreamFn;
  getApiKey?: (provider: string) => string | Promise<string | undefined>;
  onPayload?: SimpleStreamOptions["onPayload"];
  onResponse?: SimpleStreamOptions["onResponse"];
  beforeToolCall?: (context: BeforeToolCallContext, signal?: AbortSignal) => Promise<BeforeToolCallResult>;
  afterToolCall?: (context: AfterToolCallContext, signal?: AbortSignal) => Promise<AfterToolCallResult>;
  steeringMode?: QueueMode;
  followUpMode?: QueueMode;
  sessionId?: string;
  thinkingBudgets?: ThinkingBudgets;
  transport?: Transport;
  maxRetryDelayMs?: number;
  toolExecution?: ToolExecutionMode;
}
```

### 5.2 AgentLoopConfig

```typescript
// packages/agent/src/types.ts
export interface AgentLoopConfig extends SimpleStreamOptions {
  model: Model<any>;
  convertToLlm: (messages: AgentMessage[]) => Message[];
  transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;
  getApiKey?: (provider: string) => string | Promise<string | undefined>;
  getSteeringMessages?: () => Promise<AgentMessage[]>;
  getFollowUpMessages?: () => Promise<AgentMessage[]>;
  toolExecution?: ToolExecutionMode;
  beforeToolCall?: (context: BeforeToolCallContext, signal?: AbortSignal) => Promise<BeforeToolCallResult>;
  afterToolCall?: (context: AfterToolCallContext, signal?: AbortSignal) => Promise<AfterToolCallResult>;
}
```

### 5.3 可扩展点

| 扩展点 | 用途 |
|--------|------|
| `convertToLlm` | AgentMessage → LLM Message 转换 |
| `transformContext` | 上下文变换（消息裁剪、注入） |
| `getApiKey` | 动态获取 API key���OAuth token） |
| `getSteeringMessages` | 运行中注入消息 |
| `getFollowUpMessages` | 停止后注入消息 |
| `beforeToolCall` | 工具执行前检查/阻止 |
| `afterToolCall` | 修改工具执行结果 |

## 六、参考

- 源码：`packages/agent/src/agent.ts`
- 循环逻辑：`packages/agent/src/agent-loop.ts`
- 类型定义：`packages/agent/src/types.ts`
- 代理：`packages/agent/src/proxy.ts`
- 测试：`packages/agent/test/agent.test.ts`、`packages/agent/test/agent-loop.test.ts`