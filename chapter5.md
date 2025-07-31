# 第 5 章：Memory 系统设计

Gemini CLI 的 Memory 系统提供了一个分层的上下文管理机制，允许在不同层级定义项目相关的指令和知识。本章深入分析 Memory 系统的设计理念、实现细节和扩展机制。

## 5.1 Memory 系统概述

Memory 系统是 Gemini CLI 的核心特性之一，它通过 GEMINI.md 文件为 AI 助手提供项目特定的上下文信息。这些信息可以包括项目约定、代码规范、架构说明、常用命令等任何有助于 AI 理解和操作项目的内容。

### 5.1.1 设计目标

Memory 系统的设计目标包括：

1. **分层管理**：支持全局、项目、目录等多层级的上下文定义
2. **模块化组织**：通过导入机制支持内容的模块化管理
3. **动态发现**：自动发现和加载相关的 GEMINI.md 文件
4. **安全性**：防止路径遍历和循环导入等安全问题
5. **可扩展性**：支持自定义文件名和扩展机制

### 5.1.2 核心组件

Memory 系统由以下核心组件构成：

- **memoryDiscovery.ts**：负责发现和加载 GEMINI.md 文件
- **memoryImportProcessor.ts**：处理文件中的导入指令
- **memoryTool.ts**：提供 save_memory 工具供 AI 保存信息
- **memoryCommand.ts**：提供用户交互的命令接口

## 5.2 Memory 发现机制

### 5.2.1 文件发现算法

Memory 发现机制使用了一个精心设计的算法来查找和加载 GEMINI.md 文件：

```typescript
// packages/core/src/utils/memoryDiscovery.ts
async function getGeminiMdFilePathsInternal(
  currentWorkingDirectory: string,
  userHomePath: string,
  debugMode: boolean,
  fileService: FileDiscoveryService,
  extensionContextFilePaths: string[] = [],
  fileFilteringOptions: FileFilteringOptions,
  maxDirs: number,
): Promise<string[]>
```

发现顺序遵循从全局到特定的原则：

1. **全局配置**：首先检查 `~/.gemini/GEMINI.md`
2. **向上扫描**：从当前目录向上遍历到项目根目录
3. **向下扫描**：使用 BFS 算法在项目内搜索
4. **扩展文件**：加载扩展提供的上下文文件

实现中使用 Set 数据结构来确保文件路径的唯一性：

```typescript
const allPaths = new Set<string>();
const geminiMdFilenames = getAllGeminiMdFilenames();

for (const geminiMdFilename of geminiMdFilenames) {
  // 检查全局配置目录
  const globalMemoryPath = path.join(
    resolvedHome,
    GEMINI_CONFIG_DIR,
    geminiMdFilename,
  );
  
  // 尝试访问全局文件
  try {
    await fs.access(globalMemoryPath, fsSync.constants.R_OK);
    allPaths.add(globalMemoryPath);
  } catch {
    // 文件不存在或不可读
  }
  
  // 继续向上和向下扫描...
}

### 5.2.2 项目根目录识别

系统通过查找 `.git` 目录来识别项目根目录：

```typescript
async function findProjectRoot(startDir: string): Promise<string | null> {
  let currentDir = path.resolve(startDir);
  while (true) {
    const gitPath = path.join(currentDir, '.git');
    try {
      const stats = await fs.stat(gitPath);
      if (stats.isDirectory()) {
        return currentDir;
      }
    } catch (error) {
      // 继续向上查找
    }
    const parentDir = path.dirname(currentDir);
    if (parentDir === currentDir) {
      return null;
    }
    currentDir = parentDir;
  }
}
```

向上扫描过程中的重要细节：

```typescript
// 确定扫描的终止目录
const ultimateStopDir = projectRoot
  ? path.dirname(projectRoot)
  : path.dirname(resolvedHome);

// 向上扫描逻辑
while (currentDir && currentDir !== path.dirname(currentDir)) {
  // 跳过全局 .gemini 目录，因为已经单独处理
  if (currentDir === path.join(resolvedHome, GEMINI_CONFIG_DIR)) {
    break;
  }
  
  const potentialPath = path.join(currentDir, geminiMdFilename);
  try {
    await fs.access(potentialPath, fsSync.constants.R_OK);
    // 仅在不是全局路径时添加
    if (potentialPath !== globalMemoryPath) {
      upwardPaths.unshift(potentialPath); // 保持层级顺序
    }
  } catch {
    // 文件不存在或不可读
  }
  
  // 到达终止目录时停止
  if (currentDir === ultimateStopDir) {
    break;
  }
  
  currentDir = path.dirname(currentDir);
}
```

### 5.2.3 文件过滤选项

Memory 系统支持文件过滤配置：

```typescript
export const DEFAULT_MEMORY_FILE_FILTERING_OPTIONS: FileFilteringOptions = {
  respectGitIgnore: false,    // Memory 文件不受 .gitignore 影响
  respectGeminiIgnore: true,  // 但遵守 .geminiignore
};
```

文件过滤由 FileDiscoveryService 类实现，它管理两种忽略规则：

```typescript
export class FileDiscoveryService {
  private gitIgnoreFilter: GitIgnoreFilter | null = null;
  private geminiIgnoreFilter: GitIgnoreFilter | null = null;
  private projectRoot: string;

  constructor(projectRoot: string) {
    this.projectRoot = path.resolve(projectRoot);
    
    // 如果是 Git 仓库，加载 .gitignore 规则
    if (isGitRepository(this.projectRoot)) {
      const parser = new GitIgnoreParser(this.projectRoot);
      try {
        parser.loadGitRepoPatterns();
      } catch (_error) {
        // ignore file not found
      }
      this.gitIgnoreFilter = parser;
    }
    
    // 始终尝试加载 .geminiignore 规则
    const gParser = new GitIgnoreParser(this.projectRoot);
    try {
      gParser.loadPatterns(GEMINI_IGNORE_FILE_NAME);
    } catch (_error) {
      // ignore file not found
    }
    this.geminiIgnoreFilter = gParser;
  }
  
  shouldIgnoreFile(
    filePath: string,
    options: FilterFilesOptions = {}
  ): boolean {
    const { respectGitIgnore = true, respectGeminiIgnore = true } = options;
    
    if (respectGitIgnore && this.shouldGitIgnoreFile(filePath)) {
      return true;
    }
    if (respectGeminiIgnore && this.shouldGeminiIgnoreFile(filePath)) {
      return true;
    }
    return false;
  }
}
```

### 5.2.4 BFS 文件搜索

向下扫描使用广度优先搜索（BFS）算法实现高效的文件发现：

```typescript
export async function bfsFileSearch(
  rootDir: string,
  options: BfsFileSearchOptions,
): Promise<string[]> {
  const {
    fileName,
    ignoreDirs = [],
    maxDirs = Infinity,
    debug = false,
    fileService,
  } = options;
  
  const foundFiles: string[] = [];
  const queue: string[] = [rootDir];
  const visited = new Set<string>();
  let scannedDirCount = 0;

  while (queue.length > 0 && scannedDirCount < maxDirs) {
    const currentDir = queue.shift()!;
    if (visited.has(currentDir)) {
      continue;
    }
    visited.add(currentDir);
    scannedDirCount++;

    let entries: Dirent[];
    try {
      entries = await fs.readdir(currentDir, { withFileTypes: true });
    } catch {
      // 忽略无法读取的目录（如权限问题）
      continue;
    }

    for (const entry of entries) {
      const fullPath = path.join(currentDir, entry.name);
      
      // 应用文件过滤规则
      if (fileService?.shouldIgnoreFile(fullPath, {
        respectGitIgnore: options.fileFilteringOptions?.respectGitIgnore,
        respectGeminiIgnore: options.fileFilteringOptions?.respectGeminiIgnore,
      })) {
        continue;
      }

      if (entry.isDirectory()) {
        if (!ignoreDirs.includes(entry.name)) {
          queue.push(fullPath);
        }
      } else if (entry.isFile() && entry.name === fileName) {
        foundFiles.push(fullPath);
      }
    }
  }

  return foundFiles;
}
```

## 5.3 导入机制

### 5.3.1 导入语法

GEMINI.md 文件支持使用 `@` 语法导入其他 Markdown 文件：

```markdown
# 项目说明

@./conventions.md
@./architecture/overview.md
```

### 5.3.2 导入处理流程

导入处理器 `processImports` 负责递归处理导入指令：

```typescript
export async function processImports(
  content: string,
  basePath: string,
  debugMode: boolean = false,
  importState: ImportState = {
    processedFiles: new Set(),
    maxDepth: 10,
    currentDepth: 0,
  },
): Promise<string>
```

关键特性：

1. **循环检测**：通过 `processedFiles` 集合防止循环导入
2. **深度限制**：默认最大导入深度为 10 层
3. **路径验证**：防止路径遍历攻击
4. **错误处理**：导入失败时插入错误注释

导入处理的核心逻辑：

```typescript
// 导入语法的正则表达式，支持 @path/to/file.md 和 @./path/to/file.md
const importRegex = /@([./]?[^\s\n]+\.[^\s\n]+)/g;

let processedContent = content;
let match: RegExpExecArray | null;

while ((match = importRegex.exec(content)) !== null) {
  const importPath = match[1];
  
  // 验证导入路径安全性
  if (!validateImportPath(importPath, basePath, [basePath])) {
    processedContent = processedContent.replace(
      match[0],
      `<!-- Import failed: ${importPath} - Path traversal attempt -->`,
    );
    continue;
  }
  
  // 仅支持 .md 文件
  if (!importPath.endsWith('.md')) {
    logger.warn(
      `Import processor only supports .md files. Attempting to import non-md file: ${importPath}`,
    );
    processedContent = processedContent.replace(
      match[0],
      `<!-- Import failed: ${importPath} - Only .md files are supported -->`,
    );
    continue;
  }
  
  const fullPath = path.resolve(basePath, importPath);
  
  // 检查循环导入
  if (importState.processedFiles.has(fullPath)) {
    processedContent = processedContent.replace(
      match[0],
      `<!-- Import skipped: ${importPath} - Already imported (circular dependency) -->`,
    );
    continue;
  }
  
  try {
    // 读取并递归处理导入的文件
    const importedContent = await fs.readFile(fullPath, 'utf-8');
    const processedImportedContent = await processImports(
      importedContent,
      path.dirname(fullPath),
      debugMode,
      {
        ...importState,
        processedFiles: new Set([...importState.processedFiles, fullPath]),
        currentDepth: importState.currentDepth + 1,
        currentFile: fullPath,
      },
    );
    
    // 替换导入语句为处理后的内容
    processedContent = processedContent.replace(
      match[0],
      `<!-- Imported from: ${importPath} -->\n${processedImportedContent}\n<!-- End of import from: ${importPath} -->`,
    );
  } catch (error) {
    // 处理导入错误
    processedContent = processedContent.replace(
      match[0],
      `<!-- Import failed: ${importPath} - ${errorMessage} -->`,
    );
  }
}
```

### 5.3.3 安全性考虑

导入机制包含多重安全检查：

```typescript
export function validateImportPath(
  importPath: string,
  basePath: string,
  allowedDirectories: string[],
): boolean {
  // 拒绝 URL
  if (/^(file|https?):\/\//.test(importPath)) {
    return false;
  }
  
  const resolvedPath = path.resolve(basePath, importPath);
  
  // 确保路径在允许的目录内
  return allowedDirectories.some((allowedDir) => {
    const normalizedAllowedDir = path.resolve(allowedDir);
    return resolvedPath.startsWith(normalizedAllowedDir);
  });
}
```

## 5.4 Memory 工具

### 5.4.1 save_memory 工具

Memory 工具允许 AI 在对话过程中保存重要信息：

```typescript
const memoryToolSchemaData: FunctionDeclaration = {
  name: 'save_memory',
  description: 'Saves a specific piece of information or fact to your long-term memory.',
  parameters: {
    type: Type.OBJECT,
    properties: {
      fact: {
        type: Type.STRING,
        description: 'The specific fact or piece of information to remember.',
      },
    },
    required: ['fact'],
  },
};
```

### 5.4.2 存储格式

保存的信息以 Markdown 列表形式追加到全局 Memory 文件中：

```markdown
## Gemini Added Memories
- 用户喜欢使用 TypeScript
- 项目使用 pnpm 作为包管理器
- 代码风格偏好函数式编程
```

### 5.4.3 文件操作

Memory 工具使用专门的文件操作方法确保内容正确追加：

```typescript
static async performAddMemoryEntry(
  text: string,
  memoryFilePath: string,
  fsAdapter: FileSystemAdapter,
): Promise<void> {
  let processedText = text.trim();
  // 移除可能被误解为 Markdown 列表项的前导连字符
  processedText = processedText.replace(/^(-+\s*)+/, '').trim();
  const newMemoryItem = `- ${processedText}`;
  
  try {
    await fsAdapter.mkdir(path.dirname(memoryFilePath), { recursive: true });
    let content = '';
    try {
      content = await fsAdapter.readFile(memoryFilePath, 'utf-8');
    } catch (_e) {
      // 文件不存在，将创建带有标题和条目的新文件
    }
    
    const headerIndex = content.indexOf(MEMORY_SECTION_HEADER);
    
    if (headerIndex === -1) {
      // 未找到标题，追加标题和条目
      const separator = ensureNewlineSeparation(content);
      content += `${separator}${MEMORY_SECTION_HEADER}\n${newMemoryItem}\n`;
    } else {
      // 找到标题，在适当位置插入新条目
      const beforeHeader = content.substring(0, headerIndex);
      const fromHeader = content.substring(headerIndex);
      
      // 在标题后查找第一个非空行
      const lines = fromHeader.split('\n');
      let insertIndex = 1; // 标题后的第一行
      
      // 跳过空行
      while (insertIndex < lines.length && lines[insertIndex].trim() === '') {
        insertIndex++;
      }
      
      // 在标题和第一个内容之间插入新条目
      lines.splice(insertIndex, 0, newMemoryItem);
      content = beforeHeader + lines.join('\n');
    }
    
    await fsAdapter.writeFile(memoryFilePath, content, 'utf-8');
  } catch (error) {
    throw new Error(`Failed to save memory: ${error}`);
  }
}
```

辅助函数 `ensureNewlineSeparation` 确保内容之间有适当的分隔：

```typescript
function ensureNewlineSeparation(currentContent: string): string {
  if (currentContent.length === 0) return '';
  if (currentContent.endsWith('\n\n') || currentContent.endsWith('\r\n\r\n'))
    return '';
  if (currentContent.endsWith('\n') || currentContent.endsWith('\r\n'))
    return '\n';
  return '\n\n';
}
```

## 5.5 命令接口

### 5.5.1 用户命令

Memory 系统提供了三个主要命令：

1. **`/memory show`**：显示当前加载的 Memory 内容
2. **`/memory add <text>`**：通过 save_memory 工具添加新信息
3. **`/memory refresh`**：重新加载 Memory 文件

### 5.5.2 命令实现

命令通过 `memoryCommand` 对象定义：

```typescript
export const memoryCommand: SlashCommand = {
  name: 'memory',
  description: 'Commands for interacting with memory.',
  kind: CommandKind.BUILT_IN,
  subCommands: [
    {
      name: 'show',
      action: async (context) => {
        const memoryContent = context.services.config?.getUserMemory() || '';
        const fileCount = context.services.config?.getGeminiMdFileCount() || 0;
        // 显示内容
      },
    },
    // ...
  ],
};
```

## 5.6 系统集成

### 5.6.1 配置集成

Memory 内容在 Config 类中管理：

```typescript
class Config {
  private userMemory: string = '';
  private geminiMdFileCount: number = 0;
  
  getUserMemory(): string {
    return this.userMemory;
  }
  
  setUserMemory(newUserMemory: string): void {
    this.userMemory = newUserMemory;
  }
}
```

### 5.6.2 系统提示词集成

Memory 内容作为后缀追加到系统提示词：

```typescript
export function getCoreSystemPrompt(userMemory?: string): string {
  // 支持通过环境变量覆盖系统提示词
  let systemMdEnabled = false;
  let systemMdPath = path.resolve(path.join(GEMINI_CONFIG_DIR, 'system.md'));
  const systemMdVar = process.env.GEMINI_SYSTEM_MD;
  
  if (systemMdVar) {
    const systemMdVarLower = systemMdVar.toLowerCase();
    if (!['0', 'false'].includes(systemMdVarLower)) {
      systemMdEnabled = true;
      if (!['1', 'true'].includes(systemMdVarLower)) {
        // 使用自定义路径
        let customPath = systemMdVar;
        if (customPath.startsWith('~/')) {
          customPath = path.join(os.homedir(), customPath.slice(2));
        }
        systemMdPath = path.resolve(customPath);
      }
      // 覆盖启用时要求文件存在
      if (!fs.existsSync(systemMdPath)) {
        throw new Error(`missing system prompt file '${systemMdPath}'`);
      }
    }
  }
  
  const basePrompt = systemMdEnabled
    ? fs.readFileSync(systemMdPath, 'utf8')
    : /* 默认系统提示词 */;
  
  // Memory 内容作为后缀追加
  const memorySuffix =
    userMemory && userMemory.trim().length > 0
      ? `\n\n---\n\n${userMemory.trim()}`
      : '';
      
  return `${basePrompt}${memorySuffix}`;
}
```

### 5.6.3 Memory 内容加载流程

完整的 Memory 内容加载流程由 `loadServerHierarchicalMemory` 函数协调：

```typescript
export async function loadServerHierarchicalMemory(
  currentWorkingDirectory: string,
  debugMode: boolean,
  fileService: FileDiscoveryService,
  extensionContextFilePaths: string[] = [],
  fileFilteringOptions?: FileFilteringOptions,
  maxDirs: number = 200,
): Promise<{ memoryContent: string; fileCount: number }> {
  // 1. 发现所有相关的 GEMINI.md 文件
  const userHomePath = homedir();
  const filePaths = await getGeminiMdFilePathsInternal(
    currentWorkingDirectory,
    userHomePath,
    debugMode,
    fileService,
    extensionContextFilePaths,
    fileFilteringOptions || DEFAULT_MEMORY_FILE_FILTERING_OPTIONS,
    maxDirs,
  );
  
  if (filePaths.length === 0) {
    return { memoryContent: '', fileCount: 0 };
  }
  
  // 2. 读取文件并处理导入
  const contentsWithPaths = await readGeminiMdFiles(filePaths, debugMode);
  
  // 3. 连接所有内容，添加源标记
  const combinedInstructions = concatenateInstructions(
    contentsWithPaths,
    currentWorkingDirectory,
  );
  
  return { memoryContent: combinedInstructions, fileCount: filePaths.length };
}
```

## 5.7 高级特性

### 5.7.1 多文件支持

系统支持配置多个 Memory 文件名：

```typescript
export function setGeminiMdFilename(newFilename: string | string[]): void {
  if (Array.isArray(newFilename)) {
    if (newFilename.length > 0) {
      currentGeminiMdFilename = newFilename.map((name) => name.trim());
    }
  } else if (newFilename && newFilename.trim() !== '') {
    currentGeminiMdFilename = newFilename.trim();
  }
}
```

### 5.7.2 内容合并策略

多个 Memory 文件的内容通过源标记进行区分：

```typescript
function concatenateInstructions(
  instructionContents: GeminiFileContent[],
  currentWorkingDirectoryForDisplay: string,
): string {
  return instructionContents
    .filter((item) => typeof item.content === 'string')
    .map((item) => {
      const displayPath = path.isAbsolute(item.filePath)
        ? path.relative(currentWorkingDirectoryForDisplay, item.filePath)
        : item.filePath;
      return `--- Context from: ${displayPath} ---\n${trimmedContent}\n--- End of Context from: ${displayPath} ---`;
    })
    .join('\n\n');
}
```

### 5.7.3 性能优化

系统包含多项性能优化措施：

1. **目录扫描限制**：默认最多扫描 200 个目录
2. **BFS 算法**：使用广度优先搜索提高发现效率
3. **缓存机制**：Config 对象缓存已加载的 Memory 内容
4. **异步处理**：所有文件操作均为异步

## 5.8 扩展和自定义

### 5.8.1 自定义文件名

通过环境变量或配置文件可以自定义 Memory 文件名：

```json
{
  "memoryFiles": ["GEMINI.md", "AI_CONTEXT.md", "PROJECT_GUIDE.md"]
}
```

### 5.8.2 扩展集成

IDE 扩展可以提供额外的上下文文件：

```typescript
const extensionContextFilePaths = [
  '/path/to/extension/context/ide-specifics.md',
  '/path/to/extension/context/shortcuts.md'
];
```

### 5.8.3 自定义过滤规则

可以通过 `.geminiignore` 文件排除特定目录：

```
node_modules/
dist/
*.test.md
```

## 5.9 最佳实践

### 5.9.1 GEMINI.md 文件组织

建议的文件组织结构：

```
project/
├── GEMINI.md              # 项目总体说明
├── docs/
│   ├── GEMINI.md         # 文档相关上下文
│   └── conventions.md    # 可被导入的约定文档
├── src/
│   └── GEMINI.md         # 源代码相关上下文
└── tests/
    └── GEMINI.md         # 测试相关上下文
```

### 5.9.2 内容编写指南

1. **简洁明了**：使用清晰的标题和简短的说明
2. **结构化**：使用 Markdown 格式组织内容
3. **模块化**：将大文件拆分为可导入的小模块
4. **版本控制**：将 GEMINI.md 文件纳入版本控制

### 5.9.3 导入使用建议

```markdown
# 主 GEMINI.md

## 项目概述
本项目是一个 Web 应用...

## 详细文档
@./docs/architecture.md
@./docs/api-guidelines.md
@./docs/testing-strategy.md
```

## 本章小结

Memory 系统是 Gemini CLI 的一个精心设计的特性，它通过分层的上下文管理机制，使 AI 助手能够更好地理解和操作特定项目。系统的核心优势包括：

1. **灵活的发现机制**：自动发现从全局到项目特定的所有相关上下文
2. **安全的导入系统**：支持模块化组织同时防止安全漏洞
3. **动态内容管理**：AI 可以在交互过程中保存和更新信息
4. **良好的扩展性**：支持自定义配置和第三方扩展集成

通过合理使用 Memory 系统，开发者可以为 AI 助手提供丰富的项目上下文，从而获得更准确、更符合项目规范的代码生成和操作建议。