# 08 - 插件与 Skill 系统

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 的可扩展性通过插件系统和技能系统实现。插件可包含技能、命令、Agent、Hooks、MCP 服务器和 LSP 服务器。技能是用 Markdown + Frontmatter 定义的可调用能力单元。

## 插件架构

### 三层插件来源

```
Built-in Plugins (@builtin)        // 编译时内置
    ↓
Installed Plugins (marketplace)     // 用户安装
    ↓
Local Skills (.claude/skills/)      // 项目本地
```

### plugin.json 清单

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "...",
  "author": { "name": "...", "email": "...", "url": "..." },

  "commands": "./commands/",
  "agents": "./agents/",
  "skills": "./skills/",
  "outputStyles": "./output-styles/",

  "hooks": { "PostToolUse": [{ "type": "bash", "command": "..." }] },
  "mcpServers": { "server-name": { "type": "stdio", "command": "..." } },
  "lspServers": { "ls-name": { ... } },

  "userConfig": {
    "API_KEY": {
      "type": "string",
      "required": true,
      "sensitive": true     // → 存储到 keychain
    }
  },

  "dependencies": ["other-plugin"],
  "settings": { "agent": { ... } }
}
```

### 插件生命周期

```
发现 → 加载 → 验证 → 安装 → 启用 → 执行 → 卸载
  ↓      ↓      ↓      ↓      ↓      ↓      ↓
扫描   读取   Zod    初始化  注册   运行   清理
市场  plugin schema   组件  settings
       .json  check
```

### 安装管理

**文件**: `utils/plugins/installedPluginsManager.ts` (1200+ 行)

V2 安装系统使用 `installed_plugins.json`：

```json
{
  "plugins": {
    "plugin-name@marketplace-name": {
      "scope": "user|project|local|managed",
      "installation_path": "/path/to/plugin",
      "marketplace": "github|npm|url|git|file",
      "version": "1.2.3",
      "sha": "commit-hash"
    }
  }
}
```

### Marketplace 来源

**文件**: `utils/plugins/marketplaceManager.ts` (2800+ 行)

```typescript
type MarketplaceSource =
  | { source: 'github', repo: 'owner/repo', ref?, path? }
  | { source: 'npm', package: 'package-name' }
  | { source: 'url', url: string }
  | { source: 'git', url: string, ref? }
  | { source: 'file', path: string }
  | { source: 'directory', path: string }
  | { source: 'hostPattern', hostPattern: string }
```

### 内置插件

**文件**: `plugins/builtinPlugins.ts`

```typescript
type BuiltinPluginDefinition = {
  name: string
  description: string
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  isAvailable?: () => boolean
  defaultEnabled?: boolean
}
```

启用状态决定：用户设置 → 插件默认 → true

## 技能系统 (Skills)

### SKILL.md 格式

```yaml
---
name: "Display Name"
description: "One-line description"
when-to-use: "Trigger conditions"
allowed-tools: [BashTool, ReadTool]
argument-hint: "[file]"
model: "claude-opus"
context: "fork"                    # fork = 子代理执行
agent: "agent-name"
disable-model-invocation: false
user-invocable: true
effort: "high"
paths: ["src/**", "lib/**"]
hooks:
  PostToolUse: |
    ...
---

# Skill Content

可用变量:
- ${CLAUDE_SKILL_DIR}     # 技能目录路径
- ${CLAUDE_SESSION_ID}    # 会话 ID
- ${ARG_NAME}             # 参数替换
- !`command`              # 内联 bash（非 MCP）
```

### 技能来源

| 来源 | 说明 |
|------|------|
| `skills` | 本地 `.claude/skills/` 目录 |
| `plugin` | 插件提供 |
| `bundled` | 内置编译 |
| `managed` | 企业管理 |
| `mcp` | MCP 服务器提示 |

### 技能加载流程

**文件**: `skills/loadSkillsDir.ts` (1000+ 行)

```
1. 读取 SKILL.md
2. 解析 frontmatter → FrontmatterData
3. 验证 hooks (HooksSchema)
4. 提取字段 (name, description, allowed-tools, model, context...)
5. 创建 Command 对象
6. 构建 getPromptForCommand():
   - 替换 ${CLAUDE_SKILL_DIR}, ${CLAUDE_SESSION_ID}
   - 替换参数 ${ARG_NAME}
   - 执行内联 shell (!`...`)
   - 返回 ContentBlockParam[]
```

### 技能执行 (SkillTool)

**文件**: `tools/SkillTool/SkillTool.ts` (38KB)

两种执行模式：

| 模式 | context 值 | 行为 |
|------|-----------|------|
| Inline | undefined | 在主 agent 中直接运行 |
| Fork | `'fork'` | 在隔离子代理中运行（独立 token 预算、失败隔离） |

```
SkillTool.call()
  ├─ 获取 frontmatter + 内容
  ├─ 替换变量 (${CLAUDE_SKILL_DIR}, 参数)
  ├─ 执行内联 shell (非 MCP)
  ├─ 获取技能权限允许列表
  ├─ Fork 模式: 创建 forked context
  │   ├─ 新消息列表
  │   ├─ 新 token 预算
  │   └─ 新 agentId
  ├─ 调用 runAgent (fork) 或 query (inline)
  └─ 返回 ToolResult
```

### 内置技能 (Bundled Skills)

**文件**: `skills/bundledSkills.ts` (221行)

```typescript
type BundledSkillDefinition = {
  name: string
  description: string
  allowedTools?: string[]
  model?: string
  context?: 'inline' | 'fork'
  files?: Record<string, string>     // 参考文件
  getPromptForCommand: (args, context) => Promise<ContentBlockParam[]>
}

// 参考文件延迟提取到 ~/.claude/bundled-skills/{skillName}/
```

## Hooks 系统

### Hook 事件类型

```typescript
type HookEvent =
  | 'Setup'              // 会话开始前
  | 'SessionStart'       // 会话开始
  | 'SessionEnd'         // 会话结束
  | 'UserPromptSubmit'   // 用户提交
  | 'PreToolUse'         // 工具调用前
  | 'PostToolUse'        // 工具调用后
  | 'PostToolUseFailure' // 工具失败后
  | 'PermissionDenied'   // 权限拒绝
  | 'PreCompact'         // 压缩前
  | 'PostCompact'        // 压缩后
  | 'SubagentStart'      // 子代理启动
  | 'SubagentStop'       // 子代理停止
  | 'FileChanged'        // 文件变化
  | 'CwdChanged'         // 工作目录变化
  // ...更多
```

### Hook 类型

```json
{
  "hooks": {
    "PostToolUse": [
      { "type": "bash", "command": "prettier --write $FILE", "timeout": 10000 },
      { "type": "prompt", "content": "Validate the output..." },
      { "type": "http", "method": "POST", "url": "https://..." },
      { "type": "agent", "agentName": "validator" }
    ]
  }
}
```

### Hook 执行流程

```
Hook Event → 匹配配置 → 收集输入
    ↓
执行 Hook (bash/prompt/http/agent)
    ↓
收集输出 (JSON 解析)
    ↓
验证 Response Schema
    ↓
应用结果:
  - continue: bool
  - suppressOutput: bool
  - decision: approve|block
  - additionalContext: string
```

### Hook 环境变量

```bash
CLAUDE_EVENT_NAME="PostToolUse"
CLAUDE_EVENT_PAYLOAD='{"messages":[...], "context":{...}}'
CLAUDE_HOOK_ID="unique-hook-id"
CLAUDE_SESSION_ID="session-uuid"
CLAUDE_PLUGIN_OPTION_KEY="value"        # 插件 userConfig
```

## 安全考虑

- **Manifest Zod 验证**: 严格 schema check
- **路径遍历防护**: `../` 检查
- **官方市场名称保护**: homograph attack 防护
- **依赖图验证**: 循环检查
- **Hook 沙箱**: timeout、环境变量清理、子进程管理

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `plugins/builtinPlugins.ts` | 内置插件注册 |
| `utils/plugins/schemas.ts` (1450+行) | plugin.json 验证 |
| `utils/plugins/pluginLoader.ts` (3500+行) | 插件加载验证 |
| `utils/plugins/marketplaceManager.ts` (2800+行) | 市场管理 |
| `services/plugins/pluginOperations.ts` | 安装/卸载操作 |
| `skills/loadSkillsDir.ts` (1000+行) | 技能加载解析 |
| `skills/bundledSkills.ts` | 内置技能注册 |
| `tools/SkillTool/SkillTool.ts` | 技能执行 |
| `utils/hooks.ts` (4000+行) | Hook 执行引擎 |
| `types/hooks.ts` | Hook 类型定义 |
