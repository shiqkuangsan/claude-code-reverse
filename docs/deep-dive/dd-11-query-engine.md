# Deep Dive 11 - Query 流式引擎

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## 概览

query() 是 Claude Code 的核心对话循环，实现为异步生成器状态机。协调消息准备、API 流式调用、并行工具执行、错误恢复和自动继续。

## 状态机

每次迭代经过 5 个阶段：消息准备 → API 调用 → 工具执行 → 错误恢复 → 继续判定。

### 消息准备

按序执行 4 层压缩 + 系统提示组装 + 用户上下文前置。

### API 流式调用

`callModel()` → `queryModelWithStreaming()` 流式返回 assistant 消息和 tool_use 块。

### StreamingToolExecutor

并行工具执行器，并发安全工具可并行运行，非并发工具独占。Bash 错误级联取消兄弟工具。

### 错误恢复

| 错误 | 恢复策略 |
|------|---------|
| Prompt Too Long | Collapse Drain → Reactive Compact |
| Max Output Tokens | 升级到 64K → 3 次重试 |
| Media Size | stripImages 重试 |
| 429/529 | 指数退避，最多 3 次 |

### Withholding 模式

PTL/MaxOTK 错误从流式中扣留，直到恢复路径处理完毕。

### Stop Hooks

Claude 完成后执行用户定义钩子，可阻止继续或注入反馈。

### Token Budget

feature('TOKEN_BUDGET') 启用时：90% 消耗 → 继续带 nudge；递减 < 500 tokens × 3 次 → 停止。

### 多后端

Direct API / AWS Bedrock / Azure Foundry / Google Vertex AI，通过 `getAnthropicClient()` 工厂选择。

### 重试逻辑

指数退避，持久模式最大 5 分钟。429/529 前台查询重试 3 次，后台查询立即失败。

## 关键文件索引

| 文件 | 职责 |
|------|------|
| query.ts (1729行) | 主状态机 |
| query/config.ts | 运行时配置快照 |
| query/tokenBudget.ts | Token 预算 |
| query/stopHooks.ts | Stop Hook 执行 |
| services/api/claude.ts (3400+行) | API 调用 |
| services/api/client.ts | 多后端工厂 |
| services/api/withRetry.ts | 重试逻辑 |
| services/tools/StreamingToolExecutor.ts | 并行工具执行 |
