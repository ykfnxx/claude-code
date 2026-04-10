# OpenAI Codex 深度分析

## 分析范围

基于 OpenAI Codex 开源代码 (Rust实现) 的深度分析，重点关注：
1. 状态管理机制
2. 上下文管理架构
3. 记忆系统实现
4. 可借鉴的设计决策

## 1. 项目结构概览

```
codex-rs/
├── core/              # Agent核心逻辑 (308KB codex.rs)
├── state/             # 状态持久化层
├── protocol/          # 协议定义
├── skills/            # 技能系统
└── cli/               # 命令行界面
```

## 2. 状态管理架构

### 2.1 双SQLite数据库设计

Codex采用**双数据库分离**策略：

```rust
// state/src/runtime.rs
pub struct StateRuntime {
    codex_home: PathBuf,
    pool: Arc<sqlx::SqlitePool>,        // state.sqlite
    logs_pool: Arc<sqlx::SqlitePool>,   // logs.sqlite (分离！)
}
```

**分离原因：**
- 日志写入频繁，会锁表
- 状态查询频繁，需要低延迟
- 分离后避免锁竞争

**V6可借鉴：**
```rust
// Simu-Emperor V6 状态架构
pub struct StateRuntime {
    state_pool: SqlitePool,     // sessions, agents, threads
    logs_pool: SqlitePool,      // events, messages
    vector_store: ChromaClient, // embeddings
}
```

### 2.2 SQLite配置优化

```rust
// state/src/runtime.rs
fn base_sqlite_options(path: &Path) -> SqliteConnectOptions {
    SqliteConnectOptions::new()
        .filename(path)
        .create_if_missing(true)
        .journal_mode(SqliteJournalMode::Wal)      // WAL模式！
        .synchronous(SqliteSynchronous::Normal)     // 性能平衡
        .busy_timeout(Duration::from_secs(5))
        .log_statements(LevelFilter::Off)
}

async fn open_state_sqlite(path: &Path, migrator: &Migrator) -> anyhow::Result<SqlitePool> {
    let options = base_sqlite_options(path)
        .auto_vacuum(SqliteAutoVacuum::Incremental); // 增量vacuum
    
    let pool = SqlitePoolOptions::new()
        .max_connections(5)                          // 限制连接数
        .connect_with(options)
        .await?;
    
    migrator.run(&pool).await?;                     // sqlx迁移
    Ok(pool)
}
```

**关键配置：**
- **WAL模式**: 读写并发，崩溃恢复
- **增量vacuum**: 自动空间回收
- **5连接上限**: SQLite推荐值
- **sqlx迁移**: 版本化schema管理

### 2.3 日志分区管理

Codex有一个精妙的日志分区系统：

```rust
// state/src/runtime.rs
const LOG_PARTITION_SIZE_LIMIT_BYTES: i64 = 10 * 1024 * 1024;  // 10MB/分区
const LOG_PARTITION_ROW_LIMIT: i64 = 1_000;                     // 1000行/分区

// 分区策略：
// - 每个thread_id一个分区
// - 无thread的按process_uuid分组
// - 无process的放入默认分区
```

**V6可借鉴：**
- 按agent_id分区event logs
- 自动rotate避免单表过大
- 异步清理旧分区

## 3. 上下文管理架构

### 3.1 ContextManager核心

```rust
// core/src/context_manager/history.rs
pub(crate) struct ContextManager {
    items: Vec<ResponseItem>,           // 历史记录
    token_info: Option<TokenUsageInfo>, // token统计
    reference_context_item: Option<TurnContextItem>, // 差分基线
}
```

**关键方法：**
```rust
impl ContextManager {
    // 记录新items
    pub(crate) fn record_items<I>(&mut self, items: I, policy: TruncationPolicy)
    
    // 生成prompt (自动归一化)
    pub(crate) fn for_prompt(mut self, input_modalities: &[InputModality]) -> Vec<ResponseItem>
    
    // 估计token数 (字节启发式)
    pub(crate) fn estimate_token_count(&self, turn_context: &TurnContext) -> Option<i64>
    
    // 移除最旧item (上下文压缩用)
    pub(crate) fn remove_first_item(&mut self)
}
```

### 3.2 差分上下文更新

Codex的一个精妙设计：**只传递变化的部分**

```rust
// core/src/context_manager/updates.rs
pub(crate) fn build_settings_update_items(
    previous: Option<&TurnContextItem>,
    previous_turn_settings: Option<&PreviousTurnSettings>,
    next: &TurnContext,
    shell: &Shell,
    exec_policy: &Policy,
    personality_feature_enabled: bool,
) -> Vec<ResponseItem> {
    let developer_update_sections = [
        // 按优先级排序：模型切换 > 权限 > 协作模式 > 实时模式
        build_model_instructions_update_item(previous_turn_settings, next),
        build_permissions_update_item(previous, next, exec_policy),
        build_collaboration_mode_update_item(previous, next),
        build_realtime_update_item(previous, previous_turn_settings, next),
        build_personality_update_item(previous, next, personality_feature_enabled),
    ]
    .into_iter()
    .flatten()
    .collect();
    
    // 只生成非空更新
    build_developer_update_item(developer_update_sections)
}
```

**对比V5：**
```python
# V5: 每次都传完整context
system_prompt = f"""
环境: {environment}
权限: {permissions}
历史: {history}
"""

# Codex: 差分更新
if previous_context:
    delta = compute_delta(previous, next)
    # 只发送: "模型从gpt-4切换到claude-3"
    # 而不是重复环境、权限等不变信息
```

### 3.3 历史归一化

```rust
// core/src/context_manager/normalize.rs
pub(crate) fn normalize_history(items: &mut Vec<ResponseItem>) {
    // 1. 合并连续的assistant消息
    merge_consecutive_assistant_messages(items);
    
    // 2. 合并重复的function_call
    merge_duplicate_function_calls(items);
    
    // 3. 移除未匹配的function_call_output
    remove_unmatched_outputs(items);
    
    // 4. 修复call/output顺序
    fix_call_output_ordering(items);
}
```

**V5问题：** 没有归一化，导致：
- 重复function calls
- 未匹配的outputs
- 顺序错乱

## 4. 记忆系统实现

### 4.1 Stage1/Stage2管道

Codex的记忆系统采用**两阶段异步提取**：

```rust
// state/src/model/memories.rs
/// Stage1: 单线程记忆提取输出
pub struct Stage1Output {
    pub thread_id: ThreadId,
    pub rollout_path: PathBuf,
    pub source_updated_at: DateTime<Utc>,
    pub raw_memory: String,         // 提取的记忆
    pub rollout_summary: String,    // 线程摘要
    pub cwd: PathBuf,
    pub git_branch: Option<String>,
    pub generated_at: DateTime<Utc>,
}

/// Stage1任务认领结果
pub enum Stage1JobClaimOutcome {
    Claimed { ownership_token: String },
    SkippedUpToDate,           // 已有更新的输出
    SkippedRunning,            // 其他worker在处理
    SkippedRetryBackoff,       // 在退避期
    SkippedRetryExhausted,     // 重试耗尽
}
```

### 4.2 Stage1提取流程

```rust
// state/src/runtime/memories.rs (168KB!)
impl StateRuntime {
    /// 认领Stage1提取任务
    pub async fn claim_stage1_job(&self, params: Stage1StartupClaimParams<'_>)
        -> anyhow::Result<Vec<Stage1JobClaim>>
    {
        // 1. 查询待处理的threads
        // 2. 尝试获取分布式锁
        // 3. 返回成功认领的任务
    }
    
    /// 提交Stage1输出
    pub async fn submit_stage1_output(&self, output: Stage1Output, token: &str)
        -> anyhow::Result<()>
    {
        // 1. 验证ownership_token
        // 2. 检查source_updated_at是否过时
        // 3. 写入stage1_outputs表
        // 4. 标记phase2为dirty
    }
}
```

### 4.3 Phase2整合

```rust
// state/src/model/memories.rs
pub enum Phase2JobClaimOutcome {
    Claimed {
        ownership_token: String,
        input_watermark: i64,      // 输入水位线
    },
    SkippedNotDirty,               // 无需整合
    SkippedRunning,                // 其他worker在处理
}

pub struct Phase2InputSelection {
    pub selected: Vec<Stage1Output>,
    pub previous_selected: Vec<Stage1Output>,
    pub retained_thread_ids: Vec<ThreadId>,
    pub removed: Vec<Stage1OutputRef>,
}
```

**设计亮点：**
1. **分布式锁**: ownership_token防止并发冲突
2. **幂等性**: source_updated_at检查避免重复处理
3. **水位线**: input_watermark追踪处理进度
4. **优雅降级**: 多种Skip原因，避免无效工作

## 5. 上下文压缩实现

### 5.1 压缩触发策略

```rust
// core/src/compact.rs
pub(crate) async fn run_inline_auto_compact_task(
    sess: Arc<Session>,
    turn_context: Arc<TurnContext>,
    initial_context_injection: InitialContextInjection,
) -> CodexResult<()> {
    // 1. 加载compaction prompt模板
    let prompt = turn_context.compact_prompt().to_string();
    
    // 2. 执行compaction task (内部turn)
    run_compact_task_inner(sess, turn_context, input, initial_context_injection).await
}
```

### 5.2 压缩Prompt模板

```markdown
<!-- core/templates/compact/prompt.md -->
You are performing a CONTEXT CHECKPOINT COMPACTION. 
Create a handoff summary for another LLM that will resume the task.

Include:
- Current progress and key decisions made
- Important context, constraints, or user preferences  
- What remains to be done (clear next steps)
- Any critical data, examples, or references needed to continue

Be concise, structured, and focused on helping the next LLM seamlessly continue the work.
```

### 5.3 压缩结果处理

```rust
// core/src/compact.rs
async fn run_compact_task_inner(...) -> CodexResult<()> {
    // 1. 执行LLM调用生成summary
    let summary_suffix = get_last_assistant_message_from_turn(history_items)
        .unwrap_or_default();
    let summary_text = format!("{SUMMARY_PREFIX}\n{summary_suffix}");
    
    // 2. 收集用户消息保留
    let user_messages = collect_user_messages(history_items);
    
    // 3. 构建压缩后的历史
    let mut new_history = build_compacted_history(
        Vec::new(), 
        &user_messages, 
        &summary_text
    );
    
    // 4. 可选：注入初始上下文
    if matches!(initial_context_injection, InitialContextInjection::BeforeLastUserMessage) {
        inject_initial_context(&mut new_history);
    }
    
    // 5. 替换session历史
    sess.replace_history(new_history).await;
    Ok(())
}
```

## 6. Turn-based执行模型

### 6.1 TurnContext

```rust
// core/src/codex.rs
pub struct TurnContext {
    pub sub_id: String,                    // turn唯一ID
    pub config: Arc<Config>,
    pub model_info: ModelInfo,
    pub model_context_window: i64,
    pub sandbox_policy: Arc<SandboxPolicy>,
    pub approval_policy: Arc<ApprovalPolicy>,
    pub cwd: PathBuf,
    pub collaboration_mode: CollaborationMode,
    pub features: Arc<FeatureGate>,
    pub personality: Option<Personality>,
    pub realtime_active: Option<bool>,
    pub turn_timing_state: TurnTimingState,
    pub turn_metadata_state: TurnMetadataState,
}
```

### 6.2 Session协调

```rust
pub struct Session {
    state: Arc<StateRuntime>,
    services: Arc<Services>,
    context_manager: Arc<RwLock<ContextManager>>,
    // ...
}

impl Session {
    /// 执行一个turn
    pub async fn run_turn(&self, turn_context: Arc<TurnContext>, input: UserInput) 
        -> CodexResult<()> 
    {
        // 1. 记录turn开始
        // 2. 检查是否需要压缩
        // 3. 准备prompt (差分更新)
        // 4. 调用LLM
        // 5. 处理响应 (tool calls等)
        // 6. 更新历史
        // 7. 记录turn结束
    }
}
```

## 7. 关键可借鉴点总结

### 7.1 立即可以实施

1. **SQLite WAL模式**: 立即改善并发性能
2. **差分上下文更新**: 减少token消耗
3. **历史归一化**: 避免重复/未匹配项
4. **压缩Prompt模板**: 标准化压缩行为

### 7.2 中期实施

1. **双SQLite设计**: 分离state和logs
2. **日志分区**: 按agent_id分区
3. **sqlx迁移**: 版本化schema
4. **Token估计**: 字节启发式而非tiktoken

### 7.3 长期实施

1. **Stage1/Stage2记忆**: 异步提取管道
2. **分布式任务认领**: ownership_token模式
3. **完整Rust重写**: 验证功能后迁移

## 8. 与V5的对比表

| 特性 | V5 (当前) | Codex | V6目标 |
|------|-----------|-------|--------|
| 数据库 | SQLite (单库) | SQLite (双库) | SQLite (双库+WAL) |
| 上下文更新 | 全量 | 差分 | 差分 |
| 历史管理 | 原始JSONL | 归一化+压缩 | 归一化+压缩 |
| Token估计 | tiktoken | 字节启发式 | 字节启发式 |
| 记忆系统 | 无 | Stage1/Stage2 | Stage1/Stage2 |
| 执行模型 | 事件驱动 | Turn-based | Turn-based |
| 语言 | Python | Rust | Python→Rust |

## 9. 代码参考

### 9.1 关键文件路径

```
codex-rs/
├── core/src/
│   ├── codex.rs                 # Session/TurnContext (308KB)
│   ├── context_manager/
│   │   ├── history.rs           # ContextManager (28KB)
│   │   ├── updates.rs           # 差分更新 (8KB)
│   │   └── normalize.rs         # 历史归一化
│   └── compact.rs               # 上下文压缩 (16KB)
├── state/src/
│   ├── runtime.rs               # StateRuntime (14KB)
│   ├── runtime/
│   │   ├── memories.rs          # 记忆系统 (168KB!)
│   │   ├── threads.rs           # 线程管理 (50KB)
│   │   └── logs.rs              # 日志管理 (65KB)
│   └── model/
│       ├── memories.rs          # 记忆模型
│       └── thread_metadata.rs   # 线程元数据
└── core/templates/
    └── compact/
        ├── prompt.md            # 压缩提示模板
        └── summary_prefix.md    # 摘要前缀
```

### 9.2 关键行数统计

```bash
$ find codex-rs -name "*.rs" | xargs wc -l | tail -1
  约 10万行 Rust代码

$ ls -lh codex-rs/core/src/codex.rs
  308KB (核心逻辑)

$ ls -lh codex-rs/state/src/runtime/memories.rs
  168KB (记忆系统)
```

## 10. 结论

OpenAI Codex的设计体现了**工程化AI系统**的最佳实践：

1. **务实的性能优化**: WAL、分区、差分更新
2. **可靠的异步管道**: Stage1/Stage2、分布式锁
3. **清晰的架构分层**: core/state/protocol分离
4. **细致的边缘处理**: 归一化、压缩、token管理

Simu-Emperor V6应该借鉴这些经过生产验证的模式，同时保持自身的游戏领域特色。
