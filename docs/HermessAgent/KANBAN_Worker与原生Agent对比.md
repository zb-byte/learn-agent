# KANBAN Worker 与原生 AIAgent 对比分析

## 核心结论

**KANBAN Worker 本质上就是通过 CLI 启动的标准 AIAgent，使用完全相同的初始化流程和配置。**

唯一的区别是：
1. **环境变量注入**：设置 `HERMES_KANBAN_TASK` 等环境变量
2. **系统提示词追加**：在标准系统提示词基础上追加 KANBAN 生命周期指导
3. **工具集扩展**：自动启用 `kanban` 工具集（如果环境变量存在）

## 详细对比

### 1. 启动方式对比

#### 原生 CLI 启动
```bash
hermes -p <profile> chat
```

**内部流程**：
```python
# cli.py::_init_agent() (line 4579)
self.agent = AIAgent(
    model=effective_model,
    api_key=runtime.get("api_key"),
    base_url=runtime.get("base_url"),
    provider=runtime.get("provider"),
    enabled_toolsets=self.enabled_toolsets,
    disabled_toolsets=self.disabled_toolsets,
    verbose_logging=self.verbose,
    ephemeral_system_prompt=self.system_prompt if self.system_prompt else None,
    max_iterations=self.max_turns,
    session_id=self.session_id,
    platform="cli",
    # ... 其他配置
)
```

#### KANBAN Worker 启动
```bash
hermes -p <profile> --skills kanban-worker chat -q "work kanban task {task_id}"
```

**内部流程**：
```python
# kanban_db.py::_default_spawn() (line 5355-5396)
env = dict(os.environ)
env["HERMES_KANBAN_TASK"] = task.id
env["HERMES_KANBAN_WORKSPACE"] = workspace
env["HERMES_KANBAN_RUN_ID"] = str(task.current_run_id)
env["HERMES_KANBAN_CLAIM_LOCK"] = task.claim_lock

cmd = [
    "hermes", "-p", profile_arg,
    "--accept-hooks",
    "--skills", "kanban-worker",  # 自动加载 skill
    "-m", task.model_override,    # 可选的模型覆盖
    "chat", "-q", f"work kanban task {task.id}"
]

proc = subprocess.Popen(cmd, env=env, ...)
```

**关键点**：
- KANBAN Worker 通过 `subprocess.Popen` 启动一个新的 `hermes` CLI 进程
- 新进程继承修改后的环境变量
- 最终调用的仍然是 `cli.py::_init_agent()`，创建标准的 `AIAgent` 实例

### 2. AIAgent 初始化对比

#### 共同的初始化路径

**两者都通过相同的代码路径创建 AIAgent**：

```python
# cli.py::_init_agent() → agent/agent_init.py::initialize_agent()

from run_agent import AIAgent

agent = AIAgent(
    model=model,
    api_key=api_key,
    base_url=base_url,
    provider=provider,
    enabled_toolsets=enabled_toolsets,
    max_iterations=max_iterations,
    ephemeral_system_prompt=system_prompt,
    session_id=session_id,
    platform="cli",
    # ... 其他参数完全相同
)
```

**初始化流程**（`agent/agent_init.py`）：
1. 加载配置文件（`config.yaml`）
2. 解析模型字符串（provider:model）
3. 初始化模型适配器（Anthropic/OpenAI/Gemini/Bedrock）
4. 注册工具集（通过 `tools/registry.py`）
5. 初始化记忆管理器（`agent/memory_manager.py`）
6. 构建系统提示词（`agent/prompt_builder.py`）
7. 初始化上下文压缩器（`agent/context_compressor.py`）

### 3. 系统提示词对比

#### 原生 CLI 系统提示词结构

```python
# agent/prompt_builder.py::build_system_prompt()

system_prompt = [
    # 1. 基础身份和能力描述
    BASE_SYSTEM_PROMPT,
    
    # 2. 平台特定指导（CLI/Gateway/TUI）
    platform_guidance,
    
    # 3. 技能注入（如果有 --skills）
    skills_content,
    
    # 4. 工具定义
    tool_definitions,
    
    # 5. 记忆系统（如果启用）
    memory_context,
    
    # 6. 上下文文件（CLAUDE.md 等）
    context_files,
]
```

#### KANBAN Worker 系统提示词结构

```python
# agent/prompt_builder.py::build_system_prompt()
# + agent/agent_init.py (line 905-913)

system_prompt = [
    # 1. 基础身份和能力描述（相同）
    BASE_SYSTEM_PROMPT,
    
    # 2. 平台特定指导（相同）
    platform_guidance,
    
    # 3. ⭐ KANBAN 生命周期指导（新增）
    KANBAN_GUIDANCE,  # ~835 tokens
    
    # 4. ⭐ Worker 上下文（新增）
    worker_context,  # 任务详情、父任务、评论、历史尝试
    
    # 5. ⭐ Swarm 上下文（如果是 Swarm 任务）
    swarm_context,
    
    # 6. 技能注入（相同 + kanban-worker skill）
    skills_content,
    
    # 7. 工具定义（相同 + kanban 工具）
    tool_definitions,
    
    # 8. 记忆系统（相同）
    memory_context,
    
    # 9. 上下文文件（相同）
    context_files,
]
```

**KANBAN_GUIDANCE 内容**（`agent/prompt_builder.py` line 188-257）：
```python
KANBAN_GUIDANCE = """
# Kanban task execution protocol

You have been assigned ONE task from the shared board...

## Lifecycle
1. **Orient.** Call `kanban_show()` first...
2. **Work inside the workspace.**...
3. **Heartbeat on long operations.**...
4. **Block on genuine ambiguity.**...
5. **Complete with structured handoff.**...
6. **If follow-up work appears, create it; don't do it.**...

## Orchestrator mode
...

## What NOT to do
...
"""
```

**Worker Context 构建**（`kanban_db.py::build_worker_context()` line 5497）：
```python
def build_worker_context(conn, task_id: str) -> str:
    task = get_task(conn, task_id)
    parents = parent_ids(conn, task_id)
    comments = list_comments(conn, task_id)
    runs = list_runs(conn, task_id)
    
    context = f"""
## Your task

**Title**: {task.title}
**Body**:
{task.body}

**Assignee**: {task.assignee}
**Workspace**: {task.workspace_path}

## Parent task handoffs
{parent_handoffs}

## Comments
{formatted_comments}

## Your prior attempts
{formatted_runs}
"""
    return context
```

### 4. 工具集对比

#### 原生 CLI 工具集

```python
# 通过 config.yaml 或 --toolsets 参数指定
enabled_toolsets = ["file", "terminal", "web", "browser", ...]

# 工具注册（tools/registry.py）
registry.register(
    name="read_file",
    toolset="file",
    schema=READ_FILE_SCHEMA,
    handler=_handle_read_file,
    check_fn=None,  # 总是可用
)
```

**可用工具**：根据配置的 toolsets 决定

#### KANBAN Worker 工具集

```python
# 继承 Profile 配置的 toolsets
enabled_toolsets = profile_config["toolsets"]

# ⭐ 自动追加 kanban 工具集（如果环境变量存在）
if os.environ.get("HERMES_KANBAN_TASK"):
    enabled_toolsets.append("kanban")
```

**KANBAN 工具注册**（`tools/kanban_tools.py`）：
```python
def _check_kanban_mode() -> bool:
    """工具可见性检查"""
    if os.environ.get("HERMES_KANBAN_TASK"):
        return True
    return _profile_has_kanban_toolset()

# 生命周期工具（Worker 可见）
registry.register(
    name="kanban_show",
    toolset="kanban",
    schema=KANBAN_SHOW_SCHEMA,
    handler=_handle_show,
    check_fn=_check_kanban_mode,  # ⭐ 基于环境变量
)

registry.register(
    name="kanban_complete",
    toolset="kanban",
    schema=KANBAN_COMPLETE_SCHEMA,
    handler=_handle_complete,
    check_fn=_check_kanban_mode,
)

# 编排工具（仅 Orchestrator 可见）
registry.register(
    name="kanban_list",
    toolset="kanban",
    schema=KANBAN_LIST_SCHEMA,
    handler=_handle_list,
    check_fn=_check_kanban_orchestrator_mode,  # ⭐ 排除 Worker
)
```

**可用工具**：
- 原生工具（file, terminal, web 等）
- **+ KANBAN 生命周期工具**（show, complete, block, heartbeat, comment, create, link）

### 5. 记忆系统对比

#### 原生 CLI 记忆

```python
# agent/memory_manager.py

memory_manager = MemoryManager(
    session_id=session_id,
    cwd=os.getcwd(),
    providers=["file", "vector", "conversation"],
)

# 记忆存储位置
~/.hermes/memory/
~/.hermes/projects/<project>/memory/
```

**记忆范围**：
- 全局记忆（`~/.hermes/memory/`）
- 项目记忆（基于 git 仓库）
- 会话记忆（conversation history）

#### KANBAN Worker 记忆

```python
# 完全相同的 MemoryManager 初始化
memory_manager = MemoryManager(
    session_id=session_id,  # ⭐ 每个 Worker 有独立 session_id
    cwd=workspace_path,     # ⭐ 工作目录是任务 workspace
    providers=["file", "vector", "conversation"],
)
```

**记忆范围**：
- 全局记忆（相同）
- 项目记忆（基于 workspace 路径）
- **会话记忆（独立）**：每个 Worker 有独立的 session_id
- **+ 任务上下文**：通过 `worker_context` 注入父任务、评论、历史尝试

**关键差异**：
- Worker 的会话记忆是**隔离的**（不同 Worker 不共享对话历史）
- 跨任务信息通过 **KANBAN Comments** 传递，而非共享记忆

### 6. 上下文压缩对比

#### 原生 CLI 上下文压缩

```python
# agent/context_compressor.py

context_compressor = ContextCompressor(
    model=model,
    context_length=context_length,
    auxiliary_model="anthropic:claude-haiku-3-5",
)

# 压缩触发条件
if current_tokens > context_length * 0.8:
    compressed_messages = context_compressor.compress(messages)
```

**压缩策略**：
- 使用辅助模型（Haiku）压缩历史消息
- 保留最近的消息
- 压缩后的摘要作为 system message 注入

#### KANBAN Worker 上下文压缩

```python
# 完全相同的 ContextCompressor
context_compressor = ContextCompressor(
    model=model,
    context_length=context_length,
    auxiliary_model="anthropic:claude-haiku-3-5",
)
```

**压缩策略**：完全相同

**关键差异**：
- Worker 的对话历史通常较短（单任务生命周期）
- 压缩触发频率较低

### 7. 配置文件对比

#### 原生 CLI 配置

```yaml
# ~/.hermes/config.yaml

model: "anthropic:claude-opus-4"
toolsets:
  - file
  - terminal
  - web
  - browser

max_iterations: 90
verbose: false

memory:
  enabled: true
  providers:
    - file
    - vector
```

#### KANBAN Worker 配置

```yaml
# ~/.hermes/profiles/<profile>/config.yaml

model: "anthropic:claude-opus-4"
toolsets:
  - file
  - terminal
  - web
  # ⭐ kanban 工具集通过环境变量自动启用，无需显式配置

max_iterations: 90
verbose: false

memory:
  enabled: true
  providers:
    - file
    - vector

# ⭐ KANBAN 特定配置（可选）
kanban:
  max_spawn: 10  # 最大并发 Worker 数
  max_in_progress: 5  # 软限制
```

**关键点**：
- Worker 使用 **Profile 配置**（`~/.hermes/profiles/<assignee>/config.yaml`）
- `kanban` 工具集**自动启用**（基于 `HERMES_KANBAN_TASK` 环境变量）
- 其他配置（model, toolsets, memory）完全相同

## 核心差异总结表

| 维度 | 原生 CLI | KANBAN Worker | 是否相同 |
|------|---------|---------------|---------|
| **AIAgent 类** | `run_agent.AIAgent` | `run_agent.AIAgent` | ✅ 完全相同 |
| **初始化流程** | `agent_init.py::initialize_agent()` | `agent_init.py::initialize_agent()` | ✅ 完全相同 |
| **模型适配器** | Anthropic/OpenAI/Gemini/Bedrock | Anthropic/OpenAI/Gemini/Bedrock | ✅ 完全相同 |
| **工具注册机制** | `tools/registry.py` | `tools/registry.py` | ✅ 完全相同 |
| **记忆管理器** | `agent/memory_manager.py` | `agent/memory_manager.py` | ✅ 完全相同 |
| **上下文压缩器** | `agent/context_compressor.py` | `agent/context_compressor.py` | ✅ 完全相同 |
| **对话循环** | `agent/conversation_loop.py` | `agent/conversation_loop.py` | ✅ 完全相同 |
| **环境变量** | 标准环境变量 | + `HERMES_KANBAN_TASK` 等 | ⚠️ 追加 |
| **系统提示词** | 标准提示词 | + `KANBAN_GUIDANCE` + `worker_context` | ⚠️ 追加 |
| **工具集** | 配置的 toolsets | 配置的 toolsets + `kanban` | ⚠️ 追加 |
| **Skills** | 配置的 skills | 配置的 skills + `kanban-worker` | ⚠️ 追加 |
| **会话隔离** | 单一会话 | 每个 Worker 独立会话 | ⚠️ 不同 |
| **工作目录** | 用户当前目录 | 任务 workspace | ⚠️ 不同 |

## 代码路径验证

### 验证 1：AIAgent 创建路径

**原生 CLI**：
```
用户输入 → cli.py::run() → cli.py::_init_agent() → AIAgent.__init__()
```

**KANBAN Worker**：
```
Dispatcher → kanban_db.py::_default_spawn() → subprocess.Popen(["hermes", ...])
  → cli.py::run() → cli.py::_init_agent() → AIAgent.__init__()
```

**结论**：最终都调用 `AIAgent.__init__()`，使用相同的初始化逻辑。

### 验证 2：系统提示词构建路径

**原生 CLI**：
```python
# agent/prompt_builder.py::build_system_prompt()

def build_system_prompt(agent, ...):
    parts = []
    parts.append(BASE_SYSTEM_PROMPT)
    parts.append(platform_guidance)
    
    # ⭐ KANBAN 指导（如果环境变量存在）
    if agent._kanban_worker_guidance:
        parts.append(agent._kanban_worker_guidance)
    
    parts.append(skills_content)
    parts.append(tool_definitions)
    parts.append(memory_context)
    parts.append(context_files)
    
    return "\n\n".join(parts)
```

**KANBAN Worker**：
```python
# 完全相同的函数，但 agent._kanban_worker_guidance 不为空

# agent/agent_init.py (line 905-913)
from agent.prompt_builder import KANBAN_GUIDANCE

agent._kanban_worker_guidance = (
    KANBAN_GUIDANCE if "kanban_show" in agent.valid_tool_names else ""
)

# 然后调用相同的 build_system_prompt()
```

**结论**：使用相同的 `build_system_prompt()` 函数，只是 KANBAN Worker 的 `_kanban_worker_guidance` 属性不为空。

### 验证 3：工具可见性检查

**原生 CLI**：
```python
# tools/kanban_tools.py::_check_kanban_mode()

def _check_kanban_mode() -> bool:
    if os.environ.get("HERMES_KANBAN_TASK"):  # ❌ 不存在
        return True
    return _profile_has_kanban_toolset()  # ❌ 通常为 False

# 结果：kanban 工具不可见
```

**KANBAN Worker**：
```python
# tools/kanban_tools.py::_check_kanban_mode()

def _check_kanban_mode() -> bool:
    if os.environ.get("HERMES_KANBAN_TASK"):  # ✅ 存在
        return True
    return _profile_has_kanban_toolset()

# 结果：kanban 工具可见
```

**结论**：工具可见性完全由环境变量控制，使用相同的检查逻辑。

## 实际使用示例

### 示例 1：原生 CLI 使用

```bash
# 启动 CLI
hermes chat

# 用户输入
> 帮我分析这个项目的代码结构

# Agent 行为
- 使用 file 工具读取文件
- 使用 terminal 工具执行命令
- 没有 kanban 工具
- 系统提示词：标准提示词
```

### 示例 2：KANBAN Worker 使用

```bash
# Dispatcher 启动 Worker
hermes -p researcher --skills kanban-worker chat -q "work kanban task task-123"

# 环境变量
HERMES_KANBAN_TASK=task-123
HERMES_KANBAN_WORKSPACE=/tmp/workspace-task-123
HERMES_KANBAN_RUN_ID=1

# Agent 行为
- 首先调用 kanban_show() 获取任务详情
- 使用 file 工具读取文件（相同）
- 使用 terminal 工具执行命令（相同）
- 使用 kanban_complete() 完成任务
- 系统提示词：标准提示词 + KANBAN_GUIDANCE + worker_context
```

### 示例 3：通过 SDK 直接使用（等价于 KANBAN Worker）

```python
import os
from run_agent import AIAgent

# 设置环境变量（模拟 KANBAN Worker）
os.environ["HERMES_KANBAN_TASK"] = "task-123"
os.environ["HERMES_KANBAN_WORKSPACE"] = "/tmp/workspace-task-123"

# 创建 Agent（与 CLI 完全相同的方式）
agent = AIAgent(
    model="anthropic:claude-opus-4",
    enabled_toolsets=["file", "terminal", "web"],  # kanban 自动追加
    max_iterations=10,
)

# 运行对话
result = agent.run_conversation("work kanban task task-123")

# Agent 行为
# - 与 KANBAN Worker 完全相同
# - kanban 工具自动可见（因为环境变量存在）
# - 系统提示词自动包含 KANBAN_GUIDANCE
```

## 结论

### 核心发现

1. **KANBAN Worker 就是标准的 AIAgent**
   - 使用完全相同的 `AIAgent` 类
   - 使用完全相同的初始化流程
   - 使用完全相同的对话循环
   - 使用完全相同的工具注册机制
   - 使用完全相同的记忆管理器
   - 使用完全相同的上下文压缩器

2. **唯一的差异是"注入"而非"替换"**
   - **环境变量注入**：`HERMES_KANBAN_TASK` 等
   - **系统提示词追加**：在标准提示词基础上追加 KANBAN 指导
   - **工具集扩展**：在配置的工具集基础上追加 kanban 工具
   - **Skills 扩展**：在配置的 skills 基础上追加 kanban-worker

3. **外部系统可以完全复制 KANBAN Worker 的行为**
   ```python
   # 方式 1：通过环境变量
   os.environ["HERMES_KANBAN_TASK"] = task_id
   agent = AIAgent(model="...", enabled_toolsets=[...])
   
   # 方式 2：通过 CLI
   subprocess.run([
       "hermes", "-p", profile,
       "--skills", "kanban-worker",
       "chat", "-q", f"work kanban task {task_id}"
   ], env={"HERMES_KANBAN_TASK": task_id, ...})
   
   # 方式 3：通过 KANBAN API
   from hermes_cli import kanban_db as kb
   conn = kb.connect()
   task_id = kb.create_task(conn, title="...", assignee="...")
   # Dispatcher 自动启动 Worker
   ```

4. **设计优势**
   - **一致性**：所有 Agent 使用相同的核心逻辑
   - **可扩展性**：通过环境变量和工具注册扩展功能
   - **可测试性**：可以独立测试 KANBAN 工具，无需修改核心 Agent
   - **可维护性**：KANBAN 功能与核心 Agent 解耦

### 回答你的问题

> "我就是想通过源码确认下是否这么用跟原生的方式一模一样，比如系统提示词还有工具记忆等等"

**答案：是的，KANBAN Worker 与原生 AIAgent 使用完全相同的方式。**

- ✅ **系统提示词**：使用相同的构建函数，只是追加了 KANBAN 指导
- ✅ **工具**：使用相同的注册机制，只是追加了 kanban 工具集
- ✅ **记忆**：使用相同的 MemoryManager，只是工作目录不同
- ✅ **上下文压缩**：使用相同的 ContextCompressor
- ✅ **对话循环**：使用相同的 conversation_loop
- ✅ **模型适配器**：使用相同的 adapter 层

**唯一的区别是"追加"而非"替换"**，这是通过环境变量和工具注册的 `check_fn` 实现的。
