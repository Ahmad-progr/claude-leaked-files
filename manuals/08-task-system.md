# Claude Code: Task System

## Overview

Claude Code supports background tasks that run concurrently with the main conversation. Tasks can be shell commands, agent subprocesses, workflows, or monitors. They have their own lifecycle, output files, and abort controllers.

## Task Types

```typescript
type TaskType =
  | 'local_bash'           // Shell command running in background
  | 'local_agent'          // Local subagent process
  | 'remote_agent'         // Remote agent (cloud-based)
  | 'in_process_teammate'  // In-process teammate (swarm)
  | 'local_workflow'       // Workflow script (WORKFLOW_SCRIPTS flag)
  | 'monitor_mcp'          // MCP server monitor (MONITOR_TOOL flag)
  | 'dream'                // Dream task
```

## Task Registry (`tasks.ts`)

Tasks are registered similarly to tools, with feature-flag gating:

```typescript
export function getAllTasks(): Task[] {
  const tasks: Task[] = [
    LocalShellTask,
    LocalAgentTask,
    RemoteAgentTask,
    DreamTask,
  ]
  if (LocalWorkflowTask) tasks.push(LocalWorkflowTask)   // WORKFLOW_SCRIPTS
  if (MonitorMcpTask) tasks.push(MonitorMcpTask)          // MONITOR_TOOL
  return tasks
}
```

## Task Lifecycle

### States

```typescript
type TaskStatus =
  | 'pending'     // Created, not yet started
  | 'running'     // Currently executing
  | 'completed'   // Finished successfully
  | 'failed'      // Finished with error
  | 'killed'      // Manually stopped
```

### Terminal States

```typescript
export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

Terminal state checks are used to:
- Guard against injecting messages into dead teammates
- Evict finished tasks from AppState
- Run orphan-cleanup paths

## Task State

```typescript
type TaskStateBase = {
  id: string              // Unique ID with type prefix
  type: TaskType
  status: TaskStatus
  description: string
  toolUseId?: string      // Associated tool use (for UI linking)
  startTime: number
  endTime?: number
  totalPausedMs?: number
  outputFile: string      // Disk file for output
  outputOffset: number    // Current read position
  notified: boolean       // Whether completion was notified
}
```

## Task ID Generation

Task IDs include a type prefix for identification:

```typescript
const TASK_ID_PREFIXES: Record<string, string> = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}

export function generateTaskId(type: TaskType): string {
  const prefix = getTaskIdPrefix(type)
  const bytes = randomBytes(8)
  let id = prefix
  for (let i = 0; i < 8; i++) {
    id += TASK_ID_ALPHABET[bytes[i]! % TASK_ID_ALPHABET.length]
  }
  return id
}
```

**Security note**: Uses `crypto.randomBytes()` with 36^8 (~2.8 trillion) combinations to resist brute-force symlink attacks on task output files.

## Task Context

```typescript
type TaskContext = {
  abortController: AbortController
  getAppState: () => AppState
  setAppState: SetAppState
}
```

## Task Interface

The `Task` interface is intentionally minimal:

```typescript
type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

Spawn and render were originally polymorphic but were removed — each task type handles those directly. Only `kill` needs runtime dispatch via `getTaskByType()`.

## Tool Integration

Tasks are created and managed through dedicated tools:

| Tool | Purpose |
|------|---------|
| `TaskCreateTool` | Create a new background task |
| `TaskGetTool` | Get task status and details |
| `TaskUpdateTool` | Update task status |
| `TaskListTool` | List all tasks |
| `TaskOutputTool` | Read task output from disk |
| `TaskStopTool` | Stop a running task |

### Shell Task Input

```typescript
type LocalShellSpawnInput = {
  command: string
  description: string
  timeout?: number
  toolUseId?: string
  agentId?: AgentId
  kind?: 'bash' | 'monitor'  // UI display variant
}
```

## Session-Scoped State

Tasks use `setAppStateForTasks` — a special state setter that always reaches the root store:

```typescript
// Unlike setAppState, which is no-op for async agents,
// this always reaches the root store so agents at any nesting depth
// can register/clean up infrastructure that outlives a single turn.
setAppStateForTasks?: (f: (prev: AppState) => AppState) => void
```

This ensures background tasks can register and clean up regardless of how deeply nested the creating agent is.
