# Deep Dive 20 - Prompt Suggestions

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## 三大子系统

1. **Prompt Suggestion** — 预测用户下一步输入（fork 生成，13 个过滤规则）
2. **Speculation** — 在 overlay 中提前执行建议（仅 ANT，Copy-on-Write 隔离）
3. **Tips** — 34+ 个上下文感知的静态提示（启蒙/功能发现）

## Prompt Suggestion

- 触发：Claude 完成响应后（stop hooks 阶段）
- 生成：runForkedAgent（禁用工具，复用缓存）
- 过滤：13 规则（too_few_words, evaluative, claude_voice, multiple_sentences 等）
- 格式：2-12 个单词，匹配用户风格

## Speculation（推测执行）

- Overlay 目录：~/.claude/temp/speculation/{pid}/{id}/
- Copy-on-Write：首次写入复制到 overlay
- 工具权限：Edit/Write（overlay 内）、Read/Glob/Grep（主线程）、Bash（仅只读）
- 边界检测：complete / bash / edit / denied_tool
- 接受：复制 overlay 回主线程，计算节省时间

## Tips 系统

34+ 个 tip，冷却期 3-15 个会话，按 sessionsLastShown 选择最久未显示的。支持自定义 tips。

## 其他补全

命令补全（/前缀，Fuse.js 模糊搜索）、目录补全（LRU 缓存）、Shell 历史（60s TTL）、Slack 频道（MCP 集成）。

## 关键文件

| 文件 | 职责 |
|------|------|
| services/PromptSuggestion/promptSuggestion.ts | 建议生成 |
| services/PromptSuggestion/speculation.ts | 推测执行 |
| services/tips/tipRegistry.ts | 34+ tip 定义 |
| services/tips/tipScheduler.ts | tip 选择 |
| utils/suggestions/ | 命令/目录/Shell 补全 |
