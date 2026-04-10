# Simu-Emperor V6 架构提案

## 研究背景

基于对以下两个关键系统的深度分析：
1. **OpenAI Codex** (Rust实现，官方开源)
2. **MCP (Model Context Protocol)** (skill/service 标准)

## 1. 核心发现对比

### 1.1 OpenAI Codex 架构

```
codex-rs/
├── core/                    # Agent核心逻辑
│   ├── src/
│   │   ├── codex.rs         # Session/TurnContext 核心协调
│   │   ├── context_manager/ # 上下文管理
│   │   │   ├── history.rs   # ResponseItem历史管理
│   │   │   ├── updates.rs   # 差分上下文更新
│   │   │   └── normalize.rs # 历史规范化
│   │   ├── compact.rs       # LLM-based上下文压缩
│   │   └── ...
│   └── templates/           # 提示模板
├── state/                   # 状态持久化层
│   ├── src/
│   │   ├── runtime.rs       # StateRuntime (SQLite双库)
│   │   ├── runtime/
│   │   │   ├── threads.rs   # 线程CRUD
│   │   │   ├── memories.rs  # Stage1/Stage2记忆提取
│   │   │   └── logs.rs      # 日志分区管理
│   │   └── model/
│   │       ├── thread_metadata.rs  # 线程元数据
│   │       └── memories.rs         # 记忆模型
│   └── migrations/          # SQLx迁移
├── skills/                  # 技能系统 (提示模板)
└── protocol/                # 协议定义
```

**关键设计决策：**
- **双SQLite数据库**: state.sqlite(核心状态) + logs.sqlite(日志)
- **Turn-based处理**: 每个turn有独立context snapshot
- **差分上下文更新**: 只传递变化的部分(environment/permissions/model)
- **LLM-based压缩**: 显式compaction task，结构化prompt模板
- **Stage1/Stage2记忆**: 先提取再整合的异步管道

### 1.2 MCP 架构

```
MCP服务层:
├── Resources (只读数据)
├── Tools (可执行操作)
└── Prompts (可复用模板)

客户端:
├── 连接管理 (stdio/sse)
├── Capability协商
└── 工具调用路由
```

**关键设计决策：**
- **服务解耦**: Agent通过MCP调用外部能力
- **Capability声明**: 服务自检暴露能力
- **渐进式发现**: 动态工具加载

## 2. Simu-Emperor V6 设计

### 2.1 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  Web UI     │  │  CLI        │  │  Claude Code/Other      │  │
│  └──────┬──────┘  └──────┬──────┘  │  Agents (via MCP)       │  │
│         └─────────────────┘         └─────────────────────────┘  │
│                          │                                      │
│                    ┌─────┴─────┐                                │
│                    │  SDK      │  Python SDK (当前)             │
│                    │  (Python) │  未来: Rust/Go/TS SDKs         │
│                    └─────┬─────┘                                │
└──────────────────────────┼──────────────────────────────────────┘
                           │ SSE/HTTP
┌──────────────────────────┼──────────────────────────────────────┐
│                     MCP Service Layer (新)                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  Game Tools │  │  Memory     │  │  Storage/Retrieval      │  │
│  │  Service    │  │  Service    │  │  Service                │  │
│  │  (游戏交互)  │  │  (上下文管理)│  │  (向量检索)              │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  Code Exec  │  │  File       │  │  Custom Skills...       │  │
│  │  Service    │  │  Service    │  │                         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                           │ Internal API
┌──────────────────────────┼──────────────────────────────────────┐
│                     Core Server (Rust - 未来)                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  Session    │  │  Event      │  │  Persistence            │  │
│  │  Manager    │  │  Router     │  │  (SQLite/WAL)           │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐                                │
│  │  Queue      │  │  WebSocket  │                                │
│  │  Controller │  │  Manager    │                                │
│  └─────────────┘  └─────────────┘                                │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 与V5的关键差异

| 方面 | V5 (当前) | V6 (提案) |
|------|-----------|-----------|
| 工具位置 | Agent代码中硬编码 | MCP服务暴露 |
| 上下文管理 | JSONL文件+SQLite | 专用Memory MCP服务 |
| Agent核心 | 基类继承 | bub框架 + hooks |
| 多Agent | 父子任务 | 对等Agent通过MCP发现 |
| 服务器 | Python FastAPI | Rust (长期) |
| 状态存储 | 分散(JSONL/Chroma) | 统一SQLite+WAL |

### 2.3 三阶段迁移路径

```
Phase 1: MCP服务层 (2-3周)
├── 将游戏交互工具移至MCP服务
├── Agent通过MCP调用而非直接调用
└── 保持现有Python服务器

Phase 2: Agent核心重构 (3-4周)
├── 引入bub作为最小agent核心
├── 自定义tape/context系统通过hooks
├── 支持Claude Code集成

Phase 3: 服务器Rust重写 (4-6周)
├── 功能验证完成后开始
├── 借鉴Codex的state/runtime设计
└── 双SQLite架构(state + logs)
```

## 3. 关键技术决策

### 3.1 为何选择MCP而非Function Calling？

**MCP优势：**
1. **解耦**: Agent无需了解工具实现
2. **发现**: 动态capability协商
3. **多Agent**: 不同agent可暴露不同服务
4. **生态**: 社区标准，工具复用

**实现策略：**
- 游戏工具 → GameTools MCP服务
- 上下文管理 → ContextManager MCP服务
- 记忆检索 → Memory MCP服务 (基于Chroma)

### 3.2 Agent核心: bub vs 自研

**选择bub的理由：**
```rust
// bub核心 (~200行)
pub struct Bub<S: State> {
    state: S,
    hooks: Hooks,
}

impl<S: State> Bub<S> {
    pub fn step(&mut self, event: Event) -> Result<Action> {
        self.hooks.before_step(event)?;
        let action = self.state.process(event)?;
        self.hooks.after_step(&action)?;
        Ok(action)
    }
}
```

**自定义扩展点：**
- `tape.rs`: 基于republic的turn记录
- `context.rs`: 差分上下文更新 (借鉴Codex)
- `memory.rs`: Stage1/Stage2记忆管道

### 3.3 状态管理: 借鉴Codex

**双SQLite设计：**
```rust
// state.sqlite - 核心状态
struct StateRuntime {
    pool: SqlitePool,           // 5连接, WAL模式
    // threads, agents, sessions
}

// logs.sqlite - 操作日志
struct LogsRuntime {
    pool: SqlitePool,
    // 分区管理, 10MB/分区, 自动rotate
}
```

**Codex值得借鉴的实现：**
- **sqlx迁移**: 版本化schema管理
- **WAL模式**: 并发读写优化
- **日志分区**: 避免单表过大
- **增量vacuum**: 自动空间回收

### 3.4 上下文压缩策略

**V6将采用混合策略：**
```
1. 轻量级压缩 (token count > 80%窗口)
   ├── 截断早期tool output
   └── 保留user message和decision points

2. LLM-based压缩 (token count > 95%窗口)
   ├── 触发compaction task
   ├── 使用结构化prompt (借鉴Codex)
   └── 生成handoff summary

3. 长期记忆 (跨session)
   ├── Stage1: 提取关键决策/状态
   ├── Stage2: 跨session整合
   └── 检索时注入relevant memories
```

## 4. 记忆系统设计

### 4.1 分层记忆架构

```
┌──────────────────────────────────────┐
│  Working Memory (上下文窗口)          │
│  - Current session turn history      │
│  - 自动压缩管理                       │
└──────────────────────────────────────┘
           │
           ▼ (overflow/compaction)
┌──────────────────────────────────────┐
│  Short-term Memory (SQLite)           │
│  - Session summaries                  │
│  - Recent N sessions                  │
│  - Turn-level metadata                │
└──────────────────────────────────────┘
           │
           ▼ (periodic extraction)
┌──────────────────────────────────────┐
│  Long-term Memory (Chroma + SQLite)   │
│  - Stage1: 提取的记忆片段              │
│  - Stage2: 整合的knowledge graph      │
│  - Vector embedding for retrieval     │
└──────────────────────────────────────┘
```

### 4.2 Stage1/Stage2记忆管道

借鉴Codex的memories实现：

```rust
// Stage1: 单session提取
struct Stage1Output {
    thread_id: String,
    raw_memory: String,         // 提取的记忆文本
    rollout_summary: String,    // session摘要
    cwd: PathBuf,
    git_branch: Option<String>,
    generated_at: DateTime<Utc>,
}

// Stage2: 跨session整合
struct Phase2InputSelection {
    selected: Vec<Stage1Output>,
    retained_thread_ids: Vec<String>,
    removed: Vec<Stage1OutputRef>,
}
```

### 4.3 检索策略

```python
# 检索时注入上下文
def retrieve_memories(query: str, session_context: dict) -> list:
    # 1. 向量相似度检索
    similar = vector_store.similarity_search(query, k=5)
    
    # 2. 时间衰减加权
    scored = [(m, score * time_decay(m.created_at)) for m in similar]
    
    # 3. 相关性重排序
    reranked = rerank_by_relevance(scored, session_context)
    
    # 4. 格式化注入prompt
    return format_memories(reranked[:3])
```

## 5. 实现优先级

### 5.1 P0: MCP服务层 (立即开始)

**GameTools MCP服务：**
```typescript
// 暴露当前Agent代码中的工具
{
  "tools": [
    {"name": "create_incident", "description": "..."},
    {"name": "update_indicator", "description": "..."},
    {"name": "create_task_session", "description": "..."},
    // ...
  ]
}
```

**迁移步骤：**
1. 创建 `packages/mcp-services/game-tools/`
2. 将 `tool_*.py` 逻辑移至MCP服务
3. Agent通过 `@mcp.tool()` 调用而非直接实现
4. 保持向后兼容的fallback

### 5.2 P1: 上下文管理优化

**引入差分更新：**
```python
# 当前: 每次传递完整context
context = {
    "environment": {...},      # 每次都一样
    "permissions": {...},      # 很少变化
    "history": [...]           # 只新增了几条
}

# V6: 差分更新
if previous_context:
    delta = compute_delta(previous_context, current_context)
    send_delta(delta)  # 只传变化部分
else:
    send_full_context()  # 首次全量
```

### 5.3 P2: 记忆系统增强

**实施Stage1/Stage2：**
1. 后台任务提取session记忆
2. 向量存储集成
3. 检索时自动注入

### 5.4 P3: 服务器Rust重写

**前提条件：**
- MCP服务层稳定运行
- Agent核心基于bub重构完成
- 性能瓶颈验证

## 6. 风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| MCP引入延迟 | 中 | 中 | 本地stdio连接，<10ms |
| Rust学习曲线 | 中 | 中 | 先验证功能，后迁移 |
| 数据迁移 | 低 | 高 | 提供migration脚本 |
| 性能回退 | 低 | 高 | A/B测试，可回滚 |

## 7. 结论

V6架构的核心改进：

1. **解耦**: 通过MCP分离agent核心与工具实现
2. **标准**: 采用社区标准，降低生态锁定
3. **可扩展**: 新工具通过添加MCP服务而非修改agent
4. **性能**: 借鉴Codex的SQLite+WAL设计
5. **记忆**: 分层记忆系统，支持长期上下文

下一步行动：
- [ ] 创建MCP服务原型
- [ ] 评估bub框架适配性
- [ ] 设计数据迁移方案
