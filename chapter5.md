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

### 5.2.3 文件过滤选项

Memory 系统支持文件过滤配置：

```typescript
export const DEFAULT_MEMORY_FILE_FILTERING_OPTIONS: FileFilteringOptions = {
  respectGitIgnore: false,    // Memory 文件不受 .gitignore 影响
  respectGeminiIgnore: true,  // 但遵守 .geminiignore
};
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
  // 确保正确的换行分隔
  const separator = ensureNewlineSeparation(content);
  
  // 查找或创建 Memory 部分
  const headerIndex = content.indexOf(MEMORY_SECTION_HEADER);
  
  if (headerIndex === -1) {
    // 添加新部分
    content += `${separator}${MEMORY_SECTION_HEADER}\n${newMemoryItem}\n`;
  } else {
    // 追加到现有部分
    // ...
  }
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
  const basePrompt = /* ... 基础提示词 ... */;
  
  const memorySuffix =
    userMemory && userMemory.trim().length > 0
      ? `\n\n---\n\n${userMemory.trim()}`
      : '';
      
  return `${basePrompt}${memorySuffix}`;
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