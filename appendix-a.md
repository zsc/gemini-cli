# 附录 A：关键数据结构参考

本附录汇总了 Gemini CLI 中的核心数据结构和接口定义，为开发者提供快速查阅参考。

## 目录

1. [工具系统接口](#工具系统接口)
2. [API 集成类型](#api-集成类型)
3. [配置系统类型](#配置系统类型)
4. [CLI 界面类型](#cli-界面类型)
5. [命令系统类型](#命令系统类型)
6. [主题系统类型](#主题系统类型)
7. [认证和 MCP 类型](#认证和-mcp-类型)
8. [遥测系统类型](#遥测系统类型)

## 工具系统接口

工具系统是 Gemini CLI 的核心组件，定义了 AI 如何与外部系统交互。所有工具都实现了统一的接口规范。

### Tool<TParams, TResult> 接口

```typescript
interface Tool<TParams = unknown, TResult extends ToolResult = ToolResult> {
  // 工具的内部名称（用于 API 调用）
  name: string;
  
  // 用户友好的显示名称
  displayName: string;
  
  // 工具功能描述
  description: string;
  
  // 在 ACP 交互时显示的图标
  icon: Icon;
  
  // 来自 @google/genai 的函数声明模式
  schema: FunctionDeclaration;
  
  // 工具输出是否应渲染为 Markdown
  isOutputMarkdown: boolean;
  
  // 工具是否支持实时（流式）输出
  canUpdateOutput: boolean;
  
  // 参数验证
  validateToolParams(params: TParams): string | null;
  
  // 获取工具操作的预执行描述
  getDescription(params: TParams): string;
  
  // 确定工具将影响的文件系统路径
  toolLocations(params: TParams): ToolLocation[];
  
  // 确定执行前是否需要用户确认
  shouldConfirmExecute(
    params: TParams,
    abortSignal: AbortSignal
  ): Promise<ToolCallConfirmationDetails | false>;
  
  // 执行工具
  execute(
    params: TParams,
    signal: AbortSignal,
    updateOutput?: (output: string) => void
  ): Promise<TResult>;
}
```

### ToolResult 接口

工具执行结果的标准格式：

```typescript
interface ToolResult {
  // 工具动作和结果的简短摘要
  // 例如："Read 5 files"、"Wrote 256 bytes to foo.txt"
  summary?: string;
  
  // 包含在 LLM 历史中的内容
  // 应表示工具执行的实际结果
  llmContent: PartListUnion;
  
  // 用于用户显示的 Markdown 字符串
  // 提供结果的用户友好摘要或可视化
  returnDisplay: ToolResultDisplay;
}

type ToolResultDisplay = string | FileDiff;

interface FileDiff {
  fileDiff: string;
  fileName: string;
  originalContent: string | null;
  newContent: string;
}
```

### ToolCallConfirmationDetails 类型

工具执行确认的详细信息，支持多种确认类型：

```typescript
// 文件编辑确认
interface ToolEditConfirmationDetails {
  type: 'edit';
  title: string;
  onConfirm: (
    outcome: ToolConfirmationOutcome,
    payload?: ToolConfirmationPayload
  ) => Promise<void>;
  fileName: string;
  fileDiff: string;
  originalContent: string | null;
  newContent: string;
  isModifying?: boolean;
}

// Shell 命令执行确认
interface ToolExecuteConfirmationDetails {
  type: 'exec';
  title: string;
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
  command: string;
  rootCommand: string;
}

// MCP 工具确认
interface ToolMcpConfirmationDetails {
  type: 'mcp';
  title: string;
  serverName: string;
  toolName: string;
  toolDisplayName: string;
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
}

// 信息确认
interface ToolInfoConfirmationDetails {
  type: 'info';
  title: string;
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
  prompt: string;
  urls?: string[];
}

type ToolCallConfirmationDetails =
  | ToolEditConfirmationDetails
  | ToolExecuteConfirmationDetails
  | ToolMcpConfirmationDetails
  | ToolInfoConfirmationDetails;
```

### ToolConfirmationOutcome 枚举

用户对工具执行请求的响应选项：

```typescript
enum ToolConfirmationOutcome {
  ProceedOnce = 'proceed_once',           // 仅本次允许
  ProceedAlways = 'proceed_always',       // 本会话始终允许
  ProceedAlwaysServer = 'proceed_always_server', // 对该服务器始终允许
  ProceedAlwaysTool = 'proceed_always_tool',     // 对该工具始终允许
  ModifyWithEditor = 'modify_with_editor',        // 使用编辑器修改
  Cancel = 'cancel'                               // 取消执行
}
```

### Icon 枚举

工具图标类型：

```typescript
enum Icon {
  FileSearch = 'fileSearch',
  Folder = 'folder',
  Globe = 'globe',
  Hammer = 'hammer',
  LightBulb = 'lightBulb',
  Pencil = 'pencil',
  Regex = 'regex',
  Terminal = 'terminal'
}
```

## API 集成类型

这些类型定义了与 Gemini API 的交互协议，包括流式响应处理和会话管理。

### GeminiEventType 枚举

定义了流式响应中的所有事件类型：

```typescript
enum GeminiEventType {
  Content = 'content',                          // 文本内容
  ToolCallRequest = 'tool_call_request',        // 工具调用请求
  ToolCallResponse = 'tool_call_response',      // 工具调用响应
  ToolCallConfirmation = 'tool_call_confirmation', // 工具确认
  UserCancelled = 'user_cancelled',             // 用户取消
  Error = 'error',                              // 错误
  ChatCompressed = 'chat_compressed',           // 对话压缩
  Thought = 'thought',                          // 模型思考过程
  MaxSessionTurns = 'max_session_turns',        // 达到会话轮次限制
  Finished = 'finished',                        // 完成
  LoopDetected = 'loop_detected'                // 检测到循环
}
```

### ServerGeminiStreamEvent 类型

流式事件的联合类型，每种事件都有特定的结构：

```typescript
// 文本内容事件
type ServerGeminiContentEvent = {
  type: GeminiEventType.Content;
  value: string;
};

// 模型思考事件
type ServerGeminiThoughtEvent = {
  type: GeminiEventType.Thought;
  value: {
    subject: string;
    description: string;
  };
};

// 工具调用请求事件
type ServerGeminiToolCallRequestEvent = {
  type: GeminiEventType.ToolCallRequest;
  value: {
    callId: string;
    name: string;
    args: Record<string, unknown>;
    isClientInitiated: boolean;
    prompt_id: string;
  };
};

// 工具调用响应事件
type ServerGeminiToolCallResponseEvent = {
  type: GeminiEventType.ToolCallResponse;
  value: {
    callId: string;
    responseParts: PartListUnion;
    resultDisplay: ToolResultDisplay | undefined;
    error: Error | undefined;
  };
};

// 错误事件
type ServerGeminiErrorEvent = {
  type: GeminiEventType.Error;
  value: {
    error: {
      message: string;
      status?: number;
    };
  };
};

// 对话压缩事件
type ServerGeminiChatCompressedEvent = {
  type: GeminiEventType.ChatCompressed;
  value: {
    originalTokenCount: number;
    newTokenCount: number;
  } | null;
};

// 所有流式事件的联合类型
type ServerGeminiStreamEvent =
  | ServerGeminiContentEvent
  | ServerGeminiToolCallRequestEvent
  | ServerGeminiToolCallResponseEvent
  | ServerGeminiToolCallConfirmationEvent
  | ServerGeminiUserCancelledEvent
  | ServerGeminiErrorEvent
  | ServerGeminiChatCompressedEvent
  | ServerGeminiThoughtEvent
  | ServerGeminiMaxSessionTurnsEvent
  | ServerGeminiFinishedEvent
  | ServerGeminiLoopDetectedEvent;
```

### Turn 类

管理 Agent 循环中的单个轮次：

```typescript
class Turn {
  readonly pendingToolCalls: ToolCallRequestInfo[];
  
  constructor(
    private readonly chat: GeminiChat,
    private readonly prompt_id: string
  );
  
  // 运行单个轮次，生成流式事件
  async *run(
    req: PartListUnion,
    signal: AbortSignal
  ): AsyncGenerator<ServerGeminiStreamEvent>;
}
```

### GeminiCodeRequest 类型

表示发送到 Gemini API 的请求：

```typescript
// 目前是 PartListUnion 的别名，作为主要内容
// 未来可扩展以包含其他请求参数
type GeminiCodeRequest = PartListUnion;
```

## 配置系统类型

配置系统管理 Gemini CLI 的所有运行时设置，支持多层配置源和动态加载。

### Config 类

主配置类，管理所有设置：

```typescript
class Config {
  private readonly sessionId: string;
  private readonly model: string;
  private readonly embeddingModel: string;
  private readonly sandbox: SandboxConfig | undefined;
  private readonly targetDir: string;
  private readonly debugMode: boolean;
  private readonly approvalMode: ApprovalMode;
  private readonly telemetrySettings: TelemetrySettings;
  private readonly mcpServers?: Record<string, MCPServerConfig>;
  
  constructor(params: ConfigParameters);
  
  // 主要方法
  async init(): Promise<void>;
  getModel(): string;
  getContentGeneratorConfig(): ContentGeneratorConfig;
  getToolRegistry(): ToolRegistry;
  getMcpServers(): Record<string, MCPServerConfig> | undefined;
  // ... 更多 getter 方法
}
```

### ConfigParameters 接口

Config 构造函数的参数接口：

```typescript
interface ConfigParameters {
  sessionId: string;
  embeddingModel?: string;
  sandbox?: SandboxConfig;
  targetDir: string;
  debugMode: boolean;
  question?: string;
  fullContext?: boolean;
  coreTools?: string[];
  excludeTools?: string[];
  toolDiscoveryCommand?: string;
  toolCallCommand?: string;
  mcpServerCommand?: string;
  mcpServers?: Record<string, MCPServerConfig>;
  userMemory?: string;
  geminiMdFileCount?: number;
  approvalMode?: ApprovalMode;
  showMemoryUsage?: boolean;
  contextFileName?: string | string[];
  accessibility?: AccessibilitySettings;
  telemetry?: TelemetrySettings;
  usageStatisticsEnabled?: boolean;
  fileFiltering?: {
    respectGitIgnore?: boolean;
    respectGeminiIgnore?: boolean;
    enableRecursiveFileSearch?: boolean;
  };
  checkpointing?: boolean;
  proxy?: string;
  cwd: string;
  fileDiscoveryService?: FileDiscoveryService;
  bugCommand?: BugCommandSettings;
  model: string;
  extensionContextFilePaths?: string[];
  maxSessionTurns?: number;
  experimentalAcp?: boolean;
  listExtensions?: boolean;
  extensions?: GeminiCLIExtension[];
  blockedMcpServers?: Array<{ name: string; extensionName: string }>;
  noBrowser?: boolean;
  summarizeToolOutput?: Record<string, SummarizeToolOutputSettings>;
  ideMode?: boolean;
  ideClient?: IdeClient;
}
```

### MCPServerConfig 类

MCP（Model Context Protocol）服务器配置：

```typescript
class MCPServerConfig {
  constructor(
    // stdio 传输配置
    readonly command?: string,
    readonly args?: string[],
    readonly env?: Record<string, string>,
    readonly cwd?: string,
    
    // sse 传输配置
    readonly url?: string,
    
    // streamable http 传输配置
    readonly httpUrl?: string,
    readonly headers?: Record<string, string>,
    
    // websocket 传输配置
    readonly tcp?: string,
    
    // 通用配置
    readonly timeout?: number,
    readonly trust?: boolean,
    
    // 元数据
    readonly description?: string,
    readonly includeTools?: string[],
    readonly excludeTools?: string[],
    readonly extensionName?: string,
    
    // OAuth 配置
    readonly oauth?: MCPOAuthConfig,
    readonly authProviderType?: AuthProviderType
  );
}
```

### ApprovalMode 枚举

工具执行审批模式：

```typescript
enum ApprovalMode {
  DEFAULT = 'default',      // 标准确认流程
  AUTO_EDIT = 'autoEdit',   // 自动批准编辑操作
  YOLO = 'yolo'            // 自动批准所有操作（谨慎使用）
}
```

### 其他配置类型

```typescript
// 沙箱配置
interface SandboxConfig {
  command: 'docker' | 'podman' | 'sandbox-exec';
  image: string;
}

// 无障碍设置
interface AccessibilitySettings {
  disableLoadingPhrases?: boolean;
}

// 遥测设置
interface TelemetrySettings {
  enabled?: boolean;
  target?: TelemetryTarget;
  otlpEndpoint?: string;
  logPrompts?: boolean;
  outfile?: string;
}

// 文件过滤选项
interface FileFilteringOptions {
  respectGitIgnore: boolean;
  respectGeminiIgnore: boolean;
}

// 扩展信息
interface GeminiCLIExtension {
  name: string;
  version: string;
  isActive: boolean;
  path: string;
}
```

## CLI 界面类型

这些类型定义了 CLI 用户界面的状态管理和显示逻辑。

### StreamingState 枚举

UI 的流式响应状态：

```typescript
enum StreamingState {
  Idle = 'idle',                                // 空闲状态
  Responding = 'responding',                    // 正在响应
  WaitingForConfirmation = 'waiting_for_confirmation'  // 等待用户确认
}
```

### HistoryItem 类型

对话历史记录项的联合类型，支持多种消息类型：

```typescript
// 基础接口
interface HistoryItemBase {
  text?: string;  // 用户/AI/信息/错误消息的文本内容
}

// 用户消息
type HistoryItemUser = HistoryItemBase & {
  type: 'user';
  text: string;
};

// AI 响应
type HistoryItemGemini = HistoryItemBase & {
  type: 'gemini';
  text: string;
};

// AI 内容（流式）
type HistoryItemGeminiContent = HistoryItemBase & {
  type: 'gemini_content';
  text: string;
};

// 系统信息
type HistoryItemInfo = HistoryItemBase & {
  type: 'info';
  text: string;
};

// 错误消息
type HistoryItemError = HistoryItemBase & {
  type: 'error';
  text: string;
};

// 关于信息
type HistoryItemAbout = HistoryItemBase & {
  type: 'about';
  cliVersion: string;
  osVersion: string;
  sandboxEnv: string;
  modelVersion: string;
  selectedAuthType: string;
  gcpProject: string;
};

// 会话统计
type HistoryItemStats = HistoryItemBase & {
  type: 'stats';
  duration: string;
};

// 模型使用统计
type HistoryItemModelStats = HistoryItemBase & {
  type: 'model_stats';
};

// 工具使用统计
type HistoryItemToolStats = HistoryItemBase & {
  type: 'tool_stats';
};

// 退出消息
type HistoryItemQuit = HistoryItemBase & {
  type: 'quit';
  duration: string;
};

// 工具组（批量显示）
type HistoryItemToolGroup = HistoryItemBase & {
  type: 'tool_group';
  tools: IndividualToolCallDisplay[];
};

// Shell 命令
type HistoryItemUserShell = HistoryItemBase & {
  type: 'user_shell';
  text: string;
};

// 压缩事件
type HistoryItemCompression = HistoryItemBase & {
  type: 'compression';
  compression: CompressionProps;
};

// 完整的历史项类型
type HistoryItem = HistoryItemWithoutId & { id: number };
```

### ToolCallStatus 枚举

工具调用的执行状态：

```typescript
enum ToolCallStatus {
  Pending = 'Pending',        // 等待执行
  Canceled = 'Canceled',      // 已取消
  Confirming = 'Confirming',  // 等待确认
  Executing = 'Executing',    // 正在执行
  Success = 'Success',        // 执行成功
  Error = 'Error'            // 执行失败
}
```

### 工具显示相关类型

```typescript
// 单个工具调用的显示信息
interface IndividualToolCallDisplay {
  callId: string;
  name: string;
  description: string;
  resultDisplay: ToolResultDisplay | undefined;
  status: ToolCallStatus;
  confirmationDetails: ToolCallConfirmationDetails | undefined;
  renderOutputAsMarkdown?: boolean;
}

// 压缩属性
interface CompressionProps {
  isPending: boolean;
  originalTokenCount: number | null;
  newTokenCount: number | null;
}

// 消息类型枚举（内部使用）
enum MessageType {
  INFO = 'info',
  ERROR = 'error',
  USER = 'user',
  ABOUT = 'about',
  STATS = 'stats',
  MODEL_STATS = 'model_stats',
  TOOL_STATS = 'tool_stats',
  QUIT = 'quit',
  GEMINI = 'gemini',
  COMPRESSION = 'compression'
}
```

## 命令系统类型

命令系统提供了可扩展的斜杠命令框架，支持内置命令、文件命令和 MCP 提示。

### SlashCommand 接口

定义命令的标准契约：

```typescript
interface SlashCommand {
  // 命令的主要名称
  name: string;
  
  // 可选的别名
  altNames?: string[];
  
  // 命令描述
  description: string;
  
  // 命令类型
  kind: CommandKind;
  
  // 扩展命令的可选元数据
  extensionName?: string;
  
  // 执行命令的动作函数
  // 对于只包含子命令的父命令，此项可选
  action?: (
    context: CommandContext,
    args: string
  ) => void | SlashCommandActionReturn | Promise<void | SlashCommandActionReturn>;
  
  // 提供参数补全
  // 例如：为 `/chat resume <tag>` 补全标签
  completion?: (
    context: CommandContext,
    partialArg: string
  ) => Promise<string[]>;
  
  // 子命令列表
  subCommands?: SlashCommand[];
}

// 命令类型枚举
enum CommandKind {
  BUILT_IN = 'built-in',      // 内置命令
  FILE = 'file',              // 文件命令
  MCP_PROMPT = 'mcp-prompt'   // MCP 提示
}
```

### CommandContext 接口

传递给命令动作的上下文对象：

```typescript
interface CommandContext {
  // 调用属性
  invocation?: {
    raw: string;    // 原始的、未修剪的用户输入
    name: string;   // 匹配的命令主名称
    args: string;   // 命令名称后的参数字符串
  };
  
  // 核心服务和配置
  services: {
    config: Config | null;
    settings: LoadedSettings;
    git: GitService | undefined;
    logger: Logger;
  };
  
  // UI 状态和历史管理
  ui: {
    // 添加新项到历史显示
    addItem: UseHistoryManagerReturn['addItem'];
    
    // 清除所有历史项和控制台屏幕
    clear: () => void;
    
    // 设置调试模式下应用程序页脚中的临时调试消息
    setDebugMessage: (message: string) => void;
    
    // 当前待处理的历史项（如果有）
    pendingItem: HistoryItemWithoutId | null;
    
    // 设置待处理项
    setPendingItem: (item: HistoryItemWithoutId | null) => void;
    
    // 加载新的历史项集合，替换当前历史
    loadHistory: UseHistoryManagerReturn['loadHistory'];
    
    // 切换特殊显示模式
    toggleCorgiMode: () => void;
    toggleVimEnabled: () => Promise<boolean>;
  };
  
  // 会话特定数据
  session: {
    stats: SessionStatsState;
    // 用户在此会话中批准的 shell 命令的临时列表
    sessionShellAllowlist: Set<string>;
  };
}
```

### SlashCommandActionReturn 类型

命令动作的返回类型联合：

```typescript
// 调度工具调用
interface ToolActionReturn {
  type: 'tool';
  toolName: string;
  toolArgs: Record<string, unknown>;
}

// 退出应用
interface QuitActionReturn {
  type: 'quit';
  messages: HistoryItem[];
}

// 显示消息
interface MessageActionReturn {
  type: 'message';
  messageType: 'info' | 'error';
  content: string;
}

// 打开对话框
interface OpenDialogActionReturn {
  type: 'dialog';
  dialog: 'help' | 'auth' | 'theme' | 'editor' | 'privacy';
}

// 加载历史记录
interface LoadHistoryActionReturn {
  type: 'load_history';
  history: HistoryItemWithoutId[];
  clientHistory: Content[];  // 生成客户端的历史
}

// 提交提示词
interface SubmitPromptActionReturn {
  type: 'submit_prompt';
  content: string;
}

// 确认 Shell 命令
interface ConfirmShellCommandsActionReturn {
  type: 'confirm_shell_commands';
  commandsToConfirm: string[];     // 需要用户确认的 shell 命令列表
  originalInvocation: {            // 确认后重新运行的原始调用上下文
    raw: string;
  };
}

// 所有返回类型的联合
type SlashCommandActionReturn =
  | ToolActionReturn
  | MessageActionReturn
  | QuitActionReturn
  | OpenDialogActionReturn
  | LoadHistoryActionReturn
  | SubmitPromptActionReturn
  | ConfirmShellCommandsActionReturn;
```

### ICommandLoader 接口

命令加载器的契约：

```typescript
interface ICommandLoader {
  // 从加载器的源发现并返回斜杠命令列表
  loadCommands(signal: AbortSignal): Promise<SlashCommand[]>;
}
```

## 主题系统类型

主题系统管理 CLI 的视觉外观，包括语法高亮和颜色方案。

### ColorsTheme 接口

定义主题的颜色配置：

```typescript
interface ColorsTheme {
  type: ThemeType;          // 主题类型
  Background: string;       // 背景色
  Foreground: string;       // 前景色
  LightBlue: string;        // 浅蓝色
  AccentBlue: string;       // 强调蓝色
  AccentPurple: string;     // 强调紫色
  AccentCyan: string;       // 强调青色
  AccentGreen: string;      // 强调绿色
  AccentYellow: string;     // 强调黄色
  AccentRed: string;        // 强调红色
  DiffAdded: string;        // 差异添加行颜色
  DiffRemoved: string;      // 差异删除行颜色
  Comment: string;          // 注释颜色
  Gray: string;             // 灰色
  GradientColors?: string[]; // 渐变色数组
}

// 主题类型
type ThemeType = 'light' | 'dark' | 'ansi' | 'custom';

// 自定义主题扩展接口
interface CustomTheme extends ColorsTheme {
  type: 'custom';
  name: string;
}
```

### Theme 类

管理语法高亮主题：

```typescript
class Theme {
  // 无特定高亮规则时的默认前景色
  readonly defaultColor: string;
  
  // 主题名称
  readonly name: string;
  
  // 主题类型
  readonly type: ThemeType;
  
  // 主题颜色配置
  readonly colors: ColorsTheme;
  
  constructor(
    name: string,
    type: ThemeType,
    rawMappings: Record<string, CSSProperties>,
    colors: ColorsTheme
  );
  
  // 获取给定 highlight.js 类名的 Ink 兼容颜色字符串
  getInkColor(hljsClass: string): string | undefined;
}
```

## 认证和 MCP 类型

这些类型支持 OAuth 认证和 Model Context Protocol 集成。

### MCPOAuthConfig 接口

MCP 服务器的 OAuth 配置：

```typescript
interface MCPOAuthConfig {
  enabled?: boolean;         // 是否启用 OAuth
  clientId?: string;         // OAuth 客户端 ID
  clientSecret?: string;     // OAuth 客户端密钥
  authorizationUrl?: string; // 授权端点 URL
  tokenUrl?: string;         // 令牌端点 URL
  scopes?: string[];         // 请求的权限范围
  redirectUri?: string;      // 重定向 URI
  tokenParamName?: string;   // SSE 连接的令牌参数名
}
```

### OAuth 响应类型

```typescript
// OAuth 授权响应
interface OAuthAuthorizationResponse {
  code: string;
  state: string;
}

// OAuth 令牌响应
interface OAuthTokenResponse {
  access_token: string;
  token_type: string;
  expires_in?: number;
  refresh_token?: string;
  scope?: string;
}

// 动态客户端注册响应
interface OAuthClientRegistrationResponse {
  client_id: string;
  client_secret?: string;
  client_id_issued_at?: number;
  client_secret_expires_at?: number;
  redirect_uris: string[];
  grant_types: string[];
  response_types: string[];
  token_endpoint_auth_method: string;
  code_challenge_method?: string[];
  scope?: string;
}
```

### 认证类型

```typescript
// 认证提供者类型
enum AuthProviderType {
  DYNAMIC_DISCOVERY = 'dynamic_discovery',      // 动态发现
  GOOGLE_CREDENTIALS = 'google_credentials'     // Google 凭据
}

// 认证类型（来自 core/contentGenerator.ts）
enum AuthType {
  LOGIN_WITH_GOOGLE = 'login_with_google',
  CLOUD_SHELL = 'cloud_shell',
  USE_GEMINI = 'use_gemini',
  USE_VERTEX_AI = 'use_vertex_ai'
}
```

## 遥测系统类型

遥测系统收集使用数据和性能指标。

### TelemetryEvent 类型

所有遥测事件的联合类型：

```typescript
type TelemetryEvent =
  | StartSessionEvent      // 会话开始
  | EndSessionEvent        // 会话结束
  | UserPromptEvent        // 用户提示
  | ToolCallEvent          // 工具调用
  | ApiRequestEvent        // API 请求
  | ApiErrorEvent          // API 错误
  | ApiResponseEvent       // API 响应
  | FlashFallbackEvent     // 模型回退
  | LoopDetectedEvent      // 循环检测
  | FlashDecidedToContinueEvent  // Flash 决定继续
  | SlashCommandEvent;     // 斜杠命令使用
```

### 主要遥测事件类

```typescript
// 会话开始事件
class StartSessionEvent {
  'event.name': 'cli_config';
  'event.timestamp': string;  // ISO 8601
  model: string;
  embedding_model: string;
  sandbox_enabled: boolean;
  core_tools_enabled: string;
  approval_mode: string;
  api_key_enabled: boolean;
  vertex_ai_enabled: boolean;
  debug_enabled: boolean;
  mcp_servers: string;
  telemetry_enabled: boolean;
  telemetry_log_user_prompts_enabled: boolean;
  file_filtering_respect_git_ignore: boolean;
}

// 工具调用事件
class ToolCallEvent {
  'event.name': 'tool_call';
  'event.timestamp': string;
  function_name: string;
  function_args: Record<string, unknown>;
  duration_ms: number;
  success: boolean;
  decision?: ToolCallDecision;
  error?: string;
  error_type?: string;
  prompt_id: string;
}

// API 响应事件
class ApiResponseEvent {
  'event.name': 'api_response';
  'event.timestamp': string;
  model: string;
  status_code?: number | string;
  duration_ms: number;
  error?: string;
  input_token_count: number;
  output_token_count: number;
  cached_content_token_count: number;
  thoughts_token_count: number;
  tool_token_count: number;
  total_token_count: number;
  response_text?: string;
  prompt_id: string;
  auth_type?: string;
}
```

### ToolCallDecision 枚举

工具调用决策类型：

```typescript
enum ToolCallDecision {
  ACCEPT = 'accept',    // 接受执行
  REJECT = 'reject',    // 拒绝执行
  MODIFY = 'modify'     // 修改后执行
}
```

### LoopType 枚举

检测到的循环类型：

```typescript
enum LoopType {
  CONSECUTIVE_IDENTICAL_TOOL_CALLS = 'consecutive_identical_tool_calls',  // 连续相同工具调用
  CHANTING_IDENTICAL_SENTENCES = 'chanting_identical_sentences',          // 重复相同句子
  LLM_DETECTED_LOOP = 'llm_detected_loop'                               // LLM 检测到循环
}
```

## 本章小结

本附录提供了 Gemini CLI 核心数据结构的完整参考。这些类型定义构成了系统的基础架构：

- **工具系统接口**定义了 AI 与外部系统交互的标准契约
- **API 集成类型**管理与 Gemini API 的流式通信协议
- **配置系统类型**提供了灵活的多层配置管理
- **CLI 界面类型**支持丰富的用户交互体验
- **命令系统类型**实现了可扩展的命令框架
- **主题系统类型**管理视觉外观和语法高亮
- **认证和 MCP 类型**支持多种认证方式和协议集成
- **遥测系统类型**收集使用数据以改进产品

理解这些数据结构对于：
- 开发自定义工具和扩展
- 集成 MCP 服务器
- 创建自定义主题
- 调试和故障排查
- 深入了解系统架构

都具有重要意义。开发者应将本附录作为快速参考，结合主文档了解这些类型在实际系统中的应用。