---
title: "后台任务管理 — 并发控制与 Tmux 集成"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 后台任务管理 — 并发控制与 Tmux 集成

> 源码路径：`/mnt/e/code/cc/omo-code/src/features/background-agent/`, `src/features/tmux-subagent/`
> 核心文件：`BackgroundManager.ts`, `TmuxSessionManager.ts`, `ConcurrencyManager.ts`
> 技术栈：Promise + Polling + Tmux CLI

---

## 1. 概述

后台任务管理是 Oh-My-OpenAgent 实现并行多 Agent 的核心能力。它包含两部分：**BackgroundManager**（任务生命周期）和 **TmuxSessionManager**（实时可视化）。

---

## 2. BackgroundManager

### 2.1 核心架构

```typescript
// src/features/background-agent/BackgroundManager.ts
export class BackgroundManager {
  private tasks: Map<string, BackgroundTask> = new Map()
  private concurrencyManager: ConcurrencyManager
  private circuitBreaker: CircuitBreaker

  constructor(config: BackgroundTaskConfig) {
    this.concurrencyManager = new ConcurrencyManager({
      perModel: config.max_per_model ?? 5,
      perProvider: config.max_per_provider ?? 5,
      global: config.max_global ?? 20,
    })

    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: config.circuit_breaker?.failure_threshold ?? 3,
      resetTimeout: config.circuit_breaker?.reset_timeout ?? 60_000,
    })
  }
}
```

### 2.2 任务状态机

```
pending → running → completed
                   → error
                   → cancelled
                   → interrupt
```

### 2.3 并发控制

```typescript
// src/features/background-agent/ConcurrencyManager.ts
export class ConcurrencyManager {
  checkLimit(model: string, provider: string): boolean {
    const modelCount = this.countByModel(model)
    const providerCount = this.countByProvider(provider)
    const globalCount = this.countGlobal()

    return (
      modelCount < this.config.perModel &&
      providerCount < this.config.perProvider &&
      globalCount < this.config.global
    )
  }

  enqueue(task: BackgroundTask): void {
    const queue = this.queues.get(task.model)
    queue?.push(task)
  }

  dequeue(model: string): BackgroundTask | undefined {
    return this.queues.get(model)?.shift()
  }
}
```

### 2.4 轮询与稳定性检测

```typescript
// src/features/background-agent/polling.ts
export async function pollTask(
  taskId: string,
  interval = 3_000,
  stabilityWindow = 10_000,
): Promise<TaskResult> {
  let lastResult: string | undefined
  let stableSince: number | undefined

  while (true) {
    const task = await getTaskStatus(taskId)

    if (task.status === 'completed' || task.status === 'error') {
      return task
    }

    if (task.result === lastResult) {
      if (!stableSince) {
        stableSince = Date.now()
      } else if (Date.now() - stableSince > stabilityWindow) {
        // 结果稳定超过 10 秒，视为完成
        return { ...task, status: 'completed' }
      }
    } else {
      stableSince = undefined
      lastResult = task.result
    }

    await sleep(interval)
  }
}
```

---

## 3. Tmux 集成

### 3.1 TmuxSessionManager

```typescript
// src/features/tmux-subagent/TmuxSessionManager.ts
export class TmuxSessionManager {
  private sessions: Map<string, TmuxPane> = new Map()

  async createPane(config: TmuxConfig): Promise<TmuxPane> {
    // 1. 创建 tmux window
    await this.runTmux(['new-window', '-n', config.name])

    // 2. 分割窗格（Grid 布局用于多 Agent 可视化）
    if (config.layout === 'grid') {
      await this.splitGrid(config.name, config.agents?.length ?? 2)
    }

    // 3. 在每个 pane 中启动 Agent
    for (let i = 0; i < config.agents.length; i++) {
      await this.runInPane(
        config.name,
        i,
        `opencode --agent ${config.agents[i]}`,
      )
    }

    const pane: TmuxPane = {
      id: config.name,
      agents: config.agents,
      layout: config.layout,
      createdAt: Date.now(),
    }

    this.sessions.set(config.name, pane)
    return pane
  }

  async sendInput(paneId: string, paneIndex: number, input: string): Promise<void> {
    // 向特定 pane 发送输入
    await this.runTmux([
      'send-keys',
      '-t',
      `${paneId}:${paneIndex}`,
      input,
      'Enter',
    ])
  }

  async capturePane(paneId: string, paneIndex: number): Promise<string> {
    const output = await this.runTmux([
      'capture-pane',
      '-t',
      `${paneId}:${paneIndex}`,
      '-p',
    ])
    return output
  }
}
```

### 3.2 Grid 布局

```typescript
// 多 Agent 网格布局 (2x2 示例)
async splitGrid(sessionName: string, count: number): Promise<void> {
  const cols = Math.ceil(Math.sqrt(count))
  const rows = Math.ceil(count / cols)

  // 垂直分割 (cols - 1 次)
  for (let i = 1; i < cols; i++) {
    await this.runTmux(['split-window', '-h', '-t', `${sessionName}:0`])
  }

  // 水平分割 (rows - 1) * cols 次
  for (let r = 1; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      await this.runTmux(['split-window', '-v', '-t', `${sessionName}:${c}`])
    }
  }

  // 均分布局
  await this.runTmux(['select-layout', '-t', sessionName, 'even-horizontal'])
}
```

---

## 4. 后台任务与 Tmux 联动

```typescript
// src/features/background-agent/background-tmux.ts
export async function launchBackgroundWithTmux(
  config: BackgroundTaskConfig,
  tmuxConfig: TmuxConfig,
  ctx: ToolContext,
): Promise<{ taskId: string; paneId: string }> {
  // 1. 创建 Tmux pane
  const pane = await tmuxManager.createPane({
    ...tmuxConfig,
    agents: [config.agent ?? 'sisyphus'],
  })

  // 2. 启动后台任务
  const task = await backgroundManager.spawn({
    ...config,
    tmuxPaneId: pane.id,
  })

  // 3. 监听完成事件
  backgroundManager.on('task:completed', async ({ taskId, result }) => {
    // 在 Tmux pane 中显示结果
    await tmuxManager.sendInput(pane.id, 0, `\n\n[TASK COMPLETED]\n${result}\n`)
    // 保持 pane 打开 5 秒
    await sleep(5_000)
    // 关闭 pane
    await tmuxManager.closePane(pane.id)
  })

  return { taskId: task.id, paneId: pane.id }
}
```

---

## 5. 与 OpenCode / Claude Code 对比

| 维度 | Claude Code | OpenCode | Oh-My-OpenAgent |
|------|-------------|----------|-----------------|
| 后台任务 | run_in_background | TaskTool 子会话 | **BackgroundManager** |
| 并发控制 | 无 | 无 | **per-model/provider 限制** |
| 断路器 | 无 | 无 | **自动失败恢复** |
| 稳定性检测 | 无 | 无 | **10s 结果不变视为完成** |
| Tmux 集成 | 无 | 无 | **实时可视化** |
| 轮询间隔 | — | — | **3s 可配置** |

---

*文档版本：v1.0 | 更新：2026-04-06*
