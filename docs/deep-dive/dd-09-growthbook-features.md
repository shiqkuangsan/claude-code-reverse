# Deep Dive 09 - GrowthBook 特性门控与遥测

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 使用**双层特性门控系统**：Bun 编译时 `feature()` 实现死代码消除，运行时 GrowthBook SDK 实现远程特性评估和 A/B 实验。遥测通过 OpenTelemetry 导出，采样策略由 GrowthBook 动态配置驱动。

## 编译时: feature()

```typescript
import { feature } from 'bun:bundle'

// 编译时评估 → false 分支被完全删除
export const AFK_MODE_BETA = feature('TRANSCRIPT_CLASSIFIER')
  ? 'afk-mode-2026-01-31'
  : ''
```

Bun 打包器在构建时评估 `feature()` 调用，条件为 false 的分支代码（包括导入）被完全消除，减少包体积和运行时开销。

### 已知编译时特性

`TRANSCRIPT_CLASSIFIER`, `COORDINATOR_MODE`, `KAIROS`, `FORK_SUBAGENT`, `CONNECTOR_TEXT`, `CHICAGO_MCP`, `WORKFLOW_SCRIPTS`, `MONITOR_TOOL`, `TOKEN_BUDGET`, `CONTEXT_COLLAPSE`, `BREAK_CACHE_COMMAND`, `WEB_BROWSER_TOOL`, `PROMPT_CACHE_BREAK_DETECTION` 等。

## 运行时: GrowthBook SDK

**文件**: `services/analytics/growthbook.ts` (~1155行)

### 初始化

```typescript
const initializeGrowthBook = memoize(async () => {
  // 单例，仅初始化一次
  // 配置: remoteEval: true, cacheKeyAttributes: ['id', 'organizationUUID']
  // 等待信任对话框完成后才发 HTTP 请求
  // 无认证时依赖磁盘缓存
})
```

### 用户属性

```typescript
type GrowthBookUserAttributes = {
  id: string                    // 设备 ID
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  organizationUUID?: string
  accountUUID?: string
  subscriptionType?: string
  rateLimitTier?: string
  appVersion?: string
  email?: string
}
```

### 特性值访问

| 函数 | 行为 | 推荐场景 |
|------|------|---------|
| `getFeatureValue_CACHED_MAY_BE_STALE<T>` | 立即返回（内存/磁盘缓存） | 启动关键路径 |
| `getDynamicConfig_CACHED_MAY_BE_STALE<T>` | 同上，语义化对象配置 | 批处理参数等 |
| `checkGate_CACHED_OR_BLOCKING` | 快速 true，慢速阻塞 | 用户触发特性 |
| `getFeatureValue_DEPRECATED<T>` | 阻塞等初始化 | **废弃** |

### 四级缓存

```
1. 环境变量覆盖 (CLAUDE_INTERNAL_FC_OVERRIDES) — eval 工具
2. 本地配置覆盖 (growthBookOverrides) — /config 界面（仅 ants）
3. 内存缓存 (remoteEvalFeatureValues) — 已初始化的值
4. 磁盘缓存 (~/.claude.json cachedGrowthBookFeatures) — 跨进程持久化
```

## 已知特性门控标志 (tengu_* 命名)

### 核心功能

| 标志 | 用途 |
|------|------|
| `tengu_session_memory` | 会话记忆系统 |
| `tengu_auto_background_agents` | 自动后台代理 |
| `tengu_amber_flint` | Agent Swarms 杀开关 |
| `tengu_herring_clock` | 团队内存 |
| `tengu_passport_quail` | 自动内存提取 |

### 远程与桥接

| 标志 | 用途 |
|------|------|
| `tengu_ccr_bridge` | CCR 远程集成 |
| `tengu_bridge_repl_v2` | REPL 桥接 v2 |
| `tengu_remote_backend` | 远程后端 |

### UI 与交互

| 标志 | 用途 |
|------|------|
| `tengu_terminal_panel` | 终端面板布局 |
| `tengu_desktop_upsell` | 桌面应用推广 |
| `tengu_destructive_command_warning` | 破坏性命令警告 |

### 性能与配置

| 标志 | 用途 |
|------|------|
| `tengu_event_sampling_config` | 事件采样率 |
| `tengu_1p_event_batch_config` | 日志批处理参数 |
| `tengu_max_version_config` | 自动更新控制 |
| `tengu_cicada_nap_ms` | 后台刷新节流 |

## 实验追踪

```typescript
// 曝光日志 — 同一会话每特性最多记录一次
function logExposureForFeature(feature: string) {
  if (loggedExposures.has(feature)) return
  const expData = experimentDataByFeature.get(feature)
  if (expData) {
    loggedExposures.add(feature)
    logGrowthBookExperimentTo1P({ experimentId, variationId, userAttributes })
  }
}
```

## 分析事件日志

### 队列驱动设计

启动期间事件排队到内存数组。Sink 附加后通过 `queueMicrotask()` 异步清空。

### 采样策略

```typescript
// 按事件名查阅 tengu_event_sampling_config
const sampleRate = getSampleRate(eventName)  // 0-1 或 null
if (Math.random() >= sampleRate) return      // 丢弃
metadata.sample_rate = sampleRate            // 注入采样率
```

### Sink 路由

- **Datadog**: 通用分析（可被 `tengu_frond_boric` 杀开关禁用）
- **First-Party**: 特权事件（`_PROTO_*` 字段提取到 BQ 列）

## 第一方事件导出器

**文件**: `services/analytics/firstPartyEventLoggingExporter.ts`

基于 OpenTelemetry `LogRecordExporter`：

- 端点: `https://api.anthropic.com/api/event_logging/batch`
- 批处理: 5s 或 200 事件（可通过 `tengu_1p_event_batch_config` 配置）
- 重试: 二次方退避 `delay = min(base × attempts², max)`
- 磁盘持久化: 失败事件写入 `~/.claude/telemetry/` (JSONL)
- 启动时重试上次运行的失败事件

## OpenTelemetry 初始化

```
延迟加载: SDK ~400KB, gRPC ~700KB
初始化等待信任对话框
导出器: OTLP (gRPC/HTTP) + BigQuery + Console (dev)
关闭: 2s 超时，并行关闭 meter/logger/tracer providers
```

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `services/analytics/growthbook.ts` (1155行) | GrowthBook 集成、特性评估、曝光日志 |
| `services/analytics/index.ts` | 事件队列、Sink 附加 |
| `services/analytics/sink.ts` | Datadog/1P 路由、采样 |
| `services/analytics/firstPartyEventLogger.ts` | 1P 事件 API、采样策略 |
| `services/analytics/firstPartyEventLoggingExporter.ts` | OTel 导出器、重试、磁盘持久化 |
| `services/analytics/metadata.ts` | 事件元数据丰富化 |
| `utils/telemetry/instrumentation.ts` | OTel 初始化、导出器配置 |
| `constants/betas.ts` | feature() 编译时评估 |
