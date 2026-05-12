# Claude Code 文档整理规划：按系统主线补齐章节

## 规划目标

这份规划只覆盖用户指定的范围：

```text
架构全景 | 启动链路
请求链路 | Agent 循环
Prompt 系统
状态、会话、记忆 | 会话恢复
工具系统 | Skill 系统 | Plugin 系统 | Hooks
多代理
```

暂时不整理以下无关或旁支主题：

- 权限系统、YOLO / Auto 分类器、提示注入防御、沙箱系统；
- Effort / Fast Mode / Thinking；
- 未发布 Feature Flag 路线图；
- 安全、合规、远程执行等只在主线需要时短引用，不单独展开。

## 总体主线

全书需要按一条读者能顺着走的主线展开：

```text
Claude Code 是什么
  -> 如何启动
  -> 一次用户请求如何被构造成 API 请求
  -> Agent Loop 如何驱动模型、工具、上下文和恢复
  -> Prompt 如何约束模型行为
  -> 状态、会话和记忆如何让系统跨轮次持续工作
  -> 工具、Skill、Plugin、Hooks 如何扩展模型的能力边界
  -> 多代理如何把单 Agent 扩展成并行协作系统
```

因此后续整理不按参考文章原编号走，而按系统运行路径重新排序。

## 推荐章节序列

| 顺序 | 章节主题 | 状态 | 建议输出文件 | 主要来源 | 说明 |
|---:|---|---|---|---|---|
| 1 | 架构全景 | 待整理 | `tasks/claude-code-architecture-overview.md` | `book/src/part1/ch01.md`、现有已整理文章 | 作为全书地图，画出 CLI/UI/QueryEngine/Agent Loop/Prompt/Tool/State/API/MCP/Agent 的关系 |
| 2 | 启动链路 | 待整理 | `tasks/claude-code-startup-chain.md` | `book/src/part1/ch01.md` + 当前仓库源码 | 从 `cli.tsx`、`main.tsx`、`init.ts` 到 REPL / pipe mode / QueryEngine |
| 3 | 请求链路 | 待新写 | `tasks/claude-code-request-pipeline.md` | 当前已整理的 Agent Loop、Prompt、Tool、Cache 文章 + API 源码 | 专门讲一次 API 请求如何组装：system、messages、tools、betas、provider、cache_control、streaming |
| 4 | Agent 循环 | 已完成 | `tasks/claude-code-agent-loop.md` | 已整理 | 作为运行时锚点章节 |
| 5 | 系统提示词架构 | 已完成 | `tasks/claude-code-system-prompt-architecture.md` | 已整理 | Prompt 系统第一层：组装、边界、缓存和优先级 |
| 6 | 系统提示词行为指令 | 已完成 | `tasks/claude-code-system-prompt-behavior-directives.md` | 已整理，参考 `book/src/part2/ch06.md` | Prompt 系统第二层：行为模式 |
| 7 | 工具提示词 | 已完成 | `tasks/claude-code-tool-prompts-micro-harnesses.md` | 已整理 | Prompt 系统第三层：工具 description 的局部行为控制 |
| 8 | 状态与会话总览 | 待整理 | `tasks/claude-code-state-session-overview.md` | `book/src/part1/ch01.md`、`book/src/part6/ch24.md`、当前仓库源码 | 讲 AppState、bootstrap state、QueryEngine、消息历史、token 状态、文件状态 |
| 9 | CLAUDE.md 与记忆注入 | 待整理 | `tasks/claude-code-claudemd-memory.md` | `book/src/part5/ch19.md` | 讲用户指令如何作为覆盖层进入 prompt / context |
| 10 | 会话恢复与跨会话记忆 | 待整理 | `tasks/claude-code-session-recovery-memory.md` | `book/src/part6/ch24.md` | 讲 transcript persistence、JSONL、resume、session memory、跨会话记忆 |
| 11 | 自动压缩 | 已完成 | `tasks/claude-code-auto-compact.md` | 已整理 | 放入“状态、会话、记忆”大组，作为上下文容量管理 |
| 12 | 压缩后状态恢复 | 已完成 | `tasks/claude-code-post-compact-restoration.md` | 已整理 | 接在自动压缩后，解释压缩后的恢复通道 |
| 13 | 微压缩 | 已完成 | `tasks/claude-code-microcompact.md` | 已整理 | 上下文修剪的轻量层 |
| 14 | 工具系统 | 已完成 | `tasks/claude-code-tool-system.md` | 已整理 | 工具接口、注册、过滤、Schema、渲染 |
| 15 | 工具执行编排 | 已完成 | `tasks/claude-code-tool-execution-orchestration.md` | 已整理 | 工具执行、并发、权限链、流式、中断 |
| 16 | Skill 系统 | 待整理 | `tasks/claude-code-skill-system.md` | `book/src/part6/ch22.md` | 技能定义、发现、预算、调用、权限、生命周期 |
| 17 | Plugin 系统 | 待整理 | `tasks/claude-code-plugin-system.md` | `book/src/part6/ch22b.md` | 插件清单、安装、信任、市场、组件加载 |
| 18 | Hooks | 待整理 | `tasks/claude-code-hooks.md` | `book/src/part5/ch18.md` | Hook 事件、执行模型、退出码协议、配置合并、案例 |
| 19 | 多代理：Agent 派生 | 待整理 | `tasks/claude-code-multi-agent-spawn.md` | `book/src/part6/ch20.md` | AgentTool、标准子 Agent、Fork、Coordinator、工具池隔离 |
| 20 | 多代理：Teams 协作 | 待整理 | `tasks/claude-code-multi-agent-teams.md` | `book/src/part6/ch20b.md` | 队友 Agent、消息路由、TaskList、Claim Loop、团队记忆 |
| 21 | 多代理：远程规划 | 可选整理 | `tasks/claude-code-multi-agent-ultraplan.md` | `book/src/part6/ch20c.md` | 如果要覆盖远程多代理规划，再整理这一篇 |

## 分批执行计划

### 第一批：补全入口和请求主链路

这一批先补“读者进入系统的地图”，否则后续章节虽然细，但缺少总览坐标。

1. `架构全景`
2. `启动链路`
3. `请求链路`

完成标准：

- 每篇都有一张主流程图或架构图；
- 明确它和 Agent Loop、Prompt、Tool、State 的关系；
- 不展开无关安全/权限/沙箱主题，只保留必要引用。

### 第二批：补齐状态、会话、记忆

这一批把已有压缩文章连接到更大的状态系统里。

1. `状态与会话总览`
2. `CLAUDE.md 与记忆注入`
3. `会话恢复与跨会话记忆`

完成标准：

- 区分三类状态：运行时状态、会话历史、持久记忆；
- 解释压缩、恢复、记忆注入分别在状态系统中的位置；
- 避免和已完成的自动压缩 / 压缩后恢复 / 微压缩重复。

### 第三批：补齐扩展系统

这一批解释模型能力如何通过工具以外的机制扩展。

1. `Skill 系统`
2. `Plugin 系统`
3. `Hooks`

完成标准：

- Skill 讲“指令能力如何被发现和加载”；
- Plugin 讲“外部包如何进入系统”；
- Hooks 讲“用户如何拦截生命周期”；
- 不把权限系统、沙箱系统展开成独立章节。

### 第四批：补齐多代理

这一批解释单 Agent 如何扩展成协作系统。

1. `多代理：Agent 派生`
2. `多代理：Teams 协作`
3. `多代理：远程规划`（可选）

完成标准：

- 先讲 AgentTool / Fork / Fresh Agent 的基本委托；
- 再讲 Teams 的队友协作和任务调度；
- 最后视需要补远程规划；
- 多代理章节要与已有“Agent 委托提示词”和“Agent Loop”互相引用，避免重复。

## 不整理清单

以下参考文章暂时不进入本轮整理范围：

| 参考文章 | 原因 |
|---|---|
| `book/src/part5/ch16.md` 权限系统 | 用户本轮范围未包含独立权限系统；只在工具执行和 Hooks 中必要引用 |
| `book/src/part5/ch17.md` YOLO 分类器 | 属于权限/自动安全分类，不进入当前主线 |
| `book/src/part5/ch17b.md` 提示注入防御 | 属于安全专题，不进入当前主线 |
| `book/src/part5/ch18b.md` 沙箱系统 | 属于安全与隔离专题，不进入当前主线 |
| `book/src/part6/ch21.md` Effort / Fast Mode / Thinking | 属于模型推理控制专题，不进入当前主线 |
| `book/src/part6/ch23.md` Feature Flag 路线图 | 属于版本路线与未发布功能专题，不进入当前主线 |

## 每章固定验证项

后续每整理一章，都按已有规则验证：

- 保留原稿中的源码行号、源码片段和英文原代码；
- 检查目录结构是否清晰易读，不满足则主动优化；
- 全文必须串成一条明确主线；
- 删除 `暂时无法展示`、`TODO` 等占位；
- 用 Mermaid 或 Markdown 表格替代原图表占位；
- 检查章节编号、旧章节引用、代码块闭合；
- 更新 `tasks/todo.md` 的执行摘要和结果复盘。

