---
title: "命令系统 — Slash 命令与动态加载"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 命令系统 — Slash 命令与动态加载

> 源码路径：`/mnt/e/code/cc/opencode/packages/opencode/src/command/`
> 核心文件：`command/index.ts`
> 技术栈：Effect + Markdown Frontmatter + MCP Prompts

---

## 1. 概述

OpenCode 的命令（Command）系统与 Claude Code 的 Slash 命令设计思路相似，但实现细节有所不同：

- **Claude Code**：命令定义在 `src/tools/` 下，通过 `buildTool()` 统一注册
- **OpenCode**：命令定义在 `.opencode/command/*.md` 文件中，通过 Markdown frontmatter 声明式定义

这种设计让命令更像"配置文件"而非"代码"，用户可以零代码添加自定义命令。

---

## 2. 命令定义类型

```typescript
// packages/opencode/src/command/index.ts
export namespace Command {
  export const Info = z.object({
    name: z.string(),
    description: z.string().optional(),
    agent: z.string().optional(),     // 可选的 Agent 覆盖
    model: z.string().optional(),     // 可选的模型覆盖
    source: z.enum(["command", "mcp", "skill"]),
    template: z.string().or(z.promise(z.string())),
    subtask: z.boolean().optional(),
    hints: z.array(z.string()),       // 模板参数提示 ($1, $2 等)
  })

  export interface Interface {
    readonly get: (name: string) => Effect.Effect<Command.Info>
    readonly list: () => Effect.Effect<Command.Info[]>
  }
}
```

---

## 3. 内置命令

### 3.1 init 命令

引导用户创建 `AGENTS.md` 文件：

```typescript
// 引导用户完成 AGENTS.md 配置流程
```

### 3.2 review 命令

Review 代码变更（commit / branch / PR）：

```bash
/opencode review commit      # Review 当前 commit
/opencode review branch      # Review 当前分支
/opencode review pr          # Review PR
```

---

## 4. 自定义命令加载

### 4.1 文件格式

`.opencode/command/` 下的 Markdown 文件：

```markdown
---
name: mycommand
description: A custom command for something
agent: explore  # 可选：指定使用的 Agent
model: sonnet    # 可选：指定使用的模型
---

Do something with $1 and $2.
```

- `name` — 命令名称（文件名就是默认值）
- `description` — 命令描述
- `agent` — 可选，覆盖使用的 Agent
- `model` — 可选，覆盖使用的模型
- `template` — 命令内容，支持 `$1`, `$2` 等占位符

### 4.2 参数提示提取

```typescript
// 从模板中提取 $N 参数提示
const template = await readFile(templatePath)
const hints: string[] = []
const paramRegex = /\$(\d+)/g
let match
while ((match = paramRegex.exec(template)) !== null) {
  const num = parseInt(match[1])
  if (!hints.includes(`$${num}`)) {
    hints.push(`$${num}`)
  }
}
```

提取出的 hints 用于在用户输入时提供补全提示。

### 4.3 MCP Prompts 暴露为命令

MCP 服务器定义的 prompts 也可以作为命令暴露：

```typescript
// MCP 配置中定义的 prompts
const mcpPrompts = yield* mcp.prompts()
// 转换为 Command.Info
for (const [key, prompt] of Object.entries(mcpPrompts)) {
  commands.set(key, {
    name: key,
    source: "mcp",
    template: prompt.template,
    hints: extractHints(prompt.template),
  })
}
```

---

## 5. 与 Claude Code 对比

| 维度 | Claude Code | OpenCode |
|------|------------|----------|
| 定义方式 | TypeScript `buildTool()` | Markdown frontmatter |
| 自定义命令 | 需要写代码 | 写 Markdown 文件即可 |
| 参数提示 | 硬编码 | 自动从 `$N` 占位符提取 |
| Agent 覆盖 | 内置 agentType | frontmatter `agent` 字段 |
| MCP 集成 | 无直接集成 | MCP prompts 直接暴露为命令 |

---

*文档版本：v1.0 | 更新：2026-04-06*
