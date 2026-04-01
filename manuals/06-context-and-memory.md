# Claude Code: Context and Memory System

## Overview

Claude Code injects rich contextual information into every conversation — git status, CLAUDE.md project instructions, memory files, and more. This context is memoized per session and cache-clearable when the underlying data changes.

## System Context (`context.ts`)

### Git Status

The `getGitStatus()` function collects a snapshot of the repository state at conversation start:

```typescript
export const getGitStatus = memoize(async (): Promise<string | null> => {
  const isGit = await getIsGit()
  if (!isGit) return null

  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short'], ...),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5'], ...),
    execFileNoThrow(gitExe(), ['config', 'user.name'], ...),
  ])

  // Truncate status at 2000 chars to avoid overwhelming context
  const truncatedStatus = status.length > MAX_STATUS_CHARS
    ? status.substring(0, MAX_STATUS_CHARS) + '\n... (truncated)'
    : status

  return [
    'This is the git status at the start of the conversation...',
    `Current branch: ${branch}`,
    `Main branch: ${mainBranch}`,
    ...(userName ? [`Git user: ${userName}`] : []),
    `Status:\n${truncatedStatus || '(clean)'}`,
    `Recent commits:\n${log}`,
  ].join('\n\n')
})
```

**Key details:**
- Uses `--no-optional-locks` to avoid interfering with other git operations
- All five git commands run in **parallel** via `Promise.all`
- Status is truncated at 2000 characters to keep context manageable
- Memoized for the duration of the session (snapshot in time)

### System Context

```typescript
export const getSystemContext = memoize(async (): Promise<{ [k: string]: string }> => {
  const gitStatus = isRemote || !shouldIncludeGitInstructions()
    ? null
    : await getGitStatus()

  return {
    ...(gitStatus && { gitStatus }),
    // Cache breaker injection (ant-only debugging)
  }
})
```

### User Context

```typescript
export const getUserContext = memoize(async (): Promise<{ [k: string]: string }> => {
  const shouldDisableClaudeMd =
    isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
    (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)

  const claudeMd = shouldDisableClaudeMd
    ? null
    : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

  // Cache for auto-mode classifier
  setCachedClaudeMdContent(claudeMd || null)

  return {
    ...(claudeMd && { claudeMd }),
    currentDate: `Today's date is ${getLocalISODate()}.`,
  }
})
```

**CLAUDE.md discovery rules:**
- `CLAUDE_CODE_DISABLE_CLAUDE_MDS`: Hard off, always
- `--bare` mode: Skip auto-discovery (cwd walk), BUT honor explicit `--add-dir`
- Otherwise: Walk directory tree to find all CLAUDE.md files

### Cache Invalidation

Both `getSystemContext` and `getUserContext` are memoized with clearable caches:

```typescript
export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}
```

## CLAUDE.md Files

CLAUDE.md is the primary way users provide project-specific instructions to Claude Code. The system discovers these files by walking the directory tree:

- `CLAUDE.md` in the current working directory
- `CLAUDE.md` in parent directories
- `CLAUDE.md` in additional directories specified via `--add-dir`

### Memory Files

Memory files are a special subset of CLAUDE.md content:

```typescript
const claudeMd = getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))
```

Memory files are filtered before CLAUDE.md assembly to avoid duplication with the auto-memory system.

### Nested Memory Attachments

The system tracks which CLAUDE.md paths have been injected as nested memory attachments to avoid re-injection:

```typescript
loadedNestedMemoryPaths?: Set<string>
// Dedup for memoryFilesToAttachments — readFileState is an LRU
// that evicts entries in busy sessions, so its .has() check alone
// can re-inject the same CLAUDE.md dozens of times.
```

## Project Onboarding (`projectOnboardingState.ts`)

New projects get onboarding steps:

```typescript
export function getSteps(): Step[] {
  const hasClaudeMd = existsSync(join(getCwd(), 'CLAUDE.md'))
  const isWorkspaceDirEmpty = isDirEmpty(getCwd())

  return [
    {
      key: 'workspace',
      text: 'Ask Claude to create a new app or clone a repository',
      isComplete: false,
      isCompletable: true,
      isEnabled: isWorkspaceDirEmpty,
    },
    {
      key: 'claudemd',
      text: 'Run /init to create a CLAUDE.md file with instructions',
      isComplete: hasClaudeMd,
      isCompletable: true,
      isEnabled: !isWorkspaceDirEmpty,
    },
  ]
}
```

Onboarding is shown up to 4 times, then auto-dismissed:

```typescript
export const shouldShowProjectOnboarding = memoize((): boolean => {
  if (
    projectConfig.hasCompletedProjectOnboarding ||
    projectConfig.projectOnboardingSeenCount >= 4 ||
    process.env.IS_DEMO
  ) return false
  return !isProjectOnboardingComplete()
})
```

## Prompt History (`history.ts`)

### Storage Format

History is stored as JSONL (JSON Lines) in `~/.claude/history.jsonl`:

```typescript
type LogEntry = {
  display: string                              // What the user typed
  pastedContents: Record<number, StoredPastedContent>  // Pasted text/images
  timestamp: number
  project: string                              // Project root path
  sessionId?: string
}
```

### Paste Content Storage

Large pasted content is stored externally via content-addressable hashing:

```typescript
if (content.content.length <= MAX_PASTED_CONTENT_LENGTH) {  // 1024 chars
  // Store inline
  storedPastedContents[id] = { id, type, content: content.content }
} else {
  // Store in paste store with content hash
  const hash = hashPastedText(content.content)
  storedPastedContents[id] = { id, type, contentHash: hash }
  void storePastedText(hash, content.content)  // Fire-and-forget
}
```

### Reference System

Pasted content is referenced in prompts as:
- Text: `[Pasted text #1 +10 lines]`
- Image: `[Image #2]`

These are expanded back to full content via `expandPastedTextRefs()`.

### History Reading

History is read in reverse order (newest first) with session-aware ordering:

```typescript
export async function* getHistory(): AsyncGenerator<HistoryEntry> {
  const otherSessionEntries: LogEntry[] = []

  for await (const entry of makeLogEntryReader()) {
    if (entry.sessionId === currentSession) {
      yield await logEntryToHistoryEntry(entry)  // Current session first
    } else {
      otherSessionEntries.push(entry)            // Buffer other sessions
    }
  }
  // Then yield other session entries
  for (const entry of otherSessionEntries) {
    yield await logEntryToHistoryEntry(entry)
  }
}
```

This ensures **current session entries appear first** when pressing Up arrow, preventing interleaving from concurrent sessions.

### Write Buffering

History writes are buffered and flushed asynchronously:

```typescript
let pendingEntries: LogEntry[] = []
let isWriting = false

async function flushPromptHistory(retries: number): Promise<void> {
  if (isWriting || pendingEntries.length === 0) return
  if (retries > 5) return  // Stop after 5 retries
  isWriting = true
  try {
    await immediateFlushHistory()
  } finally {
    isWriting = false
    if (pendingEntries.length > 0) {
      await sleep(500)           // Avoid hot loop
      void flushPromptHistory(retries + 1)
    }
  }
}
```

File writes use file locking (`lock()`) to handle concurrent sessions:

```typescript
release = await lock(historyPath, {
  stale: 10000,
  retries: { retries: 3, minTimeout: 50 },
})
await appendFile(historyPath, jsonLines.join(''), { mode: 0o600 })
```

### Undo Support

The `removeLastFromHistory()` function supports undoing the most recent history entry (used when Esc rewinds a conversation):

```typescript
export function removeLastFromHistory(): void {
  if (!lastAddedEntry) return
  const entry = lastAddedEntry
  lastAddedEntry = null

  const idx = pendingEntries.lastIndexOf(entry)
  if (idx !== -1) {
    pendingEntries.splice(idx, 1)    // Fast path: still in buffer
  } else {
    skippedTimestamps.add(entry.timestamp)  // Slow path: already flushed
  }
}
```

### Cleanup Registration

A cleanup handler ensures all pending entries are flushed on process exit:

```typescript
registerCleanup(async () => {
  if (currentFlushPromise) await currentFlushPromise
  if (pendingEntries.length > 0) await immediateFlushHistory()
})
```
