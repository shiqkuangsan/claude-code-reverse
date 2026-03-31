# 10 - 服务层与基础设施

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 的服务层涵盖 API 通信、认证、分析、成本追踪、设置管理、LSP 集成和远程会话等核心基础设施。

## API 服务

**文件**: `services/api/client.ts`, `services/api/claude.ts`

### 多后端支持

```typescript
// 通过环境变量动态选择后端
| 环境变量 | 后端 |
|---------|------|
| CLAUDE_CODE_USE_BEDROCK | AWS Bedrock |
| CLAUDE_CODE_USE_FOUNDRY | Azure Foundry |
| CLAUDE_CODE_USE_VERTEX | Google Vertex AI |
| (默认) | Anthropic Direct API |
```

### 请求配置

```typescript
// 默认请求头
'x-app': 'cli'
'X-Claude-Code-Session-Id': sessionId
'x-client-app': sdkConsumerIdentifier
'x-client-request-id': uuid()          // 超时日志关联

// 超时
API_TIMEOUT_MS = 600_000               // 10 分钟
```

### 流式调用

`claude.ts` 处理消息流、增量事件、使用量跟踪。支持多种 beta 特性：prompt caching、context management、fast mode、thinking。

### 错误处理与重试

**文件**: `services/api/errors.ts`, `services/api/withRetry.ts`

- Prompt 太长：解析 token 限制 (`parsePromptTooLongTokenCounts`)
- 配额提取：`extractQuotaStatusFromError()`, `extractQuotaStatusFromHeaders()`
- 可配置重试 + 指数退避

## 认证系统

**文件**: `services/oauth/`

### 认证方式

| 方式 | 来源 |
|------|------|
| API Key | `ANTHROPIC_API_KEY` 环境变量或密钥助手 |
| OAuth Token | Claude.ai 订阅（PKCE 流程） |
| 令牌刷新 | `refreshOAuthToken()` 自动续期 |

### OAuth 流程

```typescript
// 1. 构建授权 URL (PKCE)
const authUrl = buildAuthorizationUrl({ codeChallenge, scope, loginHint })

// 2. 用户授权后交换令牌
const tokens = await exchangeCodeForTokens(authCode, codeVerifier)

// 3. 令牌持久化到 ~/.claude/auth.json

// 4. 定期刷新
await checkAndRefreshOAuthTokenIfNeeded()
```

## 成本追踪

**文件**: `cost-tracker.ts`, `costHook.ts`

### 使用量类型

```typescript
type ModelUsage = {
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens: number
  cacheCreationInputTokens: number
  webSearchRequests: number
  cost: number                          // USD
}
```

### 成本计算

- `calculateUSDCost()`: 基于模型价格表
- 提示词缓存折扣
- Web 搜索单独计费
- 会话结束时持久化到项目配置

## 分析与遥测

**文件**: `services/analytics/`

### 事件架构

队列驱动设计：事件排队直到 sink 附加。

### GrowthBook 特性门控

```typescript
// 远程评估特性值
getFeatureValue_CACHED_MAY_BE_STALE('feature_name')

// 用户属性
{
  deviceId, platform, orgUuid, subscriptionType,
  rateLimitTier, appVersion, ...
}
```

### 第一方事件记录

- OpenTelemetry 导出器实现
- 分批处理：`tengu_1p_event_batch_config`
- 端点：`https://api.anthropic.com/v1/events`
- OpenTelemetry SDK 延迟加载 (~400KB)

## 策略与限制

**文件**: `services/claudeAiLimits.ts`, `services/policyLimits/`

### 限制类型

| 窗口 | 说明 |
|------|------|
| five_hour | 5 小时会话限制 |
| seven_day | 7 天周限制 |
| seven_day_opus / seven_day_sonnet | 模型特定限制 |
| overage | 超额限制 |

### 早期警告

多级阈值：利用率 0.25/0.5/0.75 对应 15%/35%/60% 时间进度。

### 配额状态

```typescript
type QuotaStatus = 'allowed' | 'allowed_warning' | 'rejected'
```

## 设置系统

**文件**: `utils/settings/`

### 设置源优先级

```
cliArg > command > userSettings > projectSettings > localSettings
  > managedSettings > policySettings
```

### 设置文件

| 文件 | 范围 |
|------|------|
| `~/.claude/settings.json` | 用户全局 |
| `.claude/settings.json` | 项目级 |
| `.claude/settings.local.json` | 项目本地（gitignore） |
| 企业 MDM | macOS/Windows 策略 |

### 设置验证

Zod schema 验证 + 权限规则过滤 + 详细错误报告。

## 状态管理

**文件**: `state/AppState.tsx`, `state/AppStateStore.ts`

React Context + Zustand 式存储：

```typescript
// 不可变部分 (DeepImmutable<T>)
settings, model, uiState, permissionContext, remoteSession

// 可变部分
tasks: TaskStateMap
mcp: { clients, tools, commands, resources }
plugins: { loaded, errors }
agentNameRegistry: Map<name, agentId>
```

选择器优化：`useAppState(selector)` 仅当选中值变化时重新渲染。

## 对话历史

**文件**: `history.ts`

- 全局历史文件：`~/.claude/history.jsonl`
- 大粘贴 (>1KB) 存储在单独 paste store
- 反向读取优化：`readLinesReverse()` 从文件末尾开始
- 上限：100 条历史项目

## LSP 集成

**文件**: `services/lsp/LSPServerManager.ts`

- 按需启动 LSP 服务器
- 基于文件扩展名路由请求
- 文件同步：didOpen/didChange/didSave/didClose
- 诊断注册

## 远程会话

**文件**: `remote/`, `server/`, `upstreamproxy/`

### 远程会话管理

```typescript
class RemoteSessionManager {
  // WebSocket 连接
  sessionsWebSocket: SessionsWebSocket
  // 消息适配
  sdkMessageAdapter: SDKMessageAdapter
  // 权限桥接
  remotePermissionBridge: RemotePermissionBridge
}
```

### Upstream Proxy

CCR 容器中的 HTTPS 代理：
- 会话令牌：`/run/ccr/session_token`
- CA 证书下载与合并
- CONNECT → WebSocket 中继
- 环境变量注入：`HTTPS_PROXY`, `SSL_CERT_FILE`

## 数据迁移

**文件**: `migrations/`

版本化迁移系统（当前版本 11）：
- 幂等设计
- 仅针对 userSettings
- 模型重命名：Sonnet 4.5 → 4.6, Opus → Opus 1M
- 功能演变：权限、快速模式等

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `services/api/client.ts` | API 客户端工厂 |
| `services/api/claude.ts` | 流式消息处理 |
| `services/oauth/client.ts` | OAuth 流程 |
| `cost-tracker.ts` | 成本追踪 |
| `services/analytics/` | 分析与遥测 |
| `services/policyLimits/` | 策略限制 |
| `utils/settings/settings.ts` | 设置管理 |
| `state/AppStateStore.ts` | 全局状态存储 |
| `history.ts` | 对话历史 |
| `services/lsp/` | LSP 集成 |
| `remote/` | 远程会话 |
| `upstreamproxy/` | 上游代理 |
| `migrations/` | 数据迁移 |
