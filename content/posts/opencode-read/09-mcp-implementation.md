---
title: "MCP 实现 — 协议支持与工具翻译"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# MCP 实现 — 协议支持与工具翻译

> 源码路径：`/mnt/e/code/cc/opencode/packages/opencode/src/mcp/`
> 核心文件：`mcp/index.ts`
> 技术栈：@modelcontextprotocol/sdk + Effect + JSON Schema

---

## 1. 概述

OpenCode 对 MCP (Model Context Protocol) 的支持非常完整，不仅作为 MCP 客户端使用外部服务器，还实现了完整的 MCP 工具翻译层。相比 Claude Code，OpenCode 的 MCP 实现更加体系化。

---

## 2. MCP Service 接口

```typescript
// packages/opencode/src/mcp/index.ts
export interface Interface {
  readonly status: () => Effect.Effect<Record<string, Status>>
  readonly clients: () => Effect.Effect<Record<string, MCPClient>>
  readonly tools: () => Effect.Effect<Record<string, Tool>>
  readonly prompts: () => Effect.Effect<Record<string, PromptInfo & { client: string }>>
  readonly resources: () => Effect.Effect<Record<string, ResourceInfo & { client: string }>>

  readonly add: (name: string, mcp: Config.Mcp) => Effect.Effect<{ status: Record<string, Status> | Status }>
  readonly connect: (name: string) => Effect.Effect<void>
  readonly disconnect: (name: string) => Effect.Effect<void>

  // OAuth 支持
  readonly auth: {
    startOAuth: (name: string) => Effect.Effect<void>
    handleCallback: (name: string, callbackUrl: string) => Effect.Effect<void>
  }
}
```

---

## 3. MCP 配置

```typescript
// packages/opencode/src/config/config.ts
export namespace Mcp {
  // 本地 MCP 服务器
  export const Local = z.object({
    type: z.literal("local"),
    command: z.array(z.string()),
    args: z.array(z.string()).optional(),
    env: z.record(z.string()).optional(),
    enabled: z.boolean().optional(),
    timeout: z.number().optional(),
  })

  // 远程 MCP 服务器
  export const Remote = z.object({
    type: z.literal("remote"),
    url: z.string().url(),
    enabled: z.boolean().optional(),
    headers: z.record(z.string()).optional(),
    oauth: McpOAuth.optional(),
    timeout: z.number().optional(),
  })

  // OAuth 配置
  export const McpOAuth = z.object({
    clientId: z.string(),
    clientSecret: z.string(),
    scopes: z.array(z.string()).optional(),
    authUrl: z.string().optional(),
    tokenUrl: z.string().optional(),
  })
}
```

配置示例：

```jsonc
{
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": ["npx", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
      "enabled": true
    },
    "github": {
      "type": "remote",
      "url": "https://mcp.github.com",
      "oauth": {
        "clientId": "xxx",
        "clientSecret": "{env:GITHUB_CLIENT_SECRET}",
        "scopes": ["repo", "user"]
      }
    }
  }
}
```

---

## 4. 传输层支持

```typescript
// packages/opencode/src/mcp/index.ts

// 1. Stdio — 本地进程通信
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js"
const transport = new StdioClientTransport({
  command: "npx",
  args: ["mcp-server", ...],
})

// 2. Streamable HTTP — HTTP 流式传输
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js"
const transport = new StreamableHTTPClientTransport({
  url: "https://mcp.server.com/mcp",
  requestOptions: { headers: { Authorization: `Bearer ${token}` } },
})

// 3. SSE — Server-Sent Events
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js"
const transport = new SSEClientTransport(new URL("https://mcp.server.com/sse"))
```

---

## 5. 工具翻译层

### 5.1 MCP 工具转 OpenCode Tool

```typescript
// packages/opencode/src/mcp/index.ts (lines 133-161)
function convertMcpTool(mcpTool: MCPToolDef, client: MCPClient, timeout?: number): Tool {
  const inputSchema = mcpTool.inputSchema

  // 转换为 JSON Schema
  const schema: JSONSchema7 = {
    ...(inputSchema as JSONSchema7),
    type: "object",
    properties: (inputSchema.properties ?? {}) as JSONSchema7["properties"],
    additionalProperties: false,
  }

  return dynamicTool({
    id: `${sanitizedClient}:${sanitizedToolName}`,
    description: mcpTool.description ?? "",
    inputSchema: jsonSchema(schema),

    execute: async (args: unknown) => {
      const result = await client.callTool({
        name: mcpTool.name,
        arguments: (args || {}) as Record<string, unknown>,
      }, CallToolResultSchema, {
        resetTimeoutOnProgress: true,
        timeout,
      })

      // 转换为 OpenCode 工具结果格式
      return {
        title: mcpTool.name,
        output: formatMcpResult(result),
        metadata: {
          client: sanitizedClient,
          toolName: mcpTool.name,
        },
      }
    },
  })
}
```

### 5.2 工具命名

```typescript
// 工具名称格式: clientName:toolName
const sanitizedClient = clientName.replace(/[^a-zA-Z0-9_-]/g, "_")
const sanitizedToolName = mcpTool.name.replace(/[^a-zA-Z0-9_-]/g, "_")
const toolId = `${sanitizedClient}:${sanitizedToolName}`
```

---

## 6. OAuth 认证流程

```typescript
// packages/opencode/src/mcp/index.ts
async function startOAuthFlow(config: McpOAuth, redirectUrl: string) {
  // 1. 动态客户端注册 (RFC 7591)
  const registration = await fetch(config.registrationUrl ?? DEFAULT_REG_URL, {
    method: "POST",
    body: JSON.stringify({
      client_name: "opencode",
      redirect_uris: [redirectUrl],
      grant_types: ["authorization_code"],
      response_types: ["code"],
    }),
  })

  // 2. 构建授权 URL
  const authUrl = new URL(config.authUrl ?? DEFAULT_AUTH_URL)
  authUrl.searchParams.set("client_id", registration.clientId)
  authUrl.searchParams.set("redirect_uri", redirectUrl)
  authUrl.searchParams.set("scope", config.scopes?.join(" ") ?? "read")
  authUrl.searchParams.set("response_type", "code")

  // 3. 打开浏览器授权
  await open(authUrl.toString())

  // 4. 等待回调
  const callbackUrl = await waitForCallback()

  // 5. 交换 token
  const token = await exchangeCodeForToken(callbackUrl, registration)
}
```

---

## 7. 状态管理

```typescript
// MCP 服务器状态
export type Status =
  | "connected"    // 已连接
  | "disabled"     // 被禁用
  | "failed"       // 连接失败
  | "needs_auth"   // 需要 OAuth 认证
  | "needs_client_registration"  // 需要客户端注册
```

---

## 8. 与 Claude Code 对比

| 维度 | Claude Code | OpenCode |
|------|------------|----------|
| MCP 角色 | MCP 工具消费者 | MCP 工具消费者 + 完整 MCP 服务器 |
| 工具转换 | 基础映射 | 完整 JSON Schema 转换 |
| 传输协议 | stdio | stdio + HTTP + SSE |
| OAuth | 无 | 完整 OAuth 流程支持 |
| 动态注册 | 无 | RFC 7591 客户端注册 |
| 提示词暴露 | 无 | MCP prompts → 命令 |
| 资源暴露 | 无 | MCP resources → 内部资源 |
| 超时控制 | 工具级超时 | MCP 层超时 + resetTimeoutOnProgress |

---

*文档版本：v1.0 | 更新：2026-04-06*
