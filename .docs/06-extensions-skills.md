# 扩展与技能系统分析

## 概述

PI MONO 的扩展系统包含两个核心机制：
- **Extensions** - TypeScript 模块，提供完整的功能扩展能力
- **Skills** - 轻量级的 Markdown 文件，用于定义可复用的指令模板

Extensions 系统位于 `packages/coding-agent/src/core/extensions/`，Skills 系统位于 `packages/coding-agent/src/core/skills.ts`。

## Extensions 系统

### 核心文件结构

| 文件 | 职责 |
|------|------|
| `types.ts` | 类型定义 (~1500 行)，包含 ExtensionAPI、事件类型、工具定义 |
| `loader.ts` | 扩展加载器，使用 jiti 动态加载 TypeScript 模块 |
| `runner.ts` | 扩展运行器，管理生命周期和事件派发 |
| `wrapper.ts` | 工具包装器 |
| `index.ts` | 公共导出 |

### ExtensionFactory 模式

扩展模块导出工厂函数而非类实例：

```typescript
// packages/coding-agent/examples/extensions/hello.ts
import { Type } from "@mariozechner/pi-ai";
import { defineTool, type ExtensionAPI } from "@mariozechner/pi-coding-agent";

const helloTool = defineTool({
  name: "hello",
  label: "Hello",
  description: "A simple greeting tool",
  parameters: Type.Object({
    name: Type.String({ description: "Name to greet" }),
  }),
  async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
    return {
      content: [{ type: "text", text: `Hello, ${params.name}!` }],
      details: { greeted: params.name },
    };
  },
});

export default function (pi: ExtensionAPI) {
  pi.registerTool(helloTool);
}
```

**参考源码**: `packages/coding-agent/examples/extensions/hello.ts:1-26`

### ExtensionAPI 接口

扩展通过 `ExtensionAPI` 对象注册工具、命令、快捷键和订阅事件：

```typescript
// packages/coding-agent/src/core/extensions/types.ts:1035-1261
export interface ExtensionAPI {
  // 事件订阅
  on(event: "session_start", handler: ExtensionHandler<SessionStartEvent>): void;
  on(event: "context", handler: ExtensionHandler<ContextEvent, ContextEventResult>): void;
  on(event: "tool_call", handler: ExtensionHandler<ToolCallEvent, ToolCallEventResult>): void;
  on(event: "tool_result", handler: ExtensionHandler<ToolResultEvent, ToolResultEventResult>): void;
  // ... 其他事件类型

  // 工具注册
  registerTool<TParams extends TSchema, TDetails, TState>(tool: ToolDefinition<TParams, TDetails, TState>): void;

  // 命令注册
  registerCommand(name: string, options: Omit<RegisteredCommand, "name" | "sourceInfo">): void;

  // 快捷键注册
  registerShortcut(shortcut: KeyId, options: { description?: string; handler }): void;

  // CLI 标志注册
  registerFlag(name: string, options: { description?: string; type; default? }): void;
  getFlag(name: string): boolean | string | undefined;

  // 消息渲染
  registerMessageRenderer<T>(customType: string, renderer: MessageRenderer<T>): void;

  // 消息发送
  sendMessage(message, options?): void;
  sendUserMessage(content, options?): void;
  appendEntry(customType: string, data?): void;

  // 会话控制
  setSessionName(name: string): void;
  setLabel(entryId: string, label: string | undefined): void;

  // 工具控制
  getActiveTools(): string[];
  getAllTools(): ToolInfo[];
  setActiveTools(toolNames: string[]): void;

  // 模型注册
  setModel(model: Model<any>): Promise<boolean>;
  getThinkingLevel(): ThinkingLevel;
  setThinkingLevel(level: ThinkingLevel): void;
  registerProvider(name: string, config: ProviderConfig): void;
  unregisterProvider(name: string): void;

  // 共享事件总线
  events: EventBus;
}
```

**参考源码**: `packages/coding-agent/src/core/extensions/types.ts:1035-1261`

### 事件系统

扩展可以订阅以下事件类型：

```typescript
// packages/coding-agent/src/core/extensions/types.ts:907-928
export type ExtensionEvent =
  | ResourcesDiscoverEvent      // 资源发现
  | SessionEvent              // 会话事件
  | ContextEvent              // 上下文事件
  | BeforeProviderRequestEvent // 提供商请求前
  | AfterProviderResponseEvent // 提供商响应后
  | BeforeAgentStartEvent     // Agent 启动前
  | AgentStartEvent          // Agent 启动
  | AgentEndEvent           // Agent 结束
  | TurnStartEvent          // 轮次开始
  | TurnEndEvent           // 轮次结束
  | MessageStartEvent      // 消息开始
  | MessageUpdateEvent  // 消息更新
  | MessageEndEvent    // 消息结束
  | ToolExecutionStartEvent  // 工具执行开始
  | ToolExecutionUpdateEvent // 工具执行更新
  | ToolExecutionEndEvent     // 工具执行结束
  | ModelSelectEvent         // 模型选择
  | UserBashEvent         // 用户 Bash
  | InputEvent           // 输入事件
  | ToolCallEvent        // 工具调用
  | ToolResultEvent;   // 工具结果
```

**关键事件**:
- `session_start` / `session_shutdown` - 会话生命周期
- `before_agent_start` - Agent 循环前修改提示词
- `context` - 修改聊天上下文消息
- `tool_call` / `tool_result` - 拦截或修改工具执行

**参考源码**: `packages/coding-agent/src/core/extensions/types.ts:907-928`

### ExtensionContext

事件处理器接收 `ExtensionContext`:

```typescript
// packages/coding-agent/src/core/extensions/types.ts:286-315
export interface ExtensionContext {
  ui: ExtensionUIContext;           // UI 交互
  hasUI: boolean;                  // UI 是否可用
  cwd: string;                     // 当前目录
  sessionManager: ReadonlySessionManager;
  modelRegistry: ModelRegistry;
  model: Model<any> | undefined;
  isIdle(): boolean;
  signal: AbortSignal | undefined;
  abort(): void;
  hasPendingMessages(): boolean;
  shutdown(): void;
  getContextUsage(): ContextUsage | undefined;
  compact(options?: CompactOptions): void;
  getSystemPrompt(): string;
}

export interface ExtensionCommandContext extends ExtensionContext {
  waitForIdle(): Promise<void>;
  newSession(options?): Promise<{ cancelled: boolean }>;
  fork(entryId: string, options?): Promise<{ cancelled: boolean }>;
  navigateTree(targetId: string, options?): Promise<{ cancelled: boolean }>;
  switchSession(sessionPath: string): Promise<{ cancelled: boolean }>;
  reload(): Promise<void>;
}
```

**参考源码**: `packages/coding-agent/src/core/extensions/types.ts:286-345`

### 扩展 UI 上下文

交互模式下提供完整 UI 功能：

```typescript
// packages/coding-agent/src/core/extensions/types.ts:119-263
export interface ExtensionUIContext {
  // 对话框
  select(title: string, options: string[], opts?): Promise<string | undefined>;
  confirm(title: string, message: string, opts?): Promise<boolean>;
  input(title: string, placeholder?: string, opts?): Promise<string | undefined>;

  // 通知
  notify(message: string, type?: "info" | "warning" | "error"): void;

  // 终端输入监听
  onTerminalInput(handler: TerminalInputHandler): () => void;

  // 状态栏
  setStatus(key: string, text: string | undefined): void;
  setWorkingMessage(message?: string): void;
  setWorkingIndicator(options?: WorkingIndicatorOptions): void;
  setHiddenThinkingLabel(label?: string): void;

  // 组件
  setWidget(key: string, content: string[] | undefined, options?): void;
  setFooter(factory: FooterFactory | undefined): void;
  setHeader(factory: HeaderFactory | undefined): void;

  // 主题
  getAllThemes(): { name: string; path: string | undefined }[];
  getTheme(name: string): Theme | undefined;
  setTheme(theme: string | Theme): { success: boolean; error?: string };

  // 编辑器
  setEditorComponent(factory): void;
  setEditorText(text: string): void;
  getEditorText(): string;
  editor(title: string, prefill?: string): Promise<string | undefined>;

  // 覆盖层
  custom<T>(factory, options?): Promise<T>;
}
```

**参考源码**: `packages/coding-agent/src/core/extensions/types.ts:119-263`

### 工具定义

```typescript
// packages/coding-agent/src/core/extensions/types.ts:390-452
export interface ToolDefinition<TParams extends TSchema, TDetails, TState> {
  name: string;
  label: string;
  description: string;
  promptSnippet?: string;
  promptGuidelines?: string[];
  parameters: TParams;
  renderShell?: "default" | "self";
  prepareArguments?: (args: unknown) => Static<TParams>;
  executionMode?: ToolExecutionMode;

  execute(
    toolCallId: string,
    params: Static<TParams>,
    signal: AbortSignal | undefined,
    onUpdate: AgentToolUpdateCallback<TDetails> | undefined,
    ctx: ExtensionContext,
  ): Promise<AgentToolResult<TDetails>>;

  renderCall?: (args, theme, context) => Component;
  renderResult?: (result, options, theme, context) => Component;
}

export function defineTool<TParams, TDetails, TState>(
  tool: ToolDefinition<TParams, TDetails, TState>,
): ToolDefinition<TParams, TDetails, TState> {
  return tool;
}
```

**参考源码**: `packages/coding-agent/src/core/extensions/types.ts:390-452`

### 扩展加载机制

**加载流程** (`packages/coding-agent/src/core/extensions/loader.ts`):

1. **路径发现**
   - 项目本地: `cwd/.pi/extensions/`
   - 全局: `agentDir/extensions/`
   - 显式配置路径

2. **发现规则**
   ```
   - 直接文件: *.ts 或 *.js → 加载
   - 子目录 with index: */index.ts 或 */index.js → 加载
   - 子目录 with package.json: */package.json 有 "pi.extensions" → 按声明加载
   ```

3. **模块加载**
   - 使用 jiti 动态加载 TypeScript
   - Bun 模式下使用 virtualModules 解析bundled 包
   - Node.js 模式下使用别名解析

```typescript
// packages/coding-agent/src/core/extensions/loader.ts:292-304
async function loadExtensionModule(extensionPath: string) {
  const jiti = createJiti(import.meta.url, {
    moduleCache: false,
    ...(isBunBinary
      ? { virtualModules: VIRTUAL_MODULES, tryNative: false }
      : { alias: getAliases() }),
  });
  const module = await jiti.import(extensionPath, { default: true });
  const factory = module as ExtensionFactory;
  return typeof factory !== "function" ? undefined : factory;
}
```

4. **工厂执行**
   - 创建 Extension 对象
   - 创建 ExtensionAPI
   - 执行工厂函数，注册工具/命令/处理器
   - 返回加载结果

**参考源码**: `packages/coding-agent/src/core/extensions/loader.ts:292-557`

### 扩展运行器

`ExtensionRunner` 管理扩展生命周期和事件派发：

```typescript
// packages/coding-agent/src/core/extensions/runner.ts:213-928
export class ExtensionRunner {
  constructor(
    extensions: Extension[],
    runtime: ExtensionRuntime,
    cwd: string,
    sessionManager: SessionManager,
    modelRegistry: ModelRegistry,
  ) { }

  bindCore(actions, contextActions, providerActions?): void;
  bindCommandContext(actions?): void;
  setUIContext(uiContext?): void;

  // 获取注册内容
  getAllRegisteredTools(): RegisteredTool[];
  getToolDefinition(toolName: string): RegisteredTool["definition"] | undefined;
  getFlags(): Map<string, ExtensionFlag>;
  getShortcuts(resolvedKeybindings): Map<KeyId, ExtensionShortcut>;
  getRegisteredCommands(): ResolvedCommand[];

  // 事件派发
  async emit<TEvent>(event: TEvent): Promise<RunnerEmitResult<TEvent>>;
  async emitToolCall(event: ToolCallEvent): Promise<ToolCallEventResult | undefined>;
  async emitToolResult(event: ToolResultEvent): Promise<ToolResultEventResult | undefined>;
  async emitContext(messages: AgentMessage[]): Promise<AgentMessage[]>;
  async emitBeforeAgentStart(...): Promise<BeforeAgentStartCombinedResult | undefined>;
  async emitInput(text, images, source): Promise<InputEventResult>;

  // 创建上下文
  createContext(): ExtensionContext;
  createCommandContext(): ExtensionCommandContext;
}
```

**关键方法**:
- `bindCore()` - 绑定核心动作，填充运行时
- `emit()` - 派发事件到所有处理器
- `createContext()` - 为事件处理器创建上下文

**参考源码**: `packages/coding-agent/src/core/extensions/runner.ts:213-928`

### Provider 注册

扩展可以注册或覆盖模型提供商：

```typescript
// packages/coding-agent/src/core/extensions/types.ts:1268-1296
export interface ProviderConfig {
  baseUrl?: string;
  apiKey?: string;
  api?: Api;
  streamSimple?: (model, context, options?) => AssistantMessageEventStream;
  headers?: Record<string, string>;
  authHeader?: boolean;
  models?: ProviderModelConfig[];
  oauth?: {
    name: string;
    login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials>;
    refreshToken(credentials): Promise<OAuthCredentials>;
    getApiKey(credentials): string;
    modifyModels?(models, credentials): Model<Api>[];
  };
}

// 使用示例
pi.registerProvider("my-proxy", {
  baseUrl: "https://proxy.example.com",
  apiKey: "PROXY_API_KEY",
  api: "anthropic-messages",
  models: [
    {
      id: "claude-sonnet-4-20250514",
      name: "Claude 4 Sonnet (proxy)",
      reasoning: false,
      input: ["text", "image"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: 200000,
      maxTokens: 16384
    }
  ]
});
```

**参考源码**: `packages/coding-agent/src/core/extensions/types.ts:1268-1296`

## Skills 系统

### 文件位置

`packages/coding-agent/src/core/skills.ts` (~508 行)

### Skill 定义

```typescript
// packages/coding-agent/src/core/skills.ts:67-86
export interface SkillFrontmatter {
  name?: string;
  description?: string;
  "disable-model-invocation"?: boolean;
}

export interface Skill {
  name: string;
  description: string;
  filePath: string;
  baseDir: string;
  sourceInfo: SourceInfo;
  disableModelInvocation: boolean;
}
```

### SKILL.md 格式

```markdown
---
name: web-fetch
description: 使用 curl 或 fetch 获取网页内容
disable-model-invocation: false
---

# Web Fetch Skill

此技能用于获取网页内容。

## 使用方法

使用 read 工具读取 URL，或使用 bash 执行 curl 命令。

## 示例

<skill name="web-fetch" location="skills/web-fetch">
https://example.com
</skill>
```

**目录结构**:
```
skills/
├── web-fetch/
│   └── SKILL.md
└── my-skill/
    └── SKILL.md
```

或直接在 skills 目录放置 Markdown 文件（文件名作为技能名）。

### 验证规则

```typescript
// packages/coding-agent/src/core/skills.ts:92-116
function validateName(name: string, parentDirName: string): string[] {
  const errors: string[] = [];

  // 名称必须匹配父目录
  if (name !== parentDirName) {
    errors.push(`name "${name}" does not match parent directory "${parentDirName}"`);
  }

  // 名称长度限制
  if (name.length > MAX_NAME_LENGTH) {  // 64 字符
    errors.push(`name exceeds ${MAX_NAME_LENGTH} characters`);
  }

  // 字符限制
  if (!/^[a-z0-9-]+$/.test(name)) {
    errors.push(`name contains invalid characters (must be lowercase a-z, 0-9, hyphens only)`);
  }

  // 不能以连字符开头或结尾
  if (name.startsWith("-") || name.endsWith("-")) {
    errors.push(`name must not start or end with a hyphen`);
  }

  return errors;
}
```

**参考源码**: `packages/coding-agent/src/core/skills.ts:92-116`

### 技能加载位置

```
加载优先级（后者覆盖前者）:
1. agentDir/skills/        (用户全局技能)
2. cwd/.pi/skills/        (项目技能)
3. 显式 skillPaths 配置
```

优先级规则：`project > user > package`

### 格式化提示词

```typescript
// packages/coding-agent/src/core/skills.ts:339-374
export function formatSkillsForPrompt(skills: Skill[]): string {
  const visibleSkills = skills.filter((s) => !s.disableModelInvocation);

  if (visibleSkills.length === 0) return "";

  const lines = [
    "\n\nThe following skills provide specialized instructions for specific tasks.",
    "Use the read tool to load a skill's file when the task matches its description.",
    "",
    "<available_skills>",
  ];

  for (const skill of visibleSkills) {
    lines.push("  <skill>");
    lines.push(`    <name>${escapeXml(skill.name)}</name>`);
    lines.push(`    <description>${escapeXml(skill.description)}</description>`);
    lines.push(`    <location>${escapeXml(skill.filePath)}</location>`);
    lines.push("  </skill>");
  }

  lines.push("</available_skills>");
  return lines.join("\n");
}
```

输出格式（Agent Skills 标准）:
```xml
<available_skills>
  <skill>
    <name>web-fetch</name>
    <description>获取网页内容</description>
    <location>/path/to/skills/web-fetch/SKILL.md</location>
  </skill>
</available_skills>
```

**参考源码**: `packages/coding-agent/src/core/skills.ts:339-374`

### 使用 Skill

通过提示词展开：
```
使用 web-fetch skill 获取 https://example.com
```

LLM 将读取 SKILL.md 文件并根据其中指令执行任务。

## Pi Packages 分发机制

扩展和技能可以通过 npm 包分发：

```json
// package.json
{
  "name": "my-pi-extension",
  "version": "1.0.0",
  "pi": {
    "extensions": ["dist/index.js"],
    "skills": ["skills/*"],
    "themes": ["themes/*"]
  }
}
```

加载器会读取 `package.json` 的 `pi` 字段发现扩展、技能和主题。

**参考源码**: `packages/coding-agent/src/core/extensions/loader.ts:399-417`

## 最佳实践

### Extension 编写

1. **使用 defineTool() 定义工具** - 保留类型推断

2. **事件处理器返回类型化结果**:
   ```typescript
   pi.on("context", async (event, ctx) => {
     // 修改消息
     return { messages: event.messages };
   });

   pi.on("tool_result", async (event, ctx) => {
     // 修改结果内容
     return { content: modifiedContent };
   });
   ```

3. **Provider 注册使用延迟加载** - 在初始化阶段注册会被队列化，在 `bindCore()` 后生效

4. **使用 hasUI 检查** - 非交互模式下部分功能不可用

### Skill 编写

1. **名称规范** - 小写字母、数字、连字符，匹配目录名
2. **描述清晰** - LLM 根据描述选择技能
3. **包含使用说明** - SKILL.md 应说明何时、如何使用
4. **使用相对路径** - 技能文件中引用相对路径时应说明基于技能目录

## 总结

| 特性 | Extensions | Skills |
|------|-----------|----------|
| 实现方式 | TypeScript 模块 | Markdown 文件 |
| 功能强度 | 完整功能 | 指令模板 |
| 工具注册 | 支持 | 不支持 |
| 命令注册 | 支持 | 不支持 |
| 事件订阅 | 支持 | 不支持 |
| UI 交互 | 支持 | 不支持 |
| 使用方式 | /command 或工具调用 | 提示词展开 |
| 分发方式 | npm 包 | npm 包 |

Extensions 系统提供了完整的可扩展性，适合需要深度定制的场景。Skills 系统是轻量级方案，适合定义可复用的指令模板。

**参考源码位置**:
- Extension 核心: `packages/coding-agent/src/core/extensions/`
- Extension 示例: `packages/coding-agent/examples/extensions/`
- Skill 实现: `packages/coding-agent/src/core/skills.ts`