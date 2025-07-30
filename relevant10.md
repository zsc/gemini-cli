# Chapter 10: MCP Protocol Integration - Relevant Information

## MCP Architecture Overview

### Core Components

1. **MCP Client (mcp-client.ts)**
   - Main orchestrator for MCP server connections
   - Handles server discovery, connection, and tool/prompt discovery
   - Manages server statuses (CONNECTED, CONNECTING, DISCONNECTED)
   - Supports multiple transport types: Stdio, SSE, HTTP
   - Built on @modelcontextprotocol/sdk

2. **MCP Tool System (mcp-tool.ts)**
   - DiscoveredMCPTool class extends BaseTool
   - Wraps MCP tools as Gemini-compatible tools
   - Handles tool name validation and transformation
   - Implements confirmation mechanism for untrusted servers
   - Converts MCP responses to Gemini-compatible format

3. **OAuth Integration**
   - Full OAuth 2.0 support with PKCE
   - Dynamic client registration
   - Token storage and management
   - Auto-discovery via well-known endpoints
   - Refresh token support

### Key Features

#### Server Discovery and Connection
- Supports three transport types:
  - **Stdio**: For local command-based servers
  - **SSE (Server-Sent Events)**: For streaming connections
  - **HTTP**: For request-response style servers
- Parallel discovery for multiple servers
- Connection status tracking and event listeners

#### Tool Discovery
- Automatic tool discovery from connected servers
- Tool filtering via includeTools/excludeTools config
- Name sanitization for Gemini API compatibility
- Trust model with per-server and per-tool allowlisting

#### Prompt System
- MCP prompts become slash commands automatically
- Argument parsing with named and positional parameters
- Help subcommand generation
- Integration with CLI command system

#### OAuth Flow
1. **Auto-discovery**: Detects 401 errors and WWW-Authenticate headers
2. **Configuration discovery**: Via .well-known endpoints
3. **Dynamic registration**: If no client_id provided
4. **PKCE flow**: Secure authorization code exchange
5. **Token management**: Secure storage in ~/.gemini/mcp-oauth-tokens.json
6. **Token refresh**: Automatic refresh when expired

### Configuration

```typescript
interface MCPServerConfig {
  // For stdio transport
  command?: string;
  args?: string[];
  env?: Record<string, string>;
  cwd?: string;
  
  // For SSE transport
  url?: string;
  
  // For HTTP transport
  httpUrl?: string;
  
  // Common options
  headers?: Record<string, string>;
  timeout?: number;
  trust?: boolean;
  includeTools?: string[];
  excludeTools?: string[];
  
  // OAuth configuration
  oauth?: MCPOAuthConfig;
  authProviderType?: AuthProviderType;
}
```

### CLI Integration

#### /mcp Command
- `/mcp` or `/mcp list`: Shows all MCP servers and their tools
- `/mcp desc`: Shows descriptions
- `/mcp schema`: Shows parameter schemas
- `/mcp auth <server>`: Initiates OAuth authentication
- `/mcp refresh`: Re-discovers tools from all servers

#### Status Indicators
- 🟢 Connected and ready
- 🔄 Connecting/starting up
- 🔴 Disconnected
- OAuth status displayed inline

### Error Handling
- Network errors with helpful messages
- OAuth errors with authentication hints
- Sandbox-aware error messages
- Graceful degradation for missing servers

### Security Model
1. **Trust levels**:
   - Trusted servers (no confirmation needed)
   - Per-server allowlisting
   - Per-tool allowlisting
2. **OAuth token security**:
   - Tokens stored with 0600 permissions
   - Automatic expiry handling
   - Refresh token support
3. **Confirmation dialogs**:
   - Shows server and tool names
   - Options to trust always

### Integration Flow
1. Config loads MCP server definitions
2. discoverMcpTools() called during startup
3. Each server connected in parallel
4. Tools and prompts discovered from each server
5. Tools registered in ToolRegistry
6. Prompts registered in PromptRegistry
7. CLI commands created from prompts
8. Status tracking for UI updates

### Testing
- Integration test demonstrates simple MCP server
- Uses @modelcontextprotocol/sdk for server implementation
- Tests tool invocation through CLI
