# Claude Code vs OpenAI Codex — 上下文管理机制对比分析

> 本文档从架构、时机、取舍三个维度深度对比 Claude Code 和 OpenAI Codex 的上下文管理机制。

---

## 0. 快速导航

| 对比维度 | 本章 |
|---------|------|
| 架构语言与数据模型 | §1 |
| 系统提示组装 | §2 |
| 上下文注入时机 | §3 |
| 会话历史管理 | §4 |
| Token 预算与压缩触发 | §5 |
| 压缩机制（核心对比） | §6 |
| 工具结果处理 | §7 |
| 用户指令注入（CLAUDE.md vs AGENTS.md） | §8 |
| 缓存策略 | §9 |
| 设计哲学总结 | §10 |

---

## 1. 架构语言与数据模型

| 维度 | Claude Code | OpenAI Codex |
|------|------------|-------------|
| **语言** | TypeScript (Bun runtime) | Rust |
| **API 协议** | Anthropic Messages API | OpenAI Responses API |
| **消息模型** | `Message[]` 数组，类型化联合类型 (`type: 'assistant' \| 'user' \| 'system'`) | `Vec<ResponseItem>` 追加向量，角色区分 (`role: 'developer' \| 'user' \| 'assistant'`) |
| **会话所有权** | `QueryEngine` 实例持有 `mutableMessages: Message[]` | `Session` 持有 `ContextManager` (内含 `Vec<ResponseItem>`) |
| **并发模型** | 单线程事件循环 (Bun) | 异步 Tokio runtime，单活跃 Turn (`Mutex<ActiveTurn>`) |

**WHY 不同**:
- Claude Code 选择 TypeScript 降低开发门槛，利用 Bun 的高性能 I/O；单线程模型天然避免并发问题
- Codex 选择 Rust 追求极致性能和内存安全；需要显式的 `Mutex` 和 `CancellationToken` 管理并发

---

## 2. 系统提示组装

### Claude Code

```
fetchSystemPromptParts() → {
  defaultSystemPrompt: string[]  // getSystemPrompt(tools, model, dirs, mcpClients)
  userContext: { claudeMd, currentDate }
  systemContext: { gitStatus, cacheBreaker }
}

最终 systemPrompt = asSystemPrompt([
  ...(customPrompt ?? defaultSystemPrompt),
  ...(memoryMechanicsPrompt),
  ...(appendSystemPrompt),
])
```

**特点**：
- `systemPrompt` 是一个**字符串数组**拼接，在每次 `submitMessage()` 时组装一次
- `userContext` 和 `systemContext` 通过 `memoize()` 缓存整个会话
- Git 状态是**快照**（会话开始时获取一次，不会更新）
- 无增量 Diff 机制——每次 API 调用都发送完整的 systemPrompt

### OpenAI Codex

```
base_instructions: String  // 模型目录中的系统提示（68-351行）

build_initial_context() → Vec<ResponseItem>:
  developer_sections: [
    model_switch, permissions, developer_instructions,
    memory, collaboration_mode, realtime, personality,
    apps, skills, plugins, commit_attribution
  ]
  contextual_user_sections: [
    user_instructions (AGENTS.md),
    environment_context (cwd, shell, OS, git, subagents)
  ]
```

**特点**：
- `base_instructions` 固定不变，每次 API 调用都发送
- 初始上下文作为 `ResponseItem` 注入到**对话历史**中（developer + user 角色）
- 后续轮次使用 **增量 Diff 引擎**，只发送变化部分
- Git 状态通过环境上下文每次**动态获取**

### 对比

| 维度 | Claude Code | Codex |
|------|------------|-------|
| 系统提示位置 | 顶层 `systemPrompt` 参数 | `base_instructions` + 对话历史中的 developer 消息 |
| 缓存策略 | 整体字符串数组作为缓存前缀 | `base_instructions` 固定前缀 + 增量 Diff 最小化变化 |
| 更新粒度 | 全量重发（memoize 缓存） | 增量 Diff（仅发送变化） |
| Git 状态 | 会话级快照 | 每轮动态获取 |
| 个性/人格 | 不支持 | 支持 friendly/pragmatic 个性注入 |

**WHY 差异**：
- Claude Code 依赖 Anthropic API 的 **prefix caching**——系统提示作为固定前缀自动获得缓存
- Codex 面向多提供商（OpenAI + Ollama + LM Studio），需要自己管理缓存效率，增量 Diff 是优化手段

---

## 3. 上下文注入时机

### Claude Code 时序

```
submitMessage() 调用:
  T1: fetchSystemPromptParts() → 组装 systemPrompt, userContext, systemContext
  T2: processUserInput() → 处理用户输入（slash commands, attachments）
  T3: messages.push(...messagesFromUserInput) → 追加用户消息
  T4: recordTranscript(messages) → 持久化
  T5: for await (message of query(messages, systemPrompt, ...)):
      → 每个 yield 的 assistant/user message 都 push 到 messages
      → API 调用使用完整的 messages 数组
```

**关键点**：Claude Code 没有显式的"上下文组装"步骤。`query()` 函数接收完整的 `messages` 数组，直接传给 API。

### OpenAI Codex 时序

```
run_turn() 调用:
  T1: 预采样压缩检查
  T2: record_context_updates_and_set_reference_context_item()
      → 首轮: build_initial_context() 全量注入
      → 后续: build_settings_update_items() 增量 Diff
  T3: 解析技能/插件/MCP 引用
  T4: 运行 hooks
  T5: 记录用户输入 + 技能/插件注入到历史
  T6: 采样循环:
      T6a: clone_history() → for_prompt() → 标准化
      T6b: build_prompt(input, tools, base_instructions)
      T6c: API 调用
      T6d: 工具执行 → 结果注入历史
      T6e: 检查 token 预算 → 可能触发轮中压缩
```

### 对比

| 时机 | Claude Code | Codex |
|------|------------|-------|
| 系统提示注入 | 每次 submitMessage 前组装 | 会话创建时设置 base_instructions + 首轮注入 developer/user 消息 |
| 环境上下文 | 会话级缓存（memoize） | 首轮全量 + 后续增量 Diff |
| 工具规格 | 每次调用传入 tools 参数 | 每次采样请求重新构建 |
| 用户指令 | userContext.claudeMd（缓存） | 首轮注入 + 后续 Diff |

---

## 4. 会话历史管理

### Claude Code: 简单数组 + 无标准化

```typescript
// QueryEngine.ts
private mutableMessages: Message[]

// 消息追加
this.mutableMessages.push(message)

// 传给 API
query({ messages, systemPrompt, ... })
```

- **无标准化步骤**：消息直接追加，不做 normalize
- **无孤儿检测**：工具调用和工具结果不做配对检查
- **无内置截断**：工具输出不截断（依赖模型自行处理长输出）
- **内存清理**：仅在压缩后通过 `splice(0, boundaryIdx)` 清除旧消息

### OpenAI Codex: 追加向量 + 标准化网关

```rust
// history.rs
pub struct ContextManager {
    items: Vec<ResponseItem>,  // 追加只
}

// 插入时截断
process_item() → truncate_function_output_payload(output, policy * 1.2)

// API 调用前标准化
for_prompt() → [
    ensure_call_outputs_present(),   // 配对修复
    remove_orphan_outputs(),         // 孤儿清理
    strip_images_when_unsupported(), // 模态过滤
]
```

- **三趟标准化**：确保 API 调用始终收到格式正确的输入
- **插入时截断**：工具输出在写入时就截断（1.2x 策略）
- **Ghost 快照**：后台快照存储在历史中但过滤给模型
- **显式克隆**：`clone_history().for_prompt()` 确保 API 调用使用快照

### 对比

| 维度 | Claude Code | Codex |
|------|------------|-------|
| 存储结构 | `Message[]` 数组 | `Vec<ResponseItem>` |
| 标准化 | 无 | 三趟标准化（配对/孤儿/模态） |
| 工具输出截断 | 无内置截断 | 插入时截断 (TruncationPolicy × 1.2) |
| API 调用隔离 | 直接引用同一数组 | 克隆后标准化 |
| 并发安全 | 单线程事件循环 | Mutex 保护，克隆读取 |

---

## 5. Token 预算与压缩触发

### Claude Code

```typescript
// autoCompact.ts
getEffectiveContextWindowSize(model) = contextWindow - maxOutputTokens

getAutoCompactThreshold(model) = effectiveContextWindow - 13_000 buffer

shouldAutoCompact(messages, model):
  tokenCount = tokenCountWithEstimation(messages)
  return tokenCount >= autoCompactThreshold
```

- **上下文窗口**：200K tokens (Claude Sonnet/Opus)
- **压缩阈值**：上下文窗口 - 20K（输出预留） - 13K（缓冲）≈ **167K tokens**
- **Token 估算**：`roughTokenCountEstimation()` 字节级估算
- **触发时机**：每轮 API 调用前检查
- **警告/阻断**：在阈值前 20K 发出警告，在阈值前 3K 阻断

### OpenAI Codex

```rust
// codex.rs
auto_compact_limit = model_info.auto_compact_token_limit()

// 预采样检查
if total_usage_tokens >= auto_compact_limit → compact

// 轮中检查
if token_limit_reached && needs_follow_up → compact
```

- **上下文窗口**：272K tokens (GPT-5.x)
- **压缩阈值**：由 `effective_context_window_percent` 决定（通常 < 100%）
- **Token 估算**：客户端字节估算 + 服务端 API 报告（双层）
- **触发时机**：预采样（API 调用前）+ 轮中（工具执行后）+ 模型切换时

### 对比

| 维度 | Claude Code | Codex |
|------|------------|-------|
| 上下文窗口 | 200K | 272K |
| 压缩缓冲 | 13K tokens | auto_compact_token_limit（模型特定） |
| Token 追踪 | 客户端估算 + API usage 报告 | 客户端估算 + 服务端 TokenUsage 双层 |
| 触发时机 | 每轮 API 调用前 | 预采样 + 轮中 + 模型切换 |
| 阻断机制 | 3K buffer 强制阻断 | 无显式阻断，依赖压缩 |

---

## 6. 压缩机制（核心对比）

这是两者最大的差异点。

### Claude Code: 全量压缩 + 详细摘要

```
compactConversation(messages, ...):
  1. 计算压缩前 token 数
  2. 运行 PreCompact hooks
  3. 组装压缩提示（详细的 9 节结构化摘要模板）
  4. streamCompactSummary():
     → 将全部消息 + 压缩提示发送给 LLM
     → LLM 生成 <analysis> + <summary> 结构化摘要
  5. IF 提示过长 → 截断最旧的 API 轮次组，重试（最多2次）
  6. 清除 readFileState 缓存
  7. 创建压缩后附件：
     → 重新读取最近修改的文件（最多5个，每个5K token）
     → 恢复 Plan 状态
     → 恢复 Agent 状态
     → 恢复已调用的 Skills 内容（最多25K token）
     → 重新注入工具/MCP/Agent 列表
  8. 创建 compact_boundary 消息
  9. 替换历史：[compact_boundary] + [摘要作为 user message] + [附件]
```

**压缩提示结构（9节）**：
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections
4. Errors and Fixes
5. Problem Solving
6. All User Messages
7. Pending Tasks
8. Current Work
9. Optional Next Step

**压缩后恢复**：
- 文件内容：最近修改的最多 5 个文件，每个最多 5K tokens
- Skills 内容：已调用的 skills，总共最多 25K tokens
- 工具列表：重新注入完整的工具/MCP/Agent 描述

### OpenAI Codex: 双路径压缩 + 语义摘要

```
内联压缩 (非 OpenAI 提供商):
  1. 构建 Prompt = 完整历史 + 压缩系统提示
  2. LLM 生成摘要
  3. 构建压缩历史:
     → 用户消息（最新优先，最多 20K tokens）
     → 摘要后缀
  4. IF 超出上下文 → 逐个丢弃最旧条目
  5. replace_compacted_history()

远程压缩 (OpenAI 提供商):
  1. 预修剪：移除最旧的 codex 生成的条目
  2. 调用 /compact_conversation_history 服务端 API
  3. 处理响应，构建压缩历史
  4. 轮中：在最后用户消息前重新注入初始上下文
```

### 核心对比

| 维度 | Claude Code | Codex |
|------|------------|-------|
| **压缩粒度** | 全量（整个会话） | 支持全量 + 部分压缩 |
| **摘要模板** | 9 节结构化模板，非常详细 | 简洁的系统提示（9行） |
| **用户消息保留** | 不保留原始消息，全部进入摘要 | 保留最新 20K token 的用户消息 |
| **文件恢复** | 重新读取最近 5 个文件（25K budget） | 不恢复文件内容 |
| **技能恢复** | 重新注入已调用 skills（25K budget） | 清除 reference_context_item → 下一轮全量注入 |
| **提示过长处理** | 截断最旧 API 轮次组 + 重试（2次） | 截断最旧条目逐个丢弃 |
| **双路径** | 无（统一用 Anthropic API） | 有（OpenAI 用远程，其他用内联） |
| **轮中压缩** | 不支持（仅 API 调用前） | 支持（工具执行后可触发） |
| **缓存共享** | Fork agent 复用主对话 prompt cache | 无显式缓存共享 |
| **压缩后上下文** | compact_boundary + 摘要 + 附件（~50K budget） | 用户消息 + 摘要后缀 |

**WHY 差异分析**：

Claude Code 的压缩更"重"但更精确：
- **WHY 9 节模板**：确保摘要覆盖所有关键信息维度。实验表明简单的摘要容易丢失文件名、代码片段等细节。
- **WHY 文件恢复**：压缩后模型失去所有文件上下文。重新读取最近修改的文件让模型能无缝继续工作。
- **WHY Skills 恢复**：Skills 可能很大（~20K），但其中的指令对后续工作至关重要。

Codex 的压缩更"轻"但更灵活：
- **WHY 保留用户消息**：用户消息包含原始意图和修正。即使摘要丢失细节，原始消息可作为回退参考。
- **WHY 双路径**：OpenAI 的服务端压缩 API 更高效；非 OpenAI 提供商需要客户端压缩。
- **WHY 轮中压缩**：长时间工具执行链可能导致 token 超限。轮中压缩避免丢失整个 turn。

---

## 7. 工具结果处理

### Claude Code

- 工具结果作为 `tool_result` 内容块直接追加到 `Message[]`
- **无内置截断**：长输出原样发送
- 压缩时：工具结果中的图片替换为 `[image]` 占位符
- 工具规格通过 `tools` 参数传入，支持 ToolSearch 动态发现

### OpenAI Codex

- 工具调用和结果作为 `ResponseItem::FunctionCall` / `FunctionCallOutput` 追加
- **插入时截断**：`TruncationPolicy × 1.2` 确保输出不超预算
- **标准化**：`ensure_call_outputs_present()` 为无输出调用插入合成 "aborted" 输出
- **孤儿清理**：`remove_orphan_outputs()` 移除无匹配调用的输出
- 工具规格每次采样请求通过 `built_tools()` 重新构建

### 对比

| 维度 | Claude Code | Codex |
|------|------------|-------|
| 截断策略 | 无内置截断 | TruncationPolicy × 1.2（插入时） |
| 配对保证 | 无 | 三趟标准化 |
| 图片处理 | 压缩时替换为占位符 | 模型不支持时替换为占位文字 |
| 工具发现 | ToolSearch 动态发现 | tool_search + tool_suggest 双机制 |

---

## 8. 用户指令注入（CLAUDE.md vs AGENTS.md）

### Claude Code: CLAUDE.md

```
getUserContext() → {
  claudeMd: getClaudeMds(filterInjectedMemoryFiles(getMemoryFiles())),
  currentDate: "Today's date is ..."
}
```

- **发现机制**：从 cwd 向上遍历目录树查找 `CLAUDE.md` / `CLAUDE.local.md`
- **注入位置**：`userContext` 对象，作为系统提示的一部分
- **缓存**：整个会话 `memoize()`，不会更新
- **嵌套内存**：支持 `~/.claude/memory/` 目录下的项目级记忆文件
- **--bare 模式**：跳过自动发现，仅保留显式 `--add-dir`

### OpenAI Codex: AGENTS.md

```
UserInstructions {
  text: agent_instructions_content,
  directory: cwd.to_string_lossy()
}
.serialize_to_text() → "# AGENTS.md instructions for <path>\n<INSTRUCTIONS>...</INSTRUCTIONS>"
```

- **发现机制**：从 cwd 向上遍历目录树查找 `AGENTS.md`
- **注入位置**：作为 `ResponseItem::Message` (role="user") 注入到对话历史
- **缓存**：首轮全量注入 + 后续增量 Diff
- **分层作用域**：更深层目录的 AGENTS.md 覆盖浅层的
- **子目录延迟加载**：根目录 + cwd-to-root 的 AGENTS.md 预加载；子目录的需要 agent 主动发现

### 对比

| 维度 | CLAUDE.md | AGENTS.md |
|------|----------|----------|
| 文件名 | `CLAUDE.md` / `CLAUDE.local.md` | `AGENTS.md` |
| 注入位置 | 系统提示 (userContext) | 对话历史 (user role message) |
| 更新策略 | 会话级缓存 | 增量 Diff |
| 作用域 | cwd 向上遍历 | 分层覆盖（更深层优先） |
| 记忆系统 | `~/.claude/memory/` | `codex-rs/templates/memories/`（两阶段提取） |

---

## 9. 缓存策略

### Claude Code

- **Prefix Caching**：依赖 Anthropic API 的自动 prefix caching
- `systemPrompt` + `userContext` + `systemContext` 构成缓存前缀
- 压缩时使用 **Fork Agent** 复用主对话的 prompt cache
- 显式的 cache break 机制：`CLAUDE_CODE_BREAK_CACHE` 环境变量

### OpenAI Codex

- **Prefix Stability**：base_instructions 固定在最前面 → 初始上下文 → 用户消息
- **增量 Diff** 最小化变化，保持前缀稳定
- 压缩从最旧处开始，保持前缀不被破坏
- **effective_context_window_percent < 100%** 预留输出空间

### 对比

| 维度 | Claude Code | Codex |
|------|------------|-------|
| 缓存机制 | API 自动 prefix caching | 手动前缀稳定 + 增量 Diff |
| 缓存粒度 | 系统提示级别 | 消息级别（增量 Diff） |
| 压缩缓存 | Fork Agent 复用 cache | 双路径（内联/远程） |
| 缓存破坏 | 显式 cache break | 基线丢失 → 全量注入 |

---

## 10. 设计哲学总结

| 哲学 | Claude Code | OpenAI Codex |
|------|------------|-------------|
| **核心假设** | 单提供商（Anthropic），利用 API 特性 | 多提供商，需要通用机制 |
| **复杂度分配** | 简单数据模型 + 复杂压缩 | 复杂数据模型 + 多层防御 |
| **标准化** | 信任模型处理不一致 | 每次调用前强制标准化 |
| **截断策略** | 不截断，依赖压缩 | 写入时截断 + 读取时标准化 |
| **压缩质量** | 重模板（9节），高精度恢复 | 轻模板（简洁），灵活双路径 |
| **增量更新** | 无（全量重发） | 有（增量 Diff 引擎） |
| **并发安全** | 单线程事件循环 | Mutex + 克隆读取 |
| **Token 管理** | 客户端估算 + API 报告 | 双层（估算 + 服务端真值） |

**一句话总结**：

> **Claude Code** 偏向"简单粗暴但有效"——不做标准化、不做截断、不做增量，但压缩模板极其详细，恢复机制完善。
> **Codex** 偏向"防御性工程"——三趟标准化、插入时截断、增量 Diff、双路径压缩，每层都有兜底。

两者都选择了 **LLM 摘要替代滑动窗口**，这是 2024-2025 年 AI Agent 上下文管理的共识。
