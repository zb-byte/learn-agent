# Hermes Agent 作为 SDK 的可用性分析

## 问题

Hermes Agent 作为一个 SDK，Agent Worker 是否可以被外部系统直接使用？

## 答案

**是的，Hermes Agent 可以作为 SDK 被外部系统直接使用，包括 KANBAN Worker。**

Hermes Agent 设计为一个**可编程的 Python SDK**，提供了多层次的集成方式。

## 详细分析

### 1. Hermes Agent 的定位

根据 `pyproject.toml` 和 `README.md`：

```toml
[project]
name = "hermes-agent"
version = "0.14.0"
description = "The self-improving AI agent — creates skills from experience, improves them during use, and runs anywhere"
```

**核心特点**:
- ✅ **Python 包**: 可通过 `pip install hermes-agent` 安装
- ✅ **MIT 许可证**: 开源，可商用
- ✅ **模块化设计**: 核心组件可独立使用
- ✅ **多种集成方式**: CLI、SDK、Gateway、批处理

### 2. SDK 使用方式

#### 2.1 直接导入 AIAgent 类

**最核心的 SDK 接口**: `run_agent.AIAgent`

```python
from run_agent import AIAgent

# 创建 Agent 实例
agent = AIAgent(
    model="anthropic:claude-opus-4",
    api_key="your-api-key",
    max_iterations=10,
    enabled_toolsets=["file", "terminal", "web"],
    verbose_logging=True,
)

# 运行对话
result = agent.run_conversation("帮我分析这个项目的代码结构")

# 获取结果
print(result["messages"])
print(result["completed"])
```

**已有使用案例**:
1. **batch_runner.py**: 批量轨迹生成
2. **cli.py**: CLI 交互界面
3. **tools/delegate_tool.py**: 子 Agent 委托
4. **hermes_cli/oneshot.py**: 一次性查询

#### 2.2 KANBAN Worker 的 SDK 使用

**KANBAN Worker 本质上就是通过 SDK 启动的 Agent**:

```python
# hermes_cli/kanban_db.py::_default_spawn()
# 启动命令
cmd = ["hermes", "-p", profile, "chat", "-q", "work kanban task {task.id}"]

# 等价于 SDK 调用
from run_agent import AIAgent

agent = AIAgent(
    model=profile_config["model"],
    enabled_toolsets=profile_config["toolsets"] + ["kanban"],
    ephemeral_system_prompt=None,  # 使用默认系统提示词
    # ... 其他配置
)

# 设置环境变量
os.environ["HERMES_KANBAN_TASK"] = task.id
os.environ["HERMES_KANBAN_WORKSPACE"] = workspace

# 运行
result = agent.run_conversation(f"work kanban task {task.id}")
```

### 3. 外部系统集成方式

#### 方式 1: 直接使用 AIAgent (推荐)

**适用场景**: 需要完全控制 Agent 行为的外部系统

```python
# 外部系统代码
from run_agent import AIAgent

class MyCustomSystem:
    def __init__(self):
        self.agent = AIAgent(
            model="openai:gpt-4",
            api_key=os.environ["OPENAI_API_KEY"],
            enabled_toolsets=["file", "terminal"],
            max_iterations=20,
            save_trajectories=True,
        )
    
    def process_task(self, task_description: str):
        """处理任务"""
        result = self.agent.run_conversation(task_description)
        return {
            "success": result["completed"],
            "messages": result["messages"],
            "api_calls": result["api_calls"],
        }
    
    def process_with_context(self, task: str, context_files: list):
        """带上下文处理"""
        # 可以传递上下文文件
        self.agent.context_files = context_files
        return self.agent.run_conversation(task)
```

**优点**:
- ✅ 完全控制 Agent 配置
- ✅ 可以自定义工具集
- ✅ 可以注入自定义系统提示词
- ✅ 可以访问完整的对话历史

#### 方式 2: 使用 KANBAN 系统

**适用场景**: 需要任务队列、并发控制、重试机制的系统

```python
# 外部系统代码
from hermes_cli import kanban_db as kb

class MyTaskSystem:
    def __init__(self):
        self.conn = kb.connect()
    
    def submit_task(self, title: str, body: str, assignee: str):
        """提交任务到 KANBAN"""
        task_id = kb.create_task(
            self.conn,
            title=title,
            body=body,
            assignee=assignee,  # 指定 Profile
            tenant="my-system",
            priority=10,
        )
        return task_id
    
    def wait_for_completion(self, task_id: str, timeout: int = 3600):
        """等待任务完成"""
        import time
        start = time.time()
        while time.time() - start < timeout:
            task = kb.get_task(self.conn, task_id)
            if task.status == "done":
                # 获取结果
                runs = kb.list_runs(self.conn, task_id)
                latest = runs[-1] if runs else None
                return {
                    "success": True,
                    "summary": latest.summary if latest else None,
                    "metadata": latest.metadata if latest else None,
                }
            elif task.status == "blocked":
                return {"success": False, "reason": "blocked"}
            time.sleep(5)
        return {"success": False, "reason": "timeout"}
```

**优点**:
- ✅ 内置任务队列和调度
- ✅ 自动重试和错误处理
- ✅ 并发控制（`max_spawn`）
- ✅ 任务依赖管理
- ✅ 跨任务协作（通过 comments）

#### 方式 3: 通过 CLI 调用

**适用场景**: 简单集成，不需要深度定制

```python
import subprocess
import json

class SimpleCLIIntegration:
    def run_task(self, prompt: str):
        """通过 CLI 运行任务"""
        result = subprocess.run(
            ["hermes", "chat", "-q", prompt],
            capture_output=True,
            text=True,
        )
        return result.stdout
    
    def run_kanban_task(self, task_id: str, profile: str):
        """运行 KANBAN 任务"""
        env = os.environ.copy()
        env["HERMES_KANBAN_TASK"] = task_id
        
        result = subprocess.run(
            ["hermes", "-p", profile, "chat", "-q", f"work kanban task {task_id}"],
            env=env,
            capture_output=True,
            text=True,
        )
        return result.returncode == 0
```

**优点**:
- ✅ 简单易用
- ✅ 不需要导入 Python 模块
- ✅ 进程隔离

**缺点**:
- ❌ 无法访问内部状态
- ❌ 通信开销较大

#### 方式 4: 通过 Gateway API

**适用场景**: 需要 HTTP API 的外部系统

```python
import requests

class GatewayAPIIntegration:
    def __init__(self, gateway_url: str, api_key: str):
        self.gateway_url = gateway_url
        self.api_key = api_key
    
    def send_message(self, user_id: str, message: str):
        """通过 Gateway 发送消息"""
        response = requests.post(
            f"{self.gateway_url}/api/message",
            json={
                "user_id": user_id,
                "message": message,
            },
            headers={"Authorization": f"Bearer {self.api_key}"},
        )
        return response.json()
```

**优点**:
- ✅ 跨语言支持
- ✅ 网络隔离
- ✅ 支持多平台（Telegram、Discord 等）

### 4. KANBAN Worker 的外部使用

#### 4.1 作为任务执行引擎

**场景**: 外部系统需要一个可靠的 AI 任务执行引擎

```python
# 外部系统集成示例
from hermes_cli import kanban_db as kb

class AITaskEngine:
    """基于 KANBAN 的 AI 任务执行引擎"""
    
    def __init__(self, board: str = "default"):
        self.board = board
        self.conn = kb.connect(board=board)
    
    def create_analysis_task(self, data_path: str, analysis_type: str):
        """创建数据分析任务"""
        task_id = kb.create_task(
            self.conn,
            title=f"分析 {data_path}",
            body=f"""
            请分析以下数据文件：{data_path}
            分析类型：{analysis_type}
            
            要求：
            1. 读取数据文件
            2. 进行统计分析
            3. 生成可视化图表
            4. 输出分析报告
            """,
            assignee="data-analyst",  # 指定专门的分析 Profile
            tenant="external-system",
            workspace_kind="dir",
            workspace_path="/data/workspace",
        )
        return task_id
    
    def create_code_review_task(self, pr_url: str):
        """创建代码审查任务"""
        task_id = kb.create_task(
            self.conn,
            title=f"审查 PR: {pr_url}",
            body=f"""
            请审查以下 Pull Request：{pr_url}
            
            审查要点：
            1. 代码质量
            2. 安全性
            3. 性能影响
            4. 测试覆盖率
            """,
            assignee="code-reviewer",
            tenant="external-system",
            skills=["github-code-review"],  # 强制加载特定 skill
        )
        return task_id
    
    def create_pipeline(self, tasks: list):
        """创建任务流水线"""
        task_ids = []
        parent_id = None
        
        for task_spec in tasks:
            task_id = kb.create_task(
                self.conn,
                title=task_spec["title"],
                body=task_spec["body"],
                assignee=task_spec["assignee"],
                parents=[parent_id] if parent_id else [],
                tenant="external-system",
            )
            task_ids.append(task_id)
            parent_id = task_id
        
        return task_ids
    
    def get_task_result(self, task_id: str):
        """获取任务结果"""
        task = kb.get_task(self.conn, task_id)
        if task.status != "done":
            return None
        
        runs = kb.list_runs(self.conn, task_id)
        latest = runs[-1] if runs else None
        
        return {
            "summary": latest.summary if latest else None,
            "metadata": latest.metadata if latest else None,
            "result": task.result,
        }
```

#### 4.2 作为 Swarm 编排引擎

**场景**: 需要多 Agent 协作的复杂任务

```python
from hermes_cli import kanban_swarm as ks

class SwarmOrchestrator:
    """基于 KANBAN Swarm 的多 Agent 编排"""
    
    def __init__(self):
        self.conn = kb.connect()
    
    def create_research_swarm(self, topic: str):
        """创建研究 Swarm"""
        swarm = ks.create_swarm(
            self.conn,
            goal=f"深入研究主题：{topic}",
            workers=[
                ks.SwarmWorkerSpec(
                    profile="researcher-web",
                    title="网络资料收集",
                    body=f"收集关于 {topic} 的最新网络资料",
                    skills=["web-research"],
                ),
                ks.SwarmWorkerSpec(
                    profile="researcher-paper",
                    title="学术论文检索",
                    body=f"检索关于 {topic} 的学术论文",
                    skills=["academic-search"],
                ),
                ks.SwarmWorkerSpec(
                    profile="researcher-code",
                    title="代码示例收集",
                    body=f"收集关于 {topic} 的代码示例",
                    skills=["github-search"],
                ),
            ],
            verifier_assignee="reviewer",
            synthesizer_assignee="writer",
            tenant="external-system",
        )
        
        return swarm.root_id
    
    def monitor_swarm(self, root_id: str):
        """监控 Swarm 进度"""
        blackboard = ks.latest_blackboard(self.conn, root_id)
        topology = blackboard.get("topology", {})
        
        worker_ids = topology.get("worker_ids", [])
        verifier_id = topology.get("verifier_id")
        synthesizer_id = topology.get("synthesizer_id")
        
        status = {}
        for wid in worker_ids:
            task = kb.get_task(self.conn, wid)
            status[wid] = task.status
        
        return {
            "workers": status,
            "verifier": kb.get_task(self.conn, verifier_id).status if verifier_id else None,
            "synthesizer": kb.get_task(self.conn, synthesizer_id).status if synthesizer_id else None,
        }
```

### 5. SDK 的核心组件

#### 5.1 可独立使用的模块

| 模块 | 功能 | 外部可用性 |
|------|------|-----------|
| `run_agent.AIAgent` | 核心 Agent 类 | ✅ 完全可用 |
| `model_tools` | 工具系统 | ✅ 可用 |
| `tools.registry` | 工具注册表 | ✅ 可用 |
| `agent.agent_init` | Agent 初始化 | ✅ 可用 |
| `agent.conversation_loop` | 对话循环 | ✅ 可用 |
| `agent.prompt_builder` | 提示词构建 | ✅ 可用 |
| `agent.memory_manager` | 记忆管理 | ✅ 可用 |
| `hermes_cli.kanban_db` | KANBAN 数据库 | ✅ 完全可用 |
| `hermes_cli.kanban_swarm` | Swarm 编排 | ✅ 完全可用 |

#### 5.2 工具扩展

**外部系统可以注册自定义工具**:

```python
from tools.registry import registry

# 注册自定义工具
registry.register(
    name="my_custom_tool",
    toolset="external",
    schema={
        "name": "my_custom_tool",
        "description": "我的自定义工具",
        "parameters": {
            "type": "object",
            "properties": {
                "input": {"type": "string", "description": "输入参数"}
            },
            "required": ["input"],
        },
    },
    handler=my_custom_handler,
    check_fn=lambda: True,  # 总是可用
)

def my_custom_handler(args: dict, **kwargs) -> str:
    """自定义工具处理函数"""
    input_data = args.get("input")
    # 处理逻辑
    result = process_data(input_data)
    return json.dumps({"result": result})
```

### 6. 实际使用案例

#### 案例 1: 批量轨迹生成 (batch_runner.py)

```python
# batch_runner.py 的核心逻辑
from run_agent import AIAgent

def process_single_prompt(prompt, config):
    agent = AIAgent(
        model=config["model"],
        max_iterations=config["max_iterations"],
        enabled_toolsets=selected_toolsets,
        save_trajectories=False,
        skip_context_files=True,
        skip_memory=True,
    )
    
    result = agent.run_conversation(prompt, task_id=task_id)
    trajectory = agent._convert_to_trajectory_format(
        result["messages"],
        prompt,
        result["completed"]
    )
    
    return trajectory
```

**用途**: 生成训练数据，用于训练下一代工具调用模型

#### 案例 2: 子 Agent 委托 (delegate_tool.py)

```python
# tools/delegate_tool.py 的核心逻辑
from run_agent import AIAgent

def spawn_subagent(task_description, parent_agent):
    subagent = AIAgent(
        model=parent_agent.model,
        enabled_toolsets=parent_agent.enabled_toolsets,
        max_iterations=10,
        # 继承父 Agent 的配置
    )
    
    result = subagent.run_conversation(task_description)
    return result
```

**用途**: 并行处理多个子任务

#### 案例 3: 一次性查询 (oneshot.py)

```python
# hermes_cli/oneshot.py 的核心逻辑
from run_agent import AIAgent

def oneshot_query(prompt, model, toolsets):
    agent = AIAgent(
        model=model,
        enabled_toolsets=toolsets,
        max_iterations=5,
        save_trajectories=False,
    )
    
    result = agent.run_conversation(prompt)
    return result["messages"][-1]["content"]
```

**用途**: 快速查询，不保存历史

### 7. 外部系统集成的最佳实践

#### 7.1 配置管理

```python
class HermesAgentWrapper:
    """Hermes Agent 的外部系统包装器"""
    
    def __init__(self, config_path: str):
        self.config = self._load_config(config_path)
        self.agent = None
    
    def _load_config(self, path: str):
        """加载配置"""
        import yaml
        with open(path) as f:
            return yaml.safe_load(f)
    
    def initialize_agent(self):
        """初始化 Agent"""
        from run_agent import AIAgent
        
        self.agent = AIAgent(
            model=self.config["model"],
            api_key=self.config.get("api_key"),
            enabled_toolsets=self.config.get("toolsets", []),
            max_iterations=self.config.get("max_iterations", 10),
            verbose_logging=self.config.get("verbose", False),
        )
    
    def execute_task(self, task: dict):
        """执行任务"""
        if not self.agent:
            self.initialize_agent()
        
        result = self.agent.run_conversation(task["prompt"])
        return {
            "task_id": task["id"],
            "success": result["completed"],
            "output": result["messages"][-1]["content"],
            "api_calls": result["api_calls"],
        }
```

#### 7.2 错误处理

```python
class RobustAgentRunner:
    """带错误处理的 Agent 运行器"""
    
    def run_with_retry(self, prompt: str, max_retries: int = 3):
        """带重试的运行"""
        from run_agent import AIAgent
        
        for attempt in range(max_retries):
            try:
                agent = AIAgent(
                    model="anthropic:claude-opus-4",
                    max_iterations=10,
                )
                
                result = agent.run_conversation(prompt)
                return result
                
            except Exception as e:
                if attempt == max_retries - 1:
                    raise
                print(f"Attempt {attempt + 1} failed: {e}, retrying...")
                time.sleep(2 ** attempt)  # 指数退避
```

#### 7.3 资源管理

```python
class ManagedAgentPool:
    """Agent 池管理"""
    
    def __init__(self, pool_size: int = 4):
        self.pool_size = pool_size
        self.agents = []
        self.lock = threading.Lock()
    
    def get_agent(self):
        """获取可用 Agent"""
        with self.lock:
            if len(self.agents) < self.pool_size:
                agent = AIAgent(
                    model="anthropic:claude-opus-4",
                    max_iterations=10,
                )
                self.agents.append(agent)
                return agent
            
            # 等待可用 Agent
            # 实现池化逻辑
            pass
    
    def release_agent(self, agent):
        """释放 Agent"""
        # 清理状态
        agent.messages = []
```

### 8. 限制和注意事项

#### 8.1 当前限制

| 限制 | 说明 | 解决方案 |
|------|------|---------|
| 进程隔离 | KANBAN Worker 是独立进程 | 使用 KANBAN API 通信 |
| 状态共享 | Agent 之间不共享内存 | 通过数据库或文件系统共享 |
| 并发控制 | 需要手动管理并发 | 使用 `max_spawn` 配置 |
| 工具注册 | 需要在启动前注册 | 使用插件系统 |

#### 8.2 性能考虑

```python
# 不推荐：频繁创建 Agent
for task in tasks:
    agent = AIAgent(...)  # 每次都创建新实例
    agent.run_conversation(task)

# 推荐：复用 Agent 实例
agent = AIAgent(...)
for task in tasks:
    agent.messages = []  # 清空历史
    agent.run_conversation(task)
```

#### 8.3 安全考虑

```python
# 限制工具集
agent = AIAgent(
    enabled_toolsets=["file", "web"],  # 不包含 terminal
    # 防止执行危险命令
)

# 限制文件访问
agent = AIAgent(
    enabled_toolsets=["file"],
    # 配合 HERMES.md 限制访问路径
)
```

### 9. 总结

#### 9.1 Hermes Agent 作为 SDK 的优势

| 优势 | 说明 |
|------|------|
| ✅ **完全开源** | MIT 许可证，可商用 |
| ✅ **模块化设计** | 核心组件可独立使用 |
| ✅ **多层次集成** | SDK、CLI、Gateway、KANBAN |
| ✅ **工具扩展性** | 可注册自定义工具 |
| ✅ **任务编排** | KANBAN + Swarm 支持复杂工作流 |
| ✅ **生产就绪** | 内置重试、错误处理、并发控制 |

#### 9.2 KANBAN Worker 的外部可用性

**完全可用**，外部系统可以：

1. ✅ 通过 `kanban_db` API 创建和管理任务
2. ✅ 通过 `kanban_swarm` API 创建多 Agent 协作
3. ✅ 通过环境变量控制 Worker 行为
4. ✅ 通过 Profile 配置定制 Worker 能力
5. ✅ 通过 Skills 扩展 Worker 功能
6. ✅ 通过 Comments 实现跨任务通信
7. ✅ 通过 Metadata 传递结构化数据

#### 9.3 推荐集成方式

| 场景 | 推荐方式 | 理由 |
|------|---------|------|
| 简单查询 | 直接使用 `AIAgent` | 最简单，完全控制 |
| 任务队列 | 使用 KANBAN | 内置调度和重试 |
| 多 Agent 协作 | 使用 KANBAN Swarm | 内置编排和同步 |
| 跨语言集成 | 使用 Gateway API | HTTP 接口，语言无关 |
| 批量处理 | 参考 `batch_runner.py` | 已有成熟实现 |

## 结论

**Hermes Agent 不仅可以作为 SDK 被外部系统使用，而且设计上就是为此优化的。**

KANBAN Worker 是 Hermes Agent SDK 的一个高级应用，外部系统可以：
- 直接使用 `AIAgent` 类进行简单集成
- 使用 KANBAN 系统进行复杂任务编排
- 使用 Swarm 系统进行多 Agent 协作
- 通过插件系统扩展功能

这使得 Hermes Agent 成为一个**真正的 AI Agent 开发框架**，而不仅仅是一个独立应用。
