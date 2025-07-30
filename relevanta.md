# Key Data Structures and Interfaces Collection

This file collects important data structures and interfaces from the Gemini CLI codebase for Appendix A.

## Core Package Interfaces

### Tool System Interfaces (packages/core/src/tools/tools.ts)

#### Tool<TParams, TResult>
Base interface for all tools in the system. Defines the contract for tool implementations including execution, validation, and confirmation.

Key properties:
- name: string - Internal API name
- displayName: string - User-friendly name
- description: string - Tool description
- icon: Icon - Display icon
- schema: FunctionDeclaration - Parameter schema
- isOutputMarkdown: boolean - Markdown output flag
- canUpdateOutput: boolean - Streaming support

Key methods:
- validateToolParams(params): string | null
- getDescription(params): string
- toolLocations(params): ToolLocation[]
- shouldConfirmExecute(params, signal): Promise<ToolCallConfirmationDetails | false>
- execute(params, signal, updateOutput?): Promise<TResult>

#### ToolResult
Interface for tool execution results:
- summary?: string - One-line summary
- llmContent: PartListUnion - Content for LLM history
- returnDisplay: ToolResultDisplay - User display content

#### ToolCallConfirmationDetails
Union type for different confirmation dialogs:
- ToolEditConfirmationDetails - File edit confirmations
- ToolExecuteConfirmationDetails - Shell command confirmations
- ToolMcpConfirmationDetails - MCP tool confirmations
- ToolInfoConfirmationDetails - Information confirmations

#### ToolConfirmationOutcome
Enum for user confirmation decisions:
- ProceedOnce
- ProceedAlways
- ProceedAlwaysServer
- ProceedAlwaysTool
- ModifyWithEditor
- Cancel

### API Integration Types (packages/core/src/core/)

#### GeminiCodeRequest (geminiRequest.ts)
Type alias for PartListUnion representing requests to Gemini API.

#### Turn Management (turn.ts)
Manages agentic loop turns within server context.

##### ServerGeminiStreamEvent
Union type for all streaming events:
- ServerGeminiContentEvent - Text content
- ServerGeminiToolCallRequestEvent - Tool call requests
- ServerGeminiToolCallResponseEvent - Tool call results
- ServerGeminiToolCallConfirmationEvent - Confirmation requests
- ServerGeminiUserCancelledEvent - User cancellations
- ServerGeminiErrorEvent - Errors
- ServerGeminiChatCompressedEvent - Chat compression
- ServerGeminiThoughtEvent - Model thoughts
- ServerGeminiMaxSessionTurnsEvent - Session limits
- ServerGeminiFinishedEvent - Completion events
- ServerGeminiLoopDetectedEvent - Loop detection

##### GeminiEventType
Enum for event types in the streaming protocol:
- Content
- ToolCallRequest
- ToolCallResponse
- ToolCallConfirmation
- UserCancelled
- Error
- ChatCompressed
- Thought
- MaxSessionTurns
- Finished
- LoopDetected

### Configuration System (packages/core/src/config/config.ts)

#### Config
Main configuration class managing all settings.

Key properties:
- sessionId: string
- model: string
- embeddingModel: string
- sandbox?: SandboxConfig
- targetDir: string
- debugMode: boolean
- approvalMode: ApprovalMode
- telemetrySettings: TelemetrySettings
- mcpServers?: Record<string, MCPServerConfig>

#### ConfigParameters
Interface for Config constructor parameters.

#### MCPServerConfig
Configuration for MCP (Model Context Protocol) servers:
- Transport options: command/args, url, httpUrl, tcp
- Auth: oauth?, authProviderType?
- Metadata: description, includeTools, excludeTools

#### ApprovalMode
Enum for tool execution approval modes:
- DEFAULT - Standard confirmations
- AUTO_EDIT - Auto-approve edits
- YOLO - Auto-approve everything

### Telemetry Types (packages/core/src/telemetry/types.ts)

#### TelemetryEvent
Union type for all telemetry events:
- StartSessionEvent - Session initialization
- EndSessionEvent - Session termination
- UserPromptEvent - User inputs
- ToolCallEvent - Tool executions
- ApiRequestEvent - API requests
- ApiResponseEvent - API responses
- FlashFallbackEvent - Model fallbacks
- LoopDetectedEvent - Loop detection
- SlashCommandEvent - Command usage

## CLI Package Interfaces

### UI Types (packages/cli/src/ui/types.ts)

#### StreamingState
Enum for UI streaming states:
- Idle
- Responding
- WaitingForConfirmation

#### HistoryItem
Union type for conversation history items:
- HistoryItemUser - User messages
- HistoryItemGemini - AI responses
- HistoryItemInfo - System information
- HistoryItemError - Error messages
- HistoryItemAbout - About information
- HistoryItemStats - Session statistics
- HistoryItemModelStats - Model usage stats
- HistoryItemToolStats - Tool usage stats
- HistoryItemToolGroup - Grouped tool calls
- HistoryItemCompression - Compression events

#### ToolCallStatus
Enum for tool execution status:
- Pending
- Canceled
- Confirming
- Executing
- Success
- Error

### Command System Types (packages/cli/src/ui/commands/types.ts)

#### SlashCommand
Interface for command definitions:
- name: string - Primary command name
- altNames?: string[] - Alternative names
- description: string - Help text
- kind: CommandKind - Command type
- action?: (context, args) => SlashCommandActionReturn
- completion?: (context, partial) => Promise<string[]>
- subCommands?: SlashCommand[]

#### CommandContext
Context object passed to command actions:
- invocation: { raw, name, args }
- services: { config, settings, git, logger }
- ui: { addItem, clear, setDebugMessage, etc. }
- session: { stats, sessionShellAllowlist }

#### SlashCommandActionReturn
Union type for command action results:
- ToolActionReturn - Schedule tool execution
- MessageActionReturn - Display message
- QuitActionReturn - Exit application
- OpenDialogActionReturn - Show dialog
- LoadHistoryActionReturn - Replace history
- SubmitPromptActionReturn - Submit to AI
- ConfirmShellCommandsActionReturn - Confirm shells

### Service Interfaces (packages/cli/src/services/types.ts)

#### ICommandLoader
Interface for command loading sources:
- loadCommands(signal: AbortSignal): Promise<SlashCommand[]>

## Additional Core Types

### ToolRegistry (packages/core/src/tools/tool-registry.ts)
Manages tool registration and discovery.

### PromptRegistry (packages/core/src/prompts/prompt-registry.ts)
Manages prompt templates and MCP prompts.

### FileDiscoveryService (packages/core/src/services/fileDiscoveryService.ts)
Handles file system discovery with gitignore support.

### GitService (packages/core/src/services/gitService.ts)
Provides git repository operations.

### GeminiClient (packages/core/src/core/client.ts)
Main client for Gemini API interactions.

### GeminiChat (packages/core/src/core/geminiChat.ts)
Manages conversation state and history.

## Theme System Types

### Theme Interfaces (packages/cli/src/ui/themes/theme.ts)

#### ColorsTheme
Interface for theme color definitions:
- type: ThemeType ('light' | 'dark' | 'ansi' | 'custom')
- Background: string
- Foreground: string
- LightBlue: string
- AccentBlue: string
- AccentPurple: string
- AccentCyan: string
- AccentGreen: string
- AccentYellow: string
- AccentRed: string
- DiffAdded: string
- DiffRemoved: string
- Comment: string
- Gray: string
- GradientColors?: string[]

#### Theme
Class for managing syntax highlighting themes:
- name: string
- type: ThemeType
- defaultColor: string
- getInkColor(hljsClass: string): string | undefined

## OAuth and MCP Types

### OAuth Types (packages/core/src/mcp/oauth-provider.ts)

#### MCPOAuthConfig
Configuration for MCP server OAuth:
- enabled?: boolean
- clientId?: string
- clientSecret?: string
- authorizationUrl?: string
- tokenUrl?: string
- scopes?: string[]
- redirectUri?: string
- tokenParamName?: string

#### OAuthTokenResponse
OAuth token response structure:
- access_token: string
- token_type: string
- expires_in?: number
- refresh_token?: string
- scope?: string

#### OAuthClientRegistrationResponse
Dynamic client registration result:
- client_id: string
- client_secret?: string
- client_id_issued_at?: number
- client_secret_expires_at?: number
- redirect_uris: string[]
- grant_types: string[]
- response_types: string[]
- token_endpoint_auth_method: string