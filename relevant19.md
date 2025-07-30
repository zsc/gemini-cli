# Chapter 19: Extension Development Guide - Relevant Information

This file collects relevant information for writing Chapter 19: Extension Development Guide.

## Extension System Architecture

### Extension Loading (packages/cli/src/config/extension.ts)
- Extensions are loaded from `.gemini/extensions/` directory in both workspace and home directory
- Each extension must have a `gemini-extension.json` config file
- Extension config structure:
  ```typescript
  interface ExtensionConfig {
    name: string;
    version: string;
    mcpServers?: Record<string, MCPServerConfig>;
    contextFileName?: string | string[];  // defaults to 'GEMINI.md'
    excludeTools?: string[];
  }
  ```
- Extensions can provide:
  - MCP servers for tools
  - Context files (like GEMINI.md)
  - Tool exclusions

### Tool Registry System (packages/core/src/tools/tool-registry.ts)
- ToolRegistry manages all tools (built-in, discovered, MCP)
- Tools can be:
  - Built-in tools (registered directly)
  - Discovered tools (from command discovery)
  - MCP tools (from MCP servers)
- Tool discovery via commands returns FunctionDeclaration[]
- Schema sanitization for Gemini API compatibility

## Custom Tool Development

### Tool Interface (packages/core/src/tools/tools.ts)
- Tools implement the Tool interface with:
  - name, displayName, description
  - icon, schema (FunctionDeclaration)
  - execute() method
  - Optional: ModifiableTool interface for tools that support editing

### Discovered Tool Pattern (tool-registry.ts)
- Tools can be discovered via command line
- Discovery command configured in settings
- Returns JSON array of FunctionDeclaration objects
- Tool execution via subprocess with JSON I/O

### ModifiableTool Pattern (modifiable-tool.ts)
- Allows tools to support external editor modification
- Provides diff view for user to edit proposed changes
- Used by file editing tools

## MCP Server Integration

### MCP Tool Discovery (mcp-client.ts)
- MCP servers can be configured in settings or extensions
- Discovery process:
  - Connect to MCP server via stdio, SSE, or HTTP transport
  - Discover tools and prompts from server
  - Handle OAuth authentication if required
  - Register tools in ToolRegistry
- MCP tools wrapped in DiscoveredMCPTool class
- Trust model: untrusted tools require user confirmation

### MCP Server Configuration
```typescript
interface MCPServerConfig {
  command?: string;       // for stdio transport
  url?: string;          // for SSE/HTTP transport
  environment?: Record<string, string>;
  authProvider?: AuthProviderType;
  trust?: boolean;
  timeout?: number;
}
```

### MCP Tool Implementation (mcp-tool.ts)
- Tools from MCP servers are wrapped as DiscoveredMCPTool
- Handles parameter schema conversion for Gemini API
- Supports trust/allowlist for auto-approval
- Tool naming: can be simple or fully qualified (serverName__toolName)

## Command Extension

### Command System Architecture (CommandService.ts)
- Provider-based loader pattern
- Multiple loaders: BuiltinCommandLoader, FileCommandLoader, McpPromptLoader
- Conflict resolution:
  - Extension commands renamed if conflict (extensionName.commandName)
  - Non-extension commands use "last wins" strategy

### File-based Commands (FileCommandLoader.ts)
- Commands defined in TOML files
- Locations:
  - User: ~/.gemini/commands/
  - Project: .gemini/commands/
  - Extensions: extension/commands/
- TOML format:
  ```toml
  prompt = "Your prompt template"
  description = "Optional description"
  ```
- Support for:
  - Argument placeholders: {{args}}
  - Shell command injection: $(command)
  - Nested folder structure becomes namespaced commands

### Command Processors
- ArgumentProcessor: handles {{args}} placeholder
- ShellProcessor: executes $(command) and injects results
- Extensible processor pipeline

## Configuration Extension

### Extension Configuration Structure
- Extensions can provide configuration via gemini-extension.json
- Configuration can include:
  - MCP servers
  - Context files (GEMINI.md variations)
  - Tool exclusions
- Extensions loaded from:
  - Workspace: .gemini/extensions/
  - Home: ~/.gemini/extensions/

### Settings Integration
- Extensions can be enabled/disabled via settings
- Extension names used for namespacing conflicts
- Active extensions tracked in Config

## IDE Integration

### VSCode IDE Companion (vscode-ide-companion)
- Provides IDE context to Gemini CLI
- Features:
  - Sets GEMINI_CLI_IDE_WORKSPACE_PATH environment variable
  - Runs IDE server for file context
  - Command to launch Gemini CLI in terminal
- Communication via IDE server protocol

### IDE Context System
- IDE can provide:
  - Open files list
  - Current file selections
  - Workspace structure
- Used by Gemini to understand current development context

## Best Practices and Examples

### Extension Development Workflow
1. Create extension directory structure
2. Add gemini-extension.json configuration
3. Implement tools/commands/MCP servers
4. Test in local .gemini/extensions/
5. Package and distribute

### Common Extension Patterns
- Tool extensions via MCP servers
- Context augmentation via GEMINI.md files
- Custom commands via TOML files
- IDE integrations for specific editors