# KANBAN 页面启动指南

## 核心结论

**KANBAN 没有独立的"页面"，而是通过 Web Dashboard 访问。**

启动步骤：
1. 启动 Web Dashboard
2. 在浏览器中访问 Dashboard
3. Dashboard 中包含 KANBAN 看板

## 启动方式

### 1. 基础启动（默认 Profile）

```bash
hermes dashboard
```

**默认行为**：
- 绑定到 `127.0.0.1:9119`
- 自动打开浏览器
- 使用默认 HERMES_HOME：`~/.hermes`

### 2. 指定 Hermes Home 启动

#### 方式 1：通过环境变量

```bash
# 使用自定义 HERMES_HOME
HERMES_HOME=/opt/hermes-data hermes dashboard

# 使用指定 Profile（自动设置 HERMES_HOME）
hermes -p myprofile dashboard

# Profile 对应的 HERMES_HOME 路径：
# ~/.hermes/profiles/<profile>/
```

#### 方式 2：通过 Profile 参数

```bash
# 使用 -p 参数指定 Profile
# HERMES_HOME 会自动设置为 ~/.hermes/profiles/<profile>/
hermes -p coder dashboard
hermes -p researcher dashboard
```

### 3. 高级启动选项

```bash
# 自定义端口和主机
hermes dashboard --port 8080 --host 0.0.0.0

# 允许公网访问（危险）
hermes dashboard --insecure --host 0.0.0.0

# 不自动打开浏览器
hermes dashboard --no-open

# 启用嵌入式 TUI（浏览器中的终端）
hermes dashboard --tui
# 或
HERMES_DASHBOARD_TUI=1 hermes dashboard

# 跳过 Web UI 构建（使用预构建的 dist）
hermes dashboard --skip-build
```

### 4. 管理运行中的 Dashboard

```bash
# 查看运行状态
hermes dashboard --status

# 停止所有运行中的 Dashboard
hermes dashboard --stop
```

## HERMES_HOME 详解

### 什么是 HERMES_HOME？

`HERMES_HOME` 是 Hermes Agent 的数据根目录，包含：
- `config.yaml` - 配置文件
- `kanban/` - KANBAN 数据库和看板
- `profiles/` - Profile 配置目录
- `skills/` - 用户技能
- `memory/` - 记忆存储
- `sessions/` - 会话数据
- `.env` - 环境变量

### 默认路径

| 场景 | HERMES_HOME 路径 |
|------|-----------------|
| 默认 | `~/.hermes` |
| 指定 Profile (`-p coder`) | `~/.hermes/profiles/coder/` |
| 自定义环境变量 | `$HERMES_HOME` |

### KANBAN 数据存储位置

```
<HERMES_HOME>/kanban/
├── boards/
│   ├── default/           # 默认看板
│   │   ├── kanban.db      # SQLite 数据库
│   │   ├── workspaces/    # 任务工作空间
│   │   └── logs/          # Worker 日志
│   └── my-project/        # 自定义看板
│       ├── kanban.db
│       ├── workspaces/
│       └── logs/
└── current                # 当前激活看板的符号链接
```

### 跨 Profile 的 KANBAN 隔离

```bash
# Profile A 的 KANBAN
hermes -p profile-a kanban list
# 使用: ~/.hermes/profiles/profile-a/kanban/

# Profile B 的 KANBAN
hermes -p profile-b kanban list
# 使用: ~/.hermes/profiles/profile-b/kanban/

# 完全隔离，互不影响
```

## 完整使用流程

### 首次使用

```bash
# 1. 初始化 KANBAN 数据库
hermes kanban init

# 2. 启动 Gateway（调度器）
hermes gateway start

# 3. 启动 Web Dashboard
hermes dashboard

# 4. 浏览器访问
# http://127.0.0.1:9119
```

### 指定 Profile 使用

```bash
# 1. 创建 Profile
hermes -p myproject setup

# 2. 初始化该 Profile 的 KANBAN
HERMES_HOME=~/.hermes/profiles/myproject hermes kanban init
# 或
hermes -p myproject kanban init

# 3. 启动 Gateway（使用该 Profile）
hermes -p myproject gateway start

# 4. 启动 Dashboard（使用该 Profile）
hermes -p myproject dashboard

# 5. 浏览器访问
# http://127.0.0.1:9119
```

## Dashboard 功能

### 主要功能页面

1. **Config** - 配置管理
   - API Keys
   - 模型选择
   - 工具集配置

2. **Sessions** - 会话历史
   - 浏览历史对话
   - 恢复会话
   - 导出对话

3. **Kanban** - 任务看板
   - 创建任务
   - 分配任务
   - 查看任务状态
   - 查看日志

4. **Memory** - 记忆管理
   - 查看全局记忆
   - 查看项目记忆
   - 搜索记忆

### KANBAN 页面 URL

启动 Dashboard 后，直接访问：
```
http://127.0.0.1:9119
```

KANBAN 看板是 Dashboard 的一部分，不需要单独的 URL。

## 常见问题

### Q1: 如何在不同端口启动？

```bash
hermes dashboard --port 8080
```

### Q2: 如何远程访问 Dashboard？

```bash
# 警告：这会暴露 API Keys
hermes dashboard --host 0.0.0.0 --insecure
```

### Q3: 如何在 Docker 中使用？

```dockerfile
ENV HERMES_HOME=/opt/hermes-data
ENV HERMES_PROFILE=myprofile

# 启动时
hermes -p $HERMES_PROFILE dashboard
```

### Q4: 如何切换 KANBAN 看板？

```bash
# 查看所有看板
hermes kanban boards list

# 切换看板
hermes kanban boards switch my-project

# 创建新看板
hermes kanban boards create my-project --name "My Project"

# 查看当前看板
hermes kanban boards show
```

### Q5: 如何为指定项目启动 Dashboard？

```bash
# 方式 1：使用 Profile
hermes -p myproject dashboard

# 方式 2：直接设置环境变量
HERMES_HOME=~/.hermes/profiles/myproject hermes dashboard

# 方式 3：切换到项目看板后再启动
hermes kanban boards switch myproject
hermes dashboard
```

## 命令速查表

| 命令 | 说明 |
|------|------|
| `hermes dashboard` | 启动 Web Dashboard |
| `hermes dashboard --port 8080` | 自定义端口 |
| `hermes -p <profile> dashboard` | 使用指定 Profile |
| `hermes dashboard --status` | 查看运行状态 |
| `hermes dashboard --stop` | 停止运行中实例 |
| `hermes kanban init` | 初始化 KANBAN 数据库 |
| `hermes kanban boards list` | 列出所有看板 |
| `hermes kanban boards switch <slug>` | 切换当前看板 |
| `hermes kanban create <title>` | 创建任务 |
| `hermes kanban list` | 列出任务 |
| `hermes gateway start` | 启动调度器 |

## 源码参考

### Dashboard 启动
- 文件：`hermes_cli/main.py`
- 函数：`cmd_dashboard()` (line 10429)
- 参数解析：line 13454-13506

### HERMES_HOME 预处理
- 文件：`hermes_cli/main.py`
- 函数：`_preload_profile_hermes_home()` (line 120)
- 说明：在模块导入前设置 HERMES_HOME

### KANBAN CLI
- 文件：`hermes_cli/kanban.py`
- 命令解析：`build_parser()` (line 191)
- 看板管理：`_dispatch_boards()` (line 956)

### HERMES_HOME 解析
- 文件：`hermes_constants.py`
- 函数：`get_hermes_home()` (line 43)
- 函数：`get_default_hermes_root()` (line 104)
