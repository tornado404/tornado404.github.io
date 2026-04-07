---
title: "会话与上下文 — 消息流、压缩与记忆"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 会话与上下文 — 消息流、压缩与记忆

> 源码路径：`/mnt/e/code/cc/opencode/packages/opencode/src/session/`
> 核心文件：`prompt.ts`, `processor.ts`, `llm.ts`, `compaction.ts`, `system.ts`
> 技术栈：Effect + AI SDK + SQLite + Drizzle ORM

---

## 1. 概述

OpenCode 的会话系统承担着消息管理、LLM 调用、事件处理和上下文压缩等核心职责。相比 Claude Code 的文件系统转录本，OpenCode 使用 **SQLite + Drizzle ORM** 持久化所有会话数据，并通过 **Compaction（压缩）** 机制管理上下文长度。

---

## 2. 会话数据模型

### 2.1 Session.Info

```typescript
Session.Info = {
  id: SessionID,           // 唯一标识
  slug: string,            // URL 友好 slug
  projectID: ProjectID,     // 所属项目
  directory: string,       // 工作目录
  parentID?: SessionID,     // 父会话（Task 子 Agent 时使用）
  title: string,           // 会话标题
  summary?: {
    additions: number,
    deletions: number,
    files: string[],
    diffs: string,
  },
  share?: { url: string },  // 分享链接
  time: {
    created: number,
    updated: number,
    compacting?: number,   // 上次压缩时间
    archived?: number,     // 归档时间
  }
}
```

### 2.2 Message 类型

```typescript
type MessageV2 = {
  id: MessageID
  sessionID: SessionID
  role: "user" | "assistant"
  parts: (
    | TextPart       // 文本内容
    | ToolPart       // 工具调用
    | ReasoningPart  // 思考过程
    | PatchPart      // 文件变更快照
    | FilePart       // 文件引用
  )[]
  time: { created: number; updated: number }
}

type TextPart = {
  type: "text"
  text: string
}

type ToolPart = {
  type: "tool_use" | "tool_result"
  id: string
  tool: string
  state: {
    status: "running" | "completed" | "error"
    input?: unknown
    output?: string
    metadata?: Record<string, unknown>
    title?: string
    time: { start: number; end?: number }
    attachments?: FilePart[]
  }
}

type ReasoningPart = {
  type: "reasoning"
  id: string
  text: string
}

type PatchPart = {
  type: "patch"
  hash: string
  files: string[]  // 变更的文件列表
}
```

---

## 3. 系统提示词体系

### 3.1 三层提示词结构

```
┌─────────────────────────────────────────────────────┐
│ Layer 1: Agent Prompt (Agent.prompt)                │
│   自定义 Agent 提示 或 Agent 类型对应提示          │
├─────────────────────────────────────────────────────┤
│ Layer 2: System Prompt (provider-specific)          │
│   根据模型选择: anthropic.txt / gpt.txt / ...      │
├─────────────────────────────────────────────────────┤
│ Layer 3: User System Prompt (message.metadata.system)│
│   用户通过消息元数据注入的额外提示                  │
└─────────────────────────────────────────────────────┘
```

### 3.2 Provider 级提示词

```typescript
// packages/opencode/src/session/system.ts
export function provider(model: Provider.Model) {
  if (model.api.id.includes("gpt-4") || model.api.id.includes("o1") || model.api.id.includes("o3"))
    return [PROMPT_BEAST]
  if (model.api.id.includes("gpt")) {
    if (model.api.id.includes("codex")) return [PROMPT_CODEX]
    return [PROMPT_GPT]
  }
  if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
  if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
  if (model.api.id.toLowerCase().includes("trinity")) return [PROMPT_TRINITY]
  if (model.api.id.toLowerCase().includes("kimi")) return [PROMPT_KIMI]
  return [PROMPT_DEFAULT]
}
```

OpenCode 支持 **11 种**模型专属提示词，远超 Claude Code 的覆盖范围。

### 3.3 提示词文件内容

**Anthropic 提示词** (`anthropic.txt`, 8,212 字节)：
- 语气与风格规范
- Emoji 使用策略
- TodoWrite 任务管理
- 专业客观性规则

**Beast 提示词** (`beast.txt`, 11,080 字节) — GPT-4/o1/o3：
- 最详尽的提示词
- 强调任务规划
- 工具使用最佳实践

---

## 4. LLM 流式接口

### 4.1 核心流式处理

```typescript
// packages/opencode/src/session/llm.ts (lines 1-127)
export async function stream(input: StreamRequest) {
  // 1. 获取语言模型和配置
  const [language, cfg, provider, auth] = await Promise.all([
    Provider.getLanguage(input.model),
    Config.get(),
    Provider.getProvider(input.model.providerID),
    Auth.get(input.model.providerID),
  ])

  // 2. 构建系统提示（三层合并）
  const system: string[] = []
  system.push([
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    ...input.system,
    ...(input.user.system ? [input.user.system] : []),
  ].filter(x => x).join("\n"))

  // 3. 插件钩子：允许插件转换系统提示
  await Plugin.trigger("experimental.chat.system.transform", { model: input.model }, { system })

  // 4. 解析工具（基于权限）
  const tools = await resolveTools(input)

  // 5. 转换消息格式
  const messages = isWorkflow
    ? input.messages
    : [
        ...system.map((x): ModelMessage => ({ role: "system", content: x })),
        ...input.messages,
      ]

  // 6. 调用 AI SDK
  return streamText({
    model: wrapLanguageModel({ ... }),
    activeTools: Object.keys(tools).filter((x) => x !== "invalid"),
    tools,
    maxOutputTokens,
    abortSignal: input.abort,
    messages,
    temperature: params.temperature,
    topP: params.topP,
    maxRetries: input.retries ?? 0,
  })
}
```

### 4.2 工具调用自动修复

```typescript
// packages/opencode/src/session/llm.ts
async experimental_repairToolCall(failed: { toolCall: ToolCall; reason: string }) {
  // 当工具调用参数不符合 schema 时，自动修复
  // 利用 AI 生成正确的参数
}
```

---

## 5. 会话处理器 — 事件循环

```typescript
// packages/opencode/src/session/processor.ts
export namespace SessionProcessor {
  export type Result = "compact" | "stop" | "continue"

  export const layer = Layer.effect(
    Service,
    Effect.gen(function* () {
      const create = Effect.fn("SessionProcessor.create")(function* (input: Input) {
        const ctx: ProcessorContext = {
          toolcalls: {},        // 活跃工具调用
          shouldBreak: false,   // 是否中断
          snapshot: initialSnapshot,  // 文件变更快照
          blocked: false,       // 是否被权限阻塞
          needsCompaction: false, // 是否需要压缩
          reasoningMap: {},     // 思考过程映射
        }

        const handleEvent = Effect.fn("SessionProcessor.handleEvent")(
          function* (value: StreamEvent) {
            switch (value.type) {
              case "tool-call": {
                // 创建 tool_use part
                ctx.toolcalls[value.toolCallId] = yield* session.updatePart({
                  ...match,
                  tool: value.toolName,
                  state: { status: "running", input: value.input, time: { start: Date.now() } },
                })

                // Doom loop 检测
                const recentParts = parts.slice(-DOOM_LOOP_THRESHOLD)
                if (isDoomLoop(recentParts)) {
                  yield* permission.ask({
                    permission: "doom_loop",
                    patterns: [value.toolName],
                    sessionID: ctx.sessionID,
                    metadata: { tool: value.toolName, input: value.input },
                    always: [value.toolName],
                    ruleset: agent.permission,
                  })
                }
                return
              }

              case "tool-result": {
                // 记录工具输出
                yield* session.updatePart({
                  ...ctx.toolcalls[value.toolCallId],
                  state: {
                    status: "completed",
                    input: value.input ?? match.state.input,
                    output: value.output.output,
                    metadata: value.output.metadata,
                    title: value.output.title,
                    time: { start: match.state.time.start, end: Date.now() },
                    attachments: value.output.attachments,
                  },
                })
                delete ctx.toolcalls[value.toolCallId]
                return
              }

              case "finish-step": {
                // 捕获文件变更快照
                if (ctx.snapshot) {
                  const patch = yield* snapshot.patch(ctx.snapshot)
                  if (patch.files.length) {
                    yield* session.updatePart({
                      type: "patch",
                      hash: patch.hash,
                      files: patch.files,
                    })
                  }
                }

                // 检查 token 溢出
                if (isOverflow({ cfg: yield* config.get(), tokens: usage.tokens, model: ctx.model })) {
                  ctx.needsCompaction = true
                }
                return
              }
            }
          },
        )

        const process = Effect.fn("SessionProcessor.process")(function* (streamInput: LLM.StreamInput) {
          const stream = llm.stream(streamInput)

          yield* stream.pipe(
            Stream.tap(event => handleEvent(event)),  // 处理每个事件
            Stream.takeUntil(() => ctx.needsCompaction),  // 压缩时停止
            Stream.runDrain,
          ).pipe(
            Effect.retry(SessionRetry.policy({ ... })),  // 重试策略
            Effect.catch(halt),
            Effect.ensuring(cleanup()),
          )

          if (aborted && !ctx.assistantMessage.error) yield* abort()
          if (ctx.needsCompaction) return "compact"
          if (ctx.blocked || ctx.assistantMessage.error || aborted) return "stop"
          return "continue"  // 继续下一轮
        })

        return { message, partFromToolCall, abort, process }
      })

      return Service.of({ create })
    }),
  )
}
```

**事件类型**：
- `start` — 开始处理
- `text` — 文本片段
- `reasoning` — 思考过程
- `tool-call` — 工具调用发起
- `tool-result` — 工具执行结果
- `finish-step` — 步骤完成
- `error` — 错误

---

## 6. 上下文压缩（Compaction）

### 6.1 触发条件

当 token 使用超过阈值时触发：

```typescript
function isOverflow({ cfg, tokens, model }): boolean {
  const maxTokens = cfg.provider?.[model.providerID]?.maxTokens ?? DEFAULT_MAX_TOKENS
  return tokens.total > maxTokens * OVERFLOW_RATIO
}
```

### 6.2 压缩流程

```typescript
// packages/opencode/src/session/compaction.ts
async function compact(sessionID: SessionID): Promise<void> {
  // 1. 获取旧消息
  const messages = yield* Session.messages(sessionID)

  // 2. 使用 compaction agent 生成摘要
  const summary = yield* Agent.generate({
    description: "Summarize conversation for context compaction",
  })

  // 3. 将摘要注入为 system prompt
  // 4. 删除旧消息
  // 5. 更新 compacting 时间戳
}
```

### 6.3 Compaction Agent 提示词

```typescript
// packages/opencode/src/agent/prompt/compaction.txt
"You are a helpful AI assistant tasked with summarizing conversations."

Focus on:
- What was accomplished
- Current work in progress
- Files being modified
- Next steps
- User constraints and technical decisions to persist
```

---

## 7. 重试策略

```typescript
// packages/opencode/src/session/retry.ts
export const policy = (options?: RetryOptions) =>
  Schedule.exponential(options?.delay ?? "100 millis", 2).pipe(
    Schedule.intersect(Schedule.elapsed()),
    Schedule.whileInput((cause) =>
      Cause.isFailure(cause) && isRetryable(cause) && options?.count < 3
    ),
  )
```

---

## 8. 与 Claude Code 对比

| 维度 | Claude Code | OpenCode |
|------|------------|----------|
| 消息存储 | 文件系统转录本 | SQLite + Drizzle ORM |
| 上下文管理 | System Prompt + 预算计算 | Compaction 摘要机制 |
| 模型提示词 | 单一通用提示 | 11 种 provider 特化提示 |
| 思考过程 | Extended Thinking Block | ReasoningPart 独立类型 |
| 重试策略 | 基础重试 | Effect Schedule 精确控制 |
| Doom loop | 简单计数检测 | 权限请求 + 阈值检测 |
| 插件钩子 | hooks.ts | `experimental.chat.system.transform` |

---

*文档版本：v1.0 | 更新：2026-04-06*
