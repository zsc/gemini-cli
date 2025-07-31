# 附录 C：故障排查指南

本指南帮助您诊断和解决使用 Gemini CLI 时可能遇到的常见问题。

## 目录

1. [快速诊断流程](#快速诊断流程)
2. [认证与配置问题](#认证与配置问题)
3. [API 与网络错误](#api-与网络错误)
4. [工具执行错误](#工具执行错误)
5. [MCP 集成问题](#mcp-集成问题)
6. [性能与资源问题](#性能与资源问题)
7. [调试技巧](#调试技巧)
8. [错误恢复机制](#错误恢复机制)
9. [常见错误速查表](#常见错误速查表)

## 快速诊断流程

当遇到问题时，按以下步骤快速定位：

### 1. 检查基础环境

```bash
# 检查 Node.js 版本（需要 18+）
node --version

# 检查 Gemini CLI 版本
gemini --version

# 验证认证状态
gemini
# 然后输入 /auth 查看当前认证方式
```

### 2. 查看错误日志

Gemini CLI 会在临时目录生成详细的错误报告：

```bash
# macOS/Linux
ls /tmp/gemini-client-error-*.json

# Windows
dir %TEMP%\gemini-client-error-*.json
```

错误报告文件名格式：`gemini-client-error-{type}-{timestamp}.json`

### 3. 启用调试模式

通过环境变量启用详细日志：

```bash
# 使用 DEBUG 环境变量
DEBUG=true gemini

# 或使用 DEBUG_MODE
export DEBUG_MODE=1
gemini

# 指定调试端口（用于 Node.js 调试）
DEBUG=true DEBUG_PORT=9229 gemini
```

### 4. 检查会话日志

会话日志存储在项目的 `.gemini` 目录：

```bash
# 查看日志文件
cat .gemini/logs.json

# 查看检查点
ls .gemini/checkpoint-*.json
```

## 认证与配置问题

### 认证方式检查

Gemini CLI 支持多种认证方式，每种都有特定的配置要求：

#### 1. Google 账号登录（推荐）

```bash
# 使用 /auth 命令选择 "Login with Google"
gemini
> /auth
```

**常见问题：**
- **403 Forbidden**: 项目未启用 Gemini Code Assist
  - 解决：访问 https://goo.gle/set-up-gemini-code-assist 启用服务
- **401 Unauthorized**: 认证令牌过期
  - 解决：重新运行 `/auth` 进行登录

#### 2. Gemini API Key

需要设置环境变量：

```bash
export GEMINI_API_KEY="your-api-key"
gemini
```

**常见问题：**
- **"GEMINI_API_KEY environment variable not found"**
  - 解决：确保设置了 GEMINI_API_KEY 环境变量
  - 可以使用 `.env` 文件（自动加载）

#### 3. Vertex AI

需要以下配置之一：

```bash
# 选项 1：项目和位置
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"

# 选项 2：Express 模式
export GOOGLE_API_KEY="your-api-key"
```

**常见问题：**
- **缺少必需的环境变量**
  - 错误信息会明确指出需要哪些变量

### 配置文件位置

配置文件按以下优先级加载：

1. 项目级：`.gemini/settings.json`
2. 用户级：`~/.config/gemini/settings.json`
3. 全局级：`/etc/gemini/settings.json`（Unix）或 `%PROGRAMDATA%\gemini\settings.json`（Windows）

### 环境变量优先级

环境变量会覆盖配置文件中的设置：

```bash
# 临时覆盖模型设置
GEMINI_MODEL="gemini-2.0-flash-thinking-exp-1219" gemini
```

## API 与网络错误

### 配额错误处理

#### 1. Pro 模型配额超限

**错误信息：** `Quota exceeded for quota metric 'Gemini 2.5 Pro Requests'`

**自动处理：**
- OAuth 用户：自动切换到 Flash 模型
- API Key 用户：需要手动处理

**解决方案：**
- 免费用户：升级到 Gemini Code Assist Standard/Enterprise
- 付费用户：使用 `/auth` 切换到 AI Studio API Key
- 等待配额重置（通常为每日重置）

#### 2. 速率限制（429 错误）

**自动重试机制：**
- 默认 5 次重试
- 指数退避：5-30 秒延迟
- 支持 Retry-After 响应头

**手动处理：**
```bash
# 切换到备用模型
GEMINI_MODEL="gemini-2.0-flash-exp-0111" gemini

# 或在会话中使用 /model 命令
> /model gemini-2.0-flash-exp-0111
```

### 网络连接问题

#### 1. 代理配置

支持标准环境变量：

```bash
# HTTP 代理
export HTTP_PROXY="http://proxy.company.com:8080"
export HTTPS_PROXY="http://proxy.company.com:8080"

# 不使用代理的地址
export NO_PROXY="localhost,127.0.0.1"
```

#### 2. 超时错误

**默认超时：**
- API 请求：根据 Retry-After 头动态调整
- MCP 工具：10 分钟

**自定义超时：**
```json
// .gemini/settings.json
{
  "requestTimeoutMs": 120000  // 2 分钟
}
```

### 服务器错误（5xx）

**自动处理：**
- 自动重试（最多 5 次）
- 记录详细错误日志

**诊断步骤：**
1. 检查 Google 服务状态页面
2. 查看错误报告文件
3. 尝试切换认证方式或区域

## 工具执行错误

### Shell 命令执行问题

#### 1. 命令验证失败

**常见错误：**
- **"Command cannot be empty"**: 空命令
- **"Directory cannot be absolute"**: 使用了绝对路径
- **"Directory must exist"**: 指定的目录不存在
- **"Could not identify command root"**: 无法识别命令

**解决方案：**
```bash
# 正确：使用相对路径
> Run: cd src && npm test

# 错误：使用绝对路径
> Run: cd /Users/name/project/src  # 会被拒绝
```

#### 2. 权限确认

首次执行命令需要用户确认：
- **Y**: 允许本次执行
- **A**: 始终允许该命令
- **N**: 拒绝执行

### 文件操作错误

#### 1. 常见文件系统错误

- **ENOENT**: 文件不存在
  ```bash
  # 解决：先检查文件是否存在
  > List files in src/
  ```

- **EACCES**: 权限不足
  ```bash
  # 解决：检查文件权限
  ls -la problematic-file.txt
  ```

- **ETIMEDOUT**: 操作超时
  - 通常发生在处理大文件时
  - 考虑分批处理

#### 2. 编辑冲突

当编辑的内容与文件实际内容不匹配时：

```
Error: The content to be replaced was not found in the file
```

**解决步骤：**
1. 使用 `read_file` 查看当前内容
2. 确保编辑时包含足够的上下文
3. 注意保留原始的缩进和空白字符

### 工具并发执行

多个工具可能并行执行，注意：

1. **文件锁定**：同时编辑同一文件可能失败
2. **依赖顺序**：确保有依赖关系的操作按顺序执行
3. **资源竞争**：避免同时执行资源密集型操作

## MCP 集成问题

### MCP 服务器连接状态

使用 `/mcp` 命令查看所有 MCP 服务器状态：

```bash
> /mcp
```

**状态说明：**
- 🟢 **CONNECTED**: 正常连接
- 🟡 **CONNECTING**: 连接中
- 🔴 **DISCONNECTED**: 断开或错误

### 常见 MCP 错误

#### 1. 服务器启动失败

**错误：** `Failed to start MCP server`

**排查步骤：**
```bash
# 1. 验证服务器配置
cat ~/.config/gemini/settings.json | grep mcp

# 2. 测试服务器可执行性
which your-mcp-server

# 3. 检查权限
ls -la $(which your-mcp-server)
```

#### 2. OAuth 认证要求

某些 MCP 服务器需要 OAuth 认证：

**错误：** `WWW-Authenticate: Bearer realm="..."` 

**解决方案：**
1. 按提示访问认证 URL
2. 完成 OAuth 流程
3. 令牌会自动保存

#### 3. 工具发现超时

**默认超时：** 10 分钟

**自定义配置：**
```json
// settings.json
{
  "mcpServers": {
    "slow-server": {
      "command": "slow-mcp-server",
      "timeout": 600000  // 10 分钟
    }
  }
}
```

### MCP 调试技巧

1. **查看服务器日志**
   ```bash
   # 启用 MCP 调试日志
   MCP_DEBUG=1 gemini
   ```

2. **隔离测试**
   ```bash
   # 直接运行 MCP 服务器
   your-mcp-server --help
   ```

3. **验证工具可用性**
   ```bash
   > /tools
   # 查看所有可用工具，包括 MCP 提供的
   ```

## 性能与资源问题

### 响应缓慢

#### 1. 模型响应慢

**自动处理：**
- OAuth 用户检测到慢响应会自动切换到 Flash 模型

**手动优化：**
```bash
# 使用更快的模型
> /model gemini-2.0-flash-exp-0111

# 或启动时指定
GEMINI_MODEL="gemini-2.0-flash-exp-0111" gemini
```

#### 2. 上下文过大

**症状：**
- 响应时间显著增加
- 可能出现上下文长度错误

**解决方案：**
```bash
# 压缩历史记录
> /compress

# 清除不必要的消息
> /clear
```

### 内存使用

#### 1. 查看内存统计

```bash
> /stats
```

显示：
- 会话 token 使用情况
- 工具调用统计
- 内存文件加载情况

#### 2. 内存优化技巧

**GEMINI.md 文件管理：**
- 保持文件简洁，避免冗余信息
- 使用分层结构（项目级 > 目录级）
- 定期清理过时内容

**检查点管理：**
```bash
# 列出所有检查点
ls .gemini/checkpoint-*.json

# 删除旧检查点
rm .gemini/checkpoint-old-*.json
```

### 大文件处理

#### 1. 文件大小限制

- 单个文件读取限制：默认 2000 行
- 可以分批读取：
  ```
  Read file.txt from line 2000 to 4000
  ```

#### 2. 搜索优化

对于大型代码库：
```bash
# 使用 glob 而不是递归搜索
> Search for "*.ts" files in src/

# 使用 grep 的类型过滤
> Search for "function" in TypeScript files
```

## 调试技巧

### 日志级别控制

#### 1. 控制台日志

```javascript
// 日志优先级
console.debug()  // 仅在 DEBUG 模式显示
console.warn()   // 警告（如 429 错误）
console.error()  // 错误（如 5xx 错误）
```

#### 2. 详细错误报告

错误报告位置：
- macOS/Linux: `/tmp/gemini-client-error-*.json`
- Windows: `%TEMP%\gemini-client-error-*.json`

报告内容：
```json
{
  "error": {
    "message": "错误信息",
    "stack": "调用栈"
  },
  "context": {
    // 请求上下文
  }
}
```

### VSCode 集成调试

#### 1. 启用 Node.js 调试

```bash
# 使用调试模式启动
DEBUG=true DEBUG_PORT=9229 gemini
```

#### 2. VSCode 调试配置

`.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to Gemini CLI",
      "port": 9229,
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}
```

### 常用诊断命令

```bash
# 查看当前配置
> /auth          # 认证状态
> /model         # 当前模型
> /stats         # 使用统计
> /tools         # 可用工具
> /mcp           # MCP 服务器状态

# 内存和历史
> /memory show   # 显示加载的内存文件
> /compress      # 压缩对话历史
```

### 问题隔离技巧

1. **最小复现案例**
   - 创建新的空目录测试
   - 逐步添加配置直到问题出现

2. **环境变量检查**
   ```bash
   env | grep GEMINI
   env | grep GOOGLE
   ```

3. **配置文件验证**
   ```bash
   # 验证 JSON 格式
   jq . ~/.config/gemini/settings.json
   ```

## 错误恢复机制

### 检查点与恢复

#### 1. 自动检查点

启用检查点功能：
```json
// settings.json
{
  "checkpointing": true
}
```

检查点保存位置：`.gemini/checkpoints/`

#### 2. 恢复工具调用

```bash
# 列出可恢复的检查点
> /restore

# 恢复特定检查点
> /restore checkpoint-name
```

恢复内容包括：
- 对话历史
- 客户端历史
- 项目文件状态（如果启用了 Git 快照）

### 会话恢复

#### 1. 意外退出后恢复

会话日志自动保存在 `.gemini/logs.json`

```bash
# 查看最近的会话
cat .gemini/logs.json | jq '.[-10:]'
```

#### 2. 手动保存会话状态

```bash
# 保存当前对话到检查点
> Save this conversation

# 稍后恢复
> /restore saved-conversation
```

### 错误后的清理

#### 1. 清理临时文件

```bash
# 清理错误报告
rm /tmp/gemini-client-error-*.json

# 清理损坏的日志
rm .gemini/logs.json.*.bak
```

#### 2. 重置配置

```bash
# 备份当前配置
cp ~/.config/gemini/settings.json ~/.config/gemini/settings.json.bak

# 重置为默认配置
rm ~/.config/gemini/settings.json
```

### 紧急恢复步骤

如果 Gemini CLI 完全无法启动：

1. **清理缓存和日志**
   ```bash
   rm -rf .gemini/
   rm -rf ~/.config/gemini/cache/
   ```

2. **检查环境变量冲突**
   ```bash
   unset GEMINI_MODEL
   unset GEMINI_API_KEY
   ```

3. **使用最小配置启动**
   ```bash
   gemini --no-config
   ```

## 常见错误速查表

| 错误信息 | 可能原因 | 快速解决方案 |
|---------|---------|-------------|
| `GEMINI_API_KEY environment variable not found` | 未设置 API Key | `export GEMINI_API_KEY="your-key"` |
| `Quota exceeded for quota metric` | 配额用尽 | 等待重置或切换模型/认证方式 |
| `403 Forbidden` | 项目未启用服务 | 访问设置页面启用 Gemini Code Assist |
| `401 Unauthorized` | 认证过期 | 运行 `/auth` 重新认证 |
| `429 Too Many Requests` | 请求过快 | 自动重试，或手动等待 |
| `ENOENT: no such file` | 文件不存在 | 检查文件路径是否正确 |
| `EACCES: permission denied` | 权限不足 | 检查文件权限 |
| `Command cannot be empty` | Shell 命令为空 | 提供有效命令 |
| `Directory cannot be absolute` | 使用了绝对路径 | 改用相对路径 |
| `The content to be replaced was not found` | 编辑内容不匹配 | 重新读取文件，确保匹配 |
| `Context length exceeded` | 上下文过长 | 使用 `/compress` 或 `/clear` |
| `Failed to start MCP server` | MCP 服务器错误 | 检查服务器配置和权限 |
| `Network timeout` | 网络连接问题 | 检查网络和代理设置 |

## 获取帮助

### 官方资源

1. **GitHub Issues**: https://github.com/google-gemini/gemini-cli/issues
2. **文档站点**: https://docs.anthropic.com/en/docs/gemini-cli
3. **社区论坛**: 查看项目 README 中的链接

### 报告问题时包含

1. **错误信息**: 完整的错误输出
2. **环境信息**: 
   ```bash
   gemini --version
   node --version
   echo $GEMINI_MODEL
   ```
3. **重现步骤**: 最小可重现示例
4. **错误报告文件**: `/tmp/gemini-client-error-*.json` 的内容

### 社区支持

- 搜索已有 issues 看是否有类似问题
- 提供清晰的问题描述和上下文
- 如果找到解决方案，分享给其他用户

---

**提示**: 大多数问题可以通过检查配置、重新认证或清理缓存解决。遇到持续性问题时，启用调试模式收集更多信息。