# Claude Code: Feature Flags and Build System

## Overview

Claude Code uses a compile-time feature flag system (`bun:bundle`) that enables dead code elimination. This allows shipping different builds (internal vs. external) from a single codebase, with entire modules tree-shaken from builds where they're not needed.

## Feature Flag System

### Usage Pattern

```typescript
import { feature } from 'bun:bundle'

// Conditional require — tree-shaken when flag is false
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

// Guard usage
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
}
```

### Known Feature Flags

| Flag | Purpose |
|------|---------|
| `PROACTIVE` | Proactive behavior (sleep tool, proactive commands) |
| `KAIROS` | Assistant mode, push notifications, file sending |
| `KAIROS_BRIEF` | Brief command |
| `KAIROS_PUSH_NOTIFICATION` | Push notification tool |
| `KAIROS_GITHUB_WEBHOOKS` | PR subscription webhooks |
| `BRIDGE_MODE` | Remote control bridge (mobile/web) |
| `DAEMON` | Remote control server daemon |
| `VOICE_MODE` | Voice input mode |
| `HISTORY_SNIP` | History snip compaction |
| `WORKFLOW_SCRIPTS` | Workflow automation scripts |
| `AGENT_TRIGGERS` | Scheduled agent cron jobs |
| `AGENT_TRIGGERS_REMOTE` | Remote trigger execution |
| `MONITOR_TOOL` | MCP server monitor |
| `WEB_BROWSER_TOOL` | Web browser tool |
| `OVERFLOW_TEST_TOOL` | Overflow testing tool |
| `CONTEXT_COLLAPSE` | Context collapse optimization |
| `TERMINAL_PANEL` | Terminal capture panel |
| `CCR_REMOTE_SETUP` | Claude Code Remote setup |
| `COORDINATOR_MODE` | Multi-agent coordinator mode |
| `ULTRAPLAN` | Enhanced planning |
| `TORCH` | Torch command |
| `UDS_INBOX` | Unix Domain Socket messaging |
| `FORK_SUBAGENT` | Fork subagent support |
| `BUDDY` | Buddy feature |
| `REACTIVE_COMPACT` | Reactive compaction |
| `CACHED_MICROCOMPACT` | Cached micro-compaction |
| `TOKEN_BUDGET` | Token budget tracking |
| `EXPERIMENTAL_SKILL_SEARCH` | Skill search indexing |
| `TEMPLATES` | Job classifier templates |
| `TRANSCRIPT_CLASSIFIER` | Auto-mode transcript classifier |
| `BREAK_CACHE_COMMAND` | Cache breaking (ant-only debugging) |
| `COMMIT_ATTRIBUTION` | Commit attribution tracking |
| `BG_SESSIONS` | Background sessions |
| `TEAMMEM` | Team memory sync |
| `MCP_SKILLS` | MCP-provided skills |

## Build Types

### External Build

The public npm package with:
- All internal-only tools removed
- Internal-only commands removed
- Feature flags resolved to their external values
- Anti-debugging protection enabled
- Dead code eliminated for disabled features

### Internal Build (ant)

Anthropic employee build with:
- All features enabled
- Internal-only tools and commands available
- Debug tools (TungstenTool, ConfigTool, REPLTool)
- Additional telemetry and analytics
- No anti-debugging protection

## Environment-Based Gating

Beyond compile-time flags, runtime checks gate behavior:

### User Type

```typescript
process.env.USER_TYPE === 'ant'  // Anthropic internal employee
```

Used for:
- Internal-only commands (50+ commands)
- Internal-only tools (ConfigTool, TungstenTool, REPLTool)
- Enhanced permission bypass checks
- Event loop stall detector
- Repo classification for auto-undercover mode

### Build Type Check

```typescript
"external" !== 'ant'  // Replaced at build time
```

The string `"external"` is the build-time value. In internal builds, this becomes `"ant" !== 'ant'` (false), skipping anti-debugging and other external-only guards.

### Runtime Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_SIMPLE` | Reduced tool set (Bash + Read + Edit) |
| `CLAUDE_CODE_REMOTE` | Remote execution mode |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | Disable CLAUDE.md loading |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | Skip history recording |
| `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` | Benchmark mode |
| `CLAUDE_CODE_VERIFY_PLAN` | Enable plan verification tool |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` | Synchronous plugin install |
| `CLAUDE_CODE_USE_BEDROCK` | Use AWS Bedrock |
| `CLAUDE_CODE_USE_VERTEX` | Use GCP Vertex AI |
| `ENABLE_LSP_TOOL` | Enable LSP tool |
| `IS_SANDBOX` | Running in sandbox environment |
| `IS_DEMO` | Demo mode |
| `NODE_ENV` | Standard Node.js environment |

## Dead Code Elimination Patterns

### Conditional Require

```typescript
// The bundler sees this as a constant expression and eliminates
// the require() call and all transitively imported modules
const SleepTool = feature('PROACTIVE')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

### Guarded Feature Strings

Feature-gated strings are kept inside guarded modules to prevent them from appearing in external builds:

```typescript
// query.ts — snip feature uses strings that must not leak
const snipModule = feature('HISTORY_SNIP')
  ? require('./services/compact/snipCompact.js')
  : null
```

### Lazy Require for Circular Dependencies

```typescript
// Lazy require to break circular dependency
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js')
    .TeamCreateTool as typeof import('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
```

### Dynamic Import for Deferred Loading

```typescript
// Heavy module only loaded when actually invoked
const usageReport: Command = {
  async getPromptForCommand(args, context) {
    const real = (await import('./commands/insights.js')).default
    return real.getPromptForCommand(args, context)
  },
}
```

## Biome Lint Integration

The codebase uses custom lint rules to enforce import ordering:

```typescript
// biome-ignore-all assist/source/organizeImports: ANT-ONLY import markers must not be reordered
```

This prevents automatic import sorting from breaking the intentional side-effect import ordering in `main.tsx` and other critical files.

## Migration System

Model and feature migrations are versioned:

```typescript
const CURRENT_MIGRATION_VERSION = 11;

function runMigrations(): void {
  if (getGlobalConfig().migrationVersion !== CURRENT_MIGRATION_VERSION) {
    // Run all migrations
    // Save new version
  }
}
```

This ensures each migration runs exactly once, even as new migrations are added.
