# Codex 架构分析文档

> 基于 OpenAI Codex 开源代码（Rust实现）的深度架构分析

---

## 文档导航

### 本文档
- [[01-codex-architecture-analysis|OpenAI Codex 深度分析]] - 完整的架构分析

### 上级目录
- [[../README|Claude Code 分析文档主页]]
- [[../v6-design/README|V6 设计文档]]

### 相关分析
- [[../11-上下文管理机制深度分析|上下文管理机制]] - Claude Code 实现
- [[../08-状态管理|状态管理]] - Claude Code 实现

---

## 分析目标

本文档分析 OpenAI Codex（官方 Rust CLI 实现）的架构设计，重点关注：

1. **状态管理机制** - 双 SQLite 设计、WAL 模式、日志分区
2. **上下文管理机制** - 差分更新、历史归一化、上下文压缩
3. **记忆系统** - Stage1/Stage2 异步提取管道
4. **可借鉴的设计决策** - 对 Simu-Emperor V6 的启示

---

## 核心发现速览

### 状态管理
| 设计 | Codex 实现 | V6 方案 |
|------|-----------|---------|
| 数据库架构 | state.sqlite + logs.sqlite | 采用 |
| SQLite 模式 | WAL + 增量 vacuum | 采用 |
| 日志分区 | 按 thread_id 分区，10MB/分区 | 参考 |
| Schema 管理 | sqlx 迁移 | 采用 |

### 上下文管理
| 设计 | Codex 实现 | V6 方案 |
|------|-----------|---------|
| 更新策略 | 差分更新（只传变化） | 采用 |
| 历史归一化 | 自动合并重复项 | 采用 |
| 压缩策略 | LLM-based compaction | 采用 |
| Token 估计 | 字节启发式 | 参考 |

### 记忆系统
| 设计 | Codex 实现 | V6 方案 |
|------|-----------|---------|
| 提取管道 | Stage1 + Stage2 | 采用 |
| 并发控制 | ownership_token | 采用 |
| 存储后端 | SQLite + 文件系统 | 调整 |

---

## 文档关系图

```
docs/
├── README.md                          # 文档主页
├── 01-11.md                           # Claude Code 分析
│
├── v6-design/                         # V6 架构设计
│   ├── 01-architecture-proposal.md    # ← 依赖本文档
│   ├── 02-codex-analysis.md           # ← 本文档副本
│   └── 03-mcp-integration.md          # ← 服务化方案
│
└── codex/                             # 本文档目录
    └── 01-codex-architecture-analysis.md
```

---

## 关键文件映射

### Codex 源码结构
```
codex-rs/
├── core/src/
│   ├── codex.rs              # 308KB - Session/TurnContext
│   ├── context_manager/
│   │   ├── history.rs        # 28KB - ContextManager
│   │   ├── updates.rs        # 8KB - 差分更新
│   │   └── normalize.rs      # 历史归一化
│   └── compact.rs            # 16KB - 上下文压缩
│
├── state/src/
│   ├── runtime.rs            # 14KB - StateRuntime
│   ├── runtime/
│   │   ├── memories.rs       # 168KB - 记忆系统
│   │   ├── threads.rs        # 50KB - 线程管理
│   │   └── logs.rs           # 65KB - 日志管理
│   └── model/
│       ├── memories.rs       # 记忆模型
│       └── thread_metadata.rs
│
└── core/templates/
    └── compact/
        ├── prompt.md         # 压缩提示模板
        └── summary_prefix.md
```

---

*最后更新: 2026-04-14*
