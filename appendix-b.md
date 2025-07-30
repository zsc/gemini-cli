# 附录 B：配置选项完整列表

本附录提供 Gemini CLI 所有可用配置选项的详细说明和示例。配置可以通过命令行参数、环境变量或配置文件进行设置。

## 目录

1. [配置优先级](#配置优先级)
2. [命令行选项](#命令行选项)
3. [环境变量](#环境变量)
4. [配置文件](#配置文件)
5. [认证配置](#认证配置)
6. [模型配置](#模型配置)
7. [工具配置](#工具配置)
8. [MCP 服务器配置](#mcp-服务器配置)
9. [沙箱配置](#沙箱配置)
10. [遥测配置](#遥测配置)
11. [界面和体验配置](#界面和体验配置)
12. [文件过滤配置](#文件过滤配置)
13. [高级配置](#高级配置)
14. [配置示例](#配置示例)

## 配置优先级

配置选项按以下优先级顺序应用（从高到低）：

1. 命令行参数
2. 系统级配置文件
3. 工作区配置文件
4. 用户配置文件
5. 环境变量
6. 默认值

## 命令行选项

Gemini CLI 提供了丰富的命令行选项来控制其行为。所有选项都可以通过 `gemini --help` 查看。

### 基本选项

#### `--help, -h`
显示帮助信息并退出。

```bash
gemini --help
```

#### `--version, -v`
显示版本信息并退出。

```bash
gemini --version
```

### 模型和提示词选项

#### `--model, -m <model>`
指定要使用的 AI 模型。

- **默认值**: `gemini-2.5-pro` 或环境变量 `GEMINI_MODEL` 的值
- **示例**:
  ```bash
  gemini --model gemini-2.5-flash
  gemini -m gemini-2.5-flash-lite
  ```

#### `--prompt, -p <prompt>`
非交互模式下的提示词。提供此选项后，CLI 将执行提示词并退出。

- **示例**:
  ```bash
  gemini --prompt "分析当前目录的代码结构"
  gemini -p "生成 README.md 文件"
  ```

#### `--prompt-interactive, -i <prompt>`
执行提供的提示词，然后继续保持交互模式。

- **注意**: 不能与 `--prompt` 同时使用
- **示例**:
  ```bash
  gemini --prompt-interactive "加载项目上下文"
  gemini -i "分析代码库并等待进一步指令"
  ```

### 调试和开发选项

#### `--debug, -d`
启用调试模式，显示详细的日志信息。

- **默认值**: `false`
- **示例**:
  ```bash
  gemini --debug
  gemini -d
  ```

### 上下文管理选项

#### `--all-files, -a`
将所有文件包含在上下文中（而不是仅包含相关文件）。

- **默认值**: `false`
- **注意**: 可能会消耗大量 token
- **示例**:
  ```bash
  gemini --all-files
  gemini -a
  ```

#### `--show-memory-usage`
在状态栏显示内存使用情况。

- **默认值**: `false`
- **示例**:
  ```bash
  gemini --show-memory-usage
  ```

### 操作模式选项

#### `--yolo, -y`
YOLO 模式：自动接受所有操作，无需用户确认。

- **默认值**: `false`
- **警告**: 使用此选项时要特别小心，因为 AI 将自动执行所有建议的操作
- **示例**:
  ```bash
  gemini --yolo
  gemini -y
  ```

#### `--checkpointing, -c`
启用文件编辑的检查点功能，允许回滚更改。

- **默认值**: `false`
- **示例**:
  ```bash
  gemini --checkpointing
  gemini -c
  ```

### 沙箱选项

#### `--sandbox, -s`
在沙箱环境中运行命令，提供隔离的执行环境。

- **示例**:
  ```bash
  gemini --sandbox
  gemini -s
  ```

#### `--sandbox-image <image>`
指定沙箱使用的 Docker/Podman 镜像。

- **默认值**: `gcr.io/gemini-cli/gemini-cli:latest`
- **示例**:
  ```bash
  gemini --sandbox --sandbox-image my-custom-image:latest
  ```

### 扩展和 MCP 选项

#### `--extensions, -e <extensions...>`
指定要使用的扩展列表。如果不提供，将使用所有可用扩展。

- **示例**:
  ```bash
  gemini --extensions extension1 extension2
  gemini -e my-extension
  ```

#### `--list-extensions, -l`
列出所有可用的扩展并退出。

- **示例**:
  ```bash
  gemini --list-extensions
  gemini -l
  ```

#### `--allowed-mcp-server-names <names...>`
指定允许使用的 MCP 服务器名称列表。

- **示例**:
  ```bash
  gemini --allowed-mcp-server-names server1 server2
  ```

### 遥测选项

#### `--telemetry`
启用遥测数据收集。

- **注意**: 此标志控制是否发送遥测数据。其他 `--telemetry-*` 选项仅设置特定值，但不会自动启用遥测
- **示例**:
  ```bash
  gemini --telemetry
  ```

#### `--telemetry-target <target>`
设置遥测目标。

- **可选值**: `local`, `gcp`
- **示例**:
  ```bash
  gemini --telemetry --telemetry-target gcp
  ```

#### `--telemetry-otlp-endpoint <endpoint>`
设置 OpenTelemetry 协议端点。

- **覆盖**: 环境变量 `OTEL_EXPORTER_OTLP_ENDPOINT` 和配置文件设置
- **示例**:
  ```bash
  gemini --telemetry --telemetry-otlp-endpoint http://localhost:4317
  ```

#### `--telemetry-log-prompts`
启用或禁用遥测中的用户提示词记录。

- **示例**:
  ```bash
  gemini --telemetry --telemetry-log-prompts
  ```

#### `--telemetry-outfile <file>`
将所有遥测输出重定向到指定文件。

- **示例**:
  ```bash
  gemini --telemetry --telemetry-outfile telemetry.log
  ```

### 网络选项

#### `--proxy <proxy>`
设置 Gemini 客户端的代理。

- **格式**: `schema://user:password@host:port`
- **覆盖**: 环境变量 `HTTPS_PROXY`, `HTTP_PROXY` 等
- **示例**:
  ```bash
  gemini --proxy http://proxy.example.com:8080
  gemini --proxy socks5://user:pass@proxy.example.com:1080
  ```

### 实验性选项

#### `--experimental-acp`
启动 ACP（Agent Control Protocol）模式。

- **注意**: 这是实验性功能，可能会在未来版本中更改
- **示例**:
  ```bash
  gemini --experimental-acp
  ```

#### `--ide-mode`
在 IDE 模式下运行，与 VSCode 等编辑器集成。

- **注意**: 仅在 `TERM_PROGRAM=vscode` 且不在沙箱中时生效
- **示例**:
  ```bash
  gemini --ide-mode
  ```

### 已弃用选项

以下选项已被弃用，将在未来版本中移除：

- `--all_files`: 请使用 `--all-files`
- `--show_memory_usage`: 请使用 `--show-memory-usage`

## 环境变量

Gemini CLI 支持通过环境变量进行配置。环境变量可以在系统级别设置，也可以通过 `.env` 文件设置。

### 环境变量文件发现

CLI 会按以下顺序查找 `.env` 文件：

1. 从当前目录开始，向上搜索到根目录
2. 优先使用 `.gemini/.env` 而不是 `.env`
3. 如果没有找到，回退到 `~/.gemini/.env` 或 `~/.env`

### 认证相关

#### `GEMINI_API_KEY`
Gemini API 密钥，用于直接访问 Gemini API。

- **使用场景**: 使用 `gemini-api-key` 认证方式时必需
- **示例**:
  ```bash
  export GEMINI_API_KEY="your-api-key-here"
  ```

#### `GOOGLE_API_KEY`
Google API 密钥，用于 Vertex AI Express 模式。

- **使用场景**: 使用 Vertex AI 认证方式的 Express 模式
- **示例**:
  ```bash
  export GOOGLE_API_KEY="your-google-api-key"
  ```

#### `GOOGLE_CLOUD_PROJECT`
Google Cloud 项目 ID。

- **使用场景**: 使用 Vertex AI 认证方式时需要
- **特殊处理**: 在 Cloud Shell 中，除非在 `.env` 文件中明确设置，否则默认为 `cloudshell-gca`
- **示例**:
  ```bash
  export GOOGLE_CLOUD_PROJECT="my-gcp-project"
  ```

#### `GOOGLE_CLOUD_LOCATION`
Google Cloud 区域位置。

- **使用场景**: 使用 Vertex AI 认证方式时需要
- **示例**:
  ```bash
  export GOOGLE_CLOUD_LOCATION="us-central1"
  ```

#### `GOOGLE_GENAI_USE_VERTEXAI`
强制使用 Vertex AI 进行认证。

- **值**: `true` 启用
- **示例**:
  ```bash
  export GOOGLE_GENAI_USE_VERTEXAI="true"
  ```

#### `GOOGLE_GENAI_USE_GCA`
使用 Google Cloud 认证。

- **值**: `true` 启用
- **示例**:
  ```bash
  export GOOGLE_GENAI_USE_GCA="true"
  ```

#### `CLOUD_SHELL`
指示是否在 Google Cloud Shell 环境中运行。

- **值**: `true` 表示在 Cloud Shell 中
- **注意**: 通常由 Cloud Shell 环境自动设置

### 模型配置

#### `GEMINI_MODEL`
默认使用的 AI 模型。

- **默认值**: `gemini-2.5-pro`
- **覆盖**: 可被命令行参数 `--model` 覆盖
- **示例**:
  ```bash
  export GEMINI_MODEL="gemini-2.5-flash"
  ```

### 调试和开发

#### `DEBUG`
启用调试模式。

- **值**: `true` 或 `1` 启用调试
- **示例**:
  ```bash
  export DEBUG="true"
  ```

#### `DEBUG_MODE`
备用的调试模式标志。

- **值**: `true` 或 `1` 启用调试
- **示例**:
  ```bash
  export DEBUG_MODE="1"
  ```

#### `DEBUG_PORT`
调试服务器端口。

- **默认值**: `9229`
- **使用场景**: 在调试模式下运行时
- **示例**:
  ```bash
  export DEBUG_PORT="9230"
  ```

### 网络和代理

#### `HTTPS_PROXY` / `https_proxy`
HTTPS 代理服务器地址。

- **格式**: `http://proxy:port` 或 `http://user:pass@proxy:port`
- **优先级**: 命令行参数 `--proxy` 会覆盖此设置
- **示例**:
  ```bash
  export HTTPS_PROXY="http://proxy.company.com:8080"
  export https_proxy="http://proxy.company.com:8080"
  ```

#### `HTTP_PROXY` / `http_proxy`
HTTP 代理服务器地址。

- **格式**: 同 `HTTPS_PROXY`
- **注意**: 如果设置了 `HTTPS_PROXY`，通常会优先使用
- **示例**:
  ```bash
  export HTTP_PROXY="http://proxy.company.com:8080"
  export http_proxy="http://proxy.company.com:8080"
  ```

### 遥测

#### `OTEL_EXPORTER_OTLP_ENDPOINT`
OpenTelemetry 导出器端点。

- **默认值**: `http://localhost:4317`
- **覆盖**: 可被命令行参数或配置文件覆盖
- **示例**:
  ```bash
  export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4317"
  ```

#### `OTLP_GOOGLE_CLOUD_PROJECT`
用于遥测的 Google Cloud 项目。

- **使用场景**: 将遥测数据发送到 GCP 时
- **示例**:
  ```bash
  export OTLP_GOOGLE_CLOUD_PROJECT="telemetry-project"
  ```

### 沙箱环境

#### `GEMINI_SANDBOX`
指定沙箱命令。

- **可选值**: `docker`, `podman`, `sandbox-exec`
- **注意**: 如果设置为 `true`，系统会自动检测可用的容器运行时
- **示例**:
  ```bash
  export GEMINI_SANDBOX="docker"
  ```

#### `GEMINI_SANDBOX_IMAGE`
沙箱使用的容器镜像。

- **默认值**: `gcr.io/gemini-cli/gemini-cli:latest`
- **示例**:
  ```bash
  export GEMINI_SANDBOX_IMAGE="my-registry/my-image:tag"
  ```

#### `GEMINI_SANDBOX_IMAGE_TAG`
沙箱镜像标签。

- **示例**:
  ```bash
  export GEMINI_SANDBOX_IMAGE_TAG="v1.2.3"
  ```

#### `SANDBOX`
指示当前是否在沙箱内运行。

- **注意**: 由沙箱环境自动设置，不应手动设置
- **值**: 沙箱类型（如 `sandbox-exec`）

#### `SEATBELT_PROFILE`
macOS 沙箱配置文件。

- **使用场景**: 在 macOS 上使用 `sandbox-exec` 时
- **值**: `none` 禁用沙箱配置文件

### 用户界面和体验

#### `NO_BROWSER`
禁止自动打开浏览器。

- **使用场景**: OAuth 认证时防止自动打开浏览器，需要手动打开认证 URL
- **值**: 任何非空值都会禁用浏览器启动
- **提示**: 如果浏览器打开失败，CLI 会提示使用 `NO_BROWSER=true` 重新运行
- **示例**:
  ```bash
  export NO_BROWSER="1"
  # 或者一次性使用
  NO_BROWSER=true gemini
  ```

#### `TERM_PROGRAM`
终端程序标识。

- **使用场景**: 用于检测 IDE 模式（如 VSCode）
- **自动设置**: 通常由终端程序自动设置，无需手动配置
- **示例值**: `vscode`, `iTerm.app`, `Terminal`
- **重要**: `--ide-mode` 选项仅在 `TERM_PROGRAM=vscode` 时生效

### 系统配置

#### `GEMINI_CLI_SYSTEM_SETTINGS_PATH`
覆盖系统配置文件的默认路径。

- **默认值**:
  - macOS: `/Library/Application Support/GeminiCli/settings.json`
  - Windows: `C:\ProgramData\gemini-cli\settings.json`
  - Linux: `/etc/gemini-cli/settings.json`
- **示例**:
  ```bash
  export GEMINI_CLI_SYSTEM_SETTINGS_PATH="/custom/path/settings.json"
  ```

### 构建和开发环境

#### `BUILD_SANDBOX`
在构建过程中构建沙箱镜像。

- **值**: `1` 或 `true` 启用
- **使用场景**: 开发和 CI/CD 环境

#### `BUILD_SANDBOX_FLAGS`
构建沙箱时的额外标志。

- **示例**:
  ```bash
  export BUILD_SANDBOX_FLAGS="--no-cache"
  ```

#### `CLI_VERSION`
CLI 版本（构建时设置）。

#### `IS_NIGHTLY`
指示是否为夜间构建版本。

- **值**: `true` 表示夜间构建

#### `MANUAL_VERSION`
手动覆盖版本号。

- **使用场景**: 特殊构建或测试

### 测试环境

#### `IS_DOCKER`
指示是否在 Docker 容器中运行。

#### `VERBOSE`
启用详细输出。

- **使用场景**: 构建和测试脚本

#### `KEEP_OUTPUT`
保留测试输出文件。

- **值**: `true` 保留输出
- **使用场景**: 调试测试失败

### 环境变量在配置文件中的使用

在配置文件中，可以使用环境变量替换。这个功能在字符串、数组和嵌套对象中都可以使用：

```json
{
  "mcpServers": {
    "myServer": {
      "command": "$HOME/bin/mcp-server",
      "args": ["--config", "${CONFIG_DIR}/server.conf"],
      "env": {
        "API_KEY": "${MY_API_KEY}",
        "CONFIG_PATH": "$CONFIG_DIR/config.json",
        "DEBUG": "${DEBUG:-false}"  // 支持默认值语法
      }
    }
  },
  "proxy": "${HTTPS_PROXY}",
  "telemetry": {
    "enabled": "${TELEMETRY_ENABLED}",
    "target": "${TELEMETRY_TARGET:-local}"
  }
}
```

支持的语法：
- `$VAR_NAME`: 简单变量引用
- `${VAR_NAME}`: 带花括号的变量引用（推荐，避免歧义）
- 环境变量替换在配置加载时进行
- 未定义的变量会保留原始字符串

## 配置文件

Gemini CLI 支持通过 JSON 格式的配置文件进行设置。配置文件支持 JSON5 格式（允许注释和尾随逗号），并且支持环境变量替换。

### 配置文件位置

配置文件按以下层级组织：

1. **系统级配置** (`settings.json`)
   - macOS: `/Library/Application Support/GeminiCli/settings.json`
   - Windows: `C:\ProgramData\gemini-cli\settings.json`
   - Linux: `/etc/gemini-cli/settings.json`
   - 可通过 `GEMINI_CLI_SYSTEM_SETTINGS_PATH` 环境变量覆盖

2. **用户级配置** (`~/.gemini/settings.json`)
   - 适用于用户的全局设置

3. **工作区配置** (`<workspace>/.gemini/settings.json`)
   - 特定于项目的设置

### 配置文件结构

完整的配置文件示例：

```json
{
  // 主题设置
  "theme": "default",
  
  // 自定义主题定义
  "customThemes": {
    "myTheme": {
      "name": "My Custom Theme",
      "styles": {
        "user": "cyan",
        "assistant": "green",
        "tool": "yellow"
      }
    }
  },
  
  // 认证类型
  "selectedAuthType": "gemini-api-key",
  
  // 沙箱设置
  "sandbox": false,
  
  // 核心工具配置
  "coreTools": [
    "ReadFileTool",
    "WriteFileTool",
    "EditTool",
    "ShellTool"
  ],
  
  // 排除的工具
  "excludeTools": ["WebSearchTool"],
  
  // MCP 服务器配置
  "mcpServers": {
    "myServer": {
      "command": "/path/to/mcp-server",
      "args": ["--port", "3000"],
      "env": {
        "API_KEY": "${MY_API_KEY}"
      }
    }
  },
  
  // 允许的 MCP 服务器
  "allowMCPServers": ["myServer", "approvedServer"],
  
  // 排除的 MCP 服务器
  "excludeMCPServers": ["untrustedServer"],
  
  // 界面设置
  "showMemoryUsage": true,
  "hideWindowTitle": false,
  "hideTips": false,
  "hideBanner": false,
  "vimMode": false,
  
  // 上下文文件名
  "contextFileName": "GEMINI.md",
  
  // 辅助功能
  "accessibility": {
    "disableLoadingPhrases": false
  },
  
  // 遥测设置
  "telemetry": {
    "enabled": false,
    "target": "local",
    "otlpEndpoint": "http://localhost:4317",
    "logPrompts": true,
    "outfile": "/path/to/telemetry.log"
  },
  
  // 使用统计
  "usageStatisticsEnabled": true,
  
  // 首选编辑器
  "preferredEditor": "vim",
  
  // Bug 报告命令
  "bugCommand": {
    "urlTemplate": "https://github.com/myorg/myrepo/issues/new?title={title}&body={body}"
  },
  
  // 检查点设置
  "checkpointing": {
    "enabled": false
  },
  
  // 文件过滤
  "fileFiltering": {
    "respectGitIgnore": true,
    "respectGeminiIgnore": true,
    "enableRecursiveFileSearch": true
  },
  
  // 会话限制
  "maxSessionTurns": 100,
  
  // 工具输出摘要
  "summarizeToolOutput": {
    "ShellTool": {
      "tokenBudget": 2000
    }
  },
  
  // IDE 模式
  "ideMode": false,
  
  // 自动更新
  "disableAutoUpdate": false,
  
  // 内存发现
  "memoryDiscoveryMaxDirs": 100,
  
  // Node.js 内存设置
  "autoConfigureMaxOldSpaceSize": true
}
```

### 配置选项详解

#### 界面和主题

- **`theme`**: 界面主题名称
  - 内置主题: `default`, `default-light`, `atom-one-dark`, `dracula`, `github-dark`, `github-light`, 等
  - 可以引用 `customThemes` 中定义的自定义主题

- **`customThemes`**: 自定义主题定义
  - 键为主题 ID，值为主题配置对象
  - 支持自定义各种 UI 元素的颜色

- **`vimMode`**: 启用 Vim 键绑定模式
  - `true`: 启用 Vim 模式
  - `false`: 使用默认键绑定

- **`showMemoryUsage`**: 在状态栏显示内存使用情况
- **`hideWindowTitle`**: 隐藏窗口标题
- **`hideTips`**: 隐藏使用提示
- **`hideBanner`**: 隐藏启动横幅

#### 认证和安全

- **`selectedAuthType`**: 默认认证类型
  - `"oauth-personal"`: 使用 Google 账号登录
  - `"gemini-api-key"`: 使用 Gemini API 密钥
  - `"vertex-ai"`: 使用 Vertex AI
  - `"cloud-shell"`: Cloud Shell 认证

- **`sandbox`**: 沙箱模式设置
  - `true`: 启用沙箱
  - `false`: 禁用沙箱
  - `"docker"`: 使用 Docker 沙箱
  - `"podman"`: 使用 Podman 沙箱

#### 工具配置

- **`coreTools`**: 启用的核心工具列表
  - 如果未指定，默认启用所有核心工具
  - 可用工具: `LSTool`, `ReadFileTool`, `GrepTool`, `GlobTool`, `EditTool`, `WriteFileTool`, `WebFetchTool`, `ReadManyFilesTool`, `ShellTool`, `MemoryTool`, `WebSearchTool`

- **`excludeTools`**: 要排除的工具列表
  - 优先级高于 `coreTools`

- **`toolDiscoveryCommand`**: 自定义工具发现命令
- **`toolCallCommand`**: 自定义工具调用命令
- **`mcpServerCommand`**: MCP 服务器命令

#### MCP 服务器配置

- **`mcpServers`**: MCP 服务器配置映射
  - 每个服务器的配置选项：
    - `command`: 启动命令
    - `args`: 命令参数数组
    - `env`: 环境变量
    - `cwd`: 工作目录
    - `url`: SSE 传输的 URL
    - `httpUrl`: HTTP 传输的 URL
    - `tcp`: WebSocket 传输的地址
    - `timeout`: 超时时间（毫秒）
    - `trust`: 是否信任此服务器
    - `includeTools`: 仅包含的工具
    - `excludeTools`: 排除的工具
    - `oauth`: OAuth 配置

- **`allowMCPServers`**: 允许的 MCP 服务器白名单
- **`excludeMCPServers`**: 排除的 MCP 服务器黑名单

#### 文件和内存管理

- **`contextFileName`**: 上下文文件名
  - 默认: `"GEMINI.md"`
  - 可以是字符串或字符串数组

- **`fileFiltering`**: 文件过滤选项
  - `respectGitIgnore`: 是否遵守 .gitignore
  - `respectGeminiIgnore`: 是否遵守 .geminiignore
  - `enableRecursiveFileSearch`: 启用递归文件搜索

- **`memoryDiscoveryMaxDirs`**: 内存发现最大目录数
  - 限制向上搜索 GEMINI.md 文件的目录层数

#### 性能和限制

- **`maxSessionTurns`**: 最大会话轮数
  - `-1`: 无限制
  - 正整数: 限制对话轮数

- **`summarizeToolOutput`**: 工具输出摘要配置
  - 为每个工具配置 token 预算
  - 超过预算时自动摘要输出

- **`autoConfigureMaxOldSpaceSize`**: 自动配置 Node.js 内存限制
  - 自动设置 --max-old-space-size 参数

#### 开发和调试

- **`preferredEditor`**: 首选编辑器
  - 用于打开文件的默认编辑器

- **`bugCommand`**: Bug 报告配置
  - `urlTemplate`: URL 模板，支持 `{title}` 和 `{body}` 占位符

- **`checkpointing`**: 检查点配置
  - `enabled`: 是否启用文件编辑检查点

- **`ideMode`**: IDE 模式
  - 与 VSCode 等编辑器集成时使用

#### 遥测和统计

- **`telemetry`**: 遥测配置
  - `enabled`: 是否启用
  - `target`: 目标（`"local"` 或 `"gcp"`）
  - `otlpEndpoint`: OTLP 端点
  - `logPrompts`: 是否记录提示词
  - `outfile`: 输出文件路径

- **`usageStatisticsEnabled`**: 启用使用统计
  - 收集匿名使用数据

- **`disableAutoUpdate`**: 禁用自动更新检查

### 配置合并规则

1. 数组类型的配置会被完全覆盖，不会合并
2. 对象类型的配置会进行深度合并
3. `customThemes` 和 `mcpServers` 会进行特殊的合并处理
4. 环境变量替换在加载时进行

### 配置验证

- 配置文件支持 JSON5 格式（允许注释）
- 无效的配置项会被忽略
- 语法错误会导致配置文件被跳过
- 错误信息会在启动时显示

## 认证配置

Gemini CLI 支持多种认证方式，每种方式有不同的配置要求。

### 认证方式概览

1. **Gemini API Key** (`gemini-api-key`)
   - 最简单的认证方式
   - 适合个人开发和测试

2. **Google OAuth** (`oauth-personal`)
   - 使用 Google 账号登录
   - 无需管理 API 密钥

3. **Vertex AI** (`vertex-ai`)
   - 企业级解决方案
   - 支持两种模式：标准模式和 Express 模式

4. **Cloud Shell** (`cloud-shell`)
   - Google Cloud Shell 环境专用
   - 自动配置

### Gemini API Key 认证

#### 配置步骤

1. 获取 API 密钥：
   - 访问 [Google AI Studio](https://makersuite.google.com/app/apikey)
   - 创建新的 API 密钥

2. 设置环境变量：
   ```bash
   export GEMINI_API_KEY="your-api-key-here"
   ```

3. 或在 `.env` 文件中设置：
   ```
   GEMINI_API_KEY=your-api-key-here
   ```

4. 在配置文件中指定认证类型：
   ```json
   {
     "selectedAuthType": "gemini-api-key"
   }
   ```

#### 安全建议

- 不要将 API 密钥提交到版本控制
- 使用 `.gemini/.env` 而不是项目根目录的 `.env`
- 定期轮换 API 密钥

### Google OAuth 认证

#### 配置步骤

1. 首次运行时选择 "Login with Google"
2. 浏览器会自动打开进行授权
3. 授权后令牌会安全存储

#### 配置选项

```json
{
  "selectedAuthType": "oauth-personal"
}
```

#### 令牌存储位置

- macOS: `~/Library/Application Support/GeminiCli/`
- Linux: `~/.config/gemini-cli/`
- Windows: `%APPDATA%\gemini-cli\`

### Vertex AI 认证

#### 标准模式配置

需要设置 Google Cloud 项目和位置：

```bash
export GOOGLE_CLOUD_PROJECT="my-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
```

确保已安装并配置 gcloud CLI：

```bash
gcloud auth application-default login
```

#### Express 模式配置

使用 API 密钥而不是 OAuth：

```bash
export GOOGLE_API_KEY="your-google-api-key"
```

#### 配置文件设置

```json
{
  "selectedAuthType": "vertex-ai"
}
```

#### 强制使用 Vertex AI

即使设置了 `GEMINI_API_KEY`，也可以强制使用 Vertex AI：

```bash
export GOOGLE_GENAI_USE_VERTEXAI="true"
```

### Cloud Shell 认证

#### 自动检测

Cloud Shell 环境会自动检测并配置：

- 环境变量 `CLOUD_SHELL=true` 自动设置
- 默认项目设置为 `cloudshell-gca`

#### 覆盖默认项目

在 `.env` 文件中设置：

```
GOOGLE_CLOUD_PROJECT=my-actual-project
```

### 认证优先级

1. 命令行参数（如果支持）
2. 环境变量
3. 配置文件中的 `selectedAuthType`
4. 自动检测（Cloud Shell）

### 认证故障排查

#### Gemini API Key 问题

错误信息：
```
GEMINI_API_KEY environment variable not found
```

解决方法：
- 确认环境变量已设置
- 检查 `.env` 文件位置
- 验证 API 密钥有效性

#### Vertex AI 问题

错误信息：
```
When using Vertex AI, you must specify either:
• GOOGLE_CLOUD_PROJECT and GOOGLE_CLOUD_LOCATION environment variables.
• GOOGLE_API_KEY environment variable (if using express mode).
```

解决方法：
- 设置所需的环境变量
- 确认 gcloud 认证状态
- 检查项目权限

#### OAuth 问题

常见问题：
- 浏览器未自动打开：设置 `NO_BROWSER=1` 并手动打开 URL
- 令牌过期：重新运行认证流程
- 权限不足：检查 Google 账号权限

### 多环境配置

使用不同的配置文件管理多个环境：

1. 开发环境 (`.gemini/settings.dev.json`)：
   ```json
   {
     "selectedAuthType": "gemini-api-key"
   }
   ```

2. 生产环境 (`.gemini/settings.prod.json`)：
   ```json
   {
     "selectedAuthType": "vertex-ai"
   }
   ```

3. 通过环境变量切换：
   ```bash
   export GEMINI_CONFIG_FILE=".gemini/settings.prod.json"
   ```

## 模型配置

Gemini CLI 支持多种 Gemini 模型，可以根据需求选择不同的模型。

### 可用模型

#### 默认模型

- **`gemini-2.5-pro`**: 默认模型，性能和质量均衡
- **`gemini-2.5-flash`**: 快速响应模型，适合实时交互
- **`gemini-2.5-flash-lite`**: 轻量级模型，最快响应速度
- **`gemini-embedding-001`**: 嵌入模型，用于文本向量化

### 配置方式

#### 1. 命令行参数

```bash
# 使用快速模型
gemini --model gemini-2.5-flash

# 使用轻量级模型
gemini -m gemini-2.5-flash-lite
```

#### 2. 环境变量

```bash
export GEMINI_MODEL="gemini-2.5-flash"
```

#### 3. 配置文件

虽然配置文件中没有直接的 `model` 字段，但可以通过环境变量在配置文件中设置：

```json
{
  "mcpServers": {
    "myServer": {
      "env": {
        "MODEL": "${GEMINI_MODEL}"
      }
    }
  }
}
```

### 模型选择指南

#### gemini-2.5-pro

- **优点**：
  - 最佳的理解和推理能力
  - 适合复杂的代码生成和分析
  - 支持长上下文

- **缺点**：
  - 响应速度相对较慢
  - 成本较高

- **适用场景**：
  - 复杂的代码重构
  - 架构设计
  - 深度代码分析

#### gemini-2.5-flash

- **优点**：
  - 快速响应
  - 成本效益高
  - 仍保持良好的质量

- **缺点**：
  - 对于极复杂的任务可能不如 Pro 模型

- **适用场景**：
  - 日常代码编写
  - 快速原型开发
  - 交互式编程辅助

#### gemini-2.5-flash-lite

- **优点**：
  - 最快的响应速度
  - 最低的成本
  - 适合高频调用

- **缺点**：
  - 能力有限
  - 不适合复杂任务

- **适用场景**：
  - 简单的代码补全
  - 语法检查
  - 快速查询

### Flash Fallback 机制

CLI 支持自动回退机制，当遇到配额限制时可以自动切换到 Flash 模型：

1. 当 `gemini-2.5-pro` 遇到配额限制时，自动切换到 `gemini-2.5-flash`
2. 用户会收到提示，可以选择是否接受回退
3. 一旦接受，当前会话将继续使用 Flash 模型

### 模型限制

#### Token 限制

不同模型有不同的 token 限制：

- **输入 token 限制**：影响可以处理的上下文长度
- **输出 token 限制**：影响单次响应的最大长度
- **总 token 限制**：输入 + 输出的总和

#### 速率限制

- **每分钟请求数** (RPM)
- **每分钟 token 数** (TPM)
- **每日请求数** (RPD)

### 模型特定功能

#### 代码执行

所有主要模型都支持代码执行能力，但质量有所不同：

- `gemini-2.5-pro`: 最佳的代码理解和生成
- `gemini-2.5-flash`: 平衡的代码能力
- `gemini-2.5-flash-lite`: 基本的代码能力

#### 函数调用

所有模型都支持工具/函数调用，这是 Gemini CLI 的核心功能。

### 性能优化建议

1. **开发阶段**：使用 `gemini-2.5-flash` 进行快速迭代
2. **复杂任务**：切换到 `gemini-2.5-pro`
3. **批量处理**：考虑使用 `gemini-2.5-flash-lite`
4. **混合使用**：在同一会话中根据任务切换模型

### 模型迷移

当从其他 AI 工具迁移时：

- GPT-4 → `gemini-2.5-pro`
- GPT-3.5 → `gemini-2.5-flash`
- 轻量级需求 → `gemini-2.5-flash-lite`

## 工具配置

Gemini CLI 提供了丰富的内置工具，并支持通过多种方式扩展和自定义工具。

### 内置工具

#### 文件系统工具

- **`LSTool`**: 列出目录内容
  - 类似 `ls` 命令
  - 支持过滤和排序

- **`ReadFileTool`**: 读取文件内容
  - 支持大文件分段读取
  - 自动检测文件编码
  - 支持图片和 PDF 文件

- **`WriteFileTool`**: 写入文件
  - 创建或覆盖文件
  - 自动创建目录

- **`EditTool`**: 编辑文件
  - 基于字符串替换
  - 保持原有格式
  - 支持批量替换

#### 搜索工具

- **`GrepTool`**: 文本搜索
  - 基于 ripgrep
  - 支持正则表达式
  - 支持多种输出模式

- **`GlobTool`**: 文件名匹配
  - 支持 glob 模式
  - 快速文件发现

#### 系统工具

- **`ShellTool`**: 执行 shell 命令
  - 支持交互式确认
  - 持久化 shell 会话
  - 支持超时控制

#### 网络工具

- **`WebFetchTool`**: 获取网页内容
  - HTML 转 Markdown
  - AI 处理网页内容
  - 15 分钟缓存

- **`WebSearchTool`**: 网络搜索
  - 集成搜索引擎
  - 返回结构化结果

#### 特殊工具

- **`MemoryTool`**: 管理上下文内存
  - 保存和加载 GEMINI.md
  - 管理项目上下文

- **`ReadManyFilesTool`**: 批量读取文件
  - 优化的批处理
  - 并行读取

### 工具配置方式

#### 1. 启用特定工具

在配置文件中指定要启用的工具：

```json
{
  "coreTools": [
    "ReadFileTool",
    "WriteFileTool",
    "EditTool",
    "ShellTool",
    "GrepTool"
  ]
}
```

如果不指定 `coreTools`，默认启用所有工具。

#### 2. 排除特定工具

```json
{
  "excludeTools": [
    "WebSearchTool",
    "ShellTool"
  ]
}
```

`excludeTools` 的优先级高于 `coreTools`。

#### 3. 工具参数配置

某些工具支持通过配置传递参数：

```json
{
  "coreTools": [
    "ShellTool(timeout=60000)",
    "GrepTool(maxResults=100)"
  ]
}
```

### 自定义工具

#### 1. 工具发现命令

通过外部命令发现工具：

```json
{
  "toolDiscoveryCommand": "/path/to/tool-discovery-script"
}
```

脚本应返回 JSON 格式的工具描述：

```json
[
  {
    "name": "myTool",
    "description": "My custom tool",
    "inputSchema": {
      "type": "object",
      "properties": {
        "input": { "type": "string" }
      }
    }
  }
]
```

#### 2. 工具调用命令

指定如何调用自定义工具：

```json
{
  "toolCallCommand": "/path/to/tool-executor"
}
```

执行器会收到 JSON 格式的调用请求：

```json
{
  "tool": "myTool",
  "arguments": {
    "input": "example"
  }
}
```

### 工具输出摘要

当工具输出过长时，可以配置自动摘要：

```json
{
  "summarizeToolOutput": {
    "ShellTool": {
      "tokenBudget": 2000
    },
    "ReadFileTool": {
      "tokenBudget": 5000
    }
  }
}
```

- `tokenBudget`: 输出的最大 token 数
- 超过限制时会使用 AI 摘要

### 工具使用策略

#### 安全性考虑

1. **ShellTool 风险**
   - 默认需要用户确认
   - 可以通过 `--yolo` 模式跳过确认
   - 建议在沙箱中运行

2. **文件操作限制**
   - 遵守 .gitignore
   - 遵守 .geminiignore
   - 不会操作系统文件

3. **网络访问控制**
   - WebFetchTool 仅读取
   - 不会发送 POST 请求
   - 支持代理配置

#### 性能优化

1. **批量操作**
   - 使用 ReadManyFilesTool 而不是多次 ReadFileTool
   - 使用 MultiEdit 而不是多次 Edit

2. **搜索策略**
   - 先用 GlobTool 找文件
   - 再用 GrepTool 搜索内容
   - 避免使用 shell 的 find/grep

3. **缓存利用**
   - WebFetchTool 有 15 分钟缓存
   - 文件系统操作无缓存

### 工具组合示例

#### 最小化配置（只读）

```json
{
  "coreTools": [
    "ReadFileTool",
    "GrepTool",
    "GlobTool",
    "LSTool"
  ]
}
```

#### 开发配置

```json
{
  "coreTools": [
    "ReadFileTool",
    "WriteFileTool",
    "EditTool",
    "GrepTool",
    "GlobTool",
    "LSTool",
    "ShellTool",
    "MemoryTool"
  ],
  "excludeTools": [
    "WebSearchTool"  // 避免意外的网络访问
  ]
}
```

#### 完整功能配置

```json
{
  // 不指定 coreTools，启用所有
  "excludeTools": [],
  "summarizeToolOutput": {
    "ShellTool": {
      "tokenBudget": 3000
    },
    "ReadFileTool": {
      "tokenBudget": 5000
    },
    "WebFetchTool": {
      "tokenBudget": 2000
    }
  }
}
```

### 工具故障排查

#### 工具未找到

- 检查工具名称拼写
- 确认未被 `excludeTools` 排除
- 检查 `coreTools` 列表

#### 工具执行失败

- 查看详细错误信息
- 检查权限问题
- 验证参数格式

#### 性能问题

- 配置输出摘要
- 减少并发工具调用
- 使用更高效的工具组合

## MCP 服务器配置

Model Context Protocol (MCP) 允许 Gemini CLI 与外部服务和工具集成。

### MCP 概述

MCP 是一个标准化协议，用于：
- 扩展 AI 助手的能力
- 提供对外部数据和服务的访问
- 实现自定义工具和集成

### 配置 MCP 服务器

#### 基本配置结构

```json
{
  "mcpServers": {
    "serverName": {
      "command": "/path/to/mcp-server",
      "args": ["--port", "3000"],
      "env": {
        "API_KEY": "${MY_API_KEY}"
      }
    }
  }
}
```

#### 完整配置选项

```json
{
  "mcpServers": {
    "myServer": {
      // 基本配置
      "command": "/usr/local/bin/mcp-server",
      "args": ["--verbose"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}",
        "API_TOKEN": "${API_TOKEN}"
      },
      "cwd": "/path/to/working/directory",
      
      // 传输配置（选择其一）
      "url": "http://localhost:3000/sse",     // SSE 传输
      "httpUrl": "http://localhost:3001",     // HTTP 传输
      "tcp": "localhost:3002",                // WebSocket 传输
      
      // 高级选项
      "timeout": 30000,                        // 超时（毫秒）
      "trust": true,                           // 信任此服务器
      "description": "我的自定义 MCP 服务器",
      
      // 工具过滤
      "includeTools": ["tool1", "tool2"],     // 仅包含这些工具
      "excludeTools": ["dangerous_tool"],     // 排除这些工具
      
      // OAuth 配置
      "oauth": {
        "provider": "google",
        "clientId": "${OAUTH_CLIENT_ID}",
        "scopes": ["read", "write"]
      },
      
      // 认证提供者类型
      "authProviderType": "dynamic_discovery"
    }
  }
}
```

### 传输类型

#### 1. stdio 传输（默认）

最常见的传输方式，通过标准输入/输出通信：

```json
{
  "mcpServers": {
    "stdio-server": {
      "command": "python",
      "args": ["-m", "my_mcp_server"]
    }
  }
}
```

#### 2. SSE (Server-Sent Events) 传输

适合长连接和实时更新：

```json
{
  "mcpServers": {
    "sse-server": {
      "url": "http://localhost:8080/events",
      "headers": {
        "Authorization": "Bearer ${API_TOKEN}"
      }
    }
  }
}
```

#### 3. HTTP 传输

标准 RESTful API 通信：

```json
{
  "mcpServers": {
    "http-server": {
      "httpUrl": "https://api.example.com/mcp",
      "headers": {
        "X-API-Key": "${API_KEY}"
      }
    }
  }
}
```

#### 4. WebSocket 传输

双向实时通信：

```json
{
  "mcpServers": {
    "ws-server": {
      "tcp": "ws://localhost:8080/socket"
    }
  }
}
```

### 服务器访问控制

#### 允许列表

只允许特定的 MCP 服务器：

```json
{
  "allowMCPServers": ["trustedServer1", "trustedServer2"]
}
```

#### 排除列表

禁用特定的 MCP 服务器：

```json
{
  "excludeMCPServers": ["untrustedServer"]
}
```

#### 命令行控制

```bash
# 只允许特定服务器
gemini --allowed-mcp-server-names server1 server2
```

### OAuth 集成

#### Google OAuth 配置

```json
{
  "mcpServers": {
    "google-drive-server": {
      "command": "mcp-google-drive",
      "oauth": {
        "provider": "google",
        "clientId": "${GOOGLE_CLIENT_ID}",
        "clientSecret": "${GOOGLE_CLIENT_SECRET}",
        "scopes": [
          "https://www.googleapis.com/auth/drive.readonly"
        ]
      },
      "authProviderType": "google_credentials"
    }
  }
}
```

#### 动态发现

```json
{
  "mcpServers": {
    "dynamic-server": {
      "command": "mcp-dynamic",
      "authProviderType": "dynamic_discovery"
    }
  }
}
```

### 扩展集成

扩展可以提供 MCP 服务器配置：

```json
// 扩展的 package.json
{
  "gemini-cli": {
    "mcpServers": {
      "extension-server": {
        "command": "${extensionPath}/bin/server",
        "description": "扩展提供的 MCP 服务器"
      }
    }
  }
}
```

### MCP 提示词

MCP 服务器可以提供提示词模板：

```bash
# 使用 MCP 提供的提示词
gemini --prompt "@server/prompt-name"
```

### 常见 MCP 服务器示例

#### 数据库访问

```json
{
  "mcpServers": {
    "postgres": {
      "command": "mcp-postgres",
      "env": {
        "DATABASE_URL": "postgresql://user:pass@localhost/db"
      },
      "includeTools": ["query", "schema"]
    }
  }
}
```

#### 文件系统扩展

```json
{
  "mcpServers": {
    "fs-extra": {
      "command": "mcp-fs-extra",
      "args": ["--root", "/safe/directory"],
      "trust": true
    }
  }
}
```

#### API 网关

```json
{
  "mcpServers": {
    "api-gateway": {
      "httpUrl": "https://api.company.com/mcp",
      "headers": {
        "Authorization": "Bearer ${COMPANY_API_TOKEN}"
      },
      "timeout": 60000
    }
  }
}
```

### 故障排查

#### 服务器未启动

- 检查命令路径是否正确
- 验证环境变量是否设置
- 查看服务器日志输出

#### 连接超时

- 增加 `timeout` 值
- 检查网络连接
- 验证服务器是否正在运行

#### 工具未显示

- 检查 `includeTools`/`excludeTools` 配置
- 验证服务器是否正确暴露工具
- 使用 `--mcp-server-command` 调试

## 沙箱配置

沙箱提供了一个隔离的执行环境，确保 AI 生成的代码和命令在安全的环境中运行。

### 沙箱类型

Gemini CLI 支持三种沙箱实现：

1. **Docker** - 最常用，跨平台支持
2. **Podman** - 无需 root 权限的容器方案
3. **sandbox-exec** - macOS 原生沙箱

### 启用沙箱

#### 命令行方式

```bash
# 自动检测可用的沙箱
gemini --sandbox

# 指定沙箱类型
export GEMINI_SANDBOX="docker"
gemini --sandbox

# 使用自定义镜像
gemini --sandbox --sandbox-image my-image:latest
```

#### 配置文件方式

```json
{
  "sandbox": true,  // 或指定类型: "docker", "podman"
  "sandboxImage": "gcr.io/gemini-cli/gemini-cli:latest"
}
```

#### 环境变量方式

```bash
export GEMINI_SANDBOX="docker"
export GEMINI_SANDBOX_IMAGE="my-custom-image:v1.0"
```

### Docker 沙箱配置

#### 默认配置

- 镜像: `gcr.io/gemini-cli/gemini-cli:latest`
- 挂载当前目录为工作目录
- 网络访问受限
- 无特权模式

#### 自定义 Docker 镜像

创建自定义 Dockerfile：

```dockerfile
FROM gcr.io/gemini-cli/gemini-cli:latest

# 安装额外工具
RUN apt-get update && apt-get install -y \
    postgresql-client \
    redis-tools

# 添加自定义配置
COPY custom-config /etc/gemini/
```

构建并使用：

```bash
docker build -t my-gemini-sandbox .
gemini --sandbox --sandbox-image my-gemini-sandbox
```

#### Docker 环境变量

```bash
# 构建沙箱镜像时的标志
export BUILD_SANDBOX_FLAGS="--no-cache --platform linux/amd64"

# 自定义镜像标签
export GEMINI_SANDBOX_IMAGE_TAG="v2.0"
```

### Podman 沙箱配置

Podman 提供无根容器支持：

```bash
# 安装 Podman
sudo apt-get install podman  # Debian/Ubuntu
brew install podman          # macOS

# 使用 Podman 沙箱
export GEMINI_SANDBOX="podman"
gemini --sandbox
```

#### Podman 特定配置

```json
{
  "sandbox": "podman",
  "sandboxImage": "localhost/my-gemini:latest"
}
```

### macOS sandbox-exec

使用 macOS 原生沙箱技术：

```bash
export GEMINI_SANDBOX="sandbox-exec"
gemini --sandbox
```

#### 沙箱配置文件

Gemini CLI 提供多种预定义的沙箱配置：

- `sandbox-macos-permissive-open.sb` - 宽松，开放网络
- `sandbox-macos-permissive-closed.sb` - 宽松，无网络
- `sandbox-macos-restrictive-open.sb` - 严格，开放网络
- `sandbox-macos-restrictive-closed.sb` - 严格，无网络

#### 禁用 Seatbelt

```bash
export SEATBELT_PROFILE="none"
gemini --sandbox
```

### 沙箱内的限制

#### 文件系统访问

- 只能访问当前目录及子目录
- 无法访问系统目录
- 无法修改沙箱配置

#### 网络访问

取决于沙箱配置：
- 默认允许出站连接
- 可配置完全隔离
- 不允许监听端口

#### 进程限制

- 不能启动特权进程
- 资源使用受限
- 不能访问主机进程

### 沙箱自动检测

当设置 `--sandbox` 但未指定类型时，CLI 会按以下顺序检测：

1. 检查 `GEMINI_SANDBOX` 环境变量
2. 检测 Docker 是否可用
3. 检测 Podman 是否可用  
4. 在 macOS 上尝试 sandbox-exec
5. 如果都不可用，报错退出

### 沙箱内环境

#### 环境标识

在沙箱内运行时，会设置环境变量：

```bash
SANDBOX="docker"  # 或 "podman", "sandbox-exec"
```

#### 预装工具

默认沙箱镜像包含：
- Node.js 运行时
- 基本开发工具（git, curl, wget）
- 文本编辑器（vim, nano）
- 构建工具（make, gcc）

### 性能考虑

#### 启动时间

- Docker/Podman: 首次运行需要拉取镜像
- sandbox-exec: 即时启动，无需准备

#### 资源使用

- 内存限制: 默认继承主机限制
- CPU 限制: 无默认限制
- 磁盘空间: 仅当前目录

### 开发和调试

#### 构建本地沙箱

```bash
# 启用沙箱构建
export BUILD_SANDBOX="true"
npm run build

# 使用详细输出
export VERBOSE="true"
```

#### 调试沙箱问题

```bash
# 检查沙箱环境
gemini --sandbox --prompt "echo $SANDBOX && pwd && ls -la"

# 测试网络连接
gemini --sandbox --prompt "curl -I https://google.com"
```

### 安全最佳实践

1. **始终使用沙箱** - 特别是运行不信任的代码时
2. **定期更新镜像** - 获取安全补丁
3. **最小权限原则** - 使用最严格的配置
4. **审查输出** - 即使在沙箱中也要谨慎

### 故障排查

#### Docker 未找到

```
ERROR: install docker or podman or specify command in GEMINI_SANDBOX
```

解决方法：
- 安装 Docker Desktop
- 或安装 Podman
- 或在 macOS 上使用 sandbox-exec

#### 镜像拉取失败

- 检查网络连接
- 验证镜像名称
- 尝试手动拉取：`docker pull <image>`

#### 权限错误

- 确保 Docker daemon 正在运行
- 检查用户是否在 docker 组中
- 考虑使用 Podman（无需 root）

## 遥测配置

遥测功能帮助改进 Gemini CLI，通过收集使用数据来识别问题和优化体验。

### 遥测概述

遥测数据包括：
- 使用的命令和功能
- 性能指标
- 错误和崩溃报告
- 系统环境信息

不包括：
- 个人身份信息（除非明确配置）
- 文件内容
- 敏感数据

### 启用和禁用遥测

#### 通过命令行

```bash
# 启用遥测
gemini --telemetry

# 使用时明确启用
gemini --telemetry --telemetry-log-prompts
```

#### 通过配置文件

```json
{
  "telemetry": {
    "enabled": true,
    "target": "local",
    "logPrompts": false
  }
}
```

#### 通过环境变量

```bash
# 禁用遥测
export GEMINI_TELEMETRY_ENABLED="false"
```

### 遥测目标

#### 本地目标（默认）

将遥测数据发送到本地 OpenTelemetry 收集器：

```json
{
  "telemetry": {
    "enabled": true,
    "target": "local",
    "otlpEndpoint": "http://localhost:4317"
  }
}
```

#### GCP 目标

将数据发送到 Google Cloud：

```json
{
  "telemetry": {
    "enabled": true,
    "target": "gcp"
  }
}
```

需要设置：
```bash
export OTLP_GOOGLE_CLOUD_PROJECT="my-project-id"
```

### OpenTelemetry 配置

#### OTLP 端点

配置自定义 OTLP 端点：

```bash
# 环境变量方式
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4317"

# 命令行方式
gemini --telemetry --telemetry-otlp-endpoint http://custom:4317

# 配置文件方式
{
  "telemetry": {
    "otlpEndpoint": "http://custom:4317"
  }
}
```

#### 配置优先级

1. 命令行参数
2. 环境变量
3. 配置文件
4. 默认值 (`http://localhost:4317`)

### 提示词记录

控制是否记录用户输入的提示词：

```json
{
  "telemetry": {
    "enabled": true,
    "logPrompts": true  // 默认 true
  }
}
```

禁用提示词记录：
```bash
gemini --telemetry --telemetry-log-prompts=false
```

### 文件输出

将遥测数据保存到文件而不是发送：

```bash
# 命令行方式
gemini --telemetry --telemetry-outfile telemetry.log

# 配置文件方式
{
  "telemetry": {
    "enabled": true,
    "outfile": "/path/to/telemetry.log"
  }
}
```

### 使用统计

独立于遥测的使用统计功能：

```json
{
  "usageStatisticsEnabled": true  // 默认 true
}
```

使用统计包括：
- 会话开始/结束
- 使用的工具
- 命令频率
- 基本性能指标

### 遥测数据类型

#### 跟踪数据 (Traces)

记录操作的执行路径：
- API 调用链路
- 工具执行顺序
- 错误传播路径

#### 指标数据 (Metrics)

性能和使用指标：
- 响应时间
- Token 使用量
- 内存使用
- 错误率

#### 日志数据 (Logs)

结构化日志事件：
- 错误信息
- 警告
- 调试信息（如果启用）

### 隐私设置

#### 数据最小化

```json
{
  "telemetry": {
    "enabled": true,
    "logPrompts": false,
    "minimizeData": true
  }
}
```

#### 数据脱敏

自动脱敏的数据：
- 文件路径中的用户名
- API 密钥和令牌
- 个人识别信息

### 本地收集器设置

#### 使用 Docker 运行收集器

```yaml
# docker-compose.yml
version: '3'
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib
    ports:
      - "4317:4317"  # OTLP gRPC
      - "4318:4318"  # OTLP HTTP
    volumes:
      - ./otel-config.yaml:/etc/otelcol-contrib/config.yaml
```

#### 收集器配置示例

```yaml
# otel-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  file:
    path: /tmp/gemini-telemetry.json
  
  prometheus:
    endpoint: "0.0.0.0:8889"

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [file]
    metrics:
      receivers: [otlp]
      exporters: [prometheus]
```

### 开发环境遥测

#### 本地遥测脚本

```bash
# 启动本地遥测服务器
npm run telemetry:local

# 查看遥测数据
npm run telemetry:view
```

#### 调试遥测

```bash
# 启用详细日志
export OTEL_LOG_LEVEL=debug
gemini --telemetry
```

### 企业环境配置

#### 代理支持

遥测通过配置的代理发送：

```bash
export HTTPS_PROXY="http://corporate-proxy:8080"
gemini --telemetry
```

#### 自定义证书

```bash
export NODE_EXTRA_CA_CERTS="/path/to/ca-cert.pem"
```

### 故障排查

#### 遥测未发送

检查：
1. `--telemetry` 标志是否设置
2. 端点是否可达
3. 防火墙规则
4. 代理配置

#### 性能影响

如果遥测影响性能：
1. 使用文件输出代替网络
2. 降低采样率
3. 禁用详细跟踪

#### 查看本地数据

```bash
# 如果使用文件输出
tail -f /path/to/telemetry.log | jq .

# 如果使用本地收集器
curl http://localhost:8889/metrics
```

### 合规性

#### GDPR 合规

- 用户明确同意后才启用
- 可随时禁用
- 不收集个人数据
- 支持数据删除请求

#### 数据保留

- 本地：由用户控制
- GCP：遵循 Google 数据保留政策
- 文件输出：无自动删除

## 界面和体验配置

Gemini CLI 提供了丰富的界面自定义选项，以适应不同用户的喜好和需求。

### 主题配置

#### 内置主题

```json
{
  "theme": "default"  // 可选值见下文
}
```

可用的内置主题：
- `default` - 默认深色主题
- `default-light` - 默认浅色主题  
- `ansi` - ANSI 标准色
- `ansi-light` - ANSI 浅色
- `atom-one-dark` - Atom One Dark 主题
- `ayu` - Ayu 深色
- `ayu-light` - Ayu 浅色
- `dracula` - Dracula 主题
- `github-dark` - GitHub 深色
- `github-light` - GitHub 浅色
- `googlecode` - Google Code 风格
- `no-color` - 无颜色（纯文本）
- `shades-of-purple` - Shades of Purple
- `xcode` - Xcode 风格

#### 自定义主题

创建自定义主题：

```json
{
  "customThemes": {
    "myTheme": {
      "name": "My Custom Theme",
      "base": "dark",  // 或 "light"
      "colors": {
        // 文本颜色
        "text": "#ffffff",
        "textSecondary": "#999999",
        "textMuted": "#666666",
        
        // 角色颜色
        "user": "#00ff00",
        "assistant": "#00ffff",
        "system": "#ffff00",
        "tool": "#ff00ff",
        
        // UI 元素
        "border": "#333333",
        "background": "#000000",
        "backgroundSecondary": "#111111",
        
        // 状态颜色
        "success": "#00ff00",
        "error": "#ff0000",
        "warning": "#ffff00",
        "info": "#00ffff",
        
        // 代码高亮
        "keyword": "#ff79c6",
        "string": "#f1fa8c",
        "number": "#bd93f9",
        "comment": "#6272a4",
        "function": "#50fa7b"
      }
    }
  },
  "theme": "myTheme"  // 使用自定义主题
}
```

### Vim 模式

启用 Vim 键绑定：

```json
{
  "vimMode": true
}
```

支持的 Vim 操作：
- 模式：正常、插入、视觉
- 移动：h, j, k, l, w, b, 0, $, gg, G
- 编辑：i, a, o, O, x, dd, yy, p
- 搜索：/, ?, n, N
- 其他：u (撤销), Ctrl+R (重做)

### 显示选项

#### 内存使用显示

```json
{
  "showMemoryUsage": true
}
```

在状态栏显示：
- 当前上下文使用的 token 数
- 占模型限制的百分比
- 内存文件数量

#### 界面元素控制

```json
{
  "hideWindowTitle": false,  // 隐藏窗口标题
  "hideTips": false,         // 隐藏使用提示
  "hideBanner": false        // 隐藏启动横幅
}
```

### 辅助功能配置

```json
{
  "accessibility": {
    "disableLoadingPhrases": true  // 禁用动态加载提示
  }
}
```

适用于：
- 屏幕阅读器用户
- 远程连接低带宽环境
- 自动化脚本

### 编辑器集成

#### 首选编辑器

```json
{
  "preferredEditor": "code"  // 或 "vim", "nano", "emacs" 等
}
```

当 AI 需要打开文件编辑时使用。

#### IDE 模式

```json
{
  "ideMode": true
}
```

或通过命令行：
```bash
gemini --ide-mode
```

注意：仅在 VSCode 终端中生效（`TERM_PROGRAM=vscode`）。

### 交互体验

#### 会话限制

```json
{
  "maxSessionTurns": 100  // -1 表示无限制
}
```

限制单个会话的对话轮数，防止无限循环。

#### 自动更新

```json
{
  "disableAutoUpdate": false
}
```

控制是否自动检查更新。

### 命令行完成

CLI 支持智能补全：
- Tab 键补全命令
- 参数提示
- 历史命令搜索

### 快捷键

#### 默认快捷键

- `Ctrl+C`: 中断当前操作
- `Ctrl+D`: 退出 CLI
- `Ctrl+L`: 清屏
- `↑`/`↓`: 浏览历史
- `Tab`: 自动补全
- `Ctrl+R`: 搜索历史

#### Vim 模式快捷键

- `Esc`: 返回正常模式
- `i`: 进入插入模式
- `v`: 进入视觉模式
- `:`: 命令模式

### 提示和帮助

#### 使用提示

默认显示有用的提示，可以禁用：

```json
{
  "hideTips": true
}
```

#### 启动横幅

控制启动时的 ASCII 艺术横幅：

```json
{
  "hideBanner": true
}
```

### 国际化

虽然 CLI 主要使用英文，但支持：
- UTF-8 编码
- 多语言输入输出
- 本地化时间格式

### 性能优化

#### Node.js 内存配置

```json
{
  "autoConfigureMaxOldSpaceSize": true
}
```

自动调整 Node.js 堆内存限制，适应大型项目。

### 示例配置

#### 最小化界面

```json
{
  "theme": "no-color",
  "hideBanner": true,
  "hideTips": true,
  "hideWindowTitle": true,
  "accessibility": {
    "disableLoadingPhrases": true
  }
}
```

#### 丰富体验

```json
{
  "theme": "dracula",
  "vimMode": true,
  "showMemoryUsage": true,
  "preferredEditor": "code",
  "maxSessionTurns": -1
}
```

#### 辅助功能优化

```json
{
  "theme": "default-light",
  "accessibility": {
    "disableLoadingPhrases": true
  },
  "hideWindowTitle": true,
  "vimMode": false
}
```

## 文件过滤配置

文件过滤配置控制 Gemini CLI 如何处理和访问文件，确保安全性和效率。

### 基本配置

```json
{
  "fileFiltering": {
    "respectGitIgnore": true,
    "respectGeminiIgnore": true,
    "enableRecursiveFileSearch": true
  }
}
```

### Git 忽略文件

#### respectGitIgnore

控制是否遵守 `.gitignore` 规则：

```json
{
  "fileFiltering": {
    "respectGitIgnore": true  // 默认值
  }
}
```

当启用时：
- 自动忽略 `.gitignore` 中列出的文件
- 支持嵌套的 `.gitignore` 文件
- 遵循 Git 的标准规则

### Gemini 忽略文件

#### respectGeminiIgnore

控制是否遵守 `.geminiignore` 规则：

```json
{
  "fileFiltering": {
    "respectGeminiIgnore": true  // 默认值
  }
}
```

#### .geminiignore 文件格式

与 `.gitignore` 格式相同：

```gitignore
# 忽略所有日志文件
*.log

# 忽略特定目录
node_modules/
dist/
build/

# 忽略敏感文件
.env
*.key
*.pem

# 但保留特定文件
!important.log
```

### 递归文件搜索

#### enableRecursiveFileSearch

控制是否启用递归搜索：

```json
{
  "fileFiltering": {
    "enableRecursiveFileSearch": true  // 默认值
  }
}
```

当禁用时：
- 仅搜索当前目录
- 不进入子目录
- 提高大型目录的性能

### 内存文件特殊规则

对于 GEMINI.md 等内存文件，有不同的默认规则：

```javascript
// 内存文件默认值
DEFAULT_MEMORY_FILE_FILTERING_OPTIONS = {
  respectGitIgnore: false,      // 不忽略 git 规则
  respectGeminiIgnore: true     // 但遵守 gemini 规则
}

// 普通文件默认值
DEFAULT_FILE_FILTERING_OPTIONS = {
  respectGitIgnore: true,
  respectGeminiIgnore: true
}
```

### 上下文文件名配置

#### contextFileName

指定要查找的上下文文件名：

```json
{
  "contextFileName": "GEMINI.md"  // 默认值
}
```

支持多个文件名：

```json
{
  "contextFileName": [
    "GEMINI.md",
    "CONTEXT.md",
    ".gemini-context"
  ]
}
```

### 内存发现限制

#### memoryDiscoveryMaxDirs

限制向上搜索 GEMINI.md 的目录层数：

```json
{
  "memoryDiscoveryMaxDirs": 100  // 默认值
}
```

防止在深层目录结构中无限搜索。

### 文件类型过滤

虽然不是直接配置，但工具支持文件类型过滤：

```javascript
// 使用 GrepTool 时
grep --type js "pattern"  // 仅搜索 JavaScript 文件

// 使用 GlobTool 时
glob "**/*.{js,ts}"      // 仅匹配 JS/TS 文件
```

### 安全限制

无论配置如何，以下文件始终被保护：

1. **系统文件**
   - `/etc/*`
   - `/usr/*`
   - `/System/*` (macOS)
   - `C:\Windows\*` (Windows)

2. **敏感文件**
   - SSH 密钥 (`~/.ssh/*`)
   - 配置文件 (`~/.config/*`)
   - 证书文件 (`*.pem`, `*.key`)

3. **大文件**
   - 二进制文件通常被跳过
   - 超大文本文件会被截断

### 性能优化建议

#### 大型项目

```json
{
  "fileFiltering": {
    "respectGitIgnore": true,
    "respectGeminiIgnore": true,
    "enableRecursiveFileSearch": false  // 禁用递归
  },
  "memoryDiscoveryMaxDirs": 10  // 减少搜索范围
}
```

#### 小型项目

```json
{
  "fileFiltering": {
    "respectGitIgnore": false,  // 包含所有文件
    "respectGeminiIgnore": true,
    "enableRecursiveFileSearch": true
  }
}
```

### 常见用例

#### 排除测试文件

创建 `.geminiignore`：

```gitignore
# 排除所有测试文件
**/*.test.js
**/*.spec.ts
__tests__/
test/
tests/
```

#### 仅包含源代码

```gitignore
# 排除一切
*

# 但包含源代码
!src/
!src/**/*
!package.json
!README.md
```

#### 保护敏感数据

```gitignore
# 环境变量
.env
.env.*

# 私钥和证书
*.key
*.pem
*.p12
*.pfx

# 配置文件
config/production.json
secrets/
```

### 故障排查

#### 文件未找到

1. 检查 `.gitignore` 规则
2. 检查 `.geminiignore` 规则
3. 验证文件路径
4. 使用 `--all-files` 参数测试

#### 性能问题

1. 禁用递归搜索
2. 添加更多忽略规则
3. 减少 `memoryDiscoveryMaxDirs`
4. 使用更具体的搜索模式

## 高级配置

本节涵盖一些高级配置选项，适合有特殊需求的用户。

### 检查点 (Checkpointing)

检查点功能允许回滚文件更改：

```json
{
  "checkpointing": {
    "enabled": true
  }
}
```

或通过命令行：
```bash
gemini --checkpointing
```

功能特点：
- 基于 Git 实现
- 自动创建提交
- 支持回滚操作
- 不影响主分支

### Bug 报告配置

自定义 bug 报告 URL：

```json
{
  "bugCommand": {
    "urlTemplate": "https://github.com/myorg/myrepo/issues/new?title={title}&body={body}"
  }
}
```

支持的占位符：
- `{title}`: Bug 标题
- `{body}`: Bug 描述
- `{version}`: CLI 版本
- `{platform}`: 操作系统

### 工具输出摘要

为工具配置输出限制：

```json
{
  "summarizeToolOutput": {
    "ShellTool": {
      "tokenBudget": 2000
    },
    "ReadFileTool": {
      "tokenBudget": 5000
    },
    "WebFetchTool": {
      "tokenBudget": 1500
    }
  }
}
```

当输出超过 tokenBudget 时：
1. 使用 AI 摘要输出
2. 保留关键信息
3. 提示用户查看完整输出

### 扩展管理

#### 指定扩展

```bash
# 仅加载特定扩展
gemini --extensions ext1 ext2

# 列出所有可用扩展
gemini --list-extensions
```

#### 扩展配置格式

扩展的 `package.json`：

```json
{
  "name": "my-extension",
  "gemini-cli": {
    "contextFiles": ["context/setup.md"],
    "mcpServers": {
      "my-server": {
        "command": "${extensionPath}/server.js"
      }
    },
    "excludeTools": ["DangerousTool"]
  }
}
```

### ACP 模式

Agent Control Protocol 实验性功能：

```bash
gemini --experimental-acp
```

特点：
- 更高级的代理控制
- 支持多代理协作
- 实验性 API

### 自定义工具开发

#### 工具发现脚本

```json
{
  "toolDiscoveryCommand": "/usr/local/bin/discover-tools"
}
```

脚本输出格式：
```json
[
  {
    "name": "CustomTool",
    "description": "自定义工具",
    "inputSchema": {
      "type": "object",
      "properties": {
        "action": {
          "type": "string",
          "enum": ["read", "write", "delete"]
        },
        "path": {
          "type": "string"
        }
      },
      "required": ["action", "path"]
    }
  }
]
```

#### 工具执行脚本

```json
{
  "toolCallCommand": "/usr/local/bin/execute-tool"
}
```

脚本接收 JSON 输入：
```json
{
  "tool": "CustomTool",
  "arguments": {
    "action": "read",
    "path": "/example/file.txt"
  }
}
```

### 代理配置

支持多种代理协议：

```bash
# HTTP 代理
gemini --proxy http://proxy:8080

# SOCKS5 代理
gemini --proxy socks5://proxy:1080

# 认证代理
gemini --proxy http://user:pass@proxy:8080
```

代理优先级：
1. `--proxy` 命令行参数
2. `HTTPS_PROXY` 环境变量
3. `HTTP_PROXY` 环境变量

### 性能调优

#### Node.js 堆内存

```json
{
  "autoConfigureMaxOldSpaceSize": true
}
```

自动设置适当的堆内存大小，防止 OOM 错误。

#### 会话限制

```json
{
  "maxSessionTurns": 50  // 限制对话轮数
}
```

防止无限循环和资源耗尽。

### 安全增强

#### MCP 服务器信任

```json
{
  "mcpServers": {
    "trustedServer": {
      "command": "mcp-server",
      "trust": true  // 信任此服务器
    }
  }
}
```

#### 工具黑白名单

```json
{
  "coreTools": ["ReadFileTool", "EditTool"],  // 白名单
  "excludeTools": ["ShellTool"]                // 黑名单
}
```

### 多环境管理

#### 环境特定配置

使用不同的配置文件：

```bash
# 开发环境
export GEMINI_CONFIG_PATH=".gemini/dev.json"
gemini

# 生产环境
export GEMINI_CONFIG_PATH=".gemini/prod.json"
gemini
```

#### 条件配置

虽然不直接支持，但可以通过环境变量实现：

```json
{
  "telemetry": {
    "enabled": "${ENABLE_TELEMETRY}",
    "target": "${TELEMETRY_TARGET}"
  }
}
```

### 调试和诊断

#### 详细日志

```bash
# 启用调试模式
gemini --debug

# 设置日志级别
export LOG_LEVEL=debug
```

#### 性能分析

```bash
# 启用性能分析
node --inspect gemini

# 使用 Chrome DevTools 连接
```

### 未来特性预览

一些实验性功能可能需要特殊标志：

```bash
# 启用所有实验性功能
export GEMINI_EXPERIMENTAL=true
```

注意：实验性功能可能不稳定，仅供测试使用。

## 配置示例

本节提供一些常见场景的完整配置示例，帮助您快速开始。

### 个人开发环境

适合日常编码和小型项目：

```json
{
  "theme": "dracula",
  "selectedAuthType": "gemini-api-key",
  "vimMode": true,
  "showMemoryUsage": true,
  "preferredEditor": "code",
  "fileFiltering": {
    "respectGitIgnore": true,
    "respectGeminiIgnore": true,
    "enableRecursiveFileSearch": true
  },
  "telemetry": {
    "enabled": false
  },
  "checkpointing": {
    "enabled": true
  }
}
```

配合环境变量：
```bash
export GEMINI_API_KEY="your-api-key"
export GEMINI_MODEL="gemini-2.5-flash"
```

### 企业团队环境

适合团队协作和企业级项目：

```json
{
  "theme": "github-light",
  "selectedAuthType": "vertex-ai",
  "sandbox": "docker",
  "sandboxImage": "company.com/gemini-sandbox:approved",
  "coreTools": [
    "ReadFileTool",
    "WriteFileTool",
    "EditTool",
    "GrepTool",
    "GlobTool"
  ],
  "excludeTools": ["ShellTool"],
  "mcpServers": {
    "company-api": {
      "httpUrl": "https://api.company.com/mcp",
      "headers": {
        "Authorization": "Bearer ${COMPANY_TOKEN}"
      },
      "timeout": 30000
    }
  },
  "telemetry": {
    "enabled": true,
    "target": "gcp",
    "logPrompts": false
  },
  "bugCommand": {
    "urlTemplate": "https://jira.company.com/create?project=GEMINI&title={title}"
  },
  "proxy": "${CORPORATE_PROXY}",
  "fileFiltering": {
    "respectGitIgnore": true,
    "respectGeminiIgnore": true
  }
}
```

`.geminiignore` 文件：
```gitignore
# 公司敏感数据
/data/
/secrets/
*.key
*.pem
config/production.json
```

### 安全优先配置

最大化安全性，限制功能：

```json
{
  "theme": "no-color",
  "sandbox": "docker",
  "coreTools": [
    "ReadFileTool",
    "GrepTool",
    "GlobTool"
  ],
  "excludeTools": [
    "ShellTool",
    "WriteFileTool",
    "EditTool",
    "WebFetchTool",
    "WebSearchTool"
  ],
  "telemetry": {
    "enabled": false
  },
  "usageStatisticsEnabled": false,
  "fileFiltering": {
    "respectGitIgnore": true,
    "respectGeminiIgnore": true,
    "enableRecursiveFileSearch": false
  },
  "maxSessionTurns": 20,
  "disableAutoUpdate": true
}
```

### 性能优化配置

针对大型项目优化：

```json
{
  "theme": "ansi",
  "showMemoryUsage": true,
  "accessibility": {
    "disableLoadingPhrases": true
  },
  "fileFiltering": {
    "respectGitIgnore": true,
    "respectGeminiIgnore": true,
    "enableRecursiveFileSearch": false
  },
  "memoryDiscoveryMaxDirs": 5,
  "summarizeToolOutput": {
    "ShellTool": {
      "tokenBudget": 1000
    },
    "ReadFileTool": {
      "tokenBudget": 3000
    },
    "WebFetchTool": {
      "tokenBudget": 1000
    }
  },
  "autoConfigureMaxOldSpaceSize": true,
  "hideBanner": true,
  "hideTips": true
}
```

### 离线开发配置

无网络访问环境：

```json
{
  "selectedAuthType": "gemini-api-key",
  "excludeTools": [
    "WebFetchTool",
    "WebSearchTool"
  ],
  "telemetry": {
    "enabled": false
  },
  "disableAutoUpdate": true,
  "mcpServers": {
    "local-server": {
      "command": "/usr/local/bin/offline-mcp-server",
      "args": ["--offline-mode"]
    }
  }
}
```

### CI/CD 环境配置

自动化流水线使用：

```json
{
  "theme": "no-color",
  "selectedAuthType": "vertex-ai",
  "accessibility": {
    "disableLoadingPhrases": true
  },
  "hideBanner": true,
  "hideTips": true,
  "hideWindowTitle": true,
  "telemetry": {
    "enabled": true,
    "outfile": "/logs/gemini-telemetry.log"
  },
  "maxSessionTurns": 50,
  "checkpointing": {
    "enabled": false
  }
}
```

启动命令：
```bash
gemini --yolo --prompt "执行测试并生成报告"
```

### 教育/学习环境

适合教学和学习：

```json
{
  "theme": "github-light",
  "selectedAuthType": "oauth-personal",
  "vimMode": false,
  "showMemoryUsage": true,
  "sandbox": "docker",
  "coreTools": [
    "ReadFileTool",
    "WriteFileTool",
    "EditTool",
    "ShellTool"
  ],
  "telemetry": {
    "enabled": false
  },
  "bugCommand": {
    "urlTemplate": "mailto:teacher@school.edu?subject={title}&body={body}"
  },
  "maxSessionTurns": 30,
  "summarizeToolOutput": {
    "ShellTool": {
      "tokenBudget": 5000
    }
  }
}
```

### 多语言项目配置

支持多种编程语言的项目：

```json
{
  "contextFileName": [
    "GEMINI.md",
    "CONTEXT.md",
    ".gemini-context"
  ],
  "mcpServers": {
    "python-lsp": {
      "command": "python-lsp-server"
    },
    "typescript-lsp": {
      "command": "typescript-language-server",
      "args": ["--stdio"]
    },
    "rust-analyzer": {
      "command": "rust-analyzer"
    }
  },
  "fileFiltering": {
    "respectGitIgnore": true,
    "respectGeminiIgnore": true
  }
}
```

### 最小化配置

最简单的工作配置：

```json
{
  "selectedAuthType": "gemini-api-key"
}
```

配合 `.env` 文件：
```
GEMINI_API_KEY=your-api-key-here
```

### 调试配置

开发和调试 Gemini CLI 本身：

```json
{
  "theme": "default",
  "telemetry": {
    "enabled": true,
    "target": "local",
    "otlpEndpoint": "http://localhost:4317"
  },
  "showMemoryUsage": true,
  "checkpointing": {
    "enabled": true
  }
}
```

启动命令：
```bash
DEBUG=true gemini --debug
```

---

**本章小结**：Gemini CLI 提供了极其灵活的配置系统，支持通过命令行参数、环境变量和配置文件进行细粒度控制。关键要点：

1. **配置优先级**：命令行 > 系统配置 > 工作区配置 > 用户配置 > 环境变量 > 默认值
2. **分层配置**：允许在不同层级设置不同的配置，实现灵活的环境管理
3. **安全性**：通过沙箱、工具限制等机制确保安全
4. **可扩展性**：支持 MCP 服务器、自定义工具等扩展机制
5. **性能优化**：提供多种配置选项优化大型项目的性能

选择适合您使用场景的配置，并根据需要进行调整。记住始终保护好您的 API 密钥和敏感信息。