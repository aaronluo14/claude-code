# Claude Code 文件工具机制深度分析

> 源码版本：2026-03-31 泄露快照
>
> 核心文件：
> - `src/tools/FileReadTool/FileReadTool.ts`
> - `src/tools/FileWriteTool/FileWriteTool.ts`
> - `src/tools/FileEditTool/FileEditTool.ts`
> - `src/utils/file.ts`

---

## 一、核心结论

**模型不能直接接触文件系统。** 所有文件操作（读、写、编辑）都是通过标准的 `tool_use`（function calling）协议完成的。

整个交互的本质是：

```
模型生成 tool_use tokens
  → 框架解析并执行文件操作（带安全校验）
  → 框架把结果格式化为 tool_result tokens
  → 拼回 token 序列
  → 模型继续做 next token prediction
```

模型的全部"智能"来自对 token 序列的预测；框架是"循环驱动器"，把模型的输出转化为副作用，再把副作用结果喂回模型。

---

## 二、三个核心工具

### 2.1 工具一览表

| 工具名 | 工具类 | 用途 | 输入字段 |
|---|---|---|---|
| `Read` | `FileReadTool` | 读取文件内容 | `file_path`, `offset?`, `limit?`, `pages?`（PDF） |
| `Write` | `FileWriteTool` | 全量写入/创建文件 | `file_path`, `content` |
| `Edit` | `FileEditTool` | 精确字符串替换 | `file_path`, `old_string`, `new_string`, `replace_all?` |

### 2.2 模型看到的工具描述

**Read 工具的 prompt**（`FileReadTool/prompt.ts`）：

```
Reads a file from the local filesystem. You can access any file
directly by using this tool.

Usage:
- The file_path parameter must be an absolute path, not a relative path
- By default, it reads up to 2000 lines starting from the beginning
- Results are returned using cat -n format, with line numbers starting at 1
- This tool allows Claude Code to read images (eg PNG, JPG, etc).
  When reading an image file the contents are presented visually...
- This tool can read PDF files (.pdf)...
- This tool can read Jupyter notebooks (.ipynb files)...
- This tool can only read files, not directories.
```

**Write 工具的 prompt**（`FileWriteTool/prompt.ts`）：

```
Writes a file to the local filesystem.

Usage:
- This tool will overwrite the existing file if there is one
- If this is an existing file, you MUST use the Read tool first
- Prefer the Edit tool for modifying existing files — it only
  sends the diff. Only use this tool to create new files or for
  complete rewrites.
- NEVER create documentation files (*.md) or README files unless
  explicitly requested.
```

**Edit 工具的 prompt**（`FileEditTool/prompt.ts`）：

```
Performs exact string replacements in files.

Usage:
- You must use your Read tool at least once before editing
- When editing text from Read tool output, ensure you preserve
  the exact indentation as it appears AFTER the line number prefix
- ALWAYS prefer editing existing files
- The edit will FAIL if old_string is not unique
- Use replace_all for renaming across the file
```

---

## 三、Read 工具：把文件内容喂给模型

### 3.1 模型的请求

```json
{
  "type": "tool_use",
  "id": "toolu_abc123",
  "name": "Read",
  "input": {
    "file_path": "/home/user/project/src/utils.ts",
    "offset": 1,
    "limit": 100
  }
}
```

### 3.2 框架的执行流程

```
FileReadTool.call()
  │
  ├─ ① 重复读取检测（dedup）
  │   └─ 同一文件、同一范围、mtime 未变 → 返回 stub
  │      "File unchanged since last read..."
  │      （节省 ~18% Read 调用的 cache_creation tokens）
  │
  ├─ ② validateInput()
  │   ├─ 权限检查（deny rules）
  │   ├─ 二进制扩展名检查（除 PDF/图片外的二进制拒绝）
  │   ├─ 阻塞设备路径检查（/dev/zero, /dev/random 等）
  │   └─ PDF 页数范围校验
  │
  ├─ ③ 按文件类型分支处理
  │   ├─ Notebook (.ipynb) → readNotebook() → cells 数组
  │   ├─ Image (.png/.jpg/...) → readImageWithTokenBudget() → base64
  │   ├─ PDF (.pdf) → readPDF() / extractPDFPages() → base64
  │   └─ Text → readFileInRange() → 文本内容
  │
  ├─ ④ Token 数检查
  │   └─ validateContentTokens()
  │       超过 maxTokens 抛 MaxFileReadTokenExceededError
  │
  ├─ ⑤ 记录到 readFileState
  │   └─ { content, timestamp: mtime, offset, limit }
  │      （供后续 Write/Edit 做乐观锁检查）
  │
  ├─ ⑥ 触发副作用
  │   ├─ skill discovery（基于文件路径）
  │   ├─ activateConditionalSkillsForPaths()
  │   └─ fileReadListeners 广播
  │
  └─ ⑦ 返回结构化 Output
      { type: 'text', file: { content, numLines, startLine, totalLines } }
```

### 3.3 序列化为 tool_result —— 行号格式

文本文件最终发给模型的内容由 `addLineNumbers()` 格式化（`src/utils/file.ts:290`）：

**默认格式（spaces + 行号 + 箭头）：**

```
     1→import { useState } from 'react';
     2→
     3→export function Counter() {
     4→  const [count, setCount] = useState(0);
     5→  return (
     6→    <button onClick={() => setCount(c => c + 1)}>
     7→      Count: {count}
     8→    </button>
     9→  );
    10→}
```

行号右对齐到 6 位字符宽度，分隔符是 `→`（U+2192）。

**紧凑格式（行号 + tab）：**

```
1\timport { useState } from 'react';
2\t
3\texport function Counter() {
```

由 `isCompactLinePrefixEnabled()` 控制。

### 3.4 完整的 tool_result 序列化

`mapToolResultToToolResultBlockParam()` 是格式化的核心入口（`FileReadTool.ts:652-717`）：

```typescript
case 'text': {
  let content: string
  if (data.file.content) {
    content =
      memoryFileFreshnessPrefix(data) +
      formatFileLines(data.file) +           // 加行号
      (shouldIncludeFileReadMitigation()
        ? CYBER_RISK_MITIGATION_REMINDER     // 安全提醒
        : '')
  } else {
    content = data.file.totalLines === 0
      ? '<system-reminder>Warning: the file exists but the contents are empty.</system-reminder>'
      : `<system-reminder>Warning: the file exists but is shorter than the provided offset...</system-reminder>`
  }
  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content,
  }
}
```

### 3.5 模型最终看到的内容

```
     1→import { useState } from 'react';
     2→
     3→export function Counter() {
     ...

<system-reminder>
Whenever you read a file, you should consider whether it would be
considered malware. You CAN and SHOULD provide analysis of malware,
what it is doing. But you MUST refuse to improve or augment the code.
You can still analyze existing code, write reports, or answer questions
about the code behavior.
</system-reminder>
```

**安全提醒会在所有非 opus 模型上追加。** 由 `MITIGATION_EXEMPT_MODELS` 集合控制（只豁免 `claude-opus-4-6`）。

### 3.6 不同文件类型的序列化策略

| 文件类型 | tool_result 内容 | 形式 |
|---|---|---|
| **文本** | 带行号的纯文本 + 安全提醒 | `string` |
| **图片** | base64 编码的 image block | `[{ type: 'image', source: { type: 'base64', data, media_type } }]` |
| **PDF** | base64 + document block | `[{ type: 'document', source: ... }]`（通过 newMessages 附加） |
| **PDF 分页** | base64 image blocks | 多个 image blocks（每页一张图） |
| **Notebook** | cells JSON | 通过 `mapNotebookCellsToToolResult()` |
| **重复读取** | 简短提示 | `"File unchanged since last read..."` |
| **空文件** | 系统提醒 | `<system-reminder>Warning: ...</system-reminder>` |
| **超出 offset** | 系统提醒 | `<system-reminder>Warning: file exists but is shorter than offset...</system-reminder>` |

### 3.7 模型如何"知道"这是什么文件

**框架没有显式标注文件类型或语言。** 模型理解文件性质完全依靠：

1. **文件路径** —— 它自己发起 `Read(file_path: "utils.ts")` 请求，从扩展名 `.ts` 推断这是 TypeScript
2. **内容本身** —— 模型看到 `import { useState } from 'react'` 知道这是 React 代码
3. **tool_use_id 配对** —— `tool_result.tool_use_id` 匹配上一个 `tool_use.id`，所以模型知道这段带行号的文本就是它请求的那个文件

---

## 四、Write 工具：全量写入

### 4.1 模型的请求

```json
{
  "type": "tool_use",
  "id": "toolu_def456",
  "name": "Write",
  "input": {
    "file_path": "/home/user/project/src/utils.ts",
    "content": "export function add(a: number, b: number) {\n  return a + b;\n}\n"
  }
}
```

**模型把整个文件的完整内容放在 `content` 字段中。**

### 4.2 框架的执行流程

```
FileWriteTool.call()
  │
  ├─ ① validateInput()
  │   ├─ Team memory secret 检查
  │   ├─ 权限规则（deny list）
  │   ├─ UNC 路径安全（Windows 防 NTLM 凭据泄漏）
  │   ├─ 读后写约束:
  │   │   如果文件已存在但 readFileState 没有记录
  │   │   → "File has not been read yet. Read it first."
  │   └─ 乐观锁:
  │       如果 mtime > readTimestamp
  │       → "File has been modified since read"
  │
  ├─ ② 副作用预处理
  │   ├─ skill 发现 + 激活
  │   └─ diagnosticTracker.beforeFileEdited()
  │
  ├─ ③ 写入前准备（原子区前）
  │   ├─ mkdir(parent_dir) 确保父目录存在
  │   └─ fileHistoryTrackEdit() 备份原始内容
  │
  ├─ ④ 原子区（read-modify-write）
  │   ├─ readFileSyncWithMetadata() 读取当前状态
  │   ├─ 二次校验 mtime（防止 race condition）
  │   └─ writeTextContent() 写入磁盘
  │       ├─ 编码：utf8 / utf16le
  │       └─ 行尾：始终用模型提供的 LF（不再保留原文件风格）
  │
  ├─ ⑤ 通知下游
  │   ├─ LSP didChange + didSave（触发 lint 诊断）
  │   ├─ notifyVscodeFileUpdated（VSCode diff view）
  │   └─ readFileState.set 更新时间戳
  │
  └─ ⑥ 返回结果
      ├─ create: type='create', structuredPatch=[]
      └─ update: type='update', structuredPatch=patch
```

### 4.3 tool_result 序列化

```typescript
mapToolResultToToolResultBlockParam({ filePath, type }, toolUseID) {
  switch (type) {
    case 'create':
      return {
        tool_use_id: toolUseID,
        type: 'tool_result',
        content: `File created successfully at: ${filePath}`,
      }
    case 'update':
      return {
        tool_use_id: toolUseID,
        type: 'tool_result',
        content: `The file ${filePath} has been updated successfully.`,
      }
  }
}
```

**模型只收到一句确认消息，不再回传写入的内容。** 因为模型自己提供的 content 就是真相，无需重复。

### 4.4 关键约束："必须先 Read"

```typescript
// FileWriteTool.ts:198-205
const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
if (!readTimestamp || readTimestamp.isPartialView) {
  return {
    result: false,
    message: 'File has not been read yet. Read it first before writing to it.',
    errorCode: 2,
  }
}
```

设计目的：**防止模型盲目覆盖未知内容的文件**。这强制了 read-before-write 的工作模式。

### 4.5 关键约束：换行符策略

```typescript
// FileWriteTool.ts:300-305
// Write is a full content replacement — the model sent explicit line endings
// in `content` and meant them. Do not rewrite them. Previously we preserved
// the old file's line endings (or sampled the repo via ripgrep for new
// files), which silently corrupted e.g. bash scripts with \r on Linux when
// overwriting a CRLF file or when binaries in cwd poisoned the repo sample.
writeTextContent(fullFilePath, content, enc, 'LF')
```

Write 工具**强制使用 LF**——不再保留原文件的 CRLF 风格。这是一个明确的 bug 修复：之前保留 CRLF 会在 Linux 上让 bash 脚本被破坏。

---

## 五、Edit 工具：精确字符串替换

### 5.1 模型的请求

```json
{
  "type": "tool_use",
  "id": "toolu_ghi789",
  "name": "Edit",
  "input": {
    "file_path": "/home/user/project/src/utils.ts",
    "old_string": "  return a + b;",
    "new_string": "  return a + b + 1;",
    "replace_all": false
  }
}
```

**模型只发送需要替换的片段**，不是整个文件。这就是为什么 prompt 说 "Prefer the Edit tool for modifying existing files — it only sends the diff"。

### 5.2 框架的执行流程

```
FileEditTool.call()
  │
  ├─ ① validateInput()
  │   ├─ old_string !== new_string（拒绝空操作）
  │   ├─ 文件大小限制（1 GiB 防 OOM）
  │   ├─ 文件存在性检查:
  │   │   ├─ 不存在 + old_string === '' → 允许创建
  │   │   ├─ 不存在 + old_string !== '' → 拒绝 + 建议相似路径
  │   │   └─ 存在 + old_string === '' + 非空文件 → 拒绝
  │   ├─ ipynb 拒绝（应使用 NotebookEditTool）
  │   ├─ Read-before-Edit 约束
  │   ├─ 乐观锁（mtime 检查）
  │   ├─ findActualString() 智能匹配
  │   │   （处理引号风格：" vs " 等）
  │   ├─ 唯一性检查:
  │   │   如果 matches > 1 且 !replace_all
  │   │   → "Found N matches... provide more context or use replace_all"
  │   └─ Claude 配置文件特殊校验
  │       （validateInputForSettingsFileEdit）
  │
  ├─ ② 副作用预处理
  │   ├─ skill 发现
  │   └─ diagnosticTracker.beforeFileEdited()
  │
  ├─ ③ 原子区
  │   ├─ readFileForEdit() 读取当前内容
  │   ├─ 二次 mtime 校验
  │   ├─ findActualString() 重新定位
  │   ├─ preserveQuoteStyle() 保留引号风格
  │   ├─ getPatchForEdit() 生成 patch
  │   │   ├─ file.replace(oldStr, newStr)（默认）
  │   │   └─ file.replaceAll(oldStr, newStr)（replace_all=true）
  │   └─ writeTextContent() 写入磁盘
  │       └─ 保留原文件的编码和行尾风格
  │
  ├─ ④ 通知下游
  │   ├─ LSP didChange + didSave
  │   └─ notifyVscodeFileUpdated
  │
  └─ ⑤ 返回 patch + 元数据
```

### 5.3 唯一性检查（Edit 的核心约束）

```typescript
// FileEditTool.ts:329-343
const matches = file.split(actualOldString).length - 1
if (matches > 1 && !replace_all) {
  return {
    result: false,
    behavior: 'ask',
    message: `Found ${matches} matches of the string to replace, but
              replace_all is false. To replace all occurrences, set
              replace_all to true. To replace only one occurrence,
              please provide more context to uniquely identify the
              instance.\nString: ${old_string}`,
    errorCode: 9,
  }
}
```

这强制模型：
- 要么提供足够上下文让 `old_string` 唯一匹配
- 要么显式声明 `replace_all: true`

防止意外修改多处相同代码。

### 5.4 智能引号匹配

```typescript
// FileEditTool.ts:316-322
const actualOldString = findActualString(file, old_string)
```

`findActualString` 处理一些常见的字符变体：
- 直引号 `"` vs 弯引号 `"` `"`
- 直单引号 `'` vs 弯单引号 `'` `'`
- 不同的破折号变体

这样模型可以用 ASCII 引号写 `old_string`，即使文件里是 Unicode 引号也能匹配上。然后 `preserveQuoteStyle()` 确保 `new_string` 也用对应的风格。

### 5.5 tool_result 序列化

```typescript
mapToolResultToToolResultBlockParam(data, toolUseID) {
  const { filePath, userModified, replaceAll } = data
  const modifiedNote = userModified ? '... user modified ...' : ''
  if (replaceAll) {
    return {
      tool_use_id: toolUseID,
      type: 'tool_result',
      content: `The file ${filePath} has been updated${modifiedNote}.
                All occurrences were successfully replaced.`,
    }
  }
  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: `The file ${filePath} has been updated successfully${modifiedNote}.`,
  }
}
```

---

## 六、共同的安全机制

### 6.1 Read-before-Write 约束

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   未读取过的文件不能被 Write 或 Edit                     │
│                                                         │
│   readFileState (Map<filePath, ReadState>)              │
│     ↑                                                   │
│     │ Read tool 写入                                    │
│     ↓                                                   │
│   Write/Edit tool 校验：必须存在记录                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**目的**：防止模型基于幻觉覆盖未知内容的文件。

### 6.2 乐观锁（Stale Read Detection）

```typescript
const lastWriteTime = getFileModificationTime(fullFilePath)
const lastRead = readFileState.get(fullFilePath)
if (!lastRead || lastWriteTime > lastRead.timestamp) {
  // Windows mtime 可能在内容未变时变化（云同步、防病毒等）
  // 完整读取时，对比内容作为兜底
  const isFullRead = lastRead && lastRead.offset === undefined
  const contentUnchanged = isFullRead && originalContents === lastRead.content
  if (!contentUnchanged) {
    throw new Error(FILE_UNEXPECTEDLY_MODIFIED_ERROR)
  }
}
```

**目的**：防止用户/linter 在 Read 和 Write/Edit 之间修改了文件，导致模型基于过时内容做出错误编辑。

### 6.3 权限检查

```typescript
const denyRule = matchingRuleForInput(
  fullFilePath,
  appState.toolPermissionContext,
  'edit' | 'read',
  'deny',
)
if (denyRule !== null) {
  return { result: false, message: '...denied by permission settings.' }
}
```

通过 `.claude/settings.json` 的权限规则可以禁止读/写特定路径。

### 6.4 Secret 检查

```typescript
const secretError = checkTeamMemSecrets(fullFilePath, content)
if (secretError) {
  return { result: false, message: secretError }
}
```

防止模型将密钥写入 team memory 文件（会被自动同步）。

### 6.5 UNC 路径防护

```typescript
// SECURITY: On Windows, fs.existsSync() on UNC paths triggers SMB
// authentication which could leak credentials to malicious servers.
if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
  return { result: true }   // 让权限检查后置处理
}
```

防止 Windows 上的 NTLM 凭据泄漏。

### 6.6 fileHistory 备份

```typescript
if (fileHistoryEnabled()) {
  await fileHistoryTrackEdit(
    updateFileHistoryState,
    absoluteFilePath,
    parentMessage.uuid,
  )
}
```

每次写入前自动备份原始内容，支持 undo / 回滚。备份按内容哈希索引（idempotent），失败的 staleness 检查只是留下未用的备份，不会损坏状态。

---

## 七、副作用与集成

### 7.1 LSP 集成

每次文件变更后通知 LSP server：

```typescript
const lspManager = getLspServerManager()
if (lspManager) {
  clearDeliveredDiagnosticsForFile(`file://${fullFilePath}`)
  lspManager.changeFile(fullFilePath, content)  // didChange
  lspManager.saveFile(fullFilePath)              // didSave (触发诊断)
}
```

这让 TypeScript / Python LSP 等能立即重新分析变更，结果通过 attachment 注入到下一轮上下文中。

### 7.2 VSCode 集成

```typescript
notifyVscodeFileUpdated(fullFilePath, oldContent, content)
```

VSCode 扩展会接收到变更通知，可以显示 diff view。

### 7.3 Skill 发现

```typescript
const newSkillDirs = await discoverSkillDirsForPaths([fullFilePath], cwd)
if (newSkillDirs.length > 0) {
  addSkillDirectories(newSkillDirs).catch(() => {})
}
activateConditionalSkillsForPaths([fullFilePath], cwd)
```

读/写一个文件可能触发条件 skill 的激活（例如读到 `package.json` 触发 npm 相关 skill）。

### 7.4 fileReadListeners

```typescript
for (const listener of fileReadListeners.slice()) {
  listener(resolvedFilePath, content)
}
```

其他服务可以注册监听器，在文件被读取时收到通知（例如内存提取服务）。

---

## 八、Tokenization 视角

### 8.1 整个交互的本质

最终送给 Claude 模型的，是这样的一段 token 序列：

```
[system_prompt tokens...]
[user_message tokens...]
[assistant tokens: "让我读取这个文件。" + tool_use(Read, ...)]
[user tokens: tool_result("     1→import { useState }...")]
[assistant tokens: tool_use(Edit, ...)]
[user tokens: tool_result("The file ... has been updated successfully.")]
[assistant tokens: "我已经修改完成了。"]
                                ↑
                          next token prediction
```

**模型对此的认知**：这就是一段普通的 token 序列。所谓的 "function calling"，在模型视角是它在训练阶段学会的一种特殊语法约定——当我想读文件时，应该生成符合 `tool_use` 格式的 token；当我看到 `tool_result` token 时，那是工具的执行结果。

### 8.2 Agent Loop 的本质

```
while (true) {
  response = model.predict(tokens)        // 纯 next token prediction
  if (response 包含 tool_use) {
    result = execute_tool(tool_use)        // 框架副作用：读/写文件
    tokens.append(tool_result(result))     // 把结果拼回 token 序列
    continue
  } else {
    break                                  // 模型生成了纯文本，结束
  }
}
```

**框架做的所有事情——文件 I/O、权限检查、行号添加、安全提醒注入、LSP 通知——都发生在 token 预测之外**。框架是"循环驱动器"，把模型的 token 输出转化为副作用，再把副作用的结果转化为新的 token 喂回去。

### 8.3 这种设计的几个关键洞见

1. **模型不需要"懂"文件系统** —— 它只需要懂 tool_use 协议。所有文件系统的细节（路径、编码、权限、原子性）都被框架抽象掉了。

2. **协议层兼容性至关重要** —— 如果 API 网关（如某些 Bedrock 代理）不支持 tool_use 协议，模型只能在文本中"模拟"调用（XML 标签），但框架不解析这种文本，循环直接退出。

3. **上下文管理就是 token 管理** —— 文件越大 → tool_result 越长 → token 数越多 → 越接近上下文窗口上限。这就是为什么 Read 有 `offset/limit`、有 dedup、有 token 数检查。

4. **行号是为了引用** —— 行号本身不是文件的一部分，框架加上去是为了让模型在 Edit 时能用 "第 N 行附近的代码" 这种方式精确指代。Edit 的 prompt 明确告诉模型："Never include any part of the line number prefix in the old_string or new_string."

---

## 九、完整端到端流程图

```
┌────────────────────────────────────────────────────────────────┐
│ 用户: "把 utils.ts 里的 add 函数改成返回 a+b+1"                  │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│ 第 1 轮：模型决定先读文件                                        │
│                                                                │
│   assistant:                                                   │
│     [text] "我先看看当前的实现。"                               │
│     [tool_use] Read(file_path="/.../utils.ts")                 │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│ 框架: FileReadTool.call()                                       │
│   ├─ 权限/路径检查                                              │
│   ├─ readFileInRange() 从磁盘读取                               │
│   ├─ readFileState.set() 记录 mtime                            │
│   └─ addLineNumbers() 加行号                                    │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│ 第 2 轮：模型收到文件内容                                        │
│                                                                │
│   user:                                                        │
│     [tool_result tool_use_id="...."]                           │
│       "     1→export function add(a, b) {                      │
│             2→  return a + b;                                  │
│             3→}                                                │
│        <system-reminder>...malware check...</system-reminder>" │
│                                                                │
│   模型决定如何修改 → 生成 Edit tool_use                         │
│                                                                │
│   assistant:                                                   │
│     [tool_use] Edit(                                           │
│       file_path="/.../utils.ts",                               │
│       old_string="  return a + b;",                            │
│       new_string="  return a + b + 1;"                         │
│     )                                                          │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│ 框架: FileEditTool.call()                                       │
│   ├─ 校验 Read-before-Edit                                      │
│   ├─ 校验 mtime（乐观锁）                                       │
│   ├─ 校验 old_string 唯一性                                     │
│   ├─ findActualString() 智能匹配                                │
│   ├─ file.replace() 生成新内容                                  │
│   ├─ writeTextContent() 写入磁盘                                │
│   ├─ LSP / VSCode 通知                                          │
│   └─ readFileState.set() 更新 mtime                            │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│ 第 3 轮：模型收到成功确认                                        │
│                                                                │
│   user:                                                        │
│     [tool_result] "The file /.../utils.ts has been updated     │
│                    successfully."                              │
│                                                                │
│   模型决定回复用户 → 生成纯文本                                  │
│                                                                │
│   assistant:                                                   │
│     [text] "已经修改完成，现在 add(a, b) 返回 a+b+1。"          │
│                                                                │
│   没有 tool_use → queryLoop 退出                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 十、源码位置索引

| 内容 | 文件 | 行号 |
|---|---|---|
| **FileReadTool** | | |
| FileReadTool 定义 | FileReadTool.ts | 337 |
| Input Schema | FileReadTool.ts | 227-243 |
| Output Schema | FileReadTool.ts | 248-332 |
| call() 入口 | FileReadTool.ts | 496-651 |
| 重复读取 dedup | FileReadTool.ts | 540-573 |
| callInner() 核心 | FileReadTool.ts | 804-1086 |
| mapToolResultToToolResultBlockParam | FileReadTool.ts | 652-717 |
| CYBER_RISK_MITIGATION_REMINDER | FileReadTool.ts | 729-738 |
| readImageWithTokenBudget | FileReadTool.ts | 1097-1183 |
| Read prompt 模板 | FileReadTool/prompt.ts | 27-49 |
| addLineNumbers | utils/file.ts | 290-319 |
| **FileWriteTool** | | |
| FileWriteTool 定义 | FileWriteTool.ts | 94 |
| Input Schema | FileWriteTool.ts | 56-65 |
| validateInput | FileWriteTool.ts | 153-222 |
| call() | FileWriteTool.ts | 223-417 |
| 强制 LF 注释 | FileWriteTool.ts | 300-305 |
| mapToolResultToToolResultBlockParam | FileWriteTool.ts | 418-433 |
| writeTextContent | utils/file.ts | 84-98 |
| Write prompt 模板 | FileWriteTool/prompt.ts | 10-18 |
| **FileEditTool** | | |
| FileEditTool 定义 | FileEditTool.ts | 86 |
| validateInput | FileEditTool.ts | 137-362 |
| 唯一性检查 | FileEditTool.ts | 329-343 |
| findActualString | FileEditTool.ts | 316 |
| call() | FileEditTool.ts | 387-574 |
| mapToolResultToToolResultBlockParam | FileEditTool.ts | 575-594 |
| readFileForEdit | FileEditTool.ts | 599-625 |
| Edit prompt 模板 | FileEditTool/prompt.ts | 12-28 |
| Input/Output Schema | FileEditTool/types.ts | 6-83 |
