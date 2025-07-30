# Memory System Relevant Information

This file collects relevant information about the memory system implementation in Gemini CLI.

## Core Memory System Components

### 1. Memory Discovery (memoryDiscovery.ts)
- **Purpose**: Discovers and loads hierarchical GEMINI.md files from various locations
- **Key Functions**:
  - `loadServerHierarchicalMemory()`: Main entry point for loading memory
  - `getGeminiMdFilePathsInternal()`: Finds all GEMINI.md files in hierarchy
  - `findProjectRoot()`: Locates git repository root
  - `readGeminiMdFiles()`: Reads and processes imports in memory files
  - `concatenateInstructions()`: Combines multiple memory files with source markers

- **Discovery Order**:
  1. Global memory: `~/.gemini/GEMINI.md`
  2. Upward scan: From CWD to project root/home directory
  3. Downward scan: BFS search within project directory
  4. Extension context files (if provided)

- **Features**:
  - Respects file filtering options (gitignore, geminiignore)
  - Processes import directives recursively
  - Adds source markers to track content origin
  - Configurable max directory scan limit

### 2. Import Processing (memoryImportProcessor.ts)
- **Purpose**: Processes `@path/to/file.md` import directives in GEMINI.md files
- **Key Features**:
  - Circular import detection and prevention
  - Path traversal security validation
  - Maximum import depth limit (10 by default)
  - Error handling with inline comments
  - Only supports .md file imports
  - Recursive import processing

### 3. Memory Tool (memoryTool.ts)
- **Purpose**: Provides `save_memory` tool for AI to persist facts
- **Key Components**:
  - Tool schema for Gemini API
  - Memory section header: `## Gemini Added Memories`
  - Global memory location: `~/.gemini/GEMINI.md`
  - Configurable filename support (can use custom names)
  - Appends memory items as markdown list items

### 4. Memory Commands (memoryCommand.ts)
- **Available Commands**:
  - `/memory show`: Display current memory content
  - `/memory add <text>`: Add new memory via save_memory tool
  - `/memory refresh`: Reload memory from source files

### 5. Configuration Integration
- Memory content stored in Config class (`userMemory` property)
- File count tracking (`geminiMdFileCount`)
- Memory included in system prompt via `getCoreSystemPrompt()`
- File filtering options for memory discovery

### 6. System Prompt Integration
- Memory appended to system prompt as suffix
- Format: `---\n\n{memory content}`
- Only included if memory has content

## Key Design Decisions

1. **Hierarchical Loading**: Memory files are discovered from most global to most specific
2. **Import Support**: Allows modular memory organization via imports
3. **Security**: Path validation prevents traversal attacks
4. **Performance**: BFS with max directory limit prevents excessive scanning
5. **Flexibility**: Supports custom filenames and multiple memory files
6. **Context Preservation**: Source markers help identify memory origin