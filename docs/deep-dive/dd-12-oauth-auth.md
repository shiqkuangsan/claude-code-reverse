# Deep Dive 12 - OAuth / Auth 认证体系

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## 认证方式优先级

1. **Claude.ai OAuth** — PKCE 流程，主要方式
2. **环境变量 OAuth** — `CLAUDE_CODE_OAUTH_TOKEN`（CCR/Desktop）
3. **FD OAuth** — `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`（CCR 子进程）
4. **API Key Helper** — 用户定义脚本（5 分钟缓存）
5. **直接 API Key** — `ANTHROPIC_API_KEY`
6. **Auth Token** — `ANTHROPIC_AUTH_TOKEN`

## OAuth PKCE 流程

1. 生成 Code Verifier (32 random bytes base64url) + State (CSRF)
2. 启动 localhost HTTP 监听器
3. 打开浏览器授权 URL
4. 用户授权 → 回调带 auth code
5. exchangeCodeForTokens() → access + refresh token
6. 保存到 Keychain / .credentials.json

## Token 存储

- macOS: Keychain ("Claude Code-credentials")
- Linux: ~/.claude/.credentials.json
- 安全措施: `security -i` 隐藏进程参数

## Token 刷新

- 自动检测过期（5 分钟缓冲）
- 多进程文件锁同步
- 最多 5 次重试
- 跨进程 .credentials.json mtime 监控

## OAuth 作用域

| 作用域 | 用途 |
|--------|------|
| user:inference | 推理访问 |
| user:profile | 用户信息 |
| user:sessions:claude_code | 会话管理 |
| user:mcp_servers | MCP 访问 |
| user:file_upload | 文件上传 |

## Keychain 预取

启动时并行执行 OAuth + API Key 两个 keychain 读取，与模块导入重叠（~65ms）。

## 关键文件索引

| 文件 | 职责 |
|------|------|
| services/oauth/client.ts | OAuth 客户端 |
| services/oauth/auth-code-listener.ts | 本地回调监听 |
| utils/auth.ts (2000+行) | 认证主逻辑 |
| utils/secureStorage/macOsKeychainStorage.ts | Keychain 存储 |
| utils/secureStorage/keychainPrefetch.ts | 预取优化 |
| constants/oauth.ts | 端点和作用域 |
| commands/login/ | /login 命令 |
| commands/logout/ | /logout 命令 |
