# Deep Dive 18 - Background Sessions / Daemon

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## 后台会话命令

| 命令 | 用途 |
|------|------|
| --bg | 创建后台会话 |
| ps | 列出后台会话 |
| logs [session] | 查看会话日志 |
| attach [session] | 附加到会话 |
| kill [session] | 终止会话 |

## Daemon 模式

--daemon 启动守护进程，--daemon-worker 启动工作节点。

## 会话持久化

后台会话持续运行，支持断开/重连。通过 entrypoints/cli.tsx 的快速路径处理。

## 关键文件

| 文件 | 职责 |
|------|------|
| utils/background/ | 后台会话管理 |
| entrypoints/cli.tsx | 快速路径分发 |
