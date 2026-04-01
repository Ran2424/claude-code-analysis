# Claude Code — 服务、上下文、状态与屏幕

本文完整覆盖 `src/services/`、`src/context/`、`src/bootstrap/state.ts`、`src/coordinator/`、`src/server/` 和 `src/screens/` 下的所有文件。每个导出符号都会列出完整签名、核心逻辑、配置和依赖。

---

## 目录

1. [bootstrap/state.ts](#bootstrapstatets)
2. [coordinator/coordinatorMode.ts](#coordinatorcoordinatormodets)
3. [server/types.ts](#servertypets)
4. [server/createDirectConnectSession.ts](#servercreatedirectconnectsessionts)
5. [server/directConnectManager.ts](#serverdirectconnectmanagerts)
6. [services/analytics/config.ts](#servicesanalyticsconfigts)
7. [services/analytics/growthbook.ts](#servicesanalyticsgrowthbookts)
8. [services/analytics/metadata.ts](#servicesanalyticsmetadatats)
9. [services/analytics/index.ts](#servicesanalyticsindexts)
10. [services/analytics/sink.ts](#servicesanalyticssinkts)
11. [services/analytics/sinkKillswitch.ts](#servicesanalyticssinkkillswitchts)
12. [services/analytics/datadog.ts](#servicesanalyticsdatadogts)
13. [services/analytics/firstPartyEventLogger.ts](#servicesanalyticsfirstpartyeventloggerts)
14. [services/analytics/firstPartyEventLoggingExporter.ts](#servicesanalyticsfirstpartyeventloggingexporterts)
15. [services/api/bootstrap.ts](#servicesapibootstrapts)
16. [services/api/client.ts](#servicesapiclientts)
17. [services/api/claude.ts](#servicesapicludets)
18. [services/api/dumpPrompts.ts](#servicesapidumppromptsts)
19. [services/api/emptyUsage.ts](#servicesapiemptyusagets)
20. [services/api/errorUtils.ts](#servicesapierrorutilsts)
21. [services/api/errors.ts](#servicesapierrorsts)
22. [services/api/filesApi.ts](#servicesapifilesapits)
23. [services/api/firstTokenDate.ts](#servicesapifirsttokendatets)
24. [services/api/grove.ts](#servicesapigrovet)
25. [services/api/logging.ts](#servicesapiloggingts)
26. [services/api/metricsOptOut.ts](#servicesapimetricsoptoutts)
27. [services/api/overageCreditGrant.ts](#servicesapiovaragecreditgrantts)
28. [services/api/promptCacheBreakDetection.ts](#servicesapipromptcachebreakdetectionts)
29. [services/api/referral.ts](#servicesapireferralts)
30. [services/api/sessionIngress.ts](#servicesapisessioningressts)
31. [services/api/ultrareviewQuota.ts](#servicesapiultrareviewquotats)
32. [services/api/usage.ts](#servicesapiusagets)
33. [services/api/withRetry.ts](#servicesapiwithretryts)
34. [services/AgentSummary/agentSummary.ts](#servicesagentsummaryagentsummaryts)
35. [services/autoDream/autoDream.ts](#servicesautodreamautodreamts)
36. [services/autoDream/config.ts](#servicesautodreamconfigts)
37. [services/autoDream/consolidationLock.ts](#servicesautodreamconsolidationlockts)
38. [services/autoDream/consolidationPrompt.ts](#servicesautodreamconsolidationpromptts)
39. [services/awaySummary.ts](#servicesawaysummaryts)
40. [services/claudeAiLimits.ts](#servicesclaudeailimitsts)
41. [services/claudeAiLimitsHook.ts](#servicesclaudeailimitshookts)
42. [services/compact/apiMicrocompact.ts](#servicescompactapimicrocompactts)
43. [services/compact/autoCompact.ts](#servicescompactautocompactts)
44. [services/compact/compact.ts](#servicescompactcompactts)
45. [services/compact/compactWarningHook.ts](#servicescompactcompactwarninghookts)
46. [services/compact/compactWarningState.ts](#servicescompactcompactwarningstatets)
47. [services/compact/grouping.ts](#servicescompactgroupingts)
48. [services/compact/microCompact.ts](#servicescompactmicrocompactts)
49. [services/compact/postCompactCleanup.ts](#servicescompactpostcompactcleanupts)
50. [services/compact/prompt.ts](#servicescompactpromptts)
51. [services/compact/sessionMemoryCompact.ts](#servicescompactsessionmemorycompactts)
52. [services/compact/timeBasedMCConfig.ts](#servicescompacttimebasedmcconfigts)
53. [services/diagnosticTracking.ts](#servicesdiagnostictrackingts)
54. [services/internalLogging.ts](#servicesinternalloggingts)
55. [services/MagicDocs/magicDocs.ts](#servicesmagicdocsmagicdocsts)
56. [services/MagicDocs/prompts.ts](#servicesmagicdocspromptsts)
57. [services/mcpServerApproval.tsx](#servicesmcpserverapprovaltsx)
58. [services/mockRateLimits.ts](#servicesmockratelimitsts)
59. [services/MCP (mcp/)](#services-mcp)
60. [services/notifier.ts](#servicesnotifierts)
61. [services/preventSleep.ts](#servicespreventsleepts)
62. [services/PromptSuggestion/promptSuggestion.ts](#servicespromptsuggestionpromptsuggestionts)
63. [services/PromptSuggestion/speculation.ts](#servicespromptsuggestionspeculationts)
64. [services/rateLimitMocking.ts](#servicesratelimitmockingts)
65. [services/rateLimitMessages.ts](#servicesratelimitmessagests)
66. [services/SessionMemory/prompts.ts](#servicessessionmemorypromptsts)
67. [services/SessionMemory/sessionMemory.ts](#servicessessionmemorysessionmemoryts)
68. [services/SessionMemory/sessionMemoryUtils.ts](#servicessessionmemorysessionmemoryutilsts)
69. [services/tokenEstimation.ts](#servicestokenestimationts)
70. [services/vcr.ts](#servicesvcrts)
71. [services/voice.ts](#servicesvoicets)
72. [services/voiceKeyterms.ts](#servicesvoicekeytermsts)
73. [services/voiceStreamSTT.ts](#servicesvoicestreamsttts)
74. [context/QueuedMessageContext.tsx](#contextqueuedmessagecontexttsx)
75. [context/fpsMetrics.tsx](#contextfpsmetricstsx)
76. [context/mailbox.tsx](#contextmailboxtsx)
77. [context/modalContext.tsx](#contextmodalcontexttsx)
78. [context/notifications.tsx](#contextnotificationstsx)
79. [context/overlayContext.tsx](#contextoverlaycontexttsx)
80. [context/promptOverlayContext.tsx](#contextpromptoverlaycontexttsx)
81. [context/stats.tsx](#contextstatstsx)
82. [context/voice.tsx](#contextvoicetsx)
83. [screens/Doctor.tsx](#screensdoctortsx)
84. [screens/REPL.tsx](#screensrepltsx)
85. [screens/ResumeConversation.tsx](#screensresumeconversationtsx)

---

## bootstrap/state.ts

**路径：** `src/bootstrap/state.ts`

**用途：** 整个 Claude Code 进程唯一的全局会话状态单例。它是每个会话的指标、模型配置、遥测句柄和功能开关的权威来源。它被设计成依赖图中的严格叶子节点，除非通过显式的安全间接层，否则不会从 `src/utils/` 里引入任何内容。

**导出的关键类型：**

```typescript
export type ChannelEntry =
  | { kind: 'plugin'; name: string; marketplace: string; dev?: boolean }
  | { kind: 'server'; name: string; dev?: boolean }

export type AttributedCounter = {
  add(value: number, additionalAttributes?: Attributes): void
}
```

**`State` 类型（内部，不直接导出）：** 这个对象包含大约 80 个字段，包括：
- `originalCwd: string` - 进程启动时解析后的 cwd（NFC 归一化，已解析符号链接）
- `projectRoot: string` - 稳定的身份根目录（启动时设定，后续即使 `EnterWorktreeTool` 也不会改）
- `totalCostUSD: number`, `totalAPIDuration: number`, `totalAPIDurationWithoutRetries: number`
- `totalToolDuration: number`, `turnHookDurationMs: number`, `turnToolDurationMs: number`
- `totalLinesAdded: number`, `totalLinesRemoved: number`
- `cwd: string` - 当前工作目录（可变，随 shell.ts 的 setCwd 改变）
- `modelUsage: { [modelName: string]: ModelUsage }` - 按模型统计使用情况
- `mainLoopModelOverride: ModelSetting | undefined`, `initialMainLoopModel: ModelSetting`
- `sessionId: SessionId` - 在 `clearConversation` 时重新生成的 UUID
- `parentSessionId: SessionId | undefined` - 上一个会话，用于血缘追踪
- `isInteractive: boolean`, `kairosActive: boolean`, `strictToolResultPairing: boolean`
- `sdkAgentProgressSummariesEnabled: boolean`, `userMsgOptIn: boolean`
- `clientType: string`（默认 `'cli'`）、`sessionSource: string | undefined`
- `meter: Meter | null`, `sessionCounter`, `locCounter`, `prCounter`, `commitCounter`, `costCounter`, `tokenCounter`, `codeEditToolDecisionCounter`, `activeTimeCounter` - OTel 指标
- `sessionId: SessionId`（初始化时 `randomUUID`）、`parentSessionId: SessionId | undefined`
- `loggerProvider: LoggerProvider | null`, `eventLogger: ReturnType<typeof logs.getLogger> | null`
- `meterProvider: MeterProvider | null`, `tracerProvider: BasicTracerProvider | null`
- `agentColorMap: Map<string, AgentColorName>`, `agentColorIndex: number`
- `lastAPIRequest`, `lastAPIRequestMessages`, `lastClassifierRequests`, `cachedClaudeMdContent`
- `inMemoryErrorLog: Array<{ error: string; timestamp: string }>`
- `inlinePlugins: string[]`, `chromeFlagOverride: boolean | undefined`
- `sessionBypassPermissionsMode: boolean`, `scheduledTasksEnabled: boolean`
- `sessionCronTasks: SessionCronTask[]`, `sessionCreatedTeams: Set<string>`
- `sessionTrustAccepted: boolean`, `sessionPersistenceDisabled: boolean`
- `hasExitedPlanMode: boolean`, `needsPlanModeExitAttachment: boolean`, `needsAutoModeExitAttachment: boolean`
- `initJsonSchema: Record<string, unknown> | null`, `registeredHooks: Partial<Record<HookEvent, RegisteredHookMatcher[]>> | null`
- `planSlugCache: Map<string, string>` - sessionId → wordSlug
- `teleportedSessionInfo: { isTeleported, hasLoggedFirstMessage, sessionId } | null`
- `invokedSkills: Map<string, { skillName, skillPath, content, invokedAt, agentId }>` - 以 `"${agentId ?? ''}:${skillName}"` 为键
- `slowOperations: Array<{ operation, durationMs, timestamp }>` - 仅 ant 的开发状态栏
- `sdkBetas: string[] | undefined`, `mainThreadAgentType: string | undefined`
- `isRemoteMode: boolean`, `directConnectServerUrl: string | undefined`
- `systemPromptSectionCache: Map<string, string | null>`, `lastEmittedDate: string | null`
- `additionalDirectoriesForClaudeMd: string[]`, `allowedChannels: ChannelEntry[]`, `hasDevChannels: boolean`
- `sessionProjectDir: string | null` - transcript 目录覆盖
- `promptCache1hAllowlist: string[] | null`, `promptCache1hEligible: boolean | null`
- `afkModeHeaderLatched: boolean | null`, `fastModeHeaderLatched: boolean | null`
- `cacheEditingHeaderLatched: boolean | null`, `thinkingClearLatched: boolean | null`
- `promptId: string | null`, `lastMainRequestId: string | undefined`
- `lastApiCompletionTimestamp: number | null`, `pendingPostCompaction: boolean`

**导出的函数（getter/setter/mutator）：**

```typescript
export function getSessionId(): SessionId
export function regenerateSessionId(options?: { setCurrentAsParent?: boolean }): SessionId
export function getParentSessionId(): SessionId | undefined
export function switchSession(sessionId: SessionId, projectDir?: string | null): void
export const onSessionSwitch: Signal<[id: SessionId]>['subscribe']
export function getSessionProjectDir(): string | null
export function getOriginalCwd(): string
export function getProjectRoot(): string
export function setOriginalCwd(cwd: string): void
export function setProjectRoot(cwd: string): void  // --worktree 启动时专用
export function getCwdState(): string
export function setCwdState(cwd: string): void
export function getDirectConnectServerUrl(): string | undefined
export function setDirectConnectServerUrl(url: string): void
export function addToTotalDurationState(duration: number, durationWithoutRetries: number): void
export function resetTotalDurationStateAndCost_FOR_TESTS_ONLY(): void
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
export function getStatsStore(): { observe(name: string, value: number): void } | null
export function setStatsStore(store: { observe(name: string, value: number): void } | null): void
export function updateLastInteractionTime(immediate?: boolean): void
export function flushInteractionTime(): void
export function addToTotalLinesChanged(added: number, removed: number): void
export function getTotalLinesAdded(): number
export function getTotalLinesRemoved(): number
export function getTotalInputTokens(): number
export function getTotalOutputTokens(): number
export function getTotalCacheReadInputTokens(): number
export function getTotalCacheCreationInputTokens(): number
export function getTotalWebSearchRequests(): number
export function getTurnOutputTokens(): number
export function getCurrentTurnTokenBudget(): number | null
export function snapshotOutputTokensForTurn(budget: number | null): void
export function getBudgetContinuationCount(): number
export function incrementBudgetContinuationCount(): void
export function setHasUnknownModelCost(): void
export function hasUnknownModelCost(): boolean
export function getLastMainRequestId(): string | undefined
export function setLastMainRequestId(requestId: string): void
export function getLastApiCompletionTimestamp(): number | null
export function setLastApiCompletionTimestamp(timestamp: number): void
export function markPostCompaction(): void
export function consumePostCompaction(): boolean
export function getLastInteractionTime(): number
export function markScrollActivity(): void
export function getIsScrollDraining(): boolean
export function waitForScrollIdle(): Promise<void>
export function getModelUsage(): { [modelName: string]: ModelUsage }
export function getUsageForModel(model: string): ModelUsage | undefined
export function getMainLoopModelOverride(): ModelSetting | undefined
export function getInitialMainLoopModel(): ModelSetting
export function setMainLoopModelOverride(model: ModelSetting | undefined): void
// ... 还有很多针对 isInteractive、clientType、sessionSource、遥测计数器等的 setter
```

**核心逻辑：**
- `STATE` 是一个模块级单例，在导入时通过 `getInitialState()` 初始化
- `updateLastInteractionTime(immediate?)`：默认延迟执行，把按键事件合并成每次 Ink 渲染只记录一次 `Date.now()`；在渲染后的 `useEffect` 回调里传 `immediate=true`
- `flushInteractionTime()`：Ink 在每次渲染循环前调用
- 滚动排空：`markScrollActivity()` 会设置一个 150ms 的防抖标志，后台轮询会看 `getIsScrollDraining()` 并让出执行；`waitForScrollIdle()` 以 150ms 间隔轮询
- `switchSession()` 会原子地更新 `sessionId + sessionProjectDir`；并发出 `sessionSwitched` 信号
- `regenerateSessionId()` 可以选择把当前会话设为父会话（用于 plan mode → implementation 的血缘链）
- `markPostCompaction()` / `consumePostCompaction()` 是一次性 latch，第一次消费后自动重置

**配置：**
- `SCROLL_DRAIN_IDLE_MS = 150`
- `RESERVOIR_SIZE`（直方图采样）= 1024（在 `stats.tsx` 中）

**依赖：** `@anthropic-ai/sdk`、`@opentelemetry/api`、`@opentelemetry/sdk-*`、`src/utils/crypto.js`、`src/utils/signal.js`、`src/utils/settings/settingsCache.js`、`src/types/ids.js`

---

## coordinator/coordinatorMode.ts

**路径：** `src/coordinator/coordinatorMode.ts`

**用途：** 实现多 worker 的“协调器模式”，让 Claude Code 编排多个并行 subagent。它提供系统提示词、用户上下文注入、模式检测，以及会话恢复时的对齐逻辑。

**导出：**

```typescript
export function isCoordinatorMode(): boolean
export function matchSessionMode(
  sessionMode: 'coordinator' | 'normal' | undefined
): string | undefined
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string
): { [k: string]: string }
export function getCoordinatorSystemPrompt(): string
```

**核心逻辑：**
- `isCoordinatorMode()`：读取 `CLAUDE_CODE_COORDINATOR_MODE` 环境变量；只有 bundle 标志 `feature('COORDINATOR_MODE')` 打开时才生效
- `matchSessionMode()`：恢复会话时，把当前 coordinator mode 和存档中的 session mode 对齐。它会原地修改 `process.env.CLAUDE_CODE_COORDINATOR_MODE`（因为 `isCoordinatorMode()` 是实时读取它的）。如果模式被切换，会返回用户可见的警告信息；如果无需修改，则返回 `undefined`。同时会记录 `tengu_coordinator_mode_switched` 分析事件
- `getCoordinatorUserContext()`：返回 `{ workerToolsContext: string }`，其中包含 worker 工具列表、MCP server 名称，以及 scratchpad 目录（如果 GrowthBook gate `tengu_scratch` 打开）。在 `CLAUDE_CODE_SIMPLE` 模式下，只允许 Bash/Read/Edit
- `getCoordinatorSystemPrompt()`：返回一个多段系统提示词（1500+ 字符），说明协调器角色、可用工具（Agent、SendMessage、TaskStop）、任务工作流阶段（Research → Synthesis → Implementation → Verification）、并发策略、worker prompt 编写指南，以及完整示例会话

**内部常量：**
```typescript
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
```

**配置：**
- `COORDINATOR_MODE` bundle feature flag
- `CLAUDE_CODE_COORDINATOR_MODE` 环境变量
- `CLAUDE_CODE_SIMPLE` 环境变量 - 把 worker 工具集限制为 Bash/Read/Edit
- GrowthBook gate `tengu_scratch` - 启用 scratchpad 目录上下文

**依赖：** `bun:bundle`、`constants/tools.js`、`services/analytics/growthbook.js`、`services/analytics/index.js`、各类工具名常量、`utils/envUtils.js`

---

## server/types.ts

**路径：** `src/server/types.ts`

**用途：** Claude Code server（direct-connect 模式）的共享类型定义。提供会话创建响应的 Zod 校验模式。

**导出：**

```typescript
export const connectResponseSchema: () => ZodObject<{
  session_id: ZodString
  ws_url: ZodString
  work_dir: ZodString.optional()
}>

export type ServerConfig = {
  port: number
  host?: string
  authToken?: string
}

export type SessionState = 'starting' | 'running' | 'detached' | 'stopping' | 'stopped'

export type SessionInfo = {
  sessionId: string
  state: SessionState
  wsUrl: string
  workDir?: string
  createdAt: number
  lastActivity: number
}

export type SessionIndexEntry = {
  sessionId: string
  createdAt: number
  workDir?: string
}

export type SessionIndex = Record<string, SessionIndexEntry>
```

**核心逻辑：** `connectResponseSchema()` 是工厂函数，不是缓存值，这样 Zod 可以延迟加载。它由 `createDirectConnectSession.ts` 用来校验 `POST /sessions` 的响应体。

**依赖：** `zod`

---

## server/createDirectConnectSession.ts

**路径：** `src/server/createDirectConnectSession.ts`

**用途：** 在远程 direct-connect Claude Code server 上创建一个会话。它会向 `/sessions` 发起 POST 请求，校验响应，然后返回一个可直接交给 REPL 或 headless runner 使用的 `DirectConnectConfig`。

**导出：**

```typescript
export class DirectConnectError extends Error {
  constructor(message: string)
  name: 'DirectConnectError'
}

export async function createDirectConnectSession(opts: {
  serverUrl: string
  authToken?: string
  cwd: string
  dangerouslySkipPermissions?: boolean
}): Promise<{
  config: DirectConnectConfig
  workDir?: string
}>
```

**核心逻辑：**
- 向 `${serverUrl}/sessions` 发送 JSON：`{ cwd, dangerously_skip_permissions? }`
- 如果提供了 `authToken`，会加上 `Authorization: Bearer ${authToken}`
- 通过 `connectResponseSchema().safeParse()` 校验响应 JSON
- 返回 `{ config: { serverUrl, sessionId, wsUrl, authToken }, workDir }`
- 在 fetch 失败、HTTP 非 OK、或响应解析失败时抛出 `DirectConnectError`

**依赖：** `server/types.js`、`server/directConnectManager.js`、`utils/errors.js`、`utils/slowOperations.js`

---

## server/directConnectManager.ts

**路径：** `src/server/directConnectManager.ts`

**用途：** 与远程 direct-connect Claude Code server 通信的 WebSocket 客户端。负责消息路由、权限请求/响应、中断信号，以及连接生命周期。

**导出：**

```typescript
export type DirectConnectConfig = {
  serverUrl: string
  sessionId: string
  wsUrl: string
  authToken?: string
}

export type DirectConnectCallbacks = {
  onMessage: (message: SDKMessage) => void
  onPermissionRequest: (request: SDKControlPermissionRequest, requestId: string) => void
  onConnected?: () => void
  onDisconnected?: () => void
  onError?: (error: Error) => void
}

export class DirectConnectSessionManager {
  constructor(config: DirectConnectConfig, callbacks: DirectConnectCallbacks)
  connect(): void
  sendMessage(content: RemoteMessageContent): boolean
  respondToPermissionRequest(requestId: string, result: RemotePermissionResponse): void
  sendInterrupt(): void
  disconnect(): void
  isConnected(): boolean
}
```

**核心逻辑：**
- `connect()`：带 `Authorization: Bearer` 头打开 WebSocket（Bun WebSocket headers 会覆盖同名头）；并设置 `open`、`message`、`close`、`error` 监听器
- 消息解析：按 NDJSON 行拆分并逐行解析，然后分发：
  - `control_request` 且子类型为 `can_use_tool` → `onPermissionRequest()`
  - 未识别的 control 子类型 → 自动发送错误响应，避免 server 卡住
  - 被过滤掉的：`control_response`、`keep_alive`、`control_cancel_request`、`streamlined_text`、`streamlined_tool_use_summary`、以及 subtype 为 `post_turn_summary` 的 system 消息
  - 其他全部 → `onMessage()`
- `sendMessage()`：格式化为 `SDKUserMessage`（`{ type: 'user', message: { role: 'user', content }, parent_tool_use_id: null, session_id: '' }`）
- `respondToPermissionRequest()`：格式化为 `SDKControlResponse`，根据 allow/deny 写入 `behavior` 和 `updatedInput` 或 `message`
- `sendInterrupt()`：发送 `{ type: 'control_request', request_id: crypto.randomUUID(), request: { subtype: 'interrupt' } }`

**依赖：** `entrypoints/agentSdkTypes.js`、`entrypoints/sdk/controlTypes.js`、`remote/RemoteSessionManager.js`、`utils/debug.js`、`utils/slowOperations.js`、`utils/teleport/api.js`

---

## services/analytics/config.ts

**路径：** `src/services/analytics/config.ts`

**用途：** 共享的 analytics 配置，提供跨所有后端统一禁用 analytics 的逻辑。

**导出：**

```typescript
export function isAnalyticsDisabled(): boolean
export function isFeedbackSurveyDisabled(): boolean
```

**核心逻辑：**
- `isAnalyticsDisabled()`：当 `NODE_ENV === 'test'`、`CLAUDE_CODE_USE_BEDROCK`、`CLAUDE_CODE_USE_VERTEX`、`CLAUDE_CODE_USE_FOUNDRY` 为真，或者 `isTelemetryDisabled()` 返回 true 时，返回 `true`
- `isFeedbackSurveyDisabled()`：当 `NODE_ENV === 'test'` 或 `isTelemetryDisabled()` 时返回 `true`；它**不会**因为 3P 提供方（Bedrock/Vertex/Foundry）而禁用，因为 survey 是本地 UI，不会接触 transcript 数据；企业侧会通过 OTEL 采集

**依赖：** `utils/envUtils.js`、`utils/privacyLevel.js`

---

## services/analytics/growthbook.ts

**路径：** `src/services/analytics/growthbook.ts`

**用途：** GrowthBook feature flag 和动态配置客户端。它提供缓存和阻塞两种读取方式，处理带磁盘持久化的远程求值，管理刷新生命周期，并提供开发/测试用的覆盖 API。

**关键类型：**

```typescript
export type GrowthBookUserAttributes = {
  user_id?: string
  org_id?: string
  user_type?: string
  // ... 其他兼容 Statsig 的属性
}
```

**导出：**

```typescript
export function onGrowthBookRefresh(listener: () => void): () => void
export function hasGrowthBookEnvOverride(feature: string): boolean
export function getAllGrowthBookFeatures(): Record<string, unknown>
export function getGrowthBookConfigOverrides(): Record<string, unknown>
export function setGrowthBookConfigOverride(feature: string, value: unknown): void
export function clearGrowthBookConfigOverrides(): void
export function getApiBaseUrlHost(): string | undefined
export function initializeGrowthBook(): Promise<GrowthBook | null>
export function getFeatureValue_DEPRECATED<T>(feature: string, defaultValue: T): Promise<T>
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(feature: string, defaultValue: T): T
export function getFeatureValue_CACHED_WITH_REFRESH<T>(feature: string, defaultValue: T): T  // 已废弃
export function checkStatsigFeatureGate_CACHED_MAY_BE_STALE(gate: string): boolean
export function checkSecurityRestrictionGate(gate: string): Promise<boolean>
export function checkGate_CACHED_OR_BLOCKING(gate: string): Promise<boolean>
export function getDynamicConfig_CACHED_MAY_BE_STALE<T>(config: string, defaultValue: T): T
export function refreshGrowthBookAfterAuthChange(): void
```

**核心逻辑：**
- **初始化 (`initializeGrowthBook()`)：** 单例并带缓存。它会从 `~/.claude/cachedGrowthBookFeatures` 读取磁盘缓存的 feature，应用 `CLAUDE_INTERNAL_FC_OVERRIDES` 环境变量覆盖（JSON），再以 5000ms 超时连接 GrowthBook 远端，随后建立周期刷新。在 analytics 被禁用，或者 API key 模式下没有 `user_id` 时返回 `null`
- **远程求值兼容：** GrowthBook 的 remote eval 返回 `{ value }`，但客户端期望 `{ defaultValue }`。代码会把 `{ value: V }` 转成 `{ defaultValue: V }` 后再写入 `remoteEvalFeatureValues` Map，并通过 `syncRemoteEvalToDisk()` 同步到磁盘
- **缓存层级：**
  - `_DEPRECATED`：阻塞等待 `initializeGrowthBook()` Promise
  - `_CACHED_MAY_BE_STALE`：直接从内存缓存同步返回，可能在刷新后过期
  - `_CACHED_OR_BLOCKING`：先等待初始化，再返回缓存值；只用于安全 gate
- **安全 gate (`checkSecurityRestrictionGate()`)：** 会等待初始化并检查 gate 值，若未初始化则阻塞。用于企业策略执行
- **刷新监听：** `onGrowthBookRefresh()` 注册一个监听器，每次 GrowthBook 刷新后调用；返回取消订阅函数
- **覆盖：** `setGrowthBookConfigOverride()` / `clearGrowthBookConfigOverrides()` 提供进程内覆盖 map；`CLAUDE_INTERNAL_FC_OVERRIDES` JSON 环境变量提供进程级覆盖

**配置：**
- 磁盘缓存：`~/.claude/cachedGrowthBookFeatures`
- 初始化超时：5000ms
- `CLAUDE_INTERNAL_FC_OVERRIDES` 环境变量：JSON 覆盖 map

**依赖：** `growthbook` SDK、`services/analytics/config.js`、`utils/auth.js`、`utils/config.js`

---

## services/analytics/metadata.ts

**路径：** `src/services/analytics/metadata.ts`

**用途：** analytics 的事件元数据增强层。它提供用于构建结构化 `EventMetadata` 对象的类型和工具，其中包含环境上下文、进程指标，以及从工具输入中安全抽取的遥测数据。

**常量：**
- `TOOL_INPUT_STRING_TRUNCATE_AT = 512` - 超过这个长度的字符串会被截断
- `TOOL_INPUT_STRING_TRUNCATE_TO = 128` - 截断后的目标长度
- `TOOL_INPUT_MAX_JSON_CHARS = 4096` - JSON 输入上限，超过就丢弃
- `MAX_FILE_EXTENSION_LENGTH = 10` - 文件扩展名最大字符数

**导出：**

```typescript
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never  // 标记类型

export function sanitizeToolNameForAnalytics(toolName: string): never  // 返回标记类型

export function isToolDetailsLoggingEnabled(): boolean  // 由 OTEL_LOG_TOOL_DETAILS 环境变量控制

export function isAnalyticsToolDetailsLoggingEnabled(
  mcpServerType: string | undefined,
  mcpServerBaseUrl: string | undefined
): boolean

export function mcpToolDetailsForAnalytics(
  toolName: string,
  mcpServerType: string | undefined,
  mcpServerBaseUrl: string | undefined
): { mcpServerName?: never; mcpToolName?: never }

export function extractMcpToolDetails(
  toolName: string
): { serverName: string; mcpToolName: string } | undefined

export function extractSkillName(
  toolName: string,
  input: unknown
): never | undefined  // 返回标记类型或 undefined

export function extractToolInputForTelemetry(input: unknown): string | undefined

export function getFileExtensionForAnalytics(filePath: string): never | undefined

export function getFileExtensionsFromBashCommand(
  command: string,
  simulatedSedEditFilePath?: string
): never | undefined

export type EnvContext = {
  userType: string
  isCI: boolean
  platform: string
  // ... 其他上下文字段
}

export type ProcessMetrics = {
  heapUsedMB: number
  heapTotalMB: number
  rssMB: number
  externalMB: number
}

export type EventMetadata = {
  // 增强后的事件载荷类型
}

export type EnrichMetadataOptions = {
  includeProcessMetrics?: boolean
  // ...
}

export async function getEventMetadata(options?: EnrichMetadataOptions): Promise<EventMetadata>
export async function buildEnvContext(): Promise<EnvContext>  // memoized
```

**核心逻辑：**
- `BUILTIN_MCP_SERVER_NAMES` 集合受 `CHICAGO_MCP` feature flag 保护，用来判断哪些 MCP server 算“内建”
- `extractToolInputForTelemetry()`：先把输入 JSON 序列化，再把超过 `TOOL_INPUT_STRING_TRUNCATE_AT` 的字符串截断到 `TOOL_INPUT_STRING_TRUNCATE_TO`，并把整体 JSON 上限限制为 `TOOL_INPUT_MAX_JSON_CHARS`
- `getFileExtensionsFromBashCommand()`：用正则解析 bash 命令，抽取文件扩展名；`sed -i` 会通过 `simulatedSedEditFilePath` 特殊处理
- `buildEnvContext()` 是 memoized 的，每个进程只会调用一次并缓存结果
- Agent 身份会把 turn 分成三类：teammate（别的智能体的 subagent）、subagent（由 coordinator 生成）、standalone

**依赖：** `services/analytics/growthbook.js`、`utils/envUtils.js`、`utils/platform.js`

---

## services/analytics/index.ts

**路径：** `src/services/analytics/index.ts`

**用途：** analytics 的主入口模块。它本身没有依赖，提供一个队列式 facade 来承接所有事件日志。事件会一直排队，直到挂上 sink，避免启动顺序问题。

**设计：** 明确不引入任何依赖，以避免 import cycle。事件会先进入 `eventQueue`，等 `attachAnalyticsSink()` 通过 `queueMicrotask` 把它们刷出。

**导出：**

```typescript
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never  // 标记类型
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never  // PII 标记类型

export function stripProtoFields<V>(
  metadata: Record<string, V>
): Record<string, V>

export type AnalyticsSink = {
  logEvent: (eventName: string, metadata: LogEventMetadata) => void
  logEventAsync: (eventName: string, metadata: LogEventMetadata) => Promise<void>
}

export function attachAnalyticsSink(newSink: AnalyticsSink): void
export function logEvent(eventName: string, metadata: LogEventMetadata): void
export async function logEventAsync(eventName: string, metadata: LogEventMetadata): Promise<void>
export function _resetForTesting(): void
```

**内部类型：**
```typescript
type LogEventMetadata = { [key: string]: boolean | number | undefined }
type QueuedEvent = { eventName: string; metadata: LogEventMetadata; async: boolean }
```

**核心逻辑：**
- `attachAnalyticsSink()`：幂等，如果 sink 已经设置过就直接返回。它会通过 `queueMicrotask` 清空队列，避免阻塞启动。对 ant 用户（`USER_TYPE === 'ant'`），还会记录 `analytics_sink_attached` 和 `queued_event_count`
- `stripProtoFields()`：移除事件元数据里所有以 `_PROTO_` 开头的键（用于非 1P 目标）。如果没有 `_PROTO_` 键，会直接返回同一个引用
- `_PROTO_*` 键会路由到带 PII 标记的 BigQuery 列；在送到 Datadog 前会剥离，但会保留给 `firstPartyEventLoggingExporter`
- 元数据类型刻意限制为 `boolean | number | undefined`，除非显式转成标记类型，否则不允许字符串

---

## services/analytics/sink.ts

**路径：** `src/services/analytics/sink.ts`

**用途：** analytics sink 实现，把事件路由到 Datadog 和 first-party event logging 两个后端，同时处理采样、元数据增强和每个 sink 的 kill switch。

**导出：**

```typescript
export function createAnalyticsSink(options?: {
  isInteractive?: boolean
}): AnalyticsSink
```

**核心逻辑：**
- 事件会发往两个 sink：Datadog（通过 `logDatadogEvent`）和 first-party event logging（通过 `log1PEvent`）
- 在分发前检查 `isSinkKilled('datadog')` 和 `isSinkKilled('firstParty')`
- 采样由 `tengu_event_sampling_config` 动态配置控制；如果被采样，会把 `sample_rate` 写进 metadata
- 发往 Datadog 前会用 `stripProtoFields()` 去掉 `_PROTO_*` 键
- 分发前会先调用 `getEventMetadata()` 做 metadata 增强

---

## services/analytics/sinkKillswitch.ts

**路径：** `src/services/analytics/sinkKillswitch.ts`

**用途：** 单个 sink 的 analytics kill switch，由 GrowthBook JSON 配置控制。

**导出：**

```typescript
export type SinkName = 'datadog' | 'firstParty'

export function isSinkKilled(sink: SinkName): boolean
```

**核心逻辑：**
- 配置名：`tengu_frond_boric`（经过混淆/编码的名字）
- 结构：`{ datadog?: boolean, firstParty?: boolean }`，其中 `true` 表示停止向该 sink 发送
- 默认值是 `{}`，也就是不关闭任何 sink。fail-open：配置缺失或格式不对时，sink 仍保持开启
- **不能**在 `isGrowthBookEnabled()` 里调用，否则会递归；应该在每条事件的分发点调用

**依赖：** `services/analytics/growthbook.js`

---

## services/analytics/datadog.ts

**路径：** `src/services/analytics/datadog.ts`

**用途：** Claude Code analytics 的 Datadog 指标和事件日志集成。

**核心逻辑：**
- 通过 `@datadog/datadog-ci` 或直接 HTTP 调用 Datadog API 发送事件
- 当 `isAnalyticsDisabled()` 返回 true 时关闭
- 事件命名空间：所有 Claude Code 事件都使用 `tengu_*` 前缀
- 标签包含版本、平台、用户类型、会话元数据

---

## services/analytics/firstPartyEventLogger.ts

**路径：** `src/services/analytics/firstPartyEventLogger.ts`

**用途：** first-party event logging 集成，把事件路由到内部事件日志系统。

**核心逻辑：**
- 由 `is1PEventLoggingEnabled()` 控制；它会检查 GrowthBook feature `tengu_fpel` 和 `isAnalyticsDisabled()`
- 用 `getEventMetadata()` 产出的 `EventMetadata` 增强事件
- 处理来自 `_PROTO_*` 键的 proto 字段提升

---

## services/analytics/firstPartyEventLoggingExporter.ts

**路径：** `src/services/analytics/firstPartyEventLoggingExporter.ts`

**用途：** OpenTelemetry log exporter，把日志路由到 first-party event logging 流水线。

**核心逻辑：**
- 实现 OTel `LogRecordExporter` 接口
- 把 `_PROTO_*` 键提升到 BQ 目标中的顶层 proto 字段
- 在提升后再调用 `stripProtoFields()` 做防御性清理
- 只发送到 first-party 流水线，不发 Datadog

---

## services/api/bootstrap.ts

**路径：** `src/services/api/bootstrap.ts`

**用途：** 在启动时用会话相关配置初始化 API client。它把认证、代理和遥测设置串起来。

**核心逻辑：**
- 配置 `ANTHROPIC_API_URL` 基础地址、认证头、代理设置
- 在启动早期调用 `initializeGrowthBook()`
- 设置 OTel span 管理
- 处理 Bedrock TLS 校验跳过开关 `CLAUDE_CODE_SKIP_BEDROCK_TLS`

---

## services/api/client.ts

**路径：** `src/services/api/client.ts`

**用途：** 提供已配置好的 SDK client 实例，以及发起 API 调用时用的辅助工具。

**导出：**

```typescript
export function getClient(): Anthropic
export function getClientForModel(model: string): Anthropic
```

**核心逻辑：**
- client 会延迟创建并缓存
- 会应用 `ANTHROPIC_BASE_URL` 环境变量里的基础地址覆盖
- 通过 `getMtlsConfig()` 配置 mTLS
- 如果设置了 `HTTPS_PROXY` / `HTTP_PROXY`，则经由代理转发

---

## services/api/claude.ts

**路径：** `src/services/api/claude.ts`

**用途：** Claude completions 的主 API 调用层。负责 prompt caching、额外请求体参数、任务预算配置、元数据和用户消息格式化。

**导出：**

```typescript
export function getExtraBodyParams(betaHeaders?: string[]): JsonObject
export function getPromptCachingEnabled(model: string): boolean
export function getCacheControl(opts: {
  scope?: string
  querySource?: QuerySource
}): { type: 'ephemeral' | 'persistent'; ttl?: number; scope?: string }
export function configureTaskBudgetParams(
  taskBudget: number,
  outputConfig: OutputConfig,
  betas: string[]
): void
export function getAPIMetadata(): { user_id: string }
export async function verifyApiKey(
  apiKey: string,
  isNonInteractiveSession: boolean
): Promise<boolean>
export function userMessageToMessageParam(
  message: UserMessage,
  addCache: boolean,
  enablePromptCaching: boolean,
  querySource?: QuerySource
): MessageParam
```

**内部函数（不导出）：**
- `configureEffortParams()`：按 effort level 设置 thinking budget
- `should1hCacheTTL()`：检查当前用户/模型是否适用 1 小时 TTL

**核心逻辑：**
- **Prompt caching：** `getPromptCachingEnabled()` 会检查模型 allowlist。`getCacheControl()` 通常返回 `{ type: 'ephemeral' }`；当 1h TTL gate 通过时返回 `{ type: 'ephemeral', ttl: 3600 }`
- **1h TTL gate：** `should1hCacheTTL()` 会检查 `tengu_prompt_cache_1h_config` GrowthBook allowlist（会话稳定，保存在 `STATE.promptCache1hAllowlist`）以及 `STATE.promptCache1hEligible`（同样会被 latch，避免会话中途因额度变化而翻转）
- **Anti-distillation：** `tengu_anti_distill_fake_tool_injection` GrowthBook gate 会在 API 调用里注入 fake tools，作为训练数据质量信号
- **额外请求体参数：** `getExtraBodyParams()` 负责拼装 beta headers、模型特定参数和上下文管理配置

**依赖：** `@anthropic-ai/sdk`、`bootstrap/state.js`、`services/analytics/growthbook.js`、`services/compact/apiMicrocompact.js`

---

## services/api/dumpPrompts.ts

**路径：** `src/services/api/dumpPrompts.ts`

**用途：** 面向 ant 用户的调试工具，把 API request/response 的 JSONL 日志写到磁盘，便于排查 prompt 和分享 bug report。

**导出：**

```typescript
export function getLastApiRequests(): Array<{ timestamp: string; request: unknown }>
export function clearApiRequestCache(): void
export function clearDumpState(agentIdOrSessionId: string): void
export function clearAllDumpState(): void
export function addApiRequestToCache(requestData: unknown): void
export function getDumpPromptsPath(agentIdOrSessionId?: string): string
export function createDumpPromptsFetch(
  agentIdOrSessionId: string
): ClientOptions['fetch']
```

**核心逻辑：**
- 仅限 ant 用户，其他用户为 no-op
- `MAX_CACHED_REQUESTS = 5` - 最近请求的内存环形缓冲区
- 通过 `setImmediate` 延迟写入，避免阻塞请求路径
- JSONL 记录类型包括：`init`、`system_update`、`message`、`response`
- 通过 fingerprint 做 per-session 状态追踪，避免反复写相同的 system prompt
- 路径：`~/.claude/dump-prompts/<agentIdOrSessionId>.jsonl`

---

## services/api/emptyUsage.ts

**路径：** `src/services/api/emptyUsage.ts`

**用途：** 在 usage 数据不可用时，提供一个零值 `Usage` 对象。

**导出：**

```typescript
export const EMPTY_USAGE: Usage
export type NonNullableUsage = {
  input_tokens: number
  output_tokens: number
  cache_read_input_tokens: number
  cache_creation_input_tokens: number
}
```

---

## services/api/errorUtils.ts

**路径：** `src/services/api/errorUtils.ts`

**用途：** API 错误分类与处理工具。

**导出：**

```typescript
export function isRateLimitError(error: unknown): boolean
export function isOverloadedError(error: unknown): boolean
export function isAuthError(error: unknown): boolean
export function isConnectionError(error: unknown): boolean
export function getRetryAfterMs(error: unknown): number | undefined
```

---

## services/api/errors.ts

**路径：** `src/services/api/errors.ts`

**用途：** 定义所有面向用户的 API 错误消息常量，以及对应的分类函数。

**导出：**

```typescript
export const API_ERROR_MESSAGE_PREFIX = 'API Error'
export function startsWithApiErrorPrefix(text: string): boolean

export const PROMPT_TOO_LONG_ERROR_MESSAGE: string
export function isPromptTooLongMessage(msg: string): boolean
export function parsePromptTooLongTokenCounts(
  rawMessage: string
): { actualTokens: number; limitTokens: number } | null
export function getPromptTooLongTokenGap(msg: string): number | undefined

export function isMediaSizeError(raw: unknown): boolean
export function isMediaSizeErrorMessage(msg: string): boolean

export const CREDIT_BALANCE_TOO_LOW_ERROR_MESSAGE: string
export const INVALID_API_KEY_ERROR_MESSAGE: string
export const INVALID_API_KEY_ERROR_MESSAGE_EXTERNAL: string
export const TOKEN_REVOKED_ERROR_MESSAGE: string
export const CCR_AUTH_ERROR_MESSAGE: string
export const REPEATED_529_ERROR_MESSAGE: string
export const CUSTOM_OFF_SWITCH_MESSAGE: string
export const API_TIMEOUT_ERROR_MESSAGE: string
export const OAUTH_ORG_NOT_ALLOWED_ERROR_MESSAGE: string

export function getPdfTooLargeErrorMessage(): string
export function getPdfPasswordProtectedErrorMessage(): string
export function getPdfInvalidErrorMessage(): string
export function getImageTooLargeErrorMessage(): string
export function getRequestTooLargeErrorMessage(): string
export function getTokenRevokedErrorMessage(): string
export function getOauthOrgNotAllowedErrorMessage(): string
```

---

## services/api/filesApi.ts

**路径：** `src/services/api/filesApi.ts`

**用途：** Files API 客户端，用于下载、上传、列举和管理 Files API（beta）里的文件。

**常量：**
- `FILES_API_BETA_HEADER = 'files-api-2025-04-14,oauth-2025-04-20'`
- `MAX_FILE_SIZE_BYTES = 500 * 1024 * 1024`（500MB）
- `DEFAULT_CONCURRENCY = 5`
- `MAX_RETRIES = 3`
- `BASE_DELAY_MS = 500`

**导出：**

```typescript
export type File = {
  fileId: string
  relativePath: string
  mimeType?: string
}

export type FilesApiConfig = {
  apiKey?: string
  baseUrl?: string
  sessionId?: string
}

export type DownloadResult = {
  fileId: string
  relativePath: string
  success: boolean
  error?: string
  savedPath?: string
}

export type UploadResult = {
  fileId: string
  relativePath: string
  success: boolean
  error?: string
  remoteFileId?: string
}

export type FileMetadata = {
  id: string
  filename: string
  created_at: number
  purpose: string
  size: number
}

export class UploadNonRetriableError extends Error {}

export function parseFileSpecs(fileSpecs: string[]): File[]
export async function downloadFile(fileId: string, config: FilesApiConfig): Promise<Buffer>
export function buildDownloadPath(
  basePath: string,
  sessionId: string,
  relativePath: string
): string | null
export async function downloadAndSaveFile(
  attachment: File,
  config: FilesApiConfig
): Promise<DownloadResult>
export async function downloadSessionFiles(
  files: File[],
  config: FilesApiConfig,
  concurrency?: number
): Promise<DownloadResult[]>
export async function uploadFile(
  filePath: string,
  relativePath: string,
  config: FilesApiConfig,
  opts?: { retries?: number }
): Promise<UploadResult>
export async function uploadSessionFiles(
  files: File[],
  config: FilesApiConfig,
  concurrency?: number
): Promise<UploadResult[]>
export async function listFilesCreatedAfter(
  afterCreatedAt: number,
  config: FilesApiConfig
): Promise<FileMetadata[]>
```

**核心逻辑：**
- `buildDownloadPath()`：防止路径穿越。如果 `relativePath` 里包含 `..`，或者解析后落到 `basePath/sessionId` 之外，就返回 `null`
- 下载/上传都使用指数退避：`BASE_DELAY_MS * 2^attempt`，并带 jitter
- `uploadSessionFiles()` / `downloadSessionFiles()`：支持并发执行，默认并发数为 5
- `listFilesCreatedAfter()`：使用 cursor 分页，收集所有页

---

## services/api/firstTokenDate.ts

**路径：** `src/services/api/firstTokenDate.ts`

**用途：** 获取并保存用户第一次进行 Claude Code API 调用的日期。

**导出：**

```typescript
export async function fetchAndStoreClaudeCodeFirstTokenDate(): Promise<void>
```

**核心逻辑：**
- 请求 `/api/organization/claude_code_first_token_date`
- 保存到 `claudeCodeFirstTokenDate` 配置字段
- 幂等：如果已经保存过，就直接跳过

---

## services/api/grove.ts

**路径：** `src/services/api/grove.ts`

**用途：** Grove 是一个面向消费者的 Terms/Privacy Policy 通知功能。它负责拉取、缓存，并判断是否要向用户展示 Grove 提示。

**常量：**
- `GROVE_CACHE_EXPIRATION_MS = 24 * 60 * 60 * 1000`（24 小时）

**导出：**

```typescript
export type AccountSettings = {
  groveEnabled: boolean
  groveNoticeViewed: boolean
  // ...
}

export type GroveConfig = {
  enabled: boolean
  forceShow: boolean
  // ...
}

export type ApiResult<T> = { success: true; data: T } | { success: false; error: string }

export async function getGroveSettings(): Promise<ApiResult<AccountSettings>>  // memoized 24h
export async function markGroveNoticeViewed(): Promise<void>
export async function updateGroveSettings(groveEnabled: boolean): Promise<void>
export async function isQualifiedForGrove(): Promise<boolean>
export async function getGroveNoticeConfig(): Promise<ApiResult<GroveConfig>>  // memoized 24h
export function calculateShouldShowGrove(
  settingsResult: ApiResult<AccountSettings>,
  configResult: ApiResult<GroveConfig>,
  showIfAlreadyViewed: boolean
): boolean
export async function checkGroveForNonInteractive(): Promise<void>
```

**核心逻辑：**
- 采用 cache-first + 后台刷新：先立即返回缓存数据，过期后再后台刷新
- `calculateShouldShowGrove()`：检查 config 是否启用、settings 是否已经看过、以及用户是否符合资格
- `checkGroveForNonInteractive()`：在非交互模式下调用，用来记录 grove 状态，但不展示 UI

---

## services/api/logging.ts

**路径：** `src/services/api/logging.ts`

**用途：** API 查询/成功/错误日志，带 gateway 检测和 OTel span 管理。

**导出：**

```typescript
export type GlobalCacheStrategy = 'tool_based' | 'system_prompt' | 'none'

export function logAPIQuery(opts: {
  model: string
  querySource: QuerySource
  // ... 其他字段
}): void

export function logAPIError(opts: {
  error: unknown
  model: string
  querySource: QuerySource
  // ... 其他字段
}): void

export function logAPISuccessAndDuration(opts: {
  model: string
  usage: Usage
  querySource: QuerySource
  durationMs: number
  // ... 其他字段
}): void
```

**核心逻辑：**
- Gateway 检测：从 `ANTHROPIC_BASE_URL` 识别 litellm、helicone、portkey、cloudflare-ai-gateway、kong、braintrust、databricks
- 事件：`tengu_api_query`、`tengu_api_error`、`tengu_api_success`
- API 调用前后会创建/结束 OTel spans
- 会在 teleported session 里记录额外字段
- 重新导出 `EMPTY_USAGE` 和 `NonNullableUsage`

---

## services/api/metricsOptOut.ts

**路径：** `src/services/api/metricsOptOut.ts`

**用途：** 检查当前组织是否启用了 metrics 收集。实现了两层缓存。

**常量：**
- `CACHE_TTL_MS = 60 * 60 * 1000`（内存 1 小时）
- `DISK_CACHE_TTL_MS = 24 * 60 * 60 * 1000`（磁盘 24 小时）
- Endpoint：`api/claude_code/organizations/metrics_enabled`

**导出：**

```typescript
export async function checkMetricsEnabled(): Promise<MetricsStatus>
export function _clearMetricsEnabledCacheForTesting(): void
```

**核心逻辑：**
- 两层缓存：内存（1h TTL）→ 磁盘（24h TTL）→ 网络
- 需要 `profile` OAuth scope；如果未认证或缺少 scope，则返回 `enabled: true`
- `MetricsStatus`：`{ enabled: boolean; source: 'cache' | 'network' | 'default' }`

---

## services/api/overageCreditGrant.ts

**路径：** `src/services/api/overageCreditGrant.ts`

**用途：** 管理已订阅用户超出套餐额度时的 overage credit grant 信息。

**常量：**
- `CACHE_TTL_MS = 60 * 60 * 1000`（1 小时）

**导出：**

```typescript
export type OverageCreditGrantInfo = {
  hasGrant: boolean
  amount?: number
  currency?: string
  expiresAt?: string
}

export type OverageCreditGrantCacheEntry = {
  data: OverageCreditGrantInfo
  fetchedAt: number
  orgId: string
}

export function getCachedOverageCreditGrant(): OverageCreditGrantInfo | null
export function invalidateOverageCreditGrantCache(): void
export async function refreshOverageCreditGrantCache(): Promise<void>
export function formatGrantAmount(info: OverageCreditGrantInfo): string | null
```

**核心逻辑：**
- 按 org 维度缓存，存放在 `overageCreditGrantCache` Map 中（key 是 org ID）
- `formatGrantAmount()`：把金额格式化为货币字符串（例如 `"$5.00"`），没有 grant 时返回 `null`

---

## services/api/promptCacheBreakDetection.ts

**路径：** `src/services/api/promptCacheBreakDetection.ts`

**用途：** 检测异常的 prompt cache break，这通常表示服务端缓存被淘汰。发现后会把 diff 文件写到磁盘，方便调试。

**常量：**
- `CACHE_TTL_1HOUR_MS = 3_600_000`（1 小时）
- `MIN_CACHE_MISS_TOKENS = 2_000` - 判定为明显 break 的最低阈值
- 95% 阈值 - cache reads 需要降到预期的 5% 以下才算 break
- `MAX_TRACKED_SOURCES = 10`

**导出：**

```typescript
export type PromptStateSnapshot = {
  messages: MessageParam[]
  systemPrompt: string
  tools: unknown[]
  timestamp: number
  querySource: QuerySource
}

export const CACHE_TTL_1HOUR_MS: number

export function recordPromptState(snapshot: PromptStateSnapshot): void
export async function checkResponseForCacheBreak(
  querySource: QuerySource,
  cacheReadTokens: number,
  cacheCreationTokens: number,
  messages: MessageParam[],
  agentId?: string,
  requestId?: string
): Promise<void>
export function notifyCacheDeletion(querySource: QuerySource, agentId?: string): void
export function notifyCompaction(querySource: QuerySource, agentId?: string): void
export function cleanupAgentTracking(agentId: string): void
export function resetPromptCacheBreakDetection(): void
```

**核心逻辑：**
- 两阶段检测：调用前 `recordPromptState()`，调用后 `checkResponseForCacheBreak()`
- 按 source 追踪的 Map：key 是 `querySource`（或 `agent:${agentId}`）
- 跟踪的 source 前缀包括：`repl_main_thread`、`sdk`、`agent:custom`、`agent:default`、`agent:builtin`
- 检测到 break 时会把 diff 文件写到 `~/.claude/tmp/cache-break-*.diff`
- 事件：`tengu_prompt_cache_break`
- `notifyCacheDeletion()` / `notifyCompaction()`：在有意清理 cache 后抑制误报

---

## services/api/referral.ts

**路径：** `src/services/api/referral.ts`

**用途：** 管理 Claude Code 订阅用户的 referral 计划资格、兑换和 guest pass。

**常量：**
- `CACHE_EXPIRATION_MS = 24 * 60 * 60 * 1000`（24 小时）

**导出：**

```typescript
export async function fetchReferralEligibility(
  campaign?: string
): Promise<ReferralEligibilityResponse>
export async function fetchReferralRedemptions(
  campaign?: string
): Promise<ReferralRedemptionsResponse>
export function checkCachedPassesEligibility(): {
  eligible: boolean
  needsRefresh: boolean
  hasCache: boolean
}
export function formatCreditAmount(reward: ReferrerReward): string
export function getCachedReferrerReward(): ReferrerRewardInfo | null
export function getCachedRemainingPasses(): number | null
export async function fetchAndStorePassesEligibility(): Promise<ReferralEligibilityResponse | null>
export async function getCachedOrFetchPassesEligibility(): Promise<ReferralEligibilityResponse | null>
export async function prefetchPassesEligibility(): Promise<void>
```

**核心逻辑：**
- 仅限 max subscription；非 max 用户会返回 `null` / 不符合资格
- 通过 in-flight 去重：多个 `getCachedOrFetchPassesEligibility()` 调用共享同一个 pending Promise
- 缓存 TTL 为 24 小时

---

## services/api/sessionIngress.ts

**路径：** `src/services/api/sessionIngress.ts`

**用途：** 管理 session log ingress。它通过乐观并发的 append-log 机制支持多写者场景，例如不同机器继续同一个会话。

**常量：**
- `MAX_RETRIES = 10`
- `BASE_DELAY_MS = 500` - 指数退避基准

**导出：**

```typescript
export async function appendSessionLog(
  sessionId: string,
  entry: SessionLogEntry,
  url: string
): Promise<boolean>
export async function getSessionLogs(
  sessionId: string,
  url: string
): Promise<Entry[] | null>
export async function getSessionLogsViaOAuth(
  sessionId: string,
  accessToken: string,
  orgUUID: string
): Promise<Entry[] | null>
export async function getTeleportEvents(
  sessionId: string,
  accessToken: string,
  orgUUID: string
): Promise<Entry[] | null>
export function clearSession(sessionId: string): void
export function clearAllSessions(): void
```

**核心逻辑：**
- 乐观并发：append 时带 `Last-Uuid` 头；如果返回 409，就采用 server 提供的最后 UUID
- 每个 session 都有顺序包装器，防止 append 顺序错乱
- `getTeleportEvents()`：分页获取，最多 100 页、每页 1000 条事件
- `clearSession()` / `clearAllSessions()`：清除内存中的顺序包装器状态

---

## services/api/ultrareviewQuota.ts

**路径：** `src/services/api/ultrareviewQuota.ts`

**用途：** 获取订阅用户的 ultrareview（深度代码审查）额度信息。

**导出：**

```typescript
export type UltrareviewQuotaResponse = {
  used: number
  limit: number
  resetsAt: string
}

export async function fetchUltrareviewQuota(): Promise<UltrareviewQuotaResponse | null>
```

**核心逻辑：**
- Endpoint：`/v1/ultrareview/quota`
- 仅订阅用户可用；非订阅用户或出错时返回 `null`
- 超时 5 秒

---

## services/api/usage.ts

**路径：** `src/services/api/usage.ts`

**用途：** 从 Claude Code API 获取当前用户/组织的 usage 统计。

**导出：**

```typescript
export async function fetchUsage(): Promise<UsageStats | null>
export type UsageStats = {
  // usage breakdown fields
}
```

---

## services/api/withRetry.ts

**路径：** `src/services/api/withRetry.ts`

**用途：** 用指数退避包装 API 调用的通用重试器。

**导出：**

```typescript
export async function withRetry<T>(
  fn: () => Promise<T>,
  opts?: {
    maxRetries?: number
    baseDelayMs?: number
    shouldRetry?: (error: unknown) => boolean
  }
): Promise<T>
```

---

## services/AgentSummary/agentSummary.ts

**路径：** `src/services/AgentSummary/agentSummary.ts`

**用途：** 对 agent 对话做周期性后台总结，在保留关键信息的同时压缩上下文。

**核心逻辑：**
- 每 30 秒在后台跑一次
- 用主 Claude 模型生成摘要
- 压缩后的摘要会作为 system message 注回去
- 供 subagent 和 teammate 用来处理长会话

---

## services/autoDream/autoDream.ts

**路径：** `src/services/autoDream/autoDream.ts`

**用途：** 后台记忆整合系统。它会周期性扫描 session transcript，并用 fork 出来的 agent 把学习到的内容整合进持久记忆文件。

**常量：**
- `SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000`（10 分钟）
- `DEFAULTS = { minHours: 24, minSessions: 5 }` - 在开始整合前至少需要的时间和会话数

**导出：**

```typescript
export function initAutoDream(): () => void  // 返回 stop/cleanup 函数
```

**核心逻辑：**
- gate 顺序：时间检查（minHours）→ 会话数检查（minSessions）→ consolidation lock
- GrowthBook config `tengu_onyx_plover` 控制 `{ minHours, minSessions, enabled }`
- `initAutoDream()` 会注册为 post-sampling hook；返回 cleanup 函数
- 轮询使用 `SESSION_SCAN_INTERVAL_MS`
- 通过 `buildConsolidationPrompt()` 启动 fork 出来的 agent

---

## services/autoDream/config.ts

**路径：** `src/services/autoDream/config.ts`

**用途：** autoDream 记忆整合功能的配置 gate。

**导出：**

```typescript
export function isAutoDreamEnabled(): boolean
```

**核心逻辑：** 用户设置优先于 GrowthBook gate `tengu_onyx_plover`。先看 `userSettings.autoDream`，再看 GrowthBook。

---

## services/autoDream/consolidationLock.ts

**路径：** `src/services/autoDream/consolidationLock.ts`

**用途：** 用文件实现的 mutex lock，防止不同 session / 进程同时做记忆整合。

**常量：**
- `LOCK_FILE = '.consolidate-lock'` - 位于 memory 目录
- `HOLDER_STALE_MS = 60 * 60 * 1000`（1 小时）- 锁过期阈值

**导出：**

```typescript
export async function readLastConsolidatedAt(): Promise<number>
export async function tryAcquireConsolidationLock(): Promise<number | null>
export async function rollbackConsolidationLock(priorMtime: number): Promise<void>
export async function listSessionsTouchedSince(sinceMs: number): Promise<string[]>
export async function recordConsolidation(): Promise<void>
```

**核心逻辑：**
- lock file 的 mtime 就是 `lastConsolidatedAt` 时间戳，兼作锁和时间戳
- 基于 PID 的所有权判断：如果 PID 死了，或者锁已超过 1 小时，就当作 stale 并覆盖
- `tryAcquireConsolidationLock()`：成功时返回之前的 mtime，已被占用则返回 `null`
- `rollbackConsolidationLock()`：在整合失败时把 mtime 恢复成 `priorMtime`

---

## services/autoDream/consolidationPrompt.ts

**路径：** `src/services/autoDream/consolidationPrompt.ts`

**用途：** 生成记忆整合 agent 的系统提示词。

**导出：**

```typescript
export function buildConsolidationPrompt(
  memoryRoot: string,
  transcriptDir: string,
  extra: string
): string
```

**核心逻辑：** 返回一个 4 阶段提示词：
1. 定向：先读现有 memory 文件，理解当前状态
2. 收集近期信号：读取上次整合以来的 session transcript
3. 整合：把新学习到的内容并入 memory 文件
4. 清理与索引：删除过期条目，更新索引文件

---

## services/awaySummary.ts

**路径：** `src/services/awaySummary.ts`

**用途：** 生成 “away summary” - 当用户离开一个长会话后重新回来时，显示一段简短的回顾。

**核心逻辑：**
- 当 `lastInteractionTime` 的间隔超过阈值时触发
- 用 Claude 生成一段 1-3 句的简短总结，说明用户离开期间发生了什么
- 以 system message 的形式显示在提示符上方

---

## services/claudeAiLimits.ts

**路径：** `src/services/claudeAiLimits.ts`

**用途：** 获取并管理 Claude.ai 认证用户的 rate limit 信息。

**导出：**

```typescript
export async function fetchClaudeAiLimits(): Promise<ClaudeAiLimits | null>
export type ClaudeAiLimits = {
  // rate limit fields
}
```

---

## services/claudeAiLimitsHook.ts

**路径：** `src/services/claudeAiLimitsHook.ts`

**用途：** 用 React hook 访问 Claude.ai 的 rate limit 数据，并自动刷新。

**导出：**

```typescript
export function useClaudeAiLimits(): ClaudeAiLimits | null
```

---

## services/compact/apiMicrocompact.ts

**路径：** `src/services/compact/apiMicrocompact.ts`

**用途：** 使用服务端 `cache_edits` 能力的 API 原生上下文管理策略。它可以在不做完整客户端重写的情况下编辑 context window。

**常量：**
- `DEFAULT_MAX_INPUT_TOKENS = 180_000`
- `DEFAULT_TARGET_INPUT_TOKENS = 40_000`

**类型 `ContextEditStrategy`：**
```typescript
export type ContextEditStrategy =
  | {
      type: 'clear_tool_uses_20250919'
      trigger?: { type: 'input_tokens'; value: number }
      keep?: { type: 'tool_uses'; value: number }
      clear_tool_inputs?: boolean | string[]
      exclude_tools?: string[]
      clear_at_least?: { type: 'input_tokens'; value: number }
    }
  | {
      type: 'clear_thinking_20251015'
      keep: { type: 'thinking_turns'; value: number } | 'all'
    }

export type ContextManagementConfig = {
  edits: ContextEditStrategy[]
}
```

**导出：**

```typescript
export function getAPIContextManagement(options?: {
  hasThinking?: boolean
  isRedactThinkingActive?: boolean
  clearAllThinking?: boolean
}): ContextManagementConfig | undefined
```

**核心逻辑：**
- 工具清理策略只在 ant 可用，由 `USE_API_CLEAR_TOOL_RESULTS` 和 `USE_API_CLEAR_TOOL_USES` 环境变量控制
- `TOOLS_CLEARABLE_RESULTS`：shell tools、Glob、Grep、FileRead、WebFetch、WebSearch
- `TOOLS_CLEARABLE_USES`：FileEdit、FileWrite、NotebookEdit
- Thinking 清理：当 `hasThinking && !isRedactThinkingActive` 时，加上 `clear_thinking_20251015`
- 当 `clearAllThinking`（>1 小时 idle，表示确认 cache miss）时，只保留最近 1 个 thinking turn

---

## services/compact/autoCompact.ts

**路径：** `src/services/compact/autoCompact.ts`

**用途：** 自动 context compaction。当 context window 使用率超过阈值时，触发完整会话总结。

**核心逻辑：**
- 监控 token 使用量，默认阈值是 context window 的 90%
- 触发后调用 `compact()` 总结对话
- 在会话里插入 `CompactBoundaryMessage` 标记 compaction 点
- compaction 后重置 token 追踪

---

## services/compact/compact.ts

**路径：** `src/services/compact/compact.ts`

**用途：** 完整会话 compaction，用 LLM 生成的摘要替换会话历史。

**核心逻辑：**
- 使用 `prompt.ts` 里更详细的分析提示词
- 可以做部分 compaction（保留最近消息）
- `NO_TOOLS_PREAMBLE` 是关键指令，防止 compaction 模型在总结时调用工具
- 把 compact summary 写成一个 synthetic `<compact_summary>` 标记消息

---

## services/compact/compactWarningHook.ts

**路径：** `src/services/compact/compactWarningHook.ts`

**用途：** 访问 compact warning suppression 状态的 React hook。

**导出：**

```typescript
export function useCompactWarningSuppression(): boolean
```

---

## services/compact/compactWarningState.ts

**路径：** `src/services/compact/compactWarningState.ts`

**用途：** 在 microcompact 运行后，存储并控制 “建议 compact” 警告的抑制状态。

**导出：**

```typescript
export const compactWarningStore: Store<boolean>
export function suppressCompactWarning(): void
export function clearCompactWarningSuppression(): void
```

---

## services/compact/grouping.ts

**路径：** `src/services/compact/grouping.ts`

**用途：** 按 API round 对会话消息分组（每个 assistant message 连同它前面的 user message）。

**导出：**

```typescript
export function groupMessagesByApiRound(messages: Message[]): Message[][]
```

**核心逻辑：** 按 assistant `message.id` 边界分组。每组包含一个 API round（user + assistant + tool results）。

---

## services/compact/microCompact.ts

**路径：** `src/services/compact/microCompact.ts`

**用途：** microcompaction。它通过清空 tool result 内容来轻量压缩上下文，而不是做完整会话总结。它有两条路径：缓存 microcompact（通过 API `cache_edits`）和基于时间的 microcompact（cache 冷时直接改内容）。

**导出的常量：**
```typescript
export const TIME_BASED_MC_CLEARED_MESSAGE = '[旧的工具结果内容已清除]'
```

**可压缩工具集：**
```typescript
const COMPACTABLE_TOOLS = new Set([
  FILE_READ_TOOL_NAME, SHELL_TOOL_NAMES..., GREP_TOOL_NAME, GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME, WEB_FETCH_TOOL_NAME, FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME
])
```

**导出：**

```typescript
export function consumePendingCacheEdits():
  import('./cachedMicrocompact.js').CacheEditsBlock | null

export function getPinnedCacheEdits():
  import('./cachedMicrocompact.js').PinnedCacheEdits[]

export function pinCacheEdits(
  userMessageIndex: number,
  block: import('./cachedMicrocompact.js').CacheEditsBlock
): void

export function markToolsSentToAPIState(): void

export function resetMicrocompactState(): void

export function estimateMessageTokens(messages: Message[]): number

export type PendingCacheEdits = {
  trigger: 'auto'
  deletedToolIds: string[]
  baselineCacheDeletedTokens: number
}

export type MicrocompactResult = {
  messages: Message[]
  compactionInfo?: {
    pendingCacheEdits?: PendingCacheEdits
  }
}

export function evaluateTimeBasedTrigger(
  messages: Message[],
  querySource: QuerySource | undefined
): { gapMinutes: number; config: TimeBasedMCConfig } | null

export async function microcompactMessages(
  messages: Message[],
  toolUseContext?: ToolUseContext,
  querySource?: QuerySource
): Promise<MicrocompactResult>
```

**核心逻辑：**
- `microcompactMessages()` 的分发顺序：
  1. 检查时间触发：如果距离上次 assistant 太久超过阈值，则走 `maybeTimeBasedMicrocompact()`，并直接短路
  2. 缓存 MC 路径：如果启用了 `CACHED_MICROCOMPACT` feature，模型受支持，且是 main thread source，就走 `cachedMicrocompactPath()`
  3. 否则：原样返回 messages（旧逻辑已移除）
- **缓存 MC 路径：** 按 user message 分组注册 tool results；调用 `getToolResultsToDelete()`；把 `CacheEditsBlock` 排队到 `pendingCacheEdits`；不会直接改 message 内容
- **时间 MC 路径：** 直接把 tool result `content` 改成 `TIME_BASED_MC_CLEARED_MESSAGE`；重置缓存 MC 状态；通知 cache break detection
- `estimateMessageTokens()`：粗略估算，带 4/3 padding；图片/文档按 2000 token 计算
- `isMainThreadSource()`：按前缀匹配 `repl_main_thread`（兼容 `repl_main_thread:outputStyle:custom` 之类的输出样式）
- 事件：`tengu_cached_microcompact`、`tengu_time_based_microcompact`

---

## services/compact/postCompactCleanup.ts

**路径：** `src/services/compact/postCompactCleanup.ts`

**用途：** 在任何 compaction 之后运行清理任务，无论是自动 compaction 还是手动 `/compact`。

**导出：**

```typescript
export function runPostCompactCleanup(querySource?: QuerySource): void
```

**核心逻辑：** 清除 microcompact 状态、context collapse 状态、system prompt sections、classifier approvals、speculative checks、beta tracing 状态，以及 session messages cache。

---

## services/compact/prompt.ts

**路径：** `src/services/compact/prompt.ts`

**用途：** 供 compact 总结使用的提示词常量。

**主要导出：**
- `NO_TOOLS_PREAMBLE`：在 compact 调用前追加的关键指令，防止模型在总结时调用任何工具
- `DETAILED_ANALYSIS_INSTRUCTION_BASE`：完整 compaction 的指令
- `DETAILED_ANALYSIS_INSTRUCTION_PARTIAL`：部分 compaction 的指令（保留最近消息）

---

## services/compact/sessionMemoryCompact.ts

**路径：** `src/services/compact/sessionMemoryCompact.ts`

**用途：** session memory compaction。它比 full compact 更轻，只总结较老的 session memory 段，同时保留最近的 tool context。

**常量：**
```typescript
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10000,
  minTextBlockMessages: 5,
  maxTokens: 40000
}
```

**导出：**

```typescript
export type SessionMemoryCompactConfig = {
  minTokens: number
  minTextBlockMessages: number
  maxTokens: number
}

export function setSessionMemoryCompactConfig(config: SessionMemoryCompactConfig): void
export function getSessionMemoryCompactConfig(): SessionMemoryCompactConfig
export function resetSessionMemoryCompactConfig(): void
export function hasTextBlocks(message: Message): boolean
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[],
  startIndex: number
): number
export function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number
): number
export function shouldUseSessionMemoryCompaction(): boolean
export async function trySessionMemoryCompaction(
  messages: Message[],
  agentId?: string,
  autoCompactThreshold?: number
): Promise<CompactionResult | null>
```

**核心逻辑：**
- GrowthBook config `tengu_sm_compact_config` 会覆盖默认值
- 由 `ENABLE_CLAUDE_CODE_SM_COMPACT` / `DISABLE_CLAUDE_CODE_SM_COMPACT` 环境变量控制
- 还要求 GrowthBook gates `tengu_session_memory` 和 `tengu_sm_compact` 都开启
- `adjustIndexToPreserveAPIInvariants()`：保证切点不会留下孤立的 `tool_use` 而没有 `tool_result`
- `calculateMessagesToKeepIndex()`：通过二分搜索计算 index，同时尊重 min/max token 边界

---

## services/compact/timeBasedMCConfig.ts

**路径：** `src/services/compact/timeBasedMCConfig.ts`

**用途：** 基于时间的 microcompact 配置（在空闲间隔触发）。

**导出：**

```typescript
export type TimeBasedMCConfig = {
  enabled: boolean
  gapThresholdMinutes: number
  keepRecent: number
}

export function getTimeBasedMCConfig(): TimeBasedMCConfig
```

**核心逻辑：**
- GrowthBook config：`tengu_slate_heron`
- 默认值：`{ enabled: false, gapThresholdMinutes: 60, keepRecent: 5 }`

---

## services/diagnosticTracking.ts

**路径：** `src/services/diagnosticTracking.ts`

**用途：** 追踪文件诊断信息（lint error、type error）在文件编辑前后的变化，用来发现 Claude Code 引入的回归。

**常量：**
- `MAX_DIAGNOSTICS_SUMMARY_CHARS = 4000`

**导出：**

```typescript
export interface Diagnostic {
  severity: 'error' | 'warning' | 'information' | 'hint'
  message: string
  source?: string
  range: { start: { line: number; character: number }; end: { line: number; character: number } }
}

export interface DiagnosticFile {
  uri: string
  diagnostics: Diagnostic[]
}

export class DiagnosticTrackingService {
  static getInstance(): DiagnosticTrackingService
  initialize(mcpClient: unknown): void
  shutdown(): Promise<void>
  reset(): void
  ensureFileOpened(fileUri: string): Promise<void>
  beforeFileEdited(filePath: string): Promise<void>
}
```

**核心逻辑：**
- 通过 `getInstance()` 实现单例
- `beforeFileEdited()`：在编辑前捕获文件当前 diagnostics，这样编辑后就能对比
- `initialize()`：连接 LSP MCP client 获取诊断数据
- `shutdown()`：刷新所有待处理的 diagnostic 对比

---

## services/internalLogging.ts

**路径：** `src/services/internalLogging.ts`

**用途：** 面向内部（ant）用户的日志工具，用于排查问题时记录 K8s namespace、container ID 和工具权限上下文。

**导出：**

```typescript
export async function getContainerId(): Promise<string | null>  // memoized
export async function logPermissionContextForAnts(
  toolPermissionContext: unknown,
  moment: string
): Promise<void>
```

**核心逻辑：**
- K8s namespace：从 `/var/run/secrets/kubernetes.io/serviceaccount/namespace` 读取
- Container ID：从 `/proc/self/mountinfo` 里提取（第一个 overlay/device entry）
- 两者都会 memoize，因为 container 身份在会话中不会变
- 只有在 `USER_TYPE === 'ant'` 时才会记录

---

## services/MagicDocs/magicDocs.ts

**路径：** `src/services/MagicDocs/magicDocs.ts`

**用途：** 自动维护的 markdown 文档，它们会跟随 Claude Code 做出的代码改动保持同步。

**核心逻辑：**
- 监控文件编辑并重新生成关联的 markdown 文档
- 用 Claude 理解改动的语义含义
- 提示词定义在 `prompts.ts`

---

## services/MagicDocs/prompts.ts

**路径：** `src/services/MagicDocs/prompts.ts`

**用途：** magic docs 生成用的系统提示词和指令模板。

---

## services/mcpServerApproval.tsx

**路径：** `src/services/mcpServerApproval.tsx`

**用途：** 启动时为待审批的 project MCP servers 显示审批对话框，复用现有 Ink root 实例。

**导出：**

```typescript
export async function handleMcpjsonServerApprovals(root: Root): Promise<void>
```

**核心逻辑：**
- 通过 `getMcpConfigsByScope('project')` 查询 project 范围内的 MCP server
- 再用 `getProjectMcpServerStatus()` 过滤出状态为 `'pending'` 的 server
- 只有一个 pending server 时，渲染 `MCPServerApprovalDialog`
- 有多个 pending server 时，渲染 `MCPServerMultiselectDialog`
- 在返回前会通过 Promise/resolve 模式等待用户决定

---

## services/mockRateLimits.ts

**路径：** `src/services/mockRateLimits.ts`

**用途：** 用于开发/测试的 rate limit 场景模拟，不需要真的撞到 API 限额。仅限 ant。

**导出：**

```typescript
export type MockHeaderKey =
  | 'x-ratelimit-requests-remaining'
  | 'x-ratelimit-tokens-remaining'
  // ... 其他 rate limit header 名称

export type MockScenario =
  | 'primary_hard_limit'
  | 'secondary_hard_limit'
  | 'approaching_limit'
  | 'fast_mode_rate_limit'
  | 'burst_limit'
  // ... 总共 20+ 个场景

export function setMockHeader(key: MockHeaderKey, value?: string): void
export function addExceededLimit(type: string, hoursFromNow: number): void
export function setMockEarlyWarning(
  claimAbbrev: string,
  utilization: number,
  hoursFromNow?: number
): void
export function clearMockEarlyWarning(): void
export function setMockRateLimitScenario(scenario: MockScenario): void
export function getMockHeaderless429Message(): string | null
export function getMockHeaders(): MockHeaders | null
export function getMockStatus(): string
export function clearMockHeaders(): void
export function applyMockHeaders(headers: Headers): Headers
export function shouldProcessMockLimits(): boolean
export function getCurrentMockScenario(): MockScenario | null
export function getScenarioDescription(scenario: MockScenario): string
export function setMockSubscriptionType(type: SubscriptionType | null): void
export function getMockSubscriptionType(): SubscriptionType | null
export function shouldUseMockSubscription(): boolean
export function setMockBillingAccess(hasAccess: boolean | null): void
export function isMockFastModeRateLimitScenario(): boolean
export function checkMockFastModeRateLimit(isFastModeActive?: boolean): MockHeaders | null
```

---

## services/notifier.ts

**路径：** `src/services/notifier.ts`

**用途：** 用于长时间运行操作的系统级桌面通知。

**核心逻辑：**
- 当任务完成且用户离开时，会在 macOS/Linux 上发送通知
- 使用 `node-notifier` 或原生 OS 通知 API
- 受用户偏好和焦点状态控制

---

## services/preventSleep.ts

**路径：** `src/services/preventSleep.ts`

**用途：** 在 Claude Code 任务运行期间阻止系统睡眠。

**核心逻辑：**
- 使用平台专用 API（macOS 的 caffeinate、Linux 的 systemd-inhibit）
- 返回一个 cleanup 函数，用来恢复睡眠
- 只在工具执行阶段启用

---

## services/PromptSuggestion/promptSuggestion.ts

**路径：** `src/services/PromptSuggestion/promptSuggestion.ts`

**用途：** 基于当前代码库上下文生成 prompt 建议，用于输入框自动补全。

**核心逻辑：**
- 分析最近的 git 改动、打开的文件和任务模式
- 返回排序后的建议列表
- 按上下文哈希做缓存，避免重复生成

---

## services/PromptSuggestion/speculation.ts

**路径：** `src/services/PromptSuggestion/speculation.ts`

**用途：** 预执行式 speculation。在用户确认前先跑可能的下一条命令，再决定保留还是丢弃结果。

**核心逻辑：**
- 监控用户输入模式，预测下一步动作
- 预热常见工具执行（读文件、搜索）
- 如果预测错了，就取消 speculative execution

---

## services/rateLimitMocking.ts

**路径：** `src/services/rateLimitMocking.ts`

**用途：** 处理 rate limit mock 应用和错误检查的 facade 层。

**导出：**

```typescript
export function processRateLimitHeaders(headers: Headers): Headers
export function shouldProcessRateLimits(isSubscriber: boolean): boolean
export function checkMockRateLimitError(
  currentModel: string,
  isFastModeActive?: boolean
): APIError | null
export function isMockRateLimitError(error: unknown): boolean
export { shouldProcessMockLimits }  // 从 mockRateLimits.ts re-export
```

---

## services/rateLimitMessages.ts

**路径：** `src/services/rateLimitMessages.ts`

**用途：** 面向人的 rate limit 消息格式化和展示逻辑。

**核心逻辑：**
- 把 rate limit headers 格式化成用户更容易看懂的消息
- 处理早期 warning、hard limit 和 reset 时间展示
- 把时间戳本地化到用户时区

---

## services/SessionMemory/prompts.ts

**路径：** `src/services/SessionMemory/prompts.ts`

**用途：** session memory 操作（摘要、检索、整合）使用的提示词模板。

---

## services/SessionMemory/sessionMemory.ts

**路径：** `src/services/SessionMemory/sessionMemory.ts`

**用途：** 管理持久化 session memory，跨会话存储和读取相关上下文片段。

**核心逻辑：**
- GrowthBook gate：`tengu_session_memory`
- 把 memory 存在 `~/.claude/sessions/<sessionId>/memory.jsonl`
- 用语义相似度检索相关记忆
- 作为 system prompt 追加内容整合进对话上下文

---

## services/SessionMemory/sessionMemoryUtils.ts

**路径：** `src/services/SessionMemory/sessionMemoryUtils.ts`

**用途：** session memory 操作的辅助函数（格式化、过滤、路径解析）。

---

## services/tokenEstimation.ts

**路径：** `src/services/tokenEstimation.ts`

**用途：** 不调用 API tokenizer 的粗略 token 估算。

**导出：**

```typescript
export function roughTokenCountEstimation(text: string): number
```

**核心逻辑：** 以 `text.length / 4` 近似 token 数（英文/代码大约 4 个字符一个 token）。用于 compaction 决策前的预估。

---

## services/vcr.ts

**路径：** `src/services/vcr.ts`

**用途：** VCR（录放机）测试夹具系统，用来录制和回放 API 交互，保证测试可重复。

**导出：**

```typescript
export async function withVCR<T>(
  messages: unknown[],
  f: () => Promise<T>
): Promise<T>

export async function withFixture<T>(
  input: unknown,
  fixtureName: string,
  f: () => Promise<T>
): Promise<T>

export async function withStreamingVCR<T>(
  messages: unknown[],
  f: () => Promise<T>
): Promise<T>
```

**核心逻辑：**
- 对输入做 SHA1 哈希，生成 fixture 文件名（确定性、内容寻址）
- `FORCE_VCR` 环境变量：ant 可以在测试环境外强制启用 VCR 模式
- CI 防护：如果 fixture 缺失且没有设置 `VCR_RECORD`，就会失败，避免静默漏测
- fixture 存储目录：`src/test-fixtures/vcr/`

---

## services/voice.ts

**路径：** `src/services/voice.ts`

**用途：** push-to-talk 语音输入的录音服务。支持 macOS/Linux/Windows 上的原生音频（通过 NAPI 的 cpal），Linux 上还有 SoX 和 arecord 兜底。

**常量：**
- `RECORDING_SAMPLE_RATE = 16000`
- `RECORDING_CHANNELS = 1`
- `SILENCE_DURATION_SECS = '2.0'` - SoX 静音检测
- `SILENCE_THRESHOLD = '3%'`

**导出：**

```typescript
export type RecordingAvailability = {
  available: boolean
  reason: string | null
}

export async function checkVoiceDependencies(): Promise<{
  available: boolean
  missing: string[]
  installCommand: string | null
}>

export async function requestMicrophonePermission(): Promise<boolean>

export async function checkRecordingAvailability(): Promise<RecordingAvailability>

export async function startRecording(
  onData: (chunk: Buffer) => void,
  onEnd: () => void,
  options?: { silenceDetection?: boolean }
): Promise<boolean>

export function stopRecording(): void

export function _resetArecordProbeForTesting(): void
export function _resetAlsaCardsForTesting(): void
```

**核心逻辑：**
- **后端优先级：** native（cpal via NAPI）→ arecord（ALSA，仅 Linux）→ SoX rec
- **原生模块：** `audio-capture-napi` 会在第一次按 voice keypress 时才延迟加载（dlopen 的冷启动会阻塞事件循环，大约 warm 1s / cold 8s）
- **arecord probe：** 是 memoized 的异步探测，不只检查二进制是否存在，还会验证设备能否打开；带 150ms race timer
- **Linux ALSA 保护：** 在使用 native cpal 之前先检查 `/proc/asound/cards`，避免产生多余 stderr
- **WSL 处理：** 会区分 WSL1（无音频）、Win10 WSL2（无音频）和 Win11 WSLg（PulseAudio 可用）
- SoX 参数：原始 PCM 16kHz/16-bit/mono，并带 `--buffer 1024` 以便更快 flush chunk 和做静音检测
- push-to-talk 模式：`silenceDetection: false` 会忽略原生模块由静音触发的 `onEnd`

---

## services/voiceKeyterms.ts

**路径：** `src/services/voiceKeyterms.ts`

**用途：** 为语音识别生成领域相关的词汇提示（Deepgram 的 “keywords”），提高 `voice_stream` endpoint 的 STT 准确度。

**常量：**
- `MAX_KEYTERMS = 50`
- `GLOBAL_KEYTERMS`：硬编码列表，包含 `'MCP'`、`'symlink'`、`'grep'`、`'regex'`、`'localhost'`、`'codebase'`、`'TypeScript'`、`'JSON'`、`'OAuth'`、`'webhook'`、`'gRPC'`、`'dotfiles'`、`'subagent'`、`'worktree'`

**导出：**

```typescript
export function splitIdentifier(name: string): string[]

export async function getVoiceKeyterms(
  recentFiles?: ReadonlySet<string>
): Promise<string[]>
```

**核心逻辑：**
- `splitIdentifier()`：把 camelCase / PascalCase / kebab-case / snake_case / path 标识符拆成单词；丢弃长度 ≤2 的片段
- `getVoiceKeyterms()`：把全局词条 + 项目根目录 basename + git 分支词 + 最近文件名词组合起来
- 项目根目录 basename 会保留整体，不拆分，以匹配完整项目名短语
- git 分支词通过 `splitIdentifier()` 拆分；最近文件按文件名 stem 拆分
- 总数上限是 `MAX_KEYTERMS = 50`

---

## services/voiceStreamSTT.ts

**路径：** `src/services/voiceStreamSTT.ts`

**用途：** `voice_stream` WebSocket STT 客户端。通过 OAuth 凭证连接到 `wss://api.anthropic.com/api/ws/speech_to_text/voice_stream`，用于 push-to-talk 转写。

**常量：**
- `VOICE_STREAM_PATH = '/api/ws/speech_to_text/voice_stream'`
- `KEEPALIVE_INTERVAL_MS = 8_000`
- `FINALIZE_TIMEOUTS_MS = { safety: 5_000, noData: 1_500 }`（导出给测试）
- 线协议消息：`KEEPALIVE_MSG = '{"type":"KeepAlive"}'`、`CLOSE_STREAM_MSG = '{"type":"CloseStream"}'`

**导出：**

```typescript
export const FINALIZE_TIMEOUTS_MS: { safety: number; noData: number }

export type VoiceStreamCallbacks = {
  onTranscript: (text: string, isFinal: boolean) => void
  onError: (error: string, opts?: { fatal?: boolean }) => void
  onClose: () => void
  onReady: (connection: VoiceStreamConnection) => void
}

export type FinalizeSource =
  | 'post_closestream_endpoint'
  | 'no_data_timeout'
  | 'safety_timeout'
  | 'ws_close'
  | 'ws_already_closed'

export type VoiceStreamConnection = {
  send: (audioChunk: Buffer) => void
  finalize: () => Promise<FinalizeSource>
  close: () => void
  isConnected: () => boolean
}

export function isVoiceStreamAvailable(): boolean

export async function connectVoiceStream(
  callbacks: VoiceStreamCallbacks,
  options?: { language?: string; keyterms?: string[] }
): Promise<VoiceStreamConnection | null>
```

**核心逻辑：**
- 只对 OAuth 认证用户可用；依赖 `isOAuthAuthEnabled()` 和有效 access token
- 路由到 `api.anthropic.com`，而不是 `claude.ai`，以避免 Cloudflare TLS fingerprinting 挑战
- `VOICE_STREAM_BASE_URL` 环境变量可以用于测试覆盖
- URL 参数：`encoding=linear16`、`sample_rate=16000`、`channels=1`、`endpointing_ms=300`、`utterance_end_ms=1000`
- GrowthBook gate `tengu_cobalt_frost`：通过 `use_conversation_engine=true&stt_provider=deepgram-nova3` 启用 Nova 3 STT provider
- keyterms 会以重复的 `keyterms=` 查询参数附加
- keepalive 间隔：8 秒
- `finalize()` 会发送 `CloseStream`，并在 `noData`（1.5s）、`safety`（5s）计时器和 `TranscriptEndpoint` 消息之间竞速
- 线协议消息类型：`TranscriptText`、`TranscriptEndpoint`、`TranscriptError`、`error`
- 仅限 ant 构建（受 `feature('VOICE_MODE')` gate 控制）

---

## context/QueuedMessageContext.tsx

**路径：** `src/context/QueuedMessageContext.tsx`

**用途：** 队列消息系统的 React context，负责追踪当前轮次结束后等待发送给 Claude 的消息。

**导出：**
```typescript
export const QueuedMessageContext: React.Context<QueuedMessage[]>
export function useQueuedMessages(): QueuedMessage[]
export function QueuedMessageProvider(props: { children: React.ReactNode }): JSX.Element
```

---

## context/fpsMetrics.tsx

**路径：** `src/context/fpsMetrics.tsx`

**用途：** FPS（frames per second）测量 context，用于监控 Ink 终端渲染性能。

**导出：**
```typescript
export function FpsMetricsProvider(props: { children: React.ReactNode }): JSX.Element
export function useFpsMetrics(): { fps: number; frameCount: number }
```

---

## context/mailbox.tsx

**路径：** `src/context/mailbox.tsx`

**用途：** 提供智能体之间的消息上下文，也就是用于接收其他 agent/worker 消息的 “mailbox”。

**导出：**
```typescript
export type MailboxMessage = { from: string; content: string; timestamp: number }
export const MailboxContext: React.Context<MailboxMessage[]>
export function MailboxProvider(props: { children: React.ReactNode }): JSX.Element
export function useMailbox(): MailboxMessage[]
```

---

## context/modalContext.tsx

**路径：** `src/context/modalContext.tsx`

**用途：** 管理 modal dialog 状态的 context，追踪当前打开的是哪个 modal，并提供 open/close 动作。

**导出：**
```typescript
export type ModalState = { isOpen: boolean; content: React.ReactNode | null }
export const ModalContext: React.Context<ModalState>
export function ModalProvider(props: { children: React.ReactNode }): JSX.Element
export function useModal(): { open: (content: React.ReactNode) => void; close: () => void }
```

---

## context/notifications.tsx

**路径：** `src/context/notifications.tsx`

**用途：** 通知队列管理。它会在状态栏里显示 toast 风格通知，并支持优先级排序、去重、折叠/合并和超时处理。

**类型：**

```typescript
type Priority = 'low' | 'medium' | 'high' | 'immediate'

type BaseNotification = {
  key: string
  invalidates?: string[]
  priority: Priority
  timeoutMs?: number
  fold?: (accumulator: Notification, incoming: Notification) => Notification
}

type TextNotification = BaseNotification & { text: string; color?: keyof Theme }
type JSXNotification = BaseNotification & { jsx: React.ReactNode }
export type Notification = TextNotification | JSXNotification
```

**导出：**

```typescript
const DEFAULT_TIMEOUT_MS = 8000

export function useNotifications(): {
  addNotification: (content: Notification) => void
  removeNotification: (key: string) => void
}
```

**核心逻辑：**
- 通知状态存在 `AppState` 里（`notifications.current` + `notifications.queue`）
- `immediate` 优先级：绕过队列，立即显示，并把当前非 immediate 通知重新排队
- `fold` 函数：把相同 key 的通知合并（累加器模式）
- 去重：队列和 current 中同一个 key 只保留一条
- `invalidates[]`：删除队列里指定 key 的通知，如果当前通知匹配也会清掉
- `DEFAULT_TIMEOUT_MS = 8000`（8 秒）；超时后自动切到下一条队列通知
- 模块级 `currentTimeoutId` 跟踪当前自动关闭计时器

---

## context/overlayContext.tsx

**路径：** `src/context/overlayContext.tsx`

**用途：** 用于 Escape 键协调的 overlay 跟踪。它会记录当前打开的 overlays（对话框、选择器等），避免 cancel handler 把 Escape 误判成别的操作。

**常量：**
```typescript
const NON_MODAL_OVERLAYS = new Set(['autocomplete'])
```

**导出：**

```typescript
export function useRegisterOverlay(id: string, enabled?: boolean): void
export function useIsOverlayActive(): boolean
export function useIsModalOverlayActive(): boolean
```

**核心逻辑：**
- 状态保存在 `AppState.activeOverlays: Set<string>`
- `useRegisterOverlay()`：在 mount 时注册（`useEffect`），在 unmount 时通过 cleanup 注销
- overlay 关闭时：通过 `instances.get(process.stdout)?.invalidatePrevFrame()`（via `useLayoutEffect`）强制 full-damage diff，避免高 overlay（例如 FuzzyPicker）留下 ghost cells
- `useIsOverlayActive()`：`activeOverlays.size > 0`
- `useIsModalOverlayActive()`：集合中任一不在 `NON_MODAL_OVERLAYS` 的 overlay 都算

---

## context/promptOverlayContext.tsx

**路径：** `src/context/promptOverlayContext.tsx`

**用途：** prompt 级 overlay 管理的 context，追踪那些会影响 prompt 输入焦点和行为的 overlay。

**导出：**
```typescript
export function useRegisterPromptOverlay(id: string, enabled?: boolean): void
export function useIsPromptOverlayActive(): boolean
```

---

## context/stats.tsx

**路径：** `src/context/stats.tsx`

**用途：** 进程内性能指标存储，直方图使用 reservoir sampling。进程退出时会把指标持久化到项目配置里。

**常量：**
- `RESERVOIR_SIZE = 1024` - 直方图的 reservoir sampling 容量

**类型：**

```typescript
export type StatsStore = {
  increment(name: string, value?: number): void
  set(name: string, value: number): void
  observe(name: string, value: number): void
  add(name: string, value: string): void
  getAll(): Record<string, number>
}
```

**导出：**

```typescript
export function createStatsStore(): StatsStore
export const StatsContext: React.Context<StatsStore | null>
export function StatsProvider(props: { store?: StatsStore; children: React.ReactNode }): JSX.Element
export function useStats(): StatsStore
export function useCounter(name: string): (value?: number) => void
export function useGauge(name: string): (value: number) => void
export function useTimer(name: string): (value: number) => void
// ... 还有更多 hook 导出
```

**核心逻辑：**
- `createStatsStore()`：创建一个 stats store，内部有三种结构：
  - `metrics: Map<string, number>` - 计数器和 gauge
  - `histograms: Map<string, Histogram>` - reservoir-sampled 分布
  - `sets: Map<string, Set<string>>` - 唯一值集合（按 `.size` 报告）
- `observe()`：用 reservoir sampling（Algorithm R）处理直方图，容量为 `RESERVOIR_SIZE = 1024`
- `getAll()`：返回扁平的 `Record<string, number>`，包含直方图百分位数（`_p50`、`_p95`、`_p99`）、min、max、avg、count
- `StatsProvider`：会在进程 `'exit'` 事件上把指标 flush 到项目配置中的 `lastSessionMetrics`
- `useCounter()` / `useGauge()` / `useTimer()`：返回绑定好的 store 方法，并做 memoization

---

## context/voice.tsx

**路径：** `src/context/voice.tsx`

**用途：** 语音模式 context，把录音状态、转写内容和连接状态提供给 UI。

**导出：**

```typescript
export type VoiceContextValue = {
  isRecording: boolean
  transcript: string
  isConnecting: boolean
  error: string | null
  startRecording: () => void
  stopRecording: () => void
}

export const VoiceContext: React.Context<VoiceContextValue>
export function VoiceProvider(props: { children: React.ReactNode }): JSX.Element
export function useVoice(): VoiceContextValue
```

---

## screens/Doctor.tsx

**路径：** `src/screens/Doctor.tsx`

**用途：** `/doctor` 命令的 UI 屏幕，展示系统健康诊断，包括版本信息、环境检查、MCP server 状态、沙盒状态、键位警告和可用更新。

**参数：**
```typescript
type Props = {
  onDone: () => void
}
```

**内部类型：**
```typescript
type AgentInfo = {
  name: string
  version: string
  // ...
}

type VersionLockInfo = {
  locked: boolean
  version?: string
  // ...
}
```

**核心逻辑：**
- 使用 `getDoctorDiagnostic()` 做环境检查
- 调用 `checkContextWarnings()` 查上下文相关问题
- 通过 `getNpmDistTags()` 和 `getGcsDistTags()` 读取 dist tags（包在 Suspense 里）
- 渲染多个子区块：`SandboxDoctorSection`、`ValidationErrorsList`、`KeybindingWarnings`、`McpParsingWarnings`
- 显示版本对比：当前版本 vs 最新 npm/GCS 版本
- 展示 agent 信息、版本锁定状态和更新通道

---

## screens/REPL.tsx

**路径：** `src/screens/REPL.tsx`

**用途：** 主交互式 REPL 屏幕，也就是整套 Claude Code 交互会话的核心对话 UI。

**核心逻辑：**
- 管理完整会话生命周期：用户输入 → API 查询 → 工具执行 → 响应展示
- 处理所有交互功能：对话历史、compaction、clear、文件编辑、权限
- 把 slash command 路由到对应命令 handler
- 管理 modal dialog（权限、MCP 审批等）
- 接入所有 context provider：notifications、overlays、voice、stats
- 这是代码库里最大的组件，负责整体用户交互循环

---

## screens/ResumeConversation.tsx

**路径：** `src/screens/ResumeConversation.tsx`

**用途：** 会话恢复流程的 UI。它会展示最近会话列表和预览，并允许用户选中一个会话继续。

**核心逻辑：**
- 从磁盘加载 session index
- 为每个 session 渲染 `SessionPreview` 组件
- 处理键盘导航（方向键、Enter 选择）
- 按 project root 过滤会话，或者显示全部
- 通过回调把选中的 session ID 传回调用方

---

## 跨章节架构说明

### 状态管理架构

代码库里有三层不同的状态：

1. **`bootstrap/state.ts`** - 全局可变单例，用于 session 级状态（成本、模型、遥测）。它故意是 DAG 的叶子节点，不从 services 导入任何内容。

2. **React `AppState`** - 通过 Zustand-like store（`state/AppState.ts`）管理 UI 状态。通过 `useAppState()` 选择器和 `useSetAppState()` 访问。

3. **React Contexts** - 面向具体领域的 context（notifications、overlays、stats、voice），用于子树范围内的状态。

### Analytics 架构

事件会经过一个多层流水线：

```text
logEvent() → index.ts 队列 → sink.ts 分发
  → [sampling 检查] → [isSinkKilled 检查]
  → datadog.ts（剥离 _PROTO_ 键）
  → firstPartyEventLogger.ts（把 _PROTO_ 键提升到 proto 字段）
```

当以下条件成立时，所有 analytics 都会被禁用：`NODE_ENV === test`、3P providers（Bedrock/Vertex/Foundry），或者 `isTelemetryDisabled()` 返回 true。

### Compact / Microcompact 架构

上下文压缩分成多层：

1. **`apiMicrocompact.ts`** - API 原生、服务端编辑（cache_edits），不改动客户端内容
2. **`microCompact.ts`** - 客户端工具结果清理（缓存路径 + 时间路径）
3. **`autoCompact.ts`** - 基于阈值触发的完整对话总结
4. **`compact.ts`** - 完整总结实现（LLM 调用）
5. **`sessionMemoryCompact.ts`** - 更轻的 session memory 段总结

### GrowthBook Feature Flag 命名约定

所有 feature flag 都使用带 `tengu_` 前缀的混淆/编码名字，以避免被抓取：
- Feature flags：`tengu_prompt_cache_1h_config`、`tengu_session_memory`、`tengu_sm_compact` 等
- Kill switches：`tengu_frond_boric`（analytics sink killswitch）
- Voice：`tengu_cobalt_frost`（Nova 3 STT）
- AutoDream：`tengu_onyx_plover`
- Coordinator mode：通过 `feature('COORDINATOR_MODE')` bundle flag 检查
