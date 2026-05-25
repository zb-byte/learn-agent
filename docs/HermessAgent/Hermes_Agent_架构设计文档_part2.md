#### 3.2.2 agent_init.py

**职责**: Agent 初始化逻辑（从 run_agent.py 提取，1400+ 行）

**核心功能**:
```python
def init_agent(agent, **kwargs):
    """
    初始化 Agent 的所有组件
    
    步骤:
    1. 提供商自动检测 (provider auto-detection)
    2. 凭证解析 (credential resolution)
    3. 上下文引擎引导 (context engine bootstrap)
    4. 工具集验证 (toolset validation)
    5. 记忆管理器初始化
    6. 会话数据库连接
    """
    
    # 1. 提供商检测
    if not agent.provider:
        agent.provider = _detect_provider_from_base_url(agent.base_url)
    
    # 2. 凭证解析
    if not agent.api_key:
        agent.api_key = _resolve_api_key(agent.provider)
    
    # 3. 工具集验证
    agent.enabled_tools = check_toolset_requirements(
        agent.enabled_toolsets,
        agent.disabled_toolsets
    )
    
    # 4. 记忆管理器
    agent._memory_manager = MemoryManager()
    
    # 5. 会话数据库
    agent._session_db = SessionDB(session_id=agent.session_id)
```

**设计亮点**:
- 提取到独立模块，保持 `run_agent.py` 简洁
- 支持测试 patch (`_ra()` 延迟导入模式)
- 完整的错误处理和日志记录

#### 3.2.3 conversation_loop.py

**职责**: 对话循环核心逻辑（3900+ 行）

**主循环结构**:
```python
def run_conversation(agent, user_message, ...):
    """
    核心对话循环
    """
    # 1. 恢复或构建系统提示词
    _restore_or_build_system_prompt(agent, system_message, conversation_history)
    
    # 2. 加载记忆上下文
    memory_context = agent._memory_manager.prefetch_all(user_message)
    
    # 3. 获取工具定义
    tools = get_tool_definitions(
        agent.enabled_toolsets,
        agent.disabled_toolsets,
        agent.quiet_mode
    )
    
    # 4. 主循环
    api_call_count = 0
    while (api_call_count < agent.max_iterations 
           and agent.iteration_budget.remaining > 0) \
           or agent._budget_grace_call:
        
        # 检查中断
        if agent._interrupt_requested:
            break
        
        # 调用模型
        try:
            response = agent.client.chat.completions.create(
                model=agent.model,
                messages=messages,
                tools=tools,
                stream=True,
            )
        except Exception as e:
            # 错误分类与重试
            reason = classify_api_error(e)
            if reason == FailoverReason.RATE_LIMIT:
                time.sleep(jittered_backoff(attempt))
                continue
            elif reason == FailoverReason.CONTEXT_LENGTH:
                # 触发压缩
                messages = agent._compressor.compress_if_needed(messages)
                continue
            else:
                raise
        
        # 解析响应
        if response.tool_calls:
            # 执行工具
            for tool_call in response.tool_calls:
                result = handle_function_call(
                    tool_call.function.name,
                    tool_call.function.arguments,
                    task_id=task_id
                )
                messages.append({
                    'role': 'tool',
                    'tool_call_id': tool_call.id,
                    'content': result
                })
            api_call_count += 1
        else:
            # 完成
            final_response = response.content
            break
    
    # 5. 后处理
    agent._memory_manager.sync_all(user_message, final_response)
    agent._session_db.save_messages(messages)
    
    return {
        'completed': True,
        'final_response': final_response,
        'messages': messages,
        'api_calls': api_call_count,
    }
```

**关键特性**:
- **中断支持**: 检查 `_interrupt_requested` 标志
- **预算跟踪**: `iteration_budget` 管理父子 Agent 共享预算
- **优雅降级**: 一次 grace call 允许超预算完成当前轮次
- **错误恢复**: 分类错误，智能重试或降级
- **上下文压缩**: 自动触发压缩避免超限

#### 3.2.4 agent_runtime_helpers.py

**职责**: 运行时辅助函数

**核心功能**:
```python
# 消息格式转换
def convert_to_openai_format(messages: List[Dict]) -> List[Dict]:
    """统一转换为 OpenAI 格式"""

# 流式响应处理
def process_stream_delta(delta, scrubber):
    """处理流式响应增量"""

# 工具调用解析
def parse_tool_calls(response) -> List[ToolCall]:
    """从响应中提取工具调用"""

# 回调触发
def trigger_callback(callback, *args, **kwargs):
    """安全触发回调，捕获异常"""
```

---

### 3.3 Layer 3: 模型适配层

#### 3.3.1 设计理念

**问题**: 不同 LLM 提供商的 API 格式不同
- OpenAI: `messages` 格式，`tool_calls` 数组
- Anthropic: `system` 参数分离，`cache_control` 标记
- Gemini: `contents` 格式，`function_call` 对象
- Bedrock: boto3 客户端，特殊认证

**解决方案**: 适配器模式
- 内部统一使用 OpenAI 格式
- 每个提供商一个适配器
- 适配器负责双向转换

#### 3.3.2 anthropic_adapter.py

**职责**: Anthropic Messages API 适配器

**核心功能**:
```python
def build_anthropic_request(
    messages: List[Dict],
    model: str,
    tools: List[Dict],
    max_tokens: int = None,
    thinking_mode: str = "adaptive",
    thinking_budget: int = None,
) -> Dict:
    """
    OpenAI 格式 → Anthropic 格式
    
    转换:
    1. 提取 system 消息
    2. 转换 tool_calls 格式
    3. 应用 cache_control 标记
    4. 配置 thinking 模式
    5. 计算 max_tokens
    """
    
    # 提取 system
    system_messages = [m for m in messages if m['role'] == 'system']
    system = "\n\n".join(m['content'] for m in system_messages)
    
    # 转换消息
    anthropic_messages = []
    for msg in messages:
        if msg['role'] == 'system':
            continue
        elif msg['role'] == 'assistant' and msg.get('tool_calls'):
            # 转换工具调用格式
            anthropic_messages.append({
                'role': 'assistant',
                'content': [
                    {
                        'type': 'tool_use',
                        'id': tc['id'],
                        'name': tc['function']['name'],
                        'input': json.loads(tc['function']['arguments'])
                    }
                    for tc in msg['tool_calls']
                ]
            })
        else:
            anthropic_messages.append(msg)
    
    # 应用缓存控制
    apply_anthropic_cache_control(anthropic_messages, system)
    
    # 配置 thinking
    if thinking_mode == "adaptive":
        thinking_config = {
            "type": "adaptive",
            "budget_tokens": thinking_budget or 16000
        }
    else:
        thinking_config = {"type": "enabled"}
    
    # 计算 max_tokens
    if not max_tokens:
        max_tokens = _get_anthropic_max_output(model)
    
    return {
        'model': model,
        'system': system,
        'messages': anthropic_messages,
        'tools': tools,
        'max_tokens': max_tokens,
        'thinking': thinking_config,
    }
```

**认证支持**:
```python
def _is_oauth_token(api_key: str) -> bool:
    """检测是否为 OAuth token"""
    return api_key.startswith("sk-ant-oat")

def build_anthropic_client(api_key, base_url):
    """构建 Anthropic 客户端"""
    if _is_oauth_token(api_key):
        # OAuth 认证
        return Anthropic(
            auth_token=api_key,
            base_url=base_url,
            default_headers={"anthropic-beta": "..."}
        )
    else:
        # API Key 认证
        return Anthropic(api_key=api_key, base_url=base_url)
```

**设计亮点**:
- 自动检测认证类型
- 智能计算 max_tokens
- 支持 adaptive thinking
- 提示词缓存优化

#### 3.3.3 其他适配器

**gemini_native_adapter.py**:
- 转换为 Gemini `contents` 格式
- 处理 `function_call` / `function_response`
- 支持流式响应

**bedrock_adapter.py**:
- boto3 客户端管理
- AWS 凭证链解析
- 消息格式适配

**codex_responses_adapter.py**:
- Codex Responses API 适配
- 工具调用 ID 生成
- 响应格式转换

---

### 3.4 Layer 4: 工具系统层

#### 3.4.1 设计理念

**核心思想**: 自注册 + 动态发现

**工作流程**:
```
1. tools/registry.py (注册中心，无依赖)
2. tools/*.py (每个工具文件导入 registry，调用 register())
3. model_tools.py (导入 registry，触发工具发现)
4. run_agent.py (导入 model_tools，使用工具)
```

**优势**:
- 添加新工具只需创建文件，无需修改其他代码
- 工具元数据集中管理
- 支持动态启用/禁用
- 避免循环依赖

#### 3.4.2 tools/registry.py

**职责**: 工具注册中心

**核心 API**:
```python
class ToolRegistry:
    def __init__(self):
        self._tools: Dict[str, ToolEntry] = {}
    
    def register(
        self,
        name: str,
        toolset: str,
        schema: Dict,
        handler: Callable,
        check_fn: Callable = None,
        requires_env: bool = False,
        is_async: bool = False,
        description: str = "",
        emoji: str = "🔧",
        max_result_size_chars: int = None,
        dynamic_schema_overrides: Callable = None,
    ):
        """
        注册一个工具
        
        参数:
        - name: 工具名称
        - toolset: 所属工具集
        - schema: JSON Schema (OpenAI function calling 格式)
        - handler: 处理函数 (同步或异步)
        - check_fn: 可用性检查函数
        - requires_env: 是否需要环境 (terminal)
        - is_async: 是否为异步函数
        - description: 描述
        - emoji: 显示图标
        - max_result_size_chars: 结果最大长度
        - dynamic_schema_overrides: 动态 schema 覆盖
        """
        entry = ToolEntry(...)
        self._tools[name] = entry
    
    def get_tool(self, name: str) -> Optional[ToolEntry]:
        """获取工具"""
        return self._tools.get(name)
    
    def get_all_tools(self) -> Dict[str, ToolEntry]:
        """获取所有工具"""
        return self._tools.copy()
    
    def dispatch(self, name: str, args: Dict, **kwargs) -> str:
        """调度工具执行"""
        entry = self._tools.get(name)
        if not entry:
            return tool_error(f"Tool {name} not found")
        
        # 异步桥接
        if entry.is_async:
            result = _run_async(entry.handler(args, **kwargs))
        else:
            result = entry.handler(args, **kwargs)
        
        return result

# 全局单例
registry = ToolRegistry()
```

**工具发现**:
```python
def discover_builtin_tools(tools_dir: Path = None) -> List[str]:
    """
    自动发现并导入工具模块
    
    扫描 tools/*.py，检查是否包含 registry.register() 调用
    """
    tools_path = tools_dir or Path(__file__).parent
    module_names = []
    
    for path in sorted(tools_path.glob("*.py")):
        if path.name in {"__init__.py", "registry.py", "mcp_tool.py"}:
            continue
        
        # AST 检查是否调用 registry.register()
        if _module_registers_tools(path):
            module_names.append(f"tools.{path.stem}")
    
    # 导入模块（触发注册）
    imported = []
    for mod_name in module_names:
        try:
            importlib.import_module(mod_name)
            imported.append(mod_name)
        except Exception as e:
            logger.warning(f"Could not import {mod_name}: {e}")
    
    return imported
```

**check_fn 缓存**:
```python
# TTL 缓存避免重复检查
_CHECK_FN_TTL_SECONDS = 30.0
_check_fn_cache: Dict[Callable, Tuple[float, bool]] = {}

def _check_fn_cached(fn: Callable) -> bool:
    """缓存 check_fn 结果 30 秒"""
    now = time.monotonic()
    cached = _check_fn_cache.get(fn)
    if cached:
        ts, value = cached
        if now - ts < _CHECK_FN_TTL_SECONDS:
            return value
    
    try:
        value = bool(fn())
    except Exception:
        value = False
    
    _check_fn_cache[fn] = (now, value)
    return value
```
