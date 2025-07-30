# Chapter 14: Tool Execution Flow - Relevant Information

## Overview
This file collects relevant information about tool execution flow in Gemini CLI, including:
- Tool request from AI to execution completion
- Parameter validation
- User confirmation
- Parallel execution
- Result feedback

## Key Files to Examine
1. Core tool system files:
   - packages/core/src/tools/tool-registry.ts
   - packages/core/src/core/coreToolScheduler.ts
   - packages/core/src/tools/modifiable-tool.ts
   - packages/core/src/core/nonInteractiveToolExecutor.ts

2. CLI tool scheduling:
   - packages/cli/src/ui/hooks/useToolScheduler.ts
   - packages/cli/src/ui/hooks/useReactToolScheduler.ts

3. Tool implementations:
   - packages/core/src/tools/*.ts (various tool implementations)

4. Tool confirmation and UI:
   - packages/cli/src/ui/components/messages/ToolConfirmationMessage.tsx
   - packages/cli/src/ui/components/messages/ToolMessage.tsx
   - packages/cli/src/ui/components/messages/ToolGroupMessage.tsx

## Information Collection

### 1. Tool Registry (packages/core/src/tools/tool-registry.ts)
- Central registry for all tools in the system
- Manages tool discovery (command-based and MCP-based)
- Tool registration and retrieval
- Function declaration generation for Gemini API
- Discovered tools execute via subprocess with JSON parameters
- Parameter sanitization for Gemini API compatibility
- Support for overwriting existing tools

### 2. CoreToolScheduler (packages/core/src/core/coreToolScheduler.ts)
- Central orchestrator for tool execution lifecycle
- States: validating -> awaiting_approval/scheduled -> executing -> success/error/cancelled
- Handles parallel execution of multiple tools
- User confirmation workflow with ApprovalMode (YOLO mode skips confirmation)
- Support for modifiable tools with editor integration
- Live output streaming for tools that support it
- Converts tool results to FunctionResponse format for Gemini
- Prevents scheduling new tools while others are running
- Tracks execution duration and outcomes

### 3. React Tool Scheduler Hook (packages/cli/src/ui/hooks/useReactToolScheduler.ts)
- React integration layer for CoreToolScheduler
- Manages tool call display state with additional UI metadata
- Maps core tool states to UI display states
- Tracks whether tool responses have been submitted to Gemini
- Handles live output updates for streaming tools
- Provides schedule function and markToolsAsSubmitted for UI control
- Transforms tool calls into display-friendly format with proper names/descriptions

### 4. Modifiable Tool System (packages/core/src/tools/modifiable-tool.ts)
- Interface for tools that support user modification before execution
- ModifyContext provides methods for getting/setting content
- Opens external editor (VS Code, vim, etc.) for user to modify proposed changes
- Creates temp files for diff viewing and editing
- Returns updated parameters and diff after user edits
- Used primarily for file editing tools where users want to review/modify changes

### 5. Tool Confirmation UI (packages/cli/src/ui/components/messages/ToolConfirmationMessage.tsx)
- React component for displaying tool confirmation prompts
- Handles different confirmation types: edit, exec, info, mcp
- Edit confirmations show diff with syntax highlighting
- Execution confirmations show command to be run
- Provides options: allow once, allow always, modify with editor, cancel
- Escape key cancels the operation
- Special handling for "modify in progress" state

### 6. Tool Base Classes and Types (packages/core/src/tools/tools.ts)
- Tool interface defines contract for all tools
- BaseTool provides default implementations
- Key methods: validateToolParams, getDescription, shouldConfirmExecute, execute
- ToolResult contains llmContent (for AI) and returnDisplay (for user)
- ToolConfirmationOutcome enum defines user choices
- Different confirmation detail types for different tool categories
- Support for tool locations (affected file paths)

### 7. Edit Tool Example (packages/core/src/tools/edit.ts)
- Concrete implementation of ModifiableTool
- Performs text replacements in files
- Validates file paths and ensures within root directory
- calculateEdit method computes the changes before execution
- Supports creating new files (empty old_string)
- Provides detailed diff for user confirmation
- Handles multiple replacements with expected_replacements parameter
- User can modify the proposed changes before execution

### 8. Turn and Event Types (packages/core/src/core/turn.ts)
- Defines event types for tool execution flow
- ToolCallRequestInfo: contains callId, name, args, prompt_id
- ToolCallResponseInfo: contains results and display information
- GeminiEventType enum defines all possible events
- ServerGeminiToolCallRequestEvent: emitted when AI requests tool
- ServerGeminiToolCallResponseEvent: emitted when tool completes
- ServerGeminiToolCallConfirmationEvent: emitted when confirmation needed
- handlePendingFunctionCall processes Gemini API function calls
- Generates unique callId if not provided
- Emits tool call request events for downstream processing

### 9. Non-Interactive Tool Executor (packages/core/src/core/nonInteractiveToolExecutor.ts)
- Simplified tool execution for non-interactive contexts
- No confirmation prompts or live output
- Direct tool execution with error handling
- Logs tool calls with telemetry
- Converts tool results to FunctionResponse format
- Used in batch processing or automated workflows