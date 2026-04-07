---
title: "IDE 集成 — Zed、VS Code 与 ACP 协议"
date: 2026-04-06
draft: false
authors: ["钟子期"]
---

# IDE 集成 — Zed、VS Code 与 ACP 协议

> 源码路径：`/mnt/e/code/cc/opencode/packages/opencode/src/ide/`, `packages/opencode/src/acp/`, `packages/extensions/`
> 核心文件：`ide/index.ts`, `acp/agent.ts`, `acp/session.ts`
> 技术栈：ACP + HTTP + WebSocket + stdio

---

## 1. 概述

OpenCode 的 IDE 集成采用了**协议分层**的思路：

- **VS Code**：通过 HTTP REST API 集成
- **Zed**：通过 ACP (Agent Client Protocol) 深度集成
- **其他编辑器**：HTTP API 或 ACP

这与 Claude Code 的 VS Code / JetBrains 扩展模式形成对比。

---

## 2. IDE 检测

```typescript
// packages/opencode/src/ide/index.ts
export function detectIDE(): IDE {
  // 1. 检查 TERM_PROGRAM
  switch (process.env.TERM_PROGRAM) {
    case "vscode": return IDE.VSCode
    case "vscode-insiders": return IDE.VSCodeInsiders
    case "Cursor": return IDE.Cursor
    case "VSCodium": return IDE.VSCodium
    case "Windsurf": return IDE.Windsurf
    case "zed": return IDE.Zed
  }

  // 2. 检查 GIT_ASKPASS（VS Code 系列使用）
  if (process.env.GIT_ASKPASS?.includes("vscode")) {
    return IDE.VSCode
  }

  // 3. 检查 OPENCODE_CALLER 环境变量
  switch (process.env.OPENCODE_CALLER) {
    case "vscode": return IDE.VSCode
    case "cursor": return IDE.Cursor
    case "zed": return IDE.Zed
  }

  return IDE.Unknown
}
```

---

## 3. VS Code 集成

### 3.1 快捷键

```typescript
// packages/opencode/sdks/vscode/src/extension.ts
const KEYBINDINGS = {
  QUICK_LAUNCH: {
    mac: "Cmd+Esc",
    winlinux: "Ctrl+Esc",
  },
  NEW_SESSION: {
    mac: "Cmd+Shift+Esc",
    winlinux: "Ctrl+Shift+Esc",
  },
  FILE_REFERENCES: {
    mac: "Cmd+Option+K",
    winlinux: "Ctrl+Alt+K",
  },
}
```

### 3.2 通信架构

```
┌─────────────────────────────────────────────────────────┐
│                   VS Code Extension                      │
│                                                          │
│  1. 监听快捷键                                            │
│  2. 生成随机端口                                          │
│  3. 启动 opencode --port [PORT]                         │
│  4. 通过 HTTP API 通信                                   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼ HTTP (localhost:[PORT])
┌─────────────────────────────────────────────────────────┐
│                   opencode server                        │
│                                                          │
│  POST /tui/append-prompt    — 发送提示词                 │
│  GET  /status               — 获取状态                    │
│  WS   /events              — 实时事件流                   │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Zed 集成 — ACP 协议

### 4.1 Zed 扩展定义

```toml
# packages/extensions/zed/extension.toml
[extension]
name = "opencode"
agent_server = "opencode"  # 启动命令

[[platform]]
os = "macos"
arch = "arm64"
url = "https://..."

[[platform]]
os = "macos"
arch = "x86_64"
url = "https://..."

[[platform]]
os = "linux"
arch = "arm64"
url = "https://..."

[[platform]]
os = "linux"
arch = "x86_64"
url = "https://..."

[[platform]]
os = "windows"
arch = "x86_64"
url = "https://..."
```

### 4.2 ACP 协议实现

```typescript
// packages/opencode/src/cli/cmd/acp.ts
export async function startACPserver(options: ACPOptions) {
  // 1. 创建 stdio 双工流
  const input = process.stdin
  const output = process.stdout

  // 2. 初始化 AgentSideConnection
  const connection = new AgentSideConnection({
    input,
    output,
    protocol: "acp",
  })

  // 3. 监听消息
  connection.on("session/new", async (msg) => {
    const session = await Session.create({ ... })
    return { sessionId: session.id }
  })

  connection.on("session/load", async (msg) => {
    const session = await Session.load(msg.sessionId)
    return { session }
  })

  connection.on("session/prompt", async (msg) => {
    const result = await SessionPrompt.prompt({
      sessionID: msg.sessionId,
      messageID: msg.messageId,
      parts: msg.parts,
    })
    return { result }
  })
}
```

### 4.3 ACP 类型定义

```typescript
// packages/opencode/src/acp/types.ts
export namespace ACP {
  export type MessageType =
    | "session/new"
    | "session/load"
    | "session/prompt"
    | "session/result"
    | "permission/request"
    | "permission/reply"
    | "tool/call"
    | "tool/result"
    | "error"

  export interface Message<T = unknown> {
    id: string
    type: MessageType
    payload: T
  }
}
```

---

## 5. HTTP 服务器

```typescript
// packages/opencode/src/server/index.ts
import { Hono } from "hono"

const app = new Hono()

// TUI 提示词追加
app.post("/tui/append-prompt", async (c) => {
  const { prompt } = await c.req.json()
  await session.appendPrompt(prompt)
  return c.json({ ok: true })
})

// 状态查询
app.get("/status", async (c) => {
  return c.json({
    status: "idle",
    sessionId: currentSession?.id,
  })
})

// WebSocket 实时事件
app.webSocket("/events", (c) => {
  const ws = c.req.websocket()
  eventBus.subscribe((event) => {
    ws.send(JSON.stringify(event))
  })
})
```

---

## 6. 与 Claude Code 对比

| 维度 | Claude Code | OpenCode |
|------|------------|----------|
| VS Code | 官方扩展 | SDK 示例 |
| JetBrains | 官方扩展 | 无 |
| Zed | 无 | 深度集成（ACP） |
| 通信协议 | 扩展 API | HTTP REST + ACP |
| 上下文共享 | 终端选择 | 活动标签 + 选择内容 |
| 快捷键 | 固定 | 可配置 |

Claude Code 的 IDE 扩展更加成熟，覆盖了 VS Code 和 JetBrains 两个主流 IDE；OpenCode 则在 Zed 上有更深的集成。

---

*文档版本：v1.0 | 更新：2026-04-06*
