# 第 3 章：工具系统架构

工具系统是 Gemini CLI 的核心功能之一，它使 AI 能够执行各种操作，如文件读写、代码编辑、命令执行等。本章深入剖析工具系统的设计与实现，包括工具的定义、注册、发现、调度和执行的完整生命周期。

## 3.1 工具系统概述

### 3.1.1 设计理念

Gemini CLI 的工具系统采用了插件化架构，具有以下特点：

1. **统一接口**：所有工具实现相同的 `Tool` 接口，确保一致的行为
2. **声明式定义**：工具通过 schema 声明参数和功能，与 Gemini API 无缝集成
3. **安全执行**：内置确认机制，防止危险操作
4. **可扩展性**：支持内置工具、外部命令工具和 MCP 协议工具
5. **流式输出**：支持实时输出更新，提供更好的用户体验
6. **类型安全**：使用 TypeScript 确保参数和返回值的类型安全

### 3.1.2 工具类型

系统支持三种类型的工具：

- **内置工具**：直接在代码中实现的工具（如文件操作、搜索等）
- **发现工具**：通过外部命令动态发现的工具
- **MCP 工具**：通过 Model Context Protocol 服务器提供的工具

### 3.1.3 工具生命周期

工具从注册到执行完成的完整生命周期包括：

1. **注册阶段**：工具被注册到 ToolRegistry
2. **发现阶段**：通过配置的命令或 MCP 服务器发现新工具
3. **验证阶段**：验证工具参数的有效性
4. **确认阶段**：根据配置决定是否需要用户确认
5. **执行阶段**：实际执行工具逻辑
6. **响应阶段**：将结果返回给 Gemini API

## 3.2 工具接口与基类

### 3.2.1 Tool 接口定义

`Tool` 接口是所有工具的核心契约，定义了工具的属性和行为：

```typescript
interface Tool<TParams = unknown, TResult extends ToolResult = ToolResult> {
  // 标识和描述
  name: string;                    // API 调用时使用的内部名称
  displayName: string;             // 用户友好的显示名称
  description: string;             // 工具功能描述
  icon: Icon;                      // UI 显示图标
  schema: FunctionDeclaration;     // Gemini API 的函数声明
  
  // 输出特性
  isOutputMarkdown: boolean;       // 输出是否应渲染为 Markdown
  canUpdateOutput: boolean;        // 是否支持流式输出
  
  // 核心方法
  validateToolParams(params: TParams): string | null;
  getDescription(params: TParams): string;
  toolLocations(params: TParams): ToolLocation[];
  shouldConfirmExecute(params: TParams, signal: AbortSignal): 
    Promise<ToolCallConfirmationDetails | false>;
  execute(params: TParams, signal: AbortSignal, 
    updateOutput?: (output: string) => void): Promise<TResult>;
}
```

关键方法说明：

- **validateToolParams**: 在执行前验证参数，返回错误信息或 null
- **getDescription**: 生成工具操作的人类可读描述
- **toolLocations**: 返回工具将影响的文件系统位置列表
- **shouldConfirmExecute**: 决定是否需要用户确认，支持异步操作
- **execute**: 执行工具的核心逻辑，支持取消信号和流式输出

### 3.2.2 BaseTool 抽象类

`BaseTool` 提供了工具的基础实现：

```typescript
abstract class BaseTool<TParams, TResult> implements Tool<TParams, TResult> {
  constructor(
    readonly name: string,
    readonly displayName: string,
    readonly description: string,
    readonly icon: Icon,
    readonly parameterSchema: Schema,
    readonly isOutputMarkdown: boolean = true,
    readonly canUpdateOutput: boolean = false,
  ) {}
  
  get schema(): FunctionDeclaration {
    return {
      name: this.name,
      description: this.description,
      parameters: this.parameterSchema,
    };
  }
  
  // 默认实现，子类可覆盖
  validateToolParams(params: TParams): string | null {
    return null;
  }
  
  shouldConfirmExecute(): Promise<ToolCallConfirmationDetails | false> {
    return Promise.resolve(false);
  }
  
  // 必须由子类实现
  abstract execute(params: TParams, signal: AbortSignal, 
    updateOutput?: (output: string) => void): Promise<TResult>;
}
```

### 3.2.3 工具结果结构

工具执行结果需要同时满足 AI 模型和用户界面的需求：

```typescript
interface ToolResult {
  summary?: string;              // 一行总结，用于快速了解结果
  llmContent: PartListUnion;     // 供 LLM 使用的内容
  returnDisplay: ToolResultDisplay; // 用户界面显示内容
}

type ToolResultDisplay = string | FileDiff;

interface FileDiff {
  fileDiff: string;              // diff 格式的文件变更
  fileName: string;              // 文件名
  originalContent: string | null; // 原始内容，null 表示新文件
  newContent: string;            // 新内容
}
```

`PartListUnion` 支持多种内容类型：
- 纯文本字符串
- 二进制数据（图片、文档等）
- 函数响应
- 混合内容数组

### 3.2.4 工具位置接口

工具位置用于跟踪工具操作影响的文件系统路径：

```typescript
interface ToolLocation {
  type: 'file' | 'directory';
  path: string;
}
```

这个接口主要用于：
- IDE 集成中的文件监控
- 权限检查
- 操作影响范围分析

## 3.3 工具注册与发现

### 3.3.1 ToolRegistry 类

`ToolRegistry` 是工具管理的中心：

```typescript
class ToolRegistry {
  private tools: Map<string, Tool> = new Map();
  
  registerTool(tool: Tool): void {
    if (this.tools.has(tool.name)) {
      console.warn(`Tool "${tool.name}" is already registered. Overwriting.`);
    }
    this.tools.set(tool.name, tool);
  }
  
  async discoverAllTools(): Promise<void> {
    // 清除之前发现的工具
    this.clearDiscoveredTools();
    
    // 从命令行发现工具
    await this.discoverAndRegisterToolsFromCommand();
    
    // 从 MCP 服务器发现工具
    await discoverMcpTools(/* ... */);
  }
  
  getTool(name: string): Tool | undefined {
    return this.tools.get(name);
  }
  
  getAllTools(): Tool[] {
    return Array.from(this.tools.values())
      .sort((a, b) => a.displayName.localeCompare(b.displayName));
  }
}
```

### 3.3.2 命令行工具发现

通过执行配置的发现命令来动态加载工具：

1. 执行 `toolDiscoveryCommand`（如 `npm run list-tools`）
2. 解析返回的 JSON，提取函数声明
3. 创建 `DiscoveredTool` 实例并注册

```typescript
class DiscoveredTool extends BaseTool {
  constructor(
    private readonly config: Config,
    name: string,
    readonly description: string,
    readonly parameterSchema: Record<string, unknown>,
  ) {
    const discoveryCmd = config.getToolDiscoveryCommand()!;
    const callCommand = config.getToolCallCommand()!;
    // 在描述中添加工具发现和调用的元信息
    description += `
This tool was discovered from the project by executing the command \`${discoveryCmd}\` on project root.
When called, this tool will execute the command \`${callCommand} ${name}\` on project root.`;
    
    super(name, name, description, Icon.Hammer, parameterSchema, false, false);
  }

  async execute(params: ToolParams): Promise<ToolResult> {
    const callCommand = this.config.getToolCallCommand()!;
    const child = spawn(callCommand, [this.name]);
    
    // 将参数通过 stdin 传递给子进程
    child.stdin.write(JSON.stringify(params));
    child.stdin.end();
    
    // 收集输出和错误信息
    let stdout = '';
    let stderr = '';
    let error: Error | null = null;
    let code: number | null = null;
    let signal: NodeJS.Signals | null = null;
    
    // 等待进程完成并收集结果
    await new Promise<void>((resolve) => {
      child.stdout.on('data', (data) => { stdout += data.toString(); });
      child.stderr.on('data', (data) => { stderr += data.toString(); });
      child.on('error', (err) => { error = err; });
      child.on('close', (_code, _signal) => {
        code = _code;
        signal = _signal;
        resolve();
      });
    });
    
    // 根据执行结果构造返回值
    // ...
  }
}
```

发现的工具会自动获得：
- 标准化的错误处理
- 进程管理和清理
- 超时控制
- 输出格式化

### 3.3.3 参数净化

为确保与 Gemini API 的兼容性，工具参数需要净化：

```typescript
function sanitizeParameters(schema?: Schema) {
  // 移除 anyOf 与 default 的冲突
  if (schema.anyOf) {
    schema.default = undefined;
  }
  
  // 确保 enum 只用于 STRING 类型
  if (schema.enum && schema.type !== Type.STRING) {
    schema.type = Type.STRING;
  }
  
  // 移除不支持的 format 值
  if (schema.type === Type.STRING && 
      schema.format !== 'enum' && 
      schema.format !== 'date-time') {
    schema.format = undefined;
  }
  
  // 递归处理嵌套 schema
}
```

## 3.4 工具调度与执行

### 3.4.1 CoreToolScheduler

`CoreToolScheduler` 管理工具执行的完整生命周期，是工具系统的核心调度器：

```typescript
interface CoreToolSchedulerOptions {
  toolRegistry: Promise<ToolRegistry>;
  outputUpdateHandler?: OutputUpdateHandler;
  onAllToolCallsComplete?: AllToolCallsCompleteHandler;
  onToolCallsUpdate?: ToolCallsUpdateHandler;
  getPreferredEditor: () => EditorType | undefined;
  config: Config;
}

class CoreToolScheduler {
  private toolCalls: ToolCall[] = [];
  private toolRegistry: Promise<ToolRegistry>;
  private outputUpdateHandler?: OutputUpdateHandler;
  private onAllToolCallsComplete?: AllToolCallsCompleteHandler;
  private onToolCallsUpdate?: ToolCallsUpdateHandler;
  
  constructor(options: CoreToolSchedulerOptions) {
    this.config = options.config;
    this.toolRegistry = options.toolRegistry;
    this.outputUpdateHandler = options.outputUpdateHandler;
    this.onAllToolCallsComplete = options.onAllToolCallsComplete;
    this.onToolCallsUpdate = options.onToolCallsUpdate;
    this.getPreferredEditor = options.getPreferredEditor;
  }
  
  async schedule(
    request: ToolCallRequestInfo | ToolCallRequestInfo[],
    signal: AbortSignal
  ): Promise<void> {
    // 验证没有正在运行的工具
    if (this.isRunning()) {
      throw new Error('Cannot schedule while tools are running');
    }
    
    // 创建工具调用实例
    const requests = Array.isArray(request) ? request : [request];
    const newToolCalls = await this.createToolCalls(requests);
    
    // 批量处理工具调用
    await this.processToolCalls(newToolCalls, signal);
  }
  
  private async processToolCalls(
    toolCalls: ToolCall[],
    signal: AbortSignal
  ): Promise<void> {
    // 并行执行不需要确认的工具
    const noConfirmCalls = toolCalls.filter(call => !call.needsConfirmation);
    await Promise.all(
      noConfirmCalls.map(call => this.executeToolCall(call, signal))
    );
    
    // 顺序处理需要确认的工具
    const confirmCalls = toolCalls.filter(call => call.needsConfirmation);
    for (const call of confirmCalls) {
      await this.processToolCallWithConfirmation(call, signal);
    }
  }
}
```

调度器的关键职责：
- **状态管理**：跟踪每个工具调用的状态
- **并发控制**：优化执行顺序，提高性能
- **事件通知**：向 UI 层报告状态变化
- **错误处理**：统一处理各种异常情况

### 3.4.2 工具调用状态

工具调用经历以下状态转换：

```typescript
type ToolCall = 
  | ValidatingToolCall      // 参数验证中
  | ScheduledToolCall       // 已调度，等待执行
  | WaitingToolCall         // 等待用户确认
  | ExecutingToolCall       // 执行中
  | SuccessfulToolCall      // 执行成功
  | ErroredToolCall         // 执行失败
  | CancelledToolCall       // 已取消
```

状态转换图：
```
validating → scheduled → executing → success
     ↓           ↓           ↓          ↓
   error    awaiting_    cancelled    error
           approval
               ↓
           cancelled
```

### 3.4.3 确认机制

工具执行前的确认流程提供了灵活的安全控制：

#### 确认类型

```typescript
type ToolCallConfirmationDetails = 
  | ToolEditConfirmationDetails    // 文件编辑确认
  | ToolExecuteConfirmationDetails  // 命令执行确认
  | ToolMCPConfirmationDetails      // MCP 工具确认
  | ToolInfoConfirmationDetails;    // 通用信息确认

interface ToolEditConfirmationDetails {
  type: 'edit';
  title: string;
  fileName: string;
  fileDiff: string;
  originalContent: string | null;
  newContent: string;
  onConfirm: (outcome: ToolConfirmationOutcome, 
    payload?: ToolConfirmationPayload) => Promise<void>;
}

interface ToolExecuteConfirmationDetails {
  type: 'execute';
  title: string;
  detail: string;
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
}
```

#### 确认结果

```typescript
enum ToolConfirmationOutcome {
  ProceedOnce = 'proceed-once',       // 仅这次执行
  ProceedAlways = 'proceed-always',   // 本会话总是执行
  ModifyWithEditor = 'modify-with-editor', // 编辑器修改
  Cancel = 'cancel'                   // 取消执行
}
```

#### 确认流程实现

1. **检查是否需要确认**：
   ```typescript
   const confirmationDetails = await tool.shouldConfirmExecute(params, signal);
   if (confirmationDetails) {
     this.setStatusInternal(callId, 'awaiting_approval', confirmationDetails);
   }
   ```

2. **处理用户响应**：
   ```typescript
   switch (outcome) {
     case ToolConfirmationOutcome.ProceedOnce:
       // 直接执行
       await this.executeToolCall(toolCall, signal);
       break;
       
     case ToolConfirmationOutcome.ProceedAlways:
       // 记录到允许列表，然后执行
       this.addToAllowlist(toolCall);
       await this.executeToolCall(toolCall, signal);
       break;
       
     case ToolConfirmationOutcome.ModifyWithEditor:
       // 打开编辑器修改参数
       const modifiedParams = await this.modifyWithEditor(toolCall);
       await this.executeWithModifiedParams(toolCall, modifiedParams);
       break;
       
     case ToolConfirmationOutcome.Cancel:
       // 取消执行
       this.setStatusInternal(callId, 'cancelled', 'User cancelled');
       break;
   }
   ```

3. **自动批准模式**：
   ```typescript
   // 根据配置的 ApprovalMode 决定是否自动批准
   if (this.config.getApprovalMode() === ApprovalMode.AutoApprove) {
     return ToolConfirmationOutcome.ProceedAlways;
   }
   ```

### 3.4.4 输出处理

将工具输出转换为 Gemini API 的函数响应：

```typescript
function convertToFunctionResponse(
  toolName: string,
  callId: string,
  llmContent: PartListUnion
): PartListUnion {
  // 处理字符串输出
  if (typeof llmContent === 'string') {
    return createFunctionResponsePart(callId, toolName, llmContent);
  }
  
  // 处理二进制内容
  if (llmContent.inlineData || llmContent.fileData) {
    const functionResponse = createFunctionResponsePart(
      callId, toolName, `Binary content of type ${mimeType}`
    );
    return [functionResponse, llmContent];
  }
  
  // 处理其他类型...
}
```

## 3.5 内置工具实现

### 3.5.1 文件编辑工具（EditTool）

`EditTool` 是最复杂的内置工具之一，实现了 `ModifiableTool` 接口，提供精确的文本替换功能：

#### 参数定义

```typescript
interface EditToolParams {
  file_path: string;              // 文件的绝对路径
  old_string: string;             // 要替换的文本
  new_string: string;             // 替换后的文本
  expected_replacements?: number; // 期望的替换次数
  modified_by_user?: boolean;     // 是否被用户手动修改
}
```

#### 核心实现

```typescript
class EditTool extends BaseTool<EditToolParams, ToolResult> 
  implements ModifiableTool<EditToolParams> {
  
  static readonly Name = 'replace';
  
  constructor(private readonly config: Config) {
    super(
      EditTool.Name,
      'Edit',
      `Replaces text within a file. By default, replaces a single occurrence...
      CRITICAL for old_string: Must uniquely identify the single instance to change. 
      Include at least 3 lines of context BEFORE and AFTER the target text...`,
      Icon.Pencil,
      {
        properties: {
          file_path: {
            description: "The absolute path to the file to modify. Must start with '/'.",
            type: Type.STRING,
          },
          old_string: {
            description: 'The exact literal text to replace, preferably unescaped...',
            type: Type.STRING,
          },
          new_string: {
            description: 'The exact literal text to replace `old_string` with...',
            type: Type.STRING,
          },
          expected_replacements: {
            type: Type.NUMBER,
            description: 'Number of replacements expected. Defaults to 1...',
            minimum: 1,
          },
        },
        required: ['file_path', 'old_string', 'new_string'],
      }
    );
  }
  
  async calculateEdit(params: EditToolParams): Promise<CalculatedEdit> {
    const { file_path, old_string, new_string, expected_replacements = 1 } = params;
    
    try {
      // 读取当前文件内容
      const currentContent = await fs.promises.readFile(file_path, 'utf-8');
      
      // 计算出现次数
      const occurrences = currentContent.split(old_string).length - 1;
      
      // 验证匹配次数
      if (occurrences === 0) {
        return {
          error: {
            display: 'No matches found for old_string',
            raw: `Pattern not found in file: ${old_string}`
          },
          currentContent,
          newContent: currentContent,
          occurrences: 0,
          isNewFile: false
        };
      }
      
      if (occurrences !== expected_replacements) {
        return {
          error: {
            display: `Found ${occurrences} matches but expected ${expected_replacements}`,
            raw: 'Occurrence count mismatch'
          },
          currentContent,
          newContent: currentContent,
          occurrences,
          isNewFile: false
        };
      }
      
      // 执行替换
      const newContent = currentContent.split(old_string).join(new_string);
      
      return {
        currentContent,
        newContent,
        occurrences,
        isNewFile: false
      };
    } catch (error) {
      // 文件不存在时创建新文件
      if (isNodeError(error) && error.code === 'ENOENT') {
        if (old_string === '') {
          return {
            currentContent: null,
            newContent: new_string,
            occurrences: 1,
            isNewFile: true
          };
        }
      }
      throw error;
    }
  }
  
  getModifyContext(abortSignal: AbortSignal): ModifyContext<EditToolParams> {
    return {
      getFilePath: (params) => params.file_path,
      getCurrentContent: async (params) => {
        const result = await this.calculateEdit(params);
        return result.currentContent || '';
      },
      getProposedContent: async (params) => {
        const result = await this.calculateEdit(params);
        return result.newContent;
      },
      createUpdatedParams: (oldContent, modifiedProposedContent, originalParams) => {
        // 使用 ensureCorrectEdit 确保编辑的正确性
        const correctedEdit = ensureCorrectEdit(
          oldContent,
          originalParams.old_string,
          originalParams.new_string,
          modifiedProposedContent
        );
        return {
          ...originalParams,
          old_string: correctedEdit.old_string,
          new_string: correctedEdit.new_string,
          modified_by_user: true
        };
      }
    };
  }
}
```

关键特性：
- **精确匹配**：要求提供足够的上下文确保唯一匹配
- **多次替换**：通过 `expected_replacements` 支持批量替换
- **新文件创建**：当文件不存在且 `old_string` 为空时创建新文件
- **用户修改支持**：实现 `ModifiableTool` 接口，允许用户在确认时修改
- **智能纠错**：`ensureCorrectEdit` 函数处理用户修改后的参数调整

### 3.5.2 文件操作工具集

#### ReadFileTool - 文件读取工具

读取文件内容并添加行号，方便后续编辑操作：

```typescript
class ReadFileTool extends BaseTool<ReadFileParams, ToolResult> {
  static Name = 'read_file';
  
  async execute(params: ReadFileParams): Promise<ToolResult> {
    const content = await fs.promises.readFile(params.file_path, 'utf-8');
    
    // 添加行号
    const lines = content.split('\n');
    const numberedContent = lines.map((line, i) => 
      `${(i + 1).toString().padStart(5)}→${line}`
    ).join('\n');
    
    return {
      summary: `Read ${params.file_path}`,
      llmContent: numberedContent,
      returnDisplay: numberedContent
    };
  }
}
```

#### WriteFileTool - 文件写入工具

创建新文件或完全覆盖现有文件：

```typescript
class WriteFileTool extends BaseTool<WriteFileParams, ToolResult> {
  static Name = 'write_file';
  
  async shouldConfirmExecute(params: WriteFileParams): 
    Promise<ToolEditConfirmationDetails | false> {
    // 检查文件是否已存在
    const exists = await fileExists(params.file_path);
    if (!exists) return false; // 新文件不需要确认
    
    // 显示将被覆盖的内容
    const currentContent = await fs.promises.readFile(params.file_path, 'utf-8');
    return {
      type: 'edit',
      title: `Overwrite ${params.file_path}`,
      fileName: params.file_path,
      fileDiff: Diff.createPatch(params.file_path, currentContent, params.content),
      originalContent: currentContent,
      newContent: params.content,
      onConfirm: async () => { /* ... */ }
    };
  }
}
```

#### GlobTool - 文件模式搜索

使用 glob 模式搜索文件：

```typescript
class GlobTool extends BaseTool<GlobParams, ToolResult> {
  static Name = 'glob';
  
  async execute(params: GlobParams): Promise<ToolResult> {
    const { pattern, rootPath } = params;
    
    // 使用 fast-glob 进行高效搜索
    const files = await glob(pattern, {
      cwd: rootPath,
      ignore: ['**/node_modules/**', '**/.git/**'],
      onlyFiles: true,
      absolute: true
    });
    
    // 按修改时间排序
    const sortedFiles = await this.sortByModificationTime(files);
    
    return {
      summary: `Found ${files.length} files matching ${pattern}`,
      llmContent: sortedFiles.join('\n'),
      returnDisplay: this.formatFileList(sortedFiles)
    };
  }
}
```

#### GrepTool - 内容搜索工具

使用 ripgrep 进行高效的正则表达式搜索：

```typescript
class GrepTool extends BaseTool<GrepParams, ToolResult> {
  static Name = 'grep';
  
  async execute(params: GrepParams): Promise<ToolResult> {
    const args = ['--json'];
    
    // 构建 ripgrep 参数
    if (params.case_insensitive) args.push('-i');
    if (params.regex) args.push('-e');
    if (params.whole_word) args.push('-w');
    if (params.fixed_strings) args.push('-F');
    
    args.push(params.pattern);
    if (params.path) args.push(params.path);
    
    // 执行 ripgrep
    const results = await this.executeRipgrep(args);
    
    // 格式化结果
    return {
      summary: `Found ${results.matchCount} matches in ${results.fileCount} files`,
      llmContent: results.formatted,
      returnDisplay: results.formatted
    };
  }
}
```

### 3.5.3 系统工具

#### ShellTool - Shell 命令执行

执行 shell 命令，支持后台进程和进程组管理：

```typescript
class ShellTool extends BaseTool<ShellToolParams, ToolResult> {
  static Name = 'run_shell_command';
  
  constructor(private readonly config: Config) {
    super(
      ShellTool.Name,
      'Shell',
      `This tool executes a given shell command as \`bash -c <command>\`. 
      Command can start background processes using \`&\`. 
      Command is executed as a subprocess that leads its own process group...`,
      Icon.Terminal,
      // ... schema 定义
      false, // 输出不是 markdown
      true   // 支持流式输出
    );
  }
  
  async shouldConfirmExecute(params: ShellToolParams): 
    Promise<ToolExecuteConfirmationDetails | false> {
    // 检查是否在允许列表中
    if (this.isAllowed(params.command)) return false;
    
    // 检查是否是危险命令
    const commandRoots = getCommandRoots(params.command);
    if (this.isDangerous(commandRoots)) {
      return {
        type: 'execute',
        title: 'Execute shell command',
        detail: this.formatCommandDetail(params),
        onConfirm: async (outcome) => { /* ... */ }
      };
    }
    
    return false;
  }
  
  async execute(
    params: ShellToolParams, 
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<ToolResult> {
    const service = new ShellExecutionService(
      this.config.getProjectRoot(),
      params.directory
    );
    
    // 设置流式输出
    if (updateOutput) {
      service.on('output', (event: ShellOutputEvent) => {
        updateOutput(this.formatPartialOutput(event));
      });
    }
    
    // 执行命令
    const result = await service.execute(params.command, signal);
    
    return {
      summary: `Executed: ${params.command}`,
      llmContent: this.formatResult(result),
      returnDisplay: this.formatResult(result)
    };
  }
}
```

#### MemoryTool - 记忆管理工具

管理项目和用户级别的持久化记忆：

```typescript
class MemoryTool extends BaseTool<MemoryToolParams, ToolResult> {
  static Name = 'save_memory';
  
  async execute(params: MemoryToolParams): Promise<ToolResult> {
    const { scope, key, value } = params;
    
    // 确定存储位置
    const memoryPath = scope === 'project' 
      ? path.join(this.config.getProjectRoot(), '.gemini', 'memory')
      : path.join(this.config.getUserSettingsDir(), 'memory');
    
    // 创建内存文件
    const fileName = `${key}.md`;
    const filePath = path.join(memoryPath, fileName);
    
    await fs.promises.mkdir(memoryPath, { recursive: true });
    await fs.promises.writeFile(filePath, value, 'utf-8');
    
    return {
      summary: `Saved ${scope} memory: ${key}`,
      llmContent: `Memory saved to ${filePath}`,
      returnDisplay: `Saved ${key} to ${scope} memory`
    };
  }
}

## 3.6 MCP 工具集成

### 3.6.1 MCP 工具发现

MCP（Model Context Protocol）允许外部服务器提供工具，支持多种传输协议：

```typescript
// MCP 服务器配置
interface MCPServerConfig {
  transport: 'stdio' | 'sse' | 'http';
  command?: string;              // stdio 传输使用
  args?: string[];               // stdio 传输使用
  url?: string;                  // http/sse 传输使用
  headers?: Record<string, string>;
  authProvider?: AuthProviderType;
  sseOptions?: SSEClientTransportOptions;
  httpOptions?: StreamableHTTPClientTransportOptions;
}

async function discoverMcpTools(
  mcpServers: Record<string, McpServerConfig>,
  toolRegistry: ToolRegistry,
  promptRegistry: PromptRegistry
): Promise<void> {
  // 更新发现状态
  mcpDiscoveryState = MCPDiscoveryState.IN_PROGRESS;
  
  for (const [serverName, config] of Object.entries(mcpServers)) {
    try {
      // 更新服务器状态
      serverStatuses.set(serverName, MCPServerStatus.CONNECTING);
      notifyStatusChange(serverName, MCPServerStatus.CONNECTING);
      
      // 创建 MCP 客户端
      const client = await createMcpClient(serverName, config);
      
      // 初始化客户端
      await client.initialize();
      
      // 发现工具
      const toolsResult = await client.request(
        { method: 'tools/list' },
        ListToolsResultSchema
      );
      
      // 注册每个工具
      for (const tool of toolsResult.tools) {
        const mcpTool = new DiscoveredMCPTool(
          serverName,
          client,
          mcpToTool(tool)
        );
        toolRegistry.registerTool(mcpTool.asFullyQualifiedTool());
      }
      
      // 发现提示词
      const promptsResult = await client.request(
        { method: 'prompts/list' },
        ListPromptsResultSchema
      );
      
      for (const prompt of promptsResult.prompts) {
        promptRegistry.registerPrompt({
          ...prompt,
          serverName,
          invoke: async (params) => client.getPrompt(prompt.name, params)
        });
      }
      
      // 更新状态为已连接
      serverStatuses.set(serverName, MCPServerStatus.CONNECTED);
      notifyStatusChange(serverName, MCPServerStatus.CONNECTED);
      
    } catch (error) {
      // 处理连接错误
      console.error(`Failed to connect to MCP server ${serverName}:`, error);
      serverStatuses.set(serverName, MCPServerStatus.DISCONNECTED);
      notifyStatusChange(serverName, MCPServerStatus.DISCONNECTED);
    }
  }
  
  mcpDiscoveryState = MCPDiscoveryState.COMPLETED;
}
```

#### 创建 MCP 客户端

```typescript
async function createMcpClient(
  serverName: string,
  config: McpServerConfig
): Promise<Client> {
  const { transport: mcpTransportType } = config;
  let transport: Transport;
  
  switch (mcpTransportType) {
    case 'stdio': {
      // 使用标准输入输出传输
      const command = config.command!;
      const args = config.args || [];
      transport = new StdioClientTransport({
        command,
        args,
        env: process.env,
      });
      break;
    }
    
    case 'sse': {
      // 使用服务器发送事件传输
      const authHeaders = await getAuthHeaders(serverName, config);
      transport = new SSEClientTransport(
        new URL(config.url!),
        {
          ...config.sseOptions,
          headers: { ...config.headers, ...authHeaders },
        }
      );
      break;
    }
    
    case 'http': {
      // 使用 HTTP 传输
      const authHeaders = await getAuthHeaders(serverName, config);
      transport = new StreamableHTTPClientTransport(
        new URL(config.url!),
        {
          ...config.httpOptions,
          headers: { ...config.headers, ...authHeaders },
        }
      );
      break;
    }
  }
  
  const client = new Client({ name: 'gemini-cli', version: '1.0.0' }, {});
  await client.connect(transport);
  return client;
}
```

### 3.6.2 DiscoveredMCPTool 实现

```typescript
class DiscoveredMCPTool extends BaseTool<unknown, ToolResult> {
  constructor(
    private readonly serverName: string,
    private readonly client: Client,
    schema: FunctionDeclaration,
    private readonly fqName?: string
  ) {
    const toolName = fqName || schema.name;
    const displayName = fqName 
      ? `${schema.name} (${serverName})` 
      : schema.name;
      
    super(
      toolName,
      displayName,
      schema.description || '',
      Icon.Plug,
      schema.parameters || {},
      true,  // isOutputMarkdown
      false  // canUpdateOutput
    );
  }
  
  async shouldConfirmExecute(
    params: unknown,
    signal: AbortSignal
  ): Promise<ToolMCPConfirmationDetails | false> {
    // 检查服务器是否需要 OAuth
    if (mcpServerRequiresOAuth.get(this.serverName)) {
      return false; // 已授权，不需要确认
    }
    
    return {
      type: 'mcp',
      title: `Call MCP Tool: ${this.displayName}`,
      detail: `Server: ${this.serverName}\nTool: ${this.name}\nParams: ${JSON.stringify(params, null, 2)}`,
      onConfirm: async (outcome) => {
        if (outcome === ToolConfirmationOutcome.ProceedAlways) {
          // 记录服务器不需要每次确认
          this.markServerTrusted(this.serverName);
        }
      }
    };
  }
  
  async execute(params: unknown, signal: AbortSignal): Promise<ToolResult> {
    try {
      // 设置超时
      const timeoutMs = MCP_DEFAULT_TIMEOUT_MSEC;
      const timeoutPromise = new Promise<never>((_, reject) => {
        setTimeout(() => reject(new Error('MCP tool timeout')), timeoutMs);
      });
      
      // 调用 MCP 工具
      const resultPromise = this.client.request(
        {
          method: 'tools/call',
          params: {
            name: this.schema.name,
            arguments: params as Record<string, unknown>,
          },
        },
        CallToolResultSchema
      );
      
      // 等待结果或超时
      const result = await Promise.race([resultPromise, timeoutPromise]);
      
      // 转换结果
      const content = result.content;
      const llmContent = this.convertToPartList(content);
      const display = this.formatForDisplay(content);
      
      return {
        summary: `Called ${this.displayName}`,
        llmContent,
        returnDisplay: display
      };
      
    } catch (error) {
      // 错误处理
      if (error instanceof Error && error.message.includes('OAuth')) {
        mcpServerRequiresOAuth.set(this.serverName, true);
      }
      throw error;
    }
  }
  
  asFullyQualifiedTool(): DiscoveredMCPTool {
    // 创建全限定名工具，避免名称冲突
    if (this.fqName) return this;
    
    const fqName = `${this.serverName}__${this.name}`;
    return new DiscoveredMCPTool(
      this.serverName,
      this.client,
      this.schema,
      fqName
    );
  }
  
  private convertToPartList(content: unknown): PartListUnion {
    // 将 MCP 内容转换为 Gemini API 格式
    if (Array.isArray(content)) {
      return content.map(item => this.convertContentItem(item));
    }
    return this.convertContentItem(content);
  }
}

## 3.7 CLI 层集成

### 3.7.1 React Hook 集成

`useReactToolScheduler` Hook 将核心调度器与 React UI 连接：

```typescript
function useReactToolScheduler(
  onComplete: (tools: CompletedToolCall[]) => void,
  config: Config
): [TrackedToolCall[], ScheduleFn, MarkToolsAsSubmittedFn] {
  const [toolCalls, setToolCalls] = useState<TrackedToolCall[]>([]);
  
  const scheduler = useMemo(() => 
    new CoreToolScheduler({
      toolRegistry,
      outputUpdateHandler: (callId, output) => {
        // 更新 UI 中的实时输出
      },
      onToolCallsUpdate: (calls) => {
        // 转换核心状态到 UI 状态
        setToolCalls(convertToTrackedCalls(calls));
      },
      onAllToolCallsComplete
    })
  , []);
  
  return [toolCalls, scheduler.schedule, markAsSubmitted];
}
```

### 3.7.2 工具确认 UI

不同类型的确认界面：

1. **编辑确认**：显示文件差异，支持修改
2. **执行确认**：显示将执行的命令
3. **MCP 确认**：显示服务器和工具信息
4. **信息确认**：通用确认对话框

## 3.8 扩展工具系统

### 3.8.1 创建自定义内置工具

```typescript
class MyCustomTool extends BaseTool<MyParams, ToolResult> {
  constructor() {
    super(
      'my_tool',              // 内部名称
      'My Custom Tool',       // 显示名称
      'Description...',       // 描述
      Icon.Hammer,            // 图标
      {                       // 参数 schema
        properties: {
          param1: { type: Type.STRING }
        },
        required: ['param1']
      }
    );
  }
  
  async execute(params: MyParams): Promise<ToolResult> {
    // 实现工具逻辑
    return {
      summary: 'Executed successfully',
      llmContent: 'Result for LLM',
      returnDisplay: 'Result for user'
    };
  }
}
```

### 3.8.2 通过命令行暴露工具

创建工具发现脚本：

```javascript
// list-tools.js
const tools = [
  {
    name: 'my_external_tool',
    description: 'External tool description',
    parameters: {
      properties: {
        input: { type: 'string' }
      }
    }
  }
];

console.log(JSON.stringify(tools));
```

配置工具命令：
```json
{
  "toolDiscoveryCommand": "node list-tools.js",
  "toolCallCommand": "node call-tool.js"
}
```

### 3.8.3 实现 MCP 服务器

遵循 MCP 协议规范，提供工具列表和执行端点。

## 本章小结

Gemini CLI 的工具系统展现了优秀的架构设计：

1. **统一抽象**：通过 Tool 接口统一了不同来源的工具
2. **安全机制**：内置的确认流程保护用户免受危险操作
3. **扩展性强**：支持多种方式添加新工具
4. **状态管理**：清晰的状态机管理工具执行生命周期
5. **流式支持**：原生支持实时输出更新
6. **错误处理**：完善的错误捕获和报告机制

工具系统是 Gemini CLI 将 AI 能力转化为实际操作的关键桥梁，其设计充分考虑了安全性、扩展性和用户体验。通过这个系统，AI 助手可以安全、高效地执行各种任务，从简单的文件读取到复杂的代码修改，为开发者提供了强大的自动化能力。