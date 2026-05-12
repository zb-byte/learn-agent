# Claude Code 源码深度解析 — 架构、设计与工程实践

> 面向技术负责人的内部分享
>
> 基于反编译还原的 Claude Code v2.1.888 源码分析

---

## 目录

1. [项目概览](#1-项目概览)
2. [整体架构](#2-整体架构)
3. [启动链路：从 CLI 到 REPL](#3-启动链路从-cli-到-repl)
4. [核心查询循环](#4-核心查询循环)
5. [工具系统](#5-工具系统)
6. [终端 UI 框架](#6-终端-ui-框架)
7. [状态管理](#7-状态管理)
8. [API 层与多云适配](#8-api-层与多云适配)
9. [上下文工程](#9-上下文工程)
10. [权限与安全模型](#10-权限与安全模型)
11. [构建与特性开关](#11-构建与特性开关)
12. [工程亮点与可借鉴实践](#12-工程亮点与可借鉴实践)
13. [总结](#13-总结)

---

## 1. 项目概览

### 这是什么

Claude Code 是 Anthropic 官方的 CLI 编码助手，用户在终端中与 Claude 交互，完成代码编写、调试、重构、Git 操作等任务。本分析基于社区反编译还原的源码版本。

### 关键数据

| 维度 | 数据 |
|------|------|
| 源码文件 | **2,768** 个 TypeScript/TSX 文件 |
| 代码行数 | **~51.5 万行** |
| 工具数量 | **57+** 个内置工具 |
| 组件数量 | **147+** 个 UI 组件 |
| 构建产物 | 单文件 bundle ~25MB (5,326 模块) |
| 运行时 | **Bun**（非 Node.js） |
| UI 框架 | **React 19 + 自定义 Ink 终端渲染器** |
| tsc 类型错误 | ~1,341 个（来自反编译，不影响运行时） |

### 技术栈

```
Runtime:   Bun (ESM, TSX)
UI:        React 19 + React Compiler + 自定义 Ink reconciler
语言:      TypeScript 6.0 + TSX (react-jsx transform)
验证:      Zod 4.x (tool schema / input validation)
CLI:       Commander.js
包管理:    Bun workspaces
图像:      Sharp
SDK:       @anthropic-ai/sdk 0.80
```

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                    用户终端 (Terminal)                     │
├─────────────────────────────────────────────────────────┤
│  REPL.tsx (5009行) — 交互式 REPL 屏幕                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────────────┐ │
│  │ 消息渲染   │ │ 输入处理   │ │ 权限审批 UI              │ │
│  │ Messages  │ │ PromptIn │ │ PermissionRequest       │ │
│  └──────────┘ └──────────┘ └──────────────────────────┘ │
├─────────────────────────────────────────────────────────┤
│  QueryEngine.ts (1320行) — 会话编排层                      │
│  ┌──────────────────────────────────────────────────────┐│
│  │ query.ts (1732行) — 核心查询循环 (AsyncGenerator)      ││
│  │   消息构建 → API 调用 → 流式处理 → 工具执行 → 结果回传   ││
│  └──────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────┤
│  工具层 (57+ tools)                                       │
│  ┌─────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌─────┐ ┌──────┐  │
│  │Bash │ │File* │ │Grep  │ │Agent │ │Web* │ │ MCP  │  │
│  └─────┘ └──────┘ └──────┘ └──────┘ └─────┘ └──────┘  │
├─────────────────────────────────────────────────────────┤
│  服务层                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐│
│  │API 调用   │ │MCP 客户端│ │上下文构建  │ │权限/策略     ││
│  │claude.ts │ │client.ts │ │context.ts│ │permissions  ││
│  └──────────┘ └──────────┘ └──────────┘ └─────────────┘│
├─────────────────────────────────────────────────────────┤
│  基础设施层                                               │
│  AppState (Zustand-like) · Ink 渲染器 · Feature Flags    │
└─────────────────────────────────────────────────────────┘
```

**分层原则**：自上而下是 UI → 业务编排 → 工具执行 → 服务/基础设施。层间通过明确的类型接口（Message、Tool、PermissionResult）通信。

---

## 3. 启动链路：从 CLI 到 REPL

整个启动流程经过精心优化，分为快速路径和完整路径：

```
cli.tsx (bootstrap)
  ├─ 快速路径: --version → 直接输出版本号（零模块加载）
  ├─ 快速路径: --daemon-worker / --dump-system-prompt
  └─ 完整路径: → main.tsx
                  ├─ Commander.js 参数解析
                  ├─ 并行初始化:
                  │   ├─ MDM 原始数据读取 (plutil/reg query)
                  │   ├─ macOS Keychain 预取 (OAuth + API Key)
                  │   ├─ GrowthBook Feature Flags
                  │   ├─ 远程托管设置加载
                  │   └─ 官方 MCP 服务器 URL 预取
                  ├─ 认证检查 (OAuth / API Key)
                  ├─ 信任对话框 / 引导流程
                  ├─ 会话恢复 (如有中断的会话)
                  └─ launchRepl()
                      └─ Ink.render(<App><REPL /></App>)
```

### 关键工程决策

1. **启动性能优化**：`--version` 走零依赖快速路径；MDM 和 Keychain 读取在 import 阶段并行启动
2. **动态导入**：`main.tsx` 中 Coordinator、Kairos 等模块通过 `require()` + feature flag 按需加载
3. **Side-effect 顺序**：profile checkpoint 在最顶部，确保启动计时精确

---

## 4. 核心查询循环

`query.ts` 是整个系统的心脏——一个 **AsyncGenerator** 驱动的流式对话循环：

```
用户输入
  │
  ▼
normalizeMessagesForAPI() — 消息格式标准化
  │
  ▼
prependUserContext() + appendSystemContext() — 注入上下文
  │
  ▼
createStream() — 调用 Anthropic API (流式)
  │
  ▼
┌─────────────────────────────────────────────┐
│            流式事件处理循环                     │
│                                             │
│  text_delta    → 渲染文本到 UI                │
│  tool_use      → 权限检查 → 执行工具           │
│  tool_result   → 回传工具结果给 API            │
│  thinking      → 处理思考过程                  │
│  stop_reason   → 判断是否需要继续              │
│                                             │
│  循环直到: stop_reason = end_turn             │
└─────────────────────────────────────────────┘
  │
  ▼
auto-compact 检测 — 上下文过长时自动压缩
  │
  ▼
返回 StreamEvent 给 REPL 渲染
```

### query() 核心签名

```typescript
export async function* query(
  messages: Message[],
  options: QueryOptions,
): AsyncGenerator<StreamEvent>
```

### QueryEngine 的编排职责

`QueryEngine.ts` 在 `query()` 之上提供：
- **会话管理**：消息持久化、会话恢复
- **文件状态追踪**：编辑历史、文件快照
- **归因管理**：Git commit 归属追踪
- **SDK 兼容层**：Agent SDK 集成
- **Compaction**：上下文压缩触发与执行

---

## 5. 工具系统

### 工具接口设计

```typescript
type Tool = {
  name: string
  description(): string
  inputSchema: ToolInputJSONSchema     // Zod → JSON Schema
  call(args, context, canUseTool, parentMessage, onProgress): Promise<ToolResult>

  // 权限
  checkPermissions(input, context): Promise<PermissionResult>

  // 渲染
  renderToolUseMessage(args, message): ReactNode
  renderToolResultMessage(result, message): ReactNode

  // 行为标记
  isReadOnly(): boolean
  isDestructive(): boolean
  isConcurrencySafe(): boolean
  isOpenWorld(): boolean        // 访问外部资源
  isSearchOrReadCommand(): boolean  // UI 折叠优化
}
```

### 工具分类

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件操作** | FileRead, FileEdit, FileWrite, Glob, Grep | 读写搜索，核心能力 |
| **执行** | Bash, PowerShell | Shell 命令执行 |
| **代理** | Agent, SendMessage, Task* | 子代理、团队协作 |
| **Web** | WebSearch, WebFetch, WebBrowser | 网络搜索与浏览 |
| **规划** | EnterPlanMode, ExitPlanMode, VerifyPlanExecution | 结构化规划 |
| **Git** | EnterWorktree, ExitWorktree | 隔离工作区 |
| **交互** | AskUserQuestion, SkillTool | 用户交互 |
| **MCP** | MCPTool, McpAuth, ListMcpResources | MCP 协议扩展 |
| **Jupyter** | NotebookEdit | Notebook 操作 |

### 工具注册机制

```typescript
// src/tools.ts — 集中注册
export function getAllBaseTools(): Tools { ... }
export function getTools(permissionContext): Tools { ... }
export function assembleToolPool(options): Tools { ... }
```

- 每个工具在独立目录 `src/tools/<ToolName>/`
- 懒加载 schema：`lazySchema(() => z.strictObject({...}))`
- Feature flag 控制可用工具集
- MCP 工具动态注入，与内置工具去重合并

### 工具 Schema 示例 (BashTool)

```typescript
const inputSchema = lazySchema(() => z.strictObject({
  command: z.string().describe('The command to execute'),
  timeout: semanticNumber(z.number().optional()),
  description: z.string().optional(),
  run_in_background: semanticBoolean(z.boolean().optional()),
  dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional()),
}))
```

`semanticNumber` / `semanticBoolean` 允许字符串语义值（如 "true"/"false"）自动转换——这是 LLM 工具调用中的实用设计。

### MCP 工具桥接

外部 MCP 服务器工具通过 `buildMcpToolName()` 转换为内置格式：

```
mcp__<serverName>__<toolName>  →  内置 Tool 对象
```

关键桥接点：`src/services/mcp/client.ts` 将 MCP 的 JSON Schema、annotations（readOnlyHint、destructiveHint）映射为内部 Tool 接口。

---

## 6. 终端 UI 框架

### 自定义 Ink 架构

Claude Code 没有使用标准 Ink 库，而是 **fork 了一套自定义终端渲染框架**：

```
src/ink/
  ├── ink.tsx      — 核心渲染器 (5000+ 行)
  ├── reconciler.ts — React reconciler 集成
  ├── root.ts      — Root 创建与 render 管理
  ├── screen.ts    — 终端屏幕缓冲区
  └── termio/      — 终端 I/O 抽象层
```

**核心设计**：
- 基于 `react-reconciler` 构建自定义渲染器
- 集成 **Yoga** 布局引擎（与 React Native 相同的 Flexbox 实现）
- DOM 树结构（DOMElement / TextNode）映射 React 组件到终端
- 屏幕缓冲区 diff，最小化终端写入

### 组件体系

```
App.tsx (根 Provider)
  ├─ REPL.tsx (5009行 — 主屏幕)
  │   ├─ Messages.tsx — 消息列表
  │   │   └─ VirtualMessageList — 虚拟滚动
  │   ├─ PromptInput/ — 输入系统
  │   │   ├─ PromptInput.tsx (99KB — 最复杂组件)
  │   │   ├─ PromptInputFooter — 状态栏
  │   │   └─ HistorySearchInput — 历史搜索
  │   └─ permissions/ — 35 个权限审批组件
  │       ├─ BashPermissionRequest/
  │       └─ FileEditPermissionRequest/
  ├─ messages/ — 40+ 消息类型渲染组件
  │   ├─ UserTextMessage
  │   ├─ AssistantTextMessage
  │   ├─ AssistantToolUseMessage
  │   └─ CollapsedReadSearchContent
  └─ design-system/ — 主题化组件库
```

### React Compiler 集成

所有组件使用 React Compiler 的 `_c()` 进行自动记忆化：

```typescript
import { c as _c } from "react/compiler-runtime";

function Component(t0) {
  const $ = _c(42);  // 缓存槽位
  let t1;
  if ($[0] !== prop1) {
    t1 = <Expensive />;
    $[0] = prop1;
    $[1] = t1;
  } else {
    t1 = $[1];  // 缓存命中，跳过重渲染
  }
  return t1;
}
```

这是反编译产物的典型模式。React Compiler 在编译时自动插入缓存逻辑，避免手动 `useMemo` / `useCallback`。

### 性能优化策略

| 策略 | 实现 |
|------|------|
| **虚拟滚动** | `VirtualMessageList` — 只渲染可见 + overscan 消息 |
| **Markdown 缓存** | 500 条 token 缓存，避免重复解析 |
| **屏幕缓冲区 diff** | 最小化终端 I/O |
| **Viewport culling** | ScrollBox 中的视口裁剪 |
| **量化滚动** | 减少滚动事件触发的重渲染频率 |

---

## 7. 状态管理

### 自建轻量 Store

没有使用 Redux 或 Zustand，而是 **自建了一个极简 store**：

```typescript
// src/state/store.ts
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T>
```

### AppState 结构

```typescript
// src/state/AppStateStore.ts
type AppState = {
  messages: Message[]           // 对话消息列表
  tools: Tools                 // 当前可用工具集
  toolPermissionContext: ...   // 权限上下文
  settings: ...                // 用户设置
  queryState: ...              // 查询状态
  uiState: ...                 // UI 状态（主题、滚动等）
  // ... 50+ 字段
}
```

### 数据流

```
AppState (全局状态)
    │
    ├─ useAppState(selector) — 组件订阅状态切片
    ├─ store.setState(updater) — 更新状态
    │
    ▼
React 重渲染 (React Compiler 自动优化)
    │
    ▼
Ink 渲染器 → 终端输出
```

---

## 8. API 层与多云适配

### 多 Provider 支持

```typescript
// src/services/api/claude.ts (3420行)

支持的 Provider:
├─ Anthropic Direct    — API Key / OAuth
├─ AWS Bedrock         — Credential 刷新, Bearer Token
├─ Google Vertex       — GCP Credentials
└─ Azure Foundry       — API Key / Azure AD
```

### 流式调用模式

```typescript
// 核心流程
const stream = await client.messages.create({
  model: ...,
  max_tokens: ...,
  system: [...systemPromptBlocks],
  messages: [...normalizedMessages],
  tools: [...toolSchemas],
  stream: true,              // 开启流式
  betas: [...modelBetas],    // Beta 功能
})

// 事件处理
for await (const event of stream) {
  switch (event.type) {
    case 'content_block_delta':   // 文本/工具增量
    case 'content_block_stop':    // 内容块结束
    case 'message_stop':          // 消息结束
    case 'message_delta':         // usage 统计
  }
}
```

### 重试与容错

- `streamWithRetry()` — 指数退避重试
- `FallbackTriggeredError` — Provider 降级
- Prompt cache break detection — 缓存失效检测

---

## 9. 上下文工程

Claude Code 最核心的竞争力之一是 **上下文构建**——在每次 API 调用前，动态组装丰富的项目上下文：

### System Context 来源

```
src/context.ts
  │
  ├─ Git 状态
  │   ├─ 当前分支、主分支
  │   ├─ 工作区变更 (git status)
  │   └─ 最近 commit 消息
  │
  ├─ CLAUDE.md 文件
  │   ├─ 项目根目录 CLAUDE.md
  │   ├─ 子目录 CLAUDE.md
  │   └─ 用户全局 ~/.claude/CLAUDE.md
  │
  ├─ Memory 文件
  │   ├─ .claude/memory/ 下的持久记忆
  │   └─ 按类型分类 (user, feedback, project, reference)
  │
  ├─ 环境信息
  │   ├─ 平台 (macOS/Linux/Windows)
  │   ├─ 当前日期
  │   └─ 工作目录
  │
  └─ System Prompt 常量
      ├─ 核心行为指令
      ├─ 工具使用指南
      ├─ 权限模式说明
      └─ Git 集成指令
```

### 上下文优化策略

| 策略 | 说明 |
|------|------|
| **Memoize** | `getGitStatus()` 等 function 被 `lodash/memoize` 缓存 |
| **Auto-compact** | 上下文超限时自动压缩旧消息 |
| **Reactive compact** | (Feature flag) 响应式压缩 |
| **Context collapse** | (Feature flag) 上下文折叠 |
| **Micro-compact** | 小范围压缩，保留关键信息 |

---

## 10. 权限与安全模型

### 权限模式

```typescript
type PermissionMode =
  | 'default'              // 每次询问
  | 'plan'                 // 规划模式（只读）
  | 'auto'                 // 自动审批（基于规则）
  | 'bypassPermissions'    // 绕过所有权限
  | 'acceptEdits'          // 自动接受编辑
  | 'dontAsk'              // 自动拒绝
  | 'bubble'               // 向上层冒泡
```

### 权限检查流程

```
工具调用请求
  │
  ├─ 1. tool.checkPermissions() — 工具级别检查
  ├─ 2. 文件系统权限 — 读/写路径白名单
  ├─ 3. Gitignore 风格规则匹配
  ├─ 4. Pre-tool-use Hook 执行
  ├─ 5. 安全分类器 (auto 模式)
  └─ 6. 用户审批 UI (如需要)
      └─ PermissionRequest 组件渲染
```

### 规则格式

```
Bash                          — 整个工具
Bash(git *)                   — glob 匹配
Bash(prefix:git status)       — 前缀匹配
Edit(~/.claude/**)            — 路径匹配
```

### 危险文件/目录保护

```typescript
const DANGEROUS_FILES = ['.gitconfig', '.bashrc', '.zshrc', '.mcp.json', ...]
const DANGEROUS_DIRECTORIES = ['.git', '.vscode', '.idea', '.claude', ...]
```

---

## 11. 构建与特性开关

### Feature Flag 系统

```typescript
import { feature } from 'bun:bundle'

if (feature('COORDINATOR_MODE')) {
  // 这段代码在构建时如果 flag=false 会被完全剔除 (DCE)
}
```

**85 个 Feature Flag**，主要类别：

| 类别 | Flags |
|------|-------|
| 核心功能 | DAEMON, BG_SESSIONS, BRIDGE_MODE |
| 代理高级 | AGENT_TRIGGERS, AGENT_MEMORY_SNAPSHOT |
| AI 功能 | COORDINATOR_MODE, KAIROS, PROACTIVE |
| UI/UX | VOICE_MODE, AUTO_THEME |
| 实验性 | CONTEXT_COLLAPSE, REACTIVE_COMPACT |
| Ant 专属 | REPL_TOOL, CHICAGO_MCP |

### 构建流程

```bash
bun build src/entrypoints/cli.tsx --outdir dist --target bun
# → dist/cli.js (~25MB, 单文件, 5326 模块)
```

`feature()` 在开发环境中被 polyfill 为始终返回 `false`：
```typescript
// cli.tsx 顶部
const feature = (_name: string) => false;
```

---

## 12. 工程亮点与可借鉴实践

### 1. AsyncGenerator 驱动的查询循环

`query()` 返回 `AsyncGenerator<StreamEvent>`，用生成器模式优雅处理流式对话。相比回调/Promise 链，生成器提供了：
- 自然的暂停/恢复语义
- 消费者可以按需拉取事件
- 与 React 组件生命周期天然契合

**适用场景**：任何需要处理长生命周期流式交互的系统。

### 2. 自定义终端 UI 渲染器

基于 `react-reconciler` + Yoga 构建自定义终端渲染器，而非依赖 Ink 库。这让团队可以：
- 精确控制渲染性能（屏幕缓冲区 diff）
- 实现虚拟滚动等高级特性
- 避免上游库的限制

**适用场景**：需要高度定制终端 UI 的 CLI 工具。

### 3. 工具系统设计

57+ 工具统一实现 `Tool` 接口，包含权限检查、进度追踪、结果渲染等完整生命周期。关键设计：
- **Lazy schema**：延迟构建 JSON Schema，减少启动开销
- **Behavior flags**（readOnly, destructive, concurrencySafe）：让上层无需理解工具细节即可做出安全决策
- **MCP 桥接**：外部工具透明地转为内部 Tool 对象

**适用场景**：需要可扩展工具链的 AI Agent / 编码助手。

### 4. 启动性能优化

- `--version` 零依赖快速路径
- I/O 密集操作（Keychain、MDM）在 import 阶段并行启动
- 动态 import 隔离重模块

**适用场景**：需要快速启动的 CLI 工具。

### 5. 上下文工程

Claude Code 的核心竞争力——将项目上下文（Git、CLAUDE.md、Memory）系统化地注入每次 API 调用。这比简单的 system prompt 工程要复杂得多：
- 多源上下文聚合
- 分层缓存（memoize）
- 上下文压缩（auto-compact）
- Token 预算管理

**适用场景**：任何 LLM 应用的 prompt 管理策略。

### 6. Feature Flag 驱动的代码剔除

85 个 feature flag 配合 Bun 的 tree-shaking，实现：
- 同一份代码库编译出不同能力的产品版本
- Ant 专属功能完全不出现在外部构建中
- 实验性功能可以独立开关

**适用场景**：多 SKU 产品、SaaS 功能灰度发布。

---

## 13. 总结

### 架构核心理念

Claude Code 的架构可以总结为三个关键词：

1. **流式优先** — 从 API 调用到 UI 渲染，全链路流式处理，用户体验极致流畅
2. **工具驱动** — 所有能力通过统一的 Tool 接口暴露，可扩展、可组合、可审批
3. **上下文丰富** — 系统化地将项目信息注入 LLM，是产品体验差异化的关键

### 对团队的启示

| 方面 | 启示 |
|------|------|
| **AI 应用架构** | Agent 循环不复杂，但工具系统、权限模型、上下文管理才是工程难点 |
| **CLI 工具** | React + 自定义 reconciler 可以构建非常复杂的终端 UI |
| **工程规模** | 51 万行代码的 TypeScript 项目，需要清晰的分层和模块化 |
| **构建策略** | Feature flag + DCE 是管理复杂产品矩阵的有效手段 |
| **性能** | 启动优化（快速路径、并行 I/O）和运行时优化（虚拟滚动、缓存）都需要系统性设计 |

### 一句话总结

> Claude Code 不是一个"调 API 的 CLI 脚本"，而是一个 **工程化的 AI Agent 运行时**——其复杂度集中在工具生命周期管理、权限安全模型、流式交互架构和上下文工程四个维度上。
