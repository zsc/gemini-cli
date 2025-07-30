# Chapter 13: Conversation Flow - Relevant Information

## Key Files to Examine

### Core Flow Files
- packages/cli/src/gemini.tsx - Main entry point and conversation loop
- packages/cli/src/ui/App.tsx - UI component handling user interaction
- packages/cli/src/ui/hooks/useGeminiStream.ts - Streaming response handler
- packages/core/src/core/geminiChat.ts - Core chat logic
- packages/core/src/core/turn.ts - Turn management
- packages/core/src/core/prompts.ts - Prompt construction

### Supporting Files
- packages/cli/src/ui/components/InputPrompt.tsx - User input component
- packages/cli/src/ui/components/messages/* - Message display components
- packages/core/src/core/contentGenerator.ts - Content generation
- packages/core/src/core/client.ts - API client

## Information to Collect
1. User input capture and processing
2. Prompt construction and context building
3. API request formation
4. Streaming response handling
5. Message display and UI updates
6. Turn management and history
7. Error handling in conversation flow

## Collected Information

### 1. Entry Point and Initialization (gemini.tsx)
- Main function handles initial setup
- Determines interactive vs non-interactive mode
- Interactive mode renders React app with Ink
- Non-interactive mode uses runNonInteractive function
- Sets up authentication, configuration, and extensions

### 2. App Component (App.tsx)
- Main UI component using React with Ink
- Manages conversation state and history
- useGeminiStream hook handles the actual conversation flow
- Handles various UI states (help, dialogs, error displays)
- SessionStatsProvider tracks statistics

### 3. Conversation Flow Hook (useGeminiStream.ts)
- Core hook that manages the conversation lifecycle
- Handles:
  - User input processing
  - Command detection (slash commands, shell commands, @ commands)
  - API streaming
  - Tool call scheduling and execution
  - History management
  - Error handling and recovery
- StreamingState enum: Idle, Responding, WaitingForConfirmation
- Uses AbortController for cancellation

### 4. Chat Core (geminiChat.ts)
- GeminiChat class manages conversation with Gemini API
- Maintains conversation history
- Validates content and history
- Handles API requests with retry logic
- Manages fallback to Flash model on quota errors
- Supports streaming and non-streaming responses
- Logs telemetry data

### 5. Prompt System (prompts.ts)
- getCoreSystemPrompt function builds system instructions
- Includes tool descriptions and usage guidelines
- Can be overridden with custom system.md file
- Incorporates user memory if available
- Context-aware (git repo, sandbox environment)
- Defines agent behavior and constraints

### 6. User Input Processing
- InputPrompt component handles user typing
- Supports multi-line input with TextBuffer
- Completion suggestions for files and commands
- History navigation (up/down arrows)
- Clipboard image support (Ctrl+V)
- Shell mode for command history
- Vim mode support

### 7. Streaming Response Processing
- submitQuery function initiates conversation
- processGeminiStreamEvents handles event stream
- Event types: Content, ToolCall, Error, Thought, etc.
- Real-time UI updates through setPendingHistoryItem
- AbortController for cancellation (ESC key)
- Tool scheduling and execution integration

### 8. Turn Management (turn.ts)
- Turn class encapsulates a conversation turn
- Manages tool call lifecycle
- Yields stream events for processing
- Handles thought bubbles
- Tracks pending tool calls
- Supports debug mode responses

### 9. Message Display
- GeminiMessage component for AI responses
- MarkdownDisplay for rich formatting
- Real-time streaming text updates
- Tool messages with status indicators
- Error messages with formatted details
- Compression messages for context management

### 10. Error Handling and Recovery
- Multiple error types: API, Auth, Quota, Tool execution
- Automatic fallback to Flash model on quota errors
- Retry logic with exponential backoff
- User-friendly error messages
- Loop detection for repetitive behavior
- Session turn limits