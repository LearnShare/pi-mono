# Pi Agent: 扩展开发 API 参考

> 基于 pi.dev/docs/latest/extensions 官方文档和源码整理
> 最后更新: 2026-04-29

---

## 1. 扩展基础

### 扩展入口

扩展导出默认工厂函数，接收 `ExtensionAPI`：

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  // 同步或异步工厂函数
}

// 异步工厂（用于启动时获取远程配置等）
export default async function (pi: ExtensionAPI) {
  const response = await fetch("http://localhost:1234/v1/models");
  const data = await response.json();
  pi.registerProvider("local-openai", { ... });
}
```

### 可用导入

| 包 | 用途 |
|------|------|
| `@mariozechner/pi-coding-agent` | 扩展类型（`ExtensionAPI`, `ExtensionContext`, 事件） |
| `typebox` | 工具参数 Schema 定义 |
| `@mariozechner/pi-ai` | AI 工具（`StringEnum` 等） |
| `@mariozechner/pi-tui` | TUI 组件 |
| `node:*` | Node.js 内置模块 |

### 扩展文件结构

**单文件：**
```
~/.pi/agent/extensions/my-extension.ts
```

**多文件目录：**
```
~/.pi/agent/extensions/my-extension/
├── index.ts        # 入口
├── tools.ts        # 辅助模块
└── utils.ts
```

**带依赖的包：**
```
~/.pi/agent/extensions/my-extension/
├── package.json
├── node_modules/
└── src/index.ts
```

```json
// package.json
{
  "name": "my-extension",
  "dependencies": { "zod": "^3.0.0" },
  "pi": { "extensions": ["./src/index.ts"] }
}
```

---

## 2. ExtensionAPI 核心方法

### `pi.on(event, handler)`

订阅生命周期事件。所有事件签名详见第 4 节。

```typescript
pi.on("tool_call", async (event, ctx) => {
  // event 包含事件特定数据
  // ctx: ExtensionContext
});
```

### `pi.registerTool(definition)`

注册可被 LLM 调用的自定义工具。

```typescript
import { Type } from "typebox";

pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "Does something useful",

  // TypeBox 定义参数 Schema
  parameters: Type.Object({
    input: Type.String({ description: "Input value" }),
    count: Type.Optional(Type.Integer({ description: "Count" })),
  }),

  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // toolCallId: string - 工具调用 ID
    // params: 解析后的参数对象
    // signal: AbortSignal - 用于取消
    // onUpdate: (data) => void - 发送流式更新
    // ctx: ExtensionContext

    // 返回结果
    return {
      content: [{ type: "text", text: `Result: ${params.input}` }],
      details: { /* 任意 JSON 可序列化数据 */ },
    };
  },
});
```

**工具参数的重要约定：**

- 使用 `StringEnum` 而非 `Type.Union(Type.Literal())` 定义枚举，后者在 Google 的 API 上不兼容：

```typescript
import { StringEnum } from "@mariozechner/pi-ai";

// 正确
action: StringEnum(["list", "add"] as const)

// 错误 - 不能与 Google 兼容
action: Type.Union([Type.Literal("list"), Type.Literal("add")])
```

### `pi.registerCommand(name, options)`

注册斜杠命令（如 `/mycommand`）。

```typescript
pi.registerCommand("greet", {
  description: "Greet someone",
  handler: async (args, ctx) => {
    // args: string - 命令后的参数文本
    // ctx: ExtensionCommandContext（扩展了 ExtensionContext）
    ctx.ui.notify(`Hello ${args || "world"}!`, "info");
  },
});
```

### `pi.registerShortcut(shortcut, options)`

注册键盘快捷键。

```typescript
pi.registerShortcut("ctrl+shift+x", {
  description: "Do something",
  handler: async (ctx) => {
    // ctx: ExtensionCommandContext
    ctx.ui.notify("Shortcut triggered!", "info");
  },
});
```

### `pi.registerFlag(name, options)`

注册 CLI flag。

```typescript
pi.registerFlag("my-flag", {
  description: "Enable my feature",
  handler: async (value, ctx) => {
    // value: string | true | undefined
  },
});
```

### `pi.registerProvider(name, config)`

注册自定义模型提供商。

```typescript
pi.registerProvider("my-llm", {
  baseUrl: "https://api.my-llm.com/v1",
  apiKey: "MY_LLM_API_KEY",
  api: "openai-completions",
  models: [...],
  oauth: { /* OAuth 配置 */ },
  streamSimple: streamMyProvider, // 自定义流式实现
});
```

### `pi.unregisterProvider(name)`

移除已注册的提供商。

```typescript
pi.unregisterProvider("my-llm");
```

### `pi.sendMessage(message, options?)`

发送消息给 LLM（用于命令中主动发起 LLM 调用）。

```typescript
await pi.sendMessage({
  role: "user",
  content: [{ type: "text", text: "Analyze this code" }],
});
```

### `pi.sendUserMessage(content, options?)`

从扩展发送用户消息（触发 agent 处理）。

```typescript
await pi.sendUserMessage([
  { type: "text", text: "Continue with the refactoring" },
]);
```

### `pi.appendEntry(customType, data?)`

向会话追加自定义条目（持久化扩展数据）。

```typescript
pi.appendEntry("my_state", { todos: [...], count: 5 });
```

### `pi.setSessionName(name)`

设置会话显示名称。

```typescript
pi.setSessionName("Refactoring module X");
```

### `pi.getSessionName()`

获取当前会话名称。

### `pi.setLabel(entryId, label)`

给树节点设置标签（便于 `/tree` 导航）。

```typescript
pi.setLabel("entry-id-123", "checkpoint-1");
```

### `pi.getCommands()`

获取所有已注册的命令列表。

### `pi.registerMessageRenderer(customType, renderer)`

注册自定义消息渲染器。

```typescript
pi.registerMessageRenderer("my_type", {
  render: (entry, context) => {
    return { summary: "Summary text", detail: "Detail text" };
  },
});
```

### `pi.exec(command, args, options?)`

执行另一个扩展命令。

```typescript
await pi.exec("hello", "world", { waitForIdle: true });
```

### `pi.getActiveTools() / pi.getAllTools() / pi.setActiveTools(names)`

管理当前启用的工具。

```typescript
const active = pi.getActiveTools();
const all = pi.getAllTools();
pi.setActiveTools(["read", "bash", "my_tool"]);
```

### `pi.setModel(model)`

切换模型。

```typescript
pi.setModel({ provider: "anthropic", id: "claude-sonnet-4-20250514", ... });
```

### `pi.getThinkingLevel() / pi.setThinkingLevel(level)`

获取/设置思考级别。

```typescript
const level = pi.getThinkingLevel(); // "off" | "minimal" | "low" | "medium" | "high" | "xhigh"
pi.setThinkingLevel("high");
```

### `pi.events`

事件总线，用于扩展间通信。

```typescript
// 扩展 A 发射事件
pi.events.emit("my-ext:data", { key: "value" });

// 扩展 B 监听
pi.events.on("my-ext:data", (data) => {
  console.log(data);
});
```

---

## 3. ExtensionContext

所有事件处理器接收 `ctx: ExtensionContext`。

### `ctx.ui` — 用户交互方法

| 方法 | 说明 |
|------|------|
| `ctx.ui.notify(message, level)` | 显示通知（"info", "success", "warning", "error"） |
| `ctx.ui.confirm(title, message)` | 确认对话框，返回 `boolean` |
| `ctx.ui.select(title, options)` | 选择列表，返回选中值 |
| `ctx.ui.input(title, placeholder?)` | 文本输入，返回 `string` |
| `ctx.ui.editor(title, initial?)` | 多行编辑器，返回 `string` |
| `ctx.ui.setStatus(name, text)` | 在底部栏设置状态文本 |
| `ctx.ui.setWidget(name, lines, placement?)` | 在编辑器上方或下方显示 widget |
| `ctx.ui.setTitle(text)` | 设置终端标题 |
| `ctx.ui.setEditorText(text)` | 在编辑器中设置文本 |
| `ctx.ui.setHiddenThinkingLabel(label)` | 自定义折叠的 thinking 标签 |
| `ctx.ui.setWorkingIndicator(text)` | 自定义流式工作指示器 |
| `ctx.ui.setFooter(component)` | 设置完全自定义的底部栏 |
| `ctx.ui.setHeader(component)` | 设置自定义头部 |
| `ctx.ui.setEditorComponent(component)` | 替换编辑器为自定义组件 |
| `ctx.ui.custom(component)` | 显示完全自定义的 TUI 组件 |

### `ctx.hasUI`

布尔值，`false` 在 print 模式和 JSON 模式下。

### `ctx.cwd`

当前工作目录。

### `ctx.sessionManager`

只读访问会话状态。

```typescript
ctx.sessionManager.getEntries()  // 所有条目
ctx.sessionManager.getBranch()   // 当前分支
ctx.sessionManager.getLeafId()   // 当前叶节点 ID
ctx.sessionManager.getSessionFile() // 会话文件路径
```

### `ctx.modelRegistry / ctx.model`

模型注册表和当前模型访问。

### `ctx.signal`

当前 agent 的 AbortSignal（用于 fetch 等异步操作的取消）。

```typescript
const response = await fetch("https://api.example.com", {
  signal: ctx.signal,
});
```

### `ctx.isIdle() / ctx.abort() / ctx.hasPendingMessages()`

流程控制。

```typescript
if (ctx.isIdle()) { /* agent 空闲 */ }
ctx.abort(); // 终止当前操作
const hasPending = ctx.hasPendingMessages();
```

### `ctx.shutdown()`

优雅关闭 pi。

### `ctx.getContextUsage()`

获取当前上下文使用情况（token 数等）。

### `ctx.compact(options?)`

触发压缩（非阻塞）。

```typescript
ctx.compact({
  customInstructions: "Focus on recent changes",
  onComplete: (result) => ctx.ui.notify("Done", "info"),
  onError: (error) => ctx.ui.notify(error.message, "error"),
});
```

### `ctx.getSystemPrompt()`

获取当前系统提示字符串。

### `ExtensionCommandContext` 特有方法

命令处理器（`pi.registerCommand`）收到的 `ctx` 扩展了以下方法：

```typescript
// 等待 agent 空闲
await ctx.waitForIdle();

// 新建会话
await ctx.newSession({
  parentSession: "session-path",
  setup: async (sm) => { /* 设置新会话 */ },
  withSession: async (ctx) => { /* 在新会话中工作 */ },
});

// Fork 会话
await ctx.fork("entry-id-123", {
  position: "before", // 或 "at"（克隆）
  withSession: async (ctx) => { /* 在 forked 会话中工作 */ },
});

// 导航树
await ctx.navigateTree("entry-id-456", {
  summarize: true,
  customInstructions: "Focus on error handling",
});

// 切换会话
await ctx.switchSession("/path/to/session.jsonl", {
  withSession: async (ctx) => { /* ... */ },
});

// 重载（同 /reload）
await ctx.reload();
```

---

## 4. 完整事件参考

### 事件生命周期

```
pi 启动
  ├─ session_start { reason }
  └─ resources_discover { reason }

用户发送提示
  ├─ input（可拦截、转换、处理）
  ├─ before_agent_start（可注入消息、修改系统提示）
  ├─ agent_start
  ├─ turn_start
  ├─ context（可修改消息）
  ├─ before_provider_request（检查/替换 payload）
  ├─ after_provider_response（检查状态码/响应头）
  │   LLM 响应，可能调用工具：
  │     ├─ tool_execution_start
  │     ├─ tool_call（可阻止）
  │     ├─ tool_execution_update
  │     ├─ tool_result（可修改）
  │     └─ tool_execution_end
  ├─ turn_end
  ├─ agent_end

/new, /resume
  ├─ session_before_switch（可取消）
  ├─ session_shutdown
  ├─ session_start { reason }
  └─ resources_discover

/fork, /clone
  ├─ session_before_fork（可取消）
  ├─ session_shutdown
  ├─ session_start { reason }
  └─ resources_discover

/compact 或自动压缩
  ├─ session_before_compact（可取消或自定义）
  └─ session_compact

/tree 导航
  ├─ session_before_tree（可取消或自定义）
  └─ session_tree

/model 或 Ctrl+P
  └─ model_select

退出
  └─ session_shutdown
```

### 事件签名

#### Resource Events

**`resources_discover`** — 扩展可贡献额外技能、提示和主题路径。

```typescript
pi.on("resources_discover", async (event, ctx) => {
  // event.cwd - 当前工作目录
  // event.reason - "startup" | "reload"
  return {
    skillPaths: ["/path/to/skills"],
    promptPaths: ["/path/to/prompts"],
    themePaths: ["/path/to/themes"],
  };
});
```

#### Session Events

**`session_start`** — 会话开始。

```typescript
pi.on("session_start", async (event, ctx) => {
  // event.reason - "startup" | "reload" | "new" | "resume" | "fork"
  // event.previousSessionFile - 前一会话文件路径
});
```

**`session_before_switch`** — 切换会话前。

```typescript
pi.on("session_before_switch", async (event, ctx) => {
  // event.reason - "new" | "resume"
  // event.targetSessionFile - 目标会话文件
  return { cancel: true }; // 取消切换
});
```

**`session_before_fork`** — Fork/clone 前。

```typescript
pi.on("session_before_fork", async (event, ctx) => {
  // event.entryId - 选中的条目 ID
  // event.position - "before" (/fork) 或 "at" (/clone)
  return { cancel: true };
});
```

**`session_before_compact`** — 压缩前。可取消或提供自定义摘要。

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  // event.preparation - 压缩准备数据
  // event.branchEntries - 分支上的所有条目
  // event.customInstructions - 用户自定义指令
  // event.signal - AbortSignal

  // 取消：
  return { cancel: true };

  // 自定义摘要：
  return {
    compaction: {
      summary: "Your summary...",
      firstKeptEntryId: event.preparation.firstKeptEntryId,
      tokensBefore: event.preparation.tokensBefore,
      details: { /* 自定义数据 */ },
    },
  };
});
```

**`session_compact`** — 压缩完成。

```typescript
pi.on("session_compact", async (event, ctx) => {
  // event.compactionEntry - 保存的压缩条目
  // event.fromExtension - 是否来自扩展
});
```

**`session_before_tree`** — 树导航前。

```typescript
pi.on("session_before_tree", async (event, ctx) => {
  // event.preparation.targetId - 导航目标
  // event.preparation.oldLeafId - 当前位置
  // event.preparation.entriesToSummarize - 将摘要的条目

  return { cancel: true };
  // 或提供自定义摘要：
  return { summary: { summary: "...", details: {} } };
});
```

**`session_tree`** — 树导航完成。

```typescript
pi.on("session_tree", async (event, ctx) => {
  // event.newLeafId, event.oldLeafId
  // event.summaryEntry, event.fromExtension
});
```

**`session_shutdown`** — 会话关闭。

```typescript
pi.on("session_shutdown", async (event, ctx) => {
  // event.reason - "quit" | "reload" | "new" | "resume" | "fork"
  // event.targetSessionFile - 目标会话文件
  // 清理、保存状态等
});
```

#### Agent Events

**`before_agent_start`** — 用户提交提示后、agent 循环前。

```typescript
pi.on("before_agent_start", async (event, ctx) => {
  // event.prompt - 用户提示文本
  // event.images - 附件图片
  // event.systemPrompt - 当前链式系统提示
  // event.systemPromptOptions - 构建系统提示的结构化选项

  return {
    message: {
      customType: "my-extension",
      content: "Additional context for the LLM",
      display: true,
    },
    systemPrompt: event.systemPrompt + "\nExtra instructions...",
  };
});
```

**`agent_start / agent_end`** — agent 处理开始/结束。

```typescript
pi.on("agent_start", async (_event, ctx) => {});

pi.on("agent_end", async (event, ctx) => {
  // event.messages - 此轮提示产生的消息
});
```

**`turn_start / turn_end`** — 每轮 LLM 响应 + 工具调用。

```typescript
pi.on("turn_start", async (event, ctx) => {
  // event.turnIndex, event.timestamp
});

pi.on("turn_end", async (event, ctx) => {
  // event.turnIndex, event.message, event.toolResults
});
```

**`message_start / message_update / message_end`** — 消息生命周期。

```typescript
pi.on("message_start", async (event, ctx) => {
  // event.message
});

pi.on("message_update", async (event, ctx) => {
  // event.message
  // event.assistantMessageEvent - token 级流式事件
});

pi.on("message_end", async (event, ctx) => {
  // event.message
});
```

**`tool_execution_start / tool_execution_update / tool_execution_end`** — 工具执行生命周期。

```typescript
pi.on("tool_execution_start", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.args
});

pi.on("tool_execution_update", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.args, event.partialResult
});

pi.on("tool_execution_end", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.result, event.isError
});
```

**`context`** — 每次 LLM 调用前。可非破坏性修改消息。

```typescript
pi.on("context", async (event, ctx) => {
  // event.messages - 深拷贝，安全修改
  return { messages: filteredMessages };
});
```

**`before_provider_request`** — 发送请求前。

```typescript
pi.on("before_provider_request", (event, ctx) => {
  // event.payload - 提供商特定 payload
  // 返回新 payload 替换
  // return { ...event.payload, temperature: 0 };
});
```

**`after_provider_response`** — 收到响应后。

```typescript
pi.on("after_provider_response", (event, ctx) => {
  // event.status - HTTP 状态码
  // event.headers - 规范化响应头
  if (event.status === 429) {
    console.log("rate limited", event.headers["retry-after"]);
  }
});
```

#### Tool Events

**`tool_call`** — 工具调用前。**可阻止执行**。

```typescript
import { isToolCallEventType } from "@mariozechner/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
  // event.toolName, event.toolCallId, event.input

  // 使用类型守卫
  if (isToolCallEventType("bash", event)) {
    // event.input 是 { command: string; timeout?: number }
    event.input.command = `source ~/.profile\n${event.input.command}`;

    if (event.input.command.includes("rm -rf")) {
      return { block: true, reason: "Dangerous command" };
    }
  }

  // 自定义工具类型守卫
  if (isToolCallEventType<"my_tool", MyToolInput>("my_tool", event)) {
    event.input.action; // 类型安全
  }
});
```

**`tool_result`** — 工具执行完成后。**可修改结果**。

```typescript
import { isBashToolResult } from "@mariozechner/pi-coding-agent";

pi.on("tool_result", async (event, ctx) => {
  // event.toolName, event.toolCallId, event.input
  // event.content, event.details, event.isError

  // 使用类型守卫获取 Bash 详情
  if (isBashToolResult(event)) {
    // event.details 类型为 BashToolDetails
  }

  // 修改结果
  return { content: [...newContent], details: {...updatedDetails}, isError: false };
});
```

#### User Bash Events

**`user_bash`** — 用户执行 `!` 或 `!!` 命令时。

```typescript
import { createLocalBashOperations } from "@mariozechner/pi-coding-agent";

pi.on("user_bash", (event, ctx) => {
  // event.command - bash 命令
  // event.excludeFromContext - 是否 !! 前缀
  // event.cwd - 工作目录

  // 选项 1：提供自定义操作（如 SSH）
  return { operations: remoteBashOps };

  // 选项 2：包装内置 bash
  return {
    operations: {
      exec(command, cwd, options) {
        return local.exec(`source ~/.profile\n${command}`, cwd, options);
      },
    },
  };
});
```

#### Input Events

**`input`** — 用户输入后、技能/模板扩展前。

```typescript
pi.on("input", async (event, ctx) => {
  // event.text - 原始输入
  // event.images - 附件图片
  // event.source - "interactive" | "rpc" | "extension"

  // 转换输入
  if (event.text.startsWith("?quick "))
    return { action: "transform", text: `Respond briefly: ${event.text.slice(7)}` };

  // 处理（不经过 LLM）
  if (event.text === "ping") {
    ctx.ui.notify("pong", "info");
    return { action: "handled" };
  }

  // 放行（默认）
  return { action: "continue" };
});
```

#### Model Events

**`model_select`** — 模型切换时。

```typescript
pi.on("model_select", async (event, ctx) => {
  // event.model - 新模型
  // event.previousModel - 先前模型
  // event.source - "set" | "cycle" | "restore"
  ctx.ui.notify(`Model: ${event.model.id}`, "info");
});
```

---

## 5. 状态管理

### 通过工具返回 details 持久化

```typescript
pi.registerTool({
  name: "todo_manager",
  // ...
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    return {
      content: [{ type: "text", text: "Todo added" }],
      details: { todos: currentTodos, nextId: id + 1 },
    };
  },
});
```

### 从会话历史恢复状态

```typescript
pi.on("session_start", async (_event, ctx) => {
  let todos = [];
  for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type === "message" && entry.message.toolName === "todo_manager") {
      todos = entry.message.details?.todos ?? todos;
    }
  }
  // 重建状态
});
```

### 事件总线通信

```typescript
// 扩展间通信
const eventBus = createEventBus();

// 加载到 DefaultResourceLoader
const loader = new DefaultResourceLoader({ eventBus });
```

---

## 6. 自定义工具编写指南

### ToolDefinition 类型

```typescript
interface ToolDefinition {
  name: string;            // 工具名称（给 LLM 使用）
  label: string;           // 显示标签
  description: string;     // 描述（LLM 决定何时调用）
  parameters: TSchema;     // TypeBox Schema
  promptSnippet?: string;  // 添加到系统提示的工具代码片段
  promptGuidelines?: string[]; // 添加到系统提示的指南
  neverDelete?: boolean;   // 标记为不可删除
  executionMode?: "parallel" | "sequential"; // 执行模式
  terminate?: boolean;     // true 时工具调用后结束
  execute(toolCallId: string, params: T, signal: AbortSignal, onUpdate: (data: any) => void, ctx: ExtensionContext): Promise<ExecuteResult>;
}
```

### 执行模式

- `"parallel"`（默认）：单条助手消息中的多个工具调用并行执行
- `"sequential"`：顺序执行，适用于有共享状态的工具（如游戏光标）

### 流式更新

```typescript
async execute(toolCallId, params, signal, onUpdate, ctx) {
  for (const chunk of processLongOperation(params)) {
    onUpdate({ progress: chunk.percent, partial: chunk.data });
  }
  return { content: [...], details: {} };
}
```

---

## 7. 自定义流式 API（Custom Streaming）

适用于非标准 API 的提供商。使用 `pi.registerProvider()` 的 `streamSimple` 参数。

### 流模式

```typescript
import {
  type AssistantMessage,
  type AssistantMessageEventStream,
  type Context,
  type Model,
  type SimpleStreamOptions,
  calculateCost,
  createAssistantMessageEventStream,
} from "@mariozechner/pi-ai";

function streamMyProvider(
  model: Model<any>,
  context: Context,
  options?: SimpleStreamOptions
): AssistantMessageEventStream {
  const stream = createAssistantMessageEventStream();
  const output: AssistantMessage = {
    role: "assistant", content: [], api: model.api,
    provider: model.provider, model: model.id,
    usage: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, totalTokens: 0, cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 } },
    stopReason: "stop", timestamp: Date.now(),
  };

  (async () => {
    try {
      stream.push({ type: "start", partial: output });
      // ... 处理响应，推送 content 事件 ...
      stream.push({ type: "done", reason: output.stopReason, message: output });
      stream.end();
    } catch (error) {
      output.stopReason = options?.signal?.aborted ? "aborted" : "error";
      stream.push({ type: "error", reason: output.stopReason, error: output });
      stream.end();
    }
  })();

  return stream;
}
```

### 事件类型

按以下顺序推送事件：

1. `{ type: "start", partial: output }` — 流开始
2. 内容事件（可重复）：
   - `{ type: "text_start", contentIndex, partial }` — 文本块开始
   - `{ type: "text_delta", contentIndex, delta, partial }` — 文本增量
   - `{ type: "text_end", contentIndex, content, partial }` — 文本块结束
   - `{ type: "thinking_start", contentIndex, partial }` — 思考开始
   - `{ type: "thinking_delta", contentIndex, delta, partial }` — 思考增量
   - `{ type: "thinking_end", contentIndex, content, partial }` — 思考结束
   - `{ type: "toolcall_start", contentIndex, partial }` — 工具调用开始
   - `{ type: "toolcall_delta", contentIndex, delta, partial }` — 工具调用 JSON 增量
   - `{ type: "toolcall_end", contentIndex, toolCall, partial }` — 工具调用结束
3. `{ type: "done", reason, message }` 或 `{ type: "error", reason, error }` — 流结束

### 注册自定义流

```typescript
pi.registerProvider("my-provider", {
  baseUrl: "https://api.example.com",
  apiKey: "MY_API_KEY",
  api: "my-custom-api",
  models: [...],
  streamSimple: streamMyProvider,
});
```

---

## 8. OAuth 支持

```typescript
import type { OAuthCredentials, OAuthLoginCallbacks } from "@mariozechner/pi-ai";

pi.registerProvider("corporate-ai", {
  baseUrl: "https://ai.corp.com/v1",
  api: "openai-responses",
  models: [...],
  oauth: {
    name: "Corporate AI (SSO)",

    async login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials> {
      // 选项 1：浏览器 OAuth
      callbacks.onAuth({ url: "https://sso.corp.com/authorize?..." });

      // 选项 2：设备码流程
      callbacks.onDeviceCode({
        userCode: "ABCD-1234",
        verificationUri: "https://sso.corp.com/device",
      });

      // 选项 3：提示输入令牌
      const code = await callbacks.onPrompt({ message: "Enter SSO code:" });

      // 交换令牌
      return {
        refresh: tokens.refreshToken,
        access: tokens.accessToken,
        expires: Date.now() + tokens.expiresIn * 1000,
      };
    },

    async refreshToken(credentials: OAuthCredentials): Promise<OAuthCredentials> {
      // 实现令牌刷新
    },

    getApiKey(credentials: OAuthCredentials): string {
      return credentials.access;
    },

    modifyModels(models, credentials) {
      // 可选：根据用户订阅修改模型列表
      return models;
    },
  },
});
```

### OAuthLoginCallbacks

```typescript
interface OAuthLoginCallbacks {
  onAuth(params: { url: string }): void;                              // 打开浏览器
  onDeviceCode(params: { userCode: string; verificationUri: string }): void; // 设备码
  onPrompt(params: { message: string }): Promise<string>;             // 提示输入
}
```

### OAuthCredentials

```typescript
interface OAuthCredentials {
  refresh: string;   // 刷新令牌
  access: string;    // 访问令牌
  expires: number;   // 过期时间戳（毫秒）
}
```

---

## 9. 配置参考

### ProviderConfig

```typescript
interface ProviderConfig {
  baseUrl?: string;
  apiKey?: string;
  api?: Api;
  streamSimple?: (model, context, options?) => AssistantMessageEventStream;
  headers?: Record<string, string>;
  authHeader?: boolean;
  models?: ProviderModelConfig[];
  oauth?: { name, login, refreshToken, getApiKey, modifyModels? };
}
```

### ProviderModelConfig

```typescript
interface ProviderModelConfig {
  id: string;
  name: string;
  api?: Api;
  reasoning: boolean;
  input: ("text" | "image")[];
  cost: { input: number; output: number; cacheRead: number; cacheWrite: number };
  contextWindow: number;
  maxTokens: number;
  headers?: Record<string, string>;
  compat?: {
    supportsStore?: boolean;
    supportsDeveloperRole?: boolean;
    supportsReasoningEffort?: boolean;
    reasoningEffortMap?: Partial<Record<ThinkingLevel, string>>;
    supportsUsageInStreaming?: boolean;
    maxTokensField?: "max_completion_tokens" | "max_tokens";
    requiresToolResultName?: boolean;
    requiresAssistantAfterToolResult?: boolean;
    requiresThinkingAsText?: boolean;
    requiresReasoningContentOnAssistantMessages?: boolean;
    thinkingFormat?: "openai" | "deepseek" | "zai" | "qwen" | "qwen-chat-template";
    cacheControlFormat?: "anthropic";
  };
}
```

---

## 10. 测试建议

测试自定义提供商时，参考 `packages/ai/test/` 中的测试文件：

| 测试文件 | 测试目标 |
|---------|---------|
| `stream.test.ts` | 基础流式、文本输出 |
| `tokens.test.ts` | token 计数和使用量 |
| `abort.test.ts` | AbortSignal 处理 |
| `empty.test.ts` | 空/最小响应 |
| `context-overflow.test.ts` | 上下文窗口限制 |
| `image-limits.test.ts` | 图片输入处理 |
| `unicode-surrogate.test.ts` | Unicode 边界情况 |
| `tool-call-without-result.test.ts` | 工具调用边界情况 |
| `cross-provider-handoff.test.ts` | 提供商间上下文交接 |