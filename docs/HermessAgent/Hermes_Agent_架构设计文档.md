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
