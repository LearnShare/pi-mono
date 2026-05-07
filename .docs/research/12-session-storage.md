# Pi Agent 对话数据存储研究报告

> 基于 pi-mono 源码深度分析
> 研究主题：对话数据存储位置与格式

---

## 执行摘要

本文档深入分析 pi-coding-agent 的对话数据存储机制，涵盖存储位置、文件格式、数据结构等核心机制。

---

## 1. 存储位置

### 1.1 默认存储目录

```
~/.pi/agent/sessions/
```

按工作目录（cwd）组织：

```
~/.pi/agent/sessions/
├── /path/to/project-a/
│   ├── session-uuid1.jsonl
│   ├── session-uuid2.jsonl
│   └── ...
├── /path/to/project-b/
│   └── ...
└── ...
```

### 1.2 源码位置

```typescript
// packages/coding-agent/src/config.ts
export function getSessionsDir(): string {
    return join(getAgentDir(), "sessions");
}

// packages/coding-agent/src/core/session-manager.ts
const sessionsDir = getSessionsDir();
```

### 1.3 配置方式

| 方式 | 说明 |
|------|------|
| 默认 | 自动存放在 `~/.pi/agent/sessions/{cwd}/` |
| CLI 参数 | `--session <path>` 指定特定会话文件 |
| 临时模式 | `--no-session` 禁用持久化，纯内存会话 |

---

## 2. 文件格式：JSONL

### 2.1 格式说明

Session 使用 **JSONL（JSON Lines）** 格式存储：
- 每行一个 JSON 对象
- 第一行是文件头（Session Header）
- 后续行是条目（Entry）

### 2.2 示例文件结构

```jsonl
{"type":"session","version":3,"id":"uuid-7","timestamp":"2026-04-29T10:00:00.000Z","cwd":"/path/to/project","parentSession":null}
{"type":"message","id":"msg-1","parentId":null,"timestamp":"2026-04-29T10:00:01.000Z","message":{"role":"user","content":[{"type":"text","text":"Hello"}],...}}
{"type":"message","id":"msg-2","parentId":"msg-1","timestamp":"2026-04-29T10:00:02.000Z","message":{"role":"assistant","content":[{...}],...}}
{"type":"compaction","id":"compact-1","parentId":"msg-5","timestamp":"2026-04-29T10:30:00.000Z","summary":"...","firstKeptEntryId":"msg-6","tokensBefore":80000,...}
```

---

## 3. 数据结构

### 3.1 文件头（Session Header）

```typescript
interface SessionHeader {
    type: "session";
    version?: number;  // v1 sessions 没有此字段
    id: string;        // UUID v7
    timestamp: string; // ISO 8601
    cwd: string;       // 工作目录
    parentSession?: string;  // 父会话路径（fork时）
}
```

### 3.2 条目类型

| 类型 | 说明 | 用途 |
|------|------|------|
| `message` | 消息条目 | 用户/助手消息、工具调用、工具结果 |
| `thinking_level_change` | 思考级别变更 | 记录思考级别变化 |
| `model_change` | 模型变更 | 记录模型切换 |
| `compaction` | 压缩摘要 | 长会话压缩后的历史摘要 |
| `branch_summary` | 分支摘要 | 树导航时生成的历史摘要 |
| `custom` | 扩展自定义数据 | 扩展状态持久化 |
| `label` | 标签 | 用户定义的树节点标签 |

### 3.3 消息条目结构

```typescript
interface SessionMessageEntry extends SessionEntryBase {
    type: "message";
    message: AgentMessage;  // 完整消息对象
}

interface AgentMessage {
    role: "user" | "assistant" | "toolResult";
    content: (TextContent | ImageContent | ThinkingContent | ToolCall)[];
    provider?: string;
    model?: string;
    usage?: Usage;
    stopReason?: StopReason;
    timestamp: number;
}
```

### 3.4 压缩条目结构

```typescript
interface CompactionEntry<T = unknown> extends SessionEntryBase {
    type: "compaction";
    summary: string;           // LLM 生成的摘要
    firstKeptEntryId: string; // 保留的第一条消息ID
    tokensBefore: number;     // 压缩前的token数
    details?: T;               // 扩展自定义数据
    fromHook?: boolean;        // 是否由扩展生成
}
```

---

## 4. 会话树结构

### 4.1 树形组织

Session 使用树结构管理多个分支：

```
session.jsonl
├── Entry 1 (user) ──────────────────────────→ root
│   ├── Entry 2 (assistant)
│   │   ├── Entry 3 (tool call)
│   │   │   └── Entry 4 (tool result)
│   │   └── Entry 5 (assistant) ──────→ Branch A (leaf)
│   │       └── Entry 6 (user)
│   │           └── Entry 7 (assistant)
│   │
│   └── Entry 5' (assistant from fork) ──────→ Branch B (leaf)
│       └── Entry 6' (user)
```

### 4.2 条目关系

- 每个条目有 `id` 和 `parentId`
- 通过 `parentId` 形成树形结构
- `leafId` 指向当前活跃分支的叶节点

---

## 5. 关键源码位置

| 文件 | 作用 |
|------|------|
| `packages/coding-agent/src/config.ts` | getSessionsDir() 定义 |
| `packages/coding-agent/src/core/session-manager.ts` | JSONL 读写、树结构操作 |
| `packages/coding-agent/src/core/agent-session.ts` | Session 管理、事件发射 |
| `packages/coding-agent/src/modes/interactive/session-selector.ts` | 会话选择 UI |

---

## 6. 研究问题解答

### Q1: 对话数据存储在哪里？

`~/.pi/agent/sessions/{cwd}/` 目录下的 JSONL 文件。

### Q2: 存储格式是什么？

JSONL（JSON Lines），每行一个 JSON 对象。

### Q3: 会话如何按项目隔离？

按工作目录（cwd）组织子目录，不同项目有不同的会话文件夹。

### Q4: 支持多少历史对话？

通过压缩机制支持无限长的会话，压缩后之前的消息仍保留在文件中，但通过摘要方式传递给 LLM。

---

## 7. 相关配置

### 7.1 压缩配置

```json
{
    "compaction": {
        "enabled": true,
        "reserveTokens": 16384,
        "keepRecentTokens": 20000
    }
}
```

### 7.2 会话相关命令

| 命令 | 说明 |
|------|------|
| `/resume` | 浏览选择之前的会话 |
| `/new` | 开始新会话 |
| `/fork` | 从之前的用户消息创建新分支 |
| `/clone` | 复制当前分支到新会话 |
| `/compact` | 手动触发压缩 |

---

*本文档为研究产出，后续可根据需要进一步补充和修正。*