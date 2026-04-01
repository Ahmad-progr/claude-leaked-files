# Claude Code: Slash Commands

## Overview

Claude Code provides ~70+ slash commands accessible via `/command-name` in the REPL. Commands are loaded from multiple sources: built-in, bundled skills, plugin skills, skill directories, MCP servers, and workflows.

## Command Types

Commands are classified into three types:

| Type | Description |
|------|-------------|
| `prompt` | Expands to text sent to the model (skills, workflows) |
| `local` | Executes locally and returns text output |
| `local-jsx` | Renders interactive Ink UI in the terminal |

## Complete Command List

### Navigation & Session

| Command | Description |
|---------|-------------|
| `/clear` | Clear the screen/transcript |
| `/exit` | Exit Claude Code |
| `/resume` | Resume a previous session |
| `/session` | Session management (QR code/URL for remote) |
| `/rename` | Rename the current session |
| `/compact` | Compact conversation context (summarize history) |
| `/rewind` | Undo recent changes |
| `/branch` | Git branch management |

### Configuration

| Command | Description |
|---------|-------------|
| `/config` | View/edit configuration |
| `/theme` | Change terminal theme |
| `/color` | Change agent color |
| `/vim` | Toggle vim keybinding mode |
| `/keybindings` | Customize keyboard shortcuts |
| `/permissions` | Manage permission rules |
| `/hooks` | View/manage hooks configuration |
| `/statusline` | Toggle status line display |
| `/fast` | Toggle fast output mode |
| `/model` | Change the active model |
| `/effort` | Set reasoning effort level |
| `/advisor` | Configure advisor model |
| `/output-style` | Change output formatting style |
| `/privacy-settings` | Manage privacy/analytics settings |
| `/sandbox-toggle` | Toggle sandbox mode |

### Code & Development

| Command | Description |
|---------|-------------|
| `/init` | Create a CLAUDE.md file with project instructions |
| `/diff` | Show current changes |
| `/review` | Code review |
| `/ultrareview` | Enhanced code review |
| `/security-review` | Security-focused code review |
| `/plan` | Enter/exit planning mode |
| `/files` | List tracked files |
| `/context` | View current context information |

### Information & Help

| Command | Description |
|---------|-------------|
| `/help` | Show help information |
| `/cost` | Show session cost breakdown |
| `/usage` | Show usage information |
| `/status` | Show session status |
| `/stats` | Show session statistics |
| `/doctor` | Diagnose configuration issues |
| `/release-notes` | Show changelog |
| `/insights` | Generate usage analysis report (lazy-loaded, 113KB module) |

### Skills & Plugins

| Command | Description |
|---------|-------------|
| `/skills` | List available skills |
| `/plugin` | Manage plugins |
| `/reload-plugins` | Reload all plugins |
| `/agents` | List available agent types |

### Sharing & Communication

| Command | Description |
|---------|-------------|
| `/copy` | Copy last message to clipboard |
| `/feedback` | Send feedback |
| `/btw` | Quick note (inject context) |
| `/export` | Export conversation |
| `/share` | Share session (internal) |
| `/stickers` | Stickers |
| `/tag` | Tag the current session |

### Account & Auth

| Command | Description |
|---------|-------------|
| `/login` | Log in to Anthropic account |
| `/logout` | Log out |
| `/passes` | View Claude passes |
| `/upgrade` | Upgrade plan |

### Infrastructure

| Command | Description |
|---------|-------------|
| `/mcp` | Manage MCP servers |
| `/ide` | IDE integration management |
| `/desktop` | Desktop app management |
| `/mobile` | Mobile QR code |
| `/chrome` | Chrome extension |
| `/terminal-setup` | Configure terminal settings |

### History & Memory

| Command | Description |
|---------|-------------|
| `/memory` | View/manage persistent memory |
| `/thinkback` | View thinking history |
| `/thinkback-play` | Replay thinking history |
| `/tasks` | View background tasks |
| `/summary` | Summarize conversation |

## Feature-Flagged Commands

Some commands are only available with specific feature flags:

| Command | Feature Flag |
|---------|-------------|
| `/proactive` | PROACTIVE or KAIROS |
| `/brief` | KAIROS or KAIROS_BRIEF |
| `/assistant` | KAIROS |
| `/bridge` | BRIDGE_MODE |
| `/remote-control-server` | DAEMON + BRIDGE_MODE |
| `/voice` | VOICE_MODE |
| `/force-snip` | HISTORY_SNIP |
| `/workflows` | WORKFLOW_SCRIPTS |
| `/web` (remote setup) | CCR_REMOTE_SETUP |
| `/subscribe-pr` | KAIROS_GITHUB_WEBHOOKS |
| `/ultraplan` | ULTRAPLAN |
| `/torch` | TORCH |
| `/peers` | UDS_INBOX |
| `/fork` | FORK_SUBAGENT |
| `/buddy` | BUDDY |

## Internal-Only Commands

These are available only for Anthropic internal builds (`USER_TYPE === 'ant'`):

- `/backfill-sessions`, `/break-cache`, `/bughunter`, `/commit`, `/commit-push-pr`
- `/ctx-viz`, `/good-claude`, `/issue`, `/init-verifiers`
- `/mock-limits`, `/bridge-kick`, `/version`
- `/reset-limits`, `/onboarding`, `/share`, `/summary`, `/teleport`
- `/ant-trace`, `/perf-issue`, `/env`, `/oauth-refresh`, `/debug-tool-call`
- `/agents-platform`, `/autofix-pr`

## Command Loading Architecture

Commands are loaded from multiple sources in priority order:

```typescript
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),
    getPluginCommands(),
    getWorkflowCommands ? getWorkflowCommands(cwd) : [],
  ])

  return [
    ...bundledSkills,        // 1. Bundled skills (highest priority)
    ...builtinPluginSkills,  // 2. Built-in plugin skills
    ...skillDirCommands,     // 3. User skill directories
    ...workflowCommands,     // 4. Workflow scripts
    ...pluginCommands,       // 5. External plugin commands
    ...pluginSkills,         // 6. External plugin skills
    ...COMMANDS(),           // 7. Built-in commands (lowest priority)
  ]
})
```

### Availability Filtering

Commands are filtered by authentication state:

```typescript
function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai':
        if (isClaudeAISubscriber()) return true
        break
      case 'console':
        if (!isClaudeAISubscriber() && !isUsing3PServices() && isFirstPartyAnthropicBaseUrl())
          return true
        break
    }
  }
  return false
}
```

This is **not memoized** because auth state can change mid-session (e.g., after `/login`).

## Remote-Safe and Bridge-Safe Commands

### Remote-Safe Commands

A subset of commands are safe for remote mode (`--remote`):

`/session`, `/exit`, `/clear`, `/help`, `/theme`, `/color`, `/vim`, `/cost`, `/usage`, `/copy`, `/btw`, `/feedback`, `/plan`, `/keybindings`, `/statusline`, `/stickers`, `/mobile`, `/fast`, `/passes`, `/privacy-settings`

### Bridge-Safe Commands

Commands safe to execute when received over the Remote Control bridge (mobile/web):

- `prompt` type commands (skills) are always safe
- `local-jsx` commands are always blocked (they render Ink UI)
- `local` commands need explicit allowlisting: `/compact`, `/clear`, `/cost`, `/summary`, `/release-notes`, `/files`

## Lazy Loading

The `/insights` command demonstrates lazy-loading for expensive modules:

```typescript
const usageReport: Command = {
  type: 'prompt',
  name: 'insights',
  description: 'Generate a report analyzing your Claude Code sessions',
  async getPromptForCommand(args, context) {
    // 113KB module only loaded when actually invoked
    const real = (await import('./commands/insights.js')).default
    return real.getPromptForCommand(args, context)
  },
}
```

## Cache Management

Command loading is memoized, with explicit cache clearing:

```typescript
export function clearCommandsCache(): void {
  clearCommandMemoizationCaches()  // loadAllCommands, getSkillToolCommands, etc.
  clearPluginCommandCache()
  clearPluginSkillsCache()
  clearSkillCaches()
}
```

This is called when plugins change, settings update, or dynamic skills are discovered.
