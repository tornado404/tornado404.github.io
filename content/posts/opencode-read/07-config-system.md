---
title: "配置系统 — 多层配置合并与路径解析"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# 配置系统 — 多层配置合并与路径解析

> 源码路径：`/mnt/e/code/cc/opencode/packages/opencode/src/config/`
> 核心文件：`config.ts`, `paths.ts`, `tui.ts`, `tui-schema.ts`
> 技术栈：Zod + Effect + JSONC

---

## 1. 概述

OpenCode 的配置系统支持 **多层合并**，优先级从高到低：

1. **Managed Config** (企业 MDM) — 最高优先级
2. **Project Config** (`.opencode/`)
3. **User Config** (`~/.config/opencode/`)
4. **Default Values**

这与 Claude Code 基于 TOML 的单文件配置形成对比，OpenCode 的 JSONC 格式更灵活（支持注释）。

---

## 2. 配置加载层级

### 2.1 Managed Config (企业级)

```typescript
// packages/opencode/src/config/config.ts
const managedConfigPaths = {
  darwin: "/Library/Application Support/opencode/",
  win32: "C:\\ProgramData\\opencode\\",
  linux: "/etc/opencode/",
}

// 企业 IT 可通过 MDM 强制覆盖用户配置
// 适用于安全策略、合规要求等场景
```

### 2.2 Project Config

```typescript
// 从当前目录向上扫描到 worktree 根
const projectConfigPaths = [
  path.join(cwd, ".opencode"),
  path.join(parentDir, ".opencode"),
  // ...
  path.join(worktreeRoot, ".opencode"),
]
```

### 2.3 User Config

```typescript
// XDG Base Directory Specification
const userConfigDir = {
  darwin: "~/Library/Application Support/opencode/",
  linux: "~/.config/opencode/",
  win32: "%APPDATA%\\opencode\\",
}
```

### 2.4 环境变量覆盖

```typescript
// 环境变量可覆盖任意配置项
if (process.env.OPENCODE_CONFIG) {
  // 使用指定配置文件
}
if (process.env.OPENCODE_CONFIG_CONTENT) {
  // 直接使用内联 JSON 配置
}
```

---

## 3. 配置 Schema

```typescript
// packages/opencode/src/config/config.ts (~849 行)
export const Info = z.object({
  // === 模型配置 ===
  model: z.string().optional().describe("Default model (provider/model format)"),
  small_model: z.string().optional().describe("Small model for lightweight tasks"),

  // === Agent 配置 ===
  default_agent: z.string().optional(),
  agent: z.record(z.string(), Agent.Config).optional(),

  // === Provider 配置 ===
  provider: z.record(z.string(), Provider.Config).optional(),
  disabled_providers: z.array(z.string()).optional(),
  enabled_providers: z.array(z.string()).optional(),

  // === MCP 配置 ===
  mcp: z.record(z.string(), Mcp.Config).optional(),

  // === Skills 配置 ===
  skills: z.object({
    paths: z.array(z.string()).optional(),
    urls: z.array(z.string()).optional(),
  }).optional(),

  // === 自定义命令 ===
  command: z.record(z.string(), Command.Config).optional(),

  // === 权限覆盖 ===
  permission: Permission.Config.optional(),

  // === 插件配置 ===
  plugin: z.record(z.string(), Plugin.Spec).optional(),

  // === LSP 配置 ===
  lsp: z.record(z.string(), Lsp.Config).optional(),

  // === 格式化配置 ===
  formatter: z.record(z.string(), z.any()).optional(),

  // === 分享配置 ===
  share: z.enum(["manual", "auto", "disabled"]).optional(),

  // === 键盘绑定 ===
  keybinds: Keybind.Config.optional(),

  // === 实验特性 ===
  experimental: z.record(z.string(), z.any()).optional(),

  // === ... 80+ 更多配置项 ===
})
```

---

## 4. 路径解析与 JSONC 支持

```typescript
// packages/opencode/src/config/paths.ts
export function resolveConfig(path: string): Config {
  // 1. 读取文件内容（支持 JSONC — 带注释的 JSON）
  const content = readFileWithComments(path)

  // 2. 环境变量替换: {env:VAR_NAME}
  const withEnvVars = content.replace(
    /\{env:([A-Z_]+)\}/g,
    (_, name) => process.env[name] ?? ""
  )

  // 3. 文件引用替换: {file:path/to/file}
  const withFileRefs = withEnvVars.replace(
    /\{file:([^}]+)\}/g,
    (_, ref) => readFile(resolve(ref))
  )

  // 4. 波浪号展开: ~/path
  const expanded = withFileRefs.replace(/^~/, os.homedir())

  // 5. Zod 验证
  return Config.Info.parse(JSON.parse(expanded))
}
```

**JSONC 支持**：
```jsonc
{
  // 这是注释
  "model": "anthropic/claude-sonnet-4",
  "provider": {
    "anthropic": {
      "api_key": "{env:ANTHROPIC_API_KEY}"  // 环境变量引用
    }
  }
}
```

---

## 5. TUI 配置

```typescript
// packages/opencode/src/config/tui-schema.ts
export const TuiInfo = z.object({
  theme: z.record(z.string(), z.any()).optional(),
  keybinds: z.record(z.string(), z.any()).optional(),
  plugin: z.record(z.string(), z.boolean()).optional(),
  scroll_speed: z.number().optional(),
  diff_style: z.enum(["side-by-side", "unified"]).optional(),
  mouse: z.boolean().optional(),
})
```

---

## 6. 键盘绑定配置

```typescript
// packages/opencode/src/config/config.ts (lines 610-709)
export const KeybindInfo = z.object({
  // 基础
  leader: z.string().optional(),
  exit: z.union([z.string(), z.array(z.string())]).optional(),

  // 会话管理
  "session:new": z.union([z.string(), z.array(z.string())]).optional(),
  "session:list": z.union([z.string(), z.array(z.string())]).optional(),
  "session:fork": z.union([z.string(), z.array(z.string())]).optional(),
  "session:rename": z.union([z.string(), z.array(z.string())]).optional(),
  "session:delete": z.union([z.string(), z.array(z.string())]).optional(),
  "session:share": z.union([z.string(), z.array(z.string())]).optional(),

  // 消息导航
  "message:up": z.union([z.string(), z.array(z.string())]).optional(),
  "message:down": z.union([z.string(), z.array(z.string())]).optional(),

  // 模型选择
  "model:select": z.union([z.string(), z.array(z.string())]).optional(),
  "agent:select": z.union([z.string(), z.array(z.string())]).optional(),

  // ...
})
```

---

## 7. 与 Claude Code 对比

| 维度 | Claude Code | OpenCode |
|------|------------|----------|
| 配置文件格式 | TOML | JSONC |
| 配置分层 | 无层级（单文件） | 4 层（Managed > Project > User > Default） |
| 注释支持 | TOML 原生支持 | JSONC 支持 |
| Schema 验证 | Hugo 配置合并 | Zod 验证 |
| 环境变量 | `env()` 函数 | `{env:VAR}` 占位符 |
| 文件引用 | 无 | `{file:path}` 占位符 |
| 企业 MDM | 无 | Managed Config 支持 |
| 键盘绑定 | 代码硬编码 | JSON 配置 |

---

*文档版本：v1.0 | 更新：2026-04-06*
