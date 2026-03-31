# Deep Dive 21 - Output Styles

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## 概览

Output Styles 是注入系统提示词的对话模式定制机制。支持内置、自定义和插件来源。

## 内置风格

| 风格 | 特点 |
|------|------|
| default | 高效简洁 |
| Explanatory | 教育性解释（✨ Insight 标记） |
| Learning | 请求用户参与（TODO(human) 标记） |

## 自定义风格

Markdown + YAML frontmatter 格式：
```yaml
---
name: "Style Name"
description: "Description"
keep-coding-instructions: true
---
Prompt content...
```

目录：`.claude/output-styles/` 或 `~/.claude/output-styles/`

## 插件风格

通过 plugin.json 的 outputStyles 字段声明。命名：`pluginName:styleName`。
支持 `force-for-plugin: true` 自动应用。

## keepCodingInstructions

false = 删除标准编码指引，允许完全重定义。true = 保留。

## 优先级

内置 < 插件 < 管理设置 < 用户设置 < 项目设置。插件强制风格最高。

## 关键文件

| 文件 | 职责 |
|------|------|
| constants/outputStyles.ts | 类型定义、内置风格、加载逻辑 |
| outputStyles/loadOutputStylesDir.ts | 目录加载 |
| utils/plugins/loadPluginOutputStyles.ts | 插件加载 |
| constants/prompts.ts | 系统提示词注入 |
