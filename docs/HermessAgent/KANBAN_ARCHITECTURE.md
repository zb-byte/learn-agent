# Hermes Kanban 系统架构文档

> 基于源码 `/Users/wangzhongbin/Documents/code/fangzhen/hermes-agent` 逐行分析

---

## 目录

1. [架构总览图](#1-架构总览图)
2. [原子工具（9个）](#2-原子工具9个)
3. [任务执行者 Worker Agent](#3-任务执行者-worker-agent)
4. [任务调度 Dispatcher](#4-任务调度-dispatcher)
5. [任务创建入口](#5-任务创建入口)

---

## 1. 架构总览图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          任务创建入口 (Entry Points)                         │
│                                                                             │
│  ┌─────────────┐  ┌──────────────────┐  ┌───────────────────┐              │
│  │  CLI 入口    │  │  Slash 入口       │  │  Agent 工具调用    │              │
│  │ hermes kanban│  │ /kanban create   │  │ kanban_create()   │              │
│  │   create     │  │                  │  │                   │              │
│  └──────┬───────┘  └────────┬─────────┘  └─────────┬─────────┘              │
│         │                   │                       │                        │
│         └───────────────────┼───────────────────────┘                        │
│                             │                                                │
│                             ▼                                                │
│                   ┌──────────────────┐                                       │
│                   │  kanban_db.py    │                                       │
│                   │  create_task()   │                                       │
│                   │  → INSERT SQLite │                                       │
│                   └────────┬─────────┘                                       │
└────────────────────────────┼─────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       任务调度 Dispatcher (Gateway 内嵌)                      │
│                                                                             │
│  ┌──────────────────────────────┐  ┌─────────────────────────────────────┐  │
│  │ Dispatcher Watcher           │  │ Notifier Watcher                    │  │
│  │ (asyncio Task, 每60s tick)   │  │ (asyncio Task, 每5s 轮询)          │  │
│  │                              │  │                                     │  │
│  │  dispatch_once() 8步调度:    │  │  订阅-游标模型:                      │  │
│  │  ① 收割僵尸子进程            │  │  ① 收集所有 board 的订阅            │  │
│  │  ② 回收TTL过期任务           │  │  ② claim_unseen_events_for_sub     │  │
│  │  ③ 回收无心跳任务            │  │  ③ 推送终态事件到 Discord/Telegram │  │
│  │  ④ 检测崩溃任务(PID消亡)     │  │  ④ advance_cursor / unsub         │  │
│  │  ⑤ 状态推进 todo→ready      │  │                                     │  │
│  │  ⑥ 派发 ready 任务          │  └─────────────────────────────────────┘  │
│  │  ⑦ 派发 review 任务         │                                           │
│  │  ⑧ 强制超时/熔断保护        │                                           │
│  └──────────┬───────────────────┘                                           │
│             │                                                               │
│             │ _default_spawn() 构造子进程:                                   │
│             │ hermes -p <assignee> chat -q "work kanban task <id>"          │
│             │ 注入 10+ 环境变量 (HERMES_KANBAN_TASK, HERMES_KANBAN_WORKSPACE│
│             │   HERMES_KANBAN_DB, HERMES_KANBAN_BOARD, HERMES_PROFILE ...) │
│             │                                                               │
│             ▼                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       任务执行者 Worker Agent (子进程)                        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ 系统提示词三层结构                                                    │    │
│  │  ┌ stable:  身份 + KANBAN_GUIDANCE + skill 提示                      │    │
│  │  ├ context: AGENTS.md / .cursorrules                                │    │
│  │  └ volatile: 记忆快照 / USER.md / 时间戳                             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  用户提示词: "work kanban task {task.id}"                                   │
│  + build_worker_context() 构建的完整任务上下文                              │
│                                                                             │
│  ┌──────────────────────────────────────────────┐                          │
│  │ 可用工具 (9个 kanban_* + 通用工具)             │                          │
│  │                                               │                          │
│  │ kanban_show      → 查看任务详情               │                          │
│  │ kanban_complete  → 完成任务,结构化交接         │                          │
│  │ kanban_block     → 阻塞任务,等待人工          │                          │
│  │ kanban_heartbeat → 发送心跳,延长TTL           │                          │
│  │ kanban_comment   → 添加评论到任务线程          │                          │
│  │ kanban_create    → 创建子任务                  │                          │
│  │ kanban_list      → 列出任务(仅编排器)          │                          │
│  │ kanban_unblock   → 解除阻塞(仅编排器)          │                          │
│  │ kanban_link      → 添加父子依赖关系            │                          │
│  └──────────────────────────────────────────────┘                          │
│                                                                             │
│  技能: kanban-worker (自动加载) + task.skills (按需)                        │
│  子Agent: delegate_task (用于短时推理子任务)                                 │
│  记忆: 支持 MemoryStore + USER.md                                          │
└─────────────────────────────────────────────────────────────────────────────┘

                              │
                              ▼
                    ┌──────────────────┐
                    │  SQLite 数据库    │
                    │ ~/.hermes/kanban  │
                    │   .db            │
                    │                  │
                    │ tasks 表         │
                    │ task_runs 表     │
                    │ task_comments 表 │
                    │ task_events 表   │
                    │ task_links 表    │
                    │ kanban_notify_   │
                    │   subs 表        │
                    └──────────────────┘
```

---

## 2. 原子工具（9个）

> 源码位置: `tools/kanban_tools.py`

所有 kanban 工具定义在同一个文件中，通过 `registry.register()` 注册到工具注册表。每个工具由三部分组成：**Schema（参数定义）** → **Handler（处理函数）** → **Registration（注册）**。

### 2.1 工具门控机制

工具通过 `_check_kanban_mode()` 决定是否对当前 Agent 可见：

```python
# tools/kanban_tools.py:62-76
def _check_kanban_mode() -> bool:
    """任务生命周期工具在以下条件下可用：
    1. HERMES_KANBAN_TASK 已设置（调度器生成的 worker 子进程）
    2. 当前 profile 的 toolsets 配置中包含 "kanban"（编排器模式）
    """
    if os.environ.get("HERMES_KANBAN_TASK"):  # Worker 模式
        return True
    return _profile_has_kanban_toolset()       # 编排器模式
```

### 2.2 kanban_show — 查看任务完整状态

```python
# tools/kanban_tools.py:255-328
def _handle_show(args: dict, **kw) -> str:
    """读取任务的完整状态：任务信息、父/子任务、评论、运行历史、事件。

    核心逻辑：
    1. 从 args 或 HERMES_KANBAN_TASK 环境变量获取 task_id
    2. 连接 SQLite 数据库，调用 kb.get_task() 查询任务行
    3. 同时查询 parents/children/comments/events/runs 关联数据
    4. 调用 kb.build_worker_context() 构建预格式化的 worker_context 文本
    5. 以 JSON 格式返回所有信息
    """
    tid = _default_task_id(args.get("task_id"))  # 优先取参数，否则读环境变量
    # ...
    kb, conn = _connect(board=board)
    task = kb.get_task(conn, tid)          # 查任务主表
    comments = kb.list_comments(conn, tid) # 查评论
    events = kb.list_events(conn, tid)     # 查事件日志
    runs = kb.list_runs(conn, tid)         # 查运行历史
    parents = kb.parent_ids(conn, tid)     # 查父任务
    children = kb.child_ids(conn, tid)     # 查子任务
    # ... 打包为 JSON 返回
```

**使用场景**: Worker 启动后第一步调用 `kanban_show()` 获取完整上下文。

### 2.3 kanban_create — 创建新任务

```python
# tools/kanban_tools.py:639-720
def _handle_create(args: dict, **kw) -> str:
    """创建新的 kanban 任务。编排器用它分发工作。

    核心逻辑：
    1. 校验必填参数：title（标题）和 assignee（执行者 profile）
    2. 解析可选参数：parents, tenant, priority, workspace_kind, skills 等
    3. 调用 kb.create_task() → INSERT INTO tasks 表
    4. 返回新任务的 task_id 和 status

    关键参数：
    - parents: 父任务 ID 列表，新任务在所有父任务完成后才自动提升为 ready
    - idempotency_key: 幂等键，防止重复创建
    - skills: 强制加载的技能列表（如 ["translation", "github-code-review"]）
    """
    title = args.get("title")
    assignee = args.get("assignee")  # 必填：调度器只会派发有 assignee 的任务
    parents = args.get("parents") or []
    tenant = args.get("tenant") or os.environ.get("HERMES_TENANT")
    # ...
    new_tid = kb.create_task(
        conn,
        title=str(title).strip(),
        body=body,
        assignee=str(assignee),
        parents=tuple(parents),
        tenant=tenant,
        priority=int(priority) if priority is not None else 0,
        workspace_kind=str(workspace_kind),
        skills=skills,
        initial_status=str(initial_status),
        created_by=os.environ.get("HERMES_PROFILE") or "worker",
    )
    return _ok(task_id=new_tid, status=new_task.status)
```

### 2.4 kanban_complete — 完成任务

```python
# tools/kanban_tools.py:392-511
def _handle_complete(args: dict, **kw) -> str:
    """标记当前任务完成，附带结构化交接信息。

    核心逻辑：
    1. 校验 task_id 和 worker 归属权（_enforce_worker_task_ownership）
    2. 校验 summary 或 result 至少提供一个
    3. 如果提供了 created_cards，验证这些子任务确实由当前 worker 创建
       （防止幻觉：HallucinatedCardsError）
    4. 如果提供了 artifacts，将路径列表合并到 metadata 中
       （gateway notifier 会读取并上传为原生附件）
    5. 调用 kb.complete_task() → UPDATE tasks SET status='done'
    6. 返回 task_id 和 run_id

    参数说明：
    - summary: 1-3 句人工可读的完成描述（推荐）
    - metadata: 机器可读的结构化数据（changed_files, tests_run 等）
    - created_cards: 本次运行中通过 kanban_create 创建的子任务 ID 列表
    - artifacts: 产出的文件绝对路径列表（如 PDF、图片）
    """
    tid = _default_task_id(args.get("task_id"))
    ownership_err = _enforce_worker_task_ownership(tid)  # Worker 只能完成自己的任务
    # ...
    ok = kb.complete_task(
        conn, tid,
        result=result, summary=summary, metadata=metadata,
        created_cards=created_cards,
        expected_run_id=_worker_run_id(tid),
    )
    # ... HallucinatedCardsError 处理
    return _ok(task_id=tid, run_id=run.id)
```

### 2.5 kanban_comment — 添加评论

```python
# tools/kanban_tools.py:603-636
def _handle_comment(args: dict, **kw) -> str:
    """向任务线程添加评论。

    核心逻辑：
    1. 校验 task_id 和 body（必填）
    2. author 从环境变量 HERMES_PROFILE 派生，不接受调用者传入
       （防止伪造系统指令注释）
    3. 调用 kb.add_comment() → INSERT INTO task_comments 表
    4. 返回 comment_id

    安全设计：author 故意从运行时身份派生而非 args 传入，因为
    build_worker_context 会将评论渲染为 "**{author}** (timestamp): {body}"，
    如果允许自定义 author，worker 可以伪造 "hermes-system" 等权威名称
    向后续 worker 注入指令。
    """
    tid = args.get("task_id")  # 可以是自己的任务，也可以是其他任务
    body = args.get("body")
    # author 从环境变量派生，不可由调用者伪造
    author = os.environ.get("HERMES_PROFILE") or "worker"
    kb, conn = _connect(board=board)
    cid = kb.add_comment(conn, tid, author=author, body=str(body))
    return _ok(task_id=tid, comment_id=cid)
```

### 2.6 kanban_block — 阻塞任务

```python
# tools/kanban_tools.py:514-549
def _handle_block(args: dict, **kw) -> str:
    """将任务转为 blocked 状态，等待人工输入。

    核心逻辑：
    1. 校验 task_id、worker 归属权、reason（必填）
    2. 调用 kb.block_task() → UPDATE tasks SET status='blocked'
    3. reason 会展示在看板和后续 unblock 时的上下文中
    """
    reason = args.get("reason")  # 必填：说明需要什么人工输入
    ok = kb.block_task(conn, tid, reason=reason, expected_run_id=_worker_run_id(tid))
```

### 2.7 kanban_heartbeat — 发送心跳

```python
# tools/kanban_tools.py:552-600
def _handle_heartbeat(args: dict, **kw) -> str:
    """在长时间操作中发送心跳信号，表示 worker 仍然存活。

    核心逻辑：
    1. 调用 kb.heartbeat_claim() 延长 claim TTL
    2. 调用 kb.heartbeat_worker() 记录心跳事件
    3. 不做任何工作变更，纯副作用
    """
    claim_lock = os.environ.get("HERMES_KANBAN_CLAIM_LOCK")
    kb.heartbeat_claim(conn, tid, claimer=claim_lock)  # 延长 TTL
    kb.heartbeat_worker(conn, tid, note=note)           # 记录事件
```

### 2.8 kanban_list — 列出任务（仅编排器）

```python
# tools/kanban_tools.py:331-389
def _handle_list(args: dict, **kw) -> str:
    """列出任务摘要。仅编排器可用，Worker 看不到此工具。

    核心逻辑：
    1. 检查编排器权限（_require_orchestrator_tool）
    2. 调用 kb.recompute_ready() 刷新依赖状态
    3. 调用 kb.list_tasks() 查询，支持 filter: assignee/status/tenant/limit
    4. 返回紧凑的任务摘要列表
    """
    guard = _require_orchestrator_tool("kanban_list")  # Worker 被拒绝
    rows = kb.list_tasks(conn, assignee=assignee, status=status,
                         tenant=tenant, limit=limit + 1)
```

### 2.9 kanban_unblock — 解除阻塞（仅编排器）

```python
# tools/kanban_tools.py:723-748
def _handle_unblock(args: dict, **kw) -> str:
    """将 blocked 任务转回 ready。仅编排器可用。

    核心逻辑：调用 kb.unblock_task() → UPDATE tasks SET status='ready'
    """
    guard = _require_orchestrator_tool("kanban_unblock")
    ok = kb.unblock_task(conn, str(tid))
```

### 2.10 kanban_link — 添加依赖关系

```python
# tools/kanban_tools.py:751-770
def _handle_link(args: dict, **kw) -> str:
    """在两个已存在的任务之间添加 parent→child 依赖边。

    核心逻辑：调用 kb.link_tasks() → INSERT INTO task_links
    子任务在所有父任务完成前不会提升为 ready。检测循环依赖和自引用。
    """
    kb.link_tasks(conn, parent_id=parent_id, child_id=child_id)
```

### 工具汇总表

| 工具 | 功能 | 权限 | DB 操作 | 行号 |
|------|------|------|---------|------|
| `kanban_show` | 查看任务完整状态 | Worker + 编排器 | SELECT | 255-328 |
| `kanban_list` | 列出任务摘要 | 仅编排器 | SELECT | 331-389 |
| `kanban_complete` | 完成任务+结构化交接 | Worker + 编排器 | UPDATE | 392-511 |
| `kanban_block` | 阻塞任务等待人工 | Worker + 编排器 | UPDATE | 514-549 |
| `kanban_heartbeat` | 发送心跳延长TTL | Worker + 编排器 | UPDATE | 552-600 |
| `kanban_comment` | 添加评论到线程 | Worker + 编排器 | INSERT | 603-636 |
| `kanban_create` | 创建子任务 | Worker + 编排器 | INSERT | 639-720 |
| `kanban_unblock` | 解除阻塞 | 仅编排器 | UPDATE | 723-748 |
| `kanban_link` | 添加依赖关系 | Worker + 编排器 | INSERT | 751-770 |

---

## 3. 任务执行者 Worker Agent

> Worker 是一个标准的 Hermes Agent 对话循环，通过子进程启动，模型通过 tool call 与 Kanban 系统交互。

### 3.1 Agent 上下文注入

Worker 的上下文通过 **环境变量** 注入。`_default_spawn()` 在启动子进程时注入以下关键环境变量：

```python
# hermes_cli/kanban_db.py:5294-5353
env["HERMES_KANBAN_TASK"] = task.id                 # 任务 ID（核心标识）
env["HERMES_KANBAN_WORKSPACE"] = workspace           # 工作区路径
env["HERMES_KANBAN_DB"] = str(kanban_db_path(...))   # 数据库路径
env["HERMES_KANBAN_BOARD"] = resolved_board          # 看板 slug
env["HERMES_KANBAN_RUN_ID"] = str(task.current_run_id) # 当前运行 ID
env["HERMES_KANBAN_CLAIM_LOCK"] = task.claim_lock    # 任务锁
env["HERMES_PROFILE"] = profile_arg                  # profile 名称
env["HERMES_HOME"] = resolve_profile_env(...)        # profile 目录
env["HERMES_KANBAN_WORKSPACES_ROOT"] = str(...)      # 工作区根目录
# 如果有 tenant:
env["HERMES_TENANT"] = task.tenant                   # 租户隔离
# 如果有 branch:
env["HERMES_KANBAN_BRANCH"] = task.branch_name       # 分支名
```

这些环境变量的作用：
- `HERMES_KANBAN_TASK` 触发 `_check_kanban_mode()` 返回 True，激活 kanban 工具集
- 其他变量在工具内部被读取（如 `_default_task_id()` 读取 `HERMES_KANBAN_TASK` 作为默认 task_id）

### 3.2 Worker 使用的工具

Worker 子进程启动时，kanban 工具通过 `check_fn` 门控自动注册：

```python
# tools/kanban_tools.py:1218-1297 — 工具注册
# check_fn=_check_kanban_mode 的工具（Worker 可见）：
#   kanban_show, kanban_complete, kanban_block,
#   kanban_heartbeat, kanban_comment, kanban_create, kanban_link
# check_fn=_check_kanban_orchestrator_mode 的工具（Worker 不可见）：
#   kanban_list, kanban_unblock
```

除 kanban 工具外，Worker 还可以使用所有通用工具（terminal、file 操作等），具体取决于 profile 配置。

### 3.3 记忆（Memory）

Worker Agent **支持记忆系统**。在 `agent_init.py` 中：

```python
# agent/agent_init.py:983-991
from tools.memory_tool import MemoryStore
agent._memory_store = MemoryStore(
    db_path=memory_db_path,
    session_id=agent.session_id,
    enabled=agent._memory_enabled,
)
```

系统提示词中根据 `memory` 是否在 `valid_tool_names` 中来注入记忆使用指导：

```python
# agent/system_prompt.py:105-106
if "memory" in agent.valid_tool_names:
    tool_guidance.append(MEMORY_GUIDANCE)
```

KANBAN_GUIDANCE 中建议 Worker 在记忆条目前加租户前缀以实现多租户隔离。

### 3.4 系统提示词和用户提示词组装

**系统提示词**采用三层结构：

```python
# agent/system_prompt.py:10-21（结构定义）
# 1. stable — 身份、工具指导、技能提示（变化少，利于缓存）
# 2. context — 上下文文件 (AGENTS.md, .cursorrules)
# 3. volatile — 记忆快照、USER.md、时间戳（每次对话重建）
```

KANBAN_GUIDANCE 在初始化时预解析，避免每次重建系统提示词时重复计算：

```python
# agent/agent_init.py:845-848
from agent.prompt_builder import KANBAN_GUIDANCE
agent._kanban_worker_guidance = (
    KANBAN_GUIDANCE if "kanban_show" in agent.valid_tool_names else ""
)
```

然后在 `build_system_prompt` 中注入到 stable 层：

```python
# agent/system_prompt.py:115-120
_kanban_guidance = getattr(agent, "_kanban_worker_guidance", None)
if _kanban_guidance:
    tool_guidance.append(_kanban_guidance)
elif _kanban_guidance is None and "kanban_show" in agent.valid_tool_names:
    tool_guidance.append(KANBAN_GUIDANCE)  # 兜底
```

**用户提示词**只有一句话：

```python
# hermes_cli/kanban_db.py:5293
prompt = f"work kanban task {task.id}"
```

Worker 调用 `kanban_show()` 时，`build_worker_context()` 返回的完整上下文文本包含：
1. 任务标题（必需）
2. 任务 body（可选，上限 8KB）
3. 先前尝试记录（最近 N 次关闭的 run）
4. 父任务的结构化交接结果（summary + metadata）
5. 当前 assignee 的跨任务角色历史（最近 5 次完成运行）
6. 评论线程（最近 N 条）

```python
# hermes_cli/kanban_db.py:5497-5519
def build_worker_context(conn, task_id) -> str:
    """返回 worker 应该阅读的完整文本来理解其任务。

    顺序：
      1. 任务标题（必需）
      2. 任务体（可选的开启帖子，上限 8 KB）
      3. 此任务的先前尝试（最多显示最近 _CTX_MAX_PRIOR_ATTEMPTS 次）
      4. 每个已完成父任务的结构化交接结果
      5. 分配者的跨任务角色历史（最近 5 次完成运行）
      6. 评论线程（最多显示最近 _CTX_MAX_COMMENTS 条）
    """
```

### 3.5 是否会调用 SKILL

**是的**。`_default_spawn()` 自动加载 `kanban-worker` 技能：

```python
# hermes_cli/kanban_db.py:5364-5390
# 自动加载 kanban-worker 技能，提供模式库：
# - 良好的 summary/metadata 形状示例
# - 重试场景诊断
# - 阻塞原因示例
# - 工作区处理指南
if _kanban_worker_skill_available(env.get("HERMES_HOME")):
    cmd.extend(["--skills", "kanban-worker"])
# 任务指定的额外技能
if task.skills:
    for sk in task.skills:
        if sk and sk != "kanban-worker":
            cmd.extend(["--skills", sk])
```

`kanban-worker` 技能位于 `skills/devops/kanban-worker/SKILL.md`。

### 3.6 是否有子 Agent

**是的**。Worker 可以通过 `delegate_task` 工具创建子 Agent：

```python
# KANBAN_GUIDANCE 中的明确指导（agent/prompt_builder.py:254-256）：
"- Do not call `delegate_task` as a board substitute. `delegate_task` is "
"for short reasoning subtasks inside your own run; board tasks are for "
"cross-agent handoffs that outlive one API loop."
```

两种协作方式的区别：
- **delegate_task**: 单次运行内的短时推理子任务，生命周期不超过当前 API 循环
- **kanban_create**: 跨 Agent 交接，创建新的 board 任务，生命周期独立于当前 worker

并发限制通过 `max_concurrent_children` 控制（默认 3）：

```python
# agent/run_agent.py:2276-2300
def _cap_delegate_task_calls(tool_calls, max_delegate=3):
    """截断过多的 delegate_task 调用以执行并发上限。"""
```

---

## 4. 任务调度 Dispatcher

> Dispatcher 是 Gateway 内嵌的后台 asyncio Task，负责清理异常任务、推进状态、派发 Worker 子进程。

### 4.1 Dispatcher Watcher

```python
# gateway/run.py:4994-5110
async def _kanban_dispatcher_watcher(self) -> None:
    """Gateway 内嵌的看板调度器 — 每 dispatch_interval_seconds 执行一次。

    由 gateway/run.py:4086 在 Gateway 启动时创建：
        asyncio.create_task(self._kanban_dispatcher_watcher())

    关键配置（从 config.yaml 的 kanban 节读取）：
    - dispatch_in_gateway: 是否在 gateway 内运行调度（默认 True）
    - dispatch_interval_seconds: tick 间隔（默认 60s，最小 1s）
    - max_spawn: 最大并发 worker 数（实时限制，非每 tick 预算）
    - max_in_progress: 最大同时运行任务数
    - failure_limit: 连续失败几次后自动阻塞（默认 2）
    - dispatch_stale_timeout_seconds: 无心跳超时检测（0 = 禁用）
    """
    interval = float(kanban_cfg.get("dispatch_interval_seconds", 60) or 60)
    interval = max(interval, 1.0)  # 安全线：低于此值容易出问题
    # ...
    while self._running:
        # 每个 board 单独调度
        for board_slug in board_slugs:
            result = await asyncio.to_thread(_tick_once_for_board, board_slug)
        await asyncio.sleep(interval)  # 等待下一个 tick
```

### 4.2 dispatch_once — 调度心脏（8步）

```python
# hermes_cli/kanban_db.py:4702-5001
def dispatch_once(conn, *, spawn_fn=None, max_spawn=None, failure_limit=2) -> DispatchResult:
    """执行一个调度 tick。

    步骤：
      ① 收割僵尸子进程（os.waitpid WNOHANG，仅 Unix）
      ② 回收 TTL 过期的 running 任务（release_stale_claims）
      ③ 回收无心跳的 running 任务（detect_stale_running）
      ④ 检测崩溃的 running 任务（PID 消亡 → detect_crashed_workers）
      ⑤ 状态推进：todo → ready（recompute_ready，前提：所有父任务 done）
      ⑥ 派发 ready 任务（claim_task → _default_spawn）
      ⑦ 派发 review 任务（加载 sdlc-review 技能）
      ⑧ 强制超时 + 熔断保护（enforce_max_runtime → failure_limit 自动阻塞）

    全程受 max_spawn 并发上限和 failure_limit 熔断保护。
    max_spawn 是实时并发上限（running 计数 + 本次 spawned），不是每 tick 预算。
    """

    # ① 收割僵尸子进程
    if os.name != "nt":
        while True:
            try:
                _pid, _status = os.waitpid(-1, os.WNOHANG)  # 非阻塞回收
            except ChildProcessError:
                break
            if _pid == 0:
                break
            _record_worker_exit(_pid, _status)  # 记录退出状态

    result = DispatchResult()
    # ② 回收TTL过期任务
    result.reclaimed = release_stale_claims(conn)
    # ③ 回收无心跳任务
    result.stale = detect_stale_running(conn, stale_timeout_seconds=...)
    # ④ 检测崩溃任务
    result.crashed = detect_crashed_workers(conn)
    # ⑤ 状态推进 todo → ready
    result.promoted = recompute_ready(conn)
    # ⑥ 强制超时
    result.timed_out = enforce_max_runtime(conn)

    # ⑥ 派发 ready 任务
    running_count = conn.execute(
        "SELECT COUNT(*) FROM tasks WHERE status = 'running'"
    ).fetchone()[0]

    ready_rows = conn.execute(
        "SELECT id, assignee FROM tasks "
        "WHERE status = 'ready' AND claim_lock IS NULL "
        "ORDER BY priority DESC, created_at ASC"  # 按优先级和创建时间排序
    ).fetchall()

    for row in ready_rows:
        # 并发上限检查
        if max_spawn and running_count + spawned >= max_spawn:
            break
        # 检查 assignee 是否是有效的 Hermes profile
        if profile_exists and not profile_exists(row["assignee"]):
            result.skipped_nonspawnable.append(row["id"])
            continue
        # 重试保护：拒绝在短时间内重复派发
        guard_reason = check_respawn_guard(conn, row["id"])
        if guard_reason:
            continue
        # 原子性地声明任务
        claimed = claim_task(conn, row["id"], ttl_seconds=ttl_seconds)
        # 解析工作区路径
        workspace = resolve_workspace(claimed, board=board)
        # 生成 Worker 子进程
        pid = _spawn(claimed, str(workspace), board=board)
        if pid:
            _set_worker_pid(conn, claimed.id, int(pid))
        spawned += 1

    # ⑦ 派发 review 任务（相同并发模型）
    review_rows = conn.execute(
        "SELECT id, assignee FROM tasks "
        "WHERE status = 'review' AND claim_lock IS NULL"
    ).fetchall()
    # ... 类似 ready 派发逻辑，但强制加载 sdlc-review 技能
    claimed.skills = ["sdlc-review"]
```

### 4.3 _default_spawn — 构造 Worker 子进程

```python
# hermes_cli/kanban_db.py:5267-5431
def _default_spawn(task, workspace, *, board=None) -> Optional[int]:
    """Fire-and-forget 子进程启动。

    构造命令：hermes -p <assignee> chat -q "work kanban task <id>"
    返回子进程 PID，供后续 tick 检测崩溃。
    """
    profile_arg = normalize_profile_name(task.assignee)
    prompt = f"work kanban task {task.id}"

    # 构造环境变量（10+ 个）
    env = dict(os.environ)
    env["HERMES_KANBAN_TASK"] = task.id
    env["HERMES_KANBAN_WORKSPACE"] = workspace
    env["HERMES_KANBAN_DB"] = str(kanban_db_path(board=board))
    env["HERMES_KANBAN_WORKSPACES_ROOT"] = str(workspaces_root(board=board))
    env["HERMES_KANBAN_BOARD"] = resolved_board
    env["HERMES_PROFILE"] = profile_arg
    env["HERMES_HOME"] = resolve_profile_env(profile_arg)
    if task.tenant:
        env["HERMES_TENANT"] = task.tenant
    if task.branch_name:
        env["HERMES_KANBAN_BRANCH"] = task.branch_name
    if task.current_run_id is not None:
        env["HERMES_KANBAN_RUN_ID"] = str(task.current_run_id)
    if task.claim_lock:
        env["HERMES_KANBAN_CLAIM_LOCK"] = task.claim_lock

    # 构造命令行
    cmd = [
        *_resolve_hermes_argv(),
        "-p", profile_arg,
        "--accept-hooks",            # 允许 profile 级的 hooks
        "--skills", "kanban-worker", # 自动加载 kanban-worker 技能
        # ... 追加 task.skills 指定的技能
        # ... 追加 model_override（如果指定）
        "chat",
        "-q", prompt,                # 用户提示词
    ]

    # 输出重定向到 <board-root>/logs/<task_id>.log
    log_path = worker_logs_dir(board=board) / f"{task.id}.log"
    proc = subprocess.Popen(
        cmd,
        cwd=workspace,                  # 工作目录设为任务工作区
        stdin=subprocess.DEVNULL,        # 无标准输入
        stdout=log_f,                    # 输出到日志文件
        stderr=subprocess.STDOUT,        # 错误合并到标准输出
        env=env,
        start_new_session=True,          # 脱离控制终端（但不脱离父进程）
    )
    return proc.pid
```

### 4.4 熔断保护

```python
# hermes_cli/kanban_db.py — 连续失败自动阻塞
DEFAULT_FAILURE_LIMIT = 2  # 默认连续失败 2 次后自动阻塞

def _record_spawn_failure(conn, task_id, error, failure_limit):
    """记录 spawn 失败，超过 failure_limit 次连续失败后自动阻塞任务。

    防止调度器对无法修复的任务无限重试。
    counter 只在成功完成时重置（不在成功 spawn 时重置），
    因为 spawn 成功不等于运行成功。
    """
    conn.execute(
        "UPDATE tasks SET consecutive_failures = consecutive_failures + 1 "
        "WHERE id = ?", (task_id,)
    )
    if consecutive_failures >= effective_limit:
        conn.execute("UPDATE tasks SET status = 'blocked' WHERE id = ?", (task_id,))
        _append_event(conn, task_id, "gave_up", {"reason": error})
```

### 4.5 Notifier Watcher — 通知推送

```python
# gateway/run.py:4495-4829
async def _kanban_notifier_watcher(self, interval=5.0):
    """独立并行运行的通知推送器。

    通过订阅-游标模型将终态事件推送到 Discord/Telegram：
    - 每 5 秒轮询一次
    - 只推送 TERMINAL_KINDS = ("completed", "blocked", "gave_up", "crashed", "timed_out")
    - 游标机制确保不重复推送
    - 订阅只在任务达到真正终态（done/archived）时删除
    - 非终态事件（crashed/timed_out）保留订阅，因为 dispatcher 可能重试

    核心流程：
    1. 遍历所有 board，收集所有 notify_subs 订阅
    2. 对每个订阅，调用 claim_unseen_events_for_sub() 获取新事件
    3. 格式化消息（含 worker 身份标签 @assignee）
    4. 通过 adapter.send() 推送到对应平台
    5. 成功后 advance_cursor；失败后 rewind_cursor 或删除订阅
    """
    TERMINAL_KINDS = ("completed", "blocked", "gave_up", "crashed", "timed_out")
    MAX_SEND_FAILURES = 3  # 连续 3 次发送失败后删除订阅

    while self._running:
        # ... 遍历 boards, 收集 deliveries
        for ev in events:
            if kind == "completed":
                msg = f"✔ {tag}Kanban {task_id} done — {title}{handoff}"
            elif kind == "blocked":
                msg = f"⏸ {tag}Kanban {task_id} blocked{reason}"
            elif kind == "gave_up":
                msg = f"✖ {tag}Kanban {task_id} gave up after repeated failures"
            elif kind == "crashed":
                msg = f"✖ {tag}Kanban {task_id} worker crashed; dispatcher will retry"
            elif kind == "timed_out":
                msg = f"⏱ {tag}Kanban {task_id} timed out; will retry"
        # ...
```

---

## 5. 任务创建入口

> Kanban 任务有三种创建入口，最终都调用 `kanban_db.create_task()` 写入 SQLite。

### 5.1 CLI 入口 — `hermes kanban create`

**源码位置**: `hermes_cli/kanban.py:1266-1326`

```python
# hermes_cli/kanban.py:1266
def _cmd_create(args: argparse.Namespace) -> int:
    """处理 hermes kanban create 命令。"""
    # 解析工作区类型和分支名
    ws_kind, ws_path = _parse_workspace_flag(args.workspace)
    branch_name = _parse_branch_flag(args.branch)

    with kb.connect() as conn:
        task_id = kb.create_task(
            conn,
            title=args.title,          # 必填：任务标题
            body=args.body,            # 可选：详细描述
            assignee=args.assignee,    # 可选：执行者 profile
            created_by=args.created_by or _profile_author(),
            workspace_kind=ws_kind,    # scratch / dir / worktree
            workspace_path=ws_path,
            branch_name=branch_name,
            tenant=args.tenant,
            priority=args.priority,
            parents=tuple(args.parent or ()),
            triage=bool(args.triage),
            idempotency_key=args.idempotency_key,
            max_runtime_seconds=max_runtime,
            skills=args.skills,
            initial_status=args.initial_status,
        )
```

**使用示例**:

```bash
# 基本创建（自动进入 running 状态）
hermes kanban create "Fix authentication bug" --assignee developer-a

# 带详细说明和优先级
hermes kanban create "Implement API endpoint" \
  --body "Create REST endpoint for user management" \
  --assignee backend-dev \
  --priority 5

# 创建子任务（依赖关系）
hermes kanban create "Write unit tests" \
  --assignee qa-engineer \
  --parent t_abc123def456

# 创建工作树任务
hermes kanban create "Feature branch work" \
  --workspace worktree \
  --branch feat/t-abc123-feature \
  --assignee developer-b

# 创建到待筛选列
hermes kanban create "Research competitor pricing" \
  --assignee analyst \
  --triage

# 带幂等键（防重复）
hermes kanban create "Daily report" \
  --assignee reporter \
  --idempotency-key "daily-report-2026-05-21"

# 带技能和运行时限制
hermes kanban create "Translate docs to Japanese" \
  --assignee translator \
  --skill translation \
  --max-runtime 2h

# JSON 输出（脚本集成）
hermes kanban create "Automated task" --assignee bot --json
```

### 5.2 Slash 入口 — `/kanban create`

**源码位置**: `hermes_cli/kanban.py:2602-2677`

```python
# hermes_cli/kanban.py:2602
def run_slash(rest: str) -> str:
    """处理 /kanban 命令。

    rest 是 /kanban 之后的所有文本。
    使用 shlex.split() 解析后委托给同一个 argparse 树，
    确保 Slash 入口和 CLI 入口行为完全一致。
    """
    tokens = shlex.split(rest) if rest else []
    # 构建 argparse 解析器
    kanban_parser = build_parser(_top_sub)
    kanban_parser.prog = "/kanban"  # 让 usage 显示 /kanban 而非 hermes kanban
    args = kanban_parser.parse_args(tokens)
    kanban_command(args)  # 委托给同一个命令处理器
```

**使用示例** (在 Hermes 交互式 CLI 或 Gateway 中):

```
/kanban create "Review PR #123" --assignee senior-dev

/kanban create "Update documentation" \
  --body "Add API docs for new endpoints" \
  --assignee technical-writer \
  --priority 3 \
  --parent t_abc123
```

### 5.3 Agent 工具调用 — `kanban_create()`

**源码位置**: `tools/kanban_tools.py:639-720`

Agent 在执行过程中通过 tool call 调用 `kanban_create`，参数通过结构化 JSON 传递（避免 shell 引号问题）。

**使用场景**: 编排器 Agent 分解工作时创建子任务：

```python
# Agent 内部调用示例（tool call）
kanban_create(
    title="Validate test results",
    assignee="qa-lead",
    body="Review automated test results and file bugs",
    parents=["t_abc123def456"],  # 当前任务 ID 作为父任务
    priority=1,
)
```

### 5.4 三种入口的对比

| 特性 | CLI 入口 | Slash 入口 | Agent 工具 |
|------|---------|-----------|-----------|
| 使用者 | 人类（终端） | 人类（聊天） | Agent（自动） |
| 参数传递 | argparse | shlex + argparse | JSON tool call |
| 输出格式 | 文本 / JSON | 文本 | JSON |
| 归属标识 | `--created-by` | profile author | HERMES_PROFILE |
| 最终调用 | `kb.create_task()` | `kb.create_task()` | `kb.create_task()` |

---

## 附录：任务状态机

```
                    ┌─────────────┐
     create --triage │   triage    │
     ──────────────► │             │
                    └──────┬──────┘
                           │ specify (人工/AI 充实)
                           ▼
                    ┌─────────────┐
     create         │    todo     │ ◄─── kanban_create(parents=[...])
     (默认parents) ─►│             │      (所有父任务未完成时)
                    └──────┬──────┘
                           │ recompute_ready()
                           │ (所有父任务 done)
                           ▼
                    ┌─────────────┐
                    │    ready    │ ◄─── kanban_unblock()
                    │             │      (仅编排器)
                    └──────┬──────┘
                           │ claim_task() + _default_spawn()
                           ▼
                    ┌─────────────┐
                    │   running   │ ◄─── claim_task() (终端拉取)
                    │             │
                    └──┬──┬──┬────┘
                       │  │  │
          ┌────────────┘  │  └──────────────┐
          ▼               ▼                 ▼
   ┌─────────────┐ ┌─────────────┐   ┌─────────────┐
   │   blocked   │ │    done     │   │   review    │
   │ (需人工输入) │ │ (完成)      │   │ (等待审查)  │
   └──────┬──────┘ └─────────────┘   └──────┬──────┘
          │                                   │
          │ kanban_unblock()                  │ 审查通过
          └────────► ready                    ▼
                                        ┌─────────────┐
                                        │    done     │
                                        └─────────────┘
```
