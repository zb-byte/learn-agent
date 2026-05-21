# Kanban Dispatcher 原理详解

> 核心源码:
> - `gateway/run.py:4994-5400` — 嵌入式 Dispatcher Watcher (调度循环)
> - `gateway/run.py:4495-4829` — Notifier Watcher (事件通知)
> - `hermes_cli/kanban_db.py:4702-5001` — `dispatch_once()` (单次调度核心)
> - `hermes_cli/kanban_db.py:5267-5431` — `_default_spawn()` (Worker 子进程生成)

---

## 一、整体架构

Gateway 进程内嵌了两个后台 asyncio Task，共享同一个事件循环但职责分离:

```
Gateway.start()
  │
  ├── asyncio.create_task(_kanban_notifier_watcher)    ← 通知推送 (5s 轮询)
  │     任务完成/阻塞/崩溃 → 推送消息到 Discord/Telegram/...
  │
  └── asyncio.create_task(_kanban_dispatcher_watcher)  ← 任务调度 (60s 轮询)
        检测 ready 任务 → 拉起 Worker 子进程 → 监控健康
```

**关键约束**: 所有 SQLite 操作通过 `asyncio.to_thread()` 投到线程池，绝不在事件循环中阻塞 (WAL 锁会卡住整个 Gateway)。

---

## 二、Worker 本质: 标准 HermesAgent chat 会话

Dispatcher 派发的 Worker **不是什么特殊进程**，就是一个标准的 `HermesAgent` chat 会话，通过 CLI 启动:

```bash
hermes -p <assignee> --accept-hooks --skills kanban-worker chat -q "work kanban task <task_id>"
```

这个命令和用户在终端里手动敲 `hermes chat` 没有任何区别，只不过:
- `-p <assignee>`: 以任务的指派人 profile 运行 (模型、toolset、配置各不相同)
- `--skills kanban-worker`: 注入 kanban 生命周期 skill (完成/阻塞/心跳的规范模板)
- `-q "work kanban task <id>"`: 初始 prompt 直接驱动 Agent 开始工作
- 环境变量注入了 `HERMES_KANBAN_TASK` 等，让 Agent 内部的 `kanban_tools` 知道自己处于 Worker 模式

```
Gateway 进程
  │
  ├── Dispatcher Watcher (asyncio Task，Gateway 内部)
  │     │
  │     └── dispatch_once() → _default_spawn()
  │           │
  │           └── subprocess.Popen()  ← fork 子进程
  │                 │
  │                 └── Worker 子进程 (完全独立)
  │                       │
  │                       └── hermes chat -q "work kanban task T001"
  │                             │
  │                             └── HermesAgent (标准 Agent 实例)
  │                                   │
  │                                   ├── 读取 HERMES_KANBAN_TASK 环境变量
  │                                   ├── 加载 kanban_tools (complete/block/heartbeat)
  │                                   ├── 执行任务 (写代码、跑测试、搜网页...)
  │                                   └── 调用 kanban_complete() 或 kanban_block()
  │                                         │
  │                                         └── 写入 SQLite DB
  │                                               → Notifier Watcher 检测到事件
  │                                                 → 推送给用户
```

---

## 三、Dispatcher Watcher — 调度循环

> 源码: `gateway/run.py:4994-5400`

### 3.1 启动与配置读取

```python
# gateway/run.py:5016-5045
async def _kanban_dispatcher_watcher(self) -> None:
    # 读取配置，决定是否启用内嵌 dispatcher
    # 环境变量覆盖: HERMES_KANBAN_DISPATCH_IN_GATEWAY=false 可强制关闭
    env_override = os.environ.get("HERMES_KANBAN_DISPATCH_IN_GATEWAY", "")
    if env_override in {"0", "false", "no", "off"}:
        return  # 关闭 → 用户需自行跑 `hermes kanban daemon`

    cfg = _load_config()
    kanban_cfg = cfg.get("kanban", {})
    if not kanban_cfg.get("dispatch_in_gateway", True):
        return  # 配置关闭

    interval = float(kanban_cfg.get("dispatch_interval_seconds", 60) or 60)
    max_spawn = kanban_cfg.get("max_spawn", None)           # 并发 Worker 上限
    max_in_progress = kanban_cfg.get("max_in_progress", None) # 运行中任务数上限
    failure_limit = kanban_cfg.get("failure_limit", DEFAULT_FAILURE_LIMIT)  # 自动 block 阈值
    stale_timeout_seconds = kanban_cfg.get("dispatch_stale_timeout_seconds", 0)  # 心跳超时

    await asyncio.sleep(5)  # 等 Gateway adapters 连接完成
```

### 3.2 主循环

```python
# gateway/run.py:5350-5400
while self._running:
    try:
        # Phase 1: Auto-decompose — 将 triage 任务拆解为可执行的任务图
        if auto_decompose_enabled:
            await asyncio.to_thread(_auto_decompose_tick)

        # Phase 2: 核心 — 对每个 board 执行一次 dispatch_once
        results = await asyncio.to_thread(_tick_once)

        # Phase 3: 日志 — 只在实际有动作时输出 (空闲网关保持安静)
        for slug, res in (results or []):
            if res and getattr(res, "spawned", None):
                logger.info(
                    "kanban dispatcher [%s]: spawned=%d reclaimed=%d crashed=%d ...",
                    slug, len(res.spawned), res.reclaimed, ...
                )

        # Phase 4: 健康遥测 — 连续 N 次 ready 队列非空但 0 spawn → 告警
        if ready_pending and not any_spawned:
            bad_ticks += 1
        if bad_ticks >= HEALTH_WINDOW:  # 默认 6 次 = 6 分钟
            logger.warning("kanban dispatcher stuck: ... Check profile health")

    except asyncio.CancelledError:
        raise  # Gateway 关闭时取消
    except Exception:
        logger.exception("unexpected watcher error")

    # 分片睡眠: 每 1s 检查一次 self._running，确保关闭时响应快
    slept = 0.0
    while slept < interval and self._running:
        await asyncio.sleep(min(1.0, interval - slept))
        slept += 1.0
```

### 3.3 Multi-Board 分发

```python
# gateway/run.py:5141-5217
def _tick_once_for_board(slug: str):
    """对单个 board 执行一次 dispatch"""
    conn = None
    # 损坏数据库检测: 同一指纹(board路径+mtime+size)跳过，变了才重试
    fingerprint = _board_db_fingerprint(slug)
    if disabled_corrupt_boards.get(slug) == fingerprint:
        return None  # 之前标记损坏且文件没变 → 跳过

    try:
        conn = _kb.connect(board=slug)  # 每个 board 独立连接，不共享
        return _kb.dispatch_once(
            conn,
            board=slug,
            max_spawn=max_spawn,
            max_in_progress=max_in_progress,
            failure_limit=failure_limit,
            stale_timeout_seconds=stale_timeout_seconds,
        )
    except sqlite3.DatabaseError as exc:
        if _is_corrupt_board_db_error(exc):
            disabled_corrupt_boards[slug] = fingerprint  # 标记损坏
        ...
    finally:
        conn.close()

def _tick_once():
    """遍历所有 board，每个执行一次 tick"""
    boards = _kb.list_boards(include_archived=False)
    return [(slug, _tick_once_for_board(slug)) for slug in boards]
```

---

## 四、dispatch_once — 单次调度核心

> 源码: `hermes_cli/kanban_db.py:4702-5001`

这是 Dispatcher 的心脏，每个 tick 执行一次。

### 4.1 执行步骤总览

```
dispatch_once(conn)
  │
  ├─ Step 1: 收割僵尸子进程 (os.waitpid WNOHANG)
  ├─ Step 2: 回收超时任务 (TTL 过期的 claim)
  ├─ Step 3: 检测心跳超时任务 (stale_timeout)
  ├─ Step 4: 检测崩溃 Worker (PID 不存在)
  ├─ Step 5: 强制超时 (max_runtime_seconds)
  ├─ Step 6: 提升 todo → ready (所有父任务完成)
  ├─ Step 7: 派发 ready 任务 (spawn Worker 子进程)
  └─ Step 8: 派发 review 任务 (spawn Review Agent)
```

### 4.2 Step 1-6: 清理与状态推进

```python
# kanban_db.py:4762-4790

# Step 1: 收割僵尸子进程 (Linux only)
# Worker 子进程退出后变成 <defunct>，必须 waitpid 回收
# 同时记录退出码，后续 detect_crashed_workers 用来区分:
#   - 正常退出但没调 kanban_complete → 协议违规 → auto-block
#   - 非0退出/OOM/SIGKILL → 真崩溃 → 计数
if os.name != "nt":
    while True:
        _pid, _status = os.waitpid(-1, os.WNOHANG)  # 非阻塞
        if _pid == 0: break
        _record_worker_exit(_pid, _status)

result = DispatchResult()

# Step 2: 回收超时 claim — claim 的 TTL 到了但 Worker 没完成
result.reclaimed = release_stale_claims(conn)

# Step 3: 心跳超时 — Worker 还活着但长时间没更新心跳
result.stale = detect_stale_running(conn, stale_timeout_seconds=stale_timeout_seconds)

# Step 4: 崩溃检测 — Worker 的 PID 已经不存在了
result.crashed = detect_crashed_workers(conn)

# Step 5: 运行时间超限 — 超过 max_runtime_seconds 强制回收
result.timed_out = enforce_max_runtime(conn)

# Step 6: 状态推进 — todo → ready (所有 parent 任务都完成了)
result.promoted = recompute_ready(conn)
```

### 4.3 Step 7: 派发 ready 任务

```python
# kanban_db.py:4807-4926

# 查询所有 ready + 未被 claim 的任务，按优先级+创建时间排序
ready_rows = conn.execute(
    "SELECT id, assignee FROM tasks "
    "WHERE status = 'ready' AND claim_lock IS NULL "
    "ORDER BY priority DESC, created_at ASC"
).fetchall()

# 并发上限检查: max_in_progress 控制 board 级别运行中任务数
if max_in_progress is not None:
    in_progress = conn.execute(
        "SELECT COUNT(*) FROM tasks WHERE status = 'running'"
    ).fetchone()[0]
    if in_progress >= max_in_progress:
        return result  # 达到上限，本 tick 不再 spawn
    # 只补差到上限
    remaining = max_in_progress - in_progress
    max_spawn = min(max_spawn or remaining, remaining)

for row in ready_rows:
    if running_count + spawned >= max_spawn:
        break  # 达到并发上限

    if not row["assignee"]:
        result.skipped_unassigned.append(row["id"])  # 无负责人 → 跳过
        continue

    # 非真实 profile (如 orion-cc 终端通道) → 跳过
    # 这些任务由终端通过 claim_task 主动拉取
    if not profile_exists(row["assignee"]):
        result.skipped_nonspawnable.append(row["id"])
        continue

    # 重启守卫: 刚失败的任务不立即重试，防止抖动
    guard_reason = check_respawn_guard(conn, row["id"])
    if guard_reason is not None:
        result.respawn_guarded.append((row["id"], guard_reason))
        continue

    # 原子 claim: 给任务加锁，防止并发 dispatcher 重复派发
    claimed = claim_task(conn, row["id"], ttl_seconds=ttl_seconds)
    if claimed is None:
        continue  # 被其他 dispatcher 抢先了

    # 解析工作区路径
    workspace = resolve_workspace(claimed, board=board)
    set_workspace_path(conn, claimed.id, str(workspace))

    # 核心: 调用 spawn 函数拉起 Worker 子进程
    try:
        pid = _spawn(claimed, str(workspace), board=board)
        if pid:
            _set_worker_pid(conn, claimed.id, int(pid))  # 记录 PID 用于崩溃检测
        result.spawned.append((claimed.id, claimed.assignee, str(workspace)))
        spawned += 1
    except Exception as exc:
        # spawn 失败 → 计数，超过 failure_limit → auto-block (熔断)
        auto = _record_spawn_failure(conn, claimed.id, str(exc), failure_limit=failure_limit)
        if auto:
            result.auto_blocked.append(claimed.id)
```

### 4.4 Step 8: 派发 review 任务

```python
# kanban_db.py:4928-5001
# 与 ready 派发逻辑完全对称，但:
# - 查询 status = 'review' 的任务
# - 强制注入 sdlc-review skill (代码审查专用)
# - 同样受 max_spawn 并发限制

review_rows = conn.execute(
    "SELECT id, assignee FROM tasks "
    "WHERE status = 'review' AND claim_lock IS NULL "
    "ORDER BY priority DESC, created_at ASC"
).fetchall()

for row in review_rows:
    # ... 同样的并发检查、profile检查、claim逻辑 ...
    claimed.skills = ["sdlc-review"]  # 审查 agent 专用 skill
    pid = _spawn(claimed, str(workspace), board=board)
    # ...
```

---

## 四、_default_spawn — Worker 子进程生成

> 源码: `hermes_cli/kanban_db.py:5267-5431`

### 4.1 环境变量注入

```python
# kanban_db.py:5294-5353
env = dict(os.environ)

# Worker 身份标记 — 整个代码库通过此变量判断 "我在 dispatcher 派发的 Worker 中"
env["HERMES_KANBAN_TASK"] = task.id

# 工作区隔离 — 每个 Worker 有独立的工作目录
env["HERMES_KANBAN_WORKSPACE"] = workspace

# Board 钉死 — Worker 只能看到自己 board 的 DB 和文件
env["HERMES_KANBAN_DB"] = str(kanban_db_path(board=board))
env["HERMES_KANBAN_WORKSPACES_ROOT"] = str(workspaces_root(board=board))
env["HERMES_KANBAN_BOARD"] = resolved_board

# Profile 隔离 — Worker 以其 assignee 的 profile 运行
env["HERMES_HOME"] = resolve_profile_env(profile_arg)
env["HERMES_PROFILE"] = profile_arg

# 运行身份 — 当前 attempt 的 run_id
env["HERMES_KANBAN_RUN_ID"] = str(task.current_run_id)
```

### 4.2 命令构建

```python
# kanban_db.py:5355-5396
cmd = [
    *_resolve_hermes_argv(),   # 找到 hermes 可执行文件
    "-p", profile_arg,          # 以 assignee 的 profile 运行
    "--accept-hooks",           # 接受 profile 配置的 hooks
]

# 自动加载 kanban-worker skill (生命周期指导、retry 诊断模板等)
if _kanban_worker_skill_available(env.get("HERMES_HOME")):
    cmd.extend(["--skills", "kanban-worker"])

# 任务级别的额外 skill
if task.skills:
    for sk in task.skills:
        if sk and sk != "kanban-worker":
            cmd.extend(["--skills", sk])

# 模型覆盖
if task.model_override:
    cmd.extend(["-m", task.model_override])

cmd.extend(["chat", "-q", f"work kanban task {task.id}"])
```

### 4.3 子进程启动

```python
# kanban_db.py:5397-5431
# 日志重定向到 <board-root>/logs/<task-id>.log (追加模式，unblock 重跑也保留)
log_path = worker_logs_dir(board=board) / f"{task.id}.log"
log_f = open(log_path, "ab")

proc = subprocess.Popen(
    cmd,
    cwd=workspace,                   # 工作目录 = 任务工作区
    stdin=subprocess.DEVNULL,         # 无标准输入 (非交互)
    stdout=log_f,                     # 输出重定向到日志文件
    stderr=subprocess.STDOUT,         # stderr 合并到 stdout
    env=env,                          # 注入上面的环境变量
    start_new_session=True,           # 脱离控制 tty (不被 Ctrl+C 杀死)
)
# 注意: log_f 不 close — 子进程继承 fd 继续写入
return proc.pid  # 返回 PID 给 dispatcher 做崩溃检测
```

---

## 五、Notifier Watcher — 事件通知

> 源码: `gateway/run.py:4495-4829`

### 5.1 职责

将任务的**终态事件**推送到用户所在的平台 (Discord/Telegram/...)，让人类无需主动查询。

### 5.2 订阅-推送模型

```
用户执行 `hermes kanban subscribe <task_id>`
  → 写入 kanban_notify_subs 表 (platform, chat_id, thread_id, cursor)

Notifier Watcher (每 5s):
  │
  ├─ 1. 遍历所有 board 的订阅
  ├─ 2. claim_unseen_events_for_sub(): 原子获取 cursor 之后的新事件
  ├─ 3. 按事件类型构造消息:
  │     completed → "✔ Kanban T001 done — <summary>"
  │     blocked   → "⏸ Kanban T001 blocked: <reason>"
  │     crashed   → "✖ Kanban T001 worker crashed; dispatcher will retry"
  │     timed_out → "⏱ Kanban T001 timed out (max_runtime=300s)"
  │     gave_up   → "✖ Kanban T001 gave up after repeated spawn failures"
  ├─ 4. adapter.send() 推送到 Discord/Telegram/...
  ├─ 5. 成功 → 推进 cursor; 失败 → 回退 cursor + 计数
  └─ 6. 连续 3 次发送失败 → 丢弃订阅 (bot 被踢/聊天已删)
```

### 5.3 核心逻辑

```python
# gateway/run.py:4651-4810 (简化)
deliveries = await asyncio.to_thread(_collect)  # 收集所有待推送事件

for d in deliveries:
    for ev in d["events"]:
        kind = ev.kind  # completed / blocked / crashed / timed_out / gave_up

        # 构造平台消息
        msg = format_kanban_event(kind, task, ev.payload)

        # 发送到平台
        try:
            await adapter.send(sub["chat_id"], msg, metadata=metadata)
            # completed 事件额外投递 artifacts (文件附件)
            if kind == "completed":
                await self._deliver_kanban_artifacts(adapter, ...)
            sub_fail_counts.pop(sub_key, None)  # 清零失败计数
        except Exception:
            fails += 1
            if fails >= 3:
                # 连续失败 → 丢弃订阅
                await asyncio.to_thread(self._kanban_unsub, sub)
            else:
                # 暂时失败 → 回退 cursor，下次重试
                await asyncio.to_thread(self._kanban_rewind, sub, ...)

    # 所有事件投递成功 → 推进 cursor (去重机制)
    await asyncio.to_thread(self._kanban_advance, sub, d["cursor"])

    # 只有任务真正终态 (done/archived) 才取消订阅
    # crashed/blocked/gave_up 不取消 — dispatcher 可能重试
    if task.status in {"done", "archived"}:
        await asyncio.to_thread(self._kanban_unsub, sub)
```

---

## 六、熔断与防护机制

### 6.1 Spawn 失败熔断

```python
# kanban_db.py:4517-4531
# 每个 task 有 consecutive_failures 计数器
# spawn 失败 → +1; 成功完成 → 清零
# 超过 failure_limit → auto-block (不再重试)

def _record_spawn_failure(conn, task_id, error, failure_limit):
    """记录 spawn 失败，超过阈值则自动 block"""
    failures = get_consecutive_failures(conn, task_id) + 1
    set_consecutive_failures(conn, task_id, failures)
    if failures >= failure_limit:
        block_task(conn, task_id, reason=error)  # 熔断!
        return True  # auto-blocked
    return False
```

### 6.2 重启守卫 (Respawn Guard)

```python
# kanban_db.py:4576+
# 防止刚失败的任务被立即重试:
# - quota/auth 错误 → 等待一段时间
# - 最近一次 spawn 仍在运行 → 不重复 spawn
# 守卫只延迟一 tick，不做永久拒绝；持续失败仍由熔断器兜底
guard_reason = check_respawn_guard(conn, row["id"])
if guard_reason is not None:
    result.respawn_guarded.append((row["id"], guard_reason))
    continue  # 本 tick 跳过
```

### 6.3 崩溃检测

```python
# kanban_db.py:4223+
# 两层检测:
# 1. PID 消失 (os.kill(pid, 0) 失败) → Worker 进程不存在了
# 2. 协议违规: Worker 正常退出但没调用 kanban_complete/kanban_block
#    → 记录为 auto-block (不需要人工干预)

def detect_crashed_workers(conn):
    running = conn.execute("SELECT id, worker_pid FROM tasks WHERE status='running'").fetchall()
    crashed = []
    for row in running:
        pid = row["worker_pid"]
        if pid and not _pid_alive(pid):
            crashed.append(row["id"])
            # 检查退出码: 0退出 = 协议违规 → auto-block
            exit_info = _get_recorded_exit(pid)
            if exit_info and exit_info.code == 0:
                auto_block(row["id"], reason="worker exited without completing")
    return crashed
```

---

## 七、DispatchResult 数据结构

```python
# kanban_db.py:3681-3710
@dataclass
class DispatchResult:
    """单次 dispatch tick 的结果"""

    reclaimed: int = 0                    # TTL 过期回收数
    promoted: int = 0                     # todo→ready 提升数
    spawned: list[tuple[str,str,str]]     # (task_id, assignee, workspace) 三元组
    skipped_unassigned: list[str]         # 无负责人的任务
    skipped_nonspawnable: list[str]       # 非 Hermes profile 的任务 (终端通道)
    crashed: list[str]                    # PID 消失的崩溃任务
    auto_blocked: list[str]              # 熔断自动 block 的任务
    timed_out: list[str]                 # 超过 max_runtime 的任务
    stale: list[str]                     # 心跳超时的任务
    respawn_guarded: list[tuple[str,str]] # (task_id, guard_reason) 被重启守卫跳过的
```

---

## 八、完整调用链

```
Gateway.start()
  │
  └── asyncio.create_task(_kanban_dispatcher_watcher)
        │
        ├── asyncio.sleep(5)  等 adapters 连接
        │
        └── while self._running:  ← 主循环 (默认 60s/tick)
              │
              ├── asyncio.to_thread(_auto_decompose_tick)
              │     └── kanban_decompose.decompose_task()
              │           triage 任务 → LLM 拆解 → 生成子任务图
              │
              ├── asyncio.to_thread(_tick_once)
              │     │
              │     ├── 遍历所有 board
              │     │
              │     └── 对每个 board: _tick_once_for_board(slug)
              │           │
              │           └── kanban_db.dispatch_once(conn, board=slug)
              │                 │
              │                 ├── Step 1: os.waitpid() 收割僵尸
              │                 ├── Step 2: release_stale_claims() 回收超时
              │                 ├── Step 3: detect_stale_running() 心跳超时
              │                 ├── Step 4: detect_crashed_workers() PID 崩溃
              │                 ├── Step 5: enforce_max_runtime() 运行超时
              │                 ├── Step 6: recompute_ready() todo→ready
              │                 │
              │                 ├── Step 7: 派发 ready 任务
              │                 │     for task in ready_rows:
              │                 │       ├── profile_exists() 过滤非 profile
              │                 │       ├── check_respawn_guard() 重启守卫
              │                 │       ├── claim_task() 原子加锁
              │                 │       ├── resolve_workspace() 解析工作区
              │                 │       └── _default_spawn() → subprocess.Popen
              │                 │             hermes -p <assignee> chat -q "work kanban task <id>"
              │                 │             │
              │                 │             └── Worker 子进程 (独立进程)
              │                 │                   env: HERMES_KANBAN_TASK=<id>
              │                 │                   读取 kanban_tools → 执行任务
              │                 │                   → kanban_complete() / kanban_block()
              │                 │
              │                 └── Step 8: 派发 review 任务 (同上 + sdlc-review skill)
              │
              ├── 健康遥测: bad_ticks >= 6 → 告警
              │
              └── 分片睡眠: 1s * 60 次 (可快速响应 shutdown)


Gateway.start()
  │
  └── asyncio.create_task(_kanban_notifier_watcher)  ← 并行运行
        │
        └── while self._running:  ← 5s/tick
              │
              └── asyncio.to_thread(_collect)
                    │ 遍历所有 board 的 kanban_notify_subs
                    │ claim_unseen_events_for_sub(): 原子获取新事件
                    │
                    └── for delivery in deliveries:
                          ├── adapter.send() 推送到 Discord/Telegram
                          ├── completed → 额外投递 artifacts
                          ├── 成功 → advance cursor
                          ├── 失败 → rewind cursor + 计数
                          └── 3次失败 → 丢弃订阅
```

---

## 九、并发模型总结

| 维度 | 说明 |
|------|------|
| **Dispatcher 并发** | 单 Gateway 内单线程 asyncio，SQLite 操作投线程池 |
| **Board 隔离** | 每个 board 独立 DB 连接，独立 tick |
| **Worker 并发上限** | `max_spawn` 控制同时运行 Worker 数 (board 级) |
| **任务并发上限** | `max_in_progress` 控制 running 状态任务数 |
| **Claim 原子性** | `claim_task()` SQL 原子操作，防止多 Dispatcher 重复派发 |
| **子进程隔离** | `start_new_session=True` + 独立 env，Worker 崩溃不影响 Gateway |
| **通知去重** | cursor 机制 + `claim_unseen_events_for_sub()` 原子操作 |
| **熔断器** | `failure_limit` 连续失败 → auto-block，防止无限重试 |
