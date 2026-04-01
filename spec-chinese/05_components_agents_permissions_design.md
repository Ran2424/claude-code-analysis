# Claude Code — 组件：Agents、权限、设计系统与功能模块

本文档是 `src/components/` 各子目录的完整说明，覆盖 agents 管理、权限 UI、设计系统基础组件，以及大量功能模块（MCP、memory、tasks、teams、diff、grove、hooks、HelpV2、TrustDialog、ManagedSettingsSecurityDialog、ClaudeCodeHint、HighlightedCode、LogoV2、DesktopUpsell、FeedbackSurvey、LspRecommendation、Passes、Spinner、PromptInput、CustomSelect、Settings、sandbox、shell、skills、ui、wizard）。

---

## 目录

1. [agents/](#1-agents)
2. [permissions/](#2-permissions)
3. [design-system/](#3-design-system)
4. [wizard/](#4-wizard)
5. [mcp/](#5-mcp)
6. [memory/](#6-memory)
7. [tasks/](#7-tasks)
8. [teams/](#8-teams)
9. [diff/](#9-diff)
10. [grove/](#10-grove)
11. [hooks/](#11-hooks-componentshooks)
12. [HelpV2/](#12-helpv2)
13. [TrustDialog/](#13-trustdialog)
14. [ManagedSettingsSecurityDialog/](#14-managedsettingssecuritydialog)
15. [ClaudeCodeHint/](#15-claudecodehint)
16. [HighlightedCode/](#16-highlightedcode)
17. [LogoV2/](#17-logov2)
18. [DesktopUpsell/](#18-desktopupsell)
19. [FeedbackSurvey/](#19-feedbacksurvey)
20. [LspRecommendation/](#20-lsprecommendation)
21. [Passes/](#21-passes)
22. [Spinner/](#22-spinner)
23. [PromptInput/](#23-promptinput)
24. [CustomSelect/](#24-customselect)
25. [Settings/](#25-settings)
26. [sandbox/](#26-sandbox)
27. [shell/](#27-shell)
28. [skills/](#28-skills)
29. [ui/](#29-ui)

---

## 1. agents/

agents 子系统提供完整 UI，用于创建、查看、编辑、删除和列出自定义 Claude sub-agent（存储为 `.claude/agents/` 下带 YAML front matter 的 Markdown 文件）。

### 1.1 AgentDetail

**文件：** `agents/AgentDetail.tsx`

**用途：** 只读展示单个 `AgentDefinition` 的详情，显示全部 agent 元数据（文件路径、描述、tools、model、permission mode、memory、hooks、skills、颜色块，以及非内置 agent 的 system prompt）。

**Props 接口：**
```typescript
type Props = {
  agent: AgentDefinition;
  tools: Tools;
  allAgents?: AgentDefinition[];
  onBack: () => void;
}
```

**关键行为：**
- 通过 `resolveAgentTools(agent, tools, false)` 解析 tools 列表，会显示 “All tools”、合法的命名工具，并对未识别的工具名给出警告符号。
- 通过 `getAgentColor(agent.agentType)` 计算 `backgroundColor`，用于颜色预览块。
- 在 Confirmation 上下文中绑定 `confirm:no`，触发 `onBack`。
- 绑定 `return` 键触发 `onBack`。
- 仅对非 built-in agents（`!isBuiltInAgent(agent)`）使用 `<Markdown>` 渲染 system prompt。
- 使用 `getAgentModelDisplay(agent.model)` 显示 model。
- 使用 `getMemoryScopeDisplay(agent.memory)` 显示 memory。
- 当 skills 条目超过 10 个时，内联显示数量。

**导出：** `AgentDetail`

---

### 1.2 AgentEditor

**文件：** `agents/AgentEditor.tsx`

**用途：** 用于修改现有 `AgentDefinition` 的全屏编辑器。支持编辑 agent type（名称）、system prompt、description（whenToUse）、tools、model、颜色、memory 和 effort。保存时调用 `updateAgentFile`。通过子状态机提供分步编辑流程。

**Props 接口：**
```typescript
type Props = {
  agent: AgentDefinition;
  tools: Tools;
  existingAgents: AgentDefinition[];
  onComplete: (message: string) => void;
  onCancel: () => void;
}
```

**关键行为：**
- 多步骤流程：主编辑菜单 → 单字段编辑器（每个字段都有自己的 step）。
- 编辑菜单包含：Edit system prompt、Edit description、Select tools、Select model、Choose color、（条件性）Choose memory、Back。
- 每个字段编辑器都会复用 wizard 步骤中的子组件（ToolSelector、ModelSelector、ColorPicker 等）。
- 保存时调用 `updateAgentFile(agent, ...)`，并刷新应用状态中的 agent。
- 允许改名之前先用 `validateAgentType()` 校验 agent type。
- 全程使用 React compiler memoization。

**导出：** `AgentEditor`

---

### 1.3 AgentNavigationFooter

**文件：** `agents/AgentNavigationFooter.tsx`

**用途：** 渲染变暗的页脚提示行，展示键盘导航说明。集成 Ctrl+C/D 退出状态，在双击检测期间显示 “Press X again to exit”。

**Props 接口：**
```typescript
type Props = {
  instructions?: string;
}
// 默认值： "Press ↑↓ to navigate · Enter to select · Esc to go back"
```

**关键行为：**
- 调用 `useExitOnCtrlCDWithKeybindings()` 检测待退出状态。
- 当 `exitState.pending` 为 true 时，将说明文本覆盖为 `Press ${exitState.keyName} again to exit`。
- 使用 `marginLeft={2}` 和 `dimColor` 渲染。

**导出：** `AgentNavigationFooter`

---

### 1.4 AgentsList

**文件：** `agents/AgentsList.tsx`

**用途：** 显示指定来源的 agent 列表（all、built-in、userSettings、projectSettings 等），按来源分组。支持键盘导航、可选的 “Create new agent” 入口，并在某个 agent 被其他作用域遮蔽时显示覆盖警告。

**Props 接口：**
```typescript
type Props = {
  source: SettingSource | 'all' | 'built-in' | 'plugin';
  agents: ResolvedAgent[];
  onBack: () => void;
  onSelect: (agent: AgentDefinition) => void;
  onCreateNew?: () => void;
  changes?: string[];
}
```

**关键行为：**
- 状态：`selectedAgent`（当前高亮的 agent）、`isCreateNewSelected`（是否聚焦 “Create new agent”）。
- 自动选中第一个 agent，或在未选择时选中 “Create new”。
- `handleKeyDown`：上下箭头在列表中移动（循环），Enter 激活当前项。
- 通过 `AGENT_SOURCE_GROUPS` 按来源分组。
- built-in agents 单独通过 `renderBuiltInAgentsSection()` 渲染（变暗、不可点）。
- 每个 agent 的展示包括：名称、可选 model 显示（中点分隔）、memory 标签、覆盖遮蔽警告。
- 被遮蔽的 agent 变暗显示，并带有警告符号（`figures.warning`）和 “shadowed by X” 文本。
- 顶部的 changes 列表使用 success 颜色展示。

**导出：** `AgentsList`

---

### 1.5 AgentsMenu

**文件：** `agents/AgentsMenu.tsx`

**用途：** agents 管理 UI 的顶层协调器。实现状态机，模式包括：`list-agents`、`create-agent`、`agent-menu`、`view-agent`、`edit-agent`、`delete-confirm`。负责读取/写入 agent 定义相关的应用状态。

**Props 接口：**
```typescript
type Props = {
  tools: Tools;
  onExit: (result?: string, options?: { display?: CommandResultDisplay }) => void;
}
```

**关键行为：**
- `modeState` 使用由 `mode` 字段区分的联合类型。
- 从应用状态读取 `agentDefinitions`、`mcpTools`、`toolPermissionContext`。
- 通过 `useMergedTools()` 合并 tools。
- 将 agents 按来源分为 8 个桶（built-in、userSettings、projectSettings、policySettings、localSettings、flagSettings、plugin、all）。
- 通过 `resolveAgentOverrides()` 处理显示时覆盖。
- `handleAgentCreated`：添加变更消息，回到 `list-agents/all` 视图。
- `handleAgentDeleted`：调用 `deleteAgentFromFile()`，更新应用状态，添加变更消息。
- 在 `agent-menu` 模式下：渲染 `<Select>`，选项有 View / Edit（若可编辑）/ Delete（若可编辑）/ Back。可编辑性要求 `source` 不在 `['built-in', 'plugin', 'flagSettings']` 中。
- 在 `delete-confirm` 模式下：显示确认 `<Dialog>`，提供 Yes/No。
- 退出时：格式化变更摘要，或输出 “Agents dialog dismissed” 系统消息。

**导出：** `AgentsMenu`

---

### 1.6 ColorPicker

**文件：** `agents/ColorPicker.tsx`

**用途：** agent type 颜色的交互式选择器。展示全部 `AGENT_COLORS` 以及一个 “Automatic color” 选项，并实时预览该颜色在 agent 名称上的效果。

**Props 接口：**
```typescript
type Props = {
  agentName: string;
  currentColor?: AgentColorName | 'automatic';
  onConfirm: (color: AgentColorName | undefined) => void;
}
```

**关键行为：**
- `COLOR_OPTIONS = ['automatic', ...AGENT_COLORS]`。
- 状态：`selectedIndex` 根据 `currentColor` 或 0 初始化。
- 上/下箭头移动，Enter 确认。选择 `'automatic'` 时调用 `onConfirm(undefined)`。
- 预览框展示带背景色或反色样式的 agent 名称。
- 使用 `AGENT_COLOR_TO_THEME_COLOR` 映射实际终端颜色值。

**导出：** `ColorPicker`

---

### 1.7 ModelSelector

**文件：** `agents/ModelSelector.tsx`

**用途：** 用于从标准列表中选择 agent model 的 `<Select>` 组件（`getAgentModelOptions()`）。如果当前 model 是完整 model ID 且不在标准 alias 列表里，会把它作为自定义选项插入。

**Props 接口：**
```typescript
interface ModelSelectorProps {
  initialModel?: string;
  onComplete: (model?: string) => void;
  onCancel?: () => void;
}
```

**关键行为：**
- 如果 `initialModel` 不在标准 options 列表中，会在前面插入 `{ value: initialModel, label: initialModel, description: "Current model (custom ID)" }`。
- 默认值：`initialModel ?? 'sonnet'`。
- 取消时：如果提供了 `onCancel()` 就调用它，否则调用 `onComplete(undefined)`。
- 渲染说明文案：`Model determines the agent's reasoning capabilities and speed.`。

**导出：** `ModelSelector`

---

### 1.8 ToolSelector

**文件：** `agents/ToolSelector.tsx`

**用途：** 选择 agent 可访问工具的多选 UI。把 tools 分成几个桶：Read-only、Edit、Execution、MCP（按 server）、Other。支持 “All tools” 通配选择。

**Props 接口：**
```typescript
type Props = {
  tools: Tools;
  initialTools: string[] | undefined;
  onComplete: (selectedTools: string[] | undefined) => void;
  onCancel?: () => void;
}

type ToolBucket = {
  name: string;
  toolNames: Set<string>;
  isMcp?: boolean;
};

type ToolBuckets = {
  READ_ONLY: ToolBucket;
  EDIT: ToolBucket;
  EXECUTION: ToolBucket;
  MCP: ToolBucket;
  OTHER: ToolBucket;
};
```

**关键行为：**
- MCP tools 按 server 名称通过 `getMcpServerBuckets()` 动态分组。
- 不属于任何桶的工具进入 OTHER。
- `AGENT_TOOL_NAME` 不会出现在可选列表中。
- 支持按单个 tool 切换；桶内支持 “Select all” / “Deselect all” 快捷操作。
- 上/下导航；Space/Enter 切换；Tab/Shift+Tab 在桶之间移动。
- 提交时：如果未选任何 tools 且没有开启 wildcard，则传空数组。若使用确认快捷方式且未选中任何项，也可以传 `undefined`（表示 all tools）。

**导出：** `ToolSelector`

---

### 1.9 agentFileUtils.ts

**文件：** `agents/agentFileUtils.ts`

**用途：** 读写 agent 定义 Markdown 文件的文件系统工具函数。

**导出：**
```typescript
function formatAgentAsMarkdown(
  agentType: string,
  whenToUse: string,
  tools: string[] | undefined,
  systemPrompt: string,
  color?: string,
  model?: string,
  memory?: AgentMemoryScope,
  effort?: EffortValue,
): string

function getAgentDirectoryPath(location: SettingSource): string
function getRelativeAgentDirectoryPath(location: SettingSource): string
function getNewAgentFilePath(agent: { source: SettingSource; agentType: string }): string
function getActualAgentFilePath(agent: AgentDefinition): string
function getNewRelativeAgentFilePath(agent: { source: SettingSource | 'built-in'; agentType: string }): string
function getActualRelativeAgentFilePath(agent: AgentDefinition): string
async function saveAgentToFile(...)
async function updateAgentFile(...)
async function deleteAgentFromFile(...)
```

**关键行为：**
- 所有写入都通过 `writeFileAndFlush` 完成，它在写完后调用 `handle.datasync()`，保证持久性。
- `formatAgentAsMarkdown` 会对 `whenToUse` 里的反斜杠、双引号和换行进行转义，适配 YAML 双引号字符串。
- 当 `tools` 为 `undefined` 或 `['*']`（表示允许所有 tools）时，会完全省略 tools 字段。

---

### 1.10 generateAgent.ts

**文件：** `agents/generateAgent.ts`

**用途：** 使用一次 LLM 调用（`queryModelWithoutStreaming`）从自然语言用户提示自动生成 agent 配置。返回结构化 JSON，包含 `identifier`、`whenToUse` 和 `systemPrompt`。

**导出：**
```typescript
type GeneratedAgent = {
  identifier: string
  whenToUse: string
  systemPrompt: string
}

async function generateAgent(
  userPrompt: string,
  model: ModelName,
  existingIdentifiers: string[],
  abortSignal: AbortSignal,
): Promise<GeneratedAgent>
```

**关键行为：**
- 使用详细 system prompt（`AGENT_CREATION_SYSTEM_PROMPT`）指导 Claude 设计 agent persona、写 system prompt、创建 identifier（小写、2-4 个词、只用连字符、避免 “helper”/“assistant”）。
- 当 `isAutoMemoryEnabled()` 为真时，会把 `AGENT_MEMORY_INSTRUCTIONS` 追加到 system prompt。
- 通过 `prependUserContext()` 加入用户上下文。
- 从响应中解析 JSON；如果直接解析失败，会回退到正则提取。
- 触发 `tengu_agent_definition_generated` 分析事件。
- 缺少字段或字段无效时抛错。

---

### 1.11 types.ts

**文件：** `agents/types.ts`

**用途：** agents UI 状态机的共享类型定义。

**导出：**
```typescript
const AGENT_PATHS = {
  FOLDER_NAME: '.claude',
  AGENTS_DIR: 'agents',
} as const

type ModeState =
  | { mode: 'main-menu' }
  | { mode: 'list-agents'; source: SettingSource | 'all' | 'built-in' }
  | { mode: 'agent-menu'; agent: AgentDefinition; previousMode: ModeState }
  | { mode: 'view-agent'; agent: AgentDefinition; previousMode: ModeState }
  | { mode: 'create-agent' }
  | { mode: 'edit-agent'; agent: AgentDefinition; previousMode: ModeState }
  | { mode: 'delete-confirm'; agent: AgentDefinition; previousMode: ModeState }

type AgentValidationResult = {
  isValid: boolean
  warnings: string[]
  errors: string[]
}
```

---

### 1.12 utils.ts

**文件：** `agents/utils.ts`

**导出：**
```typescript
function getAgentSourceDisplayName(
  source: SettingSource | 'all' | 'built-in' | 'plugin'
): string
// 返回：'Agents' | 'Built-in agents' | 'Plugin agents' | capitalize(getSettingSourceName(source))
```

---

### 1.13 validateAgent.ts

**文件：** `agents/validateAgent.ts`

**导出：**
```typescript
type AgentValidationResult = {
  isValid: boolean
  errors: string[]
  warnings: string[]
}

function validateAgentType(agentType: string): string | null
// 返回错误信息；若合法则返回 null。
// 规则：必填，/^[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]$/，长度 3-50。

function validateAgent(
  agent: Omit<CustomAgentDefinition, 'location'>,
  availableTools: Tools,
  existingAgents: AgentDefinition[],
): AgentValidationResult
// 校验：agentType、whenToUse（最少 10 字符 / 最多 5000）、tools 数组、
// system prompt（最少 20 字符 / 最多 10000 字符为警告阈值）。
// 检查跨来源的重复 agentType。
// 使用 resolveAgentTools 检查无效 tool 名称。
```

---

### 1.14 new-agent-creation/CreateAgentWizard

**文件：** `agents/new-agent-creation/CreateAgentWizard.tsx`

**用途：** 通过在 `<WizardProvider>` 内组合步骤组件来组装多步新 agent 创建向导。根据 feature flag 条件性包含 `MemoryStep`。

**Props 接口：**
```typescript
type Props = {
  tools: Tools;
  existingAgents: AgentDefinition[];
  onComplete: (message: string) => void;
  onCancel: () => void;
}
```

**Wizard Step 顺序（从 0 开始）：**
0. `LocationStep` - 项目作用域 vs 个人作用域
1. `MethodStep` - 用 Claude 生成 vs 手工配置
2. `GenerateStep` - 自然语言提示词 → LLM 生成（手工模式跳过）
3. `TypeStep` - agent 标识符 / 名称
4. `PromptStep` - system prompt 文本
5. `DescriptionStep` - whenToUse 描述
6. `ToolsStep` - tool 选择
7. `ModelStep` - model 选择
8. `ColorStep` - 颜色选择
9. `MemoryStep` - memory 作用域（取决于 `isAutoMemoryEnabled()`）
10. `ConfirmStepWrapper` - 检查并保存

**关键行为：**
- WizardProvider 标题：`"Create new agent"`，`showStepCounter: false`。
- WizardProvider 的 `onComplete` 是 no-op（真正完成由 `ConfirmStepWrapper` 处理）。
- 通过 `onCancel` 传给 WizardProvider。

---

### 1.15 向导步骤

#### LocationStep
提供两个选项：`"Project (.claude/agents/)"`（`projectSettings`）或 `"Personal (~/.claude/agents/)"`（`userSettings`）。更新 `wizardData.location` 并调用 `goNext()`。

#### MethodStep
提供 `"Generate with Claude (recommended)"` 或 `"Manual configuration"`。生成模式会设置 `wizardData.method = 'generate'`、`wasGenerated: true`、`goNext()`；手工模式设置 `method: 'manual'`、`wasGenerated: false`，并通过 `goToStep(3)` 跳到第 3 步。

#### GenerateStep
- 文本输入，用于填写 agent 的自然语言描述。
- 提交时调用 `generateAgent(prompt, model, existingIdentifiers, abortSignal)`。
- 生成期间显示动画 `<Spinner>`。
- 成功后填充 `wizardData.agentType`、`systemPrompt`、`whenToUse`、`generatedAgent`。
- 生成完成后直接跳到第 6 步（ToolsStep），跳过手工的名称/prompt/description 步骤。
- 生成期间按 Esc 会通过 abort signal 取消。
- 支持外部编辑器（`chat:externalEditor` keybinding）。

#### TypeStep
Props: `{ existingAgents: AgentDefinition[] }`。用于输入 agent 标识符。通过 `validateAgentType()` 校验，并更新 `wizardData.agentType`。

#### PromptStep
用于 system prompt 的大文本输入。强制至少 20 个字符。支持外部编辑器。更新 `wizardData.systemPrompt`。

#### DescriptionStep
用于 `whenToUse` 的文本输入。必填，至少 1 个字符。支持外部编辑器。更新 `wizardData.whenToUse`。

#### ToolsStep
Props: `{ tools: Tools }`。在 `WizardDialogLayout` 中包裹 `<ToolSelector>`。更新 `wizardData.selectedTools`。

#### ModelStep
在 `WizardDialogLayout` 中包裹 `<ModelSelector>`。更新 `wizardData.selectedModel`。

#### ColorStep
在 `WizardDialogLayout` 中包裹 `<ColorPicker>`。确认后构建包含所有累积数据的 `wizardData.finalAgent` 对象，并更新 `wizardData.selectedColor`。

#### MemoryStep
条件步骤（仅在 `isAutoMemoryEnabled()` 时显示）。提供按推荐作用域排序的 memory scope 选项（项目 agent 先项目后个人，个人 agent 先个人后项目），包括 “None” 选项。更新 `wizardData.selectedMemory`。

#### ConfirmStep / ConfirmStepWrapper
Props: `{ tools: Tools; existingAgents: AgentDefinition[]; onComplete: (message: string) => void }`。展示所有已选项摘要，通过 `validateAgent()` 校验，调用 `saveAgentToFile()`，然后刷新应用状态中的 agents，并调用 `onComplete(message)`。

---

## 2. permissions/

permissions 子系统负责渲染交互式对话框，询问用户是否批准、拒绝或为工具使用配置规则。每种工具都有自己的权限请求组件。

### 2.1 核心类型与 PermissionRequest

**文件：** `permissions/PermissionRequest.tsx`

**用途：** 顶层分发器，根据每个工具类型选择正确的权限请求组件并渲染，同时处理通知和空闲检测。

**关键类型：**
```typescript
type PermissionRequestProps<Input extends AnyObject = AnyObject> = {
  toolUseConfirm: ToolUseConfirm<Input>;
  toolUseContext: ToolUseContext;
  onDone(): void;
  onReject(): void;
  verbose: boolean;
  workerBadge: WorkerBadgeProps | undefined;
  setStickyFooter?: (jsx: React.ReactNode | null) => void;
}

type ToolUseConfirm<Input extends AnyObject = AnyObject> = {
  assistantMessage: AssistantMessage;
  tool: Tool<Input>;
  description: string;
  input: z.infer<Input>;
  toolUseContext: ToolUseContext;
  toolUseID: string;
  permissionResult: PermissionDecision;
  permissionPromptStartTimeMs: number;
  classifierCheckInProgress?: boolean;
  classifierAutoApproved?: boolean;
  classifierMatchedRule?: string;
  workerBadge?: WorkerBadgeProps;
  onUserInteraction(): void;
  onAbort(): void;
  onDismissCheckmark?(): void;
  onAllow(updatedInput, permissionUpdates: PermissionUpdate[], feedback?, contentBlocks?): void;
  onReject(feedback?, contentBlocks?): void;
  recheckPermission(): Promise<void>;
}
```

**Tool → Component 映射：**
| 工具 | 组件 |
|------|-----------|
| FileEditTool | FileEditPermissionRequest |
| FileWriteTool | FileWritePermissionRequest |
| BashTool | BashPermissionRequest |
| PowerShellTool | PowerShellPermissionRequest |
| WebFetchTool | WebFetchPermissionRequest |
| NotebookEditTool | NotebookEditPermissionRequest |
| ExitPlanModeV2Tool | ExitPlanModePermissionRequest |
| EnterPlanModeTool | EnterPlanModePermissionRequest |
| SkillTool | SkillPermissionRequest |
| AskUserQuestionTool | AskUserQuestionPermissionRequest |
| GlobTool / GrepTool / FileReadTool | FilesystemPermissionRequest |
| ReviewArtifactTool（feature flag） | ReviewArtifactPermissionRequest |
| WorkflowTool（feature flag） | WorkflowPermissionRequest |
| MonitorTool（feature flag） | MonitorPermissionRequest |
| default | FallbackPermissionRequest |

**导出：** `PermissionRequest`, `PermissionRequestProps`, `ToolUseConfirm`

---

### 2.2 PermissionDialog

**文件：** `permissions/PermissionDialog.tsx`

**用途：** 所有工具权限请求共用的视觉容器。渲染标题栏（可带 worker badge）、副标题和 children，外层包一层样式化边框盒。

**Props 接口：**
```typescript
type Props = {
  title: string;
  subtitle?: React.ReactNode;
  color?: keyof Theme;
  titleColor?: keyof Theme;
  innerPaddingX?: number;
  workerBadge?: WorkerBadgeProps;
  titleRight?: React.ReactNode;
  children: React.ReactNode;
}
```

**关键行为：**
- 渲染 `<PermissionRequestTitle>`，传入 title、subtitle、可选颜色覆盖和 workerBadge。
- 标题行使用 `justifyContent="space-between"`，让 `titleRight` 右对齐。
- children 在内部 `Box` 中渲染，并设置 `paddingX={innerPaddingX}`。

**导出：** `PermissionDialog`

---

### 2.3 BashPermissionRequest

**文件：** `permissions/BashPermissionRequest/BashPermissionRequest.tsx`

**用途：** `BashTool` 的权限对话框。处理 classifier 自动批准动画、sed 编辑检测（重定向到 `SedEditPermissionRequest`）、sandbox 检测，以及一组更丰富的选项。

**关键行为：**
- `ClassifierCheckingSubtitle`：独立子组件，以 20fps 渲染动画 shimmer “Attempting to auto-approve…” 文本，避免整个对话框重渲染。
- 根据 `classifierCheckInProgress` prop 显示 shimmer 副标题。
- 如果命令匹配 `sed` 编辑模式，则转给 `SedEditPermissionRequest`。
- 如果 `shouldUseSandbox()`，显示 sandbox 专用选项。
- 选项由 `bashToolUseOptions()` 计算。
- 通过 `usePermissionRequestLogging()` 记录权限决策。
- 支持危险命令警告显示。

**导出：** `BashPermissionRequest`

---

### 2.4 FilePermissionDialog

**文件：** `permissions/FilePermissionDialog/FilePermissionDialog.tsx`

**用途：** 文件操作权限的通用可复用对话框（供 FileEdit、FileWrite、NotebookEdit 权限请求使用）。处理路径显示、符号链接检测、IDE diff 集成和选项渲染。

**Props 接口：**
```typescript
type FilePermissionDialogProps<T extends ToolInput = ToolInput> = {
  toolUseConfirm: ToolUseConfirm;
  toolUseContext: ToolUseContext;
  onDone: () => void;
  onReject: () => void;
  title: string;
  subtitle?: React.ReactNode;
  question?: string | React.ReactNode;
  content?: React.ReactNode;
  completionType?: CompletionType;
  languageName?: string;
  path: string | null;
  parseInput: (input: unknown) => T;
  operationType?: FileOperationType;
  ideDiffSupport?: IDEDiffSupport<T>;
  workerBadge: WorkerBadgeProps | undefined;
}
```

**关键行为：**
- 若未覆盖，语言名会通过 `getLanguageName(path)` 异步推导。
- 当 `operationType !== 'read'` 时，检查符号链接目标。
- 当可用 IDE diff 时，显示 `<ShowInIDEPrompt>`。
- 通过 `usePermissionRequestLogging()` 记录日志。

**导出：** `FilePermissionDialog`, `FilePermissionDialogProps`

---

### 2.5 WorkerBadge

**文件：** `permissions/WorkerBadge.tsx`

**用途：** 显示请求权限的 swarm worker 的彩色徽章。

**Props 接口：**
```typescript
type WorkerBadgeProps = {
  name: string;
  color: string;
}
```

**渲染：** `● @{name}`，圆点使用 worker 的颜色。

**导出：** `WorkerBadge`, `WorkerBadgeProps`

---

### 2.6 其他权限请求组件

每个组件都遵循 `PermissionRequestProps` 接口，并在 `<PermissionDialog>` 或 `<FilePermissionDialog>` 内渲染。

- `FileEditPermissionRequest`：渲染文件编辑 diff，使用 `FilePermissionDialog`，`operationType: 'write'`。
- `FileWritePermissionRequest`：渲染完整的拟写入文件内容，使用 `FilePermissionDialog`，`operationType: 'write'`。
- `FilesystemPermissionRequest`：供 GlobTool / GrepTool / FileReadTool 使用，展示只读访问对话框，`operationType: 'read'`。
- `NotebookEditPermissionRequest`：用于 Jupyter notebook 编辑，渲染 notebook cell diff，使用 `languageName` 由 cell 类型推导的 `FilePermissionDialog`。
- `WebFetchPermissionRequest`：展示将要 fetch 的 URL 和 fetch 选项。
- `PowerShellPermissionRequest`：类似 BashPermissionRequest，但用于 PowerShell 命令，使用 `powershellToolUseOptions()`。
- `SedEditPermissionRequest`：当 BashPermissionRequest 检测到 sed 编辑模式时渲染，展示 sed 命令会生成的文件 diff。
- `EnterPlanModePermissionRequest`：进入 plan mode 的简单确认。
- `ExitPlanModePermissionRequest`：完整的 plan 审查，并使用 sticky footer 保持响应选项可见。
- `SkillPermissionRequest`：用于执行一个 skill/command。
- `AskUserQuestionPermissionRequest`：多问题表单，带导航。子组件包括 `PreviewBox`、`PreviewQuestionView`、`QuestionNavigationBar`、`QuestionView`、`SubmitQuestionsView`、`use-multiple-choice-state.ts`。
- `ComputerUseApproval`：用于 computer-use 动作。
- `FallbackPermissionRequest`：未识别工具类型的通用回退，展示工具名和描述。
- `SandboxPermissionRequest`：当命令以 sandbox mode 运行时显示。

---

### 2.7 PermissionDecisionDebugInfo

**文件：** `permissions/PermissionDecisionDebugInfo.tsx`

**用途：** 在 verbose mode 开启时，展示权限决策的调试信息（决策原因、规则细节、classifier 结果）。

---

### 2.8 PermissionExplanation

**文件：** `permissions/PermissionExplanation.tsx`

**用途：** 展示请求某条权限规则的原因说明。`usePermissionExplainerUI` hook 管理 explainer 区域的展开/折叠状态。

**导出：** `PermissionExplainerContent`, `usePermissionExplainerUI`

---

### 2.9 PermissionPrompt

**文件：** `permissions/PermissionPrompt.tsx`

**用途：** 将权限请求 UI 包裹在全屏布局管理和 sticky footer 支持中，供 plan mode 响应使用。

---

### 2.10 PermissionRequestTitle

**文件：** `permissions/PermissionRequestTitle.tsx`

**用途：** 渲染权限对话框的彩色标题栏。标题文本使用 permission 颜色，下面可带可选 worker badge。

**Props 接口：**
```typescript
type Props = {
  title: string;
  subtitle?: React.ReactNode;
  color?: keyof Theme;
  workerBadge?: WorkerBadgeProps;
}
```

---

### 2.11 PermissionRuleExplanation

**文件：** `permissions/PermissionRuleExplanation.tsx`

**用途：** 显示权限规则建议，例如用户点击 “Always allow” 或 “Always deny” 时将创建的规则（如 “Allow bash: git *”）。

---

### 2.12 WorkerPendingPermission

**文件：** `permissions/WorkerPendingPermission.tsx`

**用途：** 在 swarm coordinator 视图中，用于展示某个 worker agent 的待处理权限请求。

---

### 2.13 hooks.ts

**文件：** `permissions/hooks.ts`

**导出：**
```typescript
type UnaryEvent = {
  completion_type: CompletionType;
  language_name: string | Promise<string>;
}

function usePermissionRequestLogging(
  toolUseConfirm: ToolUseConfirm,
  unaryEvent: UnaryEvent,
): void
```

**关键行为：**
- 挂载时触发 `tengu_permission_request_start`。
- 卸载时触发带结果信息的 `tengu_permission_request_end`。
- 将 `permissionResult` 转为结构化日志字符串。

---

### 2.14 shellPermissionHelpers.tsx

**文件：** `permissions/shellPermissionHelpers.tsx`

**用途：** `BashPermissionRequest` 和 `PowerShellPermissionRequest` 共用的 JSX 辅助函数与选项构建器（例如构建 approve/deny/always-allow 选项列表）。

---

### 2.15 useShellPermissionFeedback.ts

**文件：** `permissions/useShellPermissionFeedback.ts`

**用途：** 管理 shell 权限对话框里在用户选择 “No (feedback)” 时出现的反馈文本输入框。

---

### 2.16 utils.ts

**文件：** `permissions/utils.ts`

**导出：**
```typescript
function logUnaryPermissionEvent(
  toolUseConfirm: ToolUseConfirm,
  unaryEvent: UnaryEvent,
  result: string,
): void
```

---

### 2.17 rules/ 子目录

`/permissions` 命令屏幕的权限规则管理 UI。

| 文件 | 作用 |
|------|------|
| `AddPermissionRules.tsx` | 新增 allow/deny 规则的表单 |
| `AddWorkspaceDirectory.tsx` | 新增受信任工作区目录的表单 |
| `PermissionRuleDescription.tsx` | 渲染规则的人类可读说明 |
| `PermissionRuleInput.tsx` | 规则模式的文本输入与校验 |
| `PermissionRuleList.tsx` | 显示现有规则并支持删除 |
| `RecentDenialsTab.tsx` | 显示最近被拒绝的工具使用，可转为规则 |
| `RemoveWorkspaceDirectory.tsx` | 移除工作区目录的确认对话框 |
| `WorkspaceTab.tsx` | 显示受信任工作区目录的标签页 |

---

## 3. design-system/

组成所有终端 UI 视觉基础的可复用原始组件。

### 3.1 Byline

**文件：** `design-system/Byline.tsx`

**用途：** 用 middot 分隔符（` · `）串联 children，用于内联元信息展示。会自动过滤 null/undefined/false children。

**Props 接口：**
```typescript
type Props = {
  children: React.ReactNode;
}
```

**关键行为：**
- 使用 `Children.toArray()`，它会过滤 falsy nodes。
- 如果没有有效 children，返回 `null`。
- 仅在相邻有效元素之间渲染分隔符。

**导出：** `Byline`

---

### 3.2 Dialog

**文件：** `design-system/Dialog.tsx`

**用途：** 确认/取消对话框容器。注册 `confirm:no` 和 Ctrl+C/D keybindings。展示标题、可选副标题、children，以及键盘提示页脚。

**Props 接口：**
```typescript
type DialogProps = {
  title: React.ReactNode;
  subtitle?: React.ReactNode;
  children: React.ReactNode;
  onCancel: () => void;
  color?: keyof Theme;
  hideInputGuide?: boolean;
  hideBorder?: boolean;
  inputGuide?: (exitState: ExitState) => React.ReactNode;
  isCancelActive?: boolean;
}
```

**关键行为：**
- `isCancelActive=false` 时，禁用 `confirm:no` 和退出 keybindings（适合内部 TextInput 需要 Esc 的场景）。
- 默认输入提示：`Enter to confirm · Esc to cancel`（若存在退出待确认状态，则显示 `Press X again to exit`）。
- 除非 `hideBorder`，否则内容包在 `<Pane>` 中。
- 标题使用粗体并按 `color` 着色。

**导出：** `Dialog`

---

### 3.3 Divider

**文件：** `design-system/Divider.tsx`

**Props 接口：**
```typescript
type DividerProps = {
  width?: number;
  color?: keyof Theme;
  char?: string;
  padding?: number;
  title?: string;
}
```

**关键行为：**
- 没有 title 时：在 `<Text>` 中输出 `char.repeat(effectiveWidth)`。
- 有 title 时：左填充 + 空格 + title + 空格 + 右填充；左右填充分配尽量均匀，左边取 floor。
- 通过 `useTerminalSize()` 获取终端宽度。

**导出：** `Divider`

---

### 3.4 FuzzyPicker

**文件：** `design-system/FuzzyPicker.tsx`

**用途：** 功能完整的模糊搜索选择器，可选预览面板。支持 up/down/down-to-top 列表方向、Tab/Shift+Tab 辅助动作、右侧或底部预览、匹配数标签。

**Props 接口：**
```typescript
type PickerAction<T> = {
  action: string;
  handler: (item: T) => void;
}

type Props<T> = {
  title: string;
  placeholder?: string;
  initialQuery?: string;
  items: readonly T[];
  getKey: (item: T) => string;
  renderItem: (item: T, isFocused: boolean) => React.ReactNode;
  renderPreview?: (item: T) => React.ReactNode;
  previewPosition?: 'bottom' | 'right';
  visibleCount?: number;
  direction?: 'down' | 'up';
  onQueryChange: (query: string) => void;
  onSelect: (item: T) => void;
  onTab?: PickerAction<T>;
  onShiftTab?: PickerAction<T>;
  onFocus?: (item: T | undefined) => void;
  onCancel: () => void;
  emptyMessage?: string | ((query: string) => string);
  matchLabel?: string;
  selectAction?: string;
  extraHints?: React.ReactNode;
}
```

**关键行为：**
- 常量：`DEFAULT_VISIBLE=8`、`CHROME_ROWS=10`、`MIN_VISIBLE=2`。
- 会根据终端高度自动调整可见数量。
- 当聚焦项变化时触发 `onFocus`。
- `direction='up'` 时，`items[0]` 显示在底部（atuin 风格）；箭头方向与屏幕方向一致。

**导出：** `FuzzyPicker`

---

### 3.5 KeyboardShortcutHint

**文件：** `design-system/KeyboardShortcutHint.tsx`

**用途：** 渲染键盘快捷键提示，比如 “ctrl+o to expand” 或 “(tab to toggle)”。

**Props 接口：**
```typescript
type Props = {
  shortcut: string;
  action: string;
  parens?: boolean;
  bold?: boolean;
}
```

**导出：** `KeyboardShortcutHint`

---

### 3.6 ListItem

**文件：** `design-system/ListItem.tsx`

**用途：** 选择型 UI 的标准列表项，包含指针（❯）、勾选（✓）、滚动提示箭头、聚焦/选中颜色和禁用状态。

**Props 接口：**
```typescript
type ListItemProps = {
  isFocused: boolean;
  isSelected?: boolean;
  children: ReactNode;
  description?: string;
  showScrollDown?: boolean;
  showScrollUp?: boolean;
  styled?: boolean;
  disabled?: boolean;
  declareCursor?: boolean;
}
```

**关键行为：**
- 聚焦且未选中：指针（❯）使用 suggestion 颜色。
- 选中但未聚焦：勾选（✓）使用 suggestion 颜色。
- 禁用：不显示指示符，文本变暗。
- `styled=false` 时，children 原样渲染，便于自定义样式。

**导出：** `ListItem`

---

### 3.7 LoadingState

**文件：** `design-system/LoadingState.tsx`

**用途：** 异步加载状态的 spinner + 文案。

**Props 接口：**
```typescript
type LoadingStateProps = {
  message: string;
  bold?: boolean;
  dimColor?: boolean;
  subtitle?: string;
}
```

**导出：** `LoadingState`

---

### 3.8 Pane

**文件：** `design-system/Pane.tsx`

**用途：** 一个由彩色顶部分隔线界定的终端区域，所有斜杠命令屏幕都会用到。

**Props 接口：**
```typescript
type PaneProps = {
  children: React.ReactNode;
  color?: keyof Theme;
}
```

**关键行为：**
- 在 modal 内渲染时（`useIsInsideModal()`）：跳过 Divider（modal 框架本身就是边框），并以 `paddingX={1}` 和 `flexShrink={0}` 渲染。
- 普通渲染：`paddingTop={1}` + `<Divider color={color}>` + `<Box paddingX={2}>children</Box>`。

**导出：** `Pane`

---

### 3.9 ProgressBar

**文件：** `design-system/ProgressBar.tsx`

**用途：** 使用 Unicode block 字符的横向文本进度条。

**Props 接口：**
```typescript
type Props = {
  ratio: number;
  width: number;
  fillColor?: keyof Theme;
  emptyColor?: keyof Theme;
}
```

**关键行为：**
- 使用 9 个 block 字符：`[' ', '▏', '▎', '▍', '▌', '▋', '▊', '▉', '█']`。
- 将 ratio 限制在 [0, 1]。
- 整格填充：`Math.floor(ratio * width)`。
- 部分格：`Math.floor(remainder * BLOCKS.length)`。
- `fillColor` 作为文字颜色；`emptyColor` 作为背景。

**导出：** `ProgressBar`

---

### 3.10 Ratchet

**文件：** `design-system/Ratchet.tsx`

**用途：** 通过保持已见最大高度来防止布局跳动/收缩。用于内容可以增长但重渲染时不应缩回去的场景。

**Props 接口：**
```typescript
type Props = {
  children: React.ReactNode;
  lock?: 'always' | 'offscreen';
}
```

**关键行为：**
- `lock='always'`：始终强制 minHeight（内部 Box 使用已见最大高度，并受终端行数上限约束）。
- `lock='offscreen'`：仅在元素在终端视口外时强制 minHeight。
- 使用 `useTerminalViewport()` 判断可见性。
- `useLayoutEffect` 在每次渲染后测量内部内容高度。

**导出：** `Ratchet`

---

### 3.11 StatusIcon

**文件：** `design-system/StatusIcon.tsx`

**用途：** 渲染彩色状态指示图标。

**Props 接口：**
```typescript
type Status = 'success' | 'error' | 'warning' | 'info' | 'pending' | 'loading'
type Props = {
  status: Status;
  withSpace?: boolean;
}
```

**Status Config:**
| 状态 | 图标 | 颜色 |
|--------|------|-------|
| success | ✓ (`figures.tick`) | success（绿色） |
| error | ✗ (`figures.cross`) | error（红色） |
| warning | ⚠ (`figures.warning`) | warning（黄色） |
| info | ℹ (`figures.info`) | suggestion（蓝色） |
| pending | ○ (`figures.circle`) | dimColor |
| loading | … | dimColor |

**导出：** `StatusIcon`

---

### 3.12 Tabs

**文件：** `design-system/Tabs.tsx`

**用途：** 带键盘导航的标签页容器。支持受控和非受控模式、固定内容高度、可选 banner，以及由内容触发的 tab 切换。

**Props 接口：**
```typescript
type TabsProps = {
  children: Array<React.ReactElement<TabProps>>;
  title?: string;
  color?: keyof Theme;
  defaultTab?: string;
  hidden?: boolean;
  useFullWidth?: boolean;
  selectedTab?: string;
  onTabChange?: (tabId: string) => void;
  banner?: React.ReactNode;
  disableNavigation?: boolean;
  initialHeaderFocused?: boolean;
  contentHeight?: number;
  navFromContent?: boolean;
}

type TabsContextValue = {
  selectedTab: string | undefined;
  width: number | undefined;
  headerFocused: boolean;
  focusHeader: () => void;
  blurHeader: () => void;
  registerOptIn: () => () => void;
}
```

**关键行为：**
- Tab 组件通过 `TabsContext` 识别自己是否是当前选中标签。
- 左/右箭头或 Tab 键在 header 聚焦时切换标签。
- `navFromContent=true` 时，内容区也能触发标签切换。

**导出：** `Tabs`, `TabsContext`, `TabsContextValue`

---

### 3.13 ThemeProvider

**文件：** `design-system/ThemeProvider.tsx`

**用途：** 为组件树提供主题状态（dark/light/auto）。`auto` 通过 OSC 11 终端查询检测系统主题。

**Props 接口：**
```typescript
type Props = {
  children: React.ReactNode;
  initialState?: ThemeSetting;
  onThemeSave?: (setting: ThemeSetting) => void;
}

type ThemeContextValue = {
  themeSetting: ThemeSetting;
  setThemeSetting: (s: ThemeSetting) => void;
  setPreviewTheme: (s: ThemeSetting) => void;
  savePreview: () => void;
  cancelPreview: () => void;
  currentTheme: ThemeName;
}
```

**关键行为：**
- 在 theme picker 交互期间，`previewTheme` 优先于 `themeSetting`。
- 系统主题初始值来自 `$COLORFGBG` 环境变量，之后由 OSC 11 watcher 修正。
- Provider 外默认主题（供测试使用）：`'dark'`。

**导出：** `ThemeProvider`, `ThemeContext`, `useTheme`

---

### 3.14 ThemedBox

**文件：** `design-system/ThemedBox.tsx`

**用途：** 主题感知的 Box 组件，会把所有 border/background 颜色 prop 中的 theme key（`keyof Theme`）解析成真实终端颜色，再传给底层 Ink `Box`。

**Props 类型：** `BaseStylesWithoutColors & ThemedColorProps & EventHandlerProps`

```typescript
type ThemedColorProps = {
  borderColor?: keyof Theme | Color;
  borderTopColor?: keyof Theme | Color;
  borderBottomColor?: keyof Theme | Color;
  borderLeftColor?: keyof Theme | Color;
  borderRightColor?: keyof Theme | Color;
  backgroundColor?: keyof Theme | Color;
}
```

**关键行为：**
- `resolveColor()`：原始颜色（`#`、`rgb(`、`ansi256(`、`ansi:`）直接透传；theme key 则走查找。

**导出：** `ThemedBox`（默认导出）、`Props`

---

### 3.15 ThemedText

**文件：** `design-system/ThemedText.tsx`

**用途：** 主题感知的 Text 组件，会解析 `keyof Theme` 的颜色值，并支持 `TextHoverColorContext` 的级联着色。

**Props 接口：**
```typescript
type Props = {
  color?: keyof Theme | Color;
  backgroundColor?: keyof Theme;
  dimColor?: boolean;
  bold?: boolean;
  italic?: boolean;
  underline?: boolean;
  strikethrough?: boolean;
  inverse?: boolean;
  wrap?: Styles['textWrap'];
  children?: ReactNode;
}
```

**导出：** `ThemedText`, `Props`, `TextHoverColorContext`

---

### 3.16 color.ts

**文件：** `design-system/color.ts`

**用途：** 柯里化的主题感知着色函数。

**导出：**
```typescript
function color(
  c: keyof Theme | Color | undefined,
  theme: ThemeName,
  type: ColorType = 'foreground',
): (text: string) => string
```

**关键行为：**
- 原始颜色值不会走 theme lookup。
- theme key 值会在 `getTheme(theme)` 中查找，并交给 `colorize()`。

---

## 4. wizard/

供 agent 创建以及未来其他流程使用的通用多步向导框架。

### 4.1 WizardProvider

**文件：** `wizard/WizardProvider.tsx`

**用途：** 管理向导状态的上下文提供者：当前 step、数据累积、导航历史和完成状态。

**Props 接口：**
```typescript
type WizardProviderProps<T> = {
  steps: WizardStepComponent<T>[];
  initialData?: Partial<T>;
  onComplete: (data: T) => void;
  onCancel: () => void;
  children?: React.ReactNode;
  title?: string;
  showStepCounter?: boolean;
}

type WizardContextValue<T> = {
  currentStepIndex: number;
  totalSteps: number;
  wizardData: Partial<T>;
  updateWizardData: (partial: Partial<T>) => void;
  goNext: () => void;
  goBack: () => void;
  goToStep: (index: number) => void;
  cancel: () => void;
  title: string | undefined;
  showStepCounter: boolean;
}
```

**关键行为：**
- 维护 `navigationHistory` 栈以支持 `goBack()`。
- 最后一步调用 `goNext()` 会设置 `isCompleted=true`，随后在 effect 中触发 `onComplete(wizardData)`。
- 调用 `useExitOnCtrlCDWithKeybindings()` 在向导层注册 Ctrl+C/D。

**导出：** `WizardProvider`, `WizardContext`

---

### 4.2 WizardDialogLayout

**文件：** `wizard/WizardDialogLayout.tsx`

**用途：** 向导步骤的标准布局——把内容包在 `<Dialog>` 中，标题带 step counter，并带导航页脚。

**Props 接口：**
```typescript
type Props = {
  title?: string;
  color?: keyof Theme;
  children: ReactNode;
  subtitle?: string;
  footerText?: ReactNode;
}
```

**关键行为：**
- 当 `showStepCounter` 为 true 时，标题格式为 `"${title} (${currentStep + 1}/${totalSteps})"`。
- 内层 Dialog 使用 `isCancelActive={false}`（由 wizard 自己管理取消）。
- 在 Dialog 下方渲染 `<WizardNavigationFooter instructions={footerText}>`。

**导出：** `WizardDialogLayout`

---

### 4.3 WizardNavigationFooter

**文件：** `wizard/WizardNavigationFooter.tsx`

**用途：** 在向导对话框底部显示键盘提示。

**Props 接口：**
```typescript
type Props = {
  instructions?: ReactNode;
}
```

---

### 4.4 useWizard

**文件：** `wizard/useWizard.ts`

**用途：** 在步骤组件内部访问 `WizardContext`。若在 `WizardProvider` 外使用则抛错。

**导出：**
```typescript
function useWizard<T extends Record<string, unknown> = Record<string, unknown>>(): WizardContextValue<T>
```

---

### 4.5 index.ts

重新导出 `WizardProvider`、`useWizard` 和与 wizard 相关的类型。

---

## 5. mcp/

Model Context Protocol server 管理 UI。

### 文件概览

| 文件 | 作用 |
|------|------|
| `CapabilitiesSection.tsx` | 渲染 server 能力列表（tools、resources、prompts） |
| `ElicitationDialog.tsx` | MCP elicitation 对话框（server 请求结构化用户输入） |
| `MCPAgentServerMenu.tsx` | 管理 MCP agent 类型 servers 的菜单 |
| `MCPListPanel.tsx` | 显示所有已连接 MCP servers 及其状态的面板 |
| `MCPReconnect.tsx` | 用于重新连接断开 server 的 UI |
| `MCPRemoteServerMenu.tsx` | 远程 MCP server 配置菜单 |
| `MCPSettings.tsx` | 顶层 MCP 设置屏幕 |
| `MCPStdioServerMenu.tsx` | stdio 类型 MCP server 的菜单 |
| `MCPToolDetailView.tsx` | 单个 MCP tool 的详情视图 |
| `MCPToolListView.tsx` | MCP server 提供的 tool 列表 |
| `McpParsingWarnings.tsx` | 显示 MCP 配置中的 YAML/JSON 解析警告 |
| `index.ts` | 重新导出 |
| `utils/reconnectHelpers.tsx` | server 重连逻辑的辅助函数 |

**Key Types (from MCPSettings):**
```typescript
// Server status display combines name, transport type, connection state, tool count
```

---

## 6. memory/

agent memory 文件管理 UI。

### 文件

| 文件 | 作用 |
|------|------|
| `MemoryFileSelector.tsx` | 用于选择要查看/编辑的 memory 文件的文件选择器 |
| `MemoryUpdateNotification.tsx` | agent 更新 memory 时显示的 toast 风格通知 |

**MemoryFileSelector Props:**
```typescript
type Props = {
  onSelect: (filePath: string) => void;
  onCancel: () => void;
}
```

---

## 7. tasks/

后台任务和远程会话监控 UI。

### 文件概览

| 文件 | 作用 |
|------|------|
| `AsyncAgentDetailDialog.tsx` | 异步/排队中 agent 任务的详情对话框 |
| `BackgroundTask.tsx` | 任务列表中的单条后台任务行 |
| `BackgroundTaskStatus.tsx` | 后台任务状态徽章 |
| `BackgroundTasksDialog.tsx` | 列出所有后台任务的完整对话框 |
| `DreamDetailDialog.tsx` | “dream”（autoDream 后台整合）任务的详情 |
| `InProcessTeammateDetailDialog.tsx` | 处理中 swarm teammate 的详情 |
| `RemoteSessionDetailDialog.tsx` | 远程 Claude Code 会话的详情 |
| `RemoteSessionProgress.tsx` | 远程会话活动的进度展示 |
| `ShellDetailDialog.tsx` | shell 后台任务的详情 |
| `ShellProgress.tsx` | shell 命令的进度行 |
| `renderToolActivity.tsx` | 渲染运行中任务的当前 tool 活动 |
| `taskStatusUtils.tsx` | 任务状态显示的工具函数 |

---

## 8. teams/

swarm / multi-agent team 状态 UI。

### 文件

| 文件 | 作用 |
|------|------|
| `TeamStatus.tsx` | 显示所有活动 team 成员（swarm workers）的状态 |
| `TeamsDialog.tsx` | 管理 team 组成并查看 worker 详情的对话框 |

---

## 9. diff/

文件 diff 查看 UI。

### 文件

| 文件 | 作用 |
|------|------|
| `DiffDetailView.tsx` | 支持滚动的全屏 diff 详情 |
| `DiffDialog.tsx` | 包裹 `DiffDetailView` 的对话框 |
| `DiffFileList.tsx` | 带统计摘要的变更文件列表 |

---

## 10. grove/

Grove（共享项目工作区）集成。

### 文件

| 文件 | 作用 |
|------|------|
| `Grove.tsx` | Grove 集成的主 UI 组件 |

---

## 11. hooks/ (components/hooks/)

这里是 **组件级** 的 hooks 子目录，不是顶层的 `src/hooks/`。它们属于 hooks 命令设置 UI。

### 文件

| 文件 | 作用 |
|------|------|
| `HooksConfigMenu.tsx` | Claude hooks（pre/post tool hooks）的配置菜单 |
| `PromptDialog.tsx` | 输入 hook prompt/command 的对话框 |
| `SelectEventMode.tsx` | 选择 hook 事件类型（PreToolUse、PostToolUse 等） |
| `SelectHookMode.tsx` | 选择 hook 执行模式（allow、block、prompt） |
| `SelectMatcherMode.tsx` | 选择匹配 tools 的方式（all、specific、pattern） |
| `ViewHookMode.tsx` | 只读查看已有 hook 配置 |

---

## 12. HelpV2/

通过 `/help` 打开的第二代帮助 UI。

### 文件

| 文件 | 作用 |
|------|------|
| `Commands.tsx` | 渲染命令参考标签页 |
| `General.tsx` | 渲染通用帮助/提示标签页 |
| `HelpV2.tsx` | 带 Tabs 的顶层帮助屏幕（General、Commands） |

---

## 13. TrustDialog/

用于文件和工作区目录的信任确认对话框。

### 文件

| 文件 | 作用 |
|------|------|
| `TrustDialog.tsx` | 询问用户是否信任某个目录或文件的对话框 |
| `utils.ts` | 信任决策辅助函数 |

---

## 14. ManagedSettingsSecurityDialog/

受管/策略设置的安全警告。

### 文件

| 文件 | 作用 |
|------|------|
| `ManagedSettingsSecurityDialog.tsx` | 当策略设置覆盖用户偏好时的警告对话框 |
| `utils.ts` | 检测受管设置冲突的工具函数 |

---

## 15. ClaudeCodeHint/

插件/命令提示菜单。

### 文件

| 文件 | 作用 |
|------|------|
| `PluginHintMenu.tsx` | 以菜单形式显示插件提供的提示 |

---

## 16. HighlightedCode/

语法高亮代码渲染。

### 文件

| 文件 | 作用 |
|------|------|
| `Fallback.tsx` | 非高亮代码块的回退实现 |

主 `HighlightedCode.tsx` 位于父级 `components/` 目录，而不在这个子目录中；这里仅提供 fallback。

---

## 17. LogoV2/

动画 logo / 欢迎页 / feed 系统。

### 文件

| 文件 | 作用 |
|------|------|
| `AnimatedAsterisk.tsx` | 旋转星号 logo 动画 |
| `AnimatedClawd.tsx` | 动画版 “Clawd” 吉祥物 |
| `ChannelsNotice.tsx` | 可用 channels 的提示 |
| `Clawd.tsx` | 静态 Clawd 吉祥物 |
| `CondensedLogo.tsx` | 适合空间有限场景的紧凑 logo |
| `EmergencyTip.tsx` | 紧急提示覆盖层 |
| `Feed.tsx` | 可滚动的公告/提示 feed |
| `FeedColumn.tsx` | feed 项目的列布局 |
| `GuestPassesUpsell.tsx` | guest passes 功能的加购引导 |
| `LogoV2.tsx` | 主 logo 组件（动画星号 + 欢迎语） |
| `Opus1mMergeNotice.tsx` | 关于 Opus 1M 模型合并的提示 |
| `OverageCreditUpsell.tsx` | 购买超额 credit 的引导 |
| `VoiceModeNotice.tsx` | 语音模式可用性的提示 |
| `WelcomeV2.tsx` | 带 logo 和 feed 的完整欢迎页 |
| `feedConfigs.tsx` | feed 内容的配置数据 |

---

## 18. DesktopUpsell/

启动期间的桌面应用引导页。

### 文件

| 文件 | 作用 |
|------|------|
| `DesktopUpsellStartup.tsx` | 启动时展示的全屏桌面应用引导页 |

---

## 19. FeedbackSurvey/

应用内反馈和调查系统。

### 文件

| 文件 | 作用 |
|------|------|
| `FeedbackSurvey.tsx` | 主调查容器 |
| `FeedbackSurveyView.tsx` | 渲染单个调查问题 |
| `TranscriptSharePrompt.tsx` | 询问用户是否共享对话 transcript |
| `submitTranscriptShare.ts` | 提交 transcript share 的 API 调用 |
| `useDebouncedDigitInput.ts` | 数字评分输入的 hook（防抖） |
| `useFeedbackSurvey.tsx` | 管理调查显示生命周期的主 hook |
| `useMemorySurvey.tsx` | 处理 memory 专用调查提示的 hook |
| `usePostCompactSurvey.tsx` | 处理 compact 后调查的 hook |
| `useSurveyState.tsx` | 调查状态管理核心 hook |

---

## 20. LspRecommendation/

LSP / IDE 集成推荐 UI。

### 文件

| 文件 | 作用 |
|------|------|
| `LspRecommendationMenu.tsx` | 建议安装 LSP / IDE 插件的菜单 |

---

## 21. Passes/

Guest passes 系统 UI。

### 文件

| 文件 | 作用 |
|------|------|
| `Passes.tsx` | 显示并管理 Claude 的 guest passes |

---

## 22. Spinner/

动画 spinner 组件。

### 文件

| 文件 | 作用 |
|------|------|
| `FlashingChar.tsx` | 会闪烁开/关的字符 |
| `GlimmerMessage.tsx` | 完整的 shimmer 动画消息文本 |
| `ShimmerChar.tsx` | 单个带 shimmer 动画的字符 |
| `SpinnerAnimationRow.tsx` | spinner 动画的单行 |
| `SpinnerGlyph.tsx` | 实际的动画 spinner glyph |
| `TeammateSpinnerLine.tsx` | 用于 teammate/worker 状态的 spinner 行 |
| `TeammateSpinnerTree.tsx` | 所有 teammates 的 spinner 树 |
| `index.ts` | 重新导出 `Spinner` 作为主导出 |
| `teammateSelectHint.ts` | 返回 teammate 选择的键盘提示 |
| `useShimmerAnimation.ts` | 生成 shimmer 动画索引的 hook |
| `useStalledAnimation.ts` | 检测动画是否卡住的 hook |
| `utils.ts` | spinner 工具函数 |

**SpinnerGlyph：** 使用 Unicode braille 或其他字符的终端动画 spinner，由时钟 tick 间隔驱动。

**useShimmerAnimation：** 返回 `[ref, glimmerIndex]`。`glimmerIndex` 表示 shimmer 动画横扫文本时的前沿位置。

---

## 23. PromptInput/

REPL 屏幕底部的主提示输入区域。

### 文件

| 文件 | 作用 |
|------|------|
| `HistorySearchInput.tsx` | Ctrl+R 历史搜索输入层 |
| `IssueFlagBanner.tsx` | 标记问题的横幅 |
| `Notifications.tsx` | 输入框内通知气泡 |
| `PromptInput.tsx` | 顶层 prompt 输入协调器 |
| `PromptInputFooter.tsx` | 页脚区域（模型名、token 数等） |
| `PromptInputFooterLeftSide.tsx` | 页脚左侧（模型指示器） |
| `PromptInputFooterSuggestions.tsx` | 文件/命令建议下拉框 |
| `PromptInputHelpMenu.tsx` | 快速帮助菜单（按 ? 显示） |
| `PromptInputModeIndicator.tsx` | 显示当前输入模式（normal、plan 等） |
| `PromptInputQueuedCommands.tsx` | 显示等待运行的排队命令 |
| `PromptInputStashNotice.tsx` | 显示已暂存输入的提示 |
| `SandboxPromptFooterHint.tsx` | 页脚中的 sandbox 模式提示 |
| `ShimmeredInput.tsx` | 带 shimmer 加载状态的文本输入 |
| `VoiceIndicator.tsx` | 语音模式活动指示器 |
| `inputModes.ts` | 输入模式类型定义与转换 |
| `inputPaste.ts` | 粘贴处理逻辑 |
| `useMaybeTruncateInput.ts` | 在显示时截断长输入的 hook |
| `usePromptInputPlaceholder.ts` | 生成占位文本的 hook |
| `useShowFastIconHint.ts` | 控制 fast-mode 图标提示可见性的 hook |
| `useSwarmBanner.ts` | 用于 prompt 区域的 swarm 状态横幅 hook |
| `utils.ts` | 输入工具函数 |

---

## 24. CustomSelect/

可访问的终端 select / dropdown 组件。

### 文件

| 文件 | 作用 |
|------|------|
| `SelectMulti.tsx` | 多选下拉框 |
| `index.ts` | 重新导出 |
| `option-map.ts` | 选项数据结构和工具函数 |
| `select-input-option.tsx` | 输入模式下的选项渲染 |
| `select-option.tsx` | 单个选项渲染 |
| `select.tsx` | 主单选组件 |
| `use-multi-select-state.ts` | 多选状态 hook |
| `use-select-input.ts` | select 输入处理 hook |
| `use-select-navigation.ts` | 键盘导航 hook |
| `use-select-state.ts` | 单选状态 hook |

**Select Props（核心）：**
```typescript
type SelectProps<T extends string = string> = {
  options: Array<{ value: T; label: string; description?: string; disabled?: boolean }>;
  defaultValue?: T;
  onChange: (value: T) => void;
  onCancel?: () => void;
  isDisabled?: boolean;
}
```

**SelectMulti Props：**
```typescript
type SelectMultiProps<T extends string = string> = {
  options: Array<{ value: T; label: string }>;
  defaultValues?: T[];
  onChange: (values: T[]) => void;
  onCancel?: () => void;
}
```

---

## 25. Settings/

`/config` 对应的设置斜杠命令屏幕。

### 文件

| 文件 | 作用 |
|------|------|
| `Config.tsx` | 配置设置标签页（API key、model 等） |
| `Settings.tsx` | 带标签页的顶层设置屏幕 |
| `Status.tsx` | 显示连接和认证状态的状态标签页 |
| `Usage.tsx` | 使用统计标签页 |

---

## 26. sandbox/

Sandbox 配置 UI（由 `/sandbox` 打开）。

### 文件

| 文件 | 作用 |
|------|------|
| `SandboxConfigTab.tsx` | sandbox 启用/禁用和模式选择 |
| `SandboxDependenciesTab.tsx` | 显示 sandbox 内依赖 |
| `SandboxDoctorSection.tsx` | sandbox 健康检查的诊断区域 |
| `SandboxOverridesTab.tsx` | 按项目的 sandbox 覆盖项 |
| `SandboxSettings.tsx` | 带标签页的顶层 sandbox 设置屏幕 |

---

## 27. shell/

shell 输出展示组件。

### 文件

| 文件 | 作用 |
|------|------|
| `ExpandShellOutputContext.tsx` | 控制 shell 输出展开状态的上下文提供者 |
| `OutputLine.tsx` | 支持 ANSI 的 shell 输出单行 |
| `ShellProgressMessage.tsx` | 进行中的 shell 命令展示，带实时输出 |
| `ShellTimeDisplay.tsx` | 显示 shell 命令的已用/总耗时 |

---

## 28. skills/

skills（来自插件的斜杠命令）UI。

### 文件

| 文件 | 作用 |
|------|------|
| `SkillsMenu.tsx` | 列出可用 skills 并支持搜索的菜单 |

---

## 29. ui/

不属于 design-system 的通用 UI 原语。

### 文件

| 文件 | 作用 |
|------|------|
| `OrderedList.tsx` | 带编号的列表容器 |
| `OrderedListItem.tsx` | 有序列表中的单项 |
| `TreeSelect.tsx` | 树状/层级选择组件 |

**TreeSelect** 支持可展开/折叠的嵌套选项树，用于层级化的 tool 或目录选择。

---

## 贯穿全局的模式

### React Compiler Memoization
所有组件都使用 `import { c as _c } from "react/compiler-runtime"` 以及 `$` cache array 模式，让 React Compiler 自动做 memoization。这里的源码 TypeScript 仍是标准 React 写法，这只是构建时转换。

### Theme Integration
- 所有颜色 prop 都接受 `keyof Theme`（语义 token，如 `'permission'`、`'suggestion'`、`'success'`、`'error'`、`'warning'`、`'inactive'`）或原始 CSS 颜色字符串。
- `ThemedBox` 和 `ThemedText` 会在渲染时通过 `useTheme()` 把 theme token 解析成原始颜色。

### Keybinding System
- 组件通过 `useKeybinding(action, handler, { context, isActive })` 使用键位绑定。
- 上下文包括：`'Confirmation'`、`'Settings'`、`'Chat'`、`'Application'`。
- 标准动作包括：`'confirm:no'`（Esc/N）、`'app:exit'`（Ctrl+C）、`'app:interrupt'`（Ctrl+D）。

### Permission Flow
1. 工具使用触发权限检查 → `PermissionResult`，行为为 `'ask'` / `'allow'` / `'deny'` / `'passthrough'`。
2. 当行为为 `'ask'` 时 → 创建 `ToolUseConfirm` 对象 → 渲染 `PermissionRequest` 组件。
3. 用户选择选项 → 调用 `onAllow(updatedInput, permissionUpdates)` 或 `onReject(feedback)`。
4. 权限更新会写入 settings，供后续自动批准使用。

### Wizard Pattern
1. `WizardProvider` 持有 step 数组和累计数据。
2. 每一步调用 `useWizard()` 获取导航函数。
3. 各 step 调用 `updateWizardData(partial)` 然后 `goNext()` / `goToStep(n)`。
4. 最后一步直接调用外部 `onComplete(message)`，不是通过 wizard 自己的 `onComplete`。

### Agent File Format
```markdown
---
name: agent-identifier
description: "When to use this agent..."
tools: BashTool, FileEditTool
model: sonnet
effort: 3
color: blue
memory: project
---

System prompt content here...
```
