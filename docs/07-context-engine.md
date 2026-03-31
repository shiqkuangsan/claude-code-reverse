# 07 - 上下文引擎

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

上下文引擎负责在每轮对话开始时收集系统信息、加载 CLAUDE.md 指令、组装 system prompt。设计核心是 **memoization**——所有上下文在对话期间缓存，避免重复 I/O。

## 上下文收集

**文件**: `context.ts` (~190行)

### 系统上下文

```typescript
export const getSystemContext = memoize(async (): Promise<{ [k: string]: string }>) => {
  // Git 状态信息（仅非 CLAUDE_CODE_REMOTE 且已启用 Git 时）
  gitStatus: {
    currentBranch,
    mainBranch,         // 用于 PR
    gitUser,
    'git status --short',  // 截断到 2000 字符
    'git log --oneline -5'
  }
})
```

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
- 缓存到 bootstrap state 供 yoloClassifier 使用

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
