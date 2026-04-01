# Claude Code 源码深度分析

> 本文档对 Claude Code（Anthropic 的 CLI 工具）的源码架构进行深度分析。Claude Code 是一个 ~512K LOC 的 TypeScript 应用，运行在 Bun runtime 上，使用 React + Ink 构建 Terminal UI。

## 文档目录

| 章节 | 文件 | 内容 |
|------|------|------|
| 第一章 | [01-整体架构概览.md](./01-整体架构概览.md) | 入口点、初始化流程、模块组织、特性标志 |
| 第二章 | [02-核心引擎分析.md](./02-核心引擎分析.md) | QueryEngine、Query Loop、消息处理管道 |
| 第三章 | [03-prompt-management.md](./03-prompt-management.md) | Prompt管理、系统提示词组装、Prompt工程技巧 |
| 第四章 | [04-任务管理机制.md](./04-任务管理机制.md) | 任务生命周期、多Agent协调、Agent间通信 |
| 第五章 | [05-agent状态轮转流程.md](./05-agent状态轮转流程.md) | Agent状态机、查询循环转换、Terminal/Continue原因 |
| 第六章 | [06-LLM调用兜底约束机制.md](./06-LLM调用兜底约束机制.md) | 重试逻辑、模型降级、压缩机制、断路器 |
| 第七章 | [07-工具系统.md](./07-工具系统.md) | 工具注册、验证管道、执行引擎、权限系统 |
| 第八章 | [08-状态管理.md](./08-状态管理.md) | 全局状态、响应式Store、AppState字段详解 |
| 第九章 | [09-命令系统与服务层.md](./09-命令系统与服务层.md) | Slash命令系统、服务层架构、MCP/LSP/OAuth |
| 总结 | [10-Harness工程总结.md](./10-Harness工程总结.md) | 设计哲学、关键技术决策、架构权衡 |

## 重点分析领域

本文档系列重点分析以下四个核心领域（根据用户要求）：

### 1. Prompt管理和Prompt工程技巧
详见 [第三章](./03-prompt-management.md)

- **四层组装流水线**: 静态内容 → 动态边界 → 注册管理的动态内容 → 优先级覆盖链
- **Prompt缓存策略**: global/org/null三级缓存作用域，SYSTEM_PROMPT_DYNAMIC_BOUNDARY分割标记
- **12种Prompt工程技巧**: 思维链草稿、结构化输出强制、No-Tools声明、数值长度锚点等

### 2. 任务管理机制
详见 [第四章](./04-任务管理机制.md)

- **双重任务系统**: 运行时任务(Runtime Tasks) vs 用户管理的Todo任务
- **多Agent协调**: Coordinator Mode、Agent Spawning、Swarm架构
- **Agent间通信**: 文件邮箱系统、结构化协议消息、优先级命令队列

### 3. Agent状态轮转流程
详见 [第五章](./05-agent状态轮转流程.md)

- **查询循环状态机**: while(true)无限循环 + State可变状态
- **11种Terminal原因**: 从正常完成到各种异常终止
- **7种Continue原因**: 从正常工具循环到各种恢复路径

### 4. LLM调用兜底约束机制
详见 [第六章](./06-LLM调用兜底约束机制.md)

- **五层恢复栈**: 模型降级 → 自动压缩 → Reactive Compact → 输出Token升级 → 流式回退
- **withRetry架构**: 10次重试、指数退避、429/529/401/ECONNRESET分别处理
- **三个断路器**: 自动压缩(3次失败)、Auto-Mode(GrowthBook动态配置)、Fast Mode(429/529 cooldown)

## 技术栈

| 类别 | 技术 |
|------|------|
| 运行时 | Bun |
| 语言 | TypeScript (strict) |
| Terminal UI | React + Ink |
| CLI 解析 | Commander.js |
| Schema 验证 | Zod v4 |
| 代码搜索 | ripgrep |
| 协议 | MCP SDK, LSP |
| API | Anthropic SDK |
| 遥测 | OpenTelemetry + gRPC |
| Feature Flags | GrowthBook |
| 认证 | OAuth 2.0, JWT, macOS Keychain |

## 源码规模

```
~1,900 文件
~512,000 行代码
```

关键大文件：
| 文件 | 行数 | 职责 |
|------|------|------|
| `src/screens/REPL.tsx` | ~5,000 | 交互式Terminal UI |
| `src/services/api/claude.ts` | ~3,419 | API调用层 |
| `src/services/compact/compact.ts` | ~1,667 | 对话压缩引擎 |
| `src/commands.ts` | ~25,000 | 命令注册 |
| `src/main.tsx` | ~4,684 | CLI入口 |
| `src/Tool.ts` | ~29,000 | 工具类型系统 |
| `src/QueryEngine.ts` | ~1,295 | 查询引擎 |
| `src/query.ts` | ~1,729 | 查询循环 |
| `src/services/tools/toolExecution.ts` | ~1,745 | 工具执行管道 |
| `src/bootstrap/state.ts` | ~1,760 | 全局状态 |
| `src/utils/toolResultStorage.ts` | ~950 | 工具结果存储 |
