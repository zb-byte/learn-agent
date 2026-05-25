#### 3.4.3 model_tools.py

**职责**: 工具编排层，提供公共 API

**核心 API**:
```python
# 1. 获取工具定义
def get_tool_definitions(
    enabled_toolsets: List[str] = None,
    disabled_toolsets: List[str] = None,
    quiet_mode: bool = False,
) -> List[Dict]:
    """
    返回可用工具的 JSON Schema 列表
    
    流程:
    1. 解析工具集 (resolve_toolset)
    2. 过滤禁用的工具
    3. 检查可用性 (check_fn)
    4. 应用动态 schema 覆盖
    5. 包装为 OpenAI function calling 格式
    """
    tools = []
    for name, entry in registry.get_all_tools().items():
        # 检查工具集
        if not _is_tool_enabled(name, enabled_toolsets, disabled_toolsets):
            continue
        
        # 检查可用性
        if entry.check_fn and not _check_fn_cached(entry.check_fn):
            continue
        
        # 应用动态覆盖
        schema = dict(entry.schema)
        if entry.dynamic_schema_overrides:
            overrides = entry.dynamic_schema_overrides()
            schema.update(overrides)
        
        # 包装
        tools.append({
            "type": "function",
            "function": schema
        })
    
    return tools

# 2. 执行工具
def handle_function_call(
    function_name: str,
    function_args: str | Dict,
    task_id: str = None,
    user_task: str = None,
) -> str:
    """
    调度工具执行
    
    流程:
    1. 解析参数 (JSON string → dict)
    2. 调用 registry.dispatch()
    3. 处理异常
    4. 截断过长结果
    5. 返回字符串结果
    """
    # 解析参数
    if isinstance(function_args, str):
        try:
            args = json.loads(function_args)
        except json.JSONDecodeError:
            return tool_error(f"Invalid JSON arguments: {function_args}")
    else:
        args = function_args
    
    # 调度
    try:
        result = registry.dispatch(
            function_name,
            args,
            task_id=task_id,
            user_task=user_task
        )
    except Exception as e:
        logger.exception(f"Tool {function_name} failed")
        return tool_error(f"Tool execution failed: {str(e)}")
    
    # 截断
    entry = registry.get_tool(function_name)
    if entry and entry.max_result_size_chars:
        if len(result) > entry.max_result_size_chars:
            result = result[:entry.max_result_size_chars] + "\n[truncated]"
    
    return result

# 3. 工具集查询
def get_toolset_for_tool(name: str) -> str:
    """返回工具所属的工具集"""
    entry = registry.get_tool(name)
    return entry.toolset if entry else "unknown"

def get_all_tool_names() -> List[str]:
    """返回所有工具名称"""
    return list(registry.get_all_tools().keys())

def check_toolset_requirements() -> Dict[str, Dict]:
    """检查工具集的依赖是否满足"""
    # 返回每个工具集的可用性状态
```

**异步桥接**:
```python
# 持久化事件循环，避免 "Event loop is closed" 错误
_tool_loop = None
_tool_loop_lock = threading.Lock()

def _get_tool_loop():
    """返回持久化的事件循环"""
    global _tool_loop
    with _tool_loop_lock:
        if _tool_loop is None or _tool_loop.is_closed():
            _tool_loop = asyncio.new_event_loop()
        return _tool_loop

def _run_async(coro):
    """
    从同步上下文运行异步协程
    
    策略:
    - 如果当前线程有运行中的循环 → 在新线程中运行
    - 否则 → 使用持久化循环
    """
    try:
        loop = asyncio.get_running_loop()
    except RuntimeError:
        loop = None
    
    if loop and loop.is_running():
        # 在新线程中运行
        import concurrent.futures
        with concurrent.futures.ThreadPoolExecutor(max_workers=1) as pool:
            future = pool.submit(lambda: asyncio.run(coro))
            return future.result(timeout=300)
    else:
        # 使用持久化循环
        loop = _get_tool_loop()
        return loop.run_until_complete(coro)
```

#### 3.4.4 工具实现示例

**tools/file_tools.py**:
```python
from tools.registry import registry, tool_error

# 定义 schema
READ_FILE_SCHEMA = {
    "name": "read_file",
    "description": "Read contents of a file",
    "parameters": {
        "type": "object",
        "properties": {
            "file_path": {
                "type": "string",
                "description": "Path to the file"
            },
            "offset": {
                "type": "integer",
                "description": "Line offset (optional)"
            },
            "limit": {
                "type": "integer",
                "description": "Number of lines to read (optional)"
            }
        },
        "required": ["file_path"]
    }
}

# 定义处理函数
def read_file_handler(args: Dict, **kwargs) -> str:
    """读取文件内容"""
    file_path = args.get("file_path")
    offset = args.get("offset", 0)
    limit = args.get("limit")
    
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            lines = f.readlines()
        
        if offset:
            lines = lines[offset:]
        if limit:
            lines = lines[:limit]
        
        # 添加行号
        numbered = [f"{i+1}\t{line}" for i, line in enumerate(lines)]
        return "".join(numbered)
    
    except FileNotFoundError:
        return tool_error(f"File not found: {file_path}")
    except Exception as e:
        return tool_error(f"Error reading file: {str(e)}")

# 注册工具
registry.register(
    name="read_file",
    toolset="file_operations",
    schema=READ_FILE_SCHEMA,
    handler=read_file_handler,
    description="Read file contents with line numbers",
    emoji="📄",
    max_result_size_chars=50000,
)
```

**tools/terminal_tool.py** (异步工具):
```python
import asyncio
from tools.registry import registry

BASH_SCHEMA = {
    "name": "bash",
    "description": "Execute a bash command",
    "parameters": {
        "type": "object",
        "properties": {
            "command": {
                "type": "string",
                "description": "The command to execute"
            },
            "timeout": {
                "type": "integer",
                "description": "Timeout in seconds (default: 120)"
            }
        },
        "required": ["command"]
    }
}

async def bash_handler(args: Dict, **kwargs) -> str:
    """异步执行 bash 命令"""
    command = args.get("command")
    timeout = args.get("timeout", 120)
    
    try:
        proc = await asyncio.create_subprocess_shell(
            command,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        
        stdout, stderr = await asyncio.wait_for(
            proc.communicate(),
            timeout=timeout
        )
        
        output = stdout.decode() + stderr.decode()
        return output
    
    except asyncio.TimeoutError:
        return tool_error(f"Command timed out after {timeout}s")
    except Exception as e:
        return tool_error(f"Command failed: {str(e)}")

def check_terminal_requirements() -> bool:
    """检查终端是否可用"""
    # 检查环境变量、Docker 等
    return True

registry.register(
    name="bash",
    toolset="terminal",
    schema=BASH_SCHEMA,
    handler=bash_handler,
    check_fn=check_terminal_requirements,
    requires_env=True,
    is_async=True,  # 标记为异步
    description="Execute bash commands",
    emoji="💻",
)
```

#### 3.4.5 toolsets.py

**职责**: 工具集定义和解析

**核心工具列表**:
```python
_HERMES_CORE_TOOLS = [
    # 文件操作
    "read_file", "write_file", "edit_file", "list_directory",
    
    # 终端
    "bash", "python_repl",
    
    # 浏览器
    "browser_navigate", "browser_click", "browser_screenshot",
    
    # 搜索
    "web_search", "grep_search",
    
    # Agent
    "delegate_task", "ask_user",
    
    # 记忆
    "memory_save", "memory_search",
    
    # ... 更多
]
```

**工具集定义**:
```python
TOOLSET_DEFINITIONS = {
    # 基础工具集
    "file_operations": {
        "tools": ["read_file", "write_file", "edit_file", "list_directory"],
        "description": "File reading and writing"
    },
    
    "terminal": {
        "tools": ["bash", "python_repl"],
        "description": "Command execution",
        "requirements": ["terminal_available"]
    },
    
    # 组合工具集
    "development": {
        "includes": ["file_operations", "terminal", "web_search"],
        "description": "Full development toolset"
    },
    
    "research": {
        "includes": ["web_search", "file_operations"],
        "description": "Research and analysis"
    },
    
    # 场景工具集
    "safe": {
        "tools": ["read_file", "web_search", "grep_search"],
        "description": "Read-only safe tools"
    },
}
```

**解析函数**:
```python
def resolve_toolset(name: str) -> List[str]:
    """
    解析工具集为工具列表
    
    支持:
    - 基础工具集: "file_operations" → ["read_file", "write_file", ...]
    - 组合工具集: "development" → 递归解析 includes
    - 单个工具: "read_file" → ["read_file"]
    """
    if name in TOOLSET_DEFINITIONS:
        defn = TOOLSET_DEFINITIONS[name]
        
        # 直接工具列表
        if "tools" in defn:
            return defn["tools"]
        
        # 组合工具集
        if "includes" in defn:
            tools = []
            for included in defn["includes"]:
                tools.extend(resolve_toolset(included))
            return list(set(tools))  # 去重
    
    # 单个工具
    if registry.get_tool(name):
        return [name]
    
    return []

def validate_toolset(name: str) -> Dict[str, Any]:
    """
    验证工具集的依赖
    
    返回:
    {
        "available": bool,
        "missing_requirements": List[str],
        "tools": List[str]
    }
    """
```

---

### 3.5 Layer 5: 记忆与上下文层

#### 3.5.1 memory_manager.py

**职责**: 协调多个记忆提供商

**设计理念**:
- 单一集成点
- 支持多个提供商（但只允许一个外部插件）
- 统一的 prefetch / sync 接口

**核心 API**:
```python
class MemoryManager:
    def __init__(self):
        self._providers: List[MemoryProvider] = []
        self._external_provider_count = 0
    
    def add_provider(self, provider: MemoryProvider):
        """
        添加记忆提供商
        
        限制: 只允许一个外部插件提供商
        """
        if provider.is_external:
            if self._external_provider_count > 0:
                logger.warning("Only one external memory provider allowed")
                return
            self._external_provider_count += 1
        
        self._providers.append(provider)
    
    def build_system_prompt(self) -> str:
        """构建记忆相关的系统提示词"""
        parts = []
        for provider in self._providers:
            prompt = provider.build_system_prompt()
            if prompt:
                parts.append(prompt)
        return "\n\n".join(parts)
    
    def prefetch_all(self, user_message: str) -> str:
        """
        预取记忆上下文
        
        在模型调用前执行，返回相关记忆
        """
        contexts = []
        for provider in self._providers:
            try:
                context = provider.prefetch(user_message)
                if context:
                    contexts.append(context)
            except Exception as e:
                logger.warning(f"Provider {provider} prefetch failed: {e}")
        
        if not contexts:
            return ""
        
        # 包装为系统消息
        combined = "\n\n".join(contexts)
        return f"<memory-context>\n{combined}\n</memory-context>"
    
    def sync_all(self, user_message: str, assistant_response: str):
        """
        同步记忆
        
        在对话完成后执行，保存新记忆
        """
        for provider in self._providers:
            try:
                provider.sync(user_message, assistant_response)
            except Exception as e:
                logger.warning(f"Provider {provider} sync failed: {e}")
    
    def queue_prefetch_all(self, user_message: str):
        """
        异步预取（后台）
        
        用于提前加载下一轮的记忆
        """
        for provider in self._providers:
            if hasattr(provider, 'queue_prefetch'):
                provider.queue_prefetch(user_message)
```

**StreamingContextScrubber**:
```python
class StreamingContextScrubber:
    """
    流式清理记忆上下文标签
    
    问题: <memory-context> 标签可能跨越多个流式块
    解决: 状态机跟踪，缓冲部分标签
    """
    
    def __init__(self):
        self._in_span = False
        self._buf = ""
    
    def feed(self, text: str) -> str:
        """
        处理流式文本块
        
        返回: 可见部分（去除 <memory-context> 内容）
        """
        buf = self._buf + text
        self._buf = ""
        out = []
        
        while buf:
            if self._in_span:
                # 查找关闭标签
                idx = buf.lower().find("</memory-context>")
                if idx == -1:
                    # 未找到，缓冲可能的部分标签
                    held = self._max_partial_suffix(buf, "</memory-context>")
                    self._buf = buf[-held:] if held else ""
                    return "".join(out)
                # 找到，跳过内容
                buf = buf[idx + len("</memory-context>"):]
                self._in_span = False
            else:
                # 查找开启标签
                idx = buf.lower().find("<memory-context>")
                if idx == -1:
                    # 未找到，输出大部分，缓冲可能的部分标签
                    held = self._max_partial_suffix(buf, "<memory-context>")
                    if held:
                        out.append(buf[:-held])
                        self._buf = buf[-held:]
                    else:
                        out.append(buf)
                    return "".join(out)
                # 找到，输出之前的内容
                if idx > 0:
                    out.append(buf[:idx])
                buf = buf[idx + len("<memory-context>"):]
                self._in_span = True
        
        return "".join(out)
    
    def flush(self) -> str:
        """流结束时刷新缓冲"""
        return self._buf if not self._in_span else ""
```
