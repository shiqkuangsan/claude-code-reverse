# Deep Dive 13 - Sandbox / Bash 安全解析

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## Bash 安全解析

23 项安全检查（parseForSecurity）：不完整命令、jq system()、混淆标志、壳元字符、命令替换 ($()/${}/``)、重定向、IFS 注入、/proc/self/environ、花括号展开、控制字符、Unicode 空格、Zsh 危险命令等。

## 只读命令检测

安全只读命令白名单：ls, cat, grep, find, head, tail, wc, stat, file, which, echo, printf 等。

## Zsh 危险命令

zmodload, emulate, sysopen/sysread/syswrite, zpty, ztcp, mapfile — 可绕过标准安全模型。

## SandboxManager

管理 Bash 命令沙箱化执行：网络白名单（从 WebFetch 规则提取）、文件系统 allowWrite/denyWrite、代理支持。

## 关键文件

| 文件 | 职责 |
|------|------|
| utils/bash/bashSecurity.ts | 23 项安全检查 |
| utils/sandbox/ | SandboxManager |
| utils/permissions/shouldUseSandbox.ts | 沙箱决策 |
| tools/BashTool/ | 权限检查集成 |
