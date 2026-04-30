# CLI 命令参考

> 基于 pi.dev/docs/latest 官方文档整理

---

## 1. 基本用法

```bash
pi [options] [@files...] [messages...]
```

---

## 2. 模式参数

| 参数 | 说明 |
|------|------|
| `-p`, `--print` | 打印响应后退出（非交互模式） |
| `--mode json` | JSON 事件流模式 |
| `--mode rpc` | RPC 模式 |

---

## 3. 会话参数

| 参数 | 说明 |
|------|------|
| `-c`, `--continue` | 继续最近的会话 |
| `-r`, `--resume` | 浏览选择会话 |
| `--session <path\|id>` | 打开特定会话 |
| `--fork <session>` | 从指定会话创建分支 |

---

## 4. 模型参数

| 参数 | 说明 |
|------|------|
| `--provider <name>` | 指定提供商（如 `anthropic`、`openai`） |
| `--model <pattern>` | 指定模型（支持 `provider/id` 和 `:thinking` 后缀） |
| `--thinking <level>` | 思考级别（`off`、`minimal`、`low`、`medium`、`high`、`xhigh`） |

示例：

```bash
pi --provider anthropic --model claude-sonnet-4-20250514
pi --model claude-sonnet-4-20250514:high  # 使用 high 思考级别
```

---

## 5. 工具参数

| 参数 | 说明 |
|------|------|
| `--tools <list>` | 工具白名单（逗号分隔） |
| `--no-tools` | 禁用所有工具 |

示例：

```bash
pi --tools read,edit,bash  # 只启用这3个工具
pi --no-tools              # 纯对话模式
```

---

## 6. 扩展与技能参数

| 参数 | 说明 |
|------|------|
| `-e`, `--extension <source>` | 加载扩展（文件或目录） |
| `--no-extensions` | 禁用扩展自动发现 |
| `--skill <path>` | 加载技能 |
| `--no-skills` | 禁用技能自动发现 |

示例：

```bash
pi -e ./my-extension.ts
pi -e ./my-extension-dir/
pi --skill ./my-skill/
```

---

## 7. 主题与上下文参数

| 参数 | 说明 |
|------|------|
| `--theme <path>` | 加载主题文件 |
| `--no-context-files` | 禁用 AGENTS.md/CLAUDE.md 发现 |

---

## 8. 其他参数

| 参数 | 说明 |
|------|------|
| `--verbose` | 详细输出 |
| `--offline` | 离线模式 |
| `--help` | 显示帮助 |
| `--version` | 显示版本 |

---

## 9. 管道用法

```bash
# 从文件读取
cat README.md | pi -p "Summarize this"

# 从图片输入
pi -p @screenshot.png "What's in this image?"

# 多文件输入
pi @file1.ts @file2.md "Compare these files"
```

---

## 10. 文件参数 (@files)

| 语法 | 说明 |
|------|------|
| `@filename` | 引用当前目录文件 |
| `@path/to/file` | 引用相对路径文件 |
| `@/absolute/path` | 引用绝对路径文件 |
| `@*.ts` | 通配符引用 |