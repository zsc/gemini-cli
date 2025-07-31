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

命令系统的核心特点：
- **提供者模式架构**：CommandService 基于 provider-based loader pattern，可以轻松添加新的命令源
- **并行加载优化**：所有加载器并行执行，使用 `Promise.allSettled` 确保单个失败不影响整体
- **智能冲突解决**：扩展命令自动重命名避免冲突，用户命令可覆盖内置命令

## CommandService：命令协调中心

### 职责定位

`CommandService` 是命令系统的协调中心，作为 orchestrator 负责：

1. 并行调用多个加载器
2. 聚合所有命令结果
3. 处理命令名称冲突
4. 提供统一的命令访问接口

该服务采用私有构造函数模式，强制通过异步工厂方法创建实例，确保所有命令在实例化前完成加载。

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

加载器执行顺序的重要性：
- **内置命令优先**：BuiltinCommandLoader 应该首先加载，作为基础命令集
- **文件命令其次**：FileCommandLoader 按用户→项目→扩展的顺序加载
- **容错机制**：使用 `Promise.allSettled` 确保单个加载器失败不会影响其他加载器

### 冲突解决策略

命令名称冲突的处理遵循精心设计的规则：

1. **扩展命令智能重命名**：
   ```typescript
   if (cmd.extensionName && commandMap.has(cmd.name)) {
     let renamedName = `${cmd.extensionName}.${cmd.name}`;
     let suffix = 1;
     // 持续尝试直到找到不冲突的名称
     while (commandMap.has(renamedName)) {
       renamedName = `${cmd.extensionName}.${cmd.name}${suffix}`;
       suffix++;
     }
   }
   ```

2. **非扩展命令覆盖策略**：
   - 用户命令（~/.gemini/commands/）覆盖内置命令
   - 项目命令（.gemini/commands/）覆盖用户命令
   - 基于加载器顺序的"后来者优先"原则

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
- **特殊命令**：corgi（彩蛋）、vim（编辑器模式）

加载器实现细节：
```typescript
export class BuiltinCommandLoader implements ICommandLoader {
  constructor(private config: Config | null) {}
  
  async loadCommands(_signal: AbortSignal): Promise<SlashCommand[]> {
    const allDefinitions: Array<SlashCommand | null> = [
      aboutCommand,
      authCommand,
      // ... 其他命令
      ideCommand(this.config),  // 某些命令需要注入 config
      restoreCommand(this.config),
    ];
    
    // 过滤掉 null 值（条件性加载的命令）
    return allDefinitions.filter((cmd): cmd is SlashCommand => cmd !== null);
  }
}
```

加载器特点：
- **同步返回**：虽然接口是异步的，但内置命令无需网络请求
- **依赖注入**：某些命令（如 ideCommand）需要 config 参数
- **条件性加载**：命令可以根据配置返回 null 来禁用自己

### FileCommandLoader：文件命令加载

文件命令支持用户和项目级别的自定义命令，通过 TOML 文件定义。

#### 目录扫描机制

```typescript
private getCommandDirectories(): CommandDirectory[] {
  const dirs: CommandDirectory[] = [];
  
  // 1. 用户命令
  dirs.push({ path: getUserCommandsDir() });
  
  // 2. 项目命令（覆盖用户命令）
  dirs.push({ path: getProjectCommandsDir(this.projectRoot) });
  
  // 3. 扩展命令（最后处理以检测所有冲突）
  if (this.config) {
    const activeExtensions = this.config
      .getExtensions()
      .filter((ext) => ext.isActive)
      .sort((a, b) => a.name.localeCompare(b.name)); // 确定性排序
      
    // ... 添加扩展命令目录
  }
}
```

#### TOML 格式定义与验证

使用 Zod schema 进行严格验证：
```typescript
const TomlCommandDefSchema = z.object({
  prompt: z.string({
    required_error: "The 'prompt' field is required.",
    invalid_type_error: "The 'prompt' field must be a string.",
  }),
  description: z.string().optional(),
});
```

#### 高级特性

1. **参数注入**：`{{args}}` 占位符
   - 存在时使用 ShorthandArgumentProcessor
   - 不存在时使用 DefaultArgumentProcessor

2. **Shell 命令执行**：`!{command}` 语法
   - 自动检测 SHELL_INJECTION_TRIGGER
   - 添加 ShellProcessor 到处理管道

3. **嵌套目录支持**：
   ```typescript
   // 文件路径 commands/git/commit.toml
   // 映射为命令 /git:commit
   const baseCommandName = relativePath
     .split(path.sep)
     .map((segment) => segment.replaceAll(':', '_')) // 防止命名冲突
     .join(':');
   ```

4. **扩展标记**：扩展命令自动添加 `[extensionName]` 前缀到描述中

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
  altNames?: string[];        // 别名列表（如 help 的别名是 ?）
  description: string;        // 命令描述
  kind: CommandKind;          // 命令类型（BUILT_IN、FILE、MCP）
  extensionName?: string;     // 扩展名称（用于冲突解决）
  action?: CommandAction;     // 执行函数
  completion?: CompletionFn;  // 自动补全函数
  subCommands?: SlashCommand[]; // 子命令数组
}
```

命令类型枚举：
```typescript
enum CommandKind {
  BUILT_IN = 'built-in',  // 内置命令
  FILE = 'file',          // 文件定义的命令
  MCP = 'mcp',            // MCP 协议命令
}
```

### CommandContext 执行上下文

命令执行时可访问的完整上下文，提供了丰富的服务和 UI 操作接口：

```typescript
interface CommandContext {
  invocation?: {
    raw: string;    // 原始输入（如 "/memory add test"）
    name: string;   // 命令名（如 "memory"）
    args: string;   // 参数字符串（如 "add test"）
  };
  services: {
    config: Config | null;        // 配置服务
    settings: LoadedSettings;     // 合并后的设置
    git: GitService | undefined;  // Git 服务（项目相关）
    logger: Logger;               // 日志服务
  };
  ui: {
    addItem: Function;            // 添加历史项
    clear: Function;              // 清空界面
    refreshStatic: Function;      // 刷新静态内容
    setShowHelp: Function;        // 显示帮助
    openThemeDialog: Function;    // 打开主题对话框
    openAuthDialog: Function;     // 打开认证对话框
    // ... 更多 UI 操作
  };
  session: {
    stats: SessionStatsState;     // 会话统计
    sessionShellAllowlist: Set<string>;  // Shell 命令白名单
  };
}
```

### 命令返回类型

命令可以返回多种动作类型：
```typescript
type SlashCommandActionReturn = 
  | ScheduleToolActionReturn      // 调度工具执行
  | MessageActionReturn            // 显示消息
  | OpenDialogActionReturn         // 打开对话框
  | SubmitPromptActionReturn       // 提交提示词
  | ConfirmShellCommandsReturn     // 确认 Shell 命令
  | void;                          // 无返回（命令自行处理）
```

## 命令执行流程

### SlashCommandProcessor

命令处理器是一个 React Hook，负责解析用户输入并执行相应命令。它整合了命令加载、执行和结果处理的完整生命周期。

#### 初始化过程

```typescript
useEffect(() => {
  const load = async () => {
    const controller = new AbortController();
    
    // 创建所有加载器
    const loaders: ICommandLoader[] = [
      new McpPromptLoader(mcpClients || []),
      new BuiltinCommandLoader(config),
      new FileCommandLoader(config),
    ];
    
    // 通过 CommandService 加载所有命令
    const commandService = await CommandService.create(
      loaders,
      controller.signal,
    );
    
    setCommands(commandService.getCommands());
  };
}, [config]);
```

#### 处理步骤

1. **输入验证**：
   ```typescript
   if (!trimmed.startsWith('/') && !trimmed.startsWith('?')) {
     return false;
   }
   ```

2. **命令解析与查找**：
   ```typescript
   // 两阶段查找策略
   // 第一阶段：精确匹配主名称
   let foundCommand = currentCommands.find((cmd) => cmd.name === part);
   
   // 第二阶段：如果没有找到，检查别名
   if (!foundCommand) {
     foundCommand = currentCommands.find((cmd) =>
       cmd.altNames?.includes(part),
     );
   }
   ```

3. **子命令遍历**：支持多级命令如 `/memory add`

4. **执行命令与遥测**：
   ```typescript
   if (config) {
     const event = new SlashCommandEvent(
       resolvedCommandPath[0],
       resolvedCommandPath.length > 1
         ? resolvedCommandPath.slice(1).join(' ')
         : undefined,
     );
     logSlashCommand(config, event);
   }
   ```

5. **结果处理**：
   ```typescript
   switch (result.type) {
     case 'tool':
       return {
         type: 'schedule_tool',
         toolName: result.toolName,
         toolArgs: result.toolArgs,
       };
     case 'dialog':
       switch (result.dialog) {
         case 'help': setShowHelp(true); break;
         case 'auth': openAuthDialog(); break;
         // ... 其他对话框
       }
     // ... 其他结果类型
   }
   ```

#### 特殊处理：Shell 命令一次性白名单

当用户选择"继续执行"时，通过一次性白名单机制临时扩展权限：
```typescript
if (oneTimeShellAllowlist && oneTimeShellAllowlist.size > 0) {
  fullCommandContext.session = {
    ...fullCommandContext.session,
    sessionShellAllowlist: new Set([
      ...fullCommandContext.session.sessionShellAllowlist,
      ...oneTimeShellAllowlist,
    ]),
  };
}
```

## Prompt 处理器系统

### 处理器管道架构

文件命令通过灵活的处理器管道转换 prompt，每个处理器实现 `IPromptProcessor` 接口：

```typescript
export interface IPromptProcessor {
  process(prompt: string, context: CommandContext): Promise<string>;
}
```

处理器按顺序执行，形成转换管道：
```typescript
let processedPrompt = validDef.prompt;
for (const processor of processors) {
  processedPrompt = await processor.process(processedPrompt, context);
}
```

### ShellProcessor：Shell 命令执行

#### 核心功能

1. **命令提取**：使用正则表达式查找所有 `!{...}` 模式
   ```typescript
   private static readonly SHELL_INJECTION_REGEX = /!\{([^}]*)\}/g;
   ```

2. **权限检查**：通过 `checkCommandPermissions` 验证每个命令
   - 硬拒绝（hard denial）：配置明确禁止的命令
   - 软拒绝（soft denial）：需要用户确认的命令

3. **批量执行**：收集所有命令，检查权限后批量执行

4. **输出替换**：将命令输出替换回原始占位符位置

#### 安全机制层次

```typescript
const { allAllowed, disallowedCommands, blockReason, isHardDenial } =
  checkCommandPermissions(command, config!, sessionShellAllowlist);

if (!allAllowed) {
  if (isHardDenial) {
    // 不可恢复的安全错误
    throw new Error(
      `${this.commandName} cannot be run. ${blockReason || '...'}`
    );
  }
  // 收集需要确认的命令
  disallowedCommands.forEach((uc) => commandsToConfirm.add(uc));
}
```

### ArgumentProcessor：参数处理

两种参数处理器适应不同场景：

1. **ShorthandArgumentProcessor**：
   - 处理 `{{args}}` 占位符
   - 将整个参数字符串注入到占位符位置

2. **DefaultArgumentProcessor**：
   - 没有 `{{args}}` 时使用
   - 将参数附加到 prompt 末尾

### ConfirmationRequiredError：中断与恢复

特殊的错误类型用于实现命令执行的中断与恢复：

```typescript
export class ConfirmationRequiredError extends Error {
  constructor(
    message: string,
    public commandsToConfirm: string[],
  ) {
    super(message);
    this.name = 'ConfirmationRequiredError';
  }
}
```

处理流程：
1. 处理器发现需要确认的命令，抛出错误
2. FileCommandLoader 捕获错误，返回确认请求
3. UI 层显示确认对话框
4. 用户确认后，使用一次性白名单重新执行命令

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

内置命令需要实现完整的 SlashCommand 接口。以下是一些实际示例：

#### 简单命令示例（help）
```typescript
export const helpCommand: SlashCommand = {
  name: 'help',
  altNames: ['?'],  // 支持别名
  description: 'for help on gemini-cli',
  kind: CommandKind.BUILT_IN,
  action: (_context, _args): OpenDialogActionReturn => {
    console.debug('Opening help UI ...');
    return {
      type: 'dialog',
      dialog: 'help',
    };
  },
};
```

#### 带子命令的示例（memory）
```typescript
export const memoryCommand: SlashCommand = {
  name: 'memory',
  description: 'Commands for interacting with memory.',
  kind: CommandKind.BUILT_IN,
  subCommands: [
    {
      name: 'show',
      description: 'Show the current memory contents.',
      kind: CommandKind.BUILT_IN,
      action: async (context) => {
        const memoryContent = context.services.config?.getUserMemory() || '';
        const fileCount = context.services.config?.getGeminiMdFileCount() || 0;
        
        // 直接操作 UI
        context.ui.addItem({
          type: MessageType.INFO,
          text: `Current memory from ${fileCount} file(s)...`,
        }, Date.now());
      },
    },
    {
      name: 'add',
      description: 'Add content to the memory.',
      kind: CommandKind.BUILT_IN,
      action: (context, args) => {
        // 参数验证
        if (!args || args.trim() === '') {
          return {
            type: 'message',
            messageType: 'error',
            content: 'Usage: /memory add <text to remember>',
          };
        }
        
        // 返回工具调用
        return {
          type: 'tool',
          toolName: 'save_memory',
          toolArgs: { fact: args.trim() },
        };
      },
    },
  ],
};
```

#### 条件性命令示例（ide）
```typescript
export const ideCommand = (config: Config | null): SlashCommand | null => {
  // 根据配置决定是否启用命令
  if (!config || !config.getIdeIntegration()) {
    return null;
  }
  
  return {
    name: 'ide',
    description: 'Open files in your IDE',
    kind: CommandKind.BUILT_IN,
    action: async (context, args) => {
      // 命令实现
    },
  };
};

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

系统启动时的命令加载顺序对冲突解决至关重要：

```typescript
const loaders: ICommandLoader[] = [
  new McpPromptLoader(mcpClients || []),    // MCP 命令最先
  new BuiltinCommandLoader(config),         // 内置命令其次
  new FileCommandLoader(config),            // 文件命令最后
];
```

这个顺序确保：
- MCP 命令作为外部集成，有最高优先级
- 内置命令提供核心功能
- 文件命令可以覆盖内置命令（用户定制）

### 自动补全支持

命令可以提供 `completion` 函数支持参数补全：

```typescript
completion: async (context, partialArg) => {
  // 基于上下文返回补全选项
  const availableOptions = await fetchOptions(context);
  return availableOptions.filter(opt => 
    opt.startsWith(partialArg)
  );
}
```

补全函数特性：
- 异步执行，可以进行网络请求
- 接收完整的命令上下文
- 支持动态生成选项

## 性能优化

### 并行加载策略

```typescript
const results = await Promise.allSettled(
  loaders.map((loader) => loader.loadCommands(signal)),
);

// 即使某个加载器失败，也继续处理其他结果
for (const result of results) {
  if (result.status === 'fulfilled') {
    allCommands.push(...result.value);
  } else {
    console.debug('A command loader failed:', result.reason);
  }
}
```

### 服务延迟初始化

避免阻塞启动的优化模式：

```typescript
const logger = useMemo(() => {
  const l = new Logger(config?.getSessionId() || '');
  // initialize() 是异步的，但构造函数同步返回
  // 使用时 await logger.initialize()
  return l;
}, [config]);

const gitService = useMemo(() => {
  if (!config?.getProjectRoot()) {
    return;
  }
  return new GitService(config.getProjectRoot());
}, [config]);
```

### 命令查找优化

当前实现使用两阶段线性查找，未来可优化为预计算的查找表：
```typescript
// TODO: 可以在 CommandService 中预计算查找表
// 将所有名称和别名映射到命令，避免运行时遍历
```

## 错误处理与容错

### 加载阶段容错

三层防护确保稳定性：

1. **加载器级别**：每个加载器内部捕获错误
   ```typescript
   try {
     const files = await glob('**/*.toml', options);
     // ...
   } catch (error) {
     if (error.code !== 'ENOENT') {
       console.error(`Error loading commands:`, error);
     }
   }
   ```

2. **文件级别**：单个文件解析失败不影响其他
   ```typescript
   const commands = (await Promise.all(commandPromises))
     .filter((cmd): cmd is SlashCommand => cmd !== null);
   ```

3. **服务级别**：Promise.allSettled 确保容错

### 执行阶段错误处理

```typescript
try {
  const result = await commandToExecute.action(fullCommandContext, args);
  // 处理结果
} catch (e) {
  if (e instanceof ConfirmationRequiredError) {
    // 特殊错误类型，触发确认流程
    return {
      type: 'confirm_shell_commands',
      commandsToConfirm: e.commandsToConfirm,
    };
  }
  // 其他错误显示给用户
  addItem({
    type: MessageType.ERROR,
    text: `Command error: ${e.message}`,
  }, Date.now());
}
```

### 用户友好的错误消息

- TOML 解析错误：显示具体的验证错误
- 命令未找到：建议相似命令
- 权限错误：清晰说明原因和解决方法

## 本章小结

Gemini CLI 的命令系统通过精心设计的分层架构实现了高度的灵活性和可扩展性。CommandService 作为中央协调器，统一管理来自不同源的命令；加载器模式允许轻松添加新的命令源；统一的 SlashCommand 接口确保了命令行为的一致性；而 Prompt 处理器系统则为高级命令功能提供了强大支持。这种设计不仅满足了当前的功能需求，也为未来的扩展预留了充分的空间。