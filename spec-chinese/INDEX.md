# Claude Code — Spec 索引

> 所有 spec 文档的快速索引。
> 总覆盖量：15 份 Markdown，约 990 KB。

---

## Spec 文件

| # | 文件 | 大小 | 内容简介 |
|---|------|------|----------|
| — | [00_overview.md](00_overview.md) | 16 KB | 总体架构、仓库结构、数据流、权限模型、设置分层 |
| 01 | [01_core_entry_query.md](01_core_entry_query.md) | 73 KB | `main.tsx`、`query.ts`、`QueryEngine.ts`、入口点、历史、成本跟踪、token 预算 |
| 02 | [02_commands.md](02_commands.md) | 71 KB | 全部 100+ 个斜杠命令，包括参数、选项和实现方式 |
| 03 | [03_tools.md](03_tools.md) | 67 KB | 全部 40+ 个工具：输入 schema、权限、输出、共享工具函数 |
| 04 | [04_components_core_messages.md](04_components_core_messages.md) | 93 KB | 130 个顶层 UI 组件，以及全部消息渲染组件 |
| 05 | [05_components_agents_permissions_design.md](05_components_agents_permissions_design.md) | 64 KB | Agent 创建向导、权限对话框、设计系统、PromptInput、Spinner |
| 06 | [06_services_context_state.md](06_services_context_state.md) | 95 KB | Analytics、API client、session memory、autoDream、compact、voice、contexts、state |
| 07 | [07_hooks.md](07_hooks.md) | 84 KB | 全部 104 个 React hook，包括参数、返回类型和行为 |
| 08 | [08_ink_terminal.md](08_ink_terminal.md) | 78 KB | 自定义终端框架：React reconciler、Yoga 布局、screen buffer、ANSI tokenizer |
| 09 | [09_bridge_cli_remote.md](09_bridge_cli_remote.md) | 75 KB | Bridge 协议、JWT auth、SSE/WebSocket/Hybrid transport、远程会话 |
| 10 | [10_utils.md](10_utils.md) | 60 KB | 约 564 个工具文件，按类别整理 |
| 11 | [11_special_systems.md](11_special_systems.md) | 64 KB | Buddy/Tamagotchi、memdir、keybindings、skills、voice、plugins、migrations |
| 12 | [12_constants_types.md](12_constants_types.md) | 83 KB | 全部常量、类型、OAuth 配置、system prompts、tool limits、beta headers |
| 13 | [13_rust_codebase.md](13_rust_codebase.md) | 63 KB | 完整 Rust 重写：全部 9 个 crate、33 个工具、query loop、TUI、bridge |

---

## 快速查找

### “X 在哪里有文档？”

| 主题 | Spec 文件 | 章节 |
|-------|-----------|------|
| 主入口点（`main.tsx`） | 01 | §1 |
| Query / turn 执行循环 | 01 | §2–3 |
| Token 预算与 compact | 01 | §token-budget |
| 工具基类与框架 | 03 | §1 |
| BashTool | 03 | §BashTool |
| FileEditTool | 03 | §FileEditTool |
| AgentTool（sub-agents） | 03 | §AgentTool |
| WebSearchTool | 03 | §WebSearchTool |
| MCPTool | 03 | §MCPTool |
| 全部斜杠命令 | 02 | §per-command |
| `/compact` 命令 | 02 | §compact |
| `/mcp` 命令 | 02 | §mcp |
| `/plan` 命令 | 02 | §plan |
| 权限对话框系统 | 05 | §permissions |
| 权限规则（settings） | 05 | §rules |
| PromptInput 组件 | 05 | §PromptInput |
| 消息渲染 | 04 | §messages |
| Spinner 组件 | 05 | §Spinner |
| Agent 创建向导 | 05 | §agents |
| Claude API client | 06 | §api/claude |
| Analytics / telemetry | 06 | §analytics |
| Session memory | 06 | §SessionMemory |
| AutoDream consolidation | 06 | §autoDream |
| Rate limiting | 06 | §claudeAiLimits |
| Context compaction | 06 | §compact |
| React contexts | 06 | §context |
| Bootstrap state（80+ 字段） | 06 | §bootstrap |
| Coordinator mode | 06 | §coordinator |
| 全部 React hooks | 07 | §per-hook |
| Ink reconciler | 08 | §reconciler |
| Yoga layout engine | 08 | §layout |
| Screen buffer / rendering | 08 | §screen |
| ANSI/CSI/ESC 处理 | 08 | §termio |
| Bridge 协议 | 09 | §bridge |
| JWT 认证 | 09 | §jwtUtils |
| SSE transport | 09 | §SSETransport |
| WebSocket transport | 09 | §WebSocketTransport |
| 远程会话 | 09 | §remote |
| Buddy/Tamagotchi | 11 | §buddy |
| Gacha 机制（PRNG） | 11 | §buddy-gacha |
| Memory directory 系统 | 11 | §memdir |
| Keybinding parser | 11 | §keybindings |
| Skills 系统 | 11 | §skills |
| Voice / STT | 11 | §voice |
| Plugin 系统 | 11 | §plugins |
| Model migration 历史 | 11 | §migrations |
| 全部常量 | 12 | §constants |
| System prompt 架构 | 12 | §prompts |
| OAuth 配置 | 12 | §oauth |
| Beta feature headers | 12 | §betas |
| Cyber risk instruction | 12 | §cyberRisk |
| Tool name 常量 | 12 | §tools |
| 全部 TypeScript 类型 | 12 | §types |
| Rust 重写总览 | 13 | §1 |
| Rust 工具实现 | 13 | §cc-tools |
| Rust query loop | 13 | §cc-query |
| Rust TUI | 13 | §cc-tui |
| Rust bridge | 13 | §cc-bridge |

---

## 关键数字

| 指标 | 数值 |
|--------|-------|
| TypeScript/TSX 文件总数 | ~1,902 |
| 代码总行数 | ~800K+ |
| 斜杠命令数量 | 100+ |
| 工具数量 | 40+ |
| React hooks 数量 | 104 |
| React 组件数量 | 389 个文件 |
| Services 数量 | 130 个文件 |
| Utility 文件数量 | ~564 |
| Ink 终端框架文件数 | 96 |
| Bridge 协议文件数 | 31 |
| Rust crates 数量 | 9 |
| Rust 源文件数量 | 47 |
| Spec 文档体量 | ~990 KB |

---

## 一段话理解架构

Claude Code 是一个终端里的 AI 编码助手。它本质上是一个 React 应用，只是运行目标不是浏览器，而是自定义的终端 UI 框架 Ink。主循环（`query.ts` + `QueryEngine.ts`）负责从 Claude API 流式接收响应、在用户许可下执行工具，并在大约 200K token 的上下文窗口里自动做 compact。系统上层有 100+ 个斜杠命令、40+ 个工具（文件 I/O、shell、web、agents、MCP）、一个可并行执行任务的多 agent 系统、长期记忆系统、语音输入、通过 WebSocket/SSE 接到 IDE 和远程会话的 bridge 协议，以及插件 / skills 市场。与此同时，整个代码库正在用 Rust（`claude-code-rust/`）做一套完整、独立的重写。

---

*根据 Claude Code 源码分析，于 2026-03-31 生成。*
