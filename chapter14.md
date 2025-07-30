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

当 AI 模型决定需要使用工具时，会在响应中包含 FunctionCall 部分：

```typescript
// packages/core/src/core/turn.ts
private handlePendingFunctionCall(
  fnCall: FunctionCall,
): ServerGeminiStreamEvent | null {
  const callId =
    fnCall.id ??
    `${fnCall.name}-${Date.now()}-${Math.random().toString(16).slice(2)}`;
  const name = fnCall.name || 'undefined_tool_name';
  const args = (fnCall.args || {}) as Record<string, unknown>;

  const toolCallRequest: ToolCallRequestInfo = {
    callId,
    name,
    args,
    isClientInitiated: false,
    prompt_id: this.prompt_id,
  };

  this.pendingToolCalls.push(toolCallRequest);

  // 发出工具调用请求事件
  return { type: GeminiEventType.ToolCallRequest, value: toolCallRequest };
}
```

### 14.2.2 事件流处理

在 CLI 层，`useGeminiStream` hook 监听并处理这些事件：

```typescript
// packages/cli/src/ui/hooks/useGeminiStream.ts
case ServerGeminiEventType.ToolCallRequest:
  toolCallRequests.push(event.value);
  break;

// 流处理完成后，批量调度工具调用
if (toolCallRequests.length > 0) {
  scheduleToolCalls(toolCallRequests, signal);
}
```

## 14.3 CoreToolScheduler：工具调度核心

CoreToolScheduler 是工具执行的核心协调器，负责管理工具调用的完整生命周期。

### 14.3.1 工具调用状态机

工具调用经历以下状态转换：

```typescript
type Status = 
  | 'validating'        // 验证参数
  | 'awaiting_approval' // 等待用户确认
  | 'scheduled'         // 已调度，等待执行
  | 'executing'         // 执行中
  | 'success'          // 执行成功
  | 'error'            // 执行失败
  | 'cancelled';       // 已取消
```

### 14.3.2 调度流程实现

```typescript
// packages/core/src/core/coreToolScheduler.ts
async schedule(
  request: ToolCallRequestInfo | ToolCallRequestInfo[],
  signal: AbortSignal,
): Promise<void> {
  // 1. 防止并发调度
  if (this.isRunning()) {
    throw new Error(
      'Cannot schedule new tool calls while other tool calls are actively running',
    );
  }

  // 2. 创建工具调用对象
  const newToolCalls: ToolCall[] = requestsToProcess.map((reqInfo) => {
    const toolInstance = toolRegistry.getTool(reqInfo.name);
    if (!toolInstance) {
      return createErrorToolCall(reqInfo);
    }
    return {
      status: 'validating',
      request: reqInfo,
      tool: toolInstance,
      startTime: Date.now(),
    };
  });

  // 3. 验证和确认阶段
  for (const toolCall of newToolCalls) {
    if (this.config.getApprovalMode() === ApprovalMode.YOLO) {
      this.setStatusInternal(reqInfo.callId, 'scheduled');
    } else {
      const confirmationDetails = await toolInstance.shouldConfirmExecute(
        reqInfo.args,
        signal,
      );
      if (confirmationDetails) {
        this.setStatusInternal(callId, 'awaiting_approval', confirmationDetails);
      }
    }
  }

  // 4. 尝试执行已调度的工具
  this.attemptExecutionOfScheduledCalls(signal);
}
```

## 14.4 用户确认机制

### 14.4.1 确认类型

系统支持多种确认类型，每种类型有不同的 UI 展示：

```typescript
// packages/core/src/tools/tools.ts
type ToolCallConfirmationDetails =
  | ToolEditConfirmationDetails    // 文件编辑确认
  | ToolExecuteConfirmationDetails  // 命令执行确认
  | ToolMcpConfirmationDetails     // MCP 工具确认
  | ToolInfoConfirmationDetails;   // 信息获取确认
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

对于文件编辑操作，用户可以在确认前修改内容：

```typescript
// packages/cli/src/ui/components/messages/ToolConfirmationMessage.tsx
if (confirmationDetails.type === 'edit') {
  if (confirmationDetails.isModifying) {
    // 显示"正在修改"状态
    return <ModifyingIndicator />;
  }

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

## 14.5 工具执行实现

### 14.5.1 并行执行机制

当所有工具调用都处于终态或已调度状态时，系统会并行执行所有已调度的工具：

```typescript
// packages/core/src/core/coreToolScheduler.ts
private attemptExecutionOfScheduledCalls(signal: AbortSignal): void {
  const allCallsFinalOrScheduled = this.toolCalls.every(
    (call) =>
      call.status === 'scheduled' ||
      call.status === 'cancelled' ||
      call.status === 'success' ||
      call.status === 'error',
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
```

### 14.5.2 实时输出支持

支持实时输出的工具可以在执行过程中更新显示：

```typescript
const liveOutputCallback =
  scheduledCall.tool.canUpdateOutput && this.outputUpdateHandler
    ? (outputChunk: string) => {
        this.outputUpdateHandler(callId, outputChunk);
        // 更新工具调用的实时输出
        this.toolCalls = this.toolCalls.map((tc) =>
          tc.request.callId === callId && tc.status === 'executing'
            ? { ...tc, liveOutput: outputChunk }
            : tc,
        );
      }
    : undefined;

// 执行工具
await scheduledCall.tool.execute(
  scheduledCall.request.args,
  signal,
  liveOutputCallback,
);
```

## 14.6 结果处理与回传

### 14.6.1 结果格式转换

工具执行结果需要转换为 Gemini API 期望的 FunctionResponse 格式：

```typescript
// packages/core/src/core/coreToolScheduler.ts
export function convertToFunctionResponse(
  toolName: string,
  callId: string,
  llmContent: PartListUnion,
): PartListUnion {
  if (typeof llmContent === 'string') {
    return {
      functionResponse: {
        id: callId,
        name: toolName,
        response: { output: llmContent },
      },
    };
  }

  // 处理其他类型的响应内容
  if (Array.isArray(llmContent)) {
    const functionResponse = createFunctionResponsePart(
      callId,
      toolName,
      'Tool execution succeeded.',
    );
    return [functionResponse, ...llmContent];
  }

  // ... 其他情况的处理
}
```

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