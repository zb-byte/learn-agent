# KANBAN 环境变量作用域分析

## 问题

`env["HERMES_KANBAN_TASK"] = task.id` 这个环境变量的影响范围是多大？是整个进程吗？

## 答案

**是的，影响范围是整个 Worker 子进程及其所有子进程**，但**不影响父进程（Dispatcher）**。

## 详细分析

### 1. 环境变量的设置位置

**代码位置**: `hermes_cli/kanban_db.py::_default_spawn()`

```python
def _default_spawn(task: Task, workspace: str, *, board: Optional[str] = None) -> Optional[int]:
    """启动 hermes -p <profile> chat -q ... 子进程"""
    
    # 第 1 步: 复制父进程的环境变量
    env = dict(os.environ)  # 创建一个新的字典副本
    
    # 第 2 步: 添加 KANBAN 特定的环境变量
    env["HERMES_KANBAN_TASK"] = task.id
    env["HERMES_KANBAN_WORKSPACE"] = workspace
    env["HERMES_KANBAN_RUN_ID"] = str(task.current_run_id)
    env["HERMES_KANBAN_CLAIM_LOCK"] = task.claim_lock
    env["HERMES_PROFILE"] = profile_arg
    # ... 其他环境变量
    
    # 第 3 步: 使用这个环境变量字典启动子进程
    proc = subprocess.Popen(
        cmd,
        env=env,  # ← 关键：传递修改后的环境变量字典
        start_new_session=True,  # ← 创建新的会话
        # ...
    )
    
    return proc.pid
```

### 2. 作用域范围

```
┌─────────────────────────────────────────────────────────────┐
│ Dispatcher 进程 (父进程)                                     │
│ os.environ["HERMES_KANBAN_TASK"] = 不存在                    │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ _default_spawn() 函数                                   │ │
│  │ env = dict(os.environ)  # 复制环境变量                  │ │
│  │ env["HERMES_KANBAN_TASK"] = "task-123"  # 只在 env 字典中│ │
│  └────────────────────────────────────────────────────────┘ │
│                           │                                  │
│                           │ subprocess.Popen(env=env)        │
│                           ↓                                  │
└───────────────────────────┼──────────────────────────────────┘
                            │
                            │ 创建新进程
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Worker 进程 (子进程)                                         │
│ os.environ["HERMES_KANBAN_TASK"] = "task-123"  ✓            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ AIAgent 初始化                                          │ │
│  │ - 读取 os.environ.get("HERMES_KANBAN_TASK")            │ │
│  │ - 注册 kanban_* 工具                                    │ │
│  │ - 添加 KANBAN_GUIDANCE 到系统提示词                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 工具调用 (tools/kanban_tools.py)                        │ │
│  │ - kanban_show(): 读取 os.environ.get("HERMES_KANBAN_TASK")│
│  │ - kanban_complete(): 验证任务所有权                     │ │
│  │ - kanban_block(): 验证任务所有权                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                           │                                  │
│                           │ 如果 Worker 启动子进程           │
│                           ↓                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Worker 的子进程（例如：terminal 工具启动的 shell）      │ │
│  │ os.environ["HERMES_KANBAN_TASK"] = "task-123"  ✓        │ │
│  │ 继承了父进程（Worker）的所有环境变量                    │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 3. 关键技术点

#### 3.1 环境变量隔离

```python
# 在 _default_spawn() 中
env = dict(os.environ)  # 创建副本，不修改父进程的环境变量
env["HERMES_KANBAN_TASK"] = task.id  # 只在副本中添加

# Dispatcher 进程的环境变量不受影响
# os.environ["HERMES_KANBAN_TASK"] 仍然不存在（或保持原值）
```

**结果**:
- ✅ Worker 子进程可以读取 `HERMES_KANBAN_TASK`
- ✅ Worker 的所有子进程都可以读取（继承）
- ❌ Dispatcher 父进程**不会**被修改

#### 3.2 进程隔离

```python
proc = subprocess.Popen(
    cmd,
    env=env,  # 传递独立的环境变量字典
    start_new_session=True,  # 创建新的会话，进一步隔离
)
```

**`start_new_session=True` 的作用**:
- 创建新的进程组和会话
- Worker 进程不会收到 Dispatcher 的信号（如 Ctrl+C）
- 即使 Dispatcher 退出，Worker 也可以继续运行

### 4. 环境变量的读取位置

#### 4.1 工具门控检查

**位置**: `tools/kanban_tools.py`

```python
def _check_kanban_mode() -> bool:
    """检查是否为 KANBAN Worker"""
    if os.environ.get("HERMES_KANBAN_TASK"):
        return True  # 是 Worker，注册工具
    return _profile_has_kanban_toolset()

def _check_kanban_orchestrator_mode() -> bool:
    """检查是否为编排器（不是 Worker）"""
    if os.environ.get("HERMES_KANBAN_TASK"):
        return False  # 是 Worker，不注册编排器工具
    return _profile_has_kanban_toolset()
```

**结果**:
- Worker 进程: `kanban_show`, `kanban_complete`, `kanban_block` 等工具可用
- Dispatcher 进程: 这些工具**不可用**（因为环境变量不存在）

#### 4.2 任务所有权验证

**位置**: `tools/kanban_tools.py::_enforce_worker_task_ownership()`

```python
def _enforce_worker_task_ownership(tid: str) -> Optional[str]:
    """防止 Worker 操作其他任务"""
    env_tid = os.environ.get("HERMES_KANBAN_TASK")
    if not env_tid:
        # 不是 Worker（可能是编排器或 CLI），允许操作
        return None
    if tid != env_tid:
        # Worker 试图操作其他任务，拒绝
        return tool_error(
            f"worker is scoped to task {env_tid}; refusing to mutate {tid}"
        )
    return None
```

**安全机制**:
- Worker 只能操作自己的任务（`HERMES_KANBAN_TASK` 指定的任务）
- 防止 Worker 意外或恶意修改其他任务
- 编排器（没有环境变量）可以操作任何任务

#### 4.3 默认任务 ID 解析

**位置**: `tools/kanban_tools.py::_default_task_id()`

```python
def _default_task_id(arg: Optional[str]) -> Optional[str]:
    """解析任务 ID，优先使用参数，否则使用环境变量"""
    if arg:
        return arg
    env_tid = os.environ.get("HERMES_KANBAN_TASK")
    return env_tid or None
```

**便利性**:
- Worker 调用 `kanban_show()` 时不需要传递 `task_id` 参数
- 自动使用环境变量中的任务 ID

### 5. 多 Worker 并发场景

```
Dispatcher 进程
├─ Worker 1 (task-123)
│  └─ env["HERMES_KANBAN_TASK"] = "task-123"
│     └─ 只能操作 task-123
│
├─ Worker 2 (task-456)
│  └─ env["HERMES_KANBAN_TASK"] = "task-456"
│     └─ 只能操作 task-456
│
└─ Worker 3 (task-789)
   └─ env["HERMES_KANBAN_TASK"] = "task-789"
      └─ 只能操作 task-789
```

**隔离保证**:
- 每个 Worker 有独立的环境变量
- Worker 之间不会互相干扰
- 每个 Worker 只能看到和操作自己的任务

### 6. 环境变量的生命周期

```
时间线:
─────────────────────────────────────────────────────────────

T0: Dispatcher 调用 _default_spawn()
    └─ env = dict(os.environ)
    └─ env["HERMES_KANBAN_TASK"] = "task-123"

T1: subprocess.Popen(env=env) 创建 Worker 进程
    └─ Worker 进程继承 env 字典中的所有环境变量

T2: Worker 进程运行
    └─ os.environ.get("HERMES_KANBAN_TASK") 返回 "task-123"
    └─ 工具可以读取和验证

T3: Worker 调用 kanban_complete() 完成任务

T4: Worker 进程退出
    └─ 环境变量随进程销毁

T5: Dispatcher 仍在运行
    └─ os.environ.get("HERMES_KANBAN_TASK") 仍然不存在
```

### 7. 实际影响范围总结

| 进程/组件 | 能否读取 `HERMES_KANBAN_TASK` | 说明 |
|----------|-------------------------------|------|
| Dispatcher 父进程 | ❌ 否 | 环境变量只在子进程的 env 字典中 |
| Worker 子进程 | ✅ 是 | 通过 `subprocess.Popen(env=env)` 传递 |
| Worker 的子进程 | ✅ 是 | 继承父进程（Worker）的环境变量 |
| 其他 Worker | ❌ 否 | 每个 Worker 有独立的环境变量 |
| CLI 直接调用 | ❌ 否 | 用户手动运行 `hermes chat` 时没有这个变量 |

### 8. 为什么这样设计？

#### 8.1 安全性
- **任务隔离**: 每个 Worker 只能操作自己的任务
- **防止误操作**: Worker 不能意外修改其他任务
- **审计追踪**: 环境变量记录了 Worker 的身份

#### 8.2 便利性
- **自动上下文**: Worker 不需要手动传递任务 ID
- **工具简化**: `kanban_show()` 可以无参数调用
- **一致性**: 所有工具都能访问相同的任务上下文

#### 8.3 可扩展性
- **多后端支持**: 环境变量在 local/docker/modal/ssh 后端都有效
- **子进程继承**: Worker 启动的任何子进程都能访问任务 ID
- **跨语言**: 任何语言的子进程都能读取环境变量

### 9. 代码验证示例

假设你想验证环境变量的作用域，可以这样测试：

```python
# 在 Dispatcher 进程中
print(f"Dispatcher: {os.environ.get('HERMES_KANBAN_TASK')}")  # 输出: None

# 启动 Worker
_default_spawn(task, workspace)

# Dispatcher 进程中再次检查
print(f"Dispatcher after spawn: {os.environ.get('HERMES_KANBAN_TASK')}")  # 输出: None

# 在 Worker 进程中（通过日志查看）
# 输出: Worker: task-123
```

### 10. 潜在问题和注意事项

#### 10.1 环境变量泄漏
- ✅ **不会泄漏**: 因为使用 `dict(os.environ)` 创建副本
- ✅ **进程隔离**: 每个 Worker 有独立的环境变量

#### 10.2 子进程继承
- ⚠️ **注意**: Worker 启动的所有子进程都会继承 `HERMES_KANBAN_TASK`
- 💡 **用途**: 可以在 shell 脚本中读取 `$HERMES_KANBAN_TASK`

#### 10.3 环境变量大小限制
- ⚠️ **限制**: 操作系统对环境变量总大小有限制（通常 32KB-128KB）
- ✅ **当前设计**: KANBAN 只传递少量元数据，不会超限

## 总结

`env["HERMES_KANBAN_TASK"] = task.id` 的影响范围是：

1. **Worker 子进程**: ✅ 完全可见
2. **Worker 的所有子进程**: ✅ 继承可见
3. **Dispatcher 父进程**: ❌ 不可见（隔离）
4. **其他 Worker**: ❌ 不可见（隔离）

这是一个**进程级别的作用域**，通过 `subprocess.Popen(env=env)` 传递给子进程，实现了：
- ✅ 任务隔离
- ✅ 安全验证
- ✅ 便利访问
- ✅ 跨后端兼容
