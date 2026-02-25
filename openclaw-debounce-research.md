# OpenClaw 会话防抖机制调研

> 调研时间：2026-02-25
> OpenClaw 版本：2026.2.24 (df9a474)
> 源码仓库：https://github.com/openclaw/openclaw

## 背景

OpenClaw 据说支持"会话防抖"——用户分多条发送消息时，系统等用户发完再统一回复。实际测试发现该功能未生效，通过源码分析找出原因。

## 核心发现

OpenClaw 有 **两层独立的消息缓冲机制**，作用于不同阶段：

### 第一层：Inbound Debounce（入站防抖）

- **源码位置**：`src/auto-reply/inbound-debounce.ts`
- **作用阶段**：消息到达时，提交给 AI 之前
- **机制**：固定时间窗口，纯计时器，无语义检测
- **适用通道**：仅 IM 通道（iMessage、WhatsApp、Telegram、Signal、Discord）
- **不适用**：浏览器 UI、CLI TUI（走 Gateway WebSocket `chat.send` 路径）
- **配置项**：`messages.inbound.debounceMs` 和 `messages.inbound.byChannel`
- **默认值**：`0`（禁用）

工作流程：
```
消息1 → 入buffer，启动 setTimeout(debounceMs)
消息2 → 入buffer，重置 setTimeout
消息3 → 入buffer，重置 setTimeout
debounceMs 到期 → 合并为 "消息1\n消息2\n消息3" → 提交给 AI
```

配置示例（`~/.openclaw/openclaw.json`）：
```json
{
  "messages": {
    "inbound": {
      "debounceMs": 2000,
      "byChannel": {
        "imessage": 2500,
        "whatsapp": 2000
      }
    }
  }
}
```

### 第二层：Followup Queue（后续消息队列）

- **源码位置**：`src/auto-reply/reply/queue/` 目录
- **作用阶段**：AI 正在处理一条消息时，新到达的消息排队等待
- **适用通道**：所有通道（包括 Gateway WebSocket）
- **默认模式**：`collect`（合并排队消息为一个请求）
- **默认 debounce**：`1000ms`
- **配置项**：`messages.queue.mode`、`messages.queue.debounceMs`

当 AI 正在处理消息 A 时：
```
消息B 到达 → isActive=true → 进入 followup queue
消息C 到达 → isActive=true → 进入 followup queue
消息A 处理完成 → scheduleFollowupDrain()
  → waitForQueueDebounce(1000ms) → 等1秒看有无新消息
  → collect 模式：合并 B+C 为一个请求 → 交给 AI
```

## 各客户端行为对比

| 客户端 | 第一条消息 | 后续消息 | 合并方式 |
|--------|-----------|---------|---------|
| 浏览器 UI | 立即处理 | 前端 JS 拦截入 `chatQueue`，逐条串行发送，**不合并** | 无 |
| CLI TUI | 立即处理 | 直达 Gateway → 服务端 followup queue | `collect` 模式合并 |
| IM 通道 + debounceMs>0 | **等待 debounceMs** | 合并到同一批次 | inbound debounce 合并 |

### 浏览器 UI 的前端队列机制

源码：`ui/src/ui/app-chat.ts`

```typescript
// 第 29-31 行
export function isChatBusy(host: ChatHost) {
  return host.chatSending || Boolean(host.chatRunId);
}

// 第 190-193 行
if (isChatBusy(host)) {
  enqueueChatMessage(host, message, attachmentsToSend, refreshSessions);
  return;  // 消息根本没发到 Gateway
}
```

发第一条消息后 `chatSending` 立即为 `true`，后续消息被前端拦截放入 `chatQueue` 数组，等 AI 回复完成后逐条串行发送。

### CLI TUI 的行为

源码：`src/tui/tui-command-handlers.ts`

CLI TUI 没有 `isChatBusy` 检查，消息直接通过 WebSocket 发到 Gateway。但 Gateway 服务端的 `resolveActiveRunQueueAction` 会把它放入 followup queue。好处是 `collect` 模式会把排队的多条消息合并处理。

## 防抖的判断逻辑

`shouldDebounce` 只判断消息结构属性，**不涉及语言检测或语义分析**：

| 判断维度 | 具体检查 | 涉及语义？ |
|---------|---------|----------|
| 有没有文本 | `msg.text` 是否为空 | 否 |
| 有没有媒体/附件 | 图片、文件、位置等 | 否 |
| 是不是斜杠命令 | `/stop`、`/new` 等精确匹配 | 否 |
| 是不是转发消息 | Telegram 专用 | 否 |

## 配置不热更新

`resolveInboundDebounceMs` 在 monitor 启动时只调用一次，修改配置后必须重启 gateway：

```bash
openclaw gateway restart
```

## 业界更优方案

1. **Typing Indicator 感知**：利用"对方正在输入"信号暂停防抖计时器（OpenClaw 有 typing indicator 代码但仅用于输出端）
2. **轻量级正则启发式**：检测不完整语句（如以"的""了""呢"结尾的短句），动态延长等待时间
3. **小模型快速判断**：用极小分类模型判断消息是否完整（<50ms 推理）
4. **自适应动态窗口**：根据用户历史发送间隔自动调整 debounceMs

## 结论

- OpenClaw **有**完整的入站防抖实现，但默认禁用（`debounceMs=0`）
- 防抖**仅对 IM 通道有效**，浏览器 UI 和 CLI TUI 走不同管道
- 浏览器 UI 最差（前端拦截，逐条串行），CLI TUI 其次（followup queue collect 合并），IM 通道最优（真正的防抖合并）
- 防抖是纯计时器，无语义检测能力
