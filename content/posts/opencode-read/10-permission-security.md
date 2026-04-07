---
title: "权限与安全 — Ruleset 驱动的细粒度控制"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 权限与安全 — Ruleset 驱动的细粒度控制

> 源码路径：`/mnt/e/code/cc/opencode/packages/opencode/src/permission/`
> 核心文件：`permission/index.ts`
> 技术栈：Effect + Pattern 匹配 + Zod

---

## 1. 概述

OpenCode 的权限系统采用 **Ruleset 数组** 模型，相比 Claude Code 的 `PermissionMode` 枚举更加精细化。每个 Agent 持有一个权限规则数组，通过 pattern 匹配实现对工具、文件路径和命令的细粒度控制。

---

## 2. Ruleset 类型

```typescript
// packages/opencode/src/permission/index.ts
export namespace Permission {
  export const Rule = z.object({
    permission: z.string(),   // 权限类型 (read, edit, bash, task, etc.)
    pattern: z.string(),      // 匹配模式 (glob 风格)
    action: z.enum(["allow", "deny", "ask"]),
  })

  export type Ruleset = z.infer<typeof Rule>[]
}
```

### 权限类型

```typescript
// 文件操作
"read"        // 读取文件
"edit"        // 编辑文件
"write"       // 写入文件
"glob"        // 文件模式匹配
"grep"        // 搜索
"ls"          // 目录列表
"bash"        // Shell 命令

// 网络操作
"webfetch"    // HTTP 请求
"websearch"   // 网络搜索
"codesearch"  // 代码搜索

// 任务管理
"task"        // 派生子 Agent
"todowrite"   // Todo 列表修改
"question"    // 用户交互

// 其他
"external_directory"  // 访问工作区外目录
"lsp"         // LSP 操作
"doom_loop"   // 循环检测豁免
```

---

## 3. Agent 默认权限

### 3.1 build Agent

```typescript
{
  permission: [
    // 全部允许
    { permission: "read", pattern: "*", action: "allow" },
    { permission: "edit", pattern: "*", action: "allow" },
    { permission: "bash", pattern: "*", action: "allow" },
    { permission: "glob", pattern: "*", action: "allow" },
    { permission: "grep", pattern: "*", action: "allow" },
    // 外部目录白名单
    { permission: "external_directory", pattern: "~/**", action: "allow" },
    { permission: "external_directory", pattern: "/tmp/**", action: "allow" },
  ]
}
```

### 3.2 plan Agent

```typescript
{
  permission: [
    { permission: "read", pattern: "*", action: "allow" },
    { permission: "glob", pattern: "*", action: "allow" },
    { permission: "grep", pattern: "*", action: "allow" },
    // 仅允许编辑计划文件
    { permission: "edit", pattern: ".opencode/plans/*", action: "allow" },
    // 禁止所有写入和执行
    { permission: "write", pattern: "*", action: "deny" },
    { permission: "bash", pattern: "*", action: "deny" },
  ]
}
```

### 3.3 explore Agent

```typescript
{
  permission: [
    { permission: "read", pattern: "*", action: "allow" },
    { permission: "glob", pattern: "*", action: "allow" },
    { permission: "grep", pattern: "*", action: "allow" },
    // 禁止所有修改操作
    { permission: "edit", pattern: "*", action: "deny" },
    { permission: "bash", pattern: "*", action: "deny" },
    { permission: "write", pattern: "*", action: "deny" },
  ]
}
```

---

## 4. 权限评估

```typescript
// packages/opencode/src/permission/index.ts
export function evaluate(
  permission: string,   // 权限类型 (如 "bash")
  pattern: string,      // 资源路径 (如 "/path/to/file")
  ruleset: Ruleset,     // 规则数组
): { action: "allow" | "deny" | "ask"; rules: Rule[] } {
  // 1. 找出匹配该权限类型的所有规则
  const matched = ruleset.filter(rule =>
    rule.permission === permission &&
    matchesGlob(pattern, rule.pattern)
  )

  // 2. 优先级: deny > ask > allow
  //    最后一条匹配的规则决定结果
  if (matched.length === 0) {
    return { action: "ask", rules: [] }
  }

  const last = matched[matched.length - 1]!
  return { action: last.action, rules: matched }
}
```

**优先级规则**：
- deny 规则总是优先
- ask 作为中间状态
- allow 作为默认行为
- 规则按顺序匹配，最后一条生效

---

## 5. 权限请求流程

```typescript
// 工具执行时请求权限
async function executeTool(tool: Tool.Def, args: unknown, ctx: Tool.Context) {
  const { action } = Permission.evaluate(
    tool.id,
    extractFilePaths(args),  // 从参数中提取文件路径
    ctx.agent.permission,
  )

  switch (action) {
    case "allow":
      return tool.execute(args, ctx)

    case "deny":
      throw new Permission.DeniedError({
        tool: tool.id,
        args,
        rules: matchedRules,
      })

    case "ask": {
      // 暂停执行，等待用户确认
      await ctx.ask({
        permission: tool.id,
        patterns: extractFilePaths(args),
        always: [],  // 用户的 "Always" 选择
        sessionID: ctx.sessionID,
        metadata: { tool: tool.id },
      })

      // 用户确认后重新评估
      const { action: retryAction } = Permission.evaluate(...)
      if (retryAction === "deny") {
        throw new Permission.DeniedError(...)
      }
      return tool.execute(args, ctx)
    }
  }
}
```

### 用户响应选项

```typescript
interface Request {
  permission: string
  patterns: string[]
  always?: string[]  // "Always allow" 选项
}

// 用户响应
type Replied = {
  type: "once"    // 临时允许一次
} | {
  type: "always"  // 永久添加到白名单
} | {
  type: "reject"  // 拒绝
}
```

---

## 6. 配置覆盖

```jsonc
{
  "permission": {
    "read": [
      { "pattern": "*", "action": "allow" },
      { "pattern": "**/secrets/**", "action": "deny" }
    ],
    "bash": [
      { "pattern": "rm -rf /**", "action": "deny" },
      { "pattern": "git **", "action": "allow" },
      { "pattern": "npm **", "action": "allow" }
    ]
  }
}
```

---

## 7. 与 Claude Code 对比

| 维度 | Claude Code | OpenCode |
|------|------------|----------|
| 权限模型 | PermissionMode 枚举 (browse/auto/bypass/plan) | Ruleset 数组 + Pattern 匹配 |
| 文件级别 | 无 | Glob Pattern 精细控制 |
| 命令级别 | 无 | Bash 命令白名单 |
| 外部目录 | 单一开关 | 白名单 Pattern |
| Doom loop | 简单计数 | 权限豁免机制 |
| 永久授权 | 无 | "Always" 选项 |
| 企业 MDM | 无 | Managed Config 覆盖 |
| 配置位置 | config.toml | JSONC + config.json |

### 7.1 关键差异

**Claude Code** 的权限模型是**粗粒度**的：
- `browse` — 只读
- `auto` — 自动批准安全操作
- `bypass` — 无限制
- `plan` — 仅计划模式

**OpenCode** 的 Ruleset 是**细粒度**的：
- 每种权限都可以有独立的 Pattern 规则
- 可以针对特定文件/目录设置不同行为
- 支持 deny/ask/allow 三级控制
- 用户可以选择 "Always" 永久授权

---

## 8. 核心文件索引

| 文件 | 职责 |
|------|------|
| `packages/opencode/src/permission/index.ts` | Ruleset 定义 + 评估逻辑 |

---

*文档版本：v1.0 | 更新：2026-04-06*
