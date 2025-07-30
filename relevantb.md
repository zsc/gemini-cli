# Configuration Information Collection for Appendix B

This file collects all configuration-related information from the Gemini CLI codebase.

## 1. Command-line Options (from packages/cli/src/config/config.ts)
- `--model, -m`: Model to use (default: process.env.GEMINI_MODEL || 'gemini-2.5-pro')
- `--prompt, -p`: Non-interactive mode prompt
- `--prompt-interactive, -i`: Execute prompt and continue in interactive mode
- `--sandbox, -s`: Run in sandbox mode
- `--sandbox-image`: Sandbox image URI
- `--debug, -d`: Run in debug mode (default: false)
- `--all-files, -a`: Include ALL files in context (default: false)
- `--show-memory-usage`: Show memory usage in status bar
- `--yolo, -y`: Automatically accept all actions (YOLO mode)
- `--telemetry`: Enable telemetry
- `--telemetry-target`: Set telemetry target (local or gcp)
- `--telemetry-otlp-endpoint`: Set OTLP endpoint for telemetry
- `--telemetry-log-prompts`: Enable/disable logging of user prompts
- `--telemetry-outfile`: Redirect telemetry output to file
- `--checkpointing, -c`: Enables checkpointing of file edits
- `--experimental-acp`: Starts the agent in ACP mode
- `--allowed-mcp-server-names`: Allowed MCP server names
- `--extensions, -e`: List of extensions to use
- `--list-extensions, -l`: List all available extensions and exit
- `--ide-mode`: Run in IDE mode
- `--proxy`: Proxy for gemini client
- `--version, -v`: Show version
- `--help, -h`: Show help

## 2. Environment Variables
### Authentication
- `GEMINI_API_KEY`: API key for Gemini
- `GOOGLE_API_KEY`: API key for Google (Vertex AI express mode)
- `GOOGLE_CLOUD_PROJECT`: GCP project for Vertex AI
- `GOOGLE_CLOUD_LOCATION`: GCP location for Vertex AI
- `GOOGLE_GENAI_USE_VERTEXAI`: Use Vertex AI
- `GOOGLE_GENAI_USE_GCA`: Use Google Cloud Auth
- `CLOUD_SHELL`: Running in Cloud Shell

### Model
- `GEMINI_MODEL`: Default model to use

### Debugging
- `DEBUG`: Enable debug mode
- `DEBUG_MODE`: Alternative debug mode flag
- `DEBUG_PORT`: Port for debugging (default: 9229)

### Telemetry
- `OTEL_EXPORTER_OTLP_ENDPOINT`: OpenTelemetry endpoint
- `OTLP_GOOGLE_CLOUD_PROJECT`: GCP project for telemetry

### Proxy
- `HTTPS_PROXY`: HTTPS proxy
- `https_proxy`: HTTPS proxy (lowercase)
- `HTTP_PROXY`: HTTP proxy
- `http_proxy`: HTTP proxy (lowercase)

### Sandbox
- `GEMINI_SANDBOX`: Sandbox command (docker/podman/sandbox-exec)
- `GEMINI_SANDBOX_IMAGE`: Docker/Podman image for sandbox
- `GEMINI_SANDBOX_IMAGE_TAG`: Tag for sandbox image
- `SANDBOX`: Indicates running inside sandbox
- `SEATBELT_PROFILE`: macOS sandbox profile
- `BUILD_SANDBOX`: Build sandbox during build process
- `BUILD_SANDBOX_FLAGS`: Flags for building sandbox

### Other
- `NO_BROWSER`: Disable browser launch
- `TERM_PROGRAM`: Terminal program (used for IDE mode detection)
- `GEMINI_CLI_SYSTEM_SETTINGS_PATH`: Override system settings path
- `IS_DOCKER`: Running in Docker
- `IS_NIGHTLY`: Nightly build
- `MANUAL_VERSION`: Manual version override
- `CLI_VERSION`: CLI version
- `VERBOSE`: Verbose output
- `KEEP_OUTPUT`: Keep test output

## 3. Settings File Schema (from packages/cli/src/config/settings.ts)

### Settings Interface:
```typescript
interface Settings {
  theme?: string;
  customThemes?: Record<string, CustomTheme>;
  selectedAuthType?: AuthType;
  sandbox?: boolean | string;
  coreTools?: string[];
  excludeTools?: string[];
  toolDiscoveryCommand?: string;
  toolCallCommand?: string;
  mcpServerCommand?: string;
  mcpServers?: Record<string, MCPServerConfig>;
  allowMCPServers?: string[];
  excludeMCPServers?: string[];
  showMemoryUsage?: boolean;
  contextFileName?: string | string[];
  accessibility?: AccessibilitySettings;
  telemetry?: TelemetrySettings;
  usageStatisticsEnabled?: boolean;
  preferredEditor?: string;
  bugCommand?: BugCommandSettings;
  checkpointing?: CheckpointingSettings;
  autoConfigureMaxOldSpaceSize?: boolean;
  fileFiltering?: {
    respectGitIgnore?: boolean;
    respectGeminiIgnore?: boolean;
    enableRecursiveFileSearch?: boolean;
  };
  hideWindowTitle?: boolean;
  hideTips?: boolean;
  hideBanner?: boolean;
  maxSessionTurns?: number;
  summarizeToolOutput?: Record<string, SummarizeToolOutputSettings>;
  vimMode?: boolean;
  ideMode?: boolean;
  disableAutoUpdate?: boolean;
  memoryDiscoveryMaxDirs?: number;
}
```

### Sub-interfaces:
```typescript
interface AccessibilitySettings {
  disableLoadingPhrases?: boolean;
}

interface TelemetrySettings {
  enabled?: boolean;
  target?: 'local' | 'gcp';
  otlpEndpoint?: string;
  logPrompts?: boolean;
  outfile?: string;
}

interface CheckpointingSettings {
  enabled?: boolean;
}

interface SummarizeToolOutputSettings {
  tokenBudget?: number;
}

interface BugCommandSettings {
  urlTemplate: string;
}

interface MCPServerConfig {
  command?: string;
  args?: string[];
  env?: Record<string, string>;
  cwd?: string;
  url?: string;
  httpUrl?: string;
  headers?: Record<string, string>;
  tcp?: string;
  timeout?: number;
  trust?: boolean;
  description?: string;
  includeTools?: string[];
  excludeTools?: string[];
  extensionName?: string;
  oauth?: MCPOAuthConfig;
  authProviderType?: 'dynamic_discovery' | 'google_credentials';
}
```

## 4. Settings File Locations
- System: 
  - macOS: `/Library/Application Support/GeminiCli/settings.json`
  - Windows: `C:\ProgramData\gemini-cli\settings.json`
  - Linux: `/etc/gemini-cli/settings.json`
  - Override: `GEMINI_CLI_SYSTEM_SETTINGS_PATH`
- User: `~/.gemini/settings.json`
- Workspace: `<workspace>/.gemini/settings.json`

## 5. Settings Priority (highest to lowest)
1. System settings
2. Workspace settings
3. User settings
4. Command-line arguments
5. Environment variables

## 6. Auth Types (from packages/core/src/core/contentGenerator.ts)
- `oauth-personal`: Login with Google
- `gemini-api-key`: Use Gemini API key
- `vertex-ai`: Use Vertex AI
- `cloud-shell`: Cloud Shell authentication

## 7. Default Values
- Model: `gemini-2.5-pro`
- Flash Model: `gemini-2.5-flash`
- Flash Lite Model: `gemini-2.5-flash-lite`
- Embedding Model: `gemini-embedding-001`
- Telemetry Target: `local`
- OTLP Endpoint: `http://localhost:4317`
- Debug Port: `9229`

## 8. File Filtering Defaults
- For memory files:
  - respectGitIgnore: false
  - respectGeminiIgnore: true
- For all other files:
  - respectGitIgnore: true
  - respectGeminiIgnore: true

## 9. Sandbox Configuration
- Valid commands: `docker`, `podman`, `sandbox-exec`
- Default image: `gcr.io/gemini-cli/gemini-cli:latest`

## 10. .env File Discovery
- Searches from current directory up to root
- Prefers `.gemini/.env` over `.env`
- Falls back to `~/.gemini/.env` or `~/.env`

## 11. Environment Variable Resolution in Settings
- Supports `$VAR_NAME` and `${VAR_NAME}` syntax in settings files
- Works in strings, arrays, and nested objects