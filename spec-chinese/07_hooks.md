# Claude Code — React Hooks

本文覆盖 `src/hooks/`、`src/hooks/toolPermission/` 和 `src/hooks/notifs/` 中的每一个 hook。每个条目都会说明用途、参数/props、返回值、关键逻辑和依赖。

---

## 目录

1. [核心 / 工具 Hooks](#核心--工具-hooks)
2. [输入与文本编辑 Hooks](#输入与文本编辑-hooks)
3. [权限与工具使用 Hooks](#权限与工具使用-hooks)
4. [Swarm / Teammate Hooks](#swarm--teammate-hooks)
5. [IDE 集成 Hooks](#ide-集成-hooks)
6. [远程与会话 Hooks](#远程与会话-hooks)
7. [插件与建议 Hooks](#插件与建议-hooks)
8. [通知 Hooks (`notifs/`)](#通知-hooks-notifs)
9. [工具权限子系统 (`toolPermission/`)](#工具权限子系统-toolpermission)
10. [`hooks/` 里的非 Hook 工具](#hooks-里的非-hook-工具)

---

## 核心 / 工具 Hooks

### `useAfterFirstRender`

**文件：** `hooks/useAfterFirstRender.ts`

**用途：** ANT 内部的启动时间测量 hook。首次渲染后会把启动时间写到 stderr；如果环境变量 `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` 设了值，还会调用 `process.exit(0)`。

**参数：** 无

**返回值：** `void`

**关键逻辑：**
- 读取环境变量 `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER`
- 使用 `[]` 依赖的 `useEffect`，在第一次 commit 后触发
- 从 `MACRO.STARTUP_TIMESTAMP` 计算耗时，把结果写到 `process.stderr`，然后退出

**依赖：** `useEffect`（React）

---

### `useApiKeyVerification`

**文件：** `hooks/useApiKeyVerification.ts`

**用途：** 管理 API key 验证的完整生命周期，包括 loading、valid、invalid、missing 和 error，并暴露 `reverify` 回调。它还会在信任对话框关闭前阻止 `apiKeyHelper` 脚本运行，防止 RCE。

**参数：** 无

**返回值：** `ApiKeyVerificationResult` - `{ status: 'loading'|'valid'|'invalid'|'missing'|'error', reverify: () => void, errorMessage?: string }`

**关键逻辑：**
- 订阅 `AppState` 里的 `trustDialogAccepted` 和 `apiKeyVerificationStatus`
- 在挂载时，以及每次调用 `reverify` 后，用 `useEffect` 执行验证
- 在信任对话框出现前跳过 `apiKeyHelper` 进程
- 通过 `useCallback` 返回稳定的 `reverify` 回调

**依赖：** `useAppState`、`useSetAppState`、`useCallback`、`useEffect`

---

### `useBlink`

**文件：** `hooks/useBlink.ts`

**用途：** 返回一个和 animation-frame 时钟同步的闪烁布尔值。终端失焦或组件处于 offscreen（OffscreenFreeze）时会暂停。

**参数：**
- `enabled: boolean` - 为 false 时始终返回 `true`（光标一直可见）
- `intervalMs?: number` - 闪烁周期，单位毫秒，默认 530

**返回值：** `[ref: RefObject<unknown>, isVisible: boolean]`

**关键逻辑：**
- 使用 `useAnimationFrame`（Ink hook）读取共享时钟计数器
- 用 `intervalMs / frameMs` 做分割，在偶数/奇数之间切换
- 返回 Ink 的 `useOffscreenFreeze` 提供的同一个 ref，用于在不可见时暂停

**依赖：** `useAnimationFrame`（ink）、`useTerminalFocus`（ink）、`useRef`、`useMemo`

---

### `useCommandQueue`

**文件：** `hooks/useCommandQueue.ts`

**用途：** 把统一命令队列暴露成一个响应式数组。任何组件都可以订阅它，而不必手写外部 store 订阅。

**参数：** 无

**返回值：** `readonly QueuedCommand[]`

**关键逻辑：**
- 在 `messageQueueManager` 的 subscribe/getSnapshot 之上封装 `useSyncExternalStore`
- 只有队列引用变化时才重渲染，而不是每次 push 都重渲染

**依赖：** `useSyncExternalStore`（React）、`messageQueueManager`

---

### `useCopyOnSelect`

**文件：** `hooks/useCopyOnSelect.ts`

**用途：** 用户松开鼠标（mouseup）或双击/三击时，把选中的文本自动复制到剪贴板。这个文件还导出 `useSelectionBgColor`，用于 selected text 的主题颜色。

**参数：**
- `selection: SelectionState` - 当前 Ink 选择状态
- `isActive: boolean` - 只有为 true 时才启用
- `onCopied?: () => void` - 剪贴板写入完成后的回调

**返回值：** `void`

**关键逻辑：**
- 通过 `useEffect` 订阅 Ink 鼠标事件
- 在 mouseup 时，如果 selection 非空且 `isActive`，就调用 `navigator.clipboard.writeText`
- `useSelectionBgColor()` 会读取 AppState 主题，返回正确的高亮颜色

**依赖：** `useEffect`、`useAppState`、`useCallback`

---

### `useDoublePress`

**文件：** `hooks/useDoublePress.ts`

**用途：** 返回一个在 800ms 时间窗内检测“双击按键”的回调。用于 Ctrl+C/D 退出和双击 Escape 清空输入。

**参数：**
- `setPending: (show: boolean) => void` - 首次按下后传 `true`，超时后传 `false`
- `onDoublePress: () => void` - 在时间窗内第二次按下时调用
- `onFirstPress?: () => void` - 首次按下时可选的副作用

**返回值：** `() => void` - 包装后的按键处理器

**关键逻辑：**
- 用 ref 记录 `lastPressTime`
- 调用时如果距离上次按下小于 800ms，就调用 `onDoublePress` 并重置；否则调用 `setPending(true)`，设置一个 800ms 定时器来恢复 `setPending(false)`，并在需要时调用 `onFirstPress`

**依赖：** `useRef`、`useCallback`

---

### `useElapsedTime`

**文件：** `hooks/useElapsedTime.ts`

**用途：** 计算可读的 elapsed time 字符串，比如 `"1m 23s"`。任务运行时会持续更新，结束后冻结。

**参数：**
- `startTime: number` - 开始计时的 Unix 时间戳（毫秒）
- `isRunning: boolean` - 为 false 时冻结 elapsed
- `ms?: number` - 更新间隔，默认 1000ms
- `pausedMs?: number` - 已累计暂停时间，计算时会减掉
- `endTime?: number` - 如果提供，会冻结在这个时间戳

**返回值：** `string` - 格式化后的 elapsed time，如 `"5s"`、`"1m 23s"`、`"2h 5m"`

**关键逻辑：**
- 使用 `useSyncExternalStore` 监听基于计时器的外部时钟
- 时钟每 `ms` 通过 `setInterval` 更新一次；每个订阅者在下一个 tick 前都能拿到稳定快照
- 用 `formatDuration` 格式化时间差

**依赖：** `useSyncExternalStore`、`useRef`

---

### `useExitOnCtrlCD`

**文件：** `hooks/useExitOnCtrlCD.ts`

**用途：** 实现双击 Ctrl+C / Ctrl+D 退出。它会返回 pending 状态，调用方可以显示“再按一次退出”的提示。

**参数：**
- `useKeybindingsHook: (bindings: ...) => void` - 可注入的绑定 hook
- `onInterrupt?: () => void` - 首次 Ctrl+C 时调用
- `onExit?: () => void` - 第二次按下时调用
- `isActive?: boolean` - 开启或关闭这个处理器

**返回值：** `ExitState` - `{ pending: boolean, keyName: string | null }`

**关键逻辑：**
- 内部对 Ctrl+C 和 Ctrl+D 都使用 `useDoublePress`
- 首次按下后把 `pending = true`；超时或第二次按下后清除
- `keyName` 记录按的是哪个键（`'Ctrl-C'` 或 `'Ctrl-D'`）

**依赖：** `useDoublePress`、`useState`、`useCallback`

---

### `useExitOnCtrlCDWithKeybindings`

**文件：** `hooks/useExitOnCtrlCDWithKeybindings.ts`

**用途：** 把 `useExitOnCtrlCD` 接到 keybinding 系统上的便捷封装。

**参数：**
- `onExit?: () => void`
- `onInterrupt?: () => void`
- `isActive?: boolean`

**返回值：** `ExitState`

**关键逻辑：** 把 `useKeybindings` 作为 hook 参数传给 `useExitOnCtrlCD`。

**依赖：** `useExitOnCtrlCD`、`useKeybindings`

---

### `useMemoryUsage`

**文件：** `hooks/useMemoryUsage.ts`

**用途：** 每 10 秒轮询一次 Node.js `process.memoryUsage().heapUsed`，在内存过高或关键时返回状态。

**参数：** 无

**返回值：** `MemoryUsageInfo | null` - 正常时返回 `null`；当 heap 超过 1.5 GB（high）或 2.5 GB（critical）时返回 `{ heapUsed: number, status: 'high' | 'critical' }`

**关键逻辑：**
- 使用 `useInterval`（usehooks-ts），周期 10 000ms
- 阈值：`HIGH_HEAP_MB = 1536`、`CRITICAL_HEAP_MB = 2560`
- 低于阈值时返回 `null`

**依赖：** `useInterval`、`useState`、`useEffect`

---

### `useMinDisplayTime`

**文件：** `hooks/useMinDisplayTime.ts`

**用途：** 通过保证每个值至少显示 `minMs` 毫秒，减少 UI 闪烁。

**参数：**
- `value: T` - 要显示的值
- `minMs: number` - 最小显示时长

**返回值：** `T` - “稳定”显示值，可能比 `value` 落后

**关键逻辑：**
- 用 `useRef` 记录当前稳定值和设置时间
- `value` 变化时，如果 `Date.now() - lastChanged >= minMs`，就立刻更新；否则设置一个 `setTimeout`，等剩余时间到了再更新

**依赖：** `useState`、`useRef`、`useEffect`

---

### `useNotifyAfterTimeout`

**文件：** `hooks/useNotifyAfterTimeout.ts`

**用途：** 在用户 6 秒没有操作后发送桌面（系统级）通知，用来提醒 Claude 已经在无人看管地工作。

**参数：**
- `message: string` - 通知正文
- `notificationType: string` - 事件类型标识，用于 analytics

**返回值：** `void`

**关键逻辑：**
- 挂载后等待 6 000ms，再检查终端是否失焦
- 只有终端不在焦点时才触发
- 调用原生模块里的 `sendDesktopNotification`

**依赖：** `useEffect`、`useRef`

---

### `useTimeout`

**文件：** `hooks/useTimeout.ts`

**用途：** 返回一个在 `delay` 毫秒后变成 `true` 的布尔值。`resetTrigger` 变化时会重置计时器。

**参数：**
- `delay: number` - 等待毫秒数
- `resetTrigger?: number` - 这个值变化会重置计时器

**返回值：** `boolean` - 延迟到期前为 `false`，之后为 `true`

**关键逻辑：** 一个简单的 `useState` + `useEffect` + `setTimeout`。清理函数会在重跑或卸载时清掉定时器。

**依赖：** `useState`、`useEffect`

---

### `useSettings`

**文件：** `hooks/useSettings.ts`

**用途：** 从全局 AppState 读取当前 settings。它是响应式的，设置变化时会重渲染（例如文件监听器触发）。

**参数：** 无

**返回值：** `ReadonlySettings`

**关键逻辑：** 直接返回 `useAppState(s => s.settings)`。

**依赖：** `useAppState`

---

### `useSettingsChange`

**文件：** `hooks/useSettingsChange.ts`

**用途：** 订阅 settings 变更检测器；每当磁盘上的 settings 文件被修改，就会用新的 settings 和变更来源调用 `onChange`。

**参数：**
- `onChange: (source: string, settings: Settings) => void`

**返回值：** `void`

**关键逻辑：**
- 用 `useEffect` 订阅 `settingsChangeDetector.subscribe(onChange)`
- 返回取消订阅函数作为 cleanup

**依赖：** `useEffect`、`settingsChangeDetector`

---

### `useDeferredHookMessages`

**文件：** `hooks/useDeferredHookMessages.ts`

**用途：** 在挂载时异步把 `SessionStart` hook 消息插入消息列表，避免阻塞首屏渲染。

**参数：**
- `pendingHookMessages: Message[]` - 由 session-start hooks 生成的消息
- `setMessages: SetMessages` - 消息列表更新器

**返回值：** `() => Promise<void>` - 一个稳定的 async 回调，用来触发注入

**关键逻辑：**
- 通过 `setTimeout(0)` 延后，等第一次渲染完成后再注入 hook 消息
- 用 `useRef` 避免 stale closure 问题

**依赖：** `useRef`、`useCallback`

---

### `useDiffData`

**文件：** `hooks/useDiffData.ts`

**用途：** 在挂载时获取当前 git diff 的统计信息和 hunks（供 `/diff` 命令视图使用）。

**参数：** 无

**返回值：** `DiffData` - `{ stats: DiffStats, files: string[], hunks: DiffHunk[], loading: boolean }`

**关键逻辑：**
- 挂载后调用 `getGitDiff()`，它会在 cwd 里跑 `git diff`
- 异步获取完成前一直保持 `loading: true`

**依赖：** `useState`、`useEffect`

---

### `useFileHistorySnapshotInit`

**文件：** `hooks/useFileHistorySnapshotInit.ts`

**用途：** 从保存在 conversation log 里的 snapshot 数据里，一次性初始化 file history 状态，恢复 `/resume` 之间的文件时间戳。

**参数：**
- `initialFileHistorySnapshots: FileHistorySnapshot[]`
- `fileHistoryState: FileHistoryState`
- `onUpdateState: (state: FileHistoryState) => void`

**返回值：** `void`

**关键逻辑：**
- 用 `useEffect` 的 `[]` 依赖只运行一次
- 把 `initialFileHistorySnapshots` 合并进 `fileHistoryState`，但不会覆盖更新的条目

**依赖：** `useEffect`

---

### `useInputBuffer`

**文件：** `hooks/useInputBuffer.ts`

**用途：** 提供一个带 debounce 的撤销缓冲区，用于文本输入，实现“撤销上一次粘贴”或“撤销上一次编辑”。

**参数：**
- `maxBufferSize: number` - 保留条目的最大数量
- `debounceMs: number` - 等待多长时间后把当前值提交进缓冲区

**返回值：** `UseInputBufferResult` - `{ pushToBuffer, undo, canUndo, clearBuffer }`

**关键逻辑：**
- 用 ref 维护一个 `string[]` 撤销栈
- `pushToBuffer` 是 debounced 的：短时间内多次变化会合并成一个缓冲条目
- `undo` 会弹出栈，并用前一个值调用 `onChange`

**依赖：** `useRef`、`useCallback`、`useEffect`

---

### `useLogMessages`

**文件：** `hooks/useLogMessages.ts`

**用途：** 在每次渲染后增量地把消息写到 conversation transcript 文件（`.jsonl`）里，而不是每次都重写整个 transcript。

**参数：**
- `messages: readonly Message[]` - 当前消息列表
- `ignore?: boolean` - 为 true 时跳过记录

**返回值：** `void`

**关键逻辑：**
- 用 ref 追踪 `lastProcessedIndex`，只处理新增消息
- 处理边界情况：compaction（transcript 变短）、首次渲染、head-pointer rewind
- 只对新增消息调用 `recordTranscript(messages, from, to)`
- 去重 compact-summary 边界

**依赖：** `useEffect`、`useRef`

---

### `useMainLoopModel`

**文件：** `hooks/useMainLoopModel.ts`

**用途：** 返回当前会话的解析后模型名。GrowthBook flags 刷新时会重新计算，这样中途的 model alias 解析也能保持最新。

**参数：** 无

**返回值：** `ModelName`

**关键逻辑：**
- 从 AppState 读取 `settings.model`
- 通过 `useEffect` 订阅 `onGrowthBookRefresh`；每次刷新就递增计数器，强制重渲染
- 调用 `resolveModelAlias(model)` 把用户可见的 alias（比如 `opus`）转换成具体模型 ID

**依赖：** `useAppState`、`useEffect`、`useState`

---

### `useManagePlugins`

**文件：** `hooks/useManagePlugins.ts`

**用途：** 在挂载时加载插件列表，并接上插件生命周期管理：下架强制执行、MCP/LSP 插件计数，以及“需要刷新”通知。

**参数：**
- `{ enabled?: boolean }`

**返回值：** `void`

**关键逻辑：**
- 挂载时（如果启用）调用 `loadPlugins()`，并把结果写回 AppState
- 通过读取 settings 里的 `delistedPlugins` 强制删除已下架插件
- 统计 active MCP 和 LSP 插件数量，并写进 AppState，供 `/doctor` 诊断使用
- **不会**自动刷新；刷新必须通过 `/reload-plugins` 显式触发

**依赖：** `useEffect`、`useSetAppState`、`useAppState`

---

### `useMergedClients`

**文件：** `hooks/useMergedClients.ts`

**用途：** 通过 server name 去重，把两个 MCP client 列表（settings 里初始的 + 动态加载的）合并起来。

**参数：**
- `initialClients: MCPServerConnection[]`
- `mcpClients: MCPServerConnection[]`

**返回值：** `MCPServerConnection[]`

**关键逻辑：** 使用 `lodash.uniqBy([...initialClients, ...mcpClients], 'name')`。`useMemo` 的依赖是合并后列表长度和名称集合。

**依赖：** `useMemo`、`lodash.uniqBy`

---

### `useMergedCommands`

**文件：** `hooks/useMergedCommands.ts`

**用途：** 把初始命令和 MCP 侧加载的命令按命令名去重。

**参数：**
- `initialCommands: Command[]`
- `mcpCommands: Command[]`

**返回值：** `Command[]`

**关键逻辑：** 对 `uniqBy([...initialCommands, ...mcpCommands], getCommandName)` 做 `useMemo`。

**依赖：** `useMemo`

---

### `useMergedTools`

**文件：** `hooks/useMergedTools.ts`

**用途：** 把内建工具、MCP 工具和权限上下文过滤组合起来，构成会话的完整工具池。

**参数：**
- `initialTools: Tool[]`
- `mcpTools: Tool[]`
- `toolPermissionContext: ToolPermissionContext`

**返回值：** `Tools`（组装好的工具集）

**关键逻辑：**
- 调用 `assembleToolPool(initialTools, mcpTools)` 生成合并列表
- 再调用 `mergeAndFilterTools(pool, toolPermissionContext)` 去掉被禁用的工具

**依赖：** `useMemo`

---

### `useSkillsChange`

**文件：** `hooks/useSkillsChange.ts`

**用途：** 当磁盘上的 skill 文件发生变化，或者 GrowthBook flags 刷新时，保持命令列表是最新的。

**参数：**
- `cwd: string | undefined` - 扫描 skills 时使用的当前工作目录
- `onCommandsChange: (commands: Command[]) => void` - 更新命令列表的回调

**返回值：** `void`

**关键逻辑：**
- 订阅 `skillChangeDetector.subscribe(handleChange)`，在 skill 文件写入时触发
- 文件变化时：调用 `clearCommandsCache()` + `getCommands(cwd)`，然后执行 `onCommandsChange`
- 订阅 `onGrowthBookRefresh(handleGrowthBookRefresh)`；GB flag 刷新时调用 `clearCommandMemoizationCaches()` + `getCommands(cwd)`，重新评估 feature-gated 命令

**依赖：** `useEffect`、`useCallback`

---

### `useUpdateNotification`

**文件：** `hooks/useUpdateNotification.ts`

**用途：** 当自动更新下载完成时，返回新的 semver 版本字符串，用于在状态栏显示。没有新版本，或者版本和上次通知的一样时，返回 `null`。

**参数：**
- `updatedVersion: string | null | undefined` - 下载到的版本（来自 auto-updater）
- `initialVersion?: string` - 基准版本（默认 `MACRO.VERSION`）

**返回值：** `string | null`

**关键逻辑：**
- 用 `semver` 解析两个版本，提取 `major.minor.patch`
- 用 `useState` 记录上一次通知过的 semver
- 如果新 semver 和 `lastNotifiedSemver` 不同，就更新状态并返回新值（触发通知）；否则返回 `null`

**依赖：** `useState`、`semver`

---

## 输入与文本编辑 Hooks

### `useTextInput`

**文件：** `hooks/useTextInput.ts`

**用途：** 完整的 readline 风格文本输入处理器。负责光标位置、多行编辑、kill ring（Ctrl+K/U/W）、yank（Ctrl+Y / Meta+Y）、历史导航（方向键上下）和 ghost text 渲染。

**参数：** `UseTextInputProps`，包括：
- `value: string` - 当前文本值（受控）
- `onChange: (value: string) => void`
- `onSubmit?: (value: string) => void`
- `onExit?: () => void`
- `onHistoryUp / onHistoryDown / onHistoryReset / onClearInput`
- `focus?: boolean`
- `mask?: string` - 用这个字符串遮住所有字符
- `multiline?: boolean`
- `cursorChar: string`
- `columns: number` - 终端宽度，用于换行
- `externalOffset: number` - 外部控制的光标偏移
- `onOffsetChange: (offset: number) => void`
- `inputFilter?: (input: string, key: Key) => string`
- `inlineGhostText?: InlineGhostText`
- `disableCursorMovementForUpDownKeys?: boolean`
- `disableEscapeDoublePress?: boolean`
- `maxVisibleLines?: number`

**返回值：** `TextInputState` - `{ onInput, renderedValue, offset, setOffset, cursorLine, cursorColumn, viewportCharOffset, viewportCharEnd }`

**关键逻辑：**
- 使用 `utils/Cursor.js` 里的 `Cursor` 类管理文本缓冲区和位置运算
- 把按键映射成光标移动：Ctrl+A（home）、Ctrl+E（end）、Ctrl+F/B（前进/后退）、Ctrl+N/P（下一/上一行）、Meta+F/B（按单词导航）
- kill ring：Ctrl+K（删到结尾）、Ctrl+U（删到开头）、Ctrl+W（删单词）。连续 kill 会追加到 ring
- yank：Ctrl+Y 插入最后一次 kill；Meta+Y 在 ring 里循环
- 双击 Ctrl+C 会清空或退出（通过 `useDoublePress`）
- 双击 Escape 会清空输入，并显示“Esc again to clear”提示
- SSH 合并的 Enter 检测：`text\r` 形式会触发 submit
- 兼容 SSH/tmux 的原始 `\x7f` DEL 字符
- 当 `inlineGhostText.insertPosition === offset` 时，在光标位置渲染 inline ghost text

**依赖：** `useDoublePress`、`useNotifications`、`Cursor` 类、`useCallback`

---

### `useVimInput`

**文件：** `hooks/useVimInput.ts`

**用途：** 在 `useTextInput` 之上扩展完整 Vim normal/insert 模式状态机，包括 operator（d、c、y）、motion、dot-repeat、find（f/F/t/T）、text objects（iw、aw 等）和 yank register。

**参数：** `UseVimInputProps`，和 `UseTextInputProps` 一样，外加：
- `onModeChange?: (mode: VimMode) => void`
- `onUndo?: () => void`

**返回值：** `VimInputState` - 在 `TextInputState` 之上增加 `{ mode: VimMode, setMode }`

**关键逻辑：**
- 在 INSERT 模式下，先跑 `inputFilter`，再把按键交给 `useTextInput`
- 在 NORMAL 模式下，把按键通过 `vim/transitions.ts` 里的 `transition(state.command, input, ctx)` 分发
- 管理 `vimStateRef`（当前模式 + 待处理命令累加器）和 `persistentRef`（register、lastFind、lastChange，用于 dot-repeat）
- INSERT 中按 Escape：调用 `switchToNormalMode()`，光标左移一格
- NORMAL 模式下的箭头键：映射到 h/j/k/l motion
- NORMAL 空闲状态下按 `?`：向输入里写入 `?`，进入 `/` 搜索
- `setModeExternal` 允许外部程序化切换模式（`/vim` 命令会用到）

**依赖：** `useTextInput`、`useState`、`useRef`、`useCallback`、vim operators/transitions

---

### `useSearchInput`

**文件：** `hooks/useSearchInput.ts`

**用途：** 用于搜索框的完整 readline 风格文本输入（历史搜索、全局搜索）。包含 kill ring、yank 和按词导航。

**参数：** `UseSearchInputOptions` - `{ initialValue?, onKeyDown?, placeholder? }`

**返回值：** `{ query: string, setQuery, cursorOffset: number, handleKeyDown: (key: Key, input: string) => void }`

**关键逻辑：**
- 实现和 `useTextInput` 一样的 Ctrl 键映射，但把它做成独立 reducer，不依赖 React state 管 cursor offset
- 用在 `HistorySearchInput` 和 `GlobalSearchDialog`

**依赖：** `useState`、`useCallback`、`useRef`、kill ring 工具

---

### `useArrowKeyHistory`

**文件：** `hooks/useArrowKeyHistory.tsx`

**用途：** 用方向键在输入历史里导航，支持懒加载分块、按模式过滤和草稿保留。

**参数：**
- `onSetInput: (value: string) => void`
- `currentInput: string`
- `pastedContents: string[]` - 通过粘贴检测得到、需要从历史匹配里排除的内容
- `setCursorOffset?: (offset: number) => void`
- `currentMode?: PromptInputMode`

**返回值：** `{ handleHistoryUp, handleHistoryDown, handleHistoryReset }`

**关键逻辑：**
- 从 `getHistory()` 里按 50 条一批懒加载
- 用 index 指针导航；第一次按 Up 时会保存当前草稿
- 按 mode 过滤条目（例如 bash mode 只返回 bash 历史）
- 第一次使用时会显示 “Search history: Ctrl+R” 的提示通知
- 当 `currentInput` 外部变化时重置 index

**依赖：** `useState`、`useRef`、`useCallback`、`useNotifications`

---

### `useHistorySearch`

**文件：** `hooks/useHistorySearch.ts`

**用途：** 实现 `Ctrl+R` 向后增量历史搜索，包括 query 匹配和键盘导航。

**参数：**
- `onSetInput: (value: string) => void`
- `currentInput: string`
- 以及一些 keybinding 选项

**返回值：** `{ historyQuery, setHistoryQuery, historyMatch, historyFailedMatch, handleKeyDown }`

**关键逻辑：**
- 注册 `history:search` keybinding（Ctrl+R）来激活搜索模式
- 在搜索模式下，注册 `historySearch:*` bindings（Enter 确认、Escape 取消、上下切换匹配）
- 按 `historyQuery` 的子串匹配过滤历史条目
- `historyFailedMatch: boolean` - 当 query 有文本但没有匹配时为 true

**依赖：** `useKeybinding`、`useKeybindings`、`useState`、`useCallback`、`useEffect`

---

### `useTypeahead`

**文件：** `hooks/useTypeahead.tsx`

**用途：** prompt input 的主 typeahead / autocomplete 引擎。通过去抖的模糊匹配、shell completion 和 MCP resources，处理 `@file`、`/command`、`#channel` 和目录建议。

**参数：** 一个很大的 props 对象，包括：
- `inputValue: string`、`cursorOffset: number`
- `commands: Command[]`、`agents: AgentDefinition[]`
- `mcpResources: MCPResource[]`
- `isLoading: boolean`
- `onSelect: (value: string) => void`
- `onToggleVisible: (show: boolean) => void`

**返回值：** `{ suggestions, selectedIndex, handleKeyDown, isSuggesting, suggestionType, ... }`

**关键逻辑：**
- 从输入里检测 suggestion 上下文：`@token` 触发文件/resource/agent 建议；`/` 触发命令建议；`#channel` 触发 Slack channel 建议（如果有 Slack MCP）
- 文件建议使用 `generateUnifiedSuggestions`（nucleo + Fuse.js 排序）
- 命令建议使用 `generateCommandSuggestions`，并生成参数提示
- shell completion 使用 `getShellCompletions`（bash/zsh）
- 路径 completion 使用 `getPathCompletions` / `getDirectoryCompletions`
- 通过 `useRegisterOverlay` 注册为 overlay，这样 Escape/Enter/方向键会被捕获
- 用 `useDebounceCallback`（usehooks-ts）限制文件查找频率
- 内部追踪键盘导航状态（`selectedIndex`）
- `/resume` 查询会通过 `searchSessionsByCustomTitle` 给出恢复建议

**依赖：** `useInput`（ink，向后兼容桥接）、`useRegisterOverlay`、`useKeybindings`、`useDebounceCallback`、`useState`、`useRef`、`useMemo`、`useCallback`、`useEffect`、`generateUnifiedSuggestions`、`generateCommandSuggestions`、`getShellCompletions`

---

### `usePasteHandler`

**文件：** `hooks/usePasteHandler.ts`

**用途：** 处理 bracketed paste 模式检测、大粘贴分块、图片文件路径检测，以及 macOS 剪贴板图片兜底。

**参数：**
- `{ onPaste: (text: string) => void, onInput: (text: string, key: Key) => void, onImagePaste?: (base64: string, ...) => void }`

**返回值：** `{ wrappedOnInput, pasteState, isPasting }`

**关键逻辑：**
- 通过 `\x1b[?2004h` / `\x1b[200~` / `\x1b[201~` escape 序列检测 bracketed paste
- 把大粘贴（>1000 字符）拆成多个 `onPaste` 调用，避免阻塞事件循环
- 检测粘贴内容里的图片文件路径（`.png`、`.jpg`、`.gif` 等），并触发 `onImagePaste`
- 在 macOS 上：如果是图片剪贴板内容，会回退到 `pbpaste`

**依赖：** `useState`、`useRef`、`useCallback`、`useEffect`

---

### `useVoice`

**文件：** `hooks/useVoice.ts`

**用途：** 使用 `voice_stream` STT endpoint 的按住说话录音。自动重复的按键事件会延长录音；在 `RELEASE_TIMEOUT_MS` 后松开按键则停止。

**参数：**
- `{ onTranscript: (text: string) => void, enabled: boolean }`

**返回值：** `{ state: 'idle'|'recording'|'processing', handleKeyEvent: (fallbackMs?: number) => void }`

**关键逻辑：**
- 调用 `connectVoiceStream()` 打开到 `voice_stream` STT 的 WebSocket
- 把用户 locale 映射成 Deepgram 用的 BCP-47 语言代码（支持 20+ 语言）
- 自动重复检测：120ms 内到达的 key event 会被视为“按住”
- 修饰键组合（如 Ctrl+Space）使用 2 000ms 的 `FIRST_PRESS_FALLBACK_MS`
- 裸字符按键需要 5 次快速按下（HOLD_THRESHOLD）才会真正激活；2 次时只给预热反馈
- 使用 `useTerminalFocus`，在终端失焦时暂停录音
- 拉取 voice keyterms 以提升领域词识别

**依赖：** `useState`、`useRef`、`useCallback`、`useEffect`、`useTerminalFocus`、`connectVoiceStream`、`getVoiceKeyterms`

---

### `useVoiceEnabled`

**文件：** `hooks/useVoiceEnabled.ts`

**用途：** 把用户意图（`settings.voiceEnabled`）、OAuth 认证检查和 GrowthBook kill-switch 合并成一个布尔值，用来表示语音模式是否可用。

**参数：** 无

**返回值：** `boolean`

**关键逻辑：**
- `userIntent` 来自 `useAppState(s => s.settings.voiceEnabled === true)`
- `authed` 以 `authVersion` 为 memo 依据，避免每次渲染都做昂贵的 `hasVoiceAuth()` 调用
- `isVoiceGrowthBookEnabled()` 不做 memo（因为它是便宜的缓存查找，这样 mid-session kill-switch 也能立即生效）

**依赖：** `useAppState`、`useMemo`

---

### `useVoiceIntegration`

**文件：** `hooks/useVoiceIntegration.tsx`

**用途：** 编排完整的语音模式集成：读取 keybinding、检测按住的按键、激活 `useVoice`、录音期间抑制全角空格输入，并显示状态通知。

**参数：**
- `{ onTranscript: (text: string) => void, isModalOverlayActive: boolean }`

**返回值：** `{ voiceState: VoiceState, handleVoiceKeyEvent }`

**关键逻辑：**
- 从 keybinding context 读取 `voice:activate`（默认是空格键）
- 通过统计快速 key event 来检测是否“按住”按键（裸字符 HOLD_THRESHOLD=5，修饰键组合为 1）
- 在 WARMUP_THRESHOLD=2 次事件后显示预热通知
- 在 REPL 还没把 `handleKeyDown` 接到 `<Box onKeyDown>` 前，先用 `useInput`（ink）做兼容桥
- 依赖 `useIsModalOverlayActive()`：modal 打开时不会激活 voice
- 做 dead-code elimination：只有在 `feature('VOICE_MODE')` 开启时才条件性 require `useVoice`，否则用 no-op stub
- 调用 `normalizeFullWidthSpace` 处理全角空格（日本 IME），把它交给 transcript，而不是激活语音

**依赖：** `useVoice`、`useVoiceEnabled`、`useInput`（ink）、`useOptionalKeybindingContext`、`useIsModalOverlayActive`、`useNotifications`、`useState`、`useRef`、`useMemo`、`useCallback`、`useEffect`

---

### `useVirtualScroll`

**文件：** `hooks/useVirtualScroll.ts`

**用途：** `ScrollBox` 内 `MessageRow` 的 React 级虚拟化。只挂载视口内加 overscan 的项目，并用 spacer box 保持滚动高度。

**参数：**
- `scrollRef: RefObject<ScrollBoxHandle | null>` - `ScrollBox` 的引用
- `itemKeys: readonly string[]` - 每个 item 的稳定 key
- `columns: number` - 终端宽度；变化时会触发高度缓存重缩放

**返回值：** `VirtualScrollResult`：
- `range: [startIndex, endIndex)` - 半开区间的渲染切片
- `topSpacer: number` - 第一个渲染项前的行数
- `bottomSpacer: number` - 最后一个渲染项后的行数
- `measureRef: (key) => ref` - 挂到每个 item 根 `Box` 上做高度测量
- `spacerRef: RefObject<DOMElement>` - 挂到顶部 spacer 上，用于无漂移的原点追踪
- `offsets: ArrayLike<number>` - 每个 item 的累计 y offset
- `getItemTop: (index) => number` - 读取实时 Yoga `computedTop`
- `getItemElement: (index) => DOMElement | null`
- `getItemHeight: (index) => number | undefined`
- `scrollToIndex: (i) => void`

**关键逻辑：**
- `DEFAULT_ESTIMATE = 3` 行，供未测量 item 使用；`OVERSCAN_ROWS = 80`；`COLD_START_COUNT = 30`
- `SCROLL_QUANTUM = 40` 行：把 scrollTop 量化，这样只有 mounted range 真需要移动时才让 React 重渲染，而不是每次滚轮都重渲染
- `SLIDE_STEP = 25`：限制每次 commit 新挂载的数量，控制 reconcile 时间
- `PESSIMISTIC_HEIGHT = 1`：用于 coverage back-walk，确保视口被覆盖
- `MAX_MOUNTED_ITEMS = 300`
- 列数变化时，不会清空缓存，而是按 `oldCols/newCols` 缩放所有缓存高度
- 使用 `useSyncExternalStore` 订阅 ScrollBox 的 scroll-top 外部 store
- 使用 `useLayoutEffect` 在每次 commit 后测量 Yoga 高度
- sticky-scroll：当固定在底部时，总是渲染最后 N 个 item

**依赖：** `useRef`、`useMemo`、`useDeferredValue`、`useLayoutEffect`、`useSyncExternalStore`、`ScrollBox` handle

---

## 权限与工具使用 Hooks

### `useCanUseTool`

**文件：** `hooks/useCanUseTool.tsx`

**用途：** 工具执行的核心权限闸门。每次工具使用尝试都会走这里，然后按 coordinator、interactive、swarm-worker 的不同路径处理，最后解析出一个 `PermissionDecision`。

**参数：**
- `setToolUseConfirmQueue: SetState<ToolUseConfirm[]>`
- `setToolPermissionContext: (ctx: ToolPermissionContext) => void`

**返回值：** `CanUseToolFn` - `async (tool, input, toolUseContext, assistantMessage, toolUseID) => PermissionDecision`

**关键逻辑：**
1. 调用 `hasPermissionsToUseTool` 拿到初始决策（`allow`、`deny` 或 `ask`）。
2. 如果是 `allow` 或 `deny`：记录日志并立刻返回。
3. 如果是 `ask`：
   a. swarm worker：交给 `handleSwarmWorkerPermission`（通过 mailbox 转发给 leader）。
   b. coordinator worker：交给 `handleCoordinatorPermission`（先等 hooks + classifier，再往下走）。
   c. 其他情况：交给 `handleInteractivePermission`（显示对话框，并同时竞争 hooks/classifier/bridge/channel）。
4. 创建一个 `PermissionContext` 对象，里面带齐所有回调（logDecision、persistPermissions、tryClassifier、runHooks 等）。

**依赖：** `useCallback`、`useAppState`、`useSetAppState`、`createPermissionContext`、`createPermissionQueueOps`、`handleCoordinatorPermission`、`handleInteractivePermission`、`handleSwarmWorkerPermission`

---

### `CancelRequestHandler`（作为 `useCancelRequest` 模块导出）

**文件：** `hooks/useCancelRequest.ts`

**用途：** 一个返回 `null` 的 React 组件，负责注册三类取消/中断 keybinding：
1. `chat:cancel`（Escape） - 取消正在运行的任务，或者弹出队列里的命令。
2. `app:interrupt`（Ctrl+C） - 取消当前任务；在 teammate 视图里，还会杀掉所有 agents 并退出。
3. `chat:killAgents`（Ctrl+X Ctrl+K） - 双击模式，用来停止所有后台 agent。

**参数：** `CancelRequestHandlerProps`：
- `setToolUseConfirmQueue`、`onCancel`、`onAgentsKilled`
- `isMessageSelectorVisible`、`screen`
- `abortSignal?: AbortSignal`
- `popCommandFromQueue?`、`vimMode`、`isLocalJSXCommand`、`isSearchingHistory`、`isHelpOpen`
- `inputMode?`、`inputValue?`、`streamMode?`

**返回值：** `null`

**关键逻辑：**
- `handleCancel`：优先级 1 是任务运行中的 abort signal；优先级 2 是队列不空时弹出命令；否则调用 `onCancel`
- `handleInterrupt`：如果在 teammate 视图，先杀掉所有 agent 并退出，然后再调用 `handleCancel`
- `handleKillAgents`：第一次按会显示“Press again”提示；3 000ms（`KILL_AGENTS_CONFIRM_WINDOW_MS`）内第二次按下时，会杀掉所有 `local_agent` 任务、发 SDK events，并排队一个聚合通知
- `isEscapeActive` / `isCtrlCActive` 这些保护条件：overlay、vim INSERT、transcript、history-search、help 等状态下会跳过
- `chat:killAgents` 总会注册，避免 Ctrl+K（chord prefix）直接落到 readline

**依赖：** `useKeybinding`、`useAppState`、`useSetAppState`、`useCommandQueue`、`useNotifications`、`useIsOverlayActive`、`useCallback`、`useRef`、`killAllRunningAgentTasks`、`emitTaskTerminatedSdk`

---

### `useSwarmPermissionPoller`

**文件：** `hooks/useSwarmPermissionPoller.ts`

**用途：** 当当前进程作为 worker agent 运行时，每 500ms 轮询 swarm leader 的权限响应。收到响应后，会调用注册好的回调（`onAllow` 或 `onReject`）。

**参数：** 无

**返回值：** `void`

**关键逻辑：**
- 只有在 `isSwarmWorker()` 返回 `true` 时才启用
- 使用 `useInterval`（usehooks-ts），轮询间隔 `POLL_INTERVAL_MS = 500`
- 对 `pendingCallbacks` 中每个 `requestId` 调 `pollForResponse(requestId, agentName, teamName)`
- 收到响应后：调用 `processResponse(response)` 执行回调，然后调用 `removeWorkerResponse`
- 模块级 `pendingCallbacks: Map<string, PermissionResponseCallback>` 和 `pendingSandboxCallbacks: Map<...>`
- 导出的辅助函数：`registerPermissionCallback`、`unregisterPermissionCallback`、`hasPermissionCallback`、`clearAllPendingCallbacks`、`processMailboxPermissionResponse`、`registerSandboxPermissionCallback`、`hasSandboxPermissionCallback`、`processSandboxPermissionResponse`

**依赖：** `useCallback`、`useEffect`、`useRef`、`useInterval`、`permissionSync`

---

## Swarm / Teammate Hooks

### `useSwarmInitialization`

**文件：** `hooks/useSwarmInitialization.ts`

**用途：** 在挂载时初始化 swarm 功能（teammate context 和 hooks）。既处理恢复的会话（transcript 里有 teamName/agentName），也处理新 spawn（环境变量）。

**参数：**
- `setAppState: SetAppState`
- `initialMessages: Message[] | undefined`
- `{ enabled?: boolean }`

**返回值：** `void`

**关键逻辑：**
- 先检查 `isAgentSwarmsEnabled()`，不满足就不做任何事
- 恢复会话路径：从 `initialMessages[0]` 读 `teamName`/`agentName`，调用 `initializeTeammateContextFromSession`，再读 team 文件拿 `agentId`，最后调用 `initializeTeammateHooks`
- 新 spawn 路径：调用 `getDynamicTeamContext()` 读环境变量，再调用 `initializeTeammateHooks`

**依赖：** `useEffect`

---

### `useTeammateViewAutoExit`

**文件：** `hooks/useTeammateViewAutoExit.ts`

**用途：** 当正在查看的 teammate 被杀掉、失败、出错，或者从 task map 里被驱逐时，自动退出 teammate 查看模式。

**参数：** 无

**返回值：** `void`

**关键逻辑：**
- 只选取 `viewingAgentTaskId` 和当前被查看的 task，避免因为无关流式更新而重渲染
- 把 task 收窄成 `InProcessTeammateTask`
- 如果 task 被驱逐，或者状态是 `killed`、`failed`，或者带 error，就退出
- 如果状态是 `running`、`completed` 或 `pending`，则不退出

**依赖：** `useEffect`、`useAppState`、`useSetAppState`、`exitTeammateView`

---

### `useBackgroundTaskNavigation`

**文件：** `hooks/useBackgroundTaskNavigation.ts`

**用途：** 管理后台 task list 的键盘导航。Shift+Up/Down 移动选择；Enter 进入查看；`f` 打开完整 transcript；`k` kill task；Escape 退出。

**参数：** `options?: { isActive?: boolean }`

**返回值：** `{ handleKeyDown: (key: Key, input: string) => void }`

**关键逻辑：**
- 从 AppState 读取后台任务列表
- `selectedIndex` 会在列表变化时被限制在 `[0, tasks.length - 1]`
- Enter：把被选中 task 的 ID 写入 AppState 的 `viewingAgentTaskId`
- `f`：把 `showFullTranscript = true` 写入 AppState
- `k`：调用 `killTask(task.id)`
- Escape：调用 `exitTeammateView(setAppState)`

**依赖：** `useAppState`、`useSetAppState`、`useState`、`useEffect`、`useCallback`

---

### `useInboxPoller`

**文件：** `hooks/useInboxPoller.ts`

**用途：** 每秒轮询一次 team lead 的 inbox（或者在 idle 时按需轮询），然后路由消息。处理 permission requests/responses、sandbox permissions、plan approvals、shutdown、team permission updates、mode-set requests，以及普通消息。

**参数：**
- `{ enabled: boolean, isLoading: boolean, focusedInputDialog: string | null, onSubmitMessage: (msg) => void }`

**返回值：** `void`

**关键逻辑：**
- 使用 `useInterval`，周期 1 000ms；如果 `!enabled` 就 no-op
- 从 team lead 的 mailbox 目录读取消息
- 按消息类型分发：
  - `permission_request` → 加到 `toolUseConfirmQueue`
  - `permission_response` → 调 `processMailboxPermissionResponse`
  - `sandbox_permission_request` → 调 sandbox permission handler
  - `sandbox_permission_response` → 调 `processSandboxPermissionResponse`
  - `plan_approval` → 路由到 plan approval handler
  - `shutdown` → 优雅退出
  - `team_permission_update` → 更新 AppState 的 permission context
  - `mode_set` → 改权限模式
  - 普通消息 → 在 idle 时调用 `onSubmitMessage`
- 只有在 `!isLoading` 且没有焦点输入对话框时，才会投递待处理消息

**依赖：** `useInterval`、`useEffect`、`useRef`、`useAppState`、`useSetAppState`、`processMailboxPermissionResponse`、`processSandboxPermissionResponse`

---

### `useTaskListWatcher`

**文件：** `hooks/useTaskListWatcher.ts`

**用途：** 监听 task list 目录，自动接手打开但尚未归属的任务（tasks mode）。它会原子地 claim 任务，避免 race condition。

**参数：**
- `{ taskListId?: string, isLoading: boolean, onSubmitTask: (prompt: string) => boolean }`

**返回值：** `void`

**关键逻辑：**
- 挂载时调用 `ensureTasksDir`，然后 `watch(tasksDir, debouncedCheck)`，`DEBOUNCE_MS = 1000`
- 用稳定 ref 保存 `isLoading` 和 `onSubmitTask`，避免 Bun PathWatcherManager deadlock（oven-sh/bun#27469），也避免每一轮都重新创建 watcher
- `checkForTasks`：列出任务，找出 `status=pending`、`owner=undefined`、所有 `blockedBy` 都完成的任务，然后调用 `claimTask(taskListId, task.id, agentId)`
- 把任务格式化成 `"Complete all open tasks. Start with task #N: ...\n\nDescription"`
- 还会有一个跟 `isLoading` 相关的 `useEffect`：在空闲时触发检查

**依赖：** `fs.watch`、`useEffect`、`useRef`

---

### `useTasksV2`

**文件：** `hooks/useTasksV2.ts`

**用途：** 暴露持久化 TodoV2 UI 的当前 task list。所有消费者共享一个 `TasksV2Store`（单例文件监听器），避免 watcher 抖动。

**参数：** 无

**返回值：** `Task[] | undefined` - 当隐藏时返回 `undefined`（全部完成超过 5 秒，或者为空）

**关键逻辑：**
- `TasksV2Store` 类负责：`fs.watch`、`onTasksUpdated` 订阅、去抖 fetch（`DEBOUNCE_MS=50`）、隐藏计时器（`HIDE_DELAY_MS=5000`）、fallback poll（`FALLBACK_POLL_MS=5000`）
- `getSnapshot` 在 `#hidden = true` 时返回 `undefined`
- 通过 `useSyncExternalStore` 订阅；第一个订阅者启动 store，最后一个取消订阅者时停止 store
- 只有在 `isTodoV2Enabled()` 且（没有 team context，或者自己是 team lead）时才生效
- `useTasksV2WithCollapseEffect`：除了 `useTasksV2` 之外，还会在列表变隐藏时把 AppState 里的展开任务视图折叠掉

**依赖：** `useSyncExternalStore`、`useEffect`、`useAppState`、`useSetAppState`、`fs.watch`

---

### `useSessionBackgrounding`

**文件：** `hooks/useSessionBackgrounding.ts`

**用途：** 管理当前会话的 Ctrl+B 后台化/前台化。当任务被前台化时，会把该任务的消息同步到主消息列表。

**参数：**（很大的 props，包括 setMessages、setIsLoading、tools 等）

**返回值：** `{ handleBackgroundSession: () => void }`

**关键逻辑：**
- `handleBackgroundSession`：如果当前有任务在跑，就把它后台化（写进 AppState 的 background tasks）；否则把第一个后台任务前台化
- 前台化时：通过 `setMessages` 把后台任务的消息注入主列表，并恢复任务的流

**依赖：** `useCallback`、`useAppState`、`useSetAppState`

---

### `useScheduledTasks`

**文件：** `hooks/useScheduledTasks.ts`

**用途：** 在 REPL 里挂载 cron 调度器。触发的任务会以 `later` priority 通过 `enqueuePendingNotification` 入队；teammate 范围的 cron 会直接注入那个 teammate 的消息流。

**参数：**
- `{ isLoading: boolean, assistantMode?: boolean, setMessages: Dispatch<SetStateAction<Message[]>> }`

**返回值：** `void`

**关键逻辑：**
- 在 effect 层由 `isKairosCronEnabled()` 控制
- 使用 `isLoadingRef` 避免 `isLoading` 的 stale closure
- `onFireTask` 回调：如果任务带 `agentId`，就找到 teammate 并调用 `injectUserMessageToTeammate`；否则创建 `ScheduledTaskFireMessage` 并把 prompt 入队
- `createCronScheduler` 是共享调度器核心（headless 模式下 `print.ts` 也会用）
- `isKilled` 回调会在每个 tick 里重新检查 `isKairosCronEnabled()`，作为 mid-session killswitch

**依赖：** `useEffect`、`useRef`、`useAppStateStore`、`useSetAppState`、`createCronScheduler`

---

## IDE 集成 Hooks

### `useIDEIntegration`

**文件：** `hooks/useIDEIntegration.tsx`

**用途：** 管理启动时的 IDE 自动连接。它会检测正在运行的 IDE，给发现的 IDE server 配置动态 MCP，然后在需要时显示 IDE onboarding 对话框。

**参数：**
- `{ autoConnectIdeFlag, ideToInstallExtension, setDynamicMcpConfig, setShowIdeOnboarding, setIDEInstallationState }`

**返回值：** `void`

**关键逻辑：**
- 挂载时调用 `detectIDEs()`，寻找正在运行的 IDE extension server
- 如果找到了：调用 `setDynamicMcpConfig`，把 IDE 的 MCP server config 加进去
- 检查 `settings.ideHintShownCount`，决定要不要显示 onboarding
- 处理 `ideToInstallExtension` CLI flag，用于直接安装 IDE extension 的流程

**依赖：** `useEffect`、`useRef`、`useAppState`

---

### `useIdeAtMentioned`

**文件：** `hooks/useIdeAtMentioned.ts`

**用途：** 监听来自 IDE extension 的 `at_mentioned` MCP 通知，并把文件/行号上下文交给回调，这样 Claude 就能引用那个文件。

**参数：**
- `mcpClients: MCPServerConnection[]`
- `onAtMentioned: (filePath: string, lineNumber?: number) => void`

**返回值：** `void`

**关键逻辑：**
- 使用 `getConnectedIdeClient(mcpClients)` 找到 IDE client
- 通过 `ideClient.client.setNotificationHandler` 注册 `at_mentioned` 的通知处理器
- 把解析后的 `filePath` 和 `lineNumber` 传给 `onAtMentioned`

**依赖：** `useEffect`

---

### `useIdeConnectionStatus`

**文件：** `hooks/useIdeConnectionStatus.ts`

**用途：** 返回当前 IDE 连接状态（`connected`、`disconnected`、`pending` 或 `null`）以及 IDE 名称。

**参数：**
- `mcpClients?: MCPServerConnection[]`

**返回值：** `{ status: IDEConnectionStatus | null, ideName: string | null }`

**关键逻辑：**
- 用 `useMemo` 遍历 `mcpClients`，通过 `isIdeClient(client)` 找 IDE client
- 把 client 的连接状态映射到 status enum

**依赖：** `useMemo`

---

### `useIdeLogging`

**文件：** `hooks/useIdeLogging.ts`

**用途：** 在 IDE client 上注册 `log_event` MCP 通知处理器，把 IDE telemetry 事件转发到 analytics 系统。

**参数：**
- `mcpClients: MCPServerConnection[]`

**返回值：** `void`

**关键逻辑：**
- 调用 `getConnectedIdeClient` 找到 IDE client
- 注册一个经 Zod 校验的处理器：`{ method: 'log_event', params: { eventName, eventData } }`
- 调用 `logEvent('tengu_ide_${eventName}', eventData)`

**依赖：** `useEffect`、`zod`

---

### `useIdeSelection`

**文件：** `hooks/useIdeSelection.ts`

**用途：** 监听来自 IDE 的 `selection_changed` MCP 通知，并把它们作为 `IDESelection` 对象传给 REPL。

**参数：**
- `mcpClients: MCPServerConnection[]`
- `onSelect: (selection: IDESelection) => void`

**返回值：** `void`

**关键逻辑：**
- 找到 IDE client，注册 `selection_changed` 的通知处理器
- 把原始通知 payload 转成 `IDESelection` 格式：`{ filePath, text, lineStart, lineEnd, lineCount }`

**依赖：** `useEffect`

---

### `useDiffInIDE`

**文件：** `hooks/useDiffInIDE.ts`

**用途：** 通过 MCP RPC 在连接的 IDE 里打开文件 diff。会处理用户保存/关闭/拒绝，以便最终确认或回滚编辑。

**参数：**
- `{ onChange, toolUseContext, filePath, edits, editMode }`

**返回值：** `{ closeTabInIDE, showingDiffInIDE, ideName, hasError }`

**关键逻辑：**
- 调用 `ideClient.client.request('show_diff', { filePath, edits, ... })`，在 IDE 里打开 diff
- 等待 IDE 返回 `saved`、`closed` 或 `rejected`
- 如果是 `saved`：调用 `onChange` 应用编辑
- 如果是 `rejected`：调用 abort controller
- 返回 `closeTabInIDE()`，这样权限对话框可以程序化关闭 tab

**依赖：** `useState`、`useRef`、`useCallback`、`useEffect`

---

## 远程与会话 Hooks

### `useDirectConnect`

**文件：** `hooks/useDirectConnect.ts`

**用途：** 管理到 DirectConnect server（本地 server 模式）的 WebSocket 连接。负责在 server 和 REPL 之间路由入站消息和权限请求。

**参数：**
- `{ config, setMessages, setIsLoading, setToolUseConfirmQueue, tools }`

**返回值：** `UseDirectConnectResult` - `{ isConnected, send }`

**关键逻辑：**
- 使用 `directConnectManager` 管理 WebSocket 生命周期
- 把入站 SDK messages 转成 `Message[]` 格式
- 通过 `setToolUseConfirmQueue` 处理工具权限请求
- 断开后带指数退避重连

**依赖：** `useEffect`、`useRef`、`useState`

---

### `useSSHSession`

**文件：** `hooks/useSSHSession.ts`

**用途：** 把 SSH session manager 接到 REPL。处理重连、断开时的优雅关闭，以及 transcript 消息注入。

**参数：**
- `{ session, setMessages, setIsLoading, setToolUseConfirmQueue, tools }`

**返回值：** `UseSSHSessionResult` - `{ isConnected, disconnect }`

**关键逻辑：**
- 订阅 SSH session 事件：`message`、`connect`、`disconnect`、`error`
- 断开时：注入一条 system message，说明已经断开并给出重连选项
- 重连时：注入一条 system message，确认已经恢复连接
- 处理优雅关闭：在关闭前先把待处理消息 drain 完

**依赖：** `useEffect`、`useRef`、`useState`

---

### `useRemoteSession`

**文件：** `hooks/useRemoteSession.ts`

**用途：** 完整的 CCR（Claude Code Remote）WebSocket 会话管理。处理双向消息转换、流式 tool use、权限请求/响应流程、响应超时检测、会话标题更新和 subagent 任务计数。

**参数：** 一个很大的 props 对象，包括：
- `config: AppConfig`
- `setMessages: SetMessages`
- `setIsLoading`、`setToolUseConfirmQueue`
- `tools: Tool[]`
- `onSessionTitleUpdate?: (title: string) => void`

**返回值：** `UseRemoteSessionResult` - `{ isConnected, sendMessage, sessionId, ... }`

**关键逻辑：**
- 挂载时连接 CCR WebSocket，断开后重连
- 入站时把 `SDKMessage` 转成内部 `Message` 格式
- 出站时把消息转成 SDK 格式，供 CCR 消费
- 管理流式 tool use：累积 `input_json_delta` chunk，在 `tool_use` 完成时弹出权限对话框
- 响应超时：如果模型 30 秒没有响应，就设置一个标志
- 会话标题：订阅 `session_title_update` 事件，并调用 `onSessionTitleUpdate`
- subagent 任务计数：跟踪 `subagent_start` / `subagent_end` 事件，统计正在运行的 agent

**依赖：** `useEffect`、`useState`、`useRef`、`useCallback`、`SessionsWebSocket`

---

### `useAssistantHistory`

**文件：** `hooks/useAssistantHistory.ts`

**用途：** 在仅查看模式里，随着用户向上滚动，懒加载远程会话历史里的旧消息。

**参数：**
- `{ config, setMessages, scrollRef, onPrepend }`

**返回值：** `{ maybeLoadOlder: () => Promise<void> }`

**关键逻辑：**
- 在向上滚动时调用 `loadSessionHistory(sessionId, pageToken)`
- 把加载到的消息 prepend 到消息列表前面
- 链式补足 viewport：如果新消息还没填满视口，就立刻再加载下一页
- 滚动锚定：在 prepend 之前保存当前滚动位置，之后再恢复
- 当页数耗尽时，在顶部显示一个 sentinel 消息（“Beginning of conversation”）

**依赖：** `useCallback`、`useRef`、`useEffect`

---

### `useMailboxBridge`

**文件：** `hooks/useMailboxBridge.ts`

**用途：** 把 mailbox 消息 context 和 REPL 的 submit 函数桥接起来。在空闲时，revision 变化会触发 mailbox 轮询。

**参数：**
- `{ isLoading: boolean, onSubmitMessage: (msg) => void }`

**返回值：** `void`

**关键逻辑：**
- 订阅 AppState 里的 `mailboxRevision` 变化
- 当 revision 增加且 `!isLoading` 时，调用 `pollMailbox()`，并通过 `onSubmitMessage` 提交任何待处理消息

**依赖：** `useEffect`、`useRef`、`useAppState`

---

### `useReplBridge`

**文件：** `hooks/useReplBridge.tsx`

**用途：** 完整的 REPL bridge 会话管理，也是把 REPL 接到 Claude API 的主 hook。它管理完整的 query 执行循环、权限流、流式消息组装、compact 操作和 bridge 连通性。

**参数：** 很大的 props，包括 tools、messages、setMessages、config 和许多回调。

**返回值：** 很大的结果对象，包括 `{ onQuery, isLoading, abortController, toolUseConfirmQueue, ... }`

**关键逻辑：**（这个文件超过 75k tokens，这里根据前 80 行和摘要提炼）
- 管理 `abortController` 生命周期：每次 query 创建一个新的，cancel 时 abort
- 通过 `query.ts` 调用流式 Claude API
- 把流式 `AssistantMessage` 的 delta 事件拼装起来
- 把 `tool_use` 路由到 `canUseTool` 做权限 gating
- 处理 compact boundary 检测和自动 compact 触发
- 接入 bridge 回调，用于 CCR 的权限转发
- 管理本地 session history 记录

**依赖：** `useState`、`useRef`、`useCallback`、`useEffect`、`useMemo`、`useCanUseTool`、`useLogMessages`、`query`、`compact`

---

### `useTeleportResume`

**文件：** `hooks/useTeleportResume.tsx`

**用途：** 管理 teleport 到远程 Code Session 的异步生命周期：加载状态、错误状态、被选中的 session 跟踪，以及 `resumeSession` 回调。

**参数：**
- `source: TeleportSource` - `'cliArg' | 'localCommand'`（用于 analytics）

**返回值：** `{ resumeSession, isResuming, error, selectedSession, clearError }`

**关键逻辑：**
- `resumeSession(session)`：把 `isResuming = true`，记录 `tengu_teleport_resume_session`，调用 `teleportResumeCodeSession(session.id)`，再设置 `teleportedSessionInfo` 以便可靠地记录日志
- 出错时：包装成 `TeleportResumeError`，带 `isOperationError` 标志，方便 UI 区分
- 使用 React Compiler（`_c`）做 memoization

**依赖：** `useState`、`useCallback`、`teleportResumeCodeSession`、`setTeleportedSessionInfo`

---

## 插件与建议 Hooks

### `usePromptSuggestion`

**文件：** `hooks/usePromptSuggestion.ts`

**用途：** 管理 AI prompt completion 建议：为当前输入获取建议，追踪接受/忽略/提交结果，并记录 telemetry。

**参数：**
- `{ inputValue: string, isAssistantResponding: boolean }`

**返回值：** `{ suggestion: string | null, markAccepted, markShown, logOutcomeAtSubmission }`

**关键逻辑：**
- 在用户 300ms 没有继续输入后，去抖调用 `generatePromptSuggestion(inputValue)`
- `markAccepted()`：记录用户按了 Tab 接受建议
- `markShown()`：记录 ghost-text 建议何时变得可见
- `logOutcomeAtSubmission()`：在提交时调用；记录 `accept`（按过 Tab）、`ignore`（显示了但没接受）、或 `no_suggestion`

**依赖：** `useState`、`useRef`、`useCallback`、`useEffect`、`generatePromptSuggestion`

---

### `usePromptsFromClaudeInChrome`

**文件：** `hooks/usePromptsFromClaudeInChrome.tsx`

**用途：** 监听从 Claude in Chrome 扩展发来的 prompt（通过 MCP 通知）。同时会把当前权限模式同步回扩展。

**参数：**
- `mcpClients: MCPServerConnection[]`
- `toolPermissionMode: PermissionMode`

**返回值：** `void`

**关键逻辑：**
- 找到 Chrome extension 的 MCP client
- 注册 `prompt_from_chrome` 的通知处理器
- 把收到的 prompt 提交到命令队列
- 当 `toolPermissionMode` 变化时，把 `mode_changed` 通知回传给扩展

**依赖：** `useEffect`、`useRef`

---

### `usePrStatus`

**文件：** `hooks/usePrStatus.ts`

**用途：** 每 60 秒轮询一次 `gh pr status`，检测当前分支 PR 的 review 状态变化。

**参数：**
- `isLoading: boolean`
- `enabled?: boolean`

**返回值：** `PrStatusState` - `{ reviewStatus: 'approved'|'changes_requested'|'pending'|null, prUrl: string | null }`

**关键逻辑：**
- 使用 `useInterval`，周期 60 000ms；`isLoading` 时跳过轮询
- 超过 60 分钟没有新 turn 后停止轮询
- 如果一次 fetch 超过 4 秒，就永久禁用（通常意味着没有 `gh` binary 或根本没有 PR）
- 在子进程里跑 `gh pr status --json reviewDecision,url`

**依赖：** `useInterval`、`useState`、`useRef`

---

### `useClaudeCodeHintRecommendation`

**文件：** `hooks/useClaudeCodeHintRecommendation.tsx`

**用途：** 从 Claude 响应里解析 `<claude-code-hint />` 标签，提示安装插件。每个 session 内每个插件只显示一次。

**参数：** 无

**返回值：** `{ recommendation: PluginRecommendation | null, handleResponse: (accepted: boolean) => void }`

**关键逻辑：**
- 监控 `messages`，找出包含 `<claude-code-hint plugin="name" />` XML 的 assistant message
- 用 `usePluginRecommendationBase` 的状态机控制显示
- `handleResponse(true)`：安装插件；`false`：关闭

**依赖：** `usePluginRecommendationBase`、`useAppState`、`useEffect`

---

### `useLspPluginRecommendation`

**文件：** `hooks/useLspPluginRecommendation.tsx`

**用途：** 当用户编辑的文件扩展名匹配受支持语言，而且 LSP binary 存在时，推荐一个 LSP 插件。

**参数：** 无

**返回值：** `{ recommendation: PluginRecommendation | null, handleResponse: (accepted: boolean) => void }`

**关键逻辑：**
- 监听 `messages` 里的 `FileEdit` / `FileWrite` tool result message
- 提取文件扩展名，检查是否有匹配的 LSP 插件
- 通过 `usePluginRecommendationBase` 避免在别的推荐还在显示时弹出
- 每个 session 只显示一次（记录在 AppState）

**依赖：** `usePluginRecommendationBase`、`useAppState`、`useEffect`

---

### `usePluginRecommendationBase`

**文件：** `hooks/usePluginRecommendationBase.tsx`

**用途：** 插件建议共享的状态机。它会阻止在 remote mode、已有推荐正在显示、或者检查仍在进行时弹出推荐。

**参数：** 泛型 `T`

**返回值：** `{ recommendation: T | null, clearRecommendation, tryResolve: (candidate: T | null) => void }`

**关键逻辑：**
- `tryResolve(candidate)`：如果是 remote mode 或已经有推荐在显示，就什么都不做；否则设置 `recommendation`
- `clearRecommendation()`：清空当前推荐
- 用 `inFlight` ref 防止并发检查

**依赖：** `useState`、`useRef`、`useCallback`

---

### `useOfficialMarketplaceNotification`

**文件：** `hooks/useOfficialMarketplaceNotification.tsx`

**用途：** 处理官方 marketplace 的首次启动自动安装，并显示成功/失败的启动通知。

**参数：** 无

**返回值：** `void`

**关键逻辑：**
- 挂载时检查 `settings.autoInstallOfficialMarketplace`
- 如果开了且还没安装：调用 `installOfficialMarketplace()`
- 成功或失败时都显示一个 `startup` priority 通知

**依赖：** `useEffect`、`useRef`、`useNotifications`

---

### `useChromeExtensionNotification`

**文件：** `hooks/useChromeExtensionNotification.tsx`

**用途：** 显示关于 Chrome extension 状态的启动通知：需要订阅、尚未安装，或者默认启用。

**参数：** 无

**返回值：** `void`

**关键逻辑：**
- 内部使用 `useStartupNotification`（notifs/）
- 检查 settings/config 里的 `chrome_extension_status`
- 针对不同状态返回对应的通知文案

**依赖：** `useStartupNotification`

---

### `useClipboardImageHint`

**文件：** `hooks/useClipboardImageHint.ts`

**用途：** 当终端获得焦点且剪贴板里有图片时，显示一条通知。它会去抖 1 000ms，并带 30 秒冷却时间。

**参数：**
- `isFocused: boolean`
- `enabled: boolean`

**返回值：** `void`

**关键逻辑：**
- 监听 `isFocused` 的变化
- 获得焦点时，通过 `navigator.clipboard.read()` 检查 `image/*` MIME
- 如果找到图片：调用 `addNotification({ key: 'clipboard-image', text: 'Image in clipboard · Ctrl+V to attach' })`
- 冷却：用上次显示时间戳防止通知刷屏

**依赖：** `useEffect`、`useRef`、`useNotifications`

---

### `useManagePlugins`

**文件：** `hooks/useManagePlugins.ts` — （同上；完整说明见“核心”部分）

---

### `useIssueFlagBanner`

**文件：** `hooks/useIssueFlagBanner.ts`

**用途：** ANT 内部：当会话出现摩擦信号（例如反复 tool retry），并且已经活跃至少 30 分钟、提交次数达到 3 次以上时，显示 issue-flag banner。

**参数：**
- `messages: readonly Message[]`
- `submitCount: number`

**返回值：** `boolean` - 是否显示 banner

**关键逻辑：**
- 统计最近消息中的 tool error、retry 和 refusal
- 只对 ANT 内部用户启用（检查 `isAntInternal()`）
- 30 分钟冷却期存储在 localStorage

**依赖：** `useMemo`、`useRef`

---

### `useQueueProcessor`

**文件：** `hooks/useQueueProcessor.ts`

**用途：** 当当前没有活跃 query、也没有阻塞性 UI 时，处理统一命令队列里的待办命令。

**参数：**
- `{ executeQueuedInput: (cmd: QueuedCommand) => void, hasActiveLocalJsxUI: boolean, queryGuard: () => boolean }`

**返回值：** `void`

**关键逻辑：**
- 使用两个 `useSyncExternalStore` 订阅：一个看命令队列，一个看“是否 loading”状态
- 当队列非空、`!isLoading`、`!hasActiveLocalJsxUI` 且 `queryGuard()` 为 true 时，取出并执行下一个命令
- 用 `useEffect` 在状态变化时触发处理

**依赖：** `useSyncExternalStore`、`useEffect`、`messageQueueManager`

---

### `useTurnDiffs`

**文件：** `hooks/useTurnDiffs.ts`

**用途：** 从消息列表里提取每个 turn 的文件 diff，供 `/diff` 视图展示。它做增量处理，每次渲染只扫描新消息。

**参数：**
- `messages: Message[]`

**返回值：** `TurnDiff[]` - 按时间倒序排列、修改过文件的 turn 列表

**关键逻辑：**
- `TurnDiff`：`{ turnIndex, userPromptPreview, timestamp, files: Map<string, TurnFileDiff>, stats }`
- 从不是 tool result、也不是 `isMeta` 的 user message 中识别 turn 边界
- 在每个 turn 里收集 `FileEdit` / `FileWrite` tool result
- 新文件 hunks：从 `content` 生成 synthetic `+line` hunks
- 同一个 turn 里对同一文件的多次编辑会累积
- cache ref 保存 `completedTurns` + `currentTurn` + `lastProcessedIndex`，所以处理复杂度是 O(n_new)

**依赖：** `useMemo`、`useRef`

---

### `useSkillImprovementSurvey`

**文件：** `hooks/useSkillImprovementSurvey.ts`

**用途：** 管理 skill improvement survey 对话框。由 `AppState.skillImprovementSurvey` 触发；接受时会把改进应用到 skills 文件里。

**参数：**
- `setMessages: SetMessages`

**返回值：** `{ isOpen: boolean, suggestion: SkillSuggestion | null, handleSelect: (selection) => void }`

**关键逻辑：**
- 从 `AppState.skillImprovementSurvey` 读取待处理 survey
- `handleSelect('apply')`：调用 `applySkillImprovement(suggestion)`，并注入一条 system message
- `handleSelect('dismiss')`：把 survey 从 AppState 里清掉
- 无论选什么，都会清空 `AppState.skillImprovementSurvey`

**依赖：** `useAppState`、`useSetAppState`、`useCallback`

---

### `useGlobalKeybindings`

**文件：** `hooks/useGlobalKeybindings.tsx`

**用途：** 一个返回 `null` 的 React 组件，负责注册全局 keybinding handler：Ctrl+T（切换 todos）、Ctrl+O（切换 transcript/messages）、Ctrl+E（切换 show-all）、Escape/Ctrl+C（退出 transcript）。

**参数：** 包括 `screen`、`isLoading`、`isSearchingHistory`、`isHelpOpen` 等 props

**返回值：** `null`

**关键逻辑：**
- 注册 `view:toggleTasks`、`view:toggleTranscript`、`view:toggleShowAll`
- KAIROS feature flag：`view:toggleTasks` 只有在 `isTodoV2Enabled()` 时才启用
- `view:toggleTranscript` 会把 AppState 里的 `expandedView` 设成 `'messages'` 或 `'none'`
- `view:toggleShowAll` 会设置 `showAllMessages`

**依赖：** `useKeybinding`、`useAppState`、`useSetAppState`

---

### `CommandKeybindingHandlers`

**文件：** `hooks/useCommandKeybindings.tsx`

**用途：** 把所有 `command:*` keybinding 动作注册成 slash command 提交器，例如 `command:compact`、`command:memory`、`command:config` 等。

**参数：** `{ onSubmit: (cmd: string) => void, isActive?: boolean }`

**返回值：** `null`

**关键逻辑：**
- 调用 `useKeybindings`，传入 `command:X` → `() => onSubmit('/X')` 的映射
- 从 keybinding action 名里推导 slash command 名称

**依赖：** `useKeybindings`

---

### `useAwaySummary`

**文件：** `hooks/useAwaySummary.ts`

**用途：** 当终端失焦 5 分钟且没有 turn 正在进行时，追加一条“你不在的时候发生了什么”的摘要消息。

**参数：**
- `messages: readonly Message[]`
- `setMessages: SetMessages`
- `isLoading: boolean`

**返回值：** `void`

**关键逻辑：**
- 由 bundle flag `feature('AWAY_SUMMARY')` 和 GrowthBook flag `tengu_sedge_lantern` 控制
- 订阅 `subscribeTerminalFocus` 的 blur/focus 事件
- blur 时启动 `BLUR_DELAY_MS = 5 * 60_000` 计时器
- 计时器触发时：如果仍在 loading，就把 `pendingRef = true`（延后执行）；否则调用 `generate()`
- `generate()`：调用 `generateAwaySummary(messages, signal)` → 再追加 `createAwaySummaryMessage(text)`
- focus 时：清计时器、abort 进行中的生成、清 `pendingRef`
- 第二个 `useEffect` 监听 `isLoading`：如果 `!isLoading` 且 `pendingRef` 还在、而且仍然失焦，就触发 `generate()`
- `hasSummarySinceLastUserTurn()`：向后遍历，避免重复摘要

**依赖：** `useEffect`、`useRef`、`useCallback`、`subscribeTerminalFocus`、`generateAwaySummary`

---

### `useAfterFirstRender` / `renderPlaceholder`

`useAfterFirstRender` 见上面的“核心 / 工具 Hooks”。

---

## 通知 Hooks (`notifs/`)

`notifs/` 里的所有 hooks 都使用 `context/notifications.js` 的 `useNotifications()` 往状态栏通知队列里塞条目。多数 hook 都受 `!getIsRemoteMode()` 保护。

### `useStartupNotification`

**文件：** `hooks/notifs/useStartupNotification.ts`

**用途：** 一次性挂载即触发通知的基础原语。它把 remote-mode gate 和“每个 session 只跑一次”的 ref 保护封装起来，供大多数 `notifs/` hooks 使用。

**参数：**
- `compute: () => Result | Promise<Result>` - 返回 `null` 表示跳过，返回一个 `Notification` 表示单条，返回 `Notification[]` 表示多条

**返回值：** `void`

**关键逻辑：**
- `hasRunRef` 防止重渲染时再次触发
- 在 `Promise.resolve().then(...)` 里运行 `compute`，支持 async
- 出错时通过 `logError` 记录
- 在 remote mode 下直接跳过

**依赖：** `useEffect`、`useRef`、`useNotifications`

---

### `useAutoModeUnavailableNotification`

**文件：** `hooks/notifs/useAutoModeUnavailableNotification.ts`

**用途：** 当 Shift+Tab 模式轮转越过本该是 auto mode 的位置时，显示一次性警告，解释为什么 auto mode 不可用。

**参数：** 无

**关键逻辑：**
- 检测这个回绕：`mode === 'default' && prevMode !== 'default' && prevMode !== 'auto' && !isAutoModeAvailable && hasAutoModeOptIn()`
- 调用 `getAutoModeUnavailableReason()` 获取具体原因（circuit-breaker、org-allowlist、settings）
- `shownRef` 防止同一个 session 显示多次
- 由 `feature('TRANSCRIPT_CLASSIFIER')` 控制

---

### `useCanSwitchToExistingSubscription`

**文件：** `hooks/notifs/useCanSwitchToExistingSubscription.tsx`

**用途：** 在多个 session 中最多显示 3 次（`MAX_SHOW_COUNT=3`）的一条通知，提醒那些已经有 Claude Pro/Max 订阅、但正在用 API key 登录的用户运行 `/login`。

**关键逻辑：**
- 读取 `globalConfig.subscriptionNoticeCount`；如果已经 ≥ 3 就不显示
- 调用 `getOauthProfileFromApiKey()` 检查是否有 Pro/Max 订阅
- 每次显示时把 `subscriptionNoticeCount` 写回 global config
- 渲染一条 `color="suggestion"` 的 JSX 通知

---

### `useDeprecationWarningNotification`

**文件：** `hooks/notifs/useDeprecationWarningNotification.tsx`

**用途：** 当当前模型已弃用时，显示一条 `color="warning"` 的通知。

**参数：** `model: string`

**关键逻辑：**
- 每次 `model` 变化时调用 `getModelDeprecationWarning(model)`
- 用 `lastWarningRef` 防止重渲染时重复加入同一条通知
- 一旦模型变成未弃用，就重置追踪

---

### `useFastModeNotification`

**文件：** `hooks/notifs/useFastModeNotification.tsx`

**用途：** 针对 fast mode 状态变化显示实时通知：冷却开始/结束、组织级启用/禁用，以及 overage 拒绝。

**关键逻辑：**
- 订阅 `onCooldownTriggered`、`onCooldownExpired`、`onFastModeOverageRejection`、`onOrgFastModeChanged`
- 当 fast mode 开着但组织禁用时，会把 AppState 里的 fast mode 关掉
- 显示 immediate-priority 通知，颜色用 `color="fastMode"` 或 `color="warning"`

---

### `useIDEStatusIndicator`

**文件：** `hooks/notifs/useIDEStatusIndicator.tsx`

**用途：** 在通知区域显示 IDE 连接状态：安装扩展的提示、JetBrains 信息、安装错误，或者当前选择的预览。

**参数：** `{ ideInstallationStatus, ideSelection, mcpClients }`

**关键逻辑：**
- 使用 `useIdeConnectionStatus(mcpClients)` 获取当前状态
- “安装扩展”提示最多显示 `MAX_IDE_HINT_SHOW_COUNT=5` 次（记录在 globalConfig）
- JetBrains 会显示一条信息通知，和 VS Code 流程不同
- 当选中了文件/文本时，会把选择预览显示成持久通知

---

### `useInstallMessages`

**文件：** `hooks/notifs/useInstallMessages.tsx`

**用途：** 显示原生安装器问题的启动通知（PATH 没配好、alias 没设、安装报错）。

**关键逻辑：**
- 调用 `checkInstall()` 获取安装消息
- 把消息类型映射到优先级：`error` / `userActionRequired` → `high`；`path` / `alias` → `medium`；其他 → `low`
- 颜色：`error` → `color="error"`；其他 → `color="warning"`

---

### `useLspInitializationNotification`

**文件：** `hooks/notifs/useLspInitializationNotification.tsx`

**用途：** 每 5 000ms 轮询 LSP server 状态；当 LSP manager 或单个 server 初始化失败时，显示通知。

**关键逻辑：**
- 由 `ENABLE_LSP_TOOL` 环境变量控制
- 轮询 `getInitializationStatus()` 和 `getLspServerManager()`
- 通过 `notifiedErrorsRef` set 去重错误
- 把错误写入 `appState.plugins.errors`，供 `/doctor` 展示

---

### `useMcpConnectivityStatus`

**文件：** `hooks/notifs/useMcpConnectivityStatus.tsx`

**用途：** 当 MCP server 连接失败或者需要认证时，显示通知。

**参数：** `{ mcpClients?: MCPServerConnection[] }`

**关键逻辑：**
- 按连接状态过滤 `mcpClients`：`failed`、`needs_auth`
- 分别追踪 `claudeai` client（connector）和本地 client（server）
- 显示带计数的 JSX 通知，并附上 `· /mcp` 导航提示

---

### `useModelMigrationNotifications`

**文件：** `hooks/notifs/useModelMigrationNotifications.tsx`

**用途：** 在自动模型迁移后立即显示一次性通知，比如 Sonnet 4.5 → 4.6、Opus Pro → Opus 4.6。

**关键逻辑：**
- 用 `useStartupNotification` 和一组 `MIGRATIONS` 检查函数
- 每个检查都会从 `globalConfig` 读一个时间戳字段；如果时间戳在最近 3 秒内，就说明这次启动触发了迁移，于是返回通知

---

### `useNpmDeprecationNotification`

**文件：** `hooks/notifs/useNpmDeprecationNotification.tsx`

**用途：** 当 Claude Code 是通过 npm 安装方式运行时（已弃用），显示一条 15 秒的警告通知。

**关键逻辑：**
- 如果 `isInBundledMode()` 或设置了 `DISABLE_INSTALLATION_CHECKS` 环境变量，就跳过
- 调用 `getCurrentInstallationType()`；如果是 `'development'` 安装，也跳过

---

### `usePluginAutoupdateNotification`

**文件：** `hooks/notifs/usePluginAutoupdateNotification.tsx`

**用途：** 订阅 `onPluginsAutoUpdated`，当插件在后台自动更新时，提示用户运行 `/reload-plugins`。

**关键逻辑：**
- 用 `useState([])` 保存 `updatedPlugins` 列表
- `onPluginsAutoUpdated` 订阅会传回更新过的插件 ID
- 把插件名提取出来（去掉 `@marketplace` 后缀），再显示一条 `color="success"` 的 JSX 通知
- 超时 10 000ms

---

### `usePluginInstallationStatus`

**文件：** `hooks/notifs/usePluginInstallationStatus.tsx`

**用途：** 当一个或多个插件安装失败时显示通知（来自 AppState 的 `plugins.installationStatus`）。

**关键逻辑：**
- 从 AppState 里读 `installationStatus.marketplaces` 和 `installationStatus.plugins`
- 过滤 `status === 'failed'` 的项，并 memoize 计数
- 显示 `"N plugins failed to install · /plugin for details"`，优先级 `medium`

---

### `useRateLimitWarningNotification`

**文件：** `hooks/notifs/useRateLimitWarningNotification.tsx`

**用途：** 显示 rate limit 警告：1）进入 overage mode 时立即通知；2）接近限额时显示 warning。

**参数：** `model: string`

**关键逻辑：**
- 使用 `useClaudeAiLimits()` 获取响应式 limit 数据
- `getRateLimitWarning(limits, model)`：生成接近限额的文案
- `getUsingOverageText(limits)`：生成 overage mode 文案
- overage 通知每次进入 overage 只显示一次（通过 `hasShownOverageNotification` state 跟踪）
- 团队/企业环境：如果用户没有 billing access，就跳过 overage 通知
- warning 通知只在文案变化时显示（通过 `shownWarningRef` 去重）

---

### `useSettingsErrors`

**文件：** `hooks/notifs/useSettingsErrors.tsx`

**用途：** 监控 settings 校验错误（来自 `getSettingsWithAllErrors`），并在状态栏里显示/移除 warning 通知。

**返回值：** `ValidationError[]` - 当前错误列表（也用于 `/doctor` 显示）

**关键逻辑：**
- 初始状态同步来自 `getSettingsWithAllErrors()`
- 通过 `useSettingsChange` 订阅：当文件变化时重新读取错误
- 显示 `"Found N settings issues · /doctor for details"`，超时 60 000ms
- 当错误清空时删除通知

---

### `useTeammateShutdownNotification` / `useTeammateLifecycleNotification`

**文件：** `hooks/notifs/useTeammateShutdownNotification.ts`

**用途：** 当进程内 teammate 启动或结束时，发出批量 spawn/shutdown 通知。它用 fold() 把 `"1 agent spawned"` + `"1 agent spawned"` 合成 `"2 agents spawned"`。

**关键逻辑：**
- 从 AppState 读取 `tasks`
- 用 `seenRunningRef` / `seenCompletedRef` 追踪已经看过的 running/completed ID
- `makeSpawnNotif(count)` / `makeShutdownNotif(count)` 带 5 000ms 超时和 fold 函数
- 作为 `useTeammateLifecycleNotification` 导出

---

## 工具权限子系统 (`toolPermission/`)

### `PermissionContext.ts`

**文件：** `hooks/toolPermission/PermissionContext.ts`

**用途：** 创建 `PermissionContext` 对象的工厂。这个共享上下文会传给三个 permission handler，里面包含了批准、拒绝、记录、入队和持久化 permission decision 所需的全部回调。

**导出函数：**
- `createPermissionContext(tool, input, toolUseContext, assistantMessage, toolUseID, setToolPermissionContext, queueOps?) → PermissionContext`
- `createPermissionQueueOps(setToolUseConfirmQueue) → PermissionQueueOps` - 把 React state setter 桥接到通用 queue 接口
- `createResolveOnce<T>(resolve) → ResolveOnce<T>` - 用于 race 的原子检查并标记已 resolve 保护

**PermissionContext 方法：**
- `logDecision(args, opts?)` - 委托给 `logPermissionDecision`
- `logCancelled()` - 记录 `tengu_tool_use_cancelled` 事件
- `persistPermissions(updates)` - 调 `persistPermissionUpdates` 并更新 AppState
- `resolveIfAborted(resolve)` - 如果 abort signal 已触发，直接短路
- `cancelAndAbort(feedback?, isAbort?, contentBlocks?)` - 构建 deny decision，并在适当情况下 abort controller
- `tryClassifier(pendingCheck, updatedInput)` - 等待 classifier 自动批准（仅 bash；`BASH_CLASSIFIER` feature flag）
- `runHooks(permissionMode, suggestions, updatedInput?, startTimeMs?)` - 顺序执行 PermissionRequest hooks
- `buildAllow(updatedInput, opts?)` → `PermissionAllowDecision`
- `buildDeny(message, reason)` → `PermissionDenyDecision`
- `handleUserAllow(updatedInput, permissionUpdates, feedback?, startTimeMs?, contentBlocks?, decisionReason?)` - 持久化更新、记录日志并返回 allow decision
- `handleHookAllow(finalInput, permissionUpdates, startTimeMs?)` - hook 来源的 allow 也一样
- `pushToQueue(item)`、`removeFromQueue()`、`updateQueueItem(patch)` - 通过 `queueOps` 管理队列

---

### `permissionLogging.ts`

**文件：** `hooks/toolPermission/permissionLogging.ts`

**用途：** 所有工具权限 decision 的集中 analytics 和 telemetry 记录。它会把数据分发到 Statsig（`logEvent`）、OTel telemetry、code-edit metrics，以及 `toolUseContext.toolDecisions` map。

**导出函数：**
- `logPermissionDecision(ctx, args, startTimeMs?)` - 主入口
- `isCodeEditingTool(toolName)` - 检查工具是否是 Edit/Write/NotebookEdit
- `buildCodeEditToolAttributes(tool, input, decision, source)` - 构建 OTel attributes，包括从文件路径提取语言

**Analytics 事件：**
- `tengu_tool_use_granted_in_config` - 被 settings allowlist 自动批准
- `tengu_tool_use_granted_in_prompt_permanent` / `_temporary` - 用户批准
- `tengu_tool_use_granted_by_permission_hook` - hook 批准
- `tengu_tool_use_granted_by_classifier` - classifier 批准
- `tengu_tool_use_rejected_in_prompt` - 任意拒绝
- `tengu_tool_use_denied_in_config` - 被 settings denylist 拒绝

---

### `handlers/coordinatorHandler.ts`

**文件：** `hooks/toolPermission/handlers/coordinatorHandler.ts`

**用途：** 处理 coordinator-worker 的权限流：先运行 hooks，再运行 classifier（两者都按顺序 await），然后再落到交互式对话框。

**导出：** `handleCoordinatorPermission(params) → Promise<PermissionDecision | null>`

**参数：** `CoordinatorPermissionParams` - `{ ctx, pendingClassifierCheck?, updatedInput, suggestions, permissionMode }`

**逻辑：**
1. `await ctx.runHooks(...)` - 如果 hooks 已经返回决策，就直接返回。
2. 如果开了 `BASH_CLASSIFIER` flag：`await ctx.tryClassifier?.(...)` - 如果 classifier 返回了结果，就返回。
3. 返回 `null` - 让调用方继续落到 `handleInteractivePermission`
4. 如果出现意料之外的错误：记录日志并返回 null（优雅回退到对话框）

---

### `handlers/interactiveHandler.ts`

**文件：** `hooks/toolPermission/handlers/interactiveHandler.ts`

**用途：** 处理交互式（主 agent）权限流。它会建立 `ToolUseConfirm` 队列项，并把用户交互和后台自动检查（hooks、classifier、bridge、channel）做竞速。

**导出：** `handleInteractivePermission(params, resolve) → void`（同步搭建）

**关键逻辑：**
- 创建一个 `PermissionConfirm` 队列项，带齐回调：`onAbort`、`onAllow`、`onReject`、`recheckPermission`、`onUserInteraction`、`onDismissCheckmark`
- `createResolveOnce` 保护确保只有第一次 resolve 生效
- `userInteracted` 标志：用户一旦交互过，就阻止 classifier 在 200ms 宽限期内自动批准
- **竞速 1 - 用户**：`onAllow` / `onReject` / `onAbort`
- **竞速 2 - Hooks**：异步 `ctx.runHooks(...)`，如果赢了就把项从队列里移除并 resolve
- **竞速 3 - Classifier**：`executeAsyncClassifierCheck(...)`，如果批准，会显示 checkmark UI 3 秒（聚焦）或 1 秒（失焦），然后从队列里移除
- **竞速 4 - Bridge（CCR）**：把 `permission_request` 发给 CCR；订阅 CCR 响应；如果赢了，就记录并 resolve
- **竞速 5 - Channel**：把结构化 `permission_request` 发给所有活跃的 channel MCP servers（Telegram、iMessage）；订阅响应；如果赢了，就 resolve
- Checkmark dismissal：`onDismissCheckmark` 允许用户在 checkmark 窗口里按 Escape
- Abort：如果对话中途 abort signal 触发，`claim()` 会竞速把结果 resolve 成 cancel

---

### `handlers/swarmWorkerHandler.ts`

**文件：** `hooks/toolPermission/handlers/swarmWorkerHandler.ts`

**用途：** 处理 swarm-worker 的权限流：先尝试 classifier 自动批准，然后把请求通过 mailbox 转发给 team leader，等待 leader 响应。

**导出：** `handleSwarmWorkerPermission(params) → Promise<PermissionDecision | null>`

**逻辑：**
1. 如果不是 `isAgentSwarmsEnabled()` 或不是 `isSwarmWorker()`，返回 `null`
2. 如果开了 `BASH_CLASSIFIER`：先尝试 `ctx.tryClassifier?.(...)`
3. 创建一个 `Promise<PermissionDecision>`，等 leader 响应时 resolve
4. 通过 `registerPermissionCallback` 注册 `onAllow` / `onReject`
5. 调 `sendPermissionRequestViaMailbox(request)` 通知 leader
6. 在 AppState 里设置 `pendingWorkerRequest`，用于可视化提示
7. 如果 abort：resolve 成 `cancelAndAbort`
8. 出错时：返回 `null`，让本地 UI 继续处理

---

## `hooks/` 里的非 Hook 工具

这些文件放在 `hooks/` 目录里，但并不是 React hooks。

### `fileSuggestions.ts`

**文件：** `hooks/fileSuggestions.ts`

**导出：**
- `generateFileSuggestions(query, cwd, ...) → Promise<SuggestionItem[]>` - 主入口。管理一个 `FileIndex` 单例（native Rust/nucleo），通过 `git ls-files` 拉取跟踪文件，失败时回退到 `ripgrep`。最多返回 15 条带分数的匹配。
- `startBackgroundCacheRefresh(cwd)` - 排队在后台刷新未跟踪文件。
- `clearFileSuggestionCaches()` - 在 `/clear` 时调用，重置索引。
- `applyFileSuggestion(suggestion, inputValue, cursorOffset) → string` - 把输入里的 `@token` 替换成选中的文件路径。
- `findLongestCommonPrefix(suggestions) → string` - 用于 Tab 自动补全公共前缀。
- `onIndexBuildComplete(callback)` - 当后台索引构建完成时通知。

**关键设计：**
- 路径签名（mtime + size）会让缓存失效，但不会触发完整重建
- 支持 `.ignore` / `.rgignore`
- 对 `@dir/` 补全做目录名提取

---

### `unifiedSuggestions.ts`

**文件：** `hooks/unifiedSuggestions.ts`

**导出：**
- `generateUnifiedSuggestions(query, mcpResources, agents, showOnEmpty) → Promise<SuggestionItem[]>` - 把文件建议（nucleo）、MCP resource 建议（Fuse.js）和 agent 建议合并成一个最多 15 条的排序列表。

**关键设计：**
- 文件建议使用 nucleo 分数（0–1 浮点）
- MCP resource 建议使用 Fuse.js 分数（越低越好，排序时会反转）
- agent 建议始终放在较低优先级

---

### `renderPlaceholder.ts`

**文件：** `hooks/renderPlaceholder.ts`

**导出：**
- `renderPlaceholder(placeholder, hidePlaceholderText?, cursorChar?) → string` - 一个纯函数，用来渲染带 cursor 字符的 placeholder 文本。`hidePlaceholderText = true`（语音录音模式）时，只返回 cursor。

