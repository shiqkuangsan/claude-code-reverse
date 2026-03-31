# Deep Dive 15 - Cost 计算与限额

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## 成本计算

calculateUSDCost() 基于模型定价：Sonnet $3/$15, Opus 4.5 $5/$25, Opus 4.6 fast $30/$150, Haiku $1/$5 per Mtok。Prompt cache 读取 10% 折扣。Web 搜索单独计费。

## 使用量追踪

ModelUsage: inputTokens, outputTokens, cacheRead, cacheCreation, webSearchRequests, costUSD。按模型聚合，会话结束时持久化。

## 速率限制

| 类型 | 窗口 |
|------|------|
| five_hour | 5 小时 |
| seven_day | 7 天 |
| seven_day_opus | Opus 专属 |
| seven_day_sonnet | Sonnet 专属 |
| overage | 额外用量 |

早期警告：多级阈值（25%@15%, 50%@35%, 75%@60%, 90%@72%）。

## 关键文件

| 文件 | 职责 |
|------|------|
| cost-tracker.ts | 成本聚合、持久化 |
| utils/modelCost.ts | 定价、USD 计算 |
| services/claudeAiLimits.ts | 速率限制 |
| services/rateLimitMessages.ts | 限制消息 |
