---
title: "MCP 集成 — 三层架构与 OAuth 支持"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# MCP 集成 — 三层架构与 OAuth 支持

> 源码路径：`/mnt/e/code/cc/omo-code/src/mcp/`, `src/features/mcp-oauth/`
> 核心文件：`websearch/`, `context7/`, `grep_app/`, `mcp-oauth.ts`
> 技术栈：@modelcontextprotocol/sdk + OAuth 2.0 + PKCE + DCR (RFC 7591)

---

## 1. 概述

Oh-My-OpenAgent 的 MCP 架构分为三层，比 OpenCode 基础层更加丰富：

```
┌─────────────────────────────────────────────┐
│        Layer 3: Skill-Embedded MCPs         │
│   每个 Skill 可嵌入自己的 MCP 工具           │
│   按需启动，作用域精确，context 高效         │
├─────────────────────────────────────────────┤
│        Layer 2: Claude Code MCPs            │
│   .mcp.json 定义的环境变量展开              │
│   stdio + HTTP 混合                        │
├─────────────────────────────────────────────┤
│        Layer 1: Built-in MCPs                │
│   websearch (Exa/Tavily)                    │
│   context7 (官方文档)                       │
│   grep_app (GitHub 代码搜索)               │
└─────────────────────────────────────────────┘
```

---

## 2. 内置 MCP 服务器

### 2.1 WebSearch MCP

```typescript
// src/mcp/websearch/
export const websearchMCP = {
  name: 'websearch',
  description: 'Web search using Exa or Tavily',

  async callTool(args: { query: string; num_results?: number }) {
    const apiKey = config.websearch?.api_key ?? process.env.EXA_API_KEY
    const provider = config.websearch?.provider ?? 'exa'

    if (provider === 'exa') {
      const result = await fetch('https://api.exa.ai/search', {
        method: 'POST',
        headers: { 'Authorization': `Bearer ${apiKey}` },
        body: JSON.stringify({
          query: args.query,
          numResults: args.num_results ?? 10,
          type: 'neural',
        }),
      })
      return formatExaResults(await result.json())
    }

    // Tavily fallback
    const result = await fetch('https://api.tavily.com/search', {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${apiKey}` },
      body: JSON.stringify({
        query: args.query,
        max_results: args.num_results ?? 10,
      }),
    })
    return formatTavilyResults(await result.json())
  },
}
```

### 2.2 Context7 MCP

```typescript
// src/mcp/context7/
export const context7MCP = {
  name: 'context7',
  description: 'Official library documentation via Context7',

  async callTool(args: { library: string; query: string }) {
    // 调用 Context7 API 获取库文档
    const result = await fetch('https://context7.com/api/v1/query', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        library: args.library,
        query: args.query,
      }),
    })

    return formatContext7Results(await result.json())
  },
}
```

### 2.3 GrepApp MCP

```typescript
// src/mcp/grep_app/
export const grepAppMCP = {
  name: 'grep_app',
  description: 'GitHub code search via Grep.app',

  async callTool(args: { query: string; language?: string }) {
    const url = new URL('https://grep.app/api/search')
    url.searchParams.set('q', args.query)
    if (args.language) {
      url.searchParams.set('lang', args.language)
    }

    const result = await fetch(url.toString())
    return formatGrepAppResults(await result.json())
  },
}
```

---

## 3. Skill-MCP 三层架构

### Layer 1: Built-in MCPs

在 `src/mcp/` 中定义，直接连接。

### Layer 2: Claude Code MCPs

```typescript
// src/features/claude-code-mcp-loader/
export async function loadClaudeCodeMcps(): Promise<void> {
  // 读取 .mcp.json
  const mcpPath = path.join(ctx.directory, '.mcp.json')
  if (!await exists(mcpPath)) return

  const mcpConfig = JSON.parse(await readFile(mcpPath))

  // 展开环境变量
  for (const [name, config] of Object.entries(mcpConfig)) {
    config.env = expandEnvVars(config.env)
  }

  // 连接每个 MCP
  for (const [name, config] of Object.entries(mcpConfig)) {
    const client = await MCPClient.connect(config)
    await skillMcpManager.register(name, client)
  }
}
```

### Layer 3: Skill-Embedded MCPs

```typescript
// src/features/skill-mcp-manager/
export class SkillMcpManager {
  async loadForSkill(skillName: string): Promise<void> {
    const skill = await skillLoader.load(skillName)

    if (!skill.metadata.mcp) return

    // 每个 Skill 有自己的 MCP 配置
    const mcpConfig = {
      type: skill.metadata.mcp.type,  // 'stdio' | 'http'
      command: skill.metadata.mcp.command,
      args: skill.metadata.mcp.args,
      env: {
        ...process.env,
        ...skill.metadata.mcp.env,
        SKILL_CONTEXT: JSON.stringify({
          skill: skillName,
          variables: skill.variables,
        }),
      },
    }

    const client = await MCPClient.connect(mcpConfig)

    // 注册为 skill_<name>_<tool> 格式
    const tools = await client.listTools()
    for (const tool of tools) {
      this.registry.register(`skill_${skillName}_${tool.name}`, tool)
    }

    this.activeClients.set(skillName, client)
  }
}
```

---

## 4. OAuth 支持

```typescript
// src/features/mcp-oauth/mcp-oauth.ts
export class McpOAuth {
  // 动态客户端注册 (RFC 7591)
  async registerClient(mcpServer: string): Promise<RegisteredClient> {
    const config = this.serverConfigs[mcpServer]?.oauth

    const response = await fetch(config.registrationUrl ?? DEFAULT_DCR_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        client_name: `oh-my-opencode:${mcpServer}`,
        redirect_uris: [this.callbackUrl],
        grant_types: ['authorization_code'],
        response_types: ['code'],
        scope: config.scopes?.join(' ') ?? 'default',
      }),
    })

    return response.json()
  }

  // PKCE 授权码流程
  async authorize(mcpServer: string): Promise<void> {
    const { client, config } = await this.getClientConfig(mcpServer)

    // 生成 PKCE verifier + challenge
    const verifier = generateRandomString(64)
    const challenge = await sha256(verifier)

    // 构建授权 URL
    const authUrl = new URL(config.authUrl ?? DEFAULT_AUTH_URL)
    authUrl.searchParams.set('client_id', client.clientId)
    authUrl.searchParams.set('redirect_uri', this.callbackUrl)
    authUrl.searchParams.set('scope', config.scopes?.join(' ') ?? 'default')
    authUrl.searchParams.set('response_type', 'code')
    authUrl.searchParams.set('code_challenge', challenge)
    authUrl.searchParams.set('code_challenge_method', 'S256')

    // 打开浏览器
    await open(authUrl.toString())

    // 等待回调
    const callback = await this.waitForCallback()

    // 交换 token
    const token = await this.exchangeCode(callback.code, verifier, client)
    await this.storeToken(mcpServer, token)
  }
}
```

---

## 5. 与 OpenCode 对比

| 维度 | OpenCode | Oh-My-OpenAgent |
|------|----------|-----------------|
| MCP 层数 | 1 | **3** |
| 内置 MCP | 无 | **websearch, context7, grep_app** |
| Skill-MCP | 无 | **按需启动 + 作用域隔离** |
| OAuth | 基础 | **PKCE + DCR 完整支持** |
| 环境变量展开 | 无 | **JSONC env 展开** |
| MCP 工具命名 | client:tool | **skill_xxx_tool** |

---

*文档版本：v1.0 | 更新：2026-04-06*
