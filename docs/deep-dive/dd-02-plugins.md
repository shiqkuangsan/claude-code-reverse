# Deep Dive 02 - 插件系统全解

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 的插件系统支持完整的生态：发现 → 安装 → 验证 → 加载 → 执行 → 卸载。插件可包含 commands、agents、skills、hooks、MCP 服务器、LSP 服务器和输出样式。代码量约 10,000+ 行。

## plugin.json 清单格式

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "插件描述",
  "author": { "name": "Author", "email": "...", "url": "..." },
  "homepage": "https://...",
  "repository": "https://...",
  "license": "MIT",
  "keywords": ["tag1", "tag2"],

  "commands": "./commands/",
  "agents": "./agents/",
  "skills": "./skills/",
  "outputStyles": "./output-styles/",

  "hooks": { "PostToolUse": [{ "type": "command", "command": "..." }] },
  "mcpServers": { "server-name": { "type": "stdio", "command": "node", "args": ["server.js"] } },
  "lspServers": { "ls-name": { "command": "...", "extensionToLanguage": { ".ts": "typescript" } } },

  "userConfig": {
    "API_KEY": {
      "type": "string",
      "title": "API Key",
      "description": "Your API key",
      "required": true,
      "sensitive": true
    }
  },

  "channels": [{ "server": "telegram", "displayName": "Telegram" }],
  "dependencies": ["other-plugin"],
  "settings": { "agent": { ... } }
}
```

### userConfig 类型

| type | 说明 | 特殊字段 |
|------|------|---------|
| `string` | 文本输入 | `multiple`: 允许数组 |
| `number` | 数值输入 | `min`, `max` |
| `boolean` | 布尔开关 | |
| `directory` | 目录选择 | |
| `file` | 文件选择 | |

`sensitive: true` → 存储到 Keychain/credentials，不进 settings.json。

## 插件来源层次

```
Built-in Plugins (@builtin)         编译时内置
    ↓
Managed Plugins (policySettings)     企业策略推送
    ↓
User-Scope (~/.claude/settings.json) 全局安装
    ↓
Project-Scope (.claude/settings.json) 项目级
    ↓
Local-Scope (.claude/.claude/...)     项目本地
    ↓
Session/CLI (--plugin-dir)            会话临时
```

## Marketplace 来源类型

| 来源 | 格式 | 示例 |
|------|------|------|
| `github` | `owner/repo` | `anthropics/claude-plugins` |
| `npm` | 包名 | `@org/my-plugin` |
| `url` | HTTP(S) URL | `https://example.com/marketplace.json` |
| `git` | Git URL | `git@github.com:org/repo.git` |
| `file` | 本地文件 | `./my-marketplace.json` |
| `directory` | 本地目录 | `./plugins/` |
| `hostPattern` | 正则主机名 | `^github\\.mycompany\\.com$` |

## 安装流程 (V2)

### installed_plugins.json 格式

```json
{
  "version": 2,
  "plugins": {
    "plugin-name@marketplace": [{
      "scope": "user",
      "installPath": "~/.claude/plugins/cache/marketplace/plugin-name/1.0.0/",
      "version": "1.0.0",
      "installedAt": "2024-01-15T10:30:00.000Z",
      "gitCommitSha": "abc123..."
    }]
  }
}
```

### 安装步骤

```
1. 解析插件标识符 → name + marketplace
2. 搜索 materialized marketplace 中的插件
3. 写入 enabledPlugins: true 到 settings (intent 声明)
4. 缓存插件到 ~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/
5. 记录到 installed_plugins.json
```

## 组件加载

### Commands/Skills 加载

- `*.md` 文件 → `/plugin:command-name`
- `skill.md` 在子目录中 → 技能
- 命名空间用 `:` 分隔

### 变量替换

```
${CLAUDE_PLUGIN_ROOT}    → 插件版本化安装目录
${CLAUDE_PLUGIN_DATA}    → 持久数据目录（跨版本存活）
${user_config.KEY}       → 用户配置值
```

Sensitive 值在 skill/agent 内容中 → placeholder: `[sensitive option 'KEY' not available]`

### Hooks 加载

- 从 enabled 插件收集 hooksConfig
- 注册到全局 hooks 注册表
- 支持热重载：监听设置变化，自动 clear + reload

### MCP 服务器加载

- 支持 inline config、.mcp.json 文件、.mcpb bundle
- 重复检测：同名/同 URL 的服务器被 suppressed
- 用户配置变量替换

## 安全验证

### 名称仿冒防护

```typescript
BLOCKED_OFFICIAL_NAME_PATTERN  // 检测 "official-anthropic" 等变体
validateOfficialNameSource()   // 保留名称仅来自 anthropics/ GitHub org
```

### 路径遍历防护

检查 `..` 段在相对路径中，防止逃逸插件目录。

### Non-ASCII 检测

防止同形字攻击（Cyrillic 'а' vs Latin 'a'）。

## 依赖解析

### 安装时：DFS 闭包

```typescript
resolveDependencyClosure(rootId, lookup, alreadyEnabled)
// DFS 遍历，检测循环，跨 marketplace 默认阻止
```

### 加载时：验证降级

```typescript
verifyAndDemote(plugins)
// Fixed-point loop：不满足依赖的插件降级为 disabled
// 传递性失败：A→B→C，C 缺失则 A 和 B 都降级
```

## 启用/禁用机制

### enabledPlugins 格式

```json
{
  "enabledPlugins": {
    "plugin@marketplace": true,    // 启用
    "plugin@marketplace": false,   // 显式禁用
    "plugin@marketplace": ["^1.0"] // 版本约束（= 启用）
  }
}
```

### 内置插件启用

```
用户设置 (enabledPlugins[pluginId])
    ↓ (undefined 时)
插件定义 defaultEnabled
    ↓ (undefined 时)
true
```

## /plugin 命令 UI

| Tab | 功能 |
|-----|------|
| Manage | 启用/禁用、卸载、配置选项 |
| Discover | 浏览 marketplace、安装 |
| Marketplace Management | 添加/删除/刷新 marketplace |
| Validate | 验证 plugin.json / marketplace.json |

## 企业策略

```json
{
  "pluginPolicy": {
    "blockedMarketplaces": [{ "source": "...", "reason": "..." }],
    "strictKnownMarketplaces": [{ "source": "github", "repo": "..." }],
    "blockLocalPlugins": true
  }
}
```

## 缓存管理

```
~/.claude/plugins/
├── known_marketplaces.json
├── installed_plugins.json
├── cache/{marketplace}/{plugin}/{version}/
├── marketplaces/
└── .plugin_session/{random-id}/
```

## 关键文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `utils/plugins/schemas.ts` | 1750+ | Zod 验证 |
| `utils/plugins/pluginLoader.ts` | 3500+ | 发现、缓存、加载 |
| `utils/plugins/marketplaceManager.ts` | 2800+ | Marketplace 管理 |
| `utils/plugins/installedPluginsManager.ts` | 1200+ | V2 安装跟踪 |
| `services/plugins/pluginOperations.ts` | 700+ | 安装/卸载操作 |
| `plugins/builtinPlugins.ts` | 160 | 内置插件注册 |
| `commands/plugin/` | 490KB | 完整 UI |
