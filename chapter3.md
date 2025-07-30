# 第 3 章：工具系统架构

工具系统是 Gemini CLI 的核心功能之一，它使 AI 能够执行各种操作，如文件读写、代码编辑、命令执行等。本章深入剖析工具系统的设计与实现，包括工具的定义、注册、发现、调度和执行的完整生命周期。

## 3.1 工具系统概述

### 3.1.1 设计理念

Gemini CLI 的工具系统采用了插件化架构，具有以下特点：

1. **统一接口**：所有工具实现相同的 `Tool` 接口，确保一致的行为
2. **声明式定义**：工具通过 schema 声明参数和功能，与 Gemini API 无缝集成
3. **安全执行**：内置确认机制，防止危险操作
4. **可扩展性**：支持内置工具、外部命令工具和 MCP 协议工具

### 3.1.2 工具类型

系统支持三种类型的工具：

- **内置工具**：直接在代码中实现的工具（如文件操作、搜索等）
- **发现工具**：通过外部命令动态发现的工具
- **MCP 工具**：通过 Model Context Protocol 服务器提供的工具

## 3.2 工具接口与基类

### 3.2.1 Tool 接口定义

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

```typescript
interface ToolResult {
  summary?: string;              // 一行总结
  llmContent: PartListUnion;     // 供 LLM 使用的内容
  returnDisplay: ToolResultDisplay; // 用户界面显示内容
}

type ToolResultDisplay = string | FileDiff;

interface FileDiff {
  fileDiff: string;
  fileName: string;
  originalContent: string | null;
  newContent: string;
}
```

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
  async execute(params: ToolParams): Promise<ToolResult> {
    // 通过子进程执行 toolCallCommand
    const child = spawn(this.config.getToolCallCommand(), [this.name]);
    child.stdin.write(JSON.stringify(params));
    // 处理输出、错误、退出码等
  }
}
```

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

`CoreToolScheduler` 管理工具执行的完整生命周期：

```typescript
class CoreToolScheduler {
  private toolCalls: ToolCall[] = [];
  
  async schedule(
    request: ToolCallRequestInfo | ToolCallRequestInfo[],
    signal: AbortSignal
  ): Promise<void> {
    // 验证没有正在运行的工具
    if (this.isRunning()) {
      throw new Error('Cannot schedule while tools are running');
    }
    
    // 创建工具调用实例
    const newToolCalls = this.createToolCalls(requests);
    
    // 处理每个工具调用
    for (const toolCall of newToolCalls) {
      await this.processToolCall(toolCall, signal);
    }
  }
}
```

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

工具执行前的确认流程：

1. 调用 `shouldConfirmExecute()` 检查是否需要确认
2. 返回确认详情（编辑、执行、MCP、信息类型）
3. 等待用户响应
4. 根据响应处理：
   - `ProceedOnce`: 执行一次
   - `ProceedAlways`: 总是执行（本会话）
   - `ModifyWithEditor`: 使用编辑器修改
   - `Cancel`: 取消执行

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

`EditTool` 是最复杂的内置工具之一，实现了 `ModifiableTool` 接口：

```typescript
class EditTool extends BaseTool implements ModifiableTool {
  async shouldConfirmExecute(params: EditToolParams): 
    Promise<ToolEditConfirmationDetails | false> {
    const { currentContent, newContent, error } = 
      await this.calculateEdit(params);
      
    if (error) return false;
    
    const fileDiff = Diff.createPatch(
      fileName, currentContent, newContent
    );
    
    return {
      type: 'edit',
      title: `Edit ${fileName}`,
      fileName,
      fileDiff,
      originalContent: currentContent,
      newContent,
      onConfirm: async (outcome, payload) => {
        // 处理用户确认
      }
    };
  }
  
  async execute(params: EditToolParams): Promise<ToolResult> {
    const { newContent, isNewFile } = await this.calculateEdit(params);
    
    // 写入文件
    await fs.promises.writeFile(params.file_path, newContent);
    
    return {
      summary: isNewFile ? 
        `Created ${fileName}` : 
        `Modified ${fileName}`,
      llmContent: /* ... */,
      returnDisplay: { fileDiff, fileName, originalContent, newContent }
    };
  }
}
```

关键特性：
- 精确的字符串匹配和替换
- 支持创建新文件
- 用户可修改建议的更改
- 显示差异对比
- 多次替换支持

### 3.5.2 文件操作工具集

其他文件操作工具的简要说明：

- **ReadFileTool**: 读取文件内容，添加行号
- **WriteFileTool**: 创建或覆盖文件
- **GlobTool**: 基于模式搜索文件
- **GrepTool**: 使用正则表达式搜索内容
- **ListDirectoryTool**: 列出目录内容

### 3.5.3 系统工具

- **ShellTool**: 执行 shell 命令，支持确认和流式输出
- **MemoryTool**: 管理项目级和用户级记忆
- **WebFetchTool**: 获取网页内容
- **WebSearchTool**: 执行网络搜索

## 3.6 MCP 工具集成

### 3.6.1 MCP 工具发现

MCP（Model Context Protocol）允许外部服务器提供工具：

```typescript
async function discoverMcpTools(
  mcpServers: Record<string, McpServerConfig>,
  toolRegistry: ToolRegistry
) {
  for (const [serverName, config] of Object.entries(mcpServers)) {
    const client = await createMcpClient(serverName, config);
    
    // 列出服务器提供的工具
    const tools = await client.listTools();
    
    // 注册每个工具
    for (const tool of tools) {
      const mcpTool = new DiscoveredMCPTool(
        serverName, client, tool
      );
      toolRegistry.registerTool(mcpTool);
    }
  }
}
```

### 3.6.2 DiscoveredMCPTool 实现

```typescript
class DiscoveredMCPTool extends BaseTool {
  async execute(params: unknown): Promise<ToolResult> {
    // 通过 JSON-RPC 调用 MCP 服务器
    const result = await this.client.callTool(
      this.toolName, params
    );
    
    return {
      llmContent: result.content,
      returnDisplay: this.formatDisplay(result)
    };
  }
  
  asFullyQualifiedTool(): DiscoveredMCPTool {
    // 处理工具名称冲突
    return new DiscoveredMCPTool(
      /* 使用 serverName__toolName 格式 */
    );
  }
}
```

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