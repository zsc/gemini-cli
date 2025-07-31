# 第 19 章：扩展开发指南

Gemini CLI 设计了一个灵活的扩展系统，允许开发者通过多种方式扩展其功能。本章将详细介绍如何开发自定义工具、MCP 服务器、命令扩展以及 IDE 集成，帮助你构建强大的 Gemini CLI 扩展。

## 扩展系统概览

### 扩展类型

Gemini CLI 支持以下几种扩展方式：

1. **工具扩展**：通过命令行工具发现或 MCP 协议添加新工具
2. **命令扩展**：使用 TOML 文件定义自定义斜杠命令
3. **上下文扩展**：通过 GEMINI.md 文件提供额外的上下文信息
4. **IDE 集成**：为特定编辑器提供深度集成

### 扩展加载机制

扩展系统的核心实现位于 `packages/cli/src/config/extension.ts`。扩展从以下位置按优先级加载：

- **工作区扩展**：`{workspaceDir}/.gemini/extensions/`
- **用户扩展**：`~/.gemini/extensions/`

当存在同名扩展时，工作区扩展会覆盖用户扩展。扩展加载过程会：
1. 扫描所有扩展目录下的子目录
2. 查找每个子目录中的 `gemini-extension.json` 配置文件
3. 验证配置文件的有效性（必须包含 name 和 version 字段）
4. 收集扩展的上下文文件
5. 根据用户配置决定扩展是否激活

每个扩展必须包含 `gemini-extension.json` 配置文件：

```json
{
  "name": "my-extension",
  "version": "1.0.0",
  "mcpServers": {
    "my-server": {
      "command": "node my-mcp-server.js",
      "args": ["--port", "3000"],
      "env": {
        "API_KEY": "${MY_API_KEY}"
      },
      "cwd": "server",
      "timeout": 30000,
      "trust": false
    }
  },
  "contextFileName": ["GEMINI.md", "API.md"],
  "excludeTools": ["tool-to-exclude"]
}
```

### 扩展激活控制

用户可以通过配置文件的 `enabledExtensions` 字段控制扩展激活：
- 空数组或未定义：激活所有扩展
- `["none"]`：禁用所有扩展
- `["ext1", "ext2"]`：只激活指定的扩展

## 自定义工具开发

### 方式一：命令行工具发现

通过配置 `toolDiscoveryCommand` 和 `toolCallCommand`，可以让 Gemini CLI 发现和调用外部工具。这种机制的实现位于 `packages/core/src/tools/tool-registry.ts`。

#### 1. 实现发现命令

发现命令应返回 FunctionDeclaration 数组的 JSON。系统会执行配置的命令，并解析标准输出：

```javascript
// discover-tools.js
const tools = [
  {
    name: "analyze_code",
    description: "Analyze code quality and suggest improvements",
    parameters: {
      type: "object",
      properties: {
        filePath: {
          type: "string",
          description: "Path to the file to analyze"
        },
        checkType: {
          type: "string",
          enum: ["quality", "security", "performance"],
          description: "Type of analysis to perform"
        }
      },
      required: ["filePath"]
    }
  }
];

console.log(JSON.stringify(tools));
```

发现命令的实现要点：
- 输出必须是有效的 JSON 格式
- 支持返回单个工具对象或工具数组
- 支持嵌套在 `tool` 或 `tools` 字段中的声明
- 标准输出大小限制为 10MB
- 支持使用 shell 引号解析命令参数

#### 2. 实现调用命令

调用命令接收工具名作为第一个参数，从 stdin 读取参数 JSON：

```javascript
// call-tool.js
const toolName = process.argv[2];
let input = '';

process.stdin.on('data', chunk => input += chunk);
process.stdin.on('end', () => {
  const params = JSON.parse(input);
  
  if (toolName === 'analyze_code') {
    try {
      // 执行分析逻辑
      const result = analyzeCode(params.filePath, params.checkType);
      // 成功时输出 JSON 结果到 stdout
      console.log(JSON.stringify(result));
      process.exit(0);
    } catch (error) {
      // 错误时输出到 stderr
      console.error(error.message);
      process.exit(1);
    }
  }
});
```

调用命令的错误处理机制：
- 退出码为 0 时，stdout 内容作为工具结果
- 非零退出码、信号终止或 stderr 有输出时，返回详细错误信息
- 错误信息包含：Stdout、Stderr、Error、Exit Code、Signal

#### 3. 配置工具发现

在 `.gemini/settings.json` 中配置：

```json
{
  "toolDiscoveryCommand": "node discover-tools.js",
  "toolCallCommand": "node call-tool.js"
}
```

发现的工具会被包装为 `DiscoveredTool` 实例，自动添加以下说明：
- 工具来源信息
- 执行命令详情
- 配置说明
- 错误处理说明

### 方式二：MCP 服务器

Model Context Protocol (MCP) 提供了更强大的工具集成方式。MCP 客户端实现位于 `packages/core/src/tools/mcp-client.ts`，支持多种传输协议和高级特性。

#### 1. 创建 MCP 服务器

```typescript
// my-mcp-server.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

const server = new Server({
  name: 'my-mcp-server',
  version: '1.0.0',
});

// 注册工具
server.setRequestHandler('tools/list', async () => {
  return {
    tools: [
      {
        name: 'database_query',
        description: 'Execute SQL queries on the project database',
        inputSchema: {
          type: 'object',
          properties: {
            query: {
              type: 'string',
              description: 'SQL query to execute'
            },
            database: {
              type: 'string',
              description: 'Database name',
              default: 'main'
            }
          },
          required: ['query']
        }
      }
    ]
  };
});

// 处理工具调用
server.setRequestHandler('tools/call', async (request) => {
  const { name, arguments: args } = request.params;
  
  if (name === 'database_query') {
    const result = await executeQuery(args.query, args.database);
    return {
      content: [
        {
          type: 'text',
          text: JSON.stringify(result, null, 2)
        }
      ]
    };
  }
});

// 启动服务器
const transport = new StdioServerTransport();
await server.connect(transport);
```

#### 2. 配置 MCP 服务器

MCP 服务器支持多种配置方式和传输协议：

##### 通过扩展配置
在扩展的 `gemini-extension.json` 中配置：

```json
{
  "name": "database-tools",
  "version": "1.0.0",
  "mcpServers": {
    "database": {
      "command": "node dist/my-mcp-server.js",
      "args": ["--verbose"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      },
      "cwd": "./server",
      "trust": false,
      "timeout": 30000,
      "includeTools": ["database_query", "schema_info"],
      "excludeTools": ["dangerous_operation"]
    }
  }
}
```

##### 通过设置文件配置
在 `.gemini/settings.json` 中直接配置：

```json
{
  "mcpServers": {
    "http-server": {
      "httpUrl": "https://api.example.com/mcp",
      "headers": {
        "API-Key": "${API_KEY}"
      },
      "timeout": 60000
    },
    "sse-server": {
      "url": "https://sse.example.com/events",
      "oauth": {
        "enabled": true,
        "authorizationUrl": "https://auth.example.com/authorize",
        "tokenUrl": "https://auth.example.com/token",
        "scopes": ["read", "write"]
      }
    }
  }
}
```

#### 3. 支持的传输协议

MCP 客户端支持三种传输协议：

1. **Stdio（标准输入输出）**：通过进程间通信
   - 适合本地命令行工具
   - 支持环境变量和工作目录配置
   
2. **SSE（Server-Sent Events）**：通过 HTTP 长连接
   - 适合云服务集成
   - 支持 OAuth 认证
   
3. **HTTP（Streamable HTTP）**：通过 HTTP 请求
   - 适合 REST API 风格的服务
   - 支持自定义请求头

#### 4. OAuth 认证支持

MCP 服务器支持自动 OAuth 发现和认证：

```typescript
// 服务器返回 401 时的 WWW-Authenticate 头
response.headers['www-authenticate'] = 'Bearer realm="https://auth.example.com/.well-known/oauth"';

// 自动发现 OAuth 配置并引导用户认证
// 认证令牌会自动存储并在后续请求中使用
```

#### 5. 工具过滤和超时控制

- **includeTools**：白名单模式，只包含指定工具
- **excludeTools**：黑名单模式，排除指定工具（优先级更高）
- **timeout**：工具调用超时时间（默认 10 分钟）
- **trust**：是否信任该服务器（影响确认提示）

### 方式三：可修改工具

对于需要用户编辑的工具（如文件编辑），可以实现 `ModifiableTool` 接口：

```typescript
import { ModifiableTool, ModifyContext } from '@google/gemini-cli-core';

export class CustomEditTool implements ModifiableTool<EditParams> {
  getModifyContext(abortSignal: AbortSignal): ModifyContext<EditParams> {
    return {
      getFilePath: (params) => params.filePath,
      getCurrentContent: async (params) => {
        return await fs.readFile(params.filePath, 'utf-8');
      },
      getProposedContent: async (params) => {
        return applyEdits(params);
      },
      createUpdatedParams: (oldContent, newContent, originalParams) => {
        return {
          ...originalParams,
          newContent
        };
      }
    };
  }
}
```

## 命令扩展开发

### TOML 命令格式

在 `.gemini/commands/` 目录下创建 TOML 文件定义命令：

```toml
# analyze-project.toml
prompt = """
Analyze the project structure and provide insights about:
1. Code organization
2. Dependencies
3. Potential improvements

Project root: $(pwd)
File count: $(find . -type f -name "*.ts" -o -name "*.js" | wc -l)
"""

description = "Analyze project structure and provide improvement suggestions"
```

### 高级命令功能

#### 1. 参数支持

使用 `{{args}}` 占位符接收参数：

```toml
# test-file.toml
prompt = """
Run tests for the specified file and analyze results:
{{args}}

Test output:
$(npm test -- {{args}})
"""
description = "Run tests for a specific file"
```

#### 2. Shell 命令注入

使用 `$(command)` 语法执行 shell 命令：

```toml
# git-status.toml
prompt = """
Analyze current git status and suggest next steps:

$(git status --porcelain)
$(git log --oneline -n 5)
"""
description = "Analyze git repository status"
```

#### 3. 命名空间

文件路径自动转换为命名空间：
- `commands/code/analyze.toml` → `/code:analyze`
- `commands/git/commit-helper.toml` → `/git:commit-helper`

## 上下文扩展

### GEMINI.md 文件

扩展可以提供 GEMINI.md 文件来增强 AI 的上下文理解：

```markdown
# Project Context

## Architecture Overview
This extension provides database tools for Gemini CLI.

## Key Concepts
- Connection pooling for performance
- Transaction support with automatic rollback
- Query builder for safe SQL generation

## Usage Guidelines
When using database tools:
1. Always use parameterized queries
2. Prefer read-only operations
3. Request confirmation for destructive operations
```

### 多文件支持

在 `gemini-extension.json` 中配置多个上下文文件：

```json
{
  "contextFileName": ["GEMINI.md", "API.md", "ARCHITECTURE.md"]
}
```

## IDE 集成开发

### VSCode 扩展示例

创建 VSCode 扩展与 Gemini CLI 集成：

```typescript
// extension.ts
import * as vscode from 'vscode';

export async function activate(context: vscode.ExtensionContext) {
  // 提供工作区路径
  context.environmentVariableCollection.replace(
    'GEMINI_CLI_IDE_WORKSPACE_PATH',
    vscode.workspace.rootPath || ''
  );
  
  // 注册命令
  context.subscriptions.push(
    vscode.commands.registerCommand('gemini.analyzeFile', async () => {
      const activeEditor = vscode.window.activeTextEditor;
      if (!activeEditor) return;
      
      const terminal = vscode.window.createTerminal('Gemini CLI');
      terminal.show();
      terminal.sendText(`gemini "Analyze this file: ${activeEditor.document.fileName}"`);
    })
  );
}
```

### IDE 服务器协议

实现 IDE 服务器以提供更丰富的上下文：

```typescript
// ide-server.ts
import { createServer } from 'net';

interface IDEContext {
  openFiles: string[];
  currentFile?: string;
  selectedText?: string;
}

const server = createServer((socket) => {
  socket.on('data', (data) => {
    const request = JSON.parse(data.toString());
    
    if (request.method === 'getContext') {
      const context: IDEContext = {
        openFiles: getOpenFiles(),
        currentFile: getCurrentFile(),
        selectedText: getSelectedText()
      };
      
      socket.write(JSON.stringify({
        id: request.id,
        result: context
      }));
    }
  });
});

server.listen(50505);
```

## 最佳实践

### 1. 错误处理

始终实现健壮的错误处理：

```typescript
try {
  const result = await riskyOperation();
  return { success: true, data: result };
} catch (error) {
  console.error(`Tool execution failed: ${error}`);
  return {
    success: false,
    error: error instanceof Error ? error.message : 'Unknown error'
  };
}
```

### 2. 性能优化

- 实现超时机制
- 使用流式处理大数据
- 缓存频繁访问的结果

### 3. 安全考虑

- 验证所有输入参数
- 使用参数化查询防止注入
- 实现适当的权限检查

### 4. 用户体验

- 提供清晰的工具描述
- 实现进度反馈
- 支持中断操作

## 调试和测试

### 本地测试

1. 创建测试扩展目录：
   ```bash
   mkdir -p .gemini/extensions/my-extension
   ```

2. 添加配置和实现文件

3. 运行 Gemini CLI 查看扩展是否加载：
   ```bash
   gemini /tools
   ```

### 日志和调试

启用调试模式查看详细信息：

```bash
export GEMINI_DEBUG=true
gemini
```

在代码中添加日志：

```typescript
if (process.env.GEMINI_DEBUG) {
  console.error('[MyExtension] Processing request:', request);
}
```

## 分发扩展

### 打包建议

1. 创建清晰的目录结构
2. 包含 README 和示例
3. 提供安装脚本
4. 记录依赖关系

### 示例扩展结构

```
my-extension/
├── gemini-extension.json
├── README.md
├── GEMINI.md
├── package.json
├── dist/
│   └── index.js
├── commands/
│   ├── analyze.toml
│   └── refactor.toml
└── examples/
    └── usage.md
```

## 本章小结

Gemini CLI 的扩展系统提供了多种方式来增强其功能。通过工具发现、MCP 协议、自定义命令和 IDE 集成，开发者可以为特定领域或工作流创建强大的扩展。遵循本章介绍的模式和最佳实践，你可以构建出既强大又易用的 Gemini CLI 扩展，为用户提供更好的 AI 辅助开发体验。