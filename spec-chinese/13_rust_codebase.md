# Claude Code - Rust 代码库

## 概览

`claude-code-rust/` 下的 Rust 代码库是对 TypeScript 版 Claude Code CLI 的一套**完整独立重写**，使用 async Rust 实现。它不是 FFI 绑定层，也不是部分移植，与 TypeScript 实现不共享任何运行时代码。它重新实现了相同的工具名称和语义、权限模型、`CLAUDE.md` 发现、自动压缩逻辑、MCP（Model Context Protocol）客户端、桥接协议和 cron 调度器，全部基于 Tokio 运行时完成。

### 架构

```
claude-code-rust/
├── Cargo.toml                  # workspace 根
└── crates/
    ├── core/       (cc-core)       # 共享类型、配置、权限、历史、hooks
    ├── api/        (cc-api)        # API 客户端 + SSE 流
    ├── tools/      (cc-tools)      # 所有工具实现（33 个工具）
    ├── query/      (cc-query)      # Agentic 查询循环、compact、cron 调度器
    ├── tui/        (cc-tui)        # ratatui 终端 UI
    ├── commands/   (cc-commands)   # 斜杠命令实现
    ├── mcp/        (cc-mcp)        # MCP（Model Context Protocol）客户端
    ├── bridge/     (cc-bridge)     # 到 claude.ai Web UI 的桥接
    └── cli/        (claude-code)    # 二进制入口点
```

**依赖流：**
```
cli → query → tools → core
         ↓         ↗
        api  →  core
         ↓
       commands → core
         ↓
        tui   → core
         ↓
        mcp   → core
         ↓
       bridge → core
```

---

## 工作区根：`Cargo.toml`

**路径：** `claude-code-rust/Cargo.toml`

Cargo workspace，`resolver = "2"`，所有成员 crate 统一使用 `2021` edition 和 `1.0.0` 版本。

### 工作区成员
| 成员路径 | 包名 | 类型 |
|---|---|---|
| `crates/core` | `cc-core` | 库 |
| `crates/api` | `cc-api` | 库 |
| `crates/tools` | `cc-tools` | 库 |
| `crates/query` | `cc-query` | 库 |
| `crates/tui` | `cc-tui` | 库 |
| `crates/commands` | `cc-commands` | 库 |
| `crates/mcp` | `cc-mcp` | 库 |
| `crates/bridge` | `cc-bridge` | 库 |
| `crates/cli` | `claude-code` | 二进制（`[[bin]] name = "claude"`） |

### 关键共享依赖

| crate | 版本 | 特性 |
|---|---|---|
| `tokio` | 1.44 | `full` |
| `reqwest` | 0.12 | `json`, `stream`, `rustls-tls` |
| `ratatui` | 0.29 | default |
| `crossterm` | 0.28 | `event-stream` |
| `clap` | 4 | `derive`, `env`, `string` |
| `serde` | 1 | `derive` |
| `serde_json` | 1 | default |
| `anyhow` | 1 | default |
| `thiserror` | 2 | default |
| `tracing` | 0.1 | default |
| `tracing-subscriber` | 0.3 | `env-filter` |
| `uuid` | 1 | `v4` |
| `chrono` | 0.4 | `serde` |
| `regex` | 1 | default |
| `glob` | 0.3 | default |
| `walkdir` | 2 | default |
| `similar` | 2 | default（声明了，但使用不多） |
| `once_cell` | 1 | default |
| `parking_lot` | 0.12 | default |
| `dashmap` | 6 | default |
| `tokio-util` | 0.7 | `codec`, `sync` |
| `async-trait` | 0.1 | default |
| `schemars` | 0.8 | `derive` |
| `nix` | 0.29 | `process`, `signal`, `user` |
| `base64` | 0.22 | default |
| `sha2` | 0.10 | default |
| `hex` | 0.4 | default |

---

## 代码包：`cc-core`

**路径：** `crates/core/src/lib.rs`

所有其他 crate 共享的核心 crate。定义了整个系统使用的所有类型，包含 9 个内联子模块。

### 模块：`error`

**`ClaudeError` 枚举**（通过 `thiserror` 实现 `std::error::Error`）：
- `Api(String)` - 通用 API 错误
- `ApiStatus { status_code: u16, message: String }` - HTTP 状态错误
- `Auth(String)` - 认证失败
- `PermissionDenied(String)` - 工具权限被拒绝
- `Tool(String)` - 工具执行错误
- `Io(#[from] std::io::Error)` - I/O 错误
- `Json(#[from] serde_json::Error)` - JSON 解析错误
- `Http(#[from] reqwest::Error)` - HTTP 客户端错误
- `RateLimit { retry_after: Option<u64> }` - 429 限流
- `ContextWindowExceeded` - 上下文窗口已满
- `MaxTokensReached` - 达到 max_tokens
- `Cancelled` - 用户/信号取消
- `Config(String)` - 配置加载/保存错误
- `Mcp(String)` - MCP 协议错误
- `Other(String)` - 兜底错误

**关键方法：**
- `is_retryable(&self) -> bool` - 对 `RateLimit` 和状态码为 529 的 `ApiStatus` 返回 true
- `is_context_limit(&self) -> bool` - 对 `ContextWindowExceeded` 和 `MaxTokensReached` 返回 true

### 模块：`types`

**`Role` 枚举：** `User`, `Assistant`

**`ContentBlock` 枚举**（serde untagged）：
- `Text { text: String }`
- `Image { source: ImageSource }`
- `ToolUse { id: String, name: String, input: Value }`
- `ToolResult { tool_use_id: String, content: ToolResultContent, is_error: Option<bool> }`
- `Thinking { thinking: String, signature: String }`
- `RedactedThinking { data: String }`
- `Document { source: DocumentSource, citations: Option<CitationsConfig> }`

**`MessageContent` 枚举**（serde untagged）：
- `Text(String)`
- `Blocks(Vec<ContentBlock>)`

**`Message` 结构体：**
- 字段：`role: Role`, `content: MessageContent`
- `Message::user(text)` - 便捷构造
- `Message::assistant(text)` - 便捷构造
- `Message::user_blocks(blocks: Vec<ContentBlock>)` - 多块用户消息
- `Message::assistant_blocks(blocks)` - 多块助手消息
- `get_text() -> Option<&str>` - 返回第一个 Text block 或 Text 内容
- `get_all_text() -> String` - 拼接所有文本块
- `get_tool_use_blocks() -> Vec<ContentBlock>` - 过滤 ToolUse 块
- `get_thinking_blocks() -> Vec<ContentBlock>` - 过滤 Thinking 块
- `has_tool_use() -> bool`
- `content_blocks() -> &[ContentBlock]`

**`UsageInfo` 结构体：**
- 字段：`input_tokens: u32`, `output_tokens: u32`, `cache_creation_input_tokens: u32`, `cache_read_input_tokens: u32`
- `total_input() -> u32` - 输入 token + cache token 总和
- `total() -> u32` - 所有 token 总和
- 实现 `Default`

**`ToolDefinition` 结构体：** `{ name: String, description: String, input_schema: Value }`

**支持类型：** `MessageCost`、`ImageSource { type, media_type, data }`、`DocumentSource`、`CitationsConfig`、`ToolResultContent`（Text/Blocks 枚举）

### 模块：`config`

**`Config` 结构体** - 运行时配置：
- `api_key: Option<String>`
- `api_base: Option<String>`
- `model: String`
- `max_tokens: u32`
- `permission_mode: PermissionMode`
- `verbose: bool`
- `output_format: OutputFormat`
- `max_turns: u32`
- `system_prompt: Option<String>`
- `append_system_prompt: Option<String>`
- `no_claude_md: bool`
- `auto_compact: bool`
- `thinking_budget: Option<u32>`
- `mcp_servers: Vec<McpServerConfig>`
- `hooks: HashMap<HookEvent, Vec<HookEntry>>`

**关键方法：**
- `resolve_api_key() -> Option<String>` - 先查 `config.api_key`，再查 `ANTHROPIC_API_KEY` 环境变量
- `resolve_api_base() -> String` - 先查 `ANTHROPIC_BASE_URL`，否则回退到常量
- `effective_model() -> &str`
- `effective_max_tokens() -> u32`

**`PermissionMode` 枚举：**
- `Default` - 自动允许只读操作
- `AcceptEdits` - 自动允许所有编辑
- `BypassPermissions` - 不提示，直接允许一切
- `Plan` - 只读规划模式

**`OutputFormat` 枚举：** `Text`, `Json`, `StreamJson`

**`HookEvent` 枚举：** `PreToolUse`, `PostToolUse`, `Stop`, `UserPromptSubmit`, `Notification`

**`HookEntry` 结构体：** `{ command: String, tool_filter: Option<String>, blocking: bool }`

**`McpServerConfig` 结构体：** `{ name: String, command: String, args: Vec<String>, env: HashMap<String, String>, url: Option<String>, server_type: McpServerType }`

**`Settings` 结构体** - 持久化的用户偏好，保存在 `~/.claude/settings.json`：
- `async fn load() -> Result<Settings>` - 反序列化 JSON，文件缺失时返回默认值
- `async fn save(&self) -> Result<()>` - 序列化为 JSON，并创建父目录

### 模块：`constants`

所有常量都是 `pub const`：

| 常量 | 值 |
|---|---|
| `APP_NAME` | `"claude"` |
| `DEFAULT_MODEL` | `"claude-opus-4-6"` |
| `SONNET_MODEL` | `"claude-sonnet-4-6"` |
| `HAIKU_MODEL` | `"claude-haiku-4-5-20251001"` |
| `DEFAULT_MAX_TOKENS` | `32_000` |
| `MAX_TOKENS_HARD_LIMIT` | `65_536` |
| `DEFAULT_COMPACT_THRESHOLD` | `0.9` |
| `MAX_TURNS_DEFAULT` | `10` |
| `ANTHROPIC_API_BASE` | `"https://api.anthropic.com"` |
| `ANTHROPIC_API_VERSION` | `"2023-06-01"` |
| `ANTHROPIC_BETA_HEADER` | `"interleaved-thinking-2025-05-14,token-efficient-tools-2025-02-19,files-api-2025-04-14"` |
| `CLAUDE_MD_FILENAME` | `"CLAUDE.md"` |
| `SETTINGS_FILENAME` | `"settings.json"` |
| `HISTORY_FILENAME` | `"history.json"` |
| `CONFIG_DIR_NAME` | `".claude"` |

**工具名常量：**
- `TOOL_NAME_BASH = "Bash"`
- `TOOL_NAME_FILE_EDIT = "Edit"`
- `TOOL_NAME_FILE_READ = "Read"`
- `TOOL_NAME_FILE_WRITE = "Write"`
- `TOOL_NAME_GLOB = "Glob"`
- `TOOL_NAME_GREP = "Grep"`
- `TOOL_NAME_WEB_FETCH = "WebFetch"`
- `TOOL_NAME_WEB_SEARCH = "WebSearch"`
- `TOOL_NAME_NOTEBOOK_EDIT = "NotebookEdit"`
- `TOOL_NAME_AGENT = "Task"`（子智能体）
- `TOOL_NAME_TODO_WRITE = "TodoWrite"`
- `TOOL_NAME_ASK_USER = "AskUserQuestion"`
- `TOOL_NAME_ENTER_PLAN_MODE = "EnterPlanMode"`
- `TOOL_NAME_EXIT_PLAN_MODE = "ExitPlanMode"`
- `TOOL_NAME_POWERSHELL = "PowerShell"`
- `TOOL_NAME_SLEEP = "Sleep"`
- `TOOL_NAME_CRON_CREATE = "CronCreate"`
- `TOOL_NAME_CRON_DELETE = "CronDelete"`
- `TOOL_NAME_CRON_LIST = "CronList"`
- `TOOL_NAME_ENTER_WORKTREE = "EnterWorktree"`
- `TOOL_NAME_EXIT_WORKTREE = "ExitWorktree"`
- `TOOL_NAME_LIST_MCP_RESOURCES = "ListMcpResources"`
- `TOOL_NAME_READ_MCP_RESOURCE = "ReadMcpResource"`
- `TOOL_NAME_TOOL_SEARCH = "ToolSearch"`
- `TOOL_NAME_BRIEF = "Brief"`
- `TOOL_NAME_CONFIG = "Config"`
- `TOOL_NAME_SEND_MESSAGE = "SendMessage"`
- `TOOL_NAME_SKILL = "Skill"`

### 模块：`context`

**`ContextBuilder`** - 构建注入到 system prompt 的系统上下文字符串：

- `build_system_context(working_dir: &Path) -> String`
  - 平台（操作系统 + 架构）
  - 当前工作目录
  - Git 状态（运行 `git status --short`）
  - 最近 5 条 git commit（运行 `git log --oneline -5`）

- `build_user_context(working_dir: &Path, no_claude_md: bool) -> String`
  - 当前日期/时间（来自 `chrono::Local::now()`）
  - `CLAUDE.md` 发现：从 `working_dir` 一直向上走到文件系统根目录，收集途中所有 `CLAUDE.md` 文件；同时读取 `~/.claude/CLAUDE.md`
  - 返回所有发现的 `CLAUDE.md` 内容拼接后的结果

### 模块：`permissions`

**`PermissionDecision` 枚举：** `Allow`, `AllowPermanently`, `Deny`, `DenyPermanently`

**`PermissionRequest` 结构体：** `{ tool_name: String, description: String, details: Option<String>, is_read_only: bool }`

**`PermissionHandler` trait：**
- `check_permission(&self, tool_name: &str) -> PermissionDecision`
- `request_permission(&self, request: &PermissionRequest) -> PermissionDecision`

**`AutoPermissionHandler`** - 自动的非交互式处理器：
- `BypassPermissions` → 允许所有请求
- `AcceptEdits` → 允许所有请求
- `Plan` → 只有 `is_read_only == true` 时允许，否则拒绝
- `Default` → 只有 `is_read_only == true` 时允许，否则拒绝

### 模块：`history`

**`ConversationSession` 结构体：**
```
id: String (UUID v4)
created_at: DateTime<Utc>
updated_at: DateTime<Utc>
messages: Vec<Message>
model: String
title: Option<String>
working_dir: String
```

**函数：**
- `save_session(session: &ConversationSession) -> Result<()>` - 写入 `~/.claude/conversations/<id>.json`
- `load_session(id: &str) -> Result<Option<ConversationSession>>` - 从上面的路径读取
- `list_sessions() -> Result<Vec<ConversationSession>>` - 读取 `~/.claude/conversations/` 下所有 `.json` 文件，按 `updated_at` 降序排序
- `delete_session(id: &str) -> Result<()>` - 删除文件

### 模块：`cost`

**`ModelPricing` 结构体：** `{ input_per_mtok: f64, output_per_mtok: f64, cache_creation_per_mtok: f64, cache_read_per_mtok: f64 }`

**价格常量：**
| 模型 | 输入（$/MTok） | 输出（$/MTok） |
|---|---|---|
| `OPUS` | $15.00 | $75.00 |
| `SONNET` | $3.00 | $15.00 |
| `HAIKU` | $0.80 | $4.00 |

**`CostTracker` 结构体** - 使用 `AtomicU64` 的无锁实现：
- `input_tokens: AtomicU64`
- `output_tokens: AtomicU64`
- `cache_creation_tokens: AtomicU64`
- `cache_read_tokens: AtomicU64`
- `add_usage(input, output, cache_creation, cache_read)` - 原子累加
- `total_cost_usd(model: &str) -> f64` - 读取原子值，并按模型子串匹配查价格
- `summary(model: &str) -> String` - 人类可读的成本 + token 数
- 实现 `Default`

### 模块：`hooks`

**`HookContext` 结构体：** `{ event: String, tool_name: Option<String>, tool_input: Option<Value>, tool_output: Option<String>, is_error: Option<bool>, session_id: Option<String> }`

**`HookOutcome` 枚举：** `Allowed`, `Blocked(String)`, `Modified(Value)`

**`run_hooks(hooks, event, context, working_dir) -> HookOutcome`**（async）：
- 遍历对应 `HookEvent` 的 `Vec<HookEntry>`
- 按 `tool_filter` 过滤（对 `tool_name` 做 glob 匹配）
- 通过 `tokio::process::Command` 启动 shell 命令
- 将 `HookContext` 作为 JSON 写到 stdin
- 如果 `blocking: true` 且退出码 != 0，返回 `HookOutcome::Blocked(stderr)`
- 否则返回 `HookOutcome::Allowed`

### 与 TypeScript 的关系

`cc-core` 对应分散在 TypeScript 中的这些文件：`src/constants/`、`src/context.ts`、`src/history.ts`、`src/cost-tracker.ts`、`src/costHook.ts`、`src/schemas/hooks.ts`，以及 `src/services/api/` 的一部分。权限模式、hook 事件和配置结构都与 TypeScript 的 `Config` 类型一一对应。

---

## 代码包：`cc-api`

**路径：** `crates/api/src/lib.rs`

完整的 async Messages API 客户端，支持 SSE 流。

### 模块：`types`

**`CreateMessageRequest` 结构体：** 通过 `CreateMessageRequestBuilder` 构建：
- `model: String`
- `max_tokens: u32`
- `messages: Vec<ApiMessage>`
- `system: Option<SystemPrompt>`
- `tools: Option<Vec<ApiToolDefinition>>`
- `temperature: Option<f32>`
- `top_p: Option<f32>`
- `top_k: Option<u32>`
- `stop_sequences: Option<Vec<String>>`
- `thinking: Option<ThinkingConfig>`
- `stream: bool`（内部始终设为 `true`）

**`CreateMessageRequestBuilder`** - 流式 builder：
- `CreateMessageRequest::builder(model, max_tokens) -> Self`
- `.messages(Vec<ApiMessage>)`
- `.system(SystemPrompt)` 或 `.system_text(String)`
- `.tools(Vec<ApiToolDefinition>)`
- `.temperature(f32)`, `.top_p(f32)`, `.top_k(u32)`
- `.stop_sequences(Vec<String>)`
- `.thinking(ThinkingConfig)`
- `.build() -> CreateMessageRequest`

**`ThinkingConfig`：** `{ type: "enabled", budget_tokens: u32 }`
- `ThinkingConfig::enabled(budget: u32) -> Self`

**`SystemPrompt` 枚举**（serde untagged）：
- `Text(String)` - 简单文本 system prompt
- `Blocks(Vec<SystemBlock>)` - 带缓存控制的结构化块

**`SystemBlock`：** `{ type: "text", text: String, cache_control: Option<CacheControl> }`

**`CacheControl`：** `{ type: "ephemeral" }`
- `CacheControl::ephemeral() -> Self`

**`ApiMessage`：** `{ role: String, content: Value }`
- `From<&Message> for ApiMessage` - 把 `cc_core::types::Message` 转成 API 格式

**`ApiToolDefinition`：** `{ name: String, description: String, input_schema: Value, cache_control: Option<CacheControl> }`
- `From<&ToolDefinition> for ApiToolDefinition`
- 列表中的最后一个工具会带上 `cache_control: Some(CacheControl::ephemeral())`（prompt caching）

**`CreateMessageResponse`：** `{ id, type, role, content, model, stop_reason, stop_sequence, usage }`

**`ApiErrorResponse`：** `{ type: String, error: ApiErrorDetail }`

### 模块：`streaming`

**`StreamEvent` 枚举**（serde `#[serde(tag = "type")]`）：
- `MessageStart { message: CreateMessageResponse }`
- `MessageDelta { delta: MessageDeltaData, usage: Option<StreamUsage> }`
- `MessageStop`
- `ContentBlockStart { index: usize, content_block: PartialContentBlock }`
- `ContentBlockDelta { index: usize, delta: ContentDelta }`
- `ContentBlockStop { index: usize }`
- `Ping`
- `Error { error_type: String, message: String }`

**`ContentDelta` 枚举：**
- `TextDelta { text: String }`
- `InputJsonDelta { partial_json: String }`
- `ThinkingDelta { thinking: String }`
- `SignatureDelta { signature: String }`

**`StreamHandler` trait：**
- `fn on_event(&self, event: &StreamEvent)` - 每个 SSE 事件都会回调

**`NullStreamHandler`** - 无头模式下的空实现

**`StreamAccumulator`** - 把流事件收集成完整消息：
- `on_event(&mut self, event: &StreamEvent)` - 处理所有事件类型
- `finish(self) -> (Message, UsageInfo, Option<String>)` - 返回 `(assistant_message, usage, stop_reason)`

累积过程中使用的内部 `PartialBlock` 枚举：
- `Text(String)`
- `ToolUse { id: String, name: String, json_buf: String }`
- `Thinking { thinking_buf: String, signature_buf: String }`

### 模块：`sse_parser`

**`SseFrame` 结构体：** `{ event: Option<String>, data: Option<String> }`

**`SseLineParser`** - 有状态的逐行 SSE 解析器：
- `feed_line(&mut self, line: &str) -> Option<SseFrame>`
- 按 SSE 规范处理 `event:`、`data:` 和空行边界

### 模块：`client`

**`ClientConfig` 结构体：**
- `api_key: String`
- `api_base: String`（默认：`ANTHROPIC_API_BASE`）
- `timeout_secs: u64`（默认：600）
- `max_retries: u32`（默认：5）

**`AnthropicClient` 结构体：**
- `AnthropicClient::new(config: ClientConfig) -> Result<Self>` - 校验 API key，构建带 `rustls-tls` 的 `reqwest::Client`，设置 `anthropic-version` 和 `anthropic-beta` 头
- `AnthropicClient::from_config(cfg: &Config) -> Result<Self>` - 从 Config 解析 key/base

**`create_message(request) -> Result<CreateMessageResponse>`** - 非流式 POST 到 `/v1/messages`

**`create_message_stream(request, handler) -> Result<mpsc::Receiver<StreamEvent>>`**（async）：
1. 把 request 的 `stream` 设为 `true`
2. 启动 `tokio::spawn` 后台任务调用 `process_sse_stream()`
3. 返回带 256 缓冲区的 `mpsc::Receiver<StreamEvent>`
4. 后台任务通过 `SseLineParser` 逐行读取响应体
5. 调用 `frame_to_event()` 解析每个 frame
6. 发送到 channel，并调用 `handler.on_event()`

**`send_with_retry(request_fn) -> Result<reqwest::Response>`** - 指数退避：
- 最多重试 5 次
- 初始延迟：1 秒
- 每次重试乘以 2，上限 60 秒
- 遵守 `Retry-After` 响应头（覆盖退避延迟）
- 对 429（`RateLimit`）和 529（`ApiStatus` overloaded）重试

**`frame_to_event(frame: SseFrame) -> Option<StreamEvent>`** - 按 `frame.event` 分发：
- `"ping"` → `StreamEvent::Ping`
- `"message_start"` → 将 `data` 反序列化为 `StreamEvent::MessageStart`
- `"content_block_start"` → `StreamEvent::ContentBlockStart`
- `"content_block_delta"` → `StreamEvent::ContentBlockDelta`
- `"content_block_stop"` → `StreamEvent::ContentBlockStop`
- `"message_delta"` → `StreamEvent::MessageDelta`
- `"message_stop"` → `StreamEvent::MessageStop`
- `"error"` → `StreamEvent::Error`

### 与 TypeScript 的关系

对应 `src/services/api/claude.ts`、`src/services/api/client.ts` 和 `src/services/api/errorUtils.ts`。实现了相同的 SSE 流协议和重试逻辑。`CacheControl::ephemeral()` 的 prompt caching 与 TypeScript 版本一致，beta header 也完全相同。

---

## 代码包：`cc-tools`

**路径：** `crates/tools/src/`

实现全部 33 个内置工具。每个工具都是一个实现了 `Tool` trait 的零大小结构体。

### 核心类型（`lib.rs`）

**`ToolResult` 结构体：**
- `content: String`
- `is_error: bool`
- `metadata: Option<Value>` - 给 TUI 渲染用的可选结构化数据
- `ToolResult::success(content)` / `ToolResult::error(content)` / `.with_metadata(meta)`

**`PermissionLevel` 枚举：** `None`, `ReadOnly`, `Write`, `Execute`, `Dangerous`

**`ToolContext` 结构体：**
- `working_dir: PathBuf`
- `permission_mode: PermissionMode`
- `permission_handler: Arc<dyn PermissionHandler>`
- `cost_tracker: Arc<CostTracker>`
- `session_id: String`
- `non_interactive: bool`
- `mcp_manager: Option<Arc<cc_mcp::McpManager>>`
- `config: cc_core::config::Config`
- `resolve_path(&self, path: &str) -> PathBuf` - 把相对路径解析到 `working_dir`
- `check_permission(tool_name, description, is_read_only) -> Result<(), ClaudeError>`

**`Tool` trait**（async_trait）：
- `fn name(&self) -> &str`
- `fn description(&self) -> &str`
- `fn permission_level(&self) -> PermissionLevel`
- `fn input_schema(&self) -> Value` - 工具参数的 JSON Schema
- `async fn execute(&self, input: Value, ctx: &ToolContext) -> ToolResult`
- `fn to_definition(&self) -> ToolDefinition` - 使用上面的方法生成默认实现

**`all_tools() -> Vec<Box<dyn Tool>>`** - 返回全部 33 个工具

**`find_tool(name: &str) -> Option<Box<dyn Tool>>`** - 按精确名称查找

### 工具：`BashTool`（`bash.rs`）

**名称：** `"Bash"`
**权限级别：** `Execute`

输入 schema：`{ command: string, timeout: optional u64 (seconds) }`

**算法：**
1. 通过 `ctx.check_permission()` 校验权限
2. Windows 下执行 `cmd /C <command>`；Unix 下执行 `bash -c <command>`
3. 默认超时：120 秒；最大超时：600 秒
4. 使用 `tokio::io::BufReader` 收集 stdout 和 stderr
5. 输出超过 100,000 字符时截断并提示
6. 非 0 退出码 → `ToolResult::error`，包含 stdout+stderr+exit_code
7. 退出码为 0 → `ToolResult::success`，返回 stdout（stderr 非空时附加）

### 工具：`FileReadTool`（`file_read.rs`）

**名称：** `"Read"`
**权限级别：** `ReadOnly`

输入 schema：`{ file_path: string, offset: optional u32 (1-based line), limit: optional u32 }`

**算法：**
1. 通过 `ctx.resolve_path()` 解析路径
2. 默认 limit：2000 行
3. 读取整个文件并按换行拆分
4. 应用 offset（从 1 开始）和 limit
5. 输出格式：`{line_number}\t{content}`
6. 二进制文件返回错误（通过 `std::io::ErrorKind::InvalidData` 检测）
7. 图片和 PDF 返回 stub 消息

### 工具：`FileEditTool`（`file_edit.rs`）

**名称：** `"Edit"`
**权限级别：** `Write`

输入 schema：`{ file_path: string, old_string: string, new_string: string, replace_all: optional bool }`

**算法：**
1. 校验 `old_string != new_string`
2. 读取当前文件内容
3. 统计 `old_string` 的出现次数
4. 如果 `replace_all == false`（默认）且次数 > 1：返回错误（有歧义）
5. 如果 `replace_all == true`：使用 `str::replace()`（替换全部）
6. 如果 `replace_all == false` 且次数 == 1：使用 `str::replacen(old, new, 1)`
7. 把更新后的内容写回文件

### 工具：`FileWriteTool`（`file_write.rs`）

**名称：** `"Write"`
**权限级别：** `Write`

输入 schema：`{ file_path: string, content: string }`

**算法：**
1. 解析路径
2. 通过 `tokio::fs::create_dir_all()` 创建父目录
3. 把内容写入文件
4. 在成功消息里报告行数和字节数

### 工具：`GlobTool`（`glob_tool.rs`）

**名称：** `"Glob"`
**权限级别：** `ReadOnly`

输入 schema：`{ pattern: string, path: optional string }`

**算法：**
1. 解析基础路径（默认 `working_dir`）
2. 组合 base + pattern 构造完整 glob 模式
3. 使用 `glob::glob()` 做模式匹配
4. 按修改时间排序（最新优先）
5. 最多返回 250 条结果
6. 返回以换行分隔的相对路径列表

### 工具：`GrepTool`（`grep_tool.rs`）

**名称：** `"Grep"`
**权限级别：** `ReadOnly`

输入 schema：`{ pattern: string, path: optional string, glob: optional string, type: optional string, output_mode: optional enum, context: optional u32, head_limit: optional u32, offset: optional u32, -i: optional bool, -n: optional bool, -A: optional u32, -B: optional u32, -C: optional u32, multiline: optional bool }`

**算法：**
1. 使用 `RegexBuilder`，开启 `case_insensitive` 和 `multi_line`
2. 使用 `walkdir::WalkDir` 遍历目录树
3. 跳过隐藏目录、`node_modules/`、`target/`、`__pycache__/`、`.git/`
4. 按 glob 模式或文件类型扩展名映射过滤
5. 三种输出模式：
   - `files_with_matches` - 文件路径列表（默认）
   - `content` - 匹配行及可选上下文（-A/-B/-C）
   - `count` - 每个文件的匹配次数
6. 支持 `head_limit` 和 `offset` 分页

**类型快捷方式**（例如 `type="js"` → `["js", "jsx", "mjs", "cjs"]`）：
- `js`, `ts`, `py`, `rs`, `go`, `java`, `rb`, `cpp`, `c`, `cs`, `php`, `swift`, `kt`, `html`, `css`, `json`, `yaml`, `md`

### 工具：`WebFetchTool`（`web_fetch.rs`）

**名称：** `"WebFetch"`
**权限级别：** `ReadOnly`

输入 schema：`{ url: string, prompt: optional string }`

**算法：**
1. `reqwest` GET，请求超时 30 秒，重定向上限 10 次
2. User-Agent：`"Claude-Code/1.0"`
3. 如果是 HTML content-type：运行 `strip_html()`，这是一个手写状态机，用于去除标签、脚本和样式，并把 `&amp;`、`&lt;`、`&gt;`、`&nbsp;` 实体转回文本
4. 内容超过 100,000 字符时截断
5. 返回文本内容

### 工具：`WebSearchTool`（`web_search.rs`）

**名称：** `"WebSearch"`
**权限级别：** `ReadOnly`

输入 schema：`{ query: string, num_results: optional u32 (default 5) }`

**算法：**
1. 检查 `BRAVE_SEARCH_API_KEY` 环境变量：
   - 如果存在：调用 Brave Search API `https://api.search.brave.com/res/v1/web/search`
   - 为每个结果返回 title + URL + description
2. 回退：DuckDuckGo Instant Answer API `https://api.duckduckgo.com/?q=...&format=json`
3. 最多返回 `num_results` 条格式化结果

### 工具：`NotebookEditTool`（`notebook_edit.rs`）

**名称：** `"NotebookEdit"`
**权限级别：** `Write`

输入 schema：`{ notebook_path: string, cell_id: optional string, cell_index: optional u32, source: optional string, cell_type: optional string, mode: string (replace|insert|delete) }`

**算法：**
1. 使用 `serde_json` 解析 `.ipynb` JSON
2. 单元查找：通过 UUID 字符串，或 `cell-N` 索引模式
3. **replace 模式：** 更新 `source`，把 `outputs = []`、`execution_count = null`
4. **insert 模式：** 在指定索引或 `cell_id` 之后插入新单元；通过 `timestamp XOR random` 生成 8 位十六进制单元 ID
5. **delete 模式：** 按 id/index 删除单元
6. 把更新后的 notebook 写回文件

### 工具：`TaskCreateTool`, `TaskGetTool`, `TaskUpdateTool`, `TaskListTool`, `TaskStopTool`, `TaskOutputTool`（`tasks.rs`）

全局存储：`TASK_STORE: Lazy<Arc<DashMap<String, Task>>>`

**`Task` 结构体：** `{ id: String, subject: String, description: String, status: TaskStatus, owner: Option<String>, blocks: Vec<String>, blocked_by: Vec<String>, metadata: HashMap<String, Value>, output: Vec<String>, created_at: DateTime<Utc>, updated_at: DateTime<Utc> }`

**`TaskStatus` 枚举：** `Pending`, `InProgress`, `Completed`, `Deleted`, `Running`, `Failed`

| 工具 | 名称 | 说明 |
|---|---|---|
| `TaskCreateTool` | `"TaskCreate"` | 创建任务，使用 UUID，并存入 `TASK_STORE` |
| `TaskGetTool` | `"TaskGet"` | 按 ID 返回任务 JSON |
| `TaskUpdateTool` | `"TaskUpdate"` | 更新任务字段；`status=deleted` 时从存储中移除 |
| `TaskListTool` | `"TaskList"` | 列出所有未删除任务，可选状态过滤 |
| `TaskStopTool` | `"TaskStop"` | 把任务状态设为 `Failed` |
| `TaskOutputTool` | `"TaskOutput"` | 向任务的 `output` 向量追加文本 |

### 工具：`CronCreateTool`, `CronDeleteTool`, `CronListTool`（`cron.rs`）

全局存储：`CRON_STORE: Lazy<Arc<RwLock<HashMap<String, CronTask>>>>`

**`CronTask` 结构体：** `{ id: String, cron: String, prompt: String, recurring: bool, durable: bool, created_at: DateTime<Utc> }`

| 工具 | 名称 | 说明 |
|---|---|---|
| `CronCreateTool` | `"CronCreate"` | 创建计划任务；校验 cron 表达式；如果 `durable=true`，持久化到 `.claude/scheduled_tasks.json`；最多 50 个任务 |
| `CronDeleteTool` | `"CronDelete"` | 按 ID 从存储中移除任务（durable 时也会删磁盘数据） |
| `CronListTool` | `"CronList"` | 以自然语言形式列出所有计划任务 |

**`cron_matches(cron: &str, now: &DateTime<Local>) -> bool`：**
- 解析 5 段 cron：minute、hour、day-of-month、month、day-of-week
- 支持：`*`、`*/N`（步进）、`N-M`（范围）、`N,M,...`（列表）

**`validate_cron(cron: &str) -> Result<()>`** - 校验字段范围（minute 0–59，hour 0–23，等等）

**`cron_to_human(cron: &str) -> String`** - 用自然语言描述调度

**`pop_due_tasks() -> Vec<CronTask>`** - 返回当前时间匹配的任务，并从存储中移除非 recurring 任务

### 工具：`TodoWriteTool`（`todo_write.rs`）

**名称：** `"TodoWrite"`
**权限级别：** `None`

输入 schema：`{ todos: Array<{ id: string, content: string, status: string, priority: string }> }`

替换整个 todo 列表。返回包含 pending/in_progress/completed 计数的摘要。

### 工具：`AskUserQuestionTool`（`ask_user.rs`）

**名称：** `"AskUserQuestion"`
**权限级别：** `None`

输入 schema：`{ question: string, options: optional Array<string> }`

在 `non_interactive` 模式下：返回错误 `"Cannot prompt user in non-interactive mode"`。
否则：返回 `ToolResult::success("")`，并附带 metadata `{ type: "ask_user", question, options }`，交给 TUI 层处理。

### 工具：`EnterPlanModeTool`（`enter_plan_mode.rs`）

**名称：** `"EnterPlanMode"`
**权限级别：** `None`

返回带 metadata `{ type: "enter_plan_mode" }` 的 `ToolResult::success`。表示会话切换到 Plan 权限模式。

### 工具：`ExitPlanModeTool`（`exit_plan_mode.rs`）

**名称：** `"ExitPlanMode"`
**权限级别：** `None`

输入 schema：`{ summary: optional string }`

返回带 metadata `{ type: "exit_plan_mode", summary }` 的成功结果。表示退出 Plan 模式。

### 工具：`PowerShellTool`（`powershell.rs`）

**名称：** `"PowerShell"`
**权限级别：** `Execute`

输入 schema：`{ command: string, timeout: optional u64 }`

执行模式与 `BashTool` 相同。Windows 下使用 `powershell -NoProfile -NonInteractive -Command`；其他平台下使用 `pwsh`。

### 工具：`EnterWorktreeTool`, `ExitWorktreeTool`（`worktree.rs`）

全局：`WORKTREE_SESSION: Lazy<Arc<RwLock<Option<WorktreeSession>>>>`

**`WorktreeSession`：** `{ branch: String, path: PathBuf, original_dir: PathBuf }`

**`EnterWorktreeTool`**（`"EnterWorktree"`）：
- 输入：`{ branch: string, path: optional string }`
- 运行 `git worktree add -b <branch> <path>`
- 把会话保存到 `WORKTREE_SESSION`

**`ExitWorktreeTool`**（`"ExitWorktree"`）：
- 输入：`{ action: "keep" | "remove", discard_changes: optional bool }`
- `keep`：锁定 worktree，清空会话
- `remove`：检查未提交变更（需要 `discard_changes=true` 才能覆盖），运行 `git worktree remove --force <path>`，然后 `git branch -D <branch>`

### 工具：`SendMessageTool`（`send_message.rs`）

**名称：** `"SendMessage"`
**权限级别：** `None`

全局：`INBOX: Lazy<DashMap<String, Vec<AgentMessage>>>`

输入 schema：`{ to: string, message: string, metadata: optional Value }`

- 向 `INBOX` 中名为 `to` 的收件人发送消息
- `to = "*"` 时广播给所有现有 key
- `drain_inbox(recipient: &str) -> Vec<AgentMessage>` - 删除并返回所有消息
- `peek_inbox(recipient: &str) -> Vec<AgentMessage>` - 仅查看，不删除

### 工具：`SkillTool`（`skill_tool.rs`）

**名称：** `"Skill"`
**权限级别：** `None`

输入 schema：`{ skill: string, arguments: optional string }`

**算法：**
1. `skill = "list"` → 枚举 `.claude/commands/*.md` 和 `~/.claude/commands/*.md`，从 YAML frontmatter 或第一个标题中提取描述
2. 否则：先从项目命令目录，再从用户命令目录解析 `<skill>.md`
3. 去掉 YAML frontmatter（`---` 块）
4. 用传入的 arguments 字符串替换 `$ARGUMENTS`
5. 以 `ToolResult::success` 返回文件内容

### 工具：`SleepTool`（`sleep.rs`）

**名称：** `"Sleep"`
**权限级别：** `None`

输入 schema：`{ duration: f64 (seconds) }`

调用 `tokio::time::sleep(Duration::from_secs_f64(duration))`。最大 300 秒。

### 工具：`ToolSearchTool`（`tool_search.rs`）

**名称：** `"ToolSearch"`
**权限级别：** `None`

输入 schema：`{ query: string, max_results: optional u32 (default 5) }`

静态 `TOOL_CATALOG: &[(&str, &str, &[&str])]` - 32 条 `(name, description, keywords)` 记录。

**评分算法：**
- `select:Name` 语法 → 精确名称匹配得分 100
- 否则对每个目录项：
  - 名称精确匹配：+20
  - 名称包含 query：+10
  - 描述包含 query：+5
  - 关键词精确匹配：+8
  - 关键词包含 query：+3
- 返回得分大于 0 的前 `max_results` 项

### 工具：`BriefTool`（`brief.rs`）

**名称：** `"Brief"`
**权限级别：** `None`

输入 schema：`{ message: string, status: optional string, attachments: optional Array<string> (file paths) }`

解析附件元数据（文件大小、是否为图片，依据扩展名判断）。返回 `ToolResult::success("")`，metadata 为 `{ message, status, sentAt, attachments: [{ path, size, isImage }] }`。

### 工具：`ConfigTool`（`config_tool.rs`）

**名称：** `"Config"`
**权限级别：** `None`

输入 schema：`{ action: "get" | "set", key: string, value: optional Value }`

读写 `~/.claude/settings.json`。支持的 key：`model`、`max_tokens`、`verbose`、`permission_mode`、`auto_compact`。`get` 时返回当前值，`set` 时写入并确认。

### 工具：`ListMcpResourcesTool`, `ReadMcpResourceTool`（`mcp_resources.rs`）

| 工具 | 名称 | 说明 |
|---|---|---|
| `ListMcpResourcesTool` | `"ListMcpResources"` | 调用 `ctx.mcp_manager.list_all_resources()`，返回 JSON |
| `ReadMcpResourceTool` | `"ReadMcpResource"` | 输入：`{ uri: string }`。调用 `ctx.mcp_manager.read_resource(uri)` |

如果 `ctx.mcp_manager` 为 `None`，两个工具都会返回错误。

### 与 TypeScript 的关系

`cc-tools` 对应 TypeScript 中 `src/` 里的工具实现（例如 bash 在工具系统里，文件操作在 ReadTool/EditTool/WriteTool 中等）。工具名与 TypeScript 常量完全一致，`ToolContext` 对应 TypeScript 的 `ToolUseContext`。

---

## 代码包：`cc-query`

**路径：** `crates/query/src/`

核心的 agentic query loop crate。包含 4 个源文件。

### 模块：`lib.rs` - 主查询循环

**`QueryOutcome` 枚举：**
- `EndTurn { message: Message, usage: UsageInfo }` - 模型发出 `end_turn`
- `MaxTokens { partial_message: Message, usage: UsageInfo }` - 达到 token 限制
- `Cancelled` - 取消 token 被触发
- `Error(ClaudeError)` - 不可恢复错误

**`QueryConfig` 结构体：**
- `model: String`
- `max_tokens: u32`
- `max_turns: u32`（默认：`MAX_TURNS_DEFAULT = 10`）
- `system_prompt: Option<String>`
- `append_system_prompt: Option<String>`
- `thinking_budget: Option<u32>`
- `temperature: Option<f32>`
- `QueryConfig::default()` 使用 `DEFAULT_MODEL` + `DEFAULT_MAX_TOKENS`
- `QueryConfig::from_config(cfg: &Config)` - 从 Config 读取 model + max_tokens

**`QueryEvent` 枚举：**
- `Stream(StreamEvent)` - 原始 API 流事件
- `ToolStart { tool_name, tool_id }` - 工具开始执行
- `ToolEnd { tool_name, tool_id, result, is_error }` - 工具完成
- `TurnComplete { turn: u32, stop_reason: String }` - 模型轮次结束
- `Status(String)` - 信息提示
- `Error(String)` - 错误通知

**`run_query_loop(client, messages, tools, tool_ctx, config, cost_tracker, event_tx, cancel_token) -> QueryOutcome`**（async）：

主 agentic 循环：
1. 轮次计数加 1；如果 `> max_turns`，返回 `EndTurn`
2. 检查 `cancel_token.is_cancelled()` → `Cancelled`
3. 把 `messages` 转成 `Vec<ApiMessage>`，把 tools 转成 `Vec<ApiToolDefinition>`
4. 调用 `build_system_prompt(config)` 构建 `SystemPrompt`
5. 构建 `CreateMessageRequest`（如果提供了 `budget`，则带 thinking 配置）
6. 创建 `ChannelStreamHandler` 或 `NullStreamHandler`
7. 调用 `client.create_message_stream()`，接收 `mpsc::Receiver<StreamEvent>`
8. 内层循环：`tokio::select!` 监听取消或流事件；把事件喂给 `StreamAccumulator`
9. 在 `MessageStop` 或 channel 关闭时调用 `accumulator.finish()`
10. 通过 `cost_tracker.add_usage()` 统计成本
11. 把 assistant 消息追加到 `messages`
12. 如果 stop reason 是 `end_turn` 或 `tool_use`，调用 `auto_compact_if_needed()`
13. 根据 `stop_reason` 分发：
    - `"end_turn"` / `"stop_sequence"` / 未知 → 触发 `Stop` hook → 返回 `EndTurn`
    - `"max_tokens"` → 返回 `MaxTokens`
    - `"tool_use"` → 执行所有 tool_use blocks（见下），追加结果，然后 `continue`

**`tool_use` 轮次中的工具执行：**
1. 对每个 `ContentBlock::ToolUse { id, name, input }`：
2. 发出 `QueryEvent::ToolStart`
3. 通过 `cc_core::hooks::run_hooks()` 触发 `PreToolUse` hooks；如果 `HookOutcome::Blocked` → `ToolResult::error("Blocked by hook: ...")`
4. 否则调用 `execute_tool(name, input, tools, ctx)`
5. 触发 `PostToolUse` hooks
6. 发出 `QueryEvent::ToolEnd`
7. 把 `ContentBlock::ToolResult` 放进 `result_blocks`
8. 用 `Message::user_blocks(result_blocks)` 追加到对话

**`execute_tool(name, input, tools, ctx) -> ToolResult`**（async）：
- 在 slice 中按名称查找工具，调用 `tool.execute(input, ctx)`
- 未知工具 → `ToolResult::error("Unknown tool: {name}")`

**`build_system_prompt(config) -> SystemPrompt`：**
- 用 `\n\n` 连接 `system_prompt` 和 `append_system_prompt`
- 为空时回退到默认 `"You are Claude, an AI assistant by Anthropic."`

**`run_single_query(client, messages, config) -> Result<Message>`**（async）：
- 单次 API 调用，不走工具循环，使用 `NullStreamHandler`
- 返回完整的 assistant message

**`ChannelStreamHandler`** - 实现 `StreamHandler`：
- `on_event(&self, event)` 转发到 `mpsc::UnboundedSender<QueryEvent>`

### 模块：`compact.rs` - 自动压缩

**常量：**
- `AUTOCOMPACT_BUFFER_TOKENS = 13_000`
- `WARNING_THRESHOLD_BUFFER_TOKENS = 20_000`
- `AUTOCOMPACT_TRIGGER_FRACTION = 0.90`
- `KEEP_RECENT_MESSAGES = 10`
- `MAX_CONSECUTIVE_FAILURES = 3`

**`AutoCompactState` 结构体：**
- `compaction_count: u32`
- `consecutive_failures: u32`
- `disabled: bool` - 熔断器；连续失败 3 次后置位

**`TokenWarningState` 枚举：** `Ok`, `Warning`, `Critical`

**`context_window_for_model(model: &str) -> u32`：**
- 对匹配 `"opus-4"`、`"sonnet-4"`、`"haiku-4"`、`"claude-3-5"` 的模型返回 `200_000`
- 其他返回 `100_000`

**`calculate_token_warning_state(input_tokens, model) -> TokenWarningState`：**
- 使用 `WARNING_THRESHOLD_BUFFER_TOKENS` 决定 Warning 与 Critical

**`should_auto_compact(state, input_tokens, model) -> bool`：**
- 如果 `state.disabled`，返回 false
- 如果 `input_tokens / context_window > AUTOCOMPACT_TRIGGER_FRACTION`，返回 true

**`summarise_head(client, messages_to_summarize, model) -> Result<String>`**（async）：
- 调用 API，请求对提供的对话做摘要
- 返回包在 `<compact-summary>...</compact-summary>` XML 标签中的摘要

**`compact_conversation(client, messages, model) -> Result<Vec<Message>>`**（async）：
- 切分对话：head = `messages[0..total-KEEP_RECENT_MESSAGES]`，tail = 最后 10 条消息
- 对 head 调用 `summarise_head()`
- 返回 `[Message::user(summary)] + tail`

**`auto_compact_if_needed(client, messages, input_tokens, model, state) -> Option<Vec<Message>>`**（async）：
- 检查 `should_auto_compact()`，调用 `compact_conversation()`
- 成功：重置 `consecutive_failures`，增加 `compaction_count`
- 失败：增加 `consecutive_failures`；如果 `>= MAX_CONSECUTIVE_FAILURES`，则禁用
- 成功返回 `Some(new_messages)`，不需要或失败则返回 `None`

### 模块：`agent_tool.rs` - 子智能体工具

**`AgentTool`** 实现 `Tool`：
- **名称：** `"Task"`（常量 `TOOL_NAME_AGENT`）
- **权限级别：** `Execute`

输入 schema：`{ description: string, prompt: string, tools: optional Array<string>, system_prompt: optional string, max_turns: optional u32, model: optional string }`

**算法：**
1. 从 `ANTHROPIC_API_KEY` 环境变量创建专用 `AnthropicClient`
2. 过滤工具列表：如果提供了 `tools` 字段，就只用这个子集；始终排除 `TOOL_NAME_AGENT`（防止递归）
3. 调用 `run_query_loop()`，其中：
   - `event_tx = None`（子智能体不向 TUI 转发）
   - 使用新的 `ToolContext`，但工作目录、权限模式等保持一致
4. 把最终 assistant message 的文本作为 `ToolResult::success()` 返回

### 模块：`cron_scheduler.rs` - 后台 cron

**`start_cron_scheduler(tools, tool_ctx, cancel_token) -> JoinHandle<()>`：**
- 启动 `tokio::spawn(run_scheduler_loop(...))`

**`run_scheduler_loop(tools, tool_ctx, cancel_token)`**（async loop）：
1. 计算距离下一分钟边界还有多少秒：`sleep(60 - now.second() + 1)`
2. 调用 `cc_tools::cron::pop_due_tasks()` 获取匹配任务
3. 对每个到期任务：启动 `run_query_loop()`，其中：
   - 单条 user message 来自 `task.prompt`
   - `event_tx = None`（后台，无 UI）
   - 使用 `cancel_token` 的 clone
4. 循环持续，直到取消

### 与 TypeScript 的关系

`cc-query` 对应 `src/query.ts`、`src/query/`、`src/services/compact/autoCompact.ts`、`src/coordinator/`，以及 `src/services/autoDream/` 的一部分。`AgentTool` 对应 TypeScript 的 `Task` 工具。

---

## 代码包：`cc-tui`

**路径：** `crates/tui/src/lib.rs`

基于 `ratatui` + `crossterm` 的终端 UI，替代了 TypeScript 的 `ink`/React 渲染层。

### `App` 结构体

```
config: Config
cost_tracker: Arc<CostTracker>
messages: Vec<(Role, String)>
input: String
input_history: Vec<String>
history_index: Option<usize>
scroll_offset: u16
is_streaming: bool
streaming_text: String
status_message: Option<String>
should_quit: bool
show_help: bool
```

### 关键方法

**`handle_key_event(&mut self, key: KeyEvent) -> Option<String>`：**
- `Ctrl+C`：如果正在流式输出 → 取消；如果输入为空 → 退出；否则清空输入
- `Ctrl+D`：如果输入为空 → 退出
- 字符输入：追加到 `self.input`
- `Backspace`：删除 `self.input` 的最后一个字符
- `Enter`：返回 `Some(input)` 供调用方处理（空输入忽略）
- `Up`/`Down`：浏览 `input_history`
- `PageUp`/`PageDown`：调整 `scroll_offset`
- `F1` / `?`：切换帮助覆盖层

**`handle_query_event(&mut self, event: QueryEvent)`：**
- `Stream(ContentBlockDelta::TextDelta)` → 追加到 `streaming_text`
- `ToolStart { tool_name, .. }` → 设置 `status_message = "Running {tool_name}..."`
- `ToolEnd { .. }` → 清空 `status_message`
- `TurnComplete { .. }` → 把 `streaming_text` 放入 `messages`，清除 streaming 状态
- `Status(msg)` → 设置 `status_message`
- `Error(msg)` → 设置带错误前缀的 `status_message`

**`take_input(&mut self) -> String`：**
- 返回并清空 `self.input`
- 推入 `input_history`（头部去重）

**`add_message(&mut self, role: Role, text: String)`** - 追加到消息向量

### 模块：`render`

**`render_app(f: &mut Frame, app: &App)`：**
- 通过 `Layout::vertical` 把终端分成 3 个竖向区域：
  1. 消息区域（弹性填充）
  2. 输入区域（3 行）
  3. 状态栏（1 行）
- **消息区：** 渲染每个 `(role, text)` 对，`Role::User` 用 Cyan，`Role::Assistant` 用 Green。如果正在流式输出，把部分 `streaming_text` 以 Yellow italic 追加进去
- **输入区：** 边框 `Block`，标题为 `"Input"`；显示 `self.input`，并在末尾追加光标 `_`
- **状态栏：** 显示 `{model} | {cost_summary}`，颜色为 Dark Gray

### 模块：`widgets`

**`render_permission_dialog(f: &mut Frame, question: &str, options: &[String])`：**
- 居中的弹窗对话框
- 显示问题文本
- 列出编号选项
- 以 `Clear` + `Block` + `Paragraph` 覆盖层形式渲染

**`render_spinner(f: &mut Frame, area: Rect, frame_count: u64)`：**
- 轮转 braille spinner 字符：`⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏`
- 通过 `frame_count % 10` 取索引

### 模块：`input`

**`is_slash_command(input: &str) -> bool`** - 如果以 `"/"` 开头则返回 true

**`parse_slash_command(input: &str) -> (&str, &str)`** - 把 `"/name args"` 拆成 `("name", "args")`

### 终端设置

**`setup_terminal() -> Result<Terminal<CrosstermBackend<Stdout>>>`：**
1. `enable_raw_mode()`（crossterm）
2. `execute!(stdout, EnterAlternateScreen)`（crossterm）
3. 创建 `Terminal::new(CrosstermBackend::new(stdout))`

**`restore_terminal(terminal: &mut Terminal<...>)`：**
1. `disable_raw_mode()`
2. `execute!(stdout, LeaveAlternateScreen)`
3. `terminal.show_cursor()`

### 与 TypeScript 的关系

`cc-tui` 替代了整个 TypeScript `src/ink/` 渲染系统、`src/components/` 和 React/Ink 组件树。ratatui 的方案本质上不同（立即模式渲染 vs React reconciler），但提供了等价的视觉功能：消息历史、流式文本、输入框、状态栏、权限对话框。

---

## 代码包：`cc-commands`

**路径：** `crates/commands/src/lib.rs`

交互式 REPL 的斜杠命令实现。

### 核心类型

**`CommandContext` 结构体：**
```
config: Config
cost_tracker: Arc<CostTracker>
messages: Vec<Message>
working_dir: PathBuf
```

**`CommandResult` 枚举：**
- `Message(String)` - 向用户显示文本
- `UserMessage(String)` - 作为用户消息注入对话
- `ConfigChange(Config)` - 更新运行配置
- `ClearConversation` - 清空消息历史
- `SetMessages(Vec<Message>)` - 替换消息历史
- `Exit` - 结束会话
- `Silent` - 无输出
- `Error(String)` - 显示错误

**`SlashCommand` trait**（async_trait）：
- `fn name(&self) -> &str`
- `fn aliases(&self) -> Vec<&str>`（默认空）
- `fn description(&self) -> &str`
- `fn help(&self) -> &str`
- `fn hidden(&self) -> bool`（默认 false）
- `async fn execute(&self, args: &str, ctx: &CommandContext) -> CommandResult`

### 命令注册表

**`all_commands() -> Vec<Box<dyn SlashCommand>>`** - 返回所有内置命令

**`find_command(name: &str) -> Option<Box<dyn SlashCommand>>`** - 按名称或别名匹配

**`execute_command(input: &str, ctx: &CommandContext) -> Option<CommandResult>`**（async）：
- 从输入中解析斜杠命令
- 查找匹配命令
- 如果没有匹配，返回 `None`（交给 query loop 处理）

### 已实现命令

| 结构体 | 名称 | 别名 | 说明 |
|---|---|---|---|
| `HelpCommand` | `help` | `h`, `?` | 列出可用的斜杠命令 |
| `ClearCommand` | `clear` | `cls` | 清空对话历史 |
| `CompactCommand` | `compact` | — | 手动压缩对话 |
| `CostCommand` | `cost` | — | 显示当前会话成本 |
| `ExitCommand` | `exit` | `quit`, `q` | 退出 REPL |
| `ModelCommand` | `model` | — | 显示/修改当前模型 |
| `ConfigCommand` | `config` | — | 显示/更新配置 |
| `VersionCommand` | `version` | — | 显示版本信息 |
| `ResumeCommand` | `resume` | — | 恢复之前的对话 |
| `StatusCommand` | `status` | — | 显示会话状态 |
| `DiffCommand` | `diff` | — | 显示文件 diff |
| `MemoryCommand` | `memory` | — | 管理 `CLAUDE.md` 记忆 |
| `BugCommand` | `bug` | — | 提交 bug 报告 |
| `DoctorCommand` | `doctor` | — | 运行诊断 |
| `LoginCommand` | `login` | — | 认证 |
| `LogoutCommand` | `logout` | — | 清除认证 |
| `InitCommand` | `init` | — | 初始化项目 `CLAUDE.md` |
| `ReviewCommand` | `review` | — | 代码审查工作流 |
| `HooksCommand` | `hooks` | — | 管理事件 hooks |
| `McpCommand` | `mcp` | — | 管理 MCP servers |
| `PermissionsCommand` | `permissions` | — | 显示/编辑权限 |
| `PlanCommand` | `plan` | — | 进入/退出 plan 模式 |
| `TasksCommand` | `tasks` | — | 查看后台任务 |
| `SessionCommand` | `session` | — | 会话管理 |
| `ThinkingCommand` | `thinking` | — | 切换 extended thinking |
| `ExportCommand` | `export` | — | 导出对话 |
| `SkillsCommand` | `skills` | — | 列出/管理 skills |
| `RewindCommand` | `rewind` | — | 回退对话状态 |
| `StatsCommand` | `stats` | — | 显示使用统计 |
| `FilesCommand` | `files` | — | 列出上下文文件 |
| `RenameCommand` | `rename` | — | 重命名当前会话 |
| `EffortCommand` | `effort` | — | 设置 effort/thinking 等级 |
| `SummaryCommand` | `summary` | — | 总结对话 |
| `CommitCommand` | `commit` | — | 运行 git commit 工作流 |

### 与 TypeScript 的关系

`cc-commands` 对应 TypeScript 的 `src/commands/` 目录（150+ 文件）。每个 TypeScript 命令模块（例如 `src/commands/compact/`、`src/commands/model/`）都映射为这个 crate 里的一个结构体。斜杠命令名称和行为都被保留。

---

## 代码包：`cc-mcp`

**路径：** `crates/mcp/src/lib.rs`

完整的 MCP（Model Context Protocol）客户端实现。使用 JSON-RPC 2.0 通过 stdio 子进程传输。

### JSON-RPC 类型

**`JsonRpcRequest`：** `{ jsonrpc: "2.0", method: String, params: Option<Value>, id: Option<u64> }`
- `JsonRpcRequest::new(method, params, id)` - 普通请求
- `JsonRpcRequest::notification(method, params)` - 无 id 通知

**`JsonRpcResponse`：** `{ jsonrpc: "2.0", id: Option<u64>, result: Option<Value>, error: Option<JsonRpcError> }`

**`JsonRpcError`：** `{ code: i32, message: String, data: Option<Value> }`

### MCP 协议类型

**`InitializeParams`：** `{ protocol_version: "2024-11-05", capabilities: ClientCapabilities, client_info: ClientInfo }`

**`ClientCapabilities`：** `{ roots: Option<RootsCapability> }`

**`InitializeResult`：** `{ protocol_version: String, capabilities: ServerCapabilities, server_info: ServerInfo }`

**`ServerCapabilities`：** `{ tools: Option<ToolsCapability>, resources: Option<ResourcesCapability>, prompts: Option<PromptsCapability> }`

**`McpTool`：** `{ name: String, description: Option<String>, input_schema: Value }`
- `From<&McpTool> for ToolDefinition` - 转成 cc-core 的 ToolDefinition

**`CallToolParams`：** `{ name: String, arguments: Option<Value> }`

**`CallToolResult`：** `{ content: Vec<McpContent>, is_error: Option<bool> }`

**`McpContent` 枚举**（serde tagged）：
- `Text { type: "text", text: String }`
- `Image { type: "image", data: String, mime_type: String }`
- `Resource { type: "resource", resource: ResourceContents }`

**`McpResource`：** `{ uri: String, name: String, description: Option<String>, mime_type: Option<String> }`

**`McpPrompt`：** `{ name: String, description: Option<String>, arguments: Option<Vec<McpPromptArgument>> }`

### 传输层

**`McpTransport` trait**（async_trait）：
- `async fn send(&mut self, request: &JsonRpcRequest) -> Result<()>`
- `async fn recv(&mut self) -> Result<Option<JsonRpcResponse>>`
- `async fn close(&mut self)`

**`StdioTransport`：**
- `StdioTransport::spawn(command: &str, args: &[String], env: &HashMap<String, String>) -> Result<Self>`
  - 用 piped stdin/stdout 启动子进程
  - 启动后台 reader task，把行转发到 `mpsc::UnboundedReceiver<String>`
- `send()` - 把 JSON 序列化后加换行写到 stdin
- `recv()` - 从 channel 收到数据并反序列化为 JSON-RPC 响应

### `McpClient`

**`McpClient::connect_stdio(config: &McpServerConfig) -> Result<Self>`**（async）：
1. 调用 `StdioTransport::spawn()`
2. 调用 `initialize()` - 发送 `initialize` 请求，接收 `InitializeResult`
3. 发送 `notifications/initialized` 通知
4. 如果存在 `capabilities.tools`：调用 `tools/list`，保存到 `self.tools`
5. 如果存在 `capabilities.resources`：调用 `resources/list`，保存
6. 如果存在 `capabilities.prompts`：调用 `prompts/list`，保存
7. 返回已连接的 client

**`call<T: DeserializeOwned>(&mut self, method, params) -> Result<T>`：**
- 顺序请求/响应：发送带递增 ID 的请求
- 循环调用 `transport.recv()`，直到响应 ID 匹配
- 反序列化 `result` 字段

**`call_tool(&mut self, name: &str, arguments: Option<Value>) -> Result<CallToolResult>`：**
- 用 `CallToolParams` 调 `tools/call`

**`list_resources(&mut self) -> Result<Vec<McpResource>>`** - `resources/list`

**`read_resource(&mut self, uri: &str) -> Result<ResourceContents>`** - `resources/read`

### `McpManager`

管理多个命名的 MCP server 连接。

**`McpManager::connect_all(configs: &[McpServerConfig]) -> Result<Self>`**（async）：
- 尝试连接每个 server；失败时记录 warning，但不终止

**`all_tool_definitions(&self) -> Vec<ToolDefinition>`：**
- 给每个工具名加上 `"{server_name}_"` 前缀，实现命名空间隔离

**`call_tool(&self, prefixed_name: &str, arguments: Option<Value>) -> Result<CallToolResult>`：**
- 去掉 server 前缀识别所属 server
- 路由到正确的 `McpClient`

**`list_all_resources(&self) -> Result<Vec<McpResource>>`：**
- 汇总所有已连接 server 的资源

**`read_resource(&self, uri: &str) -> Result<ResourceContents>`：**
- 逐个 server 尝试，直到有一个返回结果

**`server_count(&self) -> usize`**, **`server_names(&self) -> Vec<String>`**

**`mcp_result_to_string(result: &CallToolResult) -> String`：**
- 把 `McpContent::Text` 转成文本，`McpContent::Image` 转成 `[image: mime_type]`，`McpContent::Resource` 转成 URI/文本

### 与 TypeScript 的关系

`cc-mcp` 对应 `src/services/mcpClient.ts`（TypeScript MCP 实现）。实现了相同的 MCP 协议版本（`2024-11-05`）、stdio 传输和工具命名空间约定。

---

## 代码包：`cc-bridge`

**路径：** `crates/bridge/src/lib.rs`

实现把本地 Claude Code CLI 连接到 claude.ai Web UI 的桥接协议。这样可以从浏览器会话远程控制 CLI。

### 配置

**`BridgeConfig` 结构体：**
- `enabled: bool`
- `server_url: String`
- `device_id: String`
- `session_token: Option<String>`
- `polling_interval_ms: u64`（默认：1000）
- `max_reconnect_attempts: u32`（默认：10）

### 协议类型

**`BridgeMessage` 枚举**（serde tagged - 从 server 到 client 的消息）：
- `UserMessage { content: String, attachments: Vec<String> }`
- `PermissionResponse { tool_use_id: String, decision: PermissionDecision }`
- `Cancel`
- `Ping`

**`BridgeEvent` 枚举**（serde tagged - 从 client 到 server 的事件）：
- `TextDelta { text: String }`
- `ToolStart { tool_name: String, tool_id: String }`
- `ToolEnd { tool_name: String, tool_id: String, result: String, is_error: bool }`
- `PermissionRequest { tool_use_id: String, tool_name: String, description: String }`
- `TurnComplete { stop_reason: String }`
- `Error { message: String }`
- `Pong`

**`PermissionDecision` 枚举：** `Allow`, `AllowPermanently`, `Deny`, `DenyPermanently`

**`BridgeState` 枚举：** `Connecting`, `Connected`, `Reconnecting { attempt: u32 }`, `Disconnected`

### 会话管理

**`BridgeSession::new(config: BridgeConfig) -> (Self, mpsc::Receiver<BridgeMessage>, mpsc::Sender<BridgeEvent>)`：**
- 创建双向通信的 channel 对

**`BridgeManager::start(config, msg_tx, event_rx) -> Self`：**
- 启动 `run_poll_loop()` 后台任务
- 返回带 `JoinHandle` 的 manager

### Polling 循环

**`run_poll_loop(config, msg_tx, event_rx)`**（async）：
1. 通过 `reqwest` GET 长轮询 `{server_url}/sessions/{id}/poll`
2. 收到响应后：反序列化 `BridgeMessage` 数组，并逐个发送到 `msg_tx`
3. Drain `event_rx`：把累积的 `BridgeEvent` 通过 POST 发送到 `{server_url}/sessions/{id}/events`
4. 网络错误时：指数退避，最多到 `max_reconnect_attempts`
5. 401/403 时：设置状态为 `Disconnected`，退出循环

### 模块：`jwt`

**`JwtClaims` 结构体：** `{ sub: String, exp: u64, iat: u64, device_id: String }`

**`decode_payload(token: &str) -> Result<JwtClaims>`：**
- 按 `"."` 拆分 token，取索引 1（payload 段）
- 用 `base64` crate 解码（URL-safe，无 padding）
- 把 JSON 反序列化为 `JwtClaims`

**`is_expired(claims: &JwtClaims) -> bool`：**
- 将 `claims.exp` 与 `SystemTime::now()` 的 Unix 时间戳比较

### 模块：`trusted_device`

**`device_fingerprint() -> String`：**
- 收集：`hostname()`（来自 `hostname` crate）、`USER` 环境变量、home 目录路径
- 用 `sha2` crate 对拼接字符串做 SHA-256
- 用 `hex` crate 返回小写十六进制字符串（前 16 个字符）

### 与 TypeScript 的关系

`cc-bridge` 对应 `src/bridge/`（31 个 TypeScript 文件，包括 `bridgeMain.ts`、`bridgeMessaging.ts`、`replBridge.ts`、`jwtUtils.ts`、`trustedDevice.ts` 等）。实现了相同的轮询式桥接协议和 JWT 处理。

---

## 代码包：`claude-code`（CLI 二进制）

**路径：** `crates/cli/src/main.rs`

二进制入口点，生成 `claude` 可执行文件，并把所有 crate 连接起来。

### CLI 参数（通过 clap derive 的 `Cli` 结构体）

| 标志 | 类型 | 说明 |
|---|---|---|
| `prompt` | `Option<String>`（位置参数） | 非交互式提示 |
| `-p, --print` | `bool` | 打印模式（非交互式别名） |
| `-m, --model` | `Option<String>` | 覆盖模型 |
| `--permission-mode` | `Option<CliPermissionMode>` | 权限模式 |
| `--resume` | `Option<String>` | 按 ID 恢复会话 |
| `--max-turns` | `u32`（默认：10） | 最大对话轮次 |
| `-s, --system-prompt` | `Option<String>` | 覆盖 system prompt |
| `--append-system-prompt` | `Option<String>` | 追加到 system prompt |
| `--no-claude-md` | `bool` | 跳过 `CLAUDE.md` 加载 |
| `--output-format` | `Option<CliOutputFormat>` | 输出格式 |
| `-v, --verbose` | `bool` | 启用详细日志 |
| `--api-key` | `Option<String>` | API key |
| `--max-tokens` | `Option<u32>` | 覆盖 max tokens |
| `--cwd` | `Option<PathBuf>` | 工作目录 |
| `--dangerously-skip-permissions` | `bool` | BypassPermissions 模式 |
| `--dump-system-prompt` | `bool` | 打印 system prompt 后退出 |
| `--mcp-config` | `Option<PathBuf>` | MCP server 配置 JSON 文件 |
| `--no-auto-compact` | `bool` | 禁用自动压缩 |

**`CliPermissionMode` 枚举**（clap ValueEnum）：`Default`, `AcceptEdits`, `BypassPermissions`, `Plan`

**`CliOutputFormat` 枚举**（clap ValueEnum）：`Text`, `Json`, `StreamJson`

### `McpToolWrapper`

为 MCP servers 提供的工具实现 `Tool`：
- `permission_level()` → `Execute`
- `execute()` 会去掉 tool name 的 server 前缀，调用 `McpManager::call_tool()`，再通过 `mcp_result_to_string()` 转换结果

### `main()` 函数

1. 用 clap 解析 `Cli` 参数
2. 设置 `tracing_subscriber`（verbose → DEBUG，默认 → WARN）
3. 从 `~/.claude/settings.json` 加载 `Settings`
4. 按 `settings → CLI overrides` 分层构建 `Config`
5. 确定 `working_dir`（来自 `--cwd` 或 `std::env::current_dir()`）
6. 创建 `Arc<CostTracker>`
7. 构建 system context 字符串：
   - 读取 `crates/cli/src/system_prompt.txt`（通过 `include_str!` 在编译期嵌入）
   - 调用 `ContextBuilder::build_system_context()`
   - 调用 `ContextBuilder::build_user_context()`（除非传了 `--no-claude-md`）
   - 把所有部分拼起来
8. 如果指定 `--dump-system-prompt`：打印并退出
9. 创建 `AnthropicClient::from_config()`
10. 用 `AutoPermissionHandler` 创建 `ToolContext`
11. 如果提供了 MCP 配置，则调用 `McpManager::connect_all()`
12. 构建工具列表：`cc_tools::all_tools()` + `AgentTool` + 每个 MCP tool 的 `McpToolWrapper`
13. 创建 `CancellationToken`，并用 `start_cron_scheduler()` 启动 cron 调度器
14. 如果提供了 prompt 或 `--print`：调用 `run_headless()`
15. 否则：调用 `run_interactive()`

### `run_headless(prompt, client, messages, tools, tool_ctx, config, cost_tracker, output_format)`

1. 从参数或 stdin 读取 prompt（如果没有位置参数）
2. 把 `Message::user(prompt)` 追加到 messages
3. 用 `mpsc::unbounded_channel()` 为事件启动 `run_query_loop()`
4. Drain 事件 channel：
   - `Text` 格式：直接打印 `QueryEvent::Stream(TextDelta)` 的文本，并打印工具名
   - `Json` 格式：收集完整响应，输出为单个 JSON 对象
   - `StreamJson` 格式：把每个 `QueryEvent` 以 NDJSON 行输出
5. 在 `QueryOutcome::EndTurn` 或错误时返回

### `run_interactive()`

交互式 TUI REPL：
1. 通过 `cc_tui::setup_terminal()` 设置终端
2. 退出时恢复终端（使用 `defer` 风格）
3. 如果传了 `--resume`，处理会话恢复
4. 主事件循环每 16ms poll 一次（来自 crossterm 的 `EventStream`）：
   - 通过 `app.handle_key_event()` 处理 `crossterm::event::KeyEvent`
   - 按 Enter 时：如果是斜杠命令（`is_slash_command()`），调用 `cc-commands` 的 `execute_command()`
   - 普通消息：追加到 `messages`，并用 `tokio::spawn` 启动 `run_query_loop()`
   - 通过 `Arc<Mutex<Vec<Message>>>` 在主任务和子任务之间共享结果同步
   - 用 `event_rx.try_recv()` drain 查询事件
   - 调用 `app.handle_query_event()` 更新 TUI 状态
   - 通过 `terminal.draw(|f| render_app(f, &app))` 重新渲染
5. 每完成一轮后，把会话保存到 `cc_core::history::save_session()`

### 系统 Prompt（`system_prompt.txt`）

二进制在编译时嵌入。内容：
> You are Claude Code, an AI coding assistant by Anthropic.

指导原则：
- 先读文件再改文件
- 优先修改已有文件，不要轻易新建
- 写出干净、符合惯例的代码
- 改完后跑测试
- 使用 git log/diff 了解代码库上下文
- 回复要简洁
- 产出可直接投入生产的代码
- 不要引入安全漏洞

### 与 TypeScript 的关系

`claude-code` CLI 对应 `src/entrypoints/cli.tsx`（TypeScript 的主 CLI 入口）、`src/main.tsx`、`src/screens/REPL.tsx` 和 `src/cli/` 目录。CLI flag 名称和行为都被保留，包括 `--print`、`--output-format`、`--permission-mode` 和 `--resume`。

---

## 横切架构说明

### 异步运行时
所有异步代码都使用带 `"full"` 特性的 `tokio`。`#[tokio::main]` 宏用于 `crates/cli/src/main.rs` 的 `main()`。所有工具都通过 `async_trait` 使用 `async fn execute()`。

### 取消
`tokio_util::sync::CancellationToken` 会贯穿 `run_query_loop()`、cron 调度器和 TUI 事件循环。`Ctrl+C` 会触发这个 token。

### 全局状态
通过 `once_cell::sync::Lazy` 实现的三个 `DashMap`/`RwLock` 单例：
- `TASK_STORE`（cc-tools/tasks.rs） - 任务管理
- `INBOX`（cc-tools/send_message.rs） - 智能体间消息传递
- `CRON_STORE`（cc-tools/cron.rs） - 计划任务
- `WORKTREE_SESSION`（cc-tools/worktree.rs） - 当前 git worktree

### 错误处理
- 库使用 `thiserror` 定义类型化的 `ClaudeError`
- CLI binary 使用 `anyhow` 做便捷的错误传播
- 工具错误永远不会 panic，始终返回 `ToolResult::error()`

### 提示缓存
`cc-api` 会自动把 `CacheControl::ephemeral()` 应用到：
- System prompt blocks（使用 `SystemPrompt::Blocks` 时）
- tools 列表中的最后一个工具定义

### 日志
使用 `tracing` + `tracing-subscriber` 和 `EnvFilter`。默认级别是 WARN；`--verbose` 会打开 DEBUG。所有日志调用都带结构化字段。

### TypeScript 对应关系总览

| TypeScript 区域 | Rust 代码包 |
|---|---|
| `src/entrypoints/cli.tsx`, `src/main.tsx` | `crates/cli` |
| `src/services/api/` | `crates/api` |
| `src/query.ts`, `src/query/` | `crates/query` |
| `src/components/`, `src/ink/` | `crates/tui` |
| `src/commands/` | `crates/commands` |
| `src/constants/`, `src/context.ts` 等 | `crates/core` |
| 工具实现（Bash、Read、Edit 等） | `crates/tools` |
| MCP client（`src/services/mcpClient.ts`） | `crates/mcp` |
| `src/bridge/` | `crates/bridge` |
