# Relevant Information for Chapter 3: Tool System Architecture

## Tool System Files to Explore

### Core Tool Files (packages/core/src/tools/)
- tool-registry.ts - Tool registration and management
- tools.ts - Tool definitions and exports
- modifiable-tool.ts - Base classes for tools
- edit.ts, glob.ts, grep.ts, ls.ts - File operation tools
- shell.ts - Shell command execution
- mcp-tool.ts, mcp-client.ts - MCP integration
- memoryTool.ts - Memory management tool
- read-file.ts, write-file.ts - File I/O tools
- web-fetch.ts, web-search.ts - Web tools

### Core Scheduler Files (packages/core/src/core/)
- coreToolScheduler.ts - Tool scheduling and execution
- nonInteractiveToolExecutor.ts - Non-interactive tool execution

### CLI Tool Integration (packages/cli/src/)
- ui/hooks/useToolScheduler.ts - React hook for tool scheduling
- ui/hooks/useReactToolScheduler.ts - React-specific scheduler

## Information to Collect
1. Tool interface and base classes
2. Tool registration mechanism
3. Tool discovery process
4. Tool scheduling and execution flow
5. Tool result handling
6. Error handling in tools
7. Tool confirmation UI
8. Parallel tool execution
9. Tool cancellation
10. MCP tool integration

## Key Findings from Code Analysis

### 1. Tool Interface and Base Classes (tools.ts)

#### Tool Interface
- `Tool<TParams, TResult>` - Core interface for all tools
- Key properties:
  - `name`: Internal name used for API calls
  - `displayName`: User-friendly name
  - `description`: Tool description
  - `icon`: Icon for UI display
  - `schema`: FunctionDeclaration for Gemini API
  - `isOutputMarkdown`: Whether output renders as markdown
  - `canUpdateOutput`: Supports streaming output

#### Key Methods:
- `validateToolParams(params)`: Parameter validation
- `getDescription(params)`: Pre-execution description
- `toolLocations(params)`: Affected file paths
- `shouldConfirmExecute(params, signal)`: Confirmation check
- `execute(params, signal, updateOutput?)`: Tool execution

#### BaseTool Abstract Class
- Implements common functionality
- Generates schema from name, description, and parameterSchema
- Default implementations for validation and confirmation

#### Tool Result Structure
```typescript
interface ToolResult {
  summary?: string; // One-line summary
  llmContent: PartListUnion; // Content for LLM history
  returnDisplay: ToolResultDisplay; // User display (string or FileDiff)
}
```

#### Confirmation Details
- Multiple confirmation types: edit, exec, mcp, info
- ToolConfirmationOutcome enum:
  - ProceedOnce
  - ProceedAlways
  - ProceedAlwaysServer
  - ProceedAlwaysTool
  - ModifyWithEditor
  - Cancel

### 2. Tool Registry (tool-registry.ts)

#### ToolRegistry Class
- Central registry for all tools
- Manages tool registration and discovery
- Key methods:
  - `registerTool(tool)`: Register a tool
  - `discoverAllTools()`: Discover from command line and MCP
  - `discoverMcpTools()`: Discover only MCP tools
  - `discoverToolsForServer(serverName)`: Discover for specific MCP server
  - `getFunctionDeclarations()`: Get tool schemas for API
  - `getAllTools()`: Get all registered tools
  - `getTool(name)`: Get specific tool

#### Tool Discovery
1. **Command-line discovery**:
   - Executes tool discovery command from config
   - Parses JSON output for function declarations
   - Creates DiscoveredTool instances
   - Sanitizes parameters for Gemini API compatibility

2. **MCP discovery**:
   - Discovers from configured MCP servers
   - Creates DiscoveredMCPTool instances
   - Supports per-server discovery

#### DiscoveredTool Class
- Extends BaseTool
- Executes via subprocess with tool call command
- Handles stdout, stderr, exit codes, signals
- Returns structured error info on failure

#### Parameter Sanitization
- Removes unsupported properties for Gemini API
- Handles anyOf/default conflicts
- Ensures enum compatibility
- Processes nested schemas recursively

### 3. Core Tool Scheduler (coreToolScheduler.ts)

#### CoreToolScheduler Class
- Manages tool execution lifecycle
- Handles confirmation flow
- Tracks tool call states
- Supports parallel execution

#### Tool Call States
```typescript
type ToolCall = 
  | ValidatingToolCall      // Parameter validation
  | ScheduledToolCall       // Ready to execute
  | WaitingToolCall         // Awaiting user approval
  | ExecutingToolCall       // Currently running
  | SuccessfulToolCall      // Completed successfully
  | ErroredToolCall         // Failed with error
  | CancelledToolCall       // User cancelled
```

#### Key Features:
1. **State Management**:
   - Tracks all tool calls with status
   - Transitions through states based on events
   - Preserves execution history

2. **Confirmation Handling**:
   - Wraps tool confirmation callbacks
   - Supports different approval modes (YOLO mode skips)
   - Handles modify-with-editor flow

3. **Output Conversion**:
   - `convertToFunctionResponse()`: Formats tool output for Gemini API
   - Handles various content types (string, parts, binary)
   - Creates proper FunctionResponse parts

4. **Event Handlers**:
   - `outputUpdateHandler`: Live output updates
   - `onAllToolCallsComplete`: Batch completion
   - `onToolCallsUpdate`: State change notifications

#### Execution Flow:
1. Tool calls scheduled with `schedule()`
2. Validation occurs first
3. If approval needed, enters awaiting_approval state
4. On confirmation, moves to scheduled then executing
5. During execution, live output updates sent
6. Completes with success, error, or cancelled

### 4. Built-in Tools

#### EditTool (edit.ts)
- Implements ModifiableTool interface
- Replaces text within files
- Features:
  - Single or multiple replacements
  - User can modify proposed changes
  - Shows diff before applying
  - Ensures exact string matching
  - Requires significant context for uniqueness

#### File Operation Tools
- **ReadFileTool**: Reads file content with line numbers
- **WriteFileTool**: Creates/overwrites files
- **GlobTool**: Pattern-based file search
- **GrepTool**: Content search with regex
- **ListDirectoryTool**: Lists directory contents

#### Other Core Tools
- **ShellTool**: Execute shell commands
- **MemoryTool**: Manage context memory
- **WebFetchTool**: Fetch web content
- **WebSearchTool**: Search the web

### 5. MCP Tool Integration

#### DiscoveredMCPTool (mcp-tool.ts)
- Wrapper for MCP server tools
- Communicates via JSON-RPC
- Supports tool name disambiguation
- Handles server lifecycle

#### MCP Client (mcp-client.ts)
- Manages MCP server connections
- Discovers tools from servers
- Handles authentication (OAuth)
- Server process management

### 6. CLI Integration

#### useReactToolScheduler Hook
- React integration for tool scheduling
- Tracks tool display state
- Updates UI during execution
- Manages pending history items
- Converts core tool states to UI states

#### Tool Display States
```typescript
enum ToolCallStatus {
  Pending,
  Executing,
  Success,
  Error,
  Cancelled,
  AwaitingApproval
}
```

### 7. Tool Confirmation UI Flow

1. **Edit Confirmations**:
   - Shows file diff
   - User can proceed, cancel, or modify
   - Supports inline editing

2. **Execution Confirmations**:
   - Shows command to execute
   - Root command highlighted
   - Safety warnings

3. **MCP Confirmations**:
   - Shows server and tool name
   - Server-specific permissions

4. **Info Confirmations**:
   - General purpose prompts
   - URL access warnings