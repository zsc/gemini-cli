# Chapter 6: Configuration System - Relevant Information

## Configuration Files to Examine

### Core Package Configuration
- packages/core/src/config/config.ts
- packages/core/src/config/config.test.ts
- packages/core/src/config/models.ts
- packages/core/src/config/flashFallback.test.ts

### CLI Package Configuration
- packages/cli/src/config/config.ts
- packages/cli/src/config/config.test.ts
- packages/cli/src/config/config.integration.test.ts
- packages/cli/src/config/auth.ts
- packages/cli/src/config/auth.test.ts
- packages/cli/src/config/settings.ts
- packages/cli/src/config/settings.test.ts
- packages/cli/src/config/extension.ts
- packages/cli/src/config/extension.test.ts
- packages/cli/src/config/sandboxConfig.ts

## Key Concepts to Cover
1. Multi-layer configuration system
2. Configuration sources and priority
3. Configuration merging strategies
4. Dynamic loading mechanisms
5. Settings management
6. Authentication configuration
7. Extension configuration
8. Sandbox configuration
9. Environment variables
10. Default values and validation

## Core Config Class Structure (packages/core/src/config/config.ts)

### Key Properties:
- sessionId, model, embeddingModel
- targetDir, debugMode, cwd
- sandbox configuration
- tool registry and settings (coreTools, excludeTools)
- MCP server configurations
- memory settings (userMemory, geminiMdFileCount)
- approval mode settings
- telemetry settings
- file filtering options
- authentication configuration
- extension settings
- IDE mode settings

### Key Methods:
- initialize(): async initialization
- refreshAuth(): refresh authentication
- getters for all configuration values
- createToolRegistry(): creates and registers tools
- flash fallback handler support

### Configuration Parameters Interface:
```typescript
interface ConfigParameters {
  sessionId: string;
  embeddingModel?: string;
  sandbox?: SandboxConfig;
  targetDir: string;
  debugMode: boolean;
  question?: string;
  fullContext?: boolean;
  coreTools?: string[];
  excludeTools?: string[];
  toolDiscoveryCommand?: string;
  toolCallCommand?: string;
  mcpServerCommand?: string;
  mcpServers?: Record<string, MCPServerConfig>;
  userMemory?: string;
  geminiMdFileCount?: number;
  approvalMode?: ApprovalMode;
  showMemoryUsage?: boolean;
  contextFileName?: string | string[];
  accessibility?: AccessibilitySettings;
  telemetry?: TelemetrySettings;
  usageStatisticsEnabled?: boolean;
  fileFiltering?: FileFilteringOptions;
  checkpointing?: boolean;
  proxy?: string;
  cwd: string;
  fileDiscoveryService?: FileDiscoveryService;
  bugCommand?: BugCommandSettings;
  model: string;
  extensionContextFilePaths?: string[];
  maxSessionTurns?: number;
  experimentalAcp?: boolean;
  listExtensions?: boolean;
  extensions?: GeminiCLIExtension[];
  blockedMcpServers?: Array<{ name: string; extensionName: string }>;
  noBrowser?: boolean;
  summarizeToolOutput?: Record<string, SummarizeToolOutputSettings>;
  ideMode?: boolean;
  ideClient?: IdeClient;
}
```

## CLI Config Loading (packages/cli/src/config/config.ts)

### Command Line Arguments:
- model, prompt, prompt-interactive
- sandbox, sandbox-image
- debug, all-files, show-memory-usage
- yolo (auto-accept mode)
- telemetry settings
- checkpointing
- experimental-acp
- allowed-mcp-server-names
- extensions, list-extensions
- ide-mode
- proxy

### Configuration Loading Process:
1. Parse command line arguments
2. Load extensions and determine active ones
3. Set context filename for memory
4. Create FileDiscoveryService
5. Load hierarchical memory (GEMINI.md files)
6. Merge MCP servers from settings and extensions
7. Process allowed/excluded MCP servers
8. Load sandbox configuration
9. Create Config instance with merged settings

## Settings System (packages/cli/src/config/settings.ts)

### Settings Hierarchy (Priority order - highest to lowest):
1. System settings (/Library/Application Support/GeminiCli/settings.json on macOS)
2. Workspace settings (.gemini/settings.json in project)
3. User settings (~/.gemini/settings.json)

### Key Settings:
- theme and customThemes
- selectedAuthType
- sandbox configuration
- tool settings (coreTools, excludeTools)
- MCP server configurations
- telemetry settings
- file filtering options
- editor preferences
- accessibility settings
- vim mode
- IDE mode settings

### Environment Variable Support:
- Settings values can reference environment variables using $VAR_NAME or ${VAR_NAME}
- .env file discovery (searches up directory tree)
- Special handling for Cloud Shell environment

### Settings Loading Process:
1. Load environment variables (.env files)
2. Load and parse system settings
3. Load and parse user settings
4. Load and parse workspace settings
5. Resolve environment variables in all settings
6. Merge settings with proper precedence
7. Handle legacy theme name conversions

### LoadedSettings Class:
- Maintains separate system, user, and workspace settings
- Provides merged view of all settings
- Supports updating settings by scope
- Automatically saves changes

## Authentication Configuration (packages/cli/src/config/auth.ts)

### Auth Methods:
1. LOGIN_WITH_GOOGLE - OAuth login flow
2. CLOUD_SHELL - Automatic in Google Cloud Shell
3. USE_GEMINI - API key authentication (requires GEMINI_API_KEY env var)
4. USE_VERTEX_AI - Vertex AI authentication (requires GOOGLE_CLOUD_PROJECT and GOOGLE_CLOUD_LOCATION, or GOOGLE_API_KEY)

### Validation:
- Checks for required environment variables based on auth method
- Returns specific error messages for missing configuration
- Environment loaded before validation

## Extension System (packages/cli/src/config/extension.ts)

### Extension Discovery:
- Extensions loaded from `.gemini/extensions/` directories
- Searches both workspace and home directory
- Each extension requires `gemini-extension.json` config file

### Extension Configuration:
```typescript
interface ExtensionConfig {
  name: string;
  version: string;
  mcpServers?: Record<string, MCPServerConfig>;
  contextFileName?: string | string[];
  excludeTools?: string[];
}
```

### Extension Loading Process:
1. Scan extension directories in workspace and home
2. Load and validate extension configs
3. Resolve context files (defaults to GEMINI.md)
4. Deduplicate extensions by name
5. Annotate active/inactive status based on CLI args

### Extension Integration:
- MCP servers from extensions merged with settings
- Extension context files added to memory
- Tool exclusions aggregated across extensions

## Sandbox Configuration (packages/cli/src/config/sandboxConfig.ts)

### Sandbox Commands:
- docker - Container-based sandboxing
- podman - Alternative container runtime
- sandbox-exec - macOS native sandboxing

### Sandbox Resolution Order:
1. Check if already in sandbox (SANDBOX env var)
2. Check GEMINI_SANDBOX environment variable
3. Check command line arguments
4. Check settings file
5. Auto-detect available commands (sandbox-exec > docker > podman)

### Sandbox Configuration:
- command: The sandbox runtime to use
- image: Container image URI (for docker/podman)
- Image source priority: CLI arg > env var > package.json config

### Error Handling:
- Validates sandbox command is supported
- Checks command exists on system
- Clear error messages for missing dependencies

## Configuration Constants and Defaults

### Model Defaults (packages/core/src/config/models.ts):
- DEFAULT_GEMINI_MODEL = 'gemini-2.5-pro'
- DEFAULT_GEMINI_FLASH_MODEL = 'gemini-2.5-flash'
- DEFAULT_GEMINI_FLASH_LITE_MODEL = 'gemini-2.5-flash-lite'
- DEFAULT_GEMINI_EMBEDDING_MODEL = 'gemini-embedding-001'

### Approval Modes:
- DEFAULT = 'default' - User confirms each tool execution
- AUTO_EDIT = 'autoEdit' - Auto-approve edit operations
- YOLO = 'yolo' - Auto-approve all operations

### File Filtering Defaults:
- DEFAULT_MEMORY_FILE_FILTERING_OPTIONS:
  - respectGitIgnore: false
  - respectGeminiIgnore: true
- DEFAULT_FILE_FILTERING_OPTIONS:
  - respectGitIgnore: true
  - respectGeminiIgnore: true

### Environment Variables:
- GEMINI_MODEL - Override default model
- GEMINI_API_KEY - API key for USE_GEMINI auth
- GOOGLE_CLOUD_PROJECT - GCP project for Vertex AI
- GOOGLE_CLOUD_LOCATION - GCP region for Vertex AI
- GOOGLE_API_KEY - Alternative auth for Vertex AI
- GEMINI_SANDBOX - Override sandbox settings
- GEMINI_SANDBOX_IMAGE - Container image for sandbox
- SANDBOX - Indicates running inside sandbox
- NO_BROWSER - Disable browser launch
- DEBUG / DEBUG_MODE - Enable debug mode
- HTTPS_PROXY / HTTP_PROXY - Proxy settings
- OTEL_EXPORTER_OTLP_ENDPOINT - Telemetry endpoint
- CLOUD_SHELL - Google Cloud Shell detection
- GEMINI_CLI_SYSTEM_SETTINGS_PATH - Override system settings path

## Configuration Flow Summary

### Initialization Process:
1. Parse command line arguments (yargs)
2. Load environment variables (.env files)
3. Load settings hierarchy (system > workspace > user)
4. Load and validate extensions
5. Merge configuration sources with proper precedence
6. Initialize core services (FileDiscoveryService, GitService)
7. Load hierarchical memory (GEMINI.md files)
8. Create Config instance with all merged settings

### Configuration Precedence (highest to lowest):
1. Command line arguments
2. Environment variables
3. System settings
4. Workspace settings (.gemini/settings.json)
5. User settings (~/.gemini/settings.json)
6. Extension configurations
7. Default values

### Dynamic Configuration:
- Settings can be updated at runtime via LoadedSettings.setValue()
- Authentication can be refreshed without restarting
- Model can be switched during session (e.g., flash fallback)
- Approval mode can be changed dynamically