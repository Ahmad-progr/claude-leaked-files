# Claude Code: Query Engine and Loop

## Overview

The query system is the heart of Claude Code. It manages the conversation lifecycle, streaming responses, tool execution, context management, and error recovery. The system is built on **AsyncGenerators**, providing a clean, composable streaming architecture.

## QueryEngine (`QueryEngine.ts`)

`QueryEngine` is a class that owns the conversation state for one session. It's used by the SDK/headless path and can manage multiple turns within the same conversation.

### Construction

```typescript
class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private readFileState: FileStateCache
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()
}
```

### Configuration

```typescript
type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  verbose?: boolean
  replayUserMessages?: boolean
}
```

### `submitMessage()` — The Entry Point

Each user message goes through `submitMessage()`, which is an AsyncGenerator:

```typescript
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown>
```

**Key steps in submitMessage:**

1. **Clear per-turn state**: Resets skill discovery names
2. **Set working directory**: `setCwd(cwd)`
3. **Wrap permission tracking**: Decorates `canUseTool` to track denials
4. **Resolve model and thinking config**: Uses user-specified or defaults
5. **Fetch system prompt parts**: System prompt, user context, system context
6. **Inject memory mechanics**: If custom prompt + memory path override
7. **Build system prompt**: Concatenate custom/default + memory + append
8. **Register structured output**: If JSON schema + synthetic output tool
9. **Process user input**: Handle slash commands, attachments
10. **Execute query loop**: Yields streaming events

### Permission Denial Tracking

```typescript
const wrappedCanUseTool: CanUseToolFn = async (tool, input, ...) => {
  const result = await canUseTool(tool, input, ...)
  if (result.behavior !== 'allow') {
    this.permissionDenials.push({
      tool_name: sdkCompatToolName(tool.name),
      tool_use_id: toolUseID,
      tool_input: input,
    })
  }
  return result
}
```

This decorator pattern adds SDK-level reporting without modifying the permission system itself.

## Query Loop (`query.ts`)

The `query()` function is the core streaming loop that handles API calls, tool execution, and recovery. It's implemented as a nested pair of AsyncGenerators.

### Architecture

```typescript
export async function* query(params: QueryParams): AsyncGenerator<...> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // Notify consumed commands on normal completion
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

### Loop State

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined  // Why previous iteration continued
}
```

### Per-Iteration Pipeline

Each iteration of the `while (true)` loop runs this pipeline:

```
1. Skill Discovery Prefetch (async, runs during streaming)
     |
2. yield { type: 'stream_request_start' }
     |
3. Query Chain Tracking (chainId + depth)
     |
4. Tool Result Budget Enforcement (persist large results to disk)
     |
5. Snip Compaction (HISTORY_SNIP feature)
     |
6. Microcompact (cached cache-editing compaction)
     |
7. Context Collapse (CONTEXT_COLLAPSE feature)
     |
8. System Prompt Assembly (appendSystemContext)
     |
9. Auto-Compact (threshold-based conversation summarization)
     |
10. API Call (streaming response from Anthropic)
     |
11. Tool Execution (StreamingToolExecutor)
     |
12. Recovery / Continue Decision
```

### Context Management Layers

Claude Code employs **five layers** of context management, composing cleanly in sequence:

1. **Tool Result Budget**: Large tool results are persisted to disk with previews sent to the model. Tools with `maxResultSizeChars: Infinity` are exempt.

2. **Snip Compaction** (`HISTORY_SNIP`): Removes old messages that exceed a token threshold, replacing them with boundary markers.

3. **Microcompact**: Fine-grained compaction using cached cache-editing. Defers boundary messages until after API response for accurate token accounting.

4. **Context Collapse** (`CONTEXT_COLLAPSE`): A read-time projection over full history. Summary messages live in a separate store, not the REPL array. Persists across turns.

5. **Auto-Compact**: Threshold-based full conversation summarization. Runs last so earlier layers can reduce context enough to avoid it.

### Token Budget Tracking

```typescript
const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null
```

The task budget tracks remaining tokens across compaction boundaries:
- Before compaction, the server can see full history
- After compaction, `taskBudgetRemaining` tells the server what was summarized away

### Max Output Tokens Recovery

When the model hits `max_output_tokens`, the loop can retry up to 3 times:

```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3
```

Withheld errors are not yielded to SDK callers until recovery is exhausted, preventing premature session termination.

### Thinking Rules

A detailed comment in query.ts documents the "rules of thinking" for the API:

1. A message with `thinking` or `redacted_thinking` blocks must be in a query with `max_thinking_length > 0`
2. A thinking block may not be the last message in a block
3. Thinking blocks must be preserved for the duration of an assistant trajectory

## ToolUseContext

The context object threaded through all tool calls:

```typescript
type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mainLoopModel: string
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
    refreshTools?: () => Tools
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  messages: Message[]
  fileReadingLimits?: { maxTokens?: number; maxSizeBytes?: number }
  globLimits?: { maxResults?: number }
  queryTracking?: QueryChainTracking
  contentReplacementState?: ContentReplacementState
  // ... many more fields
}
```

### Notable Fields

- **`setAppStateForTasks`**: Always-shared state setter for session-scoped infrastructure (background tasks, hooks). Unlike `setAppState`, this always reaches the root store even from nested subagents.
- **`handleElicitation`**: Handles URL elicitations from MCP tool errors (`-32042`)
- **`nestedMemoryAttachmentTriggers`**: Deduplication set for CLAUDE.md injection
- **`contentReplacementState`**: Per-thread content replacement state for tool result budget
- **`renderedSystemPrompt`**: Frozen parent prompt for fork subagents (avoids cache-busting)

## Query Chain Tracking

Each iteration increments the query chain depth:

```typescript
const queryTracking = toolUseContext.queryTracking
  ? { chainId: toolUseContext.queryTracking.chainId, depth: depth + 1 }
  : { chainId: deps.uuid(), depth: 0 }
```

This enables analytics tracking across multi-turn tool-use chains.

## Feature-Gated Imports

The query module uses dead-code-elimination patterns for feature-gated functionality:

```typescript
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? require('./services/compact/reactiveCompact.js')
  : null

const contextCollapse = feature('CONTEXT_COLLAPSE')
  ? require('./services/contextCollapse/index.js')
  : null

const snipModule = feature('HISTORY_SNIP')
  ? require('./services/compact/snipCompact.js')
  : null
```

This ensures excluded features are completely tree-shaken from external builds.
