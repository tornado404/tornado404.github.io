---
title: "架构概览 — 插件架构与初始化流程"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 架构概览 — 插件架构与初始化流程

> 源码路径：`/mnt/e/code/cc/omo-code/src/`
> 核心文件：`index.ts`, `create-hooks.ts`, `create-managers.ts`, `create-tools.ts`
> 技术栈：Bun + TypeScript + Zod v4 + Effect + @opencode-ai/plugin

---

## 1. 概述

Oh-My-OpenAgent 是一个 **OpenCode 插件**，通过 `@opencode-ai/plugin` 暴露的 Hook 接口将自己的功能编织进 OpenCode 的执行管线。它不是 fork，而是**装饰层**——对 OpenCode 的行为进行扩展和覆盖，而不修改其核心代码。

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Oh-My-OpenCode 插件                          │
│                                                              │
│  OhMyOpenCodePlugin(ctx)                                       │
│    │                                                         │
│    ├─ loadPluginConfig()      配置加载与 Zod 验证            │
│    ├─ createManagers()        管理器实例化                    │
│    ├─ createTools()           工具注册                       │
│    ├─ createHooks()           Hook 组合                       │
│    └─ createPluginInterface() OpenCode 接口暴露               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      OpenCode 核心                              │
│                                                              │
│  Agent System ─── Tool Registry ─── Hook Pipeline ─── Session │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 目录结构

```
src/
├── index.ts                           # 插件入口
├── plugin-config.ts                   # 多层配置加载
├── create-hooks.ts                   # 52 个 Hook 组合
├── create-managers.ts                # 管理器实例化
├── create-tools.ts                   # 工具注册入口
│
├── agents/                           # 11 个 Agent
│   ├── sisyphus/                    # 主协调者
│   │   ├── index.ts
│   │   ├── prompt.ts
│   │   ├── gpt-5-4.ts              # GPT-5.4 优化提示
│   │   ├── gemini.ts
│   │   └── anthropic.ts
│   ├── hephaestus/                  # 深工作者 (GPT 原生)
│   ├── prometheus/                  # 战略规划者
│   ├── atlas/                       # Todo 编排者
│   ├── oracle/                       # 架构顾问
│   ├── librarian/                   # 文档搜索
│   ├── explore/                     # 快速 grep
│   ├── metis/                       # 计划差距分析
│   ├── momus/                       # 计划审查
│   ├── sisyphus-junior/            # 轻量委托者
│   ├── multimodal-looker/          # 视觉分析
│   ├── builtin-agents.ts           # Agent 工厂模式
│   ├── types.ts                    # Agent 类型定义
│   └── dynamic-agent-prompt-builder.ts  # 动态提示构建
│
├── tools/                           # 26 个扩展工具
│   ├── delegate-task/               # 分类委托任务
│   ├── hashline-edit/               # 哈希锚定编辑
│   ├── background-task/             # 后台任务
│   ├── skill/                       # Skill 调用
│   ├── skill-mcp/                  # Skill-MCP 集成
│   ├── session-manager/             # 会话管理
│   ├── call-omo-agent/              # 直接调用 OMO Agent
│   ├── look-at/                    # 视觉分析
│   ├── lsp/                        # LSP 工具
│   ├── ast-grep/                   # AST 搜索
│   ├── grep/                       # 正则搜索
│   ├── glob/                       # 文件模式
│   ├── interactive-bash/           # Tmux Bash
│   ├── slashcommand/               # 命令发现
│   ├── task/                       # 任务 CRUD
│   └── plugin/                     # 工具注册表
│
├── hooks/                           # 52 个 Hook
│   ├── context-window-monitor/     # Token 预算监控
│   ├── session-notification/       # 桌面通知
│   ├── background-notification/    # 后台任务通知
│   ├── ralph-loop/                 # 自引用循环检测
│   ├── comment-checker/            # AI slop 检测
│   ├── thinking-block-validator/   # 思考块验证
│   ├── edit-error-recovery/        # 编辑错误恢复
│   ├── compaction-context-injector/ # 压缩上下文保留
│   ├── runtime-fallback/            # 模型错误恢复
│   ├── todo-continuation-enforcer/ # Todo 强制完成
│   ├── preemptive-compaction/       # 抢占式压缩
│   └── 40+ 更多...
│
├── features/                        # 19 个功能模块
│   ├── background-agent/           # 后台 Agent 管理
│   ├── tmux-subagent/             # Tmux 会话管理
│   ├── skill-mcp-manager/         # Skill-MCP 生命周期
│   ├── opencode-skill-loader/      # Skill 加载
│   ├── builtin-skills/             # 8 个内置 Skill
│   ├── builtin-commands/           # 8 个内置命令
│   ├── claude-code-agent-loader/   # OpenCode Agent 兼容
│   ├── claude-code-command-loader/  # OpenCode 命令兼容
│   ├── claude-code-mcp-loader/    # OpenCode MCP 兼容
│   ├── claude-code-plugin-loader/  # OpenCode 插件兼容
│   ├── mcp-oauth/                  # MCP OAuth
│   ├── comment-checker/            # AI slop 检查
│   └── 8 more...
│
├── mcp/                             # 3 个内置 MCP
│   ├── websearch/                  # Exa/Tavily 搜索
│   ├── context7/                    # 官方文档
│   └── grep_app/                   # GitHub 代码搜索
│
├── plugin-handlers/                 # OpenCode Hook 处理
│   ├── config-handler.ts            # 6 阶段配置加载
│   ├── agent-config-handler.ts      # Agent 配置
│   ├── tool-config-handler.ts       # 工具配置
│   ├── mcp-config-handler.ts       # MCP 配置
│   ├── command-config-handler.ts    # 命令注册
│   └── provider-config-handler.ts   # Provider 配置
│
├── config/                          # Zod v4 Schema (27 个文件)
│   └── schema/
│       ├── oh-my-opencode-config.ts
│       ├── agent-overrides.ts
│       ├── categories.ts
│       ├── hooks.ts
│       ├── fallback-models.ts
│       └── 22 more...
│
├── cli/                             # CLI 工具
│   ├── install.ts                   # 交互式安装
│   ├── run.ts                      # 非交互式执行
│   ├── doctor.ts                   # 健康诊断
│   └── mcp-oauth/                  # OAuth 工具
│
└── shared/                          # 100+ 工具函数
```

---

## 4. 初始化流程详解

### 4.1 插件入口

```typescript
// src/index.ts
export function OhMyOpenCodePlugin(ctx: PluginContext): PluginInterface {
  // 1. 加载并验证配置
  const config = loadPluginConfig(ctx)

  // 2. 实例化管理器
  const managers = createManagers(config, ctx)

  // 3. 构建工具注册表
  const tools = createTools(config, managers)

  // 4. 组合 Hook
  const hooks = createHooks(config, managers)

  // 5. 暴露 OpenCode 接口
  return createPluginInterface(tools, hooks, managers)
}
```

### 4.2 管理器系统

```typescript
// src/create-managers.ts
function createManagers(config: Config, ctx: PluginContext) {
  return {
    // Tmux 会话管理器 — 实时可视化多 Agent
    tmuxManager: new TmuxSessionManager(config.tmux),

    // 后台 Agent 管理器 — 并发控制
    backgroundManager: new BackgroundManager(config.background_task),

    // Skill-MCP 管理器 — 生命周期管理
    skillMcpManager: new SkillMcpManager(config),

    // 配置处理器 — 6 阶段配置应用
    configHandler: new ConfigHandler(config),
  }
}
```

### 4.3 工具注册

```typescript
// src/create-tools.ts
function createTools(config: Config, managers: Managers) {
  const registry = new ToolRegistry()

  // LSP 工具 (6 个)
  registry.register('lsp_goto_definition', createLspGotoDefinition(managers))
  registry.register('lsp_find_references', createLspFindReferences(managers))
  registry.register('lsp_symbols', createLspSymbols(managers))
  registry.register('lsp_diagnostics', createLspDiagnostics(managers))
  registry.register('lsp_prepare_rename', createLspPrepareRename(managers))
  registry.register('lsp_rename', createLspRename(managers))

  // 自定义工具
  registry.register('task', createDelegateTaskTool(config))
  registry.register('background_output', createBackgroundOutputTool(managers))
  registry.register('background_cancel', createBackgroundCancelTool(managers))
  registry.register('call_omo_agent', createCallOmoAgentTool(config))
  registry.register('hashline_edit', createHashlineEditTool(config))
  registry.register('look_at', createLookAtTool(config))
  registry.register('skill', createSkillTool(managers))
  registry.register('skill_mcp', createSkillMcpTool(managers))
  registry.register('interactive_bash', createInteractiveBashTool(managers))

  // 会话管理工具
  registry.register('session_list', createSessionListTool(ctx))
  registry.register('session_read', createSessionReadTool(ctx))
  registry.register('session_search', createSessionSearchTool(ctx))
  registry.register('session_info', createSessionInfoTool(ctx))

  // 搜索工具
  registry.register('grep', createGrepTool(config))
  registry.register('glob', createGlobTool(config))
  registry.register('ast_grep_search', createAstGrepSearchTool(config))
  registry.register('ast_grep_replace', createAstGrepReplaceTool(config))

  return registry
}
```

### 4.4 Hook 组合

```typescript
// src/create-hooks.ts
function createHooks(config: Config, managers: Managers) {
  return {
    // === 10 个 OpenCode 钩子接口 ===

    config: createConfigHandler(config, managers),

    tool: createToolHandler(config, managers),

    ['chat.message']: createChatMessageHook(config, managers),

    ['chat.params']: createChatParamsHook(config),

    ['chat.headers']: createChatHeadersHook(config),

    event: createEventHook(config, managers),

    ['tool.execute.before']: createToolBeforeHooks(config, managers),

    ['tool.execute.after']: createToolAfterHooks(config, managers),

    ['experimental.chat.messages.transform']: createTransformHooks(config, managers),

    ['experimental.session.compacting']: createCompactingHook(config, managers),
  }
}
```

---

## 5. 与 OpenCode 的架构对比

| 维度 | OpenCode | Oh-My-OpenAgent |
|------|----------|-----------------|
| 架构风格 | 单体式 | 插件装饰层 |
| 入口点 | CLI/App | `OhMyOpenCodePlugin()` |
| 依赖注入 | Effect Layer | 构造函数注入 |
| 配置验证 | Zod | Zod v4 (27 个 Schema 文件) |
| 工具注册 | ToolRegistry Service | 插件 ToolHandler |
| Hooks | 基础 Plugin Hooks | 52 个三层 Hook |
| 持久化 | SQLite + Drizzle | 复用 OpenCode |
| 通信 | Session 消息 | Hook 事件 + SDK |

---

## 6. 核心文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/index.ts` | — | 插件入口，5 步初始化 |
| `src/plugin-config.ts` | — | 多层配置加载 |
| `src/create-hooks.ts` | — | 52 个 Hook 组合 |
| `src/create-managers.ts` | — | 管理器实例化 |
| `src/create-tools.ts` | — | 工具注册入口 |
| `src/plugin/tool-registry.ts` | — | 工具注册表实现 |
| `src/plugin-handlers/config-handler.ts` | — | 6 阶段配置处理 |

---

*文档版本：v1.0 | 更新：2026-04-06*
