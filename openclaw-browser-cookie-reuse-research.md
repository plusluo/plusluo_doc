# OpenClaw 浏览器 Cookie/Session 复用机制调研

> **作者：plusluo**
> **日期：2026-02-26**

---

## 一、核心问题

如果要通过 OpenClaw 拉取一个需要登录的外部页面内容，而你在浏览器中已经登录过且状态有效，**是否可以不需要密码直接访问？**

**答案：完全可以。** 这是 OpenClaw 浏览器模块的核心设计之一。

---

## 二、浏览器架构

OpenClaw 使用 **CDP (Chrome DevTools Protocol) + Playwright** 混合架构操控 Chromium 浏览器：

```
OpenClaw Agent
    │
    │ 调用浏览器工具（navigate/click/snapshot...）
    ▼
Playwright (Node.js)
    │
    │ connectOverCDP()
    ▼
Chrome/Chromium 实例
    │
    │ --user-data-dir=~/.openclaw/browser/openclaw/user-data
    │ --remote-debugging-port=18800
    │ --disable-blink-features=AutomationControlled  ← 隐藏自动化标记
    ▼
持久化用户数据目录（Cookie/localStorage/sessionStorage 自动保存）
```

---

## 三、Cookie/Session 持久化原理

关键代码在 `src/browser/chrome.ts`：

```typescript
// 第 62-64 行
export function resolveOpenClawUserDataDir(profileName = DEFAULT_OPENCLAW_BROWSER_PROFILE_NAME) {
  return path.join(CONFIG_DIR, "browser", profileName, "user-data");
}
// 默认路径：~/.openclaw/browser/openclaw/user-data
```

Chrome 启动参数：

```typescript
const args: string[] = [
  `--remote-debugging-port=${profile.cdpPort}`,
  `--user-data-dir=${userDataDir}`,    // ← 持久化目录
  "--no-first-run",
  "--no-default-browser-check",
  "--disable-sync",
  "--disable-blink-features=AutomationControlled",  // ← 反检测
];
```

**为什么 Cookie 会自动保留：**
- `--user-data-dir` 指向持久化目录，不会在重启时清除
- 没有使用 `--incognito` 模式
- Chrome 原生的 Cookie 存储（SQLite 数据库）完全有效
- 项目中不使用 Playwright 的 `storageState` 序列化（搜索结果为 0）

---

## 四、三种复用登录状态的方式

### 方式 1：在 OpenClaw 管理的浏览器中登录（最简单）

1. 启动 OpenClaw 后，在它管理的浏览器窗口中**手动登录**目标网站
2. Cookie 和 session 自动保存到 `~/.openclaw/browser/openclaw/user-data`
3. 下次启动时，**登录状态自动保留**
4. Agent 使用浏览器工具访问该网站时，自动携带已有 Cookie

**优点**：最简单，无需额外配置
**缺点**：需要手动登录一次，且是在 OpenClaw 自己的 Chrome 实例中

### 方式 2：`attachOnly` 模式——连接你自己的 Chrome（推荐）

先手动启动你日常使用的 Chrome（带调试端口）：

```bash
# macOS
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=18800 \
  --user-data-dir=/Users/你的用户名/Library/Application\ Support/Google/Chrome

# 或者使用你已有的 Chrome 配置目录
```

然后配置 `openclaw.json`：

```json
{
  "browser": {
    "attachOnly": true,
    "profiles": {
      "openclaw": {
        "cdpUrl": "http://127.0.0.1:18800"
      }
    }
  }
}
```

OpenClaw 通过 CDP 连接到你的 Chrome，**直接使用你已有的所有登录状态**。

**优点**：复用你日常浏览器的全部登录状态，无需重新登录任何网站
**缺点**：需要手动以调试端口启动 Chrome

### 方式 3：Chrome Extension Relay——最无缝

配置 `openclaw.json`：

```json
{
  "browser": {
    "profiles": {
      "chrome": {
        "driver": "extension",
        "cdpUrl": "http://127.0.0.1:18792"
      }
    }
  }
}
```

安装 OpenClaw 浏览器扩展后，点击扩展图标即可将当前标签页暴露给 OpenClaw 控制。

**优点**：完全继承你日常 Chrome 的所有登录状态，最无缝
**缺点**：需要安装浏览器扩展

---

## 五、编程式 Cookie 操作 API

文件：`src/browser/pw-tools-core.storage.ts`

OpenClaw 还提供了编程式的 Cookie/Storage 操作能力：

| API | 功能 |
|-----|------|
| `cookiesGetViaPlaywright` | 获取当前页面的所有 cookies |
| `cookiesSetViaPlaywright` | 设置单个 cookie（支持 name/value/domain/path/expires/httpOnly/secure/sameSite） |
| `cookiesClearViaPlaywright` | 清除所有 cookies |
| `storageGetViaPlaywright` | 读取 localStorage/sessionStorage |
| `storageSetViaPlaywright` | 写入 localStorage/sessionStorage |
| `storageClearViaPlaywright` | 清除 localStorage/sessionStorage |

通过 HTTP 端点暴露（`src/browser/routes/agent.storage.ts`）：

| 端点 | 功能 |
|------|------|
| `GET /cookies` | 读取当前页面 cookies |
| `POST /cookies/set` | 设置 cookie |
| `POST /cookies/clear` | 清除 cookies |

### 举例：手动注入 Cookie

如果你从其他工具导出了 Cookie，可以通过 API 直接注入：

```typescript
// Agent 可以通过工具调用设置 cookie
cookiesSetViaPlaywright({
  name: "session_id",
  value: "abc123",
  domain: ".example.com",
  path: "/",
  httpOnly: true,
  secure: true
});
```

---

## 六、反自动化检测

Chrome 启动参数中包含：

```
--disable-blink-features=AutomationControlled
```

这会隐藏 `navigator.webdriver` 标记，降低被网站检测为自动化浏览器的风险。配合持久化的用户数据目录，使得 OpenClaw 的浏览器行为更接近真实用户。

---

## 七、实际场景举例

### 场景 1：拉取需要登录的 GitHub 私有仓库页面

```
步骤：
1. 使用方式 2，连接到你日常已登录 GitHub 的 Chrome
2. 告诉 Agent："帮我打开 github.com/my-org/private-repo 的 README 页面，总结内容"
3. Agent 调用 navigate 工具 → Chrome 携带你的 GitHub session cookie
4. 页面正常加载（无需登录）→ Agent 截图/提取内容 → 返回总结
```

### 场景 2：拉取公司内部 Wiki 内容

```
步骤：
1. 在 OpenClaw 管理的浏览器中手动登录公司 SSO（方式 1）
2. Cookie 自动保存
3. 之后随时让 Agent 访问内部 Wiki 页面，登录状态持续有效
4. 下次重启 OpenClaw 后，Cookie 仍然保留（除非服务端 session 过期）
```

### 场景 3：通过 Extension Relay 操作已打开的页面

```
步骤：
1. 你在 Chrome 中打开了某个需要登录的后台管理页面
2. 点击 OpenClaw 浏览器扩展图标
3. 当前标签页暴露给 OpenClaw 控制
4. 告诉 Agent："帮我在这个页面上导出最近 30 天的数据"
5. Agent 直接在你已登录的页面上操作，无需任何认证
```

---

## 八、关键代码位置

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/browser/chrome.ts` | 351 | Chrome 启动（user-data-dir、CDP 端口、反检测参数） |
| `src/browser/pw-session.ts` | 816 | Playwright 会话管理（connectOverCDP） |
| `src/browser/config.ts` | 311 | 浏览器配置解析（attachOnly、profiles） |
| `src/browser/profiles.ts` | 114 | 浏览器 Profile 管理 |
| `src/browser/server-context.ts` | 692 | 浏览器上下文生命周期 |
| `src/browser/pw-tools-core.storage.ts` | 129 | Cookie/Storage 操作 API |
| `src/browser/extension-relay.ts` | — | Chrome 扩展中继模式 |
| `src/browser/routes/agent.storage.ts` | — | Cookie HTTP 端点 |
