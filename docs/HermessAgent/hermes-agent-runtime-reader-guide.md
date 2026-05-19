# Hermes-Agent Runtime 结构化导读

> 面向听众/读者的结构化材料。本文不是逐字讲稿，而是帮助读者先建立地图；讲者可以结合 `hermes-agent-runtime-sharing.md` 展开细节。

---

## 1. 多 Agent：`delegate_task` 如何接入运行时

### 一眼看懂

`delegate_task` 是 Hermes-Agent 的多 Agent 入口，但它不是后台任务系统，也不是普通并行工具调用。

| 结论 | 含义 |
|---|---|
| 它首先是一个 tool | 模型通过 function calling 主动选择 `delegate_task` |
| 它由父 loop 特判执行 | 执行时需要 `parent_agent`，不走普通 `registry.dispatch` |
| 它会创建子 `AIAgent` | 子 Agent 有自己的 prompt、tools、session、task id、模型调用循环 |
| 父 Agent 只拿汇总结果 | 子 Agent 中间工具调用不直接进入父上下文 |
| 控制权最终回到父模型 | 子结果作为 `tool` message 回填，父模型下一轮继续推理 |

一句话：

```text
delegate_task = 父 Agent loop 里的同步 tool call + 子 Agent 独立 loop + 结果汇总回填。
```

---

### 总体结构

```text
Parent Model
  │
  │ tool_call: delegate_task(...)
  ▼
Parent AIAgent Loop
  │  校验 tool name / JSON / fan-out 上限 / 去重
  ▼
Tool Executor
  │  agent._dispatch_delegate_task(args)
  ▼
delegate_task Runtime
  │  创建 1..N 个 Child AIAgent
  ▼
Child AIAgent Loop(s)
  │  独立调用模型、执行工具、生成 final_response
  ▼
delegate_task JSON result
  │
  ▼
Parent messages.append(role="tool", tool_call_id=...)
  │
  ▼
Parent Model 下一轮读取子 Agent 汇总结果
```

---

### 源码地图

| 关注点 | 关键位置 | 读源码时看什么 |
|---|---|---|
| 工具集暴露 | `toolsets.py` | `delegation -> ["delegate_task"]` |
| 工具 schema 与注册 | `tools/delegate_tool.py` | `DELEGATE_TASK_SCHEMA`、`registry.register(...)`、`dynamic_schema_overrides` |
| agent 级工具声明 | `model_tools.py` | `_AGENT_LOOP_TOOLS` 包含 `delegate_task` |
| 父 loop 校验 | `agent/conversation_loop.py` | tool name 修复、JSON 校验、`_cap_delegate_task_calls`、`_deduplicate_tool_calls` |
| 工具执行分发 | `agent/tool_executor.py` | `function_name == "delegate_task"` 分支 |
| 统一派发入口 | `run_agent.py` | `AIAgent._dispatch_delegate_task(...)` 注入 `parent_agent=self` |
| 子 Agent 构造 | `tools/delegate_tool.py` | `_build_child_agent(...)` |
| 子 Agent 执行 | `tools/delegate_tool.py` | `_run_single_child(...)` 调 `child.run_conversation(...)` |
| 并行边界 | `agent/tool_dispatch_helpers.py` | `_should_parallelize_tool_batch(...)` 不把 `delegate_task` 当普通并行工具 |

---

### 一次调用的生命周期

| 阶段 | 发生什么 | 结果进入哪里 |
|---|---|---|
| 1. 模型选择 | 父模型返回 `delegate_task` tool call | `assistant_message.tool_calls` |
| 2. 父 loop 校验 | 校验工具名、JSON 参数、委派数量、重复调用 | 父 loop 内部状态 |
| 3. 父工具执行 | `agent._dispatch_delegate_task(function_args)` | `tools.delegate_tool.delegate_task(...)` |
| 4. 子 Agent 构造 | 根据 goal/context/toolsets/role 创建子 `AIAgent` | 子 Agent 独立运行时 |
| 5. 子 loop 运行 | 子 Agent 调模型、执行工具、继续 loop | 子 Agent messages/session |
| 6. 结果聚合 | 子 Agent final response 被整理成 `results[]` | `delegate_task` JSON 字符串 |
| 7. 回填父 loop | 父工具执行器 append `role="tool"` 消息 | 父 Agent messages |
| 8. 父模型继续 | 下一轮父模型读取子 Agent 汇总结果 | 新一轮模型请求 |

---

### 子 Agent 被隔离在哪里

`delegate_task` 创建的子 Agent 不是父 Agent 的“线程副本”，而是一个收窄后的新运行时。

| 维度 | 子 Agent 行为 |
|---|---|
| 会话历史 | 不继承父 messages |
| System Prompt | 使用 `ephemeral_system_prompt`，由 goal/context/workspace/role 生成 |
| User Message | 以 delegated `goal` 作为子 Agent 当前任务 |
| Context Files | `skip_context_files=True` |
| Memory | `skip_memory=True`，leaf 默认不能写共享 memory |
| Toolsets | 默认继承父可用工具集；显式传入时与父 toolsets 求交 |
| 禁用工具 | leaf 不能用 `delegate_task`、`clarify`、`memory`、`send_message`、`execute_code` |
| Session | 共用父 `session_db`，但 `parent_session_id` 指向父 session |
| Task ID | 使用稳定 `subagent_id`，隔离 terminal/file_state |
| Iteration Budget | `iteration_budget=None`，子 Agent 用自己的 `max_iterations` |
| Model/Provider | 默认继承父配置，也可通过 delegation 配置覆盖 |
| Progress | 子 Agent progress callback 被转发给父显示层 / Gateway / TUI |

---

### 父 loop 与子 loop 的关系

```text
父 loop:
  负责决定是否调用 delegate_task
  负责校验 tool call
  负责等待 delegate_task 返回
  负责把结果回填成 tool message
  负责下一轮综合与最终回答

子 loop:
  负责完成被委派的 goal
  负责独立调用大模型
  负责执行自己可用的工具
  负责产出 final_response
  不负责直接回答最终用户
```

关键边界：

```text
父 Agent 保留决策权；
子 Agent 只负责隔离探索或执行；
子 Agent 的中间轨迹不会直接污染父上下文；
父模型必须基于 tool result 再综合。
```

---

### 多 Agent vs 普通并行工具调用

| 对比项 | 普通 concurrent tool calls | `delegate_task` 多 Agent |
|---|---|---|
| 并行对象 | 同一轮里的多个工具函数 | 多个子 `AIAgent` loop |
| 触发方式 | 父模型一次返回多个 tool calls | 父模型调用 `delegate_task`，可传 `tasks[]` |
| 上下文 | 共享父 Agent 当前 messages | 每个子 Agent 有独立 messages/session |
| 模型调用 | 不新增独立 Agent loop | 每个子 Agent 独立调用模型 |
| 结果回填 | 每个工具结果按原 tool_call_id 回填 | 一个 `delegate_task` tool result，包含 `results[]` |
| 适用场景 | 多个安全、独立、低状态工具调用 | 研究、调试、代码审查、并行探索等 reasoning-heavy 子任务 |
| 重要限制 | 只读白名单、路径不冲突或 MCP 显式允许 | 父 loop 同步等待；不是 durable 后台任务 |

容易误读的一点：

```text
delegate_task 本身正常不作为普通并行 tool batch 执行；
它的并行主要发生在 delegate_task(tasks=[...]) 内部。
```

---

### 结果长什么样

子 Agent 完成后，`delegate_task` 返回给父 loop 的不是完整 transcript，而是聚合 JSON。

```json
{
  "results": [
    {
      "task_index": 0,
      "status": "completed",
      "summary": "子 Agent 的最终总结",
      "api_calls": 3,
      "duration_seconds": 12.4,
      "model": "child-model",
      "exit_reason": "completed",
      "tokens": {
        "input": 12000,
        "output": 900
      },
      "tool_trace": [
        {
          "tool": "read_file",
          "args_bytes": 128,
          "result_bytes": 4096,
          "status": "ok"
        }
      ]
    }
  ],
  "total_duration_seconds": 12.6
}
```

父 loop 随后写入：

```json
{
  "role": "tool",
  "name": "delegate_task",
  "content": "{...results...}",
  "tool_call_id": "call_xxx"
}
```

下一轮父模型读取这个 `tool` message，再决定继续验证、继续派生，还是输出最终答复。

---

### 运行时收口能力

| 能力 | 为什么需要 |
|---|---|
| 深度限制 | 防止 Agent 树无限递归 |
| 并发上限 | 防止一次 fan-out 过多子 Agent |
| spawn pause | TUI/RPC 可暂停新建子 Agent |
| active registry | 让 TUI 能展示、定位、打断具体 subagent |
| interrupt propagation | 父 Agent 被打断时，子 Agent 能停在迭代边界 |
| child timeout | 子 Agent 卡死时，父 loop 不无限等待 |
| heartbeat | 子 Agent 长时间工作时，避免 Gateway 误判空闲 |
| cost rollup | 子 Agent token/cost 汇总回父 session |
| file-state reminder | 子 Agent 修改父已读文件时，提醒父 Agent 重新读取 |
| subagent hooks | 插件可观察子 Agent 生命周期 |

---

### 讲者展开点

- 讲 `delegate_task` 时，先强调它是“任务级派生”，不是“动作级并行工具调用”。
- 讲父子关系时，重点放在“父 Agent 保留决策权，子 Agent 只返回 summary”。
- 讲源码时，可以按 `conversation_loop.py -> tool_executor.py -> run_agent.py -> delegate_tool.py -> child.run_conversation()` 这一条线展开。
- 讲可靠性时，可以把 depth limit、timeout、interrupt、heartbeat、cost rollup 作为多 Agent 不失控的关键防线。

---

## 2. 自我学习进化：Hermes 如何把经验留到下一次

### 一眼看懂

Hermes-Agent 的“学习”不是训练模型，也不是更新模型参数。它把经验写到运行时外部资产里，让下一次会话、下一次任务、下一个 Agent loop 能重新拿到这些经验。

| 学习层 | 保存什么 | 下次如何使用 |
|---|---|---|
| Declarative Memory | 用户偏好、环境事实、项目约定 | 作为 memory snapshot 进入 system prompt，或由外部 provider 召回 |
| Episodic Memory | 过去会话、工具调用、最终结果 | 通过 `session_search` 搜索和定位历史消息 |
| Procedural Memory | 做某类任务的方法、步骤、坑点 | 通过 skills 被列出、查看、更新和复用 |
| Background Review | 回合结束后的经验复盘 | fork review agent 写 memory / skills |
| External Memory | Honcho、Hindsight、Mem0 等外部记忆后端 | turn start prefetch、turn end sync、delegation observation |

一句话：

```text
Hermes 的进化 = 把一次任务中的事实、偏好、流程和结果，沉淀成下一次 loop 可读的上下文与工具资产。
```

---

### 总体结构

```text
User Turn
  │
  ▼
AIAgent Loop
  │
  ├─ 读：System Prompt memory snapshot
  ├─ 读：External memory prefetch 注入当前 user message
  ├─ 读：skills_list / skill_view 获取流程知识
  ├─ 读：session_search 回看历史会话
  │
  ├─ 写：memory tool 更新 MEMORY.md / USER.md
  ├─ 写：skill_manage 创建或修补 SKILL.md
  └─ 写：SessionDB 持久化本轮 messages
  │
  ▼
Turn End
  │
  ├─ external memory sync_all + queue_prefetch_all
  └─ background review fork 复盘 memory / skills
```

---

### 源码地图

| 关注点 | 关键位置 | 读源码时看什么 |
|---|---|---|
| 内建长期记忆 | `tools/memory_tool.py` | `MemoryStore`、`MEMORY.md`、`USER.md`、frozen snapshot |
| 内建记忆加载 | `agent/agent_init.py` | 根据配置启用 memory / user profile 并加载 `MemoryStore` |
| System Prompt 注入 | `agent/system_prompt.py` | `format_for_system_prompt("memory")` / `format_for_system_prompt("user")` |
| 外部记忆接口 | `agent/memory_provider.py` | `MemoryProvider` lifecycle 与 optional hooks |
| 外部记忆编排 | `agent/memory_manager.py` | `prefetch_all`、`sync_all`、`queue_prefetch_all`、`on_delegation` |
| turn 内召回 | `agent/conversation_loop.py` | `on_turn_start`、`prefetch_all`、`build_memory_context_block` |
| turn 后同步 | `run_agent.py` | `_sync_external_memory_for_turn(...)` |
| 历史会话存储 | `hermes_state.py` | `SessionDB`、sessions/messages 表、FTS5 / trigram 索引 |
| 历史会话回忆 | `tools/session_search_tool.py` | discover / scroll / browse 三种查询形态 |
| 技能查看 | `tools/skills_tool.py` | `skills_list`、`skill_view`、progressive disclosure |
| 技能演化 | `tools/skill_manager_tool.py` | `skill_manage` 的 create / patch / edit / write_file |
| 后台复盘 | `agent/background_review.py` | fork review agent，只允许 memory / skill tools |

---

### 四类“学习资产”

| 资产 | 形态 | 适合保存 | 不适合保存 |
|---|---|---|---|
| `USER.md` | 用户画像 | 用户偏好、沟通方式、稳定习惯 | 临时任务状态 |
| `MEMORY.md` | Agent 笔记 | 环境事实、项目约定、工具怪癖 | 大段日志、一次性结果 |
| SessionDB | 会话 transcript | 过去做过什么、工具结果、最终答复 | 需要每轮都自动注入的偏好 |
| Skills | `SKILL.md` + references/templates/scripts | 可复用流程、操作步骤、踩坑经验 | 只对今天这次任务成立的细节 |

---

### 内建 Memory：把事实写进未来 prompt

内建 memory 由 `tools/memory_tool.py` 提供，核心是两个文件：

```text
~/.hermes/memories/MEMORY.md
~/.hermes/memories/USER.md
```

| 机制 | 作用 |
|---|---|
| `memory(action="add")` | 新增一条长期记忆 |
| `memory(action="replace")` | 用短唯一文本定位并更新旧记忆 |
| `memory(action="remove")` | 删除不再成立的记忆 |
| content scan | 阻止明显 prompt injection / secret exfiltration 内容进入 memory |
| char limit | 控制 `MEMORY.md` / `USER.md` 注入 prompt 的体积 |
| frozen snapshot | session 启动时读取快照，中途写盘但不改当前 system prompt |

关键边界：

```text
memory 写入是 durable 的；
但当前 session 的 system prompt 不会立刻重渲染。
下一次 session、prompt invalidation 或 compression 边界后，新的 memory 才进入 prompt snapshot。
```

这样做的目的不是偷懒，而是保持 system prompt prefix 稳定，减少 provider prompt cache 失效。

---

### External Memory：把召回和同步交给 provider

外部记忆通过 `MemoryProvider` 抽象接入，`MemoryManager` 统一调度。它允许 Hermes 使用 Honcho、Hindsight、Mem0 等后端，但运行时只允许一个 external provider，避免工具 schema 膨胀和多个记忆后端互相打架。

| 时机 | 调用 | 作用 |
|---|---|---|
| Agent init | `initialize_all(session_id, ...)` | 初始化 provider、建立 session 作用域 |
| System Prompt 构建 | `build_system_prompt()` | 注入 provider 的静态说明或状态 |
| Turn start | `on_turn_start(...)` | 告诉 provider 新回合开始 |
| 模型请求前 | `prefetch_all(user_message)` | 召回相关长期上下文 |
| API-call-time | `build_memory_context_block(...)` | 把召回结果包进 `<memory-context>` 注入当前 user message |
| Turn end | `sync_all(user, assistant)` | 把完成的用户/助手 exchange 同步到 provider |
| Turn end | `queue_prefetch_all(user)` | 预热下一轮召回 |
| Memory tool 写入 | `on_memory_write(...)` | 外部 provider 镜像内建 memory 写入 |
| Subagent 完成 | `on_delegation(...)` | 记录父 Agent 观察到的子 Agent 结果 |
| 压缩前 | `on_pre_compress(messages)` | 在旧消息被压缩前提取 provider 关心的信息 |

关键边界：

```text
external memory prefetch 不写入 session DB；
它只在当前 API request 的 user message 上临时注入。
```

---

### Session Search：从“过去做过什么”里学习

`session_search` 不是总结器，而是对 SQLite SessionDB 的检索工具。它让 Agent 在需要时回看过去会话，而不是把所有历史都塞进 prompt。

| 模式 | 怎么触发 | 用途 |
|---|---|---|
| Browse | 不传参数 | 看最近会话列表 |
| Discover | 传 `query` | FTS5 搜索历史消息，返回命中窗口和 session bookends |
| Scroll | 传 `session_id + around_message_id` | 围绕某条历史消息继续向前/向后看 |

适合用来回答：

```text
之前怎么修过类似问题？
上次这个项目的约定是什么？
某个工具报错当时是怎么解决的？
用户以前让我不要怎么做？
```

---

### Skills：把“怎么做”变成可复用流程

Skills 是 Hermes 的 procedural memory。Memory 记录事实，Skills 记录做事方法。

| 工具 | 作用 |
|---|---|
| `skills_list` | 低成本列出技能元数据 |
| `skill_view` | 按需加载某个 `SKILL.md` 或支持文件 |
| `skill_manage(create)` | 创建新的技能 |
| `skill_manage(patch)` | 精准修补已有技能 |
| `skill_manage(edit)` | 重写已有技能 |
| `skill_manage(write_file)` | 给技能补充 references / templates / scripts / assets |
| `skill_manage(delete)` | 删除不再需要的用户技能 |

技能目录形态：

```text
~/.hermes/skills/
  skill-name/
    SKILL.md
    references/
    templates/
    scripts/
    assets/
```

什么时候应该沉淀成 skill：

| 信号 | 例子 |
|---|---|
| 用户纠正了做法 | “以后不要这样格式化”、“这种任务先查日志” |
| 出现可复用调试路径 | 某类 CI 失败、某类 provider 兼容问题 |
| 某个 skill 不完整 | 用 skill 时发现缺步骤或坑点 |
| 复杂任务跑通 | 形成稳定流程、命令、校验步骤 |

---

### Background Review：回合结束后的自我复盘

`agent/background_review.py` 体现了 Hermes 的“自我改进 loop”。它不是在主任务中打断用户，而是在 turn 完成后 fork 一个 review agent。

```text
主 Agent 完成答复
  -> 判断是否触发 memory / skill review
  -> fork review AIAgent
  -> 继承父模型/provider/credential/cache
  -> 限制工具白名单：memory + skills
  -> review conversation snapshot
  -> 写 MEMORY.md / USER.md 或 patch skills
  -> 输出简短 Self-improvement review 摘要
```

它的关键设计：

| 设计点 | 作用 |
|---|---|
| response 之后运行 | 不抢主任务上下文和注意力 |
| fork review agent | 用同一套 AIAgent loop 自我审视 |
| 继承 runtime | 复用父 provider、credential、prefix cache |
| `skip_memory=True` | 防止 review harness 污染 external memory provider |
| 工具白名单 | 只允许 memory / skill management，避免后台乱执行 |
| best-effort | review 失败不影响用户当前任务 |

---

### 学习闭环

```text
用户反馈 / 任务经验 / 工具结果
  │
  ├─ 稳定事实         -> memory(USER.md / MEMORY.md)
  ├─ 历史过程         -> SessionDB + session_search
  ├─ 可复用做法       -> skills(SKILL.md + supporting files)
  ├─ 外部长期召回     -> MemoryProvider
  └─ 回合后自动复盘   -> background review
      │
      ▼
下一次 AIAgent Loop
  │
  ├─ system prompt snapshot
  ├─ user message memory-context injection
  ├─ skill_view loaded procedures
  └─ session_search retrieved history
```

---

### 边界：它不会做什么

| 不会做 | 正确理解 |
|---|---|
| 不会更新大模型权重 | 所有学习都发生在外部状态、文档、工具和检索层 |
| 不会把所有历史自动塞进 prompt | 历史主要进 SessionDB，需要 `session_search` 按需召回 |
| 不会让 memory 立即改写当前 system prompt | 内建 memory 当前 session 写盘，但 prompt snapshot 保持稳定 |
| 不应把 transient failure 写成长期规则 | 缺依赖、临时网络失败、未配置凭证不该变成“永远不能用” |
| 不应把一次性任务写成 skill | skill 应该是 class-level、可复用流程 |

---

### 讲者展开点

- 先讲边界：Hermes 的学习是“运行时外部记忆与流程沉淀”，不是 fine-tuning。
- 再讲四层资产：`USER.md/MEMORY.md`、SessionDB、Skills、External Memory。
- 讲 prompt cache 时，强调 frozen snapshot：写 memory 不等于当前 system prompt 立刻变化。
- 讲 skill 时，把它和 memory 区分开：memory 是“知道什么”，skill 是“怎么做”。
- 讲 background review 时，强调它是 response 后的 fork agent，且只能用 memory/skill 工具。
