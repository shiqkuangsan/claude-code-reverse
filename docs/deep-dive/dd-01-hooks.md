# Deep Dive 01 - Hooks 机制全解

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 的 Hooks 系统是一套**事件驱动的生命周期钩子机制**，支持 26 个事件类型、4 种 Hook 类型（bash/prompt/agent/http）、同步/异步执行、条件匹配和多源聚合。Hooks 是确定性的 shell 命令——Claude 无法跳过它们。

## 26 个 Hook 事件

### 会话级事件

| 事件 | 触发点 | 用途 |
|------|--------|------|
| `Setup` | 仓库初始化/维护时 | 环境准备 |
| `SessionStart` | 会话开始（startup/resume/clear/compact） | 初始化、环境变量注入 |
| `SessionEnd` | 会话结束（超时 1.5s） | 清理 |
| `UserPromptSubmit` | 用户提交提示词后 | 输入验证/增强 |

### 工具执行事件

| 事件 | 触发点 | 用途 |
|------|--------|------|
| `PreToolUse` | 工具执行前 | 权限决策、输入修改 |
| `PostToolUse` | 工具成功执行后 | 格式化、验证 |
| `PostToolUseFailure` | 工具执行失败时 | 故障排查 |
| `PermissionRequest` | 权限对话显示时 | 自动化权限决策 |

### Stop 和子代理事件

| 事件 | 触发点 | 用途 |
|------|--------|------|
| `Stop` | Claude 生成完成后 | 最终验证（支持阻塞） |
| `SubagentStart` | 子代理启动时 | 监控 |
| `SubagentStop` | 子代理完成时 | 验证 |

### 文件和配置事件

| 事件 | 触发点 | 用途 |
|------|--------|------|
| `CwdChanged` | 工作目录更改时 | 环境重载 |
| `FileChanged` | 监视文件变更时 | 响应式操作 |
| `ConfigChange` | 配置文件更改时 | 热更新 |
| `InstructionsLoaded` | CLAUDE.md/规则文件加载时 | 观测 |

### 其他事件

`PreCompact` / `PostCompact` / `Notification` / `Elicitation` / `ElicitationResult` / `TaskCreated` / `TaskCompleted` / `TeammateIdle` / `WorktreeCreate` / `WorktreeRemove`

## Hook 配置格式

### settings.json 结构

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|Bash",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write $FILE",
            "timeout": 30,
            "if": "Bash(git *)"
          }
        ]
      }
    ]
  }
}
```

### Matcher 语法

| 语法 | 示例 | 含义 |
|------|------|------|
| 精确匹配 | `"Write"` | 匹配 Write 工具 |
| 管道分隔 | `"Write\|Edit"` | 匹配任一 |
| 正则表达式 | `"^Write.*"` | 正则匹配 |
| 通配 | `"*"` 或 `""` | 匹配所有 |

### if 条件（权限规则语法）

```json
{ "if": "Bash(git *)" }       // 仅匹配 git 命令
{ "if": "Write(src/**.ts)" }  // 仅匹配 TS 文件写入
```

仅适用于工具事件（PreToolUse、PostToolUse、PostToolUseFailure、PermissionRequest）。

## 4 种 Hook 类型

### 1. Bash 命令 Hook (type: "command")

```json
{
  "type": "command",
  "command": "bash script or command",
  "shell": "bash",
  "timeout": 30,
  "statusMessage": "Running check...",
  "once": true,
  "async": false,
  "asyncRewake": false,
  "if": "permission_rule"
}
```

**执行流程：**
1. 替换变量：`${CLAUDE_PLUGIN_ROOT}`, `${CLAUDE_PLUGIN_DATA}`, `${user_config.X}`
2. 生成子进程，通过 stdin 传入 JSON 输入
3. 收集 stdout/stderr

**Exit Code 语义：**
- `0` — 成功
- `1` — 非阻塞错误，显示 stderr
- `2` — **阻塞**，显示 stderr 并阻止继续

**环境变量：**
```bash
CLAUDE_PROJECT_DIR      # 项目根目录
CLAUDE_PLUGIN_ROOT      # 插件目录
CLAUDE_PLUGIN_DATA      # 插件数据目录
CLAUDE_PLUGIN_OPTION_*  # 插件用户配置
CLAUDE_ENV_FILE         # 用于 SessionStart/CwdChanged/FileChanged
CLAUDE_SESSION_ID       # 会话 ID
```

**CLAUDE_ENV_FILE 机制：** SessionStart/Setup/CwdChanged/FileChanged Hook 可写入 bash exports：
```bash
export VAR_NAME=value
```
后续 Bash 命令自动注入这些定义。

### 2. Prompt Hook (type: "prompt")

```json
{
  "type": "prompt",
  "prompt": "Evaluate: Is the condition met? Input: $ARGUMENTS",
  "model": "claude-sonnet-4-6",
  "timeout": 30
}
```

**执行流程：**
1. 替换 `$ARGUMENTS` 为 Hook 输入 JSON
2. 调用 LLM（默认 `getSmallFastModel()`，通常 Haiku）
3. LLM 返回 `{"ok": true}` 或 `{"ok": false, "reason": "..."}`

### 3. Agent Hook (type: "agent")

```json
{
  "type": "agent",
  "prompt": "Verify all tests pass by reading test output",
  "model": "claude-sonnet-4-6",
  "timeout": 60
}
```

**执行流程：**
1. 创建专用 Hook 代理实例 (`hook-agent-<uuid>`)
2. 使用 `query()` 启动多轮对话（最多 50 轮）
3. **可调用所有工具**进行验证
4. 必须通过 `StructuredOutput` 工具返回 `{"ok": true/false, "reason": "..."}`
5. 自动注册 Stop Hook 强制 StructuredOutput 调用

### 4. HTTP Hook (type: "http")

```json
{
  "type": "http",
  "url": "https://example.com/webhook",
  "headers": { "Authorization": "Bearer $MY_TOKEN" },
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 300
}
```

**安全特性：**
- URL 白名单：`allowedHttpHookUrls` 策略
- 环境变量白名单：仅 `allowedEnvVars` 中的变量被插值
- CRLF 注入防护：头部值清理
- SSRF 防护：`ssrfGuardedLookup` 验证解析的 IP

## Hook 响应 Schema

### 同步响应

```typescript
{
  // 通用字段
  continue?: boolean,            // false → 阻止继续
  suppressOutput?: boolean,      // 隐藏 stdout
  stopReason?: string,           // continue:false 时的消息
  decision?: 'approve' | 'block',
  reason?: string,
  systemMessage?: string,        // 警告消息

  // 事件特定输出
  hookSpecificOutput?: {
    hookEventName: string,

    // PreToolUse 特定
    permissionDecision?: 'allow' | 'deny' | 'ask',
    updatedInput?: Record<string, unknown>,   // 修改工具输入

    // PostToolUse 特定
    updatedMCPToolOutput?: unknown,

    // SessionStart 特定
    initialUserMessage?: string,              // 注入初始消息
    watchPaths?: string[],                     // 注册文件监视

    // PermissionRequest 特定
    decision?: {
      behavior: 'allow' | 'deny',
      updatedInput?: Record<string, unknown>,
      updatedPermissions?: PermissionUpdate[],
    },

    additionalContext?: string                 // 添加上下文供 Claude 查看
  }
}
```

### 异步响应

```typescript
{ "async": true, "asyncTimeout": 30000 }
```

首行输出后 Hook 立即返回，继续后台执行。通过 `AsyncHookRegistry` 追踪。

### asyncRewake 模式

Hook 在后台运行，完成时：
- exit 0 → 静默成功
- exit 2 → **唤醒模型**，排队任务通知
- 其他 → 仅显示给用户

## Hook 执行引擎

**文件**: `utils/hooks.ts` (~5000行)

### 执行流程

```
1. 信任检查 (shouldSkipHookDueToTrust)
   ↓
2. 获取匹配 Hook (getMatchingHooks)
   ├─ 快照配置（启动时预加载）
   ├─ 已注册 Hook（插件、内置回调）
   └─ 会话 Hook（运行时添加）
   ↓
3. 去重（同插件内相同命令折叠）
   ↓
4. if 条件过滤（权限规则评估）
   ↓
5. 并行执行（各 Hook 独立超时）
   ↓
6. 结果汇聚（合并消息、权限、上下文）
```

### Hook 源优先级

```
userSettings (~/.claude/settings.json)
    ↓
projectSettings (.claude/settings.json)
    ↓
localSettings (.claude/settings.local.json)
    ↓
pluginHook (插件提供)
    ↓
builtinHook (内部回调)
    ↓
sessionHook (运行时添加)
```

### 去重机制

去重键 = `{pluginRoot || skillRoot || ''}\0{payload}`

同一插件内的相同命令被折叠，不同插件的相同命令各自保留。

## 超时配置

| 场景 | 默认超时 | 可覆盖 |
|------|---------|--------|
| 工具 Hook | 10 分钟 | `hook.timeout` 字段 |
| SessionEnd Hook | 1.5 秒 | `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` |
| HTTP Hook | 10 分钟 | `hook.timeout` 字段 |
| Prompt Hook | 30 秒 | `hook.timeout` 字段 |
| Agent Hook | 60 秒 | `hook.timeout` 字段 |

## Stop Hook 特殊行为

**文件**: `query/stopHooks.ts`

Stop Hook 运行在 REPL 上下文中（关键差异）：
- 支持函数 Hook（TypeScript 回调）
- 可阻塞继续对话 (`preventContinuation: true`)
- 并行执行，收集阻塞错误
- exit code 2 → 反馈给模型

## 会话 Hook（运行时）

**文件**: `utils/hooks/sessionHooks.ts`

```typescript
// 添加命令 Hook
addSessionHook(setAppState, sessionId, event, matcher, hook)

// 添加函数 Hook（TypeScript 回调）
addFunctionHook(setAppState, sessionId, event, matcher, callback, errorMessage)

// 移除
removeFunctionHook(setAppState, sessionId, event, hookId)
clearSessionHooks(setAppState, sessionId)
```

## 插件 Hook 集成

插件通过 `plugin.json` 的 `hooks` 字段或 SKILL.md frontmatter 的 `hooks` 字段声明。

加载时机：`setup.ts` 中 `loadPluginHooks()` + `setupPluginHookHotReload()`。

变量替换：`${CLAUDE_PLUGIN_ROOT}` → 插件目录，`${user_config.X}` → 用户配置。

## 企业管控

### allowManagedHooksOnly

启用时仅运行已注册/策略 Hook，设置和会话 Hook 被忽略。

### shouldDisableAllHooksIncludingManaged

完全禁用所有 Hook。

## 实际应用案例

### 格式化后置钩子

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{ "type": "command", "command": "prettier --write $FILE" }]
    }]
  }
}
```

### Git 命令安全检查

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "check-git-auth.sh",
        "if": "Bash(git push*)"
      }]
    }]
  }
}
```

### Agent 验证测试

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "",
      "hooks": [{
        "type": "agent",
        "prompt": "Verify all tests pass by reading test output in the transcript"
      }]
    }]
  }
}
```

### 外部 Webhook 通知

```json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "",
      "hooks": [{
        "type": "http",
        "url": "https://monitoring.example.com/session-started",
        "headers": { "Authorization": "Bearer $TOKEN" },
        "allowedEnvVars": ["TOKEN"]
      }]
    }]
  }
}
```

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `utils/hooks.ts` (~5000行) | 核心执行引擎 |
| `types/hooks.ts` | Hook 类型定义和 Zod Schema |
| `schemas/hooks.ts` | Hook 配置 Schema |
| `utils/hooks/sessionHooks.ts` | 运行时 Hook 管理 |
| `utils/hooks/hooksSettings.ts` | Hook 发现和源管理 |
| `utils/hooks/hooksConfigSnapshot.ts` | 启动时快照 |
| `utils/hooks/execPromptHook.ts` | Prompt Hook 执行 |
| `utils/hooks/execAgentHook.ts` | Agent Hook 执行 |
| `utils/hooks/execHttpHook.ts` | HTTP Hook 执行（含 SSRF 防护） |
| `utils/hooks/AsyncHookRegistry.ts` | 异步 Hook 追踪 |
| `query/stopHooks.ts` | Stop Hook 特殊处理 |
