# Claude Code — 特殊系统：Buddy、Memory、Keybindings、Skills、Voice、Plugins 等

---

## 目录

1. [Buddy（Companion/Tamagotchi）系统](#1-buddycompaniontamagotchi系统)
2. [Memory Directory（memdir）系统](#2-memory-directorymemdir系统)
3. [Keybindings 系统](#3-keybindings系统)
4. [Skills 系统](#4-skills系统)
5. [Voice 系统](#5-voice系统)
6. [Plugins 系统](#6-plugins系统)
7. [输出样式](#7-输出样式)
8. [Hooks 模式](#8-hooks-模式)
9. [Native-TypeScript 端口（native-ts/）](#9-native-typescript-端口native-ts)
10. [MoreRight Hook（moreright/）](#10-moreright-hookmoreright)
11. [迁移](#11-迁移)
12. [核心类型定义（types/）](#12-核心类型定义types)
13. [远程会话系统（remote/）](#13-远程会话系统remote)

---

## 1. Buddy（Companion/Tamagotchi）系统

### 1.1 系统概览

Buddy 系统是一个虚拟伙伴（“拓麻歌子”），会以 ASCII art sprite 的形式出现在用户 prompt 输入框旁边。每个用户都会得到一个确定性的伙伴，它的外观（物种、眼睛、帽子、稀有度、shiny 状态、stats）由用户 ID 经过 seeded PRNG 计算得出，而不是存储在配置里。伙伴的 “灵魂”（名字和个性）由 AI 生成，并持久化到 config。这个功能由 `BUDDY` feature flag 控制，原计划在 2026 年 4 月 1 日到 7 日之间作为一个 “teaser window” 上线。

**源目录：** `src/buddy/`

**文件：**
- `buddy/types.ts` - 类型定义、species/eyes/hats/stats 常量、rarity 权重
- `buddy/companion.ts` - PRNG、rolling 和 companion 重建逻辑
- `buddy/sprites.ts` - 18 个 species 的 ASCII sprite 帧
- `buddy/prompt.ts` - 向 system prompt 注入 companion 介绍文本
- `buddy/useBuddyNotification.tsx` - 启动提示通知的 React hook
- `buddy/CompanionSprite.tsx` - 渲染动画 sprite + 对话气泡的 React 组件

### 1.2 架构

系统把 companion 数据拆成两部分：

- **Bones**（`CompanionBones`）：由 `hash(userId + SALT)` 推导出的确定性视觉/数值数据，不会存储，每次读取都会重新计算。这样用户不能通过改配置把自己伪造成传奇稀有度。
- **Soul**（`CompanionSoul`）：AI 生成的名字和个性，持久化到 `config.companion`，类型为 `StoredCompanion`。
- **Full Companion**（`Companion = CompanionBones & CompanionSoul & { hatchedAt: number }`）：在读取时由 `getCompanion()` 组装。

rolling 过程只需一次 seeded PRNG（`mulberry32`）遍历，就能确定性地选择：
1. 稀有度层级（加权随机）
2. species
3. 眼睛样式
4. 帽子（common 时没有）
5. shiny 标志（1% 概率）
6. stats（一个峰值 stat、一个低谷 stat，其余分散；floor 随稀有度变化）
7. `inspirationSeed`（传给 AI 生成 soul）

### 1.3 Species 编码（Canary 绕过）

**关键细节：** species 名称字符串使用 `String.fromCharCode(...)` 字面量编码，以避免触发 build-time string scan（检查 model codename 的 `excluded-strings.txt`）。其中一个 species 名称会撞上内部 model codename 的 canary。18 个 species 都统一使用这种编码方式。

```typescript
// 在 buddy/types.ts 中：
const c = String.fromCharCode
export const duck = c(0x64,0x75,0x63,0x6b) as 'duck'
// ... 其余 18 个 species 都这样编码
```

### 1.4 PRNG：Mulberry32

**算法：**
```typescript
function mulberry32(seed: number): () => number {
  let a = seed >>> 0
  return function () {
    a |= 0
    a = (a + 0x6d2b79f5) | 0
    let t = Math.imul(a ^ (a >>> 15), 1 | a)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}
```

- 体积很小的 32-bit PRNG，统计分布不错
- 种子来自 `hashString(userId + SALT)`，其中 `SALT = 'friend-2026-401'`
- `hashString` 在 Bun 可用时用原生 hash，否则回退到 FNV-1a（32-bit）

**缓存：** `roll()` 会按 `userId + SALT` 记住上一次结果，因为它会在三个热路径里被调用：500ms 的 sprite tick、每次按键时的 PromptInput 渲染，以及每轮的 observer。

### 1.5 数据结构

#### `Rarity`
```typescript
export const RARITIES = ['common', 'uncommon', 'rare', 'epic', 'legendary'] as const
export type Rarity = (typeof RARITIES)[number]
```

#### `RARITY_WEIGHTS`
```typescript
export const RARITY_WEIGHTS = {
  common: 60,    // 60%
  uncommon: 25,  // 25%
  rare: 10,      // 10%
  epic: 4,       // 4%
  legendary: 1,  // 1%
} as const
```

#### `RARITY_FLOOR`（各稀有度的 stat 最低值）
```typescript
const RARITY_FLOOR: Record<Rarity, number> = {
  common: 5,
  uncommon: 15,
  rare: 25,
  epic: 35,
  legendary: 50,
}
```

#### `RARITY_STARS`（UI 显示）
```typescript
export const RARITY_STARS = {
  common: '★',
  uncommon: '★★',
  rare: '★★★',
  epic: '★★★★',
  legendary: '★★★★★',
}
```

#### `RARITY_COLORS`（映射到 Theme keys）
```typescript
export const RARITY_COLORS = {
  common: 'inactive',
  uncommon: 'success',
  rare: 'permission',
  epic: 'autoAccept',
  legendary: 'warning',
}
```

#### `Species`（共 18 个）
```typescript
export const SPECIES = [
  duck, goose, blob, cat, dragon, octopus, owl, penguin,
  turtle, snail, ghost, axolotl, capybara, cactus, robot,
  rabbit, mushroom, chonk,
] as const
export type Species = (typeof SPECIES)[number]
```

#### `Eye`（6 种样式）
```typescript
export const EYES = ['·', '✦', '×', '◉', '@', '°'] as const
```

#### `Hat`（8 种类型）
```typescript
export const HATS = [
  'none', 'crown', 'tophat', 'propeller',
  'halo', 'wizard', 'beanie', 'tinyduck',
] as const
```

#### `StatName`（5 个 stats）
```typescript
export const STAT_NAMES = ['DEBUGGING', 'PATIENCE', 'CHAOS', 'WISDOM', 'SNARK'] as const
```

#### `CompanionBones`（确定性，不存储）
```typescript
export type CompanionBones = {
  rarity: Rarity
  species: Species
  eye: Eye
  hat: Hat
  shiny: boolean
  stats: Record<StatName, number>
}
```

#### `CompanionSoul`（AI 生成，存储）
```typescript
export type CompanionSoul = {
  name: string
  personality: string
}
```

#### `StoredCompanion`（配置里持久化的内容）
```typescript
export type StoredCompanion = CompanionSoul & { hatchedAt: number }
```

#### `Companion`（运行时组装结果）
```typescript
export type Companion = CompanionBones & CompanionSoul & { hatchedAt: number }
```

#### `Roll`（`rollFrom` 的输出）
```typescript
export type Roll = {
  bones: CompanionBones
  inspirationSeed: number
}
```

### 1.6 Stat Rolling 算法

```typescript
function rollStats(rng, rarity): Record<StatName, number> {
  const floor = RARITY_FLOOR[rarity]  // 依稀有度不同，范围 5–50
  const peak = pick(rng, STAT_NAMES)   // 一个 stat 获得 +50 奖励
  let dump = pick(rng, STAT_NAMES)     // 另一个 stat 被惩罚
  while (dump === peak) dump = pick(rng, STAT_NAMES)

  for (name of STAT_NAMES) {
    if (name === peak)   stats[name] = min(100, floor + 50 + rng()*30)
    else if (name === dump) stats[name] = max(1, floor - 10 + rng()*15)
    else                 stats[name] = floor + rng()*40
  }
}
```

### 1.7 Sprite 系统

每个 species 都有 **3 帧** ASCII 动画，每帧 5 行高、12 列宽。模板中的 `{E}` 占位符会替换成 companion 的 `eye` 字符。

**帧结构：**
- 第 0 行：帽子槽位（如果没有帽子则为空，或者像烟雾 `~`、火花 `*` 这类环境细节）
- 第 1–4 行：主体图案

**帽子渲染：** 只有当第 0 行是空白时才会放帽子。如果所有帧的第 0 行都是空白，就会把这一行裁掉（没有帽子时可节省一个终端行）。

**Hat ASCII art（`HAT_LINES` 中）：**
| Hat | ASCII |
|-----|-------|
| none | (empty) |
| crown | `   \^^^/    ` |
| tophat | `   [___]    ` |
| propeller | `    -+-     ` |
| halo | `   (   )    ` |
| wizard | `    /^\     ` |
| beanie | `   (___)    ` |
| tinyduck | `    ,>      ` |

**Idle 动画序列：**
```typescript
const IDLE_SEQUENCE = [0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]
// -1 = 在 frame 0 上 blink（特殊）
// TICK_MS = 每步 500ms
```

**Pet 动画：** 执行 `/buddy pet` 之后，爱心会向上飘 5 个 tick（大约 2.5 秒）：
```typescript
const PET_HEARTS = [
  `   ${H}    ${H}   `,
  `  ${H}  ${H}   ${H}  `,
  ` ${H}   ${H}  ${H}   `,
  `${H}  ${H}      ${H} `,
  '·    ·   ·  '
]
```

### 1.8 全部导出

#### `buddy/companion.ts`
| 导出 | 类型 | 说明 |
|--------|------|------|
| `Roll` | type | `{ bones: CompanionBones; inspirationSeed: number }` |
| `roll(userId: string)` | `() => Roll` | 确定性滚动（按 userId+SALT 记忆） |
| `rollWithSeed(seed: string)` | `() => Roll` | 从任意 seed 字符串滚动 |
| `companionUserId()` | `() => string` | 返回 oauthAccount UUID 或 userID，或者 `'anon'` |
| `getCompanion()` | `() => Companion \| undefined` | 从 config + roll 组装 companion |

#### `buddy/sprites.ts`
| 导出 | 类型 | 说明 |
|--------|------|------|
| `renderSprite(bones, frame?)` | `(CompanionBones, number?) => string[]` | 返回 sprite 的行数组 |
| `spriteFrameCount(species)` | `(Species) => number` | 某个 species 的帧数（全部都是 3） |
| `renderFace(bones)` | `(CompanionBones) => string` | 用于内联显示的简短脸部字符串 |

#### `buddy/prompt.ts`
| 导出 | 类型 | 说明 |
|--------|------|------|
| `companionIntroText(name, species)` | `(string, string) => string` | companion 介绍用的 system prompt 文本 |
| `getCompanionIntroAttachment(messages?)` | `(Message[]?) => Attachment[]` | 如果尚未注入，则返回介绍附件 |

#### `buddy/useBuddyNotification.tsx`
| 导出 | 类型 | 说明 |
|--------|------|------|
| `isBuddyTeaserWindow()` | `() => boolean` | 当前本地日期是否为 2026 年 4 月 1–7 日 |
| `isBuddyLive()` | `() => boolean` | 当前本地日期是否为 2026 年 4 月或之后 |
| `useBuddyNotification()` | `() => void` | React hook：显示彩虹色 `/buddy` 启动提示 |
| `findBuddyTriggerPositions(text)` | `(string) => Array<{start,end}>` | 查找文本中的 `/buddy` 出现位置 |

### 1.9 配置

- `config.companion` - 存储 `StoredCompanion`（name、personality、hatchedAt）
- `config.companionMuted` - 为 true 时，停止 companion 介绍注入
- Feature flag：`feature('BUDDY')` - 控制所有 buddy 功能
- SALT 常量：`'friend-2026-401'` - 混入 hash，防止 replay attacks

---

## 2. Memory Directory（memdir）系统

### 2.1 系统概览

memory 系统让 Claude 在跨会话中保留持久化、基于文件的记忆。记忆以带 YAML frontmatter 的 markdown 文件形式存放在每个项目目录里。它包含：

1. **Auto memory**（`~/.claude/projects/<sanitized-git-root>/memory/`）- 每个用户、每个项目独立
2. **Team memory**（`<auto-mem-path>/team/`）- 供贡献者共享（feature gated）
3. **Memory scanning** - 只读 frontmatter header，先建 manifest，不加载全文
4. **Relevance selection** - 用 Sonnet 模型从候选里选出最相关的文件（最多 5 个）
5. **Freshness warnings** - 按时间老化情况注入 `<system-reminder>` 标签

**源目录：** `src/memdir/`

### 2.2 Memory Directory 结构

```text
~/.claude/
  projects/
    <sanitized-git-root>/
      memory/
        MEMORY.md           -- 入口索引（总是加载）
        <topic-file>.md     -- 带 frontmatter 的单独 memory 文件
        team/               -- 团队共享 memory（TEAMMEM feature）
          MEMORY.md
          <topic-file>.md
        logs/               -- KAIROS 模式：只追加的每日日志
          YYYY/
            MM/
              YYYY-MM-DD.md
```

### 2.3 Memory 类型分类

所有 memory 都属于四种类型之一（存储在 frontmatter 的 `type:` 字段中）：

| 类型 | 作用域（combined 模式） | 说明 |
|------|------------------------|------|
| `user` | 始终私有 | 用户的角色、目标、知识、偏好 |
| `feedback` | 默认私有；也可用于团队级约定 | 来自纠正和确认的工作方式建议 |
| `project` | 强烈偏向 team | 正在进行的工作、目标、bug、incident（无法从代码直接推导） |
| `reference` | 通常为 team | 指向外部系统（Linear、Grafana、Slack）的引用 |

**不该保存的内容：**
- 代码模式、架构、文件结构（读代码就能知道）
- Git 历史、最近变更（用 `git log`）
- 调试方案/修复配方（修复就在代码里）
- CLAUDE.md 中已有的内容
- 短期任务细节、进行中的工作

### 2.4 Memory Frontmatter 格式

```markdown
---
name: {{memory name}}
description: {{one-line description — 用于未来对话中判断相关性，所以要具体}}
type: {{user, feedback, project, reference}}
---

{{memory content}}
```

对于 `feedback` 和 `project` 类型，正文应包含：
- 以规则/事实开头
- 一行 `**Why:**`，说明原因
- 一行 `**How to apply:**`，说明何时/何地生效

### 2.5 架构：数据流

```text
用户查询进入
    │
    ▼
scanMemoryFiles(memoryDir)          -- 读取所有 .md 文件的 frontmatter
    │  returns MemoryHeader[]
    ▼
filterOut(alreadySurfaced)          -- 跳过之前轮次已经展示过的文件
    │
    ▼
selectRelevantMemories(query, ...)  -- 把 manifest 交给 Sonnet sideQuery
    │  returns up to 5 filenames
    ▼
返回 RelevantMemory[]（path + mtimeMs）
    │
    ▼
调用方注入文件内容        -- 读取完整文件内容
调用方添加新鲜度提醒      -- memoryFreshnessNote(mtimeMs)
```

### 2.6 Memory 扫描（`memoryScan.ts`）

**`scanMemoryFiles(memoryDir, signal)`**
- 使用 `readdir(memoryDir, { recursive: true })` 找到所有 `.md` 文件
- 排除 `MEMORY.md`（入口文件，已经在 system prompt 里）
- 每个文件只读取前 `FRONTMATTER_MAX_LINES = 30` 行
- 解析 `description` 和 `type` frontmatter
- 按 `mtimeMs` 从新到旧排序
- 上限 `MAX_MEMORY_FILES = 200`
- 返回 `MemoryHeader[]`

**`MemoryHeader` 类型：**
```typescript
export type MemoryHeader = {
  filename: string
  filePath: string
  mtimeMs: number
  description: string | null
  type: MemoryType | undefined
}
```

**`formatMemoryManifest(memories)`**
把 headers 格式化成给 selector prompt 用的文本 manifest：
```text
- [user] user_role.md (2026-01-15T12:00:00Z): user is a senior Go engineer focused on observability
- [feedback] no_mocks.md (2026-01-10T09:30:00Z): integration tests must use real database
```

### 2.7 相关性选择（`findRelevantMemories.ts`）

**选择用的 system prompt：**
```text
You are selecting memories that will be useful to Claude Code as it processes a user's query.
Return a list of filenames for the memories that will clearly be useful (up to 5).
Only include memories you are certain will be helpful. Be selective and discerning.
- If unsure, do not include it.
- If no memories are clearly useful, return an empty list.
- If recently-used tools are provided, do not select reference docs for those tools
  (skip API usage docs, DO keep warnings/gotchas about active tools).
```

**`sideQuery` 参数：**
- Model: `getDefaultSonnetModel()`
- Max tokens: 256
- 输出格式：JSON schema `{ selected_memories: string[] }`
- Query source: `'memdir_relevance'`

**`findRelevantMemories` signature:**
```typescript
export async function findRelevantMemories(
  query: string,
  memoryDir: string,
  signal: AbortSignal,
  recentTools: readonly string[] = [],
  alreadySurfaced: ReadonlySet<string> = new Set(),
): Promise<RelevantMemory[]>

export type RelevantMemory = {
  path: string
  mtimeMs: number
}
```

### 2.8 Memory Freshness 系统（`memoryAge.ts`）

所有函数都是纯函数，只接收 `mtimeMs` 时间戳：

| 导出 | 签名 | 说明 |
|--------|-----------|------|
| `memoryAgeDays(mtimeMs)` | `(number) => number` | 向下取整后的天数差，最小为 0 |
| `memoryAge(mtimeMs)` | `(number) => string` | 人类可读：`today`、`yesterday`、`N days ago` |
| `memoryFreshnessText(mtimeMs)` | `(number) => string` | 陈旧提示文本（≤1 天时为空；更旧时会警告） |
| `memoryFreshnessNote(mtimeMs)` | `(number) => string` | 用 `<system-reminder>` 包裹的 freshness 文本 |

**陈旧警告文本（适用于超过 1 天的 memory）：**
```text
This memory is N days old. Memories are point-in-time observations, not live state —
claims about code behavior or file:line citations may be outdated.
Verify against current code before asserting as fact.
```

### 2.9 Memory 路径解析（`paths.ts`）

**`isAutoMemoryEnabled()`** - 优先级链：
1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY` env var（truthy → OFF，falsy-defined → ON）
2. `CLAUDE_CODE_SIMPLE`（`--bare` mode）→ OFF
3. 没有 `CLAUDE_CODE_REMOTE_MEMORY_DIR` 的 `CLAUDE_CODE_REMOTE` → OFF
4. `settings.autoMemoryEnabled`（项目级 opt-out）
5. 默认：**enabled**

**`getAutoMemPath()`** - 解析顺序（由 `getProjectRoot()` memoized）：
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` env var（Cowork spaces）
2. settings 中的 `autoMemoryDirectory`（policy > flag > local > user sources；支持 `~/` 展开）
3. `<memoryBase>/projects/<sanitized-git-root>/memory/`
4. `memoryBase` = `CLAUDE_CODE_REMOTE_MEMORY_DIR` 或 `~/.claude`

**`validateMemoryPath()` 的安全校验：**
- 拒绝相对路径（非 `isAbsolute`）
- 拒绝 root / near-root（长度 < 3）
- 拒绝 Windows drive-root（`C:`）
- 拒绝 UNC 路径（`\\server\share`、`//`）
- 拒绝 null bytes
- 做 NFC 归一化

**`getAutoMemDailyLogPath(date?)`** - KAIROS 模式的每日日志路径：
```text
<autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md
```

**所有路径导出：**
| 导出 | 说明 |
|--------|------|
| `isAutoMemoryEnabled()` | 检查 auto-memory 是否开启 |
| `isExtractModeActive()` | 后台 memory 抽取 agent 是否会运行 |
| `getMemoryBaseDir()` | Base dir：env override 或 `~/.claude` |
| `getAutoMemPath()` | 完整 memory 目录路径（memoized） |
| `getAutoMemDailyLogPath(date?)` | KAIROS 模式的每日日志路径 |
| `getAutoMemEntrypoint()` | auto-mem 目录里的 `MEMORY.md` 路径 |
| `isAutoMemPath(absolutePath)` | 检查路径是否在 auto-memory 目录内 |
| `hasAutoMemPathOverride()` | 检查 Cowork env override 是否生效 |

### 2.10 Team Memory 路径（`teamMemPaths.ts`）

Team memory 放在 `<autoMemPath>/team/`。这里有很强的路径穿越保护：

**`PathTraversalError`** - 对注入尝试抛出的自定义错误类。

**`sanitizePathKey(key)`** - 拒绝：
- Null bytes
- URL 编码的穿越（如 `%2e%2e%2f`）
- Unicode 归一化攻击（全角 `．．／` → `../`）
- 反斜杠
- 绝对路径

**`validateTeamMemWritePath(filePath)`** - 两遍校验：
1. `path.resolve()` 消除 `..` 段，并检查字符串包含关系
2. `realpathDeepestExisting()` 跟随符号链接并验证真实包含关系

**`realpathDeepestExisting(absolutePath)`** - 向上遍历目录树直到 `realpath()` 成功，处理 dangling symlink（通过 `lstat` 检测）和 symlink loop（ELOOP）。

**所有导出：**
| 导出 | 说明 |
|--------|------|
| `PathTraversalError` | traversal 尝试的错误类 |
| `isTeamMemoryEnabled()` | GrowthBook gate `tengu_herring_clock` + auto-memory enabled |
| `getTeamMemPath()` | `<autoMemPath>/team/` |
| `getTeamMemEntrypoint()` | `<autoMemPath>/team/MEMORY.md` |
| `isTeamMemPath(filePath)` | 字符串级包含关系检查 |
| `isTeamMemFile(filePath)` | team enabled + 路径包含 |
| `validateTeamMemWritePath(filePath)` | 完整的 symlink-safe 写入校验 |
| `validateTeamMemKey(relativeKey)` | 先 sanitize 相对 key，再校验写入路径 |

### 2.11 Memory Prompt 构建（`memdir.ts`）

**常量：**
```typescript
export const ENTRYPOINT_NAME = 'MEMORY.md'
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000
```

**`truncateEntrypointContent(raw)`** - 先按行截断，再在字节上截断到最后一个换行之前，并追加说明消息，指出触发了哪些上限。

**`buildMemoryLines(displayName, memoryDir, extraGuidelines?, skipIndex?)`** - 构建行为指令，但不包含 MEMORY.md 内容。用于 system prompt。

**`buildMemoryPrompt({ displayName, memoryDir, extraGuidelines? })`** - 和 `buildMemoryLines` 类似，但包含 MEMORY.md 内容。用于 agent memory。

**`buildSearchingPastContextSection(autoMemDir)`** - 条件性添加 “Searching past context” 小节，并附带 grep 命令（受 `tengu_coral_fern` GrowthBook feature 控制）。

**`ensureMemoryDirExists(memoryDir)`** - 幂等 mkdir（recursive）；对非 EEXIST 错误只记录，不抛出。

**`loadMemoryPrompt()`** - 顶层分发器：
- KAIROS + kairosActive → `buildAssistantDailyLogPrompt()`
- TEAMMEM + team enabled → `buildCombinedMemoryPrompt()`
- Auto enabled → `buildMemoryLines()` joined
- 否则 → `null`

### 2.12 Memory 类型常量（`memoryTypes.ts`）

所有导出都用于 system prompt 构建：

| 导出 | 类型 | 说明 |
|--------|------|------|
| `MEMORY_TYPES` | `readonly string[]` | `['user', 'feedback', 'project', 'reference']` |
| `MemoryType` | type | 四种类型字符串的联合 |
| `parseMemoryType(raw)` | `(unknown) => MemoryType \| undefined` | 解析 frontmatter 值 |
| `TYPES_SECTION_COMBINED` | `readonly string[]` | 双目录模式的 prompt 小节 |
| `TYPES_SECTION_INDIVIDUAL` | `readonly string[]` | 单目录模式的 prompt 小节 |
| `WHAT_NOT_TO_SAVE_SECTION` | `readonly string[]` | memory 内容的排除列表 |
| `MEMORY_DRIFT_CAVEAT` | `string` | 关于 memory 陈旧性的单条警告 |
| `WHEN_TO_ACCESS_SECTION` | `readonly string[]` | 何时读取 memories |
| `TRUSTING_RECALL_SECTION` | `readonly string[]` | 从 memory 推荐内容前的校验指引 |
| `MEMORY_FRONTMATTER_EXAMPLE` | `readonly string[]` | prompt 里用的 frontmatter 示例块 |

### 2.13 Team Memory Prompts（`teamMemPrompts.ts`）

**`buildCombinedMemoryPrompt(extraGuidelines?, skipIndex?)`** - 当 auto memory 和 team memory 都启用时，构建组合 prompt。包含：
- 两个目录路径
- 两种作用域的说明（private vs team）
- 带 `<scope>` 标签的组合 `TYPES_SECTION_COMBINED`
- 双 MEMORY.md 索引说明

---

## 3. Keybindings 系统

### 3.1 系统概览

keybindings 系统提供了一层完全可配置的键盘快捷键。用户可以通过 `~/.claude/keybindings.json` 自定义绑定（受 `tengu_keybinding_customization_release` GrowthBook feature 控制）。系统支持：
- 单键绑定
- 多键 chord（例如 `ctrl+x ctrl+k`）
- 上下文作用域绑定（Chat、Global、Confirmation 等）
- null-unbinding（把 action 设为 `null` 来禁用默认绑定）
- 命令绑定（`command:help` 执行斜杠命令）
- 通过 `chokidar` 文件 watcher 热重载

**源目录：** `src/keybindings/`

### 3.2 架构

```text
DEFAULT_BINDINGS (KeybindingBlock[])
    │
    ▼ parseBindings()
ParsedBinding[]   <── 启动时解析默认绑定
    │
    ▼ 与用户绑定合并（user 追加在后面，所以 user 优先）
ParsedBinding[]   <── 合并后的绑定（同 key+context 时最后一个生效）
    │
    ▼
KeybindingProvider (React context)
    │
    ├── resolveKeyWithChordState()  ◄── 每次按键都会调用
    │   ├── chord_started → pending 状态
    │   ├── chord_cancelled → reset
    │   ├── match → 调用 action
    │   ├── unbound → 吞掉事件
    │   └── none → 透传
    │
    └── useKeybinding(action, context, handler)  ◄── 组件 hook
```

### 3.3 Keybinding 上下文

`KEYBINDING_CONTEXTS` 中定义了 18 个上下文：

| Context | 触发条件 |
|---------|----------|
| `Global` | 全局激活，不受焦点影响 |
| `Chat` | chat input 聚焦时 |
| `Autocomplete` | autocomplete 菜单可见时 |
| `Confirmation` | 显示确认/权限对话框时 |
| `Help` | help overlay 打开时 |
| `Transcript` | 查看 transcript 时 |
| `HistorySearch` | 搜索命令历史（ctrl+r）时 |
| `Task` | 前台有 task/agent 运行时 |
| `ThemePicker` | theme picker 打开时 |
| `Settings` | settings 菜单打开时 |
| `Tabs` | tab 导航激活时 |
| `Attachments` | 在选择对话框里浏览图片附件时 |
| `Footer` | 页脚指示器聚焦时 |
| `MessageSelector` | message selector（rewind）打开时 |
| `DiffDialog` | diff 对话框打开时 |
| `ModelPicker` | model picker 打开时 |
| `Select` | select/list 组件聚焦时 |
| `Plugin` | plugin 对话框打开时 |

### 3.4 默认绑定（`defaultBindings.ts`）

**平台相关按键：**
- `IMAGE_PASTE_KEY`：`alt+v`（Windows），`ctrl+v`（其他）
- `MODE_CYCLE_KEY`：`meta+m`（没有 VT mode 的 Windows），`shift+tab`（其他）
- VT mode 支持检查：Node ≥22.17.0 或 ≥24.2.0，Bun ≥1.2.23

**按上下文的完整默认绑定：**

**Global context：**
| Key | Action |
|-----|--------|
| `ctrl+c` | `app:interrupt`（不可重绑定） |
| `ctrl+d` | `app:exit`（不可重绑定） |
| `ctrl+l` | `app:redraw` |
| `ctrl+t` | `app:toggleTodos` |
| `ctrl+o` | `app:toggleTranscript` |
| `ctrl+shift+b` | `app:toggleBrief`（仅 KAIROS/KAIROS_BRIEF） |
| `ctrl+shift+o` | `app:toggleTeammatePreview` |
| `ctrl+r` | `history:search` |
| `ctrl+shift+f` / `cmd+shift+f` | `app:globalSearch`（QUICK_SEARCH feature） |
| `ctrl+shift+p` / `cmd+shift+p` | `app:quickOpen`（QUICK_SEARCH feature） |
| `meta+j` | `app:toggleTerminal`（TERMINAL_PANEL feature） |

**Chat context：**
| Key | Action |
|-----|--------|
| `escape` | `chat:cancel` |
| `ctrl+x ctrl+k` | `chat:killAgents`（chord） |
| `shift+tab` / `meta+m` | `chat:cycleMode` |
| `meta+p` | `chat:modelPicker` |
| `meta+o` | `chat:fastMode` |
| `meta+t` | `chat:thinkingToggle` |
| `enter` | `chat:submit` |
| `up` | `history:previous` |
| `down` | `history:next` |
| `ctrl+_` / `ctrl+shift+-` | `chat:undo` |
| `ctrl+x ctrl+e` / `ctrl+g` | `chat:externalEditor` |
| `ctrl+s` | `chat:stash` |
| `ctrl+v` / `alt+v` | `chat:imagePaste` |
| `shift+up` | `chat:messageActions`（MESSAGE_ACTIONS feature） |
| `space` | `voice:pushToTalk`（VOICE_MODE feature） |

**Autocomplete：**
| Key | Action |
|-----|--------|
| `tab` | `autocomplete:accept` |
| `escape` | `autocomplete:dismiss` |
| `up` / `down` | `autocomplete:previous` / `autocomplete:next` |

**Confirmation：**
| Key | Action |
|-----|--------|
| `y` / `enter` | `confirm:yes` |
| `n` / `escape` | `confirm:no` |
| `up` / `down` | `confirm:previous` / `confirm:next` |
| `tab` | `confirm:nextField` |
| `space` | `confirm:toggle` |
| `shift+tab` | `confirm:cycleMode` |
| `ctrl+e` | `confirm:toggleExplanation` |
| `ctrl+d` | `permission:toggleDebug` |

**Scroll：**
| Key | Action |
|-----|--------|
| `pageup` / `pagedown` | `scroll:pageUp` / `scroll:pageDown` |
| `wheelup` / `wheeldown` | `scroll:lineUp` / `scroll:lineDown` |
| `ctrl+home` / `ctrl+end` | `scroll:top` / `scroll:bottom` |
| `ctrl+shift+c` / `cmd+c` | `selection:copy` |

**Task：**
| Key | Action |
|-----|--------|
| `ctrl+b` | `task:background` |

### 3.5 Parser（`parser.ts`）

**`parseKeystroke(input: string): ParsedKeystroke`**
按 `+` 拆分，识别修饰键（大小写不敏感别名）：
- `ctrl` / `control`
- `alt` / `opt` / `option`
- `shift`
- `meta`
- `cmd` / `command` / `super` / `win`
- 特殊键：`esc`→`escape`，`return`→`enter`，`space`→` `，`↑↓←→`

**`parseChord(input: string): Chord`**
按空格分隔步骤，返回 `ParsedKeystroke[]`。特殊情况：单独的空格 `" "` 是空格键，而不是分隔符。

**`keystrokeToString(ks: ParsedKeystroke): string`**
规范字符串表示（内部使用）。

**`keystrokeToDisplayString(ks, platform?): string`**
平台友好的显示：macOS 上用 `opt`，其他平台用 `alt`；macOS 上用 `cmd`，其他平台用 `super`。

**`parseBindings(blocks: KeybindingBlock[]): ParsedBinding[]`**
把 `KeybindingBlock[]`（原始配置）转换为扁平的 `ParsedBinding[]`。

### 3.6 Matching（`match.ts`）

**`getKeyName(input: string, key: Key): string | null`**
把 Ink 的 boolean key flags 映射成字符串名称（如 `key.escape` → `'escape'`、`key.upArrow` → `'up'`、单字符转为小写等）。

**`matchesKeystroke(input, key, target): boolean`**
- 提取 Ink `Key` 对象里的 key name 和 modifiers
- **怪癖：** 在 Ink 中按下 escape 时，`key.meta = true`（旧终端行为）。匹配 escape 本身时会忽略这一点。
- Ink 把 alt 和 meta 当作别名处理（终端限制）：`target.alt || target.meta` 会匹配到 `key.meta`
- super/cmd 是独立的，只能通过 kitty keyboard protocol 到达

**`matchesBinding(input, key, binding): boolean`**
只匹配单键绑定（chord 长度必须为 1）。

### 3.7 Resolver（`resolver.ts`）

**`resolveKey(input, key, activeContexts, bindings): ResolveResult`**
单键解析的纯函数：
- 最后一个匹配的 binding 生效（用于 user overrides）
- 返回 `{ type: 'match', action }`、`{ type: 'none' }` 或 `{ type: 'unbound' }`

**`resolveKeyWithChordState(input, key, activeContexts, bindings, pending): ChordResolveResult`**
带状态的多键 chord 解析：
- chord 进行中时按 escape 会取消
- 通过 `chordToString(chord)` 对 chord 候选分组，正确处理 null-unbinding（chord 上的 null override 会阻止其前缀进入 chord-wait）
- 额外返回：`{ type: 'chord_started', pending }`、`{ type: 'chord_cancelled' }`

**`ChordResolveResult`:**
```typescript
type ChordResolveResult =
  | { type: 'match'; action: string }
  | { type: 'none' }
  | { type: 'unbound' }
  | { type: 'chord_started'; pending: ParsedKeystroke[] }
  | { type: 'chord_cancelled' }
```

**`getBindingDisplayText(action, context, bindings): string | undefined`**
按逆序搜索给定 action+context 的 binding（最后一个生效）。

**`keystrokesEqual(a, b): boolean`**
比较时把 alt/meta 合并成一个逻辑修饰键。

### 3.8 类型定义（`types.ts` — 推导）

```typescript
type KeybindingContextName = (typeof KEYBINDING_CONTEXTS)[number]

type ParsedKeystroke = {
  key: string
  ctrl: boolean
  alt: boolean
  shift: boolean
  meta: boolean
  super: boolean
}

type Chord = ParsedKeystroke[]

type ParsedBinding = {
  chord: Chord
  action: string | null
  context: KeybindingContextName
}

type KeybindingBlock = {
  context: string
  bindings: Record<string, string | null>
}
```

### 3.9 Schema（`schema.ts`）

用于 `keybindings.json` 校验的 Zod v4 schemas：

**`KeybindingBlockSchema`** - 校验 `{ context, bindings: { key: action | null } }`。bindings 接受：
- 来自 `KEYBINDING_ACTIONS` 的已知 action 字符串
- `command:` 前缀绑定（`/^command:[a-zA-Z0-9:\-_]+$/`）
- `null`（表示解绑）

**`KeybindingsSchema`** - 包含可选的 `$schema` 和 `$docs` 元数据：
```json
{
  "$schema": "https://www.schemastore.org/claude-code-keybindings.json",
  "$docs": "https://code.claude.com/docs/en/keybindings",
  "bindings": [...]
}
```

**完整 `KEYBINDING_ACTIONS` 列表（80 项）：**
分类包括：`app:*`、`history:*`、`chat:*`、`autocomplete:*`、`confirm:*`、`tabs:*`、`transcript:*`、`historySearch:*`、`task:*`、`theme:*`、`help:*`、`attachments:*`、`footer:*`、`messageSelector:*`、`messageActions:*`、`diff:*`、`modelPicker:*`、`select:*`、`plugin:*`、`permission:*`、`settings:*`、`voice:*`

### 3.10 Validation（`validate.ts`）

**`KeybindingWarningType`:** `'parse_error' | 'duplicate' | 'reserved' | 'invalid_context' | 'invalid_action'`

**`KeybindingWarning`:**
```typescript
type KeybindingWarning = {
  type: KeybindingWarningType
  severity: 'error' | 'warning'
  message: string
  key?: string
  context?: string
  action?: string
  suggestion?: string
}
```

**校验检查：**
1. `validateUserConfig(blocks)` - 校验 block 结构（context 字符串、bindings 对象）
2. `checkDuplicates(blocks)` - 同一 context 内的重复键（归一化后比较）
3. `checkReservedShortcuts(bindings)` - 对照 `NON_REBINDABLE` 和平台级 `TERMINAL_RESERVED` / `MACOS_RESERVED`
4. `checkDuplicateKeysInJson(jsonString)` - 对原始 JSON 字符串做扫描（JSON.parse 会静默丢弃前面的重复键）
5. 特例：`voice:pushToTalk` 绑定到裸字母键会给出警告（暖机期间会把字符打进输入框）
6. `command:` 绑定必须在 `Chat` context 中

### 3.11 Reserved Shortcuts（`reservedShortcuts.ts`）

**`NON_REBINDABLE`**（error severity）：
- `ctrl+c` - interrupt/exit（硬编码双击逻辑）
- `ctrl+d` - exit（硬编码）
- `ctrl+m` - 在终端中等同于 Enter（两者都会发送 CR）

**`TERMINAL_RESERVED`:**
- `ctrl+z` - Unix SIGTSTP（warning）
- `ctrl+\` - 终端 SIGQUIT（error）

**`MACOS_RESERVED`**（仅 macOS，全部是 error）：
- `cmd+c/v/x/q/w/tab/space`

注：`ctrl+s`（XOFF）和 `ctrl+q`（XON）故意不在 reserved list 里，因为现代终端会关闭 flow control，而且 Claude Code 还把 `ctrl+s` 用作 stash。

### 3.12 User Bindings Loader（`loadUserBindings.ts`）

**文件位置：** `~/.claude/keybindings.json`

**`isKeybindingCustomizationEnabled()`** - GrowthBook gate：`tengu_keybinding_customization_release`。

**文件监听：** 使用 `chokidar`，并设置：
- `stabilityThreshold: 500ms` - 等待写入稳定
- `pollInterval: 200ms`
- `atomic: true`
- 监听父目录，而不只是文件本身（处理文件创建）

**`loadKeybindings()` / `loadKeybindingsSyncWithWarnings()`** - 读取 + 合并 + 校验。返回 `{ bindings: ParsedBinding[], warnings: KeybindingWarning[] }`。

**所有导出：**
| 导出 | 说明 |
|--------|------|
| `isKeybindingCustomizationEnabled()` | GrowthBook gate 检查 |
| `getKeybindingsPath()` | `~/.claude/keybindings.json` |
| `loadKeybindings()` | 异步加载（watcher 使用） |
| `loadKeybindingsSync()` | 同步加载（React useState initializer） |
| `loadKeybindingsSyncWithWarnings()` | 带 warnings 的同步加载 |
| `initializeKeybindingWatcher()` | 设置 chokidar watcher |
| `disposeKeybindingWatcher()` | 清理 watcher |
| `subscribeToKeybindingChanges` | 订阅 reload 事件 |
| `getCachedKeybindingWarnings()` | 当前缓存的 warnings |
| `resetKeybindingLoaderForTesting()` | 测试重置 |

### 3.13 Template Generator（`template.ts`）

**`generateKeybindingsTemplate()`** - 生成一个文档完善的模板 JSON 文件，包含全部默认绑定。会过滤掉 `NON_REBINDABLE` 快捷键，避免 `/doctor` 警告。包含 `$schema` 和 `$docs` 元数据。

### 3.14 Shortcut Display（`shortcutFormat.ts`）

**`getShortcutDisplay(action, context, fallback)`** - 非 React 的快捷键显示辅助函数。若绑定未找到，会记录一次 `tengu_keybinding_fallback_used` 分析事件（按 action+context 只记录一次）。迁移期间会回退到硬编码字符串。

---

## 4. Skills 系统

### 4.1 系统概览

Skills 是 Claude 的斜杠命令，执行 AI prompt 或本地逻辑。它们来自多个来源：
- **Bundled skills** - 编译进二进制，所有用户都可用
- **Disk-based skills** - `.claude/skills/` 目录里的 markdown 文件
- **Plugin skills** - 已安装插件提供
- **MCP skills** - 由 MCP server tool 定义生成

**源目录：** `src/skills/`

### 4.2 内置技能注册表（`bundledSkills.ts`）

**`BundledSkillDefinition`** type:
```typescript
type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}
```

**文件抽取（`files` 字段）：** 当 skill 带有 `files` 时，它们会在第一次调用时被抽取到每个进程独立的 nonce 目录里，使用 `O_NOFOLLOW | O_EXCL | O_CREAT` 标志（目录权限 0o700，文件权限 0o600）。在 skill prompt 前会加上 `Base directory for this skill: <dir>`，这样模型就能读取这些文件。

**Registry API：**
| 导出 | 说明 |
|--------|------|
| `registerBundledSkill(definition)` | 启动时注册 bundled skill |
| `getBundledSkills()` | 获取所有已注册 bundled skills 的副本 |
| `clearBundledSkills()` | 清空 registry（测试用） |
| `getBundledSkillExtractDir(skillName)` | 确定性的抽取目录 |

### 4.3 内置技能初始化（`bundled/index.ts`）

**`initBundledSkills()`** 注册以下 skills：

| 技能 | 功能门控 | 说明 |
|-------|-------------|------|
| `updateConfig` | — | 更新 Claude Code 配置 |
| `keybindings` | — | 管理键盘快捷键 |
| `verify` | — | 验证工作 / 运行检查 |
| `debug` | — | 调试辅助 |
| `loremIpsum` | — | 生成 lorem ipsum 文本 |
| `skillify` | — | 创建新 skills |
| `remember` | — | 显式保存到 memory |
| `simplify` | — | 简化代码/文本 |
| `batch` | — | 批量操作 |
| `stuck` | — | 卡住时的帮助 |
| `dream` | `KAIROS \| KAIROS_DREAM` | 每夜 memory 提炼 |
| `hunter` | `REVIEW_ARTIFACT` | 代码审查 |
| `loop` | `AGENT_TRIGGERS` | 周期性 agent 触发 |
| `scheduleRemoteAgents` | `AGENT_TRIGGERS_REMOTE` | 安排远程 agent 运行 |
| `claudeApi` | `BUILDING_CLAUDE_APPS` | Claude API 参考 skill |
| `claudeInChrome` | `shouldAutoEnableClaudeInChrome()` | Claude in Chrome 集成 |
| `runSkillGenerator` | `RUN_SKILL_GENERATOR` | skill 生成工具 |

### 4.4 技能目录加载器（`loadSkillsDir.ts`）

**`LoadedFrom` type:**
```typescript
type LoadedFrom =
  | 'commands_DEPRECATED'
  | 'skills'
  | 'plugin'
  | 'managed'
  | 'bundled'
  | 'mcp'
```

**`getSkillsPath(source, dir)`** - 返回给定 source 和子目录（`skills` 或 `commands`）对应的配置目录路径。

loader 处理：
- frontmatter 解析（`name`、`description`、`argument-hint`、`when-to-use`、`allowed-tools`、`model`、`disableNonInteractive`、`hooks`、`context`、`agent`、`effort`、`paths`）
- prompt 里的 shell 执行（frontmatter 中的反引号命令）
- gitignore 集成
- 参数替换（`$ARGUMENTS`、命名参数 `$ARG_NAME`）
- `isRestrictedToPluginOnly()` policy 检查
- 多级发现：从项目目录一路到 home + user config

---

## 5. Voice 系统

### 5.1 系统概览

Voice mode 让 Claude Code 支持按住说话式交互（push-to-talk）。启用 voice 需要：
1. OAuth 认证（使用 claude.ai 的 `voice_stream` endpoint）
2. GrowthBook feature flag `VOICE_MODE`
3. kill-switch gate：`tengu_amber_quartz_disabled`（负向 flag）

**源文件：** `src/voice/voiceModeEnabled.ts`

### 5.2 API

| 导出 | 说明 |
|--------|------|
| `isVoiceGrowthBookEnabled()` | `feature('VOICE_MODE') && !tengu_amber_quartz_disabled` |
| `hasVoiceAuth()` | OAuth provider + 有效 access token 存在 |
| `isVoiceModeEnabled()` | 完整检查：`hasVoiceAuth() && isVoiceGrowthBookEnabled()` |

**为什么需要 OAuth：**
- Voice 使用 claude.ai 的 `voice_stream` endpoint
- API keys、Bedrock、Vertex、Foundry 都不支持
- `isAnthropicAuthEnabled()` 检查 provider，`getClaudeAIOAuthTokens()` 验证 token 确实存在

**Kill-switch 设计：**
- flag 默认值是 `false` = 没有被 kill
- 如果磁盘缓存缺失或过期，会被当作 “not killed” → 新安装立即可用
- 紧急关闭：把 `tengu_amber_quartz_disabled` 改成 `true`

**默认绑定：** `space` → `voice:pushToTalk`（在 `defaultBindings.ts` 的 VOICE_MODE feature gate 下定义）

**校验警告：** 把 `voice:pushToTalk` 绑定到裸字母键（没有修饰键）会给出警告，因为在激活 warmup 期间字符会直接打进输入框。推荐使用 `space` 或像 `meta+k` 这样的组合键。

---

## 6. Plugins 系统

### 6.1 系统概览

Plugins 会给 Claude Code 增加额外的 skills、hooks、MCP servers、LSP servers 和 output styles。插件分为两类：
1. **Built-in plugins** - 随 CLI 一起发布，会出现在 `/plugin` UI 里，用户可开关
2. **Marketplace plugins** - 通过 `/plugin install` 从 GitHub 仓库安装

**源文件：** `src/plugins/`

### 6.2 内置插件注册表（`builtinPlugins.ts`）

Built-in plugins 使用 `{name}@builtin` 作为 ID 格式。

**`BuiltinPluginDefinition`** type（来自 `types/plugin.ts`）:
```typescript
type BuiltinPluginDefinition = {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  isAvailable?: () => boolean
  defaultEnabled?: boolean
}
```

**`LoadedPlugin`** type:
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

**`getBuiltinPlugins()`** - 返回 `{ enabled: LoadedPlugin[], disabled: LoadedPlugin[] }`。启用状态优先级：用户设置 > `defaultEnabled` > `true`。若 `isAvailable() === false`，则整个插件会被省略。

**所有导出：**
| 导出 | 说明 |
|--------|------|
| `BUILTIN_MARKETPLACE_NAME` | `'builtin'` - 哨兵 marketplace 名称 |
| `registerBuiltinPlugin(definition)` | 启动时注册 |
| `isBuiltinPluginId(pluginId)` | 检查是否以 `@builtin` 结尾 |
| `getBuiltinPluginDefinition(name)` | 按名称获取定义 |
| `getBuiltinPlugins()` | 获取 enabled/disabled 拆分 |
| `getBuiltinPluginSkillCommands()` | 把 enabled plugins 里的 skills 作为 Commands 取出 |
| `clearBuiltinPlugins()` | 测试重置 |

### 6.3 内置插件初始化（`bundled/index.ts`）

```typescript
export function initBuiltinPlugins(): void {
  // 目前还没有注册 built-in plugins——这里只是给后续迁移
  // 那些应当可由用户开关的 bundled skills 留个脚手架。
}
```

这个文件只是脚手架——从当前代码快照来看，还没有注册任何 built-in plugins。基础设施已经准备好，可以把 bundled skills 迁移成可开关的 plugin 系统。

### 6.4 插件错误类型（`types/plugin.ts`）

`PluginError` 是一个很大的 discriminated union，包含 20+ 种错误变体：

| Type | Key fields |
|------|-----------|
| `path-not-found` | `path`, `component` |
| `git-auth-failed` | `gitUrl`, `authType: 'ssh' \| 'https'` |
| `git-timeout` | `gitUrl`, `operation: 'clone' \| 'pull'` |
| `network-error` | `url`, `details?` |
| `manifest-parse-error` | `manifestPath`, `parseError` |
| `manifest-validation-error` | `manifestPath`, `validationErrors[]` |
| `plugin-not-found` | `pluginId`, `marketplace` |
| `marketplace-not-found` | `marketplace`, `availableMarketplaces[]` |
| `marketplace-load-failed` | `marketplace`, `reason` |
| `mcp-config-invalid` | `serverName`, `validationError` |
| `mcp-server-suppressed-duplicate` | `serverName`, `duplicateOf` |
| `lsp-config-invalid` | `serverName`, `validationError` |
| `lsp-server-start-failed` | `serverName`, `reason` |
| `lsp-server-crashed` | `exitCode`, `signal?` |
| `lsp-request-timeout` | `method`, `timeoutMs` |
| `lsp-request-failed` | `method`, `error` |
| `marketplace-blocked-by-policy` | `marketplace`, `blockedByBlocklist?`, `allowedSources[]` |
| `dependency-unsatisfied` | `dependency`, `reason: 'not-enabled' \| 'not-found'` |
| `plugin-cache-miss` | `installPath` |
| `hook-load-failed` | `hookPath`, `reason` |
| `component-load-failed` | `component`, `path`, `reason` |
| `mcpb-download-failed` | `url`, `reason` |
| `mcpb-extract-failed` | `mcpbPath`, `reason` |
| `mcpb-invalid-manifest` | `mcpbPath`, `validationError` |
| `generic-error` | `error` |

**`getPluginErrorMessage(error)`** - 为任意 `PluginError` 变体返回人类可读字符串。

**`PluginLoadResult`:**
```typescript
type PluginLoadResult = {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
  errors: PluginError[]
}
```

---

## 7. 输出样式

### 7.1 系统概览

输出样式是 markdown 文件，用来定义 Claude 回复的自定义格式指令。它们从项目 `.claude/` 和用户 `~/.claude/` 目录下的 `output-styles/` 子目录加载。

**源文件：** `src/outputStyles/loadOutputStylesDir.ts`

### 7.2 配置

- **位置：** `.claude/output-styles/*.md`（project）和 `~/.claude/output-styles/*.md`（user）
- **文件命名：** `filename.md` → style name `filename`
- **Frontmatter 字段：**
  - `name`: 覆盖 style 名称（默认使用去掉 `.md` 的文件名）
  - `description`: 在 output style picker 中显示的描述
  - `keep-coding-instructions`: 布尔值，是否保留 coding 专用指令（`true`/`false` 或 boolean）
  - `force-for-plugin`: 只对 plugin output styles 有效；在普通 styles 上会被忽略并给出警告

### 7.3 API

**`getOutputStyleDirStyles(cwd: string): Promise<OutputStyleConfig[]>`** - 带 memo 的异步加载器。会扫描项目层级和用户配置里的 `output-styles` 子目录。返回 `OutputStyleConfig[]`。

**`OutputStyleConfig`**（来自 `constants/outputStyles.ts`）：
```typescript
type OutputStyleConfig = {
  name: string
  description: string
  prompt: string
  source: string
  keepCodingInstructions?: boolean
}
```

**`clearOutputStyleCaches()`** - 清空带 memo 的 `getOutputStyleDirStyles`、`loadMarkdownFilesForSubdir` 和 plugin output style 缓存。

---

## 8. Hooks 模式

### 8.1 系统概览

Hooks 会在特定生命周期事件上执行副作用（PreToolUse、PostToolUse、PostResponse 等）。Schema 通过 discriminated union 定义四种 hook 类型，并可通过 `if` 条件做条件执行。

**源文件：** `src/schemas/hooks.ts`

### 8.2 Hook 类型

#### `BashCommandHook`（`type: 'command'`）
```typescript
{
  type: 'command'
  command: string
  if?: string
  shell?: 'bash' | 'powershell'
  timeout?: number
  statusMessage?: string
  once?: boolean
  async?: boolean
  asyncRewake?: boolean
}
```

#### `PromptHook`（`type: 'prompt'`）
```typescript
{
  type: 'prompt'
  prompt: string
  if?: string
  timeout?: number
  model?: string
  statusMessage?: string
  once?: boolean
}
```

#### `HttpHook`（`type: 'http'`）
```typescript
{
  type: 'http'
  url: string
  if?: string
  timeout?: number
  headers?: Record<string, string>
  allowedEnvVars?: string[]
  statusMessage?: string
  once?: boolean
}
```

#### `AgentHook`（`type: 'agent'`）
```typescript
{
  type: 'agent'
  prompt: string
  if?: string
  timeout?: number
  model?: string
  statusMessage?: string
  once?: boolean
}
```

**关于 `AgentHook` 的注意事项：** Zod schema 里绝对不能用 `.transform()`，因为这个 schema 会被 `parseSettingsFile` 使用；如果把经过 transform 的函数再经过 `JSON.stringify` 做 round-trip，`prompt` 字段会静默丢失。

### 8.3 Hook Matcher 和 Settings 结构

```typescript
type HookMatcher = {
  matcher?: string
  hooks: HookCommand[]
}

type HooksSettings = Partial<Record<HookEvent, HookMatcher[]>>
```

**`IfConditionSchema`：** 共享的 `if` 字段使用 permission rule 语法（`Bash(git *)`、`Read(*.ts)`）在 hook 启动前过滤是否执行。它会针对 `tool_name` 和 `tool_input` 进行求值。

### 8.4 导出

| 导出 | 说明 |
|--------|------|
| `HookCommandSchema` | 四种 hook command 类型的 discriminated union |
| `HookMatcherSchema` | `{ matcher?, hooks[] }` |
| `HooksSchema` | `Partial<Record<HookEvent, HookMatcher[]>>` |
| `HookCommand` | 从 schema 推导出的类型 |
| `BashCommandHook` | `Extract<HookCommand, { type: 'command' }>` |
| `PromptHook` | `Extract<HookCommand, { type: 'prompt' }>` |
| `AgentHook` | `Extract<HookCommand, { type: 'agent' }>` |
| `HttpHook` | `Extract<HookCommand, { type: 'http' }>` |
| `HookMatcher` | 从 matcher schema 推导出的类型 |
| `HooksSettings` | `Partial<Record<HookEvent, HookMatcher[]>>` |

---

## 9. Native-TypeScript 端口（native-ts/）

`native-ts/` 目录里放的是 Rust NAPI native modules 的纯 TypeScript port。当原生模块无法加载时（比如没有预编译二进制的平台），会使用这些实现。

### 9.1 颜色差异（`native-ts/color-diff/index.ts`）

**用途：** `vendor/color-diff-src` 的 port，用于 diff 视图里的 syntax-highlighted word-level diff 渲染。

**实现：** 使用 `highlight.js`（lazy-loaded，推迟约 50MB 的 grammar 注册）和 `diff` npm 包的 `diffArrays`。

**与 native 的语义差异：**
- 使用 highlight.js 做语法高亮，而不是 syntect/bat
- scope 颜色只是近似 syntect 的输出，但普通标识符和 `=` `:` 运算符会按默认前景色显示（hljs grammar 里没有 scope）
- `BAT_THEME` env 只是 stub（总是返回默认主题）
- 输出结构（行号、marker、背景、word-diff）与 native 完全一致

**Lazy loading 模式：**
```typescript
let cachedHljs: HLJSApi | null = null
function hljs(): HLJSApi {
  if (cachedHljs) return cachedHljs
  const mod = require('highlight.js')
  cachedHljs = 'default' in mod && mod.default ? mod.default : mod
  return cachedHljs!
}
```

**关键类型：**
```typescript
type Hunk = {
  oldStart: number
  oldLines: number
  newStart: number
  newLines: number
  lines: string[]
}

type SyntaxTheme = { ... }
```

### 9.2 文件索引（`native-ts/file-index/index.ts`）

**用途：** `vendor/file-index-src` 的 port，用于高性能 fuzzy file path 搜索，替代 nucleo（Rust）。

**算法：** 近似 fzf-v2/nucleo 打分，包含：
- Bitmap reject：路径里缺少任意 needle 字母就 O(1) 拒绝
- 通过 `indexOf` 做贪心最早位置扫描（JSC/V8 中 SIMD 加速）
- 打分常量：

| 常量 | 值 | 说明 |
|----------|-------|------|
| `SCORE_MATCH` | 16 | 每个匹配字符的基础分 |
| `BONUS_BOUNDARY` | 8 | 路径边界匹配奖励（`/ \ - _ . space`） |
| `BONUS_CAMEL` | 6 | camelCase 边界奖励 |
| `BONUS_CONSECUTIVE` | 4 | 连续匹配奖励 |
| `BONUS_FIRST_CHAR` | 8 | 匹配首字符奖励 |
| `PENALTY_GAP_START` | 3 | gap 开始惩罚 |
| `PENALTY_GAP_EXTENSION` | 1 | 每个 gap 字符的惩罚 |

**测试文件惩罚：** 路径里包含 `"test"` 会有 1.05x 的分数惩罚（上限仍然是 1.0）。

**分数语义：** 越小越好。`score = position_in_results / result_count`，所以最佳匹配是 0.0。

**大小写智能匹配：** 全小写 query → 不区分大小写；只要有大写 → 区分大小写。

**`FileIndex` class：**
```typescript
class FileIndex {
  loadFromFileList(fileList: string[]): void
  loadFromFileListAsync(fileList: string[]): { queryable: Promise<void>; done: Promise<void> }
  search(query: string, limit: number): SearchResult[]
}

type SearchResult = {
  path: string
  score: number
}
```

**异步加载：** `loadFromFileListAsync` 每进行 `CHUNK_MS = 4ms` 的同步工作就让出事件循环。完成第一块后就会 resolve `queryable`（可立即搜索部分结果）。对于 27 万路径列表，到第一个可查询状态大约需要 5–10ms。

**顶层缓存：** 保存最常见的 100 个顶层路径段，用于空查询结果。

**`TOP_LEVEL_CACHE_LIMIT = 100`**，**`MAX_QUERY_LEN = 64`**

### 9.3 Yoga 布局（`native-ts/yoga-layout/`）

**用途：** Meta Yoga flexbox 引擎（`yoga-layout/load`）的纯 TypeScript port，用于 Ink 的布局引擎。

**`enums.ts` - Yoga 常量以 `const` 对象形式提供：**

| 枚举 | 取值 |
|------|------|
| `Align` | Auto(0), FlexStart(1), Center(2), FlexEnd(3), Stretch(4), Baseline(5), SpaceBetween(6), SpaceAround(7), SpaceEvenly(8) |
| `BoxSizing` | BorderBox(0), ContentBox(1) |
| `Dimension` | Width(0), Height(1) |
| `Direction` | Inherit(0), LTR(1), RTL(2) |
| `Display` | Flex(0), None(1), Contents(2) |
| `Edge` | Left(0), Top(1), Right(2), Bottom(3), ... |
| `Errata` | None(0), StretchFlexBasis(1), All(2147483647) |
| `ExperimentalFeature` | WebFlexBasis(0) |
| `FlexDirection` | Column(0), ColumnReverse(1), Row(2), RowReverse(3) |
| `Gutter` | Column(0), Row(1), All(2) |
| `Justify` | FlexStart(0), Center(1), FlexEnd(2), SpaceBetween(3), SpaceAround(4), SpaceEvenly(5) |
| `MeasureMode` | Undefined(0), Exactly(1), AtMost(2) |
| `NodeType` | Default(0), Text(1) |
| `Overflow` | Visible(0), Hidden(1), Scroll(2) |
| `PositionType` | Static(0), Relative(1), Absolute(2) |
| `Unit` | Undefined(0), Point(1), Percent(2), Auto(3) |
| `Wrap` | NoWrap(0), Wrap(1), WrapReverse(2) |

**`index.ts` - 已实现的 Yoga 子集：**
- flex-direction（row/column 及 reverse）
- flex-grow / flex-shrink / flex-basis
- align-items / align-self（实际上除了 baseline 以外都支持）
- justify-content（6 个值全支持）
- margin / padding / border / gap
- width / height / min / max（point、percent、auto）
- position：relative / absolute
- display：flex / none / contents
- measure functions（文本节点）
- margin: auto
- flex-wrap + align-content
- baseline alignment（与 spec 保持一致，但 Ink 里不会用到）

**未实现：** aspect-ratio、box-sizing: content-box、RTL direction

---

## 10. MoreRight Hook（moreright/）

### 10.1 系统概览

`useMoreRight` 是一个 React hook，作为 **外部构建的 stub** 存在。真正的实现只对内部用户开放。这个 stub 返回 hook 接口的 no-op 实现。

**源文件：** `src/moreright/useMoreRight.tsx`

### 10.2 接口

```typescript
export function useMoreRight(_args: {
  enabled: boolean
  setMessages: (action: M[] | ((prev: M[]) => M[])) => void
  inputValue: string
  setInputValue: (s: string) => void
  setToolJSX: (args: M) => void
}): {
  onBeforeQuery: (input: string, all: M[], n: number) => Promise<boolean>
  onTurnComplete: (all: M[], aborted: boolean) => Promise<void>
  render: () => null
}
```

**Stub 行为：**
- `onBeforeQuery` 总是返回 `true`（允许查询）
- `onTurnComplete` 是 no-op
- `render` 返回 `null`

内部真实实现提供的是 query 前处理和 turn 后处理功能；这些细节在外部构建中被隐藏。

---

## 11. 迁移

migrations 会在启动时运行，用来把存储的 config/settings 更新成当前格式。所有 migration 都是幂等的（可安全重复执行）。

### 11.1 迁移概览

| Migration | Source | Target | Condition |
|-----------|--------|--------|-----------|
| `migrateAutoUpdatesToSettings` | `globalConfig.autoUpdates: false` | `userSettings.env.DISABLE_AUTOUPDATER: '1'` | 仅在用户偏好禁用（不是 native protection）时 |
| `migrateBypassPermissionsAcceptedToSettings` | `globalConfig.bypassPermissionsModeAccepted` | `userSettings.skipDangerousModePermissionPrompt: true` | globalConfig 字段存在时 |
| `migrateEnableAllProjectMcpServersToSettings` | `projectConfig.enableAllProjectMcpServers` 等 | `localSettings` | 这 3 个字段任意一个存在时 |
| `migrateFennecToOpus` | `userSettings.model` 的 fennec aliases | Opus 4.6 aliases | 仅限 ant 用户 |
| `migrateLegacyOpusToCurrent` | 显式 Opus 4.0/4.1 model 字符串 | `'opus'` alias | 1P provider + legacy remap enabled |
| `migrateOpusToOpus1m` | `userSettings.model: 'opus'` | `'opus[1m]'` | `isOpus1mMergeEnabled()` |
| `migrateReplBridgeEnabledToRemoteControlAtStartup` | `globalConfig.replBridgeEnabled` | `globalConfig.remoteControlAtStartup` | 旧 key 存在、新 key 不存在时 |
| `migrateSonnet1mToSonnet45` | `userSettings.model: 'sonnet[1m]'` | `'sonnet-4-5-20250929[1m]'` | 仅一次（`sonnet1m45MigrationComplete` flag） |
| `migrateSonnet45ToSonnet46` | 显式 Sonnet 4.5 字符串 | `'sonnet'` 或 `'sonnet[1m]'` | 1P + Pro/Max/TeamPremium |
| `resetAutoModeOptInForDefaultOffer` | `userSettings.skipAutoPermissionPrompt` | 清除该设置 | TRANSCRIPT_CLASSIFIER + 指定条件 |
| `resetProToOpusDefault` | 默认 model 设置 | 写入 migration timestamp | 1P Pro 用户 |

### 11.2 模型命名演进（由 migrations 还原）

migration 历史揭示了完整的 model codename / alias 演进：

**Opus 系列：**
1. `fennec-latest` - 早期 Opus 4.x 的内部 codename
2. `claude-opus-4-0` / `claude-opus-4-20250514` - Opus 4.0 发布
3. `claude-opus-4-1` / `claude-opus-4-1-20250805` - Opus 4.1 发布
4. `opus-4-5-fast` - Opus 4.5 fast 变体
5. `opus` - 当前 Opus 4.6 alias
6. `opus[1m]` - 带 1M context window 的 Opus 4.6

**Sonnet 系列：**
1. `sonnet[1m]` - 早期 1M context 的 Sonnet（目标是 Sonnet 4.5）
2. `claude-sonnet-4-5-20250929` / `sonnet-4-5-20250929` - 明确的 Sonnet 4.5
3. `claude-sonnet-4-5-20250929[1m]` / `sonnet-4-5-20250929[1m]` - 带 1M 的 Sonnet 4.5
4. `sonnet` - 当前 Sonnet 4.6 alias
5. `sonnet[1m]` - 带 1M context 的 Sonnet 4.6（重新 alias）

**内部 codename：**
- `fennec-latest` → Opus 4.x 系列（预发布内部名）
- `fennec-fast-latest` → Opus fast 变体

### 11.3 迁移细节

#### `migrateAutoUpdatesToSettings`
- 只在 `globalConfig.autoUpdates === false` 且 `autoUpdatesProtectedForNative !== true` 时迁移
- 向 `userSettings.env` 添加 `DISABLE_AUTOUPDATER: '1'`
- 立即设置 `process.env.DISABLE_AUTOUPDATER = '1'`
- 从 globalConfig 中删除 `autoUpdates` 和 `autoUpdatesProtectedForNative`

#### `migrateFennecToOpus`（仅 ant）
alias 映射：
- `fennec-latest[1m]` → `opus[1m]`
- `fennec-latest` → `opus`
- `fennec-fast-latest` / `opus-4-5-fast` → `opus[1m]` + `fastMode: true`

#### `migrateLegacyOpusToCurrent`（仅 1P）
迁移：`claude-opus-4-20250514`、`claude-opus-4-1-20250805`、`claude-opus-4-0`、`claude-opus-4-1` → `'opus'`
并设置 `legacyOpusMigrationTimestamp`，用于一次性通知。

#### `migrateOpusToOpus1m`
对于符合条件的 Max/Team Premium 1P 用户，将 `'opus'` → `'opus[1m]'`。
这是幂等的：只有当 `userSettings.model === 'opus'` 时才会生效。
如果迁移后的值等于当前默认值，就会清空 `model` setting。

#### `migrateSonnet1mToSonnet45`
只运行一次（由 globalConfig 里的 `sonnet1m45MigrationComplete` 保护）。
如果存在，还会迁移内存中的 `mainLoopModelOverride`。

#### `migrateSonnet45ToSonnet46`
迁移：
- `claude-sonnet-4-5-20250929` / `sonnet-4-5-20250929` → `'sonnet'`
- `claude-sonnet-4-5-20250929[1m]` / `sonnet-4-5-20250929[1m]` → `'sonnet[1m]'`
并设置 `sonnet45To46MigrationTimestamp` 用于通知（新用户 `numStartups <= 1` 时跳过）。

#### `migrateReplBridgeEnabledToRemoteControlAtStartup`
把 `replBridgeEnabled`（内部实现细节）重命名为 `remoteControlAtStartup`（用户可见 key）。只在旧 key 存在而新 key 不存在时执行。

#### `resetAutoModeOptInForDefaultOffer`
一次性操作（由 `hasResetAutoModeOptInForDefaultOffer` flag 保护）。
对那些接受过旧的两选项对话框、但自动模式并非默认模式的用户，清除 `skipAutoPermissionPrompt`，让新的对话框重新出现，并带上 “make it my default” 选项。
只在 `getAutoModeEnabledState() === 'enabled'` 时运行（不是 `opt-in`）。

#### `resetProToOpusDefault`
为 1P 上的 Pro 订阅者设置 migration timestamp，提示 UI：Opus 现在是默认模型。对其他用户则只是标记迁移完成。

---

## 12. 核心类型定义（types/）

### 12.1 带品牌的 ID 类型（`types/ids.ts`）

防止在编译期混用 SessionId 和 AgentId：

```typescript
type SessionId = string & { readonly __brand: 'SessionId' }
type AgentId = string & { readonly __brand: 'AgentId' }
```

**AgentId 格式：** `a` + 可选 `<label>-` + 16 个 hex 字符
Pattern: `/^a(?:.+-)?[0-9a-f]{16}$/`

| 导出 | 说明 |
|--------|------|
| `SessionId` | branded session ID type |
| `AgentId` | branded agent ID type |
| `asSessionId(id)` | 把字符串 cast 成 SessionId |
| `asAgentId(id)` | 把字符串 cast 成 AgentId |
| `toAgentId(s)` | 校验并 brand 为 AgentId，或者返回 null |

### 12.2 命令类型（`types/command.ts`）

**`PromptCommand`** - 会生成 prompt 的 skill/command：
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
  context?: 'inline' | 'fork'
  agent?: string
  effort?: EffortValue
  paths?: string[]
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>
}
```

**执行上下文：**
- `'inline'`（默认）：skill 内容扩展进当前对话
- `'fork'`：skill 作为 sub-agent 运行，拥有独立上下文和 token budget

### 12.3 插件类型（`types/plugin.ts`）

见第 6.4 节中的 `LoadedPlugin`、`BuiltinPluginDefinition`、`PluginError` 和 `PluginLoadResult`。

---

## 13. 远程会话系统（remote/）

### 13.1 系统概览

远程会话系统管理多用户和远程连接的 Claude Code 会话，在本地 CLI 进程与远程会话基础设施之间建立桥接。

**源文件：** `src/remote/`

**文件：**
- `SessionsWebSocket.ts` - 用于服务器推送会话事件的 WebSocket 客户端
- `RemoteSessionManager.ts` - 会话状态管理与重连逻辑
- `remotePermissionBridge.ts` - 远程会话的权限请求路由
- `sdkMessageAdapter.ts` - 把 SDK message 格式适配成内部 message 格式

### 13.2 组件职责

**`SessionsWebSocket`** - 维护到远程会话服务器的 WebSocket 连接。负责：
- 指数退避重连
- 会话事件路由
- 认证 token 刷新集成

**`RemoteSessionManager`** - 远程会话的顶层协调器：
- 管理会话生命周期（create、attach、detach、resume）
- 跟踪活动远程会话
- 将权限请求路由到对应会话
- 处理会话后台化 / 前台化

**`remotePermissionBridge`** - 把远程会话里的权限请求桥回本地 UI：
- 远程 worker 会话发起的权限请求会转发给交互式 CLI
- 响应会通过会话基础设施回传给 worker

**`sdkMessageAdapter`** - 在 SDK wire-format 消息与 UI 层使用的内部 `Message` 类型之间进行转换。

---

*spec 文档结束。*
