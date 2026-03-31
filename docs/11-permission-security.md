# 11 - 权限模型与安全机制

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 实现了多层权限系统，确保工具执行的安全性。包括四种权限模式、多源规则、自动分类器、沙箱执行和否认追踪等机制。

## 权限模式

```typescript
type PermissionMode =
  | 'default'            // 提示所有不安全操作
  | 'auto'               // 使用分类器自动审批安全操作
  | 'bypassPermissions'  // 跳过所有提示（需显式启用）
  | 'plan'               // 计划模式，仅显示不执行
```

## 权限决策流

```
1. validateInput()
   ↓ 合法 / 拒绝

2. tool.checkPermissions()
   ↓ allow / ask / deny

3. 检查 always-allow / always-deny 规则
   ↓

4. 运行权限请求钩子 (PermissionRequestHooks)
   ↓

5. [auto 模式] 运行分类器 (classifierDecision)
   ↓

6. [default 模式] 显示权限提示给用户
   ↓

7. 应用结果 (allow + updatedInput / deny + reason)
```

## 权限规则

### 规则结构

```typescript
type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: 'allow' | 'deny'
  ruleValue: PermissionRuleValue      // 工具名/glob 模式/bash 命令
}
```

### 规则来源优先级

```
policySettings (企业策略) — 最高
projectSettings (.claude/settings.json)
userSettings (~/.claude/settings.json)
localSettings (.claude/settings.local.json)
cliArg (--allowedTools / --disallowedTools)
command (会话内命令)
session (运行时会话规则) — 最低
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
  shouldAvoidPermissionPrompts?: boolean    // 后台代理无 UI
  awaitAutomatedChecksBeforeDialog?: boolean
}>
```

## Auto 模式分类器

**文件**: `utils/permissions/classifierDecision.ts`

### 工作原理

```typescript
const classifyYoloAction = (action, context) => {
  // 1. 将工具调用编码为紧凑字符串 (如 "Bash(git push)")
  // 2. 调用模型分类器判断安全性
  // 3. 缓存决策避免重复调用
  // 4. 拒绝时回退到权限提示
  return { approved: boolean, reason: string }
}
```

### 否认追踪

**文件**: `utils/permissions/denialTracking.ts`

防止分类器失败导致无限重试：

```typescript
const DENIAL_LIMITS = {
  PER_TOOL_WINDOW: 5,          // 5 秒窗口内最多拒绝次数
  FALLBACK_THRESHOLD: 3,        // 3 次后回退到权限提示
  SESSION_RESET_MS: 60_000,     // 60 秒重置计数器
}
```

## 工具级权限

### BashTool 权限

```typescript
// bashToolHasPermission():
// 1. 解析命令安全性 (parseForSecurity)
// 2. 检查是否为只读命令
// 3. 检查沙箱要求
// 4. 匹配 always-allow 规则
// 5. 返回 allow/ask/deny
```

支持参数级条件匹配：

```typescript
// preparePermissionMatcher():
// 解析 bash 命令，返回子命令匹配器
// 规则如 "Bash(git push)" 可精确匹配特定命令
```

### FileRead/FileEdit/FileWrite 权限

- FileRead: 检查路径有效性、块设备
- FileEdit: 基于内容行的权限
- FileWrite: 基于路径的权限

## 沙箱系统

**文件**: `utils/sandbox/`, `utils/permissions/shouldUseSandbox.ts`

### 沙箱条件

```typescript
function shouldUseSandbox(command: string): boolean {
  // 检查命令类型和安全级别
  // SandboxManager 集成
}
```

### SandboxManager

管理 bash 命令的沙箱化执行，隔离文件系统访问。

## 信任边界

### TrustDialog

**文件**: `components/TrustDialog/`

在交互模式中，REPL 渲染前显示信任对话框：
- 工作区 CLAUDE.md 检查（是否包含外部引用）
- MCP 服务器审批
- API 密钥确认
- AutoMode 同意

### 信任流程

```
showSetupScreens()
  ├─ Onboarding (首次运行)
  ├─ TrustDialog (工作区信任)
  ├─ MCP 服务审批
  ├─ API 密钥审批
  ├─ AutoMode 同意
  └─ [信任后] 应用环境变量和钩子
```

## 权限 UI

**文件**: `components/permissions/`

20+ 个权限对话框组件：

| 组件 | 用途 |
|------|------|
| `BashPermissionRequest` | Shell 命令执行确认 |
| `FilePermissionDialog` | 文件读/写确认 |
| `WebFetchPermissionRequest` | 网络请求确认 |
| `SkillPermissionRequest` | 技能执行确认 |
| `ComputerUseApproval` | 屏幕控制确认 |

## Hooks 安全

### PreToolUse Hooks

在工具执行前介入，可修改输入或阻止：

```typescript
type PreToolUseHook = {
  if?: string                    // 条件匹配
  updatedInput?: Record<string, unknown>
  deny?: { reason: string }
}
```

### PostToolUse Hooks

在工具完成后介入，可修改结果：

```typescript
type PostToolUseHook = {
  if?: string
  replaceResult?: { content: string }
}
```

### Hook 超时

工具 hooks 默认 10 分钟超时，防止阻塞。

## 安全存储

**文件**: `utils/secureStorage/`

- macOS Keychain 集成（启动时异步预取）
- 敏感插件配置存储到 keychain
- 令牌文件权限受限
- 支持 prctl 不可转储标志

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `utils/permissions/permissions.ts` (800+行) | 权限系统核心 |
| `utils/permissions/classifierDecision.ts` | Auto 模式分类器 |
| `utils/permissions/denialTracking.ts` | 否认追踪 |
| `utils/permissions/PermissionMode.ts` | 权限模式定义 |
| `utils/permissions/PermissionRule.ts` | 规则类型 |
| `utils/sandbox/` | 沙箱管理 |
| `utils/secureStorage/` | 安全存储 |
| `hooks/toolPermission/` | 权限钩子集成 |
| `components/permissions/` | 权限 UI 组件 |
| `components/TrustDialog/` | 信任对话框 |
