---
title: "多智能体系统 — 11 个专业化 Agent 与分类编排"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 多智能体系统 — 11 个专业化 Agent 与分类编排

> 源码路径：`/mnt/e/code/cc/omo-code/src/agents/`
> 核心文件：`sisyphus/`, `builtin-agents.ts`, `dynamic-agent-prompt-builder.ts`, `types.ts`
> 技术栈：Zod + Effect + 动态提示词生成

---

## 1. 概述

Oh-My-OpenAgent 的核心创新在于 **11 个专业化 Agent**，每个 Agent 针对特定任务类型和模型进行了优化。这些 Agent 不是简单的提示词变体，而是拥有独立的工具集、模型选择策略和委托规则。

```
Sisyphus (主协调者)
  ├── Prometheus (战略规划) ──→ Hephaestus (深度实现)
  ├── Atlas (Todo 编排) ──→ Sisyphus-Junior (类别执行)
  ├── Oracle (架构顾问) ──→ Momus (计划审查)
  ├── Librarian (文档搜索) ──→ Explore (快速 grep)
  └── Metis (计划差距分析)
```

---

## 2. Agent 总览

| Agent | 模式 | 主模型 | 角色 | 成本 |
|-------|------|--------|------|------|
| **Sisyphus** | primary | claude-opus-4-6 | 主协调者，全局调度 | 昂贵 |
| **Hephaestus** | subagent | gpt-5.4 | 自主深工作者，端到端完成 | 昂贵 |
| **Prometheus** | primary | claude-opus-4-6 | 战略规划者，访谈模式 | 昂贵 |
| **Atlas** | primary | claude-sonnet-4-6 | Todo 编排者，任务执行 | 中等 |
| **Oracle** | subagent | gpt-5.4 | 架构顾问，只读咨询 | 昂贵 |
| **Librarian** | subagent | minimax-m2.7 | 文档/OSS 搜索，多仓库分析 | 便宜 |
| **Explore** | subagent | grok-code-fast-1 | 快速 grep，上下文理解 | 便宜 |
| **Metis** | subagent | claude-opus-4-6 | 计划差距分析，预规划 | 昂贵 |
| **Momus** | subagent | gpt-5.4 | 计划审查者，吹毛求疵评审 | 昂贵 |
| **Sisyphus-Junior** | subagent | 类别决定 | 轻量级委托，类别路由 | 变化 |
| **Multimodal-Looker** | subagent | gpt-5.4 | 视觉内容分析，多模态 | 昂贵 |

---

## 3. Agent 元数据

```typescript
// src/agents/types.ts
export interface AgentDefinition {
  // 基本信息
  name: string
  description: string
  category: AgentCategory      // 'orchestration' | 'advisor' | 'exploration' | 'execution'
  cost: 'free' | 'cheap' | 'expensive'

  // 何时使用
  useWhen?: string[]           // 触发条件
  avoidWhen?: string[]         // 避免条件
  triggers?: string[]           // 关键词触发

  // 模型配置
  model: ModelConfig
  fallback_models?: string[]    // 回退模型链

  // 工具约束
  allowed_tools?: string[]
  disallowed_tools?: string[]

  // 提示词
  prompt: AgentPromptConfig
  variants?: Record<string, AgentPromptConfig>

  // 行为
  maxTurns?: number
  canDelegate?: boolean         // 能否委托给其他 Agent
}
```

---

## 4. Sisyphus — 主协调者

Sisyphus 是整个系统的中枢，采用 **Sisyphus 隐喻**（每日推石上山的希腊神话）来定义其行为哲学。

### 4.1 提示词结构

```typescript
// src/agents/sisyphus/prompt.ts
export const SISYPHUS_PROMPT = `
You are Sisyphus, a master software engineer in the SF Bay Area.

Your job is not to do everything yourself — it's to coordinate a team of specialists.

## Your Team

- **Hephaestus**: Deep worker for goal-oriented execution
- **Prometheus**: Strategic planner for complex features
- **Atlas**: Todo orchestrator for multi-step tasks
- **Oracle**: Architecture consultant for high-level decisions
- **Librarian**: Documentation and code search specialist
- **Explore**: Fast codebase grep for understanding
- **Metis**: Plan gap analyzer for pre-planning
- **Momus**: Plan reviewer for critical feedback

## When to Delegate

| Situation | Agent |
|-----------|-------|
| Need deep implementation | Hephaestus |
| Need strategic planning | Prometheus |
| Need todo management | Atlas |
| Need architecture advice | Oracle |
| Need documentation lookup | Librarian |
| Need fast grep | Explore |
| Need plan gap analysis | Metis |
| Need plan review | Momus |

## Your Principles

1. **Delegate, don't do** — If a specialist can do it better, delegate
2. **Classify first** — Understand the request before choosing an approach
3. **Preserve context** — Pass enough context to workers
4. **Verify results** — Don't assume work is done until verified
5. **Iterate** — Small steps, early feedback

## Intent Gate (Phase 0)

Before classifying, detect the true intent:
- Is this a quick question? → Answer directly
- Is this a complex task? → Classify and delegate
- Is this a planning request? → Invoke Prometheus
- Is this a code exploration? → Invoke Explore
`
```

### 4.2 动态提示构建

```typescript
// src/agents/dynamic-agent-prompt-builder.ts
export function buildSisyphusPrompt(config: Config): string {
  const sections: string[] = []

  // 1. 核心角色定义
  sections.push(SISYPHUS_PROMPT)

  // 2. 动态触发节（基于可用 Agent）
  sections.push(buildKeyTriggersSection(config))

  // 3. 工具选择表（按成本排序）
  sections.push(buildToolSelectionTable(config))

  // 4. 委托表（领域 → Agent 映射）
  sections.push(buildDelegationTable(config))

  // 5. 类别技能委托指南
  sections.push(buildCategorySkillsDelegationGuide(config))

  // 6. Agent 特定指南
  if (config.agents.explore?.enabled !== false) {
    sections.push(buildExploreSection(config))
  }
  if (config.agents.librarian?.enabled !== false) {
    sections.push(buildLibrarianSection(config))
  }
  if (config.agents.oracle?.enabled !== false) {
    sections.push(buildOracleSection(config))
  }

  return sections.filter(Boolean).join('\n\n')
}
```

### 4.3 模型解析 (4 步)

```typescript
// Sisyphus 的模型选择遵循 4 步解析
function resolveAgentModel(config: Config, agent: AgentDefinition): string {
  // 1. 配置覆盖
  if (config.agents[agent.name]?.model) {
    return config.agents[agent.name].model
  }

  // 2. 类别默认模型
  if (config.categories[agent.category]?.default_model) {
    return config.categories[agent.category].default_model
  }

  // 3. Provider 回退链
  for (const fallback of agent.fallback_models ?? []) {
    if (isProviderAvailable(fallback)) {
      return fallback
    }
  }

  // 4. 系统默认
  return config.model ?? 'claude-sonnet-4-6'
}
```

---

## 5. 内置 Agent 分类

### 5.1 协调类 (Orchestration)

**Prometheus** — 战略规划者：
- 6 阶段访谈模式
- 需求提取和验证
- 生成结构化计划

**Atlas** — Todo 编排者：
- 使用 `task()` 工具管理任务
- 动态 Agent 选择
- 类别路由

### 5.2 顾问类 (Advisor)

**Oracle** — 架构顾问：
- 只读咨询模式
- XML 结构化输出
- 决策框架：务实简约

**Metis** — 计划差距分析：
- 意图分类：重构/构建/协作/架构/研究
- 识别隐藏意图
- AI slop 模式检测

**Momus** — 计划审查者：
- 参考验证（文件存在、行号有效）
- 可执行性检查
- 只报告阻塞性问题

### 5.3 探索类 (Exploration)

**Librarian** — 多仓库文档搜索：
- 4 阶段请求分类：概念 → 实现 → 上下文 → 综合
- 集成 Context7、GitHub CLI、Web 搜索

**Explore** — 快速 grep：
- 并行多角度搜索
- 结构化结果输出

### 5.4 执行类 (Execution)

**Hephaestus** — 自主深工作者：
- GPT-5.4 原生优化
- 端到端任务完成
- 不提前停止

**Sisyphus-Junior** — 轻量委托者：
- 基于类别的模型选择
- 最小化上下文开销

---

## 6. 分类系统

```typescript
// 8 个内置分类
export const BUILTIN_CATEGORIES = {
  quick: {
    description: 'Quick questions and simple edits',
    default_model: 'claude-haiku-4',
    agents: ['explore'],
  },
  code: {
    description: 'Code implementation tasks',
    default_model: 'claude-sonnet-4-6',
    agents: ['hephaestus', 'sisyphus-junior'],
  },
  frontend: {
    description: 'Frontend development',
    default_model: 'claude-sonnet-4-6',
    agents: ['hephaestus', 'sisyphus-junior'],
  },
  backend: {
    description: 'Backend and API development',
    default_model: 'claude-sonnet-4-6',
    agents: ['hephaestus', 'sisyphus-junior'],
  },
  infra: {
    description: 'Infrastructure and DevOps',
    default_model: 'gpt-5.4',
    agents: ['oracle', 'hephaestus'],
  },
  data: {
    description: 'Data engineering',
    default_model: 'claude-opus-4-6',
    agents: ['oracle', 'hephaestus'],
  },
  security: {
    description: 'Security analysis',
    default_model: 'gpt-5.4',
    agents: ['oracle', 'momus'],
  },
  quality: {
    description: 'Testing and code quality',
    default_model: 'claude-sonnet-4-6',
    agents: ['momus', 'hephaestus'],
  },
}
```

---

## 7. 与 Claude Code / OpenCode 对比

| 维度 | Claude Code | OpenCode | Oh-My-OpenAgent |
|------|-------------|----------|-----------------|
| Agent 数量 | 7 | 7 | **11** |
| 主协调 | Coordinator Mode (Prompt) | Session Tree | **Sisyphus (专用 Agent)** |
| 子 Agent | Worker (同质) | 独立 Session | **专业化分工** |
| 模型选择 | 单一模型 | 单一模型 | **多模型编排** |
| 委托机制 | SendMessageTool | TaskTool | **DelegateTask + CallOmoAgent** |
| 分类路由 | 无 | 无 | **8 个内置分类** |
| 模型回退 | 无 | 无 | **Fallback 链 + 断路器** |

---

*文档版本：v1.0 | 更新：2026-04-06*
