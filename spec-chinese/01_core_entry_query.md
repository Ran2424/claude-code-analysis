# Claude Code — 核心入口点与查询系统

## 目录

1. [entrypoints/cli.tsx — 引导调度器](#entrypointsclisx--引导调度器)
2. [main.tsx — 完整 CLI 入口点](#maintsx--完整-cli-入口点)
3. [replLauncher.tsx — REPL UI 启动器](#repplaunchertsx--repl-ui-启动器)
4. [entrypoints/init.ts — 初始化与遥测](#entrypointsinits--初始化与遥测)
5. [entrypoints/mcp.ts — MCP 服务器入口点](#entrypointsmcpts--mcp-服务器入口点)
6. [entrypoints/agentSdkTypes.ts — Agent SDK 公共 API](#entrypointsagentsdktypests--agent-sdk-公共-api)
7. [entrypoints/sandboxTypes.ts — 沙盒配置类型](#entrypointssandboxtypests--沙盒配置类型)
8. [entrypoints/sdk/coreSchemas.ts — SDK 核心 Zod 模式](#entrypointssdkcoreshematss--sdk-核心-zod-模式)
9. [entrypoints/sdk/coreTypes.ts — SDK 核心 TypeScript 类型](#entrypointssdkcoretypests--sdk-核心-typescript-类型)
10. [entrypoints/sdk/controlSchemas.ts — SDK 控制协议模式](#entrypointssdkcontrolschematss--sdk-控制协议模式)
11. [query.ts — 核心异步查询循环](#queryts--核心异步查询循环)
12. [QueryEngine.ts — 有状态查询引擎 (SDK/Headless)](#queryenginets--有状态查询引擎-sdkheadless)
13. [query/config.ts — 查询配置快照](#queryconfigts--查询配置快照)
14. [query/deps.ts — 查询依赖注入](#querydepsts--查询依赖注入)
15. [query/stopHooks.ts — 停止钩子编排](#querystophooksts--停止钩子编排)
16. [query/tokenBudget.ts — 令牌预算跟踪](#querytokenbudgetts--令牌预算跟踪)
17. [context.ts — 系统与用户上下文提供者](#contextts--系统与用户上下文提供者)
18. [history.ts — 提示历史管理](#historyts--提示历史管理)
19. [cost-tracker.ts — 会话成本跟踪](#cost-trackerts--会话成本跟踪)
20. [costHook.ts — React 成本摘要钩子](#costhookts--react-成本摘要钩子)
21. [projectOnboardingState.ts — 项目引导状态](#projectonboardingstatets--项目引导状态)
22. [bootstrap/state.ts — 全局会话状态](#bootstrapstatets--全局会话状态)
23. [assistant/sessionHistory.ts — 远程会话历史分页](#assistantsessionhistoryts--远程会话历史分页)

---

## entrypoints/cli.tsx — 引导调度器

### 用途

当用户运行 `claude` 时执行的第一个模块。作为轻量级引导调度器，在加载任何重量级模块之前，检查进程参数以寻找已知的快速路径。每个快速路径仅动态导入所需内容。只有当没有快速路径匹配时，才会加载完整的 CLI (`main.tsx`)。

### 关键流程

```
process.argv 解析
  ├── --version / -v → 打印 MACRO.VERSION，退出（零导入）
  ├── --dump-system-prompt → 转储渲染后的系统提示（仅限 ant，DUMP_SYSTEM_PROMPT 特性）
  ├── --claude-in-chrome-mcp → runClaudeInChromeMcpServer()
  ├── --chrome-native-host → runChromeNativeHost()
  ├── --computer-use-mcp → runComputerUseMcpServer()（CHICAGO_MCP 特性）
  ├── --daemon-worker=<kind> → runDaemonWorker()（DAEMON 特性）
  ├── remote-control|rc|remote|sync|bridge → bridgeMain()（BRIDGE_MODE 特性）
  ├── daemon → daemonMain()（DAEMON 特性）
  ├── ps|logs|attach|kill|--bg|--background → 后台处理程序（BG_SESSIONS 特性）
  ├── new|list|reply → templatesMain()（TEMPLATES 特性）
  ├── environment-runner → environmentRunnerMain()（BYOC_ENVIRONMENT_RUNNER 特性）
  ├── self-hosted-runner → selfHostedRunnerMain()（SELF_HOSTED_RUNNER 特性）
  ├── --worktree + --tmux → execIntoTmuxWorktree()
  └── （默认）→ startCapturingEarlyInput() → 导入 main.tsx → cliMain()
```

### 顶层副作用（在模块加载时）

| 副作用 | 用途 |
|---|---|
| `process.env.COREPACK_ENABLE_AUTO_PIN = '0'` | 防止 yarnpkg 被添加到 package.json |
| `process.env.NODE_OPTIONS += '--max-old-space-size=8192'` | CCR 环境（16GB 容器）堆大小 |
| `ABLATION_BASELINE` 标志 | 为 harness-science L0 消融设置多个 `CLAUDE_CODE_*` 环境变量 |

### 导出

```typescript
// 无命名导出 — 模块在底部执行 main() IIFE
async function main(): Promise<void>  // 内部，不导出
```

### 检查的特性标志

- `DUMP_SYSTEM_PROMPT` — 仅限 ant 的系统提示转储
- `CHICAGO_MCP` — 计算机使用 MCP 服务器
- `DAEMON` — 守护进程工作器和监督器
- `BRIDGE_MODE` — 远程控制桥接
- `BG_SESSIONS` — 后台会话管理
- `TEMPLATES` — 模板作业命令
- `BYOC_ENVIRONMENT_RUNNER` — BYOC 无头运行器
- `SELF_HOSTED_RUNNER` — 自托管运行器
- `ABLATION_BASELINE` — harness-science 基线

### 依赖项

- `bun:bundle` (`feature`)
- `../utils/startupProfiler.js`
- `../utils/config.js` (enableConfigs)
- 各种快速路径模块动态加载

---

## main.tsx — 完整 CLI 入口点

### 用途

主要的 CLI 模块。仅在 `entrypoints/cli.tsx` 确定没有快速路径匹配后加载。定义 Commander.js 命令树，处理所有 CLI 标志，协调启动（迁移、信任对话框、MCP 配置、工具加载），并启动交互式 REPL 或无头/打印 (`-p`) 路径。

### 导出函数

```typescript
export async function main(): Promise<void>
export function startDeferredPrefetches(): void
```

#### `main()`

完整 CLI 的主要入口点。按顺序负责：

1. **安全性**：设置 `process.env.NoDefaultCurrentDirectoryInExePath = '1'`（Windows PATH 攻击防护）
2. **警告处理程序**：`initializeWarningHandler()`
3. **信号处理程序**：SIGINT（打印模式下跳过），退出光标重置
4. **早期参数处理**：
   - `cc://` / `cc+unix://` URL 重写（DIRECT_CONNECT 特性）
   - `--handle-uri` 深度链接处理（LODESTONE 特性）
   - `claude assistant [sessionId]` 重写（KAIROS 特性）
   - `claude ssh <host>` 重写（SSH_REMOTE 特性）
5. **设置标志解析**：在 `init()` 之前运行 `eagerLoadSettings()`
6. **Commander.js 设置**：定义完整的命令树（见下面的 CLI 标志）
7. **`init()`** 调用：验证配置，设置网络，遥测加载承诺
8. **迁移运行**：`runMigrations()` 在版本 `CURRENT_MIGRATION_VERSION = 11`
9. **信任检查**：如果之前未接受，则显示信任对话框
10. **遥测初始化**：`initializeTelemetryAfterTrust()`
11. **会话设置**：模型、权限、MCP 服务器、工具、智能体
12. **启动**：要么 `showSetupScreens()` + `launchRepl()`，要么无头 `runHeadless()`

#### `startDeferredPrefetches()`

在第一次 REPL 渲染后调用，以避免阻塞初始绘制。在以下情况下跳过：
- `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER=1`
- `--bare` 模式（isBareMode()）

预取（触发即忘）：
- `initUser()`
- `getUserContext()`
- `prefetchSystemContextIfSafe()`
- `getRelevantTips()`
- AWS/GCP 凭据（如果启用 Bedrock/Vertex）
- `countFilesRoundedRg()`（3 秒超时）
- `initializeAnalyticsGates()`
- `prefetchOfficialMcpUrls()`
- `refreshModelCapabilities()`
- `settingsChangeDetector.initialize()`
- `skillChangeDetector.initialize()`（非 bare 模式）

### 常量

| 常量 | 值 | 用途 |
|---|---|---|
| `CURRENT_MIGRATION_VERSION` | `11` | 运行配置迁移的版本门 |

### CLI 标志（Commander.js）

| 标志 | 类型 | 描述 |
|---|---|---|
| `-p, --print <prompt>` | `string` | 非交互式/无头模式 |
| `--model <model>` | `string` | 模型覆盖 |
| `--fallback-model <model>` | `string` | 回退模型用于重试 |
| `--permission-mode <mode>` | 枚举 | 工具执行的权限模式 |
| `--dangerously-skip-permissions` | `boolean` | 绕过所有权限检查 |
| `--verbose` | `boolean` | 详细输出 |
| `--debug` | `boolean` | 调试模式 |
| `--mcp-config <file>` | `string` | MCP 配置文件路径 |
| `--add-dir <dir>` | `string[]` | CLAUDE.md 的额外目录 |
| `--resume [sessionId]` | `string?` | 恢复之前的会话 |
| `--bare` | `boolean` | 简单/精简模式（无 UI 附加功能） |
| `--settings <path|json>` | `string` | 标志层设置覆盖 |
| `--setting-sources <sources>` | `string` | 允许的设置源 |
| `--output-format <format>` | string | 无头模式的输出格式 |
| `--max-turns <n>` | `number` | 无头模式的最大轮次 |
| `--max-budget-usd <n>` | `number` | 成本预算限制 |
| `--task-budget <n>` | `number` | API 任务预算 |
| `--no-streaming` | `boolean` | 禁用流式传输 |
| `--worktree <branch>` | `string` | Git worktree 模式 |
| `--agent <type>` | `string` | 主线程智能体类型 |
| `--tmux` / `--tmux=classic` | `boolean` | 使用 tmux |
| `--plugin-dir <dir>` | `string[]` | 仅会话插件目录 |
| `--input-format <format>` | string | 输入格式 |
| `--sdk-betas <betas>` | `string` | 逗号分隔的 SDK beta 头 |
| `--allowedTools <tools>` | `string` | 工具白名单（CLI 覆盖） |
| `--disallowedTools <tools>` | `string` | 工具黑名单（CLI 覆盖） |

### 迁移函数

当 `globalConfig.migrationVersion !== 11` 时，在 `runMigrations()` 中执行：

| 函数 | 用途 |
|---|---|
| `migrateAutoUpdatesToSettings()` | 移动自动更新配置 |
| `migrateBypassPermissionsAcceptedToSettings()` | 移动绕过标志 |
| `migrateEnableAllProjectMcpServersToSettings()` | 移动 MCP 启用设置 |
| `resetProToOpusDefault()` | 将 Pro 模型重置为 Opus |
| `migrateSonnet1mToSonnet45()` | 重命名模型字符串 |
| `migrateLegacyOpusToCurrent()` | 升级旧版 Opus |
| `migrateSonnet45ToSonnet46()` | 重命名为 Sonnet 4.6 |
| `migrateOpusToOpus1m()` | 重命名为 Opus 1m |
| `migrateReplBridgeEnabledToRemoteControlAtStartup()` | 重命名桥接标志 |
| `resetAutoModeOptInForDefaultOffer()` | 重置自动模式选择加入（TRANSCRIPT_CLASSIFIER） |
| `migrateFennecToOpus()` | 仅限 ant 的 Fennec 模型迁移 |

### 顶层副作用

```typescript
profileCheckpoint('main_tsx_entry')    // 启动性能分析
startMdmRawRead()                       // 并行 MDM 子进程（plutil/reg query）
startKeychainPrefetch()                 // 并行 macOS 钥匙链读取
```

### 关键内部函数

```typescript
function logManagedSettings(): void
function isBeingDebugged(): boolean
function logSessionTelemetry(): void
function getCertEnvVarTelemetry(): Record<string, boolean>
async function logStartupTelemetry(): Promise<void>
function runMigrations(): void
function prefetchSystemContextIfSafe(): void
function loadSettingsFromFlag(settingsFile: string): void
function loadSettingSourcesFromFlag(settingSourcesArg: string): void
function eagerLoadSettings(): void
function initializeEntrypoint(isNonInteractive: boolean): void
```

### 待定状态类型（特性门控）

```typescript
type PendingConnect = {
  url: string | undefined
  authToken: string | undefined
  dangerouslySkipPermissions: boolean
}

type PendingAssistantChat = {
  sessionId?: string
  discover: boolean
}

type PendingSSH = {
  host: string | undefined
  cwd: string | undefined
  permissionMode: string | undefined
  dangerouslySkipPermissions: boolean
  local: boolean
  extraCliArgs: string[]
}
```

### 特性标志

`DIRECT_CONNECT`, `KAIROS`, `SSH_REMOTE`, `LODESTONE`, `COORDINATOR_MODE`, `TRANSCRIPT_CLASSIFIER`, `BREAK_CACHE_COMMAND`, `HISTORY_SNIP`, `DAEMON`, `BG_SESSIONS`, `TEMPLATES`

---

## replLauncher.tsx — REPL UI 启动器

### 用途

轻量异步启动器，动态导入 `App` 和 `REPL` 组件（避免循环依赖）并将其渲染到 Ink 根节点。作为单独文件存在，以便 `App` 和 `REPL` 被延迟加载。

### 导出

```typescript
export async function launchRepl(
  root: Root,
  appProps: AppWrapperProps,
  replProps: REPLProps,
  renderAndRun: (root: Root, element: React.ReactNode) => Promise<void>,
): Promise<void>
```

### 类型

```typescript
type AppWrapperProps = {
  getFpsMetrics: () => FpsMetrics | undefined
  stats?: StatsStore
  initialState: AppState
}
```

### 实现

动态导入 `./components/App.js` 和 `./screens/REPL.js`，然后调用 `renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)`。

### 依赖项

- `react`
- `./context/stats.js`（仅类型）
- `./ink.js`（仅类型）
- `./screens/REPL.js`（仅类型）
- `./state/AppStateStore.js`（仅类型）
- `./utils/fpsTracker.js`（仅类型）

---

## entrypoints/init.ts — 初始化与遥测

### 用途

处理所有早期初始化任务，这些任务必须在 CLI 安全地进行 API 调用或显示 UI 之前完成。已记忆化，因此每个进程仅运行一次。遥测初始化延迟到信任建立之后。

### 导出

```typescript
export const init: () => Promise<void>  // 使用 lodash-es/memoize 记忆化
export function initializeTelemetryAfterTrust(): void
```

#### `init()` — 记忆化异步初始化

运行一次。顺序：

1. `enableConfigs()` — 验证并激活配置系统
2. `applySafeConfigEnvironmentVariables()` — 在信任前应用安全的环境变量
3. `applyExtraCACertsFromConfig()` — 在第一个 TLS 连接之前注入 `NODE_EXTRA_CA_CERTS`
4. `setupGracefulShutdown()` — 在退出时注册刷新/清理
5. `initialize1PEventLogging()` + GrowthBook 刷新监听器（延迟）
6. `populateOAuthAccountInfoIfNeeded()`（触发即忘）
7. `initJetBrainsDetection()`（触发即忘）
8. `detectCurrentRepository()`（触发即忘）
9. `initializeRemoteManagedSettingsLoadingPromise()`（如果符合条件）
10. `initializePolicyLimitsLoadingPromise()`（如果符合条件）
11. `recordFirstStartTime()`
12. `configureGlobalMTLS()`
13. `configureGlobalAgents()`（代理）
14. `preconnectAnthropicApi()` — 与动作处理程序工作重叠 TCP+TLS
15. 上游代理初始化（仅限 CLAUDE_CODE_REMOTE）
16. `setShellIfWindows()` — 在 Windows 上配置 git-bash
17. `registerCleanup(shutdownLspServerManager)`
18. `registerCleanup(cleanupSessionTeams)`（延迟导入）
19. `ensureScratchpadDir()`（如果启用 scratchpad）

**错误处理**：`ConfigParseError` → 显示 `InvalidConfigDialog`（交互式）或 `stderr`（非交互式）。其他错误重新抛出。

#### `initializeTelemetryAfterTrust()`

在信任建立后调用一次。对于远程设置符合条件的用户：等待设置加载，然后在初始化之前调用 `applyConfigEnvironmentVariables()`。对于具有 beta 跟踪的 SDK/无头模式：首先急切初始化。

内部：`doInitializeTelemetry()` → `setMeterState()` → `initializeTelemetry()`（延迟加载的 OpenTelemetry，~400KB）

`AttributedCounter` 工厂：包装 OpenTelemetry `Counter`，在每次 `add()` 调用时始终将 `getTelemetryAttributes()` 与任何附加属性合并。

### 依赖项

- `../bootstrap/state.js`
- `../utils/config.js`
- `../services/lsp/manager.js`
- `../services/oauth/client.js`
- `../services/policyLimits/index.js`
- `../services/remoteManagedSettings/index.js`
- `../utils/apiPreconnect.js`
- `../utils/caCertsConfig.js`
- `../utils/cleanupRegistry.js`
- `../utils/gracefulShutdown.js`
- `../utils/managedEnv.js`
- `../utils/mtls.js`
- `../utils/proxy.js`
- `../utils/telemetry/betaSessionTracing.js`
- `../utils/telemetryAttributes.js`
- `../utils/windowsPaths.js`

---

## entrypoints/mcp.ts — MCP 服务器入口点

### 用途

将 Claude Code 作为 MCP（模型上下文协议）服务器启动，通过 `stdio` 传输暴露 Claude 的内置工具。服务器名称：`claude/tengu`。

### 导出

```typescript
export async function startMCPServer(
  cwd: string,
  debug: boolean,
  verbose: boolean,
): Promise<void>
```

### 实现详情

- **传输**：`StdioServerTransport`（stdin/stdout）
- **能力**：`{ tools: {} }`
- **文件状态缓存**：LRU，限制 100 个文件 / 25 MB
- **暴露的命令**：仅 `review` 命令（通过 `MCP_COMMANDS`）
- **工具暴露**：来自 `getTools(toolPermissionContext)` 的所有工具，权限上下文为空

#### `ListTools` 处理程序

对于每个工具：
1. 调用 `tool.prompt(...)` 获取描述
2. 通过 `zodToJsonSchema()` 将 `tool.inputSchema` 转换为 JSON 模式
3. 将 `tool.outputSchema`（如果存在）转换为 JSON 模式 — 仅当根类型为 `object`（非 `anyOf`/`oneOf`）时包含

#### `CallTool` 处理程序

1. 通过 `getTools(emptyPermissionContext)` 获取工具
2. 按名称查找工具；如果未找到则抛出异常
3. 调用 `tool.isEnabled()`、`tool.validateInput()`，然后 `tool.call()`
4. 构建一个 `ToolUseContext`，包含：
   - `isNonInteractiveSession: true`
   - `thinkingConfig: { type: 'disabled' }`
   - `mcpClients: []`
5. 返回 `{ content: [{ type: 'text', text: result }] }` 或 `{ isError: true, content: [...] }`

### 常量

| 常量 | 值 |
|---|---|
| `READ_FILE_STATE_CACHE_SIZE` | `100` |
| MCP 服务器名称 | `'claude/tengu'` |
| MCP 服务器版本 | `MACRO.VERSION` |

---

## entrypoints/agentSdkTypes.ts — Agent SDK 公共 API

### 用途

Claude Code Agent SDK 类型的主要入口点。重新导出所有公共 SDK 类型，并声明抛出 `'not implemented'` 的存根函数 — 实际实现由真实的 SDK 运行时（CLI 进程）提供。此文件是 SDK 使用者的仅类型接口。

### 导出

```typescript
// 控制协议（alpha）
export type { SDKControlRequest, SDKControlResponse } from './sdk/controlTypes.js'

// 核心类型（通用可序列化）
export * from './sdk/coreTypes.js'

// 运行时类型（回调、接口）
export * from './sdk/runtimeTypes.js'

// 设置类型
export type { Settings } from './sdk/settingsTypes.generated.js'

// 工具类型
export * from './sdk/toolTypes.js'
```

### 导出函数（存根）

```typescript
export function tool<Schema extends AnyZodRawShape>(
  _name: string,
  _description: string,
  _inputSchema: Schema,
  _handler: (args: InferShape<Schema>, extra: unknown) => Promise<CallToolResult>,
  _extras?: { annotations?: ToolAnnotations; searchHint?: string; alwaysLoad?: boolean },
): SdkMcpToolDefinition<Schema>

export function createSdkMcpServer(
  _options: CreateSdkMcpServerOptions,
): McpSdkServerConfigWithInstance

export class AbortError extends Error {}

// V1 API
export function query(_params: {
  prompt: string | AsyncIterable<SDKUserMessage>
  options?: InternalOptions | Options
}): InternalQuery | Query

// V2 API（alpha/不稳定）
export function unstable_v2_createSession(_options: SDKSessionOptions): SDKSession
export function unstable_v2_resumeSession(_sessionId: string, _options: SDKSessionOptions): SDKSession
export async function unstable_v2_prompt(_message: string, _options: SDKSessionOptions): Promise<SDKResultMessage>

// 会话管理
export async function getSessionMessages(_sessionId: string, _options?: GetSessionMessagesOptions): Promise<SessionMessage[]>
export async function listSessions(_options?: ListSessionsOptions): Promise<SDKSessionInfo[]>
export async function getSessionInfo(_sessionId: string, _options?: GetSessionInfoOptions): Promise<SDKSessionInfo | undefined>
export async function renameSession(_sessionId: string, _title: string, _options?: SessionMutationOptions): Promise<void>
export async function tagSession(_sessionId: string, _tag: string | null, _options?: SessionMutationOptions): Promise<void>
```

### 重新导出的常量

```typescript
export const HOOK_EVENTS = [
  'PreToolUse', 'PostToolUse', 'PostToolUseFailure', 'Notification',
  'UserPromptSubmit', 'SessionStart', 'SessionEnd', 'Stop', 'StopFailure',
  'SubagentStart', 'SubagentStop', 'PreCompact', 'PostCompact',
  'PermissionRequest', 'PermissionDenied', 'Setup', 'TeammateIdle',
  'TaskCreated', 'TaskCompleted', 'Elicitation', 'ElicitationResult',
  'ConfigChange', 'WorktreeCreate', 'WorktreeRemove', 'InstructionsLoaded',
  'CwdChanged', 'FileChanged',
] as const

export const EXIT_REASONS = [
  'clear', 'resume', 'logout', 'prompt_input_exit', 'other',
  'bypass_permissions_disabled',
] as const
```

---

## entrypoints/sandboxTypes.ts — 沙盒配置类型

### 用途

沙盒配置类型的单一事实来源。SDK 和设置验证都从此处导入。

### 导出

#### 模式（Zod，通过 `lazySchema`）

```typescript
export const SandboxNetworkConfigSchema   // 可选对象
export const SandboxFilesystemConfigSchema  // 可选对象
export const SandboxSettingsSchema          // 直通对象
```

#### 推断的 TypeScript 类型

```typescript
export type SandboxSettings         // 来自 SandboxSettingsSchema
export type SandboxNetworkConfig    // NonNullable<...>
export type SandboxFilesystemConfig // NonNullable<...>
export type SandboxIgnoreViolations // NonNullable<SandboxSettings['ignoreViolations']>
```

### 模式详情

#### `SandboxNetworkConfigSchema`

| 字段 | 类型 | 描述 |
|---|---|---|
| `allowedDomains` | `string[]?` | 允许的出站域名 |
| `allowManagedDomainsOnly` | `boolean?` | 仅限托管设置的域名强制执行 |
| `allowUnixSockets` | `string[]?` | 仅限 macOS 的 unix 套接字路径 |
| `allowAllUnixSockets` | `boolean?` | 禁用 unix 套接字阻塞 |
| `allowLocalBinding` | `boolean?` | 允许本地端口绑定 |
| `httpProxyPort` | `number?` | HTTP 代理端口 |
| `socksProxyPort` | `number?` | SOCKS 代理端口 |

#### `SandboxFilesystemConfigSchema`

| 字段 | 类型 | 描述 |
|---|---|---|
| `allowWrite` | `string[]?` | 额外允许写入的路径 |
| `denyWrite` | `string[]?` | 额外拒绝写入的路径 |
| `denyRead` | `string[]?` | 额外拒绝读取的路径 |
| `allowRead` | `string[]?` | 在 denyRead 区域内重新允许的路径 |
| `allowManagedReadPathsOnly` | `boolean?` | 仅限托管设置的读取路径强制执行 |

#### `SandboxSettingsSchema`

| 字段 | 类型 | 描述 |
|---|---|---|
| `enabled` | `boolean?` | 启用沙盒化 |
| `failIfUnavailable` | `boolean?` | 如果沙盒不可用则硬失败 |
| `autoAllowBashIfSandboxed` | `boolean?` | 沙盒化时自动允许 Bash |
| `allowUnsandboxedCommands` | `boolean?` | 允许 `dangerouslyDisableSandbox` 参数 |
| `network` | `SandboxNetworkConfig?` | 网络配置 |
| `filesystem` | `SandboxFilesystemConfig?` | 文件系统配置 |
| `ignoreViolations` | `Record<string, string[]>?` | 每规则违规忽略 |
| `enableWeakerNestedSandbox` | `boolean?` | 较弱的嵌套沙盒 |
| `enableWeakerNetworkIsolation` | `boolean?` | macOS：允许 com.apple.trustd.agent |
| `excludedCommands` | `string[]?` | 从沙盒化中排除的命令 |
| `ripgrep` | `{ command: string; args?: string[] }?` | 自定义 ripgrep 配置 |

注意：模式使用 `.passthrough()` — 接受未记录的 `enabledPlatforms` 字段。

---

## entrypoints/sdk/coreSchemas.ts — SDK 核心 Zod 模式

### 用途

SDK 数据类型的单一事实来源。TypeScript 类型是从这些模式代码生成的。所有模式都使用 `lazySchema()` 包装器（延迟求值）。

### 模式组

#### 使用与模型

```typescript
export const ModelUsageSchema  // inputTokens, outputTokens, cacheRead/Write, webSearch, costUSD, contextWindow, maxOutputTokens
```

#### 输出格式

```typescript
export const OutputFormatTypeSchema   // z.literal('json_schema')
export const BaseOutputFormatSchema   // { type: OutputFormatTypeSchema }
export const JsonSchemaOutputFormatSchema  // { type: 'json_schema', schema: Record<string, unknown> }
export const OutputFormatSchema       // = JsonSchemaOutputFormatSchema
```

#### 配置

```typescript
export const ApiKeySourceSchema       // 'user' | 'project' | 'org' | 'temporary' | 'oauth'
export const ConfigScopeSchema        // 'local' | 'user' | 'project'
export const SdkBetaSchema            // z.literal('context-1m-2025-08-07')
```

#### 思考配置

```typescript
export const ThinkingAdaptiveSchema  // { type: 'adaptive' }  — Claude 决定（Opus 4.6+）
export const ThinkingEnabledSchema   // { type: 'enabled', budgetTokens?: number }  — 固定预算
export const ThinkingDisabledSchema  // { type: 'disabled' }
export const ThinkingConfigSchema    // 上述三个的联合
```

#### MCP 服务器配置

```typescript
export const McpStdioServerConfigSchema   // { type?: 'stdio', command, args?, env? }
export const McpSSEServerConfigSchema     // { type: 'sse', url, headers? }
export const McpHttpServerConfigSchema    // { type: 'http', url, headers? }
export const McpSdkServerConfigSchema     // { type: 'sdk', name }
export const McpServerConfigForProcessTransportSchema  // stdio|sse|http|sdk 的联合
export const McpClaudeAIProxyServerConfigSchema  // { type: 'claudeai-proxy', url, id }
export const McpServerStatusConfigSchema  // 进程传输 + claudeai-proxy 的联合
export const McpSetServersResultSchema    // { added, removed, errors }
```

#### MCP 服务器状态

```typescript
export const McpServerStatusSchema  // { name, status, serverInfo?, error?, config?, scope?, tools?, capabilities? }
// 状态枚举：'connected' | 'failed' | 'needs-auth' | 'pending' | 'disabled'
```

#### 权限类型

```typescript
export const PermissionUpdateDestinationSchema  // 'userSettings' | 'projectSettings' | 'localSettings' | 'session' | 'cliArg'
export const PermissionBehaviorSchema          // 'allow' | 'deny' | 'ask'
export const PermissionRuleValueSchema         // { toolName, ruleContent? }
export const PermissionUpdateSchema            // 可区分联合：addRules | replaceRules | removeRules | setMode | addDirectories | removeDirectories
export const PermissionDecisionClassificationSchema  // 'user_temporary' | 'user_permanent' | 'user_reject'
export const PermissionResultSchema            // { behavior: 'allow', updatedInput?, ... } | { behavior: 'deny', message, ... }
export const PermissionModeSchema              // 'default' | 'acceptEdits' | 'bypassPermissions' | 'plan' | 'dontAsk'
```

#### 钩子类型

```typescript
export const HOOK_EVENTS  // 28 个事件名称的常量数组（与 coreTypes.ts 相同）
export const HookEventSchema  // z.enum(HOOK_EVENTS)
export const BaseHookInputSchema  // { session_id, transcript_path, cwd, permission_mode?, agent_id? }
export const HookInputSchema     // 完整钩子输入（扩展基类）
```

#### 消息类型

所有 `SDKMessage*` 模式定义了 `query()` 发出的完整消息分类法：

| 模式 | 描述 |
|---|---|
| `SDKMessageSchema` | 所有消息类型的根联合 |
| `SDKUserMessageSchema` | 用户轮次消息 |
| `SDKStreamlinedTextMessageSchema` | 优化的纯文本消息 |
| `SDKStreamlinedToolUseSummaryMessageSchema` | 工具使用摘要（优化） |
| `SDKPostTurnSummaryMessageSchema` | 轮次结束摘要 |

---

## entrypoints/sdk/coreTypes.ts — SDK 核心 TypeScript 类型

### 用途

SDK 的 TypeScript 类型声明，从 `coreSchemas.ts` 代码生成。不手动编辑。

### 导出

```typescript
// 重新导出沙盒类型
export type { SandboxFilesystemConfig, SandboxIgnoreViolations, SandboxNetworkConfig, SandboxSettings }
  from '../sandboxTypes.js'

// 所有生成的类型
export * from './coreTypes.generated.js'

// 无法用 Zod 模式表达的实用类型
export type { NonNullableUsage } from './sdkUtilityTypes.js'
```

### 常量数组（运行时）

```typescript
export const HOOK_EVENTS = [...] as const  // 28 个钩子事件名称
export const EXIT_REASONS = ['clear', 'resume', 'logout', 'prompt_input_exit', 'other', 'bypass_permissions_disabled'] as const
```

---

## entrypoints/sdk/controlSchemas.ts — SDK 控制协议模式

### 用途

SDK 控制协议的 Zod 模式 — SDK 实现（Python SDK、桌面应用）与 CLI 进程之间的双向通信通道。通过 stdin/stdout 使用 `SDKControlRequest` / `SDKControlResponse` 消息包装器。

### 控制请求模式

每个请求都有一个 `subtype` 区分器：

| 模式 | `subtype` | 描述 |
|---|---|---|
| `SDKControlInitializeRequestSchema` | `'initialize'` | 初始化 SDK 会话（钩子、MCP、智能体、jsonSchema） |
| `SDKControlInterruptRequestSchema` | `'interrupt'` | 中断当前轮次 |
| `SDKControlPermissionRequestSchema` | `'can_use_tool'` | 请求工具权限 |
| `SDKControlSetPermissionModeRequestSchema` | `'set_permission_mode'` | 更改权限模式 |
| `SDKControlSetModelRequestSchema` | `'set_model'` | 更改活动模型 |
| `SDKControlSetMaxThinkingTokensRequestSchema` | `'set_max_thinking_tokens'` | 设置思考预算 |
| `SDKControlMcpStatusRequestSchema` | `'mcp_status'` | 获取 MCP 服务器状态 |
| `SDKControlGetContextUsageRequestSchema` | `'get_context_usage'` | 获取上下文窗口细分 |
| `SDKControlRewindFilesRequestSchema` | `'rewind_files'` | 回滚自用户消息以来的文件更改 |
| `SDKControlCancelAsyncMessageRequestSchema` | `'cancel_async_message'` | 丢弃排队的异步消息 |
| `SDKControlSeedReadStateRequestSchema` | `'seed_read_state'` | 种子 readFileState 缓存 |
| `SDKHookCallbackRequestSchema` | `'hook_callback'` | 传递钩子回调 |
| `SDKControlMcpMessageRequestSchema` | `'mcp_message'` | 向 MCP 服务器发送 JSON-RPC |
| `SDKControlMcpSetServersRequestSchema` | `'mcp_set_servers'` | 替换动态 MCP 服务器 |
| `SDKControlReloadPluginsRequestSchema` | `'reload_plugins'` | 从磁盘重新加载插件 |
| `SDKControlMcpReconnectRequestSchema` | `'mcp_reconnect'` | 重新连接失败的 MCP 服务器 |
| `SDKControlMcpToggleRequestSchema` | `'mcp_toggle'` | 启用/禁用 MCP 服务器 |
| `SDKControlStopTaskRequestSchema` | `'stop_task'` | 停止正在运行的任务 |
| `SDKControlApplyFlagSettingsRequestSchema` | `'apply_flag_settings'` | 合并标志设置层 |
| `SDKControlGetSettingsRequestSchema` | `'get_settings'` | 获取有效 + 每源设置 |
| `SDKControlElicitationRequestSchema` | `'elicitation'` | MCP 启发请求 |

### 控制响应模式

```typescript
export const SDKControlInitializeResponseSchema  // commands, agents, output_style, models, account, pid?, fast_mode_state?
export const SDKControlMcpStatusResponseSchema   // { mcpServers: McpServerStatus[] }
export const SDKControlGetContextUsageResponseSchema  // 详细的上下文细分
export const SDKControlRewindFilesResponseSchema      // { canRewind, error?, filesChanged?, insertions?, deletions? }
export const SDKControlCancelAsyncMessageResponseSchema  // { cancelled: boolean }
export const SDKControlMcpSetServersResponseSchema    // { added, removed, errors }
export const SDKControlReloadPluginsResponseSchema    // { commands, agents, plugins, mcpServers, error_count }
export const SDKControlGetSettingsResponseSchema      // { effective, sources, applied? }
export const SDKControlElicitationResponseSchema      // { action: 'accept'|'decline'|'cancel', content? }
```

### 有线消息包装器

```typescript
// 外部请求信封
export const SDKControlRequestSchema = z.object({
  type: z.literal('control_request'),
  request_id: z.string(),
  request: SDKControlRequestInnerSchema(),  // 所有请求类型的联合
})

// 响应信封
export const SDKControlResponseSchema = z.object({
  type: z.literal('control_response'),
  response: z.union([ControlResponseSchema(), ControlErrorResponseSchema()]),
})

// 取消（针对长时间运行的请求）
export const SDKControlCancelRequestSchema = z.object({
  type: z.literal('control_cancel_request'),
  request_id: z.string(),
})
```

### 聚合消息类型

```typescript
// CLI 写入 stdout 的消息
export const StdoutMessageSchema = z.union([
  SDKMessageSchema(),
  SDKStreamlinedTextMessageSchema(),
  SDKStreamlinedToolUseSummaryMessageSchema(),
  SDKPostTurnSummaryMessageSchema(),
  SDKControlResponseSchema(),
  SDKControlRequestSchema(),
  SDKControlCancelRequestSchema(),
  SDKKeepAliveMessageSchema(),
])

// CLI 从 stdin 读取的消息
export const StdinMessageSchema = z.union([
  SDKUserMessageSchema(),
  SDKControlRequestSchema(),
  SDKControlResponseSchema(),
  SDKKeepAliveMessageSchema(),
  SDKUpdateEnvironmentVariablesMessageSchema(),
])
```

### 钩子回调匹配器

```typescript
export const SDKHookCallbackMatcherSchema = z.object({
  matcher: z.string().optional(),
  hookCallbackIds: z.array(z.string()),
  timeout: z.number().optional(),
})
```

### 上下文使用响应详情

`SDKControlGetContextUsageResponseSchema` 响应包括：

| 字段 | 类型 |
|---|---|
| `categories` | `Array<{ name, tokens, color, isDeferred? }>` |
| `totalTokens` | `number` |
| `maxTokens` | `number` |
| `rawMaxTokens` | `number` |
| `percentage` | `number` |
| `gridRows` | `Array<Array<ContextGridSquare>>` |
| `model` | `string` |
| `memoryFiles` | `Array<{ path, type, tokens }>` |
| `mcpTools` | `Array<{ name, serverName, tokens, isLoaded? }>` |
| `deferredBuiltinTools` | `Array<{ name, tokens, isLoaded }>?` |
| `systemTools` | `Array<{ name, tokens }>?` |
| `systemPromptSections` | `Array<{ name, tokens }>?` |
| `agents` | `Array<{ agentType, source, tokens }>` |
| `slashCommands` | `{ totalCommands, includedCommands, tokens }?` |
| `skills` | `{ totalSkills, includedSkills, tokens, skillFrontmatter[] }?` |
| `autoCompactThreshold` | `number?` |
| `isAutoCompactEnabled` | `boolean` |
| `messageBreakdown` | 详细消息令牌细分 `?` |
| `apiUsage` | `{ input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens }` 或 `null` |

---

## query.ts — 核心异步查询循环

### 用途

核心智能查询循环。驱动用户提示、Claude API 和工具执行之间的来回交互。实现为 `AsyncGenerator`，产生 `StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage` 并返回 `Terminal` 值。

### 导出

```typescript
export type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number
  skipCacheWrite?: boolean
  taskBudget?: { total: number }
  deps?: QueryDeps
}

export async function* query(
  params: QueryParams,
): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal
>
```

### 内部状态类型

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined  // 前一次迭代继续的原因
}
```

### 常量

| 常量 | 值 | 描述 |
|---|---|---|
| `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` | `3` | max_output_tokens 错误的最大恢复重试次数 |

### 查询循环架构

```
query(params)
  └── queryLoop(params, consumedCommandUuids)
        ├── 快照配置 (buildQueryConfig())
        ├── 开始内存预取 (startRelevantMemoryPrefetch)
        └── while (true):
              1. yield { type: 'stream_request_start' }
              2. 构建 queryTracking (chainId/depth)
              3. 获取压缩边界后的消息
              4. 应用工具结果预算 (applyToolResultBudget)
              5. 如果需要则截断压缩 (HISTORY_SNIP 特性)
              6. 微压缩 (deps.microcompact)
              7. 上下文折叠 (CONTEXT_COLLAPSE 特性)
              8. 构建 fullSystemPrompt
              9. 自动压缩 (deps.autocompact) → 可能产生压缩边界消息
             10. 检查阻塞令牌限制（如果未压缩且未反应性压缩）
             11. 调用模型 (deps.callModel) → 流式助手消息
             12. 执行工具 (runTools 或 StreamingToolExecutor)
             13. 产生消息，工具结果
             14. 处理停止钩子 (handleStopHooks)
             15. 检查继续条件：
                  - 停止钩子阻塞 → 继续并带阻塞错误
                  - 超过 maxTurns → 返回 'max_turns'
                  - 无工具使用 → 检查 tokenBudget → 返回 'end_turn'
                  - 有工具使用 → 继续循环
```

### 恢复路径

查询循环包括几个错误恢复路径：

| 错误 | 恢复 |
|---|---|
| `max_output_tokens` | 重试最多 `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` 次，增加预算 |
| 提示过长 | 反应性压缩（REACTIVE_COMPACT 特性）或返回 `blocking_limit` |
| 流式回退 | 标记孤立消息，创建新的 `StreamingToolExecutor` |
| FallbackTriggeredError | 切换到回退模型，重试 |
| 上下文折叠溢出 | 通过 CONTEXT_COLLAPSE 特性排出暂存的折叠 |

### 特性标志

`HISTORY_SNIP`, `CONTEXT_COLLAPSE`, `REACTIVE_COMPACT`, `CACHED_MICROCOMPACT`, `TOKEN_BUDGET`, `BG_SESSIONS`

### 工具执行集成

- **流式工具执行**（门控于 `config.gates.streamingToolExecution`）：使用 `StreamingToolExecutor` 类
- **顺序工具执行**：使用 `services/tools/toolOrchestration.js` 中的 `runTools()`
- 工具结果作为 `UserMessage` 对象产生回循环中

### 关键行为

- **`persistReplacements`**：工具结果内容替换会为 `agent:*` 和 `repl_main_thread*` 查询源持久化
- **`backfillObservableInput`**：在产生之前向工具输入添加可观察字段（例如，扩展的文件路径）— 仅当添加新字段时，不覆盖
- **标记**：流式回退失败的孤立消息通过 `{ type: 'tombstone', message }` 事件标记
- **查询链跟踪**：每次迭代增加 `queryTracking.depth`；第一次迭代创建新的 `chainId` UUID

---

## QueryEngine.ts — 有状态查询引擎 (SDK/Headless)

### 用途

拥有对话的完整查询生命周期和会话状态。为 SDK/无头 (`-p`) 路径设计。每个会话一个实例；每次 `submitMessage()` 调用开始新轮次，同时保留所有状态（消息、文件缓存、使用情况、权限拒绝）。

### 导出

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  verbose?: boolean
  replayUserMessages?: boolean
  handleElicitation?: ToolUseContext['handleElicitation']
  includePartialMessages?: boolean
  setSDKStatus?: (status: SDKStatus) => void
  abortController?: AbortController
  orphanedPermission?: OrphanedPermission
  snipReplay?: (yieldedSystemMsg: Message, store: Message[]) => { messages: Message[]; executed: boolean } | undefined
}

export class QueryEngine {
  constructor(config: QueryEngineConfig)
  async *submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean },
  ): AsyncGenerator<SDKMessage, void, unknown>
  abort(): void
  getMessages(): Message[]
  getTotalUsage(): NonNullableUsage
  getPermissionDenials(): SDKPermissionDenial[]
}
```

### 内部状态

```typescript
private config: QueryEngineConfig
private mutableMessages: Message[]
private abortController: AbortController
private permissionDenials: SDKPermissionDenial[]
private totalUsage: NonNullableUsage
private hasHandledOrphanedPermission: boolean
private readFileState: FileStateCache
private discoveredSkillNames: Set<string>  // 每次 submitMessage() 清除
private loadedNestedMemoryPaths: Set<string>  // 跨轮次增长
```

### `submitMessage()` 流程

1. 清除 `discoveredSkillNames`
2. 通过 `setCwd(cwd)` 设置 CWD
3. 包装 `canUseTool` 以跟踪权限拒绝
4. 解析初始模型和思考配置
5. `fetchSystemPromptParts()` — 构建系统提示、用户上下文、系统上下文
6. 从 `customSystemPrompt | defaultSystemPrompt` + `memoryMechanicsPrompt?` + `appendSystemPrompt?` 构建 `systemPrompt`
7. 注册结构化输出强制执行（如果 `jsonSchema` + 合成输出工具）
8. 构建初始 `processUserInputContext`
9. 处理孤立权限（每个生命周期一次）
10. 为用户提示调用 `processUserInput()`
11. 将新消息推送到 `mutableMessages`
12. 持久化到转录本（在 API 调用之前 — 确保即使中途被杀死，`--resume` 也能工作）
13. 如果 `replayUserMessages: true` 则重放用户消息
14. 从 `processUserInput` 结果更新 `ToolPermissionContext.alwaysAllowRules.command`
15. 使用更新的消息和模型重新构建 `processUserInputContext`
16. 从 `query()` 生成器流式传输 — 转换为 `SDKMessage`，产生
17. 通过 `accumulateUsage()` / `updateUsage()` 跟踪使用情况
18. 处理本地命令输出、压缩边界、截断边界

### 转录本持久化

在 `submitMessage()` 中，用户消息在进入 API 查询循环之前写入转录本。时间变体：
- **`--bare` / `isBareMode()`**：触发即忘（在 SSD 上节省约 4ms）
- **`CLAUDE_CODE_EAGER_FLUSH` 或 `CLAUDE_CODE_IS_COWORK`**：等待 + 刷新
- **默认**：等待

### `ProcessUserInputContext` 内部

第一次构建（在斜杠命令处理之前）：
- `setMessages`：写回 `mutableMessages`
- `isNonInteractiveSession: true`

第二次构建（在斜杠命令处理之后）：
- `setMessages`：无操作（斜杠命令已提交）

---

## query/config.ts — 查询配置快照

### 用途

捕获查询入口处的不可变配置值。与每次迭代的状态分离，以使未来的 `step()` 提取（纯归约器模式）易于处理。故意排除 `feature()` 门（那些是构建时树摇边界）。

### 导出

```typescript
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean  // Statsig: 'tengu_streaming_tool_execution2'
    emitToolUseSummaries: boolean    // 环境变量: CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES
    isAnt: boolean                   // 环境变量: USER_TYPE === 'ant'
    fastModeEnabled: boolean         // 环境变量: !CLAUDE_CODE_DISABLE_FAST_MODE
  }
}

export function buildQueryConfig(): QueryConfig
```

### 门详情

| 门 | 源 | 键 |
|---|---|---|
| `streamingToolExecution` | Statsig（缓存，可能过时） | `tengu_streaming_tool_execution2` |
| `emitToolUseSummaries` | 环境变量 | `CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES` |
| `isAnt` | 环境变量 | `USER_TYPE === 'ant'` |
| `fastModeEnabled` | 环境变量 | `!CLAUDE_CODE_DISABLE_FAST_MODE` |

---

## query/deps.ts — 查询依赖注入

### 用途

`query()` 的 I/O 依赖项，通过 `QueryParams.deps` 传递。支持在不使用 `spyOn` 每模块样板的情况下注入测试假对象。

### 导出

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming       // API 流式调用
  microcompact: typeof microcompactMessages       // 微压缩
  autocompact: typeof autoCompactIfNeeded         // 自动压缩
  uuid: () => string                              // UUID 生成
}

export function productionDeps(): QueryDeps
```

### 生产依赖

| 依赖 | 实现 |
|---|---|
| `callModel` | `services/api/claude.js` 中的 `queryModelWithStreaming` |
| `microcompact` | `services/compact/microCompact.js` 中的 `microcompactMessages` |
| `autocompact` | `services/compact/autoCompact.js` 中的 `autoCompactIfNeeded` |
| `uuid` | Node.js `crypto` 模块中的 `randomUUID` |

---

## query/stopHooks.ts — 停止钩子编排

### 用途

编排所有轮次结束钩子：`Stop`、`SubagentStop`、`TeammateIdle`、`TaskCompleted`。还处理后台副作用（提示建议、内存提取、自动梦境、计算机使用清理）。

### 导出

```typescript
export async function* handleStopHooks(
  messagesForQuery: Message[],
  assistantMessages: AssistantMessage[],
  systemPrompt: SystemPrompt,
  userContext: { [k: string]: string },
  systemContext: { [k: string]: string },
  toolUseContext: ToolUseContext,
  querySource: QuerySource,
  stopHookActive?: boolean,
): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  StopHookResult
>

type StopHookResult = {
  blockingErrors: Message[]
  preventContinuation: boolean
}
```

### 执行顺序

1. **`saveCacheSafeParams()`** — 为提示建议 / btw 查询快照上下文（仅限主线程和 SDK）
2. **模板作业分类**（TEMPLATES 特性，仅限主线程，非子智能体）：`classifyAndWriteState()` — 最大 60 秒超时
3. **后台副作用**（非 bare 模式）：
   - `executePromptSuggestion()`（触发即忘，除非 `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=false`）
   - `executeExtractMemories()`（EXTRACT_MEMORIES 特性，仅限主线程）
   - `executeAutoDream()`（非子智能体）
4. **计算机使用清理**（CHICAGO_MCP 特性，仅限主线程）：`cleanupComputerUseAfterTurn()`
5. **`executeStopHooks()`** — 并行运行 Stop/SubagentStop 钩子，产生进度消息
6. **摘要消息**（如果任何钩子运行）
7. **通知**（如果发生钩子错误）
8. **队友钩子**（如果 `isTeammate()`）：
   - `executeTaskCompletedHooks()` 用于此智能体拥有的进行中任务
   - `executeTeammateIdleHooks()`

### 钩子结果类型

| 结果字段 | 效果 |
|---|---|
| `blockingError` | 创建 `UserMessage` 并设置 `isMeta: true`，添加到 `blockingErrors` |
| `preventContinuation` | 设置标志，产生 `hook_stopped_continuation` 附件 |
| 中止信号 | 产生 `UserInterruptionMessage`，返回 `{ blockingErrors: [], preventContinuation: true }` |

### 遥测事件

- `tengu_pre_stop_hooks_cancelled` — 钩子中途中止
- `tengu_stop_hook_error` — 钩子执行异常

---

## query/tokenBudget.ts — 令牌预算跟踪

### 用途

跨查询循环迭代跟踪令牌预算使用情况，以决定是否基于 `+500k auto-continue` 特性（与 API `task_budget` 不同）继续或停止。

### 导出

```typescript
export type BudgetTracker = {
  continuationCount: number
  lastDeltaTokens: number
  lastGlobalTurnTokens: number
  startedAt: number
}

export function createBudgetTracker(): BudgetTracker

export type TokenBudgetDecision =
  | { action: 'continue'; nudgeMessage: string; continuationCount: number; pct: number; turnTokens: number; budget: number }
  | { action: 'stop'; completionEvent: { continuationCount: number; pct: number; turnTokens: number; budget: number; diminishingReturns: boolean; durationMs: number } | null }

export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision
```

### 常量

| 常量 | 值 | 描述 |
|---|---|---|
| `COMPLETION_THRESHOLD` | `0.9` | 预算消耗 90% → 停止 |
| `DIMINISHING_THRESHOLD` | `500` | 两次检查间令牌增量 < 500 → 收益递减 |

### 算法

```
checkTokenBudget(tracker, agentId, budget, globalTurnTokens):
  if agentId or budget null or budget <= 0:
    return { action: 'stop', completionEvent: null }

  pct = round(turnTokens / budget * 100)
  deltaSinceLast = globalTurnTokens - tracker.lastGlobalTurnTokens

  isDiminishing = (
    continuationCount >= 3 AND
    deltaSinceLast < 500 AND
    lastDeltaTokens < 500
  )

  if !isDiminishing AND turnTokens < budget * 0.9:
    → return { action: 'continue', nudgeMessage: ... }

  if isDiminishing OR continuationCount > 0:
    → return { action: 'stop', completionEvent: { ..., diminishingReturns: isDiminishing } }

  → return { action: 'stop', completionEvent: null }
```

---

## context.ts — 系统与用户上下文提供者

### 用途

提供记忆化的上下文函数，构建每个对话前预置的 `systemContext` 和 `userContext` 字典。两者都缓存到对话结束。包括 git 状态、CLAUDE.md 内容和当前日期。

### 导出

```typescript
export function getSystemPromptInjection(): string | null
export function setSystemPromptInjection(value: string | null): void

export const getGitStatus: () => Promise<string | null>  // 记忆化
export const getSystemContext: () => Promise<{ [k: string]: string }>  // 记忆化
export const getUserContext: () => Promise<{ [k: string]: string }>    // 记忆化
```

### 常量

| 常量 | 值 | 描述 |
|---|---|---|
| `MAX_STATUS_CHARS` | `2000` | `git status --short` 输出的最大字符数 |

### `getGitStatus()` — 记忆化

在测试环境中返回 `null`。否则：
1. 检查 `getIsGit()` — 如果不是 git 仓库则返回 `null`
2. 并行运行：`getBranch()`、`getDefaultBranch()`、`git status --short`、`git log --oneline -n 5`、`git config user.name`
3. 在 `MAX_STATUS_CHARS` 处截断状态（附加截断通知）
4. 返回格式化字符串，包含分支、主分支、git 用户、状态和最近提交

**格式**：
```
This is the git status at the start of the conversation. Note that this status is a snapshot in time, and will not update during the conversation.

Current branch: <branch>

Main branch (you will usually use this for PRs): <mainBranch>

Git user: <userName>

Status:
<status or "(clean)">

Recent commits:
<log>
```

### `getSystemContext()` — 记忆化

```typescript
{
  gitStatus?: string,    // 来自 getGitStatus() — 对于 CCR 或 git 禁用时跳过
  cacheBreaker?: string, // "[CACHE_BREAKER: injection]" — 仅限 BREAK_CACHE_COMMAND 特性
}
```

在以下情况下跳过 git 状态：
- `CLAUDE_CODE_REMOTE=true`（CCR 环境）
- `shouldIncludeGitInstructions()` 返回 false

### `getUserContext()` — 记忆化

```typescript
{
  claudeMd?: string,    // 组合的 CLAUDE.md 内容
  currentDate: string,  // "Today's date is YYYY-MM-DD."
}
```

在以下情况下禁用 CLAUDE.md 加载：
- `CLAUDE_CODE_DISABLE_CLAUDE_MDS=true`
- `isBareMode()` 且没有 `--add-dir` 目录

副作用：调用 `setCachedClaudeMdContent()` 以缓存内容供自动模式分类器使用。

### `setSystemPromptInjection()` 副作用

当注入更改时，清除 `getUserContext.cache` 和 `getSystemContext.cache`（来自 lodash 记忆化）以强制重建。

---

## history.ts — 提示历史管理

### 用途

管理持久的提示历史（上箭头导航和 Ctrl+R 模糊搜索）。历史作为 JSONL 存储在 `~/.claude/history.jsonl` 中。处理粘贴内容，包括内联（≤ 1024 字节）和基于哈希的外部存储。条目按项目根目录划分范围。

### 导出

```typescript
// 粘贴内容助手
export function getPastedTextRefNumLines(text: string): number
export function formatPastedTextRef(id: number, numLines: number): string
export function formatImageRef(id: number): string
export function parseReferences(input: string): Array<{ id: number; match: string; index: number }>
export function expandPastedTextRefs(input: string, pastedContents: Record<number, PastedContent>): string

// 历史读取
export async function* makeHistoryReader(): AsyncGenerator<HistoryEntry>
export async function* getTimestampedHistory(): AsyncGenerator<TimestampedHistoryEntry>
export async function* getHistory(): AsyncGenerator<HistoryEntry>

// 历史写入
export function addToHistory(command: HistoryEntry | string): void
export function clearPendingHistoryEntries(): void
export function removeLastFromHistory(): void

// 类型
export type TimestampedHistoryEntry = {
  display: string
  timestamp: number
  resolve: () => Promise<HistoryEntry>
}
```

### 常量

| 常量 | 值 | 描述 |
|---|---|---|
| `MAX_HISTORY_ITEMS` | `100` | `getHistory()` 返回的最大条目数 |
| `MAX_PASTED_CONTENT_LENGTH` | `1024` | 内联与基于哈希存储的阈值 |

### 内部类型

```typescript
type LogEntry = {
  display: string
  pastedContents: Record<number, StoredPastedContent>
  timestamp: number
  project: string
  sessionId?: string
}

type StoredPastedContent = {
  id: number
  type: 'text' | 'image'
  content?: string          // 内联（≤ MAX_PASTED_CONTENT_LENGTH）
  contentHash?: string      // 大粘贴的哈希引用
  mediaType?: string
  filename?: string
}
```

### 历史文件

**路径**：`join(getClaudeConfigHomeDir(), 'history.jsonl')`

**锁定**：使用基于文件的锁文件，参数：
- `stale: 10000` 毫秒
- 重试：3 次，最小超时 50 毫秒

### `addToHistory()` 行为

1. 如果 `CLAUDE_CODE_SKIP_PROMPT_HISTORY=true`（tmux 子进程会话）则跳过
2. 在首次调用时注册清理钩子（在进程退出时刷新待处理条目）
3. 异步调用 `addToPromptHistory()`（触发即忘）

### `getHistory()` 排序

当前会话条目在前（最新优先），然后其他会话条目（也最新优先）。相同的 `MAX_HISTORY_ITEMS` 窗口。这防止并发会话交错上箭头历史。

### `removeLastFromHistory()` — 撤销最后条目

快速路径：如果条目仍在 `pendingEntries` 中，则拼接移除。
慢速路径：如果已刷新，则添加时间戳到 `skippedTimestamps` 集（读取时参考）。
一次性：使用后清除 `lastAddedEntry`。

### 粘贴引用模式

引用格式：`\[(Pasted text|Image|\.\.\.Truncated text) #(\d+)(?: \+\d+ lines)?(\.)*\]`

示例：
- `[Pasted text #1]` — 零行粘贴
- `[Pasted text #1 +10 lines]` — 多行粘贴（计数 `\n` 出现次数）
- `[Image #2]` — 图像粘贴

---

## cost-tracker.ts — 会话成本跟踪

### 用途

管理会话成本/使用情况跟踪、持久化到项目配置以及格式化显示。将所有状态委托给 `bootstrap/state.ts`。

### 导出

```typescript
// 从 bootstrap/state.ts 重新导出
export { getTotalCostUSD as getTotalCost }
export { getTotalDuration }
export { getTotalAPIDuration }
export { getTotalAPIDurationWithoutRetries }
export { addToTotalLinesChanged }
export { getTotalLinesAdded, getTotalLinesRemoved }
export { getTotalInputTokens, getTotalOutputTokens }
export { getTotalCacheReadInputTokens, getTotalCacheCreationInputTokens }
export { getTotalWebSearchRequests }
export { hasUnknownModelCost }
export { resetStateForTests, resetCostState }
export { setHasUnknownModelCost }
export { getModelUsage, getUsageForModel }
export { formatCost }  // 内部函数，也导出

// 新函数
export function getStoredSessionCosts(sessionId: string): StoredCostState | undefined
export function restoreCostStateForSession(sessionId: string): boolean
export function saveCurrentSessionCosts(fpsMetrics?: FpsMetrics): void
export function addToTotalSessionCost(cost: number, usage: Usage, model: string): number
export function formatTotalCost(): string
```

### 内部类型

```typescript
type StoredCostState = {
  totalCostUSD: number
  totalAPIDuration: number
  totalAPIDurationWithoutRetries: number
  totalToolDuration: number
  totalLinesAdded: number
  totalLinesRemoved: number
  lastDuration: number | undefined
  modelUsage: { [modelName: string]: ModelUsage } | undefined
}
```

### `addToTotalSessionCost()` 详情

1. 调用 `addToTotalModelUsage()` — 聚合每模型令牌计数
2. 调用 bootstrap/state.ts 中的 `addToTotalCostState()`
3. 记录到 OpenTelemetry 计数器（`getCostCounter()`、`getTokenCounter()`）
4. 处理来自 `getAdvisorUsage(usage)` 的顾问使用情况 — 为每个顾问模型递归调用自身
5. 返回总成本（包括顾问成本）

### `saveCurrentSessionCosts()` — 持久化字段

写入项目配置：
- `lastCost`、`lastAPIDuration`、`lastAPIDurationWithoutRetries`、`lastToolDuration`、`lastDuration`
- `lastLinesAdded`、`lastLinesRemoved`
- `lastTotalInputTokens`、`lastTotalOutputTokens`
- `lastTotalCacheCreationInputTokens`、`lastTotalCacheReadInputTokens`
- `lastTotalWebSearchRequests`
- `lastFpsAverage`、`lastFpsLow1Pct`（来自 `fpsMetrics`）
- `lastModelUsage`（每模型：输入/输出/缓存令牌、网络搜索、成本）
- `lastSessionId`

### `formatTotalCost()` — 显示格式

```
Total cost:            $X.XXXX
Total duration (API):  Xs
Total duration (wall): Xs
Total code changes:    N lines added, N lines removed
Usage by model:
   claude-sonnet-4-6:  N input, N output, N cache read, N cache write ($X.XXXX)
```

### `formatCost()` 助手

```typescript
function formatCost(cost: number, maxDecimalPlaces: number = 4): string
// 如果 cost > 0.5 返回 "$X.XX"，否则返回 "$X.XXXX"（或更少小数位）
```

---

## costHook.ts — React 成本摘要钩子

### 用途

React 钩子，注册 `process.exit` 监听器，在进程退出时打印成本摘要并保存会话成本。

### 导出

```typescript
export function useCostSummary(getFpsMetrics?: () => FpsMetrics | undefined): void
```

### 行为

- 在挂载时运行一次（空依赖数组）
- 在 `process.exit` 时：
  1. 如果 `hasConsoleBillingAccess()`：将 `formatTotalCost()` 写入 stdout
  2. 调用 `saveCurrentSessionCosts(getFpsMetrics?.())`
- 在卸载时清理退出监听器

---

## projectOnboardingState.ts — 项目引导状态

### 用途

跟踪和控制项目引导清单的显示（当用户首次打开新项目时显示）。

### 导出

```typescript
export type Step = {
  key: string
  text: string
  isComplete: boolean
  isCompletable: boolean
  isEnabled: boolean
}

export function getSteps(): Step[]
export function isProjectOnboardingComplete(): boolean
export function maybeMarkProjectOnboardingComplete(): void
export const shouldShowProjectOnboarding: () => boolean  // 记忆化
export function incrementProjectOnboardingSeenCount(): void
```

### 步骤

| 键 | 启用条件 | 完成检查 |
|---|---|---|
| `'workspace'` | `isDirEmpty(getCwd())` | 从不自动完成（用户必须操作） |
| `'claudemd'` | `!isDirEmpty(getCwd())` | `existsSync(join(getCwd(), 'CLAUDE.md'))` |

### `shouldShowProjectOnboarding()` — 记忆化

如果以下任何一项为真，则返回 `false`：
- `projectConfig.hasCompletedProjectOnboarding === true`
- `projectConfig.projectOnboardingSeenCount >= 4`
- 设置了 `process.env.IS_DEMO`
- `isProjectOnboardingComplete()` 返回 true

### `maybeMarkProjectOnboardingComplete()` 行为

在缓存配置中 `hasCompletedProjectOnboarding: true` 时短路（避免每次提示提交时访问文件系统）。完成时保存到项目配置。

---

## bootstrap/state.ts — 全局会话状态

### 用途

Claude Code 进程的单一模块级状态单例。**不要在此处添加更多状态**（用三重注释强调记录）。包含所有会话范围的值，包括成本、令牌、模型配置、遥测、智能体状态和会话身份。

### 状态结构

`State` 类型是一个大型扁平对象。关键分组：

#### 会话身份

| 字段 | 类型 | 描述 |
|---|---|---|
| `sessionId` | `SessionId` | UUID（初始化时 randomUUID） |
| `parentSessionId` | `SessionId?` | 用于谱系跟踪的父会话 |
| `originalCwd` | `string` | 启动时的 CWD（NFC 标准化） |
| `projectRoot` | `string` | 稳定的项目根目录（由 --worktree 设置；会话中不更新） |
| `cwd` | `string` | 当前工作目录（由 setCwd 更新） |
| `sessionProjectDir` | `string | null` | 包含会话 JSONL 的目录；null = 从 originalCwd 派生 |

#### 成本和令牌计数器

| 字段 | 类型 |
|---|---|
| `totalCostUSD` | `number` |
| `totalAPIDuration` | `number` |
| `totalAPIDurationWithoutRetries` | `number` |
| `totalToolDuration` | `number` |
| `totalLinesAdded`、`totalLinesRemoved` | `number` |
| `modelUsage` | `{ [modelName: string]: ModelUsage }` |

#### 每轮次计数器（每轮次重置）

| 字段 | 类型 |
|---|---|
| `turnHookDurationMs` | `number` |
| `turnToolDurationMs` | `number` |
| `turnClassifierDurationMs` | `number` |
| `turnToolCount` | `number` |
| `turnHookCount` | `number` |
| `turnClassifierCount` | `number` |

#### 模型配置

| 字段 | 类型 | 描述 |
|---|---|---|
| `mainLoopModelOverride` | `ModelSetting?` | 由 --model 标志设置 |
| `initialMainLoopModel` | `ModelSetting` | 在启动时设置 |
| `modelStrings` | `ModelStrings | null` | 加载的模型字符串定义 |

#### 遥测 / OpenTelemetry

| 字段 | 类型 |
|---|---|
| `meter` | `Meter | null` |
| `meterProvider` | `MeterProvider | null` |
| `loggerProvider` | `LoggerProvider | null` |
| `tracerProvider` | `BasicTracerProvider | null` |
| `eventLogger` | `ReturnType<typeof logs.getLogger> | null` |
| `sessionCounter` | `AttributedCounter | null` |
| `locCounter`、`prCounter`、`commitCounter` | `AttributedCounter | null` |
| `costCounter`、`tokenCounter` | `AttributedCounter | null` |
| `codeEditToolDecisionCounter`、`activeTimeCounter` | `AttributedCounter | null` |
| `statsStore` | `{ observe(name, value): void } | null` |

#### 会话标志

| 字段 | 默认值 | 描述 |
|---|---|---|
| `isInteractive` | `false` | 交互式 REPL 模式 |
| `kairosActive` | `false` | 助手（KAIROS）模式激活 |
| `strictToolResultPairing` | `false` | HFI 模式 — 不匹配时抛出 |
| `sessionBypassPermissionsMode` | `false` | 不持久化 |
| `sessionPersistenceDisabled` | `false` | 禁用转录本写入 |
| `sessionTrustAccepted` | `false` | 仅会话信任（主目录） |
| `hasExitedPlanMode` | `false` | 用于重新进入指导 |
| `scheduledTasksEnabled` | `false` | 由 cron 调度器设置 |
| `isRemoteMode` | `false` | --remote 标志 |

#### 提示缓存闩锁

所有初始为 `null`（尚未触发），一旦翻转为 `true` 并保持 `true`：

| 字段 | 用途 |
|---|---|
| `afkModeHeaderLatched` | 粘性 AFK_MODE_BETA_HEADER |
| `fastModeHeaderLatched` | 粘性 FAST_MODE_BETA_HEADER |
| `cacheEditingHeaderLatched` | 粘性缓存编辑 beta 头 |
| `thinkingClearLatched` | 空闲超过 1 小时后清除思考 |

### 导出函数（选录）

#### 会话身份

```typescript
export function getSessionId(): SessionId
export function regenerateSessionId(options?: { setCurrentAsParent?: boolean }): SessionId
export function getParentSessionId(): SessionId | undefined
export function switchSession(sessionId: SessionId, projectDir?: string | null): void
export function getSessionProjectDir(): string | null
export const onSessionSwitch: (listener: (id: SessionId) => void) => () => void
export function getOriginalCwd(): string
export function setOriginalCwd(cwd: string): void
export function getProjectRoot(): string
export function setProjectRoot(cwd: string): void
export function getCwdState(): string
export function setCwdState(cwd: string): void
```

#### 成本 / 持续时间跟踪

```typescript
export function addToTotalDurationState(duration: number, durationWithoutRetries: number): void
export function addToTotalCostState(cost: number, modelUsage: ModelUsage, model: string): void
export function getTotalCostUSD(): number
export function getTotalAPIDuration(): number
export function getTotalDuration(): number
export function getTotalAPIDurationWithoutRetries(): number
export function getTotalToolDuration(): number
export function addToToolDuration(duration: number): void
export function getTurnHookDurationMs(): number
export function addToTurnHookDuration(duration: number): void
export function resetTurnHookDuration(): void
export function getTurnHookCount(): number
export function getTurnToolDurationMs(): number
export function resetTurnToolDuration(): void
export function getTurnToolCount(): number
export function getTurnClassifierDurationMs(): number
export function addToTurnClassifierDuration(duration: number): void
export function resetTurnClassifierDuration(): void
export function getTurnClassifierCount(): number
```

#### 令牌会计

```typescript
export function getTotalInputTokens(): number   // sumBy modelUsage.inputTokens
export function getTotalOutputTokens(): number
export function getTotalCacheReadInputTokens(): number
export function getTotalCacheCreationInputTokens(): number
export function getTotalWebSearchRequests(): number
export function getModelUsage(): { [modelName: string]: ModelUsage }
export function getUsageForModel(model: string): ModelUsage | undefined
```

#### 轮次令牌预算

```typescript
export function getTurnOutputTokens(): number   // 当前轮次输出令牌
export function getCurrentTurnTokenBudget(): number | null
export function snapshotOutputTokensForTurn(budget: number | null): void
export function getBudgetContinuationCount(): number
export function incrementBudgetContinuationCount(): void
```

#### 压缩后跟踪

```typescript
export function markPostCompaction(): void       // 设置 pendingPostCompaction=true
export function consumePostCompaction(): boolean // 压缩后返回 true 一次，重置
```

#### 成本状态持久化

```typescript
export function resetCostState(): void
export function setCostStateForRestore({ totalCostUSD, totalAPIDuration, ... }): void
export function resetStateForTests(): void
```

#### 滚动排出

```typescript
export function markScrollActivity(): void
export function getIsScrollDraining(): boolean
export async function waitForScrollIdle(): Promise<void>
```

#### 模型

```typescript
export function getMainLoopModelOverride(): ModelSetting | undefined
export function getInitialMainLoopModel(): ModelSetting
export function setMainLoopModelOverride(model: ModelSetting | undefined): void
export function setInitialMainLoopModel(model: ModelSetting): void
export function getSdkBetas(): string[] | undefined
export function setSdkBetas(betas: string[] | undefined): void
```

#### 遥测设置器

```typescript
export function setMeter(meter: Meter, createAttributedCounter: ...): void
export function getSessionCounter(): AttributedCounter | null
export function setStatsStore(store: ...): void
export function getStatsStore(): ...
export function updateLastInteractionTime(immediate?: boolean): void
export function flushInteractionTime(): void
```

#### 杂项会话状态

```typescript
export function setIsInteractive(v: boolean): void
export function getIsNonInteractiveSession(): boolean
export function setKairosActive(v: boolean): void
export function isSessionPersistenceDisabled(): boolean
export function setSessionPersistenceDisabled(v: boolean): void
export function setMainThreadAgentType(type: string | undefined): void
export function setIsRemoteMode(v: boolean): void
export function setClientType(type: string): void
export function setSessionSource(source: string | undefined): void
export function setInlinePlugins(dirs: string[]): void
export function getAdditionalDirectoriesForClaudeMd(): string[]
export function setAdditionalDirectoriesForClaudeMd(dirs: string[]): void
export function setAllowedChannels(channels: ChannelEntry[]): void
export function setAllowedSettingSources(sources: SettingSource[]): void
export function setSdkBetas(betas: string[] | undefined): void
export function setCachedClaudeMdContent(content: string | null): void
export function getLastMainRequestId(): string | undefined
export function setLastMainRequestId(requestId: string): void
export function setTeleportedSessionInfo(info: ...): void
```

### 重要设计说明

- **单例**：`STATE` 是模块级常量，通过 `getInitialState()` 初始化一次
- **`projectRoot` 与 `originalCwd`**：`projectRoot` 在启动时设置（包括由 `--worktree`）并且 **从不** 由 `EnterWorktreeTool` 更新。`originalCwd` 用于文件操作；`projectRoot` 用于会话身份（历史、技能）
- **`sessionProjectDir`**：在 `switchSession()` 和 `regenerateSessionId()` 时始终重置。`null` 表示“从 `originalCwd` 派生”
- **滚动排出**：模块级（不在 `STATE` 中）— 短暂的热路径标志，150 毫秒去抖
- **引导隔离**：此模块必须保持为导入 DAG 中的叶节点（不能直接从 `src/utils/` 导入 — 使用路径别名）

### `AttributedCounter` 类型

```typescript
export type AttributedCounter = {
  add(value: number, additionalAttributes?: Attributes): void
}
```

### `ChannelEntry` 类型

```typescript
export type ChannelEntry =
  | { kind: 'plugin'; name: string; marketplace: string; dev?: boolean }
  | { kind: 'server'; name: string; dev?: boolean }
```

---

## assistant/sessionHistory.ts — 远程会话历史分页

### 用途

从 Claude API 获取远程（CCR/BYOC）会话的对话事件历史。由助手/传送功能用于在恢复远程会话时重放对话历史。

### 导出

```typescript
export const HISTORY_PAGE_SIZE = 100

export type HistoryPage = {
  events: SDKMessage[]      // 页面内按时间顺序
  firstId: string | null    // 最旧事件 ID → 下一个更旧页面的 before_id 游标
  hasMore: boolean          // true = 存在更旧事件
}

export type HistoryAuthCtx = {
  baseUrl: string
  headers: Record<string, string>
}

export async function createHistoryAuthCtx(sessionId: string): Promise<HistoryAuthCtx>
export async function fetchLatestEvents(ctx: HistoryAuthCtx, limit?: number): Promise<HistoryPage | null>
export async function fetchOlderEvents(ctx: HistoryAuthCtx, beforeId: string, limit?: number): Promise<HistoryPage | null>
```

### API 端点

基础 URL：`${getOauthConfig().BASE_API_URL}/v1/sessions/${sessionId}/events`

| 函数 | 查询参数 | 描述 |
|---|---|---|
| `fetchLatestEvents` | `{ limit, anchor_to_latest: true }` | 最新的 `limit` 个事件，按时间顺序 |
| `fetchOlderEvents` | `{ limit, before_id: beforeId }` | 游标之前的事件 |

### 认证

需要 OAuth 访问令牌 + 组织 UUID。请求头：
- 标准 OAuth 请求头（`getOAuthHeaders(accessToken)`）
- `anthropic-beta: ccr-byoc-2025-07-29`
- `x-organization-uuid: orgUUID`

### 请求配置

- **超时**：每个请求 15,000 毫秒
- **状态验证**：`validateStatus: () => true`（手动状态检查）
- **错误处理**：网络错误或非 200 状态时返回 `null`；记录 HTTP 状态到调试

### 响应类型

```typescript
type SessionEventsResponse = {
  data: SDKMessage[]
  has_more: boolean
  first_id: string | null
  last_id: string | null
}
```

---

## 跨领域关注点

### 启动性能分析

贯穿这些文件，`profileCheckpoint(label)` 调用标记启动性能分析的时间里程碑。关键检查点：

| 检查点 | 位置 |
|---|---|
| `main_tsx_entry` | main.tsx 的第一行（在导入之前） |
| `main_tsx_imports_loaded` | 所有导入加载后 |
| `cli_entry` | cli.tsx 在 profile 导入后 |
| `cli_before_main_import` | 在 `import('../main.js')` 之前 |
| `cli_after_main_import` | main.ts 加载后 |
| `cli_after_main_complete` | `main()` 返回后 |
| `init_function_start` / `init_function_end` | init.ts 边界 |
| `main_function_start` | 进入 `main()` |
| `query_fn_entry` | 每次查询循环迭代 |
| `query_api_streaming_start` | 第一次 API 流式调用之前 |

### 特性标志系统

来自 `bun:bundle` 的特性标志（`feature('FLAG_NAME')`）是**构建时**死代码消除门，而不是运行时开关。bun 打包器在构建时评估这些，并从外部构建中移除不可达的代码分支。

运行时门使用：
- `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` — Statsig 实验门
- `isEnvTruthy(process.env.*)` — 环境变量门
- GrowthBook 特性值 — 用于更复杂的多变量实验

### 查询源值

`query()` 和 `handleStopHooks()` 中的 `querySource: QuerySource` 参数对每个查询的来源进行分类：

| 值 | 描述 |
|---|---|
| `'repl_main_thread'` | 交互式 REPL，主用户轮次 |
| `'sdk'` | SDK/无头路径 |
| `'agent:*'` | 子智能体（AgentTool） |
| `'compact'` | 压缩分叉 |
| `'session_memory'` | 会话内存分叉 |

以 `'agent:'` 或 `'repl_main_thread'` 开头的值启用内容替换的转录本持久化。

### 依赖注入模式

`QueryDeps`（在 `query/deps.ts` 中）是此代码库中 DI 的主要示例。测试可以直接将 `deps` 传递给 `QueryParams`，而不是在 6-8 个文件中使用 `jest.spyOn`。该模式有意狭窄（4 个依赖项），并注明可以扩展。