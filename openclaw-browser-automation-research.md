# OpenClaw Browser Automation 深度解析

> plusluo 基于 OpenClaw 源码（版本 2026.2.25）分析，所有描述均以实际代码为依据。

---

## 一、模块定位

Browser Automation 是 OpenClaw 的浏览器自动化子系统，让 AI Agent 能像人一样操控真实的 Chrome 浏览器——导航网页、点击元素、输入文本、截图、提取内容等。它不是简单的网页爬虫，而是一个完整的**远程浏览器 RPA 引擎**，是 OpenClaw "Agent 不只是聊天"理念的核心技术支撑。

**典型使用场景**：
- 用户说"帮我查一下明天北京的天气"→ Agent 打开天气网站 → 截图/提取内容 → 回复用户
- 用户说"帮我在某网站上填写表单"→ Agent 打开网页 → 定位输入框 → 填入数据 → 点击提交
- 用户说"帮我把这个网页整理成摘要"→ Agent 导航 URL → 获取页面快照 → LLM 生成摘要

---

## 二、整体架构（C/S 模式）

整个模块采用**客户端-服务端（Client/Server）分离架构**：

```
Agent Tool (client) ←── HTTP API ──→ Browser Server ←── Playwright/CDP ──→ Chrome 浏览器
```

### 三层结构

| 层级 | 代码位置 | 职责 |
|------|----------|------|
| **Agent Tool** | `src/agents/tools/browser-tool.ts`（830 行） | 注册为 Agent 可调用的工具，解析 Agent 指令并转发给 Client |
| **Browser Client** | `src/browser/client.ts` + `client-actions*.ts` | 封装 HTTP 调用，提供操作 API 给 Agent Tool 使用 |
| **Browser Server** | `src/browser/server.ts` + `server-context.ts`（692 行） | 管理浏览器实例生命周期，提供 HTTP API，执行实际的浏览器操作 |

**设计优势**：C/S 分离使得浏览器实例可以独立于 Agent 进程运行，支持多 Agent 共享同一浏览器实例，也为远程浏览器场景（如云端 Chrome）提供了天然支持。

---

## 三、Agent 可用操作一览

> **代码位置**：`src/agents/tools/browser-tool.schema.ts`（113 行）、`src/agents/tools/browser-tool.ts`（830 行）

Agent 调用 `browser` 工具时，通过 `action` 字段选择具体操作：

| 操作 | 作用 | 说明 |
|------|------|------|
| `navigate` | 导航到指定 URL | 支持任意 HTTP/HTTPS 地址 |
| `click` | 点击页面元素 | 通过坐标或选择器定位 |
| `type` / `fill` | 在输入框中键入文本 | `type` 模拟逐键输入，`fill` 直接填充 |
| `screenshot` | 截取当前页面截图 | 返回图片给 Agent 作为视觉输入 |
| `snapshot` | 获取页面无障碍树快照 | 结构化 DOM 描述，Agent 可精确理解页面布局 |
| `scroll_down` / `scroll_up` | 滚动页面 | 用于查看长页面内容 |
| `go_back` / `go_forward` | 前进 / 后退 | 浏览器历史导航 |
| `wait` | 等待页面加载或元素出现 | 确保异步内容就绪 |
| `evaluate` | 执行任意 JavaScript 代码 | 最灵活的操作方式 |
| `download` | 下载文件 | 将文件下载到本地 |
| `observe` | 观察页面状态 | 监测页面变化 |
| `get_storage` / `set_storage` | 读写 localStorage / Cookie | 操作浏览器存储 |

---

## 四、两种浏览器驱动模式

### 4.1 Playwright 模式（默认）

> **代码位置**：`src/browser/pw-session.ts`（816 行）

- 使用 `playwright-core` 库启动和控制浏览器
- 功能最全，支持所有操作类型
- 自动管理浏览器进程的启动与销毁

**核心依赖**（`package.json`）：

```json
"playwright-core": "^1.52.0"
```

### 4.2 CDP 直连模式

> **代码位置**：`src/browser/cdp.ts`（463 行）

- 通过 **Chrome DevTools Protocol** 直接连接已运行的 Chrome 实例
- 适合连接远程浏览器或已有的 Chrome 进程
- 不需要 Playwright 额外启动浏览器

---

## 五、关键技术组件详解

### 5.1 Chrome 可执行文件查找

> **代码位置**：`src/browser/chrome.executables.ts`（626 行）

跨平台自动查找本地安装的 Chrome 浏览器：

| 平台 | 搜索范围 |
|------|----------|
| **macOS** | Chrome、Chrome Canary、Chromium、Arc、Brave、Edge 等 |
| **Linux** | `/usr/bin/google-chrome`、chromium-browser、snap 安装路径等 |
| **Windows** | `Program Files` 下的 Chrome 路径 |

如果本地未找到 Chrome，会尝试自动下载 Chromium。

### 5.2 无障碍树快照（Accessibility Snapshot）

> **代码位置**：`src/browser/pw-role-snapshot.ts`（455 行）

这是 Browser Automation 最核心的技术之一。Agent 不需要像人一样"看"截图的像素，而是获取页面的**结构化无障碍树（Accessibility Tree）**：

```
// 无障碍树示例：
// [button "登录"] [textbox "用户名" value=""] [link "忘记密码"]
// Agent 可以精确知道页面上有哪些可交互元素及其文本内容
```

**优势**：
- 比截图识别更精确、更可靠
- Token 消耗远低于传输图片
- 能识别隐藏元素、嵌套结构等

### 5.3 浏览器扩展中继（Extension Relay）

> **代码位置**：`src/browser/extension-relay.ts`（824 行，模块最大文件）

通过注入浏览器扩展获取更多控制能力：
- 监听网络请求
- 操作 Cookie
- 拦截和修改页面行为
- 获取扩展层面才能访问的浏览器 API

### 5.4 导航守卫（Navigation Guard）

> **代码位置**：`src/browser/navigation-guard.ts`（93 行）

安全机制，防止 Agent 导航到危险 URL：
- 过滤恶意地址
- 限制协议类型
- 保护用户安全

### 5.5 浏览器配置文件（Profiles）

> **代码位置**：`src/browser/profiles.ts`（114 行）

支持持久化浏览器配置文件：
- Agent 可以**复用登录状态**（已登录的网站无需重新登录）
- 保存 Cookie、localStorage 等会话数据
- 多个 Agent 可以使用不同配置文件

### 5.6 AI 辅助交互

> **代码位置**：`src/browser/pw-ai.ts`（66 行）、`pw-ai-module.ts`（52 行）

结合 LLM 理解页面语义，让 Agent 可以用**自然语言描述**要操作的元素，而不必精确定位 CSS 选择器。例如 Agent 可以说"点击登录按钮"，AI 模块会自动定位到 `<button>登录</button>` 元素。

### 5.7 Playwright 交互层

> **代码位置**：`src/browser/pw-tools-core.interactions.ts`（647 行）

封装了所有底层的 Playwright 页面交互操作：
- 元素点击（支持坐标和选择器）
- 文本输入和表单填充
- 键盘和鼠标事件模拟
- 元素等待和状态检测

### 5.8 下载管理

> **代码位置**：`src/browser/pw-tools-core.downloads.ts`（281 行）

处理文件下载场景：
- 拦截下载事件
- 保存文件到指定路径
- 支持下载进度追踪

### 5.9 存储操作

> **代码位置**：`src/browser/pw-tools-core.storage.ts`（129 行）

读写浏览器存储：
- localStorage 读写
- Cookie 管理
- sessionStorage 操作

---

## 六、Server Context 生命周期管理

> **代码位置**：`src/browser/server-context.ts`（692 行）、`server-context.types.ts`（66 行）、`server-lifecycle.ts`（49 行）

Browser Server 管理浏览器实例的完整生命周期：

```
创建 Context → 启动浏览器 → 建立页面会话 → 执行操作 → 关闭页面 → 销毁 Context
```

关键能力：
- **多会话管理**：同一个浏览器实例可以管理多个页面/标签
- **资源回收**：自动检测空闲会话并关闭，释放内存
- **错误恢复**：浏览器崩溃后自动重启

---

## 七、Browser 配置系统

> **代码位置**：`src/browser/config.ts`（311 行）、`constants.ts`（9 行）

可配置的参数包括：
- 浏览器可执行文件路径
- 启动参数（headless、窗口大小等）
- 代理设置
- 超时时间
- 是否使用持久化 Profile
- 服务端口和地址

```
// 文件：src/browser/constants.ts
// 定义了默认端口、超时等常量
```

---

## 八、与 Agent 的集成方式

> **代码位置**：`src/agents/tools/browser-tool.ts`（830 行）、`src/agents/sandbox/browser.ts`（401 行）

### 8.1 工具注册

Browser 作为 Agent 的内置工具之一，在工具目录（`src/agents/tool-catalog.ts`）中注册。Agent 在需要时自动识别并调用。

### 8.2 沙箱模式

> **代码位置**：`src/agents/sandbox/browser.ts`（401 行）

在安全沙箱环境中运行浏览器，限制浏览器的权限范围，防止：
- 访问本地文件系统
- 执行系统命令
- 访问受限网络资源

### 8.3 Bridge 机制

> **代码位置**：`src/browser/bridge-server.ts`（111 行）、`src/agents/sandbox/browser-bridges.ts`（12 行）

Bridge 是浏览器沙箱与宿主环境之间的通信桥梁，允许在受限环境中安全地传递数据。

---

## 九、完整源文件索引

```
src/browser/
├── config.ts                        # 浏览器配置定义（311 行）
├── constants.ts                     # 常量定义（9 行）
├── chrome.ts                        # Chrome 浏览器管理（351 行）
├── chrome.executables.ts            # 跨平台 Chrome 查找（626 行）
├── cdp.ts                           # CDP 直连模式（463 行）
├── client.ts                        # Browser Client（338 行）
├── client-actions.ts                # Client 操作入口（5 行）
├── client-actions-core.ts           # Client 核心操作（260 行）
├── client-actions-observe.ts        # Client 观察操作（185 行）
├── client-actions-state.ts          # Client 状态操作（285 行）
├── server.ts                        # Browser Server（111 行）
├── server-context.ts                # Server 会话管理（692 行）
├── server-context.types.ts          # Server 类型定义（66 行）
├── server-lifecycle.ts              # Server 生命周期（49 行）
├── server-middleware.ts             # Server 中间件（38 行）
├── bridge-server.ts                 # Bridge 通信服务（111 行）
├── extension-relay.ts               # 浏览器扩展中继（824 行）
├── navigation-guard.ts              # 导航守卫（93 行）
├── profiles.ts                      # 浏览器配置文件（114 行）
├── screenshot.ts                    # 截图功能（59 行）
├── pw-session.ts                    # Playwright 会话（816 行）
├── pw-ai.ts                         # AI 辅助交互（66 行）
├── pw-ai-module.ts                  # AI 模块加载（52 行）
├── pw-ai-state.ts                   # AI 状态管理（10 行）
├── pw-role-snapshot.ts              # 无障碍树快照（455 行）
├── pw-tools-core.ts                 # Playwright 工具核心入口（9 行）
├── pw-tools-core.interactions.ts    # 交互操作（647 行）
├── pw-tools-core.snapshot.ts        # 快照操作（222 行）
├── pw-tools-core.state.ts           # 状态操作（210 行）
├── pw-tools-core.downloads.ts       # 下载管理（281 行）
├── pw-tools-core.storage.ts         # 存储操作（129 行）
├── pw-tools-core.trace.ts           # 追踪调试（38 行）
├── pw-tools-core.responses.ts       # 响应处理（124 行）
└── pw-tools-core.activity.ts        # 活动检测（69 行）

src/agents/tools/
├── browser-tool.ts                  # Agent 浏览器工具（830 行）
└── browser-tool.schema.ts           # 工具参数 Schema（113 行）

src/agents/sandbox/
├── browser.ts                       # 浏览器沙箱（401 行）
└── browser-bridges.ts               # 沙箱通信桥（12 行）
```

**总代码量**：约 7,500+ 行 TypeScript，是 OpenClaw 中规模较大的独立模块之一。

---

## 十、总结

OpenClaw 的 Browser Automation 模块是一个**工程化程度很高的浏览器自动化系统**，其核心价值在于：

1. **C/S 分离架构** — 浏览器实例独立运行，支持共享和远程场景
2. **双驱动模式** — Playwright（全功能）+ CDP（直连），适配不同场景
3. **无障碍树优先** — 相比截图识别，结构化数据更精确、更高效
4. **AI 辅助定位** — 自然语言描述元素，降低操作复杂度
5. **安全纵深** — 导航守卫 + 沙箱隔离 + Bridge 通信，多层防护
6. **持久化 Profile** — 复用登录状态，减少重复认证

它让 AI Agent 从"只能聊天"进化到"能操作互联网"，是 OpenClaw "Agent 不只是聊天"理念的核心技术支撑。
