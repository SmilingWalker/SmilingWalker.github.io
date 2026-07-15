---
title: 5 个 Code Agent 上下文压缩的源码实现
published: 2026-07-15
pinned: false
description: ""
tags: [code-agent, context-compaction, claude-code, codex, opencode, crush, pi]
category: AI
draft: false
---

[原理篇](/posts/code-agent-compaction-原理/)讲了上下文压缩的通用设计。这篇逐个拆 5 个项目的源码实现，看它们各自怎么做的。

源码路径：
- Claude Code: `G:/ai-project/claude-code-cli/`
- Codex: `G:/ai-project/codex/`
- opencode: `G:/ai-project/opencode/`
- crush: `G:/ai-project/crush/`
- pi: `G:/ai-project/pi/`

## Claude Code：三层递进压缩

### 核心文件

| 文件 | 作用 |
|---|---|
| `services/compact/autoCompact.ts` | 自动压缩触发条件 |
| `services/compact/compact.ts` | 主压缩算法（LLM 总结） |
| `services/compact/microCompact.ts` | 微压缩（清旧工具结果） |
| `services/compact/sessionMemoryCompact.ts` | 滑动窗口压缩 |
| `services/compact/prompt.ts` | 总结提示词模板 |
| `services/compact/grouping.ts` | 按 API 轮次分组 |
| `utils/context.ts` | 上下文窗口大小 |

### 触发条件

三层递进，全部基于 token 数。

**第一层：microCompact**（每次请求前运行）

不调 LLM，只清理旧工具结果。触发方式有两种：

```typescript
// microCompact.ts:422-444 - 基于时间
// 距上一条 assistant 消息超过 gapThresholdMinutes（cache 已过期）
// 清理不增加成本，因为 cache 已经冷了

// microCompact.ts:305 - 基于 cache_edits
// 用 cache_edits API 在不破坏 prompt cache 前缀的前提下删除旧工具结果
```

只清理这些工具的结果（`microCompact.ts:41-50`）：`FileRead`, `Bash/shell`, `Grep`, `Glob`, `WebSearch`, `WebFetch`, `FileEdit`, `FileWrite`。保留最近 N 个，其余替换成 `'[Old tool result content cleared]'`。

**第二层：sessionMemoryCompact**（token 超阈值时先尝试这个）

滑动窗口，保留尾部消息，丢弃前面的。配置（`sessionMemoryCompact.ts:57-61`）：

```typescript
minTokens = 10_000        // 最少保留这么多
minTextBlockMessages = 5  // 最少保留这么多条文本消息
maxTokens = 40_000        // 最多保留这么多
```

从 `lastSummarizedIndex+1` 开始向后（更早）扩展，直到同时满足 minTokens 和 minTextBlockMessages，或达到 maxTokens。不能跨越最后一个 compact boundary。

**第三层：compactConversation**（sessionMemory 不够才用这个）

```typescript
// autoCompact.ts:62-76
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000

export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  // effectiveContextWindow = contextWindow - min(maxOutputTokens, 20_000)
  return effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
}
```

`shouldAutoCompact`（`autoCompact.ts:160-239`）判断 `tokenCount >= autoCompactThreshold` 时触发。有熔断器：连续失败 3 次（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`）后停止重试。

### 压缩算法

`compactConversation`（`compact.ts:387-763`）的核心设计是 **fork agent 复用 prompt cache**：

```typescript
// compact.ts:1188-1200
const result = await runForkedAgent({
  promptMessages: [summaryRequest],
  cacheSafeParams,
  canUseTool: createCompactCanUseTool(),  // 所有工具 deny
  querySource: 'compact',
  forkLabel: 'compact',
  maxTurns: 1,
  skipCacheWrite: true,
})
```

为什么要 fork？因为总结请求需要把整个对话历史发过去，如果用新会话发，prompt cache 全部 miss。通过 fork 复用主会话的 cache prefix（系统提示、工具定义、上下文消息前缀），只追加一条总结请求。

fork 失败则回退到普通流式路径（`streamCompactSummary`，`compact.ts:1136`），系统提示换成 `"You are a helpful AI assistant tasked with summarizing conversations."`。

### 总结提示词

Claude Code 的总结 prompt 是 5 个项目里最详细的（`prompt.ts:293`），要求生成 9 个结构化小节：

1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections（含完整代码片段）
4. Errors and fixes
5. Problem Solving
6. All user messages（逐条列出所有非工具结果的用户消息）
7. Pending Tasks
8. Current Work
9. Optional Next Step（含原文引用以防任务漂移）

提示词强制模型先写 `<analysis>` 草稿块再写 `<summary>` 块。`formatCompactSummary`（`prompt.ts:311`）最后把 `<analysis>` 块剥离，只留 `<summary>`。

### 保留策略

压缩后的消息结构（`compact.ts:299-310`）：

```
[boundaryMarker, ...summaryMessages, ...(messagesToKeep), ...attachments, ...hookResults]
```

**重新注入的内容**（`compact.ts:532-585`）：

- 最近读取的文件：最多 5 个，每文件 5K token，总预算 50K（`createPostCompactFileAttachments`，L122-129）
- Plan 文件、plan mode 状态
- 已加载的 skills：每个截断到 5K token，总预算 25K（L1494-1534）
- MCP instructions、deferred tools delta

**丢弃**：所有被总结的旧消息、`readFileState` 缓存、图片（替换为 `[image]` 文本标记）。

### 独特设计：PTL 重试

如果总结请求本身超长（prompt-too-long），Claude Code 按 API 轮次分组丢弃最旧的组（`grouping.ts`）：

```typescript
// compact.ts:260-275
const tokenGap = getPromptTooLongTokenGap(ptlResponse)
// 累加各组 token 直到覆盖 gap
dropCount = Math.min(dropCount, groups.length - 1)  // 至少留一组
```

按 token gap 或 20% 比例计算丢多少，最多重试 3 次。

## Codex：三路分派

### 核心文件

| 文件 | 作用 |
|---|---|
| `codex-rs/core/src/compact.rs` | 本地 LLM 总结 |
| `codex-rs/core/src/compact_token_budget.rs` | Token 预算压缩（直接重置） |
| `codex-rs/core/src/compact_remote.rs` | 服务端压缩 v1 |
| `codex-rs/core/src/compact_remote_v2.rs` | 服务端压缩 v2 |
| `codex-rs/core/src/session/turn.rs` | 触发分派逻辑 |
| `codex-rs/core/src/session/context_window.rs` | token 阈值状态 |
| `codex-rs/prompts/templates/compact/prompt.md` | 总结提示词 |

### 触发条件

两个触发点（`turn.rs`）：

**Pre-turn**（轮次开始前，`turn.rs:800-825`）：

```rust
let token_status = context_window_token_status(sess, turn_context).await;
if token_status.token_limit_reached {
    run_auto_compact(..., CompactionReason::ContextLimit, CompactionPhase::PreTurn).await?
}
```

**Mid-turn**（轮次中间，工具继续需要时，`turn.rs:348-372`）：

```rust
if needs_follow_up && (sess.take_new_context_window_request().await || token_limit_reached) {
    run_auto_compact(..., InitialContextInjection::BeforeLastUserMessage(world_state),
                     CompactionReason::ContextLimit, CompactionPhase::MidTurn).await
}
```

还有两种特殊触发（`turn.rs:861-949`）：
- `CompHashChanged`：comp_hash 变化
- `ModelDownshift`：切换到更小上下文窗口的模型

阈值默认 90%（`openai_models.rs:457-468`），可被用户配置 `model_auto_compact_token_limit` 覆盖。

### 三路分派

`run_auto_compact`（`turn.rs:956-1032`）根据 feature flag 和 provider 分派：

```rust
async fn run_auto_compact(...) -> CodexResult<()> {
    if turn_context.config.features.enabled(Feature::TokenBudget) {
        // 路径 1：直接重置，不调 LLM
        crate::compact_token_budget::run_inline_auto_compact_task(...).await?;
        return Ok(());
    }
    if should_use_remote_compact_task(turn_context.provider.info()) {
        if features.enabled(Feature::RemoteCompactionV2) {
            // 路径 2a：服务端压缩 v2
            run_inline_remote_auto_compact_task_v2(...).await?
        } else {
            // 路径 2b：服务端压缩 v1
            run_inline_remote_auto_compact_task(...).await?
        }
    } else {
        // 路径 3：本地 LLM 总结
        run_inline_auto_compact_task(...).await?
    }
}
```

### 路径 1：TokenBudget（直接重置）

```rust
// compact_token_budget.rs:45-90
// "Token-budget compaction skips model/server summarization
//  and installs a fresh context window instead."
async fn run_compact_task_inner(...) -> CodexResult<()> {
    sess.start_new_context_window(turn_context, world_state).await;
    Ok(())
}
```

完全跳过 LLM 总结，靠 `world_state` 重建上下文。最激进，相当于"全忘了，从头来"。

### 路径 3：本地 LLM 总结

`compact.rs:92-378` 的流程：

1. 把总结 prompt 追加到当前历史末尾（不剥离旧消息）
2. 调模型流式生成
3. 取最后一条 assistant 消息作为总结
4. 加 `SUMMARY_PREFIX` 前缀
5. 收集所有 user 消息
6. 用 `build_compacted_history` 构造新历史

总结 prompt（`prompts/templates/compact/prompt.md`）相对简洁：

```
You are performing a CONTEXT CHECKPOINT COMPACTION.
Create a handoff summary for another LLM that will resume the task.
Include:
- Current progress and key decisions made
- Important context, constraints, or user preferences
- What remains to be done (clear next steps)
- Any critical data, examples, or references needed to continue
```

### 保留策略

本地总结路径保留近期 user 消息（`compact.rs:599-660`）：

```rust
const COMPACT_USER_MESSAGE_MAX_TOKENS: usize = 20_000;
// 从 user_messages 末尾向前遍历，累加 token 直到用尽 20K 预算
// 超预算的最后一条会被 truncate_text 截断
```

保留：近期 user 消息（最多 20K token）+ 总结消息。

丢弃：所有 assistant 消息、所有工具调用/结果（`FunctionCall`、`FunctionCallOutput`、`Reasoning` 在 `should_keep_compacted_history_item` 中返回 false）。

### 独特设计：mid-turn vs pre-turn 注入语义

```rust
// compact.rs:539-584
// BeforeLastUserMessage（mid-turn）：
//   在最后一个真实 user 消息之前插入 canonical initial context
//   因为模型训练时期望 mid-turn 压缩后总结是历史最后一项

// DoNotInject（pre-turn/manual）：
//   不注入，清空 reference_context_item
//   让下一轮正常重新注入 initial context
```

mid-turn 压缩发生在工具调用中间，模型需要继续执行未完成的任务，所以要在最后 user 消息前注入 world state。pre-turn 压缩发生在新一轮开始前，下一轮会正常注入，不需要额外处理。

## opencode：锚定摘要

### 核心文件

| 文件 | 作用 |
|---|---|
| `packages/core/src/session/compaction.ts` | 压缩核心逻辑 |
| `packages/core/src/config/compaction.ts` | 配置模式定义 |
| `packages/core/src/session/runner/llm.ts` | 运行循环，触发压缩 |
| `packages/core/src/session/history.ts` | 历史加载（过滤旧消息） |
| `packages/core/src/session/runner/to-llm-message.ts` | 压缩消息渲染 |

### 触发条件

两种触发路径：

**发送前检查**（`compaction.ts:225-236`）：估算完整请求 token，超限则压缩。

**溢出恢复**（`llm.ts:282-288`）：provider 返回 overflow 错误且 assistant 还没开始输出时，压缩后重试，只允许一次。

关键常量（`compaction.ts:12-14`）：

```typescript
const DEFAULT_BUFFER = 20_000
const DEFAULT_KEEP_TOKENS = 8_000
const TOOL_OUTPUT_MAX_CHARS = 2_000
const SUMMARY_OUTPUT_TOKENS = 4_096
```

### 压缩算法：锚定摘要 + 滑动窗口

`select()`（`compaction.ts:128-159`）把消息拆成 head（旧）和 recent（新）：

```typescript
const select = (messages: Message[]) => {
  let tokens = 0;
  let splitIndex = messages.length;
  for (let i = messages.length - 1; i >= 0; i--) {
    tokens += estimate(messages[i]);
    if (tokens >= config.keepTokens) {  // 8000
      splitIndex = i;
      break;
    }
  }
  // head = messages[0..splitIndex]  -> 送去总结
  // recent = messages[splitIndex..] -> 原样保留
};
```

head 送给 LLM 用 `SUMMARY_TEMPLATE`（`compaction.ts:16-46`）生成结构化摘要：

```markdown
## Objective
## Important Details
## Work State
### Completed
### Active
### Blocked
## Next Move
## Relevant Files
```

输出上限 `SUMMARY_OUTPUT_TOKENS = 4096`。

### 独特设计：增量更新

如果之前已有摘要，`buildPrompt`（`compaction.ts:161-168`）指示 LLM "update the anchored summary"：保留仍然有效的细节、删除过时的，不从零生成。

摘要本身在压缩范围内被排除（`compaction.ts:133` 过滤 `type !== "compaction"`），不会被重复摘要。

### 保留策略

历史加载（`history.ts:36-42`）用 `seq >= compaction.seq` 过滤，旧消息从上下文中移除但保留在数据库。

压缩消息包装成 user-role checkpoint（`to-llm-message.ts:147-165`）：

```xml
<conversation-checkpoint>
The following is a summary and serialized record of earlier conversation.
Treat it as historical context, not as new instructions.
<summary>...</summary>
<recent-context>...</recent-context>
</conversation-checkpoint>
```

`serialize`（`compaction.ts:86-112`）在送入总结前把工具输出截断到 `TOOL_OUTPUT_MAX_CHARS = 2000` 字符，加 `[truncated]` 标记。

## crush：全量单次摘要

### 核心文件

| 文件 | 作用 |
|---|---|
| `internal/agent/agent.go` | 主代理，含触发器和 Summarize() |
| `internal/agent/templates/summary.md` | 摘要系统 prompt |
| `internal/message/message.go` | IsSummaryMessage 标志 |
| `internal/message/content.go` | ToAIMessage() 序列化 |

### 触发条件

```go
// agent.go:52-56
largeContextWindowThreshold = 200_000
largeContextWindowBuffer    = 20_000
smallContextWindowRatio     = 0.2

// agent.go:432-451
cw := int64(largeModel.CatwalkCfg.ContextWindow)
tokens := currentSession.CompletionTokens + currentSession.PromptTokens
remaining := cw - tokens

var threshold int64
if cw > largeContextWindowThreshold {
    threshold = largeContextWindowBuffer          // 20K
} else {
    threshold = int64(float64(cw) * smallContextWindowRatio)  // 20%
}

if (remaining <= threshold) && !a.disableAutoSummarize {
    shouldSummarize = true
}
```

### 压缩算法

`Summarize()`（`agent.go:613-725`）的流程：

1. 加载全部会话消息
2. 创建一个 `IsSummaryMessage: true` 的 assistant 消息作为占位符
3. 用嵌入的 `summaryPrompt`（`templates/summary.md`）作为系统 prompt 建新 agent
4. 以完整消息历史作为输入流式生成摘要
5. 把摘要持久化，设 `currentSession.SummaryMessageID`

```go
// agent.go:1196-1209
func buildSummaryPrompt(todos []session.Todo) string {
    var sb strings.Builder
    sb.WriteString("Provide a detailed summary of our conversation above.")
    if len(todos) > 0 {
        sb.WriteString("\n\n## Current Todo List\n\n")
        for _, t := range todos {
            fmt.Fprintf(&sb, "- [%s] %s\n", t.Status, t.Content)
        }
        sb.WriteString("\nInclude these tasks and their statuses in your summary. ")
        sb.WriteString("Instruct the resuming assistant to use the `todos` tool to continue tracking progress.")
    }
    return sb.String()
}
```

### 独特设计：无近期窗口 + role 重映射

crush 是 5 个项目里唯一不保留近期消息的。摘要就是唯一的上下文。模板明确说："此摘要是恢复对话时唯一可用的上下文。假设之前所有消息都将丢失。"

摘要后的截断（`agent.go:805-817`）：

```go
if session.SummaryMessageID != "" {
    summaryMsgIndex := -1
    for i, msg := range msgs {
        if msg.ID == session.SummaryMessageID {
            summaryMsgIndex = i
            break
        }
    }
    if summaryMsgIndex != -1 {
        msgs = msgs[summaryMsgIndex:]  // 截断摘要前的所有消息
        msgs[0].Role = message.User    // role 从 assistant 重映射为 user
    }
}
```

摘要的 role 被改成 `User`，这样喂给 LLM 时表现为一条 user 消息。

如果压缩时 agent 正在执行工具调用，原始 prompt 会被重新排队并加前缀（`agent.go:593`）：

```go
call.Prompt = fmt.Sprintf(
    "The previous session was interrupted because it got too long, the initial user request was: `%s`",
    call.Prompt)
```

## pi：结构化 checkpoint

### 核心文件

| 文件 | 作用 |
|---|---|
| `packages/coding-agent/src/core/compaction/compaction.ts` | 压缩核心逻辑 |
| `packages/coding-agent/src/core/compaction/utils.ts` | 序列化、文件追踪、系统 prompt |
| `packages/coding-agent/src/core/agent-session.ts` | 触发逻辑 |
| `packages/coding-agent/src/core/session-manager.ts` | 上下文重建 |
| `packages/coding-agent/src/core/messages.ts` | 消息类型和转换 |

### 触发条件

三种触发路径（`agent-session.ts`）：

```typescript
// 1. 阈值触发 (agent-session.ts:1985)
if (shouldCompact(contextTokens, contextWindow, settings)) {
    return await this._runAutoCompaction("threshold", false);
}

// 2. 溢出恢复 (agent-session.ts:1930)
if (sameModel && isContextOverflow(assistantMessage, contextWindow)) {
    if (this._overflowRecoveryAttempted) return false;
    this._overflowRecoveryAttempted = true;
    return await this._runAutoCompaction("overflow", willRetry);
}

// 3. 手动 (agent-session.ts:1736)
// 由 /compact 命令或扩展触发
```

默认配置（`compaction.ts:106-110`）：

```typescript
export const DEFAULT_COMPACTION_SETTINGS: CompactionSettings = {
    enabled: true,
    reserveTokens: 16384,     // 触发阈值 buffer
    keepRecentTokens: 20000,  // 保留近期消息的 token 预算
};
```

### 压缩算法

**步骤 A：找切点**（`compaction.ts:377-413`）

从最新消息向前累加 token，达到 `keepRecentTokens`（20000）时找最近的合法切点：

```typescript
for (let i = endIndex - 1; i >= startIndex; i--) {
    accumulatedTokens += messageTokens;
    if (accumulatedTokens >= keepRecentTokens) {
        // 找 >= i 的最近合法切点
        break;
    }
}
```

切点合法性（`isCutPointMessage`，`compaction.ts:282`）：`user`、`assistant`、`bashExecution`、`custom` 可以切，**`toolResult` 不能切**。

Token 估算用 `chars/4` 启发式（`estimateTokens`，`compaction.ts:240`），图片按 4800 字符。

**步骤 B：LLM 总结**（`compaction.ts:546`）

旧消息通过 `serializeConversation`（`utils.ts:109`）序列化成纯文本，包装进 `<conversation>` 标签，用结构化 prompt 生成总结：

```markdown
## Goal
## Constraints & Preferences
## Progress (### Done / ### In Progress / ### Blocked)
## Key Decisions
## Next Steps
## Critical Context
```

系统 prompt（`utils.ts:168`）明确禁止续写对话："Do NOT continue the conversation. ONLY output the structured summary."

### 独特设计 1：增量更新

如果已有旧摘要，改用 `UPDATE_SUMMARIZATION_PROMPT`（`compaction.ts:474`），从最近的 `CompactionEntry` 取出 `previousSummary` 作为基础，要求"保留所有既有信息、移动进度项、更新下一步"。

### 独特设计 2：split-turn 处理

如果切点落在一个 turn 中间，`findCutPoint` 返回 `isSplitTurn: true`。此时额外用 `TURN_PREFIX_SUMMARIZATION_PROMPT`（`compaction.ts:718`）总结这个 turn 的前缀，两段总结拼接：

```typescript
summary = `${historyResult}\n\n---\n\n**Turn Context (split turn):**\n\n${turnPrefixResult}`;
```

保留下来的 turn 后缀仍有上下文可依。

### 独特设计 3：文件操作追踪

`extractFileOperations`（`compaction.ts:41`）从被丢弃消息的 tool call 里提取 read/write/edit 的文件路径，以 XML 标签追加到摘要末尾：

```xml
<read-files>
src/auth/refresh.ts
src/auth/middleware.ts
</read-files>
<modified-files>
src/auth/refresh.ts
</modified-files>
```

模型即使丢失了历史细节，仍知道哪些文件被读过/改过。文件列表从上一次压缩的 `details` 里继承累计。

### 独特设计 4：扩展钩子

```typescript
// extensions/types.ts:578-588
export interface SessionBeforeCompactEvent {
    type: "session_before_compact";
    preparation: CompactionPreparation;
    reason: "manual" | "threshold" | "overflow";
    willRetry: boolean;
    signal: AbortSignal;
}
```

扩展可以返回 `{ cancel: true }` 取消压缩，或返回 `{ compaction: CompactionResult }` 完全替换默认总结逻辑。示例（`custom-compaction.ts`）用 Gemini Flash 替代默认模型做总结。

## 横向对比

| 维度 | Claude Code | Codex | opencode | crush | pi |
|---|---|---|---|---|---|
| 语言 | TypeScript | Rust | TypeScript | Go | TypeScript |
| 触发阈值 | 窗口-13K (~93%) | 90% | 窗口-max(output,20K) | 剩余<20K / <20% | 窗口-16K (~92%) |
| 压缩层次 | 三层递进 | 三路分派 | 单层 | 单层 | 单层 |
| 主算法 | LLM 总结 (fork agent) | LLM / 服务端 / 重置 | LLM 锚定摘要 | LLM 全量摘要 | LLM 结构化摘要 |
| 滑动窗口 | sessionMemory 10-40K | 20K user 消息 | 8K recent | 无 | 20K recent |
| 增量更新 | 否 | 否 | 是 | 否 | 是 |
| 溢出恢复 | PTL 重试 3次 | remove_first_item | 1次 | 无 | 1次 |
| 文件追踪 | 重新注入5个文件 | world_state | 无 | 无 | XML 标签 |
| Todo 处理 | 无 | 无 | 无 | 注入 prompt | 无 |
| 扩展钩子 | PreCompact hooks | pre_compact hooks | 无 | 无 | session_before_compact |
| 熔断 | 连续失败3次 | 无 | 无 | 无 | 无 |
| 摘要 role | user (continued session) | user (SUMMARY_PREFIX) | user (checkpoint) | user (重映射) | user (summary 标签) |

## 设计模式总结

从源码里能归纳出几个共性设计决策和各自的取舍：

**1. 阈值设计：buffer 大小是核心权衡。** buffer 太小，压缩后没几轮又满了；buffer 太大，压缩太早浪费可用上下文。5 个项目的 buffer 在 8K-20K 之间。Claude Code 13K，Codex 10%（约 20K），pi 16K，opencode 20K，crush 20K。

**2. 切点选择：tool result 不能切。** 5 个项目都保证 tool call 和 tool result 成对保留。pi 显式检查 `isCutPointMessage`，其他项目隐式处理。

**3. 摘要里必须有"下一步"。** 所有项目的摘要 prompt 都要求写明当前进度和下一步，防止压缩后任务漂移。Claude Code 的第 9 节 "Optional Next Step（含原文引用）" 最严格。

**4. 系统提示和工具定义不压缩。** 压缩只作用于对话历史。系统提示、工具定义在压缩范围外始终完整保留。

**5. 近期消息保留量是权衡。** crush 不保留（最激进，上下文最薄），opencode 8K（最小），pi 和 Codex 20K，Claude Code 10-40K（最灵活）。保留越多上下文越准确，但压缩效果越小。

**6. 增量更新 vs 全量总结。** opencode 和 pi 支持增量更新旧摘要，token 消耗更少但可能累积信息损失。Claude Code、Codex、crush 每次全量总结，更准确但更贵。

**7. Prompt cache 是压缩设计的重要考量。** Claude Code 的 fork agent、microCompact 的 cache_edits、Codex 的 `remove_first_item` 保留 cache prefix，都是为了在压缩时尽量不破坏 prompt cache。
