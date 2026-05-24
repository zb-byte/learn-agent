# Hermes Agent - delegate_task 子任务模式详解

> 基于源码：`/Users/wangzhongbin/Documents/code/fangzhen/hermes-agent/tools/delegate_tool.py`

## 1. 核心定位

`delegate_task` 是 Hermes Agent 的**工具（Tool）**，而非独立的调度系统。它通过工具调用机制实现子 Agent 的创建和管理。

### 1.1 本质

```python
# tools/delegate_tool.py
registry.register(
    name="delegate_task",
    toolset="delegation",
    schema=DELEGATE_TASK_SCHEMA,
    handler=lambda args, **kw: delegate_task(...),
    check_fn=check_delegate_requirements,
    emoji="🔀",
)
```

**关键特性**：
- **工具类型**：注册在 `tools.registry` 中的标准工具
- **工具集**：属于 `delegation` toolset
- **调用方式**：模型通过 function calling 调用
- **执行模式**：同步阻塞，父 Agent 等待子 Agent 完成

---

## 2. 架构设计

### 2.1 子 Agent 架构

```python
"""
Delegate Tool -- Subagent Architecture

Spawns child AIAgent instances with isolated context, restricted toolsets,
and their own terminal sessions. Supports single-task and batch (parallel)
modes. The parent blocks until all children complete.

Each child gets:
  - A fresh conversation (no parent history)
  - Its own task_id (own terminal session, file ops cache)
  - A restricted toolset (configurable, with blocked tools always stripped)
  - A focused system prompt built from the delegated goal + context
"""
```

### 2.2 隔离机制

| 隔离维度 | 实现方式 | 说明 |
|---------|---------|------|
| **上下文** | 独立 conversation | 子 Agent 看不到父 Agent 的对话历史 |
| **终端会话** | 独立 task_id | 每个子 Agent 有自己的工作目录和状态 |
| **工具集** | 受限 toolsets | 子 Agent 只能使用指定的工具 |
| **系统提示** | 专用 prompt | 根据 goal + context 构建 |
| **迭代预算** | 独立 budget | 每个子 Agent 有自己的预算（默认 50） |

---

## 3. 调度机制

### 3.1 执行模式

```python
def delegate_task(
    goal: Optional[str] = None,
    context: Optional[str] = None,
    toolsets: Optional[List[str]] = None,
    tasks: Optional[List[Dict[str, Any]]] = None,
    max_iterations: Optional[int] = None,
    role: Optional[str] = None,
    parent_agent=None,
) -> str:
    """
    Spawn one or more child agents to handle delegated tasks.
    
    Supports two modes:
      - Single: provide goal (+ optional context, toolsets, role)
      - Batch:  provide tasks array [{goal, context, toolsets, role}, ...]
    
    Returns JSON with results array, one entry per task.
    """
```

**两种模式**：

#### 单任务模式
```python
# 模型调用
delegate_task(
    goal="Debug the authentication bug",
    context="Error: 401 Unauthorized in login.py:45",
    toolsets=["terminal", "file"]
)
```

#### 批量并行模式
```python
# 模型调用
delegate_task(
    tasks=[
        {"goal": "Research API A", "toolsets": ["web"]},
        {"goal": "Research API B", "toolsets": ["web"]},
        {"goal": "Research API C", "toolsets": ["web"]}
    ]
)
```

### 3.2 并发控制

```python
# 默认配置
_DEFAULT_MAX_CONCURRENT_CHILDREN = 3
DEFAULT_CHILD_TIMEOUT = 600  # 10 分钟超时

def _get_max_concurrent_children() -> int:
    """Read delegation.max_concurrent_children from config."""
    cfg = _load_config()
    val = cfg.get("max_concurrent_children")
    if val is not None:
        result = max(1, int(val))
        if result > 10:
            logger.warning(
                "delegation.max_concurrent_children=%d: each child consumes "
                "API tokens independently. High values multiply cost linearly.",
                result,
            )
        return result
    return _DEFAULT_MAX_CONCURRENT_CHILDREN
```

**并发策略**：
- 单任务：直接执行，无线程池开销
- 批量任务：使用 `ThreadPoolExecutor` 并行执行
- 最大并发数：可配置（默认 3，可通过 `delegation.max_concurrent_children` 调整）

### 3.3 线程池执行

```python
# 批量模式的并发执行
if n_tasks == 1:
    # 单任务 -- 直接运行
    result = _run_single_child(0, task["goal"], child, parent_agent)
    results.append(result)
else:
    # 批量 -- 并行运行
    with ThreadPoolExecutor(max_workers=max_children) as executor:
        futures = {}
        for i, t, child in children:
            future = executor.submit(
                _run_single_child,
                task_index=i,
                goal=t["goal"],
                child=child,
                parent_agent=parent_agent,
            )
            futures[future] = i
        
        # 等待所有任务完成
        for future in futures:
            result = future.result()
            results.append(result)
```

---

## 4. 上下文继承

### 4.1 不继承的内容

```python
# _build_child_agent 函数
child = AIAgent(
    # ... 其他参数
    skip_context_files=True,      # ❌ 不加载 CLAUDE.md 等上下文文件
    skip_memory=True,              # ❌ 不加载记忆系统
    clarify_callback=None,         # ❌ 不能与用户交互
    # 独立的对话历史
    ephemeral_system_prompt=child_prompt,  # 专用系统提示
)
```

**完全隔离**：
- ❌ 父 Agent 的对话历史
- ❌ CLAUDE.md / MEMORY.md 等上下文文件
- ❌ 记忆系统（memory）
- ❌ 用户交互能力（clarify）

### 4.2 继承的内容

```python
# 从父 Agent 继承的配置
child = AIAgent(
    base_url=effective_base_url,           # ✅ API 端点
    api_key=effective_api_key,             # ✅ API 密钥
    model=effective_model,                 # ✅ 模型名称
    provider=effective_provider,           # ✅ 提供商
    max_tokens=getattr(parent_agent, "max_tokens", None),  # ✅ 最大 token
    reasoning_config=child_reasoning,      # ✅ 推理配置
    fallback_model=parent_fallback,        # ✅ 降级模型链
    platform=parent_agent.platform,        # ✅ 平台标识
    session_db=getattr(parent_agent, "_session_db", None),  # ✅ 会话数据库
    parent_session_id=getattr(parent_agent, "session_id", None),  # ✅ 父会话 ID
)
```

**继承内容**：
- ✅ API 凭证（base_url, api_key, provider）
- ✅ 模型配置（model, max_tokens, reasoning_config）
- ✅ 降级链（fallback_model）
- ✅ 会话关联（session_db, parent_session_id）
- ✅ 平台标识（platform）

### 4.3 工作目录提示

```python
def _resolve_workspace_hint(parent_agent) -> Optional[str]:
    """Best-effort local workspace hint for child prompts."""
    candidates = [
        os.getenv("TERMINAL_CWD"),
        getattr(parent_agent, "terminal_cwd", None),
        getattr(parent_agent, "cwd", None),
    ]
    for candidate in candidates:
        if candidate and os.path.isabs(candidate) and os.path.isdir(candidate):
            return candidate
    return None

# 在子 Agent 的系统提示中注入
if workspace_path:
    parts.append(
        "\nWORKSPACE PATH:\n"
        f"{workspace_path}\n"
        "Use this exact path for local repository/workdir operations."
    )
```

**工作目录继承**：
- 尝试从父 Agent 获取工作目录
- 注入到子 Agent 的系统提示中
- 避免子 Agent 猜测路径

---

## 5. 记忆系统

### 5.1 完全隔离

```python
child = AIAgent(
    skip_memory=True,  # 子 Agent 不加载记忆系统
)
```

**子 Agent 无法**：
- ❌ 读取父 Agent 的记忆
- ❌ 写入共享的 MEMORY.md
- ❌ 访问记忆工具（memory tool）

### 5.2 阻止的工具

```python
# 子 Agent 永远无法访问的工具
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # ❌ 禁止递归委托（除非是 orchestrator）
    "clarify",         # ❌ 禁止用户交互
    "memory",          # ❌ 禁止写入共享记忆
    "send_message",    # ❌ 禁止跨平台副作用
    "execute_code",    # ❌ 禁止脚本执行（应逐步推理）
])
```

### 5.3 委托结果通知

```python
# 父 Agent 的记忆管理器会收到委托结果通知
if parent_agent._memory_manager:
    for entry in results:
        parent_agent._memory_manager.on_delegation(
            task=task_goal,
            result=entry.get("summary", ""),
            child_session_id=child.session_id,
        )
```

**记忆流向**：
- 子 Agent → 父 Agent：通过 `on_delegation` 回调
- 父 Agent 可以选择性地记录委托结果
- 子 Agent 本身不写入记忆

---

## 6. 递归委托（嵌套 Agent）

### 6.1 角色系统

```python
# 两种角色
role: str = "leaf" | "orchestrator"

# leaf（默认）：不能进一步委托
# orchestrator：可以创建自己的子 Agent
```

### 6.2 深度限制

```python
MAX_DEPTH = 1  # 默认：扁平结构（父 -> 子）
_MAX_SPAWN_DEPTH_CAP = 3  # 最大深度上限

def _get_max_spawn_depth() -> int:
    """Read delegation.max_spawn_depth from config, clamped to [1, 3]."""
    cfg = _load_config()
    val = cfg.get("max_spawn_depth")
    if val is None:
        return MAX_DEPTH
    clamped = max(_MIN_SPAWN_DEPTH, min(_MAX_SPAWN_DEPTH_CAP, int(val)))
    return clamped
```

**深度控制**：
- `depth 0`：父 Agent
- `depth 1`：子 Agent（默认最大深度）
- `depth 2-3`：孙 Agent（需配置 `delegation.max_spawn_depth`）

### 6.3 角色降级

```python
# _build_child_agent 函数
child_depth = getattr(parent_agent, "_delegate_depth", 0) + 1
max_spawn = _get_max_spawn_depth()
orchestrator_ok = _get_orchestrator_enabled() and child_depth < max_spawn
effective_role = role if (role == "orchestrator" and orchestrator_ok) else "leaf"
```

**降级规则**：
1. 如果 `child_depth >= max_spawn_depth`：强制降级为 `leaf`
2. 如果 `orchestrator_enabled=false`：强制降级为 `leaf`
3. 否则：保持请求的角色

### 6.4 Orchestrator 工具集

```python
# Orchestrators 保留 delegation 工具集
if effective_role == "orchestrator" and "delegation" not in child_toolsets:
    child_toolsets.append("delegation")
```

**Orchestrator 特权**：
- 可以调用 `delegate_task` 创建自己的子 Agent
- 仍然不能使用 `clarify`, `memory`, `send_message`, `execute_code`

---

## 7. 工具集（Toolsets）

### 7.1 工具集继承

```python
# 默认工具集
DEFAULT_TOOLSETS = ["terminal", "file", "web"]

# 从父 Agent 继承
parent_enabled = getattr(parent_agent, "enabled_toolsets", None)
if parent_enabled is not None:
    parent_toolsets = set(parent_enabled)
else:
    parent_toolsets = set(DEFAULT_TOOLSETS)
```

### 7.2 工具集交集

```python
if toolsets:
    # 子 Agent 请求的工具集必须是父 Agent 的子集
    expanded_parent = _expand_parent_toolsets(parent_toolsets)
    child_toolsets = [t for t in toolsets if t in expanded_parent]
```

**限制规则**：
- 子 Agent 不能获得父 Agent 没有的工具
- 子 Agent 可以请求更窄的工具集
- 交集为空时使用默认工具集

### 7.3 阻止的工具集

```python
def _strip_blocked_tools(toolsets: List[str]) -> List[str]:
    """Remove toolsets that contain only blocked tools."""
    blocked_toolset_names = {
        "delegation",      # 除非是 orchestrator
        "clarify",         # 禁止用户交互
        "memory",          # 禁止记忆写入
        "code_execution",  # 禁止 execute_code
    }
    return [t for t in toolsets if t not in blocked_toolset_names]
```

### 7.4 MCP 工具集继承

```python
def _get_inherit_mcp_toolsets() -> bool:
    """Whether narrowed child toolsets should keep the parent's MCP toolsets."""
    cfg = _load_config()
    return is_truthy_value(cfg.get("inherit_mcp_toolsets"), default=True)

# 保留父 Agent 的 MCP 工具集
if _get_inherit_mcp_toolsets():
    child_toolsets = _preserve_parent_mcp_toolsets(
        child_toolsets, parent_toolsets
    )
```

**MCP 工具集**：
- 默认情况下，子 Agent 继承父 Agent 的所有 MCP 工具集
- 可通过 `delegation.inherit_mcp_toolsets=false` 禁用

---

## 8. 系统提示构建

### 8.1 基础提示

```python
def _build_child_system_prompt(
    goal: str,
    context: Optional[str] = None,
    workspace_path: Optional[str] = None,
    role: str = "leaf",
    max_spawn_depth: int = 2,
    child_depth: int = 1,
) -> str:
    parts = [
        "You are a focused subagent working on a specific delegated task.",
        "",
        f"YOUR TASK:\n{goal}",
    ]
    if context and context.strip():
        parts.append(f"\nCONTEXT:\n{context}")
    if workspace_path:
        parts.append(
            "\nWORKSPACE PATH:\n"
            f"{workspace_path}\n"
            "Use this exact path for local repository/workdir operations."
        )
```

**提示结构**：
1. 角色定位：专注的子 Agent
2. 任务目标：`goal` 参数
3. 上下文信息：`context` 参数
4. 工作目录：从父 Agent 继承

### 8.2 Orchestrator 提示

```python
if role == "orchestrator":
    parts.append(
        "\n## Subagent Spawning (Orchestrator Role)\n"
        "You have access to the `delegate_task` tool and CAN spawn "
        "your own subagents to parallelize independent work.\n\n"
        "WHEN to delegate:\n"
        "- The goal decomposes into 2+ independent subtasks that can "
        "run in parallel (e.g. research A and B simultaneously).\n"
        "- A subtask is reasoning-heavy and would flood your context "
        "with intermediate data.\n\n"
        "WHEN NOT to delegate:\n"
        "- Single-step mechanical work — do it directly.\n"
        "- Trivial tasks you can execute in one or two tool calls.\n"
    )
```

**Orchestrator 指导**：
- 何时委托：并行任务、推理密集型任务
- 何时不委托：简单任务、机械操作
- 深度限制：明确告知当前深度和最大深度

---

## 9. 技能（Skills）加载

### 9.1 完全不加载

```python
child = AIAgent(
    skip_context_files=True,  # 跳过所有上下文文件
)
```

**子 Agent 无法访问**：
- ❌ 父 Agent 的技能（skills）
- ❌ CLAUDE.md 中定义的技能
- ❌ 任何自定义技能

### 9.2 原因

子 Agent 的设计哲学是**完全隔离**：
- 只通过 `goal` 和 `context` 参数接收信息
- 不依赖任何外部上下文
- 确保可预测性和可重现性

---

## 10. 完整执行流程

### 10.1 构建阶段（主线程）

```python
# 1. 保存父 Agent 的工具名称
_parent_tool_names = list(_model_tools._last_resolved_tool_names)

# 2. 构建所有子 Agent（主线程，线程安全）
children = []
for i, t in enumerate(task_list):
    child = _build_child_agent(
        task_index=i,
        goal=t["goal"],
        context=t.get("context"),
        toolsets=t.get("toolsets"),
        model=creds["model"],
        max_iterations=effective_max_iter,
        task_count=n_tasks,
        parent_agent=parent_agent,
        role=effective_role,
    )
    children.append((i, t, child))

# 3. 恢复父 Agent 的工具名称
_model_tools._last_resolved_tool_names = _parent_tool_names
```

### 10.2 执行阶段（工作线程）

```python
def _run_single_child(task_index, goal, child, parent_agent):
    # 1. 启动心跳线程（保持父 Agent 活跃）
    _heartbeat_thread.start()
    
    # 2. 注册子 Agent（用于中断传播）
    _register_subagent(record)
    
    # 3. 运行子 Agent（带超时）
    with ThreadPoolExecutor(max_workers=1) as executor:
        future = executor.submit(
            child.run_conversation,
            user_message=goal,
            task_id=child_task_id,
        )
        result = future.result(timeout=child_timeout)
    
    # 4. 构建结果
    return {
        "task_index": task_index,
        "status": status,
        "summary": summary,
        "api_calls": api_calls,
        "duration_seconds": duration,
        "tokens": {...},
        "tool_trace": [...],
    }
```

### 10.3 清理阶段

```python
finally:
    # 1. 停止心跳线程
    _heartbeat_stop.set()
    _heartbeat_thread.join(timeout=5)
    
    # 2. 注销子 Agent
    _unregister_subagent(_subagent_id)
    
    # 3. 释放凭证租约
    if child_pool and leased_cred_id:
        child_pool.release_lease(leased_cred_id)
    
    # 4. 恢复父 Agent 工具名称
    _model_tools._last_resolved_tool_names = _parent_tool_names
    
    # 5. 关闭子 Agent 资源
    if hasattr(child, "close"):
        child.close()
```

---

## 11. 结果返回格式

### 11.1 成功结果

```json
{
  "results": [
    {
      "task_index": 0,
      "status": "completed",
      "summary": "Found the bug in login.py:45. The issue was...",
      "api_calls": 12,
      "duration_seconds": 45.3,
      "model": "anthropic/claude-opus-4.6",
      "exit_reason": "completed",
      "tokens": {
        "input": 15000,
        "output": 3000
      },
      "tool_trace": [
        {"tool": "read_file", "args_bytes": 50, "result_bytes": 1200, "status": "ok"},
        {"tool": "terminal", "args_bytes": 80, "result_bytes": 500, "status": "ok"}
      ]
    }
  ],
  "total_duration_seconds": 45.3
}
```

### 11.2 失败结果

```json
{
  "results": [
    {
      "task_index": 0,
      "status": "failed",
      "summary": null,
      "error": "Subagent did not produce a response.",
      "api_calls": 5,
      "duration_seconds": 30.0,
      "exit_reason": "max_iterations"
    }
  ],
  "total_duration_seconds": 30.0
}
```

### 11.3 超时结果

```json
{
  "results": [
    {
      "task_index": 0,
      "status": "timeout",
      "summary": null,
      "error": "Subagent timed out after 600s with 3 API call(s) completed",
      "api_calls": 3,
      "duration_seconds": 600.0,
      "exit_reason": "timeout"
    }
  ],
  "total_duration_seconds": 600.0
}
```

### 11.4 状态类型

```python
status: str = "completed" | "failed" | "timeout" | "interrupted" | "error"

# completed: 子 Agent 成功完成任务
# failed: 子 Agent 未能产生有效输出
# timeout: 子 Agent 超时
# interrupted: 父 Agent 被中断，子 Agent 被取消
# error: 子 Agent 执行过程中抛出异常
```

---

## 12. 心跳机制

### 12.1 目的

```python
# 心跳间隔
_HEARTBEAT_INTERVAL = 30  # 每 30 秒一次

# 防止父 Agent 被网关超时杀死
def _heartbeat_loop():
    while not _heartbeat_stop.wait(_HEARTBEAT_INTERVAL):
        touch = getattr(parent_agent, "_touch_activity", None)
        if touch:
            touch(f"delegate_task: subagent {task_index} working")
```

**作用**：
- 定期更新父 Agent 的活动时间戳
- 防止网关因"无活动"而杀死父 Agent
- 子 Agent 工作时，父 Agent 保持活跃

### 12.2 停滞检测

```python
# 停滞阈值
_HEARTBEAT_STALE_CYCLES_IDLE = 15      # 空闲 15 个周期 = 450s
_HEARTBEAT_STALE_CYCLES_IN_TOOL = 40   # 工具内 40 个周期 = 1200s

# 检测逻辑
iter_advanced = child_iter > _last_seen_iter[0]
tool_changed = child_tool != _last_seen_tool[0]
if iter_advanced or tool_changed:
    _stale_count[0] = 0
else:
    _stale_count[0] += 1

if _stale_count[0] >= stale_limit:
    logger.warning("Subagent appears stale — stopping heartbeat")
    break  # 停止心跳，让网关超时触发
```

**停滞判断**：
- 空闲状态：450 秒无进展 → 停滞
- 工具内状态：1200 秒无进展 → 停滞
- 停滞后停止心跳，让网关超时机制介入

---

## 13. 文件状态协调

### 13.1 跨 Agent 文件追踪

```python
# 记录父 Agent 已读取的文件
parent_reads_snapshot = list(file_state.known_reads(parent_task_id))

# 子 Agent 运行后，检查是否修改了父 Agent 读过的文件
sibling_writes = file_state.writes_since(
    parent_task_id, wall_start, parent_reads_snapshot
)

if sibling_writes:
    mod_paths = sorted({p for paths in sibling_writes.values() for p in paths})
    reminder = (
        "\n\n[NOTE: subagent modified files the parent previously read — "
        "re-read before editing: " + ", ".join(mod_paths[:8]) + "]"
    )
    entry["summary"] = entry["summary"] + reminder
```

**协调机制**：
- 父 Agent 读取文件 → 记录到 `file_state`
- 子 Agent 修改文件 → 记录到 `file_state`
- 子 Agent 完成后 → 检查冲突，提醒父 Agent 重新读取

### 13.2 独立 task_id

```python
# 每个子 Agent 有独立的 task_id
child_task_id = _subagent_id or f"subagent-{task_index}-{uuid.uuid4().hex[:8]}"

# 用于隔离文件操作缓存和终端会话
result = child.run_conversation(
    user_message=goal,
    task_id=child_task_id,
)
```

---

## 14. 成本聚合

### 14.1 子 Agent 成本追踪

```python
# 捕获子 Agent 的成本
_child_cost_usd = float(getattr(child, "session_estimated_cost_usd", 0.0) or 0.0)

entry["_child_cost_usd"] = _child_cost_usd
```

### 14.2 成本汇总到父 Agent

```python
# 聚合所有子 Agent 的成本
_children_cost_total = 0.0
for entry in results:
    child_cost = entry.pop("_child_cost_usd", 0.0)
    _children_cost_total += float(child_cost)

# 累加到父 Agent 的会话成本
if _children_cost_total > 0.0:
    current = float(getattr(parent_agent, "session_estimated_cost_usd", 0.0) or 0.0)
    parent_agent.session_estimated_cost_usd = current + _children_cost_total
```

**成本流向**：
- 子 Agent 记录自己的 API 调用成本
- 父 Agent 汇总所有子 Agent 的成本
- 嵌套 Orchestrator：每层汇总直接子 Agent，自然累积

---

## 15. 中断传播

### 15.1 注册机制

```python
# 注册子 Agent 到父 Agent
if hasattr(parent_agent, "_active_children"):
    with parent_agent._active_children_lock:
        parent_agent._active_children.append(child)
```

### 15.2 中断传播

```python
# 父 Agent 被中断时，自动传播到所有子 Agent
def interrupt_subagent(subagent_id: str) -> bool:
    """Request that a single running subagent stop."""
    with _active_subagents_lock:
        record = _active_subagents.get(subagent_id)
    if not record:
        return False
    agent = record.get("agent")
    if agent:
        agent.interrupt(f"Interrupted via TUI ({subagent_id})")
        return True
    return False
```

### 15.3 清理

```python
finally:
    # 注销子 Agent
    if hasattr(parent_agent, "_active_children"):
        with parent_agent._active_children_lock:
            parent_agent._active_children.remove(child)
```

---

## 16. 进度回调

### 16.1 回调构建

```python
def _build_child_progress_callback(
    task_index: int,
    goal: str,
    parent_agent,
    task_count: int = 1,
    subagent_id: Optional[str] = None,
    parent_id: Optional[str] = None,
    depth: Optional[int] = None,
    model: Optional[str] = None,
    toolsets: Optional[List[str]] = None,
) -> Optional[callable]:
    """Build a callback that relays child agent tool calls to parent display."""
```

### 16.2 事件类型

```python
class DelegateEvent(str, enum.Enum):
    TASK_SPAWNED = "delegate.task_spawned"
    TASK_PROGRESS = "delegate.task_progress"
    TASK_COMPLETED = "delegate.task_completed"
    TASK_FAILED = "delegate.task_failed"
    TASK_THINKING = "delegate.task_thinking"
    TASK_TOOL_STARTED = "delegate.tool_started"
    TASK_TOOL_COMPLETED = "delegate.tool_completed"
```

### 16.3 显示路径

**CLI 路径**：
```python
if spinner:
    spinner.print_above(f" {prefix}├─ 🔀 {short_goal}")
    spinner.print_above(f' {prefix}├─ 💭 "{thinking_text}"')
    spinner.print_above(f" {prefix}├─ {emoji} {tool_name}")
```

**Gateway 路径**：
```python
if parent_cb:
    parent_cb("subagent.start", preview=goal)
    parent_cb("subagent.thinking", preview=thinking_text)
    parent_cb("subagent.tool", tool_name, preview, args)
    parent_cb("subagent.complete", preview=summary, status=status)
```

---

## 17. 配置选项

### 17.1 delegation 配置块

```yaml
# config.yaml
delegation:
  # 子 Agent 的最大迭代次数（默认 50）
  max_iterations: 50
  
  # 最大并发子 Agent 数量（默认 3）
  max_concurrent_children: 3
  
  # 子 Agent 超时时间（秒，默认 600）
  child_timeout_seconds: 600
  
  # 最大嵌套深度（默认 1，范围 1-3）
  max_spawn_depth: 1
  
  # 是否启用 orchestrator 角色（默认 true）
  orchestrator_enabled: true
  
  # 子 Agent 是否继承父 Agent 的 MCP 工具集（默认 true）
  inherit_mcp_toolsets: true
  
  # 子 Agent 危险命令自动批准（默认 false）
  subagent_auto_approve: false
  
  # 子 Agent 使用的模型（可选）
  model: "anthropic/claude-sonnet-4.6"
  
  # 子 Agent 使用的提供商（可选）
  provider: "openrouter"
  
  # 子 Agent 使用的 API 端点（可选）
  base_url: "https://api.example.com/v1"
  
  # 子 Agent 使用的 API 密钥（可选）
  api_key: "sk-..."
  
  # 推理努力级别（可选）
  reasoning_effort: "medium"
```

### 17.2 配置优先级

```python
def _load_config() -> dict:
    """Load delegation config from CLI_CONFIG or persistent config."""
    # 1. 运行时配置（CLI_CONFIG）
    from cli import CLI_CONFIG
    cfg = CLI_CONFIG.get("delegation") or {}
    if cfg:
        return cfg
    
    # 2. 持久化配置（config.yaml）
    from hermes_cli.config import load_config
    full = load_config()
    return full.get("delegation") or {}
```

**优先级**：
1. 运行时配置（`CLI_CONFIG`）
2. 持久化配置（`config.yaml`）
3. 默认值

---

## 18. 安全机制

### 18.1 危险命令批准

```python
def _subagent_auto_deny(command: str, description: str, **kwargs) -> str:
    """Auto-deny dangerous commands in subagent threads (safe default)."""
    logger.warning(
        "Subagent auto-denied dangerous command: %s (%s). "
        "Set delegation.subagent_auto_approve: true to allow.",
        command, description,
    )
    return "deny"

def _subagent_auto_approve(command: str, description: str, **kwargs) -> str:
    """Auto-approve dangerous commands in subagent threads (opt-in YOLO)."""
    logger.warning(
        "Subagent auto-approved dangerous command: %s (%s)",
        command, description,
    )
    return "once"
```

**默认行为**：
- 子 Agent 的危险命令**自动拒绝**
- 可通过 `delegation.subagent_auto_approve=true` 启用自动批准
- 避免子 Agent 在工作线程中调用 `input()` 导致死锁

### 18.2 暂停机制

```python
_spawn_paused: bool = False

def set_spawn_paused(paused: bool) -> bool:
    """Globally block/unblock new delegate_task spawns."""
    global _spawn_paused
    with _spawn_pause_lock:
        _spawn_paused = bool(paused)
        return _spawn_paused

# 检查暂停状态
if is_spawn_paused():
    return tool_error(
        "Delegation spawning is paused. Clear the pause via the TUI "
        "or the `delegation.pause` RPC before retrying."
    )
```

**暂停功能**：
- 全局暂停新的子 Agent 创建
- 已运行的子 Agent 继续执行
- 用于控制失控的委托树

---

## 19. 超时诊断

### 19.1 0-API-Call 超时

```python
# 当子 Agent 超时且未发起任何 API 调用时，生成诊断报告
if is_timeout and child_api_calls == 0:
    diagnostic_path = _dump_subagent_timeout_diagnostic(
        child=child,
        task_index=task_index,
        timeout_seconds=float(child_timeout),
        duration_seconds=float(duration),
        worker_thread=_worker_thread_holder.get("t"),
        goal=goal,
    )
```

### 19.2 诊断内容

```python
def _dump_subagent_timeout_diagnostic(...) -> Optional[str]:
    """Write a structured diagnostic dump for a subagent that timed out."""
    lines = [
        "# Subagent timeout diagnostic",
        "## Timeout",
        f"  configured_timeout: {timeout_seconds}s",
        f"  actual_duration: {duration_seconds:.2f}s",
        "## Goal",
        goal_preview,
        "## Child config",
        f"  model: {child.model}",
        f"  provider: {child.provider}",
        "## Toolsets",
        f"  enabled_toolsets: {child.enabled_toolsets}",
        "## Prompt / schema sizes",
        f"  system_prompt_bytes: {len(sys_prompt.encode('utf-8'))}",
        "## Activity summary",
        child.get_activity_summary(),
        "## Worker thread stack at timeout",
        traceback.format_stack(worker_frame),
    ]
    dump_path.write_text("\n".join(lines))
    return str(dump_path)
```

**诊断文件位置**：
- `~/.hermes/logs/subagent-timeout-{subagent_id}-{timestamp}.log`

---

## 20. 最佳实践

### 20.1 何时使用 delegate_task

✅ **适合的场景**：
- 推理密集型子任务（调试、代码审查、研究综合）
- 会产生大量中间数据的任务
- 可并行的独立工作流（同时研究 A 和 B）
- 需要隔离上下文的任务

❌ **不适合的场景**：
- 无需推理的机械多步骤工作 → 使用 `execute_code`
- 单个工具调用 → 直接调用工具
- 需要用户交互的任务 → 子 Agent 无法使用 `clarify`
- 需要持久化的长期工作 → 使用 `cronjob` 或后台终端

### 20.2 上下文传递

```python
# ❌ 错误：假设子 Agent 知道上下文
delegate_task(goal="Fix the bug")

# ✅ 正确：明确传递所有相关信息
delegate_task(
    goal="Fix the authentication bug",
    context="""
    Error: 401 Unauthorized in login.py:45
    Stack trace: ...
    Project structure: src/auth/login.py, src/auth/token.py
    Constraints: Must maintain backward compatibility
    """
)
```

### 20.3 语言一致性

```python
# ❌ 错误：用户用中文提问，但没告诉子 Agent
delegate_task(goal="研究这个 API")

# ✅ 正确：明确指定输出语言
delegate_task(
    goal="研究这个 API",
    context="请用中文回复所有结果"
)
```

### 20.4 结果验证

```python
# ❌ 错误：盲目信任子 Agent 的自我报告
result = delegate_task(goal="Upload file to S3")
# 假设上传成功

# ✅ 正确：验证子 Agent 的声明
result = delegate_task(
    goal="Upload file to S3 and return the S3 URL",
    context="Must return the full S3 URL in the summary"
)
# 解析 URL，验证文件是否真的存在
```

---

## 21. 总结

### 21.1 核心特性

| 特性 | 实现 |
|-----|------|
| **本质** | 工具（Tool），通过 function calling 调用 |
| **隔离** | 独立上下文、终端、工具集、预算 |
| **继承** | API 凭证、模型配置、降级链 |
| **记忆** | 完全隔离，子 Agent 无法访问父 Agent 记忆 |
| **技能** | 不加载，子 Agent 无法使用父 Agent 技能 |
| **递归** | 支持，通过 `role="orchestrator"` 启用 |
| **并发** | 支持，批量模式并行执行 |
| **超时** | 600 秒默认超时，可配置 |

### 21.2 设计哲学

1. **完全隔离**：子 Agent 只通过 `goal` 和 `context` 接收信息
2. **同步阻塞**：父 Agent 等待子 Agent 完成
3. **结果聚合**：只返回摘要，中间过程不进入父 Agent 上下文
4. **安全第一**：默认拒绝危险命令，支持暂停机制
5. **可观测性**：完整的进度回调、成本追踪、超时诊断

### 21.3 与主 Agent 的区别

| 维度 | 主 Agent | 子 Agent |
|-----|---------|---------|
| 对话历史 | 完整历史 | 独立对话 |
| 上下文文件 | 加载 CLAUDE.md | 不加载 |
| 记忆系统 | 可读写 | 完全隔离 |
| 用户交互 | 可使用 clarify | 不可用 |
| 工具集 | 完整工具集 | 受限工具集 |
| 迭代预算 | 90（默认） | 50（默认） |
| 递归委托 | 可以 | 取决于 role |

---

## 22. 参考资料

### 22.1 源码文件

- `tools/delegate_tool.py` - 主实现文件（2800+ 行）
- `agent/agent_init.py` - Agent 初始化
- `agent/iteration_budget.py` - 迭代预算
- `tools/registry.py` - 工具注册表

### 22.2 配置文件

- `config.yaml` - 用户配置
- `delegation` 配置块 - 委托相关配置

### 22.3 相关概念

- **Subagent Architecture** - 子 Agent 架构
- **Orchestrator Pattern** - 编排器模式
- **Tool Calling** - 工具调用机制
- **Context Isolation** - 上下文隔离
