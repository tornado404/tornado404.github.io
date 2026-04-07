---
title: "技能系统 — 8 个内置 Skill 与四层发现机制"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 技能系统 — 8 个内置 Skill 与四层发现机制

> 源码路径：`/mnt/e/code/cc/omo-code/src/features/opencode-skill-loader/`, `src/features/builtin-skills/`
> 核心文件：`skill-loader.ts`, `skill-registry.ts`
> 技术栈：YAML Frontmatter + Template Resolution + MCP Integration

---

## 1. 概述

Oh-My-OpenAgent 的 Skill 系统是其最具创新性的功能之一。它将**技能定义**（SKILL.md）与 **MCP 服务器**深度集成，每个 Skill 可以嵌入自己的 MCP 工具，按需启动，作用域精确。

```
Skill = 技能定义(SKILL.md) + MCP 工具集 + 模板解析
```

---

## 2. 四层发现机制

```typescript
// src/features/opencode-skill-loader/skill-loader.ts
export const SKILL_SCOPES = {
  project: {                    // 优先级最高
    dir: '.opencode/skills/',
    description: 'Project-specific skills',
  },
  opencode: {                   // OpenCode 内置
    dir: '${HOME}/.opencode/skills/',
    description: 'User-installed skills',
  },
  global: {                     // 全局技能
    dir: '${HOME}/.config/opencode/skills/',
    description: 'Global skills',
  },
  omo: {                        // OMO 内置
    embedded: true,
    description: 'Oh-My-OpenAgent built-in skills',
  },
} as const
```

---

## 3. SKILL.md 格式

```markdown
---
name: git-master
description: Advanced git workflows and branching strategies
author: oh-my-opencode
version: 1.0.0
model: claude-sonnet-4-6  # 推荐模型
tags: [git, vcs, workflow]
---

# Git Master Skill

You are a git expert. Your job is to help with:

## Workflows

- Feature branch workflow
- Gitflow
- Trunk-based development
- Hotfix procedures

## Commands

```bash
# Create feature branch
git checkout -b feature/$TICKET

# Interactive rebase
git rebase -i HEAD~$N

# Bisect for bug finding
git bisect start
git bisect bad
git bisect good $COMMIT
```

## When to Use

Use this skill when the user mentions:
- git, branch, commit, rebase, merge
- PR, pull request, code review
- stash, cherry-pick, bisect

## Template Variables

- `$TICKET` - Issue/ticket number
- `$BRANCH` - Current branch name
- `$COMMIT` - Commit hash
```

---

## 4. 内置 Skills (8 个)

| Skill | 行数 | 功能 |
|-------|------|------|
| **git-master** | 1,111 | 高级 Git 工作流、分支策略 |
| **playwright** | 312 | 浏览器自动化测试 |
| **agent-browser** | — | Agent 专用浏览器 |
| **dev-browser** | — | 开发浏览器工具 |
| **frontend-ui-ux** | 79 | UI/UX 设计指导 |
| **review-work** | — | 代码审查自动化 |
| **ai-slop-remover** | — | AI 生成代码检测清理 |
| **websearch** | — | (内置 MCP) 网络搜索 |

---

## 5. Skill 加载流程

```typescript
// src/features/opencode-skill-loader/skill-loader.ts
export class SkillLoader {
  async load(name: string, scope: SkillScope): Promise<Skill> {
    // 1. 定位 SKILL.md
    const skillPath = this.resolvePath(scope, name)
    if (!await exists(skillPath)) {
      throw new SkillNotFoundError(name, scope)
    }

    // 2. 解析 Frontmatter
    const { frontmatter, content } = await parseSkillMarkdown(skillPath)

    // 3. 验证 Schema
    const validated = SkillSchema.parse(frontmatter)

    // 4. 检查模型兼容性
    if (validated.model && !isModelCompatible(validated.model, ctx.model)) {
      throw new ModelIncompatibleError(name, validated.model)
    }

    // 5. 解析模板变量
    const resolvedContent = this.resolveTemplates(content, ctx.variables)

    return {
      name: validated.name,
      description: validated.description,
      content: resolvedContent,
      metadata: validated,
    }
  }

  async invoke(name: string, variables?: Record<string, string>): Promise<SkillResult> {
    const skill = await this.load(name)

    // 注入为系统提示词
    const message = await this.injectAsSystemMessage(skill, variables)

    // 执行
    const result = await this.agent.execute(message)

    return result
  }
}
```

---

## 6. Skill-MCP 集成

```typescript
// src/features/skill-mcp-manager/SkillMcpManager.ts
export class SkillMcpManager {
  // Skill 按需启动自己的 MCP 服务器
  private activeMcps: Map<string, MCPClient> = new Map()

  async loadSkillMcp(skillName: string): Promise<void> {
    const skill = await this.skillLoader.load(skillName)

    if (!skill.metadata.mcp) return

    // 1. 解析 MCP 配置
    const mcpConfig = skill.metadata.mcp

    // 2. 启动 Skill 专属 MCP
    const client = await MCPClient.connect({
      type: mcpConfig.type,  // 'stdio' | 'http'
      command: mcpConfig.command,
      args: mcpConfig.args,
      env: {
        ...process.env,
        ...mcpConfig.env,
        SKILL_CONTEXT: JSON.stringify({
          skill: skillName,
          variables: skill.variables,
        }),
      },
    })

    // 3. 注册 Skill 专属工具
    const tools = await client.listTools()
    for (const tool of tools) {
      const scopedName = `skill_${skillName}_${tool.name}`
      this.toolRegistry.register(scopedName, tool)
    }

    this.activeMcps.set(skillName, client)
  }

  async unloadSkillMcp(skillName: string): Promise<void> {
    const client = this.activeMcps.get(skillName)
    if (client) {
      await client.disconnect()
      this.activeMcps.delete(skillName)

      // 注销 Skill 工具
      const tools = getToolsForSkill(skillName)
      for (const tool of tools) {
        this.toolRegistry.unregister(tool.name)
      }
    }
  }
}
```

---

## 7. 模板解析

```typescript
// src/features/opencode-skill-loader/template-resolver.ts
export class TemplateResolver {
  resolve(content: string, variables: Record<string, string>): string {
    return content
      // $VAR 语法
      .replace(/\$(\w+)/g, (_, name) =>
        variables[name] ?? this.promptFor(name) ?? `{{${name}}}`)
      // ${{ expression }} 语法
      .replace(/\${{\s*(.+?)\s*}}/g, (_, expr) =>
        this.evaluateExpression(expr, variables))
  }

  // 交互式变量提示
  private async promptFor(name: string): Promise<string | null> {
    // 用于在 Skill 调用时收集缺失的变量
  }
}
```

---

## 8. 与 OpenCode 对比

| 维度 | OpenCode | Oh-My-OpenAgent |
|------|----------|-----------------|
| Skill 内置 | 无 | **8 个内置 Skill** |
| Skill 定义 | 无 | **SKILL.md + Frontmatter** |
| Skill 加载 | 无 | **四层发现机制** |
| Skill-MCP | 无 | **Skill 专属 MCP 按需启动** |
| 模板解析 | 无 | **$VAR / ${{ expr }}** |
| 模型过滤 | 无 | **per-Skill 模型要求** |

---

*文档版本：v1.0 | 更新：2026-04-06*
