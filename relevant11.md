# VSCode Extension Implementation - Relevant Information

## Directory Structure
- packages/vscode-ide-companion/
  - src/
    - extension.ts - Main extension entry point
    - ide-server.ts - MCP server implementation
    - open-files-manager.ts - Tracks open files and selections
    - utils/logger.ts - Logging utilities

## Extension Architecture

### 1. Extension Activation (extension.ts)
- Activates on VSCode startup (`onStartupFinished`)
- Sets environment variables:
  - `GEMINI_CLI_IDE_WORKSPACE_PATH` - Current workspace path
  - `GEMINI_CLI_IDE_SERVER_PORT` - Dynamic port for MCP server
- Registers command: `gemini-cli.runGeminiCLI` to launch CLI
- Starts IDE server on activation

### 2. IDE Server (ide-server.ts)
- Implements MCP (Model Context Protocol) server
- Uses Express.js HTTP server with dynamic port allocation
- Key endpoints:
  - POST /mcp - MCP protocol endpoint
  - GET /mcp - Session management
- Session management with unique IDs
- Sends `ide/contextUpdate` notifications when files change
- Keep-alive mechanism (60s ping interval)

### 3. Open Files Manager (open-files-manager.ts)
- Tracks workspace state:
  - Open files (max 10 files)
  - Active file
  - Cursor position
  - Selected text (max 16KB)
- Event listeners:
  - onDidChangeActiveTextEditor
  - onDidChangeTextEditorSelection
  - onDidCloseTextDocument
  - onDidDeleteFiles
  - onDidRenameFiles
- Debounced change notifications (50ms)

### 4. CLI Integration (packages/core/src/ide/)
- ide-client.ts: CLI-side client for connecting to extension
  - Reads environment variables set by extension
  - Validates workspace path matches CLI directory
  - Establishes MCP connection
  - Handles connection states (Connected/Disconnected/Connecting)
- ideContext.ts: Context store for IDE state
  - Manages IdeContext data structure
  - Provides subscription mechanism
  - Used in prompt generation

## Communication Flow

1. VSCode Extension starts → Sets env vars → Starts MCP server
2. CLI launches → Reads env vars → Connects to MCP server
3. User interacts with VSCode → OpenFilesManager tracks changes
4. Changes trigger → MCP server sends notifications → CLI updates context
5. CLI includes IDE context in prompts to Gemini

## Key Features Enabled

1. **Open Files Context**: Gemini knows which files are open in VSCode
2. **Cursor Position**: Gemini knows where the cursor is located
3. **Selected Text**: Gemini can see what text is selected (up to 16KB)
4. **Workspace Awareness**: Ensures CLI runs in same directory as VSCode

## Protocol: MCP (Model Context Protocol)
- JSON-RPC based protocol
- Streamable HTTP transport
- Session-based communication
- Custom notification: `ide/contextUpdate`

## Environment Variables
- `GEMINI_CLI_IDE_WORKSPACE_PATH`: Set by extension, read by CLI
- `GEMINI_CLI_IDE_SERVER_PORT`: Dynamic port for MCP server

## Extension Manifest Highlights
- Publisher: google
- Categories: AI
- Activation: onStartupFinished
- Engine: VSCode ^1.101.0
- Dependencies: @modelcontextprotocol/sdk, express, cors, zod
