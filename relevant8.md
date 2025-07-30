# Chapter 8: Command System Design - Relevant Information

## Core Command System Architecture

### CommandService
- **File**: packages/cli/src/services/CommandService.ts
- **Purpose**: Orchestrates command discovery and loading
- **Key Features**:
  - Provider-based loader pattern
  - Parallel loading from multiple sources
  - Conflict resolution for extension commands
  - Command renaming strategy: `extensionName.commandName`

### Command Loaders

#### BuiltinCommandLoader
- **File**: packages/cli/src/services/BuiltinCommandLoader.ts
- **Purpose**: Loads hard-coded slash commands
- **Commands**: about, auth, bug, chat, clear, compress, copy, corgi, docs, editor, extensions, help, ide, init, mcp, memory, privacy, quit, restore, stats, theme, tools, vim

#### FileCommandLoader
- **File**: packages/cli/src/services/FileCommandLoader.ts
- **Purpose**: Loads custom commands from TOML files
- **Features**:
  - Scans user, project, and extension directories
  - TOML format validation with Zod schema
  - Supports prompt templates with placeholders
  - Shell command injection with `!{...}` syntax
  - Argument processing with `{{args}}` placeholder

#### McpPromptLoader
- **File**: packages/cli/src/services/McpPromptLoader.ts
- **Purpose**: Exposes MCP server prompts as commands
- **Features**:
  - Named and positional argument parsing
  - Auto-generated help subcommands
  - Prompt invocation and response handling

### Command Types and Context

#### SlashCommand Interface
- **File**: packages/cli/src/ui/commands/types.ts
- **Key Properties**:
  - name, altNames, description
  - kind: BUILT_IN, FILE, MCP_PROMPT
  - action: async function returning SlashCommandActionReturn
  - completion: optional argument completion
  - subCommands: nested command support
  - extensionName: for extension commands

#### CommandContext
- **Components**:
  - invocation: raw input, command name, args
  - services: config, settings, git, logger
  - ui: history management, dialogs, state updates
  - session: stats, shell allowlist

#### Command Action Returns
- tool: Schedule a tool execution
- message: Display info/error message
- dialog: Open help/auth/theme/editor/privacy dialog
- load_history: Replace conversation history
- submit_prompt: Send content to Gemini
- confirm_shell_commands: Request shell permission
- quit: Exit application

### Command Processing Pipeline

#### SlashCommandProcessor Hook
- **File**: packages/cli/src/ui/hooks/slashCommandProcessor.ts
- **Flow**:
  1. Command parsing and path resolution
  2. Primary name and alias matching
  3. Sub-command traversal
  4. Action execution with full context
  5. Result handling and state updates

### Prompt Processors

#### IPromptProcessor Interface
- **File**: packages/cli/src/services/prompt-processors/types.ts
- **Purpose**: Transform prompts in a pipeline
- **Placeholders**:
  - `{{args}}`: Argument injection
  - `!{...}`: Shell command execution

#### ShellProcessor
- **File**: packages/cli/src/services/prompt-processors/shellProcessor.ts
- **Features**:
  - Shell command permission checking
  - Session allowlist management
  - Confirmation flow for disallowed commands
  - Command output injection

## Command Discovery Order
1. MCP prompts (first)
2. Built-in commands (second)
3. File commands: User → Project → Extensions

## Conflict Resolution Strategy
- Extension commands renamed if conflicts exist
- Non-extension commands use "last wins" strategy
- Deterministic loading order for predictability