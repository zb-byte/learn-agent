# Context Compression 机制深度解析

## 概述

Context Compression（上下文压缩）是 Hermes Agent 应对长对话的核心机制。当对话历史超过模型上下文窗口的阈值时，自动将中间的对话回合压缩成结构化摘要，保留头部和尾部的完整上下文，确保对话可以无限延续。

## 核心设计理念

- **自动触发**：基于 token 数量自动判断是否需要压缩
- **保护关键上下文**：头部（系统提示 + 初始交互）和尾部（最近的对话）完整保留
- **结构化摘要**：使用辅助模型生成结构化的中间回合摘要
- **迭代更新**：后续压缩会更新之前的摘要，而非从头开始
- **工具对完整性**：确保 tool_call 和 tool_result 配对完整

---

## 系统组件

### 1. ContextCompressor（压缩器）

**位置**：`agent/context_compressor.py`

**职责**：
- 判断是否需要压缩（`should_compress`）
- 执行压缩算法（`compress`）
- 管理压缩状态和历史摘要
- 处理工具调用对的完整性

**关键属性**：
```python
class ContextCompressor:
    model: str                    # 主模型
    context_length: int           # 上下文窗口大小
    threshold_percent: float      # 触发阈值百分比（默认 0.50）
    threshold_tokens: int         # 触发阈值 token 数
    protect_first_n: int          # 保护头部消息数（默认 3）
    protect_last_n: int           # 保护尾部消息数（默认 20）
    summary_target_ratio: float   # 摘要目标比例（默认 0.20）
    tail_token_budget: int        # 尾部 token 预算
    summary_model: str            # 摘要模型（可选）
    _previous_summary: str        # 上次压缩的摘要（用于迭代更新）
```

### 2. 触发检查器（Trigger Checker）

**位置**：`agent/context_compressor.py:should_compress`

**判断逻辑**：
```python
def should_compress(self, prompt_tokens: int = None) -> bool:
    tokens = prompt_tokens or self.last_prompt_tokens
    
    # 1. Token 数未达到阈值
    if tokens < self.threshold_tokens:
        return False
    
    # 2. 反抖动保护：最近 2 次压缩效果都 <10%
    if self._ineffective_compression_count >= 2:
        logger.warning("Compression skipped — last 2 compressions saved <10% each")
        return False
    
    return True
```

**阈值计算**：
```python
threshold_tokens = max(
    int(context_length * threshold_percent),
    MINIMUM_CONTEXT_LENGTH  # 64K tokens
)
```

**示例**：
- 200K 上下文窗口 × 50% = 100K tokens 触发
- 32K 上下文窗口 × 50% = 16K tokens，但最低 64K，所以不会触发

### 3. 边界保护器（Boundary Protector）

**职责**：确定哪些消息需要保护，哪些可以压缩

#### 头部保护（Head Protection）

**位置**：`agent/context_compressor.py:_protect_head_size`

```python
def _protect_head_size(self, messages: List[Dict]) -> int:
    head = 0
    if messages and messages[0].get("role") == "system":
        head = 1  # 系统提示总是保护
    return head + self.protect_first_n
```

**保护内容**：
- 系统提示（总是保护）
- 前 N 条非系统消息（默认 3 条）

**为什么保护头部**：
- 系统提示包含工具定义、memory、技能等核心上下文
- 初始交互通常包含任务目标和约束

#### 尾部保护（Tail Protection）

**位置**：`agent/context_compressor.py:_find_tail_cut_by_tokens`

**策略**：基于 token 预算，而非固定消息数

```python
def _find_tail_cut_by_tokens(self, messages, head_end, token_budget=None):
    if token_budget is None:
        token_budget = self.tail_token_budget  # 默认 = threshold * 0.20
    
    # 从后往前累积 token
    accumulated = 0
    cut_idx = len(messages)
    
    for i in range(len(messages) - 1, head_end - 1, -1):
        msg_tokens = estimate_message_tokens(messages[i])
        
        # 超过预算且已保护最少 3 条消息
        if accumulated + msg_tokens > token_budget * 1.5 and (len(messages) - i) >= 3:
            break
        
        accumulated += msg_tokens
        cut_idx = i
    
    # 确保最后一条用户消息在尾部（防止任务丢失）
    cut_idx = self._ensure_last_user_message_in_tail(messages, cut_idx, head_end)
    
    return cut_idx
```

**保护内容**：
- 最近的 ~20K tokens 对话（根据上下文窗口动态调整）
- 最后一条用户消息（必须保护，防止任务丢失）
- 至少 3 条消息（硬性最小值）

**为什么基于 token 而非消息数**：
- 消息长度差异巨大（一条 terminal 输出可能 10K tokens）
- Token 预算更精确地控制压缩后的大小
- 自动适应不同上下文窗口的模型

### 4. 工具对完整性保护器（Tool Pair Sanitizer）

**位置**：`agent/context_compressor.py:_sanitize_tool_pairs`

**问题**：压缩可能破坏 tool_call 和 tool_result 的配对

**场景 1**：孤儿 tool_result
```
[HEAD]
assistant: tool_calls=[{id: "call_123", name: "read_file"}]  ← 被压缩掉
tool: tool_call_id="call_123", content="..."                 ← 孤儿！
[TAIL]
```

**场景 2**：孤儿 tool_call
```
[HEAD]
assistant: tool_calls=[{id: "call_456", name: "terminal"}]   ← 保留
[MIDDLE - 被压缩]
tool: tool_call_id="call_456", content="..."                 ← 被压缩掉
[TAIL]
```

**修复策略**：
```python
def _sanitize_tool_pairs(self, messages):
    # 1. 收集所有存活的 tool_call_id
    surviving_call_ids = {tc.id for msg in messages 
                          if msg.role == "assistant" 
                          for tc in msg.tool_calls}
    
    # 2. 移除孤儿 tool_result
    messages = [m for m in messages 
                if not (m.role == "tool" and m.tool_call_id not in surviving_call_ids)]
    
    # 3. 为孤儿 tool_call 插入占位 result
    for msg in messages:
        if msg.role == "assistant":
            for tc in msg.tool_calls:
                if tc.id not in result_call_ids:
                    messages.insert_after(msg, {
                        "role": "tool",
                        "tool_call_id": tc.id,
                        "content": "[Result from earlier conversation — see context summary]"
                    })
    
    return messages
```

### 5. 摘要生成器（Summary Generator）

**位置**：`agent/context_compressor.py:_generate_summary`

**使用模型**：
- 优先使用 `summary_model`（配置的辅助模型，通常是便宜快速的模型）
- 失败时回退到主模型
- 使用 `auxiliary_client` 调用（独立的 API 客户端）

**摘要结构**：
```markdown
## Active Task
[最重要！用户最近的未完成请求，原话引用]

## Goal
[用户的总体目标]

## Constraints & Preferences
[用户偏好、编码风格、约束、重要决策]

## Completed Actions
[已完成的具体操作，带工具名称和结果]
1. READ config.py:45 — found `==` should be `!=` [tool: read_file]
2. PATCH config.py:45 — changed `==` to `!=` [tool: patch]

## Active State
[当前工作状态：目录、分支、修改的文件、测试状态]

## In Progress
[正在进行的工作]

## Blocked
[阻塞项、错误、未解决的问题]

## Key Decisions
[重要技术决策及原因]

## Resolved Questions
[已回答的问题及答案]

## Pending User Asks
[用户提出但未回答的问题]

## Relevant Files
[读取、修改、创建的文件]

## Remaining Work
[剩余工作]

## Critical Context
[关键值、错误消息、配置细节]
```

**迭代更新**：
```python
if self._previous_summary:
    # 有旧摘要 → 迭代更新
    prompt = f"""
    PREVIOUS SUMMARY:
    {self._previous_summary}
    
    NEW TURNS TO INCORPORATE:
    {new_content}
    
    Update the summary:
    - PRESERVE existing info
    - ADD new completed actions (continue numbering)
    - Move "In Progress" → "Completed Actions" when done
    - Update "Active Task" to latest unfulfilled request
    """
else:
    # 首次压缩 → 从头生成
    prompt = f"""
    Create a structured checkpoint summary.
    
    TURNS TO SUMMARIZE:
    {content}
    """
```

**摘要预算**：
```python
def _compute_summary_budget(self, turns_to_summarize):
    content_tokens = estimate_tokens(turns_to_summarize)
    budget = int(content_tokens * 0.20)  # 20% 的压缩内容
    return max(2000, min(budget, 12000))  # 2K-12K tokens
```

### 6. 工具输出修剪器（Tool Output Pruner）

**位置**：`agent/context_compressor.py:_prune_old_tool_results`

**目的**：在 LLM 摘要之前，先用廉价的规则修剪旧工具输出

**策略**：
```python
def _prune_old_tool_results(self, messages, protect_tail_count, protect_tail_tokens):
    # Pass 1: 去重相同的工具结果
    content_hashes = {}
    for i in reversed(range(len(messages))):
        if messages[i].role == "tool":
            h = md5(messages[i].content)
            if h in content_hashes:
                messages[i].content = "[Duplicate tool output — same as recent call]"
            else:
                content_hashes[h] = i
    
    # Pass 2: 替换旧工具结果为信息性摘要
    for i in range(prune_boundary):
        if messages[i].role == "tool" and len(messages[i].content) > 200:
            summary = _summarize_tool_result(tool_name, tool_args, content)
            messages[i].content = summary
    
    # Pass 3: 截断大型 tool_call 参数
    for i in range(prune_boundary):
        if messages[i].role == "assistant":
            for tc in messages[i].tool_calls:
                if len(tc.arguments) > 500:
                    tc.arguments = _truncate_tool_call_args_json(tc.arguments)
    
    return messages
```

**工具结果摘要示例**：
```python
# 原始（3000 chars）
{"exit_code": 0, "stdout": "test_auth.py::test_login PASSED\ntest_auth.py::test_logout PASSED\n..."}

# 摘要（80 chars）
"[terminal] ran `pytest tests/` -> exit 0, 47 lines output"
```

---

## 压缩流程

### 完整流程图

```
用户消息 → API 调用
  ↓
收到响应，更新 last_prompt_tokens
  ↓
检查：prompt_tokens >= threshold_tokens?
  ├─ No → 继续对话
  └─ Yes → 触发压缩
      ↓
  1. 工具输出修剪（廉价预处理）
      ├─ 去重相同工具结果
      ├─ 替换旧结果为摘要
      └─ 截断大型参数
      ↓
  2. 确定边界
      ├─ 头部：系统提示 + 前 3 条消息
      ├─ 尾部：最近 ~20K tokens（至少 3 条）
      └─ 中间：需要压缩的部分
      ↓
  3. 生成摘要（调用辅助模型）
      ├─ 首次：从头生成结构化摘要
      └─ 后续：迭代更新之前的摘要
      ↓
  4. 组装压缩后的消息列表
      ├─ 头部消息（原样）
      ├─ 摘要消息（新插入）
      └─ 尾部消息（原样）
      ↓
  5. 修复工具对完整性
      ├─ 移除孤儿 tool_result
      └─ 为孤儿 tool_call 插入占位 result
      ↓
  6. 剥离历史图片
      └─ 替换旧图片为占位符
      ↓
返回压缩后的消息列表
  ↓
继续对话
```

### 触发时机

**位置**：`agent/conversation_loop.py:3398`

```python
# 在每次 API 调用后检查
if agent.compression_enabled and _compressor.should_compress(_real_tokens):
    agent._safe_print("  ⟳ compacting context…")
    messages, active_system_prompt = agent._compress_context(
        messages, system_message,
        approx_tokens=agent.context_compressor.last_prompt_tokens,
        task_id=effective_task_id,
    )
```

**检查点**：
- 每次 API 调用返回后
- 在处理工具调用之前
- 在保存会话之前

**Token 计算**：
```python
if _compressor.last_prompt_tokens > 0:
    # 优先使用上次 API 返回的 prompt_tokens
    _real_tokens = _compressor.last_prompt_tokens
else:
    # 回退到估算（包含工具定义）
    _real_tokens = estimate_request_tokens_rough(messages, tools=agent.tools)
```

**为什么只用 prompt_tokens**：
- `completion_tokens` 不占用上下文窗口
- 思考模型（DeepSeek R1、QwQ）的 reasoning tokens 会虚高
- 只有 prompt 部分才会累积导致上下文溢出

### 压缩策略

#### 策略 1：工具输出修剪（Cheap Pre-pass）

**时机**：在 LLM 摘要之前

**操作**：
1. 去重相同内容的工具结果
2. 替换大型工具输出为信息性摘要
3. 截断大型 tool_call 参数

**收益**：
- 无 API 成本
- 可节省 20-40% 的 tokens
- 保留关键信息（命令、文件路径、退出码）

#### 策略 2：结构化摘要（LLM Summarization）

**时机**：工具修剪后，仍需进一步压缩

**操作**：
1. 序列化中间回合为文本
2. 调用辅助模型生成结构化摘要
3. 插入摘要消息

**收益**：
- 高压缩比（10:1 或更高）
- 保留语义和上下文
- 结构化便于模型理解

#### 策略 3：迭代更新（Iterative Summary）

**时机**：第二次及以后的压缩

**操作**：
1. 复用上次的摘要
2. 只摘要新增的中间回合
3. 更新摘要的相应部分

**收益**：
- 避免信息丢失
- 摘要质量随时间提升
- 减少摘要模型的输入

---

## 配置项

### 核心配置

**位置**：`~/.hermes/config.toml`

```toml
[compression]
# 是否启用自动压缩（默认 true）
enabled = true

# 触发阈值：上下文窗口的百分比（默认 0.50 = 50%）
threshold = 0.50

# 摘要目标比例：压缩内容的百分比（默认 0.20 = 20%）
target_ratio = 0.20

# 保护头部消息数（系统提示之外，默认 3）
protect_first_n = 3

# 保护尾部消息数（已废弃，现在用 token 预算）
protect_last_n = 20

# 摘要失败时是否中止压缩（默认 false）
# true: 保留所有消息，冻结对话直到手动 /compress
# false: 插入静态占位符，丢弃中间消息
abort_on_summary_failure = false
```

### 辅助模型配置

```toml
[auxiliary]
# 摘要模型（留空使用主模型）
summary_model = "anthropic/claude-haiku-3.5"

# 摘要超时（毫秒，默认 60000）
[auxiliary.compression]
timeout = 60000
```

### 模型特定阈值

**位置**：`agent/auxiliary_client.py:_compression_threshold_for_model`

```python
def _compression_threshold_for_model(model: str) -> Optional[float]:
    # 小上下文模型：更早触发
    if "gpt-3.5" in model or "llama-3.1-8b" in model:
        return 0.40  # 40%
    
    # 大上下文模型：更晚触发
    if "claude-3" in model or "gpt-4" in model:
        return 0.60  # 60%
    
    return None  # 使用默认值
```

### 环境变量

```bash
# 禁用压缩（调试用）
export HERMES_COMPRESSION_DISABLED=1

# 强制使用主模型摘要（不用辅助模型）
export HERMES_COMPRESSION_USE_MAIN_MODEL=1
```

---

## 压缩效果

### 压缩比示例

**场景 1：工具密集型对话**
```
压缩前：150 条消息，~120K tokens
  ├─ 头部：4 条消息（系统提示 + 3 条）
  ├─ 中间：136 条消息（大量 terminal/read_file 输出）
  └─ 尾部：10 条消息（最近 ~18K tokens）

压缩后：15 条消息，~35K tokens
  ├─ 头部：4 条消息（原样）
  ├─ 摘要：1 条消息（~8K tokens）
  └─ 尾部：10 条消息（原样）

节省：~85K tokens（71%）
```

**场景 2：对话密集型**
```
压缩前：80 条消息，~95K tokens
  ├─ 头部：4 条消息
  ├─ 中间：66 条消息（主要是对话，少量工具）
  └─ 尾部：10 条消息

压缩后：15 条消息，~45K tokens
  ├─ 头部：4 条消息
  ├─ 摘要：1 条消息（~12K tokens）
  └─ 尾部：10 条消息

节省：~50K tokens（53%）
```

### 反抖动保护

**问题**：压缩效果不佳时，可能陷入无限循环

**场景**：
```
第 1 次压缩：120K → 110K（节省 8%）
第 2 次压缩：110K → 105K（节省 5%）
第 3 次压缩：105K → 103K（节省 2%）
...（每次只节省一点点）
```

**保护机制**：
```python
if self._ineffective_compression_count >= 2:
    logger.warning("Compression skipped — last 2 compressions saved <10% each")
    return False  # 停止自动压缩
```

**用户操作**：
- `/new` - 开始新会话
- `/compress <topic>` - 手动聚焦压缩

---

## 关键设计决策

### 1. 为什么保护尾部用 token 预算而非消息数？

**问题**：消息长度差异巨大

```
消息 1: "好的" (2 tokens)
消息 2: terminal 输出 (15K tokens)
消息 3: read_file 结果 (8K tokens)
```

如果固定保护 20 条消息：
- 可能只保护了 5K tokens（20 条短消息）
- 也可能保护了 200K tokens（20 条长消息）

**解决方案**：基于 token 预算

```python
tail_token_budget = threshold_tokens * summary_target_ratio
# 例如：100K 阈值 × 0.20 = 20K tokens 尾部预算
```

**收益**：
- 压缩后大小可预测
- 自动适应不同模型的上下文窗口
- 避免过度保护或保护不足

### 2. 为什么要确保最后一条用户消息在尾部？

**问题**：`_align_boundary_backward` 可能把用户消息推到中间

**场景**：
```
[HEAD]
...
user: "现在重构 auth 模块改用 JWT"          ← 最后的用户消息
assistant: tool_calls=[read_file, ...]
tool: result 1
tool: result 2
tool: result 3                              ← 边界对齐到这里
[TAIL]
assistant: "好的，我来..."
```

**后果**：
- 用户消息被压缩到摘要的 "Pending User Asks"
- 但 `SUMMARY_PREFIX` 说"只响应摘要之后的消息"
- 模型看不到活跃任务，陷入停滞或重复已完成的工作

**解决方案**：
```python
def _ensure_last_user_message_in_tail(self, messages, cut_idx, head_end):
    last_user_idx = self._find_last_user_message_idx(messages, head_end)
    if last_user_idx >= cut_idx:
        return cut_idx  # 已在尾部
    
    # 拉回边界，确保用户消息在尾部
    return max(last_user_idx, head_end + 1)
```

**参考**：Issue #10896

### 3. 为什么要迭代更新摘要而非每次从头生成？

**问题**：多次压缩会丢失信息

**场景**：
```
第 1 次压缩：回合 1-100 → 摘要 A
第 2 次压缩：回合 1-200 → 摘要 B（重新摘要 1-100，可能丢失 A 中的细节）
第 3 次压缩：回合 1-300 → 摘要 C（重新摘要 1-200，可能丢失 B 中的细节）
```

**解决方案**：迭代更新

```
第 1 次压缩：回合 1-100 → 摘要 A
第 2 次压缩：摘要 A + 回合 101-200 → 摘要 B（保留 A，添加新内容）
第 3 次压缩：摘要 B + 回合 201-300 → 摘要 C（保留 B，添加新内容）
```

**收益**：
- 信息累积而非丢失
- 摘要质量随时间提升
- 减少摘要模型的输入量

### 4. 为什么要修剪工具输出而非直接摘要？

**对比**：

**方案 A：直接 LLM 摘要**
```
成本：每次压缩调用 LLM
时间：2-5 秒
效果：高质量摘要
```

**方案 B：规则修剪 + LLM 摘要**
```
成本：规则修剪免费，LLM 只处理剩余部分
时间：规则修剪 <100ms，LLM 1-3 秒
效果：规则修剪 20-40%，LLM 再压缩 50-70%
```

**选择**：方案 B（两阶段）

**原因**：
- 工具输出有明确的模式（命令、文件路径、退出码）
- 规则可以提取关键信息，丢弃冗余输出
- 减少 LLM 输入，降低成本和延迟
- 保留的信息更精确（不会被 LLM 改写）

### 5. 为什么要去重工具结果？

**场景**：反复读取同一个文件

```
回合 10: read_file("config.py") → 3000 chars
回合 25: read_file("config.py") → 3000 chars（内容相同）
回合 40: read_file("config.py") → 3000 chars（内容相同）
```

**不去重**：9000 chars 占用上下文

**去重后**：
```
回合 10: read_file("config.py") → 3000 chars（保留）
回合 25: "[Duplicate tool output — same as recent call]"
回合 40: "[Duplicate tool output — same as recent call]"
```

**收益**：
- 节省 6000 chars
- 保留最新的完整副本
- 模型仍然知道"读取了多次"

---

## 故障处理

### 1. 摘要生成失败

**原因**：
- 辅助模型不可用（404、503）
- 超时
- 返回非 JSON 响应
- 网络错误

**处理策略**：

#### 策略 A：回退到主模型

```python
if summary_model and summary_model != main_model:
    try:
        summary = call_llm(model=summary_model, ...)
    except Exception as e:
        logger.warning("Summary model failed, falling back to main model")
        summary_model = ""  # 清空，下次用主模型
        summary = call_llm(model=main_model, ...)
```

**适用**：模型不可用、超时、非 JSON 响应

#### 策略 B：冷却期

```python
except Exception as e:
    self._summary_failure_cooldown_until = time.monotonic() + 60
    logger.warning("Summary failed, pausing for 60s")
    return None
```

**适用**：瞬态错误（网络、限流）

#### 策略 C：中止压缩（可选）

```python
if not summary and self.abort_on_summary_failure:
    logger.warning("Summary failed — aborting compression")
    self._last_compress_aborted = True
    return messages  # 原样返回，不压缩
```

**配置**：`compression.abort_on_summary_failure = true`

**适用**：用户希望保留所有上下文，宁可冻结对话

#### 策略 D：静态占位符（默认）

```python
if not summary:
    summary = f"""
    {SUMMARY_PREFIX}
    Summary generation was unavailable. {n_dropped} message(s) were
    removed to free context space but could not be summarized.
    Continue based on recent messages and current file state.
    """
```

**适用**：摘要失败但仍需释放空间

### 2. 压缩效果不佳

**症状**：
```
压缩前：120K tokens
压缩后：115K tokens（只节省 5K，4%）
```

**原因**：
- 尾部保护过多（token 预算过大）
- 中间回合太少（只有 5-10 条消息）
- 摘要过长（摘要本身占用太多 tokens）

**保护机制**：
```python
if self._ineffective_compression_count >= 2:
    logger.warning("Compression skipped — last 2 compressions saved <10% each")
    return False  # 停止自动压缩
```

**用户操作**：
- `/new` - 开始新会话（推荐）
- `/compress <topic>` - 聚焦压缩特定主题
- 调整配置：降低 `target_ratio`，减少 `protect_first_n`

### 3. 工具对不匹配

**症状**：API 返回错误

```
Error: No tool call found for function call output with call_id "call_123"
```

**原因**：压缩破坏了 tool_call 和 tool_result 的配对

**保护机制**：
```python
compressed = self._sanitize_tool_pairs(compressed)
```

**自动修复**：
- 移除孤儿 tool_result
- 为孤儿 tool_call 插入占位 result

### 4. 上下文溢出

**症状**：即使压缩后仍然超过上下文窗口

**原因**：
- 尾部保护过多
- 摘要过长
- 系统提示过大

**处理**：
```python
# 错误分类器识别上下文溢出
if classified.reason == FailoverReason.context_overflow:
    if classified.should_compress:
        # 触发压缩
        messages = compress_context(messages)
    else:
        # 已经压缩过，无法再压缩
        return "Context window exhausted. Please start a new session with /new"
```

---

## 调试技巧

### 1. 查看压缩日志

```bash
# 启用 verbose 模式
export HERMES_VERBOSE=1

# 查看压缩触发和效果
tail -f ~/.hermes/agent.log | grep -E "compress|Context compression"
```

**示例输出**：
```
Context compression triggered (105234 tokens >= 100000 threshold)
Model context limit: 200000 tokens (50% = 100000)
Summarizing turns 5-145 (140 turns), protecting 4 head + 10 tail messages
Pre-compression: pruned 23 old tool result(s)
Compressed: 150 -> 15 messages (~85000 tokens saved, 71%)
Compression #1 complete
```

### 2. 手动触发压缩

```bash
# CLI 中使用 /compress
/compress

# 聚焦压缩特定主题
/compress authentication refactoring
```

### 3. 检查压缩状态

```python
# 在代码中
compressor = agent.context_compressor
print(f"Context length: {compressor.context_length}")
print(f"Threshold: {compressor.threshold_tokens}")
print(f"Last prompt tokens: {compressor.last_prompt_tokens}")
print(f"Should compress: {compressor.should_compress()}")
print(f"Compression count: {compressor.compression_count}")
print(f"Ineffective count: {compressor._ineffective_compression_count}")
```

### 4. 禁用压缩（调试用）

```toml
# config.toml
[compression]
enabled = false
```

或

```bash
export HERMES_COMPRESSION_DISABLED=1
```

### 5. 查看摘要内容

**位置**：压缩后的消息列表中

```python
for msg in messages:
    if msg.get("role") in ["user", "assistant"]:
        content = msg.get("content", "")
        if content.startswith(SUMMARY_PREFIX):
            print("=== SUMMARY ===")
            print(content)
```

---

## 性能优化

### 1. 使用辅助模型

**配置**：
```toml
[auxiliary]
summary_model = "anthropic/claude-haiku-3.5"
```

**收益**：
- 成本：Haiku 比 Opus 便宜 ~50x
- 速度：Haiku 比 Opus 快 ~3x
- 质量：摘要任务 Haiku 足够好

**对比**：
```
使用 Opus 摘要：
- 成本：$0.15 per 1M input tokens
- 时间：3-5 秒
- 质量：优秀

使用 Haiku 摘要：
- 成本：$0.003 per 1M input tokens（便宜 50x）
- 时间：1-2 秒（快 2-3x）
- 质量：良好（摘要任务足够）
```

### 2. 工具输出修剪

**收益**：
- 无 API 成本
- 节省 20-40% tokens
- 减少摘要模型输入

**示例**：
```
修剪前：150 条消息，120K tokens
修剪后：150 条消息，75K tokens（节省 45K）
摘要输入：75K tokens（而非 120K）
```

### 3. 迭代摘要

**收益**：
- 减少摘要模型输入
- 提升摘要质量

**对比**：
```
从头摘要：
- 输入：回合 1-200（100K tokens）
- 输出：摘要（8K tokens）

迭代摘要：
- 输入：上次摘要（8K）+ 新回合 101-200（50K）= 58K tokens
- 输出：更新摘要（10K tokens）
- 节省输入：42K tokens
```

---

## 相关文件索引

### 核心实现
- `agent/context_compressor.py` - 压缩器主逻辑
- `agent/conversation_compression.py` - 压缩入口和协调
- `agent/conversation_loop.py:3398` - 触发检查
- `agent/auxiliary_client.py` - 辅助模型调用

### 配置和初始化
- `agent/agent_init.py:1183-1415` - 压缩器初始化
- `hermes_cli/config.py` - 配置加载

### 工具和辅助
- `agent/model_metadata.py` - Token 估算
- `agent/redact.py` - 敏感信息脱敏
- `agent/error_classifier.py` - 上下文溢出检测

### 测试文件
- `tests/agent/test_context_compressor.py`
- `tests/agent/test_compression_tool_pairs.py`
- `tests/agent/test_compression_iterative.py`

---

## 总结

Context Compression 是 Hermes Agent 实现无限对话的核心机制，通过以下手段实现了高效、可靠的上下文管理：

1. **智能触发**：基于 token 阈值自动判断，反抖动保护防止无效循环
2. **精确保护**：头部保留核心上下文，尾部基于 token 预算动态保护
3. **两阶段压缩**：规则修剪（廉价）+ LLM 摘要（高质量）
4. **迭代更新**：摘要累积而非重建，信息不丢失
5. **完整性保证**：自动修复工具对，确保 API 兼容性
6. **故障恢复**：多层回退策略，摘要失败不阻塞对话
7. **性能优化**：辅助模型降低成本，工具修剪减少输入

这个机制让 Agent 能够处理数百回合的长对话，同时保持响应速度和成本可控。

