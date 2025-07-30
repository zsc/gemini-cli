# Chapter 9: Interactive Session Management - Collected Information

## Session Management Files

### SessionContext.tsx
- Location: packages/cli/src/ui/contexts/SessionContext.tsx
- Provides React context for session state management
- Key components: SessionStatsProvider, useSessionStats hook
- Manages session metrics, prompt counts, token usage
- Integrates with uiTelemetryService for metrics tracking

### session.ts (utils)
- Location: packages/core/src/utils/session.ts
- Simple session ID generation using randomUUID()
- Creates unique session identifier for each CLI session

## History Management Files

### useHistoryManager.ts
- Location: packages/cli/src/ui/hooks/useHistoryManager.ts
- Hook for managing conversation history
- Key functions:
  - addItem: Adds new history items with unique IDs
  - updateItem: Updates existing history items (deprecated for performance)
  - clearItems: Clears entire history
  - loadHistory: Loads saved history (used for restore)
- Prevents duplicate consecutive user messages
- Uses timestamp-based ID generation

### useInputHistory.ts  
- Location: packages/cli/src/ui/hooks/useInputHistory.ts
- Hook for managing user input history navigation
- Enables up/down arrow navigation through previous inputs
- Stores original query before navigation
- Returns to original query when navigating past history

### useShellHistory.ts
- Location: packages/cli/src/ui/hooks/useShellHistory.ts
- Hook for managing shell command history

## Context Maintenance Files

### ideContext.ts
- Location: packages/core/src/ide/ideContext.ts
- IDE context management

### OverflowContext.tsx
- Location: packages/cli/src/ui/contexts/OverflowContext.tsx
- Context for managing overflow states

### StreamingContext.tsx
- Location: packages/cli/src/ui/contexts/StreamingContext.tsx
- Context for streaming responses

## Checkpoint/Restore Files

### restoreCommand.ts
- Location: packages/cli/src/ui/commands/restoreCommand.ts
- Command for restoring tool call checkpoints
- Key features:
  - Lists available checkpoints if no args
  - Restores history, client history, and project state
  - Uses git snapshots for file restoration
  - Stores checkpoints in .gemini/checkpoints directory

### chatCommand.ts
- Location: packages/cli/src/ui/commands/chatCommand.ts
- Chat command with session management features
- Subcommands:
  - list: Lists saved conversation checkpoints
  - save: Saves current conversation with a tag
  - resume/load: Loads a saved conversation
  - delete: Deletes a saved checkpoint
- Stores conversations as checkpoint-<tag>.json files

### GitService
- Location: packages/core/src/services/gitService.ts
- Creates shadow git repository for checkpointing
- Key methods:
  - setupShadowGitRepository: Creates hidden git repo
  - createFileSnapshot: Creates git commit for current state
  - restoreProjectFromSnapshot: Restores files to previous commit
- Uses project-specific history directory

## History Item Types
From types.ts:
- HistoryItemUser: User messages
- HistoryItemGemini: AI responses
- HistoryItemGeminiContent: AI content responses
- HistoryItemInfo: Info messages
- HistoryItemError: Error messages
- HistoryItemAbout: About information
- HistoryItemStats: Session statistics
- HistoryItemModelStats: Model usage stats
- HistoryItemToolStats: Tool usage stats
- HistoryItemQuit: Session end
- HistoryItemToolGroup: Tool execution group
- HistoryItemUserShell: Shell commands
- HistoryItemCompression: Compression events

## Key Architecture Insights

1. **Session Identity**: Each session gets a unique UUID
2. **History Management**: React-based state management with hooks
3. **Persistence**: Two levels of persistence:
   - Tool checkpoints: Includes file snapshots via git
   - Conversation checkpoints: Just the chat history
4. **Metrics Tracking**: Session stats tracked via telemetry service
5. **Navigation**: Input history navigation separate from conversation history

## Context Management

### StreamingContext
- Simple context for streaming state (idle, responding, waiting_for_confirmation)
- Used to coordinate UI state during AI responses

### OverflowContext  
- Manages UI overflow states for expandable content
- Tracks which components are overflowing via ID sets
- Provides actions to add/remove overflowing IDs

### IDEContext
- Manages integration with VSCode extension
- Tracks open files, active file, cursor position, selected text
- Uses pub-sub pattern for context updates
- Factory pattern for creating isolated stores

## Shell History
- Persists shell command history to .gemini/shell_history
- Max 100 commands stored
- Newest commands stored first in memory
- Navigation via getPreviousCommand/getNextCommand
- Separate from conversation history

## Logger Checkpoints
- Logger class handles conversation checkpoints
- Methods: saveCheckpoint, loadCheckpoint, deleteCheckpoint
- Stores as checkpoint-<tag>.json files
- Contains full conversation history in Gemini Content format