# Claude Code Subagent 机制深度分析

> 源码版本：2026-03-31 泄露快照
>
> 核心文件：`src/tools/AgentTool/AgentTool.tsx`、`src/tools/AgentTool/runAgent.ts`、`src/tools/AgentTool/prompt.ts`、`src/tools/AgentTool/loadAgentsDir.ts`、`src/tools/AgentTool/builtInAgents.ts`

---

## 一、核心结论

**Claude Code 的 subagent 不是框架层面的特殊机制，而是一个普通的 tool（工具）。**

模型通过标准的 `tool_use`（function calling）协议来启动子代理，和调用 `Bash`、`FileRead`、`FileEdit` 等工具完全相同。框架不主动决定何时派生子代理——这个判断 100% 由模型自己完成。

---

## 二、AgentTool 的身份

### 2.1 工具注册

`AgentTool` 在工具注册表中的身份：

```typescript
// src/tools/AgentTool/constants.ts
export const AGENT_TOOL_NAME = 'Agent'
export const LEGACY_AGENT_TOOL_NAME = 'Task'   // 历史别名，兼容旧权限规则
```

它通过 `buildTool()` 构建，和其他工具完全一样：

```typescript
// src/tools/AgentTool/AgentTool.tsx:196
export const AgentTool = buildTool({
  name: AGENT_TOOL_NAME,         // "Agent"
  aliases: [LEGACY_AGENT_TOOL_NAME],  // ["Task"]
  searchHint: 'delegate work to a subagent',
  async prompt({ agents, tools }) { ... },
  async call({ prompt, subagent_type, ... }, toolUseContext, ...) { ... },
  inputSchema() { ... },
  outputSchema() { ... },
})
```

### 2.2 模型看到的 Input Schema

模型在工具定义中看到的参数：

```typescript
// src/tools/AgentTool/AgentTool.tsx:82-88
z.object({
  description: z.string()
    .describe('A short (3-5 word) description of the task'),
  prompt: z.string()
    .describe('The task for the agent to perform'),
  subagent_type: z.string().optional()
    .describe('The type of specialized agent to use for this task'),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional()
    .describe("Optional model override for this agent"),
  run_in_background: z.boolean().optional()
    .describe('Set to true to run this agent in the background'),
})
```

额外可选参数（特性开关控制）：
- `name`: 给 agent 命名，可通过 SendMessage 寻址
- `team_name`: 团队名
- `mode`: 权限模式
- `isolation`: `"worktree"` 或 `"remote"`，隔离模式
- `cwd`: 工作目录覆盖

---

## 三、模型的决策依据

### 3.1 动态生成的工具描述

模型是通过阅读 `Agent` 工具的 prompt 来"学会"何时该派生子代理的。这个 prompt 由 `prompt.ts` 的 `getPrompt()` 函数动态生成，包含：

**核心开头：**
```
Launch a new agent to handle complex, multi-step tasks autonomously.

The Agent tool launches specialized agents (subprocesses) that
autonomously handle complex tasks. Each agent type has specific
capabilities and tools available to it.
```

**可用 agent 类型列表：**
```
Available agent types and the tools they have access to:
- Explore: Fast agent specialized for exploring codebases... (Tools: All tools except Agent, ...)
- general-purpose: General-purpose agent for researching... (Tools: All tools)
- Plan: Read-only collaborative mode... (Tools: ...)
```

**使用指南（非 coordinator 模式时完整展示）：**
- 何时不该用 Agent（简单文件读取、已知路径搜索）
- 如何并行启动多个 agent
- 如何写好 prompt（"Brief the agent like a smart colleague who just walked into the room"）
- 前台 vs 后台运行的选择

### 3.2 Agent 列表注入方式

有两种模式（由 `shouldInjectAgentListInMessages()` 控制）：

| 模式 | 位置 | 优点 | 缺点 |
|---|---|---|---|
| **内联模式（默认）** | 写在工具描述中 | 简单直接 | MCP/插件变化 → 工具描述变化 → prompt cache 失效 |
| **附件模式（实验中）** | 通过 `agent_listing_delta` attachment 注入到消息中 | 工具描述静态，prompt cache 稳定 | 需要额外的附件消息 |

内联模式下，agent 列表变化（MCP 连接、插件加载、权限变更）会导致整个工具 schema 的 prompt cache 失效。源码注释：

> The dynamic agent list was ~10.2% of fleet cache_creation tokens

---

## 四、完整调用链

### 4.1 触发时机

```
模型返回:
{
  "type": "tool_use",
  "name": "Agent",
  "input": {
    "description": "搜索认证相关代码",
    "prompt": "在代码库中找到所有与用户认证相关的实现...",
    "subagent_type": "Explore"
  }
}
```

### 4.2 执行流程

```
queryLoop 检测到 tool_use block
  │
  ├─ needsFollowUp = true
  │
  ├─ 工具执行器 dispatch → AgentTool.call()
  │   │
  │   ├─ ① 解析 subagent_type
  │   │   ├─ 有值 → 从 agentDefinitions 中查找
  │   │   ├─ 省略 + fork 开启 → 使用 FORK_AGENT（继承上下文）
  │   │   └─ 省略 + fork 关闭 → 默认 GENERAL_PURPOSE_AGENT
  │   │
  │   ├─ ② 检查前置条件
  │   │   ├─ MCP 服务器要求
  │   │   ├─ 权限规则（是否被 deny）
  │   │   ├─ 递归 fork 防护（fork 子代理不能再 fork）
  │   │   └─ in-process teammate 限制
  │   │
  │   ├─ ③ 决定运行模式
  │   │   ├─ 同步前台（默认）
  │   │   ├─ 异步后台（run_in_background = true）
  │   │   ├─ 隔离 worktree（isolation = "worktree"）
  │   │   └─ 远程（isolation = "remote"，ant-only）
  │   │
  │   ├─ ④ 调用 runAgent()
  │   │   └─ 启动全新的 query() 循环（独立的 while(true) agent loop）
  │   │
  │   └─ ⑤ 收集结果，作为 tool_result 返回
  │
  └─ queryLoop 收到 tool_result → 发给模型 → 下一轮循环
```

### 4.3 runAgent() 内部初始化

`runAgent()`（`runAgent.ts:248`）负责为子代理搭建完整的运行环境：

```
runAgent()
  │
  ├─ 解析模型: getAgentModel(agentDef.model, parentModel, overrideModel)
  │
  ├─ 创建 agentId: createAgentId()
  │
  ├─ 构建初始消息:
  │   ├─ forkContextMessages（fork 模式才有，继承父对话）
  │   ├─ promptMessages（用户的 prompt 包装为 user message）
  │   └─ 过滤不完整的 tool calls（filterIncompleteToolCalls）
  │
  ├─ 获取上下文:
  │   ├─ userContext（CLAUDE.md 等，Explore/Plan 可省略）
  │   └─ systemContext（git status 等，Explore/Plan 可省略）
  │
  ├─ 构建 system prompt:
  │   └─ agentDefinition.getSystemPrompt() + enhanceSystemPromptWithEnvDetails()
  │
  ├─ 解析工具集: resolveAgentTools(agentDef, availableTools)
  │
  ├─ 初始化 agent 专属 MCP 服务器
  │
  ├─ 执行 SubagentStart hooks
  │
  ├─ 预加载 skills
  │
  ├─ 创建子代理上下文: createSubagentContext()
  │   ├─ 独立的 messages 数组
  │   ├─ 独立的 readFileState 缓存
  │   ├─ 独立的 abortController（异步时）
  │   └─ 独立的权限配置
  │
  └─ 调用 query() → 子代理的 agent loop 开始运行
```

---

## 五、内置 Agent 类型

### 5.1 注册机制

```typescript
// src/tools/AgentTool/builtInAgents.ts:22
export function getBuiltInAgents(): AgentDefinition[] {
  const agents = [GENERAL_PURPOSE_AGENT, STATUSLINE_SETUP_AGENT]

  if (areExplorePlanAgentsEnabled()) {
    agents.push(EXPLORE_AGENT, PLAN_AGENT)
  }
  if (isNonSdkEntrypoint) {
    agents.push(CLAUDE_CODE_GUIDE_AGENT)
  }
  if (feature('VERIFICATION_AGENT') && ...) {
    agents.push(VERIFICATION_AGENT)
  }
  return agents
}
```

### 5.2 各 Agent 详解

| Agent | agentType | 模型 | 工具集 | 特点 |
|---|---|---|---|---|
| **General Purpose** | `general-purpose` | 继承父代理 | 所有工具 (`['*']`) | 通用型，可搜索/编码/执行 |
| **Explore** | `Explore` | haiku（外部）/ inherit（ant） | 除 Agent, FileEdit, FileWrite, NotebookEdit, ExitPlanMode 外的所有工具 | 只读搜索，省略 CLAUDE.md 和 gitStatus，追求速度 |
| **Plan** | `Plan` | — | 只读工具 | 协作规划模式 |
| **Claude Code Guide** | — | — | — | 非 SDK 模式下可用，帮助引导 |
| **Verification** | `verification` | — | — | 实验性，需要特性开关 |
| **Statusline Setup** | — | — | — | 状态栏初始化 |

### 5.3 Explore Agent 的 System Prompt

```
You are a file search specialist for Claude Code...

=== CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
This is a READ-ONLY exploration task. You are STRICTLY PROHIBITED from:
- Creating new files
- Modifying existing files
- Deleting files
...

Your strengths:
- Rapidly finding files using glob patterns
- Searching code and text with powerful regex patterns
- Reading and analyzing file contents

NOTE: You are meant to be a fast agent that returns output as quickly
as possible. In order to achieve this you must:
- Make efficient use of the tools
- Wherever possible you should try to spawn multiple parallel tool calls
```

### 5.4 General Purpose Agent 的 System Prompt

```
You are an agent for Claude Code, Anthropic's official CLI for Claude.
Given the user's message, you should use the tools available to complete
the task. Complete the task fully — don't gold-plate, but don't leave
it half-done.

When you complete the task, respond with a concise report covering what
was done and any key findings — the caller will relay this to the user,
so it only needs the essentials.
```

---

## 六、自定义 Agent

### 6.1 定义方式

用户可以在 `.claude/agents/` 目录下放 markdown 文件来定义自定义 agent：

```markdown
---
name: my-reviewer
description: 代码审查专家，用于审查 PR 变更
tools:
  - FileRead
  - Grep
  - Glob
  - Bash
model: sonnet
permissionMode: plan
maxTurns: 20
skills:
  - code-review
mcpServers:
  - github
hooks:
  SubagentStop:
    - command: "echo 'Review completed'"
---

你是一个专业的代码审查员。审查代码时关注：
1. 逻辑正确性
2. 性能问题
3. 安全漏洞
4. 代码风格

审查完成后，给出结构化的审查报告。
```

### 6.2 Frontmatter 字段

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `name` | string | 是 | agent 类型标识，模型通过 subagent_type 引用 |
| `description` | string | 是 | whenToUse，告诉模型何时使用此 agent |
| `tools` | string[] | 否 | 允许使用的工具白名单，默认全部 |
| `disallowedTools` | string[] | 否 | 工具黑名单 |
| `model` | string | 否 | 模型覆盖（sonnet/opus/haiku/inherit） |
| `effort` | string/int | 否 | 思考努力程度 |
| `permissionMode` | string | 否 | 权限模式 |
| `maxTurns` | int | 否 | 最大循环轮次 |
| `skills` | string[] | 否 | 预加载的 skill |
| `mcpServers` | array | 否 | agent 专属 MCP 服务器 |
| `hooks` | object | 否 | 生命周期钩子 |
| `background` | boolean | 否 | 是否始终后台运行 |
| `memory` | string | 否 | 持久记忆范围（user/project/local） |
| `isolation` | string | 否 | 隔离模式（worktree/remote） |
| `initialPrompt` | string | 否 | 首轮追加的 prompt |

### 6.3 Agent 来源优先级

```
built-in  →  plugin  →  userSettings  →  projectSettings  →  flagSettings  →  policySettings
  ↓            ↓            ↓               ↓                  ↓               ↓
 最低         ...          ...             ...                ...             最高
```

相同 `agentType` 的 agent，后面的来源覆盖前面的。

---

## 七、子代理 vs 父代理的隔离

### 7.1 独立的部分

| 维度 | 隔离方式 |
|---|---|
| **消息历史** | 子代理有独立的 messages[]，不共享父对话 |
| **System Prompt** | 子代理有自己的 system prompt |
| **工具集** | 根据 AgentDefinition 过滤，可以比父代理少 |
| **模型** | 可以不同（如 Explore 用 haiku） |
| **文件状态缓存** | 独立的 readFileState（fork 模式克隆父缓存） |
| **AbortController** | 异步子代理有独立的 controller |
| **权限上下文** | 可覆盖 permissionMode，可限制 allowedTools |
| **TodoWrite 状态** | 独立的 agentId 作为 key，agent 结束时清理 |

### 7.2 共享的部分

| 维度 | 共享方式 |
|---|---|
| **文件系统** | 默认共享（除非 isolation: "worktree"） |
| **AppState store** | 同步子代理共享 setAppState；异步子代理通过 rootSetAppState 写入 |
| **API metrics** | 子代理的 TTFT/OTPS 转发给父代理显示 |
| **Bash 进程** | 子代理 spawn 的后台 bash 在 agent 结束时被 kill |

### 7.3 Fork 模式的特殊性

Fork 模式（省略 `subagent_type` 且特性开启时）与普通子代理的关键区别：

```
普通子代理                          Fork 子代理
├─ 零上下文启动                      ├─ 继承父对话的完整 messages
├─ 独立的 system prompt              ├─ 继承父的 system prompt
├─ 不同的工具集                      ├─ 继承父的完整工具集
├─ 可以用不同模型                    ├─ 不应换模型（共享 prompt cache）
└─ prompt 需要写完整背景             └─ prompt 只需写"指令"（有上下文）
```

Fork 的核心优势是**共享 prompt cache**，源码注释：

> Forks are cheap because they share your prompt cache. Don't set `model`
> on a fork — a different model can't reuse the parent's cache.

---

## 八、上下文优化

### 8.1 Explore/Plan 的轻量化

Explore 和 Plan agent 作为只读搜索代理，做了多项上下文裁剪：

| 裁剪项 | 节省量 | 机制 |
|---|---|---|
| **省略 CLAUDE.md** | ~5-15 Gtok/week | `omitClaudeMd: true` + `tengu_slim_subagent_claudemd` 开关 |
| **省略 gitStatus** | ~1-3 Gtok/week | 检测 agentType === 'Explore' \|\| 'Plan' 时移除 |
| **跳过 agentId/SendMessage 尾注** | ~135 chars × 34M runs/week | `ONE_SHOT_BUILTIN_AGENT_TYPES` 集合 |

### 8.2 工具 Schema Cache 优化

Agent 列表从工具描述中移到 attachment 消息的实验：

```typescript
// src/tools/AgentTool/prompt.ts:59
export function shouldInjectAgentListInMessages(): boolean {
  // 通过 GrowthBook 控制：tengu_agent_list_attach
}
```

目的：工具描述变为静态 → prompt cache 不因 agent 列表变化而失效。

---

## 九、生命周期管理

### 9.1 启动阶段

```
AgentTool.call()
  → runAgent()
    → 注册 Perfetto trace
    → 初始化 MCP 服务器
    → 执行 SubagentStart hooks
    → 预加载 skills
    → 注册 frontmatter hooks
    → 记录初始 transcript
    → 启动 query() 循环
```

### 9.2 运行阶段

子代理运行自己独立的 `while(true)` agent loop，和父代理的循环完全解耦：
- 同步子代理：父代理阻塞等待
- 异步子代理：父代理继续运行，子代理完成后通过 notification 通知

### 9.3 清理阶段（finally block）

```typescript
// runAgent.ts:816-858  finally block
finally {
  await mcpCleanup()                    // 清理 agent 专属 MCP 服务器
  clearSessionHooks(agentId)            // 清理 session hooks
  cleanupAgentTracking(agentId)         // 清理 prompt cache 追踪
  readFileState.clear()                 // 释放文件状态缓存
  initialMessages.length = 0            // 释放消息数组内存
  unregisterPerfettoAgent(agentId)      // 释放 Perfetto 条目
  clearAgentTranscriptSubdir(agentId)   // 清理 transcript 子目录
  // 清理 todos 条目（防止内存泄漏）
  rootSetAppState(prev => {
    const { [agentId]: _removed, ...todos } = prev.todos
    return { ...prev, todos }
  })
  killShellTasksForAgent(agentId)       // kill 后台 bash 任务
  killMonitorMcpTasksForAgent(agentId)  // kill Monitor MCP 任务
}
```

源码注释解释了 todos 清理的必要性：

> Whale sessions spawn hundreds of agents; each orphaned key is a small
> leak that adds up.

---

## 十、设计哲学总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Claude Code 的 Subagent 设计哲学：                                  │
│                                                                     │
│  1. Agent 是一个普通工具                                             │
│     没有特殊的"子代理协议"，模型通过标准 function calling 调用。      │
│     框架对 Agent 工具和 Bash 工具一视同仁。                          │
│                                                                     │
│  2. 何时派生由模型决定                                               │
│     框架通过工具描述（prompt）告诉模型有哪些 agent 可用，             │
│     以及什么时候该用 / 不该用。判断权在模型手中。                     │
│                                                                     │
│  3. 子代理是独立的 agent loop                                        │
│     不是"在父循环中执行一步"，而是启动一个完整的                     │
│     while(true) { API 调用 → 工具执行 } 循环。                      │
│                                                                     │
│  4. 工具集是可裁剪的                                                 │
│     不同 agent 可以有不同的工具集（Explore 没有写入工具），           │
│     实现了"最小权限"原则。                                           │
│                                                                     │
│  5. 上下文是可优化的                                                 │
│     只读 agent 省略 CLAUDE.md 和 gitStatus，                        │
│     fork agent 共享 prompt cache，                                   │
│     追求 token 效率的最大化。                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 十一、源码位置索引

| 内容 | 文件 | 行号 |
|---|---|---|
| AgentTool 定义入口 | AgentTool.tsx | 196 |
| Input Schema | AgentTool.tsx | 82-88 |
| call() 主逻辑 | AgentTool.tsx | 239-400 |
| subagent_type 路由 | AgentTool.tsx | 322-356 |
| Fork 递归防护 | AgentTool.tsx | 332 |
| 工具名常量 | constants.ts | 1-12 |
| getPrompt() 动态描述 | prompt.ts | 66-287 |
| agent 列表附件开关 | prompt.ts | 59-63 |
| formatAgentLine() | prompt.ts | 43-46 |
| runAgent() 完整实现 | runAgent.ts | 248-860 |
| 初始化 MCP 服务器 | runAgent.ts | 95-218 |
| finally 清理 | runAgent.ts | 816-858 |
| filterIncompleteToolCalls | runAgent.ts | 866-904 |
| getBuiltInAgents() | builtInAgents.ts | 22-72 |
| GENERAL_PURPOSE_AGENT | built-in/generalPurposeAgent.ts | 25-34 |
| EXPLORE_AGENT | built-in/exploreAgent.ts | 64-83 |
| AgentDefinition 类型 | loadAgentsDir.ts | 106-165 |
| parseAgentFromMarkdown | loadAgentsDir.ts | 541-755 |
| parseAgentFromJson | loadAgentsDir.ts | 445-516 |
| getAgentDefinitionsWithOverrides | loadAgentsDir.ts | 296-393 |
