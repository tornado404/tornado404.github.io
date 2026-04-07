---
title: "工具系统 — Effect 框架与工具注册"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 工具系统 — Effect 框架与工具注册

> 源码路径：`/mnt/e/code/cc/opencode/packages/opencode/src/tool/`
> 核心文件：`tool.ts`, `registry.ts`, `bash.ts`, `read.ts`, `edit.ts`, `truncate.ts`
> 技术栈：Effect + Zod + AI SDK

---

## 1. 概述

OpenCode 的工具系统与 Claude Code 有本质区别：

- **Claude Code**：工具是 TypeScript 函数，注册到 `Tools` 映射中，通过 `canUseTool` 权限检查
- **OpenCode**：工具是 **Effect Service**，通过 `Tool.define()` 定义，使用 Zod 做参数验证，整体作为 Effect 层的一部分注入

这种设计使得工具的依赖注入、测试和替换都非常自然——只需要替换 Layer 即可。

---

## 2. 工具核心类型

### 2.1 Tool.Info — 工具定义

```typescript
// packages/opencode/src/tool/tool.ts
export namespace Tool {
  // 工具初始化参数
  export interface InitContext {
    cwd?: string
    agent?: string
    // 其他上下文信息
  }

  // 工具执行结果
  export interface Result<M extends Metadata = Metadata> {
    title: string          // 工具执行标题
    metadata: M           // 元数据（可自定义）
    output: string        // 执行输出文本
    attachments?: Omit<MessageV2.FilePart, "id" | "sessionID" | "messageID">[]
  }

  // 工具执行上下文
  export interface Context<M extends Metadata = Metadata> {
    sessionID: SessionID
    messageID: MessageID
    agent: string
    abort: AbortSignal
    callID?: string
    extra?: { [key: string]: any }
    messages: MessageV2.WithParts[]   // 当前会话所有消息
    metadata(input: { title?: string; metadata?: M }): void  // 更新 part 状态
    ask(input: Omit<Permission.Request, "id" | "sessionID" | "tool">): Promise<void>  // 请求权限
  }

  // 工具执行定义
  export interface Def<Parameters extends z.ZodType = z.ZodType, M extends Metadata = Metadata> {
    description: string
    parameters: Parameters
    execute(args: z.infer<Parameters>, ctx: Context): Promise<Result<M>>
    formatValidationError?(error: z.ZodError): string
  }

  // 工具信息（带初始化）
  export interface Info<Parameters extends z.ZodType = z.ZodType, M extends Metadata = Metadata> {
    id: string
    init: (ctx?: InitContext) => Promise<Def<Parameters, M>>
  }

  // 工厂函数
  export function define<Parameters extends z.ZodType, Result extends Metadata>(
    id: string,
    init: ((ctx?: InitContext) => Promise<Def<Parameters, Result>>) | Def<Parameters, Result>,
  ): Info<Parameters, Result>
}
```

### 2.2 工具包装器 — 自动验证与截断

`Tool.define()` 内部通过 `wrap()` 函数添加了两层保护：

```typescript
// packages/opencode/src/tool/tool.ts (lines 58-94)
function wrap<Parameters extends z.ZodType, Result extends Metadata>(
  id: string,
  init: ((ctx?: InitContext) => Promise<Def<Parameters, Result>>) | Def<Parameters, Result>,
) {
  return async (initCtx?: InitContext) => {
    const toolInfo = init instanceof Function ? await init(initCtx) : { ...init }
    const execute = toolInfo.execute

    // 第一层：Zod 参数验证
    toolInfo.execute = async (args, ctx) => {
      try {
        toolInfo.parameters.parse(args)
      } catch (error) {
        throw new Error(toolInfo.formatValidationError?.(error as z.ZodError)
          ?? formatZodError(error as z.ZodError))
      }

      // 第二层：输出截断
      const result = await execute(args, ctx)
      const truncated = await Truncate.output(result.output, {}, initCtx?.agent)

      return {
        ...result,
        output: truncated.content,
        metadata: {
          ...result.metadata,
          truncated: truncated.truncated,
          ...(truncated.truncated && { outputPath: truncated.outputPath }),
        },
      }
    }

    return toolInfo
  }
}
```

所有工具的执行都会自动经过：**参数验证 → 执行 → 输出截断** 三步。

---

## 3. 工具注册中心

### 3.1 Service 接口

```typescript
// packages/opencode/src/tool/registry.ts
export namespace ToolRegistry {
  export interface Interface {
    readonly ids: () => Effect.Effect<string[]>
    readonly named: { task: Tool.Info; read: Tool.Info }  // 关键工具引用
    readonly tools: (
      model: { providerID: ProviderID; modelID: ModelID },
      agent?: Agent.Info,
    ) => Effect.Effect<(Tool.Def & { id: string })[]>
  }
}
```

### 3.2 内置工具注册

```typescript
// packages/opencode/src/tool/registry.ts (lines 76-156)
export const layer: Layer.Layer<Service, never, ...> = Layer.effect(
  Service,
  Effect.gen(function* () {
    const invalid = yield* build(InvalidTool)
    const ask = yield* build(QuestionTool)
    const bash = yield* build(BashTool)
    const read = yield* build(ReadTool)
    const glob = yield* build(GlobTool)
    const grep = yield* build(GrepTool)
    const edit = yield* build(EditTool)
    const write = yield* build(WriteTool)
    const task = yield* build(TaskTool)
    const fetch = yield* build(WebFetchTool)
    const todo = yield* build(TodoWriteTool)
    const search = yield* build(WebSearchTool)
    const code = yield* build(CodeSearchTool)
    const skill = yield* build(SkillTool)
    const patch = yield* build(ApplyPatchTool)
    const lsp = yield* build(LspTool)
    const batch = yield* build(BatchTool)
    const plan = yield* build(PlanExitTool)
    // ...
  }),
)
```

### 3.3 动态工具加载

```typescript
// packages/opencode/src/tool/registry.ts (lines 114-124)
const customToolsDir = path.join(ctx.cwd, 'tool')
const toolsDir = path.join(ctx.cwd, 'tools')

for (const dir of [customToolsDir, toolsDir]) {
  const entries = yield* fs.pipe(
    fsys.list(dir),
    Stream.map(entry => entry.name),
    Stream.filter(name => name.endsWith('.js') || name.endsWith('.ts')),
    Stream.map(name => path.join(dir, name)),
    Stream.map(path => import(path)),
    Stream.map(m => m.default as Tool.Info),
    Stream.runList(),
  )
}
```

从项目目录的 `tool/` 或 `tools/` 子目录动态加载自定义工具。

### 3.4 工具过滤

工具根据模型和特性标志动态过滤：

```typescript
// packages/opencode/src/tool/registry.ts
if (codesearch?.enabled !== false && !isO1OrO3) {
  tools.set('codesearch', yield* build(CodeSearchTool))
}
if (model.api.id.includes('gpt-4') || model.api.id.includes('o1') || model.api.id.includes('o3')) {
  tools.set('apply_patch', yield* build(ApplyPatchTool))
}
if (config.experimental?.batch_tool) {
  tools.set('batch', yield* build(BatchTool))
}
```

---

## 4. 核心工具详解

### 4.1 Read 工具

```typescript
// packages/opencode/src/tool/read.ts
export const ReadTool = Tool.define("read", async (ctx) => {
  const parameters = z.object({
    filePath: z.string().describe("Absolute path to the file"),
    offset: z.number().optional().describe("Line number to start (1-indexed)"),
    limit: z.number().optional().default(2000).describe("Max lines to read"),
  })

  return {
    description: "Read file contents",
    parameters,
    async execute(params, ctx) {
      // 1. 追踪文件读取时间
      yield* FileTime.track(params.filePath, "read")
      // 2. 检查二进制文件
      // 3. 处理目录列表
      // 4. LSP 预热
      // 5. 返回内容（自动截断）
    },
  }
})
```

特点：
- 智能错误提示（拼写纠正 "Did you mean?"）
- 支持图片和 PDF（base64 附件）
- LSP 文件预热
- 默认 2000 行上限，50KB 字节上限

### 4.2 Edit 工具 — 高级模糊匹配

这是 OpenCode 最复杂的工具。它实现了 **9 种回退策略**：

```typescript
// packages/opencode/src/tool/edit.ts (lines 637-647)
const replacers: Replacer[] = [
  new SimpleReplacer(),           // 1. 精确匹配
  new LineTrimmedReplacer(),       // 2. 去除空白后匹配
  new BlockAnchorReplacer(),       // 3. 首尾行锚定 + Levenshtein 距离
  new WhitespaceNormalizedReplacer(), // 4. 空白归一化
  new IndentationFlexibleReplacer(), // 5. 灵活缩进
  new EscapeNormalizedReplacer(), // 6. 转义序列处理
  new TrimmedBoundaryReplacer(),   // 7. 边界去空白
  new ContextAwareReplacer(),     // 8. 上下文感知
  new MultiOccurrenceReplacer(),  // 9. 多处匹配
]
```

**BlockAnchorReplacer** 最为强大：

```typescript
// packages/opencode/src/tool/edit.ts (lines 240-373)
class BlockAnchorReplacer {
  // 1. 取 oldString 的第一行和最后一行作为锚点
  // 2. 在文件中搜索包含这两个锚点的块
  // 3. 使用 Levenshtein 距离计算相似度
  // 4. 单一候选 → 相似度 > 0.0 即匹配
  // 5. 多候选 → 相似度 > 0.3 才匹配
  // 6. 返回最相似的块进行替换
}
```

这解决了用户复制代码时代码缩进/空白不一致的问题。

### 4.3 Bash 工具 — Shell AST 解析

```typescript
// packages/opencode/src/tool/bash.ts
export const BashTool = Tool.define("bash", async (ctx) => {
  const parameters = z.object({
    command: z.string().describe("Shell command to execute"),
    timeout: z.number().optional().describe("Timeout in milliseconds"),
    workdir: z.string().optional().describe("Working directory"),
    description: z.string().describe("Short description (5-10 words)"),
  })

  return {
    description,
    parameters,
    async execute(params, ctx) {
      // 1. 解析命令 AST（tree-sitter）
      // 2. 提取文件路径参数
      // 3. 检查外部目录访问权限
      // 4. 执行命令（CrossSpawnSpawner）
      // 5. 收集输出和诊断信息
    },
  }
})
```

Bash 工具使用 **tree-sitter** 解析 shell 脚本，提取命令中的文件路径，用于权限检查。这比 Claude Code 的正则匹配更加精确。

### 4.4 Task 工具 — 子 Agent 派生

```typescript
// packages/opencode/src/tool/task.ts
export const TaskTool = Tool.define("task", async (ctx) => {
  const parameters = z.object({
    description: z.string().describe("Short (3-5 words) description"),
    prompt: z.string().describe("The task for the agent"),
    subagent_type: z.string().describe("Agent type to use"),
    task_id: z.string().optional().describe("Resume a previous task"),
    command: z.string().optional(),
  })

  return {
    description,
    parameters,
    async execute(params, ctx) {
      // 1. 获取指定的子 Agent
      const agent = yield* Agent.get(params.subagent_type)

      // 2. 创建新会话（子会话）
      const session = yield* Session.create({
        parentID: ctx.sessionID,  // 关联父会话
        title: params.description,
        permission: [...],
      })

      // 3. 在子会话中运行 Agent
      const result = yield* SessionPrompt.prompt({
        messageID,
        sessionID: session.id,
        model: { ... },
        agent: agent.name,
        tools: { ... },
        parts: promptParts,
      })

      // 4. 返回结果（含 task_id 可恢复）
      return {
        title: params.description,
        metadata: { sessionId: session.id },
        output: `task_id: ${session.id}\n\n<task_result>${text}</task_result>`,
      }
    },
  }
})
```

---

## 5. 工具在 LLM 中的集成

### 5.1 工具解析

```typescript
// packages/opencode/src/session/llm.ts (lines 388-474)
const resolveTools = Effect.fn("SessionPrompt.resolveTools")(function* (input) {
  const tools: Record<string, AITool> = {}

  // 1. 构建工具执行上下文
  const context = (args: any, options: ToolExecutionOptions): Tool.Context => ({
    sessionID: input.session.id,
    abort: options.abortSignal!,
    messageID: input.processor.message.id,
    callID: options.toolCallId,
    extra: { model: input.model, bypassAgentCheck: input.bypassAgentCheck },
    agent: input.agent.name,
    messages: input.messages,
    metadata: (val) => Effect.runPromise(/* 更新 part 状态 */),
    ask: (req) => Effect.runPromise(/* 请求权限 */),
  })

  // 2. 转换原生工具为 AI SDK 工具
  for (const item of yield* registry.tools({ modelID, providerID }, input.agent)) {
    const schema = ProviderTransform.schema(input.model, z.toJSONSchema(item.parameters))
    tools[item.id] = tool({
      id: item.id,
      description: item.description,
      inputSchema: jsonSchema(schema),
      execute(args, options) {
        return Effect.runPromise(
          Effect.gen(function* () {
            yield* plugin.trigger("tool.execute.before", { tool: item.id, ... }, { args })
            const result = yield* Effect.promise(() => item.execute(args, ctx))
            yield* plugin.trigger("tool.execute.after", { tool: item.id, ... }, result)
            return result
          }),
        )
      },
    })
  }

  // 3. 转换 MCP 工具
  for (const [key, item] of Object.entries(yield* mcp.tools())) {
    const schema = yield* Effect.promise(() => Promise.resolve(asSchema(item.inputSchema).jsonSchema))
    // ... MCP 工具转换
  }

  return tools
})
```

### 5.2 权限过滤

```typescript
// packages/opencode/src/session/llm.ts (lines 339-345)
function resolveTools(input: Pick<...>) {
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

## 6. 输出截断服务

```typescript
// packages/opencode/src/tool/truncate.ts
export namespace Truncate {
  // Max Lines: 2000
  // Max Bytes: 50 KB
  // Retention: 7 天

  export async function output(
    text: string,
    options: { maxLines?: number; maxBytes?: number },
    agent?: string,
  ): Promise<{ content: string; truncated: boolean; outputPath?: string }> {
    const lines = text.split('\n')
    const maxLines = options.maxLines ?? 2000
    const maxBytes = options.maxBytes ?? 50 * 1024

    if (lines.length <= maxLines && text.length <= maxBytes) {
      return { content: text, truncated: false }
    }

    // 写入临时文件
    const truncatedContent = lines.slice(0, maxLines).join('\n')
    const outputPath = path.join(TRUNCATION_DIR, `${Date.now()}.txt`)
    await fsys.writeFile(outputPath, truncatedContent)

    return {
      content: `Output truncated. Full content written to: ${outputPath}\n\n${truncatedContent.slice(0, maxBytes)}...`,
      truncated: true,
      outputPath,
    }
  }
}
```

---

## 7. 与 Claude Code 对比

| 维度 | Claude Code | OpenCode |
|------|------------|----------|
| 工具定义 | TypeScript 对象 | Effect Service + Zod |
| 参数验证 | 内联验证 | Zod Schema 自动验证 |
| 编辑模糊匹配 | 无（精确匹配） | 9 种回退策略 + Levenshtein |
| Bash 解析 | 正则提取路径 | tree-sitter AST 解析 |
| 工具包装 | 手动包装 | `wrap()` 统一包装 |
| 权限检查 | `canUseTool()` 函数 | `Permission.disabled()` 过滤 |
| 工具发现 | 文件系统 glob | Effect Service 动态加载 |
| 子 Agent | `AgentTool` 直接调用 | `TaskTool` 创建子会话 |

---

*文档版本：v1.0 | 更新：2026-04-06*
