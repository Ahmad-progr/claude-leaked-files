# Claude Code: UI and Rendering

## Overview

Claude Code uses **React + Ink** to render a rich terminal UI. Ink is a React renderer for the terminal — it translates React components into ANSI escape sequences for terminal display.

## Ink Wrapper (`ink.ts`)

The `ink.ts` module wraps all Ink rendering calls with a `ThemeProvider`:

```typescript
function withTheme(node: ReactNode): ReactNode {
  return createElement(ThemeProvider, null, node)
}

export async function render(node: ReactNode, options?): Promise<Instance> {
  return inkRender(withTheme(node), options)
}

export async function createRoot(options?: RenderOptions): Promise<Root> {
  const root = await inkCreateRoot(options)
  return {
    ...root,
    render: node => root.render(withTheme(node)),
  }
}
```

This ensures every component has access to the theme without call sites needing to mount it explicitly.

## Component Exports

Claude Code exports a comprehensive set of UI components:

### Design System

| Component | Purpose |
|-----------|---------|
| `Box` / `ThemedBox` | Themed container with flexbox layout |
| `Text` / `ThemedText` | Themed text with color/style support |
| `ThemeProvider` | Theme context provider |
| `Button` | Interactive button component |
| `Link` | Clickable link |
| `Spacer` | Flexible space filler |
| `Newline` | Line break |
| `NoSelect` | Non-selectable content |
| `RawAnsi` | Raw ANSI escape sequence passthrough |
| `Ansi` | Parsed ANSI rendering |

### Hooks

| Hook | Purpose |
|------|---------|
| `useApp` | Access app context (exit, etc.) |
| `useInput` | Keyboard input handling |
| `useStdin` | Raw stdin access |
| `useSelection` | Focus/selection management |
| `useInterval` / `useAnimationTimer` | Timed updates |
| `useAnimationFrame` | Frame-synced updates |
| `useTerminalViewport` | Terminal size tracking |
| `useTerminalTitle` | Set terminal title |
| `useTerminalFocus` | Terminal focus state |
| `useTabStatus` | Tab status updates |
| `useTheme` / `useThemeSetting` | Theme access |
| `usePreviewTheme` | Theme preview |

### Events

| Event | Purpose |
|-------|---------|
| `InputEvent` | Keyboard input events |
| `ClickEvent` | Mouse click events |
| `TerminalFocusEvent` | Terminal focus/blur |
| `EventEmitter` | Custom event system |

### Utilities

| Utility | Purpose |
|---------|---------|
| `measureElement` | Measure rendered dimensions |
| `wrapText` | Text wrapping for terminal width |
| `supportsTabStatus` | Check terminal capability |
| `FocusManager` | Focus management across components |

## Application Structure

### REPL Launch

```typescript
export async function launchRepl(
  root: Root,
  appProps: AppWrapperProps,
  replProps: REPLProps,
  renderAndRun: (root: Root, element: React.ReactNode) => Promise<void>,
): Promise<void> {
  const { App } = await import('./components/App.js')
  const { REPL } = await import('./screens/REPL.js')
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)
}
```

**Note**: Both `App` and `REPL` are **dynamically imported** at launch time to keep the initial bundle smaller and startup faster.

### App Wrapper Props

```typescript
type AppWrapperProps = {
  getFpsMetrics: () => FpsMetrics | undefined
  stats?: StatsStore
  initialState: AppState
}
```

## Theme System

Themes are managed through the `ThemeProvider` and accessed via hooks:

```typescript
type ThemeName = string  // e.g., 'dark', 'light', 'solarized'
type Theme = {
  // Color definitions for all UI elements
  // Background colors, text colors, borders, etc.
}
```

Tools can specify background colors for their names:

```typescript
userFacingNameBackgroundColor?(input): keyof Theme | undefined
```

## Tool Rendering

Each tool defines multiple rendering methods for different UI states:

### Tool Use (Invocation)

```typescript
renderToolUseMessage(
  input: Partial<z.infer<Input>>,  // Partial because we render before params fully stream
  options: { theme: ThemeName; verbose: boolean; commands?: Command[] },
): React.ReactNode
```

### Tool Result

```typescript
renderToolResultMessage?(
  content: Output,
  progressMessages: ProgressMessage<P>[],
  options: {
    style?: 'condensed'
    theme: ThemeName
    tools: Tools
    verbose: boolean
    isTranscriptMode?: boolean
    isBriefOnly?: boolean
    input?: unknown  // For compact summaries
  },
): React.ReactNode
```

### Progress

```typescript
renderToolUseProgressMessage?(
  progressMessages: ProgressMessage<P>[],
  options: {
    tools: Tools
    verbose: boolean
    terminalSize?: { columns: number; rows: number }
    inProgressToolCallCount?: number
  },
): React.ReactNode
```

### Grouped Rendering

Multiple parallel tool uses can be rendered as a group:

```typescript
renderGroupedToolUse?(
  toolUses: Array<{
    param: ToolUseBlockParam
    isResolved: boolean
    isError: boolean
    isInProgress: boolean
    progressMessages: ProgressMessage<P>[]
    result?: { param: ToolResultBlockParam; output: unknown }
  }>,
  options: { shouldAnimate: boolean; tools: Tools },
): React.ReactNode | null
```

### Activity Description

For spinner display:

```typescript
getActivityDescription?(input): string | null
// Examples: "Reading src/foo.ts", "Running bun test", "Searching for pattern"
```

### Search/Read Collapsing

Tools can indicate their output should be collapsed in the UI:

```typescript
isSearchOrReadCommand?(input): {
  isSearch: boolean  // grep, find, glob
  isRead: boolean    // cat, head, file read
  isList?: boolean   // ls, tree, du
}
```

### Transcript Search

Tools can provide flattened text for transcript search indexing:

```typescript
extractSearchText?(out: Output): string
// Must match what renderToolResultMessage shows in transcript mode
// to avoid count/highlight mismatches
```

### Truncation Detection

```typescript
isResultTruncated?(output: Output): boolean
// Gates click-to-expand in fullscreen mode
```

## Cost Summary Hook

The `useCostSummary` React hook displays cost summary on exit:

```typescript
export function useCostSummary(getFpsMetrics?): void {
  useEffect(() => {
    const f = () => {
      if (hasConsoleBillingAccess()) {
        process.stdout.write('\n' + formatTotalCost() + '\n')
      }
      saveCurrentSessionCosts(getFpsMetrics?.())
    }
    process.on('exit', f)
    return () => { process.off('exit', f) }
  }, [])
}
```

## FPS Tracking

Claude Code tracks rendering performance:

```typescript
type FpsMetrics = {
  averageFps: number
  low1PctFps: number  // 1st percentile (worst frames)
}
```

These metrics are:
- Persisted with session costs
- Logged in `tengu_exit` analytics events
- Used for performance monitoring
