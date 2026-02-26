# OpenClaw 心跳巡检机制调研

> 调研基于 OpenClaw 2026.2.24 (df9a474) 源码
> 源码仓库：https://github.com/openclaw/openclaw

---

## 一、心跳机制是什么？

OpenClaw 内置了一套 **定时巡检机制（Heartbeat）**，即使没有用户消息，也会按固定间隔唤醒 AI，让它检查是否有待办事项需要处理。

核心思路：**每隔一段时间，发一条特殊 prompt 给 AI，让它读取工作区的 `HEARTBEAT.md` 文件，判断有没有需要关注的事情。如果无事发生，AI 回复 `HEARTBEAT_OK`；如果有事，AI 直接输出告警内容。**

---

## 二、默认配置与常量

心跳相关配置位于 `agents.defaults.heartbeat`，当前用户 **未显式配置**，全部使用默认值：

| 常量 | 默认值 | 含义 |
|------|-------|------|
| `DEFAULT_HEARTBEAT_EVERY` | `"30m"` | 巡检间隔，默认 **30 分钟** |
| `DEFAULT_HEARTBEAT_ACK_MAX_CHARS` | `300` | AI 回复中带 HEARTBEAT_OK 时，允许的最大附带文本字符数（超出视为有实质内容） |
| `DEFAULT_HEARTBEAT_TARGET` | `"none"` | 告警投递目标，默认 **不投递到任何频道** |
| `DEFAULT_HEARTBEAT_FILENAME` | `"HEARTBEAT.md"` | 工作区中的心跳任务文件名 |
| `HEARTBEAT_TOKEN` | `"HEARTBEAT_OK"` | AI 无事可报时的固定应答令牌 |

**可配置项一览**（通过 `openclaw.json` 的 `agents.defaults.heartbeat` 设置）：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "prompt": "自定义 prompt（默认使用内置）",
        "target": "none | last | <频道ID>",
        "to": "投递对象",
        "model": "心跳专用模型覆盖",
        "ackMaxChars": 300,
        "activeHours": {
          "start": "09:00",
          "end": "22:00",
          "timezone": "Asia/Shanghai"
        }
      }
    }
  }
}
```

---

## 三、心跳 Prompt —— AI 每次收到什么？

### 3.1 常规巡检 Prompt（默认）

每 30 分钟发送给 AI 的 prompt 原文：

```
Read HEARTBEAT.md if it exists (workspace context). Follow it strictly.
Do not infer or repeat old tasks from prior chats.
If nothing needs attention, reply HEARTBEAT_OK.
```

**中文翻译**：
> 如果工作区中存在 HEARTBEAT.md 文件，请读取它（工作区上下文）。严格按照其中的指示执行。
> 不要从之前的聊天中推测或重复旧任务。
> 如果没有需要关注的事项，请回复 HEARTBEAT_OK。

实际发送时，会自动追加当前时间行：

```
Read HEARTBEAT.md if it exists (workspace context). Follow it strictly.
Do not infer or repeat old tasks from prior chats.
If nothing needs attention, reply HEARTBEAT_OK.
Current time: 2026-02-25 17:30 (Asia/Shanghai)
```

### 3.2 系统 Prompt 中的心跳说明

AI 的系统消息中还嵌入了一段心跳行为说明：

```
## Heartbeats
Heartbeat prompt: <心跳 prompt 文本>
If you receive a heartbeat poll (a user message matching the heartbeat prompt
above), and there is nothing that needs attention, reply exactly:
HEARTBEAT_OK
OpenClaw treats a leading/trailing "HEARTBEAT_OK" as a heartbeat ack
(and may discard it).
If something needs attention, do NOT include "HEARTBEAT_OK"; reply with
the alert text instead.
```

**中文翻译**：
> ## 心跳机制
> 心跳提示词：<心跳 prompt 文本>
> 如果你收到一次心跳轮询（一条与上述心跳 prompt 匹配的用户消息），
> 且没有需要关注的事项，请精确回复：
> HEARTBEAT_OK
> OpenClaw 会将开头/末尾的 "HEARTBEAT_OK" 视为心跳确认（并可能丢弃该回复）。
> 如果有需要关注的事项，**不要**包含 "HEARTBEAT_OK"；直接回复告警内容。

### 3.3 事件驱动的替代 Prompt

除了定时巡检，以下事件也会触发心跳，但使用不同的 prompt：

**异步命令完成（exec-event）**：

```
An async command you ran earlier has completed. The result is shown in
the system messages above. Please relay the command output to the user
in a helpful way. If the command succeeded, share the relevant output.
If it failed, explain what went wrong.
```

> 你之前运行的一个异步命令已完成。结果显示在上方的系统消息中。
> 请以有帮助的方式将命令输出转述给用户。如果命令成功了，分享相关输出。
> 如果失败了，解释哪里出了问题。

**定时任务触发（cron）**：

```
A scheduled reminder has been triggered. The reminder content is:

<事件文本>

Please relay this reminder to the user in a helpful and friendly way.
```

> 一个定时提醒已被触发。提醒内容为：
>
> <事件文本>
>
> 请以友好且有帮助的方式将此提醒转述给用户。

---

## 四、完整执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    心跳调度器 (startHeartbeatRunner)              │
│                                                                 │
│  启动 → 解析所有 agent → 计算 intervalMs → 设置 setTimeout       │
│                                                                 │
│  每 30 分钟触发一次（或被 exec-event / cron / wake 事件触发）      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    前置检查 (Preflight)                           │
│                                                                 │
│  1. heartbeatsEnabled 全局开关是否启用？                           │
│  2. isHeartbeatEnabledForAgent 该 agent 是否启用心跳？             │
│  3. resolveHeartbeatIntervalMs 间隔是否有效？                      │
│  4. isWithinActiveHours 当前是否在活跃时段内？                      │
│  5. getQueueSize(Main) > 0 是否有正在处理的请求？（有则跳过）       │
│  6. HEARTBEAT.md 文件是否有效为空？（空则跳过常规轮询）              │
│                                                                 │
│  任一检查不通过 → 跳过本轮，等下次触发                              │
└────────────────────────┬────────────────────────────────────────┘
                         │ 全部通过
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    构建 Prompt                                    │
│                                                                 │
│  exec-event → buildExecEventPrompt                              │
│  cron       → buildCronEventPrompt                              │
│  interval   → resolveHeartbeatPrompt + 追加当前时间行              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    调用 AI (getReplyFromConfig)                   │
│                                                                 │
│  构建消息上下文：                                                  │
│    Body:     心跳 prompt + 当前时间                               │
│    Provider: "heartbeat" / "exec-event" / "cron-event"          │
│    SessionKey: 心跳专用 session                                   │
│    isHeartbeat: true                                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    处理 AI 回复                                    │
│                                                                 │
│  ┌─ 无回复 ────────────→ 状态 "ok-empty"                         │
│  │                       恢复 session updatedAt                  │
│  │                       清理转录（截断回心跳前大小）                │
│  │                                                               │
│  ├─ 含 HEARTBEAT_OK ──→ 状态 "ok-token"                         │
│  │                       同上处理（视为无事发生）                    │
│  │                                                               │
│  ├─ 重复内容 ──────────→ 状态 "skipped"（24h 内相同内容去重）       │
│  │                                                               │
│  └─ 有实质告警内容 ───→ 状态 "sent"                               │
│                          通过 deliverOutboundPayloads             │
│                          投递到配置的频道                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、触发原因分类

心跳不仅仅是定时轮询，还支持多种事件驱动触发：

| 触发原因 | 说明 | 是否受 HEARTBEAT.md 空文件影响 |
|---------|------|---------------------------|
| `interval` | 定时轮询到期（默认 30 分钟） | 是（空文件则跳过） |
| `exec-event` | 异步命令完成后触发 | 否（绕过文件检查） |
| `cron` | 定时任务（cron job）触发 | 否（绕过文件检查） |
| `wake` | 外部唤醒请求 | 否（绕过文件检查） |
| `hook` | 内部钩子触发 | 否（绕过文件检查） |
| `manual` | 手动触发 | 否 |
| `retry` | 上次失败后重试 | 否 |

---

## 六、HEARTBEAT.md —— 任务定义文件

心跳机制的核心驱动力是工作区中的 `HEARTBEAT.md` 文件。当前内容：

```markdown
# HEARTBEAT.md

# Keep this file empty (or with only comments) to skip heartbeat API calls.
# Add tasks below when you want the agent to check something periodically.
```

**行为规则**：
- 文件 **有效为空**（只有注释和空行）→ `isHeartbeatContentEffectivelyEmpty` 返回 `true` → **跳过常规轮询，不调用 AI，不消耗 token**
- 文件中写有任务内容 → AI 每 30 分钟读取并执行检查
- exec-event / cron / wake 事件 → **无论文件是否为空都会触发**

**使用示例**（在 HEARTBEAT.md 中添加巡检任务）：

```markdown
# HEARTBEAT.md

- 检查 ~/projects/my-app 目录下是否有新的错误日志
- 如果 disk usage 超过 80%，告警提醒我
- 检查 GitHub 仓库 my-org/my-repo 是否有新的 PR 需要 review
```

---

## 七、回复处理的精妙设计

### 7.1 转录清理

心跳结果为 "ok"（无事发生）时，OpenClaw 会：
1. **截断转录文件**（`pruneHeartbeatTranscript`）：将 session 的 `.jsonl` 文件恢复到心跳前的大小，避免无效的心跳对话污染上下文窗口
2. **恢复 updatedAt**（`restoreHeartbeatUpdatedAt`）：避免心跳更新影响 session 的"最近活跃"排序

### 7.2 重复告警去重

如果 AI 连续多次心跳回复了 **相同内容**，且在 24 小时内，后续重复会被 **静默丢弃**（状态 `"skipped"`，原因 `"duplicate"`），避免重复骚扰用户。

### 7.3 可见性控制

```javascript
DEFAULT_VISIBILITY = {
    showOk: false,       // 不显示"一切正常"消息
    showAlerts: true,     // 显示告警消息
    useIndicator: true    // 在 UI 中显示状态指示器
}
```

支持按频道、按账号独立配置可见性。

---

## 八、当前用户环境的心跳状态

根据 `openclaw status` 输出：

```
│ Heartbeat │ 30m (main) │
```

- 心跳 **已启用**，间隔 30 分钟
- 但由于 `HEARTBEAT.md` 有效为空，**常规定时轮询实际不会调用 AI**
- 仅 exec-event / cron 等事件驱动时才会真正唤醒 AI

---

## 九、核心结论

1. **间隔**：默认每 **30 分钟** 触发一次，可通过 `heartbeat.every` 配置
2. **执行内容**：发送 prompt 让 AI 读取 `HEARTBEAT.md`，按其中定义的任务进行检查
3. **省 token 设计**：HEARTBEAT.md 为空时自动跳过 AI 调用；"ok" 结果自动清理转录；重复告警 24h 去重
4. **不仅是定时器**：还承载了异步命令完成通知、cron 定时任务等事件驱动场景
5. **纯 prompt 驱动**：没有硬编码的检查逻辑，所有巡检任务完全由 HEARTBEAT.md 内容决定，AI 自主判断是否有异常
