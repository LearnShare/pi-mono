# Configuration System - pi-coding-agent

## 1. 执行摘要

pi-coding-agent 采用分层配置系统，支持从全局到项目再到运行时参数的多级配置覆盖。核心由 `SettingsManager` (settings-manager.ts:241-1067) 管理配置合并和持久化，`AuthStorage` (auth-storage.ts:191-524) 处理敏感凭证。配置优先级为：默认值 < 全局 settings.json < 项目 .pi/settings.json < 环境变量 < CLI 参数。敏感信息（API Key）存储在 auth.json，采用文件锁和 0600 权限保护，支持 shell 命令 (!) 动态获取和环境变量回退。

## 2. 核心数据结构

### 2.1 配置存储结构

#### 全局配置 (~/.pi/agent/settings.json)
```json
{
  "defaultProvider": "google",
  "defaultModel": "gemini-2.0-flash",
  "defaultThinkingLevel": "medium",
  "transport": "sse",
  "steeringMode": "one-at-a-time",
  "followUpMode": "one-at-a-time",
  "theme": "default",
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  },
  "retry": {
    "enabled": true,
    "maxRetries": 3,
    "baseDelayMs": 2000,
    "provider": {
      "timeoutMs": 60000,
      "maxRetries": 3,
      "maxRetryDelayMs": 60000
    }
  },
  "terminal": {
    "showImages": true,
    "imageWidthCells": 60,
    "clearOnShrink": false,
    "showTerminalProgress": false
  },
  "images": {
    "autoResize": true,
    "blockImages": false
  },
  "extensions": [],
  "skills": [],
  "prompts": [],
  "themes": [],
  "packages": []
}
```

#### 项目配置 (.pi/settings.json)
与全局配置相同结构，优先级更高，用于项目级别覆盖。

#### 认证存储 (~/.pi/agent/auth.json)
```json
{
  "anthropic": { "type": "api_key", "key": "sk-ant-..." },
  "google": { "type": "oauth", "accessToken": "...", "refreshToken": "...", "expires": 1234567890 }
}
```

#### 自定义模型 (~/.pi/agent/models.json)
```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434",
      "apiKey": "!echo local-ollama-key",
      "models": [
        { "id": "llama3", "name": "Llama 3", "reasoning": true, "input": ["text"], "contextWindow": 128000 }
      ]
    }
  }
}
```

### 2.2 Settings 接口 (settings-manager.ts:76-113)

```typescript
interface Settings {
  lastChangelogVersion?: string;
  defaultProvider?: string;
  defaultModel?: string;
  defaultThinkingLevel?: "off" | "minimal" | "low" | "medium" | "high" | "xhigh";
  transport?: "sse" | "websocket";
  steeringMode?: "all" | "one-at-a-time";
  followUpMode?: "all" | "one-at-a-time";
  theme?: string;
  compaction?: CompactionSettings;
  branchSummary?: BranchSummarySettings;
  retry?: RetrySettings;
  hideThinkingBlock?: boolean;
  shellPath?: string;
  quietStartup?: boolean;
  shellCommandPrefix?: string;
  npmCommand?: string[];
  collapseChangelog?: boolean;
  enableInstallTelemetry?: boolean;
  packages?: PackageSource[];
  extensions?: string[];
  skills?: string[];
  prompts?: string[];
  themes?: string[];
  enableSkillCommands?: boolean;
  terminal?: TerminalSettings;
  images?: ImageSettings;
  enabledModels?: string[];
  doubleEscapeAction?: "fork" | "tree" | "none";
  treeFilterMode?: "default" | "no-tools" | "user-only" | "labeled-only" | "all";
  thinkingBudgets?: ThinkingBudgetsSettings;
  editorPaddingX?: number;
  autocompleteMaxVisible?: number;
  showHardwareCursor?: boolean;
  markdown?: MarkdownSettings;
  warnings?: WarningSettings;
  sessionDir?: string;
}
```

### 2.3 CLI 参数 (args.ts:12-51)

```typescript
interface Args {
  provider?: string;
  model?: string;
  apiKey?: string;
  systemPrompt?: string;
  appendSystemPrompt?: string[];
  thinking?: ThinkingLevel;
  continue?: boolean;
  resume?: boolean;
  session?: string;
  fork?: string;
  sessionDir?: string;
  models?: string[];
  tools?: string[];
  noTools?: boolean;
  extensions?: string[];
  skills?: string[];
  promptTemplates?: string[];
  themes?: string[];
}
```

## 3. 关键流程图

### 3.1 配置加载与合并流程

```
启动阶段
---------------------------------------------------------------------------

  默认值          全局配置            项目配置
  defaults.ts  -> settings.json -> .pi/settings.json
       |            |                  |
       +------------+------------------+
                    |
                    V
           deepMergeSettings (L116-144)
                    |
    +---------------+---------------+
    |               |               |
 环境变量        CLI 参数        /set 命令
 PI_*            --model         运行时覆盖
    |               |               |
    +---------------+---------------+
                    |
                    V
           最终合并配置
           SettingsManager
```

### 3.2 API Key 解析优先级

```
getApiKey() 优先级 (auth-storage.ts:455-516)
---------------------------------------------------------------------------

   1. Runtime Override (CLI --api-key)
      |
      V
   2. auth.json API Key
      |
      V
   3. auth.json OAuth Token (自动刷新)
      |
      V
   4. 环境变量 (ANTHROPIC_API_KEY, etc.)
      |
      V
   5. models.json apiKey 配置 (!shell 命令或环境变量引用)
      |
      V
   6. models.json 自定义 provider fallback resolver
```

### 3.3 会话级服务创建流程 (agent-session-services.ts:129-170)

```
createAgentSessionServices()
---------------------------------------------------------------------------

   1. 初始化 SettingsManager (从全局+项目 settings.json)
      settings-manager.ts:273-276
      
   2. 初始化 AuthStorage (从 auth.json)
      auth-storage.ts:202-204
      
   3. 初始化 ModelRegistry (加载内置+自定义模型)
      model-registry.ts:329-331
      
   4. 初始化 ResourceLoader (加载扩展/技能/主题/包资源)
      resource-loader.ts:137-142
      
   5. 注册扩展提供的 providers
      model-registry.ts:147-158
      
   6. 应用扩展 flag 值
      agent-session-services.ts:76-122
      
   7. 返回 AgentSessionServices
      { cwd, agentDir, authStorage, settingsManager,
        modelRegistry, resourceLoader, diagnostics }
```

## 4. 源码位置

### 4.1 配置核心

| 功能 | 文件 | 行号 |
|------|------|------|
| Settings 接口定义 | settings-manager.ts | 76-113 |
| SettingsManager 类 | settings-manager.ts | 241-1067 |
| 配置深度合并 | settings-manager.ts | 116-144 |
| 文件存储实现 | settings-manager.ts | 157-222 |
| 热重载 | settings-manager.ts | 403-429 |
| 配置迁移 | settings-manager.ts | 334-393 |

### 4.2 认证存储

| 功能 | 文件 | 行号 |
|------|------|------|
| AuthStorage 类 | auth-storage.ts | 191-524 |
| 文件后端 | auth-storage.ts | 52-166 |
| API Key 解析 | auth-storage.ts | 455-516 |
| OAuth 刷新 | auth-storage.ts | 400-444 |

### 4.3 模型注册

| 功能 | 文件 | 行号 |
|------|------|------|
| ModelRegistry 类 | model-registry.ts | 315-890 |
| 自定义模型加载 | model-registry.ts | 438-491 |
| API Key 获取 | model-registry.ts | 663-702 |

### 4.4 配置解析

| 功能 | 文件 | 行号 |
|------|------|------|
| shell/环境变量解析 | resolve-config-value.ts | 17-86 |
| 命令缓存 | resolve-config-value.ts | 78-86 |

### 4.5 CLI 参数

| 功能 | 文件 | 行号 |
|------|------|------|
| 参数解析 | args.ts | 59-184 |
| 帮助输出 | args.ts | 186-342 |

### 4.6 路径常量

| 功能 | 文件 | 行号 |
|------|------|------|
| 配置文件路径 | config.ts | 341-398 |
| 配置目录常量 | config.ts | 325 |

## 5. 最佳实践

### 5.1 全局配置 (用户级)

推荐在 `~/.pi/agent/settings.json` 设置：
```json
{
  "defaultProvider": "google",
  "defaultModel": "gemini-2.0-flash",
  "defaultThinkingLevel": "medium",
  "quietStartup": true,
  "collapseChangelog": true
}
```

### 5.2 项目配置 (团队级)

在项目根目录 `.pi/settings.json` 覆盖：
```json
{
  "defaultProvider": "anthropic",
  "enabledModels": ["claude-sonnet", "claude-haiku"],
  "extensions": ["./pi-extensions"],
  "skills": ["./pi-skills"]
}
```

### 5.3 敏感信息管理

1. 使用环境变量 (推荐)：
   ```bash
   export ANTHROPIC_API_KEY=sk-ant-...
   ```

2. 使用 auth.json：
   ```bash
   pi install anthropic  # 交互式登录
   ```

3. 使用 shell 命令动态获取 (models.json)：
   ```json
   {
     "providers": {
       "ollama": {
         "apiKey": "!echo local-key"
       }
     }
   }
   ```

### 5.4 运行时覆盖

- CLI 参数最高优先级：`pi --provider openai --model gpt-4o`
- 会话内可用 `/set` 命令修改配置
- 环境变量覆盖部分设置：`PI_CLEAR_ON_SHRINK=1`, `PI_HARDWARE_CURSOR=1`

### 5.5 自定义模型配置

```json
// ~/.pi/agent/models.json
{
  "providers": {
    "local-ollama": {
      "baseUrl": "http://localhost:11434",
      "models": [
        {
          "id": "llama3:70b",
          "name": "Llama 3 70B",
          "reasoning": true,
          "contextWindow": 128000
        }
      ]
    },
    "openai-compatible": {
      "baseUrl": "https://api.example.com/v1",
      "apiKey": "!cat ~/.api-key",
      "headers": {
        "X-Custom-Header": "value"
      },
      "compat": {
        "thinkingFormat": "deepseek"
      }
    }
  }
}
```

## 6. 环境变量参考

### 6.1 pi 系统变量

| 变量 | 作用 | 默认值 |
|------|------|--------|
| PI_CODING_AGENT_DIR | Agent 数据目录 | ~/.pi/agent |
| PI_PACKAGE_DIR | 包资源目录 | 自动检测 |
| PI_OFFLINE | 禁用网络操作 | false |
| PI_TELEMETRY | 匿名使用统计 | true |
| PI_SHARE_VIEWER_URL | /share 查看器 URL | https://pi.dev/session/ |
| PI_CLEAR_ON_SHRINK | 终端内容收缩时清空 | false |
| PI_HARDWARE_CURSOR | 硬件光标 | false |
| PI_STARTUP_BENCHMARK | 启动性能分析 | false |
| PI_TIMING | 详细时序日志 | false |

### 6.2 API Key 环境变量

详见 args.ts:298-331，支持的 Provider：
- ANTHROPIC_API_KEY
- OPENAI_API_KEY
- AZURE_OPENAI_API_KEY
- DEEPSEEK_API_KEY
- GEMINI_API_KEY
- GROQ_API_KEY
- MISTRAL_API_KEY
- AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY
- 20+ 更多...

## 7. 未来改进方向

### 7.1 配置验证
- 引入 JSON Schema 验证 (当前使用 TypeBox in model-registry.ts:194)
- 配置错误时的友好提示
- 配置迁移自动化

### 7.2 安全增强
- 支持 keychain/secure storage 集成
- API Key 加密存储
- 敏感配置项隐藏 (e.g., --api-key 不回显)

### 7.3 用户体验
- pi config 命令 TUI 配置界面
- 配置变更的热重载
- 配置冲突检测与建议
- 配置模板/预设系统

### 7.4 可扩展性
- 扩展自定义配置项
- 配置版本管理
- 配置同步 (跨设备)
- 环境特定配置 (dev/staging/prod)