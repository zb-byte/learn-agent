# KANBAN 子进程并发限制分析

## 问题

KANBAN Dispatcher 能启动多少个子进程（Worker）？

## 答案

**默认情况下：无限制（理论上可以启动无限多个 Worker）**

**可配置限制：通过 `config.yaml` 中的 `kanban.max_spawn` 和 `kanban.max_in_progress` 参数控制**

## 详细分析

### 1. 并发控制参数

KANBAN 系统提供了两个并发控制参数：

#### 1.1 `max_spawn` - 全局并发上限

**位置**: `hermes_cli/kanban_db.py::dispatch_once()`

```python
def dispatch_once(
    conn: sqlite3.Connection,
    *,
    max_spawn: Optional[int] = None,  # 默认 None = 无限制
    # ...
):
    """运行一次 dispatcher tick
    
    ``max_spawn`` is a **live concurrency cap**, not a per-tick spawn budget:
    it counts tasks already in ``status='running'`` plus this tick's spawns
    against the limit. So ``max_spawn=4`` means "at most 4 workers running
    at any time across the whole board"
    """
```

**含义**:
- `None` (默认): **无限制**，可以启动任意多个 Worker
- `max_spawn=4`: 整个 Board 上**最多同时运行 4 个 Worker**
- 这是一个**实时并发上限**，不是每次 tick 的启动预算

**计算方式**:
```python
# 统计当前正在运行的任务数
running_count = 0
if max_spawn is not None:
    running_count = int(
        conn.execute(
            "SELECT COUNT(*) FROM tasks WHERE status = 'running'"
        ).fetchone()[0]
    )

# 遍历 ready 任务，尝试启动
spawned = 0
for row in ready_rows:
    # 检查是否达到并发上限
    if max_spawn is not None and running_count + spawned >= max_spawn:
        break  # 停止启动新 Worker
    
    # 启动 Worker
    # ...
    spawned += 1
```

#### 1.2 `max_in_progress` - 进行中任务上限

**位置**: `hermes_cli/kanban_db.py::dispatch_once()`

```python
# Honour kanban.max_in_progress: if the board already has enough running
# tasks, skip spawning this tick so slow workers (local LLMs,
# resource-constrained hosts) can finish what they have before more tasks
# pile up and time out.
if max_in_progress is not None and ready_rows:
    in_progress = conn.execute(
        "SELECT COUNT(*) FROM tasks WHERE status = 'running'"
    ).fetchone()[0]
    if in_progress >= max_in_progress:
        return result  # 跳过本次 tick，不启动任何新 Worker
    
    # 只启动足够的 Worker 达到上限
    remaining = max_in_progress - in_progress
    if max_spawn is None or max_spawn > remaining:
        max_spawn = remaining
```

**含义**:
- `None` (默认): **无限制**
- `max_in_progress=10`: 当已有 10 个 Worker 运行时，**暂停启动新 Worker**
- 用于防止慢速 Worker（本地 LLM、资源受限主机）堆积和超时

**与 `max_spawn` 的区别**:
- `max_spawn`: 硬性并发上限，永远不超过
- `max_in_progress`: 软性限制，达到后暂停启动，等待现有 Worker 完成

### 2. 配置方式

**位置**: `gateway/run.py::_kanban_dispatcher_watcher()`

```python
# 从 config.yaml 读取配置
kanban_cfg = cfg.get("kanban", {})

# 读取 max_spawn 配置
max_spawn = kanban_cfg.get("max_spawn", None)  # 默认 None
if max_spawn is not None:
    logger.info(f"kanban dispatcher: max_spawn={max_spawn}")

# 读取 max_in_progress 配置
raw_max_in_progress = kanban_cfg.get("max_in_progress", None)  # 默认 None
max_in_progress = None
if raw_max_in_progress is not None:
    try:
        max_in_progress = int(raw_max_in_progress)
        if max_in_progress < 1:
            logger.warning("kanban.max_in_progress is below 1; ignoring")
            max_in_progress = None
        else:
            logger.info(f"kanban dispatcher: max_in_progress={max_in_progress}")
    except (TypeError, ValueError):
        logger.warning("invalid kanban.max_in_progress; ignoring")
        max_in_progress = None
```

**配置文件示例** (`~/.hermes/config.yaml`):

```yaml
kanban:
  # 全局并发上限：最多同时运行 4 个 Worker
  max_spawn: 4
  
  # 进行中任务上限：达到 10 个后暂停启动
  max_in_progress: 10
  
  # Dispatcher 轮询间隔（秒）
  dispatch_interval_seconds: 60
  
  # 任务失败重试次数
  failure_limit: 2
```

### 3. 默认值总结

| 参数 | 默认值 | 含义 | 影响 |
|------|--------|------|------|
| `max_spawn` | `None` | **无限制** | 可以启动任意多个 Worker |
| `max_in_progress` | `None` | **无限制** | 不会暂停启动 |
| `dispatch_interval_seconds` | `60` | 60 秒 | 每 60 秒检查一次 ready 任务 |
| `failure_limit` | `2` | 2 次 | 连续失败 2 次后自动阻塞任务 |

**结论**: 
- **默认情况下，KANBAN 可以启动无限多个 Worker 子进程**
- 唯一的限制是系统资源（CPU、内存、文件描述符等）

### 4. 实际并发场景

#### 场景 1: 无限制（默认）

```yaml
kanban:
  # 不配置 max_spawn 和 max_in_progress
```

**行为**:
- 有 100 个 ready 任务 → 启动 100 个 Worker
- 有 1000 个 ready 任务 → 启动 1000 个 Worker
- **限制**: 仅受系统资源限制

**风险**:
- ⚠️ 可能耗尽系统资源（内存、CPU、文件描述符）
- ⚠️ 大量并发 API 调用可能触发速率限制
- ⚠️ 可能导致系统不稳定

#### 场景 2: 设置 `max_spawn=4`

```yaml
kanban:
  max_spawn: 4
```

**行为**:
```
Tick 1:
  - 当前运行: 0 个
  - Ready 任务: 10 个
  - 启动: 4 个 (达到上限)
  - 结果: 4 个 Worker 运行，6 个任务仍在 ready

Tick 2 (60 秒后):
  - 当前运行: 4 个 (假设都还在运行)
  - Ready 任务: 6 个
  - 启动: 0 个 (已达上限)
  - 结果: 仍然 4 个 Worker 运行

Tick 3 (120 秒后):
  - 当前运行: 2 个 (假设 2 个已完成)
  - Ready 任务: 6 个
  - 启动: 2 个 (补充到上限)
  - 结果: 4 个 Worker 运行
```

**优点**:
- ✅ 控制资源使用
- ✅ 避免 API 速率限制
- ✅ 系统稳定

#### 场景 3: 设置 `max_in_progress=10`

```yaml
kanban:
  max_in_progress: 10
```

**行为**:
```
Tick 1:
  - 当前运行: 0 个
  - Ready 任务: 20 个
  - 启动: 10 个 (达到 max_in_progress)
  - 结果: 10 个 Worker 运行

Tick 2 (60 秒后):
  - 当前运行: 10 个 (假设都还在运行)
  - Ready 任务: 10 个
  - 启动: 0 个 (达到 max_in_progress，跳过本次 tick)
  - 结果: 仍然 10 个 Worker 运行

Tick 3 (120 秒后):
  - 当前运行: 5 个 (假设 5 个已完成)
  - Ready 任务: 10 个
  - 启动: 5 个 (补充到 max_in_progress)
  - 结果: 10 个 Worker 运行
```

**用途**:
- 适合慢速 Worker（本地 LLM）
- 防止任务堆积和超时

#### 场景 4: 同时设置两个参数

```yaml
kanban:
  max_spawn: 4
  max_in_progress: 10
```

**行为**:
- `max_spawn` 优先级更高（硬性限制）
- 实际上限 = `min(max_spawn, max_in_progress)` = `4`
- 等价于只设置 `max_spawn=4`

### 5. 并发限制的实现细节

#### 5.1 计数逻辑

```python
# 统计当前运行的任务数
running_count = int(
    conn.execute(
        "SELECT COUNT(*) FROM tasks WHERE status = 'running'"
    ).fetchone()[0]
)

# 遍历 ready 任务
spawned = 0
for row in ready_rows:
    # 检查并发上限
    if max_spawn is not None and running_count + spawned >= max_spawn:
        break  # 停止启动
    
    # 尝试启动 Worker
    claimed = claim_task(conn, row["id"])
    if claimed is None:
        continue  # 声明失败，跳过
    
    # 启动成功
    spawned += 1
```

**关键点**:
- `running_count`: 当前已运行的 Worker 数量
- `spawned`: 本次 tick 已启动的 Worker 数量
- 检查条件: `running_count + spawned >= max_spawn`

#### 5.2 任务状态转换

```
ready → running (启动 Worker)
  ↓
running → done (Worker 调用 kanban_complete)
running → blocked (Worker 调用 kanban_block)
running → ready (TTL 超时，重新声明)
```

**并发计数只统计 `status='running'` 的任务**

#### 5.3 Review 任务的特殊处理

**位置**: `hermes_cli/kanban_db.py::dispatch_once()`

```python
# Review tasks (status='review') are also spawned, but counted
# against max_spawn alongside ready tasks, so the total number of
# running workers (ready + review) respects the cap.
review_rows = conn.execute(
    "SELECT id, assignee FROM tasks "
    "WHERE status = 'review' AND claim_lock IS NULL "
    "ORDER BY priority DESC, created_at ASC"
).fetchall()

for row in review_rows:
    if max_spawn is not None and running_count + spawned >= max_spawn:
        break  # 同样受并发限制
    # ...
```

**说明**:
- `review` 状态的任务也会被启动
- 同样计入 `max_spawn` 限制

### 6. 系统资源限制

即使不设置 `max_spawn`，系统仍有物理限制：

#### 6.1 操作系统限制

| 资源 | Linux 默认限制 | macOS 默认限制 | Windows 限制 |
|------|---------------|---------------|-------------|
| 最大进程数 | `ulimit -u` (~30000) | `ulimit -u` (~2666) | 受内存限制 |
| 最大文件描述符 | `ulimit -n` (~1024) | `ulimit -n` (~256) | 受内存限制 |
| 最大内存 | 物理内存 + Swap | 物理内存 + Swap | 物理内存 + Pagefile |

#### 6.2 Python 限制

- **线程数**: Python 没有硬性限制，但受 OS 限制
- **子进程数**: 受 OS 进程数限制
- **内存**: 每个 Worker 进程约 100-500 MB（取决于模型和工具）

#### 6.3 实际并发能力估算

假设：
- 每个 Worker 占用 200 MB 内存
- 系统有 16 GB 内存
- 预留 4 GB 给系统和 Dispatcher

**理论最大并发**:
```
(16 GB - 4 GB) / 200 MB = 12 GB / 200 MB = 60 个 Worker
```

**推荐配置**:
```yaml
kanban:
  max_spawn: 30  # 保守估计，留有余量
```

### 7. 性能优化建议

#### 7.1 根据硬件配置

| 硬件配置 | 推荐 `max_spawn` | 说明 |
|---------|-----------------|------|
| 2 核 4 GB | 2-4 | 轻量级任务 |
| 4 核 8 GB | 4-8 | 中等任务 |
| 8 核 16 GB | 8-16 | 重度任务 |
| 16 核 32 GB | 16-32 | 高并发场景 |

#### 7.2 根据任务类型

| 任务类型 | 推荐 `max_spawn` | 原因 |
|---------|-----------------|------|
| API 密集型 | 4-8 | 避免速率限制 |
| CPU 密集型 | = CPU 核心数 | 充分利用 CPU |
| 内存密集型 | 根据内存计算 | 避免 OOM |
| I/O 密集型 | 2-4 × CPU 核心数 | I/O 等待时可以并发 |

#### 7.3 根据 API 速率限制

假设 API 速率限制为 60 RPM (Requests Per Minute):

```yaml
kanban:
  max_spawn: 4  # 每个 Worker 约 15 RPM，总共 60 RPM
  dispatch_interval_seconds: 60
```

### 8. 监控和调试

#### 8.1 查看当前运行的 Worker 数量

```bash
# 通过数据库查询
sqlite3 ~/.hermes/kanban.db "SELECT COUNT(*) FROM tasks WHERE status='running'"

# 通过 CLI
hermes kanban list --status running
```

#### 8.2 查看 Dispatcher 日志

```bash
# Gateway 日志中会显示
kanban dispatcher: max_spawn=4
kanban dispatcher: max_in_progress=10
```

#### 8.3 查看系统资源使用

```bash
# 查看进程数
ps aux | grep "hermes.*chat" | wc -l

# 查看内存使用
ps aux | grep "hermes.*chat" | awk '{sum+=$6} END {print sum/1024 " MB"}'
```

### 9. 常见问题

#### Q1: 为什么设置了 `max_spawn=4` 但看到 5 个 Worker？

**A**: 可能的原因：
1. 有 1 个 Worker 刚完成但还没退出
2. 有 1 个 Worker 是手动启动的（不受 Dispatcher 控制）
3. 统计时机问题（Worker 正在退出）

#### Q2: 如何动态调整并发限制？

**A**: 修改 `config.yaml` 后，Dispatcher 会在下一次 tick 时读取新配置：

```bash
# 编辑配置
vim ~/.hermes/config.yaml

# 等待下一次 tick（默认 60 秒）
# 或重启 Gateway
```

#### Q3: 并发限制会影响手动启动的 Worker 吗？

**A**: **不会**。`max_spawn` 只影响 Dispatcher 自动启动的 Worker。手动运行 `hermes -p profile chat -q "work kanban task xxx"` 不受限制。

#### Q4: 如何处理大量 ready 任务？

**A**: 三种策略：

1. **增加并发**: 提高 `max_spawn`
2. **优先级排序**: 设置任务 `priority` 字段
3. **分批处理**: 使用 `max_in_progress` 控制节奏

### 10. 代码路径总结

```
配置读取:
gateway/run.py::_kanban_dispatcher_watcher()
  ↓ 读取 config.yaml
  ↓ kanban.max_spawn (默认 None)
  ↓ kanban.max_in_progress (默认 None)
  ↓
hermes_cli/kanban_db.py::dispatch_once()
  ↓ 统计 running_count
  ↓ 遍历 ready_rows
  ↓ 检查 running_count + spawned >= max_spawn
  ↓ 如果未达上限，启动 Worker
  ↓
hermes_cli/kanban_db.py::_default_spawn()
  ↓ subprocess.Popen(["hermes", "-p", profile, "chat", "-q", ...])
  ↓ 返回 PID
```

## 总结

### 默认行为
- **无限制**: 可以启动任意多个 Worker 子进程
- **仅受系统资源限制**: CPU、内存、文件描述符等

### 可配置限制
- **`max_spawn`**: 全局并发上限（硬性限制）
- **`max_in_progress`**: 进行中任务上限（软性限制）

### 推荐配置
```yaml
kanban:
  max_spawn: 8  # 根据硬件和任务类型调整
  max_in_progress: 16  # 可选，用于慢速 Worker
  dispatch_interval_seconds: 60
  failure_limit: 2
```

### 关键设计
- **实时并发上限**: `max_spawn` 统计所有 `status='running'` 的任务
- **每次 tick 检查**: 每 60 秒（默认）检查一次并启动新 Worker
- **进程隔离**: 每个 Worker 是独立的子进程，互不干扰
