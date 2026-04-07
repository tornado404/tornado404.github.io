---
title: "生命周期钩子 — 52 个 Hook 的三层架构"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 生命周期钩子 — 52 个 Hook 的三层架构

> 源码路径：`/mnt/e/code/cc/omo-code/src/hooks/`, `src/plugin/hooks/`
> 核心文件：`create-session-hooks.ts`, `create-tool-guard-hooks.ts`, `create-transform-hooks.ts`
> 技术栈：Hook Pipeline + Effect + EventEmitter

---

## 1. 概述

Oh-My-OpenAgent 的 Hook 系统是其可扩展性的核心。整个系统包含 **52 个命名 Hook**，分为三层：

```
┌──────────────────────────────────────────────┐
│           Core Hooks (24 个)                 │
│  Session 生命周期、上下文监控、恢复           │
├──────────────────────────────────────────────┤
│        Continuation Hooks (15 个)            │
│  工具守卫、输出转换、持久化                   │
├──────────────────────────────────────────────┤
│          Skill Hooks (2 个)                  │
│  Skill 加载和上下文                           │
├──────────────────────────────────────────────┤
│         Transform Hooks (5 个)                │
│  消息转换、关键词检测、上下文注入             │
├──────────────────────────────────────────────┤
│           事件 Hooks (6 个)                   │
│  会话事件、桌面通知                           │
└──────────────────────────────────────────────┘
```

---

## 2. 会话 Hooks (Core — 24 个)

### 2.1 上下文窗口监控

```typescript
// src/hooks/context-window-monitor/
export const contextWindowMonitor = {
  name: 'context-window-monitor',

  // OpenAI 128K, Claude 200K, Gemini 1M
  thresholds: {
    'claude-opus-4-6': { warn: 0.70, critical: 0.85 },
    'claude-sonnet-4-6': { warn: 0.70, critical: 0.85 },
    'gpt-5.4': { warn: 0.75, critical: 0.90 },
  },

  async check(session: Session, usage: TokenUsage) {
    const model = session.model
    const threshold = this.thresholds[model] ?? { warn: 0.75, critical: 0.90 }

    const ratio = usage.total / getModelLimit(model)

    if (ratio >= threshold.critical) {
      return 'compact'  // 触发压缩
    }
    if (ratio >= threshold.warn) {
      return 'warn'    // 发出警告
    }
    return 'ok'
  },
}
```

### 2.2 抢占式压缩

```typescript
// src/hooks/preemptive-compaction/
export const preemptiveCompaction = {
  name: 'preemptive-compaction',

  async shouldCompact(session: Session, config: Config): Promise<boolean> {
    const usage = await session.getTokenUsage()
    const ratio = usage.total / getModelLimit(session.model)
    const threshold = config.experimental?.compaction_threshold ?? 0.70

    // 当达到 70% 阈值时自动压缩
    return ratio >= threshold
  },

  async compact(session: Session): Promise<CompactionResult> {
    // 1. 识别可压缩的消息（工具结果）
    const compressible = session.messages.filter(msg =>
      msg.role === 'tool' && !msg.isCritical
    )

    // 2. 生成摘要
    const summary = await generateSummary(compressible)

    // 3. 替换为摘要消息
    await session.replace(compressible, {
      role: 'system',
      content: `[Compressed ${compressible.length} messages]\n${summary}`,
    })

    // 4. 保留 Todo 和关键上下文
    await preserveCriticalContext(session)

    return { compressed: compressible.length, preserved: ['todos', 'key_context'] }
  },
}
```

### 2.3 模型回退

```typescript
// src/hooks/runtime-fallback/
export const runtimeFallback = {
  name: 'runtime-fallback',

  circuitBreaker: new CircuitBreaker({
    failureThreshold: 3,
    resetTimeout: 60_000,  // 60 秒后重试
  }),

  async handle(error: Error, session: Session): Promise<FallbackResult> {
    if (!this.circuitBreaker.isOpen()) {
      const currentModel = session.model
      const fallback = config.model_fallback?.[currentModel]

      if (fallback) {
        this.circuitBreaker.recordSuccess()
        await session.switchModel(fallback)
        return { switched: true, newModel: fallback }
      }
    }

    if (error.type === 'context_window_exceeded') {
      // 上下文超限 → 切换到更大上下文模型
      const larger = findLargerContextModel(session.model)
      if (larger) {
        return { switched: true, newModel: larger }
      }
    }

    return { switched: false }
  },
}
```

---

## 3. 工具守卫 Hooks (Continuation — 15 个)

### 3.1 AI Slop 检测

```typescript
// src/hooks/comment-checker/
export const commentChecker = {
  name: 'comment-checker',

  patterns: [
    // 过度注释
    /^\s*#\s*This (file|function|class) (is used to|represents|handles)/m,
    // 明显的 AI 生成注释
    /^\s*#\s*(Absolutely!|Certainly!|Here's|Let me|In this code)/m,
    // 重复的 TODO
    /TODO.*TODO.*TODO/s,
  ],

  async check(code: string): Promise<SlopResult> {
    const matches: SlopMatch[] = []

    for (const pattern of this.patterns) {
      const found = code.match(pattern)
      if (found) {
        matches.push({
          pattern: pattern.source,
          line: findLineNumber(code, found.index!),
          severity: classifySeverity(pattern, found[0]),
        })
      }
    }

    if (matches.length > config.comment_checker?.max_allowed ?? 3) {
      return {
        hasSlop: true,
        matches,
        action: config.comment_checker?.auto_fix ? 'fix' : 'warn',
      }
    }

    return { hasSlop: false, matches: [] }
  },
}
```

### 3.2 Todo 强制执行

```typescript
// src/hooks/todo-continuation-enforcer/
export const todoContinuationEnforcer = {
  name: 'todo-continuation-enforcer',

  // 如果 Agent 空闲超过这个时间且有未完成 Todo，就把它拉回来
  idleThresholdMs: 30_000,

  async check(session: Session): Promise<EnforceResult> {
    const todos = await session.getTodos()
    const activeTodos = todos.filter(t => !t.completed)

    if (activeTodos.length === 0) return { shouldEnforce: false }

    const lastActivity = session.lastActivityTimestamp
    const idleTime = Date.now() - lastActivity

    if (idleTime > this.idleThresholdMs) {
      // Agent 空闲太久 → 强制继续 Todo
      return {
        shouldEnforce: true,
        message: `You have ${activeTodos.length} incomplete todos. Continue working on them.`,
        resumeFrom: activeTodos[0],
      }
    }

    return { shouldEnforce: false }
  },
}
```

### 3.3 JSON 错误恢复

```typescript
// src/hooks/json-error-recovery/
export const jsonErrorRecovery = {
  name: 'json-error-recovery',

  async fix(error: ParseError, context: ToolContext): Promise<string> {
    if (error.type === 'unexpected_token') {
      // 尝试补全括号
      const fixed = fixBracketBalance(error.input)
      try {
        JSON.parse(fixed)
        return fixed
      } catch {
        // 尝试更激进的修复
        return fixCommonJsonErrors(error.input)
      }
    }
    return error.input
  },
}
```

---

## 4. Transform Hooks (5 个)

### 4.1 关键词检测 → Agent 触发

```typescript
// src/hooks/keyword-detector/
export const keywordDetector = {
  keywords: {
    architect: { agent: 'oracle', confidence: 0.9 },
    refactor: { agent: 'prometheus', confidence: 0.8 },
    search: { agent: 'librarian', confidence: 0.85 },
    explore: { agent: 'explore', confidence: 0.95 },
    plan: { agent: 'prometheus', confidence: 0.85 },
    review: { agent: 'momus', confidence: 0.8 },
    security: { agent: 'oracle', confidence: 0.9 },
  },

  detect(message: string): DetectionResult | null {
    const lower = message.toLowerCase()

    for (const [keyword, config] of Object.entries(this.keywords)) {
      if (lower.includes(keyword)) {
        return {
          agent: config.agent,
          confidence: config.confidence,
          matched: keyword,
        }
      }
    }

    return null
  },
}
```

### 4.2 上下文注入

```typescript
// src/hooks/contextInjectorMessagesTransform/
export const contextInjector = {
  async transform(messages: Message[], ctx: ChatContext): Promise<Message[]> {
    if (!ctx.isFirstMessage) return messages

    const injected: Message[] = []

    // AGENTS.md
    const agentsMd = path.join(ctx.directory, 'AGENTS.md')
    if (await exists(agentsMd)) {
      injected.push({
        role: 'system',
        content: `[AGENTS.md]\n${await readFile(agentsMd)}`,
      })
    }

    // README.md (前 200 行)
    const readme = path.join(ctx.directory, 'README.md')
    if (await exists(readme)) {
      injected.push({
        role: 'system',
        content: `[README.md]\n${await readFileLines(readme, 0, 200)}`,
      })
    }

    // CLAUDE.md (如果有)
    const claudeMd = path.join(ctx.directory, 'CLAUDE.md')
    if (await exists(claudeMd)) {
      injected.push({
        role: 'system',
        content: `[CLAUDE.md]\n${await readFile(claudeMd)}`,
      })
    }

    return [...injected, ...messages]
  },
}
```

---

## 5. 事件 Hooks (6 个)

```typescript
// src/hooks/session-notification/
export const sessionNotification = {
  events: ['session.created', 'session.completed', 'session.error', 'session.idle'],

  async handle(event: SessionEvent): Promise<void> {
    switch (event.type) {
      case 'session.created':
        await send({
          title: 'Session Started',
          body: event.session.title,
        })
        break

      case 'session.completed':
        await send({
          title: 'Session Completed',
          body: `${event.session.title}: ${event.summary}`,
        })
        break

      case 'session.error':
        await send({
          title: 'Session Error',
          body: event.error.message,
          urgency: 'critical',
        })
        break
    }
  },
}
```

---

## 6. Hook 配置禁用

```typescript
// 用户可以通过配置禁用任意 Hook
{
  "disabled_hooks": [
    "comment-checker",
    "session-notification",
    "tool-output-truncator"
  ],

  "experimental": {
    "think_mode": true,
    "compaction_threshold": 0.75,
    "safe_hook_creation": true
  }
}
```

---

## 7. 与 OpenCode 对比

| 维度 | OpenCode | Oh-My-OpenAgent |
|------|----------|-----------------|
| Hook 总数 | ~10 | **52** |
| 压缩策略 | 阈值触发 | **抢占式 70% 阈值** |
| 错误恢复 | 基础重试 | **断路器 + 模型回退** |
| 工具守卫 | 无 | **AI slop、JSON 修复、文件守卫** |
| 通知 | 无 | **桌面通知** |
| Todo 强制 | 无 | **空闲拉回机制** |
| 思考块验证 | 无 | **Thinking Block 验证** |

---

*文档版本：v1.0 | 更新：2026-04-06*
