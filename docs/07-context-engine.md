# 07 - 上下文引擎

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

上下文引擎负责在每轮对话开始时收集系统信息、加载 CLAUDE.md 指令、组装 system prompt。设计核心是 **memoization**——所有上下文在对话期间缓存，避免重复 I/O。

## 上下文收集

**文件**: `context.ts` (~190行)

> `context.ts` 是让 Agent 知道「现在是几点？这是谁的项目？有没有我必须背下来的本地规矩？」的**潜意识注射站**。如果说 `query.ts` 是神经循环、`QueryEngine.ts` 是底盘，那么 `context.ts` 就是把环境包裹成「世界观和自我意识」喂给大模型的关键数据源。

### 系统上下文

```typescript
export const getSystemContext = memoize(async (): Promise<{ [k: string]: string }>) => {
  // Git 状态信息（仅非 CLAUDE_CODE_REMOTE 且已启用 Git 时）
  gitStatus: {
    currentBranch,
    mainBranch,         // 用于 PR
    gitUser,
    'git status --short',  // 截断到 MAX_STATUS_CHARS=2000 字符
    'git log --oneline -5'
  }
})
```

### Git 环境态感知（`getGitStatus`）

这是一个自动嗅探当前工作区源码版本管理状态的工作流：

- **并发探查**：一次并发执行**四个命令**——
  1. 获取当前分支
  2. 获取主分支（用于 PR）
  3. `git status --short`
  4. `git log --oneline -5`（最近 5 条记录）
  5. （并发但独立）`git config user.name`

- **抗爆防刷屏机制**：返回巨大的 Git 变更树会严重污染上下文。系统设定了 `MAX_STATUS_CHARS = 2000`。一旦超出，强制截断并附加固定提示：

  > `...(truncated because it exceeds 2k characters. If you need more information, run 'git status' using BashTool)`

  这是一种巧妙的诱导：**强迫 AI 在真的需要读全貌时，自己长出对应的「手」去调 BashTool**，而不是无脑塞入 system prompt。

- **固化提示词**：开场白强制加上 `Note that this status is a snapshot in time...`（这是初始快照，聊天期间不会更新）

### 缓存击穿调试机制（`cacheBreaker`）

`getSystemContext` 还包含一个内部调试机制：对于 Anthropic 内部员工/调试场景（`ant-only` 且配置了 `BREAK_CACHE_COMMAND` 环境变量），系统会植入一段毫无规律的魔法字符串，使得 API 服务端的 Prompt Cache 失效，强迫大模型重新完整过一遍上下文。这是工程团队的内部排查工具，普通用户不会触发。

### 用户上下文

```typescript
export const getUserContext = memoize(async (): Promise<{ [k: string]: string }>) => {
  claudeMd: getClaudeMds(),      // 所有 CLAUDE.md 文件内容
  currentDate: 'Today\'s date is YYYY-MM-DD.'
})
```

### CLAUDE.md 加载机制

- 目录树遍历发现（从 cwd 向上到根目录）
- `--bare` 模式下跳过（除非有 `--add-dir`）
- 可通过 `CLAUDE_CODE_DISABLE_CLAUDE_MDS` 环境变量禁用
- 注入的内存文件被 `filterInjectedMemoryFiles()` 过滤
- **缓存隔离反环策略**：抓取后的 Memory 字符串被压进内存全局缓存（`setCachedClaudeMdContent`），避免鉴权系统形成循环依赖（`filesystem` → `permissions` → `yoloClassifier` → `claudemd`）
- 缓存到 bootstrap state 供 yoloClassifier 使用

### memoize 防刷扫描

`getSystemContext` 与 `getUserContext` 都用了 `lodash.memoize` 包装。**所有的环境包裹数据都会被缓存**，确保单次会话循环中不会被反复触发扫描。这意味着：

- Git 状态每次会话只采集一次（除非显式 `flushContextCache`）
- `CLAUDE.md` 内容只读一次
- 时间戳锚点也只生成一次

### 「冷启动注射」设计哲学

通过这个精简的文件，可以学到一种很高级的 Agent 工程技巧：**Cold-start Injection**（冷启动注射）。

1. **不需要把所有工具都交给模型，而是把环境「喂」给模型**：系统并没有专门做一个 `GetGitStatusTool` 让 AI 第一步总是去调用，而是选择在一开始悄悄运行后垫在系统提示词下面，**极大节省开局第一个 RoundTrip 的耗时**
2. **永远不要信任外界的信息长度**：即便只是短短一个 `git status` 也可能因为前端装了一个巨大的 vendor 依赖且忘了配 `.gitignore` 而导致十几万字。`MAX_STATUS_CHARS` 这种防御性截断 + 诱导 AI 主动去挖坑的设计，是底层 Agent 不崩溃的安全底线

## System Prompt 组装

```
systemPrompt (来自 AI backend 默认提示)
    ↓
+ systemContext (Git 状态、cache breaker)
    ↓
+ userContext (CLAUDE.md、当前日期)
    ↓
+ prependUserContext() → 前置用户/项目信息
    ↓
完整 system prompt → API 请求
```

## 上下文压缩策略

CC 采用多层策略应对上下文窗口限制，在 `query.ts` 的每轮迭代中按序执行：

### 1. 工具结果预算 (applyToolResultBudget)

每个工具定义 `maxResultSizeChars` 阈值。超过时：
- 完整输出保存到 `~/.cache/Claude Code/tool-results/`
- 向模型返回预览 + 文件路径

### 2. Snip 紧凑化

Feature gate 控制的片段化策略。

### 3. 微紧凑化 (microcompact)

**文件**: `services/compact/microcompact.ts`

API 级别的上下文管理，选择性消息缩减。

### 4. 上下文折叠 (contextCollapse)

**文件**: `services/contextCollapse/`

为长序列消息创建折叠视图，避免完整压缩。Feature gate: `CONTEXT_COLLAPSE`。

### 5. 自动紧凑化 (autoCompact)

**文件**: `services/compact/autoCompact.ts`

当消息历史超过 token 阈值时自动触发：
- 消息分组：`groupMessagesByApiRound()` 按 API 往返分组
- 图像剥离：`stripImagesFromMessages()` 减少 token
- 系统提示保留：始终保留系统消息

### 紧凑化后恢复

**文件**: `services/compact/postCompactCleanup.ts`

```typescript
// 恢复最重要的上下文
POST_COMPACT_MAX_FILES_TO_RESTORE = 5        // 最多恢复 5 个文件
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5000     // 每项技能最多 5K token
POST_COMPACT_TOTAL_TOKEN_BUDGET = 50000      // 总预算 50K token
```

## Token 预算追踪

**文件**: `query/tokenBudget.ts`

当 `feature('TOKEN_BUDGET')` 启用时：
- 跟踪每轮 token 使用
- 检查是否超过 500K 自动继续阈值
- 触发后续自动继续迭代

## 内存系统

### 内存目录

**文件**: `memdir/`

```
~/.claude/projects/<project-slug>/memory/
├── MEMORY.md          # 入口点 (200行 / 25KB 上限)
├── auto/              # 自动内存 (GrowthBook 门控)
└── team/              # 团队内存 (TEAMMEM 特性)
```

### 内存类型

四分类法：`user` / `feedback` / `project` / `reference`

### 内存截断

- 行数限制优先 (MAX_ENTRYPOINT_LINES: 200)
- 字节限制后续 (MAX_ENTRYPOINT_BYTES: 25KB)
- 在最后一个完整行处截断

### 会话内存

**文件**: `services/SessionMemory/`

后台自动化服务，通过分叉子代理定期提取会话关键信息：
- 触发条件：上下文窗口增长超过配置阈值
- 更新频率：token 增长 + 工具调用次数
- 远程配置门控：`tengu_session_memory`, `tengu_sm_config`

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `context.ts` | 系统/用户上下文收集（memoized） |
| `query.ts` | 查询主循环、上下文压缩编排 |
| `services/compact/autoCompact.ts` | 自动紧凑化 |
| `services/compact/microcompact.ts` | 微紧凑化 |
| `services/compact/postCompactCleanup.ts` | 紧凑化后恢复 |
| `services/contextCollapse/` | 上下文折叠 |
| `memdir/` | 内存目录管理 |
| `services/SessionMemory/` | 会话内存服务 |
| `query/tokenBudget.ts` | Token 预算追踪 |
