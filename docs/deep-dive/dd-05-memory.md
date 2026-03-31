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

**文件**: `services/SessionMemory/`

后台自动维护当前对话的注释，使用 forked subagent 不中断主对话。

### 触发条件

```
初始化：消息 token 超过 minimumMessageTokensToInit
更新条件（满足任一）：
  - token 增长 ≥ minimumTokensBetweenUpdate AND 工具调用 ≥ toolCallsBetweenUpdates
  - token 增长 ≥ threshold AND 最后一轮无工具调用
```

### 配置源

GrowthBook 动态配置 (`tengu_sm_config`) > DEFAULT_SESSION_MEMORY_CONFIG

### 文件位置

`~/.claude/projects/<slug>/session_memory.md` — 仅允许 Edit 该文件。

## Extract Memories 服务

**文件**: `services/extractMemories/`

在每个完整查询循环末尾运行（模型响应无工具调用时），通过 forked agent 自动提取持久内存。

### 门控

```
功能门: tengu_passport_quail (GrowthBook)
+ 自动内存启用
+ 非远程模式
+ 仅主 agent
```

### 互斥逻辑

`hasMemoryWritesSince()` 检查主 agent 是否写入自动内存路径。如果是，跳过 forked 提取（避免冲突）。

### 工具权限

```
允许: Read/Grep/Glob (无限制), Bash (仅读操作)
      Edit/Write (仅限自动内存目录)
拒绝: 所有其他工具
```

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
