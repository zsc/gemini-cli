# Chapter 15: 文件操作工作流 - Relevant Information

## Overview
This chapter should cover:
- File read/write operations
- File editing 
- File searching
- Permission control
- Conflict handling
- Diff display

## Key Files to Investigate
1. File operation tools in core/src/tools/
   - read-file.ts
   - write-file.ts
   - edit.ts
   - glob.ts
   - grep.ts
   - ls.ts
   - read-many-files.ts

2. File utilities in core/src/utils/
   - fileUtils.ts
   - editCorrector.ts
   - gitIgnoreParser.ts
   - bfsFileSearch.ts

3. Diff display components
   - cli/src/ui/components/messages/DiffRenderer.tsx

4. Permission/security related
   - sandboxConfig.ts
   - File permission checks

## Investigation Log

### 1. ReadFileTool (packages/core/src/tools/read-file.ts)
- Main class for reading files
- Handles text, images (PNG, JPG, etc), PDFs, SVG, binary files
- Parameters: absolute_path, offset (line number), limit (number of lines)
- Security: validates path is absolute and within root directory
- Uses fileService.shouldGeminiIgnoreFile() to check .geminiignore
- Calls processSingleFileContent() from fileUtils.ts
- Records telemetry metrics for file operations

### 2. WriteFileTool (packages/core/src/tools/write-file.ts)
- Writes content to files
- Parameters: file_path, content, modified_by_user
- Implements ModifiableTool interface - user can modify content
- Security: validates absolute path and within root
- Confirmation flow: shows diff before writing (unless AUTO_EDIT mode)
- Uses editCorrector to handle content correction
- Creates parent directories if needed
- Differentiates between CREATE and UPDATE operations for metrics

### 3. EditTool (packages/core/src/tools/edit.ts)
- Replaces text within files
- Parameters: file_path, old_string, new_string, expected_replacements
- Can handle multiple replacements with expected_replacements parameter
- Empty old_string creates new file
- Uses ensureCorrectEdit() for intelligent error correction
- Shows diff for confirmation
- Implements ModifiableTool interface

### 4. GlobTool (packages/core/src/tools/glob.ts)
- Finds files matching glob patterns
- Parameters: pattern, path, case_sensitive, respect_git_ignore
- Returns files sorted by modification time (newest first)
- Uses fileDiscovery service for git-aware filtering
- Default ignores: node_modules, .git
- Separates recent files (within 24h) from older ones

### 5. GrepTool (packages/core/src/tools/grep.ts)
- Searches for regex patterns in file contents
- Parameters: pattern, path, include (glob filter)
- Three strategies in order:
  1. git grep (if in git repo)
  2. system grep
  3. JavaScript fallback
- Returns matches with file path and line numbers
- Respects gitignore patterns

### 6. fileUtils.ts (packages/core/src/utils/fileUtils.ts)
Key functions:
- isWithinRoot(): Security check for path containment
- detectFileType(): Determines text/image/pdf/binary/svg
- isBinaryFile(): Content-based binary detection
- processSingleFileContent(): Main file reading logic
  - 20MB file size limit
  - Text files: 2000 lines default limit, 2000 chars per line
  - Images/PDFs: Returns base64 encoded data
  - Binary files: Returns error message

### 7. editCorrector.ts (packages/core/src/utils/editCorrector.ts)
Intelligent edit correction system:
- ensureCorrectEdit(): Corrects mismatched old_string
- Handles escaping issues (e.g., \\n vs \n)
- Uses LLM to correct if simple unescaping fails
- Caches correction results
- Detects external file modifications (checks mtime vs last edit timestamp)
- trimPairIfPossible(): Tries trimming whitespace if match fails

### 8. DiffRenderer.tsx (packages/cli/src/ui/components/messages/DiffRenderer.tsx)
- Renders file diffs with syntax highlighting
- Handles new file detection (shows as code block)
- Line number display with proper alignment
- Context gap handling (shows separator for large gaps)
- Color coding: green for additions, red for deletions
- Tab normalization (converts to spaces)

### 9. FileDiscoveryService (packages/core/src/services/fileDiscoveryService.ts)
- Centralized file filtering service
- Handles both .gitignore and .geminiignore patterns
- Used by glob and other tools for consistent filtering
- Methods:
  - filterFiles(): Filters array of paths
  - shouldGitIgnoreFile(): Check single file
  - shouldGeminiIgnoreFile(): Check single file

### 10. sandboxConfig.ts (packages/cli/src/config/sandboxConfig.ts)
- Configures sandbox execution for security
- Supports: docker, podman, sandbox-exec (macOS)
- Environment variable: GEMINI_SANDBOX
- Auto-detects available sandbox command
- Returns undefined if no sandbox needed/available

## Key Workflows Identified

### 1. File Reading Workflow
1. Tool receives absolute path
2. Validates path is absolute and within root
3. Checks .geminiignore patterns
4. Calls processSingleFileContent()
5. Detects file type
6. Handles based on type (text/image/binary)
7. Returns appropriate content format

### 2. File Writing/Editing Workflow
1. Validate parameters and path
2. Read current content (if exists)
3. Apply edit corrections if needed
4. Show diff for user confirmation
5. Create parent directories
6. Write file
7. Record metrics

### 3. File Search Workflow (Glob)
1. Validate search path
2. Execute glob search
3. Apply git/geminiignore filtering
4. Sort by modification time
5. Return absolute paths

### 4. Content Search Workflow (Grep)
1. Validate regex pattern
2. Try git grep (fastest)
3. Fall back to system grep
4. Fall back to JS implementation
5. Parse and format results

### 5. Conflict Resolution
- Edit operations detect if file was modified externally
- Uses file mtime vs tool execution timestamps
- Intelligent correction for escaping issues
- LLM-based correction for complex mismatches

### 6. Permission Control
- All paths must be absolute
- All paths must be within project root
- Respects .gitignore and .geminiignore
- Optional sandbox execution for additional security
