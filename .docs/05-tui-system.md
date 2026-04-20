# pi-tui 终端 UI 系统

## 概述

pi-tui 是终端 UI 库，提供差分渲染的交互式界面。

**核心目标**:
- 差分渲染 (只重绘变化部分)
- 组件系统
- 键盘处理
- 焦点管理
- 光标定位

## 文件结构

```
packages/tui/
├── src/
│   ├── index.ts            # 入口
│   ├── tui.ts          # 主类 (~1243 行)
│   ├── terminal.ts     # 终端接口
│   ├── component.ts   # 组件基类
│   ├── keys.ts      # 按键处理
│   ├── keybindings.ts
│   ├── undo-stack.ts
│   ├── kill-ring.ts
│   ├── stdin-buffer.ts
│   ├── fuzzy.ts
│   ├── autocomplete.ts
│   ├── utils.ts
│   ├── editor-component.ts
│   └── components/
│       ├── box.ts
│       ├── text.ts
│       ├── input.ts
│       ├── select-list.ts
│       ├── editor.ts
│       ├── loader.ts
│       ├── image.ts
│       ├── markdown.ts
│       ├── settings-list.ts
│       ├── truncated-text.ts
│       └── ...
```

## 核心接口

### Component 接口

```typescript
export interface Component {
  // 渲染到行数组
  render(width: number): string[];
  
  // 处理键盘输入 (有焦点时)
  handleInput?(data: string): void;
  
  // 是否需要键盘释放事件 (Kitty 协议)
  wantsKeyRelease?: boolean;
  
  // 失效缓存
  invalidate(): void;
}
```

### Focusable 接口

```typescript
export interface Focusable {
  // TUI 设置焦点时
  focused: boolean;
}
```

### 行渲染

```typescript
// 组件渲染返回行数组
render(width: number): string[];

// 示例
const component = new Text("Hello World");
const lines = component.render(80);
// ["Hello World"]
```

### 光标标记

```typescript
// 光标位置标记 - 零宽转义序列
export const CURSOR_MARKER = "\x1b_pi:c\x07";

// 组件在 Focusable 实现中
render(width: number): string[] {
  if (this.focused) {
    return [CURSOR_MARKER + "text here"];
  }
  return ["text here"];
}
```

## TUI 类

### 主类定义

```typescript
export class Container implements Component {
  children: Component[] = [];
  
  addChild(component: Component): void { ... }
  removeChild(component: Component): void { ... }
  clear(): void { ... }
  invalidate(): void { ... }
  render(width: number): string[] { ... }
}

export class TUI extends Container {
  private focusedComponent: Component | null = null;
  private children: Component[] = [];
  
  // 生命周期
  start(): void { ... }    // 开始事件循环
  stop(): void { ... }     // 停止
  
  // 焦点管理
  setFocus(component: Component | null): void { ... }
  getFocused(): Component | null { ... }
  
  // 渲染
  render(): string[] { ... }
  
  // 添加子组件
  addChild(component: Component): void { ... }
  
  // 清除
  clear(): void { ... }
}
```

### 生命周期

```typescript
const tui = new TUI(terminal);

// 添加组件
tui.addChild(header);
tui.addChild(messages);
tui.addChild(editor);
tui.addChild(footer);

// 开始
tui.start();

// 停止
tui.stop();
```

### 焦点管理

```typescript
// 设置焦点
tui.setFocus(editorComponent);

// 移除焦点
tui.setFocus(null);

// 检查焦点
const focused = tui.getFocused();
```

## 容器实现

```typescript
class Container implements Component {
  children: Component[] = [];
  
  addChild(component: Component): void {
    this.children.push(component);
  }
  
  removeChild(component: Component): void {
    const index = this.children.indexOf(component);
    if (index !== -1) {
      this.children.splice(index, 1);
    }
  }
  
  clear(): void {
    this.children = [];
  }
  
  invalidate(): void {
    for (const child of this.children) {
      child.invalidate?.();
    }
  }
  
  render(width: number): string[] {
    const lines: string[] = [];
    for (const child of this.children) {
      lines.push(...child.render(width));
    }
    return lines;
  }
}
```

## 预置组件

### 文本组件

```typescript
class Text implements Component {
  constructor(private text: string) {}
  
  render(width: number): string[] {
    return [this.text];
  }
}
```

### Box 组件

```typescript
interface BoxOptions {
  border?: boolean;
  borderStyle?: "single" | "double" | "rounded";
  padding?: number;
  margin?: number;
}

class Box implements Component {
  private content: Component;
  
  constructor(content: Component, options: BoxOptions = {}) { ... }
  
  render(width: number): string[] { ... }
}
```

### Spacer 组件

```typescript
class Spacer implements Component {
  constructor(private minHeight: number = 1, private maxHeight?: number) { ... }
  
  render(width: number): string[] { ... }
}
```

### Loader 组件

```typescript
class Loader implements Component {
  constructor(
    private message: string = "Loading",
    private frames: string[] = ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"],
  ) {}
  
  render(width: number): string[] { ... }
  
  // 动画状态
  private frame = 0;
  tick(): void { ... }
}
```

### Image 组件 (终端图像)

```typescript
class Image implements Component {
  constructor(
    private data: string,  // Base64
    private mimeType: string,
    options?: ImageOptions,
  ) {}
  
  render(width: number): string[] { ... }
}
```

### Input 组件

```typescript
class Input implements Component, Focusable {
  focused: boolean = false;
  private value: string = "";
  private cursor: number = 0;
  
  constructor(options: InputOptions = {}) { ... }
  
  handleInput(data: string): void {
    switch (data) {
      case "backspace":
        // 删除字符
        break;
      case "left":
        // 光标左移
        break;
      case "right":
        // 光标右移
        break;
      case "enter":
        // 提交
        break;
      default:
        // 插入字符
        this.value = this.value.slice(0, this.cursor) + data + this.value.slice(this.cursor);
        this.cursor += data.length;
        break;
    }
  }
  
  render(width: number): string[] { ... }
  
  invalidate(): void { ... }
}
```

### SelectList 组件

```typescript
class SelectList implements Component, Focusable {
  focused: boolean = false;
  private items: string[] = [];
  private selected: number = 0;
  private scroll: number = 0;
  
  constructor(items: string[], options: SelectListOptions = {}) { ... }
  
  handleInput(data: string): void {
    switch (data) {
      case "up":
        this.selected = Math.max(0, this.selected - 1);
        break;
      case "down":
        this.selected = Math.min(this.items.length - 1, this.selected + 1);
        break;
      case "enter":
        this.selectedIndex = this.selected;
        break;
      case " Home":
        this.selected = 0;
        break;
      case " End":
        this.selected = this.items.length - 1;
        break;
    }
  }
  
  render(width: number): string[] { ... }
}
```

### EditorComponent

```typescript
class EditorComponent implements Component, Focusable {
  focused: boolean = false;
  private content: string[] = [];
  private scroll: number = 0;
  private cursor: { row: number; col: number } = { row: 0, col: 0 };
  
  handleInput(data: string): void {
    // 完整编辑器处理
  }
  
  render(width: number): string[] { ... }
}
```

## 差分渲染

### 渲染优化

```typescript
class OptimizedComponent implements Component {
  private cache: string[] = [];
  
  render(width: number): string[] {
    const newLines = this.compute(width);
    
    // 差分: 只输出变化部分
    if (this.cachedLines.length !== newLines.length) {
      this.cache = newLines;
      return newLines;
    }
    
    // 逐行比较
    const diff: string[] = [];
    for (let i = 0; i < newLines.length; i++) {
      if (this.cache[i] !== newLines[i]) {
        diff.push(newLines[i]);
      }
    }
    
    this.cache = newLines;
    return diff;
  }
  
  invalidate(): void {
    this.cache = [];
  }
}
```

### 渲染输出

```typescript
// 差分更新
const diff = computeRenderDiff(currentContent, newContent);

// ANSI 转义序列
const clear = "\x1b[J";
const cursorHome = "\x1b[H";
const clearLine = "\x1b[2K";

// 完整刷新
terminal.write(cursorHome + clear + lines.join("\r\n"));

// 差分刷新
terminal.write(diff.map((line, i) => {
  return `\x1b[${i + 1};1H${line}`;
}).join("\r\n"));
```

## 终端接口

### Terminal 接口

```typescript
interface Terminal {
  // 基础
  write(data: string): void;
  writeRaw(data: string): void;
  
  // 大小
  get width(): number;
  get height(): number;
  onResize(callback: (width: number, height: number) => void): void;
  
  // 输入
  onData(callback: (data: string) => void): void;
  onKey(callback: (key: Key) => void): void;
  
  // 特性
  isUnsupported?: boolean;
}
```

### ProcessTerminal

```typescript
class ProcessTerminal implements Terminal {
  constructor() {
    // 使用 process.stdin/process.stdout
  }
  
  write(data: string): void {
    process.stdout.write(data);
  }
  
  get width(): number {
    return process.stdout.columns;
  }
  
  get height(): number {
    return process.stdout.rows;
  }
}
```

## 键盘处理

### keys.ts

```typescript
// 按键类型
interface Key {
  name: string;        // "Enter", "Backspace", "Space"
  codepoint?: number;
  ctrl?: boolean;
  alt?: boolean;
  shift?: boolean;
}

// 特殊按键
const specialKeys: Record<string, Key> = {
  Enter: { name: "Enter" },
  Backspace: { name: "Backspace" },
  Tab: { name: "Tab" },
  Escape: { name: "Escape" },
  ArrowUp: { name: "ArrowUp" },
  ArrowDown: { name: "ArrowDown" },
  ArrowLeft: { name: "ArrowLeft" },
  ArrowRight: { name: "ArrowRight" },
  Home: { name: "Home" },
  End: { name: "End" },
  PageUp: { name: "PageUp" },
  PageDown: { name: "PageDown" },
  Delete: { name: "Delete" },
  Insert: { name: "Insert" },
};

// 修饰键处理
function isKeyRelease(data: string): boolean { ... }
function matchesKey(data: string, pattern: string | Key): boolean { ... }
```

### 按键序列解析

```typescript
// CSI 序列
"\x1b[A"  // ArrowUp
"\x1b[B"  // ArrowDown
"\x1b[C"  // ArrowRight
"\x1b[D"  // ArrowLeft
"\x1b[H"  // Home
"\x1b[F"  // End
"\x1b[5~" // PageUp
"\x1b[6~" // PageDown

// SS3 序列 (xterm)
"\x1bOA"  // ArrowUp
"\x1bOB"  // ArrowDown
"\x1bOC"  // ArrowRight
"\x1bOD"  // ArrowLeft

// Kitty 键盘协议
"\x1b[<i;1u"  // 修饰键+ Key
```

### 处理流程

```typescript
terminal.onData((data) => {
  // 1. 解析按键
  const key = parseKey(data);
  
  // 2. 发送事件
  if (key) {
    terminal.onKey(key);
  } else {
    terminal.onData(data);
  }
});

function parseKey(data: string): Key | null {
  // 解析 CSI/SS3 序列
  if (data.startsWith("\x1b[")) {
    // CSI 序列
    const match = data.match(/^\x1b\[(\d+);(\d+)u$/);
    if (match) {
      return {
        name: getKeyName(parseInt(match[1])),
        ctrl: (parseInt(match[2]) & 1,
        shift: (parseInt(match[2])) & 2,
        alt: (parseInt(match[2])) & 4,
      };
    }
  }
  return null;
}
```

## 覆盖层

### OverlayHandle

```typescript
export interface OverlayHandle {
  hide(): void;
  setHidden(hidden: boolean): void;
  isHidden(): boolean;
  focus(): void;
  unfocus(): void;
  isFocused(): boolean;
}
```

### 显示覆盖层

```typescript
const handle = tui.showOverlay(component, {
  anchor: "center",
  width: "50%",
  maxHeight: "80%",
});
```

## 工具函数

### utils.ts

```typescript
// 字符串可见宽度
function visibleWidth(str: string): number {
  // 东亚文字宽度
  return str.length;
}

// 列切片
function sliceByColumn(lines: string[], start: number, end: number): string[];

// 提取片段
function extractSegments(lines: string[], pattern: string): Segment[];

// 截断
function truncateToWidth(str: string, width: number): string;
```

## 样式

### 样式支持

```typescript
// ANSI 码
const styles = {
  reset: "\x1b[0m",
  bold: "\x1b[1m",
  dim: "\x1b[2m",
  italic: "\x1b[3m",
  underline: "\x1b[4m",
  
  // 前景色
  black: "\x1b[30m",
  red: "\x1b[31m",
  green: "\x1b[32m",
  yellow: "\x1b[33m",
  blue: "\x1b[34m",
  magenta: "\x1b[35m",
  cyan: "\x1b[36m",
  white: "\x1b[37m",
  
  // 背景色
  bgBlack: "\x1b[40m",
  // ...
};

// 使用
const styledText = `${styles.bold}Hello${styles.reset} World`;
```

## 主题支持

### 主题配置

```typescript
interface Theme {
  // 颜色
  colors: {
    background: string;
    foreground: string;
    selection: string;
    // ...
  };
  
  // 字体
  fonts: {
    normal: string;
    bold: string;
    italic: string;
  };
  
  // 样式映射
  styles: Record<string, string>;
}
```

### 主题使用

```typescript
class ThemedComponent implements Component {
  constructor(private theme: Theme) {}
  
  render(width: number): string[] {
    return [
      this.theme.colors.background + "..." + this.theme.colors.foreground
    ];
  }
}
```

## 自动补全

### Autocomplete

```typescript
interface AutocompleteItem {
  label: string;
  description?: string;
  action?: () => void;
}

interface AutocompleteOptions {
  items: AutocompleteItem[];
  fuzzy?: boolean;
}

class Autocomplete implements Component {
  constructor(options: AutocompleteOptions) {}
  
  filter(query: string): AutocompleteItem[] { ... }
  render(width: number): string[] { ... }
}
```

## 总结

pi-tui 核心设计：

1. **Component 接口** - 渲染、行处理、焦点管理
2. **Container/TUI** - 容器和焦点管理
3. **预置组件** - Text, Box, Input, SelectList, Editor
4. **差分渲染** - 优化终端输出
5. **键盘处理** - 按键解析 (CSI/SS3/Kitty)

关键特性：
- 组件化设计
- 焦点管理
- 键盘输入
- 差分渲染
- 主题支持