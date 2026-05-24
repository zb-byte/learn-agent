# Hermes Agent - 迭代预算系统 (Iteration Budget)

> 基于源码：`/Users/wangzhongbin/Documents/code/fangzhen/hermes-agent`

## 1. 概述

迭代预算系统是 Hermes Agent 中用于控制 agent 执行次数的核心机制，防止无限循环和资源过度消耗。

**核心文件**：
- `agent/iteration_budget.py` - 预算类实现
- `agent/conversation_loop.py` - 预算消耗逻辑
- `agent/agent_init.py` - 预算初始化

---

## 2. IterationBudget 类实现

### 2.1 源码结构

```python
# agent/iteration_budget.py

class IterationBudget:
    """Thread-safe iteration counter for an agent.
    
    Each agent (parent or subagent) gets its own IterationBudget.
    The parent's budget is capped at max_iterations (default 90).
    Each subagent gets an independent budget capped at
    delegation.max_iterations (default 50) — this means total
    iterations across parent + subagents can exceed the parent's cap.
    """
    
    def __init__(self, max_total: int):
        self.max_total = max_total
        self._used = 0
        self._lock = threading.Lock()
```

### 2.2 默认配置

```python
# agent/agent_init.py

def __init__(
    self,
    max_iterations: int = 90,  # 父 agent 默认预算
    ...
):
    # 父 agent 创建预算，子 agent 继承
    agent.iteration_budget = iteration_budget or IterationBudget(max_iterations)
```

**预算分配**：
- **父 agent**: `max_iterations=90` (默认)
- **子 agent**: `delegation.max_iterations=50` (默认)
- 父子预算**独立计算**，总迭代次数可超过父预算

---

## 3. 核心方法详解

### 3.1 consume() - 消耗预算

```python
def consume(self) -> bool:
    """Try to consume one iteration. Returns True if allowed."""
    with self._lock:
        if self._used >= self.max_total:
            return False
        self._used += 1
        return True
```

**功能**：
- 尝试消耗 1 次迭代预算
- 返回 `True`：允许继续执行
- 返回 `False`：预算耗尽，应停止
- **线程安全**：使用 `threading.Lock` 保护

**使用场景**（conversation_loop.py）：

```python
# 每次 agent 循环开始时消耗预算
if agent._budget_grace_call:
    agent._budget_grace_call = False
elif not agent.iteration_budget.consume():
    _turn_exit_reason = "budget_exhausted"
    if not agent.quiet_mode:
        agent._safe_print(
            f"\n⚠️  Iteration budget exhausted "
            f"({agent.iteration_budget.used}/{agent.iteration_budget.max_total} iterations used)"
        )
    break
```

### 3.2 refund() - 退还预算

```python
def refund(self) -> None:
    """Give back one iteration (e.g. for execute_code turns)."""
    with self._lock:
        if self._used > 0:
            self._used -= 1
```

**功能**：
- 退还 1 次迭代预算
- 用于不应计入预算的操作
- **线程安全**：同样使用锁保护

**使用场景 1：压缩重启**（conversation_loop.py）：

```python
if restart_with_compressed_messages:
    api_call_count -= 1
    agent.iteration_budget.refund()  # 压缩重启不计入预算
    retry_count += 1
    restart_with_compressed_messages = False
    continue
```

**使用场景 2：execute_code 豁免**（conversation_loop.py）：

```python
# Refund the iteration if the ONLY tool(s) called were
# execute_code (programmatic tool calling). These are
# cheap RPC-style calls that shouldn't eat the budget.
_tc_names = {tc.function.name for tc in assistant_message.tool_calls}
if _tc_names == {"execute_code"}:
    agent.iteration_budget.refund()
```

### 3.3 线程安全属性

#### used - 已使用次数

```python
@property
def used(self) -> int:
    with self._lock:
        return self._used
```

- 返回已消耗的迭代次数
- 通过锁保护，确保读取一致性

#### remaining - 剩余次数

```python
@property
def remaining(self) -> int:
    with self._lock:
        return max(0, self.max_total - self._used)
```

- 返回剩余可用迭代次数
- 计算公式：`max(0, max_total - used)`
- 同样通过锁保护

---

## 4. 线程安全设计

### 4.1 为什么需要线程安全？

在并发场景下，多个线程可能同时：
- 调用 `consume()` 消耗预算
- 调用 `refund()` 退还预算
- 读取 `used` 或 `remaining` 属性

**没有锁保护的风险**：
- 数据竞争（data race）
- 部分更新（partial update）
- 读取到不一致的值

### 4.2 锁的使用

所有操作都使用 `threading.Lock` 保护：

```python
class IterationBudget:
    def __init__(self, max_total: int):
        self._lock = threading.Lock()
    
    # 所有方法都使用 with self._lock
    def consume(self) -> bool:
        with self._lock:
            # 原子操作
    
    def refund(self) -> None:
        with self._lock:
            # 原子操作
    
    @property
    def used(self) -> int:
        with self._lock:
            # 原子读取
```

### 4.3 线程安全测试

源码包含完整的并发测试（`tests/run_agent/test_iteration_budget_race.py`）：

```python
def test_iteration_budget_used_is_thread_safe():
    """Iterating used while other threads consume/refund must not crash."""
    budget = IterationBudget(max_total=1000)
    num_threads = 10
    operations_per_thread = 200
    
    def worker(consume: bool):
        for _ in range(operations_per_thread):
            if consume:
                budget.consume()
            else:
                budget.refund()
            _ = budget.used  # 同时读取属性
    
    with ThreadPoolExecutor(max_workers=num_threads * 2) as executor:
        # 一半线程 consume，一半 refund
        futures = []
        for i in range(num_threads):
            consume = i < num_threads // 2
            futures.append(executor.submit(worker, consume))
            futures.append(executor.submit(worker, consume))
        
        for f in futures:
            f.result()
    
    # 验证最终状态一致
    assert 0 <= budget.used <= budget.max_total
```

---

## 5. 实际使用流程

### 5.1 初始化

```python
# agent/conversation_loop.py - run_conversation 函数开始

def run_conversation(agent, user_message, ...):
    # 每次对话开始时重新创建预算
    agent.iteration_budget = IterationBudget(agent.max_iterations)
```

### 5.2 主循环消耗

```python
# 主循环中每次迭代消耗预算
while True:
    # Grace call: 预算耗尽后给模型最后一次机会
    if agent._budget_grace_call:
        agent._budget_grace_call = False
    elif not agent.iteration_budget.consume():
        _turn_exit_reason = "budget_exhausted"
        agent._safe_print(
            f"\n⚠️  Iteration budget exhausted "
            f"({agent.iteration_budget.used}/{agent.iteration_budget.max_total})"
        )
        break
    
    # 执行 agent 迭代...
```

### 5.3 特殊情况退还

```python
# 情况 1: 压缩重启
if restart_with_compressed_messages:
    api_call_count -= 1
    agent.iteration_budget.refund()
    continue

# 情况 2: execute_code 豁免
_tc_names = {tc.function.name for tc in assistant_message.tool_calls}
if _tc_names == {"execute_code"}:
    agent.iteration_budget.refund()
```

### 5.4 完整示例

```python
budget = IterationBudget(max_total=90)

while True:
    # 1. 尝试消耗预算
    if not budget.consume():
        print(f"预算耗尽: {budget.used}/{budget.max_total}")
        break
    
    # 2. 执行 agent 迭代
    action = agent.step()
    
    # 3. 特殊操作退还预算
    if action.type == "execute_code":
        budget.refund()
        execute_code(action.code)
    
    # 4. 显示剩余预算
    print(f"剩余预算: {budget.remaining}")
```

---

## 6. 预算配置对比

### 6.1 父子 agent 预算

| Agent 类型 | 默认预算 | 配置来源 | 说明 |
|-----------|---------|---------|------|
| 父 agent | 90 | `max_iterations` | 主 agent 的迭代上限 |
| 子 agent | 50 | `delegation.max_iterations` | 委托任务的迭代上限 |

### 6.2 预算独立性

```python
# 父 agent
parent_budget = IterationBudget(max_total=90)

# 子 agent (独立预算)
child_budget = IterationBudget(max_total=50)

# 总迭代次数 = 父预算 + 所有子预算
# 可能超过 90 次！
```

**重要特性**：
- 父子预算**完全独立**
- 子 agent 不消耗父 agent 预算
- 总迭代次数可以超过父预算上限

---

## 7. 不计入预算的操作

### 7.1 execute_code

```python
# 程序化工具调用不计入预算
if _tc_names == {"execute_code"}:
    agent.iteration_budget.refund()
```

**原因**：
- `execute_code` 是廉价的 RPC 调用
- 不应消耗宝贵的迭代预算

### 7.2 压缩重启

```python
# 上下文压缩重启不计入预算
if restart_with_compressed_messages:
    agent.iteration_budget.refund()
```

**原因**：
- 压缩是系统内部操作
- 不是真正的 agent 决策迭代

---

## 8. 测试用例

### 8.1 基本功能测试

```python
def test_iteration_budget_consume_returns_false_when_exhausted():
    """consume() must return False once the budget is exhausted."""
    budget = IterationBudget(max_total=3)
    assert budget.consume() is True
    assert budget.consume() is True
    assert budget.consume() is True
    assert budget.consume() is False  # 耗尽
```

### 8.2 退还功能测试

```python
def test_iteration_budget_refund_restores_consume():
    """refund() after consume() must allow one more consume()."""
    budget = IterationBudget(max_total=2)
    assert budget.consume() is True
    assert budget.consume() is True
    assert budget.consume() is False  # 耗尽
    budget.refund()                   # 退还
    assert budget.consume() is True   # 可以再次消耗
```

### 8.3 属性测试

```python
def test_iteration_budget_used_reflects_consume_and_refund():
    """used property must accurately reflect consume() and refund() calls."""
    budget = IterationBudget(max_total=10)
    
    assert budget.used == 0
    budget.consume()
    assert budget.used == 1
    budget.consume()
    assert budget.used == 2
    budget.refund()
    assert budget.used == 1
    budget.refund()
    assert budget.used == 0

def test_iteration_budget_remaining():
    """remaining property must equal max_total - used."""
    budget = IterationBudget(max_total=5)
    
    assert budget.remaining == 5
    budget.consume()
    assert budget.remaining == 4
    budget.consume()
    budget.consume()
    assert budget.remaining == 2
    budget.refund()
    assert budget.remaining == 3
```

---

## 9. 设计要点总结

### 9.1 核心特性

1. **线程安全**：所有操作使用 `threading.Lock` 保护
2. **独立预算**：父子 agent 各自独立计数
3. **可退还**：特殊操作可以退还预算
4. **原子操作**：consume/refund/read 都是原子的

### 9.2 使用原则

1. **每次迭代消耗**：agent 主循环每次迭代调用 `consume()`
2. **特殊操作退还**：`execute_code`、压缩重启等调用 `refund()`
3. **检查返回值**：`consume()` 返回 `False` 时立即停止
4. **显示进度**：使用 `used` 和 `remaining` 向用户展示进度

### 9.3 安全保障

1. **防止无限循环**：预算耗尽自动停止
2. **资源控制**：限制单个任务的最大执行次数
3. **并发安全**：多线程环境下正确工作
4. **测试覆盖**：完整的单元测试和并发测试

---

## 10. 与其他系统的关系

### 10.1 BudgetConfig (tools/budget_config.py)

**注意**：`BudgetConfig` 是**完全不同**的系统！

```python
# tools/budget_config.py
@dataclass(frozen=True)
class BudgetConfig:
    """工具结果持久化的预算配置（字符数限制）"""
    default_result_size: int = 100_000      # 单个结果大小
    turn_budget: int = 200_000              # 单轮总预算
    preview_size: int = 1_500               # 预览大小
```

**区别**：
- `IterationBudget`：控制**迭代次数**
- `BudgetConfig`：控制**工具结果大小**（字符数）

### 10.2 Grace Call 机制

```python
# 预算耗尽后给模型最后一次机会
if agent._budget_grace_call:
    agent._budget_grace_call = False
elif not agent.iteration_budget.consume():
    break
```

**作用**：
- 预算耗尽时不立即停止
- 给模型一次机会完成当前任务
- 避免在关键时刻中断

---

## 11. 最佳实践

### 11.1 配置建议

```python
# 简单任务
budget = IterationBudget(max_total=30)

# 标准任务（默认）
budget = IterationBudget(max_total=90)

# 复杂任务
budget = IterationBudget(max_total=150)
```

### 11.2 监控建议

```python
# 定期检查预算使用情况
if budget.remaining < 10:
    logger.warning(f"预算即将耗尽: {budget.remaining} 次剩余")

# 记录预算耗尽事件
if not budget.consume():
    logger.error(f"预算耗尽: {budget.used}/{budget.max_total}")
    telemetry.record("budget_exhausted", {
        "used": budget.used,
        "max": budget.max_total
    })
```

### 11.3 调试技巧

```python
# 开发环境：使用更大的预算
if os.getenv("DEBUG"):
    budget = IterationBudget(max_total=200)

# 记录每次消耗
original_consume = budget.consume
def logged_consume():
    result = original_consume()
    logger.debug(f"Budget consume: {budget.used}/{budget.max_total}")
    return result
budget.consume = logged_consume
```

---

## 12. 总结

Hermes Agent 的迭代预算系统是一个**简洁而强大**的资源控制机制：

✅ **线程安全**：完整的锁保护机制  
✅ **灵活退还**：支持特殊操作豁免  
✅ **独立预算**：父子 agent 各自计数  
✅ **测试完备**：包含并发测试  
✅ **易于使用**：简单的 API 设计  

通过这个系统，Hermes Agent 能够有效防止无限循环，同时保持足够的灵活性来处理复杂任务。
