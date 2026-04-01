# Claude Code Harness Engineering 总结

## 概述

Claude Code 作为一个 ~512K LOC 的 TypeScript CLI 应用，展现了极其精密的 Harness Engineering（"套件工程"）——即在 LLM 调用和用户交互之间构建的完整工程化基础设施层。本章总结其核心设计哲学和关键技术决策。

## 1. Harness Engineering 的定义

在 LLM 应用中，"Harness"指的是将原始的 LLM API 调用转化为可靠、可控、可观测的产品级系统所需的所有外围工程：

```
用户输入 → [Harness] → LLM API → [Harness] → 用户输出
              ↑                          ↑
         Prompt组装                结果验证/约束
         上下文管理                错误恢复
         工具调度                  状态追踪
         权限控制                  并发管理
```

Claude Code 的 Harness 由以下层次组成：

| 层次 | 职责 | 核心文件 |
|------|------|----------|
| **入口编排** | CLI解析、初始化、特性门控 | `entrypoints/cli.tsx`, `main.tsx` |
| **Prompt组装** | 系统提示词、上下文注入、缓存优化 | `constants/prompts.ts`, `context.ts`, `utils/api.ts` |
| **查询引擎** | LLM调用循环、工具调度、状态管理 | `QueryEngine.ts`, `query.ts` |
| **工具系统** | 工具注册、验证、执行、权限 | `Tool.ts`, `tools.ts`, `services/tools/` |
| **任务管理** | 任务生命周期、多Agent协调 | `Task.ts`, `tasks/`, `utils/swarm/` |
| **安全约束** | 重试、降级、压缩、预算控制 | `services/api/withRetry.ts`, `services/compact/` |
| **状态管理** | 全局状态、响应式Store | `bootstrap/state.ts`, `state/` |
| **服务集成** | MCP、LSP、Analytics、OAuth | `services/` |

## 2. 核心设计模式

### 2.1 分层缓存策略

Claude Code 对 Prompt 缓存的优化达到了极其精细的程度：

- **全局缓存作用域** (`cacheScope: 'global'`): 系统提示词的静态部分（Intro, System, Tools 等段落），跨用户/跨组织共享
- **组织缓存作用域** (`cacheScope: 'org'`): 组织特定的内容
- **动态边界标记**: `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 精确分割静态和动态内容
- **记忆化段注册**: `systemPromptSection()` 确保动态内容只计算一次
- **内容哈希路径**: 设置文件使用内容哈希生成路径，避免随机 UUID 导致缓存失效

这种设计使得系统提示词的大部分内容可以跨请求复用缓存，显著降低了 token 成本（约 12x 输入 token 成本的差异）。

### 2.2 渐进式恢复策略

Claude Code 实现了多层次的错误恢复机制：

```
API错误 → 重试(withRetry, 10次, 指数退避)
   ↓ (连续529)
模型降级(FallbackTriggeredError, Opus → Sonnet)
   ↓ (prompt过长)
自动压缩(autoCompact, 压缩对话历史)
   ↓ (压缩失败)
Reactive Compact(实时压缩)
   ↓ (max_output_tokens)
输出Token升级(8k → 64k)
   ↓ (仍然失败)
流式→非流式回退
   ↓ (上下文溢出)
Snip Compact(截断历史)
   ↓ (所有恢复失败)
终止并报告错误
```

每一层恢复机制都是独立触发的，且大多数有断路器保护（如自动压缩最多连续失败3次后停止尝试）。

### 2.3 功能门控与死代码消除

利用 Bun 的 `bun:bundle` 的 `feature()` 函数实现编译时功能门控：

```typescript
const module = feature('FLAG')
  ? require('./featureModule.js')
  : null
```

约 20 个功能标志（PROACTIVE, KAIROS, DAEMON, VOICE_MODE 等）使得不同构建可以包含或排除整个子系统。`process.env.USER_TYPE === 'ant'` 检查在构建时被内联为常量，外部构建中将 Ant-only 代码完全消除。

### 2.4 并行预取架构

启动时间被优化到毫秒级别：

1. **Import 阶段并行**: `startMdmRawRead()` 和 `startKeychainPrefetch()` 作为顶层副作用在 import 评估时并行启动
2. **Init 阶段并行**: MDM settings, keychain reads, API preconnect 同时执行
3. **首次渲染后延迟预取**: `startDeferredPrefetches()` 在用户看到 UI 后才启动非关键预取（user context, tips, file counts 等）
4. **查询时并行**: `fetchSystemPromptParts()` 并行获取 systemPrompt, userContext, systemContext

### 2.5 工具结果预算管理

为防止工具结果膨胀导致上下文溢出：

- **每消息聚合预算**: `applyToolResultBudget()` 限制单条消息中所有工具结果的总大小
- **磁盘置换**: 超大结果持久化到磁盘文件，替换为简短预览
- **微压缩**: 时间基线的旧工具结果驱逐（Read, Bash, Grep, Glob 等）
- **缓存微压缩**: API 侧缓存删除，避免重复计算

## 3. 关键架构决策分析

### 3.1 为什么选择无限循环而非递归

查询循环使用 `while(true)` + 状态对象而非递归调用：

- **优势**: 避免调用栈增长、状态转换清晰（State对象驱动）、更容易实现恢复和断路器
- **代价**: 需要手动管理 `state = { ... }` 的不可变更新

### 3.2 为什么使用文件邮箱而非 IPC

Agent间通信使用文件系统而非进程间通信：

- **跨进程一致性**: 无论 agent 是 in-process 还是独立 tmux 进程，协议相同
- **崩溃恢复**: 文件在进程崩溃后仍然存在
- **调试友好**: 可以直接查看 JSON 文件内容
- **代价**: 文件锁开销（proper-lockfile, 10 retries, 5-100ms backoff）

### 3.3 为什么使用可变全局状态而非纯依赖注入

`bootstrap/state.ts` 的 1760 行全局可变状态看起来反直觉，但有实际原因：

- **减少参数传递**: 避免将 session ID、model config 等通过 10+ 层函数传递
- **循环依赖打破**: 全局状态作为各模块的共享协调点
- **性能**: 避免频繁创建大的 context 对象
- **代价**: 测试需要更多 mock，隐式依赖

## 4. 可观测性设计

Claude Code 实现了全面的可观测性：

- **分析事件**: `logEvent()` 跟踪所有关键操作（startup, query, tool use, compact, retry 等）
- **诊断日志**: `logForDiagnosticsNoPII()` 记录无 PII 的诊断信息
- **Profile checkpoints**: `profileCheckpoint()` 标记启动各阶段耗时
- **Telemetry**: OpenTelemetry metrics/logs/traces（lazy-loaded）
- **Session transcripts**: 完整的对话记录持久化到磁盘，支持 `/resume`

## 5. 安全设计

- **权限系统**: 每次工具调用都经过权限检查（canUseTool），支持多种模式（default/plan/bypassPermissions/auto）
- **沙箱**: Bash 工具支持沙箱执行
- **Hook 系统**: 用户可配置 hooks 拦截工具调用
- **Cyber Risk Instruction**: 系统提示词包含安全指导
- **Prompt 注入检测**: 工具结果中检测潜在的 prompt 注入

## 6. 扩展性设计

- **插件系统**: `src/plugins/` 支持第三方插件
- **Skill 系统**: `src/skills/` 支持可复用的工作流
- **MCP 集成**: `src/services/mcp/` 支持 Model Context Protocol 服务器
- **Agent 定义**: JSON/Markdown 格式的自定义 agent 配置
- **命令扩展**: 通过插件添加自定义 slash 命令

## 7. 总结

Claude Code 的 Harness Engineering 展示了一个成熟的 LLM 应用应具备的工程化水平：

1. **可靠性**: 多层恢复机制确保即使在 API 错误、网络中断、上下文溢出等异常情况下仍能优雅降级
2. **性能**: 并行预取、缓存优化、延迟加载确保快速启动和响应
3. **可扩展性**: 插件、Skill、MCP、Agent 定义提供了丰富的扩展点
4. **可观测性**: 全面的日志、遥测、诊断确保问题可追踪
5. **安全性**: 权限系统、沙箱、Hook、安全指令形成了多层防御

这些设计模式对于构建任何生产级 LLM 应用都具有参考价值。
