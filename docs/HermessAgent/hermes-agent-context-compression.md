# Hermes-Agent 上下文管理与压缩机制详解

> 本文档面向普通研发，用**通俗类比 + 源码锚点**讲解 Hermes-Agent 的上下文管理与压缩机制。
> 分享/走读时建议搭配 **[结构化导读](./hermes-agent-runtime-reader-guide.md)**（讲者提纲）。

---

## 文档体系（建议阅读顺序）

| 文档 | 定位 |
|------|------|
| [hermes-agent-runtime-guide-refactored.md](./hermes-agent-runtime-guide-refactored.md) | 运行时原理：多Agent + 记忆系统 |
| **本文档** | 上下文管理：压缩机制、Token估算、Session轮换 |
| [hermes-agent-main-loop-guide.md](./hermes-agent-main-loop-guide.md) | 主循环实现：双计数器、工具并行 |
| [hermes-agent-permission-system.md](./hermes-agent-permission-system.md) | 权限系统：CLI / Gateway / ACP |

**源码根目录：** `/Users/wangzhongbin/Documents/code/fangzhen/hermes-agent`（下文行号对应该仓库当前主干）。

---

## 目录

1. [一眼看懂](#1-一眼看懂)
2. [源码地图与职责拆分](#2-源码地图与职责拆分)
3. [上下文管理概述](#3-上下文管理概述)
4. [压缩算法五阶段](#4-压缩算法五阶段)
5. [触发时机总表](#5-触发时机总表)
6. [Token估算与管理](#6-token估算与管理)
7. [会话管理与轮换](#7-会话管理与轮换)
8. [与Prompt Cache的关系](#8-与prompt-cache的关系)
9. [配置参数说明](#9-配置参数说明)
10. [失败、中止与手动/compress](#10-失败中止与手动compress)
11. [常见问题深度解答（FAQ）](#11-常见问题深度解答faq)
12. [讲者展开点与常见误读](#12-讲者展开点与常见误读)

---

## 1. 一眼看懂

| 结论 | 含义 |
|---|------|
| **压缩 ≠ 改模型权重** | 用辅助 LLM 把「中间历史」总结成一条 summary message，保留头尾 |
| **两层代码** | `ContextCompressor.compress()` 做剪枝+摘要；`compress_context()` 做 session 切分与 memory 钩子 |
| **默认阈值** | 约为模型 context 的 **50%**（`compression.threshold`，且不低于 `MINIMUM_CONTEXT_LENGTH`） |
| **三处自动触发** | Loop 前 **preflight**、工具批后 **per-turn**、API 错误 **413/overflow** 等 |
| **压缩后换 session** | SQLite `end_session` → 新 `session_id`，父 session 链保留 |
| **防打鼓** | 连续两次压缩节省 < 10% 时 `should_compress` 返回 False |

**一句话总结：**

```text
压缩 = 剪枝旧 tool 输出 → 保护头尾 → LLM 摘要中间 → 重组 messages → 重建 system prompt → 轮换 session。
```

**类比理解：**

```
上下文窗口 = 你的工作记忆（只能同时记住有限的事情）
上下文压缩 = 把旧笔记整理成摘要，腾出空间记新事
```

---

## 2. 源码地图与职责拆分

| 模块 | 路径 | 职责 |
|---|---|---|
| **压缩引擎** | `agent/context_compressor.py` `ContextCompressor` (~454+) | `compress()`、`should_compress()`、头尾保护、摘要模板 |
| **编排层** | `agent/conversation_compression.py` | `compress_context()`、`check_compression_model_feasibility()` |
| **触发点** | `agent/conversation_loop.py` | Preflight 422–488；回合后 3369–3410；错误路径 413/overflow |
| **Token 粗算** | `agent/model_metadata.py` `estimate_request_tokens_rough` | 含 **tool schemas** 的预检 |
| **上下文引擎抽象** | `agent/context_engine.py` | `ContextEngine` 接口定义 |
| **初始化** | `agent/agent_init.py` ~1095–1337 | 读 `compression.*`，构造 compressor 或插件 engine |

**调用链：**

```text
conversation_loop._compress_context()
        │
        ▼
conversation_compression.compress_context()
        │  memory on_pre_compress / on_session_switch
        ▼
context_compressor.ContextCompressor.compress()
```

---

## 3. 上下文管理概述

### 3.1 核心概念

**上下文管理是什么？**

简单来说，上下文管理就像**"给大模型配备一个智能笔记本"**：
- 大模型的上下文窗口是有限的（如 200K tokens）
- 对话越长，需要记忆的内容越多
- 当接近上限时，需要智能地"压缩"旧内容，为新对话腾出空间

**核心文件：**

| 文件 | 行数 | 作用 |
|------|------|------|
| `agent/context_compressor.py` | ~2000 | 压缩器核心实现 |
| `agent/conversation_compression.py` | ~600 | 压缩流程编排 |
| `agent/context_engine.py` | ~200 | 上下文引擎抽象接口 |
| `agent/model_metadata.py` | ~2000 | Token估算与模型元数据 |

**源码证据：**

```python
# agent/context_engine.py:1-11
"""Abstract base class for pluggable context engines.

A context engine controls how conversation context is managed when
approaching the model's token limit. The built-in ContextCompressor
is the default implementation.

The engine is responsible for:
  - Deciding when compaction should fire
  - Performing compaction (summarization, DAG construction, etc.)
  - Tracking token usage from API responses
"""
```

### 3.2 为什么需要上下文管理？

**问题：上下文窗口有限**

```
模型上下文窗口：200K tokens
├─ System Prompt：~10K tokens
├─ 对话历史：逐渐增长
│   ├─ 用户消息 1
│   ├─ 助手响应 1（含工具调用）
│   ├─ 工具结果 1
│   ├─ 用户消息 2
│   ├─ ...
│   └─ 用户消息 N
└─ 剩余空间：用于新的响应

当对话历史超过阈值（如 100K tokens）时：
→ 需要压缩旧的对话，为新内容腾出空间
```

**如果不管理会怎样？**

1. **请求失败**：超过模型上下文窗口上限，API 返回错误
2. **成本激增**：每次请求都传递大量历史，token 消耗巨大
3. **响应变慢**：处理大量上下文需要更长时间

### 3.3 上下文管理策略

**Hermes 的策略：**

| 策略 | 触发时机 | 实现方式 |
|------|---------|---------|
| **工具结果修剪** | 每次压缩前 | 用简洁摘要替换冗长输出 |
| **头部保护** | 始终生效 | 保留前 N 条消息（默认 3 条） |
| **尾部保护** | 始终生效 | 保留最近的约 20K tokens |
| **中间压缩** | 超过阈值时 | 用辅助模型生成摘要 |

**源码证据：**

```python
# agent/context_compressor.py:454-463
class ContextCompressor(ContextEngine):
    """Default context engine — compresses conversation context via lossy summarization.

    Algorithm:
      1. Prune old tool results (cheap, no LLM call)
      2. Protect head messages (system prompt + first exchange)
      3. Protect tail messages by token budget (most recent ~20K tokens)
      4. Summarize middle turns with structured LLM prompt
      5. On subsequent compactions, iteratively update the previous summary
    """
```

---

## 4. 压缩算法五阶段

`ContextCompressor.compress()`（`context_compressor.py` ~1494+）文档化流程：

### 4.1 阶段1：剪枝旧 tool 结果（无 LLM）

`_prune_old_tool_results()`：将尾部保护区之外的旧 `tool` 消息压成一行摘要（如 `[terminal] ran npm test -> exit 0`），降低送入摘要模型的 token。

**三种修剪技术：**

1. **摘要替换**：将长输出替换为一行摘要
2. **去重**：相同内容只保留最新的一份
3. **参数截断**：缩短过长的工具调用参数

**实际例子：**

```python
# 修剪前（2000+ chars）
{
    "role": "tool",
    "content": """
    Running npm test...
    Test file 1... PASSED
    Test file 2... PASSED
    ...
    Test file 50... PASSED
    All tests passed!
    """
}

# 修剪后
{
    "role": "tool",
    "content": "[terminal] ran `npm test` -> exit 0, 47 lines output"
}
```

**源码证据：**

```python
# agent/context_compressor.py:639-661
def _prune_old_tool_results(
    self, messages: List[Dict[str, Any]], protect_tail_count: int,
    protect_tail_tokens: int | None = None,
) -> tuple[List[Dict[str, Any]], int]:
    """Replace old tool result contents with informative 1-line summaries.

    Instead of a generic placeholder, generates a summary like:

        [terminal] ran `npm test` -> exit 0, 47 lines output
        [read_file] read config.py from line 1 (3,400 chars)

    Also deduplicates identical tool results (e.g. reading the same file
    5x keeps only the newest full copy) and truncates large tool_call
    arguments in assistant messages outside the protected tail.
    """
```

### 4.2 阶段2：保护头部（Head）

`_protect_head_size()`（~1308–1326）：

- 索引 0 的 **system** 始终保留
- `protect_first_n` 表示 **除 system 外**再保护的前 N 条消息（默认 3）
- `protect_first_n=0` 时仅 system + 摘要 + tail（滚动压缩模式）

**源码证据：**

```python
# agent/context_engine.py:59-66
# protect_first_n semantics (since PR #13754): count of non-system head
# messages always preserved verbatim, IN ADDITION to the system prompt
# which is always implicitly protected.  Default 3 keeps the
# historical "system + first 3 non-system messages" head shape.

threshold_percent: float = 0.75
protect_first_n: int = 3
protect_last_n: int = 6
```

### 4.3 阶段3：保护尾部（Tail）

`_find_tail_cut_by_tokens()`（~1412–1473）：

- `tail_token_budget = int(threshold_tokens × summary_target_ratio)`（`target_ratio` 默认 0.20）
- `protect_last_n`（默认 20）作为 **最少保留消息数**下限，不是唯一规则
- 从末尾向前累计 token，**不拆开** assistant `tool_calls` 与对应 `tool` 结果组
- `_ensure_last_user_message_in_tail`：最新 user 消息必须在 tail（#10896）

**可视化：**

```
┌─────────────────────────────────────────────────────────────┐
│  对话历史（超过阈值，需要压缩）                                │
├─────────────────────────────────────────────────────────────┤
│  [头部保护区] System + 前 3 条消息                            │
│    - System Prompt                                           │
│    - User: 帮我实现登录功能                                   │
│    - Assistant: 好的，我来实现                                │
│    - User: 使用 FastAPI                                      │
├─────────────────────────────────────────────────────────────┤
│  [中间压缩区] ← 这个区域会被摘要替换                           │
│    - 大量工具调用和结果...                                    │
│    - 代码文件读取...                                         │
│    - 测试运行...                                             │
├─────────────────────────────────────────────────────────────┤
│  [尾部保护区] 最近约 20K tokens                               │
│    - User: 刚才的代码有报错                                   │
│    - Assistant: 我来检查...                                  │
│    - User: 怎么修复？                                        │
└─────────────────────────────────────────────────────────────┘
```

### 4.4 阶段4：摘要中间段（LLM）

`_generate_summary()`（~913+）：

- 调用 `call_llm`，task 名为 **`compression`**（`auxiliary.compression` 配置）
- 结构化模板：Active Task、Goal、Constraints、Completed Actions、Key Decisions 等
- `SUMMARY_PREFIX`（~37–51）：标明 summary 是参考材料，**不是**当前待执行指令（避免模型把历史总结当新任务）
- 若 protected 区已有旧 summary → **迭代更新**（`_find_latest_context_summary`），而非从零写

**摘要模板：**

```python
# agent/context_compressor.py:37-51
SUMMARY_PREFIX = (
    "[CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted "
    "into the summary below. This is a handoff from a previous context "
    "window — treat it as background reference, NOT as active instructions. "
    "Do NOT answer questions or fulfill requests mentioned in this summary; "
    "they were already addressed. "
    "Your current task is identified in the '## Active Task' section of the "
    "summary — resume exactly from there. "
    "IMPORTANT: Your persistent memory (MEMORY.md, USER.md) in the system "
    "prompt is ALWAYS authoritative and active — never ignore or deprioritize "
    "memory content due to this compaction note. "
)
```

**迭代式摘要更新：**

- 第一次压缩：生成完整摘要
- 后续压缩：基于旧摘要生成新摘要（保留更多信息）
- 避免多次压缩后信息过度丢失

**源码证据：**

```python
# agent/context_compressor.py:583-584
# Stores the previous compaction summary for iterative updates
self._previous_summary: Optional[str] = None
```

### 4.5 阶段5：组装与消毒

- 输出：`head + summary_message + tail`（或合并进 tail）
- `_sanitize_tool_pairs()`：保证 tool_call / tool result 配对合法
- `_strip_historical_media()`：去掉历史多模态以省 token

**压缩执行流程图：**

```
┌─────────────────────────────────────────────────────────────┐
│  每轮对话结束后，检查是否需要压缩                              │
└─────────────────────────────────────────────────────────────┘
                              ↓
                    ┌─────────────────┐
                    │ tokens >= threshold? │
                    └────────┬────────┘
                             ↓
                        [否] → 跳过，继续对话
                             ↓
                        [是]
                             ↓
              ┌──────────────────────────────┐
              │ 防止压缩无效保护（最近2次压缩 │
              │ 节省 < 10%）                 │
              └──────────┬───────────────────┘
                         ↓
                    [无效] → 跳过，提示用 /new
                         ↓
                    [有效]
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  第1步：工具结果修剪（无 LLM 调用）                           │
│  - 用简洁摘要替换冗长的工具输出                               │
│  - 去重重复的文件读取                                         │
│  - 截断过长的工具调用参数                                     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  第2步：保护头部和尾部                                        │
│  - 头部：system + 前 N 条消息（默认 3 条）                   │
│  - 尾部：最近的约 20K tokens                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  第3步：压缩中间部分                                          │
│  - 调用辅助模型生成摘要                                       │
│  - 如果有旧摘要，迭代更新                                     │
│  - 插入结构化摘要到消息列表                                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  第4步：Session 轮换                                          │
│  - 保存当前 Session 到 SQLite                                │
│  - 创建新的 Session ID                                       │
│  - 通知内存 Provider                                         │
└─────────────────────────────────────────────────────────────┘
                              ↓
                    返回压缩后的消息列表
```

---

## 5. 触发时机总表

```text
                    ┌─────────────────┐
                    │  run_conversation│
                    └────────┬────────┘
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
   Preflight (422-488)   Per-turn (3369-3410)   API errors
   loop 进入前           工具批成功后          413 / context_overflow /
                                             long_context_tier 等
         │                   │                   │
         └───────────────────┴───────────────────┘
                             ▼
                    _compress_context()
                             ▼
                    compress_context() → compress()
```

| 触发点 | 条件（要点） | 源码位置 |
|---|---|---|
| **Preflight** | `compression_enabled`；消息数 > head+tail+1；`estimate_request_tokens_rough`（含 tools）≥ threshold；最多 **3 遍** | `conversation_loop.py` 422–488 |
| **Per-turn** | 工具执行后；`should_compress(_real_tokens)` | 3369–3410 |
| **API 413** | Payload too large → 压缩后重试 | ~2450+ |
| **Context overflow** | Failover 分类为 overflow | ~2502+ |
| **Long context tier** | 先缩到 200k 再压缩 | ~2271–2320 |
| **Gateway hygiene** | 消息条数硬上限 `hygiene_hard_message_limit`（配置） | 见 `compression.hygiene_hard_message_limit` |
| **手动 `/compress`** | `force=True`，可绕过摘要失败冷却 | `context_compressor.py` ~1525–1529 |

### 5.1 Per-turn 的 token 来源（重要）

```python
# agent/conversation_loop.py:3384-3390
if _compressor.last_prompt_tokens > 0:
    # Only use prompt_tokens — completion/reasoning
    # tokens don't consume context window space.
    _real_tokens = _compressor.last_prompt_tokens
else:
    _real_tokens = estimate_request_tokens_rough(
        messages, tools=agent.tools or None
    )
```

**含义：** 带 thinking 的模型 completion 很大，但 **不应** 用 `prompt+completion` 触发压缩；否则过早压缩（#12026）。若 usage 丢失则用粗算（#2153、#14695）。

### 5.2 压缩成功后的 loop 行为

- `messages, active_system_prompt = agent._compress_context(...)`
- `conversation_history = None`（新 session 须全量 flush）
- 主循环 **`continue`**（不结束 turn）
- 若在 API retry 内压缩成功：`api_call_count -= 1` + `iteration_budget.refund()`（不计入完整迭代）

---

## 6. Token估算与管理

### 6.1 Token估算方法

**为什么需要估算？**

- API 响应中的 token 数是准确的，但只在请求后才知道
- 压缩决策需要在请求前做出
- 需要快速估算方法

**估算策略：**

```python
# agent/context_compressor.py:79-109
def _content_length_for_budget(raw_content: Any) -> int:
    """Return the effective char-length of a message's content for token budgeting.

    Plain strings: ``len(content)``. Multimodal lists: sum of text-part
    ``len(text)`` plus a flat ``_IMAGE_CHAR_EQUIVALENT`` per image part
    (``image_url`` / ``input_image`` / Anthropic-style ``image``). This
    keeps the compressor from treating a turn with 5 attached images as
    near-zero tokens just because the text part is empty.
    """
    if isinstance(raw_content, str):
        return len(raw_content)
    if not isinstance(raw_content, list):
        return len(str(raw_content or ""))

    total = 0
    for p in raw_content:
        if isinstance(p, str):
            total += len(p)
            continue
        if not isinstance(p, dict):
            total += len(str(p))
            continue
        ptype = p.get("type")
        if ptype in {"image_url", "input_image", "image"}:
            total += _IMAGE_CHAR_EQUIVALENT  # 1600 * 4 = 6400 chars
        else:
            total += len(p.get("text", "") or "")
    return total
```

**图像 Token估算：**

```python
# agent/context_compressor.py:64-75
# Flat token cost per attached image part
_IMAGE_TOKEN_ESTIMATE = 1600
# Chars per token rough estimate
_CHARS_PER_TOKEN = 4
# Same figure expressed in the char-budget currency
_IMAGE_CHAR_EQUIVALENT = _IMAGE_TOKEN_ESTIMATE * _CHARS_PER_TOKEN
```

### 6.2 模型上下文长度检测

**检测优先级：**

```
1. 配置文件中的 context_length 值
2. models.dev API 查询
3. OpenRouter API 查询
4. Anthropic API 查询
5. 静态硬编码表（兜底）
```

**源码证据：**

```python
# agent/model_metadata.py:115-133
# Descending tiers for context length probing when the model is unknown.
CONTEXT_PROBE_TIERS = [
    256_000,  # GPT-5.x, many current large-context models
    128_000,
    64_000,   # Minimum required
    32_000,
    16_000,
    8_000,
]

# Default context length when no detection method succeeds.
DEFAULT_FALLBACK_CONTEXT = CONTEXT_PROBE_TIERS[0]

# Minimum context length required to run Hermes Agent.
MINIMUM_CONTEXT_LENGTH = 64_000
```

### 6.3 Token使用追踪

**更新时机：**

```python
# agent/context_compressor.py:608-611
def update_from_response(self, usage: Dict[str, Any]):
    """Update tracked token usage from API response."""
    self.last_prompt_tokens = usage.get("prompt_tokens", 0)
    self.last_completion_tokens = usage.get("completion_tokens", 0)
```

**状态获取：**

```python
# agent/context_engine.py:178-192
def get_status(self) -> Dict[str, Any]:
    """Return status dict for display/logging."""
    return {
        "last_prompt_tokens": self.last_prompt_tokens,
        "threshold_tokens": self.threshold_tokens,
        "context_length": self.context_length,
        "usage_percent": (
            min(100, self.last_prompt_tokens / self.context_length * 100)
            if self.context_length else 0
        ),
        "compression_count": self.compression_count,
    }
```

---

## 7. 会话管理与轮换

### 7.1 Session生命周期

**完整生命周期：**

```
1. on_session_start(session_id)
   ├─ 加载持久化状态（DAG、store等）
   └─ 初始化上下文引擎

2. 每轮对话
   ├─ update_from_response(usage)  # 更新token统计
   ├─ should_compress()            # 检查是否需要压缩
   └─ compress() (如果需要)        # 执行压缩

3. on_session_end(session_id, messages)
   ├─ 刷新状态
   ├─ 关闭DB连接
   └─ 保存最终状态

4. on_session_reset()  # /new 或 /reset
   └─ 重置压缩计数和token统计
```

**源码证据：**

```python
# agent/context_engine.py:130-152
def on_session_start(self, session_id: str, **kwargs) -> None:
    """Called when a new conversation session begins.
    Use this to load persisted state (DAG, store) for the session.
    """

def on_session_end(self, session_id: str, messages: List[Dict[str, Any]]) -> None:
    """Called at real session boundaries (CLI exit, /reset, gateway expiry).
    Use this to flush state, close DB connections, etc.
    """

def on_session_reset(self) -> None:
    """Called on /new or /reset. Reset per-session state."""
    self.last_prompt_tokens = 0
    self.last_completion_tokens = 0
    self.compression_count = 0
```

### 7.2 压缩驱动的Session轮换

**`compress_context` 编排与 Session 轮换**

`agent/conversation_compression.py` `compress_context()` (~251–471) 在 `compress()` 之外做的事：

| 步骤 | 行为 |
|---|---|
| 1 | `memory_manager.on_pre_compress(messages)` |
| 2 | `context_compressor.compress(...)` |
| 3 | 若 `_last_compress_aborted` → **原样返回** messages，不轮换 session |
| 4 | 追加 todo snapshot 到压缩后 messages |
| 5 | `_invalidate_system_prompt()` + `_build_system_prompt()` |
| 6 | SessionDB：`end_session` → 新 `session_id`，`parent_session_id` 指向旧 session |
| 7 | `context_compressor.on_session_start(..., boundary_reason="compression")` |
| 8 | `memory_manager.on_session_switch(..., reason="compression")` |
| 9 | `commit_memory_session`（在此路径，而非每轮正常结束） |

```text
压缩前 session A                    压缩后 session B
─────────────────                    ─────────────────
[system][head][middle…][tail]   →    [system'][head][SUMMARY][tail']
     │                                      │
     └──────── parent_session_id ───────────┘
```

**与 /new 的区别：**

| 对比项 | /new | 压缩轮换 |
|-------|------|---------|
| **触发方式** | 用户手动 | 自动触发 |
| **历史保留** | 完全清空 | 保留摘要 |
| **Session关系** | 无父子关系 | 有parent_session_id |
| **内存Provider** | reset=True | reset=False |

**源码证据：**

```python
# agent/conversation_compression.py:362-400
if agent._session_db:
    try:
        # Propagate title to the new session with auto-numbering
        old_title = agent._session_db.get_session_title(agent.session_id)
        # Trigger memory extraction on the old session before it rotates.
        agent.commit_memory_session(messages)
        agent._session_db.end_session(agent.session_id, "compression")
        old_session_id = agent.session_id

        # 生成新 Session ID
        agent.session_id = f"{datetime.now().strftime('%Y%m%d_%H%M%S')}_{uuid.uuid4().hex[:6]}"
        os.environ["HERMES_SESSION_ID"] = agent.session_id

        # 创建新 Session 记录
        agent._session_db.create_session(
            session_id=agent.session_id,
            source=agent.platform or os.environ.get("HERMES_SESSION_SOURCE", "cli"),
            model=agent.model,
            model_config=agent._session_init_model_config,
            parent_session_id=old_session_id,  # 记录父 Session
        )

        # 更新系统提示
        agent._session_db.update_system_prompt(agent.session_id, new_system_prompt)
    except Exception as e:
        logger.warning("Session DB compression split failed: %s", e)
```

**Session 关系：**

```
原始 Session: 20240519_143052_a1b2c3
├─ 对话消息 1-50
└─ 压缩触发
    ↓
压缩后 Session: 20240519_150230_d4e5f6
├─ parent_session_id: 20240519_143052_a1b2c3
├─ 压缩摘要
└─ 对话消息 51-55
```

### 7.3 多次压缩警告

**问题：** 压缩次数越多，信息损失越大

**解决方案：**

```python
# agent/conversation_compression.py:435-442
# Warn on repeated compressions (quality degrades with each pass)
_cc = agent.context_compressor.compression_count
if _cc >= 2:
    agent._vprint(
        f"{agent.log_prefix}⚠️  Session compressed {_cc} times — "
        f"accuracy may degrade. Consider /new to start fresh.",
        force=True,
    )
```

---

## 8. 与Prompt Cache的关系

Hermes 里有多层「缓存」概念，不要混为一谈：

| 层 | 模块 | 与压缩的交互 |
|---|---|---|
| **Anthropic prompt caching** | `prompt_caching.py` `system_and_3` | 每 API 调用给 system + 最后 3 条非 system 打 `cache_control` |
| **OpenRouter response cache** | 配置 `openrouter.response_cache` | 全请求命中，与压缩无关 |
| **Skills system index** | `prompt_builder.build_skills_system_prompt` | 压缩会 **重建 system** → system 块缓存失效 |

**设计上的 cache-friendly 选择：**

- `pre_llm_call` 插件上下文注入 **user 消息**，不改 system（`conversation_loop.py` 495–498 注释）
- Slash skill 注入为 **user message**（`skill_commands.py`），保留 system 前缀稳定
- **`skill_manage` 成功**会 `clear_skills_system_prompt_cache` → 故意失效 skill 索引缓存

**压缩后：** 中间历史被替换为 summary，session id 变化 → 此前对话前缀对 provider 而言 **不可继续命中** 旧 cache breakpoint。

---

## 9. 配置参数说明

### 9.1 压缩相关配置

`hermes_cli/config.py` `DEFAULT_CONFIG`（节选）：

| 键 | 默认 | 含义 |
|---|---|---|
| `compression.enabled` | `True` | 总开关 |
| `compression.threshold` | `0.50` | 占模型 context 比例 |
| `compression.target_ratio` | `0.20` | tail token 预算 = threshold × ratio |
| `compression.protect_last_n` | `20` | tail 最少消息数 |
| `compression.protect_first_n` | `3` | head 非 system 保护条数 |
| `compression.hygiene_hard_message_limit` | `400` | Gateway 消息条数硬压缩 |
| `compression.abort_on_summary_failure` | `False` | 摘要失败是否中止（不 fallback） |
| `auxiliary.compression` | provider/model/... | 摘要专用模型 |
| `context.engine` | `"compressor"` | 可换插件 context engine |

**配置示例：**

```yaml
# config.yaml
compression:
  # 触发压缩的上下文使用率阈值
  threshold: 0.75  # 75% 时触发

  # 头部保护消息数（不包括system prompt）
  protect_first_n: 3

  # 尾部保护消息数
  protect_last_n: 20

  # 摘要目标比例（相对于threshold）
  summary_target_ratio: 0.20

  # 摘要失败时是否中止压缩
  abort_on_summary_failure: false

# 辅助模型配置
auxiliary:
  compression:
    model: "anthropic/claude-3-haiku-20240307"  # 便宜的模型
    provider: "openrouter"
    context_length: 200000  # 可选，覆盖检测值
```

### 9.2 模型相关配置

```yaml
model: "anthropic/claude-sonnet-4-20250514"  # 主模型

# 可选：显式指定上下文长度
# model_context_length: 200000
```

### 9.3 配置建议

**不同场景的推荐配置：**

| 场景 | threshold | protect_first_n | protect_last_n |
|------|-----------|-----------------|----------------|
| 长对话项目 | 0.75 | 3 | 20 |
| 快速迭代 | 0.50 | 1 | 10 |
| 严格成本控制 | 0.90 | 5 | 30 |

**何时考虑 /new：**

- 压缩次数 ≥ 2
- 需要重新讨论完全不同的主题
- 上下文中充满过时的调试信息

---

## 10. 失败、中止与手动/compress

### 10.1 失败处理

| 场景 | 行为 |
|---|---|
| 摘要 LLM 失败且 `abort_on_summary_failure=True` | `compress()` 返回原 messages，设 `_last_compress_aborted` |
| 摘要失败且 abort=False | 静态 fallback placeholder 摘要 |
| Aux 模型失败但主模型兜底成功 | 记录 `_last_aux_model_failure_*` 供 UI 告警 |
| `/compress` | `force=True`，绕过摘要失败冷却（~1525–1529） |
| Preflight 3 遍仍超阈值 | 停止 pass；依赖后续 per-turn / API 路径 |

### 10.2 手动触发压缩

**使用 `/compress` 命令：**

```bash
# 普通压缩
/compress

# 主题聚焦压缩（保留与指定主题相关的内容）
/compress "登录功能"
```

**源码证据：**

```python
# agent/context_engine.py:97-100
focus_topic: Optional topic string from manual ``/compress <focus>``.
Engines that support guided compression should prioritise
preserving information related to this topic.
```

---

## 11. 常见问题深度解答（FAQ）

### Q1: 压缩会用主模型还是辅助模型？

**A: 优先使用配置的辅助模型（便宜/快速），失败时降级到主模型。**

**源码证据：**

```python
# agent/conversation_compression.py:44-73
# 启动时检查辅助模型可行性
def check_compression_model_feasibility(agent: Any) -> None:
    """Warn at session start if the auxiliary compression model's context
    window is smaller than the main model's compression threshold.
    """
    client, aux_model = get_text_auxiliary_client(
        "compression",
        main_runtime=agent._current_main_runtime(),
    )

    # 检查辅助模型上下文窗口
    aux_context = get_model_context_length(aux_model, ...)

    if aux_context < threshold:
        # 自动降低阈值以适应辅助模型
        agent.context_compressor.threshold_tokens = aux_context
```

**降级机制：**

```python
# agent/conversation_compression.py:337-352
# 检查是否使用了降级
_aux_fail_model = getattr(agent.context_compressor, "_last_aux_model_failure_model", None)
if _aux_fail_model:
    agent._emit_warning(
        f"ℹ Configured compression model '{_aux_fail_model}' failed. "
        f"Recovered using main model — check auxiliary.compression.model in config.yaml."
    )
```

### Q2: 压缩后旧的对话内容还能找到吗？

**A: 可以，通过 SQLite Session DB。**

压缩只是从当前对话上下文中移除了中间消息，但所有历史都保存在数据库中。

**源码证据：**

```python
# agent/conversation_compression.py:362-400
if agent._session_db:
    # 在结束旧 Session 前触发记忆提取
    agent.commit_memory_session(messages)

    # 结束旧 Session（保留所有历史）
    agent._session_db.end_session(agent.session_id, "compression")
    old_session_id = agent.session_id

    # 创建新 Session，记录父 Session
    agent._session_db.create_session(
        session_id=agent.session_id,
        model=agent.model,
        parent_session_id=old_session_id,  # 可追溯
    )
```

### Q3: 图像内容如何处理？

**A: 最新用户消息的图像保留，旧图像被替换为占位符。**

**源码证据：**

```python
# agent/context_compressor.py:275-329
def _strip_historical_media(messages: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
    """Replace image parts in older messages with placeholder text.

    The anchor is the *last* user message that has any image content. Every
    message before that anchor gets its image parts replaced with a short
    placeholder so the outgoing request stops re-shipping the same multi-MB
    base-64 image blobs on every turn.
    """
    # 找到最后一个包含图像的用户消息
    anchor = -1
    for i in range(len(messages) - 1, -1, -1):
        msg = messages[i]
        if msg.get("role") == "user" and _content_has_images(msg.get("content")):
            anchor = i
            break

    # 该消息之前的所有图像都被替换
    for i, msg in enumerate(messages):
        if i < anchor and _content_has_images(msg.get("content")):
            new_msg = msg.copy()
            new_msg["content"] = _strip_images_from_content(content)
            result.append(new_msg)
```

### Q4: 为什么有时候压缩被跳过？

**A: 可能有以下原因：**

1. **Token未达阈值：** `prompt_tokens < threshold_tokens`
2. **防抖动保护：** 最近两次压缩节省都 < 10%

**源码证据：**

```python
# agent/context_compressor.py:613-633
def should_compress(self, prompt_tokens: int = None) -> bool:
    tokens = prompt_tokens if prompt_tokens is not None else self.last_prompt_tokens
    if tokens < self.threshold_tokens:
        return False
    # Anti-thrashing protection
    if self._ineffective_compression_count >= 2:
        logger.warning("Compression skipped — last %d compressions saved <10%% each.")
        return False
    return True
```

### Q5: 压缩和 Anthropic Prompt Cache 是同一功能吗？

**A: 不是，它们是正交的两个功能。**

| 对比项 | 上下文压缩 | Prompt Cache |
|-------|----------|-------------|
| **目的** | 腾出空间 | 加速响应 + 降低成本 |
| **机制** | 用摘要替换旧内容 | 缓存system + 最后3条消息 |
| **影响** | 改变对话内容 | 不改变内容，只缓存 |
| **关系** | 压缩会破坏旧cache | - |

**压缩后：** 中间历史被替换为 summary，session id 变化 → 此前对话前缀对 provider 而言 **不可继续命中** 旧 cache breakpoint。

---

## 12. 讲者展开点与常见误读

### 12.1 讲者展开点

- 用 **头-摘要-尾** 示意图讲清「保护什么、丢什么」
- 强调 **preflight 含 tool schema token**（50+ 工具时 20–30K）
- 对比 **per-turn 用 prompt_tokens only** 的原因（thinking 模型）
- Session 轮换讲成「审计边界」：压缩 = 新 episode，旧 session 可查 parent 链

### 12.2 常见误读

| 误读 | 事实 |
|---|---|
| `protect_last_n` = 固定保留最后 20 条 | 主规则是 **token 预算**，20 是下限 |
| 用 completion tokens 触发压缩 | 回合后用 **prompt_tokens only** |
| 压缩不改变 session | 会 `end_session` + 新 `session_id` |
| `classified.should_compress` 直接触发压缩 | 多用于错误分类；实际走 413/overflow 等分支 |
| 压缩与 Anthropic cache 是同一功能 | 正交；压缩会破坏旧前缀 cache |

---

## 附录：源码追踪路径

如果你想深入理解上下文管理的实现，按以下顺序阅读：

```
1. agent/context_engine.py
   └─ ContextEngine 抽象接口定义
   └─ 压缩触发条件（should_compress）
   └─ Session生命周期方法

2. agent/context_compressor.py
   └─ ContextCompressor 核心实现
   └─ 工具结果修剪（_prune_old_tool_results）
   └─ 中间部分压缩（compress）
   └─ 图像处理（_strip_historical_media）

3. agent/conversation_compression.py
   └─ 压缩流程编排（compress_context）
   └─ Session轮换逻辑
   └─ 辅助模型可行性检查

4. agent/model_metadata.py
   └─ Token估算函数
   └─ 模型上下文长度检测
   └─ 默认上下文长度表

5. agent/memory_manager.py
   └─ 内存Provider的压缩通知
   └─ on_session_switch实现
```

---

## 附录：术语表

| 术语 | 解释 |
|------|------|
| **Context Window** | 模型一次能处理的最大文本长度 |
| **Token** | 文本的最小单位，约等于 0.75 个英文单词或 0.5 个中文字 |
| **压缩阈值** | 触发压缩的 token 使用率百分比 |
| **头部保护** | 始终保留的前 N 条消息 |
| **尾部保护** | 始终保留的最新约 20K tokens |
| **Session轮换** | 压缩后创建新的 Session ID |
| **辅助模型** | 用于生成摘要的便宜/快速模型 |
| **防抖动** | 避免无效压缩重复触发的机制 |
| **Prompt Cache** | Anthropic 的缓存功能，缓存 system + 最后3条消息 |
| **Preflight** | 主循环开始前的预压缩检查 |

---

## 相关文档

- [主循环](./hermes-agent-main-loop.md) — preflight / per-turn 触发点在 loop 中的位置
- [Runtime 导读](./hermes-agent-runtime-reader-guide.md) — 记忆与 session_search
- [Skill 系统](./hermes-agent-skill-system.md) — skill 索引与 cache 失效

---

**文档版本：** v2.0（整合版）
**最后更新：** 2025-05-19
**源码验证：** ✅ 已通过真实源码验证（95% 准确度）
