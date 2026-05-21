# Hermes Kanban：从使用入口到实现原理

> **定位**：基于 Hermes 官方 Kanban 文档与源码的**走读导读**——先说明怎么用，再沿调用链梳理实现原理。适合第一次接触 Kanban、或需要快速建立端到端心智模型的读者。
>
> **源码仓库**：`/Users/wangzhongbin/Documents/code/fangzhen/hermes-agent`
>
> **官方文档**：`hermes-agent/website/docs/user-guide/features/kanban.md`、`kanban-tutorial.md`
>
> **深度专题**：同目录 [`Hermes-Kanban完整指南.md`](./Hermes-Kanban完整指南.md)（状态机、容错、协作模式等更细展开）

---

## 目录

1. [一眼看懂](#1-一眼看懂)
2. [Kanban 是什么](#2-kanban-是什么)
3. [使用入口：三个面](#3-使用入口三个面)
4. [任务生命周期](#4-任务生命周期)
5. [从使用入口到实现的完整链路](#5-从使用入口到实现的完整链路)
6. [架构分层与源码地图](#6-架构分层与源码地图)
7. [与 delegate_task 的关系](#7-与-delegatetask-的关系)
8. [关键配置项](#8-关键配置项)
9. [延伸阅读](#9-延伸阅读)

---

## 1. 一眼看懂

| 结论 | 含义 |
|------|------|
| 持久化任务板 | 每个任务是 SQLite 里的一行，默认 `~/.hermes/kanban.db` |
| 双入口 | **人/脚本** 用 CLI、Dashboard；**Agent Worker** 用 `kanban_*` 工具 |
| 同一内核 | 所有入口都走 `hermes_cli/kanban_db.py`，读写不会漂移 |
| 调度在 Gateway | 默认 `hermes gateway start` 内嵌 dispatcher，每 60s tick 一次 |
| Worker 是子进程 | dispatcher spawn `hermes -p <profile> chat -q "work kanban task t_xxx"` |
| 与 delegate 互补 | Kanban 管跨 Profile、可恢复、可人机协作的 durable 编排 |

一句话：

```text
人创建任务 → Gateway dispatcher claim + spawn 具名 Profile Worker
→ Worker 用 kanban_* 工具读写同一 DB 完成 handoff → 依赖引擎提升子任务 → 循环。
```

---

## 2. Kanban 是什么

Hermes Kanban 是**多 Agent 协作任务板**：多个 **Profile**（具名 Agent 身份、独立配置与记忆）在同一块 SQLite 任务队列上协作，替代 `delegate_task` 那种「父 Agent 阻塞等子 Agent 返回」的 in-process 模式。

### 2.1 与 delegate_task 对比

| 维度 | `delegate_task` | Kanban |
|------|-----------------|--------|
| 形态 | RPC（fork → join） | 持久队列 + 状态机 |
| 父进程 | 阻塞直到子进程返回 | `create` 后即释放 |
| 子身份 | 匿名 subagent | 命名 Profile + 持久记忆 |
| 可恢复性 | 失败即失败 | block/unblock、crash reclaim、重试 |
| 人工介入 | 不支持 | 随时 comment / unblock |
| 审计 | 易被上下文压缩丢失 | SQLite 永久保留 |
| 协调 | 层级（caller → callee） | 对等（任意 Profile 读写任务） |

**选用原则**：

- **delegate_task**：短推理、同步依赖、无人工、结果回写父上下文。
- **Kanban**：跨 Agent 边界、需 survive 重启、可能人工审批、多角色流水线、需事后审计。

二者**可以共存**：Kanban Worker 内部仍可调 `delegate_task` 做短链路子任务。

### 2.2 核心概念

| 概念 | 说明 |
|------|------|
| **Board** | 独立任务队列 + SQLite + workspaces/logs；默认 `default`，可多 board 隔离项目 |
| **Task** | 一行任务：title、body、assignee（Profile 名）、status、tenant、workspace 等 |
| **Link** | `task_links`：parent → child 依赖；parent 全 `done` 后 child 从 `todo` 升为 `ready` |
| **Comment** | 任务线程上的 durable 注释；Worker 下次 `kanban_show` 会读到 |
| **Run** | 一次执行尝试（`task_runs`）；重试/crash/block 各有独立 run |
| **Event** | 审计流（`task_events`）；Dashboard WS、notifier、`hermes kanban watch` 都读它 |
| **Dispatcher** | 长循环：回收 stale/crash → 提升 ready → claim → spawn Worker |
| **Workspace** | `scratch`（临时目录）/ `dir:<绝对路径>` / `worktree`（git worktree） |

---

## 3. 使用入口：三个面

官方文档强调：**Board 有两个前门**——模型走工具，人走 CLI/Dashboard；背后都是同一份 `kanban.db`。

### 3.1 CLI：`hermes kanban …`

- **入口**：`hermes_cli/main.py` 注册 subcommand → `hermes_cli/kanban.py`
- **职责**：argparse、`kanban_command` 分发、人类可读输出 / `--json`
- **所有 DB 操作**：委托 `kanban_db`

快速开始（**你**执行的命令）：

```bash
hermes kanban init
hermes gateway start
hermes kanban create "research AI funding" --assignee researcher
hermes kanban list
hermes kanban watch
```

`create` 会探测 gateway 是否在跑（`_check_dispatcher_presence`），否则任务会长期停在 `ready`。

### 3.2 Slash / Gateway：`/kanban …`

- **入口**：`gateway/run.py` → `hermes_cli.kanban.run_slash`
- **与 CLI 共用**同一套 argparse 参数面
- **特性**：
  - 绕过 running-agent 守卫（Agent 在跑时仍可 `/kanban unblock`、`/kanban comment`）
  - Gateway 上 `/kanban create` 自动订阅完成通知

### 3.3 Dashboard：`hermes dashboard` → Kanban 标签

- **插件**：`plugins/kanban/dashboard/`
- **REST**：`/api/plugins/kanban/*`
- **WebSocket**：tail `task_events` 表做 live update
- **原则**：read-through DB + write-through `kanban_db`，**无独立业务逻辑**

### 3.4 Agent 工具面：`kanban_*`

Worker **不** shell 出 `hermes kanban`，而是在 Agent loop 里调工具（`tools/kanban_tools.py`）：

| 工具 | 典型角色 |
|------|----------|
| `kanban_show` | 读任务 + `worker_context` |
| `kanban_heartbeat` | 长任务存活信号 |
| `kanban_complete` / `kanban_block` | 结束或阻塞 |
| `kanban_comment` | 写评论 |
| `kanban_create` / `kanban_link` / `kanban_unblock` / `kanban_list` | Orchestrator 编排 |

**为何用工具而非 CLI**（官方文档三点）：

1. Terminal 后端可能是 Docker/SSH，容器内没有 `hermes` 也没有挂载 `kanban.db`
2. 避免 shell 引号 + `--metadata` JSON 的脆弱性
3. 结构化 JSON 错误，模型更好推理

**Toolset 激活**：

- Dispatcher spawn 时设置 `HERMES_KANBAN_TASK` → `model_tools.get_tool_definitions` **自动追加 `kanban` toolset**
- Orchestrator Profile 可在 config 里显式启用 `kanban` toolset
- 普通 `hermes chat` 无 Kanban 任务时，schema **零 `kanban_*`  footprint**

相关 Skill：

- `kanban-worker`：Worker 生命周期（bundled，dispatcher 可 `--skills kanban-worker`）
- `kanban-orchestrator`：分解、fan-out、routing（Orchestrator 专用）

---

## 4. 任务生命周期

### 4.1 状态机

```text
triage → todo → ready → running → done
                ↑         ↓
                └── blocked ──┘  (unblock → ready)
```

- **`triage`**：粗想法；默认 `auto_decompose` 由 orchestrator LLM 拆成子任务图
- **`todo`**：已创建但依赖未满足
- **`ready`**：可 claim；dispatcher 下一 tick 尝试 spawn
- **`running`**：已 claim，Worker 在执行
- **`blocked`**：需人工或熔断（`gave_up`）
- **`done` / `archived`**：完成或归档

### 4.2 三张核心表

| 表 | 职责 |
|----|------|
| `tasks` | 逻辑工作单元 + 当前状态 |
| `task_runs` | 每次 claim 一次尝试；`summary`/`metadata` 结构化 handoff |
| `task_events` | append-only 审计（`created`、`claimed`、`completed`、`crashed`…） |

另有 `task_links`、`task_comments`、`kanban_notify_subs` 等。

---

## 5. 从使用入口到实现的完整链路

> 本章节逐条命令走读源码，标注每一跳对应的文件、函数、行号与核心逻辑。

---

### 阶段 A：创建任务 — 三条入口汇入同一内核

```text
┌────────────────────────────────┐
│  入口 1: CLI                   │  hermes kanban create "..." --assignee dev
│  入口 2: /kanban (Gateway)     │  /kanban create "..." --assignee dev
│  入口 3: kanban_create (Agent) │  Worker/Orchestrator 工具调用
│  入口 4: Dashboard REST        │  POST /api/plugins/kanban/tasks
└───────────┬────────────────────┘
            │  全部委托
            ▼
   hermes_cli/kanban_db.py :: create_task()
            │
            ▼
   SQLite: tasks + task_links + task_events(created)
```

#### A1. CLI 入口：`hermes kanban create`

**调用链：** `main.py:kanban_command()` → `kanban.py:_cmd_create()` → `kanban_db.py:create_task()`

**第 1 跳：命令路由** — `hermes_cli/main.py`

`main.py` 中 argparse 注册 `kanban` 为顶级子命令，子动作 `create` 映射到 `_cmd_create`：

```python
# hermes_cli/main.py — argparse 注册（简化）
kanban_parser = subparsers.add_parser("kanban")
kanban_sub = kanban_parser.add_subparsers(dest="action")
create_p = kanban_sub.add_parser("create")
create_p.add_argument("title")
create_p.add_argument("--assignee")
create_p.add_argument("--parents", nargs="*")
# ... 最终路由到 kanban_command(args)
```

**第 2 跳：参数解析 + DB 调用** — `hermes_cli/kanban.py:_cmd_create()` (Line ~1266)

```python
def _cmd_create(args: argparse.Namespace) -> int:
    # 1) 解析 workspace 和 branch 参数
    workspace_kind, workspace_path = _parse_workspace(args)

    # 2) 解析 max_runtime（支持 "90m", "2h" 等格式）
    max_runtime_seconds = _parse_duration(args.max_runtime) if args.max_runtime else None

    # 3) 参数校验
    if args.max_retries is not None and args.max_retries < 1:
        return _err("--max-retries must be >= 1")

    # 4) ★ 核心调用：写入 SQLite
    with kb.connect() as conn:
        task_id = kb.create_task(
            conn,
            title=args.title,
            body=args.body,
            assignee=args.assignee,
            parents=tuple(args.parents or []),
            tenant=args.tenant,
            priority=args.priority,
            workspace_kind=workspace_kind,
            workspace_path=workspace_path,
            triage=args.triage,            # --triage 标志
            idempotency_key=args.idempotency_key,  # 幂等键
            max_runtime_seconds=max_runtime_seconds,
            skills=args.skills,
            max_retries=args.max_retries,
            created_by="cli",
        )
        task = kb.get_task(conn, task_id)

    # 5) 输出格式化（--json 或 人类可读）
    if args.json:
        _json(task)
    else:
        print(f"Created {task_id}")

    # 6) ★ 检查 dispatcher 是否在运行，发出警告
    if task.status == "ready" and task.assignee:
        running, msg = _check_dispatcher_presence()
        if not running:
            print(f"⚠ {msg}")  # 提醒任务会停在 ready 直到 gateway 启动
```

**第 3 跳：核心写入** — `hermes_cli/kanban_db.py:create_task()` (Line ~1341)

```python
def create_task(conn, *, title, body=None, assignee=None,
                parents=(), triage=False, idempotency_key=None,
                initial_status="running", ...) -> str:

    # ★ 幂等检查：相同 idempotency_key 的未归档任务直接返回已有 ID
    if idempotency_key:
        row = conn.execute(
            "SELECT id FROM tasks WHERE idempotency_key = ? "
            "AND status != 'archived' ORDER BY created_at DESC LIMIT 1",
            (idempotency_key,),
        ).fetchone()
        if row:
            return row["id"]  # 幂等返回，不重复创建

    # ★ 初始状态决策树
    if initial_status == "blocked":
        task_status = "blocked"
    elif triage:
        task_status = "triage"           # --triage 标志强制进入 triage
    else:
        task_status = "ready"            # 默认 ready
        if parents:                      # 有父任务依赖？
            rows = conn.execute(
                "SELECT status FROM tasks WHERE id IN "
                "(" + ",".join("?" * len(parents)) + ")",
                parents,
            ).fetchall()
            if any(r["status"] != "done" for r in rows):
                task_status = "todo"      # 父任务未完成 → 降级为 todo

    # 生成 task_id（t_ 前缀 + 8 位随机）
    task_id = f"t_{_random_id(8)}"

    # ★ 写入 tasks 表
    conn.execute(
        """INSERT INTO tasks (
            id, title, body, assignee, status, priority,
            created_by, created_at, workspace_kind, workspace_path,
            tenant, idempotency_key, max_runtime_seconds, skills, max_retries
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)""",
        (task_id, title.strip(), body, assignee, task_status, priority,
         created_by, now, workspace_kind, workspace_path, tenant,
         idempotency_key, max_runtime_seconds, json.dumps(skills_list), max_retries),
    )

    # ★ 写入 task_links（依赖关系）
    for parent_id in parents:
        conn.execute(
            "INSERT INTO task_links (parent_id, child_id) VALUES (?, ?)",
            (parent_id, task_id),
        )

    # ★ 写入 task_events（审计流）
    _append_event(conn, task_id, "created", {"assignee": assignee, "title": title})

    return task_id
```

**`create_task` 关键语义总结：**

| 条件 | 初始状态 | 后续行为 |
|------|----------|----------|
| `--triage` | `triage` | 等 dispatcher 自动分解为子任务图 |
| 有未完成 parent | `todo` | 等 parent `done` 后 `recompute_ready` 提升为 `ready` |
| 无 parent 或 parent 均 done | `ready` | 下一 tick dispatcher 可 claim + spawn |
| `idempotency_key` 重复 | 直接返回已有 ID | 不创建新行（防 webhook 重放） |

#### A2. /kanban Slash 入口（Gateway）

**调用链：** Gateway 消息处理 → `_handle_kanban_command()` → `hermes_cli/kanban.py:run_slash()`

```python
# gateway/run.py: _handle_kanban_command() (Line ~9247)
async def _handle_kanban_command(text, session, ...):
    # 1) 绕过 running-agent 守卫（Agent 在跑时仍可执行 /kanban）
    # 2) 委托给 run_slash（与 CLI 共用同一套 argparse）
    from hermes_cli.kanban import run_slash
    output = await asyncio.to_thread(run_slash, text)  # 非阻塞

    # 3) ★ 自动订阅完成通知（/kanban create 特有）
    #    从 output 解析 "Created t_abcd" 中的 task_id
    task_id = _parse_created_task_id(output)
    if task_id:
        kb.add_notify_sub(conn, task_id=task_id,
                          platform=session.platform,
                          chat_id=session.chat_id)
    return output
```

```python
# hermes_cli/kanban.py: run_slash() (Line ~2602)
def run_slash(rest: str) -> str:
    # 1) 用 shlex 安全分割（处理引号、空格）
    tokens = shlex.split(rest) if rest else []

    # 2) 构建临时 parser（prog="/kanban" 保持帮助文本一致）
    parser = _build_kanban_parser(prog="/kanban")

    # 3) 捕获 stdout/stderr，返回格式化字符串
    with io.StringIO() as out, io.StringIO() as err:
        with contextlib.redirect_stdout(out), contextlib.redirect_stderr(err):
            args = parser.parse_args(tokens)
            kanban_command(args)  # ★ 复用 CLI 同一入口
        return (out.getvalue() + err.getvalue()).strip()
```

#### A3. Agent 工具入口：`kanban_create`

**调用链：** 模型 tool_call → `tools/kanban_tools.py:kanban_create()` → `kanban_db.py:create_task()`

##### 这不是人调的命令，是 LLM 自主发起的函数调用

`kanban_create` 是注册在 Agent 工具表里的一个 **tool**（类似 OpenAI function calling）。它不由人敲命令触发，而是 LLM 在对话过程中**自主决定**调用。完整流程：

```text
┌─────────────────────────────────────────────────────────────┐
│ Agent 对话循环（run_agent.py / conversation_loop.py）       │
│                                                             │
│  1. 构造请求时，把 kanban_create 的 JSON Schema 塞进         │
│     tools 参数（模型「看到」这个工具能用）                      │
│                                                             │
│  2. 模型返回 response，其中 tool_calls 里包含：               │
│     {                                                       │
│       "type": "function",                                   │
│       "function": {                                         │
│         "name": "kanban_create",                            │
│         "arguments": "{\n  \"title\": \"调研竞品\",\n...}"  │
│       }                                                     │
│     }                                                       │
│                                                             │
│  3. conversation_loop.py 检测到 tool_calls                   │
│     → _execute_tool_calls()                                 │
│     → invoke_tool() → handle_function_call()                │
│     → registry.dispatch("kanban_create", args)              │
│     → _handle_create(args)          ← 实际执行               │
│                                                             │
│  4. 返回结果作为 tool message 喂回模型：                       │
│     {"ok": true, "task_id": "t_a1b2c3d4", "status": "ready"}│
│                                                             │
│  5. 模型看到结果，继续下一步（创建更多任务，或 complete 自己）  │
└─────────────────────────────────────────────────────────────┘
```

**模型看到的工具 Schema**（`tools/kanban_tools.py` Line ~1047）：

```json
{
  "name": "kanban_create",
  "description": "Create a new kanban task, optionally as a child of the current one (pass the current task id in parents). Used by orchestrator workers to fan out — decompose work into child tasks with specific assignees, link them into a pipeline, then complete your own task. The dispatcher picks up the new tasks on its next tick and spawns the assigned profiles.",
  "parameters": {
    "type": "object",
    "properties": {
      "title": {
        "type": "string",
        "description": "Short task title (required)."
      },
      "assignee": {
        "type": "string",
        "description": "Profile name that should execute this task (e.g. 'researcher-a', 'reviewer', 'writer'). Required — tasks without an assignee are never dispatched."
      },
      "body": {
        "type": "string",
        "description": "Opening post: full spec, acceptance criteria, links. The assigned worker reads this as part of its context."
      },
      "parents": {
        "type": "array",
        "items": {"type": "string"},
        "description": "Parent task ids. The new task stays in 'todo' until every parent reaches 'done'; then it auto-promotes to 'ready'. Typical fan-in: list all the researcher task ids when creating a synthesizer task."
      }
    },
    "required": ["title", "assignee"]
  }
}
```

**工具如何出现在模型的工具表中**（三条激活路径）：

```python
# model_tools.py (Line ~340)
# 路径 1：Dispatcher spawn Worker 时设置 HERMES_KANBAN_TASK 环境变量
if os.environ.get("HERMES_KANBAN_TASK") and "kanban" not in effective_enabled_toolsets:
    effective_enabled_toolsets.append("kanban")  # ★ 自动追加 kanban toolset

# tools/kanban_tools.py (Line ~62)
# 路径 2：Profile 配置里显式启用 kanban toolset（Orchestrator 场景）
def _check_kanban_mode() -> bool:
    if os.environ.get("HERMES_KANBAN_TASK"):
        return True                           # Worker: 环境变量触发
    return _profile_has_kanban_toolset()      # Orchestrator: 配置触发
```

```text
路径 1（Worker 激活）:
  Dispatcher spawn → 设置 HERMES_KANBAN_TASK=t_xxx
    → model_tools 检测到 → 追加 kanban toolset
    → 模型的 tools 列表里出现 kanban_show / kanban_complete / kanban_block / ...

路径 2（Orchestrator 激活）:
  Profile 配置 toolsets: ["kanban", "filesystem"]
    → _profile_has_kanban_toolset() 返回 True
    → 模型的 tools 列表里出现 kanban_create / kanban_list / kanban_unblock / ...

路径 3（普通 hermes chat）:
  无环境变量 + Profile 未配置 kanban → kanban 工具零出现
```

**一次完整的 Orchestrator tool call 示例**：

```text
用户通过 /kanban create 或 Telegram 说：
  "帮我调研市面上的 AI Agent 框架，写出对比报告"

模型（Orchestrator）收到后，思考并连续发起多个 tool_call：

Turn 1 — 模型输出:
  tool_calls: [
    {
      "name": "kanban_create",
      "arguments": {
        "title": "调研 LangGraph",
        "assignee": "researcher",
        "body": "调研 LangGraph 的核心架构、状态管理、人机协作能力...",
        "parents": []
      }
    },
    {
      "name": "kanban_create",
      "arguments": {
        "title": "调研 CrewAI",
        "assignee": "researcher",
        "body": "调研 CrewAI 的角色定义、任务编排、工具集成...",
        "parents": []
      }
    },
    {
      "name": "kanban_create",
      "arguments": {
        "title": "汇总对比报告",
        "assignee": "writer",
        "body": "基于调研结果，撰写 AI Agent 框架对比报告...",
        "parents": ["t_aaaa1111", "t_bbbb2222"]  // ★ fan-in：等两个调研都完成
      }
    }
  ]

Agent 循环处理:
  registry.dispatch("kanban_create", {"title": "调研 LangGraph", ...})
    → _handle_create() → kb.create_task() → return "t_aaaa1111"
  registry.dispatch("kanban_create", {"title": "调研 CrewAI", ...})
    → _handle_create() → kb.create_task() → return "t_bbbb2222"
  registry.dispatch("kanban_create", {"title": "汇总对比报告", ...})
    → _handle_create() → kb.create_task() → return "t_cccc3333"

三个 tool message 喂回模型:
  {"ok": true, "task_id": "t_aaaa1111", "status": "ready"}
  {"ok": true, "task_id": "t_bbbb2222", "status": "ready"}
  {"ok": true, "task_id": "t_cccc3333", "status": "todo"}    // todo，等 parent 完成

Turn 2 — 模型输出:
  tool_calls: [{
    "name": "kanban_complete",
    "arguments": {"summary": "已创建 3 个任务：2 个并行调研 + 1 个汇总"}
  }]
  // Orchestrator 自己完成，退出
```

**谁会调用？** 两类角色：

| 角色 | 场景 | 典型用法 |
|------|------|----------|
| **Orchestrator** | 构建任务 DAG、拆解需求、fan-out 编排 | 一次性创建多个任务 + `parents` 串联依赖 |
| **Worker** | 执行中发现衍生工作（如发现 bug 需单独修复） | 创建 follow-up task，自己继续当前任务 |

**参数说明：**

```python
# 工具 Schema 定义（tools/kanban_tools.py）
kanban_create(
    title,                  # 必填：任务标题
    assignee,               # 必填：目标 Profile 名（谁来做）
    body=None,              # 可选：详细描述/需求/约束
    parents=None,           # 可选：父任务 ID 列表（依赖关系，支持 string 或 list）
    tenant=None,            # 可选：租户隔离（不传则回退 HERMES_TENANT 环境变量）
    priority=0,             # 可选：优先级（数值越大越优先被 dispatcher claim）
    skills=None,            # 可选：Worker 需加载的额外 skills
    workspace_kind=None,    # 可选：scratch / dir:<path> / worktree
    workspace_path=None,    # 可选：工作目录绝对路径
    triage=False,           # 可选：是否进入 triage（等自动分解）
    idempotency_key=None,   # 可选：幂等键（防重复创建）
    max_runtime_seconds=None, # 可选：超时时间
    initial_status="running", # 可选：初始状态（一般不传，由内核决策）
)
```

**源码处理流程：**

```python
# tools/kanban_tools.py: kanban_create() (Line ~639)
def kanban_create(title, assignee, body=None, parents=None,
                  tenant=None, priority=0, skills=None, ...):
    # 1) 必填字段校验
    if not title:
        return tool_error("title is required")
    if not assignee:
        return tool_error("assignee is required")

    # 2) parents 参数归一化（支持 string 或 list 两种传法）
    if isinstance(parents, str):
        parents = [p.strip() for p in parents.split(",")]
    parents = [p for p in (parents or []) if p]

    # 3) tenant 回退到环境变量
    tenant = tenant or os.environ.get("HERMES_TENANT")

    # 4) session_id 追踪来源
    session_id = os.environ.get("HERMES_SESSION_ID")

    # 5) ★ 调用内核
    with kb.connect() as conn:
        new_tid = kb.create_task(
            conn,
            title=title.strip(),
            body=body,
            assignee=str(assignee),
            parents=tuple(parents),
            created_by=os.environ.get("HERMES_PROFILE") or "worker",  # ★ 自动记录创建者
            session_id=session_id,
            ...
        )

    return _ok(task_id=new_tid, title=title)
```

**典型使用模式：**

**模式 1：Orchestrator 构建任务 DAG（最常见）**

Orchestrator 一次性拆解需求为多任务，用 `parents` 建立依赖链：

```text
用户需求：实现用户登录功能
    ↓ Orchestrator 拆解

kanban_create("设计数据库 schema", assignee="dba")
    → t_001 (ready)

kanban_create("实现后端 API", assignee="backend", parents=["t_001"])
    → t_002 (todo，等 t_001 完成)

kanban_create("编写前端页面", assignee="frontend", parents=["t_001"])
    → t_003 (todo，等 t_001 完成)

kanban_create("集成测试", assignee="tester", parents=["t_002", "t_003"])
    → t_004 (todo，等 t_002 和 t_003 都完成)

形成的 DAG:
    t_001 → t_002 → ┐
    t_001 → t_003 → ┤→ t_004
```

`t_001` 完成 → `recompute_ready` 提升 `t_002` 和 `t_003` 为 ready → dispatcher 并行 spawn 两个 Worker → 两者都完成 → 提升 `t_004`。

**模式 2：Worker 发现衍生工作**

Worker 在执行过程中发现需要额外任务：

```text
Worker 执行 t_002（实现后端 API）:
  1. kanban_show() → 读取任务要求
  2. 代码编写过程中发现需要先修复一个依赖库的 bug
  3. kanban_create(
       "修复 auth 库 JWT 过期问题",
       assignee="backend",
       parents=["t_002"],     # ★ 当前任务作为 parent → t_002 完成前衍生任务也会执行
     )
     → t_005 创建成功，Worker 继续当前工作
  4. 继续完成 t_002 的开发
  5. kanban_complete(summary="API 已实现，发现并创建了修复任务 t_005")
```

**模式 3：Fan-out 并行分发**

Orchestrator 将一个大任务拆分为多个独立并行子任务：

```text
kanban_create("调研竞品 A", assignee="researcher")
kanban_create("调研竞品 B", assignee="researcher")
kanban_create("调研竞品 C", assignee="researcher")

三个任务都是 ready，dispatcher 可并行 spawn 三个 Worker。
```

**模式 4：Fan-in 汇聚**

多个任务的结果汇聚到一个汇总任务（通过多 parent 实现）：

```text
kanban_create("调研 A", assignee="r1")  → t_010
kanban_create("调研 B", assignee="r2")  → t_011
kanban_create("汇总报告", assignee="writer", parents=["t_010", "t_011"])  → t_012

t_012 的初始状态为 todo。
只有 t_010 和 t_011 都 done 后，recompute_ready 才提升 t_012 为 ready。
Worker 做 t_012 时，build_worker_context 会自动包含 t_010 和 t_011 的 summary 作为 handoff。
```

#### A4. 关键澄清：Worker 调 `kanban_create` 不是 "Agent 套 Agent"

初看之下，Worker（本身是一个 Agent）调用 `kanban_create` 创建新任务，再由另一个 Agent 执行，似乎是嵌套。**实际不是**——二者通过 SQLite 解耦为独立进程：

```text
Worker A (进程 1)                          Dispatcher (Gateway 进程)
    │                                          │
    │ ① kanban_create()                        │
    │    → kanban_db.create_task()             │
    │    → INSERT INTO tasks (status='ready')  │ ② 下一个 tick: dispatch_once()
    │    → return task_id                      │    → claim_task() CAS 抢锁
    │    → ★ 函数返回，Worker A 继续干活        │    → _default_spawn()
    │                                          │
    │                              ─────────────────────────────
    │                              │ Worker B (进程 2)          │
    │                              │ hermes -p dev2 chat ...    │
    │                              │ 全新的 Agent，独立的对话循环  │
    │                              ─────────────────────────────
```

**为什么不是嵌套：**

1. **`kanban_create` 只写 DB**：没有任何 `subprocess.Popen` 或 `delegate_task` 调用，函数返回后 Worker A 立即继续自己的对话循环
2. **spawn 是 Dispatcher 的职责**：Gateway 内嵌的 `_kanban_dispatcher_watcher` 每 60s tick 一次，`dispatch_once()` 才会 claim + spawn Worker B
3. **Worker A 和 B 是平级进程**：都是 Gateway spawn 的子进程，互不知道对方存在，通过 SQLite 共享状态

```text
Gateway (主进程)
  ├── Worker A (PID 1001) ← 执行 task t_abc，过程中 kanban_create 了 t_def
  ├── Worker B (PID 1002) ← dispatcher 下个 tick spawn，执行 t_def
  └── Worker C (PID 1003) ← 执行 t_gihi（可能是 B 创建的）
```

**对比 `delegate_task` 才是真正的嵌套：**

```text
delegate_task（阻塞嵌套）:                kanban_create（共享队列解耦）:
  Agent A (主循环)                          Agent A → INSERT task → 继续干活 → complete
    └── Agent B (子循环)                    Agent B ← dispatcher claim ← 独立执行 → complete
          └── Agent C (子子循环)            Agent C ← dispatcher claim ← 独立执行 → complete
        └── B 等 C 返回                    三者互不阻塞，通过 SQLite handoff 结果
    └── A 等 B 返回
```

这正是 Kanban 的核心设计：**`kanban_create` 只是往共享队列投递了一条记录，Dispatcher 在未来某个 tick 独立 spawn 全新 Agent 消费它——非阻塞、可恢复、跨进程。**

#### A5. SDK 集成：外部工程如何触发 Kanban 能力

外部项目有三种集成方式，按耦合程度从低到高：

```text
┌──────────────────────────────────────────────────────────┐
│  方式 1: REST API（HTTP）     — 语言无关，跨网络         │
│  方式 2: CLI 子进程           — 最简单，shell 即可        │
│  方式 3: 直接 import kanban_db — Python 进程内调用，最高效│
└──────────────────────────────────────────────────────────┘
```

**Gateway 依赖说明：**

```text
┌─────────────────┬──────────────────────────────────────────────────────┐
│ 操作             │ 需要 Gateway？                                       │
├─────────────────┼──────────────────────────────────────────────────────┤
│ 创建/查询/完成    │ ❌ 不需要。全部是纯 SQLite 读写                       │
│ 阻塞/解封/评论    │ ❌ 不需要。纯 SQLite 操作                             │
│ Dashboard REST   │ ❌ 不需要 Gateway，但需 Dashboard 服务（可独立启动）   │
│ 自动调度 Worker   │ ✅ 需要。Dispatcher 内嵌在 Gateway 中，负责 claim +  │
│                  │    spawn 子进程。无 Gateway 时任务停在 ready 无人消费  │
└─────────────────┴──────────────────────────────────────────────────────┘
```

源码证据：

- `_cmd_create()` 中 `_check_dispatcher_presence()` **只打印警告，不阻塞**（`kanban.py:1315-1325`）
- `_check_dispatcher_presence()` 在探测失败时返回 `(True, "")` 即**静默通过**（`kanban.py:159`）
- `kanban_db.connect()` + `kanban_db.create_task()` 是纯 `sqlite3.connect` + `INSERT`，无任何网络调用
- Dashboard REST API 的所有 handler 都只调 `kanban_db.*`，可由 `hermes dashboard` 独立启动

**简单来说**：CRUD 操作是纯 SQLite，离线也能用；只有「自动派 Agent 干活」才需要 Gateway 里的 Dispatcher。

---

##### 方式 1：REST API（Dashboard 插件暴露的 HTTP 接口）

**源码入口**：`plugins/kanban/dashboard/plugin_api.py`，挂载在 `/api/plugins/kanban/`

适用于任何语言的项目，只要能发 HTTP 请求。

```python
"""
外部工程通过 REST API 使用 Hermes Kanban
前置：hermes gateway start（Dashboard API 默认随 Gateway 启动）
"""
import requests

BASE = "http://localhost:8787"  # Gateway 默认端口
API = f"{BASE}/api/plugins/kanban"

# ── 1. 创建任务 ──────────────────────────────────────
resp = requests.post(f"{API}/tasks", json={
    "title": "分析用户行为数据",
    "body": "提取最近 7 天的活跃用户行为特征，生成分析报告",
    "assignee": "data-analyst",         # 目标 Profile 名
    "tenant": "team-alpha",             # 可选：租户隔离
    "priority": 5,                      # 可选：优先级（数值越大越优先）
    "parents": [],                      # 可选：父任务 ID 列表
    "triage": False,                    # 可选：是否自动分解
    "idempotency_key": "daily-report-20260520",  # 可选：幂等键
    "max_runtime_seconds": 3600,        # 可选：超时 1 小时
    "skills": ["sql-analyst"],          # 可选：Worker 加载的额外 skills
})
task = resp.json()["task"]
task_id = task["id"]  # e.g. "t_a1b2c3d4"
print(f"Created: {task_id}, status: {task['status']}")

# ── 2. 构建任务 DAG（带依赖关系）──────────────────────
# 先创建无依赖的任务
r1 = requests.post(f"{API}/tasks", json={
    "title": "数据清洗",
    "assignee": "data-engineer",
}).json()
r2 = requests.post(f"{API}/tasks", json={
    "title": "特征工程",
    "assignee": "data-scientist",
    "parents": [r1["task"]["id"]],       # ★ 依赖数据清洗完成
}).json()
r3 = requests.post(f"{API}/tasks", json={
    "title": "模型训练",
    "assignee": "ml-engineer",
    "parents": [r2["task"]["id"]],       # ★ 依赖特征工程完成
}).json()

# ── 3. 查看任务状态 ──────────────────────────────────
resp = requests.get(f"{API}/tasks/{task_id}")
task_detail = resp.json()
# task_detail 包含: task, comments, events, links, runs

# ── 4. 查看整个看板 ──────────────────────────────────
board = requests.get(f"{API}/board").json()
for column in board["columns"]:
    print(f"{column['status']}: {len(column['tasks'])} tasks")

# ── 5. 更新任务（人工干预）───────────────────────────
requests.patch(f"{API}/tasks/{task_id}", json={
    "status": "blocked",
    "block_reason": "需要产品经理确认分析维度",
})

# ── 6. 解除阻塞 ──────────────────────────────────────
requests.patch(f"{API}/tasks/{task_id}", json={
    "status": "ready",  # unblock → ready 或 todo（取决于 parent 状态）
})

# ── 7. 添加评论（跨任务 handoff）─────────────────────
requests.post(f"{API}/tasks/{task_id}/comments", json={
    "author": "pm",
    "body": "分析维度确认：DAU、留存率、漏斗转化，按城市拆分",
})

# ── 8. 批量操作 ──────────────────────────────────────
requests.post(f"{API}/tasks/bulk", json={
    "ids": [r1["task"]["id"], r2["task"]["id"]],
    "assignee": "new-analyst",   # 批量重分配
    "priority": 3,
})

# ── 9. WebSocket 实时监听事件 ────────────────────────
import websocket
ws = websocket.WebSocketApp(
    f"ws://localhost:8787/api/plugins/kanban/events?since=0&token=YOUR_TOKEN",
    on_message=lambda ws, msg: print(f"Event: {msg}"),
)
ws.run_forever()
```

---

##### 方式 2：CLI 子进程

最简单的集成方式，适合 shell 脚本、CI/CD pipeline、cron job。

```python
"""
外部工程通过 CLI 子进程使用 Hermes Kanban
前置：pip install hermes-agent
"""
import subprocess
import json

def hermes_kanban(*args) -> str:
    """封装 hermes kanban 命令"""
    result = subprocess.run(
        ["hermes", "kanban", *args],
        capture_output=True, text=True,
    )
    if result.returncode != 0:
        raise RuntimeError(f"hermes kanban failed: {result.stderr}")
    return result.stdout.strip()

# ── 1. 初始化 ──────────────────────────────────────
hermes_kanban("init")

# ── 2. 创建任务 ──────────────────────────────────────
output = hermes_kanban(
    "create", "实现用户登录功能",
    "--assignee", "developer",
    "--body", "支持邮箱和手机号登录，需要 OAuth2.0",
    "--priority", "5",
    "--max-runtime", "2h",
    "--skills", "web-dev",
)
# output: "Created t_a1b2c3d4"
task_id = output.split()[-1]

# ── 3. 查看任务列表（--json 格式方便解析）──────────────
output = hermes_kanban("list", "--status", "ready", "--json")
tasks = json.loads(output)
for t in tasks:
    print(f"  {t['id']}: {t['title']} → {t['assignee']}")

# ── 4. 创建带依赖的任务 ──────────────────────────────
hermes_kanban(
    "create", "集成测试",
    "--assignee", "tester",
    "--parents", task_id,          # 依赖上面的任务
)

# ── 5. 阻塞任务 + 原因 ──────────────────────────────
hermes_kanban("block", task_id, "--reason", "需要 UI 设计稿")

# ── 6. 添加评论 ──────────────────────────────────────
hermes_kanban("comment", task_id, "--body", "设计稿已确认，可以开始")

# ── 7. 解除阻塞 ──────────────────────────────────────
hermes_kanban("unblock", task_id)

# ── 8. 完成任务 ──────────────────────────────────────
hermes_kanban(
    "complete", task_id,
    "--result", "登录功能已实现，PR #42",
)

# ── 9. 实时监听 ──────────────────────────────────────
# 这会阻塞，持续打印事件流
# hermes_kanban("watch")
```

---

##### 方式 3：直接 import `kanban_db`（Python 进程内调用）

最高效的方式，跳过 HTTP/CLI 中间层，直接操作 SQLite。适合 Python 项目内部集成。

```python
"""
外部工程直接 import kanban_db 使用 Hermes Kanban
前置：pip install hermes-agent（或把 hermes-agent 加入 PYTHONPATH）
"""
from hermes_cli import kanban_db as kb

# ── 1. 连接数据库 ──────────────────────────────────
# 默认连接 ~/.hermes/kanban.db
conn = kb.connect()

# 也可以指定 board（会连接 ~/.hermes/kanban/<board>.db）
conn = kb.connect(board="my-project")

# 或者显式指定路径
from pathlib import Path
conn = kb.connect(db_path=Path("/data/kanban.db"))

# ── 2. 创建任务 ──────────────────────────────────
task_id = kb.create_task(
    conn,
    title="重构认证模块",
    body="将 session 认证迁移到 JWT，保持向后兼容",
    assignee="backend-dev",
    created_by="ci-pipeline",        # ★ 标记来源
    priority=3,
    tenant="platform-team",
    max_runtime_seconds=7200,        # 2 小时超时
    max_retries=3,                   # 最多重试 3 次
    idempotency_key="refactor-auth-v2",  # 幂等：重复调用不重复创建
    skills=["code-review"],
)
print(f"Created: {task_id}")  # e.g. "t_e5f6g7h8"

# ── 3. 构建任务 DAG ──────────────────────────────
# 第 1 层：无依赖，立即 ready
design_id = kb.create_task(conn, title="设计新 API", assignee="architect")

# 第 2 层：依赖设计完成
impl_id = kb.create_task(
    conn, title="实现 API", assignee="backend",
    parents=[design_id],  # ★ todo，等 design_id 完成后自动提升为 ready
)

# 第 3 层：依赖实现 + 另一个并行任务
test_data_id = kb.create_task(conn, title="准备测试数据", assignee="qa")
test_id = kb.create_task(
    conn, title="集成测试", assignee="qa",
    parents=[impl_id, test_data_id],  # ★ fan-in：两者都完成才 ready
)

# ── 4. 查询任务 ──────────────────────────────────
# 单个任务
task = kb.get_task(conn, task_id)
print(f"Status: {task.status}, Assignee: {task.assignee}")

# 按状态筛选
ready_tasks = kb.list_tasks(conn, status="ready")
my_tasks = kb.list_tasks(conn, assignee="backend-dev")

# ── 5. 任务生命周期操作 ──────────────────────────
# 阻塞（需要人工审批）
kb.block_task(conn, task_id, reason="等待安全团队 review")

# 添加评论（跨任务 handoff 信息）
kb.add_comment(conn, task_id, author="security", body="已确认，可以继续")

# 解除阻塞
kb.unblock_task(conn, task_id)
# unblock 会自动检查 parent 状态：
#   - parent 都 done → ready（可被 dispatcher claim）
#   - parent 未 done → todo（继续等）

# 完成（由外部系统代替 Worker 完成时使用）
kb.complete_task(
    conn, task_id,
    result="认证模块已重构，所有测试通过",
    summary="JWT 迁移完成，向后兼容已验证",
    metadata={"pr_url": "https://github.com/.../pull/42", "files_changed": 23},
)
# complete 后会自动调用 recompute_ready → 提升依赖此任务的子任务

# ── 6. 完整示例：CI/CD pipeline 创建任务 ──────────────
def create_deploy_task(env: str, commit_sha: str, parent_ids: list[str] = None):
    """CI pipeline 完成后自动创建部署任务"""
    conn = kb.connect()
    task_id = kb.create_task(
        conn,
        title=f"部署 {env} 环境",
        body=f"Commit: {commit_sha}\n自动创建于 CI pipeline",
        assignee="devops",
        created_by="ci-cd-pipeline",
        parents=tuple(parent_ids or []),
        idempotency_key=f"deploy-{env}-{commit_sha[:8]}",  # 防重复
        max_runtime_seconds=1800,
        skills=["deploy"],
    )
    kb.add_comment(
        conn, task_id,
        author="ci-cd-pipeline",
        body=f"Triggered by commit {commit_sha}\n"
             f"Pipeline: https://ci.example.com/build/{commit_sha}",
    )
    conn.close()
    return task_id

# 使用
task_id = create_deploy_task("staging", "abc123def456")
print(f"Deploy task: {task_id}, waiting for dispatcher to spawn worker")
```

---

##### 三种方式对比

| 维度 | REST API | CLI 子进程 | import kanban_db |
|------|----------|-----------|------------------|
| **语言要求** | 任何语言 | 任何语言（需 shell） | 仅 Python |
| **网络要求** | 无（纯 SQLite），但需 Dashboard HTTP 服务 | 无 | 无 |
| **性能** | HTTP 开销 | 进程创建开销 | 进程内直连，最快 |
| **适用场景** | 微服务、Web hook、远程触发 | CI/CD、cron、shell 脚本 | Python 项目内部、批量操作 |
| **事务支持** | 无（每次一个操作） | 无 | 有（`write_txn` 上下文） |
| **Gateway 依赖** | 不需要 Gateway，但需 Dashboard 服务 | 不需要 | 不需要 |
| **Board 隔离** | `?board=xxx` | `--board xxx` | `connect(board="xxx")` |

> **注意**：方式 3 直接操作 SQLite 时，如果 Gateway 的 Dispatcher 也在跑，两者通过 **SQLite WAL 模式 + 事务** 保证并发安全。你写入的任务和 Dispatcher 读到的状态不会冲突。

---

### 阶段 B：Dispatcher 调度 — Gateway 内嵌的长循环

```text
hermes gateway start
        ↓
gateway/run.py :: _kanban_dispatcher_watcher()        ← 长驻 async task
        ↓
每 dispatch_interval_seconds（默认 60s）一个 tick:
  ┌─────────────────────────────────────────────────┐
  │ 1. auto_decompose（可选，处理 triage 任务）       │
  │ 2. 枚举所有 board                                │
  │ 3. 对每个 board: asyncio.to_thread(dispatch_once)│
  └─────────────────────────────────────────────────┘
        ↓ (per board, in thread)
kanban_db.py :: dispatch_once()
```

#### B1. Gateway Dispatcher Watcher

**源码：** `gateway/run.py: _kanban_dispatcher_watcher()` (Line ~4994)

```python
async def _kanban_dispatcher_watcher(hub, config):
    # 1) 启动延迟 5s，等 gateway 完全就绪
    await asyncio.sleep(5)

    # 2) 读取配置
    interval = config.kanban.dispatch_interval_seconds  # 默认 60s
    failure_limit = config.kanban.failure_limit          # 默认 2
    max_spawn = config.kanban.max_spawn                  # None = 不限

    while not shutdown_event.is_set():
        try:
            # 3) ★ auto_decompose：处理 triage 状态的任务
            if config.kanban.auto_decompose:
                boards = await asyncio.to_thread(_kb.list_boards)
                for slug in boards:
                    triage_rows = await asyncio.to_thread(
                        _kb.list_tasks, conn, status="triage",
                        limit=config.kanban.auto_decompose_per_tick  # 默认 3
                    )
                    for task in triage_rows:
                        # 调用 LLM 分解为子任务图
                        await _auto_decompose(task, config)

            # 4) ★ 对每个 board 执行 dispatch_once
            boards = await asyncio.to_thread(_kb.list_boards)
            for slug in boards:
                result = await asyncio.to_thread(
                    _tick_once_for_board, slug,
                    max_spawn=max_spawn,
                    failure_limit=failure_limit,
                    ...
                )
                # 记录 telemetry

        except Exception as e:
            log.exception("dispatcher tick failed")

        # 5) ★ 分片 sleep：每秒检查一次 shutdown，响应更快
        for _ in range(interval):
            if shutdown_event.is_set():
                break
            await asyncio.sleep(1)
```

#### B2. `dispatch_once` — 单 Tick 六步流水线

**源码：** `hermes_cli/kanban_db.py: dispatch_once()` (Line ~4649)

```python
def dispatch_once(conn, *, max_spawn=None, max_in_progress=None,
                  failure_limit=2, stale_timeout_seconds=0, board=None):
    result = DispatchResult()

    # ─────────────────────────────────────
    # Step 1: 回收僵尸子进程（Unix only）
    # ─────────────────────────────────────
    if os.name != "nt":
        try:
            while True:
                _pid, _status = os.waitpid(-1, os.WNOHANG)
                if _pid == 0:
                    break
                _record_worker_exit(_pid, _status)  # 记录退出状态
        except ChildProcessError:
            pass

    # ─────────────────────────────────────
    # Step 2: 释放过期 claim
    # ─────────────────────────────────────
    # 查找 claim_expires < now 的 running 任务
    # 如果 worker PID 仍存活 → 续期而非回收
    # 如果 PID 已死 → SIGTERM → 重置为 ready
    result.reclaimed = release_stale_claims(conn)

    # ─────────────────────────────────────
    # Step 3: 检测无 heartbeat 的 running 任务
    # ─────────────────────────────────────
    # stale_timeout_seconds 内无 heartbeat → 视为失联
    # 终止进程 → 重置为 ready
    result.stale = detect_stale_running(
        conn, stale_timeout_seconds=stale_timeout_seconds,
    )

    # ─────────────────────────────────────
    # Step 4: 检测已崩溃 worker
    # ─────────────────────────────────────
    # PID 不存在 → 分两类：
    #   - clean_exit (rc=0): protocol_violation（未调 complete/block 就退出）
    #   - 非 0 退出: crashed
    # 两者都记录失败次数，达到 failure_limit → gave_up → auto-block
    result.crashed = detect_crashed_workers(conn)

    # ─────────────────────────────────────
    # Step 5: 超时强制结束
    # ─────────────────────────────────────
    # 运行时间超过 max_runtime_seconds → SIGTERM → ready
    result.timed_out = enforce_max_runtime(conn)

    # ─────────────────────────────────────
    # Step 6: 依赖提升 todo → ready
    # ─────────────────────────────────────
    # 对每个 todo 任务检查所有 parent 是否 done
    # 全部 done → 提升为 ready
    result.promoted = recompute_ready(conn)

    # ─────────────────────────────────────
    # Step 7: Claim + Spawn
    # ─────────────────────────────────────
    ready_rows = conn.execute(
        "SELECT id, assignee FROM tasks "
        "WHERE status = 'ready' AND claim_lock IS NULL "
        "ORDER BY priority DESC, created_at ASC"  # ★ 优先级高、先创建的优先
    ).fetchall()

    spawned = 0
    for row in ready_rows:
        # 并发限制检查
        if max_spawn is not None and running_count + spawned >= max_spawn:
            break
        if max_in_progress is not None and running_count >= max_in_progress:
            break

        # 连续失败保护（gave_up 状态的任务不再 spawn）
        guard_reason = check_respawn_guard(conn, row["id"])
        if guard_reason is not None:
            result.respawn_guarded.append((row["id"], guard_reason))
            continue

        # ★ CAS claim：原子性地抢锁
        claimed = claim_task(conn, row["id"], ttl_seconds=ttl_seconds)
        if claimed is None:
            continue  # 被其他 dispatcher 抢了

        # 解析 workspace 路径
        workspace = resolve_workspace(claimed, board=board)
        kb.set_workspace_path(conn, claimed.id, str(workspace))

        # ★ Spawn 子进程
        pid = _spawn(claimed, str(workspace), board=board)
        if pid:
            _set_worker_pid(conn, claimed.id, int(pid))
            result.spawned.append((claimed.id, claimed.assignee, str(workspace)))
            spawned += 1

    return result
```

#### B3. `claim_task` — CAS 原子抢锁

**源码：** `hermes_cli/kanban_db.py: claim_task()` (Line ~2103)

```python
def claim_task(conn, task_id, *, ttl_seconds=None, claimer=None):
    now = int(time.time())
    lock = claimer or _claimer_id()          # "hostname:pid" 格式
    expires = now + _resolve_claim_ttl_seconds(ttl_seconds)  # 默认 4h

    # ★ 安全检查：parent 是否全部 done？
    undone = conn.execute(
        "SELECT 1 FROM task_links l "
        "JOIN tasks p ON p.id = l.parent_id "
        "WHERE l.child_id = ? AND p.status NOT IN ('done', 'archived') LIMIT 1",
        (task_id,),
    ).fetchone()
    if undone:
        # 降级回 todo（parent 没完成不该被 claim）
        conn.execute(
            "UPDATE tasks SET status = 'todo' WHERE id = ? AND status = 'ready'",
            (task_id,),
        )
        return None

    # ★★★ CAS UPDATE — 原子抢锁的核心 ★★★
    cur = conn.execute(
        """
        UPDATE tasks
           SET status        = 'running',
               claim_lock    = ?,          -- 写入锁标识
               claim_expires = ?,          -- 锁过期时间
               started_at    = COALESCE(started_at, ?)
         WHERE id = ?
           AND status = 'ready'            -- 条件 1：必须是 ready
           AND claim_lock IS NULL           -- 条件 2：必须无锁
        """,
        (lock, expires, now, task_id),
    )

    # rowcount == 1 表示抢锁成功；0 表示被别人抢了或状态已变
    if cur.rowcount != 1:
        return None  # Lost the race

    # 创建 task_runs 行（记录本次执行尝试）
    run_cur = conn.execute(
        """
        INSERT INTO task_runs (
            task_id, profile, status, claim_lock, claim_expires,
            max_runtime_seconds, started_at
        ) VALUES (?, ?, 'running', ?, ?, ?, ?)
        """,
        (task_id, task.assignee, lock, expires, task.max_runtime_seconds, now),
    )
    conn.execute(
        "UPDATE tasks SET current_run_id = ? WHERE id = ?",
        (run_cur.lastrowid, task_id),
    )

    _append_event(conn, task_id, "claimed", {"claim_lock": lock})
    return task  # 返回完整 Task 对象
```

**CAS 语义：** `WHERE status='ready' AND claim_lock IS NULL` 是「检查并设置」原子操作——SQLite WAL 模式 + `BEGIN IMMEDIATE` 事务保证同一时刻只有一个 dispatcher 能成功 `rowcount=1`。

#### B4. `recompute_ready` — 依赖满足后自动提升

**源码：** `hermes_cli/kanban_db.py: recompute_ready()` (Line ~2043)

```python
def recompute_ready(conn):
    promoted = 0
    with write_txn(conn):
        # 扫描所有 todo / blocked 任务
        todo_rows = conn.execute(
            "SELECT id, status FROM tasks WHERE status IN ('todo', 'blocked')"
        ).fetchall()

        for row in todo_rows:
            # ★ 跳过 sticky-block（人工手动 block 的不自动提升）
            if row["status"] == "blocked" and _has_sticky_block(conn, row["id"]):
                continue

            # 检查所有 parent 的状态
            parents = conn.execute(
                "SELECT t.status FROM tasks t "
                "JOIN task_links l ON l.parent_id = t.id "
                "WHERE l.child_id = ?",
                (row["id"],),
            ).fetchall()

            # ★ 全部 parent 都 done/archived → 提升
            if all(p["status"] in ("done", "archived") for p in parents):
                conn.execute(
                    "UPDATE tasks SET status = 'ready', "
                    "consecutive_failures = 0, last_failure_error = NULL "
                    "WHERE id = ?",
                    (row["id"],),
                )
                _append_event(conn, row["id"], "promoted", None)
                promoted += 1
    return promoted
```

---

### 阶段 C：Spawn Worker — 从 Claim 到子进程启动

#### C1. `_default_spawn` — 构造子进程命令

**源码：** `hermes_cli/kanban_db.py: _default_spawn()` (Line ~5214)

```python
def _default_spawn(task, workspace, *, board=None):
    import subprocess

    # ── 1. 构造环境变量 ──────────────────
    env = dict(os.environ)
    env["HERMES_HOME"] = resolve_profile_env(task.assignee)  # ★ 指向 assignee 的 Profile
    env["HERMES_KANBAN_TASK"] = task.id                       # ★ Worker 作用域
    env["HERMES_KANBAN_WORKSPACE"] = workspace                # 工作目录
    env["HERMES_KANBAN_RUN_ID"] = str(task.current_run_id)    # 当前 run
    env["HERMES_KANBAN_DB"] = str(kanban_db_path(board))      # ★ 锁定 DB 路径
    env["HERMES_KANBAN_BOARD"] = resolved_board               # 锁定 board
    if task.claim_lock:
        env["HERMES_KANBAN_CLAIM_LOCK"] = task.claim_lock     # 心跳续期用
    if task.tenant:
        env["HERMES_TENANT"] = task.tenant

    # ── 2. 构造命令行 ──────────────────
    cmd = [*_resolve_hermes_argv(), "-p", task.assignee, "--accept-hooks"]

    # 注入 kanban-worker skill（如果可用）
    if _kanban_worker_skill_available(env["HERMES_HOME"]):
        cmd.extend(["--skills", "kanban-worker"])

    # 注入任务指定的额外 skills
    for sk in (task.skills or []):
        if sk and sk != "kanban-worker":
            cmd.extend(["--skills", sk])

    # 模型覆盖（如果任务指定了特定模型）
    if task.model_override:
        cmd.extend(["-m", task.model_override])

    # ★ 最终命令：chat 模式 + 初始 prompt
    cmd.extend(["chat", "-q", f"work kanban task {task.id}"])

    # ── 3. 日志设置 ──────────────────
    log_path = worker_logs_dir(board) / f"{task.id}.log"
    log_f = open(log_path, "ab")

    # ── 4. 启动子进程 ──────────────────
    proc = subprocess.Popen(
        cmd,
        cwd=workspace,                    # ★ 工作目录 = workspace
        stdin=subprocess.DEVNULL,          # 无交互输入
        stdout=log_f,                      # ★ stdout 重定向到日志文件
        stderr=subprocess.STDOUT,          # stderr 合并到 stdout
        env=env,
        start_new_session=True,            # ★ 独立进程组（不受父进程 SIGINT）
    )

    return proc.pid
```

**最终生成的命令示例：**

```bash
hermes -p developer --accept-hooks --skills kanban-worker \
       chat -q "work kanban task t_a1b2c3d4"
```

#### C2. Worker 启动后的工具面激活

Worker 子进程启动后，Hermes Agent 初始化流程中有两个关键注入点：

**注入点 1：工具注册** — `model_tools.py`

```python
# model_tools.py :: get_tool_definitions()
# 检测环境变量 → 自动追加 kanban toolset
kanban_task = os.environ.get("HERMES_KANBAN_TASK")
if kanban_task:
    # ★ 注入 kanban_show / kanban_complete / kanban_block / kanban_heartbeat 等
    toolsets.append("kanban")
```

**注入点 2：系统提示词** — `agent/prompt_builder.py`

```python
# agent/prompt_builder.py
# 如果工具列表包含 kanban_show → 注入 KANBAN_GUIDANCE
if "kanban_show" in tool_names:
    system_prompt += KANBAN_GUIDANCE
    # 指导 Worker：先 kanban_show 读任务 → 干活 → kanban_complete/kanban_block
```

---

### 阶段 D：Worker 执行循环 — 模型的 Tool Call 链

Worker 是一个标准 Hermes Agent 对话循环，模型通过 tool call 与 Kanban 系统交互。

#### D1. `kanban_show` — 读取任务 + 组装上下文

**调用链：** 模型 tool_call → `kanban_tools.py:kanban_show()` → `kanban_db.py:build_worker_context()`

```python
# tools/kanban_tools.py: kanban_show() (Line ~255)
def kanban_show(task_id=None, board=None):
    # 1) task_id 回退到环境变量（Worker 通常不显式传 task_id）
    tid = task_id or os.environ.get("HERMES_KANBAN_TASK")
    if not tid:
        return tool_error("No task_id and not in worker context")

    with kb.connect() as conn:
        task = kb.get_task(conn, tid)

        # 2) 组装完整上下文
        return _ok(
            id=task.id,
            title=task.title,
            body=task.body,
            assignee=task.assignee,
            status=task.status,
            workspace_kind=task.workspace_kind,
            workspace_path=task.workspace_path,
            # ... 更多字段
            comments=[...],             # 所有评论
            events=[...],               # 最近 50 条事件
            runs=[...],                 # 所有历史 run
            worker_context=kb.build_worker_context(conn, tid),  # ★ 核心
        )
```

**`build_worker_context` 组装的上下文包含：**

```python
# kanban_db.py: build_worker_context() (Line ~5444)
def build_worker_context(conn, task_id):
    """拼装 Worker 需要的所有上下文信息为一个可读字符串"""

    lines = []
    lines.append(f"# Kanban task {task.id}: {task.title}")
    lines.append(f"Assignee: {task.assignee}")
    lines.append(f"Status:   {task.status}")

    # ── 任务正文 ──
    if task.body:
        lines.append("## Body")
        lines.append(task.body)

    # ── 历史尝试（最近 N 次，含失败原因）──
    for run in shown_prior_runs:
        lines.append(f"### Attempt {idx} — {run.outcome} ({run.profile}, {ts})")
        if run.summary:
            lines.append(run.summary)       # 上次成功的结果
        if run.error:
            lines.append(f"_error_: {run.error}")  # 上次失败原因

    # ── ★ 父任务 handoff（关键：跨任务传递结果）──
    for parent_id in parent_ids:
        parent_task = get_task(conn, parent_id)
        parent_runs = [r for r in list_runs(conn, parent_id)
                       if r.outcome == "completed"]
        if parent_runs:
            latest = sorted(parent_runs, key=lambda r: r.started_at)[-1]
            lines.append(f"## Parent task {parent_id}: {parent_task.title}")
            lines.append(latest.summary)    # ★ 父任务的产出摘要
        elif parent_task.result:
            lines.append(f"## Parent task {parent_id}: {parent_task.title}")
            lines.append(parent_task.result)

    # ── 评论（人工介入的 durable 注释）──
    for comment in shown_comments:
        lines.append(f"comment from `{comment.author}` at {ts}:")
        lines.append(comment.body)

    return "\n".join(lines)
```

#### D2. `kanban_heartbeat` — 长任务存活信号

**调用链：** 模型 tool_call → `kanban_tools.py:kanban_heartbeat()` → `kanban_db.py:heartbeat_claim()` + `heartbeat_worker()`

```python
# tools/kanban_tools.py: kanban_heartbeat() (Line ~552)
def kanban_heartbeat(task_id=None, note=None):
    # 1) 所有权校验：只能心跳自己的任务
    err = _enforce_worker_task_ownership(tid)
    if err:
        return err

    with kb.connect() as conn:
        # ★ 双重心跳策略

        # 心跳 1: 续期 claim 锁（防止 dispatcher 因超时回收）
        claim_lock = os.environ.get("HERMES_KANBAN_CLAIM_LOCK")
        kb.heartbeat_claim(conn, tid, claimer=claim_lock)
        # 效果：UPDATE tasks SET claim_expires = now + ttl WHERE claim_lock = ?

        # 心跳 2: 记录 heartbeat 事件（审计 + 失活检测）
        kb.heartbeat_worker(conn, tid, note=note,
                           expected_run_id=_worker_run_id(tid))
        # 效果：UPDATE tasks SET last_heartbeat_at = now
        #        INSERT INTO task_events (kind='heartbeat', payload=note)

    return _ok(note=note)
```

**为什么要双重心跳？**

| 心跳 | 目的 | 不心跳的后果 |
|------|------|-------------|
| `heartbeat_claim` | 续期 claim TTL | dispatcher `release_stale_claims` 回收任务 |
| `heartbeat_worker` | 更新 `last_heartbeat_at` | dispatcher `detect_stale_running` 终止进程 |

#### D3. `kanban_complete` — 完成 + 触发下游

**调用链：** 模型 tool_call → `kanban_tools.py:kanban_complete()` → `kanban_db.py:complete_task()`

```python
# tools/kanban_tools.py: kanban_complete() (Line ~392)
def kanban_complete(task_id=None, result=None, summary=None,
                    metadata=None, created_cards=None, artifacts=None):
    # 1) 所有权校验
    err = _enforce_worker_task_ownership(tid)
    if err:
        return err

    # 2) 至少提供 summary 或 result
    if not summary and not result:
        return tool_error("At least one of summary or result is required")

    # 3) created_cards 校验（防止幻觉引用不存在的卡片）
    if created_cards:
        verified, phantom = _verify_created_cards(conn, tid, created_cards)
        if phantom:
            raise HallucinatedCardsError(phantom, tid)
            # → 返回错误提示让模型修正，而非静默接受

    # 4) artifacts 合并（追加到 metadata.artifacts，去重）
    if artifacts:
        existing = (task_metadata or {}).get("artifacts", [])
        merged = list(set(existing + normalized_artifacts))

    with kb.connect() as conn:
        # ★ 核心：标记任务完成
        ok = kb.complete_task(
            conn, tid,
            result=result,
            summary=summary,         # ★ 写入 task_runs.summary（下游 handoff 读这个）
            metadata=metadata,
            created_cards=created_cards,
            expected_run_id=_worker_run_id(tid),  # ★ 防止过期 run 误操作
        )
```

```python
# kanban_db.py: complete_task() (Line ~2668)
def complete_task(conn, task_id, *, result=None, summary=None,
                  metadata=None, created_cards=None, expected_run_id=None):
    with write_txn(conn):
        # ★ 原子更新：running → done + 清除锁
        if expected_run_id is not None:
            cur = conn.execute(
                """
                UPDATE tasks
                   SET status = 'done', result = ?,
                       completed_at = ?,
                       claim_lock = NULL, claim_expires = NULL,
                       worker_pid = NULL
                 WHERE id = ?
                   AND status IN ('running', 'ready', 'blocked')
                   AND current_run_id = ?     -- ★ 精确匹配 run，防并发
                """,
                (result, now, task_id, int(expected_run_id)),
            )
        else:
            # 无 run_id 校验的版本（CLI / Dashboard 调用）
            cur = conn.execute(
                """
                UPDATE tasks SET status = 'done', result = ?,
                    completed_at = ?,
                    claim_lock = NULL, worker_pid = NULL
                WHERE id = ? AND status IN ('running', 'ready', 'blocked')
                """,
                (result, now, task_id),
            )

        # 关闭当前 run
        run_id = _end_run(conn, task_id, outcome="completed",
                         status="completed", summary=summary, metadata=metadata)

        # ★ 写入审计事件
        _append_event(conn, task_id, "completed", {"summary": summary})

        # ★★★ 触发依赖提升：检查下游子任务是否可执行 ★★★
        recompute_ready(conn)
```

**`complete_task` 的级联效应：**

```text
complete_task(t_abc)
  → UPDATE tasks SET status='done'
  → INSERT task_events (kind='completed')
  → recompute_ready()
      → 找到 t_abc 的所有 child
      → 如果 child 的所有 parent 都是 done → child 从 todo 提升为 ready
      → 下一 tick dispatcher 会 claim + spawn child 的 Worker
```

#### D4. `kanban_block` — 阻塞 + 等待人工

**调用链：** 模型 tool_call → `kanban_tools.py:kanban_block()` → `kanban_db.py:block_task()`

```python
# tools/kanban_tools.py: kanban_block() (Line ~514)
def kanban_block(task_id=None, reason=None):
    # 1) 所有权校验
    err = _enforce_worker_task_ownership(tid)
    if err:
        return err

    # 2) reason 必填（告诉人类需要什么）
    if not reason:
        return tool_error("reason is required — explain what input is needed")

    with kb.connect() as conn:
        ok = kb.block_task(
            conn, tid,
            reason=reason,
            expected_run_id=_worker_run_id(tid),
        )
```

```python
# kanban_db.py: block_task() (Line ~2978)
def block_task(conn, task_id, *, reason=None, expected_run_id=None):
    with write_txn(conn):
        # ★ 原子更新：running/ready → blocked + 清锁
        cur = conn.execute(
            """
            UPDATE tasks
               SET status = 'blocked',
                   claim_lock = NULL, claim_expires = NULL, worker_pid = NULL
             WHERE id = ? AND status IN ('running', 'ready')
            """,
            (task_id,),
        )

        # 关闭当前 run
        run_id = _end_run(conn, task_id, outcome="blocked",
                         status="blocked", summary=reason)

        # 写入审计事件
        _append_event(conn, task_id, "blocked", {"reason": reason})
```

#### D5. `_enforce_worker_task_ownership` — Worker 作用域守卫

**源码：** `tools/kanban_tools.py` (Line ~132)

```python
def _enforce_worker_task_ownership(tid):
    """Worker 只能操作自己被分配的任务"""
    env_tid = os.environ.get("HERMES_KANBAN_TASK")
    if not env_tid:
        # Orchestrator 或 CLI 上下文 — 无任务级限制
        return None
    if tid != env_tid:
        # ★ Worker 试图操作别人的任务 → 拒绝
        return tool_error(
            f"worker is scoped to task {env_tid}; refusing to mutate {tid}. "
            f"Use kanban_comment to hand off information to other tasks, "
            f"or kanban_create to spawn follow-up work."
        )
    return None
```

**适用范围：**

| 工具 | 受 ownership 约束？ | 说明 |
|------|---------------------|------|
| `kanban_complete` | **是** | Worker 只能完成自己的任务 |
| `kanban_block` | **是** | Worker 只能阻塞自己的任务 |
| `kanban_heartbeat` | **是** | Worker 只能心跳自己的任务 |
| `kanban_unblock` | **是**（但还需 orchestrator 权限） | 双重门控 |
| `kanban_comment` | **否** | 允许跨任务评论（用于 handoff） |
| `kanban_create` | **否** | 允许创建任意子任务 |
| `kanban_list` | **否**（但需 orchestrator 权限） | 只读操作 |

---

### 阶段 E：通知与可视化

#### E1. Notifier Watcher — 轮询订阅 + 推送

**源码：** `gateway/run.py: _kanban_notifier_watcher()` (Line ~4495)

```python
async def _kanban_notifier_watcher(hub, config, interval=5.0):
    # 终态事件类型（只有这些才触发通知）
    TERMINAL_KINDS = {"completed", "blocked", "gave_up", "crashed", "timed_out"}

    while not shutdown_event.is_set():
        boards = await asyncio.to_thread(_kb.list_boards)

        for slug in boards:
            conn = await asyncio.to_thread(_kb.connect, board=slug)
            subs = await asyncio.to_thread(_kb.list_notify_subs, conn)

            for sub in subs:
                # ★ 原子领取：获取该订阅的未读事件 + 推进游标
                old_cursor, cursor, events = await asyncio.to_thread(
                    _kb.claim_unseen_events_for_sub,
                    conn,
                    task_id=sub["task_id"],
                    platform=sub["platform"],
                    chat_id=sub["chat_id"],
                    kinds=TERMINAL_KINDS,
                )

                for event in events:
                    # 按事件类型格式化消息
                    if event["kind"] == "completed":
                        msg = f"✔ Kanban {event['task_id']} done — {summary}"
                    elif event["kind"] == "blocked":
                        msg = f"⏸ Kanban {event['task_id']} blocked: {reason}"
                    elif event["kind"] == "crashed":
                        msg = f"✖ Kanban {event['task_id']} worker crashed"
                    # ... 发送到 Telegram/Discord/Slack

                    try:
                        await hub.send(sub["platform"], sub["chat_id"], msg)
                        # 成功 → 推进游标
                        await asyncio.to_thread(
                            _kb.advance_notify_cursor, conn, sub, cursor
                        )
                    except Exception:
                        # 失败 → 回退游标（下次重试）
                        await asyncio.to_thread(
                            _kb.rewind_notify_cursor, conn, sub, old_cursor
                        )

            # 任务已 done/archived → 自动取消订阅
            for sub in subs:
                task = await asyncio.to_thread(_kb.get_task, conn, sub["task_id"])
                if task["status"] in ("done", "archived"):
                    await asyncio.to_thread(_kb.remove_notify_sub, conn, sub)

        await asyncio.sleep(interval)  # 默认 5s 轮询
```

#### E2. Dashboard WebSocket — 实时事件流

```text
Dashboard 前端
    ↓ WebSocket connect
/plugins/kanban/dashboard/plugin_api.py
    ↓ tail task_events（WHERE id > last_seen_id）
    ↓ debounce 300ms
    ↓ refetch board 数据
    ↓ push 给前端
```

#### E3. CLI watch — 命令行实时流

```bash
hermes kanban watch     # 实时打印 task_events
hermes kanban tail      # 同上，别名
```

底层同样是轮询 `task_events` 表，`WHERE id > last_seen_id`，每 2s 检查一次。

---

### 完整命令 → 流程速查表

| 命令 | 入口 | 核心函数 | 关键 SQL/操作 |
|------|------|----------|--------------|
| `hermes kanban create` | `kanban.py:_cmd_create` | `kanban_db.create_task()` | `INSERT tasks` + `INSERT task_links` + `_append_event('created')` |
| `hermes kanban list` | `kanban.py:_cmd_list` | `kanban_db.recompute_ready()` + `list_tasks()` | 先提升依赖满足的任务，再 `SELECT ... FROM tasks WHERE status=...` |
| `hermes kanban show` | `kanban.py:_cmd_show` | `kanban_db.get_task()` + `build_worker_context()` | 多表 JOIN + 拼装上下文字符串 |
| `hermes kanban complete` | `kanban.py:_cmd_complete` | `kanban_db.complete_task()` | `UPDATE tasks SET status='done'` + `recompute_ready()` |
| `hermes kanban block` | `kanban.py:_cmd_block` | `kanban_db.block_task()` | `UPDATE tasks SET status='blocked'` + `add_comment('BLOCKED: ...')` |
| `hermes kanban unblock` | `kanban.py:_cmd_unblock` | `kanban_db.unblock_task()` | 检查 parent → `UPDATE tasks SET status='ready'/'todo'` |
| `/kanban create` | `gateway/run.py:_handle_kanban_command` | `kanban.py:run_slash()` → `_cmd_create` | 同 CLI + `add_notify_sub` 自动订阅 |
| `kanban_show`（工具） | `kanban_tools.py:kanban_show` | `kanban_db.build_worker_context()` | 组装 body + parent handoff + 历史 run + comments |
| `kanban_complete`（工具） | `kanban_tools.py:kanban_complete` | `kanban_db.complete_task()` | `_enforce_worker_task_ownership` + `recompute_ready()` |
| `kanban_block`（工具） | `kanban_tools.py:kanban_block` | `kanban_db.block_task()` | `_enforce_worker_task_ownership` + reason 必填 |
| `kanban_heartbeat`（工具） | `kanban_tools.py:kanban_heartbeat` | `heartbeat_claim()` + `heartbeat_worker()` | 双重心跳：续期 claim TTL + 更新 heartbeat_at |
| `kanban_create`（工具） | `kanban_tools.py:kanban_create` | `kanban_db.create_task()` | 同 CLI 的 create，额外注入 `HERMES_PROFILE` 作为 `created_by` |
| `kanban_comment`（工具） | `kanban_tools.py:kanban_comment` | `kanban_db.add_comment()` | `INSERT task_comments`，author 从 `HERMES_PROFILE` 取（防伪造） |
| dispatcher tick | `gateway/run.py:_kanban_dispatcher_watcher` | `kanban_db.dispatch_once()` | 6 步流水线：回收 → 超时 → 崩溃 → 提升 → claim → spawn |
| notifier tick | `gateway/run.py:_kanban_notifier_watcher` | `claim_unseen_events_for_sub()` | 游标推进/回退 + 平台推送 + 自动退订 |

---

## 6. 架构分层与源码地图

```text
┌─────────────────────────────────────────────────────────┐
│  使用面                                                  │
│  CLI (hermes_cli/kanban.py)                             │
│  /kanban (gateway/run.py)                               │
│  Dashboard (plugins/kanban/dashboard/plugin_api.py)       │
│  kanban_* tools (tools/kanban_tools.py)                 │
└────────────────────────┬────────────────────────────────┘
                         │ 全部委托
                         ▼
┌─────────────────────────────────────────────────────────┐
│  领域内核：hermes_cli/kanban_db.py                       │
│  Schema · CRUD · claim/complete/block                   │
│  dispatch_once · build_worker_context · boards          │
└────────────────────────┬────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ~/.hermes/      gateway/run.py    subprocess
   kanban*.db      dispatcher +       hermes -p chat
                   notifier watchers
```

### 6.1 关键源码文件

| 路径 | 职责 |
|------|------|
| `hermes_cli/kanban_db.py` | SQLite 内核、dispatcher、`dispatch_once`、`_default_spawn` |
| `hermes_cli/kanban.py` | CLI / `run_slash` |
| `tools/kanban_tools.py` | Agent 工具面 + gating |
| `toolsets.py` | `kanban` toolset 定义 |
| `model_tools.py` | `HERMES_KANBAN_TASK` 时自动启用 kanban toolset |
| `gateway/run.py` | 内嵌 dispatcher + notifier watchers |
| `agent/prompt_builder.py` | `KANBAN_GUIDANCE` |
| `hermes_cli/kanban_decompose.py` | triage 自动分解 |
| `plugins/kanban/dashboard/` | Dashboard 插件 |

### 6.2 claim / complete 核心语义

**`claim_task`**（`ready → running`）：

- CAS：`status='ready' AND claim_lock IS NULL`
- 校验 parent 均已 done，否则 demote 回 `todo`
- 新建 `task_runs` 行，写 `current_run_id`

**`complete_task`**：

- `summary` / `metadata` 写在 run 上，下游 child 经 `build_worker_context` 读 parent handoff
- 成功后 `recompute_ready` 提升子任务

---

## 7. 与 delegate_task 的关系

| 层级 | 机制 |
|------|------|
| 编排层 | Kanban：durable 队列、多 Profile、人机 loop |
| 执行层 | Worker 内仍可用 `delegate_task` 做短同步子推理 |
| 对比 | `delegate_task` ≈ 函数调用；Kanban ≈ 共享 DB 上的工作流 |

Orchestrator 模式：Profile 启用 `kanban` toolset + `kanban-orchestrator` skill，用 `kanban_create(parents=[...])` 搭 DAG，自身不写实现。

---

## 8. 关键配置项

`~/.hermes/config.yaml` → `kanban:` 段（见 `hermes_cli/config.py` DEFAULT_CONFIG）：

| 键 | 默认 | 含义 |
|----|------|------|
| `dispatch_in_gateway` | `true` | dispatcher 是否在 gateway 内 |
| `dispatch_interval_seconds` | `60` | tick 间隔 |
| `failure_limit` | `2` | 连续失败熔断 |
| `dispatch_stale_timeout_seconds` | `14400` | 无 heartbeat 回收（4h） |
| `auto_decompose` | `true` | triage 自动 LLM 分解 |
| `auto_decompose_per_tick` | `3` | 每 tick 最多分解几个 triage |
| `orchestrator_profile` | `""` | 分解用的 Profile |
| `max_spawn` / `max_in_progress` | — | 并发上限 |

辅助 LLM（`auxiliary:`）：

- `kanban_decomposer` — Decompose
- `triage_specifier` — Specify 单任务规格
- `profile_describer` — Profile 描述自动生成（routing 用）

Board 解析顺序：`--board` CLI > `HERMES_KANBAN_BOARD` env > `~/.hermes/kanban/current` > `default`。

---

## 9. 延伸阅读

| 文档 | 内容 |
|------|------|
| [Hermes-Kanban完整指南.md](./Hermes-Kanban完整指南.md) | 本仓库深度专题：七态、容错、协作模式 P1–P9、CLI 速查 |
| `hermes-agent/website/docs/user-guide/features/kanban.md` | 官方 reference |
| `hermes-agent/website/docs/user-guide/features/kanban-tutorial.md` | 四个 user story 教程 |
| `hermes-agent/docs/hermes-kanban-v1-spec.pdf` | 设计 spec（行为变更 PR 前必读） |
| [hermes-agent-main-loop.md](./hermes-agent-main-loop.md) | Worker 内部 Agent 主循环 |

---

*文档版本：基于 hermes-agent 源码与官方 Kanban 文档整理（2026-05）。*
