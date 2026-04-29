# PI Agent 设置、自定义和扩展方法

## 概述

PI Agent 是一个极简主义终端编码框架，通过 TypeScript 模块实现功能增强。它在核心保持精简的同时，支持通过扩展、技能、模板、主题和包等方式进行定制。

## 安装与初始设置

### 安装

```bash
npm install -g @mariozechner/pi-coding-agent
```

安装完成后，在项目目录中执行 `pi` 即可启动。

### 认证配置

PI Agent 支持两种认证方式：

1. **交互式登录**：使用 `/login` 命令通过浏览器完成订阅提供商的认证
2. **环境变量**：预先设置 API 密钥

```bash
export ANTHROPIC_API_KEY=your_key_here
export OPENAI_API_KEY=your_key_here
```

## 配置文件系统

PI Agent 使用 JSON 配置文件，支持全局和项目级配置。

### 配置文件位置

| 位置 | 作用域 | 说明 |
|------|--------|------|
| `~/.pi/agent/settings.json` | 全局 | 适用于所有项目 |
| `.pi/settings.json` | 项目 | 当前目录覆盖全局配置 |

### 配置优先级

项目设置会覆盖全局设置，嵌套对象会进行合并：

```json
// ~/.pi/agent/settings.json (全局)
{
  "theme": "dark",
  "compaction": { "enabled": true, "reserveTokens": 16384 }
}

// .pi/settings.json (项目)
{
  "compaction": { "reserveTokens": 8192 }
}

// 最终生效配置
{
  "theme": "dark",
  "compaction": { "enabled": true, "reserveTokens": 8192 }
}
```

## 核心配置选项

### 模型与思考级别

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `defaultProvider` | string | - | 默认提供商，如 `"anthropic"`, `"openai"` |
| `defaultModel` | string | - | 默认模型 ID |
| `defaultThinkingLevel` | string | - | 思考级别：`off`, `minimal`, `low`, `medium`, `high`, `xhigh` |
| `hideThinkingBlock` | boolean | `false` | 隐藏输出中的思考块 |
| `thinkingBudgets` | object | - | 自定义各思考级别的 token 预算 |

### 思考级别预算配置

```json
{
  "thinkingBudgets": {
    "minimal": 1024,
    "low": 4096,
    "medium": 10240,
    "high": 32768
  }
}
```

### UI 与显示

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `theme` | string | `"dark"` | 主题名称：`dark`, `light` 或自定义 |
| `quietStartup` | boolean | `false` | 隐藏启动标题 |
| `collapseChangelog` | boolean | `false` | 更新后显示精简的变更日志 |
| `editorPaddingX` | number | `0` | 编辑器水平内边距 (0-3) |
| `autocompleteMaxVisible` | number | `5` | 自动完成下拉框最大可见项 (3-20) |
| `showHardwareCursor` | boolean | `false` | 显示终端光标 |

### 会话压缩

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `compaction.enabled` | boolean | `true` | 启用自动压缩 |
| `compaction.reserveTokens` | number | `16384` | 为 LLM 响应保留的 token |
| `compaction.keepRecentTokens` | number | `20000` | 保留的近期 token 数（不摘要） |

### 重试机制

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `retry.enabled` | boolean | `true` | 启用自动重试 |
| `retry.maxRetries` | number | `3` | 最大重试次数 |
| `retry.baseDelayMs` | number | `2000` | 指数退避基础延迟 |

### 资源加载

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `packages` | array | `[]` | 要加载的 npm/git 包 |
| `extensions` | string[] | `[]` | 本地扩展文件或目录路径 |
| `skills` | string[] | `[]` | 本地技能文件或目录路径 |
| `prompts` | string[] | `[]` | 本地提示模板路径 |
| `themes` | string[] | `[]` | 本地主题文件或目录路径 |
| `enableSkillCommands` | boolean | `true` | 注册技能为 `/skill:name` 命令 |

### 数组配置语法

支持 glob 模式和排除：
- `!pattern` - 排除匹配项
- `+path` - 强制包含指定路径
- `-path` - 强制排除指定路径

```json
{
  "themes": ["~/.pi/themes/*", "!*-old", "+/absolute/path/theme.json"]
}
```

### 包配置

字符串形式加载包的所有资源：

```json
{
  "packages": ["pi-skills", "@org/my-extension"]
}
```

对象形式选择加载特定资源：

```json
{
  "packages": [
    {
      "source": "pi-skills",
      "skills": ["brave-search", "transcribe"],
      "extensions": []
    }
  ]
}
```

## 完整配置示例

```json
{
  "$schema": "https://pi.dev/schema/settings.json",
  "defaultProvider": "anthropic",
  "defaultModel": "claude-sonnet-4-20250514",
  "defaultThinkingLevel": "medium",
  "theme": "dark",
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  },
  "retry": {
    "enabled": true,
    "maxRetries": 3
  },
  "enabledModels": ["claude-*", "gpt-4o"],
  "packages": ["pi-skills"],
  "extensions": ["~/.pi/extensions/*.ts"],
  "skills": ["~/.pi/skills/**/*"]
}
```

## 主题自定义

### 主题文件位置

| 位置 | 说明 |
|------|------|
| Built-in | `dark`, `light` |
| `~/.pi/agent/themes/*.json` | 全局主题 |
| `.pi/themes/*.json` | 项目主题 |

### 创建自定义主题

创建主题文件 `~/.pi/agent/themes/my-theme.json`：

```json
{
  "$schema": "https://raw.githubusercontent.com/badlogic/pi-mono/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "my-theme",
  "vars": {
    "primary": "#00aaff",
    "secondary": 242
  },
  "colors": {
    "accent": "primary",
    "border": "primary",
    "borderAccent": "#00ffff",
    "borderMuted": "secondary",
    "success": "#00ff00",
    "error": "#ff0000",
    "warning": "#ffff00",
    "muted": "secondary",
    "dim": 240,
    "text": "",
    "thinkingText": "secondary",
    "selectedBg": "#2d2d30",
    "userMessageBg": "#2d2d30",
    "toolTitle": "primary",
    "toolOutput": "",
    "mdHeading": "#ffaa00",
    "mdLink": "primary",
    "mdCode": "#00ffff",
    "syntaxKeyword": "primary",
    "syntaxFunction": "#00aaff",
    "syntaxString": "#00ff00",
    "thinkingOff": "secondary",
    "thinkingMinimal": "primary",
    "thinkingLow": "#00aaff",
    "thinkingMedium": "#00ffff",
    "thinkingHigh": "#ff00ff",
    "thinkingXhigh": "#ff0000"
  }
}
```

### 颜色值格式

| 格式 | 示例 | 说明 |
|------|------|------|
| Hex | `"#ff0000"` | 6位十六进制RGB |
| 256色 | `39` | xterm 256色板索引 (0-255) |
| 变量 | `"primary"` | 引用 `vars` 中定义的颜色 |
| 默认 | `""` | 使用终端默认颜色 |

**注意**：编辑当前活动的自定义主题文件时，PI 会自动热重载以立即显示效果。

## 技能系统

### 什么是技能

技能是自包含的能力包，Agent 按需加载。每个技能提供：
- 特定任务的专业工作流
- 设置说明
- 辅助脚本
- 参考文档

### 技能文件位置

| 作用域 | 位置 |
|--------|------|
| 全局 | `~/.pi/agent/skills/` |
| 项目 | `.pi/skills/` |
| 包内 | `skills/` 目录或 `package.json` 的 `pi.skills` 条目 |

### SKILL.md 格式

```markdown
---
name: brave-search
description: Web search and content extraction via Brave Search API. Use for searching documentation, facts, or any web content.
---

# Brave Search

## Setup

```bash
cd /path/to/brave-search && npm install
```

## Search

```bash
./search.js "query"        # Basic search
./search.js "query" --content  # Include page content
```
```

### 命名规范

- 1-64 字符
- 仅小写字母、数字和连字符
- 不能以连字符开头或结尾
- 不能包含连续连字符
- 必须与父目录名称匹配

### 使用技能

技能通过提示词展开使用。当任务与技能描述匹配时，Agent 会加载对应的 SKILL.md 文件。

```bash
/skill:brave-search  # 加载并执行技能
/skill:pdf-tools extract  # 加载技能并传递参数
```

### 跨 Agent 技能共享

```json
{
  "skills": [
    "~/.claude/skills",
    "~/.codex/skills"
  ]
}
```

## 扩展系统

### 扩展能力

扩展是 TypeScript 模块，可以：
- 注册自定义工具供 LLM 调用
- 拦截生命周期事件
- 注入上下文
- 自定义压缩行为
- 与用户交互（选择、确认、输入）
- 自定义 UI 组件
- 注册自定义命令
- 持久化状态数据

### 扩展文件位置

| 位置 | 作用域 | 说明 |
|------|--------|------|
| `~/.pi/agent/extensions/*.ts` | 全局 | 所有项目可用 |
| `~/.pi/agent/extensions/*/index.ts` | 全局 | 子目录形式 |
| `.pi/extensions/*.ts` | 项目 | 仅当前项目 |
| `.pi/extensions/*/index.ts` | 项目 | 子目录形式 |

### 快速开始

创建 `~/.pi/agent/extensions/my-extension.ts`：

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // 响应事件
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("Extension loaded!", "info");
  });

  // 拦截危险命令
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
      if (!ok) return { block: true, reason: "Blocked by user" };
    }
  });

  // 注册自定义工具
  pi.registerTool({
    name: "greet",
    label: "Greet",
    description: "Greet someone by name",
    parameters: Type.Object({
      name: Type.String({ description: "Name to greet" }),
    }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      return {
        content: [{ type: "text", text: `Hello, ${params.name}!` }],
        details: {},
      };
    },
  });

  // 注册命令
  pi.registerCommand("hello", {
    description: "Say hello",
    handler: async (args, ctx) => {
      ctx.ui.notify(`Hello ${args || "world"}!`, "info");
    },
  });
}
```

### 测试扩展

```bash
# 使用 -e 或 --extension 标志测试单个扩展
pi -e ./my-extension.ts
```

### 扩展结构风格

**单文件** - 简单扩展：
```
~/.pi/agent/extensions/
└── my-extension.ts
```

**带 index.ts 的目录** - 多文件扩展：
```
~/.pi/agent/extensions/
└── my-extension/
    ├── index.ts      # 入口点
    ├── tools.ts      # 工具模块
    └── utils.ts      # 辅助模块
```

**带依赖的包** - 需要 npm 包：
```
~/.pi/agent/extensions/
└── my-extension/
    ├── package.json
    ├── package-lock.json
    ├── node_modules/
    └── src/
        └── index.ts
```

package.json:
```json
{
  "name": "my-extension",
  "dependencies": {
    "zod": "^3.0.0"
  },
  "pi": {
    "extensions": ["./src/index.ts"]
  }
}
```

### 可导入的依赖

| 包 | 用途 |
|----|------|
| `@mariozechner/pi-coding-agent` | 扩展类型定义 |
| `typebox` | 工具参数 Schema 定义 |
| `@mariozechner/pi-ai` | AI 工具函数 |
| `@mariozechner/pi-tui` | TUI 组件 |

### 热重载

放在自动发现位置的扩展支持热重载：
```
/reload  # 重新加载扩展、技能、提示和主题
```

## 会话配置

### 会话目录

```json
{ "sessionDir": ".pi/sessions" }
```

支持绝对路径、相对路径和 `~` 展开。CLI 标志 `--session-dir` 优先级最高。

### 可用模型

配置模型循环列表（Ctrl+P 切换）：

```json
{
  "enabledModels": ["claude-*", "gpt-4o", "gemini-2*"]
}
```

## 命令行选项

### 常用 CLI 标志

```bash
# 模型选择
pi --model claude-sonnet-4-20250514
pi --provider anthropic
pi --thinking medium

# 会话选项
pi --session <id>       # 指定会话
pi --resume             # 恢复会话
pi --continue           # 继续最近会话
pi --no-session         # 不保存会话

# 工具控制
pi --tools read,bash,edit   # 仅启用指定工具
pi --no-tools           # 禁用所有工具
pi --read-only          # 仅使用只读工具

# 扩展和技能
pi --extension <path>   # 加载扩展
pi --no-extensions      # 禁用扩展
pi --skill <path>       # 加载技能
pi --no-skills          # 禁用技能

# 提示模板
pi --prompt <path>      # 加载提示模板
pi --no-prompts         # 禁用提示模板

# 主题
pi --theme <name>       # 使用主题
pi --no-themes          # 禁用主题

# 输出控制
pi -p "prompt"          # 打印模式
pi --mode json          # JSON 输出模式
pi --mode rpc           # RPC 模式
```

## 提示模板

### 模板位置

| 位置 | 说明 |
|------|------|
| `~/.pi/agent/prompts/*.md` | 全局模板 |
| `.pi/prompts/*.md` | 项目模板 |

### 模板格式

```markdown
---
name: my-template
description: Description shown in /prompts
---

Your prompt template content here.
Use {{variable}} for substitution.
```

### 使用模板

```bash
/my-template    # 调用模板
```

## 键绑定配置

键绑定配置文件：`~/.pi/agent/keybindings.json`

```json
{
  "app.editor.submit": "ctrl+s",
  "app.tools.expand": "e"
}
```

## 安全注意事项

1. **扩展安全**：扩展以系统完整权限运行，可以执行任意代码。仅从可信来源安装。

2. **技能安全**：技能可以指示模型执行任何操作，可能包含模型可执行的可执行代码。使用前审查技能内容。

3. **环境变量**：API 密钥存储在环境变量中，不会被意外提交到版本控制。

4. **权限门**：使用扩展拦截危险命令（如 `rm -rf`, `sudo`），要求用户确认。
