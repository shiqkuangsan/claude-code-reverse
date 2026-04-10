# Deep Dive 05 - Memory 系统

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 的内存系统由**五层机制**组成：MEMORY.md 索引、自动内存提取、Session Memory、Auto Dream 整合和团队内存同步。所有机制协同工作，让 Claude 在跨会话间保持上下文连贯。

## 目录结构

```
~/.claude/projects/<project-slug>/
├── memory/                         # 自动内存根目录
│   ├── MEMORY.md                   # 入口索引 (200行/25KB 限制)
│   ├── *.md                        # 个别内存文件
│   ├── team/                       # 团队共享内存
│   │   ├── MEMORY.md
│   │   └── *.md
│   ├── auto/                       # 后台自动提取
│   │   └── *.md
│   └── logs/                       # KAIROS 日志模式
│       └── YYYY/MM/YYYY-MM-DD.md
└── session_memory.md               # Session Memory
```

## 内存类型

| 类型 | 作用域 | 何时保存 | 用途 |
|------|--------|---------|------|
| **user** | 私密 | 用户角色/偏好 | 个性化交互 |
| **feedback** | 私密/团队 | 用户修正或确认 | 指导行为 |
| **project** | 通常团队 | 项目状态/目标 | 理解背景 |
| **reference** | 通常团队 | 外部系统指针 | 记住信息位置 |

### 不能保存什么

代码模式/架构/文件路径（可从代码推导）、Git 历史（git log 是权威源）、调试方案（在代码中）、CLAUDE.md 已有内容、临时任务细节。

## MEMORY.md 入口点

### 加载与截断

```typescript
MAX_ENTRYPOINT_LINES = 200    // 行数优先
MAX_ENTRYPOINT_BYTES = 25_000 // 字节数后续
```

截断策略：先按行 → 再按字节（在最后完整行处截断）→ 追加警告消息。

### 内存文件格式

```markdown
---
name: 记忆标题
description: 一行描述（用于相关性决策）
type: user|feedback|project|reference
---

记忆内容

feedback/project 推荐结构:
- 规则/事实（首行）
- **Why:** 原因
- **How to apply:** 应用场景
```

### 索引格式

```markdown
- [标题](file.md) — 单行钩子（约150字符以下）
```

## 自动内存启用

```
优先级（第一个定义的获胜）：
1. CLAUDE_CODE_DISABLE_AUTO_MEMORY 环境变量
2. --bare / CLAUDE_CODE_SIMPLE (禁用)
3. CCR 模式无持久存储
4. settings.json autoMemoryEnabled
5. 默认：启用
```

## 内存扫描与相关性

### scanMemoryFiles()

```typescript
// 递归读目录，过滤 .md（排除 MEMORY.md）
// 读前 30 行获取 frontmatter
// 按 mtime 排序（最新优先）
// 限制 MAX_MEMORY_FILES=200
```

### findRelevantMemories()

```typescript
// 1. 扫描内存文件（排除已显示的）
// 2. 格式化清单: [type] filename (timestamp): description
// 3. 用 Sonnet 侧查询选择最多 5 个最相关文件
// 4. 返回文件路径 + mtime
```

## Session Memory 服务

**文件**: `services/SessionMemory/sessionMemory.ts`（~500 行）

后台自动维护当前对话的注释，使用 forked subagent 不中断主对话。

### 三重门控触发器（`shouldExtractMemory()`）

`shouldExtractMemory()` 是判断是否触发提炼的核心逻辑门，必须**同时满足**三个条件，以防消耗 Token 或打断节奏：

| 门控                          | 含义                                                                |
| ----------------------------- | ----------------------------------------------------------------- |
| **容量门** `hasMetUpdateThreshold` | 距上次总结后新增 Token 数必须达到配置阈值                            |
| **行为门** `hasMetToolCallThreshold` | 距上次记录系统须调用了足够多的新工具（发生足够多的实质性动作）          |
| **时机门** `!hasToolCallsInLastTurn` | **绝不会在模型等待工具回调时截断**——必须等到模型「思考完一段落」 |

### 配置源

GrowthBook 动态配置 (`tengu_sm_config`) > DEFAULT_SESSION_MEMORY_CONFIG

### Forked Agent 隔离执行

```typescript
// 1. 创建全新隔离上下文，防污染当前对话状态
const subagentContext = createSubagentContext(toolUseContext)

// 2. 派生子代理，享受父线索的 Prompt Cache 命中
runForkedAgent({
  ...subagentContext,
  forkLabel: 'session_memory',
  canUseTool: createMemoryFileCanUseTool(memoryPath),  // 极度严格权限
})
```

### 文件位置

`~/.claude/projects/<slug>/session_memory.md` — **仅允许 Edit 该单一文件**。

### 手动唤起逃生口

`manuallyExtractSessionMemory` 暴露给斜杠命令层（CLI 侧 `/summary`）。当处于紧急状态或任务交接时，用户主动键入指令将**绕过所有 Token 检测闸**，强制拉取所有上下文总结入记忆块。

### 切面式 Hook 解耦

`sessionMemory` 并没有粗暴写死在 `query.ts` 的 `while(true)` 循环内，而是利用：

```typescript
registerPostSamplingHook(extractSessionMemory)
```

挂载在 **Post Sampling Hook** 生命周期的结束点。这是极其优雅的切面编程设计，保持核心状态机的纯净。详见 [`dd-01-hooks.md`](dd-01-hooks.md) 双管线对比章节。

## Extract Memories 服务

**文件**: `services/extractMemories/extractMemories.ts`（~600 行）

在每个完整查询循环末尾运行（模型响应无工具调用时），通过 forked agent 自动提取持久内存。**与 sessionMemory 走完全不同的 Hook 管线**——它由 `stopHooks.ts` 的 `handleStopHooks` 调度（详见 [`dd-01-hooks.md`](dd-01-hooks.md)）。

### 门控（多层防穿透）

```
1. 功能门: tengu_passport_quail (GrowthBook)
2. 自动内存启用
3. 非远程模式
4. 仅主 agent
5. 节流器: turnsSinceLastExtraction (受 tengu_bramble_lintel 配置控制)
6. 互斥检查: hasMemoryWritesSince（若主 agent 已直接写过 memory 文件则跳过）
```

只有通过**所有门控**后才会真正启动子代理。

### 闭包沙盒状态机（`initExtractMemories`）

作者没有把状态变量设为全局共享，而是用一个 `initExtractMemories()` 工厂闭包持有：

```typescript
function initExtractMemories() {
  let inProgress = false       // 当前有没有上一次记忆正在生成？
  let pendingContext = null    // 用于 trailing run 的合并状态

  return async function extract(ctx) {
    if (inProgress) {
      pendingContext = ctx     // Stash 逻辑：合并到待处理
      return
    }
    inProgress = true
    try {
      await runExtraction(ctx)
    } finally {
      inProgress = false
      if (pendingContext) {
        const next = pendingContext
        pendingContext = null
        extract(next)          // Trailing run：立刻衔接尾随提取
      }
    }
  }
}
```

**Trailing run 设计**：如果当前模型思考很快，记忆生成还在跑，用户又触发了新一轮——系统会把当前记录更新至 `pendingContext`，等前一条记忆刚写完（`finally` 块），立马衔接开启尾随提取，**永远不死锁且不漏看任何新句子**。

### 双重大脑互斥锁（`hasMemoryWritesSince`）

代码考虑到了极端场景：用户在主对话中可能直接说「你把这个给我记在配置里」。如果主 agent 已经主动用 `FileWriteTool` 往 memory 文件夹写过规则，互斥锁会立刻 `return`，**防止主系统和后台代理发生人格分裂和重复抢占写入**。

### Prompt Cache 寄生体系

子代理虽然是后台新开的 API 请求，但因为它是从**主线循环上下文中完美克隆出的参数 Fork**，它能极大地通过 Anthropic API 的 Prompt Cache 特性，无需重新计费前面的万字长文，只须付一点点新运算 Token——**在几乎零成本的基础上获得智能提取服务**。

日志中能看到这段输出：`cache: read=... create=... input=... (X% hit)`，这是 Cache 寄生的命中证据。

### 工具权限：`createAutoMemCanUseTool`

```
允许: Read/Grep/Glob (无限制读取整个工程)
      Bash (仅 ls/find/cat/stat 等只读命令)
      Edit/Write (仅限 ~/.claude/projects/.../memory/ 目录)
拒绝: 所有其他工具（包括动业务源码的 FileEdit/FileWrite/写 Bash）
```

### 60 秒宽限期逃生窗口（`drainPendingExtraction`）

```typescript
drainPendingExtraction({ timeoutMs: 60_000 })
```

CLI 被 `Ctrl+C` 中断退出前，挂起一个带超时回收器的钩子，向系统申请 **60 秒隐形宽限期**，保证即使用户强退程序，刚刚学会的知识依然会被拼死写入硬盘。

### 用户前台反馈（`createMemorySavedMessage`）

当后台线程写出优质记忆片段时，会偷偷向主干道塞回一条幽灵消息：`appendSystemMessage?.(msg)`。终端 UI 在聊天间隙会冷不丁升起一个小标签：`[X memories saved]`。

## sessionMemory vs extractMemories：权限对比

两套后台子代理虽然都在「模型回答后」触发，但权限范围**天差地别**——这是 Claude Code 防御性设计的精髓体现：

| 维度              | sessionMemory                               | extractMemories                                  |
| ----------------- | ------------------------------------------- | ------------------------------------------------ |
| **触发钩子**      | `registerPostSamplingHook`（采样后切面）       | `handleStopHooks` (`stopHooks.ts`)                |
| **触发时机**      | 每次模型采样后                                | 主对话稳定终止时                                   |
| **权限工厂**      | `createMemoryFileCanUseTool`                | `createAutoMemCanUseTool`                        |
| **可读工具**      | （无）                                        | Read / Grep / Glob / 只读 Bash（`ls`/`find`/`cat`） |
| **可写工具**      | **仅 FILE_EDIT_TOOL，仅一个目标文件**          | Edit / Write（仅在 memory 目录内）                  |
| **写入目标**      | `session_memory.md` 单文件                   | `~/.claude/projects/<slug>/memory/` 整个目录       |
| **设计意图**      | 极度严格——只能更新一份长程摘要                 | 适度宽松——允许复盘上下文写更细的记忆词条             |
| **强退保护**      | 无                                          | `drainPendingExtraction` 60 秒宽限期                |

## Auto Dream / DreamTask

**文件**: `services/autoDream/`, `tasks/DreamTask/`

后台内存整合 — 将日志/转录蒸馏为主题文件 + MEMORY.md 索引。

### 三级门

```
时间门: hoursSince(lastConsolidatedAt) ≥ minHours (默认 24)
扫描节流: SESSION_SCAN_INTERVAL_MS = 10 分钟
会话门: 新会话数 ≥ minSessions (默认 5)
锁门: tryAcquireConsolidationLock() 防重叠
```

### DreamTask 状态

```typescript
type DreamTaskState = {
  phase: 'starting' | 'updating'    // Edit/Write 时翻转到 updating
  sessionsReviewing: number
  filesTouched: string[]             // 观察到的文件路径
  turns: DreamTurn[]                 // MAX_TURNS=30
  priorMtime: number                 // 锁回滚
}
```

### 工具限制

Bash 仅读（ls, find, grep, cat, stat），Edit/Write 仅在内存目录。

## 内存注入到上下文

### 加载分发

```
KAIROS + autoMem     → buildAssistantDailyLogPrompt()
TEAMMEM 启用         → buildCombinedMemoryPrompt() (私密+团队)
autoMem 启用         → buildMemoryLines()
其他                 → null
```

### 时新性提醒

```typescript
memoryFreshnessNote(mtimeMs)
// > 1 天: "这个记忆是 X 天前。记忆是时间点观察，不是活态"
// 建议：推荐前验证，冲突时更新或删除
```

## 团队内存

### 启用条件

```typescript
isTeamMemoryEnabled() = isAutoMemoryEnabled() && getFeatureValue('tengu_herring_clock')
```

### 同步 API

| 操作 | 端点 | 语义 |
|------|------|------|
| Pull | `GET /api/claude_code/team_memory` | Server wins per-key |
| Push | `PUT /api/claude_code/team_memory` | Delta (仅哈希不同) |
| Probe | `GET ...&view=hashes` | 元数据探针 |

### 限制

```
单个文件: 250KB
PUT body: 200KB
哈希: sha256:<hex>
```

### 安全

- 路径遍历防护 + 符号链接解析
- 推送前 secret scanner 扫描敏感数据

## 压缩与内存交互

### 自动压缩阈值

```typescript
autoCompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS(13000)
```

### 压缩后清理

清除：microcompact 状态、上下文折叠缓存、getUserContext 缓存、分类器审批、Bash 权限缓存。

### 压缩后恢复

```
POST_COMPACT_MAX_FILES_TO_RESTORE = 5
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5K
POST_COMPACT_TOTAL_TOKEN_BUDGET = 50K
```

## 整体架构图

```
主循环
  ├─ loadMemoryPrompt() → MEMORY.md + 指导 → 系统提示
  ├─ [模型生成响应]
  ├─ Post-sampling hooks:
  │   ├─ extractMemories (前台，如果主 agent 写内存则跳过)
  │   ├─ sessionMemory (后台，更新 session_memory.md)
  │   └─ autoDream (后台，满足三级门时)
  ├─ autoCompactIfNeeded()
  └─ drainPendingExtraction()

Forked Agents (隔离，缓存共享):
  ├─ extractMemories (从对话提取)
  ├─ sessionMemory (更新注释)
  ├─ autoDream (蒸馏整合)
  └─ compact (总结对话)
```

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `memdir/paths.ts` | 路径解析、安全验证 |
| `memdir/memoryTypes.ts` | 类型定义、前言格式 |
| `memdir/memdir.ts` | MEMORY.md 加载、截断、提示构建 |
| `memdir/memoryScan.ts` | 文件扫描、frontmatter 解析 |
| `memdir/findRelevantMemories.ts` | 相关性选择（Sonnet 侧查询） |
| `memdir/memoryAge.ts` | 时新性警告 |
| `memdir/teamMemPaths.ts` | 团队内存路径、安全验证 |
| `services/SessionMemory/` | Session Memory 后台服务 |
| `services/extractMemories/` | 自动内存提取 |
| `services/autoDream/` | Auto Dream 整合 |
| `services/teamMemorySync/` | 团队内存同步 API |
| `tasks/DreamTask/` | Dream 任务状态 |
| `services/compact/` | 压缩服务 |
| `commands/memory/` | /memory 命令 |
