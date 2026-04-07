---
title: "插件接入 — Hook 系统与 OpenCode 集成"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 插件接入 — Hook 系统与 OpenCode 集成

> 源码路径：`/mnt/e/code/cc/omo-code/src/plugin-handlers/`, `src/hooks/`
> 核心文件：`config-handler.ts`, `agent-config-handler.ts`, `tool-config-handler.ts`
> 技术栈：@opencode-ai/plugin + Effect + Zod

---

## 1. 概述

Oh-My-OpenAgent 通过 OpenCode 的 **Plugin Hook 接口**接入其执行管线。它不像 Claude Code 的工具那样直接调用，而是通过**拦截和装饰** OpenCode 的关键生命周期来实现扩展。

---

## 2. 10 个 OpenCode 钩子接口

OpenCode 的 `@opencode-ai/plugin` 定义了以下钩子接口，Oh-My-OpenAgent 全部实现了它们：

| 钩子 | 触发时机 | Oh-My-OpenAgent 处理 |
|------|----------|---------------------|
| `config` | 配置加载阶段 | 6 阶段配置管道 |
| `tool` | 工具注册阶段 | 26 个工具注册 |
| `chat.message` | 聊天消息处理 | 首次消息变体、会话设置 |
| `chat.params` | 聊天参数调整 | Anthropic effort level |
| `chat.headers` | HTTP 头注入 | Copilot x-initiator |
| `event` | 会话生命周期 | 通知、桌面提醒 |
| `tool.execute.before` | 工具执行前 | 文件守卫、标签截断 |
| `tool.execute.after` | 工具执行后 | 输出截断、元数据存储 |
| `experimental.chat.messages.transform` | 消息转换 | 上下文注入、思考块验证 |
| `experimental.session.compacting` | 会话压缩时 | 上下文 + Todo 保留 |

---

## 3. 6 阶段配置管道

```typescript
// src/plugin-handlers/config-handler.ts
export async function handleConfig(
  phase: ConfigPhase,
  ctx: PluginContext,
): Promise<void> {

  switch (phase) {
    case "provider": {
      // 阶段 1: Provider 配置 — 模型缓存状态
      const providers = yield* Config.provider.all()
      // 缓存 provider 信息用于后续模型选择
      ctx.cache.set('providers', providers)
      break
    }

    case "plugin-components": {
      // 阶段 2: 插件组件 — 加载内置 Agent、Skills、命令
      yield* loadBuiltinAgents(config)
      yield* loadBuiltinSkills(config)
      yield* loadBuiltinCommands(config)
      break
    }

    case "agent": {
      // 阶段 3: Agent 配置 — 应用 Agent 覆盖、模型选择
      yield* applyAgentOverrides(config)
      yield* resolveAgentModels(config)
      yield* injectAgentPrompts(config)
      break
    }

    case "tool": {
      // 阶段 4: 工具配置 — 注册 26 个工具、禁用不需要的
      yield* registerTools(config)
      yield* disableTools(config.disabled_tools)
      yield* configureToolCapabilities(config)
      break
    }

    case "mcp": {
      // 阶段 5: MCP 配置 — 设置内置 MCP (websearch, context7, grep_app)
      yield* setupBuiltinMcps(config)
      yield* connectExternalMcps(config.mcps)
      break
    }

    case "command": {
      // 阶段 6: 命令配置 — 注册内置命令
      yield* registerBuiltinCommands(config)
      yield* registerCustomCommands(config.commands)
      break
    }
  }
}
```

---

## 4. 工具钩子

### 4.1 执行前钩子 (tool.execute.before)

```typescript
// src/plugin/hooks/create-tool-guard-hooks.ts
export function createToolExecuteBeforeHooks(config: Config) {
  return [
    // 写文件守卫 — 防止覆盖重要文件
    {
      name: 'write-existing-file-guard',
      handler: async (tool: ToolCall, ctx: ToolContext) => {
        if (tool.name === 'write' || tool.name === 'edit') {
          const filePath = extractFilePath(tool.args)
          if (isProtectedFile(filePath)) {
            throw new ToolBlockedError(
              `Writing to ${filePath} is protected. ` +
              `This file should not be modified by AI agents.`
            )
          }
        }
      },
    },

    // Bash 文件读取守卫
    {
      name: 'bash-file-read-guard',
      handler: async (tool: ToolCall, ctx: ToolContext) => {
        if (tool.name === 'bash') {
          const dangerous = detectDangerousCommands(tool.args.command)
          if (dangerous) {
            await ctx.ask({
              permission: 'bash',
              patterns: [dangerous],
              reason: `Potentially dangerous command detected: ${dangerous}`,
            })
          }
        }
      },
    },

    // 标签截断器
    {
      name: 'tool-label-truncator',
      handler: async (tool: ToolCall, ctx: ToolContext) => {
        if (tool.args.description?.length > 100) {
          tool.args.description = truncate(tool.args.description, 100)
        }
      },
    },

    // 规则注入器 — 注入 Claude Code 用户规则
    {
      name: 'rules-injector',
      handler: async (tool: ToolCall, ctx: ToolContext) => {
        const rules = loadUserRules(ctx.sessionID)
        if (rules.length > 0) {
          ctx.messages.push({
            role: 'system',
            content: `User Rules:\n${rules.join('\n')}`,
          })
        }
      },
    },
  ]
}
```

### 4.2 执行后钩子 (tool.execute.after)

```typescript
// src/plugin/hooks/create-tool-after-hooks.ts
export function createToolExecuteAfterHooks(config: Config) {
  return [
    // 输出截断器
    {
      name: 'tool-output-truncator',
      handler: async (result: ToolResult, ctx: ToolContext) => {
        const maxLines = config.experimental?.max_output_lines ?? 2000
        const maxBytes = config.experimental?.max_output_bytes ?? 50 * 1024

        if (result.output.split('\n').length > maxLines) {
          result.output = truncateOutput(result.output, maxLines, maxBytes)
          result.metadata.truncated = true
        }
      },
    },

    // 元数据存储
    {
      name: 'tool-metadata-store',
      handler: async (result: ToolResult, ctx: ToolContext) => {
        ctx.cache.set(`tool:${tool.name}:${Date.now()}`, {
          duration: result.durationMs,
          success: result.error === undefined,
          tokenEstimate: estimateTokens(result.output),
        })
      },
    },
  ]
}
```

---

## 5. 消息转换钩子

```typescript
// src/plugin/hooks/create-transform-hooks.ts
export function createTransformHooks(config: Config) {
  return {
    ['experimental.chat.messages.transform']: async (
      messages: Message[],
      ctx: ChatContext,
    ) => {
      // 1. 上下文注入 — AGENTS.md / README.md
      if (ctx.isFirstMessage) {
        const contextFiles = discoverContextFiles(ctx.directory)
        for (const file of contextFiles) {
          messages.unshift({
            role: 'system',
            content: `Context from ${file.path}:\n${file.content}`,
          })
        }
      }

      // 2. 思考块验证
      if (config.experimental?.think_mode) {
        for (const msg of messages) {
          if (msg.role === 'assistant') {
            validateThinkingBlocks(msg.content)
          }
        }
      }

      // 3. 关键词检测 → Agent 触发
      const intent = detectKeywordIntent(messages[messages.length - 1]?.content)
      if (intent) {
        ctx.metadata.suggestedAgent = intent.agent
      }

      return messages
    },

    ['experimental.session.compacting']: async (
      session: Session,
      summary: CompactionSummary,
    ) => {
      // 压缩时保留 Todo 和关键上下文
      const todos = yield* Session.getTodos(session.id)
      const criticalContext = yield* extractCriticalContext(session)

      summary.preserved = {
        todos,
        criticalContext,
      }
    },
  }
}
```

---

## 6. 事件钩子

```typescript
// src/plugin/hooks/create-session-hooks.ts
export function createEventHooks(config: Config) {
  return {
    event: async (event: SessionEvent) => {
      switch (event.type) {
        case 'session.created':
          await config.notification?.send(
            `New session: ${event.session.title}`
          )
          break

        case 'session.idle':
          if (config.babysitting?.enabled) {
            // 空闲时检查是否需要干预
            yield* babysitSession(event.session)
          }
          break

        case 'session.error':
          await config.notification?.send(
            `Session error: ${event.error.message}`
          )
          break
      }
    },
  }
}
```

---

## 7. 与 OpenCode Plugin 系统对比

| 维度 | OpenCode 基础 | Oh-My-OpenAgent |
|------|---------------|-----------------|
| 钩子数量 | ~10 个基础钩子 | 52 个三层 Hook |
| 配置阶段 | 无 | 6 阶段管道 |
| 工具钩子 | 无 | 执行前守卫 + 执行后处理 |
| 消息钩子 | 无 | 上下文注入 + 思考块验证 |
| 事件钩子 | 基础事件 | 通知 + 干预 |

---

*文档版本：v1.0 | 更新：2026-04-06*
