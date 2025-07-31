# 第 14 章：工具执行流程

工具执行是 Gemini CLI 的核心功能之一，它允许 AI 模型通过调用各种工具来完成文件操作、代码编辑、搜索等任务。本章详细分析从 AI 请求工具调用到执行完成的完整流程，包括参数验证、用户确认、并行执行和结果回传等关键环节。

## 14.1 工具执行流程概览

工具执行涉及多个组件的协同工作：

```
AI 模型 → FunctionCall → Turn 处理 → CoreToolScheduler → Tool 执行 → 结果回传
                                           ↓
                                    用户确认（可选）
```

主要流程步骤：

1. **AI 请求阶段**：Gemini API 返回包含 FunctionCall 的响应
2. **事件转换阶段**：Turn 将 FunctionCall 转换为 ToolCallRequest 事件
3. **调度阶段**：CoreToolScheduler 管理工具调用的生命周期
4. **确认阶段**：根据配置和工具类型决定是否需要用户确认
5. **执行阶段**：实际执行工具逻辑
6. **响应阶段**：将结果转换为 FunctionResponse 返回给 AI

## 14.2 工具调用的触发机制

### 14.2.1 从 Gemini API 响应到工具调用请求

Gemini API 通过 FunctionCall 机制与工具系统集成。当 AI 模型分析用户请求后判断需要执行特定操作（如文件编辑、代码搜索、命令执行等）时，会在流式响应中包含 FunctionCall 部分。

Turn 类负责处理 Gemini API 的流式响应，并识别其中的工具调用请求：

```typescript
// packages/core/src/core/turn.ts
private handlePendingFunctionCall(
  fnCall: FunctionCall,
): ServerGeminiStreamEvent | null {
  // 生成唯一的调用 ID，如果 API 未提供则创建一个
  const callId =
    fnCall.id ??
    `${fnCall.name}-${Date.now()}-${Math.random().toString(16).slice(2)}`;
  const name = fnCall.name || 'undefined_tool_name';
  const args = (fnCall.args || {}) as Record<string, unknown>;

  // 构建标准化的工具调用请求
  const toolCallRequest: ToolCallRequestInfo = {
    callId,
    name,
    args,
    isClientInitiated: false,  // 标记为 AI 发起的调用
    prompt_id: this.prompt_id, // 关联到当前对话轮次
  };

  // 收集所有待处理的工具调用
  this.pendingToolCalls.push(toolCallRequest);

  // 发出工具调用请求事件供下游处理
  return { type: GeminiEventType.ToolCallRequest, value: toolCallRequest };
}
```

关键设计点：
- **唯一标识**：每个工具调用都有唯一的 callId，用于追踪整个生命周期
- **批量收集**：Turn 会收集一轮对话中的所有工具调用，支持并行执行
- **事件驱动**：通过事件机制解耦 API 响应处理和工具执行逻辑

### 14.2.2 事件流处理

在 CLI 层，`useGeminiStream` hook 作为核心协调器，负责处理从 Core 层传递的各种事件。它维护了完整的事件处理管道：

```typescript
// packages/cli/src/ui/hooks/useGeminiStream.ts
case ServerGeminiEventType.ToolCallRequest:
  // 收集所有工具调用请求，等待流结束后统一处理
  toolCallRequests.push(event.value);
  break;

// 流处理完成后，批量调度工具调用
if (toolCallRequests.length > 0) {
  // scheduleToolCalls 内部会：
  // 1. 创建 CoreToolScheduler 实例（如果需要）
  // 2. 批量提交所有工具调用请求
  // 3. 管理确认流程和执行状态
  scheduleToolCalls(toolCallRequests, signal);
}
```

事件流处理的关键特性：

1. **批量优化**：等待 AI 响应流完全结束后再调度工具，避免部分执行
2. **取消支持**：通过 AbortSignal 支持用户随时取消操作
3. **状态同步**：React 层通过 `useReactToolScheduler` 与 Core 层的调度器保持同步

## 14.3 CoreToolScheduler：工具调度核心

CoreToolScheduler 是工具执行的核心协调器，实现了复杂的状态管理、并发控制和用户交互逻辑。它是整个工具系统的大脑，确保工具调用的安全性和正确性。

### 14.3.1 工具调用状态机

每个工具调用都是一个状态机，经历严格定义的状态转换。CoreToolScheduler 维护了详细的状态类型定义：

```typescript
// 基础状态类型
type Status = 
  | 'validating'        // 验证参数
  | 'awaiting_approval' // 等待用户确认
  | 'scheduled'         // 已调度，等待执行
  | 'executing'         // 执行中
  | 'success'          // 执行成功
  | 'error'            // 执行失败
  | 'cancelled';       // 已取消

// 每个状态对应的详细类型定义
type ValidatingToolCall = {
  status: 'validating';
  request: ToolCallRequestInfo;
  tool: Tool;
  startTime?: number;
  outcome?: ToolConfirmationOutcome;
};

type WaitingToolCall = {
  status: 'awaiting_approval';
  request: ToolCallRequestInfo;
  tool: Tool;
  confirmationDetails: ToolCallConfirmationDetails;
  startTime?: number;
  outcome?: ToolConfirmationOutcome;
};

type ExecutingToolCall = {
  status: 'executing';
  request: ToolCallRequestInfo;
  tool: Tool;
  liveOutput?: string;  // 支持实时输出
  startTime?: number;
  outcome?: ToolConfirmationOutcome;
};

// 联合类型确保类型安全
type ToolCall =
  | ValidatingToolCall
  | ScheduledToolCall
  | ErroredToolCall
  | SuccessfulToolCall
  | ExecutingToolCall
  | CancelledToolCall
  | WaitingToolCall;
```

状态转换规则：
- `validating` → `awaiting_approval` 或 `scheduled`（取决于是否需要确认）
- `awaiting_approval` → `scheduled`、`cancelled` 或保持（用户修改时）
- `scheduled` → `executing`（当所有工具都准备好时）
- `executing` → `success` 或 `error`
- 任何状态都可以转换到 `cancelled`（用户取消时）

### 14.3.2 调度流程实现

调度流程是 CoreToolScheduler 的核心功能，它确保工具调用的有序性、安全性和可控性：

```typescript
// packages/core/src/core/coreToolScheduler.ts
async schedule(
  request: ToolCallRequestInfo | ToolCallRequestInfo[],
  signal: AbortSignal,
): Promise<void> {
  // 1. 防止并发调度 - 确保没有正在执行或等待确认的工具
  if (this.isRunning()) {
    throw new Error(
      'Cannot schedule new tool calls while other tool calls are actively running (executing or awaiting approval).',
    );
  }

  const requestsToProcess = Array.isArray(request) ? request : [request];
  const toolRegistry = await this.toolRegistry;

  // 2. 创建工具调用对象，处理工具未找到的情况
  const newToolCalls: ToolCall[] = requestsToProcess.map(
    (reqInfo): ToolCall => {
      const toolInstance = toolRegistry.getTool(reqInfo.name);
      if (!toolInstance) {
        // 立即返回错误状态
        return {
          status: 'error',
          request: reqInfo,
          response: createErrorResponse(
            reqInfo,
            new Error(`Tool "${reqInfo.name}" not found in registry.`),
          ),
          durationMs: 0,
        };
      }
      return {
        status: 'validating',
        request: reqInfo,
        tool: toolInstance,
        startTime: Date.now(),
      };
    },
  );

  // 更新内部状态并通知观察者
  this.toolCalls = this.toolCalls.concat(newToolCalls);
  this.notifyToolCallsUpdate();

  // 3. 验证和确认阶段
  for (const toolCall of newToolCalls) {
    if (toolCall.status !== 'validating') {
      continue; // 跳过已经是错误状态的工具
    }

    const { request: reqInfo, tool: toolInstance } = toolCall;
    try {
      // YOLO 模式：跳过所有确认，直接调度
      if (this.config.getApprovalMode() === ApprovalMode.YOLO) {
        this.setStatusInternal(reqInfo.callId, 'scheduled');
      } else {
        // 询问工具是否需要用户确认
        const confirmationDetails = await toolInstance.shouldConfirmExecute(
          reqInfo.args,
          signal,
        );

        if (confirmationDetails) {
          // 包装确认回调，添加状态管理逻辑
          const originalOnConfirm = confirmationDetails.onConfirm;
          const wrappedConfirmationDetails: ToolCallConfirmationDetails = {
            ...confirmationDetails,
            onConfirm: (
              outcome: ToolConfirmationOutcome,
              payload?: ToolConfirmationPayload,
            ) =>
              this.handleConfirmationResponse(
                reqInfo.callId,
                originalOnConfirm,
                outcome,
                signal,
                payload,
              ),
          };
          this.setStatusInternal(
            reqInfo.callId,
            'awaiting_approval',
            wrappedConfirmationDetails,
          );
        } else {
          // 不需要确认，直接调度
          this.setStatusInternal(reqInfo.callId, 'scheduled');
        }
      }
    } catch (error) {
      // 验证阶段出错
      this.setStatusInternal(
        reqInfo.callId,
        'error',
        createErrorResponse(
          reqInfo,
          error instanceof Error ? error : new Error(String(error)),
        ),
      );
    }
  }

  // 4. 尝试执行已调度的工具
  this.attemptExecutionOfScheduledCalls(signal);
  // 5. 检查是否所有工具都已完成
  this.checkAndNotifyCompletion();
}
```

调度流程的关键机制：
- **并发保护**：防止在有工具正在执行时启动新的调度
- **容错处理**：工具未找到或验证失败不会影响其他工具
- **灵活的确认机制**：支持 YOLO、AUTO_EDIT、MANUAL 等多种模式
- **事件通知**：每次状态变化都会通知观察者

## 14.4 用户确认机制

### 14.4.1 确认类型

系统支持多种确认类型，每种类型针对不同的风险级别和操作类型提供定制化的 UI 展示：

```typescript
// packages/core/src/tools/tools.ts
type ToolCallConfirmationDetails =
  | ToolEditConfirmationDetails    // 文件编辑确认
  | ToolExecuteConfirmationDetails  // 命令执行确认
  | ToolMcpConfirmationDetails     // MCP 工具确认
  | ToolInfoConfirmationDetails;   // 信息获取确认

// 文件编辑确认的详细结构
interface ToolEditConfirmationDetails {
  type: 'edit';
  fileName: string;          // 要编辑的文件名
  fileDiff: string;          // diff 内容
  originalContent: string;   // 原始内容
  newContent: string;        // 新内容
  isModifying?: boolean;     // 是否正在外部编辑器中修改
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
}

// 命令执行确认的详细结构
interface ToolExecuteConfirmationDetails {
  type: 'exec';
  rootCommand: string;       // 要执行的根命令
  directory?: string;        // 执行目录
  riskLevel: 'low' | 'medium' | 'high';  // 风险级别
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
}
```

### 14.4.2 确认选项

用户可以选择以下操作：

```typescript
enum ToolConfirmationOutcome {
  ProceedOnce = 'proceed_once',              // 仅允许这一次
  ProceedAlways = 'proceed_always',          // 总是允许此类操作
  ProceedAlwaysServer = 'proceed_always_server', // 总是允许此服务器
  ProceedAlwaysTool = 'proceed_always_tool',     // 总是允许此工具
  ModifyWithEditor = 'modify_with_editor',   // 使用编辑器修改
  Cancel = 'cancel',                         // 取消操作
}
```

### 14.4.3 编辑确认的特殊处理

文件编辑操作是最复杂的确认场景，因为它支持用户在确认前修改内容。这种灵活性通过 ModifiableTool 接口和外部编辑器集成实现：

```typescript
// packages/cli/src/ui/components/messages/ToolConfirmationMessage.tsx
if (confirmationDetails.type === 'edit') {
  if (confirmationDetails.isModifying) {
    // 显示"正在修改"状态 - 用户正在外部编辑器中编辑
    return (
      <Box
        minWidth="90%"
        borderStyle="round"
        borderColor={Colors.Gray}
        justifyContent="space-around"
        padding={1}
        overflow="hidden"
      >
        <Text>Modify in progress: </Text>
        <Text color={Colors.AccentGreen}>
          Save and close external editor to continue
        </Text>
      </Box>
    );
  }

  question = `Apply this change?`;
  options.push(
    {
      label: 'Yes, allow once',
      value: ToolConfirmationOutcome.ProceedOnce,
    },
    {
      label: 'Yes, allow always',
      value: ToolConfirmationOutcome.ProceedAlways,
    },
    {
      label: 'Modify with external editor',  // 允许用户使用外部编辑器
      value: ToolConfirmationOutcome.ModifyWithEditor,
    },
    { label: 'No (esc)', value: ToolConfirmationOutcome.Cancel },
  );

  // 显示 diff 和确认选项
  bodyContent = (
    <DiffRenderer
      diffContent={confirmationDetails.fileDiff}
      filename={confirmationDetails.fileName}
      availableTerminalHeight={availableBodyContentHeight()}
      terminalWidth={childWidth}
    />
  );
}
```

当用户选择 "Modify with external editor" 时，CoreToolScheduler 会：

```typescript
// packages/core/src/core/coreToolScheduler.ts
if (outcome === ToolConfirmationOutcome.ModifyWithEditor) {
  const waitingToolCall = toolCall as WaitingToolCall;
  if (isModifiableTool(waitingToolCall.tool)) {
    const modifyContext = waitingToolCall.tool.getModifyContext(signal);
    const editorType = this.getPreferredEditor();
    
    // 更新状态为"正在修改"
    this.setStatusInternal(callId, 'awaiting_approval', {
      ...waitingToolCall.confirmationDetails,
      isModifying: true,
    });

    // 调用外部编辑器
    const { updatedParams, updatedDiff } = await modifyWithEditor(
      waitingToolCall.request.args,
      modifyContext,
      editorType,
      signal,
    );

    // 更新参数和 diff
    this.setArgsInternal(callId, updatedParams);
    this.setStatusInternal(callId, 'awaiting_approval', {
      ...waitingToolCall.confirmationDetails,
      fileDiff: updatedDiff,
      isModifying: false,
    });
  }
}
```

## 14.5 工具执行实现

### 14.5.1 并行执行机制

CoreToolScheduler 实现了智能的并行执行机制，它会等待所有工具都准备就绪后再统一执行。这种设计确保了用户体验的一致性，避免部分工具先执行导致的困惑：

```typescript
// packages/core/src/core/coreToolScheduler.ts
private attemptExecutionOfScheduledCalls(signal: AbortSignal): void {
  // 检查所有工具是否都处于可执行状态
  const allCallsFinalOrScheduled = this.toolCalls.every(
    (call) =>
      call.status === 'scheduled' ||     // 已调度，可执行
      call.status === 'cancelled' ||     // 已取消
      call.status === 'success' ||       // 已成功
      call.status === 'error',           // 已失败
  );

  if (allCallsFinalOrScheduled) {
    const callsToExecute = this.toolCalls.filter(
      (call) => call.status === 'scheduled',
    );

    // 并行执行所有已调度的工具
    callsToExecute.forEach((toolCall) => {
      this.executeToolCall(toolCall, signal);
    });
  }
}

private async executeToolCall(
  scheduledCall: ScheduledToolCall,
  signal: AbortSignal,
): Promise<void> {
  const { callId, name: toolName } = scheduledCall.request;
  
  // 转换为执行中状态
  this.setStatusInternal(callId, 'executing');

  try {
    // 设置实时输出回调（如果工具支持）
    const liveOutputCallback = this.createLiveOutputCallback(scheduledCall);

    // 执行工具
    const toolResult = await scheduledCall.tool.execute(
      scheduledCall.request.args,
      signal,
      liveOutputCallback,
    );

    // 处理执行结果
    if (signal.aborted) {
      this.setStatusInternal(callId, 'cancelled', 'User cancelled tool execution.');
    } else {
      const response = convertToFunctionResponse(
        toolName,
        callId,
        toolResult.llmContent,
      );
      this.setStatusInternal(callId, 'success', {
        callId,
        responseParts: response,
        resultDisplay: toolResult.returnDisplay,
        error: undefined,
      });
    }
  } catch (executionError) {
    this.setStatusInternal(
      callId,
      'error',
      createErrorResponse(scheduledCall.request, executionError),
    );
  }
}
```

并行执行的优势：
- **效率最大化**：多个独立工具可以同时执行
- **体验一致性**：用户看到的是批量执行，而非零散的单个执行
- **错误隔离**：一个工具的失败不会影响其他工具的执行

### 14.5.2 实时输出支持

实时输出是提升用户体验的重要特性，特别是对于长时间运行的工具（如 shell 命令、文件搜索等）。系统通过回调机制实现了流式输出更新：

```typescript
// packages/core/src/core/coreToolScheduler.ts
private createLiveOutputCallback(
  scheduledCall: ScheduledToolCall,
): ((outputChunk: string) => void) | undefined {
  const { callId } = scheduledCall.request;
  
  // 检查工具是否支持实时输出
  if (!scheduledCall.tool.canUpdateOutput || !this.outputUpdateHandler) {
    return undefined;
  }

  return (outputChunk: string) => {
    // 通知外部处理器（通常是 UI 层）
    if (this.outputUpdateHandler) {
      this.outputUpdateHandler(callId, outputChunk);
    }
    
    // 更新内部状态
    this.toolCalls = this.toolCalls.map((tc) =>
      tc.request.callId === callId && tc.status === 'executing'
        ? { ...tc, liveOutput: outputChunk }
        : tc,
    );
    
    // 通知观察者状态变化
    this.notifyToolCallsUpdate();
  };
}
```

以 Shell 工具为例，它如何实现实时输出：

```typescript
// packages/core/src/tools/shell.ts
const { result: resultPromise } = ShellExecutionService.execute(
  commandToExecute,
  cwd,
  (event: ShellOutputEvent) => {
    if (!updateOutput) return;

    let currentDisplayOutput = '';
    let shouldUpdate = false;

    switch (event.type) {
      case 'data':
        // 累积输出
        if (event.stream === 'stdout') {
          cumulativeStdout += event.chunk;
        } else {
          cumulativeStderr += event.chunk;
        }
        currentDisplayOutput =
          cumulativeStdout +
          (cumulativeStderr ? `\n${cumulativeStderr}` : '');
        
        // 节流：每 100ms 更新一次
        if (Date.now() - lastUpdateTime > OUTPUT_UPDATE_INTERVAL_MS) {
          shouldUpdate = true;
        }
        break;
        
      case 'binary_detected':
        // 二进制流特殊处理
        isBinaryStream = true;
        currentDisplayOutput = '[Binary output detected. Halting stream...]';
        shouldUpdate = true;
        break;
        
      case 'binary_progress':
        // 显示二进制数据接收进度
        currentDisplayOutput = `[Receiving binary output... ${formatMemoryUsage(
          event.bytesReceived,
        )} received]`;
        if (Date.now() - lastUpdateTime > OUTPUT_UPDATE_INTERVAL_MS) {
          shouldUpdate = true;
        }
        break;
    }

    if (shouldUpdate) {
      updateOutput(currentDisplayOutput);
      lastUpdateTime = Date.now();
    }
  },
  signal,
);
```

实时输出的关键考量：
- **节流控制**：避免过于频繁的 UI 更新
- **二进制处理**：检测并特殊处理二进制输出
- **内容累积**：保留完整输出历史供最终结果使用

## 14.6 结果处理与回传

### 14.6.1 结果格式转换

工具执行结果需要转换为 Gemini API 期望的 FunctionResponse 格式。这个转换过程需要处理多种可能的返回类型，包括纯文本、媒体内容和复杂的结构化数据：

```typescript
// packages/core/src/core/coreToolScheduler.ts
export function convertToFunctionResponse(
  toolName: string,
  callId: string,
  llmContent: PartListUnion,
): PartListUnion {
  // 处理数组中只有一个元素的情况
  const contentToProcess =
    Array.isArray(llmContent) && llmContent.length === 1
      ? llmContent[0]
      : llmContent;

  // 简单字符串响应
  if (typeof contentToProcess === 'string') {
    return {
      functionResponse: {
        id: callId,
        name: toolName,
        response: { output: contentToProcess },
      },
    };
  }

  // 处理多部分响应（如文本 + 图片）
  if (Array.isArray(llmContent)) {
    const functionResponse = createFunctionResponsePart(
      callId,
      toolName,
      'Tool execution succeeded.',
    );
    return [functionResponse, ...llmContent];
  }

  // 处理已经是 FunctionResponse 格式的情况
  if (
    typeof contentToProcess === 'object' &&
    contentToProcess !== null &&
    'functionResponse' in contentToProcess
  ) {
    return contentToProcess;
  }

  // 其他复杂对象，作为媒体内容返回
  const functionResponse = createFunctionResponsePart(
    callId,
    toolName,
    'Tool execution succeeded.',
  );
  return [functionResponse, contentToProcess as Part];
}

// 辅助函数：创建标准的 FunctionResponse Part
function createFunctionResponsePart(
  callId: string,
  toolName: string,
  output: string,
): Part {
  return {
    functionResponse: {
      id: callId,
      name: toolName,
      response: { output },
    },
  };
}
```

转换逻辑的设计考量：
- **灵活性**：支持多种返回类型，包括媒体内容
- **一致性**：确保所有响应都包含 FunctionResponse 元素
- **可扩展性**：新的工具类型可以返回自定义格式

### 14.6.2 错误处理

工具执行失败时，错误信息会被格式化为 FunctionResponse：

```typescript
const createErrorResponse = (
  request: ToolCallRequestInfo,
  error: Error,
): ToolCallResponseInfo => ({
  callId: request.callId,
  error,
  responseParts: {
    functionResponse: {
      id: request.callId,
      name: request.name,
      response: { error: error.message },
    },
  },
  resultDisplay: error.message,
});
```

## 14.7 非交互式执行

对于批处理或自动化场景，系统提供了非交互式执行器：

```typescript
// packages/core/src/core/nonInteractiveToolExecutor.ts
export async function executeToolCall(
  config: Config,
  toolCallRequest: ToolCallRequestInfo,
  toolRegistry: ToolRegistry,
  abortSignal?: AbortSignal,
): Promise<ToolCallResponseInfo> {
  const tool = toolRegistry.getTool(toolCallRequest.name);
  
  if (!tool) {
    return createErrorResponse(toolCallRequest, new Error('Tool not found'));
  }

  try {
    // 直接执行，无需确认或实时输出
    const toolResult = await tool.execute(
      toolCallRequest.args,
      abortSignal,
    );

    return {
      callId: toolCallRequest.callId,
      responseParts: convertToFunctionResponse(
        toolCallRequest.name,
        toolCallRequest.callId,
        toolResult.llmContent,
      ),
      resultDisplay: toolResult.returnDisplay,
      error: undefined,
    };
  } catch (e) {
    return createErrorResponse(toolCallRequest, e);
  }
}
```

## 14.8 工具执行的可观测性

### 14.8.1 执行日志

每次工具调用都会记录详细的执行信息：

```typescript
logToolCall(config, {
  'event.name': 'tool_call',
  'event.timestamp': new Date().toISOString(),
  function_name: toolCallRequest.name,
  function_args: toolCallRequest.args,
  duration_ms: durationMs,
  success: true,
  prompt_id: toolCallRequest.prompt_id,
});
```

### 14.8.2 UI 状态更新

React 层通过 `useReactToolScheduler` hook 管理 UI 状态：

```typescript
// packages/cli/src/ui/hooks/useReactToolScheduler.ts
const toolCallsUpdateHandler: ToolCallsUpdateHandler = useCallback(
  (updatedCoreToolCalls: ToolCall[]) => {
    setToolCallsForDisplay((prevTrackedCalls) =>
      updatedCoreToolCalls.map((coreTc) => {
        const existingTrackedCall = prevTrackedCalls.find(
          (ptc) => ptc.request.callId === coreTc.request.callId,
        );
        return {
          ...coreTc,
          responseSubmittedToGemini:
            existingTrackedCall?.responseSubmittedToGemini ?? false,
        } as TrackedToolCall;
      }),
    );
  },
  [setToolCallsForDisplay],
);
```

## 14.9 高级特性

### 14.9.1 可修改工具

某些工具（如文件编辑）支持在执行前修改参数：

```typescript
// packages/core/src/tools/modifiable-tool.ts
export interface ModifiableTool<ToolParams> extends Tool<ToolParams> {
  getModifyContext(abortSignal: AbortSignal): ModifyContext<ToolParams>;
}

export interface ModifyContext<ToolParams> {
  getFilePath: (params: ToolParams) => string;
  getCurrentContent: (params: ToolParams) => Promise<string>;
  getProposedContent: (params: ToolParams) => Promise<string>;
  createUpdatedParams: (
    oldContent: string,
    modifiedProposedContent: string,
    originalParams: ToolParams,
  ) => ToolParams;
}
```

### 14.9.2 批量确认模式

通过 `ApprovalMode` 配置，可以控制确认行为：

```typescript
enum ApprovalMode {
  YOLO = 'yolo',           // 跳过所有确认
  AUTO_EDIT = 'auto_edit', // 自动确认编辑操作
  MANUAL = 'manual',       // 手动确认每个操作
}
```

## 本章小结

工具执行流程是 Gemini CLI 实现 AI 辅助功能的核心机制。通过 CoreToolScheduler 的状态管理、灵活的确认机制、并行执行能力和实时输出支持，系统能够安全、高效地执行各种工具操作。非交互式执行器的存在使得系统也能很好地支持自动化场景。整个流程设计充分考虑了用户体验、安全性和扩展性，为 AI 与实际工作环境的深度集成提供了坚实基础。