# pi-tui 终端 UI 系统分析

## 一、概述

pi-tui 是 PI MONO 项目的终端 UI 库，提供基于终端的文本用户界面实现。其核心设计目标是**高效渲染**和**组件化**。

### 核心文件结构

```
packages/tui/src/
├── tui.ts                    # 核心 TUI 类和差分渲染引擎
├── components/               # 内置组件
│   ├── text.ts             # 文本显示
│   ├── box.ts             # 布局容器
│   ├── input.ts           # 输入组件
│   ├── editor.ts          # 编辑器组件
│   ├── markdown.ts        # Markdown 渲染
│   ├── select-list.ts     # 选择列表
│   ├── settings-list.ts   # 设置列表
│   ├── truncated-text.ts  # 截断文本
│   ├── spacer.ts         # 空白组件
│   ├── image.ts         # 图像组件
│   ├── loader.ts        # 加载指示器
│   └── cancellable-loader.ts
├── terminal.ts            # 终端接口抽象
├── terminal-image.ts    # 终端图像支持
├── keys.ts             # 键盘输入处理
├── keybindings.ts       # 键绑定管理
├── autocomplete.ts      # 自动补全
├── utils.ts           # 工具函数
└── index.ts          # 导出入口
```

## 二、差分渲染策略

pi-tui 的核心渲染策略在 `TUI.doRender()` 方法中实现,采用三层策略优化渲染性能。

### 2.1 第一层:完全重新渲染

在以下情况下执行完全重新渲染:

- **首次渲染**: 之前没有渲染过内容 (`previousLines.length === 0`)
- **终端宽度变化**: 宽度变化会改变文本换行,必须重新计算所有行
- **终端高度变化** (非 Termux 环境): 视口大小变化需要重新对齐
- **内容缩小且无 overlay**: 内容明显变少时清空多余行

```typescript
// packages/tui/src/tui.ts:918
const fullRender = (clear: boolean): void => {
    this.fullRedrawCount += 1;
    let buffer = "\x1b[?2026h"; // Begin synchronized output
    if (clear) buffer += "\x1b[2J\x1b[H\x1b[3J"; // Clear screen, home, clear scrollback
    // ... render all lines
    buffer += "\x1b[?2026l"; // End synchronized output
};
```

**同步输出模式**: 使用 `DECSC line ( Synchronized Output )` 序列 (`\x1b[?2026h/l`) 防止终端在渲染过程中刷新屏幕,避免闪烁。

### 2.2 第二层:差分渲染

当只有部分内容变化时,只更新变化的行:

1. 逐行比较 `previousLines` 和 `newLines`,找到 `firstChanged` 和 `lastChanged`
2. 只渲染从 `firstChanged` 到 `lastChanged` 的行
3. 清除多余的旧行
4. 更新硬件光标位置

```typescript
// packages/tui/src/tui.ts:984-998
for (let i = 0; i < maxLines; i++) {
    const oldLine = i < this.previousLines.length ? this.previousLines[i] : "";
    const newLine = i < newLines.length ? newLines[i] : "";
    if (oldLine !== newLine) {
        if (firstChanged === -1) firstChanged = i;
        lastChanged = i;
    }
}
```

### 2.3 第三层:无操作优化

当检测到完全没有变化时:

- 不执行任何渲染
- 仍然更新硬件光标位置 (支持 IME 输入法候选窗口)

```typescript
// packages/tui/src/tui.ts:1009
if (firstChanged === -1) {
    this.positionHardwareCursor(cursorPos, newLines.length);
    this.previousViewportTop = prevViewportTop;
    this.previousHeight = height;
    return;
}
```

### 2.4 渲染性能优化

| 优化技术 | 说明 |
|---------|------|
| 最小渲染区间 | 只渲染变化的行,不是整个屏幕 |
| 内容缓存 | 每个组件缓存上一次渲染的结果 |
| 渲染节流 | 最小渲染间隔 16ms (约 60fps) |
| 宽字符处理 | 正确处理 CJK 宽字符和 ANSI 转义序列 |
| 最大行数追踪 | 追踪历史最大渲染行数,避免频繁重排 |

## 三、组件系统

### 3.1 Component 接口

所有组件必须实现 `Component` 接口:

```typescript
// packages/tui/src/tui.ts:17
export interface Component {
    render(width: number): string[];
    handleInput?(data: string): void;
    wantsKeyRelease?: boolean;
    invalidate(): void;
}
```

### 3.2 内置组件列表

| 组件 | 文件 | 用途 |
|------|------|------|
| **Text** | `components/text.ts` | 多行文本显示,支持自动换行 |
| **Box** | `components/box.ts` | 布局容器,包含子组件 |
| **Input** | `components/input.ts` | 单行文本输入 |
| **Editor** | `components/editor.ts` | 多行代码编辑器,支持语法高亮和高级编辑 |
| **Markdown** | `components/markdown.ts` | Markdown 渲染 |
| **SelectList** | `components/select-list.ts` | 可滚动选择列表 |
| **SettingsList** | `components/settings-list.ts` | 设置项列表 |
| **TruncatedText** | `components/truncated-text.ts` | 截断文本,显示省略号 |
| **Spacer** | `components/spacer.ts` | 空白占位组件 |
| **Image** | `components/image.ts` | 终端图像渲染 (iTerm2/Kitty 协议) |
| **Loader** | `components/loader.ts` | 加载动画指示器 |
| **CancellableLoader** | `components/cancellable-loader.ts` | 可取消的加载指示器 |

### 3.3 Text 组件示例

```typescript
import { Text } from "@anthropic-ai/pi-tui";

const text = new Text("Hello, World!", 1, 1);
const lines = text.render(80);
console.log(lines);
```

### 3.4 Focusable 接口

支持焦点和硬件光标的组件实现 `Focusable` 接口:

```typescript
// packages/tui/src/tui.ts:52
export interface Focusable {
    focused: boolean;
}
```

当组件获得焦点时,在渲染输出中插入 `CURSOR_MARKER` (`\x1b_pi:c\x07`),TUI 会找到并定位硬件光标。

```typescript
// packages/tui/src/tui.ts:68
export const CURSOR_MARKER = "\x1b_pi:c\x07";
```

## 四、Overlay 系统

Overlay 是渲染在基础内容之上的浮动组件,用于模态对话框、选择列表等场景。

### 4.1 OverlayOptions

```typescript
export interface OverlayOptions {
    width?: SizeValue;           // 宽度或百分比
    minWidth?: number;
    maxHeight?: SizeValue;       // 最大高度或百分比
    anchor?: OverlayAnchor; // 锚点位置
    offsetX?: number;
    offsetY?: number;
    row?: SizeValue;       // 行位置
    col?: SizeValue;      // 列位置
    margin?: OverlayMargin | number;
    visible?: (termWidth: number, termHeight: number) => boolean;
    nonCapturing?: boolean;
}
```

### 4.2 使用方式

```typescript
const tui = new TUI(terminal);
const overlay = tui.showOverlay(component, {
    width: "50%",
    anchor: "center",
    margin: 5,
});

// 隐藏 overlay
overlay.hide();
```

### 4.3 Overlay 渲染

Overlay 通过 `compositeOverlays()` 方法合成到基础内容上,支持多层叠加和透明度处理。

## 五、主题系统

pi-tui 使用**函数式主题**,每个样式元素都是接受字符串、返回字符串的函数。

### 5.1 主题接口示例

```typescript
// packages/tui/test/test-themes.ts
export const defaultSelectListTheme: SelectListTheme = {
    selectedPrefix: (text: string) => chalk.blue(text),
    selectedText: (text: string) => chalk.bold(text),
    description: (text: string) => chalk.dim(text),
    scrollInfo: (text: string) => chalk.dim(text),
    noMatch: (text: string) => chalk.dim(text),
};
```

### 5.2 主题应用

主题在组件渲染时应用:

```typescript
// 组件内部渲染逻辑
const rendered = this.theme.selectedText(this.item.label);
return this.theme.selectedPrefix("▸ ") + rendered;
```

### 5.3 可定制主题

- **SelectListTheme**: 选择列表样式
- **MarkdownTheme**: Markdown 渲染样式
- **EditorTheme**: 编辑器样式 (包含 borderColor 和 SelectListTheme)
- **ImageTheme**: 图像渲染样式

## 六、键盘输入处理

### 6.1 keys.ts

支持两种键盘协议:

- **Legacy 序列**: 传统终端转义序列
- **Kitty 键盘协议**: 增强的键盘协议,支持更多修饰键组合

```typescript
// 支持的键标识符
type KeyId = "ctrl+c" | "escape" | "up" | "down" | "ctrl+shift+a" | ...;
```

### 6.2 Key 辅助对象

```typescript
import { matchesKey, parseKey, Key } from "@anthropic-ai/pi-tui";

if (matchesKey(data, "ctrl+c")) { /* 处理 Ctrl+C */ }
const keyId = parseKey(data); // 解析输入为键标识符
```

### 6.3 自动补全

```typescript
// packages/tui/src/autocomplete.ts
export interface AutocompleteProvider {
    getAutocompletions(input: string): Promise<AutocompleteSuggestions>;
}
```

## 七、终端能力检测

### 7.1 支持的协议

| 协议 | 能力 |
|------|------|
| iTerm2 | 图像 (Inline Images) |
| Kitty | 图像, 键盘协议 |
| Windows Terminal | 有限支持 |
| GNOME Terminal | 有限支持 |

### 7.2 能力检测

```typescript
// packages/tui/src/terminal-image.ts
import { getCapabilities, detectCapabilities } from "@anthropic-ai/pi-tui";

const caps = getCapabilities();
// caps.images: 是否支持终端图像
// caps.kittyProtocol: 是否支持 Kitty 协议
```

### 7.3 图像渲染

```typescript
import { renderImage, encodeKitty, encodeITerm2 } from "@anthropic-ai/pi-tui";

const encoded = encodeKitty(imageBuffer);
// 使用终端协议编码图像
```

## 八、实用工具

### 8.1 文本处理

```typescript
import { truncateToWidth, visibleWidth, wrapTextWithAnsi } from "@anthropic-ai/pi-tui";

const width = visibleWidth("你好"); // 4 (CJK 字符视为双宽)
const truncated = truncateToWidth("很长很长的文本", 10); // 截断到指定宽度
const wrapped = wrapTextWithAnsi("文本内容", 80); // ANSI 感知的换行
```

### 8.2 键绑定

```typescript
import { getKeybindings, setKeybindings, KeybindingsManager } from "@anthropic-ai/pi-tui";

const bindings = getKeybindings("editor");
setKeybindings("custom", customBindings);
```

### 8.3 模糊匹配

```typescript
import { fuzzyFilter, fuzzyMatch } from "@anthropic-ai/pi-tui";

const matches = fuzzyFilter("pyt", ["python", "python3", "pyramid"]);
// returns ["python", "pyramid"]
```

## 九、参考实现

- **TUI 核心**: `packages/tui/src/tui.ts`
- **组件**: `packages/tui/src/components/*.ts`
- **键盘处理**: `packages/tui/src/keys.ts`
- **终端图像**: `packages/tui/src/terminal-image.ts`
- **组件测试**: `packages/tui/test/*.test.ts`

## 十、总结

pi-tui 是一个设计精良的终端 UI 库,其核心特点:

1. **高效差分渲染**: 三层策略确保最小化终端输出
2. **组件化设计**: 统一的组件接口,易于扩展
3. **主题函数式**: 灵活的主题定制机制
4. **协议支持**: 多终端协议的自动检测和适配
5. **IME 兼容**: 正确的硬件光标位置支持输入法

该系统的设计对于构建高效的终端应用具有良好的参考价值。