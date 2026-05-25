# Hermes Agent 架构设计文档

> **目标读者**: 希望设计和实现类似 Agent 系统的开发者  
> **文档版本**: v1.0  
> **基于代码版本**: Hermes Agent v0.14.0  
> **最后更新**: 2026-05-25

---

## 目录

1. [系统概述](#1-系统概述)
2. [整体架构](#2-整体架构)
3. [核心模块详解](#3-核心模块详解)
4. [关键设计模式](#4-关键设计模式)
5. [数据流与生命周期](#5-数据流与生命周期)
6. [扩展性设计](#6-扩展性设计)
7. [性能优化策略](#7-性能优化策略)
8. [设计建议](#8-设计建议)

---

## 1. 系统概述

### 1.1 Hermes Agent 是什么

Hermes Agent 是一个**生产级的 AI Agent 框架**，支持：
- 多模型提供商（Anthropic, OpenAI, Google Gemini, AWS Bedrock 等）
- 80+ 内置工具（文件操作、终端执行、浏览器自动化、子 Agent 等）
- 多种用户界面（CLI、TUI、消息网关）
- 自动上下文管理（压缩、缓存、记忆）
- 企业级特性（凭证池、速率限制、错误恢复）

### 1.2 核心特性

| 特性 | 说明 |
|------|------|
| **模型无关** | 统一接口支持多个 LLM 提供商 |
| **工具系统** | 自注册、动态加载、并行执行 |
| **上下文管理** | 自动压缩、提示词缓存、记忆系统 |
| **多界面** | CLI、TUI、Telegram、Discord、Slack 等 |
| **可扩展** | 插件系统、自定义工具、技能系统 |
| **生产就绪** | 错误恢复、速率限制、日志、监控 |

---

## 2. 整体架构

### 2.1 分层架构图

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: 用户接口层 (User Interface Layer)                  │
│  ┌──────────┐  ┌──────────┐  ┌─────────────────────────┐   │
│  │   CLI    │  │   TUI    │  │  Gateway (多平台消息)    │   │
│  └──────────┘  └──────────┘  └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: 核心 Agent 层 (Core Agent Layer)                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  AIAgent (run_agent.py)                              │   │
│  │  • agent_init.py (初始化)                            │   │
│  │  • conversation_loop.py (对话循环)                   │   │
│  │  • agent_runtime_helpers.py (运行时辅助)             │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 模型适配层 (Model Adapter Layer)                   │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │ Anthropic    │ │   Gemini     │ │   Bedrock    │        │
│  │   Adapter    │ │   Adapter    │ │   Adapter    │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 4: 工具系统层 (Tool System Layer)                     │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │ model_tools  │ │   registry   │ │  tools/*.py  │        │
│  │     .py      │ │     .py      │ │  (80+ 工具)  │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 5: 记忆与上下文层 (Memory & Context Layer)            │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │   memory_    │ │   context_   │ │   prompt_    │        │
│  │  manager.py  │ │ compressor   │ │  builder.py  │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 6: 支持系统层 (Support Systems Layer)                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │ credential_  │ │    error_    │ │   retry_     │        │
│  │   pool.py    │ │ classifier   │ │   utils.py   │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 7: 数据持久化层 (Data Persistence Layer)              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │  hermes_     │ │ trajectory_  │ │ checkpoint_  │        │
│  │  state.py    │ │ compressor   │ │  manager.py  │        │
│  │  (SQLite)    │ │     .py      │ │              │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 架构特点

1. **分层解耦**: 每层职责清晰，上层依赖下层，下层不依赖上层
2. **适配器模式**: 模型适配层隔离不同 LLM 提供商的差异
3. **插件化**: 工具、记忆提供商、上下文引擎都支持插件扩展
4. **异步友好**: 支持同步和异步工具，自动桥接
5. **可观测性**: 完整的日志、指标、追踪支持

---

## 3. 核心模块详解

### 3.1 Layer 1: 用户接口层

#### 3.1.1 CLI (cli.py)

**职责**: 提供交互式命令行界面

**核心组件**:
```python
class HermesCLI:
    def __init__(self):
        self.agent = None
        self.session = Session()  # prompt_toolkit
        self.console = Console()  # Rich
        self.spinner = KawaiiSpinner()
        
    def run(self):
        """主循环: 读取输入 → 处理命令 → 显示结果"""
        while True:
            user_input = self.session.prompt("> ")
            if user_input.startswith("/"):
                self.process_command(user_input)
            else:
                self.send_message(user_input)
```

**关键功能**:
- **命令系统**: 中央注册表 (`hermes_cli/commands.py`)，支持别名、自动补全
- **皮肤引擎**: 数据驱动的主题系统 (`hermes_cli/skin_engine.py`)
- **动画显示**: `KawaiiSpinner` 提供可爱的加载动画
- **会话管理**: 持久化会话历史，支持搜索和恢复

**设计亮点**:
```python
# 命令注册表 - 单一数据源
COMMAND_REGISTRY = [
    CommandDef("background", "Run in background", "Session", 
               aliases=("bg",), args_hint="<prompt>"),
    CommandDef("model", "Switch model", "Configuration",
               aliases=("m",), args_hint="<model_name>"),
    # ... 更多命令
]

# 自动生成: 帮助文本、自动补全、网关路由
```

#### 3.1.2 TUI (ui-tui/)

**职责**: 基于 Ink (React for terminal) 的现代终端 UI

**技术栈**:
- **前端**: TypeScript + Ink (React)
- **后端**: Python JSON-RPC 服务 (`tui_gateway/`)
- **通信**: WebSocket 或 stdio

**架构**:
```
ui-tui/src/
├── entry.tsx          # 入口点
├── app.tsx            # 主应用组件
├── gatewayClient.ts   # JSON-RPC 客户端
├── components/        # UI 组件
│   ├── ChatView.tsx
│   ├── ToolOutput.tsx
│   └── StatusBar.tsx
└── hooks/             # React hooks
    └── useAgent.ts
```

**优势**:
- 实时流式输出
- 丰富的交互体验
- 组件化、可测试

#### 3.1.3 Gateway (gateway/)

**职责**: 多平台消息网关，统一接口对接各种消息平台

**支持平台** (20+):
- Telegram, Discord, Slack, WhatsApp
- WeChat, DingTalk, Feishu, QQ
- Matrix, Signal, Mattermost
- Email, SMS, Webhook

**核心设计**:
```python
# gateway/run.py
class HermesGateway:
    def __init__(self):
        self.platforms = {}  # 平台适配器
        self.sessions = {}   # 会话管理
        
    async def handle_message(self, event: MessageEvent):
        """统一消息处理流程"""
        # 1. 解析平台特定消息
        # 2. 创建或恢复会话
        # 3. 调用 AIAgent
        # 4. 格式化响应
        # 5. 发送回平台
```

**平台适配器模式**:
```python
# gateway/platforms/telegram.py
class TelegramAdapter(PlatformAdapter):
    async def receive_message(self) -> MessageEvent:
        """接收消息"""
        
    async def send_message(self, text: str):
        """发送消息"""
        
    async def send_typing(self):
        """发送输入中状态"""
```

---

### 3.2 Layer 2: 核心 Agent 层

#### 3.2.1 AIAgent 类 (run_agent.py)

**职责**: Agent 的核心类，管理整个对话生命周期

**初始化** (60+ 参数):
```python
class AIAgent:
    def __init__(
        self,
        # 模型配置
        base_url: str = None,
        api_key: str = None,
        provider: str = None,
        model: str = "",
        
        # 工具配置
        enabled_toolsets: List[str] = None,
        disabled_toolsets: List[str] = None,
        
        # 行为配置
        max_iterations: int = 90,
        quiet_mode: bool = False,
        
        # 会话配置
        session_id: str = None,
        platform: str = None,
        
        # 高级配置
        credential_pool = None,
        iteration_budget = None,
        fallback_model: Dict = None,
        
        # 回调
        tool_progress_callback = None,
        stream_delta_callback = None,
        # ... 更多回调
    ):
        # 初始化逻辑在 agent_init.py
        from agent.agent_init import init_agent
        init_agent(self, **locals())
```

**核心方法**:
```python
def run_conversation(
    self,
    user_message: str,
    system_message: str = None,
    conversation_history: List[Dict] = None,
    task_id: str = None,
) -> Dict[str, Any]:
    """
    运行完整对话循环
    
    返回:
    {
        'completed': bool,
        'final_response': str,
        'messages': List[Dict],
        'api_calls': int,
        'usage': Dict,
    }
    """
```

**简化接口**:
```python
def chat(self, message: str) -> str:
    """简单接口，直接返回响应文本"""
    result = self.run_conversation(message)
    return result['final_response']
```
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
