# Hermes Agent - delegate_task 触发机制详解

> 基于源码：`/Users/wangzhongbin/Documents/code/fangzhen/hermes-agent`

## 核心结论

**`delegate_task` 的触发完全由模型（LLM）自主决定，系统不会自动触发。**

---

## 1. 触发机制本质

### 1.1 工具调用模式

```python
# tools/delegate_tool.py
registry.register(
    name="delegate_task",
    toolset="delegation",
    schema=DELEGATE_TASK_SCHEMA,
    handler=lambda args, **kw: delegate_task(...),
)
```

**关键点**：
- `delegate_task` 是一个**标准工具（Tool）**
- 通过 OpenAI Function Calling 机制暴露给模型
- 模型看到工具的 schema（包括 description）
- **模型自主决定**是否调用这个工具

### 1.2 不是自动触发

❌ **系统不会**：
- 根据任务复杂度自动触发
- 分析用户请求后自动委托
- 在后台自动创建子 Agent

✅ **模型会**：
- 读取工具的 description
- 根据 description 中的指导决定是否调用
- 自主构造 function call 参数

---

## 2. 模型如何看到 delegate_task

### 2.1 工具定义流程

```python
# model_tools.py
def get_tool_definitions(
    enabled_toolsets: List[str] = None,
    disabled_toolsets: List[str] = None,
    quiet_mode: bool = False,
) -> List[Dict[str, Any]]:
    """Get tool definitions for model API calls with toolset-based filtering."""
    # 从 registry 获取所有工具
    # 根据 enabled_toolsets / disabled_toolsets 过滤
    # 返回 OpenAI 格式的工具定义列表
```

**流程**：
1. `AIAgent` 初始化时调用 `get_tool_definitions()`
2. 根据 `enabled_toolsets` 过滤工具
3. 如果 `"delegation"` 在 enabled_toolsets 中，`delegate_task` 被包含
4. 工具定义（包括 description）发送给模型

### 2.2 工具 Schema 结构

```python
# 模型看到的 JSON Schema
{
  "type": "function",
  "function": {
    "name": "delegate_task",
    "description": "Spawn one or more subagents in isolated contexts...",
    "parameters": {
      "type": "object",
      "properties": {
        "goal": {
          "type": "string",
          "description": "What the subagent should accomplish..."
        },
        "context": {...},
        "toolsets": {...},
        "tasks": {...},
        "role": {...}
      }
    }
  }
}
```

---

## 3. 提示词指导（Description）

### 3.1 动态生成的 Description

```python
# tools/delegate_tool.py
def _build_top_level_description() -> str:
    """Compose the delegate_task tool description with current runtime limits."""
    max_children = _get_max_concurrent_children()
    max_depth = _get_max_spawn_depth()
    orchestrator_on = _get_orchestrator_enabled()
    
    return (
        "Spawn one or more subagents to work on tasks in isolated contexts. "
        "Each subagent gets its own conversation, terminal session, and toolset. "
        "Only the final summary is returned -- intermediate tool results "
        "never enter your context window.\n\n"
        "WHEN TO USE delegate_task:\n"
        "- Reasoning-heavy subtasks (debugging, code review, research synthesis)\n"
        "- Tasks that would flood your context with intermediate data\n"
        "- Parallel independent workstreams (research A and B simultaneously)\n\n"
        "WHEN NOT TO USE (use these instead):\n"
        "- Mechanical multi-step work with no reasoning needed -> use execute_code\n"
        "- Single tool call -> just call the tool directly\n"
        "- Tasks needing user interaction -> subagents cannot use clarify\n"
        # ... 更多指导
    )
```

**关键特性**：
- Description 在**每次** `get_tool_definitions()` 调用时动态生成
- 包含当前配置的实际限制（max_concurrent_children, max_spawn_depth）
- 提供明确的使用场景和反模式

### 3.2 完整的 Description 内容

```python
# 源码：tools/delegate_tool.py:2534-2584
WHEN TO USE delegate_task:
- Reasoning-heavy subtasks (debugging, code review, research synthesis)
- Tasks that would flood your context with intermediate data
- Parallel independent workstreams (research A and B simultaneously)

WHEN NOT TO USE (use these instead):
- Mechanical multi-step work with no reasoning needed -> use execute_code
- Single tool call -> just call the tool directly
- Tasks needing user interaction -> subagents cannot use clarify
- Durable long-running work that must outlive the current turn -> 
  use cronjob (action='create') or terminal(background=True, 
  notify_on_complete=True) instead. delegate_task runs SYNCHRONOUSLY 
  inside the parent turn: if the parent is interrupted (user sends a 
  new message, /stop, /new) the child is cancelled with status=
  'interrupted' and its work is discarded. Children cannot continue 
  in the background.

IMPORTANT:
- Subagents have NO memory of your conversation. Pass all relevant 
  info (file paths, error messages, constraints) via the 'context' field.
- If the user is writing in a non-English language, or asked for 
  output in a specific language / tone / style, say so in 'context' 
  (e.g. "respond in Chinese", "return output in Japanese"). 
  Otherwise subagents default to English and their summaries will 
  contaminate your final reply with the wrong language.
- Subagent summaries are SELF-REPORTS, not verified facts. A subagent 
  that claims "uploaded successfully" or "file written" may be wrong. 
  For operations with external side-effects (HTTP POST/PUT, remote 
  writes, file creation at shared paths, publishing), require the 
  subagent to return a verifiable handle (URL, ID, absolute path, HTTP 
  status) and verify it yourself — fetch the URL, stat the file, read 
  back the content — before telling the user the operation succeeded.
- Leaf subagents (role='leaf', the default) CANNOT call: 
  delegate_task, clarify, memory, send_message, execute_code.
- Orchestrator subagents (role='orchestrator') retain 
  delegate_task so they can spawn their own workers, but still 
  cannot use clarify, memory, send_message, or execute_code.
```

---

## 4. 模型决策过程

### 4.1 模型的推理链

```
用户请求
    ↓
模型分析任务
    ↓
查看可用工具列表
    ↓
读取 delegate_task 的 description
    ↓
判断：是否符合 "WHEN TO USE" 的场景？
    ↓
    ├─ 是 → 构造 function_call
    │         {
    │           "name": "delegate_task",
    │           "arguments": {
    │             "goal": "...",
    │             "context": "...",
    │             "toolsets": [...]
    │           }
    │         }
    │
    └─ 否 → 使用其他工具或直接回复
```

### 4.2 示例：模型的决策

**场景 1：推理密集型任务**
```
用户：Debug this authentication bug and find the root cause

模型思考：
- 这是推理密集型任务（debugging）
- 会产生大量中间数据（日志、堆栈跟踪）
- 符合 "WHEN TO USE" 的第一条
→ 决定调用 delegate_task

模型输出：
{
  "name": "delegate_task",
  "arguments": {
    "goal": "Debug the authentication bug and identify root cause",
    "context": "Error: 401 Unauthorized in login.py:45\nStack trace: ...",
    "toolsets": ["terminal", "file"]
  }
}
```

**场景 2：简单文件读取**
```
用户：Read the contents of config.yaml

模型思考：
- 这是单个工具调用
- 符合 "WHEN NOT TO USE" 的第二条
→ 决定直接调用 read_file

模型输出：
{
  "name": "read_file",
  "arguments": {
    "file_path": "config.yaml"
  }
}
```

---

## 5. 系统提示中的工具指导

### 5.1 系统提示结构

```python
# agent/system_prompt.py
def build_system_prompt_parts(agent, system_message=None) -> Dict[str, str]:
    """Assemble the system prompt as three ordered parts."""
    return {
        "stable": "...",    # 身份、工具指导、技能提示
        "context": "...",   # 上下文文件、用户消息
        "volatile": "..."   # 记忆快照、时间戳
    }
```

### 5.2 工具相关的指导

```python
# agent/prompt_builder.py

# 记忆工具指导
MEMORY_GUIDANCE = (
    "You have persistent memory across sessions. Save durable facts using the memory "
    "tool: user preferences, environment details, tool quirks, and stable conventions..."
)

# 技能工具指导
SKILLS_GUIDANCE = (
    "After completing a complex task (5+ tool calls), fixing a tricky error, "
    "or discovering a non-trivial workflow, save the approach as a "
    "skill with skill_manage so you can reuse it next time..."
)

# 会话搜索指导
SESSION_SEARCH_GUIDANCE = (
    "When the user references something from a past conversation or you suspect "
    "relevant cross-session context exists, use session_search to recall it..."
)
```

**关键点**：
- 系统提示中**没有**专门针对 `delegate_task` 的指导
- 工具指导只针对 `memory`, `skill_manage`, `session_search` 等
- `delegate_task` 的所有指导都在**工具 schema 的 description** 中

### 5.3 工具可见性检查

```python
# agent/system_prompt.py:105-122
tool_guidance = []
if "memory" in agent.valid_tool_names:
    tool_guidance.append(MEMORY_GUIDANCE)
if "session_search" in agent.valid_tool_names:
    tool_guidance.append(SESSION_SEARCH_GUIDANCE)
if "skill_manage" in agent.valid_tool_names:
    tool_guidance.append(SKILLS_GUIDANCE)
```

**注意**：
- 系统提示中的工具指导是**可选的**
- 只有特定工具（memory, session_search, skill_manage）有专门指导
- `delegate_task` **不在**这个列表中

---

## 6. 配置对触发的影响

### 6.1 工具集启用

```yaml
# config.yaml
agent:
  enabled_toolsets:
    - terminal
    - file
    - web
    - delegation  # ← 必须包含才能使用 delegate_task
```

**影响**：
- 如果 `delegation` 不在 `enabled_toolsets` 中
- `delegate_task` 不会出现在工具列表中
- 模型无法调用（即使想调用也不行）

### 6.2 工具集禁用

```yaml
# config.yaml
agent:
  disabled_toolsets:
    - delegation  # ← 明确禁用
```

**影响**：
- `delegate_task` 被从工具列表中移除
- 模型看不到这个工具

### 6.3 子 Agent 的工具集

```python
# tools/delegate_tool.py
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # 子 Agent 默认不能递归委托
    "clarify",
    "memory",
    "send_message",
    "execute_code",
])

def _strip_blocked_tools(toolsets: List[str]) -> List[str]:
    """Remove toolsets that contain only blocked tools."""
    blocked_toolset_names = {
        "delegation",      # 除非是 orchestrator
        "clarify",
        "memory",
        "code_execution",
    }
    return [t for t in toolsets if t not in blocked_toolset_names]
```

**影响**：
- 子 Agent（role='leaf'）看不到 `delegate_task`
- Orchestrator（role='orchestrator'）可以看到并调用

---

## 7. 实际触发示例

### 7.1 用户请求 → 模型调用

**用户输入**：
```
Research three different approaches to implement authentication:
1. JWT tokens
2. Session-based auth
3. OAuth 2.0

Compare their pros and cons.
```

**模型推理**：
```
这个任务：
- 需要研究 3 个独立主题
- 可以并行执行
- 每个研究会产生大量中间数据
- 符合 "Parallel independent workstreams"

→ 决定使用 delegate_task 批量模式
```

**模型输出**：
```json
{
  "name": "delegate_task",
  "arguments": {
    "tasks": [
      {
        "goal": "Research JWT token authentication",
        "context": "Focus on security, implementation complexity, and scalability",
        "toolsets": ["web"]
      },
      {
        "goal": "Research session-based authentication",
        "context": "Focus on security, implementation complexity, and scalability",
        "toolsets": ["web"]
      },
      {
        "goal": "Research OAuth 2.0 authentication",
        "context": "Focus on security, implementation complexity, and scalability",
        "toolsets": ["web"]
      }
    ]
  }
}
```

### 7.2 系统执行流程

```
1. 模型生成 function_call
   ↓
2. run_agent.py 接收到 tool_calls
   ↓
3. 调用 handle_function_call("delegate_task", arguments)
   ↓
4. delegate_tool.py 的 delegate_task() 函数执行
   ↓
5. 创建 3 个子 Agent 并行执行
   ↓
6. 等待所有子 Agent 完成
   ↓
7. 返回结果给模型
   ↓
8. 模型综合结果回复用户
```

---

## 8. 不会自动触发的证据

### 8.1 源码证据 1：工具注册

```python
# tools/delegate_tool.py:2783-2801
registry.register(
    name="delegate_task",
    toolset="delegation",
    schema=DELEGATE_TASK_SCHEMA,
    handler=lambda args, **kw: delegate_task(...),
    check_fn=check_delegate_requirements,
    emoji="🔀",
    dynamic_schema_overrides=_build_dynamic_schema_overrides,
)
```

**分析**：
- 这是标准的工具注册
- 没有任何自动触发逻辑
- 只有 `handler` 函数，等待被调用

### 8.2 源码证据 2：工具调用入口

```python
# model_tools.py
def handle_function_call(
    function_name: str,
    function_args: dict,
    task_id: str = None,
    user_task: str = None,
    parent_agent=None,
) -> str:
    """Handle a function call from the model."""
    # 从 registry 查找工具
    entry = registry.get_entry(function_name)
    if not entry:
        return f"Error: Unknown function {function_name}"
    
    # 调用工具的 handler
    return entry.handler(function_args, task_id=task_id, parent_agent=parent_agent)
```

**分析**：
- 工具调用完全由模型的 `function_call` 触发
- 系统只是被动响应
- 没有任何主动分析和触发的逻辑

### 8.3 源码证据 3：对话循环

```python
# agent/conversation_loop.py
def run_conversation(agent, user_message, ...):
    while True:
        # 调用模型 API
        response = client.chat.completions.create(
            model=agent.model,
            messages=messages,
            tools=agent.tools,  # ← 工具列表传给模型
        )
        
        # 检查模型是否调用了工具
        if response.tool_calls:
            for tool_call in response.tool_calls:
                # 执行模型选择的工具
                result = handle_function_call(
                    tool_call.function.name,
                    tool_call.function.arguments,
                )
        else:
            # 模型没有调用工具，直接返回
            break
```

**分析**：
- 系统只是把工具列表传给模型
- 模型决定是否调用工具
- 系统被动执行模型的决策

---

## 9. 模型能力的影响

### 9.1 不同模型的表现

**强模型（Claude Opus, GPT-4）**：
- 能准确理解 description 中的指导
- 正确判断何时使用 delegate_task
- 合理构造 goal 和 context 参数

**弱模型（GPT-3.5, 小型模型）**：
- 可能误解使用场景
- 过度使用或不使用 delegate_task
- 参数构造不够精确

### 9.2 提示词工程的影响

**用户可以通过提示词影响模型决策**：

```
# 鼓励使用
用户：Please use subagents to parallelize this research task.

# 禁止使用
用户：Do this yourself, don't delegate to subagents.

# 明确指导
用户：Break this into 3 parallel research tasks using delegate_task.
```

---

## 10. 总结

### 10.1 触发机制

| 维度 | 说明 |
|-----|------|
| **触发方式** | 模型自主决策，通过 function calling |
| **决策依据** | 工具 schema 的 description |
| **系统角色** | 被动响应，不主动触发 |
| **配置影响** | 只影响工具可见性，不影响触发逻辑 |
| **用户影响** | 可通过提示词引导模型决策 |

### 10.2 关键要点

1. **完全由模型决定**
   - 系统不分析任务复杂度
   - 系统不自动创建子 Agent
   - 一切取决于模型的推理

2. **Description 是关键**
   - 模型只能看到 description
   - Description 提供使用指导
   - Description 动态生成，包含实际配置

3. **工具可见性是前提**
   - `delegation` 必须在 enabled_toolsets 中
   - 子 Agent 的工具集受限
   - Orchestrator 有特殊权限

4. **用户可以引导**
   - 通过提示词明确要求
   - 通过配置控制可用性
   - 通过 CLAUDE.md 设置偏好

### 10.3 与其他系统的对比

| 系统 | 触发方式 |
|-----|---------|
| **Hermes delegate_task** | 模型自主决策 |
| **AutoGPT** | 系统自动分解任务 |
| **LangChain Agents** | 规则引擎 + 模型决策 |
| **MetaGPT** | 预定义角色和流程 |

Hermes 的设计哲学：
- 信任模型的判断能力
- 通过 description 提供指导
- 保持系统简单和可预测

---

## 11. 源码位置索引

| 功能 | 文件 | 行号 |
|-----|------|------|
| 工具注册 | `tools/delegate_tool.py` | 2783-2801 |
| Description 构建 | `tools/delegate_tool.py` | 2489-2584 |
| 工具定义获取 | `model_tools.py` | ~100-200 |
| 工具调用处理 | `model_tools.py` | ~300-400 |
| 对话循环 | `agent/conversation_loop.py` | ~500-1000 |
| 系统提示构建 | `agent/system_prompt.py` | 60-200 |

---

## 12. 参考资料

- OpenAI Function Calling: https://platform.openai.com/docs/guides/function-calling
- Anthropic Tool Use: https://docs.anthropic.com/claude/docs/tool-use
- Hermes Agent 文档: https://hermes-agent.nousresearch.com/docs