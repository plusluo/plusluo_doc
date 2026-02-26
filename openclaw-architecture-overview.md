# OpenClaw 整体架构深度解析

> plusluo
> 本文基于 OpenClaw 源码（版本 2026.2.25）进行分析，所有架构描述均以实际代码为依据，避免网络文章中的不准确信息。
> 适用于技术汇报与程序员学习。

---

## 一、项目定位与核心理念

OpenClaw（曾用名 ClawDBot → Moltbot → OpenClaw）是一个**开源的本地优先个人 AI 助手平台**，由 Peter Steinberger 创建，GitHub 120k+ Stars，MIT 许可证。

**一句话定位**：通过一个统一的 Gateway（网关）控制平面，将 AI Agent 能力投射到用户已有的 WhatsApp、Telegram、Slack、Discord 等 15+ 消息平台上，让 AI 助手像一个"数字员工"一样替用户执行实际任务（文件操作、Shell 命令、浏览器控制、日程管理等）。

**核心设计理念**（来自 `VISION.md`）：
- **本地优先（Local-first）**：数据和计算默认在用户自己的机器上运行
- **模块化解耦**：核心网关 + 可插拔通道 + 可插拔技能
- **Agent 不只是聊天**：能执行命令、操作文件、控制浏览器、定时任务

---

## 二、技术栈总览

| 维度 | 选型 |
|------|------|
| **主语言** | TypeScript (ESM, ES2022) |
| **运行时** | Node.js ≥ 22，可选 Bun |
| **包管理** | pnpm 10.23（monorepo workspace） |
| **构建工具** | tsdown（TS bundler）、Vite（Control UI） |
| **测试框架** | Vitest + V8 覆盖率（≥70% 阈值） |
| **Lint/格式化** | Oxlint + Oxfmt |
| **HTTP 服务器** | Express 5.x |
| **WebSocket** | ws 库 |
| **前端 UI** | Lit 3.3 Web Components + Vite 7.3 |
| **Schema 验证** | Zod 4.x + TypeBox |
| **向量数据库** | sqlite-vec（嵌入式） |
| **部署方式** | Docker / Docker Compose / Fly.io / Render / launchd / systemd |
| **原生应用** | Swift 6.x（macOS/iOS）、Kotlin + Jetpack Compose（Android） |

---

## 三、整体架构全景图

```
┌──────────────────────────────────────────────────────────────────────┐
│                           用户交互层                                  │
│  WhatsApp  Telegram  Discord  Slack  Signal  iMessage  WebChat ...   │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐        │
│  │Channel│  │Channel│  │Channel│  │Channel│  │Channel│  │Channel│    │
│  │Plugin │  │Plugin │  │Plugin │  │Plugin │  │Plugin │  │Plugin │    │
│  └───┬───┘  └───┬───┘  └───┬───┘  └───┬───┘  └───┬───┘  └───┬───┘  │
└──────┼──────────┼──────────┼──────────┼──────────┼──────────┼────────┘
       │          │          │          │          │          │
       └──────────┴──────────┴────┬─────┴──────────┴──────────┘
                                  │
┌─────────────────────────────────┼────────────────────────────────────┐
│                        Gateway 控制平面                               │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ HTTP Server │  │ WebSocket    │  │ Config       │                │
│  │ (Express)   │  │ Server (ws)  │  │ Reloader     │                │
│  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘                │
│         │                │                  │                        │
│  ┌──────┴────────────────┴──────────────────┴───────┐                │
│  │              Gateway Runtime State               │                │
│  │  • Auth (token/password/Tailscale)               │                │
│  │  • Broadcast (scope-based WS events)             │                │
│  │  • Chat Run Registry (session queue)             │                │
│  │  • Tool Event Recipients                         │                │
│  └──────────────────────┬───────────────────────────┘                │
│                         │                                            │
│  ┌──────────┐  ┌────────┴───────┐  ┌────────────┐  ┌─────────────┐  │
│  │ Routing  │  │ Session Mgmt  │  │ Heartbeat  │  │ Cron        │  │
│  │ Layer    │  │ Layer         │  │ Runner     │  │ Service     │  │
│  └──────────┘  └───────────────┘  └────────────┘  └─────────────┘  │
└─────────────────────────────────┬────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼────────────────────────────────────┐
│                          Agent 执行层                                 │
│  ┌─────────────────────────────────────────────────────┐             │
│  │              Pi Agent Runtime                       │             │
│  │  • Agent Loop (工具调用循环)                          │             │
│  │  • 50+ Built-in Tools (bash/read/write/edit/grep…)  │             │
│  │  • Compaction (上下文压缩)                            │             │
│  │  • Skills System (52 个技能)                         │             │
│  └──────────────────────┬──────────────────────────────┘             │
│                         │                                            │
│  ┌──────────┐  ┌────────┴───────┐  ┌────────────┐  ┌─────────────┐  │
│  │ Memory   │  │ Security      │  │ Browser    │  │ TTS/STT     │  │
│  │ System   │  │ (Exec审批/沙箱)│  │ Automation │  │ (语音交互)    │  │
│  └──────────┘  └───────────────┘  └────────────┘  └─────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼────────────────────────────────────┐
│                          基础设施层                                    │
│  ┌──────────┐  ┌───────────────┐  ┌────────────┐  ┌─────────────┐   │
│  │ Daemon   │  │ Outbound      │  │ Plugin     │  │ Hook        │   │
│  │ Service  │  │ Delivery      │  │ System     │  │ System      │   │
│  └──────────┘  └───────────────┘  └────────────┘  └─────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 四、各架构环节详解

### 4.1 Gateway 控制平面 — 系统的"大脑"

**作用与价值**：Gateway 是整个系统的核心枢纽，负责接收来自各消息渠道的请求，路由到正确的 Agent 会话，管理认证、广播事件、配置热重载等。所有消息的进出都必须经过 Gateway。

**代码证据**：

Gateway 的启动入口在 `src/gateway/server.impl.ts` 中的 `startGatewayServer()` 函数，启动流程清晰地展示了各子系统的编排：

```
// 文件：src/gateway/server.impl.ts
// 启动流程（简化）：
// 1. 读取/迁移配置 → 2. 加载插件 → 3. 创建 Runtime State
// 4. 设置认证 → 5. 启动侧车服务（Cron/Heartbeat/Channels/Discovery）
// 6. 注册 Agent 事件处理器 → 7. 返回 GatewayServer 实例
```

Gateway 由三大服务组成：

#### 4.1.1 HTTP Server

> **代码位置**：`src/gateway/server-http.ts`

基于 Express 5.x，提供以下路由：
- `/v1/chat/completions` — OpenAI 兼容 Chat API
- `/v1/responses` — OpenResponses API
- `/a2ui/`、`/canvas-host/` — Canvas 可视化工作区
- Control UI 静态资源服务
- Hooks HTTP 端点
- 各通道 HTTP 回调（如 Slack HTTP 请求）

#### 4.1.2 WebSocket Server

> **代码位置**：`src/gateway/server/ws-connection.ts`、`ws-connection/message-handler.ts`

采用 `ws` 库的 `noServer` 模式，核心流程：
1. 客户端连接 → 服务端发送 `connect.challenge`（含 nonce）
2. 客户端回复认证信息 → 验证通过后建立连接
3. 所有后续通信通过 JSON 消息在 WebSocket 上进行

```
// 文件：src/gateway/server/ws-connection.ts
// 连接时发送 challenge 事件：
// wsSend(ws, { type: "connect.challenge", nonce: ... })
// 管理客户端集合：clients: Set<GatewayWsClient>
```

#### 4.1.3 Runtime State（运行时状态）

> **代码位置**：`src/gateway/server-runtime-state.ts`

`createGatewayRuntimeState()` 创建的运行时状态包含：
- `httpServer` / `wss` — 服务器实例
- `clients: Set<GatewayWsClient>` — 已连接的 WS 客户端集合
- `broadcast` / `broadcastToConnIds` — 基于 scope 的事件广播
- `chatRunState` — 聊天运行注册表（按 sessionId 排列的运行队列）
- `dedupe` — 去重映射表

---

### 4.2 认证系统 — 多层安全保障

**作用与价值**：Gateway 暴露在网络上，必须有严格的认证机制防止未授权访问。OpenClaw 实现了多种认证方式以适应不同部署场景。

> **代码位置**：`src/gateway/auth.ts`

**支持的认证模式**（`ResolvedGatewayAuthMode`）：
- `"none"` — 无认证（仅本地开发）
- `"token"` — Bearer Token 认证
- `"password"` — 密码认证
- `"trusted-proxy"` — 信任前置代理

**支持的认证方法**（`GatewayAuthResult.method`）：
- `"tailscale"` — 通过 Tailscale whois API 验证用户身份
- `"device-token"` — 设备 Token（原生应用配对场景）
- 以及上述 token/password/none

**安全防护**：
- 速率限制（`AuthRateLimiter`）防暴力破解
- 本地请求检测（`isLocalDirectRequest`）判断是否环回地址直连
- 可信代理 Header 检测，防止认证绕过

```
// 文件：src/gateway/auth.ts
// 关键类型：
// type ResolvedGatewayAuthMode = "none" | "token" | "password" | "trusted-proxy"
// type GatewayAuthResult = { method: "none"|"token"|"password"|"tailscale"|"device-token"|"trusted-proxy" }
```

---

### 4.3 消息路由层 — 精确的消息分发

**作用与价值**：当来自不同渠道（如 Telegram 私聊、Discord 群组）的消息到达 Gateway 时，路由层负责确定这条消息应该由哪个 Agent 处理、属于哪个会话。

> **代码位置**：`src/routing/`

#### 路由解析核心

> **代码位置**：`src/routing/resolve-route.ts`

**输入**（`ResolveAgentRouteInput`）：channel、accountId、peer（kind + id）、parentPeer、guildId、teamId、memberRoleIds

**匹配优先级**（从高到低）：
1. `binding.peer` — 精确匹配特定对等方
2. `binding.peer.parent` — 匹配父级对等方
3. `binding.guild + roles` — 匹配服务器 + 角色
4. `binding.guild` — 匹配服务器
5. `binding.team` — 匹配团队
6. `binding.account` — 匹配账户
7. `binding.channel` — 匹配渠道
8. `default` — 默认路由

#### Session Key 构建

> **代码位置**：`src/routing/session-key.ts`

会话键格式：`agent:<agentId>:<rest>`

支持多种 DM 范围策略：
- `main` → 所有 DM 共享一个主会话
- `per-peer` → 每个对等方一个会话：`agent:<id>:direct:<peerId>`
- `per-channel-peer` → 每渠道每对等方：`agent:<id>:<channel>:direct:<peerId>`
- `per-account-channel-peer` → 最细粒度：`agent:<id>:<channel>:<accountId>:direct:<peerId>`

群聊/频道键：`agent:<id>:<channel>:<peerKind>:<peerId>`

```
// 文件：src/routing/session-key.ts
// 示例：buildAgentPeerSessionKey()
// DM scope "per-peer" → "agent:main:direct:user123"
// DM scope "per-channel-peer" → "agent:main:telegram:direct:user123"
// 群聊 → "agent:main:discord:guild:123456"
```

---

### 4.4 通道适配层 — 15+ 平台的统一接入

**作用与价值**：OpenClaw 需要与 WhatsApp、Telegram、Discord、Slack、Signal、iMessage 等完全不同 API 和协议的平台通信。通道适配层通过统一的插件契约（`ChannelPlugin`）将每个平台的差异屏蔽掉，让 Gateway 核心无需关心具体平台实现。

#### 4.4.1 通道注册表

> **代码位置**：`src/channels/registry.ts`

```
// 8 个核心内置通道（有序）：
// CHAT_CHANNEL_ORDER = ["telegram", "whatsapp", "discord", "irc",
//                       "googlechat", "slack", "signal", "imessage"]
// 每个通道有元数据：label、docs、blurb、systemImage
// 支持别名：imsg → imessage、gchat → googlechat
```

#### 4.4.2 ChannelPlugin 插件契约

> **代码位置**：`src/channels/plugins/types.plugin.ts`

这是每个通道必须实现的核心接口，包含 **20+ 个适配器**：

| 适配器 | 职责 |
|--------|------|
| `config` | 多账号配置管理（listAccountIds、resolveAccount） |
| `setup` | CLI 设置向导 |
| `pairing` | 设备配对（如 WhatsApp QR 码） |
| `security` | DM 安全策略（allowFrom 白名单） |
| `groups` / `mentions` | 群组策略和 @提及处理 |
| `outbound` | 出站消息投递（direct/gateway/hybrid 三种模式） |
| `gateway` | 网关生命周期（startAccount/stop/QR 登录） |
| `streaming` | 流式消息（块流式传输） |
| `threading` | 线程/话题支持 |
| `heartbeat` | 通道心跳检查 |
| `agentTools` | 通道专属 Agent 工具 |
| `directory` | 联系人/群组目录查询 |
| `resolver` | 目标地址解析 |

#### 4.4.3 Dock 系统（轻量级行为描述）

> **代码位置**：`src/channels/dock.ts`

**设计亮点**：将 ChannelPlugin（重型）和 ChannelDock（轻量）分离。Dock 只包含共享代码路径需要的行为描述（能力声明、分片限制、命令门控等），不引入平台 SDK 等重依赖。

```
// 文件：src/channels/dock.ts
// DOCKS: Record<ChatChannelId, ChannelDock>
// 包含所有 8 个核心通道的轻量级行为配置
// 插件注册的通道通过 buildDockFromPlugin() 动态转换
```

#### 4.4.4 通道生命周期管理

> **代码位置**：`src/gateway/server-channels.ts`

`createChannelManager()` 管理通道的启动/停止，**带指数退避重启策略**：
- 初始重试间隔：5 秒
- 最大重试间隔：5 分钟
- 最大重试次数：10 次
- 支持手动停止标记（防止自动重启）

---

### 4.5 插件系统 — 可扩展的核心

**作用与价值**：插件系统是 OpenClaw 可扩展性的基石，允许第三方通过标准化接口注册新的消息通道、Agent 工具、CLI 命令、HTTP 路由、生命周期钩子等。

> **代码位置**：`src/plugins/`

#### 4.5.1 插件定义与注册

> **代码位置**：`src/plugins/types.ts`

`OpenClawPluginApi` 是扩展与核心交互的唯一接口：

```typescript
// 文件：src/plugins/types.ts（简化）
// OpenClawPluginApi 提供的注册方法：
//   registerChannel()      — 注册消息通道
//   registerTool()         — 注册 Agent 工具
//   registerHook()         — 注册事件钩子
//   registerHttpHandler()  — 注册 HTTP 处理器
//   registerHttpRoute()    — 注册 HTTP 路由
//   registerGatewayMethod()— 注册网关 RPC 方法
//   registerCli()          — 注册 CLI 子命令
//   registerService()      — 注册后台服务
//   registerProvider()     — 注册 LLM 提供者
//   registerCommand()      — 注册控制命令
//   on(hookName, handler)  — 注册生命周期钩子
```

**支持 24 种生命周期钩子**：`before_model_resolve`、`before_prompt_build`、`before_agent_start`、`llm_input/output`、`agent_end`、`before/after_compaction`、`message_received/sending/sent`、`before/after_tool_call`、`session_start/end`、`gateway_start/stop` 等。

#### 4.5.2 插件加载器

> **代码位置**：`src/plugins/loader.ts`

加载流程：
1. 扫描 `extensions/` 目录下的 `openclaw.plugin.json` manifest
2. 动态导入插件入口模块
3. 调用 `plugin.register(api)` 完成注册
4. 支持热重载和运行时动态加载

#### 4.5.3 插件注册表

> **代码位置**：`src/plugins/registry.ts`

`PluginRegistry` 维护所有已注册的插件、通道、工具、钩子的索引，供运行时快速查找。

#### 4.5.4 38 个官方扩展

所有扩展位于 `extensions/` 目录，分为以下类别：

| 类别 | 扩展 |
|------|------|
| **消息通道**（19） | telegram、discord、slack、signal、imessage、whatsapp、line、matrix、nostr、irc、googlechat、msteams、feishu（飞书）、mattermost、synology-chat、nextcloud-talk、tlon、twitch、bluebubbles |
| **记忆系统**（2） | memory-core、memory-lancedb |
| **语音/通话**（2） | voice-call、talk-voice |
| **设备控制**（2） | device-pair、phone-control |
| **认证**（3） | google-gemini-cli-auth、minimax-portal-auth、qwen-portal-auth |
| **诊断**（1） | diagnostics-otel |
| **AI 任务**（2） | llm-task、copilot-proxy |
| **其他**（5） | open-prose、lobster、thread-ownership、zalo、zalouser |

每个通道扩展遵循统一结构：

```
extensions/<channel>/
  ├── index.ts                  # 入口：导出 plugin 定义
  ├── openclaw.plugin.json      # 插件 manifest
  ├── package.json              # npm 包定义
  └── src/
      ├── channel.ts            # ChannelPlugin 实现
      └── runtime.ts            # 运行时引用管理
```

**代码证据**（Telegram 扩展入口）：

```
// 文件：extensions/telegram/index.ts
// const plugin = {
//   id: "telegram",
//   register(api: OpenClawPluginApi) {
//     setTelegramRuntime(api.runtime);
//     api.registerChannel({ plugin: telegramPlugin as ChannelPlugin });
//   }
// };
// export default plugin;
```

---

### 4.6 Agent 执行层 — AI 的"手和脚"

**作用与价值**：Agent 执行层是 OpenClaw 的核心智能引擎，负责接收用户请求、调用 LLM 推理、执行工具操作、管理上下文和会话。

#### 4.6.1 Agent Loop（核心循环）

> **代码位置**：`src/agents/` 目录（429+ 个文件）

Agent 的核心是一个 **工具调用循环**：

```
用户消息 → LLM 推理 → 生成工具调用 → 执行工具 → 结果返回 LLM → 继续推理…
直到 LLM 返回 stop_reason = "end_turn"（无更多工具调用）
```

关键文件：
- `agents/agent-scope.ts` — Agent 作用域管理（工作目录、配置、技能加载）
- `agents/context.ts` — 上下文构建（系统提示 + 用户消息 + 工具结果）
- `agents/compaction.ts` — 上下文压缩（当 token 超限时自动摘要历史对话）

#### 4.6.2 上下文压缩（Compaction）

> **代码位置**：`src/agents/compaction.ts`

当对话历史的 token 数接近模型上下文窗口时，自动触发压缩：
- 保留系统提示和最近的交互
- 中间的历史对话被 LLM 摘要为简短描述
- 支持手动和自动两种触发模式

```
// 文件：src/agents/compaction.ts
// 核心逻辑：检测 token 使用 → 选择压缩区间 → LLM 摘要 → 替换原始消息
```

#### 4.6.3 50+ 内置工具

Agent 可以调用的工具涵盖：
- **文件操作**：read、write、edit（精确匹配 + 唯一替换）
- **Shell 命令**：bash（带安全审批）
- **搜索**：grep、find、ls
- **浏览器控制**：基于 Playwright 的网页自动化
- **通道专属工具**：各平台 SDK 提供的能力

---

### 4.7 记忆系统 — Agent 的长期记忆

**作用与价值**：记忆系统让 Agent 能够跨会话记住用户偏好、重要事实和历史交互，实现真正的"个性化 AI 助手"。

> **代码位置**：`src/memory/`

#### 4.7.1 混合检索架构

> **代码位置**：`src/memory/hybrid.ts`、`src/memory/manager-search.ts`

OpenClaw 采用 **向量语义搜索 + FTS5 全文检索** 的混合检索策略：

```
// 文件：src/memory/hybrid.ts
// 混合检索流程：
// 1. 向量语义搜索（sqlite-vec 嵌入式向量数据库）
// 2. FTS5 全文索引搜索
// 3. MMR (Maximal Marginal Relevance) 去重排序
// 4. 时间衰减加权
```

#### 4.7.2 Embedding 管理

> **代码位置**：`src/memory/embeddings.ts`

- 使用 LLM 提供的 embedding API 生成向量
- 带缓存机制避免重复计算

#### 4.7.3 MMR 去重排序

> **代码位置**：`src/memory/mmr.ts`

Maximal Marginal Relevance 算法确保返回的记忆条目既相关又多样：

```
// 文件：src/memory/mmr.ts
// MMR 公式：score = λ * relevance - (1-λ) * max_similarity_to_selected
// 在相关性和多样性之间取平衡
```

#### 4.7.4 时间衰减

> **代码位置**：`src/memory/temporal-decay.ts`

记忆的重要性随时间衰减，最近的记忆权重更高。

#### 4.7.5 查询扩展

> **代码位置**：`src/memory/query-expansion.ts`

使用 LLM 对用户查询进行扩展，提高检索召回率。

---

### 4.8 心跳系统 — Agent 的主动行为

**作用与价值**：心跳系统让 Agent 能够周期性地"自主苏醒"，检查是否有需要主动通知用户的事项（如定时提醒、状态变化等），而不仅仅是被动响应用户消息。

> **代码位置**：`src/infra/heartbeat-runner.ts`（38KB，系统中最大的基础设施文件之一）

核心机制：

```
// 文件：src/infra/heartbeat-runner.ts
// HeartbeatRunner 的核心功能：
// 1. 多 Agent 心跳：每个 agent 独立间隔和配置
// 2. 活跃时段检测（isWithinActiveHours）：只在指定时间窗口内触发
// 3. 系统事件过滤：exec completion、cron 事件不触发心跳
// 4. 心跳可见性控制：可配置是否在通道中显示心跳消息
// 5. 出站投递：集成 deliverOutboundPayloads 投递心跳结果
// 6. ACK 控制：heartbeat.ackMaxChars 裁剪/抑制心跳输出
// 7. 24h 去重：避免重复通知
```

心跳摘要类型（`HeartbeatSummary`）：
- `enabled` — 是否启用
- `every` — 间隔描述
- `everyMs` — 毫秒间隔
- `prompt` — 心跳检查的 AI 提示词
- `target` — 目标通道
- `model` — 使用的 LLM 模型
- `ackMaxChars` — 确认消息最大字符数

---

### 4.9 定时任务系统 — Agent 的日程表

**作用与价值**：让 Agent 能够在指定时间执行任务，如每天早上汇报天气、每周五生成周报等。

> **代码位置**：`src/cron/`

#### 4.9.1 调度核心

> **代码位置**：`src/cron/service/timer.ts`（878 行）

支持三种调度类型：
- **`at`** — 一次性定时任务
- **`every`** — 固定间隔重复（如 every 30min）
- **`cron`** — 标准 cron 表达式（使用 `croniter` 解析）

```
// 文件：src/cron/types.ts
// CronJobConfig 定义了完整的任务配置：
// schedule（调度表达式）、prompt（AI 执行提示）、
// target（目标通道）、timezone、enabled 等
```

#### 4.9.2 任务管理

> **代码位置**：`src/cron/service/jobs.ts`（635 行）

- 任务持久化存储
- 执行日志记录
- 失败重试策略

---

### 4.10 安全体系 — 多层防护

**作用与价值**：由于 Agent 能执行 Shell 命令和文件操作，安全是 OpenClaw 的重中之重。

#### 4.10.1 命令执行审批

> **代码位置**：`src/infra/exec-approvals.ts`

三级安全策略：
- `"deny"` — 禁止所有命令执行
- `"allowlist"` — 仅允许白名单中的命令
- `"full"` — 允许所有命令（需谨慎）

询问模式：
- `"off"` — 不询问，直接按安全级别执行
- `"on-miss"` — 命令不在白名单时询问用户
- `"always"` — 每次都询问

```
// 文件：src/infra/exec-approvals.ts
// ExecSecurity: "deny" | "allowlist" | "full"
// ExecAsk: "off" | "on-miss" | "always"
// 白名单存储：~/.openclaw/exec-approvals.json
// Socket 通信：~/.openclaw/exec-approvals.sock
```

#### 4.10.2 审批转发

> **代码位置**：`src/infra/exec-approval-forwarder.ts`

将命令审批请求转发到用户的消息渠道（如 Telegram），支持远程审批。

#### 4.10.3 安全审计

> **代码位置**：`src/security/audit.ts`（1021 行）

全面的安全审计系统，记录所有敏感操作。

#### 4.10.4 技能扫描器

> **代码位置**：`src/security/skill-scanner.ts`

扫描第三方技能中的潜在安全风险。

---

### 4.11 出站投递系统 — 可靠的消息送达

**作用与价值**：Agent 产生的回复需要可靠地送达到正确的消息渠道。出站系统处理消息的格式化、分片、重试和投递。

> **代码位置**：`src/infra/outbound/`

#### 4.11.1 投递核心

> **代码位置**：`src/infra/outbound/deliver.ts`

```
// 文件：src/infra/outbound/deliver.ts
// deliverOutboundPayloads() 的核心流程：
// 1. 解析投递目标（来自会话元数据）
// 2. 根据目标通道选择处理器
// 3. 委托给 plugin.outbound 适配器
// 4. 支持 text/media/poll 发送
// 5. 带分块支持（长消息自动拆分）
```

#### 4.11.2 目标解析

> **代码位置**：`src/infra/outbound/targets.ts`

```
// 文件：src/infra/outbound/targets.ts
// resolveSessionDeliveryTarget() 的策略：
// - "last" 模式：回复到最后一个来源通道
// - turnSourceChannel 防止跨通道路由竞态
// - 通过 plugin.outbound.resolveTarget 规范化目标地址
```

---

### 4.12 配置热重载 — 零停机更新

**作用与价值**：生产环境中修改配置不应需要重启整个服务。热重载系统通过文件监视和差异检测，实现配置变更的自动应用。

> **代码位置**：`src/gateway/config-reload.ts`

```
// 文件：src/gateway/config-reload.ts
// 热重载核心机制：
// 1. chokidar 文件监视器（300ms debounce）
// 2. diffConfigPaths(prev, next) 递归比较配置对象变化
// 3. 基于路径前缀的分级策略：
//    - hooks.* → 热重载
//    - gateway.* → 需要完全重启
//    - cron → 热重载
//    - 通道插件通过 plugin.reload.configPrefixes 贡献热重载规则
// 4. GatewayReloadPlan 决定每个子系统的处理方式：
//    - restartGateway / reloadHooks / restartChannels
//    - restartCron / restartHeartbeat / restartBrowserControl
```

---

### 4.13 守护进程管理 — 后台长运行

**作用与价值**：让 OpenClaw Gateway 作为系统服务长期运行在后台，开机自启动。

> **代码位置**：`src/daemon/service.ts`

跨平台统一抽象 `GatewayService`：

| 平台 | 实现 | 文件 |
|------|------|------|
| **macOS** | `launchd`（plist） | `src/daemon/launchd.ts` |
| **Linux** | `systemd`（user service） | `src/daemon/` |
| **Windows** | `Scheduled Task`（schtasks） | `src/daemon/` |

统一接口：`install`、`uninstall`、`stop`、`restart`、`isLoaded`、`readCommand`、`readRuntime`

```
// 文件：src/daemon/service.ts
// resolveGatewayService() 是工厂函数
// 根据 process.platform 返回对应平台实现
```

---

### 4.14 Hook 系统 — 事件驱动的扩展点

**作用与价值**：Hook 系统允许在 Agent 的关键生命周期节点（消息收发、工具调用前后、会话创建等）注入自定义逻辑。

> **代码位置**：`src/hooks/`

#### 内置 Hook

> **代码位置**：`src/hooks/internal-hooks.ts`

内置 Hook 处理器处理核心行为，如 exec-approval 集成、消息接收转发等。

#### Hook 加载器

> **代码位置**：`src/hooks/loader.ts`

支持从配置文件和插件动态加载 Hook 处理器。

---

### 4.15 技能系统 — Agent 的能力包

**作用与价值**：技能（Skills）是预定义的工具集合和提示词模板，让 Agent 快速获得特定领域的能力。

> **代码位置**：`skills/` 目录（52 个技能）

覆盖领域：
- **生产力**：apple-notes、apple-reminders、bear-notes、notion、obsidian、things-mac、trello
- **开发工具**：coding-agent、gh-issues、github、tmux
- **通信**：discord、slack、imsg（iMessage）
- **AI 服务**：gemini、openai-image-gen、openai-whisper
- **媒体**：camsnap（拍照）、peekaboo（截屏）、video-frames、spotify-player
- **工具**：weather、summarize、1password

---

### 4.16 ACP 桥接 — IDE 集成

**作用与价值**：ACP（Agent Client Protocol）是连接 OpenClaw 与 IDE（如 Zed）的标准化协议。

> **代码位置**：`src/acp/`

架构：

```
IDE (ACP Client) ←── stdio/NDJSON ──→ openclaw acp (ACP Server) ←── WebSocket ──→ Gateway
```

核心翻译器（`src/acp/translator.ts`）实现了完整的 ACP Agent 接口：
- `initialize()` — 声明能力
- `newSession()` / `loadSession()` — 会话管理
- `prompt()` — 消息转发
- `cancel()` — 取消请求

安全特性：2MB 提示大小限制、会话创建速率限制（120 次/10 秒）。

---

### 4.17 语音能力 — 语音交互

**作用与价值**：支持语音唤醒和语音对话，让用户像对真人说话一样与 AI 助手交互。

> **代码位置**：`src/tts/tts.ts`（948 行）、`Swabble/`

- **TTS**（文字转语音）：集成 ElevenLabs
- **STT**（语音转文字）：集成 Deepgram
- **Wake Word**：`Swabble/` — 基于 macOS Speech.framework 的本地唤醒词检测（默认 "clawd"）

---

### 4.18 浏览器自动化 — 网页交互

**作用与价值**：让 Agent 能够像人一样操作浏览器，访问网页、填写表单、截图等。

> **代码位置**：`src/browser/`

基于 **Playwright** 的浏览器自动化引擎，Agent 可以通过工具调用来：
- 打开/导航网页
- 点击元素、输入文本
- 截取屏幕截图
- 提取页面内容

---

### 4.19 广播系统 — 实时事件分发

**作用与价值**：将 Gateway 内部的事件实时推送给所有已连接的 WebSocket 客户端（如 Control UI、原生应用）。

> **代码位置**：`src/gateway/server-broadcast.ts`

```
// 文件：src/gateway/server-broadcast.ts
// 基于 scope 的事件广播
// 事件权限控制：
//   exec.approval.* → 需要 operator.approvals 权限
//   设备配对事件 → 需要 operator.pairing 权限
// 慢消费者保护：bufferedAmount 超限时关闭连接或跳过
```

---

### 4.20 优雅关闭 — 有序退出

**作用与价值**：确保系统关闭时所有子系统有序停止，不丢失数据。

> **代码位置**：`src/gateway/server-close.ts`

关闭编排顺序：
1. 停止 Bonjour/Tailscale 发现
2. 关闭 Canvas Host
3. 停止所有频道插件
4. 停止 Gmail Watcher、Cron、Heartbeat
5. 广播 `shutdown` 事件
6. 清理定时器和订阅
7. 关闭所有 WS 客户端（code 1012 "service restart"）
8. 停止配置重载器和浏览器控制
9. 关闭 WS Server 和 HTTP Server

---

## 五、数据流：一条消息的完整旅程

```
1. 用户在 Telegram 发送消息 "帮我写一个 Python 脚本"
   │
2. Telegram Channel Plugin 接收消息
   │  (extensions/telegram/src/channel.ts → gateway adapter)
   │
3. Gateway 接收 → 认证检查
   │  (src/gateway/auth.ts)
   │
4. 路由层解析目标 Agent 和 Session
   │  (src/routing/resolve-route.ts → 匹配 binding 优先级)
   │  生成 sessionKey: "agent:main:telegram:direct:user123"
   │
5. 会话管理：加载/创建会话
   │  (src/sessions/)
   │
6. 记忆检索：查找相关历史记忆
   │  (src/memory/manager-search.ts → 混合检索)
   │
7. Agent Loop 启动
   │  (src/agents/)
   │  构建上下文：系统提示 + 技能 + 记忆 + 用户消息
   │
8. LLM 推理 → 返回工具调用 (bash: "创建 script.py")
   │
9. 安全审批：检查命令是否在白名单
   │  (src/infra/exec-approvals.ts)
   │
10. 执行工具 → 返回结果给 LLM → 继续推理
    │  (循环直到 end_turn)
    │
11. Agent 生成最终回复
    │
12. 出站投递：格式化 + 分片 + 投递到 Telegram
    │  (src/infra/outbound/deliver.ts → telegram plugin.outbound)
    │
13. 用户在 Telegram 收到回复
```

---

## 六、Monorepo 工程结构

```
openclaw/
├── src/                    # 核心源代码 (67 个顶层模块, TypeScript)
│   ├── gateway/            # Gateway 服务器 (186 文件)
│   ├── agents/             # AI 代理核心 (429 文件, 项目最大模块)
│   ├── commands/           # CLI 命令 (248 文件)
│   ├── infra/              # 基础设施 (255 文件)
│   ├── config/             # 配置系统 (173 文件)
│   ├── cli/                # CLI 框架 (142 文件)
│   ├── channels/           # 通道抽象 (152 文件)
│   ├── telegram/           # Telegram 实现 (101 文件)
│   ├── discord/            # Discord 实现 (123 文件)
│   ├── slack/              # Slack 实现 (88 文件)
│   ├── web/                # WhatsApp 实现 (83 文件)
│   ├── memory/             # 记忆系统
│   ├── plugins/            # 插件系统
│   ├── routing/            # 路由层
│   ├── sessions/           # 会话管理
│   ├── hooks/              # Hook 系统
│   ├── cron/               # 定时任务
│   ├── daemon/             # 守护进程
│   ├── security/           # 安全模块
│   ├── browser/            # 浏览器自动化
│   ├── tts/                # 语音
│   ├── acp/                # ACP 桥接
│   └── ...                 # 更多模块
├── extensions/             # 38 个可插拔扩展
├── skills/                 # 52 个 Agent 技能
├── ui/                     # Control UI (Lit + Vite)
├── apps/                   # 原生应用
│   ├── macos/              # Swift macOS 菜单栏应用
│   ├── ios/                # Swift iOS 应用
│   ├── android/            # Kotlin Android 应用
│   └── shared/             # 跨平台共享 (OpenClawKit)
├── vendor/a2ui/            # A2UI 可视化工作区标准
├── Swabble/                # Swift 语音唤醒守护进程
├── docs/                   # Mintlify 文档 (含中文/日文翻译)
├── scripts/                # 88+ 构建/测试/部署脚本
└── test/                   # 测试基础设施
```

---

## 七、关键设计模式总结

| 模式 | 体现 | 代码证据 |
|------|------|----------|
| **插件化架构** | 通道、工具、钩子都通过插件注册 | `src/plugins/types.ts` — `OpenClawPluginApi` |
| **适配器模式** | 20+ 适配器接口统一各通道差异 | `src/channels/plugins/types.plugin.ts` |
| **事件驱动** | 24 种生命周期钩子 | `src/plugins/types.ts` — on() 方法 |
| **配置驱动** | 热重载 + 路径级差异检测 | `src/gateway/config-reload.ts` |
| **分层解耦** | Gateway / Agent / Channel 三层分离 | `src/gateway/` + `src/agents/` + `src/channels/` |
| **混合检索** | 向量搜索 + FTS5 + MMR 去重 | `src/memory/hybrid.ts` + `src/memory/mmr.ts` |
| **优雅降级** | 通道重启指数退避、慢消费者保护 | `src/gateway/server-channels.ts`、`server-broadcast.ts` |
| **跨平台抽象** | 守护进程 launchd/systemd/schtasks 统一接口 | `src/daemon/service.ts` |
| **安全纵深** | 认证 + 审批 + 白名单 + 审计 + 扫描 | `src/gateway/auth.ts` + `src/infra/exec-approvals.ts` + `src/security/` |

---

## 八、与 claw0 教学项目的对照

`claw0`（`/Users/plusluo/Documents/code/claw0/`）是 OpenClaw 的教学简化版，通过 10 个渐进式 Python 脚本复现核心架构思想：

| 阶段 | claw0 教学版 | OpenClaw 生产版 |
|------|-------------|----------------|
| **s01: Agent Loop** | 简单 `while True` + `stop_reason` | Lane-based 并发、retry 洋葱模型 |
| **s02: Tool Use** | 4 个基础工具 | 50+ 工具 + 安全策略 |
| **s03: Sessions** | JSON 文件持久化 | JSONL 转录 + sessions.json 元数据 |
| **s04: Multi-Channel** | CLI + 文件模拟 | 15+ 真实消息平台 |
| **s05: Gateway** | websockets + JSON-RPC 2.0 | Express + ws，插件 HTTP 路由，TLS |
| **s06: Routing** | 优先级绑定 | 多层级路由 + 身份关联 |
| **s07: Soul & Memory** | SOUL.md + TF-IDF 搜索 | SQLite-vec + FTS5 + embedding |
| **s08: Heartbeat** | Thread + timer | 6 步检查链、Lane 互斥、24h 去重 |
| **s09: Cron** | 3 种调度类型 | 完整 cron 解析器、时区、SQLite 日志 |
| **s10: Delivery** | 文件队列 + 指数退避 | SQLite 队列、jitter、优先级、批量投递 |

---

## 九、总结

OpenClaw 是一个**架构精良、模块化程度极高**的 AI Agent 平台。其核心竞争力在于：

1. **Gateway 控制平面**：WebSocket + HTTP 双协议，统一管理所有消息流和 Agent 生命周期
2. **插件化通道架构**：20+ 适配器接口的 ChannelPlugin 契约，让新通道的接入成本极低
3. **混合记忆系统**：向量搜索 + 全文检索 + MMR + 时间衰减，实现高质量记忆检索
4. **安全纵深**：从认证到命令审批到安全审计，多层防护确保 Agent 可控
5. **可观测性**：热重载、广播系统、诊断 Hook 让运维具备完整可见性
6. **跨平台覆盖**：TypeScript 核心 + Swift/Kotlin 原生应用 + 语音唤醒，覆盖全端场景
