# Claude Code — 工具系统

## 目录

1. [工具框架（Tool.ts）](#1-工具框架toolts)
2. [工具注册表（tools.ts）](#2-工具注册表toolsts)
3. [任务框架（Task.ts / tasks.ts）](#3-任务框架taskts--tasksts)
4. [核心文件工具](#4-核心文件工具)
   - [FileReadTool](#41-filereadtool)
   - [FileWriteTool](#42-filewritetool)
   - [FileEditTool](#43-fileedittool)
5. [Shell 执行工具](#5-shell-执行工具)
   - [BashTool](#51-bashtool)
   - [PowerShellTool](#52-powershelltool)
6. [搜索工具](#6-搜索工具)
   - [GlobTool](#61-globtool)
   - [GrepTool](#62-greptool)
7. [Agent / 多智能体工具](#7-agent--多智能体工具)
   - [AgentTool](#71-agenttool)
   - [TeamCreateTool](#72-teamcreatetool)
   - [TeamDeleteTool](#73-teamdeletetool)
   - [SendMessageTool](#74-sendmessagetool)
8. [任务管理工具](#8-任务管理工具)
   - [TaskStopTool](#81-taskstoptool)
   - [TaskOutputTool](#82-taskoutputtool)
   - [TodoWriteTool（V1）](#83-todowritetool-v1)
   - [TaskCreateTool（V2）](#84-taskcreatetool-v2)
   - [TaskGetTool（V2）](#85-taskgettool-v2)
   - [TaskUpdateTool（V2）](#86-taskupdatetool-v2)
   - [TaskListTool（V2）](#87-tasklisttool-v2)
9. [Web 工具](#9-web-工具)
   - [WebFetchTool](#91-webfetchtool)
   - [WebSearchTool](#92-websearchtool)
10. [MCP 集成工具](#10-mcp-集成工具)
    - [MCPTool](#101-mcptool)
    - [McpAuthTool](#102-mcpauthtool)
    - [ListMcpResourcesTool](#103-listmcpresourcestool)
    - [ReadMcpResourceTool](#104-readmcpresourcetool)
11. [Plan Mode 工具](#11-plan-mode-工具)
    - [EnterPlanModeTool](#111-enterplanmodetool)
    - [ExitPlanModeV2Tool](#112-exitplanmodev2tool)
12. [Notebook 工具](#12-notebook-工具)
13. [Worktree 工具](#13-worktree-工具)
    - [EnterWorktreeTool](#131-enterworkreetool)
    - [ExitWorktreeTool](#132-exitworkreetool)
14. [调度工具](#14-调度工具)
    - [CronCreateTool](#141-croncreatetool)
    - [CronDeleteTool](#142-crondeletetool)
    - [CronListTool](#143-cronlisttool)
15. [元工具 / 发现工具](#15-元工具--发现工具)
    - [ToolSearchTool](#151-toolsearchtool)
    - [AskUserQuestionTool](#152-askuserquestiontool)
16. [Kairos / 特殊模式工具](#16-kairos--特殊模式工具)
    - [BriefTool（SendUserMessage）](#161-brieftool-sendusermessage)
    - [SleepTool](#162-sleeptool)
    - [RemoteTriggerTool](#163-remotetriggertool)
17. [SDK / 输出工具](#17-sdk--输出工具)
    - [SyntheticOutputTool（StructuredOutput）](#171-syntheticoutputtool-structuredoutput)
18. [Skill Tool](#18-skill-tool)
19. [LSP 工具](#19-lsp-工具)
20. [REPL 工具](#20-repl-工具)
21. [Config 工具](#21-config-工具)
22. [共享工具函数](#22-共享工具函数)
    - [tools/utils.ts](#221-toolsutilsts)
    - [tools/shared/gitOperationTracking.ts](#222-toolssharedgitoperationtrackingts)
    - [tools/shared/spawnMultiAgent.ts](#223-toolssharedspawnmultiagentts)
23. [测试工具](#23-测试工具)

---

## 1. 工具框架

**源文件：** `src/Tool.ts`

### 1.1 核心接口

```typescript
export type Tool<Input extends ZodType = ZodType, Output = unknown, Progress = unknown> = {
  // 标识
  name: string
  isMcp?: boolean
  mcpInfo?: { serverName: string; toolName: string }
  isLsp?: boolean
  alwaysLoad?: boolean
  shouldDefer?: boolean

  // 模式（getter 属性，用于延迟初始化）
  readonly inputSchema: Input
  readonly outputSchema?: ZodType<Output>

  // 元数据
  description(): Promise<string>
  prompt(): Promise<string>
  userFacingName(input?: z.infer<Input>): string
  maxResultSizeChars?: number
  searchHint?: string

  // 能力标志（可按次调用传入输入）
  isEnabled(permissionContext?: ToolPermissionContext): boolean
  isConcurrencySafe(input?: z.infer<Input>): boolean
  isReadOnly(input?: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean
  toAutoClassifierInput(input: z.infer<Input>): string
  isSearchOrReadCommand?: (input: z.infer<Input>) => { isSearch: boolean; isRead: boolean }

  // 执行
  validateInput?(input: z.infer<Input>): Promise<ValidationResult>
  checkPermissions(input: z.infer<Input>, context: ToolUseContext): Promise<PermissionDecision>
  call(input: z.infer<Input>, context: ToolUseContext): Promise<ToolResult<Output>>

  // UI 渲染（React / Ink）
  renderToolUseMessage(input: z.infer<Input>, options: RenderOptions): ReactNode | null
  renderToolUseProgressMessage?(progress: Progress, input?: z.infer<Input>): ReactNode | null
  renderToolUseQueuedMessage?(input: z.infer<Input>): ReactNode | null
  renderToolUseRejectedMessage?(input: z.infer<Input>, ...): ReactNode | null
  renderToolResultMessage(output: Output, ...): ReactNode | null
  renderToolUseErrorMessage?(error: Error, ...): ReactNode | null

  // 输出映射
  mapToolResultToToolResultBlockParam(
    output: Output,
    toolUseID: string,
    context: ToolUseContext,
  ): ToolResultBlockParam

  // 路径跟踪（用于权限规则）
  getPath?(input?: z.infer<Input>): string | undefined
}
```

### 1.2 ToolDef - 定义形状

`ToolDef<Input, Output, Progress>` 是传给 `buildTool()` 的形状。它和 `Tool` 的字段基本一致，只是去掉了 `buildTool()` 会自动补齐的默认值。

### 1.3 buildTool()

```typescript
function buildTool<Input, Output, Progress>(def: ToolDef<Input, Output, Progress>): Tool<...>
```

它会补上安全默认值：
- `isEnabled` → `() => true`
- `isConcurrencySafe` → `() => false`
- `isReadOnly` → `() => false`
- `checkPermissions` → `async () => ({ behavior: 'allow', updatedInput: input })`
- `toAutoClassifierInput` → `() => ''`
- `userFacingName` → `() => def.name`

### 1.4 ToolUseContext

传给每个 `call()` 和 `checkPermissions()` 的上下文对象：

```typescript
type ToolUseContext = {
  // 配置
  options: {
    commands: Command[]
    tools: Tool[]
    mcpClients: MCPClient[]
    mainLoopModel: string
    thinkingConfig: ThinkingConfig
    // ... 其他选项
  }

  // 中断信号
  abortController: AbortController

  // 应用状态访问器
  getAppState(): AppState
  setAppState(fn: (prev: AppState) => AppState): void

  // 文件读取跟踪（用于 read-before-write 强制检查）
  readFileState: Map<string, { mtime: number; content: string }>

  // 权限上下文
  permissionContext: ToolPermissionContext

  // UI 注入
  setToolJSX: SetToolJSXFn

  // 回调
  onPermissionRequest(request: PermissionRequest): Promise<PermissionDecision>
  onToolCallStart(toolName: string, input: unknown): void
  onToolCallEnd(toolName: string, result: unknown): void

  // Agent / teammate 上下文
  agentId?: AgentId
  isSubagent?: boolean
  isCoordinator?: boolean

  // 特定工具分类的附加字段
  globLimits?: { maxResults: number }
}
```

### 1.5 ToolPermissionContext

```typescript
type ToolPermissionContext = {
  mode: PermissionMode  // 'default' | 'plan' | 'auto' | 'bypassPermissions' | 'acceptEdits'
  alwaysAllow: PermissionRule[]
  alwaysDeny: PermissionRule[]
  alwaysAsk: PermissionRule[]
  additionalWorkingDirectories: string[]
  toolPermissions: Record<string, ToolPermissionOverride>
}
```

### 1.6 PermissionResult / PermissionDecision

```typescript
type PermissionDecision =
  | { behavior: 'allow'; updatedInput: Input }
  | { behavior: 'ask'; message: string; decisionReason?: string }
  | { behavior: 'deny'; message: string }
  | { behavior: 'passthrough' }  // 始终询问用户

type ValidationResult =
  | { result: true }
  | { result: false; message: string; errorCode?: number }
```

### 1.7 ToolResult

```typescript
type ToolResult<T> = { data: T }
```

### 1.8 Tools 类型别名与辅助函数

```typescript
type Tools = Tool[]

function findToolByName(tools: Tools, name: string): Tool | undefined
function toolMatchesName(tool: Tool, name: string): boolean
```

---

## 2. 工具注册表

**源文件：** `src/tools.ts`

### 2.1 getAllBaseTools()

返回内置工具的完整有序列表。顺序必须和 Statsig 缓存配置保持一致。

```typescript
function getAllBaseTools(): Tool[]
```

包含内容（按条件）：
- 始终包含：BashTool、GlobTool、GrepTool、FileReadTool、FileEditTool、FileWriteTool、AgentTool、WebFetchTool、WebSearchTool、NotebookEditTool、TodoWriteTool、TaskStopTool、AskUserQuestionTool、SkillTool、MCPTool、EnterPlanModeTool、ExitPlanModeV2Tool、ToolSearchTool、TaskCreateTool、TaskGetTool、TaskUpdateTool、TaskListTool、TeamCreateTool、TeamDeleteTool、SendMessageTool、TaskOutputTool、SyntheticOutputTool、EnterWorktreeTool、ExitWorktreeTool、BriefTool、RemoteTriggerTool
- 仅 Ant：ConfigTool、TungstenTool、REPLTool
- 特性门控（`feature('KAIROS')` + `isKairosCronEnabled()`）：CronCreateTool、CronDeleteTool、CronListTool
- 特性门控（`feature('AGENT_TRIGGERS')`）：RemoteTriggerTool
- 特性门控（`feature('SLEEP_TOOL')`）：SleepTool
- 特性门控（`feature('MONITOR_TOOL')`）：MonitorMcpTask
- 启用 LSP 时：LSPTool

### 2.2 getTools(permissionContext)

```typescript
function getTools(permissionContext: ToolPermissionContext): Tool[]
```

- 如果设置了 `CLAUDE_CODE_SIMPLE` 环境变量：只返回 `[BashTool, FileReadTool, FileEditTool]`
- 调用 `getAllBaseTools()`，再用 `filterToolsByDenyRules()` 过滤
- 在 REPL 模式下：隐藏 `REPL_ONLY_TOOLS`（Bash、Read、Write、Edit、Glob、Grep、NotebookEdit、Agent）

### 2.3 assembleToolPool()

```typescript
function assembleToolPool(
  baseTools: Tool[],
  mcpTools: Tool[],
): Tool[]
```

- 合并内置工具和 MCP 工具
- 按名称排序，保证 prompt cache 稳定
- 去重（同名时以内置工具优先）

### 2.4 filterToolsByDenyRules()

```typescript
function filterToolsByDenyRules(
  tools: Tool[],
  permissionContext: ToolPermissionContext,
): Tool[]
```

移除名称命中 permissionContext 中全局拒绝规则的工具。

### 2.5 Tool 预设

```typescript
const TOOL_PRESETS = {
  'full': /* all tools */,
  'minimal': /* BashTool, FileReadTool, FileEditTool */,
  // ...
}

function parseToolPreset(preset: string): Tool[] | null
function getToolsForDefaultPreset(): Tool[]
function getMergedTools(base: Tool[], overrides: Tool[]): Tool[]
```

---

## 3. 任务框架

**源文件：** `src/Task.ts`、`src/tasks.ts`

### 3.1 TaskType

```typescript
type TaskType =
  | 'local_bash'       // prefix: 'b'
  | 'local_agent'      // prefix: 'a'
  | 'remote_agent'     // prefix: 'r'
  | 'in_process_teammate' // prefix: 't'
  | 'local_workflow'   // prefix: 'w'
  | 'monitor_mcp'      // prefix: 'm'
  | 'dream'            // prefix: 'd'
```

### 3.2 TaskStatus

```typescript
type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'

function isTerminalTaskStatus(status: TaskStatus): boolean
// 返回 true 表示 'completed'、'failed'、'killed'
```

### 3.3 TaskStateBase

```typescript
type TaskStateBase = {
  id: string          // prefix + 8 个随机 base-36 字符
  type: TaskType
  status: TaskStatus
  description: string
  toolUseId?: string
  startTime: number   // Date.now()
  endTime?: number
  totalPausedMs?: number
  outputFile: string  // getTaskOutputPath(id)
  outputOffset: number
  notified: boolean
}
```

### 3.4 ID 生成

```typescript
// 字母表：'0123456789abcdefghijklmnopqrstuvwxyz'
// 36^8 ≈ 2.8 万亿种组合
function generateTaskId(type: TaskType): string
// 返回：prefix + 8 个 crypto-random base-36 字符

function createTaskStateBase(
  id: string,
  type: TaskType,
  description: string,
  toolUseId?: string,
): TaskStateBase
```

### 3.5 Task 接口

```typescript
type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

### 3.6 Task 注册表

```typescript
// src/tasks.ts
function getAllTasks(): Task[]
// 返回：[LocalShellTask、LocalAgentTask、RemoteAgentTask、DreamTask，
//        以及可选的 LocalWorkflowTask、MonitorMcpTask]

function getTaskByType(type: TaskType): Task | undefined
```

---

## 4. 核心文件工具

### 4.1 FileReadTool

**工具名：** `Read`
**源文件：** `src/tools/FileReadTool/FileReadTool.ts`

**特性：**
- `isConcurrencySafe: true`
- `isReadOnly: true`
- `maxResultSizeChars: Infinity`（防止通过磁盘持久化形成循环读取）
- `strict: true`
- `searchHint: 'read files, images, PDFs, notebooks'`
- `isSearchOrReadCommand: { isSearch: false, isRead: true }`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `file_path` | `string` | 是 | 文件绝对路径 |
| `offset` | `integer` | 否 | 从哪一行开始读 |
| `limit` | `integer` | 否 | 读取多少行 |
| `pages` | `string` | 否 | PDF 页码范围，如 `"1-5"`、`"3"`、`"10-20"`；仅 PDF；每次最多 20 页 |

**输出模式（按 `type` 区分联合类型）：**

```typescript
// 文本文件
{ type: 'text'; content: string; numLines: number; startLine: number; totalLines: number }

// 图片文件
{ type: 'image'; base64: string; mimeType: string; originalSize: number; dimensions: { width: number; height: number } }

// Jupyter notebook
{ type: 'notebook'; cells: NotebookCell[] }

// PDF（完整）
{ type: 'pdf'; base64: string; originalSize: number }

// PDF（提取后的页面）
{ type: 'parts'; pages: PDFPage[] }

// 未变化（自上次读取后内容未变）
{ type: 'file_unchanged' }
```

**安全 / 校验：**
- 阻止设备文件：`/dev/zero`、`/dev/random`、`/dev/urandom`、`/dev/full`、`/dev/stdin`、`/dev/tty`，以及其他无限流设备
- 把文件登记进 `readFileState` 缓存（路径 → `{mtime, content}`），供 FileEditTool/FileWriteTool 做 read-before-write 强制检查
- Windows 下跳过 UNC 路径处理

**导出：**
```typescript
function registerFileReadListener(listener: FileReadListener): void
class MaxFileReadTokenExceededError extends Error
```

---

### 4.2 FileWriteTool

**工具名：** `Write`
**源文件：** `src/tools/FileWriteTool/FileWriteTool.ts`

**特性：**
- `strict: true`
- `maxResultSizeChars: 100_000`
- `searchHint: 'create or overwrite files'`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `file_path` | `string` | 是 | 文件绝对路径 |
| `content` | `string` | 是 | 要写入的完整文件内容 |

**输出模式：**

```typescript
{
  type: 'create' | 'update'
  filePath: string
  content: string
  structuredPatch: StructuredPatch
  originalFile: string | null   // 新文件时为 null
  gitDiff?: string              // 可选的 git diff
}
```

**校验 / 安全：**
- **read-before-write 强制：** 对于已存在文件，必须先进入 `readFileState` 缓存，也就是必须先用 FileReadTool 读过
- **mtime 过期检查：** 如果文件自上次读取后已经变更，则拒绝覆盖，避免踩掉并发修改
- **文件大小限制：** 最大 1 GiB
- **UNC 路径安全：** Windows 上对 UNC 路径（`\\server\share`）跳过读检查
- **`.ipynb` 文件：** 转交给 `NotebookEditTool`
- **团队内存保护：** 阻止写入 team memory secret 文件
- **拒绝规则：** 检查 permission context 中的 deny 规则
- **LF 行尾：** 新内容统一写成 LF，不随平台变化
- 成功编辑后通知 LSP 客户端

---

### 4.3 FileEditTool

**工具名：** `Edit`
**源文件：** `src/tools/FileEditTool/FileEditTool.ts`

**特性：**
- `strict: true`
- `maxResultSizeChars: 100_000`
- `searchHint: 'modify file contents in place'`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `file_path` | `string` | 是 | 要编辑的文件绝对路径 |
| `old_string` | `string` | 是 | 要查找的文本（除非 `replace_all` 为 true，否则必须恰好出现一次） |
| `new_string` | `string` | 是 | 替换文本（必须和 `old_string` 不同） |
| `replace_all` | `boolean` | 否（默认 `false`） | 是否替换所有 `old_string` 出现位置 |

**输出模式：**

```typescript
{
  filePath: string
  oldString: string
  newString: string
  originalFile: string      // 编辑前的文件内容
  structuredPatch: StructuredPatch
  userModified: boolean     // 用户是否改过提议的 diff
  replaceAll: boolean
  gitDiff?: string
}
```

**校验 / 安全：**
- `old_string` 必须和 `new_string` 不同
- 文件必须先通过 `readFileState` 读取过（read-before-write 强制）
- mtime 过期检查（同 FileWriteTool）
- `old_string` 必须能在文件内容里找到
- 除非 `replace_all` 为 true，否则 `old_string` 最多只能出现 1 次
- 文件大小上限：1 GiB
- UNC 路径安全跳过
- `.ipynb` 文件会转交给 `NotebookEditTool`
- 团队内存 secret 保护
- 检查 permission context 的拒绝规则
- 使用 `findActualString()` 做引号归一化（同时处理直引号/弯引号）
- 使用 `preserveQuoteStyle()` 保持原始引号风格
- 成功编辑后通知 LSP 客户端

---

## 5. Shell 执行工具

### 5.1 BashTool

**工具名：** `Bash`
**源文件：** `src/tools/BashTool/BashTool.tsx`

**特性：**
- `maxResultSizeChars`（随输出类型变化）
- `searchHint`（来自 prompt）
- 支持后台任务执行
- 支持沙盒（Linux 用 bwrap，macOS 用 sandbox-exec）

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `command` | `string` | 是 | 要执行的 shell 命令 |
| `timeout` | `number` | 否 | 超时时间，单位毫秒（最大值随上下文变化） |
| `description` | `string` | 否 | 命令用途的可读描述 |
| `run_in_background` | `boolean` | 否 | 作为后台任务启动；当 `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=true` 时会从 schema 中移除 |
| `dangerouslyDisableSandbox` | `boolean` | 否 | 覆盖沙盒模式 |
| `_simulatedSedEdit` | internal | — | 永远不出现在模型可见 schema 中；用于模拟 sed 编辑 |

**输出模式：**

```typescript
{
  stdout: string
  stderr: string
  interrupted: boolean
  isImage?: boolean               // stdout 包含 base64 图片数据
  backgroundTaskId?: string       // run_in_background=true 时会设置
  backgroundedByUser?: boolean    // 用户按了 Ctrl+B
  assistantAutoBackgrounded?: boolean  // 达到阻塞预算后自动转后台
  dangerouslyDisableSandbox?: boolean
  returnCodeInterpretation?: string    // 退出码的语义含义
  noOutputExpected?: boolean
  structuredContent?: unknown
  persistedOutputPath?: string     // 输出太大时持久化到这里
  persistedOutputSize?: number     // 持久化后的总字节数
}
```

**时间常量：**
- 进度阈值：`2_000ms`（2 秒后显示进度转圈）
- 自动转后台阈值：`120_000ms`（2 分钟后自动转后台）
- `ASSISTANT_BLOCKING_BUDGET_MS`：`15_000ms`（assistant/Kairos 模式下，主 agent 15 秒后自动转后台）

**权限行为：**
- 按命令字符串检查权限规则
- `bypassPermissions` 模式：允许所有命令
- `auto` / `acceptEdits` 模式：允许只读 bash 命令，不询问；写命令会询问
- `default` 模式：默认都会询问，除非被显式允许

**沙盒：**
- Linux：基于 `bwrap`（bubblewrap）隔离
- macOS：基于 `sandbox-exec` 隔离
- Windows：不启用沙盒（bwrap/sandbox-exec 只适用于 POSIX）
- `dangerouslyDisableSandbox` 可按次覆盖
- 企业策略：如果要求沙盒但环境不支持，则阻止执行

**阻止模式：**
- `detectBlockedSleepPattern()`：如果命令第一条语句就是 `sleep N` 且 `N>=2`，会检测出来并建议改用 SleepTool

**Git 操作跟踪：**
- 每条 bash 命令结束后都会调用 `trackGitOperations()`，用于上报 commit、push、PR 创建等分析事件
- `isSearchOrReadBashCommand()`：用于把命令分类给 UI 折叠

**导出：**
```typescript
function isSearchOrReadBashCommand(command: string): { isSearch: boolean; isRead: boolean }
function detectBlockedSleepPattern(command: string): string | null
type BashToolInput = { command: string; timeout?: number; description?: string; run_in_background?: boolean; dangerouslyDisableSandbox?: boolean }
```

---

### 5.2 PowerShellTool

**工具名：** `PowerShell`
**源文件：** `src/tools/PowerShellTool/PowerShellTool.tsx`

Windows 原生的 PowerShell 执行工具，接口和 BashTool 对齐。

**特性：**
- 只在 Windows 上使用；需要已安装 PowerShell（`pwsh`）
- 通过 `getCachedPowerShellPath()` 找到 PowerShell 路径
- 通过共享的 `trackGitOperations()` 跟踪 git 操作
- 沙盒策略和 BashTool 相同（在 Linux/macOS 上运行 `pwsh` 时仍用 POSIX 沙盒）
- `PROGRESS_THRESHOLD_MS: 2_000ms`
- `ASSISTANT_BLOCKING_BUDGET_MS: 15_000ms`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `command` | `string` | 是 | 要执行的 PowerShell 命令 |
| `timeout` | `number` | 否 | 可选超时时间（最大 `getMaxTimeoutMs()`） |
| `description` | `string` | 否 | 命令用途描述 |
| `run_in_background` | `boolean` | 否 | 后台执行；当 `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=true` 时会移除 |
| `dangerouslyDisableSandbox` | `boolean` | 否 | 覆盖沙盒模式 |

**输出模式：**

```typescript
{
  stdout: string
  stderr: string
  interrupted: boolean
  returnCodeInterpretation?: string
  isImage?: boolean
  persistedOutputPath?: string
  persistedOutputSize?: number
  backgroundTaskId?: string
  backgroundedByUser?: boolean
  assistantAutoBackgrounded?: boolean
}
```

**PowerShell 专属特性：**
- `PS_SEARCH_COMMANDS`：`Select-String`、`Get-ChildItem`、`FindStr`、`where.exe`（相当于 grep/find）
- `PS_READ_COMMANDS`：`Get-Content`、`Get-Item`、`Test-Path`、`Resolve-Path`、`Get-Process`、`Get-Service`、`Get-ChildItem`、`Get-Location`、`Get-FileHash`、`Get-Acl`、`Format-Hex`
- `PS_SEMANTIC_NEUTRAL_COMMANDS`：`Write-Output`、`Write-Host`
- `detectBlockedSleepPattern()`：会拦截以 `Start-Sleep N`、`Start-Sleep -Seconds N`、`sleep N` 开头的第一条语句
- `DISALLOWED_AUTO_BACKGROUND_COMMANDS`：`['start-sleep', 'sleep']`（不会自动转后台）
- Windows 原生沙盒策略：如果企业要求沙盒但这是 Windows 原生环境，则阻止执行

**导出：**
```typescript
export type PowerShellToolInput
function detectBlockedSleepPattern(command: string): string | null
```

---

## 6. 搜索工具

### 6.1 GlobTool

**工具名：** `Glob`
**源文件：** `src/tools/GlobTool/GlobTool.ts`

**特性：**
- `isConcurrencySafe: true`
- `isReadOnly: true`
- `searchHint: 'find files by name pattern or wildcard'`
- `isSearchOrReadCommand: { isSearch: true, isRead: false }`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `pattern` | `string` | 是 | Glob 模式（如 `"**/*.js"`、`"src/**/*.ts"`） |
| `path` | `string` | 否 | 搜索目录（默认 cwd） |

**输出模式：**

```typescript
{
  filenames: string[]   // 相对于 cwd 的路径
  durationMs: number
  numFiles: number
  truncated: boolean    // 如果结果被截断则为 true
}
```

**行为：**
- 默认上限：100 个文件（可通过 `context.globLimits?.maxResults` 覆盖）
- 结果按修改时间排序（最新的排前面）
- 路径会转成相对 cwd 的形式，便于压缩

---

### 6.2 GrepTool

**工具名：** `Grep`
**源文件：** `src/tools/GrepTool/GrepTool.ts`

**特性：**
- `isConcurrencySafe: true`
- `isReadOnly: true`
- `strict: true`
- `maxResultSizeChars: 20_000`
- `searchHint: 'search file contents with regex (ripgrep)'`
- `isSearchOrReadCommand: { isSearch: true }`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `pattern` | `string` | 是 | 正则表达式模式 |
| `path` | `string` | 否 | 要搜索的文件或目录 |
| `glob` | `string` | 否 | Glob 过滤器（如 `"*.js"`、`"**/*.tsx"`） |
| `output_mode` | `'content' \| 'files_with_matches' \| 'count'` | 否（默认 `'files_with_matches'`） | 输出格式 |
| `-B` | `number` | 否 | 每个命中前的行数（需要 `output_mode: 'content'`） |
| `-A` | `number` | 否 | 每个命中后的行数（需要 `output_mode: 'content'`） |
| `-C` | `number` | 否 | 每个命中前后的行数 |
| `context` | `number` | 否 | `-C` 的别名 |
| `-n` | `boolean` | 否 | 显示行号（需要 `output_mode: 'content'`） |
| `-i` | `boolean` | 否 | 不区分大小写 |
| `type` | `string` | 否 | 文件类型过滤器（如 `"js"`、`"py"`） |
| `head_limit` | `number` | 否（默认 `250`） | 只输出前 N 行 / 条结果 |
| `offset` | `number` | 否（默认 `0`） | 跳过前 N 条结果 |
| `multiline` | `boolean` | 否（默认 `false`） | 启用多行匹配（`.` 也能匹配换行） |

**输出模式：**

```typescript
{
  mode: 'content' | 'files_with_matches' | 'count'
  numFiles: number
  filenames: string[]
  content?: string        // 当 mode='content' 时
  numLines?: number       // 当 mode='content' 时
  numMatches?: number     // 当 mode='count' 时
  appliedLimit?: number
  appliedOffset?: number
}
```

**实现细节：**
- 底层依赖 `ripgrep`（`rg`）二进制
- 排除 VCS 目录：`.git`、`.svn`、`.hg`、`.bzr`、`.jj`、`.sl`
- 使用 `max-columns: 500` 防止单行过长
- 默认 `head_limit: 250`

---

## 7. Agent / 多智能体工具

### 7.1 AgentTool

**工具名：** `Agent`（别名：`Task`）
**常量：** `AGENT_TOOL_NAME`、`LEGACY_AGENT_TOOL_NAME`
**源文件：** `src/tools/AgentTool/AgentTool.tsx`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `description` | `string` | 是 | 3-5 个词的任务描述 |
| `prompt` | `string` | 是 | 给 agent 的完整任务 prompt |
| `subagent_type` | `string` | 否 | 要启动的子智能体类型 |
| `model` | `'sonnet' \| 'opus' \| 'haiku'` | 否 | agent 的模型别名 |
| `run_in_background` | `boolean` | 否 | 作为后台任务启动 |
| `name` | `string` | 否 | 用于消息通信的命名 agent |
| `team_name` | `string` | 否 | 关联到某个 team |
| `mode` | `string` | 否 | 权限模式覆盖 |
| `isolation` | `'worktree' \| 'remote'` | 否 | 隔离策略 |
| `cwd` | `string` | 否 | 工作目录（仅 Kairos） |

**输出模式：**

同步完成：
```typescript
{
  status: 'completed'
  result: string
}
```

异步启动：
```typescript
{
  status: 'async_launched'
  agentId: string
  description: string
  prompt: string
}
```

**行为：**
- `120_000ms` 后自动转后台
- `2_000ms` 后显示进度
- 支持 fork 子智能体（`subagent_type: 'fork'`）
- 多智能体 swarm 集成：在 team 内部可以启动带名字的 teammate
- `isolation: 'worktree'` 会创建 git worktree 做隔离执行
- `isolation: 'remote'` 会在远程会话中运行

---

### 7.2 TeamCreateTool

**工具名：** `TeamCreate`
**源文件：** `src/tools/TeamCreateTool/TeamCreateTool.ts`
**门控：** `isAgentSwarmsEnabled()`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `team_name` | `string` | 是 | team 名称 |
| `description` | `string` | 否 | team 描述 |
| `agent_type` | `string` | 否 | team 成员的默认 agent 类型 |

**输出模式：**

```typescript
{
  team_name: string
  team_file_path: string
  lead_agent_id: string
}
```

**行为：**
- 在 `~/.claude/teams/<team_name>.json` 创建 team 文件
- 将任务列表切换为 team 作用域任务列表
- 把 team 注册到会话清理逻辑中（退出时自动清理）
- 强制“一位 leader 只能有一个 team”：如果已经有 team，再调用会返回错误

---

### 7.3 TeamDeleteTool

**工具名：** `TeamDelete`
**源文件：** `src/tools/TeamDeleteTool/TeamDeleteTool.ts`
**门控：** `isAgentSwarmsEnabled()`
**输入：** `{}`（空对象）

**输出模式：**

```typescript
{
  success: boolean
  message: string
  team_name?: string
}
```

**行为：**
- 如果任何非 leader 成员仍处于 `isActive !== false`，就拒绝删除
- 调用 `cleanupTeamDirectories(teamName)` 删除 team 文件和 worktree
- 从会话清理中注销 team
- 清空 teammate 的颜色分配
- 清空 leader 的 team 名称（任务列表回退到 session ID）
- 清空 app state 里的 team 上下文和 inbox

---

### 7.4 SendMessageTool

**工具名：** `SendMessage`
**源文件：** `src/tools/SendMessageTool/SendMessageTool.ts`
**门控：** `isAgentSwarmsEnabled()`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `to` | `string` | 是 | 收件人：agent 名、`'*'`（广播）、`'uds:<path>'` 或 `'bridge:<session-id>'` |
| `summary` | `string` | 否 | 给 UI 用的简短摘要 |
| `message` | `string \| StructuredMessage` | 是 | 消息内容 |

**StructuredMessage 联合类型：**

```typescript
type StructuredMessage =
  | { type: 'shutdown_request'; reason?: string }
  | { type: 'shutdown_response'; status: 'ok' | 'error'; message?: string }
  | { type: 'plan_approval_response'; approved: boolean; comment?: string; requestId: string }
```

**路由：**
- **进程内 agent：** 把消息放进 agent 的 inbox，或者恢复被暂停的 agent
- **Mailbox（teammate）：** 写入 `~/.claude/mailboxes/<name>.json`
- **UDS socket：** 通过 Unix domain socket 发送（用于本地进程间通信）
- **Bridge（跨机器）：** 通过 Remote Control API 路由；需要用户安全检查（不能自动批准）

**权限：** Bridge 消息需要通过 `decisionReason` 安全门才能继续。

---

## 8. 任务管理工具

### 8.1 TaskStopTool

**工具名：** `TaskStop`（别名：`KillShell`）
**源文件：** `src/tools/TaskStopTool/TaskStopTool.ts`
**特性：** `shouldDefer: true`、`isConcurrencySafe: true`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `task_id` | `string` | 否 | 要停止的任务 ID（来自后台任务启动） |
| `shell_id` | `string` | 否 | `task_id` 的旧别名 |

**输出模式：**

```typescript
{
  message: string
  task_id: string
  task_type: TaskType
  command?: string    // 适用于 bash 任务
}
```

**校验：**
- 任务必须存在于 app state 中
- 任务必须还在运行中（非终态）

---

### 8.2 TaskOutputTool

**工具名：** `TaskOutput`（`TASK_OUTPUT_TOOL_NAME`）
**源文件：** `src/tools/TaskOutputTool/TaskOutputTool.tsx`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `task_id` | `string` | 是 | 要读取输出的任务 ID |
| `block` | `boolean` | 否（默认 `true`） | 是否阻塞，直到任务完成 |
| `timeout` | `number` | 否（默认 `30_000`，范围 `0–600_000ms`） | 最长等待时间，毫秒 |

**输出模式：**

```typescript
{
  retrieval_status: 'success' | 'timeout' | 'not_ready'
  task: TaskOutput | null
}

type TaskOutput = {
  task_id: string
  task_type: TaskType
  status: TaskStatus
  description: string
  output: string
  exitCode?: number
  error?: string
  prompt?: string     // 适用于 agent 任务
  result?: string     // agent 任务最终结果文本
}
```

---

### 8.3 TodoWriteTool（V1）

**工具名：** `TodoWrite`
**源文件：** `src/tools/TodoWriteTool/TodoWriteTool.ts`
**特性：** `strict: true`、`shouldDefer: true`、`maxResultSizeChars: 100_000`
**门控：** 当 `isTodoV2Enabled()` 时禁用

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `todos` | `TodoItem[]` | 是 | todo 列表的完整替换内容 |

`TodoItem` 模式：
```typescript
{
  id: string
  content: string
  status: 'pending' | 'in_progress' | 'completed'
  priority: 'high' | 'medium' | 'low'
}
```

**输出模式：**

```typescript
{
  oldTodos: TodoItem[]
  newTodos: TodoItem[]
  verificationNudgeNeeded?: boolean
}
```

**行为：**
- 以原子方式替换整个 todo 列表
- 当所有 todo 都完成后，会自动清空列表
- `verificationNudgeNeeded`：提示 UI 让模型去核对已完成项

---

### 8.4 TaskCreateTool（V2）

**工具名：** `TaskCreate`
**源文件：** `src/tools/TaskCreateTool/TaskCreateTool.ts`
**门控：** `isTodoV2Enabled()`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `subject` | `string` | 是 | 简短任务标题 |
| `description` | `string` | 是 | 详细任务描述 |
| `activeForm` | `object` | 否 | 当前活跃任务的表单数据 |
| `metadata` | `object` | 否 | 任意元数据 |

**输出模式：**

```typescript
{
  task: {
    id: string
    subject: string
  }
}
```

**行为：**
- 创建后运行 `executeTaskCreatedHooks`
- 自动展开 UI 里的 task 面板
- 如果 hook 抛错，就删除任务

---

### 8.5 TaskGetTool（V2）

**工具名：** `TaskGet`
**源文件：** `src/tools/TaskGetTool/TaskGetTool.ts`
**门控：** `isTodoV2Enabled()`
**特性：** `shouldDefer: true`、`isReadOnly: true`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `taskId` | `string` | 是 | 要获取的任务 ID |

**输出模式：**

```typescript
{
  task: {
    id: string
    subject: string
    description: string
    status: TaskStatus
    blocks: string[]        // 这个任务阻塞的任务 ID
    blockedBy: string[]     // 阻塞这个任务的任务 ID
  } | null
}
```

---

### 8.6 TaskUpdateTool（V2）

**工具名：** `TaskUpdate`
**源文件：** `src/tools/TaskUpdateTool/TaskUpdateTool.ts`
**门控：** `isTodoV2Enabled()`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `taskId` | `string` | 是 | 要更新的任务 ID |
| `subject` | `string` | 否 | 更新后的标题 |
| `description` | `string` | 否 | 更新后的描述 |
| `activeForm` | `object` | 否 | 更新后的表单数据 |
| `status` | `TaskStatus \| 'deleted'` | 否 | 新状态；`'deleted'` 表示删除任务 |
| `addBlocks` | `string[]` | 否 | 这个任务应当阻塞的任务 ID |
| `addBlockedBy` | `string[]` | 否 | 阻塞这个任务的任务 ID |
| `owner` | `string` | 否 | 指定 owner |
| `metadata` | `object` | 否 | 更新后的元数据 |

**输出模式：**

```typescript
{
  success: boolean
  taskId: string
  updatedFields: string[]
  error?: string
  statusChange?: { from: TaskStatus; to: TaskStatus | 'deleted' }
  verificationNudgeNeeded?: boolean
}
```

**行为：**
- 当状态切换到 `completed` 时运行 `executeTaskCompletedHooks`
- 当状态变成 `in_progress` 时，会自动把 owner 设为调用这个工具的 agent
- owner 变化时会写 mailbox 通知
- `verificationNudgeNeeded`：如果连续完成了 3 个以上任务而没有核对步骤，就会设为 true

---

### 8.7 TaskListTool（V2）

**工具名：** `TaskList`
**源文件：** `src/tools/TaskListTool/TaskListTool.ts`
**门控：** `isTodoV2Enabled()`
**特性：** `shouldDefer: true`、`isReadOnly: true`
**输入：** `{}`（空对象）

**输出模式：**

```typescript
{
  tasks: Array<{
    id: string
    subject: string
    status: TaskStatus
    owner?: string
    blockedBy: string[]     // 只保留未完成的阻塞任务
  }>
}
```

**行为：**
- 过滤掉带 `_internal` metadata 标记的任务
- 从 `blockedBy` 里剔除已经完成的 ID

---

## 9. Web 工具

### 9.1 WebFetchTool

**工具名：** `WebFetch`
**源文件：** `src/tools/WebFetchTool/WebFetchTool.ts`
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `url` | `string` (URL) | 是 | 要抓取的 URL |
| `prompt` | `string` | 是 | 用于总结抓取内容的指令 |

**输出模式：**

```typescript
{
  bytes: number
  code: number          // HTTP 状态码
  codeText: string
  result: string        // 处理 / 总结后的内容
  durationMs: number
  url: string
}
```

**权限：** 按 hostname 规则控制。预先批准的 host 会直接绕过 prompt。规则格式：`domain:hostname`。

**实现：**
- 先把 HTML 转成 Markdown，再继续处理
- 内容超过阈值时，会用 `applyPromptToMarkdown(prompt, markdown)` 调 Haiku 做总结
- 尊重 `abortController` 信号

---

### 9.2 WebSearchTool

**工具名：** `WebSearch`
**源文件：** `src/tools/WebSearchTool/WebSearchTool.ts`
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `query` | `string`（至少 2 个字符） | 是 | 搜索查询 |
| `allowed_domains` | `string[]` | 否 | 只允许这些域名的结果 |
| `blocked_domains` | `string[]` | 否 | 排除这些域名的结果 |

**输出模式：**

```typescript
{
  query: string
  results: SearchResult[] | string   // 没有结果或是评论时为字符串
  durationSeconds: number
}

type SearchResult = {
  title: string
  url: string
  snippet: string
}
```

**权限：** `passthrough`，也就是总会提示用户

**实现：**
- 使用 beta 工具：`web_search_20250305`
- 每次调用最多执行 8 次搜索操作
- 只在 `firstParty`、`vertex` 和 `foundry` API provider 上启用

---

## 10. MCP 集成工具

### 10.1 MCPTool

**工具名：** `mcp`（在每个 server 上会覆盖成 `mcp__<server>__<tool>`）
**源文件：** `src/tools/MCPTool/MCPTool.ts`

**特性：**
- `isMcp: true`
- `maxResultSizeChars: 100_000`
- `permission: 'passthrough'`（总是询问）
- 通过 `mcpClient.ts` 按 server 实例化时，会覆盖所有方法

**输入模式：** `z.object({}).passthrough()`，也就是接受任意对象
**输出：** `string`

**导出：**
```typescript
type MCPProgress  // 重新导出自 mcp progress types
```

---

### 10.2 McpAuthTool

**工具名：** `mcp__<serverName>__authenticate`
**源文件：** `src/tools/McpAuthTool/McpAuthTool.ts`

这是一个伪工具工厂，不是标准的 `buildTool()` 实例。

```typescript
function createMcpAuthTool(
  serverName: string,
  config: ScopedMcpServerConfig,
): Tool<InputSchema, McpAuthOutput>
```

**输入：** `{}`（空）

**输出模式：**

```typescript
type McpAuthOutput = {
  status: 'auth_url' | 'unsupported' | 'error'
  message: string
  authUrl?: string    // 当 status='auth_url' 时存在
}
```

**行为：**
- 适用于已经安装但还需要 OAuth 认证的 MCP server
- 启动 `performMCPOAuthFlow()`，并且 `skipBrowserOpen: true`
- 返回授权 URL，用户可以自己在浏览器里打开
- 后台续接：OAuth 完成后，调用 `reconnectMcpServerImpl()`，再通过前缀替换把真实工具换回 app state
- `claudeai-proxy` transport：返回 `'unsupported'`，并引导用户去 `/mcp`
- 静默认证（缓存的 IdP token）：返回成功消息，不给 URL

---

### 10.3 ListMcpResourcesTool

**工具名：** `mcp__listResources`（`LIST_MCP_RESOURCES_TOOL_NAME`）
**源文件：** `src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts`
**特性：** `shouldDefer: true`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `server` | `string` | 否 | 按 server 名称过滤 |

**输出模式：**

```typescript
Array<{
  uri: string
  name: string
  mimeType?: string
  description?: string
  server: string
}>
```

**说明：** 不会直接放进 `getTools()`；只有当 MCP server 带有 resources 时才会加入。

---

### 10.4 ReadMcpResourceTool

**工具名：** `ReadMcpResourceTool`
**源文件：** `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts`
**特性：** `shouldDefer: true`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `server` | `string` | 是 | MCP server 名称 |
| `uri` | `string` | 是 | 要读取的 resource URI |

**输出模式：**

```typescript
{
  contents: Array<{
    uri: string
    mimeType?: string
    text?: string
    blobSavedTo?: string    // 二进制 blob 保存到磁盘时的路径
  }>
}
```

**实现：**
- 二进制 blob 会保存到磁盘；`getBinaryBlobSavedMessage()` 会返回路径引用字符串

---

## 11. Plan Mode 工具

### 11.1 EnterPlanModeTool

**工具名：** `EnterPlanMode`
**源文件：** `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`、`isConcurrencySafe: true`、`isReadOnly: true`

**输入：** `{}`（空对象）

**输出模式：**

```typescript
{
  message: string   // 确认消息
}
```

**行为：**
- 把权限模式设成 `'plan'`（只读计划模式）
- 被 `--channels` 标志禁用
- 不能从 agent（teammate）上下文调用，只能主 agent 调用
- 会把当前权限模式保存到 `prePlanMode`，方便 `ExitPlanMode` 恢复

---

### 11.2 ExitPlanModeV2Tool

**工具名：** `ExitPlanMode`（`EXIT_PLAN_MODE_V2_TOOL_NAME`）
**源文件：** `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`

**模型可见输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `allowedPrompts` | `AllowedPrompt[]` | 否 | 预先批准的工具调用（passthrough schema） |

`AllowedPrompt`：
```typescript
{
  tool: 'Bash'
  prompt: string
}
```

**SDK 输入模式**（额外增加）：

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `plan` | `string` | 否 | SDK 模式下的计划文本 |
| `planFilePath` | `string` | 否 | SDK 模式下的计划文件路径 |

**输出模式：**

```typescript
type Output = {
  plan: string
  isAgent: boolean
  filePath?: string
  hasTaskTool?: boolean
  planWasEdited?: boolean
  awaitingLeaderApproval?: boolean
  requestId?: string
}
```

**行为：**
- 对于需要 `isPlanModeRequired()` 的 teammate：向 team-lead 的 mailbox 发送 `plan_approval_request` 消息；返回 `awaitingLeaderApproval: true` 和一个 `requestId`
- 对于非 teammate：把权限模式恢复到 `prePlanMode`（通常是 `default` 或 `auto`）
- 被 `--channels` 标志禁用

**导出：**
```typescript
type AllowedPrompt
const _sdkInputSchema  // 包含 plan/planFilePath 的扩展 schema
type Output
```

---

## 12. Notebook 工具

### NotebookEditTool

**工具名：** `NotebookEdit`
**源文件：** `src/tools/NotebookEditTool/NotebookEditTool.ts`
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `notebook_path` | `string` | 是 | `.ipynb` 文件的绝对路径 |
| `cell_id` | `string` | 否 | 目标 cell ID（编辑 / 删除时必填；新增 cell 时省略） |
| `new_source` | `string` | 是 | 这个 cell 的新源代码内容 |
| `cell_type` | `'code' \| 'markdown'` | 否 | cell 类型（用于新 cell） |
| `edit_mode` | `'replace' \| 'insert' \| 'delete'` | 否（默认 `'replace'`） | 编辑操作 |

**输出模式：**

```typescript
{
  new_source: string
  cell_id?: string
  cell_type: 'code' | 'markdown'
  language: string
  edit_mode: 'replace' | 'insert' | 'delete'
  error?: string
  notebook_path: string
  original_file: string   // 编辑前完整的 notebook JSON
  updated_file: string    // 编辑后完整的 notebook JSON
}
```

**校验 / 安全：**
- 需要 read-before-write（和 FileEditTool / FileWriteTool 一样）
- mtime 过期检查
- UNC 路径安全跳过
- 在 cell 替换时清空 `execution_count` 和 `outputs`（避免显示旧输出）

---

## 13. Worktree 工具

### 13.1 EnterWorktreeTool

**工具名：** `EnterWorktree`（`ENTER_WORKTREE_TOOL_NAME`）
**源文件：** `src/tools/EnterWorktreeTool/EnterWorktreeTool.ts`
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `name` | `string` | 否 | worktree 的可选名字。每个 `/` 分段只能包含字母、数字、点、下划线和短横线；总长最多 64 个字符。如果不提供，会自动生成随机名字。 |

**输出模式：**

```typescript
{
  worktreePath: string
  worktreeBranch?: string
  message: string
}
```

**行为：**
- 先确认当前不是这个 session 已创建的 worktree 会话
- 创建 worktree 前会先解析到主仓库根目录
- 调用 `createWorktreeForSession(sessionId, slug)` 创建 git worktree（或者 hook 驱动的 worktree）
- 把 `cwd`、`originalCwd` 更新为 worktree 路径
- 清空 system prompt sections cache（这样 `env_info_simple` 会用 worktree 上下文重新计算）
- 清空依赖 cwd 的 memoized cache
- 上报 `tengu_worktree_created` 分析事件

---

### 13.2 ExitWorktreeTool

**工具名：** `ExitWorktree`（`EXIT_WORKTREE_TOOL_NAME`）
**源文件：** `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts`
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `action` | `'keep' \| 'remove'` | 是 | `'keep'` 保留磁盘上的 worktree 和 branch；`'remove'` 两者都删掉 |
| `discard_changes` | `boolean` | 否 | 当 `action='remove'` 且 worktree 有未提交文件或未合并提交时，必须是 `true` |

**输出模式：**

```typescript
{
  action: 'keep' | 'remove'
  originalCwd: string
  worktreePath: string
  worktreeBranch?: string
  tmuxSessionName?: string
  discardedFiles?: number     // action='remove' 时设置
  discardedCommits?: number   // action='remove' 时设置
  message: string
}
```

**校验：**
- 只对当前 session 里由 `EnterWorktreeTool` 创建的 worktree 生效（通过 `getCurrentWorktreeSession()` 做作用域保护）
- 当 `action='remove'` 且没有 `discard_changes: true` 时：
  - 调用 `countWorktreeChanges()`：使用 `git status --porcelain` + `git rev-list --count <originalHead>..HEAD`
  - 如果有未提交文件或未合并提交，会返回错误并列出来
  - 如果 git 状态无法判断，也会返回错误（fail-closed）

**行为：**
- `action='keep'`：调用 `keepWorktree()`，恢复到原始 cwd，并保留 worktree 以便后续继续用
- `action='remove'`：如果有 tmux session 就先杀掉，然后调用 `cleanupWorktree()`（删除 worktree 和 branch），再恢复会话
- 两种动作都会恢复 `cwd`、`originalCwd`，必要时也恢复 `projectRoot`，并清空 cache
- 上报 `tengu_worktree_kept` 或 `tengu_worktree_removed` 分析事件

---

## 14. 调度工具

这些工具受 `isKairosCronEnabled()` 门控（需要 `feature('KAIROS')` + GB gate）。

### 14.1 CronCreateTool

**工具名：** `CronCreate`（`CRON_CREATE_TOOL_NAME`）
**源文件：** `src/tools/ScheduleCronTool/CronCreateTool.ts`
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `cron` | `string` | 是 | 标准 5 段 cron 表达式，使用本地时间：`"M H DoM Mon DoW"` |
| `prompt` | `string` | 是 | 每次触发时要入队的 prompt |
| `recurring` | `boolean` | 否（默认 `true`） | `true` = 每次 cron 命中都触发（到 `DEFAULT_MAX_AGE_DAYS` 后自动过期）；`false` = 触发一次后自动删除 |
| `durable` | `boolean` | 否（默认 `false`） | `true` = 持久化到 `.claude/scheduled_tasks.json`，重启后仍在；`false` = 只存在于内存，session 结束即消失 |

**输出模式：**

```typescript
{
  id: string            // 供 CronDelete/CronList 引用的 job ID
  humanSchedule: string // 人类可读的 schedule（例如 "Every 5 minutes"）
  recurring: boolean
  durable?: boolean
}
```

**校验：**
- 必须是合法的 5 段 cron 表达式
- 这个表达式必须在未来一年内至少命中一次日历日期
- 最多允许 50 个并发定时任务
- Durable cron 不支持 teammate（teammate 不会跨 session 持久）

**常量：**
- `MAX_JOBS: 50`
- `DEFAULT_MAX_AGE_DAYS`：定义在 prompt.ts 里

---

### 14.2 CronDeleteTool

**工具名：** `CronDelete`
**源文件：** `src/tools/ScheduleCronTool/CronDeleteTool.ts`
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `id` | `string` | 是 | `CronCreate` 返回的 job ID |

**输出模式：**

```typescript
{
  id: string    // 被取消的 job ID
}
```

**校验：**
- 指定 ID 的 job 必须存在
- Teammate 只能删除自己的 cron job（由 `agentId` 强制所有权）

---

### 14.3 CronListTool

**工具名：** `CronList`（`CRON_LIST_TOOL_NAME`）
**源文件：** `src/tools/ScheduleCronTool/CronListTool.ts`
**特性：** `shouldDefer: true`、`isConcurrencySafe: true`、`isReadOnly: true`、`maxResultSizeChars: 100_000`
**输入：** `{}`（空对象）

**输出模式：**

```typescript
{
  jobs: Array<{
    id: string
    cron: string
    humanSchedule: string
    prompt: string
    recurring?: boolean
    durable?: boolean
  }>
}
```

**行为：**
- Teammate 只能看到自己的 cron jobs（按 `agentId` 过滤）
- Team lead（没有 teammate 上下文）能看到全部 jobs

---

## 15. 元工具 / 发现工具

### 15.1 ToolSearchTool

**工具名：** `ToolSearch`（`TOOL_SEARCH_TOOL_NAME`）
**源文件：** `src/tools/ToolSearchTool/ToolSearchTool.ts`
**特性：** `maxResultSizeChars: 100_000`、`isConcurrencySafe: true`、`isReadOnly: true`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `query` | `string` | 是 | `select:<name>` 表示按名字精确查找工具；也可以用关键词做模糊搜索 |
| `max_results` | `number` | 否（默认 `5`） | 最多返回多少个工具 |

**输出模式：**

```typescript
{
  matches: string[]              // 匹配到的工具名
  query: string
  total_deferred_tools: number
  pending_mcp_servers?: string[] // 仍在加载中的 MCP server
}
```

**评分算法：**

| 匹配类型 | 内置分值 | MCP 分值 |
|------------|---------------|-----------|
| 名称局部精确匹配 | 10 | 12 |
| 名称子串匹配 | 5 | 6 |
| `searchHint` 的单词边界匹配 | 4 | — |
| description 匹配 | 2 | — |

**行为：**
- `select:<name>` 前缀：按精确名称查找，会拿到完整 schema 定义
- 关键词：在所有延迟加载工具里做模糊评分
- `mapToolResultToToolResultBlockParam`：会返回 matched tools 的 `tool_reference` block，并把它们的 schema 注入上下文

**导出：**
```typescript
function clearToolSearchDescriptionCache(): void
```

---

### 15.2 AskUserQuestionTool

**工具名：** `AskUserQuestion`（`ASK_USER_QUESTION_TOOL_NAME`）
**源文件：** `src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx`
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`、`requiresUserInteraction: true`

**模型可见输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `questions` | `Question[]` | 是 | 1–4 个要问用户的问题 |
| `answers` | `object` | 否 | 用户回复后由 UI 注入（不在模型可见 schema 里） |
| `annotations` | `object` | 否 | 元数据注释 |
| `metadata` | `object` | 否 | 任意元数据 |

`Question` 模式：
```typescript
{
  question: string
  header?: string
  options: QuestionOption[]   // 2-4 个选项
  multiSelect?: boolean
}

type QuestionOption = {
  label: string
  value: string
  description?: string
}
```

**输出模式：**

```typescript
{
  questions: Question[]
  answers: Record<string, string | string[]>  // question → answer(s)
  annotations?: object
}
```

**行为：**
- `--channels` 标志开启时会禁用（没有终端可弹对话框）
- UI 会渲染交互式问题对话框；`answers` 字段由 UI 层注入
- 支持单选和多选两种问题类型

**导出：**
```typescript
const _sdkInputSchema   // 包含 answers 字段的完整输入 schema
const _sdkOutputSchema  // 完整输出 schema
type Question
type QuestionOption
```

---

## 16. Kairos / 特殊模式工具

### 16.1 BriefTool（SendUserMessage）

**工具名：** `SendUserMessage`（`BRIEF_TOOL_NAME`，别名：`LEGACY_BRIEF_TOOL_NAME`）
**源文件：** `src/tools/BriefTool/BriefTool.ts`
**门控：** `isBriefEnabled()` —— 需要 `feature('KAIROS')` 或 `feature('KAIROS_BRIEF')` + Growthbook gate + `userMsgOptIn` 或 `kairosActive`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `message` | `string`（markdown） | 是 | 要发给用户的消息 |
| `attachments` | `string[]` | 否 | 要附加的文件路径 |
| `status` | `'normal' \| 'proactive'` | 是 | 消息状态：`'proactive'` 表示主动更新 |

**输出模式：**

```typescript
type Output = {
  message: string
  attachments?: Array<{
    path: string
    size: number
    isImage: boolean
    file_uuid: string
  }>
  sentAt?: string   // ISO 时间戳
}
```

**导出：**
```typescript
function isBriefEntitled(): boolean  // 是否具备权限
function isBriefEnabled(): boolean   // 是否具备权限且功能开关开启
type Output
```

---

### 16.2 SleepTool

**工具名：** `Sleep`
**源文件：** `src/tools/SleepTool/`（这里只存在 `prompt.ts`；工具实现会在 `feature('SLEEP_TOOL')` 时通过 `require()` 加载）
**门控：** `feature('SLEEP_TOOL')`

**用途：** 等待指定时长，但不会一直占着 shell 进程。可以被用户中断。

**prompt 要点：**
- 当用户说要 sleep / rest、在等某件事、或者暂时没别的事时用
- 可以和其他工具并发调用
- 比 `Bash(sleep ...)` 更适合，因为不会占住 shell 进程
- 睡眠期间会定期收到 `<tick>` 提示；睡前要先检查有没有别的工作
- 每次唤醒都会消耗一次 API 调用；prompt cache 在 5 分钟无操作后过期

---

### 16.3 RemoteTriggerTool

**工具名：** `RemoteTrigger`（`REMOTE_TRIGGER_TOOL_NAME`）
**源文件：** `src/tools/RemoteTriggerTool/RemoteTriggerTool.ts`
**门控：** `feature('AGENT_TRIGGERS')` + `getFeatureValue_CACHED_MAY_BE_STALE('tengu_surreal_dali', false)` + `isPolicyAllowed('allow_remote_sessions')`
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`、`isConcurrencySafe: true`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `action` | `'list' \| 'get' \| 'create' \| 'update' \| 'run'` | 是 | 对 triggers 做 CRUD 操作 |
| `trigger_id` | `string` | 否 | `get`、`update`、`run` 必填（正则：`[\w-]+`） |
| `body` | `Record<string, unknown>` | 否 | `create` 和 `update` 的 JSON body |

**输出模式：**

```typescript
{
  status: number   // HTTP 状态码
  json: string     // 序列化后的响应体
}
```

**实现：**
- 调用 `${BASE_API_URL}/v1/code/triggers` REST API
- 使用 OAuth token（`checkAndRefreshOAuthTokenIfNeeded()` + `getClaudeAIOAuthTokens()`）
- 需要 org UUID（`getOrganizationUUID()`）
- API beta header：`ccr-triggers-2026-01-30`
- 超时：`20_000ms`
- `isReadOnly`：`list` 和 `get` 时为 `true`，其他操作为 `false`

---

## 17. SDK / 输出工具

### 17.1 SyntheticOutputTool（StructuredOutput）

**工具名：** `StructuredOutput`
**源文件：** `src/tools/SyntheticOutputTool/SyntheticOutputTool.ts`
**门控：** 仅在非交互式会话（SDK / `--output-format json` 模式）中启用

**输入：** passthrough，也就是任意对象，随后会通过 AJV 按给定 JSON schema 校验
**输出：** `'Structured output provided successfully'`（字符串）

**工厂：**

```typescript
function createSyntheticOutputTool(jsonSchema: object): Tool
```

- 使用 `WeakMap` 做缓存（同一个 schema 对象会返回同一个 tool）
- 在 SDK / `--output-format json` 模式下使用，强制模型输出结构化最终结果
- AJV 校验确保输出和调用方提供的 JSON schema 一致

---

## 18. Skill Tool

**工具名：** `Skill`（`SKILL_TOOL_NAME`）
**源文件：** `src/tools/SkillTool/SkillTool.ts`
**特性：** `maxResultSizeChars: 100_000`

**用途：** 运行本地文件或 MCP skill server 里定义的 prompt 命令（skills）。

**导出：**
```typescript
type Progress  // SkillToolProgress 的重新导出
```

---

## 19. LSP 工具

**工具名：** `LSP`（`LSP_TOOL_NAME`）
**源文件：** `src/tools/LSPTool/LSPTool.ts`
**门控：** `ENABLE_LSP_TOOL=true` 环境变量
**特性：** `isLsp: true`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `operation` | `string` | 是 | 下列之一：`goToDefinition`、`findReferences`、`hover`、`documentSymbol`、`workspaceSymbol`、`goToImplementation`、`prepareCallHierarchy`、`incomingCalls`、`outgoingCalls` |
| `filePath` | `string` | 是 | 文件绝对路径 |
| `line` | `number` | 是 | 从 1 开始的行号 |
| `character` | `number` | 是 | 从 1 开始的字符位置 |

**限制：**
- 最大文件大小：10 MB

---

## 20. REPL 工具

**工具名：** `REPL`
**源文件：** `src/tools/REPLTool/`
**门控：** 仅 Ant 可用（`USER_TYPE === 'ant'`）+ 通过 `require()` 加载

**常量（`src/tools/REPLTool/constants.ts`）：**

```typescript
const REPL_TOOL_NAME = 'REPL'

function isReplModeEnabled(): boolean
// 只有在：CLAUDE_CODE_REPL 不为 falsy 且（CLAUDE_REPL_MODE=1 或（USER_TYPE='ant' 且 CLAUDE_CODE_ENTRYPOINT='cli'））时为 true
// SDK 入口默认是关闭 REPL 模式

const REPL_ONLY_TOOLS = new Set([
  'Read', 'Write', 'Edit', 'Glob', 'Grep', 'Bash', 'NotebookEdit', 'Agent',
])
// 在 REPL 模式下对模型隐藏；模型必须用 REPL 进行批量操作
```

**原始工具（`src/tools/REPLTool/primitiveTools.ts`）：**

```typescript
function getReplPrimitiveTools(): readonly Tool[]
// 返回：[FileReadTool, FileWriteTool, FileEditTool, GlobTool, GrepTool, BashTool, NotebookEditTool, AgentTool]
// 使用 lazy getter 避免 TDZ 循环依赖
// 即便对模型隐藏，这些工具在 REPL VM 上下文里仍然可用
```

---

## 21. Config 工具

**工具名：** `Config`（`CONFIG_TOOL_NAME`）
**源文件：** `src/tools/ConfigTool/ConfigTool.ts`
**门控：** 仅 Ant 可用
**特性：** `shouldDefer: true`、`maxResultSizeChars: 100_000`

**输入模式：**

| 参数 | 类型 | 必填 | 说明 |
|-----------|------|------|-------------|
| `setting` | `string` | 是 | 配置键（例如 `"theme"`、`"model"`） |
| `value` | `string \| boolean \| number` | 否 | 新值（GET 操作时省略） |

**输出模式：**

```typescript
{
  success: boolean
  operation?: 'get' | 'set'
  setting?: string
  value?: unknown
  previousValue?: unknown
  newValue?: unknown
  error?: string
}
```

**权限：**
- GET 操作：自动允许
- SET 操作：需要用户权限提示

**来源：** 配置可以来自 `'global'` config 或 `'settings'` config。

**导出：**
```typescript
type Input
type Output
```

---

## 22. 共享工具函数

### 22.1 tools/utils.ts

```typescript
/**
 * 给用户消息打上 sourceToolUseID 标记，这样在工具完成前它们会保持短暂状态。
 * 这样可以避免 UI 里出现重复的 “is running” 消息。
 */
function tagMessagesWithToolUseID(
  messages: (UserMessage | AttachmentMessage | SystemMessage)[],
  toolUseID: string | undefined,
): (UserMessage | AttachmentMessage | SystemMessage)[]

/**
 * 从父消息里提取某个工具名对应的 tool use ID。
 */
function getToolUseIDFromParentMessage(
  parentMessage: AssistantMessage,
  toolName: string,
): string | undefined
```

---

### 22.2 tools/shared/gitOperationTracking.ts

和 shell 无关的 git 操作跟踪模块，用于统计使用情况。BashTool 和 PowerShellTool 都一样会用到。

**导出类型：**

```typescript
type CommitKind = 'committed' | 'amended' | 'cherry-picked'
type BranchAction = 'merged' | 'rebased'
type PrAction = 'created' | 'edited' | 'merged' | 'commented' | 'closed' | 'ready'
```

**关键函数：**

```typescript
/**
 * 扫描命令和输出，找出值得在 UI 摘要里展示的 git 操作。
 * 检测范围：git commit、git push、git merge、git rebase、gh pr *、glab mr create、curl PR API。
 */
function detectGitOperation(
  command: string,
  output: string,
): {
  commit?: { sha: string; kind: CommitKind }
  push?: { branch: string }
  branch?: { ref: string; action: BranchAction }
  pr?: { number: number; url?: string; action: PrAction }
}

/**
 * 上报分析事件和 OTLP 计数器。
 * 在每个 Bash/PowerShell 命令完成后调用（且只在 exit code 0 时）。
 */
function trackGitOperations(
  command: string,
  exitCode: number,
  stdout?: string,
): void

// 供测试导出
function parseGitCommitId(stdout: string): string | undefined
```

**检测到的操作：**
- `git commit` → `tengu_git_operation{operation: 'commit'}`，并递增 commit OTLP 计数器
- `git commit --amend` → 还会额外触发 `tengu_git_operation{operation: 'commit_amend'}`
- `git push` → `tengu_git_operation{operation: 'push'}`
- `gh pr create/edit/merge/comment/close/ready` → `tengu_git_operation{operation: 'pr_<action>'}`，并创建 PR OTLP 计数器 + 把 session 关联到 PR URL
- `glab mr create` → `tengu_git_operation{operation: 'pr_create'}`，递增 PR OTLP 计数器
- `curl POST` 到 PR endpoints → `tengu_git_operation{operation: 'pr_create'}`

**Git 命令正则：** 允许 `git` 和子命令之间夹带全局选项，例如 `git -c commit.gpgsign=false commit`。

---

### 22.3 tools/shared/spawnMultiAgent.ts

从 TeammateTool 中抽出来的共享模块，用于 teammate / subagent 创建，供 AgentTool 复用。

**关键函数：**

```typescript
// 内部辅助
function getDefaultTeammateModel(leaderModel: string | null): string
// 检查 globalConfig.teammateDefaultModel；null → 跟随 leader；undefined → 使用硬编码回退值
```

**后端类型：**
- `in-process`：把 teammate 作为进程内协程启动（不拉外部进程）
- 基于 tmux 的 pane：在 swarm session 里新开一个 tmux pane
- 外部进程后端

**检测：**
- `detectAndGetBackend()`：探测可用后端
- `isInProcessEnabled()`：检查是否能用进程内启动
- `isTmuxAvailable()`：检查 pane 后端是否可用

**环境继承：**
- `buildInheritedEnvVars()`：为启动的 teammate 构建环境变量集合
- 传递的关键环境变量：`TEAMMATE_COMMAND_ENV_VAR`、模型覆盖、插件路径等

---

## 23. 测试工具

### TestingPermissionTool

**工具名：** `TestingPermission`
**源文件：** `src/tools/testing/TestingPermissionTool.tsx`
**门控：** 只在 `NODE_ENV === 'test'` 时启用（硬编码：`"production" === 'test'`，所以生产环境永远禁用）

```typescript
export const TestingPermissionTool: Tool<InputSchema, string>
```

**输入：** `{}`（空对象）
**输出：** `'TestingPermission executed successfully'`

**行为：**
- `checkPermissions()` 永远返回 `{ behavior: 'ask', message: 'Run test?' }`
- 用于端到端权限对话框测试
- 所有 render 函数都返回 `null`
- `isConcurrencySafe: true`、`isReadOnly: true`
- 永远不会出现在生产工具列表里（构建时就禁用）

---

## 附录：工具名常量

| 常量 | 值 | 源文件 |
|----------|-------|--------|
| `BASH_TOOL_NAME` | `'Bash'` | `BashTool/toolName.ts` |
| `FILE_READ_TOOL_NAME` | `'Read'` | `FileReadTool/prompt.ts` |
| `FILE_WRITE_TOOL_NAME` | `'Write'` | `FileWriteTool/prompt.ts` |
| `FILE_EDIT_TOOL_NAME` | `'Edit'` | `FileEditTool/constants.ts` |
| `GLOB_TOOL_NAME` | `'Glob'` | `GlobTool/prompt.ts` |
| `GREP_TOOL_NAME` | `'Grep'` | `GrepTool/prompt.ts` |
| `AGENT_TOOL_NAME` | `'Agent'` | `AgentTool/constants.ts` |
| `LEGACY_AGENT_TOOL_NAME` | `'Task'` | `AgentTool/constants.ts` |
| `NOTEBOOK_EDIT_TOOL_NAME` | `'NotebookEdit'` | `NotebookEditTool/constants.ts` |
| `TASK_OUTPUT_TOOL_NAME` | `'TaskOutput'` | `TaskOutputTool/` |
| `ASK_USER_QUESTION_TOOL_NAME` | `'AskUserQuestion'` | `AskUserQuestionTool/` |
| `SKILL_TOOL_NAME` | `'Skill'` | `SkillTool/` |
| `TOOL_SEARCH_TOOL_NAME` | `'ToolSearch'` | `ToolSearchTool/` |
| `CONFIG_TOOL_NAME` | `'Config'` | `ConfigTool/` |
| `BRIEF_TOOL_NAME` | `'SendUserMessage'` | `BriefTool/` |
| `SLEEP_TOOL_NAME` | `'Sleep'` | `SleepTool/prompt.ts` |
| `REMOTE_TRIGGER_TOOL_NAME` | (来自 `prompt.ts`) | `RemoteTriggerTool/prompt.ts` |
| `ENTER_WORKTREE_TOOL_NAME` | (来自 `constants.ts`) | `EnterWorktreeTool/constants.ts` |
| `EXIT_WORKTREE_TOOL_NAME` | (来自 `constants.ts`) | `ExitWorktreeTool/constants.ts` |
| `TEAM_DELETE_TOOL_NAME` | (来自 `constants.ts`) | `TeamDeleteTool/constants.ts` |
| `CRON_CREATE_TOOL_NAME` | (来自 `prompt.ts`) | `ScheduleCronTool/prompt.ts` |
| `CRON_DELETE_TOOL_NAME` | (来自 `prompt.ts`) | `ScheduleCronTool/prompt.ts` |
| `CRON_LIST_TOOL_NAME` | (来自 `prompt.ts`) | `ScheduleCronTool/prompt.ts` |
| `EXIT_PLAN_MODE_V2_TOOL_NAME` | `'ExitPlanMode'` | `ExitPlanModeTool/` |
| `LIST_MCP_RESOURCES_TOOL_NAME` | `'mcp__listResources'` | `ListMcpResourcesTool/` |
| `POWERSHELL_TOOL_NAME` | `'PowerShell'` | `PowerShellTool/toolName.ts` |
| `REPL_TOOL_NAME` | `'REPL'` | `REPLTool/constants.ts` |
| `LSP_TOOL_NAME` | `'LSP'` | `LSPTool/` |

---

## 附录：工具特性门控汇总

| 工具 | 门控 / 条件 |
|------|-----------------|
| `ConfigTool`、`REPLTool` | `USER_TYPE === 'ant'` |
| `CronCreate/Delete/List` | `feature('KAIROS')` + `isKairosCronEnabled()` GB gate |
| `SleepTool` | `feature('SLEEP_TOOL')` |
| `RemoteTriggerTool` | `feature('AGENT_TRIGGERS')` + `tengu_surreal_dali` GB flag + `allow_remote_sessions` policy |
| `BriefTool` | `feature('KAIROS')` 或 `feature('KAIROS_BRIEF')` + GB gate + `userMsgOptIn` 或 `kairosActive` |
| `TeamCreate/Delete`、`SendMessage` | `isAgentSwarmsEnabled()` |
| `TaskCreate/Get/Update/List` | `isTodoV2Enabled()` |
| `TodoWriteTool` | `!isTodoV2Enabled()` |
| `LSPTool` | `ENABLE_LSP_TOOL=true` 环境变量 |
| `TestingPermissionTool` | `NODE_ENV === 'test'`（生产环境始终禁用） |
| `MonitorMcpTask` | `feature('MONITOR_TOOL')` |
| `LocalWorkflowTask` | `feature('WORKFLOW_SCRIPTS')` |

---

## 附录：deferred vs. alwaysLoad

带有 `shouldDefer: true` 的工具会在初始 prompt 上下文中隐藏，以节省 token。模型会通过 `ToolSearchTool` 用关键词搜索，或者用 `select:<name>` 查询来发现它们。`ToolSearchTool` 会把匹配到的工具以 `tool_reference` block 的形式注入，这样工具 schema 就会进入上下文。

带有 `alwaysLoad: true` 的工具即使在默认会 defer 的上下文里也会始终包含。

没有这两个标志的工具会默认出现在初始 prompt 里。

**工具名：** `CronDelete`
