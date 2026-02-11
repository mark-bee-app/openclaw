# OpenClaw Gateway 协议完整调研报告

> **版本**: 1.0
> **日期**: 2026-02-12
> **协议版本**: 3
> **作者**: OpenClaw Protocol Research Team

---

## 目录

1. [架构概览](#1-架构概览)
2. [协议基础](#2-协议基础)
3. [认证机制](#3-认证机制)
4. [消息帧格式](#4-消息帧格式)
5. [RPC 方法详解](#5-rpc-方法详解)
6. [事件系统](#6-事件系统)
7. [权限模型](#7-权限模型)
8. [会话管理](#8-会话管理)
9. [客户端实现指南](#9-客户端实现指南)
10. [附录](#10-附录)

---

## 1. 架构概览

### 1.1 系统架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         OpenClaw Gateway                          │
│                         (WebSocket :18789)                        │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐      │
│  │   Protocol    │  │     Auth      │  │    Router     │      │
│  │   Handler    │  │    Manager    │  │              │      │
│  └───────┬──────┘  └───────┬──────┘  └───────┬──────┘      │
│          │                 │                 │                 │
│  ┌───────┴─────────────────┴─────────────────┴──────┐      │
│  │              RPC Method Dispatchers              │      │
│  │  agent | chat | sessions | nodes | channels ...│      │
│  └───────┬───────────────────────────────────────┘      │
│          │                                                 │
│  ┌───────┴───────────────────────────────────────┐      │
│  │              Business Logic Layer              │      │
│  │  - Agent Runtime (Pi Agent)                 │      │
│  │  - Session Manager                         │      │
│  │  - Channel Adapters                        │      │
│  │  - Tool Executors                         │      │
│  └──────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
    ┌────▼────┐     ┌────▼────┐    ┌────▼────┐
    │   CLI   │     │  Web UI │    │  Node   │
    │ Client  │     │ Client  │    │ Client  │
    └─────────┘     └─────────┘    └─────────┘
```

### 1.2 连接流程

```
客户端                    服务器
  │                         │
  │  ──── WebSocket 握手 ──→│
  │←───────────────────────── │  连接建立
  │                         │
  │  ──── connect 请求 ───→│
  │        (含认证凭据)       │
  │                         │  验证认证...
  │←────── hello-ok ─────────│
  │        (协议版本、能力)    │
  │                         │
  │  ──── agent RPC ─────→│  开始通信
  │←───── 响应/事件 ────────│
  │                         │
```

### 1.3 目录结构

```
src/gateway/
├── protocol/                   # 协议定义
│   ├── schema/                # TypeBox JSON Schema
│   │   ├── frames.ts         # 核心帧定义
│   │   ├── agent.ts          # Agent 相关
│   │   ├── sessions.ts       # 会话相关
│   │   ├── logs-chat.ts      # Chat 相关
│   │   ├── channels.ts       # 渠道相关
│   │   ├── nodes.ts          # 节点相关
│   │   ├── devices.ts        # 设备相关
│   │   ├── error-codes.ts    # 错误码
│   │   └── ...
│   └── index.ts              # 导出和验证器
├── server-methods/           # RPC 方法实现
│   ├── agent.ts             # agent, agent.identity.get, agent.wait
│   ├── chat.ts              # chat.history/send/abort/inject
│   ├── sessions.ts          # sessions.*
│   ├── config.ts            # config.*
│   ├── agents.ts            # agents.*
│   ├── nodes.ts             # node.*
│   ├── devices.ts           # device.*
│   ├── channels.ts          # channels.*
│   └── ...
├── auth.ts                  # 认证逻辑
├── client.ts                # 客户端类 (GatewayClient)
└── server-methods.ts        # 方法路由和权限验证
```

---

## 2. 协议基础

### 2.1 协议版本

当前协议版本为 **3**，在 `src/gateway/protocol/schema/protocol-schemas.ts` 中定义：

```typescript
export const PROTOCOL_VERSION = 3 as const;
```

### 2.2 传输层

- **协议**: WebSocket
- **默认端口**: 18789
- **默认地址**: `ws://127.0.0.1:18789`
- **最大 Payload**: 25MB (可配置)
- **心跳间隔**: 30秒 (可配置)

### 2.3 数据格式

所有消息使用 **JSON** 格式，Schema 通过 **TypeBox** 定义，确保类型安全。

### 2.4 代码位置

| 组件 | 文件路径 |
|------|---------|
| 协议常量 | `src/gateway/protocol/schema/protocol-schemas.ts` |
| 核心帧 | `src/gateway/protocol/schema/frames.ts` |
| 基础类型 | `src/gateway/protocol/schema/primitives.ts` |
| 协议入口 | `src/gateway/protocol/index.ts` |

---

## 3. 认证机制

### 3.1 认证模式概览

Gateway 支持四种认证模式：

| 模式 | 描述 | 配置项 |
|------|------|--------|
| **Token** | 预共享令牌 | `gateway.auth.token` |
| **Password** | 密码认证 | `gateway.auth.password` |
| **Tailscale** | 通过 Tailscale 代理 | `gateway.auth.allowTailscale` |
| **Device Token** | 设备签名 + 令牌轮换 | 自动管理 |

### 3.2 连接参数 (ConnectParams)

完整定义位于 `src/gateway/protocol/schema/frames.ts`:

```typescript
export const ConnectParamsSchema = Type.Object({
  minProtocol: Type.Integer({ minimum: 1 }),
  maxProtocol: Type.Integer({ minimum: 1 }),
  client: Type.Object({
    id: GatewayClientIdSchema,           // 客户端标识
    displayName: Type.Optional(NonEmptyString),
    version: NonEmptyString,             // 客户端版本
    platform: NonEmptyString,            // 运行平台
    deviceFamily: Type.Optional(NonEmptyString),
    mode: GatewayClientModeSchema,       // backend | ui | probe | node
    instanceId: Type.Optional(NonEmptyString),
  }),
  caps: Type.Optional(Type.Array(NonEmptyString)),  // 客户端能力
  commands: Type.Optional(Type.Array(NonEmptyString)),
  permissions: Type.Optional(Type.Record(NonEmptyString, Type.Boolean())),
  pathEnv: Type.Optional(Type.String()),
  role: Type.Optional(NonEmptyString),   // operator | node
  scopes: Type.Optional(Type.Array(NonEmptyString)),
  device: Type.Optional(Type.Object({     // 设备身份认证
    id: NonEmptyString,
    publicKey: NonEmptyString,
    signature: NonEmptyString,
    signedAt: Type.Integer({ minimum: 0 }),
    nonce: Type.Optional(NonEmptyString),
  })),
  auth: Type.Optional(Type.Object({       // 认证凭据
    token: Type.Optional(Type.String()),
    password: Type.Optional(Type.String()),
  })),
  locale: Type.Optional(Type.String()),
  userAgent: Type.Optional(Type.String()),
});
```

### 3.3 认证流程详解

#### Token 认证

```typescript
// 配置 (在 ~/.openclaw/openclaw.json)
{
  "gateway": {
    "auth": {
      "mode": "token",
      "token": "your-secret-token-here"
    }
  }
}

// 连接请求
{
  "type": "req",
  "id": "uuid",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": { "id": "my-client", "version": "1.0", "platform": "web" },
    "auth": { "token": "your-secret-token-here" }
  }
}
```

#### Password 认证

```typescript
// 配置
{
  "gateway": {
    "auth": {
      "mode": "password",
      "password": "your-password-here"
    }
  }
}

// 连接请求 (auth 字段包含 password)
{
  "auth": { "password": "your-password-here" }
}
```

#### Device Token 认证

设备认证使用非对称签名，允许服务器颁发持久化令牌：

```typescript
// 1. 客户端生成设备密钥对 (首次)
const deviceIdentity = {
  deviceId: "unique-device-id",
  publicKeyPem: "...",
  privateKeyPem: "..."
};

// 2. 构建认证 payload
const payload = JSON.stringify({
  deviceId: deviceIdentity.deviceId,
  clientId: "gateway-client",
  clientMode: "backend",
  role: "operator",
  scopes: ["operator.admin"],
  signedAtMs: Date.now(),
  token: storedToken ?? null,
  nonce: serverNonce
});

// 3. 签名
const signature = signDevicePayload(privateKeyPem, payload);

// 4. 发送连接请求
{
  "device": {
    "id": deviceIdentity.deviceId,
    "publicKey": publicKeyPem,
    "signature": signature,
    "signedAt": Date.now(),
    "nonce": serverNonce
  }
}
```

### 3.4 HelloOk 响应

服务器认证成功后返回：

```typescript
export const HelloOkSchema = Type.Object({
  type: Type.Literal("hello-ok"),
  protocol: Type.Integer({ minimum: 1 }),
  server: Type.Object({
    version: NonEmptyString,
    commit: Type.Optional(NonEmptyString),
    host: Type.Optional(NonEmptyString),
    connId: NonEmptyString,
  }),
  features: Type.Object({
    methods: Type.Array(NonEmptyString),  // 可用的 RPC 方法
    events: Type.Array(NonEmptyString),   // 支持的事件
  }),
  snapshot: SnapshotSchema,
  canvasHostUrl: Type.Optional(NonEmptyString),
  auth: Type.Optional(Type.Object({      // 新颁发的设备令牌
    deviceToken: NonEmptyString,
    role: NonEmptyString,
    scopes: Type.Array(NonEmptyString),
    issuedAtMs: Type.Optional(Type.Integer({ minimum: 0 })),
  })),
  policy: Type.Object({
    maxPayload: Type.Integer({ minimum: 1 }),
    maxBufferedBytes: Type.Integer({ minimum: 1 }),
    tickIntervalMs: Type.Integer({ minimum: 1 }),
  }),
});
```

### 3.5 认证代码位置

| 文件 | 功能 |
|------|------|
| `src/gateway/auth.ts` | 所有认证逻辑实现 |
| `src/gateway/device-auth.ts` | 设备认证辅助 |
| `src/infra/device-identity.ts` | 设备密钥管理 |
| `src/infra/device-auth-store.ts` | 令牌持久化 |

---

## 4. 消息帧格式

### 4.1 RequestFrame (请求)

客户端发送请求到服务器：

```typescript
export const RequestFrameSchema = Type.Object({
  type: Type.Literal("req"),
  id: NonEmptyString,              // UUID，用于匹配响应
  method: NonEmptyString,          // RPC 方法名
  params: Type.Optional(Type.Unknown()),
});
```

**示例**:

```json
{
  "type": "req",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "method": "agent",
  "params": {
    "message": "Hello, world!",
    "sessionKey": "main",
    "idempotencyKey": "req-123"
  }
}
```

### 4.2 ResponseFrame (响应)

服务器返回响应：

```typescript
export const ResponseFrameSchema = Type.Object({
  type: Type.Literal("res"),
  id: NonEmptyString,              // 对应的请求 ID
  ok: Type.Boolean(),
  payload: Type.Optional(Type.Unknown()),
  error: Type.Optional(ErrorShapeSchema),
});
```

**错误格式**:

```typescript
export const ErrorShapeSchema = Type.Object({
  code: NonEmptyString,
  message: NonEmptyString,
  details: Type.Optional(Type.Unknown()),
  retryable: Type.Optional(Type.Boolean()),
  retryAfterMs: Type.Optional(Type.Integer({ minimum: 0 })),
});
```

### 4.3 EventFrame (事件)

服务器主动推送事件：

```typescript
export const EventFrameSchema = Type.Object({
  type: Type.Literal("event"),
  event: NonEmptyString,           // 事件类型名
  payload: Type.Optional(Type.Unknown()),
  seq: Type.Optional(Type.Integer({ minimum: 0 })),
  stateVersion: Type.Optional(StateVersionSchema),
});
```

### 4.4 GatewayFrame (联合类型)

使用判别式联合:

```typescript
export const GatewayFrameSchema = Type.Union(
  [RequestFrameSchema, ResponseFrameSchema, EventFrameSchema],
  { discriminator: "type" }
);
```

---

## 5. RPC 方法详解

### 5.1 方法列表概览

完整方法列表位于 `src/gateway/server-methods-list.ts`:

```typescript
export const GATEWAY_EVENTS = [
  "connect.challenge", "agent", "chat", "presence", "tick",
  "shutdown", "health", "heartbeat", "cron",
  "node.pair.requested", "node.pair.resolved",
  "device.pair.requested", "device.pair.resolved",
  "voicewake.changed", "exec.approval.requested",
  "exec.approval.resolved",
];
```

### 5.2 Agent 方法

#### agent - 发送消息给 agent

**文件**: `src/gateway/server-methods/agent.ts`

**请求参数**:

```typescript
export const AgentParamsSchema = Type.Object({
  message: NonEmptyString,
  agentId: Type.Optional(NonEmptyString),
  to: Type.Optional(Type.String()),         // 目标地址
  replyTo: Type.Optional(Type.String()),
  sessionId: Type.Optional(Type.String()),
  sessionKey: Type.Optional(Type.String()),
  thinking: Type.Optional(Type.String()),     // thinking level
  deliver: Type.Optional(Type.Boolean()),     // 是否发送到渠道
  attachments: Type.Optional(Type.Array(Type.Unknown())),
  channel: Type.Optional(Type.String()),
  replyChannel: Type.Optional(Type.String()),
  accountId: Type.Optional(Type.String()),
  replyAccountId: Type.Optional(Type.String()),
  threadId: Type.Optional(Type.String()),
  groupId: Type.Optional(Type.String()),
  groupChannel: Type.Optional(Type.String()),
  groupSpace: Type.Optional(Type.String()),
  timeout: Type.Optional(Type.Integer({ minimum: 0 })),
  lane: Type.Optional(Type.String()),
  extraSystemPrompt: Type.Optional(Type.String()),
  idempotencyKey: NonEmptyString,         // 必需
  label: Type.Optional(SessionLabelString),
  spawnedBy: Type.Optional(Type.String()),
});
```

**响应 (accepted)**:

```json
{
  "type": "res",
  "id": "...",
  "ok": true,
  "payload": {
    "runId": "unique-run-id",
    "status": "accepted",
    "acceptedAt": 1704067200000
  }
}
```

**响应 (completed)**:

```json
{
  "type": "res",
  "id": "...",
  "ok": true,
  "payload": {
    "runId": "unique-run-id",
    "status": "ok",
    "summary": "completed",
    "result": { ... }
  }
}
```

#### agent.identity.get

获取 agent 的身份信息：

```typescript
// 请求
{ "agentId": "my-agent" }

// 响应
{
  "agentId": "my-agent",
  "name": "My Agent",
  "avatar": "url",
  "emoji": "🤖"
}
```

#### agent.wait

等待 agent 运行完成：

```typescript
// 请求
{
  "runId": "run-id",
  "timeoutMs": 30000
}

// 响应
{
  "runId": "run-id",
  "status": "ok",  // ok | timeout | error
  "startedAt": 1704067200000,
  "endedAt": 1704067230000,
  "error": "..."
}
```

### 5.3 Chat 方法

#### chat.send

**文件**: `src/gateway/server-methods/chat.ts`

发送聊天消息 (WebChat 原生方法):

```typescript
export const ChatSendParamsSchema = Type.Object({
  sessionKey: NonEmptyString,
  message: Type.String(),
  thinking: Type.Optional(Type.String()),
  deliver: Type.Optional(Type.Boolean()),
  attachments: Type.Optional(Type.Array(Type.Unknown())),
  timeoutMs: Type.Optional(Type.Integer({ minimum: 0 })),
  idempotencyKey: NonEmptyString,
});
```

**响应**:

```json
{
  "runId": "client-run-id",
  "status": "started"
}
```

#### chat.history

获取聊天历史：

```typescript
// 请求
{ "sessionKey": "main", "limit": 200 }

// 响应
{
  "sessionKey": "main",
  "sessionId": "uuid",
  "messages": [ ... ],  // 最多 1000 条
  "thinkingLevel": "high",
  "verboseLevel": "full"
}
```

#### chat.abort

中止运行中的聊天：

```typescript
// 请求
{
  "sessionKey": "main",
  "runId": "run-id"  // 可选，不指定则中止所有
}

// 响应
{
  "ok": true,
  "aborted": true,
  "runIds": ["run-id"]
}
```

#### chat.inject

向会话注入消息（不触发 agent）：

```typescript
// 请求
{
  "sessionKey": "main",
  "message": "System notice",
  "label": "System"
}

// 响应
{
  "ok": true,
  "messageId": "msg-id"
}
```

### 5.4 Sessions 方法

**文件**: `src/gateway/server-methods/sessions.ts`

#### sessions.list

列出所有会话：

```typescript
export const SessionsListParamsSchema = Type.Object({
  limit: Type.Optional(Type.Integer({ minimum: 1 })),
  activeMinutes: Type.Optional(Type.Integer({ minimum: 1 })),
  includeGlobal: Type.Optional(Type.Boolean()),
  includeUnknown: Type.Optional(Type.Boolean()),
  includeDerivedTitles: Type.Optional(Type.Boolean()),
  includeLastMessage: Type.Optional(Type.Boolean()),
  label: Type.Optional(SessionLabelString),
  spawnedBy: Type.Optional(NonEmptyString),
  agentId: Type.Optional(NonEmptyString),
  search: Type.Optional(Type.String()),
});
```

**响应**:

```json
{
  "ts": 1704067200000,
  "sessions": [
    {
      "key": "main",
      "label": "Main Session",
      "agentId": "default",
      "updatedAt": 1704067200000,
      "title": "我的会话",
      "lastMessage": "最近消息..."
    }
  ]
}
```

#### sessions.patch

更新会话配置：

```typescript
export const SessionsPatchParamsSchema = Type.Object({
  key: NonEmptyString,
  label: Type.Optional(Type.Union([SessionLabelString, Type.Null()])),
  thinkingLevel: Type.Optional(Type.Union([NonEmptyString, Type.Null()])),
  verboseLevel: Type.Optional(Type.Union([NonEmptyString, Type.Null()])),
  reasoningLevel: Type.Optional(Type.Union([NonEmptyString, Type.Null()])),
  model: Type.Optional(Type.Union([NonEmptyString, Type.Null()])),
  sendPolicy: Type.Optional(Type.Union([
    Type.Literal("allow"),
    Type.Literal("deny"),
    Type.Null()
  ])),
  // ... 更多字段
});
```

#### sessions.delete

删除会话：

```typescript
// 请求
{
  "key": "session-key",
  "deleteTranscript": true
}

// 响应
{
  "ok": true,
  "key": "session-key",
  "deleted": true,
  "archived": ["/path/to/transcript.bak"]
}
```

### 5.5 节点方法

#### node.invoke

调用节点上的方法：

```typescript
// 请求
{
  "nodeId": "my-node",
  "method": "browser.navigate",
  "params": { "url": "https://example.com" }
}
```

### 5.6 完整方法列表

| 分类 | 方法 |
|------|------|
| **健康** | `health` |
| **日志** | `logs.tail` |
| **渠道** | `channels.status`, `channels.logout` |
| **配置** | `config.get/set/apply/patch/schema` |
| **向导** | `wizard.start/next/cancel/status` |
| **模型** | `models.list` |
| **Agents** | `agents.list/create/update/delete/files.*` |
| **技能** | `skills.status/bins/install/update` |
| **更新** | `update.run` |
| **语音** | `talk.mode`, `tts.*`, `voicewake.*` |
| **使用情况** | `usage.status`, `usage.cost` |
| **Cron** | `cron.list/status/add/update/remove/run/runs` |
| **执行批准** | `exec.approvals.*`, `exec.approval.*` |
| **节点配对** | `node.pair.*`, `node.rename` |
| **设备配对** | `device.pair.*`, `device.token.*` |
| **节点操作** | `node.list/describe/invoke` |
| **发送** | `send`, `poll`, `wake` |
| **Agent** | `agent`, `agent.identity.get`, `agent.wait` |
| **Chat** | `chat.history/send/abort/inject` |
| **会话** | `sessions.list/preview/resolve/patch/reset/delete/compact` |
| **浏览器** | `browser.request` |

---

## 6. 事件系统

### 6.1 事件列表

| 事件名 | 方向 | 描述 |
|--------|------|------|
| `connect.challenge` | S→C | 连接挑战，返回 nonce |
| `agent` | S→C | Agent 事件流 |
| `chat` | S→C | Chat 状态更新 |
| `presence` | S→C | 在线状态变更 |
| `tick` | S→C | 心跳事件 |
| `shutdown` | S→C | 服务器关闭通知 |
| `health` | S→C | 健康状态 |
| `heartbeat` | S→C | 节点心跳 |
| `cron` | S→C | 定时任务事件 |
| `node.pair.requested` | S→C | 节点配对请求 |
| `node.pair.resolved` | S→C | 节点配对结果 |
| `device.pair.requested` | S→C | 设备配对请求 |
| `device.pair.resolved` | S→C | 设备配对结果 |
| `voicewake.changed` | S→C | 语音唤醒状态变更 |
| `exec.approval.requested` | S→C | 执行批准请求 |
| `exec.approval.resolved` | S→C | 执行批准结果 |

### 6.2 Agent 事件

```typescript
export const AgentEventSchema = Type.Object({
  runId: NonEmptyString,
  seq: Type.Integer({ minimum: 0 }),
  stream: NonEmptyString,  // "delta" | "final" | "error" | ...
  ts: Type.Integer({ minimum: 0 }),
  data: Type.Record(Type.String(), Type.Unknown()),
});
```

### 6.3 Chat 事件

```typescript
export const ChatEventSchema = Type.Object({
  runId: NonEmptyString,
  sessionKey: NonEmptyString,
  seq: Type.Integer({ minimum: 0 }),
  state: Type.Union([
    Type.Literal("delta"),
    Type.Literal("final"),
    Type.Literal("aborted"),
    Type.Literal("error"),
  ]),
  message: Type.Optional(Type.Unknown()),
  errorMessage: Type.Optional(Type.String()),
  usage: Type.Optional(Type.Unknown()),
  stopReason: Type.Optional(Type.String()),
});
```

### 6.4 Tick 事件

```typescript
export const TickEventSchema = Type.Object({
  ts: Type.Integer({ minimum: 0 }),
});
```

### 6.5 Shutdown 事件

```typescript
export const ShutdownEventSchema = Type.Object({
  reason: NonEmptyString,
  restartExpectedMs: Type.Optional(Type.Integer({ minimum: 0 })),
});
```

---

## 7. 权限模型

### 7.1 角色

| 角色 | 描述 |
|------|------|
| `operator` | 操作员（UI、CLI 等）|
| `node` | 节点设备 |

### 7.2 权限范围

| 范围 | 描述 |
|------|------|
| `operator.admin` | 管理员权限，可修改配置 |
| `operator.read` | 只读权限 |
| `operator.write` | 读写权限 |
| `operator.approvals` | 执行批准权限 |
| `operator.pairing` | 配对权限 |

### 7.3 方法权限分类

**只读方法** (READ_SCOPE):

```typescript
const READ_METHODS = new Set([
  "health", "logs.tail", "channels.status", "status",
  "usage.status", "usage.cost", "tts.status",
  "tts.providers", "models.list", "agents.list",
  "agent.identity.get", "skills.status", "voicewake.get",
  "sessions.list", "sessions.preview", "cron.list",
  "cron.status", "cron.runs", "system-presence",
  "last-heartbeat", "node.list", "node.describe",
  "chat.history",
]);
```

**写入方法** (WRITE_SCOPE):

```typescript
const WRITE_METHODS = new Set([
  "send", "agent", "agent.wait", "wake", "talk.mode",
  "tts.enable", "tts.disable", "tts.convert",
  "tts.setProvider", "voicewake.set", "node.invoke",
  "chat.send", "chat.abort", "browser.request",
]);
```

**批准方法** (APPROVALS_SCOPE):

```typescript
const APPROVAL_METHODS = new Set([
  "exec.approval.request",
  "exec.approval.resolve",
]);
```

**配对方法** (PAIRING_SCOPE):

```typescript
const PAIRING_METHODS = new Set([
  "node.pair.request", "node.pair.list", "node.pair.approve",
  "node.pair.reject", "node.pair.verify",
  "device.pair.list", "device.pair.approve",
  "device.pair.reject", "device.token.rotate",
  "device.token.revoke", "node.rename",
]);
```

**节点角色专用方法**:

```typescript
const NODE_ROLE_METHODS = new Set([
  "node.invoke.result",
  "node.event",
  "skills.bins",
]);
```

### 7.4 权限验证流程

文件: `src/gateway/server-methods.ts`

```typescript
function authorizeGatewayMethod(method: string, client: GatewayRequestOptions["client"]) {
  const role = client?.connect?.role ?? "operator";
  const scopes = client?.connect?.scopes ?? [];

  // 节点角色检查
  if (NODE_ROLE_METHODS.has(method)) {
    if (role === "node") return null;
    return errorShape(ErrorCodes.INVALID_REQUEST, `unauthorized role: ${role}`);
  }

  // 管理员权限
  if (scopes.includes(ADMIN_SCOPE)) return null;

  // 检查特定权限范围
  if (APPROVAL_METHODS.has(method) && !scopes.includes(APPROVALS_SCOPE)) {
    return errorShape(ErrorCodes.INVALID_REQUEST, "missing scope: operator.approvals");
  }
  // ... 更多检查
}
```

---

## 8. 会话管理

### 8.1 会话模型

会话表示隔离的 agent 对话：

| 会话类型 | 密钥格式 | 描述 |
|----------|----------|------|
| 主会话 | `main` | 默认的直接对话 |
| Agent 会话 | `agent:{id}` | 特定 agent 的会话 |
| 渠道会话 | `{channel}+{id}` | 特定渠道的会话 |
| 标签会话 | `label:{name}` | 用户命名的会话 |

### 8.2 会话存储

**位置**: `~/.openclaw/sessions.db` (SQLite)

**结构**:

```typescript
type SessionEntry = {
  sessionId: string;              // UUID
  updatedAt: number;
  thinkingLevel?: string;
  verboseLevel?: string;
  reasoningLevel?: string;
  systemSent?: boolean;
  sendPolicy?: "allow" | "deny";
  model?: string;
  providerOverride?: string;
  label?: string;
  spawnedBy?: string;
  channel?: string;
  groupId?: string;
  groupChannel?: string;
  space?: string;
  deliveryContext?: Record<string, unknown>;
  skillsSnapshot?: unknown;
  lastChannel?: string;
  lastTo?: string;
  lastAccountId?: string;
  // ... 更多字段
};
```

### 8.3 会话密钥解析

```typescript
// src/routing/session-key.ts
export function parseAgentSessionKey(key: string): {
  agentId?: string;
  label?: string;
  channel?: string;
  accountId?: string;
} | null {
  // main -> agentId: undefined
  // agent:my-agent -> agentId: "my-agent"
  // label:My Chat -> label: "My Chat"
  // whatsapp+1234567890 -> channel: "whatsapp", accountId: "1234567890"
}
```

---

## 9. 客户端实现指南

### 9.1 使用 GatewayClient 类

OpenClaw 提供了现成的客户端类，位于 `src/gateway/client.ts`:

```typescript
import { GatewayClient } from "./gateway/client.js";

const client = new GatewayClient({
  url: "ws://127.0.0.1:18789",
  token: "your-token",
  clientName: "my-app",
  clientVersion: "1.0.0",
  platform: "web",
  role: "operator",
  scopes: ["operator.write"],
  onEvent: (evt) => {
    console.log("Event:", evt.event, evt.payload);
  },
  onHelloOk: (hello) => {
    console.log("Connected:", hello.server);
  },
  onClose: (code, reason) => {
    console.log("Closed:", code, reason);
  },
});

client.start();

// 发送请求
const result = await client.request("sessions.list", { limit: 10 });
console.log(result);
```

### 9.2 原生 WebSocket 客户端

```typescript
import WebSocket from "ws";

const ws = new WebSocket("ws://127.0.0.1:18789");

const pending = new Map();

ws.on("open", () => {
  // 发送连接请求
  const connect = {
    type: "req",
    id: generateUUID(),
    method: "connect",
    params: {
      minProtocol: 3,
      maxProtocol: 3,
      client: {
        id: "my-client",
        version: "1.0.0",
        platform: "node",
        mode: "backend",
      },
      auth: { token: "your-token" },
      role: "operator",
      scopes: ["operator.write"],
    },
  };
  ws.send(JSON.stringify(connect));
});

ws.on("message", (data) => {
  const msg = JSON.parse(data.toString());

  if (msg.type === "res") {
    // 响应
    const pending = pending.get(msg.id);
    if (pending) {
      pending.resolve(msg);
      pending.delete(msg.id);
    }
  } else if (msg.type === "event") {
    // 事件
    handleEvent(msg);
  }
});

// 发送 agent 请求
async function sendAgentRequest(message: string) {
  const id = generateUUID();
  const req = {
    type: "req",
    id,
    method: "agent",
    params: {
      message,
      sessionKey: "main",
      idempotencyKey: id,
    },
  };

  const p = new Promise((resolve) => pending.set(id, { resolve }));
  ws.send(JSON.stringify(req));
  return await p;
}
```

### 9.3 处理流式响应

对于需要等待最终响应的方法：

```typescript
// 使用 expectFinal 选项
const response = await client.request(
  "agent",
  { message: "Hello", idempotencyKey: "id-1" },
  { expectFinal: true }
);
```

### 9.4 心跳处理

```typescript
let lastTick = Date.now();

client.onEvent = (evt) => {
  if (evt.event === "tick") {
    lastTick = Date.now();
  }
};

// 检测超时
setInterval(() => {
  const gap = Date.now() - lastTick;
  if (gap > 60000) {  // 2x tickInterval
    console.warn("Tick timeout, reconnecting...");
    client.stop();
    client.start();
  }
}, 30000);
```

### 9.5 设备认证客户端

```typescript
import {
  loadOrCreateDeviceIdentity,
  signDevicePayload,
} from "./infra/device-identity.js";

const deviceIdentity = loadOrCreateDeviceIdentity();

const client = new GatewayClient({
  // ... 其他选项
  deviceIdentity,
  role: "operator",
  scopes: ["operator.admin"],
});

// 令牌会自动轮换和管理
```

---

## 10. 附录

### 10.1 错误码

文件: `src/gateway/protocol/schema/error-codes.ts`

```typescript
export const ErrorCodes = {
  NOT_LINKED: "NOT_LINKED",
  NOT_PAIRED: "NOT_PAIRED",
  AGENT_TIMEOUT: "AGENT_TIMEOUT",
  INVALID_REQUEST: "INVALID_REQUEST",
  UNAVAILABLE: "UNAVAILABLE",
} as const;
```

### 10.2 客户端模式

```typescript
export const GATEWAY_CLIENT_MODES = {
  BACKEND: "backend",    // 后台服务
  UI: "ui",             // Web UI
  PROBE: "probe",        // 探测器
  NODE: "node",          // 节点设备
};
```

### 10.3 客户端能力

```typescript
export const GATEWAY_CLIENT_CAPS = {
  TOOL_EVENTS: "tool-events",  // 接收工具执行事件
};
```

### 10.4 WebSocket 关闭代码

| 代码 | 描述 |
|------|------|
| 1000 | 正常关闭 |
| 1006 | 异常关闭（无 close 帧）|
| 1008 | 策略违规 |
| 1012 | 服务重启 |
| 4000 | 心跳超时 |

### 10.5 配置参考

```json
{
  "gateway": {
    "port": 18789,
    "bind": "127.0.0.1",
    "auth": {
      "mode": "token",
      "token": "your-secret-token",
      "allowTailscale": false
    },
    "controlUi": {
      "enabled": true,
      "basePath": "/"
    }
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-6",
      "workspace": "~/.openclaw/workspace"
    }
  }
}
```

### 10.6 相关文件清单

| 文件 | 行数 | 功能 |
|------|------|------|
| `src/gateway/protocol/schema/frames.ts` | 165 | 核心帧定义 |
| `src/gateway/protocol/schema/agent.ts` | 107 | Agent 参数和事件 |
| `src/gateway/protocol/schema/sessions.ts` | 120 | 会话参数 |
| `src/gateway/protocol/schema/logs-chat.ts` | 82 | Chat 参数和事件 |
| `src/gateway/server-methods.ts` | 220 | 方法路由和权限 |
| `src/gateway/server-methods/agent.ts` | 524 | agent 方法实现 |
| `src/gateway/server-methods/chat.ts` | 703 | chat 方法实现 |
| `src/gateway/server-methods/sessions.ts` | 483 | sessions 方法实现 |
| `src/gateway/auth.ts` | 278 | 认证逻辑 |
| `src/gateway/client.ts` | 442 | 客户端实现 |

---

## 总结

OpenClaw Gateway 是一个设计精良的 WebSocket 网关系统：

1. **协议清晰**: 基于 JSON 的简单消息格式
2. **类型安全**: 使用 TypeBox 定义完整的 Schema
3. **认证灵活**: 支持多种认证方式，包括设备令牌轮换
4. **权限细粒度**: 基于角色和范围的权限模型
5. **可扩展**: RPC 方法可通过插件系统扩展
6. **生产就绪**: 完善的错误处理、心跳、重连机制

---

*报告结束*
