# 03 - 工具框架与执行管线

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 的工具系统是整个产品的核心能力层。通过统一的 `Tool` 接口、构建者模式的 `buildTool` 工厂、多层权限检查和 hooks 机制，实现了 45+ 个工具的一致管理和安全执行。

## Tool 接口

**文件**: `Tool.ts` (~793行)

### 核心字段

```typescript
type Tool<Input, Output, P extends ToolProgressData> = {
  // === 标识 ===
  name: string                           // 唯一标识符 (如 "Bash", "FileRead")
  aliases?: string[]                     // 向后兼容别名
  searchHint?: string                    // 工具搜索关键词 (3-10 字)
  isMcp?: boolean                        // 是否为 MCP 工具

  // === 执行 ===
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>

  // === 权限与验证 ===
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput(input, context): Promise<ValidationResult>

  // === 描述与提示 ===
  description(input, options): Promise<string>
  prompt(options): Promise<string>

  // === 渲染 ===
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, options): React.ReactNode
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode

  // === 分类 ===
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isConcurrencySafe(input): boolean
  isSearchOrReadCommand?(input): { isSearch, isRead, isList }

  // === 安全 ===
  maxResultSizeChars: number             // 结果持久化阈值
  strict?: boolean                        // 严格模式
}
```

### ToolResult 结构

```typescript
type ToolResult<T> = {
  data: T                                 // 工具输出数据
  newMessages?: Message[]                 // 附加消息
  contextModifier?: (ctx: ToolUseContext) => ToolUseContext  // 上下文修改
  mcpMeta?: { _meta?, structuredContent? } // MCP 元数据
}
```

### buildTool 工厂

构建者模式，填充常见默认值，简化工具定义：

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  }
}

// 默认值
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,     // fail-closed 策略
  isReadOnly: () => false,
  isDestructive: () => false,
  checkPermissions: () => ({ behavior: 'allow', updatedInput }),
}
```

## ToolUseContext

全局工具执行上下文，跨所有工具共享：

```typescript
type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mcpClients: MCPServerConnection[]
    mainLoopModel: string
    thinkingConfig: ThinkingConfig
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    isNonInteractiveSession: boolean
  }

  // 状态
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev) => AppState): void
  messages: Message[]

  // 权限
  toolPermissionContext: ToolPermissionContext  // Deep Immutable

  // 进度与 UI
  setToolJSX?: SetToolJSXFn
  addNotification?: (notif) => void

  // 代理（仅子代理上下文）
  agentId?: AgentId
  agentType?: string
}
```

## 工具注册表

**文件**: `tools.ts` (~390行)

### 三层获取

```typescript
// 1. 所有基础工具（唯一真实来源）
export function getAllBaseTools(): Tools

// 2. 按权限上下文过滤
export function getTools(permissionContext: ToolPermissionContext): Tools

// 3. 合并内置 + MCP 工具
export function assembleToolPool(permissionContext, mcpTools): Tools
```

### 工具池组装

```typescript
export function assembleToolPool(permissionContext, mcpTools) {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // 内置工具优先，按名称去重
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

### 完整工具清单

**核心工具**（无条件）:

| 工具 | 用途 |
|------|------|
| `Bash` | 执行 shell 命令 |
| `FileRead` | 读取文件内容（支持图片/PDF/Notebook） |
| `FileEdit` | 精确字符串替换编辑 |
| `FileWrite` | 创建/覆写文件 |
| `Glob` | 文件模式匹配搜索 |
| `Grep` | 内容正则搜索（基于 ripgrep） |
| `Agent` | 启动子代理 |
| `SendMessage` | 向运行中的代理发消息 |
| `WebFetch` | 获取网页内容 |
| `WebSearch` | 网络搜索 |
| `AskUserQuestion` | 向用户提问 |
| `TodoWrite` | 写入 TODO 列表 |
| `Brief` | 简短输出模式 |
| `ToolSearch` | 搜索可用工具（延迟加载） |
| `Skill` | 调用技能 |
| `EnterPlanMode` / `ExitPlanMode` | 进入/退出计划模式 |
| `TaskCreate/Get/List/Output/Stop/Update` | 任务管理 |
| `TeamCreate` / `TeamDelete` | 团队管理 |
| `NotebookEdit` | Jupyter Notebook 编辑 |
| `LSP` | Language Server Protocol 调用 |
| `McpAuth` | MCP 认证 |
| `MCPTool` | MCP 工具代理 |
| `ListMcpResources` / `ReadMcpResource` | MCP 资源访问 |
| `RemoteTrigger` | 远程触发器 |
| `ScheduleCron` | 定时任务 |
| `Sleep` | 休眠工具 |

**条件工具**（特性门控）:

| 工具 | 门控条件 |
|------|---------|
| `REPL` | `USER_TYPE='ant'` |
| `CronTools` | `AGENT_TRIGGERS` |
| `WorkflowTool` | `WORKFLOW_SCRIPTS` |
| `WebBrowserTool` | `WEB_BROWSER_TOOL` |
| `EnterWorktree/ExitWorktree` | worktree 模式 |

## 权限系统

**文件**: `utils/permissions/permissions.ts` (~800行)

### 权限模式

```typescript
type PermissionMode =
  | 'default'           // 提示所有不安全操作
  | 'bypassPermissions' // 跳过所有提示
  | 'auto'              // 使用分类器自动审批
  | 'plan'              // 计划模式，仅显示不执行
```

### 权限决策流

```
1. validateInput()
   ↓ 合法 / 拒绝
2. tool.checkPermissions()
   ↓ allow / ask / deny
3. 检查 always-allow / always-deny 规则
   ↓
4. 运行权限钩子
   ↓
5. [auto 模式] 运行分类器
   ↓
6. [default 模式] 显示权限提示给用户
```

### 权限规则

```typescript
type PermissionRule = {
  source: 'policySettings' | 'projectSettings' | 'userSettings'
        | 'localSettings' | 'cliArg' | 'command' | 'session'
  ruleBehavior: 'allow' | 'deny'
  ruleValue: PermissionRuleValue
}
```

### 权限上下文

```typescript
type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  shouldAvoidPermissionPrompts?: boolean     // 后台代理无 UI
}>
```

### Auto 模式分类器

**文件**: `utils/permissions/classifierDecision.ts`

```typescript
const classifyYoloAction = (action, context) => {
  // 将工具调用编码为紧凑字符串 (如 "Bash(git push)")
  // 调用模型分类器
  // 缓存决策避免重复调用
  // 拒绝时回退到权限提示
  return { approved: boolean, reason: string }
}
```

### 否认追踪

防止分类器失败导致无限重试：

```typescript
const DENIAL_LIMITS = {
  PER_TOOL_WINDOW: 5,           // 5秒窗口内最多拒绝次数
  FALLBACK_THRESHOLD: 3,         // 3次后回退到权限提示
  SESSION_RESET_MS: 60_000,      // 60秒重置计数器
}
```

## Hooks 系统

**文件**: `utils/hooks/`

三类 hooks 分别在工具执行的不同阶段介入：

### PreToolUse Hooks

在工具执行**前**触发，可修改输入或阻止执行：

```typescript
type PreToolUseHook = {
  if?: string                              // 条件匹配
  updatedInput?: Record<string, unknown>   // 修改输入
  deny?: { reason: string }                // 阻止执行
}
```

### PostToolUse Hooks

在工具完成**后**触发，可修改结果：

```typescript
type PostToolUseHook = {
  if?: string
  replaceResult?: { content: string }      // 替换结果
}
```

### Permission Request Hooks

在权限提示**前**触发，可修改规则：

```typescript
type PermissionRequestHook = {
  if?: string
  deny?: { reason: string }
  allow?: true
}
```

### 条件匹配

工具可实现自定义匹配器支持参数级条件：

```typescript
async preparePermissionMatcher(input) {
  // BashTool 示例：解析命令，返回子命令匹配器
  const parsed = await parseForSecurity(input.command)
  return (pattern) => parsed.commands.some(cmd => matchPattern(pattern, cmd))
}
```

## 工具延迟加载

**文件**: `utils/toolSearch.ts`, `tools/ToolSearchTool/`

当工具数量超过阈值时，延迟工具描述以节省 token：

```typescript
export function isToolSearchEnabledOptimistic(): boolean {
  return (getAllBaseTools().length + mcpToolCount) > TOOL_SEARCH_THRESHOLD
}
```

流程：
1. 初始提示仅包含 `ToolSearchTool`
2. 模型调用 `ToolSearch(query)` 按关键词搜索
3. 返回匹配工具的完整定义
4. 模型在下一轮使用返回的工具

每个工具的 `searchHint` 字段（3-10 字）用于关键词匹配。

## 工具实现案例

### BashTool

**文件**: `tools/BashTool/BashTool.tsx` (1000+ 行)

```typescript
const BashTool = buildTool({
  name: 'Bash',
  maxResultSizeChars: 30_000,

  inputSchema: z.object({
    command: z.string(),
    description: z.string().optional(),
    timeout_ms: z.number().optional(),
    run_in_background: z.boolean().optional(),
  }),

  async call(input, context, _canUseTool, parentMessage, onProgress) {
    // 异步生成器逐步 yield 进度
    const generator = runShellCommand({ input, abortController, ... })
    let result
    do {
      result = await generator.next()
      if (!result.done) onProgress?.({ type: 'bash_progress', ... })
    } while (!result.done)
    return { data: result.value }
  },

  checkPermissions: (input, context) => bashToolHasPermission(input, context),
  isReadOnly: (input) => isReadOnlyCommand(input.command),
  isConcurrencySafe: () => true,
})
```

### FileReadTool

**文件**: `tools/FileReadTool/FileReadTool.ts` (600+ 行)

特点：
- **图片处理**: 检测 PNG/JPG，下采样至 token 限制
- **PDF 支持**: 按页范围提取
- **Notebook**: 转换 Jupyter 单元为结构化内容
- **技能发现**: 自动发现和加载技能目录

### FileEditTool vs FileWriteTool

| 维度 | FileEdit | FileWrite |
|------|----------|-----------|
| 模式 | 精确字符串替换 | 创建/完整覆写 |
| 粒度 | 基于 `old_string → new_string` | 整个文件内容 |
| 权限 | 针对内容行 | 针对路径 |
| 结果展示 | diff 格式 | 内容确认 |
| 典型场景 | 修改现有代码 | 创建新文件 |

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `Tool.ts` | Tool 接口定义、buildTool 工厂、ToolUseContext |
| `tools.ts` | 工具注册表、工具池组装 |
| `tools/BashTool/` | Shell 命令执行 |
| `tools/FileReadTool/` | 文件/图片/PDF/Notebook 读取 |
| `tools/FileEditTool/` | 精确字符串替换编辑 |
| `tools/FileWriteTool/` | 文件创建/覆写 |
| `tools/AgentTool/` | 子代理启动 |
| `tools/ToolSearchTool/` | 工具延迟搜索 |
| `services/tools/toolExecution.ts` | 工具执行编排 |
| `utils/permissions/permissions.ts` | 权限系统核心 |
| `utils/permissions/classifierDecision.ts` | Auto 模式分类器 |
