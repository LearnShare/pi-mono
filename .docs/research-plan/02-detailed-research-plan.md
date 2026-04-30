# 详细研究计划：Pi Session/Tools/Provider 生命周期与通信机制

> 研究目标：深入理解 pi-coding-agent 和 pi-ai 的核心运行时机制
> 最后更新：2026-04-29

---

## 研究背景与动机

Pi Agent 的核心价值在于其「紧凑核心 + 可扩展架构」的设计理念。要真正掌握如何自定义、调试和扩展 pi，需要深入理解其内部运行时机制：

1. **Session/Tool 生命周期** — 理解消息如何流转、工具何时调用、会话如何存续
2. **Provider 通信机制** — 理解 pi 如何与 20+ LLM 提供商交互、流式响应的处理
3. **Tool 执行流程** — 理解工具的发现、执行、结果回传和 LLM 上下文更新的完整链路

这三者相互关联：User Message → Session Manager → Agent Loop → Provider → Tool Execution → Session Update → (循环)

---

## 主题一：Session 与 Chat 消息生命周期

### 1.1 研究目标

- 理解 Session 的创建、存续、压缩、fork 和树结构导航的完整生命周期
- 理解 Chat 消息流（User → Assistant → Tool Call → Tool Result → ...）如何闭环
- **注意：Tool 相关的详细研究放到主题三**

### 1.2 源码位置

| 文件 | 作用 |
|------|------|
| `packages/coding-agent/src/core/agent-session.ts` | Session 管理、消息队列、事件发射 |
| `packages/coding-agent/src/core/session-manager.ts` | JSONL 读写、树结构操作 |
| `packages/coding-agent/src/core/compaction/*.ts` | 压缩/分支摘要生命周 |
| `packages/coding-agent/src/core/messages.ts` | 自定义消息类型转换 |
| `packages/coding-agent/src/core/resource-loader.ts` | 资源（工具/技能/模板）发现 |

### 1.3 详细研究问题

#### Session 生命周期

1. **Session 创建**
   - `SessionManager.create()` 如何初始化 JSONL 文件头？
   - 头部的 `metadata` 字段包含什么？（模式、provider、model 等）
   - `AgentSession` 如何关联到具体的 Session 文件？

2. **Session 继续**
   - `SessionManager.continueRecent()` 如何找到最近的会话？
   - 会话恢复后，消息如何反序列化到 `AgentMessage[]`？
   - 压缩摘要（CompactionEntry）如何被重新解释为 LLM 上下文？

3. **Session 分支（Fork/Clone/Tree）**
   - Fork 时，SessionManager 如何复制现有消息到新文件？
   - Clone 与 Fork 的区别是什么？（Clone 复制活跃分支；Fork 从旧消息点创建）
   - `/tree` 导航时，分支如何「丢弃未合并到摘要」vs「保留在文件中」？

4. **Session 压缩（Compaction）**
   - 触发阈值如何计算？（`contextTokens > contextWindow - reserveTokens`）
   - 切点（cut point）如何确定？（turn 边界、keepRecentTokens）
   - 压缩生成摘要的 LLM 调用是同步还是异步？结果如何合并回会话？

5. **Session 退出与恢复**
   - `session_shutdown` 事件何时触发？扩展如何清理资源？
   - 下次启动时，Session 如何关联回同一文件？

#### Chat 消息流

1. **消息类型与角色**
   - pi 定义了哪些自定义消息类型？（`bashExecution`, `custom`, `branchSummary`, `compactionSummary`）
   - 每种消息类型在 LLM 上下文中如何表示？
   - 压缩后，之前的 `assistant`、`toolResult` 消息是否保留？

2. **Turn 边界**
   - 什么定义了一个「Turn」？（User Message → 零或多个 Tool Calls → Assistant Stop）
   - Turn 之间的事件顺序是什么？（`turn_start` → `tool_call` → `tool_result` → `turn_end`）

3. **消息队列（Steer/FollowUp）**
   - `steer()` 和 `followUp()` 排队的消息何时投递？
   - 投递时，是插入到当前消息序列的哪个位置？

#### Tool 发现时机（简述）

1. **工具注册点**
   - 工具在 `session_start` 时被加载，还是在每次 LLM 调用前发现？
   - 扩展注册的工具在哪个生命周期点可用？

#### 主题三的 Tool 研究将涵盖

- Tool 调用链路：`tool_execution_start` → `tool_execution_update` → `tool_execution_end` → `tool_result`
- Tool 结果处理：`tool_result` 消息构建、`details` 字段、上下文注入时机
- 执行模式：并行 vs 串行

### 1.4 输出产物

- **流程图**：Session 创建 → 消息流转 → 压缩/分支 → 退出的完整生命周期
- **状态机**：不同会话状态（创建/继续/fork）之间的转换
- **数据结构**：Session 文件格式、消息类型定义

---

## 主题二：Pi 与 Provider 之间的通信机制

### 2.1 研究目标

- 理解 pi 如何将消息序列转换为 Provider 兼容的请求格式
- 理解流式响应（Server-Sent Events、Server-Side Streaming）如何被消费和处理
- 理解 Provider 特定功能（Thinking Block、Tool Call、Usage Tracking）的处理差异

### 2.2 源码位置

| 文件 | 作用 |
|------|------|
| `packages/ai/src/types.ts` | 通用类型定义：Message、Content、Usage |
| `packages/ai/src/providers/*.ts` | 20+ Provider 的流式实现 |
| `packages/ai/src/context.ts` | 请求上下文构建 |
| `packages/ai/src/utils/event-stream.ts` | 流式事件发射 |

### 2.3 详细研究问题

#### 请求构建

1. **`AgentMessage[]` → Provider Payload**
   - `convertToLlm(messages)` 如何转换 pi 消息到 Provider 消息？
   - 角色映射（User/Assistant/ToolResult → Provider 对应格式）如何处理？
   - 系统提示在请求中的位置？

2. **内容序列化**
   - `TextContent`、`ImageContent`、`ThinkingContent` 如何序列化给不同 Provider？
   - Anthropic vs OpenAI vs Google 的内容格式差异如何归一化？

3. **工具定义序列化**
   - pi 的 `ToolDefinition` 如何转换为 Provider 的工具 schema？（JSON Schema / Tool Definitions）
   - Provider 对工具参数格式的要求如何兼容？（`anyOf` vs `oneOf` vs `enum`）

#### 流式响应处理

1. **事件流架构**
   - `AssistantMessageEventStream` 的事件类型：`start` → `text_delta` / `thinking_delta` / `toolcall_delta` → `done` / `error`
   - 每个事件的 `partial` 字段包含什么？增量还是完整状态？

2. **Provider 特定处理**
   - **Anthropic**：`contentBlock_delta`、`message_delta`、`message_stop`、`content_block_stop` 事件
   - **OpenAI**：`content` delta、`function_call` delta、`tool_calls` delta
   - **Google**：`candidates` delta、`content`、`groundingInlineData`
   - 各 Provider 的 stop reason（`stop`、`length`、`tool_use`、`max_tokens`）如何映射？

3. **Usage 追踪**
   - Provider 返回的 usage（input_tokens、output_tokens）如何累加？
   - 缓存 token（cache_read、cache_write）如何报告？

#### Provider 特定功能

1. **Thinking / Extended Thinking**
   - Anthropic 的 `thinking` content block 如何处理？
   - OpenAI o1 模型的 reasoning（草莓问题）如何映射到 `ThinkingContent`？
   - Thinking Level 控制（Thinking Budget）如何实现？

2. **Tool Calling**
   - Provider 原生工具调用（Anthropic、OpenAI）vs 手动函数调用（某些兼容 API）的区别
   - 流式工具调用（`eager_input_streaming`）vs 块工具调用的差异

3. **多 Provider 注册**
   - `pi.registerProvider()` 的机制：一个 Provider 可以有多个模型
   - 模型选择时的 Provider 查找逻辑

### 2.4 输出产物

- **API 调用链路图**：请求构建 → HTTP 请求 → 流式响应 → 事件处理 → 状态更新
- **Provider 比较表**：各 Provider 的请求/响应格式差异、处理逻辑
- **协议映射**：pi 统一事件 ↔ Provider 特定事件的对应表

---

## 主题三：Tool 发现、执行、结果收集与反馈流程

### 3.1 研究目标

- 理解工具的静态发现（加载时）vs 动态注册（运行时）的机制
- 理解工具的执行模式（顺序/并行/预览）及其实现
- 理解工具执行结果的序列化、存储和上下文注入

### 3.2 源码位置

| 文件 | 作用 |
|------|------|
| `packages/coding-agent/src/core/tools/*.ts` | 6 个内置工具实现 |
| `packages/coding-agent/src/core/tool-definitions.ts` | 工具 schema 和定义 |
| `packages/coding-agent/src/core/extensions/types.ts` | 扩展工具注册相关事件 |
| `packages/agent/src/agent-loop.ts` | 工具调用在 Agent Loop 中的位置 |

### 3.3 详细研究问题

#### Tool 发现

1. **静态发现**
   - `ResourceLoader` 如何扫描 `extensions/*.ts` 文件？
   - 发现时，工具的 `parameters` schema（TypeBox）何时被解析？
   - 工具的 `description` 和 `promptSnippet` 在何处被使用？

2. **动态注册**
   - 扩展如何通过 `pi.registerTool()` 注册工具？
   - `session_start` 事件中注册的工具 vs 运行时注册的工具何时生效？
   - 同一工具名被多次注册时，如何处理冲突？

3. **工具可用性**
   - 通过 `--tools` CLI 参数如何过滤可用工具？
   - 工具如何随 Session 持久化？（`toolDefinitions` 存储在会话中？）

#### Tool 执行

1. **执行入口**
   - `AgentLoop` 如何将 LLM 返回的 `toolCall` 转换为实际的函数调用？
   - `ToolExecutor` 接口的定义和实现是什么？

2. **执行模式**
   - **Parallel Mode**（默认）：同一助手消息中的多个工具调用如何并行执行？
   - **Sequential Mode**：何时使用？（如游戏状态共享）
   - **Preview Mode**：工具执行前的预览如何工作？（参数验证/preview UI）

3. **参数处理**
   - TypeBox schema 如何验证工具输入？
   - 不合法的参数如何返回错误？

4. **执行上下文**
   - 工具执行时的 `cwd` 从何而来？
   - `signal`（AbortSignal）如何传递？取消时会发生什么？
   - `onUpdate` 回调用于什么？（流式输出）

#### 结果收集与反馈

1. **结果构建**
   - `ExecuteResult` 接口：`content`、`details`、`isError` 如何设置？
   - `details` 字段的类型安全如何保证？（TypeBox / 自定义类型）

2. **结果序列化**
   - 文本结果如何序列化为 `TextContent`？
   - 图片结果如何处理？（`ImageContent`）
   - 错误结果如何标记？（`isError: true`）

3. **上下文注入**
   - `toolResult` 消息何时被追加到 LLM 上下文？
   - 工具结果中的 `details` 是否会被发送给 LLM？
   - 多轮工具调用中，工具结果是否累积？

4. **会话持久化**
   - 执行后的 `ToolResultMessage` 何时写入 Session 文件？
   - 压缩时，工具结果如何被摘要？

### 3.4 输出产物

- **Tool 生命周期图**：发现 → 注册 → 执行 → 结果 → 持久化
- **执行模式流程**：Parallel / Sequential / Preview 的控制流
- **数据结构**：ToolDefinition、ExecuteResult、ToolResultMessage 的完整定义

---

## 主题四：Prompt 与 Skill 的分层、组装、发现与使用

### 4.1 研究目标

- 理解 Prompt 的分层结构（系统提示 → 项目上下文 → 技能 → 动态指令）如何组装
- 理解 Prompt 发送给 Provider 前的最终格式化
- 理解 Skill 的发现机制、加载时机和使用方式
- 理解 Prompt Template 的展开和参数替换

### 4.2 源码位置

| 文件 | 作用 |
|------|------|
| `packages/coding-agent/src/core/system-prompt.ts` | 系统提示构建（buildSystemPrompt） |
| `packages/coding-agent/src/core/prompt-templates.ts` | Prompt Template 加载与展开 |
| `packages/coding-agent/src/core/skills.ts` | Skill 加载与格式化（formatSkillsForPrompt） |
| `packages/coding-agent/src/core/resource-loader.ts` | 资源（提示/技能/主题）发现 |
| `packages/coding-agent/src/core/agent-session.ts` | Prompt 注入到消息流 |

### 4.3 详细研究问题

#### Prompt 分层与组装

1. **Prompt 层叠顺序**
   - 系统提示（`buildSystemPrompt()` 构建）的默认结构是什么？
   - 项目上下文文件（AGENTS.md / CLAUDE.md）追加在哪个位置？
   - 技能列表追加在哪个位置？（条件是什么）
   - 自定义 `--append-system-prompt` / `APPEND_SYSTEM.md` 追加位置？
   - 日期和工作目录追加位置？
   - 最终的 Prompt 顺序：系统提示 → 自定义 → 项目上下文 → 技能 → 日期/cwd

2. **自定义 Prompt 处理**
   - `--system-prompt` CLI 参数如何替换默认系统提示？
   - `.pi/SYSTEM.md` 与全局 `~/.pi/agent/SYSTEM.md` 的优先级？
   - `APPEND_SYSTEM.md` 的追加逻辑？
   - 扩展的 `before_agent_start` 事件如何修改/替换系统提示？

3. **Prompt 内容注入**
   - 系统提示是作为独立的 `system` 消息？还是嵌入到每个请求的 `system` 字段？
   - Provider 如何处理 system prompt？（Anthropic / OpenAI / Google 的差异）

#### Prompt 发送

1. **请求格式化**
   - `convertToLlm(messages)` 如何将 messages 转换为 Provider 格式？
   - system prompt 在转换中如何处理？
   - 上下文窗口超限时的处理？（截断 vs 压缩）

2. **Provider 差异**
   - Anthropic：system prompt 作为独立字段
   - OpenAI：system prompt 作为消息数组中的 role="system" 消息
   - Google：system prompt 作为 `system_instruction` 字段
   - 各 Provider 对 system prompt 的 token 限制？

#### Skill 发现

1. **发现位置**
   - 全局：`~/.pi/agent/skills/` 和 `~/.agents/skills/`
   - 项目：`.pi/skills/` 和 `.agents/skills/`（从 cwd 向上遍历到 git 根目录）
   - 包：`skills/` 目录或 `package.json` 中的 `pi.skills` 条目
   - CLI：`--skill <path>` 参数

2. **发现时机**
   - Skill 在 `session_start` 时发现，还是延迟发现（按需）？
   - `ResourceLoader.loadSkills()` 的调用路径？

3. **解析与验证**
   - 如何解析 SKILL.md 的 frontmatter？（name、description、license 等）
   - 验证规则（Agent Skills 标准）如何应用？
   - 验证失败时如何处理？（警告 vs 拒绝加载）

#### Skill 使用

1. **内联展开**
   - `/skill:name` 命令如何展开 Skill 内容？
   - 展开后的 XML 格式：`<skill name="" location="">...` 的处理
   - 参数如何传递？（`/skill:name arg1 arg2`）

2. **系统提示引用**
   - 何时将 Skill 追加到系统提示？（`formatSkillsForPrompt()` 的条件）
   - 追加条件：工具集中必须有 `read` 工具可用
   - `disableModelInvocation: true` 的 Skill 不追加到系统提示，只能手动调用

3. **Skill 引用相对路径**
   - Skill 文件中引用的相对路径如何解析？（相对于 SKILL.md 所在目录）

#### Prompt Template

1. **发现与加载**
   - Prompt Template 在 `prompts/*.md` 中如何发现？
   - 如何发现嵌套目录中的模板？
   - Frontmatter 解析（description、argument-hint）？

2. **展开与参数替换**
   - `/template` 命令如何展开？
   - 参数替换：`$1`, `$2`, `$@`, `${@:N}`, `${@:N:L}` 的替换逻辑？
   - 不传参数 vs 传参数的处理差异？

### 4.4 输出产物

- **Prompt 组装流程图**：各层 Prompt 的叠加顺序
- **Prompt 格式化表**：各 Provider 的 system prompt 位置差异
- **Skill 生命周期**：发现 → 加载 → 验证 → 追加/展开 → 使用
- **Template 展开**：参数替换规则与示例

---

## 研究方法

### 源码阅读策略

1. **从入口追溯**：找到核心 API（如 `createAgentSession()`），顺藤摸瓜找到关键函数
2. **从事件追溯**：在事件处理函数中下断点，理解事件顺序
3. **从边界追溯**：Provider 接口层（`stream*` 函数）和工具执行层（`execute()` 函数）是两个关键边界

### 调试与验证策略

1. **使用 `--verbose` 启动**：观察初始化流程
2. **使用 `/session` 查看会话状态**：检查消息和 token 计数
3. **使用 LLM 的 `/model` 调试模式**：观察请求/响应的 JSON 格式

### 文档对齐策略

- 每个研究问题对应到 `packages/ai/docs/` 或 `packages/coding-agent/docs/` 的现有文档
- 验证研究结论与官方文档的一致性

---

## 研究里程碑

### 阶段 1：Session 生命周期与消息流（预计 1.5 小时）

- [ ] Session 创建与文件格式
- [ ] 消息类型与序列化
- [ ] Turn 边界与事件流
- [ ] 压缩与分支摘要
- [ ] Tool 发现时机（简述）

### 阶段 2：Provider 通信（预计 2 小时）

- [ ] 请求构建（Message → Provider Payload）
- [ ] 流式响应处理（事件流架构）
- [ ] Provider 特定功能（Thinking、Tool Call）
- [ ] Usage 追踪

### 阶段 3：Tool 执行流程（预计 1 小时）

- [ ] 发现与注册机制
- [ ] 执行模式（Parallel/Sequential/Preview）
- [ ] 结果收集与上下文注入
- [ ] 会话持久化

### 阶段 4：Prompt 与 Skill（预计 1 小时）

- [ ] Prompt 分层与组装
- [ ] Prompt 发送格式化
- [ ] Skill 发现与验证
- [ ] Skill 使用（内联展开 / 系统提示引用）
- [ ] Prompt Template 发现与展开

### 阶段 5：综合与整合（预计 0.5 小时）

- [ ] 完整流程图绘制
- [ ] 核心数据结构整理
- [ ] 研究报告撰写

**总计预计：6.5 小时**

---

## 预期输出

1. **`.docs/research/08-session-lifecycle.md`** — Session 生命周期与消息流详解
2. **`.docs/research/09-provider-communication.md`** — Provider 通信机制详解
3. **`.docs/research/10-tool-execution-flows.md`** — Tool 发现/执行/结果收集/反馈流程详解
4. **`.docs/research/11-prompt-skill-flows.md`** — Prompt 分层/组装/发送 与 Skill 发现/使用流程详解
5. **综合流程图**（可放在 `08` 中）— 四者整合的完整运行时链路

---

## 注意事项

- 研究聚焦在 **runtime** 机制而非 UI 交互
- 避免陷入边缘 case，优先理解默认行为
- Provider 具体实现差异（如特定 env var 处理）可简化，重点在通用框架
- 结果需要包含代码引用（文件:行号）以供后续查阅