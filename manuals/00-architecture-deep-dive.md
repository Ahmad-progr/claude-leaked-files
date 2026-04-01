# Claude Code: Architecture Deep Dive

A comprehensive guide to understanding how every component in Claude Code connects — from the moment you type `claude` to the moment a response appears in your terminal.

---

## 1. The Big Picture

Claude Code is an **agentic CLI** — it doesn't just send a prompt and receive a response. It runs a loop: the LLM responds, requests tool calls, Claude Code executes them, sends results back, and the LLM continues. This loop is the core of everything.

```
User types "claude"
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  main.tsx — CLI Entrypoint                                       │
│                                                                  │
│  1. Side-effect imports (MDM, keychain prefetch)                 │
│  2. Module evaluation (~135ms, subprocess reads in parallel)     │
│  3. Commander.js parses argv                                     │
│  4. preAction hook: MDM await → init() → sinks → migrations     │
│  5. Detect mode: interactive vs headless (-p)                    │
│  6. Subcommand routing (mcp serve, ssh, assistant, open, etc.)   │
│                                                                  │
│         ┌──────────────┐              ┌──────────────────┐       │
│         │  Interactive  │              │  Headless (-p)   │       │
│         │  (REPL path)  │              │  (print path)    │       │
│         └──────┬───────┘              └────────┬─────────┘       │
└────────────────┼───────────────────────────────┼─────────────────┘
                 │                               │
                 ▼                               ▼
         ┌──────────────┐              ┌──────────────────┐
         │  setup.ts     │              │  QueryEngine.ts  │
         │  → REPL.tsx   │              │  (SDK lifecycle)  │
         │  → Ink render │              │                  │
         └──────┬───────┘              └────────┬─────────┘
                 │                               │
                 ▼                               ▼
         ┌─────────────────────────────────────────────┐
         │              query.ts                        │
         │  AsyncGenerator streaming loop               │
         │  (shared by both paths)                      │
         └──────────────────┬──────────────────────────┘
                            │
              ┌─────────────┼─────────────────┐
              ▼             ▼                  ▼
       ┌───────────┐ ┌───────────┐   ┌──────────────┐
       │ Anthropic  │ │   Tool    │   │   Context     │
       │ API call   │ │ Execution │   │  Management   │
       │ (streaming)│ │ (50+ tools)│  │  (5 layers)   │
       └───────────┘ └───────────┘   └──────────────┘
```

## 2. Startup Pipeline (main.tsx)

The startup is the most performance-optimized part of the codebase. Understanding it reveals how production CLI tools should boot.

### Phase-by-Phase

```
Time 0ms ─── profileCheckpoint('main_tsx_entry')
             startMdmRawRead()          ← Subprocess: plutil/reg query
             startKeychainPrefetch()    ← Subprocess: macOS keychain reads

Time 0-135ms ── Module evaluation (200+ import statements)
                While imports load, MDM + keychain subprocesses run in parallel

Time ~135ms ── profileCheckpoint('main_tsx_imports_loaded')

Time ~140ms ── main() function called:
               ├─ Security: NoDefaultCurrentDirectoryInExePath (Windows PATH hijack prevention)
               ├─ Signal handlers: SIGINT, exit (cursor reset)
               ├─ Anti-debugging check (external builds only)
               ├─ Subcommand detection: ssh, assistant, open, cc:// URL, deep links
               ├─ Mode detection: -p/--print, --init-only, --sdk-url, !isTTY
               ├─ Client type: cli, sdk-cli, sdk-typescript, remote, github-action, etc.
               ├─ eagerLoadSettings() ← Parse --settings flag before init()
               └─ run() ← Commander.js command tree

Time ~150ms ── Commander preAction hook:
               ├─ await ensureMdmSettingsLoaded()      ← ~Free (subprocess finished)
               ├─ await ensureKeychainPrefetchCompleted() ← ~Free (subprocess finished)
               ├─ await init()                         ← Heavy: telemetry, auth, config
               ├─ initSinks()                          ← Event queue drain
               ├─ runMigrations()                      ← Version-gated, runs once
               └─ loadRemoteManagedSettings()          ← Non-blocking

Time ~200ms ── Default command .action() handler:
               ├─ setup() ← Node check, UDS, hooks, worktree, permissions
               ├─ initBundledSkills(), initBuiltinPlugins()
               ├─ getCommands(cwd) ← Parallel: skills + plugins + workflows
               └─ REPL render OR headless execution

Time ~250ms ── First render visible to user
               └─ startDeferredPrefetches() ← Everything else (runs while user types)
```

### Client Type Detection

Claude Code identifies its execution context at startup:

```
CLAUDE_CODE_ENTRYPOINT    │  clientType
──────────────────────────┼────────────────
(unset, TTY)              │  cli
(unset, !TTY or -p)       │  sdk-cli
sdk-ts                    │  sdk-typescript
sdk-py                    │  sdk-python
mcp serve                 │  mcp
claude-vscode             │  claude-vscode
local-agent               │  local-agent
claude-desktop            │  claude-desktop
remote / session token    │  remote
GITHUB_ACTIONS=true       │  github-action
```

This drives behavior differences across the codebase: permission handling, telemetry, UI, and features.

## 3. The Two Execution Paths

### Interactive Path (REPL)

```
setup.ts
  │
  ├─ Node.js ≥18 check
  ├─ UDS messaging server (Unix Domain Socket for IPC)
  ├─ Terminal backup restoration (iTerm2, Terminal.app)
  ├─ setCwd() ← Must be first (hooks depend on it)
  ├─ captureHooksConfigSnapshot()
  ├─ initializeFileChangedWatcher()
  ├─ Worktree creation (--worktree → git worktree + tmux)
  ├─ Background services (session memory, context collapse)
  ├─ Command prefetch: getCommands(cwd)
  ├─ Permission validation (bypassPermissions → sandbox check)
  └─ tengu_started telemetry event
        │
        ▼
replLauncher.tsx
  │
  ├─ Dynamic import: App.js, REPL.js (deferred for fast startup)
  └─ renderAndRun(root, <App><REPL /></App>)
        │
        ▼
REPL.tsx (React + Ink)
  │
  ├─ Text input with history (↑/↓, Ctrl+R search)
  ├─ Slash command autocomplete
  ├─ Tool use rendering (progress, results, groups)
  ├─ Permission prompt dialogs
  ├─ On submit → processUserInput() → query()
  └─ useCostSummary() ← Hook: prints costs on exit
```

### Headless Path (SDK / --print)

```
QueryEngine.ts
  │
  ├─ constructor(config) ← Tools, commands, MCP, model, budget
  │
  └─ async *submitMessage(prompt)
       │
       ├─ setCwd(cwd)
       ├─ Wrap canUseTool() for denial tracking
       ├─ fetchSystemPromptParts() ← System prompt + contexts
       ├─ Build final systemPrompt (custom + memory + append)
       ├─ processUserInput() ← Slash commands, attachments
       ├─ recordTranscript() ← Persist for --resume
       ├─ yield buildSystemInitMessage() ← SDK init event
       │
       └─ for await (event of query(...))
            └─ yield SDKMessage events to caller
```

Both paths converge at `query()`.

## 4. The Query Loop (query.ts)

This is the most complex and important part of the system. It's an `AsyncGenerator` that manages the complete agentic conversation loop.

### Data Flow Per Iteration

```
┌──────────────────────────────────────────────────────────────────┐
│                    QUERY LOOP ITERATION                          │
│                                                                  │
│  messages[] ──────────────────────────────────────────────┐      │
│       │                                                   │      │
│       ▼                                                   │      │
│  ┌─────────────────┐                                      │      │
│  │ 1. Tool Result   │ Large results → disk file + preview │      │
│  │    Budget        │ Exempt: tools with maxResultSize=∞  │      │
│  └────────┬────────┘                                      │      │
│           ▼                                               │      │
│  ┌─────────────────┐                                      │      │
│  │ 2. Snip         │ Remove old messages above threshold  │      │
│  │    Compaction    │ (HISTORY_SNIP feature flag)          │      │
│  └────────┬────────┘                                      │      │
│           ▼                                               │      │
│  ┌─────────────────┐                                      │      │
│  │ 3. Microcompact │ Fine-grained cached cache-editing    │      │
│  │                  │ (CACHED_MICROCOMPACT feature)        │      │
│  └────────┬────────┘                                      │      │
│           ▼                                               │      │
│  ┌─────────────────┐                                      │      │
│  │ 4. Context      │ Read-time projection over full       │      │
│  │    Collapse     │ history. Summaries in separate store. │      │
│  │                  │ (CONTEXT_COLLAPSE feature)           │      │
│  └────────┬────────┘                                      │      │
│           ▼                                               │      │
│  ┌─────────────────┐                                      │      │
│  │ 5. Auto-Compact │ Threshold-based full summarization   │      │
│  │                  │ Runs last — earlier layers may       │      │
│  │                  │ prevent it from being needed.        │      │
│  └────────┬────────┘                                      │      │
│           ▼                                               │      │
│  ┌─────────────────┐    ┌──────────────────────────┐      │      │
│  │ 6. API Call     │───▶│ Anthropic Streaming API   │      │      │
│  │  prependUser    │    │ model, tools, system,     │      │      │
│  │  Context +      │    │ thinking, effort, advisor │      │      │
│  │  appendSystem   │    └────────────┬─────────────┘      │      │
│  │  Context        │                 │                     │      │
│  └─────────────────┘                 │                     │      │
│                                      ▼                     │      │
│                           ┌──────────────────┐             │      │
│                           │ Streaming Events  │             │      │
│                           │ (yield to caller) │             │      │
│                           └────────┬─────────┘             │      │
│                                    │                       │      │
│                    ┌───────────────┼──────────────┐        │      │
│                    ▼               ▼              ▼        │      │
│              ┌──────────┐   ┌──────────┐   ┌──────────┐   │      │
│              │ Text     │   │ Thinking │   │ Tool Use │   │      │
│              │ blocks   │   │ blocks   │   │ blocks   │   │      │
│              └──────────┘   └──────────┘   └────┬─────┘   │      │
│                                                  │        │      │
│                                                  ▼        │      │
│                                    ┌──────────────────┐   │      │
│                                    │ Tool Execution   │   │      │
│                                    │ (StreamingTool   │   │      │
│                                    │  Executor)       │   │      │
│                                    │                  │   │      │
│                                    │ Permission check │   │      │
│                                    │ → validateInput  │   │      │
│                                    │ → checkPerms     │   │      │
│                                    │ → call()         │   │      │
│                                    └──��─────┬─────────┘   │      │
│                                             │             │      │
│                                             ▼             │      │
│                              tool_results[] ──────────────┘      │
│                              (appended to messages,              │
│                               loop continues)                    │
│                                                                  │
│  LOOP EXITS WHEN:                                                │
│  • No tool_use blocks in response (model is done)                │
│  • maxTurns reached                                              │
│  • Budget exhausted (maxBudgetUsd or taskBudget)                 │
│  • Token blocking limit hit                                      │
│  • Abort signal fired                                            │
│  • Unrecoverable error                                           │
└──────────────────────────────────────────────────────────────────┘
```

### Recovery Mechanisms

The loop handles several failure modes:

| Failure | Recovery | Limit |
|---------|----------|-------|
| `max_output_tokens` | Retry with escalated token limit | 3 attempts |
| `prompt_too_long` | Reactive compact → context collapse drain → error | 1 attempt each |
| Streaming fallback | Tombstone orphaned messages, restart with fallback model | 1 attempt |
| Tool execution error | Return error as tool_result, model retries | No limit |
| Permission denied | Track denial, model receives rejection message | Threshold triggers prompt |

### The Streaming Tool Executor

When `streamingToolExecution` gate is enabled, tools begin executing **while the API is still streaming**:

```
API Stream: [text...] [tool_use A starts] [tool_use A input streams]
                                    │
                                    ▼
                        StreamingToolExecutor
                        ├─ Validates A's input as it arrives
                        ├─ Starts permission check for A
                        └─ Begins executing A before B even starts streaming

API Stream: [tool_use B starts] [tool_use B input] [stream ends]
                       │
                       ▼
           StreamingToolExecutor
           ├─ Queues B
           ├─ Runs B in parallel with A (if A.isConcurrencySafe)
           └─ Collects all results
```

## 5. Tool System Architecture

### Tool Lifecycle

```
Model outputs tool_use block
        │
        ▼
┌─────────────────┐     ┌─────────────────┐
│ 1. Find tool by │────▶│ toolMatchesName  │  Checks name + aliases
│    name/alias   │     └─────────────────┘
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. Parse input  │  Zod schema validation (inputSchema)
│    with schema  │  backfillObservableInput() on clones for SDK/hooks
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 3. validateInput│  Tool-specific: path safety, env checks
│    (optional)   │  Returns ValidationResult { result, message }
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 4. Permission Check Pipeline                     │
│                                                  │
│  a. Deny rules (blanket tool/server deny)        │
│  b. Auto-mode classifier (if auto mode)          │
│  c. PreToolUse hooks                             │
│  d. checkPermissions() (tool-specific)           │
│  e. General permission logic (mode-based)        │
│  f. User prompt (if needed, interactive only)    │
│                                                  │
│  Result: allow | deny | error                    │
└────────┬────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│ 5. call()       │  Execute with ToolUseContext + onProgress callback
│                 │  Returns ToolResult { data, newMessages?, contextModifier? }
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ 6. Result Processing                     │
│                                          │
│  a. mapToolResultToToolResultBlockParam() │
│  b. PostToolUse hooks                    │
│  c. Size check vs maxResultSizeChars     │
│     → Persist to disk if exceeded        │
│  d. renderToolResultMessage() for UI     │
│  e. Append to messages as tool_result    │
└──────────────────────────────────────────┘
```

### Tool Registry Assembly

```
getAllBaseTools()                          ← Source of truth (50+ tools)
       │
       ├─ Always: Agent, Bash, Read, Edit, Write, Notebook, WebFetch,
       │          WebSearch, Todo, AskUser, Skill, ExitPlanMode, Brief, ...
       │
       ├─ Conditional: GlobTool, GrepTool (skip if bfs/ugrep embedded)
       │
       ├─ Feature-flagged: SleepTool, CronTools, WebBrowser, Monitor, ...
       │
       └─ Environment: ConfigTool (ant), TungstenTool (ant), TaskTools (v2), ...
              │
              ▼
       getTools(permissionContext)
              │
              ├─ CLAUDE_CODE_SIMPLE → Bash + Read + Edit only
              ├─ filterToolsByDenyRules() ← Remove blanket-denied tools
              ├─ REPL mode → Hide primitive tools (REPL wraps them)
              └─ isEnabled() check per tool
                     │
                     ▼
              assembleToolPool(permissionContext, mcpTools)
                     │
                     ├─ Built-in tools (sorted alphabetically)
                     ├─ + MCP tools (sorted, filtered by deny rules)
                     ├─ uniqBy('name') ← Built-ins win on conflict
                     └─ Result: stable-ordered tool array
                            │
                            ▼
                     Sent to Anthropic API as tool definitions
                     (order matters for prompt cache hits)
```

## 6. State Architecture

### AppState (Immutable Store)

```typescript
AppState = {
  toolPermissionContext: ToolPermissionContext  // Permission rules + mode
  mcp: {
    tools: Tools                               // MCP-provided tools
    clients: MCPServerConnection[]             // Connected MCP servers
    commands: Command[]                        // MCP-provided commands
  }
  fastMode: FastModeState                      // Fast output toggle
  effortValue: EffortValue                     // Reasoning effort level
  advisorModel: string | null                  // Advisor model override
  fileHistory: FileHistoryState                // File change tracking
  attribution: AttributionState                // Commit attribution
  // ... more fields
}
```

State updates use immutable updater functions:
```typescript
setAppState(prev => ({ ...prev, fileHistory: updater(prev.fileHistory) }))
```

### ToolUseContext (Threaded Through Everything)

This is the "God object" that carries all runtime state through the tool execution pipeline:

```
ToolUseContext
├── options
│   ├── tools: Tools              ← Available tools
│   ├── commands: Command[]       ← Available commands
│   ├── mainLoopModel: string     ← Current model
│   ├── thinkingConfig            ← Thinking mode
│   ├── mcpClients                ← MCP connections
│   ├── maxBudgetUsd              ← Spending limit
│   └── refreshTools()            ← Hot-reload tools
│
├── abortController               ← Cancel signal
├── readFileState: FileStateCache ← LRU file content cache
├── messages: Message[]           ← Conversation history
│
├── State accessors
│   ├── getAppState()             ← Read immutable state
│   ├── setAppState()             ← Update state (no-op for async agents)
│   └── setAppStateForTasks()     ← Always reaches root (for background tasks)
│
├── UI callbacks (REPL only)
│   ├── setToolJSX()              ← Render tool UI
│   ├── addNotification()         ← OS notifications
│   ├── sendOSNotification()      ← iTerm2/Kitty/bell
│   └── openMessageSelector()     ← Message filter UI
│
├── Deduplication sets
│   ├── nestedMemoryAttachmentTriggers
│   ├── loadedNestedMemoryPaths    ← Prevents re-injecting same CLAUDE.md
│   ├── dynamicSkillDirTriggers
│   └── discoveredSkillNames       ← Telemetry
│
├── Content management
│   ├── contentReplacementState    ← Tool result budget tracking
│   └── renderedSystemPrompt       ← Frozen for fork subagents
│
└── Tracking
    ├── queryTracking: { chainId, depth }
    ├── toolDecisions: Map          ← Cached permission decisions
    └── localDenialTracking         ← For async subagents
```

### Subagent Context

When Claude spawns a subagent (via `AgentTool`), the context is cloned with overrides:

```
Parent ToolUseContext
        │
        ▼ createSubagentContext()
        │
Child ToolUseContext
├── setAppState → no-op (async agents don't write to parent state)
├── setAppStateForTasks → parent's real setter (tasks outlive agents)
├── contentReplacementState → cloned (cache-sharing forks need identical decisions)
├── readFileState → cloned (LRU independence)
├── localDenialTracking → new instance (accumulates separately)
├── agentId → new AgentId
└── renderedSystemPrompt → frozen from parent (avoids cache bust)
```

## 7. Command System Architecture

### Loading Pipeline

```
getCommands(cwd)
       │
       ▼
loadAllCommands(cwd)  ← Memoized by cwd
       │
       ├── getSkills(cwd) ← Parallel:
       │   ├── getSkillDirCommands(cwd)  ← ~/.claude/commands/, .claude/commands/
       │   ├── getPluginSkills()          ← From installed plugins
       │   ├── getBundledSkills()         ← Registered at startup
       │   └── getBuiltinPluginSkillCommands()
       │
       ├── getPluginCommands()            ← Plugin-provided commands
       │
       └── getWorkflowCommands(cwd)       ← WORKFLOW_SCRIPTS feature
              │
              ▼
       Assembly order (first wins on name conflict):
       1. bundledSkills
       2. builtinPluginSkills
       3. skillDirCommands
       4. workflowCommands
       5. pluginCommands
       6. pluginSkills
       7. COMMANDS() ← 70+ built-in commands
              │
              ▼
       Filter: meetsAvailabilityRequirement()  ← Auth-state dependent
       Filter: isCommandEnabled()               ← Feature flags
       Insert: getDynamicSkills()               ← Discovered at runtime
```

### Command Execution Flow

```
User types "/compact" or model invokes Skill tool
        │
        ▼
findCommand("compact", commands)  ← Name or alias match
        │
        ▼
Switch on command.type:
        │
        ├── "prompt" → getPromptForCommand() → inject into messages → query()
        │              (Skills, workflows — model processes the expanded text)
        │
        ├── "local" → execute() → return LocalCommandResult
        │              (Cost, compact, status — returns text directly)
        │
        └── "local-jsx" → renderJSX() → Ink component
                           (Config, resume, permissions — interactive UI)
```

## 8. Context Collection Architecture

### What Gets Injected Into Every Conversation

```
System Prompt Assembly
├── Default system prompt (or custom via --system-prompt)
│   ├── Role and behavior instructions
│   ├── Tool usage guidelines
│   ├── Code style and safety rules
│   └── Memory mechanics (if auto-memory configured)
│
├── appendSystemPrompt (via --append-system-prompt)
│
├── System Context (appendSystemContext):
│   ├── gitStatus ← Snapshot at conversation start
│   │   ├── Current branch
│   │   ├── Main branch (for PRs)
│   │   ├── Git user name
│   │   ├── Status (truncated at 2000 chars)
│   │   └── Recent 5 commits
│   │
│   └── cacheBreaker (ant-only debugging)
│
└── User Context (prependUserContext):
    ├── claudeMd ← CLAUDE.md content from directory walk
    │   ├── ./CLAUDE.md
    │   ├── ../CLAUDE.md (parent directories)
    │   ├── ~/.claude/CLAUDE.md (user-level)
    │   └── --add-dir CLAUDE.md files
    │
    └── currentDate ← "Today's date is 2026-03-31."
```

### Attachment System

Beyond the system prompt, per-turn attachments inject additional context:

```
User message submitted
        │
        ▼
getAttachmentMessages()
├── Relevant memory prefetch (sideQuery to find related memories)
├── Skill discovery (which skills might be relevant)
├── Nested memory files (CLAUDE.md from tool-accessed directories)
└── File state attachments (recently read/modified files)
        │
        ▼
filterDuplicateMemoryAttachments()
        │
        ▼
Injected as AttachmentMessage[] before the query
```

## 9. Cost Tracking Architecture

```
API Response received with Usage
        │
        ▼
addToTotalSessionCost(cost, usage, model)
├── addToTotalModelUsage() ← Per-model accumulation
├── addToTotalCostState()  ← Global counters
├── OpenTelemetry counters ← costCounter, tokenCounter
│
└── Recursive: for each advisorUsage in response
    └── addToTotalSessionCost(advisorCost, advisorUsage, advisorModel)
        │
        ▼
On process exit:
saveCurrentSessionCosts()
├── Write to project config:
│   ├── lastCost, lastAPIDuration, lastToolDuration
│   ├── lastLinesAdded, lastLinesRemoved
│   ├── lastModelUsage (per-model breakdown)
│   ├── lastSessionId
│   └── lastFpsAverage, lastFpsLow1Pct
│
On next startup:
├── Log tengu_exit event with previous session's costs
└── restoreCostStateForSession() if --resume matches sessionId
```

## 10. Background Task Architecture

```
AgentTool.call() or TaskCreateTool.call()
        │
        ▼
generateTaskId(type)  ← e.g., "a7k2m9x1" (prefix 'a' = local_agent)
        │
        ▼
createTaskStateBase(id, type, description)
├── status: 'pending'
├── outputFile: getTaskOutputPath(id)  ← Disk file for output
└── startTime: Date.now()
        │
        ▼
setAppState(prev => ({
  ...prev,
  tasks: [...prev.tasks, taskState]
}))
        │
        ▼
Task runs in background:
├── Output written to outputFile
├── Status updates: pending → running → completed/failed/killed
├── setAppStateForTasks() ← Always reaches root store
└── On completion: notified flag set
        │
        ▼
TaskOutputTool.call(taskId)  ← Model reads output from disk
TaskStopTool.call(taskId)    ← Model or user kills task
```

## 11. Session Persistence Architecture

```
Every assistant/user message
        │
        ▼
recordTranscript(messages)
├── Write to session JSONL file
├── --bare mode: fire-and-forget (non-blocking)
└── Normal mode: await (for --resume reliability)

On user prompt submission:
addToHistory(entry)
├── Small pastes (<1024 chars): stored inline in JSONL
├── Large pastes: content-hash → paste store file
└── Images: separate image-cache (not in history)
        │
        ▼
flushPromptHistory()
├── File lock (concurrent sessions)
├── Append to ~/.claude/history.jsonl
└── Retry up to 5 times with 500ms backoff

On --resume:
loadConversationForResume(sessionId)
├── Read session transcript
├── restoreCostStateForSession()
├── Rebuild message array
└── Continue conversation
```

## 12. How It All Connects: A Complete Request

Here's what happens when you type "fix the bug in auth.ts" in the REPL:

```
1. REPL captures input, calls addToHistory()
2. processUserInput() checks for /slash commands → none found
3. createUserMessage() wraps input
4. getAttachmentMessages() → memory prefetch, skill discovery
5. Messages array updated: [...existing, userMessage, ...attachments]
6. recordTranscript() persists to disk

7. query() begins:
   a. Tool result budget applied (large results → disk)
   b. Snip compaction (if over threshold)
   c. Microcompact (cached edits)
   d. Context collapse (if enabled)
   e. Auto-compact (if over threshold)

8. API call: messages + system prompt + tools → Anthropic
   - prependUserContext (CLAUDE.md, date)
   - appendSystemContext (git status)
   - Tool definitions (sorted, stable order)

9. Streaming response arrives:
   - Text blocks → yield to REPL → render
   - Thinking blocks → preserved for trajectory
   - tool_use blocks → StreamingToolExecutor

10. Tool execution (e.g., Read auth.ts):
    a. Find tool by name
    b. validateInput(input)
    c. Permission pipeline → auto-allow (read-only in plan mode)
    d. FileReadTool.call({file_path: "auth.ts"})
    e. Result → tool_result message

11. Loop continues: messages now include tool_result
    → API call with updated context
    → Model responds with Edit tool call

12. Tool execution (Edit auth.ts):
    a. Permission check → user prompted (write operation)
    b. User approves → FileEditTool.call({...})
    c. Result appended, loop continues

13. Model responds with text (no tool_use) → loop exits
14. notifyCommandLifecycle() for any consumed commands
15. Final message rendered in REPL
16. Cost updated: addToTotalSessionCost()
```

---

## Key Architectural Principles

| Principle | How It's Applied |
|-----------|-----------------|
| **Fail closed** | Tool defaults assume writes, non-concurrent, non-destructive |
| **Cache stability** | Tool ordering, file naming, and settings paths designed for API cache hits |
| **Parallel everything** | Startup prefetching, Promise.all for independent ops, streaming tool execution |
| **Deferred work** | Heavy operations run after first render, while user types |
| **Composable streaming** | AsyncGenerators with yield* delegation for layered processing |
| **Defense in depth** | 3-layer permissions, trust gating, sandbox detection, anti-debugging |
| **Single-codebase builds** | Feature flags tree-shake entire subsystems from external builds |
| **Immutable state** | DeepImmutable types, updater functions, no direct mutation |
