# 第 18 章：安全与隐私保护

Gemini CLI 在设计之初就将安全性和隐私保护作为核心考虑，实现了多层次的安全机制，包括认证授权、沙箱执行、路径验证、数据保护等。本章深入分析这些安全特性的设计理念和实现细节。

## 18.1 认证与授权体系

### 18.1.1 多种认证方式

Gemini CLI 支持多种认证方式，以适应不同的使用场景和安全需求：

```typescript
export enum AuthType {
  LOGIN_WITH_GOOGLE = 'login-with-google',
  CLOUD_SHELL = 'cloud-shell', 
  USE_GEMINI = 'use-gemini',
  USE_VERTEX_AI = 'use-vertex-ai'
}
```

每种认证方式都有其特定的安全考量和验证逻辑（`packages/cli/src/config/auth.ts`）：

1. **Google 登录认证**：使用 OAuth 2.0 流程，支持 PKCE（Proof Key for Code Exchange）以增强安全性
2. **Cloud Shell 认证**：利用 Google Cloud Shell 的 ADC（Application Default Credentials）
3. **Gemini API Key**：通过环境变量配置，适合 CI/CD 环境
4. **Vertex AI 认证**：支持项目级别的访问控制，可以使用 GOOGLE_API_KEY 或 GOOGLE_CLOUD_PROJECT + GOOGLE_CLOUD_LOCATION 组合

认证方法验证确保了必要的环境变量和配置存在：

```typescript
// packages/cli/src/config/auth.ts
export const validateAuthMethod = (authMethod: string): string | null => {
  loadEnvironment(); // 支持从 .env 文件加载配置
  
  if (authMethod === AuthType.USE_VERTEX_AI) {
    const hasVertexProjectLocationConfig =
      !!process.env.GOOGLE_CLOUD_PROJECT && !!process.env.GOOGLE_CLOUD_LOCATION;
    const hasGoogleApiKey = !!process.env.GOOGLE_API_KEY;
    if (!hasVertexProjectLocationConfig && !hasGoogleApiKey) {
      return '详细的错误提示...';
    }
  }
  // 其他认证方式的验证...
}
```

### 18.1.2 OAuth 2.0 安全实现

OAuth 实现采用了业界最佳实践，特别是在 MCP OAuth 支持中（`packages/core/src/mcp/oauth-provider.ts`）：

```typescript
// OAuth 安全配置
const OAUTH_CLIENT_ID = '681255809395-oo8ft2oprdrnp9e3aqf6av3hmdib135j.apps.googleusercontent.com';
const OAUTH_SCOPE = [
  'https://www.googleapis.com/auth/cloud-platform',
  'https://www.googleapis.com/auth/userinfo.email',
  'https://www.googleapis.com/auth/userinfo.profile',
];

// PKCE 实现
interface PKCEParams {
  codeVerifier: string;
  codeChallenge: string;
  state: string;
}
```

关键安全特性：
- **状态参数验证**：每次认证流程生成唯一的 state 参数，防止 CSRF 攻击
- **代码验证器**：PKCE 机制使用 code_verifier 和 code_challenge，防止授权码拦截
- **最小权限原则**：仅请求必要的 OAuth 作用域
- **动态客户端注册**：支持 OAuth 动态客户端注册，增强灵活性

MCP OAuth 提供了完整的 OAuth 2.0 流程实现：

```typescript
// packages/core/src/mcp/oauth-provider.ts
export class MCPOAuthProvider {
  private static readonly REDIRECT_PORT = 7777;
  private static readonly REDIRECT_PATH = '/oauth/callback';
  
  // 支持动态客户端注册
  static async registerClient(
    registrationUrl: string,
    config: MCPOAuthConfig
  ): Promise<OAuthClientRegistrationResponse> {
    // 实现细节...
  }
}
```

### 18.1.3 凭证存储安全

凭证存储采用了多重保护措施（`packages/core/src/mcp/oauth-token-storage.ts`）：

```typescript
// 凭证文件权限设置
await fs.writeFile(
  tokenFile,
  JSON.stringify(tokenArray, null, 2),
  { mode: 0o600 } // 仅所有者可读写
);
```

存储架构：
- **集中管理**：所有凭证存储在用户目录下的 `.gemini` 文件夹
- **分类存储**：
  - OAuth 凭证：`~/.gemini/oauth_creds.json`
  - MCP 令牌：`~/.gemini/mcp-oauth-tokens.json`
- **结构化存储**：使用标准化的接口定义

```typescript
// MCP OAuth 凭证结构
export interface MCPOAuthCredentials {
  serverName: string;
  token: MCPOAuthToken;
  clientId?: string;
  tokenUrl?: string;
  mcpServerUrl?: string;
  updatedAt: number; // 时间戳用于跟踪更新
}
```

凭证管理类提供了安全的 CRUD 操作：

```typescript
export class MCPOAuthTokenStorage {
  private static readonly TOKEN_FILE = 'mcp-oauth-tokens.json';
  private static readonly CONFIG_DIR = '.gemini';
  
  // 确保配置目录存在
  private static async ensureConfigDir(): Promise<void> {
    const configDir = path.dirname(this.getTokenFilePath());
    await fs.mkdir(configDir, { recursive: true });
  }
}
```

## 18.2 沙箱执行环境

### 18.2.1 容器沙箱

Gemini CLI 支持通过 Docker 或 Podman 运行在隔离的容器环境中（`packages/cli/src/utils/sandbox.ts`）：

```typescript
const args = [
  'run', '-i', '--rm', '--init',
  '--workdir', containerWorkdir,
  '--volume', `${workdir}:${containerWorkdir}`,
  // 其他安全相关参数
];
```

容器沙箱的关键安全特性：

1. **用户权限映射**：智能处理容器内的用户权限问题
   ```typescript
   // 自动检测是否需要使用当前用户的 UID/GID
   async function shouldUseCurrentUserInSandbox(): Promise<boolean> {
     const envVar = process.env.SANDBOX_SET_UID_GID?.toLowerCase().trim();
     
     // 对 Debian/Ubuntu 系统默认启用，避免权限问题
     if (os.platform() === 'linux') {
       const osReleaseContent = await readFile('/etc/os-release', 'utf8');
       if (osReleaseContent.includes('ID=debian') || 
           osReleaseContent.includes('ID=ubuntu')) {
         return true;
       }
     }
     return false;
   }
   ```

2. **文件系统隔离**：仅挂载必要的目录，支持跨平台路径转换
   ```typescript
   function getContainerPath(hostPath: string): string {
     if (os.platform() !== 'win32') {
       return hostPath;
     }
     // Windows 路径转换为容器路径格式
     const withForwardSlashes = hostPath.replace(/\\/g, '/');
     const match = withForwardSlashes.match(/^([A-Z]):\/(.*)/i);
     if (match) {
       return `/${match[1].toLowerCase()}/${match[2]}`;
     }
     return hostPath;
   }
   ```

3. **环境变量管理**：智能处理 PATH 和 PYTHONPATH 等关键环境变量
   ```typescript
   // 仅映射工作目录内的路径到容器
   if (process.env.PATH) {
     const paths = process.env.PATH.split(pathSeparator);
     for (const p of paths) {
       const containerPath = getContainerPath(p);
       if (containerPath.toLowerCase().startsWith(containerWorkdir.toLowerCase())) {
         pathSuffix += `:${containerPath}`;
       }
     }
   }
   ```

4. **端口映射**：支持通过环境变量配置端口转发
   ```typescript
   function ports(): string[] {
     return (process.env.SANDBOX_PORTS ?? '')
       .split(',')
       .filter((p) => p.trim())
       .map((p) => p.trim());
   }
   
   // 使用 socat 进行端口转发
   ports().forEach((p) =>
     shellCmds.push(
       `socat TCP4-LISTEN:${p},bind=$(hostname -i),fork,reuseaddr TCP4:127.0.0.1:${p} 2> /dev/null &`
     )
   );
   ```

5. **自定义初始化**：支持项目级别的沙箱配置
   ```typescript
   const projectSandboxBashrc = path.join(SETTINGS_DIRECTORY_NAME, 'sandbox.bashrc');
   if (fs.existsSync(projectSandboxBashrc)) {
     shellCmds.push(`source ${getContainerPath(projectSandboxBashrc)};`);
   }
   ```

### 18.2.2 macOS Seatbelt 沙箱

在 macOS 上，支持使用系统原生的 Seatbelt 沙箱技术：

```typescript
const BUILTIN_SEATBELT_PROFILES = [
  'permissive-open',      // 宽松模式，允许网络访问
  'permissive-closed',    // 宽松模式，禁止网络访问
  'permissive-proxied',   // 宽松模式，通过代理访问网络
  'restrictive-open',     // 严格模式，允许网络访问
  'restrictive-closed',   // 严格模式，禁止网络访问
  'restrictive-proxied',  // 严格模式，通过代理访问网络
];
```

Seatbelt 沙箱的实现细节：

```typescript
// packages/cli/src/utils/sandbox.ts
if (config.command === 'sandbox-exec') {
  const profile = (process.env.SEATBELT_PROFILE ??= 'permissive-open');
  let profileFile = new URL(`sandbox-macos-${profile}.sb`, import.meta.url).pathname;
  
  // 支持自定义配置文件
  if (!BUILTIN_SEATBELT_PROFILES.includes(profile)) {
    profileFile = path.join(SETTINGS_DIRECTORY_NAME, `sandbox-macos-${profile}.sb`);
  }
  
  // 传递必要的目录访问权限
  const args = [
    '-D', `TARGET_DIR=${fs.realpathSync(process.cwd())}`,
    '-D', `TMP_DIR=${fs.realpathSync(os.tmpdir())}`,
    '-D', `HOME_DIR=${fs.realpathSync(os.homedir())}`,
    '-D', `CACHE_DIR=${fs.realpathSync(execSync(`getconf DARWIN_USER_CACHE_DIR`).toString().trim())}`,
    '-f', profileFile,
    // ...
  ];
}
```

代理支持：
```typescript
// 支持在沙箱内使用代理
if (proxyCommand) {
  const proxy = process.env.HTTPS_PROXY || 
                process.env.HTTP_PROXY || 
                'http://localhost:8877';
  sandboxEnv['HTTPS_PROXY'] = proxy;
  sandboxEnv['https_proxy'] = proxy; // 兼容小写环境变量
}
```

### 18.2.3 沙箱配置管理

```typescript
function getSandboxCommand(sandbox?: boolean | string): SandboxConfig['command'] | '' {
  // 检查是否已在沙箱内
  if (process.env.SANDBOX) {
    return '';
  }
  
  // 验证沙箱命令
  if (!isSandboxCommand(sandbox)) {
    console.error(`ERROR: invalid sandbox command '${sandbox}'`);
    process.exit(1);
  }
  
  // 确认命令存在
  if (!commandExists.sync(sandbox)) {
    console.error(`ERROR: missing sandbox command '${sandbox}'`);
    process.exit(1);
  }
}
```

沙箱环境的识别：
- 容器沙箱：检查 `process.env.SANDBOX` 的值（如 'docker', 'podman'）
- macOS Seatbelt：`process.env.SANDBOX === 'sandbox-exec'`
- 支持嵌套检测，避免双重沙箱

## 18.3 文件系统安全

### 18.3.1 路径验证

所有文件操作都经过严格的路径验证：

```typescript
export function isWithinRoot(pathToCheck: string, rootDirectory: string): boolean {
  const normalizedPathToCheck = path.resolve(pathToCheck);
  const normalizedRootDirectory = path.resolve(rootDirectory);
  
  // 确保根目录路径以分隔符结尾（除非是根路径本身）
  const rootWithSeparator = normalizedRootDirectory === path.sep
    ? normalizedRootDirectory
    : normalizedRootDirectory + path.sep;
    
  return normalizedPathToCheck === normalizedRootDirectory ||
         normalizedPathToCheck.startsWith(rootWithSeparator);
}
```

### 18.3.2 文件操作限制

```typescript
validateToolParams(params: WriteFileToolParams): string | null {
  // 必须使用绝对路径
  if (!path.isAbsolute(params.file_path)) {
    return 'File path must be absolute';
  }
  
  // 必须在根目录内
  if (!isWithinRoot(params.file_path, this.config.getTargetDir())) {
    return 'File path must be within the root directory';
  }
  
  // 检查是否为目录
  const stats = fs.statSync(params.file_path);
  if (stats.isDirectory()) {
    return 'Cannot write to a directory';
  }
}
```

### 18.3.3 文件大小限制

```typescript
const fileSizeInBytes = stats.size;
const maxFileSize = 20 * 1024 * 1024; // 20MB

if (fileSizeInBytes > maxFileSize) {
  throw new Error(
    `File size exceeds the 20MB limit: ${filePath} ` +
    `(${(fileSizeInBytes / (1024 * 1024)).toFixed(2)}MB)`
  );
}
```

## 18.4 Shell 执行安全

### 18.4.1 命令执行隔离

Shell 命令执行采用了多重安全措施：

```typescript
const child = spawn(shell, shellArgs, {
  cwd,
  stdio: ['ignore', 'pipe', 'pipe'], // 隔离标准输入
  detached: !isWindows, // 使用进程组便于终止
  env: {
    ...process.env,
    // 可以在这里过滤敏感环境变量
  }
});
```

### 18.4.2 进程管理

```typescript
// 优雅终止进程
child.kill('SIGTERM');

// 强制终止（超时后）
setTimeout(() => {
  if (isWindows) {
    spawn('taskkill', ['/F', '/T', '/PID', child.pid.toString()]);
  } else {
    process.kill(-child.pid, 'SIGKILL');
  }
}, SIGKILL_TIMEOUT_MS);
```

## 18.5 数据隐私保护

### 18.5.1 遥测数据脱敏

遥测系统设计时就考虑了隐私保护：

```typescript
// 遥测事件定义（不包含敏感数据）
export const EVENT_USER_PROMPT = 'gemini_cli.user_prompt';
export const EVENT_TOOL_CALL = 'gemini_cli.tool_call';
export const EVENT_API_REQUEST = 'gemini_cli.api_request';

// 仅记录统计信息，不记录具体内容
export const METRIC_TOKEN_USAGE = 'gemini_cli.token.usage';
export const METRIC_SESSION_COUNT = 'gemini_cli.session.count';
```

### 18.5.2 用户数据处理

```typescript
// 用户信息缓存（仅存储必要信息）
async function fetchAndCacheUserInfo(client: OAuth2Client): Promise<void> {
  const userInfo = await response.json();
  if (userInfo.email) {
    await cacheGoogleAccount(userInfo.email); // 仅缓存邮箱
  }
}
```

### 18.5.3 隐私声明

系统提供清晰的隐私声明组件：

```typescript
export const GeminiPrivacyNotice = () => {
  return (
    <Box flexDirection="column">
      <Text bold>Gemini API Key Notice</Text>
      <Text>
        By using the Gemini API, you are agreeing to:
        - Google APIs Terms of Service
        - Gemini API Additional Terms
      </Text>
    </Box>
  );
};
```

## 18.6 网络安全

### 18.6.1 代理支持

支持在受限网络环境中使用代理：

```typescript
// 代理配置
if (proxyCommand) {
  let proxy = process.env.HTTPS_PROXY || 
              process.env.HTTP_PROXY || 
              'http://localhost:8877';
  
  // 容器内代理设置
  args.push('--env', `HTTPS_PROXY=${proxy}`);
  args.push('--env', `HTTP_PROXY=${proxy}`);
  
  // NO_PROXY 配置
  const noProxy = process.env.NO_PROXY;
  if (noProxy) {
    args.push('--env', `NO_PROXY=${noProxy}`);
  }
}
```

### 18.6.2 TLS/SSL 验证

所有 API 通信都使用 HTTPS，确保传输安全。

## 18.7 安全最佳实践

### 18.7.1 环境变量安全

```typescript
// 敏感环境变量的安全处理
const validateAuthMethod = (authMethod: string): string | null => {
  if (authMethod === AuthType.USE_GEMINI) {
    if (!process.env.GEMINI_API_KEY) {
      return 'GEMINI_API_KEY environment variable not found';
    }
  }
  // 不在日志中打印 API Key
};
```

### 18.7.2 错误处理

```typescript
// 避免在错误信息中泄露敏感信息
catch (error) {
  console.error('Authentication failed:', 
    error instanceof Error ? error.message : 'Unknown error'
  );
  // 不记录完整的错误堆栈
}
```

### 18.7.3 权限最小化

- 文件操作仅限于指定的根目录
- OAuth 仅请求必要的权限范围
- 沙箱环境限制系统调用
- 容器运行时避免使用 root 用户

## 18.8 合规性考虑

### 18.8.1 数据驻留

- 凭证存储在用户本地
- 遥测数据匿名化处理
- 支持完全离线运行（使用 API Key）

### 18.8.2 审计能力

- 所有工具调用都有日志记录
- 文件操作需要用户确认
- 支持只读模式运行

## 本章小结

Gemini CLI 的安全设计体现了纵深防御的理念：

1. **多层认证**：支持多种认证方式，适应不同安全需求
2. **执行隔离**：通过沙箱技术隔离执行环境
3. **访问控制**：严格的路径验证和权限检查
4. **数据保护**：凭证加密存储，遥测数据脱敏
5. **网络安全**：支持代理和 TLS 加密
6. **隐私优先**：最小化数据收集，透明的隐私政策

这些安全措施共同构建了一个可信赖的 AI 辅助开发环境，让用户能够安心地使用 Gemini CLI 处理敏感代码和数据。安全不是一个特性，而是贯穿整个系统设计的基本原则。