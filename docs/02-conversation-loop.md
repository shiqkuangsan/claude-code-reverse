# 02 - 核心对话循环

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 的对话循环采用**异步生成器驱动**架构，从用户输入出发，通过 API 流式调用获取响应，解析工具调用并执行，反馈结果后继续迭代，直到模型返回 `end_turn`。

```
用户输入 → query() 异步生成器 → API 流式调用
                                    ↓
                           解析 tool_use blocks
                                    ↓
                     权限检查 → 执行工具 → 返回结果
                                    ↓
                           添加结果消息 → 继续循环
```

## 核心入口: query()

**文件**: `query.ts` (~2400行)

```typescript
export async function* query(params: QueryParams):
  AsyncGenerator<StreamEvent | RequestStartEvent | Message | ToolUseSummaryMessage, Terminal>
```

- **返回类型**: 异步生成器，逐个 yield 事件流
- **终止值**: `Terminal` 对象（包含停止原因）
- **调用方**: REPL 组件消费生成器事件来更新 UI

### 状态结构

```typescript
type State = {
  messages: Message[]                          // 完整对话历史
  toolUseContext: ToolUseContext               // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  pendingToolUseSummary: Promise<...> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined             // 上次 continue 的原因
}
```

## 单轮流程

每一轮迭代经过 5 个阶段：

### 阶段 1: 前置处理

```typescript
// query.ts:330-365
// 初始化查询链追踪
queryTracking.increment()
// 启动技能发现预抓取
prefetchSkillDiscovery()
// 通知上层新一轮开始
yield { type: 'stream_request_start' }
```

### 阶段 2: 消息准备（Token 管理）

这是 CC 应对上下文窗口限制的关键环节：

```typescript
// query.ts:365-550
applyToolResultBudget(messages)    // 工具结果预算（大输出持久化到磁盘）
snipMessages(messages)             // Snip 紧凑化
microcompact(messages)             // 微紧凑化（删除冗余空白等）
contextCollapse(messages)          // 上下文折叠（合并相似消息）
autoCompact(messages)              // 自动紧凑化（超过阈值时触发摘要）
```

### 阶段 3: API 流式调用

```typescript
// query.ts:650-900
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  tools: toolUseContext.options.tools,
  model: mainLoopModel,
  thinkingConfig,
  ...
})) {
  yield message  // 逐条 yield 给 UI 层
  // 识别 tool_use blocks
  if (message.type === 'tool_use') {
    toolUseBlocks.push(message)
  }
}
```

### 阶段 4: 工具执行

```typescript
// query.ts:900-1000+
// 递归调用 runTools() 执行所有工具
const toolResults = await runTools(toolUseBlocks, toolUseContext)
// 将工具结果作为新消息添加到历史
messages.push(...toolResults)
```

### 阶段 5: 循环判定

```typescript
// query.ts:1100-1200+
if (stop_reason === 'end_turn') {
  return Terminal  // 结束循环
}
if (toolUseBlocks.length > 0) {
  continue  // 有工具调用，继续迭代
}
// 错误恢复路径：自动紧凑 / 响应式紧凑
```

## 工具执行编排

**文件**: `services/tools/toolExecution.ts` (~600行)

### 执行生命周期

```
validateInput()
    ↓
checkPermissions()
    ↓ (allow/ask/deny)
runPreToolUseHooks()
    ↓
tool.call()          ← 实际执行
    ↓
runPostToolUseHooks()
    ↓
返回 ToolResult
```

### 流式工具执行器

**文件**: `services/tools/StreamingToolExecutor.ts`

当多个工具并行时，逐个流式执行并 yield 结果：

```typescript
for await (const result of executor.execute(toolUseBlocks)) {
  yield result  // 不等所有工具完成，逐个返回
}
```

好处：更快给模型部分结果、实时进度可视化。

## 消息类型体系

```typescript
type Message =
  | UserMessage              // 用户输入
  | AssistantMessage         // AI 响应（可含 tool_use）
  | AttachmentMessage        // 附件（文件、技能）
  | ProgressMessage          // 工具进度
  | SystemMessage            // 系统提示
  | TombstoneMessage         // 删除标记
  | ToolUseSummaryMessage    // 工具使用摘要
```

### AssistantMessage 结构

```typescript
type AssistantMessage = {
  type: 'assistant'
  uuid: string
  message: {
    role: 'assistant'
    content: ContentBlockParam[]  // 文本块、tool_use 块、thinking 块
  }
}
```

### ToolUseBlock

```typescript
type ToolUseBlock = {
  type: 'tool_use'
  id: string                          // 工具使用 ID
  name: string                        // 工具名称 (如 "Bash", "FileRead")
  input: Record<string, unknown>      // 输入参数
}
```

## 工具结果持久化

**文件**: `utils/toolResultStorage.ts`

每个工具定义 `maxResultSizeChars` 阈值（如 BashTool = 30,000 字符）。超过阈值时：

1. 完整输出保存到 `~/.cache/Claude Code/tool-results/`
2. 向模型返回预览 + 文件路径
3. 模型可通过 FileRead 获取完整内容

### 内容替换预算

```typescript
type ContentReplacementState = {
  toolUseIdToReplacement: Map<string, ContentReplacement>
  filePathToSize: Map<string, number>
}

// 跨消息聚合限制工具结果大小
applyToolResultBudget(messages, contentReplacementState, persistCallback)
```

## Coordinator 模式

**文件**: `coordinator/coordinatorMode.ts`

当 `CLAUDE_CODE_COORDINATOR_MODE` 环境变量启用时，CC 切换为协调器角色：

- **协调器职责**: 帮助用户、指导工作者、综合结果
- **工作者**: 通过 AgentTool 异步执行研究/实现/验证任务
- **并行性**: 独立任务并发执行
- **综合**: 协调器在下达后续指令前必须理解工作者发现

```typescript
export function isCoordinatorMode(): boolean
export function getCoordinatorUserContext(mcpClients, scratchpadDir): UserContext
```

## 对话历史管理

**文件**: `assistant/sessionHistory.ts`

与后端会话事件 API 交互，支持分页加载：

```typescript
export async function fetchLatestEvents(ctx, limit): Promise<HistoryPage | null>
export async function fetchOlderEvents(ctx, beforeId, limit): Promise<HistoryPage | null>

type HistoryPage = {
  events: SDKMessage[]
  firstId: string | null
  hasMore: boolean
}
```

## 关键数据流

```
1. 用户在 REPL 中输入消息
   ↓
2. REPL 组件调用 query(params)
   ↓
3. query() 准备消息（紧凑化/折叠）
   ↓
4. 调用 callModel() 流式请求 API
   ↓
5. yield 流式事件给 UI（文本、thinking、tool_use）
   ↓
6. 收集 toolUseBlocks
   ↓
7. toolExecution.runTools() 逐个执行：
   │  validateInput → checkPermissions → preHooks → call → postHooks
   ↓
8. yield 工具结果
   ↓
9. 添加结果消息到 messages[]
   ↓
10. 判断是否继续：
    • end_turn → 返回 Terminal，结束
    • 有 tool_use → continue，回到步骤 3
    • 错误 → 恢复路径（紧凑化后重试）
```

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `query.ts` | 核心对话循环（异步生成器） |
| `services/tools/toolExecution.ts` | 工具执行编排 |
| `services/tools/StreamingToolExecutor.ts` | 流式工具执行器 |
| `assistant/sessionHistory.ts` | 会话历史分页加载 |
| `coordinator/coordinatorMode.ts` | 协调器模式 |
| `interactiveHelpers.tsx` | 交互式对话框管理 |
