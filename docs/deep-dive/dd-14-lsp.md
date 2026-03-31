# Deep Dive 14 - LSP 集成

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## 架构

LSPServerManager 管理多个 LSP 服务器实例，按文件扩展名路由请求。

## 服务器生命周期

stopped → starting → running → error（crash 后自动重启，最多 3 次）。Lazy init：首次请求时启动。

## 文件同步

didOpen / didChange / didSave / didClose 四个标准通知。全量文本更新模式。

## 诊断系统

- passiveFeedback.ts：异步收集 publishDiagnostics
- LSPDiagnosticRegistry：去重（LRU 缓存）、音量限制（每文件 10 个，总 30 个）、编辑时清除

## LSPTool（8 种操作）

goToDefinition / findReferences / hover / documentSymbol / workspaceSymbol / goToImplementation / prepareCallHierarchy / incomingCalls+outgoingCalls

## 插件集成

通过 plugin.json 的 lspServers 字段或 .lsp.json 文件声明。命名空间：`plugin:name:server`。

## 关键文件

| 文件 | 职责 |
|------|------|
| services/lsp/LSPServerManager.ts | 多服务器管理 |
| services/lsp/LSPServerInstance.ts | 单服务器生命周期 |
| services/lsp/LSPClient.ts | JSON-RPC 通信 |
| services/lsp/LSPDiagnosticRegistry.ts | 诊断去重 |
| services/lsp/passiveFeedback.ts | 诊断收集 |
| tools/LSPTool/LSPTool.ts | 8 种代码智能操作 |
