# Claude Code 系统提示词架构文档整理

## 需求规格

- 将用户提供的系统提示词架构笔记整理为一篇独立 Markdown 文档。
- 章节标题使用更直白的表述，避免“缓存摩擦”“边界分治”等晦涩说法。
- 保留源码线索、核心流程、三条缓存路径、优先级链和工程启发。
- 在“系统提示词的完整构建流程”章节增加流程图，帮助读者理解 `getSystemPrompt` 的三条出口和默认路径。
- 不在页面直接输出全文，只写入 Markdown 文件。
- 验证环节包含：检查文件存在、检查章节结构、确认没有明显占位文本。
- 第二篇文档输出到 `tasks/claude-code-system-prompt-behavior-directives.md`。
- 第二篇主题为 Claude Code 系统提示词中的行为指令模式，按 6 种模式组织：极简主义、渐进式升级、可逆性意识、工具偏好、Agent 委托、数值锚定。
- 第二篇需保留源码线索，删除“暂时无法展示”等占位内容，并用 Mermaid 或 Markdown 表格替代原文中的图表占位。
- 第三篇文档输出到 `tasks/claude-code-tool-prompts-micro-harnesses.md`。
- 第三篇主题为工具提示词作为“微型驾驭器”，需要结合第二篇六种行为模式来解释 Bash/Edit/Read/Grep/Agent/Skill 六个工具的 description 设计。
- 后续所有文档整理必须保留原稿中的源码行号、源码片段和英文原代码；不得改动行号，不得删减原代码，不得翻译原代码。
- 后续所有文档整理必须检查目录结构是否清晰易读；如果原目录不利于理解，要主动优化目录层级和章节顺序。
- 后续所有文档整理必须串成一条非常明确的主线，避免叙事线散乱、重复解释同一个点，并主动修正容易让人误解的表述。
- 第四篇文档输出到 `tasks/claude-code-auto-compact.md`。
- 第四篇主题为自动压缩，按“触发判定 -> 熔断保护 -> 摘要提示词 -> 执行流程 -> PTL 重试 -> 用户可控性 -> 设计原则”组织。
- 第五篇文档输出到 `tasks/claude-code-post-compact-restoration.md`。
- 第五篇主题为压缩后的文件状态保留，按“压缩前快照 -> 文件恢复 -> 技能恢复 -> 刻意不恢复 -> Plan/PlanMode -> Delta 重播 -> 完整编排 -> 用户策略”组织。
- 第六篇文档输出到 `tasks/claude-code-microcompact.md`。
- 第六篇主题为微压缩，按“全量压缩的代价 -> 时间触发清理 -> 缓存微压缩 -> API Context Management -> 工具集合差异 -> 缓存中断协调 -> 子代理隔离 -> 用户策略”组织。
- 第七篇文档输出到 `tasks/claude-code-cache-architecture.md`。
- 第七篇主题为缓存架构与断点设计，按“前缀匹配约束 -> cache_control 断点 -> global/org/null 范围 -> TTL 锁存 -> beta header 锁存 -> thinking clear -> 全景架构 -> 用户策略”组织。
- 第八篇文档输出到 `tasks/claude-code-cache-break-detection.md`。
- 第八篇主题为缓存中断检测系统，按“为什么需要检测 -> 两阶段架构 -> PreviousState 快照 -> recordPromptState 请求前归因 -> checkResponseForCacheBreak 响应后确认 -> 解释引擎 -> 诊断输出 -> 清理机制 -> 用户策略”组织。
- 第九篇文档输出到 `tasks/claude-code-cache-optimization-patterns.md`。
- 第九篇主题为缓存优化模式，按“检测发现变化源 -> 分析变化为什么破坏前缀 -> 用记忆化/降精度/附件化/占位符/预算/会话缓存把动态变静态”组织。
- 第十篇文档输出到 `tasks/claude-code-tool-system.md`。
- 第十篇主题为工具系统，按“接口契约 -> 安全默认值 -> 注册过滤 -> MCP 融合 -> 结果预算 -> UI 渲染 -> 延迟加载 -> 模式沉淀”组织。
- 第十一篇文档输出到 `tasks/claude-code-agent-loop.md`。
- 第十一篇主题为 Agent Loop，作为全书锚点章节，按“为什么不是 REPL -> 状态机拓扑 -> 单轮迭代八阶段 -> 恢复/降级/中止 -> 序列图 -> 设计模式”组织。
- 第十二篇文档输出到 `tasks/claude-code-tool-execution-orchestration.md`。
- 第十二篇主题为工具执行编排，按“Agent Loop 中的工具阶段 -> 批次分区 -> runTools 调度 -> 单工具生命周期 -> 权限链 -> StreamingToolExecutor -> 结果预算 -> hooks 中断点 -> 模式沉淀”组织。
- 新一轮整理只覆盖用户指定范围：架构全景、启动链路、请求链路、Agent 循环、Prompt 系统、状态/会话/记忆、会话恢复、工具系统、Skill 系统、Plugin 系统、Hooks、多代理。
- 新一轮暂不整理权限系统、YOLO 分类器、提示注入防御、沙箱系统、Effort/Fast Mode/Thinking、Feature Flag 路线图等无关专题。
- 新一轮先输出规划文件 `tasks/claude-code-book-coverage-plan.md`，后续按规划分章节整理。

## 执行计划

- [x] 创建独立分享文档 `tasks/claude-code-agent-runtime-sharing.md`，资料来源为 `tasks/` 已有 ClaudeCode 文档和 ClaudeCode 源码。
- [x] 文档定位为面向已有业务 Agent 运行时经验的读者，不显式提 auto-mind，不写成源码锚点集合。
- [x] 只讲业务同学有体感的 ClaudeCode 核心模块：Agent Loop、Context/Prompt、工具系统、工具执行、上下文压缩恢复、Skill/Plugin、多代理。
- [x] 每个模块按“解决什么问题 -> 核心实现 -> 优势/启发”组织，不深入代码细节。
- [x] 验证分享稿目录清晰、主线连贯、无占位文本、没有跑偏到 UI/CLI/Ink 交互入口。
- [x] 按 1 小时分享结构复查 `tasks/claude-code-agent-runtime-sharing.md` 的章节清晰度，将章节收束为：任务推进、模型行为约束、工作现场、工具调用、安全防线、扩展与长期运行、总结。
- [x] 补齐独立话题：Prompt Engineering、Memory、Compression、Prompt Cache、Security / Prompt Injection，避免它们散落在其他章节里。
- [x] 验证新版分享稿不显式提 auto-mind，不跑偏到 UI/CLI/Ink，不写成源码锚点集合。
- [x] 根据用户反馈删除分享稿中的讲者旁白，避免出现“分享时可以这样讲”“推荐时间分配”“不需要讲太深”等面向讲者的表达。
- [x] 根据用户反馈补强分享稿中“Claude Code 怎么做”的示例化解释，尤其是系统提示词分段如何被组装和进入缓存分拣。
- [x] 验证新增示例仍然保持读者视角，不变成源码锚点集合，也不出现讲者旁白。
- [x] 分析 `/Users/wangzhongbin/Documents/code/630/auto-mind` 的项目结构、业务模块和 Agent 能力边界。
- [x] 梳理 auto-mind 中大家熟悉的模块：入口服务、上下文、LLM、工具执行、观察/进度、记忆、干预、评估与网关输出。
- [x] 对照 `tasks/` 中 ClaudeCode 文档，映射 ClaudeCode 在 Agent Loop、工具系统、上下文/压缩、Skill/Plugin、多代理、缓存、会话恢复上的做法。
- [x] 产出面向分享的讲解路径：先从 auto-mind 熟悉模块切入，再解释 ClaudeCode 的对应设计，最后列出 ClaudeCode 有而 auto-mind 暂缺的能力。
- [x] 验证分析结论有源码或文档依据，避免把未确认能力说成已有能力。
- [x] 创建《Plugin 系统》Markdown 文档。
- [x] 结合参考 `book/src/part6/ch22b.md` 与当前 `src/utils/plugins/*`、`src/commands/plugin/*` 源码校准插件链路。
- [x] 创建《Skill 系统》Markdown 文档。
- [x] 结合参考 `book/src/part6/ch22.md` 与当前 `src/skills/*`、`src/tools/SkillTool/*`、`src/services/skillSearch/*` 源码校准技能链路。
- [x] 根据用户反馈优化《Skill 系统》文档，改为“多来源 -> Command -> 轻量目录 -> SkillTool 调用 -> inline/fork -> contextModifier -> invokedSkills 恢复”的生命周期主线。
- [x] 删除/弱化按来源和执行细节堆叠的冗余结构，保留关键源码线索和核心代码片段。
- [x] 验证新版《Skill 系统》目录连贯性、占位文本、旧源码路径和代码块闭合。
- [x] 创建《多代理：Agent 派生》Markdown 文档。
- [x] 结合参考 `book/src/part6/ch20.md` 与当前 `src/tools/AgentTool/*` 源码校准 AgentTool / fork / runAgent 链路。
- [x] 检查三篇章节结构、占位文本、旧源码路径、代码块闭合和无关专题扩展。
- [x] 确认仓库中没有现成 docs/book 目录，选择 `tasks/claude-code-system-prompt-architecture.md` 作为输出路径。
- [x] 写入整理后的 Markdown 文档。
- [x] 检查文档章节结构与占位文本。
- [x] 补充结果复盘。
- [x] 在 1.10 节插入 `getSystemPrompt` 构建流程图。
- [x] 检查 Mermaid 代码块和章节结构。
- [x] 更新结果复盘。
- [x] 按“分段 -> 易变 -> 边界 -> API 分拣 -> 构建流程 -> 优先级”重新组织现有文档。
- [x] 收束重复内容，统一 `splitSysPromptPrefix` 三条路径的命名和顺序。
- [x] 检查章节编号、占位文本、Mermaid 代码块和关键术语。
- [x] 补充本轮结果复盘。
- [x] 创建第二篇行为指令 Markdown 文档。
- [x] 将 6 种行为模式统一整理为“定义 -> 源码例子 -> 为什么有效 -> 可复用模板”结构。
- [x] 检查第二篇章节结构、占位文本、Markdown 代码块闭合和 Mermaid 图表。
- [x] 补充第二篇结果复盘。
- [x] 创建第三篇工具提示词 Markdown 文档。
- [x] 将六个工具按“局部风险 -> 提示词机制 -> 复用第二篇行为模式 -> 可复用经验”整理。
- [x] 将原文占位图表改为 Mermaid 或 Markdown 表格。
- [x] 检查第三篇章节结构、占位文本、Markdown 代码块闭合和 Mermaid 图表。
- [x] 补充第三篇结果复盘。
- [x] 创建第四篇自动压缩 Markdown 文档。
- [x] 保留原稿中的源码行号和代码片段原文，不翻译、不删减、不改动。
- [x] 主动优化第四篇目录结构，确保目录清晰易读。
- [x] 将原文占位图表改为 Mermaid 或 Markdown 表格。
- [x] 检查第四篇章节结构、目录清晰度、占位文本、Markdown 代码块闭合和 Mermaid 图表。
- [x] 补充第四篇结果复盘。
- [x] 创建第五篇压缩后状态恢复 Markdown 文档。
- [x] 保留原稿中的源码行号和代码片段原文，不翻译、不删减、不改动。
- [x] 主动优化第五篇目录结构，确保目录清晰易读且主线明确。
- [x] 将原文占位图表改为 Mermaid 或 Markdown 表格。
- [x] 检查第五篇章节结构、目录清晰度、占位文本、Markdown 代码块闭合和 Mermaid 图表。
- [x] 补充第五篇结果复盘。
- [x] 创建第六篇微压缩 Markdown 文档。
- [x] 保留原稿中的源码行号和代码片段原文，不翻译、不删减、不改动。
- [x] 主动优化第六篇目录结构，确保目录清晰易读且主线明确。
- [x] 将原文占位图表改为 Mermaid 或 Markdown 表格。
- [x] 检查第六篇章节结构、目录清晰度、占位文本、Markdown 代码块闭合和 Mermaid 图表。
- [x] 补充第六篇结果复盘。
- [x] 创建第七篇缓存架构 Markdown 文档。
- [x] 保留原稿中的源码行号和代码片段原文，不翻译、不删减、不改动。
- [x] 主动优化第七篇目录结构，确保目录清晰易读且主线明确。
- [x] 将原文占位图表改为 Mermaid 或 Markdown 表格。
- [x] 检查第七篇章节结构、目录清晰度、占位文本、Markdown 代码块闭合和 Mermaid 图表。
- [x] 补充第七篇结果复盘。
- [x] 创建第八篇缓存中断检测 Markdown 文档。
- [x] 保留原稿中的源码行号和代码片段原文，不翻译、不删减、不改动。
- [x] 主动优化第八篇目录结构，确保目录清晰易读且主线明确。
- [x] 将原文占位图表改为 Mermaid 或 Markdown 表格。
- [x] 检查第八篇章节结构、目录清晰度、占位文本、Markdown 代码块闭合和 Mermaid 图表。
- [x] 补充第八篇结果复盘。
- [x] 创建第九篇缓存优化模式 Markdown 文档。
- [x] 保留原稿中的源码行号和代码片段原文，不翻译、不删减、不改动。
- [x] 主动优化第九篇目录结构，确保目录清晰易读且主线明确。
- [x] 将原文占位图表改为 Mermaid 或 Markdown 表格。
- [x] 检查第九篇章节结构、目录清晰度、占位文本、Markdown 代码块闭合和 Mermaid 图表。
- [x] 补充第九篇结果复盘。
- [x] 创建第十篇工具系统 Markdown 文档。
- [x] 保留原稿中的源码行号和代码片段原文，不翻译、不删减、不改动。
- [x] 主动优化第十篇目录结构，确保目录清晰易读且主线明确。
- [x] 将原文占位图表改为 Mermaid 或 Markdown 表格。
- [x] 检查第十篇章节结构、目录清晰度、占位文本、Markdown 代码块闭合和 Mermaid 图表。
- [x] 补充第十篇结果复盘。
- [x] 创建第十一篇 Agent Loop Markdown 文档。
- [x] 保留原稿中的源码行号和代码片段原文，不翻译、不删减、不改动。
- [x] 主动优化第十一篇目录结构，确保目录清晰易读且主线明确。
- [x] 将原文占位图表改为 Mermaid 或 Markdown 表格。
- [x] 检查第十一篇章节结构、目录清晰度、占位文本、Markdown 代码块闭合和 Mermaid 图表。
- [x] 补充第十一篇结果复盘。
- [x] 创建第十二篇工具执行编排 Markdown 文档。
- [x] 保留原稿中的源码行号和代码片段原文，不翻译、不删减、不改动。
- [x] 主动优化第十二篇目录结构，确保目录清晰易读且主线明确。
- [x] 将原文占位图表改为 Mermaid 或 Markdown 表格。
- [x] 检查第十二篇章节结构、目录清晰度、占位文本、Markdown 代码块闭合和 Mermaid 图表。
- [x] 补充第十二篇结果复盘。
- [x] 创建后续章节整理规划文档。
- [x] 明确本轮只整理用户指定范围，排除无关安全、推理控制和 Feature Flag 路线专题。
- [x] 将后续章节拆为入口主链路、状态会话记忆、扩展系统、多代理四批。
- [x] 为每个待整理章节指定建议输出文件和参考来源。
- [x] 创建《架构全景》Markdown 文档。
- [x] 结合参考 `book/src/part1/ch01.md` 与当前 `claude-code` 源码校准架构事实。
- [x] 只覆盖架构全景主线，不展开权限、沙箱、YOLO、Effort、Feature Flag 路线图等无关专题。
- [x] 主动优化《架构全景》目录结构，确保它能作为后续章节总地图。
- [x] 检查《架构全景》章节结构、占位文本、旧源码路径、代码块闭合和 Mermaid 图表。
- [x] 创建《启动链路》Markdown 文档。
- [x] 结合参考 `book/src/part1/ch01.md` 与当前 `cli.tsx/main.tsx/init.ts` 源码校准启动事实。
- [x] 按“轻量 bootstrap -> 完整 CLI -> init 环境准备 -> 模式分流 -> REPL/QueryEngine”组织《启动链路》。
- [x] 不展开无关安全、权限、沙箱、YOLO、Effort、Feature Flag 路线图专题。
- [x] 检查《启动链路》章节结构、占位文本、旧源码路径、代码块闭合和 Mermaid 图表。
- [x] 根据用户最新反馈，将《启动链路》进一步收束为“启动 Agent 的逻辑”，弱化 CLI 快速出口、预取、Keychain 等外围细节。
- [x] 按“Agent 会话需要哪些材料 -> 启动链路如何装配这些材料 -> 交给 REPL 或 QueryEngine”的主线重写。
- [x] 验证新版《启动链路》目录是否直指 Agent 启动，检查占位文本、旧源码路径、代码块闭合和冗余章节。
- [x] 创建《请求链路》Markdown 文档。
- [x] 结合当前 `QueryEngine.ts`、`query.ts`、`services/api/claude.ts`、`utils/api.ts` 源码校准请求构建事实。
- [x] 按“用户输入 -> 输入处理 -> 会话持久化 -> query() -> 上下文注入 -> API 参数构建 -> streaming 响应”组织《请求链路》。
- [x] 避免重复展开 Agent Loop 状态机、Prompt 具体段落、缓存优化细节和工具执行细节。
- [x] 检查《请求链路》章节结构、占位文本、旧源码路径、代码块闭合和 Mermaid 图表。
- [x] 根据用户反馈优化《Agent Loop》文档，降低晦涩度，改为先讲主分岔和单轮生命周期，再进入状态与恢复细节。
- [x] 保留关键源码线索，将过密源码摘录改成“源码锚点 + 人话解释”的阅读方式。
- [x] 验证新版《Agent Loop》目录连贯性、占位文本、代码块闭合和章节主线。
- [x] 创建《状态与会话总览》Markdown 文档。
- [x] 结合 `AppStateStore.ts`、`store.ts`、`bootstrap/state.ts`、`QueryEngine.ts`、`sessionStorage.ts`、`fileStateCache.ts` 源码校准状态分层。
- [x] 按“运行时 UI 状态 -> 会话级 QueryEngine 状态 -> 进程级 bootstrap state -> transcript 持久状态 -> 文件状态缓存 -> 压缩/恢复衔接”组织。
- [x] 避免重复展开自动压缩、压缩后恢复、微压缩和 CLAUDE.md 记忆注入细节。
- [x] 检查《状态与会话总览》章节结构、占位文本、旧源码路径、代码块闭合和 Mermaid 图表。

## 操作摘要

- 已查看当前任务目录与已有 todo 内容。
- 输出路径选择为 `tasks/claude-code-system-prompt-architecture.md`，避免引入新的源码目录结构。
- 已写入整理后的 Markdown 文档，采用“分段 -> 分界 -> 分拣 -> 构建流程 -> 最终优先级”的叙事顺序。
- 已检查章节标题，确认没有残留占位标记或不够直白的旧标题表达。
- 已定位 1.10 `getSystemPrompt` 章节，准备插入构建流程图。
- 已在 1.10 节开头插入 Mermaid 流程图，覆盖极简模式、PROACTIVE/KAIROS 分支和默认构建路径。
- 已检查 Mermaid 代码块位置，并确认没有新增占位文本或已排除的晦涩表达。
- 本轮将基于用户确认的组织建议继续整理同一文档，重点调整叙事顺序和减少重复说明。
- 已将文档重排为 1.1 到 1.10：前言、分段注册表、易变段落、静态/动态边界、API 前分拣、完整构建流程、最终优先级、缓存约束、工程模式、启发。
- 已统一 `splitSysPromptPrefix` 三条路径为“强制跳过 global cache / global cache + boundary / 默认 org cache”，并补充 1.5 到 1.6 的过渡说明。
- 已检查 Markdown 代码块数量为 86，成对闭合；章节编号连续；未发现残留占位文本或错误编号。
- 开始整理第二篇行为指令文档，输出路径定为 `tasks/claude-code-system-prompt-behavior-directives.md`。
- 已核对相关源码片段：`src/constants/prompts.ts`、`src/tools/BashTool/prompt.ts`、`src/tools/AgentTool/prompt.ts`、`src/tools/AgentTool/forkSubagent.ts`。
- 已写入第二篇行为指令文档，采用“行为控制层 -> 六种模式 -> 共同结构 -> 对我们的启发”的叙事顺序。
- 已将原文中的图表占位改为 Mermaid 决策矩阵和 Markdown 表格。
- 已检查第二篇章节结构，确认 2.1 到 2.9 连续；Markdown 代码块数量为 54，成对闭合。
- 开始整理第三篇工具提示词文档，输出路径定为 `tasks/claude-code-tool-prompts-micro-harnesses.md`。
- 已按 lessons 回顾整理规则，并核对相关源码片段：BashTool、FileEditTool、FileReadTool、GrepTool、AgentTool、SkillTool 的 prompt / tool 实现。
- 已写入第三篇工具提示词文档，采用“系统提示词全局行为层 -> 工具提示词局部行为层 -> 六工具拆解 -> 模式映射 -> 七条原则”的叙事顺序。
- 已将第三篇与第二篇的六种行为模式建立映射，说明每个工具 description 如何局部落地工具偏好、可逆性、渐进式升级、Agent 委托和数值锚定。
- 已将原文占位图表替换为 Mermaid 双层驾驭架构图、Bash 多命令决策树、SkillTool 三级截断流程图和六工具模式映射表。
- 已检查第三篇章节结构，确认 3.1 到 3.12 连续；Markdown 代码块数量为 62，成对闭合。
- 用户补充硬性要求：文档中涉及到的代码行数和原代码必须保留原样，不改动、不删减、不翻译。已同步到 `tasks/lessons.md` 和本 todo。
- 用户补充硬性要求：文档整理不只是组织叙事，还要检查目录结构是否清晰易读；若不满足则主动优化。已同步到 `tasks/lessons.md` 和本 todo。
- 开始整理第四篇自动压缩文档，输出路径定为 `tasks/claude-code-auto-compact.md`。
- 已核对相关源码片段：`src/services/compact/autoCompact.ts`、`src/services/compact/prompt.ts`、`src/services/compact/compact.ts`、`src/services/compact/grouping.ts`、`src/commands/compact/compact.ts`。
- 用户补充硬性要求：全文必须串成一条明确主线，层次清楚，避免重复解释，并修正容易误解的表述。已同步到 `tasks/lessons.md` 和本 todo。
- 已写入第四篇自动压缩文档，采用“触发判定 -> 熔断保护 -> 摘要提示词 -> 执行流程 -> PTL 重试 -> 每轮编排 -> 用户控制 -> 设计原则”的主线。
- 已将原文占位图表替换为 Mermaid 流程图、Markdown 表格和 ASCII 阈值示意。
- 已修正文档中容易误解的缓冲区表述，说明 3K manual compact blocking buffer 不参与自动压缩阈值主公式。
- 已完成第四篇结构验证：正文标题为 4.1 到 4.10，代码块数量 94 且成对闭合，未发现占位文本或错误源码路径。
- 开始整理第五篇压缩后状态恢复文档，输出路径定为 `tasks/claude-code-post-compact-restoration.md`。
- 已核对相关源码片段：`src/services/compact/compact.ts`、`src/utils/attachments.ts`、`src/tools/FileReadTool/FileReadTool.ts`。
- 已写入第五篇压缩后状态恢复文档，采用“压缩制造状态断层 -> 分通道选择性恢复 -> 用户如何利用恢复规则”的主线。
- 已将原文占位图表替换为 Mermaid 决策树、完整编排流程图和 Markdown 对照表。
- 已修正容易误解的表述：不再把 5K token 简化为固定代码行数，改为说明取决于语言和代码密度。
- 已完成第五篇结构验证：正文标题为 5.1 到 5.11，代码块数量 46 且成对闭合，未发现正文占位文本或错误源码路径。
- 开始整理第六篇微压缩文档，输出路径定为 `tasks/claude-code-microcompact.md`。
- 已核对相关源码片段：`src/services/compact/microCompact.ts`、`src/services/compact/timeBasedMCConfig.ts`、`src/services/compact/apiMicrocompact.ts`、`src/services/api/claude.ts`、`src/services/api/promptCacheBreakDetection.ts`。
- 已写入第六篇微压缩文档，采用“全量压缩代价高 -> 冷缓存时间清理 -> 热缓存 cache_edits -> API 声明式管理 -> 副作用协调”的主线。
- 已将原文占位图表替换为 Mermaid 流程图、序列表达的文字链路和 Markdown 对照表。
- 已修正重复叙事：将 cached MC 的子代理隔离详细解释统一放在 6.7，6.3 只保留启用条件。
- 已完成第六篇结构验证：正文标题为 6.1 到 6.11，代码块数量 72 且成对闭合，未发现正文占位文本或错误源码路径。
- 开始整理第七篇缓存架构与断点设计文档，输出路径定为 `tasks/claude-code-cache-architecture.md`。
- 用户中断第七篇整理后切换到第八篇缓存中断检测系统；当前以最新请求为准，开始整理第八篇，输出路径定为 `tasks/claude-code-cache-break-detection.md`。
- 已核对相关源码片段：`src/services/api/promptCacheBreakDetection.ts`、`src/services/api/claude.ts`、`src/services/compact/microCompact.ts`、`src/services/compact/compact.ts`、`src/services/compact/autoCompact.ts`、`src/tools/AgentTool/runAgent.ts`。
- 已写入第八篇缓存中断检测文档，采用“请求前留证据 -> 响应后判案 -> 解释原因 -> 输出诊断 -> 清理生命周期”的主线。
- 已将原文占位图表替换为 Mermaid 两阶段流程图和完整检测流程图，并用 Markdown 表格替代字段清单、阈值、归因规则。
- 已修正章节引用口径：按当前文档序列使用第 8 章标题，不保留原稿中第 14 章/第 13 章/第 15 章的交叉编号。
- 已完成第八篇结构验证：正文标题为 8.1 到 8.12，代码块数量 68 且成对闭合，未发现正文占位文本或错误源码路径。
- 继续完成此前中断的第七篇缓存架构与断点设计文档。
- 已核对相关源码片段：`src/utils/api.ts`、`src/services/api/claude.ts`、`src/bootstrap/state.ts`、`src/constants/prompts.ts`、`src/services/api/promptCacheBreakDetection.ts`。
- 已写入第七篇缓存架构文档，采用“前缀稳定约束 -> cache_control 断点 -> global/org/null 范围 -> TTL 锁存 -> beta header 锁存 -> thinking clear -> 检测系统验证”的主线。
- 已将原文占位图表替换为 Markdown 决策表、锁存状态 Mermaid 图和缓存架构全景 Mermaid 图。
- 已完成第七篇结构验证：正文标题为 7.1 到 7.11，代码块数量 64 且成对闭合，未发现正文占位文本或错误源码路径。
- 开始整理第九篇缓存优化模式文档，输出路径定为 `tasks/claude-code-cache-optimization-patterns.md`。
- 已核对相关源码片段：`src/constants/common.ts`、`src/tools/WebSearchTool/prompt.ts`、`src/tools/AgentTool/prompt.ts`、`src/utils/attachments.ts`、`src/tools/SkillTool/prompt.ts`、`src/tools/BashTool/prompt.ts`、`src/utils/toolSchemaCache.ts`、`src/utils/api.ts`、`src/constants/prompts.ts`。
- 已写入第九篇缓存优化模式文档，采用“检测发现变化源 -> 判断是否必须在前缀 -> 通过移动位置、降低频率、归一化、预算控制和精确缓存键消除抖动”的主线。
- 已将“7 个以上模式”整理为 8 个模式：日期记忆化、月度粒度、Agent 列表附件化、Skill 列表预算、`$TMPDIR` 占位符、条件段落省略、工具 Schema 会话级缓存、Schema 缓存键特化。
- 已将原文占位图表替换为 Markdown 模式总览表、常见陷阱表和 Mermaid 决策流程图。
- 已完成第九篇结构验证：正文标题为 9.1 到 9.14，代码块数量 70 且成对闭合，未发现正文占位文本、旧章节编号或错误源码路径。
- 开始整理第十篇工具系统文档，输出路径定为 `tasks/claude-code-tool-system.md`。
- 已核对相关源码片段：`src/Tool.ts`、`src/tools.ts`、`src/constants/toolLimits.ts`、`src/tools/GrepTool/GrepTool.ts`、`src/tools/BashTool/BashTool.tsx`、`src/tools/FileReadTool/FileReadTool.ts`、`src/utils/api.ts`、`src/tools/ToolSearchTool/prompt.ts`。
- 已写入第十篇工具系统文档，采用“工具不是数组，而是定义 -> 默认值 -> 注册过滤 -> MCP 融合 -> API Schema -> 调用执行 -> 预算控制 -> UI 渲染”的主线。
- 已将原文占位图表替换为 Tool 字段表、预算参数表、设计模式表和 Mermaid 管线图/渲染流程图。
- 已修正容易误解的章节引用：按当前整理序列使用第十篇编号，不保留原稿中的第 2 章/第 4 章/第 13 章交叉编号。
- 已完成第十篇结构验证：正文标题为 10.1 到 10.12，代码块数量 74 且成对闭合，未发现正文占位文本、旧章节引用或错误源码路径。
- 开始整理第十一篇 Agent Loop 文档，输出路径定为 `tasks/claude-code-agent-loop.md`。
- 已核对相关源码片段：`src/query.ts`、`src/utils/api.ts`、`src/services/api/claude.ts`、`src/utils/messages.ts`。
- 已写入第十一篇 Agent Loop 文档，采用“Agent Loop 不是 REPL -> State/transition 拓扑 -> 单轮八阶段 -> 恢复与降级 -> 序列图 -> 设计模式”的主线。
- 已将原文占位图表替换为 REPL 对比表、Continue/Terminal 表、Mermaid 状态机图和 Mermaid 序列图。
- 已修正旧章节引用口径：按当前整理序列使用第十一篇编号，并将跨章节定位改为按主题引用。
- 已完成第十一篇结构验证：正文标题为 11.1 到 11.17，代码块数量 104 且成对闭合，未发现正文占位文本或旧章节编号；原稿中的 `restored-src` 源码参考作为原始行号线索保留。
- 开始整理第十二篇工具执行编排文档，输出路径定为 `tasks/claude-code-tool-execution-orchestration.md`。
- 已核对相关源码片段：`src/services/tools/toolOrchestration.ts`、`src/services/tools/toolExecution.ts`、`src/services/tools/toolHooks.ts`、`src/services/tools/StreamingToolExecutor.ts`、`src/utils/toolResultStorage.ts`、`src/constants/toolLimits.ts`。
- 已写入第十二篇工具执行编排文档，采用“Agent Loop 工具阶段 -> 分区调度 -> 单工具生命周期 -> 权限链 -> 流式执行器 -> 结果预算 -> hooks 中断点 -> 模式沉淀”的主线。
- 已将原文占位图表替换为 Mermaid 分区图、生命周期图、权限链图、流式状态图和 Markdown 对照表。
- 已完成第十二篇结构验证：正文标题为 12.1 到 12.13，代码块数量 96 且成对闭合，未发现正文占位文本或旧章节引用；原稿中的 `restored-src` 源码参考作为原始行号线索保留。
- 已创建后续章节整理规划文档 `tasks/claude-code-book-coverage-plan.md`。
- 已将后续整理拆成四批：入口和请求主链路、状态会话记忆、扩展系统、多代理。
- 已明确暂不整理的无关参考文章：权限系统、YOLO 分类器、提示注入防御、沙箱系统、Effort/Fast Mode/Thinking、Feature Flag 路线图。
- 开始整理《架构全景》，输出路径定为 `tasks/claude-code-architecture-overview.md`。
- 已核对参考文章 `book/src/part1/ch01.md`，并结合当前源码校准入口事实：当前恢复版先走 `src/entrypoints/cli.tsx`，再进入完整 CLI `src/main.tsx`。
- 已核对相关源码片段：`src/entrypoints/cli.tsx`、`src/main.tsx`、`src/entrypoints/init.ts`、`src/replLauncher.tsx`、`src/components/App.tsx`、`src/state/store.ts`、`src/state/AppStateStore.ts`、`src/QueryEngine.ts`、`src/tools.ts`。
- 已写入《架构全景》文档，采用“轻量启动 -> 初始化 -> UI/AppState -> QueryEngine/Agent Loop -> Prompt/Tool/API/Memory -> 后续章节归位”的主线。
- 已完成《架构全景》结构验证：正文标题为 1.1 到 1.17，代码块数量 38 且成对闭合，未发现占位文本、旧源码路径或旧章节引用。
- 开始整理《启动链路》，输出路径定为 `tasks/claude-code-startup-chain.md`。
- 已核对《启动链路》相关源码片段：`src/entrypoints/cli.tsx`、`src/main.tsx`、`src/entrypoints/init.ts`、`src/replLauncher.tsx`、`src/components/App.tsx`、`src/QueryEngine.ts`。
- 已写入《启动链路》文档，采用“轻量 bootstrap -> 完整 CLI -> Commander preAction -> init -> headless QueryEngine / interactive REPL”的主线。
- 已明确启动链路边界：只解释进入可运行会话前的分流和初始化，不展开权限、沙箱、YOLO、推理控制等无关专题。
- 已完成《启动链路》结构验证：正文标题为 2.1 到 2.12，代码块数量 24 且成对闭合，未发现占位文本或旧源码路径。
- 开始整理《请求链路》，输出路径定为 `tasks/claude-code-request-pipeline.md`。
- 已核对《请求链路》相关源码片段：`src/QueryEngine.ts`、`src/utils/processUserInput/processUserInput.ts`、`src/query.ts`、`src/utils/api.ts`、`src/services/api/claude.ts`。
- 已写入《请求链路》文档，采用“用户输入 -> 输入处理 -> transcript 先落盘 -> query() -> 上下文注入 -> API 参数构建 -> streaming request”的主线。
- 已明确《请求链路》边界：不重复展开 Agent Loop 状态机、Prompt 具体段落、缓存优化模式和工具执行编排，只说明它们如何进入请求参数。
- 已完成《请求链路》结构验证：正文标题为 3.1 到 3.15，代码块数量 50 且成对闭合；唯一 `TODO` 命中来自保留的原始源码片段，按规则不改动。
- 开始整理《状态与会话总览》，输出路径定为 `tasks/claude-code-state-session-overview.md`。
- 已核对《状态与会话总览》相关源码片段：`src/state/store.ts`、`src/state/AppStateStore.ts`、`src/QueryEngine.ts`、`src/bootstrap/state.ts`、`src/utils/sessionStorage.ts`、`src/utils/fileStateCache.ts`。
- 已写入《状态与会话总览》文档，采用“AppState -> QueryEngine -> queryLoop State -> bootstrap state -> transcript -> FileStateCache -> 压缩/恢复衔接”的主线。
- 已明确本章边界：只做状态分层总览，不重复展开自动压缩、压缩后恢复、微压缩和 CLAUDE.md 记忆注入内部细节。
- 已完成《状态与会话总览》结构验证：正文标题为 4.1 到 4.14，代码块数量 42 且成对闭合，未发现占位文本、旧源码路径或无关专题命中。

## 当前任务：优化分享稿 5.1 Prompt Injection 是横切问题

### 需求规格

- 将 `tasks/claude-code-agent-runtime-sharing.md` 的 `5.1 Prompt Injection 是横切问题` 从简单来源列表，优化为“外部内容如何跨 Prompt / Context / Tool / Permission / Hooks / Sandbox 传播”的横切问题解释。
- 保持读者视角，不写讲者旁白，不展开成源码锚点集合。
- 只优化第 5 章安全主线中和 5.1 直接相关的内容，避免跑偏到完整权限系统、沙箱系统或专门的提示注入防御专题。
- 验证文档结构、代码块闭合、占位文本和跑偏词。

### 执行计划

- [x] 读取分享稿第 5 章现状，确认 5.1 当前表达缺口。
- [x] 重写 5.1，使其突出 Prompt Injection 的传播链路和横切防线。
- [x] 小幅校准第 5 章前后衔接，避免“只发生在工具调用前”这种局部化理解。
- [x] 验证 Markdown 结构、代码块闭合、占位文本和跑偏词。
- [x] 补充本轮操作摘要与结果复盘。

### 操作摘要

- 已读取 `tasks/claude-code-agent-runtime-sharing.md` 第 5 章，当前 5.1 主要是来源列表和一句定义，尚未充分解释为什么 Prompt Injection 是横切问题。
- 本轮设计选择：用“入口 -> 传播 -> 决策 -> 落地 -> 回填”的链路解释横切风险，再映射到 Prompt / Context / Tool / Permission / Hooks / Sandbox 六层防线。
- 已重写 `5.1 Prompt Injection 是横切问题`：新增传播链路、横切层级解释、关键词定义表和分层防线表。
- 已校准第 5 章前置总结，将 Prompt Injection 防御从“三个阶段”扩展为“外部内容进入、上下文呈现、模型决策、工具落地、结果回填”。
- 已完成验证：代码块围栏数量为 152，成对闭合；第 5 章标题层级连续；未发现占位文本、讲者旁白或 auto-mind/UI/CLI 跑偏词。

### 结果复盘

- 本轮完成 `tasks/claude-code-agent-runtime-sharing.md` 的 5.1 优化。新版不再只列 Prompt Injection 来源，而是解释“不可信内容如何沿 Agent Runtime 链路传播”，并把 Prompt / Context / Tool / Permission / Hooks / Sandbox 的责任边界放到同一张表中。
- 本任务为文档优化，不涉及代码运行测试；验证方式为 Markdown 结构检查、代码块闭合检查和占位/跑偏词检索。

## 当前任务：优化分享稿 5.2 Prompt 层

### 需求规格

- 将 `tasks/claude-code-agent-runtime-sharing.md` 的 `5.2 Prompt 层：先告诉模型边界` 从简单提醒列表，优化为“Prompt 层如何降低模型误读外部内容”的机制说明。
- 保持和 5.1 的横切链路衔接：5.2 只讲 Prompt 层，不抢 5.3 Context 边界和 5.4 Tool / Permission 落地校验的内容。
- 明确 Prompt 是前置行为约束，不是最终安全边界。
- 保持读者视角，不写讲者旁白，不展开成源码锚点集合。
- 验证文档结构、代码块闭合、占位文本和跑偏词。

### 执行计划

- [x] 读取第 5 章当前 5.2 和前后文，确认表达缺口。
- [x] 重写 5.2，补充 Prompt 层的指令优先级、数据身份标注和危险意图改写机制。
- [x] 检查 5.2 与 5.1 / 5.3 / 5.4 的边界，避免重复解释。
- [x] 验证 Markdown 结构、代码块闭合、占位文本和跑偏词。
- [x] 补充本轮操作摘要与结果复盘。

### 操作摘要

- 已读取第 5 章现状。当前 5.2 只有“系统提示词提醒模型区分几类内容”和“Prompt 不是最终防线”，缺少机制解释。
- 本轮设计选择：把 5.2 改为三层 Prompt 机制：先建立指令优先级，再把外部内容标为数据，最后把危险外部指令改写为“需要拒绝或报告的内容”，同时强调最终授权在运行时。
- 已重写 `5.2 Prompt 层：先告诉模型边界`：新增“模型看到一段文字时，应该把它当成指令还是数据”的核心问题，并展开指令优先级、数据身份、危险内容替代处理路径。
- 已检查 5.2 和 5.3 / 5.4 的边界：5.2 只讲模型理解阶段，Context 包装和运行时校验仍留给后续小节。
- 已完成验证：代码块围栏数量为 156，成对闭合；未发现占位文本、讲者旁白或 auto-mind/UI 跑偏词。

### 结果复盘

- 本轮完成 `tasks/claude-code-agent-runtime-sharing.md` 的 5.2 优化。新版把 Prompt 层定位为“解释器”：帮助模型先区分上级指令、用户任务和外部数据，但明确不承担最终授权。
- 本任务为文档优化，不涉及代码运行测试；验证方式为 Markdown 结构检查、代码块闭合检查和占位/跑偏词检索。

## 结果复盘

- 已完成文档整理，输出文件为 `tasks/claude-code-system-prompt-architecture.md`。
- 文档共 880 行，覆盖 `systemPromptSection`、`DANGEROUS_uncachedSystemPromptSection`、`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`、`splitSysPromptPrefix`、`getSystemPrompt`、`buildEffectiveSystemPrompt`、缓存设计约束和工程启发，并在 `getSystemPrompt` 章节补充了构建流程图。
- 验证方式为 Markdown 结构检查与占位/晦涩术语检索；本任务不涉及运行代码测试。
- 本轮已按新的叙事主线完成重组，当前文档共 901 行。验证方式包括章节结构检索、占位文本检索、错误编号检索、Markdown 代码块闭合检查；本任务仍不涉及运行代码测试。
- 第二篇已完成，输出文件为 `tasks/claude-code-system-prompt-behavior-directives.md`。文档共 746 行，覆盖极简主义指令、渐进式升级、可逆性意识、工具偏好引导、Agent 委托指引、数值锚定六种行为模式。验证方式包括章节结构检索、占位文本检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 第三篇已完成，输出文件为 `tasks/claude-code-tool-prompts-micro-harnesses.md`。文档共 732 行，覆盖 BashTool、FileEditTool、FileReadTool、GrepTool、AgentTool、SkillTool 六个工具的工具提示词设计，并与第二篇行为模式建立对应关系。验证方式包括章节结构检索、占位文本检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 第四篇已完成，输出文件为 `tasks/claude-code-auto-compact.md`。文档共 902 行，覆盖自动压缩阈值计算、前置守卫、连续失败熔断器、9 段压缩提示词、`compactConversation()` 执行流程、PTL 重试、`autoCompactIfNeeded()` 编排、手动 `/compact` 和用户控制策略。验证方式包括忽略代码块后的章节结构检查、占位文本检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 第五篇已完成，输出文件为 `tasks/claude-code-post-compact-restoration.md`。文档共 854 行，覆盖压缩前快照、文件恢复预算、Skill 重注入、`sentSkillNames` 的刻意不恢复、Plan/PlanMode 附件、Delta 完整重播、异步 Agent 附件、完整编排和用户策略。验证方式包括忽略代码块后的章节结构检查、占位文本检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 第六篇已完成，输出文件为 `tasks/claude-code-microcompact.md`。文档共 907 行，覆盖基于时间的微压缩、缓存微压缩、API Context Management、可压缩工具集差异、缓存中断检测协调、子代理隔离、用户策略和版本演化。验证方式包括忽略代码块后的章节结构检查、占位文本检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 第八篇已完成，输出文件为 `tasks/claude-code-cache-break-detection.md`。文档共 944 行，覆盖两阶段检测架构、`PreviousState` 快照、`recordPromptState()` 请求前变化检测、`checkResponseForCacheBreak()` 响应后 token 判定、Cache Deletion/Compaction 特殊路径、中断解释引擎、analytics/diff 输出、清理机制和用户策略。验证方式包括忽略代码块后的章节结构检查、占位文本检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 第七篇已完成，输出文件为 `tasks/claude-code-cache-architecture.md`。文档共 758 行，覆盖 Prompt Caching 前缀匹配模型、`cache_control`、`splitSysPromptPrefix()` 三条路径、global/org/null 缓存范围、1h TTL 锁存、beta header sticky-on 锁存、Thinking Clear 锁存、缓存架构全景和与第八篇检测系统的连接。验证方式包括忽略代码块后的章节结构检查、占位文本检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 第九篇已完成，输出文件为 `tasks/claude-code-cache-optimization-patterns.md`。文档共 768 行，覆盖 8 个缓存优化模式，并将它们统一到“识别变化源 -> 理解变化本质 -> 将动态变为静态”的决策框架中。验证方式包括章节结构检索、占位文本与旧编号检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 第十篇已完成，输出文件为 `tasks/claude-code-tool-system.md`。文档共 902 行，覆盖 Tool 接口契约、`buildTool()` 失败关闭默认值、`tools.ts` 三级注册过滤、MCP 工具融合、两级工具结果预算、三阶段 UI 渲染、ToolSearch 延迟加载和可复用设计模式。验证方式包括章节结构检索、占位文本与旧引用检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 第十一篇已完成，输出文件为 `tasks/claude-code-agent-loop.md`。文档共 1381 行，覆盖 `query()` / `queryLoop()` 入口、`State` 跨迭代状态、Continue/Terminal 状态拓扑、上下文预处理、API 消息标准化、流式响应、fallback、prompt-too-long / max_output_tokens 恢复、工具执行、附件注入、next_turn 和设计模式。验证方式包括章节结构检索、占位文本与旧编号检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 第十二篇已完成，输出文件为 `tasks/claude-code-tool-execution-orchestration.md`。文档共 1218 行，覆盖 `partitionToolCalls()`、`runTools()` 并发/串行调度、`runToolUse()` 单工具生命周期、Pre/PostToolUse hooks、权限决策链、`StreamingToolExecutor` 流式执行、Bash 错误级联、工具结果持久化、每消息聚合预算、空结果填充和版本演化信号。验证方式包括章节结构检索、占位文本与旧章节引用检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 后续章节整理规划已完成，输出文件为 `tasks/claude-code-book-coverage-plan.md`。规划覆盖用户指定的 12 个主题组，并拆为 21 个建议章节，其中已有 9 个完成章节，后续优先补“架构全景、启动链路、请求链路”。验证方式为对照参考文章标题和现有任务文档；本任务不涉及运行代码测试。
- 《架构全景》已完成，输出文件为 `tasks/claude-code-architecture-overview.md`。文档共 726 行，覆盖轻量 bootstrap、完整 CLI 入口、初始化环境、React Ink UI 外壳、AppState Store、QueryEngine 会话边界、Agent Loop 位置、Prompt 系统、工具与扩展层、状态/记忆/恢复层、API/缓存层和后续章节归位。验证方式包括章节结构检索、占位文本/旧路径检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 《启动链路》已完成，输出文件为 `tasks/claude-code-startup-chain.md`。文档共 549 行，覆盖 `cli.tsx` 轻量 bootstrap、快速出口、`main.tsx` 顶层预取、运行模式判定、Commander `preAction`、`init()`、headless `QueryEngine` 和 interactive REPL 分流。验证方式包括章节结构检索、占位文本/旧路径检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 《请求链路》已完成，输出文件为 `tasks/claude-code-request-pipeline.md`。文档共 1131 行，覆盖 `QueryEngine.submitMessage()`、`processUserInput()`、transcript 先落盘、本地命令短路、`query()` 上下文注入、API 层工具 schema 构建、消息标准化、system blocks、cache/beta 参数闭包和 streaming request。验证方式包括章节结构检索、占位文本/旧路径检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 《状态与会话总览》已完成，输出文件为 `tasks/claude-code-state-session-overview.md`。文档共 1085 行，覆盖 `AppState`、`createStore()`、`QueryEngine` 会话状态、`queryLoop` turn 级状态、`bootstrap/state.ts` 进程级单例、transcript JSONL、`FileStateCache`、压缩/恢复跨状态层位置和状态边界易混点。验证方式包括章节结构检索、占位文本/旧路径检索、Markdown 代码块闭合检查；本任务不涉及运行代码测试。
- 用户反馈《状态与会话总览》题目和结构仍显得有点乱；已重新按“状态活多久，就放在哪一层”的生命周期主线优化。具体调整：标题改为“从运行态到可恢复历史”；开头补充读法；将 `createStore()` 从独立层级改为 `AppState` 的实现细节；标题改为“运行时共享层 / 会话层 / Turn 层 / 进程层 / 持久层 / 文件状态缓存”；补强 transcript 与 FileStateCache 的区别、压缩/恢复的跨层定位。验证结果：正文标题为 4.1 到 4.14，代码块数量 44 且成对闭合，未发现占位文本、旧源码路径或无关专题命中。
- 已完成《Plugin 系统》《Skill 系统》《多代理：Agent 派生》三篇文档整理，分别输出到 `tasks/claude-code-plugin-system.md`、`tasks/claude-code-skill-system.md`、`tasks/claude-code-multi-agent-spawn.md`。
- 已按用户限定范围收束三篇内容：Plugin 只讲外部包进入运行时，Skill 只讲指令能力发现/加载/执行，多代理只讲 AgentTool 派生、fork、runAgent 和生命周期清理，不展开无关安全、沙箱、推理控制或 Feature Flag 路线专题。
- 已完成三篇统一验证：章节结构可检索；代码块分别为 Plugin 52、Skill 72、多代理 64，均成对闭合；未发现“暂时无法”、旧源码路径或无关专题关键词命中。
- 根据用户反馈统一收束《Plugin 系统》《Skill 系统》《多代理：Agent 派生》三篇：不再试图讲完整模块细节，改为每篇只保留一个核心问题和一条生命周期主线。
- Plugin 新版主线：plugin.json 声明能力 -> settings/marketplace 表达意图 -> reconciler 本地物化 -> pluginLoader 校验读取 -> component loaders 拆组件 -> refreshActivePlugins 交换进当前会话。文档从 1111 行压缩到 297 行。
- Skill 新版主线保持为：多来源 skill -> 统一 Command -> 轻量目录 -> SkillTool 调用 -> inline/fork -> contextModifier -> invokedSkills 恢复，并补充“不讲 Skill 编写教程、不展开内置 Skill 内容”的边界。文档保持 558 行，代码块闭合。
- 多代理新版主线：AgentTool 发起派生 -> AgentDefinition 决定角色 -> subagent_type 决定普通/fork -> runAgent 构造独立运行舱 -> 同步或后台回流 -> 生命周期清理。文档从 1354 行压缩到 354 行。
- 已完成三篇新版验证：Plugin 12 个二级标题、Skill 16 个二级标题、多代理 13 个二级标题；代码块围栏分别为 24、54、24 且成对闭合；未发现 `暂时无法`、`TODO`、`restored-src`、旧章节引用或跑偏提示词。
- 根据用户反馈重构《Plugin 系统》目录：从按模块堆叠改为按生命周期主线组织，形成“边界定义 -> 全链路总览 -> Manifest 声明 -> Marketplace Source -> Reconciler 物化 -> 本地缓存 -> 组件加载 -> Refresh 激活 -> 后台安装 -> 内置插件 -> 链路图/边界/模式”的顺序。
- 已保留 Plugin 文档的重点源码片段和核心内容，同时将 commands / skills、agents、hooks 收束为“加载阶段”的三个子节，减少与 manifest、reconciler、refresh 的重复解释。
- 已完成重构后验证：正文标题为 17.1 到 17.15，代码块数量 52 且成对闭合，未发现占位文本、旧源码路径或无关专题命中。
- 根据用户反馈重构《启动链路》文档：围绕“CLI 启动不能一上来加载所有东西，但进入会话前必须完成配置、认证、网络和状态准备”这一核心矛盾，改为“逐层闸门”主线。
- 已将《启动链路》的难懂术语改成人话解释：把 `main.tsx` 开头的启动动作解释为“提前启动慢读取，稍后在 preAction 等待”，避免用抽象术语压过读者理解。
- 已完成《启动链路》重构验证：正文标题为 2.1 到 2.13，代码块数量 46 且成对闭合，未发现占位文本、旧源码路径或无关专题命中。
- 根据用户最新反馈再次收束《启动链路》：旧版仍偏“CLI 启动优化”，新版改为“Agent 会话如何被启动”，围绕“运行材料”展开。
- 已删除/弱化外围细节：不再把快速出口、预取、Keychain/MDM、启动性能作为章节主线；只保留与 Agent 启动分流和运行材料装配直接相关的源码坐标。
- 已完成新版验证：文档压缩到 376 行，正文标题为 2.1 到 2.12，代码块围栏数量 42 且成对闭合；未发现 `暂时无法`、`TODO`、`restored-src`、`副作用`、`Keychain`、`MDM`、`快速出口`、`--version` 等跑偏信号。
- 根据用户反馈优化《Agent Loop》：旧版 1381 行过于像源码笔记，新版改为“先区分 REPL/query/queryLoop -> 再看 tool_use 主分岔 -> 再解释上下文、API、恢复、工具和 next_turn”的阅读路径。
- 已保留关键源码线索和核心代码片段，同时将大量细节改为源码锚点表格，避免源码块压过主线。
- 已完成新版验证：文档压缩到 512 行，正文标题为 11.1 到 11.16，代码块围栏数量 42 且成对闭合；未发现 `暂时无法`、`TODO`、错误旧章节引用或无关专题展开。
- 根据用户反馈优化《Skill 系统》：旧版 1124 行按来源和执行细节展开过多，新版改为“专业指令如何按需进入 Agent Loop”的生命周期主线。
- 已将主线收束为：多来源 skill -> 统一 Command -> 轻量目录 -> SkillTool 调用 -> inline/fork -> contextModifier -> invokedSkills 恢复。
- 已完成新版验证：文档压缩到 558 行，正文标题为 16.1 到 16.16，代码块围栏数量 54 且成对闭合；未发现 `暂时无法`、`TODO`、`restored-src`、旧章节引用或无关专题展开。
- 根据用户反馈更新《架构全景》核心流程图：删除图中的 UI 层、启动层、运行模式分流和 AppState 节点，只保留输入 -> QueryEngine -> Agent Loop -> Prompt / Tools / API / Memory -> 模型与结果回流的核心链路。
- 已完成 `auto-mind` 对标 ClaudeCode 的分享入口分析：`auto-mind` 适合从 Controller、AgentService、Context、LLMProvider、ProgressManager/Executor/Observer、MessageDispatcher 六个大家熟悉的模块切入；ClaudeCode 对应可讲 Agent Loop、工具系统、上下文压缩、Skill/Plugin、多代理、会话恢复与缓存。
- 验证方式：检查 `auto-mind` 核心源码文件与 `tasks/` 中 ClaudeCode 相关文档存在，并检索 `runReActLoop`、`buildSnapshotForRender`、`registerPlan`、`toolsCall`、`tool_use`、`自动压缩`、`AgentTool`、`ToolSearch` 等关键锚点；本任务为架构分析，不涉及运行代码测试。
- 已创建独立分享稿 `tasks/claude-code-agent-runtime-sharing.md`，从 ClaudeCode 视角串起 QueryEngine、Agent Loop、Context/Prompt、Tool System、Tool Execution、Compact/Resume、Recovery、Skill/Plugin、Multi-Agent。
- 分享稿不显式提 auto-mind，不写源码锚点集合；每个核心模块按“解决什么问题 -> 核心实现 -> 优势”组织，并补充完整链路、设计原则、推荐讲法和进一步阅读路径。
- 验证结果：文档 844 行，1 个一级标题、14 个二级标题，代码块围栏 54 个且成对闭合；未发现 `TODO`、`TBD`、`暂时无法`、`auto-mind`、`Ink`、`CLI`、`Controller`、`AgentService`、`源码锚点`、`对照` 等跑偏或占位信号。
- 根据用户连续反馈复查并重构分享稿章节：旧版按模块平铺，Prompt Engineering、Memory、Compression、Prompt Cache、Security / Prompt Injection 没有成为清晰一级话题；新版改为 1 小时分享结构：开场、任务推进、模型行为约束、模型工作现场、工具调用、安全多层防线、扩展与长期运行、完整链路、时间建议、进一步阅读。
- 新版明确 Prompt Engineering 属于“模型行为约束”，不放在任务推进；Prompt Injection 属于“安全横切问题”，不塞进工具执行；压缩、记忆、缓存统一归到“模型每轮基于什么工作”；工具章节只讲工具调用如何被接住。
- 验证结果：新版分享稿 796 行，1 个一级标题、10 个二级标题、28 个三级标题，代码块围栏 80 个且成对闭合；关键话题 `Prompt Injection`、`Prompt Cache`、`Compression`、`Memory` 均已覆盖；未发现占位文本、auto-mind 对照、UI/CLI/Ink 跑偏或源码锚点集合倾向。
- 根据用户反馈，分享前发给工程师的文档不能出现讲者备课口吻。已删除 `tasks/claude-code-agent-runtime-sharing.md` 中的“1 小时分享建议”和时间分配内容，将“值得带走的观点”统一改为“工程启发”，并把“可以这样理解 / 如果分享后需要继续展开”等表述改成读者视角正文。
- 根据用户反馈补强分享稿中“Claude Code 怎么做”的具体示例：Agent Loop 增加“修 bug 并验证”的 transition 示例；Prompt Engineering 增加系统提示词从逻辑数组、动态边界到 `cacheScope` block 的组装示例；Context/Memory/Compression、Tool Runtime、Security、Skill/Plugin/Multi-Agent 分别补充缩小版运行链路。
- 根据用户反馈将 `2.2 行为指令模式`、`2.3 工具提示词是微型驾驭器` 改为“核心设计 + 示例解释”结构：2.2 补充行为指令如何压制模型默认倾向，并用近似系统提示词示例逐行注释；2.3 补充工具 description 的局部控制器定位，并用 BashTool / GrepTool 近似结构逐行注释。
- 已完成 `3.1 Context：模型看到的是整理后的运行现场` 优化：按“核心设计 + 示例解释”结构重写，补充 `messagesForQuery` 从压缩边界、工具结果预算、history snip、microcompact、context collapse、autocompact 到 `appendSystemContext` / `prependUserContext` / `callModel` 的近似源码链路，并逐行加中文注释。验证结果：代码块闭合，关键实现角色齐全，未发现讲者旁白或跑偏词。
- 根据用户反馈补强第 3 章问题后的 Claude Code 对应方案：将“哪些状态要在 resume 后恢复”改为“会话中断或压缩后，哪些运行现场需要补回来”，并补充 session resume 与 post-compact restoration 两条恢复链路的方案表。
- 根据用户反馈，将两条恢复链路移入 `核心实现逻辑`，作为独立 `3.5 Recovery：会话和压缩后的现场如何补回来` 小节，而不是穿插进 Memory 或 Compression；文档中明确两条链路是恢复机制，不是两个保存文件。
- 已完成 `3.2 Memory：记忆不是单一存储` 优化：按“核心设计 + 示例解释”结构展开，使用 `QueryEngine.mutableMessages`、`readFileState`、`recordTranscript()`、`queryLoop State`、`FileStateCache`、`bootstrap STATE` 等实现角色说明 Claude Code 如何按状态寿命拆分记忆。验证结果：代码块闭合，关键实现角色齐全，未发现讲者旁白或跑偏词。
- 根据用户反馈进一步降低 `3.2 Memory` 理解门槛：先用工作记忆、情景记忆、语义/长期记忆、程序化记忆、外部工作现场记忆建立概念直觉，再映射到 Claude Code 的 `messagesForQuery`、transcript、`CLAUDE.md`、system/tool prompt、Skill、`FileStateCache` 等工程实现。
- 根据用户反馈继续顺滑 `3.2 Memory` 在“状态容器”之后的衔接：将后续问题改成“工作记忆 / 情景记忆 / 语义长期记忆 / 程序化记忆 / 外部现场记忆”对应的工程问题，避免概念分类和源码容器之间断层。
