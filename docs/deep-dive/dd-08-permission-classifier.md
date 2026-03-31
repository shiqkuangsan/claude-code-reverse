# Deep Dive 08 - Permission Classifier（权限分类器与 Auto 模式）

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 的权限系统由**多层决策机制**构成：静态规则 → 钩子 → 分类器 → UI 提示。Auto 模式使用 YOLO 分类器（基于 Claude API）自动判断操作安全性。Bash 命令经过 AST 级别的安全解析。

## 权限模式

| 模式 | 行为 | 快捷键循环 |
|------|------|-----------|
| `default` | 每个操作都需确认 | ✓ |
| `acceptEdits` | CWD 内编辑自动允许 | ✓ |
| `plan` | 审查模式，操作暂停 | ✓ |
| `bypassPermissions` | 允许所有（企业） | ✓ |
| `auto` | 分类器自动决策（ANT only） | ✓ |

## 权限决策流

```
1. validateInput()               → 输入合法性
2. 检查 deny 规则                → 最高优先级，立即拒绝
3. 检查 allow 规则               → glob 模式匹配
4. 执行 PermissionRequest hooks  → 外部干预
5. YOLO 分类器（auto 模式）       → Claude API 安全评估
6. 权限确认 UI                   → 用户交互决策
```

## YOLO 分类器

**文件**: `utils/permissions/yoloClassifier.ts`

### 两阶段分类（默认 'both' 模式）

| 阶段 | max_tokens | 目的 |
|------|-----------|------|
| Stage 1 (快速) | 64 | 即时决策，`stop_sequences=['</block>']` |
| Stage 2 (思考) | 4096 | 链式思考减少假阳性 |

### XML 输出格式

```xml
阻止: <block>yes</block><reason>one short sentence</reason>
允许: <block>no</block>
```

### 转录构建

分类器接收完整会话转录（JSONL 格式）：
```json
{"user":"setup command"}
{"Bash":"ls -la"}
{"FileEdit":{"path":"file.txt","newContent":"..."}}
```

工具输入通过 `tool.toAutoClassifierInput()` 投影，自动屏蔽敏感参数。

### 白名单工具（跳过分类器）

FileRead, Grep, Glob, LSP, ToolSearch, TODO, Task 命令, AskUserQuestion, PlanMode, Team 操作, SendMessage, Sleep 等。

### 缓存

系统提示 + CLAUDE.md 使用 `cache_control`（1h TTL），跨多次分类器调用共享。

## 否认追踪

```typescript
DENIAL_LIMITS = {
  maxConsecutive: 3,    // 连续拒绝 3 次后降级
  maxTotal: 20          // 总拒绝 20 次后降级
}
```

超过限制 → 禁用分类器 → 回退到用户确认提示。成功执行清零连续计数。

## 权限规则系统

### 规则源优先级

```
policySettings (企业策略) — 最高
globalSettings (~/.claude/settings.json)
workspaceSettings (.claude/settings.json)
localSettings (.claude/settings.local.json)
cliArg (--allow-tools / --deny-tools)
command (运行时)
session (临时) — 最低
```

### 规则格式

| 格式 | 示例 | 含义 |
|------|------|------|
| 工具级 | `Bash` | 允许所有 Bash |
| 前缀 | `Bash(npm:*)` | 匹配 npm 前缀 |
| 通配符 | `Bash(python*)` | glob 匹配 |
| 精确 | `Bash(npm install)` | 精确匹配 |

### Shell 规则匹配

安全包装器自动剥离：`env`, `timeout`, `strace`, `nohup`, `xargs` 等。
```
timeout 300 npm run build  →  匹配 Bash(npm run:*)
```

## Bash 安全解析

### 23 项安全检查

| # | 检查项 |
|---|--------|
| 1 | 不完整命令 |
| 2-3 | jq system() / 文件参数 |
| 4-6 | 混淆标志 / 壳元字符 / 危险变量 |
| 7-8 | 换行符 / 命令替换 |
| 9-10 | 输入/输出重定向 |
| 11-13 | IFS 注入 / Git commit 替换 / /proc/self/environ |
| 14-18 | 畸形令牌 / 反斜杠空格 / 花括号展开 / 控制字符 / Unicode 空格 |
| 19-23 | 中间 # / Zsh 危险 / 反斜杠操作符 / 注释不同步 / 带引号换行 |

### 危险命令模式

```typescript
// 命令替换
$()  ${}  $[]  <()  >()  ``

// Zsh 特有
zmodload, emulate, sysopen, zpty, ztcp, mapfile
```

## 文件系统权限

### 路径验证流程

```
1. deny 规则 → 拒绝
2. 内部可编辑路径 (.claude/plans/, scratchpad.md, agent 内存)
3. 安全检查 (危险文件/目录)
4. 工作目录检查 (acceptEdits 模式)
5. 内部可读路径 (代理输出, 会话记忆)
6. 沙箱白名单
7. allow 规则 → 允许
```

### 危险文件/目录

```
文件: .gitconfig, .bashrc, .zshrc, .profile, .mcp.json
目录: .git, .vscode, .idea, .claude
```

## 沙箱机制

- SandboxManager 管理 Bash 命令沙箱化
- 网络白名单从 WebFetch 规则提取
- 文件系统 allowWrite/denyWrite 规则

## 危险权限检测

启动时扫描所有权限规则，识别危险模式：
- `Bash(*)` — 允许所有命令
- `Bash(python:*)` — 允许任何 python
- `Agent(*)` — 绕过子代理分类器
- PowerShell: `Invoke-Expression`, `Start-Process` 等

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `utils/permissions/permissions.ts` (800+行) | 权限核心逻辑 |
| `utils/permissions/yoloClassifier.ts` | YOLO 分类器 |
| `utils/permissions/classifierDecision.ts` | 分类器决策路由 |
| `utils/permissions/denialTracking.ts` | 否认追踪 |
| `utils/permissions/PermissionMode.ts` | 模式定义 |
| `utils/permissions/pathValidation.ts` | 路径验证 |
| `utils/permissions/filesystem.ts` | 文件系统权限 |
| `utils/bash/bashSecurity.ts` | Bash 安全解析 |
| `utils/sandbox/` | 沙箱管理 |
| `components/permissions/` | 权限 UI 组件 |
