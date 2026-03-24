# Code Hacker - VS Code Custom Agent

一个媲美 Claude Code 的 VS Code 自定义 Chat Agent，基于 **6 个 MCP Server** + VS Code 内建工具，覆盖文件操作、Git、代码分析、持久记忆、**代码审查**、**自动重构与结构化 Diff** 和网络访问。共 **48 个工具**，完整复刻 Claude Code 的核心能力并在代码审查维度超越它。

---

## 架构设计

### 设计理念

Claude Code 之所以强大，在于它不只是一个聊天窗口——它是一个拥有完整工具链的**自主编程 Agent**。它能读代码、改代码、搜代码、跑命令、管理 Git、记住上下文，形成一个闭环的开发工作流。

Code Hacker 的设计目标是：**在 VS Code Copilot Chat 中复刻这套闭环能力**。

核心思路：
1. **拆分关注点** — 将 Claude Code 的能力拆成 6 个独立的 MCP Server，每个 server 只做一件事
2. **组合大于继承** — 通过 chatmode 文件将多个 server 组装成一个完整的 Agent
3. **利用内建能力** — 网络访问直接复用 VS Code 内建的 `fetch`，不重复造轮子
4. **安全沙箱** — 每个 server 有独立的安全策略（路径检查、命令黑名单、文件白名单）
5. **超越而非模仿** — 代码审查和结构化 Diff 是 Claude Code 不具备的能力，基于 AST 级分析和 ydiff 算法

### 系统架构图

```
┌───────────────────────────────────────────────────────────────────────────┐
│                          VS Code Copilot Chat                             │
│                        ┌──────────────────────┐                           │
│                        │   Mode Selector      │                           │
│                        │  → "Code Hacker"     │                           │
│                        └──────────┬───────────┘                           │
│                                   │                                       │
│                        ┌──────────▼───────────┐                           │
│                        │  code-hacker          │                          │
│                        │  .chatmode.md         │  ← 系统提示词             │
│                        │                      │  ← 工具绑定               │
│                        │  tools:              │  ← 行为准则               │
│                        │   filesystem-command/*│                          │
│                        │   git-tools/*        │                           │
│                        │   code-intel/*       │                           │
│                        │   memory-store/*     │                           │
│                        │   code-review/*      │                           │
│                        │   code-refactor/*    │                           │
│                        │   fetch              │                           │
│                        └──────────┬───────────┘                           │
│                                   │                                       │
│                ┌──────────────────┼──────────────────┐                    │
│                │       MCP Protocol (stdio)          │                    │
│                │                                     │                    │
│  ┌─────────────▼─────────────┐  ┌────────────▼──────────────┐            │
│  │   filesystem.py           │  │   git_tools.py            │            │
│  │   (12 tools)              │  │   (11 tools)              │            │
│  │                           │  │                           │            │
│  │  • read/write/edit        │  │  • status/diff/log        │            │
│  │  • find/search            │  │  • add/commit             │            │
│  │  • execute_command        │  │  • branch/checkout         │            │
│  │  • directory ops          │  │  • stash/blame            │            │
│  └───────────────────────────┘  └───────────────────────────┘            │
│                                                                           │
│  ┌───────────────────────────┐  ┌───────────────────────────┐            │
│  │   code_intel.py           │  │  memory_store.py          │            │
│  │   (5 tools)               │  │  (7 tools)                │            │
│  │                           │  │                           │            │
│  │  • AST 分析               │  │  • save/get/search        │            │
│  │  • 符号提取                │  │  • list/delete            │            │
│  │  • 项目概览                │  │  • scratchpad             │  ┌────────┐│
│  │  • 引用查找                │  │                           │  │ fetch  ││
│  │  • 依赖图                  │  │  ┌───────────────┐        │  │(内建)  ││
│  └───────────────────────────┘  │  │.agent-memory/ │        │  │        ││
│                                  │  │  *.json       │        │  │网页抓取 ││
│                                  │  └───────────────┘        │  │API 调用 ││
│  ┌───────────────────────────┐  └───────────────────────────┘  └────────┘│
│  │   code_review.py          │  代码审查 (8 tools)                       │
│  │                           │                                           │
│  │  • review_project/file    │                                           │
│  │  • review_function        │                                           │
│  │  • health_score           │                                           │
│  │  • find_long/complex      │                                           │
│  │  • suggest_reorg          │                                           │
│  │  • review_diff_text       │                                           │
│  └───────────────────────────┘                                           │
│                                                                           │
│  ┌───────────────────────────┐                                           │
│  │   code_refactor.py        │  自动重构 + 结构化 Diff (4 tools)          │
│  │                           │                                           │
│  │  • auto_refactor          │   ┌─────────────────────────────┐         │
│  │  • ydiff_files            │   │  lib/                       │         │
│  │  • ydiff_commit           │   │  ├── refactor_auto.py       │         │
│  │  • ydiff_git_changes      │   │  └── ydiff_python.py        │         │
│  │                           │   │      (自包含，无外部依赖)     │         │
│  └───────────────────────────┘   └─────────────────────────────┘         │
└───────────────────────────────────────────────────────────────────────────┘
```

### 各 MCP Server 的职责

| Server | 文件 | 工具数 | 职责 | 设计原则 |
|--------|------|--------|------|----------|
| **filesystem-command** | `filesystem.py` | 12 | 文件 CRUD、精确编辑、文件搜索、命令执行 | Agent 的"手"——一切文件操作的基础。`edit_file` 复刻 Claude Code 的 Edit 工具 |
| **git-tools** | `git_tools.py` | 11 | Git 完整工作流 | 专用工具比通用命令更安全。LLM 直接调用结构化函数，不需记忆 git 语法 |
| **code-intel** | `code_intel.py` | 5 | 代码理解与分析 | AST 解析弥补 LLM 短板，`project_overview` 一键生成项目全景 |
| **memory-store** | `memory_store.py` | 7 | 跨会话持久记忆 | 结构化 JSON 存储 + 分类/标签/搜索。`scratchpad` 用于复杂推理 |
| **code-review** | `code_review.py` | 8 | 代码质量审查 | **独有能力**——Claude Code 没有。自包含 AST 分析引擎，量化代码质量、定位热点、生成重组建议 |
| **code-refactor** | `code_refactor.py` | 4 | 自动重构 + 结构化 Diff | **独有能力**——自动拆分长函数/大文件、ydiff AST 级 diff 生成交互式 HTML 报告 |

### 数据流：典型工作场景

**场景 A：代码修改**
```
用户: "把项目里所有的 print 改成 logging"

  ① project_overview(".")           → 了解项目结构
  ② search_files_ag("print(", "py") → 定位所有 print 语句
  ③ read_file_lines("app.py", 10, 25) → 确认上下文
  ④ edit_file("app.py", old, new)   → 精确替换
  ⑤ git_diff()                      → 确认修改结果
  ⑥ memory_save(...)                → 记住进度
```

**场景 B：AI 代码审查（Code Hacker 独有）**
```
用户: "审查这个项目的代码质量"

  ① health_score("/path/to/project")       → 快速获取 72/100 (B)
  ② review_project("/path/to/project")     → 完整扫描: 5个高危 + 12个中危
  ③ find_complex_functions(...)             → 定位 TOP 5 复杂函数
  ④ review_function("app.py", "process_data") → 深度分析 + 重构建议
  ⑤ auto_refactor(..., apply=False)         → 预览自动重构计划
  ⑥ auto_refactor(..., apply=True)          → 执行重构
  ⑦ ydiff_commit(".", "HEAD")              → 生成结构化 diff HTML 报告
```

**场景 C：Review AI 生成的代码**
```
用户: "帮我 review 这段 AI 生成的代码"

  ① review_diff_text(old_code, new_code)  → 对比新旧代码结构变化
     → 新增 3 函数, 删除 1 函数, 修改 2 函数
     → ⚠ process_all: 复杂度 8→15↑, 超标
     → ⚠ handle_request: 过长 (62行)
  ② review_function(...)                   → 深入分析问题函数
  ③ edit_file(...)                         → 修复问题
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
│  code_review.py:                        │
│    └─ 所有工具只读，不修改任何文件        │
│                                         │
│  code_refactor.py:                      │
│    ├─ auto_refactor 默认预览模式          │
│    ├─ 重构执行前可创建 .bak 备份          │
│    └─ ydiff 仅生成 HTML 报告，不改源码    │
│                                         │
└─────────────────────────────────────────┘
```

---

## 与 Claude Code 功能对比

### 逐项对比

| 能力维度 | Claude Code | Code Hacker | 胜出 |
|---------|------------|-------------|------|
| **文件读取** | `Read` — 支持行号、PDF、图片 | `read_file` + `read_file_lines` | Claude Code（多模态） |
| **文件写入** | `Write` — 创建或覆写 | `write_file` + `append_file` | 平手 |
| **精确编辑** | `Edit` — old→new 替换 | `edit_file` — 同样模式 | 平手 |
| **文件搜索** | `Glob` — ripgrep | `find_files` — pathlib.rglob | Claude Code（更快） |
| **内容搜索** | `Grep` — ripgrep | `search_files_ag` — Silver Searcher | 平手 |
| **命令执行** | `Bash` — 完整 shell、后台 | `execute_command` — 安全沙箱 | Claude Code（更强）/ Code Hacker（更安全） |
| **Git 操作** | `Bash` + git 命令 | 11 个专用 git 工具 | **Code Hacker** |
| **代码分析** | 靠 LLM 阅读 | AST 解析 + 符号提取 | **Code Hacker** |
| **项目理解** | `Agent` 多轮探索 | `project_overview` 一键 | **Code Hacker** |
| **依赖分析** | 无 | `dependency_graph` | **Code Hacker** |
| **持久记忆** | Markdown 文件系统 | JSON 结构化 + 分类/标签 | **Code Hacker** |
| **网络访问** | `WebFetch` + `WebSearch` | `fetch`（VS Code 内建） | Claude Code（多搜索） |
| **代码审查** | 无专用工具 | `review_project/file/function` + `health_score` | **Code Hacker 独有** |
| **自动重构** | 无 | `auto_refactor` — 自动拆分函数/文件 | **Code Hacker 独有** |
| **结构化 Diff** | 无（只有行级 diff） | `ydiff_files/commit/git_changes` — AST 级别 | **Code Hacker 独有** |
| **变更审查** | 无 | `review_diff_text` — 量化新旧代码差异 | **Code Hacker 独有** |
| **HTML 报告** | 无 | `generate_report` — 可视化质量报告 | **Code Hacker 独有** |
| **子代理** | `Agent` 并行派生 | 无 | Claude Code |
| **图片/PDF** | 支持 | 不支持 | Claude Code |
| **Notebook** | `NotebookEdit` | 不支持 | Claude Code |

### 优势总结

**Code Hacker 独有/更强 (9 项)：**
- **代码审查**：`review_project/file/function` 量化代码质量，Claude Code 完全没有
- **自动重构**：`auto_refactor` 自动拆分长函数和大文件，预览+执行模式
- **结构化 Diff**：`ydiff_files/commit` 基于 AST 的 diff，理解代码移动/重命名，Claude Code 只有行级 diff
- **变更审查**：`review_diff_text` 对比新旧代码结构变化，量化复杂度变化方向
- **健康评分**：`health_score` 一键给项目打分 0-100
- Git 操作：11 个结构化工具 vs. 手写 git 命令
- 代码分析：AST 精确解析 vs. LLM 阅读猜测
- 项目概览：一次调用 vs. 多轮探索
- 记忆系统：结构化 JSON vs. Markdown 文件

**Claude Code 更强 (5 项)：**
- 命令执行：完整 Bash shell + 管道 + 后台进程
- 子代理：并行派生探索代理
- 多模态：图片 + PDF 读取
- Web 搜索：搜索引擎检索
- Notebook 编辑

### 覆盖率

```
Claude Code 核心能力覆盖率:

  文件操作    ████████████████████  100%  (Read/Write/Edit/Glob/Grep)
  Git 操作    ████████████████████  100%  (甚至更细粒度)
  命令执行    ██████████████░░░░░░   70%  (缺少后台运行、管道)
  代码分析    ████████████████████  100%+ (AST 解析超越)
  持久记忆    ████████████████████  100%+ (结构化存储超越)
  代码审查    ████████████████████  ∞%   (Claude Code 没有此能力)
  结构化Diff  ████████████████████  ∞%   (Claude Code 没有此能力)
  自动重构    ████████████████████  ∞%   (Claude Code 没有此能力)
  网络访问    ██████████████░░░░░░   70%  (缺少搜索引擎)
  子代理      ░░░░░░░░░░░░░░░░░░░░    0%  (VS Code 不支持)
  多模态      ░░░░░░░░░░░░░░░░░░░░    0%  (MCP 限制)
  Notebook    ░░░░░░░░░░░░░░░░░░░░    0%  (可后续扩展)
  ─────────────────────────────────────────
  共同能力覆盖率   约 75%
  独有能力        +3 个维度超越 Claude Code
```

---

## 项目文件

```
.
├── filesystem.py              # MCP 1: 文件读写、编辑、搜索、命令执行 (12 tools)
├── git_tools.py               # MCP 2: Git 全套操作 (11 tools)
├── code_intel.py              # MCP 3: AST 分析、符号提取、依赖图 (5 tools)
├── memory_store.py            # MCP 4: 持久记忆 + 思考板 (7 tools)
├── code_review.py             # MCP 5: 代码质量审查 (8 tools)
├── code_refactor.py           # MCP 6: 自动重构 + 结构化 Diff (4 tools)
├── lib/
│   ├── __init__.py
│   ├── ydiff_python.py        # AST 结构化 diff 引擎
│   └── refactor_auto.py       # 自动重构引擎（函数拆分、文件拆分）
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
        "args": ["/你的绝对路径/vscode-custom-agents/filesystem.py"]
      },
      "git-tools": {
        "type": "stdio",
        "command": "python",
        "args": ["/你的绝对路径/vscode-custom-agents/git_tools.py"]
      },
      "code-intel": {
        "type": "stdio",
        "command": "python",
        "args": ["/你的绝对路径/vscode-custom-agents/code_intel.py"]
      },
      "memory-store": {
        "type": "stdio",
        "command": "python",
        "args": ["/你的绝对路径/vscode-custom-agents/memory_store.py"]
      },
      "code-review": {
        "type": "stdio",
        "command": "python",
        "args": ["/你的绝对路径/vscode-custom-agents/code_review.py"]
      },
      "code-refactor": {
        "type": "stdio",
        "command": "python",
        "args": ["/你的绝对路径/vscode-custom-agents/code_refactor.py"]
      }
    }
  }
}
```

> **替换 `/你的绝对路径/`** 为实际路径。所有 MCP server 均自包含，无外部依赖。

### 第二步：验证 MCP 连接

添加后，VS Code 底部状态栏会显示 MCP 服务器状态。确保 6 个服务器都显示为已连接。

如果未连接，检查：
- Python 路径是否正确（可能需要用 `python3` 替代 `python`）
- `mcp` 包是否已安装
- 文件路径是否为绝对路径

### 第三步：放置 Agent 文件

将 `code-hacker.chatmode.md` 放在你要使用的**项目根目录**下。

> 关键配置 — `tools` 字段必须使用 `服务器名/*` 通配符格式：
> ```yaml
> tools: ["filesystem-command/*", "git-tools/*", "code-intel/*", "memory-store/*", "code-review/*", "code-refactor/*", "fetch"]
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

### code-review (8 个工具)

| 工具 | 说明 |
|------|------|
| `review_project` | 扫描整个项目，健康评分 + 问题列表 + 重组建议 |
| `review_file` | 单文件分析，函数按复杂度排名 |
| `review_function` | 深度分析某个函数，给出具体重构建议 |
| `health_score` | 快速获取项目 0-100 健康评分 |
| `find_long_functions` | 查找最长函数排行 |
| `find_complex_functions` | 查找最高复杂度函数排行 |
| `suggest_reorg` | 文件重组建议（按命名模式和类分布） |
| `review_diff_text` | 对比新旧代码字符串，分析变更影响 |

### code-refactor (4 个工具)

| 工具 | 说明 |
|------|------|
| `auto_refactor` | 自动重构：拆分长函数和大文件（支持预览/执行） |
| `ydiff_files` | 结构化 AST 级别 diff：对比两个 Python 文件 |
| `ydiff_commit` | Git commit 结构化 diff，多文件 HTML 报告 |
| `ydiff_git_changes` | 对比任意两个 git ref 之间的结构化变更 |

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
