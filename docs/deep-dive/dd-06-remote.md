# Deep Dive 06 - Remote Control / Bridge / Teleport

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 支持多种远程操作模式：Remote Sessions（CCR 环境运行）、Bridge（移动/Web 客户端连接）、Teleport（会话转移）、Daemon（后台会话）和 Direct Connect（cc:// URL）。这些机制共同构成了 CC 的分布式能力层。

## Remote Sessions

### RemoteSessionManager

**文件**: `remote/RemoteSessionManager.ts`

管理 WebSocket 长连接到 CCR（Claude Code Remote）环境：

```typescript
class RemoteSessionManager {
  sessionsWebSocket: SessionsWebSocket     // WebSocket 连接
  sdkMessageAdapter: SDKMessageAdapter     // 消息适配
  remotePermissionBridge: RemotePermissionBridge  // 权限桥接
}
```

### 消息流

```
本地 CC ←→ WebSocket ←→ CCR 环境中的 CC 实例
          ↕
    SDK 消息适配
    (sdkMessageAdapter.ts)
          ↕
    权限桥接
    (remotePermissionBridge.ts)
```

### 远程权限桥接

```typescript
// 本地权限决策
// 远程响应格式: allow (+ updatedInput) / deny (+ message)
```

## Bridge 协议

**文件**: `bridge/`

Bridge 是 IDE/移动/Web 客户端连接 CC 的中间层。

### 架构

```
IDE/移动客户端 ←→ Bridge ←→ CC REPL
                   ↕
            消息转换/路由
```

### 安全命令白名单

```typescript
BRIDGE_SAFE_COMMANDS   // 移动/Web 安全的命令子集
REMOTE_SAFE_COMMANDS   // 远程模式安全的命令子集
```

## Teleport

**文件**: `utils/teleport/`

Teleport 将本地会话上下文传送到远程 CCR 环境：

```typescript
// 1. 打包会话上下文（消息、设置、工具状态）
// 2. 传输到 CCR
// 3. 在远程环境恢复会话
// 4. 注册 RemoteAgentTask 用于轮询

const session = await teleportToRemote({
  initialMessage: prompt,
  description,
  signal: abortSignal,
})
```

### Teleport API

```typescript
// HTTP POST 发送消息
// 远程消息内容格式适配
```

## Upstream Proxy

**文件**: `upstreamproxy/`

CCR 容器中的 HTTPS 代理，处理子进程的网络访问：

### 功能

```
会话令牌: /run/ccr/session_token
CA 证书: 下载与合并
CONNECT → WebSocket 中继 (relay.ts)
环境变量注入:
  - HTTPS_PROXY
  - SSL_CERT_FILE
  - NO_PROXY
```

### 初始化

```typescript
if (process.env.CLAUDE_CODE_REMOTE) {
  initUpstreamProxy()  // 为子进程注入凭证
}
```

## Daemon 模式

### 后台会话管理

```
--bg              创建后台会话
ps                列出后台会话
logs [session]    查看会话日志
attach [session]  附加到后台会话
kill [session]    终止后台会话
--daemon          启动 daemon 进程
--daemon-worker   内部工作进程
```

### 快速路径

```typescript
// entrypoints/cli.tsx 中的快速路径检查
if (args.includes('--bg')) → 后台会话快速路径
if (args.includes('ps'))  → 列出会话快速路径
if (args.includes('logs')) → 日志快速路径
```

## Direct Connect

### cc:// URL 处理

```typescript
// 解析 cc:// 或 cc+unix:// URLs
// 直接连接到指定会话
const session = await createDirectConnectSession(url)
```

## SSH Remote

```typescript
// 两种模式:
createSSHSession(...)        // 远程 SSH 会话
createLocalSSHSession(...)   // 本地 SSH 会话
```

## Server 模块

**文件**: `server/`

HTTP/WebSocket 服务器，用于远程控制接口。

## CLI 传输层

**文件**: `cli/transports/`

不同的传输层实现，支持 stdio、WebSocket 等协议。

## REPL Bridge

### IDE 集成

```typescript
// replBridge 用于 VS Code/JetBrains 集成
// 消息在 IDE 和 CC REPL 之间传递
// 支持 remote-control 模式
```

### Remote Control

```
--remote-control [name]    启用 Remote Control
// 允许外部客户端连接并控制 CC 会话
```

## 远程 Agent 执行

```
AgentTool (isolation: 'remote')
  ├─ checkRemoteAgentEligibility()
  ├─ teleportToRemote(command, prompt)
  ├─ registerRemoteAgentTask()
  └─ 返回 { status: 'remote_launched', taskId, sessionUrl }

RemoteAgentTask
  ├─ 长连接 polling
  ├─ SDK 消息适配
  ├─ 进度追踪
  └─ 完成/失败通知
```

## 入口点分流

**文件**: `entrypoints/cli.tsx`

```
process.argv 解析:
  --chrome-native-host → Chrome MCP 服务器
  --daemon-worker → 内部工作进程
  remote-control → Remote Control 模式
  bridge → Bridge 模式
  daemon → Daemon 进程
  --bg/ps/logs/attach/kill → 后台会话管理
  cc:// → Direct Connect
  claude assistant → Assistant 模式
  claude ssh → SSH 模式
```

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `remote/RemoteSessionManager.ts` | 远程会话管理 |
| `remote/sdkMessageAdapter.ts` | SDK 消息适配 |
| `remote/remotePermissionBridge.ts` | 远程权限桥接 |
| `server/` | HTTP/WebSocket 服务器 |
| `upstreamproxy/relay.ts` | CONNECT → WebSocket 中继 |
| `bridge/` | Bridge 协议 |
| `utils/teleport/` | Teleport API |
| `cli/transports/` | 传输层 |
| `cli/handlers/` | CLI 处理器 |
| `entrypoints/cli.tsx` | 入口分流 |
| `tasks/RemoteAgentTask/` | 远程 Agent 任务 |
