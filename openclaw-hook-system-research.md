# OpenClaw Hook System（事件钩子系统）调研

> **作者：plusluo**
> **日期：2026-02-26**

---

## 一、概述

Hook System 是 OpenClaw 的**事件驱动拦截/观察机制**，在 Agent 生命周期的关键节点插入自定义逻辑。它不是 hook 浏览器，而是 hook OpenClaw 自身的运行流程。

系统由两层组成：
- **插件 Hook 系统**（`src/plugins/hooks.ts` + `src/plugins/types.ts`）：成熟的类型安全生命周期 Hook，支持 24 个具名事件
- **内部 Hook 系统**（`src/hooks/`）：基于 `type:action` 事件键的早期系统

---

## 二、全部 24 个生命周期事件

### 2.1 Agent 运行类（6 个）

| Hook 名称 | 类型 | 能力 |
|-----------|------|------|
| `before_model_resolve` | 修改型（sequential） | 可覆盖 model/provider 选择 |
| `before_prompt_build` | 修改型（sequential） | 可注入 systemPrompt / prependContext |
| `before_agent_start` | 修改型（sequential） | Agent 启动前拦截，合并 model+prompt 覆盖 |
| `llm_input` | 观察型（parallel） | 观察发送给 LLM 的完整输入 |
| `llm_output` | 观察型（parallel） | 观察 LLM 返回的输出和 token 用量 |
| `agent_end` | 观察型（parallel） | Agent 运行结束后分析已完成的对话 |

### 2.2 上下文压缩类（3 个）

| Hook 名称 | 类型 | 能力 |
|-----------|------|------|
| `before_compaction` | 观察型（parallel） | 压缩前通知 |
| `after_compaction` | 观察型（parallel） | 压缩后通知 |
| `before_reset` | 观察型（parallel） | `/new` 或 `/reset` 清空会话前触发 |

### 2.3 消息收发类（4 个）

| Hook 名称 | 类型 | 能力 |
|-----------|------|------|
| `message_received` | 观察型（parallel） | 收到消息通知 |
| `message_sending` | 修改型（sequential） | **可修改或取消**待发消息（content/cancel） |
| `message_sent` | 观察型（parallel） | 消息发送后通知 |
| `before_message_write` | 修改型（sync） | 可阻止/修改消息写入 JSONL 会话记录 |

### 2.4 工具调用类（3 个）

| Hook 名称 | 类型 | 能力 |
|-----------|------|------|
| `before_tool_call` | 修改型（sequential） | **可修改参数或阻止**工具调用（params/block/blockReason） |
| `after_tool_call` | 观察型（parallel） | 工具调用完成后通知（含结果和耗时） |
| `tool_result_persist` | 修改型（sync） | 可修改写入会话记录的工具结果消息 |

### 2.5 子代理类（4 个）

| Hook 名称 | 类型 | 能力 |
|-----------|------|------|
| `subagent_spawning` | 修改型（sequential） | 子代理生成前，可返回 ok/error/threadBindingReady |
| `subagent_delivery_target` | 修改型（sequential） | 可覆盖子代理消息投递目标 |
| `subagent_spawned` | 观察型（parallel） | 子代理已生成通知 |
| `subagent_ended` | 观察型（parallel） | 子代理结束通知（含 outcome） |

### 2.6 会话类（2 个）

| Hook 名称 | 类型 | 能力 |
|-----------|------|------|
| `session_start` | 观察型（parallel） | 会话启动通知 |
| `session_end` | 观察型（parallel） | 会话结束通知 |

### 2.7 网关类（2 个）

| Hook 名称 | 类型 | 能力 |
|-----------|------|------|
| `gateway_start` | 观察型（parallel） | 网关启动通知 |
| `gateway_stop` | 观察型（parallel） | 网关停止通知 |

---

## 三、两种 Hook 类型的区别

| 特性 | 修改型（sequential） | 观察型（parallel） |
|------|---------------------|-------------------|
| 执行顺序 | 串行，按优先级依次执行 | 并行，所有 handler 同时触发 |
| 能否修改数据 | 可以修改、阻止、覆盖 | 只能观察，不能修改 |
| 返回值 | 返回修改后的数据，传递给下一个 handler | 无返回值要求 |
| 典型场景 | 拦截工具调用、修改消息内容 | 日志记录、指标收集 |

---

## 四、Hook 注册方式

```typescript
// 方式 1：插件 API 的类型安全注册（推荐）
api.on("before_tool_call", async (event) => {
  if (event.toolName === "bash") {
    return { block: true, blockReason: "bash is disabled" };
  }
}, { priority: 10 });

// 方式 2：遗留的内部 Hook 注册
api.registerHook("command:new", async (payload) => {
  // 处理 /new 命令
});
```

---

## 五、4 个内置 Hook

| Hook 名称 | 监听事件 | 作用 |
|-----------|---------|------|
| `session-memory` | `command:new`, `command:reset` | 会话重置时自动保存上下文到 memory |
| `boot-md` | `gateway:startup` | 网关启动时执行 BOOT.md 引导脚本 |
| `command-logger` | `command` | 所有命令写入审计日志 |
| `bootstrap-extra-files` | `agent:bootstrap` | 注入额外的 workspace 引导文件 |

---

## 六、Hook 与浏览器的关系

Hook 系统与浏览器是**间接关系**。当 Agent 调用浏览器工具（如 `navigate`、`click`、`snapshot`）时：

- `before_tool_call` 会被触发，`event.toolName` 包含浏览器工具名称
- 可以在此拦截/修改浏览器工具调用的参数，或通过 `block` 阻止调用
- `after_tool_call` 会在浏览器操作完成后触发，包含执行结果和耗时

Hook 本身不直接启动或控制浏览器。

### 举例

```typescript
// 拦截所有浏览器 navigate 操作，禁止访问内网地址
api.on("before_tool_call", async (event) => {
  if (event.toolName === "navigate") {
    const url = event.params.url;
    if (url.includes("192.168.") || url.includes("10.0.")) {
      return { block: true, blockReason: "禁止访问内网地址" };
    }
  }
});

// 记录所有浏览器操作的耗时
api.on("after_tool_call", async (event) => {
  if (["navigate", "click", "snapshot"].includes(event.toolName)) {
    console.log(`浏览器操作 ${event.toolName} 耗时 ${event.durationMs}ms`);
  }
});
```

---

## 七、关键代码位置

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/plugins/types.ts` | 764 | 全部 24 个 Hook 事件定义和签名 |
| `src/plugins/hooks.ts` | 754 | Hook Runner 实现（执行调度逻辑） |
| `src/hooks/internal-hooks.ts` | 285 | 内部 Hook 系统（注册/触发） |
| `src/hooks/loader.ts` | 193 | Hook 加载器（从目录/配置加载） |
| `src/hooks/types.ts` | 68 | Hook 元数据类型 |
| `src/gateway/hooks.ts` | 391 | 网关 Webhook Hook（外部 HTTP 触发） |
| `src/gateway/hooks-mapping.ts` | 527 | Webhook 映射和模板渲染 |
