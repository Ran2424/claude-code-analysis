# Claude Code — Ink 终端渲染系统

## 概览

ink 目录包含一个完整的、自定义的终端 UI 框架，构建在 React 之上。它是开源 Ink 库经过大量修改和扩展后的分支，针对 Claude Code 的需求做了调整：全屏备用屏幕渲染、硬件加速滚动、文本选择、搜索高亮、鼠标跟踪、双向文本，以及细粒度性能埋点。

这个系统可以概括为一条流水线：

```
React tree
    → React Reconciler (reconciler.ts)
    → Virtual DOM (dom.ts)
    → Yoga layout engine (layout/)
    → Output buffer (output.ts)
    → Screen cell buffer (screen.ts)
    → Diff engine (log-update.ts)
    → Patch optimizer (optimizer.ts)
    → Terminal write (terminal.ts / termio/)
```

对外的顶层入口是 `/x/Bigger-Projects/Claude-Code/src/ink.ts`，它会用必需的 `ThemeProvider` 包裹内部的 `root.ts` `render` 和 `createRoot` API。

---

## 按文件参考

### `/x/Bigger-Projects/Claude-Code/src/ink.ts` — 公共 API 模块

**用途：** 作为包级门面，重新导出 ink 子系统的全部公共 API。它会在每次 `render()` 和 `createRoot()` 调用外层包上 `ThemeProvider`，这样 `ThemedBox`/`ThemedText` 就能在任何调用点直接工作，而不用调用方手动挂载 provider。

**导出：**
- `render(node, options?)` — 异步，挂载一个被 `ThemeProvider` 包裹的 React 树；返回 `Instance`
- `createRoot(options?)` — 异步；返回一个 `Root`，其 `.render()` 方法同样会包上 `ThemeProvider`
- `RenderOptions`, `Instance`, `Root` — 来自 `root.ts` 的类型重新导出
- `color` — 来自设计系统的颜色模块
- `Box`, `BoxProps` — 带主题的 box（设计系统 `ThemedBox`）
- `Text`, `TextProps` — 带主题的 text（设计系统 `ThemedText`）
- `ThemeProvider`, `usePreviewTheme`, `useTheme`, `useThemeSetting`
- `Ansi` — ANSI 字符串渲染组件
- `BaseBox`, `BaseBoxProps` — 不带主题的原始 ink Box
- `BaseText`, `BaseTextProps` — 不带主题的原始 ink Text
- `Button`, `ButtonProps`, `ButtonState`
- `Link`, `LinkProps`
- `Newline`, `NewlineProps`
- `NoSelect`
- `RawAnsi`
- `Spacer`
- `DOMElement` — 虚拟 DOM 元素类型
- `ClickEvent`, `EventEmitter`, `Event`, `Key`, `InputEvent`
- `TerminalFocusEvent`, `TerminalFocusEventType`
- `FocusManager`
- `FlickerReason`
- `useAnimationFrame`, `useApp`, `useInput`, `useAnimationTimer`, `useInterval`
- `useSelection`, `useStdin`, `useTabStatus`, `useTerminalFocus`
- `useTerminalTitle`, `useTerminalViewport`
- `measureElement`
- `supportsTabStatus`
- `wrapText`

---

### `/x/Bigger-Projects/Claude-Code/src/ink/constants.ts`

**用途：** 共享的时间常量。

**导出：**
- `FRAME_INTERVAL_MS = 16` — 目标帧间隔（约 60 fps），供节流后的 `scheduleRender` 使用。

---

### `/x/Bigger-Projects/Claude-Code/src/ink/ink.tsx` — 核心 Ink 类

**用途：** 中央调度器。`class Ink` 持有 React fiber root、yoga 布局树、双缓冲屏幕、焦点管理器、stdin/stdout 事件处理器、选择状态和主渲染循环。

**`Ink` 类**

构造函数接收 `Options`：
```typescript
type Options = {
  stdout: NodeJS.WriteStream
  stdin: NodeJS.ReadStream
  stderr: NodeJS.WriteStream
  exitOnCtrlC: boolean
  patchConsole: boolean
  waitUntilExit?: () => Promise<void>
  onFrame?: (event: FrameEvent) => void
}
```

关键私有状态：
| 字段 | 类型 | 说明 |
|---|---|---|
| `log` | `LogUpdate` | 写 ANSI 到 stdout 的 diff 引擎 |
| `terminal` | `Terminal` | `{ stdout, stderr }` 写流 |
| `scheduleRender` | throttled fn | 节流到 16 ms，前后沿都触发，并通过 `queueMicrotask` 延后 |
| `container` | `FiberRoot` | React-reconciler 的 fiber root（ConcurrentRoot 模式） |
| `rootNode` | `DOMElement` | `ink-root` DOM 节点 |
| `focusManager` | `FocusManager` | DOM 风格的焦点状态机 |
| `renderer` | `Renderer` | `createRenderer()` 闭包 |
| `stylePool` | `StylePool` | 会话级 ANSI 样式驻留池 |
| `charPool` | `CharPool` | 会话级字符驻留池 |
| `hyperlinkPool` | `HyperlinkPool` | 会话级超链接 URL 驻留池 |
| `frontFrame` / `backFrame` | `Frame` | 双缓冲屏幕帧 |
| `selection` | `SelectionState` | alt-screen 模式下的文本选择 |
| `searchHighlightQuery` | `string` | 当前 /search 词条 |
| `searchPositions` | object or null | 当前匹配高亮的预扫描位置 |
| `altScreenActive` | `boolean` | 由 `<AlternateScreen>` 设定 |
| `prevFrameContaminated` | `boolean` | 强制下一帧整屏重绘 |

**Render loop (`onRender`):**
1. 运行 `createRenderer()`，遍历 DOM、执行 yoga、填充后缓冲区屏幕
2. 应用选择覆盖层（把 `selection` 中的单元格反相）
3. 应用搜索高亮（把匹配 `searchHighlightQuery` 的单元格反相）
4. 应用定位高亮（通过 `searchPositions` 把当前匹配高亮成黄字/粗体）
5. 通过 `log.render()` 比较后帧和前帧，得到 `Patch[]`
6. 调用 `optimize(patches)` 做合并和去重
7. 调用 `writeDiffToTerminal()` 序列化补丁并把 ANSI 写到 stdout
8. 如果接好了 `onFrame`，就发出该事件
9. 交换前后帧

**alt-screen 处理：**
- `setAltScreenActive(active, mouseTracking)` — 由 `<AlternateScreen>` 在插入阶段调用；开启 BSU/ESU 同步输出、DECSTBM 硬件滚动提示和感知选择的重绘
- `resetFramesForAltScreen()` — 把两帧都换成空白屏幕，并把 `prevFrameContaminated = true`
- `reenterAltScreen()` — 在 SIGCONT 后重新确认 alt-screen 状态

**Resize 处理（`handleResize`）：**
- 同步执行（不做 debounce），确保 `terminalColumns`/`terminalRows` 和 yoga 保持一致
- 对 alt-screen：重置帧缓冲，并设置 `needsEraseBeforePaint = true`，这样擦除会在下一次 BSU/ESU 块里原子完成

**Console 补丁：**
- `patchConsole()` — 拦截 `console.log/warn/error`，让它们写到单独的文件描述符（不是 stdout），避免输出混在一起
- `patchStderr()` — stderr 同理

**关键公共方法：**
- `render(node)` — 调用 `reconciler.updateContainer()`
- `unmount()` — 优雅清理：恢复 console、关闭鼠标跟踪、退出 alt-screen、写出最终帧、释放 yoga 节点
- `waitUntilExit()` — 返回一个在 `unmount()` 时解决的 promise
- `clearTextSelection()` — 清除选择状态并强制重绘
- `setSearchHighlight(query)` — 设置实时搜索词
- `setSearchPositions(positions, rowOffset, currentIdx)` — 为搜索导航设置定位高亮

---

### `/x/Bigger-Projects/Claude-Code/src/ink/root.ts` — 公共入口点

**用途：** 把 `Ink` 包到对外的 `render()` 和 `createRoot()` API 里；同时管理 `instances` 映射，让对同一个 stdout 流重复调用 `render()` 时复用同一个 `Ink` 实例。

**导出：**
```typescript
type RenderOptions = {
  stdout?: NodeJS.WriteStream
  stdin?: NodeJS.ReadStream
  stderr?: NodeJS.WriteStream
  exitOnCtrlC?: boolean
  patchConsole?: boolean
  onFrame?: (event: FrameEvent) => void
}

type Instance = {
  rerender: Ink['render']
  unmount: Ink['unmount']
  waitUntilExit: Ink['waitUntilExit']
  cleanup: () => void
}

type Root = {
  render: (node: ReactNode) => void
  unmount: () => void
  waitUntilExit: () => Promise<void>
}

export const renderSync(node, options?): Instance   // synchronous mount
export default async function render(node, options?): Promise<Instance>
export async function createRoot(options?): Promise<Root>
```

`renderSync` 只在内部使用；对外导出的 `render` 和 `createRoot` 是通过 `ink.ts` 暴露的异步版本。

---

### `/x/Bigger-Projects/Claude-Code/src/ink/instances.ts`

**用途：** 以 `NodeJS.WriteStream` 为键的模块级单例映射。确保每个 stdout 流只有一个 `Ink` 实例。

```typescript
const instances = new Map<NodeJS.WriteStream, Ink>()
export default instances
```

---

## DOM 层

### `/x/Bigger-Projects/Claude-Code/src/ink/dom.ts` — 虚拟 DOM

**用途：** 定义虚拟 DOM 节点类型和全部变更操作。它模仿了一个适配终端的最小浏览器 DOM API。每次变更都会把受影响节点及其所有祖先标记为 `dirty`，这样渲染过程就能跳过干净的子树。

**Element name types:**
```typescript
type ElementNames =
  | 'ink-root'      // 文档根；拥有 FocusManager
  | 'ink-box'       // Flex 容器（yoga 节点）
  | 'ink-text'      // 叶子文本容器（yoga measure func）
  | 'ink-virtual-text' // 行内文本，没有 yoga 节点
  | 'ink-link'      // OSC 8 超链接包装器，没有 yoga 节点
  | 'ink-progress'  // 终端进度指示器，没有 yoga 节点
  | 'ink-raw-ansi'  // 预渲染 ANSI，带 rawWidth/rawHeight 的 yoga 节点

type TextName = '#text'
type NodeNames = ElementNames | TextName
```

**`DOMElement` structure:**
```typescript
type DOMElement = {
  nodeName: ElementNames
  attributes: Record<string, DOMNodeAttribute>
  childNodes: DOMNode[]
  textStyles?: TextStyles
  onComputeLayout?: () => void   // 由 reconciler.resetAfterCommit 调用
  onRender?: () => void           // 指向节流后的 scheduleRender
  onImmediateRender?: () => void  // 同步调用，用于测试
  hasRenderedContent?: boolean    // React 19 测试模式保护
  dirty: boolean
  isHidden?: boolean
  _eventHandlers?: Record<string, unknown>  // 事件处理器（和 attrs 分开存）
  // 滚动状态（overflow: scroll 的盒子）：
  scrollTop?: number
  pendingScrollDelta?: number
  scrollClampMin?: number
  scrollClampMax?: number
  scrollHeight?: number
  scrollViewportHeight?: number
  scrollViewportTop?: number
  stickyScroll?: boolean
  scrollAnchor?: { el: DOMElement; offset: number }
  focusManager?: FocusManager    // 只在 ink-root 上有
  debugOwnerChain?: string[]     // CLAUDE_CODE_DEBUG_REPAINTS 模式
  yogaNode?: LayoutNode
  style: Styles
  parentNode: DOMElement | undefined
}
```

**导出函数：**
- `createNode(nodeName)` — 分配一个 `DOMElement`；为 `ink-text` 和 `ink-raw-ansi` 挂上 yoga measure 函数
- `createTextNode(text)` — 分配一个 `TextNode`
- `appendChildNode(node, childNode)` — 追加子节点，同步 yoga 树，标记脏
- `insertBeforeNode(node, newChild, beforeChild)` — 在前面插入；yoga 索引要单独算，因为有些节点没有 yoga 节点
- `removeChildNode(node, removeNode)` — 删除子节点，收集 `pendingClears`，标记脏
- `setAttribute(node, key, value)` — 只在值变化时设置属性；跳过 `children`
- `setStyle(node, style)` — 浅比较样式对象；没变就跳过
- `setTextStyles(node, textStyles)` — 浅比较；没变就跳过
- `setTextNodeValue(node, text)` — 更新文本并标记脏
- `markDirty(node?)` — 沿祖先链把 `dirty = true`；对 `ink-text`/`ink-raw-ansi` 叶子节点标记 yoga 脏
- `scheduleRenderFrom(node?)` — 一路走到根并调用 `onRender()`
- `clearYogaNodeReferences(node)` — 递归清空 `yogaNode` 指针（先于 `freeRecursive()` 调用）
- `findOwnerChainAtRow(root, y)` — DFS 找到覆盖屏幕第 `y` 行的 React 组件栈（调试重绘模式）

**脏检查优化：** `stylesEqual` 和 `shallowEqual` 会阻止 React 每次渲染都创建一个值相同的新样式对象时把节点标脏。

**Yoga 索引与 DOM 索引：** `ink-virtual-text`、`ink-link` 和 `ink-progress` 这类节点没有 yoga 节点。`insertBeforeNode` 只统计有 yoga 的子节点，来算出正确的 yoga 插入位置。

---

## Reconciler

### `/x/Bigger-Projects/Claude-Code/src/ink/reconciler.ts`

**用途：** 配置 `react-reconciler` 把 ink 虚拟 DOM 当作宿主环境。这是 React fiber 树和 ink DOM 树之间的桥。

**Reconciler type parameters:**
```typescript
createReconciler<
  ElementNames,   // Type
  Props,          // Props
  DOMElement,     // Container
  DOMElement,     // Instance
  TextNode,       // TextInstance
  DOMElement,     // SuspenseInstance
  unknown,        // HydratableInstance
  unknown,        // PublicInstance
  DOMElement,     // HostContext (root)
  HostContext,    // ChildSet
  null,           // UpdatePayload (unused in React 19)
  NodeJS.Timeout, // TimeoutHandle
  -1,             // NoTimeout
  null            // TransitionStatus
>
```

**Key reconciler methods:**

| 方法 | 行为 |
|---|---|
| `createInstance(type, props, rootContainer, context, fiber)` | 调用 `createNode(type)`；通过 `applyProp` 应用所有属性；必要时从 fiber 里捕获 `debugOwnerChain` |
| `createTextInstance(text)` | 调用 `createTextNode(text)` |
| `appendInitialChild` / `appendChild` | 调用 `appendChildNode` |
| `insertBefore` | 调用 `insertBeforeNode` |
| `removeChild` | 调用 `removeChildNode`；并通知 `focusManager.handleNodeRemoved` |
| `commitUpdate(instance, updatePayload, type, oldProps, newProps)` | 对比新旧 props 并应用变更；用 `diff()` 找出变化的键 |
| `commitTextUpdate(textInstance, oldText, newText)` | 调用 `setTextNodeValue` |
| `hideInstance(instance)` / `unhideInstance(instance)` | 设置 `isHidden` 和 `LayoutDisplay.None` / 恢复显示 |
| `prepareForCommit` | 记录开始时间 |
| `resetAfterCommit(rootNode)` | 记录提交耗时；调用 `onComputeLayout()`（yoga 布局）；测试模式下触发 `onImmediateRender`；生产模式下调用 `onRender()` |
| `commitMount(instance, type, props)` | 如果 `autoFocus` 设了，就调用 `focusManager.focus()` |

**`applyProp(node, key, value)`:** Routes to `setStyle` (key=`style`), `setTextStyles` (key=`textStyles`), `setEventHandler` (key in `EVENT_HANDLER_PROPS`), or `setAttribute`.

**事件处理器分离：** 事件处理器 props 存在 `node._eventHandlers` 里，而不是 `node.attributes` 里。这样处理器引用变了也不会把节点标脏，避免破坏 blit 优化。

**`getOwnerChain(fiber)`:** 沿着 React fiber 的 `_debugOwner` / `return` 链收集组件名。用于 `CLAUDE_CODE_DEBUG_REPAINTS` 模式，把闪烁归因到源组件。

**性能埋点导出：**
- `recordYogaMs(ms)` / `getLastYogaMs()` — yoga 布局耗时
- `markCommitStart()` / `getLastCommitMs()` — React commit 耗时
- `resetProfileCounters()`
- `dispatcher` — `Dispatcher` 实例（事件分发）
- `isDebugRepaintsEnabled()` — 只读取一次 `CLAUDE_CODE_DEBUG_REPAINTS` 环境变量

---

## 布局引擎

### `/x/Bigger-Projects/Claude-Code/src/ink/layout/node.ts` — 布局节点接口

**用途：** 布局节点的抽象接口。把 ink DOM 和具体的 Yoga WASM 实现解耦。

**Constants:**
```typescript
LayoutEdge: { All, Horizontal, Vertical, Left, Right, Top, Bottom, Start, End }
LayoutGutter: { All, Column, Row }
LayoutDisplay: { Flex, None }
LayoutFlexDirection: { Row, RowReverse, Column, ColumnReverse }
LayoutAlign: { Auto, Stretch, FlexStart, Center, FlexEnd }
LayoutJustify: { FlexStart, Center, FlexEnd, SpaceBetween, SpaceAround, SpaceEvenly }
LayoutWrap: { NoWrap, Wrap, WrapReverse }
LayoutPositionType: { Relative, Absolute }
LayoutOverflow: { Visible, Hidden, Scroll }
LayoutMeasureMode: { Undefined, Exactly, AtMost }
```

**`LayoutNode` interface** (abbreviated):
```typescript
interface LayoutNode {
  // Tree operations
  insertChild(child, index): void
  removeChild(child): void
  getChildCount(): number
  getParent(): LayoutNode | null

  // Layout computation
  calculateLayout(width?, height?): void
  setMeasureFunc(fn: LayoutMeasureFunc): void
  unsetMeasureFunc(): void
  markDirty(): void

  // Layout reading (post-layout)
  getComputedLeft(): number
  getComputedTop(): number
  getComputedWidth(): number
  getComputedHeight(): number
  getComputedBorder(edge): number
  getComputedPadding(edge): number

  // Style setters (width, height, min/max, flex*, align*, justify, display,
  //                position, overflow, margin, padding, border, gap)

  // Lifecycle
  free(): void
  freeRecursive(): void
}
```

### `/x/Bigger-Projects/Claude-Code/src/ink/layout/yoga.ts` — Yoga 适配器

**用途：** 通过封装原生 Yoga WASM 节点（`src/native-ts/yoga-layout`）来实现 `LayoutNode`。把 `LayoutEdge`/`LayoutGutter`/等字符串枚举映射成 Yoga 枚举值。

**Class `YogaLayoutNode`:**
- Holds a `YogaNode` (`this.yoga`)
- `setMeasureFunc`: wraps the `LayoutMeasureFunc` to translate `MeasureMode` enum values
- `calculateLayout(width)`: calls `this.yoga.calculateLayout(width, undefined, Direction.LTR)` — height is always undefined (intrinsic)

边和间距枚举映射存放在静态的 `EDGE_MAP` 和 `GUTTER_MAP` 对象里。

### `/x/Bigger-Projects/Claude-Code/src/ink/layout/engine.ts`

**用途：** 布局节点工厂。它只是多了一层一行的转发：`createLayoutNode()` 调用 `createYogaLayoutNode()`。这样就能在不改 DOM 层的情况下替换布局后端。

### `/x/Bigger-Projects/Claude-Code/src/ink/layout/geometry.ts`

**用途：** 渲染流水线中会用到的二维几何原语。

**导出：**
```typescript
type Point = { x: number; y: number }
type Size = { width: number; height: number }
type Rectangle = Point & Size
type Edges = { top: number; right: number; bottom: number; left: number }

edges(all): Edges
edges(v, h): Edges
edges(t, r, b, l): Edges
addEdges(a, b): Edges
resolveEdges(partial?): Edges
ZERO_EDGES: Edges
unionRect(a, b): Rectangle   // bounding union
clampRect(rect, size): Rectangle
withinBounds(size, point): boolean
clamp(value, min?, max?): number
```

---

## 样式

### `/x/Bigger-Projects/Claude-Code/src/ink/styles.ts`

**用途：** box/text 样式的 TypeScript 类型，以及把样式 props 转成 `LayoutNode` 调用的 `applyStyles()`。

**Color types:**
```typescript
type RGBColor = `rgb(${number},${number},${number})`
type HexColor = `#${string}`
type Ansi256Color = `ansi256(${number})`
type AnsiColor = 'ansi:black' | 'ansi:red' | ...  // 16 named colors
type Color = RGBColor | HexColor | Ansi256Color | AnsiColor
```

**`TextStyles`：** `{ color?, backgroundColor?, dim?, bold?, italic?, underline?, strikethrough?, inverse? }` —— 渲染时通过 chalk/colorize 应用，不属于 yoga 属性。

**`Styles`:** The complete set of layout and text style props including:
- `textWrap`: `'wrap' | 'wrap-trim' | 'end' | 'middle' | 'truncate-end' | 'truncate' | 'truncate-middle' | 'truncate-start'`
- `position`: `'absolute' | 'relative'`
- `top | bottom | left | right`: `number | '${number}%'`
- `columnGap | rowGap | gap`: number
- `margin | marginX | marginY | marginTop | marginBottom | marginLeft | marginRight`: number
- `padding | paddingX | paddingY | paddingTop | paddingBottom | paddingLeft | paddingRight`: number
- `flexGrow | flexShrink | flexBasis`: number
- `flexDirection`: `'row' | 'row-reverse' | 'column' | 'column-reverse'`
- `flexWrap`: `'wrap' | 'nowrap' | 'wrap-reverse'`
- `alignItems | alignSelf | justifyContent`
- `width | height | minWidth | minHeight | maxWidth | maxHeight`: number or `'${number}%'` or `'100%'`
- `display`: `'flex' | 'none'`
- `overflow | overflowX | overflowY`: `'visible' | 'hidden' | 'scroll'`
- `borderStyle`: `BorderStyle`
- `borderColor | borderTopColor | borderRightColor | borderBottomColor | borderLeftColor`: Color
- `borderDimColor | ...`: boolean
- `borderTop | borderRight | borderBottom | borderLeft`: boolean
- `color | backgroundColor | dimColor | bold | italic | underline | strikethrough | inverse`

**`applyStyles(yogaNode, styles)`:** Translates each Styles property to `yogaNode.setXxx()` calls. Percentage values use `setWidthPercent`, etc. Position/overflow/display use enum mapping.

---

## 屏幕缓冲区

### `/x/Bigger-Projects/Claude-Code/src/ink/screen.ts`

**用途：** 核心的基于单元格的屏幕缓冲区。把渲染结果存成一个二维单元格网格。每个单元格都压缩进两个 32 位整数里，以节省内存。

**Cell encoding (packed as two `Int32Array` elements per cell):**
- Word 0 (low 32 bits): `charId` (high 22 bits) | `styleId` (low 10 bits)
- Word 1 (high 32 bits): `hyperlinkId` (high 16 bits) | `width` (2 bits) | flags

`CellWidth` enum: `Single = 0`, `Wide = 1`, `SpacerTail = 2`, `SpacerHead = 3`

**Pools (shared across all screens for zero-allocation diffing):**

`CharPool`:
- `intern(char)`: returns a stable integer ID; ASCII chars use a fast `Int32Array` lookup; others use a `Map`
- `get(id)`: retrieves the string
- Pool index 0 = space, index 1 = empty (spacer cell)

`HyperlinkPool`:
- `intern(hyperlink?)`: returns 0 for no hyperlink
- `get(id)`: returns the URL string or `undefined`

`StylePool`:
- `intern(styles: AnsiCode[])`: returns a tagged integer ID; bit 0 = `VISIBLE_ON_SPACE` flag (background/inverse/underline affect spaces)
- `get(id)`: strips bit-0 flag and returns `AnsiCode[]`
- `transition(fromId, toId)`: returns the cached ANSI transition string (pre-serialized diff)
- `withInverse(baseId)`: returns ID of style with SGR 7 (inverse) added
- `withCurrentMatch(baseId)`: returns ID of style with inverse + bold + yellow-fg + underline (current search match)
- `withSelectionBg(baseId)`: returns ID of style with selection background color applied
- `setSelectionBg(bg)`: sets the selection background `AnsiCode` (clears cache)

**`Screen` type:**
```typescript
type Screen = {
  cells: Int32Array   // packed cell data, width * height * 2 words per cell
  width: number
  height: number
  charPool: CharPool
  stylePool: StylePool
  hyperlinkPool: HyperlinkPool
  softWrap: Uint8Array  // 1 bit per row; 1 = soft-wrapped continuation
  noSelect: Uint8Array  // 1 bit per cell; set on gutter/non-selectable regions
}
```

**Key functions:**
- `createScreen(width, height, stylePool, charPool, hyperlinkPool)` — allocates a new screen
- `resetScreen(screen)` — zeroes all cells
- `setCellAt(screen, x, y, char, styleId, width, hyperlink?)` — writes a cell
- `cellAt(screen, x, y)` — reads a cell
- `cellAtIndex(screen, idx)` — reads cell at raw index
- `isEmptyCellAt(screen, x, y)` — both packed words = 0
- `blitRegion(dst, src, srcRect, dstX, dstY)` — bulk-copy a rectangle from one screen to another (pure `Int32Array` copy, no decoding)
- `shiftRows(screen, top, bottom, delta)` — hardware scroll simulation: move rows up/down within bounds
- `markNoSelectRegion(screen, x, y, w, h)` — set the noSelect bit on a rectangular region
- `diffEach(prev, next, callback)` — iterate only changed cells between two screens
- `migrateScreenPools(screen, charPool, hyperlinkPool)` — re-intern all cells when pools are replaced (generational reset)

---

## 渲染流水线

### `/x/Bigger-Projects/Claude-Code/src/ink/renderer.ts`

**用途：** 创建一个 `Renderer` 函数（对根 DOM 节点和 `StylePool` 的闭包），每次调用时把虚拟 DOM 转成一个 `Frame` 对象。

**`Renderer` type:** `(options: RenderOptions) => Frame`

**`RenderOptions`:**
```typescript
{
  frontFrame: Frame
  backFrame: Frame
  isTTY: boolean
  terminalWidth: number
  terminalRows: number
  altScreen: boolean
  prevFrameContaminated: boolean
}
```

**Algorithm:**
1. Check yoga computed dimensions; return empty frame if invalid
2. For alt-screen: clamp `height` to `terminalRows` (enforces the invariant)
3. Reuse or create `Output` with the back-buffer screen
4. Reset `layoutShifted`, `scrollHint`, `scrollDrainNode`
5. If `prevFrameContaminated` or an absolute node was removed: disable blit (pass `prevScreen = undefined`)
6. Call `renderNodeToOutput(node, output, { prevScreen })`
7. After render: if a scroll-drain node remains, call `markDirty(drainNode)` to schedule the next drain frame
8. Return `Frame` with the rendered screen, viewport, cursor position, and scroll hint

**Cursor position:**
- Alt-screen: `y = min(screen.height, terminalRows) - 1` — keeps cursor inside viewport to prevent LF-induced scroll
- Main-screen: `y = screen.height`
- `visible = !isTTY || screen.height === 0` — cursor is hidden during active rendering

The `Output` instance is reused across frames so its `charCache` (per-line grapheme cluster cache) persists between renders, making steady-state spinner/clock renders near zero-allocation.

### `/x/Bigger-Projects/Claude-Code/src/ink/render-node-to-output.ts`

**用途：** 递归遍历 DOM 树，把 write/blit/clear/clip/shift 操作写到 `Output` 缓冲区里。这就是从布局到像素的桥。

**Key exported state:**
- `didLayoutShift()` / `resetLayoutShifted()` — set when any node moves; gates the full-damage path in `ink.tsx`
- `getScrollHint()` / `resetScrollHint()` — DECSTBM hardware scroll hint for `LogUpdate`
- `getScrollDrainNode()` / `resetScrollDrainNode()` — identifies a ScrollBox with remaining `pendingScrollDelta`
- `consumeFollowScroll()` — consumes the at-bottom follow-scroll event for selection adjustment

**Scroll drain parameters:**
```
SCROLL_MIN_PER_FRAME = 4        // minimum rows applied per frame
SCROLL_INSTANT_THRESHOLD = 5   // ≤ this: drain all at once (xterm.js wheel click)
SCROLL_HIGH_PENDING = 12        // threshold for high-speed drain
SCROLL_STEP_MED = 2             // medium pending drain step
SCROLL_STEP_HIGH = 3            // high pending drain step
```

**ScrollBox rendering (three-pass algorithm):**
1. **First pass:** compute scrollHeight from yoga, apply `pendingScrollDelta` (proportional drain), handle `stickyScroll`, detect `scrollAnchor`
2. **Second pass (blit path):** when content is unchanged and layout hasn't shifted, blit the scrollbox from `prevScreen` and emit a hardware `shiftRows` hint
3. **Third pass (full render):** render children with `clip` and `y-offset = -scrollTop`; record `absoluteRectsCur` for position:absolute descendants

**Blit optimization:** When a node's bounding box matches `nodeCache` and the node is not dirty, the renderer blits from `prevScreen` instead of re-rendering. The condition is: `!dirty && prevScreen && !hasRemovedChild && !layoutShifted`. This makes steady-state frames O(changed cells).

**`nodeCache` updates:** After rendering each node, `nodeCache.set(node, { x, y, width, height })` records the screen-space bounding box for hit-testing and blit reuse.

### `/x/Bigger-Projects/Claude-Code/src/ink/output.ts`

**用途：** 收集渲染操作（write、blit、clip、clear、no-select、shift），并在 `get()` 里把它们应用到 `Screen` 缓冲区。

**`Operation` union type:**
```typescript
type Operation =
  | WriteOperation    // { type: 'write', x, y, text, softWrap? }
  | ClipOperation     // { type: 'clip', clip: Clip }
  | UnclipOperation   // { type: 'unclip' }
  | BlitOperation     // { type: 'blit', srcScreen, srcRect, dstX, dstY }
  | ClearOperation    // { type: 'clear', x, y, width, height }
  | NoSelectOperation // { type: 'noSelect', x, y, width, height }
  | ShiftOperation    // { type: 'shift', top, bottom, delta }
```

**`Clip` type:** `{ x1?, x2?, y1?, y2? }` — undefined on an axis means unbounded. Clips are intersected when nested.

**Write operation processing:**
The `WriteOperation` handler is the hot path. For each character:
1. Tokenize the text (ANSI codes) via `@alcalzone/ansi-tokenize`
2. Build a `ClusteredChar[]` array (grapheme + width + styleId + hyperlink), cached per unique line via `charCache: Map<string, ClusteredChar[]>`
3. Apply bidirectional reordering (`reorderBidi`) on Windows/xterm.js
4. Call `setCellAt` for each grapheme

**`charCache`:** Keyed by the raw ANSI string line. Cache miss: tokenize + cluster + intern styles. Cache hit: reuse the `ClusteredChar[]` array directly. The cache persists across frames (it lives on the `Output` instance). This is the dominant performance optimization for text-heavy content.

**Tab expansion:** Tabs in text are expanded to spaces at write time using screen x-position (not at measurement time), so the rendered width matches the measured width.

### `/x/Bigger-Projects/Claude-Code/src/ink/render-to-screen.ts`

**用途：** 用于搜索扫描的离屏渲染器。它会把 React 元素渲染到一个指定宽度的独立 `Screen` 缓冲区里，不会写到终端。用于预扫描消息内容，找出搜索匹配位置。

**`renderToScreen(el, width)`:** Returns `{ screen: Screen; height: number }`. Uses a shared persistent root/container/pools (LegacyRoot mode for synchronous rendering) so repeated calls reuse Yoga nodes. Unmounts between calls to free resources.

**`scanPositions(screen, query)`:** Scans a rendered screen for all occurrences of `query`, returning `MatchPosition[]` with `{ row, col, len }` in message-relative coordinates.

**`applyPositionedHighlight(screen, positions, rowOffset, currentIdx, stylePool)`:** Applies the current-match yellow/bold/underline style to the cell range at `positions[currentIdx]` and inverse style to all other positions.

### `/x/Bigger-Projects/Claude-Code/src/ink/frame.ts`

**用途：** 定义 `Frame` 类型以及 diff/patch 的类型层次。

**`Frame` type:**
```typescript
type Frame = {
  readonly screen: Screen
  readonly viewport: Size
  readonly cursor: Cursor
  readonly scrollHint?: ScrollHint | null
  readonly scrollDrainPending?: boolean
}
```

**`Patch` union type:**
```typescript
type Patch =
  | { type: 'stdout'; content: string }
  | { type: 'clear'; count: number }
  | { type: 'clearTerminal'; reason: FlickerReason; debug?: {...} }
  | { type: 'cursorHide' }
  | { type: 'cursorShow' }
  | { type: 'cursorMove'; x: number; y: number }
  | { type: 'cursorTo'; col: number }
  | { type: 'carriageReturn' }
  | { type: 'hyperlink'; uri: string }
  | { type: 'styleStr'; str: string }

type Diff = Patch[]
```

**`shouldClearScreen(prevFrame, frame)`:** Returns `'resize' | 'offscreen' | undefined`:
- `'resize'` — viewport dimensions changed
- `'offscreen'` — current or previous screen height exceeds viewport rows

**`FlickerReason`:** `'resize' | 'offscreen' | 'clear'`

**`FrameEvent`:** Timing breakdown emitted to `onFrame`:
```typescript
type FrameEvent = {
  durationMs: number
  phases?: {
    renderer: number; diff: number; optimize: number; write: number
    patches: number; yoga: number; commit: number
    yogaVisited: number; yogaMeasured: number; yogaCacheHits: number; yogaLive: number
  }
  flickers: Array<{ desiredHeight, availableHeight, reason }>
}
```

### `/x/Bigger-Projects/Claude-Code/src/ink/log-update.ts` — Diff 引擎

**用途：** 通过比较新 `Frame` 和旧帧来计算 `Diff`（`Patch` 对象列表）；必要时也处理整屏清空。

**`LogUpdate` class:**

Constructor takes `{ isTTY, stylePool }`. Maintains `previousOutput: string` (deprecated legacy string tracking).

Key methods:
- `render(prevFrame, frame)` — main diff entry point. If `shouldClearScreen()` triggers, prepends a `clearTerminal` patch. Otherwise runs the incremental diff algorithm
- `renderPreviousOutput_DEPRECATED(prevFrame)` — used for final output on exit (writes the last frame to terminal)
- `reset()` — clears `previousOutput` (called after SIGCONT)

**Incremental diff algorithm (`renderDiff`):**
1. Walk rows top-down, comparing `prevScreen` and `screen` cell-by-cell via `diffEach`
2. For unchanged rows: emit cursor moves to skip them
3. For changed rows: emit `styleStr` transitions + `stdout` content patches + `hyperlink` patches
4. Handle wide chars: skip `SpacerTail` cells; emit a space for `SpacerHead` (end-of-line wrap guard)
5. Track hyperlink state across rows (emit `LINK_END` when hyperlink changes)
6. After last row: position cursor per `frame.cursor`

**DECSTBM hardware scroll (`scrollHint`):**
When the back frame has a `scrollHint` and no layout shift occurred and the viewport can accommodate the shift:
- Emit `setScrollRegion(top, bottom)` (DECSTBM)
- Emit `scrollUp(n)` (CSI S) or `scrollDown(n)` (CSI T)
- Emit `RESET_SCROLL_REGION`
- Only repaint the rows that changed due to the scroll (a narrow repair band)

This replaces O(viewport) cell writes with O(scrolled region) writes for smooth scroll in fullscreen mode.

### `/x/Bigger-Projects/Claude-Code/src/ink/optimizer.ts`

**用途：** 单遍补丁列表优化器。

**`optimize(diff)`:** Rules applied:
- Remove empty `stdout` patches
- Remove no-op `cursorMove(0,0)` patches
- Remove `clear` patches with count 0
- **Merge** consecutive `cursorMove` patches (add x/y)
- **Collapse** consecutive `cursorTo` patches (keep last)
- **Concat** adjacent `styleStr` patches
- **Deduplicate** consecutive `hyperlink` patches with same URI
- **Cancel** adjacent `cursorHide`/`cursorShow` pairs

### `/x/Bigger-Projects/Claude-Code/src/ink/node-cache.ts`

**用途：** 保存每个 `DOMElement` 的渲染边界框，用于 blit 优化和命中测试。

**导出：**
```typescript
type CachedLayout = { x: number; y: number; width: number; height: number; top?: number }
const nodeCache = new WeakMap<DOMElement, CachedLayout>()
const pendingClears = new WeakMap<DOMElement, Rectangle[]>()

addPendingClear(parent, rect, isAbsolute): void
consumeAbsoluteRemovedFlag(): boolean
```

`pendingClears` holds rectangles of removed children that must be painted over on the next render. `absoluteNodeRemoved` gates the global blit disable path for absolute-positioned removals.

---

## 终端 I/O

### `/x/Bigger-Projects/Claude-Code/src/ink/terminal.ts`

**用途：** 终端能力检测，以及 `writeDiffToTerminal` 序列化器。

**`Terminal` type:** `{ stdout: NodeJS.WriteStream; stderr: NodeJS.WriteStream }`

**Capability detection:**
- `isSynchronizedOutputSupported()` — returns `true` for iTerm2, WezTerm, Warp, ghostty, kitty, VS Code, alacritty, foot, etc. Returns `false` for tmux (parses but doesn't implement DEC 2026 properly)
- `isProgressReportingAvailable()` — returns `true` for ConEmu, Ghostty 1.2.0+, iTerm2 3.6.6+; excludes Windows Terminal (interprets OSC 9;4 as notifications)
- `supportsExtendedKeys()` — detects Kitty keyboard protocol or modifyOtherKeys support
- `isXtermJs()` — set by XTVERSION probe (survives SSH, unlike TERM_PROGRAM)
- `setXtversionName(name)` — called by `App.tsx` after terminal query response

**`writeDiffToTerminal(terminal, diff, syncOutput)`:**
Serializes a `Diff` into ANSI sequences and writes to `terminal.stdout`. If `syncOutput === SYNC_OUTPUT_SUPPORTED`, wraps in BSU/ESU (DEC 2026 synchronized output). The serialization is a tight loop over the `Patch` array:
- `stdout`: write content directly
- `clear`: `eraseLines(count)` — moves cursor up and erases
- `clearTerminal`: `getClearTerminalSequence()` — erase screen + scrollback
- `cursorHide` / `cursorShow`: emit HIDE/SHOW_CURSOR sequences
- `cursorMove`: emit `cursorMove(x, y)` relative move
- `cursorTo`: emit `cursorTo(col)` absolute column
- `carriageReturn`: emit `\r`
- `hyperlink`: emit `link(uri)` or `LINK_END`
- `styleStr`: write pre-serialized ANSI transition string directly

`SYNC_OUTPUT_SUPPORTED` constant is computed once at module load.

---

## Termio 层

### `/x/Bigger-Projects/Claude-Code/src/ink/termio.ts` — Termio 公共 API

Re-exports:
- `Parser` from `termio/parser.ts`
- All types: `Action`, `Color`, `CursorAction`, `CursorDirection`, `EraseAction`, `Grapheme`, `LinkAction`, `ModeAction`, `NamedColor`, `ScrollAction`, `TextSegment`, `TextStyle`, `TitleAction`, `UnderlineStyle`
- `colorsEqual`, `defaultStyle`, `stylesEqual`

### `/x/Bigger-Projects/Claude-Code/src/ink/termio/types.ts`

**用途：** 所有 ANSI 解析器输出的语义类型定义。

**Key types:**
```typescript
// 16-color palette
type NamedColor = 'black' | 'red' | ... | 'brightWhite'

// 3-way color union
type Color =
  | { type: 'named'; name: NamedColor }
  | { type: 'indexed'; index: number }   // 0-255
  | { type: 'rgb'; r: number; g: number; b: number }
  | { type: 'default' }

type UnderlineStyle = 'none' | 'single' | 'double' | 'curly' | 'dotted' | 'dashed'

type TextStyle = {
  bold: boolean; dim: boolean; italic: boolean; underline: UnderlineStyle
  blink: boolean; inverse: boolean; hidden: boolean; strikethrough: boolean
  overline: boolean; fg: Color; bg: Color; underlineColor: Color
}

// All parsed actions
type Action =
  | { type: 'text'; graphemes: Grapheme[]; style: TextStyle }
  | { type: 'cursor'; action: CursorAction }
  | { type: 'erase'; action: EraseAction }
  | { type: 'scroll'; action: ScrollAction }
  | { type: 'mode'; action: ModeAction }
  | { type: 'link'; action: LinkAction }
  | { type: 'title'; action: TitleAction }
  | { type: 'tabStatus'; action: TabStatusAction }
  | { type: 'sgr'; params: string }
  | { type: 'bell' }
  | { type: 'reset' }
  | { type: 'unknown'; sequence: string }
```

**`TabStatusAction`:** `{ indicator?: Color | null; status?: string | null; statusColor?: Color | null }` — for OSC 21337 tab chrome metadata.

Utility functions: `defaultStyle()`, `stylesEqual(a, b)`, `colorsEqual(a, b)`.

### `/x/Bigger-Projects/Claude-Code/src/ink/termio/ansi.ts`

**用途：** 基础 ANSI 常量和 C0 控制字符代码。

**导出：**
- `C0` object — complete C0 control character table (NUL through DEL)
- `ESC = '\x1b'`, `BEL = '\x07'`, `SEP = ';'`
- `ESC_TYPE` — escape sequence introducers: `CSI=0x5b, OSC=0x5d, DCS=0x50, APC=0x5f, PM=0x5e, SOS=0x58, ST=0x5c`
- `isC0(byte)`, `isEscFinal(byte)`

### `/x/Bigger-Projects/Claude-Code/src/ink/termio/csi.ts`

**用途：** CSI（Control Sequence Introducer）序列生成与常量。

**导出：**
- `CSI_PREFIX = ESC + '['`
- `CSI_RANGE` — parameter/intermediate/final byte ranges
- `isCSIParam`, `isCSIIntermediate`, `isCSIFinal`
- `csi(...args)` — sequence generator: `ESC [ params... final`
- `CSI` enum — final byte codes: `CUU=0x41(A), CUD=0x42(B), CUF=0x43(C), CUB=0x44(D), CNL=0x45(E), CPL=0x46(F), CHA=0x47(G), CUP=0x48(H), ED=0x4a(J), EL=0x4b(K), SU=0x53(S), SD=0x54(T), SGR=0x6d(m), DECSTBM=0x72(r), ...`
- Pre-generated sequences: `CURSOR_HOME`, `ERASE_SCREEN`, `ERASE_SCROLLBACK`, `RESET_SCROLL_REGION`, `PASTE_START`, `PASTE_END`, `FOCUS_IN`, `FOCUS_OUT`, `ENABLE_KITTY_KEYBOARD`, `DISABLE_KITTY_KEYBOARD`, `ENABLE_MODIFY_OTHER_KEYS`, `DISABLE_MODIFY_OTHER_KEYS`
- Parameterized: `cursorMove(x,y)`, `cursorTo(col)`, `cursorPosition(row,col)`, `eraseLines(count)`, `setScrollRegion(top,bottom)`, `scrollUp(n)`, `scrollDown(n)`

### `/x/Bigger-Projects/Claude-Code/src/ink/termio/dec.ts`

**用途：** DEC 私有模式序列生成。

**导出：**
- `DEC` enum — mode numbers: `CURSOR_VISIBLE=25, ALT_SCREEN=47, ALT_SCREEN_CLEAR=1049, MOUSE_NORMAL=1000, MOUSE_BUTTON=1002, MOUSE_ANY=1003, MOUSE_SGR=1006, FOCUS_EVENTS=1004, BRACKETED_PASTE=2004, SYNCHRONIZED_UPDATE=2026`
- `decset(mode)` / `decreset(mode)` — `CSI ? N h` / `CSI ? N l`
- Pre-generated: `BSU` (begin synchronized update), `ESU`, `EBP/DBP` (bracketed paste), `EFE/DFE` (focus events), `SHOW_CURSOR/HIDE_CURSOR`, `ENTER_ALT_SCREEN/EXIT_ALT_SCREEN`
- `ENABLE_MOUSE_TRACKING` — combination of 1000+1002+1003+1006 set
- `DISABLE_MOUSE_TRACKING` — reverse order reset

### `/x/Bigger-Projects/Claude-Code/src/ink/termio/osc.ts`

**用途：** OSC（Operating System Command）序列生成，以及剪贴板 / tab 状态支持。

**导出：**
- `OSC_PREFIX = ESC + ']'`, `ST = ESC + '\\'`
- `osc(...parts)` — generates `ESC ] parts BEL` (or ST for Kitty)
- `wrapForMultiplexer(sequence)` — wraps in tmux DCS passthrough (`ESC P tmux ; ... ESC \`) or GNU screen DCS if `TMUX`/`STY` env vars set
- `link(url)` / `LINK_END` — OSC 8 hyperlink start/end
- `setClipboard(text)` — OSC 52 + optional pbcopy/tmux load-buffer; returns `ClipboardPath`
- `getClipboardPath()` — `'native' | 'tmux-buffer' | 'osc52'` without side effects
- `tmuxLoadBuffer(text)` — async: runs `tmux load-buffer [-w] -`
- `tabStatus({indicator, status, statusColor})` / `CLEAR_TAB_STATUS` — OSC 21337 tab chrome
- `supportsTabStatus()` — detects iTerm2 / Claude Code terminal from env
- `CLEAR_ITERM2_PROGRESS` — clears iTerm2 progress bar

**Clipboard path decision:**
- `native`: macOS + no SSH_CONNECTION → use `pbcopy`
- `tmux-buffer`: inside tmux → use `tmux load-buffer [-w]`
- `osc52`: fallback → write OSC 52 raw sequence

### `/x/Bigger-Projects/Claude-Code/src/ink/termio/sgr.ts`

**用途：** SGR（Select Graphic Rendition）参数解析器。

**`applySGR(paramStr, style)`:** Parses semicolon/colon separated SGR params and mutates a `TextStyle`. Handles:
- SGR 0: reset
- SGR 1/2/3/4/5/7/8/9/53: bold/dim/italic/underline/blink/inverse/hidden/strikethrough/overline
- SGR 21/22/23/24/25/27/28/29/55: attribute reset
- SGR 30-37/90-97: named fg colors
- SGR 40-47/100-107: named bg colors
- SGR 38/48: extended fg/bg (256-indexed via `5` or truecolor via `2`)
- SGR 39/49: default fg/bg
- SGR 58/59: underline color
- Kitty extended underline styles (SGR 4:1-4:5)

Colon-separated subparams take priority over semicolon-separated for extended colors.

### `/x/Bigger-Projects/Claude-Code/src/ink/termio/esc.ts`

**用途：** ESC（非 CSI、非 OSC）序列解析器。处理 `ESC c`（完全重置）、`ESC 7`/`ESC 8`（保存/恢复光标）以及其他双字节序列。

**`parseEsc(sequence)`:** Returns an `Action | null`.

### `/x/Bigger-Projects/Claude-Code/src/ink/termio/tokenize.ts`

**用途：** 终端输入的流式分词器，把原始字节切成文本块和转义序列，但不解释其语义。

**States:** `ground`, `escape`, `escapeIntermediate`, `csi`, `ss3`, `osc`, `dcs`, `apc`

**`Token` type:** `{ type: 'text'; value: string } | { type: 'sequence'; value: string }`

**`Tokenizer` interface:**
```typescript
{
  feed(input: string): Token[]
  flush(): Token[]
  reset(): void
  buffer(): string
}
```

**`createTokenizer(options?)`:**
- `options.x10Mouse` — enables X10 legacy mouse event parsing (consume 3 extra bytes after `CSI M`)
- Maintains incremental state across `feed()` calls for streaming input

**Algorithm:** Character-by-character state machine. `ground` state: text passes through until `ESC` or C0 control chars. `csi` state: accumulates until CSI final byte (0x40–0x7e). `osc`/`dcs`/`apc` states: accumulate until BEL or ST.

### `/x/Bigger-Projects/Claude-Code/src/ink/termio/parser.ts`

**用途：** 包装分词器的语义解析器，会把每个序列解释成结构化的 `Action`。

**`Parser` class:**
```typescript
class Parser {
  feed(input: string): Action[]
  flush(): Action[]
  reset(): void
  getStyle(): TextStyle
}
```

**Internal structure:**
- Holds a `Tokenizer` with `x10Mouse: true`
- Maintains current `TextStyle` (updated by SGR actions)
- Calls `parseCSI`, `parseEsc`, `parseOSC` for sequence tokens
- Calls `segmentGraphemes` for text tokens

**Grapheme width detection:**
- `isEmoji(codePoint)` — ranges: 0x2600-0x26ff, 0x2700-0x27bf, 0x1F300-0x1F9FF, 0x1FA00-0x1FAFF, 0x1F1E0-0x1F1FF
- `isEastAsianWide(codePoint)` — standard CJK/Hangul ranges
- `graphemeWidth(grapheme)` — returns 1 or 2
- `segmentGraphemes(str)` — uses `Intl.Segmenter` to split by grapheme cluster

**`parseCSI(rawSequence)`:** Dispatches by final byte:
- `m` (SGR): `{ type: 'sgr', params }`
- Cursor movement (A-H, d, f): `{ type: 'cursor', action }`
- Erase (J, K, X): `{ type: 'erase', action }`
- Scroll (S, T): `{ type: 'scroll', action }`
- DEC private modes (h/l with `?` prefix): `{ type: 'mode', action }`
- Mouse events (M/m with `<` prefix): decoded SGR mouse

---

## 事件系统

### `/x/Bigger-Projects/Claude-Code/src/ink/events/event.ts`

**用途：** 提供 `stopImmediatePropagation()` 的基础 `Event` 类。

```typescript
class Event {
  stopImmediatePropagation(): void
  didStopImmediatePropagation(): boolean
}
```

### `/x/Bigger-Projects/Claude-Code/src/ink/events/terminal-event.ts`

**用途：** `TerminalEvent` 在 `Event` 基础上增加了 DOM 风格的传播属性。

**`EventTarget` type:** `{ parentNode: EventTarget | undefined; _eventHandlers?: Record<string, unknown> }`

**`TerminalEvent` properties:**
- `type: string`, `timeStamp: number`, `bubbles: boolean`, `cancelable: boolean`
- `target: EventTarget | null`, `currentTarget: EventTarget | null`
- `eventPhase: 'none' | 'capturing' | 'at_target' | 'bubbling'`
- `defaultPrevented: boolean`

**Methods:** `stopPropagation()`, `stopImmediatePropagation()` (overrides base), `preventDefault()`

Internal setters: `_setTarget`, `_setCurrentTarget`, `_setEventPhase`, `_isPropagationStopped()`, `_isImmediatePropagationStopped()`, `_prepareForTarget(target)` (hook for subclasses)

### `/x/Bigger-Projects/Claude-Code/src/ink/events/dispatcher.ts`

**用途：** 两阶段（capture + bubble）的事件分发器，参照浏览器 DOM 事件模型。

**`Dispatcher` class:**
- `dispatch(target, event)` — full capture+bubble cycle via `collectListeners` + `processDispatchQueue`; runs asynchronously (via `unstable_batchedUpdates` or React's scheduler for `discrete` vs `continuous` events)
- `dispatchDiscrete(target, event)` — discrete priority (keyboard, focus); triggers React's synchronous flush
- Internal `collectListeners(target, event)` — walks from target to root, prepending capture handlers (root-first) and appending bubble handlers (target-first); result: `[root-cap, ..., target-cap, target-bub, ..., root-bub]`
- `processDispatchQueue(listeners, event)` — calls each handler, checking `_isPropagationStopped()` and `_isImmediatePropagationStopped()` before each

**React event priority mapping:**
- Keyboard/focus → `DiscreteEventPriority`
- Mouse motion → `ContinuousEventPriority`
- Other → `DefaultEventPriority`

### `/x/Bigger-Projects/Claude-Code/src/ink/events/event-handlers.ts`

**用途：** 定义完整的事件处理器 props 集合以及反向查找表。

**`EventHandlerProps`:**
```typescript
{
  onKeyDown?, onKeyDownCapture?: KeyboardEventHandler
  onFocus?, onFocusCapture?, onBlur?, onBlurCapture?: FocusEventHandler
  onPaste?, onPasteCapture?: PasteEventHandler
  onResize?: ResizeEventHandler
  onClick?: ClickEventHandler
  onMouseEnter?, onMouseLeave?: HoverEventHandler
}
```

**`HANDLER_FOR_EVENT`:** Maps event type strings to `{ bubble?, capture? }` prop name pairs. Used by `Dispatcher` for O(1) handler lookup.

**`EVENT_HANDLER_PROPS`:** `Set<string>` of all handler prop names; used by the reconciler to route event props to `_eventHandlers` instead of `attributes`.

### `/x/Bigger-Projects/Claude-Code/src/ink/events/keyboard-event.ts`

**`KeyboardEvent extends TerminalEvent`:**
- Constructor takes a `ParsedKey`; type = `'keydown'`; `bubbles = true`; `cancelable = true`
- `key: string` — printable char for printable keys; multi-char name for specials (`'down'`, `'return'`, `'escape'`, `'f1'`, etc.)
- `ctrl, shift, meta, superKey, fn: boolean`

Key extraction: ctrl keys use the name (letter); single printable ASCII chars use the literal char; special keys use the parsed name.

### `/x/Bigger-Projects/Claude-Code/src/ink/events/click-event.ts`

**`ClickEvent extends Event`:**
- `col: number`, `row: number` — 0-indexed screen coordinates
- `localCol: number`, `localRow: number` — coordinates relative to the current handler's Box (updated by `dispatchClick` before each handler fires)
- `cellIsBlank: boolean` — true if the cell had no content (allows handlers to ignore clicks on empty space)

### Other event types:

**`InputEvent`** (events/input-event.ts): Legacy input event emitted on stdin data; carries `input: string` and `Key` object. `Key` type: `{ upArrow, downArrow, leftArrow, rightArrow, pageUp, pageDown, return, escape, ctrl, shift, tab, backspace, delete, meta, fn }`.

**`FocusEvent`** (events/focus-event.ts): `type = 'focus' | 'blur'`; `relatedTarget: DOMElement | null`

**`TerminalFocusEvent`** (events/terminal-focus-event.ts): `type: TerminalFocusEventType = 'terminal-focus-in' | 'terminal-focus-out'`; fired when DECSET 1004 focus events arrive.

**`EventEmitter`** (events/emitter.ts): Simple typed event emitter. `on(event, handler)`, `off(event, handler)`, `emit(event, ...args)`. Used by `App.tsx` for stdin data events.

---

## 焦点管理

### `/x/Bigger-Projects/Claude-Code/src/ink/focus.ts`

**`FocusManager` class:**

Stored on `ink-root` node so any node can reach it by walking `parentNode`.

State:
- `activeElement: DOMElement | null`
- `focusStack: DOMElement[]` — history for focus restoration (max 32 entries)
- `enabled: boolean`

Methods:
- `focus(node)` — blur previous, push to stack, focus new node; dispatches `focus`/`blur` events
- `blur()` — blur `activeElement`, dispatches `blur` event
- `handleNodeRemoved(node, root)` — removes node from stack, restores focus from stack if `activeElement` was in the removed subtree
- `handleClickFocus(node)` — called by `dispatchClick`; focuses the nearest focusable ancestor
- `getNextFocusable(root, direction)` — Tab/Shift+Tab cycling; collects all nodes with `tabIndex >= 0`, sorts by order, returns next/previous
- `enable()` / `disable()` — gates all focus operations

**`getFocusManager(node)`** / **`getRootNode(node)`** — utility functions exported for the reconciler.

---

## 输入解析

### `/x/Bigger-Projects/Claude-Code/src/ink/parse-keypress.ts`

**用途：** 把终端 stdin 的原始字节解析成结构化的 `ParsedKey` 对象。支持标准 ANSI 序列、Kitty 键盘协议（CSI u）、xterm modifyOtherKeys、SGR 鼠标事件和终端响应序列。

**`ParsedKey` type:**
```typescript
{
  kind: 'key' | 'mouse' | 'terminalResponse'
  name: string        // key name: 'up', 'down', 'return', 'escape', 'f1', 'a', ...
  fn: boolean
  ctrl: boolean; meta: boolean; shift: boolean; option: boolean; super: boolean
  sequence: string   // raw escape sequence
  raw: string        // original input bytes
  isPasted?: boolean
}
```

**`ParsedMouse` type:**
```typescript
{
  kind: 'mouse'
  button: number     // SGR button code
  col: number; row: number  // 1-indexed
  press: boolean    // true = press, false = release
  isWheel: boolean
  isDrag: boolean
  modifiers: { ctrl, shift, meta, alt }
}
```

**`ParsedInput = ParsedKey | ParsedMouse | TerminalResponse`**

**Regex patterns:**
- `META_KEY_CODE_RE`: `ESC + [a-zA-Z0-9]` — meta key combos
- `FN_KEY_RE`: SS3/CSI function key sequences
- `CSI_U_RE`: Kitty protocol `ESC [ codepoint [;modifier] u`
- `MODIFY_OTHER_KEYS_RE`: xterm `ESC [ 27 ; modifier ; keycode ~`
- `DECRPM_RE`, `DA1_RE`, `DA2_RE`, `KITTY_FLAGS_RE`, `CURSOR_POSITION_RE` — terminal response patterns
- `OSC_RESPONSE_RE`: OSC sequence responses
- `XTVERSION_RE`: DCS `>|` terminal name/version
- `SGR_MOUSE_RE`: `ESC [ < btn ; col ; row M/m`

**`parseMultipleKeypresses(buffer, state)`:** Main entry point. Uses the tokenizer to split the input, then dispatches each token to the appropriate parser. Handles bracketed paste (accumulates until `PASTE_END`).

**`INITIAL_STATE`:** Initial parser state for `parseMultipleKeypresses`.

---

## 命中测试

### `/x/Bigger-Projects/Claude-Code/src/ink/hit-test.ts`

**用途：** 针对 DOM 树的鼠标点击命中测试。

**`hitTest(node, col, row)`:** DFS in reverse child order (last child = top paint layer wins). Uses `nodeCache` for bounding-box lookups. Returns the deepest `DOMElement` whose rendered rect contains `(col, row)`, or `null`.

**`dispatchClick(root, col, row, cellIsBlank?)`:**
1. Runs `hitTest` to find the deepest hit node
2. Calls `focusManager.handleClickFocus()` to click-to-focus the nearest focusable ancestor
3. Creates a `ClickEvent(col, row, cellIsBlank)`
4. Bubbles up via `parentNode` chain, calling `onClick` handlers
5. Before each handler: sets `event.localCol/localRow` relative to the handler's bounding rect
6. Stops on `stopImmediatePropagation()`
7. Returns `true` if any handler fired

**`dispatchHover(root, col, row, hoveredNodes)`:** Diff-based hover dispatch. Finds the set of nodes hit at `(col, row)`, fires `onMouseEnter` for newly-entered nodes and `onMouseLeave` for exited nodes. Mutates `hoveredNodes` in place (owned by the `Ink` instance).

---

## 文本选择

### `/x/Bigger-Projects/Claude-Code/src/ink/selection.ts`

**用途：** 全屏模式下按屏幕缓冲区坐标进行的线性文本选择。

**`SelectionState` type:**
```typescript
{
  anchor: { col, row } | null
  focus: { col, row } | null
  isDragging: boolean
  anchorSpan: { lo, hi, kind: 'word' | 'line' } | null  // word/line mode
  scrolledOffAbove: string[]     // rows that scrolled off the top
  scrolledOffBelow: string[]
  scrolledOffAboveSW: boolean[]  // soft-wrap flags parallel to scrolledOffAbove
  scrolledOffBelowSW: boolean[]
  virtualAnchorRow?: number      // pre-clamp anchor row (for scroll restore)
  virtualFocusRow?: number
  lastPressHadAlt: boolean
}
```

**Exported functions:**
- `createSelectionState()` — returns zeroed state
- `startSelection(s, col, row, alt?)` — initializes anchor and focus
- `updateSelection(s, col, row)` — updates focus during drag
- `finishSelection(s)` — clears `isDragging`
- `clearSelection(s)` — resets to empty
- `hasSelection(s)` — returns true if `anchor !== null && focus !== null`
- `getSelectedText(s, screen)` — extracts the selected text; handles soft-wrap (joins wrapped lines), wide chars (skips `SpacerTail`), `noSelect` regions (excluded), and `scrolledOffAbove`/`scrolledOffBelow` accumulators
- `applySelectionOverlay(s, screen, stylePool)` — inverts cell styles in the selected region
- `selectWordAt(s, col, row, screen)` — double-click: select the word under the cursor
- `selectLineAt(s, row, screen)` — triple-click: select the entire line
- `extendSelection(s, col, row, screen)` — word/line-mode drag extension: extends to word/line boundaries
- `shiftSelection(s, dRow, min, max, screenWidth)` — keyboard scroll: shifts both anchor and focus
- `shiftAnchor(s, dRow, min, max)` — shift anchor only (keyboard selection extension)
- `moveFocus(s, move, screen)` — keyboard character/line focus extension
- `captureScrolledRows(s, firstRow, lastRow, side, screen)` — saves rows about to scroll off into `scrolledOffAbove`/`scrolledOffBelow`
- `shiftSelectionForFollow(s, delta, screen)` — called after follow-scroll to keep selection anchored to text

---

## 搜索高亮

### `/x/Bigger-Projects/Claude-Code/src/ink/searchHighlight.ts`

**`applySearchHighlight(screen, query, stylePool)`:**
- Case-insensitive scan of the screen buffer
- Builds per-row char text + `codeUnitToCell` index map (handles wide chars)
- For each match: calls `setCellStyleId(screen, ..., stylePool.withInverse(cellStyleId))` to invert the cell style
- Skips `noSelect` regions (gutters, line numbers)
- Returns `true` if any match was applied (triggers full-damage flag in caller)

The `applyPositionedHighlight` (in `render-to-screen.ts`) handles the "current match" in yellow, on top of `applySearchHighlight`'s inverse.

---

## 组件库

### `/x/Bigger-Projects/Claude-Code/src/ink/components/App.tsx`

**用途：** 根 React 组件。把 stdin 输入、终端尺寸上下文、焦点、时钟、错误边界和所有 context provider 连接起来。

**Props:** `children, stdin, stdout, stderr, exitOnCtrlC, onExit, terminalColumns, terminalRows, selection, onSelectionChange, onClickAt, onHoverAt, getHyperlinkAt, onOpenHyperlink, onMultiClick, onSelectionDrag, onStdinResume?, onCursorDeclaration`

**Responsibilities:**
- Provides `AppContext`, `StdinContext`, `TerminalSizeContext`, `ClockContext`, `TerminalFocusProvider`, `CursorDeclarationContext`, `TerminalWriteProvider`
- Listens to stdin data; calls `parseMultipleKeypresses` for each chunk
- Dispatches `KeyboardEvent` through the DOM via `dispatcher.dispatchDiscrete(rootNode, event)`
- Handles mouse events: left-button selection start/update/finish; wheel → `pendingScrollDelta`; hover → `onHoverAt`; click → `onClickAt`
- Handles pasted content via bracketed paste markers
- Detects terminal focus via DECSET 1004 `FOCUS_IN`/`FOCUS_OUT` sequences
- Runs `TerminalQuerier` on mount to probe XTVERSION and extended key support
- Re-asserts terminal modes after `STDIN_RESUME_GAP_MS = 5000` ms of stdin silence
- Handles `Ctrl+C` → `onExit` when `exitOnCtrlC = true`
- Handles `Ctrl+Z` (SIGTSTP) on non-Windows platforms

**Class component** (`PureComponent`) for stable reference identity. All mouse/keyboard state is imperative (refs), not React state, to avoid re-renders.

### `/x/Bigger-Projects/Claude-Code/src/ink/components/Box.tsx`

**用途：** 主要布局容器，类似于 `<div style="display: flex">`。

**`Props`:** All `Styles` properties (except `textWrap`) plus:
- `ref?: Ref<DOMElement>`
- `tabIndex?: number` — focus order (>= 0 participates in Tab cycling, -1 = programmatic only)
- `autoFocus?: boolean` — focus on mount
- `onClick?: (event: ClickEvent) => void`
- `onFocus?, onFocusCapture?, onBlur?, onBlurCapture?`
- `onKeyDown?, onKeyDownCapture?`
- `onMouseEnter?, onMouseLeave?`

Renders to `<ink-box>` host element. Compiled with React Compiler (memo cache `_c(42)`).

### `/x/Bigger-Projects/Claude-Code/src/ink/components/Text.tsx`

**用途：** 渲染带样式的文本。会把 children 包进 `ink-text` 和 `ink-virtual-text` 元素里。

**`Props`:**
- `color?, backgroundColor?`
- `bold?, dim?` (mutually exclusive via TypeScript union)
- `italic?, underline?, strikethrough?, inverse?`
- `wrap?: Styles['textWrap']`
- `children?: ReactNode`

Text styling props are mapped to `TextStyles` and passed as the `textStyles` prop on the host element. Layout props (flexDirection, flexGrow, etc.) are mapped to `Styles` on the `ink-text` node.

### `/x/Bigger-Projects/Claude-Code/src/ink/components/ScrollBox.tsx`

**用途：** 带命令式滚动 API 和视口裁剪能力的 Box。

**`ScrollBoxHandle`:**
```typescript
{
  scrollTo(y): void
  scrollBy(dy): void
  scrollToElement(el, offset?): void   // defers position read to render time
  scrollToBottom(): void
  getScrollTop(): number
  getPendingDelta(): number
  getScrollHeight(): number
  getFreshScrollHeight(): number       // reads Yoga directly
  getViewportHeight(): number
  getViewportTop(): number
  isSticky(): boolean
  subscribe(listener): () => void
  setClampBounds(min, max): void
}
```

**`ScrollBoxProps`:** All `Styles` except `overflow`/`overflowX`/`overflowY`, plus `stickyScroll?: boolean`.

Implementation: Sets `overflow: 'scroll'` on the underlying Box. Scroll mutations call `markDirty` + `scheduleRenderFrom` to trigger an Ink frame without going through React's reconciler. `scrollToElement` stores a `scrollAnchor` on the DOM node; `render-node-to-output` reads it at paint time (after Yoga has computed the element's position) and clears it.

### `/x/Bigger-Projects/Claude-Code/src/ink/components/AlternateScreen.tsx`

**用途：** 进入终端的备用屏幕缓冲区，用于全屏渲染。

**Props:** `children, mouseTracking?: boolean` (default `true`)

Uses `useInsertionEffect` (not `useLayoutEffect`) to send `ENTER_ALT_SCREEN` before the first Ink render frame. On cleanup (unmount), sends `EXIT_ALT_SCREEN`. Calls `instances.get(process.stdout).setAltScreenActive(true/false)` to coordinate with the Ink instance. Renders children inside a `Box` constrained to `terminalRows` height.

### `/x/Bigger-Projects/Claude-Code/src/ink/components/Link.tsx`

**用途：** 渲染 OSC 8 终端超链接。

Wraps children in an `ink-link` host element with `href` attribute. During rendering, `squashTextNodesToSegments` propagates the hyperlink URL to contained text segments.

### `/x/Bigger-Projects/Claude-Code/src/ink/components/RawAnsi.tsx`

**用途：** 渲染尺寸已知的预格式化 ANSI 字符串。

Props: `children: string, width: number, height: number`

Renders to `ink-raw-ansi` element with `rawWidth`/`rawHeight` attributes. The yoga measure function reads these dimensions directly (no string width measurement, no wrapping).

### `/x/Bigger-Projects/Claude-Code/src/ink/components/NoSelect.tsx`

**用途：** 把一个矩形区域标记为不可选（例如 gutter、行号）。

Sets the `noSelect` flag on cells in the screen buffer via `markNoSelectRegion`. Text in these cells is excluded from selection copy and search highlighting.

### `/x/Bigger-Projects/Claude-Code/src/ink/components/Newline.tsx`

**用途：** 渲染 `\n` 字符（数量可通过 `count` prop 配置）。

### `/x/Bigger-Projects/Claude-Code/src/ink/components/Spacer.tsx`

**用途：** 灵活占位符，渲染一个 `flexGrow: 1` 的 `Box`。

### `/x/Bigger-Projects/Claude-Code/src/ink/components/Button.tsx`

**用途：** 可聚焦、可点击、支持键盘激活的按钮。

**`ButtonState`:** `'idle' | 'active' | 'focus'`

**Props:** `children, onPress?, disabled?` plus styling.

Uses `useApp` for exit integration. Handles Return/Space keydown when focused.

### `/x/Bigger-Projects/Claude-Code/src/ink/components/ErrorOverview.tsx`

**用途：** 当 React 组件抛错时显示的错误边界覆盖层。会把堆栈和错误消息渲染成带格式的内容。

### Context 组件：

**`AppContext.ts`** — provides `{ exit(error?): void }`. Consumed by `useApp`.

**`StdinContext.ts`** — provides `{ stdin, setRawMode, isRawModeSupported, internal_exitOnCtrlC, internal_eventEmitter }`. Consumed by `useStdin`, `useInput`.

**`TerminalSizeContext.tsx`** — provides `{ columns: number; rows: number } | null`. Consumed by `useTerminalViewport`, `AlternateScreen`.

**`ClockContext.tsx`** — provides a shared animation clock with `subscribe(cb, keepAlive?)` and `now()`. All `useAnimationFrame` instances share one clock; idle clock (no `keepAlive` subscribers) suspends to avoid waking the process.

**`CursorDeclarationContext.ts`** — provides a setter for declaring native cursor position. Consumed by `useDeclaredCursor`. The setter signature: `(decl: CursorDeclaration | null, node?: DOMElement | null) => void`.

**`TerminalFocusContext.tsx`** — provides `useTerminalFocus()` which subscribes to DECSET 1004 focus events via `useSyncExternalStore` on `terminal-focus-state.ts`.

---

## Hooks

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-input.ts`

**`useInput(handler, options?)`:** Subscribes to stdin input events. Calls `setRawMode(true)` via `useLayoutEffect` (synchronous, before render returns). Subscribes to `internal_eventEmitter` `'input'` events. Options: `{ isActive?: boolean }`.

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-app.ts`

**`default useApp()`:** Returns `{ exit(error?): void }` from `AppContext`. Throws if used outside the App tree.

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-stdin.ts`

**`default useStdin()`:** Returns the full `StdinContext` value.

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-animation-frame.ts`

**`useAnimationFrame(intervalMs?)`:** Returns `[ref, time]`. Subscribes to the shared clock and updates `time` every `intervalMs`. Pauses (unsubscribes) when `intervalMs = null` or the element is off-screen (via `useTerminalViewport`).

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-interval.ts`

**`useInterval(callback, delay?)`:** Calls `callback` every `delay` ms. Uses the shared clock.

**`useAnimationTimer(intervalMs)`:** Returns `time` (elapsed ms). Similar to `useAnimationFrame` but without the viewport visibility check.

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-terminal-viewport.ts`

**`useTerminalViewport()`:** Returns `[ref, { isVisible }]`. Computes visibility by walking the DOM ancestor chain (including `scrollTop` offsets) during `useLayoutEffect`. Does NOT cause re-renders on visibility change — callers read the current value naturally.

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-terminal-focus.ts`

**`useTerminalFocus()`:** Returns `boolean` — whether the terminal window is focused. Uses `useSyncExternalStore` on `terminal-focus-state.ts` module-level signal.

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-terminal-title.ts`

**`useTerminalTitle(title)`:** Sets the terminal window title via OSC 0/2 on mount and clears on unmount.

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-selection.ts`

**`useSelection()`:** Returns an API object for text selection operations. Falls back to no-ops when not in fullscreen mode. The `Ink` instance is located via `instances.get(process.stdout)`.

Methods: `copySelection()`, `copySelectionNoClear()`, `clearSelection()`, `hasSelection()`, `getState()`, `subscribe(cb)`, `shiftAnchor(dRow, min, max)`, `shiftSelection(dRow, min, max)`, `moveFocus(move)`, `captureScrolledRows(first, last, side)`, `setSelectionBgColor(color)`.

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-tab-status.ts`

**`useTabStatus(kind: TabStatusKind | null)`:** Emits OSC 21337 tab status sequences. `kind = 'idle' | 'busy' | 'waiting'`. Transitions to `null` emit `CLEAR_TAB_STATUS`. Wrapped for tmux passthrough. Uses `TerminalWriteContext`.

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-declared-cursor.ts`

**`useDeclaredCursor({ line, column, active })`:** Returns a ref callback. When active, declares the cursor position to the `Ink` instance so the native cursor parks at the text caret. Uses `useLayoutEffect` (no dep array — re-declares every commit) for correct sibling handoff. Clears on unmount via a separate `useLayoutEffect` with empty deps.

### `/x/Bigger-Projects/Claude-Code/src/ink/hooks/use-search-highlight.ts`

Internal hook for wiring search query to the Ink instance's `setSearchHighlight` / `setSearchPositions`.

---

## 工具模块

### `/x/Bigger-Projects/Claude-Code/src/ink/Ansi.tsx`

**`Ansi` component:** Parses ANSI escape codes in a string and renders them using `Text` and `Link` components. Memoized. Accepts `children: string` and optional `dimColor: boolean`. Uses `termio.Parser` to extract spans and maps them to `Text` props + `Link` wrappers for hyperlinks.

### `/x/Bigger-Projects/Claude-Code/src/ink/bidi.ts`

**`reorderBidi(characters: ClusteredChar[])`:** Applies the Unicode Bidi Algorithm to a `ClusteredChar` array when running on Windows Terminal, WSL, or xterm.js (all lack native bidi). Uses `bidi-js` library. Detects need via `WT_SESSION` env var or `TERM_PROGRAM=vscode`. No-op on platforms with native bidi support.

### `/x/Bigger-Projects/Claude-Code/src/ink/clearTerminal.ts`

**`getClearTerminalSequence()`:** Returns `ERASE_SCREEN + ERASE_SCROLLBACK + CURSOR_HOME` on modern terminals. Windows: uses HVP (`ESC [ 0 f`) for cursor home on legacy console; includes scrollback clear for Windows Terminal, VS Code, and mintty.

**`clearTerminal`:** Pre-computed clear sequence (module load time).

### `/x/Bigger-Projects/Claude-Code/src/ink/colorize.ts`

**`colorize(text, styles)`:** Applies chalk-based color/style transforms to a text string. Detects the chalk level and adjusts for xterm.js and tmux environments.

**`applyTextStyles(text, textStyles)`:** Converts `TextStyles` to chalk method chain calls.

**Color level management:**
- `boostChalkLevelForXtermJs()` — upgrades chalk to level 3 (truecolor) when `TERM_PROGRAM=vscode` and chalk detected level 2
- `clampChalkLevelForTmux()` — downgrades to level 2 (256-color) inside tmux to avoid truecolor passthrough bugs; skipped when `CLAUDE_CODE_TMUX_TRUECOLOR=1`

### `/x/Bigger-Projects/Claude-Code/src/ink/get-max-width.ts`

**`getMaxWidth(node, offsetWidth?)`:** Computes the available render width for a node accounting for padding and border. Reads from `yogaNode.getComputedPadding`/`getComputedBorder` for each relevant edge.

### `/x/Bigger-Projects/Claude-Code/src/ink/line-width-cache.ts`

**`lineWidth(line: string)`:** Memoized string width for individual lines (no newlines). Cache is a `Map<string, number>`. Used by `measureText`.

### `/x/Bigger-Projects/Claude-Code/src/ink/measure-element.ts`

**`measureElement(node: DOMElement)`:** Returns `{ width, height }` by reading `yogaNode.getComputedWidth()/getComputedHeight()`. Throws if the node has no yoga node.

### `/x/Bigger-Projects/Claude-Code/src/ink/measure-text.ts`

**`measureText(text, maxWidth)`:** Single-pass computation of `{ width, height }` for a text string. Uses `lineWidth` per line. Height = sum of `ceil(lineWidth / maxWidth)` per line (or 1 when `noWrap`).

### `/x/Bigger-Projects/Claude-Code/src/ink/squash-text-nodes.ts`

**`squashTextNodes(node)`:** Concatenates all text content of a node tree into a plain string (no styles). Used by `measureTextNode` in `dom.ts`.

**`squashTextNodesToSegments(node, inheritedStyles?, inheritedHyperlink?, out?)`:** Walks the text node tree and produces `StyledSegment[]` — text with inherited styles and hyperlink URLs. Used by `output.ts` for structured rendering.

**`StyledSegment` type:** `{ text: string; styles: TextStyles; hyperlink?: string }`

### `/x/Bigger-Projects/Claude-Code/src/ink/stringWidth.ts`

**`stringWidth(str)`:** Terminal display width of a string (counts wide chars as 2, strips ANSI codes). Uses `@alcalzone/ansi-tokenize` + grapheme segmentation.

### `/x/Bigger-Projects/Claude-Code/src/ink/widest-line.ts`

**`widestLine(text)`:** Returns the display width of the widest line in a multi-line string.

### `/x/Bigger-Projects/Claude-Code/src/ink/wrap-text.ts`

**`wrapText(text, maxWidth, wrapType)`:** Applies the appropriate text wrap strategy:
- `'wrap'`: `wrapAnsi(text, maxWidth, { trim: false, hard: true })`
- `'wrap-trim'`: `wrapAnsi(text, maxWidth, { trim: true, hard: true })`
- `'truncate-end'` / `'truncate'`: append `…`
- `'truncate-middle'`: insert `…` in middle
- `'truncate-start'`: prepend `…`
- `'end'` / `'middle'`: same as corresponding truncate

Uses `sliceFit` to avoid wide-char boundary errors in slice operations.

### `/x/Bigger-Projects/Claude-Code/src/ink/wrapAnsi.ts`

Custom ANSI-aware word-wrap implementation. Handles wide chars, hyperlinks (OSC 8), and soft-wrap tracking. Returns wrapped text plus `softWrap[]` flags.

### `/x/Bigger-Projects/Claude-Code/src/ink/supports-hyperlinks.ts`

**`supportsHyperlinks()`:** Detects terminal support for OSC 8 hyperlinks. Returns `true` for iTerm2, VS Code, kitty, Ghostty, WezTerm, Warp, and others.

### `/x/Bigger-Projects/Claude-Code/src/ink/tabstops.ts`

**`expandTabs(text, startColumn?)`:** Expands tab characters to spaces based on 8-column tab stops. Used during text measurement (worst-case width). Actual tab expansion at render time uses the screen x-position.

### `/x/Bigger-Projects/Claude-Code/src/ink/render-border.ts`

**`renderBorder(node, output, x, y, width, height)`:** Draws box borders using `cli-boxes` glyphs (single, double, round, bold, classic, dashed). Supports `borderText` option to embed text into top/bottom border lines with start/end/center alignment. Colors applied via `chalk`.

### `/x/Bigger-Projects/Claude-Code/src/ink/terminal-focus-state.ts`

Module-level singleton signal for terminal focus state.

**`TerminalFocusState`:** `'focused' | 'blurred' | 'unknown'`

**导出：**
- `setTerminalFocused(v)` — updates state, notifies `useSyncExternalStore` subscribers
- `getTerminalFocused()` — returns `focusState !== 'blurred'` (unknown treated as focused)
- `getTerminalFocusState()` — returns the tristate value
- `subscribeTerminalFocus(cb)` — subscribe function for `useSyncExternalStore`
- `resetTerminalFocusState()` — resets to `'unknown'`

### `/x/Bigger-Projects/Claude-Code/src/ink/terminal-querier.ts`

**用途：** 使用 DA1/DECRQM/XTVERSION 哨兵协议查询终端能力信息（不设超时，因为 DA1 是通用哨兵）。

**`TerminalQuery<T>` type:** `{ request: string; match: (r) => r is T }`

**Query builders:**
- `decrqm(mode)` — DECRQM query; response: `DecrpmResponse`
- `da1()` — Primary Device Attributes
- `da2()` — Secondary Device Attributes
- `kittyKeyboard()` — Kitty keyboard flags query
- `cursorPosition()` — DECXCPR
- `oscColor(code, index?)` — OSC 10/11/12 color queries
- `xtversion()` — DCS `>|` terminal name/version

**`TerminalQuerier` class:**
- `send<T>(query)` — returns a `Promise<T | undefined>`
- `flush()` — sends a DA1 sentinel; all pending queries resolve when DA1 response arrives (terminals answer in order)
- Internal: holds a queue of `{ query, resolve }` entries

**`xtversion`:** Stores the XTVERSION name (set by App.tsx after the query resolves; read by `isXtermJs()`).

### `/x/Bigger-Projects/Claude-Code/src/ink/useTerminalNotification.ts`

**`TerminalWriteContext`:** React context providing a `(data: string) => void` write function that bypasses the normal Ink render pipeline (direct stdout write).

**`TerminalWriteProvider`:** `= TerminalWriteContext.Provider`

**`useTerminalNotification()`:** Returns notification methods:
- `notifyITerm2({ message, title? })` — OSC 9 iTerm2 notification
- `notifyKitty({ message, title, id })` — Kitty notification via OSC 99
- `notifyGhostty({ message, title })` — Ghostty notification via OSC 99 variant
- `notifyBell()` — raw BEL character
- `progress(state, percentage?)` — OSC 9;4 progress bar (Ghostty 1.2+, iTerm2 3.6.6+, ConEmu)

### `/x/Bigger-Projects/Claude-Code/src/ink/warn.ts`

Centralized warning emitter (wraps `console.warn` with deduplication). Used by `Box.tsx` for invalid prop combinations.

---

## 架构：完整流水线

```
[stdin bytes]
    → App.tsx handleInput()
    → parseMultipleKeypresses()  (termio tokenizer + key parser)
    → KeyboardEvent / ParsedMouse / TerminalResponse
    → dispatcher.dispatchDiscrete(rootNode, event)  // keyboard
    → React setState / component handlers

[React state change]
    → React reconciler commit
    → reconciler.resetAfterCommit(rootNode)
    → rootNode.onComputeLayout()  // yoga calculateLayout()
    → rootNode.onRender()  →  queueMicrotask(ink.onRender)

[ink.onRender()]
    → createRenderer(rootNode, stylePool)(options)
        → renderNodeToOutput(rootNode, output, { prevScreen })
            → DOM walk with blit/clip/write/scroll ops
            → output.get() → Screen (cell buffer)
        → return Frame { screen, viewport, cursor, scrollHint }
    → applySelectionOverlay(selection, frame.screen, stylePool)
    → applySearchHighlight(frame.screen, query, stylePool)
    → applyPositionedHighlight(frame.screen, positions, ...)
    → log.render(prevFrame, frame) → Diff (Patch[])
    → optimize(diff) → compressed Patch[]
    → writeDiffToTerminal(terminal, diff, syncOutput)
        → BSU (if sync supported)
        → for each Patch: write ANSI to stdout
        → ESU (if sync supported)
    → emit onFrame(FrameEvent)
    → swap frontFrame ↔ backFrame
```

### 双缓冲

The `Ink` class maintains `frontFrame` (the last displayed frame) and `backFrame` (the rendering target). After each render:
- `backFrame.screen` contains the newly rendered content
- It becomes the new `frontFrame`
- Old `frontFrame` becomes the new `backFrame` for the next render

The renderer always reads `prevScreen = frontFrame.screen` for blit operations and writes into `backFrame.screen`.

### Blit 优化

The blit path is the dominant fast path for steady-state rendering:
1. If a node's `dirty = false` AND `nodeCache` has a valid rect AND `prevScreen` is available AND no layout shift occurred AND no removed children: call `blitRegion(backScreen, prevScreen, cachedRect)` — pure `Int32Array.copyWithin`, O(cells).
2. This means spinner ticks and clock updates only re-render the changed cell ranges; the rest of the screen is copied in bulk.

### 同步输出（BSU/ESU）

When `SYNC_OUTPUT_SUPPORTED = true`, each frame is wrapped in DEC mode 2026 begin/end synchronized update sequences (`BSU` / `ESU`). This prevents terminals from rendering intermediate states during the diff write. Supported terminals: iTerm2, WezTerm, Warp, ghostty, kitty, VS Code, alacritty, foot, kitty.

### DECSTBM 硬件滚动

When a `ScrollBox`'s `scrollTop` changes and layout is otherwise stable, `render-node-to-output` sets a `ScrollHint`. `LogUpdate.render()` checks for this hint and emits:
1. `setScrollRegion(top, bottom)` — DECSTBM restricts scroll to the box's viewport
2. `SU(n)` or `SD(n)` — hardware scroll by n rows
3. `RESET_SCROLL_REGION` — restore full-screen scroll
4. Then only re-renders the narrow band of newly exposed cells

This replaces O(viewport × width) cell writes with O(exposed_rows × width) for smooth scrolling.

### 池的代际重置

The `StylePool` and `CharPool` live for the entire session (never reset) to ensure stable IDs for the blit optimization (IDs are comparable as integers across frames). The `HyperlinkPool` is reset every 5 minutes (hyperlinks are ephemeral) via `migrateScreenPools()`, which re-interns all active cells into fresh pool instances.
