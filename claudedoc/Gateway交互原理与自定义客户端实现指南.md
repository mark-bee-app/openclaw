# OpenClaw Gateway 交互原理与自定义客户端实现指南

> 本文档深入分析 OpenClaw 主网关服务与 Telegram 等 app 之间的消息交互实现原理，并指导如何创建自定义 app 或 web 页面与 Gateway 交互。

## 目录

1. [系统架构概述](#1-系统架构概述)
2. [Gateway WebSocket 协议](#2-gateway-websocket-协议)
3. [渠道插件系统](#3-渠道插件系统)
4. [ACP 协议转换层](#4-acp-协议转换层)
5. [会话管理与消息路由](#5-会话管理与消息路由)
6. [现有客户端实现分析](#6-现有客户端实现分析)
7. [自定义客户端实现指南](#7-自定义客户端实现指南)
8. [实现示例](#8-实现示例)

---

## 1. 系统架构概述

OpenClaw 采用**中心辐射型架构**，Gateway 作为中央控制平面连接所有组件：

```
┌─────────────────────────────────────────────────────────────────┐
│                        渠道层 (Channels)                        │
│  Telegram │ WhatsApp │ Slack │ Discord │ Signal │ iMessage ...  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                    消息接收/发送
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                   Gateway (WebSocket 18789)                    │
│                  ┌─────────────────────────┐                   │
│                  │   控制平面 (Control)     │                   │
│                  │   - 会话管理             │                   │
│                  │   - 消息路由             │                   │
│                  │   - 事件广播             │                   │
│                  │   - 认证授权             │                   │
│                  └─────────────────────────┘                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Web/Control  │    │  Pi Agent    │    │ 移动节点     │
│    UI        │    │   (ACP)      │    │  (Node)      │
│  客户端       │    │  工具流式     │    │  远程执行    │
└──────────────┘    └──────────────┘    └──────────────┘
```

### 核心组件

| 组件 | 说明 | 关键文件 |
|------|------|----------|
| **Gateway Server** | WebSocket 服务器，端口 18789 | `src/gateway/server.impl.ts` |
| **协议层** | 定义消息格式和 RPC 方法 | `src/gateway/protocol/` |
| **渠道插件** | 各平台消息适配器 | `src/channels/plugins/` |
| **ACP 层** | Agent Client Protocol 转换 | `src/acp/translator.ts` |
| **会话管理** | 会话状态持久化 | `src/config/sessions.js` |

---

## 2. Gateway WebSocket 协议

### 2.1 协议版本与帧格式

OpenClaw Gateway 使用 WebSocket 协议（当前版本 `v3`），支持三种帧类型：

#### Request Frame (请求帧)
客户端向 Gateway 发送请求：
```json
{
  "type": "req",
  "id": "unique-request-id",
  "method": "chat.send",
  "params": {
    "sessionKey": "main",
    "message": "Hello",
    "idempotencyKey": "request-uuid"
  }
}
```

#### Response Frame (响应帧)
Gateway 返回请求结果：
```json
{
  "type": "res",
  "id": "unique-request-id",
  "ok": true,
  "payload": {
    "runId": "run-uuid",
    "status": "started"
  },
  "error": null
}
```

#### Event Frame (事件帧)
Gateway 主动推送事件：
```json
{
  "type": "event",
  "event": "chat",
  "payload": {
    "runId": "run-uuid",
    "sessionKey": "main",
    "seq": 1,
    "state": "delta",
    "message": {
      "role": "assistant",
      "content": [{"type": "text", "text": "Hello!"}]
    }
  },
  "seq": 1
}
```

### 2.2 连接握手流程

```
客户端                              Gateway
  │                                   │
  │◄────────── connect.challenge ──────│  服务器发送 nonce
  │                                   │
  │────────── connect (auth) ─────────►│  客户端发送认证信息
  │                                   │
  │◄────────── hello-ok ───────────────││  握手成功，返回服务器信息
  │                                   │
  │    ═══════ 已连接 ═══════          │
```

#### Connect 请求示例
```json
{
  "type": "req",
  "id": "connect-1",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "my-custom-client",
      "displayName": "My App",
      "version": "1.0.0",
      "platform": "web",
      "mode": "operator"
    },
    "auth": {
      "token": "your-auth-token"
    }
  }
}
```

### 2.3 核心 RPC 方法

| 方法 | 说明 | 参数 |
|------|------|------|
| `chat.send` | 发送消息给 Agent | `sessionKey`, `message`, `idempotencyKey` |
| `chat.history` | 获取会话历史 | `sessionKey`, `limit` |
| `chat.abort` | 中止运行中的对话 | `sessionKey`, `runId` |
| `sessions.list` | 列出所有会话 | `limit` |
| `agent.chat` | 发送消息到指定渠道 | `to`, `message`, `channel` |
| `browser.navigate` | 控制浏览器导航 | `url`, `sessionId` |

### 2.4 事件类型

| 事件 | 说明 | 触发时机 |
|------|------|----------|
| `chat` | 聊天消息更新 | Agent 响应流式更新 |
| `agent` | Agent 事件 | 工具调用、状态变化 |
| `presence` | 在线状态 | 客户端连接/断开 |
| `heartbeat` | 心跳 | 定期发送 |
| `node.pair.requested` | 节点配对请求 | 新设备请求连接 |

---

## 3. 渠道插件系统

### 3.1 插件架构

渠道插件是独立的模块，实现了标准化的适配器接口：

```
ChannelPlugin
├── meta (插件元数据)
├── capabilities (功能能力)
├── adapters (适配器)
│   ├── ChannelMessagingAdapter  (消息收发)
│   ├── ChannelGatewayAdapter    (Gateway 集成)
│   ├── ChannelOutboundAdapter   (消息发送)
│   ├── ChannelAuthAdapter       (认证)
│   └── ChannelConfigAdapter     (配置)
└── gatewayMethods (RPC 方法扩展)
```

### 3.2 消息流向

```
Telegram 用户发送消息
        │
        ▼
Telegram Bot API ←→ Telegram Channel Plugin
        │
        ▼
    消息标准化 (MsgContext)
        │
        ▼
    会话路由 (sessionKey 解析)
        │
        ▼
    dispatchInboundMessage
        │
        ▼
    Pi Agent (ACP 层)
        │
        ▼
    Agent 处理响应
        │
        ▼
    ChannelOutboundAdapter
        │
        ▼
Telegram Bot API → Telegram 用户
```

### 3.3 Telegram 渠道配置

```json
{
  "channels": {
    "telegram": {
      "accounts": {
        "default": {
          "token": "your-bot-token",
          "enabled": true,
          "allowFrom": ["*"]
        }
      }
    }
  }
}
```

### 3.4 消息上下文 (MsgContext)

所有渠道消息都转换为统一的 `MsgContext` 格式：

```typescript
interface MsgContext {
  // 原始消息
  Body: string;
  RawBody: string;

  // 处理后的消息
  BodyForAgent: string;    // 注入时间戳等
  BodyForCommands: string; // 命令解析

  // 会话信息
  SessionKey: string;
  ChatType: "direct" | "group" | "channel";

  // 发送者信息
  SenderId: string;
  SenderName: string;

  // 渠道信息
  Provider: string;           // "telegram"
  OriginatingChannel: string;

  // 权限
  CommandAuthorized: boolean;
  GatewayClientScopes?: string[];

  // 线程/回复
  ReplyToId?: string;
  MessageThreadId?: string;
}
```

---

## 4. ACP 协议转换层

### 4.1 架构

ACP (Agent Client Protocol) 是 Gateway 与 Pi Agent 之间的桥梁：

```
Gateway WebSocket                ACP Translator                Pi Agent
      │                              │                            │
      │ chat.send                    │                            │
      ├─────────────────────────────►│                            │
      │                              │ initialize()              │
      │                              ├──────────────────────────►│
      │                              │◄──────────────────────────┤
      │                              │ newSession()              │
      │                              ├──────────────────────────►│
      │                              │◄──────────────────────────┤
      │                              │ prompt()                  │
      │                              ├──────────────────────────►│
      │                              │◄────── sessionUpdate ─────┤
      │◄──── chat (delta) ───────────┤                           │
      │                              │                           │
      │                              │   (工具调用循环)           │
      │                              │                           │
```

### 4.2 协议转换

**Gateway → Pi Agent:**
```typescript
// Gateway chat.send 转换为 ACP prompt
Gateway: chat.send { sessionKey, message }
    ↓
ACP: prompt({
  sessionId,
  prompt: [{ type: "text", text: message }]
})
```

**Pi Agent → Gateway:**
```typescript
// ACP sessionUpdate 转换为 Gateway chat event
ACP: sessionUpdate({ sessionUpdate: "agent_message_chunk", content })
    ↓
Gateway: chat event { state: "delta", message }
```

---

## 5. 会话管理与消息路由

### 5.1 会话模型

| 会话类型 | SessionKey 格式 | 说明 |
|----------|----------------|------|
| Main | `main` | 主要直接会话 |
| Telegram 私聊 | `telegram+USER_ID` | 与特定用户的私聊 |
| Telegram 群组 | `telegram+GROUP_ID` | 特定群组会话 |
| WhatsApp | `whatsapp+PHONE_NUMBER` | WhatsApp 会话 |
| 自定义 | 自定义格式 | 插件定义的格式 |

### 5.2 会话持久化

```
~/.openclaw/
├── sessions.db           # SQLite 会话数据库
├── workspace/
│   └── transcripts/      # 会话记录 (jsonl)
│       ├── main.jsonl
│       ├── telegram+123456789.jsonl
│       └── ...
```

### 5.3 消息路由

```typescript
// 路由逻辑 (简化版)
function routeMessage(ctx: MsgContext): string {
  // 1. 确定渠道
  const channel = ctx.Provider;

  // 2. 确定聊天类型
  if (ctx.ChatType === "direct") {
    return `${channel}+${ctx.SenderId}`;
  }

  // 3. 群组聊天
  if (ctx.ChatType === "group") {
    return `${channel}+${ctx.GroupId}`;
  }

  // 4. 默认 main
  return "main";
}
```

---

## 6. 现有客户端实现分析

### 6.1 Control UI (Web 客户端)

**位置:** `dist/control-ui/` (打包后的静态文件)

**技术栈:**
- Lit (Web Components)
- 原生 WebSocket API
- TypeScript

**关键实现:**

```typescript
// 简化的 Control UI 连接代码
class GatewayClient {
  private ws: WebSocket;

  async connect(url: string, token: string) {
    this.ws = new WebSocket(url);

    this.ws.onmessage = (event) => {
      const frame = JSON.parse(event.data);
      if (frame.type === "event") {
        this.handleEvent(frame);
      }
    };

    // 发送连接请求
    this.send({
      type: "req",
      id: nanoid(),
      method: "connect",
      params: {
        minProtocol: 3,
        maxProtocol: 3,
        client: {
          id: "control-ui",
          displayName: "Control UI",
          version: "1.0.0",
          platform: "web",
          mode: "operator"
        },
        auth: { token }
      }
    });
  }

  sendMessage(sessionKey: string, message: string) {
    this.send({
      type: "req",
      id: nanoid(),
      method: "chat.send",
      params: {
        sessionKey,
        message,
        idempotencyKey: nanoid()
      }
    });
  }

  private send(frame: unknown) {
    this.ws.send(JSON.stringify(frame));
  }
}
```

### 6.2 移动节点客户端

**特点:**
- 角色: `node` (而非 `operator`)
- 支持远程命令执行
- 设备配对机制

**连接差异:**
```json
{
  "client": {
    "mode": "node",
    "id": "ios-node",
    "platform": "ios"
  },
  "role": "node",
  "commands": ["bash", "browser", ...],
  "device": {
    "id": "device-uuid",
    "publicKey": "...",
    "signature": "...",
    "signedAt": 1234567890,
    "nonce": "server-nonce"
  }
}
```

---

## 7. 自定义客户端实现指南

### 7.1 实现方式对比

| 方式 | 复杂度 | 功能 | 适用场景 |
|------|--------|------|----------|
| **WebSocket 直接连接** | 中等 | 完整功能 | Web 应用、桌面应用 |
| **HTTP API** | 低 | 受限功能 | 简单集成 |
| **渠道插件** | 高 | 双向消息 | 新聊天平台集成 |

### 7.2 方案一：WebSocket 客户端

#### 步骤 1: 获取认证信息

```bash
# 查看当前 Gateway 配置
cat ~/.openclaw/openclaw.json | grep -A5 gateway

# 输出示例:
# "gateway": {
#   "port": 18789,
#   "auth": {
#     "token": "your-token-here"
#   }
# }
```

#### 步骤 2: 实现握手

```typescript
interface GatewayConfig {
  url: string;
  token?: string;
  clientInfo: {
    id: string;
    displayName: string;
    version: string;
    platform: string;
  };
}

class OpenClawClient {
  private ws: WebSocket | null = null;
  private config: GatewayConfig;
  private messageId = 0;
  private pendingRequests = new Map<string, {
    resolve: (value: any) => void;
    reject: (error: any) => void;
  }>();

  constructor(config: GatewayConfig) {
    this.config = config;
  }

  async connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(this.config.url);

      this.ws.onopen = async () => {
        try {
          await this.handshake();
          resolve();
        } catch (err) {
          reject(err);
        }
      };

      this.ws.onmessage = (event) => {
        this.handleMessage(JSON.parse(event.data));
      };

      this.ws.onerror = (error) => {
        reject(error);
      };
    });
  }

  private async handshake(): Promise<void> {
    const connectNonce = await this.waitForChallenge();

    const response = await this.request("connect", {
      minProtocol: 3,
      maxProtocol: 3,
      client: this.config.clientInfo,
      auth: this.config.token ? { token: this.config.token } : undefined
    });

    if (!response.ok) {
      throw new Error(`连接失败: ${response.error?.message}`);
    }

    console.log("已连接到 Gateway:", response.payload.server);
  }

  private waitForChallenge(): Promise<string> {
    return new Promise((resolve) => {
      const handler = (event: MessageEvent) => {
        const frame = JSON.parse(event.data);
        if (frame.event === "connect.challenge") {
          this.ws?.removeEventListener("message", handler);
          resolve(frame.payload.nonce);
        }
      };
      this.ws?.addEventListener("message", handler);
    });
  }

  private async request(method: string, params?: any): Promise<any> {
    const id = `req-${++this.messageId}`;

    return new Promise((resolve, reject) => {
      this.pendingRequests.set(id, { resolve, reject });

      this.ws?.send(JSON.stringify({
        type: "req",
        id,
        method,
        params
      }));

      // 设置超时
      setTimeout(() => {
        if (this.pendingRequests.has(id)) {
          this.pendingRequests.delete(id);
          reject(new Error("请求超时"));
        }
      }, 30000);
    });
  }

  private handleMessage(frame: any): void {
    if (frame.type === "res") {
      const pending = this.pendingRequests.get(frame.id);
      if (pending) {
        this.pendingRequests.delete(frame.id);
        if (frame.ok) {
          pending.resolve(frame.payload);
        } else {
          pending.reject(frame.error);
        }
      }
    } else if (frame.type === "event") {
      this.handleEvent(frame);
    }
  }

  private handleEvent(frame: any): void {
    console.log("收到事件:", frame.event, frame.payload);

    switch (frame.event) {
      case "chat":
        this.onChatEvent(frame.payload);
        break;
      case "agent":
        this.onAgentEvent(frame.payload);
        break;
      case "presence":
        this.onPresenceEvent(frame.payload);
        break;
    }
  }

  // ========== 公共 API ==========

  async sendMessage(sessionKey: string, message: string): Promise<string> {
    const result = await this.request("chat.send", {
      sessionKey,
      message,
      idempotencyKey: `msg-${Date.now()}`
    });
    return result.runId;
  }

  async getHistory(sessionKey: string, limit = 50): Promise<any[]> {
    const result = await this.request("chat.history", {
      sessionKey,
      limit
    });
    return result.messages;
  }

  async getSessions(limit = 100): Promise<any[]> {
    const result = await this.request("sessions.list", { limit });
    return result.sessions;
  }

  async abortChat(sessionKey: string, runId?: string): Promise<void> {
    await this.request("chat.abort", { sessionKey, runId });
  }

  // ========== 事件回调 ==========

  private onChatEvent(payload: any): void {
    console.log("聊天事件:", payload.state, payload.message);
    // 可以触发自定义回调
  }

  private onAgentEvent(payload: any): void {
    console.log("Agent 事件:", payload.stream, payload.data);
  }

  private onPresenceEvent(payload: any): void {
    console.log("在线状态:", payload.presence);
  }

  on(event: string, callback: (data: any) => void): void {
    // 实现事件监听器注册
  }

  disconnect(): void {
    this.ws?.close();
  }
}

// ========== 使用示例 ==========

async function main() {
  const client = new OpenClawClient({
    url: "ws://localhost:18789",
    token: "your-auth-token",  // 如果 Gateway 启用了认证
    clientInfo: {
      id: "my-custom-client",
      displayName: "My Custom App",
      version: "1.0.0",
      platform: "web"
    }
  });

  try {
    await client.connect();
    console.log("连接成功!");

    // 发送消息
    const runId = await client.sendMessage("main", "你好，介绍一下自己");
    console.log("消息已发送，runId:", runId);

    // 获取历史记录
    const history = await client.getHistory("main");
    console.log("历史消息数:", history.length);

    // 列出会话
    const sessions = await client.getSessions();
    console.log("会话列表:", sessions.map(s => s.key));

  } catch (err) {
    console.error("错误:", err);
  } finally {
    // 保持连接，接收流式响应
    // client.disconnect();
  }
}

// 在浏览器中
// main();
```

### 7.3 方案二：React Hook 实现

```typescript
import { useEffect, useState, useRef } from 'react';

interface ChatMessage {
  role: 'user' | 'assistant';
  content: string;
  timestamp?: number;
}

interface ConnectionState {
  connected: boolean;
  connecting: boolean;
  error?: string;
}

export function useOpenClawGateway(config: {
  url: string;
  token?: string;
}) {
  const [state, setState] = useState<ConnectionState>({
    connected: false,
    connecting: false
  });
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    setState(prev => ({ ...prev, connecting: true }));

    const ws = new WebSocket(config.url);
    wsRef.current = ws;

    let connectNonce: string;

    ws.onmessage = (event) => {
      const frame = JSON.parse(event.data);

      if (frame.type === 'event' && frame.event === 'connect.challenge') {
        connectNonce = frame.payload.nonce;

        // 发送连接请求
        ws.send(JSON.stringify({
          type: 'req',
          id: 'connect-1',
          method: 'connect',
          params: {
            minProtocol: 3,
            maxProtocol: 3,
            client: {
              id: 'react-client',
              displayName: 'React App',
              version: '1.0.0',
              platform: 'web',
              mode: 'operator'
            },
            auth: config.token ? { token: config.token } : undefined
          }
        }));
      }

      if (frame.type === 'res' && frame.id === 'connect-1') {
        if (frame.ok) {
          setState({ connected: true, connecting: false });
        } else {
          setState({
            connected: false,
            connecting: false,
            error: frame.error?.message
          });
        }
      }

      if (frame.type === 'event' && frame.event === 'chat') {
        const payload = frame.payload;
        if (payload.state === 'delta' || payload.state === 'final') {
          const content = payload.message?.content?.[0]?.text || '';

          if (payload.state === 'final') {
            setMessages(prev => [...prev, {
              role: 'assistant',
              content,
              timestamp: Date.now()
            }]);
          }
        }
      }
    };

    ws.onerror = () => {
      setState({
        connected: false,
        connecting: false,
        error: '连接错误'
      });
    };

    ws.onclose = () => {
      setState({
        connected: false,
        connecting: false
      });
    };

    return () => {
      ws.close();
    };
  }, [config.url, config.token]);

  const sendMessage = async (sessionKey: string, text: string) => {
    if (!wsRef.current || state.connected === false) return;

    const requestId = `msg-${Date.now()}`;

    wsRef.current.send(JSON.stringify({
      type: 'req',
      id: requestId,
      method: 'chat.send',
      params: {
        sessionKey,
        message: text,
        idempotencyKey: requestId
      }
    }));

    setMessages(prev => [...prev, {
      role: 'user',
      content: text,
      timestamp: Date.now()
    }]);
  };

  return {
    state,
    messages,
    sendMessage
  };
}

// ========== 使用示例 ==========

function ChatApp() {
  const { state, messages, sendMessage } = useOpenClawGateway({
    url: 'ws://localhost:18789',
    token: 'your-token'
  });

  const [input, setInput] = useState('');

  if (state.connecting) {
    return <div>连接中...</div>;
  }

  if (state.error) {
    return <div>错误: {state.error}</div>;
  }

  return (
    <div className="chat-app">
      <div className="messages">
        {messages.map((msg, i) => (
          <div key={i} className={`message ${msg.role}`}>
            {msg.content}
          </div>
        ))}
      </div>

      <form onSubmit={(e) => {
        e.preventDefault();
        sendMessage('main', input);
        setInput('');
      }}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="输入消息..."
        />
        <button type="submit" disabled={!state.connected}>
          发送
        </button>
      </form>
    </div>
  );
}
```

### 7.4 方案三：Python 客户端

```python
import asyncio
import json
import uuid
from typing import Any, Dict, Optional
import websockets
from websockets.client import WebSocketClientProtocol

class OpenClawGateway:
    def __init__(self, url: str = "ws://localhost:18789", token: Optional[str] = None):
        self.url = url
        self.token = token
        self.ws: Optional[WebSocketClientProtocol] = None
        self.message_id = 0
        self.pending: Dict[str, asyncio.Future] = {}
        self.event_handlers = {}

    async def connect(self) -> None:
        """连接到 Gateway"""
        self.ws = await websockets.connect(self.url)

        # 启动消息接收任务
        asyncio.create_task(self._receive_messages())

        # 等待 challenge
        nonce = await self._wait_for_challenge()

        # 发送连接请求
        response = await self._request("connect", {
            "minProtocol": 3,
            "maxProtocol": 3,
            "client": {
                "id": "python-client",
                "displayName": "Python Client",
                "version": "1.0.0",
                "platform": "python",
                "mode": "operator"
            },
            "auth": {"token": self.token} if self.token else None
        })

        if not response.get("ok"):
            raise Exception(f"连接失败: {response.get('error', {}).get('message')}")

        print(f"已连接: {response['payload']['server']['host']}")

    async def _wait_for_challenge(self) -> str:
        """等待连接 challenge"""
        future = asyncio.Future()

        async def handler():
            async for message in self.ws:
                frame = json.loads(message)
                if frame.get("event") == "connect.challenge":
                    future.set_result(frame["payload"]["nonce"])
                    return

        asyncio.create_task(handler())
        return await future

    async def _receive_messages(self):
        """接收并处理消息"""
        async for message in self.ws:
            frame = json.loads(message)

            if frame["type"] == "res":
                # 处理响应
                request_id = frame["id"]
                if request_id in self.pending:
                    future = self.pending.pop(request_id)
                    future.set_result(frame)

            elif frame["type"] == "event":
                # 处理事件
                event_name = frame["event"]
                if event_name in self.event_handlers:
                    await self.event_handlers[event_name](frame["payload"])

    async def _request(self, method: str, params: Optional[Dict] = None) -> Any:
        """发送请求并等待响应"""
        if not self.ws:
            raise Exception("未连接")

        request_id = f"req-{self.message_id}"
        self.message_id += 1

        future = asyncio.Future()
        self.pending[request_id] = future

        await self.ws.send(json.dumps({
            "type": "req",
            "id": request_id,
            "method": method,
            "params": params
        }))

        response = await asyncio.wait_for(future, timeout=30.0)
        return response

    def on(self, event: str, handler):
        """注册事件处理器"""
        self.event_handlers[event] = handler

    # ========== 公共 API ==========

    async def send_message(self, session_key: str, message: str) -> str:
        """发送消息"""
        response = await self._request("chat.send", {
            "sessionKey": session_key,
            "message": message,
            "idempotencyKey": str(uuid.uuid4())
        })

        if response["ok"]:
            return response["payload"]["runId"]
        else:
            raise Exception(response["error"]["message"])

    async def get_history(self, session_key: str, limit: int = 50) -> list:
        """获取历史记录"""
        response = await self._request("chat.history", {
            "sessionKey": session_key,
            "limit": limit
        })
        return response["payload"]["messages"] if response["ok"] else []

    async def get_sessions(self, limit: int = 100) -> list:
        """获取会话列表"""
        response = await self._request("sessions.list", { "limit": limit })
        return response["payload"]["sessions"] if response["ok"] else []

    async def abort_chat(self, session_key: str, run_id: Optional[str] = None) -> bool:
        """中止对话"""
        response = await self._request("chat.abort", {
            "sessionKey": session_key,
            "runId": run_id
        })
        return response.get("payload", {}).get("aborted", False)


# ========== 使用示例 ==========

async def main():
    client = OpenClawGateway(
        url="ws://localhost:18789",
        token="your-auth-token"
    )

    # 注册事件处理器
    async def on_chat(payload):
        state = payload.get("state")
        if state == "delta":
            print(f"[流式] {payload.get('message', {}).get('content', [{}])[0].get('text', '')}", end="")
        elif state == "final":
            print("\n[完成]")

    client.on("chat", on_chat)

    await client.connect()

    # 发送消息
    run_id = await client.send_message("main", "你好！")
    print(f"消息已发送: {run_id}")

    # 获取历史
    history = await client.get_history("main")
    print(f"历史消息数: {len(history)}")

    # 保持连接
    await asyncio.Event().wait()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 8. 实现示例

### 8.1 完整的 Web 应用示例

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>OpenClaw Chat</title>
  <style>
    body { font-family: system-ui; max-width: 800px; margin: 0 auto; padding: 20px; }
    #messages { border: 1px solid #ddd; height: 400px; overflow-y: auto; padding: 10px; margin-bottom: 10px; }
    .message { margin: 10px 0; padding: 10px; border-radius: 8px; }
    .message.user { background: #e3f2fd; text-align: right; }
    .message.assistant { background: #f5f5f5; }
    .message.system { background: #fff3cd; font-style: italic; }
    #status { padding: 5px 10px; border-radius: 4px; margin-bottom: 10px; }
    .connected { background: #c8e6c9; }
    .disconnected { background: #ffcdd2; }
    .connecting { background: #fff9c4; }
    #input-area { display: flex; gap: 10px; }
    #input { flex: 1; padding: 10px; border: 1px solid #ddd; border-radius: 4px; }
    button { padding: 10px 20px; background: #1976d2; color: white; border: none; border-radius: 4px; cursor: pointer; }
    button:disabled { background: #ccc; }
  </style>
</head>
<body>
  <h1>OpenClaw Gateway Chat</h1>

  <div id="status" class="connecting">连接中...</div>

  <div id="messages"></div>

  <div id="input-area">
    <input type="text" id="input" placeholder="输入消息..." disabled>
    <button id="send" disabled>发送</button>
  </div>

  <div style="margin-top: 20px;">
    <h3>设置</h3>
    <label>
      Gateway URL:
      <input type="text" id="gateway-url" value="ws://localhost:18789" style="width: 300px;">
    </label>
    <label style="margin-left: 20px;">
      Token (可选):
      <input type="password" id="auth-token" placeholder="如果需要的话">
    </label>
    <button onclick="reconnect()">重新连接</button>
  </div>

  <script>
    class OpenClawChat {
      constructor() {
        this.ws = null;
        this.messageId = 0;
        this.pendingRequests = new Map();
        this.currentMessage = '';
        this.sessionKey = 'main';

        this.ui = {
          status: document.getElementById('status'),
          messages: document.getElementById('messages'),
          input: document.getElementById('input'),
          send: document.getElementById('send'),
          url: document.getElementById('gateway-url'),
          token: document.getElementById('auth-token')
        };

        this.setupEventListeners();
      }

      setupEventListeners() {
        this.ui.send.addEventListener('click', () => this.sendMessage());
        this.ui.input.addEventListener('keypress', (e) => {
          if (e.key === 'Enter') this.sendMessage();
        });
      }

      async connect() {
        const url = this.ui.url.value;
        const token = this.ui.token.value;

        this.setStatus('connecting', '连接中...');
        this.ui.input.disabled = true;
        this.ui.send.disabled = true;

        try {
          this.ws = new WebSocket(url);

          this.ws.onopen = async () => {
            await this.handshake(token);
          };

          this.ws.onmessage = (event) => {
            this.handleMessage(JSON.parse(event.data));
          };

          this.ws.onerror = () => {
            this.setStatus('disconnected', '连接错误');
          };

          this.ws.onclose = () => {
            this.setStatus('disconnected', '已断开');
            this.ui.input.disabled = true;
            this.ui.send.disabled = true;
          };

        } catch (err) {
          this.setStatus('disconnected', `连接失败: ${err.message}`);
        }
      }

      async handshake(token) {
        // 等待 challenge
        const nonce = await new Promise((resolve) => {
          const handler = (event) => {
            const frame = JSON.parse(event.data);
            if (frame.event === 'connect.challenge') {
              this.ws.removeEventListener('message', handler);
              resolve(frame.payload.nonce);
            }
          };
          this.ws.addEventListener('message', handler);
        });

        // 发送连接请求
        const response = await this.request('connect', {
          minProtocol: 3,
          maxProtocol: 3,
          client: {
            id: 'web-chat-client',
            displayName: 'Web Chat',
            version: '1.0.0',
            platform: 'web',
            mode: 'operator'
          },
          auth: token ? { token } : undefined
        });

        if (response.ok) {
          this.setStatus('connected', `已连接到 ${response.payload.server.host}`);
          this.ui.input.disabled = false;
          this.ui.send.disabled = false;
          this.ui.input.focus();

          // 加载历史
          this.loadHistory();
        } else {
          this.setStatus('disconnected', `认证失败: ${response.error?.message}`);
        }
      }

      request(method, params) {
        return new Promise((resolve, reject) => {
          const id = `req-${++this.messageId}`;

          this.pendingRequests.set(id, { resolve, reject });

          this.ws.send(JSON.stringify({
            type: 'req',
            id,
            method,
            params
          }));

          setTimeout(() => {
            if (this.pendingRequests.has(id)) {
              this.pendingRequests.delete(id);
              reject(new Error('请求超时'));
            }
          }, 30000);
        });
      }

      handleMessage(frame) {
        if (frame.type === 'res') {
          const pending = this.pendingRequests.get(frame.id);
          if (pending) {
            this.pendingRequests.delete(frame.id);
            if (frame.ok) {
              pending.resolve(frame.payload);
            } else {
              pending.reject(frame.error);
            }
          }
        } else if (frame.type === 'event') {
          this.handleEvent(frame);
        }
      }

      handleEvent(frame) {
        if (frame.event === 'chat') {
          const payload = frame.payload;

          if (payload.state === 'delta') {
            // 流式更新
            const content = payload.message?.content?.[0]?.text || '';
            if (content.length > this.currentMessage.length) {
              this.currentMessage = content;
              this.updateStreamingMessage(content);
            }
          } else if (payload.state === 'final') {
            // 最终消息
            this.currentMessage = '';
            const content = payload.message?.content?.[0]?.text || '';
            this.addMessage('assistant', content);
          }
        }
      }

      async sendMessage() {
        const text = this.ui.input.value.trim();
        if (!text) return;

        this.ui.input.value = '';
        this.addMessage('user', text);

        try {
          await this.request('chat.send', {
            sessionKey: this.sessionKey,
            message: text,
            idempotencyKey: `msg-${Date.now()}`
          });
        } catch (err) {
          this.addMessage('system', `发送失败: ${err.message}`);
        }
      }

      async loadHistory() {
        try {
          const result = await this.request('chat.history', {
            sessionKey: this.sessionKey,
            limit: 50
          });

          this.ui.messages.innerHTML = '';
          for (const msg of result.messages || []) {
            const role = msg.role === 'user' ? 'user' : 'assistant';
            const content = msg.content?.[0]?.text || '';
            if (content) {
              this.addMessage(role, content, false);
            }
          }
        } catch (err) {
          console.error('加载历史失败:', err);
        }
      }

      addMessage(role, content, scroll = true) {
        const div = document.createElement('div');
        div.className = `message ${role}`;
        div.textContent = content;
        this.ui.messages.appendChild(div);

        if (scroll) {
          this.ui.messages.scrollTop = this.ui.messages.scrollHeight;
        }
      }

      updateStreamingMessage(content) {
        let lastMessage = this.ui.messages.lastElementChild;

        if (!lastMessage || !lastMessage.classList.contains('streaming')) {
          lastMessage = document.createElement('div');
          lastMessage.className = 'message assistant streaming';
          this.ui.messages.appendChild(lastMessage);
        }

        lastMessage.textContent = content;
        this.ui.messages.scrollTop = this.ui.messages.scrollHeight;
      }

      setStatus(state, message) {
        this.ui.status.className = state;
        this.ui.status.textContent = message;
      }
    }

    // 初始化
    const chat = new OpenClawChat();

    function reconnect() {
      if (chat.ws) {
        chat.ws.close();
      }
      chat.connect();
    }

    // 自动连接
    chat.connect();
  </script>
</body>
</html>
```

### 8.2 移动应用示例 (React Native)

```typescript
import React, { useState, useEffect, useRef } from 'react';
import { View, Text, TextInput, Button, ScrollView, StyleSheet } from 'react-native';

// 使用 react-native-websocket 或原生 WebSocket
import WebSocket from 'react-native-websocket';

interface Message {
  role: 'user' | 'assistant';
  content: string;
}

export function OpenClawChatScreen() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [status, setStatus] = useState<'connecting' | 'connected' | 'disconnected'>('connecting');
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    connectToGateway();
    return () => {
      wsRef.current?.close();
    };
  }, []);

  const connectToGateway = () => {
    wsRef.current = new WebSocket('ws://your-gateway-host:18789', '');

    wsRef.current.onopen = () => {
      console.log('WebSocket 已连接');
      // 发送连接请求
      sendConnectRequest();
    };

    wsRef.current.onmessage = (event) => {
      const frame = JSON.parse(event.data);
      handleGatewayMessage(frame);
    };

    wsRef.current.onerror = (error) => {
      console.error('WebSocket 错误:', error);
      setStatus('disconnected');
    };

    wsRef.current.onclose = () => {
      console.log('WebSocket 已关闭');
      setStatus('disconnected');
    };
  };

  const sendConnectRequest = () => {
    const connectFrame = {
      type: 'req',
      id: 'connect-1',
      method: 'connect',
      params: {
        minProtocol: 3,
        maxProtocol: 3,
        client: {
          id: 'rn-client',
          displayName: 'React Native App',
          version: '1.0.0',
          platform: 'ios',  // 或 'android'
          mode: 'operator'
        },
        auth: {
          token: 'your-auth-token'
        }
      }
    };

    wsRef.current?.send(JSON.stringify(connectFrame));
  };

  const handleGatewayMessage = (frame: any) => {
    if (frame.type === 'res') {
      if (frame.id === 'connect-1') {
        if (frame.ok) {
          setStatus('connected');
          loadHistory();
        }
      }
    } else if (frame.type === 'event') {
      if (frame.event === 'chat') {
        handleChatEvent(frame.payload);
      }
    }
  };

  const handleChatEvent = (payload: any) => {
    if (payload.state === 'final') {
      const content = payload.message?.content?.[0]?.text || '';
      setMessages(prev => [...prev, { role: 'assistant', content }]);
    }
  };

  const loadHistory = () => {
    const historyFrame = {
      type: 'req',
      id: `history-${Date.now()}`,
      method: 'chat.history',
      params: {
        sessionKey: 'main',
        limit: 50
      }
    };

    wsRef.current?.send(JSON.stringify(historyFrame));
  };

  const sendMessage = () => {
    if (!input.trim()) return;

    const message = input.trim();
    setInput('');
    setMessages(prev => [...prev, { role: 'user', content: message }]);

    const sendFrame = {
      type: 'req',
      id: `send-${Date.now()}`,
      method: 'chat.send',
      params: {
        sessionKey: 'main',
        message,
        idempotencyKey: `msg-${Date.now()}`
      }
    };

    wsRef.current?.send(JSON.stringify(sendFrame));
  };

  return (
    <View style={styles.container}>
      <View style={styles.header}>
        <Text style={styles.title}>OpenClaw Chat</Text>
        <View style={[
          styles.status,
          status === 'connected' && styles.connected,
          status === 'disconnected' && styles.disconnected
        ]}>
          <Text style={styles.statusText}>
            {status === 'connecting' ? '连接中...' :
             status === 'connected' ? '已连接' : '已断开'}
          </Text>
        </View>
      </View>

      <ScrollView style={styles.messages}>
        {messages.map((msg, i) => (
          <View
            key={i}
            style={[
              styles.message,
              msg.role === 'user' ? styles.userMessage : styles.assistantMessage
            ]}
          >
            <Text style={styles.messageText}>{msg.content}</Text>
          </View>
        ))}
      </ScrollView>

      <View style={styles.inputContainer}>
        <TextInput
          style={styles.input}
          value={input}
          onChangeText={setInput}
          placeholder="输入消息..."
          editable={status === 'connected'}
        />
        <Button
          title="发送"
          onPress={sendMessage}
          disabled={status !== 'connected' || !input.trim()}
        />
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
  },
  header: {
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#ddd',
  },
  title: {
    fontSize: 20,
    fontWeight: 'bold',
  },
  status: {
    marginTop: 8,
    padding: 4,
    borderRadius: 4,
    alignSelf: 'flex-start',
  },
  connected: {
    backgroundColor: '#c8e6c9',
  },
  disconnected: {
    backgroundColor: '#ffcdd2',
  },
  statusText: {
    fontSize: 12,
  },
  messages: {
    flex: 1,
    padding: 16,
  },
  message: {
    margin: 8,
    padding: 12,
    borderRadius: 8,
    maxWidth: '80%',
  },
  userMessage: {
    backgroundColor: '#e3f2fd',
    alignSelf: 'flex-end',
  },
  assistantMessage: {
    backgroundColor: '#f5f5f5',
    alignSelf: 'flex-start',
  },
  messageText: {
    fontSize: 16,
  },
  inputContainer: {
    flexDirection: 'row',
    padding: 16,
    borderTopWidth: 1,
    borderTopColor: '#ddd',
  },
  input: {
    flex: 1,
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 4,
    padding: 8,
    marginRight: 8,
  },
});
```

---

## 总结

### 关键要点

1. **Gateway 是中心枢纽**：所有消息通过 Gateway 的 WebSocket (端口 18789) 路由
2. **统一协议**：使用标准化的 JSON 帧格式进行通信
3. **插件化架构**：各渠道通过插件系统无缝集成
4. **会话隔离**：每个对话有独立的 sessionKey
5. **流式响应**：支持通过 `chat` 事件流式接收 Agent 响应

### 实现自定义客户端的关键步骤

1. **获取认证**：从 `~/.openclaw/openclaw.json` 读取 token
2. **WebSocket 握手**：发送 `connect` 请求并等待 `hello-ok`
3. **发送消息**：调用 `chat.send` 方法
4. **接收响应**：监听 `chat` 事件获取流式响应
5. **错误处理**：正确处理超时和断线重连

### 参考文件

| 文件 | 说明 |
|------|------|
| `src/gateway/protocol/schema/frames.ts` | 协议帧定义 |
| `src/gateway/server-methods/chat.ts` | 聊天方法实现 |
| `src/gateway/server-ws-runtime.ts` | WebSocket 处理 |
| `src/channels/dock.ts` | 渠道配置 |
| `src/acp/translator.ts` | ACP 协议转换 |

---

*文档生成日期: 2026-02-12*
*OpenClaw Gateway 版本: 最新*
