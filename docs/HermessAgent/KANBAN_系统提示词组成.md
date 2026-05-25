# KANBAN Worker 系统提示词组成详解

## 概述

当 KANBAN 系统创建一个新的 Agent Worker 时，系统提示词由多个部分组成，这些部分共同构成了 Worker 的完整上下文和行为指导。

## Worker 启动流程

### 1. 任务分发 (Dispatcher)

**位置**: `hermes_cli/kanban_db.py` 中的 `_default_spawn()` 函数

```python
def _default_spawn(task: Task, workspace: str, *, board: Optional[str] = None) -> Optional[int]:
    """启动 hermes -p <profile> chat -q ... 子进程"""
    
    # 构建初始提示词
    prompt = f"work kanban task {task.id}"
    
    # 设置环境变量
    env = dict(os.environ)
    env["HERMES_KANBAN_TASK"] = task.id           # 任务 ID
    env["HERMES_KANBAN_WORKSPACE"] = workspace     # 工作空间路径
    env["HERMES_KANBAN_RUN_ID"] = str(task.current_run_id)  # 运行 ID
    env["HERMES_KANBAN_CLAIM_LOCK"] = task.claim_lock       # 声明锁
    env["HERMES_PROFILE"] = profile_arg            # Profile 名称
    
    # 构建命令
    cmd = [
        "hermes", "-p", profile_arg,
        "--accept-hooks",
        "--skills", "kanban-worker",  # 自动加载 kanban-worker skill
        "chat", "-q", prompt          # -q 表示 quick query
    ]
```

**关键点**:
- 使用 `chat -q "work kanban task {task.id}"` 启动 Worker
- 通过环境变量 `HERMES_KANBAN_TASK` 标识这是一个 KANBAN Worker
- 自动加载 `kanban-worker` skill（如果可用）
- 可以通过 `task.skills` 字段强制加载额外的 skills

### 2. Agent 初始化

**位置**: `agent/agent_init.py` 中的 `init_agent()` 函数

```python
# 检测是否为 KANBAN Worker（通过环境变量）
from agent.prompt_builder import KANBAN_GUIDANCE
agent._kanban_worker_guidance = (
    KANBAN_GUIDANCE if "kanban_show" in agent.valid_tool_names else ""
)
```

**关键点**:
- 如果 `HERMES_KANBAN_TASK` 环境变量存在，则 `kanban_show` 等工具会被注册
- `KANBAN_GUIDANCE` 会被添加到系统提示词中（约 835 tokens）

## 系统提示词组成部分

### 第一部分: 基础 Agent 身份

**位置**: `agent/prompt_builder.py` 中的 `build_system_prompt()` 函数

包含:
1. **Agent 基础身份**: 来自 `agent/system_prompt.py`
2. **工具使用指导**: 根据模型类型添加（GPT/Gemini/Qwen 等）
3. **上下文文件**: 扫描 `HERMES.md`, `AGENTS.md`, `SOUL.md` 等
4. **Memory 内容**: 从 Memory Provider 加载的相关记忆

### 第二部分: KANBAN 生命周期指导

**位置**: `agent/prompt_builder.py` 中的 `KANBAN_GUIDANCE` 常量

```python
KANBAN_GUIDANCE = (
    "# Kanban task execution protocol\n"
    "You have been assigned ONE task from the shared board at `~/.hermes/kanban.db`. "
    "Your task id is in `$HERMES_KANBAN_TASK`; your workspace is `$HERMES_KANBAN_WORKSPACE`. "
    "The `kanban_*` tools in your schema are your primary coordination surface — "
    "they write directly to the shared SQLite DB and work regardless of terminal "
    "backend (local/docker/modal/ssh).\n"
    "\n"
    "## Lifecycle\n"
    "\n"
    "1. **Orient.** Call `kanban_show()` first (no args — it defaults to your task). "
    "   The response includes title, body, parent-task handoffs (summary + metadata), "
    "   any prior attempts on this task if you're a retry, the full comment thread, "
    "   and a pre-formatted `worker_context` you can treat as ground truth.\n"
    "2. **Work inside the workspace.** `cd $HERMES_KANBAN_WORKSPACE` before any file operations. "
    "   The workspace is yours for this run. Don't modify files outside it unless the task explicitly asks.\n"
    "3. **Heartbeat on long operations.** Call `kanban_heartbeat(note=...)` every few minutes "
    "   during long subprocesses (training, encoding, crawling). Skip heartbeats for short tasks. "
    "   **If your task may run longer than 1 hour, you MUST call `kanban_heartbeat` at least once an hour**\n"
    "4. **Block on genuine ambiguity.** If you need a human decision you cannot infer "
    "   (missing credentials, UX choice, paywalled source, peer output you need first), "
    "   call `kanban_block(reason=\"...\")` and stop. Don't guess.\n"
    "5. **Complete with structured handoff.** Call `kanban_complete(summary=..., metadata=...)`. "
    "   `summary` is 1–3 human-readable sentences naming concrete artifacts. "
    "   `metadata` is machine-readable facts (`{changed_files: [...], tests_run: N, decisions: [...]}`). "
    "   Downstream workers read both via their own `kanban_show`.\n"
    "6. **If follow-up work appears, create it; don't do it.** Use "
    "   `kanban_create(title=..., assignee=<right-profile>, parents=[your-task-id])` "
    "   to spawn a child task for the appropriate specialist profile instead of scope-creeping.\n"
    "\n"
    "## Orchestrator mode\n"
    "\n"
    "If your task is itself a decomposition task (e.g. a planner profile given a high-level goal), "
    "use `kanban_create` to fan out into child tasks — one per specialist, each with an explicit "
    "`assignee` and `parents=[...]` to express dependencies. Then `kanban_complete` your own task "
    "with a summary of the decomposition. Do NOT execute the work yourself; your job is routing, "
    "not implementation.\n"
    "\n"
    "## Do NOT\n"
    "\n"
    "- Do not shell out to `hermes kanban <verb>` for board operations. Use the `kanban_*` tools.\n"
    "- Do not complete a task you didn't actually finish. Block it.\n"
    "- Do not assign follow-up work to yourself. Assign it to the right specialist profile.\n"
    "- Do not call `delegate_task` as a board substitute. `delegate_task` is for short reasoning "
    "  subtasks inside your own run; board tasks are for cross-agent handoffs that outlive one API loop."
)
```

**长度**: 约 1,500-4,096 字符（约 835 tokens）

**触发条件**: 当 `kanban_show` 工具在 `agent.valid_tool_names` 中时添加

### 第三部分: Worker Context（任务上下文）

**位置**: `hermes_cli/kanban_db.py` 中的 `build_worker_context()` 函数

当 Worker 调用 `kanban_show()` 工具时，会返回完整的任务上下文：

```python
def build_worker_context(conn: sqlite3.Connection, task_id: str) -> str:
    """返回 Worker 理解任务所需的完整文本"""
    
    # 构建内容顺序:
    # 1. 任务标题（必需）
    # 2. 任务 Body（可选，最大 8 KB）
    # 3. 本任务的先前尝试记录（最近 N 次）
    # 4. 所有已完成父任务的结构化交接结果
    # 5. 该 Assignee 的跨任务角色历史（最近 5 次完成的运行）
    # 6. 评论线程（最近 N 条评论）
```

**返回格式示例**:

```markdown
# Kanban task abc-123: 实现用户认证功能

Assignee: backend-dev
Status:   running
Tenant:   myproject
Workspace: scratch @ /tmp/hermes-workspace-abc-123
Max runtime: 3600s
Terminal timeout: 3600s

## Body
实现基于 JWT 的用户认证系统，包括登录、注册、token 刷新功能。

## Prior attempts on this task
### Attempt 1 — failed (backend-dev, 2026-05-24 10:30)
尝试使用 bcrypt 加密密码，但遇到依赖安装问题
_error_: ModuleNotFoundError: No module named 'bcrypt'
_metadata_: `{"attempted_approach": "bcrypt", "blocker": "missing_dependency"}`

## Parent task results
### task-parent-001
完成了数据库 schema 设计，用户表已创建
_metadata_: `{"tables_created": ["users", "sessions"], "migration_file": "001_create_users.sql"}`

## Cross-task role history
### Recent completions by backend-dev
- task-xyz-789: 实现了 API 限流中间件 (2026-05-23)
- task-xyz-456: 修复了数据库连接池泄漏 (2026-05-22)

## Comments
**reviewer** (2026-05-24 11:00): 建议使用 argon2 而不是 bcrypt，性能更好
**backend-dev** (2026-05-24 11:05): 收到，将切换到 argon2
```

**字段限制**:
- Task Body: 最大 8 KB
- 每个字段（summary/error/metadata）: 最大 4 KB
- 先前尝试: 显示最近 5 次
- 评论: 显示最近 20 条
- 跨任务历史: 最近 5 次完成的运行

### 第四部分: Swarm Context（如果是 Swarm 任务）

**位置**: `hermes_cli/kanban_swarm.py` 中的 `_swarm_context()` 函数

如果任务是通过 `create_swarm()` 创建的，任务 Body 会自动追加 Swarm 协议说明：

```python
def _swarm_context(root_id: str, goal: str) -> str:
    return (
        "\n\n## Swarm protocol\n"
        f"- Swarm root / shared blackboard: `{root_id}`.\n"
        "- Read sibling/parent handoffs from Kanban context before working.\n"
        "- Put machine-readable facts in completion metadata.\n"
        "- Put cross-worker notes on the root task using structured comments.\n"
        f"- Goal: {goal.strip()}\n"
    )
```

**示例**:

```markdown
## Swarm protocol
- Swarm root / shared blackboard: `swarm-root-001`.
- Read sibling/parent handoffs from Kanban context before working.
- Put machine-readable facts in completion metadata.
- Put cross-worker notes on the root task using structured comments.
- Goal: 分析 Q3 季度销售数据并生成报告
```

### 第五部分: Skills 内容

**位置**: 通过 `--skills` 参数加载

1. **自动加载**: `kanban-worker` skill（如果可用）
   - 包含最佳实践示例
   - 良好的 summary/metadata 格式
   - 重试诊断指导
   - Block reason 示例

2. **任务指定**: 通过 `task.skills` 字段强制加载
   - 例如: `['translation']` 用于翻译任务
   - 例如: `['github-code-review']` 用于代码审查任务

### 第六部分: 工具定义

**位置**: `model_tools.py` 中的 `get_tool_definitions()` 函数

KANBAN Worker 可用的工具包括:

**生命周期工具** (来自 `tools/kanban_tools.py`):
- `kanban_show`: 读取任务完整状态
- `kanban_complete`: 标记任务完成
- `kanban_block`: 标记任务阻塞
- `kanban_heartbeat`: 发送心跳信号
- `kanban_comment`: 添加评论
- `kanban_create`: 创建子任务
- `kanban_link`: 添加任务依赖

**编排工具** (仅限编排器 Profile):
- `kanban_list`: 列出任务
- `kanban_unblock`: 解除任务阻塞

**其他工具**: 根据 Profile 的 `toolsets` 配置加载

## 完整系统提示词结构

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 基础 Agent 身份                                           │
│    - Agent 角色定义                                          │
│    - 工具使用指导（根据模型类型）                             │
│    - 上下文文件内容（HERMES.md, AGENTS.md, SOUL.md）         │
│    - Memory 内容                                             │
├─────────────────────────────────────────────────────────────┤
│ 2. KANBAN 生命周期指导 (KANBAN_GUIDANCE)                     │
│    - 任务执行协议                                            │
│    - 6 步生命周期                                            │
│    - 编排器模式说明                                          │
│    - 禁止事项列表                                            │
│    约 835 tokens                                             │
├─────────────────────────────────────────────────────────────┤
│ 3. Skills 内容                                               │
│    - kanban-worker skill（自动加载）                         │
│    - 任务指定的额外 skills                                   │
├─────────────────────────────────────────────────────────────┤
│ 4. 工具定义 (Tool Schemas)                                   │
│    - KANBAN 生命周期工具                                     │
│    - Profile 配置的其他工具                                  │
└─────────────────────────────────────────────────────────────┘

首次对话消息（来自 kanban_show() 工具调用）:
┌─────────────────────────────────────────────────────────────┐
│ Worker Context (build_worker_context 返回)                   │
│    - 任务标题和元数据                                        │
│    - 任务 Body                                               │
│    - 先前尝试记录                                            │
│    - 父任务交接结果                                          │
│    - 跨任务角色历史                                          │
│    - 评论线程                                                │
│    - Swarm Context（如果适用）                               │
└─────────────────────────────────────────────────────────────┘
```

## 环境变量传递

Worker 进程会接收以下环境变量:

```bash
HERMES_KANBAN_TASK=abc-123              # 任务 ID
HERMES_KANBAN_WORKSPACE=/tmp/workspace  # 工作空间路径
HERMES_KANBAN_RUN_ID=42                 # 运行 ID
HERMES_KANBAN_CLAIM_LOCK=uuid-xxx       # 声明锁
HERMES_KANBAN_BRANCH=feature-branch     # Git 分支（如果有）
HERMES_PROFILE=backend-dev              # Profile 名称
HERMES_TENANT=myproject                 # 租户（如果有）
HERMES_KANBAN_DB=/path/to/kanban.db     # 数据库路径
HERMES_KANBAN_BOARD=default             # Board 名称
TERMINAL_TIMEOUT=3600                   # 终端超时（如果有）
```

## 关键代码路径

### 1. Worker 启动
```
hermes_cli/kanban_db.py::_default_spawn()
  ↓
subprocess.Popen(["hermes", "-p", profile, "chat", "-q", "work kanban task {id}"])
  ↓
cli.py::AIAgent 初始化
  ↓
agent/agent_init.py::init_agent()
```

### 2. 系统提示词构建
```
agent/agent_init.py::init_agent()
  ↓ 检测 HERMES_KANBAN_TASK 环境变量
  ↓ 如果存在，注册 kanban_* 工具
  ↓ 设置 agent._kanban_worker_guidance = KANBAN_GUIDANCE
  ↓
agent/prompt_builder.py::build_system_prompt()
  ↓ 组装基础身份 + KANBAN_GUIDANCE + Skills + 上下文文件
  ↓
返回完整系统提示词
```

### 3. Worker Context 获取
```
Worker 首次调用 kanban_show()
  ↓
tools/kanban_tools.py::_handle_show()
  ↓
hermes_cli/kanban_db.py::build_worker_context()
  ↓ 查询任务、运行、评论、父任务
  ↓ 组装完整上下文
  ↓
返回 Markdown 格式的 Worker Context
```

## 设计要点

### 1. 分层设计
- **系统提示词**: 静态的生命周期指导和协议
- **Worker Context**: 动态的任务特定信息
- **工具调用**: 实时的状态查询和更新

### 2. 上下文控制
- 所有字段都有大小限制，防止上下文爆炸
- 优先显示最近的信息（先前尝试、评论、历史）
- 旧信息折叠为摘要

### 3. 工具门控
- 通过 `check_fn` 检查 `HERMES_KANBAN_TASK` 环境变量
- Worker 只能看到生命周期工具
- 编排器可以看到额外的列表和解除阻塞工具

### 4. 跨后端兼容
- 工具直接操作 SQLite 数据库
- 不依赖终端后端（local/docker/modal/ssh）
- 环境变量传递确保上下文一致

## 总结

KANBAN Worker 的系统提示词是一个多层次的组合:

1. **静态层**: 基础 Agent 身份 + KANBAN_GUIDANCE（约 835 tokens）
2. **配置层**: Skills 内容 + 工具定义
3. **动态层**: Worker Context（通过 `kanban_show()` 获取）
4. **环境层**: 环境变量传递任务元数据

这种设计确保了:
- Worker 有明确的生命周期协议
- 任务上下文完整且有界
- 跨任务协作通过结构化交接
- 系统可扩展且易于调试
