# Deep Dive 22 - Harness 运行时宿主全景

> 基于 `@anthropic-ai/claude-code` v2.1.88 逆向分析

## 什么是 Harness？

**Harness（运行时宿主工程）= 除了大模型本身之外的一切工程代码。**

它是把 Claude 大模型从「只会聊天的嘴」变成「有手有脚有记忆、能在真实文件系统中执行任务的 Agent」的完整脚手架。LLM 只负责思考和决策，Harness 负责：

- **感知环境** —— Git 状态、CLAUDE.md、当前时间、文件树
- **分发工具** —— 45+ 内置工具 + MCP 扩展工具的统一调度
- **管理记忆** —— 五层记忆体系（MEMORY.md、Session、Auto、Dream、Team）
- **控制预算** —— Token、美元、超时、并发的多重熔断
- **拦截危险操作** —— allow/deny/ask 三态权限 + yoloClassifier 风险评分

理解 Harness，才能理解为什么 Claude Code 是「Agent」而不是「Chatbot」。

---

## 架构全景图

```
┌──────────────────────────────────────────────────────────────────┐
│                      Harness（运行时宿主工程）                     │
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────────────┐  │
│  │  CLI 入口    │───▶│ QueryEngine │───▶│   query.ts 主循环    │  │
│  │  cli.tsx    │    │  底盘 / ECU │    │   发动机 / 状态机    │  │
│  │  Commander  │    │ 1300+ 行    │    │   1700+ 行            │  │
│  └─────────────┘    └──────┬──────┘    └──┬──────────┬─────────┘  │
│                            │              │          │            │
│                   ┌────────▼─────────┐    │          │            │
│                   │   context.ts     │    │          │            │
│                   │  环境 & 世界观注射 │    │          │            │
│                   │  ~190 行          │    │          │            │
│                   └──────────────────┘    │          │            │
│                                           │          │            │
│         ┌─────────────────────────────────▼──┐  ┌────▼─────────┐  │
│         │       Tool 系统 (Tool.ts)           │  │ compact/     │  │
│         │  工具插槽 + 沙盒 + 权限拦截 + UI 渲染│  │ 上下文防爆   │  │
│         │  ~800 行                            │  │ 三级金字塔    │  │
│         └─────────────────────────────────────┘  └──────────────┘  │
│                                                                  │
│         ┌────────────────────┐  ┌──────────────────────────┐     │
│         │   sessionMemory    │  │     extractMemories       │     │
│         │   会话级临时记忆    │  │     跨会话持久记忆         │     │
│         │   ~500 行           │  │     ~600 行                │     │
│         └────────────────────┘  └──────────────────────────┘     │
│                                                                  │
│  ┌──────────────┐  ┌────────────┐  ┌────────────────────────┐    │
│  │  Hooks 系统   │  │  MCP 协议  │  │  权限 / yoloClassifier │    │
│  │ (settings)   │  │  工具扩展   │  │  安全拦截器             │    │
│  │  26 事件类型 │  │  STDIO/SSE │  │  allow/deny/ask         │    │
│  └──────────────┘  └────────────┘  └────────────────────────┘    │
│                                                                  │
│  ════════════════ 以上全部都是 Harness ══════════════════════    │
│                                                                  │
│                ┌─────────────────────────┐                       │
│                │   Claude 大模型 API     │  ◀── 唯一的「大脑」    │
│                │   (Anthropic 云端)      │                       │
│                └─────────────────────────┘                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 端到端七阶段生命周期

当你在终端敲下「帮我查一下 QueryEngine 的问题并写个分析报告」并按下回车，背后发生的是一场由无数齿轮咬合的交响乐。从接收消息到落盘记忆的完整生命周期分为七大阶段：

```
┌─────────────────────────────────────────────────────────┐
│ [阶段 1] 终端接收与命令分发                              │
│ entrypoints/cli.tsx & /commands                         │
│ (斜杠命令本地直接拦截 / 普通对话向下流转)                │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│ [阶段 2] 极速物理环境插桩                                │
│ context.ts & QueryEngine.ts                             │
│ (挂载本地时间戳、Git 状态快照、项目 CLAUDE.md)           │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│ [阶段 3] 提示词预检与缓存折叠                            │
│ microCompact.ts / sessionMemoryCompact / autoCompact    │
│ (清理过期工具冗余 → 容量濒危时偷换为后台摘要)            │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│ [阶段 4] 大模型推理主循环开启                            │
│ query.ts / queryLoop                                    │
│ (流式渲染思考文字 → 决定调用工具 tool_use)               │
└─────────┬─────────────────────────────────┬─────────────┘
          │                                 │
          ▼                                 ▼
┌────────────────────────┐    ┌────────────────────────┐
│ [阶段 5] 工具审查沙盒  │    │ [阶段 6] 多轮缠斗自愈  │
│ Tool.ts / Permissions  │    │ query.ts / Stop Reason │
│ (执行防线 / 拦截自愈)  │    │ (携新线索再查 / 宣布完) │
└────────────┬───────────┘    └─────────────┬──────────┘
             │                              │
             └──────────────┬───────────────┘
                            ▼
┌─────────────────────────────────────────────────────────┐
│ [阶段 7] 背景幽灵善后与潜意识沉淀 (PostHooks)            │
│ 隐蔽派生纯净分身子代理 (Forked Agent)                    │
└──────────┬──────────────────────────┬──────────────────┘
           │                          │
           ▼                          ▼
┌──────────────────────┐    ┌──────────────────────────┐
│  短期复用备忘录       │    │  跨会话长期知识库          │
│  sessionMemory.ts    │    │  extractMemories.ts       │
│  (写下当前回合摘要)   │    │  (萃取经验做卡片放硬盘)   │
└──────────────────────┘    └──────────────────────────┘
```

### 阶段 1：终端接收与命令分发（`entrypoints/cli.tsx` + `/commands`）

- **终端捕获**：React Ink 构建的 TUI 捕获按键输入
- **斜杠命令路由**：正则探测 `/` 前缀
  - `/compact`、`/clear`、`/bug`、`/logout` 等被拦截，调用本地同步函数执行；**大模型甚至不知道你敲过这些字**
  - 普通文本进入主引擎
- **Fast-path 短路**：`--version`、`daemon` 等命令在主引擎前直接返回

### 阶段 2：环境感知与物理世界挂载（`context.ts` + `QueryEngine.ts`）

- **`QueryEngine` 实例化**：会话总调度室接手任务，分配防篡改 UUID，初始化计费表
- **物理世界贴图**（`getUserContext` / `getSystemContext`）：在把用户输入发给大模型之前，系统**偷偷在提示词头部塞入一坨隐藏信息**：
  - 当前电脑本地时间戳（防幻觉）
  - 自动运行并抓取 `git status --short`、`git log --oneline -5` 状态
  - 当前项目根目录的 `CLAUDE.md` 知识库

> **冷启动注射** (Cold-start Injection)：系统不需要做一个 `GetGitStatusTool` 让 AI 第一步去调用，而是在开局就把环境垫在 system prompt 下面，**节省第一个 RoundTrip 的耗时**。

### 阶段 3：缓存审计与阈值报警器（`autoCompact.ts`）

在按下发送的最后一刻，防爆探头扫描历史对话池总 Token 数：

1. **微核删减**（`microCompact.ts`）：如果时间间隔过长，偷偷删掉超长 Bash/grep 旧结果（替换为 `[Old tool result content cleared]`）
2. **重度重组**（`sessionMemoryCompact` / `compact`）：如果总量突破 `effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS(13,000)`，主线程紧急挂起——抽出后台早就写好的 `SESSION_MEMORY.md` 摘要作替换，强制折叠前缀后再发车

### 阶段 4：大模型推理主循环开启（`query.ts:queryLoop`）

进入流式推断的心智阶段，`async function*` 大显神威：

1. 大模型开始输出 **text** 和 **tool_use** 块
2. 流式返回的文本立刻送到 UI 线程渲染打印（用户感觉「它马上就有反应」）
3. 当模型必须查看代码才能回答时，抛出一个 `tool_use`

### 阶段 5：插件生态底座与拦截审批（`Tool.ts:Permissions`）

大模型决定的 `tool_use` 撞击安全墙：

1. **权限防火墙判断**：`alwaysAllow` / `alwaysAsk` / `alwaysDeny` 规则验证；破坏性操作弹窗「AI 申请执行 `npm install`，是否批准？」
2. **沙盒执行**（`Tool.call()`）：授权或安全动作（如只读 `Grep`）立即在本地执行
3. **失败闭合**（Fail-Closed/自愈）：命令出错时，工具捕获报错文本传给 Agent，依靠 `try/catch` 让它「看清错误代码再试一次」

### 阶段 6：多轮缠斗与主动完结（Multiturn & Recursion）

1. **Tool Result 回填**：执行结果回填到对话数组，`query.ts` 携带新结果发起下一轮 API 请求
2. **递归思考**：AI 继续审查查询结果，不够则再次下发 `tool_use`（这就是它思考转半天、不断查文件的本质）
3. **达成终点**（Stop Reason）：当模型分析完所有源码后，输出最终回答且不带任何工具调用，API 返回 `stop_reason: "end_turn"`，系统宣布该回合完结

### 阶段 7：幽灵善后与潜意识凝练（`extractMemories` + `sessionMemory`）

发生在用户已经收到回复、主屏静止时（极其丝滑，不影响交互）。两个后台系统通过**完全不同的钩子管线**分别触发：

- **临时会话记录**（`sessionMemory.ts`）：通过 `registerPostSamplingHook` 挂载触发，派发**写权限被严管**的分身子代理 (Forked Agent)，在后台编辑 `SESSION_MEMORY.md` 供下次折叠上下文用
- **长期记忆凝练**（`extractMemories.ts`）：通过 `handleStopHooks`（位于 `stopHooks.ts`）触发，**走的是完全不同的钩子管线**。借助 Prompt Cache Hit 几乎零成本地生成卡片索引，存入 `~/.claude/projects/.../memory/`。即使重启 CLI 这个幽灵仍然记得

> **关键洞察**：sessionMemory 与 extractMemories 经常被混为一谈，但它们**走的是两条完全独立的 Hook 管线**，触发时机、权限范围、写入目标都不同。详见 [`dd-01-hooks.md`](dd-01-hooks.md) 双管线对比章节。

---

## 五阶段 Harness 生命旅程

从 Harness 子系统视角看，一次完整交互可以拆成五个阶段的模块流转：

### 阶段 ① 冷启动与环境装载

```
用户输入 → CLI 入口解析 → QueryEngine 构造 → context.ts 注射环境
```

- **CLI 入口**：解析命令行参数，Fast-path 判断（`--version` / `daemon` 等）短路返回；预检探针验证 Node.js 版本与执行权限；初始化 React Ink TUI 渲染宿主
- **QueryEngine** 构造单例，持有：
  - `mutableMessages` 数组（持久会话上下文）
  - `readFileState` 跨请求文件缓存
  - `totalUsage` 计费器（Input + Output + CacheRead Tokens）
  - `abortController` 中断钩子
- **context.ts** 并发执行 Git 嗅探、读取 `CLAUDE.md`、注入当前日期锚点

### 阶段 ② 主循环（心跳引擎）

```
QueryEngine.submitMessage() → query() while(true) 状态机
```

`submitMessage()` 是整个系统的重中之重，通过一个大而全的 **AsyncGenerator** 暴露。每一轮用户输入都流经它的五个内部阶段：

1. **`processUserInput`**：斜杠命令拦截消化（不送大模型）
2. **`fetchSystemPromptParts`**：MCP 加载、`hasAutoMemPathOverride()` 检查、`customSystemPrompt` 装载
3. **进入 `query()` 主循环**：`for await (const message of query({...}))`
4. **流式事件拆包**：根据 `message.type` 分发处理 `stream_event`（累加 Usage）、`system`/`attachment`（边界处理）、`assistant`（`void recordTranscript` 火-and-forget 持久化）
5. **硬性预算斩断**：`maxBudgetUsd` 美元瓶颈检测、`error_max_structured_output_retries` 失败检测 → yield `is_error: true` 终止

### 阶段 ③ 工具执行（手脚协调）

```
模型返回 tool_use → Tool.checkPermissions() → 沙盒执行 → 结果回填 → 循环继续
```

- **Tool.ts** 定义了所有工具的元接口（详见 [`03-tool-system.md`](../03-tool-system.md)）：
  - **描述侧**：`inputSchema`（Zod）、`description`、`searchHint`
  - **环境与安全**：`ToolUseContext`（全局权限、文件缓存、`AbortController`、通知接口）；`isDestructive()` / `isReadOnly()` / `isConcurrencySafe()` 特征
  - **运行时拦截**：`validateInput()` 软阻拦 + `checkPermissions()` 与 `ToolPermissionContext` 联动
  - **UI 渲染钩子**：`renderToolUseMessage` / `renderToolUseProgressMessage` / `renderToolResultMessage` / `renderToolUseRejectedMessage` / `renderGroupedToolUse`（全部返回 React.ReactNode 给 Ink）
- 每个工具声明特征函数，YOLO 模式下破坏性操作仍会被单独挂起
- MCP 协议扩展工具通过 Server 发现与 JSON-RPC 握手，映射为原生 Tool 插槽参与分发
- 结果通过 `tool_result` 回填至对话数组，`query.ts` 的 `while(true)` 继续下一轮

### 阶段 ④ 后台影子工作（双记忆系统）

```
对话间隙 → shouldExtractMemory() → Fork Agent → 写记忆文件
```

**Session Memory**（短期复用备忘录）
- `shouldExtractMemory()` 三重门控：容量 + 行为 + 时机（必须等模型「思考完一段落」）
- 派生隔离子代理，权限锁死**只能 Edit `SESSION_MEMORY.md` 单一文件**
- 产出的记忆被 `sessionMemoryCompact` 复用，实现零延迟上下文压缩

**Extract Memories**（跨会话长期知识库）
- 会话结束后自动提炼项目经验，写入 `~/.claude/projects/<slug>/memory/`
- 两阶式索引：内容卡片 `.md` + `MEMORY.md` 索引条目
- 60 秒宽限期逃生窗口（`drainPendingExtraction(timeoutMs=60_000)`）：即使 Ctrl+C 退出也坚持写完
- **Prompt Cache 寄生**：从主线上下文 Fork 出来，享受 Anthropic API 的 Prompt Cache hit，几乎零成本

详见 [`dd-05-memory.md`](dd-05-memory.md)。

### 阶段 ⑤ 终止与持久化

```
无 tool_use → needsFollowUp = false → return completed → recordTranscript 落盘
```

- 模型返回中不再包含 `tool_use` → 跳出 `while(true)`
- `recordTranscript` 将完整对话序列化落盘，支持 `--resume` 恢复
- **多个卡点**（推理前、流中、压缩后）均触发落盘，抗意外杀死

---

## Harness 各子系统职责矩阵

| Harness 子系统          | 职责                                            | 现有笔记                                                       |
| ---------------------- | --------------------------------------------- | -------------------------------------------------------------- |
| CLI 入口 & 引导链        | 命令解析、预检、Fast-path、TUI 初始化、QueryEngine 交接 | [`01-bootstrap-lifecycle.md`](../01-bootstrap-lifecycle.md)    |
| QueryEngine（底盘）     | 会话生命周期、配置集成、计费监控、持久化              | [`dd-11-query-engine.md`](dd-11-query-engine.md)                |
| query.ts（发动机）      | 流式状态机、工具分发、错误自愈、上下文管理              | [`02-conversation-loop.md`](../02-conversation-loop.md)        |
| context.ts（世界观）    | Git 感知、CLAUDE.md 挂载、时间锚点                  | [`07-context-engine.md`](../07-context-engine.md)              |
| Tool 系统（工具规范）    | 工具插槽接口、沙盒管线、权限拦截、UI 渲染              | [`03-tool-system.md`](../03-tool-system.md)                    |
| compact/（防爆体系）    | 三级上下文压缩：micro → sessionMemory → legacy   | [`dd-07-context-compaction.md`](dd-07-context-compaction.md)    |
| sessionMemory（临时记忆）| 会话内后台总结 & 上下文压缩复用                       | [`dd-05-memory.md`](dd-05-memory.md)                            |
| extractMemories（持久记忆）| 跨会话项目经验提炼 & 两阶索引                      | [`dd-05-memory.md`](dd-05-memory.md)                            |
| Hooks 系统              | settings.json 事件钩子、生命周期注入、自动化行为      | [`dd-01-hooks.md`](dd-01-hooks.md)                              |
| 权限路由引擎            | YOLO 模式、allow/deny/ask、yoloClassifier        | [`dd-08-permission-classifier.md`](dd-08-permission-classifier.md) |
| MCP 协议层              | 外部工具扩展协议挂载、Server 发现与握手               | [`09-mcp-integration.md`](../09-mcp-integration.md)            |

---

## 三大设计哲学

理解了模块拓扑后，更重要的是理解 Anthropic 团队**为什么这么设计**。Harness 工程贯穿三条核心哲学：

### 1. 洋葱模型：层层包裹大模型

大模型（API）是最内核的「大脑」。Harness 的每一层都是围绕它的保护壳和能力放大器：

```
         ┌──────────────────────────────────┐
         │    最外层：CLI 入口 & 引导链      │
         │  ┌────────────────────────────┐  │
         │  │  外围：QueryEngine + Hooks │  │
         │  │  ┌──────────────────────┐  │  │
         │  │  │  中间：Tool + compact │  │  │
         │  │  │  ┌────────────────┐  │  │  │
         │  │  │  │ 最内：query.ts │  │  │  │
         │  │  │  │  ┌──────────┐  │  │  │  │
         │  │  │  │  │  LLM API │  │  │  │  │
         │  │  │  │  └──────────┘  │  │  │  │
         │  │  │  └────────────────┘  │  │  │
         │  │  └──────────────────────┘  │  │
         │  └────────────────────────────┘  │
         └──────────────────────────────────┘
```

- **最内层**：`query.ts` 直接与 API 对话，是状态机
- **中间层**：Tool 系统给它「手脚」，compact 系统防它「脑溢血」
- **外围层**：QueryEngine 管理生命周期，Hooks 系统实现自动化
- **最外层**：CLI 与 TUI 接收人类输入

每一层只负责自己的关注点，向内层暴露干净的接口，向外层屏蔽细节。

### 2. 防御性设计贯穿始终

- **上下文管理是金字塔式三级阶梯防线**，不是一刀切：
  1. `microCompact` 摘旁枝末节（缓存编辑、时间断层）
  2. `sessionMemoryCompact` 复用廉价的备胎成果（后台已写好的摘要）
  3. `compact.ts` 终极重构（昂贵的 API 总结）
- **工具执行**有沙盒 + 权限 + 并发安全三重保护
- **后台记忆代理**被零信任权限锁死：
  - `sessionMemory` 子代理只能 Edit 一个文件 (`createMemoryFileCanUseTool`)
  - `extractMemories` 子代理只能 Read/Grep/Glob 全工程 + 在 memory 目录内 Write (`createAutoMemCanUseTool`)
- **所有外部数据输入**都有长度截断兜底：`MAX_STATUS_CHARS=2000`、`MAX_ENTRYPOINT_LINES=200`、`maxResultSizeChars=30000`
- **Tool 工厂 Fail-Closed 默认**：`buildTool()` 不显式声明 `isConcurrencySafe` 时默认为 `false`

> **工程师思维的对照**：当绝大部分 AI 衍生客户端用 `messages.splice(0, length-10)` 这种粗暴野蛮、容易导致语境断裂的切片方法防爆时，Claude 团队构建的是一张立体的、防缓存穿透的阶梯网。这才是「Agent 耐力赛引擎」应当具备的素质。

### 3. 异步非阻塞的影子系统

后台记忆提取、Prompt Cache 寄生、会话总结——这些「影子工作」全部并发不阻塞：

- **双 Hook 管线分离**：sessionMemory 走 `registerPostSamplingHook`（采样后切面），extractMemories 走 `handleStopHooks`（终止后切面）。两者保持核心状态机 `query.ts` 的纯净
- **Forked Agent 完美克隆**：从主线上下文 Fork 子代理，享受 Anthropic API Prompt Cache hit，几乎零成本生成记忆
- **60 秒宽限期**：`drainPendingExtraction` 在 CLI 被 Ctrl+C 中断退出前，向系统申请 60 秒隐形宽限期，保证用户强退也能拼死写完
- **火-and-forget 持久化**：`void recordTranscript(messages)` 不 await，避免堵塞渲染帧
- **Local Command 伪装**：斜杠命令的本地输出通过 `localCommandOutputToSDKAssistantMessage` 被伪造成 AI 说的话，统一 UI 渲染逻辑

用户感知到的是丝滑的打字体验，看不见 Harness 在暗处做的大量「家务活」。

---

## 关键常量速查

理解 Harness 必须知道的数字：

| 常量                              | 数值        | 含义                                          |
| --------------------------------- | ---------- | ------------------------------------------- |
| `AUTOCOMPACT_BUFFER_TOKENS`       | 13,000     | 距离上下文窗口上限多少 Token 时触发自动压缩       |
| `MAX_STATUS_CHARS`                | 2,000      | `git status` 输出截断阈值                     |
| `MAX_ENTRYPOINT_LINES`            | 200        | `MEMORY.md` 入口加载行数上限                  |
| `MAX_ENTRYPOINT_BYTES`            | 25,000     | `MEMORY.md` 入口加载字节上限                  |
| `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` | 200,000 | 单条用户消息内所有 tool_result 累计上限       |
| `BashTool.maxResultSizeChars`     | 30,000     | Bash 工具单次结果持久化阈值                   |
| `drainPendingExtraction.timeoutMs`| 60,000     | extractMemories 强退宽限期                   |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3      | 连续压缩失败断路器阈值                        |
| `POST_COMPACT_TOTAL_TOKEN_BUDGET` | 50,000     | 压缩后恢复总预算                             |

---

## 配套深入阅读

| 主题            | 文档                                                                  |
| -------------- | -------------------------------------------------------------------- |
| **总体架构**    | [`00-architecture-overview.md`](../00-architecture-overview.md)      |
| **冷启动引导**  | [`01-bootstrap-lifecycle.md`](../01-bootstrap-lifecycle.md)          |
| **主对话循环**  | [`02-conversation-loop.md`](../02-conversation-loop.md)              |
| **Tool 系统**   | [`03-tool-system.md`](../03-tool-system.md)                          |
| **上下文引擎**  | [`07-context-engine.md`](../07-context-engine.md)                    |
| **MCP 集成**   | [`09-mcp-integration.md`](../09-mcp-integration.md)                  |
| **权限安全**    | [`11-permission-security.md`](../11-permission-security.md)          |
| **Hooks 机制**  | [`dd-01-hooks.md`](dd-01-hooks.md)                                   |
| **记忆系统**    | [`dd-05-memory.md`](dd-05-memory.md)                                 |
| **上下文压缩**  | [`dd-07-context-compaction.md`](dd-07-context-compaction.md)         |
| **权限分类器**  | [`dd-08-permission-classifier.md`](dd-08-permission-classifier.md)   |
| **Query 引擎**  | [`dd-11-query-engine.md`](dd-11-query-engine.md)                     |

---

## 总结

Claude Code 的工程价值不在于「调用了一个聪明的 LLM」，而在于围绕 LLM 构建的这套**完整的运行时宿主**：

- 它把无状态的 API 调用包装成有状态的会话引擎
- 它把概率性的文本输出约束在确定性的工具沙盒里
- 它把易丢失的对话上下文沉淀成可重用的记忆文件
- 它把后台异步工作隐藏在丝滑的用户体验之后

**Harness ≠ LLM。Harness 才是把 LLM 变成 Agent 的关键。** 这套架构思路，是任何想把大模型工程化落地的开发者都值得反复研读的范本。
