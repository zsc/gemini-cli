# 第 16 章：错误处理与恢复机制

Gemini CLI 作为一个复杂的命令行工具，需要处理来自多个层面的错误：网络请求、文件操作、工具执行、用户输入等。本章深入分析系统的错误处理架构，探讨如何通过分层的错误处理策略、智能重试机制和用户友好的错误提示，构建一个健壮可靠的 CLI 应用。

## 错误处理架构概览

### 分层错误处理模型

Gemini CLI 采用分层的错误处理架构，每一层负责处理特定类型的错误：

```
┌─────────────────────────────────────────┐
│          用户界面层 (UI Layer)          │
│   - ErrorMessage 组件                   │
│   - 错误信息格式化                     │
│   - 用户友好的提示                     │
└────────────────┬────────────────────────┘
                 │
┌────────────────┴────────────────────────┐
│         应用逻辑层 (App Layer)          │
│   - 会话管理错误                       │
│   - 命令处理错误                       │
│   - 认证流程错误                       │
└────────────────┬────────────────────────┘
                 │
┌────────────────┴────────────────────────┐
│          核心层 (Core Layer)            │
│   - API 通信错误                       │
│   - 工具执行错误                       │
│   - 重试与恢复逻辑                     │
└────────────────┬────────────────────────┘
                 │
┌────────────────┴────────────────────────┐
│        基础设施层 (Infra Layer)         │
│   - 网络错误                           │
│   - 文件系统错误                       │
│   - 系统资源错误                       │
└─────────────────────────────────────────┘
```

### 错误类型体系

系统在 `packages/core/src/utils/errors.ts` 中定义了清晰的错误类型层次结构，便于精确处理不同类型的错误：

```typescript
// 基础错误类型定义
export class ForbiddenError extends Error {}    // HTTP 403
export class UnauthorizedError extends Error {}  // HTTP 401
export class BadRequestError extends Error {}    // HTTP 400

// 网络错误类型（定义在 fetch.ts 中）
export class FetchError extends Error {
  constructor(message: string, public code?: string) {
    super(message);
    this.name = 'FetchError';
  }
}

// 节点错误类型判断
export function isNodeError(error: unknown): error is NodeJS.ErrnoException {
  return error instanceof Error && 'code' in error;
}
```

错误类型体系的设计考虑了多种来源的错误：
- **HTTP 状态码错误**：通过专门的错误类型映射 HTTP 状态码
- **系统错误**：通过 `isNodeError` 判断处理文件系统和网络错误
- **API 错误**：通过 `GaxiosError` 接口处理 Google API 客户端错误

## 核心错误处理机制

### 1. 错误捕获与转换

系统在 `packages/core/src/utils/errors.ts` 中提供了统一的错误处理工具函数，确保错误信息的安全提取和友好展示：

```typescript
// 安全的错误信息提取
export function getErrorMessage(error: unknown): string {
  if (error instanceof Error) {
    return error.message;
  }
  try {
    return String(error);
  } catch {
    return 'Failed to get error details';
  }
}

// API 错误转换为友好错误类型
export function toFriendlyError(error: unknown): unknown {
  if (error && typeof error === 'object' && 'response' in error) {
    const gaxiosError = error as GaxiosError;
    const data = parseResponseData(gaxiosError);
    if (data.error && data.error.message && data.error.code) {
      switch (data.error.code) {
        case 400:
          return new BadRequestError(data.error.message);
        case 401:
          return new UnauthorizedError(data.error.message);
        case 403:
          // 注释中特别提到：传递消息很重要，因为它可能解释原因
          // 比如"您使用的云项目未启用代码助手"
          return new ForbiddenError(data.error.message);
      }
    }
  }
  return error;
}

// 解析响应数据的辅助函数
function parseResponseData(error: GaxiosError): ResponseData {
  // 有时 Gaxios 不会对响应数据进行 JSON 化处理
  if (typeof error.response?.data === 'string') {
    return JSON.parse(error.response?.data) as ResponseData;
  }
  return error.response?.data as ResponseData;
}
```

错误转换机制的关键设计要点：
- **类型安全**：通过类型守卫确保错误处理的类型安全
- **消息保留**：特别是 403 错误，保留原始消息以提供有用的上下文
- **兼容性处理**：处理 Gaxios 客户端可能返回字符串或对象的情况

### 2. 错误报告系统

为了便于调试和问题追踪，系统在 `packages/core/src/utils/errorReporting.ts` 中实现了详细的错误报告机制：

```typescript
export async function reportError(
  error: Error | unknown,
  baseMessage: string,
  context?: Content[] | Record<string, unknown> | unknown[],
  type = 'general',
  reportingDir = os.tmpdir(),
): Promise<void> {
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
  const reportFileName = `gemini-client-error-${type}-${timestamp}.json`;
  const reportPath = path.join(reportingDir, reportFileName);

  // 错误对象标准化
  let errorToReport: { message: string; stack?: string };
  if (error instanceof Error) {
    errorToReport = { message: error.message, stack: error.stack };
  } else if (
    typeof error === 'object' &&
    error !== null &&
    'message' in error
  ) {
    errorToReport = {
      message: String((error as { message: unknown }).message),
    };
  } else {
    errorToReport = { message: String(error) };
  }

  // 多级降级策略
  try {
    stringifiedReportContent = JSON.stringify(reportContent, null, 2);
  } catch (stringifyError) {
    // 处理包含 BigInt 等不可序列化内容的情况
    console.error(
      `${baseMessage} Could not stringify report content (likely due to context):`,
      stringifyError,
    );
    // 降级：仅尝试报告错误信息
    try {
      const minimalReportContent = { error: errorToReport };
      stringifiedReportContent = JSON.stringify(minimalReportContent, null, 2);
      await fs.writeFile(reportPath, stringifiedReportContent);
      console.error(
        `${baseMessage} Partial report (excluding context) available at: ${reportPath}`,
      );
    } catch (minimalWriteError) {
      console.error(
        `${baseMessage} Failed to write even a minimal error report:`,
        minimalWriteError,
      );
    }
    return;
  }
}
```

错误报告系统的关键特性：
- **时间戳文件名**：使用 ISO 时间戳确保报告文件唯一性
- **多级降级**：从完整报告→最小报告→控制台输出的降级策略
- **序列化处理**：优雅处理不可序列化的对象（如 BigInt）
- **上下文保留**：尽可能保存错误发生时的上下文信息

## 智能重试机制

### 1. 指数退避算法

系统在 `packages/core/src/utils/retry.ts` 中实现了带抖动的指数退避算法，避免重试风暴：

```typescript
export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options?: Partial<RetryOptions>,
): Promise<T> {
  const {
    maxAttempts,
    initialDelayMs,
    maxDelayMs,
    onPersistent429,
    authType,
    shouldRetry,
  } = {
    ...DEFAULT_RETRY_OPTIONS,
    ...options,
  };

  let attempt = 0;
  let currentDelay = initialDelayMs;
  let consecutive429Count = 0;

  while (attempt < maxAttempts) {
    attempt++;
    try {
      return await fn();
    } catch (error) {
      const errorStatus = getErrorStatus(error);

      // 检查 Pro 配额错误 - OAuth 用户立即降级
      if (
        errorStatus === 429 &&
        authType === AuthType.LOGIN_WITH_GOOGLE &&
        isProQuotaExceededError(error) &&
        onPersistent429
      ) {
        const fallbackModel = await onPersistent429(authType, error);
        if (fallbackModel !== false && fallbackModel !== null) {
          // 重置重试计数器，使用新模型继续
          attempt = 0;
          consecutive429Count = 0;
          currentDelay = initialDelayMs;
          continue;
        }
      }

      // 处理重试逻辑
      const { delayDurationMs, errorStatus: delayErrorStatus } =
        getDelayDurationAndStatus(error);

      if (delayDurationMs > 0) {
        // 遵守 Retry-After 头部
        console.warn(
          `Attempt ${attempt} failed with status ${delayErrorStatus ?? 'unknown'}. ` +
          `Retrying after explicit delay of ${delayDurationMs}ms...`,
          error,
        );
        await delay(delayDurationMs);
        currentDelay = initialDelayMs;
      } else {
        // 使用指数退避和抖动
        logRetryAttempt(attempt, error, errorStatus);
        const jitter = currentDelay * 0.3 * (Math.random() * 2 - 1);
        const delayWithJitter = Math.max(0, currentDelay + jitter);
        await delay(delayWithJitter);
        currentDelay = Math.min(maxDelayMs, currentDelay * 2);
      }
    }
  }
}
```

关键算法特性：
- **默认参数**：5 次重试，初始延迟 5 秒，最大延迟 30 秒
- **抖动因子**：±30% 的随机抖动，避免同时重试
- **Retry-After 支持**：优先遵守服务器返回的重试时间
- **连续 429 检测**：检测连续的限流错误并触发降级

### 2. 配额错误的智能处理

对于 API 配额错误，系统在 `packages/core/src/utils/quotaErrorDetection.ts` 中实现了智能的模型降级策略：

```typescript
// 检测 Pro 模型配额错误
export function isProQuotaExceededError(error: unknown): boolean {
  // 匹配如下模式：
  // - "Quota exceeded for quota metric 'Gemini 2.5 Pro Requests'"
  // - "Quota exceeded for quota metric 'Gemini 2.5-preview Pro Requests'"
  // 使用字符串方法而非正则表达式，避免 ReDoS 漏洞
  
  const checkMessage = (message: string): boolean =>
    message.includes("Quota exceeded for quota metric 'Gemini") &&
    message.includes("Pro Requests'");
  
  // 处理多种错误格式
  if (typeof error === 'string') {
    return checkMessage(error);
  }
  
  if (isStructuredError(error)) {
    return checkMessage(error.message);
  }
  
  if (isApiError(error)) {
    return checkMessage(error.error.message);
  }
  
  // 检查 Gaxios 错误的响应数据
  if (error && typeof error === 'object' && 'response' in error) {
    const gaxiosError = error as {
      response?: {
        data?: unknown;
      };
    };
    if (gaxiosError.response && gaxiosError.response.data) {
      if (typeof gaxiosError.response.data === 'string') {
        return checkMessage(gaxiosError.response.data);
      }
      if (
        typeof gaxiosError.response.data === 'object' &&
        gaxiosError.response.data !== null &&
        'error' in gaxiosError.response.data
      ) {
        const errorData = gaxiosError.response.data as {
          error?: { message?: string };
        };
        return checkMessage(errorData.error?.message || '');
      }
    }
  }
  return false;
}

// 通用配额错误检测
export function isGenericQuotaExceededError(error: unknown): boolean {
  if (typeof error === 'string') {
    return error.includes('Quota exceeded for quota metric');
  }
  if (isStructuredError(error)) {
    return error.message.includes('Quota exceeded for quota metric');
  }
  if (isApiError(error)) {
    return error.error.message.includes('Quota exceeded for quota metric');
  }
  return false;
}
```

配额错误处理的设计要点：
- **安全的字符串匹配**：避免使用正则表达式，防止 ReDoS 攻击
- **多格式支持**：处理字符串、结构化错误、API 错误、Gaxios 错误等多种格式
- **精确区分**：区分 Pro 模型配额和通用配额错误，以实施不同的降级策略
- **深度检查**：递归检查嵌套的错误对象结构

## 特定场景的错误处理

### 1. 网络错误处理

网络层在 `packages/core/src/utils/fetch.ts` 中实现了超时控制和私有 IP 保护：

```typescript
// 私有 IP 地址范围定义
const PRIVATE_IP_RANGES = [
  /^10\./,
  /^127\./,
  /^172\.(1[6-9]|2[0-9]|3[0-1])\./,
  /^192\.168\./,
  /^::1$/,
  /^fc00:/,
  /^fe80:/,
];

// 网络获取错误类型
export class FetchError extends Error {
  constructor(
    message: string,
    public code?: string,
  ) {
    super(message);
    this.name = 'FetchError';
  }
}

// 私有 IP 检查
export function isPrivateIp(url: string): boolean {
  try {
    const hostname = new URL(url).hostname;
    return PRIVATE_IP_RANGES.some((range) => range.test(hostname));
  } catch (_e) {
    return false;
  }
}

// 带超时控制的网络请求
export async function fetchWithTimeout(
  url: string,
  timeout: number,
): Promise<Response> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(url, { signal: controller.signal });
    return response;
  } catch (error) {
    if (isNodeError(error) && error.code === 'ABORT_ERR') {
      throw new FetchError(`Request timed out after ${timeout}ms`, 'ETIMEDOUT');
    }
    throw new FetchError(getErrorMessage(error));
  } finally {
    clearTimeout(timeoutId);
  }
}
```

网络错误处理的关键特性：
- **私有 IP 保护**：保护内网资源，避免 SSRF 攻击
- **超时控制**：使用 AbortController 实现可靠的超时
- **错误类型标准化**：通过 FetchError 类型和错误码标准化网络错误
- **资源清理**：使用 finally 确保定时器总是被清理

### 2. 文件操作错误处理

文件操作实现了细粒度的错误处理和恢复：

```typescript
private async _getCorrectedFileContent(
  filePath: string,
  proposedContent: string,
): Promise<GetCorrectedFileContentResult> {
  try {
    originalContent = fs.readFileSync(filePath, 'utf8');
    fileExists = true;
  } catch (err) {
    if (isNodeError(err) && err.code === 'ENOENT') {
      // 文件不存在，将创建新文件
      fileExists = false;
      originalContent = '';
    } else {
      // 权限错误或其他问题
      return {
        originalContent: '',
        correctedContent: proposedContent,
        fileExists: true,
        error: {
          message: getErrorMessage(err),
          code: isNodeError(err) ? err.code : undefined,
        }
      };
    }
  }
  // 继续处理...
}
```

### 3. 工具执行错误处理

工具执行层捕获并格式化各种执行错误：

```typescript
// Shell 工具的错误处理
try {
  const result = await ShellExecutionService.execute(command, cwd, signal);
  
  if (result.aborted) {
    return {
      llmContent: 'Command was cancelled by user',
      returnDisplay: 'Command cancelled by user.',
    };
  }
  
  if (result.error) {
    return {
      llmContent: `Error: ${result.error.message}`,
      returnDisplay: `Command failed: ${getErrorMessage(result.error)}`,
    };
  }
  
  // 处理非零退出码
  if (result.exitCode !== 0) {
    return {
      llmContent: `Command exited with code: ${result.exitCode}`,
      returnDisplay: `Exit code: ${result.exitCode}`,
    };
  }
} catch (error) {
  return {
    llmContent: `Error executing command: ${getErrorMessage(error)}`,
    returnDisplay: `Error: ${getErrorMessage(error)}`,
  };
}
```

## 循环检测与防护

### 1. 工具调用循环检测

系统在 `packages/core/src/services/loopDetectionService.ts` 中实现了全面的循环检测机制：

```typescript
// 循环检测阈值常量
const TOOL_CALL_LOOP_THRESHOLD = 5;
const CONTENT_LOOP_THRESHOLD = 10;
const CONTENT_CHUNK_SIZE = 50;
const MAX_HISTORY_LENGTH = 1000;

// LLM 循环检测相关常量
const LLM_LOOP_CHECK_HISTORY_COUNT = 20;
const LLM_CHECK_AFTER_TURNS = 30;
const DEFAULT_LLM_CHECK_INTERVAL = 3;
const MIN_LLM_CHECK_INTERVAL = 5;
const MAX_LLM_CHECK_INTERVAL = 15;

export class LoopDetectionService {
  private getToolCallKey(toolCall: { name: string; args: object }): string {
    const argsString = JSON.stringify(toolCall.args);
    const keyString = `${toolCall.name}:${argsString}`;
    return createHash('sha256').update(keyString).digest('hex');
  }

  private checkToolCallLoop(toolCall: { name: string; args: object }): boolean {
    const key = this.getToolCallKey(toolCall);
    
    if (this.lastToolCallKey === key) {
      this.toolCallRepetitionCount++;
    } else {
      this.lastToolCallKey = key;
      this.toolCallRepetitionCount = 1;
    }
    
    if (this.toolCallRepetitionCount >= TOOL_CALL_LOOP_THRESHOLD) {
      logLoopDetected(this.config, new LoopDetectedEvent(
        LoopType.CONSECUTIVE_IDENTICAL_TOOL_CALLS,
        this.promptId,
      ));
      return true;
    }
    return false;
  }

  // 主入口：处理流事件并检查循环
  addAndCheck(event: ServerGeminiStreamEvent): boolean {
    if (this.loopDetected) {
      return true;
    }

    switch (event.type) {
      case GeminiEventType.ToolCallRequest:
        // 工具调用时重置内容跟踪
        this.resetContentTracking();
        this.loopDetected = this.checkToolCallLoop(event.value);
        break;
      case GeminiEventType.Content:
        this.loopDetected = this.checkContentLoop(event.value);
        break;
      default:
        break;
    }
    return this.loopDetected;
  }
}
```

工具调用循环检测的关键设计：
- **哈希键生成**：使用 SHA-256 哈希包含工具名和参数的唯一键
- **连续计数**：跟踪连续相同调用的次数
- **阈值触发**：连续 5 次相同调用触发循环检测
- **遥测集成**：检测到循环时记录遥测事件

### 2. 内容循环检测

通过滑动窗口算法检测重复内容：

```typescript
private checkContentLoop(content: string): boolean {
  // 代码块通常包含重复的语法，避免误报
  const numFences = (content.match(/```/g) ?? []).length;
  if (numFences) {
    // 检测到代码栏时重置跟踪
    this.resetContentTracking();
  }

  const wasInCodeBlock = this.inCodeBlock;
  this.inCodeBlock =
    numFences % 2 === 0 ? this.inCodeBlock : !this.inCodeBlock;
  if (wasInCodeBlock) {
    return false;
  }

  this.streamContentHistory += content;
  this.truncateAndUpdate();
  return this.analyzeContentChunksForLoop();
}

private analyzeContentChunksForLoop(): boolean {
  while (this.hasMoreChunksToProcess()) {
    // 提取固定大小的文本块
    const currentChunk = this.streamContentHistory.substring(
      this.lastContentIndex,
      this.lastContentIndex + CONTENT_CHUNK_SIZE,
    );
    const chunkHash = createHash('sha256').update(currentChunk).digest('hex');

    if (this.isLoopDetectedForChunk(currentChunk, chunkHash)) {
      logLoopDetected(
        this.config,
        new LoopDetectedEvent(
          LoopType.CHANTING_IDENTICAL_SENTENCES,
          this.promptId,
        ),
      );
      return true;
    }
    
    // 滑动窗口移动到下一个位置
    this.lastContentIndex++;
  }
  return false;
}

private isLoopDetectedForChunk(chunk: string, hash: string): boolean {
  const existingIndices = this.contentStats.get(hash);

  if (!existingIndices) {
    this.contentStats.set(hash, [this.lastContentIndex]);
    return false;
  }

  // 验证实际内容匹配，防止哈希冲突
  if (!this.isActualContentMatch(chunk, existingIndices[0])) {
    return false;
  }

  existingIndices.push(this.lastContentIndex);

  if (existingIndices.length < CONTENT_LOOP_THRESHOLD) {
    return false;
  }

  // 分析最近的出现位置，检查是否密集出现
  const recentIndices = existingIndices.slice(-CONTENT_LOOP_THRESHOLD);
  const totalDistance =
    recentIndices[recentIndices.length - 1] - recentIndices[0];
  const averageDistance = totalDistance / (CONTENT_LOOP_THRESHOLD - 1);
  const maxAllowedDistance = CONTENT_CHUNK_SIZE * 1.5;

  return averageDistance <= maxAllowedDistance;
}
```

内容循环检测的核心算法：
- **代码块过滤**：识别并跳过代码块，避免语法重复导致的误报
- **滑动窗口**：使用 50 字符的窗口大小扫描内容
- **密度分析**：计算重复块的平均距离，判断是否为循环
- **内存管理**：限制历史长度为 1000 字符，防止内存泄漏

### 3. LLM 辅助的循环检测

对于复杂的循环模式，系统使用 LLM 进行智能检测：

```typescript
// 在新回合开始时检查
async turnStarted(signal: AbortSignal) {
  this.turnsInCurrentPrompt++;

  if (
    this.turnsInCurrentPrompt >= LLM_CHECK_AFTER_TURNS &&
    this.turnsInCurrentPrompt - this.lastCheckTurn >= this.llmCheckInterval
  ) {
    this.lastCheckTurn = this.turnsInCurrentPrompt;
    return await this.checkForLoopWithLLM(signal);
  }

  return false;
}

private async checkForLoopWithLLM(signal: AbortSignal) {
  // 获取最近的对话历史
  const recentHistory = this.config
    .getGeminiClient()
    .getHistory()
    .slice(-LLM_LOOP_CHECK_HISTORY_COUNT);

  // 构建检测提示
  const contents = [
    {
      role: 'user',
      parts: [{
        text: 'Analyze the following conversation history and determine if there is a repetitive loop...',
      }],
    },
    ...recentHistory,
  ];

  // 使用 Flash 模型进行快速检测
  const result = await this.config
    .getGeminiClient()
    .generateJson(contents, schema, signal, DEFAULT_GEMINI_FLASH_MODEL);

  if (result.confidence > 0.9) {
    logLoopDetected(this.config, new LoopDetectedEvent(
      LoopType.LLM_DETECTED_LOOP,
      this.promptId
    ));
    return true;
  }
  
  // 动态调整检查间隔：高置信度时增加检查频率
  this.llmCheckInterval = Math.round(
    MIN_LLM_CHECK_INTERVAL + 
    (MAX_LLM_CHECK_INTERVAL - MIN_LLM_CHECK_INTERVAL) * (1 - result.confidence)
  );
  
  return false;
}
```

LLM 循环检测的智能特性：
- **延迟启动**：30 个回合后才开始检测，避免误报
- **动态间隔**：根据置信度调整检查频率（5-15 回合）
- **历史窗口**：分析最近 20 个对话回合
- **Flash 模型**：使用快速模型降低检测开销

## 用户体验优化

### 1. 分层错误信息

系统在 `packages/cli/src/ui/utils/errorParsing.ts` 中根据用户类型和认证方式提供不同的错误提示：

```typescript
// 免费层用户消息
const getRateLimitErrorMessageGoogleProQuotaFree = (
  currentModel: string = DEFAULT_GEMINI_MODEL,
  fallbackModel: string = DEFAULT_GEMINI_FLASH_MODEL,
) =>
  `\nYou have reached your daily ${currentModel} quota limit. ` +
  `You will be switched to the ${fallbackModel} model for the rest of this session. ` +
  `To increase your limits, upgrade to a Gemini Code Assist Standard or Enterprise plan ` +
  `with higher limits at https://goo.gle/set-up-gemini-code-assist, ` +
  `or use /auth to switch to using a paid API key from AI Studio at https://aistudio.google.com/apikey`;

// 付费层用户消息
const getRateLimitErrorMessageGoogleProQuotaPaid = (
  currentModel: string = DEFAULT_GEMINI_MODEL,
  fallbackModel: string = DEFAULT_GEMINI_FLASH_MODEL,
) =>
  `\nYou have reached your daily ${currentModel} quota limit. ` +
  `You will be switched to the ${fallbackModel} model for the rest of this session. ` +
  `We appreciate you for choosing Gemini Code Assist and the Gemini CLI. ` +
  `To continue accessing the ${currentModel} model today, consider using /auth ` +
  `to switch to using a paid API key from AI Studio at https://aistudio.google.com/apikey`;

function getRateLimitMessage(
  authType?: AuthType,
  error?: unknown,
  userTier?: UserTierId,
  currentModel?: string,
  fallbackModel?: string,
): string {
  switch (authType) {
    case AuthType.LOGIN_WITH_GOOGLE: {
      // 默认为免费层，如果未指定
      const isPaidTier =
        userTier === UserTierId.LEGACY || userTier === UserTierId.STANDARD;
      
      if (isProQuotaExceededError(error)) {
        return isPaidTier
          ? getRateLimitErrorMessageGoogleProQuotaPaid(
              currentModel || DEFAULT_GEMINI_MODEL,
              fallbackModel,
            )
          : getRateLimitErrorMessageGoogleProQuotaFree(
              currentModel || DEFAULT_GEMINI_MODEL,
              fallbackModel,
            );
      } else if (isGenericQuotaExceededError(error)) {
        return isPaidTier
          ? getRateLimitErrorMessageGoogleGenericQuotaPaid(
              currentModel || DEFAULT_GEMINI_MODEL,
            )
          : getRateLimitErrorMessageGoogleGenericQuotaFree();
      }
    }
    case AuthType.USE_GEMINI:
      return RATE_LIMIT_ERROR_MESSAGE_USE_GEMINI;
    case AuthType.USE_VERTEX_AI:
      return RATE_LIMIT_ERROR_MESSAGE_VERTEX;
    default:
      return getRateLimitErrorMessageDefault(fallbackModel);
  }
}

// API 错误解析和格式化
export function parseAndFormatApiError(
  error: unknown,
  authType?: AuthType,
  userTier?: UserTierId,
  currentModel?: string,
  fallbackModel?: string,
): string {
  if (isStructuredError(error)) {
    let text = `[API Error: ${error.message}]`;
    if (error.status === 429) {
      text += getRateLimitMessage(
        authType,
        error,
        userTier,
        currentModel,
        fallbackModel,
      );
    }
    return text;
  }

  // 处理嵌套的 JSON 错误消息
  if (typeof error === 'string') {
    const jsonStart = error.indexOf('{');
    if (jsonStart !== -1) {
      const jsonString = error.substring(jsonStart);
      try {
        const parsedError = JSON.parse(jsonString) as unknown;
        if (isApiError(parsedError)) {
          // 尝试解析嵌套的 JSON 错误
          let finalMessage = parsedError.error.message;
          try {
            const nestedError = JSON.parse(finalMessage) as unknown;
            if (isApiError(nestedError)) {
              finalMessage = nestedError.error.message;
            }
          } catch (_e) {
            // 不是嵌套的 JSON 错误，使用原始消息
          }
          let text = `[API Error: ${finalMessage} (Status: ${parsedError.error.status})]`;
          if (parsedError.error.code === 429) {
            text += getRateLimitMessage(
              authType,
              parsedError,
              userTier,
              currentModel,
              fallbackModel,
            );
          }
          return text;
        }
      } catch (_e) {
        // 不是有效的 JSON
      }
    }
    return `[API Error: ${error}]`;
  }

  return '[API Error: An unknown error occurred.]';
}
```

错误信息分层的设计亮点：
- **用户分级**：区分免费、Legacy、Standard 用户，提供针对性消息
- **升级引导**：免费用户遇到限制时提供升级链接
- **替代方案**：提供使用 API Key 等替代认证方式
- **友好语气**：对付费用户表示感谢，提升用户体验

### 2. 错误显示组件

在 UI 层，错误通过专门的 React 组件进行展示：

```typescript
// packages/cli/src/ui/components/messages/ErrorMessage.tsx
import React from 'react';
import { Text, Box } from 'ink';
import { Colors } from '../../colors.js';

interface ErrorMessageProps {
  text: string;
}

export const ErrorMessage: React.FC<ErrorMessageProps> = ({ text }) => {
  const prefix = '✕ ';
  const prefixWidth = prefix.length;

  return (
    <Box flexDirection="row" marginBottom={1}>
      <Box width={prefixWidth}>
        <Text color={Colors.AccentRed}>{prefix}</Text>
      </Box>
      <Box flexGrow={1}>
        <Text wrap="wrap" color={Colors.AccentRed}>
          {text}
        </Text>
      </Box>
    </Box>
  );
};
```

错误显示组件的设计特点：
- **视觉提示**：使用红色和×符号清晰标识错误
- **布局优化**：使用 Flexbox 布局确保正确的文本对齐
- **文本换行**：支持长错误消息的自动换行
- **间距设置**：通过 marginBottom 确保错误消息之间有适当间距

### 3. 错误上下文保留

错误发生时，系统会保存相关上下文供调试：

```typescript
// 错误报告中保存的完整上下文
const reportContent: ErrorReportData = {
  error: {
    message: error.message,
    stack: error.stack,
  },
  context: chatHistory,  // 保存完整的对话历史
  additionalInfo: {
    model: currentModel,
    authType: authType,
    userTier: userTier,
    timestamp: new Date().toISOString(),
    sessionId: sessionId,
    promptCount: promptCount,
  }
};

// 生成唯一的错误报告文件
const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
const reportFileName = `gemini-client-error-${type}-${timestamp}.json`;
```

### 4. 渐进式信息披露

错误信息根据调试模式显示不同详细程度：

```typescript
// 调试模式下的详细错误信息
if (this.config.getDebugMode()) {
  // 显示完整的 LLM 响应内容
  returnDisplayMessage = llmContent;
  // 记录详细的执行信息
  console.debug('Command execution details:', {
    command,
    cwd,
    exitCode: result.exitCode,
    stdout: result.stdout,
    stderr: result.stderr,
    error: result.error,
  });
} else {
  // 普通模式：显示简化的用户友好信息
  if (result.error) {
    returnDisplayMessage = `Command failed: ${getErrorMessage(result.error)}`;
  } else if (result.exitCode !== 0) {
    returnDisplayMessage = `Command exited with code: ${result.exitCode}`;
    // 仅显示 stderr 的前几行
    if (result.stderr) {
      const lines = result.stderr.split('\n').slice(0, 3);
      returnDisplayMessage += '\n' + lines.join('\n');
    }
  }
}

// 不同错误类型的信息粒度
function formatErrorForDisplay(error: unknown, verboseMode: boolean): string {
  if (verboseMode) {
    // 详细模式：包含堆栈跟踪
    if (error instanceof Error) {
      return `${error.message}\n\nStack trace:\n${error.stack}`;
    }
    return JSON.stringify(error, null, 2);
  } else {
    // 简洁模式：仅显示错误消息
    return getErrorMessage(error);
  }
}
```

信息披露策略：
- **默认简洁**：普通用户看到简化的错误信息
- **调试详细**：开发者模式下显示完整的执行细节
- **进行性披露**：根据错误严重程度逐步增加信息量
- **敏感信息过滤**：确保不会暴露敏感数据

## 错误恢复策略

### 1. 自动恢复机制

- **模型降级**：Pro 模型配额耗尽时自动切换到 Flash 模型
- **认证刷新**：401 错误触发 token 刷新流程
- **会话恢复**：错误后保持会话状态，允许继续对话

### 2. 用户引导恢复

- **升级提示**：免费用户遇到配额限制时提供升级链接
- **替代方案**：提供使用 API Key 等替代认证方式
- **重试建议**：临时错误时建议稍后重试

### 3. 降级服务

- **功能降级**：某些功能不可用时提供基础功能
- **本地处理**：网络错误时尽可能使用本地缓存
- **简化输出**：复杂操作失败时提供简化结果

## 监控与诊断

### 1. 错误遥测

系统集成了完整的错误监控：

```typescript
// 记录循环检测事件
logLoopDetected(config, new LoopDetectedEvent(
  LoopType.CONSECUTIVE_IDENTICAL_TOOL_CALLS,
  promptId,
));

// 记录文件操作错误
recordFileOperationMetric(
  config,
  FileOperation.CREATE,
  lines,
  mimetype,
  extension,
);
```

### 2. 调试支持

- **错误报告生成**：自动生成包含上下文的错误报告
- **堆栈跟踪**：调试模式下提供完整堆栈信息
- **时间戳记录**：所有错误包含精确时间戳

## 最佳实践总结

### 1. 错误处理分层架构

- **早期验证**：在操作前进行参数验证，避免无效操作
- **分层处理**：每层处理特定类型的错误，避免耦合
- **统一抽象**：使用标准化的错误类型和工具函数

### 2. 用户体验优先

- **友好提示**：根据用户类型提供针对性的错误信息
- **渐进披露**：默认简洁，调试模式下提供详细信息
- **视觉反馈**：使用颜色和符号清晰标识错误

### 3. 智能重试策略

- **区分错误**：区分临时错误和永久错误，合理重试
- **指数退避**：使用带抖动的指数退避算法
- **遵守服务器指示**：优先使用 Retry-After 头部

### 4. 降级与恢复

- **多级降级**：提供多级降级方案，确保基本功能可用
- **自动切换**：Pro 模型配额耗尽时自动切换到 Flash
- **保持会话**：错误后保持会话状态，允许继续对话

### 5. 调试与监控

- **上下文保留**：错误时保存足够的上下文信息
- **错误报告**：生成详细的错误报告文件
- **遥测集成**：完整的错误追踪和分析能力

### 6. 安全性考虑

- **避免信息泄漏**：不暴露敏感的系统信息
- **ReDoS 防护**：使用字符串方法而非复杂正则
- **私有 IP 保护**：检查并拒绝访问内网地址

## 本章小结

Gemini CLI 的错误处理机制体现了现代 CLI 工具的设计理念：通过分层的错误处理架构、智能的重试策略、友好的用户提示和完善的监控体系，构建了一个既健壮又易用的系统。

关键亮点包括：

1. **全面的错误类型体系**：从基础的 HTTP 错误到复杂的循环检测，每种错误都有明确的处理路径

2. **智能的降级机制**：在 API 配额耗尽时自动切换模型，确保服务的连续性

3. **多层次的循环检测**：结合哈希比对、滑动窗口和 LLM 智能检测，有效防止无限循环

4. **优秀的用户体验设计**：根据用户类型和调试模式提供适当粒度的错误信息

5. **完善的调试支持**：详细的错误报告和上下文保存机制，便于问题追踪和分析

特别是在处理 API 配额、网络异常、循环检测等关键场景时，系统展现出了优秀的错误处理和恢复能力，确保用户能够获得流畅的使用体验。这些设计思想和实践经验对于构建其他大型 CLI 应用具有重要的参考价值。