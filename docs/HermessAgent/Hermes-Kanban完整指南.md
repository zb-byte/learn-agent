# Hermes-Agent Kanban 系统完整指南

> **定位**：面向工程师的深度技术文档，围绕 Hermes Kanban 的架构设计、状态机模型、工作者协议、调度器机制和协作模式展开，关键知识点结合源码实现进行深入分析。
>
> **源码路径**：`/Users/wangzhongbin/Documents/code/fangzhen/hermes-agent`
> **参考源码**：`code-src/src/` 下的 TypeScript 实现提供了同类系统的对照分析
>
> **快速走读**：若需从 CLI/工具入口沿调用链建立端到端心智模型，请先读同目录 [`hermes-kanban-usage-to-implementation.md`](./hermes-kanban-usage-to-implementation.md)。

---

## 目录

- [一、概述与设计哲学](#一概述与设计哲学)
- [二、核心架构深度分析](#二核心架构深度分析)
- [三、状态机模型：七态设计与转换逻辑](#三状态机模型七态设计与转换逻辑)
- [四、工作者协议：从声明到交接的完整链路](#四工作者协议从声明到交接的完整链路)
- [五、调度器：Kanban 的心脏](#五调度器kanban-的心脏)
- [六、任务 vs 运行：可重试的执行模型](#六任务-vs-运行可重试的执行模型)
- [七、依赖引擎与任务拓扑](#七依赖引擎与任务拓扑)
- [八、多看板隔离架构](#八多看板隔离架构)
- [九、协作模式与设计模式](#九协作模式与设计模式)
- [十、可靠性与容错机制](#十可靠性与容错机制)
- [十一、CLI 命令速查](#十一cli-命令速查)
- [十二、最佳实践与反模式](#十二最佳实践与反模式)

---

## 一、概述与设计哲学

### 1.1 Kanban 是什么

Hermes Kanban 是一个持久化的多智能体任务协作看板，基于 SQLite 构建，支持多个命名智能体 profile 在同一个任务队列上协作。它不依赖进程内子智能体集群（如 `delegate_task`），而是通过数据库行作为任务单元，实现跨进程、跨会话的工作编排。

### 1.2 核心设计原则

```text
┌─────────────────────────────────────────────────────────┐
│              Hermes Kanban 设计哲学                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 持久化优先：每个任务是 SQLite 中的一行              │
│  2. 双接口设计：人类用 CLI，智能体用工具                │
│  3. 进程隔离：每个工作者是独立的 OS 进程                │
│  4. 可恢复性：崩溃后可重新声明任务                      │
│  5. 审计追踪：所有操作记录在 task_events 表            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

这五条原则的取舍逻辑：

- **持久化优先** vs **内存队列**：智能体任务可能运行数小时，进程崩溃不能丢任务。SQLite WAL 模式支持并发读写，同时保持单文件部署的简单性。
- **双接口设计**：人类需要 `hermes kanban` 命令进行管理操作；智能体需要 `kanban_*` 工具在运行时调用。两者共享同一 Python 层 (`kanban_db`)，保证状态一致性。
- **进程隔离** vs **线程/协程**：OS 进程隔离意味着一个工作者的内存泄漏不会影响调度器或其他工作者。代价是进程间通信只能通过数据库，但这恰好与持久化设计吻合。

> **对照源码**：TypeScript 实现中的 Task 系统 (`src/tasks/`) 采用类似的统一任务注册表模式。所有任务类型（`LocalShellTask`、`LocalAgentTask`、`RemoteAgentTask` 等）共享 `AppState.tasks` 注册表，通过 `registerTask()` 和 `updateTaskState()` 管理生命周期。任务 ID 格式也遵循前缀 + 随机字符的模式（如 `b1a2b3c4d`、`a1a2b3c4d`）。

### 1.3 Kanban vs delegate_task：为什么需要两套机制

| 维度 | delegate_task | Kanban |
| --- | --- | --- |
| 形态 | RPC 调用，fork 到 join | 持久化消息队列 + 状态机 |
| 父进程 | 阻塞直到子进程返回 | 创建后即释放 |
| 子进程身份 | 匿名子智能体 | 命名 profile，带持久化记忆 |
| 可恢复性 | 无，失败即失败 | 可阻塞、解除阻塞、重新运行 |
| 人工介入 | 不支持 | 随时可评论、解除阻塞 |
| 审计追踪 | 上下文压缩后丢失 | SQLite 中永久保存 |
| 协调方式 | 层级：调用者到被调用者 | 对等：任何 profile 可读写任何任务 |

**深度分析：何时选哪个**

- `delegate_task` 适合：短链路子任务（< 5 分钟）、同步依赖（必须等结果才能继续）、不需要人工介入。
- Kanban 适合：长时间任务、多角色协作、可能失败需要重试、需要人工审批、跨会话持续工作。

一句话：`delegate_task` 是函数调用；Kanban 是工作队列。

> **对照源码**：TypeScript 实现中，`AgentTool`（`src/tools/AgentTool/`）实现了 `delegate_task` 语义——同步子智能体阻塞父进程，异步子智能体通过通知队列回调。而 Kanban 提供的是更上层的持久化编排能力。两者不在同一抽象层。

---

## 二、核心架构深度分析

### 2.1 双接口架构

```text
┌──────────────────────────────────────────────────────────┐
│                    双接口架构                             │
└──────────────────────────────────────────────────────────┘

人类/脚本接口                    智能体接口
     │                              │
     ▼                              ▼
┌─────────────┐              ┌──────────────┐
│ hermes      │              │ kanban_show  │
│ kanban      │              │ kanban_list  │
│ create      │              │ kanban_      │
│ list        │              │ complete     │
│ show        │              │ kanban_block │
│ complete    │              │ kanban_      │
│ ...         │              │ heartbeat    │
└──────┬──────┘              └──────┬───────┘
       │                            │
       └────────────┬───────────────┘
                    ▼
            ┌───────────────┐
            │  kanban_db    │
            │  (Python 层)  │
            └───────┬───────┘
                    ▼
            ┌───────────────┐
            │ ~/.hermes/    │
            │ kanban.db     │
            │ (SQLite WAL)  │
            └───────────────┘
```

**关键设计决策：为什么不直接暴露 SQL**

双接口通过 `kanban_db` Python 层统一路由，而不是让 CLI 和工具各自直接操作数据库。原因有三：

1. **状态转换约束**：任务状态不是自由切换的（如 `done` 不能直接变回 `running`），Python 层封装了状态机校验。
2. **事件记录**：每次状态变化都要写入 `task_events` 表，这是审计追踪的基础。散落在各处的 SQL 无法保证这一点。
3. **并发安全**：SQLite WAL 模式支持并发读，但写入仍需串行化。Python 层通过事务保证原子性。

### 2.2 SQLite WAL 模式：为什么适合这个场景

```text
                    SQLite WAL 模式
    ════════════════════════════════════════

    读操作（不阻塞）          写操作（串行化）
         │                       │
         ▼                       ▼
    ┌──────────┐          ┌──────────────┐
    │ WAL 文件 │          │ 检查点合并   │
    │ 快照读   │          │ (自动/手动)  │
    └──────────┘          └──────────────┘

    优势：
    - 读不阻塞写，写不阻塞读
    - 适合"多工作者频繁读 + 偶尔写"的模式
    - 单文件部署，无需额外数据库服务
```

**深度分析：WAL 的 trade-off**

WAL 模式在 Kanban 场景下的性能特征：

- **读多写少**：工作者频繁调用 `kanban_show()` 读取任务上下文，但状态变更（complete/block/heartbeat）频率较低。WAL 的"读不阻塞写"正好匹配。
- **检查点开销**：WAL 文件增长到一定阈值会触发检查点（checkpoint），将 WAL 合并回主数据库。在任务量极大时，这可能导致短暂延迟。Kanban 通过 `PRAGMA wal_autocheckpoint` 控制阈值。
- **跨平台兼容**：SQLite WAL 在 NFS 等网络文件系统上不可靠。Kanban 设计为本地使用（`~/.hermes/`），避免了这个问题。

### 2.3 数据库表结构

```sql
-- 核心三表设计

-- 任务表：逻辑工作单元
CREATE TABLE tasks (
    id TEXT PRIMARY KEY,           -- t_ 前缀 + 随机字符
    title TEXT NOT NULL,
    body TEXT,
    assignee TEXT,                 -- profile 名称
    status TEXT DEFAULT 'todo',    -- 状态机七态
    priority INTEGER DEFAULT 2,
    current_run_id INTEGER,        -- 指向当前运行
    workspace_type TEXT,           -- scratch | dir | worktree
    workspace_path TEXT,
    max_retries INTEGER,
    created_at REAL,
    updated_at REAL
);

-- 运行表：每次执行尝试
CREATE TABLE task_runs (
    id INTEGER PRIMARY KEY,
    task_id TEXT REFERENCES tasks(id),
    profile TEXT,                  -- 执行者 profile
    status TEXT DEFAULT 'running', -- running | completed | blocked | crashed
    summary TEXT,                  -- 结构化交接摘要
    metadata TEXT,                 -- JSON 格式的交接元数据
    error TEXT,
    started_at REAL,
    finished_at REAL
);

-- 事件表：审计追踪
CREATE TABLE task_events (
    id INTEGER PRIMARY KEY,
    task_id TEXT REFERENCES tasks(id),
    run_id INTEGER,
    kind TEXT,                     -- 事件类型
    data TEXT,                     -- JSON
    created_at REAL
);
```

**设计分析**：

- **tasks + task_runs 分离**：允许同一任务多次尝试，每次尝试保留完整的执行上下文。工作者重试时可以通过 `worker_context.prior_attempts` 获取先前失败的原因。
- **task_events 作为事件溯源**：所有状态变更都记录事件，支持 `hermes kanban watch` 实时监控和 `hermes kanban tail` 事件回放。
- **JSON metadata 字段**：结构化交接的核心载体。不同任务类型的 metadata schema 可以自由扩展，不需要改表结构。

---

## 三、状态机模型：七态设计与转换逻辑

### 3.1 状态转换图

```text
                    任务状态转换流程
    ════════════════════════════════════════════════════

    创建
     │
     ▼
┌─────────┐  手动/LLM 规范化   ┌──────┐
│ triage  │──────────────────▶│ todo │
└─────────┘                   └───┬──┘
                                  │
                                  │ 所有父任务完成（依赖引擎提升）
                                  ▼
                              ┌───────┐
                              │ ready │◀────────┐
                              └───┬───┘         │
                                  │             │
                    调度器声明    │             │ 解除阻塞
                                  ▼             │
                              ┌─────────┐       │
                              │ running │       │
                              └────┬────┘       │
                                   │            │
                ┌──────────────────┼────────────┼─────────┐
                │                  │            │         │
                ▼                  ▼            ▼         ▼
           ┌─────────┐        ┌────────┐  ┌─────────┐ ┌──────────┐
           │  done   │        │blocked │  │scheduled│ │ crashed/ │
           └─────────┘        └────────┘  └─────────┘ │timed_out │
                              (需人工输入) (等待时间)  └────┬─────┘
                                   │          │            │
                                   └──────────┴────────────┘
                                              │
                                              └─────▶ 回到 ready
```

### 3.2 七态详解与转换约束

| 状态 | 图标 | 含义 | 谁可以设置 | 关键约束 |
| --- | --- | --- | --- | --- |
| `triage` | - | 原始想法，等待规范化 | 创建时 `--triage` | 只能转换到 `todo` |
| `todo` | ◻ | 已创建但等待依赖 | 系统或人工 | 依赖满足后由引擎提升到 `ready` |
| `scheduled` | ⏱ | 等待时间触发 | `kanban_schedule` | 到期后自动提升到 `ready` |
| `ready` | ▶ | 已分配且可运行 | 依赖引擎 / 解除阻塞 | 只能由调度器声明为 `running` |
| `running` | ● | 工作者正在执行 | 调度器声明 | 必须有活跃 run 记录 |
| `blocked` | ⊘ | 需要人工输入 | `kanban_block` | 必须提供 `reason` |
| `done` | ✓ | 已完成 | `kanban_complete` | 必须提供 `summary` |
| `archived` | - | 隐藏但未删除 | `hermes kanban archive` | 终态，不可恢复 |

**深度分析：状态机的非显而易见之处**

1. **`triage` 的意义**：不是所有任务一开始就有明确规范。`triage` 允许快速捕获想法，后续再由人类或 LLM 规范化为 `todo`。这是"收件箱"模式。

2. **`todo` vs `ready` 的区别**：`todo` 表示"依赖未满足"，`ready` 表示"可以立即运行"。这个区分使得依赖引擎可以在一次 tick 中批量提升多个任务，而不是每次提升一个。

3. **`scheduled` 的特殊地位**：与 `blocked` 不同，`scheduled` 不需要人工介入。它是一个纯时间触发状态，到时间后自动提升。适用场景：定时部署、定期报告生成。

4. **`running` 的原子性声明**：调度器声明任务为 `running` 是一个原子操作（SQL 事务）。在高并发下，多个调度器实例不会重复声明同一任务。这是通过 SQLite 的事务隔离实现的。

> **对照源码**：TypeScript 实现中的任务状态更简单（`pending` → `in_progress` → `completed`），但 AgentTool 的 `runAgent()` 函数实现了类似的声明-执行-完成/失败模式。异步子智能体的 `abortController` 机制对应了 Kanban 的声明超时回收。

### 3.3 状态转换的合法矩阵

```text
从 \ 到     triage  todo  scheduled  ready  running  blocked  done
triage        -      ✓       -         -       -        -       -
todo          -      -       ✓         ✓*      -        -       -
scheduled     -      -       -         ✓**     -        -       -
ready         -      -       -         -       ✓        -       -
running       -      -       -         -       -        ✓       ✓
blocked       -      -       -         ✓***    -        -       -
done          -      -       -         -       -        -       -

*  由依赖引擎自动提升
** 由时间触发器自动提升
*** 由人工 unblock 操作触发
```

这个矩阵的关键洞察：**`running` 是唯一的分叉点**——从 `running` 只能走向 `done`（成功）或 `blocked`（需要帮助）。不存在 `running` → `todo` 或 `running` → `ready` 的直接转换。这保证了工作者要么完成工作，要么明确报告问题。

---

## 四、工作者协议：从声明到交接的完整链路

### 4.1 环境变量注入

调度器生成工作者时，通过环境变量传递任务上下文：

```text
HERMES_KANBAN_TASK=t_abcd       # 任务 ID
HERMES_KANBAN_RUN_ID=42          # 运行 ID
HERMES_KANBAN_WORKSPACE=/path    # 工作空间路径
HERMES_KANBAN_BOARD=board-slug   # 看板标识
```

**为什么用环境变量而不是命令行参数**

1. **Profile 系统兼容**：Hermes 的 profile 系统通过环境变量配置子进程，Kanban 遵循同一模式。
2. **工具激活**：Kanban 工具（`kanban_show` 等）通过检测 `HERMES_KANBAN_TASK` 环境变量来决定是否激活。如果变量不存在，这些工具不会被注册到工具集中。
3. **进程树传递**：环境变量在 fork/exec 时自动继承，即使工作者内部再启动子进程，也能获取任务上下文。

> **对照源码**：TypeScript 实现中，`AgentTool` 的 `createSubagentContext()` 函数通过参数对象而非环境变量传递上下文。但 `runAgent.ts` 中的 `AsyncLocalStorage` 模式实现了类似的目的——为子智能体提供隔离的执行上下文。

### 4.2 工作者工具集详解

```python
# 1. 读取任务上下文
kanban_show()
# 返回：
# {
#   "id": "t_abc123",
#   "title": "实现用户认证 API",
#   "body": "POST /register, POST /login, POST /refresh",
#   "worker_context": "...",  # 包含父任务结果、先前尝试、评论
#   "parents": ["t_xyz789"],
#   "status": "running"
# }

# 2. 执行工作
# 工作者使用终端和文件工具编写代码、运行测试、提交更改。

# 3. 长时间操作时发送心跳
kanban_heartbeat(note="已完成 4/8 个文件的迁移")

# 4. 完成时提供结构化交接
kanban_complete(
    summary="实现了基于令牌桶的限流器，按 user_id 键控，IP 回退，所有测试通过",
    metadata={
        "changed_files": ["limiter.py", "tests/test_limiter.py"],
        "tests_run": 14,
        "decisions": ["选择令牌桶而非漏桶", "每用户 100 req/min"],
    },
)

# 或者，如果遇到阻塞
kanban_block(reason="需要决定速率限制键：user_id 还是 IP？")
```

**深度分析：`worker_context` 的构建逻辑**

`kanban_show()` 返回的 `worker_context` 不是简单的字段拼接，而是精心构建的上下文传递机制：

```text
worker_context 的组成
═══════════════════════

1. parents[]            # 父任务的 summary + metadata
   └─ 流水线模式下的上游结果

2. prior_attempts[]     # 先前运行的 outcome + error + summary
   └─ 重试时知道上一次为什么失败

3. comments[]           # 人工评论
   └─ 阻塞解除后的指导信息

4. siblings[]           # 同级任务的 summary
   └─ 扇出模式下的并行结果参考
```

这个设计解决了一个核心问题：**工作者如何在不读取完整历史的情况下获取足够上下文**。答案是通过结构化摘要而非原始日志传递信息。

### 4.3 工作者生命周期

```text
调度器 tick
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 1. 回收过期声明（TTL 超时）                          │
│ 2. 检测崩溃的工作者（PID 消失）                      │
│ 3. 提升就绪任务（所有父任务完成）                    │
│ 4. 原子性声明就绪任务                               │
│ 5. 生成工作者进程                                   │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 工作者进程启动                                       │
│   HERMES_KANBAN_TASK=t_abc123                       │
│   HERMES_KANBAN_RUN_ID=42                           │
│   HERMES_KANBAN_WORKSPACE=/path/to/workspace        │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 工作者循环                                           │
│   1. kanban_show() 读取上下文                       │
│   2. cd $HERMES_KANBAN_WORKSPACE                    │
│   3. 执行工作                                       │
│   4. 每小时 kanban_heartbeat()（长时间操作）        │
│   5. kanban_complete() 或 kanban_block()            │
└─────────────────────────────────────────────────────┘
    │
    ▼
任务转换到 done/blocked
```

### 4.4 协议违规检测

这是工作者协议中最容易被忽视但最关键的部分：

```text
工作者退出状态       任务状态       调度器行为
═══════════════════════════════════════════════════

exit 0              done          正常完成（已调用 kanban_complete）
exit 0              running       ❌ protocol_violation → 自动 blocked
exit 0              blocked       正常阻塞（已调用 kanban_block）
exit non-0          running       crashed → 可重试
exit non-0          blocked       保持 blocked
信号杀死            running       调度器检测 PID 消失 → 可重试
超时无心跳          running       调度器回收 → 可重试
```

**为什么 exit 0 + running 是违规**：工作者正常退出（exit 0）意味着它认为自己完成了，但没有调用 `kanban_complete` 或 `kanban_block`。这通常是因为：
1. 工作者的 LLM 跳过了交接步骤
2. 工作者遇到未处理异常但被静默捕获

调度器将这类任务自动设为 `blocked` 而非 `done`，是为了防止"看起来完成了但实际没有"的情况。

> **对照源码**：TypeScript 实现中，`AgentTool` 的异步子智能体通过 `enqueueAgentNotification()` 通知父智能体完成。如果子智能体异常终止，通知队列不会收到完成事件，父智能体通过超时检测发现问题。

---

## 五、调度器：Kanban 的心脏

### 5.1 调度器 tick 周期

调度器以固定间隔运行 tick，每次 tick 执行以下步骤（严格按序）：

```text
调度器 tick 执行顺序
═════════════════════

Step 1: 回收过期声明
    │   扫描 running 状态的任务
    │   检查 last_heartbeat 时间
    │   超过 TTL → 回收到 ready
    │
Step 2: 检测崩溃工作者
    │   检查每个 running 任务的 PID
    │   PID 不存在 → 记录 crashed 事件
    │   视重试次数决定 → ready 或 blocked
    │
Step 3: 依赖引擎提升
    │   扫描所有 todo 任务
    │   检查父任务是否全部 done
    │   满足 → 提升为 ready
    │
Step 4: 时间触发提升
    │   扫描所有 scheduled 任务
    │   检查触发时间是否已到
    │   到期 → 提升为 ready
    │
Step 5: 声明就绪任务
    │   按 priority 排序 ready 任务
    │   按 profile 匹配 assignee
    │   原子性 UPDATE status = 'running'
    │
Step 6: 生成工作者进程
        为每个声明的任务启动进程
        设置环境变量
        记录 run 记录
```

**深度分析：为什么步骤顺序很重要**

- **Step 1-2 在 Step 5 之前**：先回收再声明，避免"声明了新任务但还有未回收的旧声明"导致的资源浪费。
- **Step 3-4 在 Step 5 之前**：先提升再声明，确保新变为 ready 的任务在本次 tick 就能被声明，而不是等到下一次。
- **Step 5 的原子性**：使用 SQL 事务 + 行锁保证同一时刻只有一个调度器实例声明特定任务。这对于多调度器部署至关重要。

> **对照源码**：TypeScript 实现没有传统的调度器，而是通过 `QueryEngine` 的事件驱动模式隐式调度。`toolOrchestration.ts` 中的工具批处理（只读工具并发、写工具串行）体现了类似的"先分析再执行"思想。

### 5.2 声明的原子性

```sql
-- 原子性声明（伪代码）
BEGIN TRANSACTION;

-- 检查任务仍然为 ready
SELECT status FROM tasks WHERE id = 't_abc' FOR UPDATE;
-- status == 'ready' → 继续

-- 更新状态为 running
UPDATE tasks SET
    status = 'running',
    current_run_id = <new_run_id>,
    updated_at = <now>
WHERE id = 't_abc';

-- 创建 run 记录
INSERT INTO task_runs (task_id, profile, status, started_at)
VALUES ('t_abc', 'backend-dev', 'running', <now>);

COMMIT;
```

如果两个调度器同时尝试声明同一任务，SQLite 的排他锁保证只有一个成功，另一个会收到 busy 错误并重试。

### 5.3 Profile 匹配与资源控制

```text
调度器声明任务时的匹配逻辑
═════════════════════════════

任务 assignee 字段
    │
    ▼
是否匹配某个活跃 profile？
    │
    ├── 是 → 该 profile 的并发槽是否有空位？
    │         │
    │         ├── 是 → 声明任务，生成工作者
    │         └── 否 → 等待下次 tick
    │
    └── 否 → 任务停留在 ready（无匹配 profile）
```

每个 profile 可以配置最大并发工作者数。这防止单个 profile（如 `backend-dev`）同时运行太多任务导致资源争抢。

---

## 六、任务 vs 运行：可重试的执行模型

### 6.1 核心分离

```text
┌──────────────────────────────────────────────────────┐
│              任务 vs 运行                             │
├──────────────────────────────────────────────────────┤
│                                                      │
│  任务 (Task)：逻辑工作单元                           │
│    - 一行在 tasks 表中                               │
│    - 有标题、正文、分配者、状态                      │
│    - 可以被多次尝试                                  │
│    - 存活时间：从创建到归档                          │
│                                                      │
│  运行 (Run)：一次执行尝试                            │
│    - 一行在 task_runs 表中                           │
│    - 有 profile、开始时间、结束时间、结果            │
│    - 有 summary、metadata、error 字段                │
│    - 存活时间：从声明到完成/崩溃                     │
│                                                      │
│  关系：tasks.current_run_id ──▶ task_runs.id        │
│  关系：task_runs.task_id ──▶ tasks.id (多对一)      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 6.2 重试链与上下文传递

```text
任务 t_impl456 的重试历史
════════════════════════════

Run 1: running → blocked
  outcome: blocked
  error: "审查：缺少密码强度检查"
  summary: null
       │
       ▼  人工解除阻塞
Run 2: running → done
  outcome: completed
  error: null
  summary: "修复了审查中的两个问题"
  metadata: {"review_iteration": 2, ...}
```

**深度分析：为什么需要 Run 而不是直接在 Task 上记录**

1. **重试不丢失历史**：如果直接在 Task 上覆盖结果，重试后先前的失败原因就丢失了。Run 分离使得每次尝试的完整上下文都被保留。
2. **`worker_context.prior_attempts`**：新工作者启动时，`kanban_show()` 会返回先前所有 run 的摘要。工作者能精确知道"上次为什么失败"，而不是盲目重试。
3. **熔断器计数**：连续失败次数基于 run 记录统计。达到阈值后任务自动 blocked，而不是无限重试。

### 6.3 结构化交接：Kanban 的核心价值

三阶段流水线的交接示例：

```python
# 阶段 1：PM 工作者完成规范
kanban_complete(
    summary=(
        "规范已批准；POST /forgot-password 发送邮件，"
        "GET /reset/:token 渲染表单，POST /reset 应用新密码"
    ),
    metadata={
        "acceptance": [
            "过期令牌返回 410",
            "重用最近 3 个密码返回 400 并提示",
            "成功重置使所有活动会话失效",
        ]
    },
)
# 规范任务完成后，实现任务自动从 todo 提升到 ready。
```

```python
# 阶段 2：工程师工作者读取父任务结果
kanban_show()
# 返回的 worker_context 包含：
# {
#   "parents": [{
#     "id": "t_spec123",
#     "summary": "规范已批准；POST /forgot-password...",
#     "metadata": {"acceptance": [...]}
#   }]
# }

kanban_complete(
    summary="添加了 zxcvbn 强度检查，重置令牌现在是一次性的",
    metadata={
        "changed_files": [
            "auth/reset.py",
            "auth/tests/test_reset.py",
            "migrations/003_single_use_reset_tokens.sql",
        ],
        "tests_run": 11,
        "review_iteration": 1,
    },
)
```

```python
# 阶段 3：审查者工作者读取实现结果
kanban_show()
# worker_context 现在包含实现任务的 summary 和 metadata。
# 审查者知道更改了哪些文件、运行了多少测试。
```

**交接的四个关键问题**：

| 问题 | 对应 metadata 字段 | 为什么重要 |
| --- | --- | --- |
| 更改了什么？ | `changed_files` | 下游工作者知道该审查/测试哪些文件 |
| 如何验证的？ | `verification` | 不需要重新猜测验证命令 |
| 失败了怎么办？ | `retry_notes` | 重试时不用重新排查 |
| 故意留下了什么风险？ | `residual_risk` | 防止下游假设一切都安全 |

> **对照源码**：TypeScript 实现中，`AgentTool` 的异步子智能体通过 `task-notification` XML 格式传递结果，包含 `<result>` 标签。虽然格式不同，但核心思想一致：结构化的结果传递优于原始输出复制。

---

## 七、依赖引擎与任务拓扑

### 7.1 父子关系模型

```text
依赖关系示例：构建认证系统
═══════════════════════════

     t_schema (数据库 Schema)
         │
         ├── t_api (实现 API) ──────┐
         │                          │
         └── t_docs (编写文档)      │
                                    │
                              t_test (集成测试)
                                    │
                              t_deploy (部署)
```

```bash
# 创建依赖链
SCHEMA=$(hermes kanban create "设计认证数据库 Schema" --assignee architect --json | jq -r .id)
API=$(hermes kanban create "实现认证 API" --assignee backend-dev --parent "$SCHEMA" --json | jq -r .id)
DOCS=$(hermes kanban create "编写认证文档" --assignee tech-writer --parent "$SCHEMA" --json | jq -r .id)
TEST=$(hermes kanban create "认证集成测试" --assignee qa --parent "$API" --json | jq -r .id)
DEPLOY=$(hermes kanban create "部署认证服务" --assignee devops --parent "$TEST" --json | jq -r .id)
```

### 7.2 依赖提升算法

```text
依赖引擎每次 tick 的提升逻辑
═════════════════════════════

FOR each task WHERE status = 'todo':
    parents = get_parent_tasks(task)
    IF all parents have status = 'done':
        UPDATE task SET status = 'ready'
        INSERT task_event (kind='promoted', data={'reason': 'all_parents_done'})
```

**深度分析：为什么是"全部父任务完成"而不是"任一父任务完成"**

这是 DAG（有向无环图）依赖的标准语义——一个任务只有在所有依赖都满足后才能开始。如果需要"任一完成即可"的语义，应该创建多个独立任务而非一个多父任务。

**防止环形依赖**：Kanban 在 `link` 操作时检查是否会产生环。如果 `t_A → t_B → t_C → t_A`，link 操作会被拒绝。

### 7.3 依赖管理的 CLI 操作

```bash
# 创建带父任务的任务
hermes kanban create "实现 API" --assignee dev --parent t_schema123

# 事后添加依赖
hermes kanban link t_parent123 t_child456

# 移除依赖
hermes kanban unlink t_parent123 t_child456
```

---

## 八、多看板隔离架构

### 8.1 目录结构

```text
~/.hermes/
├── kanban.db                    # 默认看板，向后兼容
└── kanban/
    ├── current                  # 当前活动看板的 slug
    └── boards/
        ├── default/             # 默认看板目录
        │   ├── kanban.db
        │   ├── workspaces/
        │   └── logs/
        ├── atm10-server/        # 自定义看板
        │   ├── board.json       # 元数据
        │   ├── kanban.db
        │   ├── workspaces/
        │   └── logs/
        └── _archived/           # 归档的看板
            └── old-project-20260520/
```

### 8.2 隔离规则与看板解析

```text
看板解析顺序（优先级从高到低）
═══════════════════════════════

1. --board 参数（命令行显式指定）
2. HERMES_KANBAN_BOARD 环境变量
3. ~/.hermes/kanban/current 文件
4. default（兜底）
```

**深度分析：为什么需要多看板**

1. **项目隔离**：不同项目的任务不应该混在一起。ATM 服务器的运维任务和公司官网的开发任务没有依赖关系。
2. **权限边界**：不同团队使用不同看板，profile 的可见性可以按看板隔离。
3. **归档清理**：项目结束后归档整个看板，而不是逐个归档任务。

> **对照源码**：TypeScript 实现中的 `Worktree` 机制（`EnterWorktreeTool`/`ExitWorktreeTool`）实现了类似的隔离思想——通过 git worktree 为每个任务创建独立的文件系统视图，避免工作区冲突。

### 8.3 看板管理命令

```bash
# 列出所有看板
hermes kanban boards list

# 创建新看板
hermes kanban boards create atm10-server \
    --name "ATM10 服务器" \
    --description "Minecraft 模组服务器运维" \
    --icon 🎮

# 切换看板
hermes kanban boards switch atm10-server

# 在特定看板上操作，不切换当前看板
hermes kanban --board atm10-server list
hermes kanban --board atm10-server create "重启服务器" --assignee ops
```

---

## 九、协作模式与设计模式

### 9.1 模式 1：扇出（Fan-out）——并行分发后聚合

```text
                    并行研究
    ════════════════════════════════════════

         ┌─────────────────────┐
         │  研究 ICP 融资格局   │
         │  (orchestrator)     │
         └──────────┬──────────┘
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
    ┌────────┐ ┌────────┐ ┌────────┐
    │研究者 A│ │研究者 B│ │研究者 C│
    │北美角度│ │欧洲角度│ │亚洲角度│
    └────────┘ └────────┘ └────────┘
         │          │          │
         └──────────┼──────────┘
                    ▼
         ┌─────────────────────┐
         │  综合报告 (writer)   │
         └─────────────────────┘
```

```bash
R1=$(hermes kanban create "研究 ICP 融资 — 北美角度" \
    --assignee researcher-a --json | jq -r .id)
R2=$(hermes kanban create "研究 ICP 融资 — 欧洲角度" \
    --assignee researcher-b --json | jq -r .id)
R3=$(hermes kanban create "研究 ICP 融资 — 亚洲角度" \
    --assignee researcher-c --json | jq -r .id)

hermes kanban create "综合研究结果为启动简报" \
    --assignee writer \
    --parent "$R1" --parent "$R2" --parent "$R3" \
    --body "一页纸，300 字，中立语气"
```

**关键洞察**：扇出模式中，子任务是独立的（无 `--parent` 关系），只有聚合任务依赖所有子任务。依赖引擎确保三个研究都完成后才启动综合任务。

### 9.2 模式 2：流水线（Pipeline）——串行角色链

```text
    角色链：侦察员 → 编辑 → 作者
    ════════════════════════════════

    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │ 侦察员   │─────▶│  编辑    │─────▶│  作者    │
    │ 收集素材 │      │ 审查结构 │      │ 最终稿   │
    └──────────┘      └──────────┘      └──────────┘
```

```bash
SCOUT=$(hermes kanban create "收集每日新闻素材" \
    --assignee scout --json | jq -r .id)
EDIT=$(hermes kanban create "审查新闻结构" \
    --assignee editor --parent "$SCOUT" --json | jq -r .id)
hermes kanban create "撰写每日简报" \
    --assignee writer --parent "$EDIT"
```

**关键洞察**：流水线是线性 DAG，每个任务最多一个父任务。结构化交接在相邻任务间传递：侦察员的 `metadata` 包含素材链接，编辑的 `summary` 包含修改意见，作者据此生成最终稿。

### 9.3 模式 3：投票/法定人数（Voting/Quorum）

```text
    三个研究者 → 一个审查者选择最佳
    ════════════════════════════════════

    ┌────────┐  ┌────────┐  ┌────────┐
    │研究者 1│  │研究者 2│  │研究者 3│
    └───┬────┘  └───┬────┘  └───┬────┘
        │           │           │
        └───────────┼───────────┘
                    ▼
            ┌───────────────┐
            │  审查者/聚合器 │
            │  选择最佳方案  │
            └───────────────┘
```

适用场景：同一问题需要多个 profile 独立给出方案，然后由审查者选择最佳方案、综合共同结论或指出分歧。与扇出模式的区别在于：扇出是"分工"，投票是"竞争"。

### 9.4 模式 4：长期日志（Long-running Journal）

```bash
hermes kanban create "每日运营审查" \
    --assignee ops \
    --workspace dir:/Users/me/ops-journal/ \
    --tenant ops-team
```

配合 cron 定时触发：

```cron
0 9 * * * hermes kanban create "每日运营审查" --assignee ops --workspace dir:/Users/me/ops-journal/ --tenant ops-team
```

> **对照源码**：TypeScript 实现中的 `CronCreate`/`ScheduleCronTool` 实现了类似的定时触发机制。`durable: true` 参数将任务持久化到 `.claude/scheduled_tasks.json`，跨会话存活。7 天自动过期机制防止任务无限累积。

### 9.5 模式 5：人工介入（Human-in-the-loop）

```python
# 工作者遇到模糊决策时阻塞任务
kanban_block(reason="需要决定：使用 Redis 还是 Memcached 作为缓存？")
```

```bash
# 人类评论并解除阻塞
hermes kanban comment t_abc "使用 Redis — 我们已经在生产环境中运行它"
hermes kanban unblock t_abc
```

工作者重新生成后会读取评论，继续工作。

**深度分析：人工介入的几种模式**

1. **决策点阻塞**：工作者遇到需要人类判断的分支（如技术选型），调用 `kanban_block(reason=...)`。
2. **审批阻塞**：高风险操作（如生产部署）可以设置 `--max-retries 0`，第一次执行后必须人工确认。
3. **信息补充**：工作者缺少关键信息（如 API 密钥），阻塞等待人类提供。

> **对照源码**：TypeScript 实现中的 `AskUserQuestionTool` 实现了同步的用户交互——Agent 在执行中暂停等待用户回答。Kanban 的 `block` 则是异步的——工作者退出，人类在方便时解除阻塞。

---

## 十、可靠性与容错机制

### 10.1 心跳机制

```python
for i, item in enumerate(large_dataset):
    process(item)
    if i % 1000 == 0:
        kanban_heartbeat(note=f"已处理 {i}/{len(large_dataset)} 项")
```

**心跳的完整机制**：

```text
心跳触发条件               调度器行为
═══════════════════════════════════════════════════

距离上次心跳 > 1 小时       标记为 stale
且距离声明 > TTL（默认 4h）  回收任务 → ready（可被重新声明）
                            记录 task_event (kind='reclaimed')

距离上次心跳 < 1 小时       保持 running
且工作者 PID 存在

距离上次心跳 < 1 小时       保持 running
但工作者 PID 消失           → 转为 Step 2（崩溃检测）
```

**为什么心跳间隔是 1 小时而不是更短**：智能体执行复杂任务时，中间可能长时间没有数据库写入（比如在运行大型测试套件）。1 小时是一个平衡点——足够长以避免不必要的心跳开销，足够短以检测真正卡死的任务。

### 10.2 熔断器（Circuit Breaker）

```text
熔断器逻辑
═══════════

连续失败次数 < max_retries → 自动重试（任务回到 ready）
连续失败次数 ≥ max_retries → 自动阻塞（需要人工介入）

默认值：
  - failure_limit: 2（全局配置）
  - max_retries: 继承全局值
  - 高风险任务可覆盖：--max-retries 1
```

```bash
# 高风险任务：第一次失败就阻塞
hermes kanban create "部署到生产环境" \
    --assignee deploy-bot \
    --max-retries 1
```

**深度分析：熔断器与重试的交互**

熔断器计数的是**连续**失败次数。如果一次重试成功（即使之前失败过），计数器归零。这意味着：

```text
Run 1: failed  → 计数 = 1
Run 2: failed  → 计数 = 2 → 达到 max_retries → blocked
Run 3: (人工 unblock 后) failed → 计数 = 1（人工介入后重置？取决于实现）
```

实际上，人工 unblock 后的失败计数是否重置取决于 `max_retries` 的语义。如果 `max_retries = 2` 表示"自动重试 2 次后阻塞"，那么人工 unblock 后重试的失败不会再次触发熔断（因为已经经过了人工确认）。

### 10.3 崩溃恢复

```text
崩溃恢复流程
════════════

工作者进程崩溃
    │
    ▼
调度器 tick 检测 PID 消失
    │
    ▼
更新 run 记录：status = 'crashed'
    │
    ▼
检查连续失败次数
    │
    ├── < max_retries → 任务回到 ready（下次 tick 重新声明）
    │                    记录 task_event (kind='crash_retry')
    │
    └── ≥ max_retries → 任务变为 blocked
                         记录 task_event (kind='crash_blocked')
```

**关键：可恢复性是 Kanban 的核心优势**。在 `delegate_task` 模式下，进程崩溃意味着整个子任务丢失。在 Kanban 模式下，崩溃只是多了一次 run 记录，下次 tick 自动重试，且新工作者可以通过 `prior_attempts` 知道上次做了什么。

### 10.4 工作空间隔离与清理

| 工作空间类型 | 何时使用 | 生命周期 | 隔离级别 |
| --- | --- | --- | --- |
| `scratch` | 一次性任务 | 任务完成后清理 | 完全隔离 |
| `dir:<path>` | 共享目录 | 手动管理 | 无隔离（共享写入） |
| `worktree` | 编码任务 | 任务完成后清理 worktree | Git 级隔离 |

```bash
# Scratch：一次性处理
hermes kanban create "转换 CSV 到 JSON" --assignee converter

# 共享目录：持续写入
hermes kanban create "每日日志条目" \
    --assignee logger \
    --workspace dir:/Users/me/journal/

# Git worktree：编码任务隔离
hermes kanban create "修复认证 bug" \
    --assignee backend-dev \
    --workspace worktree
```

> **对照源码**：TypeScript 实现中的 `EnterWorktreeTool`/`ExitWorktreeTool` 实现了 git worktree 管理。worktree 创建在 `.claude/worktrees/` 目录下，任务完成后可选择保留或删除。

---

## 十一、CLI 命令速查

### 11.1 基础操作

```bash
# 初始化看板
hermes kanban init

# 创建任务
hermes kanban create "实现用户认证" \
    --assignee backend-dev \
    --body "JWT + refresh tokens" \
    --priority 2

# 列出任务
hermes kanban list
hermes kanban list --mine
hermes kanban list --status running
hermes kanban list --assignee ops

# 查看任务详情
hermes kanban show t_abc123

# 查看任务的尝试历史
hermes kanban runs t_abc123
```

### 11.2 任务生命周期

```bash
# 分配任务
hermes kanban assign t_abc123 backend-dev

# 添加评论
hermes kanban comment t_abc123 "记得添加速率限制"

# 完成任务
hermes kanban complete t_abc123 \
    --summary "实现了 JWT 认证，所有测试通过" \
    --metadata '{"tests_run": 24, "changed_files": ["auth.py"]}'

# 阻塞任务
hermes kanban block t_abc123 "需要 API 密钥"

# 解除阻塞
hermes kanban unblock t_abc123

# 归档任务
hermes kanban archive t_abc123
```

### 11.3 依赖管理

```bash
# 创建带父任务的任务
hermes kanban create "实现 API" \
    --assignee dev \
    --parent t_schema123

# 事后添加依赖
hermes kanban link t_parent123 t_child456

# 移除依赖
hermes kanban unlink t_parent123 t_child456
```

### 11.4 监控与调试

```bash
# 查看看板统计
hermes kanban stats

# 实时监控事件
hermes kanban watch
hermes kanban watch --kinds completed,blocked,gave_up

# 跟踪单个任务的事件
hermes kanban tail t_abc123

# 查看工作者日志
hermes kanban log t_abc123

# 手动触发调度器
hermes kanban dispatch
hermes kanban dispatch --dry-run
```

---

## 十二、最佳实践与反模式

### 12.1 结构化交接模板

推荐的 `metadata` 结构适用于工程和审查任务：

```python
kanban_complete(
    summary="简短的人类可读总结",
    metadata={
        "changed_files": ["path/to/file.py"],
        "verification": ["pytest tests/test_foo.py -q"],
        "dependencies": ["父任务 ID 或外部问题"],
        "blocked_reason": None,
        "retry_notes": "先前失败的原因（如果是重试）",
        "residual_risk": ["未测试或仍需人工审查的内容"],
    },
)
```

### 12.2 心跳策略

对于超过 1 小时的长时间运行任务：

```python
for i, item in enumerate(large_dataset):
    process(item)
    if i % 1000 == 0:
        kanban_heartbeat(note=f"已处理 {i}/{len(large_dataset)} 项")
```

### 12.3 反模式

| 反模式 | 问题 | 正确做法 |
| --- | --- | --- |
| 工作者 exit 0 但不调用 complete/block | 调度器标记 protocol_violation | 总是以 complete 或 block 结束 |
| 在 metadata 中放原始 stdout | 上下文膨胀，下游难以解析 | 放摘要和关键文件路径 |
| 循环依赖 | 依赖引擎无法提升 | 创建任务时用 DAG 思维 |
| 同一任务分配多个 assignee | 调度器无法匹配 profile | 拆分为多个独立任务 |
| 跨看板 link | 看板隔离禁止 | 在同一看板内组织依赖 |
| 不发心跳的长任务 | 被调度器回收 | 每小时至少一次 heartbeat |

### 12.4 推荐的 metadata 规范

```text
metadata schema 建议
═══════════════════

工程任务:
  changed_files: string[]      # 变更文件列表
  verification: string[]       # 验证命令
  tests_run: number            # 测试数量
  residual_risk: string[]      # 未覆盖的风险

审查任务:
  issues_found: number         # 发现问题数
  severity: "low"|"medium"|"high"
  blocking_issues: string[]    # 阻塞性问题

研究任务:
  sources: string[]            # 信息来源
  confidence: "low"|"medium"|"high"
  key_findings: string[]       # 关键发现
```

---

## 附：Kanban 的使用心智模型

```text
Hermes Kanban 的核心不是"派生一个子 Agent 并等待返回"，
而是把工作拆成可恢复、可审计、可人工介入的任务记录。

每个任务是一条持久化记录。   ── tasks 表
每次执行尝试是一条 run 记录。 ── task_runs 表
每次状态变化是一条 event 记录。── task_events 表
每次完成或阻塞都要提供结构化交接。
人类和智能体通过同一套数据库事实协作。

一句话：Kanban 是给 AI Agent 用的 Git——
每次操作都有记录，每次失败都可以回溯，每次交接都有上下文。
```

---

## 附：架构对照表（Kanban 概念 → TypeScript 源码映射）

| Kanban 概念 | TypeScript 对应 | 源码位置 |
| --- | --- | --- |
| 任务注册表 | `AppState.tasks` | `src/state/AppStateStore.ts` |
| 任务创建/更新 | `TaskCreateTool`/`TaskUpdateTool` | `src/tools/TaskCreateTool/` |
| 工作者声明 | `AgentTool` spawn | `src/tools/AgentTool/runAgent.ts` |
| 心跳 | Background task polling | `src/utils/task/framework.ts` |
| 结构化交接 | Agent notification XML | `src/tools/AgentTool/` |
| 依赖管理 | `blocks`/`blockedBy` 字段 | `src/tools/TaskCreateTool/` |
| 调度器 | QueryEngine + toolOrchestration | `src/QueryEngine.ts` |
| 工作空间隔离 | Worktree tools | `src/tools/EnterWorktreeTool/` |
| 定时触发 | `ScheduleCronTool` | `src/tools/ScheduleCronTool/` |
| 审计追踪 | Message history | `src/state/AppStateStore.ts` |
| 状态机 | Task status workflow | `src/tasks/` |
| Profile | Agent definitions | `src/tools/AgentTool/loadAgentsDir.ts` |
| MCP 工具 | MCP tool integration | `src/services/mcp/` |
| 协调模式 | Coordinator mode | `src/coordinator/` |
