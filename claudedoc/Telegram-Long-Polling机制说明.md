# Telegram Long Polling 机制说明

## 目录

1. [工作原理](#工作原理)
2. [无需公网 IP 的原因](#无需公网-ip-的原因)
3. [代码位置](#代码位置)
4. [配置说明](#配置说明)
5. [延迟分析](#延迟分析)
6. [故障排查](#故障排查)

---

## 工作原理

OpenClaw 默认使用 **Long Polling** 模式接收 Telegram 消息，这是 Telegram Bot API 的两种工作模式之一。

### 两种模式对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        模式 1: Long Polling (默认)                         │
│                                                                              │
│   你的电脑 (无公网IP)                     Telegram 服务器 (公网)            │
│                                                                              │
│   ┌──────────────┐      轮询请求         ┌──────────────┐                 │
│   │ OpenClaw     │ ◄───────────────────── │              │                 │
│   │  Gateway     │      主动连接          │  Telegram    │                 │
│   │              │ ─────────────────────► │  Bot API     │                 │
│   │              │      获取更新          │              │                 │
│   └──────────────┘                       └──────────────┘                 │
│                                                                              │
│   ✅ 不需要公网 IP                                                         │
│   ✅ 内网环境完全可用                                                       │
│   ⚠️ 有轻微延迟 (几秒到几十秒)                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        模式 2: Webhook (可选)                              │
│                                                                              │
│   你的电脑 (无公网IP)         ❌ 无法接收       Telegram 服务器 (公网)      │
│                                                                              │
│   ┌──────────────┐                                       ┌──────────────┐    │
│   │ OpenClaw     │         Telegram 无法访问            │  Telegram    │    │
│   │              │ ╳════════════════════════════════════╳ │  Bot API     │    │
│   │              │                                       │              │    │
│   └──────────────┘                                       └──────────────┘    │
│                                                                              │
│   ❌ 需要公网 IP 或域名                                                     │
│   ❌ 内网环境无法使用                                                       │
│   ✅ 实时性更好                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Long Polling 时序图

```
时间线
  │
  │────────────────────────────────────────────►
  │        30秒 (timeoutSeconds)              │
  │                                           │
  ▼                                           ▼
你的电脑          Telegram 服务器
    │                  ▲                    │
    │                  │                    │
    │  ───── HTTP GET /getUpdates ─────►  │
    │      (请求消息，timeout=30)            │
    │                                          │
    │  ◄──── 等待中 (最长30秒) ──────────── │
    │                                          │
    │  ◄──── HTTP 200 + 消息 ──────────────── │
    │      (有新消息时立即返回)               │
    │                                          │
  立即处理下一轮轮询 ◄───────────────────────── │
```

---

## 无需公网 IP 的原因

**核心原理：Long Polling 是出站连接**

```
你的电脑                                Telegram Bot API
    │                                       │
    │ ────── HTTP GET /getUpdates ──────► │
    │     (主动请求，你发起连接)             │
    │                                       │
    │ ◄──── HTTP 200 (消息列表) ────────── │
    │     (返回新消息或空)                  │
    │                                       │
    │ 等待几秒...                          │
    │                                       │
    │ ────── HTTP GET /getUpdates ──────► │
    │     (继续轮询)                        │
```

**关键点：**
- 你的电脑**主动发起** HTTP 请求到 Telegram 的服务器
- Telegram 服务器（`api.telegram.org`）有公网 IP，你可以随时访问它
- 这就像用浏览器访问网页一样——你不需要公网 IP，因为你主动连接对方

---

## 代码位置

### 主轮询逻辑

**文件：** `src/telegram/monitor.ts`

```typescript
// src/telegram/monitor.ts:172-183
while (!opts.abortSignal?.aborted) {
  // grammY runner 内部调用 bot.api.getUpdates()
  const runner = run(bot, createTelegramRunnerOptions(cfg));
  await runner.task();
}
```

### Runner 配置

```typescript
// src/telegram/monitor.ts:32-50
export function createTelegramRunnerOptions(cfg: OpenClawConfig): RunOptions<unknown> {
  return {
    sink: {
      concurrency: resolveAgentMaxConcurrent(cfg),
    },
    runner: {
      fetch: {
        // Match grammY defaults
        timeout: 30,  // ← 默认 Long Polling 超时时间：30秒
        // Request reactions without dropping default update types.
        allowed_updates: resolveTelegramAllowedUpdates(),
      },
      // Suppress grammY getUpdates stack traces; we log concise errors ourselves.
      silent: true,
      // Retry transient failures for a limited window before surfacing errors.
      maxRetryTime: 5 * 60 * 1000,
      retryInterval: "exponential",
    },
  };
}
```

### Bot 初始化

**文件：** `src/telegram/bot.ts:112-150`

```typescript
export function createTelegramBot(opts: TelegramBotOptions) {
  // ...
  const bot = new Bot(opts.token, client ? { client } : undefined);
  bot.api.config.use(apiThrottler());
  bot.use(sequentialize(getTelegramSequentialKey));
  // ...
}
```

### Webhook 模式（可选）

**文件：** `src/telegram/webhook.ts` 和 `src/telegram/monitor.ts:153-167`

只有当 `useWebhook=true` 时才使用 Webhook 模式：

```typescript
if (opts.useWebhook) {
  await startTelegramWebhook({
    token,
    accountId: account.accountId,
    config: cfg,
    path: opts.webhookPath,
    port: opts.webhookPort,
    secret: opts.webhookSecret,
    runtime: opts.runtime as RuntimeEnv,
    fetch: proxyFetch,
    abortSignal: opts.abortSignal,
    publicUrl: opts.webhookUrl,
  });
  return;
}
// 默认使用 Long Polling (grammY runner)
```

---

## 配置说明

### Telegram 配置结构

**类型定义：** `src/config/types.telegram.ts`

```typescript
export type TelegramAccountConfig = {
  /** Telegram API client timeout in seconds (grammY ApiClientOptions). */
  timeoutSeconds?: number;
  /** Retry policy for outbound Telegram API calls. */
  retry?: OutboundRetryConfig;
  /** Network transport overrides for Telegram. */
  network?: TelegramNetworkConfig;
  // ... 其他配置
};
```

### 完整配置示例

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "your-bot-token",

      // ========== Long Polling 配置 ==========

      // Long Polling 超时时间（秒）
      "timeoutSeconds": 30,

      // 出站消息重试配置
      "retry": {
        "attempts": 3,           // 最大重试次数
        "minDelayMs": 300,       // 最小延迟
        "maxDelayMs": 30000,     // 最大延迟
        "jitter": 0.1            // 抖动因子
      },

      // 网络配置
      "network": {
        "autoSelectFamily": true  // Node.js DNS 解析优化
      },

      // ========== 其他配置 ==========

      "dmPolicy": "open",
      "allowFrom": ["*"],
      "groupPolicy": "open",
      "textChunkLimit": 4000,
      "streamMode": "partial"
    }
  }
}
```

### 配置文件位置

- **全局配置：** `~/.openclaw/openclaw.json`
- **环境变量：** `TELEGRAM_BOT_TOKEN`（bot token 也可以从环境变量读取）

---

## 延迟分析

### timeoutSeconds 参数说明

**重要：** `timeoutSeconds` 不是轮询间隔，而是单次 `getUpdates` 请求的**最大等待时间**。

| 配置值 | 行为 |
|--------|------|
| 太小（如 5 秒） | API 请求频繁，Telegram 可能限流 |
| 默认（30 秒） | 平衡延迟和 API 使用量 |
| 太大（如 120 秒） | API 请求少，但收到消息延迟最大可能 2 分钟 |

### 实际延迟场景

| 场景 | 延迟 |
|------|------|
| 用户发送消息时，刚好在轮询中 | **< 1 秒**（立即收到） |
| 用户发送消息后，刚完成一次空轮询 | **最多 timeoutSeconds 秒** |
| 平均延迟 | **约 timeoutSeconds / 2** |

### 调整延迟的建议

```json
{
  "channels": {
    "telegram": {
      // 快速响应模式（更频繁 API 调用）
      "timeoutSeconds": 10
    }
  }
}
```

```json
{
  "channels": {
    "telegram": {
      // 省流量模式（较慢响应）
      "timeoutSeconds": 60
    }
  }
}
```

⚠️ **注意：** Telegram Bot API 有速率限制，过于频繁的请求可能被限流。

---

## 故障排查

### 检查 Long Polling 是否正常工作

1. **查看日志：**
   ```bash
   pnpm openclaw logs --follow
   ```

2. **检查 Telegram 连接状态：**
   ```bash
   pnpm openclaw doctor
   ```

3. **手动测试 Bot API：**
   ```bash
   curl "https://api.telegram.org/bot<YOUR_TOKEN>/getMe"
   ```

### 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 收不到消息 | Bot Token 错误 | 检查 `openclaw.json` 中的 `botToken` |
| 收到消息延迟很大 | `timeoutSeconds` 太大 | 减小该值（如改为 10-20） |
| 频繁断线重连 | 网络不稳定 | 检查网络连接，配置 `retry` 参数 |
| "getUpdates conflict" | 另一个进程在轮询 | 确保只有一个 OpenClaw 实例在运行 |

### 网络错误重试策略

```typescript
// src/telegram/monitor.ts:53-58
const TELEGRAM_POLL_RESTART_POLICY = {
  initialMs: 2000,      // 首次重试延迟 2 秒
  maxMs: 30_000,        // 最大延迟 30 秒
  factor: 1.8,          // 指数增长因子
  jitter: 0.25,         // 25% 随机抖动
};
```

发生网络错误时的重试行为：
- 第 1 次：等待约 2 秒后重试
- 第 2 次：等待约 3.6 秒后重试
- 第 3 次：等待约 6.5 秒后重试
- ...
- 最大延迟不超过 30 秒

---

## 总结

1. **OpenClaw 默认使用 Long Polling**，不需要公网 IP 即可接收 Telegram 消息
2. **轮询超时时间**由 `channels.telegram.timeoutSeconds` 配置，默认 30 秒
3. **消息接收延迟**平均约为 `timeoutSeconds / 2`
4. **调整延迟**需要权衡响应速度和 API 使用量

---

*文档更新日期: 2026-02-12*
*OpenClaw 版本: latest*
