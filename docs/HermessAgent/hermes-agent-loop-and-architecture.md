# Hermes-Agent Loop 流程与整体架构

> 基于源码 `/Users/wangzhongbin/Documents/code/fangzhen/hermes-agent` 分析
> 核心文件: `agent/conversation_loop.py` (4099 行)、`run_agent.py` (4123 行)、`cli.py` (14166 行)

---

## 一、整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Hermes-Agent 整体架构                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐  │
│  │   CLI 模式    │  │ Gateway 模式 │  │        Cron 定时任务          │  │
│  │  (cli.py)    │  │(gateway/run) │  │     (cron/scheduler.py)      │  │
│  │              │  │              │  │                              │  │
│  │ HermesCLI    │  │ GatewayRunner│  │ tick() 每60s检查到期任务      │  │
│  │ REPL交互循环 │  │ 多平台消息网关│  │ 独立AIAgent实例执行          │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────────────┘  │
│         │                 │                      │                      │
│         └─────────────────┼──────────────────────┘                      │
│                           ▼                                             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                      AIAgent (run_agent.py)                       │  │
│  │  ┌─────────────┐ ┌──────────────┐ ┌───────────────────────────┐ │  │
│  │  │ 会话状态管理  │ │ 模型/Provider│ │  API传输适配 (transports/) │ │  │
│  │  │(hermes_state)│ │  路由与切换  │ │ chat_completions          │ │  │
│  │  │ SQLite + WAL │ │ failover链  │ │ anthropic_messages        │ │  │
│  │  │ FTS5全文搜索  │ │ 凭证池轮转  │ │ codex_responses           │ │  │
│  │  └─────────────┘ └──────────────┘ │ bedrock_converse           │ │  │
│  │                                   └───────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────────────────┐ │  │
│  │  │              Conversation Loop (conversation_loop.py)       │ │  │
│  │  │   run_conversation() — 单用户消息的完整处理周期              │ │  │
│  │  │                                                            │ │  │
│  │  │  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌──────────┐  │ │  │
│  │  │  │ 1.初始化  │→│ 2.API调用  │→│ 3.响应解析 │→│ 4.判断    │  │ │  │
│  │  │  │ 预处理    │  │ (流式/非流)│  │ normalize │  │ 分支路由  │  │ │  │
│  │  │  └──────────┘  └───────────┘  └──────────┘  └─────┬────┘  │ │  │
│  │  │         ▲                                         │       │ │  │
│  │  │         │            ┌────────────────┐    ┌──────┴──┐    │ │  │
│  │  │         │            │ 有 tool_calls? │    │         │    │ │  │
│  │  │         │            └───────┬────────┘    │         │    │ │  │
│  │  │         │              是 ↓  │         否 ↓ │         │    │ │  │
│  │  │         │     ┌──────────────┴──┐  ┌───────┴──────┐  │ │  │
│  │  │         │     │ 5.工具执行       │  │ 6.最终响应    │  │ │  │
│  │  │         │     │ _execute_tool   │  │ break退出    │  │ │  │
│  │  │         │     │ calls()         │  │ 返回结果     │  │ │  │
│  │  │         │     └────────┬────────┘  └──────────────┘  │ │  │
│  │  │         │     上下文压缩│                            │ │  │
│  │  │         └───────────────┘                            │ │  │
│  │  └────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                           │                                         │
│         ┌─────────────────┼─────────────────┐                      │
│         ▼                 ▼                 ▼                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐     │
│  │  Tools 工具层 │  │ Memory 记忆  │  │ Skills 技能系统       │     │
│  │ (tools/*.py)  │  │ (memory_mgr) │  │ (skills/*.py)        │     │
│  │              │  │              │  │                      │     │
│  │ 文件操作     │  │ 外部记忆提供者│  │ 动态加载/创建        │     │
│  │ 浏览器自动化 │  │ 上下文预取    │  │ 后台review优化       │     │
│  │ 终端/代码执行│  │ 向量/全文检索 │  │ skill_manage工具     │     │
│  │ 搜索/图像    │  │              │  │                      │     │
│  │ MCP协议      │  │              │  │                      │     │
│  │ 76+工具      │  │              │  │                      │     │
│  └──────────────┘  └──────────────┘  └──────────────────────┘     │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                  支撑服务层                                    │  │
│  │  ┌─────────────┐ ┌──────────────┐ ┌─────────────────────┐   │  │
│  │  │ Prompt缓存  │ │ 错误分类     │ │ 迭代预算管理         │   │  │
│  │  │ (cache_ctrl)│ │(error_classify)│ │(iteration_budget)  │   │  │
│  │  │ 前缀缓存    │ │ 失败原因分类 │ │ 线程安全计数器      │   │  │
│  │  │ ~75% token  │ │ 自动恢复策略 │ │ consume/refund     │   │  │
│  │  │ 节省        │ │ failover链   │ │ 父agent 90次       │   │  │
│  │  └─────────────┘ └──────────────┘ │ 子agent 50次       │   │  │
│  │                                   └─────────────────────┘   │  │
│  │  ┌─────────────┐ ┌──────────────┐ ┌─────────────────────┐   │  │
│  │  │ 上下文压缩  │ │ 凭证池       │ │ 插件钩子系统         │   │  │
│  │  │(ctx_compress)│ │(credential_  │ │ (plugins/hooks)     │   │  │
│  │  │ 智能截断    │  │ pool)        │ │ on_session_start    │   │  │
│  │  │ 压缩触发50%│ │ 多key轮转    │ │ pre_api_request     │   │  │
│  │  │ 摘要压缩    │ │ 自动刷新     │ │ post_api_request    │   │  │
│  │  └─────────────┘ └──────────────┘ │ post_llm_call       │   │  │
│  │                                   │ transform_llm_output │   │  │
│  │                                   └─────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Gateway 平台适配层 (gateway/platforms/)          │  │
│  │  Telegram | Discord | Slack | WhatsApp | WeChat | Matrix     │  │
│  │  Signal | DingTalk | Feishu | Email | QQ | Webhook | API     │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、Conversation Loop 详细流程

### 2.1 入口与初始化

一次完整的 Conversation Loop 由 `run_conversation()` 函数驱动，对应**一条用户消息**的完整处理周期。

```
用户消息 → run_conversation(agent, user_message, ...)
                │
                ▼
        ┌──────────────────┐
        │ 1. 初始化阶段     │
        │ _install_safe_stdio│ ← 防止broken pipe崩溃
        │ _ensure_db_session│ ← SQLite会话初始化
        │ set_runtime_main  │ ← 记录当前provider/model
        │ set_session_context│ ← 日志绑定session_id
        │ set_current_write │ ← 设置技能写入来源
        │  _origin          │
        │ _restore_primary  │ ← 恢复主provider(如上轮fallback)
        │ _sanitize_surrogates│ ← 清理非法Unicode
        │ IterationBudget   │ ← 创建本轮迭代预算(默认90)
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │ 2. 系统Prompt构建 │
        │ _restore_or_build │
        │ _system_prompt()  │
        │                   │
        │ 三路状态判断:      │
        │ • present: 复用缓存│ ← prefix cache命中
        │ • null/empty: 重建 │ ← 警告+重建
        │ • missing: 首次构建│ ← 新session
        │                   │
        │ 首次构建时:        │
        │ → _build_system   │
        │   _prompt()       │
        │ → 持久化到SQLite   │
        │ → 触发on_session  │
        │   _start hook     │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │ 3. 消息构建       │
        │ 追加user message  │
        │ 内存预取(prefetch)│ ← memory_manager异步预取
        │ 插件上下文注入    │ ← pre_llm_call hooks
        │ 构建messages列表  │
        └────────┬─────────┘
                 │
                 ▼
    ┌────────────────────────────────────────────────────┐
    │                                                    │
    │  ════════  主循环 while loop  ════════            │
    │                                                    │
    │  条件: api_call_count < max_iterations             │
    │     AND iteration_budget.remaining > 0             │
    │     OR  _budget_grace_call (宽限调用)              │
    │                                                    │
    └────────────────────────────────────────────────────┘
```

### 2.2 主循环单次迭代流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     单次迭代 (Iteration)                         │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Step A: 前置检查                                         │   │
│  │                                                          │   │
│  │  • _interrupt_requested? → 中断退出                     │   │
│  │  • api_call_count++ (计数+1)                             │   │
│  │  • _budget_grace_call? → 消耗宽限标志                   │   │
│  │  • iteration_budget.consume() → 预算耗尽则退出          │   │
│  │  • step_callback → 网关钩子通知                         │   │
│  │  • /steer排水 → 用户中途引导注入到tool消息中            │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Step B: API请求构建                                      │   │
│  │                                                          │   │
│  │  • _sanitize_tool_call_arguments → 修复损坏的参数        │   │
│  │  • _repair_message_sequence → 修复角色交替               │   │
│  │  • 注入memory prefetch上下文到user消息                   │   │
│  │  • 注入ephemeral system prompt                          │   │
│  │  • apply_anthropic_cache_control → prompt缓存标记       │   │
│  │  • _sanitize_api_messages → 清理孤立tool结果            │   │
│  │  • _drop_thinking_only_and_merge_users                  │   │
│  │  • JSON规范化(sort_keys, 紧凑分隔符)                    │   │
│  │  • _sanitize_messages_surrogates → 清理代理字符         │   │
│  │  • 估算token数量                                        │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Step C: API调用 + 重试循环                               │   │
│  │                                                          │   │
│  │  内层while: retry_count < max_retries                   │   │
│  │                                                          │   │
│  │  ┌─ Nous rate limit guard → 跳过/切换fallback           │   │
│  │  ├─ _build_api_kwargs → 构建请求参数                    │   │
│  │  ├─ pre_api_request plugin hook                         │   │
│  │  ├─ 选择流式/非流式:                                    │   │
│  │  │   ├─ _interruptible_streaming_api_call (优先)        │   │
│  │  │   └─ _interruptible_api_call (后备)                  │   │
│  │  ├─ 响应验证: validate_response()                       │   │
│  │  └─ 异常处理与恢复策略:                                  │   │
│  │      ├─ UnicodeEncodeError → surrogate/ASCII修复        │   │
│  │      ├─ 图片拒绝 → strip_images, 文本模式              │   │
│  │      ├─ 429 rate limit → 凭证轮转/fallback切换         │   │
│  │      ├─ 401 auth → 刷新凭证                            │   │
│  │      ├─ context too large → 压缩上下文                  │   │
│  │      ├─ thinking签名失败 → 清除reasoning_details       │   │
│  │      ├─ llama.cpp grammar错误 → 清理schema             │   │
│  │      ├─ 图片过大 → 缩小重试                            │   │
│  │      └─ OAuth long-context beta → 禁用beta重试         │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Step D: 响应解析                                         │   │
│  │                                                          │   │
│  │  • transport.normalize_response() → 统一响应格式        │   │
│  │  • 提取: content, tool_calls, finish_reason             │   │
│  │  • Token使用量追踪: prompt/completion/total             │   │
│  │  • 成本估算: estimate_usage_cost()                      │   │
│  │  • 持久化到SQLite: update_token_counts()                │   │
│  │  • 缓存命中率统计                                       │   │
│  │  • post_api_request plugin hook                         │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Step E: 分支路由                                         │   │
│  │                                                          │   │
│  │              finish_reason?                               │   │
│  │           ┌──────┴──────┐                                │   │
│  │           ▼             ▼                                │   │
│  │    ┌────────────┐ ┌───────────┐                         │   │
│  │    │ "length"   │ │ "stop"    │                         │   │
│  │    │ 截断处理   │ │           │                         │   │
│  │    │            │ │ tool_calls?│                         │   │
│  │    │ • thinking │ │ ┌───┴───┐ │                         │   │
│  │    │   耗尽 →   │ │ ▼       ▼ │                         │   │
│  │    │   返回警告 │ │有       无 │                         │   │
│  │    │            │ │ │       │  │                         │   │
│  │    │ • 响应截断 │ │ ▼       ▼  │                         │   │
│  │    │ → 连续请求 │ │Step F  Step G                         │   │
│  │    │   最多3次  │ │工具执行 最终响应                       │   │
│  │    └────────────┘ │                                    │   │
│  │                   │                                    │   │
│  └───────────────────┼────────────────────────────────────┘   │
│                           ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Step F: 工具执行路径                                     │   │
│  │                                                          │   │
│  │  1. 校验tool_calls有效性                                 │   │
│  │     • 无效工具名 → 重试/回退                             │   │
│  │     • 参数损坏 → 修复/拒绝                              │   │
│  │  2. 去重与上限控制                                       │   │
│  │     • _deduplicate_tool_calls                           │   │
│  │     • _cap_delegate_task_calls                          │   │
│  │  3. 追加assistant消息到history                           │   │
│  │  4. _execute_tool_calls() → 并行/串行执行               │   │
│  │     • handle_function_call → 单工具调用                  │   │
│  │     • 审批机制 (approval.py) → 危险操作需确认           │   │
│  │     • 工具护栏 (tool_guardrails) → 安全检查             │   │
│  │     • 结果追加为 role="tool" 消息                       │   │
│  │  5. 工具护栏检查 → halt则break                          │   │
│  │  6. execute_code迭代 → refund预算                       │   │
│  │  7. 上下文压缩检查                                       │   │
│  │     • should_compress(tokens) → 超过阈值触发            │   │
│  │     • _compress_context → 摘要压缩历史                  │   │
│  │  8. 保存session log                                      │   │
│  │  9. continue → 回到循环顶部                              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Step G: 最终响应路径                                     │   │
│  │                                                          │   │
│  │  1. final_response = assistant_message.content           │   │
│  │  2. 清理think block → _strip_think_blocks               │   │
│  │  3. 清理内部scaffolding消息                              │   │
│  │  4. 追加final assistant消息到history                     │   │
│  │  5. break → 退出主循环                                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 循环退出后的收尾流程

```
主循环退出
      │
      ▼
┌──────────────────────────────┐
│ 退出原因判断                  │
│                              │
│ • final_response == None     │
│   AND budget耗尽?            │
│   → _handle_max_iterations() │ ← 无工具的总结请求
│   → kanban_block (如果worker)│
│                              │
│ • interrupted?               │
│   → 保留中断消息             │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ 后处理流水线                  │
│                              │
│ 1. _save_trajectory()        │ ← 保存执行轨迹
│ 2. _cleanup_task_resources() │ ← 清理VM/浏览器
│ 3. _persist_session()        │ ← 持久化到SQLite
│ 4. 诊断日志输出              │ ← 记录退出原因
│ 5. 文件变更验证footer        │ ← 检测失败的文件操作
│ 6. transform_llm_output hook │ ← 插件转换输出
│ 7. post_llm_call hook        │ ← 插件后处理
│ 8. 提取reasoning内容         │ ← 用于UI展示
│ 9. 构建result字典            │
│ 10. skill nudge检查          │ ← 周期性技能优化
│ 11. 外部记忆同步              │ ← sync+prefetch
│ 12. 后台memory/skill review  │ ← 异步agent自我改进
│ 13. on_session_end hook      │ ← 插件清理
└──────────────┬───────────────┘
               ▼
         返回 result dict
```

---

## 三、核心组件详解

### 3.1 AIAgent 类 (`run_agent.py`)

核心属性与职责：

| 属性/组件 | 说明 |
|---|---|
| `max_iterations` | 工具调用最大迭代次数，默认 90 |
| `iteration_budget` | `IterationBudget` 实例，线程安全的消耗/退还计数器 |
| `context_compressor` | 上下文压缩器，监控 token 使用并触发压缩 |
| `_session_db` | SQLite 会话数据库连接 (WAL模式) |
| `tools` | 当前可用工具定义列表 |
| `client` | OpenAI-compatible API 客户端 |
| `_fallback_chain` | 备用 provider 切换链 |
| `_credential_pool` | 多凭证轮转池 |

### 3.2 迭代预算 (`iteration_budget.py`)

```
IterationBudget(max_total=90)  ← 父agent
IterationBudget(max_total=50)  ← 子agent (委托任务)

  consume() → bool   # 尝试消耗1次迭代，返回是否允许
  refund()           # 退还1次 (execute_code调用不计入)
  used / remaining   # 线程安全属性
```

### 3.3 工具执行 (`tool_executor.py` + `tools/registry.py`)

```
assistant_message.tool_calls
         │
         ▼
_execute_tool_calls()
         │
         ├→ 校验: 工具名是否在valid_tool_names中
         ├→ 去重: _deduplicate_tool_calls
         ├→ 上限: _cap_delegate_task_calls
         ├→ 并行/串行分发执行
         │   ├→ handle_function_call(name, args)
         │   │   ├→ tools/registry.py → 查找handler
         │   │   ├→ approval.py → 危险操作审批
         │   │   ├→ tool_guardrails → 安全护栏
         │   │   └→ 执行handler → 返回结果
         │   └→ 结果追加为 role="tool" 消息
         └→ 检查guardrail halt → 决定是否终止
```

### 3.4 错误分类与恢复 (`error_classifier.py`)

```
API异常
   │
   ▼
classify_api_error() → FailoverReason枚举
   │
   ├→ rate_limit        → 凭证轮转 / fallback切换
   ├→ context_too_large → 上下文压缩
   ├→ auth_failure      → 刷新凭证
   ├→ thinking_signature→ 清除reasoning_details
   ├→ image_too_large   → 图片缩小
   ├→ billing           → fallback切换
   ├─ long_context_tier → 降级context长度
   ├─ llama_cpp_grammar → 清理schema
   └─ 其他              → 指数退避重试
```

### 3.5 会话状态管理 (`hermes_state.py`)

```
SessionDB (SQLite + WAL模式)
   │
   ├→ sessions表: session_id, title, system_prompt, token_counts
   ├→ messages表: 完整对话历史
   ├→ FTS5全文搜索: 快速会话检索
   ├→ Schema版本: v11, 支持自动迁移
   └→ 压缩触发分裂: 大会话自动拆分

路径: ~/.hermes/state.db
```

### 3.6 Prompt 缓存策略 (`prompt_caching.py`)

```
Anthropic prompt caching (system_and_3 布局):
  ┌─────────────────────────────┐
  │ system message              │ ← cache_control breakpoint
  ├─────────────────────────────┤
  │ ... conversation history    │
  │ last 3 messages             │ ← cache_control breakpoints
  └─────────────────────────────┘
  
  效果: 多轮对话 input token 节省约 75%
  条件: _use_prompt_caching = True (Anthropic/OpenRouter/兼容网关)
```

---

## 四、多模式运行架构

```
┌─────────────────────────────────────────────────────────────┐
│                     运行模式                                 │
├───────────────────┬─────────────────┬───────────────────────┤
│     CLI 模式      │   Gateway 模式   │     Cron 模式         │
│   (cli.py)        │ (gateway/run.py) │ (cron/scheduler.py)   │
├───────────────────┼─────────────────┼───────────────────────┤
│ HermesCLI REPL    │ GatewayRunner   │ Scheduler.tick()       │
│ ─────────────     │ ──────────────  │ ─────────────          │
│ prompt_toolkit UI │ asyncio事件循环 │ 60s轮询周期            │
│ 单用户交互        │ 多平台并发      │ 文件锁防并发           │
│ 实时流式输出      │ 消息队列分发    │ 独立AIAgent实例        │
│ 60+ 斜杠命令      │ 平台适配器      │ jobs.json持久化        │
│ 会话管理/切换     │ 会话隔离/复用   │ 结果推送到平台         │
│ TTS语音模式       │ 流式传输        │                        │
└───────────────────┴─────────────────┴───────────────────────┘
         │                  │                      │
         └──────────────────┼──────────────────────┘
                            ▼
                    AIAgent.run_conversation()
```

---

## 五、数据流全景

```
用户输入 (文本/图片/语音)
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 入口层                                                      │
│   CLI: HermesCLI._handle_user_input()                       │
│   Gateway: PlatformAdapter.on_message()                     │
│   Cron: Scheduler._execute_job()                            │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ AIAgent.run_conversation(user_message)                      │
│                                                             │
│  messages = [system_prompt] + conversation_history          │
│           + [user_message]                                  │
│                                                             │
│  ┌─────────────────── Loop ───────────────────┐             │
│  │                                             │             │
│  │  api_messages = prepare(messages)           │             │
│  │       + memory_context                     │             │
│  │       + plugin_injections                  │             │
│  │       + cache_control                      │             │
│  │                                             │             │
│  │  response = LLM(api_messages, tools)        │             │
│  │                                             │             │
│  │  if tool_calls:                             │             │
│  │      results = execute(tool_calls)          │             │
│  │      messages += [assistant, tool_results]  │             │
│  │      continue                               │             │
│  │  else:                                      │             │
│  │      final_response = response.content      │             │
│  │      break                                  │             │
│  │                                             │             │
│  └─────────────────────────────────────────────┘             │
│                                                             │
│  persist(messages, final_response)                          │
│  return result                                              │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
输出到用户 (文本/Markdown/代码/图片)
    │
    ├→ 后台: memory review (agent自我优化记忆)
    ├→ 后台: skill review (agent自我优化技能)
    └→ 后台: 外部记忆同步
```

---

## 六、关键设计决策

| 设计点 | 实现 | 效果 |
|---|---|---|
| **有界迭代** | IterationBudget + max_iterations(90) | 防止无限工具调用循环 |
| **宽限调用** | `_budget_grace_call` | 预算耗尽后给模型一次总结机会 |
| **流式优先** | 默认使用streaming API | 实时输出 + 90s超时健康检查 |
| **Prompt缓存** | system_and_3 布局 + byte-stable system prompt | 多轮对话节省~75% input token |
| **错误分类** | FailoverReason枚举 + 结构化恢复 | 精准匹配恢复策略，避免盲目重试 |
| **凭证池** | 多API key轮转 + 自动刷新 | 429时快速切换，无需等待重置 |
| **Fallback链** | `_fallback_chain` 预配置备用provider | provider不可用时无缝切换 |
| **上下文压缩** | 50%阈值触发 + 摘要压缩 | 长会话不溢出context window |
| **工具护栏** | tool_guardrails + 审批机制 | 危险操作需确认/可阻断 |
| **会话持久化** | SQLite WAL + FTS5 + schema迁移 | 跨重启恢复 + 快速检索 |
| **插件钩子** | 6个生命周期钩子点 | 可扩展性，不改核心代码 |

---

## 七、核心文件索引

| 文件 | 行数 | 职责 |
|---|---|---|
| `agent/conversation_loop.py` | 4099 | 主对话循环，API调用/重试/工具执行 |
| `run_agent.py` | 4123 | AIAgent类定义，初始化/构建/辅助方法 |
| `cli.py` | 14166 | CLI交互界面，REPL/命令/显示 |
| `agent/iteration_budget.py` | 62 | 线程安全迭代预算计数器 |
| `agent/error_classifier.py` | ~400 | API错误分类与恢复策略 |
| `agent/prompt_caching.py` | ~200 | Anthropic prompt缓存标记注入 |
| `agent/memory_manager.py` | ~600 | 外部记忆提供者管理 |
| `agent/tool_executor.py` | ~500 | 工具调用分发执行 |
| `agent/context_compressor.py` | ~500 | 上下文压缩与token管理 |
| `agent/model_metadata.py` | ~400 | 模型元数据与token估算 |
| `agent/credential_pool.py` | ~300 | 多凭证轮转管理 |
| `hermes_state.py` | 3238 | SQLite会话数据库 |
| `tools/registry.py` | ~300 | 工具注册表 |
| `gateway/run.py` | ~300 | 网关服务启动 |
| `cron/scheduler.py` | ~200 | 定时任务调度 |
