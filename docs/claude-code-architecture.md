# Claude Code Agent Runtime 架构图

> 基于源码整理的架构图，展示核心运行时组件及其关系

## 整体架构

```mermaid
flowchart TB
    subgraph Entry["入口层 Entry Points"]
        CLI[cli.tsx<br/>入口 + 运行时 polyfill]
        MAIN[main.tsx<br/>Commander CLI 定义]
        INIT[init.ts<br/>初始化]
    end

    subgraph UI["UI 层 (React/Ink)"]
        REPL[REPL.tsx<br/>交互式 REPL 屏幕]
        MSG[Messages.tsx<br/>消息渲染]
        INPUT[PromptInput/<br/>用户输入]
        PERM[permissions/<br/>工具权限 UI]
    end

    subgraph Core["核心运行时 Core Runtime"]
        QE[QueryEngine.ts<br/>会话编排器]
        QUERY[query.ts<br/>queryLoop 状态机]
        STATE[state/AppState.tsx<br/>应用状态管理]
        BOOTSTRAP[bootstrap/state.ts<br/>全局单例状态]
    end

    subgraph API["API 层"]
        CLAUDE_API[services/api/claude.ts<br/>Claude API 客户端]
        PROVIDERS[utils/model/providers.ts<br/>多提供商支持]
        STREAM[流式响应处理]
    end

    subgraph Tools["工具系统 Tool System"]
        TOOL_DEF[Tool.ts<br/>工具接口定义]
        TOOL_REG[tools.ts<br/>工具注册表]
        TOOL_EXEC[services/tools/<br/>工具执行编排]
        TOOLS_DIR[tools/<br/>具体工具实现]
    end

    subgraph Context["上下文管理 Context Management"]
        CTX[context.ts<br/>上下文构建]
        SYSPROMPT[utils/queryContext.ts<br/>系统提示词]
        COMPACT[services/compact/<br/>压缩服务]
        FILESTATE[utils/fileStateCache.ts<br/>文件状态缓存]
    end

    subgraph Memory["记忆与持久化 Memory & Persistence"]
        TRANSCRIPT[utils/sessionStorage.ts<br/>transcript 记录]
        HISTORY[history.ts<br/>历史管理]
        MEMDIR[memdir/<br/>memory 目录]
        CLAUDEMD[utils/claudemd.ts<br/>CLAUDE.md 加载]
    end

    subgraph Extension["扩展机制 Extensions"]
        SKILLS[skills/<br/>Skill 系统]
        PLUGINS[plugins/<br/>Plugin 加载器]
        MCP[services/mcp/<br/>MCP 协议]
        AGENT[tools/AgentTool/<br/>Multi-Agent]
    end

    subgraph Security["安全层 Security"]
        HOOKS[hooks/<br/>Hook 系统]
        PERMS[权限判断]
        SANDBOX[沙箱隔离]
    end

    CLI --> MAIN
    MAIN --> INIT
    MAIN --> REPL
    
    REPL --> QE
    REPL --> MSG
    REPL --> INPUT
    REPL --> PERM
    
    QE --> QUERY
    QE --> STATE
    QE --> TRANSCRIPT
    
    QUERY --> CLAUDE_API
    QUERY --> TOOL_EXEC
    QUERY --> COMPACT
    QUERY --> HOOKS
    
    CLAUDE_API --> PROVIDERS
    CLAUDE_API --> STREAM
    
    TOOL_EXEC --> TOOL_DEF
    TOOL_EXEC --> TOOL_REG
    TOOL_EXEC --> TOOLS_DIR
    
    QUERY --> CTX
    CTX --> SYSPROMPT
    CTX --> FILESTATE
    CTX --> CLAUDEMD
    
    QE --> HISTORY
    QE --> MEMDIR
    
    TOOL_REG --> SKILLS
    TOOL_REG --> MCP
    TOOL_REG --> AGENT
    
    TOOL_EXEC --> PERMS
    TOOL_EXEC --> SANDBOX
    
    STATE --> BOOTSTRAP

    style Core fill:#e1f5ff
    style Tools fill:#fff4e1
    style Context fill:#f0e1ff
    style Memory fill:#e1ffe1
    style Extension fill:#ffe1f0
    style Security fill:#ffe1e1
```

## 核心数据流

```mermaid
sequenceDiagram
    participant User as 用户
    participant REPL as REPL Screen
    participant QE as QueryEngine
    participant Query as query()
    participant API as Claude API
    participant Tools as Tool Runtime
    participant State as AppState

    User->>REPL: 输入消息
    REPL->>QE: submitMessage()
    QE->>State: 更新 messages
    QE->>Query: query(messages, context)
    
    loop queryLoop 状态机
        Query->>Query: 整理上下文
        Query->>API: 流式请求
        API-->>Query: 返回响应
        
        alt 有 tool_use
            Query->>Tools: runTools()
            Tools->>Tools: 权限判断
            Tools->>Tools: 执行工具
            Tools-->>Query: tool_result
            Query->>Query: 回填结果
            Query->>Query: continue (next_turn)
        else 无 tool_use
            Query->>Query: 检查截断/压缩/hook
            alt 需要恢复
                Query->>Query: 恢复策略
                Query->>Query: continue (recovery)
            else 完成
                Query->>Query: terminal (completed)
            end
        end
    end
    
    Query-->>QE: 最终结果
    QE->>State: 更新状态
    QE->>REPL: 渲染输出
    REPL-->>User: 显示结果
```

## 关键模块说明

### 1. 入口层 (Entry Points)

| 文件 | 职责 |
|------|------|
| `cli.tsx` | 真正入口，注入运行时 polyfill（feature()、MACRO、全局变量） |
| `main.tsx` | Commander.js CLI 定义，解析参数，初始化服务 |
| `init.ts` | 一次性初始化（遥测、配置、信任对话框） |

### 2. 核心运行时 (Core Runtime)

| 文件 | 职责 |
|------|------|
| `QueryEngine.ts` | 会话级编排器，管理 conversation 状态、压缩、文件历史、归因 |
| `query.ts` | 主 API 查询函数，实现 queryLoop 状态机 |
| `state/AppState.tsx` | 中央应用状态（messages、tools、permissions、MCP 连接） |
| `bootstrap/state.ts` | 模块级单例（session ID、CWD、token 计数） |

### 3. UI 层 (React/Ink)

| 目录/文件 | 职责 |
|------|------|
| `screens/REPL.tsx` | 交互式 REPL 屏幕，处理用户输入和消息显示 |
| `components/Messages.tsx` | 对话消息渲染 |
| `components/PromptInput/` | 用户输入处理 |
| `components/permissions/` | 工具权限审批 UI |
| `ink/` | 自定义 Ink 框架（forked） |

### 4. API 层

| 文件 | 职责 |
|------|------|
| `services/api/claude.ts` | 核心 API 客户端，构建请求参数，调用流式端点 |
| `utils/model/providers.ts` | 多提供商支持（Anthropic、AWS Bedrock、Google Vertex、Azure） |

### 5. 工具系统 (Tool System)

| 文件/目录 | 职责 |
|------|------|
| `Tool.ts` | 工具接口定义（name、description、inputSchema、call） |
| `tools.ts` | 工具注册表，组装工具列表 |
| `services/tools/toolOrchestration.ts` | 工具执行编排（runTools） |
| `services/tools/StreamingToolExecutor.ts` | 流式工具执行器 |
| `tools/<ToolName>/` | 具体工具实现（BashTool、FileEditTool、GrepTool、AgentTool 等） |

### 6. 上下文管理 (Context Management)

| 文件/目录 | 职责 |
|------|------|
| `context.ts` | 构建 system/user context（git status、日期、CLAUDE.md、memory） |
| `utils/queryContext.ts` | 获取系统提示词部分 |
| `services/compact/` | 压缩服务（autoCompact、reactiveCompact、microcompact） |
| `utils/fileStateCache.ts` | 文件状态缓存（模型读过哪些文件） |
| `utils/api.ts` | prependUserContext、appendSystemContext |

### 7. 记忆与持久化 (Memory & Persistence)

| 文件/目录 | 职责 |
|------|------|
| `utils/sessionStorage.ts` | transcript 记录和恢复 |
| `history.ts` | 历史管理 |
| `memdir/` | memory 目录管理 |
| `utils/claudemd.ts` | 发现和加载 CLAUDE.md 文件 |

### 8. 扩展机制 (Extensions)

| 目录 | 职责 |
|------|------|
| `skills/` | Skill 系统（按需加载专业指令） |
| `plugins/` | Plugin 加载器（外部能力包） |
| `services/mcp/` | MCP 协议实现（外部工具接入） |
| `tools/AgentTool/` | Multi-Agent 支持（子 Agent 派生） |

### 9. 安全层 (Security)

| 目录 | 职责 |
|------|------|
| `hooks/` | Hook 系统（PreToolUse、PostToolUse、Stop hooks） |
| 权限判断 | 工具执行前的权限检查 |
| 沙箱隔离 | 限制本地执行风险 |

## queryLoop 状态机

```mermaid
stateDiagram-v2
    [*] --> PrepareContext: 开始
    PrepareContext --> CallAPI: 整理上下文
    CallAPI --> ReceiveResponse: 流式接收
    
    ReceiveResponse --> HasToolUse: 判断
    
    HasToolUse --> ExecuteTools: 有 tool_use
    ExecuteTools --> FillResult: 执行工具
    FillResult --> Continue_NextTurn: 回填结果
    Continue_NextTurn --> PrepareContext
    
    HasToolUse --> CheckCompletion: 无 tool_use
    
    CheckCompletion --> CheckTruncation: 检查截断
    CheckTruncation --> Recovery_Output: 输出截断
    Recovery_Output --> Continue_Recovery: 恢复
    Continue_Recovery --> PrepareContext
    
    CheckTruncation --> CheckCompact: 未截断
    CheckCompact --> ReactiveCompact: prompt 太长
    ReactiveCompact --> Continue_Compact: 压缩后重试
    Continue_Compact --> PrepareContext
    
    CheckCompact --> CheckHooks: 未超长
    CheckHooks --> StopHook: hook 阻止
    StopHook --> Continue_Hook: 修正后继续
    Continue_Hook --> PrepareContext
    
    CheckHooks --> CheckBudget: hook 通过
    CheckBudget --> Continue_Budget: 预算允许
    Continue_Budget --> PrepareContext
    
    CheckBudget --> Terminal_Completed: 完成
    Terminal_Completed --> [*]
    
    PrepareContext --> Terminal_Error: 异常
    CallAPI --> Terminal_Error
    ExecuteTools --> Terminal_Aborted: 工具中断
    Terminal_Error --> [*]
    Terminal_Aborted --> [*]
```

## Continue Reason 和 Terminal Reason

### Continue Reason（继续原因）

| Reason | 场景 | 含义 |
|--------|------|------|
| `next_turn` | 模型返回工具调用 | 工具结果已回来，继续让模型判断下一步 |
| `reactive_compact_retry` | prompt 太长 | 压缩后重试 |
| `max_output_tokens_recovery` | 输出被截断 | 注入续写提示，让模型继续 |
| `stop_hook_blocking` | hook 要求修正 | 把阻塞原因交给模型 |
| `token_budget_continuation` | 预算还够 | 鼓励模型继续完成 |

### Terminal Reason（终止原因）

| Reason | 含义 |
|--------|------|
| `completed` | 正常完成 |
| `prompt_too_long` | prompt 太长且恢复失败 |
| `model_error` | 模型调用异常 |
| `aborted_tools` | 工具执行被中断 |
| `hook_stopped` | hook 阻止后续继续 |
| `max_turns` | 达到最大轮次 |

## 工具执行链路

```mermaid
flowchart LR
    A[模型返回 tool_use] --> B[查找工具定义]
    B --> C[Schema 校验]
    C --> D[语义校验]
    D --> E[PreToolUse Hooks]
    E --> F[权限判断]
    F --> G{并发判断}
    
    G -->|可并发| H1[并发批次]
    G -->|不可并发| H2[串行批次]
    
    H1 --> I1[并发执行]
    H2 --> I2[串行执行]
    
    I1 --> J[PostToolUse Hooks]
    I2 --> J
    
    J --> K[结果预算处理]
    K --> L[生成 tool_result]
    L --> M[回填给模型]
```

## 压缩与恢复

```mermaid
flowchart TB
    A[上下文接近上限] --> B{触发判断}
    B -->|先尝试| C[History Snip<br/>轻量截断]
    C --> D[Microcompact<br/>局部压缩]
    D --> E[Context Collapse<br/>折叠视图]
    E --> F{压力仍高?}
    
    F -->|是| G[Autocompact<br/>完整摘要]
    F -->|否| H[继续工作]
    
    G --> I[生成 summary]
    I --> J[保存 readFileState]
    J --> K[清空旧状态]
    K --> L[恢复关键现场]
    
    L --> M[文件附件]
    L --> N[Skill 附件]
    L --> O[Plan 附件]
    L --> P[Delta 附件]
    
    M --> H
    N --> H
    O --> H
    P --> H
```

## 技术栈

- **运行时**: Bun (非 Node.js)
- **构建**: `bun build` 单文件打包
- **模块系统**: ESM + TSX
- **UI 框架**: React + 自定义 Ink (Terminal UI)
- **状态管理**: Zustand-style store
- **API 客户端**: Anthropic SDK
- **类型系统**: TypeScript (有 ~1341 个 tsc 错误，但不影响运行)

## 关键设计原则

1. **状态迁移显式化** - 每次 continue 和 terminal 都有明确的 reason
2. **失败可恢复** - 不同失败原因有不同恢复策略
3. **上下文生命周期管理** - Context、Memory、Compression、Cache 协同工作
4. **工具执行链** - Schema → Hooks → Permission → Concurrency → Execution → Budget
5. **多层安全防线** - Prompt → Context → Tool → Permission → Hooks → Sandbox
6. **按需扩展** - Skill、Plugin、MCP、Multi-Agent 按需加载
7. **长任务不断线** - Recovery、Resume、Fallback、Restoration 保证连续性
