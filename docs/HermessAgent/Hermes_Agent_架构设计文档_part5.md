### 5.2 Agent 生命周期

```
1. 初始化阶段 (agent_init.py)
   ├─ 解析配置参数
   ├─ 检测提供商
   ├─ 解析凭证
   ├─ 验证工具集
   ├─ 初始化记忆管理器
   ├─ 连接会话数据库
   └─ 创建 API 客户端

2. 运行阶段 (conversation_loop.py)
   ├─ 接收用户消息
   ├─ 执行对话循环
   │  ├─ 模型调用
   │  ├─ 工具执行
   │  └─ 上下文管理
   └─ 返回结果

3. 清理阶段
   ├─ 保存会话状态
   ├─ 关闭数据库连接
   ├─ 清理临时资源
   └─ 触发清理回调
```

### 5.3 工具执行流程

```
1. 模型返回 tool_calls
   ↓
2. handle_function_call()
   ├─ 解析参数 (JSON string → dict)
   ├─ 查找工具 (registry.get_tool)
   └─ 调度执行 (registry.dispatch)
      ↓
3. registry.dispatch()
   ├─ 检查工具是否存在
   ├─ 判断是否异步
   │  ├─ 异步: _run_async(handler)
   │  └─ 同步: handler()
   ├─ 捕获异常
   └─ 返回结果
      ↓
4. 工具处理函数
   ├─ 验证参数
   ├─ 执行操作
   ├─ 格式化结果
   └─ 返回字符串
      ↓
5. 结果处理
   ├─ 截断过长结果
   ├─ 添加到消息历史
   └─ 继续对话循环
```

### 5.4 上下文压缩流程

```
1. 触发条件
   ├─ Token 数超过阈值 (75%)
   └─ 或 API 返回 context_length 错误

2. 分割消息
   ├─ 头部: 系统消息 + 第一轮对话
   ├─ 中间: 待压缩部分
   └─ 尾部: 最近 N 轮对话 (30% token 预算)

3. 生成摘要
   ├─ 格式化中间消息
   ├─ 构建摘要提示词
   ├─ 调用辅助模型
   └─ 获取结构化摘要

4. 替换消息
   ├─ 创建摘要消息
   ├─ 组合: 头部 + 摘要 + 尾部
   └─ 返回压缩后的消息列表

5. 迭代更新
   ├─ 如果已有摘要
   ├─ 合并旧摘要和新内容
   └─ 生成更新的摘要
```

---

## 6. 扩展性设计

### 6.1 插件系统

**目录结构**:
```
plugins/
├── memory/              # 记忆提供商插件
│   ├── honcho/
│   ├── mem0/
│   └── supermemory/
├── model-providers/     # 模型提供商插件
│   ├── openrouter/
│   └── anthropic/
├── context_engine/      # 上下文引擎插件
├── image_gen/           # 图像生成插件
└── observability/       # 可观测性插件
```

**插件加载**:
```python
# hermes_cli/plugins.py
def load_plugins():
    """加载所有插件"""
    plugins_dir = Path(__file__).parent.parent / "plugins"
    
    for plugin_dir in plugins_dir.iterdir():
        if not plugin_dir.is_dir():
            continue
        
        # 查找 __init__.py
        init_file = plugin_dir / "__init__.py"
        if not init_file.exists():
            continue
        
        # 导入插件
        try:
            module_name = f"plugins.{plugin_dir.name}"
            importlib.import_module(module_name)
            logger.info(f"Loaded plugin: {plugin_dir.name}")
        except Exception as e:
            logger.warning(f"Failed to load plugin {plugin_dir.name}: {e}")

def invoke_hook(hook_name: str, **kwargs):
    """调用插件钩子"""
    for plugin in _registered_plugins:
        if hasattr(plugin, hook_name):
            try:
                getattr(plugin, hook_name)(**kwargs)
            except Exception as e:
                logger.warning(f"Plugin hook {hook_name} failed: {e}")
```

**插件接口**:
```python
# plugins/memory/example/plugin.py
class ExampleMemoryPlugin(MemoryProvider):
    def __init__(self):
        self.is_external = True
    
    def build_system_prompt(self) -> str:
        """返回记忆系统的提示词"""
        return "You have access to long-term memory..."
    
    def prefetch(self, user_message: str) -> str:
        """预取相关记忆"""
        # 查询记忆数据库
        results = self.search(user_message)
        return "\n".join(results)
    
    def sync(self, user_message: str, assistant_response: str):
        """同步新记忆"""
        # 保存到记忆数据库
        self.save(user_message, assistant_response)

# 注册插件
def register():
    from agent.memory_manager import register_memory_provider
    register_memory_provider(ExampleMemoryPlugin())
```

### 6.2 添加新工具

**步骤**:

1. 创建工具文件 `tools/my_tool.py`
2. 定义 schema 和 handler
3. 调用 `registry.register()`
4. 完成！（自动发现和加载）

**示例**:
```python
# tools/weather_tool.py
from tools.registry import registry, tool_error
import requests

WEATHER_SCHEMA = {
    "name": "get_weather",
    "description": "Get current weather for a location",
    "parameters": {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "City name or coordinates"
            },
            "units": {
                "type": "string",
                "enum": ["celsius", "fahrenheit"],
                "description": "Temperature units"
            }
        },
        "required": ["location"]
    }
}

def weather_handler(args: dict, **kwargs) -> str:
    """获取天气信息"""
    location = args.get("location")
    units = args.get("units", "celsius")
    
    try:
        # 调用天气 API
        api_key = os.getenv("WEATHER_API_KEY")
        response = requests.get(
            f"https://api.weather.com/v1/current",
            params={"location": location, "units": units, "key": api_key}
        )
        data = response.json()
        
        return f"Weather in {location}: {data['temp']}°, {data['condition']}"
    
    except Exception as e:
        return tool_error(f"Failed to get weather: {str(e)}")

def check_weather_api_key() -> bool:
    """检查 API key 是否配置"""
    return bool(os.getenv("WEATHER_API_KEY"))

# 注册工具
registry.register(
    name="get_weather",
    toolset="web",
    schema=WEATHER_SCHEMA,
    handler=weather_handler,
    check_fn=check_weather_api_key,
    description="Get current weather information",
    emoji="🌤️",
)
```

### 6.3 添加新模型提供商

**步骤**:

1. 创建适配器 `agent/my_provider_adapter.py`
2. 实现请求/响应转换函数
3. 在 `agent_init.py` 中添加检测逻辑
4. 在 `conversation_loop.py` 中添加调用逻辑

**示例**:
```python
# agent/my_provider_adapter.py

def build_my_provider_request(
    messages: List[Dict],
    model: str,
    tools: List[Dict],
    **kwargs
) -> Dict:
    """
    OpenAI 格式 → MyProvider 格式
    """
    # 转换消息格式
    provider_messages = []
    for msg in messages:
        if msg['role'] == 'system':
            # MyProvider 使用单独的 system 参数
            continue
        provider_messages.append({
            'role': msg['role'],
            'content': msg['content']
        })
    
    # 转换工具格式
    provider_tools = [
        {
            'name': t['function']['name'],
            'description': t['function']['description'],
            'parameters': t['function']['parameters']
        }
        for t in tools
    ]
    
    return {
        'model': model,
        'messages': provider_messages,
        'tools': provider_tools,
    }

def parse_my_provider_response(response) -> Dict:
    """
    MyProvider 格式 → OpenAI 格式
    """
    # 转换响应
    if response.get('tool_calls'):
        return {
            'role': 'assistant',
            'content': None,
            'tool_calls': [
                {
                    'id': tc['id'],
                    'type': 'function',
                    'function': {
                        'name': tc['name'],
                        'arguments': json.dumps(tc['arguments'])
                    }
                }
                for tc in response['tool_calls']
            ]
        }
    else:
        return {
            'role': 'assistant',
            'content': response['content']
        }

def build_my_provider_client(api_key: str, base_url: str):
    """创建 MyProvider 客户端"""
    return MyProviderClient(api_key=api_key, base_url=base_url)
```

### 6.4 添加新界面

**步骤**:

1. 创建界面模块
2. 实例化 `AIAgent`
3. 调用 `run_conversation()` 或 `chat()`
4. 处理响应和回调

**示例 - Web API**:
```python
# web_api.py
from fastapi import FastAPI
from run_agent import AIAgent

app = FastAPI()
agent = AIAgent(model="claude-opus-4-7")

@app.post("/chat")
async def chat(request: dict):
    """聊天接口"""
    user_message = request.get("message")
    session_id = request.get("session_id")
    
    # 调用 Agent
    result = agent.run_conversation(
        user_message=user_message,
        conversation_history=get_history(session_id)
    )
    
    # 保存历史
    save_history(session_id, result['messages'])
    
    return {
        "response": result['final_response'],
        "completed": result['completed']
    }
```

---

## 7. 性能优化策略

### 7.1 提示词缓存

**问题**: 系统提示词在每次调用时都重新发送，浪费 token

**解决方案**: Anthropic 提示词缓存

```python
def apply_anthropic_cache_control(messages: List[Dict], system: str):
    """
    应用缓存控制标记
    
    策略:
    - 系统提示词: 缓存整个 system 参数
    - 工具定义: 缓存 tools 数组
    - 最近消息: 不缓存（频繁变化）
    
    TTL: 5 分钟
    """
    # 标记系统提示词
    if system:
        system = [
            {"type": "text", "text": system},
            {"type": "cache_control", "cache_type": "ephemeral"}
        ]
    
    # 标记工具定义（在最后一个 user 消息后）
    for i in range(len(messages) - 1, -1, -1):
        if messages[i]['role'] == 'user':
            if 'cache_control' not in messages[i]:
                messages[i]['cache_control'] = {"type": "ephemeral"}
            break
    
    return system, messages
```

**效果**:
- 首次调用: 正常计费
- 5 分钟内再次调用: 缓存命中，90% 折扣
- 长对话: 节省大量 token 成本

### 7.2 并行工具执行

**问题**: 多个独立工具串行执行，浪费时间

**解决方案**: 并行调度

```python
def _should_parallelize_tool_batch(tool_calls: List[Dict]) -> bool:
    """
    判断是否可以并行执行
    
    规则:
    - 所有工具都在 _PARALLEL_SAFE_TOOLS 中
    - 没有工具在 _NEVER_PARALLEL_TOOLS 中
    - 没有破坏性命令
    - 路径不重叠（对于文件操作）
    """
    for tc in tool_calls:
        name = tc['function']['name']
        
        # 检查黑名单
        if name in _NEVER_PARALLEL_TOOLS:
            return False
        
        # 检查白名单
        if name not in _PARALLEL_SAFE_TOOLS:
            return False
        
        # 检查破坏性命令
        if name == "bash":
            command = json.loads(tc['function']['arguments']).get('command', '')
            if _is_destructive_command(command):
                return False
    
    # 检查路径重叠
    paths = [_extract_path(tc) for tc in tool_calls]
    for i, p1 in enumerate(paths):
        for p2 in paths[i+1:]:
            if _paths_overlap(p1, p2):
                return False
    
    return True

# 并行执行
if _should_parallelize_tool_batch(tool_calls):
    with ThreadPoolExecutor(max_workers=len(tool_calls)) as executor:
        futures = [
            executor.submit(handle_function_call, tc['function']['name'], tc['function']['arguments'])
            for tc in tool_calls
        ]
        results = [f.result() for f in futures]
else:
    # 串行执行
    results = [
        handle_function_call(tc['function']['name'], tc['function']['arguments'])
        for tc in tool_calls
    ]
```

**效果**:
- 多个 `read_file` 并行: 3x 加速
- 多个 `web_search` 并行: 5x 加速

### 7.3 懒加载

**问题**: 导入大型 SDK 拖慢启动时间

**解决方案**: 延迟导入

```python
# agent/anthropic_adapter.py

# 不要在模块顶部导入
# import anthropic  # ❌ 拖慢 240ms

# 使用延迟导入
_anthropic_sdk = ...  # 哨兵值

def _get_anthropic_sdk():
    """延迟导入 Anthropic SDK"""
    global _anthropic_sdk
    if _anthropic_sdk is ...:
        try:
            import anthropic
            _anthropic_sdk = anthropic
        except ImportError:
            _anthropic_sdk = None
    return _anthropic_sdk

# 使用时才导入
def build_anthropic_client(...):
    sdk = _get_anthropic_sdk()
    if not sdk:
        raise ImportError("anthropic SDK not installed")
    return sdk.Anthropic(...)
```

**效果**:
- CLI 启动时间: 从 1.2s 降到 0.8s
- 导入 `run_agent`: 从 800ms 降到 400ms

### 7.4 连接池

**问题**: 每次 API 调用创建新连接，浪费时间

**解决方案**: 复用 HTTP 连接

```python
# OpenAI SDK 自动使用连接池
client = OpenAI(
    api_key=api_key,
    base_url=base_url,
    timeout=120.0,
    max_retries=0,  # 我们自己处理重试
)

# 持久化客户端，避免重复创建
self.client = client  # 存储在 Agent 实例中
```

**效果**:
- 每次调用节省 50-100ms（TCP 握手 + TLS 握手）

### 7.5 结果缓存

**问题**: 相同的工具调用重复执行

**解决方案**: TTL 缓存

```python
# tools/registry.py

_check_fn_cache: Dict[Callable, Tuple[float, bool]] = {}
_CHECK_FN_TTL_SECONDS = 30.0

def _check_fn_cached(fn: Callable) -> bool:
    """缓存 check_fn 结果 30 秒"""
    now = time.monotonic()
    cached = _check_fn_cache.get(fn)
    if cached:
        ts, value = cached
        if now - ts < _CHECK_FN_TTL_SECONDS:
            return value  # 缓存命中
    
    # 缓存未命中，执行并缓存
    value = bool(fn())
    _check_fn_cache[fn] = (now, value)
    return value
```

**效果**:
- 避免重复检查 Docker daemon、Modal SDK 等
- 每次 `get_tool_definitions()` 节省 10-50ms

---

## 8. 设计建议

### 8.1 核心原则

1. **分层解耦**: 每层职责清晰，依赖单向
2. **适配器隔离**: 提供商差异封装在适配器中
3. **自注册发现**: 工具、插件自动发现，无需手动注册
4. **优雅降级**: 错误时智能重试或降级，不直接失败
5. **可观测性**: 完整的日志、指标、追踪

### 8.2 模块化建议

**DO**:
- ✅ 提取大型函数到独立模块（如 `agent_init.py`, `conversation_loop.py`）
- ✅ 使用注册表模式管理动态组件
- ✅ 通过回调解耦核心逻辑和 UI
- ✅ 延迟导入大型依赖

**DON'T**:
- ❌ 在核心循环中硬编码提供商逻辑
- ❌ 在工具中直接访问 Agent 状态
- ❌ 在模块顶部导入所有依赖
- ❌ 使用全局状态（除了注册表）

### 8.3 错误处理建议

**DO**:
- ✅ 分类错误，选择合适的策略
- ✅ 记录详细的错误上下文
- ✅ 提供有意义的错误消息
- ✅ 在适当的层级捕获异常

**DON'T**:
- ❌ 吞掉异常（`except: pass`）
- ❌ 在低层抛出高层异常
- ❌ 使用裸 `except` 捕获所有异常
- ❌ 在循环中无限重试

### 8.4 性能优化建议

**DO**:
- ✅ 使用提示词缓存（Anthropic）
- ✅ 并行执行独立工具
- ✅ 延迟导入大型 SDK
- ✅ 复用 HTTP 连接
- ✅ 缓存昂贵的检查

**DON'T**:
- ❌ 过早优化
- ❌ 牺牲可读性换取微小性能提升
- ❌ 在热路径中使用反射
- ❌ 频繁创建大对象

### 8.5 测试建议

**DO**:
- ✅ 单元测试每个模块
- ✅ 集成测试关键流程
- ✅ Mock 外部依赖（API、数据库）
- ✅ 测试错误路径

**DON'T**:
- ❌ 测试实现细节
- ❌ 在测试中使用真实 API
- ❌ 忽略边界情况
- ❌ 写脆弱的测试（依赖执行顺序）

### 8.6 扩展性建议

**添加新功能时**:

1. **评估位置**: 应该在哪一层？
2. **检查接口**: 是否需要新接口？
3. **考虑插件**: 能否作为插件实现？
4. **保持兼容**: 不破坏现有 API
5. **文档更新**: 更新架构文档

**示例决策树**:
```
需要添加新功能
├─ 是新的 LLM 提供商？
│  └─ 创建适配器 (agent/*_adapter.py)
├─ 是新的工具？
│  └─ 创建工具文件 (tools/*.py)
├─ 是新的记忆后端？
│  └─ 创建插件 (plugins/memory/*)
├─ 是新的界面？
│  └─ 创建界面模块，调用 AIAgent
└─ 是核心逻辑变更？
   └─ 修改 conversation_loop.py，保持向后兼容
```

---

## 9. 总结

### 9.1 架构优势

1. **模块化**: 清晰的分层和职责划分
2. **可扩展**: 插件系统、注册表模式
3. **提供商无关**: 适配器隔离差异
4. **生产就绪**: 错误恢复、速率限制、监控
5. **性能优化**: 缓存、并行、懒加载

### 9.2 关键技术

| 技术 | 应用 | 效果 |
|------|------|------|
| 适配器模式 | 模型提供商 | 统一接口 |
| 注册表模式 | 工具系统 | 动态发现 |
| 提示词缓存 | Anthropic API | 90% 折扣 |
| 并行执行 | 工具调度 | 3-5x 加速 |
| 上下文压缩 | 长对话 | 避免超限 |
| 懒加载 | SDK 导入 | 快速启动 |

### 9.3 设计 Agent 的建议

如果你要设计一个类似的 Agent 系统，建议：

1. **从简单开始**: 先实现核心循环，再添加高级特性
2. **分层设计**: 明确每层的职责和接口
3. **适配器隔离**: 不要在核心逻辑中硬编码提供商差异
4. **工具系统**: 使用注册表模式，支持动态加载
5. **错误处理**: 分类错误，智能重试和降级
6. **可观测性**: 从一开始就加入日志和指标
7. **性能优化**: 先正确，再快速
8. **文档完善**: 架构文档和代码注释同样重要

### 9.4 参考资源

- **源码**: https://github.com/NousResearch/hermes-agent
- **文档**: https://hermes-agent.nousresearch.com/docs
- **架构图**: 见 `docs/hermes_agent_architecture.html`

---

**文档结束**

希望这份文档能帮助你理解 Hermes Agent 的架构设计，并为你设计自己的 Agent 系统提供参考！
