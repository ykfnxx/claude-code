# 第三章：Prompt管理与Prompt工程技巧

## 3.1 系统提示词组装流水线

Claude Code 的系统提示词采用**四层组装流水线**架构，在 `src/constants/prompts.ts`（914行）中实现。

### 3.1.1 Layer 1: 静态内容（全局可缓存）

静态内容在所有用户和组织之间共享，可以使用 `cacheScope: 'global'` 进行 API 级别的 prompt 缓存：

```
┌─────────────────────────────────────────────┐
│ Intro Section                                │
│ "You are an interactive agent..."            │
│ + CYBER_RISK_INSTRUCTION                     │
├─────────────────────────────────────────────┤
│ System Section                               │
│ - 输出格式规则 (Github-flavored markdown)     │
│ - 工具权限行为                                │
│ - Hook 反馈处理                               │
│ - 自动压缩通知                                │
├─────────────────────────────────────────────┤
│ Doing Tasks Section                          │
│ - "Don't gold-plate"                         │
│ - "Three similar lines > premature abstract" │
│ - 最小变更哲学                                │
│ - 错误处理方法                                │
├─────────────────────────────────────────────┤
│ Actions Section                              │
│ - 风险评估框架 (可逆性/影响范围)               │
│ - 用户确认策略                                │
│ - 破坏性操作示例列表                          │
├─────────────────────────────────────────────┤
│ Using Your Tools Section                     │
│ - 优先专用工具而非Bash                        │
│ - 并行工具调用指导                            │
├─────────────────────────────────────────────┤
│ Tone & Style Section                         │
│ - Emoji策略 ("Only if explicitly requested")  │
│ - 引用格式 (file_path:line_number)            │
│ - 不在tool call前使用冒号                     │
├─────────────────────────────────────────────┤
│ Output Efficiency Section                    │
│ - Ant: 详细沟通指导 (flowing prose, ~150行)   │
│ - External: "Be extra concise" (~30行)        │
└─────────────────────────────────────────────┘
```

**Intro Section** 的实际文本：
```
You are an interactive agent that helps users with software engineering tasks.
Use the instructions below and the tools available to you to assist the user.

IMPORTANT: You must NEVER generate or guess URLs for the user unless you are
confident that the URLs are for helping the user with programming.
```

**Doing Tasks Section** 的关键指令：
```
- Don't add features, refactor code, or make "improvements" beyond what was asked.
- Don't create helpers, utilities, or abstractions for one-time operations.
- Three similar lines of code is better than a premature abstraction.
- In general, do not propose changes to code you haven't read.
- Do not create files unless they're absolutely necessary.
```

**Ant-only 反模式指令**（外部构建中被 DCE 消除）：
```
- Default to writing no comments. Only add one when the WHY is non-obvious.
- Don't explain WHAT the code does, since well-named identifiers already do that.
- Before reporting a task complete, verify it actually works.
- Report outcomes faithfully: if tests fail, say so with the relevant output.
```

**Actions Section** 定义了风险框架：
```
Carefully consider the reversibility and blast radius of actions. Generally you
can freely take local, reversible actions like editing files or running tests.
But for actions that are hard to reverse, affect shared systems beyond your
local environment, or could otherwise be risky or destructive, check with the
user before proceeding.

Examples of risky actions:
- Destructive: deleting files/branches, dropping database tables, rm -rf
- Hard-to-reverse: force-pushing, git reset --hard, amending published commits
- Shared state: pushing code, creating/closing PRs, sending messages
- Third-party: uploading to pastebins/gists (may be cached or indexed)
```

### 3.1.2 Layer 2: 动态边界标记

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

这个特殊字符串精确分割静态内容和动态内容。**一切在边界标记之前的内容**使用 `cacheScope: 'global'`（跨用户/跨组织共享）；**一切之后的内容**包含用户/会话特定信息，不应被全局缓存。

这个设计直接影响 API 的 prompt 缓存命中率——不当放置会导致约 12x 输入 token 成本惩罚。

### 3.1.3 Layer 3: 动态内容（注册管理）

动态内容通过 `src/constants/systemPromptSections.ts` 的注册系统管理：

```typescript
// 记忆化段 — 计算一次，缓存在 bootstrap/state.ts 的 Map 中
systemPromptSection('session_guidance', () => getSessionSpecificGuidanceSection(...))
systemPromptSection('memory', () => loadMemoryPrompt())
systemPromptSection('env_info_simple', () => computeSimpleEnvInfo(model, dirs))
systemPromptSection('language', () => getLanguageSection(settings.language))
systemPromptSection('output_style', () => getOutputStyleSection(config))
systemPromptSection('scratchpad', () => getScratchpadInstructions())
systemPromptSection('frc', () => getFunctionResultClearingSection(model))
systemPromptSection('summarize_tool_results', () => SUMMARIZE_TOOL_RESULTS_SECTION)

// 危险的非缓存段 — 每次查询重新计算
// 用于 MCP instructions（服务器在 turns 之间可能连接/断开）
DANGEROUS_uncachedSystemPromptSection('mcp_instructions',
  () => isMcpInstructionsDeltaEnabled() ? null : getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns'
)
```

**Env Info Section** 的实际输出格式：
```xml
# Environment
You have been invoked in the following environment:
 - Primary working directory: /path/to/project
 - Is a git repository: Yes
 - Additional working directories: /other/path
 - Platform: darwin
 - Shell: zsh
 - OS Version: Darwin 25.3.0
 - You are powered by the model named Claude Sonnet 4.6.
   The exact model ID is claude-sonnet-4-6.
 - Assistant knowledge cutoff is August 2025.
 - The most recent Claude model family is Claude 4.5/4.6.
```

### 3.1.4 Layer 4: 优先级覆盖链

`src/utils/systemPrompt.ts` 的 `buildEffectiveSystemPrompt()` 实现了以下优先级链：

```
1. overrideSystemPrompt    → 替换一切 (loop mode)
2. Coordinator prompt      → 多agent协调器模式
3. Agent prompt            → proactive模式: APPENDED; 否则: REPLACES
4. customSystemPrompt      → --system-prompt CLI flag
5. Default system prompt   → 上述四层组装的结果
6. appendSystemPrompt      → 始终在最后添加
```

## 3.2 上下文注入机制

### 3.2.1 System Context

`src/context.ts` 的 `getSystemContext()` 收集 Git 相关信息，**追加到系统提示词末尾**：

```
This is the git status at the start of the conversation.
Note that this status is a snapshot in time, and will not update during the conversation.

Current branch: feature/my-branch
Main branch (you will usually use this for PRs): main
Git user: Developer Name
Status:
M  src/file.ts
?? new-file.ts
Recent commits:
abc1234 Fix bug
def5678 Add feature
```

注意：Git status 有 2000 字符的截断限制，超出部分会被截断并提示用户使用 BashTool 获取更多信息。

### 3.2.2 User Context

`getUserContext()` 收集的上下文**作为 meta user message 注入到对话开头**：

```typescript
// src/utils/api.ts - prependUserContext()
// 将上下文包装在 <system-reminder> XML 标签中
return `<system-reminder>
Context from project CLAUDE.md files:
${userContext.claudeMd}

Today's date is 2026-04-01.
</system-reminder>`
```

CLAUDE.md 文件从以下位置自动发现：
- 项目根目录及其父目录
- `~/.claude/` 目录
- `--add-dir` 指定的额外目录

### 3.2.3 缓存作用域分割

`src/utils/api.ts` 的 `splitSysPromptPrefix()` 将系统提示词分割为不同缓存作用域：

```
[cacheScope: null]    — attribution header (永不缓存)
[cacheScope: 'global'] — 静态内容 (跨用户共享)
[cacheScope: 'org']    — 组织特定内容
```

## 3.3 Prompt工程技巧详解

### 3.3.1 思维链草稿 (Chain-of-Thought Drafting)

**应用场景**: 对话压缩 (`src/services/compact/prompt.ts`)

压缩提示词使用 `<analysis>` 标签作为模型的推理草稿空间，然后 `<summary>` 标签包含最终输出：

```xml
<analysis>
[Your thought process, ensuring all points are covered thoroughly and accurately]
1. Chronologically analyze each message...
2. Double-check for technical accuracy...
</analysis>

<summary>
1. Primary Request and Intent:
   [Detailed description]
2. Key Technical Concepts:
   - [Concept 1]
...
</summary>
```

`formatCompactSummary()` 在压缩输出进入上下文之前剥离 `<analysis>` 部分，只保留干净的 `<summary>`。这是一个两阶段技巧：先自由草稿，再提取结构化输出。

### 3.3.2 结构化输出强制

压缩提示词定义了严格的 9 段编号格式，配合 `<example>` XML 块：

```
Your summary should include the following sections:

1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections
4. Errors and fixes
5. Problem Solving
6. All user messages
7. Pending Tasks
8. Current Work
9. Optional Next Step
```

每个段落都有详细的内容要求，例如第 8 段：
```
8. Current Work: Describe in detail precisely what was being worked on
immediately before this summary request, paying special attention to the
most recent messages from both user and assistant.
```

### 3.3.3 激进的 No-Tools 声明

压缩路径继承父级的完整工具集（为了 prompt 缓存匹配），但模型有时会尝试调用工具。解决方案是**双重声明**：

**Preamble（前置声明，放在最前面）**:
```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
```

注释解释了这个设计的统计原因：
> "With maxTurns: 1, a denied tool call means no text output → falls through to the streaming fallback (2.79% on 4.6 vs 0.01% on 4.5)"

### 3.3.4 Prompt 缓存共享

压缩 agent 通过发送与主对话相同的缓存键参数来复用 prompt 缓存：

```typescript
// 压缩agent复用主对话的缓存前缀
// 发送相同的 system prompt + tools + model + messages prefix + thinking config
// 如果缓存命中，可以节省大量输入 token
```

如果共享路径失败，会回退到独立的流式路径。

### 3.3.5 记忆化段注册

`systemPromptSection()` 创建延迟计算、一次缓存的提示词段落：

```typescript
// src/constants/systemPromptSections.ts
export function systemPromptSection(
  name: string,
  compute: () => Promise<string | null> | string | null,
): SystemPromptSection
```

缓存存储在 `bootstrap/state.ts` 的 `Map<string, string | null>` 中，在 `/clear` 和 `/compact` 时清除。

`DANGEROUS_uncachedSystemPromptSection()` 用于 MCP instructions 等在 turns 之间变化的内容。命名中的 "DANGEROUS" 是因为每次重新计算会导致缓存失效。

### 3.3.6 变量模板替换

Magic Docs 和 Session Memory 提示词使用 `{{variableName}}` 语法：

```typescript
// 单次替换，避免双重替换 bug
function applyTemplate(template: string, vars: Record<string, string>): string {
  let result = template
  for (const [key, value] of Object.entries(vars)) {
    result = result.replaceAll(`{{${key}}}`, value)
  }
  return result
}
```

自定义提示词可从用户配置文件加载：
- `~/.claude/magic-docs/prompt.md`
- `~/.claude/session-memory/config/`

### 3.3.7 数值长度锚点

**仅 Ant 构建**，外部构建中被 DCE：

```
Length limits: keep text between tool calls to ≤25 words.
Keep final responses to ≤100 words unless the task requires more detail.
```

注释说明："Numeric length anchors — research shows ~1.2% output token reduction vs qualitative 'be concise'. Ant-only to measure quality impact first."

### 3.3.8 模型特定行为修正

系统提示词中多处使用 `process.env.USER_TYPE === 'ant'` 条件分支（构建时被内联为常量），针对特定模型行为加入修正指令：

**False-claims 缓解** (针对 Capybara v8 的 29-30% FC rate):
```
Report outcomes faithfully: if tests fail, say so with the relevant output;
if you did not run a verification step, say that rather than implying it
succeeded. Never claim "all tests pass" when output shows failures.
```

**Assertiveness 修正**:
```
If you notice the user's request is based on a misconception, or spot a bug
adjacent to what they asked about, say so. You're a collaborator, not just
an executor.
```

**Comment 修正**:
```
Default to writing no comments. Only add one when the WHY is non-obvious.
```

**Thoroughness 修正**:
```
Before reporting a task complete, verify it actually works: run the test,
execute the script, check the output. Minimum complexity means no
gold-plating, not skipping the finish line.
```

### 3.3.9 渐进式披露

不同模式获得不同的提示词变体：

| 模式 | 差异 |
|------|------|
| **Simple mode** | 极简系统提示词 (~2行) |
| **Proactive mode** | 自主agent + pace + tick handling |
| **Coordinator mode** | worker编排 + 并发规则 |
| **REPL mode** | 简化的工具指导 |
| **Output style** | 可替换intro段 + 抑制编码指令 |

MCP 指令使用 delta 附件机制 (`isMcpInstructionsDeltaEnabled()`)，仅在变化时注入增量更新，而非每 turn 重新包含完整指令列表。

### 3.3.10 Agent 提示词

子 agent 使用 `DEFAULT_AGENT_PROMPT`：
```
You are an agent for Claude Code, Anthropic's official CLI for Claude.
Given the user's message, you should use the tools available to complete the task.
Complete the task fully—don't gold-plate, but don't leave it half-done.
When you complete the task, respond with a concise report covering what was done
and any key findings — the caller will relay this to the user, so it only needs
the essentials.
```

然后通过 `enhanceSystemPromptWithEnvDetails()` 追加：
```
Notes:
- Agent threads always have their cwd reset between bash calls, as a result
  please only use absolute file paths.
- In your final response, share file paths (always absolute, never relative)
  that are relevant to the task.
- For clear communication with the user the assistant MUST avoid using emojis.
- Do not use a colon before tool calls.
```

### 3.3.11 内容哈希路径

设置文件使用内容哈希生成临时文件路径（而非随机 UUID），避免 prompt 缓存失效：

```typescript
// 随机UUID路径会导致每次子进程调用都改变工具描述中的sandbox deny list
// 从而使prompt cache前缀失效，造成约12x输入token成本惩罚
settingsPath = generateTempFilePath('claude-settings', '.json', {
  contentHash: trimmedSettings  // 内容相同 → 路径相同 → 缓存命中
})
```

## 3.4 专用提示词系统

### 3.4.1 Coordinator 提示词

`src/coordinator/coordinatorMode.ts` 中 369 行的多 agent 编排提示词定义了完整的协调协议：

- **4个工作阶段**: Research(并行) → Synthesis → Implementation → Verification
- **并发规则**: 只读任务自由并行；写密集型每文件集一个
- **Worker prompt 编写指南**: Always synthesize findings, include file paths/line numbers
- **Continue vs spawn 决策矩阵**: 基于 worker 现有上下文与下一个任务的上下文重叠度

### 3.4.2 输出风格

`src/constants/outputStyles.ts` 定义可配置的输出风格：

- **Explanatory**: 详细解释模式
- **Learning**: 教学模式
- **Concise**: 简洁模式

每种风格有一个 `prompt` 字段，可以替换或增强系统提示词的 intro 和 doing-tasks 段落。

### 3.4.3 记忆系统提示词

`src/memdir/memdir.ts` 构建类型化记忆行为指令，使用封闭分类体系：

- `user` — 用户偏好和习惯
- `feedback` — 用户反馈
- `project` — 项目特定知识
- `reference` — 参考信息

### 3.4.4 XML 标签常量

`src/constants/xml.ts` 定义了所有在提示词中使用的 XML 标签名：

```typescript
LOCAL_COMMAND_STDERR_TAG  // <local-command-stderr>
LOCAL_COMMAND_STDOUT_TAG  // <local-command-stdout>
TICK_TAG                  // <tick>
TASK_NOTIFICATION_TAG     // <task-notification>
TEAMMATE_MESSAGE_TAG      // <teammate-message>
```

## 3.5 关键文件索引

| 文件 | 职责 |
|------|------|
| `src/constants/prompts.ts` | 主系统提示词工厂 (~914行) |
| `src/utils/systemPrompt.ts` | 优先级解析 |
| `src/constants/systemPromptSections.ts` | Section注册系统 |
| `src/utils/systemPromptType.ts` | Branded type `SystemPrompt` |
| `src/utils/queryContext.ts` | 查询上下文构建 |
| `src/context.ts` | 系统/用户上下文收集 (189行) |
| `src/utils/api.ts` | 上下文注入 (`prependUserContext`, `appendSystemContext`) |
| `src/services/compact/prompt.ts` | 压缩提示词模板 (374行) |
| `src/coordinator/coordinatorMode.ts` | 协调器提示词 (369行) |
| `src/constants/outputStyles.ts` | 输出风格配置 |
| `src/constants/cyberRiskInstruction.ts` | 安全指令 |
| `src/memdir/memdir.ts` | 记忆系统提示词 |
| `src/constants/xml.ts` | XML标签常量 |
