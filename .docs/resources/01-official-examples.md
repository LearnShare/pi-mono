# 官方示例

> 基于 GitHub 官方示例整理

---

Pi 官方仓库提供 50+ 示例扩展，按类别分类：

## 生命周期 & 安全

| 扩展 | 说明 |
|------|------|
| `permission-gate.ts` | 危险命令前提示确认（rm -rf、sudo 等） |
| `protected-paths.ts` | 阻止写入受保护路径（.env、.git/、node_modules/） |
| `confirm-destructive.ts` | 破坏性会话操作前确认 |
| `dirty-repo-guard.ts` | 有未提交 git 更改时阻止会话切换 |
| `sandbox/` | OS 级沙盒，使用 `@anthropic-ai/sandbox-runtime` |

## 自定义工具

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

## 命令 & UI

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

## Git 集成

| 扩展 | 说明 |
|------|------|
| `git-checkpoint.ts` | 每轮对话创建 git stash 检查点 |
| `auto-commit-on-exit.ts` | 退出时自动提交 |

## 自定义提供商

| 扩展 | 说明 |
|------|------|
| `custom-provider-anthropic/` | 自定义 Anthropic 提供商 + OAuth |
| `custom-provider-gitlab-duo/` | GitLab Duo 提供商 |
| `custom-provider-qwen-cli/` | Qwen CLI 提供商（OAuth 设备流） |