# 文档整理计划

> 目标：对 `.docs/` 中的文档进行分类、合并和内容整合
> 存放位置：`.docs/research-plan/03-documentation-organization-plan.md`

---

## 用户要求汇总

1. **合并重复文档**：不是简单删除，而是将重复/相似内容合并到目标文档
2. **分类存放**：根据相关主题和内容进行分类
3. **预留研究位置**：`02-detailed-research-plan.md` 的4个研究主题需要预留产出位置
4. **保留所有历史文档**：
   - `research/` 中的早期研究内容都有价值，全部保留
   - `research-plan/00-detailed-research-plan.md` 是已完成的历史研究计划，保留
   - `research-plan/02-detailed-research-plan.md` 是当前进行中的研究计划
5. **创建文档索引**：添加 `.docs/readme.md` 作为文档目录入口

---

## 一、当前文档状态

### 1.1 现有文件清单

```
.docs/
├── [根目录]
│   ├── pi-setup-customization-extension.md    # 安装/配置/自定义（546行）
│   ├── pi-popular-extensions-resources.md     # 社区资源（249行）
│   ├── pi-extensions-api-reference.md         # 扩展API（1124行）
│   └── pi-builtin-prompts-skills-research.md  # 内置提示词研究（599行）
│
├── pi-agent/                                   # 与根目录部分重复
│   ├── 01-setup-and-customization.md
│   ├── 02-popular-resources.md
│   └── 03-extension-development.md
│
├── research/                                   # 早期研究（全部保留）
│   ├── 00-research-strategy.md
│   ├── 01-architecture-overview.md
│   ├── 02-pi-ai-typesystem.md
│   ├── 03-agent-core.md
│   ├── 04-coding-agent-cli.md
│   ├── 05-tui-system.md
│   ├── 06-extensions-skills.md
│   └── 07-technical-summary.md
│
└── research-plan/
    ├── 00-detailed-research-plan.md           # 历史研究计划（已完成）
    ├── 01-research-checklist.md
    └── 02-detailed-research-plan.md           # 当前研究计划（进行中）
```

### 1.2 重复分析

| 根目录文档 | pi-agent/文档 | 重叠程度 |
|-----------|--------------|---------|
| `pi-setup-customization-extension.md` | `01-setup-and-customization.md` | ~90% |
| `pi-popular-extensions-resources.md` | `02-popular-resources.md` | ~60% |
| `pi-extensions-api-reference.md` | `03-extension-development.md` | ~40% |

---

## 二、分类方案

### 2.1 新目录结构

```
.docs/
├── user-guide/              # 面向终端用户：安装、使用、CLI
│   ├── 01-installation.md
│   ├── 02-basic-usage.md
│   └── 03-cli-commands.md
│
├── configuration/           # 面向配置用户：settings、自定义、providers
│   ├── 01-settings.md
│   ├── 02-customization.md
│   └── 03-providers.md
│
├── extension-dev/           # 面向扩展开发者：开发入门、API参考
│   ├── 01-getting-started.md
│   └── 02-api-reference.md
│
├── resources/               # 面向寻找资源的用户：官方示例、社区包
│   ├── 01-official-examples.md
│   └── 02-community-packages.md
│
├── internals/               # 面向高级用户/贡献者：系统化文档
│   ├── 00-architecture.md
│   └── 01-builtin-prompts.md
│
├── research/                # 研究笔记（保留+新增）
│   ├── 00-research-strategy.md
│   ├── 01-architecture-overview.md
│   ├── 02-pi-ai-typesystem.md
│   ├── 03-agent-core.md
│   ├── 04-coding-agent-cli.md
│   ├── 05-tui-system.md
│   ├── 06-extensions-skills.md
│   ├── 07-technical-summary.md
│   ├── 08-session-lifecycle.md          # 研究产出 #1
│   ├── 09-provider-communication.md     # 研究产出 #2
│   ├── 10-tool-execution.md             # 研究产出 #3
│   └── 11-prompt-skill-system.md        # 研究产出 #4
│
└── research-plan/
    ├── 00-detailed-research-plan.md     # 历史（已完成）
    ├── 01-research-checklist.md
    └── 02-detailed-research-plan.md     # 当前（进行中）
```

### 2.2 分类原则

| 目录 | 面向对象 | 内容 |
|------|---------|------|
| `user-guide/` | 终端用户 | 安装、认证、交互模式、CLI |
| `configuration/` | 配置用户 | settings、6种自定义机制、providers |
| `extension-dev/` | 扩展开发者 | 扩展开发入门、完整API参考 |
| `resources/` | 寻找资源用户 | 官方示例、社区热门包 |
| `internals/` | 高级用户 | 系统化整理后的文档 |
| `research/` | 研究者 | 所有研究笔记（历史+新产出） |
| `research-plan/` | 规划者 | 历史+当前研究计划 |

---

## 三、合并策略（核心原则：合并而非删除）

### 3.1 用户指南合并

| 源文档 | 目标文档 | 操作 |
|--------|---------|------|
| 根目录 `pi-setup-customization-extension.md` 第1-4节 | `user-guide/01-installation.md` | 提取合并 |
| 根目录 `pi-setup-customization-extension.md` 第5-6节 | `user-guide/02-basic-usage.md` | 提取合并 |
| `pi-agent/01-setup-and-customization.md` | 用户指南相关文档 | 逐节对比，合并去重 |
| `research/04-coding-agent-cli.md` | `user-guide/03-cli-commands.md` | 提取CLI内容 |

### 3.2 配置文档合并

| 源文档 | 目标文档 | 操作 |
|--------|---------|------|
| 根目录 `pi-setup-customization-extension.md` 第7-8节 | `configuration/01-settings.md` + `configuration/02-customization.md` | 拆分提取 |
| 根目录 `pi-setup-customization-extension.md` 第6节providers部分 | `configuration/03-providers.md` | 提取 |

### 3.3 扩展开发合并

| 源文档 | 目标文档 | 操作 |
|--------|---------|------|
| 根目录 `pi-extensions-api-reference.md` | `extension-dev/02-api-reference.md` | 重命名迁移 |
| `pi-agent/03-extension-development.md` | `extension-dev/01-getting-started.md` | 提取入门内容 |
| `research/06-extensions-skills.md` | `extension-dev/01-getting-started.md` | 整合 |

### 3.4 资源文档合并

| 源文档 | 目标文档 | 操作 |
|--------|---------|------|
| 根目录 `pi-popular-extensions-resources.md` 第1节 | `resources/01-official-examples.md` | 提取 |
| 根目录 `pi-popular-extensions-resources.md` 第2节 | `resources/02-community-packages.md` | 提取 |
| `pi-agent/02-popular-resources.md` | `resources/02-community-packages.md` | 合并去重 |

### 3.5 内部机制文档

| 源文档 | 目标文档 | 操作 |
|--------|---------|------|
| `research/01-architecture-overview.md` | `internals/00-architecture.md` | 整合 |
| `research/02-pi-ai-typesystem.md` | `internals/00-architecture.md` | 整合补充 |
| `research/03-agent-core.md` | `internals/00-architecture.md` | 整合补充 |
| 根目录 `pi-builtin-prompts-skills-research.md` | `internals/01-builtin-prompts.md` | 重命名迁移 |

### 3.6 完全保留的文档（不删除）

| 文档 | 原因 |
|------|------|
| `research/00-research-strategy.md` | 研究方法论参考 |
| `research/05-tui-system.md` | TUI系统分析 |
| `research/07-technical-summary.md` | 技术总结 |
| `research-plan/00-detailed-research-plan.md` | 历史研究计划（已完成） |
| `research-plan/01-research-checklist.md` | 研究检查清单 |
| `research-plan/02-detailed-research-plan.md` | 当前研究计划 |

### 3.7 待删除

| 文件 | 原因 |
|------|------|
| `pi-agent/` 整个目录 | 内容已合并到其他文档 |

### 3.8 研究产出位置（预留）

根据 `02-detailed-research-plan.md`，4个研究主题的产出位置：

| 研究主题 | 输出文件 |
|---------|---------|
| 主题1: Session生命周期 | `research/08-session-lifecycle.md` |
| 主题2: Provider通信 | `research/09-provider-communication.md` |
| 主题3: Tool执行流程 | `research/10-tool-execution.md` |
| 主题4: Prompt/Skill系统 | `research/11-prompt-skill-system.md` |

---

## 四、执行步骤

### 阶段1：创建目录结构

```bash
mkdir -p .docs/user-guide
mkdir -p .docs/configuration
mkdir -p .docs/extension-dev
mkdir -p .docs/resources
mkdir -p .docs/internals
```

创建 `.docs/readme.md` 作为文档目录索引。

### 阶段2：合并用户指南

1. 创建 `user-guide/01-installation.md`（提取根目录文档第1-3节）
2. 创建 `user-guide/02-basic-usage.md`（提取根目录文档第4节）
3. 创建 `user-guide/03-cli-commands.md`（整合 research/04）
4. 合并 `pi-agent/01-setup-and-customization.md` 相关内容

### 阶段3：合并配置文档

1. 创建 `configuration/01-settings.md`
2. 创建 `configuration/02-customization.md`
3. 创建 `configuration/03-providers.md`

### 阶段4：合并扩展开发文档

1. 创建 `extension-dev/01-getting-started.md`
2. 迁移 `pi-extensions-api-reference.md` → `extension-dev/02-api-reference.md`
3. 整合 `pi-agent/03-extension-development.md` 和 `research/06`

### 阶段5：合并资源文档

1. 创建 `resources/01-official-examples.md`
2. 创建 `resources/02-community-packages.md`

### 阶段6：处理内部机制文档

1. 创建 `internals/00-architecture.md`（整合 research/01-03）
2. 迁移 `pi-builtin-prompts-skills-research.md` → `internals/01-builtin-prompts.md`

### 阶段7：清理与收尾

1. 删除 `pi-agent/` 目录
2. 更新所有文档的交叉引用

---

## 五、时间估算

| 阶段 | 任务 | 预估时间 |
|------|------|---------|
| 1 | 创建目录 + readme | 10分钟 |
| 2 | 用户指南合并 | 30分钟 |
| 3 | 配置文档合并 | 20分钟 |
| 4 | 扩展开发合并 | 25分钟 |
| 5 | 资源文档合并 | 15分钟 |
| 6 | 内部机制文档 | 20分钟 |
| 7 | 清理与引用 | 25分钟 |

**总计：约 2.5 小时**

---

## 六、验收标准

- [ ] 目录结构符合分类方案
- [ ] 无内容丢失（所有原始内容已合并）
- [ ] 重复内容已去重整合
- [ ] `.docs/readme.md` 已创建
- [ ] 研究产出位置已预留（research/08-11）
- [ ] 所有历史研究文档均保留
- [ ] pi-agent/ 目录已删除
- [ ] 文档间交叉引用正确