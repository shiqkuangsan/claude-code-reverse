# Deep Dive 17 - Worktree 隔离

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## 两种 Worktree 用途

1. **会话 Worktree** (--worktree CLI 标志)：为整个 CC 会话创建隔离分支
2. **Agent Worktree** (isolation: 'worktree')：为子代理创建临时隔离分支

## 会话 Worktree

setup.ts 中创建：`createWorktreeForSession(sessionId, slug)`。支持 --tmux 组合。创建后 process.chdir 到 worktree 路径。

## Agent Worktree

AgentTool 中创建：`createAgentWorktree(slug)`。返回 {worktreePath, branch, headCommit}。完成后检查变更：有则保留供 review，无则自动清理。

## Worktree Hooks

WorktreeCreate / WorktreeRemove 事件在创建/删除时触发。

## EnterWorktree / ExitWorktree 工具

用户可在会话中动态进入/退出 worktree。

## 关键文件

| 文件 | 职责 |
|------|------|
| utils/git/worktree.ts | 创建/销毁/检查变更 |
| tools/EnterWorktreeTool/ | 进入 worktree |
| tools/ExitWorktreeTool/ | 退出 worktree |
| setup.ts | 会话 worktree 初始化 |
