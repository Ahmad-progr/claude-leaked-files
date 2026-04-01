# Claude Code: Design Patterns and Lessons

## Overview

This document distills the most valuable engineering patterns found in Claude Code's source — techniques that are broadly applicable to any production TypeScript/CLI application.

## 1. Side-Effect Import Ordering for Startup Optimization

**Pattern**: Place side-effect imports that spawn subprocesses at the very top of the entrypoint, before all other imports.

```typescript
// These three lines run BEFORE 200+ import statements are evaluated
profileCheckpoint('main_tsx_entry');
startMdmRawRead();       // Fires plutil/reg query subprocesses
startKeychainPrefetch();  // Fires macOS keychain reads

// ~135ms of module evaluation happens while those subprocesses run
import { Command } from '@commander-js/extra-typings';
import chalk from 'chalk';
// ... 200 more imports
```

**Why it matters**: Import evaluation is synchronous and can take 100ms+. By spawning async work first, you get free parallelism with the import phase.

**Lint guard**: `biome-ignore-all assist/source/organizeImports` prevents auto-sorters from breaking the intentional ordering.

## 2. AsyncGenerator for Streaming Agent Loops

**Pattern**: Use `AsyncGenerator` with `yield*` delegation for composable streaming loops.

```typescript
export async function* query(params: QueryParams): AsyncGenerator<StreamEvent | Message> {
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // Cleanup only on normal completion
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

**Benefits**:
- Natural streaming: consumers process events as they arrive
- Composable: inner generators can be wrapped with outer behavior
- Cancellable: `.return()` closes the entire chain
- Error propagation: exceptions bubble through `yield*`

## 3. Fail-Closed Defaults with Builder Pattern

**Pattern**: Use a factory function that applies conservative defaults, requiring explicit opt-in for dangerous capabilities.

```typescript
const TOOL_DEFAULTS = {
  isConcurrencySafe: () => false,  // Assume NOT safe
  isReadOnly: () => false,         // Assume writes
  isDestructive: () => false,
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, userFacingName: () => def.name, ...def }
}
```

**Why it matters**: New tools are automatically restricted. A developer must explicitly say `isConcurrencySafe: () => true` — they can't accidentally forget to declare it.

## 4. Memoized Async with Clearable Cache

**Pattern**: Use lodash `memoize` for expensive async computations with explicit cache invalidation.

```typescript
export const getUserContext = memoize(async () => {
  // Expensive: walks directory tree, reads files
  const claudeMd = getClaudeMds(await getMemoryFiles())
  return { claudeMd, currentDate: getLocalISODate() }
})

// Elsewhere, when the data changes:
getUserContext.cache.clear?.()
```

**Benefits**:
- First call does the work; subsequent calls are instant
- Cache can be cleared when underlying data changes
- The `?.` on `clear` handles memoize implementations without `clear`

## 5. Dead Code Elimination via Feature Flags

**Pattern**: Use compile-time feature flags with conditional `require()` for tree-shakeable builds.

```typescript
import { feature } from 'bun:bundle'

const SleepTool = feature('PROACTIVE')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

**Why `require()` and not `import()`**: `require()` is synchronous and can be evaluated at module scope. The bundler sees `feature()` as a constant and eliminates the dead branch entirely, including all transitively imported modules.

## 6. Context Object Threading

**Pattern**: Thread a mutable context object through deep call chains instead of using global state.

```typescript
type ToolUseContext = {
  options: { tools: Tools; mainLoopModel: string; ... }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  messages: Message[]
  // ... 30+ more fields
}
```

**Benefits**:
- Different execution modes (REPL vs SDK) can provide different implementations
- Subagents can override specific fields while inheriting the rest
- Testable: inject mock context without touching global state

## 7. Decorator Pattern for Cross-Cutting Concerns

**Pattern**: Wrap functions to add behavior without modifying the original.

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

This adds SDK-level denial tracking without modifying the permission system.

## 8. Branded Types as Code Review Guardrails

**Pattern**: Use descriptive type names that force developers to acknowledge safety properties.

```typescript
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = string

logEvent('tengu_startup', {
  gh_auth_status: ghAuthStatus as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
})
```

The type name itself is the review signal. Every `as` cast is a deliberate assertion: "I verified this doesn't contain PII."

## 9. Content-Addressable Hashing for Cache Stability

**Pattern**: Use content hashes instead of random IDs for file paths that end up in API prompts.

```typescript
// BAD: Random UUID per process busts API prompt cache
settingsPath = generateTempFilePath('claude-settings', '.json', {
  contentHash: undefined  // Would use random UUID
})

// GOOD: Same content = same path = cache hit
settingsPath = generateTempFilePath('claude-settings', '.json', {
  contentHash: trimmedSettings
})
```

**Why it matters**: The settings path ends up in tool descriptions sent to the API. A random UUID changes the prompt on every subprocess, invalidating the cache and causing 12x token costs.

## 10. Sorted Tool Ordering for Prompt Cache Hits

**Pattern**: Sort tools alphabetically and keep built-ins as a contiguous prefix.

```typescript
const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
return uniqBy(
  [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
  'name',
)
```

**Why it matters**: The API caches prompt prefixes. If tool ordering changes between requests (e.g., because an MCP tool name sorts between built-ins), the entire cache is invalidated.

## 11. Parallel Promise.all for Independent Operations

**Pattern**: Always use `Promise.all` for independent async operations.

```typescript
const [branch, mainBranch, status, log, userName] = await Promise.all([
  getBranch(),
  getDefaultBranch(),
  execFileNoThrow(gitExe(), ['status', '--short'], ...),
  execFileNoThrow(gitExe(), ['log', '--oneline', '-n', '5'], ...),
  execFileNoThrow(gitExe(), ['config', 'user.name'], ...),
])
```

Five git commands run in parallel instead of sequentially. This pattern appears throughout the codebase.

## 12. Deferred Work Until After First Render

**Pattern**: Split initialization into "before render" (minimal) and "after render" (everything else).

```typescript
// Before render: only what's needed for the UI to display
await setup(cwd, permissionMode, ...)

// After render: everything else
export function startDeferredPrefetches(): void {
  void initUser()
  void getUserContext()
  void getRelevantTips()
  void refreshModelCapabilities()
  // ...
}
```

**Why it matters**: Users perceive startup time as time-to-first-render. Background work that completes while the user is typing their first prompt is effectively free.

## 13. Recursive Cost Tracking

**Pattern**: Handle nested resource usage with recursive accumulation.

```typescript
export function addToTotalSessionCost(cost, usage, model): number {
  addToTotalModelUsage(cost, usage, model)
  let totalCost = cost
  for (const advisorUsage of getAdvisorUsage(usage)) {
    const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
    totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
  }
  return totalCost
}
```

Advisor models (nested model calls within responses) are tracked by recursively calling the same function.

## 14. Lazy Module Loading for Heavy Dependencies

**Pattern**: Use `() => require()` or dynamic `import()` for expensive modules.

```typescript
// 113KB module, only loaded on actual invocation
const usageReport: Command = {
  type: 'prompt',
  name: 'insights',
  async getPromptForCommand(args) {
    const real = (await import('./commands/insights.js')).default
    return real.getPromptForCommand(args)
  },
}
```

Also used to break circular dependencies:

```typescript
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
```

## 15. Write Buffering with File Locking

**Pattern**: Buffer writes, flush asynchronously, use file locks for concurrent access.

```typescript
let pendingEntries: LogEntry[] = []

async function immediateFlushHistory(): Promise<void> {
  const release = await lock(historyPath, { stale: 10000 })
  try {
    const jsonLines = pendingEntries.map(entry => jsonStringify(entry) + '\n')
    pendingEntries = []
    await appendFile(historyPath, jsonLines.join(''), { mode: 0o600 })
  } finally {
    await release()
  }
}
```

This handles multiple concurrent Claude Code sessions writing to the same history file without corruption.
