# 架构概述

> 基于源码分析整理

---

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

---

## 包结构

```
pi-monorepo/
├── packages/
│   ├── ai/                    # 统一 LLM API 层 - 20+ 提供商抽象
│   ├── agent/                 # Agent 运行时核心 - LLM 调用和工具执行
│   ├── coding-agent           # CLI 应用 - 交互式界面、会话管理
│   ├── tui/                   # 终端 UI 库 - 差分渲染
│   ├── mom/                   # Slack integration
│   ├── web-ui/                # Web components
│   └── pods/                  # GPU pod management
├── scripts/                   # Build/release scripts
├── .pi/                       # Agent config
└── .husky/                    # Git hooks
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

---

## 依赖关系图

```
┌─────────────────────────────────────────────────────┐
│ pi-coding-agent (CLI 应用)                          │
├─────────────────────────────────────────────────────┤
│ dependencies:                                        │
│ @mariozechner/pi-agent-core                         │
│ @mariozechner/pi-ai                                 │
│ @mariozechner/pi-tui                                │
│ @silvia-odwyer/photon-node                         │
│ chalk, diff, glob, yaml, ...                        │
└─────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────┐
│ pi-agent-core (Agent 运行时)                        │
├─────────────────────────────────────────────────────┤
│ dependencies:                                        │
│ @mariozechner/pi-ai                                 │
└─────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────┐
│ pi-ai (统一 LLM API)                                │
├─────────────────────────────────────────────────────┤
│ dependencies:                                        │
│ anthropic-sdk, openai, google-genai                 │
│ @mistralai/mistralai                                │
│ @aws-sdk/client-bedrock-runtime                    │
│ @sinclair/typebox                                   │
└─────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────┐
│ pi-tui (终端 UI 库)                                 │
├─────────────────────────────────────────────────────┤
│ dependencies: (无外部依赖 - 独立库)                 │
└─────────────────────────────────────────────────────┘
```

---

## 核心模块

### packages/ai

统一 LLM API 层，支持 20+ 提供商：
- Anthropic、OpenAI、Google Gemini
- DeepSeek、Mistral、Groq、xAI
- OpenRouter、Hugging Face
- Azure OpenAI、Amazon Bedrock
- Cloudflare Workers AI、Google Vertex AI
- 等等

### packages/agent

Agent 运行时核心：
- LLM 调用和响应处理
- 工具执行框架
- 消息流管理

### packages/coding-agent

CLI 应用：
- 交互式 TUI 界面
- 会话管理（JSONL 文件）
- 参数解析
- 扩展和技能加载

### packages/tui

终端 UI 库：
- 差分渲染引擎
- 布局系统
- 组件库