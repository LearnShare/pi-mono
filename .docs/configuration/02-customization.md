# 6种自定义机制

> 基于 pi.dev/docs/latest 官方文档整理

---

## 概览

Pi 提供 6 种自定义机制：

| 机制 | 作用 |
|------|------|
| **Extensions** | TypeScript 模块，可注册工具、监听事件、添加命令、自定义 UI |
| **Skills** | 按需加载的能力包，含 SKILL.md 指令和辅助脚本 |
| **Prompt Templates** | 可复用的 Markdown 提示模板，通过 `/name` 扩展 |
| **Themes** | JSON 主题文件，定义 TUI 的 51 个颜色 token |
| **Pi Packages** | 打包扩展、技能、模板、主题，通过 npm/git 共享 |
| **Custom Models** | 通过 models.json 添加自定义提供商和模型 |

---

## 1. Extensions（扩展）

扩展是 TypeScript 模块，自动发现于：

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/extensions/*.ts` | 全局 |
| `~/.pi/agent/extensions/*/index.ts` | 全局（子目录） |
| `.pi/extensions/*.ts` | 项目级 |
| `.pi/extensions/*/index.ts` | 项目级（子目录） |

通过 settings.json 指定额外路径：

```json
{
  "extensions": ["/path/to/extension.ts", "/path/to/extension/dir"]
}
```

### 扩展能力

- 注册自定义工具（`pi.registerTool()`）
- 监听生命周期事件（`pi.on()`）
- 用户交互（`ctx.ui.select/confirm/input/notify`）
- 自定义 TUI 组件（`ctx.ui.custom()`）
- 注册命令（`pi.registerCommand()`）
- 会话持久化（`pi.appendEntry()`）
- 自定义渲染（`pi.registerMessageRenderer()`）

### 示例

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
      if (!ok) return { block: true, reason: "Blocked by user" };
    }
  });

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

  pi.registerCommand("hello", {
    description: "Say hello",
    handler: async (args, ctx) => {
      ctx.ui.notify(`Hello ${args || "world"}!`, "info");
    },
  });
}
```

---

## 2. Skills（技能）

技能是包含 `SKILL.md` 的目录，按需加载。遵循 [Agent Skills 标准](https://agentskills.io/specification)。

### 发现位置

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/skills/` | 全局 |
| `~/.agents/skills/` | 全局 |
| `.pi/skills/` | 项目级 |
| `.agents/skills/` | 项目级及祖先目录 |
| 包中的 `skills/` 目录 | 包级 |

### SKILL.md 格式

```markdown
---
name: brave-search
description: Web search and content extraction via Brave Search API.
---

# Brave Search

## Setup
```bash
cd /path/to/brave-search && npm install
```

## Search
```bash
./search.js "query"
```

通过 `/skill:name` 命令调用技能。

---

## 3. Prompt Templates（提示模板）

Markdown 片段，通过 `/name` 扩展为完整提示。

### 位置

`~/.pi/agent/prompts/*.md`（全局）或 `.pi/prompts/*.md`（项目）

### 格式

```markdown
---
description: Review staged git changes
---
Review the staged changes (`git diff --cached`). Focus on:
- Bugs and logic errors
- Security issues
- Error handling gaps
```

支持参数：`$1`, `$2`, `$@`, `${@:N}`, `${@:N:L}`。

### 带参数提示

```markdown
---
description: Create a component
argument-hint: "<ComponentName> [features...]"
---
Create a React component named $1 with features: $@
```

---

## 4. Themes（主题）

JSON 文件定义 TUI 的全部 51 个颜色 token。

### 位置

`~/.pi/agent/themes/*.json`（全局）或 `.pi/themes/*.json`（项目）

### 选择主题

```json
{
  "theme": "my-theme"
}
```

### 主题结构

```json
{
  "$schema": "https://raw.githubusercontent.com/badlogic/pi-mono/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "my-theme",
  "vars": { "primary": "#00aaff" },
  "colors": {
    "accent": "primary",
    "border": "primary",
    "success": "#00ff00",
    "error": "#ff0000"
  }
}
```

颜色值支持 Hex RGB、256 色索引、vars 变量引用、终端默认值。

---

## 5. Pi Packages（包）

打包和共享扩展、技能、提示模板、主题。

### 安装

```bash
pi install npm:@foo/bar@1.0.0
pi install git:github.com/user/repo@v1
pi install /path/to/package
```

### 创建包

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

---

## 6. 上下文文件（Context Files）

使用 `AGENTS.md` 或 `CLAUDE.md` 给项目提供指令：

- `~/.pi/agent/AGENTS.md` - 全局指令
- 从 cwd 向上遍历祖先目录查找
- 当前目录

### 示例

```markdown
# Project Instructions
- Run `npm run check` after code changes.
- Do not run production migrations locally.
- Keep responses concise.
```

修改后重启 pi 或运行 `/reload`。

### 系统提示

系统提示也可通过：
- `.pi/SYSTEM.md`（项目级）
- `~/.pi/agent/SYSTEM.md`（全局）
- `APPEND_SYSTEM.md`（追加）

---

## 7. 快捷键自定义

创建 `~/.pi/agent/keybindings.json`：

```json
{
  "tui.editor.cursorUp": ["up", "ctrl+p"],
  "tui.editor.cursorDown": ["down", "ctrl+n"],
  "tui.editor.cursorLeft": ["left", "ctrl+b"],
  "tui.editor.cursorRight": ["right", "ctrl+f"],
  "tui.editor.deleteWordBackward": ["ctrl+w", "alt+backspace"],
  "tui.input.newLine": ["shift+enter", "ctrl+j"]
}
```

编辑后运行 `/reload` 生效（无需重启会话）。