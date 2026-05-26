# Hermes 启动命令分析：`python -m hermes_cli.main dashboard`

## 命令分解

```bash
cd /Users/wangzhongbin/Documents/code/fangzhen/hermes-agent
.venv/bin/python -m hermes_cli.main dashboard --port 9119
```

### 各部分说明

| 部分 | 说明 |
|------|------|
| `cd ...` | 切换到 hermes-agent 源码目录 |
| `.venv/bin/python` | 使用虚拟环境中的 Python |
| `-m hermes_cli.main` | 运行 hermes_cli.main 模块 |
| `dashboard` | dashboard 子命令 |
| `--port 9119` | 指定端口 9119 |

## HERMES_HOME 解析流程

### 源码分析（`hermes_cli/main.py`）

```python
# Line 120-205: _apply_profile_override() 函数

def _apply_profile_override() -> None:
    """在模块导入前预解析 --profile/-p 并设置 HERMES_HOME"""
    argv = sys.argv[1:]
    profile_name = None

    # 1. 检查是否有 -p / --profile 参数
    for i, arg in enumerate(argv):
        if arg in {"--profile", "-p"} and i + 1 < len(argv):
            profile_name = argv[i + 1]
            break
        elif arg.startswith("--profile="):
            profile_name = arg.split("=", 1)[1]
            break

    # 2. 如果没有 -p 参数，检查 active_profile 文件
    if profile_name is None:
        active_path = get_default_hermes_root() / "active_profile"
        if active_path.exists():
            name = active_path.read_text().strip()
            if name and name != "default":
                profile_name = name

    # 3. 如果找到了 profile，设置 HERMES_HOME
    if profile_name is not None:
        hermes_home = resolve_profile_env(profile_name)
        os.environ["HERMES_HOME"] = hermes_home

_apply_profile_override()  # Line 205: 立即执行
```

### 你的命令的解析过程

```bash
# 原始命令
.venv/bin/python -m hermes_cli.main dashboard --port 9119

# sys.argv = [
#   ".../python",
#   "dashboard",
#   "--port", 
#   "9119"
# ]

# _apply_profile_override() 执行：
# 1. 检查 -p/--profile 参数 → 未找到
# 2. 检查 ~/.hermes/active_profile 文件 → 不存在或为 "default"
# 3. profile_name = None
# 4. 不设置 HERMES_HOME 环境变量
```

### 最终 HERMES_HOME

```python
# hermes_constants.py::get_hermes_home()

def get_hermes_home() -> Path:
    """返回 Hermes 主目录（默认：~/.hermes）"""
    # 1. 检查 ContextVar 覆盖（未使用）
    # 2. 检查 HERMES_HOME 环境变量（未设置）
    # 3. 返回默认值
    return Path.home() / ".hermes"
```

**结论：你的命令使用的 HERMES_HOME 是 `~/.hermes`**

## KANBAN 数据存储位置

```
~/.hermes/kanban/
├── boards/
│   ├── default/           # 默认看板
│   │   ├── kanban.db      # KANBAN 数据库
│   │   ├── workspaces/    # 任务工作空间
│   │   └── logs/          # Worker 日志
│   └── current            # → default (符号链接)
└── kanban.db             # 旧版兼容（可能不存在）
```

## 完整路径

| 项目 | 路径 |
|------|------|
| HERMES_HOME | `/Users/wangzhongbin/.hermes/` |
| KANBAN DB | `/Users/wangzhongbin/.hermes/kanban/boards/default/kanban.db` |
| 配置文件 | `/Users/wangzhongbin/.hermes/config.yaml` |
| 环境变量 | `/Users/wangzhongbin/.hermes/.env` |
| 技能目录 | `/Users/wangzhongbin/.hermes/skills/` |
| 会话数据 | `/Users/wangzhongbin/.hermes/sessions/` |

## 如何验证

```bash
# 方法 1: 查看 kanban boards list
.venv/bin/python -m hermes_cli.main kanban boards list

# 方法 2: 查看 current board
.venv/bin/python -m hermes_cli.main kanban boards show

# 方法 3: 查看 HERMES_HOME
.venv/bin/python -c "from hermes_constants import get_hermes_home; print(get_hermes_home())"
```

## 如果想使用其他 HERMES_HOME

### 方式 1：使用 Profile

```bash
# 创建并使用 Profile
.venv/bin/python -m hermes_cli.main -p myproject dashboard --port 9119

# HERMES_HOME 会自动变为：
# ~/.hermes/profiles/myproject/
```

### 方式 2：使用环境变量

```bash
# 设置自定义 HERMES_HOME
HERMES_HOME=/custom/path .venv/bin/python -m hermes_cli.main dashboard --port 9119

# 或先导出环境变量
export HERMES_HOME=/custom/path
.venv/bin/python -m hermes_cli.main dashboard --port 9119
```

### 方式 3：设置 active_profile

```bash
# 设置默认 Profile
.venv/bin/python -m hermes_cli.main profile use myproject

# 之后所有命令都自动使用该 Profile
.venv/bin/python -m hermes_cli.main dashboard --port 9119
# 等价于：
# HERMES_HOME=~/.hermes/profiles/myproject .venv/bin/python -m hermes_cli.main dashboard --port 9119
```

## 多项目隔离示例

```bash
# 项目 A
hermes -p project-a kanban create "Implement feature X"
# 使用: ~/.hermes/profiles/project-a/kanban/

# 项目 B
hermes -p project-b kanban create "Fix bug Y"
# 使用: ~/.hermes/profiles/project-b/kanban/

# 完全隔离，互不影响
```

## 与 `hermes` CLI 的对比

| 命令 | HERMES_HOME | 说明 |
|------|-------------|------|
| `hermes dashboard` | `~/.hermes/` | 使用安装版命令 |
| `python -m hermes_cli.main dashboard` | `~/.hermes/` | 使用源码直接运行 |
| `hermes -p myproject dashboard` | `~/.hermes/profiles/myproject/` | 使用 Profile |
| `HERMES_HOME=/path ...` | `/path` | 自定义路径 |

**你的命令 (`python -m hermes_cli.main dashboard`) 等价于 `hermes dashboard`**

## 注意事项

1. **源码运行 vs 安装运行**
   - 你的命令直接运行源码，适合开发调试
   - 生产环境建议使用 `pip install` 后的 `hermes` 命令

2. **端口冲突**
   - 默认端口是 9119
   - 如果端口被占用，使用 `--port` 指定其他端口
   - 多个 Dashboard 实例需要使用不同端口

3. **KANBAN 数据共享**
   - 同一个 HERMES_HOME 下的 KANBAN 数据是共享的
   - 不同 Profile 的 KANBAN 数据是隔离的
