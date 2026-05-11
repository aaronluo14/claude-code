# normalizeMessagesForAPI 多 Pass 脆弱性分析

> 源码位置：`src/utils/messages.ts:1989-2369`
>
> 本文档分析 Claude Code 中消息序列化管道的结构性脆弱问题，以及它对 agent 工程设计的启示。

---

## 一、问题背景

`normalizeMessagesForAPI` 负责将内部的多类型 `Message[]` 数组转化为 Anthropic API 要求的严格 user/assistant 交替格式。这个函数包含 **17 个步骤**，采用**多 pass（多次遍历）** 架构——每个步骤遍历一次消息数组，做一件特定的事。

代码中的原始注释（`messages.ts:2318-2320`）明确承认了这个设计的问题：

```
These multi-pass normalizations are inherently fragile — each pass can create
conditions a prior pass was meant to handle. Consider unifying into a single
pass that cleans content, then validates in one shot.
```

翻译：这些多 pass 的标准化处理**本质上是脆弱的**——每个 pass 可能创造出前一个 pass 本应处理的条件。应考虑统一成单次 pass 来清理内容，然后一次性校验。

---

## 二、具体的 Bug 案例

### 案例 1：thinking block 与空白消息的顺序依赖

**源码位置**：`messages.ts:2313-2325`

```typescript
// messages.ts:2321-2325
const withFilteredThinking =
  filterTrailingThinkingFromLastAssistant(withFilteredOrphans)   // 步骤 ⑪
const withFilteredWhitespace =
  filterWhitespaceOnlyAssistantMessages(withFilteredThinking)    // 步骤 ⑫
const withNonEmpty = ensureNonEmptyAssistantContent(withFilteredWhitespace)  // 步骤 ⑬
```

**当前代码的执行顺序**：先 ⑪（删 thinking）→ 后 ⑫（删空白）。这是正确的。

**问题场景**：假设有一条 assistant 消息：

```json
{
  "type": "assistant",
  "content": [
    { "type": "text", "text": "\n\n" },
    { "type": "thinking", "thinking": "让我想想该怎么做..." }
  ]
}
```

**正确顺序（先 ⑪ 后 ⑫）**：

```
初始:  [text("\n\n"), thinking("让我想想...")]
                          │
步骤 ⑪ 执行:              ↓ 发现末尾是 thinking → 删掉
结果:  [text("\n\n")]
                          │
步骤 ⑫ 执行:              ↓ 发现只有空白 text → 整条删掉
结果:  （消息被移除）      ✅ 正确
```

**如果顺序反了（先 ⑫ 后 ⑪）**：

```
初始:  [text("\n\n"), thinking("让我想想...")]
                          │
步骤 ⑫ 执行:              ↓ 有 text 和 thinking 两个 block → 不算空白 → 跳过
结果:  [text("\n\n"), thinking("让我想想...")]   ← 没变
                          │
步骤 ⑪ 执行:              ↓ 发现末尾是 thinking → 删掉
结果:  [text("\n\n")]

此时 ⑫ 已经执行过了，不会再执行。
最终 [text("\n\n")] 被发送给 API。
API 拒绝：内容只有空白的 assistant 消息是非法的 ❌ 报错
```

**根因**：步骤 ⑪ 的操作（删 thinking block）创造了步骤 ⑫ 应该处理的条件（空白消息），但 ⑫ 已经在 ⑪ 之前执行完了。

源码注释中对此有明确警告（`messages.ts:2313-2316`）：

```typescript
// Order matters: strip trailing thinking first, THEN filter whitespace-only
// messages. The reverse order has a bug: a message like [text("\n\n"), thinking("...")]
// survives the whitespace filter (has a non-text block), then thinking stripping
// removes the thinking block, leaving [text("\n\n")] — which the API rejects.
```

### 案例 2：孤立 thinking 消息过滤 → 相邻 user 未合并

**源码位置**：`messages.ts:2327-2338`

```typescript
// messages.ts:2327-2333
// filterOrphanedThinkingOnlyMessages doesn't merge adjacent users (whitespace
// filter does, but only when IT fires). Merge here so smoosh can fold the
// SR-text sibling that hoistToolResults produces. The smoosh itself folds
// <system-reminder>-prefixed text siblings into the adjacent tool_result.
// Gated together: the merge exists solely to feed the smoosh; running it
// ungated changes VCR fixture hashes for @-mention scenarios (adjacent
// [prompt, attachment] users) without any benefit when the smoosh is off.
const smooshed = checkStatsigFeatureGate_CACHED_MAY_BE_STALE(
  'tengu_chair_sermon',
)
  ? smooshSystemReminderSiblings(mergeAdjacentUserMessages(withNonEmpty))
  : withNonEmpty
```

**问题**：步骤 ⑩（`filterOrphanedThinkingOnlyMessages`）可能删掉一条 assistant 消息，导致两条 user 消息变成相邻的。但 ⑩ 本身不做 user 合并。步骤 ⑫（whitespace filter）里附带了合并逻辑，但只在它真正删除了消息时才触发。

**解决方式**：在步骤 ⑭ 处额外加了一次 `mergeAdjacentUserMessages()`，但只在 feature gate `tengu_chair_sermon` 开启时才执行——因为无条件执行会改变测试 fixture 的 hash。

这是一个典型的**补丁式修复**：不是修正根因（在 ⑩ 之后立即合并），而是在下游补一个条件性的合并步骤。

### 案例 3：error tool_result 中的图片

**源码位置**：`messages.ts:2340-2343`

```typescript
// messages.ts:2340-2343
// Unconditional — catches transcripts persisted before smooshIntoToolResult
// learned to filter on is_error. Without this a resumed session with an
// image-in-error tool_result 400s forever.
const sanitized = sanitizeErrorToolResultContent(smooshed)
```

**问题**：早期版本的 `smooshIntoToolResult`（步骤 ⑭ 的一部分）没有检查 `is_error` 标志，可能把图片内容折叠进了 `is_error=true` 的 tool_result 里。这些 session 被持久化到磁盘后，恢复时会永久 400 报错。

**解决方式**：在步骤 ⑮ 加了一个**无条件的清理步骤** `sanitizeErrorToolResultContent`，专门清理 error tool_result 中的图片。注释强调"Unconditional"（无条件执行）——即使新代码不会产生这种情况，但旧 session 的持久化数据可能包含这种情况。

这是**历史兼容性导致的清理步骤**：如果所有数据都是新产生的，这步是多余的；但因为要兼容旧数据，必须保留。

---

## 三、17 个步骤的完整依赖关系

以下标注了已知的顺序依赖（`→` 表示"必须在...之前"）：

```
阶段一：重排序 + 过滤
  ① reorderAttachmentsForAPI        → ⑧（冒泡后附件才在正确位置被合并）
  ② filter(isVirtual)
  ③ 构建 stripTargets               → ⑥（标记后 ⑥ 才知道该剥离哪些 block）
  ④ filter(类型)

阶段二：类型转化 + 合并
  ⑤ system → user
  ⑥ user 标准化（剥离 + 合并）      依赖 ③
  ⑦ assistant 标准化
  ⑧ attachment → user（合并）        依赖 ①

阶段三：后处理清洗（顺序敏感！）
  ⑨  relocateToolReferenceSiblings   → ⑯（移位后 tag 才能标记正确位置）
  ⑩  filterOrphanedThinking          可能导致相邻 user → 需要 ⑭ 补合并
  ⑪  filterTrailingThinking          → ⑫ ★（必须先删 thinking，否则空白检测失效）
  ⑫  filterWhitespace               依赖 ⑪
  ⑬  ensureNonEmpty                  依赖 ⑫
  ⑭  smoosh + mergeAdjacent          依赖 ⑩（补 ⑩ 遗漏的合并）
  ⑮  sanitizeErrorToolResult         依赖 ⑭（清理 smoosh 的历史遗留）
  ⑯  appendMessageTag               依赖 ⑨⑭（所有合并/移位完成后才打标签）
  ⑰  validateImages                 （最终校验，无前置依赖）

★ = 已被注释明确警告的关键依赖
```

---

## 四、为什么是"本质上脆弱的"

### 4.1 隐式依赖

步骤之间的依赖关系**没有在类型系统或代码结构中表达**。它们只存在于：
- 代码注释中（如 2313-2316 行的顺序警告）
- 开发者的记忆中
- 偶尔的测试 fixture 中

没有编译器或 lint 规则能阻止你调换 ⑪ 和 ⑫ 的顺序。

### 4.2 级联效应

任何一个新步骤的添加都可能：
1. 创造出**上游步骤应该处理**的条件（如删除 block 导致消息变空）
2. 破坏**下游步骤的前提假设**（如合并消息导致 tool_reference 位置变化）
3. 需要在**其他位置补一个修复步骤**（如 ⑭ 补 ⑩ 的合并遗漏）

### 4.3 测试脆弱

修改一个步骤的行为会改变其输出的消息结构（如消息 hash），导致：
- VCR 测试 fixture 失效（注释中多次提到 "changes VCR fixture hashes"）
- 某些修复只能在 feature gate 后面执行（如 ⑭ 的 `tengu_chair_sermon` 门控）
- 无法轻易做 A/B 测试，因为多步骤的交互效应难以隔离

### 4.4 历史兼容负担

步骤 ⑮ 的存在完全是因为旧版本的持久化数据。即使新代码不再产生有问题的数据，这个步骤也不能删除。随着时间推移，这类"兼容性清理"步骤只会越来越多。

---

## 五、理想方案：单 Pass 架构

源码注释建议的改进方向：

```
Consider unifying into a single pass that cleans content, then validates in one shot.
```

### 当前架构 vs 理想架构

```
当前（多 Pass）：                         理想（单 Pass）：

messages                                  messages
  → pass1: reorder                          → for (msg of messages) {
  → pass2: filter virtual                       msg = convertType(msg)
  → pass3: build strip map                      msg = cleanThinking(msg)
  → pass4: filter types                         msg = cleanWhitespace(msg)
  → pass5: convert types                        msg = cleanContent(msg)
  → ...                                         msg = validate(msg)
  → pass11: strip thinking                      if (isValid(msg))
  → pass12: filter whitespace                     mergeOrPush(result, msg)
  → ...                                     }
  → pass17: validate images                 → validateAll(result)
  → return                                  → return

每步只看前一步的输出，                     对每条消息原子完成所有操作，
看不到后续步骤的影响。                     不存在"A 处理后 B 看不到"的问题。
```

### 单 Pass 的优势

| 维度 | 多 Pass | 单 Pass |
|---|---|---|
| 顺序依赖 | 17 步之间存在隐式依赖，调换可能导致 bug | 每条消息独立处理，无步骤间依赖 |
| 新增步骤 | 需要分析与所有现有步骤的交互 | 只需在消息处理函数中加逻辑 |
| 可测试性 | 需要端到端测试完整管道 | 可以单独测试单条消息的处理 |
| 性能 | 17 次遍历（常数因子大） | 1 次遍历（常数因子小） |
| 历史兼容 | 步骤越来越多 | 兼容逻辑内聚在单条消息处理中 |

### 单 Pass 的难点

- **重排序需求**：attachment 冒泡（步骤 ①）本质上需要全局视野，单条消息级别的处理做不到
- **跨消息合并**：连续 user 合并、同 ID assistant 合并都需要看"前一条消息"
- **回归风险**：380 行代码的重构，可能引入新 bug
- **feature gate 耦合**：多个步骤被 gate 控制，重构需要同步处理实验逻辑

---

## 六、对 Agent 工程设计的启示

### 1. 消息序列化应作为独立模块设计

不要让序列化逻辑散落在各处。`normalizeMessagesForAPI` 虽然是独立函数，但它的 17 步管道已经超出了一个函数应有的复杂度。应该进一步拆分为：
- **类型转换层**：内部类型 → API 类型
- **清洗层**：内容清理、格式校验
- **合并层**：连续消息合并

### 2. 尽量减少步骤间的隐式依赖

如果必须用多 pass：
- 用**类型标记**表达依赖（如 `ThinkingStripped<Message>` 类型表示已删 thinking）
- 用**单元测试**覆盖所有已知的顺序依赖
- 用**注释**明确标注 "must run before/after X"

### 3. 内部表示和 API 格式要彻底解耦

Claude Code 的做法是正确的——内部用丰富的多态类型（`user/assistant/system/attachment/progress`），API 发送前统一转换。但转换函数不应该承担这么多职责。

### 4. 考虑不可变数据流

每个步骤都返回新数组（而非原地修改），这是好的。但更好的做法是用**管道组合**（pipeline composition）让依赖关系显式化：

```typescript
const pipeline = compose(
  reorderAttachments,      // 全局重排序
  convertAndMerge,         // 类型转换 + 合并（单 pass）
  cleanAndValidate,        // 清洗 + 校验（单 pass）
  tagForSnip,              // 标记（单 pass）
)
return pipeline(messages)
```

---

## 七、相关源码位置索引

| 内容 | 文件 | 行号 |
|---|---|---|
| `normalizeMessagesForAPI` 函数入口 | `src/utils/messages.ts` | 1989 |
| attachment 冒泡重排序 | `src/utils/messages.ts` | 1481-1527 |
| attachment → user 转化 | `src/utils/messages.ts` | 3453-3640 |
| **顺序依赖 bug 注释** | `src/utils/messages.ts` | **2313-2320** |
| filterTrailingThinking | `src/utils/messages.ts` | 2321-2322 |
| filterWhitespaceOnly | `src/utils/messages.ts` | 2323-2324 |
| smoosh + merge 补丁 | `src/utils/messages.ts` | 2327-2338 |
| sanitizeErrorToolResult | `src/utils/messages.ts` | 2340-2343 |
| appendMessageTag (snip) | `src/utils/messages.ts` | 2345-2364 |
| validateImages | `src/utils/messages.ts` | 2366-2367 |
| "inherently fragile" 原文 | `src/utils/messages.ts` | **2318** |
| 调用位置 (claude.ts) | `src/services/api/claude.ts` | 1266 |
