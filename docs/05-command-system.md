# 05 - 命令系统

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 的命令系统管理 80+ 个斜杠命令，支持内置命令、插件命令、技能命令和工作流命令。通过中央注册表 `commands.ts` 统一管理，支持懒加载、特性门控和多来源聚合。

## 命令类型

```typescript
Command = CommandBase + (PromptCommand | LocalCommand | LocalJSXCommand)
```

| 类型 | 特征 | 用途 |
|------|------|------|
| `prompt` | 返回 `ContentBlockParam[]` | 向模型传递指令（技能） |
| `local` | 返回 `LocalCommandResult` | 执行本地操作，返回纯文本 |
| `local-jsx` | 返回 `React.ReactNode` | 渲染完整 Ink UI 组件 |

## CommandBase 接口

```typescript
type CommandBase = {
  name: string
  aliases?: string[]
  description: string
  isEnabled?: () => boolean        // 特性门控
  isHidden?: boolean               // 对用户隐藏
  availability?: ('claude-ai' | 'console')[]  // 认证要求
  argumentHint?: string            // UI 参数提示
  immediate?: boolean              // 跳过队列立即执行
  isSensitive?: boolean            // 参数脱敏
  source: 'builtin' | 'plugin' | 'bundled' | 'skills' | 'mcp'
  kind?: 'workflow'                // 工作流命令标识
}
```

## 命令加载策略

```
loadAllCommands(cwd)
  ├─ getSkills(cwd)
  │   ├─ skillDirCommands (从 .claude/skills/ 目录)
  │   ├─ pluginSkills (从已启用插件)
  │   ├─ bundledSkills (预编译内置)
  │   └─ builtinPluginSkills (内置插件)
  ├─ getPluginCommands() (插件市场命令)
  ├─ getWorkflowCommands(cwd) (工作流脚本)
  └─ COMMANDS() (内置命令)
      └─ 动态技能去重 (getDynamicSkills)
```

## 可用性三层过滤

1. **`availability`** (静态) — 认证/提供商要求
2. **`isEnabled()`** (动态) — 特性门控、环境变量，每次 `getCommands()` 重新评估
3. **远程安全白名单** — `REMOTE_SAFE_COMMANDS`, `BRIDGE_SAFE_COMMANDS`

## 命令实现模式

### Prompt 命令（技能调用）

```typescript
// commands/review.ts
const review: Command = {
  type: 'prompt',
  name: 'review',
  description: 'Review a pull request',
  async getPromptForCommand(args) {
    return [{ type: 'text', text: LOCAL_REVIEW_PROMPT(args) }]
  },
}
```

### Local-JSX 命令（交互 UI）

```typescript
// commands/plan/index.ts — 懒加载模式
const plan = {
  type: 'local-jsx',
  name: 'plan',
  description: 'Enable plan mode or view the current session plan',
  argumentHint: '[open|<description>]',
  load: () => import('./plan.js'),  // 避免启动时加载重 UI 组件
} satisfies Command
```

### Local 命令（纯文本结果）

```typescript
// commands/compact/index.ts
const compact = {
  type: 'local',
  name: 'compact',
  supportsNonInteractive: true,
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_COMPACT),
  load: () => import('./compact.js'),
}
```

### 动态描述命令

```typescript
// commands/model/index.ts — getter 动态计算
export default {
  type: 'local-jsx',
  name: 'model',
  get description() {
    return `Set the AI model (currently ${renderModelName(...)})`
  },
  get immediate() {
    return shouldInferenceConfigCommandBeImmediate()
  },
  load: () => import('./model.js'),
}
```

## 主要内置命令

| 命令 | 类型 | 用途 |
|------|------|------|
| `/plan` | local-jsx | 计划模式管理 |
| `/compact` | local | 上下文紧凑化 |
| `/model` | local-jsx | 模型选择 |
| `/permissions` | local-jsx | 权限规则管理 |
| `/review` | prompt | PR 代码审查 |
| `/voice` | local | 语音模式切换 |
| `/plugin` | local-jsx | 插件管理 |
| `/mcp` | local-jsx | MCP 服务器管理 |
| `/tasks` | local-jsx | 任务管理 |
| `/help` | local-jsx | 帮助信息 |
| `/clear` | local | 清除会话 |
| `/config` | local-jsx | 配置管理 |
| `/resume` | local | 恢复会话 |
| `/share` | local | 分享对话 |
| `/cost` | local | 成本统计 |
| `/vim` | local | Vim 模式切换 |
| `/diff` | local | 查看更改差异 |

## 命令查询

```typescript
findCommand(name, commands): Command | undefined
  // 按 name → getCommandName() → aliases 查询

formatDescriptionWithSource(cmd): string
  // 插件命令 → "(PluginName) description"
  // 打包命令 → "description (bundled)"
```

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `commands.ts` (755行) | 中央命令注册表、加载逻辑 |
| `types/command.ts` (217行) | Command 接口定义 |
| `commands/` (80+ 子目录) | 各命令实现 |
