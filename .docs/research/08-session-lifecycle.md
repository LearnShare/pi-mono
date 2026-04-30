# Session 生命周期与消息流研究

> 基于 pi-mono 源码深度分析
> 研究主题：02-detailed-research-plan.md 主题1

---

## 执行摘要

本文档深入分析 pi-coding-agent 的 Session 生命周期机制，涵盖会话创建、消息流转、压缩（Compaction）、分支（Fork/Clone）和树结构导航等核心功能。

---

## 1. Session 数据结构

### 1.1 Session 文件格式

Session 使用 JSONL（JSON Lines）格式存储，每个条目占一行。

**文件头（第一行）：**

```typescript
interface SessionHeader {
  type: "session";
  version?: number;  // v1 sessions 没有此字段
  id: string;
  timestamp: string;
  cwd: string;
  parentSession?: string;  // 父会话路径（用于 fork）
}
```

### 1.2 条目类型

| 类型 | 说明 |
|------|------|
| `message` | 消息条目，包含 AgentMessage |
| `thinking_level_change` | 思考级别变更 |
| `model_change` | 模型变更 |
| `compaction` | 压缩摘要 |
| `branch_summary` | 分支摘要 |
| `custom` | 扩展自定义数据 |

**消息条目结构：**

```typescript
interface SessionMessageEntry extends SessionEntryBase {
  type: "message";
  message: AgentMessage;
}
```

**压缩条目结构：**

```typescript
interface CompactionEntry<T = unknown> extends SessionEntryBase {
  type: "compaction";
  summary: string;              // LLM 生成的摘要
  firstKeptEntryId: string;     // 保留的第一条消息 ID
  tokensBefore: number;         // 压缩前的 token 数
  details?: T;                  // 扩展自定义数据
  fromHook?: boolean;           // 是否由扩展生成
}
```

---

## 2. Session 生命周期

### 2.1 创建（Create）

**入口：** `SessionManager.create()`

```typescript
// packages/coding-agent/src/core/session-manager.ts
function create(...): SessionManager {
  // 1. 写入文件头
  const header: SessionHeader = {
    type: "session",
    version: CURRENT_SESSION_VERSION,
    id: uuidv7(),
    timestamp: new Date().toISOString(),
    cwd,
    parentSession,
  };

  // 2. 初始化 AgentSession
  const agentSession = new AgentSession(sessionManager, ...);
}
```

**关键属性：**
- `id`: UUID v7 格式
- `timestamp`: ISO 8601 格式
- `cwd`: 工作目录
- `parentSession`: 父会话路径（fork 时设置）

### 2.2 继续（Continue）

**入口：** `SessionManager.continueRecent()`

```typescript
// 从 sessionDir 中找到最近的会话文件
// 反序列化消息到 AgentMessage[]
// 重建分支树结构
```

### 2.3 分支（Fork/Clone）

**入口：** `AgentSessionRuntime.fork()`

**Fork vs Clone 区别：**

| 操作 | 行为 |
|------|------|
| `/fork` (position: "before") | 从选中用户消息之前创建分支，丢弃后续消息 |
| `/clone` (position: "at") | 复制当前活跃分支到新会话 |

```typescript
// fork 核心逻辑
async fork(entryId: string, options?: {
  position?: "before" | "at";
  withSession?: (ctx) => Promise<void>;
}) {
  // 1. 触发 session_before_fork 事件
  const beforeResult = await this.emitBeforeFork(entryId, { position });
  if (beforeResult.cancelled) return { cancelled: true };

  // 2. 确定目标叶子节点
  if (position === "at") {
    targetLeafId = selectedEntry.id;  // 克隆
  } else {
    targetLeafId = selectedEntry.parentId;  // Fork
  }

  // 3. 创建分支会话
  const forkedSessionPath = sourceManager.createBranchedSession(targetLeafId);
}
```

---

## 3. 消息流（Message Flow）

### 3.1 消息类型

pi 定义了多种自定义消息类型：

| 消息类型 | 说明 |
|----------|------|
| `bashExecution` | Bash 命令执行结果 |
| `custom` | 扩展自定义消息 |
| `branchSummary` | 分支摘要（树导航时生成） |
| `compactionSummary` | 压缩摘要 |

### 3.2 Turn 边界

一个 Turn 定义为：
```
User Message → 零或多个 Tool Calls → Assistant Stop
```

**事件顺序：**
```
turn_start → tool_call → tool_result → turn_end
```

### 3.3 消息队列（Steer/FollowUp）

| 操作 | 说明 |
|------|------|
| `steer()` | 排队转向消息（当前工具调用完成后发送） |
| `followUp()` | 排队后续消息（助手全部工作完成后发送） |

---

## 4. 压缩（Compaction）

### 4.1 触发条件

```typescript
// 触发阈值计算
contextTokens > contextWindow - reserveTokens
```

**配置参数：**

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

### 4.2 压缩流程

```typescript
async compact(customInstructions?: string): Promise<CompactionResult> {
  // 1. 中断当前操作
  this._disconnectFromAgent();
  await this.abort();

  // 2. 准备压缩
  const preparation = prepareCompaction(pathEntries, settings);

  // 3. 触发扩展事件（可选）
  const result = await this._extensionRunner.emit({
    type: "session_before_compact",
    preparation,
    branchEntries: pathEntries,
    customInstructions,
  });

  // 4. 调用 LLM 生成摘要
  const compactionResult = await compact(
    preparation,
    this.model,
    apiKey,
    headers,
    customInstructions,
    signal,
    thinkingLevel,
  );

  // 5. 写入压缩条目
  const entry: CompactionEntry = {
    type: "compaction",
    summary: compactionResult.summary,
    firstKeptEntryId: compactionResult.firstKeptEntryId,
    tokensBefore: compactionResult.tokensBefore,
  };
}
```

### 4.3 扩展自定义压缩

扩展可以通过 `session_before_compact` 事件自定义压缩：

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

## 5. 树结构导航

### 5.1 会话树

Session 使用树结构管理多个分支：

```
Session
├── Entry 1 (user message)
├── Entry 2 (assistant message)
├── Entry 3 (tool call)
├── Entry 4 (tool result)
├── Entry 5 (assistant message)
│   ├── Branch A
│   │   ├── Entry 6 (user message)
│   │   └── Entry 7 (assistant message)
│   └── Branch B
│       ├── Entry 8 (user message)
│       └── Entry 9 (assistant message)
```

### 5.2 导航事件

```typescript
// 导航前
pi.on("session_before_tree", async (event, ctx) => {
  // event.preparation.targetId - 导航目标
  // event.preparation.oldLeafId - 当前位置
  // event.preparation.entriesToSummarize - 将摘要的条目
  return { cancel: true };
});

// 导航后
pi.on("session_tree", async (event, ctx) => {
  // event.newLeafId, event.oldLeafId
  // event.summaryEntry, event.fromExtension
});
```

---

## 6. 关键源码位置

| 文件 | 作用 |
|------|------|
| `packages/coding-agent/src/core/agent-session.ts` | Session 管理、消息队列、事件发射 |
| `packages/coding-agent/src/core/session-manager.ts` | JSONL 读写、树结构操作 |
| `packages/coding-agent/src/core/compaction/*.ts` | 压缩/分支摘要生命周期 |
| `packages/coding-agent/src/core/messages.ts` | 自定义消息类型转换 |
| `packages/coding-agent/src/core/agent-session-runtime.ts` | Fork/Clone/Navigate 实现 |

---

## 7. 研究问题解答

### Q1: Session 创建时头部包含什么？

头部包含：`type`、`version`、`id`、`timestamp`、`cwd`、`parentSession`

### Q2: Fork 与 Clone 的区别？

- **Fork**: 从旧消息点创建，丢弃未合并到摘要的内容
- **Clone**: 复制活跃分支到新会话，保留所有内容

### Q3: 压缩生成的 LLM 调用是同步还是异步？

是异步调用，结果通过 `compact()` 函数返回后合并到会话。

### Q4: 压缩后之前的消息是否保留？

压缩后，之前的所有消息仍保留在文件中，但只有 `firstKeptEntryId` 之后的消息会发送给 LLM。压缩摘要作为桥梁，让 LLM 理解历史上下文。

---

## 8. 流程图

### 8.1 Session 完整生命周期

```
┌─────────────────┐
│   用户启动 pi    │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 创建/继续会话    │
│ session_start   │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 用户发送消息     │
│    input        │
└────────┬────────┘
         ▼
┌─────────────────┐
│  Agent Loop     │
│  turn_start     │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 调用 LLM        │
│ before_provider │
└────────┬────────┘
         ▼
    ┌────┴────┐
    │ LLM 响应 │
    └────┬────┘
         ▼
    ┌────┴────┐
    │ 工具调用 │
    │ tool_   │
    │ call    │
    └────┬────┘
         ▼
    ┌────┴────┐
    │ 工具执行 │
    │ tool_   │
    │ result  │
    └────┬────┘
         ▼
    ┌────┴────┐
    │ turn_end │
    └────┬────┘
         ▼
┌─────────────────┐
│ 检查压缩条件     │
│ context > limit │
└────────┬────────┘
         ▼
    ┌────┴────┐
    │ 触发压缩 │ ──→ compaction 事件
    └────┬────┘
         ▼
┌─────────────────┐
│  用户切换会话    │
│ session_before  │
│ _switch         │
└────────┬────────┘
         ▼
┌─────────────────┐
│ session_shutdown│
└─────────────────┘
```

---

*本文档为研究产出，后续可根据需要进一步补充和修正。*