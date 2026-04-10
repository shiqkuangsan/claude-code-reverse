# claude-code-reverse

Reverse-engineered source of [`@anthropic-ai/claude-code`](https://www.npmjs.com/package/@anthropic-ai/claude-code) v2.1.88, restored from the published source map for educational study.

## Structure

```
sources/          # Original npm package files
  cli.js          # Bundled entry point (13MB)
  cli.js.map      # Source map (60MB)
  sdk-tools.d.ts  # SDK type definitions
  package.json    # Package metadata

restored/         # Source restored from source map
  src/            # Application source (1,884 TypeScript/TSX files)
  vendor/         # Native addon source
  node_modules/   # Bundled dependencies

docs/             # Architecture & mechanism analysis (34 documents)
  00-11           # Architecture overview series (12 docs)
  deep-dive/      # Deep-dive mechanism analysis (22 docs)
```

## Documentation

34 documents covering the complete internals of Claude Code, organized in two series:

### Architecture Overview (12 docs)

| # | Document | Topic |
|---|----------|-------|
| 00 | [Architecture Overview](docs/00-architecture-overview.md) | Bird's-eye view, module map, data flow |
| 01 | [Bootstrap & Lifecycle](docs/01-bootstrap-lifecycle.md) | Startup sequence, 10 phases, CLI options |
| 02 | [Conversation Loop](docs/02-conversation-loop.md) | query() async generator, message flow |
| 03 | [Tool System](docs/03-tool-system.md) | Tool interface, registry, permissions, hooks |
| 04 | [Agent & Task System](docs/04-agent-task-system.md) | 7 task types, AgentTool, Teams/Swarms |
| 05 | [Command System](docs/05-command-system.md) | 80+ commands, registry, filtering |
| 06 | [Terminal UI](docs/06-terminal-ui.md) | Custom Ink framework, rendering pipeline, Vim |
| 07 | [Context Engine](docs/07-context-engine.md) | CLAUDE.md, context collection, compaction |
| 08 | [Plugin & Skill System](docs/08-plugin-skill-system.md) | Plugin lifecycle, SKILL.md, hooks |
| 09 | [MCP Integration](docs/09-mcp-integration.md) | MCP client, tool mapping, resources |
| 10 | [Services & Infrastructure](docs/10-services-infrastructure.md) | API, auth, analytics, settings, LSP |
| 11 | [Permission & Security](docs/11-permission-security.md) | Permission modes, classifier, sandbox |

### Deep Dive Series (22 docs)

| # | Document | Topic |
|---|----------|-------|
| dd-01 | [Hooks](docs/deep-dive/dd-01-hooks.md) | 26 events, 4 hook types, execution engine |
| dd-02 | [Plugins](docs/deep-dive/dd-02-plugins.md) | Manifest, marketplace, lifecycle |
| dd-03 | [Subagent Orchestration](docs/deep-dive/dd-03-subagent.md) | Fork cache sharing, isolation modes |
| dd-04 | [Agent Teams / Swarms](docs/deep-dive/dd-04-teams-swarms.md) | Mailbox, permission sync, tmux/in-process |
| dd-05 | [Memory System](docs/deep-dive/dd-05-memory.md) | 5-layer architecture, auto dream |
| dd-06 | [Remote Control](docs/deep-dive/dd-06-remote.md) | CCR, Bridge, Teleport, WebSocket |
| dd-07 | [Context Compaction](docs/deep-dive/dd-07-context-compaction.md) | 4-layer token management |
| dd-08 | [Permission Classifier](docs/deep-dive/dd-08-permission-classifier.md) | Auto mode, YOLO classifier, bash security |
| dd-09 | [GrowthBook Features](docs/deep-dive/dd-09-growthbook-features.md) | Compile-time + runtime feature gates |
| dd-10 | [Settings System](docs/deep-dive/dd-10-settings.md) | 7-layer merge, MDM, hot-reload |
| dd-11 | [Query Engine](docs/deep-dive/dd-11-query-engine.md) | Streaming, error recovery, token budget |
| dd-12 | [OAuth / Auth](docs/deep-dive/dd-12-oauth-auth.md) | PKCE, keychain, multi-backend |
| dd-13 | [Sandbox / Bash Security](docs/deep-dive/dd-13-sandbox.md) | 23 security checks, AST analysis |
| dd-14 | [LSP Integration](docs/deep-dive/dd-14-lsp.md) | Server management, diagnostics, 8 operations |
| dd-15 | [Cost & Rate Limits](docs/deep-dive/dd-15-cost.md) | Model pricing, quota, early warnings |
| dd-16 | [Vim State Machine](docs/deep-dive/dd-16-vim.md) | 11 substates, motions, operators, text objects |
| dd-17 | [Worktree Isolation](docs/deep-dive/dd-17-worktree.md) | Session/agent worktrees, hooks |
| dd-18 | [Background Sessions](docs/deep-dive/dd-18-background.md) | Daemon mode, bg/ps/logs/attach/kill |
| dd-19 | [Voice Mode](docs/deep-dive/dd-19-voice.md) | Audio capture, STT, hold-to-talk |
| dd-20 | [Prompt Suggestions](docs/deep-dive/dd-20-suggestions.md) | Suggestion generation, speculation, tips |
| dd-21 | [Output Styles](docs/deep-dive/dd-21-output-styles.md) | Style system, plugin integration |
| dd-22 | [Harness Engine Overview](docs/deep-dive/dd-22-harness-engine-overview.md) | Harness 全景：洋葱模型、七阶段生命周期、三大设计哲学 |

## Tech Stack

- **Language**: TypeScript (TSX/TS), ESM modules
- **Build**: Bun bundler (single-file cli.js)
- **UI**: React + custom Ink terminal framework + Yoga layout
- **CLI**: Commander.js
- **Runtime**: Node.js 18+

## Disclaimer

This repository is for **educational and research purposes only**. All rights to Claude Code belong to [Anthropic](https://www.anthropic.com/). The restored source is derived from publicly available npm package artifacts. Do not use this code for commercial purposes or to create competing products.
