# 扩展开发入门

> 基于 pi.dev/docs/latest 官方文档整理

---

## 1. 快速开始

### 创建扩展

在以下位置创建扩展文件：

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/extensions/*.ts` | 全局 |
| `.pi/extensions/*.ts` | 项目级 |

### 基本结构

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  // 扩展代码
}
```

### 异步扩展

用于启动时获取远程配置等：

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

export default async function (pi: ExtensionAPI) {
  const response = await fetch("http://localhost:1234/v1/models");
  const data = await response.json();
  pi.registerProvider("local-openai", { ... });
}
```

---

## 2. 多文件扩展

### 目录结构

```
~/.pi/agent/extensions/my-extension/
├── index.ts      # 入口
├── tools.ts      # 辅助模块
└── utils.ts
```

---

## 3. 带依赖的包

### package.json

```json
{
  "name": "my-extension",
  "dependencies": { "zod": "^3.0.0" },
  "pi": {
    "extensions": ["./src/index.ts"]
  }
}
```

---

## 4. 常用导入

| 包 | 用途 |
|------|------|
| `@mariozechner/pi-coding-agent` | 扩展类型（`ExtensionAPI`, `ExtensionContext`, 事件） |
| `typebox` | 工具参数 Schema 定义 |
| `@mariozechner/pi-ai` | AI 工具（`StringEnum` 等） |
| `@mariozechner/pi-tui` | TUI 组件 |
| `node:*` | Node.js 内置模块 |

---

## 5. 常见用例

### 注册自定义工具

```typescript
import { Type } from "typebox";

pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "Does something useful",
  parameters: Type.Object({
    input: Type.String({ description: "Input value" }),
  }),
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    return {
      content: [{ type: "text", text: `Result: ${params.input}` }],
      details: {},
    };
  },
});
```

### 监听事件

```typescript
pi.on("tool_call", async (event, ctx) => {
  // 处理工具调用
});

pi.on("message_created", async (event, ctx) => {
  // 处理消息创建
});
```

### 注册命令

```typescript
pi.registerCommand("hello", {
  description: "Say hello",
  handler: async (args, ctx) => {
    ctx.ui.notify(`Hello ${args || "world"}!`, "info");
  },
});
```

### 用户交互

```typescript
// 确认对话框
const ok = await ctx.ui.confirm("Title", "Message");

// 下拉选择
const result = await ctx.ui.select("Title", ["Option 1", "Option 2"]);

// 文本输入
const input = await ctx.ui.input("Prompt", "default value");
```

---

## 6. 扩展位置汇总

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/extensions/*.ts` | 全局单文件 |
| `~/.pi/agent/extensions/*/index.ts` | 全局目录 |
| `.pi/extensions/*.ts` | 项目级单文件 |
| `.pi/extensions/*/index.ts` | 项目级目录 |

通过 settings.json 指定额外路径：

```json
{
  "extensions": ["/path/to/extension.ts"]
}
```