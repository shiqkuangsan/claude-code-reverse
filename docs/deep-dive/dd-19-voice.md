# Deep Dive 19 - Voice 模式

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## 概览

Voice 模式允许通过语音与 Claude Code 交互。仅 claude.ai 订阅者可用（availability: ['claude-ai']）。

## 启用条件

GrowthBook 门控 (isVoiceGrowthBookEnabled) + 用户设置。

## 架构

voice/ 目录包含音频捕获和处理逻辑。vendor/audio-capture-src/ 提供原生音频捕获 addon。

## /voice 命令

切换语音模式开关。

## 关键文件

| 文件 | 职责 |
|------|------|
| voice/ | 语音模式核心 |
| commands/voice/ | /voice 命令 |
| vendor/audio-capture-src/ | 原生音频捕获 |
