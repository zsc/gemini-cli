# 第 11 章：VSCode 扩展实现

本章深入剖析 Gemini CLI 的 VSCode 扩展（IDE Companion）的架构设计与实现细节。该扩展通过 Model Context Protocol (MCP) 协议，让 Gemini CLI 能够感知 VSCode 中的编辑器状态，包括打开的文件、光标位置和选中的文本，从而为 AI 提供更丰富的上下文信息。

## 11.1 扩展架构概览

### 11.1.1 设计目标

VSCode IDE Companion 扩展的设计围绕以下核心目标展开：

1. **无缝集成**：用户在 VSCode 中正常工作，无需额外操作即可让 Gemini CLI 获取编辑器上下文
2. **实时同步**：编辑器状态变化能够实时传递给 CLI，确保 AI 始终拥有最新的上下文
3. **轻量级实现**：最小化对 VSCode 性能的影响，避免干扰用户的正常开发流程
4. **安全隔离**：确保只有在相同工作区目录下运行的 CLI 才能访问编辑器信息

### 11.1.2 核心组件

扩展由以下核心组件构成：

```typescript
// 组件架构
VSCode Extension
├── extension.ts          // 扩展入口，生命周期管理
├── ide-server.ts        // MCP 服务器实现
├── open-files-manager.ts // 工作区状态跟踪
└── utils/logger.ts      // 日志工具
```

各组件职责明确：

- **Extension Module**: 管理扩展的激活、停用，设置环境变量，注册命令
- **IDE Server**: 实现 MCP 协议服务器，处理与 CLI 的通信
- **Open Files Manager**: 跟踪编辑器状态，包括打开的文件、光标位置、选中文本
- **Logger**: 提供统一的日志输出，便于调试和问题诊断

### 11.1.3 通信架构

扩展采用基于 MCP (Model Context Protocol) 的通信架构：

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│   VSCode    │ Events  │  Extension   │  MCP    │  Gemini CLI │
│   Editor    ├────────>│   Server     │<────────┤   Client    │
└─────────────┘         └──────────────┘         └─────────────┘
      │                        │                         │
      │                        ▼                         │
      │                 ┌──────────────┐                │
      │                 │ Open Files   │                │
      └────────────────>│  Manager     │                │
                        └──────────────┘                │
                               │                        │
                               └── Context Updates ─────┘
```

通信流程：
1. VSCode 编辑器事件触发（文件打开、光标移动等）
2. OpenFilesManager 捕获并处理这些事件
3. IDE Server 通过 MCP 协议发送上下文更新通知
4. CLI 端接收通知并更新本地上下文存储

## 11.2 扩展生命周期管理

### 11.2.1 扩展激活机制

扩展配置为在 VSCode 启动完成后自动激活：

```json
// package.json
{
  "activationEvents": [
    "onStartupFinished"  // VSCode 启动完成后激活
  ]
}
```

激活入口函数实现：

```typescript
// extension.ts
export async function activate(context: vscode.ExtensionContext) {
  // 1. 创建日志输出通道
  logger = vscode.window.createOutputChannel('Gemini CLI IDE Companion');
  log = createLogger(context, logger);
  log('Extension activated');

  // 2. 设置工作区路径环境变量
  updateWorkspacePath(context);

  // 3. 启动 IDE 服务器
  ideServer = new IDEServer(log);
  try {
    await ideServer.start(context);
  } catch (err) {
    const message = err instanceof Error ? err.message : String(err);
    log(`Failed to start IDE server: ${message}`);
  }

  // 4. 注册事件监听和命令
  context.subscriptions.push(
    vscode.workspace.onDidChangeWorkspaceFolders(() => {
      updateWorkspacePath(context);
    }),
    vscode.commands.registerCommand('gemini-cli.runGeminiCLI', () => {
      // 创建终端并运行 gemini 命令
      const terminal = vscode.window.createTerminal(`Gemini CLI`);
      terminal.show();
      terminal.sendText('gemini');
    }),
  );
}
```

### 11.2.2 环境变量设置

扩展通过环境变量与 CLI 通信，确保只有在正确的终端会话中才能连接：

```typescript
function updateWorkspacePath(context: vscode.ExtensionContext) {
  const workspaceFolders = vscode.workspace.workspaceFolders;
  
  if (workspaceFolders && workspaceFolders.length === 1) {
    // 单一工作区：设置工作区路径
    const workspaceFolder = workspaceFolders[0];
    context.environmentVariableCollection.replace(
      'GEMINI_CLI_IDE_WORKSPACE_PATH',
      workspaceFolder.uri.fsPath,
    );
  } else {
    // 多工作区或无工作区：清空环境变量
    context.environmentVariableCollection.replace(
      'GEMINI_CLI_IDE_WORKSPACE_PATH',
      '',
    );
  }
}
```

环境变量的作用：
- `GEMINI_CLI_IDE_WORKSPACE_PATH`: 告知 CLI 当前 VSCode 打开的工作区路径
- `GEMINI_CLI_IDE_SERVER_PORT`: 动态分配的 MCP 服务器端口号

### 11.2.3 服务器启动流程

IDE 服务器在扩展激活时启动，监听动态分配的端口：

```typescript
// ide-server.ts
async start(context: vscode.ExtensionContext) {
  this.context = context;
  
  // 1. 创建 Express 应用和 MCP 服务器
  const app = express();
  app.use(express.json());
  const mcpServer = createMcpServer();

  // 2. 初始化文件管理器
  const openFilesManager = new OpenFilesManager(context);
  
  // 3. 监听文件变化，发送更新通知
  const onDidChangeSubscription = openFilesManager.onDidChange(() => {
    for (const transport of Object.values(transports)) {
      sendIdeContextUpdateNotification(transport, this.log, openFilesManager);
    }
  });
  context.subscriptions.push(onDidChangeSubscription);

  // 4. 配置 MCP 端点
  app.post('/mcp', async (req, res) => {
    // 处理 MCP 协议请求
  });

  // 5. 启动服务器，监听随机端口
  this.server = app.listen(0, () => {
    const address = this.server.address();
    if (address && typeof address !== 'string') {
      const port = address.port;
      // 设置端口环境变量
      context.environmentVariableCollection.replace(
        'GEMINI_CLI_IDE_SERVER_PORT',
        port.toString(),
      );
      this.log(`IDE server listening on port ${port}`);
    }
  });
}
```

### 11.2.4 扩展停用处理

扩展停用时需要正确清理资源：

```typescript
export async function deactivate(): Promise<void> {
  log('Extension deactivated');
  
  try {
    // 1. 停止 IDE 服务器
    if (ideServer) {
      await ideServer.stop();
    }
  } catch (err) {
    const message = err instanceof Error ? err.message : String(err);
    log(`Failed to stop IDE server during deactivation: ${message}`);
  } finally {
    // 2. 释放日志通道
    if (logger) {
      logger.dispose();
    }
  }
}
```

服务器停止的实现确保所有连接正确关闭：

```typescript
async stop(): Promise<void> {
  if (this.server) {
    // 1. 关闭 HTTP 服务器
    await new Promise<void>((resolve, reject) => {
      this.server!.close((err?: Error) => {
        if (err) {
          this.log(`Error shutting down IDE server: ${err.message}`);
          return reject(err);
        }
        this.log(`IDE server shut down`);
        resolve();
      });
    });
    this.server = undefined;
  }

  // 2. 清理环境变量
  if (this.context) {
    this.context.environmentVariableCollection.clear();
  }
}

## 11.3 MCP 服务器实现

### 11.3.1 HTTP 服务器架构

IDE Server 基于 Express.js 构建，实现了 MCP 协议的 HTTP 传输层：

```typescript
const createMcpServer = () => {
  const server = new McpServer(
    {
      name: 'gemini-cli-companion-mcp-server',
      version: '1.0.0',
    },
    { capabilities: { logging: {} } },
  );
  return server;
};
```

服务器采用 Streamable HTTP 传输模式，支持长连接和流式通信：

```typescript
// 主要的 MCP 端点处理
app.post('/mcp', async (req: Request, res: Response) => {
  const sessionId = req.headers[MCP_SESSION_ID_HEADER] as string | undefined;
  let transport: StreamableHTTPServerTransport;

  if (sessionId && transports[sessionId]) {
    // 已存在的会话：使用现有传输层
    transport = transports[sessionId];
  } else if (!sessionId && isInitializeRequest(req.body)) {
    // 新会话：创建传输层
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      onsessioninitialized: (newSessionId) => {
        this.log(`New session initialized: ${newSessionId}`);
        transports[newSessionId] = transport;
      },
    });
    
    // 连接 MCP 服务器
    mcpServer.connect(transport);
  } else {
    // 无效请求：返回错误
    res.status(400).json({
      jsonrpc: '2.0',
      error: {
        code: -32000,
        message: 'Bad Request: No valid session ID provided',
      },
      id: null,
    });
    return;
  }

  // 处理请求
  await transport.handleRequest(req, res, req.body);
});
```

### 11.3.2 会话管理机制

服务器支持多个并发的 CLI 会话，每个会话都有独立的状态：

```typescript
// 会话存储
const transports: { [sessionId: string]: StreamableHTTPServerTransport } = {};
const sessionsWithInitialNotification = new Set<string>();

// 会话生命周期管理
transport.onclose = () => {
  clearInterval(keepAlive);
  if (transport.sessionId) {
    this.log(`Session closed: ${transport.sessionId}`);
    sessionsWithInitialNotification.delete(transport.sessionId);
    delete transports[transport.sessionId];
  }
};
```

会话管理的关键特性：
1. **会话隔离**：每个 CLI 实例有独立的会话 ID
2. **状态跟踪**：记录哪些会话已发送初始通知
3. **自动清理**：会话关闭时自动清理相关资源

### 11.3.3 消息处理流程

MCP 消息处理分为两个主要流程：

**1. 初始化流程**：
```typescript
if (!sessionId && isInitializeRequest(req.body)) {
  // 创建新会话
  transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: () => randomUUID(),
    onsessioninitialized: (newSessionId) => {
      transports[newSessionId] = transport;
    },
  });
  
  // 设置 keep-alive
  const keepAlive = setInterval(() => {
    try {
      transport.send({ jsonrpc: '2.0', method: 'ping' });
    } catch (e) {
      clearInterval(keepAlive);
    }
  }, 60000);
}
```

**2. 上下文更新通知**：
```typescript
function sendIdeContextUpdateNotification(
  transport: StreamableHTTPServerTransport,
  log: (message: string) => void,
  openFilesManager: OpenFilesManager,
) {
  const ideContext = openFilesManager.state;

  const notification: JSONRPCNotification = {
    jsonrpc: '2.0',
    method: 'ide/contextUpdate',
    params: ideContext,
  };
  
  log(`Sending IDE context update notification: ${JSON.stringify(notification, null, 2)}`);
  transport.send(notification);
}
```

### 11.3.4 Keep-alive 机制

为了维持长连接的稳定性，服务器实现了 keep-alive 机制：

```typescript
const keepAlive = setInterval(() => {
  try {
    // 每 60 秒发送一次 ping
    transport.send({ jsonrpc: '2.0', method: 'ping' });
  } catch (e) {
    // 发送失败，清理定时器
    this.log('Failed to send keep-alive ping, cleaning up interval.' + e);
    clearInterval(keepAlive);
  }
}, 60000); // 60 秒间隔
```

Keep-alive 的作用：
- 防止网络中间件（如代理、防火墙）因超时而关闭连接
- 及时检测连接断开，触发资源清理
- 保证 CLI 能够持续接收编辑器状态更新

## 11.4 工作区状态跟踪

### 11.4.1 OpenFilesManager 设计

OpenFilesManager 是扩展的核心组件，负责跟踪和管理编辑器状态：

```typescript
export class OpenFilesManager {
  private readonly onDidChangeEmitter = new vscode.EventEmitter<void>();
  readonly onDidChange = this.onDidChangeEmitter.event;
  private debounceTimer: NodeJS.Timeout | undefined;
  private openFiles: File[] = [];  // 最多保存 10 个文件

  constructor(private readonly context: vscode.ExtensionContext) {
    // 注册各种编辑器事件监听器
    this.registerEventListeners();
    
    // 初始化：添加当前活动文件
    if (vscode.window.activeTextEditor && 
        this.isFileUri(vscode.window.activeTextEditor.document.uri)) {
      this.addOrMoveToFront(vscode.window.activeTextEditor);
    }
  }
}
```

数据结构设计：
```typescript
interface File {
  path: string;           // 文件绝对路径
  timestamp: number;      // 添加/更新时间戳
  isActive?: boolean;     // 是否为当前活动文件
  selectedText?: string;  // 选中的文本（限制 16KB）
  cursor?: {              // 光标位置
    line: number;
    character: number;
  };
}
```

### 11.4.2 文件变化监听

OpenFilesManager 监听多种 VSCode 编辑器事件：

```typescript
private registerEventListeners() {
  const { context } = this;
  
  // 1. 活动编辑器变化
  const editorWatcher = vscode.window.onDidChangeActiveTextEditor((editor) => {
    if (editor && this.isFileUri(editor.document.uri)) {
      this.addOrMoveToFront(editor);
      this.fireWithDebounce();
    }
  });

  // 2. 文本选择变化
  const selectionWatcher = vscode.window.onDidChangeTextEditorSelection((event) => {
    if (this.isFileUri(event.textEditor.document.uri)) {
      this.updateActiveContext(event.textEditor);
      this.fireWithDebounce();
    }
  });

  // 3. 文档关闭
  const closeWatcher = vscode.workspace.onDidCloseTextDocument((document) => {
    if (this.isFileUri(document.uri)) {
      this.remove(document.uri);
      this.fireWithDebounce();
    }
  });

  // 4. 文件删除
  const deleteWatcher = vscode.workspace.onDidDeleteFiles((event) => {
    for (const uri of event.files) {
      if (this.isFileUri(uri)) {
        this.remove(uri);
      }
    }
    this.fireWithDebounce();
  });

  // 5. 文件重命名
  const renameWatcher = vscode.workspace.onDidRenameFiles((event) => {
    for (const { oldUri, newUri } of event.files) {
      if (this.isFileUri(oldUri)) {
        if (this.isFileUri(newUri)) {
          this.rename(oldUri, newUri);
        } else {
          this.remove(oldUri);
        }
      }
    }
    this.fireWithDebounce();
  });

  // 注册所有监听器
  context.subscriptions.push(
    editorWatcher, selectionWatcher, closeWatcher, 
    deleteWatcher, renameWatcher
  );
}
```

### 11.4.3 光标与选区跟踪

活动文件的上下文信息实时更新：

```typescript
private updateActiveContext(editor: vscode.TextEditor) {
  const file = this.openFiles.find(
    (f) => f.path === editor.document.uri.fsPath,
  );
  
  if (!file || !file.isActive) {
    return;
  }

  // 更新光标位置（VSCode 使用 0-based，转换为 1-based）
  file.cursor = editor.selection.active
    ? {
        line: editor.selection.active.line + 1,
        character: editor.selection.active.character,
      }
    : undefined;

  // 更新选中文本
  let selectedText: string | undefined = 
    editor.document.getText(editor.selection) || undefined;
    
  // 限制选中文本大小
  if (selectedText && selectedText.length > MAX_SELECTED_TEXT_LENGTH) {
    selectedText = 
      selectedText.substring(0, MAX_SELECTED_TEXT_LENGTH) + '... [TRUNCATED]';
  }
  
  file.selectedText = selectedText;
}
```

文件列表管理策略：
```typescript
private addOrMoveToFront(editor: vscode.TextEditor) {
  // 1. 取消之前的活动文件状态
  const currentActive = this.openFiles.find((f) => f.isActive);
  if (currentActive) {
    currentActive.isActive = false;
    currentActive.cursor = undefined;
    currentActive.selectedText = undefined;
  }

  // 2. 移除已存在的同一文件
  const index = this.openFiles.findIndex(
    (f) => f.path === editor.document.uri.fsPath,
  );
  if (index !== -1) {
    this.openFiles.splice(index, 1);
  }

  // 3. 添加到列表最前面
  this.openFiles.unshift({
    path: editor.document.uri.fsPath,
    timestamp: Date.now(),
    isActive: true,
  });

  // 4. 限制列表长度
  if (this.openFiles.length > MAX_FILES) {
    this.openFiles.pop();
  }

  // 5. 更新上下文信息
  this.updateActiveContext(editor);
}
```

### 11.4.4 状态更新防抖

为了避免频繁的状态更新影响性能，实现了防抖机制：

```typescript
private fireWithDebounce() {
  if (this.debounceTimer) {
    clearTimeout(this.debounceTimer);
  }
  
  this.debounceTimer = setTimeout(() => {
    this.onDidChangeEmitter.fire();
  }, 50); // 50ms 防抖延迟
}
```

防抖的好处：
1. **性能优化**：避免快速连续的编辑操作产生大量通知
2. **网络优化**：减少不必要的网络传输
3. **用户体验**：确保 AI 接收到的是稳定的编辑器状态

状态获取接口：
```typescript
get state(): IdeContext {
  return {
    workspaceState: {
      openFiles: [...this.openFiles],  // 返回副本，避免外部修改
    },
  };
}

## 11.5 CLI 端集成实现

### 11.5.1 IdeClient 架构

IdeClient 是 CLI 端连接 VSCode 扩展的核心组件，负责建立和管理 MCP 连接：

```typescript
export class IdeClient {
  client: Client | undefined = undefined;
  private state: IDEConnectionState = {
    status: IDEConnectionStatus.Disconnected,
  };

  constructor() {
    // 构造时自动尝试初始化连接
    this.init().catch((err) => {
      logger.debug('Failed to initialize IdeClient:', err);
    });
  }

  getConnectionStatus(): IDEConnectionState {
    return this.state;
  }
}
```

连接状态管理：
```typescript
export enum IDEConnectionStatus {
  Connected = 'connected',
  Disconnected = 'disconnected',
  Connecting = 'connecting',
}

export type IDEConnectionState = {
  status: IDEConnectionStatus;
  details?: string; // 用户友好的错误信息
};
```

### 11.5.2 连接建立流程

连接建立遵循严格的验证流程，确保安全性：

```typescript
async init(): Promise<void> {
  if (this.state.status === IDEConnectionStatus.Connected) {
    return;
  }
  this.setState(IDEConnectionStatus.Connecting);

  // 1. 验证工作区路径
  if (!this.validateWorkspacePath()) {
    return;
  }

  // 2. 获取服务器端口
  const port = this.getPortFromEnv();
  if (!port) {
    return;
  }

  // 3. 建立连接
  await this.establishConnection(port);
}
```

**工作区验证**确保 CLI 在正确的目录运行：
```typescript
private validateWorkspacePath(): boolean {
  const ideWorkspacePath = process.env['GEMINI_CLI_IDE_WORKSPACE_PATH'];
  
  if (!ideWorkspacePath) {
    this.setState(
      IDEConnectionStatus.Disconnected,
      'IDE integration requires a single workspace folder to be open in the IDE.',
    );
    return false;
  }
  
  if (ideWorkspacePath !== process.cwd()) {
    this.setState(
      IDEConnectionStatus.Disconnected,
      `Gemini CLI is running in a different directory (${process.cwd()}) ` +
      `from the IDE's open workspace (${ideWorkspacePath}).`,
    );
    return false;
  }
  
  return true;
}
```

**连接建立过程**：
```typescript
private async establishConnection(port: string) {
  let transport: StreamableHTTPClientTransport | undefined;
  
  try {
    // 1. 创建 MCP 客户端
    this.client = new Client({
      name: 'streamable-http-client',
      version: '1.0.0',
    });

    // 2. 创建 HTTP 传输层
    transport = new StreamableHTTPClientTransport(
      new URL(`http://localhost:${port}/mcp`)
    );

    // 3. 注册事件处理器
    this.registerClientHandlers();

    // 4. 连接到服务器
    await this.client.connect(transport);

    // 5. 更新状态为已连接
    this.setState(IDEConnectionStatus.Connected);
  } catch (error) {
    this.setState(
      IDEConnectionStatus.Disconnected,
      `Failed to connect to IDE server: ${error}`,
    );
    
    // 清理资源
    if (transport) {
      try {
        await transport.close();
      } catch (closeError) {
        logger.debug('Failed to close transport:', closeError);
      }
    }
  }
}
```

### 11.5.3 状态同步机制

IdeClient 通过监听 MCP 通知来保持状态同步：

```typescript
private registerClientHandlers() {
  if (!this.client) {
    return;
  }

  // 注册 IDE 上下文更新通知处理器
  this.client.setNotificationHandler(
    IdeContextNotificationSchema,
    (notification) => {
      ideContext.setIdeContext(notification.params);
    },
  );

  // 错误处理
  this.client.onerror = (_error) => {
    this.setState(IDEConnectionStatus.Disconnected, 'Client error.');
  };

  // 连接关闭处理
  this.client.onclose = () => {
    this.setState(IDEConnectionStatus.Disconnected, 'Connection closed.');
  };
}
```

状态更新时的副作用：
```typescript
private setState(status: IDEConnectionStatus, details?: string) {
  this.state = { status, details };

  if (status === IDEConnectionStatus.Disconnected) {
    logger.debug('IDE integration is disconnected. ', details);
    // 清理本地上下文存储
    ideContext.clearIdeContext();
  }
}
```

### 11.5.4 错误处理策略

IdeClient 实现了多层次的错误处理：

1. **环境检查错误**：
```typescript
private getPortFromEnv(): string | undefined {
  const port = process.env['GEMINI_CLI_IDE_SERVER_PORT'];
  if (!port) {
    this.setState(
      IDEConnectionStatus.Disconnected,
      'Gemini CLI Companion extension not found. ' +
      'Install via /ide install and restart the CLI in a fresh terminal window.',
    );
    return undefined;
  }
  return port;
}
```

2. **连接错误**：
- 网络连接失败
- 服务器未响应
- 协议不匹配

3. **运行时错误**：
- MCP 协议错误
- 传输层异常
- 会话超时

所有错误都会更新连接状态，并提供用户友好的错误信息。

## 11.6 上下文数据流

### 11.6.1 数据结构定义

IDE 上下文使用 Zod 进行类型验证和定义：

```typescript
// 文件信息结构
export const FileSchema = z.object({
  path: z.string(),              // 文件绝对路径
  timestamp: z.number(),          // 时间戳
  isActive: z.boolean().optional(), // 是否为活动文件
  selectedText: z.string().optional(), // 选中的文本
  cursor: z.object({              // 光标位置
    line: z.number(),
    character: z.number(),
  }).optional(),
});

// IDE 上下文结构
export const IdeContextSchema = z.object({
  workspaceState: z.object({
    openFiles: z.array(FileSchema).optional(),
  }).optional(),
});

export type IdeContext = z.infer<typeof IdeContextSchema>;
```

### 11.6.2 通知协议设计

使用标准的 JSON-RPC 通知格式：

```typescript
// 通知模式定义
export const IdeContextNotificationSchema = z.object({
  method: z.literal('ide/contextUpdate'),
  params: IdeContextSchema,
});

// 通知示例
{
  "jsonrpc": "2.0",
  "method": "ide/contextUpdate",
  "params": {
    "workspaceState": {
      "openFiles": [
        {
          "path": "/Users/example/project/src/index.ts",
          "timestamp": 1704067200000,
          "isActive": true,
          "cursor": { "line": 42, "character": 15 },
          "selectedText": "function calculate()"
        },
        {
          "path": "/Users/example/project/src/utils.ts",
          "timestamp": 1704067100000
        }
      ]
    }
  }
}
```

### 11.6.3 上下文存储管理

ideContext 模块实现了一个响应式的上下文存储：

```typescript
export function createIdeContextStore() {
  let ideContextState: IdeContext | undefined = undefined;
  const subscribers = new Set<IdeContextSubscriber>();

  // 通知所有订阅者
  function notifySubscribers(): void {
    for (const subscriber of subscribers) {
      subscriber(ideContextState);
    }
  }

  // 设置新的上下文
  function setIdeContext(newIdeContext: IdeContext): void {
    ideContextState = newIdeContext;
    notifySubscribers();
  }

  // 清空上下文
  function clearIdeContext(): void {
    ideContextState = undefined;
    notifySubscribers();
  }

  // 获取当前上下文
  function getIdeContext(): IdeContext | undefined {
    return ideContextState;
  }

  // 订阅上下文变化
  function subscribeToIdeContext(subscriber: IdeContextSubscriber): () => void {
    subscribers.add(subscriber);
    return () => {
      subscribers.delete(subscriber);
    };
  }

  return {
    setIdeContext,
    getIdeContext,
    subscribeToIdeContext,
    clearIdeContext,
  };
}

// 全局单例
export const ideContext = createIdeContextStore();
```

### 11.6.4 提示词集成

IDE 上下文被无缝集成到发送给 Gemini 的提示词中：

```typescript
// 在 client.ts 中的实现
if (this.config.getIdeMode()) {
  const ideContextState = ideContext.getIdeContext();
  const openFiles = ideContextState?.workspaceState?.openFiles;
  
  if (openFiles && openFiles.length > 0) {
    const contextParts: string[] = [];
    const firstFile = openFiles[0];
    const activeFile = firstFile.isActive ? firstFile : undefined;
    
    // 1. 活动文件信息
    if (activeFile) {
      contextParts.push(
        `This is the file that the user is looking at:\n- Path: ${activeFile.path}`,
      );
      
      // 2. 光标位置
      if (activeFile.cursor) {
        contextParts.push(
          `This is the cursor position in the file:\n` +
          `- Cursor Position: Line ${activeFile.cursor.line}, ` +
          `Character ${activeFile.cursor.character}`,
        );
      }
      
      // 3. 选中文本
      if (activeFile.selectedText) {
        contextParts.push(
          `This is the selected text in the file:\n- ${activeFile.selectedText}`,
        );
      }
    }
    
    // 4. 其他打开的文件
    const otherOpenFiles = activeFile ? openFiles.slice(1) : openFiles;
    if (otherOpenFiles.length > 0) {
      const recentFiles = otherOpenFiles
        .map((file) => `- ${file.path}`)
        .join('\n');
      
      const heading = activeFile
        ? `Here are some other files the user has open, with the most recent at the top:`
        : `Here are some files the user has open, with the most recent at the top:`;
        
      contextParts.push(`${heading}\n${recentFiles}`);
    }
    
    // 将上下文添加到提示词
    if (contextParts.length > 0) {
      turns.push({
        role: 'user',
        content: contextParts.join('\n\n'),
      });
    }
  }
}
```

这种集成方式的优势：
1. **透明性**：用户无需手动提供文件信息
2. **实时性**：AI 始终获得最新的编辑器状态
3. **相关性**：只提供最相关的文件信息，避免信息过载
4. **灵活性**：可以根据需要扩展上下文类型

## 11.7 用户交互设计

### 11.7.1 命令注册

扩展注册了两类命令供用户使用：

**1. VSCode 命令**：
```typescript
// 在 VSCode 命令面板中可用
vscode.commands.registerCommand('gemini-cli.runGeminiCLI', () => {
  const geminiCmd = 'gemini';
  const terminal = vscode.window.createTerminal(`Gemini CLI`);
  terminal.show();
  terminal.sendText(geminiCmd);
})
```

**2. CLI 中的 /ide 命令**：
```typescript
export const ideCommand = (config: Config | null): SlashCommand | null => {
  if (!config?.getIdeMode()) {
    return null;
  }

  return {
    name: 'ide',
    description: 'manage IDE integration',
    kind: CommandKind.BUILT_IN,
    subCommands: [
      {
        name: 'status',
        description: 'check status of IDE integration',
        // ... 实现
      },
      {
        name: 'install',
        description: 'install required VS Code companion extension',
        // ... 实现
      },
    ],
  };
};
```

### 11.7.2 状态展示

IDE 连接状态通过直观的图标和消息展示：

```typescript
// /ide status 命令的实现
action: (_context: CommandContext): SlashCommandActionReturn => {
  const connection = config.getIdeClient()?.getConnectionStatus();
  
  switch (connection?.status) {
    case IDEConnectionStatus.Connected:
      return {
        type: 'message',
        messageType: 'info',
        content: `🟢 Connected`,
      };
      
    case IDEConnectionStatus.Connecting:
      return {
        type: 'message',
        messageType: 'info',
        content: `🟡 Connecting...`,
      };
      
    default: {
      let content = `🔴 Disconnected`;
      if (connection?.details) {
        content += `: ${connection.details}`;
      }
      return {
        type: 'message',
        messageType: 'error',
        content,
      };
    }
  }
}
```

状态指示器的设计原则：
- 🟢 绿色：正常连接
- 🟡 黄色：连接中
- 🔴 红色：断开连接，附带原因说明

### 11.7.3 安装流程

扩展提供了便捷的安装命令：

```typescript
// /ide install 命令实现
action: async (context) => {
  // 1. 检查 VSCode 是否已安装
  if (!isVSCodeInstalled()) {
    context.ui.addItem({
      type: 'error',
      text: `VS Code command-line tool "${VSCODE_COMMAND}" not found in your PATH.`,
    }, Date.now());
    return;
  }

  // 2. 查找 VSIX 文件
  const bundleDir = path.dirname(fileURLToPath(import.meta.url));
  let vsixFiles = glob.sync(path.join(bundleDir, '*.vsix'));
  
  // 开发环境回退逻辑
  if (vsixFiles.length === 0) {
    const devPath = path.join(
      bundleDir, '..', '..', '..', '..', '..', 
      VSCODE_COMPANION_EXTENSION_FOLDER, '*.vsix'
    );
    vsixFiles = glob.sync(devPath);
  }

  // 3. 执行安装
  const vsixPath = vsixFiles[0];
  const command = `${VSCODE_COMMAND} --install-extension ${vsixPath} --force`;
  
  try {
    child_process.execSync(command, { stdio: 'pipe' });
    context.ui.addItem({
      type: 'info',
      text: 'VS Code companion extension installed successfully. ' +
            'Restart gemini-cli in a fresh terminal window.',
    }, Date.now());
  } catch (_error) {
    context.ui.addItem({
      type: 'error',
      text: `Failed to install VS Code companion extension.`,
    }, Date.now());
  }
}
```

### 11.7.4 故障诊断

常见问题的诊断信息设计：

1. **扩展未安装**：
```
Gemini CLI Companion extension not found. 
Install via /ide install and restart the CLI in a fresh terminal window.
```

2. **工作区不匹配**：
```
Gemini CLI is running in a different directory (/path/to/cli) 
from the IDE's open workspace (/path/to/workspace).
```

3. **多工作区问题**：
```
IDE integration requires a single workspace folder to be open in the IDE.
```

诊断信息的设计原则：
- 清晰说明问题原因
- 提供具体的解决步骤
- 包含相关的路径信息便于调试

## 11.8 安全与性能考虑

### 11.8.1 工作区隔离

扩展实现了严格的工作区隔离机制：

1. **路径验证**：
```typescript
if (ideWorkspacePath !== process.cwd()) {
  // 拒绝连接：CLI 必须在相同目录运行
}
```

2. **环境变量隔离**：
- 环境变量仅对从 VSCode 启动的终端可见
- 外部终端无法访问这些环境变量

3. **会话隔离**：
- 每个 CLI 实例有独立的会话 ID
- 会话间状态完全隔离

### 11.8.2 数据限制策略

为防止资源滥用，实施了多项限制：

```typescript
export const MAX_FILES = 10;  // 最多跟踪 10 个文件
const MAX_SELECTED_TEXT_LENGTH = 16384; // 16 KiB 选中文本限制

// 文本截断实现
if (selectedText && selectedText.length > MAX_SELECTED_TEXT_LENGTH) {
  selectedText = 
    selectedText.substring(0, MAX_SELECTED_TEXT_LENGTH) + '... [TRUNCATED]';
}
```

限制的原因：
- 避免传输大量数据影响性能
- 防止 AI 上下文过载
- 保护用户隐私（避免意外传输敏感数据）

### 11.8.3 资源管理

扩展采用了细致的资源管理策略：

1. **防抖机制**：
```typescript
private fireWithDebounce() {
  if (this.debounceTimer) {
    clearTimeout(this.debounceTimer);
  }
  this.debounceTimer = setTimeout(() => {
    this.onDidChangeEmitter.fire();
  }, 50); // 50ms 防抖
}
```

2. **订阅管理**：
```typescript
context.subscriptions.push(
  editorWatcher, 
  selectionWatcher, 
  closeWatcher,
  deleteWatcher, 
  renameWatcher
);
```

3. **清理机制**：
- 扩展停用时自动清理所有资源
- 会话关闭时清理相关状态
- Keep-alive 失败时清理定时器

### 11.8.4 连接稳定性

为确保连接稳定性，实施了多项措施：

1. **Keep-alive 机制**：
- 每 60 秒发送 ping 保持连接活跃
- 自动检测和清理失效连接

2. **错误恢复**：
```typescript
this.client.onerror = (_error) => {
  this.setState(IDEConnectionStatus.Disconnected, 'Client error.');
};

this.client.onclose = () => {
  this.setState(IDEConnectionStatus.Disconnected, 'Connection closed.');
};
```

3. **状态一致性**：
- 断开连接时自动清理本地上下文
- 防止使用过期的编辑器状态

## 本章小结

VSCode IDE Companion 扩展是 Gemini CLI 生态系统的重要组成部分，通过精心设计的架构实现了 CLI 与 VSCode 的无缝集成。

**核心成就**：

1. **透明集成**：用户无需改变工作习惯，扩展自动捕获编辑器状态并传递给 AI
2. **实时同步**：通过 MCP 协议和防抖机制，实现了高效的状态同步
3. **安全隔离**：严格的工作区验证和会话隔离确保了安全性
4. **优雅降级**：即使扩展未安装或连接失败，CLI 仍可正常工作

**技术亮点**：

1. **MCP 协议应用**：展示了 Model Context Protocol 在实际场景中的应用
2. **环境变量通信**：巧妙利用 VSCode 的环境变量机制实现进程间通信
3. **响应式设计**：通过事件驱动和订阅模式实现了灵活的状态管理
4. **资源优化**：通过防抖、限制和清理机制确保了良好的性能

**未来展望**：

扩展的设计为未来功能扩展预留了空间：
- 支持更多编辑器状态（如断点、Git 状态等）
- 支持其他 IDE（如 JetBrains 系列）
- 增强的双向通信（允许 AI 控制编辑器）

通过本章的深入分析，我们看到了一个精心设计的 VSCode 扩展如何通过标准协议、清晰的架构和周到的用户体验设计，为 AI 辅助编程提供了强大的基础设施支持。