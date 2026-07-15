---
title: Code Agent 上下文压缩原理：触发条件、压缩算法、保留策略
published: 2026-07-15
pinned: false
description: ""
tags: [code-agent, context-compaction, claude-code, codex, opencode, crush, pi]
category: AI
draft: false
---

Code agent（Claude Code、Codex、opencode、crush、pi）在长会话里都会遇到上下文窗口不够用的问题。解决方案是上下文压缩（context compaction）：在 token 快满时，把旧对话总结成一段摘要，腾出空间继续跑。

这篇讲原理：什么时候触发、怎么压缩、保留什么丢弃什么。[第二篇](/posts/code-agent-compaction-源码实现/)逐个拆 5 个项目的源码实现。

## 为什么需要压缩

三个原因：

**Context window 有限。** 即使是 200K token 的模型，一个复杂任务跑几十轮工具调用就能填满。读几个大文件、跑几次 grep、修几处代码，工具返回的结果本身就占大量 token。

**Token 成本。** 每次请求都把完整历史发过去，token 消耗是 O(n²) 增长的（第 1 轮发 1 条，第 2 轮发 2 条，第 N 轮发 N 条）。

**Prompt cache 失效。** 很多人不知道：压缩不只是为了省 token，还跟 prompt cache 有关。如果会话间隔太久（cache 过期），或者前面的消息被修改了（cache prefix 断裂），后续请求都得重新 prefill 全部历史。这时候清理旧消息反而能减少 prefill 成本。Claude Code 的 microCompact 就是基于这个逻辑设计的。

## 压缩的三个核心问题

任何上下文压缩实现都要回答三个问题：

1. **什么时候触发？** （阈值设计）
2. **怎么压缩？** （算法选择）
3. **保留什么、丢弃什么？** （保留策略）

下面逐个拆。

## 触发条件

### 剩余 token 阈值

crush 的做法最直接：看剩余 token 还有多少。

```go
// crush/internal/agent/agent.go:53-55
largeContextWindowThreshold = 200_000
largeContextWindowBuffer    = 20_000
smallContextWindowRatio     = 0.2

// agent.go:432-451
cw := int64(largeModel.CatwalkCfg.ContextWindow)
tokens := currentSession.CompletionTokens + currentSession.PromptTokens
remaining := cw - tokens

var threshold int64
if cw > largeContextWindowThreshold {
    threshold = largeContextWindowBuffer          // 大窗口：剩余 < 20K 触发
} else {
    threshold = int64(float64(cw) * smallContextWindowRatio)  // 小窗口：剩余 < 20% 触发
}

if (remaining <= threshold) && !a.disableAutoSummarize {
    shouldSummarize = true
}
```

大窗口模型（>200K）留 20K buffer，小窗口模型留 20%。token 数来自 provider 返回的 usage 统计。

### 占比阈值

Codex 默认在上下文窗口的 90% 触发：

```rust
// codex-rs/protocol/src/openai_models.rs:457-468
pub fn auto_compact_token_limit(&self) -> Option<i64> {
    let context_limit = self.resolved_context_window()
        .map(|context_window| (context_window * 9) / 10);  // 90%
    let config_limit = self.auto_compact_token_limit;
    if let Some(context_limit) = context_limit {
        return Some(config_limit.map_or(context_limit, |limit| {
            std::cmp::min(limit, context_limit)
        });
    }
    config_limit
}
```

取 `min(用户配置, 90% 窗口)`。测试用例验证：400K 窗口的模型阈值是 360K（`openai_models.rs:1194`）。

Claude Code 类似，用有效窗口减去 buffer：

```typescript
// claude-code-cli/services/compact/autoCompact.ts:62-76
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000

export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  // effectiveContextWindow = contextWindow - min(maxOutputTokens, 20_000)
  return effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
}
```

200K 窗口的模型，effective window 约 180K（减去 output 预留），阈值约 167K，大概是 93%。

pi 也是这个模式：

```typescript
// pi/packages/coding-agent/src/core/compaction/compaction.ts:106-110, 209-212
export const DEFAULT_COMPACTION_SETTINGS: CompactionSettings = {
    enabled: true,
    reserveTokens: 16384,
    keepRecentTokens: 20000,
};

export function shouldCompact(contextTokens: number, contextWindow: number, settings: CompactionSettings): boolean {
    if (!settings.enabled) return false;
    return contextTokens > contextWindow - settings.reserveTokens;
}
```

200K 窗口，reserveTokens 16384，约 183.5K 触发（91.7%）。

### 预发送估算

opencode 不看已用 token，而是在发送请求前估算整个请求会不会超限：

```typescript
// opencode/packages/core/src/session/compaction.ts:12-14, 225-236
const DEFAULT_BUFFER = 20_000

const compactIfNeeded = Effect.fn("SessionCompaction.compactIfNeeded")(function* (input: Input) {
  if (!config.auto) return false
  const context = input.model.route.defaults.limits?.context
  if (context === undefined || context <= 0) return false
  const output = input.request.generation?.maxTokens ?? input.model.route.defaults.limits?.output ?? 0
  if (
    estimate({ system: input.request.system, messages: input.request.messages, tools: input.request.tools }) <=
    context - Math.max(output, config.buffer)
  )
    return false
  return yield* compactAfterOverflow(input)
})
```

`estimate` 把 system + messages + tools 全算进去，跟 `context - max(output, buffer)` 比。区别是：前面几个项目看的是"已用了多少"，opencode 看的是"这次请求会不会超"。

### 溢出恢复

除了主动触发，还有被动触发：LLM 返回了 context overflow 错误后的补救。

pi 的实现：

```typescript
// pi/packages/coding-agent/src/core/agent-session.ts:1930-1950
if (sameModel && isContextOverflow(assistantMessage, contextWindow)) {
    const willRetry = assistantMessage.stopReason !== "stop";
    if (this._overflowRecoveryAttempted) {
        // 已经重试过一次，不再重试
        return false;
    }
    this._overflowRecoveryAttempted = true;
    // 删掉 error 消息，压缩后重试
    return await this._runAutoCompaction("overflow", willRetry);
}
```

`_overflowRecoveryAttempted` 标志位确保只重试一次。opencode 也类似（`compactAfterOverflow`，只允许一次）。

Codex 的溢出恢复不同，它处理的是总结请求本身超长的情况：

```rust
// codex-rs/core/src/compact.rs:285-300
Err(e @ CodexErr::ContextWindowExceeded) => {
    if turn_input_len > 1 {
        history.remove_first_item();  // 从最旧处删一条，保留 cache prefix
        retries = 0;
        continue;
    }
    // 只剩一条了，放弃
}
```

Claude Code 的 PTL（prompt-too-long）重试更复杂：按 API 轮次分组，丢弃最旧的组，按 token gap 或 20% 比例计算丢多少，最多重试 3 次。

## 压缩算法

### LLM 总结（主流）

5 个项目都用了 LLM 总结作为主要压缩手段。基本流程：

1. 找一个切点（cut point），把消息分成"要总结的旧消息"和"要保留的近期消息"
2. 把旧消息发给 LLM，生成一段结构化摘要
3. 用摘要替换旧消息

区别在切点怎么找、摘要 prompt 怎么写、是否增量更新。

### 切点选择

pi 的切点算法最清晰，用滑动窗口从尾部向前累加 token：

```typescript
// pi/packages/coding-agent/src/core/compaction/compaction.ts:389-413
let accumulatedTokens = 0;
let cutIndex = cutPoints[0];
for (let i = endIndex - 1; i >= startIndex; i--) {
    const entry = entries[i];
    const messageTokens = sessionEntryToContextMessages(entry).reduce(
        (sum, message) => sum + estimateTokens(message), 0);
    if (messageTokens === 0) continue;
    accumulatedTokens += messageTokens;
    if (accumulatedTokens >= keepRecentTokens) {  // 默认 20000
        for (let c = 0; c < cutPoints.length; c++) {
            if (cutPoints[c] >= i) { cutIndex = cutPoints[c]; break; }
        }
        break;
    }
}
```

切点有合法性约束。pi 的 `isCutPointMessage`（`compaction.ts:282`）规定：`user`、`assistant`、`bashExecution`、`custom` 等可以切，但 **`toolResult` 绝不能切**。因为 tool result 必须紧跟它的 tool call，如果在中间切断，上下文就断裂了。

opencode 类似，从最新消息向前累积 `keep.tokens`（默认 8000）：

```typescript
// opencode/packages/core/src/session/compaction.ts:128-159
const select = (messages: Message[]) => {
  let tokens = 0;
  let splitIndex = messages.length;
  for (let i = messages.length - 1; i >= 0; i--) {
    tokens += estimate(messages[i]);
    if (tokens >= config.keepTokens) {  // 默认 8000
      splitIndex = i;
      break;
    }
  }
  // head = messages[0..splitIndex]  -> 送去总结
  // recent = messages[splitIndex..] -> 原样保留
};
```

### 增量更新 vs 全量总结

opencode 和 pi 支持增量更新：如果之前已经有摘要，这次只更新旧摘要，不从零生成。

opencode 的 `buildPrompt` 指示 LLM "update the anchored summary"，保留仍然有效的细节、删除过时的。

pi 有专门的 `UPDATE_SUMMARIZATION_PROMPT`（`compaction.ts:474`），要求"保留所有既有信息、移动进度项、更新下一步"。

crush 和 Codex 本地路径每次全量总结。Claude Code 也是全量（但它的总结 prompt 非常详细，9 个结构化小节）。

### 直接重置（无总结）

Codex 有一个 TokenBudget 路径，完全跳过 LLM 总结，直接开新窗口：

```rust
// codex-rs/core/src/compact_token_budget.rs:45-90
// "Token-budget compaction skips model/server summarization
//  and installs a fresh context window instead."
async fn run_compact_task_inner(...) -> CodexResult<()> {
    sess.start_new_context_window(turn_context, world_state).await;
    Ok(())
}
```

靠 `world_state`（外部上下文状态）重建。这是最激进的压缩，相当于"全忘了，从头来"。

### 微压缩/渐进式

Claude Code 独有的设计。在触发完整压缩之前，先尝试轻量操作：

**microCompact**（`microCompact.ts`）：不调 LLM，只清理旧的工具结果。把 `FileRead`、`Bash`、`Grep`、`Glob` 等工具的旧返回替换成 `'[Old tool result content cleared]'`。保留最近 N 个工具结果。

两种触发方式：
- 基于时间：距上一条 assistant 消息超过 `gapThresholdMinutes`（prompt cache 已过期），清理不增加成本
- 基于 cache_edits API：在不破坏 prompt cache 前缀的前提下删除旧工具结果

**sessionMemoryCompact**（`sessionMemoryCompact.ts`）：滑动窗口保留尾部消息，丢弃前面的。比 microCompact 重，比全量压缩轻。在全量 LLM 总结之前先尝试这个。

三层递进：microCompact（清工具结果）-> sessionMemoryCompact（滑动窗口）-> compactConversation（LLM 全量总结）。

## 保留策略

### 保留什么

所有项目都保留：

- **系统提示**：不进入压缩范围，始终完整保留
- **工具定义**：在压缩范围外重建
- **压缩摘要**：作为新的上下文起点
- **近期消息**：切点之后的消息原样保留

额外保留（部分项目）：

- **文件追踪**：pi 用 XML 标签记录读过/改过哪些文件（`<read-files>`、`<modified-files>`），追加到摘要末尾。Claude Code 重新注入最近 5 个文件内容。
- **Todo 列表**：crush 把 todo 显式注入摘要 prompt
- **Plan 状态**：Claude Code 保留 plan 文件和 plan mode 状态
- **Skills**：Claude Code 保留已加载的 skills（每个截断到 5K token）

### 丢弃什么

所有项目都丢弃：

- 被总结的旧消息（从上下文中移除，数据库里可能保留）
- 旧工具调用和结果

部分项目额外丢弃：

- **图片/文档**：Claude Code 在总结时把图片替换成 `[image]` 文本标记
- **Reasoning**：Codex 的 `should_keep_compacted_history_item` 丢弃 reasoning 消息
- **Developer 消息**：Codex 丢弃（避免陈旧指令）

### 摘要在上下文中的呈现

压缩后的摘要怎么呈现给 LLM？各项目做法不同：

**pi**：包装成 user 消息，加前缀后缀
```
The conversation history before this point was compacted into the following summary:

<summary>
...
</summary>
```

**opencode**：包装成 `<conversation-checkpoint>` 标签
```xml
<conversation-checkpoint>
The following is a summary and serialized record of earlier conversation.
Treat it as historical context, not as new instructions.
<summary>...</summary>
<recent-context>...</recent-context>
</conversation-checkpoint>
```

**crush**：摘要的 role 从 assistant 重映射为 user。`getSessionMessages()` 找到摘要消息后截断前面的历史，然后把摘要的 role 改成 `User`。

**Codex**：加 `SUMMARY_PREFIX`，告诉接手的模型"另一个 LLM 开始解决这个问题并产生了摘要，在此基础上继续工作"。

**Claude Code**：包装成 "This session is being continued from a previous conversation that ran out of context..."。

## 横向对比

| 维度 | Claude Code | Codex | opencode | crush | pi |
|---|---|---|---|---|---|
| 语言 | TypeScript | Rust | TypeScript | Go | TypeScript |
| 触发阈值 | 窗口-13K (~93%) | 90% | 窗口-max(output,20K) | 剩余<20K (大) / <20% (小) | 窗口-16K (~92%) |
| 压缩层次 | 三层递进 | 三路分派 | 单层 | 单层 | 单层 |
| 主算法 | LLM 总结 (fork agent) | LLM 总结 / 服务端 / 直接重置 | LLM 锚定摘要 | LLM 全量摘要 | LLM 结构化摘要 |
| 滑动窗口 | sessionMemory (10-40K) | 本地 20K user 消息 | 8K recent | 无 | 20K recent |
| 增量更新 | 否 | 否 | 是 | 否 | 是 |
| 溢出恢复 | PTL 重试 (丢最旧组, 3次) | remove_first_item | compactAfterOverflow (1次) | 无 | isContextOverflow (1次) |
| 文件追踪 | 重新注入最近5个文件 | world_state 重建 | 无 | 无 | XML 标签 |
| 熔断 | 连续失败3次停止 | 无 | 无 | 无 | 无 |

## 核心设计模式

从 5 个项目的实现里能提炼出几个共性模式：

**1. 阈值 + buffer。** 所有项目都不是等窗口满了才压缩，而是留一个 buffer。buffer 的大小决定了"压缩后还剩多少空间继续跑"。Claude Code 13K，Codex 10%（20K），pi 16K，opencode 20K，crush 20K。

**2. 切点不能落在 tool result 中间。** tool call 和 tool result 必须成对保留。pi 显式检查 `isCutPointMessage`，Claude Code 用 `adjustIndexToPreserveAPIInvariants` 调整，opencode 在 `select` 里隐式处理（按消息边界切）。

**3. 摘要里要包含"下一步做什么"。** 5 个项目的摘要 prompt 都要求模型写明当前进度和下一步。这是为了防止压缩后任务漂移，模型忘了原来在干什么。

**4. 系统提示和工具定义不压缩。** 这两类内容在压缩范围外，始终完整保留。压缩只作用于对话历史。

**5. 保留近期消息。** 除了 crush，其他 4 个项目都保留了切点之后的近期消息（8K-40K token）。纯摘要上下文太薄，保留一些原始对话能让模型有更准确的近期上下文。

下一篇逐个拆源码实现。
