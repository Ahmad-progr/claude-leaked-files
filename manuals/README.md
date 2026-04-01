# Claude Code Source Manual

A comprehensive technical manual derived from analysis of Claude Code's TypeScript source files (March 2026 snapshot). This manual documents the internal architecture, design patterns, and engineering decisions of Anthropic's CLI tool for Claude.

## Table of Contents

| # | Document | Description |
|---|----------|-------------|
| 00 | [Architecture Deep Dive](00-architecture-deep-dive.md) | End-to-end architecture with ASCII flow diagrams, request lifecycle walkthrough |
| 01 | [Architecture Overview](01-overview.md) | High-level architecture, module map, technology stack |
| 02 | [Startup and Initialization](02-startup-and-initialization.md) | Startup sequence, parallel prefetching, trust gating, bare mode |
| 03 | [Tool System](03-tool-system.md) | Tool interface, `buildTool()` factory, registry, permissions, MCP |
| 04 | [Query Engine and Loop](04-query-engine-and-loop.md) | QueryEngine class, streaming loop, context management layers, recovery |
| 05 | [Slash Commands](05-slash-commands.md) | Complete command list, loading architecture, availability filtering |
| 06 | [Context and Memory](06-context-and-memory.md) | Git status, CLAUDE.md, memory files, prompt history, paste storage |
| 07 | [Cost and Token Tracking](07-cost-tracking.md) | Per-model usage, cost calculation, session persistence, budget management |
| 08 | [Task System](08-task-system.md) | Background tasks, lifecycle states, task types, session-scoped state |
| 09 | [Feature Flags and Builds](09-feature-flags-and-builds.md) | `bun:bundle` flags, dead code elimination, internal vs external builds |
| 10 | [UI and Rendering](10-ui-and-rendering.md) | React + Ink, theme system, tool rendering, FPS tracking |
| 11 | [Security and Permissions](11-security-and-permissions.md) | Permission modes, fail-closed defaults, trust gating, anti-debugging |
| 12 | [Design Patterns](12-design-patterns.md) | 15 reusable engineering patterns extracted from the codebase |

## Source Files Analyzed

- `main.tsx` — CLI entrypoint (785KB, the largest file)
- `QueryEngine.ts` — SDK/headless query lifecycle
- `query.ts` — Core streaming query loop
- `Tool.ts` — Tool type definitions and factory
- `tools.ts` — Tool registry with feature-flag gating
- `commands.ts` — Slash command registry (~70+ commands)
- `context.ts` — System/user context collection
- `cost-tracker.ts` — Token usage and cost tracking
- `setup.ts` — First-run setup and configuration
- `history.ts` — Session history with paste storage
- `tasks.ts` — Background task registry
- `Task.ts` — Task type definitions and state machine
- `ink.ts` — Ink renderer wrapper
- `costHook.ts` — React hook for cost display
- `replLauncher.tsx` — REPL launch wrapper
- `projectOnboardingState.ts` — New project onboarding
- `dialogLaunchers.tsx` — Dialog launch helpers
- `interactiveHelpers.tsx` — Interactive mode helpers

## Key Takeaways

1. **Startup time is a first-class concern** — Import ordering, parallel prefetching, and deferred work are carefully orchestrated
2. **Security is fail-closed by default** — Permissions, concurrency safety, and read-only flags default to the restrictive option
3. **AsyncGenerator is the streaming primitive** — The query loop uses generators for composable, cancellable streaming
4. **Feature flags enable single-codebase multi-build** — Compile-time flags tree-shake entire features from external builds
5. **Prompt cache stability drives architectural decisions** — Tool ordering, file naming, and settings paths are designed to maximize API cache hits
