+++
title = "Claude Code 记忆系统：索引与按需检索的架构设计"
date = 2026-04-07
draft = false
authors = ["钟子期"]
categories = ["Claude Code"]
tags = ["Memory", "Agent", "Architecture", "TypeScript", "Persistence"]
series = ["Claude Code架构解析"]
+++

## 概述

Claude Code 的记忆系统（memdir）是一套精心设计的持久化知识管理机制，其核心思想是：**MEMORY.md 只作为索引，内容分散在独立文件中，通过 Sonnet Selector 在每个查询循环结束时按需检索相关条目**，防止上下文溢出。

整个系统由三个相互协作的子系统构成：

- **记忆写入**：主 Agent 和记忆提取子代理（Extract Memories Subagent）共同负责识别值得持久化的知识
- **索引管理**：MEMORY.md 作为轻量索引，每个条目仅一行
- **按需检索**：在每次用户查询前，通过 Sonnet 模型从大量记忆文件中选取最相关的条目

**源码路径**：`/mnt/e/code/cc/claude-code-main`
**核心文件**：`src/memdir/`、`src/services/extractMemories/`、`src/utils/attachments.ts`
**技术栈**：Bun + TypeScript + Anthropic API

---

## 1. 记忆存储结构：索引与内容分离

### 1.1 双层架构设计

记忆系统采用**索引与内容分离**的策略，避免将大量记忆直接塞入上下文：

```
~/.claude/projects/<project>/memory/
├── MEMORY.md              # 索引文件（最多 200 行，超出截断）
├── user_role.md           # 独立记忆文件
├── feedback_testing.md    # 独立记忆文件
├── project_deadline.md    # 独立记忆文件
└── reference_linear.md    # 独立记忆文件
```

**MEMORY.md 的作用**：仅存储指向各记忆文件的指针，每条一行，格式为：

```
- [Title](file.md) — one-line hook
```

索引永远随系统 prompt 加载（截断至 200 行/25KB）。真正的记忆内容**按需加载**，不进入系统 prompt。

### 1.2 记忆文件格式

**文件**：`src/memdir/memoryTypes.ts`

每条记忆文件使用 YAML frontmatter，结构如下：

```markdown
---
name: {{memory name}}
description: {{one-line description — used to decide relevance, be specific}}
type: {{user, feedback, project, reference}}
---

记忆正文内容。
对于 feedback/project 类型，推荐结构为：
事实/规则
**Why:** 原因
**How to apply:** 如何应用
```

### 1.3 四种记忆类型

**文件**：`src/memdir/memoryTypes.ts`

系统将记忆严格限定为四种类型——这是**刻意的约束设计**，防止记忆库膨胀为代码片段的杂散仓库：

| 类型 | 作用域 | 何时保存 | 什么不该存 |
|------|--------|---------|-----------|
| `user` | 用户画像 | 学习到用户角色、偏好、知识背景 | 代码模式、架构（可从代码推导） |
| `feedback` | 行为指导 | 用户纠正或确认了非显而易见的行为 | Git 历史（`git log` 是权威来源） |
| `project` | 项目上下文 | 了解谁在做什么、为什么、何时截止 | 调试方案（修复在代码中） |
| `reference` | 外部系统指针 | 发现外部工具/看板/频道的用途 | CLAUDE.md 中已有的内容 |

**Why this matters**：如果允许存储任何"有用"信息，记忆库会变成代码片段、架构快照、活动日志的大杂烩——这些信息要么可以从代码本身推导，要么会随时间腐化成误导性断言。系统通过类型约束和"不该保存"清单将记忆聚焦于**不可推导的上下文**。

### 1.4 目录结构：auto + team

**文件**：`src/memdir/paths.ts`、`src/memdir/teamMemPaths.ts`

当 `TEAMMEM` 功能开关开启时，记忆目录扩展为双层结构：

```
~/.claude/projects/<project>/memory/
├── MEMORY.md              # 个人记忆索引
├── user_role.md
├── feedback_testing.md
└── team/                  # 团队共享记忆目录
    ├── MEMORY.md          # 团队索引
    ├── project_policy.md   # 团队级反馈/项目记忆
    └── reference_ci.md    # 团队级外部系统指针
```

团队记忆和个人记忆的边界由每个类型的 `<scope>` 字段定义。例如：
- `feedback`：默认为 private；只有项目级通用约定（如测试策略）才存为 team
- `project`：强烈倾向于 team
- `reference`：通常为 team

团队记忆的安全边界极为严格（`teamMemPaths.ts`），实现了多层防御：

```
┌──────────────────────────────────────────────────────────┐
│              团队记忆写入安全验证                            │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  第一层：路径解析验证                                      │
│    ├─ null byte 拒绝                                     │
│    ├─ URL 编码遍历检测（%2e%2e%2f = ../）                   │
│    ├─ Unicode 范式化攻击检测（全角 ../）                   │
│    └─ 反斜杠拒绝（Windows 路径分隔符）                      │
│                                                          │
│  第二层：字符串级边界检查                                   │
│    └─ resolvedPath.startsWith(teamDir + sep)             │
│                                                          │
│  第三层：符号链接穿透检测                                   │
│    └─ realpathDeepestExisting() 递归向上解析              │
│    └─ lstat 检测 dangling symlink（悬空符号链接）           │
│    └─ ELOOP 检测循环链接                                   │
│    └─ realpath 后的路径再次验证边界                         │
│                                                          │
│  防御场景：                                               │
│    - team/../../etc/passwd → 第二层拒绝                    │
│    - team/evil -> /root/hidden → 第三层拒绝                │
│    - team/$(whoami)/../secrets → 第一层拒绝                │
└──────────────────────────────────────────────────────────┘
```

---

## 2. 记忆写入：主动识别与持久化

### 2.1 主 Agent 的主动写入

主 Agent 的系统 prompt 中嵌入了详细的记忆写入指导。在对话过程中，Agent 会主动识别值得持久化的知识，并通过 Write/Edit 工具直接写入文件，**两步骤完成**：

1. 将记忆内容写入独立文件（`user_role.md`、`feedback_testing.md` 等）
2. 在 MEMORY.md 中添加一条索引

这是 Agent 在对话中**主动完成的**，无需额外触发。

### 2.2 记忆提取子代理：兜底机制

**文件**：`src/services/extractMemories/extractMemories.ts`、`src/services/extractMemories/prompts.ts`

即使主 Agent 在某个 turn 没有主动写入，记忆提取子代理（Extract Memories Subagent）也会在每个查询循环结束时自动运行。核心流程：

```
┌────────────────────────────────────────────────────────────────┐
│              记忆提取子代理生命周期                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  触发时机：handleStopHooks（每次 query 循环完成时）               │
│  执行模式：forked agent —— 完美复制父对话，共享 prompt cache       │
│                                                                │
│  Step 1: 检查主 Agent 是否已写入记忆                             │
│    └→ hasMemoryWritesSince() 扫描 tool_use 块                   │
│    └→ 如果是 → 跳过子代理，推进游标                               │
│                                                                │
│  Step 2: 构建提取 prompt                                        │
│    └→ 告知子代理本次处理的新消息数量                              │
│    └→ 注入现有记忆文件列表（避免重复）                            │
│    └→ 限制 turn 预算：最多 5 轮                                  │
│                                                                │
│  Step 3: 执行提取                                               │
│    └→ runForkedAgent() — 并行 fork                             │
│    └→ 工具集：Read/Grep/Glob/只读 Bash + Edit/Write（限记忆目录）│
│    └→ 提取路径：写入了哪些记忆文件？                              │
│                                                                │
│  Step 4: 推进游标                                               │
│    └→ lastMemoryMessageUuid = 最新消息 UUID                      │
│                                                                │
│  防重机制：                                                    │
│    - 主 Agent 已写入 → 子代理跳过                               │
│    - 新消息计数为 0 → 子代理跳过                                 │
│    - 提取进行中 → 暂存上下文，结束后追加 trailing run            │
└────────────────────────────────────────────────────────────────┘
```

**Extract 子代理的 prompt 设计**体现了几个精妙的设计决策：

```typescript
// 效率策略引导：一次并行读，依次并行写
"You MUST only use content from the last ~${newMessageCount} messages " +
"to update your persistent memories. Do not waste any turns " +
"attempting to investigate or verify that content further — " +
"no grepping source files, no reading code to confirm a pattern exists"
```

子代理被严格限定为**仅从对话内容提取**，不主动查证代码库。这避免了两个问题：
1. 不引入额外工具调用的上下文开销
2. 防止提取变成调查工作而非记忆工作

### 2.3 防重复写入机制

主 Agent 和子代理通过**游标机制**实现互斥：

```
上次提取位置（lastMemoryMessageUuid）
       ↓
[已处理消息] | [新消息] ← 子代理/主 Agent 处理这个区间
```

- 主 Agent 在处理过程中写入记忆 → 子代理检测到 `hasMemoryWritesSince()` → 跳过，推进游标
- 主 Agent 未写入 → 子代理处理新消息区间
- 子代理运行中新消息到达 → 暂存上下文 → 当前提取完成后追加 trailing run

---

## 3. 按需检索：Sonnet Selector

### 3.1 检索时机：用户查询前

**文件**：`src/utils/attachments.ts`、`src/query.ts`

记忆检索发生在每次用户 turn 之前，作为异步预取（prefetch）：

```typescript
// src/query.ts — 与主查询并行执行
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
  state.messages,
  state.toolUseContext,
)
```

`startRelevantMemoryPrefetch()` 在后台启动 Sonnet 驱动的选择流程，不阻塞主查询。

### 3.2 两阶段检索流程

**文件**：`src/memdir/findRelevantMemories.ts`、`src/memdir/memoryScan.ts`

```
┌──────────────────────────────────────────────────────────────┐
│              两阶段记忆检索流程                                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  阶段 1：scanMemoryFiles() — 扫描所有 .md 文件                  │
│    ├─ readdir(memoryDir, { recursive: true })               │
│    ├─ 过滤 *.md，排除 MEMORY.md                               │
│    ├─ 对每个文件读取前 30 行（frontmatter）                    │
│    ├─ 解析 frontmatter → 提取 filename/description/type      │
│    ├─ 按 mtime 降序排列                                        │
│    └─ 最多返回 200 个文件                                      │
│                                                              │
│  阶段 2：selectRelevantMemories() — Sonnet 选择               │
│    ├─ 构造 manifest：每行一个文件 [type] filename (时间戳): description │
│    ├─ 注入最近使用的工具列表（避免选择工具文档类记忆）           │
│    ├─ 调用 sideQuery(Sonnet)                                  │
│    ├─ 指定 output_format: json_schema                         │
│    └─ 返回最多 5 个选中的文件名                                  │
│                                                              │
│  关键约束：                                                   │
│    - 只选"确信有用"的记忆，不确定则不选                         │
│    - 最近使用过的工具的参考文档不选（对话中已有）                │
│    - 工具警告/gotcha/已知问题仍选（使用中正是关键时机）          │
└──────────────────────────────────────────────────────────────┘
```

Sonnet Selector 的 system prompt 精确定义了选择标准：

```
Return a list of filenames for the memories that will clearly be
useful to Claude Code as it processes the user's query (up to 5).
Only include memories that you are certain will be helpful based
on their name and description.
- If you are unsure if a memory will be useful, do not include it.
- If there are no memories that would clearly be useful, return [].
- Recently used tools' reference docs → DO NOT select.
- Memories with warnings/gotchas about those tools → DO select.
```

### 3.3 记忆新鲜度标注

**文件**：`src/memdir/memoryAge.ts`

被选中的记忆文件在注入上下文时，会根据文件 mtime 添加新鲜度标注：

```typescript
// mtime ≤ 1 天 → 无标注
// mtime > 1 天 → 添加过期警告
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

同时，系统 prompt 中嵌入了"信任记忆"指南：

```
## Before recommending from memory

A memory that names a specific function, file, or flag is a claim
that it existed *when the memory was written*. It may have been
renamed, removed, or never merged. Before recommending it:

- If the memory names a file path: check the file exists.
- If the memory names a function or flag: grep for it.
- If the user is about to act on your recommendation → verify first.

"A memory says X exists" is not "X exists now."
```

### 3.4 记忆去重与截断保护

**文件**：`src/memdir/memdir.ts`

- `alreadySurfaced` 集合：在同一查询中，已被主 Agent 直接引用的记忆不重复加载
- `MAX_ENTRYPOINT_LINES = 200`：MEMORY.md 超过 200 行则截断
- `MAX_ENTRYPOINT_BYTES = 25_000`：MEMORY.md 超过 25KB 则截断到最后一个换行符

---

## 4. 系统集成：记忆与查询循环的协作

### 4.1 Stop Hooks 中的提取触发

**文件**：`src/services/extractMemories/extractMemories.ts`

记忆提取通过 `handleStopHooks` 在每次 query 循环结束时触发：

```
query 循环完成（LLM 自然结束，无 tool_use 块）
       ↓
handleStopHooks()
       ↓
executeExtractMemories()  ← fire-and-forget
       ↓
runExtraction() → runForkedAgent()
       ↓
drainPendingExtraction()  ← 在 shutdown 路径等待最多 60s
```

### 4.2 附件消息注入

被选中的记忆文件通过 `<system-reminder>` 附件注入到用户消息前：

```
用户消息
  └→ [相关记忆附件 1]
  └→ [相关记忆附件 2]
  └→ [相关记忆附件 N]
  └→ (主查询执行)
```

每个附件包含新鲜度标注和时间信息，使 LLM 能够在引用记忆时意识到其时效性。

### 4.3 功能开关体系

记忆系统的各组件通过 GrowthBook feature flags 控制实验开启：

| Feature Flag | 控制功能 |
|-------------|---------|
| `tengu_passport_quail` | 记忆提取子代理是否运行 |
| `tengu_herring_clock` | 团队记忆是否启用 |
| `tengu_moth_copse` | 是否跳过 MEMORY.md 索引维护 |
| `tengu_bramble_lintel` | 提取频率（每 N 个 turn 执行一次） |
| `tengu_coral_fern` | 搜索历史上下文功能 |
| `MEMORY_SHAPE_TELEMETRY` | 记忆召回形状遥测 |

---

## 5. 记忆系统全景

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Claude Code 记忆系统全景                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  存储层（磁盘）                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ ~/.claude/projects/<project>/memory/                                  │  │
│  │   MEMORY.md          ← 索引（每条一行，200行上限）                        │  │
│  │   user_role.md       ← 独立记忆文件（frontmatter + 内容）               │  │
│  │   feedback_*.md                                                 │  │
│  │   project_*.md                                                 │  │
│  │   reference_*.md                                               │  │
│  │   team/                                                         │  │
│  │     MEMORY.md       ← 团队索引                                       │  │
│  │     *.md            ← 团队记忆                                        │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              ↑写入                        ↑写入             │
│  ┌─────────────────┐    ┌──────────────────┐    ┌──────────────────────┐   │
│  │   主 Agent       │    │  提取子代理        │    │  Sonnet Selector     │   │
│  │  (主动识别)       │    │ (stop hooks 触发) │    │  (每轮预取)           │   │
│  └─────────────────┘    └──────────────────┘    └──────────────────────┘   │
│           │                    │                         │              │
│           └──────────┬─────────┘                         │              │
│                      ↓                                       ↓              │
│               ┌─────────────────────────────────┐    ┌────────────┐       │
│               │       记忆类型约束系统             │    │  选择相关   │       │
│               │  user / feedback / project / ref │    │  记忆文件   │       │
│               │  + WHAT_NOT_TO_SAVE 边界          │    │  (≤5 个)   │       │
│               └─────────────────────────────────┘    └────────────┘       │
│                                                              ↓              │
│                                                      ┌────────────┐         │
│                                                      │ system-    │         │
│                                                      │ reminder   │         │
│                                                      │ 附件注入    │         │
│                                                      └────────────┘         │
│                                                             ↓              │
│  查询层                                                               ←───────┘
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ query() 循环                                                           │  │
│  │   1. prependUserContext → 相关记忆作为附件注入用户消息前                │  │
│  │   2. callModel() → LLM 感知记忆上下文                                  │  │
│  │   3. 工具执行 / 结果处理                                               │  │
│  │   4. handleStopHooks() → 触发提取子代理                               │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 核心设计思想总结

### 6.1 索引与内容分离

MEMORY.md 只做索引，不存储内容。这解决了两个问题：
- **上下文容量**：记忆内容按需加载，最多只注入 5 个文件
- **选择性加载**：Sonnet Selector 确保只有真正相关的记忆被加载

### 6.2 主动写入与被动提取的互补

主 Agent 主动写入 + Extract 子代理兜底，两者通过游标互斥。Agent 在对话中自然识别知识时直接写入；遗漏的部分由子代理在循环结束时补全。

### 6.3 严格类型约束防止记忆腐化

四种封闭类型 + 明确的"什么不该存"边界，确保记忆库只保存**不可从代码推导的上下文**。超过 1 天的记忆自动标注新鲜度，进一步防止过时断言。

### 6.4 安全边界是架构设计的一部分

团队记忆的路径验证不仅是安全补丁，而是被嵌入到目录结构和验证管道的核心设计中——三层验证（字符串边界 → resolve → realpath）的复杂度反映了安全与可用性之间的精确平衡。

---

> **系列导航**
> - 上一篇：[Claude Code 多Agent调度：用户消息到子Agent的全链路解析]({{< ref "/posts/cc-arch-read/01_agent-scheduling" >}})
> - 下一篇：待续
