# OpenClaw 最受欢迎的外部 Tool 与 Skill 调研

> 作者：plusluo
> 调研基于 OpenClaw 2026.2.24 (df9a474) 源码 + 社区生态
> 源码仓库：https://github.com/openclaw/openclaw
> 技能市场：https://clawhub.ai

---

## 概述

OpenClaw 的能力扩展分为两个层面：

| 层面 | 概念 | 类比 |
|------|------|------|
| **Tool（工具）** | 内置于系统的原子能力，Agent 通过 `tool_use` 直接调用 | 操作系统的系统调用 |
| **Skill（技能）** | 社区/用户编写的 Markdown 指令包，教会 Agent 如何组合工具完成特定任务 | 应用商店里的 App |

Tool 由 `src/agents/tools/` 和 `extensions/` 提供，共 25 个内置核心工具 + 36 个插件扩展。
Skill 通过 ClawHub 技能市场分发，社区已积累 **2800+** 个技能（awesome-openclaw-skills 收录数）。

**入选标准**：综合考虑以下四个维度——
1. **实用性评价**：社区讨论中的好评度和推荐频率
2. **使用量**：ClawHub 下载量 / GitHub Issue 讨论热度
3. **功能强大**：能力范围和技术深度
4. **核心必选**：作为 OpenClaw 核心体验不可或缺的部分

---

## 第一部分：Top 8 Tool（内置工具）

### 1. web_search — 联网搜索

**实现文件**：`src/agents/tools/web-search.ts`（1439 行）

**为什么入选**：Agent 从"离线聊天"到"联网 Agent"的分水岭。没有它，Agent 只能依赖训练数据，无法获取实时信息。

**核心能力**：
- 支持 **5 种搜索后端**：Brave Search、Perplexity、Grok、Gemini、Kimi
- 自动回退：首选后端失败时自动切换到备选
- 每次搜索返回结构化结果（标题、摘要、URL）
- 支持配置 `maxResults`、`country`、`freshness` 等参数

**关键代码概要**：

```typescript
// web-search.ts — 多后端搜索调度
const BACKENDS = ["brave", "perplexity", "grok", "gemini", "kimi"] as const;

async function executeSearch(params: { query: string; backend?: string }) {
  const preferred = resolveBackend(params.backend);
  try {
    return await backends[preferred].search(params.query);
  } catch {
    // 自动回退到下一个可用后端
    return await fallbackSearch(params.query, preferred);
  }
}
```

**典型场景**：

```
用户："今天 NVIDIA 股价多少？"
Agent → web_search({ query: "NVIDIA stock price today" })
     → 返回实时搜索结果
     → Agent 整理回复
```

---

### 2. browser — 浏览器自动化

**实现文件**：`src/agents/tools/browser-tool.ts`（830 行）+ `src/browser/`（35+ 文件，7500+ 行）

**为什么入选**：让 Agent 能像人一样操控真实浏览器，是 OpenClaw "Agent 不只是聊天"理念的技术支柱。社区评价：*"browser tool is what makes OpenClaw different from ChatGPT"*。

**核心能力**：
- 14 种操作：`navigate`、`click`、`type`、`screenshot`、`snapshot`（无障碍树）、`scroll`、`evaluate`（执行 JS）、`download` 等
- 两种驱动模式：Playwright 全功能 / CDP 直连
- AI 辅助交互：用自然语言描述要操作的元素
- 支持 Profile 持久化（复用登录状态）

**关键代码概要**：

```typescript
// browser-tool.ts — 操作分发
switch (params.action) {
  case "navigate":   return await client.navigate(params.url);
  case "click":      return await client.click(params.coordinate ?? params.selector);
  case "screenshot": return await client.screenshot();  // 返回 base64 图片
  case "snapshot":   return await client.snapshot();     // 返回无障碍树 JSON
  case "evaluate":   return await client.evaluate(params.expression);
  // ...14 种操作
}
```

**典型场景**：

```
用户："帮我在 GitHub 上给 openclaw 仓库点个 star"
Agent → browser({ action: "navigate", url: "https://github.com/openclaw/openclaw" })
     → browser({ action: "snapshot" })  // 获取页面结构
     → browser({ action: "click", selector: "button[aria-label='Star']" })
     → "已完成，已经帮你 star 了 openclaw 仓库"
```

---

### 3. exec — Shell 命令执行

**实现文件**：由 `pi-agent-core` SDK 提供（Profile: coding）

**为什么入选**：Agent 与操作系统交互的唯一通道。没有它，Agent 只能读写文件但无法运行任何程序。社区共识：*"exec is the hands of the agent"*。

**核心能力**：
- 执行任意 Shell 命令（bash/zsh/powershell）
- 支持超时控制、工作目录设置
- 整合 exec-approvals 安全审批（三级：deny/allowlist/full）
- 沙箱模式下在 Docker 容器中隔离执行

**安全机制**：

```typescript
// 危险命令黑名单（部分）
const DANGEROUS_COMMANDS = [
  /\brm\s+-rf\s+\//, /\bmkfs\./, /\bdd\s+if=/, /\bformat\s+/,
  // ... 更多模式
];

// 三级审批：
// Level 1 (deny):      所有命令都需审批
// Level 2 (allowlist):  白名单内自动通过，其余需审批
// Level 3 (full):       所有命令自动通过（仅限信任环境）
```

**典型场景**：

```
用户："帮我创建一个 React 项目"
Agent → exec({ command: "npx create-react-app my-app", timeout: 60000 })
     → exec({ command: "cd my-app && npm start" })
```

---

### 4. memory_search + memory_get — 语义记忆

**实现文件**：`src/agents/tools/memory-tool.ts`（243 行）+ `extensions/memory-lancedb/`

**为什么入选**：让 Agent 拥有"长期记忆"。用户的偏好、历史交互、项目上下文都能被记住和召回。OpenClaw 官方文档称其为 *"the soul of your agent"*。

**核心能力**：
- `memory_search`：语义搜索 `MEMORY.md` 和 `memory/*.md` 文件
- `memory_get`：读取记忆文件的指定片段
- LanceDB 向量后端：OpenAI 嵌入 → 向量化存储 → 相似度检索
- 自动捕获与召回：Agent 自动将重要信息写入记忆，后续对话自动召回

**关键代码概要**：

```typescript
// memory-tool.ts — 语义搜索
async function memorySearch(params: { query: string; limit?: number }) {
  // 1. 搜索 MEMORY.md 中的显式记忆
  const explicitResults = await searchMemoryFile(params.query);
  // 2. 搜索 memory/*.md 中的主题记忆
  const topicResults = await searchMemoryDir(params.query);
  // 3. LanceDB 向量搜索（如果启用）
  const vectorResults = await lancedb?.search(params.query, params.limit);
  return mergeAndRank([...explicitResults, ...topicResults, ...vectorResults]);
}
```

**典型场景**：

```
用户（3天前）："我喜欢用 TypeScript 而不是 JavaScript"
→ Agent 写入 MEMORY.md

用户（今天）："帮我写一个 HTTP 服务"
Agent → memory_search({ query: "programming language preference" })
     → 召回："用户偏好 TypeScript"
     → 自动用 TypeScript 编写
```

---

### 5. message — 多平台消息发送

**实现文件**：`src/agents/tools/message-tool.ts`（698 行）

**为什么入选**：OpenClaw 的核心差异化——Agent 不只是聊天工具，还能**主动**通过 21 个平台发送消息。这是 OpenClaw 区别于其他 AI Agent 的标志性能力。

**核心能力**：
- 支持 **21 个平台**：Discord、Telegram、Slack、WhatsApp、Signal、飞书、iMessage、Matrix、MS Teams 等
- 操作类型：`send`（发消息）、`reply`（回复）、`react`（表情回应）、`edit`（编辑）、`delete`（删除）、`pin`/`unpin`
- 支持富文本、媒体附件、@提及
- 跨平台消息转发

**关键代码概要**：

```typescript
// message-tool.ts — 多平台消息分发
type MessageAction = "send" | "reply" | "react" | "edit" | "delete" | "pin" | "unpin";

async function executeMessage(params: {
  action: MessageAction;
  channel: string;        // "discord" | "telegram" | "slack" | ...
  target: string;         // 频道/群/用户 ID
  content?: string;
  media?: MediaAttachment;
}) {
  const plugin = resolveChannelPlugin(params.channel);
  return await plugin.dispatch(params);
}
```

**典型场景**：

```
用户（在 Telegram 中）："帮我在 Slack #general 频道发一条消息说今天下午3点开会"
Agent → message({
  action: "send",
  channel: "slack",
  target: "#general",
  content: "📢 今天下午 3 点开会，请大家准时参加"
})
```

---

### 6. sessions_spawn + subagents — 多 Agent 协作

**实现文件**：`src/agents/tools/sessions-spawn-tool.ts`（94 行）+ `src/agents/tools/subagents-tool.ts`（681 行）

**为什么入选**：实现 Agent-of-Agents 架构，主 Agent 可以生成多个子 Agent 并行工作，是 OpenClaw 处理复杂任务的核心能力。

**核心能力**：
- `sessions_spawn`：在隔离 session 中生成子 Agent（支持 run/session 模式）
- `subagents`：管理子 Agent 生命周期（`list`/`kill`/`steer`）
- Lane 并发控制：同会话串行、多会话并行
- Steer 机制：运行中途可注入新指令改变子 Agent 方向
- 完成公告：子 Agent 完成后自动通知父 Agent

**关键代码概要**：

```typescript
// subagents-tool.ts — steer 子代理
async function steerSubagent(params: { label: string; message: string }) {
  // 1. 标记旧运行，抑制完成公告
  markSubagentRunForSteerRestart(resolved.entry.runId);
  // 2. 中止旧运行
  abortEmbeddedPiRun(sessionId);
  // 3. 等旧运行结束（最多 5s）
  await callGateway({ method: "agent.wait", params: { runId, timeoutMs: 5_000 } });
  // 4. 分发新运行（同一 session，上下文延续）
  await callGateway({ method: "agent", params: { message, sessionKey } });
  // 5. 替换注册记录
  replaceSubagentRunAfterSteer({ previousRunId, nextRunId });
}
```

**典型场景**：

```
用户："帮我同时调研 React 和 Vue 的最新特性，然后做个对比"
Agent → sessions_spawn({ task: "调研 React 最新特性", label: "react-research" })
     → sessions_spawn({ task: "调研 Vue 最新特性", label: "vue-research" })
     → 两个子 Agent 并行工作
     → subagents({ action: "list" })  // 监控进度
     → 收到两个完成公告后，综合生成对比报告
```

---

### 7. cron — 定时任务

**实现文件**：`src/agents/tools/cron-tool.ts`（488 行）

**为什么入选**：让 Agent 从"被动响应"变成"主动执行"——可以定时检查、定时汇报、定时执行任务。社区评价：*"cron turns OpenClaw from a chatbot into a real assistant"*。

**核心能力**：
- 操作：`add`（创建）、`update`（更新）、`remove`（删除）、`list`（列表）、`run`（手动触发）、`runs`（历史记录）、`wake`（唤醒检查）
- 支持 cron 表达式和自然语言时间
- 仅 Owner 角色可用（安全限制）
- 定时任务的输出可自动发送到指定渠道

**关键代码概要**：

```typescript
// cron-tool.ts — 创建定时任务
async function addCron(params: {
  schedule: string;     // "0 9 * * *" (每天9点) 或 "every 30 minutes"
  prompt: string;       // Agent 要执行的指令
  channel?: string;     // 输出到哪个渠道
  enabled?: boolean;
}) {
  const normalized = parseCronExpression(params.schedule);
  return await gateway.cron.add({
    schedule: normalized,
    prompt: params.prompt,
    channel: params.channel,
  });
}
```

**典型场景**：

```
用户："每天早上 9 点帮我查看 GitHub 仓库的新 Issue，汇总发到 Slack"
Agent → cron({
  action: "add",
  schedule: "0 9 * * *",
  prompt: "检查 openclaw/openclaw 仓库的新 Issue，汇总发送到 Slack #dev-updates"
})
→ "已创建定时任务，每天 9:00 UTC 执行"
```

---

### 8. image — 图像理解

**实现文件**：`src/agents/tools/image-tool.ts`（561 行）

**为什么入选**：让 Agent 拥有"视觉"——能看懂截图、图表、照片，在多模态交互中不可或缺。与 browser 的 screenshot 配合使用时尤其强大。

**核心能力**：
- 支持本地文件路径和 URL
- 发送图片给视觉模型（GPT-4o / Claude / Gemini）进行描述/分析
- 支持自定义 prompt 引导分析方向
- 自动处理图片格式转换和大小限制

**关键代码概要**：

```typescript
// image-tool.ts — 图像理解
async function analyzeImage(params: {
  path?: string;       // 本地文件路径
  url?: string;        // 远程 URL
  prompt?: string;     // 分析指令，如 "描述这张图的内容"
}) {
  const imageData = params.path
    ? await readLocalImage(params.path)
    : await fetchRemoteImage(params.url);
  
  return await visionModel.analyze({
    image: imageData,
    prompt: params.prompt ?? "Describe this image in detail.",
  });
}
```

**典型场景**：

```
用户：[发送一张数据库 ER 图]
Agent → image({ path: "/tmp/er-diagram.png", prompt: "分析这个 ER 图，列出所有表和关系" })
     → "这是一个电商系统的 ER 图，包含 users、orders、products、categories 四张表..."
```

---

## 第二部分：Top 8 Skill（社区技能）

### 1. coding-agent — 编程助手

**技能文件**：`skills/coding-agent/SKILL.md`（285 行）

**为什么入选**：OpenClaw 作为编程 Agent 的核心技能，几乎是所有开发者的必装技能。ClawHub 上下载量最高的技能之一。

**核心能力**：
- 教会 Agent 完整的软件开发工作流：分析需求 → 编写代码 → 运行测试 → 修复 bug → 代码审查
- 定义了文件操作最佳实践（先读后改、小步迭代）
- 集成 Git 操作规范
- 错误恢复策略

**SKILL.md 关键片段**：

```markdown
## Workflow
1. Understand the task fully before writing code
2. Read existing files to understand context
3. Make changes in small, testable increments
4. Run tests after each change
5. If tests fail, read the error, fix, and re-run

## File Operations
- Always `read` a file before `edit`ing it
- Use `edit` for surgical changes, `write` for new files
- Never rewrite entire large files — use targeted edits
```

**使用方式**：

```bash
npx clawhub@latest install coding-agent
# 或在 OpenClaw 对话中：
# /skill install coding-agent
```

---

### 2. gh-issues — GitHub Issue 管理

**技能文件**：`skills/gh-issues/SKILL.md`（866 行）

**为什么入选**：866 行的超大型技能，覆盖 GitHub Issue 管理的方方面面。对于使用 GitHub 做项目管理的团队来说是"杀手级"技能。

**核心能力**：
- Issue CRUD：创建、读取、更新、关闭、标签管理
- 批量操作：批量打标签、批量关闭、批量分配
- 搜索与过滤：按标签、里程碑、负责人、日期范围筛选
- 评论管理：自动回复、模板评论
- 与 cron 配合实现自动化 Issue 巡检

**SKILL.md 关键片段**：

```markdown
## Commands
- `/gh issues list` — List open issues with filters
- `/gh issues create` — Create new issue with labels
- `/gh issues triage` — AI-assisted issue triage
- `/gh issues stale` — Find and handle stale issues
- `/gh issues stats` — Issue statistics and trends

## Triage Workflow
1. Fetch untagged issues
2. Analyze content with LLM
3. Suggest labels and priority
4. Assign to appropriate team member
```

**典型场景**：

```
用户："帮我检查 openclaw/openclaw 仓库里超过 30 天没有更新的 Issue"
Agent → 使用 gh-issues 技能
     → exec({ command: "gh issue list --repo openclaw/openclaw --state open --json number,title,updatedAt" })
     → 过滤 30 天未更新的
     → 生成报告
```

---

### 3. himalaya — 邮件管理

**技能文件**：`skills/himalaya/SKILL.md`（258 行）

**为什么入选**：通过 himalaya CLI 让 Agent 能收发邮件，是 OpenClaw 作为"个人助理"的关键能力。社区评价：*"finally, an AI that can actually handle my emails"*。

**核心能力**：
- 基于 himalaya CLI（Rust 编写的跨平台邮件客户端）
- 收件箱管理：列出、搜索、阅读、标记、归档
- 发送邮件：撰写、回复、转发，支持附件
- 多账户支持：同时管理多个邮箱
- 与 cron 配合实现定时邮件检查

**SKILL.md 关键片段**：

```markdown
## Prerequisites
- himalaya CLI installed (`cargo install himalaya` or brew)
- Email account configured in ~/.config/himalaya/config.toml

## Capabilities
- List/search/read/send/reply/forward emails
- Manage folders and labels
- Download and handle attachments
- Support IMAP/JMAP/Notmuch backends
```

**典型场景**：

```
用户："检查我的未读邮件，把重要的总结一下"
Agent → exec({ command: "himalaya list --folder INBOX --filter unseen" })
     → 逐封阅读并分析
     → "你有 5 封未读邮件，其中 2 封较重要：1) 老板关于项目截止日期的通知..."
```

---

### 4. notion — Notion 知识库集成

**技能文件**：`skills/notion/SKILL.md`（173 行）

**为什么入选**：Notion 是全球最流行的知识管理工具之一，这个技能让 Agent 能直接操作 Notion 页面和数据库，实现知识的自动化管理。

**核心能力**：
- 通过 Notion API 操作页面、数据库、块（Block）
- 创建/更新/查询 Notion 页面
- 数据库查询和过滤
- 支持富文本格式
- 需要配置 Notion Integration Token

**SKILL.md 关键片段**：

```markdown
## Setup
1. Create a Notion Integration at https://www.notion.so/my-integrations
2. Set NOTION_API_KEY environment variable
3. Share target pages/databases with the integration

## Operations
- Search pages and databases
- Create/update pages with rich content
- Query databases with filters and sorts
- Append blocks to existing pages
```

**典型场景**：

```
用户："把今天的会议纪要整理到 Notion 的'会议记录'数据库里"
Agent → 使用 notion 技能
     → 查询 Notion 数据库找到"会议记录"
     → 创建新页面，填入结构化的会议纪要
     → "已创建，页面链接：https://notion.so/..."
```

---

### 5. github — GitHub 综合操作

**技能文件**：`skills/github/SKILL.md`（164 行）

**为什么入选**：覆盖 GitHub 的核心操作（不仅仅是 Issue），包括 PR 管理、仓库操作、代码搜索等。与 coding-agent 和 gh-issues 形成完整的开发工作流闭环。

**核心能力**：
- 基于 `gh` CLI（GitHub 官方命令行工具）
- PR 管理：创建、审查、合并、评论
- 仓库操作：克隆、fork、star、release
- 代码搜索：跨仓库搜索代码
- GitHub Actions：查看/触发 CI/CD 工作流

**SKILL.md 关键片段**：

```markdown
## Prerequisites
- GitHub CLI (`gh`) installed and authenticated

## Capabilities
- Create and manage pull requests
- Review code changes
- Manage releases and tags
- Search code across repositories
- View and trigger GitHub Actions workflows
```

**典型场景**：

```
用户："帮我创建一个 PR，把 feature/auth 分支合并到 main"
Agent → exec({ command: "gh pr create --base main --head feature/auth --title 'Add auth module' --body '...'" })
     → "PR #42 已创建：https://github.com/..."
```

---

### 6. canvas — 可视化画布

**技能文件**：`skills/canvas/SKILL.md`（199 行）
**配套工具**：`src/agents/tools/canvas-tool.ts`（216 行）

**为什么入选**：OpenClaw 独有的可视化能力——Agent 可以创建和操控交互式画布（基于 HTML/React），在对话之外提供丰富的可视化展示。这是 OpenClaw 的差异化杀手功能。

**核心能力**：
- `present`：展示 HTML/React 组件到画布
- `navigate`：画布内页面导航
- `eval`：在画布中执行 JavaScript
- `snapshot`：获取画布当前状态截图
- A2UI（AI-to-UI）：根据自然语言描述生成 UI

**SKILL.md 关键片段**：

```markdown
## Canvas Capabilities
- Present interactive web content (HTML, React, charts, maps)
- A2UI: Describe a UI in natural language, agent builds it
- Take snapshots for iterative design
- Execute JavaScript in canvas context

## Use Cases
- Data visualization (charts, dashboards)
- Interactive prototypes
- Document preview with rich formatting
- Map and geographic visualization
```

**典型场景**：

```
用户："用图表展示这个 CSV 数据的销售趋势"
Agent → 读取 CSV → 生成 ECharts 代码
     → canvas({ action: "present", html: "<html>...<script>echarts...</script></html>" })
     → 画布上展示交互式折线图
```

---

### 7. skill-creator — 技能创建器

**技能文件**：`skills/skill-creator/SKILL.md`（373 行）

**为什么入选**：元技能（Meta-skill）——教 Agent 如何创建新的技能。是 OpenClaw 生态自我扩展的基石，ClawHub 上 2800+ 技能中很大一部分是通过这个技能辅助创建的。

**核心能力**：
- 引导用户定义技能的目标、前置条件、工作流
- 自动生成规范的 SKILL.md 文件
- 验证技能格式和安全性
- 支持发布到 ClawHub

**SKILL.md 关键片段**：

```markdown
## Skill Creation Workflow
1. **Gather Requirements**: Ask the user what the skill should do
2. **Define Frontmatter**: Set name, description, tags, env vars
3. **Write Instructions**: Clear step-by-step guide for the agent
4. **Add Tools Section**: Define any shell scripts or tool configurations
5. **Test**: Verify the skill works as intended
6. **Publish**: Push to ClawHub via `npx clawhub publish`

## SKILL.md Structure
- Frontmatter (YAML): metadata, tags, environment variables
- Body (Markdown): instructions, workflows, examples
- Tools directory (optional): shell scripts, configs
```

**典型场景**：

```
用户："帮我创建一个技能，能自动把 Jira 上的 ticket 同步到本地 Markdown 文件"
Agent → 使用 skill-creator 技能
     → 交互式收集需求
     → 生成 skills/jira-sync/SKILL.md
     → 添加 tools/sync.sh 脚本
     → "技能已创建，你可以用 /skill install jira-sync 安装"
```

---

### 8. healthcheck — 系统健康检查

**技能文件**：`skills/healthcheck/SKILL.md`（246 行）

**为什么入选**：OpenClaw 自我诊断的核心技能。当系统出现问题时，这个技能能全面检查配置、连接、权限等，帮用户快速定位问题。官方推荐的必装技能。

**核心能力**：
- 检查 LLM Provider 连接状态和 API Key 有效性
- 验证各渠道（Discord/Telegram/Slack 等）连接状态
- 检查文件系统权限和磁盘空间
- 验证技能安装完整性
- 检查 Gateway 配置一致性
- 生成结构化健康报告

**SKILL.md 关键片段**：

```markdown
## Health Check Areas
1. **Provider Connectivity**: Test each configured LLM provider
2. **Channel Status**: Verify messaging platform connections
3. **Tool Availability**: Check required CLI tools are installed
4. **Memory Backend**: Test memory search/write operations
5. **Disk & Permissions**: Verify file system access
6. **Configuration**: Validate gateway config consistency

## Output Format
✅ Provider (Anthropic): Connected, model claude-sonnet-4-20250514 available
✅ Channel (Discord): Bot online, 3 guilds connected
⚠️ Channel (Telegram): Webhook not set
❌ Tool (himalaya): Not installed
✅ Memory: LanceDB backend operational
✅ Disk: 45GB free on /
```

**典型场景**：

```
用户："OpenClaw 好像连不上 Telegram 了，帮我查查"
Agent → 使用 healthcheck 技能
     → 逐项检查
     → "发现问题：Telegram Webhook URL 已过期，需要重新设置。执行以下命令修复..."
```

---

## Tool 与 Skill 的协作关系

```
┌──────────────────────────────────────────────────┐
│                  Skill 层（教 Agent 做什么）        │
│                                                    │
│  coding-agent  gh-issues  himalaya  notion  ...    │
│      │             │          │        │           │
│      │  ┌──────────┼──────────┼────────┘           │
│      │  │          │          │                     │
└──────┼──┼──────────┼──────────┼─────────────────────┘
       │  │          │          │
       ▼  ▼          ▼          ▼
┌──────────────────────────────────────────────────┐
│                  Tool 层（Agent 的手和眼）          │
│                                                    │
│  exec  read/write/edit  web_search  browser  ...   │
│                                                    │
└──────────────────────────────────────────────────┘
```

**Skill 不创造新能力，而是教 Agent 如何组合已有 Tool 完成复杂任务。**

例如 `gh-issues` 技能本身不实现任何 GitHub API 调用，而是教 Agent：
1. 用 `exec` 工具调用 `gh` CLI
2. 用 `memory_search` 回忆项目的标签规范
3. 用 `message` 工具将结果发送到 Slack
4. 用 `cron` 工具设置定期检查

---

## 总结对比

### Top 8 Tools

| 排名 | Tool | 核心价值 | 代码量 | 入选理由 |
|------|------|----------|--------|----------|
| 1 | **web_search** | 联网搜索 | 1439 行 | Agent 获取实时信息的唯一通道 |
| 2 | **browser** | 浏览器自动化 | 830+7500 行 | OpenClaw 差异化的标志性能力 |
| 3 | **exec** | Shell 执行 | SDK 级 | Agent 与 OS 交互的唯一通道 |
| 4 | **memory** | 语义记忆 | 243 行 | Agent "灵魂"——长期记忆 |
| 5 | **message** | 多平台消息 | 698 行 | 21 平台主动消息，核心差异化 |
| 6 | **subagents** | 多 Agent 协作 | 681 行 | Agent-of-Agents 复杂任务处理 |
| 7 | **cron** | 定时任务 | 488 行 | 从被动响应到主动执行 |
| 8 | **image** | 图像理解 | 561 行 | 多模态交互不可或缺 |

### Top 8 Skills

| 排名 | Skill | 核心价值 | SKILL.md | 入选理由 |
|------|-------|----------|----------|----------|
| 1 | **coding-agent** | 编程工作流 | 285 行 | 开发者必装，使用量最高 |
| 2 | **gh-issues** | Issue 管理 | 866 行 | 功能最全面的 GitHub 技能 |
| 3 | **himalaya** | 邮件管理 | 258 行 | 个人助理杀手级功能 |
| 4 | **notion** | 知识库集成 | 173 行 | 连接最流行的知识管理工具 |
| 5 | **github** | GitHub 综合 | 164 行 | 完整的 GitHub 操作能力 |
| 6 | **canvas** | 可视化画布 | 199 行 | OpenClaw 独有的差异化功能 |
| 7 | **skill-creator** | 技能创建 | 373 行 | 生态自我扩展的基石 |
| 8 | **healthcheck** | 系统诊断 | 246 行 | 官方推荐的运维必备 |
