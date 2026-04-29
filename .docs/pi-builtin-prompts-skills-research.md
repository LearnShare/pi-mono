# Pi Agent: 内置提示词与 Skill 研究报告

> 基于 pi-mono 源码深度分析（`packages/ai`, `packages/agent`, `packages/coding-agent`）
> 最后更新: 2026-04-29

---

## 执行摘要

Pi Agent **不包含任何内置的 Skill 包**。Skill 系统完全基于文件系统：用户从配置目录（`~/.pi/agent/skills/` 等）或通过扩展/包加载 skill 文件。核心产品本身不捆绑任何 SKILL.md。

然而，Pi 在源码中嵌入了大量**结构化的内置提示词**，覆盖系统提示、压缩摘要、分支摘要、工具描述、斜杠命令、认证引导等多个层面。本文档逐一列出。

---

## 1. 主系统提示

**文件：** `packages/coding-agent/src/core/system-prompt.ts:131-147`

这是发送给 LLM 的主要系统提示。注意：工具列表和指南动态生成，但核心结构固定。

```text
You are an expert coding assistant operating inside pi, a coding agent harness. You help users by reading files, executing commands, editing code, and writing new files.

Available tools:
{toolsList}

In addition to the tools above, you may have access to other custom tools depending on the project.

Guidelines:
{guidelines}

Pi documentation (read only when the user asks about pi itself, its SDK, extensions, themes, skills, or TUI):
- Main documentation: {readmePath}
- Additional docs: {docsPath}
- Examples: {examplesPath} (extensions, custom tools, SDK)
- When asked about: extensions (docs/extensions.md, examples/extensions/), themes (docs/themes.md), skills (docs/skills.md), prompt templates (docs/prompt-templates.md), TUI components (docs/tui.md), keybindings (docs/keybindings.md), SDK integrations (docs/sdk.md), custom providers (docs/custom-provider.md), adding models (docs/models.md), pi packages (docs/packages.md)
- When working on pi topics, read the docs and examples, and follow .md cross-references before implementing
- Always read pi .md files completely and follow links to related docs (e.g., tui.md for TUI API details)
```

### 始终包含的准则

```text
- Be concise in your responses
- Show file paths clearly when working with files
```

### 文件探索准则（根据可用工具动态选择）

```text
# 当 bash + (grep/find/ls) 同时存在时：
- Prefer grep/find/ls tools over bash for file exploration (faster, respects .gitignore)

# 当只有 bash 没有 grep/find/ls 时：
- Use bash for file operations like ls, rg, find
```

### 尾部追加内容

```text
Current date: {date}
Current working directory: {promptCwd}
```

### 项目上下文（存在 AGENTS.md/CLAUDE.md 时追加）

```text

# Project Context

Project-specific instructions and guidelines:

## {filePath}
{content}
```

### 自定义系统提示替换

当设置了 `customPrompt`（通过 `--system-prompt` CLI、`.pi/SYSTEM.md` 或扩展的 `before_agent_start`），整个默认提示被替换，但：
- `appendSystemPrompt` 仍被追加
- 项目上下文文件仍被追加
- Skill 部分仍被追加（前提是 `read` 工具可用）
- 日期和工作目录仍被追加

---

## 2. 工具描述与准则

每个内置工具定义了 `description`（工具 schema 中的描述）、`promptSnippet`（系统提示工具列表中的单行摘要）和可选的 `promptGuidelines`（系统提示准则列表中的条目）。

### 2.1 read 工具

**文件：** `packages/coding-agent/src/core/tools/read.ts:128-132`

```typescript
description: `Read the contents of a file. Supports text files and images (jpg, png, gif, webp). Images are sent as attachments. For text files, output is truncated to ${DEFAULT_MAX_LINES} lines or ${DEFAULT_MAX_BYTES / 1024}KB (whichever is hit first). Use offset/limit for large files. When you need the full file, continue with offset until complete.`
promptSnippet: "Read file contents"
promptGuidelines: ["Use read to examine files instead of cat or sed."]
```

### 2.2 bash 工具

**文件：** `packages/coding-agent/src/core/tools/bash.ts:281-284`

```typescript
description: `Execute a bash command in the current working directory. Returns stdout and stderr. Output is truncated to last ${DEFAULT_MAX_LINES} lines or ${DEFAULT_MAX_BYTES / 1024}KB (whichever is hit first). If truncated, full output is saved to a temp file. Optionally provide a timeout in seconds.`
promptSnippet: "Execute bash commands (ls, grep, find, etc.)"
promptGuidelines: (无)
```

### 2.3 write 工具

**文件：** `packages/coding-agent/src/core/tools/write.ts:189-192`

```typescript
description: "Write content to a file. Creates the file if it doesn't exist, overwrites if it does. Automatically creates parent directories."
promptSnippet: "Create or overwrite files"
promptGuidelines: ["Use write only for new files or complete rewrites."]
```

### 2.4 edit 工具

**文件：** `packages/coding-agent/src/core/tools/edit.ts:296-305`

```typescript
description: "Edit a single file using exact text replacement. Every edits[].oldText must match a unique, non-overlapping region of the original file. If two changes affect the same block or nearby lines, merge them into one edit instead of emitting overlapping edits. Do not include large unchanged regions just to connect distant changes."
promptSnippet: "Make precise file edits with exact text replacement, including multiple disjoint edits in one call"
promptGuidelines: [
  "Use edit for precise changes (edits[].oldText must match exactly)",
  "When changing multiple separate locations in one file, use one edit call with multiple entries in edits[] instead of multiple edit calls",
  "Each edits[].oldText is matched against the original file, not after earlier edits are applied. Do not emit overlapping or nested edits. Merge nearby changes into one edit.",
  "Keep edits[].oldText as small as possible while still being unique in the file. Do not pad with large unchanged regions.",
]
```

### 2.5 grep 工具

**文件：** `packages/coding-agent/src/core/tools/grep.ts:131`

```typescript
promptSnippet: "Search file contents for patterns (respects .gitignore)"
promptGuidelines: (无)
```

### 2.6 find 工具

**文件：** `packages/coding-agent/src/core/tools/find.ts:121`

```typescript
promptSnippet: "Find files by glob pattern (respects .gitignore)"
promptGuidelines: (无)
```

### 2.7 ls 工具

**文件：** `packages/coding-agent/src/core/tools/ls.ts:108`

```typescript
promptSnippet: "List directory contents"
promptGuidelines: (无)
```

---

## 3. Skill 格式化提示词

**文件：** `packages/coding-agent/src/core/skills.ts:340-366`

当加载了 skill 且 `read` 工具可用时，此 XML 块被追加到系统提示末尾：

```text

The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.
When a skill file references a relative path, resolve it against the skill directory (parent of SKILL.md / dirname of the path) and use that absolute path in tool commands.

<available_skills>
  <skill>
    <name>{skillName}</name>
    <description>{skillDescription}</description>
    <location>{skillFilePath}</location>
  </skill>
  ...
</available_skills>
```

**注意：** 设置了 `disableModelInvocation: true` 的 skill 不会出现在此处，只能通过 `/skill:name` 手动调用。

---

## 4. Skill 内联展开格式

**文件：** `packages/coding-agent/src/core/agent-session.ts:1133-1136`

当通过 `/skill:name` 手动调用 skill 时，它被展开为内联 XML 块：

```xml
<skill name="{skill.name}" location="{skill.filePath}">
References are relative to {skill.baseDir}.

{skill body content}
{args}   <!-- 可选的用户参数 -->
</skill>
```

---

## 5. 压缩（Compaction）提示词

### 5.1 压缩系统提示

**文件：** `packages/coding-agent/src/core/compaction/utils.ts:168-170`

```text
You are a context summarization assistant. Your task is to read a conversation between a user and an AI coding assistant, then produce a structured summary following the exact format specified.

Do NOT continue the conversation. Do NOT respond to any questions in the conversation. ONLY output the structured summary.
```

### 5.2 初始压缩提示词

**文件：** `packages/coding-agent/src/core/compaction/compaction.ts:454-485`

```text
The messages above are a conversation to summarize. Create a structured context checkpoint summary that another LLM will use to continue the work.

Use this EXACT format:

## Goal
[What is the user trying to accomplish? Can be multiple items if the session covers different tasks.]

## Constraints & Preferences
- [Any constraints, preferences, or requirements mentioned by user]
- [Or "(none)" if none were mentioned]

## Progress
### Done
- [x] [Completed tasks/changes]

### In Progress
- [ ] [Current work]

### Blocked
- [Issues preventing progress, if any]

## Key Decisions
- **[Decision]**: [Brief rationale]

## Next Steps
1. [Ordered list of what should happen next]

## Critical Context
- [Any data, examples, or references needed to continue]
- [Or "(none)" if not applicable]

Keep each section concise. Preserve exact file paths, function names, and error messages.
```

### 5.3 增量压缩提示词（含前次摘要更新）

**文件：** `packages/coding-agent/src/core/compaction/compaction.ts:487-524`

```text
The messages above are NEW conversation messages to incorporate into the existing summary provided in <previous-summary> tags.

Update the existing structured summary with new information. RULES:
- PRESERVE all existing information from the previous summary
- ADD new progress, decisions, and context from the new messages
- UPDATE the Progress section: move items from "In Progress" to "Done" when completed
- UPDATE "Next Steps" based on what was accomplished
- PRESERVE exact file paths, function names, and error messages
- If something is no longer relevant, you may remove it

Use this EXACT format:

## Goal
[Preserve existing goals, add new ones if the task expanded]

## Constraints & Preferences
- [Preserve existing, add new ones discovered]

## Progress
### Done
- [x] [Include previously done items AND newly completed items]

### In Progress
- [ ] [Current work - update based on progress]

### Blocked
- [Current blockers - remove if resolved]

## Key Decisions
- **[Decision]**: [Brief rationale] (preserve all previous, add new)

## Next Steps
1. [Update based on current state]

## Critical Context
- [Preserve important context, add new if needed]

Keep each section concise. Preserve exact file paths, function names, and error messages.
```

### 5.4 拆分轮次前缀压缩提示词

**文件：** `packages/coding-agent/src/core/compaction/compaction.ts:695-708`

当单轮对话过大、需要从中拆分时使用：

```text
This is the PREFIX of a turn that was too large to keep. The SUFFIX (recent work) is retained.

Summarize the prefix to provide context for the retained suffix:

## Original Request
[What did the user ask for in this turn?]

## Early Progress
- [Key decisions and work done in the prefix]

## Context for Suffix
- [Information needed to understand the retained recent work]

Be concise. Focus on what's needed to understand the kept suffix.
```

---

## 6. 分支摘要提示词

### 6.1 分支摘要提示词

**文件：** `packages/coding-agent/src/core/compaction/branch-summarization.ts:248-275`

```text
Create a structured summary of this conversation branch for context when returning later.

Use this EXACT format:

## Goal
[What was the user trying to accomplish in this branch?]

## Constraints & Preferences
- [Any constraints, preferences, or requirements mentioned]
- [Or "(none)" if none were mentioned]

## Progress
### Done
- [x] [Completed tasks/changes]

### In Progress
- [ ] [Work that was started but not finished]

### Blocked
- [Issues preventing progress, if any]

## Key Decisions
- **[Decision]**: [Brief rationale]

## Next Steps
1. [What should happen next to continue this work]

Keep each section concise. Preserve exact file paths, function names, and error messages.
```

### 6.2 分支摘要前言

**文件：** `packages/coding-agent/src/core/compaction/branch-summarization.ts:243-246`

生成的摘要注入到会话中时，前附：

```text
The user explored a different conversation branch before returning here.
Summary of that exploration:

```

---

## 7. 会话摘要消息包装

**文件：** `packages/coding-agent/src/core/messages.ts:11-24`

### 7.1 压缩摘要包装

```text
The conversation history before this point was compacted into the following summary:

<summary>
{summary}
</summary>
```

### 7.2 分支摘要包装

```text
The following is a summary of a branch that this conversation came back from:

<summary>
{summary}
</summary>
```

---

## 8. Bash 执行消息格式

**文件：** `packages/coding-agent/src/core/messages.ts:82-98`

当 bash 执行结果转换为 LLM 用户消息时：

```text
Ran `{command}`
```
后跟：
```text
{output}
```
或 `(no output)`

可选后缀：
```text
(command cancelled)
```
或
```text
Command exited with code {exitCode}
```
或
```text
[Output truncated. Full output: {fullOutputPath}]
```

---

## 9. 内置斜杠命令

**文件：** `packages/coding-agent/src/core/slash-commands.ts:18-40`

| 命令 | 描述 | 来源 |
|------|------|------|
| `/settings` | Open settings menu | 内置 |
| `/model` | Select model (opens selector UI) | 内置 |
| `/scoped-models` | Enable/disable models for Ctrl+P cycling | 内置 |
| `/export` | Export session (HTML default, or specify path: .html/.jsonl) | 内置 |
| `/import` | Import and resume a session from a JSONL file | 内置 |
| `/share` | Share session as a secret GitHub gist | 内置 |
| `/copy` | Copy last agent message to clipboard | 内置 |
| `/name` | Set session display name | 内置 |
| `/session` | Show session info and stats | 内置 |
| `/changelog` | Show changelog entries | 内置 |
| `/hotkeys` | Show all keyboard shortcuts | 内置 |
| `/fork` | Create a new fork from a previous user message | 内置 |
| `/clone` | Duplicate the current session at the current position | 内置 |
| `/tree` | Navigate session tree (switch branches) | 内置 |
| `/login` | Configure provider authentication | 内置 |
| `/logout` | Remove provider authentication | 内置 |
| `/new` | Start a new session | 内置 |
| `/compact` | Manually compact the session context | 内置 |
| `/resume` | Resume a different session | 内置 |
| `/reload` | Reload keybindings, extensions, skills, prompts, and themes | 内置 |
| `/quit` | Quit pi | 内置 |

此外，扩展可注册额外命令，skill 注册为 `/skill:name`，提示模板注册为 `/templateName`。

---

## 10. 认证引导消息

**文件：** `packages/coding-agent/src/core/auth-guidance.ts:6-25`

### 无可用模型

```text
No models available. Use /login to log into a provider via OAuth or API key. See:
  {docsPath}/providers.md
  {docsPath}/models.md
```

### 未选择模型

```text
No model selected.

Use /login to log into a provider via OAuth or API key. See:
  {docsPath}/providers.md
  {docsPath}/models.md

Then use /model to select a model.
```

### 未找到 API Key

```text
No API key found for {provider}.

Use /login to log into a provider via OAuth or API key. See:
  {docsPath}/providers.md
  {docsPath}/models.md
```

---

## 11. 交互模式快捷提示

**文件：** `packages/coding-agent/src/modes/interactive/interactive-mode.ts:599-641`

启动时显示的紧凑快捷键提示：

```text
{keyBindingHints for interrupt, clear/exit, / commands, ! bash, expand more}
Press {key} to show full startup help and loaded resources.

Pi can explain its own features and look up its docs. Ask it how to use or extend Pi.
```

展开后显示完整的快捷键表格，结构如下（键名动态替换为实际绑定）：

```markdown
**Navigation**
| Key | Action |
|-----|--------|
| {cursorUp} / {cursorDown} / {cursorLeft} / {cursorRight} | Move cursor / browse history (Up when empty) |
| {cursorWordLeft} / {cursorWordRight} | Move by word |
| {cursorLineStart} | Start of line |
| {cursorLineEnd} | End of line |
| {jumpForward} | Jump forward to character |
| {jumpBackward} | Jump backward to character |
| {pageUp} / {pageDown} | Scroll by page |

**Editing**
| Key | Action |
|-----|--------|
| {submit} | Send message |
| {newLine} | New line (Ctrl+Enter on Windows Terminal) |
| {deleteWordBackward} | Delete word backwards |
| {deleteWordForward} | Delete word forwards |
| {deleteToLineStart} | Delete to start of line |
| {deleteToLineEnd} | Delete to end of line |
| {yank} | Paste the most-recently-deleted text |
| {yankPop} | Cycle through the deleted text after pasting |
| {undo} | Undo |

**Other**
| Key | Action |
|-----|--------|
| {tab} | Path completion / accept autocomplete |
| {interrupt} | Cancel autocomplete / abort streaming |
| {clear} | Clear editor (first) / exit (second) |
| {exit} | Exit (when editor is empty) |
| {suspend} | Suspend to background |
| {cycleThinkingLevel} | Cycle thinking level |
| {cycleModelForward} / {cycleModelBackward} | Cycle models |
| {selectModel} | Open model selector |
| {expandTools} | Toggle tool output expansion |
| {toggleThinking} | Toggle thinking block visibility |
| {externalEditor} | Edit message in external editor |
| {followUp} | Queue follow-up message |
| {dequeue} | Restore queued messages |
| {pasteImage} | Paste image from clipboard |
| `/` | Slash commands |
| `!` | Run bash command |
| `!!` | Run bash command (excluded from context) |
```

---

## 12. 内置 Skill 结论

**`packages/agent` 和 `packages/coding-agent` 均不包含任何内置的 Skill。**

- `packages/agent`（`@mariozechner/pi-agent-core`）中没有任何 "skill" 的源码引用
- `packages/coding-agent`（`@mariozechner/pi-coding-agent`）中也没有任何内嵌的 SKILL.md 或 skill 注册代码
- Skill 系统是完全文件系统的：所有 skill 都从用户配置目录、项目 `.pi/skills/` 目录、或通过包/扩展安装

测试夹具目录（`packages/coding-agent/test/fixtures/skills/`）中有用于测试的示例 skill，但这些**不是**内置分发的。

---

## 13. 总结

| 类别 | 数量 | 说明 |
|------|------|------|
| 主系统提示模板 | 1 | 动态构建，含工具列表、准则、文档引用、日期/cwd |
| 内置工具描述 | 7 | read, bash, edit, write, grep, find, ls |
| 工具使用准则 | 7 条 | 分布在 4 个工具的 `promptGuidelines` 中 |
| 通用准则 | 2 条 | "Be concise", "Show file paths clearly" |
| 文件探索准则 | 2 种变体 | 取决于可用工具集 |
| Skill 格式块 | 1 个 XML 模板 | 条件追加到系统提示 |
| Skill 内联格式 | 1 个 XML 模板 | `/skill:name` 展开时使用 |
| 压缩提示词 | 4 个 | 系统提示 + 初始 + 增量 + 拆分前缀 |
| 分支摘要提示词 | 2 个 | 提示 + 前言 |
| 摘要消息包装 | 2 个 | 压缩 + 分支 |
| Bash 执行格式 | 1 个文本模板 | 含多种可选后缀 |
| 内置斜杠命令 | 21 个 | 硬编码命令列表 |
| 认证引导消息 | 3 个 | 无模型/无选择/无 Key |
| 交互模式快捷键提示 | 2 个模板 | 紧凑 + 完整表格 |
| **内置 Skill** | **0** | 完全不包含 |