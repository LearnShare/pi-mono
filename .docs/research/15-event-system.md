# pi-coding-agent 事件系统研究报告

## 执行摘要

本报告分析了 pi-coding-agent 的事件系统架构，涵盖底层 Agent 事件和上层扩展事件两个层次。底层事件系统定义在 `packages/agent` 中，由 `Agent` 类通过订阅者模式发射；上层扩展系统定义在 `packages/coding-agent` 中，通过 `ExtensionRunner` 实现更丰富的事件类型。事件系统支持会话生命周期、Turn 生命周期、工具执行、用户输入等多种事件的订阅与处理，扩展可以通过 API 订阅这些事件来实现对 Agent 行为的拦截和修改。

---

## 一、事件类型完整列表

### 1.1 会话事件 (Session Events)

位置: `packages/coding-agent/src/core/extensions/types.ts` (行 507-596)

| 事件类型 | 说明 | 可取消 |
|---------|------|-------|
| `session_start` | 会话启动、加载或重新加载 | 否 |
| `session_before_switch` | 切换会话之前 | 是 |
| `session_before_fork` | 派生会话之前 | 是 |
| `session_before_compact` | 上下文压缩之前 | 是 |
| `session_compact` | 上下文压缩完成后 | 否 |
| `session_shutdown` | 会话关闭（退出、重载、会话替换） | 否 |
| `session_before_tree` | 树形导航之前 | 是 |
| `session_tree` | 树形导航完成后 | 否 |

### 1.2 Turn 事件

位置: `packages/agent/src/types.ts` (行 350-365)
位置: `packages/coding-agent/src/core/extensions/types.ts` (行 645-658)

| 事件类型 | 说明 | 携带数据 |
|---------|------|---------|
| `turn_start` | 新 Turn 开始 | `turnIndex`, `timestamp` |
| `turn_end` | Turn 结束 | `turnIndex`, `message`, `toolResults` |

### 1.3 消息事件

位置: `packages/agent/src/types.ts` (行 350-365)

| 事件类型 | 说明 | 携带数据 |
|---------|------|---------|
| `message_start` | 消息开始 | `message` |
| `message_update` | 消息更新（流式） | `message`, `assistantMessageEvent` |
| `message_end` | 消息结束 | `message` |

### 1.4 工具事件

位置: `packages/coding-agent/src/core/extensions/types.ts` (行 759-880)

| 事件类型 | 说明 | 可取消 |
|---------|------|-------|
| `tool_call` | 工具调用之前 | 是 |
| `tool_result` | 工具结果返回后 | 否（可修改） |

具体工具事件:
- `BashToolCallEvent` / `BashToolResultEvent`
- `ReadToolCallEvent` / `ReadToolResultEvent`
- `EditToolCallEvent` / `EditToolResultEvent`
- `WriteToolCallEvent` / `WriteToolResultEvent`
- `GrepToolCallEvent` / `GrepToolResultEvent`
- `FindToolCallEvent` / `FindToolResultEvent`
- `LsToolCallEvent` / `LsToolResultEvent`
- `CustomToolCallEvent` / `CustomToolResultEvent`

### 1.5 Agent 生命周期事件

位置: `packages/agent/src/types.ts` (行 350-365)
位置: `packages/coding-agent/src/core/extensions/types.ts` (行 602-703)

| 事件类型 | 说明 |
|---------|------|
| `agent_start` | Agent 运行开始 |
| `agent_end` | Agent 运行结束 |
| `before_agent_start` | 用户提交提示后、Agent 循环开始前 |
| `context` | 每次 LLM 调用前的消息上下 |
| `before_provider_request` | 提供商请求发送前 |
| `after_provider_response` | 提供商响应接收后 |

### 1.6 工具执行事件

位置: `packages/agent/src/types.ts` (行 363-365)

| 事件类型 | 说明 |
|---------|------|
| `tool_execution_start` | 工具执行开始 |
| `tool_execution_update` | 工具执行更新（流式） |
| `tool_execution_end` | 工具执行结束 |

### 1.7 用户输入事件

位置: `packages/coding-agent/src/core/extensions/types.ts` (行 721-756)

| 事件类型 | 说明 | 可取消 |
|---------|------|-------|
| `input` | 用户输入接收后 | 是 |
| `user_bash` | 用户执行 bash 命令（! 前缀） | 是 |

### 1.8 模型事件

位置: `packages/coding-agent/src/core/extensions/types.ts` (行 708-717)

| 事件类型 | 说明 |
|---------|------|
| `model_select` | 模型选择变更 |

### 1.9 资源发现事件

位置: `packages/coding-agent/src/core/extensions/types.ts` (行 492-504)

| 事件类型 | 说明 |
|---------|------|
| `resources_discover` | 资源发现（启动、重载时） |

### 1.10 错误事件

位置: `packages/coding-agent/src/core/extensions/runner.ts` (行 1540-1545)

| 事件类型 | 说明 |
|---------|------|
| `ExtensionError` | 扩展处理错误（通过错误监听器） |

---

## 二、事件传播机制

### 2.1 底层 Agent 事件系统

位置: `packages/agent/src/agent.ts` (行 158-222, 495-542)

#### 2.1.1 订阅 API

```typescript
// 行 219-222
subscribe(listener: (event: AgentEvent, signal: AbortSignal) => Promise<void> | void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
}
```

- 使用 `Set` 存储监听器，保证顺序且无重复
- 返回取消订阅函数
- 监听器可以是同步或 async 函数

#### 2.1.2 事件发射流程

位置: `packages/agent/src/agent.ts` (行 495-542)

```typescript
private async processEvents(event: AgentEvent): Promise<void> {
    // 1. 先更新内部状态
    switch (event.type) {
        case "message_start":
            this._state.streamingMessage = event.message;
            break;
        // ... 其他状态更新
    }

    // 2. 获取当前 abort signal
    const signal = this.activeRun?.abortController.signal;
    if (!signal) {
        throw new Error("Agent listener invoked outside active run");
    }

    // 3. 按顺序 await 每个监听器
    for (const listener of this.listeners) {
        await listener(event, signal);
    }
}
```

关键特性:
- 状态更新先于监听器执行
- 监听器按订阅顺序串行执行（await）
- 传入 abort signal 用于取消

### 2.2 上层扩展事件系统

位置: `packages/coding-agent/src/core/extensions/runner.ts` (行 676-1021)

#### 2.2.1 Generic emit()

位置: `packages/coding-agent/src/core/extensions/runner.ts` (行 676-708)

```typescript
async emit<TEvent extends RunnerEmitEvent>(event: TEvent): Promise<RunnerEmitResult<TEvent>> {
    const ctx = this.createContext();
    let result: SessionBeforeEventResult | undefined;

    for (const ext of this.extensions) {
        const handlers = ext.handlers.get(event.type);
        if (!handlers || handlers.length === 0) continue;

        for (const handler of handlers) {
            try {
                const handlerResult = await handler(event, ctx);

                // 会话前事件支持取消
                if (this.isSessionBeforeEvent(event) && handlerResult) {
                    result = handlerResult as SessionBeforeEventResult;
                    if (result.cancel) {
                        return result as RunnerEmitResult<TEvent>;
                    }
                }
            } catch (err) {
                this.emitError({ extensionPath: ext.path, event: event.type, error: message, stack });
            }
        }
    }
    return result as RunnerEmitResult<TEvent>;
}
```

#### 2.2.2 专用事件方法

特殊事件使用专用方法处理不同的返回值逻辑:

| 方法 | 行号 | 特殊处理 |
|-----|------|---------|
| `emitToolCall` | 760-781 | 可阻止工具执行 |
| `emitToolResult` | 710-758 | 可修改结果内容 |
| `emitUserBash` | 783-810 | 可自定义执行 |
| `emitContext` | 812-842 | 可修改消息 |
| `emitBeforeProviderRequest` | 844-876 | 可替换 payload |
| `emitBeforeAgentStart` | 878-942 | 可添加消息/修改系统提示 |
| `emitResourcesDiscover` | 944-990 | 收集资源路径 |
| `emitInput` | 993-1021 | 输入转换链 |

#### 2.2.3 错误��理

位置: `packages/coding-agent/src/core/extensions/runner.ts` (行 477-486)

```typescript
onError(listener: ExtensionErrorListener): () => void {
    this.errorListeners.add(listener);
    return () => this.errorListeners.delete(listener);
}

emitError(error: ExtensionError): void {
    for (const listener of this.errorListeners) {
        listener(error);
    }
}
```

---

## 三、扩展如何订阅事件

### 3.1 订阅 API

位置: `packages/coding-agent/src/core/extensions/types.ts` (行 1062-1110)

通过 `ExtensionAPI.on()` 方法:

```typescript
interface ExtensionAPI {
    // 事件订阅
    on(event: "session_start", handler: ExtensionHandler<SessionStartEvent>): void;
    on(event: "session_before_switch", handler: ExtensionHandler<SessionBeforeSwitchEvent, SessionBeforeSwitchResult>): void;
    on(event: "agent_start", handler: ExtensionHandler<AgentStartEvent>): void;
    on(event: "turn_start", handler: ExtensionHandler<TurnStartEvent>): void;
    on(event: "tool_call", handler: ExtensionHandler<ToolCallEvent, ToolCallEventResult>): void;
    on(event: "tool_result", handler: ExtensionHandler<ToolResultEvent, ToolResultEventResult>): void;
    // ... 更多事件
}
```

### 3.2 处理器类型

位置: `packages/coding-agent/src/core/extensions/types.ts` (行 1062-1064)

```typescript
export type ExtensionHandler<E, R = undefined> = (
    event: E,
    ctx: ExtensionContext
) => Promise<R | void> | R | void;
```

### 3.3 使用示例

位置: 源码中没有直接示例，以下为基于 API 的典型用法:

```typescript
import { ExtensionAPI, SessionStartEvent, ToolCallEvent, ToolCallEventResult } from "@mariozechner/pi-coding-agent";

export function createExtension(api: ExtensionAPI) {
    // 订阅会话启动事件
    api.on("session_start", async (event: SessionStartEvent) => {
        console.log("Session started:", event.reason);
    });

    // 订阅工具调用事件，可阻止执行
    api.on("tool_call", async (event: ToolCallEvent): Promise<ToolCallEventResult | void> => {
        if (event.toolName === "dangerous_tool") {
            return { block: true, reason: "Tool blocked by extension" };
        }
    });

    // 订阅工具结果事件，可修改结果
    api.on("tool_result", async (event: ToolResultEvent) => {
        if (event.toolName === "bash" && event.isError) {
            // 添加错误处理逻辑
        }
    });
}
```

---

## 四、源码位置和行号

### 4.1 核心文件

| 文件 | 说明 |
|------|------|
| `packages/agent/src/types.ts` | AgentEvent 类型定义 (行 350-365) |
| `packages/agent/src/agent.ts` | Agent 类和事件发射 (行 158-222, 495-542) |
| `packages/coding-agent/src/core/extensions/types.ts` | 扩展事件类型 (行 507-1110) |
| `packages/coding-agent/src/core/extensions/runner.ts` | ExtensionRunner 事件派发 (行 676-1021) |

### 4.2 关键行号速查

#### Agent 事件系统
- `AgentEvent` 类型定义: `packages/agent/src/types.ts:350-365`
- `Agent.subscribe()` 方法: `packages/agent/src/agent.ts:219-222`
- `processEvents()` 方法: `packages/agent/src/agent.ts:495-542`
- 内部状态更新逻辑: `packages/agent/src/agent.ts:496-533`

#### 扩展事件系统
- `SessionEvent` 联合类型: `packages/coding-agent/src/core/extensions/types.ts:588-596`
- `ExtensionEvent` 联合类型: `packages/coding-agent/src/core/extensions/types.ts:941-962`
- `ExtensionAPI.on()` 定义: `packages/coding-agent/src/core/extensions/types.ts:1074-1110`
- `ExtensionRunner.emit()` 方法: `packages/coding-agent/src/core/extensions/runner.ts:676-708`
- `emitToolCall()` 方法: `packages/coding-agent/src/core/extensions/runner.ts:760-781`
- `emitToolResult()` 方法: `packages/coding-agent/src/core/extensions/runner.ts:710-758`
- 错误监听器: `packages/coding-agent/src/core/extensions/runner.ts:477-486`

---

## 五、最佳实践

### 5.1 事件订阅最佳实践

1. **尽早订阅关键事件**: 在扩展初始化时订阅 `session_start` 和 `agent_start` 事件
2. **使用 async 处理**: 异步事件处理器避免阻塞主线程
3. **正确处理取消**: 检查 `ctx.signal` 是否已中止
4. **错误边界**: 在事件处理中使用 try-catch 捕获错误，避免扩展崩溃影响 Agent

### 5.2 会话前事件最佳实践

对于 `session_before_switch`、`session_before_fork`、`session_before_compact`、`session_before_tree` 等可取消事件:

```typescript
api.on("session_before_switch", async (event, ctx): Promise<SessionBeforeSwitchResult | void> => {
    // 返回 cancel: true 可阻止会话切换
    if (someCondition) {
        return { cancel: true };
    }
});
```

### 5.3 工具事件最佳实践

#### tool_call 拦截
- 修改 `event.input` 应直接在原对象上进行
- 返回 `block: true` 阻止工具执行
- 使用 reason 提供清晰的阻止理由

#### tool_result 修改
- 通过返回 `content`、`details`、`isError` 修改结果
- 修改是可选的，不返回则保留原结果

### 5.4 错误处理最佳实践

```typescript
// 注册错误监听器
api.events.on("error", (error: ExtensionError) => {
    console.error(`Extension error in ${error.extensionPath}:`, error.error);
});
```

### 5.5 性能注意事项

1. **避免长时间同步操作**: 事件处理器应尽快返回
2. **批量处理**: 使用 `context` 事件批量修改消息，而不是每个消息订阅一次
3. **按需订阅**: 只订阅需要的事件，避免不必要的回调

---

## 六、事件流汇总图

```
用户输入
    │
    ▼
input 事件 ──────────────────────────────┐
    │                                │
    ▼                                │
before_agent_start 事件              │
    │                                │
    ▼                                │
agent_start 事件 ◄──────────────────┘
    │
    ├─► context 事件 ──► before_provider_request 事件
    │
    ▼
turn_start 事件
    │
    ├─► message_start 事件 ──► message_update 事件 ──► message_end 事件
    │                              │
    │                              └─► tool_call 事件 ──► tool_execution_start 事件
    │                                                          │
    │                              ┌───────────────────────────────┘
    │                              ▼
    │                     tool_result 事件 ──► tool_execution_end 事件
    │              │
    ▼              ▼
turn_end 事件 ◄─────┘
    │
    ▼
(循环或结束)
    │
    ▼
agent_end 事件
```

---

*报告生成时间: 2026-05-06*
*分析版本: 基于 pi-mono 仓库当前 HEAD*