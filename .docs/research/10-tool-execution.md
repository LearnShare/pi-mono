# Tool 执行流程研究

> 基于 pi-mono 源码深度分析
> 研究主题：02-detailed-research-plan.md 主题3

---

## 执行摘要

本文档深入分析 pi-coding-agent 的 Tool 发现、执行、结果收集与反馈流程，涵盖工具注册、执行模式、参数验证、结果处理等核心机制。

---

## 1. 工具发现机制

### 1.1 内置工具

pi 提供 6 个内置工具：

| 工具 | 说明 | 文件位置 |
|------|------|----------|
| `read` | 读取文件内容 | `packages/coding-agent/src/core/tools/read.ts` |
| `write` | 写入文件 | `packages/coding-agent/src/core/tools/write.ts` |
| `edit` | 编辑文件 | `packages/coding-agent/src/core/tools/edit.ts` |
| `bash` | 执行 Shell 命令 | `packages/coding-agent/src/core/tools/bash.ts` |
| `grep` | 文本搜索 | `packages/coding-agent/src/core/tools/grep.ts` |
| `find` | 文件查找 | `packages/coding-agent/src/core/tools/find.ts` |
| `ls` | 列出目录 | `packages/coding-agent/src/core/tools/ls.ts` |

### 1.2 工具定义结构

```typescript
// packages/coding-agent/src/core/extensions/types.ts
interface ToolDefinition<TParams = TSchema, TDetails = unknown, TState = any> {
  name: string;           // 工具名称（LLM 调用时使用）
  label: string;          // UI 显示标签
  description: string;    // LLM 描述
  promptSnippet?: string; // 系统提示中的工具摘要
  promptGuidelines?: string[]; // 系统提示中的指南
  parameters: TParams;    // TypeBox 参数 Schema
  renderShell?: "default" | "self";
  prepareArguments?: (args: unknown) => Static<TParams>;
  executionMode?: "parallel" | "sequential";
  execute: (
    toolCallId: string,
    params: Static<TParams>,
    signal: AbortSignal | undefined,
    onUpdate: AgentToolUpdateCallback<TDetails> | undefined,
    ctx: ExtensionContext,
  ) => Promise<AgentToolResult<TDetails>>;
  renderCall?: (...);     // 自定义调用渲染
  renderResult?: (...);   // 自定义结果渲染
}
```

### 1.3 工具创建入口

```typescript
// packages/coding-agent/src/core/tools/index.ts
export function createCodingToolDefinitions(cwd: string, options?: ToolsOptions): ToolDef[] {
  return [
    createReadToolDefinition(cwd, options?.read),
    createBashToolDefinition(cwd, options?.bash),
    createEditToolDefinition(cwd, options?.edit),
    createWriteToolDefinition(cwd, options?.write),
    createGrepToolDefinition(cwd, options?.grep),
    createFindToolDefinition(cwd, options?.find),
    createLsToolDefinition(cwd, options?.ls),
  ];
}
```

---

## 2. 工具注册

### 2.1 静态发现

扩展在 `session_start` 时被发现：

```typescript
// ResourceLoader 扫描扩展目录
- ~/.pi/agent/extensions/*.ts
- .pi/extensions/*.ts
- 自定义路径（通过 settings.json 配置）
```

### 2.2 动态注册

扩展通过 `pi.registerTool()` 动态注册：

```typescript
pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "Does something useful",
  parameters: Type.Object({
    input: Type.String({ description: "Input value" }),
  }),
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    return {
      content: [{ type: "text", text: `Result: ${params.input}` }],
      details: {},
    };
  },
});
```

### 2.3 工具过滤

通过 CLI 参数 `--tools` 过滤可用工具：

```bash
pi --tools read,edit,bash   # 只启用这3个工具
pi --no-tools               # 禁用所有工具
```

---

## 3. 执行流程

### 3.1 Agent Loop 概览

```typescript
// packages/agent/src/agent-loop.ts
async function runAgentLoop(...) {
  // 1. 流式调用 LLM
  const message = await streamAssistantResponse(context, config, signal, emit);

  // 2. 检查工具调用
  const toolCalls = message.content.filter((c) => c.type === "toolCall");

  // 3. 执行工具
  if (toolCalls.length > 0) {
    const executedToolBatch = await executeToolCalls(...);
    toolResults.push(...executedToolBatch.messages);
  }
}
```

### 3.2 工具调用链路

```
LLM 响应 (toolCall)
       │
       ▼
┌─────────────────┐
│ tool_execution  │
│ _start 事件     │
└────────┬────────┘
         ▼
┌─────────────────┐
│ tool_call 事件  │ ← 扩展可阻止
│ (可修改参数)    │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 参数验证        │ ← TypeBox Schema
│ (validateTool   │
│ Arguments)      │
└────────┬────────┘
         ▼
┌─────────────────┐
│ execute()       │
│ 执行工具        │
└────────┬────────┘
         ▼
┌─────────────────┐
│ tool_execution  │
│ _update 事件    │ ← 流式更新
└────────┬────────┘
         ▼
┌─────────────────┐
│ tool_execution  │
│ _end 事件       │
└────────┬────────┘
         ▼
┌─────────────────┐
│ tool_result     │ ← 构建结果消息
│ 事件            │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 注入到 LLM      │
│ 上下文          │
└─────────────────┘
```

### 3.3 执行模式

| 模式 | 说明 | 配置 |
|------|------|------|
| `parallel`（默认） | 同一助手消息中的多个工具调用并行执行 | 工具定义中指定 |
| `sequential` | 顺序执行，适用于有共享状态的工具（如游戏光标） | 工具定义中指定 |

```typescript
// 工具定义中指定执行模式
executionMode: "sequential"  // 或 "parallel"
```

---

## 4. 参数处理

### 4.1 TypeBox Schema 验证

```typescript
import { Type } from "typebox";

parameters: Type.Object({
  input: Type.String({ description: "Input value" }),
  count: Type.Optional(Type.Integer({ description: "Count" })),
})
```

### 4.2 参数准备（prepareArguments）

可选的兼容性适配器：

```typescript
prepareArguments?: (args: unknown) => Static<TParams>;
```

### 4.3 错误处理

不合法的参数返回错误结果：

```typescript
return {
  content: [{ type: "text", text: "Invalid parameters: ..." }],
  isError: true,
};
```

---

## 5. 结果处理

### 5.1 ExecuteResult 结构

```typescript
interface AgentToolResult<TDetails = unknown> {
  content: (TextContent | ImageContent)[];  // 发送给 LLM 的内容
  details?: TDetails;                         // 扩展特定数据（不发送给 LLM）
  isError?: boolean;                          // 是否为错误结果
}
```

### 5.2 内容类型

| 类型 | 说明 |
|------|------|
| `TextContent` | 文本结果 `{ type: "text", text: "..." }` |
| `ImageContent` | 图片结果 `{ type: "image", source: "base64", ... }` |

### 5.3 details 字段

`details` 字段用于扩展状态持久化，**不会**发送给 LLM：

```typescript
async execute(toolCallId, params, signal, onUpdate, ctx) {
  return {
    content: [{ type: "text", text: "Todo added" }],
    details: { todos: currentTodos, nextId: id + 1 },  // 扩展内部使用
  };
}
```

---

## 6. 工具执行事件

### 6.1 完整事件顺序

```
tool_execution_start  →  tool_call  →  tool_execution_update*  →  tool_result  →  tool_execution_end
```

### 6.2 扩展可拦截的事件

```typescript
// 工具调用前 - 可阻止执行
pi.on("tool_call", async (event, ctx) => {
  // event.toolName, event.toolCallId, event.input
  return { block: true, reason: "Blocked by extension" };
});

// 工具结果后 - 可修改结果
pi.on("tool_result", async (event, ctx) => {
  // event.content, event.details, event.isError
  return { content: [...newContent], isError: false };
});
```

### 6.3 流式更新

工具可以通过 `onUpdate` 回调发送流式输出：

```typescript
async execute(toolCallId, params, signal, onUpdate, ctx) {
  for (const chunk of processLongOperation(params)) {
    onUpdate({ progress: chunk.percent, partial: chunk.data });
  }
  return { content: [...], details: {} };
}
```

---

## 7. 会话持久化

### 7.1 工具结果存储

执行后的 `ToolResultMessage` 写入 Session 文件：

```typescript
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

### 7.2 从历史恢复状态

```typescript
pi.on("session_start", async (_event, ctx) => {
  let todos = [];
  for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type === "message" && entry.message.toolName === "todo_manager") {
      todos = entry.message.details?.todos ?? todos;
    }
  }
});
```

---

## 8. 关键源码位置

| 文件 | 作用 |
|------|------|
| `packages/coding-agent/src/core/tools/*.ts` | 6 个内置工具实现 |
| `packages/coding-agent/src/core/extensions/types.ts` | ToolDefinition 接口 |
| `packages/agent/src/agent-loop.ts` | 工具调用在 Agent Loop 中的位置 |
| `packages/coding-agent/src/core/tool-definition-wrapper.ts` | 工具定义包装 |

---

## 9. 研究问题解答

### Q1: 工具在哪个生命周期点可用？

工具在 `session_start` 时被发现和注册。扩展注册的工具立即可用。

### Q2: 并行 vs 串行执行如何选择？

- 默认 `parallel`：大多数工具可并行执行
- 串行 `sequential`：适用于有共享状态的工具（如游戏光标移动）

### Q3: details 字段是否发送给 LLM？

`details` 是扩展专用数据，**不会**发送给 LLM，仅用于扩展状态持久化。

### Q4: 工具结果何时注入上下文？

工具执行完成后，结果立即添加到 `context.messages`，然后继续下一轮 LLM 调用。

---

## 10. 流程图

### 10.1 完整工具执行流程

```
LLM 返回 toolCall
       │
       ▼
┌─────────────────┐
│ 事件: tool_     │
│ execution_start │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 事件: tool_call │
│ (扩展可阻止)    │
└────────┬────────┘
         ▼
    ┌────┴────┐
    │ 参数    │
    │ 验证    │
    └────┬────┘
         ▼
┌─────────────────┐
│ execute()       │
│ 执行工具        │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 事件: tool_     │
│ execution_update│
│ (流式输出)      │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 构建 Execute    │
│ Result          │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 事件: tool_     │
│ result          │
│ (扩展可修改)    │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 添加到消息队列  │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 事件: tool_     │
│ execution_end   │
└─────────────────┘
```

---

*本文档为研究产出，后续可根据需要进一步补充和修正。*