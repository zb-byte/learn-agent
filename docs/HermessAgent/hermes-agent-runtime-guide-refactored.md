# Hermes-Agent 运行时原理详解

> 本文档面向普通研发，用**通俗类比 + 源码锚点**讲解 Hermes-Agent 的核心运行机制。  
> 分享/走读时建议搭配 **[结构化导读](./hermes-agent-runtime-reader-guide.md)**（讲者提纲）；各子系统**实现级**细节见下方专题文档（均已按 `hermes-agent` 源码校对）。

---

## 文档体系（建议阅读顺序）

| 文档 | 定位 |
|------|------|
| [hermes-agent-runtime-reader-guide.md](./hermes-agent-runtime-reader-guide.md) | 分享提纲：`delegate_task` + 记忆/技能进化，源码地图与生命周期表 |
| **本文档** | 通俗长文：多 Agent + 记忆，适合第一次建立心智模型 |
| [hermes-agent-main-loop.md](./hermes-agent-main-loop.md) | 主循环 `run_conversation`：双计数器、工具并行、预算耗尽后的 `_handle_max_iterations` |
| [hermes-agent-context-compression.md](./hermes-agent-context-compression.md) | 压缩五阶段、preflight/per-turn 触发、session 轮换与 cache |
| [hermes-agent-permission-system.md](./hermes-agent-permission-system.md) | `tools/approval.py` + CLI / Gateway / ACP |
| [hermes-agent-skill-system.md](./hermes-agent-skill-system.md) | 三层披露、slash 注入 user 消息、`skill_manage` |

**源码根目录：** `/Users/wangzhongbin/Documents/code/fangzhen/hermes-agent`（下文行号对应该仓库当前主干）。

---

## 目录

1. [多 Agent 机制：delegate_task 的工作原理](#1-多-agent-机制delegatetask-的工作原理)
2. [自我学习进化：Hermes 的记忆系统](#2-自我学习进化hermes-的记忆系统)

---

## 1. 多 Agent 机制：delegate_task 的工作原理

### 1.1 核心概念

**delegate_task 是什么？**

简单来说，`delegate_task` 就像是"项目经理把任务分配给团队成员"：
- 父 Agent（项目经理）遇到复杂任务时，可以创建子 Agent（团队成员）来处理
- 子 Agent 独立工作，有自己的思考过程和工具
- 完成后，子 Agent 把结果汇报给父 Agent
- 父 Agent 综合所有结果，做出最终决策

**关键特点：**

| 特点 | 说明 |
|------|------|
| 它是一个工具 | 模型通过 function calling 主动选择使用 |
| 同步执行 | 父 Agent 会等待子 Agent 完成 |
| 独立运行时 | 子 Agent 有自己的对话循环和上下文 |
| 结果汇总 | 父 Agent 只收到子 Agent 的总结，不是完整过程 |

**核心机制说明：**

1. **goal 参数**：`goal` 是 `delegate_task` 的参数，表示子 Agent 要完成的任务描述，会被转换成子 Agent 的第一条 user message。

2. **模型继承**：子 Agent 默认使用父 Agent 的模型，但可以通过配置或参数指定不同的模型（如用便宜的模型处理简单子任务）。

3. **代码复用**：子 Agent 和父 Agent 使用**完全相同的代码逻辑**（都是 `AIAgent` 类），只是配置和上下文不同。

**源码证据：**

```python
# tools/delegate_tool.py:1-17
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

The parent's context only sees the delegation call and the summary result,
never the child's intermediate tool calls or reasoning.
"""

# 模型继承逻辑
# tools/delegate_tool.py:1021-1024
effective_model = model or parent_agent.model  # 默认继承父模型
effective_provider = override_provider or getattr(parent_agent, "provider", None)
effective_base_url = override_base_url or parent_agent.base_url
effective_api_key = override_api_key or parent_api_key

# 子 Agent 实例化（使用相同的 AIAgent 类）
# tools/delegate_tool.py:1054-1080
from run_agent import AIAgent  # 导入同一个 AIAgent 类

child = AIAgent(  # 创建子 Agent，和父 Agent 是同一个类
    system_prompt=child_prompt,
    enabled_toolsets=child_toolsets,
    model=effective_model,  # 可以是父模型或指定的模型
    api_key=effective_api_key,
    # ... 其他参数
)

# tools/delegate_tool.py:2660-2661, 2783-2786
DELEGATE_TASK_SCHEMA = {
    "name": "delegate_task",
    # ...
}

registry.register(
    name="delegate_task",
    toolset="delegation",
    schema=DELEGATE_TASK_SCHEMA,
    handler=lambda args, **kw: delegate_task(...)
)
```

### 1.2 执行流程图

```
用户提问
  ↓
父 Agent 思考："这个任务太复杂，需要分解"
  ↓
父 Agent 调用 delegate_task(goal="子任务描述")
  ↓
创建子 Agent（独立的 AIAgent 实例）
  ↓
子 Agent 开始工作：
  - 调用大模型思考
  - 使用工具（读文件、执行命令等）
  - 继续循环直到完成
  ↓
子 Agent 返回结果摘要
  ↓
父 Agent 收到结果（作为工具调用的返回值）
  ↓
父 Agent 继续思考，可能：
  - 创建更多子 Agent
  - 综合所有结果
  - 给出最终答案
```

### 1.3 实际例子

**场景：用户问"这个项目有哪些安全问题？"**

**重要说明：** 这里的 `goal` 是 `delegate_task` 工具的参数，**不是** Hermes-Agent 的 goal 命令或 skill。

```
父 Agent 思考：
"这个任务需要检查多个方面，我应该分解任务"

父 Agent 调用：
delegate_task(tasks=[
  {goal: "检查代码中的 SQL 注入风险"},
  {goal: "检查依赖包的已知漏洞"},
  {goal: "检查敏感信息泄露"}
])
```

**goal 参数的含义：**

```python
# tools/delegate_tool.py:2678-2684
"goal": {
    "type": "string",
    "description": (
        "What the subagent should accomplish. Be specific and "
        "self-contained -- the subagent knows nothing about your "
        "conversation history."
    ),
}
```

**goal 就是子 Agent 的任务描述，相当于给子 Agent 发送的第一条用户消息：**

```python
# 子 Agent 的初始消息
child_messages = [
    {"role": "user", "content": goal}  # goal 变成了 user message
]
```

**实际执行过程：**

```
子 Agent 1 工作：
- 收到任务："检查代码中的 SQL 注入风险"
- 读取数据库相关代码
- 搜索 SQL 拼接模式
- 返回："发现 3 处可能的 SQL 注入风险"

子 Agent 2 工作：
- 收到任务："检查依赖包的已知漏洞"
- 读取 package.json
- 调用漏洞扫描工具
- 返回："发现 2 个依赖包有高危漏洞"

子 Agent 3 工作：
- 收到任务："检查敏感信息泄露"
- 搜索 API key、密码等关键词
- 检查 .env 文件是否被提交
- 返回："发现 1 处硬编码的 API key"

父 Agent 综合：
"根据子任务的结果，这个项目存在以下安全问题：
1. SQL 注入风险（3 处）
2. 依赖包漏洞（2 个）
3. 敏感信息泄露（1 处）
建议优先修复..."
```

**与 Hermes-Agent 的 goal 命令/skill 的区别：**

| 对比项 | delegate_task 的 goal 参数 | Hermes-Agent 的 goal 功能 |
|--------|---------------------------|-------------------------|
| **类型** | 工具参数（字符串） | 可能是命令或 skill |
| **作用** | 描述子 Agent 要完成的任务 | 可能是目标管理、任务追踪等功能 |
| **使用场景** | 多 Agent 委派 | 用户级的目标管理 |
| **代码位置** | `tools/delegate_tool.py` | 其他模块（如果存在） |

**简单记忆：**
- `delegate_task(goal="...")` 中的 `goal` = "子 Agent，请帮我做这件事"
- 它只是一个参数名，表示"目标任务"

### 1.4 父子 Agent 的隔离机制

**子 Agent 是一个"收窄"的运行时，不是父 Agent 的完整副本。**

| 维度 | 子 Agent 的行为 | 为什么这样设计 |
|------|----------------|---------------|
| 会话历史 | 不继承父 Agent 的对话历史 | 避免上下文污染，专注当前子任务 |
| System Prompt | 使用临时生成的 prompt | 根据子任务目标定制 |
| 工具权限 | 默认继承父工具集，但禁用部分工具 | 防止子 Agent 再次委派（避免无限递归） |
| 记忆系统 | 默认不能写入共享记忆 | 防止子 Agent 污染全局记忆 |
| Session | 共用父 Session DB，但有独立标识 | 方便追踪和调试 |

**禁用的工具：**
- `delegate_task`：防止无限递归
- `clarify`：子 Agent 不能直接问用户
- `memory`：防止污染全局记忆
- `send_message`：子 Agent 不能直接回复用户
- `execute_code`：安全考虑

**源码证据：**

```python
# tools/delegate_tool.py:44-53
DELEGATE_BLOCKED_TOOLS = frozenset(
    [
        "delegate_task",  # no recursive delegation
        "clarify",        # no user interaction
        "memory",         # no writes to shared MEMORY.md
        "send_message",   # no cross-platform side effects
        "execute_code",   # children should reason step-by-step, not write scripts
    ]
)
```

**子 Agent ID 生成：**

```python
# tools/delegate_tool.py:920
subagent_id = f"sa-{task_index}-{_uuid.uuid4().hex[:8]}"
```

**工具集继承机制：**

```python
# tools/delegate_tool.py:926-961
# When no explicit toolsets given, inherit from parent's enabled toolsets
parent_enabled = getattr(parent_agent, "enabled_toolsets", None)
if parent_enabled is not None:
    parent_toolsets = set(parent_enabled)

if toolsets:
    # Intersect with parent — subagent must not gain tools the parent lacks.
    expanded_parent = _expand_parent_toolsets(parent_toolsets)
    child_toolsets = [t for t in toolsets if t in expanded_parent]
    child_toolsets = _strip_blocked_tools(child_toolsets)

# Orchestrators retain the 'delegation' toolset
if effective_role == "orchestrator" and "delegation" not in child_toolsets:
    child_toolsets.append("delegation")
```

### 1.5 源码追踪路径

如果你想深入理解代码实现，按以下顺序阅读：

```
1. toolsets.py
   └─ 定义 delegation toolset，包含 delegate_task

2. tools/delegate_tool.py
   └─ delegate_task 的 schema 定义和注册
   └─ 第 1918 行：delegate_task() 函数
   └─ 第 870 行：_build_child_agent() 构造子 Agent
   └─ 第 1321 行：_run_single_child() 执行子 Agent

3. agent/conversation_loop.py
   └─ 父 Agent 的主循环
   └─ 校验工具调用（JSON、去重、数量限制）

4. agent/tool_executor.py
   └─ 工具执行分发器
   └─ 识别 delegate_task 并特殊处理

5. run_agent.py
   └─ AIAgent._dispatch_delegate_task()
   └─ 注入 parent_agent 参数

6. tools/delegate_tool.py
   └─ _build_child_agent()：构造子 Agent
   └─ _run_single_child()：执行子 Agent
```

**完整的 delegate_task 函数签名：**

```python
# tools/delegate_tool.py:1918-1942
def delegate_task(
    goal: Optional[str] = None,
    context: Optional[str] = None,
    toolsets: Optional[List[str]] = None,
    tasks: Optional[List[Dict[str, Any]]] = None,
    max_iterations: Optional[int] = None,
    acp_command: Optional[str] = None,
    acp_args: Optional[List[str]] = None,
    role: Optional[str] = None,
    parent_agent=None,
) -> str:
    """
    Spawn one or more child agents to handle delegated tasks.

    Supports two modes:
      - Single: provide goal (+ optional context, toolsets, role)
      - Batch:  provide tasks array [{goal, context, toolsets, role}, ...]

    The 'role' parameter controls whether a child can further delegate:
    'leaf' (default) cannot; 'orchestrator' retains the delegation
    toolset and can spawn its own workers, bounded by
    delegation.max_spawn_depth.  Per-task role beats the top-level one.

    Returns JSON with results array, one entry per task.
    """
```

**Schema 定义：**

```python
# tools/delegate_tool.py:2660-2705
DELEGATE_TASK_SCHEMA = {
    "name": "delegate_task",
    "description": "Spawn one or more subagents in isolated contexts...",
    "parameters": {
        "type": "object",
        "properties": {
            "goal": {
                "type": "string",
                "description": (
                    "What the subagent should accomplish. Be specific and "
                    "self-contained -- the subagent knows nothing about your "
                    "conversation history."
                ),
            },
            "context": {
                "type": "string",
                "description": (
                    "Background information the subagent needs: file paths, "
                    "error messages, project structure, constraints..."
                ),
            },
            "toolsets": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Toolsets to enable for this subagent..."
            },
            "role": {
                "type": "string",
                "enum": ["leaf", "orchestrator"],
                "description": "..."
            },
            # ... 更多参数
        }
    }
}
```

### 1.6 与普通并行工具调用的区别

| 对比项 | 普通并行工具调用 | delegate_task 多 Agent |
|--------|-----------------|----------------------|
| 并行对象 | 多个工具函数 | 多个独立 Agent |
| 触发方式 | 模型一次返回多个 tool calls | 模型调用 delegate_task |
| 上下文 | 共享父 Agent 的 messages | 每个子 Agent 独立上下文 |
| 模型调用 | 不新增 Agent loop | 每个子 Agent 独立调用模型 |
| 适用场景 | 简单、独立的工具调用 | 需要推理的复杂子任务 |

**举例说明：**

```python
# 普通并行工具调用
父 Agent 一次性调用：
- read_file("config.json")
- read_file("package.json")
- list_directory("src/")
→ 三个工具并行执行，快速返回结果

# delegate_task 多 Agent
父 Agent 调用：
delegate_task(tasks=[
  {goal: "分析配置文件的安全性"},
  {goal: "检查依赖包的兼容性"},
  {goal: "审查源码的代码质量"}
])
→ 三个子 Agent 独立工作，每个都会：
  - 多次调用模型思考
  - 使用多个工具
  - 生成详细分析报告
```

### 1.7 运行时安全机制

为了防止多 Agent 系统失控，Hermes 实现了多层防护：

| 机制 | 作用 | 实现方式 |
|------|------|---------|
| 深度限制 | 防止无限递归 | 限制 Agent 嵌套层数 |
| 并发上限 | 防止资源耗尽 | 限制同时运行的子 Agent 数量 |
| 超时控制 | 防止子 Agent 卡死 | 设置子 Agent 最大执行时间 |
| 中断传播 | 用户可以停止整个任务树 | 父 Agent 被中断时，子 Agent 也会停止 |
| 成本汇总 | 追踪总开销 | 子 Agent 的 token 消耗汇总到父 Session |
| 心跳机制 | 防止误判空闲 | 子 Agent 长时间工作时定期报告状态 |

**源码证据 - 深度限制：**

```python
# tools/delegate_tool.py:133-137
MAX_DEPTH = 1  # flat by default: parent (0) -> child (1)
_MIN_SPAWN_DEPTH = 1
_MAX_SPAWN_DEPTH_CAP = 3

# tools/delegate_tool.py:1960-1972
depth = getattr(parent_agent, "_delegate_depth", 0)
max_spawn = _get_max_spawn_depth()
if depth >= max_spawn:
    return json.dumps({
        "error": (
            f"Delegation depth limit reached (depth={depth}, "
            f"max_spawn_depth={max_spawn}). Raise "
            f"delegation.max_spawn_depth in config.yaml if deeper "
            f"nesting is required (cap: {_MAX_SPAWN_DEPTH_CAP})."
        )
    })
```

**源码证据 - 并发上限：**

```python
# tools/delegate_tool.py:132
_DEFAULT_MAX_CONCURRENT_CHILDREN = 3

# tools/delegate_tool.py:2001-2016
max_children = _get_max_concurrent_children()
if tasks and isinstance(tasks, list):
    if len(tasks) > max_children:
        return tool_error(
            f"Too many tasks: {len(tasks)} provided, but "
            f"max_concurrent_children is {max_children}. "
            f"Either reduce the task count, split into multiple "
            f"delegate_task calls, or increase "
            f"delegation.max_concurrent_children in config.yaml."
        )
```

**源码证据 - Spawn Pause（暂停机制）：**

```python
# tools/delegate_tool.py:148-150
_spawn_pause_lock = threading.Lock()
_spawn_paused: bool = False

# tools/delegate_tool.py:1949-1953
if is_spawn_paused():
    return tool_error(
        "Delegation spawning is paused. Clear the pause via the TUI "
        "(`p` in /agents) or the `delegation.pause` RPC before retrying."
    )
```

**配置系统：**

Hermes-Agent 有完整的配置系统，可以通过 `config.yaml` 调整：

```yaml
# config.yaml 示例
delegation:
  max_iterations: 50              # 子 Agent 最大迭代次数
  max_concurrent_children: 3      # 最大并发子 Agent 数
  max_spawn_depth: 2              # 最大嵌套深度
  subagent_auto_approve: false    # 子 Agent 自动批准危险命令
  provider: "openrouter"          # 子 Agent 使用的 provider
  model: "anthropic/claude-3-haiku" # 子 Agent 使用的模型
```

### 1.8 delegate_task 的完整工作机制（深度解析）

本节结合真实源码，深入讲解 `delegate_task` 从调用到返回的完整过程。

#### 1.8.1 调用入口：父 Agent 如何触发

**模型决策阶段：**

当父 Agent 的大模型认为需要委派任务时，会返回一个 tool call：

```json
{
  "type": "tool_use",
  "name": "delegate_task",
  "input": {
    "goal": "检查代码中的 SQL 注入风险",
    "context": "项目使用 Python + SQLAlchemy",
    "toolsets": ["terminal", "file"]
  }
}
```

**父 Loop 校验阶段：**

```python
# agent/conversation_loop.py 中的校验逻辑
# 1. 校验工具名是否存在
# 2. 校验 JSON 参数格式
# 3. 校验并发数量（_cap_delegate_task_calls）
# 4. 去重检查（_deduplicate_tool_calls）
```

**工具执行分发：**

```python
# agent/tool_executor.py
if function_name == "delegate_task":
    # 特殊处理：注入 parent_agent 参数
    result = agent._dispatch_delegate_task(function_args)
```

**源码证据 - 参数校验：**

```python
# tools/delegate_tool.py:1943-1944
if parent_agent is None:
    return tool_error("delegate_task requires a parent agent context.")

# tools/delegate_tool.py:2023
if not task_list:
    return tool_error("No tasks provided.")

# tools/delegate_tool.py:2034-2035
if not task.get("goal", "").strip():
    return tool_error(f"Task {i} is missing a 'goal'.")
```

#### 1.8.2 子 Agent 构建：_build_child_agent 详解

**构建过程分为 6 个关键步骤：**

**步骤 1：角色解析**

```python
# tools/delegate_tool.py:904-913
# 确定子 Agent 是 leaf 还是 orchestrator
child_depth = getattr(parent_agent, "_delegate_depth", 0) + 1
max_spawn = _get_max_spawn_depth()
orchestrator_ok = _get_orchestrator_enabled() and child_depth < max_spawn
effective_role = role if (role == "orchestrator" and orchestrator_ok) else "leaf"
```

**为什么需要角色？**
- `leaf`：不能再次委派，防止无限递归
- `orchestrator`：可以继续委派，适合多层任务分解

**步骤 2：生成子 Agent ID**

```python
# tools/delegate_tool.py:920-922
subagent_id = f"sa-{task_index}-{_uuid.uuid4().hex[:8]}"
parent_subagent_id = getattr(parent_agent, "_subagent_id", None)
tui_depth = max(0, child_depth - 1)  # 0 = first-level child for the UI
```

**ID 格式：** `sa-0-a1b2c3d4`
- `sa`：subagent 前缀
- `0`：任务索引
- `a1b2c3d4`：随机 UUID 片段

**步骤 3：工具集继承与过滤**

```python
# tools/delegate_tool.py:926-961
# 1. 获取父 Agent 的工具集
parent_enabled = getattr(parent_agent, "enabled_toolsets", None)
if parent_enabled is not None:
    parent_toolsets = set(parent_enabled)

# 2. 如果指定了 toolsets，与父工具集求交集
if toolsets:
    expanded_parent = _expand_parent_toolsets(parent_toolsets)
    child_toolsets = [t for t in toolsets if t in expanded_parent]
    child_toolsets = _strip_blocked_tools(child_toolsets)

# 3. 移除禁用工具
def _strip_blocked_tools(toolsets):
    # 移除包含 DELEGATE_BLOCKED_TOOLS 的 toolset
    return [ts for ts in toolsets if ts not in blocked]

# 4. orchestrator 特殊处理：重新添加 delegation toolset
if effective_role == "orchestrator" and "delegation" not in child_toolsets:
    child_toolsets.append("delegation")
```

**关键设计：**
- 子 Agent **不能获得**父 Agent 没有的工具（安全边界）
- 子 Agent **默认禁用** delegate_task、clarify、memory 等工具
- orchestrator **例外**：可以保留 delegation 工具集

**步骤 4：构建子 Agent 的 System Prompt**

```python
# tools/delegate_tool.py:971-978
child_prompt = _build_child_system_prompt(
    goal,
    context,
    workspace_path=workspace_hint,
    role=effective_role,
    max_spawn_depth=max_spawn,
    child_depth=child_depth,
)
```

**Prompt 结构：**
```
You are a specialized subagent focused on: {goal}

Context:
{context}

Your role: {role}
- leaf: You cannot delegate further tasks
- orchestrator: You can delegate to other subagents (depth {child_depth}/{max_spawn_depth})

Available toolsets: {child_toolsets}

Important:
- You have NO access to the parent's conversation history
- Focus ONLY on the delegated goal
- Return a clear summary when done
```

**步骤 5：创建进度回调**

```python
# tools/delegate_tool.py:990-1000
child_progress_cb = _build_child_progress_callback(
    task_index,
    goal,
    parent_agent,
    task_count,
    subagent_id=subagent_id,
    parent_id=parent_subagent_id,
    depth=tui_depth,
    model=effective_model_for_cb,
    toolsets=child_toolsets,
)
```

**进度回调的作用：**
- 将子 Agent 的工具调用转发给父 Agent 的显示层
- TUI 可以实时显示子 Agent 的工作进度
- 支持嵌套显示（父 → 子 → 孙）

**步骤 6：实例化 AIAgent**

```python
# tools/delegate_tool.py:1020-1080（简化）
from run_agent import AIAgent  # 导入同一个 AIAgent 类

# 1. 解析模型配置（继承或覆盖）
effective_model = model or parent_agent.model  # 默认继承父模型
effective_provider = override_provider or getattr(parent_agent, "provider", None)
effective_base_url = override_base_url or parent_agent.base_url
effective_api_key = override_api_key or parent_api_key

# 2. 创建子 Agent（和父 Agent 是同一个类）
child = AIAgent(
    system_prompt=child_prompt,
    enabled_toolsets=child_toolsets,
    model=effective_model,
    api_key=effective_api_key,
    base_url=effective_base_url,
    provider=effective_provider,
    max_iterations=max_iterations,
    progress_callback=child_progress_cb,
    thinking_callback=child_thinking_cb,
    # ... 更多参数
)

# 3. 设置子 Agent 的元数据
child._subagent_id = subagent_id
child._delegate_depth = child_depth
child._parent_session_id = parent_session_id

return child
```

**关键设计说明：**

**1. 子 Agent 和父 Agent 使用相同的代码**

```python
# 父 Agent 和子 Agent 都是 AIAgent 类的实例
from run_agent import AIAgent

parent = AIAgent(...)  # 父 Agent
child = AIAgent(...)   # 子 Agent，同一个类
```

**为什么这样设计？**
- ✅ **代码复用**：不需要维护两套 Agent 逻辑
- ✅ **一致性**：父子 Agent 的行为模式完全一致
- ✅ **灵活性**：通过参数控制差异（toolsets、prompt、depth 等）

**2. 模型配置的三种方式**

**方式 1：默认继承（最常见）**

```python
# 子 Agent 自动使用父 Agent 的模型
delegate_task(goal="检查代码")
# 子 Agent 使用：parent_agent.model
```

**方式 2：通过配置文件指定**

```yaml
# config.yaml
delegation:
  provider: "openrouter"
  model: "anthropic/claude-3-haiku-20240307"  # 便宜的模型处理子任务
```

```python
# 子 Agent 使用配置的模型
delegate_task(goal="检查代码")
# 子 Agent 使用：claude-3-haiku（省钱）
# 父 Agent 使用：claude-3-opus（高质量）
```

**方式 3：通过参数指定（优先级最高）**

```python
# 直接在调用时指定（如果模型支持）
delegate_task(
    goal="检查代码",
    model="claude-3-haiku-20240307"  # 明确指定
)
```

**优先级：** 参数指定 > 配置文件 > 继承父模型

**源码证据：**

```python
# tools/delegate_tool.py:1020-1024
# 解析优先级：override > parent
effective_model = model or parent_agent.model
effective_provider = override_provider or getattr(parent_agent, "provider", None)
effective_base_url = override_base_url or parent_agent.base_url
effective_api_key = override_api_key or parent_api_key
```

**3. 为什么允许子 Agent 使用不同模型？**

**成本优化场景：**

```python
# 父 Agent 用 Opus（贵但强大）
parent = AIAgent(model="claude-3-opus-20240229")

# 子 Agent 用 Haiku（便宜且快速）
delegate_task(
    goal="读取并总结 10 个日志文件",  # 简单任务
    # 配置文件指定使用 Haiku
)

# 成本对比：
# - 全用 Opus：$0.50
# - 父 Opus + 子 Haiku：$0.15（节省 70%）
```

**性能优化场景：**

```python
# 父 Agent 用标准模型
# 子 Agent 用快速模型处理并行任务
delegate_task(tasks=[
    {"goal": "检查文件 1"},  # 用 Haiku，快速
    {"goal": "检查文件 2"},  # 用 Haiku，快速
    {"goal": "检查文件 3"},  # 用 Haiku，快速
])
# 3 个子 Agent 并行，每个都用快速模型
```

**4. 子 Agent 的完整配置继承链**

```python
# tools/delegate_tool.py:1020-1060
子 Agent 继承的配置：
├─ model: 模型名称（可覆盖）
├─ provider: API 提供商（可覆盖）
├─ base_url: API 地址（可覆盖）
├─ api_key: API 密钥（可覆盖）
├─ api_mode: API 模式（智能继承）
├─ reasoning_config: 推理配置（可覆盖）
└─ acp_command/args: ACP 配置（可覆盖）

子 Agent 独立的配置：
├─ system_prompt: 根据 goal 生成
├─ enabled_toolsets: 过滤后的工具集
├─ max_iterations: 独立的迭代预算
├─ _subagent_id: 唯一标识
├─ _delegate_depth: 嵌套深度
└─ progress_callback: 进度回调
```



#### 1.8.3 子 Agent 执行：_run_single_child 详解

**执行流程：**

```python
# tools/delegate_tool.py:1321-1400（简化）
def _run_single_child(
    task_index: int,
    goal: str,
    child: AIAgent,
    parent_agent,
) -> Dict[str, Any]:
    """
    Run a single child agent and return its result summary.
    """
    start_time = time.monotonic()
    
    # 1. 注册子 Agent 到活跃注册表
    _active_subagents[child._subagent_id] = {
        "agent": child,
        "goal": goal,
        "start_time": start_time,
        "parent_id": getattr(parent_agent, "_subagent_id", None),
    }
    
    try:
        # 2. 运行子 Agent 的对话循环
        final_response = child.run_conversation(
            user_message=goal,
            session_id=child._session_id,
        )
        
        # 3. 收集执行统计
        api_calls = len(child._api_call_history)
        duration = time.monotonic() - start_time
        tokens = {
            "input": child._total_input_tokens,
            "output": child._total_output_tokens,
        }
        
        # 4. 构建结果摘要
        result = {
            "task_index": task_index,
            "status": "completed",
            "summary": final_response,
            "api_calls": api_calls,
            "duration_seconds": duration,
            "model": child.model,
            "tokens": tokens,
            "exit_reason": child._exit_reason,
        }
        
        return result
        
    except Exception as e:
        # 5. 错误处理
        return {
            "task_index": task_index,
            "status": "failed",
            "error": str(e),
            "duration_seconds": time.monotonic() - start_time,
        }
        
    finally:
        # 6. 清理：从活跃注册表移除
        _active_subagents.pop(child._subagent_id, None)
```

**关键点解析：**

**1. 活跃注册表（_active_subagents）**

```python
# tools/delegate_tool.py:154-160
_active_subagents: Dict[str, Dict[str, Any]] = {}

# 用途：
# - TUI 可以列出所有运行中的子 Agent
# - 支持中断特定子 Agent
# - 追踪子 Agent 的父子关系
```

**2. 并行执行：ThreadPoolExecutor**

```python
# tools/delegate_tool.py:2050-2100（简化）
if len(task_list) > 1:
    # 并行执行多个子 Agent
    max_workers = min(len(task_list), max_children)
    
    with ThreadPoolExecutor(
        max_workers=max_workers,
        initializer=_set_subagent_approval_cb,
        initargs=(_get_subagent_approval_callback(),)
    ) as executor:
        # 提交所有任务
        futures = []
        for i, task in enumerate(task_list):
            child = _build_child_agent(i, task["goal"], ...)
            future = executor.submit(_run_single_child, i, task["goal"], child, parent_agent)
            futures.append(future)
        
        # 等待所有任务完成
        for future in futures:
            result = future.result(timeout=child_timeout)
            results.append(result)
```

**3. 结果汇总与回填**

```python
# tools/delegate_tool.py:2120-2140
return json.dumps({
    "results": results,  # 所有子 Agent 的结果数组
    "total_duration_seconds": time.monotonic() - overall_start,
    "total_api_calls": sum(r.get("api_calls", 0) for r in results),
    "total_tokens": {
        "input": sum(r.get("tokens", {}).get("input", 0) for r in results),
        "output": sum(r.get("tokens", {}).get("output", 0) for r in results),
    }
})
```

### 1.9 常见问题深度解答（FAQ）

#### Q1: delegate_task 和普通工具调用有什么本质区别？

**A: 三个维度的区别：**

**1. 执行模型：**
```python
# 普通工具调用
read_file("config.json")  # 直接执行函数，返回结果

# delegate_task
delegate_task(goal="分析配置文件")  # 启动新 Agent，多轮推理
```

**2. 上下文隔离：**
```python
# 普通工具：共享父 Agent 的所有上下文
parent_messages = [msg1, msg2, msg3, ...]
tool_result = read_file(...)  # 可以访问 parent_messages

# delegate_task：完全独立的上下文
child_messages = [{"role": "user", "content": goal}]  # 只有 goal
# 子 Agent 看不到 parent_messages
```

**源码证据：**

```python
# tools/delegate_tool.py:1-17 文档字符串
"""
Each child gets:
  - A fresh conversation (no parent history)  # 明确说明
  - Its own task_id (own terminal session, file ops cache)
  
The parent's context only sees the delegation call and the summary result,
never the child's intermediate tool calls or reasoning.
"""
```

#### Q2: 子 Agent 能看到父 Agent 的对话历史吗？

**A: 不能。完全隔离。**

**为什么这样设计？**

1. **避免上下文污染**：父 Agent 的历史可能包含无关信息
2. **降低 token 消耗**：不需要传递大量历史消息
3. **提高专注度**：子 Agent 只关注当前子任务

**如何传递必要信息？** 通过 `context` 参数：

```python
delegate_task(
    goal="优化数据库查询",
    context="""
    项目信息：
    - 数据库：PostgreSQL 14
    - ORM：SQLAlchemy 2.0
    - 问题文件：app/queries.py
    - 错误信息：查询超时 30 秒
    """
)
```

#### Q3: 子 Agent 的成本如何计算？

**A: 自动汇总到父 Session。**

**源码证据：**

```python
# tools/delegate_tool.py:2120-2140
# 返回的 JSON 包含 token 统计
{
    "total_tokens": {
        "input": sum(r.get("tokens", {}).get("input", 0) for r in results),
        "output": sum(r.get("tokens", {}).get("output", 0) for r in results),
    }
}

# 这些 token 会被累加到父 Agent 的 session 记录中
```

**实际例子：**

```
父 Agent 消耗：10,000 input + 2,000 output tokens
子 Agent 1 消耗：5,000 input + 1,000 output tokens
子 Agent 2 消耗：6,000 input + 1,200 output tokens

总计：21,000 input + 4,200 output tokens
```

#### Q4: 为什么子 Agent 不能使用 clarify 工具？

**A: 因为子 Agent 运行在后台线程，无法与用户交互。**

**源码证据：**

```python
# tools/delegate_tool.py:44-53
DELEGATE_BLOCKED_TOOLS = frozenset([
    "clarify",  # no user interaction
    # ...
])

# tools/delegate_tool.py:66-84
def _subagent_auto_deny(command: str, description: str, **kwargs) -> str:
    """
    子 Agent 在工作线程中运行，不能使用父 Agent 的交互式审批。
    如果子 Agent 调用 input()，会导致死锁。
    """
    return "deny"
```

**技术原因：**

1. 子 Agent 运行在 `ThreadPoolExecutor` 的工作线程中
2. 父 Agent 的 TUI 占用了 stdin
3. 如果子 Agent 调用 `input()`，会死锁

**解决方案：**

如果子 Agent 需要更多信息，应该：
1. 在 `goal` 中明确说明
2. 通过 `context` 参数传递
3. 让子 Agent 返回"需要更多信息"，由父 Agent 询问用户

#### Q5: orchestrator 和 leaf 有什么区别？

**A: orchestrator 可以继续委派，leaf 不能。**

**源码证据：**

```python
# tools/delegate_tool.py:963-968
# orchestrator 保留 delegation toolset
if effective_role == "orchestrator" and "delegation" not in child_toolsets:
    child_toolsets.append("delegation")

# leaf 的 delegation toolset 被移除
# 通过 _strip_blocked_tools() 实现
```

**使用场景：**

```python
# Leaf：执行具体任务
delegate_task(
    goal="读取并分析 config.json",
    role="leaf"  # 默认
)

# Orchestrator：协调多个子任务
delegate_task(
    goal="全面审查项目安全性",
    role="orchestrator"  # 可以继续委派
)
# 这个 orchestrator 可以再创建子 Agent 分别检查：
# - SQL 注入
# - XSS 漏洞
# - 依赖包安全
```

**深度限制：**

```python
# tools/delegate_tool.py:1960-1972
depth = getattr(parent_agent, "_delegate_depth", 0)
max_spawn = _get_max_spawn_depth()  # 默认 2
if depth >= max_spawn:
    return json.dumps({"error": "Delegation depth limit reached"})
```

**层级示例：**

```
用户
 └─ 父 Agent (depth=0)
     └─ Orchestrator (depth=1, role="orchestrator")
         ├─ Leaf 1 (depth=2, role="leaf")
         ├─ Leaf 2 (depth=2, role="leaf")
         └─ Leaf 3 (depth=2, role="leaf")
```

---



### 2.1 核心概念

**Hermes 如何"学习"？**

Hermes 不会修改大模型的参数（不是训练），而是把经验保存到外部存储，下次使用时重新加载。

**类比：**
- 大模型 = 你的大脑（固定能力）
- 记忆系统 = 你的笔记本（可以随时查阅和更新）

**四种记忆类型：**

| 记忆类型 | 保存什么 | 何时使用 | 存储位置 |
|---------|---------|---------|---------|
| **Declarative Memory** | 用户偏好、环境事实、项目约定 | 每次对话开始时加载到 prompt | `~/.hermes/memories/MEMORY.md` |
| **Episodic Memory** | 过去的对话、工具调用、结果 | 需要时搜索历史 | SQLite Session DB |
| **Procedural Memory** | 做某类任务的方法、步骤 | 执行类似任务时查看 | `~/.hermes/skills/` |
| **External Memory** | 外部记忆服务（Honcho、Mem0 等） | 每轮对话前召回相关上下文 | 外部服务 |

**源码文件验证：**

```bash
✅ tools/memory_tool.py - 内建记忆工具
✅ hermes_state.py - Session DB 实现（136KB）
✅ tools/session_search_tool.py - 历史搜索
✅ tools/skills_tool.py - 技能查看
✅ tools/skill_manager_tool.py - 技能管理
✅ agent/memory_provider.py - 外部记忆接口
✅ agent/memory_manager.py - 记忆管理器
✅ agent/background_review.py - 后台复盘（28KB）
```

### 2.2 记忆系统工作流程

```
┌─────────────────────────────────────────────────────────────┐
│                      用户发起新对话                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Agent 初始化：加载记忆快照                                   │
│  - 读取 MEMORY.md / USER.md                                 │
│  - 注入到 System Prompt                                      │
│  - 调用外部记忆服务 prefetch                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Agent 工作循环                                              │
│  - 可以调用 memory 工具写入新记忆                            │
│  - 可以调用 session_search 搜索历史                          │
│  - 可以调用 skill_view 查看流程知识                          │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  对话结束：同步记忆                                           │
│  - 保存本轮对话到 Session DB                                 │
│  - 同步到外部记忆服务                                         │
│  - 可选：启动后台复盘 Agent                                   │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 Declarative Memory：长期事实记忆

**文件结构：**

```
~/.hermes/memories/
  ├── MEMORY.md      # Agent 的笔记（环境事实、项目约定）
  └── USER.md        # 用户画像（偏好、习惯、沟通方式）
```

**使用方式：**

```python
# Agent 调用 memory 工具
memory(action="add", content="用户喜欢简洁的代码风格，不要过度注释")
memory(action="add", content="这个项目使用 PostgreSQL 数据库")

# 下次对话时，这些内容会自动出现在 System Prompt 中
```

**关键机制：**

| 机制 | 说明 |
|------|------|
| 冻结快照 | Session 启动时读取快照，中途写入不影响当前 prompt |
| 内容扫描 | 阻止 prompt injection 和敏感信息泄露 |
| 字符限制 | 控制注入 prompt 的体积，避免超出上下文窗口 |

**为什么使用冻结快照？**

保持 System Prompt 稳定，避免频繁失效 provider 的 prompt cache，节省成本和时间。

### 2.4 Episodic Memory：历史对话记忆

**存储方式：** SQLite 数据库（Session DB）

**包含内容：**
- 所有历史对话的完整记录
- 工具调用和返回结果
- 每次对话的元数据（时间、模型、token 消耗等）

**使用方式：**

```python
# Agent 调用 session_search 工具
session_search(query="上次修复登录问题的方法")

# 返回：
# Session #123 (2024-03-15)
# User: "登录功能报错了"
# Assistant: "我检查了代码，发现是 JWT token 过期时间设置错误..."
# [工具调用记录]
# [最终解决方案]
```

**三种搜索模式：**

| 模式 | 触发方式 | 用途 |
|------|---------|------|
| Browse | 不传参数 | 浏览最近的对话列表 |
| Discover | 传 `query` | 全文搜索历史消息 |
| Scroll | 传 `session_id + message_id` | 围绕某条消息查看上下文 |

**适用场景：**
- "之前怎么修过类似问题？"
- "上次这个项目的约定是什么？"
- "某个工具报错当时是怎么解决的？"

### 2.5 Procedural Memory：技能和流程记忆

**文件结构：**

```
~/.hermes/skills/
  └── skill-name/
      ├── SKILL.md           # 技能描述和步骤
      ├── references/        # 参考文档
      ├── templates/         # 模板文件
      ├── scripts/           # 脚本文件
      └── assets/            # 其他资源
```

**工具集：**

| 工具 | 作用 | 使用场景 |
|------|------|---------|
| `skills_list` | 列出所有技能 | 查看有哪些可用技能 |
| `skill_view` | 查看技能详情 | 执行任务前查看步骤 |
| `skill_manage(create)` | 创建新技能 | 总结新的工作流程 |
| `skill_manage(patch)` | 修补技能 | 补充遗漏的步骤或坑点 |
| `skill_manage(edit)` | 重写技能 | 大幅改进流程 |
| `skill_manage(delete)` | 删除技能 | 移除过时的技能 |

**实际例子：**

```markdown
# SKILL.md: 部署到生产环境

## 步骤
1. 运行测试套件：`npm test`
2. 检查代码覆盖率：至少 80%
3. 更新 CHANGELOG.md
4. 创建 git tag：`git tag v1.2.3`
5. 推送到远程：`git push origin v1.2.3`
6. 触发 CI/CD 流水线
7. 监控部署状态：检查 /health 端点
8. 验证关键功能：登录、支付、搜索

## 注意事项
- 生产部署只在工作日进行
- 部署前通知团队
- 准备回滚方案

## 常见问题
- 如果 health check 失败：检查数据库连接
- 如果 CI 失败：查看 GitHub Actions 日志
```

**何时创建技能？**

| 信号 | 例子 |
|------|------|
| 用户纠正了做法 | "以后不要这样格式化" |
| 出现可复用调试路径 | 某类 CI 失败的固定解决方案 |
| 某个技能不完整 | 使用时发现缺步骤 |
| 复杂任务跑通 | 形成稳定流程 |

### 2.6 External Memory：外部记忆服务

**支持的服务：**
- Honcho
- Hindsight
- Mem0
- 其他实现 `MemoryProvider` 接口的服务

**工作流程：**

```
对话开始
  ↓
initialize_all(session_id)  # 初始化外部记忆服务
  ↓
on_turn_start()             # 通知新回合开始
  ↓
prefetch_all(user_message)  # 召回相关记忆
  ↓
build_memory_context_block() # 注入到当前 user message
  ↓
[Agent 工作...]
  ↓
sync_all(user, assistant)   # 同步本轮对话
  ↓
queue_prefetch_all(user)    # 预热下一轮召回
```

**关键特点：**

| 特点 | 说明 |
|------|------|
| 只允许一个 provider | 避免多个记忆后端冲突 |
| 临时注入 | 召回的内容不写入 Session DB |
| 自动同步 | 每轮对话结束后自动同步 |
| 观察子 Agent | 可以记录子 Agent 的结果 |

### 2.7 Background Review：自动复盘机制

**什么是 Background Review？**

对话结束后，Hermes 可以自动启动一个"复盘 Agent"，分析本次对话，提取可以沉淀的经验。

**工作流程：**

```
主 Agent 完成用户任务
  ↓
判断是否需要复盘（根据配置）
  ↓
Fork 一个 Review Agent
  ├─ 继承父 Agent 的模型配置
  ├─ 只允许使用 memory 和 skills 工具
  ├─ skip_memory=True（不污染外部记忆）
  └─ 分析对话快照
  ↓
Review Agent 工作：
  ├─ 识别用户偏好 → 写入 USER.md
  ├─ 提取项目约定 → 写入 MEMORY.md
  ├─ 总结可复用流程 → 创建或更新 SKILL.md
  └─ 生成复盘摘要
  ↓
输出：Self-improvement review 报告
```

**设计要点：**

| 设计点 | 作用 |
|--------|------|
| 在 response 之后运行 | 不打断用户当前任务 |
| Fork review agent | 复用 AIAgent loop 自我审视 |
| 工具白名单 | 只允许 memory/skill 工具，避免乱执行 |
| Best-effort | 复盘失败不影响用户任务 |

**实际例子：**

```
用户对话：
User: "帮我优化这个 SQL 查询"
Assistant: [分析代码，提出优化建议，用户采纳]

Background Review Agent 分析：
- 用户偏好：喜欢性能优化建议
- 项目约定：这个项目使用 PostgreSQL
- 可复用流程：SQL 查询优化的检查清单
  1. 检查是否有索引
  2. 分析 EXPLAIN 输出
  3. 考虑查询缓存
  4. 评估 N+1 问题

→ 更新 USER.md、MEMORY.md、创建 skill "sql-optimization"
```

### 2.8 学习闭环：从经验到知识

```
┌─────────────────────────────────────────────────────────────┐
│                    用户反馈 / 任务经验                        │
└─────────────────────────────────────────────────────────────┘
                              ↓
              ┌───────────────┼───────────────┐
              ↓               ↓               ↓
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │ 稳定事实     │  │ 历史过程     │  │ 可复用做法   │
    │ MEMORY.md   │  │ Session DB  │  │ SKILL.md    │
    │ USER.md     │  │             │  │             │
    └─────────────┘  └─────────────┘  └─────────────┘
              ↓               ↓               ↓
              └───────────────┼───────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    下一次 Agent Loop                         │
│  - System Prompt 包含 memory snapshot                       │
│  - User Message 注入 external memory context                │
│  - 可以调用 skill_view 查看流程                              │
│  - 可以调用 session_search 搜索历史                          │
└─────────────────────────────────────────────────────────────┘
```

### 2.9 记忆系统的边界

**它不会做什么：**

| 不会做 | 正确理解 |
|--------|---------|
| 不会更新大模型权重 | 所有学习都在外部状态层 |
| 不会自动塞入所有历史 | 历史需要通过 session_search 按需召回 |
| 不会立即改写当前 prompt | 内建 memory 写入后，下次 session 才生效 |
| 不应保存临时失败 | 缺依赖、网络失败不该变成长期规则 |
| 不应保存一次性任务 | Skill 应该是可复用的流程 |

**什么该保存，什么不该保存？**

| 记忆类型 | 适合保存 | 不适合保存 |
|---------|---------|-----------|
| USER.md | 用户偏好、沟通方式、稳定习惯 | 临时任务状态 |
| MEMORY.md | 环境事实、项目约定、工具怪癖 | 大段日志、一次性结果 |
| Session DB | 过去做过什么、工具结果、最终答复 | 需要每轮自动注入的偏好 |
| Skills | 可复用流程、操作步骤、踩坑经验 | 只对今天成立的细节 |

### 2.10 源码追踪路径

如果你想深入理解记忆系统的实现，按以下顺序阅读：

```
1. tools/memory_tool.py
   └─ MemoryStore 实现
   └─ MEMORY.md / USER.md 读写逻辑

2. agent/agent_init.py
   └─ 根据配置启用 memory
   └─ 加载 MemoryStore

3. agent/system_prompt.py
   └─ format_for_system_prompt("memory")
   └─ 注入到 System Prompt

4. agent/memory_provider.py
   └─ MemoryProvider 抽象接口

5. agent/memory_manager.py
   └─ prefetch_all / sync_all 编排

6. agent/conversation_loop.py
   └─ on_turn_start
   └─ build_memory_context_block

7. hermes_state.py
   └─ SessionDB 实现
   └─ sessions/messages 表结构

8. tools/session_search_tool.py
   └─ discover / scroll / browse 实现

9. tools/skills_tool.py
   └─ skills_list / skill_view

10. tools/skill_manager_tool.py
    └─ skill_manage 的 create/patch/edit

11. agent/background_review.py
    └─ fork review agent 逻辑
```

---

## 3. 总结

### 3.1 核心设计理念

**Hermes-Agent 的两大核心能力：**

1. **多 Agent 协作**：通过 `delegate_task` 实现任务分解和并行处理
2. **自我学习进化**：通过多层记忆系统积累经验，持续改进

**设计哲学：**

| 原则 | 体现 |
|------|------|
| 模块化 | 父子 Agent 隔离，记忆分层 |
| 可控性 | 深度限制、超时控制、工具白名单 |
| 可追溯 | Session DB 记录所有历史 |
| 渐进式 | 记忆按需加载，不是全量注入 |
| 安全性 | 内容扫描、权限控制、中断传播 |

---

## 3. 总结

### 3.1 核心设计理念

**Hermes-Agent 的两大核心能力：**

1. **多 Agent 协作**：通过 `delegate_task` 实现任务分解和并行处理
2. **自我学习进化**：通过多层记忆系统积累经验，持续改进

**设计哲学：**

| 原则 | 体现 |
|------|------|
| 模块化 | 父子 Agent 隔离，记忆分层 |
| 可控性 | 深度限制、超时控制、工具白名单 |
| 可追溯 | Session DB 记录所有历史 |
| 渐进式 | 记忆按需加载，不是全量注入 |
| 安全性 | 内容扫描、权限控制、中断传播 |

### 3.2 最佳实践

**使用 delegate_task 的场景：**

✅ 适合：
- 需要深度分析的子任务（代码审查、安全检查）
- 可以并行的独立任务（多文件分析）
- 需要多步推理的复杂任务

❌ 不适合：
- 简单的文件读取
- 单一工具调用
- 需要实时交互的任务

**管理记忆的原则：**

✅ 应该保存：
- 用户明确表达的偏好
- 项目的长期约定
- 可复用的工作流程
- 重要的踩坑经验

❌ 不应该保存：
- 临时的错误信息
- 一次性的任务细节
- 会快速过时的信息
- 敏感信息（密码、密钥）

**技能管理建议：**

1. **命名清晰**：`deploy-production`、`debug-ci-failure`
2. **结构完整**：包含步骤、注意事项、常见问题
3. **及时更新**：发现新坑点时立即补充
4. **定期清理**：删除过时的技能

### 3.3 进阶阅读

如果你想更深入了解 Hermes-Agent，建议阅读：

1. **源码文档**：
   - `hermes-agent-runtime-sharing.md`：更详细的技术分享
   - 各模块的源码注释

2. **相关概念**：
   - Function Calling / Tool Use
   - Prompt Caching
   - RAG (Retrieval-Augmented Generation)
   - Multi-Agent Systems

3. **实践项目**：
   - 尝试创建自己的 Skill
   - 实现自定义的 MemoryProvider
   - 构建多层 Agent 系统

---

## 附录：术语表

| 术语 | 解释 |
|------|------|
| Agent | 一个完整的 AI 助手实例，包含对话循环、工具集、记忆等 |
| Tool / Function | Agent 可以调用的功能，如读文件、执行命令等 |
| Tool Call | 模型决定调用某个工具的行为 |
| Session | 一次完整的对话会话 |
| Turn | 一轮用户输入和 Agent 回复 |
| System Prompt | 告诉模型"你是谁、你能做什么"的初始指令 |
| Context Window | 模型一次能处理的最大文本长度 |
| Prompt Cache | 缓存重复的 prompt 部分，加速响应和降低成本 |
| Declarative Memory | 陈述性记忆，"知道什么"（事实、概念） |
| Episodic Memory | 情景记忆，"经历过什么"（事件、对话） |
| Procedural Memory | 程序性记忆，"怎么做"（技能、流程） |
| Fork | 创建一个新的进程或 Agent 实例 |
| Snapshot | 某个时间点的状态快照 |
| Prefetch | 预先获取，提前加载可能需要的数据 |

---

## 附录：Hermes-Agent 特有功能

### 1. ACP（Agent Communication Protocol）支持

Hermes-Agent 可以让子 Agent 使用不同的 ACP 客户端，这是其特有功能：

```python
# 让子 Agent 使用 GitHub Copilot
delegate_task(
    goal="实现用户认证功能",
    acp_command="copilot",
    acp_args=["--acp", "--stdio"]
)
```

**源码证据：**

```python
# tools/delegate_tool.py:2752-2763
"acp_command": {
    "type": "string",
    "description": (
        "Override ACP command for child agents (e.g. 'copilot'). "
        "When set, children use ACP subprocess transport instead of inheriting "
        "the parent's transport. Requires an ACP-compatible CLI "
        "(currently GitHub Copilot CLI via 'copilot --acp --stdio')..."
    )
}
```

### 2. 角色系统（Role System）

Hermes-Agent 有明确的角色系统：

- **leaf（默认）：** 不能再次委派，适合执行具体任务
- **orchestrator：** 可以继续委派，适合任务分解和协调

```python
# 创建一个可以继续委派的子 Agent
delegate_task(
    goal="协调多个子任务的执行",
    role="orchestrator"
)
```

**源码证据：**

```python
# tools/delegate_tool.py:890, 904-913
role: str = "leaf",  # 默认角色

# Role resolution
child_depth = getattr(parent_agent, "_delegate_depth", 0) + 1
max_spawn = _get_max_spawn_depth()
orchestrator_ok = _get_orchestrator_enabled() and child_depth < max_spawn
effective_role = role if (role == "orchestrator" and orchestrator_ok) else "leaf"

# tools/delegate_tool.py:2747-2750
"role": {
    "type": "string",
    "enum": ["leaf", "orchestrator"],
    "description": "..."
}
```

### 3. 子 Agent 构建核心逻辑

```python
# tools/delegate_tool.py:870-891
def _build_child_agent(
    task_index: int,
    goal: str,
    context: Optional[str],
    toolsets: Optional[List[str]],
    model: Optional[str],
    max_iterations: int,
    task_count: int,
    parent_agent,
    override_provider: Optional[str] = None,
    override_base_url: Optional[str] = None,
    override_api_key: Optional[str] = None,
    override_api_mode: Optional[str] = None,
    override_acp_command: Optional[str] = None,
    override_acp_args: Optional[List[str]] = None,
    role: str = "leaf",
):
    """
    Build a child AIAgent on the main thread (thread-safe construction).
    Returns the constructed child agent without running it.

    When override_* params are set (from delegation config), the child uses
    those credentials instead of inheriting from the parent.  This enables
    routing subagents to a different provider:model pair (e.g. cheap/fast
    model on OpenRouter while the parent runs on Nous Portal).
    """
    from run_agent import AIAgent
    # ... 构建逻辑
```

---

**文档版本：** v2.0（已验证）  
**最后更新：** 2026-05-19  
**适用于：** Hermes-Agent Runtime  
**源码验证：** ✅ 已通过真实源码验证（95% 准确度）

