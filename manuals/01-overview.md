# Claude Code: Architecture Overview

## What Is Claude Code?

Claude Code is Anthropic's official CLI tool for interacting with Claude models directly from the terminal. It provides an interactive REPL (Read-Eval-Print-Loop) for conversations with Claude, augmented with a rich set of tools that let the AI read files, edit code, run shell commands, search the web, and more.

## Core Architecture

Claude Code is built as a **TypeScript application** running on the **Bun** runtime, using **React + Ink** for terminal UI rendering. The architecture follows a layered design:

```
+---------------------------------------------+
|                  main.tsx                     |  CLI Entrypoint
|  (Commander.js CLI parsing, startup flow)    |
+---------------------------------------------+
          |                    |
          v                    v
+------------------+  +------------------+
|   QueryEngine    |  |      REPL        |  Two execution modes
|   (SDK/headless) |  |   (interactive)  |
+------------------+  +------------------+
          |                    |
          v                    v
+---------------------------------------------+
|               query.ts                       |  Query Loop
|  (AsyncGenerator streaming loop)             |
+---------------------------------------------+
          |
          v
+---------------------------------------------+
|        Tool System (Tool.ts + tools.ts)      |  50+ tools
|  Bash, Read, Edit, Write, Grep, Glob,        |
|  Agent, WebFetch, WebSearch, MCP tools...    |
+---------------------------------------------+
          |
          v
+---------------------------------------------+
|        Anthropic API (SDK)                   |
|  Streaming responses, tool calls, retries    |
+---------------------------------------------+
```

## Key Modules

| File | Purpose |
|------|---------|
| `main.tsx` | CLI entrypoint: argument parsing, startup orchestration, parallel prefetching |
| `QueryEngine.ts` | Session-scoped query lifecycle for SDK/headless mode |
| `query.ts` | Core streaming query loop (AsyncGenerator-based) |
| `Tool.ts` | Base tool type definitions, permission system, `buildTool()` factory |
| `tools.ts` | Tool registry: assembles all available tools with feature-flag gating |
| `commands.ts` | Slash command registry: ~70+ commands loaded conditionally |
| `context.ts` | System/user context collection: git status, CLAUDE.md, memory files |
| `cost-tracker.ts` | Token usage and cost tracking per model, per session |
| `setup.ts` | First-run setup: worktree creation, hooks, permissions, telemetry |
| `history.ts` | JSONL-based prompt history with paste content storage |
| `tasks.ts` | Background task registry (shell, agent, workflow, monitor tasks) |
| `Task.ts` | Task type definitions and state machine |
| `ink.ts` | Ink renderer wrapper with ThemeProvider |

## Execution Modes

Claude Code operates in two primary modes:

1. **Interactive (REPL)**: Full terminal UI with streaming output, tool use rendering, permission prompts, and keyboard shortcuts
2. **Headless/SDK (`--print`, `-p`)**: Pipe-friendly mode for scripting, CI/CD, and programmatic use via the Agent SDK

Both modes share the same `query()` function for the LLM interaction loop.

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Bun |
| Language | TypeScript (strict) |
| Terminal UI | React + Ink |
| CLI Framework | Commander.js |
| Validation | Zod v4 |
| API Client | Anthropic SDK |
| Feature Flags | `bun:bundle` compile-time flags |
| State Management | Custom store (immutable updates) |
