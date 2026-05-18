# Agent Runtime Complete Flow

> 基于 `code-src/src` 源码整理，将图 1、图 2、图 3 合并为一张端到端流程图，并在图下方按图中英文节点配中文注释解释完整运行流程。

## Complete Diagram

```text
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 0. Session Ingress（会话入口）                                                            │
│    Terminal REPL        Pipe stdin / -p        SDK stream-json        Resume / Continue   │
└──────────────────────────────────────────┬───────────────────────────────────────────────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. CLI Bootstrap & Mode Adapter（CLI 启动与模式适配）                                      │
│    cli.tsx / main.tsx                                                                     │
│    - parse argv / stdin / settings / MCP / permission mode / session id                   │
│    - dispatch interactive REPL / headless print / SDK / resume                            │
└───────────────────────┬──────────────────────────────────────────────┬───────────────────┘
                        │                                              │
                        ▼                                              ▼
┌────────────────────────────────────────────┐       ┌──────────────────────────────────────┐
│ 2A. Interactive Adapter（交互式适配器）       │       │ 2B. Headless / SDK Adapter（无头适配器）│
│     REPL.tsx                               │       │     cli/print.ts + QueryEngine        │
│     - PromptInput                          │       │     - runHeadless / structured IO      │
│     - UI state / permission overlay        │       │     - stdout / JSON stream             │
│     - handlePromptSubmit                   │       │     - ask / submitMessage              │
└───────────────────────┬────────────────────┘       └─────────────────┬────────────────────┘
                        │                                              │
                        └──────────────────────┬───────────────────────┘
                                               ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 3. Input Normalization / processUserInput（输入归一化）                                    │
├────────────────────────────┬──────────────────────────────┬───────────────────────────────┤
│ Text Prompt（普通文本）       │ Slash / Skill（斜杠/技能）      │ Bash Input Mode（Bash 输入模式） │
│ processTextPrompt          │ processSlashCommand          │ processBashCommand             │
└──────────────┬─────────────┴──────────────┬───────────────┴──────────────┬────────────────┘
               │                            │                              │
               │                            ├── local / local-jsx ─────────►│ Local execution /
               │                            │                              │ UI panel（本地执行/弹窗）
               │                            │                              │ shouldQuery=false
               │                            │
               │                            └── prompt skill ──────────────►│ Prompt expansion
               │                                                           │（展开为模型可见 prompt）
               └────────────────────────────────┬──────────────────────────┘
                                                ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 4. Conversation Orchestration（会话编排）                                                  │
│    REPL onQuery / QueryEngine.submitMessage                                               │
│                                                                                          │
│    Input: new user messages / attachments / slash expansion                               │
│    Assemble: systemPrompt / userContext / systemContext / tools / ToolUseContext          │
│    Maintain: mutableMessages / abortController / usage / file history / transcript        │
└──────────────────────────────────────────┬───────────────────────────────────────────────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 5. Runtime Context Bus / ToolUseContext（运行时上下文总线）                                │
│                                                                                          │
│    messages            AppState snapshot          permission context                       │
│    readFileState       MCP clients/tools          abortController                          │
│    setMessages         setAppState                refreshTools                             │
└───────────────┬──────────────────────────┬──────────────────────────────┬────────────────┘
                │                          │                              │
                ▼                          ▼                              ▼
┌──────────────────────────┐   ┌──────────────────────────┐   ┌────────────────────────────┐
│ Context Builder（上下文构建）│   │ Tool Pool Builder（工具池） │   │ Persistence / Resume（持久化）│
│ context.ts / queryContext │   │ tools.ts + MCP tools     │   │ sessionStorage / transcript │
│ CLAUDE.md / git / date    │   │ deny filter / schema     │   │ sidechain / session resume  │
└──────────────┬───────────┘   └────────────┬─────────────┘   └──────────────┬─────────────┘
               │                            │ tool schemas                  ▲ write stream
               │                            ▼                               │
               │              ┌──────────────────────────┐                  │
               └─────────────►│ 6. Agent Loop / query.ts │◄─────────────────┘
                              └─────────────┬────────────┘
                                            │
                                            ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 6.1 Query Loop Iteration（Query Loop 单轮循环）                                            │
│                                                                                          │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ A. Context Management（上下文治理）                                                  │  │
│  │    getMessagesAfterCompactBoundary / tool-result budget / snip / microcompact       │  │
│  │    contextCollapse / autoCompact / memory prefetch / token budget                   │  │
│  └──────────────────────────────────────┬─────────────────────────────────────────────┘  │
│                                         │                                                │
│                                         ▼                                                │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ B. Model API Request（模型 API 请求）                                                │  │
│  │    services/api/claude.ts                                                           │  │
│  │    normalize messages + system prompt + tool schemas + prompt cache + betas          │  │
│  └──────────────────────────────────────┬─────────────────────────────────────────────┘  │
│                                         │ stream                                         │
│                                         ▼                                                │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ C. Assistant Stream（助手流式响应）                                                   │  │
│  │    text / thinking / tool_use / api_error / stream_event                             │  │
│  └──────────────────────────────────────┬─────────────────────────────────────────────┘  │
│                                         │                                                │
│                          ┌──────────────┴──────────────┐                                 │
│                          │ tool_use block collected ?   │                                 │
│                          │（是否收集到工具调用块）         │                                 │
│                          └──────────────┬──────────────┘                                 │
│                                         │                                                │
│             ┌───────────────────────────┴────────────────────────────┐                   │
│             │                                                        │                   │
│             ▼ yes                                                    ▼ no                │
│  ┌────────────────────────────────────┐     ┌─────────────────────────────────────────┐  │
│  │ D1. Tool Orchestration（工具编排）   │     │ D2. Turn Finalization Checks（收尾检查） │  │
│  │     runTools / StreamingExecutor   │     │     no tool execution branch            │  │
│  └──────────────────┬─────────────────┘     └──────────────────────┬──────────────────┘  │
│                     │                                                │                     │
│                     ▼                                                ▼                     │
│  ┌────────────────────────────────────┐        ┌──────────────────────────────────────┐    │
│  │ permission / hooks / sandbox        │        │ Recovery（恢复）                       │    │
│  │（权限、Hooks、沙箱）                  │        │ 413 / media / max_output_tokens       │    │
│  └──────────────────┬─────────────────┘        └──────────────┬───────────────────────┘    │
│                     │                                         │ if recovered                │
│                     ▼                                         ▼                             │
│  ┌────────────────────────────────────┐        ┌──────────────────────────────────────┐    │
│  │ tool_result as user message         │        │ Stop Hooks（停止 Hooks）               │    │
│  │ write back to messages（写回消息）    │        │ blocking errors / preventContinuation │    │
│  └──────────────────┬─────────────────┘        └──────────────┬───────────────────────┘    │
│                     │                                         │ if blockingErrors           │
│                     ▼                                         ▼                             │
│  ┌────────────────────────────────────┐        ┌──────────────────────────────────────┐    │
│  │ Attachments / refreshTools          │        │ Token Budget（Token 预算）              │    │
│  │ queued commands / memory / skills   │        │ continuation nudge / completed         │    │
│  └──────────────────┬─────────────────┘        └──────────────┬───────────────────────┘    │
│                     │                                         │ if continue                 │
│                     └──────────────────────┬──────────────────┘                             │
│                                            ▼                                                │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ continue next query loop（进入下一轮循环）                                          │  │
│  │ synthesized compact messages / hook error user msg / budget nudge msg / tool result │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
│  Terminal returns（终止返回）: API error unrecoverable / Stop hook preventContinuation /   │
│  no token-budget continuation / max turns / completed                                     │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

## Flow Explanation

### 0. Session Ingress（会话入口）

`Session Ingress` 指所有进入同一套 Agent Runtime 的入口：`Terminal REPL`（终端交互）、`Pipe stdin / -p`（管道或 print 模式）、`SDK stream-json`（SDK 流式 JSON 协议）和 `Resume / Continue`（恢复已有会话）。这些入口最后都会收敛到 CLI 启动逻辑，再根据模式进入 interactive 或 headless 路径。

源码上，真正入口是 `code-src/src/entrypoints/cli.tsx`，它先做运行时 polyfill 和 fast-path 判断，再动态加载主 CLI；`main.tsx` 定义 `-p/--print`、`--input-format stream-json`、`--resume`、`--continue`、permission、MCP、settings 等参数。

### 1. CLI Bootstrap & Mode Adapter（CLI 启动与模式适配）

`CLI Bootstrap & Mode Adapter` 负责 `parse argv / stdin / settings / MCP / permission mode / session id`。这里不是 Agent 决策层，而是运行模式分流层：如果是 `-p/--print` 或 SDK 模式，会导入 `runHeadless`；如果是普通终端，会渲染 `REPL`；如果带 `--resume` / `--continue`，会先加载 transcript、恢复消息和会话元数据，再进入对应路径。

关键源码依据：

- `code-src/src/entrypoints/cli.tsx:54-75`：bootstrap 入口和 profiler。
- `code-src/src/main.tsx:968-1006`：Commander 参数定义，包括 `--print`、`--input-format`、`--resume`、permission、MCP、tools。
- `code-src/src/main.tsx:2825-2860`：headless 分支导入并调用 `runHeadless`。
- `code-src/src/main.tsx:3355-3365`：resume/fromPr/teleport/remote 进入恢复分支。

### 2A. Interactive Adapter（交互式适配器）

`Interactive Adapter` 是 `REPL.tsx`。它承接 `PromptInput`、UI 状态、权限弹窗、streaming text、tool progress、loading state、取消/中断等交互态。用户提交输入时，`REPL` 调用 `handlePromptSubmit`；`handlePromptSubmit` 再统一处理排队、local-jsx immediate command、bash mode、普通 prompt 等路径。

`REPL onQuery` 是交互模式进入 Agent Loop 前的会话门口：它把新消息追加到 UI 消息数组，设置 streaming/tool 状态，调用 `onQueryImpl`，并在 finally 中清理 loading、abort controller 和 turn complete 回调。

关键源码依据：

- `code-src/src/screens/REPL.tsx:2859-2923`：`onQuery` 追加消息、执行 before-query 回调并进入 `onQueryImpl`。
- `code-src/src/screens/REPL.tsx:3492-3523`：提交输入后调用 `handlePromptSubmit`。
- `code-src/src/utils/handlePromptSubmit.ts:120-171`：队列路径复用核心执行逻辑。
- `code-src/src/utils/handlePromptSubmit.ts:227-309`：`local-jsx` immediate command 直接本地执行并展示 UI。

### 2B. Headless / SDK Adapter（无头 / SDK 适配器）

`Headless / SDK Adapter` 对应 `cli/print.ts + QueryEngine`。`runHeadless` 负责结构化 IO、stdout / JSON stream 输出、SDK control messages、MCP tool 更新和 permission prompt tool；`runHeadlessStreaming` 会 drain command queue，然后调用 `ask()`。`ask()` 是 `QueryEngine` 的轻包装：创建一个 `QueryEngine`，再调用 `submitMessage()`。

关键源码依据：

- `code-src/src/cli/print.ts:680-695`：headless 加载初始消息和恢复状态。
- `code-src/src/cli/print.ts:821-826`：构造 headless 权限回调 `canUseTool`。
- `code-src/src/cli/print.ts:863-877`：`runHeadlessStreaming` 流式输出消息。
- `code-src/src/cli/print.ts:1937-2013`：drain command queue、合并 queued prompts、构造 tools 和 MCP clients。
- `code-src/src/cli/print.ts:2150-2214`：调用 `ask()` 并传入 prompt、tools、MCP、权限、AppState、readFile cache 等。
- `code-src/src/QueryEngine.ts:1211-1319`：`ask()` 创建 `QueryEngine` 并调用 `submitMessage()`。

### 3. Input Normalization / processUserInput（输入归一化）

`Input Normalization` 把用户输入统一成消息和控制信号。它输出的核心不是单纯文本，而是：

- `messages`：本轮要加入对话的 user / attachment / system / progress messages。
- `shouldQuery`：是否需要进入模型请求。
- `allowedTools`、`model`、`effort`、`resultText`：slash / skill / headless 输出可能改变本轮配置。

`Text Prompt` 走 `processTextPrompt`，生成普通 user message；`Slash / Skill` 走 `processSlashCommand`，其中 `local / local-jsx` 可能只在本地执行并返回 `shouldQuery=false`，而 `prompt skill` 会展开为模型可见 prompt；`Bash Input Mode` 走 `processBashCommand`。

关键源码依据：

- `code-src/src/utils/processUserInput/processUserInput.ts:85-176`：`processUserInput` 调用 base 处理，若 `shouldQuery=false` 直接返回。
- `code-src/src/utils/processUserInput/processUserInput.ts:178-264`：UserPromptSubmit hooks 可阻止继续或追加上下文。
- `code-src/src/utils/processUserInput/processUserInput.ts:516-550`：bash mode 与 slash command 分流。
- `code-src/src/utils/handlePromptSubmit.ts:396-571`：所有 queued commands 统一循环调用 `processUserInput`，然后调用 `onQuery`。

### 4. Conversation Orchestration（会话编排）

`Conversation Orchestration` 是 `REPL onQuery` 或 `QueryEngine.submitMessage` 负责的层。它接收 `new user messages / attachments / slash expansion`，组装 `systemPrompt / userContext / systemContext / tools / ToolUseContext`，维护 `mutableMessages / abortController / usage / file history / transcript`。

在 headless / SDK 路径中，`QueryEngine.submitMessage()` 每次代表同一 conversation 的一个 turn。它先构建 prompt/context 组件，然后调用 `processUserInput`，再把用户消息先写入 transcript，最后调用 `query()`。这点很关键：源码特意在 API 响应前写用户消息，避免进程被杀时 `--resume` 找不到会话。

关键源码依据：

- `code-src/src/QueryEngine.ts:180-184`：一个 `QueryEngine` 对应一个 conversation，`submitMessage()` 是同会话内的一轮。
- `code-src/src/QueryEngine.ts:287-328`：构建 default system prompt、userContext、systemContext、append prompt。
- `code-src/src/QueryEngine.ts:338-398`：构造 `ProcessUserInputContext` / `ToolUseContext`。
- `code-src/src/QueryEngine.ts:413-431`：调用 `processUserInput`。
- `code-src/src/QueryEngine.ts:433-455`：追加新消息，并在进入 query loop 前写 transcript。
- `code-src/src/QueryEngine.ts:679-690`：调用 `query()`。

### 5. Runtime Context Bus / ToolUseContext（运行时上下文总线）

`Runtime Context Bus / ToolUseContext` 是图里的横切层，不是线性步骤。它把模型循环、工具执行、UI/SDK 状态、权限、MCP、文件读取缓存、abort、消息更新函数连接起来。工具执行不是只拿一个 input，它还会读写 `AppState`、权限上下文、MCP clients、`readFileState`、`messages`，并可能通过 `refreshTools` 在下一轮看到新连接的 MCP 工具。

关键源码依据：

- `code-src/src/Tool.ts:158-250`：`ToolUseContext` 类型，包含 `options.tools`、`mcpClients`、`abortController`、`readFileState`、`getAppState`、`setAppState`、`messages`、`refreshTools` 等。
- `code-src/src/context.ts:36-149`：`getSystemContext` 构建 git/status 等系统上下文。
- `code-src/src/context.ts:155-188`：`getUserContext` 构建 CLAUDE.md 和日期等用户上下文。
- `code-src/src/tools.ts:1-160`：内置工具注册表和 feature-gated 工具入口。
- `code-src/src/cli/print.ts:1473-1501`：headless 路径动态组装 tools，合并内置、SDK MCP、dynamic MCP 和权限上下文过滤。

### 6. Agent Loop / query.ts（Agent 循环）

`Agent Loop` 是真正推动模型和工具闭环的层。`query()` 本身是薄包装，它调用 `queryLoop()`，并在正常返回后补齐 command lifecycle；`queryLoop()` 内部是 `while (true)`，每次循环都从 `State` 解构出 `messages / toolUseContext / turnCount / recovery counters`，完成一次“准备上下文 -> 请求模型 -> 处理流 -> 工具或收尾”的迭代。

关键源码依据：

- `code-src/src/query.ts:203-217`：跨轮 mutable `State` 字段。
- `code-src/src/query.ts:219-239`：`query()` 调用 `queryLoop()`。
- `code-src/src/query.ts:241-321`：`queryLoop()` 初始化 state 并进入 `while (true)`。

### A. Context Management（上下文治理）

`Context Management` 发生在每次请求模型之前。它先从 compact boundary 后取消息，然后依次应用 tool result budget、history snip、microcompact、context collapse、autoCompact，并准备 `fullSystemPrompt`。这说明 compact / memory / cache 不是模型之后的补充层，而是每轮 request 前的治理管线。

关键源码依据：

- `code-src/src/query.ts:365-394`：取 compact boundary 后消息，并应用 tool result budget。
- `code-src/src/query.ts:396-426`：history snip 与 microcompact。
- `code-src/src/query.ts:428-447`：context collapse。
- `code-src/src/query.ts:449-535`：append system context、autoCompact、post-compact messages。
- `code-src/src/utils/api.ts:437-474`：`appendSystemContext` 和 `prependUserContext` 的实际注入方式。

### B. Model API Request（模型 API 请求）

`Model API Request` 由 `services/api/claude.ts` 执行。`queryLoop()` 调用 `deps.callModel` 时传入 `prependUserContext(messagesForQuery, userContext)`、`fullSystemPrompt`、tools、thinkingConfig、signal、model、MCP tools、agent definitions、taskBudget 等。

API 层再完成：tool schema 构建、message normalization、`tool_use` / `tool_result` pairing 修复、media 限制处理、system blocks、prompt cache、betas、max tokens、thinking 和 stream request。

关键源码依据：

- `code-src/src/query.ts:659-708`：`deps.callModel` 的参数。
- `code-src/src/services/api/claude.ts:1020-1072`：API generator 入口、agentic query 和 beta 计算。
- `code-src/src/services/api/claude.ts:1232-1247`：tool schema 构建。
- `code-src/src/services/api/claude.ts:1260-1317`：normalize messages、repair pairing、strip advisor/excess media。
- `code-src/src/services/api/claude.ts:1358-1380`：组装 system prompt blocks 和 prompt caching。
- `code-src/src/services/api/claude.ts:1539-1730`：构造最终 API params。
- `code-src/src/services/api/claude.ts:1819-1837`：调用 `anthropic.beta.messages.create({ stream: true })`。

### C. Assistant Stream（助手流式响应）

`Assistant Stream` 不是一次性返回完整 message。API 层处理 `message_start`、`content_block_start`、`content_block_delta`、`content_block_stop`、`message_delta`、`message_stop`，并把每个完成的 content block 规范化成内部 `AssistantMessage` 逐个 yield。流里可能出现 `text`、`thinking`、`tool_use`、`api_error`、`stream_event`。

`queryLoop()` 收到 assistant message 后，会扫描 content 里的 `tool_use` block。源码注释明确说 `stop_reason === 'tool_use'` 不可靠，所以是否进入工具分支以“实际收集到 `tool_use` block”为准。

关键源码依据：

- `code-src/src/services/api/claude.ts:1941-1980`：遍历 raw stream events。
- `code-src/src/services/api/claude.ts:1996-2113`：处理 content block start/delta，累积 tool input JSON。
- `code-src/src/services/api/claude.ts:2172-2212`：content block stop 时 yield `AssistantMessage`。
- `code-src/src/services/api/claude.ts:2214-2278`：message_delta 写回 usage / stop_reason，并可生成 max-output API error。
- `code-src/src/query.ts:551-558`：`toolUseBlocks` 和 `needsFollowUp`。
- `code-src/src/query.ts:828-838`：扫描 assistant message 中的 `tool_use` 并设置 `needsFollowUp=true`。

### D1. Tool Orchestration（工具编排）

如果收集到 `tool_use block`，循环进入 `Tool Orchestration`。这里有两种执行方式：普通 `runTools` 和 `StreamingToolExecutor`。普通方式先按 `isConcurrencySafe` 分批，连续只读/并发安全工具可并发，不安全工具串行；streaming executor 则在 tool_use block 流式到达时就尝试启动，并按顺序回收结果。

关键源码依据：

- `code-src/src/query.ts:1363-1386`：选择 `StreamingToolExecutor.getRemainingResults()` 或 `runTools(...)`。
- `code-src/src/services/tools/toolOrchestration.ts:19-82`：`runTools` 批次调度。
- `code-src/src/services/tools/toolOrchestration.ts:91-116`：按 `isConcurrencySafe` 分区。
- `code-src/src/services/tools/StreamingToolExecutor.ts:34-40`：streaming executor 设计目标。
- `code-src/src/services/tools/StreamingToolExecutor.ts:76-124`：tool_use 到达后加入队列并启动处理。
- `code-src/src/services/tools/StreamingToolExecutor.ts:453-490`：等待并 yield 剩余结果。

### permission / hooks / sandbox（权限、Hooks、沙箱）

每个工具调用会进入 `runToolUse`，再进入 `checkPermissionsAndCallTool`。实际顺序是：解析和校验 tool input，运行 `PreToolUse hooks`，解析 hook permission decision，调用 `canUseTool` 做权限裁决，允许后执行 `tool.call`，再映射 tool result，并运行 `PostToolUse hooks`。如果权限拒绝或 hook 阻止继续，会生成对应 `tool_result` 或 attachment。

图里写 `sandbox`，是因为权限和工具执行链路会携带 sandbox / permission mode 相关上下文；具体工具（尤其 Bash）在自身实现和权限系统中处理 sandbox 策略。当前图保持在编排层，不把 sandbox 误画成 query loop 的独立顺序步骤。

关键源码依据：

- `code-src/src/services/tools/toolExecution.ts:337-490`：`runToolUse` 查找工具、处理中断、调用权限和执行管线。
- `code-src/src/services/tools/toolExecution.ts:599-733`：input schema 和 tool 自定义 validation。
- `code-src/src/services/tools/toolExecution.ts:795-862`：`PreToolUse hooks`。
- `code-src/src/services/tools/toolExecution.ts:916-1104`：权限决策与拒绝结果。
- `code-src/src/services/tools/toolExecution.ts:1206-1222`：调用 `tool.call`。
- `code-src/src/services/tools/toolExecution.ts:1290-1474`：映射 tool result 为 `tool_result` user message。
- `code-src/src/services/tools/toolExecution.ts:1481-1588`：`PostToolUse hooks` 和 hook-stopped-continuation attachment。

### tool_result as user message（工具结果作为用户消息写回）

工具结果不是直接塞进 assistant message，而是以 `tool_result` content block 包在 `UserMessage` 中返回给模型。`queryLoop()` yield 这些消息，同时把它们 normalize 成 API 可见 user messages 加入 `toolResults`。随后还会注入 queued command attachments、memory prefetch、skill discovery attachments，并在下一轮 `State.messages` 中拼接 `messagesForQuery + assistantMessages + toolResults`。

关键源码依据：

- `code-src/src/query.ts:1387-1403`：yield tool update，并把 tool result normalize 成 user messages。
- `code-src/src/query.ts:1550-1593`：工具后注入 queued command attachments。
- `code-src/src/query.ts:1595-1631`：注入 memory prefetch 和 skill discovery attachments。
- `code-src/src/query.ts:1662-1674`：下一轮前刷新 tools。
- `code-src/src/query.ts:1717-1730`：构造下一轮 `State`，transition 为 `next_turn`。

### D2. Turn Finalization Checks（无工具调用后的收尾检查）

如果没有收集到 `tool_use block`，不是直接结束，而是进入 `Turn Finalization Checks`。这部分是图 3 的核心修正点：`no tool_use` 只代表跳过工具执行分支，后面还要做恢复、Stop Hooks、Token Budget、成功性判定。

`Recovery` 先处理可恢复错误：prompt too long / 413 可以先 context-collapse drain，再 reactive compact；media size 错误可走 reactive compact；`max_output_tokens` 可先提高 max tokens 或注入 meta recovery message 让模型继续。若恢复成功，就 `continue next query loop`。

`Stop Hooks` 在模型给出有效最终响应后运行。若 hook `preventContinuation`，直接 return；若 hook 返回 blocking errors，则把 blocking error 作为消息追加回上下文并继续下一轮。

`Token Budget` 如果启用且判断需要继续，会注入 budget nudge meta user message 并继续下一轮；如果不需要继续，才返回 completed。

关键源码依据：

- `code-src/src/query.ts:1065-1088`：进入 `!needsFollowUp` 收尾路径并识别 withheld 413 / media。
- `code-src/src/query.ts:1088-1178`：context-collapse / reactive compact 恢复，成功则 continue，失败则返回错误。
- `code-src/src/query.ts:1188-1259`：`max_output_tokens` escalation / recovery message / exhausted handling。
- `code-src/src/query.ts:1261-1268`：API error 跳过 stop hooks 并返回。
- `code-src/src/query.ts:1270-1309`：Stop Hooks，`preventContinuation` return，blocking errors continue。
- `code-src/src/query.ts:1311-1344`：Token Budget continuation。
- `code-src/src/query.ts:1360`：所有检查通过后 return completed。

## Summary

完整链路可以压缩成一句话：

```text
Session Ingress 进入 CLI mode adapter，输入被 processUserInput 归一化为消息；
Conversation Orchestration 组装 prompt/context/tools/ToolUseContext；
Agent Loop 每轮先治理上下文，再请求模型流；
如果流里有 tool_use，就执行工具并把 tool_result 写回 user message 后继续；
如果没有 tool_use，就进入恢复、Stop Hooks、Token Budget 等收尾检查，满足终止条件后 completed。
```

这张图的重点是边界：`Runtime Context Bus / ToolUseContext` 是横切状态总线，`Tool Orchestration` 是模型行动闭环，`Turn Finalization Checks` 是无工具调用后的收尾闭环。三者合起来，才是源码中真实的 Agent Runtime。
