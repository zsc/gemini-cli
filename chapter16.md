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

系统定义了清晰的错误类型层次结构，便于精确处理不同类型的错误：

```typescript
// 基础错误类型
export class ForbiddenError extends Error {}    // HTTP 403
export class UnauthorizedError extends Error {}  // HTTP 401
export class BadRequestError extends Error {}    // HTTP 400

// 网络错误类型
export class FetchError extends Error {
  constructor(message: string, public code?: string) {
    super(message);
    this.name = 'FetchError';
  }
}
```

## 核心错误处理机制

### 1. 错误捕获与转换

系统提供了统一的错误处理工具函数，确保错误信息的安全提取和友好展示：

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
          return new ForbiddenError(data.error.message);
      }
    }
  }
  return error;
}
```

### 2. 错误报告系统

为了便于调试和问题追踪，系统实现了详细的错误报告机制：

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

  // 构建错误报告内容
  const reportContent: ErrorReportData = {
    error: errorToReport,
    context: context,
  };

  // 多级降级策略
  try {
    // 尝试完整报告
    await fs.writeFile(reportPath, JSON.stringify(reportContent, null, 2));
  } catch {
    try {
      // 降级：仅保存错误信息
      const minimalReport = { error: errorToReport };
      await fs.writeFile(reportPath, JSON.stringify(minimalReport, null, 2));
    } catch {
      // 最终降级：输出到控制台
      console.error('Failed to write error report');
    }
  }
}
```

## 智能重试机制

### 1. 指数退避算法

系统实现了带抖动的指数退避算法，避免重试风暴：

```typescript
export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options?: Partial<RetryOptions>,
): Promise<T> {
  const {
    maxAttempts = 5,
    initialDelayMs = 5000,
    maxDelayMs = 30000,
    shouldRetry,
  } = options;

  let currentDelay = initialDelayMs;
  
  while (attempt < maxAttempts) {
    try {
      return await fn();
    } catch (error) {
      // 检查是否应该重试
      if (!shouldRetry(error as Error)) {
        throw error;
      }

      // 计算延迟时间
      const jitter = currentDelay * 0.3 * (Math.random() * 2 - 1);
      const delayWithJitter = Math.max(0, currentDelay + jitter);
      
      await delay(delayWithJitter);
      currentDelay = Math.min(maxDelayMs, currentDelay * 2);
    }
  }
}
```

### 2. 配额错误的智能处理

对于 API 配额错误，系统实现了智能的模型降级策略：

```typescript
// 检测 Pro 模型配额错误
export function isProQuotaExceededError(error: unknown): boolean {
  const checkMessage = (message: string): boolean =>
    message.includes("Quota exceeded for quota metric 'Gemini") &&
    message.includes("Pro Requests'");
  
  // 多种错误格式的检测
  if (typeof error === 'string') {
    return checkMessage(error);
  }
  if (isStructuredError(error)) {
    return checkMessage(error.message);
  }
  // ... 更多检测逻辑
}

// 自动降级到 Flash 模型
if (isProQuotaExceededError(error) && authType === AuthType.LOGIN_WITH_GOOGLE) {
  const fallbackModel = await onPersistent429(authType, error);
  if (fallbackModel) {
    // 重置重试计数，使用新模型继续
    attempt = 0;
    continue;
  }
}
```

## 特定场景的错误处理

### 1. 网络错误处理

网络层实现了超时控制和私有 IP 保护：

```typescript
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

系统通过哈希比对检测重复的工具调用：

```typescript
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
```

### 2. 内容循环检测

通过滑动窗口算法检测重复内容：

```typescript
private analyzeContentChunksForLoop(): boolean {
  while (this.hasMoreChunksToProcess()) {
    const currentChunk = this.streamContentHistory.substring(
      this.lastContentIndex,
      this.lastContentIndex + CONTENT_CHUNK_SIZE,
    );
    const chunkHash = createHash('sha256').update(currentChunk).digest('hex');

    if (this.isLoopDetectedForChunk(currentChunk, chunkHash)) {
      return true;
    }
    
    this.lastContentIndex++;
  }
  return false;
}
```

### 3. LLM 辅助的循环检测

对于复杂的循环模式，系统使用 LLM 进行智能检测：

```typescript
private async checkForLoopWithLLM(signal: AbortSignal) {
  const recentHistory = this.config
    .getGeminiClient()
    .getHistory()
    .slice(-LLM_LOOP_CHECK_HISTORY_COUNT);

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
  
  // 动态调整检查间隔
  this.llmCheckInterval = Math.round(
    MIN_LLM_CHECK_INTERVAL + 
    (MAX_LLM_CHECK_INTERVAL - MIN_LLM_CHECK_INTERVAL) * (1 - result.confidence)
  );
}
```

## 用户体验优化

### 1. 分层错误信息

系统根据用户类型和认证方式提供不同的错误提示：

```typescript
function getRateLimitMessage(
  authType?: AuthType,
  error?: unknown,
  userTier?: UserTierId,
  currentModel?: string,
  fallbackModel?: string,
): string {
  switch (authType) {
    case AuthType.LOGIN_WITH_GOOGLE: {
      const isPaidTier = userTier === UserTierId.LEGACY || 
                        userTier === UserTierId.STANDARD;
      
      if (isProQuotaExceededError(error)) {
        return isPaidTier
          ? `您已达到 ${currentModel} 的每日配额限制。感谢您选择 Gemini Code Assist。`
          : `您已达到 ${currentModel} 的每日配额限制。升级到付费计划以获得更高限制。`;
      }
    }
    // 更多情况...
  }
}
```

### 2. 错误上下文保留

错误发生时，系统会保存相关上下文供调试：

```typescript
// 保存聊天历史
const reportContent = {
  error: {
    message: error.message,
    stack: error.stack,
  },
  context: chatHistory,
  additionalInfo: {
    model: currentModel,
    authType: authType,
    timestamp: new Date().toISOString(),
  }
};
```

### 3. 渐进式信息披露

错误信息根据调试模式显示不同详细程度：

```typescript
if (this.config.getDebugMode()) {
  // 调试模式：显示完整错误信息
  returnDisplayMessage = llmContent;
} else {
  // 普通模式：显示简化的用户友好信息
  if (result.error) {
    returnDisplayMessage = `Command failed: ${getErrorMessage(result.error)}`;
  } else if (result.exitCode !== 0) {
    returnDisplayMessage = `Command exited with code: ${result.exitCode}`;
  }
}
```

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

1. **早期验证**：在操作前进行参数验证，避免无效操作
2. **友好提示**：根据用户类型提供针对性的错误信息
3. **智能重试**：区分临时错误和永久错误，合理重试
4. **上下文保留**：错误时保存足够的上下文信息
5. **降级策略**：提供多级降级方案，确保基本功能可用
6. **监控集成**：完整的错误追踪和分析能力

## 本章小结

Gemini CLI 的错误处理机制体现了现代 CLI 工具的设计理念：通过分层的错误处理架构、智能的重试策略、友好的用户提示和完善的监控体系，构建了一个既健壮又易用的系统。特别是在处理 API 配额、网络异常、循环检测等关键场景时，系统展现出了优秀的错误处理和恢复能力，确保用户能够获得流畅的使用体验。