# Hermes-Agent Skill 系统运行逻辑详解

> 基于 `tools/skills_tool.py`、`agent/skill_commands.py`、`tools/skill_manager_tool.py`、`agent/prompt_builder.py`。

---

## 目录

1. [一眼看懂](#1-一眼看懂)
2. [源码地图](#2-源码地图)
3. [三层渐进式披露](#3-三层渐进式披露)
4. [发现与存储](#4-发现与存储)
5. [Slash 命令与 Cache 策略](#5-slash-命令与-cache-策略)
6. [`skill_manage` Agent 写技能](#6-skill_manage-agent-写技能)
7. [System Prompt 中的 Skill 索引](#7-system-prompt-中的-skill-索引)
8. [Curator 与用量遥测](#8-curator-与用量遥测)
9. [讲者展开点与常见误读](#9-讲者展开点与常见误读)

---

## 1. 一眼看懂

| 结论 | 含义 |
|---|---|
| Skill = 程序性记忆 | `SKILL.md` 记录「怎么做某类任务」，不是 `MEMORY.md` 式事实库 |
| 统一目录 | 运行时真相源：`~/.hermes/skills/`（bundled 种子、hub 安装、agent 创建共存） |
| 三层加载 | 列表元数据 → 全文 `skill_view` → 引用文件 `skill_view(..., file_path=)` |
| Slash = User 消息 | 技能内容以 **user message** 注入，**不**改 system，保护 prompt cache |
| `skill_manage` 改索引 | 成功后会 `clear_skills_system_prompt_cache` → **故意** 使 system 中 skill 索引 cache 失效 |
| 与 toolset `skills` | `skills_list` / `skill_view` / `skill_manage` 在 skills toolset 中暴露给模型 |

一句话：

```text
Skill = 磁盘上的 SKILL.md + 发现扫描 +（可选）system 索引 + 模型/slash 按需加载全文。
```

---

## 2. 源码地图

| 关注点 | 文件 | 读什么 |
|---|---|---|
| 发现与工具 | `tools/skills_tool.py` | `_find_all_skills`、`skills_list`、`skill_view` |
| Slash | `agent/skill_commands.py` | `scan_skill_commands`、`build_skill_invocation_message` |
| Agent CRUD | `tools/skill_manager_tool.py` | `skill_manage`、cache 清理 |
| System 索引 | `agent/prompt_builder.py` | `build_skills_system_prompt` (~992+) |
| 用量 | `tools/skill_usage.py` | Curator `bump_use` / `bump_patch` |
| 来源标记 | `tools/skill_provenance.py` | 后台 review vs 用户主动创建 |
| Hub / 可选包 | `tools/skills_hub.py` | official hub、`optional-skills` |
| 注册 | `toolsets.py` | `skills` toolset 工具列表 |

```1:13:tools/skill_manager_tool.py
"""
Skill Manager Tool -- Agent-Managed Skill Creation & Editing
...
Skills are the agent's procedural memory: they capture *how to do a specific
type of task* based on proven experience.
"""
```

---

## 3. 三层渐进式披露

```text
 Tier 1                    Tier 2                         Tier 3
 skills_list()             skill_view(name)               skill_view(name, file_path=)
 ─────────────             ─────────────────              ───────────────────────────
 name, description,        完整 SKILL.md +               references/, templates/,
 category（JSON）           linked_files 元数据             scripts/, assets/
        │                          │
        └──── build_skills_system_prompt() 仅嵌入 Tier-1 级元数据
```

| Tier | API / 入口 | 模型看到什么 |
|---|---|---|
| 1 | `skills_list` (`skills_tool.py` ~675) | 短描述 + 分类，用于选型 |
| 2 | `skill_view` (~850) | 正文 + 可链接文件列表 |
| 3 | `skill_view` + `file_path` | 单个参考文件内容 |

**插件技能：** `namespace:skill` 形式由 `skill_view` 解析到 `PluginManager`（~876–919）。

---

## 4. 发现与存储

### 4.1 目录

```86:90:tools/skills_tool.py
HERMES_HOME = get_hermes_home()
SKILLS_DIR = HERMES_HOME / "skills"
```

| 来源 | 说明 |
|---|---|
| Bundled | 安装时种子到 `~/.hermes/skills/` |
| Hub | `hermes skills install ...` |
| Agent / 用户 | `skill_manage` 写入同目录 |
| External dirs | `skills.external_dirs` 额外扫描 |

### 4.2 `_find_all_skills()` (~550–624)

- 遍历 `SKILLS_DIR` + external dirs，查找 `SKILL.md`
- 解析 frontmatter（首 4000 字符）：`name`、`description`、`platforms`、`metadata.hermes.*`
- 过滤：平台 gating、`skills.disabled`、`skills.platform_disabled.<platform>`
- 按 **skill name** 去重；**本地目录优先于 external**

### 4.3 SKILL.md  frontmatter（运行时使用）

| 字段 | 作用 |
|---|---|
| `name` / `description` | 列表与索引（description 有长度规范，见 AGENTS.md skill 标准） |
| `platforms` | OS 限制 `[macos]` 等 |
| `metadata.hermes.tags` / `category` | 分类与过滤 |
| `metadata.hermes.config` | 技能依赖的 `config.yaml` 键，setup 时提示 |

---

## 5. Slash 命令与 Cache 策略

### 5.1 扫描与注册

`scan_skill_commands()`（`skill_commands.py` ~263–326）：

- 生成 `_skill_commands["/slug"] → {name, description, skill_md_path, skill_dir}`
- Slug 规则：小写、空格/下划线 → 连字符
- 尊重 `skills.disabled` 与平台禁用表

`get_skill_commands()`：懒扫描；平台 scope 变化时重扫。

### 5.2 调用 → User 消息（关键）

```428:472:agent/skill_commands.py
def build_skill_invocation_message(cmd_key, user_instruction="", ...):
    ...
    activation_note = (
        f'[IMPORTANT: The user has invoked the "{skill_name}" skill, ...]'
    )
    return _build_skill_message(loaded_skill, skill_dir, activation_note, ...)
```

**设计意图（与 AGENTS.md Prompt Caching 政策一致）：**

- Slash 技能 **不** 写入 system prompt → 避免 mid-conversation 破坏 system 前缀缓存。
- 模型在下一 API 调用看到一条 **user** 消息，内含完整 skill 正文。

消费者：`cli.py`、`gateway/run.py`、`tui_gateway/server.py`、`COMMAND_REGISTRY` 等。

### 5.3 `reload_skills()`

`skill_commands.py` ~344–406：

- 重新 `scan_skill_commands`
- **故意不** 清除 `build_skills_system_prompt` 的 snapshot cache（~351–355）→ slash 重载不击穿 system cache

---

## 6. `skill_manage` Agent 写技能

Handler：`tools/skill_manager_tool.py` `skill_manage()` (~713+)

| action | 行为 |
|---|---|
| `create` | 写 `SKILL.md` + 可选目录结构 |
| `edit` | 全文替换 |
| `patch` | `old_string` / `new_string` |
| `delete` | 可带 `absorbed_into`（Curator） |
| `write_file` / `remove_file` | 技能目录下附属文件 |

**成功后（~765–788）：**

```python
clear_skills_system_prompt_cache(clear_snapshot=True)
```

→ system 里的 skill **索引** 变化，**Anthropic prompt cache 对 system 块失效**（预期行为）。

**守卫：**

- Pinned skills 不可 delete（Curator 策略）
- Agent-created 策略见 `tools/skills_guard.py` 与 schema

---

## 7. System Prompt 中的 Skill 索引

`build_skills_system_prompt()`（`prompt_builder.py` ~992+）：

- 在 **system** 中嵌入所有可用技能的 **name + description**（Tier-1）
- 引导模型：需要细节时调用 `skill_view`，不要猜
- 与 slash 注入全文 **互补**：索引常开，全文按需

| 路径 | 影响 system 索引 | 影响 prompt cache |
|---|---|---|
| Slash `/skill` | 否（仅 user 消息） | 保持 system 前缀稳定 |
| `skill_manage` 成功 | 是（清 cache） | system 块失效 |
| `reload_skills` | 否（不重算 system cache） | slash 映射更新即可 |

---

## 8. Curator 与用量遥测

| 组件 | 作用 |
|---|---|
| `tools/skill_usage.py` | `~/.hermes/skills/.usage.json`：use/patch 计数、last_activity、state |
| `agent/curator.py` | 后台归档 stale agent-created skills |
| Slash / `skill_view` | `bump_use(skill_name)`（`skill_commands.py` 454–458） |
| `skill_manage` + background review | `mark_agent_created` 仅 background；用户主动 create 不同 provenance |

**边界：** Curator 只动 `created_by: "agent"` 的技能；bundled / hub 技能不在自动归档范围。

---

## 9. 讲者展开点与常见误读

### 讲者展开点

- 对比 **Memory（声明性）vs Skill（程序性）** 再讲三层披露。
- 画两条路径：**用户 `/foo`（user 注入）** vs **模型 `skill_view`（tool 结果）**。
- 讲清 **为何 skill_manage 要 invalidate cache**（索引变了）。

### 常见误读

| 误读 | 事实 |
|---|---|
| Skill 在 git 仓库 `skills/` 下实时读 | 运行时是 `~/.hermes/skills/` |
| Slash 会改 system prompt | 注入 **user** message |
| `skills_list` 返回全文 | 仅元数据 |
| `reload_skills` 刷新 system 索引 cache | 明确 **不** 清 system prompt cache |
| 所有 skill 都可被 agent delete | pinned / bundled 有限制 |
| `optional-skills/` 默认加载 | 需 `hermes skills install official/...` |

---

## 相关文档

- [Runtime 导读](./hermes-agent-runtime-reader-guide.md) §2 — 记忆 vs 技能 vs 后台 review
- [上下文压缩](./hermes-agent-context-compression.md) §7 — cache 与 system 重建
- [主循环](./hermes-agent-main-loop.md) — `skill_manage` 后 `continue` 与 nudge
