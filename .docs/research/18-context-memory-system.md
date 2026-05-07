# Pi Agent Context 与多层记忆系统研究报告

> 基于 pi-mono 源码深度分析
> 研究主题：Context 管理与多层记忆系统
> 参考研究：research/13-memory-system.md

---

## 执行摘要

本报告深入分析 pi-coding-agent 的上下文（Context）管理与多层记忆系统。系统采用五层架构设计：L0 活跃上下文（当前会话消息）、L1 压缩摘要（历史摘要）、L2 分支摘要（树导航摘要）、L3 项目上下文（AGENTS.md/CLAUDE.md）、L4 系统提示（内置提示词）。核心机制包括语义压缩（LLM 生成结构化摘要）、Token 计算与压缩阈值管理、文件操作追踪、以及通过 `session_before_compact` 事件支持扩展自定义压缩。系统通过 Session 文件（JSONL）持久化所有消息，并结合系统提示与项目上下文文件构建完整的 LLM 上下文。

---

## 1. 上下文分层架构

### 1.1 分层概览

| 层级 | 名称 | 持久化 | 描述 |
|------|------|--------|------|
| L0 | 活跃上下文 | Session 文件 | 当前会话的活跃消息，完整的用户/助手/工具消息 |
| L1 | 压缩摘要 | Session 文件 | 历史消息的 LLM 摘要，通过 Compaction 生成 |
| L2 | 分支摘要 | Session 文件 | 树导航时的历史摘要，通过 BranchSummary 生成 |
| L3 | 项目上下文 | 文件系统 | AGENTS.md/CLAUDE.md，通过 ResourceLoader 加载 |
| L4 | 系统提示 | 代码/配置文件 | 内置提示词，通过 buildSystemPrompt 构建 |

### 1.2 消息流架构

```
用户消息
    │
    ▼
buildSessionContext()  ────────────────────────────── L0 活跃上下文
    │  (session-manager.ts:315-422)
    │   - 树遍历：从当前叶节点向上到根
    │   - 压缩处理：遇到压缩条目时插入摘要
    │   - 消息过滤：只包含有效消息类型
    ▼
buildSystemPrompt()   ─────────────────────────────── L4 系统提示
    │  (system-prompt.ts:28-172)
    │   - 工具列表与指南
    │   - 文档链接
    ▼
项目上下文加载    ─────────────────────────────────────── L3 项目上下文
    │  (resource-loader.ts)
    │   - AGENTS.md/CLAUDE.md
    ▼
发送 LLM
    │
    ▼
    ┌────┴────┐
    │ LLM 响应 │
    └────┬────┘
         ▼
Session 持久化（session-manager.ts）
         │
         ▼
    检查压缩条件
    contextTokens > contextWindow - reserveTokens
```

---

## 2. L0 活跃上下文

### 2.1 构建机制

`buildSessionContext()` 函数（session-manager.ts:315-422）负责从 Session 条目构建 LLM 可用的消息列表：

```typescript
// session-manager.ts:315-422
export function buildSessionContext(
    entries: SessionEntry[],
    leafId?: string | null,
    byId?: Map<string, SessionEntry>,
): SessionContext
```

**核心逻辑：**

1. **树遍历**：从当前叶节点（leaf）向上遍历到根节点
2. **设置提取**：提取 thinkingLevel 和 model 配置
3. **压缩处理**：遇到压缩条目时，先插入摘要，再插入保留的消息
4. **消息过滤**：只包含对 LLM 有意义的条目类型

### 2.2 消息类型过滤

| 条目类型 | 是否进入上下文 | 说明 |
|----------|----------------|------|
| `message` | ✅ 是 | 用户/助手/工具结果消息 |
| `custom_message` | ✅ 是 | 自定义消息（扩展注入） |
| `branch_summary` | ✅ 是 | 分支摘要消息 |
| `compaction` | ✅ 是 | 转换为摘要消息（compactionSummary） |
| `thinking_level_change` | ❌ 否 | 仅影响设置 |
| `model_change` | ❌ 否 | 仅影响设置 |
| `custom`（扩展数据） | ❌ 否 | 扩展内部使用 |
| `label` | ❌ 否 | 用户标签 |

### 2.3 源码位置

- **文件**：`packages/coding-agent/src/core/session-manager.ts`
- **行号**：315-422（buildSessionContext 函数）
- 导出函数 `parseSessionEntries`（284-299）
- 辅助函数 `getLatestCompactionEntry`（301-308）

---

## 3. L1 压缩摘要（Compaction）

### 3.1 压缩触发条件

```typescript
// compaction.ts:219-222
export function shouldCompact(
    contextTokens: number, 
    contextWindow: number, 
    settings: CompactionSettings
): boolean {
    if (!settings.enabled) return false;
    return contextTokens > contextWindow - settings.reserveTokens;
}
```

**默认配置**（agent-session.ts:121-125）：

```typescript
export const DEFAULT_COMPACTION_SETTINGS: CompactionSettings = {
    enabled: true,
    reserveTokens: 16384,    // 16K tokens 保留给 prompt + LLM 响应
    keepRecentTokens: 20000, // 保留最近 20K tokens
};
```

### 3.2 压缩流程

```typescript
// compaction.ts:717-797
async compact(
    preparation: CompactionPreparation,
    model: Model<any>,
    apiKey: string,
    headers?: Record<string, string>,
    customInstructions?: string,
    signal?: AbortSignal,
    thinkingLevel?: ThinkingLevel,
): Promise<CompactionResult>
```

**完整流程：**

1. **准备压缩**（prepareCompaction）：
   - 查找上一个压缩点
   - 计算切割点（findCutPoint）
   - 提取消息和文件操作

2. **生成摘要**（generateSummary）：
   - 使用 LLM 生成结构化摘要
   - 格式：Goal、Constraints、Progress、Next Steps 等

3. **合并文件操作**：
   - 提取 read/edit/write 操作
   - 追加到摘要末尾（XML 格式）

4. **保存结果**：
   - 写入 compaction 条目
   - 记录 firstKeptEntryId

### 3.3 压缩摘要格式

压缩摘要使用结构化格式（compaction.ts:454-485）：

```markdown
## Goal
[用户尝试完成的任务]

## Constraints & Preferences
- [约束和偏好]

## Progress
### Done
- [x] 已完成任务

### In Progress
- [ ] 当前工作

### Blocked
- [阻塞因素]

## Key Decisions
- **[决策]**: 理由

## Next Steps
1. 下一步操作

## Critical Context
- 关键上下文
```

### 3.4 Token 计算与阈值管理

**Token 估算**（compaction.ts:232-290）：

采用字符/4 的启发式估算（保守估计）：
- 用户消息：content.length / 4
- 助手消息：text + thinking + toolCall.length / 4
- 工具结果：content.length / 4（图片算 4800）

**上下文 Token 估算**（compaction.ts:186-214）：

```typescript
export function estimateContextTokens(messages: AgentMessage[]): ContextUsageEstimate {
    // 使用最后一个助手消息的 usage
    // 如果有消息在 usage 之后，估算其 token
    // 返回：tokens、usageTokens、trailingTokens
}
```

**切割点检测**（compaction.ts:386-448）：

```typescript
export function findCutPoint(
    entries: SessionEntry[],
    startIndex: number,
    endIndex: number,
    keepRecentTokens: number,
): CutPointResult
```

- 从最新消息向后遍历
- 累积估算 token 数
- 在有效切割点（user/assistant/custom）切割
- 不会在 toolResult 切割（必须跟随 toolCall）

### 3.5 源码位置

- **主文件**：`packages/coding-agent/src/core/compaction/compaction.ts`
- **关键函数**：
  - `shouldCompact`（219-222）
  - `estimateTokens`（232-290）
  - `estimateContextTokens`（186-214）
  - `findCutPoint`（386-448）
  - `generateSummary`（530-590）
  - `compact`（717-797）
  - `prepareCompaction`（614-689）

---

## 4. L2 分支摘要（Branch Summary）

### 4.1 用途

��用户通过 `/tree` 导航到其他分支时，为之前的分支生成摘要，以便未来可以恢复上下文。

### 4.2 生成时机

- `/tree` 导航时
- `/fork` 创建分支时（可选）

### 4.3 摘要生成

**入口收集**（branch-summarization.ts:98-136）：

```typescript
export function collectEntriesForBranchSummary(
    session: ReadonlySessionManager,
    oldLeafId: string | null,
    targetId: string,
): CollectEntriesResult
```

- 查找旧叶子到目标位置的最近公共祖先
- 收集祖先到叶子之间的条目
- 返回条目列表和公共祖先 ID

**消息准备**（branch-summarization.ts:185-237）：

```typescript
export function prepareBranchEntries(
    entries: SessionEntry[], 
    tokenBudget: number = 0
): BranchPreparation
```

- 从新到旧遍历消息
- 累积 token 直到预算耗尽
- 提取文件操作

### 4.4 分支摘要格式

```markdown
## Goal
[分支目标]

## Constraints & Preferences
- [约束]

## Progress
### Done
- [x] 完成

### In Progress
- [ ] 进行中

### Blocked
- [阻塞]

## Key Decisions
- **[决策]**: 理由

## Next Steps
1. 下一步
```

### 4.5 源码位置

- **文件**：`packages/coding-agent/src/core/compaction/branch-summarization.ts`
- **关键函数**：
  - `collectEntriesForBranchSummary`（98-136）
  - `prepareBranchEntries`（185-237）
  - `generateBranchSummary`（283-355）

---

## 5. L4 系统提示（System Prompt）

### 5.1 构建机制

`buildSystemPrompt()` 函数（system-prompt.ts:28-172）负责构建完整的系统提示：

```typescript
export function buildSystemPrompt(options: BuildSystemPromptOptions): string
```

**组成部分：**

1. **角色定义**：You are an expert coding assistant...
2. **可用工具**：从 selectedTools 参数
3. **指南**：从 promptGuidelines 参数
4. **追加提示**：appendSystemPrompt 参数
5. **项目上下文**：contextFiles 参数
6. **技能**：skills 参数
7. **日期与 CWD**：系统提示末尾

### 5.2 源码位置

- **文件**：`packages/coding-agent/src/core/system-prompt.ts`
- **关键函数**：
  - `BuildSystemPromptOptions` 接口（8-25）
  - `buildSystemPrompt`（28-172）
- **调用位置**：agent-session.ts:893-927

---

## 6. 高价值内容识别机制

### 6.1 文件操作追踪

压缩系统追踪文件操作（compaction/utils.ts:12-82）：

```typescript
export interface FileOperations {
    read: Set<string>;
    written: Set<string>;
    edited: Set<string>;
}
```

**提取逻辑**：

- 从 tool call 中提取 read/write/edit 操作
- 从之前的压缩摘要中合并文件列表
- 计算最终文件列表（去重）

### 6.2 高价值消息识别

**有效切割点**（compaction.ts:299-337）：

```typescript
function findValidCutPoints(
    entries: SessionEntry[], 
    startIndex: number, 
    endIndex: number
): number[]
```

有效切割点类型：
- user（用户消息）
- assistant（助手消息）
- bashExecution（命令执行）
- custom（自定义消息）
- branchSummary（分支摘要）
- compactionSummary（压缩摘要）

**禁止切割点**：
- toolResult（必须跟随 toolCall）

---

## 7. 语义压缩 vs 结构压缩分析

### 7.1 语义压缩

压缩使用 LLM 生成结构化摘要（compaction.ts:530-590）：

```typescript
export async function generateSummary(
    currentMessages: AgentMessage[],
    model: Model<any>,
    reserveTokens: number,
    apiKey: string,
    headers?: Record<string, string>,
    signal?: AbortSignal,
    customInstructions?: string,
    previousSummary?: string,
    thinkingLevel?: ThinkingLevel,
): Promise<string>
```

**特点**：
- 使用 `SUMMARIZATION_SYSTEM_PROMPT`（168-169）
- 将消息序列化为对话文本
- 请求 LLM ���成���构化摘要
- 支持增量更新（传入 previousSummary）
- 支持 thinking level

### 7.2 增量压缩

压缩支持增量更新（compaction.ts:487-524）：

```typescript
const UPDATE_SUMMARIZATION_PROMPT = `The messages above are NEW conversation messages...
Update the existing structured summary with new information. RULES:
- PRESERVE all existing information from the previous summary
- ADD new progress, decisions, and context from the new messages
- UPDATE the Progress section...
- UPDATE "Next Steps" based on what was accomplished
- ...
```

### 7.3 消息序列化

压缩前将消息序列化（utils.ts:109-162）：

```typescript
export function serializeConversation(messages: Message[]): string
```

格式：
```
[User]: 用户消息
[Assistant thinking]: 思考内容
[Assistant]: 回复文本
[Assistant tool calls]: toolName(arg1=json, arg2=json)
[Tool result]: 工具结果
```

**截断策略**：
- 工具结果截断到 2000 字符
- 防止摘要请求超出 token 预算

---

## 8. 摘要质量评估

### 8.1 质量保证机制

1. **结构化格式**：强制使用 Goal/Progress/Next Steps 等字段
2. **约束保留**：保留精确的文件路径、函数名、错误消息
3. **对话序列化**：直接序列化对话，防止模型继续对话
4. **增量更新**：支持合并多次压缩的摘要

### 8.2 摘要预算

```typescript
// compaction.ts:541
const maxTokens = Math.floor(0.8 * reserveTokens);

// compaction.ts:811
const maxTokens = Math.floor(0.5 * reserveTokens); // turn prefix
```

- 主摘要：0.8 * reserveTokens（约 13K tokens）
- Turn Prefix：0.5 * reserveTokens（约 8K tokens）

### 8.3 质量监控

压缩结果包含详细信息（compaction.ts:103-109）：

```typescript
export interface CompactionResult<T = unknown> {
    summary: string;
    firstKeptEntryId: string;
    tokensBefore: number;
    details?: T;  // 文件操作列表
}
```

---

## 9. 长期记忆机制

### 9.1 跨会话传递

**Session 文件持久化**：
- 所有条目存储在 JSONL 文件
- 支持会话恢复（resume）
- 支持分支（fork）

**持久化条目类型**：
- message（消息）
- custom_message（自定义消息）
- branch_summary（分支摘要）
- compaction（压缩摘要）
- model_change（模型变更）
- thinking_level_change（思考级别变更）

### 9.2 RAG 方向

目前系统未实现 RAG（检索增强生成）机制，但具备以下基础：

1. **文件操作追踪**：记录 read/edit/write 文件
2. **压缩摘要**：包含文件列表
3. **上下文文件**：AGENTS.md/CLAUDE.md

**潜在的 RAG 增强方向**：

1. **向量存储**：将压缩摘要向量化存储
2. **语义检索**：根据当前任务检索相关摘要
3. **项目文档索引**：索引项目文档供检索

---

## 10. 上下文窗口管理

### 10.1 Token 计算策略

**多级估算**（compaction.ts:135-214）：

1. **优先使用 usage**：最后一个助手消息的 totalTokens
2. **估算后续消息**：usage 之后的消息使用字符/4 估算
3. **完全估算**：无 usage 时全部使用字符/4 估算

### 10.2 压缩阈值

```typescript
// 默认配置
settings = {
    enabled: true,
    reserveTokens: 16384,     // 保留空间
    keepRecentTokens: 20000     // 保留最近消息
}

// 触发条件
shouldCompact = contextTokens > contextWindow - reserveTokens
// 即：contextTokens + reserveTokens > contextWindow
```

### 10.3 ReserveTokens 设计

ReserveTokens 预留空间用于：
- 系统提示（工具列表、指南）
- 项目上下文（AGENTS.md）
- 技能（Skills）
- 日期和 CWD
- LLM 响应空间

---

## 11. 源码位置总结

| 模块 | 文件 | 关键行号 | 功能 |
|------|------|----------|------|
| **上下文构建** | session-manager.ts | 315-422 | buildSessionContext |
| **压缩逻辑** | compaction/compaction.ts | 219-222 | shouldCompact |
| **Token 估算** | compaction/compaction.ts | 232-290 | estimateTokens |
| **切割点** | compaction/compaction.ts | 386-448 | findCutPoint |
| **生成摘要** | compaction/compaction.ts | 530-590 | generateSummary |
| **执行压缩** | compaction/compaction.ts | 717-797 | compact |
| **分支摘要** | compaction/branch-summarization.ts | 283-355 | generateBranchSummary |
| **系统提示** | system-prompt.ts | 28-172 | buildSystemPrompt |
| **消息类型** | messages.ts | 1-195 | 自定义消息类型 |
| **压缩入口** | agent-session.ts | 1599-1687 | compact 方法 |

---

## 12. 最佳实践与未来方向

### 12.1 最佳实践

1. **压缩触发阈值**：保持默认配置即可（reserveTokens: 16384）
2. **增量压缩**：系统自动处理增量摘要更新
3. **文件追踪**：依赖内置的 read/edit/write 追踪
4. **扩展自定义**：使用 `session_before_compact` 事件

### 12.2 扩展自定义压缩

```typescript
// 扩展可以自定义压缩
pi.on("session_before_compact", async (event, ctx) => {
    // 取消压缩
    return { cancel: true };

    // 自定义摘要
    return {
        compaction: {
            summary: "Custom summary...",
            firstKeptEntryId: event.preparation.firstKeptEntryId,
            tokensBefore: event.preparation.tokensBefore,
            details: { /* 自定义数据 */ },
        },
    };
});
```

### 12.3 未来方向

1. **RAG 增强**：实现基于向量存储的语义检索
2. **多模态摘要**：支持图片/代码片段的结构化提取
3. **选择性压缩**：按主题/文件压缩而非时间线性压缩
4. **上下文摘要**：摘要中包含"什么是已知的"与"什么是关键的"

---

## 13. 参考文档

- 研究产出：`research/13-memory-system.md`
- 本研究基于以下核心模块：
  - `packages/coding-agent/src/core/session-manager.ts`
  - `packages/coding-agent/src/core/compaction/compaction.ts`
  - `packages/coding-agent/src/core/compaction/branch-summarization.ts`
  - `packages/coding-agent/src/core/system-prompt.ts`
  - `packages/coding-agent/src/core/messages.ts`

---

*本文档为研究产出，基于 pi-mono 源码深度分析。后续可根据需要进一步补充和修正。*
