# Claude Code — 架构总览

> **仓库：** `X:\Bigger-Projects\Claude-Code`
> **主要语言：** TypeScript/TSX（约1,902个文件，约80万+行代码）
> **次要语言：** Rust（约47个文件，移植中）
> **打包工具：** Bun
> **UI框架：** 自定义 Ink（用于终端的React协调器）
> **运行时目标：** Node.js / Bun CLI

---

## 1. 什么是 Claude Code？

Claude Code 是一个 AI 驱动的 CLI 工具和编码助手。它是一个功能齐全的交互式终端应用程序，能够：

- 将 Claude AI 模型作为智能编码助手嵌入
- 使用基于 React 的自定义 TUI（终端用户界面）在终端中运行
- 在用户许可下执行工具（文件读/写、bash、grep、网络搜索等）
- 支持多智能体任务委托、后台智能体和集群模式
- 通过直连桥接与 IDE（VS Code、JetBrains）集成
- 支持通过 WebSocket/SSE 传输的远程会话
- 拥有插件/技能市场
- 包含语音输入（语音转文本）
- 具有伴侣“小伙伴”系统（类似拓麻歌子）
- 通过桥接协议将会话同步到云端

---

## 2. 仓库结构

```
Claude-Code/
├── src/                          # 主要 TypeScript/TSX 源代码（34 MB，约1,902个文件）
│   ├── main.tsx                  # 主要入口点（4,683 行）
│   ├── replLauncher.tsx          # REPL 模式启动器
│   ├── query.ts                  # 主要查询/轮次执行引擎（69KB）
│   ├── QueryEngine.ts            # 查询引擎类（46KB）
│   ├── Tool.ts                   # 工具基础框架（30KB）
│   ├── Task.ts                   # 任务定义
│   ├── commands.ts               # 命令注册表（25KB）
│   ├── context.ts                # 上下文管理
│   ├── cost-tracker.ts           # 成本跟踪（11KB）
│   ├── costHook.ts               # 成本钩子
│   ├── history.ts                # 会话历史（14KB）
│   ├── dialogLaunchers.tsx       # 对话框启动器（23KB）
│   ├── interactiveHelpers.tsx    # 交互式 UI 助手（57KB）
│   ├── projectOnboardingState.ts # 项目引导状态
│   ├── setup.ts                  # 初始化（21KB）
│   ├── tasks.ts                  # 任务管理
│   ├── tools.ts                  # 工具注册表（17KB）
│   ├── ink.ts                    # Ink 导出垫片
│   │
│   ├── assistant/                # 助手会话历史
│   ├── bootstrap/                # 引导/状态
│   ├── bridge/                   # 桥接协议（31个文件）
│   ├── buddy/                    # 小伙伴系统（6个文件）
│   ├── cli/                      # CLI 框架和传输层（19个文件）
│   ├── commands/                 # 87 个斜杠命令（207个文件）
│   ├── components/               # React/Ink UI 组件（389个文件，32个子目录）
│   ├── constants/                # 常量和配置值（21个文件）
│   ├── context/                  # React 上下文提供者（9个文件）
│   ├── coordinator/              # 协调器模式逻辑
│   ├── entrypoints/              # 多个入口点（8个文件）
│   ├── hooks/                    # React 钩子（104个文件）
│   ├── ink/                      # 自定义 Ink 终端框架（96个文件）
│   ├── keybindings/              # 键盘快捷键系统（14个文件）
│   ├── memdir/                   # 内存目录系统（8个文件）
│   ├── migrations/               # 设置迁移（11个文件）
│   ├── moreright/                # useMoreRight 钩子
│   ├── native-ts/                # 原生 TypeScript 绑定（4个文件）
│   ├── outputStyles/             # 输出样式加载器
│   ├── plugins/                  # 插件系统（2个文件）
│   ├── query/                    # 查询助手（4个文件）
│   ├── remote/                   # 远程会话管理（4个文件）
│   ├── schemas/                  # Zod/JSON 模式
│   ├── screens/                  # 顶层屏幕布局（3个文件）
│   ├── server/                   # 直连服务器（3个文件）
│   ├── services/                 # 业务逻辑服务（130个文件）
│   ├── skills/                   # Claude 技能/斜杠命令（20个文件）
│   ├── tools/                    # 工具实现（40+个工具，184个文件）
│   ├── types/                    # TypeScript 类型定义
│   ├── utils/                    # 工具函数（约564个文件）
│   └── voice/                    # 语音集成
│
├── claude-code-rust/             # Rust 移植（进行中，47个文件）
│   ├── Cargo.toml                # 工作区清单
│   ├── tools/                    # 27个文件 — 工具实现
│   ├── query/                    # 5个文件 — 查询系统
│   ├── cli/                      # 3个文件 — CLI 框架
│   ├── api/                      # 2个文件 — API 绑定
│   ├── bridge/                   # 2个文件 — 桥接协议
│   ├── commands/                 # 2个文件 — 命令系统
│   ├── core/                     # 2个文件 — 核心工具
│   ├── mcp/                      # 2个文件 — MCP 集成
│   └── tui/                      # 2个文件 — 终端 UI
│
├── public/                       # 静态资源
├── README.md                     # 主要文档（27KB）
└── .git/                         # Git 元数据
```

---

## 3. 高层架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户界面                                 │
│  终端（Ink TUI） ←→ React 组件 ←→ 钩子 ←→ 上下文                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       主应用程序                                 │
│  main.tsx → REPL.tsx → PromptInput → MessageList               │
│  命令（87） ←→ 命令注册表 ←→ 插件系统                           │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       查询引擎                                   │
│  query.ts → QueryEngine.ts → 工具执行 → 响应处理                │
│  令牌预算 → 停止钩子 → 压缩 → 历史                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       工具系统（40+个工具）                      │
│  BashTool, FileReadTool, FileEditTool, FileWriteTool            │
│  GlobTool, GrepTool, WebFetchTool, WebSearchTool                │
│  AgentTool, TaskCreateTool, MCPTool, SkillTool, ...             │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       服务层                                     │
│  API 客户端（claude.ts） → 分析 → 会话内存                      │
│  自动梦境 → 压缩 → 速率限制 → MCP 服务器                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       传输层                                     │
│  CLI（本地）/ 桥接（远程）/ IDE 直连                           │
│  SSETransport | WebSocketTransport | HybridTransport           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 核心子系统

### 4.1 查询 / 轮次执行（`query.ts`、`QueryEngine.ts`）
核心循环，执行以下步骤：
1. 接收用户输入
2. 构建 API 请求（系统提示 + 历史 + 工具）
3. 从 Claude API 流式传输响应
4. 处理工具使用（执行工具，反馈结果）
5. 管理令牌预算和上下文压缩
6. 跟踪成本

### 4.2 工具框架（`Tool.ts`、`tools/`）
- 基础 `Tool` 抽象类/接口
- 输入模式验证（Zod）
- 权限系统（每个工具声明所需权限）
- 40+ 个工具实现
- 危险工具的沙盒化

### 4.3 终端 UI（`ink/`、`components/`）
- 自定义 React 协调器，渲染到终端
- 基于 Yoga 的布局引擎（终端的 flexbox）
- 事件系统（键盘、鼠标、焦点）
- ANSI/CSI/转义序列处理
- 组件：消息、提示输入、旋转器、对话框等

### 4.4 命令系统（`commands/`、`commands.ts`）
- 87 个斜杠命令（例如 `/compact`、`/diff`、`/plan`、`/mcp`）
- 插件贡献的命令
- 带有模糊匹配的命令注册表
- 键盘快捷键集成

### 4.5 桥接协议（`bridge/`）
- 启用远程/云同步会话
- 基于 JWT 认证的 WebSocket/SSE 连接到云端后端服务器
- 用于 IDE 集成的 REPL 桥接
- 消息轮询、刷新门、会话运行器

### 4.6 多智能体系统（`tools/AgentTool.ts`、`components/agents/`）
- 生成隔离的 Claude 实例作为子智能体
- 后台任务执行
- 协调器模式（编排多个智能体）
- 集群模式（并行工作智能体）
- 协作智能体的团队系统

### 4.7 内存系统（`memdir/`、`services/SessionMemory/`、`services/autoDream/`）
- 短期：会话历史
- 长期：内存目录（`~/.claude/memory/` 中的 markdown 文件）
- 自动整合：“梦境”服务在空闲时整合记忆
- 内存扫描/相关性评分，用于上下文注入

### 4.8 MCP 集成（`tools/MCPTool.ts`、`components/mcp/`、`entrypoints/mcp.ts`）
- 模型上下文协议服务器支持
- 从 MCP 服务器动态注册工具
- 资源管理
- 启发式对话框支持

### 4.9 插件/技能系统（`plugins/`、`skills/`、`commands/plugin/`）
- 内置插件
- 社区插件市场
- 技能：用户可调用的斜杠命令宏
- 带有批准流程的插件信任模型

### 4.10 IDE 集成（`bridge/`、`hooks/useIDEIntegration.tsx`）
- VS Code / JetBrains 扩展通过直连连接
- 在 IDE 中实时查看差异
- 文件选择同步（IDE → Claude）
- IDE 中的状态指示器

---

## 5. 数据流：用户轮次

```
1. 用户在 PromptInput 中输入
2. 输入提交 → useCommandQueue 处理
3. 如果是斜杠命令：分派给命令处理器
4. 如果是常规提示：发送到 query.ts 的 runQuery()
5. QueryEngine 构建 API 请求：
   - 系统提示（来自 constants/prompts.ts + CLAUDE.md）
   - 消息历史（来自 history.ts）
   - 可用工具（根据权限过滤）
   - 令牌预算限制
6. 从 Claude API 流式传输响应（services/api/claude.ts）
7. 对于每个内容块：
   - 文本 → 渲染 AssistantTextMessage
   - 思考 → 渲染 AssistantThinkingMessage
   - 工具使用 → 执行工具，如果需要则显示权限对话框
8. 工具结果反馈到下一个 API 请求
9. 循环直到停止条件（没有更多工具使用、停止钩子、预算超出）
10. 最终响应渲染，历史更新，成本跟踪
```

---

## 6. 按重要性排序的关键文件

| 排名 | 文件 | 大小 | 作用 |
|------|------|------|------|
| 1 | `src/main.tsx` | 4,683 行 | 主要入口点，应用程序初始化 |
| 2 | `src/query.ts` | 69KB | 主要查询执行循环 |
| 3 | `src/QueryEngine.ts` | 46KB | 查询引擎类 |
| 4 | `src/interactiveHelpers.tsx` | 57KB | 交互式 UI 助手 |
| 5 | `src/Tool.ts` | 30KB | 工具基础框架 |
| 6 | `src/commands.ts` | 25KB | 命令注册表 |
| 7 | `src/dialogLaunchers.tsx` | 23KB | 对话框启动系统 |
| 8 | `src/setup.ts` | 21KB | 初始化 |
| 9 | `src/tools.ts` | 17KB | 工具注册表 |
| 10 | `src/history.ts` | 14KB | 会话历史 |

---

## 7. 权限模型

Claude Code 使用分层权限系统：

1. **自动** — 只读操作，信息查询
2. **询问一次** — 提示用户，在会话中记住
3. **总是询问** — 每次提示用户
4. **拒绝** — 完全阻止

权限规则存储在设置中（全局 `~/.claude/settings.json`，项目 `.claude/settings.json`），并可以使用模式进行配置。

权限类别：
- `Bash` — Shell 命令执行
- `FileRead` — 读取文件/目录
- `FileEdit` — 编辑现有文件
- `FileWrite` — 创建新文件
- `WebFetch` — HTTP 请求
- `MCP` — MCP 工具调用
- `Sandbox` — 沙盒执行

---

## 8. 设置系统

分层设置（按优先级顺序）：
1. **托管** — 企业/托管设置（只读）
2. **本地项目** — `.claude/settings.local.json`（git忽略）
3. **项目** — `.claude/settings.json`（共享）
4. **全局** — `~/.claude/settings.json`

设置包括：模型选择、权限规则、API 密钥、主题、键盘快捷键、MCP 服务器配置、测试版功能。

---

## 9. 模型支持

根据迁移文件，模型演进过程：
- `claude-3-sonnet` → `claude-sonnet-1m` → `claude-sonnet-4-5` → `claude-sonnet-4-6`
- `claude-3-opus` → `claude-opus-1m` → `claude-opus` →（各种版本）
- `claude-3-5-haiku` →（当前）
- `claude-haiku-4-5`（当前 haiku）

当前默认值（根据源代码）：`claude-sonnet-4-6` 和 `claude-opus-4-6`

---

## 10. 分析与遥测

- **第一方日志记录** — 会话事件发送到后端（`services/analytics/`）
- **Datadog** — 性能指标
- **Growthbook** — 功能标志 / A/B 测试
- **选择退出** — `services/api/metricsOptOut.ts` 处理用户选择退出

---

## 11. 规范文档索引

| 文件 | 内容 |
|------|----------|
| `00_overview.md` | 本文件 — 架构总览 |
| `01_core_entry_query.md` | 入口点、查询系统、历史、成本跟踪 |
| `02_commands.md` | 所有 87 个斜杠命令 |
| `03_tools.md` | 所有 40+ 个工具实现 |
| `04_components_core_messages.md` | 顶层组件和消息组件 |
| `05_components_agents_permissions_design.md` | 智能体、权限、设计系统、功能模块 |
| `06_services_context_state.md` | 服务、上下文提供者、状态、屏幕、服务器 |
| `07_hooks.md` | 所有 React 钩子 |
| `08_ink_terminal.md` | Ink 终端渲染框架 |
| `09_bridge_cli_remote.md` | 桥接协议、CLI 框架、远程会话 |
| `10_utils.md` | 所有工具函数（约564个文件） |
| `11_special_systems.md` | 小伙伴、内存、键盘快捷键、技能、语音、插件 |
| `12_constants_types.md` | 所有常量、类型和配置 |
| `13_rust_codebase.md` | Rust 移植/重写 |
| `INDEX.md` | 快速参考索引 |

---

*根据 Claude Code 代码库的源代码分析生成。约1,902个 TypeScript/TSX 文件，约80万+行代码。*