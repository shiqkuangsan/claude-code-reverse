# Deep Dive 04 - Agent Teams / Swarms

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Agent Teams（Swarms）使 Claude Code 能够协调多个 AI agent 并行工作。支持进程内（AsyncLocalStorage 隔离）和 tmux/iTerm2（独立进程）两种后端。核心由 TeamFile、Mailbox、Permission Sync 三大组件驱动。

## Team 创建

### TeamCreateTool 流程

```
TeamCreateTool.call()
  ├─ 特性门控: isAgentSwarmsEnabled()
  ├─ 验证: 领导者未在其他团队中
  ├─ 生成唯一团队名称
  ├─ 创建 TeamFile → ~/.claude/teams/{team}/config.json
  ├─ 初始化任务目录 → ~/.claude/tasks/{taskListId}/
  ├─ 更新 AppState.teamContext
  └─ 注册清理处理器
```

### TeamFile 格式

```typescript
type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string              // "team-lead@my-team"
  leadSessionId?: string
  hiddenPaneIds?: string[]
  teamAllowedPaths?: TeamAllowedPath[]
  members: Array<{
    agentId: string                // "researcher@my-team"
    name: string
    agentType?: string
    model?: string
    color?: string
    planModeRequired?: boolean
    joinedAt: number
    tmuxPaneId: string             // 进程内为空
    cwd: string
    worktreePath?: string
    sessionId?: string
    subscriptions: string[]
    backendType?: 'tmux' | 'iterm2' | 'in-process'
    isActive?: boolean
    mode?: PermissionMode
  }>
}
```

## 团队角色

### Lead vs Teammate

| | Lead | Teammate |
|--|------|----------|
| 常量 | `TEAM_LEAD_NAME = 'team-lead'` | 自定义名称 |
| ID | `team-lead@{team}` | `{name}@{team}` |
| `isTeammate()` | false | true |
| `isTeamLead()` | true | false |
| 权限 | 可 approve/reject plan | 需要 approval |

### Flat Roster 约束

Teammate **不能**生成其他 Teammate（禁止树状团队结构）。

## 通信协议 — Mailbox

### Inbox 位置

```
~/.claude/teams/{team}/inboxes/{agent-name}.json
```

### 消息格式

```typescript
type TeammateMessage = {
  from: string
  text: string            // 纯文本或 JSON
  timestamp: string       // ISO 8601
  read: boolean
  color?: string
  summary?: string        // 5-10 字预览
}
```

### 并发控制

使用 `proper-lockfile` 库，重试 10 次（5-100ms 退避）。

### 路由

```
SendMessage(to, message)
  ├─ 单播: writeToMailbox(name, content, team)
  ├─ 广播 ("*"): 写入所有队友 inbox
  ├─ UDS: 本地 peer socket
  └─ Bridge: 远程会话 peer
```

## 关闭协议

### 队友发起

```
队友: shutdown_request → team-lead inbox
领导: 轮询发现 → UI 审批/拒绝
领导: shutdown_response → 队友 inbox
队友: approved → 优雅退出 / rejected → 继续
```

### 领导发起

```
领导: shutdown request → 队友 inbox
     + requestTeammateShutdown(taskId)
进程内: abort controller 信号
tmux: 通过 mailbox + 进程自行退出
```

## Plan 模式审批

```
队友: 进入 plan 模式 (awaitingPlanApproval: true)
队友: 写入 permission request → permissions/pending/{id}.json
领导: 轮询发现 → UI 显示 → approve/reject
领导: plan_approval_response → 队友 inbox
队友: approved → 切换到 default 模式执行
```

### Permission 目录结构

```
~/.claude/teams/{team}/permissions/
├── pending/{requestId}.json
└── resolved/{requestId}.json
```

## 进程内执行 (In-Process)

### AsyncLocalStorage 隔离

```typescript
type TeammateContext = {
  agentId: string
  agentName: string
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string
  isInProcess: true
  abortController: AbortController
}

runWithTeammateContext(context, async () => {
  // 在此作用域内 getTeammateContext() 返回 context
  // 其他作用域看不到
})
```

### 执行循环

```
while (!aborted) {
  轮询 mailbox → 处理消息 (shutdown/permission/normal)
  runAgent() → 执行工具和 API 调用
  更新 task state
  检查空闲 → setMemberActive(false)
}
```

### 空闲回调（零轮询）

```typescript
waitForTeammatesToBecomeIdle()
  → 注册 onIdleCallbacks
  → isIdle 变为 true 时触发
  → 返回 Promise<void>
```

## Tmux/iTerm2 执行

### PaneBackend 接口

```typescript
interface PaneBackend {
  type: 'tmux' | 'iterm2' | 'in-process'
  createTeammatePaneInSwarmView(name, color): Promise<CreatePaneResult>
  sendCommandToPane(paneId, command): Promise<void>
  killPane(paneId): Promise<boolean>
  hidePane(paneId): Promise<boolean>
  showPane(paneId): Promise<boolean>
}
```

### Tmux 隔离

```
Swarm socket: claude-swarm-${process.pid}  (隔离于用户 tmux)
Session: claude-swarm
Hidden: claude-hidden
```

## 团队内存同步

### 启用条件

```typescript
isTeamMemoryEnabled() = isAutoMemoryEnabled() && getFeatureValue('tengu_herring_clock')
```

### 目录

```
~/.claude/projects/{slug}/memory/team/
├── MEMORY.md
└── *.md
```

### 同步 API

```
GET  /api/claude_code/team_memory?repo={owner/repo}          拉取
GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes  元数据
PUT  /api/claude_code/team_memory?repo={owner/repo}          推送 (delta)
```

语义：Server wins per-key（拉取覆盖本地），推送仅上传哈希不同的键。

### 安全

- `sanitizePathKey()`: 拒绝 null 字节、URL 编码遍历
- `validateTeamMemWritePath()`: 符号链接逃逸检查
- Secret scanner: 推送前扫描敏感数据

## 团队清理

```
TeamDeleteTool.call()
  ├─ 验证无活跃成员
  ├─ cleanupTeamDirectories()
  │   ├─ 销毁所有 worktree
  │   ├─ rm ~/.claude/teams/{team}/ (递归)
  │   └─ rm ~/.claude/tasks/{team}/ (递归)
  ├─ unregisterTeamForSessionCleanup()
  ├─ clearTeammateColors()
  └─ 清除 AppState.teamContext
```

SIGINT/SIGTERM 时自动 `cleanupSessionTeams()` → killOrphanedTeammatePanes + cleanupTeamDirectories。

## 功能门控

```
isAgentSwarmsEnabled():
  ANT builds → 总是启用
  External → CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true + GrowthBook 'tengu_amber_flint'
```

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `utils/swarm/teamHelpers.ts` | TeamFile 操作、清理 |
| `utils/swarm/constants.ts` | 常量定义 |
| `utils/teammate.ts` | 身份检测、Context |
| `utils/teammateContext.ts` | AsyncLocalStorage |
| `utils/teammateMailbox.ts` | Mailbox 读写 |
| `utils/swarm/permissionSync.ts` | 权限请求/响应 |
| `utils/swarm/spawnInProcess.ts` | 进程内生成 |
| `utils/swarm/inProcessRunner.ts` | 执行循环 |
| `utils/swarm/backends/` | Tmux/iTerm2/InProcess |
| `tools/TeamCreateTool/` | 团队创建 |
| `tools/TeamDeleteTool/` | 团队删除 |
| `tools/SendMessageTool/` | 通信路由 |
| `services/teamMemorySync/` | 内存同步 |
