# Claude Code — 桥接协议、CLI 框架与远程系统

## 目录

1. [桥接系统总览](#1-桥接系统总览)
2. [桥接类型与核心数据结构](#2-桥接类型与核心数据结构)
3. [桥接 API 客户端](#3-桥接-api-客户端)
4. [桥接配置与认证](#4-桥接配置与认证)
5. [桥接权限与特性门控](#5-桥接权限与特性门控)
6. [会话生命周期：独立桥接（bridgeMain.ts）](#6-会话生命周期独立桥接bridgemaints)
7. [REPL Bridge（replBridge.ts / initReplBridge.ts）](#7-repl-bridgereplbridgets--initreplbridgets)
8. [无环境桥接核心（remoteBridgeCore.ts）](#8-无环境桥接核心remotebridgecorets)
9. [传输层](#9-传输层)
10. [消息协议（bridgeMessaging.ts）](#10-消息协议bridgemessagingts)
11. [JWT 认证（jwtUtils.ts）](#11-jwt-认证jwtutilsts)
12. [Session ID 兼容层（sessionIdCompat.ts）](#12-session-id-兼容层sessionidcompatts)
13. [Work Secret 与 CCR v2 注册（workSecret.ts）](#13-work-secret-与-ccr-v2-注册worksecretts)
14. [Bridge Pointer：崩溃恢复（bridgePointer.ts）](#14-bridge-pointer崩溃恢复bridgepointerts)
15. [权限回调（bridgePermissionCallbacks.ts）](#15-权限回调bridgepermissioncallbacksts)
16. [入站消息与附件](#16-入站消息与附件)
17. [会话运行器（sessionRunner.ts）](#17-会话运行器sessionrunnerts)
18. [桥接调试与故障注入（bridgeDebug.ts）](#18-桥接调试与故障注入bridgedebugets)
19. [桥接工具函数](#19-桥接工具函数)
20. [CLI 框架](#20-cli-框架)
21. [CLI 传输层](#21-cli-传输层)
22. [远程会话系统](#22-远程会话系统)
23. [replLauncher.tsx](#23-replaunchertsx)
24. [配置默认值与 GrowthBook 标志](#24-配置默认值与-growthbook-标志)
25. [完整 API 端点参考](#25-完整-api-端点参考)
26. [WebSocket 与 SSE 协议参考](#26-websocket-与-sse-协议参考)

---

## 1. 桥接系统总览

“Bridge”（远程控制）系统允许本地 Claude Code CLI 会话被 claude.ai 网页应用驱动。它在运行中的 CLI 进程和云端后端（CCR - Cloud Code Runner）之间建立双向通信通道。

### 架构：两种桥接变体

**V1 - 基于环境的桥接（env-based）**
- 使用 Environments API（`/v1/environments/bridge`）。
- Bridge 会把自己注册成一个 “environment”，然后轮询 “work”（session dispatch）。
- 会话传输层：WebSocket（v1）或 SSE + CCRClient（CCR v2），通过 `HybridTransport` 或 `SSETransport` 实现。
- `replBridge.ts` 里的 `initBridgeCore()` 处理 REPL 侧逻辑。
- `bridgeMain.ts` 里的 `runBridgeLoop()` 处理独立的 `claude remote-control` 逻辑。

**V2 - 无环境桥接（env-less）**
- 完全不经过 Environments API。
- 直接流程：POST `/v1/code/sessions` → POST `/v1/code/sessions/{id}/bridge` → `createV2ReplTransport()`。
- 只用于 REPL 会话；daemon / print 仍走 env-based。
- 由 `tengu_bridge_repl_v2` GrowthBook 标志控制。

### 两种部署模式

**REPL Bridge（始终开启 / `/remote-control`）**
- 由 `initReplBridge()` 初始化，通常从 `useReplBridge` hook 或 `print.ts` 调用。
- 运行在现有 REPL 进程里；来自 claude.ai 的消息会被注入为用户输入。
- 代码位于 `bridge/replBridge.ts`、`bridge/initReplBridge.ts`、`bridge/remoteBridgeCore.ts`。

**独立 Bridge（`claude remote-control`）**
- 每个 session 启动一个子 Claude 进程。
- 主循环在 `bridge/bridgeMain.ts` → `runBridgeLoop()`。
- 支持多 session 启动模式：`single-session`、`worktree`、`same-dir`。

---

## 2. 桥接类型与核心数据结构

**文件：** `bridge/types.ts`

### 常量

```typescript
DEFAULT_SESSION_TIMEOUT_MS = 24 * 60 * 60 * 1000   // 24 小时
BRIDGE_LOGIN_INSTRUCTION: string  // "Remote Control is only available with claude.ai subscriptions..."
BRIDGE_LOGIN_ERROR: string        // 未登录时打印的完整错误
REMOTE_CONTROL_DISCONNECTED_MSG = 'Remote Control disconnected.'
```

### WorkData

```typescript
type WorkData = {
  type: 'session' | 'healthcheck'
  id: string  // session ID
}
```

### WorkResponse

轮询 work 时的响应（`GET .../work/poll`）：

```typescript
type WorkResponse = {
  id: string               // work item ID
  type: 'work'
  environment_id: string
  state: string
  data: WorkData
  secret: string           // base64url 编码的 JSON WorkSecret
  created_at: string
}
```

### WorkSecret

从 `WorkResponse.secret` 解码得到（base64url JSON）：

```typescript
type WorkSecret = {
  version: number                          // 必须是 1
  session_ingress_token: string            // 给 session-ingress API 调用用的 JWT
  api_base_url: string
  sources: Array<{
    type: string
    git_info?: { type: string; repo: string; ref?: string; token?: string }
  }>
  auth: Array<{ type: string; token: string }>
  claude_code_args?: Record<string, string> | null
  mcp_config?: unknown | null
  environment_variables?: Record<string, string> | null
  use_code_sessions?: boolean              // 服务端驱动的 CCR v2 选择器
}
```

### BridgeConfig

传给主循环和 API client 的配置对象：

```typescript
type BridgeConfig = {
  dir: string                    // 工作目录
  machineName: string            // 主机名
  branch: string                 // Git 分支
  gitRepoUrl: string | null
  maxSessions: number            // 多 session 模式容量
  spawnMode: SpawnMode           // 'single-session' | 'worktree' | 'same-dir'
  verbose: boolean
  sandbox: boolean
  bridgeId: string               // 客户端生成的 UUID，标识这个 bridge 实例
  workerType: string             // 作为 metadata.worker_type 发送（例如 'claude_code'）
  environmentId: string          // 客户端生成的 UUID，用于幂等注册
  reuseEnvironmentId?: string    // 后端返回的可复用 ID
  apiBaseUrl: string
  sessionIngressUrl: string      // 本地开发时可能和 apiBaseUrl 不同
  debugFile?: string
  sessionTimeoutMs?: number
}
```

### SpawnMode

```typescript
type SpawnMode = 'single-session' | 'worktree' | 'same-dir'
```

- `single-session`：单个 session，结束后 bridge 退出。
- `worktree`：常驻 server，每个 session 都有独立 git worktree。
- `same-dir`：常驻 server，所有 session 共用 cwd（可能冲突）。

### BridgeWorkerType

```typescript
type BridgeWorkerType = 'claude_code' | 'claude_code_assistant'
```

### SessionActivity

```typescript
type SessionActivityType = 'tool_start' | 'text' | 'result' | 'error'

type SessionActivity = {
  type: SessionActivityType
  summary: string   // 例如 "Editing src/foo.ts"、"Reading package.json"
  timestamp: number
}
```

### SessionHandle

`SessionSpawner.spawn()` 返回的接口：

```typescript
type SessionHandle = {
  sessionId: string
  done: Promise<SessionDoneStatus>       // 'completed' | 'failed' | 'interrupted'
  kill(): void
  forceKill(): void
  activities: SessionActivity[]          // 最近约 10 条活动的 ring buffer
  currentActivity: SessionActivity | null
  accessToken: string                    // session_ingress_token
  lastStderr: string[]                   // 最近 stderr 行的 ring buffer
  writeStdin(data: string): void
  updateAccessToken(token: string): void
}
```

### SessionSpawnOpts

```typescript
type SessionSpawnOpts = {
  sessionId: string
  sdkUrl: string
  accessToken: string
  useCcrV2?: boolean            // 启动子进程时使用 CCR v2 环境变量
  workerEpoch?: number          // useCcrV2=true 时必需
  onFirstUserMessage?: (text: string) => void
}
```

### BridgeApiClient 接口

```typescript
type BridgeApiClient = {
  registerBridgeEnvironment(config: BridgeConfig): Promise<{
    environment_id: string
    environment_secret: string
  }>
  pollForWork(
    environmentId: string,
    environmentSecret: string,
    signal?: AbortSignal,
    reclaimOlderThanMs?: number,
  ): Promise<WorkResponse | null>
  acknowledgeWork(environmentId, workId, sessionToken): Promise<void>
  stopWork(environmentId, workId, force): Promise<void>
  deregisterEnvironment(environmentId): Promise<void>
  sendPermissionResponseEvent(sessionId, event, sessionToken): Promise<void>
  archiveSession(sessionId): Promise<void>
  reconnectSession(environmentId, sessionId): Promise<void>
  heartbeatWork(environmentId, workId, sessionToken): Promise<{
    lease_extended: boolean
    state: string
  }>
}
```

### PermissionResponseEvent

```typescript
type PermissionResponseEvent = {
  type: 'control_response'
  response: {
    subtype: 'success'
    request_id: string
    response: Record<string, unknown>
  }
}
```

### BridgeLogger 接口

桥接 UI / 日志系统的完整接口（定义在 `types.ts`）：

```typescript
type BridgeLogger = {
  printBanner(config, environmentId): void
  logSessionStart(sessionId, prompt): void
  logSessionComplete(sessionId, durationMs): void
  logSessionFailed(sessionId, error): void
  logStatus(message): void
  logVerbose(message): void
  logError(message): void
  logReconnected(disconnectedMs): void
  updateIdleStatus(): void
  updateReconnectingStatus(delayStr, elapsedStr): void
  updateSessionStatus(sessionId, elapsed, activity, trail): void
  clearStatus(): void
  setRepoInfo(repoName, branch): void
  setDebugLogPath(path): void
  setAttached(sessionId): void
  updateFailedStatus(error): void
  toggleQr(): void
  updateSessionCount(active, max, mode): void
  setSpawnModeDisplay(mode): void
  addSession(sessionId, url): void
  updateSessionActivity(sessionId, activity): void
  setSessionTitle(sessionId, title): void
  removeSession(sessionId): void
  refreshDisplay(): void
}
```

---

## 3. 桥接 API 客户端

**文件：** `bridge/bridgeApi.ts`

### 导出

#### `validateBridgeId(id: string, label: string): string`

验证服务器提供的 ID 是否能安全地插入 URL path。使用模式 `/^[a-zA-Z0-9_-]+$/`。如果包含不安全字符就抛错，防止 path traversal 攻击。

#### `class BridgeFatalError extends Error`

不可重试的桥接错误。携带：
- `status: number` - HTTP 状态码
- `errorType: string | undefined` - 服务器给出的错误类型（例如 `"environment_expired"`）

#### `createBridgeApiClient(deps: BridgeApiDeps): BridgeApiClient`

HTTP client 工厂。依赖项：

```typescript
type BridgeApiDeps = {
  baseUrl: string
  getAccessToken: () => string | undefined
  runnerVersion: string
  onDebug?: (msg: string) => void
  onAuth401?: (staleAccessToken: string) => Promise<boolean>
  getTrustedDeviceToken?: () => string | undefined
}
```

**请求头**（所有 API 调用都带）：
```text
Authorization: Bearer <token>
Content-Type: application/json
anthropic-version: 2023-06-01
anthropic-beta: environments-2025-11-01
x-environment-runner-version: <runnerVersion>
X-Trusted-Device-Token: <token>  (optional, when tengu_sessions_elevated_auth_enforcement)
```

**OAuth 401 重试：** 遇到 401 时，会调用 `onAuth401(staleToken)`。如果 token 刷新成功，请求会重试一次；如果重试后还是 401，就抛 `BridgeFatalError`。

**Poll endpoint**（`pollForWork`）：用 `environmentSecret`（不是 OAuth token）作为 Bearer auth。空轮询第 1 次会记录日志，然后每连续 100 次空轮询再记一次。

#### `isExpiredErrorType(errorType: string | undefined): boolean`

如果错误类型字符串包含 `'expired'` 或 `'lifetime'`，则返回 true。

#### `isSuppressible403(err: BridgeFatalError): boolean`

如果 403 和 `external_poll_sessions` 或 `environments:manage` scope 有关，则返回 true。这类权限错误对应的是非关键操作，不应该暴露给用户。

### 错误状态处理

| HTTP 状态 | 行为 |
|-------------|----------|
| 200, 204 | 成功 |
| 401 | 带登录提示的 `BridgeFatalError` |
| 403（expired errorType） | `BridgeFatalError`：“session has expired” |
| 403（其他） | `BridgeFatalError`：访问被拒绝 / 组织权限不足 |
| 404 | `BridgeFatalError`：未找到 |
| 410 | `BridgeFatalError`，`errorType='environment_expired'` |
| 429 | 普通 `Error`：rate limited |
| 其他 | 带状态码的普通 `Error` |

---

## 4. 桥接配置与认证

**文件：** `bridge/bridgeConfig.ts`

这里负责把 auth / URL 解析统一起来。分两层：dev 覆盖（仅 Ant）和生产 OAuth。

### 导出

#### `getBridgeTokenOverride(): string | undefined`

如果 `process.env.USER_TYPE === 'ant'`，返回 `process.env.CLAUDE_BRIDGE_OAUTH_TOKEN`；否则返回 `undefined`。

#### `getBridgeBaseUrlOverride(): string | undefined`

如果 `process.env.USER_TYPE === 'ant'`，返回 `process.env.CLAUDE_BRIDGE_BASE_URL`；否则返回 `undefined`。

#### `getBridgeAccessToken(): string | undefined`

先看 dev override，然后再看 `getClaudeAIOAuthTokens()?.accessToken`。

#### `getBridgeBaseUrl(): string`

先看 dev override，然后再看 `getOauthConfig().BASE_API_URL`。

---

## 5. 桥接权限与特性门控

**文件：** `bridge/bridgeEnabled.ts`

### 导出

#### `isBridgeEnabled(): boolean`

同步判断。要求：
1. build flag `feature('BRIDGE_MODE')` 必须为 true
2. `isClaudeAISubscriber()` - 排除 Bedrock / Vertex / API key 用户
3. GrowthBook flag `tengu_ccr_bridge`（有缓存，可能过期）

#### `isBridgeEnabledBlocking(): Promise<boolean>`

和 `isBridgeEnabled()` 类似，但如果磁盘缓存是 false，会等待 GrowthBook 服务拉取结果。适合放在 entitlement gate，避免因为缓存过期而错误拒绝。

#### `getBridgeDisabledReason(): Promise<string | null>`

如果 bridge 不可用，返回用户可见的原因字符串；如果已启用，则返回 `null`。检查顺序：
1. 不是 claude.ai subscriber
2. 缺少 `user:profile` scope（setup-token / env-var OAuth tokens）
3. OAuth 账号信息里缺少 `organizationUuid`
4. `tengu_ccr_bridge` gate 关闭

#### `isEnvLessBridgeEnabled(): boolean`

门控 `tengu_bridge_repl_v2` - V2（无环境）REPL bridge 路径。带缓存，可能过期。

#### `isCseShimEnabled(): boolean`

`cse_*` → `session_*` 重标记 shim 的 kill switch。读取 `tengu_bridge_repl_v2_cse_shim_enabled`（默认 `true`）。当它是 false 时，`toCompatSessionId()` 直接 no-op。

#### `checkBridgeMinVersion(): string | null`

如果 CLI 版本低于 `tengu_bridge_min_version` 配置（默认 `'0.0.0'`），就返回错误字符串。

#### `getCcrAutoConnectDefault(): boolean`

当 `feature('CCR_AUTO_CONNECT')` 和 `tengu_cobalt_harbor` gate 都启用时返回 `true`。这是 `remoteControlAtStartup` 配置的默认值。

#### `isCcrMirrorEnabled(): boolean`

当 `feature('CCR_MIRROR')` 且 `CLAUDE_CODE_CCR_MIRROR` 环境变量为真，或者 `tengu_ccr_mirror` gate 开启时返回 `true`。

---

## 6. 会话生命周期：独立桥接（bridgeMain.ts）

**文件：** `bridge/bridgeMain.ts`

这是 `claude remote-control`（独立模式）的主循环。

### BackoffConfig

```typescript
type BackoffConfig = {
  connInitialMs: number       // 默认：2,000ms
  connCapMs: number           // 默认：120,000ms（2 分钟）
  connGiveUpMs: number        // 默认：600,000ms（10 分钟）
  generalInitialMs: number    // 默认：500ms
  generalCapMs: number       // 默认：30,000ms
  generalGiveUpMs: number     // 默认：600,000ms（10 分钟）
  shutdownGraceMs?: number    // SIGTERM→SIGKILL 宽限期。默认：30s
  stopWorkBaseDelayMs?: number // stopWork 重试基础延迟。默认：1000ms
}
```

### 常量

```typescript
STATUS_UPDATE_INTERVAL_MS = 1_000    // 实时显示刷新频率
SPAWN_SESSIONS_DEFAULT = 32          // 默认最大 session 数
```

### `runBridgeLoop(...)`

主导出函数。它负责管理：
- `activeSessions: Map<string, SessionHandle>` - 当前正在运行的 session
- `sessionStartTimes: Map<string, number>` - 用于显示已运行时长
- `sessionWorkIds: Map<string, string>` - 每个 session 对应的 work ID
- `sessionCompatIds: Map<string, string>` - 每个 session 的 `session_*` ID（每个 session 稳定）
- `sessionIngressTokens: Map<string, string>` - 每个 session 的 JWT，用于 heartbeat 认证
- `sessionTimers: Map<string, ReturnType<setTimeout>>` - 每个 session 的超时定时器
- `completedWorkIds: Set<string>` - 防止重复 stop
- `sessionWorktrees: Map<string, {...}>` - worktree 清理状态
- `timedOutSessions: Set<string>` - 被超时 watchdog 杀掉的 session
- `titledSessions: Set<string>` - 已经自动命名过的 session（抑制自动标题）
- `capacityWake: CapacityWake` - 用于在满容量睡眠时提前唤醒

**心跳逻辑：**

`heartbeatActiveWorkItems()` 会遍历所有 `activeSessions`，对每个 session 调 `api.heartbeatWork()`，并使用该 session 的 ingress token。遇到 `BridgeFatalError` 401/403（JWT 过期）时，会调用 `api.reconnectSession()` 触发服务端重新分发。返回值：
- `'ok'` - 至少有一个 heartbeat 成功
- `'auth_failed'` - 至少有一个 session 的 JWT 过期（已通过 reconnect 重新排队）
- `'fatal'` - 404/410 错误（environment expired）
- `'failed'` - 其他原因导致所有 heartbeat 都失败

**Token 刷新（主动）：**

`createTokenRefreshScheduler()` 会在每个 session 的 JWT 过期前 5 分钟触发。
- **V1 session**：调用 `handle.updateAccessToken(oauthToken)`，把新的 OAuth token 直接注入到子进程 stdin。
- **V2 session**（`v2Sessions` 集合）：调用 `api.reconnectSession()` 触发服务端重新分发并生成新 JWT（V2 子进程会校验 JWT 的 `session_id` claim，所以不能直接用 OAuth token）。

**Session 完成处理器：**

`onSessionDone()` 会清理所有 map，触发 `capacityWake.wake()`，然后：
- `status = 'completed'`：记录完成日志
- `status = 'failed'`：记录失败日志和 stderr（除非是 shutdown）
- `status = 'interrupted'`：只记 verbose 日志

对于非 interrupted 且有 `workId` 的 session，会调用 `stopWorkWithRetry()`（3 次重试，退避为 1s / 2s / 4s）。对于 worktree session，还会调用 `removeAgentWorktree()`。

**状态显示：**

每 1 秒 tick 一次，通过 `setInterval` 触发。调用 `logger.updateSessionStatus()` 给每个活跃 session 展示：已运行时长、当前活动、最近 5 条工具活动的 trail。

### Session 启动：V1 vs V2

当 work 到来时：
1. 从 `work.secret` 解码 `WorkSecret`。
2. 如果 `workSecret.use_code_sessions === true`：走 CCR v2 路径（`buildCCRv2SdkUrl`、`registerWorker()`、`useCcrV2=true`）。
3. 否则：走 V1 路径（`buildSdkUrl()`、`useCcrV2=false`）。

### SpawnMode：Worktree

当 `config.spawnMode === 'worktree'` 时：
- 调用 `createAgentWorktree()` 创建隔离的 git worktree。
- 把 worktree 路径作为 session 的工作目录。
- session 结束后调用 `removeAgentWorktree()`。

### 优雅关闭流程

1. 取消 `loopSignal`。
2. 停掉所有状态更新定时器。
3. 对每个活跃 session：调用 `handle.kill()`，等待 `handle.done`。
4. 等待所有挂起的清理（stopWork + worktree 删除）。
5. 调用 `api.deregisterEnvironment()`。
6. 如果 session 没有 fatal-exit，显示 resume 提示。

---

## 7. REPL Bridge（replBridge.ts / initReplBridge.ts）

### ReplBridgeHandle（`bridge/replBridge.ts`）

```typescript
type ReplBridgeHandle = {
  bridgeSessionId: string
  environmentId: string
  sessionIngressUrl: string
  writeMessages(messages: Message[]): void
  writeSdkMessages(messages: SDKMessage[]): void
  sendControlRequest(request: SDKControlRequest): void
  sendControlResponse(response: SDKControlResponse): void
  sendControlCancelRequest(requestId: string): void
  sendResult(): void
  teardown(): Promise<void>
}
```

### BridgeState

```typescript
type BridgeState = 'ready' | 'connected' | 'reconnecting' | 'failed'
```

### BridgeCoreParams

`initBridgeCore()` 的显式参数接口（让 daemon / 非 REPL 调用者也能用）：

```typescript
type BridgeCoreParams = {
  dir: string
  machineName: string
  branch: string
  gitRepoUrl: string | null
  title: string
  baseUrl: string
  sessionIngressUrl: string
  workerType: string
  sessionId: string             // REPL 自己的 session ID
  getAccessToken: () => string | undefined
  onAuth401?: (staleAccessToken: string) => Promise<boolean>
  toSDKMessages: (messages: Message[]) => SDKMessage[]
  initialHistoryCap: number
  pollConfig?: PollIntervalConfig
  // ... 以及 InitBridgeOptions 的回调
}
```

### InitBridgeOptions（`bridge/initReplBridge.ts`）

```typescript
type InitBridgeOptions = {
  onInboundMessage?: (msg: SDKMessage) => void | Promise<void>
  onPermissionResponse?: (response: SDKControlResponse) => void
  onInterrupt?: () => void
  onSetModel?: (model: string | undefined) => void
  onSetMaxThinkingTokens?: (maxTokens: number | null) => void
  onSetPermissionMode?: (mode: PermissionMode) => { ok: true } | { ok: false; error: string }
  onStateChange?: (state: BridgeState, detail?: string) => void
  initialMessages?: Message[]
  initialName?: string
  getMessages?: () => Message[]
  previouslyFlushedUUIDs?: Set<string>
  perpetual?: boolean
}
```

### replBridgeHandle.ts

当前活跃 REPL bridge handle 的全局指针：

```typescript
setReplBridgeHandle(h: ReplBridgeHandle | null): void
getReplBridgeHandle(): ReplBridgeHandle | null
getSelfBridgeCompatId(): string | undefined  // 返回 session_* compat ID
```

---

## 8. 无环境桥接核心（remoteBridgeCore.ts）

**文件：** `bridge/remoteBridgeCore.ts`

直接连接的 bridge，完全绕过 Environments API。

### 连接流程

1. `POST /v1/code/sessions`，body 为 `{ title, bridge: {} }` → `session.id`（`cse_*`）
2. `POST /v1/code/sessions/{id}/bridge` → `{ worker_jwt, expires_in, api_base_url, worker_epoch }`
3. `createV2ReplTransport(worker_jwt, worker_epoch)` - SSE + CCRClient
4. `createTokenRefreshScheduler(scheduleFromExpiresIn)` - 主动重新调用 `/bridge`
5. 遇到 401 SSE 时：用新的 `/bridge` 凭据重建 transport（保持相同 seq-num）

### EnvLessBridgeParams

```typescript
type EnvLessBridgeParams = {
  baseUrl: string
  orgUUID: string
  title: string
  getAccessToken: () => string | undefined
  onAuth401?: (staleAccessToken: string) => Promise<boolean>
  toSDKMessages: (messages: Message[]) => SDKMessage[]
  initialHistoryCap: number
  initialMessages?: Message[]
  onInboundMessage?: (msg: SDKMessage) => void | Promise<void>
  onUserMessage?: (text: string, sessionId: string) => boolean
  onPermissionResponse?: (response: SDKControlResponse) => void
  onInterrupt?: () => void
  onSetModel?: (model: string | undefined) => void
  onSetMaxThinkingTokens?: (maxTokens: number | null) => void
  onSetPermissionMode?: (mode: PermissionMode) => { ok: true } | { ok: false; error: string }
  onStateChange?: (state: BridgeState, detail?: string) => void
  perpetual?: boolean
}
```

### Connect Cause Telemetry

```typescript
type ConnectCause = 'initial' | 'proactive_refresh' | 'auth_401_recovery'
```

会随 `tengu_bridge_repl_v2_ws_connected` analytics event 一起发送。

---

## 9. 传输层

### ReplBridgeTransport 接口（`bridge/replBridgeTransport.ts`）

对 V1（HybridTransport）和 V2（SSETransport + CCRClient）做抽象：

```typescript
type ReplBridgeTransport = {
  write(message: StdoutMessage): Promise<void>
  writeBatch(messages: StdoutMessage[]): Promise<void>
  close(): void
  isConnectedStatus(): boolean
  getStateLabel(): string
  setOnData(callback: (data: string) => void): void
  setOnClose(callback: (closeCode?: number) => void): void
  setOnConnect(callback: () => void): void
  connect(): void
  getLastSequenceNum(): number    // V1 永远返回 0；V2 返回 SSE seq
  readonly droppedBatchCount: number  // 仅 V1；V2 永远是 0
  reportState(state: SessionState): void      // 仅 V2；V1 no-op
  reportMetadata(metadata: Record<string, unknown>): void  // 仅 V2
  reportDelivery(eventId: string, status: 'processing' | 'processed'): void  // 仅 V2
  flush(): Promise<void>          // 仅 V2；V1 立即 resolve
}
```

### `createV1ReplTransport(hybrid: HybridTransport): ReplBridgeTransport`

薄的 no-op 包装器，直接代理给 `HybridTransport`。V1 特有行为：
- `getLastSequenceNum()` 永远返回 0
- `reportState()`、`reportMetadata()`、`reportDelivery()`、`flush()` 都是 no-op

### `createV2ReplTransport(opts): Promise<ReplBridgeTransport>`

Options：
```typescript
{
  sessionUrl: string           // /v1/code/sessions/{id}
  ingressToken: string
  sessionId: string
  initialSequenceNum?: number  // SSE resume cursor
  epoch?: number               // 如果来自 /bridge，server 已经 bump 过 epoch
  heartbeatIntervalMs?: number // 默认：20s
  heartbeatJitterFraction?: number
  outboundOnly?: boolean       // 跳过 SSE 读流（mirror mode）
  getAuthToken?: () => string | undefined  // 每实例 auth（支持多 session）
}
```

**内部使用的 Close Code：**
- `4090` - epoch superseded（CCRClient 的 epoch 不匹配）
- `4091` - CCR initialize() 失败
- `4092` - SSE reconnect-budget 耗尽（由 `undefined` 映射）

**Delivery ACK 行为：** SSE 事件一到达，就会立刻触发 `'received'` 和 `'processed'` 两个 ACK，避免重启后 prompt 被“幽灵式”重复灌入。这是为了修复 `reconnectSession` 把还没 ACK 成 `'processed'` 的 prompt 重新排队的问题。

### HybridTransport（`cli/transports/HybridTransport.ts`）

继承 `WebSocketTransport`。读走 WebSocket，写走 HTTP POST。

**配置：**
```typescript
BATCH_FLUSH_INTERVAL_MS = 100     // stream_events 会先聚合再 POST
POST_TIMEOUT_MS = 15_000           // 每次尝试的超时
CLOSE_GRACE_MS = 3000              // close 时给排队写入的宽限期
```

**写入流程：**
```text
write(stream_event) ─┐
                     │ (100ms timer)
write(other) ──────► SerialBatchEventUploader.enqueue()
writeBatch() ───────┘     │
                          ▼ 串行、批量、无限重试
                     postOnce()  (单次 HTTP POST)
```

- `maxBatchSize`: 500
- `maxQueueSize`: 100,000
- `baseDelayMs`: 500，`maxDelayMs`: 8000，`jitterMs`: 1000

**Post URL：** 会把 WebSocket URL 转成 HTTP(S) POST 端点（`convertWsUrlToPostUrl()`）。

### WebSocketTransport（`cli/transports/WebSocketTransport.ts`）

基础 WebSocket transport。

**配置：**
```typescript
DEFAULT_MAX_BUFFER_SIZE = 1000
DEFAULT_BASE_RECONNECT_DELAY = 1000
DEFAULT_MAX_RECONNECT_DELAY = 30_000
DEFAULT_RECONNECT_GIVE_UP_MS = 600_000   // 10 minutes
DEFAULT_PING_INTERVAL = 10_000
DEFAULT_KEEPALIVE_INTERVAL = 300_000     // 5 minutes
SLEEP_DETECTION_THRESHOLD_MS = 60_000   // 2× max reconnect delay
KEEP_ALIVE_FRAME = '{"type":"keep_alive"}\n'
```

**永久关闭码**（不重试）：
- `1002` - 协议错误（session 已被回收）
- `4001` - session 过期 / 未找到
- `4003` - 未授权

**睡眠检测：** 如果重连尝试之间的间隔超过 `60s`，会重置重连预算并重新尝试（机器大概率睡过）。

**状态：** `'idle' | 'connected' | 'reconnecting' | 'closing' | 'closed'`

### SSETransport（`cli/transports/SSETransport.ts`）

Server-Sent Events transport。

**配置：**
```typescript
RECONNECT_BASE_DELAY_MS = 1000
RECONNECT_MAX_DELAY_MS = 30_000
RECONNECT_GIVE_UP_MS = 600_000    // 10 minutes
LIVENESS_TIMEOUT_MS = 45_000      // server 每 15s 发 keepalive
PERMANENT_HTTP_CODES = {401, 403, 404}
POST_MAX_RETRIES = 10
POST_BASE_DELAY_MS = 500
POST_MAX_DELAY_MS = 8000
```

**SSE 帧解析：**

```typescript
type SSEFrame = {
  event?: string
  id?: string       // 作为 Last-Event-ID 的 sequence number
  data?: string
}
```

帧之间用双换行分隔。`:` 后面的前导空格会按 SSE 规范剥掉。注释（`:keepalive`）会被忽略。sequence number 通过 `id` 字段跟踪。

**供测试导出：** `parseSSEFrames(buffer: string): { frames: SSEFrame[]; remaining: string }`

**sequence number 继承：** 重连时会发送 `Last-Event-ID` 或 `from_sequence_num` 查询参数，这样 server 能从旧流中断处继续。

### SerialBatchEventUploader（`cli/transports/SerialBatchEventUploader.ts`）

```typescript
type SerialBatchEventUploaderConfig<T> = {
  maxBatchSize: number         // 每次 POST 的最大条数
  maxBatchBytes?: number       // 每次 POST 的最大序列化字节数
  maxQueueSize: number         // 队列最多积压多少项，再多 enqueue() 会阻塞
  send: (batch: T[]) => Promise<void>  // 真正的 HTTP 调用
  baseDelayMs: number
  maxDelayMs: number
  jitterMs: number
  maxConsecutiveFailures?: number   // 失败 N 次后丢弃 batch 并前移
  onBatchDropped?: (batchSize, failures) => void
}
```

**`class RetryableError extends Error`**：如果在 `config.send()` 里抛这个异常，就会用服务端给出的 `retryAfterMs` 覆盖指数退避（例如 429）。

**背压：** 当达到 `maxQueueSize` 时，`enqueue()` 会阻塞。

### WorkerStateUploader（`cli/transports/WorkerStateUploader.ts`）

用于 `PUT /worker` 的合并式 uploader（session state + metadata）。

- 最多只有 1 个 in-flight PUT + 1 个 pending patch（永远不会超过 2 个槽位）。
- 合并规则：
  - 顶层 key：最后一个值生效。
  - `external_metadata` / `internal_metadata`：按 RFC 7396 做 merge（`null` 值保留，用于服务端删除）。

### CCRClient（`cli/transports/ccrClient.ts`）

```typescript
type CcrClient = {
  initialize(): Promise<void>
  close(): Promise<void>
  write(message: StdoutMessage): Promise<void>
  writeBatch(messages: StdoutMessage[]): Promise<void>
  reportState(state: SessionState): Promise<void>
  reportMetadata(metadata: Record<string, unknown>): Promise<void>
  reportDelivery(eventId: string, status: 'processing' | 'processed'): Promise<void>
}
```

---

## 10. 消息协议（bridgeMessaging.ts）

**文件：** `bridge/bridgeMessaging.ts`

### `handleServerControlRequest(request, handlers): void`

处理 server 发来的 `control_request` 消息。必须尽快响应，因为 server 大约会在 10-14 秒后把 WS 断掉。

**支持的子类型：**

| 子类型 | 行为 |
|---------|----------|
| `initialize` | 返回 `{ commands: [], output_style: 'normal', available_output_styles: ['normal'], models: [], account: {}, pid: process.pid }` |
| `set_model` | 调用 `onSetModel(request.request.model)`，返回成功 |
| `set_max_thinking_tokens` | 调用 `onSetMaxThinkingTokens(maxTokens)`，返回成功 |
| `set_permission_mode` | 调用 `onSetPermissionMode(mode)`，返回成功或错误 |
| `interrupt` | 调用 `onInterrupt()`，返回成功 |
| unknown | 返回错误：`"REPL bridge does not handle control_request subtype: ..."` |

**仅 outbound 模式：** 所有会修改状态的请求都会返回错误 `'This session is outbound-only...'`。`initialize` 仍然会成功返回。

**响应封装：**
```json
{
  "type": "control_response",
  "response": {
    "subtype": "success" | "error",
    "request_id": "<id>",
    ...
  },
  "session_id": "<sessionId>"
}
```

### `makeResultMessage(sessionId: string): SDKResultSuccess`

构造一个用于 session 归档的最小 result message：
```json
{
  "type": "result",
  "subtype": "success",
  "duration_ms": 0,
  "duration_api_ms": 0,
  "is_error": false,
  "num_turns": 0,
  "result": "",
  "stop_reason": null,
  "total_cost_usd": 0,
  "usage": {...},
  "modelUsage": {},
  "permission_denials": [],
  "session_id": "<sessionId>",
  "uuid": "<randomUUID>"
}
```

### `class BoundedUUIDSet`

用于 UUID 去重的 FIFO ring buffer。内存占用是 O(capacity)。

```typescript
class BoundedUUIDSet {
  constructor(capacity: number)
  add(uuid: string): void   // 到容量上限时移除最旧项
  has(uuid: string): boolean
  clear(): void
}
```

用于：
- `recentPostedUUIDs` - 回声抑制（我们发出的消息又被反射回来）
- `recentInboundUUIDs` - 重投去重（transport 切换后 server 重新回放历史）

---

## 11. JWT 认证（jwtUtils.ts）

**文件：** `bridge/jwtUtils.ts`

### `decodeJwtPayload(token: string): unknown | null`

不验证签名，直接解 JWT payload 段。会剥掉 `sk-ant-si-` 前缀（如果有）。返回解析后的 JSON，解析失败则返回 `null`。

### `decodeJwtExpiry(token: string): number | null`

不验证签名，直接提取 JWT 里的 `exp` Unix 秒级 claim。无法解析时返回 `null`。

### `createTokenRefreshScheduler(opts): { schedule, scheduleFromExpiresIn, cancel, cancelAll }`

**选项：**
```typescript
{
  getAccessToken: () => string | undefined | Promise<string | undefined>
  onRefresh: (sessionId: string, oauthToken: string) => void
  label: string
  refreshBufferMs?: number   // 默认：TOKEN_REFRESH_BUFFER_MS = 5 min
}
```

**常量：**
```typescript
TOKEN_REFRESH_BUFFER_MS = 5 * 60 * 1000      // 过期前 5 分钟
FALLBACK_REFRESH_INTERVAL_MS = 30 * 60 * 1000 // 30 分钟（兜底）
MAX_REFRESH_FAILURES = 3
REFRESH_RETRY_DELAY_MS = 60_000               // 1 分钟
```

**方法：**

`schedule(sessionId, token)`：从 JWT 里解 `exp`，然后把刷新安排在 `(exp × 1000 - now - refreshBufferMs)` 毫秒后。如果 token 没有可解码的 `exp`（例如 OAuth token），就保留现有 timer。

`scheduleFromExpiresIn(sessionId, expiresInSeconds)`：用显式 TTL 安排刷新。下限钳到 30 秒：`max(expiresInSeconds × 1000 - refreshBufferMs, 30_000)`。

`cancel(sessionId)`：清掉 timer，并提升 generation，令进行中的刷新失效。

`cancelAll()`：清掉所有 timer 和失败计数。

**generation 跟踪：** 每个 session 都有单调递增的 generation 计数。`doRefresh()` 会先检查 generation 是否已经变化，避免 session 被取消后又冒出孤儿 timer。

**后续刷新：** 每次刷新成功后，都会再安排一个 `FALLBACK_REFRESH_INTERVAL_MS`（30 分钟）的 follow-up 刷新，用于支撑长时间运行、超过首轮刷新窗口的 session。

---

## 12. Session ID 兼容层（sessionIdCompat.ts）

**文件：** `bridge/sessionIdCompat.ts`

处理 V2 兼容层里的 `cse_*` ↔ `session_*` ID 互转。

**问题：** CCR V2 基础设施内部使用 `cse_*` 前缀；兼容网关和面向客户端的 API（`/v1/sessions`）则期待 `session_*`。UUID 一样，只是前缀不同。

### 导出

#### `setCseShimGate(gate: () => boolean): void`

注册 GrowthBook gate `isCseShimEnabled`。由已经导入 `bridgeEnabled.ts` 的 bridge init 代码调用。SDK bundle 不会调用这个函数，所以 shim 默认是启用的。

#### `toCompatSessionId(id: string): string`

把 `cse_*` 重标记成 `session_*`，用于 compat API 调用（`/v1/sessions/{id}`、`/archive`、`/events`）。如果不是 `cse_*`，就 no-op。shim gate 关闭时也 no-op。

```
"cse_abc123" → "session_abc123"
"session_abc123" → "session_abc123"  (no-op)
```

#### `toInfraSessionId(id: string): string`

反向操作：把 `session_*` 重标记成 `cse_*`，用于基础设施调用（`/bridge/reconnect`）。如果不是 `session_*`，就 no-op。

```
"session_abc123" → "cse_abc123"
"cse_abc123" → "cse_abc123"  (no-op)
```

---

## 13. Work Secret 与 CCR v2 注册（workSecret.ts）

**文件：** `bridge/workSecret.ts`

### `decodeWorkSecret(secret: string): WorkSecret`

解码 base64url 编码的 work secret JSON。校验：
- 必须是 version-1 secret。
- `session_ingress_token` 必须是非空字符串。
- `api_base_url` 必须是字符串。

### `buildSdkUrl(apiBaseUrl: string, sessionId: string): string`

构建 V1 WebSocket URL：
- Localhost：`ws://host/v2/session_ingress/ws/{sessionId}`（直接连 session-ingress）
- Production：`wss://host/v1/session_ingress/ws/{sessionId}`（Envoy 会把 `/v1/` 重写到 `/v2/`）

### `sameSessionId(a: string, b: string): boolean`

不看前缀（`cse_` vs `session_`）比较两个 session ID。只比较 UUID 主体（最后一个 `_` 后面的部分）。为了避免对格式错误的 ID 误判，主体长度必须至少 4。

### `buildCCRv2SdkUrl(apiBaseUrl: string, sessionId: string): string`

构建 V2 HTTP(S) session URL：`{apiBaseUrl}/v1/code/sessions/{sessionId}`

### `registerWorker(sessionUrl: string, accessToken: string): Promise<number>`

`POST {sessionUrl}/worker/register`，注册为 CCR worker。返回 `worker_epoch`。这个 epoch 会被序列化成 `int64`，protojson 可能把它返回成字符串，所以这里会兼容 `string` 和 `number` 两种形式。

---

## 14. Bridge Pointer：崩溃恢复（bridgePointer.ts）

**文件：** `bridge/bridgePointer.ts`

### 用途

这是一个崩溃恢复指针，会在 session 创建后写入，定期刷新，在正常关闭时清掉。下一次启动时，`claude remote-control` 会检测旧指针，并提供恢复选项。

### 常量

```typescript
BRIDGE_POINTER_TTL_MS = 4 * 60 * 60 * 1000   // 4 小时（和 Redis BRIDGE_LAST_POLL_TTL 一致）
MAX_WORKTREE_FANOUT = 50
```

### BridgePointer 模式

```typescript
type BridgePointer = {
  sessionId: string
  environmentId: string
  source: 'standalone' | 'repl'
}
```

**文件位置：** `{projectsDir}/{sanitizedDir}/bridge-pointer.json`

### 导出

#### `getBridgePointerPath(dir: string): string`

返回指定工作目录对应的 bridge pointer 文件绝对路径。

#### `writeBridgePointer(dir, pointer): Promise<void>`

原子写入 pointer。也会用于刷新 mtime（同内容写入）。尽力而为 - 记录日志，但会吞掉错误。

#### `readBridgePointer(dir): Promise<(BridgePointer & { ageMs: number }) | null>`

读取 pointer。以下情况返回 `null`：
- 文件不存在
- JSON 格式错误
- schema 校验失败
- `mtime` 超过 4 小时（过期）

过期或无效的 pointer 会自动删除。

#### `readBridgePointerAcrossWorktrees(dir): Promise<{ pointer, dir } | null>`

支持 worktree 的 `--continue` 读取。会先快速检查给定目录；如果找不到，就通过 `getWorktreePathsPortable()` 并行扫描 git worktree 兄弟目录（上限 50）。返回最新的 pointer 和所在目录。

#### `clearBridgePointer(dir): Promise<void>`

删除 pointer 文件。幂等（正常关闭时 ENOENT 是预期内的）。

---

## 15. 权限回调（bridgePermissionCallbacks.ts）

**文件：** `bridge/bridgePermissionCallbacks.ts`

### 类型

```typescript
type BridgePermissionResponse = {
  behavior: 'allow' | 'deny'
  updatedInput?: Record<string, unknown>
  updatedPermissions?: PermissionUpdate[]
  message?: string
}

type BridgePermissionCallbacks = {
  sendRequest(
    requestId, toolName, input, toolUseId, description,
    permissionSuggestions?, blockedPath?
  ): void
  sendResponse(requestId, response: BridgePermissionResponse): void
  cancelRequest(requestId): void
  onResponse(
    requestId,
    handler: (response: BridgePermissionResponse) => void
  ): () => void  // 返回 unsubscribe 函数
}
```

### `isBridgePermissionResponse(value: unknown): value is BridgePermissionResponse`

类型谓词。检查 `value.behavior === 'allow' || value.behavior === 'deny'`。

---

## 16. 入站消息与附件

### inboundAttachments.ts

解析桥接用户消息中的 `file_uuid` 附件。

**流程：**
1. 网页端 composer 通过 `/api/{org}/upload` 上传（cookie 认证）。
2. Bridge 在用户消息里收到 `file_attachments: [{ file_uuid, file_name }]`。
3. `resolveInboundAttachments()` 会通过 `GET /api/oauth/files/{uuid}/content` 拉取每个文件。
4. 下载到 `~/.claude/uploads/{sessionId}/{uuid-prefix}-{sanitizedName}`。
5. 返回一个 `@"path"` 前缀字符串，拼到消息内容前面。

**`DOWNLOAD_TIMEOUT_MS = 30_000`**

**文件路径清理：** `sanitizeFileName()` 会去掉路径组件，并把除 `._-` 之外的非字母数字字符替换成 `_`。

**前缀格式：** 从 `file_uuid`（或随机 UUID）里取前 8 个字符，例如 `@"abc12345-filename.pdf"`。

**导出：**
```typescript
extractInboundAttachments(msg: unknown): InboundAttachment[]
resolveInboundAttachments(attachments: InboundAttachment[]): Promise<string>
prependPathRefs(
  content: string | Array<ContentBlockParam>,
  prefix: string,
): string | Array<ContentBlockParam>
resolveAndPrepend(
  msg: unknown,
  content: string | Array<ContentBlockParam>,
): Promise<string | Array<ContentBlockParam>>
```

`prependPathRefs()` 会作用在内容数组里的**最后一个**文本 block 上（因为 `processUserInputBase` 读取最后一个 block）。

### inboundMessages.ts

**`extractInboundMessageFields(msg: SDKMessage): { content, uuid } | undefined`**

从用户消息里提取内容和 UUID。会规范化图片 block：
- 把 camelCase 的 `mediaType` 转成 `media_type`（移动端兼容修复，针对 `mobile-apps#5825`）
- 如果缺少 `media_type`，会通过 `detectImageFormatFromBase64()` 推断

**`normalizeImageBlocks(blocks: ContentBlockParam[]): ContentBlockParam[]`**

快路径：如果没有坏格式 block，直接返回原始引用。只有在需要修复时才分配新对象。

---

## 17. 会话运行器（sessionRunner.ts）

**文件：** `bridge/sessionRunner.ts`

负责启动子 Claude CLI 进程，用于 bridge session。

### 常量

```typescript
MAX_ACTIVITIES = 10      // activity 历史的 ring buffer 大小
MAX_STDERR_LINES = 10    // stderr ring buffer 大小
```

### `safeFilenameId(id: string): string`

把 session ID 清理成适合文件名的形式。把除了 `_-` 之外的非字母数字字符都替换成下划线。

### PermissionRequest

子 CLI 在 stdout 上发出的权限请求消息：

```typescript
type PermissionRequest = {
  type: 'control_request'
  request_id: string
  request: {
    subtype: 'can_use_tool'
    tool_name: string
    input: Record<string, unknown>
    tool_use_id: string
  }
}
```

### SessionSpawnerDeps

```typescript
type SessionSpawnerDeps = {
  execPath: string
  scriptArgs: string[]        // 编译后二进制时为空；npm 时为 [process.argv[1]]
  env: NodeJS.ProcessEnv
  verbose: boolean
  sandbox: boolean
  debugFile?: string
  permissionMode?: string
  onDebug: (msg: string) => void
  onActivity?: (sessionId, activity) => void
  onPermissionRequest?: (sessionId, request, accessToken) => void
}
```

### 工具活动动词

```typescript
const TOOL_VERBS = {
  Read: 'Reading', Write: 'Writing', Edit: 'Editing', MultiEdit: 'Editing',
  Bash: 'Running', Glob: 'Searching', Grep: 'Searching',
  WebFetch: 'Fetching', WebSearch: 'Searching', Task: 'Running task',
  FileReadTool: 'Reading', FileWriteTool: 'Writing', FileEditTool: 'Editing',
  GlobTool: 'Searching', GrepTool: 'Searching', BashTool: 'Running',
  NotebookEditTool: 'Editing notebook', LSP: 'LSP',
}
```

---

## 18. 桥接调试与故障注入（bridgeDebug.ts）

**文件：** `bridge/bridgeDebug.ts`

给桥接恢复路径做测试用的 Ant-only 故障注入。在外部 build 中零开销。

### BridgeFault

```typescript
type BridgeFault = {
  method: 'pollForWork' | 'registerBridgeEnvironment' | 'reconnectSession' | 'heartbeatWork'
  kind: 'fatal' | 'transient'
  status: number
  errorType?: string
  count: number          // 消耗时递减；到 0 后删除
}
```

- **fatal**：抛 `BridgeFatalError` - 触发 environment teardown。
- **transient**：抛普通 `Error`（模拟 5xx / network） - 触发 retry / backoff。

### BridgeDebugHandle

```typescript
type BridgeDebugHandle = {
  fireClose: (code: number) => void       // 触发 transport 的 permanent-close handler
  forceReconnect: () => void              // 调 reconnectEnvironmentWithSession()
  injectFault: (fault: BridgeFault) => void
  wakePollLoop: () => void                // 立刻打断 at-capacity sleep
  describe: () => string                  // "envId=... sessionId=..."
}
```

### 导出

```typescript
registerBridgeDebugHandle(h: BridgeDebugHandle): void
clearBridgeDebugHandle(): void
getBridgeDebugHandle(): BridgeDebugHandle | null
injectBridgeFault(fault: BridgeFault): void
wrapApiForFaultInjection(api: BridgeApiClient): BridgeApiClient
```

`wrapApiForFaultInjection()` 会给 `pollForWork`、`registerBridgeEnvironment`、`reconnectSession` 和 `heartbeatWork` 套上故障队列检查。其他方法保持原样。

---

## 19. 桥接工具函数

### debugUtils.ts

```typescript
redactSecrets(s: string): string
```

会脱敏以下敏感字段值：`session_ingress_token`、`environment_secret`、`access_token`、`secret`、`token`。长度短于 16 个字符的值会变成 `[REDACTED]`。更长的值会显示成 `first8chars...last4chars`。

```typescript
debugTruncate(s: string): string          // 2000 字符上限，压平换行
debugBody(data: unknown): string          // 序列化 + 脱敏 + 截断
describeAxiosError(err: unknown): string  // 从 axios error 里提取 server message
extractHttpStatus(err: unknown): number | undefined
extractErrorDetail(data: unknown): string | undefined  // 检查 data.message、data.error.message
logBridgeSkip(reason, debugMsg?, v2?): void  // 记录 bridge skip 的分析和 debug
```

### bridgeStatusUtil.ts

```typescript
type StatusState = 'idle' | 'attached' | 'titled' | 'reconnecting' | 'failed'
TOOL_DISPLAY_EXPIRY_MS = 30_000   // tool activity 显示多久后消失
SHIMMER_INTERVAL_MS = 150         // shimmer 动画 tick

timestamp(): string                // "HH:MM:SS"
formatDuration(ms): string         // 重新导出自 utils/format.ts
truncatePrompt(s, width): string   // 重新导出
abbreviateActivity(summary): string  // 截到 30 字符

buildBridgeConnectUrl(environmentId, ingressUrl?): string
  // → "{baseUrl}/code?bridge={environmentId}"

buildBridgeSessionUrl(sessionId, environmentId, ingressUrl?): string
  // → "{remoteSessionUrl}?bridge={environmentId}"

computeGlimmerIndex(tick, messageWidth): number
computeShimmerSegments(text, glimmerIndex): { before, shimmer, after }

getBridgeStatus({ error, connected, sessionActive, reconnecting }): BridgeStatusInfo
  // 返回 { label, color }，供 UI 渲染

buildIdleFooterText(url): string
buildActiveFooterText(url): string
FAILED_FOOTER_TEXT = 'Something went wrong, please try again'

wrapWithOsc8Link(text, url): string   // OSC 8 终端超链接
```

### capacityWake.ts

```typescript
type CapacitySignal = { signal: AbortSignal; cleanup: () => void }
type CapacityWake = {
  signal(): CapacitySignal  // 合并 abort：外层循环或容量唤醒
  wake(): void              // 打断当前 sleep，重新准备 controller
}

createCapacityWake(outerSignal: AbortSignal): CapacityWake
```

这是 `replBridge.ts` 和 `bridgeMain.ts` 共享的原语：在满容量时睡眠，并且在 session 结束或外层 signal abort 时提前唤醒。

### flushGate.ts

```typescript
class FlushGate<T> {
  get active(): boolean
  get pendingCount(): number
  start(): void                  // 标记 flush 正在进行；enqueue() 只排队
  end(): T[]                     // 结束 flush，返回排队项
  enqueue(...items: T[]): boolean  // 如果 active 就排队，否则返回 false
  drop(): number                 // 丢弃所有（永久关闭）；返回丢弃数量
  deactivate(): void             // 清掉 active，但不丢数据（用于 transport 替换）
}
```

用于初始历史 flush 期间把并发到来的新消息先排队，避免服务端交错。

### pollConfig.ts / pollConfigDefaults.ts

`getPollIntervalConfig(): PollIntervalConfig` - 从 GrowthBook `tengu_bridge_poll_interval_config` 读取，5 分钟刷新一次。如果 schema 校验失败，就回退到默认值。

```typescript
type PollIntervalConfig = {
  poll_interval_ms_not_at_capacity: number        // 默认：2000ms
  poll_interval_ms_at_capacity: number            // 默认：600,000ms（10 分钟）
  non_exclusive_heartbeat_interval_ms: number     // 默认：0（禁用）
  multisession_poll_interval_ms_not_at_capacity: number  // 默认：2000ms
  multisession_poll_interval_ms_partial_capacity: number // 默认：2000ms
  multisession_poll_interval_ms_at_capacity: number      // 默认：600,000ms
  reclaim_older_than_ms: number                   // 默认：5000ms
  session_keepalive_interval_v2_ms: number        // 默认：120,000ms
}
```

**校验规则：**
- `poll_interval_ms_*` 和 `multisession_*` 字段：最小值 100ms。
- `non_exclusive_heartbeat_interval_ms`：最小值 0（0 = 禁用）。
- at-capacity 间隔：0（禁用）或 ≥100ms（1-99 会被拒绝，避免和秒混淆）。
- 对象级别：至少要有一种 at-capacity liveness 机制开启（heartbeat > 0 或 at-capacity poll > 0）。

### envLessBridgeConfig.ts

`getEnvLessBridgeConfig(): Promise<EnvLessBridgeConfig>` - 从 GrowthBook `tengu_bridge_repl_v2_config` 读取。默认值：

```typescript
DEFAULT_ENV_LESS_BRIDGE_CONFIG = {
  init_retry_max_attempts: 3,
  init_retry_base_delay_ms: 500,
  init_retry_jitter_fraction: 0.25,
  init_retry_max_delay_ms: 4000,
  http_timeout_ms: 10_000,
  uuid_dedup_buffer_size: 2000,
  heartbeat_interval_ms: 20_000,
  heartbeat_jitter_fraction: 0.1,
  token_refresh_buffer_ms: 300_000,
  teardown_archive_timeout_ms: 1500,
  connect_timeout_ms: 15_000,
  min_version: '0.0.0',
  should_show_app_upgrade_message: false,
}
```

校验范围：
- `init_retry_max_attempts`: 1–10
- `http_timeout_ms`: 至少 2000ms
- `heartbeat_interval_ms`: 5000–30,000ms
- `heartbeat_jitter_fraction`: 0–0.5
- `token_refresh_buffer_ms`: 30,000–1,800,000ms
- `teardown_archive_timeout_ms`: 500–2000ms
- `connect_timeout_ms`: 5,000–60,000ms

### trustedDevice.ts

管理 ELEVATED 安全等级 bridge session 的 trusted device token。

```typescript
getTrustedDeviceToken(): string | undefined
clearTrustedDeviceTokenCache(): void
clearTrustedDeviceToken(): void
enrollTrustedDevice(): Promise<void>
```

**门控：** `tengu_sessions_elevated_auth_enforcement`。如果这个 gate 关了，`getTrustedDeviceToken()` 会无条件返回 `undefined`。

**存储：** macOS keychain，通过 `getSecureStorage()`。这个对象是 memoized 的，所以启动 keychain 大概只要 40ms。

**Token 优先级：** `CLAUDE_TRUSTED_DEVICE_TOKEN` 环境变量 > keychain。

**注册：** `POST /api/auth/trusted_devices`，display name 为 `"Claude Code on {hostname()} · {platform}"`。必须在登录后 10 分钟内调用。尽力而为，不会抛错。成功后会写入 keychain，并清掉 memo cache。

### codeSessionApi.ts

CCR V2 code-session API 的轻量 HTTP 包装器。

```typescript
createCodeSession(
  baseUrl, accessToken, title, timeoutMs, tags?
): Promise<string | null>
// POST /v1/code/sessions → session.id (cse_*)
// Body: { title, bridge: {}, tags?: [...] }

type RemoteCredentials = {
  worker_jwt: string
  api_base_url: string
  expires_in: number         // 秒
  worker_epoch: number
}

fetchRemoteCredentials(
  sessionId, baseUrl, accessToken, timeoutMs, trustedDeviceToken?
): Promise<RemoteCredentials | null>
// POST /v1/code/sessions/{id}/bridge
```

`worker_epoch` 会被防御性解析（protojson 可能把 int64 变成字符串）。

### sessionRunner.ts（完整 createSessionSpawner）

`createSessionSpawner()`（由 `bridgeMain.ts` 引用）会创建一个 `SessionSpawner`，它会：
1. 使用 Claude binary 和合适的参数调用 `spawn(child_process)`。
2. 把子进程 stdout 解析成 NDJSON，并把 `control_request` 消息路由到权限回调。
3. 在大小为 10 的 ring buffer 里跟踪 `SessionActivity`。
4. 跟踪最后 10 行 stderr。
5. 监听子进程退出，按退出状态把 `handle.done` 解析成 `'completed'`（exit 0）、`'failed'`（exit != 0）或 `'interrupted'`（SIGTERM / SIGKILL）。

---

## 20. CLI 框架

### cli/exit.ts

```typescript
cliError(msg?: string): never   // stderr + process.exit(1)
cliOk(msg?: string): never      // stdout + process.exit(0)
```

统一的 CLI 退出辅助函数。`cliError` 用 `console.error`；`cliOk` 用 `process.stdout.write`。`never` 返回类型可以让 TypeScript 在调用点缩窄控制流。

### cli/ndjsonSafeStringify.ts

```typescript
ndjsonSafeStringify(value: unknown): string
```

JSON 序列化器，会把 `U+2028`（LINE SEPARATOR）和 `U+2029`（PARAGRAPH SEPARATOR）转义成 `\u2028` / `\u2029`。这两个字符虽然在 JSON 里合法，但在 JavaScript 里是行终止符，会破坏按行拆分的 NDJSON 接收端。转义后的形式仍然是合法 JSON。

### cli/handlers/auth.ts

```typescript
installOAuthTokens(tokens: OAuthTokens): Promise<void>
authLogin({ email?, sso?, console?, claudeai? }): Promise<void>
authStatus({ json?, text? }): Promise<void>
authLogout(): Promise<void>
```

**`installOAuthTokens()`** 会执行以下后续处理：
1. `performLogout({ clearOnboarding: false })` - 清掉旧状态
2. 获取 OAuth profile，或者退回到 `tokenAccount`
3. 调用 `storeOAuthAccountInfo()`
4. `saveOAuthTokensIfNeeded()` + `clearOAuthTokenCache()`
5. `fetchAndStoreUserRoles()`（尽力而为）
6. 对 claude.ai auth：`fetchAndStoreClaudeCodeFirstTokenDate()`（尽力而为）
7. 对 Console auth：`createAndStoreApiKey()`（必须成功 - 失败会抛）
8. `clearAuthRelatedCaches()`

**`authLogin()`** 快路径：如果设置了 `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` 环境变量，会直接通过 `refreshOAuthToken()` 做交换，跳过浏览器 OAuth 流程。此时还需要 `CLAUDE_CODE_OAUTH_SCOPES` 环境变量（空格分隔的 scope）。

**`authStatus()`** 的 JSON 输出字段：
```json
{
  "loggedIn": boolean,
  "authMethod": "none" | "claude.ai" | "api_key_helper" | "oauth_token" | "api_key" | "third_party",
  "apiProvider": string,
  "apiKeySource": string,       // apiKey 时存在
  "email": string | null,       // claude.ai 时存在
  "orgId": string | null,
  "orgName": string | null,
  "subscriptionType": string | null
}
```

### cli/handlers/agents.ts

```typescript
agentsHandler(): Promise<void>
```

按来源分组列出已配置的 agents。输出格式：每个 agent 一行 `agentType · model · memory`。被更高优先级来源覆盖的 shadowed agents 会显示 `(shadowed by {source})` 前缀。

### cli/handlers/autoMode.ts

```typescript
autoModeDefaultsHandler(): void      // 以 JSON 输出默认 auto mode 规则
autoModeConfigHandler(): void        // 输出生效后的配置（用户设置或默认值）
autoModeCritiqueHandler({ model? }): Promise<void>  // 对用户规则做 AI 评审
```

**Auto mode 规则分类：** `allow`、`soft_deny`、`environment`。

**评审** 使用 `sideQuery()`，配套一个专门的 system prompt，让 Claude 评估规则的清晰度、完整性、冲突和可执行性。

### cli/handlers/mcp.tsx（节选）

```typescript
mcpServeHandler({ debug?, verbose? }): Promise<void>
mcpRemoveHandler(name, { scope? }): Promise<void>
// ... 还有 add、get、list、reset、import、desktop-import 等 handler
```

### cli/handlers/plugins.ts（节选）

`claude plugin *` 和 marketplace 命令的 handler：
- `installPlugin`、`uninstallPlugin`、`enablePlugin`、`disablePlugin`
- `listPlugins`、`marketplaceSearch`、`addMarketplace`、`removeMarketplace`

### cli/print.ts

SDK 的主 `-p`（print mode）handler。负责：
- `StructuredIO` / `RemoteIO` 的 I/O
- 工具池组装
- 消息队列管理
- session state 通知
- 通过 `enableRemoteControl` 做可选的 bridge 启用

### cli/update.ts

```typescript
update(): Promise<void>
```

处理 `claude update`。会检查当前版本，识别安装类型，选择 updater（npm global、native binary、local）。如果检测到多个安装，会给出警告。

---

## 21. CLI 传输层

参见 [第 9 节 - 传输层](#9-传输层) 了解传输层级结构。

### cli/remoteIO.ts - RemoteIO 类

```typescript
class RemoteIO extends StructuredIO {
  constructor(streamUrl: string, initialPrompt?, replayUserMessages?)
}
```

SDK 模式的双向流式传输。继承自 `StructuredIO`。

**构造函数行为：**
1. 创建 `PassThrough` 输入流。
2. 读取 `CLAUDE_CODE_SESSION_ACCESS_TOKEN` 作为初始认证头。
3. 读取 `CLAUDE_CODE_ENVIRONMENT_RUNNER_VERSION` 作为 `x-environment-runner-version` 头。
4. 创建 `refreshHeaders` 闭包，在重连时动态重读 token。
5. 调用 `getTransportForUrl()` 选择 transport（WS、Hybrid 或 SSE）。

**Keep-alive：** 当 `session_keepalive_interval_v2_ms > 0` 且 transport 是 SSE/v2 时，会按该间隔发送静默 `{type:'keep_alive'}` 帧，避免上游代理的 idle timeout。

**状态 / metadata 监听：** 绑定 `setSessionStateChangedListener` 和 `setSessionMetadataChangedListener`，把 `SessionState` 变化通过 `reportState()` 和 `reportMetadata()` 传给 CCRClient。

**命令生命周期：** 绑定 `setCommandLifecycleListener`，在命令开始时触发 `reportDelivery('processing')`，命令结束时触发 `reportDelivery('processed')`。

### cli/structuredIO.ts - StructuredIO 类

```typescript
class StructuredIO {
  constructor(inputStream: Readable, replayUserMessages?)
  // 处理 SDK control 消息解析（control_request / control_response）
  // 通过 can_use_tool 协议做权限处理
  // 支持 elicitation dialog
  // 集成 hook 系统
}

const SANDBOX_NETWORK_ACCESS_TOOL_NAME = 'SandboxNetworkAccess'
```

`StructuredIO` 提供以下基础能力：
- 从 stdin 反序列化 NDJSON 消息
- 处理 `can_use_tool` 权限请求
- `control_response` 分发
- elicitation dialog 流程（`SDKControlElicitationResponseSchema`）
- 在权限决策前执行 hooks

---

## 22. 远程会话系统

### remote/SessionsWebSocket.ts - SessionsWebSocket

通过 `/v1/sessions/ws/{id}/subscribe` 查看 session 的 WebSocket 客户端。

**连接协议：**
1. 连接到 `wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe?organization_uuid={orgUuid}`
2. 发送 auth 消息：
   ```json
   { "type": "auth", "credential": { "type": "oauth", "token": "<accessToken>" } }
   ```
3. 接收 `SessionsMessage` 流。

**配置：**
```typescript
RECONNECT_DELAY_MS = 2000
MAX_RECONNECT_ATTEMPTS = 5
PING_INTERVAL_MS = 30000
MAX_SESSION_NOT_FOUND_RETRIES = 3   // 4001 在压缩期间可能是暂时性的
PERMANENT_CLOSE_CODES = { 4003 }    // unauthorized - 停止重连
```

**说明：** `4001`（session not found）会单独处理，并最多重试 3 次，因为压缩时可能会短暂返回错误的 “not found”。

**回调：**
```typescript
type SessionsWebSocketCallbacks = {
  onMessage: (message: SessionsMessage) => void
  onClose?: () => void          // 只针对永久关闭
  onError?: (error: Error) => void
  onConnected?: () => void
  onReconnecting?: () => void   // 临时断开并已安排重连
}
```

**接受的消息类型：** 任何带字符串 `type` 字段的对象（保持开放，避免漏掉新的 server 消息类型）。

### remote/RemoteSessionManager.ts - RemoteSessionManager

协调 WebSocket 订阅 + HTTP POST + 权限流程。

```typescript
type RemoteSessionConfig = {
  sessionId: string
  getAccessToken: () => string
  orgUuid: string
  hasInitialPrompt?: boolean
  viewerOnly?: boolean    // 纯查看者：不能 interrupt、不能更新标题、没有 60s 重连超时
}

type RemotePermissionResponse =
  | { behavior: 'allow'; updatedInput: Record<string, unknown> }
  | { behavior: 'deny'; message: string }
```

**回调：**
```typescript
type RemoteSessionCallbacks = {
  onMessage: (message: SDKMessage) => void
  onPermissionRequest: (request, requestId) => void
  onPermissionCancelled?: (requestId, toolUseId) => void
  onConnected?: () => void
  onDisconnected?: () => void
  onReconnecting?: () => void
  onError?: (error: Error) => void
}
```

### remote/remotePermissionBridge.ts

```typescript
createSyntheticAssistantMessage(
  request: SDKControlPermissionRequest,
  requestId: string,
): AssistantMessage
```

创建一个 synthetic `AssistantMessage`，包裹远程 `tool_use`，用于权限对话框。会使用假的 message ID `remote-{requestId}`，并把 usage stats 置空。

```typescript
createToolStub(toolName: string): Tool
```

为本地 CLI 不认识的工具（例如跑在 CCR 上的 MCP 工具）创建一个最小 `Tool` stub。会路由到 `FallbackPermissionRequest`。这个 stub 的 `renderToolUseMessage()` 最多展示 3 个输入 key-value 对。

### remote/sdkMessageAdapter.ts

把 CCR 的 `SDKMessage` 转成 REPL 的 `Message` 类型。

```typescript
type ConvertedMessage =
  | { type: 'message'; message: Message }
  | { type: 'stream_event'; event: StreamEvent }
  | { type: 'ignored' }

type ConvertOptions = {
  convertToolResults?: boolean          // direct-connect 模式
  convertUserTextMessages?: boolean     // 历史事件转换
}

convertSDKMessage(msg: SDKMessage, opts?: ConvertOptions): ConvertedMessage
isSessionEndMessage(msg: SDKMessage): boolean     // msg.type === 'result'
isSuccessResult(msg: SDKResultMessage): boolean   // msg.subtype === 'success'
getResultText(msg: SDKResultMessage): string | null
```

**转换规则：**

| SDKMessage 类型 | 转成什么 |
|-----------------|-------------|
| `assistant` | `AssistantMessage` |
| `user`（tool_result，convertToolResults=true） | `UserMessage` |
| `user`（text，convertUserTextMessages=true） | `UserMessage` |
| `user`（其他） | `ignored` |
| `stream_event` | `StreamEvent` |
| `result`（success） | `ignored` |
| `result`（error） | `SystemMessage`（warning） |
| `system`（init） | `SystemMessage`（info）：“Remote session initialized (model: ...)” |
| `system`（status: compacting） | `SystemMessage`（info）：“Compacting conversation…” |
| `system`（compact_boundary） | `SystemMessage`（compact_boundary） |
| `tool_progress` | `SystemMessage`（info）：“Tool {name} running for {n}s…” |
| `auth_status`、`tool_use_summary`、`rate_limit_event` | `ignored` |
| unknown | `ignored`（会记录日志） |

---

## 23. replLauncher.tsx

**文件：** `src/replLauncher.tsx`

```typescript
type AppWrapperProps = {
  getFpsMetrics: () => FpsMetrics | undefined
  stats?: StatsStore
  initialState: AppState
}

launchRepl(
  root: Root,
  appProps: AppWrapperProps,
  replProps: REPLProps,
  renderAndRun: (root: Root, element: React.ReactNode) => Promise<void>,
): Promise<void>
```

会懒加载 `App` 和 `REPL` 组件，并把它们包进 React 树里渲染。`main.tsx` 会用它来延迟加载沉重的 UI 组件树（App.js + REPL.js），直到真的需要交互模式时才加载。

---

## 24. 配置默认值与 GrowthBook 标志

### Bridge 相关 GrowthBook 特性开关

| 标志 | 类型 | 默认值 | 用途 |
|------|------|---------|---------|
| `tengu_ccr_bridge` | boolean | `false` | 主开关：启用 Remote Control |
| `tengu_bridge_repl_v2` | boolean | `false` | 启用无环境（V2）REPL bridge |
| `tengu_bridge_repl_v2_cse_shim_enabled` | boolean | `true` | `cse_*` → `session_*` 重标记 shim |
| `tengu_bridge_min_version` | DynamicConfig `{minVersion}` | `'0.0.0'` | V1 bridge 的最低 CLI 版本 |
| `tengu_bridge_repl_v2_config` | DynamicConfig (EnvLessBridgeConfig) | 见默认值 | V2 bridge 时序配置 |
| `tengu_bridge_poll_interval_config` | DynamicConfig (PollIntervalConfig) | 见默认值 | 轮询间隔 |
| `tengu_ccr_bridge_multi_session` | boolean | N/A | 启用多 session 启动模式 |
| `tengu_sessions_elevated_auth_enforcement` | boolean | `false` | 启用 trusted device 要求 |
| `tengu_cobalt_harbor` | boolean | `false` | 启动时自动连接 CCR |
| `tengu_ccr_mirror` | boolean | `false` | CCR mirror 模式 |

### 构建标志（bun:bundle 特性）

| 标志 | 用途 |
|------|---------|
| `BRIDGE_MODE` | 启用桥接相关代码路径 |
| `CCR_AUTO_CONNECT` | 启用 `getCcrAutoConnectDefault()` |
| `CCR_MIRROR` | 启用 `isCcrMirrorEnabled()` |
| `BASH_CLASSIFIER` | 启用 bash classifier 理由序列化 |
| `TRANSCRIPT_CLASSIFIER` | 启用 transcript classifier 理由序列化 |

### 环境变量（Bridge）

| 变量 | 用途 |
|----------|---------|
| `CLAUDE_BRIDGE_OAUTH_TOKEN` | 仅 Ant：覆盖 bridge 的 OAuth token |
| `CLAUDE_BRIDGE_BASE_URL` | 仅 Ant：覆盖 API base URL |
| `CLAUDE_TRUSTED_DEVICE_TOKEN` | 覆盖环境里的 trusted device token |
| `CLAUDE_CODE_USE_CCR_V2` | 对所有 session 使用 SSETransport + CCRClient |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | 使用 HybridTransport（WS 读 + POST 写） |
| `CLAUDE_CODE_CCR_MIRROR` | 启用 CCR mirror 模式 |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | session ingress 认证 token |
| `CLAUDE_CODE_ENVIRONMENT_RUNNER_VERSION` | 作为 `x-environment-runner-version` header 发送 |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | 通过 token exchange 走登录快路径 |
| `CLAUDE_CODE_OAUTH_SCOPES` | 当设置了 `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` 时必需 |

---

## 25. 完整 API 端点参考

### Environments API（`/v1/environments/bridge`）

所有环境 API 调用都要求：
- `anthropic-version: 2023-06-01`
- `anthropic-beta: environments-2025-11-01`
- `x-environment-runner-version: <version>`
- `Authorization: Bearer <token>`
- 可选：`X-Trusted-Device-Token: <token>`

#### POST `/v1/environments/bridge`

注册 bridge environment。

**请求：**
```json
{
  "machine_name": "hostname",
  "directory": "/path/to/dir",
  "branch": "main",
  "git_repo_url": "https://github.com/owner/repo",
  "max_sessions": 4,
  "metadata": { "worker_type": "claude_code" },
  "environment_id": "<backend-issued-id>"   // 可选：用于重新注册
}
```

**响应：**
```json
{
  "environment_id": "env_abc123",
  "environment_secret": "secret_xyz"
}
```

**超时：** 15s

#### GET `/v1/environments/{environmentId}/work/poll`

轮询新 work。认证：`environmentSecret`。

**查询参数：**
- `reclaim_older_than_ms`（可选）：回收比这个时间更老、还没被 ack 的 work

**响应：** `WorkResponse | null`（null = 没有可用 work）

**超时：** 10s

#### POST `/v1/environments/{environmentId}/work/{workId}/ack`

确认一个 work item。认证：`sessionToken`（session ingress JWT）。

**超时：** 10s

#### POST `/v1/environments/{environmentId}/work/{workId}/heartbeat`

发送 heartbeat。认证：`sessionToken`（session ingress JWT）。

**响应：**
```json
{
  "lease_extended": true,
  "state": "running",
  "last_heartbeat": "2025-03-31T...",
  "ttl_seconds": 300
}
```

**超时：** 10s

#### POST `/v1/environments/{environmentId}/work/{workId}/stop`

停止一个 work item。认证：OAuth token。

**请求：** `{ "force": true|false }`

**超时：** 10s

#### DELETE `/v1/environments/bridge/{environmentId}`

注销 environment。认证：OAuth token。

**超时：** 10s

#### POST `/v1/environments/{environmentId}/bridge/reconnect`

强制 session 重新分发。认证：OAuth token。

**请求：** `{ "session_id": "cse_abc123" }`

**超时：** 10s

### Sessions API（`/v1/sessions`）

所有调用都需要 `anthropic-beta: ccr-byoc-2025-07-29` 和 `x-organization-uuid: <orgUuid>`。

#### POST `/v1/sessions`

创建 session。

**请求：**
```json
{
  "title": "My Session",
  "events": [{ "type": "event", "data": <SDKMessage> }],
  "session_context": {
    "sources": [{ "type": "git_repository", "url": "...", "revision": "main" }],
    "outcomes": [...],
    "model": "claude-opus-4"
  },
  "environment_id": "env_abc123",
  "source": "remote-control",
  "permission_mode": "auto"
}
```

**响应：** `{ "id": "session_abc123" }`

#### GET `/v1/sessions/{sessionId}`

获取 session 元数据。返回 `{ environment_id?, title? }`。

#### PATCH `/v1/sessions/{sessionId}`

更新 session 标题。**请求：** `{ "title": "New Title" }`

#### POST `/v1/sessions/{sessionId}/archive`

归档 session。如果已经归档，会返回 `409`（幂等）。

#### POST `/v1/sessions/{sessionId}/events`

给 session 发送事件（用于权限响应）。认证：session ingress token。

**请求：**
```json
{
  "events": [
    {
      "type": "control_response",
      "response": {
        "subtype": "success",
        "request_id": "req_abc",
        "response": { "behavior": "allow" }
      }
    }
  ]
}
```

### CCR V2 Code Sessions API

#### POST `/v1/code/sessions`

**请求：**
```json
{
  "title": "Bridge Session",
  "bridge": {},
  "tags": ["optional", "tags"]
}
```

**响应：** `{ "session": { "id": "cse_abc123" } }`

#### POST `/v1/code/sessions/{sessionId}/bridge`

注册为 bridge worker 并拿 JWT。

**可选头：** `X-Trusted-Device-Token`

**响应：**
```json
{
  "worker_jwt": "sk-ant-si-...",
  "api_base_url": "https://...",
  "expires_in": 18000,
  "worker_epoch": "42"
}
```

**说明：** 每次调用都会 bump `worker_epoch` - 这个调用本身就是注册。

#### POST `/v1/code/sessions/{sessionId}/worker/register`

注册为 CCR worker（V1 CCR v2 路径）。返回 `{ "worker_epoch": "42" }`。

#### GET `/v1/code/sessions/{sessionId}/worker/events/stream`

用于接收入站事件的 SSE 流。重连时会发送 `Last-Event-ID` 或 `from_sequence_num`。

**SSE frame 格式：**
```text
event: sdk_event
id: 42
data: {"event_id":"evt_abc","payload":{...}}

:keepalive

```

#### POST `/v1/code/sessions/{sessionId}/worker/events`

向 session 发送事件。认证：worker JWT。

#### PUT `/v1/code/sessions/{sessionId}/worker`

更新 worker state。

**请求：**
```json
{
  "worker_status": "running" | "requires_action" | "completed",
  "external_metadata": { "key": "value" },
  "internal_metadata": { "key": "value" }
}
```

**metadata 合并：** RFC 7396 - key 会被新增 / 覆盖，`null` 值表示服务端删除。

#### POST `/v1/code/sessions/{sessionId}/worker/events/{eventId}/delivery`

上报事件 delivery 状态。**请求：** `{ "status": "received" | "processing" | "processed" }`

### 认证 / 受信设备

#### POST `/api/auth/trusted_devices`

给 elevated security session 注册设备。

**请求：**
```json
{ "display_name": "Claude Code on hostname · darwin" }
```

**响应：**
```json
{ "device_token": "...", "device_id": "..." }
```

**门槛：** 必须在登录后 10 分钟内调用。

### 文件附件

#### GET `/api/oauth/files/{fileUuid}/content`

下载附件内容。认证：OAuth token。返回二进制文件数据。

### Sessions WebSocket（远程查看器）

#### WS `/v1/sessions/ws/{sessionId}/subscribe?organization_uuid={orgUuid}`

以 viewer 身份连接。连接后发送：
```json
{ "type": "auth", "credential": { "type": "oauth", "token": "<accessToken>" } }
```

---

## 26. WebSocket 与 SSE 协议参考

### Session-Ingress WebSocket 协议

**URL：** `wss://api.anthropic.com/v1/session_ingress/ws/{sessionId}`

**V1 读写：** 双向消息都是 NDJSON（`StdoutMessage` / `StdinMessage`）。

**永久关闭码**（客户端停止重试）：
- `1002` - 协议错误
- `4001` - session 过期 / 未找到
- `4003` - 未授权

**Keep-alive 帧：** `{"type":"keep_alive"}` - 客户端会按 `DEFAULT_KEEPALIVE_INTERVAL`（5 分钟）发送。

**Ping/pong：** 客户端每 10 秒发一次 ping，10 秒内必须收到 pong。pong 超时后会重建连接。

### SSE Event 格式

```text
event: sdk_event
id: <sequence_number>
data: <json_payload>

:keepalive

```

- `id` 字段：单调递增的 sequence number，用于 resume（`Last-Event-ID`）
- `event` 字段：事件类型（`sdk_event`、`keep_alive` 等）
- `data` 字段：带 `event_id` 和 `payload` 的 JSON 对象

### Control Message 协议

**SDKControlRequest**（server → client）：
```json
{
  "type": "control_request",
  "request_id": "req_abc",
  "request": {
    "subtype": "initialize" | "set_model" | "set_max_thinking_tokens" | "set_permission_mode" | "interrupt" | "can_use_tool",
    ...subtype-specific fields...
  }
}
```

**SDKControlResponse**（client → server）：
```json
{
  "type": "control_response",
  "session_id": "session_abc",
  "response": {
    "subtype": "success" | "error",
    "request_id": "req_abc",
    ...response payload...
  }
}
```

**SDKControlCancelRequest**（server → client）：
```json
{
  "type": "control_cancel_request",
  "request_id": "req_abc"
}
```

### 权限请求流程

1. 子 CLI 在 stdout 上发出 `control_request`，`subtype: 'can_use_tool'`。
2. Bridge 通过 `onPermissionRequest` 回调收到它。
3. Bridge 通过 `POST /v1/sessions/{id}/events` 把它转给 server（permission response event）。
4. Claude.ai 展示审批对话框。
5. 用户同意或拒绝。
6. Server 通过 WebSocket 把 `control_response` 发回来。
7. Bridge 调用 `onPermissionResponse` 回调。
8. 子 CLI 在 stdin 收到决策并继续。

**权限响应载荷：**
```json
{
  "behavior": "allow" | "deny",
  "updatedInput": {...},          // 可选：修改后的工具输入
  "updatedPermissions": [...],    // 可选：新的权限规则
  "message": "..."                // 可选：拒绝消息
}
```

### NDJSON 安全

所有 NDJSON 格式消息（子进程 stdio 用到）都必须把 `U+2028` 和 `U+2029` 转义成 `\u2028` / `\u2029`（见 `ndjsonSafeStringify`）。它们虽然是合法 JSON，但在 JavaScript 里是行终止符，会把按行拆分的接收端弄坏。

---

*本文档基于 Claude Code 代码库源码分析生成，2026-03-31。*
