# Chapter 16: Error Handling and Recovery Mechanisms - Relevant Information

## Error Handling Files to Investigate

### Core Package Error Handling
- `/packages/core/src/utils/errors.ts` - Error types and utilities
- `/packages/core/src/utils/errorReporting.ts` - Error reporting mechanisms
- `/packages/core/src/utils/quotaErrorDetection.ts` - Quota error detection
- `/packages/core/src/utils/retry.ts` - Retry logic
- `/packages/core/src/core/geminiRequest.ts` - API error handling

### CLI Package Error Handling
- `/packages/cli/src/ui/components/messages/ErrorMessage.tsx` - Error display
- `/packages/cli/src/ui/utils/errorParsing.ts` - Error parsing utilities

### Tool Error Handling
- Various tool implementations in `/packages/core/src/tools/`

### Service Error Handling
- `/packages/core/src/services/loopDetectionService.ts` - Loop detection
- `/packages/core/src/services/shellExecutionService.ts` - Shell execution errors

## Information Collected

### 1. Error Types and Hierarchies (from errors.ts)
- **Custom Error Classes**:
  - `ForbiddenError` - HTTP 403 errors
  - `UnauthorizedError` - HTTP 401 errors  
  - `BadRequestError` - HTTP 400 errors
- **Error Utilities**:
  - `isNodeError()` - Type guard for Node.js errors
  - `getErrorMessage()` - Safe error message extraction
  - `toFriendlyError()` - Converts API errors to user-friendly error types

### 2. Error Reporting System (from errorReporting.ts)
- **Report Generation**:
  - Creates detailed JSON error reports with context
  - Saves to temporary files for debugging
  - Includes error message, stack trace, and context
- **Fallback Strategies**:
  - If JSON stringify fails, tries minimal report
  - If file write fails, logs to console
  - Handles BigInt and other non-serializable data

### 3. Retry Mechanisms (from retry.ts)
- **Exponential Backoff**:
  - Default: 5 attempts, 5s initial delay, 30s max delay
  - Adds 30% jitter to avoid thundering herd
  - Respects Retry-After headers for 429 errors
- **Smart Retry Logic**:
  - Retries on 429 (rate limit) and 5xx errors
  - Special handling for quota exceeded errors
  - OAuth users get immediate fallback to Flash model
- **Fallback Model Strategy**:
  - Pro quota exceeded → Switch to Flash model
  - Generic quota exceeded → Switch to Flash model
  - Persistent 429s (2+ consecutive) → Trigger fallback

### 4. Quota Error Detection (from quotaErrorDetection.ts)
- **Pro Quota Detection**:
  - Matches pattern: "Quota exceeded for quota metric 'Gemini ... Pro Requests'"
  - Used for immediate model fallback
- **Generic Quota Detection**:
  - Matches any "Quota exceeded for quota metric" message
  - Different messaging for free vs paid tiers

### 5. User-Facing Error Messages (from errorParsing.ts)
- **Tier-Based Messaging**:
  - Free tier: Suggests upgrading to paid plans
  - Paid tier: Appreciates user, suggests API key
- **Auth-Type Specific Messages**:
  - LOGIN_WITH_GOOGLE: Model fallback messages
  - USE_GEMINI: Request quota increase
  - USE_VERTEX_AI: Vertex quota management
- **Error Formatting**:
  - Parses nested JSON errors
  - Adds rate limit context based on error type

### 6. Loop Detection (from loopDetectionService.ts)
- **Tool Call Loops**:
  - Detects 5+ identical consecutive tool calls
  - Uses SHA256 hash of tool name + args
- **Content Loops**:
  - Analyzes 50-char chunks for repetition
  - Detects 10+ repetitions within 1.5x chunk size
  - Disabled inside code blocks
- **LLM-Based Detection**:
  - Activates after 30 turns
  - Uses Flash model to analyze conversation
  - Dynamic check interval based on confidence

### 7. Tool Error Handling (from shell.ts)
- **Shell Command Errors**:
  - Captures stdout, stderr, exit codes, signals
  - Handles binary output detection
  - Tracks background processes
  - User-friendly error formatting
- **Validation Errors**:
  - Command allowlist checking
  - Directory validation
  - Parameter schema validation

### 8. Component Error Display (from ErrorMessage.tsx)
- Simple error UI with red color and ✕ prefix
- Text wrapping for long error messages
- Consistent styling via Colors.AccentRed

### 9. Network Error Handling (from fetch.ts)
- **Custom FetchError Class**:
  - Extends Error with optional error code
  - Used for network-related errors
- **Timeout Handling**:
  - Uses AbortController for request timeout
  - Throws FetchError with ETIMEDOUT code
- **Private IP Detection**:
  - Checks for private IP ranges
  - Prevents requests to internal network

### 10. File Operation Errors (from write-file.ts)
- **Pre-write Validation**:
  - Path validation (must be within root)
  - Parameter schema validation
- **File System Errors**:
  - ENOENT: Creates new file
  - Permission errors: Returns error message
  - Directory creation: Recursive mkdir
- **Error Recovery**:
  - Shows diff even for new files
  - Graceful handling of unreadable files
  - Detailed error messages with file paths

### 11. Authentication Flow Errors
- **UnauthorizedError Handling**:
  - Used in useGeminiStream hook
  - Triggers re-authentication flow
  - Imported from core error types
- **OAuth Token Errors**:
  - Token refresh mechanisms
  - Fallback to re-authentication

### 12. Configuration Errors
- **Settings File Validation**:
  - JSON parse errors caught and reported
  - Shows file path in error message
- **MCP Configuration**:
  - Validates required fields (httpUrl, url, command)
  - Clear error messages for missing config

### 13. Telemetry and Monitoring
- **Error Tracking**:
  - Loop detection events logged
  - File operation metrics recorded
  - Error reports saved to temp files
- **Debug Mode**:
  - Enhanced error output in debug mode
  - Full stack traces when enabled

### 14. User Experience Considerations
- **Progressive Disclosure**:
  - Simple messages for users
  - Detailed logs for debugging
- **Actionable Messages**:
  - Suggests upgrades for quota errors
  - Provides links to documentation
  - Clear next steps for recovery
- **Context Preservation**:
  - Saves chat history on errors
  - Allows error report generation
  - Maintains session state