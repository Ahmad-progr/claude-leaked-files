# Claude Code: Security and Permissions

## Overview

Security is deeply embedded in Claude Code's architecture. The system uses multiple layers of defense: permission modes, tool-level checks, input validation, deny rules, sandbox detection, trust dialogs, and anti-debugging protection.

## Permission Modes

```typescript
type PermissionMode =
  | 'default'             // Ask user for each tool use
  | 'plan'                // Read-only tools auto-allowed, writes need approval
  | 'auto'                // AI classifier decides based on risk
  | 'bypassPermissions'   // All tools auto-allowed (sandbox only)
```

### Default Mode

Every tool use requires explicit user approval. This is the safest mode for normal interactive use.

### Plan Mode

Read-only operations are auto-approved. The tool's `isReadOnly(input)` method determines this:

```typescript
// In buildTool defaults:
isReadOnly: (_input?: unknown) => false  // Assume writes by default
```

### Auto Mode

An AI classifier analyzes tool uses and auto-approves safe operations. Requires the `TRANSCRIPT_CLASSIFIER` feature flag.

### Bypass Permissions Mode

All tools auto-approved. **Heavily restricted**:

```typescript
if (permissionMode === 'bypassPermissions' || allowDangerouslySkipPermissions) {
  // Check not running as root (unless in sandbox)
  if (process.getuid() === 0 && process.env.IS_SANDBOX !== '1') {
    console.error('--dangerously-skip-permissions cannot be used with root/sudo')
    process.exit(1)
  }

  // For internal builds: must be in Docker/sandbox WITHOUT internet
  if (process.env.USER_TYPE === 'ant') {
    const [isDocker, hasInternet] = await Promise.all([
      envDynamic.getIsDocker(),
      env.hasInternetAccess(),
    ])
    const isSandboxed = isDocker || isBubblewrap || isSandbox
    if (!isSandboxed || hasInternet) {
      process.exit(1)
    }
  }
}
```

## Three-Layer Permission Check

Every tool use goes through three layers:

### 1. Input Validation (`validateInput`)

Tool-specific input validation that runs first:

```typescript
validateInput?(
  input: z.infer<Input>,
  context: ToolUseContext,
): Promise<ValidationResult>

type ValidationResult =
  | { result: true }
  | { result: false; message: string; errorCode: number }
```

### 2. General Permission System

The centralized permission logic in `permissions.ts` checks:
- Permission mode (default/plan/auto/bypass)
- Always-allow rules
- Always-deny rules
- Always-ask rules
- Additional working directory restrictions

### 3. Tool-Specific Permissions (`checkPermissions`)

```typescript
checkPermissions(
  input: z.infer<Input>,
  context: ToolUseContext,
): Promise<PermissionResult>
```

## Permission Rules

Rules are organized by source and can target tools with optional patterns:

```typescript
type ToolPermissionRulesBySource = {
  [source: string]: Array<{
    toolName: string
    ruleContent?: string  // Optional pattern (e.g., "git *" for Bash)
  }>
}
```

### Permission Matcher

Tools can implement custom pattern matching for permission rules:

```typescript
preparePermissionMatcher?(
  input: z.infer<Input>,
): Promise<(pattern: string) => boolean>
```

For example, `BashTool` matches shell command patterns like `"git *"`.

### Deny Rules

Blanket deny rules filter tools before the model even sees them:

```typescript
export function filterToolsByDenyRules<T extends { name: string; mcpInfo?: ... }>(
  tools: readonly T[],
  permissionContext: ToolPermissionContext,
): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}
```

MCP server-prefix rules (e.g., `mcp__server`) strip **all** tools from that server.

## Fail-Closed Defaults

The `buildTool()` factory applies conservative defaults:

```typescript
const TOOL_DEFAULTS = {
  isConcurrencySafe: () => false,   // Assume NOT safe for parallel execution
  isReadOnly: () => false,          // Assume writes to filesystem/state
  isDestructive: () => false,       // Not irreversible by default
  checkPermissions: (input) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
}
```

**`isConcurrencySafe` defaults to `false`**: New tools won't accidentally run in parallel without explicit opt-in.

**`isReadOnly` defaults to `false`**: New tools require user permission in plan mode by default.

## Immutable Permission Context

Permission context uses `DeepImmutable` to prevent mutation:

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  // ...
}>
```

## Trust-Gated Operations

### Git Commands

Git can execute arbitrary code via hooks/config, so git operations are trust-gated:

```typescript
function prefetchSystemContextIfSafe(): void {
  if (isNonInteractiveSession) {
    void getSystemContext()  // Non-interactive: trust is implicit
    return
  }
  if (checkHasTrustDialogAccepted()) {
    void getSystemContext()  // Interactive: only if trust confirmed
  }
  // Otherwise: skip until trust is established
}
```

### Anti-Debugging

External builds detect and block debugging:

```typescript
function isBeingDebugged() {
  // Check --inspect/--inspect-brk flags
  // Check NODE_OPTIONS for inspect flags
  // Check for active inspector URL
}

if ("external" !== 'ant' && isBeingDebugged()) {
  process.exit(1)
}
```

## Denial Tracking

Permission denials are tracked for both analytics and adaptive behavior:

```typescript
// QueryEngine wraps canUseTool to track denials
if (result.behavior !== 'allow') {
  this.permissionDenials.push({
    tool_name: sdkCompatToolName(tool.name),
    tool_use_id: toolUseID,
    tool_input: input,
  })
}
```

For subagents, denial tracking is local since their `setAppState` may be a no-op:

```typescript
localDenialTracking?: DenialTrackingState
// Without this, the denial counter never accumulates and the
// fallback-to-prompting threshold is never reached.
```

## Background Agent Permissions

Background agents that can't show UI auto-deny permission prompts:

```typescript
shouldAvoidPermissionPrompts?: boolean
// When true, permission prompts are auto-denied
```

Coordinator workers can await automated checks before showing dialogs:

```typescript
awaitAutomatedChecksBeforeDialog?: boolean
// When true, classifier/hooks run before the permission dialog shows
```

## Security Classifier

For auto-mode, tools can provide compact input for the security classifier:

```typescript
toAutoClassifierInput(input: z.infer<Input>): unknown
// Examples: "ls -la" for Bash, "/tmp/x: new content" for Edit
// Return '' to skip (tools with no security relevance)
```

## Sandbox Integration

```typescript
SandboxManager.isSandboxingEnabled()
SandboxManager.areUnsandboxedCommandsAllowed()
SandboxManager.isAutoAllowBashIfSandboxedEnabled()
```

The sandbox can auto-allow bash commands when enabled, reducing permission prompts in sandboxed environments.

## Hooks Configuration Safety

Hooks configuration is snapshot on startup and monitored for changes:

```typescript
captureHooksConfigSnapshot()  // After setCwd, before any execution
// ... later ...
updateHooksConfigSnapshot()   // After worktree switch
```

This detects unauthorized hook modifications during a session.

## Analytics Type Safety

The analytics system uses a branded type to prevent accidental PII logging:

```typescript
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = string
```

The type name itself serves as a code review guardrail — developers must explicitly cast values, acknowledging they verified the content is safe.

## File Permission Safety

History and settings files are written with restrictive permissions:

```typescript
await writeFile(historyPath, '', { mode: 0o600 })  // Owner read/write only
await appendFile(historyPath, jsonLines.join(''), { mode: 0o600 })
```

## Content Hash for Cache Stability

Settings files use content-based hashing instead of random UUIDs:

```typescript
settingsPath = generateTempFilePath('claude-settings', '.json', {
  contentHash: trimmedSettings
})
```

This prevents random UUIDs in sandbox deny paths from busting API prompt caches (which would cause a 12x input token cost penalty).
