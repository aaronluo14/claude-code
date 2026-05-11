# Claude Code Agent Loop Context 构建机制深度解析

> 基于 2026-03-31 泄露的 Claude Code 源码快照分析。
> 核心文件：`QueryEngine.ts`、`query.ts`、`context.ts`、`utils/api.ts`、`utils/queryContext.ts`、`constants/prompts.ts`

---

## 一、整体架构

Agent Loop 的 context 构建分布在三个层次：

```
QueryEngine.submitMessage()     ← 会话级编排（初始化 system prompt / context）
    ↓
query() → queryLoop()           ← 循环主体（while true，每次迭代 = 一次 API 调用）
    ↓
callModel()                     ← 实际 API 请求（组装最终 payload）
```

每轮发送给 API 的请求由三大部分组成：

```
┌──────────────────────────────────────────────────┐
│                  API Request                     │
│                                                  │
│  1. System Prompt（系统提示词）                    │
│  2. Messages（消息列表）                          │
│  3. Tools（工具定义 JSON Schema）                  │
└──────────────────────────────────────────────────┘
```

---

## 二、会话初始化阶段

当用户发送一条消息时，`QueryEngine.submitMessage()` 执行一次性初始化。

### 2.1 获取三件套

```typescript
// QueryEngine.ts:292
const {
  defaultSystemPrompt,
  userContext: baseUserContext,
  systemContext,
} = await fetchSystemPromptParts({
  tools,
  mainLoopModel: initialMainLoopModel,
  additionalWorkingDirectories,
  mcpClients,
  customSystemPrompt,
})
```

三者通过 `Promise.all` 并行获取（`queryContext.ts:61`）：

| 部分 | 来源函数 | 内容 |
|---|---|---|
| `defaultSystemPrompt` | `getSystemPrompt()` in `prompts.ts` | 静态段（角色定义、行为规范、工具使用指南、编码风格）+ 动态段（session_guidance、memory、env_info、language、MCP 指令等） |
| `userContext` | `getUserContext()` in `context.ts` | CLAUDE.md 文件内容 + 当前日期 |
| `systemContext` | `getSystemContext()` in `context.ts` | Git 状态快照（当前分支、主分支、`git status --short`、最近 5 次 commit、Git 用户名） |

### 2.2 System Prompt 组装

```typescript
// QueryEngine.ts:321-325
const systemPrompt = asSystemPrompt([
  ...(customPrompt !== undefined ? [customPrompt] : defaultSystemPrompt),
  ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
])
```

优先级层级（`systemPrompt.ts:41` 的 `buildEffectiveSystemPrompt`）：

```
0. overrideSystemPrompt（最高优先级，如 loop 模式）→ 完全替换所有其他提示词
1. coordinatorSystemPrompt（多 Agent 协调器模式激活时）
2. agentSystemPrompt（子 Agent 定义）
   - Proactive 模式：追加到默认提示词后面
   - 普通模式：替换默认提示词
3. customSystemPrompt（--system-prompt CLI 参数）
4. defaultSystemPrompt（标准 Claude Code 提示词）
+ appendSystemPrompt 始终追加到末尾（除 override 模式外）
```

### 2.3 defaultSystemPrompt 的详细组成

`getSystemPrompt()` in `prompts.ts:444` 返回一个字符串数组，结构如下：

```
┌─ 静态段（可缓存）─────────────────────────────────┐
│  getSimpleIntroSection()     角色定义              │
│  getSimpleSystemSection()    系统规范              │
│  getSimpleDoingTasksSection()  任务执行指南        │
│  getActionsSection()         行为规范              │
│  getUsingYourToolsSection()  工具使用指南          │
│  getSimpleToneAndStyleSection()  语气风格          │
│  getOutputEfficiencySection()  输出效率            │
├─ DYNAMIC BOUNDARY ────────────────────────────────┤
│  session_guidance      会话特定指引 + 技能工具命令  │
│  memory                持久化记忆提示词             │
│  env_info_simple       环境信息（OS、Shell、CWD）   │
│  language              语言设置                     │
│  output_style          输出风格配置                 │
│  mcp_instructions      MCP 服务器指令              │
│  scratchpad            草稿本指令                   │
│  frc                   函数结果清理                 │
│  summarize_tool_results  工具结果摘要指令           │
│  token_budget          Token 预算指令               │
└───────────────────────────────────────────────────┘
```

### 2.4 处理用户输入

```typescript
// QueryEngine.ts:410-431
const { messages: messagesFromUserInput, shouldQuery, allowedTools } =
  await processUserInput({ input: prompt, ... })

this.mutableMessages.push(...messagesFromUserInput)
const messages = [...this.mutableMessages]
```

处理后将参数传入 `query()` 启动循环：

```typescript
// QueryEngine.ts:675-686
for await (const message of query({
  messages,
  systemPrompt,
  userContext,
  systemContext,
  canUseTool: wrappedCanUseTool,
  toolUseContext: processUserInputContext,
  maxTurns,
  taskBudget,
  // ...
}))
```

---

## 三、循环主体 — 每轮 Context 的构建流水线

核心在 `queryLoop()` (query.ts:241)，是一个 `while (true)` 循环，每次迭代代表一次 API 调用。

### 3.1 流水线总览

```
messages（来自上一轮 state 或初始输入）
    │
    ├─ Step 1: getMessagesAfterCompactBoundary()  截取压缩边界后的消息
    │
    ├─ Step 2: applyToolResultBudget()            工具结果体积裁剪
    │
    ├─ Step 3: snipCompactIfNeeded()              Snip 轻量裁剪（feature flag）
    │
    ├─ Step 4: microcompact()                     微压缩
    │
    ├─ Step 5: applyCollapsesIfNeeded()           上下文折叠（feature flag）
    │
    ├─ Step 6: appendSystemContext()              systemContext 追加到 system prompt
    │
    ├─ Step 7: autocompact()                      自动全量压缩（按 token 阈值触发）
    │
    ├─ Step 8: prependUserContext()               userContext 注入消息列表头部
    │
    ├─ Step 9: callModel()                        调用 Anthropic API
    │
    ├─ Step 10: 流式接收 assistant 回复 + 收集 tool_use blocks
    │
    ├─ Step 11: runTools() / StreamingToolExecutor  执行工具，收集 tool_result
    │
    ├─ Step 12: getAttachmentMessages()            注入附件（文件差异、记忆、技能等）
    │
    └─ Step 13: 构建下一轮 state.messages
               = messagesForQuery + assistantMessages + toolResults
               → 回到 Step 1
```

### 3.2 各步骤详解

#### Step 1: 截取压缩边界后的消息

```typescript
// query.ts:365
let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]
```

`getMessagesAfterCompactBoundary` (messages.ts:4643) 找到最后一个 `compact_boundary` 类型的系统消息，只保留其后的消息。如果之前发生过压缩，压缩前的原始历史被丢弃，只留压缩摘要 + 后续对话。

#### Step 2: 工具结果体积裁剪

```typescript
// query.ts:379-394
messagesForQuery = await applyToolResultBudget(
  messagesForQuery,
  toolUseContext.contentReplacementState,
  // ...
)
```

对超大的工具返回结果（如巨大的文件内容）进行体积限制，避免单条消息占用过多 token。超过 `maxResultSizeChars` 的工具结果会被替换为摘要引用。

#### Step 3: Snip 压缩

```typescript
// query.ts:401-410
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
}
```

轻量级的历史裁剪。移除老旧的中间消息片段，释放 token 空间。与全量压缩不同，snip 不需要额外的 API 调用来生成摘要。

#### Step 4: Microcompact（微压缩）

```typescript
// query.ts:414-426
const microcompactResult = await deps.microcompact(
  messagesForQuery, toolUseContext, querySource,
)
messagesForQuery = microcompactResult.messages
```

对单条工具结果进行缓存编辑式的微压缩。比全量压缩更轻量，支持通过 `cache_deleted_input_tokens` 追踪实际节省。

#### Step 5: Context Collapse（上下文折叠）

```typescript
// query.ts:440-447
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(
    messagesForQuery, toolUseContext, querySource,
  )
  messagesForQuery = collapseResult.messages
}
```

将早期的消息分组折叠为摘要。这是一个读时投影（read-time projection），折叠摘要存储在 collapse store 中，不修改原始消息数组。优先于 auto compact 执行——如果折叠能把 token 数降到阈值以下，就不需要触发全量压缩。

#### Step 6: 组装完整 System Prompt

```typescript
// query.ts:449-451
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext),
)
```

`appendSystemContext` 把 `systemContext`（Git 状态快照）以 `key: value` 格式追加到 system prompt 末尾：

```typescript
// utils/api.ts:437-447
export function appendSystemContext(systemPrompt, context) {
  return [
    ...systemPrompt,
    Object.entries(context)
      .map(([key, value]) => `${key}: ${value}`)
      .join('\n'),
  ].filter(Boolean)
}
```

#### Step 7: Auto Compact（自动全量压缩）

```typescript
// query.ts:454-467
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery, toolUseContext,
  { systemPrompt, userContext, systemContext, toolUseContext, forkContextMessages },
  querySource, tracking, snipTokensFreed,
)
```

触发条件：token 数接近上下文窗口限制。有效窗口计算（`autoCompact.ts:33`）：

```
有效窗口 = contextWindow - min(modelMaxOutput, 20000)
```

压缩成功后，`messagesForQuery` 被替换为 `buildPostCompactMessages(compactionResult)`（摘要消息）。设有连续失败计数器作为熔断器。

#### Step 8: 注入 User Context 到消息列表头部

```typescript
// query.ts:660
messages: prependUserContext(messagesForQuery, userContext),
```

`prependUserContext` (api.ts:449) 在消息列表最前面插入一条 `<system-reminder>` 格式的用户消息：

```typescript
createUserMessage({
  content: `<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
${CLAUDE.md 的内容}
# currentDate
Today's date is 2026-05-11.

IMPORTANT: this context may or may not be relevant to your tasks.
You should not respond to this context unless it is highly relevant.
</system-reminder>`,
  isMeta: true,
})
```

这条消息标记为 `isMeta: true`，不会在 UI 中展示给用户，但会进入 API 请求。

#### Step 9: 调用 API

```typescript
// query.ts:659-708
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: {
    model: currentModel,
    fastMode: appState.fastMode,
    fallbackModel,
    querySource,
    maxOutputTokensOverride,
    taskBudget,
    effortValue: appState.effortValue,
    advisorModel: appState.advisorModel,
    // ...
  },
}))
```

#### Step 10: 流式接收 + 收集工具调用

流式接收 assistant 回复。遇到 `tool_use` 类型的 content block 时：

```typescript
// query.ts:826-844
if (message.type === 'assistant') {
  assistantMessages.push(message)
  const msgToolUseBlocks = message.message.content.filter(
    content => content.type === 'tool_use',
  ) as ToolUseBlock[]
  if (msgToolUseBlocks.length > 0) {
    toolUseBlocks.push(...msgToolUseBlocks)
    needsFollowUp = true  // 标记需要继续循环
  }
}
```

支持 **StreamingToolExecutor**：工具在流式接收期间就开始并行执行，而非等流完成后再执行。

#### Step 11: 执行工具 + 收集结果

```typescript
// query.ts:1380-1408
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message
    toolResults.push(
      ...normalizeMessagesForAPI([update.message], tools)
        .filter(_ => _.type === 'user'),
    )
  }
  if (update.newContext) {
    updatedToolUseContext = { ...update.newContext, queryTracking }
  }
}
```

工具结果通过 `normalizeMessagesForAPI` 标准化为 API 兼容的 `tool_result` 格式。

#### Step 12: 注入附件

```typescript
// query.ts:1580-1590
for await (const attachment of getAttachmentMessages(
  null, updatedToolUseContext, null,
  queuedCommandsSnapshot,
  [...messagesForQuery, ...assistantMessages, ...toolResults],
  querySource,
)) {
  yield attachment
  toolResults.push(attachment)
}
```

附件来源（`attachments.ts:743`）：

| 附件类型 | 说明 |
|---|---|
| `at_mentioned_files` | @提到的文件内容 |
| `mcp_resources` | MCP 资源附件 |
| `agent_mentions` | Agent 提及 |
| `skill_discovery` | 技能发现（实验性） |
| `edited_text_file` | 文件编辑差异 |
| `relevant_memory` | 通过 sideQuery 预取的相关记忆 |
| `nested_memory` | 嵌套记忆文件 |
| `queued_commands` | 队列中的命令/通知 |
| `mcp_instructions_delta` | MCP 指令变更增量 |

同时还会注入 **记忆预取结果**（`pendingMemoryPrefetch`，query.ts:1599-1614）和 **技能发现预取**（`pendingSkillPrefetch`，query.ts:1620-1628），这些都是在本轮 API streaming 期间并行执行的。

#### Step 13: 构建下一轮 messages

```typescript
// query.ts:1715-1727
const next: State = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  pendingToolUseSummary: nextPendingToolUseSummary,
  // ...
}
state = next
// → 回到 while(true) 循环顶部
```

**下一轮的 messages = 当前 messagesForQuery + 本轮 assistant 回复 + 工具结果 + 附件**。

---

## 四、最终发送给 API 的完整 Context 结构

### 4.1 内部表示 vs 实际 API Payload

Claude Code 内部维护一个多类型的 `Message[]` 数组（包含 `user`、`assistant`、`system`、`attachment`、`progress` 等类型），但 **Anthropic API 只接受严格的 user/assistant 交替**。在调用 API 之前，`normalizeMessagesForAPI()` (messages.ts:1989) 会执行一次关键转换：

1. **attachment → user 消息**：通过 `normalizeAttachmentForAPI()` 转化
2. **system → user 消息**：local_command 等转为 user 消息
3. **连续 user 合并**：通过 `mergeUserMessages()` 合并为一条
4. **attachment 冒泡**：`reorderAttachmentsForAPI()` 把 attachment 向上冒泡到最近的停止点（assistant 消息或 tool_result）之后
5. **过滤**：移除 progress 消息、虚拟消息、空白助手消息等

### 4.2 Attachment 的具体转化方式

`normalizeAttachmentForAPI()` (messages.ts:3453) 根据类型做不同转化：

| 附件类型 | API 转化方式 |
|---|---|
| `file`（@提到的文件） | **伪造** `assistant: tool_use(FileReadTool)` + `user: tool_result(文件内容)` |
| `directory`（@提到的目录） | **伪造** `assistant: tool_use(BashTool, "ls ...")` + `user: tool_result(目录列表)` |
| `edited_text_file`（文件差异） | `user` 消息: "Note: xxx was modified..."，`isMeta: true` |
| `skill_discovery`（技能发现） | `user` 消息，包裹在 `<system-reminder>` 中 |
| `relevant_memory`（相关记忆） | `user` 消息 |
| `selected_lines_in_ide` | `user` 消息: "The user selected lines..." |
| `opened_file_in_ide` | `user` 消息: "The user opened file..." |
| `compact_file_reference` | `user` 消息: "Note: xxx was read before summarization..." |
| `pdf_reference` | `user` 消息: PDF 引用说明 + 阅读指引 |
| `teammate_mailbox` | `user` 消息: 队友消息列表 |
| `team_context` | `user` 消息: 团队协调上下文 |

所有 attachment 转化后的 user 消息都标记 `isMeta: true`（不在 UI 中展示），且通常包裹在 `<system-reminder>` 标签中。

### 4.3 内部 Message 数组 vs 实际 API 消息对比

```
=== Claude Code 内部 Message 数组 ===          === 实际发给 API 的 messages ===

[0] <system-reminder> userContext               [0] user: {content: [
                                                     {type:"text", text:"<system-reminder>
[1] compact_summary                                   CLAUDE.md + 日期 + compact摘要
                                                     </system-reminder>"}
                                                   ]}  ← 合并为一条 user

[2] user: "请帮我重构这个函数"                   [1] user: {content: "请帮我重构这个函数"}

[3] assistant: text + tool_use(FileRead)        [2] assistant: {content: [
                                                     {type:"text", text:"好的，让我看看..."},
                                                     {type:"tool_use", name:"Read",
                                                      id:"toolu_xxx", input:{...}}
                                                   ]}

[4] user: tool_result(文件内容)                  [3] user: {content: [
[5] attachment: edited_text_file                      {type:"tool_result", tool_use_id:"toolu_xxx",
[6] attachment: relevant_memory                        content:"文件内容..."},
                                                     {type:"text", text:"Note: xxx.ts was
                                                      modified..."},
                                                     {type:"text", text:"(记忆内容)"}
                                                   ]}  ← 全部合并为一条 user

[7] assistant: text + tool_use(FileEdit)        [4] assistant: {content: [
                                                     {type:"text", text:"我来修改..."},
                                                     {type:"tool_use", name:"FileEdit",
                                                      id:"toolu_yyy", input:{...}}
                                                   ]}

[8] user: tool_result(编辑成功)                  [5] user: {content: [
[9] attachment: skill_discovery                       {type:"tool_result", tool_use_id:"toolu_yyy",
                                                      content:"编辑成功"},
                                                     {type:"text", text:"<system-reminder>
                                                      Skills relevant to your task...
                                                      </system-reminder>"}
                                                   ]}  ← 合并为一条 user
```

### 4.4 完整 API 请求结构

```
┌─ system (System Prompt) ────────────────────────────────────────────┐
│  [静态] 角色定义 / 系统规范 / 任务指南 / 行为规范 / 工具指南 / 风格 │
│  [动态] session_guidance / memory / env_info / language / MCP...     │
│  [追加] systemContext → "gitStatus: Current branch: main\n..."      │
└─────────────────────────────────────────────────────────────────────┘

┌─ messages (严格 user/assistant 交替) ───────────────────────────────┐
│                                                                      │
│  user:       <system-reminder>CLAUDE.md + 日期</system-reminder>     │
│              + compact 摘要（若有）                                   │
│                                                                      │
│  user:       "请帮我重构这个函数"                                    │
│                                                                      │
│  assistant:  text("好的...") + tool_use("Read", {...})               │
│                                                                      │
│  user:       tool_result(文件内容)                                   │
│              + "Note: xxx.ts was modified..."                        │
│              + (相关记忆内容)                                        │
│                                                                      │
│  assistant:  text("我来修改...") + tool_use("FileEdit", {...})       │
│                                                                      │
│  user:       tool_result(编辑成功)                                   │
│              + <system-reminder>Skills relevant...</system-reminder> │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─ tools (JSON Schema 定义) ──────────────────────────────────────────┐
│  所有可用工具的 JSON Schema 定义（40+ 个工具）                       │
│  BashTool / FileReadTool / FileEditTool / GrepTool / AgentTool ... │
└─────────────────────────────────────────────────────────────────────┘
```

**核心设计点**：
- API 看到的永远是严格的 user → assistant → user → assistant 交替
- attachment 不是独立的消息类型，而是被"粘"到相邻的 user 消息上
- 连续的 user 消息一定会被合并（Bedrock 不支持连续 user，1P API 虽然支持但也会合并）
- `isMeta: true` 的消息不展示给用户，但会进入 API 请求（仅 Claude 可见）
- 文件/目录型 attachment 通过伪造 tool_use + tool_result 对来注入，让模型认为"之前已经读过这个文件"
```

---

## 五、上下文窗口保护：多层压缩防线

Claude Code 设计了一套精巧的多层防线来防止 context 超出窗口限制，每层比上一层更"激进"：

```
Layer 1: Snip（轻量裁剪）
    │     移除老旧中间消息，无需额外 API 调用
    ↓ 还不够？
Layer 2: Microcompact（微压缩）
    │     对单条工具结果做缓存编辑式压缩
    ↓ 还不够？
Layer 3: Context Collapse（上下文折叠）
    │     将早期消息分组折叠为摘要（读时投影）
    ↓ 还不够？
Layer 4: Auto Compact（主动全量压缩）
    │     调用 API 生成整个对话的摘要
    │     有效窗口 = contextWindow - min(maxOutput, 20000)
    ↓ API 返回 413 Prompt Too Long？
Layer 5: Reactive Compact（被动紧急压缩）
    │     收到 prompt_too_long 错误后紧急触发全量压缩
    ↓ 还不行？
Layer 6: Blocking Limit（阻断）
    │     停止 API 调用，提示用户手动执行 /compact
    ↓
    终止当前轮次
```

关键设计点：
- **熔断器**：连续压缩失败计数器，防止无限重试
- **单次保护**：`hasAttemptedReactiveCompact` 标记防止 reactive compact 反复触发
- **max_output_tokens 恢复**：如果模型输出被截断，最多重试 3 次，注入恢复提示词让模型继续

---

## 六、特殊 Context 注入机制

### 6.1 记忆预取（Memory Prefetch）

```typescript
// query.ts:301-304
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
  state.messages, state.toolUseContext,
)
```

在循环开始前启动异步 sideQuery 查找相关记忆。在 Step 12 处消费结果——如果已就绪则注入，否则跳过等下一轮。通过 `filterDuplicateMemoryAttachments` 过滤模型已经 Read/Write/Edit 过的记忆文件。

### 6.2 技能发现预取（Skill Discovery Prefetch）

```typescript
// query.ts:331-335
const pendingSkillPrefetch = skillPrefetch?.startSkillDiscoveryPrefetch(
  null, messages, toolUseContext,
)
```

每轮迭代开始时触发（基于 write-pivot 检测），在 API streaming 期间并行运行。发现的相关技能作为附件注入。

### 6.3 队列命令（Queued Commands）

```typescript
// query.ts:1570-1578
const queuedCommandsSnapshot = getCommandsByMaxPriority(
  sleepRan ? 'later' : 'next',
).filter(cmd => { ... })
```

支持在工具执行期间接收外部命令（如 task 完成通知），作为附件注入到下一轮消息中。Agent 作用域隔离：主线程只消费 `agentId === undefined` 的命令。

### 6.4 工具刷新

```typescript
// query.ts:1659-1671
if (updatedToolUseContext.options.refreshTools) {
  const refreshedTools = updatedToolUseContext.options.refreshTools()
  // ...
}
```

每轮之间刷新工具列表，使新连接的 MCP 服务器的工具立即可用。

---

## 七、关键数据类型

### 7.1 Message 类型

```typescript
type Message =
  | UserMessage          // 用户消息 / tool_result
  | AssistantMessage     // 助手回复 / tool_use
  | SystemMessage        // 系统消息（compact_boundary, warning, local_command 等）
  | AttachmentMessage    // 附件（文件差异、记忆、通知等）
  | ProgressMessage      // 进度消息（仅 UI 展示，不发 API）
```

### 7.2 State（循环状态）— `query.ts:268-279`

State 是 `queryLoop` while 循环中**跨迭代的可变状态**。每轮结束时通过 `state = next` 整体替换。

```typescript
// query.ts:268-279
let state: State = {
  messages: params.messages,              // ① 当前消息列表（滚雪球累积）
  toolUseContext: params.toolUseContext,   // ② 工具执行上下文
  maxOutputTokensOverride: undefined,     // ③ 输出 token 限制覆盖
  autoCompactTracking: undefined,         // ④ 压缩追踪状态
  stopHookActive: undefined,              // ⑤ 停止钩子是否激活
  maxOutputTokensRecoveryCount: 0,        // ⑥ 输出截断恢复计数（最多 3 次）
  hasAttemptedReactiveCompact: false,     // ⑦ 是否已尝试被动压缩（防循环）
  turnCount: 1,                           // ⑧ 当前轮次计数
  pendingToolUseSummary: undefined,       // ⑨ 上一轮的工具摘要（异步 Promise）
  transition: undefined,                  // ⑩ 状态转移原因
}
```

各字段详解：

| 字段 | 类型 | 作用 |
|---|---|---|
| `messages` | `Message[]` | **核心**。每轮结束拼接为 `[...messagesForQuery, ...assistantMessages, ...toolResults]`，滚雪球增长直到被压缩 |
| `toolUseContext` | `ToolUseContext` | 工具调用所需的完整上下文，包含 options（tools/model/commands）、abortController、readFileState、queryTracking 等 |
| `autoCompactTracking` | `AutoCompactTrackingState?` | 追踪压缩状态：`compacted`（是否已压缩）、`turnCounter`（压缩后经过的轮数）、`turnId`、`consecutiveFailures`（连续失败熔断器） |
| `maxOutputTokensOverride` | `number?` | 升级输出限制。当模型输出被 8K 默认上限截断时，先升到 64K 重试一次（`ESCALATED_MAX_TOKENS`） |
| `maxOutputTokensRecoveryCount` | `number` | 输出截断恢复计数器，上限 3 次。超过后不再重试，表面化错误 |
| `hasAttemptedReactiveCompact` | `boolean` | 防止 reactive compact 反复触发的单次保护标记。reset 于每轮正常的 next_turn 转移 |
| `turnCount` | `number` | 轮次计数，用于 maxTurns 限制检查和 task_budget 剩余计算 |
| `pendingToolUseSummary` | `Promise?` | 上一轮用 Haiku 异步生成的工具使用摘要。在下一轮 API streaming 前 yield（1s Haiku 隐藏在 5-30s 主模型流式中） |
| `stopHookActive` | `boolean?` | 标记停止钩子是否已激活。钩子返回 blocking errors 时注入错误消息让模型重试 |
| `transition` | `{ reason: string }?` | 标记本次循环是通过何种原因 continue 的。可能的值见下表 |

**transition.reason 可能的值**：

| reason | 含义 |
|---|---|
| `'next_turn'` | 正常的工具调用后继续下一轮 |
| `'reactive_compact_retry'` | 收到 413 后紧急压缩，用压缩后的消息重试 |
| `'collapse_drain_retry'` | 先尝试排空折叠队列来降低 token 数 |
| `'max_output_tokens_escalate'` | 输出被 8K 限制截断，升级到 64K 重试 |
| `'max_output_tokens_recovery'` | 输出被截断，注入恢复提示词继续 |
| `'stop_hook_blocking'` | 停止钩子返回了阻塞错误，注入错误让模型修正 |
| `'token_budget_continuation'` | Token 预算未用完，注入继续提示词 |

**每轮结束时的 State 构建**（`query.ts:1715-1727`）：

```typescript
const next: State = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,        // 重置
  hasAttemptedReactiveCompact: false,      // 重置
  pendingToolUseSummary: nextPendingToolUseSummary,
  maxOutputTokensOverride: undefined,      // 重置
  stopHookActive,
  transition: { reason: 'next_turn' },
}
```

### 7.3 QueryParams（查询参数）

```typescript
// query.ts:181-198
type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt              // string[]
  userContext: { [k: string]: string }     // { claudeMd, currentDate }
  systemContext: { [k: string]: string }   // { gitStatus }
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number
  skipCacheWrite?: boolean
  taskBudget?: { total: number }
  deps?: QueryDeps
}
```

---

## 八、normalizeMessagesForAPI — 消息序列化为 API 格式

> 源码位置：`utils/messages.ts:1989-2369`
>
> 调用位置：`services/api/claude.ts:1266`
>
> 作用：将内部多类型的 `Message[]` 转化为 Anthropic API 要求的严格 user/assistant 交替格式

### 8.1 执行总览

```
内部 Message[]
    │
    │  ═══ 阶段一：重排序 + 过滤 ═══
    ├── ① reorderAttachmentsForAPI()       attachment 冒泡重排
    ├── ② filter(isVirtual)                移除虚拟消息
    ├── ③ 构建 stripTargets Map            标记需剥离 doc/img 的消息
    ├── ④ filter(类型)                     过滤 progress/system/合成错误
    │
    │  ═══ 阶段二：按类型转化 + 合并 ═══
    ├── ⑤ system(local_cmd) → user         转为 user 并合并
    ├── ⑥ user → 标准化                   剥离 + 注入 + 合并
    ├── ⑦ assistant → 标准化              规范化 tool_use + 合并同 ID
    ├── ⑧ attachment → user               转化 + 合并
    │
    │  ═══ 阶段三：多轮后处理清洗 ═══
    ├── ⑨  relocateToolReferenceSiblings   移动 tool_ref 文本兄弟
    ├── ⑩  filterOrphanedThinkingOnly      过滤孤立 thinking assistant
    ├── ⑪  filterTrailingThinking          剥离末尾 thinking block
    ├── ⑫  filterWhitespaceOnly            移除空白 assistant
    ├── ⑬  ensureNonEmptyAssistant         确保 assistant 非空
    ├── ⑭  smooshSystemReminderSiblings    折叠 <system-reminder> 到 tool_result
    ├── ⑮  sanitizeErrorToolResultContent  清理 error 中的图片
    ├── ⑯  appendMessageTag               追加 [id:xxx] snip 标签
    ├── ⑰  validateImagesForAPI            验证图片尺寸
    │
    ↓
(UserMessage | AssistantMessage)[]  ← 严格 user/assistant 交替
```

### 8.2 阶段一：重排序 + 过滤（1989-2075 行）

#### ① Attachment 冒泡重排 — `reorderAttachmentsForAPI()` (1481 行)

从消息列表**底部向上扫描**，将 attachment 消息向上冒泡，直到碰到**停止点**：

- `assistant` 消息
- `user` 消息且内容包含 `tool_result`

这保证附件被放在正确位置——紧跟在它所关联的 tool_result 之后。

```typescript
// messages.ts:1481-1527（简化逻辑）
for (let i = messages.length - 1; i >= 0; i--) {
  if (message.type === 'attachment') {
    pendingAttachments.push(message)  // 收集
  } else if (isStoppingPoint) {
    // 碰到停止点 → 附件落在此处（紧跟停止点之后）
    result.push(...pendingAttachments)
    result.push(message)
    pendingAttachments.length = 0
  }
}
```

#### ② 移除虚拟消息（2000 行）

```typescript
.filter(m => !((m.type === 'user' || m.type === 'assistant') && m.isVirtual))
```

`isVirtual=true` 的消息仅在 REPL UI 中展示（如内部工具调用的展示），不应发给 API。

#### ③ 构建 stripTargets Map（2003-2054 行）

扫描合成的 API 错误消息（PDF 太大、图片太大、请求太大等），向后找到关联的 `isMeta` 用户消息，标记其中需要剥离的 block 类型（`document` / `image`），防止下次 API 调用重复发送失败的大文件。

#### ④ 类型过滤（2057-2075 行）

移除以下类型，只保留 `user`、`assistant`、`attachment`、`system(local_command)`：

- `progress` 消息（进度条，仅 UI 用）
- `system` 消息（除 `local_command` 外）
- 合成的 API 错误消息（已被 stripTargets 处理）

### 8.3 阶段二：按类型转化 + 合并（2076-2293 行）

遍历过滤后的消息，通过 `switch(message.type)` 分别处理：

#### ⑤ system (local_command) → user（2078-2092 行）

本地斜杠命令的输出（如 `/commit` 的结果）转为 user 消息，与前一条 user 合并。

#### ⑥ user 标准化（2094-2199 行）

多个子步骤：

1. **剥离 tool_reference**：
   - tool_search 未启用 → 全部剥离
   - tool_search 启用 → 仅剥离不可用工具的引用（MCP 断连等）
2. **剥离 doc/img blocks**：根据 Step③ 的 stripTargets，从指定 isMeta 消息中移除过大的 document/image block
3. **注入 turn boundary**：为包含 tool_reference 的消息添加 `TOOL_REFERENCE_TURN_BOUNDARY` 文本（防止模型误判连续 human turn）
4. **合并连续 user**：如果前一条也是 user → `mergeUserMessages()`

#### ⑦ assistant 标准化（2201-2267 行）

1. **tool_use input 规范化**：通过 `normalizeToolInputForAPI()` 清理输入（如剥离 ExitPlanMode 的 plan 字段）
2. **工具名规范化**：转为 `tool.name`（canonical name），处理工具重命名/别名
3. **非 tool_search 模式**：显式构造标准 API 字段 `{type, id, name, input}`，剥离 `caller` 等扩展字段
4. **合并同 ID assistant**：流式并行工具调用时，同一 API 响应会拆分为多条 AssistantMessage（每个 content block 一条），通过 `message.id` 匹配后 `mergeAssistantMessages()`

#### ⑧ attachment → user（2269-2291 行）

调用 `normalizeAttachmentForAPI(attachment)` 转化，返回值是 `UserMessage[]`（可能是多条，如伪造的 tool_use + tool_result 对）。然后与前一条 user 消息合并。

转化方式对照表：

| 附件类型 | API 转化方式 |
|---|---|
| `file` | 伪造 `assistant: tool_use(FileReadTool)` + `user: tool_result(文件内容)` |
| `directory` | 伪造 `assistant: tool_use(BashTool, "ls ...")` + `user: tool_result(目录列表)` |
| `edited_text_file` | `user` 消息: `"Note: xxx was modified..."` |
| `skill_discovery` | `user` 消息: `<system-reminder>Skills relevant to your task...` |
| `relevant_memory` | `user` 消息 |
| `compact_file_reference` | `user` 消息: `"Note: xxx was read before summarization..."` |
| `pdf_reference` | `user` 消息: PDF 引用说明 + 分页阅读指引 |
| `selected_lines_in_ide` | `user` 消息: `"The user selected lines X to Y..."` |
| `opened_file_in_ide` | `user` 消息: `"The user opened file xxx..."` |
| `teammate_mailbox` | `user` 消息: 队友消息列表 |
| `team_context` | `user` 消息: `<system-reminder>` 团队协调上下文 |

### 8.4 阶段三：多轮后处理清洗（2295-2369 行）

经过阶段二，result 数组已经是 `(UserMessage | AssistantMessage)[]` 格式，但可能存在各种边缘问题。以下后处理**按顺序执行，顺序不可变**：

#### ⑨ relocateToolReferenceSiblings（2301-2305 行）

Feature gate: `tengu_toolref_defer_j8m`

将 tool_reference 消息旁边的文本兄弟节点移到后面的非 ref 消息中，防止出现异常的"连续两个 human turn"模式——该模式会教导模型在 tool_result 后输出 stop sequence。

#### ⑩ filterOrphanedThinkingOnlyMessages（2311 行）

过滤**孤立的 thinking-only assistant 消息**。这类消息通常是压缩裁切时残留的——流式失败重试之间的消息被裁掉，留下签名不匹配的 thinking block，会导致 API 400 错误。

#### ⑪ filterTrailingThinkingFromLastAssistant（2322 行）

剥离**最后一条 assistant 消息末尾的 thinking block**。API 规则要求 thinking block 不能是消息的最后一个 block。

> **顺序依赖**：必须在 ⑫ 之前执行。如果先执行 ⑫，消息 `[text("\n\n"), thinking("...")]` 会通过空白过滤（有非 text block），然后 ⑪ 移除 thinking 后剩下 `[text("\n\n")]`，API 会拒绝。

#### ⑫ filterWhitespaceOnlyAssistantMessages（2324 行）

移除内容全是空白字符的 assistant 消息。

#### ⑬ ensureNonEmptyAssistantContent（2325 行）

确保每条 assistant 消息至少有一个非空 content block。

#### ⑭ smooshSystemReminderSiblings（2334-2338 行）

Feature gate: `tengu_chair_sermon`

先 `mergeAdjacentUserMessages()` 合并相邻 user，然后将 `<system-reminder>` 前缀的文本 block 折叠进相邻的 tool_result block。减少 user turn 中零散的文本 block 数量。

#### ⑮ sanitizeErrorToolResultContent（2343 行）

清理 `is_error=true` 的 tool_result 中的非法内容（如图片）。无条件执行，修复恢复旧 session 时图片在 error tool_result 中导致的永久 400 错误。

#### ⑯ appendMessageTag（2351-2363 行）

Feature flag: `HISTORY_SNIP`

给每条 user 消息追加 `[id:xxx]` 标签，供 Snip 工具在后续裁剪时引用特定消息。仅在 snip 功能运行时启用，测试环境跳过（避免改变消息 hash）。

#### ⑰ validateImagesForAPI（2367 行）

最后一步校验：确保所有图片在 API 尺寸限制内。超限则抛出 `ImageSizeError`。

### 8.5 设计评价

代码注释中坦言这种多 pass 设计 **"inherently fragile"（本质上脆弱）**：

> *"These multi-pass normalizations are inherently fragile — each pass can create conditions a prior pass was meant to handle. Consider unifying into a single pass that cleans content, then validates in one shot."*
>
> — `messages.ts:2318-2320`

每个 pass 可能创造出前一个 pass 本应处理的条件（如 ⑪⑫ 的顺序依赖 bug），但目前仍以多 pass 方式运作，通过严格的执行顺序和注释警告来维护正确性。
