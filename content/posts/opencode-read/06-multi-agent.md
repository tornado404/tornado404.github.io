---
title: "多智能体协作 — Task 工具与子 Agent 派生"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 多智能体协作 — Task 工具与子 Agent 派生

> 源码路径：`/mnt/e/code/cc/opencode/packages/opencode/src/tool/task.ts`
> 核心文件：`task.ts`, `agent/agent.ts`
> 技术栈：Effect + Session + Agent Service

---

## 1. 概述

OpenCode 的多智能体协作采用**会话隔离模式**：每个子 Agent 运行在独立的 Session 中，通过 `parentID` 关联父会话。这种设计与 Claude Code 的 Coordinator Mode 有本质区别。

```
┌─────────────────────────────────────────────────────────────┐
│                    Session Hierarchy                         │
│                                                             │
│  Parent Session (主会话)                                      │
│    │                                                         │
│    ├── parentID: null                                       │
│    │                                                         │
│    ├── TaskTool(task_type="explore") ──→ Child Session A    │
│    │                                       parentID: Parent   │
│    │                                                         │
│    ├── TaskTool(task_type="general") ──→ Child Session B     │
│    │                                       parentID: Parent   │
│    │                                                         │
│    └── User interaction...                                   │
│                                                             │
│  Result aggregation: Task ID → Resume or get result         │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 内置 Agent 类型

```typescript
// packages/opencode/src/agent/agent.ts (lines 107-233)

// 1. build — 主执行 Agent（默认）
const BUILD_AGENT: Agent.Info = {
  name: "build",
  mode: "primary",
  permission: [
    { permission: "read", pattern: "*", action: "allow" },
    { permission: "edit", pattern: "*", action: "allow" },
    { permission: "bash", pattern: "*", action: "allow" },
    // ...
  ],
}

// 2. plan — 计划模式 Agent（只读）
const PLAN_AGENT: Agent.Info = {
  name: "plan",
  mode: "primary",
  permission: [
    { permission: "read", pattern: "*", action: "allow" },
    { permission: "edit", pattern: ".opencode/plans/*", action: "allow" },
    // edit/write/bash 全被 deny
  ],
}

// 3. general — 通用研究 Agent
const GENERAL_AGENT: Agent.Info = {
  name: "general",
  mode: "subagent",
  permission: [
    { permission: "read", pattern: "*", action: "allow" },
    { permission: "edit", pattern: "*", action: "allow" },
    { permission: "bash", pattern: "*", action: "allow" },
    // todowrite 被 deny
  ],
}

// 4. explore — 快速探索 Agent（只读）
const EXPLORE_AGENT: Agent.Info = {
  name: "explore",
  mode: "subagent",
  permission: [
    { permission: "read", pattern: "*", action: "allow" },
    { permission: "glob", pattern: "*", action: "allow" },
    { permission: "grep", pattern: "*", action: "allow" },
  ],
}

// 5. compaction — 内部压缩 Agent（隐藏）
// 6. title — 会话标题生成（隐藏）
// 7. summary — 会话总结生成（隐藏）
```

**Agent Mode** 决定使用场景：
- `primary` — 可作为默认 Agent（build, plan）
- `subagent` — 仅用于 TaskTool 派生（explore, general）
- `all` — 既可默认也可派生

---

## 3. Task 工具实现

```typescript
// packages/opencode/src/tool/task.ts
export const TaskTool = Tool.define("task", async (ctx) => {
  const agents = await Agent.list().then(x =>
    x.filter(a => a.mode !== "primary")  // 仅子 Agent
  )

  const parameters = z.object({
    description: z.string().describe("Short description (3-5 words)"),
    prompt: z.string().describe("The task for the agent"),
    subagent_type: z.string().describe("Agent type to use"),
    task_id: z.string().optional().describe("Resume a previous task"),
    command: z.string().optional(),
  })

  return {
    description,
    parameters,
    async execute(params, ctx) {
      // === 恢复已有任务 ===
      if (params.task_id) {
        const session = yield* Session.load(params.task_id)
        const result = yield* SessionPrompt.prompt({
          messageID: MessageID.ascending(),
          sessionID: session.id,
          // ... 继续执行
        })
        return formatResult(result)
      }

      // === 创建新任务 ===
      // 1. 获取子 Agent
      const agent = yield* Agent.get(params.subagent_type)

      // 2. 权限检查
      const hasTaskPermission = agent.permission.some(r => r.permission === "task")
      const hasTodoWritePermission = agent.permission.some(r => r.permission === "todowrite")

      // 3. 创建子会话
      const session = yield* Session.create({
        parentID: ctx.sessionID,  // 关键：关联父会话
        title: params.description + ` (@${agent.name} subagent)`,
        permission: [
          // 禁用递归 task 调用（防止无限派生）
          ...(hasTaskPermission ? [] : [{ permission: "task", pattern: "*", action: "deny" }]),
          // todowrite 权限继承
          ...(hasTodoWritePermission ? [] : [{ permission: "todowrite", pattern: "*", action: "deny" }]),
        ],
      })

      // 4. 在子会话中执行
      const messageID = MessageID.ascending()
      const result = yield* SessionPrompt.prompt({
        messageID,
        sessionID: session.id,
        model: { modelID: model.modelID, providerID: model.providerID },
        agent: agent.name,
        tools: {
          todowrite: hasTodoWritePermission ? undefined : false,
          task: hasTaskPermission ? undefined : false,
        },
        parts: promptParts,
      })

      // 5. 格式化结果
      const text = result.parts.findLast(x => x.type === "text")?.text ?? ""

      return {
        title: params.description,
        metadata: { sessionId: session.id, model },
        output: [
          `task_id: ${session.id} (for resuming to continue this task if needed)`,
          "",
          "<task_result>",
          text,
          "</task_result>",
        ].join("\n"),
      }
    },
  }
})
```

---

## 4. 动态 Agent 生成

OpenCode 支持从描述动态生成 Agent：

```typescript
// packages/opencode/src/agent/agent.ts (lines 329-391)
generate: Effect.fn("Agent.generate")(function* (input: {
  description: string
  model?: { providerID: ProviderID; modelID: ModelID }
}) {
  // 1. 使用 generate.txt 提示词
  const system = [PROMPT_GENERATE]

  // 2. 调用 AI 生成 Agent 配置
  const result = yield* Effect.promise(() =>
    generateObject({
      temperature: 0.3,
      messages: [
        { role: "system", content: system.join("\n") },
        {
          role: "user",
          content: `Create an agent configuration: "${input.description}"`,
        },
      ],
      model: language,
      schema: z.object({
        identifier: z.string(),
        whenToUse: z.string(),
        systemPrompt: z.string(),
      }),
    }).then(r => r.object),
  )

  // 3. 返回生成的配置
  return {
    identifier: result.identifier,
    whenToUse: result.whenToUse,
    systemPrompt: result.systemPrompt,
  }
})
```

---

## 5. 与 Claude Code 对比

| 维度 | Claude Code | OpenCode |
|------|------------|----------|
| 子 Agent 创建 | `AgentTool(subagent_type="worker")` | `TaskTool(subagent_type="explore")` |
| 执行隔离 | 共享父上下文 + Fork | 独立 Session |
| 结果传递 | `<task-notification>` XML | `task_result` 文本块 |
| 任务恢复 | `SendMessageTool` | `task_id` 参数继续 |
| 任务停止 | `TaskStopTool` | 无内置（需终止 Session） |
| 动态生成 | 无 | `Agent.generate()` |
| 协作模式 | 主从协调（Coordinator） | 会话树（Session Hierarchy） |
| 并行性 | 显式并行 Launch | 隐式（各 Session 独立） |

### 5.1 关键差异

**Claude Code Coordinator**：
- 主控 Agent 通过系统提示词驱动
- Worker 通过 `<task-notification>` 异步通知主控
- 主控负责决策和结果整合
- 并行性由主控显式控制

**OpenCode Session Tree**：
- 子 Agent 在独立 Session 中运行
- 父会话通过 `task_id` 可恢复子会话
- 无中心协调者，父会话被动等待结果
- 并行性由父会话决定（可同时调用多个 TaskTool）

---

## 6. 核心文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `packages/opencode/src/agent/agent.ts` | ~420 | Agent 定义 + Service + 动态生成 |
| `packages/opencode/src/tool/task.ts` | ~166 | TaskTool 实现 |
| `packages/opencode/src/agent/prompt/generate.txt` | 4,994 字节 | Agent 动态生成提示词 |

---

*文档版本：v1.0 | 更新：2026-04-06*
