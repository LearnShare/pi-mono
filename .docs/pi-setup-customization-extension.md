# Pi Agent: 安装、设置、自定义与扩展方法

> 基于 pi.dev/docs/latest 官方文档整理
> 最后更新: 2026-04-29

---

## 1. 安装

### 全局安装（推荐）

```bash
npm install -g @mariozechner/pi-coding-agent
```

### 验证安装

```bash
pi --version
pi --help
```

### 更新

```bash
pi update              # 更新 pi 和所有非固定版本的包
pi update --self       # 仅更新 pi 自身
pi update --extensions # 仅更新包
```

---

## 2. 身份认证

### 订阅登录（/login）

在交互模式下运行 `/login`，选择提供商进行 OAuth 认证：

| 提供商 | 说明 |
|--------|------|
| Claude Pro/Max | Anthropic 订阅 |
| ChatGPT Plus/Pro (Codex) | OpenAI 订阅 |
| GitHub Copilot | GitHub Copilot 订阅（支持企业版） |
| Google Gemini CLI | Google 免费/付费账户 |
| Google Antigravity | Google 沙盒环境 |

令牌存储在 `~/.pi/agent/auth.json`，过期自动刷新。

### API Key 认证

设置环境变量：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

或者在 `/login` 中选择 API Key 提供商，将密钥存入 `auth.json`。

**支持的 API Key 提供商（部分）：**

| 提供商 | 环境变量 | auth.json key |
|--------|---------|---------------|
| Anthropic | `ANTHROPIC_API_KEY` | `anthropic` |
| OpenAI | `OPENAI_API_KEY` | `openai` |
| DeepSeek | `DEEPSEEK_API_KEY` | `deepseek` |
| Google Gemini | `GEMINI_API_KEY` | `google` |
| Mistral | `MISTRAL_API_KEY` | `mistral` |
| Groq | `GROQ_API_KEY` | `groq` |
| xAI | `XAI_API_KEY` | `xai` |
| OpenRouter | `OPENROUTER_API_KEY` | `openrouter` |
| Hugging Face | `HF_TOKEN` | `huggingface` |

### 云提供商

- **Azure OpenAI**: 需要 `AZURE_OPENAI_API_KEY` + `AZURE_OPENAI_BASE_URL`
- **Amazon Bedrock**: 支持 AWS Profile / IAM Keys / Bearer Token
- **Cloudflare Workers AI**: `CLOUDFLARE_API_KEY` + `CLOUDFLARE_ACCOUNT_ID`
- **Google Vertex AI**: 使用 Application Default Credentials

---

## 3. 基础使用

### 交互模式

```bash
cd /path/to/project
pi
```

界面四个区域：
- **启动头部** - 快捷键、上下文文件、模板、技能、扩展
- **消息区域** - 对话历史、工具调用、扩展 UI
- **编辑器** - 输入区域，边框颜色表示思考级别
- **底部栏** - 工作目录、会话名、token/缓存用量、成本、模型

### 编辑器功能

| 功能 | 操作 |
|------|------|
| 文件引用 | 输入 `@` 模糊搜索项目文件 |
| 路径补全 | Tab 键 |
| 多行输入 | Shift+Enter |
| 图片粘贴 | Ctrl+V（Windows 用 Alt+V） |
| Shell 命令 | `!command` 执行并发送输出给模型 |
| 隐藏命令 | `!!command` 执行但不发送输出 |
| 外部编辑器 | Ctrl+G 打开 `$VISUAL` 或 `$EDITOR` |

### 非交互模式

```bash
pi -p "Summarize this codebase"
cat README.md | pi -p "Summarize this text"
pi -p @screenshot.png "What's in this image?"
```

### 消息队列

| 操作 | 说明 |
|------|------|
| Enter | 排队转向消息（当前助手回合工具调用完成后发送） |
| Alt+Enter | 排队后续消息（助手全部工作完成后发送） |
| Escape | 终止并恢复排队消息到编辑器 |
| Alt+Up | 将排队消息召回编辑器 |

---

## 4. 配置

### 设置文件

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/settings.json` | 全局（所有项目） |
| `.pi/settings.json` | 项目级 |

项目设置覆盖全局设置，嵌套对象合并。

### 核心设置项

**模型 & 思考：**

```json
{
  "defaultProvider": "anthropic",
  "defaultModel": "claude-sonnet-4-20250514",
  "defaultThinkingLevel": "medium",
  "hideThinkingBlock": false
}
```

**UI & 显示：**

```json
{
  "theme": "dark",
  "quietStartup": false,
  "doubleEscapeAction": "tree",
  "editorPaddingX": 0,
  "autocompleteMaxVisible": 5,
  "showHardwareCursor": false
}
```

**压缩（Compaction）：**

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

**重试：**

```json
{
  "retry": {
    "enabled": true,
    "maxRetries": 3,
    "baseDelayMs": 2000
  }
}
```

**终端 & 图片：**

```json
{
  "terminal.showImages": true,
  "terminal.imageWidthCells": 60,
  "images.autoResize": true,
  "images.blockImages": false
}
```

### 自定义模型

通过 `~/.pi/agent/models.json` 添加本地模型（Ollama、vLLM、LM Studio）：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        { "id": "llama3.1:8b" },
        { "id": "qwen2.5-coder:7b" }
      ]
    }
  }
}
```

支持的 API 类型：`openai-completions`、`openai-responses`、`anthropic-messages`、`google-generative-ai`。

### CLI 参数

```bash
pi [options] [@files...] [messages...]
```

| 参数 | 说明 |
|------|------|
| `-p`, `--print` | 打印响应后退出 |
| `--mode json` | JSON 事件流模式 |
| `--mode rpc` | RPC 模式 |
| `--provider <name>` | 指定提供商 |
| `--model <pattern>` | 指定模型（支持 `provider/id` 和 `:thinking`） |
| `--thinking <level>` | 思考级别 |
| `-c`, `--continue` | 继续最近的会话 |
| `-r`, `--resume` | 浏览选择会话 |
| `--session <path|id>` | 打开特定会话 |
| `--tools <list>` | 工具白名单 |
| `--no-tools` | 禁用所有工具 |
| `-e`, `--extension <source>` | 加载扩展 |
| `--no-extensions` | 禁用扩展发现 |
| `--skill <path>` | 加载技能 |
| `--theme <path>` | 加载主题 |
| `--no-context-files` | 禁用 AGENTS.md/CLAUDE.md 发现 |

---

## 5. 自定义方法概览

Pi 提供 6 种自定义机制：

| 机制 | 作用 |
|------|------|
| **Extensions** | TypeScript 模块，可注册工具、监听事件、添加命令、自定义 UI |
| **Skills** | 按需加载的能力包，含 SKILL.md 指令和辅助脚本 |
| **Prompt Templates** | 可复用的 Markdown 提示模板，通过 `/name` 扩展 |
| **Themes** | JSON 主题文件，定义 TUI 的 51 个颜色 token |
| **Pi Packages** | 打包扩展、技能、模板、主题，通过 npm/git 共享 |
| **Custom Models** | 通过 models.json 添加自定义提供商和模型 |

以下各节详细介绍每种机制。

---

## 6. 自定义机制详解

### 6.1 Extensions（扩展）

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

**扩展能力：**

- 注册自定义工具（`pi.registerTool()`）
- 监听生命周期事件（`pi.on()`）
- 用户交互（`ctx.ui.select/confirm/input/notify`）
- 自定义 TUI 组件（`ctx.ui.custom()`）
- 注册命令（`pi.registerCommand()`）
- 会话持久化（`pi.appendEntry()`）
- 自定义渲染（`pi.registerMessageRenderer()`）

**示例** (`~/.pi/agent/extensions/my-extension.ts`)：

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

详细 API 参见第三篇文档。

### 6.2 Skills（技能）

技能是包含 `SKILL.md` 的目录，按需加载。遵循 [Agent Skills 标准](https://agentskills.io/specification)。

**发现位置：**

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/skills/` | 全局 |
| `~/.agents/skills/` | 全局 |
| `.pi/skills/` | 项目级 |
| `.agents/skills/` | 项目级及祖先目录 |
| 包中的 `skills/` 目录 | 包级 |

**SKILL.md 格式：**

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
./search.js "query"
```
```

通过 `/skill:name` 命令调用技能。

**推荐技能仓库：**
- [Anthropic Skills](https://github.com/anthropics/skills) - 文档处理（docx, pdf, pptx, xlsx）、Web 开发
- [Pi Skills](https://github.com/badlogic/pi-skills) - 网页搜索、浏览器自动化、Google API、转录

### 6.3 Prompt Templates（提示模板）

Markdown 片段，通过 `/name` 扩展为完整提示。

**位置：** `~/.pi/agent/prompts/*.md`（全局）或 `.pi/prompts/*.md`（项目）

**格式：**

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

**带参数提示的格式：**

```markdown
---
description: Create a component
argument-hint: "<ComponentName> [features...]"
---
Create a React component named $1 with features: $@
```

### 6.4 Themes（主题）

JSON 文件定义 TUI 的全部 51 个颜色 token。

**位置：** `~/.pi/agent/themes/*.json`（全局）或 `.pi/themes/*.json`（项目）

**选择主题：**

```json
{
  "theme": "my-theme"
}
```

**主题结构：**

```json
{
  "$schema": "https://raw.githubusercontent.com/badlogic/pi-mono/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "my-theme",
  "vars": { "primary": "#00aaff" },
  "colors": {
    "accent": "primary",
    "border": "primary",
    "success": "#00ff00",
    "error": "#ff0000",
    // ... 共 51 个必要 token
  }
}
```

颜色值支持 Hex RGB、256 色索引、vars 变量引用、终端默认值。

### 6.5 Pi Packages（包）

打包和共享扩展、技能、提示模板、主题。

**安装：**

```bash
pi install npm:@foo/bar@1.0.0
pi install git:github.com/user/repo@v1
pi install /path/to/package
```

**创建包** (`package.json`)：

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

包可包含 `video`/`image` 元数据在 [Gallery](https://pi.dev/packages) 中展示。

### 6.6 上下文文件（Context Files）

使用 `AGENTS.md` 或 `CLAUDE.md` 给项目提供指令：

- `~/.pi/agent/AGENTS.md` - 全局指令
- 从 cwd 向上遍历祖先目录查找
- 当前目录

示例：

```markdown
# Project Instructions
- Run `npm run check` after code changes.
- Do not run production migrations locally.
- Keep responses concise.
```

修改后重启 pi 或运行 `/reload`。

系统提示也可通过 `.pi/SYSTEM.md`（项目级）或 `~/.pi/agent/SYSTEM.md`（全局）替换，或通过 `APPEND_SYSTEM.md` 追加。

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

---

## 8. 会话管理

会话自动保存到 `~/.pi/agent/sessions/`，按工作目录组织。

**关键命令：**

| 命令 | 说明 |
|------|------|
| `/resume` | 浏览选择之前的会话 |
| `/new` | 开始新会话 |
| `/name <name>` | 设置会话显示名称 |
| `/session` | 显示会话信息 |
| `/tree` | 导航会话树，支持分支 |
| `/fork` | 从之前的用户消息创建新会话 |
| `/clone` | 复制当前分支到新会话 |
| `/compact [prompt]` | 手动压缩上下文 |
| `/export [file]` | 导出会话为 HTML |
| `/share` | 上传为 GitHub Gist |

**CLI 会话选项：**

```bash
pi -c                  # 继续最近会话
pi -r                  # 浏览选择会话
pi --session <path|id> # 打开特定会话
pi --fork <path|id>    # Fork 会话
pi --no-session        # 临时模式，不保存
```

---

## 9. 设计哲学

Pi 保持核心小巧，将工作流特定行为推送到扩展、技能、提示模板和包中。故意不内置 MCP、子代理、权限弹窗、规划模式、待办事项等功能。这些可以通过扩展或包安装，或使用外部工具（容器、tmux 等）替代。