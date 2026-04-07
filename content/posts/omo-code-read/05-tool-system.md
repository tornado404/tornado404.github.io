---
title: "工具系统 — 26 个扩展工具"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 工具系统 — 26 个扩展工具

> 源码路径：`/mnt/e/code/cc/omo-code/src/tools/`
> 核心文件：`plugin/tool-registry.ts`, `delegate-task/`, `background-task/`, `hashline-edit/`, `lsp/`
> 技术栈：Zod + Effect + LSP + AST-Grep

---

## 1. 概述

Oh-My-OpenAgent 在 OpenCode 原生工具基础上增加了大量扩展工具，重点在于**专业化的代码操作**（LSP、AST 重写）和**多 Agent 编排**（委托、后台任务）。

```
工具分类：
├── LSP 工具 (6 个)     — 完整的语言服务器协议支持
├── 后台任务 (2 个)     — 并发控制 + 后台管理
├── 委托任务 (1 个)     — 分类路由 + Agent 选择
├── Agent 调用 (1 个)   — 直接调用 OMO Agent
├── Skill (2 个)       — Skill 加载 + Skill-MCP
├── 视觉分析 (1 个)    — Multimodal 图像分析
├── 会话管理 (4 个)    — 会话列表/读取/搜索/信息
├── 高级编辑 (2 个)    — Hashline edit + 增强 edit
├── 搜索工具 (4 个)    — grep + glob + ast_grep x2
└── Bash (1 个)        — interactive bash + tmux
```

---

## 2. LSP 工具 (6 个)

LSP 工具提供完整的 IDE 级别的代码智能：

```typescript
// src/tools/lsp/index.ts

// LSP_GotoDefinition — 跳转到定义
export const lsp_goto_definition = Tool.define('lsp_goto_definition', {
  description: 'Jump to the definition of a symbol',
  parameters: z.object({
    symbol: z.string().describe('Symbol name to find'),
    file: z.string().optional().describe('File to search in'),
  }),
  execute: async (args, ctx) => {
    const lsp = await getLspForFile(args.file ?? ctx.currentFile)
    const result = await lsp.gotoDefinition({
      textDocument: { uri: fileToUri(args.file) },
      position: await findSymbolPosition(args.file, args.symbol),
    })
    return {
      title: `Definition of ${args.symbol}`,
      output: formatLocation(result),
      metadata: { uri: result.uri, line: result.range.start.line },
    }
  },
})

// LSP_FindReferences — 查找引用
export const lsp_find_references = Tool.define('lsp_find_references', {
  description: 'Find all references to a symbol',
  parameters: z.object({
    symbol: z.string(),
    file: z.string().optional(),
    includeDeclaration: z.boolean().default(true),
  }),
})

// LSP_Symbols — 文档符号树
export const lsp_symbols = Tool.define('lsp_symbols', {
  description: 'List all symbols in a document',
  parameters: z.object({
    file: z.string().describe('File to list symbols from'),
    kind: z.enum(['class', 'function', 'variable', 'all']).default('all'),
  }),
})

// LSP_Diagnostics — 诊断信息
export const lsp_diagnostics = Tool.define('lsp_diagnostics', {
  description: 'Get LSP diagnostics (errors, warnings, lint issues)',
  parameters: z.object({
    file: z.string().optional().describe('File to diagnose (default: current)'),
    severity: z.enum(['error', 'warning', 'info']).optional(),
  }),
})

// LSP_PrepareRename + LSP_Rename — 重命名重构
export const lsp_prepare_rename = Tool.define('lsp_prepare_rename', { ... })
export const lsp_rename = Tool.define('lsp_rename', {
  description: 'Rename a symbol across the project',
  parameters: z.object({
    oldName: z.string(),
    newName: z.string(),
    file: z.string().optional(),
  }),
})
```

---

## 3. 后台任务管理

### 3.1 BackgroundManager

```typescript
// src/features/background-agent/BackgroundManager.ts
export class BackgroundManager {
  // 并发限制: 每个模型/Provider 5 个并发
  private concurrencyManager: ConcurrencyManager

  // 轮询间隔: 3 秒
  private pollInterval = 3_000

  // 稳定性检测: 10 秒内结果不变视为完成
  private stabilityWindow = 10_000

  // 断路器: 失败自动恢复
  private circuitBreaker: CircuitBreaker

  async spawn(
    config: BackgroundTaskConfig,
    ctx: ToolContext,
  ): Promise<TaskHandle> {
    // 1. 检查并发限制
    const model = config.model ?? ctx.defaultModel
    if (this.concurrencyManager.atLimit(model)) {
      throw new ConcurrentLimitError(
        `Too many background tasks for ${model}. ` +
        `Wait for existing tasks to complete.`
      )
    }

    // 2. 创建任务
    const task: BackgroundTask = {
      id: generateTaskId(),
      status: 'pending',
      config,
      startedAt: undefined,
      result: undefined,
      agentId: config.agent ?? 'sisyphus',
    }

    // 3. 注册到管理器
    this.tasks.set(task.id, task)

    // 4. 启动后台执行
    this.executeBackground(task, ctx)

    return {
      task_id: task.id,
      status: 'pending',
      message: `Background task started: ${config.description}`,
    }
  }

  private async executeBackground(task: BackgroundTask, ctx: ToolContext) {
    task.status = 'running'
    task.startedAt = Date.now()

    // 通过 SDK 调用 OpenCode API
    const result = await ctx.client.session.prompt({
      prompt: task.config.prompt,
      agent: task.agentId,
      model: task.config.model,
    })

    task.result = result
    task.status = result.error ? 'error' : 'completed'
  }
}
```

### 3.2 后台任务工具

```typescript
// src/tools/background-task/
export const background_output = Tool.define('background_output', {
  description: 'Get output from a background task',
  parameters: z.object({
    task_id: z.string().describe('Task ID to retrieve'),
    wait: z.boolean().optional().describe('Wait for completion if still running'),
    timeout: z.number().optional().default(10_000),
  }),
})

export const background_cancel = Tool.define('background_cancel', {
  description: 'Cancel a running background task',
  parameters: z.object({
    task_id: z.string().describe('Task ID to cancel'),
    reason: z.string().optional(),
  }),
})
```

---

## 4. 委托任务工具

```typescript
// src/tools/delegate-task/
export const delegate_task = Tool.define('delegate_task', {
  description: 'Delegate a task to a specialized agent by category or directly',
  parameters: z.object({
    description: z.string().describe('Short description (3-5 words)'),
    prompt: z.string().describe('The task for the agent'),
    category: z.enum([
      'quick', 'code', 'frontend', 'backend',
      'infra', 'data', 'security', 'quality',
    ]).optional().describe('Task category for routing'),
    agent: z.string().optional().describe('Direct agent name (overrides category)'),
    task_id: z.string().optional().describe('Resume a previous task'),
  }),
  execute: async (params, ctx) => {
    // 1. 解析目标 Agent
    const targetAgent = params.agent
      ?? resolveCategoryAgent(params.category ?? inferCategory(params.prompt))

    // 2. 模型选择
    const model = resolveModelForAgent(targetAgent, ctx.config)

    // 3. 创建子会话
    const session = await ctx.client.session.create({
      parentID: ctx.sessionID,
      title: params.description,
      agent: targetAgent,
      model,
    })

    // 4. 执行
    const result = await session.prompt(params.prompt)

    // 5. 返回结果
    return {
      title: params.description,
      metadata: { sessionId: session.id, agent: targetAgent, model },
      output: formatTaskResult(result),
    }
  },
})
```

---

## 5. Hashline Edit — 哈希锚定编辑

这是 Oh-My-OpenAgent 的创新工具，解决精确编辑的问题：

```typescript
// src/tools/hashline-edit/
export const hashline_edit = Tool.define('hashline_edit', {
  description: 'Edit files using content hashes as anchors (more reliable than line numbers)',

  parameters: z.object({
    filePath: z.string(),
    oldString: z.string().describe('Content to replace'),
    newString: z.string().describe('Replacement content'),
    algorithm: z.enum(['sha256', 'md5']).default('sha256'),
  }),

  execute: async (params, ctx) => {
    // 1. 读取文件
    const content = await readFile(params.filePath)

    // 2. 计算 oldString 的 SHA256 哈希
    const hash = crypto
      .createHash(params.algorithm)
      .update(params.oldString)
      .digest('hex')

    // 3. 找到包含该哈希的行（用于调试输出）
    const lineHash = computeLineHashes(content)
    const matchingLine = lineHash.find(h => h.hash === hash)

    if (!matchingLine) {
      // 4. 降级到普通编辑
      return fallbackToEdit(params, ctx)
    }

    // 5. 执行替换
    const newContent = content.replace(params.oldString, params.newString)

    // 6. 验证新内容
    const newHash = crypto
      .createHash(params.algorithm)
      .update(params.newString)
      .digest('hex')

    // 7. 写入
    await writeFile(params.filePath, newContent)

    return {
      title: `Edited ${path.basename(params.filePath)}`,
      output: `Hashline edit successful.\nAlgorithm: ${params.algorithm}\nOld hash: ${hash}\nNew hash: ${newHash}`,
      metadata: { hash, line: matchingLine.line, algorithm: params.algorithm },
    }
  },
})
```

---

## 6. AST Grep 工具

```typescript
// src/tools/ast-grep/

// AST_Grep_Search — AST 模式搜索
export const ast_grep_search = Tool.define('ast_grep_search', {
  description: 'Search code using AST patterns (language-aware, not regex)',
  parameters: z.object({
    pattern: z.string().describe('AST pattern (e.g., "$FUNC call($ARG)")'),
    language: z.enum(['javascript', 'typescript', 'python', 'go', 'rust', 'java']),
    file: z.string().optional().describe('File or directory to search'),
    globals: z.record(z.string()).optional().describe('Pattern variables'),
  }),
  execute: async (params, ctx) => {
    // 使用 @ast-grep/napi
    const sg = await import('@ast-grep/napi')

    const results = await sg.search({
      rule: params.pattern,
      language: params.language,
      paths: [params.file ?? ctx.directory],
      globals: params.globals,
    })

    return {
      title: `AST search: ${params.pattern}`,
      output: formatASTResults(results),
      metadata: { count: results.length },
    }
  },
})

// AST_Grep_Replace — AST 模式替换
export const ast_grep_replace = Tool.define('ast_grep_replace', {
  description: 'Replace code using AST patterns with transformations',
  parameters: z.object({
    pattern: z.string(),
    replacement: z.string().describe('Replacement AST pattern with $NEW syntax'),
    language: z.enum(['javascript', 'typescript', 'python', 'go', 'rust', 'java']),
    file: z.string(),
  }),
})
```

---

## 7. 会话管理工具

```typescript
// src/tools/session-manager/
export const session_list = Tool.define('session_list', {
  description: 'List recent sessions in the project',
  parameters: z.object({
    limit: z.number().default(10),
    filter: z.string().optional(),
  }),
})

export const session_read = Tool.define('session_read', {
  description: 'Read a previous session transcript',
  parameters: z.object({
    session_id: z.string().describe('Session ID to read'),
    offset: z.number().optional(),
    limit: z.number().optional(),
  }),
})

export const session_search = Tool.define('session_search', {
  description: 'Search across all sessions for content',
  parameters: z.object({
    query: z.string(),
    sessions: z.array(z.string()).optional(),
  }),
})

export const session_info = Tool.define('session_info', {
  description: 'Get metadata about a session',
  parameters: z.object({
    session_id: z.string().optional().describe('Default: current session'),
  }),
})
```

---

## 8. 与 OpenCode 对比

| 维度 | OpenCode | Oh-My-OpenAgent |
|------|----------|-----------------|
| 工具总数 | 26 | **26 原生 + 扩展** |
| LSP 工具 | 无 | **6 个完整 LSP** |
| AST 重写 | 无 | **ast_grep 搜索/替换** |
| 后台任务 | Session 子任务 | **BackgroundManager + 并发控制** |
| 委托方式 | TaskTool | **DelegateTask + 分类路由** |
| Hashline | 无 | **SHA256/MD5 锚定编辑** |
| 视觉分析 | 无 | **look_at 多模态** |
| 会话管理 | 基础 | **完整 CRUD + 搜索** |

---

*文档版本：v1.0 | 更新：2026-04-06*
