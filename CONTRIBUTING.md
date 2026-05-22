# 开发规范

## 1. 分支管理

### 分支命名

| 类型 | 格式 | 示例 |
|------|------|------|
| 功能开发 | `feature/<name>` | `feature/network-aware-router` |
| Bug 修复 | `fix/<name>` | `fix/proxy-auth-error` |
| 重构 | `refactor/<name>` | `refactor/config-service` |
| 测试 | `test/<name>` | `test/network-detector` |

### 开发流程

1. 从 `main` 创建功能分支
2. 开发 + 测试
3. 推送到远程，创建 PR 合并到 `main`
4. 至少一人 Code Review 后合并

```bash
git checkout main
git pull origin main
git checkout -b feature/network-aware-router
# 开发...
git push origin feature/network-aware-router
# 创建 PR
```

## 2. 开发与生产环境分离

### 目录约定

```
D:\code\claude-code-router/       # 开发环境（代码仓）
├── dist/                         # 本地构建产物
├── packages/                     # 源代码
└── tests/                        # 测试配置

~/.claude-code-router/            # 生产环境（部署目录）
├── config.json                   # 生产配置
├── logs/                         # 生产日志
└── presets/                      # 生产预设
```

### 端口分离

| 环境 | 默认端口 | 配置方式 |
|------|---------|---------|
| 生产 | 3456 | `~/.claude-code-router/config.json` 中 `PORT` |
| 开发 | 13456 | 环境变量 `SERVICE_PORT=13456` |
| 冒烟测试 | 23456 | 测试文件内定义 `TEST_PORT = 23456` |
| 集成测试 | 23457 | 测试文件内定义 `TEST_PORT = 23457`（避免冲突） |

### 开发启动

```bash
# 方式 1：开发模式（开发端口）
SERVICE_PORT=13456 pnpm dev:server

# 方式 2：本地构建后部署到生产目录
pnpm build
node dist/cli.js start
```

> 注意：本地构建后 `node dist/cli.js start` 会使用 `~/.claude-code-router/config.json`（生产配置）。
> 开发测试时可用 `SERVICE_PORT=13456` 启动临时服务，不影响生产环境。

## 3. TDD 开发流程

### 红-绿-重构循环

```
1. 写失败测试（Red）
2. 写最小实现代码使测试通过（Green）
3. 重构，确保测试仍通过（Refactor）
```

### 每个特性的开发顺序

```
1. 理解需求 → 写设计文档
2. 写接口测试（集成测试级别）
3. 写单元测试
4. 实现代码
5. 重构
6. 写冒烟测试
```

## 4. 分层测试体系

### 4.1 测试分层

```
┌─────────────────────────────────┐
│         冒烟测试 (Smoke)         │  ← 验证核心功能端到端可用
│     端口: 23456                  │
├─────────────────────────────────┤
│       集成测试 (Integration)     │  ← 验证模块间协作
│     端口: 23456                  │
├─────────────────────────────────┤
│        单元测试 (Unit)           │  ← 验证单个函数/类
│     无需启动服务                  │
└─────────────────────────────────┘
```

### 4.2 测试框架

- **vitest** — 单元测试 + 集成测试 + 冒烟测试
- 测试文件放各 package 的 `tests/` 目录下

```
packages/core/
├── src/
└── tests/
    └── unit/                    # 单元测试
        └── networkDetector.test.ts

packages/server/
├── src/
└── tests/
    ├── integration/             # 集成测试
    │   └── networkRouter.integration.test.ts
    └── smoke/                   # 冒烟测试
        └── networkRouter.smoke.test.ts
```

### 4.3 测试命名规范

```typescript
describe('NetworkDetector', () => {
  describe('detect()', () => {
    it('should return intranet when DNS resolves to 10.x.x.x', async () => { ... })
    it('should return external when DNS resolution fails', async () => { ... })
  })
})
```

### 4.4 测试脚本

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

运行指定层级的测试：

```bash
# 全部测试
pnpm test

# 单元测试
npx vitest run packages/core/tests

# 集成测试
npx vitest run packages/server/tests/integration

# 冒烟测试
npx vitest run packages/server/tests/smoke
```

### 4.5 测试环境配置

集成测试和冒烟测试使用独立的测试配置：

```
tests/
├── vitest.config.ts             # 根级 vitest 配置
├── fixtures/
│   └── config.test.json         # 测试用配置（端口 23456）
└── helpers/
    └── server.ts                # 测试服务器启动/停止工具
```

### 4.6 各层测试职责

| 层级 | 范围 | 示例 |
|------|------|------|
| **单元测试** | 单个类/函数，mock 所有外部依赖 | NetworkDetector.detect() 的 DNS 解析逻辑 |
| **集成测试** | 多模块协作，启动真实服务（测试端口） | Server + NetworkDetector 生命周期 |
| **冒烟测试** | 端到端 HTTP API，验证路由注册正确 | GET /api/network-state 返回正确响应 |

## 5. 代码规范

### TypeScript

- 严格模式（项目已启用 `strict: true`）
- 优先使用 `interface` 定义类型，`type` 用于联合类型
- 导出显式类型，不使用 `any`（测试中除外）

### 提交信息

使用 Conventional Commits：

```
feat: add network-aware router
fix: fix DNS detection timeout
test: add unit tests for NetworkDetector
docs: update design document
refactor: extract config reload logic
chore: setup vitest
```

### 依赖管理

- 生产依赖安装到具体 package
- 开发/测试依赖安装到根目录或具体 package
- 使用 pnpm workspace 协议引用内部包：`"@CCR/shared": "workspace:*"`

## 6. 构建与发布

```bash
# 完整构建
pnpm build

# 单个 package 构建
pnpm build:core
pnpm build:server
pnpm build:ui

# 发布
pnpm release
```

## 7. 部署-观测循环

**核心原则：改了代码必须部署才能验证。**

### 7.1 本地部署流程

```bash
# 1. 构建变更的 package
pnpm build:core      # 如果改了 core
pnpm build:server    # 如果改了 server
pnpm build:ui        # 如果改了 UI（需要 vite build）

# 2. 构建 CLI（打包所有依赖）
npx esbuild packages/cli/src/cli.ts --bundle --platform=node --minify --outfile=packages/cli/dist/cli.js

# 3. 复制依赖文件
Copy-Item "packages/server/dist/tiktoken_bg.wasm" "packages/cli/dist/" -Force
Copy-Item "packages/ui/dist/index.html" "packages/cli/dist/" -Force

# 4. 复制到根目录 dist
Copy-Item "packages/cli/dist" "dist" -Recurse -Force

# 5. 重启服务
node dist/cli.js stop
Start-Sleep 2
Start-Process -FilePath "node" -ArgumentList "dist/cli.js","start" -WindowStyle Hidden
Start-Sleep 8
node dist/cli.js status
```

### 7.2 观测验证

```bash
# 查看日志（确认 LOG: true）
$logDir = Join-Path $env:USERPROFILE ".claude-code-router\logs"
$latestLog = Get-ChildItem $logDir -Filter "*.log" | Sort-Object LastWriteTime -Descending | Select-Object -First 1
Get-Content $latestLog.FullName -Tail 30

# 测试 API
Invoke-WebRequest -Uri "http://127.0.0.1:3456/api/network-state" -UseBasicParsing

# 打开 UI
Start-Process "http://127.0.0.1:3456/ui/"
```

### 7.3 问题定位原则

1. **加日志定位**：遇到问题先加详细日志，部署后再观测
2. **不要删日志**：调试日志保留在代码中，方便后续排查
3. **分层测试覆盖**：如果问题漏到部署阶段才发现，说明测试覆盖不足，需补充测试

### 7.4 已知的部署发现问题

| 问题 | 发现方式 | 根因 | 解决 |
|------|----------|------|------|
| DNS 切换后不检测 | 部署观测 | `dns.resolve4` 使用 c-ares resolver，进程启动时绑定 DNS 服务器，网络切换后不感知 | 改用 `dns.lookup`（调用系统 `getaddrinfo`） |
| 正则表达式转义错误 | 部署观测 | JSON 中 `intranetPattern` 写成 `"^10\\\\."` 导致正则变成 `/^10\\./` | JSON 中应写 `"^10\\."`（两个反斜杠） |

**教训**：单元测试 mock 了 DNS，绕过了真实 resolver 的行为。对于依赖系统行为的模块，集成测试应尽量用真实依赖而非 mock。