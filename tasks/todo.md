# 任务：优化 5.3 Context 层

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 目标小节：`#### 5.3 Context 层：外部内容需要边界`
- 优化目标：在不扩大章节范围的前提下，补足 Context 层如何承载安全边界的解释。
- 风格要求：延续原文工程导读风格，保持中文表达清晰、克制、可读。
- 边界要求：不改写 5.4 权限层和 5.5 Hooks / Sandbox 的职责，不引入未经上下文支撑的实现细节。

## 执行计划

- [x] 检查 5.3 原文与相邻章节语气边界。
  - 摘要：5.3 当前偏短，只表达了“包裹、标记、放置”的原则，缺少上下文边界的具体作用机制。
- [x] 优化 5.3 小节正文。
  - 摘要：增加外部内容进入上下文时的来源、位置、边界、摘要和衰减原则。
- [x] 校验章节结构、术语一致性和 diff。
  - 摘要：确认只改目标小节与任务文档，Markdown 标题层级保持稳定。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

- 已将 5.3 从原则性短段落扩展为完整说明，覆盖外部内容来源、角色、边界、新鲜度、信任级别和压缩摘要中的边界保留。
- 已明确 Context 层的职责边界：它负责降低模型误解概率，但不替代 Tool / Permission 的运行时校验。
- 已校验目标章节标题层级，`5.3`、`5.4`、`5.5` 顺序保持正常。
- 已检查 diff，正文改动限定在 `docs/claude-code-agent-runtime-sharing.md` 的 5.3 小节，另新增本任务记录。

---

# 任务：优化 6.1 Skill 小节

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 目标小节：`#### 6.1 Skill：专业指令按需进入`
- 优化目标：在不扩大第 6 章整体范围的前提下，补足 Skill 按需加载机制的设计动机、生命周期、运行时影响和恢复关系。
- 风格要求：延续原文工程导读风格，保持概念解释清晰，不做源码逐行讲解。
- 边界要求：不展开 Plugin、MCP、Multi-Agent 的细节，不把 Skill 描述成独立执行沙箱或权限边界。

## 执行计划

- [x] 检查 6.1 原文与第 6 章上下文。
  - 摘要：6.1 当前只有生命周期骨架，缺少“为什么需要轻量发现、完整加载、压缩恢复”的解释。
- [x] 优化 6.1 小节正文。
  - 摘要：补充 Skill 的目录化发现、触发加载、执行方式、上下文注入和 invokedSkills 恢复。
- [x] 校验章节结构、术语一致性和 diff。
  - 摘要：确认只扩展 6.1 与任务记录，不破坏 6.2 后续章节衔接。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

- 已将 6.1 从生命周期骨架扩展为完整机制说明，覆盖 Skill 的延迟加载动机、轻量目录、完整指令加载、inline / fork 差异和 `invokedSkills` 恢复。
- 已明确 Skill 的职责边界：它是按需进入运行时的工作方法，不是 Plugin、MCP、Multi-Agent 或权限系统的替代品。
- 已校验第 6 章标题层级，`6.1` 到 `6.5` 顺序保持正常。
- 已检查 Markdown 代码围栏数量，当前为偶数，未发现围栏截断。

---

# 任务：优化 6.2 Plugin 小节

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 目标小节：`#### 6.2 Plugin：外部能力包先物化再激活`
- 标题问题：原标题里的“物化”偏内部实现术语，读者不容易立刻理解 Plugin 的核心作用。
- 优化目标：改成更直白的标题，并补充 Plugin 从外部能力包进入 Claude Code 运行时的过程。
- 风格要求：延续第 6 章工程导读风格，解释机制但不做源码逐行讲解。
- 边界要求：不展开 MCP 协议细节，不把 Plugin 混同于单个 Skill，也不把 Plugin 描述成权限系统。

## 执行计划

- [x] 检查 6.2 原文和相邻章节衔接。
  - 摘要：当前 6.2 只有生命周期骨架和一句价值判断，标题和正文都偏抽象。
- [x] 修改 6.2 标题和正文。
  - 摘要：将标题改为更清楚的“把外部能力包展开为运行时能力”，并补充声明、安装意图、本地落地、校验读取、组件拆分、会话激活。
- [x] 校验章节结构、围栏和 diff。
  - 摘要：确认 6.2 与 6.1、6.3 衔接正常，Markdown 结构未破坏。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

- 已将标题从 `Plugin：外部能力包先物化再激活` 改为 `Plugin：把外部能力包展开为运行时能力`，降低内部术语感。
- 已扩展 6.2 正文，说明 Plugin 与 Skill 的粒度差异，以及 commands、skills、agents、hooks、MCP 配置如何进入运行时。
- 已补充生命周期阶段解释，覆盖 `plugin.json`、settings / marketplace、reconciler、pluginLoader、component loaders、refreshActivePlugins。
- 已明确 Plugin 和 MCP 的边界：Plugin 可以包含 MCP 配置，但 Plugin 本身不是 MCP。

---

# 任务：优化 6.4 Multi-Agent 小节

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 目标小节：`#### 6.4 Multi-Agent：子 Agent 是受控派生`
- 标题优化：改成更直白的 `Multi-Agent：把复杂任务拆给受控子 Agent`。
- 优化目标：补足 Multi-Agent 在复杂任务中的作用，说明为什么要拆、如何受控派生、与 Skill / Plugin / MCP 的区别。
- 风格要求：延续第 6 章工程导读风格，解释运行时策略，不展开源码逐行讲解。
- 边界要求：不把 Multi-Agent 写成多个模型自由协作，也不暗示子 Agent 可以绕过主运行时权限和工具约束。

## 执行计划

- [x] 检查 6.4 原文和相邻章节衔接。
  - 摘要：当前 6.4 只有 AgentTool 链路骨架，缺少任务拆分、上下文隔离和受控边界解释。
- [x] 修改 6.4 标题和正文。
  - 摘要：补充子 Agent 的使用场景、受控派生链路、执行模式和与其他扩展机制的区别。
- [x] 校验章节结构、围栏和 diff。
  - 摘要：确认 6.4 与 6.3、6.5 衔接正常，Markdown 结构未破坏。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

- 已将标题从 `Multi-Agent：子 Agent 是受控派生` 改为 `Multi-Agent：把复杂任务拆给受控子 Agent`。
- 已扩展 6.4 正文，说明子 Agent 适合承接大范围探索、多方案调研、独立验证、并行排查等高噪声任务。
- 已补充“受控”的具体含义，覆盖任务入口、AgentDefinition、工具上下文、权限边界、上下文隔离和生命周期。
- 已明确 Multi-Agent 与 Skill、Plugin、MCP 的区别：它不是能力接入本身，而是复杂任务的执行组织方式。

---

# 任务：优化第 7 章任务链路总结

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 目标章节：`## 7. 一轮任务会穿过哪些层`
- 优化目标：将清单式总结改成更清楚的端到端任务流水线，帮助读者理解一轮任务如何穿过编排、上下文、模型、工具、安全、扩展和恢复层。
- 风格要求：延续全文工程导读风格，作为总结章节需要更有收束感，不展开新的源码细节。
- 边界要求：不新增第 1-6 章未支撑的新概念，不把链路写成严格线性流程，保留“循环和恢复”的运行时特征。

## 执行计划

- [x] 检查第 7 章原文和前后章节衔接。
  - 摘要：当前第 7 章是 11 条清单，覆盖完整但缺少层与层之间的输入输出关系。
- [x] 重写第 7 章链路说明。
  - 摘要：补充端到端流程、关键层职责表、典型任务示例和闭环总结。
- [x] 校验标题结构、围栏和 diff。
  - 摘要：确认第 7 章与第 8 章衔接正常，Markdown 结构未破坏。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

- 已将第 7 章从 11 条清单改为端到端运行时链路说明，强调 Agent 任务不是一次性直线流程，而是可循环、可恢复的管线。
- 已补充“输入 / 处理 / 输出”视角的分层表，帮助读者看清 QueryEngine、Agent Loop、Prompt / Context、Model、Tool Runtime、Memory / Compression、Extension、Reliability 的边界。
- 已增加“修改代码并验证”的典型链路示例，串起工具调用、结果回填、测试失败继续修正、压缩和恢复路径。
- 已保留总结判断：Claude Code 的核心价值来自多个模块围绕 Agent Loop 形成闭环。

---

# 任务：校准系统提示词分段与 Prompt Cache 描述

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 优化目标：根据源码澄清 system prompt section 的计算策略与 prompt cache 边界策略，避免把“动态段”“每轮重算”“缓存边界后”混为一谈。
- 风格要求：直接更新文档，保持工程导读表达，不展开源码逐行说明。
- 边界要求：不把 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 写成 section 计算缓存；不把 `resolvedDynamicSections` 全部写成每轮重新计算。

## 执行计划

- [x] 检查第 2.1 和第 3 章相关表述。
  - 摘要：确认原文存在“稳定段 / 动态段 / 每轮重算 / 缓存边界”并列的问题，且“工作现场”作为核心术语不够贴近源码。
- [x] 更新文档正文。
  - 摘要：将第 2.1 改成两条轴：section 计算策略和 prompt cache 边界策略；将“工作现场”替换为“运行上下文 / 上下文状态”。
- [x] 校验 Markdown 结构和 diff。
  - 摘要：已确认旧的并列式描述和“工作现场”残留被清除，`git diff --check` 无空白错误。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

- 已将第 2.1 的系统提示词分段说明改为两条轴：`systemPromptSection` / `DANGEROUS_uncachedSystemPromptSection` 负责 section 计算策略，`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 负责 API prompt cache 边界。
- 已明确 `resolvedDynamicSections` 不等于全部每轮重算，缓存边界之后也不等于不能进入模型。
- 已将文档中“工作现场”统一改成“运行上下文 / 上下文状态 / 上下文管理”，避免引入源码里没有的核心术语。

---

# 任务：重写 6.4 Multi-Agent 章节

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 目标小节：`#### 6.4 Multi-Agent：把复杂任务拆给受控子 Agent`
- 优化目标：汇总前两轮源码讨论，补足何时使用多 Agent、与并行 tool call 的区别、fork / fresh 子 Agent 的边界、Agent 记忆是否独立，以及 Multi-Agent 的核心收益。
- 风格要求：直接替换现有章节，保持全文工程导读风格，既讲机制，也讲判断标准。
- 边界要求：不把 tool 并行误写成 Multi-Agent，不把 fork 与 fresh subagent 混同，不把 Agent 记忆说成完全共享或完全割裂。

## 执行计划

- [x] 检查当前 6.4 内容和相邻章节衔接。
  - 摘要：原章节已说明“受控派生”，但对使用时机、tool 并行差异、记忆隔离层次还不够完整。
- [x] 用源码结论重写 6.4。
  - 摘要：新增使用时机表、tool vs Multi-Agent 对照、fork / fresh 区分、记忆分层表和受控边界总结。
- [x] 校验 Markdown 结构、术语一致性和 diff。
  - 摘要：已检查章节替换后的表格、代码块和术语；`git diff --check` 无空白错误。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

- 已用“使用时机、与并行 tool 的区别、收益、fork / fresh 区分、记忆独立性、受控边界”六块内容重写 6.4。
- 已明确：并行 tool call 是动作级并行，Multi-Agent 是任务级派生。
- 已补充 Agent 记忆的分层结论：运行态隔离、fresh / fork 上下文差异、独立 agent memory、sidechain transcript。
- 已保留与 Skill / Plugin / MCP 的机制区分，避免把 Multi-Agent 写成能力接入层。

---

# 任务：重写 6.5 Recovery / Resume / Fallback

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 目标小节：`#### 6.5 Recovery / Resume / Fallback：长任务不断线`
- 优化目标：结合源码，把该节从“异常分类表”升级为“Claude Code 为了长任务持续性补齐了哪些系统能力”，并提炼对 Agent Runtime 设计的启发。
- 风格要求：保持工程导读语气，重点讲系统能力和设计思路，不写成源码逐行注释。
- 边界要求：不把 recovery 简化成 retry；不把 resume 简化成加载 transcript；不漏掉 subagent resume 和交互层 auto-restore。

## 执行计划

- [x] 梳理 Overflow Recovery、Output Recovery、Model Fallback、Resume、Post-Compact Restoration、Agent Resume、UX Recovery 的源码证据。
  - 摘要：确认 query、REPL、compact、resumeAgent 等路径各自补齐了不同断点。
- [x] 重写 6.5 章节。
  - 摘要：按“恢复能力地图 -> 六类机制 -> 工程启发”重组正文，突出系统完整性。
- [x] 校验 Markdown 结构、术语一致性和 diff。
  - 摘要：已完成 diff 检查并清理 3 处行尾空格；`git diff --check` 通过。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

- 已将 6.5 从“异常 -> 手段”的静态表格，改写为“Claude Code 为持续工作补齐哪些恢复能力”的系统讲解。
- 已覆盖运行中恢复、压缩后恢复、中断后恢复三条主线，并补充模型 fallback、subagent resume、用户取消后的 prompt auto-restore。
- 已强调 Resume 恢复的是可继续工作的运行环境，而不只是 transcript；Recovery 也不是 retry 的别名，而是一整套续跑策略。

---

# 任务：补充用户挫败信号的文档说明

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 优化目标：基于源码，补充 Claude Code 对“用户可能处于挫败/暴躁状态”的处理边界。
- 放置原则：不把这部分误写成安全机制、恢复机制或完整情绪管理系统；更适合作为运行时总链路中的交互质量观测补充。
- 风格要求：点到为止，维持全文工程导读语气。

## 执行计划

- [x] 检查现有章节结构并确定插入位置。
  - 摘要：第 7 章负责把运行时能力重新拼回一轮任务，适合补充“系统也会记录交互质量信号”的说明。
- [x] 更新文档正文。
  - 摘要：新增一段简短说明，澄清源码中可见的是负面关键词检测、埋点和内部反馈收集入口，而非通用情绪安抚机制。
- [x] 校验 Markdown 结构和 diff。
  - 摘要：确认新增内容不打断第 7 章主链路，且表述与源码结论一致。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

---

# 任务：分析 auto-mind 的运行时能力

## 需求规格

- 分析对象：`/Users/wangzhongbin/Documents/code/630/auto-mind`
- 参考视角：`docs/claude-code-agent-runtime-sharing.md`
- 分析目标：提炼 auto-mind 已具备的 Agent Runtime 能力，包括任务推进、上下文/状态共享、工具与执行、记忆、干预、安全、可观测性与恢复机制。
- 输出要求：以“能力清单 + runtime sharing 映射 + 总体判断”为主，不做逐文件流水账。
- 边界要求：结论必须能回指到仓库文档或源码，不把设计意图和已实现能力混为一谈。

## 执行计划

- [x] 回顾参考文档的分析框架，并确定 auto-mind 的映射维度。
  - 摘要：围绕 Agent Loop、Context / Memory、Tool Runtime、Security、Extension、Reliability 组织分析。
- [x] 阅读 auto-mind 的 README、架构文档和核心入口类。
  - 摘要：定位系统边界、请求入口、执行主链路与主要角色。
- [x] 梳理 auto-mind 的运行时共享模型。
  - 摘要：重点检查 RequestSession、RequestScope、ContextSnapshot、CurrentTurn、EventBus、Observation drain 等结构。
- [x] 提炼能力清单与局限，并和参考文档逐项映射。
  - 摘要：区分“已经形成运行时能力”“只是局部基础设施”“暂未看到”的项目。
- [x] 做交叉校验并完成结果归档。
  - 摘要：回看测试与文档证据，整理最终判断。

## 结果复盘

- 已确认 auto-mind 当前最成熟的是请求级 Agent Runtime：`AgentService` 串起 MAIN / NAVI / NLG，`Context` 与 `CurrentTurnManager` 负责快照、回灌、drain、持久化。
- 已确认其 runtime sharing 主要落在三层：`RequestScope / RequestSession` 的请求级共享、`ContextSnapshot` 的模型输入共享、`EventBus + ObservationDrainCoordinator` 的执行结果回流共享。
- 已按 Claude Runtime 文档视角区分能力成熟度：Tool Runtime、Memory、Trace、Intervene 较完整；Skill / Plugin / Multi-Agent / Resume / Compact Restoration 仍偏局部基础或设计愿景。
- 已交叉检查 `.agents/` 文档、核心源码与对应测试入口，并执行 `git diff --check -- tasks/todo.md`，未发现新增格式问题。

---

# 任务：重排 6.4 Multi-Agent 与第 4 章 Tool 调度说明

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 优化目标：
  - 基于源码澄清 Multi-Agent 的使用时机到底是“模型判断”还是“运行时规则判断”；
  - 把 tool call 的并发 / 串行执行机制移回第 4 章，讲清动作级调度；
  - 让 6.4 的讲解更连续，不在“使用时机、收益、fork、记忆、控制点”之间来回跳。
- 关键源码结论：
  - `AgentTool` 是否被调用，主要由模型基于 system prompt / tool prompt 决定；运行时负责调用后的解析、过滤和执行；
  - `runTools -> partitionToolCalls` 会按 `isConcurrencySafe(...)` 把一组 tool calls 分成可并发批次与串行批次；
  - 复杂搜索、fork、自定义 Agent、verification 等使用建议来自提示词和 Agent 定义，而不是一个统一的硬编码复杂度分类器。

## 执行计划

- [x] 核验 `AgentTool` 的提示词、调用分流和 fork / fresh 路径。
  - 摘要：确认“什么时候使用 AgentTool”主要是模型在提示词引导下作出调用选择，运行时不做统一复杂度自动判定。
- [x] 核验 tool call 的并发 / 串行编排。
  - 摘要：确认模型可在同一轮返回多个 `tool_use`，运行时再按 `isConcurrencySafe(...)` 分批执行；并发批次有上限，串行批次会逐个更新上下文。
- [x] 更新第 4 章与 6.4 正文。
  - 摘要：第 4 章补动作级调度，第 6.4 改任务级派生。
- [x] 校验 Markdown 结构和 diff。
  - 摘要：确认章节边界更清楚、无重复解释、无格式问题。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

- 已在第 4 章补充 tool call 的两层机制：模型在提示词引导下提出并行工具意图，运行时再按 `isConcurrencySafe(...)` 做分批并发 / 串行调度。
- 已明确 `partitionToolCalls(...)` 的失败关闭策略、并发批次与串行批次的上下文更新差异，以及默认并发上限。
- 已重写 6.4 前半段，澄清 Multi-Agent 的使用时机主要由模型结合提示词、`AgentTool` prompt 和 `AgentDefinition.whenToUse` 作出判断，而不是统一硬编码复杂度规则。
- 已把 “tool 并发” 与 “Multi-Agent 任务级派生” 彻底拆开，章节职责更清楚。
- 已执行 `git diff --check`，未发现空白格式问题。

---

# 任务：补充 4.2 中 isConcurrencySafe 的判断逻辑

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 目标小节：`#### 4.2 并发不是模型说了算，最终由运行时分批调度`
- 优化目标：总结 `isConcurrencySafe(...)` 允许并发的几类判断模式，明确它不等价于“是否只读”。
- 关键源码结论：
  - 有些工具直接固定返回 `true`；
  - 有些工具基于具体输入动态判断，比如 `Bash` / `PowerShell` 依赖只读性分析；
  - 工具未声明、schema 失败或判断抛错时，运行时默认不可并发。

## 执行计划

- [x] 梳理源码中 `isConcurrencySafe(...)` 的三类模式。
  - 摘要：固定可并发、基于输入动态判断、失败关闭默认串行。
- [x] 更新第 4.2 正文。
  - 摘要：把“允许并发的判断逻辑”整理成表述清楚的总结段。
- [x] 校验 Markdown 结构和 diff。
  - 摘要：确认新增内容不和现有 4.2 冲突，也不把 `isConcurrencySafe` 写窄。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

- 已在 4.2 补充 `isConcurrencySafe(...)` 的三类判断模式：固定可并发、按输入动态判断、未声明或判断失败时默认串行。
- 已明确“是否只读”只是并发安全的重要依据之一，不等价于 `isConcurrencySafe(...)` 的完整语义。
- 已保留原有工具示例，并把总结前置为机制解释，阅读顺序更顺。
- 已执行 `git diff --check`，未发现空白格式问题。

---

# 任务：补充 6.4 Multi-Agent 的工程启发

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 目标位置：`#### 6.4 Multi-Agent：把复杂任务拆给受控子 Agent` 结尾，进入 6.5 之前。
- 优化目标：结合前面对 Multi-Agent 使用时机、fork / fresh、上下文隔离和受控边界的讲解，提炼对自研 Agent Runtime 的设计启发。
- 风格要求：讲原则，不重复前文机制细节；让读者看到“为什么这套设计值得借鉴”。

## 执行计划

- [x] 检查 6.4 结尾和 6.5 开头的衔接位置。
  - 摘要：6.4 已完整讲完机制，适合在 Recovery 之前补一段小结式工程启发。
- [x] 更新 6.4 正文。
  - 摘要：补充“什么时候值得派生子 Agent、如何限制边界、如何兼顾 cache 与上下文成本”等工程原则。
- [x] 校验 Markdown 结构和 diff。
  - 摘要：确认新增段落不重复已有内容，也不打断 6.5 开头。
- [x] 结果归档。
  - 摘要：记录最终改动和验证结果。

## 结果复盘

- 已在 6.4 结尾补充 Multi-Agent 的工程启发，强调动作并发与任务派生分层、策略层与执行层分离、受控执行单元、fork / fresh 分治。
- 新增段落承接 6.4 机制总结，并自然过渡到 6.5 的长任务可靠性主题。
- 已执行 `git diff --check`，清理掉 4 处行尾空格后通过。

---

# 任务：检查 Claude Runtime 文档面向 auto-mind 开发者的优化点

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 目标受众：`/Users/wangzhongbin/Documents/code/630/auto-mind` 的开发者。
- 分析目标：指出当前文档如果作为 auto-mind 团队学习/对标材料，哪些章节需要调整表达、补充映射或降低 Claude Code 特有实现的干扰。
- 输出要求：按优先级给出优化点，包含章节位置、问题、建议改法。
- 边界要求：本次只检查并给建议，不直接改正文；结论需结合 auto-mind 当前实现，不把未实现能力说成已有能力。

## 执行计划

- [x] 检查文档结构和 Claude Code 特有章节分布。
  - 摘要：重点看第 0-7 章是否缺少 auto-mind 映射入口。
- [x] 对照 auto-mind 当前主链路和上下文/执行文档。
  - 摘要：参考 `.agents/runtime-flow.md`、`context-and-rendering.md`、`execution-and-observation.md`。
- [x] 梳理面向 auto-mind 开发者的主要阅读障碍。
  - 摘要：区分术语不对齐、能力成熟度不对齐、章节缺少落地建议三类问题。
- [x] 交叉校验并输出优化建议。
  - 摘要：结合独立审阅视角，给出最终建议清单。

## 结果复盘

- 已确认目标文档当前没有出现 `auto-mind`、`AgentService`、`MAIN`、`NAVI`、`NLG`、`ProgressManager` 等对照词，缺少面向 auto-mind 开发者的概念翻译层。
- 已确认主要优化方向不是压缩 Claude Code 机制，而是在关键章节补充“auto-mind 对照 / 当前状态 / 不可直接类比”的提示。
- 已整理 7 个优先优化点：开篇映射表、MAIN/NAVI/NLG 任务链路、Context 与 Render 分层、ProgressManager/Executor/Observer 执行闭环、Intervene 安全边界、Skill/Plugin/Multi-Agent 状态标注、Memory/Trace/Recovery 拆分。

---

# 任务：隐性优化 Claude Runtime 文档的工程启发

## 需求规格

- 目标文档：`docs/claude-code-agent-runtime-sharing.md`
- 优化目标：在不显式提到 auto-mind 或“自研 Runtime”的前提下，让文档的工程启发更适合 Agent Runtime 开发者吸收。
- 内容约束：任何机制判断必须基于 `code-src` 下 Claude Code 源码，不基于推测或项目对照。
- 表达约束：不写“对照 auto-mind”“迁移到自研 Runtime”“你们当前已有/没有”等显性映射；只自然提炼 Claude Code 的设计原则。
- 验证要求：检查正文中不出现不该出现的显式措辞，并检查 Markdown diff。

## 执行计划

- [x] 从 `code-src` 校准 Agent Loop、Context、Tool、Safety、Extension、Recovery 的关键源码事实。
  - 摘要：已核对 `query.ts`、`QueryEngine.ts`、`toolOrchestration.ts`、`StreamingToolExecutor.ts`、`toolResultStorage.ts`、`autoCompact.ts`、`sessionRestore.ts`、`runAgent.ts`、plugin schema 与 hooks 配置等源码入口。
- [x] 调整各章“工程启发”和少量总结语。
  - 摘要：已增强状态迁移、运行现场、工具执行链、安全分层、Multi-Agent 派生和恢复能力的工程启发，保持 Claude Code 导读外观。
- [x] 清理显性项目映射措辞。
  - 摘要：已删除正文中“对我们设计自己的 Agent Runtime”这类显性映射表达，避免把文档写成项目对照材料。
- [x] 校验 diff、禁用词和 Markdown 结构。
  - 摘要：已检查目标正文不含 `auto-mind`、`自研`、`对照`、`迁移到`、`你们`、`我们设计自己的` 等显性措辞；代码围栏数量为偶数，`git diff --check` 通过。

## 结果复盘

- 已完成 `docs/claude-code-agent-runtime-sharing.md` 的隐性优化：不写项目对照，不提“自研 Runtime”，只在 Claude Code 导读语境下强化工程启发。
- 改动集中在 Agent Loop、Context、Tool、Safety、Multi-Agent、Recovery 的“工程启发”与少量总结语，机制判断均先用 `code-src` 源码入口校准。
- 已通过禁用词检查、Markdown 代码围栏检查和 `git diff --check`。
