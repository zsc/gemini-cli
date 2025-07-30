# Relevant Information for Appendix C - Troubleshooting Guide

## Error Handling Patterns

### Error Classes (packages/core/src/utils/errors.ts)
- ForbiddenError: HTTP 403 errors
- UnauthorizedError: HTTP 401 errors  
- BadRequestError: HTTP 400 errors
- toFriendlyError(): Converts raw errors to user-friendly messages
- isNodeError(): Checks for Node.js errno exceptions
- getErrorMessage(): Safely extracts error messages

### Error Reporting (packages/core/src/utils/errorReporting.ts)
- reportError(): Generates detailed error reports with context
- Writes JSON reports to temp directory
- Includes error message, stack trace, and context
- Falls back to console.error if file write fails
- Location: OS temp dir, filename: gemini-client-error-{type}-{timestamp}.json

### Retry Mechanisms (packages/core/src/utils/retry.ts)
- retryWithBackoff(): Exponential backoff with jitter
- Default: 5 attempts, 5-30 second delays
- Handles 429 (rate limit) and 5xx errors
- Respects Retry-After headers
- OAuth users get immediate fallback to Flash model on quota errors
- Consecutive 429 errors trigger model fallback

## Common Issues

### Quota Errors (packages/core/src/utils/quotaErrorDetection.ts)
- isProQuotaExceededError(): Detects Pro model quota exceeded
- isGenericQuotaExceededError(): Detects any quota exceeded
- Pattern matching for "Quota exceeded for quota metric 'Gemini"
- Different messages for Free vs Paid tiers

### API Errors (packages/cli/src/ui/utils/errorParsing.ts)
- parseAndFormatApiError(): Formats API errors with context
- Rate limit messages vary by auth type:
  - LOGIN_WITH_GOOGLE: Suggests upgrade or switch to API key
  - USE_GEMINI: Request quota increase via AI Studio
  - USE_VERTEX_AI: Request quota increase via Vertex
- Automatic model fallback for OAuth users

## Configuration Errors

### Auth Validation (packages/cli/src/config/auth.ts)
- validateAuthMethod(): Checks for required environment variables
- USE_GEMINI: Requires GEMINI_API_KEY
- USE_VERTEX_AI: Requires either:
  - GOOGLE_CLOUD_PROJECT + GOOGLE_CLOUD_LOCATION
  - GOOGLE_API_KEY (express mode)
- Returns specific error messages for missing config

### Startup Warnings (packages/cli/src/utils/startupWarnings.ts)
- Reads warnings from temp file: gemini-cli-warnings.txt
- Auto-deletes file after reading
- Returns empty array if file doesn't exist
- Returns error message if read fails (not ENOENT)

## Authentication Issues

### OAuth Errors
- 401 Unauthorized: Invalid credentials
- 403 Forbidden: Project doesn't have code assist enabled
- Token refresh failures
- Missing environment variables

### API Key Issues
- Invalid or expired API keys
- Wrong project/location for Vertex AI
- Missing permissions

## Tool Execution Errors

### Shell Tool (packages/core/src/tools/shell.ts)
- Command validation before execution
- Checks against allowlist
- Directory must be relative and exist
- Empty commands rejected
- Must identify command root for permission
- Returns specific validation error messages

### File Operations
- ENOENT: File not found
- EACCES: Permission denied
- ETIMEDOUT: Operation timed out
- Path validation errors

## Debugging Approaches

### Error Reports
- Check temp directory for detailed JSON reports
- Report includes full error context and stack trace
- Filename pattern helps identify error type and time

### Logging Levels
- console.warn: Rate limit (429) errors
- console.error: Server errors (5xx)
- Retry attempts logged with status codes

## Network and API Issues

### Timeout Handling
- MCP default timeout: 10 minutes
- Configurable per operation
- AbortSignal support for cancellation

### Connection States
- Network connectivity checks
- Proxy configuration support
- SSL/TLS certificate issues

## Performance Issues

### Model Fallback
- Automatic switch to Flash model on quota/rate limits
- Preserves session continuity
- User notification of model change

### Response Times
- Slow responses trigger model fallback
- Exponential backoff prevents request flooding

## MCP Integration Issues

### Server Status (packages/core/src/tools/mcp-client.ts)
- MCPServerStatus enum: DISCONNECTED, CONNECTING, CONNECTED
- MCPDiscoveryState enum: NOT_STARTED, IN_PROGRESS, COMPLETED
- Status change listeners for monitoring
- OAuth requirement tracking per server

### Common MCP Errors
- Server connection failures
- Tool discovery timeouts
- OAuth authentication required
- Invalid server configuration

## VSCode Extension Issues

### Connection Problems
- IDE server startup failures
- Port conflicts
- Authentication mismatches

### File Sync Issues
- Open files manager errors
- Context update delays