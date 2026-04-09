# 第六章：LLM调用兜底约束机制

## 6.1 概述

Claude Code 在 LLM 调用周围构建了多层兜底约束机制，确保在 API 错误、网络中断、上下文溢出等异常情况下仍能优雅降级。这些机制形成一个**恢复栈**：最轻量的恢复手段先尝试，逐步升级到更激进的措施。

```
API错误
  │
  ├─ 429 Rate Limit ──→ 遵守retry-after, 指数退避重试
  │
  ├─ 529 Overload ──→ 计数, 达到3次后触发模型降级
  │                     ↓
  │              FallbackTriggeredError
  │              Opus → Sonnet (或其他fallback模型)
  │
  ├─ 401 Auth Error ──→ 强制OAuth token refresh, 新客户端重试
  │
  ├─ ECONNRESET/EPIPE ──→ 禁用keep-alive, 重新连接
  │
  ├─ Prompt Too Long ──→ 自动压缩(autoCompact)
  │                        ↓ (失败)
  │                     Reactive Compact
  │                        ↓ (仍然过长)
  │                     Snip Compact(截断历史)
  │                        ↓ (所有恢复耗尽)
  │                     终止: prompt_too_long
  │
  ├─ Max Output Tokens ──→ 首次升级: 8k → 64k
  │                         ↓ (再次触发)
  │                      注入continuation message重试
  │                         ↓ (超过MAX_OUTPUT_TOKENS_RECOVERY_LIMIT=3)
  │                      终止
  │
  ├─ Stream Timeout ──→ 流式→非流式回退
  │
  └─ All Recovery Failed ──→ CannotRetryError, 终止并报告
```

## 6.2 withRetry 架构

`src/services/api/withRetry.ts` (~822行) 是所有 API 调用的重试基础设施。

### 6.2.1 核心参数

```typescript
const DEFAULT_MAX_RETRIES = 10
const MAX_529_RETRIES = 3           // 连续529触发模型降级的阈值
const BASE_DELAY_MS = 500            // 基础延迟
const FLOOR_OUTPUT_TOKENS = 3000     // 上下文溢出后的最小输出token
// 指数退避: 500ms * 2^(attempt-1) + 25% jitter
```

### 6.2.2 重试循环流程

```typescript
async function* withRetry<T>(getClient, operation, options): AsyncGenerator<SystemAPIErrorMessage, T> {
  let consecutive529Errors = 0
  for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
    try {
      // 1. 检查mock rate limits (ant-only测试)
      // 2. 获取/刷新客户端实例 (首次/401/403/ECONNRESET后)
      // 3. 执行操作
      return await operation(client, attempt, retryContext)
    } catch (error) {
      // === 错误分类与处理 ===
      
      // A. Fast mode + 429/529 → 保留fast mode或触发cooldown
      if (wasFastModeActive && is429or529(error)) {
        if (retryAfterMs < SHORT_RETRY_THRESHOLD_MS) {
          await sleep(retryAfterMs)  // 短等待, 保留fast mode
          continue
        }
        triggerFastModeCooldown(...)   // 长等待, 降级到标准速度
        continue
      }
      
      // B. 529 后台源 → 立即放弃, 不放大级联
      if (is529(error) && !shouldRetry529(querySource)) {
        throw new CannotRetryError(error, context)
      }
      
      // C. 连续529 → 模型降级
      if (is529(error) && consecutive529Errors >= MAX_529_RETRIES) {
        if (options.fallbackModel) {
          throw new FallbackTriggeredError(model, fallbackModel)
        }
      }
      
      // D. 超过最大重试次数
      if (attempt > maxRetries) {
        throw new CannotRetryError(error, context)
      }
      
      // E. 不可重试的错误
      if (!shouldRetry(error)) {
        throw new CannotRetryError(error, context)
      }
      
      // F. max_tokens 上下文溢出调整
      if (isMaxTokensOverflowError(error)) {
        retryContext.maxTokensOverride = adjustedMaxTokens
        continue  // 不消耗重试次数(实际消耗)
      }
      
      // G. 计算退避延迟
      const delayMs = getRetryDelay(attempt, retryAfter)
      
      // H. 持久模式 → 分块等待 + 心跳
      if (isPersistentRetryEnabled()) {
        while (remaining > 0) {
          yield createSystemAPIErrorMessage(...)  // 心跳信号
          await sleep(HEARTBEAT_INTERVAL_MS)      // 30s分块
          remaining -= HEARTBEAT_INTERVAL_MS
        }
      } else {
        yield createSystemAPIErrorMessage(...)    // 通知调用者
        await sleep(delayMs)
      }
    }
  }
}
```

### 6.2.3 529 重试白名单

并非所有查询源都在 529 过载时重试——后台操作（摘要、标题、建议）立即放弃：

```typescript
const FOREGROUND_529_RETRY_SOURCES = new Set([
  'repl_main_thread',           // 用户主线程
  'sdk',                        // SDK调用
  'agent:custom',               // 自定义agent
  'agent:default',              // 默认agent
  'agent:builtin',              // 内置agent
  'compact',                    // 压缩
  'hook_agent',                 // Hook agent
  'hook_prompt',                // Hook提示词
  'verification_agent',         // 验证agent
  'side_question',              // 侧边问题
  'auto_mode',                  // 自动模式分类器
  // 其他所有源 → 不重试529
])
```

### 6.2.4 无人值守持久重试

启用 `CLAUDE_CODE_UNATTENDED_RETRY` 后（ant-only）：

- 429/529 无限重试
- 最大退避: `PERSISTENT_MAX_BACKOFF_MS = 5分钟`
- 重置上限: `PERSISTENT_RESET_CAP_MS = 6小时`
- 心跳间隔: `HEARTBEAT_INTERVAL_MS = 30秒`
- 长等待分块为 30s 段，每段 yield `SystemAPIErrorMessage` 保持活跃

## 6.3 模型降级

### 6.3.1 触发条件

连续 529 错误达到 `MAX_529_RETRIES = 3` 次，且配置了 `fallbackModel`：

```
Opus (主模型) ──[529]──→ 重试1
    ──[529]──→ 重试2
    ──[529]──→ 重试3
    ──[529]──→ throw FallbackTriggeredError
                ↓
         Sonnet (降级模型)
```

### 6.3.2 降级处理

`query.ts` 捕获 `FallbackTriggeredError`：

```typescript
catch (error) {
  if (error instanceof FallbackTriggeredError) {
    // 1. 切换到降级模型
    currentModel = error.fallbackModel
    
    // 2. 剥离thinking signature blocks (model-bound)
    // thinking blocks与模型绑定，降级后必须移除
    
    // 3. 丢弃部分工具结果 (如果存在)
    
    // 4. 向用户yield系统警告消息
    
    // 5. state.transition = { reason: 'next_turn' }
    continue  // 重试整个请求
  }
}
```

## 6.4 流式→非流式回退

`src/services/api/claude.ts` 实现了流式到非流式的回退：

```
1. 正常流式请求
   ↓
2. Stream idle watchdog 触发 (可配置超时)
   throw new Error('Stream idle timeout - no chunks received')
   ↓
3. catch 块 → 触发 nonStreamingFallback()
   ↓
4. 非流式调用 withRetry(maxRetries: 0)
   ↓
5. 如果非流式也失败 → 带cost tracking的错误报告
```

可以通过 `tengu_disable_streaming_to_non_streaming_fallback` 特性标志禁用此路径。

## 6.5 上下文溢出恢复栈

当 prompt 超过模型上下文窗口时，query loop 按以下顺序尝试恢复：

### 6.5.1 自动压缩 (Auto-Compact)

`src/services/compact/autoCompact.ts` 监控 token 使用量，接近上下文限制时触发：

```
tokenCount → 接近限制?
  │
  ├─ 是 → 触发 compactConversation()
  │        ├── 成功 → 用摘要替换旧消息, 继续循环
  │        └── 失败 → consecutiveFailures++
  │                    ↓
  │               达到 MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES=3?
  │                    ├── 是 → 停止尝试自动压缩 (断路器)
  │                    └── 否 → 继续循环
  │
  └─ 否 → 正常继续
```

**断路器保护**: 连续 3 次压缩失败后，停止尝试自动压缩。注释说明这可以防止"每天全球约250K API调用浪费"。

### 6.5.2 Reactive Compact

如果自动压缩不可用或失败，`reactive_compact_retry` 触发实时压缩：

```
Prompt Too Long Error
  │
  ├─ hasAttemptedReactiveCompact = false?
  │    ├── 是 → 触发 reactiveCompact.summarizeContext()
  │    │        用摘要替换上下文
  │    │        hasAttemptedReactiveCompact = true
  │    │        state.transition = { reason: 'reactive_compact_retry' }
  │    │        continue
  │    │
  │    └── 否 → 继续到下一恢复层
```

### 6.5.3 Snip Compact

`HISTORY_SNIP` feature-gated，截断旧历史：

```
snipCompactIfNeeded(messagesForQuery)
  │
  ├─ 识别可截断的旧消息
  ├─ 替换为截断边界消息
  ├─ 计算 snipTokensFreed → 传递给autocompact的阈值检查
  └─ yield boundaryMessage (如果产生了边界)
```

### 6.5.4 所有恢复耗尽

如果所有恢复手段都失败：

```
state.transition = undefined
return Terminal { reason: 'prompt_too_long' }
```

## 6.6 输出 Token 恢复

### 6.6.1 Max Output Tokens 升级

当模型因 `max_output_tokens` 限制而停止时：

```
首次触发:
  maxOutputTokensOverride = undefined (默认 ~8k)
    ↓
  升级到 64k (ESCALATED_MAX_TOKENS)
  maxOutputTokensRecoveryCount++
  state.transition = { reason: 'max_output_tokens_escalate' }
  continue

再次触发:
  注入 continuation message:
  "The previous response was cut off. Continue from where you left off."
  maxOutputTokensRecoveryCount++
  state.transition = { reason: 'max_output_tokens_recovery' }
  continue

达到 MAX_OUTPUT_TOKENS_RECOVERY_LIMIT=3:
  终止循环
```

### 6.6.2 Token Budget 系统

`src/query/tokenBudget.ts` (feature-gated `TOKEN_BUDGET`):

```typescript
const COMPLETION_THRESHOLD = 0.9    // 90%预算触发停止评估
const DIMINISHING_THRESHOLD = 500   // delta < 500 tokens视为递减收益

// 递减收益检测:
// 如果连续3+次continuation的delta < 500 tokens → 停止
// 防止无限预算循环
```

每次 continuation 注入 meta 消息：
```
"Stopped at X% of token target... Keep working"
```

## 6.7 Fast Mode 状态机

`src/utils/fastMode.ts`:

```
                    ┌───────────┐
                    │  Active   │
                    └─────┬─────┘
                          │ 429/529
                          ▼
                  ┌───────────────┐
                  │ 检查retry-after │
                  └───┬───────┬───┘
                      │       │
              短(<1min)    长(>=1min)
                      │       │
                      ▼       ▼
              ┌──────────┐ ┌──────────┐
              │ 保持Fast │ │ Cooldown │
              │ 等待+重试 │ │ 降级到    │
              └──────────┘ │ 标准速度   │
                           └────┬─────┘
                                │ 过期
                                ▼
                           ┌──────────┐
                           │ 恢复Fast │
                           └──────────┘
```

CooldownReason: `'rate_limit' | 'overloaded'`

## 6.8 工具结果预算

### 6.8.1 每消息聚合预算

`src/utils/toolResultStorage.ts`:

```
applyToolResultBudget(messages, contentReplacementState, persistCallback, unlimitedTools)
  │
  ├─ 计算每条消息中所有工具结果的总大小
  ├─ 超过限制的结果:
  │    ├── 持久化到磁盘文件
  │    ├── 替换为简短预览 + "Full output written to {path}"
  │    └── 记录contentReplacement到状态中
  └─ 无限制的工具 (如Agent) 跳过预算检查
```

### 6.8.2 微压缩 (MicroCompact)

`src/services/compact/microCompact.ts`:

- **时间基线驱逐**: 旧的工具结果（Read, Bash, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite）按时间清除
- **缓存微压缩**: 使用 API 侧缓存删除，避免重复计算

## 6.9 断路器汇总

| 断路器 | 位置 | 触发条件 | 效果 |
|--------|------|----------|------|
| 自动压缩 | `autoCompact.ts` | 3次连续失败 | 停止尝试自动压缩 |
| Auto-Mode | `permissionSetup.ts` | GrowthBook动态配置 | 禁用/启用自动模式 |
| Fast Mode | `fastMode.ts` | 429/529长延迟 | 降级到标准速度 |
| 529重试 | `withRetry.ts` | 后台源遇到529 | 立即放弃，不放大级联 |
| 上下文溢出 | `query.ts` | 所有恢复失败 | 终止并报告prompt_too_long |

## 6.10 查询循环中的完整恢复流程

```
queryLoop() 每次迭代:
  │
  ├─ 1. Memory prefetch (并行)
  ├─ 2. Snip compact (截断旧历史)
  ├─ 3. Microcompact (清除旧工具结果)
  ├─ 4. Context collapse (折叠上下文)
  ├─ 5. Auto-compact (压缩对话历史)
  ├─ 6. Build fullSystemPrompt
  ├─ 7. Apply tool result budget
  ├─ 8. Call model (streaming API)
  │    │
  │    ├─ 成功 → 继续
  │    ├─ 529 → withRetry处理
  │    ├─ Prompt too long → reactive compact → snip → 终止
  │    ├─ Max output tokens → 升级/continuation → 终止
  │    ├─ Image error → 终止
  │    └─ Abort → 终止
  │
  ├─ 9. Process streaming events
  ├─ 10. Execute tools (parallel if concurrency-safe)
  ├─ 11. Collect tool results
  ├─ 12. Run stop hooks
  ├─ 13. Check token budget
  │
  └─ 决策:
       ├── 有tool_use → state.transition = { reason: 'next_turn' }
       ├── 纯文本 → return Terminal { reason: 'completed' }
       └── 错误/中止 → return Terminal { reason: 'xxx' }
```

## 6.11 关键文件索引

| 文件 | 职责 |
|------|------|
| `src/services/api/withRetry.ts` | 重试循环 (~822行) |
| `src/services/api/claude.ts` | API调用 + 流式回退 (~3,419行) |
| `src/services/api/errors.ts` | 错误分类 (~1,207行) |
| `src/services/api/errorUtils.ts` | 连接错误详情提取 |
| `src/services/compact/autoCompact.ts` | 自动压缩触发 + 断路器 |
| `src/services/compact/compact.ts` | 压缩引擎 (~1,667行) |
| `src/services/compact/microCompact.ts` | 微压缩 |
| `src/services/compact/reactiveCompact.ts` | 响应式压缩 |
| `src/services/compact/snipCompact.ts` | 截断压缩 |
| `src/query/tokenBudget.ts` | Token预算追踪 |
| `src/utils/toolResultStorage.ts` | 工具结果预算 (~950行) |
| `src/utils/fastMode.ts` | Fast mode状态机 |
| `src/services/claudeAiLimits.ts` | Rate limit解析 |
| `src/utils/permissions/permissionSetup.ts` | Auto-mode断路器 |
| `src/query/stopHooks.ts` | 停止Hook处理 |

---

## 相关文档

### 前置阅读
- [[02-核心引擎分析]] - Query Loop 的错误处理流程
- [[11-上下文管理机制深度分析]] - 压缩相关的恢复策略

### 核心依赖
- [[03-prompt-management]] - Prompt 缓存和压缩
- [[05-agent状态轮转流程]] - 状态转换中的容错

### 关联阅读
- [[09-命令系统与服务层]] - API 层的重试机制
- [[10-Harness工程总结]] - 兜底机制的设计权衡
