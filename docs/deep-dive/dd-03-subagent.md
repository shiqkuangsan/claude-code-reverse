# Deep Dive 03 - Subagent 编排机制

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

AgentTool 是 Claude Code 最复杂的工具（1000+ 行），负责子代理的生成、执行和生命周期管理。支持同步/异步执行、Fork 缓存共享、Worktree/Remote 隔离和后台进度追踪。

## AgentTool 完整流程

```
AgentTool.call()
  ├─ 权限检查 (swarms, teammate 嵌套)
  ├─ Agent 选择
  │   ├─ subagent_type 省略 + fork 启用 → Fork 路径
  │   └─ subagent_type 指定 → 从 agentDefinitions 查找
  ├─ MCP 初始化 (等待 pending 服务器，最多 30s)
  ├─ 隔离决策
  │   ├─ worktree → createAgentWorktree()
  │   ├─ remote → teleportToRemote()
  │   └─ cwd → runWithCwdOverride()
  ├─ 同步/异步决策
  │   ├─ 同步 → await runAgent()
  │   └─ 异步 → registerAsyncAgent() + 后台 runAsyncAgentLifecycle()
  └─ 返回结果
```

## Agent 选择逻辑

```typescript
effectiveType = subagent_type ?? (isForkEnabled ? undefined : 'general-purpose')

if (effectiveType === undefined) {
  // Fork 路径 - 防递归检查
  if (isInForkChild(messages)) throw Error('Fork 不可在 fork 子进程内')
  selectedAgent = FORK_AGENT
} else {
  // 普通路径
  const found = activeAgents.find(a => a.agentType === effectiveType)
  if (!found) {
    // 检查是否被权限规则拒绝
    if (allAgents.find(a => a.agentType === effectiveType))
      throw Error(`Agent denied by rule`)
    throw Error(`Agent not found`)
  }
  selectedAgent = found
}
```

### 内置 Agent 类型

| Agent | 工具 | 权限 | 用途 |
|-------|------|------|------|
| General-Purpose | `[*]` | default | 通用任务 |
| Explore | 只读 | default | 搜索和读取 |
| Plan | 只读 | default | 计划和分析 |
| Verify | 验证工具 | default | 测试验证 |
| Fork | `[*]` | bubble | 隐式 fork（缓存共享） |

## Fork Subagent — 缓存共享优化

### 核心思想

Fork 子代理继承父亲的**完整助手消息**并构造 byte-identical 请求前缀，使得 Anthropic API 的 prompt cache 可以被复用，显著减少成本。

### buildForkedMessages()

```typescript
function buildForkedMessages(directive, assistantMessage) {
  // [1] 克隆完整助手消息（包括 thinking, text, 所有 tool_use）
  const fullAssistant = { ...assistantMessage, uuid: randomUUID() }

  // [2] 为每个 tool_use 构建占位符 tool_result
  const toolResults = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: 'Fork started — processing in background'
  }))

  // [3] 追加 fork 指令
  return [fullAssistant, toolResultMessage_with_directive]
  // 只有最后文本块差异 → 最大化缓存命中
}
```

### Fork 指令模板

```
RULES:
1. 你是 fork 子进程，不是主代理。不要 spawn 子代理
2. 不交谈、不询问、不建议
3. 使用工具直接执行，沉默工作
4. 报告前提交更改，包含 commit hash
```

## 同步 vs 异步决策

```typescript
shouldRunAsync = (
  run_in_background === true ||        // 显式请求
  selectedAgent.background === true || // Agent 定义强制
  isCoordinator ||                     // Coordinator 模式
  forceAsync ||                        // Fork 实验
  isProactiveActive                    // 主动代理
) && !isBackgroundTasksDisabled
```

## runAgent() — 内部消息循环

```typescript
async function* runAgent({ agentDefinition, promptMessages, ... }) {
  // [1] 上下文准备
  const agentId = createAgentId()
  const agentTools = resolveAgentTools(agentDefinition, availableTools, isAsync)
  const agentMcpServers = await initializeAgentMcpServers(agentDefinition)

  // [2] 系统提示
  //   - Fork: 继承父亲的 renderedSystemPrompt (byte-identical)
  //   - Explore/Plan: 省略 claudeMd 和 gitStatus 节省 tokens

  // [3] Hook 注册 (作用域到 agent 生命周期)
  registerFrontmatterHooks(agentDefinition.hooks, agentId)

  // [4] 技能预加载
  for (const skillName of agentDefinition.skills) {
    loadSkillIntoMessages(skillName, initialMessages)
  }

  // [5] 主循环
  for await (const message of query({
    messages: initialMessages,
    systemPrompt: agentSystemPrompt,
    canUseTool,
    maxTurns: agentDefinition.maxTurns,
  })) {
    // 记录到 sidechain JSONL
    await recordSidechainTranscript([message], agentId)
    yield message
  }

  // [6] 清理: MCP、hooks、文件缓存、bash 任务
}
```

## 隔离模式

### Worktree 隔离

```typescript
const worktreeInfo = await createAgentWorktree(slug)
// → { worktreePath, worktreeBranch, headCommit, gitRoot }

// Agent 在隔离 git 分支运行
// 完成后：有变更 → 保留供 review；无变更 → 清理
```

Fork + Worktree 时自动注入路径翻译通知。

### Remote 隔离 (仅 ANT)

```typescript
const session = await teleportToRemote({ initialMessage: prompt })
// → 注册 RemoteAgentTask，返回 sessionUrl
```

### CWD 覆盖

KAIROS 特性，覆盖所有文件系统和 shell 操作的工作目录。

## 进度追踪

```typescript
type AgentProgress = {
  toolUseCount: number          // 累积工具使用
  tokenCount: number            // 输入 + 累积输出
  lastActivity?: ToolActivity   // 最近工具活动
  recentActivities?: ToolActivity[]  // 最近 5 个
  summary?: string              // AgentSummary 生成的 1-2 句
}
```

### AgentSummary 服务

每 30 秒 fork 会话生成进度摘要，使用相同 CacheSafeParams 共享 prompt cache。工具在 canUseTool 回调中拒绝（不从工具列表移除）以保持请求前缀。

## 消息队列

```
SendMessage → queuePendingMessage(agentId, msg)
    → 存入 LocalAgentTaskState.pendingMessages[]
    → 工具轮次边界 drainPendingMessages() 排出
    → 注入为用户消息到下一轮

如果 agent 已停止：
    → resumeAgentBackground() 自动恢复并注入消息
```

## Abort 双层控制

```
同步代理: 共享父 AbortController
异步代理: 独立 AbortController
    └─ 可选级联: createChildAbortController(parent)

In-Process Teammate:
    Layer 1: task.abortController (杀整个 agent)
    Layer 2: currentWorkAbortController (仅中止当前 turn)
```

## 通知格式

```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>{description}</summary>
  <result>{final text}</result>
  <usage><total_tokens>N</total_tokens><tool_uses>N</tool_uses></usage>
</task-notification>
```

原子 `notified` 标志防止重复通知。

## Agent 定义 Frontmatter

```yaml
---
agentType: "Explore"
tools: ["*"]
disallowedTools: ["write"]
permissionMode: "plan"
model: "sonnet"
maxTurns: 50
isolation: "worktree"
omitClaudeMd: true
background: false
skills: ["my-skill"]
mcpServers: ["github"]
hooks:
  SubagentStart: [{ type: "command", command: "..." }]
---
```

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `tools/AgentTool/AgentTool.tsx` | 主工具：验证、路由、权限 |
| `tools/AgentTool/runAgent.ts` | 内部循环：MCP 初始化、消息记录、清理 |
| `tools/AgentTool/forkSubagent.ts` | Fork 机制：消息构建、递归防护 |
| `tools/AgentTool/agentToolUtils.ts` | 工具解析、进度、异步生命周期 |
| `tools/AgentTool/loadAgentsDir.ts` | Agent 定义加载 |
| `tasks/LocalAgentTask/` | 后台任务：注册、进度、通知 |
| `services/AgentSummary/` | 后台进度总结 |
| `coordinator/coordinatorMode.ts` | Coordinator 模式 |
