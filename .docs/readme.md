# Pi Agent 文档目录

> 本目录包含 pi-coding-agent 的完整文档体系
> 最后更新：2026-04-29

---

## 文档结构总览

```
.docs/
├── readme.md                           # 本文档
├── user-guide/                         # 用户指南（面向终端用户）
├── configuration/                      # 配置指南（面向配置用户）
├── extension-dev/                      # 扩展开发（面向开发者）
├── resources/                          # 资源索引（面向寻找资源用户）
├── internals/                          # 内部机制（面向高级用户/贡献者）
├── research/                           # 研究笔记（研究者）
└── research-plan/                      # 研究计划（规划者）
```

---

## 用户指南（user-guide/）

面向终端用户的基础使用文档

### 01-installation.md - 安装与认证
- 全局安装：`npm install -g @mariozechner/pi-coding-agent`
- 验证安装：`pi --version`
- 更新命令：`pi update`
- 订阅登录（OAuth）：Claude、ChatGPT、GitHub Copilot、Google Gemini
- API Key 认证：环境变量或 auth.json
- 云提供商：Azure OpenAI、Amazon Bedrock、Cloudflare Workers AI、Google Vertex AI

### 02-basic-usage.md - 基础使用
- 交互模式启动与界面区域
- 编辑器功能：文件引用(@)、路径补全、多行输入、图片粘贴、Shell命令(!)
- 非交互模式：管道用法
- 消息队列：steer/followUp机制
- 快捷键汇总：导航、编辑器、会话、斜杠命令
- 思考级别：off/minimal/low/medium/high/xhigh

### 03-cli-commands.md - CLI命令参考
- 基本用法：`pi [options] [@files...] [messages...]`
- 模式参数：--print、--mode json/rpc
- 会话参数：--continue、--resume、--session、--fork
- 模型参数：--provider、--model、--thinking
- 工具参数：--tools、--no-tools
- 扩展与技能：--extension、--skill
- 主题与上下文：--theme、--no-context-files

---

## 配置指南（configuration/）

面向配置用户的进阶文档

### 01-settings.md - 配置详解
- 配置文件位置：~/.pi/agent/settings.json（全局） / .pi/settings.json（项目级）
- 核心设置项：
  - 模型与思考：defaultProvider、defaultModel、defaultThinkingLevel
  - UI与显示：theme、quietStartup、doubleEscapeAction
  - 压缩：compaction.enabled、reserveTokens、keepRecentTokens
  - 重试：retry.enabled、maxRetries、baseDelayMs
  - 终端与图片：terminal.showImages、imageWidthCells
- 自定义模型：models.json配置Ollama、vLLM、LM Studio

### 02-customization.md - 6种自定义机制
- **Extensions**：TypeScript模块，注册工具、监听事件、自定义UI
- **Skills**：按需加载的能力包，含SKILL.md指令
- **Prompt Templates**：可复用Markdown提示模板，/name扩展
- **Themes**：JSON主题文件，定义TUI的51个颜色token
- **Pi Packages**：打包扩展、技能、模板、主题，npm/git共享
- **Context Files**：AGENTS.md/CLAUDE.md项目指令
- 快捷键自定义：keybindings.json

### 03-providers.md - 提供商设置
- 认证方式：/login OAuth 或 API Key
- 支持的提供商列表与环境变量
- 云提供商配置
- 自定义模型配置

---

## 扩展开发（extension-dev/）

面向扩展开发者的文档

### 01-getting-started.md - 开发入门
- 扩展入口：默认工厂函数，接收ExtensionAPI
- 多文件扩展：目录结构
- 带依赖的包：package.json配置
- 常用导入：@mariozechner/pi-coding-agent、typebox、@mariozechner/pi-tui
- 常见用例：注册工具、监听事件、注册命令、用户交互

### 02-api-reference.md - API完整参考（1124行）
- ExtensionAPI核心方法：on、registerTool、registerCommand、registerProvider等
- ExtensionContext：ui方法、sessionManager、modelRegistry、signal
- 完整事件参考：session_start、before_agent_start、tool_call、tool_result等
- 状态管理：工具details持久化、会话历史恢复
- 自定义流式API：streamSimple实现
- OAuth支持

---

## 资源索引（resources/）

寻找资源的用户的索引

### 01-official-examples.md - 官方示例
- 生命周期与安全：permission-gate、protected-paths、sandbox
- 自定义工具：todo、question、tool-override、subagent
- 命令与UI：plan-mode、tools、snake、doom-overlay
- Git集成：git-checkpoint、auto-commit-on-exit
- 自定义提供商：custom-provider-anthropic、custom-provider-gitlab-duo

### 02-community-packages.md - 社区热门包
- 安装与管理：pi install/remove/list/update
- 热门包分类：
  - 集成工具：pi-mcp-adapter、pi-lens、context-mode
  - 子代理编排：pi-subagents、taskplane、pi-crew
  - 搜索知识：greedysearch-pi、pi-oracle
  - UI增强：pi-powerline-footer、pi-studio
  - 工作流：pi-gsd、forgeflow-dev
- 资源索引链接

---

## 内部机制（internals/）

面向高级用户/贡献者的内部机制文档

### 00-architecture.md - 架构概述
- 项目信息：版本0.67.68，MIT许可
- 包结构：ai、agent、coding-agent、tui、mom、web-ui、pods
- 依赖关系图：coding-agent → agent → ai → tui
- 核心模块功能说明

### 01-builtin-prompts.md - 内置提示词详解（599行）
- 主系统提示结构
- 工具描述与准则（read、bash、edit、write等）
- 提示模板与上下文
- 思考块处理

---

## 研究笔记（research/）

研究过程中的笔记和产出

| 文件 | 内容 |
|------|------|
| 00-research-strategy.md | 研究策略 |
| 01-architecture-overview.md | 架构概述 |
| 02-pi-ai-typesystem.md | AI类型系统 |
| 03-agent-core.md | Agent核心 |
| 04-coding-agent-cli.md | CLI研究 |
| 05-tui-system.md | TUI系统 |
| 06-extensions-skills.md | 扩展与Skill |
| 07-technical-summary.md | 技术总结 |
| 08-session-lifecycle.md | Session生命周期（研究产出） |
| 09-provider-communication.md | Provider通信机制（研究产出） |
| 10-tool-execution.md | Tool执行流程（研究产出） |
| 11-prompt-skill-system.md | Prompt/Skill系统（研究产出） |

### 研究产出详情

**08-session-lifecycle.md**：
- Session文件格式（JSONL）
- 条目类型：message、compaction、branch_summary、custom
- 创建/继续/分支流程
- 消息流与Turn边界
- 压缩机制与树导航

**09-provider-communication.md**：
- 20+ Provider支持
- 消息转换流程
- 流式事件架构
- Provider特定处理（Anthropic/OpenAI/Google）
- 工具定义序列化
- Usage追踪

**10-tool-execution.md**：
- 6个内置工具（read/write/edit/bash/grep/find/ls）
- ToolDefinition结构
- 静态发现与动态注册
- Agent Loop执行流程
- 并行/串行执行模式
- 参数验证与结果处理

**11-prompt-skill-system.md**：
- Prompt分层结构（系统提示→项目上下文→技能→日期）
- buildSystemPrompt()流程
- Skill发现位置与验证
- 内联展开与系统提示追加
- Prompt Template参数替换

**12-session-storage.md**（新增）：
- 存储位置：~/.pi/agent/sessions/{cwd}/
- 文件格式：JSONL（JSON Lines）
- 数据结构：Session Header + Entry 类型
- 会话树组织

**13-memory-system.md**（新增）：
- 记忆层次：上下文/压缩/分支摘要/项目上下文
- buildSessionContext() 机制
- 压缩触发条件与流程
- 分支摘要生成
- AGENTS.md/CLAUDE.md 上下文文件
- 扩展状态持久化

---

## 研究计划（research-plan/）

研究计划文档

| 文件 | 说明 |
|------|------|
| 00-detailed-research-plan.md | 历史研究计划（已完成） |
| 01-research-checklist.md | 研究检查清单 |
| 02-detailed-research-plan.md | 当前研究计划（进行中） |
| 03-future-research-directions.md | 未来研究方向与优化建议（规划中） |

---

## 相关链接

- [Pi Agent 官方文档](https://pi.dev/docs/latest)
- [Pi Packages Gallery](https://pi.dev/packages)
- [GitHub 仓库](https://github.com/badlogic/pi-mono)
- [Discord 社区](https://discord.com/invite/3cU7Bz4UPx)