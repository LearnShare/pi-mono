# Pi Agent 记忆能力研究报告

> 基于 pi-mono 源码深度分析
> 研究主题：记忆能力、相关文件/数据库及运行机制

---

## 执行摘要

本文档深入分析 pi-coding-agent 的记忆机制，涵盖上下文管理、压缩（Compaction）、分支摘要、上下文文件等核心功能。

---

## 1. 记忆体系概览

### 1.1 记忆层次

pi-agent 的记忆体系分为多个层次：

| 层次 | 说明 | 持久化 |
|------|------|--------|
| **上下文记忆** | 当前会话的活跃消息 | Session 文件 |
| **压缩记忆** | 历史消息的 LLM 摘要 | Session 文件 |
| **分支摘要** | 树导航时的历史摘要 | Session 文件 |
| **项目上下文** | AGENTS.md/CLAUDE.md | 文件系统 |
| **系统提示** | 内置提示词 | 代码/配置文件 |
| **扩展状态** | 工具 details 持久化 | Session 文件 |

### 1.2 记忆流程

```
用户消息
    │
    ▼
┌─────────────────────┐
│ buildSessionContext│ ← 从 Session 构建上下文
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 添加系统提示        │ ← buildSystemPrompt
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 添加项目上下文      │ ← AGENTS.md/CLAUDE.md
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 发送给 LLM          │
└────────┬────────────┘
         ▼
    ┌────┴────┐
    │ LLM 响应 │
    └────┬────┘
         ▼
┌─────────────────────┐
│ 写入 Session 文件   │
└─────────────────────┘
```

---

## 2. 上下文构建机制

### 2.1 buildSessionContext()

从 Session 条目构建 LLM 可用的消息列表：

```typescript
// packages/coding-agent/src/core/session-manager.ts
export function buildSessionContext(
    entries: SessionEntry[],
    leafId?: string | null,
    byId?: Map<string, SessionEntry>,
): SessionContext
```

**核心逻辑：**

1. **树遍历**：从当前叶节点向上遍历到根
2. **压缩处理**：遇到压缩条目时，插入摘要，保留压缩后的消息
3. **消息过滤**：只包含对 LLM 有意义的消息类型

### 2.2 消息类型过滤

| 条目类型 | 是否进入上下文 |
|----------|---------------|
| `message` | ✅ 是 |
| `custom_message` | ✅ 是 |
| `branch_summary` | ✅ 是 |
| `compaction` | ✅ 是（转换为摘要消息） |
| `thinking_level_change` | ❌ 否（仅影响设置） |
| `model_change` | ❌ 否（仅影响设置） |
| `custom`（扩展数据） | ❌ 否（扩展内部使用） |
| `label` | ❌ 否 |

---

## 3. 压缩机制（Compaction）

### 3.1 触发条件

```typescript
// 触发阈值
contextTokens > contextWindow - reserveTokens

// 默认配置
{
    "compaction": {
        "enabled": true,
        "reserveTokens": 16384,
        "keepRecentTokens": 20000
    }
}
```

### 3.2 压缩流程

```typescript
async compact(customInstructions?: string): Promise<CompactionResult> {
    // 1. 准备压缩
    const preparation = prepareCompaction(pathEntries, settings);

    // 2. 扩展可以自定义压缩（session_before_compact 事件）
    const extensionResult = await emit("session_before_compact", ...);

    // 3. 调用 LLM 生成摘要
    const summary = await compact(preparation, model, ...);

    // 4. 写入压缩条目
    const entry: CompactionEntry = {
        type: "compaction",
        summary: summary.text,
        firstKeptEntryId: summary.firstKeptEntryId,
        tokensBefore: summary.tokensBefore,
    };

    // 5. 追踪文件操作
    entry.details = {
        readFiles: [...fileOps.read],
        modifiedFiles: [...fileOps.edited],
    };
}
```

### 3.3 压缩保留策略

- **保留压缩后的消息**：压缩点之后的所有消息
- **保留 firstKeptEntryId 之后的消息**：确保重要上下文不丢失
- **摘要桥接历史**：压缩前的消息通过摘要传递给 LLM

### 3.4 扩展自定义压缩

扩展可以通过事件自定义压缩行为：

```typescript
pi.on("session_before_compact", async (event, ctx) => {
    // 取消压缩
    return { cancel: true };

    // 自定义摘要
    return {
        compaction: {
            summary: "Your custom summary...",
            firstKeptEntryId: event.preparation.firstKeptEntryId,
            tokensBefore: event.preparation.tokensBefore,
            details: { /* 自定义数据 */ },
        },
    };
});
```

---

## 4. 分支摘要（Branch Summary）

### 4.1 用途

当用户通过 `/tree` 导航到其他分支时，pi 会为之前的分支生成摘要，以便未来可以恢复上下文。

### 4.2 生成时机

- `/tree` 导航时
- `/fork` 创建分支时（可选）

### 4.3 摘要内容

```typescript
interface BranchSummaryEntry extends SessionEntryBase {
    type: "branch_summary";
    fromId: string;      // 分支起始点
    summary: string;      // LLM 生成的摘要
    details?: T;          // 扩展自定义数据
    fromHook?: boolean;  // 是否由扩展生成
}
```

### 4.4 使用方式

导航到其他分支后，可以选择"总结分支"将之前分支的历史压缩为摘要。

---

## 5. 项目上下文（Context Files）

### 5.1 支持的文件

| 文件名 | 作用域 | 说明 |
|--------|--------|------|
| `AGENTS.md` | 全局/项目 | 项目特定指令 |
| `CLAUDE.md` | 全局/项目 | 与 AGENTS.md 同义 |

### 5.2 发现位置

```
搜索顺序（从 cwd 向上到 git 根目录）：
1. ./.pi/AGENTS.md
2. ./.agents/AGENTS.md
3. 向上遍历直到 git 根目录
4. ~/.pi/agent/AGENTS.md (全局)
```

### 5.3 内容格式

```markdown
# Project Instructions

- Run `npm run check` after code changes.
- Do not run production migrations locally.
- Keep responses concise.

## Coding Standards
- Use TypeScript
- Prefer functional components
```

### 5.4 追加位置

项目上下文追加到系统提示的 `# Project Context` 部分：

```text
# Project Context

Project-specific instructions and guidelines:

## /path/to/project/AGENTS.md
[文件内容]
```

---

## 6. 扩展状态持久化

### 6.1 工具 details 持久化

工具可以通过 `details` 字段持久化状态：

```typescript
async execute(toolCallId, params, signal, onUpdate, ctx) {
    return {
        content: [{ type: "text", text: "Todo added" }],
        details: { todos: currentTodos, nextId: id + 1 },  // 持久化到 Session
    };
}
```

### 6.2 从历史恢复状态

```typescript
pi.on("session_start", async (_event, ctx) => {
    let todos = [];
    for (const entry of ctx.sessionManager.getBranch()) {
        if (entry.type === "message" && entry.message.toolName === "todo_manager") {
            todos = entry.message.details?.todos ?? todos;
        }
    }
    // 重建内部状态
});
```

### 6.3 自定义条目（Custom Entry）

扩展可以存储不发送给 LLM 的数据：

```typescript
pi.appendEntry("my_state", { todos: [...], count: 5 });
// 存储为：{ type: "custom", customType: "my_state", data: {...} }
```

---

## 7. 关键源码位置

| 文件 | 作用 |
|------|------|
| `packages/coding-agent/src/core/session-manager.ts` | buildSessionContext() |
| `packages/coding-agent/src/core/compaction/compaction.ts` | 压缩逻辑 |
| `packages/coding-agent/src/core/compaction/branch-summarization.ts` | 分支摘要 |
| `packages/coding-agent/src/core/resource-loader.ts` | 上下文文件加载 |
| `packages/coding-agent/src/core/system-prompt.ts` | 系统提示构建 |

---

## 8. 记忆机制流程图

### 8.1 完整记忆流程

```
用户发送消息
       │
       ▼
┌─────────────────────┐
│ buildSessionContext│
│ - 遍历树结构        │
│ - 处理压缩          │
│ - 过滤消息          │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ buildSystemPrompt  │
│ - 工具列表          │
│ - 指南             │
│ - 文档链接         │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 加载上下文文件      │
│ - AGENTS.md        │
│ - CLAUDE.md        │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 加载 Skills         │
│ (如果 read 可用)    │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 添加日期和 cwd     │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 发送 LLM           │
└────────┬────────────┘
         ▼
    ┌────┴────┐
    │ LLM 响应 │
    └────┬────┘
         ▼
┌─────────────────────┐
│ 写入 Session       │
│ - message 条目      │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ 检查压缩条件        │
│ context > limit?    │
└────────┬────────────┘
         ▼
    ┌────┴────┐
    │ 触发压缩 │
    └────┬────┘
         ▼
┌─────────────────────┐
│ LLM 生成摘要        │
│ 写入 compaction    │
└─────────────────────┘
```

---

## 9. 研究问题解答

### Q1: pi-agent 如何记住对话历史？

通过 Session 文件（JSONL）持久化所有消息，并通过 buildSessionContext() 在每次请求时构建完整上下文。

### Q2: 长对话如何处理？

当 token 数超过阈值时触发压缩（Compaction），调用 LLM 生成历史摘要，保留压缩后的消息和摘要。

### Q3: 项目特定指令如何持久化？

通过 AGENTS.md/CLAUDE.md 文件，在每次构建系统提示时加载并追加。

### Q4: 扩展如何利用记忆机制？

- 通过工具 details 持久化状态
- 通过 custom 条目存储扩展数据
- 通过 session_start 事件从历史恢复状态

### Q5: 分支切换时如何保留历史？

通过分支摘要（branch_summary）机制，在切换分支时可选生成历史摘要。

---

## 10. 记忆相关配置

### 10.1 压缩配置

```json
{
    "compaction": {
        "enabled": true,
        "reserveTokens": 16384,
        "keepRecentTokens": 20000
    }
}
```

### 10.2 会话命令

| 命令 | 说明 |
|------|------|
| `/compact` | 手动触发压缩 |
| `/compact <instructions>` | 自定义压缩提示 |
| `/tree` | 导航会话树 |
| `/fork` | 创建分支（可选择摘要） |

---

*本文档为研究产出，后续可根据需要进一步补充和修正。*