# 第五章 Agent 状态轮转流程

## 概述

Claude Code 的核心是一个基于状态机的查询循环(query loop)，定义在 `src/query.ts`(约 1729 行)中。该循环实现了从用户输入到最终响应的完整生命周期管理，包括上下文压缩、API 调用、工具执行、错误恢复以及多种终止条件的判断。整个循环采用 `while(true)` 结构，通过 `return` 和 `continue` 驱动状态轮转，每一次迭代都执行一系列精心编排的步骤。

本文档将深入分析该状态机的每一个状态、转换条件、恢复路径以及终止条件，揭示 Claude Code 如何在不牺牲可靠性的前提下实现高度自适应的 Agent 行为。

---

## 1. 状态定义

### 1.1 State 类型

状态类型定义于 `src/query.ts` 第 204-217 行:

```typescript
type State = {
  messages: Message[]                                          // 完整对话消息数组
  toolUseContext: ToolUseContext                                // 工具使用上下文
  autoCompactTracking: AutoCompactTrackingState | undefined    // 自动压缩追踪状态
  maxOutputTokensRecoveryCount: number                         // 输出 token 上限恢复计数
  hasAttemptedReactiveCompact: boolean                         // 是否已尝试过响应式压缩
  maxOutputTokensOverride: number | undefined                  // 输出 token 上限覆盖值
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined  // 挂起的工具使用摘要
  stopHookActive: boolean | undefined                          // 停止钩子是否激活
  turnCount: number                                            // 当前轮次计数
  transition: Continue | undefined                             // 上一次迭代继续的原因
}
```

各字段职责:

- **messages**: 完整的对话消息数组，每次迭代都会向前传递。在压缩发生时会被替换为压缩后的消息集合。
- **toolUseContext**: 上下文工具使用状态，包含中止控制器、应用状态、工具列表等核心上下文。
- **autoCompactTracking**: 追踪自动压缩是否已触发、轮次计数器以及断路器失败计数。
- **maxOutputTokensRecoveryCount**: 记录输出 token 上限恢复已尝试的次数，上限为 3 次。
- **hasAttemptedReactiveCompact**: 防止在 prompt-too-long 错误后无限循环进行响应式压缩。
- **maxOutputTokensOverride**: 当设置为 `ESCALATED_MAX_TOKENS` 时，覆盖默认的 8k 输出上限。
- **pendingToolUseSummary**: 前一次迭代中正在执行的 Haiku 工具使用摘要 Promise。
- **stopHookActive**: 标识停止钩子是否在前一次迭代中注入了阻塞性错误。
- **turnCount**: 单调递增的计数器，用于追踪循环迭代次数。
- **transition**: 可辨识联合标签，记录上一次迭代继续的原因，主要用于测试断言。

### 1.2 AutoCompactTrackingState

定义于 `src/services/compact/autoCompact.ts` 第 51-60 行:

```typescript
export type AutoCompactTrackingState = {
  compacted: boolean        // 是否已触发过自动压缩
  turnCounter: number       // 最近一次压缩后的轮次计数
  turnId: string            // 每个压缩周期的唯一 ID
  consecutiveFailures?: number  // 断路器计数，成功时重置
}
```

### 1.3 初始状态

初始状态构建于 `src/query.ts` 第 268-279 行:

```typescript
let state: State = {
  messages: params.messages,
  toolUseContext: params.toolUseContext,
  maxOutputTokensOverride: params.maxOutputTokensOverride,
  autoCompactTracking: undefined,       // 首次迭代前为空
  stopHookActive: undefined,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  turnCount: 1,                         // 从 1 开始计数
  pendingToolUseSummary: undefined,
  transition: undefined,                // 首次迭代无转换原因
}
```

---

## 2. 状态转换总览

### 2.1 ASCII 状态转换图

```
                          ┌─────────────────────────────────────────────────────────┐
                          │                   查询循环入口                           │
                          └─────────────────────┬───────────────────────────────────┘
                                                │
                                                v
                              ┌─────────────────────────────────┐
                              │  A. 解构状态，技能预取，yield     │
                              │     stream_request_start        │
                              └─────────────┬───────────────────┘
                                            │
                                            v
                              ┌─────────────────────────────────┐
                              │  B. 构建消息集 messagesForQuery  │
                              │     (从压缩边界之后)              │
                              └─────────────┬───────────────────┘
                                            │
                                            v
                              ┌─────────────────────────────────┐
                              │  C. 应用工具结果预算              │
                              │     applyToolResultBudget        │
                              └─────────────┬───────────────────┘
                                            │
                                            v
                              ┌─────────────────────────────────┐
                       ┌──────│  D. Snip Compact (HISTORY_SNIP) │
                       │      │     移除旧消息范围               │
                       │      └─────────────┬───────────────────┘
                       │                    │
                       │                    v
                       │      ┌─────────────────────────────────┐
                       │      │  E. Microcompact                │
                       │      │     轻量级上下文裁剪              │
                       │      └─────────────┬───────────────────┘
                       │                    │
                       │                    v
                       │      ┌─────────────────────────────────┐
                       │      │  F. Context Collapse            │
                       │      │     (CONTEXT_COLLAPSE 特性)      │
                       │      └─────────────┬───────────────────┘
                       │                    │
                       │                    v
                       │      ┌─────────────────────────────────┐
                       │      │  G. AutoCompact                 │
                       │      │     自动压缩 (含断路器保护)       │
                       │      └─────────────┬───────────────────┘
                       │                    │
                       │                    v
                       │      ┌─────────────────────────────────┐
                       │  ┌───│  H. Blocking Limit Check        │
                       │  │   │     Token 超过硬限制检查          │
                       │  │   └─────────────┬───────────────────┘
                       │  │                 │ 超过限制
                       │  │                 v
                       │  │   ┌──────────────────────┐
                       │  │   │  终止: blocking_limit │
                       │  │   └──────────────────────┘
                       │  │                 │ 未超过
                       │  │                 v
                       │  │   ┌─────────────────────────────────┐
                       │  │   │  I. API 调用 (含模型降级)         │
                       │  │   │     while(attemptWithFallback)   │
                       │  │   │     ┌─────────────────────┐      │
                       │  │   │     │ FallbackTriggered   │      │
                       │  │   │     │ Error → 切换模型    │──────┤──→ 重试 API 调用
                       │  │   │     └─────────────────────┘      │
                       │  │   └─────────────┬───────────────────┘
                       │  │                 │
                       │  │                 v
                       │  │   ┌─────────────────────────────────┐
                       │  │   │  J. 错误处理                     │
                       │  │   │     ImageError / model_error     │
                       │  │   └─────────────┬───────────────────┘
                       │  │                 │
                       │  │                 v
                       │  │   ┌─────────────────────────────────┐
                       │  │   │  K. 流式中止检查                  │
                       │  │   │     aborted_streaming?           │
                       │  │   └─────────────┬───────────────────┘
                       │  │                 │
                       │  │                 v
                       │  │         ┌───────────────┐
                       │  │         │ 有工具调用?     │
                       │  │         └───────┬───────┘
                       │  │           No    │    Yes
                       │  │       ┌─────────┴──────────┐
                       │  │       v                    v
                       │  │  ┌──────────────┐   ┌───────────────────┐
                       │  │  │ L. 无工具分支 │   │ M. 工具执行分支    │
                       │  │  │              │   │                    │
                       │  │  │ PTL 恢复     │   │ 执行工具           │
                       │  │  │ 媒体恢复     │   │ 收集结果           │
                       │  │  │ OTK 恢复     │   │ 中止/钩子检查      │
                       │  │  │ 停止钩子     │   │ max_turns 检查     │
                       │  │  │ Token 预算   │   │                    │
                       │  │  └──────┬───────┘   └────────┬──────────┘
                       │  │         │                    │
                       │  │         v                    v
                       │  │  ┌──────────────┐   ┌───────────────────┐
                       │  │  │  终止或继续   │   │ 继续下一轮         │
                       │  │  │  completed   │   │ next_turn          │
                       │  │  └──────────────┘   └───────────────────┘
                       │  │
                       └──┴───────────────────────────────────→ 回到循环顶部
```

---

## 3. 单次迭代流程详解

每次循环迭代从 `while(true)` (第 307 行) 开始，依次执行以下步骤:

### 3.1 步骤 A: 状态解构与初始化 (第 311-363 行)

```typescript
let { toolUseContext } = state
const {
  messages, autoCompactTracking, maxOutputTokensRecoveryCount,
  hasAttemptedReactiveCompact, maxOutputTokensOverride,
  pendingToolUseSummary, stopHookActive, turnCount,
} = state
```

随后执行技能发现预取(第 331-335 行)，发出 `stream_request_start` 信号(第 337 行)，初始化查询追踪。

### 3.2 步骤 B: 构建查询消息集 (第 365 行)

```typescript
let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]
```

从最近一次压缩边界之后提取消息，构建本次迭代将要发送给模型的消息集。

### 3.3 步骤 C: 应用工具结果预算 (第 369-394 行)

```typescript
messagesForQuery = await applyToolResultBudget(
  messagesForQuery, toolUseContext.contentReplacementState, ...)
```

对消息集中的工具结果执行预算控制，确保不会因工具输出过大而溢出上下文窗口。

### 3.4 步骤 D: Snip Compact (第 400-410 行)

由 `HISTORY_SNIP` 特性门控，在 Microcompact 之前执行。移除旧的消息范围以释放 token 空间:

```typescript
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
}
```

### 3.5 步骤 E: Microcompact (第 413-426 行)

```typescript
const microcompactResult = await deps.microcompact(
  messagesForQuery, toolUseContext, querySource
)
messagesForQuery = microcompactResult.messages
```

轻量级上下文裁剪，在不触发完整压缩的情况下优化消息集。

### 3.6 步骤 F: Context Collapse (第 440-447 行)

由 `CONTEXT_COLLAPSE` 特性门控，在 AutoCompact 之前执行:

```typescript
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(
    messagesForQuery, ...
  )
  messagesForQuery = collapseResult.messages
}
```

上下文坍缩是一种比传统压缩更激进的上下文管理策略，通过标记和折叠不活跃的上下文段来释放空间。

### 3.7 步骤 G: AutoCompact (第 453-543 行)

```typescript
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery, toolUseContext, {...}, querySource, tracking, snipTokensFreed
)
```

自动压缩是上下文管理的最后一道防线。成功时重置追踪状态，失败时传播失败计数:

- **成功**: 重置 `tracking`，构建压缩后消息，捕获 `taskBudgetRemaining`
- **失败**: 递增 `consecutiveFailures`，超过 3 次则触发断路器

### 3.8 步骤 H: 阻塞限制检查 (第 615-648 行)

当没有刚发生压缩、响应式压缩被禁用、上下文坍缩被禁用时，检查 token 数是否超过硬限制:

```typescript
if (tokenCount > blockingLimit) {
  return { reason: 'blocking_limit' }
}
```

### 3.9 步骤 I: API 调用与模型降级 (第 654-954 行)

内部 `while(attemptWithFallback)` 循环。在流式传输期间，系统收集 `assistantMessages`、`toolUseBlocks` 和 `needsFollowUp` 标志。遇到 `FallbackTriggeredError` 时切换到降级模型并重试。

### 3.10 步骤 J-K: 错误处理与中止检查 (第 955-1052 行)

外层 catch 处理 `ImageSizeError`/`ImageResizeError` (返回 `image_error`)和其他不可恢复错误(返回 `model_error`)。流式完成后检查中止信号，若触发则返回 `aborted_streaming`。

### 3.11 步骤 L: 无工具调用分支 (第 1062-1358 行)

当 `!needsFollowUp` 时进入此分支，按优先级执行以下恢复路径:

1. **Withheld PTL 恢复** (第 1085-1117 行): 先尝试上下文坍缩排空，再尝试响应式压缩
2. **Withheld 媒体恢复** (第 1119-1175 行): 对超大媒体执行响应式压缩剥离重试
3. **Max output tokens 恢复** (第 1188-1256 行): 先升级到 64k，再注入恢复消息
4. **API 错误短路** (第 1262-1265 行): 跳过停止钩子，直接返回 `completed`
5. **停止钩子** (第 1267-1306 行): 返回 `stop_hook_prevented` 或继续 `stop_hook_blocking`
6. **Token 预算延续** (第 1308-1355 行): 返回 `token_budget_continuation` 或正常完成
7. **正常完成** (第 1357 行): 返回 `completed`

### 3.12 步骤 M: 工具执行分支 (第 1360-1727 行)

当 `needsFollowUp` 时进入此分支:

1. 运行工具(流式或同步)
2. 生成工具使用摘要
3. 中止检查(返回 `aborted_tools`)
4. 钩子停止检查(返回 `hook_stopped`)
5. 收集附件(队列命令、内存预取、技能预取)
6. Max turns 检查(返回 `max_turns`)
7. 构建新状态，`continue` 回到循环顶部

---

## 4. 终止原因详解

以下是所有导致循环退出(返回)的原因，按类别组织:

### 4.1 资源限制类

| 原因 | 位置 | 触发条件 |
|------|------|---------|
| `blocking_limit` | 第 646 行 | Token 数超过硬限制，且 auto-compact 关闭 |
| `max_turns` | 第 1711 行 | `nextTurnCount > maxTurns`，达到轮次上限 |
| `prompt_too_long` | 第 1175, 1182 行 | 上下文溢出且所有恢复手段耗尽 |

**blocking_limit**: 这是最严厉的资源限制。当 autocompact 被禁用、reactive compact 不可用、context collapse 也不可用时，如果 token 数超过阻塞限制，循环立即终止。这确保了在没有任何上下文管理手段的情况下不会无意义地继续消耗 API 资源。

**max_turns**: 轮次计数器 `turnCount` 在每次进入工具执行分支并成功完成后递增。当 `nextTurnCount > maxTurns` 时，循环终止并携带当前轮次计数返回。

**prompt_too_long**: 这是一种"穷尽所有恢复手段后"的终止。触发路径:
1. API 返回 413(prompt too long)错误
2. 错误在流式阶段被 withtheld(不暴露给调用者)
3. 尝试 context collapse 排空(若可用)
4. 尝试 reactive compact
5. 如果 `hasAttemptedReactiveCompact` 已经为 `true`，跳过二次尝试
6. 所有恢复失败后，暴露 withtheld 错误并终止

### 4.2 错误类

| 原因 | 位置 | 触发条件 |
|------|------|---------|
| `model_error` | 第 996 行 | 不可恢复的 API/模型错误(非图片/中止) |
| `image_error` | 第 977 行 | `ImageSizeError` 或 `ImageResizeError` |

**model_error**: 捕获所有未被特定处理器识别的错误。携带原始 `error` 对象返回，供调用者进行诊断。这是错误处理的最后兜底。

**image_error**: 专门处理图片相关错误，包括图片尺寸超出限制和图片大小调整失败。这是一个独立的终止路径，因为图片错误通常需要用户干预(如更换图片)才能解决。

### 4.3 用户中止类

| 原因 | 位置 | 触发条件 |
|------|------|---------|
| `aborted_streaming` | 第 1051 行 | 用户按 ESC 中止(流式阶段) |
| `aborted_tools` | 第 1515 行 | 用户中止(工具执行阶段) |

两个中止点分别对应不同的执行阶段:

- **aborted_streaming**: 在 API 流式响应期间，系统持续检查 abort 信号。如果用户按 ESC，流式传输被中断，剩余的流式工具结果被消费(以避免悬挂)，然后返回中止消息。
- **aborted_tools**: 在工具执行阶段，如果 abort 信号触发，工具执行被中断。系统确保所有正在执行的工具获得合成错误结果(通过 sibling abort 机制)，然后返回中止。

### 4.4 钩子控制类

| 原因 | 位置 | 触发条件 |
|------|------|---------|
| `stop_hook_prevented` | 第 1279 行 | 停止钩子设置 `preventContinuation=true` |
| `stop_hook_blocking` | 终止(第 1279 行)/继续(第 1302 行) | 停止钩子返回阻塞性错误 |
| `hook_stopped` | 第 1520 行 | 工具使用钩子指示停止 |

**stop_hook_prevented**: 停止钩子在模型生成文本响应(无工具调用)后运行。如果钩子明确设置 `preventContinuation: true`，循环立即终止。这是一种"软终止"——模型已完成响应，但钩子决定不应继续。

**stop_hook_blocking**: 停止钩子返回阻塞性错误时，有两种可能:
1. 如果同时设置了 `preventContinuation: true`，终止循环
2. 否则，将错误作为用户消息注入，设置 `stopHookActive: true`，继续循环让模型处理这些错误

**hook_stopped**: 工具执行阶段的钩子(Pre-tool hook)可以返回 `stop` 信号。当检测到 `hook_stopped_continuation` 附件时，循环立即终止。

### 4.5 正常完成

| 原因 | 位置 | 触发条件 |
|------|------|---------|
| `completed` | 第 1264, 1357 行 | 正常完成: 文本响应，无工具调用，无错误 |

这是最理想的终止路径。模型生成了完整的文本响应，没有发起工具调用，没有任何错误或钩子阻止继续。循环自然结束。

---

## 5. 继续原因详解

以下是所有导致循环继续(不返回)的转换原因:

### 5.1 正常工具执行继续

#### `next_turn`

**位置**: 第 1725 行

**触发条件**: 工具执行成功完成，工具结果需要送回模型进行下一轮推理。

**状态变更**:
```typescript
state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext,
  autoCompactTracking: tracking,
  maxOutputTokensRecoveryCount: 0,       // 重置
  hasAttemptedReactiveCompact: false,     // 重置
  maxOutputTokensOverride: undefined,     // 重置
  pendingToolUseSummary: summaryPromise,  // 新的摘要 Promise
  stopHookActive: undefined,              // 重置
  turnCount: nextTurnCount,               // 递增
  transition: { reason: 'next_turn' },
}
```

这是最频繁的转换路径。每次工具执行完成后，模型需要看到工具结果来决定下一步行动。

### 5.2 上下文恢复类

#### `collapse_drain_retry`

**位置**: 第 1110 行

**触发条件**: 上下文坍缩系统排空了待处理的坍缩队列(withheld PTL 413 之后)。

**状态变更**:
- 保持相同的消息集(排空后的)
- 保持相同的 `turnCount`
- 重置 `maxOutputTokensOverride`
- 保留 `hasAttemptedReactiveCompact`

**关键限制**: 仅触发一次——通过检查 `state.transition?.reason !== 'collapse_drain_retry'` 防止无限循环。

#### `reactive_compact_retry`

**位置**: 第 1162 行

**触发条件**: 响应式压缩在 prompt-too-long 或媒体大小错误后成功执行了完整摘要。

**状态变更**:
- 替换消息为压缩后的消息
- 重置 `autoCompactTracking: undefined`
- 设置 `hasAttemptedReactiveCompact: true`
- 重置 `maxOutputTokensOverride`

**关键保护**: `hasAttemptedReactiveCompact` 被设为 `true`，防止无限循环。注释特别指出:
```
// Resetting to false here caused an infinite loop: compact -> still too long ->
// error -> stop hook blocking -> compact -> ... burning thousands of API calls.
```

### 5.3 输出 Token 恢复类

#### `max_output_tokens_escalate`

**位置**: 第 1218 行

**触发条件**: 首次遇到输出 token 上限(OTK)，将默认 8k 上限升级到 64k。

**状态变更**:
- 保持相同的消息
- 设置 `maxOutputTokensOverride: ESCALATED_MAX_TOKENS`
- **不**递增 `maxOutputTokensRecoveryCount`

**前置条件**: 需要 `tengu_otk_slot_v1` 特性启用，且当前无手动覆盖(`maxOutputTokensOverride === undefined`)，且无环境变量覆盖。

#### `max_output_tokens_recovery`

**位置**: 第 1246 行

**触发条件**: OTK 升级后仍然命中限制，注入恢复消息让模型继续。

**状态变更**:
- 追加助手消息 + 恢复用户消息
- 递增 `maxOutputTokensRecoveryCount`
- 重置 `maxOutputTokensOverride: undefined`(防止升级重复触发)

**恢复消息内容**:
```
Output token limit hit. Resume directly — no apology, no recap of what you were doing.
Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.
```

**上限**: 最多恢复 3 次(`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`)。

### 5.4 钩子驱动类

#### `stop_hook_blocking`

**位置**: 第 1302 行

**触发条件**: 停止钩子返回阻塞性错误但未阻止继续。

**状态变更**:
- 追加助手消息 + 阻塞性错误(作为用户消息)
- 重置 `maxOutputTokensRecoveryCount: 0`
- 保留 `hasAttemptedReactiveCompact`
- 设置 `stopHookActive: true`

### 5.5 Token 预算类

#### `token_budget_continuation`

**位置**: 第 1338 行

**触发条件**: Token 预算系统判定"还有余量，应该继续工作"。

**状态变更**:
- 追加助手消息 + 提示消息(nudge message)
- 重置 `maxOutputTokensRecoveryCount: 0`
- 重置 `hasAttemptedReactiveCompact: false`
- 重置 `maxOutputTokensOverride: undefined`
- 重置 `stopHookActive`

**提示消息格式**:
```
Stopped at {pct}% of token target ({used} / {budget}). Keep working — do not summarize.
```

**适用范围**: 仅适用于主线程(非子 agent)。子 agent 和无预算的场景直接返回 `stop`。

---

## 6. 关键恢复路径流程图

### 6.1 Prompt-Too-Long 恢复路径

```
API 返回 413 (prompt_too_long)
            │
            v
    ┌───────────────────┐
    │ 错误被 withtheld   │  ← 不暴露给 SDK 调用者
    │ (流式阶段拦截)      │
    └────────┬──────────┘
             │
             v
    ┌───────────────────────┐     已是 collapse_drain_retry?
    │ Context Collapse 排空  │───────→ 跳过
    │ (若可用)               │
    └────────┬──────────────┘
             │ 排空了 committed > 0
             v
    ┌───────────────────┐
    │ collapse_drain_   │
    │ retry (继续循环)   │
    └───────────────────┘
             │ 排空了 0 (无进展)
             v
    ┌───────────────────────┐     hasAttemptedReactiveCompact?
    │ Reactive Compact      │───────→ 跳过
    │ (压缩整个上下文)       │
    └────────┬──────────────┘
             │ 成功
             v
    ┌───────────────────┐
    │ reactive_compact_ │
    │ retry (继续循环)   │
    └───────────────────┘
             │ 失败
             v
    ┌───────────────────┐
    │ 暴露 withthold 错误│
    │ 返回 prompt_too_  │
    │ long (终止)        │
    └───────────────────┘
```

### 6.2 Max Output Tokens 恢复路径

```
模型响应被截断 (max_output_tokens)
            │
            v
    ┌───────────────────┐     已有覆盖值?
    │ OTK 升级           │───────→ 跳过升级
    │ 8k → 64k           │
    └────────┬──────────┘
             │ 首次升级成功
             v
    ┌───────────────────────┐
    │ max_output_tokens_    │
    │ escalate (继续循环)    │
    └───────────────────────┘
             │ 升级后仍截断
             v
    ┌───────────────────────┐     recoveryCount >= 3?
    │ 注入恢复消息           │───────→ 放弃，返回 completed
    │ "Resume directly..."  │
    └────────┬──────────────┘
             │ recoveryCount < 3
             v
    ┌───────────────────────────┐
    │ max_output_tokens_recovery│
    │ (继续循环，最多 3 次)      │
    └───────────────────────────┘
```

---

## 7. 状态字段的跨迭代生命周期

### 7.1 重置策略分类

| 字段 | 正常工具执行 | PTL 恢复 | OTK 恢复 | 停止钩子 | Token 预算 |
|------|-------------|---------|---------|---------|-----------|
| `messages` | 追加工具结果 | 替换为压缩后 | 追加恢复消息 | 追加错误消息 | 追加提示消息 |
| `autoCompactTracking` | 保持 | 重置(undefined) | 保持 | 保持 | 保持 |
| `maxOutputTokensRecoveryCount` | 重置(0) | 保持 | 递增 | 重置(0) | 重置(0) |
| `hasAttemptedReactiveCompact` | 重置(false) | 设为 true | 保持 | 保持 | 重置(false) |
| `maxOutputTokensOverride` | 重置(undefined) | 重置 | 设置/重置 | 保持 | 重置(undefined) |
| `stopHookActive` | 重置(undefined) | 保持 | 保持 | 设为 true | 重置(undefined) |
| `turnCount` | 递增 | 保持 | 保持 | 保持 | 保持 |

### 7.2 守卫标志的协同工作

`hasAttemptedReactiveCompact` 和 `stopHookActive` 是两个关键的守卫标志，它们协同工作以防止恢复路径之间的无限循环:

- `hasAttemptedReactiveCompact`: 防止在 prompt-too-long 错误后反复尝试响应式压缩。一旦尝试过，即使压缩后上下文仍然太大，也不会再次尝试。
- `stopHookActive`: 防止停止钩子在同一轮次中被重复执行。当停止钩子注入阻塞性错误后，下一次迭代不会再次运行停止钩子。

这两个标志的交叉保护尤为重要。在 `stop_hook_blocking` 的继续路径中，`hasAttemptedReactiveCompact` 被显式保留:

```typescript
hasAttemptedReactiveCompact: state.hasAttemptedReactiveCompact,  // 保留!
```

这是因为重置此标志曾导致过严重的无限循环(数千次 API 调用浪费)，详见第 1297 行的注释。

---

## 8. 工具执行中的并发与中止

### 8.1 StreamingToolExecutor 并发模型

定义于 `src/services/tools/StreamingToolExecutor.ts`，该执行器管理多个工具的并发执行:

```typescript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

并发安全规则:
- 如果没有工具在执行，任何工具都可以开始
- 如果有工具在执行，新工具必须与所有正在执行的工具都并发安全
- 非并发安全的工具会中断队列处理

### 8.2 Sibling Abort 机制

当 Bash 工具出错时，所有兄弟工具获得合成错误结果:

```typescript
if (isErrorResult && tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.siblingAbortController.abort('sibling_error')
}
```

设计决策: 只有 Bash 错误会触发兄弟中止。原因在代码注释中说明:

> Bash commands often have implicit dependency chains (e.g. mkdir fails -> subsequent commands pointless). Read/WebFetch/etc are independent — one failure shouldn't nuke the rest.

---

## 9. Token 预算系统的状态集成

### 9.1 预算追踪器状态

定义于 `src/query/tokenBudget.ts` 第 6-11 行:

```typescript
type BudgetTracker = {
  continuationCount: number      // "继续"返回的次数
  lastDeltaTokens: number        // 上次检查以来的 token 增量
  lastGlobalTurnTokens: number   // 上次检查时的全局轮次 token 数
  startedAt: number              // 追踪器创建时间戳
}
```

### 9.2 递减收益检测

```typescript
const isDiminishing =
  tracker.continuationCount >= 3 &&                    // 至少 3 次继续
  deltaSinceLastCheck < DIMINISHING_THRESHOLD &&       // 当前增量 < 500
  tracker.lastDeltaTokens < DIMINISHING_THRESHOLD      // 上次增量 < 500
```

双重增量检查确保 Agent 是持续产出极少新工作，而非仅有一个慢步骤。

### 9.3 决策逻辑

- **继续**: 非递减收益 且 `turnTokens < budget * 0.9`
- **停止(预算耗尽)**: `turnTokens >= budget * 0.9`
- **停止(递减收益)**: 连续两次增量 < 500 且已继续 3 次以上

---

## 10. 总结

Claude Code 的查询循环状态机是一个多层次、多路径的状态转换系统。其核心设计原则包括:

1. **防御性恢复优先**: 在终止前尝试所有可用的恢复手段(压缩、坍缩排空、OTK 升级等)。
2. **断路器保护**: 每种恢复机制都有内置的失败计数器和上限，防止无限重试浪费资源。
3. **守卫标志协同**: `hasAttemptedReactiveCompact` 和 `stopHookActive` 等标志跨迭代协同工作，防止恢复路径之间的循环依赖。
4. **渐进升级**: 从轻量级(Microcompact)到重量级(AutoCompact)再到激进(Reactive Compact)的渐进式上下文管理策略。
5. **清晰的终止语义**: 每个终止原因都有明确的触发条件和携带的额外信息，便于调用者进行诊断和决策。

---

## 相关文档

### 前置阅读
- [[02-核心引擎分析]] - Query Loop 的无限循环机制
- [[04-任务管理机制]] - 任务与 Agent 状态的关系

### 核心依赖
- [[03-prompt-management]] - Prompt 中的状态指示
- [[06-LLM调用兜底约束机制]] - 状态转换的错误恢复

### 关联阅读
- [[07-工具系统]] - 工具执行对状态的影响
- [[08-状态管理]] - 全局状态与 Agent 状态
