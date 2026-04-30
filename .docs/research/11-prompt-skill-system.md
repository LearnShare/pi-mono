# Prompt / Skill 系统研究

> 基于 pi-mono 源码深度分析
> 研究主题：02-detailed-research-plan.md 主题4

---

## 执行摘要

本文档深入分析 pi-coding-agent 的 Prompt 分层、组装机制，以及 Skill 的发现、验证和使用方式。

---

## 1. Prompt 分层结构

### 1.1 系统提示构建

**入口：** `buildSystemPrompt()`

```typescript
// packages/coding-agent/src/core/system-prompt.ts
export function buildSystemPrompt(options: BuildSystemPromptOptions): string {
  // 返回完整的系统提示字符串
}
```

### 1.2 Prompt 层叠顺序

```
1. 系统提示（默认或自定义）
        ↓
2. APPEND_SYSTEM.md 追加内容
        ↓
3. 项目上下文（AGENTS.md / CLAUDE.md）
        ↓
4. Skill 列表（仅当 read 工具可用时）
        ↓
5. 日期和工作目录
```

### 1.3 默认系统提示结构

```text
You are an expert coding assistant operating inside pi, a coding agent harness.
You help users by reading files, executing commands, editing code, and writing new files.

Available tools:
- read: Read file contents
- bash: Execute shell commands
- edit: Edit files
- write: Write files

In addition to the tools above, you may have access to other custom tools...

Guidelines:
- Be concise in your responses
- Show file paths clearly when working with files
- ...

Pi documentation:
- Main documentation: ...
- Additional docs: ...

# Project Context
Project-specific instructions and guidelines:

## AGENTS.md
[内容]

# Skills
[Skill 列表]

Current date: 2026-04-29
Current working directory: /path/to/cwd
```

---

## 2. 自定义 Prompt 处理

### 2.1 自定义方式

| 方式 | 位置 | 效果 |
|------|------|------|
| `--system-prompt` CLI | 命令行 | 替换默认系统提示 |
| `.pi/SYSTEM.md` | 项目级 | 替换默认系统提示 |
| `~/.pi/agent/SYSTEM.md` | 全局 | 替换默认系统提示 |
| `APPEND_SYSTEM.md` | 项目/全局 | 追加到系统提示 |

### 2.2 优先级

```
自定义 system prompt > APPEND_SYSTEM.md 追加 > 项目上下文 > 技能 > 日期/cwd
```

### 2.3 自定义时的处理

当设置了 `customPrompt`（通过任意自定义方式）时：
- 默认系统提示被替换
- `appendSystemPrompt` 仍被追加
- 项目上下文文件仍被追加
- Skill 部分仍被追加（前提是 `read` 工具可用）
- 日期和工作目录仍被追加

---

## 3. 项目上下文文件

### 3.1 发现位置

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/AGENTS.md` | 全局 |
| `.pi/AGENTS.md` | 项目级 |
| `.agents/AGENTS.md` | 项目级及祖先目录 |
| `CLAUDE.md` | 与 AGENTS.md 同义 |

### 3.2 扫描顺序

从当前工作目录向上遍历到 git 根目录，找到第一个存在的上下文文件。

### 3.3 追加格式

```markdown
# Project Context

Project-specific instructions and guidelines:

## /path/to/AGENTS.md

[文件内容]
```

---

## 4. Skill 系统

### 4.1 Skill 定义

遵循 [Agent Skills 标准](https://agentskills.io/specification)：

```typescript
interface Skill {
  name: string;              // 技能名称（目录名）
  description: string;       // 描述
  filePath: string;          // SKILL.md 路径
  baseDir: string;           // 基础目录
  sourceInfo: SourceInfo;
  disableModelInvocation: boolean;  // 禁止自动调用
}
```

### 4.2 Frontmatter 格式

```markdown
---
name: brave-search
description: Web search and content extraction via Brave Search API.
disable-model-invocation: true
---

# Brave Search

## Setup
\```bash
cd /path/to/brave-search && npm install
\```

## Search
\```bash
./search.js "query"
\```
```

### 4.3 发现位置

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/skills/` | 全局 |
| `~/.agents/skills/` | 全局 |
| `.pi/skills/` | 项目级 |
| `.agents/skills/` | 项目级及祖先目录 |
| 包中的 `skills/` 目录 | 包级 |
| CLI `--skill <path>` | 显式指定 |

### 4.4 验证规则

- `name` 必须与父目录名匹配
- `name` 最大 64 字符
- `description` 最大 1024 字符
- 必须包含 `SKILL.md` 文件

---

## 5. Skill 使用方式

### 5.1 内联展开

通过 `/skill:name` 命令调用：

```
/skill brave-search Search for pi agent documentation
```

展开后的 XML 格式：

```xml
<skill name="brave-search" location="~/.pi/agent/skills/brave-search/">
# Brave Search

## Setup
...

## Search
./search.js "Search for pi agent documentation"
</skill>
```

### 5.2 参数传递

```
/skill:name arg1 arg2 arg3
```

### 5.3 系统提示追加

**条件：** 工具集中必须有 `read` 工具可用

```typescript
// 只有当 hasRead && skills.length > 0 时才追加
if (hasRead && skills.length > 0) {
  prompt += formatSkillsForPrompt(skills);
}
```

### 5.4 禁止自动调用

设置 `disableModelInvocation: true` 的 Skill：
- 不会追加到系统提示
- 只能通过 `/skill:name` 手动调用

---

## 6. Prompt Template 系统

### 6.1 定义

Prompt Template 是可复用的 Markdown 提示模板，通过 `/name` 扩展。

### 6.2 发现位置

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/prompts/*.md` | 全局 |
| `.pi/prompts/*.md` | 项目级 |

### 6.3 Frontmatter 格式

```markdown
---
description: Review staged git changes
---
Review the staged changes (`git diff --cached`). Focus on:
- Bugs and logic errors
- Security issues
- Error handling gaps
```

### 6.4 带参数模板

```markdown
---
description: Create a component
argument-hint: "<ComponentName> [features...]"
---
Create a React component named $1 with features: $@
```

### 6.5 参数替换规则

| 占位符 | 说明 | 示例 |
|--------|------|------|
| `$1`, `$2`, ... | 位置参数 | `$1` → 第一个参数 |
| `$@` | 所有参数 | `$@` → arg1 arg2 arg3 |
| `$ARGUMENTS` | 所有参数（新语法） | 同 `$@` |
| `${@:N}` | 从第 N 个开始的参数 | `${@:2}` → 第二个及之后 |
| `${@:N:L}` | 从第 N 个开始，长度 L | `${@:1:3}` → 前 3 个 |

---

## 7. 关键源码位置

| 文件 | 作用 |
|------|------|
| `packages/coding-agent/src/core/system-prompt.ts` | 系统提示构建 |
| `packages/coding-agent/src/core/prompt-templates.ts` | Prompt Template 加载与展开 |
| `packages/coding-agent/src/core/skills.ts` | Skill 加载与格式化 |
| `packages/coding-agent/src/core/resource-loader.ts` | 资源发现 |

---

## 8. 研究问题解答

### Q1: 最终的 Prompt 顺序是什么？

```
自定义 system prompt → APPEND → 项目上下文 → Skill → 日期/cwd
```

### Q2: 自定义 system prompt 的优先级？

```
CLI --system-prompt > .pi/SYSTEM.md > ~/.pi/agent/SYSTEM.md
```

### Q3: 什么条件下 Skill 追加到系统提示？

必须有 `read` 工具可用。

### Q4: 参数替换的完整规则？

- `$1`, `$2`: 位置参数
- `$@`, `$ARGUMENTS`: 所有参数
- `${@:N}`: 第 N 个及之后
- `${@:N:L}`: 从第 N 个开始 L 个

---

## 9. 流程图

### 9.1 Prompt 组装流程

```
buildSystemPrompt()
       │
       ▼
┌─────────────────────┐
│ 检查 customPrompt   │
│ 是否存在？          │
└────────┬────────────┘
         ▼
    ┌────┴────┐
    │ 是      │ 否
    └────┬────┘
         ▼         ▼
┌─────────────────┐ ┌─────────────────┐
│ 构建默认提示    │ │ 使用自定义提示  │
│ - tools list   │ │ + appendSection │
│ - guidelines   │ │ + context files │
│ - docs links   │ │ + skills        │
└────────┬────────┘ │ + date/cwd     │
         │          └────────────────┘
         ▼
┌─────────────────────┐
│ 追加 appendSection │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 追加项目上下文      │
│ (AGENTS.md)        │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 检查 read 工具？    │
│ 追加 skills        │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 追加日期和工作目录  │
└────────┬────────────┘
         ▼
    返回完整 Prompt
```

### 9.2 Skill 使用流程

```
/skill:name [args]
       │
       ▼
┌─────────────────────┐
│ 解析 skill name    │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 加载 SKILL.md       │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 替换参数            │
│ ($1, $2, $@, etc.)  │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 展开为 XML 格式    │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 添加到用户消息      │
└─────────────────────┘
```

---

*本文档为研究产出，后续可根据需要进一步补充和修正。*