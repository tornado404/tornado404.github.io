---
title: "架构概览 — Turborepo Monorepo 与核心模块"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 架构概览 — Turborepo Monorepo 与核心模块

> 源码路径：`/mnt/e/code/cc/opencode/packages/opencode/src/`
> 核心文件：`agent/agent.ts`, `tool/tool.ts`, `session/`, `config/config.ts`
> 技术栈：Bun + TypeScript (strict) + Effect + Drizzle ORM + SQLite

---

## 1. 概述

OpenCode 采用了与 Claude Code 截然不同的架构策略：**Turborepo Monorepo**。整个项目拆分为 30+ 个独立包，各司其职，通过 `turbo.json` 管理构建依赖。这种架构的优势在于模块边界清晰、发布灵活，但同时也带来了更高的复杂度。

```
/
├── packages/
│   ├── opencode/         # 核心引擎（最重要的包）
│   │   └── src/
│   │       ├── agent/    # Agent 定义与生命周期
│   │       ├── tool/     # 工具注册与执行
│   │       ├── session/  # 会话管理、消息处理
│   │       ├── command/  # Slash 命令系统
│   │       ├── config/   # 多层配置系统
│   │       ├── permission/  # 权限模型
│   │       ├── mcp/      # MCP 客户端
│   │       ├── provider/ # AI 模型提供商
│   │       ├── plugin/   # 插件钩子系统
│   │       ├── skill/    # Skill 发现与加载
│   │       ├── ide/      # IDE 检测与集成
│   │       ├── server/   # HTTP RPC 服务器
│   │       ├── bus/      # 事件总线
│   │       ├── storage/  # SQLite + Drizzle ORM
│   │       ├── effect/   # Effect 运行时封装
│   │       ├── acp/      # Agent Client Protocol
│   │       ├── lsp/      # Language Server Protocol
│   │       └── ...
│   ├── app/              # Web UI / TUI 前端
│   ├── console/          # CLI 入口
│   ├── desktop/          # 桌面应用封装
│   ├── desktop-electron/ # Electron 封装
│   ├── sdk/              # 外部 SDK (JS)
│   ├── sdks/             # VS Code SDK 等
│   ├── extensions/       # Zed 扩展定义
│   ├── ui/               # 共享 UI 组件库
│   ├── util/             # 共享工具库
│   └── ...
├── install/              # 安装脚本
├── infra/                # 基础设施配置
└── nix/                  # Nix 包定义
```

---

## 2. 核心包详解

### 2.1 packages/opencode — 核心引擎

这是最重要的包，所有核心逻辑都在 `packages/opencode/src/` 下：

```
opencode/src/
├── agent/
│   ├── agent.ts          # Agent 定义类型 + Service 实现 (~420 行)
│   ├── generate.txt       # Agent 动态生成提示词
│   └── prompt/            # Agent 级专用提示词
│       ├── compaction.txt # 上下文压缩提示
│       ├── explore.txt    # 探索 Agent 提示
│       ├── summary.txt    # 总结提示
│       └── title.txt      # 会话标题生成提示
├── tool/
│   ├── tool.ts            # 工具核心类型定义 (~112 行)
│   ├── registry.ts        # 工具注册中心 (~265 行)
│   ├── truncate.ts        # 输出截断服务
│   ├── schema.ts          # 工具 ID Schema
│   ├── bash.ts            # Bash 执行工具
│   ├── read.ts            # 文件读取工具
│   ├── write.ts           # 文件写入工具
│   ├── edit.ts            # 文件编辑工具 (高级模糊匹配)
│   ├── glob.ts            # 文件模式匹配
│   ├── grep.ts            # 正则搜索
│   ├── task.ts            # 子 Agent / 任务工具
│   ├── batch.ts           # 批量执行工具
│   ├── skill.ts           # Skill 调用
│   ├── lsp.ts             # LSP 集成
│   ├── multiedit.ts       # 多重编辑
│   ├── apply_patch.ts     # Patch 应用
│   └── ...
├── session/
│   ├── prompt.ts          # 会话提示词解析 (~1,908 行)
│   ├── processor.ts       # LLM 事件处理 (~523 行)
│   ├── llm.ts             # LLM 流式接口 (~358 行)
│   ├── system.ts          # 系统提示词选择
│   ├── retry.ts           # 重试策略
│   ├── prompt/            # Provider 级系统提示词
│   │   ├── anthropic.txt  # Claude 模型
│   │   ├── beast.txt      # GPT-4/o1/o3
│   │   ├── gpt.txt        # 普通 GPT
│   │   ├── gemini.txt     # Gemini
│   │   ├── codex.txt      # GitHub Copilot
│   │   ├── kimi.txt       # Kimi
│   │   ├── trinity.txt    # Trinity
│   │   ├── default.txt    # 默认兜底
│   │   └── ...
│   └── compaction.ts     # 上下文压缩
├── config/
│   ├── config.ts          # 主配置 Schema + 加载 (~849+ 行)
│   ├── tui.ts             # TUI 特定配置
│   ├── paths.ts           # 路径解析 + JSONC 支持
│   └── ...
├── permission/
│   └── index.ts           # Ruleset 权限模型
├── mcp/
│   └── index.ts           # MCP 客户端实现
├── provider/
│   └── index.ts           # 20+ AI 模型提供商
├── command/
│   └── index.ts           # 命令定义与加载
├── plugin/
│   └── index.ts           # 插件钩子系统
├── skill/
│   └── index.ts           # Skill 发现与加载
├── ide/
│   └── index.ts           # IDE 检测 (VS Code, Cursor, Zed...)
├── acp/
│   ├── agent.ts           # ACP Agent 接口
│   ├── session.ts         # ACP 会话管理
│   └── types.ts           # ACP 类型定义
├── server/
│   └── index.ts           # Hono HTTP 服务器
├── bus/
│   └── index.ts           # 事件总线
├── storage/
│   └── index.ts           # SQLite + Drizzle ORM
├── effect/
│   └── index.ts           # Effect 运行时封装
├── lsp/
│   └── index.ts           # LSP 客户端实现
├── project/
│   └── index.ts           # 项目发现
├── auth/
│   └── index.ts           # 认证系统
├── filesystem/
│   └── index.ts           # 文件系统封装
└── ...
```

### 2.2 packages/extensions — Zed 扩展

Zed 编辑器的扩展定义，使用 TOML 配置：

```toml
# packages/extensions/zed/extension.toml
[extension]
name = "opencode"
agent_server = "opencode"

[[platform]]
os = "macos"
arch = "arm64"
url = "..."

[[platform]]
os = "macos"
arch = "x86_64"
url = "..."
```

通过 `opencode acp` 命令启动 ACP 服务器，Zed 通过 stdio 双向通信。

### 2.3 packages/sdk — 外部 SDK

提供 HTTP API 客户端，支持目录头注入：

```typescript
const client = createOpencodeClient({
  baseUrl: `http://${hostname}:${port}`,
  directory: cwd,
})
```

### 2.4 packages/app — Web UI

基于 Vite + TypeScript 的前端应用，提供 Web 界面。

---

## 3. 技术栈分析

### 3.1 Effect — 核心函数式框架

OpenCode 几乎所有服务都基于 **Effect** 框架构建。Effect 是 Scala ZIO 的 TypeScript 实现，提供：

- **代数效应 (Algebraic Effects)** — 用 `Effect<S, E, A>` 表示可能失败和并发的计算
- **服务模式 (Service Pattern)** — 通过 `Service.of()` 定义接口，通过 `Layer` 组合依赖
- **Fiber 并发** — 轻量级协程，支持结构化并发
- **资源管理** — `addFinalizer` 自动清理

这与 Claude Code 使用 `AsyncLocalStorage` 的方式形成鲜明对比。Effect 的优势在于类型安全的依赖注入和组合，劣势是学习曲线较陡。

```typescript
// OpenCode 的 Service 定义模式
export const layer: Layer.Layer<Service, never, Config.Service | Auth.Service | ...> =
  Layer.effect(
    Service,
    Effect.gen(function* () {
      const config = yield* Config.Service
      const auth = yield* Auth.Service
      // ...
      return Service.of({
        get: Effect.fn("Agent.get")(function* (name: string) { /* ... */ }),
        list: Effect.fn("Agent.list")(function* () { /* ... */ }),
      })
    }),
  )
```

### 3.2 AI SDK — 模型抽象层

OpenCode 使用 Vercel 的 **AI SDK** 作为 LLM 调用层：

```typescript
import { streamText, generateObject } from 'ai'
import { wrapLanguageModel } from 'ai-provider-openai' // 或其他 provider

return streamText({
  model: wrapLanguageModel({ ... }),
  tools,
  maxOutputTokens,
  abortSignal,
  messages,
})
```

AI SDK 提供了 provider 无关的流式接口，OpenCode 通过 `provider/` 包支持 20+ 模型（Anthropic、OpenAI、Azure、Google、AWS 等）。

### 3.3 Drizzle ORM + SQLite — 持久化

会话消息使用 Drizzle ORM 存储在 SQLite 中：

- **WAL 模式** — 支持并发读写
- **迁移系统** — 带时间戳的版本化管理
- **规范化存储** — `MessageTable` + `PartTable` 分离

### 3.4 Hono — HTTP 服务器

轻量级高性能 Web 框架，用于 RPC 和 API 端点。

---

## 4. 架构分层

```
┌──────────────────────────────────────────────────────────────────┐
│                         User Interface                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────┐  │
│  │  CLI / TUI   │  │  Web UI     │  │  VS Code SDK │  │  Zed │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──┬───┘  │
└─────────┼─────────────────┼─────────────────┼─────────────┼──────┘
          │                 │                 │             │
          └─────────────────┴────────┬────────┴─────────────┘
                                     │ HTTP / ACP / stdio
┌─────────────────────────────────────┴──────────────────────────────┐
│                     packages/opencode/src/                         │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                      Server Layer                          │   │
│  │  HTTP RPC (Hono) + WebSocket + ACP Protocol Handler        │   │
│  └────────────────────────────────────────────────────────────┘   │
│                              │                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                    Session Layer                            │   │
│  │  SessionPrompt / SessionProcessor / LLM Stream / Compaction │   │
│  └────────────────────────────────────────────────────────────┘   │
│                              │                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                     Agent Layer                             │   │
│  │  Agent Service / Tool Registry / Permission / MCP           │   │
│  └────────────────────────────────────────────────────────────┘   │
│                              │                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                   Tool Execution Layer                      │   │
│  │  Bash / Read / Write / Edit / Glob / Grep / LSP / ...      │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                  Infrastructure Layer                       │   │
│  │  Effect Runtime / Config / Storage / Auth / Event Bus      │   │
│  └────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. 核心类型体系

### 5.1 Agent 定义

```typescript
// packages/opencode/src/agent/agent.ts
export namespace Agent {
  export const Info = z.object({
    name: z.string(),
    description: z.string().optional(),
    mode: z.enum(["subagent", "primary", "all"]),  // Agent 角色
    native: z.boolean().optional(),
    hidden: z.boolean().optional(),
    topP: z.number().optional(),
    temperature: z.number().optional(),
    color: z.string().optional(),
    permission: Permission.Ruleset,  // 权限规则集
    model: z.object({ modelID: ModelID.zod, providerID: ProviderID.zod }).optional(),
    variant: z.string().optional(),
    prompt: z.string().optional(),    // 自定义系统提示
    options: z.record(z.string(), z.any()),
    steps: z.number().int().positive().optional(),
  })

  export interface Interface {
    readonly get: (agent: string) => Effect.Effect<Agent.Info>
    readonly list: () => Effect.Effect<Agent.Info[]>
    readonly defaultAgent: () => Effect.Effect<string>
    readonly generate: (input: {
      description: string
      model?: { providerID: ProviderID; modelID: ModelID }
    }) => Effect.Effect<{ identifier: string; whenToUse: string; systemPrompt: string }>
  }
}
```

### 5.2 Tool 定义

```typescript
// packages/opencode/src/tool/tool.ts
export namespace Tool {
  export interface Context<M extends Metadata = Metadata> {
    sessionID: SessionID
    messageID: MessageID
    agent: string
    abort: AbortSignal
    callID?: string
    extra?: { [key: string]: any }
    messages: MessageV2.WithParts[]
    metadata(input: { title?: string; metadata?: M }): void
    ask(input: Omit<Permission.Request, "id" | "sessionID" | "tool">): Promise<void>
  }

  export interface Def<Parameters extends z.ZodType = z.ZodType, M extends Metadata = Metadata> {
    description: string
    parameters: Parameters
    execute(args: z.infer<Parameters>, ctx: Context): Promise<{
      title: string
      metadata: M
      output: string
      attachments?: Omit<MessageV2.FilePart, "id" | "sessionID" | "messageID">[]
    }>
    formatValidationError?(error: z.ZodError): string
  }

  export function define<...>(id: string, init: ...): Info<...>
}
```

### 5.3 Session 定义

```typescript
// Session 核心信息
Session.Info = {
  id: SessionID,
  slug: string,
  projectID: ProjectID,
  directory: string,
  parentID?: SessionID,    // 父会话（用于 Task 子 Agent）
  title: string,
  summary?: { additions, deletions, files, diffs },
  share?: { url },
  time: { created, updated, compacting, archived }
}

// Message 类型
type MessageV2 = {
  id: MessageID
  sessionID: SessionID
  role: "user" | "assistant"
  parts: (TextPart | ToolPart | ReasoningPart | PatchPart | FilePart)[]
  time: { created: number; updated: number }
}
```

---

## 6. 与 Claude Code 架构对比

| 维度 | Claude Code | OpenCode |
|------|------------|----------|
| 组织结构 | 单仓库 | Turborepo 30+ 包 |
| 核心框架 | Bun + AsyncLocalStorage | Bun + Effect |
| 依赖注入 | 隐式（闭包 + 参数传递） | 显式（Layer + Service） |
| 消息存储 | 文件系统转录本 | SQLite + Drizzle ORM |
| 配置 | TOML 单文件 | JSONC 多层合并 |
| AI SDK | 底层直接调用 API | Vercel AI SDK 封装 |
| 协议层 | 无专门的外部协议 | ACP (Agent Client Protocol) |
| 发布方式 | npm 全局包 | Monorepo 多包发布 |

---

## 7. 核心文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `packages/opencode/src/agent/agent.ts` | ~420 | Agent 类型定义 + Service 实现 |
| `packages/opencode/src/tool/tool.ts` | ~112 | Tool 核心类型 + wrap 函数 |
| `packages/opencode/src/tool/registry.ts` | ~265 | 工具注册中心 |
| `packages/opencode/src/session/prompt.ts` | ~1,908 | 会话提示词 + 生成循环 |
| `packages/opencode/src/session/processor.ts` | ~523 | LLM 事件处理器 |
| `packages/opencode/src/session/llm.ts` | ~358 | LLM 流式接口 |
| `packages/opencode/src/config/config.ts` | ~849+ | 配置 Schema + 加载 |
| `packages/opencode/src/permission/index.ts` | — | 权限 Ruleset 模型 |

---

*文档版本：v1.0 | 更新：2026-04-06*
