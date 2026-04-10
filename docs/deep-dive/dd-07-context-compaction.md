# Deep Dive 07 - Context Compaction（四层压缩策略）

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 概览

Claude Code 实现了精妙的**四层令牌管理系统**，在 `query.ts:365-468` 的查询循环中按序执行。从最轻量到最重量，逐步削减上下文大小，确保对话不超出模型的上下文窗口。

> **设计哲学**：当绝大部分 AI 衍生客户端用 `messages.splice(0, length-10)` 这种粗暴野蛮、容易导致语境断裂的切片方法防爆时，Claude 团队构建的是一张**金字塔式的三级防爆与阻断折叠体系**——优先切掉旁枝末角（microCompact），再尝试复用廉价的备胎成果（sessionMemoryCompact），最后才动用昂贵的终极重构（Legacy Compact）。这才是「Agent 耐力赛引擎」应当具备的素质。

## 四层执行顺序

```
Layer 1: toolResultBudget    → 大工具结果持久化到磁盘
Layer 2: snipCompact          → 按重要性裁剪旧消息
Layer 3: microCompact         → 缓存编辑/时间基线清除工具结果
Layer 3': contextCollapse     → 增量折叠（替代 autoCompact）
Layer 4: autoCompact          → 全上下文 API 总结压缩
```

## Layer 1: Tool Result Budget

**触发**: 每轮查询前自动执行

每个工具定义 `maxResultSizeChars`（如 BashTool=30,000）。单个用户消息内所有 tool_result 累计不超过 `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200,000` 字符。

超预算时：
1. 最大的结果写入 `projectDir/sessionId/tool-results/{toolUseId}.txt`
2. 模型收到 2000 字节预览 + 文件路径
3. 模型可用 FileRead 重新获取完整内容

## Layer 2: Snip Compact

**触发**: `feature('HISTORY_SNIP')` 启用时

按消息重要性启发式移除旧消息：
- 保护"尾部"（最后 5-10 条消息）
- 移除较早的完整 user/assistant 回合
- 返回 `tokensFreed` 传递给 autoCompact 阈值检查

## Layer 3: Micro Compact

**触发**: 每轮查询前自动执行

三个子路径：

### 3a. 缓存微压缩（Cached Microcompact / API 缓存无损编辑）

这是极其先锋的后端缓存编辑机制。使用 Claude API 的 `cache_edits` 功能，**通知 API 服务器在不破坏此前庞大的 Prompt 前缀缓存连贯性的前提下，只从树上摘除几个旧的 Tool Result 分支**：

- 当工具数量超过 `triggerThreshold` 时触发
- 保留最近 `keepRecent`（5-10）个工具
- 通过 API 层 `cache_edits` 块提交，返回 `cache_deleted_input_tokens`
- 不修改本地消息数组

**这种方法不仅省钱还能维持几百毫秒的首屏极速响应**——是 Claude Code 超越普通 Agent 客户端的关键技术之一。

### 3b. 时间基线微压缩（Time-Based）

当距离最后一个助手消息超过阈值时间（默认 60 分钟）时触发：
- 服务器端缓存已过期，完整前缀将被重写
- Content-clear 旧工具结果为占位符
- 保留最后 N 个工具的完整内容

## Layer 3': Context Collapse

**触发**: `feature('CONTEXT_COLLAPSE')` 启用时（替代 autoCompact）

- 不修改消息数组，维护分离的折叠存储
- 90% 阈值提交折叠，95% 阈值阻止请求
- 保留细粒度上下文（vs 单一摘要）
- 与 autoCompact 互斥（启用 collapse 时禁用 autoCompact）

## Layer 4: Auto Compact

**触发**: token 超过 `有效窗口 - AUTOCOMPACT_BUFFER_TOKENS(13,000)`

### 阈值计算

```typescript
// 关键常量
const AUTOCOMPACT_BUFFER_TOKENS = 13_000

autoCompactThreshold = getEffectiveContextWindowSize(model) - AUTOCOMPACT_BUFFER_TOKENS
```

**为什么是 13,000？** 这是 Anthropic 团队反复调优的「安全缓冲」——既要给后续工具调用和模型回答留足空间，又不能让用户提前感受到压缩延迟。当主对话的 Token 量距离阈值仅剩不到 13K 时，系统就会动作。

### 二级防线优先：Session Memory 偷换术

传统对话客户端在这时候会卡转圈——它必须去另外开一个沉重的模型调用，把前文完整吃下去生成汇总。但 Claude Code 有「影子工会」一直在后台偷偷写 `SESSION_MEMORY.md` 短篇总结（甚至帮我们做好了章节梳理），所以本模块选择**「摘果子」**：

1. 直接读取硬盘里那份不到 2000 个 Token 的旧总结文件
2. 调用 `calculateMessagesToKeepIndex` 函数拉网，计算出哪些对话是后台还没来得及写的「新鲜肉」。**为了保证不产生幻觉崩溃，它会仔细对齐结构，绝不会把模型的「思考锁 → 发工具 → 获结果」三连招拦腰劈断**
3. 把前半截废老对话直接暴力删除，在界面原位凭空注射一份带有摘要文件的「系统级紧凑幽灵节点」，随后拼接后半截的聊天

**0 额外发请求，0 等待延迟**！这是 sessionMemoryCompact 与 Legacy Compact 的本质差异。

### 警告阈值体系

| 阈值 | 缓冲 | 用途 |
|------|------|------|
| Warning | 窗口 - 20K | UI 黄色警告 |
| Error | 窗口 - 20K | UI 红色警告 |
| AutoCompact | 窗口 - 13K | 自动触发压缩 |
| Blocking | 窗口 - 3K | 阻止 API 请求 |

### 跳过条件

- querySource 是 `session_memory` 或 `compact`（递归守卫）
- `DISABLE_COMPACT` 或 `DISABLE_AUTO_COMPACT` 环境变量
- REACTIVE_COMPACT 模式启用时（仅反应式，等待 PTL 错误）
- CONTEXT_COLLAPSE 模式启用时

### 断路器

连续失败达到 `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` 次后停止尝试，避免浪费 API 调用。

### 压缩执行管道

```
1. 尝试 SessionMemory 压缩（快速，无 API 调用）
2. 失败 → compactConversation（API 总结）
   ├─ 剥离图像（保留 [image] 标记）
   ├─ 剥离可重注入附件
   ├─ 调用总结 API（forked agent + 流式备选）
   ├─ PTL 重试（最多 3 次，每次删除 20% 最早消息组）
   ├─ 生成附件
   │   ├─ 文件恢复（最多 5 个，50K token 预算）
   │   ├─ 技能恢复（最多 5 个，25K token 预算）
   │   ├─ 工具增量
   │   └─ MCP 说明增量
   └─ 执行 post_compact 钩子
```

### 压缩后清理

清除：microcompact 状态、context collapse 缓存、getUserContext 缓存、分类器审批、Bash 权限缓存、测试追踪。

**不清除**：已调用技能列表（需在多次压缩间存活）。

## Token 计数策略

| 函数 | 用途 | 方法 |
|------|------|------|
| `tokenCountWithEstimation` | 阈值检查（规范） | 最后 API usage + 新消息估计 |
| `tokenCountFromLastAPIResponse` | API 调用总量 | input + cache + output |
| `roughTokenCountEstimation` | 快速估计 | 字符数 × 1/4 × 4/3 |

## 反应式压缩

API 返回 `prompt_too_long` 错误时触发，比 autoCompact 更激进的截断策略。与 autoCompact 互斥（feature gate 控制）。

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `query.ts:365-468` | 四层协调 |
| `services/compact/autoCompact.ts` | 阈值、决策、断路器 |
| `services/compact/compact.ts` (1706行) | 主压缩 API、附件、PTL 重试 |
| `services/compact/microCompact.ts` | 缓存/时间基线微压缩 |
| `services/compact/postCompactCleanup.ts` | 压缩后缓存清理 |
| `utils/toolResultStorage.ts` | 工具结果持久化 |
| `services/compact/prompt.ts` | 压缩提示模板 |
| `services/contextCollapse/` | 折叠系统 |
