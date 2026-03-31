# 00 - Claude Code 总体架构鸟瞰图

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 项目概况

Claude Code 是 Anthropic 官方的 CLI 编程助手，运行在终端中。代码库约 1884 个 TypeScript/TSX 文件，使用 Bun 构建为单文件 `cli.js`（13MB），附带 60MB source map。

技术栈：**TypeScript + React + Ink（自定义终端 UI 框架）+ Yoga 布局引擎 + Commander.js CLI**

## 架构总览

```
┌─────────────────────────────────────────────────┐
│                    CLI 入口层                     │
│  entrypoints/cli.tsx → main.tsx → Commander.js   │
├─────────────────────────────────────────────────┤
│                  对话循环层                       │
│  query.ts (异步生成器) → API 流式调用 → 工具执行  │
├───────────────┬─────────────┬───────────────────┤
│   工具系统     │  Agent/Task │   命令系统         │
│  Tool.ts      │  Task.ts    │  commands.ts      │
│  tools/       │  tasks/     │  commands/        │
│  45+ 工具     │  7 种 Task  │  80+ 命令         │
├───────────────┴─────────────┴───────────────────┤
│                  上下文与扩展层                    │
│  context.ts    plugins/    skills/    MCP        │
│  CLAUDE.md     hooks       bundled   services/   │
├─────────────────────────────────────────────────┤
│                  服务基础设施                      │
│  API/Auth  Analytics  Settings  State  History   │
│  OAuth     GrowthBook  LSP     Cost   Migrations │
├─────────────────────────────────────────────────┤
│                  终端 UI 层                       │
│  ink/ (自定义 React Reconciler + Yoga 布局)       │
│  components/ (389 文件)  PromptInput  Vim 模式    │
└─────────────────────────────────────────────────┘
```

## 模块规模

| 模块 | 文件数 | 核心职责 |
|------|--------|---------|
| `utils/` | 564 | 工具函数（权限、设置、git、bash、内存等） |
| `components/` | 389 | React 终端 UI 组件 |
| `commands/` | 189 | 80+ 斜杠命令实现 |
| `tools/` | 184 | 45+ 工具实现 |
| `services/` | 130 | 服务层（API、分析、MCP、LSP 等） |
| `hooks/` | 104 | React hooks + 权限钩子 |
| `ink/` | 96 | 自定义 Ink 终端 UI 框架 |
| `cli/` | 19 | CLI 处理器和传输层 |
| `skills/` | 20 | 技能系统 |
| `tasks/` | 12 | 5 种 Task 类型实现 |
| `plugins/` | 2 | 插件系统核心 |
| 其他 | ~175 | context、state、types、vim 等 |

## 核心数据流

```
用户输入
    ↓
REPL 组件捕获 → 命令解析（/ 前缀）或消息提交
    ↓
query() 异步生成器
    ├─ 收集上下文 (CLAUDE.md + Git + 日期)
    ├─ 组装 system prompt
    ├─ 消息准备 (紧凑化/折叠/预算)
    ├─ API 流式调用 (callModel)
    │   └─ yield 流式事件 → UI 实时更新
    ├─ 解析 tool_use blocks
    ├─ 工具执行管线
    │   ├─ validateInput
    │   ├─ checkPermissions (规则/分类器/UI)
    │   ├─ PreToolUse hooks
    │   ├─ tool.call()
    │   └─ PostToolUse hooks
    ├─ yield 工具结果
    └─ 判断继续/结束
        ├─ end_turn → Terminal
        └─ tool_use → 继续循环
```

## 关键设计决策

### 1. 异步生成器驱动

对话循环使用 `async function* query()` 实现，逐个 yield 事件，支持流式响应和取消。

### 2. 多态 Task 系统

7 种 Task 类型通过统一接口管理，支持本地/远程/进程内多种执行模式。

### 3. 统一工具接口

`buildTool()` 工厂模式 + fail-closed 默认值。45+ 工具 + MCP 工具在同一池中管理。

### 4. 多层权限

规则 → 钩子 → 分类器 → UI 提示，四层递进保证安全。

### 5. 插件可扩展性

plugin.json 清单支持技能、命令、Agent、Hooks、MCP/LSP 服务器，完整的 marketplace 生态。

### 6. 自定义 Ink 终端 UI

基于 React Reconciler + Yoga 布局的终端 UI，支持双缓冲差分渲染（60fps）、鼠标事件、Vim 模式。

### 7. Prompt Cache 复用

Fork 子代理保持 byte-identical API 请求前缀，复用 parent 的 prompt cache 减少成本。

### 8. 多后端 API

同一代码支持 Anthropic Direct、AWS Bedrock、Azure Foundry、Google Vertex AI。

## 文档系列导航

| 编号 | 文档 | 主题 |
|------|------|------|
| 00 | [总体架构](00-architecture-overview.md) | 鸟瞰图（本文） |
| 01 | [启动流程](01-bootstrap-lifecycle.md) | 启动序列、10 个阶段、CLI 选项 |
| 02 | [对话循环](02-conversation-loop.md) | query() 异步生成器、消息流、上下文压缩 |
| 03 | [工具系统](03-tool-system.md) | Tool 接口、注册表、权限、Hooks |
| 04 | [Agent/Task](04-agent-task-system.md) | 7 种 Task、AgentTool、Team/Swarms |
| 05 | [命令系统](05-command-system.md) | 80+ 命令、注册表、三层过滤 |
| 06 | [终端 UI](06-terminal-ui.md) | Ink 框架、渲染管线、Vim 模式 |
| 07 | [上下文引擎](07-context-engine.md) | CLAUDE.md、上下文收集、压缩策略 |
| 08 | [插件/Skill](08-plugin-skill-system.md) | 插件生命周期、SKILL.md、Hooks |
| 09 | [MCP 集成](09-mcp-integration.md) | MCP 客户端、工具映射、资源 |
| 10 | [服务层](10-services-infrastructure.md) | API/Auth/Analytics/Settings/LSP |
| 11 | [权限安全](11-permission-security.md) | 权限模式、分类器、沙箱、信任 |

## 关键文件快速索引

| 文件 | 行数 | 核心角色 |
|------|------|---------|
| `main.tsx` | 4683 | 应用入口、Commander.js、生命周期 |
| `query.ts` | 2400+ | 对话循环（异步生成器） |
| `Tool.ts` | 793 | Tool 接口、buildTool、ToolUseContext |
| `tools.ts` | 390 | 工具注册表、工具池组装 |
| `commands.ts` | 755 | 命令注册表、加载逻辑 |
| `Task.ts` | 126 | Task 基类、状态机 |
| `context.ts` | 190 | 上下文收集（memoized） |
| `ink/ink.tsx` | 1722 | 自定义 Ink 终端渲染核心 |
| `interactiveHelpers.tsx` | 300+ | 设置屏幕、信任对话框 |
| `setup.ts` | 400+ | 会话级初始化 |
| `bootstrap/state.ts` | 400+ | 全局 Bootstrap 状态 |
