# CLAUDE.md

此文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。
此后所有问答的回复使用中文描述。

## 开发命令

### 构建
- `pnpm build` - 完整构建（打包 Canvas A2UI，编译 TS，生成 DTS，复制元数据）
- `pnpm ui:build` - 构建 Control UI 网页资源

### 运行
- `pnpm dev` - 以开发模式运行（直接启动 Node）
- `pnpm openclaw ...` - 通过 tsx 直接运行 TypeScript，无需构建
- `pnpm gateway:watch` - TypeScript 更改时自动重载网关
- `pnpm openclaw gateway --port 18789 --verbose` - 启动网关

### 测试
- `pnpm test` - 并行运行所有测试
- `pnpm test:watch` - 以监视模式运行测试
- `pnpm test:force path/to/test.test.ts` - 运行特定测试文件
- `pnpm test:coverage` - 运行测试并生成覆盖率报告
- `pnpm test:e2e` - 运行端到端测试

### 代码质量
- `pnpm check` - 运行所有检查（格式化 + 类型检查 + lint）
- `pnpm format` - 使用 oxfmt 格式化代码
- `pnpm lint` - 运行 oxlint（类型感知的代码检查）
- `pnpm lint:fix` - 自动修复 lint 问题并格式化
- `pnpm tsgo` - 类型检查测试文件（tsgo 包装器）

### UI 开发
- `pnpm ui:dev` - 启动 Control UI 开发服务器
- `pnpm ui:install` - 安装 UI 依赖

## 项目架构

OpenClaw 是一个**多渠道 AI 网关**，具有可扩展的消息集成功能。架构采用中心辐射型模型：

```
渠道（WhatsApp, Telegram, Slack, Discord, ...）
                    │
                    ▼
              ┌──────────────────────────────────┐
              │         Gateway (WS 18789)      │
              │       (控制平面)                 │
              └─────────────┬──────────────────┘
                            │
        ┌───────────────────┼──────────────────┐
        │                   │                  │
    Pi Agent (RPC)      CLI/Gateway          节点
    (工具流式传输)      方法              (macOS/iOS/Android)
        │                   │
    工具（bash, browser, canvas, web, sessions, ...）
```

### 核心目录

- **`src/gateway/`** - WebSocket 控制平面、会话管理、协议处理
  - `server.impl.ts` - 主网关服务器入口
  - `protocol/` - Gateway WebSocket 协议模式和类型
  - `server-methods/` - RPC 方法处理器（agent、chat、config、browser 等）

- **`src/acp/`** - Agent Client Protocol 层（Pi agent RPC 集成）
  - `server.ts` - ACP 网关服务器（基于 stdio 的 Pi agent）
  - `translator.ts` - Gateway ↔ Pi agent 协议转换

- **`src/agents/`** - Agent 运行时集成、工具、认证配置
  - `pi-embedded-runner.ts` - 嵌入式 Pi agent 运行时
  - `pi-tools.ts` - 暴露给 agent 的工具定义
  - `tools/` - 各个工具实现（browser、canvas、web、sessions、cron 等）
  - `auth-profiles/` - OAuth/API 密钥配置轮换

- **`src/channels/`** - 渠道插件系统
  - `plugins/` - 渠道插件实现（WhatsApp、Telegram、Slack、Discord 等）
  - 每个渠道实现：`ChannelGatewayAdapter`、`ChannelMessagingAdapter`、`ChannelOutboundAdapter`
  - `registry.ts` - 渠道注册表和元数据

- **`src/plugin-sdk/`** - 用于创建渠道插件和扩展的公共 SDK

- **`src/config/`** - 配置加载、验证、迁移、会话存储

- **`src/cli/`**、**`src/commands/`** - CLI 入口点和命令处理器

- **`src/wizard/`** - 交互式入门向导

- **`src/browser/`** - 基于 CDP 的浏览器控制（Chrome/Chromium）

- **`src/tui/`** - 终端 UI 组件

### Agent 工具系统

工具在 `src/agents/tools/` 中定义，并通过 ACP 层暴露给 Pi agent。工具使用 TypeBox 模式并实现：
- 输入验证模式
- 执行逻辑（同步或异步）
- 通过上下文访问 Gateway/Node 客户端

主要工具：`bash`（命令执行）、`browser`（CDP 控制）、`canvas`（A2UI）、`web`（fetch/search）、`sessions_*`（agent 间通信）、`cron`、`nodes` 等。

### 渠道插件架构

渠道是实现标准化接口的插件：
- **消息适配器** - 接收和发送消息
- **网关适配器** - 网关集成钩子
- **出站适配器** - 向目标发送消息
- **认证适配器** - 登录/登出流程
- **配置适配器** - 渠道特定配置模式

渠道通过 `src/channels/plugins/catalog.ts` 注册并动态加载。

### 会话模型

会话表示隔离的 agent 对话：
- `main` - 主要的直接对用户会话
- 基于群组 - 每个群组/聊天一个会话
- 会话密钥：`main`、`whatsapp+1234567890`、`telegram+123456789` 等
- 配置：`agents.defaults.workspace`（默认：`~/.openclaw/workspace`）

### WebSocket 协议（Gateway）

Gateway 在端口 18789（默认）上公开 WebSocket API：
- 双向事件：`hello`、`chat`、`agent_event`、`presence`、`log` 等
- RPC 风格方法：`agent.chat`、`sessions.list`、`browser.navigate` 等
- 认证：基于令牌或密码（配置：`gateway.auth`）

### 配置

配置文件：`~/.openclaw/openclaw.json`（或 Nix 模式）
- Agent 模型：`agent.model`（默认：anthropic/claude-opus-4-6）
- 工作空间：`agents.defaults.workspace`
- 渠道：`channels.whatsapp`、`channels.telegram`、`channels.slack` 等
- 网关：`gateway.port`、`gateway.bind`、`gateway.auth`

使用 `openclaw doctor` 验证和修复配置。

### Control UI

Control UI（基于 Lit 的网页 UI）从 Gateway 提供：
- 资源打包到 `dist/control-ui/`
- 开发服务器：`pnpm ui:dev`
- 生产环境：通过 Gateway HTTP 服务器的 `GET /` 提供

### 测试约定

- 单元测试：`*.test.ts`（通过 vitest 运行）
- 实时测试：`*.live.test.ts`（需要 `OPENCLAW_LIVE_TEST=1`，真实凭据）
- E2E 测试：`*.e2e.test.ts`（通过 `pnpm test:e2e` 运行）
- 测试工具在 `test/` 和 `test/helpers/` 中
- 使用 vitest 的 `describe`、`it`、`expect`、`beforeAll`、`afterAll`
- 模拟文件 I/O 和外部服务

## 重要说明

- **需要 Node >= 22.12.0**
- 使用 `pnpm` 作为包管理器（pnpm 配置要求）
- TypeScript 文件使用 `.js` 导入（ESM 模式，tsdown 编译到 `dist/` 中的 `.js`）
- 在开发中使用 `tsx` 直接运行 TS
- Gateway 是中央控制平面 - 所有客户端通过 WebSocket 连接
- 渠道是隔离的插件 - 通过插件系统添加新渠道
- Pi agent 通过 RPC 模式嵌入运行，支持工具/块流式传输
- 会话状态持久化在 `~/.openclaw/sessions.db`（SQLite）
- 工作空间文件（AGENTS.md、SOUL.md、TOOLS.md、skills/）被注入到 agent 上下文中

## Control UI 装饰器

Control UI 使用 Lit 的**旧式装饰器**（当前 Rollup 解析不支持 `accessor` 字段）：

```ts
@state() foo = "bar";
@property({ type: Number }) count = 0;
```

向 UI 组件添加响应式字段时，请保持旧式装饰器风格。
