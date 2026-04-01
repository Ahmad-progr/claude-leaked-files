# Claude Code: Startup and Initialization

## Startup Sequence

Claude Code employs an aggressive startup optimization strategy. The `main.tsx` entrypoint is carefully ordered to maximize parallelism and minimize time-to-first-render.

### Phase 1: Side-Effect Imports (Before All Other Imports)

```typescript
// 1. Mark entry time for profiling
profileCheckpoint('main_tsx_entry');

// 2. Fire MDM (Mobile Device Management) subprocess reads
//    These run plutil/reg queries in parallel with subsequent imports
startMdmRawRead();

// 3. Fire macOS keychain prefetch (OAuth + legacy API key)
//    Both reads happen in parallel instead of sequential
startKeychainPrefetch();
```

These three side-effects execute **before** the remaining ~200 import statements are evaluated. Since module evaluation in Bun takes ~135ms, the subprocess calls get a significant head start.

### Phase 2: Heavy Imports

After the side-effect imports, `main.tsx` loads:
- Commander.js for CLI argument parsing
- React and Ink for terminal UI
- All utility modules, services, and configurations

A profiling checkpoint marks when imports complete:
```typescript
profileCheckpoint('main_tsx_imports_loaded');
```

### Phase 3: Migrations

Claude Code runs synchronous migrations on startup, versioned with `CURRENT_MIGRATION_VERSION`:

```typescript
const CURRENT_MIGRATION_VERSION = 11;
function runMigrations(): void {
  if (getGlobalConfig().migrationVersion !== CURRENT_MIGRATION_VERSION) {
    migrateAutoUpdatesToSettings();
    migrateBypassPermissionsAcceptedToSettings();
    migrateSonnet45ToSonnet46();
    migrateOpusToOpus1m();
    // ... more migrations
    saveGlobalConfig(prev => ({
      ...prev,
      migrationVersion: CURRENT_MIGRATION_VERSION
    }));
  }
}
```

### Phase 4: Setup (`setup.ts`)

The `setup()` function handles:

1. **Node.js version check**: Requires Node.js 18+
2. **Session ID**: Sets custom session ID if provided
3. **UDS Messaging**: Starts Unix Domain Socket server for inter-process communication
4. **Terminal backup restoration**: Checks for interrupted iTerm2/Terminal.app setups
5. **Working directory**: Sets `cwd` (must happen before hooks)
6. **Hooks snapshot**: Captures hooks configuration to detect modifications
7. **File watcher**: Initializes `FileChanged` hook watcher
8. **Worktree creation**: If `--worktree` flag is used, creates git worktree + optional tmux session
9. **Background services**: Session memory, context collapse, version locking
10. **Command prefetch**: Pre-loads commands and plugin hooks
11. **Permission validation**: Validates `--dangerously-skip-permissions` is in safe environment
12. **Telemetry**: Emits `tengu_started` event

### Phase 5: Deferred Prefetches (After First Render)

`startDeferredPrefetches()` runs **after** the REPL has been rendered to avoid blocking the initial paint:

```typescript
export function startDeferredPrefetches(): void {
  // Skip in benchmark mode or --bare mode
  if (isEnvTruthy(process.env.CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER) || isBareMode()) {
    return;
  }

  // Process-spawning prefetches (user is still typing)
  void initUser();
  void getUserContext();
  prefetchSystemContextIfSafe();
  void getRelevantTips();

  // Cloud provider credential prefetch
  void prefetchAwsCredentialsAndBedRockInfoIfSafe();
  void prefetchGcpCredentialsIfSafe();

  // File counting for analytics
  void countFilesRoundedRg(getCwd(), AbortSignal.timeout(3000), []);

  // Feature flag and model capability refresh
  void initializeAnalyticsGates();
  void refreshModelCapabilities();

  // File change detectors
  void settingsChangeDetector.initialize();
  void skillChangeDetector.initialize();
}
```

### Trust-Gated Prefetching

Git commands can execute arbitrary code via hooks (e.g., `core.fsmonitor`, `diff.external`), so system context prefetching is trust-gated:

```typescript
function prefetchSystemContextIfSafe(): void {
  // Non-interactive: trust is implicit
  if (isNonInteractiveSession) {
    void getSystemContext();
    return;
  }
  // Interactive: only if trust dialog was already accepted
  if (checkHasTrustDialogAccepted()) {
    void getSystemContext();
  }
  // Otherwise: wait for trust to be established
}
```

## Anti-Debugging Protection

External (non-Anthropic) builds include a debugger detection guard:

```typescript
if ("external" !== 'ant' && isBeingDebugged()) {
  process.exit(1);
}
```

This checks for:
- `--inspect` / `--inspect-brk` flags in `process.execArgv`
- `--inspect` in `NODE_OPTIONS` environment variable
- Active inspector URL via Node.js inspector module

## Bare Mode (`--bare`)

The `--bare` flag is a lightweight mode for scripted calls that skips:
- All deferred prefetches
- Plugin loading and hooks
- Release notes and onboarding
- Settings/skill change detectors
- File counting and analytics
- Team memory sync
- Attribution hooks

This significantly reduces startup overhead for CI/CD usage.

## Session Persistence

Sessions are identified by a UUID and persist across process restarts:
- Cost data is saved to project config on exit
- History entries are written to `~/.claude/history.jsonl`
- Session transcripts are stored for `/resume` functionality
