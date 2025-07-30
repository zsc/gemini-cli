# Chapter 18: Security and Privacy Related Information

## Authentication and Authorization

### Authentication Methods (from packages/cli/src/config/auth.ts)
- Login with Google
- Cloud Shell
- Use Gemini API (requires GEMINI_API_KEY)
- Use Vertex AI (requires GOOGLE_CLOUD_PROJECT and GOOGLE_CLOUD_LOCATION, or GOOGLE_API_KEY)

### OAuth Token Storage (from packages/core/src/mcp/oauth-token-storage.ts)
- Tokens stored in ~/.gemini/mcp-oauth-tokens.json
- File permissions restricted to 0o600 (read/write by owner only)
- Token expiration checking with 5-minute buffer
- Secure token management with encryption at rest

## Sandbox Execution

### Docker/Podman Sandbox (from packages/cli/src/utils/sandbox.ts)
- Isolation via Docker/Podman containers
- Volume mounting controls
- Network isolation options
- User ID/GID mapping for file permissions
- Environment variable filtering
- Proxy support for restricted environments

### macOS Seatbelt Sandbox
- Multiple security profiles:
  - permissive-open
  - permissive-closed
  - permissive-proxied
  - restrictive-open
  - restrictive-closed
  - restrictive-proxied
- Profile-based file system access control
- Network access restrictions

## Privacy Notices

### Gemini API Privacy (from packages/cli/src/ui/privacy/GeminiPrivacyNotice.tsx)
- Terms of Service compliance
- API usage agreement
- Data handling policies

## OAuth and Credential Management

### OAuth2 Implementation (from packages/core/src/code_assist/oauth2.ts)
- OAuth2 flow with PKCE (Proof Key for Code Exchange)
- State parameter for CSRF protection
- Secure credential storage in ~/.gemini/oauth_creds.json
- Automatic token refresh
- Support for multiple authentication methods:
  - Web-based OAuth flow
  - User code flow (for environments without browser)
  - Cloud Shell ADC (Application Default Credentials)
  - Service account authentication

### Security Features:
- Random state generation for OAuth flows
- CSRF protection via state validation
- Secure token storage with file system permissions
- Token expiration handling
- Proxy support for restricted networks

## Telemetry and Data Collection

### Telemetry Events (from packages/core/src/telemetry/constants.ts)
- User prompts
- Tool calls
- API requests/responses
- Configuration changes
- Command usage

### Privacy Considerations:
- No sensitive data in telemetry
- Anonymized usage metrics
- Opt-out mechanisms available

## Shell Execution Security

### Shell Execution Service (from packages/core/src/services/shellExecutionService.ts)
- Process isolation
- Controlled environment variables
- Signal handling for process termination
- Binary detection and handling
- Cross-platform compatibility

## Sandbox Configuration

### Sandbox Support (from packages/cli/src/config/sandboxConfig.ts)
- Multiple sandbox backends:
  - Docker
  - Podman
  - macOS sandbox-exec (Seatbelt)
- Command validation
- Environment isolation
- Image verification