# 第 10 章：MCP 协议集成

Model Context Protocol (MCP) 是一个开放标准，允许 AI 助手与外部系统和工具进行安全、标准化的集成。本章深入探讨 Gemini CLI 如何实现对 MCP 协议的全面支持，包括服务器发现、工具映射、OAuth 认证等核心功能。

## 10.1 MCP 集成架构概览

### 10.1.1 核心组件结构

Gemini CLI 的 MCP 集成主要由以下几个核心组件构成：

1. **MCP Client（mcp-client.ts）**
   - 负责管理所有 MCP 服务器的连接生命周期
   - 协调服务器发现、工具注册和状态管理
   - 支持多种传输协议（Stdio、SSE、HTTP）

2. **MCP Tool System（mcp-tool.ts）**
   - 将 MCP 工具包装为 Gemini API 兼容的工具
   - 处理工具名称验证和转换
   - 实现安全确认机制

3. **OAuth Integration（oauth-*.ts）**
   - 完整的 OAuth 2.0 支持，包括 PKCE 流程
   - 动态客户端注册
   - 令牌管理和自动刷新

4. **Prompt System（McpPromptLoader.ts）**
   - 将 MCP prompts 自动转换为 CLI 命令
   - 参数解析和帮助生成

### 10.1.2 集成流程

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Config    │────▶│ MCP Client  │────▶│   Server    │
│  (servers)  │     │ (discover)  │     │ Connection  │
└─────────────┘     └─────────────┘     └─────────────┘
                            │
                            ▼
                    ┌─────────────┐
                    │    Tools    │
                    │  Discovery  │
                    └─────────────┘
                            │
                    ┌───────┴───────┐
                    ▼               ▼
            ┌─────────────┐ ┌─────────────┐
            │Tool Registry│ │Prompt Reg.  │
            └─────────────┘ └─────────────┘
```

## 10.2 服务器配置与发现

### 10.2.1 配置结构

MCP 服务器通过配置文件定义，支持多种传输方式：

```typescript
export class MCPServerConfig {
  constructor(
    // Stdio transport - 本地命令式服务器
    readonly command?: string,
    readonly args?: string[],
    readonly env?: Record<string, string>,
    readonly cwd?: string,
    
    // SSE transport - 服务器推送事件
    readonly url?: string,
    
    // HTTP transport - 请求响应式
    readonly httpUrl?: string,
    readonly headers?: Record<string, string>,
    
    // 通用配置
    readonly timeout?: number,
    readonly trust?: boolean,
    readonly includeTools?: string[],
    readonly excludeTools?: string[],
    
    // OAuth 配置
    readonly oauth?: MCPOAuthConfig,
    readonly authProviderType?: AuthProviderType,
  ) {}
}
```

### 10.2.2 服务器发现流程

`discoverMcpTools` 函数是服务器发现的入口点：

```typescript
export async function discoverMcpTools(
  mcpServers: Record<string, MCPServerConfig>,
  mcpServerCommand: string | undefined,
  toolRegistry: ToolRegistry,
  promptRegistry: PromptRegistry,
  debugMode: boolean,
): Promise<void> {
  mcpDiscoveryState = MCPDiscoveryState.IN_PROGRESS;
  try {
    // 处理命令行指定的服务器
    mcpServers = populateMcpServerCommand(mcpServers, mcpServerCommand);
    
    // 并行发现所有服务器
    const discoveryPromises = Object.entries(mcpServers).map(
      ([mcpServerName, mcpServerConfig]) =>
        connectAndDiscover(
          mcpServerName,
          mcpServerConfig,
          toolRegistry,
          promptRegistry,
          debugMode,
        ),
    );
    await Promise.all(discoveryPromises);
  } finally {
    mcpDiscoveryState = MCPDiscoveryState.COMPLETED;
  }
}
```

### 10.2.3 状态管理

MCP 集成实现了细粒度的状态管理：

```typescript
export enum MCPServerStatus {
  DISCONNECTED = 'disconnected',
  CONNECTING = 'connecting',
  CONNECTED = 'connected',
}

// 状态更新通知机制
function updateMCPServerStatus(
  serverName: string,
  status: MCPServerStatus,
): void {
  serverStatuses.set(serverName, status);
  // 通知所有监听器
  for (const listener of statusChangeListeners) {
    listener(serverName, status);
  }
}
```

## 10.3 传输层实现

### 10.3.1 传输类型选择

`createTransport` 函数根据配置选择合适的传输方式：

```typescript
async function createTransport(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  debugMode: boolean,
): Promise<Transport> {
  // Google 凭据提供者（特殊情况）
  if (mcpServerConfig.authProviderType === AuthProviderType.GOOGLE_CREDENTIALS) {
    const provider = new GoogleCredentialProvider(mcpServerConfig);
    // 创建带认证的传输
  }
  
  // HTTP 传输
  if (mcpServerConfig.httpUrl) {
    return new StreamableHTTPClientTransport(
      new URL(mcpServerConfig.httpUrl),
      transportOptions,
    );
  }
  
  // SSE 传输
  if (mcpServerConfig.url) {
    return new SSEClientTransport(
      new URL(mcpServerConfig.url),
      transportOptions,
    );
  }
  
  // Stdio 传输（本地进程）
  if (mcpServerConfig.command) {
    const transport = new StdioClientTransport({
      command: mcpServerConfig.command,
      args: mcpServerConfig.args || [],
      env: { ...process.env, ...mcpServerConfig.env },
      cwd: mcpServerConfig.cwd,
      stderr: 'pipe',
    });
    return transport;
  }
}
```

### 10.3.2 Stdio 传输详解

Stdio 传输用于本地 MCP 服务器，通过子进程通信：

- 支持自定义环境变量和工作目录
- 调试模式下捕获 stderr 输出
- 适用于本地工具和脚本集成

### 10.3.3 网络传输（SSE/HTTP）

网络传输支持远程 MCP 服务器：

- **SSE（Server-Sent Events）**：适用于长连接和流式通信
- **HTTP**：适用于请求-响应模式的交互
- 支持自定义请求头和超时设置

## 10.4 工具发现与映射

### 10.4.1 工具发现流程

MCP 服务器连接成功后，自动发现可用工具：

```typescript
export async function discoverTools(
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
  mcpClient: Client,
): Promise<DiscoveredMCPTool[]> {
  const mcpCallableTool = mcpToTool(mcpClient);
  const tool = await mcpCallableTool.tool();
  
  const discoveredTools: DiscoveredMCPTool[] = [];
  for (const funcDecl of tool.functionDeclarations) {
    if (!isEnabled(funcDecl, mcpServerName, mcpServerConfig)) {
      continue;
    }
    
    discoveredTools.push(
      new DiscoveredMCPTool(
        mcpCallableTool,
        mcpServerName,
        funcDecl.name!,
        funcDecl.description ?? '',
        funcDecl.parametersJsonSchema ?? { type: 'object', properties: {} },
        mcpServerConfig.timeout ?? MCP_DEFAULT_TIMEOUT_MSEC,
        mcpServerConfig.trust,
      ),
    );
  }
  return discoveredTools;
}
```

### 10.4.2 工具名称处理

由于 Gemini API 对函数名称有严格要求，需要进行名称转换：

```typescript
export function generateValidName(name: string) {
  // 替换无效字符为下划线
  let validToolname = name.replace(/[^a-zA-Z0-9_.-]/g, '_');
  
  // 如果超过 63 字符，截断并保留首尾
  if (validToolname.length > 63) {
    validToolname =
      validToolname.slice(0, 28) + '___' + validToolname.slice(-32);
  }
  return validToolname;
}
```

### 10.4.3 工具执行包装

`DiscoveredMCPTool` 类负责包装 MCP 工具：

```typescript
class DiscoveredMCPTool extends BaseTool<ToolParams, ToolResult> {
  async execute(params: ToolParams): Promise<ToolResult> {
    const functionCalls: FunctionCall[] = [{
      name: this.serverToolName,
      args: params,
    }];
    
    const responseParts: Part[] = await this.mcpTool.callTool(functionCalls);
    
    return {
      llmContent: responseParts,
      returnDisplay: getStringifiedResultForDisplay(responseParts),
    };
  }
}
```

## 10.5 Prompt 系统集成

### 10.5.1 Prompt 发现

MCP prompts 提供了预定义的对话模板：

```typescript
export async function discoverPrompts(
  mcpServerName: string,
  mcpClient: Client,
  promptRegistry: PromptRegistry,
): Promise<void> {
  const response = await mcpClient.request(
    { method: 'prompts/list', params: {} },
    ListPromptsResultSchema,
  );
  
  for (const prompt of response.prompts) {
    promptRegistry.registerPrompt({
      ...prompt,
      serverName: mcpServerName,
      invoke: (params: Record<string, unknown>) =>
        invokeMcpPrompt(mcpServerName, mcpClient, prompt.name, params),
    });
  }
}
```

### 10.5.2 Prompt 命令生成

`McpPromptLoader` 将 MCP prompts 转换为 CLI 命令：

```typescript
export class McpPromptLoader implements ICommandLoader {
  loadCommands(_signal: AbortSignal): Promise<SlashCommand[]> {
    const promptCommands: SlashCommand[] = [];
    
    for (const serverName in mcpServers) {
      const prompts = getMCPServerPrompts(this.config, serverName) || [];
      for (const prompt of prompts) {
        const newPromptCommand: SlashCommand = {
          name: prompt.name,
          description: prompt.description,
          kind: CommandKind.MCP_PROMPT,
          action: async (context, args) => {
            const promptInputs = this.parseArgs(args, prompt.arguments);
            const result = await prompt.invoke(promptInputs);
            return {
              type: 'submit_prompt',
              content: JSON.stringify(result.messages[0].content.text),
            };
          },
        };
        promptCommands.push(newPromptCommand);
      }
    }
    return Promise.resolve(promptCommands);
  }
}
```

## 10.6 OAuth 认证机制

### 10.6.1 认证流程概览

Gemini CLI 实现了完整的 OAuth 2.0 支持：

1. **自动检测**：401 错误触发 OAuth 流程
2. **配置发现**：通过 .well-known 端点获取配置
3. **动态注册**：无需预配置 client_id
4. **PKCE 流程**：增强安全性
5. **令牌管理**：安全存储和自动刷新

### 10.6.2 OAuth 发现机制

支持多种 OAuth 配置发现方式：

```typescript
static async discoverOAuthConfig(
  serverUrl: string,
): Promise<MCPOAuthConfig | null> {
  const wellKnownUrls = this.buildWellKnownUrls(serverUrl);
  
  // 1. 尝试获取受保护资源元数据
  const resourceMetadata = await this.fetchProtectedResourceMetadata(
    wellKnownUrls.protectedResource,
  );
  
  if (resourceMetadata?.authorization_servers?.length) {
    // 使用指定的授权服务器
    const authServerUrl = resourceMetadata.authorization_servers[0];
    const authServerMetadata = await this.fetchAuthorizationServerMetadata(
      authServerUrl,
    );
    
    if (authServerMetadata) {
      return this.metadataToOAuthConfig(authServerMetadata);
    }
  }
  
  // 2. 回退：尝试默认授权服务器端点
  const authServerMetadata = await this.fetchAuthorizationServerMetadata(
    wellKnownUrls.authorizationServer,
  );
  
  if (authServerMetadata) {
    return this.metadataToOAuthConfig(authServerMetadata);
  }
  
  return null;
}
```

### 10.6.3 PKCE 实现

使用 PKCE（Proof Key for Code Exchange）增强安全性：

```typescript
private static generatePKCEParams(): PKCEParams {
  // 生成 code verifier（43-128 字符）
  const codeVerifier = crypto.randomBytes(32).toString('base64url');
  
  // 生成 code challenge（SHA256）
  const codeChallenge = crypto
    .createHash('sha256')
    .update(codeVerifier)
    .digest('base64url');
  
  // 生成 state 防止 CSRF
  const state = crypto.randomBytes(16).toString('base64url');
  
  return { codeVerifier, codeChallenge, state };
}
```

### 10.6.4 令牌存储

OAuth 令牌安全存储在用户目录中：

```typescript
export class MCPOAuthTokenStorage {
  private static readonly TOKEN_FILE = 'mcp-oauth-tokens.json';
  private static readonly CONFIG_DIR = '.gemini';
  
  static async saveToken(
    serverName: string,
    token: MCPOAuthToken,
    clientId?: string,
    tokenUrl?: string,
    mcpServerUrl?: string,
  ): Promise<void> {
    await this.ensureConfigDir();
    const tokens = await this.loadTokens();
    
    const credential: MCPOAuthCredentials = {
      serverName,
      token,
      clientId,
      tokenUrl,
      mcpServerUrl,
      updatedAt: Date.now(),
    };
    
    tokens.set(serverName, credential);
    
    // 限制文件权限为 0600
    await fs.writeFile(
      tokenFile,
      JSON.stringify(tokenArray, null, 2),
      { mode: 0o600 },
    );
  }
}
```

## 10.7 安全模型

### 10.7.1 信任级别

MCP 集成实现了多层次的信任模型：

1. **可信服务器**：通过 `trust: true` 配置，无需确认
2. **服务器级允许列表**：用户确认后信任整个服务器
3. **工具级允许列表**：仅信任特定工具

### 10.7.2 确认机制

未信任的工具调用需要用户确认：

```typescript
async shouldConfirmExecute(
  _params: ToolParams,
  _abortSignal: AbortSignal,
): Promise<ToolCallConfirmationDetails | false> {
  const serverAllowListKey = this.serverName;
  const toolAllowListKey = `${this.serverName}.${this.serverToolName}`;
  
  if (this.trust) {
    return false; // 服务器可信，无需确认
  }
  
  if (
    DiscoveredMCPTool.allowlist.has(serverAllowListKey) ||
    DiscoveredMCPTool.allowlist.has(toolAllowListKey)
  ) {
    return false; // 已在允许列表中
  }
  
  return {
    type: 'mcp',
    title: 'Confirm MCP Tool Execution',
    serverName: this.serverName,
    toolName: this.serverToolName,
    onConfirm: async (outcome: ToolConfirmationOutcome) => {
      if (outcome === ToolConfirmationOutcome.ProceedAlwaysServer) {
        DiscoveredMCPTool.allowlist.add(serverAllowListKey);
      } else if (outcome === ToolConfirmationOutcome.ProceedAlwaysTool) {
        DiscoveredMCPTool.allowlist.add(toolAllowListKey);
      }
    },
  };
}
```

## 10.8 CLI 界面集成

### 10.8.1 /mcp 命令

MCP 功能通过 `/mcp` 命令暴露给用户：

```typescript
export const mcpCommand: SlashCommand = {
  name: 'mcp',
  description: 'list configured MCP servers and tools',
  subCommands: [
    {
      name: 'list',
      action: async (context, args) => {
        // 显示所有 MCP 服务器和工具
        // 支持 desc、schema、nodesc 参数
      },
    },
    {
      name: 'auth',
      action: async (context, args) => {
        // OAuth 认证流程
        const serverName = args.trim();
        await MCPOAuthProvider.authenticate(
          serverName,
          oauthConfig,
          mcpServerUrl,
        );
      },
    },
    {
      name: 'refresh',
      action: async (context) => {
        // 重新发现所有服务器的工具
        await toolRegistry.discoverMcpTools();
      },
    },
  ],
};
```

### 10.8.2 状态显示

CLI 提供了丰富的状态指示：

- 🟢 已连接并就绪
- 🔄 正在连接/启动中
- 🔴 已断开连接
- OAuth 状态内联显示
- 工具和 prompt 计数
- 服务器描述和元数据

### 10.8.3 错误处理

MCP 集成提供了友好的错误提示：

```typescript
// 网络错误
if (isNetworkError) {
  conciseError = `Cannot connect to '${mcpServerName}' - server may be down or URL incorrect`;
}

// OAuth 错误
if (status === MCPServerStatus.DISCONNECTED && needsAuthHint) {
  message += ` (type: "/mcp auth ${serverName}" to authenticate this server)`;
}

// 沙盒环境提示
if (process.env.SANDBOX) {
  conciseError += ` (check sandbox availability)`;
}
```

## 10.9 高级特性

### 10.9.1 并行发现

所有 MCP 服务器的发现过程并行执行，提高启动速度：

```typescript
const discoveryPromises = Object.entries(mcpServers).map(
  ([mcpServerName, mcpServerConfig]) =>
    connectAndDiscover(
      mcpServerName,
      mcpServerConfig,
      toolRegistry,
      promptRegistry,
      debugMode,
    ),
);
await Promise.all(discoveryPromises);
```

### 10.9.2 工具过滤

支持包含和排除特定工具：

```typescript
function isEnabled(
  funcDecl: FunctionDeclaration,
  mcpServerName: string,
  mcpServerConfig: MCPServerConfig,
): boolean {
  const { includeTools, excludeTools } = mcpServerConfig;
  
  // excludeTools 优先级高于 includeTools
  if (excludeTools && excludeTools.includes(funcDecl.name)) {
    return false;
  }
  
  return (
    !includeTools ||
    includeTools.some(
      (tool) => tool === funcDecl.name || tool.startsWith(`${funcDecl.name}(`),
    )
  );
}
```

### 10.9.3 超时控制

所有 MCP 操作都支持超时控制：

```typescript
const MCP_DEFAULT_TIMEOUT_MSEC = 10 * 60 * 1000; // 10 分钟

// 应用于连接
await mcpClient.connect(transport, {
  timeout: mcpServerConfig.timeout ?? MCP_DEFAULT_TIMEOUT_MSEC,
});

// 应用于工具调用
mcpClient.callTool = function (params, resultSchema, options) {
  return origCallTool(params, resultSchema, {
    ...options,
    timeout: mcpServerConfig.timeout ?? MCP_DEFAULT_TIMEOUT_MSEC,
  });
};
```

## 10.10 扩展开发

### 10.10.1 创建 MCP 服务器

开发者可以创建自定义 MCP 服务器：

```javascript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

const server = new McpServer({
  name: 'my-server',
  version: '1.0.0',
});

// 注册工具
server.registerTool(
  'my-tool',
  {
    title: 'My Tool',
    description: 'Does something useful',
    inputSchema: { /* ... */ },
  },
  async (params) => ({
    content: [{ type: 'text', text: 'Result' }],
  }),
);

// 注册 prompt
server.registerPrompt(
  'my-prompt',
  {
    description: 'A helpful prompt',
    arguments: [/* ... */],
  },
  async (params) => ({
    messages: [/* ... */],
  }),
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

### 10.10.2 配置集成

在 Gemini CLI 配置中添加服务器：

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["./my-mcp-server.js"],
      "description": "My custom MCP server",
      "trust": false
    }
  }
}
```

### 10.10.3 OAuth 集成

支持 OAuth 的 MCP 服务器配置：

```json
{
  "mcpServers": {
    "oauth-server": {
      "url": "https://api.example.com/mcp/sse",
      "oauth": {
        "enabled": true,
        "authorizationUrl": "https://auth.example.com/oauth/authorize",
        "tokenUrl": "https://auth.example.com/oauth/token",
        "scopes": ["read", "write"]
      }
    }
  }
}
```

## 本章小结

MCP 协议集成是 Gemini CLI 的核心扩展机制，它提供了：

1. **统一的工具集成标准**：通过 MCP 协议，任何系统都可以暴露工具给 AI 使用
2. **灵活的传输选项**：支持本地进程、网络连接等多种通信方式
3. **完整的安全模型**：包括 OAuth 认证、信任级别和用户确认机制
4. **无缝的用户体验**：工具和 prompts 自动集成到 CLI 界面
5. **强大的扩展能力**：开发者可以轻松创建自定义 MCP 服务器

通过 MCP 集成，Gemini CLI 不仅是一个 AI 对话工具，更是一个可以连接任意外部系统的智能助手平台。这种开放架构使得 Gemini CLI 能够适应各种不同的使用场景和集成需求。