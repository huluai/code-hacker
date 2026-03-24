# Code Hacker - VS Code Custom Agent

一个媲美 Claude Code 的 VS Code 自定义 Chat Agent，基于 4 个 MCP Server + VS Code 内建工具，覆盖文件操作、Git、代码分析、持久记忆和网络访问。共 **36 个工具**，完整复刻 Claude Code 的核心能力。

---

## 架构设计

### 设计理念

Claude Code 之所以强大，在于它不只是一个聊天窗口——它是一个拥有完整工具链的**自主编程 Agent**。它能读代码、改代码、搜代码、跑命令、管理 Git、记住上下文，形成一个闭环的开发工作流。

Code Hacker 的设计目标是：**在 VS Code Copilot Chat 中复刻这套闭环能力**。

核心思路：
1. **拆分关注点** — 将 Claude Code 的能力拆成 4 个独立的 MCP Server，每个 server 只做一件事
2. **组合大于继承** — 通过 chatmode 文件将多个 server 组装成一个完整的 Agent
3. **利用内建能力** — 网络访问直接复用 VS Code 内建的 `fetch`，不重复造轮子
4. **安全沙箱** — 每个 server 有独立的安全策略（路径检查、命令黑名单、文件白名单）

### 系统架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        VS Code Copilot Chat                         │
│                      ┌──────────────────────┐                        │
│                      │   Mode Selector      │                        │
│                      │  → "Code Hacker"     │                        │
│                      └──────────┬───────────┘                        │
│                                 │                                    │
│                      ┌──────────▼───────────┐                        │
│                      │  code-hacker          │                        │
│                      │  .chatmode.md         │  ← 系统提示词          │
│                      │                      │  ← 工具绑定            │
│                      │  tools:              │  ← 行为准则            │
│                      │   filesystem-command/*│                        │
│                      │   git-tools/*        │                        │
│                      │   code-intel/*       │                        │
│                      │   memory-store/*     │                        │
│                      │   fetch              │                        │
│                      └──────────┬───────────┘                        │
│                                 │                                    │
│              ┌──────────────────┼──────────────────┐                 │
│              │     MCP Protocol (stdio)            │                 │
│              │                                     │                 │
│  ┌───────────▼───────────┐  ┌──────────▼──────────┐                  │
│  │   filesystem.py       │  │   git_tools.py      │                  │
│  │   (12 tools)          │  │   (11 tools)        │                  │
│  │                       │  │                     │                  │
│  │  • read/write/edit    │  │  • status/diff/log  │                  │
│  │  • find/search        │  │  • add/commit       │                  │
│  │  • execute_command    │  │  • branch/checkout   │                  │
│  │  • directory ops      │  │  • stash/blame      │                  │
│  └───────────────────────┘  └─────────────────────┘                  │
│                                                                      │
│  ┌───────────────────────┐  ┌─────────────────────┐  ┌────────────┐ │
│  │   code_intel.py       │  │  memory_store.py    │  │ fetch      │ │
│  │   (5 tools)           │  │  (7 tools)          │  │ (VS Code   │ │
│  │                       │  │                     │  │  内建)      │ │
│  │  • AST 分析           │  │  • save/get/search  │  │            │ │
│  │  • 符号提取            │  │  • list/delete      │  │  网页抓取   │ │
│  │  • 项目概览            │  │  • scratchpad       │  │  API 调用   │ │
│  │  • 引用查找            │  │                     │  │            │ │
│  │  • 依赖图              │  │  ┌───────────────┐  │  └────────────┘ │
│  └───────────────────────┘  │  │.agent-memory/ │  │                 │
│                              │  │  *.json       │  │                 │
│                              │  └───────────────┘  │                 │
│                              └─────────────────────┘                 │
└──────────────────────────────────────────────────────────────────────┘
```

### 各 MCP Server 的职责

| Server | 文件 | 职责 | 设计原则 |
|--------|------|------|----------|
| **filesystem-command** | `filesystem.py` | 文件 CRUD、精确编辑、文件搜索、命令执行 | Agent 的"手"——一切文件操作的基础。`edit_file` 复刻 Claude Code 的 Edit 工具，支持精确字符串替换而非全文件重写 |
| **git-tools** | `git_tools.py` | Git 完整工作流 | 专用 Git 工具比通用 `execute_command` 更安全、更语义化。LLM 不需要记忆 git 命令语法，直接调用结构化的工具函数 |
| **code-intel** | `code_intel.py` | 代码理解与分析 | 弥补 LLM 的短板——LLM 擅长理解代码语义，但不擅长精确的结构分析。AST 解析提供精确的类/函数/导入信息，`project_overview` 一键生成项目全景 |
| **memory-store** | `memory_store.py` | 跨会话持久记忆 | 解决 LLM 会话间遗忘问题。结构化 JSON 存储支持分类、标签、搜索。`scratchpad` 用于复杂任务的中间推理 |

### 数据流：一个典型的代码修改任务

```
用户: "把项目里所有的 print 改成 logging"

  ① project_overview(".")
     → 了解项目结构，找到所有 Python 文件

  ② search_files_ag("print(", file_type="py")
     → 找到所有 print 语句及所在文件和行号

  ③ read_file_lines("app.py", start_line=10, end_line=25)
     → 读取上下文，确认哪些 print 需要改

  ④ edit_file("app.py", old_string='print("starting")', new_string='logger.info("starting")')
     → 精确替换，不影响其他代码

  ⑤ git_diff()
     → 确认修改结果

  ⑥ memory_save("refactor-print-to-logging", "已完成 app.py 的 print→logging 迁移", category="project")
     → 记住进度，下次会话可以继续
```

### 安全架构

```
┌─────────────────────────────────────────┐
│            安全检查层（每个 Server 独立）   │
├─────────────────────────────────────────┤
│                                         │
│  filesystem.py:                         │
│    ├─ 路径安全检查 (阻止 .. 遍历)         │
│    ├─ 文件类型白名单 (仅文本文件)          │
│    ├─ 文件大小限制 (10MB)                │
│    ├─ 命令黑名单 (rm/format/dd/...)      │
│    └─ 命令超时 (30s)                     │
│                                         │
│  git_tools.py:                          │
│    ├─ 所有操作只通过 git 子命令            │
│    ├─ 无 force push / reset --hard       │
│    └─ 命令超时 (30s)                     │
│                                         │
│  code_intel.py:                         │
│    ├─ 只读操作，不修改任何文件             │
│    └─ 搜索结果限制 (防内存溢出)           │
│                                         │
│  memory_store.py:                       │
│    ├─ 数据隔离 (.agent-memory/ 目录)      │
│    └─ JSON 格式，可审计                   │
│                                         │
└─────────────────────────────────────────┘
```

---

## 与 Claude Code 功能对比

### 逐项对比

| 能力维度 | Claude Code 工具 | Code Hacker 工具 | 对比说明 |
|---------|-----------------|-----------------|---------|
| **文件读取** | `Read` — 支持行号、offset/limit、PDF、图片 | `read_file` + `read_file_lines` | Claude Code 支持二进制文件（图片/PDF），Code Hacker 仅支持文本文件 |
| **文件写入** | `Write` — 创建或完整覆写 | `write_file` + `append_file` | 基本等价，Code Hacker 额外提供 append 操作 |
| **精确编辑** | `Edit` — old_string→new_string 替换 | `edit_file` — 同样的 old→new 模式 | 实现一致。都要求 old_string 唯一 |
| **文件搜索** | `Glob` — 基于 ripgrep 的文件名匹配 | `find_files` — Python pathlib.rglob | 功能等价，Claude Code 基于 ripgrep 更快 |
| **内容搜索** | `Grep` — ripgrep 全功能 | `search_files_ag` — Silver Searcher | 都支持正则、类型过滤、上下文行。ag 和 rg 性能接近 |
| **命令执行** | `Bash` — 完整 shell，支持后台运行 | `execute_command` — 安全沙箱执行 | Claude Code 更强大（支持管道、后台、超长超时）。Code Hacker 更安全（命令黑名单） |
| **Git 操作** | `Bash` + git 命令 | 11 个专用 git 工具 | **Code Hacker 更优**：结构化工具比手写 git 命令更安全、不易出错 |
| **代码分析** | 无专用工具，靠 LLM 阅读理解 | `analyze_python_file` + `extract_symbols` | **Code Hacker 更优**：AST 解析提供精确的结构信息，LLM 不需要逐行阅读 |
| **项目理解** | `Agent` 子代理探索 | `project_overview` 一键生成 | **Code Hacker 更优**：一次调用获得完整项目全景，无需多轮探索 |
| **依赖分析** | 无 | `dependency_graph` | Code Hacker 独有 |
| **符号引用** | `Grep` 文本搜索 | `find_references` | 功能接近，都是文本匹配（非语义分析） |
| **持久记忆** | 文件系统 memory 目录 + MEMORY.md 索引 | `memory_store` — JSON 结构化 | Claude Code 是 Markdown 文件，手动管理。Code Hacker 是结构化 JSON，支持分类/标签/搜索 |
| **思考板** | 无专用工具 | `scratchpad_write/read/append` | Code Hacker 独有，用于复杂推理 |
| **网络访问** | `WebFetch` + `WebSearch` | `fetch`（VS Code 内建） | Claude Code 额外支持搜索引擎。Code Hacker 仅支持直接 URL 抓取 |
| **子代理** | `Agent` — 可并行派生子代理 | 无 | Claude Code 独有，用于并行探索大型代码库 |
| **任务管理** | `TaskCreate/Update/List` | 无（可用 scratchpad 替代） | Claude Code 有内建任务系统，Code Hacker 用 scratchpad 近似 |
| **图片读取** | 支持 PNG/JPG 等 | 不支持 | Claude Code 是多模态的 |
| **Notebook** | `NotebookEdit` | 不支持 | Claude Code 可编辑 Jupyter Notebook |

### 优势总结

**Code Hacker 更强的地方：**
- Git 操作：11 个结构化工具 vs. 手写 git 命令，不会出现参数拼写错误
- 代码分析：AST 解析提供精确的类/函数结构，而不是让 LLM 猜
- 项目概览：一次调用 `project_overview` 获得完整信息，无需多轮对话
- 依赖图：`dependency_graph` 自动分析导入关系
- 记忆系统：结构化 JSON 存储，支持分类、标签、搜索，比文件系统更易管理

**Claude Code 更强的地方：**
- 命令执行：完整 Bash shell，支持管道、重定向、后台进程、超长运行
- 子代理：可并行派生多个探索代理，在大型代码库中效率更高
- 多模态：可读取图片和 PDF
- Web 搜索：除了 fetch URL，还能搜索引擎检索
- Notebook：可编辑 Jupyter Notebook
- 权限系统：更精细的权限控制和安全沙箱

### 覆盖率

```
Claude Code 核心能力覆盖率:

  文件操作    ████████████████████  100%  (Read/Write/Edit/Glob/Grep)
  Git 操作    ████████████████████  100%  (甚至更细粒度)
  命令执行    ██████████████░░░░░░   70%  (缺少后台运行、管道)
  代码分析    ████████████████████  100%+ (AST 解析超越 Claude Code)
  持久记忆    ████████████████████  100%+ (结构化存储超越 Claude Code)
  网络访问    ██████████████░░░░░░   70%  (缺少搜索引擎)
  子代理      ░░░░░░░░░░░░░░░░░░░░    0%  (VS Code 不支持)
  多模态      ░░░░░░░░░░░░░░░░░░░░    0%  (MCP 限制)
  Notebook    ░░░░░░░░░░░░░░░░░░░░    0%  (可后续扩展)
  ─────────────────────────────────────
  综合覆盖率   约 75%
```

---

## 项目文件

```
.
├── filesystem.py              # MCP 1: 文件读写、编辑、搜索、命令执行 (12 tools)
├── git_tools.py               # MCP 2: Git 全套操作 (11 tools)
├── code_intel.py              # MCP 3: AST 分析、符号提取、依赖图 (5 tools)
├── memory_store.py            # MCP 4: 持久记忆 + 思考板 (7 tools)
├── code-hacker.chatmode.md    # Agent 定义（系统提示词 + 工具绑定）
├── .vscode/
│   └── mcp.json               # MCP 服务器注册（仅供参考，实际需配置在用户设置中）
└── README.md
```

## 前置要求

- **VS Code** 1.99+
- **GitHub Copilot Chat** 扩展
- **Python** 3.10+
- **Git**

```bash
pip install mcp
```

可选（推荐）：安装 [The Silver Searcher](https://github.com/ggreer/the_silver_searcher) 以获得更快的代码搜索：

```bash
# macOS
brew install the_silver_searcher
# Ubuntu/Debian
sudo apt install silversearcher-ag
# Termux
pkg install the_silver_searcher
```

## 安装与配置

### 第一步：注册 MCP 服务器

MCP 服务器**不会自动启动**，必须在 VS Code 用户设置中手动注册。

打开 `settings.json`（`Ctrl+Shift+P` → `Preferences: Open User Settings (JSON)`），添加以下配置：

```json
{
  "mcp": {
    "servers": {
      "filesystem-command": {
        "type": "stdio",
        "command": "python",
        "args": ["/你的绝对路径/filesystem.py"]
      },
      "git-tools": {
        "type": "stdio",
        "command": "python",
        "args": ["/你的绝对路径/git_tools.py"]
      },
      "code-intel": {
        "type": "stdio",
        "command": "python",
        "args": ["/你的绝对路径/code_intel.py"]
      },
      "memory-store": {
        "type": "stdio",
        "command": "python",
        "args": ["/你的绝对路径/memory_store.py"]
      }
    }
  }
}
```

> **将 `/你的绝对路径/` 替换为实际路径**，例如 `/home/user/vscode-custom-agents/filesystem.py`

### 第二步：验证 MCP 连接

添加后，VS Code 底部状态栏会显示 MCP 服务器状态。确保 4 个服务器都显示为已连接。

如果未连接，检查：
- Python 路径是否正确（可能需要用 `python3` 替代 `python`）
- `mcp` 包是否已安装
- 文件路径是否为绝对路径

### 第三步：放置 Agent 文件

将 `code-hacker.chatmode.md` 放在你要使用的**项目根目录**下。

> 关键配置 — `tools` 字段必须使用 `服务器名/*` 通配符格式：
> ```yaml
> tools: ["filesystem-command/*", "git-tools/*", "code-intel/*", "memory-store/*", "fetch"]
> ```
> 其中 `fetch` 是 VS Code 内建工具，不需要额外配置。

### 第四步：开始使用

1. 在 VS Code 中打开包含 `code-hacker.chatmode.md` 的项目
2. 打开 Copilot Chat 面板（`Ctrl+Shift+I`）
3. 在顶部**模式选择器**中选择 **Code Hacker**
4. 开始对话

> **排查：** 如果模式选择器中没有 Code Hacker：
> - 确认 VS Code >= 1.99
> - 确认 `.chatmode.md` 文件在工作区根目录
> - 重启 VS Code

## 全部工具清单

### filesystem-command (12 个工具)

| 工具 | 说明 |
|------|------|
| `read_file` | 读取文件内容，支持 utf-8/gbk/gb2312 等编码 |
| `read_file_lines` | 读取指定行范围，适合大文件 |
| `write_file` | 写入文件 |
| `append_file` | 追加内容到文件 |
| `edit_file` | **精确字符串替换**（传入 old_string → new_string） |
| `find_files` | glob 模式递归搜索文件 |
| `search_files_ag` | 正则搜索文件内容（类似 ripgrep） |
| `list_directory` | 列出目录内容 |
| `get_file_info` | 文件详细信息（大小、时间、权限） |
| `create_directory` | 递归创建目录 |
| `get_current_directory` | 获取工作目录 |
| `execute_command` | 执行系统命令（已屏蔽危险命令） |

### git-tools (11 个工具)

| 工具 | 说明 |
|------|------|
| `git_status` | 工作区状态 |
| `git_diff` | 查看变更（支持 staged） |
| `git_log` | 提交历史 |
| `git_show` | 查看提交内容或特定版本的文件 |
| `git_branch` | 列出分支 |
| `git_create_branch` | 创建新分支 |
| `git_checkout` | 切换分支/恢复文件 |
| `git_add` | 暂存文件 |
| `git_commit` | 提交 |
| `git_stash` | 暂存管理（push/pop/list） |
| `git_blame` | 逐行追溯修改者 |

### code-intel (5 个工具)

| 工具 | 说明 |
|------|------|
| `analyze_python_file` | Python AST 深度分析（类、函数、导入、文档） |
| `extract_symbols` | 提取符号定义（Python/JS/TS/Java/Go/Rust） |
| `project_overview` | 项目全景（目录树、语言分布、入口点、配置） |
| `find_references` | 跨文件查找符号引用 |
| `dependency_graph` | 文件导入/被导入关系分析 |

### memory-store (7 个工具)

| 工具 | 说明 |
|------|------|
| `memory_save` | 保存记忆（支持分类和标签） |
| `memory_get` | 读取特定记忆 |
| `memory_search` | 搜索记忆（按关键词/分类/标签） |
| `memory_list` | 列出所有记忆 |
| `memory_delete` | 删除记忆 |
| `scratchpad_write/read/append` | 临时思考板（复杂推理用） |

### VS Code 内建

| 工具 | 说明 |
|------|------|
| `fetch` | 获取网页/API 内容 |

## 使用示例

```
你: 帮我分析这个项目的架构
→ project_overview → find_files → analyze_python_file → 输出分析报告

你: 把所有 print 语句改成 logging
→ search_files_ag 定位 → read_file_lines 确认上下文 → edit_file 逐个替换

你: 这个 bug 是谁引入的？
→ git_blame → git_show → 定位引入 bug 的提交和作者

你: 记住：这个项目的 API 要走 /api/v2 前缀
→ memory_save 持久化，下次会话自动回忆

你: 查一下 FastAPI 的中间件文档
→ fetch 获取文档内容并总结
```

## 自定义与扩展

### 添加文件类型白名单

编辑 `filesystem.py` 中的 `ALLOWED_EXTENSIONS`。

### 修改命令黑名单

编辑 `filesystem.py` 中的 `BLOCKED_COMMANDS`。

### 调整 Agent 行为

编辑 `code-hacker.chatmode.md` 中的系统提示词。

### 添加新的 MCP Server

1. 创建新的 `.py` 文件，使用 `FastMCP` 定义工具
2. 在 VS Code `settings.json` 中注册该 server
3. 在 `code-hacker.chatmode.md` 的 `tools` 中添加 `"新服务器名/*"`

## License

MIT
