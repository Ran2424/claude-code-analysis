# Claude Code - 组件：核心与消息

本文档覆盖 `src/components/`（顶层文件）和 `src/components/messages/`（包括 `UserToolResultMessage/` 子目录）里的所有组件。

---

## 目录

1. [架构总览](#架构总览)
2. [顶层组件](#顶层组件)
3. [messages/ 子目录](#messages-子目录)
4. [messages/UserToolResultMessage/ 子目录](#messagesusertoolresultmessage-子目录)

---

## 架构总览

所有组件都通过 **React Compiler**（`react/compiler-runtime`）编译。几乎每个组件里都会出现 `_c(N)` 缓存分配器和 `Symbol.for("react.memo_cache_sentinel")` 守卫模式，这是一种自动记忆化，不是手写优化。

UI 框架是 **Ink**（终端 React 渲染器）。所有组件里常见的 Ink 原语包括：

- `Box`, `Text` - 布局和文本
- `useInput`, `useTheme`, `useTerminalFocus`, `useAnimationFrame` - Ink hooks
- `Ansi`, `RawAnsi`, `NoSelect`, `Link` - 特殊渲染节点
- `ScrollBox`, `ScrollBoxHandle` - 可滚动区域

特性开关通过 `bun:bundle` 里的 `feature('FLAG_NAME')` 在编译期求值。外部构建里不会保留死代码。

全局状态通过 `src/state/AppState.js` 里的 `useAppState`、`useSetAppState`、`useAppStateStore` 访问。

---

## 顶层组件

### App.tsx

**用途：** 顶层 React provider 容器，把所有全局上下文 provider 嵌套起来。

**导出：** `App`

**参数：**

| 属性 | 类型 | 是否必需 | 说明 |
|---|---|---|---|
| `getFpsMetrics` | `() => FpsMetrics \| undefined` | yes | 为 FpsMetricsProvider 提供 FPS 指标 |
| `stats` | `StatsStore` | no | StatsProvider 使用的统计存储 |
| `initialState` | `AppState` | yes | 传给 AppStateProvider 的初始状态 |
| `children` | `React.ReactNode` | yes | provider 内部渲染的内容 |

**Provider 嵌套顺序：** `FpsMetricsProvider > StatsProvider > AppStateProvider`

---

### AgentProgressLine.tsx

**用途：** 渲染 coordinator agent 进度树中的单行，显示类型/名称标签、状态文本、工具使用次数和 token 数。

**导出：** `AgentProgressLine`

**参数：** 传入 `agentType`、`description`、`name`、`descriptionColor`、`taskDescription`、`toolUseCount`、`tokens`、`color`、`isLast`、`isResolved`、`isError`、`isAsync`、`shouldAnimate`、`lastToolInfo`、`hideType` 等。

**关键行为：** 使用树状连接符 `└─` 或 `├─`。显示工具次数后缀和 token 数。

---

### ApproveApiKey.tsx

**用途：** 询问用户是否批准环境里发现的自定义 API key。

**导出：** `ApproveApiKey`

**参数：** `customApiKeyTruncated`, `onDone`

**关键行为：** 根据用户选择，把结果写入 `globalConfig.customApiKeyResponses.approved` 或 `.rejected`。

---

### AutoModeOptInDialog.tsx

**用途：** 让用户选择是否进入或退出 auto mode（完整 agentic 模式）。包含法律审查过的说明文本。

**导出：** `AUTO_MODE_DESCRIPTION`, `AutoModeOptInDialog`

**参数：** `onAccept`, `onDecline`, `declineExits`

**关键行为：** 提供三种选择：接受默认值（设置 `defaultMode:'auto'`）、接受、拒绝。会记录一组埋点事件。

---

### AutoUpdater.tsx

**用途：** 基于 npm 的自动更新器。每 30 分钟轮询 GCS，检查更高版本并安装。

**导出：** `AutoUpdater`

**参数：** `isUpdating`, `onChangeIsUpdating`, `onAutoUpdaterResult`, `autoUpdaterResult`, `showSuccessMessage`, `verbose`

**关键状态：** `versions: { global?, latest? }`, `hasLocalInstall: boolean`

**关键行为：** 检查 `maxVersion` 熔断开关；读取 `installationType` 决定是否运行；按 30 分钟间隔轮询。

---

### AutoUpdaterWrapper.tsx

**用途：** 根据安装方式把自动更新逻辑路由到对应 updater（包管理器、原生安装器或 npm）。

**导出：** `AutoUpdaterWrapper`

**参数：** 同 `AutoUpdater`

**关键状态：** `useNativeInstaller: boolean | null`, `isPackageManager: boolean | null`

**关键行为：** 按 Claude Code 的安装方式渲染 `PackageManagerAutoUpdater`、`NativeAutoUpdater` 或 `AutoUpdater`。

---

### AwsAuthStatusBox.tsx

**用途：** 当 AWS 认证正在进行或出错时，显示一个带边框的 “Cloud Authentication” 状态框。

**导出：** `AwsAuthStatusBox`

**参数：** 无（从 `AwsAuthStatusManager` 单例读取）

**关键行为：** 只有在 `isAuthenticating` 或存在错误时才渲染。

---

### BaseTextInput.tsx

**用途：** `TextInput` 和 `VimTextInput` 共享的低层文本输入组件，负责光标、粘贴和高亮渲染。

**导出：** `BaseTextInput`

**参数：** `BaseTextInputProps` 加上 `inputState`、`children?`、`terminalFocus`、`highlights?`、`invert?`、`hidePlaceholderText?`

**关键行为：** 使用 `useDeclaredCursor`、`usePasteHandler` 和 `renderPlaceholder`。

---

### BashModeProgress.tsx

**用途：** 渲染进行中的 bash 命令界面：输入消息和流式进度输出。

**导出：** `BashModeProgress`

**参数：** `input`, `progress`, `verbose`

**关键行为：** 渲染 `UserBashInputMessage` + `ShellProgressMessage`，或回退到 `BashTool.renderToolUseProgressMessage`。

---

### BridgeDialog.tsx

**用途：** 远程 bridge 设置对话框。显示二维码和分支名，供移动端/远程访问。

**导出：** `BridgeDialog`

**参数：** `onDone`

**关键状态：** `showQR`, `qrText`, `branchName`

**关键行为：** 使用 `qrcode` 库渲染二维码，从 AppState 读取 bridge 状态。

---

### BypassPermissionsModeDialog.tsx

**用途：** 在使用 `--dangerously-skip-permissions` 时显示确认对话框，需要用户明确接受。

**导出：** `BypassPermissionsModeDialog`

**参数：** `onAccept`

**关键行为：** Esc 会触发 `gracefulShutdownSync(0)`，拒绝则 `gracefulShutdownSync(1)`。

---

### ChannelDowngradeDialog.tsx

**用途：** 当当前安装版本相对其他 release channel 是降级时，显示提示对话框。

**导出：** `ChannelDowngradeChoice`, `ChannelDowngradeDialog`

**参数：** `currentVersion`, `onChoice`

---

### ClickableImageRef.tsx

**用途：** 把图片引用（通过 `imageId`）渲染成可点击链接；支持 OSC 8 的终端会显示超链接，不支持时退回为样式化文本。

**导出：** `ClickableImageRef`

**参数：** `imageId`, `backgroundColor`, `isSelected`

**关键行为：** 通过 `pathToFileURL` + `supportsHyperlinks()` 生成 OSC 8 链接。

---

### ClaudeInChromeOnboarding.tsx

**用途：** Claude in Chrome 浏览器扩展开箱引导。展示安装状态，并把接受结果写入全局配置。

**导出：** `ClaudeInChromeOnboarding`

**参数：** `onDone`

**关键状态：** `isExtensionInstalled: boolean`

---

### ClaudeMdExternalIncludesDialog.tsx

**用途：** 警告用户 `CLAUDE.md` 里通过 `@path` 引入的外部文件，用户必须批准后才能继续。

**导出：** `ClaudeMdExternalIncludesDialog`

**参数：** `onDone`, `isStandaloneDialog`, `externalIncludes`

**关键行为：** 把是否已批准和是否已显示警告写入 project config。

---

### CompactSummary.tsx

**用途：** 渲染一个视觉分隔卡片，标记对话被 compact 后的边界。

**导出：** `CompactSummary`

**参数：** `message`, `screen`

**关键行为：** 显示 `messagesSummarized`、方向和 `userContext` 等元数据。

---

### ConfigurableShortcutHint.tsx

**用途：** 以用户配置的按键绑定渲染快捷键提示；如果没绑定，就回退到字面字符串。

**导出：** `ConfigurableShortcutHint`

**参数：** `action`, `context`, `fallback`, `description`, `parens`, `bold`

---

### ConsoleOAuthFlow.tsx

**用途：** claude.ai/console 认证的完整 OAuth 登录流程，管理多步状态机。

**导出：** `ConsoleOAuthFlow`

**参数：** `onDone`, `startingMessage`, `mode`, `forceLoginMethod`

**关键状态：** `OAuthStatus`，包含 `idle`、`platform_setup`、`ready_to_start`、`waiting_for_login`、`creating_api_key`、`about_to_retry`、`success`、`error`

---

### ContextSuggestions.tsx

**用途：** 渲染一组节省上下文的建议，例如“通过添加 `.gitignore` 规则减少 X 个 token”。

**导出：** `ContextSuggestions`

**参数：** `suggestions`

---

### ContextVisualization.tsx

**用途：** 可视化上下文窗口使用情况，并在 `CONTEXT_COLLAPSE` 特性开关开启时显示折叠状态。

**导出：** `ContextVisualization`

**关键行为：** 内部的 `CollapseStatus` 子组件受 `feature('CONTEXT_COLLAPSE')` 保护。

---

### CoordinatorAgentStatus.tsx

**用途：** 在侧边栏显示 coordinator/swarm agent 进度面板。

**导出：** `getVisibleAgentTasks(tasks): Task[]`, `CoordinatorTaskPanel`

---

### CostThresholdDialog.tsx

**用途：** 提示用户已经在 API 调用上花了 5 美元。

**导出：** `CostThresholdDialog`

**参数：** `onDone`

---

### CtrlOToExpand.tsx

**用途：** 渲染灰色提示“ctrl+o to expand”，用于可折叠内容；并提供 React context 防止嵌套提示重复显示。

**导出：** `SubAgentProvider`, `CtrlOToExpand`, `ctrlOToExpand(): string`

**关键行为：** `SubAgentContext` 为 `React.createContext(false)`，避免子智能体输出里重复显示提示；`ctrlOToExpand()` 在非 React 场景下返回 `chalk.dim` 字符串。

---

### DesktopHandoff.tsx

**用途：** 管理“在 Claude Desktop 中打开”的交接流程。先检查 Desktop 是否已安装；未安装则下载，再打开。

**导出：** `getDownloadUrl(): string`, `DesktopHandoff`

**参数：** `onDone`

**关键状态：** `DesktopHandoffState = 'checking' | 'prompt-download' | 'flushing' | 'opening' | 'success' | 'error'`

---

### DevBar.tsx

**用途：** 内部开发栏，显示慢操作信息（只在 dev/ant 构建中出现）。

**导出：** `DevBar`

**参数：** 无

**关键状态：** `slowOps`，每 500ms 轮询一次，显示最近 3 个慢操作。

---

### DiagnosticsDisplay.tsx

**用途：** 渲染文件里的诊断问题，例如 TypeScript 错误、lint 警告等。

**导出：** `DiagnosticsDisplay`

**参数：** `attachment`, `verbose`

**关键行为：** 非 verbose 模式显示“Found **N** new diagnostic issue(s) in N file(s)”加 `CtrlOToExpand`；verbose 模式显示逐文件细节。

---

### EffortCallout.tsx

**用途：** 当启用非默认 effort 等级时显示提示，例如“max thinking”模式。

**导出：** `EffortCallout`

**关键行为：** 受 feature gate 控制，显示 effort 符号和描述。

---

### EffortIndicator.ts

**用途：** 非组件工具模块，提供 effort 相关的显示辅助。

**导出：**

| 导出 | 签名 | 说明 |
|---|---|---|
| `getEffortNotificationText` | `(effortValue: any, model: string): string \| undefined` | 返回 effort 等级变化的通知文本 |
| `effortLevelToSymbol` | `(level: EffortLevel): string` | 把 effort 等级映射为显示符号 |

---

### ExitFlow.tsx

**用途：** 负责退出流程。如果当前在 worktree 中，就显示 worktree 清理对话框。

**导出：** `ExitFlow`

**参数：** `onDone`, `onCancel`, `showWorktree`

**关键行为：** 当 `showWorktree` 为 true 时渲染 `WorktreeExitDialog`，否则返回 null。

---

### ExportDialog.tsx

**用途：** 把对话内容导出到剪贴板或文件的对话框。

**导出：** `ExportDialog`

**参数：** `content`, `defaultFilename`, `onDone`

**关键状态：** `ExportOption = 'clipboard' | 'file'`

**关键行为：** 当选择“file”时显示文件名 `TextInput`。

---

### FallbackToolUseErrorMessage.tsx

**用途：** 当没有工具专用渲染器时，为工具调用结果渲染错误消息。

**导出：** `FallbackToolUseErrorMessage`

**参数：** `result`, `verbose`

**关键行为：** `MAX_RENDERED_LINES = 10`。会去掉 underline ANSI、sandbox violation/error XML 标签；如果被截断，显示“+N lines (ctrl+o to see all)”提示。

---

### FallbackToolUseRejectedMessage.tsx

**用途：** 当没有工具专用渲染器时，渲染“Interrupted · What should Claude do instead?” 这类被拒绝工具调用消息。

**导出：** `FallbackToolUseRejectedMessage`

**参数：** 无

**关键行为：** 把 `InterruptedByUser` 包到 `MessageResponse` 里，height=1。

---

### FastIcon.tsx

**用途：** 渲染 fast mode 的闪电图标（⚡），并在 cooldown 时显示为更淡的颜色。

**导出：** `FastIcon`, `getFastIconString(applyColor?: boolean, cooldown?: boolean): string`

**参数：** `cooldown`

---

### Feedback.tsx

**用途：** 完整的反馈提交表单。收集描述、可选转录文本，可选调用 Haiku 做 AI 辅助分类，然后打开 GitHub issue。

**导出：** `redactSensitiveInfo(text: string): string`, `Feedback`

**参数：** `abortSignal`, `messages`, `initialDescription`, `onDone`, `backgroundTasks`

**关键状态：** `Step = 'userInput' | 'consent' | 'submitting' | 'done'`

**常量：** `GITHUB_URL_LIMIT = 7250`, `GITHUB_ISSUES_REPO_URL`

**关键行为：** `redactSensitiveInfo` 会在提交前用正则移除 API key（`sk-ant-...`）。

---

### FileEditToolDiff.tsx

**用途：** 渲染文件编辑 diff。通过 React `Suspense` 异步加载 diff 数据。

**导出：** `FileEditToolDiff`

**参数：** `file_path`, `edits`

---

### FileEditToolUpdatedMessage.tsx

**用途：** 用结构化 diff 显示文件编辑摘要（增加/删除行数）。

**导出：** `FileEditToolUpdatedMessage`

**参数：** `filePath`, `structuredPatch`, `firstLine`, `fileContent`, `style`, `verbose`, `previewHint`

---

### FileEditToolUseRejectedMessage.tsx

**用途：** 显示“User rejected write/update to file”消息，并预览被拒绝的 diff 或内容。

**导出：** `FileEditToolUseRejectedMessage`

**参数：** `file_path`, `operation`, `patch`, `firstLine`, `fileContent`, `content`, `style`, `verbose`

**常量：** `MAX_LINES_TO_RENDER = 10`

---

### FilePathLink.tsx

**用途：** 把绝对文件路径渲染成 OSC 8 超链接，供支持它的终端模拟器（例如 iTerm2）点击。

**导出：** `FilePathLink`

**参数：** `filePath`, `children`

**关键行为：** 用 `pathToFileURL` 转成 `file://` URL，再包到 Ink `Link`。

---

### FullscreenLayout.tsx

**用途：** 全屏模式下的主布局容器。管理可滚动区域、固定底部槽、覆盖层、模态面板和浮动内容。

**导出：** `ScrollChromeContext`, `FullscreenLayout`

**`ScrollChromeContext`：** `{ setStickyPrompt: (p: StickyPrompt | null) => void }`

**参数：** `scrollable`, `bottom`, `overlay`, `bottomFloat`, `modal`, `modalScrollRef`, `scrollRef`, `dividerYRef`, `hidePill`

**常量：** `MODAL_TRANSCRIPT_PEEK = 2`

---

### GlobalSearchDialog.tsx

**用途：** 全文 ripgrep 搜索对话框（ctrl+shift+f）。带防抖搜索和文件预览面板。

**导出：** `GlobalSearchDialog`

**参数：** `onDone`, `onInsert`

**关键状态：** `matches`, `truncated`, `isSearching`

**常量：** `VISIBLE_RESULTS = 12`, `DEBOUNCE_MS = 100`, `PREVIEW_CONTEXT_LINES = 4`, `MAX_MATCHES_PER_FILE = 10`, `MAX_TOTAL_MATCHES = 500`

**关键行为：** 使用 `useRegisterOverlay("global-search")`。当 `columns >= 140` 时在右侧预览。搜索通过 `ripGrepStream`。

---

### HighlightedCode.tsx

**用途：** 使用原生 Rust `ColorFile` 模块渲染语法高亮源码；不可用时回退到 `HighlightedCodeFallback`。

**导出：** `HighlightedCode`

**参数：** `code`, `filePath`, `width`, `dim`

**关键状态：** `measuredWidth`

**常量：** `DEFAULT_WIDTH = 80`

**关键行为：** 尊重 `settings.syntaxHighlightingDisabled`，通过 Rust `colorDiff` 模块里的 `expectColorFile()` 工作。

---

### HistorySearchDialog.tsx

**用途：** 支持模糊搜索的历史浏览器（ctrl+r）。异步加载所有带时间戳的历史条目，支持模糊匹配和预览。

**导出：** `HistorySearchDialog`

**参数：** `initialQuery`, `onSelect`, `onCancel`

**关键状态：** `items: Item[] | null`, `query: string`

**常量：** `PREVIEW_ROWS = 6`, `AGE_WIDTH = 8`

**关键行为：** 使用 `useRegisterOverlay('history-search')`，从 `getTimestampedHistory()` 异步生成器加载，显示层使用 `FuzzyPicker`。

---

### IdeAutoConnectDialog.tsx

**用途：** 首次运行时询问是否在启动时自动连接 IDE。

**导出：** `IdeAutoConnectDialog`

**参数：** `onComplete`

**关键行为：** 保存 `globalConfig.autoConnectIde` 和 `globalConfig.hasIdeAutoConnectDialogBeenShown = true`。

---

### IdeOnboardingDialog.tsx

**用途：** IDE 集成引导（VS Code、JetBrains 等）。展示安装状态和说明。

**导出：** `IdeOnboardingDialog`

**参数：** `onDone`, `installationStatus`

**关键行为：** 挂载时立即通过 `markDialogAsShown()` 标记已显示；响应 `confirm:yes` 和 `confirm:no` 快捷键。

---

### IdeStatusIndicator.tsx

**用途：** 在状态栏显示当前 IDE 选择状态，比如活动文件或选中的行数。

**导出：** `IdeStatusIndicator`

**参数：** `ideSelection`, `mcpClients`

**关键行为：** 使用 `useIdeConnectionStatus`。只有连接且有选择时才显示。文本形如 `⧉ N lines selected` 或 `⧉ In filename.ts`。

---

### IdleReturnDialog.tsx

**用途：** 用户从空闲状态回来时的对话框，提供继续、清空、关闭或以后不再显示等选项。

**导出：** `IdleReturnDialog`

**参数：** `idleMinutes`, `totalInputTokens`, `onDone`

**类型：** `IdleReturnAction = 'continue' | 'clear' | 'dismiss' | 'never'`

---

### InterruptedByUser.tsx

**用途：** 渲染“Interrupted · What should Claude do instead?” 文本。在 ant 构建里会显示不同消息。

**导出：** `InterruptedByUser`

**参数：** 无

---

### InvalidConfigDialog.tsx

**用途：** 当 Claude 配置文件里有无效 JSON 时显示的对话框。用户可以退出或重置配置。

**导出：** `InvalidConfigDialog`, `InvalidConfigHandlerProps`, `InvalidConfigDialogProps`

**`InvalidConfigDialogProps`：** `filePath`, `errorDescription`, `onExit`, `onReset`

**关键行为：** 还导出一个可在 React 树外使用的独立 `render()`。

---

### InvalidSettingsDialog.tsx

**用途：** 当 settings 文件有校验错误时显示的对话框。用户可以继续（跳过无效文件）或退出。

**导出：** `InvalidSettingsDialog`

**参数：** `settingsErrors`, `onContinue`, `onExit`

---

### KeybindingWarnings.tsx

**用途：** 显示 keybinding 校验警告/错误。只有在启用 keybinding 自定义时才显示（ant + feature gate）。

**导出：** `KeybindingWarnings`

**参数：** 无

**关键行为：** 调用 `isKeybindingCustomizationEnabled()`；按严重级别分组；通过 `getKeybindingsPath()` 显示文件路径。

---

### LanguagePicker.tsx

**用途：** 选择首选响应语言的文本输入框。

**导出：** `LanguagePicker`

**参数：** `initialLanguage`, `onComplete`, `onCancel`

**关键状态：** `language`, `cursorOffset`

---

### LogSelector.tsx

**用途：** 功能完整的会话日志浏览器，带模糊搜索、标签过滤、agentic 搜索和会话预览。

**导出：** `LogSelectorProps`, `LogSelector`

**参数：** `logs`, `maxHeight`, `forceWidth`, `onCancel`, `onSelect`, `onLogsChanged`, `onLoadMore`, `initialSearchQuery`, `showAllProjects`, `onToggleAllProjects`, `onAgenticSearch`

**内部类型：** `AgenticSearchState`, `LogTreeNode`

---

### MarkdownTable.tsx

**用途：** 渲染 Markdown 表格 token，按 ANSI 感知的列宽计算和换行。遇到宽内容时会切成纵向 key-value 格式。

**导出：** `MarkdownTable`

**参数：** `token`, `highlight`, `forceWidth`

**常量：** `SAFETY_MARGIN = 4`, `MIN_COLUMN_WIDTH = 3`, `MAX_ROW_LINES = 4`

---

### Markdown.tsx

**用途：** 使用 `marked`（GFM 模式）渲染 Markdown 文本。内置 LRU token cache 和纯文本快路径。

**导出：** `Markdown`

**参数：** `children`, `dimColor`

**关键行为：** 模块级 LRU token cache 最多 500 条；如果是纯文本就跳过 `marked.lexer`。

---

### MemoryUsageIndicator.tsx

**用途：** 显示高/临界堆内存警告，并提供 `/heapdump` 链接。只在 ant 内部构建里出现，外部构建返回 `null`。

**导出：** `MemoryUsageIndicator`

**参数：** 无

**关键行为：** 外部构建直接返回 null；使用 `useMemoryUsage()` hook（10 秒轮询）；根据状态显示警告/错误颜色。

---

### Message.tsx

**用途：** 中央消息分发器。把每条消息/内容块路由到对应渲染组件。

**导出：** `hasThinkingContent(message): boolean`, `Message`

**参数：** `message`, `lookups`, `containerWidth`, `addMargin`, `tools`, `commands`, `verbose`, `inProgressToolUseIDs`, `progressMessagesForMessage`, `shouldAnimate`, `shouldShowDot`, `style`, `width`, `isTranscriptMode`, `isStatic`, `onOpenRateLimitOptions`, `isActiveCollapsedGroup`, `isUserContinuation`, `lastThinkingBlockId`, `latestBashOutputUUID`

---

### MessageModel.tsx

**用途：** 在 transcript 模式下显示 assistant 消息的模型标识。

**导出：** `MessageModel`

**参数：** `message`, `isTranscriptMode`

**关键行为：** 只有在 assistant 消息带 `model` 字段且有文本内容块时才渲染。

---

### MessageResponse.tsx

**用途：** 给 assistant 响应内容包上 `⎿` 前缀。使用 `MessageResponseContext` 避免嵌套前缀重复。

**导出：** `MessageResponse`

**参数：** `children`, `height`

**关键行为：** 通过 `NoSelect` 渲染 `⎿`；如果未指定 `height`，则包一层 `Ratchet`。

---

### MessageRow.tsx

**用途：** 渲染单条消息行，支持 OffscreenFreeze、模型指示器和时间戳。

**导出：** `hasContentAfterIndex(messages, index, tools, streamingToolUseIDs): boolean`, `Props`, `MessageRow`

**参数：** `message`, `isUserContinuation`, `hasContentAfter`, `tools`, `commands`, `verbose`, `inProgressToolUseIDs`, `streamingToolUseIDs`, `screen`, `canAnimate`, `onOpenRateLimitOptions`, `lastThinkingBlockId`, `latestBashOutputUUID`, `columns`, `isLoading`, `lookups`

---

### MessageSelector.tsx

**用途：** 让用户选择一个历史消息点，用来 rewind/restore。

**导出：** `MessageSelector`

**参数：** `messages`, `onPreRestore`, `onRestoreMessage`, `onRestoreCode`, `onSummarize`, `onClose`, `preselectedMessage`

**类型：** `RestoreOption = 'both' | 'conversation' | 'code' | 'summarize' | 'summarize_up_to' | 'nevermind'`

**常量：** `MAX_VISIBLE_MESSAGES = 7`

---

### MessageTimestamp.tsx

**用途：** 在 transcript 模式下显示 assistant 消息的格式化时间戳。

**导出：** `MessageTimestamp`

**参数：** `message`, `isTranscriptMode`

**关键行为：** 只对带文本内容的 assistant 消息渲染，格式为 `HH:MM AM/PM`。

---

### Messages.tsx

**用途：** 顶层会话视图组件。负责规范化消息、折叠 read/search 组、构建 lookups，并驱动虚拟滚动列表或静态列表。

**导出：** `shouldRenderStatically(screen: Screen): boolean`, `Messages`

**关键行为：** 包含 `LogoHeader = React.memo(...)`，并注明 blit 优化（必须在消息前渲染才能保证正确滚动行为）。会在 200 条消息渲染预算前过滤掉不产生可见输出的 attachments。transcript 模式下用 `VirtualMessageList` + `JumpHandle`，REPL 模式下用静态列表。

---

### ModelPicker.tsx

**用途：** 模型选择器，带 effort 等级开关。支持 fast mode 感知和会话级/全局设置。

**导出：** `Props`, `ModelPicker`

**参数：** `initial`, `sessionModel`, `onSelect`, `onCancel`, `isStandaloneCommand`, `showFastModeNotice`, `headerText`, `skipSettingsWrite`

**常量：** `NO_PREFERENCE = '__NO_PREFERENCE__'`

---

### NativeAutoUpdater.tsx

**用途：** 原生安装器构建的自动更新器。检测到新版本时调用 `nativeInstaller` 里的 `installLatest()`。

**导出：** `NativeAutoUpdater`

**参数：** `isUpdating`, `onChangeIsUpdating`, `onAutoUpdaterResult`, `autoUpdaterResult`, `showSuccessMessage`, `verbose`

**关键状态：** `versions: { current?: string | null; latest?: string | null }`

---

### NotebookEditToolUseRejectedMessage.tsx

**用途：** 显示“User rejected replace/insert/delete cell in notebook_path”并带代码预览。

**导出：** `NotebookEditToolUseRejectedMessage`

**参数：** `notebook_path`, `cell_id`, `new_source`, `cell_type`, `edit_mode`, `verbose`

---

### OffscreenFreeze.tsx

**用途：** 当内容滚到终端视口上方进入 scrollback 时，冻结子元素，以提升性能。

**导出：** `OffscreenFreeze`

**参数：** `children`

**关键行为：** 使用 `'use no memo'` 指令退出 React Compiler；通过 `useTerminalViewport` 判断可见性；把最后一次可见渲染缓存到 `useRef`；当 `inVirtualList` 为 true 时，冻结会被绕过。

---

### Onboarding.tsx

**用途：** 多步骤首次运行引导：预检、主题选择、OAuth、API key 许可、安全审查、终端设置。

**导出：** `Onboarding`

**参数：** `onDone`

**关键状态：** `currentStepIndex`, `skipOAuth`, `oauthEnabled`, `theme`

**步骤：** `StepId = 'preflight' | 'theme' | 'oauth' | 'api-key' | 'security' | 'terminal-setup'`

---

### OutputStylePicker.tsx

**用途：** 选择当前输出风格（default、concise、detailed，以及 `.claude/output-styles/` 里的自定义风格）。

**导出：** `OutputStylePickerProps`, `OutputStylePicker`

**参数：** `initialStyle`, `onComplete`, `onCancel`, `isStandaloneCommand`

**关键状态：** `styleOptions`, `isLoading`

---

### PackageManagerAutoUpdater.tsx

**用途：** 当 Claude Code 通过包管理器（brew、pip 等）安装时，提示有更新可用。

**导出：** `PackageManagerAutoUpdater`

**参数：** 与 `NativeAutoUpdater` 相同

**关键状态：** `updateAvailable`, `packageManager`

**关键行为：** 只显示通知，不会自动安装。使用 `MACRO.VERSION`（编译期常量）。

---

### PrBadge.tsx

**用途：** 渲染带 review 状态颜色的 PR 编号徽章。

**导出：** `PrBadge`

**参数：** `number`, `url`, `reviewState`, `bold`

---

### PressEnterToContinue.tsx

**用途：** 简单的“Press **Enter** to continue…” 提示，带权限颜色样式。

**导出：** `PressEnterToContinue`

**参数：** 无

---

### QuickOpenDialog.tsx

**用途：** 快速打开的模糊文件查找器（ctrl+shift+p）。会显示文件结果和语法高亮预览。

**导出：** `QuickOpenDialog`

**参数：** `onDone`, `onInsert`

**关键状态：** `results`, `query`, `focusedPath`, `preview`

**常量：** `VISIBLE_RESULTS = 8`, `PREVIEW_LINES = 20`

---

### RemoteCallout.tsx

**用途：** 一次性的提示对话框，建议用户开启 Remote Control（bridge）。挂载时会写入 `remoteDialogSeen = true`。

**导出：** `RemoteCallout`

**参数：** `onDone`

**类型：** `RemoteCalloutSelection = 'enable' | 'dismiss'`

---

### RemoteEnvironmentDialog.tsx

**用途：** 远程 Teleport 环境选择器（claude.ai/code environments）。

**导出：** `RemoteEnvironmentDialog`

**参数：** `onDone`

**关键状态：** `loadingState`, `environments`, `selectedEnvironment`, `selectedEnvironmentSource`, `error`

---

### ResumeTask.tsx

**用途：** 列出可恢复的远程 Claude Code 会话（来自 Sessions API），并按当前 git 仓库过滤。

**导出：** `ResumeTask`

**参数：** `onSelect`, `onCancel`, `isEmbedded`

**关键状态：** `sessions`, `currentRepo`, `loading`, `loadErrorType`, `retrying`, `focusedIndex`

---

### SandboxViolationExpandedView.tsx

**用途：** 显示最近的 sandboxing 违规记录（最近 10 条）。

**导出：** `SandboxViolationExpandedView`

**参数：** 无

**关键状态：** `violations`, `totalCount`

**关键行为：** 当 sandboxing 被禁用或在 Linux 上时返回 null。

---

### ScrollKeybindingHandler.tsx

**用途：** 处理滚动键盘输入，包括 j/k/方向键、PageUp/PageDown、g/G、ctrl+u/d/b/f；同时支持更平滑的滚轮加速。

**导出：** `ScrollKeybindingHandler`

**参数：** `scrollRef`, `isActive`, `onScroll`, `isModal`

**常量：** `WHEEL_ACCEL_WINDOW_MS = 40`, `WHEEL_ACCEL_STEP = 0.3`, `WHEEL_ACCEL_MAX = 6`

---

### SearchBox.tsx

**用途：** 带光标显示和占位符的样式化搜索输入框。

**导出：** `SearchBox`

**参数：** `query`, `placeholder`, `isFocused`, `isTerminalFocused`, `prefix`, `width`, `cursorOffset`, `borderless`

---

### SentryErrorBoundary.ts

**用途：** 一个 React error boundary，遇到渲染错误时静默吞掉并返回 `null`。

**导出：** `SentryErrorBoundary`

**参数：** `{ children: React.ReactNode }`

**关键行为：** 类组件；通过 `getDerivedStateFromError` 捕获渲染错误并返回 null。

---

### SessionBackgroundHint.tsx

**用途：** 显示提示，并处理 Ctrl+B 双击模式，把当前会话放到后台。

**导出：** `SessionBackgroundHint`

**参数：** `onBackgroundSession`, `isLoading`

**关键状态：** `showSessionHint: boolean`

---

### SessionPreview.tsx

**用途：** 用完整 `Messages` 组件渲染历史会话日志的只读预览。

**导出：** `SessionPreview`

**参数：** `log`, `onExit`, `onSelect`

**关键状态：** `fullLog: LogOption | null`

---

### ShowInIDEPrompt.tsx

**用途：** 当 diff 已在 IDE 中打开时，显示“Opened changes in {IDE}”确认面板，提供 Yes/No 选项。

**导出：** `ShowInIDEPrompt`

**参数：** `filePath`, `input`, `onChange`, `options`, `ideName`, `symlinkTarget`, `rejectFeedback`, `acceptFeedback`, `setFocusedOption`, `onInputModeToggle`, `focusedOption`, `yesInputMode`, `noInputMode`

---

### SkillImprovementSurvey.tsx

**用途：** 在 skill 执行后弹出调查，询问这次 skill 改进是否有帮助。

**导出：** `SkillImprovementSurvey`

**参数：** `isOpen`, `skillName`, `updates`, `handleSelect`, `inputValue`, `setInputValue`

---

### Spinner.tsx

**用途：** 重新导出主动画 Spinner 组件以及 `SpinnerMode` 类型。

**导出：** `SpinnerMode`, `Spinner`

**参数：** `mode`, `loadingStartTimeRef`, `totalPausedMsRef`, `pauseStartTimeRef`, `spinnerTip`, `responseLengthRef`, `overrideColor`, `overrideShimmerColor`

---

### Stats.tsx

**用途：** 全屏使用统计查看器，包含日期范围 tab、token 分布、模型热力图和 ASCII 图表。

**导出：** `Stats`

**参数：** `onClose`

**类型：** `StatsResult = { type: 'success'; data: ClaudeCodeStats } | { type: 'error'; message: string } | { type: 'empty' }`

**常量：** `DATE_RANGE_LABELS: Record<StatsDateRange, string>`（`7d`, `30d`, `90d`）

---

### StatusLine.tsx

**用途：** 底部状态栏，显示模型、权限模式、上下文使用量、worktree 和会话信息。

**导出：** `statusLineShouldDisplay(settings): boolean`, `StatusLine`

**关键行为：** 从 AppState/settings 中读取模型、权限模式、上下文使用量、worktree 状态和会话信息。

---

### StatusNotices.tsx

**用途：** 渲染启动时的 active notice，例如弃用警告和 MCP 错误。使用 React `use()` 异步加载 memory 文件。

**导出：** `StatusNotices`

**参数：** `agentDefinitions`

**关键行为：** 调用 `getActiveNotices(context)`；如果没有 active notices 就返回 null。

---

### StructuredDiff.tsx

**用途：** 渲染单个 diff hunk，并通过 Rust `ColorDiff` NAPI 模块做语法高亮。模块级缓存（WeakMap）可在组件卸载后保留结果。

**导出：** `StructuredDiff`

**参数：** `patch`, `dim`, `filePath`, `firstLine`, `fileContent`, `width`, `skipHighlighting`

---

### StructuredDiffList.tsx

**用途：** 渲染一组 diff hunks（来自 `StructuredDiff`），并用省略号分隔。

**导出：** `StructuredDiffList`

**参数：** `hunks`, `dim`, `width`, `filePath`, `firstLine`, `fileContent`

---

### TagTabs.tsx

**用途：** 会话日志标签过滤的水平 tab bar，带溢出处理和“← N” / “→ (tab to cycle)” 提示。

**导出：** `TagTabs`

**参数：** `tabs`, `selectedIndex`, `availableWidth`, `showAllProjects`

**常量：** `ALL_TAB_LABEL = 'All'`, `TAB_PADDING = 2`, `MAX_OVERFLOW_DIGITS = 2`

---

### TextInput.tsx

**用途：** 功能完整的文本输入框，带语音录制波形光标动画、剪贴板粘贴提示，以及 vim/normal 模式路由。

**导出：** `Props`, `TextInput`

**关键行为：** 使用 `feature('VOICE_MODE')` 控制语音录制波形光标；波形通过指数移动平均平滑；`BaseTextInput` 由 `useTextInput` hook 驱动。

**常量：** `BARS = ' ▁▂▃▄▅▆▇█'`, `CURSOR_WAVEFORM_WIDTH = 1`, `SMOOTH = 0.7`, `LEVEL_BOOST = 1.8`, `SILENCE_THRESHOLD = 0.15`

---

### ThemePicker.tsx

**用途：** 主题选择 UI，带语法高亮 diff 片段的实时预览。

**导出：** `ThemePickerProps`, `ThemePicker`

**参数：** `onThemeSelect`, `showIntroText`, `helpText`, `showHelpTextBelow`, `hideEscToCancel`, `skipExitHandling`, `onCancel`

---

### ThinkingToggle.tsx

**用途：** 切换 extended thinking 的选择器；在对话中途切换时会弹出确认提示。

**导出：** `Props`, `ThinkingToggle`

**参数：** `currentValue`, `onSelect`, `onCancel`, `isMidConversation`

**关键状态：** `confirmationPending: boolean | null`

---

### TokenWarning.tsx

**用途：** 显示上下文窗口使用警告；开启 `CONTEXT_COLLAPSE` 时会显示实时折叠进度。

**导出：** `TokenWarning`

**参数：** `tokenUsage`, `model`

**关键行为：** 内部 `CollapseLabel` 通过 `useSyncExternalStore` 订阅折叠统计存储。

---

### ToolUseLoader.tsx

**用途：** 工具调用状态点（●）动画。未完成时闪烁，成功为绿，错误为红。

**导出：** `ToolUseLoader`

**参数：** `isError`, `isUnresolved`, `shouldAnimate`

**关键行为：** 使用 `useBlink`。未完成时颜色为 undefined（dim），出错时为 `error`，成功时为 `success`。对 dim+bold 的 ANSI reset 交互比较敏感。

---

### ValidationErrorsList.tsx

**用途：** 使用点号路径把 settings 校验错误渲染成树状列表。

**导出：** `ValidationErrorsList`

**参数：** `errors`

**关键行为：** 用 `lodash-es/setWith` 把点号路径构造成嵌套树，再用 `treeify` 工具渲染；数组索引会同时显示值，方便阅读。

---

### VimTextInput.tsx

**用途：** Vim 模式文本输入。把 `BaseTextInput` 和 `useVimInput` 的状态管理包起来。

**导出：** `Props`, `VimTextInput`

**关键行为：** 通过 `useVimInput` 传递完整 vim 输入 props；当终端获得焦点时，invert 函数会用 `chalk.inverse`。

---

### VirtualMessageList.tsx

**用途：** 虚拟化的可滚动消息列表，支持搜索高亮、跳转导航和 sticky prompt。

**导出：** `StickyPrompt`, `JumpHandle`, `VirtualMessageList`

**`JumpHandle` 接口：**

| 方法 | 说明 |
|---|---|
| `jumpToIndex(index: number)` | 滚动到指定消息索引 |
| `setSearchQuery(query: string)` | 设置文本搜索查询 |
| `nextMatch()` | 跳到下一个匹配 |
| `prevMatch()` | 跳到上一个匹配 |
| `setAnchor(index: number)` | 设置滚动锚点 |
| `warmSearchIndex()` | 预热搜索索引 |
| `disarmSearch()` | 清除搜索状态 |

**参数：** `messages`, `scrollRef`, `columns`, `itemKey`, `renderItem`, `onItemClick`

---

### WorkflowMultiselectDialog.tsx

**用途：** 选择要通过 GitHub App 集成安装的 GitHub Actions workflow 的多选对话框。

**导出：** `WorkflowMultiselectDialog`

**参数：** `onSubmit`, `defaultSelections`

**Workflows：** `claude`（@Claude Code tag）、`claude-review`（自动 PR review）

---

### WorktreeExitDialog.tsx

**用途：** 从 git worktree 会话退出时显示的对话框。询问是保留 worktree、清理它，还是导出 commit。

**导出：** `WorktreeExitDialog`

**参数：** `onDone`, `onCancel`

**关键状态：** `status`, `changes`, `commitCount`, `resultMessage`

**关键行为：** 读取 git status 和 worktree commit 数；使用 `cleanupWorktree` / `keepWorktree` / `killTmuxSession`；为了避免循环导入，会延迟 require `sessionStorage`。

---

### messages/ 子目录

---

### messages/AdvisorMessage.tsx

**用途：** 渲染 advisor（内部 assistant）的 server tool use 块，带状态指示器和可选 JSON 输入。

**导出：** `AdvisorMessage`

**参数：** `block`, `addMargin`, `resolvedToolUseIDs`, `erroredToolUseIDs`, `shouldAnimate`, `verbose`, `advisorModel`

---

### messages/AssistantRedactedThinkingMessage.tsx

**用途：** 渲染被遮盖 thinking 块的占位符（“✻ Thinking…”）。

**导出：** `AssistantRedactedThinkingMessage`

**参数：** `addMargin`

---

### messages/AssistantTextMessage.tsx

**用途：** 渲染 assistant 文本回复块。处理多种特殊 API 错误字符串常量（限流消息、overload 字符串等）。

**导出：** `AssistantTextMessage`

**参数：** `param`, `addMargin`, `shouldShowDot`, `verbose`, `width`, `onOpenRateLimitOptions`

---

### messages/AssistantThinkingMessage.tsx

**用途：** 渲染 thinking 块，在 transcript/verbose 模式下要么显示“∴ Thinking (ctrl+o to expand)”摘要，要么显示完整内容。

**导出：** `AssistantThinkingMessage`

**参数：** `param`, `addMargin`, `isTranscriptMode`, `verbose`, `hideInTranscript`

---

### messages/AssistantToolUseMessage.tsx

**用途：** 渲染工具调用块。优先路由到工具自己的 `renderToolUse` 方法，否则回退。

**导出：** `AssistantToolUseMessage`

**参数：** `param`, `addMargin`, `tools`, `commands`, `verbose`, `inProgressToolUseIDs`, `progressMessagesForMessage`, `shouldAnimate`, `shouldShowDot`, `inProgressToolCallCount`, `lookups`, `isTranscriptMode`

---

### messages/AttachmentMessage.tsx

**用途：** 按 `attachment.type` 把附件消息路由到专门的渲染器。`switch` 的 default 分支用 TypeScript 断言 `NullRenderingAttachmentType`，确保穷尽。

**导出：** `AttachmentMessage`

**参数：** `addMargin`, `attachment`, `verbose`, `isTranscriptMode`

**关键行为：** 通过 feature-gated `EXPERIMENTAL_SKILL_SEARCH` 检测 demo 环境；对 `teammate_mailbox` 附件类型有特殊处理；plan 相关附件会走 `tryRenderPlanApprovalMessage`。

---

### messages/CollapsedReadSearchContent.tsx

**用途：** 渲染折叠后的“Read N files, searched M patterns”摘要行。

**导出：** `CollapsedReadSearchContent`

**参数：** `message`, `inProgressToolUseIDs`, `shouldAnimate`, `verbose`, `tools`, `lookups`, `isActiveGroup`

**常量：** `MIN_HINT_DISPLAY_MS = 700`

---

### messages/CompactBoundaryMessage.tsx

**用途：** 渲染“✻ Conversation compacted (ctrl+o for history)”边界标记。

**导出：** `CompactBoundaryMessage`

**参数：** 无

---

### messages/GroupedToolUseContent.tsx

**用途：** 通过 `tool.renderGroupedToolUse` 渲染分组后的工具调用。

**导出：** `GroupedToolUseContent`

**参数：** `message`, `tools`, `lookups`, `inProgressToolUseIDs`, `shouldAnimate`

---

### messages/HighlightedThinkingText.tsx

**用途：** 渲染 thinking/prompt 文本，可选 KAIROS brief 布局模式。brief 布局里支持 “You {timestamp}” 头部；会给 “ultrathink” 触发序列上彩虹色。

**导出：** `HighlightedThinkingText`

**参数：** `text`, `useBriefLayout`, `timestamp`

**关键行为：** 使用 `findThinkingTriggerPositions` 和 `getRainbowColor` 做 ultrathink 高亮；通过 `QueuedMessageContext` 控制 queued 状态样式。

---

### messages/HookProgressMessage.tsx

**用途：** 渲染 PreToolUse/PostToolUse hook 的执行进度。

**导出：** `HookProgressMessage`

**参数：** `hookEvent`, `lookups`, `toolUseID`, `verbose`, `isTranscriptMode`

---

### messages/nullRenderingAttachments.ts

**用途：** 定义哪些附件类型会渲染为 `null`（没有可见输出），并且应该在 200 条消息渲染预算之前过滤掉。

**导出：**

| 导出 | 类型 | 说明 |
|---|---|---|
| `NullRenderingAttachmentType` | type | 29 个不渲染附件类型字符串的联合类型 |
| `isNullRenderingAttachment(msg)` | `(msg: Message \| NormalizedMessage) => boolean` | 如果消息是不会渲染的附件，返回 true |

**不渲染类型（共 29 个）：** `hook_success`, `hook_additional_context`, `hook_cancelled`, `command_permissions`, `agent_mention`, `budget_usd`, `critical_system_reminder`, `edited_image_file`, `edited_text_file`, `opened_file_in_ide`, `output_style`, `plan_mode`, `plan_mode_exit`, `plan_mode_reentry`, `structured_output`, `team_context`, `todo_reminder`, `context_efficiency`, `deferred_tools_delta`, `mcp_instructions_delta`, `companion_intro`, `token_usage`, `ultrathink_effort`, `max_turns_reached`, `task_reminder`, `auto_mode`, `auto_mode_exit`, `output_token_usage`, `pen_mode_enter`, `pen_mode_exit`, `verify_plan_reminder`, `current_session_memory`, `compaction_reminder`, `date_change`

**注：** TypeScript 通过 `AttachmentMessage` 的 `switch` default 分支断言 `attachment.type satisfies NullRenderingAttachmentType` 来保持同步。跟踪号 CC-724。

---

### messages/PlanApprovalMessage.tsx

**用途：** 渲染计划审批请求/响应，并处理 plan 相关附件。

**导出：** `PlanApprovalRequestDisplay`, `tryRenderPlanApprovalMessage`，以及其他计划审批相关组件。

---

### messages/RateLimitMessage.tsx

**用途：** 渲染限流错误消息，并可选显示升级提示。

**导出：** `getUpsellMessage(params: UpsellParams): string | null`, `RateLimitMessage`

**关键行为：** `getUpsellMessage` 会根据订阅类型和限流原因返回升级提示文案。

---

### messages/ShutdownMessage.tsx

**用途：** 渲染 swarm agent 关闭相关消息。

**导出：** `ShutdownRequestDisplay`, `ShutdownRejectedDisplay`

**`ShutdownRequestDisplay` 参数：** `request`

**关键行为：** 显示黄色警告边框，包含 `from` 和 `reason`。`ShutdownRejectedDisplay` 则显示较柔和的灰色边框。

---

### messages/SystemAPIErrorMessage.tsx

**用途：** 显示 API 错误消息，并在重试前带倒计时。前 3 次重试会隐藏。

**导出：** `SystemAPIErrorMessage`

**参数：** `message`, `verbose`

**关键状态：** `countdownMs: number`

**常量：** `MAX_API_ERROR_CHARS = 1000`

**关键行为：** `hidden = retryAttempt < 4`；使用 `useInterval` 让倒计时朝 `retryInMs` 走；显示 `retryAttempt / maxRetries` 进度。

---

### messages/SystemTextMessage.tsx

**用途：** 渲染所有 system 消息子类型：轮次耗时、stop hook 摘要、bridge 状态、thinking、memory saved 和通用文本。

**导出：** `SystemTextMessage`

**参数：** `message`, `addMargin`, `verbose`, `isTranscriptMode`

**关键行为：** 根据 `message.subtype` 路由到 `TurnDurationMessage`、`StopHookSummaryMessage` 等；`TEAMMEM` 受 feature gate 控制；使用 `TURN_COMPLETION_VERBS` 随机挑选 completion 动词。

---

### messages/TaskAssignmentMessage.tsx

**用途：** 渲染 coordinator 分配给子智能体的任务。

**导出：** `TaskAssignmentDisplay`

**参数：** `assignment`

**关键行为：** 渲染青色边框框，包含任务 ID、assigned-by、subject 和可选 description。

---

### messages/UserAgentNotificationMessage.tsx

**用途：** 渲染从 XML `<task-notification>` 标签提取出来的 agent 完成通知。显示彩色圆点、摘要和可选细节行。

**导出：** `UserAgentNotificationMessage`

**参数：** `addMargin`, `param`

**关键行为：** 提取 `<summary>` 和 `<status>`；`getStatusColor(status)` 映射：`completed` → `success`，`failed` → `error`，`killed` → `warning`，默认 → `text`。用 `BLACK_CIRCLE` 加状态色显示摘要；如果没有 summary 就返回 null。

---

### messages/UserBashInputMessage.tsx

**用途：** 渲染 bash 命令输入（从 `<bash-input>` XML 标签提取）以及 bash 边框样式。

**导出：** `UserBashInputMessage`

**参数：** `addMargin`, `param`

**关键行为：** 提取 `<bash-input>`；没有输入时返回 null；以 `! {command}` 并使用 `bashMessageBackgroundColor` 渲染。

---

### messages/UserBashOutputMessage.tsx

**用途：** 通过委托给 `BashToolResultMessage` 来渲染 bash 工具输出（stdout + stderr）。

**导出：** `UserBashOutputMessage`

**参数：** `content`, `verbose`

**关键行为：** 提取 `<bash-stdout>` 和 `<bash-stderr>` 标签（stdout 内部的 `<persisted-output>` 也会处理）；然后交给 `BashToolResultMessage`。

---

### messages/UserChannelMessage.tsx

**用途：** 渲染通过 bridge/channel 连接收到的消息（例如来自 Slack plugin）。解析 `<channel source="..." user="..." chat_id="...">content</channel>` XML 格式。

**导出：** `UserChannelMessage`

**参数：** `addMargin`, `param`

**常量：** `TRUNCATE_AT = 60`

**关键行为：** 用正则解析 `source`、可选 `user` 属性和正文。插件提供的 server 名称会去掉最后一个 `:` 之前的前缀；显示 `CHANNEL_ARROW` 前缀、server 名、可选用户归属和截断后的正文。如果正则不匹配就返回 null。

---

### messages/UserCommandMessage.tsx

**用途：** 渲染用户斜杠命令（例如 `/help`、`/compact`），从 `<command-message>` XML 标签提取。支持普通命令展示和 `skill-format` 展示。

**导出：** `UserCommandMessage`

**参数：** `addMargin`, `param`

**关键行为：** 提取 `<command-message>` 和 `<command-args>`；如果存在 `<skill-format>true</skill-format>`，就渲染成 `Skill(commandName)` 并带细微指针图标。否则显示命令名和 `figures.pointer`。没有命令消息就返回 null。

---

### messages/UserImageMessage.tsx

**用途：** 渲染用户消息里的图片附件。如果图片已存储且终端支持超链接，就显示可点击的 OSC 8 链接。

**导出：** `UserImageMessage`

**参数：** `imageId`, `addMargin`

**关键行为：** 标签是 `[Image #N]` 或 `[Image]`。如果提供了 `imageId`，且 `getStoredImagePath(imageId)` 返回路径并且 `supportsHyperlinks()` 为 true，就用 Ink `Link` 包裹成 `file://` URL。`addMargin` 为 true 时加 `Box marginTop={1}`；否则使用 `MessageResponse` 样式让它和上面的消息连在一起。

---

### messages/UserLocalCommandOutputMessage.tsx

**用途：** 渲染来自 `<local-command-stdout>` 和 `<local-command-stderr>` 标签的本地命令输出，用于展示本地执行的 hooks/commands 输出。

**导出：** `UserLocalCommandOutputMessage`

**参数：** `content`

**关键行为：** 如果两种标签都不存在，就在 `MessageResponse` 中以 dim 显示 `NO_CONTENT_MESSAGE`。每个非空、去掉两端空白后的流都通过内部 `IndentedContent` 组件渲染。`IndentedContent` 会检查内容是否已经以 `DIAMOND_OPEN` 或 `DIAMOND_FILLED` 前缀开头，避免重复加前缀；否则根据内容类型渲染为缩进的 Markdown 或纯文本。

---

### messages/UserMemoryInputMessage.tsx

**用途：** 渲染用户触发的 memory 保存通知。以特殊的 `#` 前缀和随机挑选的确认短语显示 memory 内容。

**导出：** `UserMemoryInputMessage`

**参数：** `addMargin`, `text`

**关键行为：** 提取 `<user-memory-input>` 标签内容；找不到就返回 null。内容以 `#` 前缀显示，颜色为 `remember`，背景为 `memoryBackgroundColor`。还会在 `MessageResponse` 中用 `height={1}` 显示随机确认语句（`'Got it.'`, `'Good to know.'`, `'Noted.'`），来源是 `lodash-es/sample`。

---

### messages/UserPlanMessage.tsx

**用途：** 渲染带结构化格式的计划内容块。

**导出：** `UserPlanMessage`

**参数：** `addMargin`, `planContent`

---

### messages/UserPromptMessage.tsx

**用途：** 渲染用户文本提示，并对过长输入做截断。KAIROS 里还有 feature-gated 的 brief layout。

**导出：** `UserPromptMessage`

**参数：** `addMargin`, `param`, `isTranscriptMode`, `timestamp`

**常量：** `MAX_DISPLAY_CHARS = 10000`, `TRUNCATE_HEAD_CHARS = 2500`, `TRUNCATE_TAIL_CHARS = 2500`

**关键行为：** `KAIROS` / `KAIROS_BRIEF` 控制 `isBriefOnly` 布局；会把长文本截成 `HEAD...TAIL`，并显示“+N chars omitted”提示。

---

### messages/UserResourceUpdateMessage.tsx

**用途：** 渲染 MCP resource 和 polling 更新通知，显示变更了什么，以及为什么变更。

**导出：** `UserResourceUpdateMessage`

**参数：** `addMargin`, `param`

**内部类型：**

```ts
type ParsedUpdate = {
  kind: 'resource' | 'polling';
  server: string;
  target: string;  // 资源更新时是 URI，轮询时是工具名
  reason?: string;
}
```

**关键行为：** `parseUpdates(text)` 使用两个正则：`<mcp-resource-update server="..." uri="...">` 和 `<mcp-polling-update type="..." server="..." tool="...">`。`formatUri(uri)` 会去掉 `file://` 前缀并只显示文件名；其他 URI 如果超过 40 个字符，会截到 39 个字符再加省略号。每条更新会显示 `REFRESH_ARROW` 图标、server 名、target 和可选 reason。如果没有任何更新就返回 null。

---

### messages/UserTeammateMessage.tsx

**用途：** 渲染通过 mailbox XML 协议传来的 teammate（子智能体/协调器）消息。会把计划、shutdown 和任务分配等内容分派到专门的渲染器。

**导出：** `TeammateMessageContent`, `UserTeammateMessage`

**参数：** `addMargin`, `param`, `isTranscriptMode`

**内部类型：**

```ts
type ParsedMessage = {
  teammateId: string;
  content: string;
  color?: string;
  summary?: string;
}
```

**关键行为：** `TEAMMATE_MSG_REGEX` 匹配 `<teammate-message teammate_id="..." color="..." summary="...">content</teammate-message>`。会先过滤掉 `isShutdownApproved` 生命周期消息和 `teammate_terminated` JSON payload，避免空行残留。对每条剩余消息，按顺序尝试 `tryRenderPlanApprovalMessage`、`tryRenderShutdownMessage`、`tryRenderTaskAssignmentMessage`；都不匹配时回退到通用渲染，使用 teammate ID 作为彩色标签（`toInkColor(color)`）。特殊的 `'leader'` teammateId 会显示成 `'leader'`。

---

### messages/UserTextMessage.tsx

**用途：** 用户文本消息的总路由器。提取 XML 标签，并分派到具体子渲染器。

**导出：** `UserTextMessage`

**参数：** `addMargin`, `param`, `verbose`, `planContent`, `isTranscriptMode`, `timestamp`

**关键行为：** 分派到 `UserPlanMessage`、`UserBashInputMessage`、`UserBashOutputMessage`、`UserCommandMessage`、`UserLocalCommandOutputMessage`、`UserMemoryInputMessage`、`UserTeammateMessage`、`UserAgentNotificationMessage`、`InterruptedByUser`、`UserResourceUpdateMessage`、`UserPromptMessage` 等。

---

## messages/UserToolResultMessage/ 子目录

---

### UserToolResultMessage/BashToolResultMessage.tsx

**用途：** 渲染 bash 工具结果的通用消息。会把 stdout、stderr 和进度消息组合成一个统一的输出块。

**导出：** `BashToolResultMessage`

**参数：** `result`, `verbose`, `shouldShowDot`, `shouldAnimate`

**关键行为：** 把工具的 `renderToolResultMessage` 包进 `SentryErrorBoundary`，并渲染过滤后的进度消息中的 `HookProgressMessage` 项。

---

### UserToolResultMessage/RejectedPlanMessage.tsx

**用途：** 渲染用户拒绝的 plan 审批。把 plan 内容显示在一个样式化框里。

**导出：** `RejectedPlanMessage`

**参数：** `plan`

**关键行为：** 在 subtle 颜色的标签 `User rejected Claude's plan:` 下方，渲染一个圆角边框、`planMode` 颜色的框，里面通过 `Markdown` 显示 plan。为了兼容 Windows Terminal，使用 `overflow="hidden"`。

---

### UserToolResultMessage/RejectedToolUseMessage.tsx

**用途：** 无 props 组件。渲染通用的 “Tool use rejected” 灰色提示。

**导出：** `RejectedToolUseMessage`

**参数：** 无

**关键行为：** 完全静态，只在 `MessageResponse` 里以 `height={1}` 渲染 dim 状态的 “Tool use rejected” 文本。由 React Compiler 作为模块级常量记忆化。

---

### UserToolResultMessage/utils.tsx

**用途：** 通过消息 lookups，提供一个共享的 React hook 来从 tool use ID 解析 `Tool` 定义。

**导出：** `useGetToolFromMessages(toolUseID: string, tools: Tools, lookups: ReturnType<typeof buildMessageLookups>): { tool: Tool; toolUse: ToolUseBlockParam } | null`

**关键行为：** 这是一个 memoized hook。先查 `toolUseByToolUseID.get(toolUseID)`，再通过 `findToolByName(tools, toolUse.name)` 解析。只要任一查找失败，就返回 null。
