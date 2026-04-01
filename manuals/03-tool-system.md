# Claude Code: Tool System

## Overview

The tool system is Claude Code's primary extensibility mechanism. It defines a rich plugin interface (`Tool`) that enables Claude to interact with the filesystem, shell, web, MCP servers, and more.

## Tool Interface (`Tool.ts`)

Every tool implements a comprehensive interface with ~40+ fields and methods. The key ones:

### Core Methods

| Method | Purpose |
|--------|---------|
| `call()` | Execute the tool with given input, context, and progress callback |
| `description()` | Generate the tool's description for the LLM prompt |
| `prompt()` | Generate the tool's usage instructions for the system prompt |
| `checkPermissions()` | Tool-specific permission logic (after general permission checks) |
| `validateInput()` | Input validation before execution |

### Metadata Methods

| Method | Purpose |
|--------|---------|
| `isEnabled()` | Whether the tool is available in the current environment |
| `isReadOnly(input)` | Whether this invocation only reads (no writes) |
| `isDestructive(input)` | Whether this invocation performs irreversible operations |
| `isConcurrencySafe(input)` | Whether multiple instances can run in parallel |
| `interruptBehavior()` | `'cancel'` or `'block'` when user sends new message |

### Rendering Methods

| Method | Purpose |
|--------|---------|
| `renderToolUseMessage()` | Render the tool invocation in the UI |
| `renderToolResultMessage()` | Render the tool result in the UI |
| `renderToolUseProgressMessage()` | Render progress while tool is running |
| `renderGroupedToolUse()` | Render multiple parallel instances as a group |
| `userFacingName(input)` | Human-readable name for UI display |
| `getActivityDescription(input)` | Present-tense description for spinner |

### Schema

```typescript
readonly inputSchema: Input       // Zod schema for input validation
readonly inputJSONSchema?: ToolInputJSONSchema  // JSON Schema for MCP tools
outputSchema?: z.ZodType<unknown> // Optional output schema
```

## The `buildTool()` Factory

All tools are constructed via `buildTool()`, which provides fail-closed defaults:

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,    // Assume NOT safe
  isReadOnly: (_input?: unknown) => false,            // Assume writes
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (input) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
  userFacingName: (_input?: unknown) => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

**Key design decision**: `isConcurrencySafe` defaults to `false` and `isReadOnly` defaults to `false`. This means new tools are conservatively treated as write-capable and non-concurrent unless explicitly marked otherwise.

## Tool Result Size Management

Tools define a `maxResultSizeChars` that controls when output gets persisted to disk:

```typescript
maxResultSizeChars: number
// When exceeded, result is saved to a file and Claude receives
// a preview with the file path instead of the full content.
// Set to Infinity for tools whose output must never be persisted.
```

## Tool Registry (`tools.ts`)

### Built-in Tools

The `getAllBaseTools()` function returns the exhaustive list of all available tools:

**Always available:**
- `AgentTool` - Spawn subagent for complex tasks
- `BashTool` - Execute shell commands
- `FileReadTool` - Read files
- `FileEditTool` - Edit files with string replacement
- `FileWriteTool` - Create/overwrite files
- `NotebookEditTool` - Edit Jupyter notebooks
- `WebFetchTool` - Fetch web pages
- `WebSearchTool` - Search the web
- `TodoWriteTool` - Manage task lists
- `AskUserQuestionTool` - Ask user for input
- `SkillTool` - Invoke slash commands/skills
- `EnterPlanModeTool` - Enter planning mode
- `ExitPlanModeV2Tool` - Exit planning mode
- `TaskOutputTool` - Read background task output
- `TaskStopTool` - Stop background tasks
- `BriefTool` - Send brief messages
- `SendMessageTool` - Send messages to other agents

**Conditionally available (embedded tools check):**
- `GlobTool` - File pattern matching (skipped when bfs/ugrep embedded)
- `GrepTool` - Content search (skipped when bfs/ugrep embedded)

**Feature-flag gated:**
- `SleepTool` (PROACTIVE/KAIROS)
- `CronCreate/CronDelete/CronListTool` (AGENT_TRIGGERS)
- `RemoteTriggerTool` (AGENT_TRIGGERS_REMOTE)
- `MonitorTool` (MONITOR_TOOL)
- `SendUserFileTool` (KAIROS)
- `PushNotificationTool` (KAIROS/KAIROS_PUSH_NOTIFICATION)
- `SubscribePRTool` (KAIROS_GITHUB_WEBHOOKS)
- `WebBrowserTool` (WEB_BROWSER_TOOL)
- `OverflowTestTool` (OVERFLOW_TEST_TOOL)
- `CtxInspectTool` (CONTEXT_COLLAPSE)
- `TerminalCaptureTool` (TERMINAL_PANEL)
- `SnipTool` (HISTORY_SNIP)
- `ListPeersTool` (UDS_INBOX)
- `WorkflowTool` (WORKFLOW_SCRIPTS)

**Environment-gated:**
- `ConfigTool` (ant-only)
- `TungstenTool` (ant-only)
- `REPLTool` (ant-only)
- `SuggestBackgroundPRTool` (ant-only)
- `TaskCreate/Get/Update/ListTool` (todo v2 enabled)
- `EnterWorktreeTool/ExitWorktreeTool` (worktree mode)
- `LSPTool` (ENABLE_LSP_TOOL env)
- `PowerShellTool` (Windows PowerShell enabled)
- `TestingPermissionTool` (test environment only)
- `ToolSearchTool` (tool search enabled)

### Tool Assembly

The full tool pool is assembled by `assembleToolPool()`:

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // Sort each partition for prompt-cache stability
  // Built-ins as contiguous prefix, MCP tools after
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

**Important**: Tool ordering is stable and sorted by name to maximize Anthropic API prompt cache hit rates. Built-in tools form a contiguous prefix.

### Tool Filtering

Tools are filtered at multiple levels:

1. **Feature flags**: Compile-time elimination via `bun:bundle` feature flags
2. **Environment checks**: `process.env.USER_TYPE`, `process.env.NODE_ENV`
3. **`isEnabled()`**: Runtime checks per tool
4. **Deny rules**: `filterToolsByDenyRules()` removes blanket-denied tools
5. **REPL mode**: When REPL tool is enabled, primitive tools are hidden
6. **Simple mode**: `CLAUDE_CODE_SIMPLE` reduces to Bash + Read + Edit only

### Deferred Tool Loading (ToolSearch)

When many tools are available, Claude Code uses **deferred loading** to keep the initial prompt small:

```typescript
readonly shouldDefer?: boolean    // Tool is deferred (needs ToolSearch first)
readonly alwaysLoad?: boolean     // Tool is never deferred
searchHint?: string               // Keywords for ToolSearch matching
```

Deferred tools are sent with `defer_loading: true` and require the model to use `ToolSearch` before calling them.

## Permission System

### Three-Layer Permission Check

1. **`validateInput()`**: Tool-specific input validation (e.g., path safety)
2. **General permission logic** (`permissions.ts`): Mode-based rules (default, plan, auto, bypass)
3. **`checkPermissions()`**: Tool-specific permission logic

### Permission Modes

```typescript
type PermissionMode =
  | 'default'        // Ask for each tool use
  | 'plan'           // Read-only tools auto-allowed
  | 'auto'           // AI classifier decides
  | 'bypassPermissions'  // All tools auto-allowed (sandbox only)
```

### Permission Context

```typescript
type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  shouldAvoidPermissionPrompts?: boolean   // For background agents
  awaitAutomatedChecksBeforeDialog?: boolean  // For coordinator workers
}>
```

The `DeepImmutable` wrapper prevents accidental mutation of security-critical configuration.

## MCP (Model Context Protocol) Tools

MCP tools are loaded from external servers and integrated via:

```typescript
mcpInfo?: { serverName: string; toolName: string }
isMcp?: boolean
```

They share the same permission system and can be filtered by deny rules at the server level (e.g., `mcp__server` denies all tools from that server).
