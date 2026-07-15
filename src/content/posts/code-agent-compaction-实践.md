---
title: 从零实现一个 Code Agent 上下文压缩：300 行代码跑通完整流程
published: 2026-07-15
pinned: false
description: ""
tags: [code-agent, context-compaction, typescript, 实践]
category: AI
draft: false
---

[原理篇](/posts/code-agent-compaction-原理/)讲了设计空间，[源码篇](/posts/code-agent-compaction-源码实现/)讲了 5 个项目的实现。这篇动手写一个最小可用的上下文压缩，覆盖核心机制：token 估算、触发条件、切点选择、LLM 总结、溢出恢复、文件追踪。

代码不依赖任何外部库，纯 TypeScript，`npx tsx context-compaction.ts` 就能跑。

## 整体架构

```mermaid
flowchart TB
    subgraph 输入["对话循环"]
        A["添加消息"] --> B{"token 超阈值？"}
        B -->|否| A
        B -->|是| C["执行压缩"]
        C --> A
    end

    subgraph 压缩["压缩流程"]
        C --> D["1. 找切点<br/>滑动窗口 + toolResult 不能切"]
        D --> E["2. 拆分 head/recent"]
        E --> F["3. 提取文件操作"]
        F --> G["4. LLM 总结 head<br/>增量更新"]
        G --> H["5. 摘要 + recent + 文件追踪<br/>替换旧消息"]
    end

    subgraph 溢出["溢出恢复"]
        I["LLM 返回 overflow"] --> J{"已重试过？"}
        J -->|否| K["删 error 消息<br/>压缩<br/>重试"]
        J -->|是| L["放弃"]
    end
```

## 类型定义

```typescript
type Role = "system" | "user" | "assistant" | "tool";

interface Message {
  role: Role;
  content: string;
  toolCallId?: string;
  toolName?: string;
  isToolResult?: boolean;
}

interface CompactionConfig {
  contextWindow: number;    // 上下文窗口大小（token）
  reserveTokens: number;    // 触发阈值 buffer
  keepRecentTokens: number; // 保留近期消息的 token 预算
  maxSummaryTokens: number; // 摘要输出上限
}
```

`toolCallId` 和 `isToolResult` 用来标识 tool call 和 tool result 的配对关系。这是切点选择的关键约束。

## Token 估算

```typescript
function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);
}

function estimateMessageTokens(msg: Message): number {
  let tokens = estimateTokens(msg.content);
  if (msg.toolName) tokens += estimateTokens(msg.toolName);
  if (msg.toolCallId) tokens += 10;
  return tokens;
}

function estimateTotalTokens(messages: Message[]): number {
  return messages.reduce((sum, msg) => sum + estimateMessageTokens(msg), 0);
}
```

用 `chars/4` 启发式。为什么不用精确 tokenizer？因为 keepRecent 是预算不是硬限制，估算差一点不影响正确性。精确 tokenize 每条消息的开销太大，每次压缩都跑一遍不划算。pi 用的也是这个方法（`compaction.ts:240`）。

图片按 4800 字符估算（跟 pi 一致），这里没实现但加上很简单。

## 触发条件

```typescript
function shouldCompact(messages: Message[], config: CompactionConfig): boolean {
  const totalTokens = estimateTotalTokens(messages);
  const threshold = config.contextWindow - config.reserveTokens;
  return totalTokens > threshold;
}
```

逻辑跟 pi 的 `shouldCompact` 一致：`contextTokens > contextWindow - reserveTokens`。

200K 窗口、reserveTokens 16384，约 184K 触发（92%）。demo 里故意设小（600 窗口、150 buffer），方便触发。

为什么需要 buffer 而不是等满了再压缩？两个原因。第一，模型生成回复需要空间。第二，压缩过程本身也消耗 token：压缩要发一个总结请求，包含整个历史，buffer 太小可能压缩请求都发不出去。

## 切点选择

这是压缩算法的核心。切点之后的消息原样保留，切点之前的送给 LLM 总结。

```typescript
function isCutPointMessage(msg: Message): boolean {
  if (msg.isToolResult) return false;
  return true;
}

function findCutPoint(messages: Message[], config: CompactionConfig): number {
  let accumulatedTokens = 0;
  let cutIndex = 0;

  for (let i = messages.length - 1; i >= 0; i--) {
    accumulatedTokens += estimateMessageTokens(messages[i]);

    if (accumulatedTokens >= config.keepRecentTokens) {
      // 找 >= i 的最近合法切点
      for (let j = i; j < messages.length; j++) {
        if (isCutPointMessage(messages[j])) {
          cutIndex = j;
          break;
        }
      }
      break;
    }
  }

  return cutIndex;
}
```

从尾部向前累加 token，达到 `keepRecentTokens` 时停。然后在该位置找最近的合法切点。

`toolResult` 绝不能切。为什么？因为 tool_call 和 tool_result 必须成对保留。如果切点落在 tool_result 中间，模型看到 tool_call 但看不到完整 result，会混乱。这也是 Anthropic 和 OpenAI API 的硬性约束：tool_call 和 tool_result 必须配对出现。

当切点落在一个带 tool call 的 assistant 消息处时，其后续的 tool result 会跟着保留（因为在 cutIndex 之后）。这跟 pi 的 `isCutPointMessage`（`compaction.ts:282`）逻辑一致。

## 文件操作追踪

压缩后模型丢失了历史细节，但"哪些文件被读过/改过"是关键状态。如果模型不知道自己改过 `src/auth.ts`，可能重复修改或忽略之前的修改。

```typescript
function extractFileOperations(
  messages: Message[],
  prevFilesRead?: Set<string>,
  prevFilesModified?: Set<string>
): { filesRead: Set<string>; filesModified: Set<string> } {
  const filesRead = new Set<string>(prevFilesRead || []);
  const filesModified = new Set<string>(prevFilesModified || []);

  for (const msg of messages) {
    if (msg.toolName && msg.content) {
      const pathMatch = msg.content.match(/(?:path|file):\s*"?([^\s"]+)"?/);
      if (pathMatch) {
        const path = pathMatch[1];
        if (msg.toolName === "FileRead") {
          filesRead.add(path);
        } else if (msg.toolName === "FileEdit" || msg.toolName === "FileWrite") {
          filesModified.add(path);
        }
      }
    }
  }

  // modifiedFiles = edited ∪ written
  // readFiles = read - modified（只读未改的）
  for (const f of filesModified) {
    filesRead.delete(f);
  }

  return { filesRead, filesModified };
}
```

`modifiedFiles = edited ∪ written`，`readFiles = read - modified`。为什么这么分？因为改过的文件比只读的更重要，分开追踪让模型知道哪些是"我改过的"vs"我只看过的"。

文件列表跨多次压缩继承。`extractFileOperations` 接受 `prevFilesRead` 和 `prevFilesModified`，从上一次压缩的结果里继承累计。这跟 pi 的 `extractFileOperations`（`compaction.ts:41-69`）一致。

格式化为 XML 标签追加到摘要末尾：

```typescript
function formatFileOperations(filesRead: Set<string>, filesModified: Set<string>): string {
  let result = "";
  if (filesRead.size > 0) {
    result += `\n<read-files>\n${[...filesRead].join("\n")}\n</read-files>`;
  }
  if (filesModified.size > 0) {
    result += `\n<modified-files>\n${[...filesModified].join("\n")}\n</modified-files>`;
  }
  return result;
}
```

## LLM 总结

### 总结提示词

```typescript
const SUMMARIZATION_PROMPT = `You are a context summarization assistant.
Summarize the conversation below into a structured checkpoint.

Output format:
## Goal
## Constraints & Preferences
## Progress (### Done / ### In Progress / ### Blocked)
## Key Decisions
## Next Steps
## Critical Context

Do NOT continue the conversation. ONLY output the structured summary.`;
```

结构跟 pi 的 `SUMMARIZATION_PROMPT`（`compaction.ts:441-472`）一致。关键点：

- **必须有"Next Steps"**。防止压缩后任务漂移（task drift）：模型忘了原来在干什么，开始做别的事。
- **"Do NOT continue the conversation"**。防止模型把总结当成对话继续，开始回答摘要里的问题。
- 系统提示明确"你是总结助手"，让模型进入总结模式而不是对话模式。

### 增量更新

```typescript
const UPDATE_SUMMARIZATION_PROMPT = `You are a context summarization assistant.
Update the existing summary below using the new conversation history.
Preserve all still-relevant information, remove stale details, and merge in new facts.

<existing-summary>
{previousSummary}
</existing-summary>

Output the updated summary in the same format: ...`;
```

如果已有旧摘要，只更新而不是从零生成。为什么？省 token：不用每次把全部历史发过去，只发增量部分。跟 opencode 的锚定摘要和 pi 的 `UPDATE_SUMMARIZATION_PROMPT` 一致。

### 序列化对话

```typescript
function serializeConversation(messages: Message[]): string {
  const MAX_TOOL_RESULT_CHARS = 2000;

  return messages
    .map((msg) => {
      let content = msg.content;
      if (msg.isToolResult && content.length > MAX_TOOL_RESULT_CHARS) {
        content = content.slice(0, MAX_TOOL_RESULT_CHARS) + "\n[truncated]";
      }
      return `[${msg.role}]${msg.toolName ? ` (${msg.toolName})` : ""}: ${content}`;
    })
    .join("\n\n");
}
```

工具结果截断到 2000 字符。为什么？因为总结不需要完整工具输出，关键信息（比如"文件读取成功"、"测试通过"）在前面就够了。跟 opencode 的 `serialize`（`compaction.ts:86-112`）一致。

### 生成摘要

```typescript
async function generateSummary(
  messagesToSummarize: Message[],
  previousSummary?: string
): Promise<string> {
  // 实际实现：
  // const response = await callLLM({
  //   system: previousSummary
  //     ? UPDATE_SUMMARIZATION_PROMPT.replace("{previousSummary}", previousSummary)
  //     : SUMMARIZATION_PROMPT,
  //   messages: [{ role: "user", content: serializeConversation(messagesToSummarize) }],
  //   maxTokens: config.maxSummaryTokens,
  // });
  // return response.content;

  // 模拟输出（实际使用时替换成 LLM 调用）
  // ...
}
```

实际使用时把注释里的 `callLLM` 替换成你的 LLM API 调用。`system` 根据 `previousSummary` 是否存在选择全量还是增量 prompt。`messages` 把序列化后的对话作为 user 消息发送。

## 压缩主逻辑

```typescript
class ContextManager {
  private messages: Message[] = [];
  private lastSummary: string | undefined;
  private filesRead: Set<string> = new Set();
  private filesModified: Set<string> = new Set();
  private overflowRecoveryAttempted = false;

  constructor(private config: CompactionConfig) {}

  async compact(): Promise<CompactionResult> {
    const tokensBefore = estimateTotalTokens(this.messages);

    // 1. 找切点
    const cutIndex = findCutPoint(this.messages, this.config);

    // 2. 拆分 head（旧消息）和 recent（近期消息）
    const messagesToSummarize = this.messages.slice(0, cutIndex);
    const keptMessages = this.messages.slice(cutIndex);

    // 3. 提取文件操作（跨压缩继承）
    const { filesRead, filesModified } = extractFileOperations(
      messagesToSummarize,
      this.filesRead,
      this.filesModified
    );
    this.filesRead = filesRead;
    this.filesModified = filesModified;

    // 4. LLM 总结（增量更新）
    const summary = await generateSummary(messagesToSummarize, this.lastSummary);
    this.lastSummary = summary;

    // 5. 构建新消息列表
    const fileTracking = formatFileOperations(filesRead, filesModified);
    const summaryMessage: Message = {
      role: "user",
      content: `The conversation history before this point was compacted into the following summary:\n\n<summary>\n${summary}${fileTracking}\n</summary>`,
    };

    this.messages = [summaryMessage, ...keptMessages];

    const tokensAfter = estimateTotalTokens(this.messages);

    return { summary, keptMessages, tokensBefore, tokensAfter, filesRead, filesModified };
  }
}
```

压缩后的消息结构：`[摘要消息, ...保留的近期消息]`。

摘要包装成 user role。为什么？因为 assistant role 意味着"模型之前说的话"，模型可能把它当作需要遵循的指令。user role 意味着"这是给模型的信息"，模型更容易当作参考。所有 5 个项目都这么做。

`lastSummary` 保存上一次的摘要，用于增量更新。`filesRead` 和 `filesModified` 保存文件追踪状态，跨多次压缩继承。

## 溢出恢复

```typescript
async recoverFromOverflow(): Promise<boolean> {
  if (this.overflowRecoveryAttempted) {
    console.log("  [overflow] 已经重试过一次，放弃");
    return false;
  }

  this.overflowRecoveryAttempted = true;

  // 删除最后一条消息（通常是 error 消息）
  if (this.messages.length > 0) {
    this.messages.pop();
  }

  console.log("  [overflow] 执行压缩后重试...");
  await this.compact();
  return true;
}

resetOverflowFlag(): void {
  this.overflowRecoveryAttempted = false;
}
```

当 LLM 返回 context overflow 错误时的补救：删掉 error 消息，压缩后重试。

只重试一次。为什么？防止无限压缩循环。如果压缩后仍然 overflow，说明问题不是"历史太长"而是"单条消息太长"或"系统提示+工具定义本身就超限"，再压缩也没用。pi 用 `_overflowRecoveryAttempted` 标志位，opencode 用 defect 抛掷硬性禁止二次恢复。

`resetOverflowFlag` 在新一轮用户消息时调用，因为新消息可能不会 overflow，需要重置标志。

## 运行效果

配置：

```typescript
const config: CompactionConfig = {
  contextWindow: 600,    // 600 token 窗口（实际应该 200K+）
  reserveTokens: 150,    // 剩余 150 token 时触发
  keepRecentTokens: 150, // 保留最近 150 token 的消息
  maxSummaryTokens: 300, // 摘要最多 300 token
};
```

模拟一个修 bug 的对话（读文件 -> 改文件 -> 读另一个文件 -> 改 -> 跑测试），运行结果：

```
[user] | 消息数: 1 | token: 9 (2%)
[assistant] (FileRead) | 消息数: 2 | token: 28 (5%)
[tool] | 消息数: 3 | token: 136 (23%)
[assistant] | 消息数: 4 | token: 148 (25%)
[assistant] (FileEdit) | 消息数: 5 | token: 169 (28%)
[tool] | 消息数: 6 | token: 196 (33%)
[user] | 消息数: 7 | token: 205 (34%)
[assistant] (FileRead) | 消息数: 8 | token: 225 (38%)
[tool] | 消息数: 9 | token: 365 (61%)
[assistant] | 消息数: 10 | token: 380 (63%)
[user] | 消息数: 11 | token: 385 (64%)
[assistant] (FileEdit) | 消息数: 12 | token: 408 (68%)
[tool] | 消息数: 13 | token: 439 (73%)
[user] | 消息数: 14 | token: 441 (74%)
[assistant] (Bash) | 消息数: 15 | token: 453 (76%)
  -> token 超过阈值 (450)，触发压缩...
  ✓ 压缩完成:
    token: 453 -> 166 (63% 降)
    保留消息: 6 条
    读过文件: src/middleware.ts
    改过文件: src/auth.ts
    摘要前 100 字: ## Goal
帮我阅读 src/auth.ts 文件并修复 token 刷新 bug
...

[tool] | 消息数: 8 | token: 280 (47%)
[assistant] | 消息数: 9 | token: 286 (48%)

=== 溢出恢复演示 ===

模拟 LLM 返回 context overflow...
  [overflow] 执行压缩后重试...
恢复结果: 成功

模拟第二次 overflow...
  [overflow] 已经重试过一次，放弃
恢复结果: 失败（只重试一次）

=== 最终状态 ===
消息数: 5
token: 210 (35%)
有摘要: true
读过文件: 0 个
改过文件: 2 个
```

关键观察：

1. **第 15 条消息时触发压缩**：token 453（76%）超过阈值 450
2. **压缩后 453 -> 166**：降了 63%，保留了 6 条近期消息
3. **文件追踪成功**：读过 `src/middleware.ts`，改过 `src/auth.ts`。`src/auth.ts` 既被读又被改，所以只出现在 modified-files 里（`readFiles = read - modified`）
4. **摘要正确提取了用户目标**："帮我阅读 src/auth.ts 文件并修复 token 刷新 bug"
5. **溢出恢复**：第一次成功（压缩后重试），第二次正确拒绝（只重试一次）

## 跟生产级实现的差距

这个最小实现覆盖了核心机制，但跟 Claude Code / Codex / opencode / crush / pi 的生产级实现相比，还差这些：

**Prompt cache 策略**。最小实现没有考虑 cache。生产级实现（特别是 Claude Code）用 fork agent 复用主会话 cache prefix、cache_edits 不破坏前缀删 tool results、sticky-on latch 防 beta header 翻转。如果不考虑 cache，压缩省下的 token 可能不够补 cache miss 的 prefill 成本。

**多层递进**。最小实现是一步到位的 LLM 总结。Claude Code 有三层递进：microCompact（清工具结果，不调 LLM）-> sessionMemoryCompact（滑动窗口，不调 LLM）-> compactConversation（LLM 全量总结）。先试代价低的，不够再升级。

**系统提示词管理**。最小实现没有系统提示词。生产级实现的系统提示词有 section 注册机制（Claude Code 的 cache-safe / DANGERE_uncached）、压缩后清空缓存重算、压缩相关指令告诉模型"你会被压缩"。

**工具链路保证**。最小实现没有工具定义。生产级实现有 delta 机制追踪已发现/已宣布的工具状态（Claude Code 的三套 delta）、每步重建 ToolRouter（Codex）、toolMaterialization（opencode）。

**boundary marker**。最小实现没有 boundary marker。Claude Code 用 boundary marker 记录压缩边界、preservedSegment 链重连、从后向前扫描找最近的 boundary。Codex 用 WorldStateSnapshot baseline 做差分。opencode 用 epoch baseline_seq。

**Hook 机制**。最小实现没有 hook。Claude Code 的 PreCompact hook 让自定义逻辑注入压缩指令。pi 的 `session_before_compact` 可以完全替换压缩逻辑。

**状态保存恢复**。最小实现只追踪了文件操作。Claude Code 有 12+ 项状态保存恢复（readFileState、skills、plan、async agents、deferred tools、system prompt sections 等）。Codex 用 world_state 差分引擎。crush 把 todos 外置到 session 字段。

这些缺失的机制在[原理篇](/posts/code-agent-compaction-原理/)和[源码篇](/posts/code-agent-compaction-源码实现/)都有详细讲解。这个最小实现的价值是：把核心流程跑通，理解每个环节的"为什么"，再去看生产级实现就不会迷失在细节里。

## 完整代码

完整代码在 `G:/ai-project/ai-paper/compaction-demo/context-compaction.ts`，约 300 行，无外部依赖。`npx tsx context-compaction.ts` 直接运行。

实际使用时只需要把 `generateSummary` 里的模拟输出替换成真实的 LLM API 调用，把 `config` 的窗口大小改成实际模型的 context window（比如 200000），就能用在真实场景里了。
