# 第 8 章：命令系统设计

Gemini CLI 的命令系统采用了灵活的插件式架构，支持内置命令、自定义文件命令和 MCP 协议命令。本章将深入分析命令系统的设计理念、加载机制、执行流程以及扩展方式。

## 命令系统架构概览

### 核心设计理念

命令系统基于以下设计原则：

1. **可扩展性**：通过加载器模式支持多种命令源
2. **解耦性**：命令定义与执行环境分离
3. **一致性**：统一的命令接口和执行流程
4. **安全性**：Shell 命令执行的权限控制

### 组件关系

```
CommandService (协调器)
    ├── BuiltinCommandLoader (内置命令)
    ├── FileCommandLoader (文件命令)
    └── McpPromptLoader (MCP 命令)
           ↓
    SlashCommand[] (统一命令列表)
           ↓
    SlashCommandProcessor (执行器)
```

## CommandService：命令协调中心

### 职责定位

`CommandService` 是命令系统的协调中心，负责：

1. 并行调用多个加载器
2. 聚合所有命令结果
3. 处理命令名称冲突
4. 提供统一的命令访问接口

### 加载流程

```typescript
static async create(
  loaders: ICommandLoader[],
  signal: AbortSignal,
): Promise<CommandService> {
  // 并行加载所有命令源
  const results = await Promise.allSettled(
    loaders.map((loader) => loader.loadCommands(signal)),
  );
  
  // 聚合并去重
  const commandMap = new Map<string, SlashCommand>();
  // ... 冲突处理逻辑
}
```

### 冲突解决策略

命令名称冲突的处理遵循以下规则：

1. **扩展命令重命名**：如果扩展命令与现有命令冲突，重命名为 `extensionName.commandName`
2. **非扩展命令覆盖**：用户命令覆盖内置命令，项目命令覆盖用户命令

## 命令加载器设计

### ICommandLoader 接口

所有加载器都实现统一的接口：

```typescript
export interface ICommandLoader {
  loadCommands(signal: AbortSignal): Promise<SlashCommand[]>;
}
```

### BuiltinCommandLoader：内置命令加载

内置命令是 Gemini CLI 的核心功能集，包括：

- **系统命令**：help、quit、clear
- **配置命令**：auth、theme、privacy、settings
- **会话命令**：chat、memory、restore
- **开发命令**：tools、stats、bug、init
- **集成命令**：ide、mcp、extensions

加载器特点：
- 同步加载，无需网络请求
- 支持依赖注入（如 config）
- 条件性加载（某些命令需要特定配置）

### FileCommandLoader：文件命令加载

文件命令支持用户和项目级别的自定义命令。

#### 目录扫描顺序

1. **用户命令**：`~/.gemini/commands/`
2. **项目命令**：`<project>/.gemini/commands/`
3. **扩展命令**：`<extension>/commands/`

#### TOML 格式定义

```toml
# example.toml
prompt = "Generate a {{args}} function"
description = "Create a new function"
```

#### 高级特性

1. **参数注入**：`{{args}}` 占位符
2. **Shell 命令执行**：`!{command}` 语法
3. **嵌套目录支持**：目录结构映射为命令路径

### McpPromptLoader：MCP 命令加载

MCP (Model Context Protocol) 命令将外部服务的 prompts 暴露为本地命令。

特性：
- 动态发现 MCP 服务器的 prompts
- 自动生成 help 子命令
- 支持命名参数和位置参数

## SlashCommand 数据结构

### 核心属性

```typescript
interface SlashCommand {
  name: string;               // 主命令名
  altNames?: string[];        // 别名列表
  description: string;        // 命令描述
  kind: CommandKind;          // 命令类型
  extensionName?: string;     // 扩展名称
  action?: CommandAction;     // 执行函数
  completion?: CompletionFn;  // 自动补全
  subCommands?: SlashCommand[]; // 子命令
}
```

### CommandContext 执行上下文

命令执行时可访问的完整上下文：

```typescript
interface CommandContext {
  invocation?: {
    raw: string;    // 原始输入
    name: string;   // 命令名
    args: string;   // 参数字符串
  };
  services: {
    config: Config | null;
    settings: LoadedSettings;
    git: GitService | undefined;
    logger: Logger;
  };
  ui: {
    addItem: Function;
    clear: Function;
    // ... 更多 UI 操作
  };
  session: {
    stats: SessionStatsState;
    sessionShellAllowlist: Set<string>;
  };
}
```

## 命令执行流程

### SlashCommandProcessor

命令处理器负责解析用户输入并执行相应命令。

#### 处理步骤

1. **输入验证**：检查是否以 `/` 开头
2. **命令解析**：分割命令路径和参数
3. **命令查找**：
   - 优先匹配主名称
   - 其次匹配别名
   - 支持子命令遍历
4. **执行命令**：调用 action 函数
5. **结果处理**：根据返回类型执行相应操作

#### 命令返回类型处理

```typescript
switch (result.type) {
  case 'tool':
    // 调度工具执行
  case 'message':
    // 显示消息
  case 'dialog':
    // 打开对话框
  case 'submit_prompt':
    // 提交到 Gemini
  case 'confirm_shell_commands':
    // 请求 Shell 权限确认
}
```

## Prompt 处理器系统

### 处理器管道

文件命令支持通过处理器管道转换 prompt：

1. **ShellProcessor**：执行 Shell 命令并注入输出
2. **ArgumentProcessor**：处理参数占位符

### Shell 命令安全机制

#### 权限检查流程

1. 检查全局允许列表
2. 检查会话允许列表
3. 检查配置黑名单
4. 触发用户确认对话框

#### ConfirmationRequiredError

当遇到未授权的 Shell 命令时，抛出特殊错误触发确认流程：

```typescript
throw new ConfirmationRequiredError(
  'Shell command confirmation required',
  Array.from(commandsToConfirm),
);
```

## 命令扩展最佳实践

### 创建自定义命令

#### 1. 简单文本命令

```toml
# ~/.gemini/commands/review.toml
prompt = "Review this code for best practices"
description = "Code review helper"
```

#### 2. 带参数的命令

```toml
# ~/.gemini/commands/generate/component.toml
prompt = "Create a React component named {{args}}"
description = "Generate React component"
```

#### 3. 集成 Shell 命令

```toml
# ~/.gemini/commands/test.toml
prompt = """
Run tests for the current project:

!{npm test}

Analyze the results and suggest improvements.
"""
description = "Run tests and analyze"
```

### 开发内置命令

内置命令需要实现完整的 SlashCommand 接口：

```typescript
export const myCommand: SlashCommand = {
  name: 'mycommand',
  description: 'My custom command',
  kind: CommandKind.BUILT_IN,
  action: async (context, args) => {
    // 访问服务
    const config = context.services.config;
    
    // 更新 UI
    context.ui.addItem({
      type: MessageType.INFO,
      text: 'Command executed'
    });
    
    // 返回动作
    return {
      type: 'submit_prompt',
      content: 'Process this request...'
    };
  }
};
```

### MCP 命令集成

通过 MCP 服务器配置自动暴露命令：

```json
{
  "mcpServers": {
    "my-server": {
      "command": "my-mcp-server",
      "args": ["--port", "3000"]
    }
  }
}
```

## 命令发现与补全

### 命令发现机制

系统启动时的命令加载顺序：

1. MCP prompts（最先）
2. 内置命令（其次）  
3. 文件命令：用户 → 项目 → 扩展

### 自动补全支持

命令可以提供 `completion` 函数支持参数补全：

```typescript
completion: async (context, partialArg) => {
  // 返回可能的补全选项
  return ['option1', 'option2'];
}
```

## 性能优化

### 并行加载

所有加载器并行执行，使用 `Promise.allSettled` 确保单个加载器失败不影响其他：

```typescript
const results = await Promise.allSettled(
  loaders.map((loader) => loader.loadCommands(signal)),
);
```

### 延迟初始化

某些重量级服务（如 Logger）采用延迟初始化：

```typescript
const logger = useMemo(() => {
  const l = new Logger(config?.getSessionId() || '');
  // 异步初始化，但同步返回实例
  return l;
}, [config]);
```

## 错误处理

### 加载阶段错误

- 单个加载器失败不影响整体
- 错误记录到调试日志
- 继续加载其他命令源

### 执行阶段错误

- 捕获命令执行异常
- 显示友好错误消息
- 保持应用稳定运行

## 本章小结

Gemini CLI 的命令系统通过精心设计的分层架构实现了高度的灵活性和可扩展性。CommandService 作为中央协调器，统一管理来自不同源的命令；加载器模式允许轻松添加新的命令源；统一的 SlashCommand 接口确保了命令行为的一致性；而 Prompt 处理器系统则为高级命令功能提供了强大支持。这种设计不仅满足了当前的功能需求，也为未来的扩展预留了充分的空间。