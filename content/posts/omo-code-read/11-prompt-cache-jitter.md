---
title: "Prompt 前缀抖动 — 工具注册与动态 Hook 注入对缓存的影响"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# Prompt 前缀抖动 — 工具注册与动态 Hook 注入对缓存的影响

> 源码路径：基于 `/mnt/e/code/cc/opencode/` 和 `/mnt/e/code/cc/omo-code/`
> 分析依据：`llm.ts`, `prompt.ts`, `registry.ts`, `plugin-interface.ts`, `messages-transform.ts`, `context-injector/`

---

## 1. 问题描述

有观点认为：**OpenCode 的工具注册和 OMO 的动态 Hook 注入会导致 Prompt 前缀频繁抖动，从而引发缓存失效。**

这个说法在方向上正确，但需要严格区分哪些因素真正影响了 LLM 的缓存前缀。以下从源码层面逐层分析。

---

## 2. Prompt 前缀的构成

### 2.1 System Prompt 层

在 OpenCode 的 `llm.ts` 中，Prompt 前缀按以下顺序构建：

```typescript
// packages/opencode/src/session/llm.ts (lines 102-114)
const system: string[] = []
system.push([
  // 第 1 层: Agent Prompt 或 Provider Prompt
  ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),

  // 第 2 层: 自定义 System Prompt
  ...input.system,

  // 第 3 层: 用户 System Prompt
  ...(input.user.system ? [input.user.system] : []),
].filter(x => x).join("\n"))
```

接着触发插件钩子：

```typescript
// packages/opencode/src/session/llm.ts (lines 117-120)
await Plugin.trigger(
  "experimental.chat.system.transform",
  { sessionID: input.sessionID, model: input.model },
  { system },  // 插件可修改 system 数组
)
```

### 2.2 Tool Definitions 层

工具定义通过 `resolveTools()` 解析并传入 LLM API：

```typescript
// packages/opencode/src/session/prompt.ts (lines 388-551)
const tools = yield* resolveTools({
  agent,
  session,
  model,
  tools: lastUser.tools,
  processor: handle,
  bypassAgentCheck,
  messages: msgs,
})
```

每个工具包含：
- `description` — 工具描述文本
- `inputSchema` — JSON Schema 格式的参数定义

这两部分序列化后会成为 API 请求中的 `tools` 参数，直接影响缓存前缀。

---

## 3. 工具集的动态性分析

### 3.1 OpenCode 基础层的工具过滤

```typescript
// packages/opencode/src/tool/registry.ts (lines 196-208)
const filtered = allTools.filter((tool) => {
  // codesearch / websearch — 仅 OpenCode provider 或 EXA 开启时
  if (tool.id === "codesearch" || tool.id === "websearch") {
    return model.providerID === ProviderID.opencode || Flag.OPENCODE_ENABLE_EXA
  }

  // apply_patch — GPT 系列专属
  const usePatch =
    !!Env.get("OPENCODE_E2E_LLM_URL") ||
    (model.modelID.includes("gpt-") && !model.modelID.includes("oss") && !model.modelID.includes("gpt-4"))
  if (tool.id === "apply_patch") return usePatch
  if (tool.id === "edit" || tool.id === "write") return !usePatch

  return true
})
```

这意味着即使用**同一个 Session**，工具集也可能随模型变化：

| 模型 | apply_patch | edit/write |
|------|------------|------------|
| gpt-4o | ❌ | ✅ |
| gpt-4 | ❌ | ✅ |
| o1 | ✅ | ❌ |
| claude-sonnet-4-6 | ❌ | ✅ |

### 3.2 条件启用的工具

```typescript
// packages/opencode/src/tool/registry.ts (lines 177-179)
...(Flag.OPENCODE_EXPERIMENTAL_LSP_TOOL ? [lsp] : []),       // 特性开关
...(cfg.experimental?.batch_tool === true ? [batch] : []),   // 配置控制
...(Flag.OPENCODE_EXPERIMENTAL_PLAN_MODE && Flag.OPENCODE_CLIENT === "cli" ? [plan] : []),
```

### 3.3 OMO 的工具扩展

```typescript
// packages/opencode/src/tool/registry.ts (lines 127-132)
const plugins = yield* plugin.list()
for (const p of plugins) {
  for (const [id, def] of Object.entries(p.tool ?? {})) {
    custom.push(fromPlugin(id, def))  // OMO 注册 26+ 工具
  }
}
```

OMO 额外注册了 26+ 工具。这些工具被合并到 `custom` 数组末尾，但在过滤时同样受到 Agent 权限的约束：

```typescript
// packages/opencode/src/session/llm.ts (lines 339-344)
function resolveTools(input) {
  const disabled = Permission.disabled(
    Object.keys(input.tools),
    Permission.merge(input.agent.permission, input.permission ?? []),
  )
  return Record.filter(input.tools, (_, k) =>
    input.user.tools?.[k] !== false && !disabled.has(k)
  )
}
```

---

## 4. Hook 注入的动态性分析

### 4.1 OMO 的消息转换钩子

```typescript
// omo-code/src/plugin/messages-transform.ts
export function createMessagesTransformHandler(args: { hooks: CreatedHooks }) {
  return async (input, output): Promise<void> => {
    await args.hooks.contextInjectorMessagesTransform?.[
      "experimental.chat.messages.transform"
    ]?.(input, output)

    await args.hooks.thinkingBlockValidator?.[
      "experimental.chat.messages.transform"
    ]?.(input, output)

    await args.hooks.toolPairValidator?.[
      "experimental.chat.messages.transform"
    ]?.(input, output)
  }
}
```

注意：这是 `experimental.chat.messages.transform`，影响的是**消息数组**而非 System Prompt 本身。但 `experimental.chat.system.transform` 则直接影响 System Prompt：

```typescript
// omo-code/src/plugin/system-transform.ts
export function createSystemTransformHandler() {
  return async (input, output): Promise<void> => {
    // 目前为空，OMO 未直接修改 system prompt
  }
}
```

### 4.2 上下文注入 — 按需而非每请求

```typescript
// omo-code/src/features/context-injector/injector.ts
export function injectPendingContext(
  collector: ContextCollector,
  sessionID: string,
  parts: OutputPart[]
): InjectionResult {
  if (!collector.hasPending(sessionID)) {
    return { injected: false, contextLength: 0 }
  }

  // 仅当有 pending context 时才注入
  const pending = collector.consume(sessionID)
  parts[textPartIndex].text = `${pending.merged}\n\n---\n\n${originalText}`
  return { injected: true, contextLength: pending.merged.length }
}
```

关键发现：**上下文注入是消费式的**（`consume`）。每次注入后清空，不会反复注入相同内容。

### 4.3 First Message 变体

```typescript
// omo-code/src/plugin-interface.ts (lines 50-55)
"chat.message": createChatMessageHandler({
  firstMessageVariantGate,
  hooks,
  // ...
}),
```

首次消息会触发特殊的 Agent 变体选择逻辑，这是**一次性事件**，不影响后续轮次。

---

## 5. 缓存前缀抖动的真实来源

基于源码分析，以下是真正导致 Prompt 前缀变化的因素：

### 5.1 稳定因素（同一会话内不变化）

| 因素 | 稳定性 |
|------|--------|
| Provider 专属 System Prompt | Session 固定 |
| Base Agent System Prompt | Session 固定 |
| 内置工具集（bash/read/write...） | Session 固定 |
| OMO 注册的 26+ 工具定义 | Session 固定 |
| Agent Permission Ruleset | Session 固定 |

### 5.2 潜在抖动因素

| 因素 | 触发条件 | 抖动频率 |
|------|----------|----------|
| **工具集变化** | 切换模型（gpt → claude） | 低频 |
| **工具过滤变化** | Feature flag 切换 | 低频 |
| **会话压缩** | Token 超限触发 compaction | 低频 |
| **上下文注入** | Hook 显式注册 pending context | 按需 |
| **首次消息变体** | Session 创建时 | 一次性 |

---

## 6. 量化分析

### 6.1 工具定义大小估算

以 Claude API 的 `tools` 参数为例：

```typescript
// 单个工具的 schema 约 200-500 字节
{
  name: "bash",
  description: "Execute a shell command...",
  input_schema: {
    type: "object",
    properties: {
      command: { type: "string" },
      timeout: { type: "number", optional: true },
      ...
    }
  }
}
```

假设平均每个工具 ~400 字节，30 个工具 ≈ **12KB** 的工具定义序列化。

### 6.2 OMO 的额外开销

| 工具集 | 工具数 | 估计大小 |
|--------|--------|----------|
| OpenCode 基础 | ~24 个 | ~9.6KB |
| OMO 扩展 | +26 个 | +10.4KB |
| 合计 | ~50 个 | ~20KB |

这 20KB 的差异是**固定的**（在 Session 生命周期内），不会每轮抖动。

### 6.3 Hook 注入的实际频率

```typescript
// 上下文注入 — 有 pending 才触发
if (!collector.hasPending(sessionID)) {
  return { injected: false }
}

// 消费式 — 每次消费后清空
const pending = collector.consume(sessionID)
```

Hook 注入的内容大小取决于 `pending.merged`，通常为 **几百字节到几KB**，且仅在特定事件（如 Tool 执行前、Tool 执行后、Session 创建时）触发。

---

## 7. 结论

### 7.1 这个说法是否正确？

**部分正确，但严重程度被高估。**

OMO 的工具注册和 Hook 注入确实**增加了 Prompt 前缀的复杂度**，但：

1. **工具集的差异是 Session 级别的**，不是每轮请求都变化。只要 Agent 和模型不变，工具集就是稳定的。

2. **Hook 注入是消费式的**，不是每次请求都追加。相同的上下文不会反复注入。

3. **真正的抖动来自模型切换**（Claude ↔ GPT），这在 OpenCode 基础层就存在，OMO 并未引入新的抖动机制。

### 7.2 OMO 真正增加的缓存压力

| 因素 | 影响 | 严重程度 |
|------|------|----------|
| 工具集变大（+26 工具） | 每个请求的缓存前缀更大，但**稳定** | ⚠️ 中 |
| Hook 注入按需内容 | 上下文变化时才变更前缀 | ✅ 低 |
| First Message 变体 | 一次性，影响 Session 启动 | ✅ 低 |
| 模型回退（runtime-fallback） | 触发时重新构建工具集 | ⚠️ 低频 |

### 7.3 与 Claude Code 的对比

Claude Code 使用文件系统转录本，前缀相对稳定（系统提示词 + 工具定义），但 Claude Code 同样面临工具集动态过滤的问题（不同 PermissionMode 下工具集不同）。

**两者在缓存抖动问题上的严重程度相近**，OMO 由于工具更多、体积更大，缓存未命中时的成本略高，但抖动频率并不比 OpenCode 基础层更高。

### 7.4 缓解建议

如果担忧缓存效率，可以考虑：

1. **固定 Agent 和模型**：避免在同一会话内切换，降低工具集变化频率
2. **限制 OMO 工具注册**：通过 `disabled_tools` 配置禁用不需要的 OMO 工具
3. **减少 Hook 注入频率**：合并多次小注入为一次性大注入
4. **监控缓存命中率**：通过 API 响应中的 `cache Creation` 字段观察命中率

---

*文档版本：v1.0 | 更新：2026-04-06*
