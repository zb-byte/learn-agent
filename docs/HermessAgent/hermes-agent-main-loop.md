# Hermes-Agent 主循环（The While Loop）详解

> 基于 `hermes-agent` 源码（`agent/conversation_loop.py`）的结构化导读。本文对齐 `hermes-agent-runtime-reader-guide.md` 的深度：源码地图、生命周期表、易错点修正。

---

## 目录

1. [一眼看懂](#1-一眼看懂)
2. [源码地图](#2-源码地图)
3. [一轮用户输入的完整生命周期](#3-一轮用户输入的完整生命周期)
4. [主循环条件与双计数器](#4-主循环条件与双计数器)
5. [单次迭代的流水线](#5-单次迭代的流水线)
6. [工具执行与并行策略](#6-工具执行与并行策略)
7. [循环退出与 `turn_exit_reason`](#7-循环退出与-turn_exit_reason)
8. [预算耗尽后的真实路径（非宽限期）](#8-预算耗尽后的真实路径非宽限期)
9. [循环结束后的后处理](#9-循环结束后的后处理)
10. [与 `delegate_task` / 压缩 / 权限的交界](#10-与-delegatetask--压缩--权限的交界)
11. [常见问题深度解答（FAQ）](#11-常见问题深度解答faq)
12. [讲者展开点与常见误读](#12-讲者展开点与常见误读)

---

## 1. 一眼看懂

| 结论 | 含义 |
|---|---|
| 入口是薄转发 | `run_agent.py` 的 `AIAgent.run_conversation` 仅转发到 `agent/conversation_loop.run_conversation` |
| 核心在 `conversation_loop.py` | 单文件约 4000 行，承载一整轮 user turn 的模型调用、工具分发、重试、压缩、收尾 |
| 外层 `while` 是「模型 API 轮次」 | 每轮通常 = 一次 `chat.completions`（或等价 transport）+ 可选工具批处理 |
| 双闸门 | `api_call_count < max_iterations` **且** `iteration_budget.remaining > 0` 同时成立才继续（另有未接线的 grace 分支，见 §8） |
| 工具批处理可并行 | 只读白名单 + 路径不冲突 + MCP 显式 opt-in；`delegate_task` / `clarify` 等强制串行 |
| 权限单一入口 | `tools/approval.py` 负责危险命令检测、会话 allowlist、CLI/Gateway/ACP 三条交互路径 |
| 预算耗尽 ≠ 直接失败 | 循环退出后调用 `_handle_max_iterations`：注入总结请求 + **一次无工具** API 调用 |

**类比：**

```
用户 = 项目经理
大模型 = 技术负责人
工具 = 开发团队
主循环 = 每日站会循环
权限审批 = 安全审查

每次循环：
1. 项目经理（用户）提出需求
2. 技术负责人（模型）分析并分配任务
3. 开发团队（工具）执行任务 → 需要安全审查时等待批准
4. 回到第1步，直到项目完成
```

一句话：

```text
run_conversation = 初始化一轮 turn →（可选预压缩）→ while 模型迭代 → 工具/权限/文本分支 → 收尾持久化与记忆同步。
```

---

## 2. 源码地图

| 关注点 | 关键位置 | 读源码时看什么 |
|---|---|---|
| 公共入口 | `run_agent.py` `AIAgent.run_conversation` (~3867) | 仅 `return run_conversation(self, ...)` |
| 主循环本体 | `agent/conversation_loop.py` `run_conversation` (187–4095) | `while` 条件、内层 API retry、工具/无工具分支 |
| 迭代预算 | `agent/iteration_budget.py` `IterationBudget` | `consume` / `refund`、线程锁、子 Agent 独立预算 |
| 预算耗尽收尾 | `agent/chat_completion_helpers.py` `handle_max_iterations` (910+) | 注入 user 总结请求、无 tools 的 API 调用 |
| 工具路由 | `run_agent.py` `_execute_tool_calls` (~3778) | 串行 vs 并发选择 |
| 并行策略 | `agent/tool_dispatch_helpers.py` `_should_parallelize_tool_batch` (103+) | 白名单、路径冲突、`clarify` 禁并行 |
| 工具执行实现 | `agent/tool_executor.py` | `delegate_task` 专用分支、并发 worker 上限 8 |
| **权限核心** | `tools/approval.py` | `DANGEROUS_PATTERNS`、`detect_dangerous_command`、`check_all_command_guards` |
| **CLI 回调** | `cli.py` `set_approval_callback` | 绑定 `prompt_dangerous_approval` |
| **Gateway 队列** | `approval.py` `register_gateway_notify` (517+)、`resolve_gateway_approval` | FIFO + `threading.Event` |
| **ACP 终端** | `acp_adapter/permissions.py` `make_approval_callback` (107+) | option id → once/session/always/deny |
| **ACP 编辑** | `acp_adapter/edit_approval.py` | `maybe_require_edit_approval` |
| 预压缩 / 回合后压缩 | `conversation_loop.py` 422–488、3369–3410 | 触发 token 来源、压缩后 `continue` |
| 压缩编排 | `agent/conversation_compression.py` `compress_context` | session 切分、system prompt 重建 |
| Codex 旁路 | `conversation_loop.py` 589–596 | `api_mode == codex_app_server` 整轮 bypass Hermes loop |

**源码证据：**

```python
# agent/conversation_loop.py:1-15
"""The agent conversation loop — extracted from ``run_agent.AIAgent``.

This is the biggest single chunk pulled out of ``run_agent.py``: the
roughly 3,900-line :func:`run_conversation` body that drives one user
turn through the agent (model call, tool dispatch, retries, fallbacks,
compression, post-turn hooks, background memory/skill review nudges).
"""
```

---

## 3. 一轮用户输入的完整生命周期

```text
用户消息进入 run_conversation
  │
  ├─ 初始化：DB session、task_id、IterationBudget、消息列表、system prompt
  ├─ 可选：preflight 压缩（最多 3 遍，见 context-compression 文档）
  ├─ 可选：pre_llm_call 插件 → 仅向 user 消息注入 ephemeral 内容（不碰 system，利于 prompt cache）
  │
  ├─ 若 api_mode == codex_app_server → _run_codex_app_server_turn → return（不走下方 while）
  │
  ▼
┌──────────────────────────────────────────────────────────────┐
│  while (api_call_count < max_iterations                       │
│         and iteration_budget.remaining > 0)                  │
│     or _budget_grace_call   ← 当前仓库内从未被设为 True       │
│    每轮：checkpoint → 中断检查 → api_call_count++ → 消耗预算  │
│         → 构建 api_messages → 内层 API retry 循环             │
│         → 有 tool_calls？执行工具 + 权限检查 : 文本结束       │
│         → 回合后压缩 / continue / break                       │
└──────────────────────────────────────────────────────────────┘
  │
  ├─ 若仍无 final_response 且预算/API 次数耗尽 → _handle_max_iterations
  ├─ _persist_session、轨迹、诊断日志
  ├─ _sync_external_memory_for_turn（中断时跳过）
  ├─ _spawn_background_review（memory / skill nudge）
  └─ 返回 dict（含 turn_exit_reason、messages、cost 等）
```

**主循环负责：**

| 职责 | 说明 |
|------|------|
| **模型调用** | 构建 API 请求，调用大模型 |
| **工具分发** | 解析 tool_calls，分发到对应工具 |
| **权限控制** | 危险命令检测、用户审批、会话 allowlist |
| **错误处理** | 处理 API 错误、工具错误、格式错误 |
| **上下文管理** | 检查压缩阈值，触发压缩 |
| **预算控制** | 跟踪迭代次数，防止无限循环 |
| **记忆管理** | 注入外部记忆，保存持久化记忆 |
| **生命周期** | 处理 Session 开始/结束事件 |

---

## 4. 主循环条件与双计数器

### 4.1 循环条件（源码）

```598:623:agent/conversation_loop.py
    while (api_call_count < agent.max_iterations and agent.iteration_budget.remaining > 0) or agent._budget_grace_call:
        agent._checkpoint_mgr.new_turn()
        if agent._interrupt_requested:
            ...
            break
        api_call_count += 1
        ...
        if agent._budget_grace_call:
            agent._budget_grace_call = False
        elif not agent.iteration_budget.consume():
            _turn_exit_reason = "budget_exhausted"
            ...
            break
```

### 4.2 `api_call_count` vs `iteration_budget`

| 维度 | `api_call_count` | `iteration_budget` (`IterationBudget`) |
|---|---|---|
| 含义 | 本 turn 已发起的模型 API 轮次数 | 可消耗的「迭代预算」计数（默认与 `max_iterations` 同上限） |
| 上限来源 | `agent.max_iterations`（默认 90） | 每 turn 新建：`IterationBudget(agent.max_iterations)` |
| 递增 | 每进入 `while` 体一次 `+= 1` | 每轮 `consume()` 成功 +1（grace 分支除外） |
| 退还 | 压缩重启时 `-= 1` | 压缩重启 `refund()`；仅 `execute_code` 单工具批后 `refund()` |
| 耗尽时 | 与 `remaining <= 0` 一起触发 post-loop `_handle_max_iterations` | 循环内 `consume()` 失败 → `budget_exhausted` 并 `break` |

**可视化：**

```
┌─────────────────────────────────────────────────────────────┐
│  双计数器状态                                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  api_call_count:  ████━━━━━━━━━━━━━━━━━━  (只增不减)        │
│                     ↑                                         │
│                  累计值 = 实际API调用次数                      │
│                                                             │
│  iteration_budget: ████████████━━━━━━━━  (可消耗/退还)       │
│                       ↑↑↑                                    │
│                    消耗后可退还                               │
│                                                             │
│  检查点：                                                    │
│  ├─ api_call_count < 90 ✅                                  │
│  ├─ budget.remaining > 0 ✅                                 │
│  └─ 允许继续循环                                            │
└─────────────────────────────────────────────────────────────┘
```

**子 Agent：** 父、子各持独立 `IterationBudget`；子 Agent 上限来自 `delegation.max_iterations`（默认 50），父子合计可超过父的 90（见 `iteration_budget.py` 模块注释）。

### 4.3 `IterationBudget` 真实 API（勿按简化伪代码理解）

```17:59:agent/iteration_budget.py
class IterationBudget:
    def __init__(self, max_total: int):
        self.max_total = max_total
        self._used = 0
        self._lock = threading.Lock()

    def consume(self) -> bool:
        with self._lock:
            if self._used >= self.max_total:
                return False
            self._used += 1
            return True

    def refund(self) -> None:
        with self._lock:
            if self._used > 0:
                self._used -= 1

    @property
    def used(self) -> int: ...
    @property
    def remaining(self) -> int: ...
```

---

## 5. 单次迭代的流水线

### 5.1 进入 API 之前（每轮外层迭代）

| 步骤 | 作用 |
|---|---|
| `checkpoint_mgr.new_turn()` | 每轮迭代允许一次 checkpoint 快照 |
| `_interrupt_requested` | Gateway `/stop` 等：在迭代边界 break |
| `api_call_count += 1` | 统计 API 轮次 |
| `iteration_budget.consume()` | 消耗预算（grace 分支见 §8） |
| `step_callback` | Gateway `agent:step` 等 |
| Pre-API `/steer` drain | 若 steer 在上一轮 API 期间到达，尽量注入最近 `tool` 消息 |
| 消息修复 | `_sanitize_tool_call_arguments`、`_repair_message_sequence` |
| 构建 `api_messages` | 复制 messages；**仅 API 层** 给当前 user 拼 memory prefetch；剥离 `reasoning` 等内部字段 |
| `apply_anthropic_cache_control` | `system_and_3`：system + 最后 3 条非 system 打 cache 断点 |

### 5.2 内层 API retry 循环

每次外层迭代内还有 `while retry_count < max_retries`，处理：

- 网络/限流重试
- **413 / context overflow / long_context_tier** → 触发压缩后重试（详见 compression 文档）
- fallback 模型切换
- 压缩成功后：`api_call_count -= 1` + `iteration_budget.refund()`，**不计入一次完整迭代**

### 5.3 API 成功后的分支

```text
response 归一化
  → post_api_request 插件
  → incomplete scratchpad / Codex incomplete 等特殊重试
  → 有 tool_calls？
        ├─ 是：校验工具名与 JSON → append assistant → _execute_tool_calls
        │       → 权限检查？→ 等待用户审批 / guardrail_halt
        │       → 仅 execute_code？→ iteration_budget.refund()
        │       → 回合后压缩？→ _compress_context → continue
        │       → _save_session_log → continue
        └─ 否：空响应/流式恢复/housekeeping 回退 → 最终 assistant 文本 → break
```

**关键检查点：**

中断检查（步骤1）：

```python
# agent/conversation_loop.py:602-608
if agent._interrupt_requested:
    interrupted = True
    _turn_exit_reason = "interrupted_by_user"
    if not agent.quiet_mode:
        agent._safe_print("\n⚡ Breaking out of tool loop due to interrupt...")
    break
```

预算消耗（步骤2）：

```python
# agent/conversation_loop.py:617-623
if agent._budget_grace_call:
    agent._budget_grace_call = False
elif not agent.iteration_budget.consume():
    _turn_exit_reason = "budget_exhausted"
    if not agent.quiet_mode:
        agent._safe_print(f"\n⚠️  Iteration budget exhausted ({agent.iteration_budget.used}/{agent.iteration_budget.max_total} iterations used)")
    break
```

---

## 6. 工具执行与并行策略

### 6.1 路由入口

```3778:3797:run_agent.py
    def _execute_tool_calls(self, assistant_message, messages, ...):
        tool_calls = assistant_message.tool_calls
        if len(tool_calls) <= 1:
            return self._execute_tool_calls_sequential(...)
        if not _should_parallelize_tool_batch(tool_calls):
            return self._execute_tool_calls_sequential(...)
        return self._execute_tool_calls_concurrent(...)
```

### 6.2 何时并行

| 规则 | 说明 |
|---|---|
| `len(tool_calls) <= 1` | 始终串行 |
| 含 `clarify` | `_NEVER_PARALLEL_TOOLS`，整批串行 |
| 含 `delegate_task` | 不在 `_PARALLEL_SAFE_TOOLS`，通常整批串行 |
| 白名单工具 | `read_file`、`web_search`、`skills_list`、`skill_view`、`session_search` 等 |
| 路径类工具 | `read_file` / `write_file` / `patch`：目标路径不重叠才可并行 |
| MCP | 仅当 `is_mcp_tool_parallel_safe(tool_name)` 为真 |
| Worker 上限 | `tool_executor.py` 中 `_MAX_TOOL_WORKERS = 8` |

**并行执行策略：**

```python
# agent/tool_executor.py:54-55
# Maximum number of concurrent worker threads for parallel tool execution.
_MAX_TOOL_WORKERS = 8
```

**实际例子：**

```
模型响应：
{
  "tool_calls": [
    {"id": "1", "function": {"name": "read_file", "arguments": "{\"path\": \"config.json\"}"}},
    {"id": "2", "function": {"name": "read_file", "arguments": "{\"path\": \"package.json\"}"}},
    {"id": "3", "function": {"name": "search_files", "arguments": "{\"pattern\": \"TODO\"}"}}
  ]
}

并行执行：
Thread-1: read_file(config.json) ──→ result_1
Thread-2: read_file(package.json) ──→ result_2
Thread-3: search_files("TODO") ──→ result_3

结果收集（按原始顺序）：
messages.append({"role": "tool", "tool_call_id": "1", "content": result_1})
messages.append({"role": "tool", "tool_call_id": "2", "content": result_2})
messages.append({"role": "tool", "tool_call_id": "3", "content": result_3})
```

**源码证据：**

```python
# agent/tool_executor.py:64-69
def execute_tool_calls_concurrent(agent, assistant_message, messages: list, effective_task_id: str, api_call_count: int = 0) -> None:
    """Execute multiple tool calls concurrently using a thread pool.

    Results are collected in the original tool-call order and appended to
    messages so the API sees them in the expected sequence.
    """
```

### 6.3 Agent 级工具（不经 `registry.dispatch` 的普通路径）

在串行执行器内优先拦截，包括但不限于：

- `todo`、`session_search`、`memory`、`clarify`
- **`delegate_task`** → `agent._dispatch_delegate_task` → `tools.delegate_tool.delegate_task(parent_agent=...)`
- context-engine / memory-provider 相关工具

`delegate_task` 在串行路径有独立 spinner 与分发逻辑（`tool_executor.py` ~672–696），避免与普通 `handle_function_call` 混用。

### 6.4 工具权限确认机制

#### 6.4.1 一眼看懂权限系统

| 结论 | 含义 |
|---|---|
| 单一模块 | `tools/approval.py` 负责检测、会话 allowlist、CLI prompt、Gateway 队列、smart 审批 |
| 入口 | `terminal` 等通过 `check_all_command_guards()` 统一过闸 |
| 三种模式 | `manual` / `smart` / `off`（yolo）；**硬底线**模式任何模式都不跳过 |
| CLI | 同步 `prompt_dangerous_approval`（需 `HERMES_INTERACTIVE=1`） |
| Gateway | 异步通知 + **阻塞 agent 线程**，用户 `/approve` 或 `/deny` |
| ACP | `make_approval_callback` 桥接到 `request_permission` JSON-RPC |
| 文件编辑 | **另一套**：`acp_adapter/edit_approval.py`，仅 `write_file` / `patch` |

```text
危险命令 → detect → 按运行面（CLI/Gateway/ACP）收集用户决策 → 写入 session/permanent allowlist → 再执行 terminal。
```

#### 6.4.2 检测层：危险命令与硬底线

**危险命令检测：**

```470:481:tools/approval.py
def detect_dangerous_command(command: str):
    # 归一化：去 ANSI、NFKC、小写
    # 返回 (is_dangerous, pattern_key, description)
```

`DANGEROUS_PATTERNS` 覆盖：删库路径、`rm -rf /`、改 shell rc、读 `.env` / `~/.ssh`、管道到 `sh` 等（模块内 `_SSH_SENSITIVE_PATH`、`_HERMES_ENV_PATH` 等组合）。

**硬底线（`HARDLINE_PATTERNS`）：**

低于 yolo 的 **绝对禁止**（`check_all_command_guards` ~1056–1063）：

- 例如 `rm -rf /`、`mkfs`、fork bomb、`shutdown` 等
- **`approvals` 设为 `off` 也不能绕过**

#### 6.4.3 三条交互路径对比

```text
                    terminal_tool
                          │
                          ▼
              check_all_command_guards()
                          │
      ┌───────────────────┼───────────────────┐
      ▼                   ▼                   ▼
  容器后端            CLI 交互              Gateway / ACP
  docker/modal/      HERMES_INTERACTIVE     队列 + notify /
  daytona...         prompt_dangerous_      approval_callback
  → 自动通过          approval 同步           → 阻塞线程等待
```

| 表面 | 如何识别 | 用户如何批准 | 典型超时 |
|---|---|---|---|
| **CLI** | `HERMES_INTERACTIVE=1`（`cli.py`） | 终端同步 y/n；`set_approval_callback` | 用户输入 |
| **Gateway** | `_is_gateway_approval_context()` / session contextvars | 平台消息 + `/approve` `/deny`；阻塞 agent 线程 | `approvals.gateway_timeout`（默认 300s） |
| **ACP** | `make_approval_callback` 注入 agent | IDE `request_permission` UI | 默认 60s，超时 **deny** |
| **Batch/非交互** | 无 TTY、非 gateway | 非 cron 危险命令可能直接 deny；见 cron | — |

**容器类执行环境**（`docker`、`modal`、`daytona` 等）：host 不可达，检测层 **自动 approve**（`check_all_command_guards` ~1052–1054），避免无意义阻塞。

#### 6.4.4 Gateway 阻塞队列模型

Gateway 上 agent 在 **工作线程** 跑 `run_conversation`；用户审批在 **另一异步上下文** 完成。`approval.py` 用队列协调：

```text
Agent 线程                          Gateway 事件循环
    │                                      │
    │ 危险命令需审批                          │
    ├─ 入队 _ApprovalEntry ────────────────►│ notify_cb → 发消息带按钮
    │  thread.Event.wait()  阻塞             │
    │         ▲                              │
    │         │         用户 /approve         │
    │         └──────── resolve_gateway_approval()
    │  Event.set() → 继续执行或拒绝
    ▼
  terminal 执行
```

**要点：**

- 每个 `session_key` 独立 FIFO，避免并发审批串台。
- 超时未批 → 按 deny 处理（避免 agent 永久挂起）。
- Gateway 有两道消息守卫（`platforms/base.py` + `gateway/run.py`）；`/approve`、`/deny` 等必须 **绕过**「agent 忙碌」队列，否则死锁。

#### 6.4.5 审批模式与会话状态

**审批模式（`approvals.mode`）：**

| 模式 | 行为 |
|---|---|
| `manual` | 必须有人确认（CLI 输入 / Gateway 按钮 / ACP 面板） |
| `smart` | 辅助 LLM（task `"approval"`）输出 APPROVE / DENY / ESCALATE；`ESCALATE` 回落到 manual |
| `off` | 非 hardline 的危险模式可自动通过（等同 yolo 对危险检测层） |

**会话数据结构（线程安全）：**

| 结构 | 用途 |
|---|---|
| `_session_approved[session_key]` | 本会话已批准的 `pattern_key` 集合 |
| `_session_yolo` | 会话级 yolo |
| `_permanent_approved` | 持久化到 `config.yaml` `command_allowlist` |
| `_gateway_queues[session_key]` | Gateway 阻塞审批 FIFO |
| `_gateway_notify_cbs[session_key]` | 发 Slack/Telegram 等通知的回调 |

用户选择 **once / session / always / deny** 后，写入对应集合；`always` 进永久 allowlist。

#### 6.4.6 配置项

`hermes_cli/config.py`（节选）：

| 键 | 说明 |
|---|---|
| `approvals.mode` | `manual` / `smart` / `off` |
| `approvals.gateway_timeout` | Gateway 阻塞等待秒数 |
| `approvals.cron_mode` | `deny` / `approve` |
| `command_allowlist` | 永久批准的 pattern_key |
| `approvals.mcp_reload_confirm` | MCP 重载确认（文档注明会破坏 prompt cache） |

---

## 7. 循环退出与 `turn_exit_reason`

`run_conversation` 用 `_turn_exit_reason` 记录 turn 结束原因（写入返回 dict 的 `turn_exit_reason` 字段）。

| 值 | 典型场景 |
|---|---|
| `unknown` | 初始默认；部分 **early return** 路径未覆盖 |
| `interrupted_by_user` | 迭代开头 `_interrupt_requested` |
| `budget_exhausted` | `iteration_budget.consume()` 失败 |
| `interrupted_during_api_call` | API retry 循环后仍 marked interrupted |
| `all_retries_exhausted_no_response` | 重试耗尽仍无 response |
| `guardrail_halt` | 工具防护链判定 halt |
| `partial_stream_recovery` | 用流式 partial 文本收尾 |
| `fallback_prior_turn_content` | 空响应且上轮仅有 housekeeping 工具 |
| `empty_response_exhausted` | 空响应重试与 fallback 用尽 |
| `text_response(finish_reason=…)` | 正常无工具文本结束 |
| `error_near_max_iterations(…)` | 外层异常且接近上限 |
| `max_iterations_reached(api/max)` | post-loop `_handle_max_iterations` 路径 |

**诊断：** 若 `messages[-1].role == "tool"`，循环会以 WARNING 记录「agent stopped mid-work」（`conversation_loop.py` ~3874–3885）。

---

## 8. 预算耗尽后的真实路径（非宽限期）

### 8.1 `_budget_grace_call` 的现状

- 在 `agent/agent_init.py` 初始化为 `False`
- 循环内仅在 True 时跳过 `consume()` 并清标志
- **全仓库无 `_budget_grace_call = True` 赋值**（测试仅断言初始为 False）

因此：**不要**按「预算用尽后再给 loop 内一轮宽限」来讲述；该机制未接线。

### 8.2 实际行为：post-loop 总结调用

```3802:3819:agent/conversation_loop.py
    if final_response is None and (
        api_call_count >= agent.max_iterations
        or agent.iteration_budget.remaining <= 0
    ):
        _turn_exit_reason = f"max_iterations_reached({api_call_count}/{agent.max_iterations})"
        ...
        final_response = agent._handle_max_iterations(messages, api_call_count)
```

```910:919:agent/chat_completion_helpers.py
def handle_max_iterations(agent, messages, api_call_count):
    summary_request = (
        "You've reached the maximum number of tool-calling iterations allowed. "
        "Please provide a final response summarizing what you've found..."
    )
    messages.append({"role": "user", "content": summary_request})
    # 随后构建 api_messages，无 tools，单次 API 调用
```

| 对比 | 文档常写错的「宽限期」 | 源码实际 |
|---|---|---|
| 触发点 | loop 内再跑一轮带工具 | loop **已退出** |
| 机制 | `_budget_grace_call` | 追加 user 消息 + **无 tools** 的 summary API |
| 工具 | 可能继续调用 | 明确要求不再调用工具 |

**源码证据：**

```python
# agent/chat_completion_helpers.py:910-918
def handle_max_iterations(agent, messages: list, api_call_count: int) -> str:
    """Request a summary when max iterations are reached. Returns the final response text."""
    print(f"⚠️  Reached maximum iterations ({agent.max_iterations}). Requesting summary...")

    summary_request = (
        "You've reached the maximum number of tool-calling iterations allowed. "
        "Please provide a final response summarizing what you've found and accomplished so far, "
        "without calling any more tools."
    )
    messages.append({"role": "user", "content": summary_request})
```

---

## 9. 循环结束后的后处理

顺序要点（`conversation_loop.py` 3802–4095）：

| 步骤 | 说明 |
|---|---|
| `_handle_max_iterations` | 见 §8（仅当无 `final_response` 且预算/API 耗尽） |
| Kanban worker | `HERMES_KANBAN_TASK` 时可能代调 `kanban_block` |
| `_save_trajectory` | 可选轨迹落盘 |
| `_cleanup_task_resources` | 任务级 VM/browser 等清理 |
| `_drop_trailing_empty_response_scaffolding` | 去掉空响应脚手架再持久化 |
| `_persist_session` | JSON log + SQLite |
| Turn 诊断日志 | INFO；末条为 `tool` 时 WARNING |
| 文件变更 verifier | 可能追加到 `final_response` 脚注 |
| 插件 `transform_llm_output` / `post_llm_call` | 输出变换与钩子 |
| `_sync_external_memory_for_turn` | **用户中断则跳过** |
| `_spawn_background_review` | memory / skill 后台复盘 |
| `on_session_end` 插件 | 非 memory provider shutdown |

**注意：** `commit_memory_session` 主要在 **压缩切 session** 时调用，不是每轮正常结束都调用。

---

## 10. 与 `delegate_task` / 压缩 / 权限的交界

| 子系统 | 在主循环中的触点 |
|---|---|
| **delegate_task** | 作为 tool call 在 `_execute_tool_calls` 中同步执行；父 loop **阻塞等待**子 `run_conversation` 结束；详见 `hermes-agent-runtime-reader-guide.md` §1 |
| **压缩** | Preflight（loop 前）、回合后（工具批成功后）、API 错误路径；压缩成功可能 `continue` 且不消耗迭代计数 |
| **权限** | `terminal` 等工具在执行前通过 `check_all_command_guards()` 检测危险命令；按运行环境（CLI/Gateway/ACP）收集用户决策；写入 allowlist 后再执行 |
| **Prompt cache** | `pre_llm_call` 只改 user 消息；压缩会重建 system + 轮换 session → 前缀缓存失效 |

**权限在主循环中的位置：**

```text
while 循环
  │
  ├─ API 返回 tool_calls
  │
  ├─ _execute_tool_calls
  │   │
  │   ├─ terminal 工具？
  │   │   └─ check_all_command_guards()
  │   │       ├─ 检测危险命令
  │   │       ├─ 检查 hardline
  │   │       ├─ 检查会话 allowlist
  │   │       └─ 需要审批？
  │   │           ├─ CLI: prompt_dangerous_approval（同步）
  │   │           ├─ Gateway: 入队 + Event.wait（阻塞）
  │   │           └─ ACP: make_approval_callback（阻塞）
  │   │
  │   └─ 执行工具
  │
  └─ 继续下一轮迭代
```

---

## 11. 常见问题深度解答（FAQ）

### Q1: max_iterations 和 iteration_budget 有什么区别？

**A: 两个互补的控制机制。**

| 对比项 | api_call_count | iteration_budget |
|--------|----------------|------------------|
| **用途** | 统计实际API调用次数 | 控制可用迭代预算 |
| **增减** | 只增不减 | 可消耗可退还 |
| **上限** | max_iterations（默认90） | max_iterations（默认90） |
| **检查点** | 循环条件 | 循环条件 + consume() |

**源码证据：**

```python
# agent/conversation_loop.py:598
while (api_call_count < agent.max_iterations and agent.iteration_budget.remaining > 0) or agent._budget_grace_call:
```

**为什么需要两个？**

- `api_call_count` 提供准确的日志信息
- `iteration_budget` 支持 execute_code 的退还机制

### Q2: 工具是如何并行执行的？

**A: 使用 ThreadPoolExecutor，最多8个工作线程。**

**并行条件：**

- 模型返回多个独立的 tool_calls
- 工具之间没有数据依赖
- 工具不在阻塞列表中（如 clarify、delegate_task）

### Q3: 预算耗尽后会怎样？

**A: 模型会收到一个总结请求，要求它不调用工具直接给出摘要响应。**

这不是"额外的一轮迭代"，而是一个特殊的无工具API调用。

### Q4: yolo 模式是否意味着没有任何限制？

**A: 不，hardline 模式下的命令仍然会被拦截。**

| 误读 | 事实 |
|---|---|
| yolo = 无任何限制 | hardline 仍拦截绝对危险的命令（如 `rm -rf /`） |

`approvals` 设为 `off` 或会话级 yolo 只能绕过危险命令检测层，无法绕过 `HARDLINE_PATTERNS`。

### Q5: Gateway 审批会阻塞 agent 吗？

**A: 会，agent 线程会阻塞等待用户审批。**

Gateway 使用 FIFO 队列 + `threading.Event` 机制，agent 线程在 `Event.wait()` 处阻塞，直到用户 `/approve` 或 `/deny`，或超时。这确保了危险命令不会在用户不知情的情况下执行。

---

## 12. 讲者展开点与常见误读

### 讲者展开点

- 先画 **一轮 turn** 时间线，再 zoom 进 **单次 while 迭代**。
- 强调 **双计数器**：`execute_code` refund、压缩 restart 会让两者不同步。
- 工具并行用 **白名单 + 反例**（`delegate_task`、`clarify`）讲清边界。
- 预算耗尽务必讲 **`_handle_max_iterations`**，不要讲未接线的 grace flag。
- 权限系统用 **三层检测 + 三条路径** 讲解：
  - 检测层：hardline vs dangerous vs yolo
  - 交互层：CLI 同步 prompt、Gateway 阻塞队列、ACP JSON-RPC
  - 会话层：once / session / always / deny

### 常见误读（应对照源码纠正）

| 误读 | 事实 |
|---|---|
| 主循环在 `run_agent.py` | 在 `agent/conversation_loop.py` |
| 预算用尽 = 直接返回错误 | 多数情况还有一次无工具总结 API |
| `_budget_grace_call` 会再跑一轮 | 标志从未置 True |
| 一批 tool calls 总是并行 | 严格 `_should_parallelize_tool_batch` |
| `iteration_budget` 有 `refund(amount)` | 仅 `refund()` 退 1，且无参数 |
| 每轮结束都 `commit_memory_session` | 主要在压缩换 session 时 |
| yolo = 无任何限制 | hardline 仍拦截 |
| ACP 与 CLI 共用同一 UI 代码 | ACP 走 `request_permission`；CLI 走 prompt_toolkit 等 |
| `write_file` 走 approval.py 正则 | 编辑走 `edit_approval.py`（ACP）；CLI 另有路径 |
| Gateway 审批在 agent 同线程弹窗 | 阻塞的是 **wait**，UI 在平台 adapter |
| 子 Agent 默认同主 Agent stdin 审批 | 常配置 auto_deny，防死锁 |

---

## 源码追踪路径

如果你想深入理解主循环的实现，按以下顺序阅读：

```
1. agent/conversation_loop.py
   └─ run_conversation() 主函数
   └─ while循环（598行开始）
   └─ API调用构建（710行开始）
   └─ 工具调用处理（3106行开始）

2. agent/tool_executor.py
   └─ execute_tool_calls_concurrent() 并行执行
   └─ execute_tool_calls_sequential() 顺序执行

3. agent/iteration_budget.py
   └─ IterationBudget 类
   └─ consume() / refund() 方法

4. agent/chat_completion_helpers.py
   └─ handle_max_iterations() 预算耗尽处理

5. tools/approval.py
   └─ detect_dangerous_command() 危险命令检测
   └─ check_all_command_guards() 统一入口
   └─ register_gateway_notify() Gateway 队列
   └─ resolve_gateway_approval() 审批解析

6. acp_adapter/permissions.py
   └─ make_approval_callback() ACP 桥接

7. acp_adapter/edit_approval.py
   └─ maybe_require_edit_approval() 文件编辑审批
```

---

## 配置参数说明

### 迭代相关配置

```yaml
# config.yaml
max_iterations: 90  # 主Agent最大迭代次数

delegation:
  max_iterations: 50  # 子Agent最大迭代次数
  max_concurrent_children: 3  # 最大并发子Agent数
```

### 压缩相关配置

```yaml
compression:
  threshold: 0.75  # 触发压缩的上下文使用率
  protect_first_n: 3  # 头部保护消息数
  protect_last_n: 20  # 尾部保护消息数
```

### 权限相关配置

```yaml
approvals:
  mode: manual  # manual / smart / off
  gateway_timeout: 300  # Gateway 阻塞等待秒数
  cron_mode: deny  # cron 任务权限模式：deny / approve
  mcp_reload_confirm: true  # MCP 重载确认

command_allowlist: []  # 永远批准的命令模式
```

---

## 总结

### 核心设计理念

**Hermes 主循环的三大支柱：**

1. **双计数器机制**：精确控制资源消耗
2. **并行工具执行**：提高效率
3. **智能错误处理**：自动恢复和降级

**设计哲学：**

| 原则 | 体现 |
|------|------|
| 可靠性 | 多重错误处理和重试 |
| 效率 | 并行工具执行 |
| 可控性 | 双计数器机制 |
| 安全性 | 权限确认三层防护 |
| 可扩展性 | 插件钩子系统 |

### 最佳实践

**配置建议：**

| 场景 | max_iterations | approvals.mode |
|------|----------------|----------------|
| 快速原型 | 30-50 | off |
| 标准开发 | 90 | manual |
| 生产环境 | 90+ | smart |
| CI/CD | 150 | off + cron_mode: approve |

---

## 附录：术语表

| 术语 | 解释 |
|------|------|
| **主循环** | run_conversation 函数的核心 while 循环 |
| **API调用** | 向大模型发起的请求 |
| **工具调用** | 模型决定使用某个工具的行为 |
| **迭代** | 一次完整的"模型调用→工具执行"周期 |
| **双计数器** | api_call_count 和 iteration_budget |
| **并行执行** | 多个工具同时运行 |
| **压缩** | 将旧对话摘要以节省token |
| **危险命令** | 可能造成系统损坏的 shell 命令 |
| **Hardline** | 绝对禁止的命令模式，任何情况下都不能绕过 |
| **Allowlist** | 用户批准的命令模式列表 |
| **YOLO** | "You Only Live Once" 模式，跳过大部分权限检查 |

---

## 相关文档

- [Hermes-Agent Runtime 结构化导读](./hermes-agent-runtime-reader-guide.md) — `delegate_task`、记忆与技能进化
- [上下文压缩](./hermes-agent-context-compression.md)
- [Skill 系统](./hermes-agent-skill-system.md)

---

**文档版本：** v1.1
**最后更新：** 2025-05-19
**源码验证：** ✅ 已通过真实源码验证
