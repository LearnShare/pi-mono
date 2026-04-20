# pi-extensions 与 Skills 系统

## 概述

扩展系统是 pi 的核心可扩展机制：
- **Extensions** - TypeScript 模块，完整功能扩展
- **Skills** - 轻量级技能定义，文件系统

## Extensions 系统

### 文件结构

```
packages/coding-agent/src/core/extensions/
├── types.ts           # 类型定义 (~1471 行)
├── loader.ts         # 扩展加载器
├── runner.ts        # 扩展运行器
├── wrapper.ts       # 工具包装
└── index.ts       # 入口
```

### 扩展接口

```typescript
export interface Extension {
  name: string;
  version: string;
  description: string;
  
  register(context: ExtensionContext): Promise<void> | void;
  dispose?(): Promise<void> | void;
}
```

### 扩展上下文

```typescript
export interface ExtensionContext {
  // 会话
  session: AgentSession;
  
  // 服务
  services: ExtensionServices;
  
  // UI
  ui: ExtensionUIContext;
  
  // 事件
  events: EventBus;
  
  // 配置
  config: ExtensionConfig;
}
```

### 扩展服务

```typescript
export interface ExtensionServices {
  // 会话管理
  sessionManager: SessionManager;
  settingsManager: SettingsManager;
  modelRegistry: ModelRegistry;
  resourceLoader: ResourceLoader;
  
  // 当前上下文
  cwd: string;
  agentDir: string;
  
  // API
  api: PiApi;
}
```

### 扩展 UI 上下文

```typescript
export interface ExtensionUIContext {
  // 选择器
  select(title: string, options: string[], opts?: ExtensionUIDialogOptions): Promise<string | undefined>;
  
  // 确认对话框
  confirm(title: string, message: string, opts?: ExtensionUIDialogOptions): Promise<boolean>;
  
  // 文本输入
  input(title: string, placeholder?: string, opts?: ExtensionUIDialogOptions): Promise<string | undefined>;
  
  // 通知
  notify(message: string, type?: "info" | "warning" | "error"): void;
  
  // 终端输入监听
  onTerminalInput(handler: TerminalInputHandler): () => void;
  
  // 状态栏
  setStatus(key: string, text: string | undefined): void;
  
  // 工作消息
  setWorkingMessage(message?: string): void;
  
  // 隐藏思考标签
  setHiddenThinkingLabel(label?: string): void;
  
  // 组件
  setWidget(key: string, content: string[] | undefined, options?: ExtensionWidgetOptions): void;
  
  // 页脚
  setFooter(factory: FooterFactory | undefined): void;
  
  // 覆盖层
  showOverlay(component: Component, options?: OverlayOptions): OverlayHandle;
}
```

### 扩展选项

```typescript
export interface ExtensionUIDialogOptions {
  signal?: AbortSignal;
  timeout?: number;
}

export interface ExtensionWidgetOptions {
  placement?: "aboveEditor" | "belowEditor";
}
```

### 工具定义

```typescript
export interface ToolDefinition<TInput = any, TDetails = any> {
  name: string;
  description: string;
  parameters: TSchema;
  
  render?: (details: TDetails, options: ToolRenderResultOptions) => ToolRenderResult;
}
```

### 命令定义

```typescript
export interface CommandDefinition {
  name: string;
  description: string;
  
  execute(args: string[]): Promise<void>;
}
```

### 快捷键定义

```typescript
export interface KeybindingDefinition {
  key: string;
  command: string;
  description?: string;
}
```

### CLI 标志定义

```typescript
export interface FlagDefinition {
  name: string;
  type: "string" | "number" | "boolean";
  description: string;
  default?: any;
}
```

### 事件

```typescript
export interface ExtensionEvents {
  // 生命周期
  onInstall?: (context: ExtensionContext) => void;
  onUninstall?: (context: ExtensionContext) => void;
  
  // 会话
  onSessionStart?: (session: AgentSession) => void;
  onSessionEnd?: (session: AgentSession) => void;
  
  // 消息
  onMessage?: (message: AgentMessage) => void;
  onMessageStart?: (message: AgentMessage) => void;
  onMessageEnd?: (message: AgentMessage) => void;
  
  // 工具
  onToolCall?: (toolCall: AgentToolCall) => void;
  onToolResult?: (result: AgentToolResult) => void;
  
  // 压缩
  onBeforeCompact?: (context: CompactContext) => CompactContext;
  onAfterCompact?: (result: CompactionResult) => void;
}
```

## 扩展加载

### 文件位置

`packages/coding-agent/src/core/extensions/loader.ts`

### 加载流程

```typescript
class ExtensionLoader {
  private extensions: Map<string, Extension> = new Map();
  private toolDefs: Map<string, ToolDefinition> = new Map();
  private commands: Map<string, CommandDefinition> = new Map();
  private keybindings: Map<string, KeybindingDefinition> = new Map();
  private flags: Map<string, FlagDefinition> = new Map();
  
  async load(path: string): Promise<Extension> {
    // 1. 加载模块
    const module = await import(path);
    
    // 2. 创建扩展实例
    const extension = new module.default();
    
    // 3. 注册
    await extension.register(context);
    
    // 4. 存储
    this.extensions.set(extension.name, extension);
    
    return extension;
  }
  
  async unload(name: string): Promise<void> {
    const extension = this.extensions.get(name);
    if (extension.dispose) {
      await extension.dispose();
    }
    this.extensions.delete(name);
  }
}
```

### 扩展目录结构

```
.my-extensions/
└── my-extension/
    ├── package.json
    ├── index.ts          # 入口，导出 Extension 类
    ├── tools.ts          # 工具定义
    ├── commands.ts       # 命令定义
    └── keybindings.ts   # 快捷键定义
```

### package.json

```json
{
  "name": "my-extension",
  "version": "1.0.0",
  "main": "index.ts",
  "type": "module",
  "peerDependencies": {
    "@mariozechner/pi-coding-agent": "^0.67.0"
  }
}
```

### 扩展示例

```typescript
// index.ts
import type { Extension, ExtensionContext } from "@mariozechner/pi-coding-agent";

export default class MyExtension implements Extension {
  name = "my-extension";
  version = "1.0.0";
  description = "My custom extension";
  
  async register(context: ExtensionContext): Promise<void> {
    // 注册工具
    context.services.registerTool(myTool);
    
    // 注册命令
    context.services.registerCommand(myCommand);
    
    // 订阅事件
    context.events.subscribe("onMessage", async (message) => {
      // 处理消息
    });
  }
  
  async dispose(): Promise<void> {
    // 清理
  }
}
```

## 扩展运行器

### 文件位置

`packages/coding-agent/src/core/extensions/runner.ts`

### 扩展运行器

```typescript
class ExtensionRunner {
  private loader: ExtensionLoader;
  private extensions: Extension[] = [];
  
  async initialize(): Promise<void> {
    // 加载所有扩展
    for (const path of extensionPaths) {
      const ext = await this.loader.load(path);
      this.extensions.push(ext);
    }
  }
  
  async runToolCall(toolCall: AgentToolCall, context: AgentContext): Promise<AgentToolResult> {
    const toolDef = this.getToolDefinition(toolCall.name);
    if (!toolDef) {
      throw new Error(`Unknown tool: ${toolCall.name}`);
    }
    
    // 执行工具
    return await toolDef.execute(toolCall.args, context);
  }
  
  async runCommand(command: string, args: string[]): Promise<void> {
    const cmdDef = this.getCommandDefinition(command);
    if (!cmdDef) {
      throw new Error(`Unknown command: ${command}`);
    }
    
    return await cmdDef.execute(args);
  }
}
```

## Skills 系统

### 文件位置

`packages/coding-agent/src/core/skills.ts` (~508 行)

### Skill 定义

```typescript
export interface Skill {
  name: string;
  description: string;
  filePath: string;
  baseDir: string;
  sourceInfo: SourceInfo;
  disableModelInvocation: boolean;
}
```

### Skill 前 matter

```yaml
---
name: my-skill
description: 使用 my-tool 执行任务
disable-model-invocation: false
---

# My Skill

此技能用于...
```

### 验证规则

```typescript
function validateName(name: string, parentDirName: string): string[] {
  const errors: string[] = [];
  
  // 名称必须匹配父目录
  if (name !== parentDirName) {
    errors.push(`name "${name}" does not match parent directory "${parentDirName}"`);
  }
  
  // 名称长度限制
  if (name.length > MAX_NAME_LENGTH) {
    errors.push(`name exceeds ${MAX_NAME_LENGTH} characters`);
  }
  
  // 描述长度限制
  if (description.length > MAX_DESCRIPTION_LENGTH) {
    errors.push(`description exceeds ${MAX_DESCRIPTION_LENGTH} characters`);
  }
  
  return errors;
}
```

### 目录结构

```
.skills/
├── my-skill/
│   ├── SKILL.md        # 技能定义
│   └── prompt.md     # 可选: prompt 片段
├── another-skill/
│   └── ...
```

### SKILL.md 格式

```markdown
---
name: my-skill
description: 使用 my-tool 执行特定任务
disable-model-invocation: false
---

# My Skill

此技能用于执行...

## 使用方法

调用 `my-tool` 工具。

## 示例

```
<skill name="my-skill" location=".skills/my-skill">
实际参数
</skill>
```
```

### 技能加载

```typescript
function loadSkills(paths: string[]): LoadSkillsResult {
  const skills: Skill[] = [];
  const diagnostics: ResourceDiagnostic[] = [];
  
  for (const path of paths) {
    // 1. 查找所有子目录
    const skillDirs = findSkillDirs(path);
    
    // 2. 加载每个技能
    for (const dir of skillDirs) {
      const skillFile = path.join(dir, "SKILL.md");
      
      // 3. 解析 frontmatter
      const { frontmatter, content } = parseFrontmatter(readFileSync(skillFile));
      
      // 4. 验证
      const errors = validateName(frontmatter.name, basename(dir));
      if (errors.length > 0) {
        diagnostics.push({ path: skillFile, errors });
        continue;
      }
      
      // 5. 创建 skill
      skills.push({
        name: frontmatter.name,
        description: frontmatter.description,
        filePath: skillFile,
        baseDir: dir,
        sourceInfo: createSourceInfo(content),
        disableModelInvocation: frontmatter.disableModelInvocation,
      });
    }
  }
  
  return { skills, diagnostics };
}
```

## 工具包装

### 文件位置

`packages/coding-agent/src/core/extensions/wrapper.ts`

### 工具包装

```typescript
function wrapToolDefinition(tool: AgentTool): ToolDefinition {
  return {
    name: tool.name,
    description: tool.description,
    parameters: tool.parameters,
    
    execute(args: unknown, context: ToolExecutionContext): Promise<ToolResult> {
      const validatedArgs = validateArgs(tool.parameters, args);
      return tool.execute(validatedArgs, context);
    },
    
    render(details: ToolDetails, options: ToolRenderResultOptions): ToolRenderResult { ... },
  };
}
```

### 注册工具到 Agent

```typescript
function wrapRegisteredTools(
  tools: AgentTool[],
): ToolDefinition[] {
  return tools.map(wrapToolDefinition);
}
```

## 事件总线

### 事件类型

```typescript
type EventType =
  | "message"
  | "message_start"
  | "message_end"
  | "tool_call"
  | "tool_result"
  | "session_start"
  | "session_end";
```

### EventBus

```typescript
class EventBus {
  private listeners: Map<EventType, Set<EventListener>> = new Map();
  
  subscribe(type: EventType, listener: EventListener): () => void {
    // 订阅
  }
  
  emit(event: AgentEvent): void {
    // 触发
  }
}
```

## 配置

### 配置文件

```json
{
  "extensions": {
    "paths": [".my-extensions"],
    "disabled": ["disabled-extension"]
  },
  "skills": {
    "paths": [".skills"],
    "disabled": []
  }
}
```

### 加载配置

```typescript
// settings.json
{
  "extensions": [".my-extensions"],
  "noExtensions": false,
  "skills": [".skills"],
  "noSkills": false
}
```

## 使用示例

### 使用 Skill

```xml
<skill name="my-skill" location=".skills/my-skill">
参数内容
</skill>
```

### 使用 Extension 工具

```
使用 my-tool 工具执行...
```

### 使用 Extension 命令

```
/my-command arg1 arg2
```

### 快捷键

```
Ctrl+Shift+E  # 执行扩展命令
```

## 总结

扩展系统设计：

1. **Extensions** - 完整 TypeScript 模块
   - 工具注册
   - 命令注册
   - 快捷键注册
   - 事件订阅

2. **Skills** - 轻量文件系统定义
   - Frontmatter 配置
   - 验证规则

3. **工具包装** - AgentTool → ToolDefinition
4. **事件系统** - 生命周期事件

核心特性：
- 完整的生命周期管理
- UI 交互支持
- 事件订阅
- 配置化加载