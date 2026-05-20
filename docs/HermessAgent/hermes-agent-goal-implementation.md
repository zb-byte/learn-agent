# Hermes Agent Goal 实现原理

## 概述

Goal 是 Hermes Agent 中的持久化会话目标系统，允许用户设定一个跨多轮对话的目标，Agent 会自动持续工作直到目标完成。这是一个类似 "Ralph loop" 的自主循环机制，通过辅助模型判断目标是否达成，并自动生成延续提示词来驱动 Agent 继续工作。

## 核心设计理念

### 1. 非侵入式设计

Goal 系统的一个关键设计原则是**不修改 Agent 的系统提示词或工具集**：

- 延续提示词（continuation prompt）作为普通的用户消息追加到会话中
- 不改变系统提示词，保持 prompt caching 完整性
- 不交换工具集，Agent 能力保持一致
- 通过 `run_conversation` 正常流程处理，无特殊路径

### 2. Fail-Open 原则

Judge（判断器）失败时采用 **fail-open** 策略：

- Judge 调用失败时返回 `continue`（继续），而非阻塞进度
- 解析失败、网络错误等都不会中断 Goal 循环
- 真正的保护机制是 **turn budget**（轮次预算）和 **连续解析失败计数器**

### 3. 用户消息优先

当用户在 Goal 循环中发送新消息时：

- 用户消息会抢占（preempt）延续提示词
- Goal 循环在该轮暂停，但仍会在轮次结束后重新判断
- 如果用户消息恰好完成了目标，Judge 会判定为 `done`

## 核心组件

### 1. GoalState 数据结构

```python
@dataclass
class GoalState:
    goal: str                              # 目标文本
    status: str = "active"                 # active | paused | done | cleared
    turns_used: int = 0                    # 已使用轮次
    max_turns: int = DEFAULT_MAX_TURNS     # 最大轮次（默认 20）
    created_at: float = 0.0                # 创建时间戳
    last_turn_at: float = 0.0              # 最后一轮时间戳
    last_verdict: Optional[str] = None     # 最后判决：done | continue | skipped
    last_reason: Optional[str] = None      # 判决原因
    paused_reason: Optional[str] = None    # 暂停原因
    consecutive_parse_failures: int = 0    # 连续解析失败次数
    subgoals: List[str] = []               # 用户添加的子目标
```

**状态流转**：
```
pending → active → done
              ↓
            paused → resume → active
              ↓
           cleared
```

### 2. 持久化机制

Goal 状态通过 **SessionDB** 的 `state_meta` 表持久化：

- Key: `goal:<session_id>`
- Value: GoalState 的 JSON 序列化
- 支持 `/resume` 命令恢复会话时自动加载 Goal

**缓存策略**：
```python
_DB_CACHE: Dict[str, Any] = {}  # 按 HERMES_HOME 路径缓存 SessionDB 实例
```

避免每次调用都打开新连接，同时支持 profile 切换时使用正确的数据库。

### 3. Judge（判断器）机制

#### Judge 的职责

在每轮对话结束后，Judge 通过辅助模型判断目标是否完成：

```python
def judge_goal(
    goal: str,
    last_response: str,
    *,
    timeout: float = DEFAULT_JUDGE_TIMEOUT,
    subgoals: Optional[List[str]] = None,
) -> Tuple[str, str, bool]:
    """
    返回: (verdict, reason, parse_failed)
    - verdict: "done" | "continue" | "skipped"
    - reason: 判决原因
    - parse_failed: 是否解析失败（用于连续失败计数）
    """
```

#### Judge 系统提示词

```python
JUDGE_SYSTEM_PROMPT = """
You are a strict judge evaluating whether an autonomous agent has 
achieved a user's stated goal.

A goal is DONE only when:
- The response explicitly confirms the goal was completed, OR
- The response clearly shows the final deliverable was produced, OR
- The response explains the goal is unachievable / blocked / needs 
  user input (treat this as DONE with reason describing the block).

Otherwise the goal is NOT done — CONTINUE.

Reply ONLY with a single JSON object on one line:
{"done": <true|false>, "reason": "<one-sentence rationale>"}
"""
```

#### Judge 用户提示词

```python
JUDGE_USER_PROMPT_TEMPLATE = """
Goal:
{goal}

Agent's most recent response:
{response}

Current time: {current_time}

Is the goal satisfied?
"""
```

#### 解析 Judge 响应

```python
def _parse_judge_response(raw: str) -> Tuple[bool, str, bool]:
    """
    解析 Judge 的 JSON 响应，支持：
    1. 纯 JSON: {"done": true, "reason": "..."}
    2. Markdown 代码块包裹的 JSON
    3. 嵌入在文本中的 JSON（通过正则提取）
    4. 字符串形式的 done 值："true", "yes", "done", "1"
    
    返回: (done, reason, parse_failed)
    - parse_failed=True: 空响应或非 JSON（用于触发自动暂停）
    - parse_failed=False: API 错误（不计入连续失败）
    """
```

#### Fail-Open 策略

所有异常情况都返回 `("continue", "<error_reason>", False)`：

- 无辅助客户端配置
- API 调用失败
- 网络超时
- 模型返回格式错误

**唯一例外**：空响应或非 JSON 返回 `parse_failed=True`，用于触发自动暂停机制。

### 4. GoalManager 核心逻辑

#### 初始化

```python
class GoalManager:
    def __init__(self, session_id: str, *, default_max_turns: int = DEFAULT_MAX_TURNS):
        self.session_id = session_id
        self.default_max_turns = default_max_turns
        self._state: Optional[GoalState] = load_goal(session_id)
```

#### 设置目标

```python
def set(self, goal: str, *, max_turns: Optional[int] = None) -> GoalState:
    """创建新的 active 状态 Goal"""
    state = GoalState(
        goal=goal,
        status="active",
        turns_used=0,
        max_turns=max_turns or self.default_max_turns,
        created_at=time.time(),
    )
    self._state = state
    save_goal(self.session_id, state)
    return state
```

#### 评估轮次结果

```python
def evaluate_after_turn(
    self,
    last_response: str,
    *,
    user_initiated: bool = True,
) -> Dict[str, Any]:
    """
    在每轮对话结束后调用，执行以下步骤：
    
    1. 检查 Goal 是否 active
    2. 增加 turns_used 计数
    3. 调用 judge_goal 获取判决
    4. 更新 consecutive_parse_failures 计数
    5. 根据判决决定下一步行动
    
    返回决策字典：
    {
        "status": "active" | "paused" | "done",
        "should_continue": bool,
        "continuation_prompt": str | None,
        "verdict": "done" | "continue" | "skipped" | "inactive",
        "reason": str,
        "message": str  # 用户可见的状态消息
    }
    """
```

**决策逻辑**：

```python
# 1. 目标完成
if verdict == "done":
    state.status = "done"
    return {
        "should_continue": False,
        "message": f"✓ Goal achieved: {reason}"
    }

# 2. Judge 连续解析失败（弱模型保护）
if state.consecutive_parse_failures >= 3:
    state.status = "paused"
    state.paused_reason = "judge model returned unparseable output"
    return {
        "should_continue": False,
        "message": "⏸ Goal paused — judge model isn't returning JSON..."
    }

# 3. 轮次预算耗尽
if state.turns_used >= state.max_turns:
    state.status = "paused"
    state.paused_reason = f"turn budget exhausted ({turns_used}/{max_turns})"
    return {
        "should_continue": False,
        "message": f"⏸ Goal paused — {turns_used}/{max_turns} turns used."
    }

# 4. 继续工作
return {
    "should_continue": True,
    "continuation_prompt": self.next_continuation_prompt(),
    "message": f"↻ Continuing toward goal ({turns_used}/{max_turns}): {reason}"
}
```

#### 生成延续提示词

```python
def next_continuation_prompt(self) -> Optional[str]:
    """生成下一轮的延续提示词"""
    if not self._state or self._state.status != "active":
        return None
    
    # 有子目标时使用特殊模板
    if self._state.subgoals:
        return CONTINUATION_PROMPT_WITH_SUBGOALS_TEMPLATE.format(
            goal=self._state.goal,
            subgoals_block=self._state.render_subgoals_block(),
        )
    
    # 标准模板
    return CONTINUATION_PROMPT_TEMPLATE.format(goal=self._state.goal)
```

**标准延续提示词模板**：

```python
CONTINUATION_PROMPT_TEMPLATE = """
[Continuing toward your standing goal]
Goal: {goal}

Continue working toward this goal. Take the next concrete step. 
If you believe the goal is complete, state so explicitly and stop. 
If you are blocked and need input from the user, say so clearly and stop.
"""
```

## 高级特性

### 1. Subgoals（子目标）

用户可以在 Goal 循环中通过 `/subgoal` 命令添加额外的完成标准：

```python
def add_subgoal(self, text: str) -> str:
    """添加子目标到 active Goal"""
    self._state.subgoals.append(text.strip())
    save_goal(self.session_id, self._state)
    return text

def render_subgoals_block(self) -> str:
    """渲染为编号列表"""
    return "\n".join(f"- {i}. {text}" 
                     for i, text in enumerate(self.subgoals, start=1))
```

**带子目标的延续提示词**：

```python
CONTINUATION_PROMPT_WITH_SUBGOALS_TEMPLATE = """
[Continuing toward your standing goal]
Goal: {goal}

Additional criteria the user added mid-loop:
{subgoals_block}

Continue working toward the goal AND all additional criteria. 
If you believe the goal and every additional criterion are complete, 
state so explicitly and stop.
"""
```

**带子目标的 Judge 提示词**：

```python
JUDGE_USER_PROMPT_WITH_SUBGOALS_TEMPLATE = """
Goal:
{goal}

Additional criteria the user added mid-loop (all must also be 
satisfied for the goal to be DONE):
{subgoals_block}

Agent's most recent response:
{response}

Decision: For each numbered criterion above, find concrete 
evidence in the agent's response that the criterion is 
satisfied. Do not accept generic phrases like 'all requirements 
met' — require specific evidence. If ANY criterion lacks 
specific evidence, the goal is NOT done — return CONTINUE.

Is the goal AND every additional criterion satisfied?
"""
```

### 2. 连续解析失败保护

防止弱 Judge 模型（如 deepseek-v4-flash）无法返回正确 JSON 格式而耗尽轮次预算：

```python
DEFAULT_MAX_CONSECUTIVE_PARSE_FAILURES = 3

# 在 evaluate_after_turn 中
if parse_failed:
    state.consecutive_parse_failures += 1
else:
    state.consecutive_parse_failures = 0  # 任何成功响应都重置计数

if state.consecutive_parse_failures >= DEFAULT_MAX_CONSECUTIVE_PARSE_FAILURES:
    state.status = "paused"
    state.paused_reason = "judge model returned unparseable output N turns in a row"
    # 返回配置指导消息，告诉用户如何切换到更强的 Judge 模型
```

**关键区分**：

- **解析失败**（`parse_failed=True`）：空响应、非 JSON → 计入连续失败
- **API 错误**（`parse_failed=False`）：网络错误、超时 → 不计入连续失败

这样可以避免临时网络问题触发自动暂停。

### 3. Judge 输出预算

```python
DEFAULT_JUDGE_MAX_TOKENS = 4096  # 从 200 提升到 4096
```

**原因**：推理模型（deepseek-v4, qwq）在输出可见 JSON 之前会生成隐藏的推理内容，200 tokens 会截断 JSON，导致解析失败。4096 tokens 足以覆盖所有测试过的模型。

可通过配置覆盖：

```yaml
# ~/.hermes/config.yaml
auxiliary:
  goal_judge:
    max_tokens: 4096
```

## 集成方式

### 1. CLI 集成

```python
class HermesCLI:
    def _get_goal_manager(self):
        """返回绑定到当前 session_id 的 GoalManager"""
        sid = self.session_id
        existing = getattr(self, "_goal_manager", None)
        if existing and existing.session_id == sid:
            return existing
        
        cfg = load_config() or {}
        max_turns = int(cfg.get("goals", {}).get("max_turns", 20))
        mgr = GoalManager(session_id=sid, default_max_turns=max_turns)
        self._goal_manager = mgr
        return mgr
    
    def _after_turn_hook(self, last_response: str):
        """每轮对话结束后调用"""
        mgr = self._get_goal_manager()
        if not mgr or not mgr.is_active():
            return
        
        decision = mgr.evaluate_after_turn(last_response, user_initiated=True)
        msg = decision.get("message")
        if msg:
            print(msg)
        
        if decision.get("should_continue"):
            prompt = decision.get("continuation_prompt")
            if prompt:
                self._pending_input.put(prompt)  # 入队延续提示词
```

### 2. Gateway 集成

```python
class GatewayRunner:
    async def _after_agent_response(self, source, final_response):
        """Agent 响应后异步调用"""
        sid = self._session_id_for_source(source)
        mgr = GoalManager(session_id=sid, default_max_turns=max_turns)
        
        if not mgr.is_active():
            return
        
        decision = mgr.evaluate_after_turn(final_response, user_initiated=True)
        msg = decision.get("message")
        
        # 延迟发送状态消息，确保在 Agent 响应之后
        if msg and source:
            await self._defer_goal_status_notice_after_delivery(source, msg)
        
        if decision.get("should_continue"):
            prompt = decision.get("continuation_prompt")
            if prompt:
                # 通过 adapter 的 FIFO 入队，用户消息自然抢占
                adapter = self.adapters.get(source.platform)
                cont_event = MessageEvent(text=prompt, source=source, ...)
                await adapter.enqueue_message(cont_event)
```

## 用户命令

### /goal 命令

```bash
# 设置目标
/goal <目标文本>

# 查看状态
/goal
/goal status

# 暂停
/goal pause

# 恢复（重置轮次预算）
/goal resume

# 清除
/goal clear
```

### /subgoal 命令

```bash
# 添加子目标
/subgoal <标准文本>

# 查看所有子目标
/subgoal

# 删除子目标（1-based 索引）
/subgoal rm <N>

# 清除所有子目标
/subgoal clear
```

## 配置

```yaml
# ~/.hermes/config.yaml
goals:
  max_turns: 20  # 默认最大轮次

auxiliary:
  goal_judge:
    provider: openrouter
    model: google/gemini-3-flash-preview  # 推荐使用严格遵循 JSON 的模型
    max_tokens: 4096
```

## 工作流程示例

### 完整的 Goal 循环

```
1. 用户: /goal 实现用户认证功能

2. GoalManager.set("实现用户认证功能")
   → status = "active", turns_used = 0

3. Agent 第一轮响应: "我会创建登录和注册接口..."
   → GoalManager.evaluate_after_turn(response)
   → judge_goal() 返回 ("continue", "仅规划，未实现")
   → 生成 continuation_prompt
   → 自动入队下一轮

4. Agent 第二轮响应: "已创建 auth.py，实现了 login() 和 register()..."
   → judge_goal() 返回 ("continue", "缺少测试")
   → turns_used = 2
   → 继续

5. Agent 第三轮响应: "已添加测试，所有测试通过 ✓"
   → judge_goal() 返回 ("done", "功能完整且测试通过")
   → status = "done"
   → 显示: "✓ Goal achieved: 功能完整且测试通过"
   → 循环结束
```

### 用户消息抢占

```
1. Goal active, Agent 正在工作...

2. 用户发送新消息: "先暂停，我需要改需求"
   → 用户消息优先处理
   → Goal 循环在本轮暂停
   → 仍会在轮次结束后 evaluate_after_turn

3. 如果用户消息恰好完成了目标
   → Judge 判定 "done"
   → Goal 自动完成
```

### 轮次预算耗尽

```
1. Goal: "优化整个系统性能"
   max_turns = 20

2. Agent 工作了 20 轮，Judge 仍返回 "continue"

3. evaluate_after_turn 检测到 turns_used >= max_turns
   → status = "paused"
   → paused_reason = "turn budget exhausted (20/20)"
   → 显示: "⏸ Goal paused — 20/20 turns used. Use /goal resume to keep going."

4. 用户可以:
   - /goal resume: 重置 turns_used，继续工作
   - /goal clear: 放弃目标
```

### Judge 模型失败保护

```
1. Goal active, 使用弱 Judge 模型（如 deepseek-v4-flash）

2. 第 1 轮: Judge 返回空字符串
   → parse_failed = True
   → consecutive_parse_failures = 1
   → verdict = "continue" (fail-open)

3. 第 2 轮: Judge 返回 "Let me analyze..."（非 JSON）
   → parse_failed = True
   → consecutive_parse_failures = 2

4. 第 3 轮: Judge 又返回非 JSON
   → consecutive_parse_failures = 3
   → 触发自动暂停
   → 显示配置指导消息，建议切换到 gemini-3-flash-preview

5. 用户修改配置后 /goal resume
   → consecutive_parse_failures 重置为 0
   → 使用新的 Judge 模型继续
```

## 设计优势

### 1. 零侵入性

- 不修改系统提示词，保持 prompt caching 效率
- 不改变工具集，Agent 能力一致
- 延续提示词是普通用户消息，无特殊处理路径

### 2. 鲁棒性

- Fail-open 策略确保 Judge 失败不阻塞进度
- 轮次预算是最终保护机制
- 连续解析失败保护防止弱模型耗尽预算
- API 错误与解析失败分开处理

### 3. 用户控制

- 用户消息自然抢占延续提示词
- 可随时暂停、恢复、清除
- 可动态添加子目标调整完成标准
- 透明的状态消息和进度显示

### 4. 持久化

- 状态存储在 SessionDB，支持跨会话恢复
- `/resume` 自动加载 Goal 状态
- 支持多 profile 隔离

### 5. 可配置性

- 轮次预算可配置
- Judge 模型可独立配置
- Judge 输出预算可调整
- 支持不同 provider 的辅助模型

## Goal 模式 vs 普通使用 Agent

### 核心问题

Hermes Agent 本身就是自主的 AI Agent，会根据用户指令自动完成开发、测试等任务。那么 Goal 模式和普通使用 Agent 有什么区别？

### 关键区别：单轮尽力 vs 多轮循环

**普通使用 Agent**（单轮尽力完成）：
```
用户: 实现用户认证功能，包括登录、注册、测试

Agent 在一轮对话中尽力完成：
- 创建 auth.py
- 实现 login() 和 register()
- 创建 test_auth.py
- 写了几个基本测试
- 运行测试，都通过了

Agent: "我已经实现了用户认证功能，包括登录、注册和基本测试。"
[等待用户下一个指令]
```

**问题**：
- Agent 可能认为"差不多完成了"就停止
- 可能遗漏边界情况测试
- 可能没有写文档
- 可能没有处理错误情况
- 用户需要检查，发现问题后再提新请求

**Goal 模式**（多轮循环直到真正完成）：
```
用户: /goal 实现用户认证功能，包括登录、注册、测试

第 1 轮:
Agent: 创建 auth.py，实现基本功能
Judge: "仅实现了基本功能，缺少测试" → continue

第 2 轮（自动触发）:
Agent: 添加测试，运行通过
Judge: "测试覆盖不足，缺少边界情况" → continue

第 3 轮（自动触发）:
Agent: 补充边界测试，添加错误处理
Judge: "缺少文档" → continue

第 4 轮（自动触发）:
Agent: 添加 docstring 和 README
Judge: "功能完整，测试充分，文档齐全" → done

✓ Goal achieved!
```

### 详细对比

| 维度 | 普通使用 Agent | Goal 模式 |
|------|---------------|----------|
| **工作方式** | 单轮对话中尽力完成 | 多轮循环直到真正完成 |
| **完成判断** | Agent 自己认为完成 | Judge 客观判断是否完成 |
| **完成标准** | Agent 的主观判断 | 外部 Judge 的严格标准 |
| **遗漏处理** | 用户发现后提新请求 | Judge 发现后自动继续 |
| **持久性** | 单次对话 | 持久化，支持跨会话 |
| **用户介入** | 任务完成后检查 | 可随时暂停/添加子目标 |
| **质量保证** | 依赖 Agent 自觉 | Judge 强制检查 |
| **适用场景** | 简单明确的任务 | 复杂、标准严格的任务 |

### 实际场景对比

#### 场景 1：简单功能实现

**普通使用 Agent 足够**：
```
用户: 添加一个 getUserName() 函数

Agent:
- 添加函数
- 写测试
- 运行测试通过

Agent: "完成！"
```
- 任务简单明确
- 一轮就能完成
- 不需要循环

#### 场景 2：复杂功能实现

**Goal 模式更可靠**：

**普通使用 Agent**：
```
用户: 实现完整的用户认证系统

Agent 第一轮:
- 实现了登录和注册
- 写了几个测试
- "完成了！"

用户检查后发现:
❌ 没有密码加密
❌ 没有 session 管理
❌ 没有错误处理
❌ 测试覆盖不足

用户: "还需要密码加密"
Agent: [添加密码加密]

用户: "还需要 session 管理"
Agent: [添加 session]

用户: "测试不够"
Agent: [补充测试]

# 需要多次人工检查和跟进
```

**Goal 模式**：
```
用户: /goal 实现完整的用户认证系统

第 1 轮:
Agent: 实现基本登录注册
Judge: "缺少密码加密" → continue

第 2 轮（自动）:
Agent: 添加密码加密
Judge: "缺少 session 管理" → continue

第 3 轮（自动）:
Agent: 添加 session 管理
Judge: "错误处理不完整" → continue

第 4 轮（自动）:
Agent: 完善错误处理
Judge: "测试覆盖不足" → continue

第 5 轮（自动）:
Agent: 补充测试
Judge: "功能完整，测试充分" → done

✓ Goal achieved!

# 自动循环，无需人工跟进
```

### 为什么需要 Goal 模式？

#### 1. Agent 的"完成"可能不够完整

即使是强大的 AI Agent，在单轮对话中也可能：
- **过早停止**："主要功能实现了，应该够了"
- **遗漏细节**：忘记文档、边界测试、错误处理
- **标准不一致**：不同时候对"完成"的理解不同

#### 2. Judge 提供客观标准

```python
JUDGE_SYSTEM_PROMPT = """
A goal is DONE only when:
- The response explicitly confirms the goal was completed, OR
- The response clearly shows the final deliverable was produced, OR
- The response explains the goal is unachievable / blocked
"""
```

Judge 不会被 Agent 的"我觉得完成了"所影响，而是客观检查：
- 代码是否真的写了？
- 测试是否真的通过了？
- 文档是否真的更新了？

#### 3. 自动迭代改进

```
普通模式：
Agent 完成 → 用户检查 → 发现问题 → 用户提新请求 → Agent 修复 → 用户再检查 → ...

Goal 模式：
Agent 完成 → Judge 检查 → 发现问题 → 自动继续 → Agent 修复 → Judge 再检查 → ...
```

Goal 模式把"检查-反馈-改进"循环自动化了。

#### 4. 持久化和可恢复

```
普通模式：
用户: 实现认证系统
Agent: [工作中...]
[用户关闭会话]
[第二天打开]
用户: 继续昨天的任务？  # Agent 可能不记得具体进度

Goal 模式：
用户: /goal 实现认证系统
Agent: [工作中...]
[用户关闭会话]
[第二天打开]
用户: /resume
Goal 自动恢复: "⊙ Goal (active, 3/20 turns): 实现认证系统"
Agent: [从上次停止的地方继续]
```

### 关键问题：如果我明确要求"实现功能+测试+文档"呢？

用户可能会问：**如果我在普通模式下明确要求 Agent "实现用户认证功能，并写测试和文档"，这和 Goal 模式有什么区别？**

这是个好问题！让我们看实际场景：

#### 场景对比：明确要求的任务

**普通模式（明确要求）**：
```
用户: 实现用户认证功能，要求：
1. 登录和注册功能
2. 完整的单元测试
3. API 文档
4. 错误处理

Agent 在一轮对话中工作：
[创建 auth.py]
[实现 login() 和 register()]
[创建 test_auth.py]
[写了 5 个测试]
[添加 docstring]
[运行测试，都通过了]

Agent: "完成！我实现了登录和注册功能，写了 5 个单元测试（都通过了），
并添加了 docstring 文档。"

[等待用户下一个指令]
```

**用户检查后可能发现**：
- ❌ 测试只覆盖了正常情况，没有测试错误情况
- ❌ 没有测试密码加密
- ❌ docstring 有，但没有 README 使用说明
- ❌ 错误处理不完整（如数据库连接失败）

**用户需要继续跟进**：
```
用户: 测试不够，需要测试错误情况
Agent: [补充测试]

用户: 还需要 README 说明如何使用
Agent: [写 README]

用户: 错误处理不完整
Agent: [补充错误处理]
```

**Goal 模式（同样的要求）**：
```
用户: /goal 实现用户认证功能，要求：
1. 登录和注册功能
2. 完整的单元测试
3. API 文档
4. 错误处理

第 1 轮:
Agent: [实现基本功能和测试]
Judge: "测试只覆盖了正常情况，缺少错误情况测试" → continue

第 2 轮（自动触发）:
Agent: [补充错误情况测试]
Judge: "没有测试密码加密" → continue

第 3 轮（自动触发）:
Agent: [添加密码加密测试]
Judge: "文档只有 docstring，缺少使用说明" → continue

第 4 轮（自动触发）:
Agent: [写 README]
Judge: "错误处理不完整（数据库连接失败未处理）" → continue

第 5 轮（自动触发）:
Agent: [补充错误处理]
Judge: "所有要求都满足，测试充分，文档完整" → done

✓ Goal achieved!
```

#### 核心区别

| 维度 | 普通模式（明确要求） | Goal 模式 |
|------|---------------------|----------|
| **Agent 的理解** | "我做了测试和文档" | Judge: "测试是否真的充分？" |
| **质量标准** | Agent 主观判断 | Judge 客观检查 |
| **发现问题** | 用户检查后发现 | Judge 每轮自动检查 |
| **修复流程** | 用户提新请求 → Agent 修复 | Judge 发现 → 自动继续 |
| **用户工作量** | 需要仔细检查和多次跟进 | 设置后等待完成 |
| **中途失败** | 如测试失败，Agent 等待用户指示 | 测试失败，Judge 判断未完成，自动继续修复 |
| **长任务处理** | 一个超长响应 | 分多轮，每轮合理长度 |

#### 真实案例：测试失败的处理

**普通模式**：
```
用户: 实现用户认证功能，包括测试

Agent:
[实现功能]
[写测试]
[运行测试]
测试失败：test_login_invalid_password FAILED

Agent: "我实现了功能并写了测试，但有一个测试失败了。
看起来是密码验证逻辑有问题。你想让我修复吗？"

[等待用户指示]
```

**Goal 模式**：
```
用户: /goal 实现用户认证功能，包括测试

第 1 轮:
Agent: [实现功能，写测试，运行测试]
测试失败：test_login_invalid_password FAILED
Agent: "测试失败，我发现密码验证逻辑有问题"

Judge: "测试失败，功能未完成" → continue

第 2 轮（自动触发）:
Agent: [修复密码验证逻辑，重新运行测试]
所有测试通过

Judge: "测试通过，但缺少文档" → continue

第 3 轮（自动触发）:
Agent: [添加文档]

Judge: "完成" → done
```

#### 动态需求变化

**普通模式**：
```
用户: 实现用户认证功能，包括测试和文档

Agent: [工作中...]
[5 分钟后]

用户想加需求，但 Agent 还在工作，只能等待...

Agent: "完成了！"

用户: 还需要支持 OAuth
Agent: [重新开始工作]
```

**Goal 模式**：
```
用户: /goal 实现用户认证功能

Agent: [第 1 轮工作中...]

用户: /subgoal 支持 OAuth  # 动态添加需求
用户: /subgoal 添加 rate limiting

Agent: [继续工作，现在朝着所有目标]
Judge: [检查所有目标是否都完成]
```

#### 中断和恢复

**普通模式**：
```
用户: 实现用户认证功能，包括测试和文档

Agent: [工作中...]
[网络中断 / 用户关闭会话]

[第二天]
用户: 继续昨天的任务
Agent: "抱歉，我不记得昨天的具体进度，能再说一下吗？"
```

**Goal 模式**：
```
用户: /goal 实现用户认证功能

Agent: [第 1 轮完成]
Agent: [第 2 轮工作中...]
[网络中断 / 用户关闭会话]

[第二天]
用户: /resume
Goal 自动恢复: "⊙ Goal (active, 2/20 turns): 实现用户认证功能"
Agent: [从第 3 轮继续]
```

### 何时使用哪个？

### 更深入的问题：详细流程 Prompt vs Goal

用户可能会进一步问：**如果我在普通模式下输入非常详细的流程 Prompt，是不是就等同于 Goal 了？**

例如：
```
实现用户认证功能，严格按照以下流程：
1. 实现登录和注册功能，包括密码加密
2. 写单元测试，必须覆盖：
   - 正常登录/注册
   - 错误密码
   - 用户不存在
   - 密码加密验证
3. 运行测试，如果失败则修复，直到全部通过
4. 写完整文档，包括：
   - API 使用说明
   - 代码示例
   - 错误处理说明
5. 最后验证所有功能正常工作
```

这样是不是就等同于 Goal 了？**答案是：不等同，仍有本质区别。**

#### 实际对比

**详细流程 Prompt**：
```
用户: [上述详细流程]

Agent 在一轮对话中执行：
[实现功能，加密密码]
[写测试：test_login_success, test_login_wrong_password, ...]
[运行测试]
测试结果：5 passed

[写文档]
"""
API 使用说明：
login(username, password) - 用户登录
register(username, password) - 用户注册
...
"""

Agent: "完成！我按照流程完成了所有步骤：
1. ✓ 实现了登录和注册，使用 bcrypt 加密密码
2. ✓ 写了 5 个单元测试，覆盖了正常和异常情况，全部通过
3. ✓ 写了 API 文档和使用示例
4. ✓ 验证了所有功能正常工作"

[等待用户下一个指令]
```

**用户检查后可能发现**：
- ❌ 测试没有覆盖"空密码"情况
- ❌ 测试没有验证密码是否真的被加密存储
- ❌ 文档缺少错误码说明
- ❌ 没有测试并发登录的情况

**为什么会这样？** Agent 自己判断"我覆盖了正常和异常情况"，但它的理解可能不够全面。

**Goal 模式（同样的要求）**：
```
用户: /goal 实现用户认证功能，要求：
- 登录和注册功能，密码加密
- 完整的单元测试
- 完整的文档

第 1 轮:
Agent: [实现功能，写测试，写文档]
Agent: "完成了功能、测试和文档"

Judge 检查 Agent 的输出:
"测试没有覆盖空密码情况" → continue

第 2 轮（自动触发）:
Agent: [添加空密码测试]

Judge: "测试没有验证密码是否被加密存储" → continue

第 3 轮（自动触发）:
Agent: [添加密码加密验证测试]

Judge: "文档缺少错误码说明" → continue

第 4 轮（自动触发）:
Agent: [补充错误码文档]

Judge: "所有要求都满足，测试全面，文档完整" → done
```

#### 核心区别

| 维度 | 详细流程 Prompt | Goal 模式 |
|------|----------------|----------|
| **执行方式** | 单轮执行所有步骤 | 多轮迭代改进 |
| **自我评估** | Agent 自己判断"我做完了" | Judge 客观检查"真的做完了吗？" |
| **完成标准** | Agent 的主观理解 | Judge 的客观验证 |
| **发现遗漏** | 用户检查后发现 | Judge 每轮自动发现 |
| **失败处理** | "我遇到问题了，怎么办？" | 自动继续修复 |
| **质量保证** | 依赖 Agent 的自觉性 | 独立 Judge 强制检查 |
| **持久化** | 不支持 | 支持跨会话恢复 |
| **动态调整** | 不支持 | 支持 /subgoal |

#### 关键洞察：自我评估的局限性

**问题**：即使你给了详细流程，Agent 仍然是**自己判断**是否完成：

```python
# Agent 的内心独白
"用户要求测试覆盖正常和异常情况"
"我写了 test_login_success 和 test_login_wrong_password"
"这应该够了吧？" ✓ 自己打勾
"用户要求完整文档"
"我写了 API 说明和示例"
"这应该够了吧？" ✓ 自己打勾
"好，完成了！"
```

**Goal 模式的 Judge**：

```python
# Judge 的客观检查
Judge 看到 Agent 的输出:
- 有 test_login_success ✓
- 有 test_login_wrong_password ✓
- 但是没有 test_empty_password ✗
- 没有验证密码是否被加密 ✗
- 文档没有错误码说明 ✗

Judge: "测试不完整" → continue
```

Judge 不会被 Agent 的"我觉得完成了"所影响，而是实际检查输出内容。

#### 实际案例：测试失败的处理

**详细流程 Prompt**：
```
用户: 实现认证功能，流程：
1. 实现功能
2. 写测试
3. 运行测试，失败则修复，直到全部通过
4. 写文档

Agent:
[实现功能]
[写测试]
[运行测试]
FAILED: test_login - AssertionError: Expected 200, got 401

Agent: "我发现测试失败了，看起来是登录逻辑有问题。
让我修复一下..."

[修复代码]
[重新运行测试]
FAILED: test_register - KeyError: 'username'

Agent: "又发现一个问题，注册功能有 bug。
这个问题比较复杂，你能帮我看看吗？"

[等待用户帮助]
```

Agent 可能在多次失败后**放弃**或**寻求帮助**。

**Goal 模式**：
```
用户: /goal 实现认证功能，包括测试

第 1 轮:
Agent: [实现功能，写测试，运行测试]
FAILED: test_login - AssertionError

Judge: "测试失败，功能未完成" → continue

第 2 轮（自动触发）:
Agent: [修复登录逻辑，重新测试]
FAILED: test_register - KeyError

Judge: "测试仍然失败" → continue

第 3 轮（自动触发）:
Agent: [修复注册逻辑，重新测试]
PASSED: All tests passed

Judge: "测试通过，但缺少文档" → continue

第 4 轮（自动触发）:
Agent: [添加文档]

Judge: "完成" → done
```

Goal 模式会**持续尝试**，不会轻易放弃。

#### 长任务的处理

**详细流程 Prompt**：
```
用户: [非常详细的流程，要求实现 10 个功能]

Agent: [尝试在一个响应中完成所有 10 个功能]
[生成了 5000 行代码]
[响应超出 token 限制，被截断]

Agent: "我实现了前 7 个功能，但是响应太长了，
剩下 3 个功能需要你再问一次..."
```

**Goal 模式**：
```
用户: /goal 实现 10 个功能

第 1 轮: 实现功能 1-3
第 2 轮: 实现功能 4-6
第 3 轮: 实现功能 7-9
第 4 轮: 实现功能 10
第 5 轮: 补充测试
第 6 轮: 补充文档

每轮合理长度，不会被截断
```

#### 类比

**详细流程 Prompt** = 给员工一个详细的工作清单，让他自己检查
```
你: "按照这个清单做，做完自己检查一遍"
员工: [工作]
员工: "我都做了，自己检查过了，没问题！"
你: [实际检查] "这里不够，那里有问题..."
```

**Goal 模式** = 给员工一个目标，配一个独立的质检员
```
你: "完成这个目标"
员工: [工作]
质检员: [检查] "这里不够" → 继续
员工: [补充]
质检员: [再检查] "那里有问题" → 继续
员工: [修复]
质检员: [再检查] "达标了" → 完成
你: [收到通知] "好的！"
```

#### 本质区别总结

**详细流程 Prompt**：
- Agent 既是**执行者**又是**检查者**
- 容易出现"自己觉得做完了"的盲区
- 单轮执行，失败后可能放弃
- 依赖 Agent 的自觉性

**Goal 模式**：
- Agent 是**执行者**，Judge 是**检查者**
- 职责分离，客观评估
- 多轮迭代，持续改进直到达标
- 强制质量检查

**核心价值**：Goal 模式通过**职责分离**（执行 vs 检查）和**多轮迭代**，确保任务真正达标，而不是 Agent 自己认为达标。

### 何时使用哪个？

#### 使用普通 Agent（包括详细流程 Prompt）

### 类比

**普通使用 Agent** = 让员工"尽力完成"任务
```
你: "实现用户认证"
员工: [努力工作]
员工: "我做完了！"
你: [检查] "还缺少密码加密"
员工: [补充]
你: [再检查] "测试不够"
员工: [再补充]
# 需要你持续检查和反馈
```

**Goal 模式** = 让员工"完全达标"才算完成
```
你: "实现用户认证，标准是：功能完整、测试充分、文档齐全"
员工: [工作]
质检: "缺少密码加密" → 继续
员工: [补充]
质检: "测试不够" → 继续
员工: [补充]
质检: "全部达标" → 完成
# 质检自动把关，你不用盯着
```

### Goal vs 普通 Prompt

#### 核心区别

#### 1. 持久性 vs 一次性

**普通 Prompt**：
```
用户: 帮我实现用户认证功能
Agent: [完成响应]
[等待用户下一个指令]
```
- 一次性交互，处理完就结束
- 需要用户持续跟进："现在写测试"、"现在写文档"...

**Goal**：
```
用户: /goal 实现用户认证功能
Agent: [第一轮响应]
Judge: "未完成，继续"
Agent: [第二轮响应，自动触发]
Judge: "未完成，继续"
Agent: [第三轮响应，自动触发]
Judge: "完成！"
[自动结束]
```
- 持久化存储在 SessionDB
- 自动循环直到完成，无需用户持续输入

#### 2. 被动 vs 主动

**普通 Prompt**：
- **用户主导**：每一步都需要用户明确指令
- Agent 是**被动响应者**
- 用户需要判断任务是否完成

**Goal**：
- **Agent 主导**：设置后 Agent 自主工作
- Agent 是**主动执行者**
- Judge 自动判断任务是否完成
- 用户可以随时介入（消息抢占机制）

#### 3. 无状态 vs 有状态

**普通 Prompt**：
```python
# 无状态，每次都是新的对话
用户输入 → Agent 响应 → 结束
```

**Goal**：
```python
# 有完整状态机
GoalState {
    status: "active" | "paused" | "done" | "cleared"
    turns_used: 3/20
    last_verdict: "continue"
    consecutive_parse_failures: 0
    subgoals: ["写测试", "写文档"]
}
```

#### 4. 无保护 vs 有保护

**普通 Prompt**：
- 无限制，用户控制何时停止
- 可能陷入无效循环（用户需要手动中断）

**Goal**：
- **Turn Budget**：默认 20 轮，防止无限循环
- **连续解析失败保护**：Judge 失败 3 次自动暂停
- **用户可暂停/恢复/清除**

#### 5. 单目标 vs 可演化目标

**普通 Prompt**：
```
用户: 实现登录功能
Agent: [完成]

用户: 再加上注册功能  # 新的独立请求
Agent: [完成]
```

**Goal + Subgoals**：
```
用户: /goal 实现用户认证
Agent: [开始工作...]

用户: /subgoal 添加邮箱验证  # 动态添加标准
用户: /subgoal 支持 OAuth    # 再添加标准
Agent: [自动朝着所有标准工作]
Judge: [评估所有标准是否都满足]
```

### 实际场景对比

#### 场景 1：简单任务

**普通 Prompt 更合适**：
```
用户: 把这个函数重命名为 getUserInfo
Agent: [完成]
```
- 单步任务，立即完成
- 不需要循环和判断

#### 场景 2：多步骤任务

**Goal 更合适**：
```
用户: /goal 重构整个认证模块，提高代码质量

Agent 自动执行：
1. 分析现有代码
2. 识别问题
3. 重构代码
4. 添加测试
5. 更新文档
6. 验证功能

Judge 每轮判断是否完成，直到所有步骤都完成
```

**如果用普通 Prompt**：
```
用户: 重构认证模块
Agent: [分析了一下，给出建议]

用户: 好的，开始重构  # 需要手动跟进
Agent: [重构了部分代码]

用户: 继续，还有测试  # 需要手动跟进
Agent: [添加测试]

用户: 还有文档呢？  # 需要手动跟进
Agent: [更新文档]

用户: 验证一下功能  # 需要手动跟进
Agent: [验证]
```

### 技术实现差异

#### 普通 Prompt 流程

```python
# 简单的请求-响应
user_message = "实现登录功能"
response = agent.run_conversation([
    {"role": "user", "content": user_message}
])
# 结束，等待下一个用户输入
```

#### Goal 流程

```python
# 1. 设置 Goal
goal_manager.set("实现登录功能")

# 2. 第一轮
response1 = agent.run_conversation([
    {"role": "user", "content": "实现登录功能"}
])

# 3. Judge 判断
verdict, reason, _ = judge_goal(goal, response1)

# 4. 如果未完成，自动生成延续提示词
if verdict == "continue":
    continuation = "[Continuing toward your standing goal]\nGoal: 实现登录功能\n\nContinue working..."
    
    # 5. 自动触发第二轮
    response2 = agent.run_conversation([
        {"role": "user", "content": "实现登录功能"},
        {"role": "assistant", "content": response1},
        {"role": "user", "content": continuation}  # 自动追加
    ])
    
    # 6. 再次 Judge...
    # 循环直到 verdict == "done" 或达到 turn budget
```

### 何时使用哪个？

#### 使用普通 Prompt

- ✅ 单步任务（重命名、修复 typo、查询信息）
- ✅ 需要立即反馈的交互式探索
- ✅ 任务边界清晰，一次就能完成
- ✅ 你想完全控制每一步

#### 使用 Goal

- ✅ 多步骤复杂任务（重构、实现完整功能）
- ✅ 需要 Agent 自主工作到完成
- ✅ 任务可能需要多轮迭代
- ✅ 你想"设置后忘记"，让 Agent 自己干活
- ✅ 需要跨会话持久化（可以 `/resume` 恢复）

### 类比

**普通 Prompt** = 你给员工下达具体指令
```
"去复印这份文件"
"现在装订一下"
"放到我桌上"
```
每一步都需要你明确指示。

**Goal** = 你给员工设定目标
```
"准备明天会议的材料包"
```
员工自己规划步骤：复印、装订、准备投影、检查设备...直到完成。你可以中途检查或调整，但不需要指导每一步。

### 对比总结表

| 维度 | 普通 Prompt | Goal |
|------|------------|------|
| **持久性** | 一次性 | 持久化，跨会话 |
| **控制权** | 用户主导每一步 | Agent 自主循环 |
| **状态** | 无状态 | 有状态机 |
| **判断** | 用户判断是否完成 | Judge 自动判断 |
| **保护** | 无限制 | Turn budget + 失败保护 |
| **适用** | 简单单步任务 | 复杂多步任务 |
| **交互** | 高频交互 | 低频监督 |
| **演化** | 每次新请求 | 可动态添加子目标 |
| **恢复** | 不支持 | 支持 `/resume` |

**本质区别**：Goal 是在普通 Prompt 之上构建的**自主循环层**，让 Agent 从"被动工具"变成"主动助手"。

## 总结

Hermes Agent 的 Goal 系统是一个精心设计的自主循环机制，通过以下核心要素实现了强大而可靠的持久化目标追踪：

1. **非侵入式架构**：延续提示词作为普通用户消息，保持系统简洁
2. **Fail-open 哲学**：Judge 失败不阻塞，轮次预算是最终保护
3. **智能判断**：辅助模型评估目标完成度，支持子目标细化
4. **用户优先**：用户消息自然抢占，保持交互控制权
5. **鲁棒保护**：连续解析失败检测，防止弱模型耗尽预算
6. **持久化状态**：SessionDB 存储，支持跨会话恢复

这种设计使得 Hermes Agent 能够自主完成复杂的多步骤任务，同时保持用户控制和系统稳定性。Goal 系统将 Agent 从"被动响应工具"提升为"主动工作助手"，适合需要多轮迭代和自主决策的复杂任务场景。
