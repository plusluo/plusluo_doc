# OpenClaw 对 Pi Agent 的运用思想与关键改动

> plusluo
> 基于 OpenClaw 源码（版本 2026.2.25）分析，所有描述均以实际代码为依据。

---

## 一、背景：Pi Agent 是什么？

**Pi**（全称 Pi Coding Agent）是由 Mario Zechner 开发的开源 AI 编码智能体（`badlogic/pi-mono`），核心功能是：接收用户指令 → 调用 LLM 推理 → 执行工具（读写文件、运行命令等）→ 循环直到任务完成。

Pi 本身是一个独立的 CLI 编码工具，类似 Claude Code、Codex 等。

**OpenClaw 的关键决策**：不重新发明 Agent 引擎，而是将 Pi 作为**嵌入式 SDK** 深度集成，在其之上构建消息 Gateway、多渠道、安全、记忆等企业级能力。

---

## 二、核心运用思想：嵌入式集成而非子进程调用

### 2.1 集成方式对比

| 方式 | 描述 | OpenClaw 的选择 |
|------|------|----------------|
| **子进程调用** | 启动 `pi` CLI 作为子进程，通过 stdin/stdout 通信 | ❌ 未采用 |
| **RPC 调用** | 通过 HTTP/gRPC 调用 Pi 的服务 | ❌ 未采用 |
| **嵌入式 SDK** | 直接 import Pi 的核心库，在同一进程中实例化 | ✅ 采用 |

### 2.2 SDK 依赖关系

OpenClaw 依赖 Pi 的四个 npm 包：

```
// 文件：package.json
// "@mariozechner/pi-agent-core"    — Agent 循环、工具执行、消息类型
// "@mariozechner/pi-ai"            — LLM 抽象层：Model、streamSimple、Provider API
// "@mariozechner/pi-coding-agent"  — 高级 SDK：createAgentSession、SessionManager、内置工具
// "@mariozechner/pi-tui"           — 终端 UI（本地 TUI 模式）
```

### 2.3 嵌入式调用的核心代码

```
// 文件：src/agents/pi-embedded-runner/run/attempt.ts
// 关键调用链：
//   1. createOpenClawCodingTools()  — 组装自定义工具集
//   2. splitSdkTools()              — builtInTools=[] , customTools=全部
//   3. createAgentSession()         — 实例化 Pi 的 AgentSession [来自 pi-coding-agent]
//   4. session.prompt(message)      — Pi 内部处理 LLM 调用 + 工具循环
//
// 这意味着：OpenClaw 完全接管了工具系统，但 Agent 循环（LLM调用→工具执行→再调用）
// 由 Pi SDK 内部处理
```

### 2.4 这种思想的价值

1. **复用成熟的 Agent 循环**：Pi 的 LLM 调用、工具执行、stop_reason 判断、流式输出等逻辑无需重写
2. **保持独立演进**：Pi SDK 升级时，OpenClaw 自动获得改进（如新模型支持、性能优化）
3. **专注于差异化**：OpenClaw 团队专注于 Gateway、多渠道、安全等上层能力
4. **同进程高性能**：无 IPC/网络开销，事件回调零延迟

---

## 三、整体集成架构

```
用户消息 (Telegram/Discord/WebSocket/...)
    │
    ▼
OpenClaw Gateway (路由 + 认证 + 会话管理)
    │
    ▼
runEmbeddedPiAgent()  ← 外层重试循环 [OpenClaw 自研]
    │
    ├── 模型解析 + 认证 Profile 轮换     [OpenClaw 增强]
    ├── Hook 系统 (before_model_resolve)  [OpenClaw 增强]
    │
    ▼
runEmbeddedAttempt()  ← 单次尝试 [OpenClaw 自研]
    │
    ├── 工具组装 (createOpenClawCodingTools)  [OpenClaw 完全接管]
    ├── System Prompt 构建                    [OpenClaw 完全接管]
    ├── Pi 扩展注入 (压缩安全护栏等)           [OpenClaw 增强]
    │
    ▼
createAgentSession() → session.prompt()  ← Pi SDK 核心
    │
    ├── LLM API 调用 (streamSimple)     [Pi SDK 处理]
    ├── 工具调用 → 执行 → 结果回传       [Pi SDK 循环]
    ├── stop_reason / end_turn 判断      [Pi SDK 处理]
    │
    ▼
subscribeEmbeddedPiSession()  ← 事件桥接 [OpenClaw 自研]
    │
    ├── 流式文本处理 (think 标签过滤等)
    ├── 工具执行跟踪
    ├── 压缩事件处理
    │
    ▼
出站投递 → 用户收到回复
```

---

## 四、关键改动详解

### 4.1 改动一：完全接管工具系统

**Pi 原生工具**：`read`、`write`、`edit`、`bash`、`grep`、`find`、`ls` — 7 个编码工具。

**OpenClaw 的改动**：通过 `splitSdkTools()` 将 Pi 的 `builtInTools` 设为空数组，然后注入 40+ 自定义工具。

```
// 文件：src/agents/pi-embedded-runner/tool-split.ts
// splitSdkTools() 的核心逻辑：
//   builtInTools: []        ← 清空 Pi 原生工具
//   customTools: [全部工具]  ← 所有工具作为自定义工具传入
//
// 这样做的原因：OpenClaw 需要对每个工具施加安全策略、
// 沙箱映射、权限控制等，不能使用 Pi 的原生工具
```

```
// 文件：src/agents/pi-tools.ts (528 行)
// createOpenClawCodingTools() 组装的工具分为两大类：
//
// 第一类：Pi 核心工具的增强版
//   read   — 增加：自适应输出大小、图片 MIME 检测、沙箱路径映射
//   write  — 增加：沙箱桥接
//   edit   — 增加：Claude 参数兼容性
//   exec   — 替代 bash：增加 PTY、沙箱容器、后台运行、超时控制、审批集成
//   process — 新增：管理后台 exec 会话 (poll/log/write/kill)
//   apply_patch — 新增：OpenAI 风格多文件补丁 (仅 OpenAI provider 启用)
//
// 第二类：OpenClaw 平台工具 (Pi 完全没有的)
//   browser       — 浏览器自动化
//   canvas        — 可视化画布
//   message       — 消息发送
//   cron          — 定时任务
//   web_search    — 网页搜索
//   web_fetch     — 网页抓取
//   image         — 图像理解
//   memory_search — 语义记忆检索
//   sessions_*    — 多会话管理
//   subagents     — 子 Agent 调度
//   gateway       — 网关操作
//   nodes         — 设备节点控制
//   tts           — 文字转语音
//   ... 以及各渠道专属工具 (Discord/Telegram/Slack/WhatsApp 操作)
```

#### 工具策略流水线

OpenClaw 在工具注入前经过 **6 层策略过滤**：

```
// 文件：src/agents/pi-tools.policy.ts、tool-policy-pipeline.ts
// 工具过滤管线：
//   1. Profile Policy   — coding/messaging/minimal/full 预设
//   2. Provider Policy   — 按 LLM Provider 覆盖
//   3. Agent Policy      — 按 Agent 配置覆盖
//   4. Group Policy      — 按频道/群组级别覆盖
//   5. Sandbox Policy    — 沙箱限制
//   6. Subagent Policy   — 子 Agent 深度限制
```

4 种工具配置预设：

| Profile | 工具范围 |
|---------|---------|
| **minimal** | 仅 `session_status` |
| **coding** | read/write/edit/exec/process/memory/sessions/cron/image 等 |
| **messaging** | sessions/message 等通信工具 |
| **full** | 所有工具（无限制） |

---

### 4.2 改动二：外层重试循环 + 多 Profile 认证轮换

Pi 原生只有单次调用，OpenClaw 在其外面包了一个强大的重试循环。

```
// 文件：src/agents/pi-embedded-runner/run.ts (1164 行，Pi 集成最核心的文件)
//
// runEmbeddedPiAgent() 的 while(true) 循环：
//
// 最大迭代次数：
//   MAX_RUN_LOOP_ITERATIONS = 24 + (authProfileCount * 8)
//   下限 32，上限 160
//
// 重试触发场景：
//   1. Context Overflow → 自动压缩(compaction) → 重试（最多 3 次）
//   2. Auth 失败 (401) → 轮换到下一个 Auth Profile → 重试
//   3. Rate Limit (429) → 轮换 Profile + 冷却 → 重试
//   4. Billing 错误 (402) → 轮换 Profile → 重试
//   5. 请求超时 (408) → 轮换 Profile → 重试
//   6. Thinking Level 不支持 → 自动降级 → 重试
//   7. Tool Result 过大 → 截断 → 重试
//
// 正常终止：
//   Pi SDK 返回 stop_reason = "end_turn" → 收集结果 → 返回
```

**认证轮换机制**：

```
// 文件：src/agents/model-auth.ts (415 行)
//
// 支持多个 Auth Profile（API Key / OAuth / AWS SDK）
// 轮换策略：
//   1. 按优先级排序候选 Profile
//   2. advanceAuthProfile() 切换到下一个可用 Profile
//   3. 失败的 Profile 标记冷却期（cooldown），暂时跳过
//   4. 超时不触发冷却（超时是网络问题，不是认证问题）
//
// 举例：配置了 3 个 Anthropic API Key
//   Key-A 触发 rate limit → 自动切换 Key-B → Key-B 正常 → 继续执行
//   Key-B 也 rate limit → 切换 Key-C → Key-C 正常 → 继续
//   Key-A 冷却期过后重新可用
```

Pi 原生没有这些——它只支持单一认证，失败即报错。

---

### 4.3 改动三：完全接管 System Prompt

Pi 原生的 System Prompt 面向编码场景，OpenClaw 构建了一个**20+ 段落的复合 System Prompt**。

```
// 文件：src/agents/system-prompt.ts (688 行)
// buildAgentSystemPrompt() 构建的 System Prompt 结构：
//
//  1. 身份声明 — "You are a personal assistant running inside OpenClaw."
//  2. Tooling — 工具清单和使用指南
//  3. Tool Call Style — 工具调用风格
//  4. Safety — 安全约束（不追求自我保存、复制、资源获取）
//  5. OpenClaw CLI Reference — CLI 快速参考
//  6. Skills (mandatory) — 52 个技能列表 (<available_skills> 块)
//  7. Memory Recall — 记忆检索指导
//  8. Self-Update — 自我更新能力
//  9. Model Aliases — 模型别名映射
// 10. Workspace — 工作目录和沙箱信息
// 11. Documentation — 文档路径
// 12. Sandbox — 沙箱容器详情
// 13. Authorized Senders — 授权发送者列表
// 14. Date & Time — 时区信息
// 15. Reply Tags — 回复标签（如 ⏸SILENT）
// 16. Messaging — 消息路由指导
// 17. Voice (TTS) — 语音交互提示
// 18. Project Context — 用户上下文（含 SOUL.md 人格文件）
// 19. Silent Replies — 静默回复机制
// 20. Heartbeats — 心跳检测提示
// 21. Reactions — 表情反应指导
// 22. Reasoning Format — 推理格式
```

**三种 Prompt 模式**：

```
// 文件：src/agents/system-prompt.ts
// PromptMode:
//   "full"    — 主 Agent，包含所有段落
//   "minimal" — 子 Agent，精简版（去掉 messaging/heartbeat/voice 等）
//   "none"    — 仅基本身份行
```

Pi 原生只有一个简单的编码系统提示，OpenClaw 将其扩展为涵盖"个人 AI 助手"全场景的复合提示。

---

### 4.4 改动四：事件桥接系统

Pi SDK 的 Agent 循环通过事件发射器报告状态，OpenClaw 构建了一个完整的**事件桥接层**，将 Pi 事件转译为 Gateway 的消息流。

```
// 文件：src/agents/pi-embedded-subscribe.ts (712 行)
// subscribeEmbeddedPiSession() 订阅 Pi 的 session 事件：
//
// 事件类型及处理：
//   message_start   → 重置 assistant 消息状态
//   message_update  → 流式文本处理 + <think>/<final> 标签过滤
//   message_end     → 最终化文本 + 去重
//   tool_execution_start  → 工具开始通知（广播到 WS 客户端）
//   tool_execution_update → 工具中间输出
//   tool_execution_end    → 工具结束 + 结果收集
//   auto_compaction_start → 压缩开始通知
//   auto_compaction_end   → 压缩结束 + 重试 Promise 管理
//   agent_start / agent_end → 生命周期管理
```

**关键处理：`<think>` 标签过滤**

```
// 文件：src/agents/pi-embedded-subscribe.handlers.messages.ts (439 行)
// 问题：某些 LLM 返回的文本中包含 <think>...</think> 推理过程
// OpenClaw 需要：
//   1. 从流式输出中实时识别 <think> 标签（可能跨多个 chunk）
//   2. 过滤掉推理过程，只保留 <final> 之后的最终回复
//   3. 有状态跨块解析（chunk 可能在标签中间被切断）
```

**消息工具去重**：

```
// 文件：src/agents/pi-embedded-subscribe.handlers.tools.ts (437 行)
// 问题：Agent 可能通过 message 工具已经发送了消息，
//       然后又在最终回复中重复相同内容
// OpenClaw 检测这种重复并自动去重，避免用户收到两遍
```

Pi 原生使用 TUI 渲染器处理事件，OpenClaw 将其替换为面向分布式消息系统的事件处理。

---

### 4.5 改动五：增强的上下文压缩系统

Pi SDK 内置了自动压缩（auto compaction），但 OpenClaw 在此基础上做了三层增强。

#### 第一层：Pi SDK 自动压缩（保留）

```
// Pi SDK 内部：当上下文 token 接近模型窗口时自动触发摘要
// OpenClaw 通过事件订阅 auto_compaction_start/end 感知状态
```

#### 第二层：溢出压缩（OpenClaw 增强）

```
// 文件：src/agents/pi-embedded-runner/compact.ts (762 行)
// compactEmbeddedPiSessionDirect()：
//   当 Pi SDK 的自动压缩不够时，OpenClaw 从外部进行显式压缩
//   最多 3 次尝试
//   使用 Pi SDK 的 generateSummary() 和 estimateTokens()
```

#### 第三层：压缩安全护栏（OpenClaw 自研扩展）

```
// 文件：src/agents/pi-extensions/compaction-safeguard.ts (382 行)
// 作为 Pi Extension 注入，增加：
//   • 自适应 token 预算（根据上下文大小动态调整）
//   • 分阶段摘要策略
//   • 工具失败摘要（保留失败原因，避免重复犯错）
//   • 文件操作摘要（保留修改了哪些文件的记录）
```

#### 第四层：上下文裁剪（OpenClaw 自研扩展）

```
// 文件：src/agents/pi-extensions/context-pruning.ts
// 基于 Cache-TTL 的上下文裁剪
// 根据消息的新鲜度和重要性动态裁剪历史
```

**压缩核心逻辑**：

```
// 文件：src/agents/compaction.ts (406 行)
//
// 策略：分块 → 摘要 → 合并
//   BASE_CHUNK_RATIO = 0.4
//   SAFETY_MARGIN = 1.2
//
// 自适应分块：computeAdaptiveChunkRatio()
//   根据消息大小动态调整 chunk 比率
//
// 渐进式降级：
//   全量摘要 → (失败) → 部分摘要(跳过超大消息) → (失败) → 简单描述文本
//
// 重试：retryAsync() 包装，最多 3 次，500ms-5s 退避，20% 抖动
```

---

### 4.6 改动六：多 Provider 适配层

Pi 原生支持多 Provider，但 OpenClaw 增加了大量 Provider 特殊处理。

```
// 文件：src/agents/pi-embedded-runner/run.ts + pi-embedded-helpers/
//
// Provider 特殊处理一览：
//
// Anthropic:
//   • 拒绝魔术字符串清除（某些情况 Claude 会返回特殊拒绝标记）
//   • 回合验证（确保用户/助手角色严格交替）
//   • Claude Code 参数兼容
//   • OAuth 工具名映射
//
// Google/Gemini:
//   • 回合排序修复（applyGoogleTurnOrderingFix）
//   • 工具 Schema 清理（Gemini 对 Schema 格式更严格）
//   • 会话历史清理
//   • 思考签名剥离
//
// OpenAI/Codex:
//   • apply_patch 工具支持（仅此 Provider 启用）
//   • thinking 级别降级
//   • 推理块降级
//
// OpenRouter:
//   • 直通代理模式（任何模型 ID 可用）
//
// Ollama/vLLM:
//   • 绕过 Pi SDK 的 streamSimple，使用 /api/chat 原生接口
//   • 自定义 StreamFn 支持本地模型
```

```
// 文件：src/agents/ollama-stream.ts (552 行)
// 完全自定义的 Ollama 流适配器
// 原因：Ollama 的 API 格式与 OpenAI 兼容层有差异
// OpenClaw 直接调用 Ollama 原生 /api/chat 接口
```

**默认模型配置**：

```
// 文件：src/agents/defaults.ts
// export const DEFAULT_PROVIDER = "anthropic";
// export const DEFAULT_MODEL = "claude-opus-4-6";
// export const DEFAULT_CONTEXT_TOKENS = 200_000;
```

---

### 4.7 改动七：Lane 并发控制

OpenClaw 引入了 **Lane（车道）** 概念来管理 Agent 执行的并发。

```
// 文件：src/process/lanes.ts
// export const enum CommandLane {
//   Main = "main",          // 主执行车道
//   Cron = "cron",          // 定时任务车道
//   Subagent = "subagent",  // 子 Agent 车道
//   Nested = "nested",      // 嵌套调用车道
// }
```

```
// 文件：src/process/command-queue.ts (287 行)
// Lane 是一个命令序列化队列：
//   每个 Lane 有独立的任务队列和 maxConcurrent（默认 1）
//   同一 Lane 内的任务严格串行
//   不同 Lane 的任务可以并行
//
// 双重入队机制：
//   Session Lane: "session:{sessionKey}" → 同一会话串行
//   Global Lane: 控制全局并发
//
// 举例：
//   用户 A 发消息 → 入队 session:agent:main:telegram:direct:userA + Main Lane
//   用户 B 同时发消息 → 入队 session:agent:main:discord:direct:userB + Main Lane
//   → 两个会话可以并行（不同 session lane）
//   → 但同一会话的连续消息串行（相同 session lane）
//
//   Cron 定时任务 → 入队 Cron Lane → 不阻塞 Main Lane
```

Pi 原生是单用户 CLI 工具，没有并发控制概念。OpenClaw 作为多用户 Gateway 必须解决这个问题。

---

### 4.8 改动八：认证桥接

Pi 使用自己的 `auth.json` 格式存储 API 密钥，OpenClaw 使用 `auth-profiles.json`。需要一个桥接层。

```
// 文件：src/agents/pi-auth-json.ts (159 行)
// 将 OpenClaw 的 auth-profiles.json 凭证格式 → Pi 的 auth.json 格式
//
// OpenClaw 格式（多 Profile）：
//   auth-profiles.json = [
//     { id: "profile-1", provider: "anthropic", apiKey: "sk-..." },
//     { id: "profile-2", provider: "openai", apiKey: "sk-..." },
//   ]
//
// Pi 格式（单一）：
//   auth.json = { provider: "anthropic", apiKey: "sk-..." }
//
// 桥接：运行时根据当前活跃 Profile 动态生成 Pi 需要的 auth.json
```

```
// 文件：src/agents/pi-model-discovery.ts (22 行)
// 包装 Pi SDK 的 AuthStorage 和 ModelRegistry
// discoverAuthStorage() → Pi 的 createAuthStorage()
// discoverModels() → Pi 的 ModelRegistry
```

---

### 4.9 改动九：Pi Extension 注入机制

Pi SDK 支持 Extension（扩展），OpenClaw 通过编程方式注入自定义扩展。

```
// 文件：src/agents/pi-embedded-runner/extensions.ts (96 行)
// resolveOpenClawPiExtensions() 返回 ExtensionFactory[]
//
// 当前注入的扩展：
//   1. compaction-safeguard — 压缩安全护栏
//   2. context-pruning      — 上下文裁剪
//
// Pi 原生：从磁盘目录加载扩展
// OpenClaw：编程方式直接注入 ExtensionFactory，无需文件系统
```

---

### 4.10 改动十：工具定义适配器

Pi SDK 内部有两套工具类型，OpenClaw 需要一个适配器来桥接。

```
// 文件：src/agents/pi-tool-definition-adapter.ts (232 行)
//
// Pi SDK 的两套类型：
//   AgentTool (pi-agent-core)     — 底层工具定义
//   ToolDefinition (pi-coding-agent) — 高层工具定义
//
// 差异：
//   AgentTool.execute(params, context)
//   ToolDefinition.call(toolInput, context)
//
// 适配器：convertAgentToolToToolDefinition()
//   将 OpenClaw 注册的 AgentTool 包装为 Pi 需要的 ToolDefinition
//   处理参数映射、上下文传递、错误包装
```

---

## 五、Pi CLI vs OpenClaw 嵌入式对照表

| 方面 | Pi CLI (原生) | OpenClaw (嵌入式) |
|------|-------------|------------------|
| **调用方式** | `pi "你的任务"` 命令行 | `createAgentSession()` SDK 调用 |
| **工具数量** | 7 个编码工具 | 40+ 工具（编码 + 平台 + 渠道） |
| **工具管理** | Pi 内置，不可替换 | 完全接管，6 层策略过滤 |
| **系统提示** | 编码场景单一提示 | 20+ 段落复合提示（含人格/技能/记忆） |
| **会话存储** | `~/.pi/agent/sessions/` | `~/.openclaw/agents/<id>/sessions/` |
| **认证** | 单一凭证 | 多 Profile 轮换 + 冷却 + 故障转移 |
| **重试** | 简单重试 | 32-160 次外层循环（溢出压缩/认证轮换/降级） |
| **压缩** | SDK 自动压缩 | 4 层压缩（SDK + 溢出 + 安全护栏 + 裁剪） |
| **并发** | 单用户串行 | Lane 并发控制（多用户/多会话并行） |
| **扩展加载** | 从磁盘加载 | 编程方式注入 ExtensionFactory |
| **事件处理** | TUI 终端渲染 | 事件桥接 → Gateway WS 广播 |
| **Provider** | 标准支持 | 增强适配（Gemini 回合修复/Ollama 原生流等） |
| **安全** | 基础（工作区限制） | 9 层纵深防御（审批/沙箱/SSRF/审计...） |

---

## 六、源文件索引

### 核心运行器

```
src/agents/
├── pi-embedded.ts                              # 顶层入口 re-export (17 行)
├── pi-embedded-runner.ts                       # 二级入口 re-export (29 行)
├── pi-embedded-runner/
│   ├── run.ts                                  # 外层重试循环 (1164 行) ★核心
│   ├── run/
│   │   ├── attempt.ts                          # 单次尝试 (1416 行) ★核心
│   │   ├── params.ts                           # 运行参数类型 (107 行)
│   │   └── types.ts                            # 结果类型 (56 行)
│   ├── compact.ts                              # 溢出压缩 (762 行)
│   ├── model.ts                                # 模型解析 (153 行)
│   ├── system-prompt.ts                        # 系统提示构建 (104 行)
│   ├── extensions.ts                           # Pi 扩展加载 (96 行)
│   ├── tool-split.ts                           # 工具拆分策略 (18 行)
│   ├── lanes.ts                                # Lane 解析 (16 行)
│   └── types.ts                                # 核心类型 (106 行)
```

### 工具系统

```
src/agents/
├── pi-tools.ts                                 # 工具组装总入口 (528 行)
├── pi-tools.read.ts                            # 自定义 Read 工具 (749 行)
├── pi-tools.policy.ts                          # 工具策略过滤
├── pi-tools.schema.ts                          # Schema 规范化
├── pi-tool-definition-adapter.ts               # 工具类型适配器 (232 行)
├── openclaw-tools.ts                           # 平台工具注册 (194 行)
├── tool-catalog.ts                             # 工具目录与预设 (327 行)
└── tool-policy.ts                              # 工具策略 (206 行)
```

### 事件桥接

```
src/agents/
├── pi-embedded-subscribe.ts                    # 主事件订阅 (712 行)
├── pi-embedded-subscribe.handlers.ts           # 事件分发工厂 (67 行)
├── pi-embedded-subscribe.handlers.lifecycle.ts # 生命周期事件 (87 行)
├── pi-embedded-subscribe.handlers.messages.ts  # 消息事件 (439 行)
├── pi-embedded-subscribe.handlers.tools.ts     # 工具事件 (437 行)
└── pi-embedded-subscribe.handlers.compaction.ts# 压缩事件 (84 行)
```

### 认证与模型

```
src/agents/
├── pi-auth-json.ts                             # 认证桥接 (159 行)
├── pi-model-discovery.ts                       # 模型发现 (22 行)
├── pi-settings.ts                              # 压缩设置 (98 行)
├── model-auth.ts                               # 认证轮换 (415 行)
├── defaults.ts                                 # 默认配置 (7 行)
├── context.ts                                  # 上下文窗口 (196 行)
├── context-window-guard.ts                     # 窗口守卫 (75 行)
├── failover-error.ts                           # 故障转移错误 (241 行)
├── ollama-stream.ts                            # Ollama 原生流 (552 行)
└── compaction.ts                               # 压缩逻辑 (406 行)
```

### Pi 扩展

```
src/agents/pi-extensions/
├── compaction-safeguard.ts                     # 压缩安全护栏 (382 行)
└── context-pruning.ts                          # 上下文裁剪
```

### 系统提示

```
src/agents/
├── system-prompt.ts                            # 系统提示构建 (688 行)
└── system-prompt-params.ts                     # 提示参数类型 (96 行)
```

### 并发控制

```
src/process/
├── lanes.ts                                    # Lane 枚举 (7 行)
└── command-queue.ts                            # 命令队列 (287 行)
```

---

## 七、总结

OpenClaw 对 Pi Agent 的运用体现了一种**"SDK 嵌入 + 全面接管"**的集成思想：

| 维度 | 策略 |
|------|------|
| **复用** | Pi 的 Agent 循环（LLM 调用→工具执行→循环判断）完全复用 |
| **替换** | 工具系统完全替换，从 7 个扩展到 40+ |
| **增强** | 压缩从 1 层扩展到 4 层，认证从单一扩展到多 Profile 轮换 |
| **包装** | 事件系统从 TUI 渲染转为 Gateway 广播 |
| **新增** | 并发控制（Lane）、安全体系（9 层）、消息路由、多渠道等 |

这种架构的核心智慧在于：**把 Agent 的"大脑"（LLM 推理 + 工具循环）交给专业的 Pi SDK，OpenClaw 专注于构建"神经系统"（消息路由）、"感官"（多渠道）、"免疫系统"（安全）和"记忆"（Memory）**。两者各司其职，共同构成一个完整的 AI 助手平台。
