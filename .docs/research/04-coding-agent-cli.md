# pi-coding-agent CLI 完整流程

## 概述

pi-coding-agent 是完整的 CLI 应用，包含：
- 参数解析和模式选择
- 会话管理 (创建/继续/分支/压缩)
- AgentSession 运行时
- 工具系统
- 交互式 TUI

## 文件结构

```
packages/coding-agent/
├── src/
│   ├── main.ts                      # CLI 入口 (~736 行)
│   ├── config.ts                    # 配置
│   ├── cli/
│   │   ├── args.ts                 # 参数解析
│   │   ├── file-processor.ts        # 文件参数处理
│   │   ├── initial-message.ts      # 初始消息构建
│   │   ├── list-models.ts          # 模型列表
│   │   └── session-picker.ts      # 会话选择
│   ├── core/
│   │   ├── agent-session.ts        # AgentSession (~3077 行)
│   │   ├── agent-session-runtime.ts
│   │   ├── agent-session-services.ts
│   │   ├── session-manager.ts      # 会话管理 (~1425 行)
│   │   ├── settings-manager.ts    # 设置管理
│   │   ├── auth-storage.ts       # 认证存储
│   │   ├── tools/               # 工具系统
│   │   ├── extensions/           # 扩展系统
│   │   ├── skills.ts            # Skills
│   │   ├── model-registry.ts    # 模型注册
│   │   ├── system-prompt.ts     # 系统提示
│   │   ├── compaction/          # 会话压缩
│   │   └── ...
│   └── modes/
│       ├── interactive/          # 交互模式
│       ├── print/               # 打印模式
│       └── rpc/                # RPC 模式
```

## CLI 入口流程

### 文件位置

`packages/coding-agent/src/main.ts`

### 主函数

```typescript
export async function main(args: string[], options?: MainOptions) {
  // 1. 准备
  resetTimings();
  const offlineMode = args.includes("--offline");
  
  // 2. 处理特殊命令 (package, config)
  if (await handlePackageCommand(args)) return;
  if (await handleConfigCommand(args)) return;
  
  // 3. 解析参数
  const parsed = parseArgs(args);
  if (parsed.diagnostics.length > 0) {
    // 处理诊断
  }
  
  // 4. 确定模式
  let appMode = resolveAppMode(parsed, process.stdin.isTTY);
  // - interactive: 交互式 TUI
  // - print: 批量打印
  // - json: JSON 输出
  // - rpc: RPC 模式
  
  // 5. 运行迁移
  const { migratedAuthProviders, deprecationWarnings } = runMigrations(process.cwd());
  
  // 6. 创建会话管理器
  const sessionManager = await createSessionManager(parsed, cwd, sessionDir, settingsManager);
  
  // 7. 创建运行时
  const runtime = await createAgentSessionRuntime(createRuntime, {
    cwd: sessionManager.getCwd(),
    agentDir,
    sessionManager,
  });
  
  // 8. 根据模式运行
  if (appMode === "rpc") {
    await runRpcMode(runtime);
  } else if (appMode === "interactive") {
    const interactiveMode = new InteractiveMode(runtime, { /* options */ });
    await interactiveMode.run();
  } else {
    await runPrintMode(runtime, { /* options */ });
  }
}
```

## 参数解析

### 文件位置

`packages/coding-agent/src/cli/args.ts`

### 参数定义

```typescript
interface Args {
  // 模式
  mode: "interactive" | "print" | "json" | "rpc";
  
  // 会话选项
  session?: string;       // --session <id>
  resume?: boolean;      // --resume
  continue?: boolean;  // --continue
  noSession?: boolean; // --no-session
  fork?: string;       // --fork <session>
  
  // 模型选择
  model?: string;      // --model <pattern>
  provider?: string;  // --provider <name>
  models?: string[];  // --models <patterns>
  thinking?: ThinkingLevel;  // --thinking <level>
  
  // 工具选项
  tools?: string[];    // --tools <names>
  noTools?: boolean;  // --no-tools
  
  // 扩展选项
  extensions?: string[];   // --extension <path>
  noExtensions?: boolean; // --no-extensions
  
  // Skills 选项
  skills?: string[];    // --skill <path>
  noSkills?: boolean;  // --no-skills
  
  // 其他
  verbose?: boolean;
  print?: boolean;
  help?: boolean;
  version?: boolean;
  
  // 文件参数
  fileArgs: string[];
}
```

### 解析函数

```typescript
function parseArgs(args: string[]): Args {
  const parsed: Args = {
    mode: "interactive",
    // 默认值
  };
  
  // 解析每个参数
  for (let i = 0; i < args.length; i++) {
    const arg = args[i];
    
    switch (arg) {
      case "--model":
        parsed.model = args[++i];
        break;
      case "--provider":
        parsed.provider = args[++i];
        break;
      case "--session":
        parsed.session = args[++i];
        break;
      case "--resume":
        parsed.resume = true;
        break;
      case "--continue":
        parsed.continue = true;
        break;
      // ... 更多参数
    }
  }
  
  return parsed;
}
```

## 会话管理

### 文件位置

`packages/coding-agent/src/core/session-manager.ts`

### 会话头部

```typescript
export const CURRENT_SESSION_VERSION = 3;

export interface SessionHeader {
  type: "session";
  version?: number;        // v1 没有此字段
  id: string;
  timestamp: string;
  cwd: string;
  parentSession?: string;
}
```

### 会话条目

```typescript
export type SessionEntry =
  | SessionMessageEntry       // 消息
  | ThinkingLevelChangeEntry  // 思考级别变更
  | ModelChangeEntry        // 模型变更
  | CompactionEntry        // 压缩条目
  | BranchSummaryEntry     // 分支摘要
  | CustomEntry         // 自定义条目
  | CustomMessageEntry  // 自定义消息
  | LabelEntry         // 标签
  | SessionInfoEntry; // 会话信息
```

### 会话消息条目

```typescript
export interface SessionMessageEntry extends SessionEntryBase {
  type: "message";
  message: AgentMessage;
}

export interface SessionEntryBase {
  type: string;
  id: string;
  parentId: string | null;
  timestamp: string;
}
```

### 会话压缩条目

```typescript
export interface CompactionEntry<T = unknown> extends SessionEntryBase {
  type: "compaction";
  summary: string;
  firstKeptEntryId: string;
  tokensBefore: number;
  details?: T;
  fromHook?: boolean;
}
```

### 分支摘要条目

```typescript
export interface BranchSummaryEntry<T = unknown> extends SessionEntryBase {
  type: "branch_summary";
  fromId: string;
  summary: string;
  details?: T;
  fromHook?: boolean;
}
```

### SessionManager 类

```typescript
export class SessionManager {
  private file: string;
  private cwd: string;
  private header: SessionHeader;
  
  constructor(file: string, cwd: string, sessionDir?: string) { ... }
  
  // 获取信息
  getSessionId(): string;
  getSessionFile(): string;
  getCwd(): string;
  
  // 添加消息
  addMessage(message: AgentMessage): string;
  
  // 获取消息
  getMessage(id: string): AgentMessage;
  getMessages(): AgentMessage[];
  
  // 树形结构
  getTree(): SessionTreeNode;
  getBranch(): SessionTreeNode[];
  
  // 压缩
  compact(entries: CompactionEntry[]): void;
  
  // 分支
  branch(name: string): SessionManager;
  
  // 列表
  static list(cwd: string, sessionDir?: string): Promise<SessionInfo[]>;
  static listAll(): Promise<SessionInfo[]>;
}
```

### 会话文件格式 (JSONL)

```jsonl
{"type":"session","id":"abc123","timestamp":"2024-01-01T00:00:00Z","cwd":"/path/to/project"}
{"type":"message","id":"msg1","parentId":"abc123","timestamp":"...","message":{"role":"user","content":"Hello"}}
{"type":"message","id":"msg2","parentId":"msg1","timestamp":"...","message":{"role":"assistant","content":"Hi","content":[{"type":"text","text":"Hi"}]}}
{"type":"compaction","id":"comp1","parentId":"msg2","summary":"...","firstKeptEntryId":"msg2","tokensBefore":5000}
```

## 会话管理操作

### 创建会话

```typescript
// main.ts 中
async function createSessionManager(parsed, cwd, sessionDir, settingsManager) {
  if (parsed.noSession) {
    return SessionManager.inMemory();
  }
  
  if (parsed.fork) {
    // Fork 现有会话
    const resolved = await resolveSessionPath(parsed.fork, cwd, sessionDir);
    return SessionManager.forkFrom(resolved.path, cwd, sessionDir);
  }
  
  if (parsed.session) {
    // 打开指定会话
    const resolved = await resolveSessionPath(parsed.session, cwd, sessionDir);
    return SessionManager.open(resolved.path, sessionDir);
  }
  
  if (parsed.resume) {
    // 交互选择
    const selectedPath = await selectSession(...);
    return SessionManager.open(selectedPath, sessionDir);
  }
  
  if (parsed.continue) {
    // 继续最近的会话
    return SessionManager.continueRecent(cwd, sessionDir);
  }
  
  // 创建新会话
  return SessionManager.create(cwd, sessionDir);
}
```

### 会话选项

```typescript
interface NewSessionOptions {
  id?: string;
  parentSession?: string;
}

// 创建
SessionManager.create(cwd: string, sessionDir?: string): SessionManager;

// 打开
SessionManager.open(path: string, sessionDir?: string): SessionManager;

// Fork
SessionManager.forkFrom(sourcePath: string, cwd: string, sessionDir?: string): SessionManager;

// 继续
SessionManager.continueRecent(cwd: string, sessionDir?: string): SessionManager;

// 内存会话
SessionManager.inMemory(): SessionManager;
```

## AgentSession

### 文件位置

`packages/coding-agent/src/core/agent-session.ts` (~3077 行)

### 配置接口

```typescript
export interface AgentSessionConfig {
  agent: Agent;
  sessionManager: SessionManager;
  settingsManager: SettingsManager;
  cwd: string;
  scopedModels?: Array<{ model: Model<any>; thinkingLevel?: ThinkingLevel }>;
  resourceLoader: ResourceLoader;
  customTools?: ToolDefinition[];
  modelRegistry: ModelRegistry;
  initialActiveToolNames?: string[];
  baseToolsOverride?: Record<string, AgentTool>;
  extensionRunnerRef?: { current?: ExtensionRunner };
  sessionStartEvent?: SessionStartEvent;
}
```

### AgentSession 类

```typescript
export class AgentSession {
  private config: AgentSessionConfig;
  public readonly state: AgentState;
  
  constructor(config: AgentSessionConfig) { ... }
  
  // 主 API
  async prompt(input: string | AgentMessage, options?: PromptOptions): Promise<void>;
  async continue(): Promise<void>;
  async compact(reason: "manual" | "threshold" | "overflow"): Promise<CompactionResult>;
  async branch(name: string): Promise<void>;
  
  // 会话管理
  get model(): Model<any>;
  setModel(model: Model<any>): void;
  get thinkingLevel(): ThinkingLevel;
  setThinkingLevel(level: ThinkingLevel): void;
  
  // 模型切换
  cycleModel(): ModelCycleResult;
  
  // 统计
  getStats(): SessionStats;
  
  // 事件
  subscribe(listener: AgentSessionEventListener): () => void;
}
```

### Prompt 选项

```typescript
export interface PromptOptions {
  expandPromptTemplates?: boolean;
  images?: ImageContent[];
  streamingBehavior?: "steer" | "followUp";
  source?: InputSource;
}
```

## 工具系统

### 文件位置

`packages/coding-agent/src/core/tools/`

### 工具列表

```typescript
export const allTools = {
  read: readTool,
  bash: bashTool,
  edit: editTool,
  write: writeTool,
  grep: grepTool,
  find: findTool,
  ls: lsTool,
};

export const codingTools = [readTool, bashTool, editTool, writeTool];
export const readOnlyTools = [readTool, grepTool, findTool, lsTool];
```

### 工具类型

```typescript
export interface ToolDefinition<TInput = any, TDetails = any> {
  name: string;
  description: string;
  parameters: TSchema;  // JSON Schema
  
  render?: (details: TDetails, options: ToolRenderResultOptions) => ToolRenderResult;
}

export interface Tool<TInput = any, TDetails = any> {
  name: string;
  description: string;
  parameters: TSchema;
  
  execute(args: TInput, context: ToolExecutionContext): Promise<ToolResult<TDetails>>;
}
```

### Read 工具

```typescript
interface ReadToolInput {
  path: string;
  offset?: number;
  limit?: number;
}

interface ReadToolDetails {
  content: string;
  truncated: boolean;
  truncation?: TruncationResult;
}
```

### Bash 工具

```typescript
interface BashToolInput {
  command: string;
  timeout?: number;
}

interface BashToolDetails {
  exitCode: number | null;
  output: string;
  truncation?: TruncationResult;
}
```

### Edit 工具

```typescript
interface EditToolInput {
  path: string;
  from: string;
  to: string;
}

interface EditToolDetails {
  diff: string;
  success: boolean;
}
```

### Write 工具

```typescript
interface WriteToolInput {
  path: string;
  content: string;
}

interface WriteToolDetails {
  content: string;
}
```

## 工具接口实现

### BashOperations

```typescript
export interface BashOperations {
  exec: (
    command: string,
    cwd: string,
    options: {
      onData: (data: Buffer) => void;
      signal?: AbortSignal;
      timeout?: number;
      env?: NodeJS.ProcessEnv;
    },
  ) => Promise<{ exitCode: number | null }>;
}
```

### 本地执行

```typescript
export function createLocalBashOperations(): BashOperations {
  return {
    exec: (command, cwd, { onData, signal, timeout, env }) => {
      return new Promise((resolve, reject) => {
        const { shell, args } = getShellConfig();
        const child = spawn(shell, [...args, command], {
          cwd,
          detached: true,
          env: env ?? getShellEnv(),
          stdio: ["ignore", "pipe", "pipe"],
        });
        
        // 跟踪后台进程
        if (child.pid) trackDetachedChildPid(child.pid);
        
        // 超时处理
        if (timeout !== undefined && timeout > 0) {
          timeoutHandle = setTimeout(() => {
            timedOut = true;
            if (child.pid) killProcessTree(child.pid);
          }, timeout * 1000);
        }
        
        // 流式输出
        child.stdout?.on("data", onData);
        child.stderr?.on("data", onData);
        
        // 中止处理
        if (signal) {
          signal.addEventListener("abort", () => {
            if (child.pid) killProcessTree(child.pid);
          }, { once: true });
        }
        
        // 等待完成
        waitForChildProcess(child).then((code) => {
          // 清理
          if (child.pid) untrackDetachedChildPid(child.pid);
          if (timeoutHandle) clearTimeout(timeoutHandle);
          
          if (signal?.aborted) {
            reject(new Error("aborted"));
          } else if (timedOut) {
            reject(new Error(`timeout:${timeout}`));
          } else {
            resolve({ exitCode: code });
          }
        });
      });
    },
  };
}
```

## 交互式模式

### 文件位置

`packages/coding-agent/src/modes/interactive/interactive-mode.ts`

### InteractiveMode 类

```typescript
export class InteractiveMode {
  private runtime: AgentSessionRuntime;
  private tui: TUI;
  private running: boolean = false;
  
  constructor(runtime: AgentSessionRuntime, options: InteractiveModeOptions) { ... }
  
  async run(): Promise<void> {
    await this.init();
    
    while (this.running) {
      // 渲染 UI
      this.render();
      
      // 读取输入
      const input = await this.readInput();
      
      // 处理命令
      await this.processInput(input);
    }
  }
  
  async init(): Promise<void> { ... }
  private render(): void { ... }
  private async readInput(): Promise<string> { ... }
  private async processInput(input: string): Promise<void> { ... }
}
```

### 渲染组件

```typescript
// 消息历史渲染
renderMessages(): string[] { ... }

// 编辑器渲染
renderEditor(): string[] { ... }

// 状态栏渲染
renderFooter(): string[] { ... }

// 标题栏渲染
renderHeader(): string[] { ... }
```

### 键盘处理

```typescript
handleInput(data: string): void {
  switch (data) {
    case ctrlC:
      // 中止
      this.abort();
      break;
    case ctrlD:
      // 退出
      this.exit();
      break;
    case ctrlP:
      // 切换模型
      this.cycleModel();
      break;
    case ctrlG:
      // 中止
      this.abort();
      break;
    case "q":
      // 退出
      break;
    // ...
  }
}
```

## 打印模式

### 文件位置

`packages/coding-agent/src/modes/print-mode.ts`

### 模式

```typescript
enum PrintOutputMode {
  Text,
  JSON,
}

async function runPrintMode(
  runtime: AgentSessionRuntime,
  options: {
    mode: PrintOutputMode;
    messages: string[];
    initialMessage?: string;
    initialImages?: ImageContent[];
  },
): Promise<number> { ... }
```

## RPC 模式

### 文件位置

`packages/coding-agent/src/modes/rpc/rpc-mode.ts`

### 接口

```typescript
interface JsonRpcRequest {
  jsonrpc: "2.0";
  id: string | number;
  method: string;
  params?: any;
}

interface JsonRpcResponse {
  jsonrpc: "2.0";
  id: string | number;
  result?: any;
  error?: {
    code: number;
    message: string;
  };
}
```

### 方法

```typescript
// prompt - 发送提示
// continue - 继续
// abort - 中止
// getState - 获取状态
// getMessages - 获取消息
```

## 系统提示

### 文件位置

`packages/coding-agent/src/core/system-prompt.ts`

### 系统提示构成

```typescript
function buildSystemPrompt(
  config: SystemPromptConfig,
): string {
  const parts: string[] = [];
  
  // 1. 通用指令
  parts.push(GENERAL_INSTRUCTIONS);
  
  // 2. 工具说明
  parts.push(TOOL_DESCRIPTIONS);
  
  // 3. 交互指南
  parts.push(INTERACTION_GUIDE);
  
  // 4. 项目特定
  if (config.projectPrompts) {
    parts.push(...config.projectPrompts);
  }
  
  return parts.join("\n\n");
}
```

## 设置管理

### 文件位置

`packages/coding-agent/src/core/settings-manager.ts`

### 设置文件

```json
{
  "model": { "provider": "anthropic", "id": "claude-sonnet-4-20250514" },
  "thinking": "medium",
  "theme": "default",
  "enabledModels": ["anthropic:*", "openai:*"],
  "autoCompactThreshold": 100000,
  "sessionDir": ".pi/sessions"
}
```

## 认证存储

### 文件位置

`packages/coding-agent/src/core/auth-storage.ts`

### 认证存储

```typescript
class AuthStorage {
  // API keys
  // OAuth tokens
  // Provider 认证
  
  setApiKey(provider: string, key: string): void;
  getApiKey(provider: string): string | undefined;
  
  setOAuthToken(provider: string, token: OAuthToken): void;
  getOAuthToken(provider: string): OAuthToken | undefined;
  
  save(): void;
  load(): void;
}
```

## 总结

pi-coding-agent CLI 完整流程：

1. **参数解析** - CLI 参数转为配置
2. **模式选择** - Interactive/Print/RPC/JSON
3. **会话管理** - 创建/继续/分支/压缩
4. **AgentSession** - 运行时抽象
5. **工具系统** - 7 个内置工具
6. **TUI 渲染** - 交互式界面

关键特性：
- 完整会话持久化 (JSONL)
- 模型切换 (Ctrl+P)
- 会话分支和压缩
- 工具可扩展