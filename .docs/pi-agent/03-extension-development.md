# 为 PI Agent 编写扩展的详细方法和 API

## 概述

PI Agent 扩展是 TypeScript 模块，通过工厂函数模式实现，可订阅生命周期事件、注册自定义工具、命令和快捷键，并与用户进行交互。

## 扩展架构

### 工厂模式

扩展导出默认工厂函数而非类实例：

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  // 扩展逻辑
}
```

工厂函数可以是同步或异步的：

```typescript
// 同步工厂
export default function (pi: ExtensionAPI) {
  pi.on("session_start", () => {
    console.log("Extension loaded");
  });
}

// 异步工厂（用于一次性初始化工作，如获取远程配置）
export default async function (pi: ExtensionAPI) {
  const response = await fetch("http://localhost:1234/v1/models");
  const payload = await response.json();
  
  pi.registerProvider("local-provider", {
    baseUrl: "http://localhost:1234/v1",
    api: "openai-completions",
    models: payload.data.map((model: any) => ({
      id: model.id,
      name: model.name ?? model.id,
      reasoning: false,
      input: ["text"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: model.context_window ?? 128000,
      maxTokens: model.max_tokens ?? 4096,
    })),
  });
}
```

## ExtensionAPI 接口详解

### 事件订阅

```typescript
// 会话事件
pi.on("session_start", handler);
pi.on("session_shutdown", handler);
pi.on("session_before_switch", handler);
pi.on("session_before_fork", handler);
pi.on("session_before_compact", handler);
pi.on("session_compact", handler);
pi.on("session_before_tree", handler);
pi.on("session_tree", handler);

// Agent 事件
pi.on("before_agent_start", handler);
pi.on("agent_start", handler);
pi.on("agent_end", handler);
pi.on("turn_start", handler);
pi.on("turn_end", handler);

// 消息事件
pi.on("message_start", handler);
pi.on("message_update", handler);
pi.on("message_end", handler);

// 工具执行事件
pi.on("tool_execution_start", handler);
pi.on("tool_execution_update", handler);
pi.on("tool_execution_end", handler);

// 工具拦截事件
pi.on("tool_call", handler);   // 可拦截/修改
pi.on("tool_result", handler);   // 可修改结果

// 上下文事件
pi.on("context", handler);

// Provider 事件
pi.on("before_provider_request", handler);
pi.on("after_provider_response", handler);

// 模型事件
pi.on("model_select", handler);

// 资源事件
pi.on("resources_discover", handler);

// 输入事件
pi.on("input", handler);
pi.on("user_bash", handler);
```

### 工具注册

```typescript
pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "What this tool does",
  parameters: Type.Object({
    action: StringEnum(["list", "add"] as const),
    text: Type.Optional(Type.String()),
  }),
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // 返回结果
    return {
      content: [{ type: "text", text: "Done" }],
      details: {},
    };
  },
});
```

### 命令注册

```typescript
pi.registerCommand("mycommand", {
  description: "Description shown in help",
  getArgumentCompletions?: (prefix: string) => AutocompleteItem[] | null;
  handler: async (args, ctx) => {
    ctx.ui.notify(`Executed with args: ${args}`, "info");
  },
});
```

### 快捷键注册

```typescript
pi.registerShortcut("ctrl+shift+p", {
  description: "Toggle plan mode",
  handler: async (ctx) => {
    ctx.ui.notify("Toggled!");
  },
});
```

### CLI 标志注册

```typescript
pi.registerFlag("plan", {
  description: "Start in plan mode",
  type: "boolean",
  default: false,
});

// 获取标志值
if (pi.getFlag("plan")) {
  // Plan mode enabled
}
```

### 消息操作

```typescript
// 发送自定义消息
pi.sendMessage({
  customType: "my-extension",
  content: "Message text",
  display: true,
  details: { key: "value" },
}, {
  deliverAs: "steer",  // "steer" | "followUp" | "nextTurn"
  triggerTurn: true,
});

// 发送用户消息（触发 Agent）
pi.sendUserMessage("What is 2+2?");
pi.sendUserMessage([
  { type: "text", text: "Describe this:" },
  { type: "image", source: { type: "base64", mediaType: "image/png", data: "..." } },
]);

// 持久化扩展状态
pi.appendEntry("my-state", { count: 42 });

// 设置会话名称
pi.setSessionName("Refactoring session");

// 设置条目标签
pi.setLabel(entryId, "checkpoint-before-refactor");
```

### 工具管理

```typescript
// 获取活跃/所有工具
const active = pi.getActiveTools();
const all = pi.getAllTools();

// 设置活跃工具
pi.setActiveTools(["read", "bash", "my_custom_tool"]);
```

### 模型操作

```typescript
// 设置模型
const success = await pi.setModel(model);

// 获取/设置思考级别
const current = pi.getThinkingLevel(); // "off" | "minimal" | "low" | "medium" | "high" | "xhigh"
pi.setThinkingLevel("high");

// 注册自定义 Provider
pi.registerProvider("my-proxy", {
  baseUrl: "https://proxy.example.com",
  apiKey: "PROXY_API_KEY",
  api: "anthropic-messages",
  models: [...],
});

// 注销 Provider
pi.unregisterProvider("my-proxy");
```

### 共享事件总线

```typescript
// 扩展间通信
pi.events.on("custom:event", (data) => {
  console.log("Received:", data);
});

pi.events.emit("custom:event", { foo: "bar" });
```

## ExtensionContext 接口详解

### UI 上下文

```typescript
export interface ExtensionUIContext {
  // 对话框
  select(title: string, options: string[], opts?): Promise<string | undefined>;
  confirm(title: string, message: string, opts?): Promise<boolean>;
  input(title: string, placeholder?: string, opts?): Promise<string | undefined>;
  editor(title: string, prefill?: string): Promise<string | undefined>;

  // 通知
  notify(message: string, type?: "info" | "warning" | "error"): void;

  // 状态栏
  setStatus(key: string, text: string | undefined): void;
  setWorkingMessage(message?: string): void;
  setWorkingIndicator(options?: WorkingIndicatorOptions): void;
  setHiddenThinkingLabel(label?: string): void;

  // 小部件
  setWidget(key: string, content: string[] | undefined, options?): void;
  setFooter(factory: FooterFactory | undefined): void;
  setHeader(factory: HeaderFactory | undefined): void;

  // 主题
  getAllThemes(): { name: string; path: string | undefined }[];
  getTheme(name: string): Theme | undefined;
  setTheme(theme: string | Theme): { success: boolean; error?: string };
  theme: Theme; // 当前主题

  // 编辑器
  setEditorComponent(factory): void;
  setEditorText(text: string): void;
  getEditorText(): string;
  pasteToEditor(content: string): void;

  // 自动完成
  addAutocompleteProvider(callback): void;

  // 工具展开
  getToolsExpanded(): boolean;
  setToolsExpanded(expanded: boolean): void;

  // 自定义组件
  custom<T>(factory, options?): Promise<T>;
}
```

### 会话管理

```typescript
export interface ExtensionContext {
  cwd: string;                                   // 当前工作目录
  hasUI: boolean;                                // UI 是否可用
  sessionManager: ReadonlySessionManager;        // 会话状态只读访问
  modelRegistry: ModelRegistry;                  // 模型注册表
  model: Model<any> | undefined;                 // 当前模型
  isIdle(): boolean;                             // Agent 是否空闲
  hasPendingMessages(): boolean;                // 是否有待处理消息
  signal: AbortSignal | undefined;              // 中止信号
  abort(): void;                                 // 中止当前操作
  shutdown(): void;                              // 请求关闭
  getContextUsage(): ContextUsage | undefined;  // 获取上下文使用情况
  compact(options?: CompactOptions): void;       // 触发压缩
  getSystemPrompt(): string;                    // 获取系统提示词
}
```

### 命令上下文（扩展）

```typescript
export interface ExtensionCommandContext extends ExtensionContext {
  waitForIdle(): Promise<void>;
  newSession(options?): Promise<{ cancelled: boolean }>;
  fork(entryId: string, options?): Promise<{ cancelled: boolean }>;
  navigateTree(targetId: string, options?): Promise<{ cancelled: boolean }>;
  switchSession(sessionPath: string, options?): Promise<{ cancelled: boolean }>;
  reload(): Promise<void>;
}
```

## 事件详细说明

### 会话事件

#### session_start

会话启动时触发。

```typescript
pi.on("session_start", async (event, ctx) => {
  // event.reason: "startup" | "reload" | "new" | "resume" | "fork"
  // event.previousSessionFile: 上一会话文件（new/resume/fork 时存在）
  
  ctx.ui.notify(
    `Session: ${ctx.sessionManager.getSessionFile() ?? "ephemeral"}`, 
    "info"
  );
});
```

#### session_shutdown

会话关闭前触发。

```typescript
pi.on("session_shutdown", async (event, ctx) => {
  // event.reason: "quit" | "reload" | "new" | "resume" | "fork"
  // event.targetSessionFile: 目标会话文件
  // 执行清理工作
});
```

### Agent 事件

#### before_agent_start

Agent 循环前触发，可注入消息或修改系统提示词。

```typescript
pi.on("before_agent_start", async (event, ctx) => {
  // event.prompt: 用户提示词
  // event.images: 附加图片
  // event.systemPrompt: 当前系统提示词
  // event.systemPromptOptions: 系统提示词构建选项
  
  return {
    // 注入消息（存储在会话中，发给 LLM）
    message: {
      customType: "my-extension",
      content: "Additional context for the LLM",
      display: true,
    },
    // 替换系统提示词
    systemPrompt: event.systemPrompt + "\n\nExtra instructions...",
  };
});
```

#### agent_start / agent_end

```typescript
pi.on("agent_start", async (_event, ctx) => {
  // Agent 开始处理用户提示词
});

pi.on("agent_end", async (event, ctx) => {
  // event.messages: 本轮消息
});
```

### 工具事件

#### tool_call（可拦截）

工具调用执行前触发，可用于拦截、修改参数。

```typescript
import { isToolCallEventType } from "@mariozechner/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
  // event.toolName: 工具名称
  // event.toolCallId: 调用 ID
  // event.input: 工具参数（可变）
  
  // 使用类型收窄
  if (isToolCallEventType("bash", event)) {
    event.input.command = `source ~/.profile\n${event.input.command}`;
    
    if (event.input.command.includes("rm -rf")) {
      return { block: true, reason: "Dangerous command" };
    }
  }
  
  return { block: false }; // 或 undefined 表示继续
});
```

#### tool_result（可修改）

工具执行完成后触发，可修改结果。

```typescript
import { isBashToolResult } from "@mariozechner/pi-coding-agent";

pi.on("tool_result", async (event, ctx) => {
  // event.toolName, event.toolCallId, event.input
  // event.content, event.details, event.isError
  
  if (isBashToolResult(event)) {
    // event.details 类型为 BashToolDetails
  }
  
  // 修改结果
  return {
    content: [{ type: "text", text: "Modified" }],
    details: { modified: true },
    isError: false,
  };
});
```

### 上下文事件

#### context

```typescript
pi.on("context", async (event, ctx) => {
  // event.messages: 消息深拷贝，可安全修改
  const filtered = event.messages.filter(m => !shouldPrune(m));
  return { messages: filtered };
});
```

### 输入事件

#### input

```typescript
pi.on("input", async (event, ctx) => {
  // event.text: 用户输入文本
  // event.images: 附加图片
  // event.source: "interactive" | "rpc" | "extension"
  
  if (event.text.startsWith("?quick ")) {
    return { action: "transform", text: `Respond briefly: ${event.text.slice(7)}` };
  }
  
  if (event.text === "ping") {
    ctx.ui.notify("pong", "info");
    return { action: "handled" };
  }
  
  return { action: "continue" }; // 传递给扩展
});
```

## 自定义工具开发

### 工具定义完整结构

```typescript
import { Type } from "typebox";
import { StringEnum } from "@mariozechner/pi-ai";

pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "What this tool does (shown to LLM)",
  
  // 可选：工具提示词片段
  promptSnippet: "List or add items in the project todo list",
  
  // 可选：工具特定指南
  promptGuidelines: [
    "Use my_tool for todo planning instead of direct file edits."
  ],
  
  // 参数 Schema
  parameters: Type.Object({
    action: StringEnum(["list", "add"] as const),
    text: Type.Optional(Type.String()),
  }),
  
  // 可选：参数预处理器（向后兼容）
  prepareArguments(args) {
    return args;
  },
  
  // 可选：自定义外壳渲染
  renderShell: "default" | "self",
  
  // 执行函数
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // 检查取消
    if (signal?.aborted) {
      return { content: [{ type: "text", text: "Cancelled" }] };
    }
    
    // 流式更新
    onUpdate?.({
      content: [{ type: "text", text: "Working..." }],
      details: { progress: 50 },
    });
    
    // 返回结果
    return {
      content: [{ type: "text", text: "Done" }],      // 发给 LLM
      details: { data: result },                      // 渲染和状态使用
      terminate: true,                               // 可选：终止后续 Agent 调用
    };
  },
  
  // 可选：自定义调用渲染
  renderCall(args, theme, context) {
    return new Text(theme.fg("accent", `my_tool ${args.action}`), 0, 0);
  },
  
  // 可选：自定义结果渲染
  renderResult(result, options, theme, context) {
    return new Text(theme.fg("success", "✓ Done"), 0, 0);
  },
});
```

### 覆盖内置工具

扩展可以通过注册同名工具来覆盖内置工具：

```typescript
// 覆盖内置 read 工具
pi.registerTool({
  name: "read",
  // ... 覆盖实现
});
```

或使用 `--no-builtin-tools` 禁用所有内置工具：

```bash
pi --no-builtin-tools -e ./my-extension.ts
```

### 文件修改队列

对于修改文件的工具，使用 `withFileMutationQueue` 避免竞态条件：

```typescript
import { withFileMutationQueue } from "@mariozechner/pi-coding-agent";
import { mkdir, readFile, writeFile } from "node:fs/promises";
import { dirname, resolve } from "node:path";

async execute(_toolCallId, params, _signal, _onUpdate, ctx) {
  const absolutePath = resolve(ctx.cwd, params.path);
  
  return withFileMutationQueue(absolutePath, async () => {
    await mkdir(dirname(absolutePath), { recursive: true });
    const current = await readFile(absolutePath, "utf8");
    const next = current.replace(params.oldText, params.newText);
    await writeFile(absolutePath, next, "utf8");
    
    return {
      content: [{ type: "text", text: `Updated ${params.path}` }],
      details: {},
    };
  });
}
```

### 输出截断

工具输出必须截断以避免溢出上下文：

```typescript
import {
  truncateHead,
  truncateTail,
  formatSize,
  DEFAULT_MAX_BYTES,
  DEFAULT_MAX_LINES,
} from "@mariozechner/pi-coding-agent";

async execute(toolCallId, params, signal, onUpdate, ctx) {
  const output = await runCommand();
  
  const truncation = truncateHead(output, {
    maxLines: DEFAULT_MAX_LINES,
    maxBytes: DEFAULT_MAX_BYTES,
  });
  
  let result = truncation.content;
  
  if (truncation.truncated) {
    const tempFile = writeTempFile(output);
    result += `\n\n[Output truncated: ${truncation.outputLines} of ${truncation.totalLines} lines`;
    result += ` (${formatSize(truncation.outputBytes)} of ${formatSize(truncation.totalBytes)}).`;
    result += ` Full output saved to: ${tempFile}]`;
  }
  
  return { content: [{ type: "text", text: result }] };
}
```

## TUI 交互

### 对话框

```typescript
// 选择
const choice = await ctx.ui.select(
  "Pick one:", 
  ["A", "B", "C"]
);

// 确认
const ok = await ctx.ui.confirm(
  "Delete?", 
  "This cannot be undone"
);

// 输入
const name = await ctx.ui.input(
  "Name:", 
  "placeholder text"
);

// 多行编辑器
const text = await ctx.ui.editor(
  "Edit:", 
  "prefilled text"
);

// 通知
ctx.ui.notify("Done!", "info");
ctx.ui.notify("Warning", "warning");
ctx.ui.notify("Error", "error");
```

### 超时对话框

```typescript
const confirmed = await ctx.ui.confirm(
  "Timed Confirmation",
  "This dialog will auto-cancel in 5 seconds. Confirm?",
  { timeout: 5000 }
);
```

### 状态栏

```typescript
// 设置状态
ctx.ui.setStatus("my-ext", "Processing...");
ctx.ui.setStatus("my-ext", undefined); // 清除

// 工作指示器
ctx.ui.setWorkingMessage("Thinking deeply...");
ctx.ui.setWorkingMessage(); // 恢复默认
ctx.ui.setWorkingVisible(false); // 隐藏

// 自定义动画帧
ctx.ui.setWorkingIndicator({
  frames: [
    ctx.ui.theme.fg("dim", "·"),
    ctx.ui.theme.fg("muted", "•"),
    ctx.ui.theme.fg("accent", "●"),
  ],
  intervalMs: 120,
});
```

### 小部件

```typescript
// 编辑器上方小部件
ctx.ui.setWidget("my-widget", ["Line 1", "Line 2"]);

// 编辑器下方
ctx.ui.setWidget("my-widget", ["Line 1"], { placement: "belowEditor" });

// 自定义组件
ctx.ui.setWidget("my-widget", (tui, theme) => 
  new Text(theme.fg("accent", "Custom"), 0, 0)
);

// 清除
ctx.ui.setWidget("my-widget", undefined);
```

### 自定义组件

```typescript
import { Text, Component } from "@mariozechner/pi-tui";

const result = await ctx.ui.custom<boolean>(
  (tui, theme, keybindings, done) => {
    const text = new Text("Press Enter to confirm, Escape to cancel", 1, 1);
    
    text.onKey = (key) => {
      if (key === "return") done(true);
      if (key === "escape") done(false);
      return true;
    };
    
    return text;
  }
);
```

### 覆盖层组件

```typescript
const result = await ctx.ui.custom<string | null>(
  (tui, theme, keybindings, done) => 
    new MyOverlayComponent({ onClose: done }),
  { overlay: true }
);

// 带定位选项
const result = await ctx.ui.custom<string | null>(
  (tui, theme, keybindings, done) => new MyOverlayComponent({ onClose: done }),
  {
    overlay: true,
    overlayOptions: { 
      anchor: "top-right", 
      width: "50%", 
      margin: 2 
    },
    onHandle: (handle) => { /* handle.setHidden(true/false) */ }
  }
);
```

## 自定义编辑器

```typescript
import { CustomEditor, type ExtensionAPI } from "@mariozechner/pi-coding-agent";

class VimEditor extends CustomEditor {
  private mode: "normal" | "insert" = "insert";

  handleInput(data: string): void {
    if (matchesKey(data, "escape") && this.mode === "insert") {
      this.mode = "normal";
      return;
    }
    if (this.mode === "normal" && data === "i") {
      this.mode = "insert";
      return;
    }
    super.handleInput(data); // 应用内置快捷键和文本编辑
  }
}

export default function (pi: ExtensionAPI) {
  pi.on("session_start", (_event, ctx) => {
    ctx.ui.setEditorComponent((_tui, theme, keybindings) =>
      new VimEditor(theme, keybindings)
    );
  });
}
```

## 状态管理

### 从会话恢复状态

```typescript
export default function (pi: ExtensionAPI) {
  let items: string[] = [];

  // 从会话恢复
  pi.on("session_start", async (_event, ctx) => {
    items = [];
    for (const entry of ctx.sessionManager.getBranch()) {
      if (entry.type === "message" && entry.message.role === "toolResult") {
        if (entry.message.toolName === "my_tool") {
          items = entry.message.details?.items ?? [];
        }
      }
    }
  });

  pi.registerTool({
    name: "my_tool",
    // ...
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      items.push("new item");
      return {
        content: [{ type: "text", text: "Added" }],
        details: { items: [...items] }, // 存储用于恢复
      };
    },
  });
}
```

## 错误处理

- 扩展错误会被记录，Agent 继续运行
- `tool_call` 错误会阻塞工具（fail-safe）
- 工具 `execute` 错误必须通过抛出信号，抛出错误会被捕获，向 LLM 报告 `isError: true`，执行继续

```typescript
// 正确：抛出信号错误
async execute(toolCallId, params) {
  if (!isValid(params.input)) {
    throw new Error(`Invalid input: ${params.input}`);
  }
  return { content: [{ type: "text", text: "OK" }], details: {} };
}
```

## 模式兼容性

| 模式 | UI 方法 | 说明 |
|------|---------|------|
| Interactive | 完整 TUI | 正常操作 |
| RPC | JSON 协议 | 宿主处理 UI |
| JSON | 无操作 | 事件流到 stdout |
| Print | 无操作 | 扩展运行但不能提示 |

使用 `ctx.hasUI` 检查 UI 是否可用：

```typescript
if (ctx.hasUI) {
  const choice = await ctx.ui.select("Pick:", options);
} else {
  // 非交互模式下的替代处理
}
```

## 编写可分发扩展

### 包结构

```
my-extension/
├── package.json
├── src/
│   └── index.ts
├── skills/
│   └── my-skill/
│       └── SKILL.md
└── themes/
    └── my-theme.json
```

### package.json

```json
{
  "name": "my-pi-extension",
  "version": "1.0.0",
  "keywords": ["pi-package"],
  "dependencies": {
    "some-lib": "^1.0.0"
  },
  "peerDependencies": {
    "@mariozechner/pi-coding-agent": "*",
    "typebox": "*"
  },
  "pi": {
    "extensions": ["./dist/index.js"],
    "skills": ["./skills"],
    "themes": ["./themes"]
  }
}
```

### 发布

```bash
# 本地测试
pi -e ./my-extension

# 安装到全局
pi install ./my-extension

# 发布到 npm
npm publish

# 从 npm 安装
pi install npm:my-pi-extension

# 从 git 安装
pi install git:github.com/user/my-extension
```
