# Pi Agent: 流行扩展与资源生态

> 基于 pi.dev/packages Gallery（1852+ 包）和 GitHub 官方示例整理
> 最后更新: 2026-04-29

---

## 1. 官方示例扩展

Pi 官方仓库提供 50+ 示例扩展，按类别分类：

### 生命周期 & 安全

| 扩展 | 说明 |
|------|------|
| `permission-gate.ts` | 危险命令前提示确认（rm -rf、sudo 等） |
| `protected-paths.ts` | 阻止写入受保护路径（.env、.git/、node_modules/） |
| `confirm-destructive.ts` | 破坏性会话操作前确认 |
| `dirty-repo-guard.ts` | 有未提交 git 更改时阻止会话切换 |
| `sandbox/` | OS 级沙盒，使用 `@anthropic-ai/sandbox-runtime` |

### 自定义工具

| 扩展 | 说明 |
|------|------|
| `todo.ts` | 待办列表工具 + `/todos` 命令，含自定义渲染和状态持久化 |
| `question.ts` | 使用 `ctx.ui.select()` 向用户提问 |
| `questionnaire.ts` | 多问题输入，tab 栏导航 |
| `tool-override.ts` | 覆盖内置工具（如给 `read` 加日志/访问控制） |
| `dynamic-tools.ts` | 运行时动态注册工具 |
| `structured-output.ts` | 结构化输出工具，支持 `terminate: true` |
| `ssh.ts` | 通过 SSH 将所有工具委托给远程机器 |
| `subagent/` | 将任务委托给专用子代理 |

### 命令 & UI

| 扩展 | 说明 |
|------|------|
| `plan-mode/` | Claude Code 风格的规划模式，只读探索 |
| `tools.ts` | 交互式 `/tools` 命令管理工具开关 |
| `handoff.ts` | 将上下文转移到新专注会话 |
| `snake.ts` | 贪吃蛇游戏（自定义 UI + 键盘处理） |
| `tic-tac-toe.ts` | 井字棋 vs 代理 |
| `doom-overlay/` | 在 overlay 中运行 DOOM 游戏（35 FPS） |
| `modal-editor.ts` | Vim 风格模态编辑器 |
| `rainbow-editor.ts` | 动画彩虹文本编辑器效果 |
| `interactive-shell.ts` | 运行交互式命令（vim、htop） |
| `notify.ts` | 助手完成时桌面通知（OSC 777） |

### Git 集成

| 扩展 | 说明 |
|------|------|
| `git-checkpoint.ts` | 每轮对话创建 git stash 检查点 |
| `auto-commit-on-exit.ts` | 退出时自动提交 |

### 自定义提供商

| 扩展 | 说明 |
|------|------|
| `custom-provider-anthropic/` | 自定义 Anthropic 提供商 + OAuth |
| `custom-provider-gitlab-duo/` | GitLab Duo 提供商 |
| `custom-provider-qwen-cli/` | Qwen CLI 提供商（OAuth 设备流） |

---

## 2. 社区热门包（Gallery Top 50）

数据来源：pi.dev/packages（按下载量排序，截至 2026-04-29）

### 集成 & 工具扩展

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-mcp-adapter** (`nicopreme`) | 13.5K/mo | MCP（Model Context Protocol）适配器扩展 |
| **pi-lens** (`apmantza`) | 12.3K/mo | 实时代码反馈：LSP、linter、formatter、类型检查 |
| **pi-lean-ctx** (`pounce_ch`) | 3.1K/mo | MCP 集成，48+ 工具（ctx_session, ctx_knowledge 等） |
| **context-mode** (`mksglu`) | 40.1K/mo | MCP 插件，节省 98% 上下文窗口 |
| **pi-web-access** (`nicopreme`) | 6.1K/mo | 网页搜索、URL 抓取、GitHub 仓库克隆、PDF 提取、YouTube |

### 子代理 & 编排

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-subagents** (`nicopreme`) | 38.1K/mo | 委托任务给子代理，支持链式/并行执行 |
| **taskplane** (`henrylach`) | 21.1K/mo | AI 代理编排，并行任务执行 |
| **pi-crew** | 最新 | 协调 AI 团队、工作流、工作树、异步任务编排 |
| **pi-messenger-swarm** (`monotykamary`) | 1.9K/mo | 基于 swarm 的多代理消息和任务编排 |
| **@ifi/oh-pi-ant-colony** (`ifiokjr`) | 1.9K/mo | 自适应并发、信息素通信的自主多代理 swarm |

### 搜索 & 知识获取

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **@apmantza/greedysearch-pi** | 6.0K/mo | 多引擎 AI 搜索（Perplexity, Bing, Google AI），无需 API Key |
| **pi-oracle** (`fitchmultz`) | 2.3K/mo | ChatGPT Web 预言机扩展 |
| **dripline** (`miclivs`) | 4.7K/mo | 查询任何内容 |

### 增强 UI & 交互

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-powerline-footer** (`nicopreme`) | 5.4K/mo | Powerline 风格状态栏 |
| **pi-interactive-shell** (`nicopreme`) | 3.2K/mo | 在 overlay 中运行交互式子 shell |
| **pi-ask-user** (`edlsh`) | 2.3K/mo | 交互式 `ask_user` 工具，搜索式分面板选择 UI |
| **pi-markdown-preview** (`omacl`) | 2.7K/mo | 渲染的 Markdown + LaTeX 预览 |
| **pi-studio** (`omacl`) | 6.6K/mo | 双面板浏览器工作区 |
| **pi-screenshots-picker** (`graffioh`) | 3.0K/mo | 截图选择器 |
| **@codexstar/pi-pompom** (`engaze`) | 19.2K/mo | 3D 光线投射虚拟宠物（游戏化交互） |

### 工作流 & 开发管道

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-gsd** (`fulgidus`) | 9.6K/mo | "Get Shit Done" - 项目规划规范驱动工具包 |
| **@callumvass/forgeflow-dev** | 4.9K/mo | 开发管道：TDD 实现、代码审查、架构 |
| **@callumvass/forgeflow-pm** | 3.0K/mo | PM 管道：PRD 细化、问题创建 |
| **@feniix/pi-specdocs** (`feniix`) | 2.2K/mo | 结构化规范文档：PRD、ADR、实现计划 |
| **@ifi/pi-spec** (`ifiokjr`) | 1.8K/mo | 原生规范工具包与 `/spec` 命令 |
| **omni-pi** (`edgy2k`) | 1.7K/mo | 面试用户、记录规范、分片实现 |

### 语音 & 媒体

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **@codexstar/pi-listen** (`engaze`) | 5.1K/mo | 按住说话语音输入，19 种离线模型 |
| **glimpseui** (`haza`) | 2.1K/mo | 原生微 UI，跨平台 WebView 窗口 |

### 安全 & 防护

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **@aliou/pi-guardrails** | 3.3K/mo | 安全钩子，减少意外破坏性操作 |

### 主题 & 预设

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **oh-pi** (`telagod`) | 9.4K/mo | 一键设置，类似 oh-my-zsh |
| **@ifi/oh-pi-themes** (`ifiokjr`) | 2.1K/mo | 颜色主题：cyberpunk、nord、gruvbox、tokyo-night 等 |
| **@ifi/oh-pi-skills** (`ifiokjr`) | 1.9K/mo | 按需技能包：web-search、debug-helper、git-workflow |
| **@ifi/oh-pi-prompts** (`ifiokjr`) | 1.9K/mo | 提示模板：review、fix、explain、refactor、test、commit |
| **pi-hosts** | 最新 | 在远程主机上执行命令 |

### 提示建议 & 助手

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **@guwidoe/pi-prompt-suggester** | 1.9K/mo | 意图感知的下一个提示建议 |
| **@rpollard00/pi-materia** | 最新 | 可配置的 materia 主题代理管道 |

### 管理 & 代理工具

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-extmgr** (`ayagmar`) | 2.9K/mo | 增强的本地扩展和社区包管理 UX |
| **pi-messenger** (`nicopreme`) | 2.4K/mo | 代理间消息和文件保留系统 |
| **pi-prompt-template-model** (`nicopreme`) | 3.7K/mo | 提示模板模型选择器 |
| **@victor-software-house/pi-openai-proxy** | 4.4K/mo | OpenAI 兼容 HTTP 代理 |
| **@victor-software-house/pi-multicodex** | 3.4K/mo | Codex 账户轮换 |
| **@juanibiapina/pi-extension-settings** | 3.9K/mo | 跨扩展集中设置管理 |
| **@tintinweb/pi-subagents** | 4.1K/mo | Claude Code 风格自主子代理 |

### 提供商扩展

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-nvidia-nim** (`xryul`) | 1.7K/mo | NVIDIA NIM API 提供商，100+ 模型 |
| **@sting8k/pi-vcc** | 1.7K/mo | 算法化对话压缩，无需 LLM 调用 |
| **@aliou/pi-synthetic** | 2.2K/mo | 合成数据扩展 |
| **@0xkobold/pi-kobold** | 2.1K/mo | 元扩展，捆绑编排、网关、Ollama 等 |
| **my-pi** (`spences10`) | 3.1K/mo | 可组合 pi，含 MCP、LSP、提示预设、本地评估遥测 |

---

## 3. 使用安装与管理

```bash
# 安装包（全局）
pi install npm:pi-mcp-adapter

# 安装包（项目级，写入 .pi/settings.json）
pi install npm:pi-powerline-footer -l

# 列出已安装包
pi list

# 卸载
pi remove npm:some-package

# 更新
pi update                 # 更新所有
pi update --extension npm:pi-lens  # 更新指定扩展

# 临时试用（不安装）
pi -e npm:pi-web-access

# 启用/禁用包资源
pi config
```

### 在 settings.json 中管理

```json
{
  "packages": [
    "npm:pi-subagents",
    "npm:pi-mcp-adapter",
    "npm:pi-powerline-footer"
  ],
  "extensions": ["/path/to/local-extension.ts"],
  "skills": ["~/.claude/skills", "~/.codex/skills"],
  "prompts": ["./project-prompts/review.md"],
  "themes": ["my-theme"]
}
```

### 包筛选

```json
{
  "packages": [
    "npm:simple-pkg",
    {
      "source": "npm:my-package",
      "extensions": ["extensions/*.ts", "!extensions/legacy.ts"],
      "skills": [],
      "prompts": ["prompts/review.md"]
    }
  ]
}
```

---

## 4. 资源索引

| 资源 | 链接 |
|------|------|
| 包 Gallery（1852+ 包） | https://pi.dev/packages |
| 官方示例扩展（50+） | https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent/examples/extensions |
| SDK 示例 | https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent/examples/sdk |
| Anthropic 技能仓库 | https://github.com/anthropics/skills |
| Pi 技能仓库 | https://github.com/badlogic/pi-skills |
| 官方文档 | https://pi.dev/docs/latest |
| Discord 社区 | https://discord.com/invite/3cU7Bz4UPx |
| GitHub 仓库 | https://github.com/badlogic/pi-mono |
| 会话分享与研究 | https://github.com/badlogic/pi-share-hf |
| Agent Skills 标准 | https://agentskills.io/specification |