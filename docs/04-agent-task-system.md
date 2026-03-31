# 04 - Agent 与任务系统

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 拥有一套完整的多 Agent 任务系统，支持同步/异步子代理、进程内 Teammate、远程代理、后台 Shell 任务和 Dream 内存整理任务。核心由 `Task.ts` 基类 + 7 种 Task 类型 + `AgentTool` 编排器组成。

## Task 基础架构

**文件**: `Task.ts` (~126行)

### Task 状态机

```
pending → running → completed
                  → failed
                  → killed
```

### TaskStateBase

```typescript
type TaskStateBase = {
  id: string              // 带类型前缀的 ID (b=bash, a=agent, r=remote, t=teammate, d=dream)
  type: TaskType
  status: TaskStatus      // pending | running | completed | failed | killed
  description: string     // UI 显示描述
  toolUseId?: string      // 关联的 tool_use block ID
  startTime: number
  endTime?: number
  outputFile: string      // 任务输出文件路径
  outputOffset: number    // 增量读取偏移量
  notified: boolean       // 防止重复通知
}
```

### Task ID 生成

```typescript
// 前缀 + 8位 base36 随机字符 → 36^8 ≈ 2.8万亿组合
function generateTaskId(type: TaskType): string {
  const prefix = getTaskIdPrefix(type)  // a/b/r/t/d
  const bytes = randomBytes(8)
  return prefix + base36Encode(bytes)
}
```

## 7 种 Task 类型

### 1. LocalShellTask (local_bash)

**文件**: `tasks/LocalShellTask/LocalShellTask.tsx`

后台执行 shell 命令：
- 支持交互式输入检测（stall watchdog：输出停滞 45 秒 + 末行看起来是 prompt 时通知）
- 两种模式：`kind='bash'` 和 `kind='monitor'`
- 完成时发送 XML 通知（含 exit code）

### 2. LocalAgentTask (local_agent)

**文件**: `tasks/LocalAgentTask/LocalAgentTask.tsx`

在当前进程后台运行的 Claude Agent：

```typescript
type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string                    // 代理唯一标识
  prompt: string                     // 任务 prompt
  selectedAgent?: AgentDefinition    // agent 定义
  agentType: string                  // "main-session", "built-in:xxx", "user-xxx"
  model?: string                     // 模型 override
  abortController?: AbortController
  progress?: AgentProgress           // token/tool 统计
  isBackgrounded: boolean            // 前台/后台
  pendingMessages: string[]          // SendMessage 队列
  retain: boolean                    // UI 持有标志
  diskLoaded: boolean                // sidechain JSONL 已加载
  evictAfter?: number                // GC deadline
}

type AgentProgress = {
  toolUseCount: number
  tokenCount: number
  lastActivity?: ToolActivity
  recentActivities?: ToolActivity[]
  summary?: string                   // AgentSummary 生成的 1-2 句总结
}
```

### 3. RemoteAgentTask (remote_agent)

**文件**: `tasks/RemoteAgentTask/RemoteAgentTask.tsx`

在远程 CCR 环境中运行：
- 长连接 polling 模式
- 支持多种远程任务类型：`remote-agent`, `ultraplan`, `ultrareview`, `autofix-pr`, `background-pr`
- 支持自定义完成检查器和 Todo 列表同步

### 4. InProcessTeammateTask (in_process_teammate)

**文件**: `tasks/InProcessTeammateTask/types.ts`

同一 Node.js 进程中的 Agent（Agent Swarms）：

```typescript
type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity          // agentId, agentName, teamName, color
  awaitingPlanApproval: boolean
  permissionMode: PermissionMode
  messages?: Message[]                // 上限 TEAMMATE_MESSAGES_UI_CAP=50
  pendingUserMessages: string[]
  isIdle: boolean
  shutdownRequested: boolean
  onIdleCallbacks?: Array<() => void> // 零轮询等待机制
}
```

### 5. DreamTask (dream)

**文件**: `tasks/DreamTask/DreamTask.ts`

后台内存整理/consolidation 子代理：

```typescript
type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: 'starting' | 'updating'
  sessionsReviewing: number
  filesTouched: string[]              // 观察到的 Edit/Write 路径
  turns: DreamTurn[]                  // MAX_TURNS=30
  priorMtime: number                  // 锁回滚用
}
```

### 6-7. 条件编译类型

- **LocalWorkflowTask**: `feature('WORKFLOW_SCRIPTS')` 门控
- **MonitorMcpTask**: `feature('MONITOR_TOOL')` 门控

## AgentTool — 子代理编排器

**文件**: `tools/AgentTool/AgentTool.tsx` (1000+ 行)

这是 CC 最复杂的工具，负责生成和管理子代理。

### 输入 Schema

```typescript
z.object({
  description: z.string(),               // 3-5 字任务描述
  prompt: z.string(),                     // 任务 prompt
  subagent_type: z.string().optional(),   // 指定 agent 类型
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
  name: z.string().optional(),            // 可寻址 teammate 名称
  team_name: z.string().optional(),       // team 上下文
  mode: z.enum(['plan', 'default', 'browse']).optional(),
  isolation: z.enum(['worktree', 'remote']).optional(),
})
```

### 调用流程

```
AgentTool.call()
  ├─ 权限检查 (swarms 访问、teammate 嵌套禁止)
  ├─ Agent 选择 (fork vs 显式类型)
  ├─ MCP 服务器初始化 (最多等 30 秒)
  ├─ Isolation 决策
  │   ├─ worktree → createAgentWorktree(slug)
  │   ├─ remote → teleportToRemote()
  │   └─ cwd → runWithCwdOverride()
  ├─ 同步/异步决策
  │   ├─ 同步 → await runAgent() → 直接返回结果
  │   └─ 异步 → registerAsyncAgent() → LocalAgentTask
  └─ 返回 AgentToolResult
```

### 输出 Schema

```typescript
z.union([
  { status: 'completed', prompt, ...result },          // 同步完成
  { status: 'async_launched', agentId, outputFile },   // 异步启动
  { status: 'teammate_spawned', ... },                 // Teammate 生成
  { status: 'remote_launched', taskId, sessionUrl },   // 远程启动
])
```

### Fork 子代理

Fork 模式保持 byte-identical API 请求前缀以**复用 parent 的 prompt cache**，显著减少 API 成本：

```typescript
// fork child 检测防递归
// 构建 forked messages (保持cache-safe前缀)
// 设置独立 token 预算
```

### Worktree 隔离

```typescript
if (isolation === 'worktree') {
  worktreeInfo = await createAgentWorktree(slug)
  // Agent 在隔离 git 分支中运行
  // 完成后检查变更：有则保留供 review，无则清理
}
```

## SendMessage — 团队通信

**文件**: `tools/SendMessageTool/SendMessageTool.ts` (~913行)

### 消息类型

```typescript
// 结构化消息
| { type: 'shutdown_request' }
| { type: 'shutdown_response', approved: boolean }
| { type: 'plan_approval_response', approved: boolean }

// 纯文本消息
| { to: 'teammate-name', message: string }    // 单播
| { to: '*', message: string }                 // 广播
| { to: 'bridge:session-id', message: string } // 跨会话
```

### 消息路由

```
SendMessage(to, message)
  ├─ 查询 agentNameRegistry
  ├─ 查找 task = appState.tasks[agentId]
  ├─ 本地 in-process agent:
  │   ├─ running → queuePendingMessage (tool round 边界 drain)
  │   └─ stopped → resumeAgentBackground (自动恢复)
  ├─ Ambient team member → writeToMailbox (磁盘 inbox)
  └─ 跨会话 → 路由到 peer session
```

## Team 系统 — Agent Swarms

### TeamCreateTool

```typescript
// 创建 team → team file + task list + AppState.teamContext
const teamFile: TeamFile = {
  name: string,
  leadAgentId: string,
  leadSessionId: SessionId,
  members: TeamMember[]     // [{agentId, name, agentType, model, cwd, subscriptions}]
}
```

Team 文件存储在 `~/.claude/swarms/teamName.json`，支持进程崩溃恢复。

### 多后端支持

- **in-process**: AsyncLocalStorage 隔离（同一进程）
- **tmux**: 独立 tmux window（不同进程）

### 安全约束

- **Flat roster**: Teammate 不能生成 Teammate（禁止树状团队）
- **Permission mode**: plan/default/browse 三层权限
- **Plan approval**: 仅 lead 可 approve/reject

## Agent Summary 服务

**文件**: `services/AgentSummary/agentSummary.ts`

后台定时（每 30 秒）为运行中的 agent 生成 1-2 句进度总结：

```typescript
// Fork 子进程，共享 prompt cache
const result = await runForkedAgent({
  promptMessages: [{ content: "Describe your most recent action in 3-5 words..." }],
  cacheSafeParams: { ...baseParams, forkContextMessages: cleanMessages },
  canUseTool: () => ({ behavior: 'deny' }),  // 禁用工具保持请求前缀
})
```

## Task 管理工具

| 工具 | 用途 |
|------|------|
| `TaskCreate` | 创建待办事项任务 |
| `TaskGet` | 获取单个 task 详情 |
| `TaskList` | 列出所有 task（按 status） |
| `TaskOutput` | 读取 task 输出（按 offset tail） |
| `TaskStop` | 停止运行中的 task（多态 kill） |
| `TaskUpdate` | 更新 task 状态/描述 |

### Task Kill 流程

```
TaskStopTool.call(task_id)
  ├─ getTaskByType(task.type) → 多态 Task 实现
  ├─ Task.kill():
  │   ├─ LocalShellTask → 杀死子进程
  │   ├─ LocalAgentTask → abort controller + cleanup
  │   ├─ DreamTask → rollback mtime
  │   └─ RemoteAgentTask → archive session
  └─ updateTaskState → { status: 'killed' }
```

## Buddy 系统

**文件**: `buddy/types.ts`

装饰性伴侣系统（非核心功能），基于 userId hash 确定性生成：

```typescript
type Companion = {
  rarity: 'common' | 'uncommon' | 'rare' | 'epic' | 'legendary'  // 60/25/10/4/1%
  species: 'duck' | 'goose' | 'blob' | 'cat' | ...  // 18 种
  stats: Record<'DEBUGGING'|'PATIENCE'|'CHAOS'|'WISDOM'|'SNARK', number>
  name: string           // 模型生成
  personality: string    // 模型生成
}
```

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `Task.ts` | Task 基类、状态机、ID 生成 |
| `tasks.ts` | Task 注册表 |
| `tasks/LocalAgentTask/` | 本地 Agent 任务 |
| `tasks/RemoteAgentTask/` | 远程 Agent 任务 |
| `tasks/InProcessTeammateTask/` | 进程内 Teammate |
| `tasks/DreamTask/` | 后台内存整理 |
| `tasks/LocalShellTask/` | 后台 Shell 命令 |
| `tools/AgentTool/` | 子代理编排器 |
| `tools/SendMessageTool/` | 团队通信 |
| `tools/TeamCreateTool/` | Team 创建 |
| `services/AgentSummary/` | 进度总结服务 |
| `buddy/types.ts` | 伴侣系统 |
