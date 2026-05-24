# Background Review 机制解析

## 概述

Background Review 是 Hermes Agent 的自我改进循环（Self-Improvement Loop），在每次对话回合结束后，自动在后台评估是否需要保存 memory 或更新 skill。这个机制让 Agent 能够从每次交互中学习，持续优化自己的知识库和工作流程。

## 核心设计理念

- **非阻塞**：在后台线程运行，不影响用户体验
- **独立上下文**：Fork 一个新的 Agent 实例，使用独立的工具白名单
- **缓存复用**：继承父 Agent 的系统提示缓存，降低 API 成本
- **最佳努力**：失败不影响主流程，静默降级

---

## 架构组成

### 1. 触发器（Trigger）

#### Memory Review 触发器
- **触发条件**：基于对话回合数（turn-based）
- **计数器**：`agent._turns_since_memory`
- **阈值**：`agent._memory_nudge_interval`（默认 10 回合）
- **检查时机**：每次 `run_conversation` 开始时
- **重置时机**：触发 review 后归零

```python
# agent/conversation_loop.py:~3900
_should_review_memory = False
if (agent._memory_nudge_interval > 0
        and "memory" in agent.valid_tool_names
        and agent._memory_store):
    agent._turns_since_memory += 1
    if agent._turns_since_memory >= agent._memory_nudge_interval:
        _should_review_memory = True
```

#### Skill Review 触发器
- **触发条件**：基于工具调用迭代次数（iteration-based）
- **计数器**：`agent._iters_since_skill`
- **阈值**：`agent._skill_nudge_interval`（默认 10 次迭代）
- **检查时机**：每次对话回合结束时
- **重置时机**：触发 review 后归零

```python
# agent/conversation_loop.py:~4040
_should_review_skills = False
if (agent._skill_nudge_interval > 0
        and agent._iters_since_skill >= agent._skill_nudge_interval
        and "skill_manage" in agent.valid_tool_names):
    _should_review_skills = True
    agent._iters_since_skill = 0
```

**关键差异**：
- Memory 按**对话回合**计数（用户提问 → Agent 完整响应 = 1 回合）
- Skill 按**工具迭代**计数（每次 API 调用 + 工具执行 = 1 次迭代）

---

### 2. 启动器（Spawner）

位置：`agent/conversation_loop.py:~4055`

```python
if final_response and not interrupted and (_should_review_memory or _should_review_skills):
    try:
        agent._spawn_background_review(
            messages_snapshot=list(messages),
            review_memory=_should_review_memory,
            review_skills=_should_review_skills,
        )
    except Exception:
        pass  # Background review is best-effort
```

**启动条件**：
1. 有最终响应（`final_response` 存在）
2. 未被中断（`not interrupted`）
3. 至少一个触发器激活

**传递数据**：
- `messages_snapshot`：当前对话历史的快照（深拷贝）
- `review_memory`：是否需要 review memory
- `review_skills`：是否需要 review skills

---

### 3. Review Prompt 生成器

位置：`agent/background_review.py:34-227`

根据触发器组合选择不同的 prompt：

| 触发器组合 | Prompt 常量 | 长度 |
|-----------|------------|------|
| Memory only | `_MEMORY_REVIEW_PROMPT` | ~42 行 |
| Skill only | `_SKILL_REVIEW_PROMPT` | ~144 行 |
| Both | `_COMBINED_REVIEW_PROMPT` | ~227 行 |

**Prompt 核心指令**：

#### Memory Review Prompt
```
关注两个维度：
1. 用户透露的个人信息（角色、偏好、期望）
2. 用户对 Agent 行为的期望（工作风格、操作方式）

如果没有值得保存的内容，回复 "Nothing to save." 并停止。
```

#### Skill Review Prompt
```
目标：维护 CLASS-LEVEL 技能库，而非平铺的一次性技能列表

触发信号（任一即可）：
• 用户纠正了风格/格式/流程
• 出现了非平凡的技术/修复/调试路径
• 已加载的 skill 被证明错误/过时

优先级顺序：
1. 更新当前已加载的 skill
2. 更新现有的伞状 skill
3. 在现有 skill 下添加支持文件（references/templates/scripts）
4. 创建新的 CLASS-LEVEL 伞状 skill

禁止捕获：
• 环境依赖的失败（缺少二进制、未配置凭证）
• 工具的负面断言（"X 工具不工作"）
• 会话特定的瞬态错误
```

---

### 4. Review Agent（Fork）

位置：`agent/background_review.py:_run_review_in_thread`

#### 4.1 Fork 配置

```python
review_agent = AIAgent(
    model=agent.model,
    max_iterations=16,
    quiet_mode=True,
    platform=agent.platform,
    provider=agent.provider,
    api_mode=_parent_api_mode,  # codex_app_server → codex_responses
    base_url=_parent_runtime.get("base_url"),
    api_key=_parent_runtime.get("api_key"),
    credential_pool=getattr(agent, "_credential_pool", None),
    parent_session_id=agent.session_id,
    enabled_toolsets=getattr(agent, "enabled_toolsets", None),
    disabled_toolsets=getattr(agent, "disabled_toolsets", None),
    skip_memory=True,  # 关键：不触碰外部 memory 插件
)
```

**关键继承**：
- `_memory_store`：指向父 Agent 的 memory 存储（MEMORY.md/USER.md）
- `_cached_system_prompt`：复用父 Agent 的系统提示（命中 prefix cache）
- `session_id`：使用父 Agent 的 session_id（保证缓存键一致）

#### 4.2 工具白名单

```python
review_whitelist = {
    t["function"]["name"]
    for t in get_tool_definitions(
        enabled_toolsets=["memory", "skills"],
        quiet_mode=True,
    )
}
set_thread_tool_whitelist(
    review_whitelist,
    deny_msg_fmt="Background review denied non-whitelisted tool: {tool_name}"
)
```

**允许的工具**：
- `memory`：add/replace/remove/read 操作
- `skill_manage`：create/edit/patch/delete/write_file/remove_file 操作

**拒绝的工具**：
- 所有其他工具（terminal、web_search、file_read 等）

#### 4.3 安全防护

```python
def _bg_review_auto_deny(command, description, **kwargs):
    logger.warning(
        "Background review auto-denied dangerous command: %s (%s)",
        command, description,
    )
    return "deny"

_set_approval_callback(_bg_review_auto_deny)
```

**防护机制**：
- 安装非交互式 approval callback
- 任何危险命令自动拒绝（防止死锁）
- 清理时移除 callback（防止线程复用污染）

#### 4.4 静默执行

```python
with open(os.devnull, "w", encoding="utf-8") as _devnull, \
     contextlib.redirect_stdout(_devnull), \
     contextlib.redirect_stderr(_devnull):
    review_agent.run_conversation(
        user_message=prompt + "\n\nYou can only call memory and skill management tools...",
        conversation_history=messages_snapshot,
    )
```

**静默范围**：
- stdout/stderr 重定向到 `/dev/null`
- `suppress_status_output = True`（抑制状态消息）
- 只有最终的成功摘要会显示给用户

---

### 5. 动作摘要器（Action Summarizer）

位置：`agent/background_review.py:summarize_background_review_actions`

#### 工作流程

```
1. 扫描 review_agent._session_messages
2. 过滤出 role="tool" 且 success=True 的消息
3. 排除 prior_snapshot 中已存在的消息（防止重复）
4. 提取 message 字段，生成人类可读的摘要
```

#### 摘要格式

```python
# 输入（tool result）
{
    "success": True,
    "message": "Entry added",
    "target": "memory"
}

# 输出（用户可见）
"Memory updated"
```

#### 去重逻辑

```python
# 按 tool_call_id 匹配
if tcid and tcid in existing_tool_call_ids:
    continue

# 按 content 内容匹配（fallback）
if content_str in existing_tool_contents:
    continue
```

**为什么需要去重**：
- Review agent 继承了 `conversation_history`
- 如果不去重，会把父 Agent 的旧操作当作新操作展示
- 参见 issue #14944

---

### 6. 结果通知器（Notifier）

位置：`agent/background_review.py:506-518`

```python
if actions:
    summary = " · ".join(dict.fromkeys(actions))
    agent._safe_print(
        f"  💾 Self-improvement review: {summary}"
    )
    _bg_cb = agent.background_review_callback
    if _bg_cb:
        try:
            _bg_cb(f"💾 Self-improvement review: {summary}")
        except Exception:
            pass
```

**通知路径**：
1. **CLI 路径**：通过 `agent._safe_print` 输出到终端
2. **Gateway 路径**：通过 `agent.background_review_callback` 发送到 UI

**示例输出**：
```
💾 Self-improvement review: Memory updated · Skill "debugging-workflow" updated
```

**关键行为**：
- 如果 `actions` 为空（review agent 没有调用任何工具），**不显示任何通知**
- 用户不会看到"review 运行了但什么都没做"的消息
- 这是静默设计：只在有实际变更时才通知用户

---

## 写入决策机制

### 谁决定是否写入？

**答案：大模型（Review Agent）决定**

Background Review 机制本身**不保证**一定会写入文件。整个决策流程是：

```
1. 触发器激活（回合数/迭代数达到阈值）
   ↓
2. 启动 Review Agent（Fork 的 AIAgent）
   ↓
3. Review Agent 读取 review prompt + 对话历史
   ↓
4. 大模型分析对话内容
   ↓
5. 大模型决定：
   ├─ 有值得保存的内容 → 调用 memory/skill_manage 工具 → 写入文件
   └─ 没有值得保存的内容 → 回复 "Nothing to save." → 不调用工具 → 不写入
   ↓
6. 提取成功的工具调用
   ↓
7. 如果有工具调用 → 显示通知
   如果没有工具调用 → 静默结束（用户看不到任何提示）
```

### Prompt 中的决策指导

#### Memory Review Prompt

```
关注两个维度：
1. 用户透露的个人信息（角色、偏好、期望）
2. 用户对 Agent 行为的期望（工作风格、操作方式）

If something stands out, save it using the memory tool.
If nothing is worth saving, just say 'Nothing to save.' and stop.
```

**决策权**：完全交给大模型判断

#### Skill Review Prompt

```
Be ACTIVE — most sessions produce at least one skill update, even if small.
A pass that does nothing is a missed learning opportunity, not a neutral outcome.

'Nothing to save.' is a real option but should NOT be the default.
If the session ran smoothly with no corrections and produced no new technique,
just say 'Nothing to save.' and stop. Otherwise, act.
```

**决策倾向**：鼓励积极保存，但仍然允许"什么都不做"

### 什么时候会写入？

#### Memory 写入场景

✅ **会写入**：
- 用户透露了个人信息："我是一名数据科学家"
- 用户表达了偏好："我喜欢简洁的代码"
- 用户设定了期望："不要解释太多，直接给答案"
- 用户提到了工作习惯："我通常在周五部署"

❌ **不会写入**：
- 纯技术问答，没有个人信息
- 一次性任务："帮我分析这个 PR"
- 临时状态："我现在在调试"

#### Skill 写入场景

✅ **会写入**：
- 用户纠正了 Agent 的行为："别用 git add .，要指定文件"
- 出现了非平凡的技术路径："原来要先清理缓存才能重启"
- 已加载的 skill 被证明过时："这个 API 已经改了"
- 发现了可复用的模式："这类问题都要先检查日志"

❌ **不会写入**：
- 环境问题："command not found"（临时的，用户可以修复）
- 负面断言："X 工具不工作"（会变成永久的自我限制）
- 瞬态错误："API 超时了"（重试就好了）
- 一次性任务："总结今天的新闻"（不是可复用的技能）

### 实际行为示例

#### 场景 1：有内容保存

```
用户：记住我喜欢用 TypeScript，不要给我 JavaScript 代码
  ↓
Review Agent 分析：用户表达了明确的语言偏好
  ↓
Review Agent 调用：memory(action="add", target="user", content="偏好 TypeScript...")
  ↓
工具返回：{"success": true, "message": "Entry added", "target": "user"}
  ↓
用户看到：💾 Self-improvement review: User profile updated
```

#### 场景 2：没有内容保存

```
用户：今天天气怎么样？
  ↓
Review Agent 分析：纯信息查询，没有个人信息或技能信号
  ↓
Review Agent 回复："Nothing to save."
  ↓
没有工具调用
  ↓
用户看到：（什么都没有，静默结束）
```

#### 场景 3：Skill 积极保存

```
用户：别再用 console.log 了，用 logger.debug
  ↓
Review Agent 分析：用户纠正了代码风格，这是 FIRST-CLASS skill 信号
  ↓
Review Agent 调用：skill_manage(action="patch", skill_name="typescript-development", ...)
  ↓
工具返回：{"success": true, "message": "Skill 'typescript-development' updated"}
  ↓
用户看到：💾 Self-improvement review: Skill "typescript-development" updated
```

### 为什么这样设计？

#### 1. 避免噪音

如果每次 review 都显示"已检查，无需保存"，会产生大量无用通知：

```
❌ 糟糕的设计：
💾 Self-improvement review: No changes needed
💾 Self-improvement review: No changes needed
💾 Self-improvement review: No changes needed
（用户会忽略这些消息）

✅ 当前设计：
（静默）
（静默）
💾 Self-improvement review: Memory updated
（只在有实际变更时通知）
```

#### 2. 信任大模型的判断

- Review prompt 已经提供了详细的决策指导
- 大模型能够理解上下文，判断是否值得保存
- 人工规则无法覆盖所有边界情况

#### 3. 最佳努力原则

- Review 是增强功能，不是核心功能
- 即使大模型判断错误（该保存的没保存），也不影响主流程
- 用户可以通过显式命令（`/remember`）强制保存

### 调试：查看 Review Agent 的决策

如果想知道 Review Agent 为什么没有保存，可以：

```python
# 1. 启用 verbose 模式
export HERMES_VERBOSE=1

# 2. 查看 review agent 的完整对话
tail -f ~/.hermes/agent.log | grep -A 50 "Background review"

# 3. 手动触发 review（测试用）
agent._turns_since_memory = agent._memory_nudge_interval
# 下次对话后会触发，可以观察 review agent 的输出
```

---

## 执行时序图

```
用户提问
  ↓
run_conversation 开始
  ↓
检查 memory trigger（_turns_since_memory >= 10?）
  ↓
执行主对话循环（API 调用 + 工具执行）
  ├─ 每次迭代：_iters_since_skill += 1
  └─ 生成最终响应
  ↓
检查 skill trigger（_iters_since_skill >= 10?）
  ↓
返回响应给用户 ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
  ↓                                    │
启动 background review（如果触发）      │ 用户看到响应
  ├─ 创建 daemon 线程                   │ （无需等待 review）
  ├─ Fork AIAgent                      │
  ├─ 设置工具白名单                     │
  ├─ 运行 review prompt                │
  ├─ 写入 memory/skill 文件             │
  ├─ 生成动作摘要                       │
  └─ 通知用户（异步）← ─ ─ ─ ─ ─ ─ ─ ─ ┘
```

---

## 关键设计决策

### 1. 为什么 Memory 按回合计数，Skill 按迭代计数？

**Memory**：
- 关注用户的**长期偏好**和**个人信息**
- 这些信息通常在对话的**开头或结尾**透露
- 按回合计数更符合"每 N 次对话检查一次"的语义

**Skill**：
- 关注**技术路径**和**工作流程**
- 复杂任务可能在一个回合内有多次工具调用
- 按迭代计数能更准确地反映"工作量"

### 2. 为什么要 Fork 一个新的 Agent？

**隔离性**：
- 主 Agent 的工具集可能很大（terminal、web、file 等）
- Review 只需要 memory 和 skill 工具
- Fork 可以设置独立的工具白名单

**安全性**：
- 防止 review 过程中执行危险命令
- 防止 review 影响主对话的状态

**缓存复用**：
- 继承父 Agent 的 `_cached_system_prompt`
- 命中 Anthropic 的 prefix cache（节省 ~26% 成本）

### 3. 为什么要 `skip_memory=True`？

**问题**：
- 如果不设置，Fork 的 `AIAgent.__init__` 会重建 `_memory_manager`
- `_memory_manager` 会连接外部插件（Honcho、Mem0、Supermemory）
- Review 的 harness prompt 会被插件记录为真实对话

**后果**：
- 用户的 memory 命名空间被污染
- 外部插件存储了大量无用的 review 对话
- 参见代码注释：agent/background_review.py:381-393

**解决方案**：
- `skip_memory=True`：跳过外部插件初始化
- 手动绑定 `_memory_store`：指向父 Agent 的本地存储
- Review 写入的 memory 仍然落盘（MEMORY.md/USER.md）

### 4. 为什么要去重 tool messages？

**场景**：
```python
# 父 Agent 的对话历史
messages = [
    {"role": "user", "content": "记住我喜欢简洁的代码"},
    {"role": "assistant", "content": "好的", "tool_calls": [...]},
    {"role": "tool", "tool_call_id": "call_123", "content": '{"success": true, "message": "Memory updated"}'},
]

# Review agent 继承这个历史
review_agent.run_conversation(conversation_history=messages)

# 如果不去重，会把 call_123 当作新操作
```

**解决方案**：
- 记录 `prior_snapshot` 中所有 `tool_call_id`
- 扫描 review messages 时跳过已存在的 ID
- 参见 issue #14944

---

## 配置项

### Memory Nudge Interval

```python
# 默认值
agent._memory_nudge_interval = 10  # 每 10 回合触发一次

# 配置文件（~/.hermes/config.toml）
[memory]
nudge_interval = 5  # 自定义间隔
```

### Skill Nudge Interval

```python
# 默认值
agent._skill_nudge_interval = 10  # 每 10 次迭代触发一次

# 配置文件
[skills]
creation_nudge_interval = 15  # 自定义间隔
```

### 禁用 Background Review

```python
# 设置为 0 禁用
agent._memory_nudge_interval = 0
agent._skill_nudge_interval = 0
```

---

## 测试覆盖

### 核心测试文件

- `tests/run_agent/test_background_review.py`：基础功能测试
- `tests/run_agent/test_background_review_summary.py`：摘要生成测试
- `tests/run_agent/test_background_review_cache_parity.py`：缓存复用测试
- `tests/run_agent/test_background_review_toolset_restriction.py`：工具白名单测试

### 关键测试场景

#### 1. 清理顺序测试
```python
def test_background_review_shuts_down_memory_provider_before_close():
    # 验证 shutdown_memory_provider() 在 close() 之前调用
    assert events == [
        "init",
        "run_conversation",
        "shutdown_memory_provider",  # 必须在 close 之前
        "close",
    ]
```

#### 2. 安全回调测试
```python
def test_background_review_installs_auto_deny_approval_callback():
    # 验证 review 线程安装了非交互式 callback
    assert callable(observed["during_run"])
    assert observed["during_run"]("rm -rf /", "test") == "deny"
    assert observed["after_finally"] is None  # 清理后为空
```

#### 3. 外部插件隔离测试
```python
def test_background_review_fork_skips_external_memory_plugins():
    # 验证 skip_memory=True 被传递
    assert captured_kwargs.get("skip_memory") is True
```

---

## 性能优化

### 1. Prefix Cache 复用

**优化前**：
- Review agent 重建系统提示
- 每次 review 都是 cache miss
- 成本：~100% 系统提示 token

**优化后**：
```python
review_agent._cached_system_prompt = agent._cached_system_prompt
review_agent.session_start = agent.session_start
review_agent.session_id = agent.session_id
```
- 复用父 Agent 的缓存
- 命中率：~95%+
- 成本降低：~26%（参见 PR #17276）

### 2. 工具集配置复用

```python
review_agent = AIAgent(
    enabled_toolsets=getattr(agent, "enabled_toolsets", None),
    disabled_toolsets=getattr(agent, "disabled_toolsets", None),
)
```

**原因**：
- Anthropic 的 cache key 包含 `tools[]` 数组
- 必须保证 tools 定义字节级一致
- 运行时通过白名单限制实际可调用的工具

### 3. 静默输出

```python
with contextlib.redirect_stdout(_devnull), \
     contextlib.redirect_stderr(_devnull):
    review_agent.suppress_status_output = True
    review_agent.run_conversation(...)
```

**收益**：
- 避免 I/O 开销
- 避免日志写入
- 只保留最终摘要

---

## 故障处理

### 1. Review 失败不影响主流程

```python
try:
    agent._spawn_background_review(...)
except Exception:
    pass  # Background review is best-effort
```

**设计原则**：
- Review 是增强功能，不是核心功能
- 失败时静默降级
- 不阻塞用户的下一次交互

### 2. 危险命令自动拒绝

```python
def _bg_review_auto_deny(command, description, **kwargs):
    logger.warning("Background review auto-denied dangerous command: %s", command)
    return "deny"
```

**防护场景**：
- Review agent 尝试执行 shell 命令
- 尝试调用非白名单工具
- 触发 approval guard

### 3. 线程清理

```python
finally:
    try:
        review_agent.shutdown_memory_provider()
    except Exception:
        pass
    try:
        review_agent.close()
    except Exception:
        pass
    try:
        _set_approval_callback(None)
    except Exception:
        pass
```

**清理项**：
- Memory provider 连接
- Agent 资源
- 线程局部存储（TLS）

---

## 调试技巧

### 1. 查看 Review Prompt

```python
from agent.background_review import _COMBINED_REVIEW_PROMPT
print(_COMBINED_REVIEW_PROMPT)
```

### 2. 手动触发 Review

```python
agent._turns_since_memory = agent._memory_nudge_interval
agent._iters_since_skill = agent._skill_nudge_interval
# 下次对话结束后会触发 review
```

### 3. 查看 Review 日志

```bash
# 启用 verbose 模式
export HERMES_VERBOSE=1

# 查看 agent.log
tail -f ~/.hermes/agent.log | grep "background review"
```

### 4. 禁用 Review（调试时）

```python
agent._memory_nudge_interval = 0
agent._skill_nudge_interval = 0
```

---

## 相关文件索引

### 核心实现
- `agent/background_review.py` - Review 主逻辑
- `agent/conversation_loop.py:4040-4065` - 触发和启动
- `agent/agent_init.py:1038-1148` - 初始化配置

### 工具实现
- `tools/memory_tool.py` - Memory 工具
- `tools/skill_manager_tool.py` - Skill 管理工具

### 测试文件
- `tests/run_agent/test_background_review.py`
- `tests/run_agent/test_background_review_summary.py`
- `tests/run_agent/test_background_review_cache_parity.py`
- `tests/run_agent/test_background_review_toolset_restriction.py`

### 文档
- `references/self-improvement-loop.md` - 设计文档（如果存在）

---

## 总结

Background Review 是一个精心设计的自我改进机制，通过以下手段实现了高效、安全、非侵入的学习循环：

1. **智能触发**：Memory 按回合、Skill 按迭代，精准捕捉学习时机
2. **隔离执行**：Fork + 工具白名单 + 安全回调，保证主流程不受影响
3. **缓存优化**：继承系统提示和配置，降低 API 成本 ~26%
4. **静默运行**：后台线程 + 输出重定向，用户无感知
5. **最佳努力**：失败静默降级，不阻塞主流程

这个机制让 Hermes Agent 能够在每次交互中积累知识，持续优化自己的行为模式和技能库，真正实现了"越用越聪明"的目标。
