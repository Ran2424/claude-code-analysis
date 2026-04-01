# Claude Code — 常量、类型与配置

本文完整整理了 Claude Code CLI 代码库中 `constants/` 目录、关键根目录文件（`Tool.ts`、`Task.ts`）以及 `types/` 目录里定义的每一个常量、类型、接口和配置值。

---

## 目录

1. [API Limits (`constants/apiLimits.ts`)](#1-api-limits)
2. [Beta Headers (`constants/betas.ts`)](#2-beta-headers)
3. [Common Utilities (`constants/common.ts`)](#3-common-utilities)
4. [Cyber Risk Instruction (`constants/cyberRiskInstruction.ts`)](#4-cyber-risk-instruction)
5. [Error IDs (`constants/errorIds.ts`)](#5-error-ids)
6. [UI Figures / Glyphs (`constants/figures.ts`)](#6-ui-figures--glyphs)
7. [File Constants (`constants/files.ts`)](#7-file-constants)
8. [GitHub App (`constants/github-app.ts`)](#8-github-app)
9. [Keys / Analytics (`constants/keys.ts`)](#9-keys--analytics)
10. [Messages (`constants/messages.ts`)](#10-messages)
11. [OAuth Configuration (`constants/oauth.ts`)](#11-oauth-configuration)
12. [Output Styles (`constants/outputStyles.ts`)](#12-output-styles)
13. [Product URLs (`constants/product.ts`)](#13-product-urls)
14. [System Prompts (`constants/prompts.ts`)](#14-system-prompts)
15. [Spinner Verbs (`constants/spinnerVerbs.ts`)](#15-spinner-verbs)
16. [System Constants (`constants/system.ts`)](#16-system-constants)
17. [System Prompt Sections (`constants/systemPromptSections.ts`)](#17-system-prompt-sections)
18. [Tool Limits (`constants/toolLimits.ts`)](#18-tool-limits)
19. [Tool Sets (`constants/tools.ts`)](#19-tool-sets)
20. [Turn Completion Verbs (`constants/turnCompletionVerbs.ts`)](#20-turn-completion-verbs)
21. [XML Tags (`constants/xml.ts`)](#21-xml-tags)
22. [Tool Type System (`Tool.ts`)](#22-tool-type-system)
23. [Task Type System (`Task.ts`)](#23-task-type-system)
24. [Types Directory](#24-types-directory)
    - [IDs (`types/ids.ts`)](#241-ids-typesidsts)
    - [Permissions (`types/permissions.ts`)](#242-permissions-typespermissionsts)
    - [Commands (`types/command.ts`)](#243-commands-typescommandts)
    - [Hooks (`types/hooks.ts`)](#244-hooks-typeshooksts)
    - [Plugins (`types/plugin.ts`)](#245-plugins-typespluginsts)
    - [Logs (`types/logs.ts`)](#246-logs-typeslogsts)
    - [Text Input Types (`types/textInputTypes.ts`)](#247-text-input-types-typestextinputtypests)

---

## 1. API 限制

**文件：** `constants/apiLimits.ts`
**用途：** 由 Claude API 执行的服务端限制。保持无依赖，避免循环导入。最近核验时间：2025-12-22。

### 图片限制

| 常量 | 值 | 说明 |
|---|---|---|
| `API_IMAGE_MAX_BASE64_SIZE` | `5 * 1024 * 1024` = **5,242,880 bytes** (5 MB) | 允许的图片 base64 编码最大长度。API 会拒绝 base64 字符串超出该值的图片。这里限制的是 base64 长度，不是原始字节数（base64 会让体积增加约 33%）。 |
| `IMAGE_TARGET_RAW_SIZE` | `(API_IMAGE_MAX_BASE64_SIZE * 3) / 4` = **3,932,160 bytes** (3.75 MB) | 为了卡在 base64 上限之下而设定的原始图片目标大小。推导公式：`raw_size = base64_size * 3/4`。 |
| `IMAGE_MAX_WIDTH` | `2000` (pixels) | 图片缩放时的客户端最大宽度。API 在服务端会把超过 1568px 的图片再缩小，但客户端在 2000px 处先缩一次，以尽量保留质量。 |
| `IMAGE_MAX_HEIGHT` | `2000` (pixels) | 图片缩放时的客户端最大高度。与 `IMAGE_MAX_WIDTH` 的考虑相同。 |

### PDF 限制

| 常量 | 值 | 说明 |
|---|---|---|
| `PDF_TARGET_RAW_SIZE` | `20 * 1024 * 1024` = **20,971,520 bytes** (20 MB) | 编码前允许的 PDF 原始大小上限。API 的总请求限制是 32 MB；20 MB 原始数据转成 base64 后大约 27 MB，能给上下文留出空间。 |
| `API_PDF_MAX_PAGES` | `100` | API 接受的 PDF 最大页数。 |
| `PDF_EXTRACT_SIZE_THRESHOLD` | `3 * 1024 * 1024` = **3,145,728 bytes** (3 MB) | 超过这个大小的 PDF 会提取成页面图片，而不是作为 base64 文档块发送。这里只适用于第一方 API；非第一方始终走提取流程。 |
| `PDF_MAX_EXTRACT_SIZE` | `100 * 1024 * 1024` = **104,857,600 bytes** (100 MB) | 页面提取路径允许的最大 PDF 文件大小。超过这个值会被拒绝。 |
| `PDF_MAX_PAGES_PER_READ` | `20` | Read 工具在一次调用中可提取的最大页数，适用于传入 `pages` 参数时。 |
| `PDF_AT_MENTION_INLINE_THRESHOLD` | `10` | 页数超过这个值的 PDF 在 `@` 提及时会改走引用处理，而不是直接内联到上下文中。 |

### 媒体限制

| 常量 | 值 | 说明 |
|---|---|---|
| `API_MAX_MEDIA_PER_REQUEST` | `100` | 单次 API 请求允许的媒体项（图片 + PDF）最大数量。客户端会先做校验，避免直接撞上 API 那条不够直观的报错。 |

---

## 2. Beta 头

**文件：** `constants/betas.ts`
**用途：** 用在请求头里的 API beta 功能字符串，用于开启实验性 API 功能。

### 导出的 Beta 头常量

| 常量 | 值 | 说明 |
|---|---|---|
| `CLAUDE_CODE_20250219_BETA_HEADER` | `'claude-code-20250219'` | Core Claude Code beta header. |
| `INTERLEAVED_THINKING_BETA_HEADER` | `'interleaved-thinking-2025-05-14'` | Enables interleaved thinking (extended reasoning within tool use). |
| `CONTEXT_1M_BETA_HEADER` | `'context-1m-2025-08-07'` | Enables 1M token context window. |
| `CONTEXT_MANAGEMENT_BETA_HEADER` | `'context-management-2025-06-27'` | Context management features. |
| `STRUCTURED_OUTPUTS_BETA_HEADER` | `'structured-outputs-2025-12-15'` | Structured output response format. |
| `WEB_SEARCH_BETA_HEADER` | `'web-search-2025-03-05'` | Web search tool. |
| `TOOL_SEARCH_BETA_HEADER_1P` | `'advanced-tool-use-2025-11-20'` | Tool search for Claude API / Foundry. |
| `TOOL_SEARCH_BETA_HEADER_3P` | `'tool-search-tool-2025-10-19'` | Tool search for Vertex AI / Bedrock. |
| `EFFORT_BETA_HEADER` | `'effort-2025-11-24'` | Effort control (thinking budget). |
| `TASK_BUDGETS_BETA_HEADER` | `'task-budgets-2026-03-13'` | Task-level token budgets. |
| `PROMPT_CACHING_SCOPE_BETA_HEADER` | `'prompt-caching-scope-2026-01-05'` | Scoped prompt caching (global vs per-session). |
| `FAST_MODE_BETA_HEADER` | `'fast-mode-2026-02-01'` | Fast mode for faster output at same model quality. |
| `REDACT_THINKING_BETA_HEADER` | `'redact-thinking-2026-02-12'` | Redact thinking blocks from API response. |
| `TOKEN_EFFICIENT_TOOLS_BETA_HEADER` | `'token-efficient-tools-2026-03-28'` | Token-efficient tool schema encoding. |
| `SUMMARIZE_CONNECTOR_TEXT_BETA_HEADER` | `'summarize-connector-text-2026-03-13'` if `feature('CONNECTOR_TEXT')`, else `''` | Connector text summarization (feature-gated). |
| `AFK_MODE_BETA_HEADER` | `'afk-mode-2026-01-31'` if `feature('TRANSCRIPT_CLASSIFIER')`, else `''` | AFK (autonomous) mode (feature-gated). |
| `CLI_INTERNAL_BETA_HEADER` | `'cli-internal-2026-02-09'` if `USER_TYPE === 'ant'`, else `''` | Internal-only features. |
| `ADVISOR_BETA_HEADER` | `'advisor-tool-2026-03-01'` | Advisor tool. |

### 按提供方划分的 Beta 头集合

**`BEDROCK_EXTRA_PARAMS_HEADERS`** — `Set<string>`
Bedrock 只支持有限数量的 beta 头，而且只能通过 `extraBodyParams` 传入。这个集合保存了应该放进 Bedrock `extraBodyParams`、而不是直接放进请求头的 beta 字符串。

成员：
- `INTERLEAVED_THINKING_BETA_HEADER` (`'interleaved-thinking-2025-05-14'`)
- `CONTEXT_1M_BETA_HEADER` (`'context-1m-2025-08-07'`)
- `TOOL_SEARCH_BETA_HEADER_3P` (`'tool-search-tool-2025-10-19'`)

**`VERTEX_COUNT_TOKENS_ALLOWED_BETAS`** — `Set<string>`
Vertex `countTokens` API 允许使用的 beta 集合。其他 beta 会触发 400 错误。

成员：
- `CLAUDE_CODE_20250219_BETA_HEADER` (`'claude-code-20250219'`)
- `INTERLEAVED_THINKING_BETA_HEADER` (`'interleaved-thinking-2025-05-14'`)
- `CONTEXT_MANAGEMENT_BETA_HEADER` (`'context-management-2025-06-27'`)

---

## 3. 通用工具

**文件：** `constants/common.ts`

### 函数

**`getLocalISODate(): string`**
以本地时区返回当前日期的 ISO 格式（`YYYY-MM-DD`）。会遵守 `CLAUDE_CODE_OVERRIDE_DATE` 环境变量，供 ant 内部做日期覆盖（测试/调试）。

**`getSessionStartDate`** — memoized version of `getLocalISODate`
在会话开始时只捕获一次日期，以保证提示词缓存稳定。用于简单模式（`--bare`），避免在午夜把缓存前缀打散。它以记忆化常量导出，因此后续调用都会返回同一个值。

**`getLocalMonthYear(): string`**
以用户本地时区返回 `"Month YYYY"`（例如 `"February 2026"`）。它按月变化，不按天变化，用在工具提示词里可以尽量减少缓存失效。会遵守 `CLAUDE_CODE_OVERRIDE_DATE`。

---

## 4. 网络安全风险指令

**文件：** `constants/cyberRiskInstruction.ts`
**归属：** Safeguards 团队（David Forsythe、Kyla Guru）。**未经 Safeguards 团队审查，不要修改。**

### `CYBER_RISK_INSTRUCTION` — `string`

```
IMPORTANT: Assist with authorized security testing, defensive security, CTF challenges,
and educational contexts. Refuse requests for destructive techniques, DoS attacks, mass
targeting, supply chain compromise, or detection evasion for malicious purposes.
Dual-use security tools (C2 frameworks, credential testing, exploit development) require
clear authorization context: pentesting engagements, CTF competitions, security research,
or defensive use cases.
```

这条指令会直接注入系统提示词（既包括简单的导言部分，也包括主动/自主模式部分）。它划定了可接受的防御性安全协助与潜在有害行为之间的边界。

---

## 5. 错误 ID

**文件：** `constants/errorIds.ts`
**用途：** 用于在生产环境追踪错误来源的混淆数字标识符。按独立 `const` 导出，方便外部构建做最优的死代码消除。

**下一个 ID（以该文件为准）：** 346

| 常量 | 值 | 说明 |
|---|---|---|
| `E_TOOL_USE_SUMMARY_GENERATION_FAILED` | `344` | 标识工具使用摘要生成失败的错误。 |

---

## 6. UI 图形 / 字形

**文件：** `constants/figures.ts`

### 状态指示器

| 常量 | 值 | Unicode | 说明 |
|---|---|---|---|
| `BLACK_CIRCLE` | `'⏺'` (macOS) or `'●'` (others) | U+23FA / U+25CF | 平台自适应的旋转器/状态圆点。 |
| `BULLET_OPERATOR` | `'∙'` | U+2219 | 小圆点符号。 |
| `TEARDROP_ASTERISK` | `'✻'` | U+273B | 装饰性星号。 |
| `UP_ARROW` | `'\u2191'` (`↑`) | U+2191 | 用于 opus 1m 合并提示。 |
| `DOWN_ARROW` | `'\u2193'` (`↓`) | U+2193 | 用于滚动提示。 |
| `LIGHTNING_BOLT` | `'↯'` (`\u21af`) | U+21AF | 快速模式指示符。 |

### 努力级别指示器

| 常量 | 值 | Unicode | 说明 |
|---|---|---|---|
| `EFFORT_LOW` | `'○'` | U+25CB | 努力等级：低。 |
| `EFFORT_MEDIUM` | `'◐'` | U+25D0 | 努力等级：中。 |
| `EFFORT_HIGH` | `'●'` | U+25CF | 努力等级：高。 |
| `EFFORT_MAX` | `'◉'` | U+25C9 | 努力等级：最高（仅 Opus 4.6）。 |

### 媒体 / 触发状态

| 常量 | 值 | Unicode | 说明 |
|---|---|---|---|
| `PLAY_ICON` | `'\u25b6'` (`▶`) | U+25B6 | 播放状态指示符。 |
| `PAUSE_ICON` | `'\u23f8'` (`⏸`) | U+23F8 | 暂停状态指示符。 |

### MCP 订阅指示器

| 常量 | 值 | Unicode | 说明 |
|---|---|---|---|
| `REFRESH_ARROW` | `'\u21bb'` (`↻`) | U+21BB | 资源更新指示符。 |
| `CHANNEL_ARROW` | `'\u2190'` (`←`) | U+2190 | 传入 channel 消息指示符。 |
| `INJECTED_ARROW` | `'\u2192'` (`→`) | U+2192 | 跨会话注入消息指示符。 |
| `FORK_GLYPH` | `'\u2442'` (`⑂`) | U+2442 | fork 指令指示符。 |

### 审查状态指示器（Ultrareview）

| 常量 | 值 | Unicode | 说明 |
|---|---|---|---|
| `DIAMOND_OPEN` | `'\u25c7'` (`◇`) | U+25C7 | 运行中状态。 |
| `DIAMOND_FILLED` | `'\u25c6'` (`◆`) | U+25C6 | 已完成/失败状态。 |
| `REFERENCE_MARK` | `'\u203b'` (`※`) | U+203B | 米字标记，用于离线摘要回顾。 |

### 其他指示器

| 常量 | 值 | Unicode | 说明 |
|---|---|---|---|
| `FLAG_ICON` | `'\u2691'` (`⚑`) | U+2691 | Issue 标记横幅。 |
| `BLOCKQUOTE_BAR` | `'\u258e'` (`▎`) | U+258E | 左侧四分之一块，用作引用行前缀。 |
| `HEAVY_HORIZONTAL` | `'\u2501'` (`━`) | U+2501 | 加粗的方框横线。 |

### 桥接状态指示器

| 常量 | 值 | 说明 |
|---|---|---|
| `BRIDGE_SPINNER_FRAMES` | `['·\|·', '·/·', '·—·', '·\\·']` | bridge 旋转器的动画帧。 |
| `BRIDGE_READY_INDICATOR` | `'·✔︎·'` | bridge 就绪时显示。 |
| `BRIDGE_FAILED_INDICATOR` | `'×'` | bridge 失败时显示。 |

---

## 7. 文件常量

**文件：** `constants/files.ts`

### `BINARY_EXTENSIONS` — `Set<string>`

被视为二进制的文件扩展名集合。用于跳过这些文件上的文本类操作。共包含 113 个扩展名，分布如下：

**图片：** `.png`, `.jpg`, `.jpeg`, `.gif`, `.bmp`, `.ico`, `.webp`, `.tiff`, `.tif`

**视频：** `.mp4`, `.mov`, `.avi`, `.mkv`, `.webm`, `.wmv`, `.flv`, `.m4v`, `.mpeg`, `.mpg`

**音频：** `.mp3`, `.wav`, `.ogg`, `.flac`, `.aac`, `.m4a`, `.wma`, `.aiff`, `.opus`

**压缩包：** `.zip`, `.tar`, `.gz`, `.bz2`, `.7z`, `.rar`, `.xz`, `.z`, `.tgz`, `.iso`

**可执行文件/二进制：** `.exe`, `.dll`, `.so`, `.dylib`, `.bin`, `.o`, `.a`, `.obj`, `.lib`, `.app`, `.msi`, `.deb`, `.rpm`

**文档：** `.pdf` *(在 FileReadTool 调用处排除)*, `.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx`, `.odt`, `.ods`, `.odp`

**字体：** `.ttf`, `.otf`, `.woff`, `.woff2`, `.eot`

**字节码/虚拟机产物：** `.pyc`, `.pyo`, `.class`, `.jar`, `.war`, `.ear`, `.node`, `.wasm`, `.rlib`

**数据库文件：** `.sqlite`, `.sqlite3`, `.db`, `.mdb`, `.idx`

**设计/3D：** `.psd`, `.ai`, `.eps`, `.sketch`, `.fig`, `.xd`, `.blend`, `.3ds`, `.max`

**Flash：** `.swf`, `.fla`

**锁文件/性能分析：** `.lockb`, `.dat`, `.data`

### 函数

**`hasBinaryExtension(filePath: string): boolean`**
通过查找路径的扩展名（转小写）是否存在于 `BINARY_EXTENSIONS`，来判断它是不是二进制扩展名。

**`isBinaryContent(buffer: Buffer): boolean`**
通过检查最多 8192 字节（`BINARY_CHECK_SIZE`）来检测二进制内容：
- 一旦发现空字节（`0x00`），立即返回 `true`。
- 如果检查到的字节中有超过 10% 不是可打印字符，也不是空白字符（不含制表符 `0x09`、换行 `0x0A`、回车 `0x0D`），返回 `true`。

---

## 8. GitHub App

**文件：** `constants/github-app.ts`
**用途：** GitHub Actions 工作流集成所需的模板和元数据。

### 字符串常量

| 常量 | 值 | 说明 |
|---|---|---|
| `PR_TITLE` | `'Add Claude Code GitHub Workflow'` | 安装 GitHub app 工作流时创建的 PR 标题。 |
| `GITHUB_ACTION_SETUP_DOCS_URL` | `'https://github.com/anthropics/claude-code-action/blob/main/docs/setup.md'` | 安装文档链接。 |

### 模板常量

**`WORKFLOW_CONTENT`** — `string`
一个完整的 GitHub Actions YAML 工作流文件（`name: Claude Code`），内容包括：
- 触发条件：`issue_comment`（created）、`pull_request_review_comment`（created）、`issues`（opened、assigned）、`pull_request_review`（submitted）。
- 条件：只有在相关事件正文里提到 `@claude` 时才运行。
- 运行环境：`ubuntu-latest`。
- 所需权限：`contents: read`、`pull-requests: read`、`issues: read`、`id-token: write`、`actions: read`。
- 步骤：先执行 `actions/checkout@v4`（depth 1），再执行 `anthropics/claude-code-action@v1`，并传入 `anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}`。

**`PR_BODY`** — `string`
安装 PR 的 Markdown 正文。说明 Claude Code 是什么、工作流如何运行、安全注意事项（API key 作为密钥、写权限限制、Actions 运行历史），以及如何配置额外允许的工具。

**`CODE_REVIEW_PLUGIN_WORKFLOW_CONTENT`** — `string`
用于自动代码审查的 GitHub Actions YAML 工作流（`name: Claude Code Review`）：
- 触发条件：`pull_request` 事件（opened、synchronize、ready_for_review、reopened）。
- 使用 `anthropics/claude-code-action@v1`，并从 `https://github.com/anthropics/claude-code.git` marketplace 加载 `code-review@claude-code-plugins` 插件。
- 在 PR 上运行 `/code-review:code-review` 命令。

---

## 9. 密钥 / 分析

**文件：** `constants/keys.ts`
**用途：** GrowthBook 功能开关的客户端 key。

### `getGrowthBookClientKey(): string`

根据环境返回三种 GrowthBook SDK client key 之一：

| 条件 | 键 |
|---|---|
| `USER_TYPE === 'ant'` AND `ENABLE_GROWTHBOOK_DEV` is truthy | `'sdk-yZQvlplybuXjYh6L'` (dev/internal) |
| `USER_TYPE === 'ant'` (without dev flag) | `'sdk-xRVcrliHIlrg4og4'` (ant prod) |
| All other users (external) | `'sdk-zAZezfDKGoZuXXKe'` (external prod) |

它被实现为一个惰性函数，而不是常量，这样 `globalSettings.env` 里在模块加载后才应用的 `ENABLE_GROWTHBOOK_DEV` 也能在调用时生效。

---

## 10. 消息

**文件：** `constants/messages.ts`
**用途：** 面向用户展示的字符串常量。

| 常量 | 值 | 说明 |
|---|---|---|
| `NO_CONTENT_MESSAGE` | `'(no content)'` | 当工具结果或消息没有可显示内容时展示。 |

---

## 11. OAuth 配置

**文件：** `constants/oauth.ts`

### 作用域常量

| 常量 | 值 | 说明 |
|---|---|---|
| `CLAUDE_AI_INFERENCE_SCOPE` | `'user:inference'` | Claude.ai 推理访问范围。 |
| `CLAUDE_AI_PROFILE_SCOPE` | `'user:profile'` | 用户资料访问范围。 |
| `OAUTH_BETA_HEADER` | `'oauth-2025-04-20'` | OAuth beta 功能头。 |

### 作用域数组

**`CONSOLE_OAUTH_SCOPES`** — `readonly ['org:create_api_key', 'user:profile']`
Console 的 OAuth 范围（通过 Console 创建 API key）。

**`CLAUDE_AI_OAUTH_SCOPES`** — `readonly ['user:profile', 'user:inference', 'user:sessions:claude_code', 'user:mcp_servers', 'user:file_upload']`
Claude.ai 订阅者的 OAuth 范围（Pro/Max/Team/Enterprise）。

**`ALL_OAUTH_SCOPES`** — `string[]`
两个数组中所有范围的并集（已去重）。登录时会一次性请求所有范围，以处理 Console → Claude.ai 的跳转流程。必须与 apps 仓库里的 `OAuthConsentPage` 保持同步。

### `OauthConfig` 类型

```typescript
type OauthConfig = {
  BASE_API_URL: string
  CONSOLE_AUTHORIZE_URL: string
  CLAUDE_AI_AUTHORIZE_URL: string
  CLAUDE_AI_ORIGIN: string        // Separate from AUTHORIZE_URL for web page links
  TOKEN_URL: string
  API_KEY_URL: string
  ROLES_URL: string
  CONSOLE_SUCCESS_URL: string
  CLAUDEAI_SUCCESS_URL: string
  MANUAL_REDIRECT_URL: string
  CLIENT_ID: string
  OAUTH_FILE_SUFFIX: string
  MCP_PROXY_URL: string
  MCP_PROXY_PATH: string
}
```

### 生产环境 OAuth 配置 (`PROD_OAUTH_CONFIG`)

| 字段 | 值 |
|---|---|
| `BASE_API_URL` | `'https://api.anthropic.com'` |
| `CONSOLE_AUTHORIZE_URL` | `'https://platform.claude.com/oauth/authorize'` |
| `CLAUDE_AI_AUTHORIZE_URL` | `'https://claude.com/cai/oauth/authorize'`（通过 claude.com/cai/* 做归因，随后 307 跳转到 claude.ai） |
| `CLAUDE_AI_ORIGIN` | `'https://claude.ai'` |
| `TOKEN_URL` | `'https://platform.claude.com/v1/oauth/token'` |
| `API_KEY_URL` | `'https://api.anthropic.com/api/oauth/claude_cli/create_api_key'` |
| `ROLES_URL` | `'https://api.anthropic.com/api/oauth/claude_cli/roles'` |
| `CONSOLE_SUCCESS_URL` | `'https://platform.claude.com/buy_credits?returnUrl=/oauth/code/success%3Fapp%3Dclaude-code'` |
| `CLAUDEAI_SUCCESS_URL` | `'https://platform.claude.com/oauth/code/success?app=claude-code'` |
| `MANUAL_REDIRECT_URL` | `'https://platform.claude.com/oauth/code/callback'` |
| `CLIENT_ID` | `'9d1c250a-e61b-44d9-88ed-5944d1962f5e'` |
| `OAUTH_FILE_SUFFIX` | `''` (no suffix for production) |
| `MCP_PROXY_URL` | `'https://mcp-proxy.anthropic.com'` |
| `MCP_PROXY_PATH` | `'/v1/mcp/{server_id}'` |

### 预发环境 OAuth 配置 (`STAGING_OAUTH_CONFIG`)

仅包含在 `ant` 构建中。关键字段：

| 字段 | 值 |
|---|---|
| `BASE_API_URL` | `'https://api-staging.anthropic.com'` |
| `CONSOLE_AUTHORIZE_URL` | `'https://platform.staging.ant.dev/oauth/authorize'` |
| `CLAUDE_AI_AUTHORIZE_URL` | `'https://claude-ai.staging.ant.dev/oauth/authorize'` |
| `CLIENT_ID` | `'22422756-60c9-4084-8eb7-27705fd5cf9a'` |
| `OAUTH_FILE_SUFFIX` | `'-staging-oauth'` |
| `MCP_PROXY_URL` | `'https://mcp-proxy-staging.anthropic.com'` |

### 本地 OAuth 配置（动态）

由环境变量和默认值构建：

| Env Var | Default |
|---|---|
| `CLAUDE_LOCAL_OAUTH_API_BASE` | `'http://localhost:8000'` |
| `CLAUDE_LOCAL_OAUTH_APPS_BASE` | `'http://localhost:4000'` |
| `CLAUDE_LOCAL_OAUTH_CONSOLE_BASE` | `'http://localhost:3000'` |

Local `CLIENT_ID`: `'22422756-60c9-4084-8eb7-27705fd5cf9a'`
Local `OAUTH_FILE_SUFFIX`: `'-local-oauth'`
Local `MCP_PROXY_URL`: `'http://localhost:8205'`
Local `MCP_PROXY_PATH`: `'/v1/toolbox/shttp/mcp/{server_id}'`

### 允许的 OAuth 基础 URL（FedStart/PubSec）

```
'https://beacon.claude-ai.staging.ant.dev'
'https://claude.fedstart.com'
'https://claude-staging.fedstart.com'
```

`CLAUDE_CODE_CUSTOM_OAUTH_URL` 只允许使用这些基础 URL，以防 OAuth token 被发送到任意端点。

### MCP 客户端元数据

| 常量 | 值 |
|---|---|
| `MCP_CLIENT_METADATA_URL` | `'https://claude.ai/oauth/claude-code-client-metadata'` |

当授权服务器声明 `client_id_metadata_document_supported: true` 时，这个值会作为 MCP OAuth（CIMD/SEP-991）的 `client_id`。

### 函数

**`getOauthConfig(): OauthConfig`**
根据环境类型（`prod`/`staging`/`local`）返回对应的 OAuth 配置。如果设置了 `CLAUDE_CODE_CUSTOM_OAUTH_URL`，会先按允许列表校验后覆盖；如果设置了 `CLAUDE_CODE_OAUTH_CLIENT_ID`，也会覆盖。

**`fileSuffixForOauthConfig(): string`**
返回 OAuth 凭据存储文件的后缀：生产环境是 `''`，预发是 `'-staging-oauth'`，本地是 `'-local-oauth'`，自定义 OAuth URL 则是 `'-custom-oauth'`。

---

## 12. 输出样式

**文件：** `constants/outputStyles.ts`

### 类型

```typescript
type OutputStyleConfig = {
  name: string
  description: string
  prompt: string
  source: SettingSource | 'built-in' | 'plugin'
  keepCodingInstructions?: boolean
  forceForPlugin?: boolean   // If true, automatically applied when the plugin is enabled
}

type OutputStyles = {
  readonly [K in OutputStyle]: OutputStyleConfig | null
}
```

### 常量

| 常量 | 值 | 说明 |
|---|---|---|
| `DEFAULT_OUTPUT_STYLE_NAME` | `'default'` | 内置默认输出样式的名称（无自定义）。 |

### 内置输出样式 (`OUTPUT_STYLE_CONFIG`)

**`default`:** `null` —— 无自定义，采用标准 Claude Code 行为。

**`Explanatory`:**
- 来源：`'built-in'`
- 说明：`'Claude explains its implementation choices and codebase patterns'`
- `keepCodingInstructions: true`
- Prompt instructs Claude to provide educational insights about the codebase. Uses an `EXPLANATORY_FEATURE_PROMPT` block with a distinctive code fence format using star (`★`) borders.

**`Learning`:**
- 来源：`'built-in'`
- 说明：`'Claude pauses and asks you to write small pieces of code for hands-on practice'`
- `keepCodingInstructions: true`
- Prompt encourages hands-on practice by requesting 2–10 line code contributions for design decisions, business logic, and key algorithms. Uses a "Learn by Doing" request format with **Context**, **Your Task**, and **Guidance** fields.

### 函数

**`getAllOutputStyles(cwd: string): Promise<{ [styleName: string]: OutputStyleConfig | null }>`**
已做记忆化。按优先级合并内置、插件、用户、项目和托管样式（built-in < plugin < user < project < managed）。

**`getOutputStyleConfig(): Promise<OutputStyleConfig | null>`**
返回当前生效的输出样式配置。先检查是否有插件强制指定样式；如果多个插件都强制指定，则取第一个并记录警告。最后回退到用户设置。

**`hasCustomOutputStyle(): boolean`**
如果用户选择了非默认输出样式，则返回 `true`。

**`clearAllOutputStylesCache(): void`**
清空 `getAllOutputStyles` 的记忆化缓存。会在 `/clear` 和 `/compact` 时调用。

---

## 13. 产品 URL

**文件：** `constants/product.ts`

### URL 常量

| 常量 | 值 | 说明 |
|---|---|---|
| `PRODUCT_URL` | `'https://claude.com/claude-code'` | 主产品落地页。 |
| `CLAUDE_AI_BASE_URL` | `'https://claude.ai'` | 远程会话使用的 Claude AI 基础 URL。 |
| `CLAUDE_AI_STAGING_BASE_URL` | `'https://claude-ai.staging.ant.dev'` | 预发环境基础 URL。 |
| `CLAUDE_AI_LOCAL_BASE_URL` | `'http://localhost:4000'` | 本地开发基础 URL。 |

### 函数

**`isRemoteSessionStaging(sessionId?, ingressUrl?): boolean`**
如果 `sessionId` 包含 `'_staging_'`，或者 `ingressUrl` 包含 `'staging'`，则返回 `true`。

**`isRemoteSessionLocal(sessionId?, ingressUrl?): boolean`**
如果 `sessionId` 包含 `'_local_'`，或者 `ingressUrl` 包含 `'localhost'`，则返回 `true`。

**`getClaudeAiBaseUrl(sessionId?, ingressUrl?): string`**
按环境返回合适的基础 URL：local → staging → prod。

**`getRemoteSessionUrl(sessionId, ingressUrl?): string`**
返回用于查看远程会话的完整 URL：`${baseUrl}/code/${compatId}`。会通过 `toCompatSessionId()` 把 `cse_*` 前缀转换成 `session_*`，以兼容前端。

---

## 14. 系统提示

**文件：** `constants/prompts.ts`
**用途：** 为所有 Claude Code 会话类型生成核心系统提示词。

### URL 常量

| 常量 | 值 | 说明 |
|---|---|---|
| `CLAUDE_CODE_DOCS_MAP_URL` | `'https://code.claude.com/docs/en/claude_code_docs_map.md'` | Claude Code 文档映射 URL。 |
| `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` | `'__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'` | 分隔静态内容（跨组织可缓存）与动态内容的边界标记。系统提示数组中它之前的所有内容都可以使用 `scope: 'global'`。 |

### 模型配置（内部）

这些是提示词内部会用到、但不会导出的常量：

| 内部 | 值 | 说明 |
|---|---|---|
| `FRONTIER_MODEL_NAME` | `'Claude Opus 4.6'` | 当前前沿模型名称（更新时会标注 `@[MODEL LAUNCH]`）。 |
| `CLAUDE_4_5_OR_4_6_MODEL_IDS.opus` | `'claude-opus-4-6'` | 最新的 Opus 模型 ID。 |
| `CLAUDE_4_5_OR_4_6_MODEL_IDS.sonnet` | `'claude-sonnet-4-6'` | 最新的 Sonnet 模型 ID。 |
| `CLAUDE_4_5_OR_4_6_MODEL_IDS.haiku` | `'claude-haiku-4-5-20251001'` | 最新的 Haiku 模型 ID。 |

### 知识截止日期（按模型）

| 模型模式 | 截止日期 |
|---|---|
| `claude-sonnet-4-6` | `'August 2025'` |
| `claude-opus-4-6` | `'May 2025'` |
| `claude-opus-4-5` | `'May 2025'` |
| `claude-haiku-4*` | `'February 2025'` |
| `claude-opus-4` or `claude-sonnet-4` (generic) | `'January 2025'` |
| 其他全部 | `null`（不显示截止日期） |

### CLI 系统提示前缀值

```typescript
type CLISyspromptPrefix =
  | "You are Claude Code, Anthropic's official CLI for Claude."
  | "You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK."
  | "You are a Claude agent, built on Anthropic's Claude Agent SDK."
```

**`CLI_SYSPROMPT_PREFIXES`** — `ReadonlySet<string>`，包含这三个前缀值。`splitSysPromptPrefix` 会靠内容而不是位置来识别前缀块。

**`getCLISyspromptPrefix(options?): CLISyspromptPrefix`** — 选择逻辑：
- Vertex AI → 总是返回 `DEFAULT_PREFIX`
- 非交互且带 `appendSystemPrompt` → `AGENT_SDK_CLAUDE_CODE_PRESET_PREFIX`
- 非交互且不带 `appendSystemPrompt` → `AGENT_SDK_PREFIX`
- 交互式 → `DEFAULT_PREFIX`

### 导出的 Agent 提示词

**`DEFAULT_AGENT_PROMPT`** — `string`

```
You are an agent for Claude Code, Anthropic's official CLI for Claude. Given the user's
message, you should use the tools available to complete the task. Complete the task fully—
don't gold-plate, but don't leave it half-done. When you complete the task, respond with
a concise report covering what was done and any key findings — the caller will relay this
给用户，所以只需要保留要点。
```

作为 `AgentTool` 派生子代理的基础系统提示词。

### 系统提示区段架构

`getSystemPrompt()` 会按下面的顺序，把完整系统提示词组装成字符串数组：

**静态分段（缓存稳定）：**
1. `getSimpleIntroSection()` — Identity framing + CYBER_RISK_INSTRUCTION + URL policy
2. `getSimpleSystemSection()` — Communication rules, tool approvals, system-reminder tags, prompt injection warning, hooks, context compression
3. `getSimpleDoingTasksSection()` — Software engineering task guidance, code style rules, user help info
4. `getActionsSection()` — Action reversibility and blast radius guidance, risky action examples
5. `getUsingYourToolsSection()` — Dedicated tool preferences over Bash, parallel tool calls
6. `getSimpleToneAndStyleSection()` — No emoji, concise responses, file:line references, GitHub #issue format
7. `getOutputEfficiencySection()` — Different content for `ant` vs external users

**动态边界标记**（如果启用了全局缓存范围）：
- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`

**动态分段（由注册表管理，按分段缓存）：**
- `session_guidance` — Session-specific guidance (AskUserQuestion, shell bang, Agent tool, skills, DiscoverSkills, verification agent)
- `memory` — CLAUDE.md memory content
- `ant_model_override` — Ant-internal model overrides
- `env_info_simple` — Working directory, platform, OS, model description, knowledge cutoff
- `language` — Language preference
- `output_style` — Active output style
- `mcp_instructions` — MCP server instructions (uncached — MCP servers connect/disconnect between turns)
- `scratchpad` — Scratchpad directory instructions
- `frc` — Function result clearing instructions
- `summarize_tool_results` — Tool result summarization reminder
- `numeric_length_anchors` — (ant-only) Length limits: ≤25 words between tool calls, ≤100 words final responses
- `token_budget` — (TOKEN_BUDGET feature) Token target instructions
- `brief` — (KAIROS/KAIROS_BRIEF feature) Brief tool instructions

### 关键系统提示区段

**`getSimpleDoingTasksSection()` - 代码风格规则（ant 内部）：**
- 默认不写注释
- 只有在 WHY 不明显时才加注释
- 不要解释代码做了什么
- 不要在注释里提当前任务
- 不要删除已有注释，除非同时删掉它描述的代码

**`getActionsSection()` - 高风险操作（必须确认）：**
- 破坏性操作：删除文件/分支、删除数据库表、杀进程、`rm -rf`、覆盖未提交改动
- 难以撤销：强推、`git reset --hard`、修改已发布提交、移除包、修改 CI/CD
- 共享状态：推送代码、创建/关闭 PR/issue、发送消息（Slack、email、GitHub）、向外部服务发帖
- 上传到第三方工具（可能被缓存/索引）

**`getProactiveSection()` - 自主模式：**
- 响应 `<tick>` 提示，在轮次之间保持活跃
- 用 `SleepTool` 控制操作之间的等待时间（提示词缓存最长 5 分钟失效）
- 如果没有有用的事可做，必须调用 `SleepTool`（不要输出“still waiting”）
- 第一次唤醒：简短问候用户，询问需求
- 感知终端焦点：未聚焦 = 自主行动；聚焦 = 协作模式

**`SUMMARIZE_TOOL_RESULTS_SECTION`**（不导出）：
```
处理工具结果时，要把后续回答可能还会用到的重要信息记下来，因为原始工具结果之后可能会被清空。
```

### 关键函数

**`getSystemPrompt(tools, model, additionalWorkingDirectories?, mcpClients?): Promise<string[]>`**
系统提示词生成的主入口。当 `CLAUDE_CODE_SIMPLE=1` 时，返回 `['You are Claude Code...\nCWD: ...\nDate: ...']`。

**`computeEnvInfo(modelId, additionalWorkingDirectories?): Promise<string>`**
为子代理系统提示词生成旧格式的 `<env>` 块。

**`computeSimpleEnvInfo(modelId, additionalWorkingDirectories?): Promise<string>`**
为主系统提示词生成 `# Environment` 段（包含 worktree 警告、模型 ID 参考、Claude Code 可用性信息和 Fast Mode 说明）。

**`enhanceSystemPromptWithEnvDetails(existingSystemPrompt, model, ...): Promise<string[]>`**
用说明和环境信息增强已有系统提示词（来自 `--system-prompt`），供子代理使用。说明包括：
- 在 bash 调用之间使用绝对文件路径
- 在最终回复里分享绝对文件路径
- 避免使用 emoji
- 工具调用前不要加冒号

**`getScratchpadInstructions(): string | null`**
如果启用了 scratchpad 目录，则返回使用说明。会引导 Claude 把所有临时文件放到会话专用 scratchpad 目录，而不是 `/tmp`。

**`prependBullets(items): string[]`**
用于系统提示词格式化的工具，会给顶层条目加上 ` - `，给嵌套数组加上 `  - `。

**`getAttributionHeader(fingerprint: string): string`**（位于 `constants/system.ts`）：
生成 `x-anthropic-billing-header: cc_version=<VERSION>.<fingerprint>; cc_entrypoint=<entrypoint>;[cch=00000;][cc_workload=<workload>;]`

---

## 15. 旋转器动词

**文件：** `constants/spinnerVerbs.ts`

### `SPINNER_VERBS` — `string[]`

一个包含 186 个俏皮现在分词动词的数组，会在 Claude 工作时显示在 spinner 里。例子既有普通词（`'Computing'`、`'Processing'`、`'Working'`），也有更俏皮的词（`'Beboppin'`、`'Discombobulating'`、`'Flibbertigibbeting'`）。

完整列表：`'Accomplishing'`, `'Actioning'`, `'Actualizing'`, `'Architecting'`, `'Baking'`, `'Beaming'`, `"Beboppin'"`, `'Befuddling'`, `'Billowing'`, `'Blanching'`, `'Bloviating'`, `'Boogieing'`, `'Boondoggling'`, `'Booping'`, `'Bootstrapping'`, `'Brewing'`, `'Bunning'`, `'Burrowing'`, `'Calculating'`, `'Canoodling'`, `'Caramelizing'`, `'Cascading'`, `'Catapulting'`, `'Cerebrating'`, `'Channeling'`, `'Channelling'`, `'Choreographing'`, `'Churning'`, `'Clauding'`, `'Coalescing'`, `'Cogitating'`, `'Combobulating'`, `'Composing'`, `'Computing'`, `'Concocting'`, `'Considering'`, `'Contemplating'`, `'Cooking'`, `'Crafting'`, `'Creating'`, `'Crunching'`, `'Crystallizing'`, `'Cultivating'`, `'Deciphering'`, `'Deliberating'`, `'Determining'`, `'Dilly-dallying'`, `'Discombobulating'`, `'Doing'`, `'Doodling'`, `'Drizzling'`, `'Ebbing'`, `'Effecting'`, `'Elucidating'`, `'Embellishing'`, `'Enchanting'`, `'Envisioning'`, `'Evaporating'`, `'Fermenting'`, `'Fiddle-faddling'`, `'Finagling'`, `'Flambéing'`, `'Flibbertigibbeting'`, `'Flowing'`, `'Flummoxing'`, `'Fluttering'`, `'Forging'`, `'Forming'`, `'Frolicking'`, `'Frosting'`, `'Gallivanting'`, `'Galloping'`, `'Garnishing'`, `'Generating'`, `'Gesticulating'`, `'Germinating'`, `'Gitifying'`, `'Grooving'`, `'Gusting'`, `'Harmonizing'`, `'Hashing'`, `'Hatching'`, `'Herding'`, `'Honking'`, `'Hullaballooing'`, `'Hyperspacing'`, `'Ideating'`, `'Imagining'`, `'Improvising'`, `'Incubating'`, `'Inferring'`, `'Infusing'`, `'Ionizing'`, `'Jitterbugging'`, `'Julienning'`, `'Kneading'`, `'Leavening'`, `'Levitating'`, `'Lollygagging'`, `'Manifesting'`, `'Marinating'`, `'Meandering'`, `'Metamorphosing'`, `'Misting'`, `'Moonwalking'`, `'Moseying'`, `'Mulling'`, `'Mustering'`, `'Musing'`, `'Nebulizing'`, `'Nesting'`, `'Newspapering'`, `'Noodling'`, `'Nucleating'`, `'Orbiting'`, `'Orchestrating'`, `'Osmosing'`, `'Perambulating'`, `'Percolating'`, `'Perusing'`, `'Philosophising'`, `'Photosynthesizing'`, `'Pollinating'`, `'Pondering'`, `'Pontificating'`, `'Pouncing'`, `'Precipitating'`, `'Prestidigitating'`, `'Processing'`, `'Proofing'`, `'Propagating'`, `'Puttering'`, `'Puzzling'`, `'Quantumizing'`, `'Razzle-dazzling'`, `'Razzmatazzing'`, `'Recombobulating'`, `'Reticulating'`, `'Roosting'`, `'Ruminating'`, `'Sautéing'`, `'Scampering'`, `'Schlepping'`, `'Scurrying'`, `'Seasoning'`, `'Shenaniganing'`, `'Shimmying'`, `'Simmering'`, `'Skedaddling'`, `'Sketching'`, `'Slithering'`, `'Smooshing'`, `'Sock-hopping'`, `'Spelunking'`, `'Spinning'`, `'Sprouting'`, `'Stewing'`, `'Sublimating'`, `'Swirling'`, `'Swooping'`, `'Symbioting'`, `'Synthesizing'`, `'Tempering'`, `'Thinking'`, `'Thundering'`, `'Tinkering'`, `'Tomfoolering'`, `'Topsy-turvying'`, `'Transfiguring'`, `'Transmuting'`, `'Twisting'`, `'Undulating'`, `'Unfurling'`, `'Unravelling'`, `'Vibing'`, `'Waddling'`, `'Wandering'`, `'Warping'`, `'Whatchamacalliting'`, `'Whirlpooling'`, `'Whirring'`, `'Whisking'`, `'Wibbling'`, `'Working'`, `'Wrangling'`, `'Zesting'`, `'Zigzagging'`

### `getSpinnerVerbs(): string[]`

根据设置里的用户配置返回 spinner 动词：
- 没有配置 → 返回 `SPINNER_VERBS`
- `mode: 'replace'` → 如果 `config.verbs` 非空就返回它，否则返回 `SPINNER_VERBS`
- `mode: 'append'` → 返回 `[...SPINNER_VERBS, ...config.verbs]`

---

## 16. 系统常量

**文件：** `constants/system.ts`
**用途：** 提取出来、用于打破循环依赖的关键系统常量。

### CLI 系统提示前缀

通过 `CLI_SYSPROMPT_PREFIXES` 集合导出的三个可能前缀值：

1. `DEFAULT_PREFIX`: `"You are Claude Code, Anthropic's official CLI for Claude."` — 用于交互式会话和 Vertex AI。
2. `AGENT_SDK_CLAUDE_CODE_PRESET_PREFIX`: `"You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK."` — 用于带 `appendSystemPrompt` 的非交互模式。
3. `AGENT_SDK_PREFIX`: `"You are a Claude agent, built on Anthropic's Claude Agent SDK."` — 用于不带 `appendSystemPrompt` 的非交互模式。

### 归因 Header

**`getAttributionHeader(fingerprint: string): string`**

为 API 请求生成 `x-anthropic-billing-header`。格式如下：
```
x-anthropic-billing-header: cc_version=<VERSION>.<fingerprint>; cc_entrypoint=<entrypoint>;[cch=00000;][cc_workload=<workload>;]
```

- `cc_version`: `${MACRO.VERSION}.${fingerprint}` — 标识构建版本。
- `cc_entrypoint`: 来自环境变量 `CLAUDE_CODE_ENTRYPOINT`（默认 `'unknown'`）。
- `cch=00000`: 原生客户端证明占位符（仅在启用 `NATIVE_CLIENT_ATTESTATION` feature 时存在）。Bun 的 HTTP 栈会在发送前把这些 0 覆写成计算出的 hash。
- `cc_workload`: 按轮次生效的 workload 提示，供 API 路由使用（缺省表示交互式默认值）。

默认启用。可以通过 `CLAUDE_CODE_ATTRIBUTION_HEADER=false` 或 GrowthBook 开关 `tengu_attribution_header` 关闭。

---

## 17. 系统提示区段

**文件：** `constants/systemPromptSections.ts`
**用途：** 系统提示分段的缓存基础设施。

### 类型

```typescript
type ComputeFn = () => string | null | Promise<string | null>

type SystemPromptSection = {
  name: string
  compute: ComputeFn
  cacheBreak: boolean
}
```

### 函数

**`systemPromptSection(name, compute): SystemPromptSection`**
创建一个带缓存的系统提示分段（`cacheBreak: false`）。只计算一次，并一直缓存到 `/clear` 或 `/compact`。

**`DANGEROUS_uncachedSystemPromptSection(name, compute, _reason): SystemPromptSection`**
创建一个每轮都会重新计算的易变系统提示分段（`cacheBreak: true`）。当值变化时，它会打破提示缓存。`_reason` 参数用于说明为什么必须打破缓存。

**`resolveSystemPromptSections(sections): Promise<(string | null)[]>`**
解析所有分段并返回计算结果。未标记为 `cacheBreak` 的分段会优先使用缓存值。

**`clearSystemPromptSections(): void`**
清空所有系统提示分段状态，并重置 beta 头锁存器。会在 `/clear` 和 `/compact` 时调用。

---

## 18. 工具限制

**文件：** `constants/toolLimits.ts`

### 尺寸常量

| 常量 | 值 | 说明 |
|---|---|---|
| `DEFAULT_MAX_RESULT_SIZE_CHARS` | `50_000` | 工具结果写盘前允许的默认最大字符数。单个工具可以声明更低的上限；这是系统级总上限。 |
| `MAX_TOOL_RESULT_TOKENS` | `100_000` | 工具结果允许的最大 token 数。大约相当于 400 KB 文本（按每 token 约 4 字节估算）。 |
| `BYTES_PER_TOKEN` | `4` | 字节/Token 转换的保守估计值，实际情况可能不同。 |
| `MAX_TOOL_RESULT_BYTES` | `MAX_TOOL_RESULT_TOKENS * BYTES_PER_TOKEN` = **400,000 bytes** (400 KB) | 工具结果允许的最大字节数（由 token 上限推导）。 |
| `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` | `200_000` | 单条用户消息里所有工具结果块的最大总字符数（即一轮并行结果的总预算）。避免 N 个并行工具各自都打到 50K，最后在一轮里凑出例如 10 × 40K = 400K。可通过 GrowthBook 标记 `tengu_hawthorn_window` 覆盖。 |
| `TOOL_SUMMARY_MAX_LENGTH` | `50` | 紧凑视图里工具摘要字符串的最大字符数。供 `getToolUseSummary()` 实现使用。 |

---

## 19. 工具集合

**文件：** `constants/tools.ts`
**用途：** 定义不同代理上下文下允许或禁止的工具。

### `ALL_AGENT_DISALLOWED_TOOLS` — `Set<string>`

所有子代理（非主线程）都禁止使用的工具：
- `TaskOutputTool` — 输出路由工具（仅主线程）
- `ExitPlanModeV2Tool` — Plan mode 退出（主线程抽象）
- `EnterPlanModeTool` — Plan mode 进入（主线程抽象）
- `AgentTool` — 嵌套 agents（外部用户禁用；`ant` 用户可用）
- `AskUserQuestionTool` — 需要人工交互
- `TaskStopTool` — 需要主线程任务状态
- `WorkflowTool` — 防止递归执行 workflow（启用 `WORKFLOW_SCRIPTS` feature 时）

### `CUSTOM_AGENT_DISALLOWED_TOOLS` — `Set<string>`

与 `ALL_AGENT_DISALLOWED_TOOLS` 相同，成员完全一致。

### `ASYNC_AGENT_ALLOWED_TOOLS` — `Set<string>`

异步后台代理可用的工具：
- `FileReadTool`, `WebSearchTool`, `TodoWriteTool`, `GrepTool`, `WebFetchTool`, `GlobTool`
- All shell tools (`SHELL_TOOL_NAMES`)
- `FileEditTool`, `FileWriteTool`, `NotebookEditTool`
- `SkillTool`, `SyntheticOutputTool`, `ToolSearchTool`
- `EnterWorktreeTool`, `ExitWorktreeTool`

**异步代理明确禁止：**
- `AgentTool` — Prevents recursion
- `TaskOutputTool` — Prevents recursion
- `ExitPlanModeTool` — Plan mode is a main-thread abstraction
- `TaskStopTool` — Requires access to main-thread task state
- `TungstenTool` — Uses singleton virtual terminal conflicting between agents

### `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` — `Set<string>`

仅供进程内队友使用的额外工具，不对普通异步代理开放。由 `inProcessRunner.ts` 注入：
- `TaskCreateTool`, `TaskGetTool`, `TaskListTool`, `TaskUpdateTool`
- `SendMessageTool`
- `CronCreateTool`, `CronDeleteTool`, `CronListTool` (when `AGENT_TRIGGERS` feature enabled)

### `COORDINATOR_MODE_ALLOWED_TOOLS` — `Set<string>`

协调模式下可用的工具（协调器负责调度 worker，不直接执行工作）：
- `AgentTool` — Spawns workers
- `TaskStopTool` — Stops tasks
- `SendMessageTool` — Inter-agent communication
- `SyntheticOutputTool` — Output synthesis

---

## 20. 回合完成动词

**文件：** `constants/turnCompletionVerbs.ts`

### `TURN_COMPLETION_VERBS` — `string[]`

用于回合完成消息的过去式动词。会和 "for [duration]" 搭配使用（例如 `"Worked for 5s"`）：

`'Baked'`, `'Brewed'`, `'Churned'`, `'Cogitated'`, `'Cooked'`, `'Crunched'`, `'Sautéed'`, `'Worked'`

---

## 21. XML 标签

**文件：** `constants/xml.ts`

### Command/Skill Tags

| 常量 | 值 | 说明 |
|---|---|---|
| `COMMAND_NAME_TAG` | `'command-name'` | 标记消息中的 skill/command 名称。 |
| `COMMAND_MESSAGE_TAG` | `'command-message'` | 标记 skill/command 的消息内容。 |
| `COMMAND_ARGS_TAG` | `'command-args'` | 标记 skill/command 参数。 |

### Terminal / Bash Tags

| 常量 | 值 | 说明 |
|---|---|---|
| `BASH_INPUT_TAG` | `'bash-input'` | 包裹用户消息里的 bash 命令输入内容。 |
| `BASH_STDOUT_TAG` | `'bash-stdout'` | 包裹 bash stdout 输出。 |
| `BASH_STDERR_TAG` | `'bash-stderr'` | 包裹 bash stderr 输出。 |
| `LOCAL_COMMAND_STDOUT_TAG` | `'local-command-stdout'` | 包裹本地命令 stdout。 |
| `LOCAL_COMMAND_STDERR_TAG` | `'local-command-stderr'` | 包裹本地命令 stderr。 |
| `LOCAL_COMMAND_CAVEAT_TAG` | `'local-command-caveat'` | 包裹本地命令的注意事项。 |

### `TERMINAL_OUTPUT_TAGS` — `readonly string[]`

所有与终端相关的标签（用于判断消息是不是终端输出，而不是用户提示）：
`[BASH_INPUT_TAG, BASH_STDOUT_TAG, BASH_STDERR_TAG, LOCAL_COMMAND_STDOUT_TAG, LOCAL_COMMAND_STDERR_TAG, LOCAL_COMMAND_CAVEAT_TAG]`

### Proactive/Autonomous Mode

| 常量 | 值 | 说明 |
|---|---|---|
| `TICK_TAG` | `'tick'` | 包裹主动模式的心跳提示。Claude 会把 `<tick>` 理解为“你醒了，现在做什么？” |

### Task Notification Tags

| 常量 | 值 | 说明 |
|---|---|---|
| `TASK_NOTIFICATION_TAG` | `'task-notification'` | 包裹后台任务完成通知。 |
| `TASK_ID_TAG` | `'task-id'` | 通知里的任务 ID。 |
| `TOOL_USE_ID_TAG` | `'tool-use-id'` | 通知里的工具使用 ID。 |
| `TASK_TYPE_TAG` | `'task-type'` | 通知里的任务类型。 |
| `OUTPUT_FILE_TAG` | `'output-file'` | 通知里的输出文件路径。 |
| `STATUS_TAG` | `'status'` | 通知里的状态。 |
| `SUMMARY_TAG` | `'summary'` | 通知里的摘要。 |
| `REASON_TAG` | `'reason'` | 通知里的原因。 |
| `WORKTREE_TAG` | `'worktree'` | worktree 信息。 |
| `WORKTREE_PATH_TAG` | `'worktreePath'` | worktree 路径。 |
| `WORKTREE_BRANCH_TAG` | `'worktreeBranch'` | worktree 分支。 |

### Feature-Specific Tags

| 常量 | 值 | 说明 |
|---|---|---|
| `ULTRAPLAN_TAG` | `'ultraplan'` | Ultraplan 模式（远程并行规划会话）。 |
| `REMOTE_REVIEW_TAG` | `'remote-review'` | 远程 review 会话通过 `/review` 返回的结果。 |
| `REMOTE_REVIEW_PROGRESS_TAG` | `'remote-review-progress'` | 来自 `run_hunt.sh` 编排器的心跳进度（约 10 秒一次）。 |
| `TEAMMATE_MESSAGE_TAG` | `'teammate-message'` | swarm 内代理间通信。 |
| `CHANNEL_MESSAGE_TAG` | `'channel-message'` | 外部 channel 消息。 |
| `CHANNEL_TAG` | `'channel'` | channel 消息里的 channel 标识。 |
| `CROSS_SESSION_MESSAGE_TAG` | `'cross-session-message'` | 来自另一个 Claude 会话收件箱的跨会话 UDS 消息。 |
| `FORK_BOILERPLATE_TAG` | `'fork-boilerplate'` | 包裹 fork 子进程首条消息里的规则/格式模板。这样 transcript 渲染器可以折叠模板，只显示指令。 |
| `FORK_DIRECTIVE_PREFIX` | `'Your directive: '` | fork 消息里指令文本前面的前缀，由渲染器去除。 |

### Slash Command Argument Patterns

**`COMMON_HELP_ARGS`** — `string[]`
Common arguments for help requests: `['help', '-h', '--help']`

**`COMMON_INFO_ARGS`** — `string[]`
当前状态 / 信息查询的常用参数：
`['list', 'show', 'display', 'current', 'view', 'get', 'check', 'describe', 'print', 'version', 'about', 'status', '?']`

---

## 22. 工具类型系统

**文件：** `Tool.ts`

### Core Types

#### `ToolInputJSONSchema`
```typescript
type ToolInputJSONSchema = {
  [x: string]: unknown
  type: 'object'
  properties?: { [x: string]: unknown }
}
```

#### `QueryChainTracking`
```typescript
type QueryChainTracking = {
  chainId: string
  depth: number
}
```

#### `ValidationResult`
```typescript
type ValidationResult =
  | { result: true }
  | { result: false; message: string; errorCode: number }
```

#### `SetToolJSXFn`
为渲染自定义 JSX UI 的工具提供的回调：
```typescript
type SetToolJSXFn = (args: {
  jsx: React.ReactNode | null
  shouldHidePromptInput: boolean
  shouldContinueAnimation?: true
  showSpinner?: boolean
  isLocalJSXCommand?: boolean
  isImmediate?: boolean
  clearLocalJSX?: boolean
} | null) => void
```

#### `ToolPermissionContext`
```typescript
type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>
```

**`getEmptyToolPermissionContext(): ToolPermissionContext`** — 返回一个最小上下文，`mode` 为 `default`，所有规则都为空对象。

#### `CompactProgressEvent`
```typescript
type CompactProgressEvent =
  | { type: 'hooks_start'; hookType: 'pre_compact' | 'post_compact' | 'session_start' }
  | { type: 'compact_start' }
  | { type: 'compact_end' }
```

#### `ToolUseContext`
传给每次工具调用的主上下文对象。包含：

- `options`: `{ commands, debug, mainLoopModel, tools, verbose, thinkingConfig, mcpClients, mcpResources, isNonInteractiveSession, agentDefinitions, maxBudgetUsd?, customSystemPrompt?, appendSystemPrompt?, querySource?, refreshTools? }`
- `abortController: AbortController`
- `readFileState: FileStateCache`
- `getAppState(): AppState`
- `setAppState(f): void`
- `setAppStateForTasks?: (f) => void` — Always-shared setAppState for session-scoped infrastructure
- `handleElicitation?: (serverName, params, signal) => Promise<ElicitResult>`
- `setToolJSX?: SetToolJSXFn`
- `addNotification?: (notif) => void`
- `appendSystemMessage?: (msg) => void`
- `sendOSNotification?: (opts) => void`
- `nestedMemoryAttachmentTriggers?: Set<string>`
- `loadedNestedMemoryPaths?: Set<string>`
- `dynamicSkillDirTriggers?: Set<string>`
- `discoveredSkillNames?: Set<string>`
- `userModified?: boolean`
- `setInProgressToolUseIDs: (f) => void`
- `setHasInterruptibleToolInProgress?: (v) => void`
- `setResponseLength: (f) => void`
- `pushApiMetricsEntry?: (ttftMs) => void`
- `setStreamMode?: (mode) => void`
- `onCompactProgress?: (event) => void`
- `setSDKStatus?: (status) => void`
- `openMessageSelector?: () => void`
- `updateFileHistoryState: (updater) => void`
- `updateAttributionState: (updater) => void`
- `setConversationId?: (id) => void`
- `agentId?: AgentId`
- `agentType?: string`
- `requireCanUseTool?: boolean`
- `messages: Message[]`
- `fileReadingLimits?: { maxTokens?, maxSizeBytes? }`
- `globLimits?: { maxResults? }`
- `toolDecisions?: Map<string, { source, decision, timestamp }>`
- `queryTracking?: QueryChainTracking`
- `requestPrompt?: (sourceName, toolInputSummary?) => (request) => Promise<PromptResponse>`
- `toolUseId?: string`
- `criticalSystemReminder_EXPERIMENTAL?: string`
- `preserveToolUseResults?: boolean`
- `localDenialTracking?: DenialTrackingState`
- `contentReplacementState?: ContentReplacementState`
- `renderedSystemPrompt?: SystemPrompt`

#### `ToolResult<T>`
```typescript
type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: { _meta?: Record<string, unknown>; structuredContent?: Record<string, unknown> }
}
```

#### `Tool<Input, Output, P>` — 完整接口

主工具接口。关键方法和属性：

| 属性/方法 | 必需 | 说明 |
|---|---|---|
| `name: string` | 是 | 工具主名称。 |
| `aliases?: string[]` | 否 | 兼容旧版本的别名。 |
| `searchHint?: string` | 否 | 供 ToolSearch 关键词匹配的一句话能力描述（3 到 10 个词）。 |
| `inputSchema: Input` | 是 | 用于输入校验的 Zod schema。 |
| `inputJSONSchema?: ToolInputJSONSchema` | 否 | 供 MCP 工具使用的替代 JSON Schema。 |
| `outputSchema?: ZodType` | 否 | 用于输出校验的 Zod schema。 |
| `maxResultSizeChars: number` | 是 | 结果写盘前允许的最大字符数。对绝不能落盘的工具（例如 `Read`）可用 `Infinity`。 |
| `strict?: boolean` | 否 | 为 `true` 时，启用严格模式以遵守 API 参数要求（需要 `tengu_tool_pear`）。 |
| `shouldDefer?: boolean` | 否（只读） | 为 `true` 时，工具会延后加载，调用前必须先经过 `ToolSearch`。 |
| `alwaysLoad?: boolean` | 否（只读） | 为 `true` 时，永不延后加载，完整 schema 总是出现在初始提示词里。 |
| `mcpInfo?: { serverName, toolName }` | 否 | MCP 服务端和工具名（所有 MCP 工具都有）。 |
| `isMcp?: boolean` | 否 | 是否为 MCP 工具。 |
| `isLsp?: boolean` | 否 | 是否为 LSP 工具。 |
| `call(args, context, canUseTool, parentMessage, onProgress?)` | 是 | 执行工具，返回 `Promise<ToolResult<Output>>`。 |
| `description(input, options)` | 是 | 返回权限对话框里显示的工具说明。 |
| `prompt(options)` | 是 | 返回注入系统提示词的工具文档。 |
| `checkPermissions(input, context)` | 是 | 返回 `Promise<PermissionResult>`。 |
| `validateInput?(input, context)` | 否 | 返回 `Promise<ValidationResult>`，在 `checkPermissions` 之前调用。 |
| `isEnabled()` | 是（默认 `true`） | 工具当前是否可用。 |
| `isConcurrencySafe(input)` | 是（默认 `false`） | 工具是否可以安全并行运行。 |
| `isReadOnly(input)` | 是（默认 `false`） | 工具是否只读（不会写入）。 |
| `isDestructive?(input)` | 否（默认 `false`） | 工具是否会执行不可逆操作。 |
| `interruptBehavior?(): 'cancel' \| 'block'` | 否（默认 `'block'`） | 用户在工具执行期间提交时会发生什么。 |
| `isSearchOrReadCommand?(input)` | 否 | 返回 `{ isSearch, isRead, isList? }`，用于 UI 折叠。 |
| `isOpenWorld?(input)` | 否 | 工具是否会影响外部状态。 |
| `requiresUserInteraction?()` | 否 | 工具是否需要交互式 UI。 |
| `inputsEquivalent?(a, b)` | 否 | 比较两个输入是否相同，用于去重。 |
| `isTransparentWrapper?()` | 否 | 为 `true` 时，把所有渲染都交给进度处理器。 |
| `backfillObservableInput?(input)` | 否 | 修改输入副本，在观察者看到之前补上旧字段或派生字段。 |
| `getPath?(input)` | 否 | 返回这个工具操作的文件路径。 |
| `preparePermissionMatcher?(input)` | 否 | 为权限规则准备 `if` 条件匹配器。 |
| `userFacingName(input)` | 是（默认 `name`） | 供 UI 展示的人类可读名称。 |
| `userFacingNameBackgroundColor?(input)` | 否 | 彩色名称显示所用的主题键。 |
| `getToolUseSummary?(input)` | 否 | 供紧凑视图使用的简短摘要（最长 `TOOL_SUMMARY_MAX_LENGTH` 个字符）。 |
| `getActivityDescription?(input)` | 否 | 给 spinner 用的现在时活动描述（例如 `"Reading src/foo.ts"`）。 |
| `toAutoClassifierInput(input)` | 是（默认 `''`） | 供自动模式安全分类器使用的紧凑表示（返回 `''` 表示跳过）。 |
| `mapToolResultToToolResultBlockParam(content, toolUseID)` | 是 | 把工具结果序列化成 API 所需格式。 |
| `renderToolResultMessage?(content, progressMessages, options)` | 否 | React 层的结果渲染；若结果在别处展示，可省略。 |
| `extractSearchText?(out)` | 否 | 用于会话搜索索引的扁平文本。 |
| `renderToolUseMessage(input, options)` | 是 | 工具调用的 React 渲染（会立即调用，输入可能还是部分内容）。 |
| `isResultTruncated?(output)` | 否 | 非详细渲染是否被截断（用于控制点击展开）。 |
| `renderToolUseTag?(input)` | 否 | 工具调用消息后面可附带的标签（超时、模型、恢复 ID 等）。 |
| `renderToolUseProgressMessage?(progressMessages, options)` | 否 | 运行中的进度渲染。 |
| `renderToolUseQueuedMessage?()` | 否 | 队列状态的 React 渲染。 |
| `renderToolUseRejectedMessage?(input, options)` | 否 | 自定义拒绝 UI，默认回退到 `<FallbackToolUseRejectedMessage />`。 |
| `renderToolUseErrorMessage?(result, options)` | 否 | 自定义错误 UI，默认回退到 `<FallbackToolUseErrorMessage />`。 |
| `renderGroupedToolUse?(toolUses, options)` | 否 | 将多个并行实例作为一个组来渲染（仅非详细模式）。 |

#### `Tools` — `readonly Tool[]`

工具集合的类型别名，用于在各处传递工具组。

#### `ToolDef` — 部分工具定义

和 `Tool` 形状相同，但可设置默认值的方法是可选的。`buildTool()` 接受这种形式。

**可设默认值的键：** `isEnabled`, `isConcurrencySafe`, `isReadOnly`, `isDestructive`, `checkPermissions`, `toAutoClassifierInput`, `userFacingName`

**默认值：**
- `isEnabled` → `() => true`
- `isConcurrencySafe` → `() => false`
- `isReadOnly` → `() => false`
- `isDestructive` → `() => false`
- `checkPermissions` → `Promise.resolve({ behavior: 'allow', updatedInput: input })`
- `toAutoClassifierInput` → `() => ''`
- `userFacingName` → `() => def.name`

#### `buildTool<D>(def: D): BuiltTool<D>`

从 `ToolDef` 构建完整 `Tool` 的工厂函数，会自动补齐默认值。所有工具导出都应经过这个函数。`userFacingName` 的默认值是 `() => def.name`。

---

## 23. 任务类型系统

**文件：** `Task.ts`

### 类型

#### `TaskType` — 联合类型
```typescript
type TaskType =
  | 'local_bash'        // 后台 shell 命令
  | 'local_agent'       // 本地运行的子代理
  | 'remote_agent'      // 远程服务器上的代理
  | 'in_process_teammate' // 进程内的 swarm 队友
  | 'local_workflow'    // 本地工作流执行
  | 'monitor_mcp'       // MCP 订阅监视器
  | 'dream'             // Dream / auto-dream 后台代理
```

#### `TaskStatus` — 联合类型
```typescript
type TaskStatus =
  | 'pending'    // 尚未开始
  | 'running'    // 正在执行
  | 'completed'  // 已成功完成
  | 'failed'     // 以错误结束
  | 'killed'     // 被手动终止
```

**`isTerminalTaskStatus(status): boolean`** — 当状态是 `'completed'`、`'failed'` 或 `'killed'` 时返回 `true`。用于防止给已结束的队友注入消息。

#### `TaskHandle`
```typescript
type TaskHandle = {
  taskId: string
  cleanup?: () => void
}
```

#### `SetAppState` — `(f: (prev: AppState) => AppState) => void`

#### `TaskContext`
```typescript
type TaskContext = {
  abortController: AbortController
  getAppState: () => AppState
  setAppState: SetAppState
}
```

#### `TaskStateBase`
```typescript
type TaskStateBase = {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  toolUseId?: string
  startTime: number       // Unix timestamp ms
  endTime?: number        // Unix timestamp ms
  totalPausedMs?: number
  outputFile: string      // Path to disk output file
  outputOffset: number    // Bytes already read from outputFile
  notified: boolean       // Whether user has been notified of completion
}
```

#### `LocalShellSpawnInput`
```typescript
type LocalShellSpawnInput = {
  command: string
  description: string
  timeout?: number
  toolUseId?: string
  agentId?: AgentId
  kind?: 'bash' | 'monitor'  // UI display variant
}
```

#### `Task`
```typescript
type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

### 任务 ID 生成

**Task ID Prefixes** (by `TaskType`):

| 类型 | 前缀 |
|---|---|
| `local_bash` | `'b'` |
| `local_agent` | `'a'` |
| `remote_agent` | `'r'` |
| `in_process_teammate` | `'t'` |
| `local_workflow` | `'w'` |
| `monitor_mcp` | `'m'` |
| `dream` | `'d'` |
| (unknown) | `'x'` |

**`TASK_ID_ALPHABET`** = `'0123456789abcdefghijklmnopqrstuvwxyz'` (36 chars)

**`generateTaskId(type): string`** — Generates a task ID as `{prefix}` + 8 random base-36 characters. Uses `randomBytes(8)` for cryptographic randomness. Total space: 36^8 ≈ 2.8 trillion combinations.

**`createTaskStateBase(id, type, description, toolUseId?): TaskStateBase`** — Creates a `TaskStateBase` with `status: 'pending'`, `startTime: Date.now()`, `outputFile: getTaskOutputPath(id)`, `outputOffset: 0`, `notified: false`.

---

## 24. types 目录

### 24.1 IDs (`types/ids.ts`)

**带品牌的类型，用来在编译期避免 ID 类型混淆：

```typescript
type SessionId = string & { readonly __brand: 'SessionId' }
type AgentId = string & { readonly __brand: 'AgentId' }
```

**函数：**
- `asSessionId(id: string): SessionId` — Cast string to SessionId (use sparingly).
- `asAgentId(id: string): AgentId` — Cast string to AgentId (use sparingly).
- `toAgentId(s: string): AgentId | null` — Validates and brands. Matches pattern `^a(?:.+-)?[0-9a-f]{16}$` (letter `a` + optional `<label>-` + 16 hex chars).

### 24.2 Permissions (`types/permissions.ts`)

#### Permission Modes

```typescript
// User-addressable external modes
const EXTERNAL_PERMISSION_MODES = ['acceptEdits', 'bypassPermissions', 'default', 'dontAsk', 'plan']
type ExternalPermissionMode = 'acceptEdits' | 'bypassPermissions' | 'default' | 'dontAsk' | 'plan'

// Internal modes (includes auto when TRANSCRIPT_CLASSIFIER enabled)
type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
type PermissionMode = InternalPermissionMode

// Runtime validation set
const INTERNAL_PERMISSION_MODES = [...EXTERNAL_PERMISSION_MODES, ...('auto' if TRANSCRIPT_CLASSIFIER)]
const PERMISSION_MODES = INTERNAL_PERMISSION_MODES
```

**模式说明：**
- `'default'` — Normal interactive mode; prompts user for each new operation type.
- `'acceptEdits'` — Auto-approves file edits without prompting.
- `'dontAsk'` — Auto-approves all tool calls.
- `'bypassPermissions'` — Bypasses all permission checks (dangerous).
- `'plan'` — Plan mode; Claude can plan but not execute.
- `'auto'` — Automatic mode with classifier-based approval (TRANSCRIPT_CLASSIFIER feature).
- `'bubble'` — Bubbles permission requests to parent context.

#### Permission Behavior
```typescript
type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

#### `PermissionRuleSource`
```typescript
type PermissionRuleSource =
  | 'userSettings' | 'projectSettings' | 'localSettings' | 'flagSettings'
  | 'policySettings' | 'cliArg' | 'command' | 'session'
```

#### `PermissionRuleValue`
```typescript
type PermissionRuleValue = { toolName: string; ruleContent?: string }
```

#### `PermissionRule`
```typescript
type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior
  ruleValue: PermissionRuleValue
}
```

#### `PermissionUpdateDestination`
```typescript
type PermissionUpdateDestination =
  | 'userSettings' | 'projectSettings' | 'localSettings' | 'session' | 'cliArg'
```

#### `PermissionUpdate` — Discriminated Union
```typescript
type PermissionUpdate =
  | { type: 'addRules'; destination; rules: PermissionRuleValue[]; behavior }
  | { type: 'replaceRules'; destination; rules: PermissionRuleValue[]; behavior }
  | { type: 'removeRules'; destination; rules: PermissionRuleValue[]; behavior }
  | { type: 'setMode'; destination; mode: ExternalPermissionMode }
  | { type: 'addDirectories'; destination; directories: string[] }
  | { type: 'removeDirectories'; destination; directories: string[] }
```

#### `AdditionalWorkingDirectory`
```typescript
type AdditionalWorkingDirectory = { path: string; source: WorkingDirectorySource }
```

#### Permission Decisions

```typescript
type PermissionAllowDecision<Input> = {
  behavior: 'allow'
  updatedInput?: Input
  userModified?: boolean
  decisionReason?: PermissionDecisionReason
  toolUseID?: string
  acceptFeedback?: string
  contentBlocks?: ContentBlockParam[]
}

type PermissionAskDecision<Input> = {
  behavior: 'ask'
  message: string
  updatedInput?: Input
  decisionReason?: PermissionDecisionReason
  suggestions?: PermissionUpdate[]
  blockedPath?: string
  metadata?: PermissionMetadata
  isBashSecurityCheckForMisparsing?: boolean
  pendingClassifierCheck?: PendingClassifierCheck
  contentBlocks?: ContentBlockParam[]
}

type PermissionDenyDecision = {
  behavior: 'deny'
  message: string
  decisionReason: PermissionDecisionReason
  toolUseID?: string
}

type PermissionResult<Input> =
  | PermissionDecision<Input>
  | { behavior: 'passthrough'; message; decisionReason?; suggestions?; blockedPath?; pendingClassifierCheck? }
```

#### `PermissionDecisionReason` — Discriminated Union

```typescript
type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'permissionPromptTool'; permissionPromptToolName: string; toolResult: unknown }
  | { type: 'hook'; hookName: string; hookSource?: string; reason?: string }
  | { type: 'asyncAgent'; reason: string }
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'workingDir'; reason: string }
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }
  | { type: 'other'; reason: string }
```

#### Classifier Types

```typescript
type ClassifierResult = {
  matches: boolean
  matchedDescription?: string
  confidence: 'high' | 'medium' | 'low'
  reason: string
}

type ClassifierBehavior = 'deny' | 'ask' | 'allow'

type ClassifierUsage = {
  inputTokens: number; outputTokens: number
  cacheReadInputTokens: number; cacheCreationInputTokens: number
}

type YoloClassifierResult = {
  thinking?: string
  shouldBlock: boolean
  reason: string
  unavailable?: boolean
  transcriptTooLong?: boolean
  model: string
  usage?: ClassifierUsage
  durationMs?: number
  promptLengths?: { systemPrompt: number; toolCalls: number; userPrompts: number }
  errorDumpPath?: string
  stage?: 'fast' | 'thinking'
  stage1Usage?: ClassifierUsage
  stage1DurationMs?: number
  stage1RequestId?: string
  stage1MsgId?: string
  stage2Usage?: ClassifierUsage
  stage2DurationMs?: number
  stage2RequestId?: string
  stage2MsgId?: string
}
```

#### Other Permission Types

```typescript
type RiskLevel = 'LOW' | 'MEDIUM' | 'HIGH'

type PermissionExplanation = {
  riskLevel: RiskLevel
  explanation: string
  reasoning: string
  risk: string
}

type ToolPermissionRulesBySource = {
  [T in PermissionRuleSource]?: string[]
}
```

### 24.3 Commands (`types/command.ts`)

#### `LocalCommandResult`
```typescript
type LocalCommandResult =
  | { type: 'text'; value: string }
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
  | { type: 'skip' }
```

#### `PromptCommand`
```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number
  argNames?: string[]
  allowedTools?: string[]
  model?: string
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  pluginInfo?: { pluginManifest: PluginManifest; repository: string }
  disableNonInteractive?: boolean
  hooks?: HooksSettings
  skillRoot?: string
  context?: 'inline' | 'fork'  // 'inline' = expands in current conversation, 'fork' = subagent
  agent?: string
  effort?: EffortValue
  paths?: string[]              // Glob patterns limiting skill visibility by touched files
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>
}
```

#### `CommandAvailability`
```typescript
type CommandAvailability = 'claude-ai' | 'console'
```
- `'claude-ai'` — Claude.ai OAuth subscriber (Pro/Max/Team/Enterprise)
- `'console'` — Console API key user (direct api.anthropic.com)

#### `CommandBase`
```typescript
type CommandBase = {
  availability?: CommandAvailability[]
  description: string
  hasUserSpecifiedDescription?: boolean
  isEnabled?: () => boolean         // Default: true
  isHidden?: boolean                // Default: false
  name: string
  aliases?: string[]
  isMcp?: boolean
  argumentHint?: string
  whenToUse?: string
  version?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  kind?: 'workflow'
  immediate?: boolean               // Executes immediately without queuing
  isSensitive?: boolean             // Args redacted from conversation history
  userFacingName?: () => string     // Default: name
}

type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

#### `ResumeEntrypoint`
```typescript
type ResumeEntrypoint =
  | 'cli_flag'
  | 'slash_command_picker'
  | 'slash_command_session_id'
  | 'slash_command_title'
  | 'fork'
```

#### `QueuePriority`
```typescript
type QueuePriority = 'now' | 'next' | 'later'
```
- `'now'` — Interrupt immediately, abort in-flight tool calls.
- `'next'` — Drain mid-turn; let current tool finish, then send.
- `'later'` — Drain end-of-turn; wait for current turn to finish.

### 24.4 Hooks (`types/hooks.ts`)

#### `PromptRequest` / `PromptResponse`
```typescript
type PromptRequest = {
  prompt: string           // request id (discriminator)
  message: string
  options: Array<{ key: string; label: string; description?: string }>
}

type PromptResponse = {
  prompt_response: string  // request id
  selected: string
}
```

#### `HookCallback`
```typescript
type HookCallback = {
  type: 'callback'
  callback: (input, toolUseID, abort, hookIndex?, context?) => Promise<HookJSONOutput>
  timeout?: number
  internal?: boolean       // Excludes from tengu_run_hook metrics
}
```

#### `HookProgress`
```typescript
type HookProgress = {
  type: 'hook_progress'
  hookEvent: HookEvent
  hookName: string
  command: string
  promptText?: string
  statusMessage?: string
}
```

#### `HookResult`
```typescript
type HookResult = {
  message?: Message
  systemMessage?: Message
  blockingError?: HookBlockingError
  outcome: 'success' | 'blocking' | 'non_blocking_error' | 'cancelled'
  preventContinuation?: boolean
  stopReason?: string
  permissionBehavior?: 'ask' | 'deny' | 'allow' | 'passthrough'
  hookPermissionDecisionReason?: string
  additionalContext?: string
  initialUserMessage?: string
  updatedInput?: Record<string, unknown>
  updatedMCPToolOutput?: unknown
  permissionRequestResult?: PermissionRequestResult
  retry?: boolean
}
```

#### `PermissionRequestResult`
```typescript
type PermissionRequestResult =
  | { behavior: 'allow'; updatedInput?: Record<string, unknown>; updatedPermissions?: PermissionUpdate[] }
  | { behavior: 'deny'; message?: string; interrupt?: boolean }
```

#### `Sync Hook Response Schema Fields`
当 hook 输出 JSON 时，可以包含：
- `continue?: boolean` — Whether Claude should continue after hook (default: `true`)
- `suppressOutput?: boolean` — Hide stdout from transcript (default: `false`)
- `stopReason?: string` — Message shown when `continue` is `false`
- `decision?: 'approve' | 'block'`
- `reason?: string`
- `systemMessage?: string`
- `hookSpecificOutput?` — Event-specific data (varies by `hookEventName`):
  - `PreToolUse`: `permissionDecision`, `permissionDecisionReason`, `updatedInput`, `additionalContext`
  - `UserPromptSubmit`: `additionalContext`
  - `SessionStart`: `additionalContext`, `initialUserMessage`, `watchPaths`
  - `Setup`: `additionalContext`
  - `SubagentStart`: `additionalContext`
  - `PostToolUse`: `additionalContext`, `updatedMCPToolOutput`
  - `PostToolUseFailure`: `additionalContext`
  - `PermissionDenied`: `retry`
  - `Notification`: `additionalContext`
  - `PermissionRequest`: `decision` (allow with `updatedInput`/`updatedPermissions`, or deny with `message`/`interrupt`)
  - `Elicitation`/`ElicitationResult`: `action` (`'accept'|'decline'|'cancel'`), `content`
  - `CwdChanged`/`FileChanged`: `watchPaths`
  - `WorktreeCreate`: `worktreePath`

### 24.5 Plugins (`types/plugin.ts`)

#### `BuiltinPluginDefinition`
```typescript
type BuiltinPluginDefinition = {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  isAvailable?: () => boolean
  defaultEnabled?: boolean   // Default: true
}
```

#### `LoadedPlugin`
```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string
  source: string
  repository: string
  enabled?: boolean
  isBuiltin?: boolean
  sha?: string
  commandsPath?: string
  commandsPaths?: string[]
  commandsMetadata?: Record<string, CommandMetadata>
  agentsPath?: string
  agentsPaths?: string[]
  skillsPath?: string
  skillsPaths?: string[]
  outputStylesPath?: string
  outputStylesPaths?: string[]
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
  settings?: Record<string, unknown>
}
```

#### `PluginComponent`
```typescript
type PluginComponent = 'commands' | 'agents' | 'skills' | 'hooks' | 'output-styles'
```

#### `PluginError` — Discriminated Union (17 variants)

| 类型 | 关键字段 |
|---|---|
| `'path-not-found'` | `source`, `plugin?`, `path`, `component` |
| `'git-auth-failed'` | `source`, `plugin?`, `gitUrl`, `authType: 'ssh' \| 'https'` |
| `'git-timeout'` | `source`, `plugin?`, `gitUrl`, `operation: 'clone' \| 'pull'` |
| `'network-error'` | `source`, `plugin?`, `url`, `details?` |
| `'manifest-parse-error'` | `source`, `plugin?`, `manifestPath`, `parseError` |
| `'manifest-validation-error'` | `source`, `plugin?`, `manifestPath`, `validationErrors` |
| `'plugin-not-found'` | `source`, `pluginId`, `marketplace` |
| `'marketplace-not-found'` | `source`, `marketplace`, `availableMarketplaces` |
| `'marketplace-load-failed'` | `source`, `marketplace`, `reason` |
| `'mcp-config-invalid'` | `source`, `plugin`, `serverName`, `validationError` |
| `'mcp-server-suppressed-duplicate'` | `source`, `plugin`, `serverName`, `duplicateOf` |
| `'hook-load-failed'` | `source`, `plugin`, `hookPath`, `reason` |
| `'component-load-failed'` | `source`, `plugin`, `component`, `path`, `reason` |
| `'mcpb-download-failed'` | `source`, `plugin`, `url`, `reason` |
| `'mcpb-extract-failed'` | `source`, `plugin`, `mcpbPath`, `reason` |
| `'mcpb-invalid-manifest'` | `source`, `plugin`, `mcpbPath`, `validationError` |
| `'lsp-config-invalid'` | `source`, `plugin`, `serverName`, `validationError` |
| `'lsp-server-start-failed'` | `source`, `plugin`, `serverName`, `reason` |
| `'lsp-server-crashed'` | `source`, `plugin`, `serverName`, `exitCode`, `signal?` |
| `'lsp-request-timeout'` | `source`, `plugin`, `serverName`, `method`, `timeoutMs` |
| `'lsp-request-failed'` | `source`, `plugin`, `serverName`, `method`, `error` |
| `'marketplace-blocked-by-policy'` | `source`, `plugin?`, `marketplace`, `blockedByBlocklist?`, `allowedSources` |
| `'dependency-unsatisfied'` | `source`, `plugin`, `dependency`, `reason: 'not-enabled' \| 'not-found'` |
| `'plugin-cache-miss'` | `source`, `plugin`, `installPath` |
| `'generic-error'` | `source`, `plugin?`, `error` |

#### `PluginLoadResult`
```typescript
type PluginLoadResult = {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
  errors: PluginError[]
}
```

### 24.6 Logs (`types/logs.ts`)

#### `SerializedMessage`
```typescript
type SerializedMessage = Message & {
  cwd: string
  userType: string
  entrypoint?: string   // CLAUDE_CODE_ENTRYPOINT value
  sessionId: string
  timestamp: string
  version: string
  gitBranch?: string
  slug?: string
}
```

#### `LogOption`
带元数据的完整会话日志描述：
```typescript
type LogOption = {
  date: string; messages: SerializedMessage[]
  fullPath?: string; value: number
  created: Date; modified: Date
  firstPrompt: string; messageCount: number
  fileSize?: number; isSidechain: boolean
  isLite?: boolean; sessionId?: string
  teamName?: string; agentName?: string
  agentColor?: string; agentSetting?: string
  isTeammate?: boolean; leafUuid?: UUID
  summary?: string; customTitle?: string; tag?: string
  fileHistorySnapshots?: FileHistorySnapshot[]
  attributionSnapshots?: AttributionSnapshotMessage[]
  contextCollapseCommits?: ContextCollapseCommitEntry[]
  contextCollapseSnapshot?: ContextCollapseSnapshotEntry
  gitBranch?: string; projectPath?: string
  prNumber?: number; prUrl?: string; prRepository?: string
  mode?: 'coordinator' | 'normal'
  worktreeSession?: PersistedWorktreeSession | null
  contentReplacements?: ContentReplacementRecord[]
}
```

#### Transcript 条目类型

下面这些类型都属于 transcript 文件里保存的 `Entry` 联合类型：

| 类型 | 说明 |
|---|---|
| `TranscriptMessage` | 带父级 UUID、sidechain 状态和代理信息的完整序列化消息。 |
| `SummaryMessage` | `{ type: 'summary'; leafUuid; summary }` |
| `CustomTitleMessage` | `{ type: 'custom-title'; sessionId; customTitle }` |
| `AiTitleMessage` | `{ type: 'ai-title'; sessionId; aiTitle }` —— AI 生成；恢复时不会再次追加。 |
| `LastPromptMessage` | `{ type: 'last-prompt'; sessionId; lastPrompt }` |
| `TaskSummaryMessage` | `{ type: 'task-summary'; sessionId; summary; timestamp }` —— 每隔一段时间从 fork 发出的周期性摘要。 |
| `TagMessage` | `{ type: 'tag'; sessionId; tag }` |
| `AgentNameMessage` | `{ type: 'agent-name'; sessionId; agentName }` |
| `AgentColorMessage` | `{ type: 'agent-color'; sessionId; agentColor }` |
| `AgentSettingMessage` | `{ type: 'agent-setting'; sessionId; agentSetting }` |
| `PRLinkMessage` | `{ type: 'pr-link'; sessionId; prNumber; prUrl; prRepository; timestamp }` |
| `ModeEntry` | `{ type: 'mode'; sessionId; mode: 'coordinator' \| 'normal' }` |
| `WorktreeStateEntry` | `{ type: 'worktree-state'; sessionId; worktreeSession: PersistedWorktreeSession \| null }` |
| `ContentReplacementEntry` | `{ type: 'content-replacement'; sessionId; agentId?; replacements }` |
| `FileHistorySnapshotMessage` | `{ type: 'file-history-snapshot'; messageId; snapshot; isSnapshotUpdate }` |
| `AttributionSnapshotMessage` | `{ type: 'attribution-snapshot'; messageId; surface; fileStates; promptCount?; ... }` |
| `ContextCollapseCommitEntry` | `{ type: 'marble-origami-commit'; sessionId; collapseId; summaryUuid; summaryContent; summary; firstArchivedUuid; lastArchivedUuid }` —— 为避免泄露功能名而做的混淆类型名。 |
| `ContextCollapseSnapshotEntry` | `{ type: 'marble-origami-snapshot'; sessionId; staged[]; armed; lastSpawnTokens }` —— 最后写入生效的快照。 |

#### `PersistedWorktreeSession`
```typescript
type PersistedWorktreeSession = {
  originalCwd: string; worktreePath: string; worktreeName: string
  worktreeBranch?: string; originalBranch?: string; originalHeadCommit?: string
  sessionId: string; tmuxSessionName?: string; hookBased?: boolean
}
```

#### `FileAttributionState`
```typescript
type FileAttributionState = {
  contentHash: string       // SHA-256 of file content
  claudeContribution: number // Characters written by Claude
  mtime: number             // File modification time
}
```

### 24.7 文本输入类型（`types/textInputTypes.ts`）

#### `VimMode`
```typescript
type VimMode = 'INSERT' | 'NORMAL'
```

#### `PromptInputMode`
```typescript
type PromptInputMode =
  | 'bash'
  | 'prompt'
  | 'orphaned-permission'
  | 'task-notification'
```

#### `QueuePriority`（`command.ts` 中也有）
```typescript
type QueuePriority = 'now' | 'next' | 'later'
```

#### `QueuedCommand`
```typescript
type QueuedCommand = {
  value: string | Array<ContentBlockParam>
  mode: PromptInputMode
  priority?: QueuePriority
  uuid?: UUID
  orphanedPermission?: OrphanedPermission
  pastedContents?: Record<number, PastedContent>
  preExpansionValue?: string      // Value before [Pasted text #N] expansion (for ultraplan detection)
  skipSlashCommands?: boolean     // Treat as plain text, skip / dispatch
  bridgeOrigin?: boolean          // Remote bridge origin; use isBridgeSafeCommand filter
  isMeta?: boolean                // Hidden from UI but model-visible
  origin?: MessageOrigin          // undefined = human (keyboard)
  workload?: string               // cc_workload= billing header tag
  agentId?: AgentId               // Target agent (undefined = main thread)
}
```

#### `InlineGhostText`
```typescript
type InlineGhostText = {
  readonly text: string            // Ghost text (e.g., "mit" for /commit)
  readonly fullCommand: string     // Full command name (e.g., "commit")
  readonly insertPosition: number  // Position in input where ghost text appears
}
```

#### `BaseTextInputProps`
所有文本输入组件的核心属性，包括：`value`、`onChange`、`onSubmit?`、`onExit?`、`columns`、`cursorOffset`、`onChangeCursorOffset`、`placeholder?`、`multiline?`、`focus?`、`mask?`、`showCursor?`、`highlightPastedText?`、`maxVisibleLines?`、`onImagePaste?`、`onPaste?`、`onIsPastingChange?`、`disableCursorMovementForUpDownKeys?`、`disableEscapeDoublePress?`、`argumentHint?`、`onUndo?`、`dimColor?`、`highlights?`、`placeholderElement?`、`inlineGhostText?`、`inputFilter?`、`onHistoryUp?`、`onHistoryDown?`、`onHistoryReset?`、`onClearInput?`、`onExitMessage?`。

#### `BaseInputState`
```typescript
type BaseInputState = {
  onInput: (input: string, key: Key) => void
  renderedValue: string; offset: number; setOffset: (offset) => void
  cursorLine: number; cursorColumn: number
  viewportCharOffset: number; viewportCharEnd: number
  isPasting?: boolean
  pasteState?: { chunks: string[]; timeoutId: ReturnType<typeof setTimeout> | null }
}
```

#### `OrphanedPermission`
```typescript
type OrphanedPermission = {
  permissionResult: PermissionResult
  assistantMessage: AssistantMessage
}
```

---

## 关键数值限制汇总

| 限制 | 值 | 上下文 |
|---|---|---|
| Image base64 max | 5 MB | API 硬限制 |
| Image raw target | 3.75 MB | 客户端推导目标值 |
| Image max dimension | 2000 px | 客户端缩放上限 |
| PDF raw max | 20 MB | 安全的 API 请求预算 |
| PDF page max (API) | 100 pages | API 硬限制 |
| PDF extract threshold | 3 MB | 超过后切换为逐页图片提取 |
| PDF max extract | 100 MB | PDF 绝对拒绝上限 |
| PDF pages per Read | 20 | 单次 Read 工具调用上限 |
| PDF inline @ mention | 10 pages | 超过后改用引用处理 |
| Media per request | 100 | 图片 + PDF 的合计上限 |
| Tool result default max | 50,000 chars | 单工具结果大小上限 |
| Tool result token max | 100,000 tokens | 约 400 KB |
| Tool results per message | 200,000 chars | 单轮聚合预算 |
| Tool summary max | 50 chars | 紧凑视图摘要上限 |
| Task ID space | ~2.8 trillion | 36^8 组合 |
| Binary check size | 8,192 bytes | 用于二进制内容检测 |
| Binary threshold | 10% | 非可打印字节比例 |
