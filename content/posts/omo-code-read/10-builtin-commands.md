---
title: "内置命令 — 8 个复杂工作流命令"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 内置命令 — 8 个复杂工作流命令

> 源码路径：`/mnt/e/code/cc/omo-code/src/features/builtin-commands/`
> 核心文件：`start-work.ts`, `refactor.ts`, `handoff.ts`, `init-deep.ts`
> 技术栈：YAML + Shell Scripting + 多 Agent 协调

---

## 1. 概述

Oh-My-OpenAgent 的内置命令（Built-in Commands）不是简单的工具调用，而是**复杂工作流**的入口点。每个命令内部协调多个 Agent、Tool 和 Hook 来完成完整的开发任务。

---

## 2. 命令总览

| 命令 | 功能 | 涉及 Agent | 阶段数 |
|------|------|-----------|--------|
| `/start-work` | 工作会话初始化 | Sisyphus | 5 |
| `/refactor` | 智能重构 | Prometheus + Hephaestus + Momus | 6 |
| `/handoff` | 会话交接 | Atlas | 4 |
| `/init-deep` | 深度调查模式 | Librarian + Explore | 3 |
| `/ralph-loop` | 循环检测恢复 | Sisyphus | 2 |
| `/remove-ai-slops` | AI slop 清理 | Momus | 3 |
| `/stop-continuation` | 会话停止保留状态 | Atlas | 2 |
| `/agent-browser` | 浏览器自动化 | Playwright Skill | 1 |

---

## 3. /start-work — 工作会话初始化

最复杂的命令之一，初始化完整的工作环境：

```typescript
// src/features/builtin-commands/start-work.ts
export async function startWorkCommand(
  args: { ticket?: string; branch?: string; plan?: string },
  ctx: ToolContext,
): Promise<CommandResult> {
  const steps: WorkStep[] = []

  // === 阶段 1: 工作树创建 ===
  if (args.branch) {
    // 创建 git worktree 支持并行开发
    await ctx.run(`git worktree add work/${args.branch}`)
    ctx.directory = path.join(ctx.directory, 'work', args.branch)
    steps.push({ phase: 'worktree', status: 'completed' })
  }

  // === 阶段 2: Boulder 状态初始化 ===
  // Sisyphus 的" boulder" = 当前任务的状态
  await ctx.run(`mkdir -p .opencode/boulder`)
  await writeFile('.opencode/boulder/state.json', {
    ticket: args.ticket,
    branch: args.branch,
    steps: [],
    startedAt: Date.now(),
  })
  steps.push({ phase: 'boulder', status: 'completed' })

  // === 阶段 3: 任务分解 ===
  const tasks = await ctx.invokeAgent('prometheus', {
    prompt: `Break down this work into granular sub-steps:\n${args.plan ?? 'Implement the requested feature'}`,
  })

  // 写入 Todo 列表
  for (const task of parseTasks(tasks)) {
    await ctx.createTodo(task)
  }
  steps.push({ phase: 'breakdown', status: 'completed', count: tasks.length })

  // === 阶段 4: 上下文加载 ===
  const contextFiles = discoverContextFiles(ctx.directory)
  for (const file of contextFiles) {
    await ctx.injectContext(file)
  }
  steps.push({ phase: 'context', status: 'completed' })

  // === 阶段 5: 启动 Sisyphus ===
  await ctx.invokeAgent('sisyphus', {
    prompt: `Start working on: ${args.plan ?? args.ticket}`,
    continueFrom: 'boulder',
  })

  return {
    summary: `Started work on ${args.ticket ?? 'task'} with ${tasks.length} sub-tasks`,
    steps,
    worktree: args.branch ? `work/${args.branch}` : undefined,
  }
}
```

---

## 4. /refactor — 智能重构

6 阶段重构工作流：

```typescript
// src/features/builtin-commands/refactor.ts
export async function refactorCommand(
  args: { target: string; pattern?: string },
  ctx: ToolContext,
): Promise<CommandResult> {
  const phases: RefactorPhase[] = []

  // === 阶段 1: 并行探索 ===
  // 启动多个 Explore Agent 并行分析
  const explorations = await Promise.all([
    ctx.invokeAgent('explore', { prompt: `Analyze ${args.target} structure` }),
    ctx.invokeAgent('explore', { prompt: `Find callers of ${args.target}` }),
    ctx.invokeAgent('explore', { prompt: `Find tests for ${args.target}` }),
  ])
  phases.push({ name: 'exploration', agents: 3, completed: true })

  // === 阶段 2: 构建代码地图 ===
  const codemap = await ctx.runTool('ast_grep_search', {
    pattern: '$FUNC()',
    language: detectLanguage(args.target),
  })
  phases.push({ name: 'codemap', files: codemap.length, completed: true })

  // === 阶段 3: 测试覆盖评估 ===
  const coverage = await ctx.runTool('lsp_diagnostics', {
    file: args.target,
  })
  phases.push({ name: 'coverage', hasTests: coverage.testFiles > 0, completed: true })

  // === 阶段 4: 计划 Agent 审查 ===
  const plan = await ctx.invokeAgent('momus', {
    prompt: `Review refactoring plan for ${args.target}. ` +
      `Check: files exist, line numbers valid, tests cover the refactored areas.`,
  })

  if (plan.hasBlockingIssues) {
    return { result: 'blocked', issues: plan.issues }
  }
  phases.push({ name: 'review', result: 'approved', completed: true })

  // === 阶段 5: 分步实现 ===
  for (const step of plan.steps) {
    await ctx.invokeAgent('hephaestus', {
      prompt: `Refactoring step: ${step.description}\n` +
        `Focus on: ${step.focus}\n` +
        `Pattern: ${args.pattern}`,
    })

    // LSP 验证
    const diagnostics = await ctx.runTool('lsp_diagnostics', { file: args.target })
    if (diagnostics.errors > 0) {
      phases.push({ name: 'verify', result: 'failed', error: 'LSP errors' })
      break
    }
  }
  phases.push({ name: 'implementation', steps: plan.steps.length, completed: true })

  // === 阶段 6: 最终验证 ===
  const finalCheck = await ctx.runTool('lsp_diagnostics', { file: args.target })
  phases.push({ name: 'final', errors: finalCheck.errors, completed: true })

  return { phases, result: 'success' }
}
```

---

## 5. /handoff — 会话交接

```typescript
// src/features/builtin-commands/handoff.ts
export async function handoffCommand(
  args: { to?: string; message?: string },
  ctx: ToolContext,
): Promise<CommandResult> {
  // 1. 收集当前状态
  const state = {
    todos: await ctx.getTodos(),
    session: await ctx.getSessionInfo(),
    boulder: await readFile('.opencode/boulder/state.json'),
    recentFiles: await ctx.getModifiedFiles(),
    pendingTasks: await ctx.getPendingTasks(),
  }

  // 2. 打包交接上下文
  const handoverMessage = [
    `# Handoff from ${ctx.user.name}`,
    `## Current State`,
    `- Session: ${state.session.id}`,
    `- Modified files: ${state.recentFiles.join(', ')}`,
    `## Todo Progress`,
    state.todos.map(t => `- [${t.done ? 'x' : ' '}] ${t.text}`).join('\n'),
    `## Message`,
    args.message ?? 'Please continue from here.',
  ].join('\n')

  // 3. 如果指定了接收者，创建新会话
  if (args.to) {
    await ctx.invokeAgent('sisyphus', {
      prompt: handoverMessage,
      agent: args.to,
    })
  }

  // 4. 保存交接快照
  await writeFile(`.opencode/handoffs/${Date.now()}.md`, handoverMessage)

  // 5. 归档当前会话
  await ctx.archiveSession()

  return {
    summary: `Handoff completed${args.to ? ` to ${args.to}` : ''}`,
    state,
  }
}
```

---

## 6. 与 OpenCode 命令对比

| 维度 | OpenCode | Oh-My-OpenAgent |
|------|----------|-----------------|
| 内置命令数 | 2 (init, review) | **8** |
| 命令复杂度 | 简单引导 | **多阶段工作流** |
| 多 Agent 协调 | 无 | **跨 Agent 协调** |
| 状态持久化 | 无 | **Boulder State** |
| Worktree 支持 | 无 | **并行开发** |
| 会话交接 | 无 | **/handoff 完整上下文** |

---

## 7. 核心文件索引

| 文件 | 职责 |
|------|------|
| `src/features/builtin-commands/start-work.ts` | 工作会话初始化 |
| `src/features/builtin-commands/refactor.ts` | 智能重构 |
| `src/features/builtin-commands/handoff.ts` | 会话交接 |
| `src/features/builtin-commands/init-deep.ts` | 深度调查 |

---

*文档版本：v1.0 | 更新：2026-04-06*
