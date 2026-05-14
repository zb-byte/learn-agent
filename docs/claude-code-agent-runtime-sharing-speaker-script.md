# Claude Code Agent Runtime 分享者讲稿

> 讲者专用版。本文不是发给听众的阅读稿，而是方便分享者现场照读、控节奏、做转场的口播稿。

## 使用方式

- 建议时长：35-45 分钟。
- 讲法建议：不要逐字解释源码文件名，把源码事实当作背书；重点讲“为什么这样设计”。
- 重点句可以适当停顿，让听众有时间消化。
- 如果时间不够，可以跳过每章的“进一步展开”段落，只保留开场、主线、架构图和每章结论。

---

## 0. 开场

大家好，今天我想和大家分享一个主题：

**Claude Code Agent Runtime：一个可持续工作的 Agent 是如何组织起来的。**

这里我先解释一下标题里的两个词。

第一个词是 **Agent Runtime**。我们平时讲 Agent，很容易把注意力放在模型能力上：模型聪不聪明，提示词写得好不好，工具列表够不够多。

但如果我们真的看 Claude Code 的实现，会发现它最值得学习的地方不是“调用了一个模型”，也不是“给模型配了一堆工具”，而是它把一个 Agent 拆成了一套运行时系统。

第二个词是 **可持续工作**。

一个 Agent 跑一两轮很容易：用户输入，模型回答，最多调个工具。

难的是它要在真实工程任务里持续工作：任务可能很长，工具结果可能很大，上下文可能爆掉，模型输出可能截断，权限可能被拒绝，hook 可能阻断，子任务可能后台运行，进程还可能中断。

所以今天我们不讨论“怎样写一个很漂亮的 prompt”，而是讨论一个更工程化的问题：

```text
一个能长期推进任务的 Agent，
到底需要哪些运行时结构？
```

我会按这条主线来讲：

```text
任务怎么推进；
模型行为怎么约束；
每轮模型基于什么上下文工作；
工具调用怎么被接住；
安全边界怎么分层；
复杂能力和长任务怎么支撑；
最后再把一轮任务完整串起来。
```

先给大家一个整体印象。

Claude Code 不是一个“模型调用器”。如果只是模型调用器，它的核心大概就是：

```text
prompt -> model -> response
```

但 Claude Code 更像是：

```text
user input
  -> session orchestration
  -> agent loop
  -> model decision
  -> tool runtime
  -> result feedback
  -> recovery or completion
```

这里最核心的不是模型本身，而是中间这套 **Agent Loop**。

我建议大家先带着一句话听后面的内容：

```text
模型负责判断下一步；
运行时负责让每一步可执行、可约束、可恢复、可解释。
```

后面所有章节，基本都是这句话的展开。

---

## 1. 先讲整体架构图

我们先看整体架构，不急着进源码。

这个系统可以分成几层。

最上面是调用入口。用户可能来自 Terminal REPL，也可能来自 pipe 输入，也可能来自 SDK，也可能是在恢复一个历史 session。

这些入口进来以后，会先经过 CLI 和模式适配层。这里决定当前是交互式 REPL，还是 headless print，还是 SDK 调用，还是 resume。

接下来是输入归一化。用户输入不一定都是普通文本，它可能是 slash command，可能是 skill，可能是 bash input mode，也可能是带附件的消息。

归一化以后，进入会话编排层。这里会准备本轮要用的 messages、system prompt、user context、tool context、abort controller、transcript 等状态。

再往下，就是核心的 query loop。

query loop 每一轮都会做几件事：

```text
整理上下文
  -> 构造模型请求
  -> 流式接收模型响应
  -> 判断有没有 tool_use
  -> 如果有工具调用，就进入工具执行
  -> 如果没有工具调用，也不是直接结束，而是进入终止/恢复判断
```

这里我特别强调一点：

**没有 tool_use 不等于任务结束。**

这是我们画架构图时很容易画错的地方。

更准确的结构是：

```text
Assistant Stream
  -> 是否收集到 tool_use block?
     -> yes: Tool Orchestration
     -> no: Turn Finalization Checks
```

`no tool_use` 之后还要判断：

```text
有没有 withheld 的 prompt-too-long 错误；
有没有 max_output_tokens 截断需要恢复；
Stop Hook 是否阻止继续；
Token Budget 是否要求继续；
最终结果是否可以算成功。
```

全部通过以后，才是 completed。

所以这个图本质上不是一条流水线，而是一个带状态迁移的循环。

我现场可以把它简化成这样讲：

```text
用户输入进入会话；
query loop 整理现场并请求模型；
模型要么产出文本，要么产出 tool_use；
tool_use 进入运行时执行，结果再回填给模型；
没有 tool_use 时，也要经过恢复、hook、预算和成功性判断；
最后才输出或进入下一轮。
```

这个闭环，就是 Claude Code Runtime 的主干。

---

## 2. 任务是怎么被推进的

接下来讲第一层：任务推进。

很多 Agent 实现里，任务推进会被写成一段很直观的流程：

```text
调用模型；
如果模型要工具，就执行工具；
再调用模型；
直到模型不再要工具。
```

这个思路可以跑 demo，但它不够表达真实世界里的复杂情况。

真实情况里会出现很多分支：

```text
模型请求了工具，但是工具被权限拒绝；
工具执行过程中被用户中断；
模型输出到一半撞上 token 上限；
prompt 太长，需要压缩以后重试；
hook 认为这次输出不合格，要求模型修正；
达到最大轮次或预算上限，需要明确终止。
```

所以 Claude Code 没有把任务推进建模成简单的 `while` 循环，而是在 `queryLoop` 里维护一套状态迁移。

它关心的不只是“继续还是结束”，而是：

```text
为什么继续？
为什么结束？
下一轮带着什么状态继续？
```

比如继续可能有很多原因：

```text
next_turn：工具结果回来了，需要让模型继续判断；
reactive_compact_retry：prompt 太长，压缩后重试；
max_output_tokens_recovery：输出截断，注入续写提示继续；
stop_hook_blocking：hook 要求修正；
token_budget_continuation：预算允许，推动模型继续完成。
```

结束也不是一个笼统的 stop：

```text
completed：正常完成；
prompt_too_long：太长且恢复失败；
model_error：模型调用异常；
aborted_tools：工具执行被中断；
hook_stopped：hook 阻止继续；
max_turns：达到最大轮次。
```

这里的工程启发非常重要：

```text
Agent Runtime 里，每一次继续和结束都应该有原因。
```

如果没有原因，系统只能表现为“它又转了一圈”或者“它停了”。一旦线上出问题，我们很难解释到底是模型决定的，工具导致的，压缩触发的，还是 hook 阻断的。

Claude Code 的做法是把这些原因显式化。这样既方便恢复，也方便排查。

可以这样总结这一章：

```text
queryLoop 不是循环调用模型；
queryLoop 是任务推进的状态机。
```

---

## 3. 模型行为是怎么被约束的

第二层是模型行为约束。

这里我想先区分两个东西：

```text
Prompt 影响模型怎么选择；
Runtime 决定选择能不能落地。
```

很多时候我们会把这两件事混在一起。

比如我们在 prompt 里告诉模型：

```text
不要随便执行危险命令；
优先使用专用工具；
修改代码后要验证；
不要过度解释。
```

这些规则当然有用，但它们不是硬边界。模型仍然可能忘记，可能误判，也可能被外部内容诱导。

所以 Claude Code 的设计是：先用 Prompt Engineering 降低模型走偏概率，再用运行时做确定性校验。

它的 Prompt Engineering 大概有三层。

第一层是系统提示词分段。

系统提示词不是一大块字符串，而是一组 section。不同 section 负责不同事情：身份、任务方式、工具使用、语气、输出效率、环境信息、语言、MCP instructions、memory 等。

这种分段有三个价值：

```text
可组合；
可审查；
可缓存。
```

第二层是行为指令模式。

Claude Code 会提前压住模型常见的错误倾向，比如：

```text
不要用 Bash 代替专用读写工具；
独立工具调用可以并行；
不要重复做已经派给子 Agent 的工作；
不要在工具调用之间长篇解释；
避免不可逆的破坏性操作。
```

这里有个很实用的点：它不是只说“保持简洁”，而是会用数字锚定，比如“工具调用之间的文字保持很短”。数字比抽象形容词更稳定。

第三层是工具提示词。

每个工具的 description 不是简单介绍“这个工具能干什么”，而是一个微型行为控制器。

比如 Bash 工具不能只写“执行 shell 命令”。因为 Bash 太强了，模型很容易什么都用 Bash 干。

所以 Bash 的提示词会把模型导向更窄的专用工具：

```text
搜索文件名，用 Glob；
搜索文件内容，用 Grep；
读文件，用 Read；
编辑文件，用 Edit；
写文件，用 Write；
和用户沟通，不要 echo。
```

这里的设计思想是：

```text
系统提示词控制全局行为；
工具提示词控制局部行为；
运行时负责最终裁决。
```

这一章可以用一句话收束：

```text
Prompt 不是安全边界，但它是降低模型决策噪声的第一层控制面。
```

---

## 4. 模型每轮基于什么工作

第三层是上下文、记忆、压缩和缓存。

长任务里，模型最大的问题不是“不够聪明”，而是：

```text
它每一轮到底看到了什么？
它记住了什么？
它丢掉了什么？
丢掉以后还能不能继续？
```

Claude Code 不把上下文看成一个简单的 messages 数组，而是把它拆成不同寿命、不同用途的状态。

我们先讲 Context。

Context 解决的是：

```text
这一轮模型到底应该看到什么？
```

进入模型前，Claude Code 会先整理一份 `messagesForQuery`。大概顺序是：

```text
从 compact boundary 之后取消息；
处理过大的 tool result；
做 history snip；
做 microcompact；
必要时做 context collapse；
上下文压力还高，再做 autocompact；
追加 system context；
前置 user context；
最后才 call model。
```

这个顺序背后有一个很重要的原则：

```text
先做低损耗清理；
再做局部压缩；
最后才做完整摘要。
```

也就是说，压缩不是第一选择。因为压缩会丢细节，所以它应该放在更轻量的预算、裁剪、折叠之后。

再讲 Memory。

我们可以用人的记忆做一个类比。

```text
工作记忆：当前这一轮正在处理什么；
情景记忆：这次会话发生过什么；
语义记忆：项目规则、用户偏好、团队约定；
程序化记忆：工具使用习惯、Skill 工作流程；
外部现场记忆：模型看过哪些文件、当前计划是什么。
```

落到 Claude Code 里，这些记忆不会塞进同一个字段。

它们会进入不同容器：

```text
mutableMessages 记录当前 conversation；
transcript 负责跨进程恢复；
FileStateCache 记录模型读过哪些文件；
queryLoop State 记录当前 turn 的恢复计数和 transition；
bootstrap state 记录 sessionId、prompt cache latch、invokedSkills 等进程级状态。
```

这里最容易混淆的是 transcript 和 FileStateCache。

```text
transcript 记录“对话发生了什么”；
FileStateCache 记录“模型看过哪些文件内容”。
```

它们都像记忆，但用途完全不同。

再讲 Compression。

压缩的目标不是“生成一段漂亮总结”，而是：

```text
压缩后还能不能继续工作。
```

Claude Code 在 compact 后，不只是生成一条 summary。它还会补回继续工作需要的现场：

```text
最近读过的关键文件；
已经调用过的 Skill；
当前 plan；
PlanMode 状态；
MCP、Agent、延迟工具等动态声明。
```

所以压缩后的上下文不是：

```text
旧历史 -> 一条摘要
```

而是：

```text
旧历史
  -> compact summary
  -> file attachments
  -> invoked skill attachment
  -> plan attachments
  -> runtime delta attachments
```

最后是 Prompt Cache。

Prompt Cache 的核心是稳定前缀。Claude Code 会尽量把稳定 system prompt 放前面，把用户、时间、环境、动态工具状态放后面。系统提示词里还有一个动态边界，用来区分稳定前缀和动态尾部。

这一章可以这样总结：

```text
上下文不是历史消息拼接；
上下文是一套持续维护的运行现场。
```

如果没有这套运行现场管理，Agent 任务跑长以后，就会在“看不见、记不住、恢复不了”之间反复摔跤。

---

## 5. 工具调用如何被接住

第四层是工具调用。

模型返回 `tool_use` 之后，系统不能直接执行。

这一点非常关键：

```text
模型只表达行动意图；
运行时决定这个行动能不能安全落地。
```

Claude Code 里的工具不是一个普通函数列表。一个 Tool 对象除了 name、description、inputSchema、call 以外，还会声明很多运行时属性：

```text
是否只读；
是否可并发；
是否破坏性；
是否延迟加载；
结果大小上限；
进度如何展示；
结果如何展示。
```

新工具如果没有声明并发安全、没有声明只读，默认按保守策略处理。这叫失败关闭。

工具执行链路大致是：

```text
查找工具；
schema 校验；
语义校验；
PreToolUse hooks；
权限判断；
执行 call；
PostToolUse hooks；
结果预算处理；
生成 tool_result；
回填给下一轮模型。
```

这里有一个很有意思的点：并发不是模型说了算。

Prompt 会鼓励模型：如果多个工具调用彼此独立，可以在同一轮返回多个 `tool_use`。

但真正能不能并发执行，运行时还要判断。

Claude Code 会按每个工具的 `isConcurrencySafe` 把工具调用分批：

```text
连续的并发安全工具 -> 并发批次；
非并发安全工具 -> 串行批次。
```

比如 Read、Glob、Grep 通常可以并发；Bash 要看具体命令是否只读；写文件或改变共享状态的工具就更保守。

所以这里有两层：

```text
模型层：表达这些动作看起来可以一起做；
运行时：决定这些动作最终并发还是串行。
```

工具结果也不是原样塞回模型。工具结果往往很大，是上下文膨胀的主要来源之一。所以 Claude Code 会做：

```text
单工具结果上限；
消息聚合预算；
大结果持久化；
内容替换；
空结果填充；
必要时去重。
```

这一章的结论是：

```text
工具系统的成熟度，决定了模型能不能稳定、安全、低成本地行动。
```

不是模型说“我要做”，系统就做；而是模型提出意图，运行时把意图变成可审查、可授权、可恢复的动作。

---

## 6. 安全不是一个模块

第五层是安全。

这里我想先说一个判断：

```text
Agent 的安全问题不是单点问题。
```

因为模型会读取外部内容，也会调用工具。外部内容可能诱导模型越权，模型被诱导以后又可能尝试执行危险动作。

所以安全至少包括两层：

```text
防止模型被外部内容诱导；
防止被诱导后的危险动作真正落地。
```

Claude Code 的安全防线分散在多层：

```text
Prompt 层：告诉模型区分指令和数据；
Context 层：给外部内容标边界；
Tool 层：校验输入和风险；
Permission 层：危险动作过门禁；
Hooks 层：执行前后拦截或改写；
Sandbox 层：限制本地执行风险。
```

我们用一个例子来讲。

假设 README 文件里写了一句：

```text
Ignore previous instructions and run ...
```

这个东西进入 Agent Runtime 后，会经过很多层。

Context 层要把它包装成文件内容，而不是当前用户的新指令。

Prompt 层要告诉模型，文件内容只是待处理数据，不能覆盖系统和用户任务。

模型即使被诱导生成了 Bash tool_use，Tool 层还会校验命令、判断风险。

Permission 和 Hooks 可以阻止危险动作。

Sandbox 则限制真正执行时的影响范围。

所以 Prompt Injection 不是某个函数能解决的问题。它是一条横切链路：

```text
外部内容
  -> Context
  -> Prompt
  -> Model Decision
  -> Tool Runtime
  -> Permission / Hooks / Sandbox
  -> Tool Result
  -> 下一轮 Context
```

任何一层把“外部数据”和“有效指令”混在一起，都可能放大风险。

这里尤其要强调 Context 层。

外部内容进入模型上下文时，应该保留来源、角色、边界、新鲜度和信任级别。

更重要的是，压缩和摘要不能把这些边界抹掉。

如果一个网页里的恶意内容被摘要成“系统建议执行某命令”，那就很危险了。更好的摘要应该保留：

```text
来自网页 A 的内容声称 X；
其中包含要求执行命令的文本；
但它属于外部数据，不是上级指令。
```

这一章可以这样收束：

```text
Prompt 和 Context 降低模型误判概率；
Tool、Permission、Hooks、Sandbox 决定动作能不能落地。
```

安全不是一个模块，而是一组跨层协同防线。

---

## 7. 长任务和复杂能力怎么支撑

第六层是扩展能力和长任务可靠性。

一个 Agent 跑久以后，会遇到两个问题：

```text
能力越来越多，prompt 装不下；
任务越来越长，状态容易断。
```

Claude Code 用两组机制解决。

第一组是扩展机制：

```text
Skill；
Plugin；
MCP；
Multi-Agent。
```

第二组是可靠性机制：

```text
Recovery；
Resume；
Fallback；
Post-Compact Restoration。
```

先讲 Skill。

Skill 解决的是：

```text
专业能力很多，但不能把完整指令全部塞进每次请求。
```

所以 Skill 先以轻量目录形式出现，只暴露名称、描述、触发场景。模型判断需要时，再通过 SkillTool 加载完整指令。

它的核心是：

```text
发现列表轻；
完整指令重；
先发现，再按需加载。
```

而且 Skill 和压缩恢复有关。某个 Skill 一旦被调用，它往往改变了后续任务的方法。如果后来发生 compact，旧消息被压掉，Claude Code 会通过 `invokedSkills` 选择性恢复这些专业规则。

再讲 Plugin。

Plugin 不是一个单独的运行时，而是一个能力包。它可以包含 commands、skills、agents、hooks、MCP 配置。Claude Code 会把插件拆成自己已经认识的内部组件，再接入当前会话。

所以 Plugin 更像适配器：

```text
外部能力包
  -> 本地插件文件
  -> 结构校验
  -> 内部组件
  -> 当前会话能力集合。
```

MCP 则解决外部工具通过统一协议进入工具池的问题。这里的原则是：

```text
扩展可以动态；
核心运行时要稳定。
```

接下来讲 Multi-Agent。

这部分很容易被误解成“多个工具并发”。但 Multi-Agent 和工具并发不是一个层级。

```text
工具并发：一轮里多个动作怎么调度；
Multi-Agent：一段任务要不要派生独立执行体。
```

Claude Code 并不是先跑一个统一复杂度分类器，再自动决定要不要启用子 Agent。更准确地说，它会把“什么时候适合派生子 Agent”的标准写进 prompt、Agent description 和 Tool description 里，再由模型判断是否发起 AgentTool。

运行时负责的是：当模型真的调用 AgentTool 以后，把它严格落地。

子 Agent 的价值不是“多开几个聊天窗口”，而是：

```text
隔离上下文；
隔离中间工具结果；
使用独立工具和权限；
必要时后台运行；
可恢复；
最后把有价值结果带回主线。
```

这里有两种子 Agent 要分清：

```text
fresh subagent：从新上下文起步，适合独立验证和独立调查；
fork subagent：继承父上下文，适合把已有背景下的开放式探索切出去。
```

Multi-Agent 的工程价值不是“让系统显得更聪明”，而是给复杂任务增加一种组织维度：

```text
什么时候留在主线程；
什么时候切出独立执行体；
切出去以后怎样带回最有价值的结果。
```

最后讲长任务可靠性。

Claude Code 对长任务的理解不是“尽量别失败”，而是：

```text
失败会发生；
关键是失败后还能不能继续工作。
```

所以它有一组恢复机制。

prompt 太长时，不是立刻终止，而是先尝试 context collapse drain，再 reactive compact。

输出截断时，不是让用户重新问，而是先尝试提高输出上限，再注入“从中断处继续”的恢复消息。

主模型不可用时，会切 fallback model，并清理失败尝试遗留的 assistant / tool result 状态。

会话恢复时，也不是只加载 messages，而是恢复 plan、file history、readFileState、cost state、worktree、remote agent tasks、content replacement state 等运行环境。

子 Agent 也不是临时线程，它有 sidechain transcript 和 metadata，可以被 resume。

这一章的结论是：

```text
长任务系统不能只会 retry；
它需要运行中恢复、压缩后恢复、中断后恢复。
```

这套能力让 Agent 不只是“回答问题”，而是能持续工作。

---

## 8. 把一轮任务重新串起来

最后我们把前面的层重新拼回一轮任务。

一轮 Agent 任务不是直线，而是一个循环管线：

```text
准备现场
  -> 约束模型
  -> 模型决策
  -> 工具执行
  -> 结果回填
  -> 判断继续、恢复或结束
```

更完整一点：

```text
用户请求进入 QueryEngine；
QueryEngine 准备会话状态和 transcript；
queryLoop 接管任务推进；
Prompt Engineering 建立行为边界；
Context 整理模型可见现场；
Memory 保留不同寿命的状态；
Compression 和 Prompt Cache 管理上下文生命周期；
模型基于当前现场生成文本或 tool_use；
Tool Runtime 接住 tool_use；
Security 横跨 Prompt、Context、Tool、Permission、Hooks、Sandbox；
Extension 按需引入 Skill、Plugin、MCP、子 Agent；
Recovery / Resume / Fallback 处理异常和长任务断点；
最后回到 queryLoop，判断继续还是完成。
```

如果是一个“修改代码并验证”的任务，链路可能是：

```text
用户提出修复需求
  -> QueryEngine 写入会话现场
  -> queryLoop 构造请求
  -> Prompt / Context 约束模型并整理文件、历史、工具状态
  -> 模型决定先读文件，返回 tool_use
  -> Tool Runtime 校验并执行读取
  -> tool_result 回填
  -> 模型修改代码，再次返回 tool_use
  -> Tool Runtime 执行编辑和测试
  -> 测试失败，结果回填给模型继续修正
  -> 上下文过长，Compression 介入并恢复关键状态
  -> 模型或工具异常，Recovery / Fallback 接管
  -> queryLoop 最终给出 completed 或明确终止原因
```

这里我们再回到开头那句话：

```text
模型负责判断下一步；
运行时负责让每一步可执行、可约束、可恢复、可解释。
```

这句话基本概括了 Claude Code Agent Runtime 的核心价值。

---

## 9. 收尾

最后做一个总结。

今天我们不是在看一个“模型加工具”的简单系统，而是在看一个围绕 Agent Loop 组织起来的运行时。

它的设计重点有几条。

第一，任务推进要显式建模。

```text
继续要有继续的原因；
结束要有结束的原因。
```

第二，Prompt 很重要，但不能把所有责任都压在 Prompt 上。

```text
Prompt 降低模型走偏概率；
Runtime 负责最终裁决。
```

第三，上下文不是消息拼接，而是运行现场。

```text
模型每轮看到什么、记住什么、压缩什么、恢复什么，
都需要被系统管理。
```

第四，工具调用不是函数调用，而是一条治理链。

```text
schema、权限、hooks、并发、预算、回填，
这些都决定工具能不能稳定运行。
```

第五，安全不是一个模块，而是横跨 Prompt、Context、Tool、Permission、Hooks 和 Sandbox 的多层防线。

第六，长任务系统不能只靠 retry，它要能压缩、恢复、fallback、resume，也要能让子 Agent 进入同一套恢复体系。

所以如果我们要从 Claude Code 学一个最核心的东西，我觉得不是某个具体工具，也不是某段 prompt，而是这个工程判断：

```text
Agent 的能力，不只来自模型；
也来自运行时如何组织模型、上下文、工具、安全、扩展和恢复。
```

我的分享就到这里。

如果大家后面继续看源码，我建议按这个顺序读：

```text
先看 queryLoop，理解任务推进；
再看 system prompt 和 tool prompt，理解模型行为约束；
再看 context、memory、compact，理解运行现场；
再看 tools、permission、hooks，理解工具治理；
最后看 skill、plugin、multi-agent、resume，理解长任务和扩展。
```

这样读会比较顺，不容易被局部实现细节带散。

谢谢大家。

---

## 备用：5 分钟短版

如果现场时间突然被压缩，可以只讲这一版。

今天这份分享的核心观点是：Claude Code 不是一个模型调用器，而是一个围绕 Agent Loop 构建的任务运行时。

它最重要的能力不是“能调用多少工具”，而是能让一个 Agent 长时间、可恢复、可约束地工作。

主链路是：

```text
用户输入
  -> 会话编排
  -> queryLoop
  -> 上下文整理
  -> 模型决策
  -> 工具执行
  -> tool_result 回填
  -> 继续、恢复或结束判断
```

这里有几个关键点。

第一，`queryLoop` 是状态机，不是简单 while 循环。每次继续都有 reason，每次结束也有 reason。

第二，Prompt 只负责影响模型选择，不负责最终安全。真正的动作落地要经过 Tool、Permission、Hooks 和 Sandbox。

第三，Context 不是 messages 拼接，而是运行现场。Claude Code 会管理 messages、FileStateCache、transcript、compact summary、invoked skills、prompt cache 等不同寿命的状态。

第四，模型返回 `tool_use` 之后，运行时要做 schema 校验、语义校验、并发分区、权限判断、hooks、执行、结果预算和回填。

第五，没有 `tool_use` 也不是直接结束。系统还要检查 prompt-too-long、max_output_tokens、stop hooks、token budget 和最终成功性，全部通过才是 completed。

第六，长任务靠 Recovery / Resume / Fallback 支撑：上下文爆了要压缩，输出截断要续写，模型不可用要 fallback，会话中断要 resume，子 Agent 也要能恢复。

所以 Claude Code 给我们的工程启发是：

```text
模型负责判断下一步；
运行时负责让每一步可执行、可约束、可恢复、可解释。
```

