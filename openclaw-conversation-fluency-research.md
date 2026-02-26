# OpenClaw 对话流畅性三大保障机制

> 作者：plusluo
> 调研基于 OpenClaw 2026.2.24 (df9a474) 源码
> 源码仓库：https://github.com/openclaw/openclaw

---

## 概述

OpenClaw 在多渠道（WhatsApp/Telegram/Discord/Slack/电话/macOS）场景下，面临一个核心挑战：**用户的输入不是一次性完成的**——可能分多条发送、可能在 Agent 执行中途改主意、可能在语音通话中有停顿。

为此，OpenClaw 设计了三层保障机制来确保对话流畅性：

| 层次 | 机制 | 核心思想 |
|------|------|----------|
| **第 1 层** | 消息防抖（Inbound Debounce） | 等用户"说完"再提交——合并缓冲窗口内的多条消息 |
| **第 2 层** | Steer 指令注入 | 用户"改主意"时不用等——实时插入新指令到运行中的 Agent |
| **第 3 层** | 语音端点检测（Voice Endpoint Detection） | 判断用户"话说完了"——基于静音时长的语音指令边界检测 |

---

## 第 1 层：消息防抖（Inbound Debounce）

### 1.1 解决什么问题

用户在聊天中经常分多条消息表达一个意思：

```
用户 14:30:01 → "帮我写一个函数"
用户 14:30:02 → "用 Python"
用户 14:30:04 → "要带类型注解的"
```

如果每条都触发一次 LLM 调用，会导致 Agent 反复启动、浪费 token、回复内容混乱。

### 1.2 技术原理

**核心文件**：`src/auto-reply/inbound-debounce.ts`

采用经典的 **trailing debounce** 模式——每次新消息到达时重置定时器，直到静默期满才统一提交：

```
消息到达 → enqueue(item)
  ├── buildKey(item)  → 生成缓冲区 key（channel:account:conversation:sender）
  ├── shouldDebounce(item) 判断是否需要防抖
  │     ├── false → 立即 flush 已有缓冲 + 立即处理当前消息
  │     └── true  → 放入缓冲区，重置定时器
  └── 定时器到期 → onFlush(items[]) 一次性处理缓冲区中所有消息
```

**关键代码概要**（`src/auto-reply/inbound-debounce.ts`）：

```typescript
export function createInboundDebouncer<T>(options: {
  buildKey: (item: T) => string;
  shouldDebounce: (item: T) => boolean;
  resolveMs: (item: T) => number;
  onFlush: (items: T[]) => Promise<void>;
}) {
  const buffers = new Map<string, DebounceBuffer<T>>();

  return {
    enqueue(item: T) {
      const key = options.buildKey(item);
      const canDebounce = options.shouldDebounce(item);
      const ms = options.resolveMs(item);
      
      if (!canDebounce || ms <= 0) {
        // 不可防抖的消息（如带附件），立即 flush 缓冲 + 处理
        flushBuffer(key);
        options.onFlush([item]);
        return;
      }
      // 放入缓冲区，重置定时器
      let buf = buffers.get(key) ?? createBuffer(key);
      buf.items.push(item);
      clearTimeout(buf.timer);
      buf.timer = setTimeout(() => flushBuffer(key), ms);
    },
  };
}
```

**消息合并逻辑**（以 WhatsApp 为例，`src/web/inbound/monitor.ts`）：

```typescript
onFlush: async (entries) => {
  if (entries.length === 1) {
    await options.onMessage(last);  // 单条直接处理
    return;
  }
  // 多条消息体用换行符拼接
  const combinedBody = entries
    .map((entry) => entry.body)
    .filter(Boolean)
    .join("\n");
  // mentions 取并集，元数据用最后一条
  const combinedMessage = { ...last, body: combinedBody, mentionedJids: [...mentioned] };
  await options.onMessage(combinedMessage);
},
```

### 1.3 配置体系

**优先级链**（`resolveInboundDebounceMs()`）：

```
代码覆盖值（overrideMs）
  → per-channel 配置（messages.inbound.byChannel.whatsapp: 5000）
    → 全局默认值（messages.inbound.debounceMs: 2000）
      → 0（未配置则禁用）
```

**配置示例**（`src/config/types.messages.ts`）：

```json5
{
  messages: {
    inbound: {
      debounceMs: 2000,        // 全局默认 2 秒
      byChannel: {
        whatsapp: 5000,        // WhatsApp 5 秒（用户打字慢）
        slack: 1500,           // Slack 1.5 秒
        discord: 1500,         // Discord 1.5 秒
      },
    },
  },
}
```

### 1.4 哪些消息跳过防抖

各渠道通过 `shouldDebounce` 回调控制：

| 渠道 | 跳过防抖的条件 |
|------|--------------|
| WhatsApp | 有媒体附件（`mediaPath` 或 `mediaType`） |
| Telegram | 有附件且非转发；控制命令 |
| Discord | 有附件；控制命令；sticker |
| Slack | 空文本；有附件；控制命令 |

### 1.5 举例

**场景：用户在 WhatsApp 上快速发了 3 条消息**

```
[14:30:01] 用户："帮我查一下"          → 进入缓冲区，启动 5s 定时器
[14:30:03] 用户："北京明天天气"          → 进入缓冲区，重置定时器(+5s)
[14:30:04] 用户："还有后天的"           → 进入缓冲区，重置定时器(+5s)
[14:30:09] 5秒静默期满 → onFlush 被触发
```

**Agent 收到的合并消息**：
```
帮我查一下
北京明天天气
还有后天的
```

Agent 一次性处理整个请求，而非 3 次分别回复。

---

## 第 2 层：Steer 指令注入

### 2.1 解决什么问题

Agent 正在执行任务（比如写一段代码），用户突然发了一条新消息改变需求：

```
用户："帮我写一个排序算法"
Agent：正在用快速排序实现中...
用户："算了，用归并排序吧"        ← 此时 Agent 还在运行
```

传统做法：等 Agent 完成后再处理新消息 → 浪费时间和 token。
Steer 做法：**直接将新指令注入到 Agent 当前的 LLM 对话中**，Agent 立即看到新指令并调整方向。

### 2.2 技术原理

Steer 存在于两个层面：

#### 层面 1：用户 → 主 Agent 的 Steer

**核心数据流**：

```
用户发送新消息（Agent 正在运行中）
  → get-reply-run.ts: 检测 shouldSteer = true
  → agent-runner.ts: 调用 queueEmbeddedPiMessage(sessionId, prompt)
  → runs.ts: 验证 isStreaming && !isCompacting
  → attempt.ts: handle.queueMessage(text) → activeSession.steer(text)
  → pi-agent-core SDK: 在下一个工具边界后注入消息
  → LLM 看到新指令，调整后续行为
```

**关键代码 1 — 队列模式判断**（`src/auto-reply/reply/get-reply-run.ts`）：

```typescript
const shouldSteer = resolvedQueue.mode === "steer" 
                  || resolvedQueue.mode === "steer-backlog";
```

**队列模式定义**（`src/config/types.queue.ts`）：

| 模式 | 行为 |
|------|------|
| `steer` | 立即注入到当前运行中；如果没在流式传输，回退为 followup |
| `steer-backlog` | 立即 steer **并且** 保留消息作为 followup（双保险） |
| `followup` | 排队等当前运行结束 |
| `collect` | 排队消息合并为一个 followup turn（默认模式） |
| `interrupt` | 中断当前运行，执行最新消息（激进模式） |

**关键代码 2 — 执行 Steer**（`src/auto-reply/reply/agent-runner.ts`）：

```typescript
if (shouldSteer && isStreaming) {
    const steered = queueEmbeddedPiMessage(followupRun.run.sessionId, followupRun.prompt);
    if (steered && !shouldFollowup) {
        await touchActiveSessionEntry();
        typing.cleanup();
        return undefined;  // 消息已注入，不需要额外回复
    }
}
```

**关键代码 3 — 消息注入**（`src/agents/pi-embedded-runner/runs.ts`）：

```typescript
export function queueEmbeddedPiMessage(sessionId: string, text: string): boolean {
    const handle = ACTIVE_EMBEDDED_RUNS.get(sessionId);
    if (!handle) return false;          // 没有活跃运行
    if (!handle.isStreaming()) return false;  // 没在流式传输
    if (handle.isCompacting()) return false;  // 正在压缩上下文
    void handle.queueMessage(text);
    return true;
}
```

**关键代码 4 — 对接 Pi SDK**（`src/agents/pi-embedded-runner/run/attempt.ts`）：

```typescript
const queueHandle: EmbeddedPiQueueHandle = {
    queueMessage: async (text: string) => {
        await activeSession.steer(text);   // ← 调用 pi-agent-core SDK 的 steer API
    },
    isStreaming: () => activeSession.isStreaming,
    isCompacting: () => subscription.isCompacting(),
    abort: abortRun,
};
setActiveEmbeddedRun(params.sessionId, queueHandle, params.sessionKey);
```

#### 层面 2：父 Agent → 子代理的 Steer

父 Agent 通过 `subagents` 工具向子代理发送 steer：

**关键代码**（`src/agents/tools/subagents-tool.ts`）：

```typescript
// 限流和安全检查
const STEER_RATE_LIMIT_MS = 2_000;       // 2 秒限流
const MAX_STEER_MESSAGE_CHARS = 4_000;    // 最大 4000 字符

// 步骤 1：标记旧运行，抑制其完成公告
markSubagentRunForSteerRestart(resolved.entry.runId);

// 步骤 2：中止旧运行 + 清空队列
abortEmbeddedPiRun(sessionId);
clearSessionQueues([childSessionKey, sessionId]);

// 步骤 3：等旧运行结束（最多 5 秒）
await callGateway({ method: "agent.wait", params: { runId, timeoutMs: 5_000 } });

// 步骤 4：分发新运行（同一个 session，上下文延续）
const response = await callGateway({
    method: "agent",
    params: { message, sessionKey: resolved.entry.childSessionKey },
});

// 步骤 5：替换子代理注册记录
replaceSubagentRunAfterSteer({ previousRunId, nextRunId, fallback });
```

### 2.3 Steer 的三个前提条件

Steer **只在以下条件同时满足时生效**：

1. ✅ Agent 有活跃运行（`ACTIVE_EMBEDDED_RUNS.has(sessionId)`）
2. ✅ 正在流式传输（`handle.isStreaming() === true`）
3. ✅ 没在压缩上下文（`handle.isCompacting() === false`）

如果条件不满足，消息会回退到 followup 队列排队。

### 2.4 举例

**场景：用户在 Agent 写代码时改了需求**

```
[14:30:00] 用户："帮我写一个冒泡排序的 Python 函数"
[14:30:01] Agent 开始运行，调用编辑器工具写代码...
[14:30:05] 用户："等等，改用快排，而且要带泛型"   ← steer 触发

→ queueEmbeddedPiMessage() 成功
→ activeSession.steer("等等，改用快排，而且要带泛型")
→ Pi SDK 在当前工具调用返回后注入这条消息
→ LLM 在下一轮看到：
    [user] "帮我写一个冒泡排序的 Python 函数"
    [assistant] 调用 edit 工具...
    [tool_result] 写入了冒泡排序代码
    [user] "等等，改用快排，而且要带泛型"    ← 新注入的消息
→ LLM 立即调整：删除冒泡排序，改写为泛型快排
```

用户无需等待 Agent 完成冒泡排序再重新下达指令。

---

## 第 3 层：语音端点检测（Voice Endpoint Detection）

### 3.1 解决什么问题

在语音通话场景中，用户说话有自然的停顿。系统需要判断：**用户是在思考还是已经说完了？**

```
用户："帮我查一下..."  [停顿 0.5 秒]  "...北京明天的天气"
                        ↑
                    这里不应该截断
```

```
用户："帮我查一下北京明天的天气"  [停顿 2 秒]
                                    ↑
                                这里应该提交
```

### 3.2 技术原理

> **重要澄清**：经过源码深入分析，OpenClaw **没有本地 VAD 小模型**。语音端点检测完全依赖**远程/系统级**的静音检测，在不同场景使用不同方案。

#### 方案 1：OpenAI Realtime API 服务端 VAD（电话通话场景）

**核心文件**：`extensions/voice-call/src/providers/stt-openai-realtime.ts`

通过 WebSocket 连接到 OpenAI Realtime API，由 OpenAI 服务端执行 VAD：

```typescript
// 连接 OpenAI Realtime API
const ws = new WebSocket("wss://api.openai.com/v1/realtime?intent=transcription");

// 配置服务端 VAD
ws.send(JSON.stringify({
  type: "transcription_session.update",
  session: {
    turn_detection: {
      type: "server_vad",              // OpenAI 服务端 VAD
      threshold: this.vadThreshold,     // 默认 0.5（0~1，越高越不敏感）
      prefix_padding_ms: 300,           // 语音开始前保留 300ms 缓冲
      silence_duration_ms: this.silenceDurationMs,  // 默认 800ms
    },
    input_audio_format: "g711_ulaw",    // 电话标准格式
    input_audio_transcription: {
      model: "gpt-4o-transcribe",       // 转录模型
    },
  },
}));
```

**事件流程**：

```
电话音频（mu-law 8kHz）
  → Twilio/Telnyx WebSocket 
  → MediaStreamHandler.handleConnection()
  → STTSession.sendAudio()（base64 编码）
  → OpenAI Realtime API（服务端 VAD + 转录）
    → speech_started 事件     → 清空 TTS 播放队列（barge-in 打断）
    → transcription.delta    → 流式部分转录
    → speech_stopped 事件    → 静音超过 800ms，认为说完
    → transcription.completed → 最终转录文本
  → 生成 NormalizedEvent(type: "call.speech")
  → CallManager.processEvent()
  → 自动回复管线
```

**Barge-in（打断）机制**——当用户开始说话时立即中断 Agent 的语音回复：

```typescript
onSpeechStart: () => {
    session.clearTtsQueue();  // 停止当前 TTS 播放
},
```

**配置项**（`extensions/voice-call/src/config.ts`）：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `vadThreshold` | 0.5 | VAD 灵敏度（0~1） |
| `silenceDurationMs` | 800 | 静音多久判为说完（毫秒） |
| `sttModel` | `gpt-4o-transcribe` | 转录模型 |

#### 方案 2：Discord 语音频道静音检测

**核心文件**：`src/discord/voice/manager.ts`

使用 Discord.js 的 `@discordjs/voice` 库内置静音检测：

```typescript
const SILENCE_DURATION_MS = 1_000;   // 1 秒静音
const MIN_SEGMENT_SECONDS = 0.35;    // 最小有效音段 0.35 秒

const stream = entry.connection.receiver.subscribe(userId, {
  end: {
    behavior: EndBehaviorType.AfterSilence,  // 静音后停止
    duration: SILENCE_DURATION_MS,            // 1000ms
  },
});
```

流程：Opus 音频 → 检测到 1 秒静音 → 停止捕获 → PCM 解码 → WAV → 送去转录 → Agent 处理。

#### 方案 3：macOS 语音唤醒（Voice Wake）

**核心文件**：`docs/platforms/mac/voicewake.md`、`src/infra/voicewake.ts`

macOS 端使用 Apple 原生 `Speech` 框架进行语音识别，端点检测基于静音窗口：

| 场景 | 静音阈值 |
|------|----------|
| 正常说话流 | 2.0 秒静音 |
| 只说了唤醒词（等待后续指令） | 5.0 秒静音 |
| 唤醒词与命令之间的"有意义停顿" | ~0.55 秒 |
| 硬性上限 | 120 秒 |

**唤醒词配置**（`src/infra/voicewake.ts`）：

```typescript
// 默认唤醒词列表，可通过配置自定义
const defaultWakeWords = ["openclaw", "claude", "computer"];
```

### 3.3 各场景静音阈值对比

| 场景 | 静音阈值 | 实现方式 | 检测位置 |
|------|----------|----------|----------|
| 电话通话 | 800ms | OpenAI server_vad | 云端（OpenAI） |
| Discord 语音频道 | 1000ms | Discord.js AfterSilence | 本地进程 |
| macOS 唤醒-正常说话 | 2000ms | Apple Speech 框架 | 本地系统 |
| macOS 唤醒-仅唤醒词 | 5000ms | Apple Speech 框架 | 本地系统 |

### 3.4 举例

**场景：电话通话中用户停顿判断**

```
[00:00] 用户开始说话 → OpenAI 检测到 speech_started
       → 触发 barge-in，Agent 停止播放当前回复

[00:02] 用户："帮我预定一下..."
[00:03] [停顿 500ms] ← 小于 800ms，不截断
[00:03.5] 用户："明天下午 3 点的会议室"
[00:05] [停顿 800ms] ← 达到阈值！

→ OpenAI 触发 speech_stopped
→ 最终转录："帮我预定一下明天下午3点的会议室"
→ 生成 call.speech 事件
→ Agent 开始处理
→ TTS 播放回复
```

如果静音阈值设太短（如 200ms），用户思考中的自然停顿会被误判为说完；设太长（如 3s），用户会觉得 Agent 反应迟钝。800ms 是电话场景的平衡点。

---

## 三层机制的协作关系

```
                         ┌──────────────────────────────────┐
                         │          用户输入                 │
                         │  (文字 / 语音 / 多条消息)         │
                         └─────────────┬────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
              文字多条消息        语音通话             Agent运行中
                    │                  │              用户发新指令
                    ▼                  ▼                  │
         ┌─────────────────┐  ┌──────────────────┐       │
         │  第1层           │  │  第3层            │       │
         │  Inbound         │  │  Voice Endpoint   │       │
         │  Debounce        │  │  Detection        │       │
         │                  │  │                   │       │
         │ 缓冲窗口内多条   │  │ 静音超时判断       │       │
         │ 消息合并为一条   │  │ 语音指令边界       │       │
         └────────┬────────┘  └────────┬──────────┘       │
                  │                    │                   │
                  └────────┬───────────┘                   │
                           │                               │
                           ▼                               ▼
                  ┌────────────────┐              ┌───────────────┐
                  │  合并后的单条   │              │   第2层        │
                  │  用户消息      │              │   Steer       │
                  │                │              │               │
                  │  → Agent 处理  │              │ 注入到当前LLM │
                  └────────────────┘              │ 对话上下文中  │
                                                  └───────────────┘
```

| 场景 | 触发机制 |
|------|----------|
| 用户快速发了 3 条文字消息 | 第 1 层 → 合并后一次性提交 |
| 用户在电话里说一句话，中间有自然停顿 | 第 3 层 → 等停顿超过 800ms 才提交 |
| Agent 正在写代码，用户说"改用另一种方案" | 第 2 层 → steer 实时注入 |
| 用户在 Agent 忙时连发 3 条消息（steer 模式） | 第 1 层合并 → 第 2 层 steer 注入 |

---

## 关键代码文件索引

### 消息防抖（Debounce）

| 文件 | 作用 |
|------|------|
| `src/auto-reply/inbound-debounce.ts` | Inbound debounce 核心（createInboundDebouncer + resolveInboundDebounceMs） |
| `src/web/inbound/monitor.ts` | WhatsApp 渠道消息合并实现 |
| `src/telegram/bot-handlers.ts` | Telegram 渠道 debounce（含 forward burst 80ms 专用防抖） |
| `src/discord/monitor/message-handler.ts` | Discord 渠道 debounce |
| `src/slack/monitor/message-handler.ts` | Slack 渠道 debounce |
| `src/config/types.messages.ts` | 配置类型定义（InboundDebounceConfig） |
| `src/utils/queue-helpers.ts` | Queue debounce（waitForQueueDebounce + buildCollectPrompt） |
| `src/auto-reply/reply/queue/state.ts` | 队列默认值（DEFAULT_QUEUE_DEBOUNCE_MS = 1000） |
| `src/auto-reply/reply/queue/drain.ts` | 队列排干逻辑（collect 模式下多消息合并为一个 prompt） |

### Steer 指令注入

| 文件 | 作用 |
|------|------|
| `src/auto-reply/reply/get-reply-run.ts` | Steer 触发判断（shouldSteer） |
| `src/auto-reply/reply/agent-runner.ts` | Steer 执行入口 |
| `src/agents/pi-embedded-runner/runs.ts` | queueEmbeddedPiMessage — 消息注入核心 |
| `src/agents/pi-embedded-runner/run/attempt.ts` | 对接 Pi SDK `activeSession.steer()` |
| `src/agents/tools/subagents-tool.ts` | 子代理 steer 工具（中止→等待→重新分发） |
| `src/agents/subagent-registry.ts` | 子代理注册中心（steer-restart 标记与替换） |
| `src/agents/subagent-announce.ts` | 子代理完成公告的 steer 投递路径 |
| `src/config/types.queue.ts` | 队列模式定义（steer / steer-backlog / followup / collect / interrupt） |

### 语音端点检测

| 文件 | 作用 |
|------|------|
| `extensions/voice-call/src/providers/stt-openai-realtime.ts` | OpenAI Realtime API 服务端 VAD 集成 |
| `extensions/voice-call/src/config.ts` | 语音配置（vadThreshold / silenceDurationMs） |
| `extensions/voice-call/src/media-stream.ts` | 电话音频 WebSocket 双向流管理 |
| `extensions/voice-call/src/webhook.ts` | 语音通话 Webhook 服务初始化 |
| `extensions/voice-call/src/types.ts` | 语音事件类型定义（call.speech / call.silence 等） |
| `src/discord/voice/manager.ts` | Discord 语音频道静音检测（AfterSilence 1000ms） |
| `src/infra/voicewake.ts` | macOS 唤醒词配置管理 |
| `src/tts/tts.ts` | TTS 引擎（Barge-in 打断支持） |

---

## 总结

| 维度 | 消息防抖 | Steer 指令注入 | 语音端点检测 |
|------|----------|----------------|-------------|
| **解决的问题** | 多条文字消息合并 | Agent 运行中途改方向 | 判断语音何时说完 |
| **触发时机** | 消息进入管线前 | Agent 流式传输中 | 音频流实时处理中 |
| **实现层** | OpenClaw 自有代码 | Pi SDK `steer()` API | 远程/系统级（OpenAI/Discord/Apple） |
| **可配置** | debounceMs per-channel | queue mode（steer/collect/followup） | vadThreshold + silenceDurationMs |
| **回退策略** | 禁用时每条独立处理 | 非流式时回退为 followup 排队 | 静音超时兜底 |
