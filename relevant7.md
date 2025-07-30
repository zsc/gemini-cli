# Chapter 7 Related Information - CLI Interface Architecture

## Key Files to Examine:
- packages/cli/src/gemini.tsx - Main entry point
- packages/cli/src/ui/App.tsx - Main App component
- packages/cli/src/ui/contexts/ - Context providers
- packages/cli/src/ui/hooks/ - Custom hooks
- packages/cli/src/ui/components/ - UI components
- packages/cli/src/ui/commands/ - Command implementations
- packages/cli/src/ui/themes/ - Theme system

## Information to Collect:
1. Ink/React architecture setup
2. Component hierarchy and structure
3. State management approach
4. Keyboard interaction handling
5. Theme system
6. Main UI flow

## 1. Ink/React Architecture Setup

### Entry Point (gemini.tsx)
- Uses `render` from 'ink' to render React component to terminal
- Main render call:
```tsx
const instance = render(
  <React.StrictMode>
    <AppWrapper
      config={config}
      settings={settings}
      startupWarnings={startupWarnings}
      version={version}
    />
  </React.StrictMode>,
  { exitOnCtrlC: false },
);
```
- Handles both interactive and non-interactive modes
- Interactive mode determined by: `!!argv.promptInteractive || (process.stdin.isTTY && input?.length === 0)`

### AppWrapper Component
- Provides two main context providers:
  1. SessionStatsProvider - manages session statistics and metrics
  2. VimModeProvider - manages vim mode state

## 2. Component Hierarchy

### Main App Component Structure
- AppWrapper
  - SessionStatsProvider
    - VimModeProvider
      - App (main component)
        - StreamingContext.Provider
          - Static (Ink's static rendering area)
            - Header
            - Tips
            - HistoryItemDisplay (for each history item)
          - OverflowProvider
            - Pending history items
            - ShowMoreLines
          - Various dialogs (ThemeDialog, AuthDialog, etc.)
          - InputPrompt
          - Footer

### Key UI Components
- **Header**: Shows banner and branding
- **Tips**: Displays helpful tips
- **HistoryItemDisplay**: Renders conversation history items
- **InputPrompt**: Main input component with autocomplete
- **Footer**: Shows model info, branch, debug info
- **Various Dialogs**: Theme selection, auth, editor settings, etc.

## 3. State Management

### Context-Based State Management
1. **SessionStatsContext**:
   - Manages session metrics
   - Provides: `stats`, `startNewPrompt`, `getPromptCount`
   - Integrates with telemetry service

2. **VimModeContext**: 
   - Manages vim mode state
   - Provides vim enable/disable toggle

3. **StreamingContext**:
   - Provides current streaming state to components

4. **OverflowContext**:
   - Manages overflow display for long content

### Local State in App Component
Key state variables:
- UI state: `showHelp`, `showErrorDetails`, `constrainHeight`, etc.
- Dialog states: `isThemeDialogOpen`, `isAuthDialogOpen`, etc.
- Model/config state: `currentModel`, `shellModeActive`
- Error states: `themeError`, `authError`, `initError`
- Terminal state: `footerHeight`, terminal dimensions

### Custom Hooks for State Management
- `useGeminiStream`: Manages streaming API responses
- `useSlashCommandProcessor`: Handles slash commands
- `useHistory`: Manages conversation history
- `useConsoleMessages`: Captures and manages console output
- `useLoadingIndicator`: Manages loading state and phrases
- `useAutoAcceptIndicator`: Manages auto-accept mode display

## 4. Keyboard Interaction Handling

### useKeypress Hook
- Core keyboard handling hook
- Supports bracketed paste mode
- Handles special key combinations
- Works with Node.js readline module

### InputPrompt Key Handling
Key bindings:
- `!` at start: Toggle shell mode
- `Escape`: Exit shell mode or suggestions
- `Ctrl+L`: Clear screen
- `Ctrl+P/N` or `Up/Down`: Navigate history
- `Tab` or `Enter`: Accept autocomplete suggestion
- `Ctrl+K`: Kill line right
- `Ctrl+U`: Kill line left
- `Ctrl+X`: Open external editor
- `Ctrl+V`: Paste clipboard image
- `Ctrl+A/E`: Move to start/end of line
- Vim mode keys when enabled

### App-Level Key Handling
Global shortcuts:
- `Ctrl+O`: Toggle error details
- `Ctrl+T`: Toggle tool descriptions
- `Ctrl+E`: Toggle IDE context detail (when available)
- `Ctrl+C`: Exit (press twice)
- `Ctrl+D`: Exit when input is empty (press twice)
- `Ctrl+S`: Disable height constraint

## 5. Theme System

### ThemeManager
- Singleton pattern for managing themes
- Supports built-in and custom themes
- Theme types: 'dark', 'light', 'ansi', 'custom'
- Built-in themes include: Ayu, Atom One Dark, Dracula, GitHub, etc.

### Theme Structure
Each theme defines colors for:
- Primary UI elements (accent colors)
- Text states (error, warning, success)
- Code syntax highlighting
- Background and foreground

### Theme Loading
- Custom themes loaded from settings
- Validates custom theme configurations
- Falls back to default theme if invalid
- Respects NO_COLOR environment variable

## 6. Main UI Flow

### Initialization Flow
1. Load settings and configuration
2. Determine interactive vs non-interactive mode
3. Set up theme manager with custom themes
4. Initialize authentication if needed
5. Render main App component

### Rendering Flow
1. **Static Area**: Previous messages rendered once
2. **Dynamic Area**: Current input and pending messages
3. **Dialogs**: Modal-like overlays for settings
4. **Input Area**: Always at bottom with autocomplete

### Update Cycle
- Uses Ink's reconciliation for efficient terminal updates
- Static component prevents re-rendering of history
- Terminal resize triggers layout recalculation
- Stream updates trigger partial re-renders

### Component Communication
- Props drilling for configuration
- Context for cross-cutting concerns
- Event emitters for system events
- Callbacks for user interactions