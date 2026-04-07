---
title: "配置系统 — Zod v4 Schema 与多层合并"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 配置系统 — Zod v4 Schema 与多层合并

> 源码路径：`/mnt/e/code/cc/omo-code/src/config/`, `src/plugin-config.ts`
> 核心文件：`schema/oh-my-opencode-config.ts`, `schema/*.ts`
> 技术栈：Zod v4 + JSONC + Deep Merge

---

## 1. 概述

Oh-My-OpenAgent 的配置系统极为精细，使用 **Zod v4** 定义了 27 个 Schema 文件，支持多层配置合并、partial 解析降级和自动迁移。

---

## 2. 配置层级

```
Project (.opencode/oh-my-opencode.jsonc)   ← 最高优先级
  ↓ Deep Merge
User (~/.config/opencode/oh-my-opencode.jsonc)
  ↓ Deep Merge
Defaults (代码内置)
```

---

## 3. 配置 Schema (27 个文件)

```typescript
// src/config/schema/oh-my-opencode-config.ts
export const OhMyOpenCodeConfig = z.object({
  // === 禁用开关 (Set 并集) ===
  disabled_agents: z.array(z.string()).optional(),
  disabled_tools: z.array(z.string()).optional(),
  disabled_mcps: z.array(z.string()).optional(),
  disabled_hooks: z.array(z.string()).optional(),
  disabled_skills: z.array(z.string()).optional(),
  disabled_commands: z.array(z.string()).optional(),

  // === MCP 环境白名单 ===
  mcp_env_allowlist: z.array(z.string()).optional(),

  // === 功能开关 ===
  hashline_edit: z.boolean().optional(),
  model_fallback: z.boolean().optional(),
  runtime_fallback: z.boolean().optional(),

  // === Agent 配置 (覆盖内置) ===
  agents: z.record(z.string(), AgentOverride).optional(),

  // === 分类配置 ===
  categories: z.record(z.string(), CategoryConfig).optional(),

  // === Claude Code 兼容性 ===
  claude_code: ClaudeCodeCompat.optional(),

  // === Sisyphus 协调者 ===
  sisyphus: SisyphusConfig.optional(),

  // === 后台任务 ===
  background_task: BackgroundTaskConfig.optional(),

  // === 通知 ===
  notification: NotificationConfig.optional(),

  // === 模型能力 ===
  model_capabilities: ModelCapabilities.optional(),

  // === Tmux 集成 ===
  tmux: TmuxConfig.optional(),

  // === 实验特性 ===
  experimental: ExperimentalConfig.optional(),

  // === 自动更新 ===
  auto_update: AutoUpdateConfig.optional(),

  // === Web 搜索 ===
  websearch: WebSearchConfig.optional(),

  // === 迁移追踪 ===
  _migrations: z.array(z.string()).optional(),
})
```

---

## 4. Agent Override Schema

```typescript
// src/config/schema/agent-overrides.ts
export const AgentOverride = z.object({
  enabled: z.boolean().optional(),
  model: z.string().optional(),
  fallback_models: z.array(z.string()).optional(),
  variant: z.string().optional(),
  temperature: z.number().optional(),
  top_p: z.number().optional(),
  max_turns: z.number().optional(),
  tools: z.object({
    allowed: z.array(z.string()).optional(),
    disallowed: z.array(z.string()).optional(),
  }).optional(),
  prompt: z.string().optional(),      // 自定义提示词
  description: z.string().optional(),
  category: z.string().optional(),
  can_delegate: z.boolean().optional(),
})
```

---

## 5. Category Schema

```typescript
// src/config/schema/categories.ts
export const CategoryConfig = z.object({
  description: z.string(),
  default_model: z.string(),
  agents: z.array(z.string()),
  skills: z.array(z.string()).optional(),
  tools: z.object({
    allowed: z.array(z.string()).optional(),
    disallowed: z.array(z.string()).optional(),
  }).optional(),
})

// 内置 8 个分类
export const BUILTIN_CATEGORIES = {
  quick: { description: 'Quick questions', default_model: 'claude-haiku-4', agents: ['explore'] },
  code: { description: 'Code implementation', default_model: 'claude-sonnet-4-6', agents: ['hephaestus', 'sisyphus-junior'] },
  frontend: { description: 'Frontend development', default_model: 'claude-sonnet-4-6', agents: ['hephaestus'] },
  backend: { description: 'Backend development', default_model: 'claude-sonnet-4-6', agents: ['hephaestus'] },
  infra: { description: 'Infrastructure', default_model: 'gpt-5.4', agents: ['oracle', 'hephaestus'] },
  data: { description: 'Data engineering', default_model: 'claude-opus-4-6', agents: ['oracle', 'hephaestus'] },
  security: { description: 'Security', default_model: 'gpt-5.4', agents: ['oracle', 'momus'] },
  quality: { description: 'Testing', default_model: 'claude-sonnet-4-6', agents: ['momus', 'hephaestus'] },
}
```

---

## 6. 多层合并

```typescript
// src/plugin-config.ts
export function loadPluginConfig(ctx: PluginContext): Config {
  const configs: Partial<Config>[] = []

  // 1. 加载项目级配置
  const projectPath = path.join(ctx.directory, '.opencode', 'oh-my-opencode.jsonc')
  if (await exists(projectPath)) {
    const projectConfig = await loadJsonc(projectPath)
    configs.push(projectConfig)
  }

  // 2. 加载用户级配置
  const userPath = path.join(ctx.homeDir, '.config', 'opencode', 'oh-my-opencode.jsonc')
  if (await exists(userPath)) {
    const userConfig = await loadJsonc(userPath)
    configs.push(userConfig)
  }

  // 3. Zod 验证 + Deep Merge
  const merged = deepMerge(configs, {
    // agents/categories 使用 deep merge
    // disabled_* 使用 Set 并集
    // 其他直接覆盖
  })

  // 4. Zod 验证（partial 降级）
  const result = OhMyOpenCodeConfig.safeParse(merged)
  if (!result.success) {
    // 尝试部分加载
    const partial = OhMyOpenCodeConfig.strict().safeParse(merged)
    if (partial.success) {
      return applyDefaults(partial.data)
    }
    throw new ConfigValidationError(result.error)
  }

  // 5. 自动迁移
  return runMigrations(result.data)
}

// Deep merge 实现 (原型污染安全)
function deepMerge(targets: Partial<Config>[]): Partial<Config> {
  const keys = new Set<string>()
  for (const t of targets) {
    for (const k of Object.keys(t)) keys.add(k)
  }

  const result: Record<string, unknown> = {}
  for (const key of keys) {
    const values = targets.map(t => t[key as keyof Config]).filter(Boolean)

    if (key.startsWith('disabled_')) {
      // Set 并集
      result[key] = [...new Set(values.flat())]
    } else if (isObject(values[0])) {
      // 深度合并对象
      result[key] = deepMerge(values as Partial<Config>[])
    } else {
      // 最后覆盖
      result[key] = values[values.length - 1]
    }
  }

  return result as Partial<Config>
}
```

---

## 7. 与 OpenCode 配置对比

| 维度 | OpenCode | Oh-My-OpenAgent |
|------|----------|-----------------|
| Schema 文件数 | 1 | **27** |
| 配置格式 | JSONC | JSONC |
| 禁用机制 | 无 | **Set 并集** |
| 合并策略 | 简单覆盖 | **Deep Merge + 字段类型区分** |
| 降级解析 | 无 | **Partial parse fallback** |
| 迁移追踪 | 无 | **_migrations 数组** |
| 分类系统 | 无 | **8 个内置分类 + 自定义** |
| Hook 配置 | 无 | **disabled_hooks** |

---

*文档版本：v1.0 | 更新：2026-04-06*
