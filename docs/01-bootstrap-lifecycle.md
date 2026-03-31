# 01 - 启动流程与生命周期

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 的启动流程经过精心优化，通过并行化子进程、延迟加载大型模块、fire-and-forget 后台任务来最小化启动延迟。整个过程可分为 **10 个阶段**。

## 启动时序

```
T=0ms   │ entrypoints/cli.tsx — 快速路径检查
        │   --version / --daemon / --bridge 等特殊模式直接返回
        │
T=10ms  │ main.tsx 入口
        │   startMdmRawRead()        [后台] MDM 配置读取
        │   startKeychainPrefetch()  [后台] macOS Keychain 预取
        │   开始导入 ~200 个模块 (~135ms)
        │
T=140ms │ 模块导入完成
        │   早期 argv 解析（cc:// URL、assistant mode、SSH 等）
        │   交互模式检测 / Client Type 判定
        │   eagerLoadSettings()
        │
T=160ms │ Commander.js preAction hook
        │   await MDM + Keychain 完成
        │   await init()     ← 全局初始化
        │   runMigrations()
        │
T=250ms │ setup() — 会话级初始化
        │   setCwd() / captureHooksConfigSnapshot()
        │   [if --worktree] createWorktreeForSession()
        │   initSessionMemory()
        │
T=300ms │ 路径分流
        │   HEADLESS (-p): connectMcp → runHeadless → query loop
        │   INTERACTIVE:   createRoot → showSetupScreens → launchRepl
        │
T=2000+ │ 用户交互 / API 调用 / 工具执行
```

## 阶段详解

### 阶段 1: 快速路径（Fast Path）

**文件**: `entrypoints/cli.tsx`

CLI 入口先检查是否命中快速路径，避免加载完整模块：

| 标志 | 行为 | 模块加载 |
|------|------|---------|
| `--version` / `-v` | 直接输出版本号 | 零 |
| `--dump-system-prompt` | 输出 system prompt | 最小 |
| `--chrome-native-host` | MCP 服务器启动 | Chrome 相关 |
| `--daemon-worker` | 内部工作进程 | daemon 相关 |
| `--bg` / `ps` / `logs` | 后台会话管理 | 后台相关 |

### 阶段 2: 极早初始化

**文件**: `main.tsx:1-20`

三个异步操作**必须在重模块导入之前启动**，以便与 ~135ms 的模块加载时间重叠：

```typescript
profileCheckpoint('main_tsx_entry')
startMdmRawRead()           // 子进程读取 MDM 配置 (plutil/reg query)
startKeychainPrefetch()     // macOS Keychain 同步读取 (~65ms)
```

### 阶段 3: 模块导入

**文件**: `main.tsx:21-209`

导入约 200 个模块，包括 Commander.js、React/Ink、配置、认证、服务等。部分模块通过 `feature()` 编译时门控实现死代码消除：

```typescript
// 外部构建中这些被编译时移除
import { isCoordinatorMode } from './coordinator/coordinatorMode.js'
import { initKairos } from './assistant/index.js'
```

### 阶段 4: 早期标志解析

**文件**: `main.tsx:520-856`

在完整初始化之前解析关键决策：

```typescript
// 交互模式检测
const isNonInteractive = hasPrintFlag || hasInitOnlyFlag || !process.stdout.isTTY
setIsInteractive(!isNonInteractive)

// Client Type 检测
// github-action / sdk-typescript / sdk-python / sdk-cli /
// claude-vscode / local-agent / claude-desktop / remote / cli
```

### 阶段 5: init() — 全局初始化

**文件**: `entrypoints/init.ts`

通过 `lodash.memoize` 保证只执行一次：

```typescript
export const init = memoize(async () => {
  enableConfigs()                              // 配置启用
  applySafeConfigEnvironmentVariables()        // 安全环境变量
  applyExtraCACertsFromConfig()                // TLS 证书（必须在任何连接前）
  setupGracefulShutdown()                      // 优雅关闭
  configureGlobalMTLS()                        // mTLS 配置
  configureGlobalAgents()                      // HTTP Agent 配置
  preconnectAnthropicApi()                     // [后台] TCP+TLS 预连接
})
```

### 阶段 6: 数据迁移

**文件**: `main.tsx:326-352`

版本化迁移系统（当前版本 11），处理模型重命名和功能演变：

```typescript
const CURRENT_MIGRATION_VERSION = 11

function runMigrations() {
  migrateAutoUpdatesToSettings()
  migrateSonnet45ToSonnet46()
  migrateOpusToOpus1m()
  // ... 共 10+ 个迁移
  saveGlobalConfig(prev => ({ ...prev, migrationVersion: CURRENT_MIGRATION_VERSION }))
}
```

### 阶段 7: setup() — 会话初始化

**文件**: `setup.ts:56-400+`

会话级配置，处理工作目录、hooks、worktree：

```typescript
export async function setup(cwd, permissionMode, ...) {
  setCwd(cwd)                           // 必须最先执行
  captureHooksConfigSnapshot()          // Hooks 配置快照
  initializeFileChangedWatcher(cwd)     // 文件变更监听

  if (worktreeEnabled) {
    const session = await createWorktreeForSession(getSessionId(), slug)
    process.chdir(session.worktreePath)
    setCwd(session.worktreePath)
  }

  initSessionMemory()                   // 会话记忆
  void getCommands(getProjectRoot())    // [后台] 预加载命令
  void loadPluginHooks()                // [后台] 加载插件 hooks
}
```

### 阶段 8: 权限初始化

**文件**: `main.tsx:1747-1772`

```typescript
const initResult = await initializeToolPermissionContext({
  allowedToolsCli, disallowedToolsCli, baseToolsCli,
  permissionMode, allowDangerouslySkipPermissions, addDirs
})
```

### 阶段 9: 交互路径分流

**Headless 模式** (`-p/--print`):
```
validateForceLoginOrg() → connectMcpBatch() → runHeadless() → query loop
```

**交互模式** (默认):
```
createRoot() → showSetupScreens() → [信任对话框/OAuth/Onboarding] → launchRepl()
```

### 阶段 10: REPL 启动

**文件**: `replLauncher.tsx`

```typescript
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js')
  const { REPL } = await import('./screens/REPL.js')
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)
}
```

## Bootstrap State

**文件**: `bootstrap/state.ts`

全局状态容器，贯穿整个生命周期：

```typescript
type State = {
  originalCwd: string              // 启动时 CWD
  projectRoot: string              // 项目根目录
  cwd: string                      // 当前工作目录
  sessionId: SessionId             // 会话 ID
  isInteractive: boolean           // 交互模式
  clientType: string               // 客户端类型
  mainLoopModelOverride: ModelSetting | undefined
  toolPermissionContext: ToolPermissionContext
  modelUsage: { [model: string]: ModelUsage }
  // ...
}
```

## CLI 选项体系

Commander.js 定义了 100+ 个选项，主要类别：

| 类别 | 关键标志 |
|------|---------|
| 交互模式 | `-p/--print`, `--bare`, `--init-only` |
| 认证/权限 | `--dangerously-skip-permissions`, `--permission-mode` |
| 模型/推理 | `--model`, `--effort`, `--thinking`, `--max-thinking-tokens` |
| 会话管理 | `-c/--continue`, `-r/--resume`, `--session-id`, `--fork-session` |
| 输入/输出 | `--output-format`, `--input-format`, `--json-schema` |
| 工具/MCP | `--tools`, `--mcp-config`, `--allowedTools`, `--disallowedTools` |
| 特殊功能 | `--worktree`, `--tmux`, `--ide`, `--chrome`, `--agent` |
| 远程 | `--remote`, `--remote-control`, `--teleport` |

## 优雅关闭

```typescript
// 退出时重置终端光标
process.on('exit', () => resetCursor())

// SIGINT 处理
process.on('SIGINT', () => {
  if (isPrintMode) return  // print.ts 有自己的处理器
  process.exit(0)
})

// 通过 registerCleanup 注册的清理函数：
// - shutdownLspServerManager
// - cleanupSessionTeams
// - 更多...
```

## 性能优化策略

1. **并行化**: MDM/Keychain 读取与模块导入并行
2. **延迟加载**: OpenTelemetry (~400KB)、gRPC (~700KB) 按需加载
3. **Fire-and-forget**: 预取、hooks、分析等非阻塞后台任务
4. **快速路径**: 特殊 CLI 命令跳过完整初始化
5. **TCP 预连接**: `preconnectAnthropicApi()` 在用户交互期间完成握手
6. **编译时门控**: `feature()` 实现死代码消除

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `entrypoints/cli.tsx` | CLI 入口网关，快速路径分发 |
| `main.tsx` (4683行) | 核心启动逻辑，Commander.js 配置 |
| `entrypoints/init.ts` | 全局初始化（memoized） |
| `setup.ts` | 会话级初始化 |
| `bootstrap/state.ts` | 全局 Bootstrap 状态管理 |
| `ink.ts` | Ink 终端 UI 包装层 |
| `replLauncher.tsx` | REPL 启动入口 |
| `interactiveHelpers.tsx` | 设置屏幕（信任对话框等） |
