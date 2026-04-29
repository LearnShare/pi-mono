# PI Agent 流行的扩展、插件和资源

## 概述

PI Agent 拥有一个日益增长的生态系统，涵盖了官方和社区贡献的扩展、技能、主题和包。本文档汇总了当前可用的资源。

## 官方资源

### 核心包

| 包名 | 说明 | 安装 |
|------|------|------|
| `@mariozechner/pi-coding-agent` | PI Agent CLI 核心 | `npm install -g @mariozechner/pi-coding-agent` |
| `@mariozechner/pi-tui` | TUI 组件库 | 随核心自动安装 |
| `@mariozechner/pi-ai` | AI 工具函数 | 随核心自动安装 |
| `typebox` | Schema 验证 | 随核心自动安装 |

### 官方包管理

```bash
# 安装扩展
pi install npm:@mariozechner/pi-extension

# 安装 git 包
pi install git:github.com/user/repo@v1

# 更新所有非固定版本包
pi update

# 更新单个包
pi update npm:@foo/bar

# 列出已安装包
pi list

# 移除包
pi remove npm:@foo/bar
```

## 官方扩展示例

PI Agent 内置了丰富的扩展示例，位于 `packages/coding-agent/examples/extensions/` 目录：

### 工具类扩展

| 扩展 | 说明 | 关键 API |
|------|------|----------|
| `hello.ts` | 最小化工具注册示例 | `registerTool` |
| `question.ts` | 带用户交互的工具 | `registerTool`, `ui.select` |
| `questionnaire.ts` | 多步骤向导工具 | `registerTool`, `ui.custom` |
| `todo.ts` | 有状态工具与持久化 | `registerTool`, `appendEntry`, `renderResult` |
| `dynamic-tools.ts` | 启动后注册工具 | `registerTool`, `session_start` |
| `structured-output.ts` | 结构化输出工具 | `registerTool`, `terminate: true` |
| `truncated-tool.ts` | 输出截断示例 | `registerTool`, `truncateHead` |
| `tool-override.ts` | 覆盖内置工具 | `registerTool` (相同名称) |

### 命令类扩展

| 扩展 | 说明 | 关键 API |
|------|------|----------|
| `pirate.ts` | 每轮修改系统提示词 | `registerCommand`, `before_agent_start` |
| `summarize.ts` | 会话摘要命令 | `registerCommand`, `ui.custom` |
| `handoff.ts` | 跨提供商模型切换 | `registerCommand`, `ui.editor`, `ui.custom` |
| `qna.ts` | 自定义 UI 问答 | `registerCommand`, `ui.custom`, `setEditorText` |
| `send-user-message.ts` | 注入用户消息 | `registerCommand`, `sendUserMessage` |
| `reload-runtime.ts` | 运行时重载 | `registerCommand`, `ctx.reload()` |
| `shutdown-command.ts` | 优雅关闭命令 | `registerCommand`, `ctx.shutdown()` |

### 事件与权限扩展

| 扩展 | 说明 | 关键 API |
|------|------|----------|
| `permission-gate.ts` | 拦截危险命令 | `on("tool_call")`, `ui.confirm` |
| `protected-paths.ts` | 保护特定路径 | `on("tool_call")` |
| `confirm-destructive.ts` | 确认破坏性会话变更 | `on("session_before_*")` |
| `dirty-repo-guard.ts` | 脏仓库警告 | `on("session_before_*")`, `exec` |
| `input-transform.ts` | 转换用户输入 | `on("input")` |
| `model-status.ts` | 响应模型变更 | `on("model_select")`, `setStatus` |
| `provider-payload.ts` | 检查 Provider 负载和响应头 | `on("before_provider_request")` |
| `system-prompt-header.ts` | 显示系统提示词信息 | `on("agent_start")`, `getSystemPrompt` |
| `claude-rules.ts` | 从文件加载规则 | `on("session_start")`, `on("before_agent_start")` |
| `prompt-customizer.ts` | 上下文感知工具指导 | `on("before_agent_start")`, `systemPromptOptions` |
| `file-trigger.ts` | 文件监视触发消息 | `sendMessage` |

### 压缩与会话扩展

| 扩展 | 说明 | 关键 API |
|------|------|----------|
| `custom-compaction.ts` | 自定义压缩摘要 | `on("session_before_compact")` |
| `trigger-compact.ts` | 手动触发压缩 | `compact()` |
| `git-checkpoint.ts` | 每轮 Git stash | `on("turn_start")`, `exec` |
| `auto-commit-on-exit.ts` | 退出时自动提交 | `on("session_shutdown")`, `exec` |

### UI 组件扩展

| 扩展 | 说明 | 关键 API |
|------|------|----------|
| `status-line.ts` | 页脚状态指示器 | `setStatus`, session events |
| `working-indicator.ts` | 自定义工作指示器 | `setWorkingIndicator`, `registerCommand` |
| `github-issue-autocomplete.ts` | GitHub Issue 自动完成 | `addAutocompleteProvider`, `exec` |
| `custom-footer.ts` | 完整替换页脚 | `registerCommand`, `setFooter` |
| `custom-header.ts` | 替换启动标题 | `on("session_start")`, `setHeader` |
| `modal-editor.ts` | Vim 风格模态编辑器 | `setEditorComponent`, `CustomEditor` |
| `rainbow-editor.ts` | 自定义编辑器样式 | `setEditorComponent` |
| `widget-placement.ts` | 小部件位置控制 | `setWidget` |
| `overlay-test.ts` | 覆盖层组件测试 | `ui.custom` with overlay |
| `overlay-qa-tests.ts` | 完整覆盖层测试套件 | `ui.custom`, overlay options |
| `notify.ts` | 简单通知 | `ui.notify` |
| `timed-confirm.ts` | 带超时的确认对话框 | `ui.confirm` with timeout |
| `mac-system-theme.ts` | 自动切换系统主题 | `setTheme`, `exec` |

### 复杂扩展

| 扩展 | 说明 | 关键 API |
|------|------|----------|
| `plan-mode/` | 完整计划模式实现 | 所有事件类型, `registerCommand`, `registerShortcut`, `registerFlag` |
| `preset.ts` | 可保存的配置预设 | `registerCommand`, `registerShortcut`, `setModel`, `appendEntry` |
| `tools.ts` | 工具开关 UI | `registerCommand`, `setActiveTools` |

### 远程与沙箱扩展

| 扩展 | 说明 | 关键 API |
|------|------|----------|
| `ssh.ts` | SSH 远程执行 | `registerFlag`, `on("user_bash")` |
| `interactive-shell.ts` | 持久化 shell 会话 | `on("user_bash")` |
| `sandbox/` | 沙箱化工具执行 | Tool operations |
| `subagent/` | 生成子 Agent | `registerTool`, `exec` |

### 游戏扩展

| 扩展 | 说明 | 关键 API |
|------|------|----------|
| `snake.ts` | 贪吃蛇游戏 | `registerCommand`, `ui.custom` |
| `space-invaders.ts` | 太空侵略者 | `registerCommand`, `ui.custom` |
| `doom-overlay/` | Doom 游戏覆盖层 | `ui.custom` with overlay |

### Provider 扩展

| 扩展 | 说明 | 关键 API |
|------|------|----------|
| `custom-provider-anthropic/` | Anthropic 代理配置 | `registerProvider` |
| `custom-provider-gitlab-duo/` | GitLab Duo 集成 | `registerProvider` with OAuth |

### 消息与会话元数据

| 扩展 | 说明 | 关键 API |
|------|------|----------|
| `message-renderer.ts` | 自定义消息渲染 | `registerMessageRenderer`, `sendMessage` |
| `event-bus.ts` | 扩展间事件通信 | `pi.events.emit` |
| `session-name.ts` | 会话命名 | `setSessionName`, `getSessionName` |
| `bookmark.ts` | 书签条目 | `setLabel` |

### 其他扩展

| 扩展 | 说明 |
|------|------|
| `antigravity-image-gen.ts` | 图像生成 (Google Antigravity) |
| `inline-bash.ts` | 工具调用中的内联 bash |
| `bash-spawn-hook.ts` | bash 生成前调整命令 |

## 官方技能仓库

### [Anthropic Skills](https://github.com/anthropics/skills)

Anthropic 官方维护的技能库：

- **文档处理**：docx, pdf, pptx, xlsx 文件处理
- **Web 开发**：前端开发辅助技能

### [Pi Skills](https://github.com/badlogic/pi-skills)

PI Agent 官方技能库：

- **Web 搜索**：搜索自动化
- **浏览器自动化**：Playwright 集成
- **Google APIs**：Google 服务集成
- **转录**：语音转文字

## 内置主题

PI Agent 包含两个默认主题：

| 主题 | 说明 | 文件 |
|------|------|------|
| `dark` | 深色主题 | [dark.json](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/src/modes/interactive/theme/dark.json) |
| `light` | 浅色主题 | [light.json](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/src/modes/interactive/theme/light.json) |

### 主题市场

社区主题仓库可以通过以下方式发现：

```bash
# 搜索带 pi-package 关键词的 npm 包
npm search pi-package

# 在 GitHub 搜索
git:github.com/*/*-pi-theme
```

## Pi 包发现

### 包市场

访问 [pi.dev/packages](https://pi.dev/packages) 查看 Pi 包画廊。

### 画廊元数据

在 `package.json` 中添加预览信息：

```json
{
  "name": "my-awesome-extension",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "video": "https://example.com/demo.mp4",
    "image": "https://example.com/screenshot.png"
  }
}
```

### 推荐关键词

发布到 npm 时使用以下关键词便于发现：
- `pi-package` - 必需
- `pi-extension` - 扩展包
- `pi-skill` - 技能包
- `pi-theme` - 主题包
- `pi-prompt` - 提示模板包

## 使用示例和模式

### 社区扩展收集模式

创建自己的扩展集合：

```bash
# 项目级扩展目录
mkdir -p .pi/extensions

# 全局扩展目录
mkdir -p ~/.pi/agent/extensions

# 安装第三方扩展
pi install npm:@your-scope/pi-extension-collection
```

### 技能集合

组织技能的最佳实践：

```
.pi/skills/
├── web/
│   ├── SKILL.md          # 技能描述
│   └── scripts/
├── data-analysis/
│   └── SKILL.md
└── automation/
    └── SKILL.md
```

### 主题集合

```
.pi/themes/
├── nord.json
├── gruvbox.json
└── tokyo-night.json
```

## 扩展依赖关系

### Peer Dependencies

不需要打包的依赖（列为 peerDependencies）：

```json
{
  "peerDependencies": {
    "@mariozechner/pi-coding-agent": "*",
    "@mariozechner/pi-ai": "*",
    "@mariozechner/pi-tui": "*",
    "typebox": "*"
  }
}
```

### 第三方运行时依赖

需要打包的依赖放在 `dependencies` 中，然后添加到 `bundledDependencies`：

```json
{
  "dependencies": {
    "axios": "^1.0.0"
  },
  "bundledDependencies": ["axios"]
}
```

## 包配置示例

### 完整包配置

```json
{
  "name": "my-pi-toolkit",
  "version": "1.0.0",
  "description": "My personal PI Agent toolkit",
  "keywords": ["pi-package", "pi-extension"],
  "license": "MIT",
  "dependencies": {
    "axios": "^1.0.0"
  },
  "bundledDependencies": ["axios"],
  "peerDependencies": {
    "@mariozechner/pi-coding-agent": "*",
    "typebox": "*"
  },
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"],
    "video": "https://example.com/demo.mp4",
    "image": "https://example.com/screenshot.png"
  }
}
```

### 过滤加载示例

在设置中过滤包的资源：

```json
{
  "packages": [
    "npm:base-pkg",
    {
      "source": "npm:my-package",
      "extensions": ["extensions/*.ts", "!extensions/legacy.ts"],
      "skills": [],
      "prompts": ["prompts/review.md"],
      "themes": ["+themes/legacy.json"]
    }
  ]
}
```

## 资源优先级

当存在冲突时，资源按以下优先级加载（后者覆盖前者）：

1. 全局目录 (`~/.pi/agent/`)
2. 项目目录 (`.pi/`)
3. 包资源 (packages)
4. 显式配置 (`settings.json`)

## 最佳实践

1. **扩展安全**：仅从可信来源安装扩展
2. **技能审查**：技能可以执行任意操作，使用前要审查内容
3. **版本控制**：固定包版本以避免意外更新
4. **目标命名空间**：避免命名冲突，使用适当的前缀
5. **文档**：为扩展和技能编写清晰的使用说明
