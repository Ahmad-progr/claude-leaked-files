# CLAUDE.md

## Project Overview

This repository contains a snapshot of Claude Code's TypeScript source files, preserved for educational and security research purposes. These are **read-only reference files** — not a buildable or runnable project.

## What This Is

A collection of ~20 TypeScript/TSX source files extracted from Claude Code's npm package via an exposed source map (March 31, 2026). The files represent core modules of Claude Code's CLI architecture.

## Key Files

| File | Purpose |
|------|---------|
| **`main.tsx`** | CLI entrypoint (785KB): command parsing (Commander.js), Ink UI init, 5-phase startup with parallel prefetching of MDM/keychain/API, migrations, deferred prefetches |
| **`QueryEngine.ts`** | Session-scoped query lifecycle for SDK/headless mode: streaming responses, tool-call loops, permission denial tracking, thinking mode |
| **`query.ts`** | Core AsyncGenerator streaming loop: 5-layer context management (tool result budget → snip → microcompact → collapse → autocompact), max_output_tokens recovery, token budget tracking |
| **`Tool.ts`** | Base tool type (~40+ fields/methods), `buildTool()` factory with fail-closed defaults, `ToolUseContext` threading, permission interfaces |
| **`tools.ts`** | Tool registry: 50+ tools with feature-flag gating via `bun:bundle`, sorted assembly for prompt-cache stability, deny rule filtering |
| **`commands.ts`** | Slash command registry: ~70+ commands from 7 sources (bundled, plugin, skill dirs, workflows, MCP, built-in), availability filtering by auth state |
| **`context.ts`** | System/user context: parallel git status collection (5 commands via Promise.all), CLAUDE.md discovery, memoized with clearable caches |
| **`cost-tracker.ts`** | Per-model token/cost tracking with recursive advisor usage, session persistence/restore, formatted display with canonical name aggregation |
| **`setup.ts`** | Setup: Node.js version check, UDS messaging, terminal backup restoration, hooks snapshot, worktree/tmux creation, permission validation, telemetry |
| **`history.ts`** | JSONL prompt history with content-addressable paste storage, file locking for concurrent sessions, session-aware ordering, undo support |
| **`tasks.ts`** | Background task registry: 6 task types (shell, agent, remote, teammate, workflow, monitor) with feature-flag gating |
| **`Task.ts`** | Task type definitions, 5-state lifecycle (pending→running→completed/failed/killed), cryptographically random ID generation |
| **`ink.ts`** | Ink renderer wrapper: ThemeProvider injection, 30+ exported components/hooks/events |
| **`costHook.ts`** | React hook: displays cost summary and persists session costs on process exit |
| **`replLauncher.tsx`** | REPL launch: dynamic imports of App and REPL components for fast startup |
| **`projectOnboardingState.ts`** | New project onboarding: workspace/CLAUDE.md step tracking, shown up to 4 times |
| **`dialogLaunchers.tsx`** | Dialog launchers: resume chooser, snapshot updates, teleport, invalid settings |
| **`interactiveHelpers.tsx`** | Interactive helpers: setup screens, error exits, render context, trust flow |

## Technology Stack

- **Runtime:** Bun
- **Language:** TypeScript (strict)
- **UI:** React + Ink (terminal)
- **CLI Framework:** Commander.js
- **Validation:** Zod v4
- **API:** Anthropic SDK
- **Feature Flags:** `bun:bundle` compile-time flags (35+ flags: PROACTIVE, KAIROS, BRIDGE_MODE, COORDINATOR_MODE, HISTORY_SNIP, etc.)
- **State Management:** Custom immutable store with `DeepImmutable` types

## Important Patterns

- **Feature-flag gating:** Tools and commands are conditionally loaded via `feature('FLAG_NAME')` from `bun:bundle` and `process.env.USER_TYPE === 'ant'` checks. Enables dead code elimination for external builds.
- **Parallel prefetching:** Side-effect imports at top of `main.tsx` fire MDM/keychain subprocesses before 200+ imports evaluate (~135ms of free parallelism). Deferred prefetches run after first render.
- **Fail-closed defaults:** `buildTool()` defaults `isConcurrencySafe` to `false` and `isReadOnly` to `false` — new tools are restrictive until explicitly opted in.
- **AsyncGenerator streaming:** The query loop uses `yield*` delegation through composable generators for cancellable, error-propagating streaming.
- **Memoized context:** `getSystemContext` and `getUserContext` are memoized with `cache.clear?.()` for invalidation when data changes.
- **Prompt cache stability:** Tool ordering is alphabetically sorted with built-ins as contiguous prefix. Settings paths use content hashes (not random UUIDs) to avoid cache busting.
- **Trust-gated git:** Git commands can execute arbitrary code via hooks, so they only run after trust dialog acceptance.
- **Branded safety types:** `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` serves as a code review guardrail for PII prevention.

## Documentation

Comprehensive technical manuals are available in the **`manuals/`** directory:

| Manual | Topic |
|--------|-------|
| [01-overview.md](manuals/01-overview.md) | Architecture diagram, module map, execution modes |
| [02-startup-and-initialization.md](manuals/02-startup-and-initialization.md) | 5-phase startup, parallel prefetching, anti-debugging, bare mode |
| [03-tool-system.md](manuals/03-tool-system.md) | Tool interface, registry, 50+ tools, permission system, MCP integration |
| [04-query-engine-and-loop.md](manuals/04-query-engine-and-loop.md) | QueryEngine class, streaming loop, 5 context layers, recovery |
| [05-slash-commands.md](manuals/05-slash-commands.md) | Complete 70+ command list, loading from 7 sources, bridge safety |
| [06-context-and-memory.md](manuals/06-context-and-memory.md) | Git status, CLAUDE.md, prompt history, paste storage |
| [07-cost-tracking.md](manuals/07-cost-tracking.md) | Per-model tracking, recursive advisor costs, session persistence |
| [08-task-system.md](manuals/08-task-system.md) | 6 task types, lifecycle states, session-scoped state |
| [09-feature-flags-and-builds.md](manuals/09-feature-flags-and-builds.md) | 35+ flags, dead code elimination, internal vs external builds |
| [10-ui-and-rendering.md](manuals/10-ui-and-rendering.md) | React + Ink, themes, tool rendering lifecycle, FPS tracking |
| [11-security-and-permissions.md](manuals/11-security-and-permissions.md) | 4 permission modes, fail-closed defaults, sandbox, anti-debugging |
| [12-design-patterns.md](manuals/12-design-patterns.md) | 15 reusable engineering patterns with code examples |

## Working With These Files

Since this is a reference archive (no `package.json`, no build system), typical workflows are:
- **Reading and analyzing** the source code
- **Searching** for patterns, architecture decisions, or implementation details
- **Comparing** against public documentation or other CLI tools
- **Documenting** findings for research purposes
- **Consulting the manuals** in `manuals/` for detailed explanations of each subsystem

Do not attempt to build, run, or install dependencies — the files are incomplete extracts.
