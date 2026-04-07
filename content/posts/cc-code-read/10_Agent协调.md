---
title: "Claude Code Agent 协调系统源码深度解读"
date: 2026-04-03
draft: false
authors: ["钟子期"]
---

# Claude Code Agent 协调系统源码深度解读

> 源码路径：`/mnt/e/code/cc/claude-code-main`  
> 核心文件：`src/tools/AgentTool/`, `src/tasks/LocalAgentTask/`, `src/utils/forkedAgent.ts`, `src/coordinator/`  
> 技术栈：Bun + TypeScript (strict) + AsyncLocalStorage

---

## 概述

Claude Code 的 Agent 协调系统是一个复杂的多层架构，支持：
- **单 Agent 执行**：基础 Agent 运行机制
- **子 Agent 派生**：父 Agent 通过 `AgentTool` 派生子 Agent
- **分支 Fork**：复制父级上下文创建并行工作单元
- **团队协作 (Teammates)**：多 Agent 协作模式
- **协调者模式 (Coordinator)**：主从架构的工作分配与同步

---

## 1. Agent 架构设计

### 1.1 Agent 定义类型体系

核心类型定义在 `src/tools/AgentTool/loadAgentsDir.ts`：

```typescript
// 基础类型 - 所有 Agent 的公共字段
export type BaseAgentDefinition = {
  agentType: string
  whenToUse: string
  tools?: string[]           // 允许的工具
  disallowedTools?: string[]  // 禁止的工具
  skills?: string[]           // 预加载的 Skills
  mcpServers?: AgentMcpServerSpec[]  // Agent 专用的 MCP 服务器
  hooks?: HooksSettings       // 生命周期钩子
  color?: AgentColorName      // UI 颜色标识
  model?: string              // 模型选择
  effort?: EffortValue        // 努力级别
  permissionMode?: PermissionMode  // 权限模式
  maxTurns?: number           // 最大轮次限制
  memory?: AgentMemoryScope   // 持久化内存范围
  isolation?: 'worktree' | 'remote'  // 隔离模式
  omitClaudeMd?: boolean      // 省略 CLAUDE.md 上下文
}

// 内置 Agent - 动态系统提示
export type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  getSystemPrompt: (params) => string  // 动态生成
}

// 自定义 Agent - 来自用户/项目/策略配置
export type CustomAgentDefinition = BaseAgentDefinition & {
  source: SettingSource
  getSystemPrompt: () => string  // 闭包存储
}

// 插件 Agent
export type PluginAgentDefinition = BaseAgentDefinition & {
  source: 'plugin'
  plugin: string
}
```

### 1.2 内置 Agent 类型

内置 Agent 定义在 `src/tools/AgentTool/builtInAgents.ts`：

```typescript
export function getBuiltInAgents(): AgentDefinition[] {
  const agents: AgentDefinition[] = [
    GENERAL_PURPOSE_AGENT,   // 通用 Agent
    STATUSLINE_SETUP_AGENT,   // 状态栏设置
  ]

  if (areExplorePlanAgentsEnabled()) {
    agents.push(EXPLORE_AGENT, PLAN_AGENT)  // 探索和规划 Agent
  }

  // 非 SDK 入口点包含代码指南 Agent
  if (isNonSdkEntrypoint) {
    agents.push(CLAUDE_CODE_GUIDE_AGENT)
  }

  // 验证 Agent (GrowthBook 控制)
  if (feature('VERIFICATION_AGENT') && tengu_hive_evidence) {
    agents.push(VERIFICATION_AGENT)
  }

  return agents
}
```

### 1.3 专业内置 Agent 详解

#### General Purpose Agent
```typescript
// src/tools/AgentTool/built-in/generalPurposeAgent.ts
export const GENERAL_PURPOSE_AGENT: BuiltInAgentDefinition = {
  agentType: 'general-purpose',
  tools: ['*'],  // 全部工具
  source: 'built-in',
  getSystemPrompt: getGeneralPurposeSystemPrompt,
}
```

#### Explore Agent (只读搜索)
```typescript
// src/tools/AgentTool/built-in/exploreAgent.ts
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  disallowedTools: [
    AGENT_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,
    FILE_WRITE_TOOL_NAME,
    NOTEBOOK_EDIT_TOOL_NAME,
  ],
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true,  // 省略 CLAUDE.md 节省 token
}
```

#### Plan Agent (架构规划)
```typescript
// src/tools/AgentTool/built-in/planAgent.ts
export const PLAN_AGENT: BuiltInAgentDefinition = {
  agentType: 'Plan',
  tools: EXPLORE_AGENT.tools,  // 与 Explore 相同工具集
  model: 'inherit',
  omitClaudeMd: true,
  // 输出格式要求包含 "Critical Files for Implementation"
}
```

#### Verification Agent (对抗性验证)
```typescript
// src/tools/AgentTool/built-in/verificationAgent.ts
// 特点：
// 1. 对抗性探测 - 尝试破坏实现
// 2. 必须运行命令验证，不能仅读代码
// 3. 输出格式：每个检查必须包含 "Command run" 和 "Output observed"
// 4. 结束判定：VERDICT: PASS | FAIL | PARTIAL
// 5. 必须包含至少一个对抗性探测
```

### 1.4 Agent 注册流程

```typescript
// src/tools/AgentTool/loadAgentsDir.ts
export const getAgentDefinitionsWithOverrides = memoize(async (cwd) => {
  // 1. 加载 Markdown 文件中的 Agent 定义
  const markdownFiles = await loadMarkdownFilesForSubdir('agents', cwd)
  
  // 2. 解析 Agent 定义
  const customAgents = markdownFiles.map(({ filePath, frontmatter, content, source }) => {
    return parseAgentFromMarkdown(filePath, baseDir, frontmatter, content, source)
  })
  
  // 3. 并行加载插件 Agent
  const pluginAgentsPromise = loadPluginAgents()
  
  // 4. 合并所有 Agent
  const allAgentsList: AgentDefinition[] = [
    ...builtInAgents,
    ...pluginAgents,
    ...customAgents,
  ]
  
  // 5. 按优先级去重
  const activeAgents = getActiveAgentsFromList(allAgentsList)
})
```

---

## 2. 任务分解与调度

### 2.1 任务类型体系

```typescript
// src/Task.ts
export type TaskType =
  | 'local_bash'        // 本地 Bash 任务
  | 'local_agent'       // 本地 Agent 任务
  | 'remote_agent'      // 远程 Agent 任务
  | 'in_process_teammate' // 进程内团队成员
  | 'local_workflow'     // 本地工作流
  | 'monitor_mcp'       // MCP 监控
  | 'dream'             // 梦境任务

export type TaskStatus =
  | 'pending'
  | 'running'
  | 'completed'
  | 'failed'
  | 'killed'
```

### 2.2 本地 Agent 任务状态

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx
export type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  selectedAgent?: AgentDefinition
  agentType: string
  model?: string
  abortController?: AbortController
  result?: AgentToolResult
  progress?: AgentProgress
  retrieved: boolean
  messages?: Message[]
  isBackgrounded: boolean  // 是否已后台化
  pendingMessages: string[]  // 待处理消息队列
  retain: boolean  // UI 是否持有此任务
  diskLoaded: boolean  // 磁盘是否已加载
  evictAfter?: number  // 驱逐时间戳
}
```

### 2.3 任务注册与调度

#### 异步 Agent 注册
```typescript
export function registerAsyncAgent({
  agentId,
  description,
  prompt,
  selectedAgent,
  setAppState,
  parentAbortController,
  toolUseId
}): LocalAgentTaskState {
  // 1. 初始化任务输出文件
  void initTaskOutputAsSymlink(agentId, getAgentTranscriptPath(asAgentId(agentId)))

  // 2. 创建 AbortController (链接到父级或独立)
  const abortController = parentAbortController
    ? createChildAbortController(parentAbortController)
    : createAbortController()

  // 3. 创建任务状态
  const taskState: LocalAgentTaskState = {
    ...createTaskStateBase(agentId, 'local_agent', description, toolUseId),
    type: 'local_agent',
    status: 'running',
    agentId,
    prompt,
    selectedAgent,
    agentType: selectedAgent.agentType ?? 'general-purpose',
    abortController,
    isBackgrounded: true,  // 立即后台化
    pendingMessages: [],
    retain: false,
    diskLoaded: false
  }

  // 4. 注册清理函数
  const unregisterCleanup = registerCleanup(async () => {
    killAsyncAgent(agentId, setAppState)
  })

  // 5. 注册到 AppState
  registerTask(taskState, setAppState)
  return taskState
}
```

#### 前台 Agent 注册（可后台化）
```typescript
export function registerAgentForeground({
  agentId,
  description,
  prompt,
  selectedAgent,
  setAppState,
  autoBackgroundMs,  // 自动后台化超时
  toolUseId
}): { taskId: string; backgroundSignal: Promise<void>; cancelAutoBackground?: () => void } {
  // 创建任务（前台模式）
  const taskState: LocalAgentTaskState = {
    ...createTaskStateBase(agentId, 'local_agent', description, toolUseId),
    type: 'local_agent',
    status: 'running',
    isBackgrounded: false,  // 前台运行
    pendingMessages: [],
    retain: false,
    diskLoaded: false
  }

  // 创建后台化信号 Promise
  let resolveBackgroundSignal: () => void
  const backgroundSignal = new Promise<void>(resolve => {
    resolveBackgroundSignal = resolve
  })
  backgroundSignalResolvers.set(agentId, resolveBackgroundSignal!)

  // 自动后台化定时器
  if (autoBackgroundMs !== undefined && autoBackgroundMs > 0) {
    const timer = setTimeout((setAppState, agentId) => {
      // 标记为后台化并解决信号
      setAppState(prev => {
        const prevTask = prev.tasks[agentId]
        if (!isLocalAgentTask(prevTask) || prevTask.isBackgrounded) {
          return prev
        }
        return {
          ...prev,
          tasks: { ...prev.tasks, [agentId]: { ...prevTask, isBackgrounded: true } }
        }
      })
      resolveBackgroundSignal!()
    }, autoBackgroundMs, setAppState, agentId)
    cancelAutoBackground = () => clearTimeout(timer)
  }

  return { taskId: agentId, backgroundSignal, cancelAutoBackground }
}
```

### 2.4 进度追踪

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx
export type ProgressTracker = {
  toolUseCount: number
  latestInputTokens: number      // API 累计值
  cumulativeOutputTokens: number  // 每轮求和
  recentActivities: ToolActivity[]  // 最近活动（最多 5 条）
}

export function updateProgressFromMessage(
  tracker: ProgressTracker,
  message: Message,
  resolveActivityDescription?: ActivityDescriptionResolver,
  tools?: Tools
): void {
  if (message.type !== 'assistant') return

  // 更新 Token 计数
  const usage = message.message.usage
  tracker.latestInputTokens = usage.input_tokens + cache_creation + cache_read
  tracker.cumulativeOutputTokens += usage.output_tokens

  // 记录工具使用
  for (const content of message.message.content) {
    if (content.type === 'tool_use') {
      tracker.toolUseCount++
      tracker.recentActivities.push({
        toolName: content.name,
        input: content.input,
        activityDescription: resolveActivityDescription?.(content.name, input),
        isSearch: classification?.isSearch,
        isRead: classification?.isRead
      })
    }
  }

  // 保持最近 5 条
  while (tracker.recentActivities.length > MAX_RECENT_ACTIVITIES) {
    tracker.recentActivities.shift()
  }
}
```

---

## 3. 多轮对话协调

### 3.1 runAgent - 核心执行循环

```typescript
// src/tools/AgentTool/runAgent.ts
export async function* runAgent({
  agentDefinition,
  promptMessages,
  toolUseContext,
  canUseTool,
  isAsync,
  querySource,
  override,
  model,
  maxTurns,
  availableTools,
  allowedTools,
  onCacheSafeParams,
  contentReplacementState,
  useExactTools,
  worktreePath,
  description,
  transcriptSubdir,
  onQueryProgress,
}): AsyncGenerator<Message, void> {
  // 1. 创建 Agent ID
  const agentId = override?.agentId ?? createAgentId()

  // 2. 初始化 Agent 上下文
  const resolvedAgentModel = getAgentModel(
    agentDefinition.model,
    toolUseContext.options.mainLoopModel,
    model,
    permissionMode
  )

  // 3. 处理消息分叉（Fork 子代理）
  const contextMessages: Message[] = forkContextMessages
    ? filterIncompleteToolCalls(forkContextMessages)
    : []
  const initialMessages: Message[] = [...contextMessages, ...promptMessages]

  // 4. 创建文件状态缓存
  const agentReadFileState = forkContextMessages !== undefined
    ? cloneFileStateCache(toolUseContext.readFileState)
    : createFileStateCacheWithSizeLimit(READ_FILE_STATE_CACHE_SIZE)

  // 5. 解析用户/系统上下文
  const [baseUserContext, baseSystemContext] = await Promise.all([
    override?.userContext ?? getUserContext(),
    override?.systemContext ?? getSystemContext()
  ])

  // 6. 优化上下文（节省 token）
  const shouldOmitClaudeMd = agentDefinition.omitClaudeMd && !override?.userContext
  const resolvedUserContext = shouldOmitClaudeMd ? userContextNoClaudeMd : baseUserContext

  // 7. 解析工具
  const resolvedTools = useExactTools
    ? availableTools
    : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools

  // 8. 构建系统提示
  const agentSystemPrompt = override?.systemPrompt
    ? override.systemPrompt
    : asSystemPrompt(await getAgentSystemPrompt(...))

  // 9. 初始化 Agent 专用 MCP 服务器
  const { clients: mergedMcpClients, tools: agentMcpTools, cleanup: mcpCleanup } =
    await initializeAgentMcpServers(agentDefinition, toolUseContext.options.mcpClients)

  // 10. 构建子代理上下文
  const agentToolUseContext = createSubagentContext(toolUseContext, {
    options: agentOptions,
    agentId,
    agentType: agentDefinition.agentType,
    messages: initialMessages,
    readFileState: agentReadFileState,
    abortController: agentAbortController,
    getAppState: agentGetAppState,
    shareSetAppState: !isAsync,
    contentReplacementState
  })

  // 11. 执行查询循环
  try {
    for await (const message of query({
      messages: initialMessages,
      systemPrompt: agentSystemPrompt,
      userContext: resolvedUserContext,
      systemContext: resolvedSystemContext,
      canUseTool,
      toolUseContext: agentToolUseContext,
      querySource,
      maxTurns: maxTurns ?? agentDefinition.maxTurns
    })) {
      // 转发 API 请求开始事件到父级指标
      if (message.type === 'stream_event' && message.event.type === 'message_start') {
        toolUseContext.pushApiMetricsEntry?.(message.ttftMs)
        continue
      }

      // 记录消息到边链转录本
      if (isRecordableMessage(message)) {
        await recordSidechainTranscript([message], agentId, lastRecordedUuid)
        yield message
      }
    }
  } finally {
    // 清理资源
    await mcpCleanup()
    if (agentDefinition.hooks) clearSessionHooks(rootSetAppState, agentId)
    cleanupAgentTracking(agentId)
    agentToolUseContext.readFileState.clear()
    unregisterPerfettoAgent(agentId)
  }
}
```

### 3.2 子代理上下文隔离

```typescript
// src/utils/forkedAgent.ts
export function createSubagentContext(
  parentContext: ToolUseContext,
  overrides?: SubagentContextOverrides
): ToolUseContext {
  // AbortController: 显式覆盖 > 共享父级 > 创建子级
  const abortController = overrides?.abortController
    ?? (overrides?.shareAbortController
      ? parentContext.abortController
      : createChildAbortController(parentContext.abortController))

  // getAppState: 包装以设置 shouldAvoidPermissionPrompts
  const getAppState = overrides?.getAppState
    ? overrides.getAppState
    : overrides?.shareAbortController
      ? parentContext.getAppState
      : () => {
          const state = parentContext.getAppState()
          if (state.toolPermissionContext.shouldAvoidPermissionPrompts) return state
          return {
            ...state,
            toolPermissionContext: {
              ...state.toolPermissionContext,
              shouldAvoidPermissionPrompts: true
            }
          }
        }

  return {
    // 可变状态 - 默认克隆以保持隔离
    readFileState: cloneFileStateCache(overrides?.readFileState ?? parentContext.readFileState),
    nestedMemoryAttachmentTriggers: new Set<string>(),
    loadedNestedMemoryPaths: new Set<string>(),
    dynamicSkillDirTriggers: new Set<string>(),
    discoveredSkillNames: new Set<string>(),
    toolDecisions: undefined,
    contentReplacementState: overrides?.contentReplacementState
      ?? (parentContext.contentReplacementState
        ? cloneContentReplacementState(parentContext.contentReplacementState)
        : undefined),

    abortController,
    getAppState,
    setAppState: overrides?.shareSetAppState ? parentContext.setAppState : () => {},
    setAppStateForTasks: parentContext.setAppStateForTasks ?? parentContext.setAppState,
    localDenialTracking: overrides?.shareSetAppState
      ? parentContext.localDenialTracking
      : createDenialTrackingState(),

    // 变异回调 - 默认无操作
    setInProgressToolUseIDs: () => {},
    setResponseLength: overrides?.shareSetResponseLength ? parentContext.setResponseLength : () => {},
    pushApiMetricsEntry: overrides?.shareSetResponseLength ? parentContext.pushApiMetricsEntry : undefined,
    updateFileHistoryState: () => {},
    updateAttributionState: parentContext.updateAttributionState,

    // UI 回调 - 子代理未定义（无法控制父级 UI）
    addNotification: undefined,
    setToolJSX: undefined,
    setStreamMode: undefined,
    setSDKStatus: undefined,
    openMessageSelector: undefined,

    // 可覆盖字段
    options: overrides?.options ?? parentContext.options,
    messages: overrides?.messages ?? parentContext.messages,
    agentId: overrides?.agentId ?? createAgentId(),
    agentType: overrides?.agentType,

    // 查询追踪链
    queryTracking: {
      chainId: randomUUID(),
      depth: (parentContext.queryTracking?.depth ?? -1) + 1
    }
  }
}
```

### 3.3 Fork 分支子代理

```typescript
// src/tools/AgentTool/forkSubagent.ts
export const FORK_AGENT: BuiltInAgentDefinition = {
  agentType: 'fork',
  tools: ['*'],        // 使用父级精确工具池
  maxTurns: 200,
  model: 'inherit',    // 继承父级模型
  permissionMode: 'bubble',  // 权限气泡到父级终端
  source: 'built-in',
  getSystemPrompt: () => ''  // 未使用，传递 systemPrompt 覆盖
}

// 构建分支消息
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage
): MessageType[] {
  // 1. 克隆完整的 assistant 消息（保留所有 tool_use 块）
  const fullAssistantMessage: AssistantMessage = {
    ...assistantMessage,
    uuid: randomUUID(),
    message: {
      ...assistantMessage.message,
      content: [...assistantMessage.message.content]
    }
  }

  // 2. 收集所有 tool_use 块
  const toolUseBlocks = assistantMessage.message.content.filter(
    (block): block is BetaToolUseBlock => block.type === 'tool_use'
  )

  // 3. 为每个 tool_use 构建占位符 tool_result
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content: [{ type: 'text' as const, text: FORK_PLACEHOLDER_RESULT }]
  }))

  // 4. 构建用户消息：所有占位符结果 + 每个子代理的指令
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,
      { type: 'text' as const, text: buildChildMessage(directive) }
    ]
  })

  return [fullAssistantMessage, toolResultMessage]
}

export function buildChildMessage(directive: string): string {
  return `<${FORK_BOILERPLATE_TAG}>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Do NOT spawn sub-agents; execute directly.
2. Do NOT converse, ask questions, or suggest next steps
3. USE your tools directly: Bash, Read, Write, etc.
4. If you modify files, commit your changes before reporting.
5. Keep your report under 500 words.
6. Your response MUST begin with "Scope:".
</${FORK_BOILERPLATE_TAG}>
${FORK_DIRECTIVE_PREFIX}${directive}`
}
```

---

## 4. 子 Agent 管理

### 4.1 AgentTool - 代理调用入口

```typescript
// src/tools/AgentTool/AgentTool.tsx
export const AgentTool = buildTool({
  name: AGENT_TOOL_NAME,
  maxResultSizeChars: 100_000,

  async call({
    prompt,
    subagent_type,
    description,
    model: modelParam,
    run_in_background,
    name,           // 团队成员名称
    team_name,      // 团队名称
    mode: spawnMode,
    isolation,
    cwd
  }, toolUseContext, canUseTool, assistantMessage, onProgress?) {

    // 1. 多 Agent Spawn 检测 (team_name + name)
    if (teamName && name) {
      return spawnTeammate({ name, prompt, team_name, ... })
    }

    // 2. Fork 分支检测 (subagent_type 未指定)
    const isForkPath = effectiveType === undefined

    // 3. 解析选中的 Agent 定义
    let selectedAgent: AgentDefinition
    if (isForkPath) {
      selectedAgent = FORK_AGENT
    } else {
      selectedAgent = findAgentByType(effectiveType)
    }

    // 4. MCP 服务器检查
    if (requiredMcpServers?.length) {
      // 等待待处理的 MCP 服务器连接
      // 验证所需 MCP 服务器可用
    }

    // 5. 工作树隔离设置
    if (effectiveIsolation === 'worktree') {
      worktreeInfo = await createAgentWorktree(slug)
    }

    // 6. 决定同步/异步执行
    const shouldRunAsync = (
      run_in_background === true ||
      selectedAgent.background === true ||
      isCoordinator ||
      forceAsync ||
      assistantForceAsync
    ) && !isBackgroundTasksDisabled

    if (shouldRunAsync) {
      // 异步执行路径
      return runAsyncAgent(...)
    } else {
      // 同步执行路径
      return runSyncAgent(...)
    }
  }
})
```

### 4.2 团队成员 Spawn

```typescript
// src/tools/shared/spawnMultiAgent.ts
export async function spawnTeammate(
  config: SpawnTeammateConfig,
  context: ToolUseContext
): Promise<{ data: SpawnOutput }> {
  // 检测后端: in-process | tmux | iTerm2
  if (isInProcessEnabled()) {
    return handleSpawnInProcess(config, context)
  }

  // pane 后端可用，使用 split-pane view
  if (useSplitPane) {
    return handleSpawnSplitPane(config, context)
  }
  return handleSpawnSeparateWindow(config, context)
}

// 进程内团队成员
async function handleSpawnInProcess(
  input: SpawnInput,
  context: ToolUseContext
): Promise<{ data: SpawnOutput }> {
  // 1. 解析模型
  const model = resolveTeammateModel(input.model, getAppState().mainLoopModel)

  // 2. 生成唯一名称
  const uniqueName = await generateUniqueTeammateName(name, teamName)

  // 3. 生成确定性 Agent ID
  const teammateId = formatAgentId(sanitizedName, teamName)

  // 4. 分配颜色
  const teammateColor = assignTeammateColor(teammateId)

  // 5. Spawn 进程内团队成员
  const result = await spawnInProcessTeammate(config, context)

  // 6. 启动代理执行循环
  if (result.taskId && result.teammateContext && result.abortController) {
    startInProcessTeammate({
      identity: { agentId: teammateId, agentName: sanitizedName, ... },
      taskId: result.taskId,
      prompt,
      model,
      agentDefinition,
      teammateContext: result.teammateContext,
      toolUseContext: { ...context, messages: [] },  // 不继承父级消息
      abortController: result.abortController
    })
  }

  // 7. 注册到团队上下文
  setAppState(prev => ({
    ...prev,
    teamContext: {
      ...prev.teamContext,
      teammates: {
        ...prev.teamContext?.teammates,
        [teammateId]: { name: sanitizedName, agentType, color: teammateColor, ... }
      }
    }
  }))

  // 8. 注册到团队文件
  const teamFile = await readTeamFileAsync(teamName)
  teamFile.members.push({ agentId: teammateId, name: sanitizedName, ... })
  await writeTeamFileAsync(teamName, teamFile)

  return { data: { teammate_id: teammateId, ... } }
}
```

### 4.3 工具过滤与解析

```typescript
// src/tools/AgentTool/agentToolUtils.ts
export function filterToolsForAgent({
  tools,
  isBuiltIn,
  isAsync = false,
  permissionMode
}): Tools {
  return tools.filter(tool => {
    // MCP 工具始终允许
    if (tool.name.startsWith('mcp__')) return true

    // 计划模式下的 ExitPlanMode
    if (toolMatchesName(tool, EXIT_PLAN_MODE_V2_TOOL_NAME) && permissionMode === 'plan') {
      return true
    }

    // 全局禁用工具
    if (ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false

    // 自定义 Agent 禁用工具
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false

    // 异步 Agent 工具限制
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) {
      if (isAgentSwarmsEnabled() && isInProcessTeammate()) {
        if (toolMatchesName(tool, AGENT_TOOL_NAME)) return true
        if (IN_PROCESS_TEAMMATE_ALLOWED_TOOLS.has(tool.name)) return true
      }
      return false
    }

    return true
  })
}
```

---

## 5. 规划与推理机制

### 5.1 Coordinator 协调者模式

```typescript
// src/coordinator/coordinatorMode.ts
export function getCoordinatorSystemPrompt(): string {
  return `You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers.

## 1. Your Role

You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible — don't delegate work that you can handle without tools

## 2. Your Tools

- **AgentTool** - Spawn a new worker
- **SendMessageTool** - Continue an existing worker
- **TaskStopTool** - Stop a running worker

## 3. Workers

When calling AgentTool, use subagent_type \`worker\`. Workers execute tasks autonomously.

## 4. Task Workflow

Most tasks can be broken down into:
1. **Research** - Workers explore and gather information
2. **Implement** - Workers make changes based on research
3. **Verify** - Workers or coordinator verify correctness
`
}
```

### 5.2 团队成员生命周期

```typescript
// src/utils/swarm/inProcessRunner.ts
// 进程内团队成员执行循环
export async function startInProcessTeammate(config: InProcessTeammateConfig): Promise<void> {
  const {
    identity,
    taskId,
    prompt,
    model,
    agentDefinition,
    teammateContext,
    toolUseContext,
    abortController
  } = config

  // 创建团队成员专用上下文
  const teammateToolContext = createSubagentContext(toolUseContext, {
    options: {
      ...toolUseContext.options,
      mainLoopModel: model,
      isNonInteractiveSession: true
    },
    agentId: identity.agentId,
    agentType: identity.agentName,
    shareSetAppState: true,  // 共享状态更新
    shareSetResponseLength: true,
    shareAbortController: true  // 随主控中止
  })

  // 提交初始提示
  await writeToMailbox(identity.agentName, {
    from: TEAM_LEAD_NAME,
    text: prompt,
    timestamp: new Date().toISOString()
  }, identity.teamName)

  // 运行团队成员循环
  for await (const message of query({ ... })) {
    if (message.type === 'task-notification') {
      // 发送任务完成通知
    }
  }
}
```

### 5.3 定期进度总结

```typescript
// src/services/AgentSummary/agentSummary.ts
const SUMMARY_INTERVAL_MS = 30_000

export function startAgentSummarization(
  taskId: string,
  agentId: AgentId,
  cacheSafeParams: CacheSafeParams,
  setAppState: TaskContext['setAppState']
): { stop: () => void } {
  let stopped = false
  let previousSummary: string | null = null

  async function runSummary(): Promise<void> {
    if (stopped) return

    // 从转录本读取当前消息
    const transcript = await getAgentTranscript(agentId)
    if (!transcript || transcript.messages.length < 3) return

    // 构建总结提示
    const summaryPrompt = buildSummaryPrompt(previousSummary)

    // 运行分支代理进行总结（禁用工具）
    const result = await runForkedAgent({
      promptMessages: [createUserMessage({ content: summaryPrompt })],
      cacheSafeParams: { ...baseParams, forkContextMessages: cleanMessages },
      canUseTool: async () => ({ behavior: 'deny', message: 'No tools needed for summary' }),
      querySource: 'agent_summary',
      skipTranscript: true
    })

    // 提取总结文本
    for (const msg of result.messages) {
      if (msg.type !== 'assistant') continue
      const textBlock = msg.message.content.find(b => b.type === 'text')
      if (textBlock?.type === 'text' && textBlock.text.trim()) {
        previousSummary = textBlock.text.trim()
        updateAgentSummary(taskId, previousSummary, setAppState)
        break
      }
    }
  }

  function scheduleNext(): void {
    if (stopped) return
    timeoutId = setTimeout(runSummary, SUMMARY_INTERVAL_MS)
  }

  scheduleNext()
  return { stop: () => { stopped = true } }
}
```

---

## 6. 自反思（Self-Reflection）实现

### 6.1 Verification Agent 的对抗性验证

```typescript
// src/tools/AgentTool/built-in/verificationAgent.ts
const VERIFICATION_SYSTEM_PROMPT = `You are a verification specialist. Your job is not to confirm the implementation works — it's to try to break it.

You have two documented failure patterns. First, verification avoidance: when faced with a check, you find reasons not to run it. Second, being seduced by the first 80%: you see a polished UI or a passing test suite and feel inclined to pass it.

=== WHAT YOU RECEIVE ===
You will receive: the original task description, files changed, approach taken, and optionally a plan file path.

=== VERIFICATION STRATEGY ===
**Frontend changes**: Start dev server → check your tools for browser automation → USE them → curl subresources
**Backend/API changes**: Start server → curl/fetch endpoints → verify response shapes
**CLI/script changes**: Run with representative inputs → verify stdout/stderr/exit codes
**Bug fixes**: Reproduce the original bug → verify fix → run regression tests

=== REQUIRED STEPS (universal baseline) ===
1. Read the project's CLAUDE.md / README for build/test commands
2. Run the build (if applicable). A broken build is an automatic FAIL.
3. Run the project's test suite. Failing tests are an automatic FAIL.
4. Run linters/type-checkers

=== OUTPUT FORMAT (REQUIRED) ===
Every check MUST follow this structure:

### Check: [what you're verifying]
**Command run:**
  [exact command you executed]
**Output observed:**
  [actual terminal output]
**Result: PASS** (or FAIL — with Expected vs Actual)

End with exactly this line:
VERDICT: PASS | FAIL | PARTIAL
`
```

### 6.2 转录分类器（Transcript Classifier）

```typescript
// src/tools/AgentTool/agentToolUtils.ts
export async function classifyHandoffIfNeeded({
  agentMessages,
  tools,
  toolPermissionContext,
  abortSignal,
  subagentType,
  totalToolUseCount
}): Promise<string | null> {
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    if (toolPermissionContext.mode !== 'auto') return null

    // 构建分类器转录本
    const agentTranscript = buildTranscriptForClassifier(agentMessages, tools)
    if (!agentTranscript) return null

    // 分类代理行为
    const classifierResult = await classifyYoloAction(
      agentMessages,
      {
        role: 'user',
        content: [{
          type: 'text',
          text: "Sub-agent has finished and is handing back control..."
        }]
      },
      tools,
      toolPermissionContext as ToolPermissionContext,
      abortSignal
    )

    if (classifierResult.shouldBlock) {
      if (classifierResult.unavailable) {
        return `Note: The safety classifier was unavailable...`
      }
      return `SECURITY WARNING: This sub-agent performed actions that may violate security policy.`
    }
  }
  return null
}
```

### 6.3 部分结果提取

```typescript
// src/tools/AgentTool/agentToolUtils.ts
export function extractPartialResult(messages: MessageType[]): string | undefined {
  // 用于 Agent 被杀死时保留已完成的进度
  for (let i = messages.length - 1; i >= 0; i--) {
    const m = messages[i]!
    if (m.type !== 'assistant') continue
    const text = extractTextContent(m.message.content, '\n')
    if (text) return text
  }
  return undefined
}

export function finalizeAgentTool(
  agentMessages: MessageType[],
  agentId: string,
  metadata: { prompt, resolvedAgentModel, isBuiltInAgent, startTime, agentType, isAsync }
): AgentToolResult {
  const lastAssistantMessage = getLastAssistantMessage(agentMessages)

  // 提取文本内容
  let content = lastAssistantMessage.message.content.filter(_ => _.type === 'text')
  if (content.length === 0) {
    // 降落到最近的有文本内容的助手消息
    for (let i = agentMessages.length - 1; i >= 0; i--) {
      const m = agentMessages[i]!
      if (m.type !== 'assistant') continue
      const textBlocks = m.message.content.filter(_ => _.type === 'text')
      if (textBlocks.length > 0) { content = textBlocks; break }
    }
  }

  const totalTokens = getTokenCountFromUsage(lastAssistantMessage.message.usage)
  const totalToolUseCount = countToolUses(agentMessages)

  // 记录完成事件
  logEvent('tengu_agent_tool_completed', {
    agent_type: agentType,
    model: resolvedAgentModel,
    total_tool_uses: totalToolUseCount,
    duration_ms: Date.now() - startTime,
    total_tokens: totalTokens
  })

  return {
    agentId,
    agentType,
    content,
    totalDurationMs: Date.now() - startTime,
    totalTokens,
    totalToolUseCount,
    usage: lastAssistantMessage.message.usage
  }
}
```

---

## 7. 关键设计模式

### 7.1 AsyncLocalStorage 上下文隔离

使用 `createSubagentContext` 创建隔离的子代理上下文，确保：
- 文件状态缓存克隆
- AbortController 链接到父级
- AppState 访问被包装
- 变异回调设为无操作

### 7.2 提示缓存共享

分支子代理使用 `CacheSafeParams` 保证与父级 API 请求前缀字节一致：
- 相同的系统提示
- 相同的工具定义序列化
- 相同的模型和思考配置
- 相同的消息前缀

### 7.3 任务框架统一接口

```typescript
// src/Task.ts
export type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}

// 支持多种任务类型
const LocalAgentTask: Task = {
  name: 'LocalAgentTask',
  type: 'local_agent',
  async kill(taskId, setAppState) {
    killAsyncAgent(taskId, setAppState)
  }
}
```

### 7.4 后台任务生命周期

```
registerAsyncAgent() → registerTask() → backgroundSignal Promise
                                    ↓
                         runAgent() [异步执行]
                                    ↓
                         completeAsyncAgent() / failAsyncAgent()
                                    ↓
                         enqueueAgentNotification() → 消息队列
                                    ↓
                         任务状态更新 → evictTaskOutput()
```

---

## 8. 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AgentTool (入口)                            │
│  • 解析输入参数 (prompt, subagent_type, run_in_background, etc.)      │
│  • 路由决策: spawn | fork | async | sync                           │
│  • MCP 服务器检查 / 隔离模式设置                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          ▼                         ▼                         ▼
   ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
   │   spawn      │         │    fork      │         │    sync      │
   │  Teammate    │         │   Subagent   │         │   Agent      │
   └──────────────┘         └──────────────┘         └──────────────┘
          │                         │                         │
          ▼                         ▼                         ▼
   ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
   │ createTeam    │         │ buildForked │         │  runAgent()  │
   │  Context      │         │  Messages()  │         │  [同步迭代]   │
   └──────────────┘         └──────────────┘         └──────────────┘
          │                         │                         │
          ▼                         ▼                         ▼
   ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
   │ inProcess /   │         │ createSub    │         │  query()     │
   │  tmux / iTerm2│         │ agentContext │         │  [核心循环]   │
   └──────────────┘         └──────────────┘         └──────────────┘
                                    │                         │
                                    ▼                         ▼
                           ┌──────────────────┐      ┌──────────────────┐
                           │ CacheSafeParams   │      │ LocalAgentTask   │
                           │ (提示缓存共享)    │      │ (任务状态管理)   │
                           └──────────────────┘      └──────────────────┘
                                                         │
                                                         ▼
                                                ┌──────────────────┐
                                                │ registerAsync/   │
                                                │ registerForeground│
                                                └──────────────────┘
```

---

## 9. 核心文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/tools/AgentTool/AgentTool.tsx` | ~1500 | AgentTool 主实现 |
| `src/tools/AgentTool/runAgent.ts` | ~973 | Agent 执行循环 |
| `src/tools/AgentTool/loadAgentsDir.ts` | ~755 | Agent 定义加载 |
| `src/tools/AgentTool/agentToolUtils.ts` | ~686 | Agent 工具解析 |
| `src/tools/AgentTool/forkSubagent.ts` | ~210 | Fork 子代理 |
| `src/tools/AgentTool/resumeAgent.ts` | ~265 | Agent 恢复 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | ~683 | 本地任务管理 |
| `src/utils/forkedAgent.ts` | ~689 | 分支代理工具 |
| `src/services/AgentSummary/agentSummary.ts` | ~179 | 进度总结 |
| `src/tools/shared/spawnMultiAgent.ts` | ~1093 | 团队成员 Spawn |
| `src/coordinator/coordinatorMode.ts` | ~369 | 协调者模式 |

---

*文档版本：v1.0 | 更新：2026-04-03*
