# Deep Dive 16 - Vim 状态机

> 基于 @anthropic-ai/claude-code v2.1.88 逆向分析

## 架构

三层：VimState (INSERT/NORMAL) → CommandState (11 种子状态) → PersistentState (寄存器/查找/变更记录)。纯函数式状态机，不可变 Cursor 类。

## 子状态

idle / count / operator / operatorCount / operatorFind / operatorTextObj / find / g / operatorG / replace / indent

## 动作库

h/l/j/k（方向）、w/b/e/W/B/E（词）、0/^/$（行位置）、G/gg（全局）、gj/gk（显示行）

## 操作符

d(delete) / c(change) / y(yank) — 支持动作、文本对象、查找组合

## 文本对象

w/W（词）、"/'（引号）、()/[]/{}/<>（括号）— inner(i) / around(a)

## 点重复

RecordedChange 记录 11 种变更类型，`.` 命令重放。

## 关键文件

| 文件 | 职责 |
|------|------|
| vim/types.ts | 状态类型定义 |
| vim/transitions.ts | 状态转移函数 |
| vim/motions.ts | 动作库 |
| vim/operators.ts | 操作符执行 |
| vim/textObjects.ts | 文本对象查找 |
| hooks/useVimInput.ts | React 集成 |
| utils/Cursor.ts (1530行) | 不可变光标类 |
