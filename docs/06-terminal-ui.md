# 06 - 终端 UI 架构

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 使用**自定义 Ink 框架**（基于 React + Yoga 布局引擎）实现终端 UI。通过 React Reconciler 将虚拟 DOM 渲染为 ANSI 终端输出，支持双缓冲差分渲染、鼠标事件、全屏模式和 Vim 快捷键。

## Ink 核心架构

**文件**: `ink/ink.tsx` (~1722行)

### Ink 类

```typescript
class Ink {
  // 终端
  readonly terminal: Terminal       // {stdout, stderr}
  private stdin: NodeJS.ReadStream

  // 双缓冲渲染
  private frontFrame: Frame         // 当前显示缓冲
  private backFrame: Frame          // 前一帧
  private stylePool: StylePool      // 样式复用
  private charPool: CharPool        // 字符复用
  private hyperlinkPool: HyperlinkPool

  // React 集成
  private container: FiberRoot      // React-Reconciler (ConcurrentRoot)
  private rootNode: dom.DOMElement  // 虚拟 DOM 根
  private renderer: Renderer        // Yoga 布局 + 渲染

  // 事件
  readonly focusManager: FocusManager
  private hoveredNodes: Set<dom.DOMElement>
  private selection: SelectionState

  // 全屏
  private altScreenActive: boolean
  private altScreenMouseTracking: boolean
}
```

### 渲染管线

```
React 状态变更
  ↓
onRender() (节流 16ms @ 60fps)
  ├─ Yoga 布局计算 (yogaNode.calculateLayout)
  ├─ React 树渲染 → 虚拟 DOM → 屏幕缓冲
  ├─ 叠加层
  │   ├─ 文本选择反转
  │   ├─ 搜索高亮
  │   └─ 位置高亮
  ├─ 损伤检测 (仅重绘变化部分)
  ├─ 差分渲染 (前帧 vs 当前帧 → 最小 CSI 序列)
  ├─ 原生光标定位 (IME/屏幕读者支持)
  └─ 写入终端
```

### 性能优化

| 策略 | 实现 |
|------|------|
| 双缓冲 | frontFrame/backFrame 交换 |
| 差分渲染 | LogUpdate 比较帧差异，仅输出变化的 CSI 序列 |
| 池化 | StylePool/CharPool/HyperlinkPool 复用对象 ID |
| 节流 | `throttle(render, 16ms)` (60fps) |
| Yoga 缓存 | 布局缓存命中率追踪 |

### 全屏模式

```typescript
enterAlternateScreen()  // 暂停 Ink，进入 alt 屏幕（用于 vim/nano/less）
exitAlternateScreen()   // 恢复 Ink，清屏，完整重绘
```

## 组件体系

### 核心 Ink 组件

| 组件 | 用途 |
|------|------|
| `Box` | Flexbox 容器 (margin/padding/gap/flexDirection) |
| `Text` | 文本渲染 (color/bold/italic/underline) |
| `ScrollBox` | 可滚动容器 (scrollTo/scrollBy/isSticky) |
| `AlternateScreen` | Alt 屏幕容器（自动管理 DEC 1049h/l） |
| `Button` | 按钮组件 |
| `Link` | 超链接 |
| `Spacer` | 弹性空间 |

### Ink Hooks

```typescript
useInput(handler)              // 键盘输入处理
useApp()                       // 应用实例访问
useStdin()                     // 标准输入
useSelection()                 // 文本选择
useAnimationFrame(callback)    // 动画帧
useTerminalViewport()          // 终端尺寸
useTerminalFocus()             // 终端焦点
useTerminalTitle(title)        // 终端标题
```

## PromptInput 组件

**文件**: `components/PromptInput/PromptInput.tsx` (1400+ 行)

### 布局结构

```
PromptInput (fullscreen flex column)
├─ 消息窗口 (ScrollBox, flex-grow)
│   └─ <Messages> (对话历史)
├─ 边框分隔
├─ 输入区 (flex-shrink, max-height=50%)
│   ├─ TextInput 或 VimTextInput
│   ├─ 模式指示器 (plan/fast/normal)
│   └─ 页脚
│       ├─ 左: 快捷键、权限、成本
│       ├─ 中: 自动完成建议
│       └─ 右: 任务/diff/loop 指示器
└─ 模态对话 (浮动)
    ├─ 全局搜索 (ctrl+shift+f)
    ├─ 快速打开 (ctrl+shift+p)
    ├─ 模型选择器 (meta+m)
    └─ 历史搜索 (ctrl+r)
```

## StructuredDiff 渲染

**文件**: `components/StructuredDiff.tsx`

使用 Rust ColorDiff NAPI 进行语法高亮的 diff 渲染：

```typescript
// WeakMap 三层缓存：Patch → theme×width×dim×gutterWidth → CachedRender
const lines = new ColorDiff(patch, firstLine, filePath, fileContent)
  .render(theme, width, dim)
```

每个 patch 最多缓存 4 个变体（适应终端调整大小）。

## 快捷键系统

**文件**: `keybindings/defaultBindings.ts`

### 上下文分层

```typescript
DEFAULT_BINDINGS = [
  { context: 'Global', bindings: {
    'ctrl+c': 'app:interrupt',
    'ctrl+d': 'app:exit',
    'ctrl+l': 'app:redraw',
    'ctrl+r': 'history:search',
    'ctrl+shift+f': 'app:globalSearch',
  }},
  { context: 'Chat', bindings: {
    'escape': 'chat:cancel',
    'enter': 'chat:submit',
    'shift+tab': 'chat:cycleMode',
    'meta+p': 'chat:modelPicker',
    'ctrl+x ctrl+e': 'chat:externalEditor',  // 弦绑定
  }},
  { context: 'Autocomplete', bindings: {
    'tab': 'autocomplete:accept',
    'escape': 'autocomplete:dismiss',
  }},
  { context: 'Scroll', bindings: {
    'pageup': 'scroll:pageUp',
    'wheelup': 'scroll:lineUp',
    'ctrl+shift+c': 'selection:copy',
  }},
  // 15+ 上下文...
]
```

### Hooks

```typescript
useKeybinding('chat:submit', handler, { context: 'Chat', isActive })
useKeybindings({ 'chat:submit': handler1, 'chat:cancel': handler2 }, options)
```

弦绑定支持：`ctrl+x ctrl+k` → 两步按键序列，自动管理 pending 状态。

## Vim 模式

**文件**: `vim/types.ts`, `vim/motions.ts`

### 完整状态机

```
VimState = INSERT | NORMAL

NORMAL:
  idle → [d/c/y] → operator (等待运动)
       → [1-9]   → count (累积数字)
       → [fFtT]  → find (查找字符)
       → [g]     → g 命令族
       → [r]     → replace
       → [><]    → indent

INSERT: 跟踪 insertedText (用于点重复 `.`)
```

### 运动库

- 字符/行: `h l j k`
- 词边界: `w b e W B E`
- 行位置: `0 ^ $`
- 文本对象: `iw aw i" a( ib` 等
- 查找: `f F t T ; ,`
- 全局: `gg G`

### 持久状态

```typescript
type PersistentState = {
  lastChange: RecordedChange | null  // 点重复 (.)
  lastFind: { type, char } | null    // ; / , 重复
  register: string                   // 复制寄存器
  registerIsLinewise: boolean
}
```

## 屏幕系统

| 屏幕 | 用途 |
|------|------|
| `REPL.tsx` | 主会话 UI（消息 + 输入 + 页脚） |
| `Doctor.tsx` | 诊断屏幕 |
| `ResumeConversation.tsx` | 恢复会话选择 |

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `ink/ink.tsx` (1722行) | Ink 核心、React Reconciler 集成 |
| `ink.ts` | 公共 API + ThemeProvider 包装 |
| `ink/components/` (18文件) | Box/Text/ScrollBox 等 |
| `components/PromptInput/` | 主输入组件 |
| `components/StructuredDiff.tsx` | Diff 语法高亮渲染 |
| `components/permissions/` (20+文件) | 权限对话框 |
| `keybindings/` (13文件) | 快捷键绑定 + 解析 |
| `vim/` (5文件) | Vim 状态机 + 运动 |
| `screens/` | 各屏幕实现 |
