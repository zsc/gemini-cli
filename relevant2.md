# Chapter 2: Gemini API 集成层 - 相关信息收集

## 需要调查的核心模块
- packages/core/src/core/geminiChat.ts - 主要的 Gemini 聊天接口
- packages/core/src/core/client.ts - API 客户端实现
- packages/core/src/core/geminiRequest.ts - 请求构建
- packages/core/src/config/config.ts - 配置管理
- packages/core/src/config/models.ts - 模型定义
- packages/core/src/core/contentGenerator.ts - 内容生成器
- packages/core/src/utils/generateContentResponseUtilities.ts - 响应处理工具
- packages/core/src/utils/retry.ts - 重试机制
- packages/core/src/utils/quotaErrorDetection.ts - 配额错误检测

## 需要了解的功能点
1. API 客户端初始化流程
2. 认证机制
3. 请求构建和参数处理
4. 流式响应处理
5. 错误处理和重试策略
6. 模型选择和配置
7. Token 限制管理

## 核心类和接口

### GeminiClient (client.ts)
- 主要的客户端类，管理与 Gemini API 的所有交互
- 职责：
  - 初始化聊天会话
  - 管理历史记录
  - 发送消息和处理流式响应
  - 压缩聊天历史
  - 工具管理
  - 错误处理和模型降级

### GeminiChat (geminiChat.ts)
- 聊天会话管理类
- 职责：
  - 维护对话历史
  - 发送消息（同步和流式）
  - 处理 API 响应
  - 历史记录验证和清理
  - 支持思考模式（thinking mode）

### ContentGenerator (contentGenerator.ts)
- 内容生成器接口和工厂
- 支持多种认证方式：
  - LOGIN_WITH_GOOGLE (OAuth)
  - USE_GEMINI (Gemini API key)
  - USE_VERTEX_AI (Vertex AI)
  - CLOUD_SHELL
- 根据认证类型创建不同的生成器实现

### 重试机制 (retry.ts)
- 实现指数退避重试策略
- 特殊处理：
  - 429 错误（速率限制）
  - 5xx 服务器错误
  - Pro 配额超限错误的即时降级
  - Retry-After 头部支持

## 关键流程

### 初始化流程
1. 创建 Config 实例
2. 创建 ContentGeneratorConfig
3. 初始化 ContentGenerator（根据认证类型）
4. 创建 GeminiClient
5. 初始化聊天会话

### 消息发送流程
1. 构建请求内容（包括历史记录）
2. 检查是否需要压缩历史
3. 添加 IDE 上下文（如果启用）
4. 循环检测
5. 发送请求到 API
6. 处理流式响应
7. 更新历史记录
8. 检查是否需要继续对话

### 错误处理和降级
- OAuth 用户在遇到配额错误时自动降级到 Flash 模型
- 支持用户确认的降级流程
- 重试机制与降级策略集成

## 模型和 Token 管理

### 默认模型 (models.ts)
- DEFAULT_GEMINI_MODEL: 'gemini-2.5-pro'
- DEFAULT_GEMINI_FLASH_MODEL: 'gemini-2.5-flash'
- DEFAULT_GEMINI_FLASH_LITE_MODEL: 'gemini-2.5-flash-lite'
- DEFAULT_GEMINI_EMBEDDING_MODEL: 'gemini-embedding-001'

### Token 限制 (tokenLimits.ts)
- 默认限制: 1,048,576 tokens
- gemini-1.5-pro: 2,097,152 tokens
- gemini-2.5 系列: 1,048,576 tokens
- gemini-2.0-flash-preview-image-generation: 32,000 tokens

### 历史压缩机制
- 触发阈值: 达到模型 token 限制的 70%
- 保留比例: 保留最近 30% 的历史
- 压缩过程:
  1. 找到压缩点（保留最近的对话）
  2. 使用专门的压缩 prompt 生成状态快照
  3. 用快照替换旧历史

## 响应处理工具 (generateContentResponseUtilities.ts)

### 主要功能
- getResponseText: 提取响应中的文本内容
- getFunctionCalls: 提取函数调用
- getStructuredResponse: 组合文本和函数调用的结构化响应

## 配额错误检测 (quotaErrorDetection.ts)

### 错误类型
- Pro 配额超限: 特定匹配 "Gemini X.X Pro Requests" 模式
- 通用配额超限: 包含 "Quota exceeded for quota metric"
- 支持多种错误格式（ApiError, StructuredError, Gaxios 错误）

## 系统提示词 (prompts.ts)

### 核心系统提示词
- 可通过 GEMINI_SYSTEM_MD 环境变量自定义
- 包含详细的行为准则和工作流程指导
- 支持用户记忆（GEMINI.md）的集成

### 压缩提示词
- 专门用于历史压缩的系统提示词
- 生成包含以下部分的 XML 状态快照：
  - overall_goal: 总体目标
  - key_knowledge: 关键知识点
  - file_system_state: 文件系统状态
  - recent_actions: 最近的操作

## 思考模式支持
- 检测模型是否支持思考模式（gemini-2.5 系列）
- 在配置中启用 thinkingConfig
- 处理思考内容的特殊逻辑（不计入历史）

## 日志和监控
- API 请求日志
- API 响应日志（包括耗时和 token 使用）
- 错误日志
- 支持遥测系统集成