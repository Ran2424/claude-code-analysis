# Claude Code — 命令参考

本文档完整整理了 Claude Code CLI 中的所有斜杠命令，内容直接来自 `src/commands/` 源码和顶层注册表 `src/commands.ts`。

---

## 目录

1. [命令系统架构](#1-command-system-architecture)
2. [命令类型定义](#2-command-type-definitions)
3. [命令注册表 (`commands.ts`)](#3-command-registry-commandsts)
4. [单个命令参考](#4-individual-command-reference)
   - [add-dir](#add-dir)
   - [advisor](#advisor)
   - [agents](#agents)
   - [ant-trace](#ant-trace-stub)
   - [autofix-pr](#autofix-pr-stub)
   - [backfill-sessions](#backfill-sessions-stub)
   - [branch / fork](#branch--fork)
   - [break-cache](#break-cache-stub)
   - [bridge-kick](#bridge-kick)
   - [bridge / remote-control / rc](#bridge--remote-control--rc)
   - [brief](#brief)
   - [btw](#btw)
   - [bughunter](#bughunter-stub)
   - [chrome](#chrome)
   - [clear / reset / new](#clear--reset--new)
   - [color](#color)
   - [commit](#commit)
   - [commit-push-pr](#commit-push-pr)
   - [compact](#compact)
   - [config / settings](#config--settings)
   - [context](#context)
   - [copy](#copy)
   - [cost](#cost)
   - [createMovedToPluginCommand (utility)](#createmovedtoplugincommand-utility)
   - [ctx_viz](#ctx_viz-stub)
   - [debug-tool-call](#debug-tool-call-stub)
   - [desktop / app](#desktop--app)
   - [diff](#diff)
   - [doctor](#doctor)
   - [effort](#effort)
   - [env](#env-stub)
   - [exit / quit](#exit--quit)
   - [export](#export)
   - [extra-usage](#extra-usage)
   - [fast](#fast)
   - [feedback / bug](#feedback--bug)
   - [files](#files)
   - [good-claude](#good-claude-stub)
   - [heapdump](#heapdump)
   - [help](#help)
   - [hooks](#hooks)
   - [ide](#ide)
   - [init](#init)
   - [init-verifiers](#init-verifiers)
   - [insights](#insights)
   - [install-github-app](#install-github-app)
   - [install-slack-app](#install-slack-app)
   - [install (component)](#install-component)
   - [issue](#issue-stub)
   - [keybindings](#keybindings)
   - [login](#login)
   - [logout](#logout)
   - [mcp](#mcp)
   - [memory](#memory)
   - [mobile / ios / android](#mobile--ios--android)
   - [mock-limits](#mock-limits-stub)
   - [model](#model)
   - [oauth-refresh](#oauth-refresh-stub)
   - [onboarding](#onboarding-stub)
   - [output-style](#output-style)
   - [passes](#passes)
   - [perf-issue](#perf-issue-stub)
   - [permissions / allowed-tools](#permissions--allowed-tools)
   - [plan](#plan)
   - [plugin / plugins / marketplace](#plugin--plugins--marketplace)
   - [pr-comments](#pr-comments)
   - [privacy-settings](#privacy-settings)
   - [rate-limit-options](#rate-limit-options)
   - [release-notes](#release-notes)
   - [reload-plugins](#reload-plugins)
   - [remote-env](#remote-env)
   - [remote-setup / web-setup](#remote-setup--web-setup)
   - [rename](#rename)
   - [reset-limits](#reset-limits-stub)
   - [resume / continue](#resume--continue)
   - [review](#review)
   - [ultrareview](#ultrareview)
   - [rewind / checkpoint](#rewind--checkpoint)
   - [sandbox](#sandbox)
   - [security-review](#security-review)
   - [session / remote](#session--remote)
   - [share](#share-stub)
   - [skills](#skills)
   - [stats](#stats)
   - [status](#status)
   - [statusline](#statusline)
   - [stickers](#stickers)
   - [summary](#summary-stub)
   - [tag](#tag)
   - [tasks / bashes](#tasks--bashes)
   - [teleport](#teleport-stub)
   - [terminal-setup](#terminal-setup)
   - [theme](#theme)
   - [think-back](#think-back)
   - [thinkback-play](#thinkback-play)
   - [ultraplan](#ultraplan)
   - [upgrade](#upgrade)
   - [usage](#usage)
   - [version](#version)
   - [vim](#vim)
   - [voice](#voice)
5. [仅内部命令汇总](#5-internal-only-commands-summary)
6. [远程安全命令](#6-remote-safe-commands)
7. [桥接安全命令](#7-bridge-safe-commands)
8. [命令可用性与特性标志](#8-command-availability--feature-flags)

---

## 1. 命令系统架构

All slash commands in Claude Code share a unified `Command` interface defined in `src/types/command.ts` and re-exported from `src/commands.ts`. The command registry is loaded lazily (memoized) in `commands.ts` and is assembled from several sources:

```
Priority order in getCommands():
  bundledSkills → builtinPluginSkills → skillDirCommands →
  workflowCommands → pluginCommands → pluginSkills → COMMANDS()
```

命令共有三种类型：

| Type | Description |
|------|-------------|
| `'local'` | Runs synchronously; returns `LocalCommandResult` (`{ type: 'text' \| 'compact' \| 'skip', value?: string }`) |
| `'local-jsx'` | Renders a React/Ink component in the TUI; returns `React.ReactNode` |
| `'prompt'` | Expands to a text prompt that is sent to the model via the main loop |

### 懒加载

每个命令的实现都通过 `load()` 方法里的动态 `import()` 加载。索引文件只保留元数据（名称、说明、类型），以尽量降低启动开销。只有命令第一次被调用时，才会加载较重的模块。

---

## 2. 命令类型定义

```typescript
// Base for all command types
interface CommandBase {
  name: string
  description: string
  aliases?: string[]
  argumentHint?: string
  isEnabled?: () => boolean
  isHidden?: boolean
  availability?: Array<'claude-ai' | 'console'>
  source?: 'builtin' | 'plugin' | 'mcp' | 'bundled' | string
  load: () => Promise<{ call: Function }>
}

// 'local' command — synchronous, returns text
interface LocalCommand extends CommandBase {
  type: 'local'
  supportsNonInteractive?: boolean
}

// 'local-jsx' command — renders Ink UI
interface LocalJSXCommand extends CommandBase {
  type: 'local-jsx'
  immediate?: boolean   // skip the "processing" spinner
}

// 'prompt' command — expanded and sent to the model
interface PromptCommand extends CommandBase {
  type: 'prompt'
  contentLength: number
  progressMessage: string
  allowedTools?: string[]
  disableModelInvocation?: boolean
  disableNonInteractive?: boolean
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
}
```

---

## 3. 命令注册表 (`commands.ts`)

**文件:** `src/commands.ts`

### 导出

| Export | Description |
|--------|-------------|
| `getCommands(cwd)` | Async; returns all commands filtered by `meetsAvailabilityRequirement` and `isCommandEnabled`. Memoized per-cwd for expensive loading; availability/isEnabled run fresh every call. |
| `INTERNAL_ONLY_COMMANDS` | Array of commands only available when `USER_TYPE === 'ant'`. Includes backfill-sessions, break-cache, bughunter, commit, commit-push-pr, ctx_viz, good-claude, issue, init-verifiers, mock-limits, bridge-kick, version, ultraplan, subscribePr, resetLimits, onboarding, share, summary, teleport, ant-trace, perf-issue, env, oauth-refresh, debug-tool-call, agents-platform, autofix-pr. |
| `REMOTE_SAFE_COMMANDS` | `Set<Command>` of commands safe in remote (--remote) mode: session, exit, clear, help, theme, color, vim, cost, usage, copy, btw, feedback, plan, keybindings, statusline, stickers, mobile. |
| `BRIDGE_SAFE_COMMANDS` | `Set<Command>` of `'local'` commands safe over the Remote Control bridge: compact, clear, cost, summary, release-notes, files. |
| `builtInCommandNames` | Memoized `Set<string>` of all built-in command names and aliases. |
| `meetsAvailabilityRequirement(cmd)` | Checks `cmd.availability` against current auth state. |
| `filterCommandsForRemoteMode(commands)` | Filters to only REMOTE_SAFE_COMMANDS. |
| `isBridgeSafeCommand(cmd)` | Returns `true` for `'prompt'` type or BRIDGE_SAFE_COMMANDS members; `false` for `'local-jsx'`. |
| `findCommand(name, commands)` | Lookup by name, computed name, or alias. |
| `hasCommand(name, commands)` | Boolean version of findCommand. |
| `getCommand(name, commands)` | Like findCommand but throws `ReferenceError` if not found. |
| `formatDescriptionWithSource(cmd)` | User-facing description with source annotation (plugin name, bundled, skill dir). |
| `getSkillToolCommands(cwd)` | Memoized; filters to prompt-type commands the model can invoke as tools. |
| `getSlashCommandToolSkills(cwd)` | Memoized; filters to true skills (skills/plugin/bundled). |
| `getMcpSkillCommands(mcpCommands)` | Filters MCP commands to model-invocable prompt skills. |
| `clearCommandMemoizationCaches()` | Clears loadAllCommands and skill index caches. |
| `clearCommandsCache()` | Full reset including plugin and skill caches. |

### 特性门控命令

The following commands are conditionally registered using `feature()` bundle flags:

| Feature Flag | Command |
|---|---|
| `BRIDGE_MODE` | `/remote-control` (rc) |
| `BRIDGE_MODE` + `DAEMON` | `remoteControlServer` |
| `KAIROS` or `KAIROS_BRIEF` | `/brief` |
| `KAIROS` | `assistant` |
| `VOICE_MODE` | `/voice` |
| `HISTORY_SNIP` | `force-snip` |
| `WORKFLOW_SCRIPTS` | `workflows` |
| `CCR_REMOTE_SETUP` | `/web-setup` |
| `ULTRAPLAN` | `/ultraplan` |
| `KAIROS_GITHUB_WEBHOOKS` | `subscribe-pr` |
| `TORCH` | `torch` |
| `UDS_INBOX` | `peers` |
| `FORK_SUBAGENT` | `fork` |
| `BUDDY` | `buddy` |
| `PROACTIVE` or `KAIROS` | `proactive` |
| `EXPERIMENTAL_SKILL_SEARCH` | clearSkillIndexCache hook |

---

## 4. 单个命令参考

---

### `/add-dir`

**文件:** `commands/add-dir/index.ts`, `commands/add-dir/add-dir.tsx`, `commands/add-dir/validation.ts`

**类型:** `local-jsx`
**语法:** `/add-dir [<path>]`
**说明:** 向当前会话的权限上下文添加一个新的工作目录。

**行为:**
- If a path argument is provided, validates it and adds it immediately (with an optional "remember" step prompting to save to local settings).
- If no path is provided, opens the `AddWorkspaceDirectory` interactive picker component.
- On success, calls `applyPermissionUpdate` (session) and optionally `persistPermissionUpdate` (local settings).
- Also calls `SandboxManager.refreshConfig()` to update bash sandboxing configuration.
- Updates bootstrap state for additional CLAUDE.md directories.

**校验（validation.ts）：**

```typescript
export type AddDirectoryResult =
  | { resultType: 'success'; absolutePath: string }
  | { resultType: 'emptyPath' }
  | { resultType: 'pathNotFound' | 'notADirectory'; directoryPath: string; absolutePath: string }
  | { resultType: 'alreadyInWorkingDirectory'; directoryPath: string; workingDir: string }

export async function validateDirectoryForWorkspace(
  directoryPath: string,
  permissionContext: ToolPermissionContext,
): Promise<AddDirectoryResult>

export function addDirHelpMessage(result: AddDirectoryResult): string
```

**关键逻辑：**
- Expands `~` and resolves symlinks before validation.
- Handles `ENOENT`, `ENOTDIR`, `EACCES`, `EPERM` gracefully (returns `pathNotFound` rather than throwing).
- Checks `pathInWorkingPath` to avoid adding redundant subdirectories.

**依赖:** `chalk`, `bootstrap/state.js`, `components/permissions/rules/AddWorkspaceDirectory`, `utils/permissions/PermissionUpdate.js`, `utils/sandbox/sandbox-adapter.js`

---

### `/advisor`

**文件:** `commands/advisor.ts`

**类型:** `local`
**语法:** `/advisor [<model>|off|unset]`
**说明:** 配置 advisor model（一个用于审阅和建议主模型输出的辅助模型）。

**Arguments:**

| Argument | Description |
|----------|-------------|
| *(none)* | Display current advisor setting and status |
| `<model>` | Set the advisor model (e.g. `opus`, `claude-opus-4-5`) |
| `off` / `unset` | Disable the advisor |

**行为:**
- Checks `canUserConfigureAdvisor()` — command is hidden/disabled if user cannot configure an advisor.
- Validates model string via `validateModel()`.
- Checks `isValidAdvisorModel()` and `modelSupportsAdvisor()`.
- Persists to `userSettings.advisorModel` via `updateSettingsForSource`.
- If base model doesn't support advisors, returns a warning (still sets the value).

**门禁:** `isEnabled: () => canUserConfigureAdvisor()`

**依赖:** `utils/advisor.js`, `utils/model/model.js`, `utils/model/validateModel.js`, `utils/settings/settings.js`

---

### `/agents`

**文件:** `commands/agents/index.ts`, `commands/agents/agents.tsx`

**类型:** `local-jsx`
**语法:** `/agents`
**说明:** 打开 Agents 管理菜单（创建、编辑、删除自定义 agent 配置）。

**行为:** Renders the `AgentsMenu` component passing current tools from `getTools(permissionContext)`.

**依赖:** `components/agents/AgentsMenu`, `tools.js`

---

### `/ant-trace`（占位）

**文件:** `commands/ant-trace/index.js`

**类型:** Stub — `isEnabled: () => false, isHidden: true`
**说明:** 内部追踪命令，在所有外部构建中都禁用。

---

### `/autofix-pr`（占位）

**文件:** `commands/autofix-pr/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/backfill-sessions`（占位）

**文件:** `commands/backfill-sessions/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/branch` / `/fork`

**文件:** `commands/branch/index.ts`, `commands/branch/branch.ts`

**类型:** `local-jsx`
**语法:** `/branch [name]`
**说明:** 在当前位置创建当前对话的分支副本（fork）。用户会立即进入这个分支；原会话可通过 `claude -r <original-session-id>` 继续。

**Arguments:**

| Argument | Description |
|----------|-------------|
| `[name]` | Optional custom title for the branch (defaults to first-prompt-derived kebab title + " (Branch)") |

**别名:** When the `FORK_SUBAGENT` feature flag is off, also registered as `/fork`.

**行为:**
1. Reads the current transcript JSONL file.
2. Generates a new UUID for the fork session.
3. Copies all non-sidechain messages, rewriting `sessionId` and adding `forkedFrom` traceability metadata to each entry.
4. Copies `content-replacement` entries (rewriting `sessionId`) to preserve prompt-cache budgeting.
5. Saves the fork to a new JSONL file.
6. Calls `getUniqueForkName()` to avoid collisions (appends " (Branch 2)", " (Branch 3)", etc.).
7. Calls `context.resume(sessionId, forkLog, 'fork')` to switch the live session into the fork.

**导出：**
```typescript
export function deriveFirstPrompt(firstUserMessage): string
export async function call(onDone, context, args): Promise<React.ReactNode>
```

**依赖:** `crypto`, `fs/promises`, `bootstrap/state.js`, `utils/sessionStorage.js`, `services/analytics/index.js`

---

### `/break-cache`（占位）

**文件:** `commands/break-cache/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/bridge-kick`

**文件:** `commands/bridge-kick.ts`

**类型:** `local`
**语法:** `/bridge-kick <subcommand> [args...]`
**说明:** Internal command to inject bridge (Remote Control) failure states for manual recovery path testing. Only enabled when `USER_TYPE === 'ant'`.

**Subcommands:**

| Subcommand | Arguments | Description |
|---|---|---|
| `close` | `<code>` | Fire WebSocket close event with given code (e.g. 1002, 1006) |
| `poll` | `<status>` or `transient` | Next poll throws BridgeFatalError(status) or a transient rejection |
| `poll` | `<status> <type>` | Poll error with explicit error_type |
| `register` | `fail [N]` | Next N registerBridgeEnvironment calls transient-fail (default 1) |
| `register` | `fatal` | Next register 403s (terminal failure) |
| `reconnect-session` | `fail` | Next 2 POST /bridge/reconnect calls return 404 |
| `heartbeat` | `<status>` | Next heartbeat throws BridgeFatalError(status) |
| `reconnect` | *(none)* | Call reconnectEnvironmentWithSession() directly |
| `status` | *(none)* | Print current bridge state |

**门禁:** `isEnabled: () => process.env.USER_TYPE === 'ant'`

**依赖:** `bridge/bridgeDebug.js`

---

### `/bridge` / `/remote-control` / `/rc`

**文件:** `commands/bridge/index.ts`, `commands/bridge/bridge.tsx`

**类型:** `local-jsx`
**Name:** `remote-control`
**别名:** `rc`
**语法:** `/remote-control [name]`
**说明:** 启动或管理 Remote Control bridge 会话，让终端可以从移动设备或 Web 界面控制。

**特性门禁:** `BRIDGE_MODE` feature flag must be on AND `isBridgeEnabled()` must return true.

The implementation (`bridge.tsx`) is a large file (~34KB) rendering a multi-step wizard for QR code display, session naming, connection state, and error recovery.

**Arguments:**

| Argument | Description |
|----------|-------------|
| `[name]` | Optional session name to display in the remote client |

**属性:**
- `immediate: true` — renders immediately without spinner
- `isHidden`: mirrors `isEnabled()`

---

### `/brief`

**文件:** `commands/brief.ts`

**类型:** `local-jsx`
**语法:** `/brief`
**说明:** 切换“仅 brief 模式”。开启后，模型对所有面向用户的输出都必须使用 `SendUserMessage`（Brief）工具，普通文本会被隐藏。由 `KAIROS` 或 `KAIROS_BRIEF` bundle 标志，以及 GrowthBook 配置 `tengu_kairos_brief_config.enable_slash_command` 门控。

**行为:**
1. Checks entitlement via `isBriefEntitled()` — if trying to enable and not entitled, logs analytics and returns an error.
2. Toggles `setUserMsgOptIn` and `context.setAppState({ isBriefOnly })`.
3. Injects a `<system-reminder>` meta-message into the next turn (unless Kairos is active) so the model immediately transitions its output style.
4. Logs `tengu_brief_mode_toggled` event.

**属性:** `immediate: true`

---

### `/btw`

**文件:** `commands/btw/index.ts`, `commands/btw/btw.tsx`

**类型:** `local-jsx`
**语法:** `/btw <question>`
**说明:** Ask a quick "by the way" side question without interrupting the main conversation. Opens a scrollable overlay with a spinner, sends the question as a forked side-query to the model, and returns the response in a scrollable panel. Pressing Enter/Escape/Space dismisses the result.

**属性:** `immediate: true` — renders without spinner.

**行为:**
- Uses `runSideQuestion()` to send the question with an abbreviated context (messages after the compact boundary, with cache-safe params from the last turn).
- Renders markdown response in a `ScrollBox`.
- Keyboard: `Up`/`Down` or `Ctrl+P`/`Ctrl+N` to scroll; `Escape`/`Enter`/`Space`/`Ctrl+C`/`Ctrl+D` to dismiss.

**依赖:** `utils/sideQuestion.js`, `components/Markdown`, `ink/components/ScrollBox`

---

### `/bughunter`（占位）

**文件:** `commands/bughunter/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/chrome`

**文件:** `commands/chrome/index.ts`, `commands/chrome/chrome.tsx`

**类型:** `local-jsx`
**语法:** `/chrome`
**说明:** Claude in Chrome (Beta) settings. Available only to `claude-ai` subscribers and non-interactive session check.

**门禁:** `availability: ['claude-ai']`, `isEnabled: () => !getIsNonInteractiveSession()`

The implementation (`chrome.tsx`, ~23KB) renders a settings panel for managing the Claude Chrome extension integration.

---

### `/clear` / `/reset` / `/new`

**文件:** `commands/clear/index.ts`, `commands/clear/clear.ts`, `commands/clear/conversation.ts`, `commands/clear/caches.ts`

**类型:** `local`
**语法:** `/clear`
**别名:** `reset`, `new`
**说明:** Clear conversation history and free up context. Starts a fresh session.

**核心函数（`clearConversation`）：**
1. Executes `SessionEnd` hooks (bounded by `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS`, default 1.5s).
2. Emits `tengu_cache_eviction_hint` analytics event.
3. Identifies background tasks to preserve (tasks with `isBackgrounded !== false`).
4. Clears messages: `setMessages(() => [])`.
5. Calls `clearSessionCaches(preservedAgentIds)`.
6. Resets cwd to original cwd, clears `readFileState`, `discoveredSkillNames`, `loadedNestedMemoryPaths`.
7. Kills/aborts foreground tasks (local shell tasks get `kill()`+`cleanup()`; agent tasks get `abortController.abort()`).
8. Clears attribution state, file history, and MCP client state in AppState.
9. Clears plan slugs and session metadata.
10. Regenerates session ID (sets old as parent for analytics).
11. Re-points task output symlinks for surviving background tasks.
12. Persists mode (coordinator/normal) and worktree state.
13. Executes `SessionStart` hooks after clearing.

**`clearSessionCaches()` (caches.ts):** Clears user/system/git context caches, file suggestions, commands/skills cache, prompt cache break detection, system prompt injection, emitted date, post-compact cleanup, skill names, memory files cache, stored images, session ingress, pending permission callbacks, tungsten session tracking, attribution caches, repository caches, bash command prefix caches, dump prompts state, invoked skills, git dir resolution, dynamic skills, LSP diagnostics, magic docs tracking, session env vars, WebFetch URL cache, ToolSearch description cache, agent definitions cache, SkillTool prompt cache.

**门禁:** `supportsNonInteractive: false`

---

### `/color`

**文件:** `commands/color/index.ts`, `commands/color/color.ts`

**类型:** `local-jsx`
**语法:** `/color <color|default>`
**说明:** Set the prompt bar / session color. Colors are drawn from `AGENT_COLORS` (the same palette used by swarm agents).

**Arguments:**

| Argument | Description |
|----------|-------------|
| `<color>` | A named color from `AGENT_COLORS` |
| `default` / `reset` / `none` / `gray` / `grey` | Reset to default (no color) |

**行为:**
- Saves color to transcript via `saveAgentColor()`.
- Updates `AppState.standaloneAgentContext.color` for immediate display.
- Blocked for swarm teammates (their colors are assigned by the team leader).

**属性:** `immediate: true`

---

### `/commit`

**文件:** `commands/commit.ts`

**类型:** `prompt`
**语法:** `/commit`
**说明:** Create a git commit using the current staged and unstaged changes.

**Allowed tools:**
```
Bash(git add:*)
Bash(git status:*)
Bash(git commit:*)
```

**Prompt behavior:**
- Injects live shell output via `executeShellCommandsInPrompt` for: `git status`, `git diff HEAD`, `git branch --show-current`, `git log --oneline -10`.
- Provides a Git Safety Protocol in the prompt (never amend, never skip hooks, etc.).
- If `USER_TYPE === 'ant'` and `isUndercover()`, prepends undercover instructions.
- Includes optional commit attribution text from `getAttributionTexts()`.
- Instructs model to create a commit using HEREDOC syntax.

**门禁:** Internal-only (`INTERNAL_ONLY_COMMANDS`).

---

### `/commit-push-pr`

**文件:** `commands/commit-push-pr.ts`

**类型:** `prompt`
**语法:** `/commit-push-pr [additional instructions]`
**说明:** Commit, push, and open a pull request in a single operation.

**Allowed tools:**
```
Bash(git checkout --branch:*)
Bash(git checkout -b:*)
Bash(git add:*)
Bash(git status:*)
Bash(git push:*)
Bash(git commit:*)
Bash(gh pr create:*)
Bash(gh pr edit:*)
Bash(gh pr view:*)
Bash(gh pr merge:*)
ToolSearch
mcp__slack__send_message
mcp__claude_ai_Slack__slack_send_message
```

**Prompt behavior:**
- Injects live shell output for: `git status`, `git diff HEAD`, `git branch --show-current`, `git diff ${defaultBranch}...HEAD`, `gh pr view --json number`.
- Orchestrates: create branch if on main, commit, push, create/update PR.
- Optionally posts to Slack if CLAUDE.md mentions Slack channels (uses ToolSearch).
- PR body template includes Summary, Test plan, Changelog sections.
- Includes `getEnhancedPRAttribution()` text.

**门禁:** Internal-only (`INTERNAL_ONLY_COMMANDS`).

---

### `/compact`

**文件:** `commands/compact/index.ts`, `commands/compact/compact.ts`

**类型:** `local`
**语法:** `/compact [custom summarization instructions]`
**说明:** Summarize the conversation, replacing message history with a compact summary. Preserves context while reducing token usage.

**Arguments:**

| Argument | Description |
|----------|-------------|
| `[instructions]` | Optional custom instructions to guide the compaction summary |

**行为:**
1. Filters messages to those after the compact boundary (strips UI-only scrollback).
2. If no custom instructions, first tries `trySessionMemoryCompaction()` (session-memory-based, cheaper).
3. If `REACTIVE_COMPACT` flag is on, routes through the reactive compaction path.
4. Otherwise falls back to traditional compaction:
   - Runs `microcompactMessages()` first to reduce tokens.
   - Calls `compactConversation()` with cache-sharing parameters.
5. After success: clears user context cache, runs `runPostCompactCleanup()`, suppresses the compact warning.
6. Returns `{ type: 'compact', compactionResult, displayText }`.

**门禁:** `isEnabled: () => !isEnvTruthy(process.env.DISABLE_COMPACT)`, `supportsNonInteractive: true`

---

### `/config` / `/settings`

**文件:** `commands/config/index.ts`, `commands/config/config.tsx`

**类型:** `local-jsx`
**语法:** `/config`
**别名:** `settings`
**说明:** Open the interactive configuration panel (`Settings` component, defaulting to the "Config" tab).

---

### `/context`

**文件:** `commands/context/index.ts`, `commands/context/context-noninteractive.ts`, `commands/context/context.tsx`

**类型:** `local-jsx` (interactive) or `local` (non-interactive)
**语法:** `/context`
**说明:** Visualize or display current context usage.

**Two variants:**
- **Interactive** (`context`): `local-jsx`, renders a colored visualization grid. Enabled when `!getIsNonInteractiveSession()`.
- **Non-interactive** (`contextNonInteractive`): `local`, outputs a markdown table with token breakdown. Enabled when `getIsNonInteractiveSession()`.

**Output (non-interactive markdown table includes):**
- Total tokens / max tokens / percentage
- Context strategy (collapse stats if `CONTEXT_COLLAPSE` is on)
- Estimated usage by category
- MCP Tools breakdown
- ANT-ONLY: System Tools, System Prompt Sections
- Custom Agents (by source: Project/User/Local/Flag/Policy/Plugin/Built-in)
- Memory Files (type, path, tokens)
- Skills (name, source, tokens)
- ANT-ONLY: Message Breakdown (tool calls, tool results, attachments, assistant messages, user messages; top tools by call+result tokens; top attachments)

**核心函数：**
```typescript
export async function collectContextData(context): Promise<ContextData>
export async function call(_args, context): Promise<{ type: 'text'; value: string }>
```

---

### `/copy`

**文件:** `commands/copy/index.ts`, `commands/copy/copy.tsx`

**类型:** `local-jsx`
**语法:** `/copy [N]`
**说明:** Copy Claude's last response to the clipboard. `/copy N` copies the Nth-latest response.

The implementation (`copy.tsx`, ~31KB) handles clipboard access, finds the most recent assistant message(s), and copies their text content.

---

### `/cost`

**文件:** `commands/cost/index.ts`, `commands/cost/cost.ts`

**类型:** `local`
**语法:** `/cost`
**说明:** Show the total cost and duration of the current session.

**行为:**
- For `claude-ai` subscribers: shows subscription usage message (or overage notice).
- For API users: calls `formatTotalCost()` to show token cost.
- ANT users always see cost breakdown even if subscriber.

**门禁:** Hidden for claude.ai subscribers (except ANT users).
**`supportsNonInteractive: true`**

---

### `createMovedToPluginCommand` (utility)

**文件:** `commands/createMovedToPluginCommand.ts`

Not a slash command itself — a factory function used to create commands that have been migrated to the plugin marketplace.

```typescript
export function createMovedToPluginCommand(options: {
  name: string
  description: string
  progressMessage: string
  pluginName: string
  pluginCommand: string
  getPromptWhileMarketplaceIsPrivate(args, context): Promise<ContentBlockParam[]>
}): Command
```

**行为:** For `USER_TYPE === 'ant'`, returns a prompt instructing the model to tell the user to install the plugin. For external users, runs the fallback `getPromptWhileMarketplaceIsPrivate` function. Used by `pr-comments`, `security-review`.

---

### `/ctx_viz`（占位）

**文件:** `commands/ctx_viz/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/debug-tool-call`（占位）

**文件:** `commands/debug-tool-call/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/desktop` / `/app`

**文件:** `commands/desktop/index.ts`, `commands/desktop/desktop.tsx`

**类型:** `local-jsx`
**语法:** `/desktop`
**别名:** `app`
**说明:** Continue the current session in Claude Desktop via handoff.

**门禁:**
- `availability: ['claude-ai']`
- `isEnabled`: only macOS or Windows x64

**行为:** Renders `DesktopHandoff` component which generates a handoff token/URL to open in the Claude Desktop app.

---

### `/diff`

**文件:** `commands/diff/index.ts`, `commands/diff/diff.tsx`

**类型:** `local-jsx`
**语法:** `/diff`
**说明:** View uncommitted git changes and per-turn diffs. Renders an interactive diff viewer.

---

### `/doctor`

**文件:** `commands/doctor/index.ts`, `commands/doctor/doctor.tsx`

**类型:** `local-jsx`
**语法:** `/doctor`
**说明:** Run diagnostics on the Claude Code installation: API key validity, model access, MCP connectivity, LSP status, plugin health, etc.

**门禁:** `isEnabled: () => !isEnvTruthy(process.env.DISABLE_DOCTOR_COMMAND)`

---

### `/effort`

**文件:** `commands/effort/index.ts`, `commands/effort/effort.tsx`

**类型:** `local-jsx`
**语法:** `/effort [low|medium|high|max|auto|help]`
**说明:** Set the effort level that controls how much "thinking" tokens the model uses.

**Arguments:**

| Value | Description |
|---|---|
| `low` | Minimal thinking tokens |
| `medium` | Moderate thinking tokens |
| `high` | Extended thinking |
| `max` | Maximum thinking tokens |
| `auto` | Model decides |
| `help` / `-h` / `--help` | Show help |
| *(none)* | Show current level |

**行为:**
- Calls `toPersistableEffort()` — some levels are session-only.
- Persists persistable values to `userSettings.effortLevel`.
- Logs `tengu_effort_command` analytics event.
- If `CLAUDE_CODE_EFFORT_LEVEL` env var is set and conflicts with the requested value, warns the user.

**`immediate`：** Dynamically set by `shouldInferenceConfigCommandBeImmediate()`.

---

### `/env`（占位）

**文件:** `commands/env/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/exit` / `/quit`

**文件:** `commands/exit/index.ts`, `commands/exit/exit.tsx`

**类型:** `local-jsx`
**语法:** `/exit`
**别名:** `quit`
**说明:** Exit the REPL gracefully.

**行为:**
- If `BG_SESSIONS` flag is on and this is a `--bg` tmux session: detaches the tmux client instead of killing it.
- If in a worktree session: shows `ExitFlow` dialog (asks about worktree cleanup).
- Otherwise: calls `gracefulShutdown(0, 'prompt_input_exit')` with a random goodbye message from `['Goodbye!', 'See ya!', 'Bye!', 'Catch you later!']`.

**属性:** `immediate: true`

---

### `/export`

**文件:** `commands/export/index.ts`, `commands/export/export.tsx`

**类型:** `local-jsx`
**语法:** `/export [filename]`
**说明:** Export the current conversation to a file or clipboard.

The implementation (~31KB) handles multiple formats, clipboard copy, and file output.

---

### `/extra-usage`

**文件:** `commands/extra-usage/index.ts`, `commands/extra-usage/extra-usage-core.ts`, `commands/extra-usage/extra-usage-noninteractive.ts`, `commands/extra-usage/extra-usage.tsx`

**Two variants:**
- `extraUsage`: `local-jsx`, for interactive sessions
- `extraUsageNonInteractive`: `local`, for `--print` / headless mode

**语法:** `/extra-usage`
**说明:** Configure extra usage (overage) to keep working when subscription limits are hit.

**门禁:** `isEnabled: () => isOverageProvisioningAllowed() && !isEnvTruthy(DISABLE_EXTRA_USAGE_COMMAND)`

**Core logic (`extra-usage-core.ts`):**
1. Marks `hasVisitedExtraUsage` in global config.
2. Invalidates overage credit grant cache.
3. For Team/Enterprise without billing access:
   - If unlimited overage already enabled → "already have unlimited."
   - If admin request pending/dismissed → "already submitted."
   - Creates an `admin_request` of type `limit_increase`.
4. For users with billing access or individual plans: opens browser to `claude.ai/settings/usage` or `claude.ai/admin-settings/usage`.

---

### `/fast`

**文件:** `commands/fast/index.ts`, `commands/fast/fast.tsx`

**类型:** `local-jsx`
**语法:** `/fast [on|off]`
**说明:** Toggle "fast mode" — switches to a faster, cheaper model (displayed as `FAST_MODE_MODEL_DISPLAY`).

**门禁:**
- `availability: ['claude-ai', 'console']`
- `isEnabled: () => isFastModeEnabled()`
- `isHidden: !isFastModeEnabled()`

**行为:**
- Clears fast mode cooldown.
- Writes `fastMode: true/undefined` to `userSettings`.
- If enabling and current model doesn't support fast mode, also switches `mainLoopModel`.
- Prefetches fast mode status on render.
- Shows a picker dialog (`FastModePicker`) with pricing info and cooldown status.

**`immediate`：** Set by `shouldInferenceConfigCommandBeImmediate()`.

---

### `/feedback` / `/bug`

**文件:** `commands/feedback/index.ts`, `commands/feedback/feedback.tsx`

**类型:** `local-jsx`
**语法:** `/feedback [report]`
**别名:** `bug`
**说明:** Submit feedback or bug reports about Claude Code.

**门禁:** Disabled when:
- `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, `CLAUDE_CODE_USE_FOUNDRY` are set
- `DISABLE_FEEDBACK_COMMAND` or `DISABLE_BUG_COMMAND` set
- `isEssentialTrafficOnly()` is true
- `USER_TYPE === 'ant'`
- `!isPolicyAllowed('allow_product_feedback')`

---

### `/files`

**文件:** `commands/files/index.ts`, `commands/files/files.ts`

**类型:** `local`
**语法:** `/files`
**说明:** List all files currently in the context (read state cache).

**门禁:** `isEnabled: () => process.env.USER_TYPE === 'ant'`

**输出:** Lists relative paths of all files in `readFileState` cache, or "No files in context."

**`supportsNonInteractive: true`**

---

### `/good-claude`（占位）

**文件:** `commands/good-claude/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/heapdump`

**文件:** `commands/heapdump/index.ts`, `commands/heapdump/heapdump.ts`

**类型:** `local`
**语法:** `/heapdump`
**说明:** Dump the JavaScript heap to `~/Desktop`.

**`isHidden: true`** — not shown in help but accessible.
**`supportsNonInteractive: true`**

**输出:** Prints `heapPath` and `diagPath` on success, or an error message.

---

### `/help`

**文件:** `commands/help/index.ts`, `commands/help/help.tsx`

**类型:** `local-jsx`
**语法:** `/help`
**说明:** Show help screen with available commands and keyboard shortcuts.

---

### `/hooks`

**文件:** `commands/hooks/index.ts`, `commands/hooks/hooks.tsx`

**类型:** `local-jsx`
**语法:** `/hooks`
**说明:** View and manage hook configurations for tool events (PreToolUse, PostToolUse, Stop, etc.).

**属性:** `immediate: true`

---

### `/ide`

**文件:** `commands/ide/index.ts`, `commands/ide/ide.tsx`

**类型:** `local-jsx`
**语法:** `/ide [open]`
**说明:** Manage IDE integrations (VS Code, JetBrains, etc.) and show connection status.

---

### `/init`

**文件:** `commands/init.ts`

**类型:** `prompt`
**语法:** `/init`
**说明:** Initialize CLAUDE.md (and optionally skills/hooks) for the current repository.

**Two prompt variants (feature-gated):**

**Old prompt (`OLD_INIT_PROMPT`):** Analyzes the codebase and creates a minimal CLAUDE.md with commands, architecture overview.

**New prompt (`NEW_INIT_PROMPT`, enabled by `NEW_INIT` flag or `CLAUDE_CODE_NEW_INIT=1`):** Multi-phase interactive setup:
- Phase 1: Ask what to set up (project CLAUDE.md, personal CLAUDE.local.md, skills, hooks)
- Phase 2: Explore codebase (manifest files, README, CI config, existing CLAUDE.md, linter config, git worktrees)
- Phase 3: Fill in gaps via `AskUserQuestion`
- Phase 4: Write CLAUDE.md at project root
- Phase 5: Write CLAUDE.local.md (added to .gitignore)
- Phase 6: Suggest and create skills in `.claude/skills/`
- Phase 7: Suggest additional optimizations (GitHub CLI, linting, hooks, format-on-edit)
- Phase 8: Summary and next steps (plugin suggestions)

**Side effect:** Calls `maybeMarkProjectOnboardingComplete()` when invoked.

---

### `/init-verifiers`

**文件:** `commands/init-verifiers.ts`

**类型:** `prompt`
**语法:** `/init-verifiers`
**说明:** Create verifier skill(s) for automated verification of code changes. Creates skills in `.claude/skills/` that can be used by the Verify agent.

**Multi-phase workflow:**
- Phase 1: Auto-detect project type and stack (web app → Playwright, CLI → Tmux, API → HTTP)
- Phase 2: Verification tool setup (Playwright installation, MCP configuration)
- Phase 3: Interactive Q&A (verifier names, dev server details, auth setup)
- Phase 4: Generate verifier skills with appropriate `allowed-tools`
- Phase 5: Confirm creation

**Verifier types and allowed tools:**

| Type | Tools |
|---|---|
| `verifier-playwright` | `Bash(npm:*)`, `Bash(yarn:*)`, `mcp__playwright__*`, Read, Glob, Grep |
| `verifier-cli` | Tmux, `Bash(asciinema:*)`, Read, Glob, Grep |
| `verifier-api` | `Bash(curl:*)`, `Bash(http:*)`, `Bash(npm:*)`, Read, Glob, Grep |

---

### `/insights`

**文件:** `commands/insights.ts` (lazy-loaded from `commands.ts`)

**类型:** `prompt`
**语法:** `/insights`
**说明:** Generate an AI-powered report analyzing Claude Code usage sessions.

**Implementation note:** This is a 113KB module (~3200 lines) that includes HTML rendering and diff utilities. It is registered in `commands.ts` as a lazy shim that dynamically imports the real module only when invoked, to avoid startup overhead.

**Key features (from the module):**
- Reads session files from `~/.claude/projects/`
- Analyzes conversations, extracts facets and patterns
- Runs summarization via Opus model
- Generates HTML and diff-line reports
- Supports remote Coder host enumeration for internal users

---

### `/install-github-app`

**文件:** `commands/install-github-app/index.ts`, plus step components

**类型:** `local-jsx`
**语法:** `/install-github-app`
**说明:** Interactive wizard to set up Claude GitHub Actions for a repository.

**门禁:**
- `availability: ['claude-ai', 'console']`
- `isEnabled: () => !isEnvTruthy(process.env.DISABLE_INSTALL_GITHUB_APP_COMMAND)`

**Wizard steps** (each is a separate React component):

| Component | Purpose |
|---|---|
| `OAuthFlowStep` | GitHub OAuth authentication |
| `CheckGitHubStep` | Verify GitHub CLI and remote setup |
| `ChooseRepoStep` | Select the repository |
| `CheckExistingSecretStep` | Check for existing ANTHROPIC_API_KEY secret |
| `ApiKeyStep` | Collect/create API key |
| `InstallAppStep` | Install the Claude GitHub App |
| `ExistingWorkflowStep` | Handle existing workflow files |
| `WarningsStep` | Display security warnings |
| `CreatingStep` | Create workflow file and secret |
| `SuccessStep` | Show success and next steps |
| `ErrorStep` | Display errors |

**Setup utility (`setupGitHubActions.ts`):** Creates `.github/workflows/claude.yml` and sets `ANTHROPIC_API_KEY` repository secret via GitHub API.

---

### `/install-slack-app`

**文件:** `commands/install-slack-app/index.ts`, `commands/install-slack-app/install-slack-app.ts`

**类型:** `local`
**语法:** `/install-slack-app`
**说明:** Open the Claude Slack app installation page in the browser.

**门禁:** `availability: ['claude-ai']`

**行为:**
- Logs `tengu_install_slack_app_clicked` event.
- Increments `slackAppInstallCount` in global config.
- Opens `https://slack.com/marketplace/A08SF47R6P4-claude` in browser.
- Returns URL if browser open fails.

---

### `install` (component)

**文件:** `commands/install.tsx`

This is a large (~27KB) React component used during the onboarding flow, not a standalone slash command. It handles installation steps, auth setup, and account configuration for first-time users.

---

### `/issue`（占位）

**文件:** `commands/issue/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/keybindings`

**文件:** `commands/keybindings/index.ts`, `commands/keybindings/keybindings.ts`

**类型:** `local`
**语法:** `/keybindings`
**说明:** Open or create the keybindings configuration file in the user's editor.

**门禁:** `isEnabled: () => isKeybindingCustomizationEnabled()` (feature in preview)

**行为:**
1. Gets keybindings file path from `getKeybindingsPath()`.
2. Creates parent directory if needed.
3. Writes a template file (using `'wx'` exclusive flag — no-op if exists).
4. Opens the file in the user's editor via `editFileInEditor()`.

---

### `/login`

**文件:** `commands/login/index.ts`, `commands/login/login.tsx`

**类型:** `local-jsx`
**语法:** `/login`
**说明:** Sign in with an account. Description dynamically shows "Switch accounts" if already authenticated.

**门禁:** `isEnabled: () => !isEnvTruthy(process.env.DISABLE_LOGIN_COMMAND)`

**Post-login actions:**
- `context.onChangeAPIKey()` — notifies the app of key change
- `stripSignatureBlocks` on messages (signatures are key-bound)
- `resetCostState()`
- `refreshRemoteManagedSettings()` (non-blocking)
- `refreshPolicyLimits()` (non-blocking)
- `resetUserCache()`
- `refreshGrowthBookAfterAuthChange()`
- `clearTrustedDeviceToken()` + `enrollTrustedDevice()` (non-blocking)
- Killswitch gate resets: `resetBypassPermissionsCheck()`, `resetAutoModeGateCheck()`
- `checkAndDisableBypassPermissionsIfNeeded()`, `checkAndDisableAutoModeIfNeeded()`

---

### `/logout`

**文件:** `commands/logout/index.ts`, `commands/logout/logout.tsx`

**类型:** `local-jsx`
**语法:** `/logout`
**说明:** Sign out from account.

**门禁:** `isEnabled: () => !isEnvTruthy(process.env.DISABLE_LOGOUT_COMMAND)`

**`performLogout()` steps:**
1. Flushes telemetry (before clearing credentials to prevent org data exposure).
2. `removeApiKey()` — deletes API key from storage.
3. `secureStorage.delete()` — wipes all secure storage.
4. Clears auth-related caches: Grove settings, OAuth tokens, policy limits, remote managed settings, betas caches, tool schema cache.
5. `resetUserCache()`, `refreshGrowthBookAfterAuthChange()`.
6. Updates global config (can optionally reset onboarding state).
7. `clearTrustedDeviceTokenCache()`.
8. `gracefulShutdownSync()` to exit.

---

### `/mcp`

**文件:** `commands/mcp/index.ts`, `commands/mcp/mcp.tsx`, `commands/mcp/addCommand.ts`, `commands/mcp/xaaIdpCommand.ts`

**类型:** `local-jsx`
**语法:** `/mcp [enable|disable [server-name]]`
**说明:** Manage Model Context Protocol (MCP) servers.

**属性:** `immediate: true`

**CLI subcommands (`mcp add` via `addCommand.ts`):**

```
claude mcp add <name> <commandOrUrl> [args...]
  -s, --scope <scope>        Configuration scope: local, user, project (default: local)
  -t, --transport <type>     Transport: stdio, sse, http (default: stdio)
  -e, --env <env...>         Environment variables (KEY=value)
  -H, --header <header...>   HTTP headers
  --client-id <id>           OAuth client ID
  --client-secret            Prompt for OAuth client secret
  --callback-port <port>     Fixed OAuth callback port
  --xaa                      Enable XAA (SEP-990) authentication
```

**XAA IdP subcommands (`claude mcp xaa` via `xaaIdpCommand.ts`):**

| Subcommand | Description |
|---|---|
| `xaa setup` | Configure IdP connection: `--issuer <url>` `--client-id <id>` `[--client-secret]` `[--callback-port]` |
| `xaa login` | Authenticate with IdP (OIDC browser flow). `[--force]` `[--id-token <jwt>]` |
| `xaa show` | Show current IdP config and login status |
| `xaa clear` | Clear IdP config and cached id_token |

**Security notes for `mcp xaa setup`:**
- Validates issuer URL: must be `https://` (or `http://localhost` for conformance).
- Validates `callbackPort` is a positive integer.
- Clears stale keychain slots when issuer or clientId changes.
- Writes settings before clearing keychain to avoid half-cleared state.

---

### `/memory`

**文件:** `commands/memory/index.ts`, `commands/memory/memory.tsx`

**类型:** `local-jsx`
**语法:** `/memory`
**说明:** Edit Claude memory files (CLAUDE.md, CLAUDE.local.md, etc.).

---

### `/mobile` / `/ios` / `/android`

**文件:** `commands/mobile/index.ts`, `commands/mobile/mobile.tsx`

**类型:** `local-jsx`
**语法:** `/mobile`
**别名:** `ios`, `android`
**说明:** Show a QR code to download the Claude mobile app.

---

### `/mock-limits`（占位）

**文件:** `commands/mock-limits/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/model`

**文件:** `commands/model/index.ts`, `commands/model/model.tsx`

**类型:** `local-jsx`
**语法:** `/model [model]`
**说明:** Set the AI model for Claude Code. Description dynamically shows the currently selected model.

**`immediate`：** Set by `shouldInferenceConfigCommandBeImmediate()`.

---

### `/oauth-refresh`（占位）

**文件:** `commands/oauth-refresh/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/onboarding`（占位）

**文件:** `commands/onboarding/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/output-style`

**文件:** `commands/output-style/index.ts`, `commands/output-style/output-style.tsx`

**类型:** `local-jsx`
**语法:** `/output-style`
**说明:** Deprecated — use `/config` to change output style.

**`isHidden: true`** — not shown in the command palette.

---

### `/passes`

**文件:** `commands/passes/index.ts`, `commands/passes/passes.tsx`

**类型:** `local-jsx`
**语法:** `/passes`
**说明:** Share a free week of Claude Code with friends (referral passes). Optionally shows "earn extra usage" if referrer rewards are available.

**门禁:** Hidden unless `checkCachedPassesEligibility().eligible && hasCache`.

---

### `/perf-issue`（占位）

**文件:** `commands/perf-issue/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/permissions` / `/allowed-tools`

**文件:** `commands/permissions/index.ts`, `commands/permissions/permissions.tsx`

**类型:** `local-jsx`
**语法:** `/permissions`
**别名:** `allowed-tools`
**说明:** Manage allow and deny rules for tool permissions.

---

### `/plan`

**文件:** `commands/plan/index.ts`, `commands/plan/plan.tsx`

**类型:** `local-jsx`
**语法:** `/plan [open|<description>]`
**说明:** Enable plan mode or view/edit the current session plan.

**Arguments:**

| Argument | Description |
|---|---|
| *(none)* | View current plan or toggle plan mode |
| `open` | Open the plan file in an external editor |
| `<description>` | Set/update the plan description |

**行为:**
- Reads the plan file via `getPlan()` / `getPlanFilePath()`.
- Calls `prepareContextForPlanMode()` and `applyPermissionUpdate()` to restrict tools when entering plan mode.
- Calls `handlePlanModeTransition()` in bootstrap state.
- If `open`, launches `editFileInEditor()` with the plan file path.
- Shows `PlanDisplay` component with plan content, path, and editor hint.

---

### `/plugin` / `/plugins` / `/marketplace`

**文件:** `commands/plugin/index.tsx`, `commands/plugin/plugin.tsx`, plus multiple sub-components and utilities

**类型:** `local-jsx`
**语法:** `/plugin [subcommand] [args]`
**别名:** `plugins`, `marketplace`
**说明:** Manage Claude Code plugins (install, uninstall, enable, disable, browse marketplace).

**属性:** `immediate: true`

**Parsed subcommands (`parseArgs.ts`):**

| Subcommand | Syntax | Description |
|---|---|---|
| *(none)* | `/plugin` | Open interactive menu |
| `install` / `i` | `/plugin install [plugin@marketplace]` | Install a plugin |
| `manage` | `/plugin manage` | Open manage plugins UI |
| `uninstall` | `/plugin uninstall [plugin]` | Uninstall a plugin |
| `enable` | `/plugin enable [plugin]` | Enable a plugin |
| `disable` | `/plugin disable [plugin]` | Disable a plugin |
| `validate` | `/plugin validate [path]` | Validate a plugin definition |
| `marketplace` | `/plugin marketplace [add\|remove\|update\|list] [target]` | Manage marketplaces |
| `help` | `/plugin help` | Show help |

**Plugin format:** `plugin-name@marketplace-name` or `plugin-name@https://marketplace-url`

**UI components:**
- `ManagePlugins.tsx` — list/toggle/remove installed plugins
- `BrowseMarketplace.tsx` — browse marketplace listings
- `AddMarketplace.tsx` — add a new marketplace URL
- `ManageMarketplaces.tsx` — manage registered marketplaces
- `DiscoverPlugins.tsx` — discovery flow for new users
- `PluginErrors.tsx` — display plugin load errors
- `PluginOptionsDialog.tsx` — configure plugin options
- `PluginOptionsFlow.tsx` — step-by-step option configuration
- `PluginSettings.tsx` — plugin settings panel
- `PluginTrustWarning.tsx` — security warning before install
- `UnifiedInstalledCell.tsx` — single plugin row in list
- `ValidatePlugin.tsx` — validation result display
- `pluginDetailsHelpers.tsx` — helpers for displaying plugin info
- `usePagination.ts` — pagination hook for marketplace listings

---

### `/pr-comments`

**文件:** `commands/pr_comments/index.ts`

**类型:** `prompt` (via `createMovedToPluginCommand`)
**语法:** `/pr-comments [PR number or args]`
**说明:** Get comments from a GitHub pull request. Uses `gh` CLI to fetch PR-level and code review comments.

**Plugin migration:** For `USER_TYPE === 'ant'`, directs to `pr-comments@claude-code-marketplace`. For external users, runs the embedded prompt.

**Fallback prompt logic:**
1. `gh pr view --json number,headRepository` to get PR info.
2. `gh api /repos/{owner}/{repo}/issues/{number}/comments` for PR-level comments.
3. `gh api /repos/{owner}/{repo}/pulls/{number}/comments` for review comments.
4. Formats with author, file#line, diff_hunk, comment text.

---

### `/privacy-settings`

**文件:** `commands/privacy-settings/index.ts`, `commands/privacy-settings/privacy-settings.tsx`

**类型:** `local-jsx`
**语法:** `/privacy-settings`
**说明:** View and update privacy settings (training data opt-out, etc.).

**门禁:** `isEnabled: () => isConsumerSubscriber()`

---

### `/rate-limit-options`

**文件:** `commands/rate-limit-options/index.ts`, `commands/rate-limit-options/rate-limit-options.tsx`

**类型:** `local-jsx`
**语法:** `/rate-limit-options`
**说明:** Show options when the rate limit is reached (upgrade, extra usage, etc.). Shown only to `claude-ai` subscribers.

**`isHidden: true`** — only used internally (e.g., triggered from the rate-limit message component, not user-invoked).

---

### `/release-notes`

**文件:** `commands/release-notes/index.ts`, `commands/release-notes/release-notes.ts`

**类型:** `local`
**语法:** `/release-notes`
**说明:** Display the changelog for Claude Code.

**`supportsNonInteractive: true`**

**行为:**
1. Tries to fetch fresh changelog via `fetchAndStoreChangelog()` with a 500ms timeout.
2. Falls back to cached notes via `getStoredChangelog()`.
3. If nothing available, shows link to `CHANGELOG_URL`.
4. Formats as: `Version X.Y.Z:\n· bullet1\n· bullet2`

---

### `/reload-plugins`

**文件:** `commands/reload-plugins/index.ts`, `commands/reload-plugins/reload-plugins.ts`

**类型:** `local`
**语法:** `/reload-plugins`
**说明:** Activate pending plugin changes in the current session (Layer-3 refresh).

**门禁:** `supportsNonInteractive: false`

**行为:**
1. In remote/CCR mode with `DOWNLOAD_USER_SETTINGS` flag: re-downloads user settings from server and fires `settingsChangeDetector.notifyChange`.
2. Calls `refreshActivePlugins(context.setAppState)`.
3. Returns summary: "Reloaded: N plugins · N skills · N agents · N hooks · N plugin MCP servers · N plugin LSP servers".
4. If errors: "N errors during load. Run /doctor for details."

---

### `/remote-env`

**文件:** `commands/remote-env/index.ts`, `commands/remote-env/remote-env.tsx`

**类型:** `local-jsx`
**语法:** `/remote-env`
**说明:** Configure the default remote environment for teleport sessions.

**门禁:**
- `isEnabled: () => isClaudeAISubscriber() && isPolicyAllowed('allow_remote_sessions')`
- Hidden when not eligible.

---

### `/remote-setup` / `/web-setup`

**文件:** `commands/remote-setup/index.ts`, `commands/remote-setup/remote-setup.tsx`, `commands/remote-setup/api.ts`

**类型:** `local-jsx`
**Name:** `web-setup`
**说明:** Set up Claude Code on the web (connects GitHub account).

**门禁:**
- `availability: ['claude-ai']`
- `isEnabled: () => getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_lantern', false) && isPolicyAllowed('allow_remote_sessions')`
- Feature flag: `CCR_REMOTE_SETUP`

---

### `/rename`

**文件:** `commands/rename/index.ts`, `commands/rename/rename.ts`, `commands/rename/generateSessionName.ts`

**类型:** `local-jsx`
**语法:** `/rename [name]`
**说明:** Rename the current conversation.

**属性:** `immediate: true`

**Arguments:**

| Argument | Description |
|---|---|
| `[name]` | New name for the session. If omitted, auto-generates one using Haiku. |

**行为:**
1. Blocked for swarm teammates (names set by team leader).
2. If no name provided: calls `generateSessionName(messages, abortSignal)`:
   - Sends conversation text to `queryHaiku` with a system prompt to generate a 2-4 word kebab-case name.
   - Returns JSON `{ name: string }`.
3. Saves name via `saveCustomTitle()` and `saveAgentName()`.
4. Syncs to bridge session title via `updateBridgeSessionTitle()` if connected.
5. Updates `AppState.standaloneAgentContext.name`.

---

### `/reset-limits`（占位）

**文件:** `commands/reset-limits/index.js`

**类型:** Stub — exports both `resetLimits` and `resetLimitsNonInteractive` as disabled stubs.

---

### `/resume` / `/continue`

**文件:** `commands/resume/index.ts`, `commands/resume/resume.tsx`

**类型:** `local-jsx`
**语法:** `/resume [conversation id or search term]`
**别名:** `continue`
**说明:** Resume a previous conversation. Opens a fuzzy-searchable list of recent sessions.

---

### `/review`

**文件:** `commands/review.ts`

**类型:** `prompt`
**语法:** `/review [PR number]`
**说明:** Review a GitHub pull request using `gh` CLI.

**Prompt logic:**
1. If no PR number: runs `gh pr list` to show open PRs.
2. If PR number: runs `gh pr view <number>` and `gh pr diff <number>`.
3. Returns a structured code review with Overview, Code quality, Suggestions, Issues/risks sections.

---

### `/ultrareview`

**文件:** `commands/review.ts`, `commands/review/ultrareviewCommand.tsx`, `commands/review/ultrareviewEnabled.ts`, `commands/review/reviewRemote.ts`, `commands/review/UltrareviewOverageDialog.tsx`

**类型:** `local-jsx`
**语法:** `/ultrareview`
**说明:** Deep automated bug-finding review running in Claude Code on the web (~10–20 min). Finds and verifies bugs in the current branch.

**门禁:** `isEnabled: () => isUltrareviewEnabled()` — checks GrowthBook `tengu_review_bughunter_config.enabled`.

**行为:** Launches a remote agent session (CCR) that runs the bughunter analysis. Shows an overage dialog if free reviews are exhausted.

---

### `/rewind` / `/checkpoint`

**文件:** `commands/rewind/index.ts`, `commands/rewind/rewind.ts`

**类型:** `local`
**语法:** `/rewind`
**别名:** `checkpoint`
**说明:** Restore the code and/or conversation to a previous point. Opens the message selector UI.

**门禁:** `supportsNonInteractive: false`

**行为:** Calls `context.openMessageSelector()` and returns `{ type: 'skip' }` (no message appended).

---

### `/sandbox`

**文件:** `commands/sandbox-toggle/index.ts`, `commands/sandbox-toggle/sandbox-toggle.tsx`

**类型:** `local-jsx`
**语法:** `/sandbox [exclude "command pattern"]`
**说明:** Configure sandbox settings for bash command execution.

**Dynamic description** shows current sandbox status: enabled/disabled, auto-allow mode, fallback allowed, managed policy.

**`isHidden`:** Hidden on unsupported platforms (`!SandboxManager.isSupportedPlatform()` or `!SandboxManager.isPlatformInEnabledList()`).
**`immediate: true`**

---

### `/security-review`

**文件:** `commands/security-review.ts`

**类型:** `prompt` (via `createMovedToPluginCommand`)
**语法:** `/security-review`
**说明:** Complete a security-focused review of pending branch changes.

**Plugin migration:** Directs `USER_TYPE === 'ant'` users to `security-review@claude-code-marketplace`.

**Fallback prompt:** Large SECURITY_REVIEW_MARKDOWN with:
- Allowed tools: `Bash(git diff:*)`, `Bash(git status:*)`, `Bash(git log:*)`, `Bash(git show:*)`, `Bash(git remote show:*)`, `Read`, `Glob`, `Grep`, `LS`, `Task`
- Live shell injection: `git status`, `git diff --name-only origin/HEAD...`, `git log origin/HEAD...`, `git diff origin/HEAD...`

**Analysis methodology:** 3-phase (explore codebase, vulnerability scan, false-positive filtering with confidence scoring). Launches parallel sub-tasks for FP filtering.

**Security categories:** Input validation (SQLi, CMDi, XXE, template injection, NoSQL injection, path traversal), auth/authz, crypto/secrets, injection/RCE, data exposure.

**Hard exclusions:** DoS, secrets on disk, rate limiting, memory issues, test-only code, log spoofing, SSRF path-only, AI prompt injection, regex, insecure docs, missing audit logs.

---

### `/session` / `/remote`

**文件:** `commands/session/index.ts`, `commands/session/session.tsx`

**类型:** `local-jsx`
**语法:** `/session`
**别名:** `remote`
**说明:** Show remote session URL and QR code (for connecting mobile/web clients).

**门禁:** `isEnabled: () => getIsRemoteMode()` — only shown in `--remote` mode.

---

### `/share`（占位）

**文件:** `commands/share/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/skills`

**文件:** `commands/skills/index.ts`, `commands/skills/skills.tsx`

**类型:** `local-jsx`
**语法:** `/skills`
**说明:** List all available skills (custom commands loaded from skill directories, plugins, and bundled skills).

---

### `/stats`

**文件:** `commands/stats/index.ts`, `commands/stats/stats.tsx`

**类型:** `local-jsx`
**语法:** `/stats`
**说明:** Show Claude Code usage statistics and activity (sessions, token usage, costs over time).

---

### `/status`

**文件:** `commands/status/index.ts`, `commands/status/status.tsx`

**类型:** `local-jsx`
**语法:** `/status`
**说明:** Show comprehensive Claude Code status: version, model, account, API connectivity, tool statuses.

**属性:** `immediate: true`

---

### `/statusline`

**文件:** `commands/statusline.tsx`

**类型:** `prompt`
**语法:** `/statusline [description]`
**说明:** Set up Claude Code's status line UI. Launches a specialized `statusline-setup` subagent.

**Allowed tools:** `AgentTool`, `Read(~/**)`, `Edit(~/.claude/settings.json)`

**`disableNonInteractive: true`**

**行为:** Creates an AgentTool call with `subagent_type: "statusline-setup"` and the user's prompt (default: "Configure my statusLine from my shell PS1 configuration").

---

### `/stickers`

**文件:** `commands/stickers/index.ts`, `commands/stickers/stickers.ts`

**类型:** `local`
**语法:** `/stickers`
**说明:** Open the Claude Code sticker ordering page at `https://www.stickermule.com/claudecode`.

**门禁:** `supportsNonInteractive: false`

---

### `/summary`（占位）

**文件:** `commands/summary/index.js`

**类型:** Stub — disabled in all external builds.
**Note:** This is in `BRIDGE_SAFE_COMMANDS` and `INTERNAL_ONLY_COMMANDS` in commands.ts; the non-stub implementation is internal-only.

---

### `/tag`

**文件:** `commands/tag/index.ts`, `commands/tag/tag.tsx`

**类型:** `local-jsx`
**语法:** `/tag <tag-name>`
**说明:** Toggle a searchable tag on the current session (for session search/filtering).

**门禁:** `isEnabled: () => process.env.USER_TYPE === 'ant'`

---

### `/tasks` / `/bashes`

**文件:** `commands/tasks/index.ts`, `commands/tasks/tasks.tsx`

**类型:** `local-jsx`
**语法:** `/tasks`
**别名:** `bashes`
**说明:** List and manage background tasks (running shell commands, agent tasks, etc.).

---

### `/teleport`（占位）

**文件:** `commands/teleport/index.js`

**类型:** Stub — disabled in all external builds.

---

### `/terminal-setup`

**文件:** `commands/terminalSetup/index.ts`, `commands/terminalSetup/terminalSetup.tsx`

**类型:** `local-jsx`
**语法:** `/terminal-setup`
**说明:** Install terminal keyboard bindings for newline input.

**Dynamic description:**
- Apple Terminal: "Enable Option+Enter key binding for newlines and visual bell"
- Other terminals: "Install Shift+Enter key binding for newlines"

**`isHidden`:** True for terminals with native CSI-u support (Ghostty, Kitty, iTerm2, WezTerm).

---

### `/theme`

**文件:** `commands/theme/index.ts`, `commands/theme/theme.tsx`

**类型:** `local-jsx`
**语法:** `/theme`
**说明:** Change the TUI color theme (dark/light/etc.).

---

### `/think-back`

**文件:** `commands/thinkback/index.ts`, `commands/thinkback/thinkback.tsx`

**类型:** `local-jsx`
**语法:** `/think-back`
**说明:** "Your 2025 Claude Code Year in Review" — a year-in-review animation.

**门禁:** `isEnabled: () => checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_thinkback')`

---

### `/thinkback-play`

**文件:** `commands/thinkback-play/index.ts`, `commands/thinkback-play/thinkback-play.ts`

**类型:** `local`
**语法:** `/thinkback-play`
**说明:** Play the thinkback animation. Hidden command called by the thinkback skill after generation is complete.

**门禁:** `checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_thinkback')`
**`isHidden: true`**, `supportsNonInteractive: false`

---

### `/ultraplan`

**文件:** `commands/ultraplan.tsx`

**类型:** `local-jsx`
**语法:** `/ultraplan [seed plan]`
**说明:** Run an extended multi-agent planning session on Claude Code on the web (CCR). Explores the codebase, creates a comprehensive implementation plan, and enters plan-approval mode.

**特性门禁:** `ULTRAPLAN` bundle flag required.

**Constants:**
```typescript
const ULTRAPLAN_TIMEOUT_MS = 30 * 60 * 1000  // 30 min
export const CCR_TERMS_URL = 'https://code.claude.com/docs/en/claude-code-on-the-web'
```

**Model:** From GrowthBook `tengu_ultraplan_model` (defaults to `ALL_MODEL_CONFIGS.opus46.firstParty`).

**行为:**
1. Checks `checkRemoteAgentEligibility()` — returns error if not eligible.
2. Opens a remote CCR session via `teleportToRemote()`.
3. Sends the ultraplan prompt (from `ultraplan/prompt.txt`, inlined at bundle time; ANT builds can override via `ULTRAPLAN_PROMPT_FILE` env).
4. Polls `pollForApprovedExitPlanMode()` every ~5s for up to 30 minutes.
5. On approval: archives the remote session, updates local app state, transitions to plan mode.
6. On `REMOTE_CONTROL_DISCONNECTED_MSG`: falls back gracefully.
7. Shows CCR terms URL in description for legal visibility.

---

### `/upgrade`

**文件:** `commands/upgrade/index.ts`, `commands/upgrade/upgrade.tsx`

**类型:** `local-jsx`
**语法:** `/upgrade`
**说明:** Upgrade to Max plan for higher rate limits and more Opus access.

**门禁:**
- `availability: ['claude-ai']`
- `isEnabled: () => !isEnvTruthy(DISABLE_UPGRADE_COMMAND) && getSubscriptionType() !== 'enterprise'`

---

### `/usage`

**文件:** `commands/usage/index.ts`, `commands/usage/usage.tsx`

**类型:** `local-jsx`
**语法:** `/usage`
**说明:** Show plan usage limits and current consumption.

**门禁:** `availability: ['claude-ai']`

---

### `/version`

**文件:** `commands/version.ts`

**类型:** `local`
**语法:** `/version`
**说明:** Print the version this session is running (not what auto-update downloaded).

**门禁:** `isEnabled: () => process.env.USER_TYPE === 'ant'`

**输出:** `"${VERSION} (built ${BUILD_TIME})"` or just `VERSION` if no build time.

**`supportsNonInteractive: true`**

**Note:** Uses `MACRO.VERSION` and `MACRO.BUILD_TIME` — build-time macros injected by the bundler.

---

### `/vim`

**文件:** `commands/vim/index.ts`, `commands/vim/vim.ts`

**类型:** `local`
**语法:** `/vim`
**说明:** Toggle between Vim and Normal (readline) editing modes for the prompt input.

**门禁:** `supportsNonInteractive: false`

**行为:**
- Reads `config.editorMode` (handles legacy `'emacs'` → `'normal'`).
- Toggles between `'normal'` and `'vim'`.
- Saves to global config.
- Logs `tengu_editor_mode_changed` event with `source: 'command'`.
- Returns description: "Editor mode set to vim/normal. Use Escape key..." / "Using standard (readline) keyboard bindings."

---

### `/voice`

**文件:** `commands/voice/index.ts`, `commands/voice/voice.ts`

**类型:** `local`
**语法:** `/voice`
**说明:** Toggle voice mode (speech-to-text input via push-to-talk).

**门禁:**
- `availability: ['claude-ai']`
- `isEnabled: () => isVoiceGrowthBookEnabled()` — feature-flagged via GrowthBook
- `isHidden: !isVoiceModeEnabled()`
- Feature flag: `VOICE_MODE` bundle flag

**Toggle ON pre-flight checks:**
1. `isVoiceModeEnabled()` — checks kill-switch + auth.
2. `checkRecordingAvailability()` — microphone hardware check.
3. `isVoiceStreamAvailable()` — checks for API key.
4. `checkVoiceDependencies()` — checks for SoX or other audio recording tool.
5. `requestMicrophonePermission()` — OS permission probe (fires permission dialog now).

**Toggle OFF:** No checks needed, always allowed.

**On success:** `updateSettingsForSource('userSettings', { voiceEnabled: true/false })`, fires `settingsChangeDetector.notifyChange`, logs `tengu_voice_toggled`.

**Language hint:** Shows STT language on first 2 enables; falls back to English if language not supported.

---

## 5. 仅内部命令汇总

These commands are registered only when `process.env.USER_TYPE === 'ant'` and `!process.env.IS_DEMO`:

| Command | Description |
|---|---|
| `backfill-sessions` | Stub (disabled) |
| `break-cache` | Stub (disabled) |
| `bughunter` | Stub (disabled) |
| `commit` | Git commit via model |
| `commit-push-pr` | Commit + push + PR via model |
| `ctx_viz` | Stub (disabled) |
| `good-claude` | Stub (disabled) |
| `issue` | Stub (disabled) |
| `init-verifiers` | Create verifier skills |
| `force-snip` | Force history snip (feature-gated) |
| `mock-limits` | Stub (disabled) |
| `bridge-kick` | Inject bridge failure states |
| `version` | Print version |
| `ultraplan` | Remote planning session (feature-gated) |
| `subscribe-pr` | GitHub webhook subscription (feature-gated) |
| `reset-limits` | Stub (disabled) |
| `reset-limits-noninteractive` | Stub (disabled) |
| `onboarding` | Stub (disabled) |
| `share` | Stub (disabled) |
| `summary` | Stub (disabled) |
| `teleport` | Stub (disabled) |
| `ant-trace` | Stub (disabled) |
| `perf-issue` | Stub (disabled) |
| `env` | Stub (disabled) |
| `oauth-refresh` | Stub (disabled) |
| `debug-tool-call` | Stub (disabled) |
| `agents-platform` | Internal agents platform command |
| `autofix-pr` | Stub (disabled) |

---

## 6. 远程安全命令

Commands available in `--remote` mode (before CCR init arrives). These only affect local TUI state:

```
session, exit, clear, help, theme, color, vim, cost, usage,
copy, btw, feedback, plan, keybindings, statusline, stickers, mobile
```

---

## 7. 桥接安全命令

`'local'`-type commands safe to execute over the Remote Control bridge (from mobile/web). `'prompt'` commands are always bridge-safe. `'local-jsx'` commands are always blocked.

```
compact, clear, cost, summary, release-notes, files
```

---

## 8. 命令可用性与特性标志

### 可用性要求

| Value | Meaning |
|---|---|
| `'claude-ai'` | Must be a claude.ai subscriber (`isClaudeAISubscriber()`) |
| `'console'` | Must be a first-party API user (not 3P, not claude.ai, not custom baseURL) |

Commands without `availability` are shown to all users.

### 影响命令的环境变量

| Variable | Effect |
|---|---|
| `DISABLE_COMPACT` | Disables `/compact` |
| `DISABLE_DOCTOR_COMMAND` | Disables `/doctor` |
| `DISABLE_INSTALL_GITHUB_APP_COMMAND` | Disables `/install-github-app` |
| `DISABLE_LOGIN_COMMAND` | Disables `/login` |
| `DISABLE_LOGOUT_COMMAND` | Disables `/logout` |
| `DISABLE_FEEDBACK_COMMAND` | Disables `/feedback` |
| `DISABLE_BUG_COMMAND` | Disables `/feedback` alias |
| `DISABLE_UPGRADE_COMMAND` | Disables `/upgrade` |
| `DISABLE_EXTRA_USAGE_COMMAND` | Disables `/extra-usage` |
| `CLAUDE_CODE_USE_BEDROCK` | Disables `/feedback` |
| `CLAUDE_CODE_USE_VERTEX` | Disables `/feedback` |
| `CLAUDE_CODE_USE_FOUNDRY` | Disables `/feedback` |
| `USER_TYPE=ant` | Enables internal-only commands |
| `IS_DEMO=1` | Suppresses internal-only commands even for ant users |
| `CLAUDE_CODE_NEW_INIT=1` | Enables new multi-phase `/init` prompt |
| `CLAUDE_CODE_ENABLE_XAA=1` | Enables `--xaa` flag in `claude mcp add` |
| `MCP_CLIENT_SECRET` | OAuth client secret for `claude mcp add --client-secret` |
| `MCP_XAA_IDP_CLIENT_SECRET` | IdP client secret for `claude mcp xaa setup --client-secret` |
| `ULTRAPLAN_PROMPT_FILE` | Override ultraplan prompt (ant builds only) |

### Bun 打包特性标志

Feature flags evaluated at bundle time via `feature('FLAG_NAME')`:

| Flag | Commands/Behavior |
|---|---|
| `BRIDGE_MODE` | Enables `/remote-control` |
| `BRIDGE_MODE` + `DAEMON` | Enables remoteControlServer |
| `KAIROS` / `KAIROS_BRIEF` | Enables `/brief` |
| `KAIROS` | Enables `assistant` command |
| `VOICE_MODE` | Enables `/voice` |
| `HISTORY_SNIP` | Enables `force-snip` |
| `WORKFLOW_SCRIPTS` | Enables `workflows` + `getWorkflowCommands` |
| `CCR_REMOTE_SETUP` | Enables `/web-setup` |
| `ULTRAPLAN` | Enables `/ultraplan` |
| `KAIROS_GITHUB_WEBHOOKS` | Enables `subscribe-pr` |
| `TORCH` | Enables `torch` |
| `UDS_INBOX` | Enables `peers` |
| `FORK_SUBAGENT` | Enables `fork` command; removes `fork` alias from `/branch` |
| `BUDDY` | Enables `buddy` |
| `PROACTIVE` / `KAIROS` | Enables `proactive` |
| `EXPERIMENTAL_SKILL_SEARCH` | Enables skill index cache clearing |
| `REACTIVE_COMPACT` | Routes `/compact` through reactive path |
| `CONTEXT_COLLAPSE` | Adds collapse stats to `/context` output |
| `NEW_INIT` | New multi-phase `/init` prompt |
| `BG_SESSIONS` | `/exit` detaches tmux instead of killing |
| `COORDINATOR_MODE` | `/clear` saves coordinator mode |
| `COMMIT_ATTRIBUTION` | `/clear` clears attribution caches |
| `TRANSCRIPT_CLASSIFIER` | `/login` runs auto-mode gate check |
| `DOWNLOAD_USER_SETTINGS` | `/reload-plugins` re-downloads settings in CCR mode |
| `MCP_SKILLS` | `getMcpSkillCommands` returns MCP prompt skills |
