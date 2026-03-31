# Deep Dive 10 - Settings 多源合并系统

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## 概览

Claude Code 采用 **7 层设置源体系**，通过深度合并、Zod 验证、三层缓存和热重载机制，支持从个人偏好到企业策略的完整配置管理。

## 7 层设置源（低→高优先级）

| # | 源 | 路径 | 可编辑 |
|---|------|------|--------|
| 1 | pluginSettings | 插件基础层 | 否 |
| 2 | userSettings | ~/.claude/settings.json | 是 |
| 3 | projectSettings | .claude/settings.json | 是 |
| 4 | localSettings | .claude/settings.local.json | 是 |
| 5 | flagSettings | --settings CLI 标志 | 否 |
| 6 | policySettings | MDM / 远程托管 / managed-settings.json | 否 |

policySettings 内部采用"第一源优先"策略：Remote API > MDM > managed-settings.json > drop-in > HKCU。

## 合并策略

- **Objects**: 递归深度合并（lodash mergeWith）
- **Arrays**: 连接 + 去重（uniq）
- **undefined**: 作为删除标记
- **后来者优先**: 高优先级源覆盖低优先级

## 三层缓存

| 层 | 粒度 | 用途 |
|----|------|------|
| 会话级 | 合并结果 | 最快查询 |
| 源级 | 单个源 | 避免重复合并 |
| 文件级 | 单个文件 | 避免重复 I/O + Zod 验证 |

`resetSettingsCache()` 一次清除所有三层。

## MDM 集成

| 平台 | 源 | 路径 |
|------|-----|------|
| macOS | Admin | /Library/Managed Preferences/com.anthropic.claudecode.plist |
| Windows | Admin | HKLM\SOFTWARE\Policies\ClaudeCode |
| Windows | User | HKCU\SOFTWARE\Policies\ClaudeCode |
| Linux | File | /etc/claude-code/managed-settings.json |

轮询频率：30 分钟。

## 远程托管设置

- 端点: `{BASE_API_URL}/api/claude_code/settings`
- 认证: API Key 或 OAuth Token
- ETag 缓存 (304 Not Modified)
- 后台轮询: 1 小时
- 故障转移: fetch 失败 → 磁盘缓存 → 打开失败

## 热重载

基于 chokidar 文件监视 + MDM 轮询：
1. 检测文件变更（1s 稳定性阈值）
2. 消费内部写入标记（5s 窗口）
3. 执行 ConfigChange 钩子
4. 重置缓存 + 通知监听器
5. `applySettingsChange()` 更新 AppState

## 环境变量两阶段应用

**阶段 1**（信任前）: 仅受信源 (user/flag/policy) + 仅安全变量
**阶段 2**（信任后）: 所有源 + 所有变量

## 关键文件索引

| 文件 | 职责 |
|------|------|
| utils/settings/settings.ts | 加载、合并、持久化 |
| utils/settings/types.ts (800+行) | Zod Schema |
| utils/settings/validation.ts | 验证、错误格式化 |
| utils/settings/settingsCache.ts | 三层缓存 |
| utils/settings/changeDetector.ts | 热重载 |
| utils/settings/applySettingsChange.ts | AppState 同步 |
| utils/settings/mdm/ | MDM 集成 |
| services/remoteManagedSettings/ | 远程托管 |
