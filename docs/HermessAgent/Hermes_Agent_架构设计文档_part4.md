#### 3.5.2 context_compressor.py

**职责**: 自动上下文窗口压缩

**问题**: 长对话会超出模型的上下文窗口限制

**解决方案**: 
1. 保护头部（系统提示词）和尾部（最近消息）
2. 压缩中间部分（旧的对话历史）
3. 使用辅助模型生成摘要
4. 迭代更新摘要

**核心逻辑**:
```python
class ContextCompressor:
    def __init__(self, auxiliary_client):
        self.aux_client = auxiliary_client
        self._last_summary_time = 0
    
    def compress_if_needed(
        self,
        messages: List[Dict],
        model: str,
        tools: List[Dict],
    ) -> List[Dict]:
        """
        检查并压缩消息历史
        
        流程:
        1. 估算 token 数
        2. 判断是否需要压缩
        3. 确定压缩范围
        4. 生成摘要
        5. 替换中间消息
        """
        # 1. 估算 token
        context_limit = get_model_context_length(model)
        estimated_tokens = estimate_messages_tokens_rough(messages)
        estimated_tokens += estimate_tools_tokens(tools)
        
        # 2. 判断是否需要压缩
        threshold = context_limit * 0.75  # 75% 触发压缩
        if estimated_tokens < threshold:
            return messages
        
        # 3. 确定压缩范围
        head_messages, middle_messages, tail_messages = \
            self._split_messages(messages, context_limit)
        
        if len(middle_messages) < 2:
            # 无法压缩
            return messages
        
        # 4. 生成摘要
        summary = self._generate_summary(middle_messages)
        
        # 5. 替换
        summary_message = {
            'role': 'user',
            'content': f"{SUMMARY_PREFIX}\n\n{summary}"
        }
        
        compressed = head_messages + [summary_message] + tail_messages
        
        logger.info(
            f"Compressed {len(messages)} messages to {len(compressed)} "
            f"(saved ~{estimated_tokens - estimate_messages_tokens_rough(compressed)} tokens)"
        )
        
        return compressed
    
    def _split_messages(
        self,
        messages: List[Dict],
        context_limit: int
    ) -> Tuple[List[Dict], List[Dict], List[Dict]]:
        """
        分割消息为 头部 / 中间 / 尾部
        
        策略:
        - 头部: 系统消息 + 第一轮对话
        - 尾部: 最近 N 轮对话（基于 token 预算）
        - 中间: 其余部分
        """
        # 头部: 系统消息
        head = [m for m in messages if m['role'] == 'system']
        non_system = [m for m in messages if m['role'] != 'system']
        
        # 尾部: 保留最近的消息（token 预算）
        tail_budget = int(context_limit * 0.3)  # 30% 给尾部
        tail = []
        tail_tokens = 0
        
        for msg in reversed(non_system):
            msg_tokens = estimate_message_tokens(msg)
            if tail_tokens + msg_tokens > tail_budget:
                break
            tail.insert(0, msg)
            tail_tokens += msg_tokens
        
        # 中间: 剩余部分
        middle_start = len(head)
        middle_end = len(messages) - len(tail)
        middle = messages[middle_start:middle_end]
        
        return head, middle, tail
    
    def _generate_summary(self, messages: List[Dict]) -> str:
        """
        使用辅助模型生成摘要
        
        提示词模板:
        - 结构化摘要（已解决问题 / 待处理问题 / 剩余工作）
        - 保留关键细节
        - 避免被误读为新指令
        """
        # 构建摘要提示词
        prompt = self._build_summary_prompt(messages)
        
        # 调用辅助模型
        summary = self.aux_client.generate(
            prompt=prompt,
            max_tokens=self._calculate_summary_budget(messages)
        )
        
        return summary
    
    def _build_summary_prompt(self, messages: List[Dict]) -> str:
        """构建摘要提示词"""
        # 格式化消息历史
        formatted = []
        for msg in messages:
            role = msg['role']
            content = msg.get('content', '')
            
            if msg.get('tool_calls'):
                # 工具调用
                for tc in msg['tool_calls']:
                    formatted.append(
                        f"[TOOL CALL] {tc['function']['name']}"
                        f"({tc['function']['arguments']})"
                    )
            elif role == 'tool':
                # 工具结果
                formatted.append(f"[TOOL RESULT] {content[:200]}...")
            else:
                formatted.append(f"[{role.upper()}] {content}")
        
        history_text = "\n".join(formatted)
        
        return f"""Summarize the following conversation history into a structured summary.

Format:
## Resolved Questions
- [List questions that were answered]

## Pending Questions  
- [List questions still being worked on]

## Active Task
[Current task the assistant is working on]

## Remaining Work
- [List concrete next steps]

## Key Context
- [Important facts, decisions, or constraints]

Conversation history:
{history_text}

Summary:"""
    
    def _calculate_summary_budget(self, messages: List[Dict]) -> int:
        """
        计算摘要的 token 预算
        
        策略: 压缩内容的 20%，最少 2000，最多 12000
        """
        compressed_tokens = estimate_messages_tokens_rough(messages)
        budget = int(compressed_tokens * 0.20)
        budget = max(2000, min(12000, budget))
        return budget
```

**工具输出修剪**:
```python
def _prune_old_tool_results(messages: List[Dict]) -> List[Dict]:
    """
    修剪旧的工具输出
    
    策略: 保留最近 N 个工具结果，旧的替换为占位符
    """
    KEEP_RECENT = 10
    tool_result_indices = [
        i for i, m in enumerate(messages)
        if m['role'] == 'tool'
    ]
    
    if len(tool_result_indices) <= KEEP_RECENT:
        return messages
    
    # 替换旧的工具结果
    pruned = messages.copy()
    for idx in tool_result_indices[:-KEEP_RECENT]:
        pruned[idx] = {
            'role': 'tool',
            'tool_call_id': messages[idx]['tool_call_id'],
            'content': _PRUNED_TOOL_PLACEHOLDER
        }
    
    return pruned
```

**设计亮点**:
- 智能分割（保护头尾）
- 结构化摘要（避免误读）
- 迭代更新（多次压缩时合并摘要）
- 工具输出修剪（廉价预处理）
- 冷却时间（避免频繁压缩）

#### 3.5.3 prompt_builder.py

**职责**: 系统提示词组装

**核心功能**:
```python
def build_system_prompt(
    agent_identity: str,
    skills_prompt: str,
    context_files_prompt: str,
    memory_prompt: str,
    environment_hints: str,
    ephemeral_prompt: str = None,
) -> str:
    """
    组装完整的系统提示词
    
    结构:
    1. Agent 身份
    2. 技能列表
    3. 上下文文件 (HERMES.md, AGENTS.md)
    4. 记忆系统指导
    5. 环境提示
    6. 临时提示词
    """
    parts = []
    
    if agent_identity:
        parts.append(agent_identity)
    
    if skills_prompt:
        parts.append(skills_prompt)
    
    if context_files_prompt:
        parts.append(context_files_prompt)
    
    if memory_prompt:
        parts.append(memory_prompt)
    
    if environment_hints:
        parts.append(environment_hints)
    
    if ephemeral_prompt:
        parts.append(ephemeral_prompt)
    
    return "\n\n".join(parts)
```

**上下文文件扫描**:
```python
def build_context_files_prompt(cwd: Path) -> str:
    """
    扫描并加载上下文文件
    
    查找顺序:
    1. .hermes.md / HERMES.md (项目特定指令)
    2. AGENTS.md (开发指南)
    3. .cursorrules (Cursor IDE 规则)
    """
    parts = []
    
    # 1. HERMES.md
    hermes_md = _find_hermes_md(cwd)
    if hermes_md:
        content = hermes_md.read_text(encoding='utf-8')
        content = _strip_yaml_frontmatter(content)
        content = _scan_context_content(content, hermes_md.name)
        parts.append(f"# Project Instructions\n\n{content}")
    
    # 2. AGENTS.md
    agents_md = cwd / "AGENTS.md"
    if agents_md.exists():
        content = agents_md.read_text(encoding='utf-8')
        content = _scan_context_content(content, "AGENTS.md")
        parts.append(f"# Development Guide\n\n{content}")
    
    return "\n\n".join(parts)
```

**注入检测**:
```python
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'disregard\s+(your|all|any)\s+(instructions|rules)', "disregard_rules"),
    # ... 更多模式
]

def _scan_context_content(content: str, filename: str) -> str:
    """
    扫描上下文文件内容，检测注入攻击
    
    如果检测到威胁模式，返回阻止消息
    """
    findings = []
    
    # 检查威胁模式
    for pattern, pid in _CONTEXT_THREAT_PATTERNS:
        if re.search(pattern, content, re.IGNORECASE):
            findings.append(pid)
    
    # 检查不可见字符
    for char in _CONTEXT_INVISIBLE_CHARS:
        if char in content:
            findings.append(f"invisible unicode U+{ord(char):04X}")
    
    if findings:
        logger.warning(f"Context file {filename} blocked: {', '.join(findings)}")
        return f"[BLOCKED: {filename} contained potential prompt injection]"
    
    return content
```

---

## 4. 关键设计模式

### 4.1 适配器模式 (Adapter Pattern)

**应用**: 模型适配层

**问题**: 不同 LLM 提供商的 API 格式不同

**解决方案**:
```python
# 内部统一使用 OpenAI 格式
messages = [
    {'role': 'system', 'content': '...'},
    {'role': 'user', 'content': '...'},
    {'role': 'assistant', 'tool_calls': [...]},
]

# 适配器转换
if provider == "anthropic":
    request = anthropic_adapter.build_request(messages, tools)
elif provider == "gemini":
    request = gemini_adapter.build_request(messages, tools)

# 调用 API
response = client.create(**request)

# 适配器转换回 OpenAI 格式
messages.append(adapter.parse_response(response))
```

**优势**:
- 核心逻辑与提供商解耦
- 添加新提供商只需实现适配器
- 统一的错误处理和重试逻辑

### 4.2 注册表模式 (Registry Pattern)

**应用**: 工具系统

**问题**: 如何动态发现和加载工具

**解决方案**:
```python
# 1. 中央注册表
registry = ToolRegistry()

# 2. 工具自注册
# tools/file_tools.py
registry.register(
    name="read_file",
    schema=READ_FILE_SCHEMA,
    handler=read_file_handler,
)

# 3. 自动发现
discover_builtin_tools()  # 导入所有工具模块

# 4. 查询和调度
tools = registry.get_all_tools()
result = registry.dispatch("read_file", {"file_path": "..."})
```

**优势**:
- 添加工具无需修改核心代码
- 元数据集中管理
- 支持动态启用/禁用
- 避免循环依赖

### 4.3 策略模式 (Strategy Pattern)

**应用**: 错误处理和重试

**问题**: 不同错误需要不同的处理策略

**解决方案**:
```python
# 错误分类
reason = classify_api_error(exception)

# 根据分类选择策略
if reason == FailoverReason.RATE_LIMIT:
    # 策略 1: 指数退避重试
    time.sleep(jittered_backoff(attempt))
    continue

elif reason == FailoverReason.CONTEXT_LENGTH:
    # 策略 2: 压缩上下文
    messages = compressor.compress_if_needed(messages)
    continue

elif reason == FailoverReason.INVALID_API_KEY:
    # 策略 3: 切换凭证
    agent.api_key = credential_pool.get_next()
    continue

elif reason == FailoverReason.MODEL_OVERLOADED:
    # 策略 4: 降级到备用模型
    agent.model = fallback_model
    continue

else:
    # 策略 5: 失败
    raise
```

**优势**:
- 错误处理逻辑清晰
- 易于添加新策略
- 可测试性强

### 4.4 观察者模式 (Observer Pattern)

**应用**: 回调系统

**问题**: 如何通知外部系统 Agent 的状态变化

**解决方案**:
```python
class AIAgent:
    def __init__(
        self,
        tool_progress_callback=None,
        stream_delta_callback=None,
        thinking_callback=None,
        # ... 更多回调
    ):
        self.callbacks = {
            'tool_progress': tool_progress_callback,
            'stream_delta': stream_delta_callback,
            'thinking': thinking_callback,
        }
    
    def _notify(self, event: str, **data):
        """通知观察者"""
        callback = self.callbacks.get(event)
        if callback:
            try:
                callback(**data)
            except Exception as e:
                logger.warning(f"Callback {event} failed: {e}")
    
    def run_conversation(self, ...):
        # 通知工具开始
        self._notify('tool_progress', tool='bash', status='started')
        
        # 执行工具
        result = handle_function_call(...)
        
        # 通知工具完成
        self._notify('tool_progress', tool='bash', status='completed')
```

**优势**:
- 核心逻辑与 UI 解耦
- 支持多个观察者
- 易于扩展新事件

### 4.5 工厂模式 (Factory Pattern)

**应用**: 客户端创建

**问题**: 根据配置创建不同的 API 客户端

**解决方案**:
```python
def create_client(provider: str, base_url: str, api_key: str):
    """工厂函数: 根据提供商创建客户端"""
    
    if provider == "anthropic":
        return build_anthropic_client(api_key, base_url)
    
    elif provider == "openai":
        return OpenAI(api_key=api_key, base_url=base_url)
    
    elif provider == "bedrock":
        return build_bedrock_client(api_key, base_url)
    
    elif provider == "gemini":
        return build_gemini_client(api_key, base_url)
    
    else:
        # 默认: OpenAI 兼容
        return OpenAI(api_key=api_key, base_url=base_url)
```

**优势**:
- 客户端创建逻辑集中
- 易于添加新提供商
- 统一的接口

---

## 5. 数据流与生命周期

### 5.1 单轮对话流程

```
1. 用户输入
   ↓
2. CLI/TUI/Gateway 接收
   ↓
3. AIAgent.run_conversation()
   ├─ 恢复/构建系统提示词
   ├─ 加载记忆上下文
   ├─ 获取工具定义
   └─ 进入主循环
      ↓
4. 主循环
   ├─ 检查预算和中断
   ├─ 调用模型 API
   │  ├─ 适配器转换请求
   │  ├─ 发送 HTTP 请求
   │  └─ 适配器解析响应
   ├─ 处理响应
   │  ├─ 如果有工具调用
   │  │  ├─ 并行/串行调度
   │  │  ├─ 执行工具
   │  │  ├─ 收集结果
   │  │  └─ 继续循环
   │  └─ 如果是文本响应
   │     └─ 退出循环
   └─ 错误处理
      ├─ 分类错误
      ├─ 选择策略
      └─ 重试或降级
      ↓
5. 后处理
   ├─ 同步记忆
   ├─ 保存会话
   ├─ 触发回调
   └─ 返回结果
```
