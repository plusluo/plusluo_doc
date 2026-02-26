# OpenClaw 防饥饿机制（Starvation Prevention）调研

> **作者：plusluo**
> **日期：2026-02-26**

---

## 一、什么是"饥饿"（Starvation）

在并发调度系统中，**饥饿**是指某些任务因为资源被其他任务持续占用而长期得不到执行的现象。典型场景：

- 高优先级任务源源不断，低优先级任务永远排不上
- 某个用户的请求占满全部资源，其他用户无法得到服务
- 错误/故障导致某条通路被永久锁死

---

## 二、OpenClaw 的整体设计思路

OpenClaw 源码中**没有显式的 "starvation prevention" 术语**（搜索 `starvation`/`starve` 返回 0 结果）。但它通过一系列精心的架构设计，**从根本上消除了产生饥饿的条件**，而不是事后补救。

核心哲学：**不做优先级排序 → 就不会有优先级饥饿**。

---

## 三、11 项防饥饿策略详解

### 策略 1：严格 FIFO 顺序（最核心的保障）

文件：`src/process/command-queue.ts`

```typescript
// 第 74-75 行
while (state.activeTaskIds.size < state.maxConcurrent && state.queue.length > 0) {
  const entry = state.queue.shift() as QueueEntry;
  // shift() = 从队列头部取出，严格先进先出
```

**同一 Lane 内的任务完全按入队时间执行，没有优先级插队机制。**

这是最重要的反饥饿保证——没有优先级队列，就不存在低优先级任务被持续跳过的问题。

#### 举例

```
Main Lane (并发=4):
  队列: [用户A消息, 用户B消息, Cron消息, 用户C消息, 用户D消息]
  
  执行顺序一定是 A → B → Cron → C → D
  不会因为 "Cron 优先级低" 而被跳过
```

---

### 策略 2：Lane 隔离——跨类型防饥饿

文件：`src/process/lanes.ts`、`src/gateway/server-lanes.ts`

通过将不同类型的工作放入**物理隔离的 Lane**，彼此不会互相阻塞：

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Main Lane     │  │  Subagent Lane  │  │   Cron Lane     │
│   并发=4        │  │   并发=8        │  │   并发=1        │
│                 │  │                 │  │                 │
│ 用户A消息       │  │ 子Agent回调     │  │ 每日摘要任务     │
│ 用户B消息       │  │ 工具完成通知     │  │ 定时检查任务     │
│ 用户C消息       │  │                 │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘
     互不阻塞            互不阻塞            互不阻塞
```

#### 举例

```
场景：系统突然涌入 100 条用户消息

没有 Lane 隔离时：
  → 100 条消息占满队列
  → 定时备份任务排在第 101 位
  → 备份任务被饿死，永远执行不到

有 Lane 隔离时：
  → 100 条消息全部进入 Main Lane 排队
  → Cron Lane 完全独立，定时备份正常执行
  → 子 Agent 回调也在 Subagent Lane 独立处理
```

---

### 策略 3：Per-Session Lane——会话间公平

文件：`src/agents/pi-embedded-runner/lanes.ts`

每个会话拥有独立的 Session Lane（并发=1），保证**不同会话之间公平调度**。

```typescript
export function resolveSessionLane(key: string) {
  const cleaned = key.trim() || CommandLane.Main;
  return cleaned.startsWith("session:") ? cleaned : `session:${cleaned}`;
}
```

#### 举例

```
场景：用户A 是重度用户，1 分钟发了 20 条消息；用户B 只发了 1 条

没有 Session Lane 时：
  → 20 条 A 的消息 + 1 条 B 的消息全部进入同一个队列
  → B 的消息排在 A 的 20 条后面
  → B 要等 A 的全部消息处理完才能得到响应（被饿死）

有 Session Lane 时：
  → A 的 20 条消息进入 session:userA lane（并发=1，内部串行）
  → B 的 1 条消息进入 session:userB lane（并发=1）
  → 两个 session lane 同时通过，进入 Main Lane 公平竞争
  → B 的消息和 A 的第一条消息同时竞争执行槽
  → B 不会被 A 的大量消息饿死
```

---

### 策略 4：有界队列 + Summarize 溢出策略

文件：`src/utils/queue-helpers.ts`

当 follow-up 队列满时（默认 cap=20），提供三种溢出策略：

| 策略 | 行为 | 防饥饿效果 |
|------|------|-----------|
| `"summarize"`（默认）| 丢弃最旧的消息，但**保留其摘要** | 被淘汰的消息不会完全丢失 |
| `"new"` | 拒绝新消息入队 | 保护已排队的旧消息 |
| `"old"` | 丢弃最旧的消息 | 可能导致旧消息饥饿 |

`summarize` 策略的关键代码：

```typescript
// queue-helpers.ts 第 194-214 行
`[Queue overflow] Dropped ${params.state.droppedCount} ${noun} due to cap.`
// 附带 summaryLines —— 被淘汰消息的摘要会注入到 Agent 提示中
```

#### 举例

```
场景：Agent 正在处理一个复杂任务（耗时 60 秒），期间用户连发 25 条消息

队列状态：cap=20，已排入 20 条，第 21-25 条触发溢出

summarize 策略（默认）：
  → 丢弃第 1-5 条（最旧的），腾出空间给第 21-25 条
  → 但第 1-5 条的内容摘要被保留："用户之前还提到了 xxx、yyy、zzz"
  → Agent 下一轮运行时能看到这些摘要，不会完全忽略早期消息

new 策略：
  → 第 21-25 条被拒绝入队
  → 前 20 条全部保留，先到先服务
```

---

### 策略 5：Heartbeat 自动丢弃——低价值流量不占资源

文件：`src/auto-reply/reply/queue-policy.ts`

```typescript
if (params.isHeartbeat) {
  return "drop";
}
```

心跳检测消息在有活跃运行时**直接丢弃**，不进入队列，确保低价值的定期探活不与用户实际消息竞争资源。

#### 举例

```
场景：系统每 30 秒发一次心跳，同时用户在排队等待

没有 Heartbeat 丢弃时：
  → 队列: [用户消息, 心跳, 用户消息, 心跳, 心跳, ...]
  → 心跳消息占用了宝贵的队列位置
  → 用户消息可能因队列溢出被丢弃

有 Heartbeat 丢弃时：
  → 心跳在入队前被直接过滤
  → 队列: [用户消息, 用户消息, ...]
  → 用户消息得到更好的服务
```

---

### 策略 6：Auth Profile Round-Robin——API Key 均匀分配

文件：`src/agents/auth-profiles/order.ts`

多个 API key 之间使用 **Round-Robin**（最久未用优先）策略：

```typescript
// order.ts 第 127-130 行
// Otherwise, use round-robin: sort by lastUsed (oldest first)
// lastGood is NOT prioritized - that would defeat round-robin
const sorted = orderProfilesByMode(deduped, store);
```

注意注释中特别强调：**不按 lastGood 排序，因为那样会破坏 round-robin 公平性**。

#### 举例

```
场景：配置了 3 个 OpenAI API Key（A、B、C）

没有 Round-Robin 时：
  → Key A 响应快，被持续使用
  → Key B 和 C 几乎不被使用
  → Key A 很快触发速率限制（429）
  → 系统可用性下降

有 Round-Robin 时：
  → 请求 1 → Key A（lastUsed 最早）
  → 请求 2 → Key B（现在 lastUsed 最早）
  → 请求 3 → Key C
  → 请求 4 → Key A（又最早了）
  → 三个 Key 负载均衡，都不容易触发限速
```

---

### 策略 7：Circuit Breaker 冷却 + 自动恢复——防止资源永久锁死

文件：`src/agents/auth-profiles/usage.ts`

当 Auth Profile 遇到错误时，采用**指数退避冷却**，但冷却期过后会**自动重置错误计数**：

```typescript
// usage.ts 第 211-215 行
// Reset error counters when ALL cooldowns have expired
// so the profile gets a fair retry window
// (circuit-breaker half-open → closed)
```

冷却时间表：`1min → 5min → 25min → 最大 1h`

#### 举例

```
场景：API Key B 因为临时网络问题连续失败 3 次

时间线：
T+0min    Key B 第 3 次失败，进入冷却期（5 分钟）
T+0~5min  Key B 被跳过，A 和 C 分担负载
T+5min    冷却期到期，错误计数重置
T+5.1min  Key B 重新参与 Round-Robin 轮询
          → "公平重试窗口"——如果 B 恢复正常，它不会被永久排除

没有自动恢复时：
  → Key B 一旦出错就被永久标记为 "bad"
  → 即使网络问题已恢复，B 也无法重新被使用
  → 只剩 A 和 C 承担所有负载 → 更容易被压垮
```

---

### 策略 8：模型回退探测——防止降级饥饿

文件：`src/agents/model-fallback.ts`

当主要模型在冷却期时，系统不会永久停留在回退模型上。`shouldProbePrimaryDuringCooldown()` 会**主动探测主要模型**：

```typescript
// model-fallback.ts 第 346-349 行
// For the primary model (i === 0), probe it if the soonest cooldown
// expiry is close or already past. This avoids staying on a fallback
// model long after the real rate-limit window clears.
```

#### 举例

```
场景：用户配置了 GPT-4o（主）→ GPT-4o-mini（备）

时间线：
T+0min   GPT-4o 遇到 429 限速，切换到 GPT-4o-mini
T+5min   限速窗口可能已过，但没人知道
T+5.1min shouldProbePrimaryDuringCooldown() 检测到冷却即将到期
         → 尝试用一个请求探测 GPT-4o
T+5.2min GPT-4o 响应成功 → 恢复使用主要模型

没有探测时：
  → 用户可能一直停留在 GPT-4o-mini（降级模型）
  → 即使 GPT-4o 早已恢复，也无人触发切回
  → 用户体验被"饿死"在低质量模型上
```

---

### 策略 9：Debounce 防消息轰炸

文件：`src/utils/queue-helpers.ts`

```typescript
// 第 111-133 行
waitForQueueDebounce() // 默认安静期 1000ms
```

在处理 follow-up 消息前等待一段安静期，将快速连续的消息**合并处理**，防止每条消息都触发一次完整的 Agent 运行。

#### 举例

```
场景：用户在 1 秒内发了 5 条消息

没有 Debounce 时：
  → 触发 5 次 Agent 运行
  → 每次运行都消耗 LLM 调用配额
  → 队列被 5 个任务塞满，其他用户可能被饿死

有 Debounce 时：
  → 等待 1000ms 安静期
  → 5 条消息合并为 1 次 Agent 运行
  → 只消耗 1 次 LLM 调用
  → 队列压力大幅降低
```

---

### 策略 10：Stuck Session 检测——防止死锁饥饿

文件：`src/logging/diagnostic.ts`

```typescript
// 第 362-373 行
// 诊断心跳每 30 秒检查一次
// 如果会话处于 processing 状态超过 120 秒 → logSessionStuck 警告
```

虽然不是自动恢复机制，但提供了**可观测性**，运维人员可以及时发现并手动干预被卡住的会话。

#### 举例

```
场景：某个 Agent 运行因 LLM 超时而卡死

T+0s     Agent 开始运行
T+30s    诊断心跳检查 → 状态正常（processing < 120s）
T+60s    诊断心跳检查 → 状态正常
T+120s   诊断心跳检查 → processing 已 120s → 触发 WARN 告警
         → 运维收到告警，可手动 /stop 或重启会话
         → 该 session lane 不会被永久锁死
```

---

### 策略 11：Lane 重置（SIGUSR1 热重启后的恢复）

文件：`src/process/command-queue.ts`

```typescript
// 第 207-221 行 resetAllLanes()
// 递增 generation 使旧任务的完成回调失效
// 清空 activeTaskIds
// 保留队列中的待处理任务并立即 drain
```

进程热重启时：已排队的用户任务**不会丢失**，重启后立即继续泵送执行。

#### 举例

```
场景：管理员发送 SIGUSR1 信号进行热重启

重启前队列状态：
  Main Lane: [执行中: 任务A] [排队中: 任务B, 任务C]

重启过程：
  1. generation 从 5 → 6
  2. 任务A 的完成回调检测到 generation 不匹配 → 忽略
  3. activeTaskIds 被清空
  4. 任务B、任务C 仍在队列中
  5. 立即 drain → 任务B 开始执行

结果：任务B 和 C 没有被饿死，热重启不影响排队中的工作
```

---

## 四、多层指数退避汇总

项目中有多处独立的指数退避机制，都服务于防饥饿的整体目标：

| 组件 | 退避时间表 | 文件 |
|------|-----------|------|
| Auth Profile 冷却 | 1min → 5min → 25min → 1h（上限） | `src/agents/auth-profiles/usage.ts` |
| 命令轮询退避 | 5s → 10s → 30s → 60s（上限） | `src/agents/command-poll-backoff.ts` |
| 投递队列重试 | 5s → 25s → 2min → 10min（最多 5 次） | `src/infra/outbound/delivery-queue.ts` |
| Announce 队列 | 2s → 4s → 8s → ... → 60s（上限） | `src/agents/subagent-announce-queue.ts` |
| Auth Rate Limit | 10 次/分钟窗口 → 5 分钟锁定 | `src/gateway/auth-rate-limit.ts` |
| Control Plane | 3 次/60s 窗口 | `src/gateway/control-plane-rate-limit.ts` |

---

## 五、关键代码位置汇总

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/process/command-queue.ts` | 287 | 核心队列引擎（FIFO + 泵送 + generation） |
| `src/process/lanes.ts` | 7 | Lane 类型枚举定义 |
| `src/gateway/server-lanes.ts` | 11 | Lane 并发度配置 |
| `src/agents/pi-embedded-runner/lanes.ts` | 16 | Session/Global Lane 解析 |
| `src/utils/queue-helpers.ts` | 260 | 队列工具（溢出策略 + debounce） |
| `src/auto-reply/reply/queue-policy.ts` | 22 | 队列策略（heartbeat 丢弃） |
| `src/agents/auth-profiles/order.ts` | 186 | Auth Profile Round-Robin 轮询 |
| `src/agents/auth-profiles/usage.ts` | 558 | Circuit Breaker 冷却 + 自动恢复 |
| `src/agents/model-fallback.ts` | 504 | 模型回退 + 主动探测 |
| `src/logging/diagnostic.ts` | 400 | Stuck Session 检测 |
| `src/config/agent-limits.ts` | 23 | 并发限制配置解析 |

---

## 六、总结

OpenClaw 的防饥饿设计哲学是**"预防优于治疗"**——通过架构层面消除产生饥饿的条件，而非事后检测和补救：

| 策略 | 防止的饥饿类型 |
|------|--------------|
| 严格 FIFO | 优先级饥饿（根本不存在优先级） |
| Lane 隔离 | 跨类型饥饿（Cron 不会被用户消息阻塞） |
| Per-Session Lane | 跨用户饥饿（重度用户不会饿死轻度用户） |
| 有界队列 + Summarize | 信息饥饿（被溢出的消息内容不完全丢失） |
| Heartbeat 丢弃 | 资源饥饿（低价值流量不占用队列位置） |
| Round-Robin | API Key 饥饿（所有 Key 均匀分配负载） |
| Circuit Breaker + 自动恢复 | 资源锁死饥饿（故障资源不会被永久排除） |
| 模型回退探测 | 降级饥饿（不会永久停留在低质量模型） |
| Debounce | 队列饥饿（减少无意义的重复 Agent 运行） |
| Stuck 检测 | 死锁饥饿（卡住的会话可被发现并干预） |
| Lane 重置 | 重启饥饿（热重启不丢失排队中的任务） |

**唯一的潜在饥饿风险点**：当 `dropPolicy` 设为 `"old"` 时，旧消息会被直接丢弃以腾出空间给新消息。这是有意的设计权衡，默认的 `"summarize"` 策略通过保留摘要来缓解此问题。
