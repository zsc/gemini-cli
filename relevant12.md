# Chapter 12: 启动与初始化流程 - 相关信息收集

## 需要调查的内容
1. CLI 入口点 (packages/cli/index.ts)
2. 主应用初始化 (gemini.tsx)
3. 配置加载过程
4. 认证流程
5. 工具发现和注册
6. MCP 服务器初始化
7. 环境检查和警告
8. 非交互式模式处理

## 收集的信息

### 1. CLI 入口点分析

From packages/cli/index.ts:
- 简单的入口脚本，使用 shebang `#!/usr/bin/env node`
- 导入并调用 main() 函数
- 捕获全局未处理的错误并退出

### 2. 主函数初始化流程 (gemini.tsx)

主要步骤：
1. **设置未处理拒绝处理器** - setupUnhandledRejectionHandler()
2. **清理检查点** - cleanupCheckpoints()
3. **加载设置** - loadSettings(workspaceRoot)
4. **解析命令行参数** - parseArguments()
5. **加载扩展** - loadExtensions(workspaceRoot)
6. **加载配置** - loadCliConfig()
7. **内存配置检查** - 可能需要重新启动以分配更多内存
8. **沙箱处理** - 如果需要，进入沙箱环境
9. **认证处理** - 特别是 OAuth 认证
10. **决定运行模式** - 交互式或非交互式

### 3. 配置系统初始化

#### 设置加载 (loadSettings)
从三个层级加载设置：
1. **系统设置** - /Library/Application Support/GeminiCli/settings.json (macOS)
2. **用户设置** - ~/.gemini/settings.json
3. **工作区设置** - .gemini/settings.json

设置合并顺序：系统 < 用户 < 工作区

#### 配置加载 (loadCliConfig)
创建 Config 对象，包含：
- 工具注册表初始化
- MCP 服务器配置
- 认证配置
- 遥测配置
- 文件过滤选项

### 4. 认证流程

支持的认证方式：
1. **LOGIN_WITH_GOOGLE** - OAuth 流程
2. **CLOUD_SHELL** - Google Cloud Shell 环境
3. **USE_GEMINI** - 使用 GEMINI_API_KEY
4. **USE_VERTEX_AI** - 使用 Vertex AI 配置

验证流程：
- 检查必需的环境变量
- OAuth 认证可能需要在沙箱前完成
- 非交互式模式有特殊的认证验证

### 5. 工具发现和注册

#### 核心工具注册
通过 createToolRegistry 注册内置工具：
- LSTool - 列出目录
- ReadFileTool - 读取文件
- GrepTool - 搜索内容
- GlobTool - 文件模式匹配
- EditTool - 编辑文件
- WriteFileTool - 写入文件
- WebFetchTool - 获取网页
- ShellTool - 执行命令
- MemoryTool - 管理记忆
- WebSearchTool - 网络搜索

#### 工具启用逻辑
- 检查 coreTools 配置
- 检查 excludeTools 配置
- 动态发现额外工具

### 6. Memory 系统初始化

#### GEMINI.md 文件发现
分层搜索策略：
1. **全局层** - ~/.gemini/GEMINI.md
2. **项目层** - 从当前目录向上搜索到项目根
3. **工作区层** - 当前目录的 .gemini/GEMINI.md
4. **扩展提供** - 扩展目录中的上下文文件

#### 内容加载和合并
- 处理导入语句 (@import)
- 合并多个 GEMINI.md 文件
- 统计文件数量供后续使用

### 7. MCP 服务器初始化

#### 服务器配置加载
- 从设置中读取 MCP 服务器配置
- 从扩展中合并 MCP 服务器
- 应用允许/排除列表过滤

#### MCP 工具注册
- 连接到配置的 MCP 服务器
- 发现并注册 MCP 提供的工具
- 处理 OAuth 认证（如需要）

### 8. 扩展系统

#### 扩展发现
从两个位置加载：
1. **工作区扩展** - .gemini/extensions/
2. **用户扩展** - ~/.gemini/extensions/

#### 扩展配置
每个扩展包含：
- gemini-extension.json 配置文件
- 可选的上下文文件（GEMINI.md）
- MCP 服务器配置
- 工具排除列表

### 9. 环境检查和启动警告

#### 系统警告
- 检查临时文件中的警告
- 用户自定义启动警告
- 配置文件解析错误

#### 沙箱环境
支持三种沙箱命令：
- Docker
- Podman
- sandbox-exec (macOS)

沙箱决策流程：
1. 检查是否已在沙箱中
2. 检查沙箱配置
3. 处理认证（沙箱可能影响 OAuth）
4. 启动沙箱容器或进程

### 10. 运行模式决策

#### 交互式模式条件
- 有 TTY 终端
- 没有通过 -p 提供的命令行输入
- 或使用了 --prompt-interactive 参数

#### 非交互式模式
触发条件：
- 通过 -p 参数提供输入
- 通过管道输入（stdin）
- 没有 TTY 终端

处理流程：
1. 初始化配置
2. 排除交互式工具（Shell、Edit、WriteFile）
3. 执行单次对话循环
4. 输出结果到 stdout

### 11. 内存管理

#### 自动内存配置
如果启用 autoConfigureMaxOldSpaceSize：
- 计算系统总内存的 50%
- 检查当前堆大小限制
- 必要时重新启动进程

#### 重启机制
使用环境变量 GEMINI_CLI_NO_RELAUNCH 防止无限重启

### 12. 遥测和日志

#### 遥测初始化
- 根据配置启用/禁用遥测
- 支持本地和 GCP 目标
- 记录会话开始事件

#### 错误处理
- 全局未处理拒绝处理器
- 配置文件错误提示
- 启动失败的优雅退出