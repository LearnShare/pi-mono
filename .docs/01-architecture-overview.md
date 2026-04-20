# pi-monorepo 架构概述

## 项目信息

| 属性 | 值 |
|------|-----|
| 项目名称 | pi-monorepo (.pi coding agent) |
| GitHub | badlogic/pi-mono |
| 作者 | Mario Zechner (@badlogic) |
| 版本 | 0.67.68 |
| 许可 | MIT |
| Node.js | >= 20.0.0 |

## 项目背景

.pi 是一个开源的 AI Coding Agent，功能类似 Claude Code，提供：
- 交互式 CLI 界面
- 工具调用能力
- 会话管理
- 可扩展架构
- 多 LLM 提供商支持

## 包结构

```
pi-monorepo/
├── packages/
│   ├── ai/              # 统一 LLM API 层 - 20+ 提供商抽象
│   ├── agent/           # Agent 运行时核心 - LLM 调用和工具执行
│   ├── coding-agent   # CLI 应用 - 交互式界面、会话管理
│   ├── tui/           # 终端 UI 库 - 差分渲染
│   ├── mom/            # Slack integration
│   ├── web-ui/         # Web components
│   └── pods/           # GPU pod management
├── scripts/           # Build/release scripts
├── .pi/              # Agent config
└── .husky/           # Git hooks
```

### 各包版本 (Lockstep)

| 包 | 版本 |
|---|------|
| @mariozechner/pi-tui | 0.67.68 |
| @mariozechner/pi-ai | 0.67.68 |
| @mariozechner/pi-agent-core | 0.67.68 |
| @mariozechner/pi-coding-agent | 0.67.68 |
| @mariozechner/pi-mom | 0.67.68 |
| @mariozechner/pi-web-ui | 0.67.68 |
| @mariozechner/pi-pods | 0.67.68 |

## 依赖关系图

```
┌─────────────────────────────────────────────────────┐
│              pi-coding-agent                     │
│  (CLI 应用)                                       │
├─────────────────────────────────────────────────────┤
│  dependencies:                                   │
│    @mariozechner/pi-agent-core                    │
│    @mariozechner/pi-ai                          │
│    @mariozechner/pi-tui                         │
│    @silvia-odwyer/photon-node                   │
│    chalk, diff, glob, yaml, ...                 │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│              pi-agent-core                       │
│  (Agent 运行时)                                   │
├─────────────────────────────────────────────────────┤
│  dependencies:                                   │
│    @mariozechner/pi-ai                          │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                  pi-ai                      │
│  (统一 LLM API)                                  │
├────────────────────────────────────────────────���────┤
│  dependencies:                                 │
│    anthropic-sdk, openai, google-genai          │
│    @mistralai/mistralai                          │
│    @aws-sdk/client-bedrock-runtime             │
│    @sinclair/typebox                           │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                  pi-tui                      │
│  (终端 UI 库)                                    │
├─────────────────────────────────────────────────────┤
│  dependencies: (无外部依赖 - 独立库)                │
└─────────────────────────────────────────────────────┘
```

## 核心数据流

```
┌─────────────┐
│ 用户输入    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│           pi-coding-agent                      │
│  ┌─────────────────────────────────────────────┐  │
│  │ main.ts - CLI 入口                         │  │
│  │ - 参数解析                              │  │
│  │ - 模式选择 (interactive/print/rpc/json) │  │
│  └──────┬──────────────────────────────────┘  │
│         ▼                                     │
│  ┌─────────────────────────────────────────────┐ │
│  │ AgentSession 运行时                      │ │
│  │ - 会话管理                                │ │
│  │ - 事件处理                                │ │
│  │ - 工具注册                               │ │
│  └──────┬──────────────────────────────────┘  │
└─────────┼─────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────┐
│           pi-agent-core                     │
│  ┌─────────────────────────────────────────────┐ │
│  │ Agent 类                                 │ │
│  │ - 状态管理                               │ │
│  │ - 消息队列 (steering/followUp)           │ │
│  └──────┬──────────────────────────────────┘ │
│         ▼                                      │
│  ┌─────────────────────────────────────────────┐ │
│  │ runAgentLoop()                            │ │
│  │ - 流式 LLM 调用                         │ │
│  │ - 工具执行                              │ │
│  └──────┬───────���─��────────────────────────┘ │
└─────────┼─────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────┐
│                pi-ai                          │
│  ┌─────────────────────────────────────────────┐   │
│  │ streamSimple(model, context, options)     │   │
│  │ - 提供商解析                              │   │
│  │ - 参数转换                              │   │
│  │ - 流式事件                             │   │
│  └──────┬──────────────────────────────────┘    │
│         ▼                                       │
│  ┌─────────────────────────────────────────────┐  │
│  │ API Provider 实现                        │  │
│  │ - anthropic.ts                           │  │
│  │ - openai-responses.ts                   │  │
│  │ - google.ts                             │  │
│  │ - ...                                   │  │
│  └──────┬──────────────────────────────────┘  │
└─────────┼─────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────┐
│            LLM Providers                         │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐   │
│  │OpenAI  │ │Anthropic│ │Google  │ │Mistral │   │
│  └────────┘ └────────┘ └────────┘ └────────┘   │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐   │
│  │ Bedrock│ │ Azure  │ │ xAI   │ │ Groq   │   │
│  └────────┘ └────────┘ └────────┘ └────────┘   │
└─────────────────────────────────────────────────────┘
```

## 工作模式

### 1. Interactive Mode (交互模式)

```
终端 → TUI → AgentSession → Agent → LLM → 工具执行 → UI 更新
```

- 完整的 TUI 界面
- 实时流式输出
- 键盘导航
- 消息历史滚动
- 会话选择器
- 模型切换 (Ctrl+P)

### 2. Print Mode (打印模式)

```
stdin/args → AgentSession → Agent → LLM → 工具执行 → 打印输出
```

- 非交互式批量处理
- 支持 stdin 输入
- 适合脚本集成

### 3. RPC Mode (RPC 模式)

```
JSON-RPC 请求 → AgentSession → Agent → LLM → JSON-RPC 响应
```

- JSON-RPC 2.0 接口
- 远程 procedure call
- 适合 IDE 集成

### 4. JSON Mode (JSON 模式)

```
输入 → AgentSession → Agent → LLM → JSON 输出
```

- 纯 JSON 输出
- 机器可读

## 核心文件结构

### pi-ai

```
packages/ai/
├── src/
│   ├── index.ts                 # 入口，导出所有
│   ├── types.ts                # 核心类型 (Api, Provider, Model, Message...)
│   ├── stream.ts               # 流式 API (stream, streamSimple, complete, completeSimple)
│   ├── api-registry.ts         # API 提供商注册表
│   ├── env-api-keys.ts        # 环境变量 API key 解析
│   ├── models.ts              # 模型系统
│   ├── oauth.ts              # OAuth 支持
│   ├── providers/
│   │   ├── register-builtins.ts    # 内置提供商延迟加载注册
│   │   ├── anthropic.ts           # Anthropic API
│   │   ├── openai-responses.ts    # OpenAI Responses API
│   │   ├── openai-completions.ts   # OpenAI Completions API
│   │   ├── google.ts              # Google Generative AI
│   │   ├── google-gemini-cli.ts   # Google Gemini CLI
│   │   ├── google-vertex.ts     # Google Vertex
│   │   ├── mistral.ts          # Mistral
│   │   ├── amazon-bedrock.ts  # Amazon Bedrock
│   │   ├── azure-openai-responses.ts
│   │   ├── openai-codex-responses.ts
│   │   ├── transform-messages.ts   # 消息转换
│   │   └── faux.ts               # Mock provider
│   ├── utils/
│   │   ├── event-stream.ts     # 事件流工具
│   │   ├── json-parse.ts      # JSON 解析
│   │   ├── sanitize-unicode.ts
│   │   └── overflow.ts
│   └── cli.ts                 # CLI 入口
```

### pi-agent-core

```
packages/agent/
├── src/
│   ├── index.ts               # 入口
│   ├── agent.ts              # Agent 类
│   ├── agent-loop.ts         # Agent 循环
│   ├── types.ts             # 类型定义
│   └── proxy.ts            # 代理工具
```

### pi-coding-agent

```
packages/coding-agent/
├── src/
│   ├── main.ts              # CLI 入口
│   ├── cli/
│   │   ├── args.ts         # 参数解析
│   │   └── ...
│   ├── core/
│   │   ├── agent-session.ts        # 会话管理
│   │   ├── agent-session-runtime.ts
│   │   ├── session-manager.ts    # 会话管理器
│   │   ├── tools/               # 工具系统
│   │   │   ├── index.ts
│   │   │   ├── read.ts
│   │   │   ├── bash.ts
│   │   │   ├── edit.ts
│   │   │   ├── write.ts
│   │   │   ├── grep.ts
│   │   │   ├── find.ts
│   │   │   └── ls.ts
│   │   ├── extensions/           # 扩展系统
│   │   │   ├── types.ts
│   │   │   ├── loader.ts
│   │   │   ├── runner.ts
│   │   │   └── wrapper.ts
│   │   ├── skills.ts           # Skills 系统
│   │   ├── model-registry.ts   # 模型注册
│   │   ├── settings-manager.ts
│   │   ├── auth-storage.ts
│   │   ├── compaction/        # 会话压缩
│   │   └── system-prompt.ts
│   ├── modes/
│   │   ├── interactive/       # 交互模式
│   │   │   ├── interactive-mode.ts
│   │   │   └── components/
│   │   ├── print/            # 打印模式
│   │   └── rpc/             # RPC 模式
│   └── config.ts            # 配置
```

### pi-tui

```
packages/tui/
├── src/
│   ├── index.ts            # 入口
│   ├── tui.ts            # 主 TUI 类
│   ├── terminal.ts       # 终端接口
│   ├── component.ts     # 组件基类
│   ├── keys.ts          # 按键处理
│   ├── components/      # 预置组件
│   │   ├── text.ts
│   │   ├── box.ts
│   │   ├── input.ts
│   │   ├── select-list.ts
│   │   ├── editor.ts
│   │   └── ...
│   └── utils.ts         # 工具函数
```

## 开发命令

```bash
# 安装依赖
npm install

# 构建所有包
npm run build

# 监听模式开发
npm run dev

# 类型检查和 Lint
npm run check

# 运行测试
npm run test

# 发布
npm run release:patch  # 补丁版本
npm run release:minor  # 次版本
```

## 总结

.pi 是一个设计精良的 AI Coding Agent Monorepo：

1. **分层架构** - 从 UI 到 LLM API 的清晰分层
2. **统一接口** - 20+ 提供商统一抽象
3. **可扩展** - Extensions + Skills 双扩展系统
4. **多模式** - Interactive/Print/RPC/JSON 四种模式
5. **高质量** - 完善的错误处理、会话管理、压缩

架构核心是 `pi-ai` 提供统一 LLM 接口，`pi-agent-core` 封装 Agent 逻辑，`pi-coding-agent` 实现 CLI 应用，`pi-tui` 提供终端渲染。