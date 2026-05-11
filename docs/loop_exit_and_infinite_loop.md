# Claude Code Agent Loop 退出机制与无限循环分析

> 源码版本：2026-03-31 泄露快照
>
> 核心文件：`src/query.ts`（queryLoop）、`src/services/compact/autoCompact.ts`、`src/query/stopHooks.ts`

---

## 一、循环退出的核心逻辑

### 1.1 唯一的循环继续信号

整个 `while (true)` 循环（`query.ts:307`）只有一个正向继续信号：

```typescript
// query.ts:558
let needsFollowUp = false

// query.ts:829-834 — 流式接收 assistant 消息时
if (message.type === 'assistant') {
  const msgToolUseBlocks = message.message.content.filter(
    content => content.type === 'tool_use',
  )
  if (msgToolUseBlocks.length > 0) {
    needsFollowUp = true   // 有结构化 tool_use block → 继续循环
  }
}
```

**只有 `content` 中存在 `type: 'tool_use'` 的 block 才会触发循环继续。** 源码注释明确说明：

```typescript
// query.ts:554-556
// Note: stop_reason === 'tool_use' is unreliable -- it's not always set correctly.
// Set during streaming whenever a tool_use block arrives — the sole
// loop-exit signal. If false after streaming, we're done (modulo stop-hook retry).
```

> tool_use block 的出现是循环继续的**唯一信号**。stop_reason 不可靠，文本中的 XML 模拟更不会被识别。

### 1.2 循环退出的判断流程

```
模型返回 assistant 消息
    │
    ├─ content 中有 tool_use block
    │   → needsFollowUp = true
    │   → 执行工具
    │   → 收集 toolResults + attachments
    │   → 构建 next state
    │   → continue（下一轮循环）
    │
    └─ content 中没有 tool_use block
        → needsFollowUp = false
        → 进入退出分支（query.ts:1062-1357）
        → 经过一系列检查后 return
```

### 1.3 退出分支的完整检查链

进入 `if (!needsFollowUp)` 后（`query.ts:1062`），按顺序执行以下检查：

```
① prompt_too_long 恢复？
   ├─ 尝试 collapse drain → continue（collapse_drain_retry）
   └─ 尝试 reactive compact → continue（reactive_compact_retry）
   └─ 都失败 → return { reason: 'prompt_too_long' }

② media_size_error 恢复？
   └─ 尝试 reactive compact → continue
   └─ 失败 → return { reason: 'image_error' }

③ max_output_tokens 恢复？
   ├─ 首次：升级到 64K → continue（max_output_tokens_escalate）
   └─ 后续：注入恢复消息 → continue（max_output_tokens_recovery）
   └─ 超过 3 次 → yield 错误消息，继续往下

④ API error？（rate_limit / auth / server_error）
   → return { reason: 'completed' }（跳过 stop hook，防 death spiral）

⑤ Stop Hook 检查
   ├─ preventContinuation = true → return { reason: 'stop_hook_prevented' }
   ├─ blockingErrors 非空 → 注入错误消息 → continue（stop_hook_blocking）
   └─ 通过 → 继续往下

⑥ Token Budget 检查
   ├─ 预算未用完 → 注入继续消息 → continue（token_budget_continuation）
   └─ 预算已满或无预算 → 继续往下

⑦ 所有检查通过
   → return { reason: 'completed' }   ← 正常退出
```

### 1.4 所有 return 退出点汇总

| 退出原因 | return 值 | 源码位置 | 触发条件 |
|---|---|---|---|
| 正常完成 | `{ reason: 'completed' }` | query.ts:1357 | 模型无 tool_use 且所有检查通过 |
| API error 退出 | `{ reason: 'completed' }` | query.ts:1264 | 最后消息是 API 错误 |
| Prompt 太长 | `{ reason: 'prompt_too_long' }` | query.ts:1175 | 压缩恢复失败 |
| 图片错误 | `{ reason: 'image_error' }` | query.ts:1175 | 媒体大小恢复失败 |
| Stop hook 阻止 | `{ reason: 'stop_hook_prevented' }` | query.ts:1279 | hook 返回 preventContinuation |
| Hook 停止 | `{ reason: 'hook_stopped' }` | query.ts:1520 | 工具执行中 hook 阻止 |
| 流式中断 | `{ reason: 'aborted_streaming' }` | query.ts:1051 | 用户 Ctrl+C（流式阶段） |
| 工具中断 | `{ reason: 'aborted_tools' }` | query.ts:1515 | 用户 Ctrl+C（工具执行阶段） |
| 达到轮次上限 | `{ reason: 'max_turns' }` | query.ts:1711 | turnCount > maxTurns |
| 阻断限制 | `{ reason: 'blocking_limit' }` | query.ts:646 | token 数超过硬限制且无压缩可用 |

---

## 二、可能导致无限循环的场景

### 2.1 第一类：模型不断返回 tool_use（模型行为导致）

这是最常见的"无限循环"场景。模型一直认为自己没做完，不断发起工具调用。

#### 场景 A：工具不断返回错误，模型不断重试

```
轮 1: 模型 → tool_use(FileEdit, ...)  → "Permission denied"
轮 2: 模型 → tool_use(FileEdit, ...)  → "Permission denied"
轮 3: 模型 → tool_use(FileEdit, ...)  → "Permission denied"
... 模型不知道放弃，一直重试同一个操作 ...
```

**防御**：仅 maxTurns（如果设置了）。模型层面没有"重试次数限制"。

#### 场景 B：探索循环（不断发现新的需要查看的内容）

```
轮 1: 模型 → Grep("bug")        → 找到 10 个文件
轮 2: 模型 → FileRead(file1)    → 发现 import 引用
轮 3: 模型 → Grep("import")     → 又找到 20 个文件
轮 4: 模型 → FileRead(file2)    → 又发现引用
... 模型陷入不断探索的循环 ...
```

**防御**：仅 maxTurns。没有"探索深度限制"机制。

#### 场景 C：Lint 修复循环

```
轮 1: 模型 → FileEdit(修改代码)  → 成功
      Stop Hook: lint 检查 → 发现 3 个错误 → 注入 blocking error
轮 2: 模型 → FileEdit(修复错误)  → 成功
      Stop Hook: lint 检查 → 修复引入了 2 个新错误 → 注入 blocking error
轮 3: 模型 → FileEdit(修复新错误) → 成功
      Stop Hook: lint 检查 → 又引入了新错误 → ...
```

**防御**：stopHookActive 标记 + maxTurns。但 stop hook 本身没有重试次数限制。

#### 场景 D：模型幻觉导致的循环

```
轮 1: 模型 → Bash("npm test")   → 测试失败
轮 2: 模型 → FileEdit(修改)     → 成功
轮 3: 模型 → Bash("npm test")   → 同样的测试失败（修改无效）
轮 4: 模型 → FileEdit(再修改)   → 成功
轮 5: 模型 → Bash("npm test")   → 还是失败
... 模型不断尝试无效的修复 ...
```

**防御**：仅 maxTurns 和用户 Ctrl+C。

### 2.2 第二类：框架内部 continue 分支反复触发

这些不是模型主动调用工具，而是 `queryLoop` 内部的 continue 分支不断重新发起 API 调用。

#### 场景 E：reactive compact 循环（已修复的真实 Bug）

**源码注释**（`query.ts:1292-1296`）：

```typescript
// Preserve the reactive compact guard — if compact already ran and
// couldn't recover from prompt-too-long, retrying after a stop-hook
// blocking error will produce the same result. Resetting to false
// here caused an infinite loop: compact → still too long → error →
// stop hook blocking → compact → … burning thousands of API calls.
```

**触发链**：

```
prompt_too_long
  → reactive compact（hasAttemptedReactiveCompact = true）
  → 压缩后仍然太长
  → API 返回 error
  → stop hook 返回 blocking error
  → continue（stop_hook_blocking）
  → 新一轮中 hasAttemptedReactiveCompact 被重置为 false  ← BUG
  → 又触发 reactive compact
  → 仍然太长
  → ... 无限循环，烧掉数千次 API 调用
```

**修复方式**：在 `stop_hook_blocking` 的 continue 中保留 `hasAttemptedReactiveCompact` 的值，不再重置。

#### 场景 F：autocompact 反复失败

**源码注释**（`autoCompact.ts:67-68`）：

```typescript
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
```

**触发条件**：上下文不可恢复地超过限制（如单条消息就超过了窗口大小），每轮都尝试压缩但每次都失败。

**修复方式**：`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`，连续失败 3 次后熔断器跳闸，不再尝试。

#### 场景 G：Stop Hook + API Error 的 Death Spiral

**源码注释**（`query.ts:1258-1261`）：

```typescript
// Skip stop hooks when the last message is an API error (rate limit,
// prompt-too-long, auth failure, etc.). The model never produced a
// real response — hooks evaluating it create a death spiral:
// error → hook blocking → retry → error → …
```

**触发链**：

```
API 返回 rate_limit error
  → stop hook 评估这个 error 消息
  → hook 认为"模型没完成任务" → 返回 blocking error
  → 注入错误消息 → continue
  → 再次调 API → 还是 rate_limit
  → stop hook 又评估 → 又 blocking
  → ... 无限循环
```

**修复方式**：当最后一条消息是 API error 时，直接跳过 stop hook，立即 return。

#### 场景 H：max_output_tokens 反复截断

```
轮 1: 模型输出被 8K 限制截断
  → 升级到 64K → continue（max_output_tokens_escalate）
轮 2: 模型输出又被 64K 截断
  → 注入恢复消息 → continue（max_output_tokens_recovery, attempt 1）
轮 3: 模型继续但又被截断
  → 注入恢复消息 → continue（attempt 2）
轮 4: 又截断
  → 注入恢复消息 → continue（attempt 3）
轮 5: 又截断
  → 超过 MAX_OUTPUT_TOKENS_RECOVERY_LIMIT(3) → 表面化错误，退出
```

**防御**：`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`，最多恢复 3 次。

---

## 三、防御机制汇总

### 3.1 硬限制（绝对防线）

| 机制 | 限制值 | 保护对象 | 源码位置 |
|---|---|---|---|
| **maxTurns** | 调用者设置 | 总循环轮次 | query.ts:1705-1711 |
| **abortController** | 用户 Ctrl+C | 随时中断 | query.ts:1015-1052, 1484-1515 |
| **blocking_limit** | 模型上下文窗口 | token 数硬上限 | query.ts:628-648 |

### 3.2 熔断器（Circuit Breaker）

| 机制 | 限制值 | 保护对象 | 源码位置 |
|---|---|---|---|
| **MAX_OUTPUT_TOKENS_RECOVERY_LIMIT** | 3 次 | 输出截断恢复 | query.ts:164 |
| **hasAttemptedReactiveCompact** | 1 次 | reactive compact | query.ts:1119-1157 |
| **MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES** | 3 次 | autocompact 连续失败 | autoCompact.ts:70 |

### 3.3 Death Spiral 防护

| 机制 | 防护对象 | 策略 | 源码位置 |
|---|---|---|---|
| **API error 跳过 stop hook** | stop hook + API error 循环 | 检测到 API error 直接 return | query.ts:1258-1264 |
| **保留 hasAttemptedReactiveCompact** | compact + stop hook 循环 | stop_hook_blocking 时不重置 | query.ts:1292-1297 |
| **stopHookActive 标记** | stop hook 重复触发 | 标记已激活状态 | query.ts:1300 |

### 3.4 间接保护

| 机制 | 作用 | 源码位置 |
|---|---|---|
| **autocompact** | 防止 token 数无限增长 | autoCompact.ts |
| **token budget 衰减检测** | 检测边际收益递减时提前停止 | query.ts:1308-1355 |
| **shouldPreventContinuation** | 工具执行中 hook 可阻止继续 | query.ts:1519-1521 |

---

## 四、仍然无法防御的场景

以下场景在当前架构下**没有有效防御**（除了 maxTurns 和用户 Ctrl+C）：

### 4.1 模型行为层面

| 场景 | 为什么无法防御 |
|---|---|
| 模型不断重试失败的工具调用 | 没有"同一工具同一参数重试 N 次就停止"的机制 |
| 模型陷入探索循环 | 没有"探索深度/广度限制" |
| 模型反复做无效修改 | 没有"修改效果检测"（stop hook 只检查 lint，不检查语义） |
| 模型幻觉导致的循环 | 框架无法判断模型的推理是否正确 |

### 4.2 时间/成本层面

| 缺失的防御 | 影响 |
|---|---|
| 没有总时间限制 | 每轮工具执行 30s × 100 轮 = 50 分钟 |
| 没有总 API 调用成本限制 | 循环 100 轮可能花费数十美元 |
| 没有单工具执行时间限制（除 Bash） | 某些工具可能 hang |

### 4.3 协议层面

| 场景 | 影响 |
|---|---|
| 代理/网关未正确传递 tools 定义 | 模型退化为文本模拟，循环直接退出（不是无限循环，而是**过早退出**） |
| 代理返回格式错误的 tool_use | 可能导致工具执行异常 → 模型重试 → 循环 |

---

## 五、设计哲学总结

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Claude Code 的循环退出设计哲学：                                │
│                                                                 │
│  "任务是否完成" 的判断权 100% 交给模型。                         │
│  框架只负责：                                                    │
│    1. 忠实执行模型请求的工具调用                                  │
│    2. 防止框架内部的 continue 分支形成无限循环                    │
│    3. 提供硬限制作为最后防线（maxTurns, Ctrl+C）                 │
│                                                                 │
│  框架不负责：                                                    │
│    1. 判断任务是否完成                                           │
│    2. 判断模型的工具调用是否有意义                                │
│    3. 判断模型是否陷入了无效循环                                  │
│    4. 限制总时间或总成本                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

这个设计的**优点**是简单、可预测、不会误杀正常的长任务。**缺点**是当模型行为异常时，框架无法自主检测和恢复，只能依赖外部硬限制。

---

## 六、源码位置索引

| 内容 | 文件 | 行号 |
|---|---|---|
| while (true) 主循环 | query.ts | 307 |
| needsFollowUp 初始化 | query.ts | 558 |
| needsFollowUp = true 设置 | query.ts | 834 |
| !needsFollowUp 退出分支入口 | query.ts | 1062 |
| prompt_too_long 恢复 | query.ts | 1085-1175 |
| max_output_tokens 恢复 | query.ts | 1188-1256 |
| API error 跳过 stop hook | query.ts | 1258-1264 |
| stop hook 检查 | query.ts | 1267-1306 |
| token budget 继续 | query.ts | 1308-1355 |
| 正常退出 return | query.ts | 1357 |
| 流式中断退出 | query.ts | 1051 |
| 工具执行中断退出 | query.ts | 1515 |
| maxTurns 检查 | query.ts | 1705-1711 |
| MAX_OUTPUT_TOKENS_RECOVERY_LIMIT | query.ts | 164 |
| death spiral 注释 | query.ts | 1171, 1260, 1295 |
| autocompact 熔断器 | autoCompact.ts | 70, 257-265 |
| autocompact 生产数据 | autoCompact.ts | 67-68 |
| stop hook 完整逻辑 | query/stopHooks.ts | 65-472 |
