# Shannon Code Wiki

## 1. 项目概述

Shannon 是一个自主的白盒 AI 渗透测试工具，专为 Web 应用和 API 设计。它通过分析源代码、识别攻击向量并执行实际的漏洞利用来证明漏洞的存在，确保只有具有可工作的概念证明的漏洞才会被包含在最终报告中。

### 1.1 核心价值

- **填补安全测试缺口**：传统渗透测试每年只进行一次，而 Shannon 可以在每次构建或发布时运行，关闭这一安全缺口。
- **完全自主操作**：从 2FA/TOTP 登录到浏览器导航、漏洞利用和报告生成，无需人工干预。
- **可重现的漏洞利用证明**：最终报告只包含已验证的可利用发现，带有可复制粘贴的概念证明。
- **代码感知动态测试**：分析源代码以指导攻击策略，然后通过实时浏览器和基于 CLI 的漏洞利用验证发现。

### 1.2 产品系列

| 版本 | 许可证 | 最佳用途 |
|------|--------|----------|
| **Shannon Lite** | AGPL-3.0 | 本地测试您自己的应用程序。 |
| **Shannon Pro** | 商业 | 需要单一 AppSec 平台（SAST、SCA、密钥、业务逻辑测试、自主渗透测试）的组织，具有 CI/CD 集成和自托管部署。 |

## 2. 项目架构

Shannon 使用多代理架构，结合白盒源代码分析和黑盒动态漏洞利用，由一个协调器管理五个阶段。架构设计旨在通过"无漏洞利用，无报告"的策略最小化误报。

### 2.1 系统架构图

```
        ┌──────────────────────┐
        │   Pre-Reconnaissance │
        │  (nmap, subfinder,   │
        │  whatweb, code scan) │
        └──────────┬───────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │   Reconnaissance     │
        │  (attack surface     │
        │   mapping)           │
        └──────────┬───────────┘
                   │
                   ▼
        ┌──────────┴───────────┐
        │          │           │
        ▼          ▼           ▼
  ┌───────────┐ ┌───────────┐ ┌───────────┐
  │ Vuln      │ │ Vuln      │ │   ...     │
  │(Injection)│ │  (XSS)    │ │           │
  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
        │              │             │
        ▼              ▼             ▼
  ┌───────────┐ ┌───────────┐ ┌───────────┐
  │ Exploit   │ │ Exploit   │ │   ...     │
  │(Injection)│ │  (XSS)    │ │           │
  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
        │              │             │
        └──────┬───────┴─────────────┘
               │
               ▼
        ┌──────────────────────┐
        │      Reporting       │
        └──────────────────────┘
```

### 2.2 架构组件

#### 2.2.1 命令行接口 (CLI)

CLI 模块负责用户交互、配置管理和启动渗透测试工作流。它支持两种运行模式：

- **本地模式**：从克隆的仓库运行，本地构建，使用 `./workspaces/` 目录
- **NPX 模式**：通过 npx 运行，从 Docker Hub 拉取镜像，使用 `~/.shannon/` 目录

#### 2.2.2 工作器 (Worker)

Worker 模块是核心执行组件，负责运行渗透测试工作流。它使用 Temporal 工作流引擎来管理任务执行和状态跟踪。

#### 2.2.3 AI 推理引擎

Shannon 使用 Anthropic 的 Claude Agent SDK 作为其推理引擎，用于分析代码、制定攻击策略和执行漏洞利用。

#### 2.2.4 工作流管理

使用 Temporal 工作流引擎来协调多阶段渗透测试过程，确保任务的可靠执行和状态管理。

## 3. 目录结构

```
├── apps/
│   ├── cli/            # 命令行工具
│   │   ├── src/        # CLI 源代码
│   │   └── infra/      # 基础设施配置
│   └── worker/         # 核心工作组件
│       ├── src/        # Worker 源代码
│       ├── prompts/    # AI 提示模板
│       └── configs/    # 配置文件
├── assets/             # 资源文件
├── repos/              # 存储库目录
├── sample-reports/     # 示例报告
├── workspaces/         # 工作空间目录
├── .env.example        # 环境变量示例
├── docker-compose.yml  # Docker 配置
└── package.json        # 项目配置
```

### 3.1 主要目录说明

| 目录 | 说明 | 关键文件 |
|------|------|----------|
| apps/cli/src | CLI 源代码，包含命令实现 | [index.ts](file:///workspace/apps/cli/src/index.ts) |
| apps/cli/infra | 基础设施配置，包含 Docker 配置 | [compose.yml](file:///workspace/apps/cli/infra/compose.yml) |
| apps/worker/src | Worker 源代码，包含核心功能 | [temporal/worker.ts](file:///workspace/apps/worker/src/temporal/worker.ts) |
| apps/worker/prompts | AI 提示模板，用于指导 AI 执行不同阶段的任务 | 多个提示文件 |
| apps/worker/configs | 配置文件，包含配置模式和示例 | [example-config.yaml](file:///workspace/apps/worker/configs/example-config.yaml) |
| workspaces/ | 工作空间目录，存储扫描结果和状态 | 自动生成的工作空间文件夹 |
| sample-reports/ | 示例报告，展示 Shannon 的能力 | 多个示例报告文件 |

## 4. 核心模块

### 4.1 CLI 模块

CLI 模块提供用户交互界面，负责解析命令行参数、配置管理和启动渗透测试工作流。

#### 4.1.1 主要命令

| 命令 | 说明 | 参数 |
|------|------|------|
| start | 启动渗透测试扫描 | --url <url>, --repo <path>, --config <path>, --output <path>, --workspace <name> |
| stop | 停止所有容器 | --clean |
| logs | 查看工作流日志 | <workspace> |
| workspaces | 列出所有工作空间 | 无 |
| status | 显示运行中的工作器 | 无 |
| setup | 配置凭证 (仅 NPX 模式) | 无 |
| build | 构建工作器镜像 (仅本地模式) | --no-cache |
| uninstall | 移除 ~/.shannon/ 和所有数据 (仅 NPX 模式) | 无 |

#### 4.1.2 关键函数

- **parseStartArgs**：解析 start 命令的参数，确保提供了必要的 URL 和仓库路径
- **start**：启动渗透测试工作流，包括构建 Docker 镜像、启动容器和提交工作流
- **stop**：停止所有 Docker 容器，可选清理工作空间
- **logs**：查看指定工作空间的日志
- **workspaces**：列出所有工作空间及其状态

### 4.2 Worker 模块

Worker 模块是核心执行组件，负责运行渗透测试工作流。它使用 Temporal 工作流引擎来管理任务执行和状态跟踪。

#### 4.2.1 主要组件

| 组件 | 说明 | 关键文件 |
|------|------|----------|
| 工作流 | 定义渗透测试的各个阶段和执行流程 | [temporal/workflows.ts](file:///workspace/apps/worker/src/temporal/workflows.ts) |
| 活动 | 执行具体的渗透测试任务 | [temporal/activities.ts](file:///workspace/apps/worker/src/temporal/activities.ts) |
| AI 执行器 | 与 Anthropic Claude 模型交互 | [ai/claude-executor.ts](file:///workspace/apps/worker/src/ai/claude-executor.ts) |
| 配置解析器 | 解析和验证配置文件 | [config-parser.ts](file:///workspace/apps/worker/src/config-parser.ts) |
| 会话管理 | 管理扫描会话和状态 | [session-manager.ts](file:///workspace/apps/worker/src/session-manager.ts) |

#### 4.2.2 关键函数

- **run**：Worker 的主入口点，解析 CLI 参数、连接到 Temporal 服务器、创建工作器、提交工作流并等待结果
- **resolveWorkspace**：解析工作空间，处理恢复现有工作空间的逻辑
- **buildPipelineInput**：构建管道输入，包含目标 URL、仓库路径和配置信息
- **waitForWorkflowResult**：等待工作流完成，显示进度和结果
- **pentestPipelineWorkflow**：渗透测试工作流的主函数，协调各个阶段的执行

### 4.3 AI 模块

AI 模块负责与 Anthropic Claude 模型交互，指导 AI 执行不同阶段的任务。

#### 4.3.1 主要组件

| 组件 | 说明 | 关键文件 |
|------|------|----------|
| Claude 执行器 | 与 Claude 模型交互，执行提示 | [ai/claude-executor.ts](file:///workspace/apps/worker/src/ai/claude-executor.ts) |
| 消息处理器 | 处理 AI 消息和响应 | [ai/message-handlers.ts](file:///workspace/apps/worker/src/ai/message-handlers.ts) |
| 输出格式化器 | 格式化 AI 输出 | [ai/output-formatters.ts](file:///workspace/apps/worker/src/ai/output-formatters.ts) |
| 进度管理器 | 管理 AI 执行进度 | [ai/progress-manager.ts](file:///workspace/apps/worker/src/ai/progress-manager.ts) |

### 4.4 审计模块

审计模块负责记录和跟踪渗透测试的执行情况。

#### 4.4.1 主要组件

| 组件 | 说明 | 关键文件 |
|------|------|----------|
| 审计会话 | 管理审计会话 | [audit/audit-session.ts](file:///workspace/apps/worker/src/audit/audit-session.ts) |
| 日志流 | 管理日志流 | [audit/log-stream.ts](file:///workspace/apps/worker/src/audit/log-stream.ts) |
| 工作流日志器 | 记录工作流执行情况 | [audit/workflow-logger.ts](file:///workspace/apps/worker/src/audit/workflow-logger.ts) |

## 5. 工作流程

Shannon 的渗透测试过程分为五个阶段：

### 5.1 预侦察阶段 (Pre-Reconnaissance)

- 使用 nmap、subfinder 和 whatweb 等工具进行外部扫描，识别目标的基础设施和技术栈
- 同时分析源代码，识别应用框架、入口点和潜在的攻击面

### 5.2 侦察阶段 (Reconnaissance)

- 根据预侦察结果构建全面的攻击面地图
- 通过浏览器自动化进行实时应用探索，将代码级见解与实际行为相关联
- 生成详细的所有入口点、API 端点和认证机制的地图

### 5.3 漏洞分析阶段 (Vulnerability Analysis)

- 并行运行 5 个专门的代理，针对不同的 OWASP 类别（注入、XSS、认证、授权、SSRF）
- 对于注入和 SSRF 等漏洞，代理执行结构化的数据流分析，跟踪用户输入到危险的接收器
- 生成**假设的可利用路径**列表，传递给验证阶段

### 5.4 漏洞利用阶段 (Exploitation)

- 继续并行工作以保持速度，专注于将假设转化为证明
- 专门的漏洞利用代理接收假设的路径，并尝试使用浏览器自动化、命令行工具和自定义脚本执行实际攻击
- 强制执行严格的**"无漏洞利用，无报告"**策略：如果假设无法成功利用以展示影响，则将其作为误报丢弃

### 5.5 报告阶段 (Reporting)

- 编译所有验证过的发现，生成专业的可操作报告
- 代理整合侦察数据和成功的漏洞利用证据，清理任何噪音或幻觉产物
- 只包含已验证的漏洞，带有**可重现的、可复制粘贴的概念证明**，提供专注于已证明风险的最终渗透测试级报告

## 6. 关键 API 和类

### 6.1 CLI 模块 API

#### 6.1.1 start 命令

```typescript
async function start({ url, repo, config, workspace, output, pipelineTesting, router, version }: {
  url: string;
  repo: string;
  config?: string;
  workspace?: string;
  output?: string;
  pipelineTesting: boolean;
  router: boolean;
  version: string;
}): Promise<void>
```

启动渗透测试扫描，参数包括目标 URL、仓库路径、配置文件路径、工作空间名称、输出目录、管道测试模式和路由器模式。

#### 6.1.2 stop 命令

```typescript
function stop(clean: boolean): void
```

停止所有容器，可选清理工作空间。

#### 6.1.3 logs 命令

```typescript
function logs(workspaceId: string): void
```

查看指定工作空间的日志。

#### 6.1.4 workspaces 命令

```typescript
function workspaces(version: string): void
```

列出所有工作空间及其状态。

### 6.2 Worker 模块 API

#### 6.2.1 run 函数

```typescript
async function run(): Promise<void>
```

Worker 的主入口点，解析 CLI 参数、连接到 Temporal 服务器、创建工作器、提交工作流并等待结果。

#### 6.2.2 pentestPipelineWorkflow 函数

```typescript
async function pentestPipelineWorkflow(input: PipelineInput): Promise<PipelineState>
```

渗透测试工作流的主函数，协调各个阶段的执行。

#### 6.2.3 resolveWorkspace 函数

```typescript
async function resolveWorkspace(client: Client, args: CliArgs): Promise<WorkspaceResolution>
```

解析工作空间，处理恢复现有工作空间的逻辑。

#### 6.2.4 loadPipelineConfig 函数

```typescript
async function loadPipelineConfig(configPath: string | undefined): Promise<PipelineConfig>
```

加载和解析管道配置。

### 6.3 AI 模块 API

#### 6.3.1 ClaudeExecutor 类

```typescript
class ClaudeExecutor {
  constructor(options: ClaudeExecutorOptions);
  async execute(prompt: string, options?: ExecuteOptions): Promise<ExecuteResult>;
}
```

与 Claude 模型交互，执行提示并返回结果。

## 7. 依赖关系

### 7.1 核心依赖

| 依赖 | 版本 | 用途 | 模块 |
|------|------|------|------|
| @anthropic-ai/claude-agent-sdk | catalog: | AI 推理引擎 | worker |
| @temporalio/activity | ^1.11.0 | Temporal 活动 | worker |
| @temporalio/client | ^1.11.0 | Temporal 客户端 | worker |
| @temporalio/worker | ^1.11.0 | Temporal 工作器 | worker |
| @temporalio/workflow | ^1.11.0 | Temporal 工作流 | worker |
| @clack/prompts | ^1.1.0 | 命令行提示 | cli |
| chokidar | ^5.0.0 | 文件系统监视 | cli |
| dotenv | ^17.3.1 | 环境变量加载 | cli |
| smol-toml | ^1.6.1 | TOML 解析 | cli |
| ajv | ^8.12.0 | JSON 模式验证 | worker |
| ajv-formats | ^2.1.1 | JSON 模式格式 | worker |
| js-yaml | ^4.1.0 | YAML 解析 | worker |
| zod | ^4.3.6 | 类型验证 | worker |
| zx | ^8.0.0 | 命令执行 | worker |

### 7.2 开发依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| @biomejs/biome | ^2.0.0 | 代码格式化和 linting |
| @types/node | ^25.0.3 | Node.js 类型定义 |
| turbo | ^2.5.0 | 构建系统 |
| typescript | ^5.9.3 | TypeScript 编译器 |
| tsdown | ^0.21.5 | TypeScript 构建工具 |
| @types/js-yaml | ^4.0.9 | js-yaml 类型定义 |

## 8. 配置与部署

### 8.1 环境变量

| 环境变量 | 说明 | 必需 | 默认值 |
|----------|------|------|--------|
| ANTHROPIC_API_KEY | Anthropic API 密钥 | 是（除非使用其他 AI 提供商） | 无 |
| CLAUDE_CODE_MAX_OUTPUT_TOKENS | Claude 输出令牌限制 | 否 | 64000 |
| TEMPORAL_ADDRESS | Temporal 服务器地址 | 否 | localhost:7233 |
| CLAUDE_CODE_USE_BEDROCK | 使用 AWS Bedrock | 否 | 0 |
| AWS_REGION | AWS 区域 | 否 | us-east-1 |
| AWS_BEARER_TOKEN_BEDROCK | AWS Bedrock 令牌 | 否 | 无 |
| CLAUDE_CODE_USE_VERTEX | 使用 Google Vertex AI | 否 | 0 |
| CLOUD_ML_REGION | Google Cloud 区域 | 否 | us-east5 |
| ANTHROPIC_VERTEX_PROJECT_ID | Google Cloud 项目 ID | 否 | 无 |
| GOOGLE_APPLICATION_CREDENTIALS | Google 服务账户密钥文件路径 | 否 | 无 |
| ANTHROPIC_BASE_URL | 自定义 Anthropic 兼容端点 | 否 | 无 |
| ANTHROPIC_AUTH_TOKEN | 自定义端点认证令牌 | 否 | 无 |
| OPENAI_API_KEY | OpenAI API 密钥（实验性） | 否 | 无 |
| OPENROUTER_API_KEY | OpenRouter API 密钥（实验性） | 否 | 无 |
| ROUTER_DEFAULT | 默认路由器提供商和模型（实验性） | 否 | 无 |

### 8.2 配置文件

Shannon 支持使用 YAML 配置文件来自定义扫描行为。配置文件包括以下部分：

#### 8.2.1 基本配置

```yaml
# 可选：描述目标环境（最多 500 字符）
description: "Next.js e-commerce app on PostgreSQL. Local dev environment — .env files contain local-only credentials, not deployed to production."

authentication:
  login_type: form
  login_url: "https://your-app.com/login"
  credentials:
    username: "test@example.com"
    password: "yourpassword"
    totp_secret: "LB2E2RX7XFHSTGCK"  # 可选，用于 2FA

  login_flow:
    - "Type $username into the email field"
    - "Type $password into the password field"
    - "Click the 'Sign In' button"

  success_condition:
    type: url_contains
    value: "/dashboard"

rules:
  avoid:
    - description: "AI should avoid testing logout functionality"
      type: path
      url_path: "/logout"

  focus:
    - description: "AI should emphasize testing API endpoints"
      type: path
      url_path: "/api"
```

#### 8.2.2 管道配置

```yaml
pipeline:
  retry_preset: subscription          # 扩展最大退避到 6 小时，100 次重试
  max_concurrent_pipelines: 2         # 一次运行 2 个（共 5 个）管道（减少突发 API 使用）
```

### 8.3 部署方式

#### 8.3.1 NPX 模式（推荐）

```bash
# 1. 配置凭证（交互式向导 — 一次性设置）
npx @keygraph/shannon setup

# 或直接导出环境变量
export ANTHROPIC_API_KEY=your-api-key

# 2. 运行渗透测试
npx @keygraph/shannon start -u https://your-app.com -r /path/to/your-repo
```

#### 8.3.2 克隆和构建模式

```bash
# 1. 克隆 Shannon
git clone https://github.com/KeygraphHQ/shannon.git
cd shannon

# 2. 配置凭证（选择一种方法）

# 选项 A：创建 .env 文件
cat > .env << 'EOF'
ANTHROPIC_API_KEY=your-api-key
CLAUDE_CODE_MAX_OUTPUT_TOKENS=64000
EOF

# 选项 B：导出环境变量
export ANTHROPIC_API_KEY="your-api-key"              # 或 CLAUDE_CODE_OAUTH_TOKEN
export CLAUDE_CODE_MAX_OUTPUT_TOKENS=64000           # 推荐

# 3. 安装依赖并构建
pnpm install
pnpm build

# 4. 运行渗透测试
./shannon start -u https://your-app.com -r /path/to/your-repo
```

## 9. 运行与使用

### 9.1 基本使用

```bash
# 基本渗透测试
npx @keygraph/shannon start -u https://example.com -r /path/to/repo

# 使用配置文件
npx @keygraph/shannon start -u https://example.com -r /path/to/repo -c /path/to/my-config.yaml

# 自定义输出目录
npx @keygraph/shannon start -u https://example.com -r /path/to/repo -o ./my-reports

# 命名工作空间
npx @keygraph/shannon start -u https://example.com -r /path/to/repo -w q1-audit

# 列出所有工作空间
npx @keygraph/shannon workspaces
```

### 9.2 监控进度

```bash
npx @keygraph/shannon logs <workspace>
npx @keygraph/shannon status
```

打开 Temporal Web UI 进行详细监控：

```bash
open http://localhost:8233
```

### 9.3 停止 Shannon

```bash
npx @keygraph/shannon stop
npx @keygraph/shannon stop --clean
npx @keygraph/shannon uninstall
```

### 9.4 工作空间和恢复

Shannon 支持**工作空间**，允许您在不重新运行已完成代理的情况下恢复中断或失败的运行。

**工作原理：**

- 每次运行都会创建一个工作空间（默认自动命名，例如 `example-com_shannon-1771007534808`）
- 工作空间存储在 `./workspaces/`（本地模式）或 `~/.shannon/workspaces/`（npx 模式）
- 使用 `-w <name>` 为您的运行指定自定义名称，以便于引用
- 要恢复任何运行，通过 `-w` 传递其工作空间名称 — Shannon 会检测哪些代理已成功完成，并从上次中断的地方继续
- 每个代理的进度通过 git 提交进行检查点保存，因此恢复的运行从干净、已验证的状态开始

```bash
# 使用命名工作空间开始
npx @keygraph/shannon start -u https://example.com -r /path/to/repo -w my-audit

# 恢复相同的工作空间（跳过已完成的代理）
npx @keygraph/shannon start -u https://example.com -r /path/to/repo -w my-audit

# 从之前运行的自动命名工作空间恢复
npx @keygraph/shannon start -u https://example.com -r /path/to/repo -w example-com_shannon-1771007534808

# 列出所有工作空间及其状态
npx @keygraph/shannon workspaces
```

## 10. 输出与结果

所有结果都保存到工作空间目录：`./workspaces/`（本地模式）或 `~/.shannon/workspaces/`（npx 模式）。使用 `-o <path>` 可在运行完成后将交付物复制到自定义输出目录。

### 10.1 输出结构

```text
workspaces/{hostname}_{sessionId}/
├── session.json          # 指标和会话数据
├── workflow.log          # 人类可读的工作流日志
├── agents/               # 每个代理的执行日志
├── prompts/              # 提示快照，用于可重现性
└── deliverables/
    └── comprehensive_security_assessment_report.md   # 最终综合安全报告
```

### 10.2 示例报告

Shannon 生成专业的安全评估报告，包括：

- 执行摘要
- 发现的漏洞详细信息
- 可重现的概念证明
- 修复建议
- 风险评估

示例报告可在 [sample-reports/](file:///workspace/sample-reports/) 目录中找到。

## 11. 安全注意事项

### 11.1 潜在的变异效应

Shannon 不是被动扫描器。漏洞利用代理旨在**主动执行攻击**以确认漏洞。此过程可能对目标应用及其数据产生变异效应。

> **警告：请勿在生产环境中运行 Shannon。**
> 
> - 它仅用于沙盒、暂存或本地开发环境，其中数据完整性不是问题。
> - 潜在的变异效应包括但不限于：创建新用户、修改或删除数据、泄露测试账户、触发注入攻击的意外副作用。

### 11.2 法律和道德使用

Shannon 仅用于合法的安全审计目的。

> **注意：在运行 Shannon 之前，您必须获得目标系统所有者的明确书面授权。**
> 
> 未经授权扫描和利用您不拥有的系统是非法的，可能会根据《计算机欺诈和滥用法案》(CFAA) 等法律被起诉。Keygraph 不对 Shannon 的任何滥用负责。

### 11.3 LLM 和自动化注意事项

- **需要验证**：虽然我们的"通过利用证明"方法已经进行了大量工程设计以消除误报，但底层 LLM 仍可能在最终报告中生成幻觉或支持不足的内容。**人类监督对于验证所有报告发现的合法性和严重性至关重要**。
- **全面性**：由于 LLM 上下文窗口的固有局限性，Shannon Lite 的分析可能不详尽。对于整个代码库的更全面、基于图的分析，**Shannon Pro** 利用其先进的数据流分析引擎来确保更深入和更全面的覆盖。

### 11.4 分析范围

- **目标漏洞**：当前版本的 Shannon Lite 专门针对以下类别的**可利用**漏洞：
  - 身份验证和授权破解
  - 注入
  - 跨站脚本 (XSS)
  - 服务器端请求伪造 (SSRF)
- **Shannon Lite 不涵盖的内容**：此列表并非所有潜在安全风险的详尽列表。Shannon Lite 的"通过利用证明"模型意味着它不会报告无法主动利用的问题，例如易受攻击的第三方库或不安全的配置。这些类型的深度静态分析发现是 **Shannon Pro** 中高级分析引擎的核心重点。

## 12. 总结与亮点回顾

Shannon 是一个强大的 AI 驱动的渗透测试工具，通过结合白盒源代码分析和黑盒动态漏洞利用，提供全面的安全评估。其主要亮点包括：

1. **完全自主操作**：从登录到报告生成，无需人工干预
2. **可重现的漏洞利用证明**：只报告已验证的可利用漏洞
3. **并行处理**：漏洞分析和利用阶段并行运行，提高效率
4. **代码感知动态测试**：分析源代码以指导攻击策略
5. **集成安全工具**：利用 Nmap、Subfinder、WhatWeb 和 Schemathesis 等工具
6. **工作空间和恢复**：支持恢复中断的运行，节省时间和资源
7. **多种 AI 提供商支持**：Anthropic、AWS Bedrock、Google Vertex AI，以及实验性的 OpenAI/Gemini

Shannon 填补了传统渗透测试的缺口，为开发团队提供了一种在每次构建或发布时运行自动化安全测试的方法，确保漏洞在到达生产环境之前被发现和修复。

## 13. 后续步骤

1. **安装和配置**：按照部署指南安装和配置 Shannon
2. **准备目标应用**：确保您有目标应用的源代码和运行实例
3. **运行首次扫描**：使用基本命令运行首次渗透测试
4. **分析报告**：查看生成的报告，验证发现的漏洞
5. **集成到 CI/CD**：考虑将 Shannon 集成到您的 CI/CD 流程中，以便在每次构建时自动运行安全测试
6. **探索高级功能**：尝试使用配置文件、工作空间和其他高级功能

通过使用 Shannon，您可以显著提高应用程序的安全性，减少漏洞到达生产环境的风险。