# 09 - MCP 协议集成

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 深度集成了 Model Context Protocol (MCP)，支持通过 MCP 服务器扩展工具、资源和提示能力。MCP 工具与内置工具统一管理，通过工具池组装实现无缝集成。

## MCP 客户端架构

**文件**: `services/mcp/client.ts` (3500+ 行)

```typescript
class MCPClient {
  connectServer(config: McpServerConfig): Promise<ConnectedMCPServer>
  listTools(): Promise<Tool[]>
  callTool(name: string, args): Promise<MCPToolResult>
  listPrompts(): Promise<Prompt[]>
  listResources(): Promise<ServerResource[]>
  readResource(uri: string): Promise<Buffer | string>
  handleOAuth(config): Promise<AuthResult>
}
```

## MCP 服务器配置

**文件**: `services/mcp/config.ts` (1500+ 行)

### 传输类型

```typescript
type McpServerConfig =
  | { type: 'stdio', command: string, args?: string[], env?: Record<string, string> }
  | { type: 'sse', url: string, headers?: Record<string, string> }
  | { type: 'http', url: string, auth?: { type: 'bearer' | 'oauth' | 'basic', ... } }
  | { type: 'websocket', url: string, auth?: { ... } }
```

### 配置来源

1. **项目配置**: `.mcp.json` 文件
2. **CLI 标志**: `--mcp-config <file-or-json>`
3. **插件**: `plugin.json` 中的 `mcpServers` 字段
4. **企业配置**: 企业级 MCP 策略
5. **Claude.ai**: Claude in Chrome 集成

### 配置加载时序

```typescript
// main.tsx 中的加载顺序
const dynamicMcpConfig = parseMcpConfigs(mcpConfigFlags)

if (enableClaudeInChrome) {
  dynamicMcpConfig = { ...dynamicMcpConfig, ...setupClaudeInChrome().mcpConfig }
}

// 企业配置检查
if (doesEnterpriseMcpConfigExist()) {
  if (strictMcpConfig) process.exit(1)
  if (!areMcpConfigsAllowedWithEnterpriseMcpConfig(dynamicMcpConfig)) process.exit(1)
}

// Headless 模式连接
await connectMcpBatch(regularMcpConfigs, 'regular')
const claudeaiTimedOut = await Promise.race([
  connectMcpBatch(claudeaiConfigs, 'claudeai'),
  timeout(CLAUDE_AI_MCP_TIMEOUT_MS)
])
```

## 工具映射

**文件**: `tools/MCPTool/MCPTool.ts` (78行)

MCP 工具通过统一的代理工具接入：

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',                           // 动态覆盖
  maxResultSizeChars: 100_000,
  async call(args): Promise<{ data: string }>,
})

// MCP 工具名称构造
// mcp__serverName__toolName
// 示例: mcp__github__search_repositories
```

### 工具池组装

```typescript
export function assembleToolPool(permissionContext, mcpTools) {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  // 内置工具优先，按名称去重
  return uniqBy([...builtInTools, ...allowedMcpTools], 'name')
}
```

## 提示映射

MCP 服务器公开的 prompts 映射到 Claude Code 的技能系统：

```typescript
// MCP prompt → Claude Code Command
Command {
  type: 'prompt',
  name: 'prompt-name',
  source: 'mcp',
  loadedFrom: 'mcp',
  getPromptForCommand: async (args) => mcpPromptContent
}
```

## 资源工具

| 工具 | 用途 |
|------|------|
| `ListMcpResources` | 列出 MCP 服务器的资源 URI |
| `ReadMcpResource` | 读取单个资源内容 |
| `McpAuth` | MCP 服务器认证（OAuth 等） |

## MCP 认证

**文件**: `tools/McpAuthTool/`

支持多种认证方式：
- Bearer Token
- OAuth 2.0 (包含 PKCE 流程)
- Basic Auth

## 与插件系统的集成

插件可通过 `plugin.json` 声明 MCP 服务器：

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server.js"],
      "env": {
        "API_KEY": "${user_config.API_KEY}"
      }
    }
  }
}
```

变量替换：
- `${CLAUDE_PLUGIN_ROOT}` — 插件根目录
- `${user_config.KEY}` — 用户配置值

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `services/mcp/client.ts` (3500+行) | MCP 客户端核心 |
| `services/mcp/config.ts` (1500+行) | MCP 配置解析 |
| `tools/MCPTool/MCPTool.ts` | MCP 工具代理 |
| `tools/ListMcpResourcesTool/` | 资源列表 |
| `tools/ReadMcpResourceTool/` | 资源读取 |
| `tools/McpAuthTool/` | MCP 认证 |
| `utils/mcp/` | MCP 工具函数 |
