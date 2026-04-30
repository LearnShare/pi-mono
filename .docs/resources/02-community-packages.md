# 社区热门包

> 数据来源：pi.dev/packages（1852+ 包），截至 2026-04-29

---

## 安装与管理

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
pi update
pi update --extension npm:pi-lens

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

---

## 热门包分类

### 集成 & 工具扩展

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-mcp-adapter** (`nicopreme`) | 13.5K/月 | MCP（Model Context Protocol）适配器扩展 |
| **pi-lens** (`apmantza`) | 12.3K/月 | 实时代码反馈：LSP、linter、formatter、类型检查 |
| **pi-lean-ctx** (`pounce_ch`) | 3.1K/月 | MCP 集成，48+ 工具（ctx_session, ctx_knowledge 等） |
| **context-mode** (`mksglu`) | 40.1K/月 | MCP 插件，节省 98% 上下文窗口 |
| **pi-web-access** (`nicopreme`) | 6.1K/月 | 网页搜索、URL 抓取、GitHub 仓库克隆、PDF 提取、YouTube |

### 子代理 & 编排

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-subagents** (`nicopreme`) | 38.1K/月 | 委托任务给子代理，支持链式/并行执行 |
| **taskplane** (`henrylach`) | 21.1K/月 | AI 代理编排，并行任务执行 |
| **pi-crew** | 最新 | 协调 AI 团队、工作流、工作树、异步任务编排 |
| **pi-messenger-swarm** (`monotykamary`) | 1.9K/月 | 基于 swarm 的多代理消息和任务编排 |
| **@ifi/oh-pi-ant-colony** (`ifiokjr`) | 1.9K/月 | 自适应并发、信息素通信的自主多代理 swarm |

### 搜索 & 知识获取

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **@apmantza/greedysearch-pi** | 6.0K/月 | 多引擎 AI 搜索（Perplexity, Bing, Google AI），无需 API Key |
| **pi-oracle** (`fitchmultz`) | 2.3K/月 | ChatGPT Web 预言机扩展 |
| **dripline** (`miclivs`) | 4.7K/月 | 查询任何内容 |

### 增强 UI & 交互

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-powerline-footer** (`nicopreme`) | 5.4K/月 | Powerline 风格状态栏 |
| **pi-interactive-shell** (`nicopreme`) | 3.2K/月 | 在 overlay 中运行交互式子 shell |
| **pi-ask-user** (`edlsh`) | 2.3K/月 | 交互式 `ask_user` 工具，搜索式分面板选择 UI |
| **pi-markdown-preview** (`omacl`) | 2.7K/月 | 渲染的 Markdown + LaTeX 预览 |
| **pi-studio** (`omacl`) | 6.6K/月 | 双面板浏览器工作区 |
| **pi-screenshots-picker** (`graffioh`) | 3.0K/月 | 截图选择器 |
| **@codexstar/pi-pompom** (`engaze`) | 19.2K/月 | 3D 光线投射虚拟宠物（游戏化交互） |

### 工作流 & 开发管道

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-gsd** (`fulgidus`) | 9.6K/月 | "Get Shit Done" - 项目规划规范驱动工具包 |
| **@callumvass/forgeflow-dev** | 4.9K/月 | 开发管道：TDD 实现、代码审查、架构 |
| **@callumvass/forgeflow-pm** | 3.0K/月 | PM 管道：PRD 细化、问题创建 |
| **@feniix/pi-specdocs** (`feniix`) | 2.2K/月 | 结构化规范文档：PRD、ADR、实现计划 |
| **@ifi/pi-spec** (`ifiokjr`) | 1.8K/月 | 原生规范工具包与 `/spec` 命令 |
| **omni-pi** (`edgy2k`) | 1.7K/月 | 面试用户、记录规范、分片实现 |

### 主题 & 预设

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **oh-pi** (`telagod`) | 9.4K/月 | 一键设置，类似 oh-my-zsh |
| **@ifi/oh-pi-themes** (`ifiokjr`) | 2.1K/月 | 颜色主题：cyberpunk、nord、gruvbox、tokyo-night 等 |
| **@ifi/oh-pi-skills** (`ifiokjr`) | 1.9K/月 | 按需技能包：web-search、debug-helper、git-workflow |
| **@ifi/oh-pi-prompts** (`ifiokjr`) | 1.9K/月 | 提示模板：review、fix、explain、refactor、test、commit |

### 管理 & 代理工具

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-extmgr** (`ayagmar`) | 2.9K/月 | 增强的本地扩展和社区包管理 UX |
| **pi-messenger** (`nicopreme`) | 2.4K/月 | 代理间消息和文件保留系统 |
| **pi-prompt-template-model** (`nicopreme`) | 3.7K/月 | 提示模板模型选择器 |
| **@victor-software-house/pi-openai-proxy** | 4.4K/月 | OpenAI 兼容 HTTP 代理 |
| **@victor-software-house/pi-multicodex** | 3.4K/月 | Codex 账户轮换 |
| **@juanibiapina/pi-extension-settings** | 3.9K/月 | 跨扩展集中设置管理 |
| **@tintinweb/pi-subagents** | 4.1K/月 | Claude Code 风格自主子代理 |

### 提供商扩展

| 包名 | 下载量 | 说明 |
|------|--------|------|
| **pi-nvidia-nim** (`xryul`) | 1.7K/月 | NVIDIA NIM API 提供商，100+ 模型 |
| **@sting8k/pi-vcc** | 1.7K/月 | 算法化对话压缩，无需 LLM 调用 |
| **@aliou/pi-synthetic** | 2.2K/月 | 合成数据扩展 |
| **@0xkobold/pi-kobold** | 2.1K/月 | 元扩展，捆绑编排、网关、Ollama 等 |
| **my-pi** (`spences10`) | 3.1K/月 | 可组合 pi，含 MCP、LSP、提示预设、本地评估遥测 |

---

## 资源索引

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