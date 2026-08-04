# Trae Agent Code Wiki

> 项目地址：https://github.com/bytedance/trae-agent
> Tech Report: https://arxiv.org/abs/2507.23370
> 许可证：MIT

---

## 目录

1. [项目概述](#1-项目概述)
2. [技术栈](#2-技术栈)
3. [项目结构](#3-项目结构)
4. [整体架构](#4-整体架构)
5. [主要模块职责](#5-主要模块职责)
6. [关键类与函数说明](#6-关键类与函数说明)
7. [依赖关系](#7-依赖关系)
8. [配置系统](#8-配置系统)
9. [项目运行方式](#9-项目运行方式)
10. [数据流与执行流程](#10-数据流与执行流程)

---

## 1. 项目概述

**Trae Agent** 是一个基于 LLM（大语言模型）的通用软件工程任务代理。它提供了一个强大的 CLI 接口，能够理解自然语言指令，并使用各种工具和 LLM 提供者执行复杂的软件工程工作流。

项目定位是**研究友好的可扩展框架**，其透明、模块化的架构使得研究人员和开发者可以轻松修改、扩展和分析。它特别适合用于**研究 AI 代理架构、进行消融实验、开发新型代理能力**。

主要特性：
- **Lakeview**：提供代理执行步骤的简短、简洁摘要
- **多 LLM 支持**：支持 OpenAI、Anthropic、Azure、OpenRouter、Ollama、Doubao、Google Gemini
- **MCP 协议支持**：通过 MCP（Model Context Protocol）集成外部工具
- **Docker 沙箱**：在隔离的 Docker 容器中安全执行命令
- **代码知识图谱（CKG）**：基于 tree-sitter 的代码库索引和查询工具
- **轨迹记录**：完整的代理执行轨迹记录与回放
- **TUI 界面**：基于 Textual 的终端用户界面

---

## 2. 技术栈

### 编程语言
- **Python >= 3.12**

### 核心依赖

| 依赖 | 用途 |
|------|------|
| `openai>=1.86.0` | OpenAI / 兼容 API 的 LLM 客户端 |
| `anthropic>=0.54.0,<=0.60.0` | Anthropic Claude API 客户端 |
| `google-genai>=1.24.0` | Google Gemini API 客户端 |
| `ollama>=0.5.1` | 本地 Ollama 模型支持 |
| `click>=8.0.0` / `asyncclick>=8.0.0` | CLI 命令行框架 |
| `pydantic>=2.0.0` | 数据验证与配置管理 |
| `rich>=13.0.0` | 终端富文本输出 |
| `textual>=0.50.0` | TUI 终端用户界面 |
| `tree-sitter==0.21.3` / `tree-sitter-languages==1.10.2` | 代码解析（CKG 模块） |
| `mcp==1.12.2` | MCP 协议客户端 |
| `ruff>=0.12.4` | Python 代码检查 |
| `pyyaml>=6.0.2` | YAML 配置支持 |
| `jsonpath-ng>=1.7.0` | JSONPath 表达式（JSON 编辑工具） |
| `python-dotenv>=1.0.0` | 环境变量加载 |
| `socksio>=1.0.0` | SOCKS 代理支持 |

### 可选依赖

| 分组 | 依赖 | 用途 |
|------|------|------|
| `test` | pytest, pytest-asyncio, pytest-mock, pytest-cov, pre-commit | 测试与 CI |
| `evaluation` | datasets, docker, pexpect, unidiff | 评估与 Docker 集成 |

### 构建工具
- **hatchling**：Python 包构建后端
- **pyinstaller**：可执行文件打包

---

## 3. 项目结构

```
trae-agent/
├── .github/                          # GitHub 配置（Issue模板、CI workflows）
├── .vscode/                          # VSCode 调试配置
├── docs/                             # 文档
│   ├── TRAJECTORY_RECORDING.md       # 轨迹记录说明
│   ├── legacy_config.md              # 旧版配置说明
│   ├── roadmap.md                    # 项目路线图
│   └── tools.md                      # 工具文档
├── tests/                            # 测试目录
│   ├── agent/                        # 代理测试
│   ├── tools/                        # 工具测试
│   ├── utils/                        # 工具测试
│   └── test_cli.py                   # CLI 测试
├── trae_agent/                       # 主源码目录
│   ├── __init__.py                   # 包入口，导出主要类
│   ├── cli.py                        # CLI 命令行入口
│   ├── agent/                        # 代理核心模块
│   │   ├── __init__.py               # Agent 门面类
│   │   ├── agent.py                  # Agent 工厂/门面
│   │   ├── agent_basics.py           # 基础数据类定义
│   │   ├── base_agent.py             # 抽象基类代理
│   │   ├── trae_agent.py             # TraeAgent 具体实现
│   │   └── docker_manager.py         # Docker 容器管理
│   ├── tools/                        # 工具模块
│   │   ├── __init__.py               # 工具注册表
│   │   ├── base.py                   # 工具基类与执行器
│   │   ├── bash_tool.py              # Bash 命令执行工具
│   │   ├── edit_tool.py              # 文本编辑器工具
│   │   ├── json_edit_tool.py         # JSON 编辑工具
│   │   ├── sequential_thinking_tool.py # 顺序思考工具
│   │   ├── task_done_tool.py         # 任务完成标记工具
│   │   ├── ckg_tool.py               # 代码知识图谱工具
│   │   ├── mcp_tool.py               # MCP 协议工具适配器
│   │   ├── docker_tool_executor.py   # Docker 工具执行器
│   │   ├── run.py                    # 异步命令执行工具
│   │   └── ckg/                      # 代码知识图谱子模块
│   │       ├── base.py               # CKG 数据类定义
│   │       └── ckg_database.py       # CKG 数据库实现
│   ├── utils/                        # 工具模块
│   │   ├── config.py                 # 配置系统（新 YAML 版）
│   │   ├── legacy_config.py          # 旧版 JSON 配置
│   │   ├── constants.py              # 常量定义
│   │   ├── lake_view.py              # Lakeview 步骤摘要
│   │   ├── trajectory_recorder.py    # 轨迹记录器
│   │   ├── mcp_client.py             # MCP 协议客户端
│   │   ├── cli/                      # CLI 控制台模块
│   │   │   ├── __init__.py           # 控制台工厂导出
│   │   │   ├── cli_console.py        # CLI 控制台抽象基类
│   │   │   ├── console_factory.py    # 控制台工厂
│   │   │   ├── rich_console.py       # Rich TUI 控制台
│   │   │   ├── rich_console.tcss     # TUI 样式文件
│   │   │   └── simple_console.py     # 简单文本控制台
│   │   └── llm_clients/              # LLM 客户端模块
│   │       ├── base_client.py        # LLM 客户端抽象基类
│   │       ├── llm_basics.py         # LLM 消息/响应数据类
│   │       ├── llm_client.py         # LLM 客户端门面
│   │       ├── anthropic_client.py   # Anthropic Claude 客户端
│   │       ├── openai_client.py      # OpenAI 客户端
│   │       ├── azure_client.py       # Azure OpenAI 客户端
│   │       ├── openrouter_client.py  # OpenRouter 客户端
│   │       ├── doubao_client.py      # 豆包客户端
│   │       ├── google_client.py      # Google Gemini 客户端
│   │       └── ollama_client.py      # Ollama 客户端
│   └── dist/                         # 分发目录
├── pyproject.toml                    # 项目配置与依赖
├── Makefile                          # 开发命令
├── CONTRIBUTING.md                   # 贡献指南
└── README.md                         # 项目 README
```

---

## 4. 整体架构

Trae Agent 采用**分层架构**，从上到下主要分为四层：

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLI 层 (cli.py)                             │
│              Click CLI 命令解析、参数配置、入口调度                      │
├─────────────────────────────────────────────────────────────────────┤
│                       Agent 层 (agent/)                              │
│   ┌──────────┐    ┌──────────────┐    ┌──────────────────────────┐  │
│   │  Agent   │───▶│  BaseAgent   │───▶│      TraeAgent           │  │
│   │  (门面)   │    │  (抽象基类)   │    │  (软件工程专用实现)         │  │
│   └──────────┘    └──────────────┘    └──────────────────────────┘  │
│                           │                                         │
│                    ┌──────┴──────┐                                  │
│                    │ DockerManager│                                  │
│                    └─────────────┘                                  │
├─────────────────────────────────────────────────────────────────────┤
│                     工具层 (tools/)                                  │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│   │ BashTool │ │EditTool │ │JSONEdit │ │CKG Tool │ │MCP Tool │ │
│   └──────────┘ └──────────┘ └──────────┘ └──────────┘ └─────────┘ │
│   ┌───────────────┐ ┌───────────────────────┐                      │
│   │ ThinkingTool  │ │   TaskDoneTool        │                      │
│   └───────────────┘ └───────────────────────┘                      │
│   ┌──────────────────┐ ┌──────────────────────┐                    │
│   │  ToolExecutor    │ │ DockerToolExecutor   │                    │
│   └──────────────────┘ └──────────────────────┘                    │
├─────────────────────────────────────────────────────────────────────┤
│                    基础设施层 (utils/)                               │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────┐      │
│   │LLMClient │ │ MCPClient│ │ LakeView │ │TrajectoryRecord│      │
│   └──────────┘ └──────────┘ └──────────┘ └────────────────┘      │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐                          │
│   │  Config  │ │ Console  │ │ CKG DB   │                          │
│   └──────────┘ └──────────┘ └──────────┘                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 核心设计模式

1. **门面模式（Facade）**：`Agent` 类作为门面，统一管理 `TraeAgent` 及 `TrajectoryRecorder` 的创建和运行
2. **策略模式（Strategy）**：`LLMClient` 根据不同的 provider 选择不同的底层客户端实现
3. **工厂模式（Factory）**：`ConsoleFactory` 根据类型创建不同控制台实现
4. **模板方法模式（Template Method）**：`BaseAgent` 定义执行流程骨架，`TraeAgent` 实现具体步骤
5. **适配器模式（Adapter）**：`MCPTool` 将 MCP 协议的工具适配为本地 `Tool` 接口

---

## 5. 主要模块职责

### 5.1 CLI 模块 (`cli.py`)

CLI 入口，基于 `click` 和 `asyncclick` 框架。提供以下主要命令：

- **`run`**：执行单个任务
- **`interactive`**：进入交互式模式，连续接收用户任务
- **`replay`**：重放之前的执行轨迹
- **`serve`**：启动 MCP 服务端模式

主要函数：
- `resolve_config_file()`：解析配置文件路径，支持 YAML 到 JSON 的自动回退
- `check_docker()`：检查 Docker CLI 和 daemon 是否可用
- `main()`：CLI 入口点

### 5.2 Agent 核心模块 (`agent/`)

#### `agent.py` - Agent 门面
- `Agent` 类：统一创建和运行代理的入口
- `AgentType` 枚举：定义代理类型（目前只有 `TraeAgent`）
- `run()` 方法：异步执行任务，管理 MCP 初始化和清理

#### `agent_basics.py` - 基础数据类
核心数据类定义：
- `AgentStepState`：单步状态枚举（THINKING, CALLING_TOOL, REFLECTING, COMPLETED, ERROR）
- `AgentState`：整体状态枚举（IDLE, RUNNING, COMPLETED, ERROR）
- `AgentStep`：单步数据类，包含思考内容、工具调用、LLM 响应、结果等
- `AgentExecution`：完整执行数据类，包含所有步骤、最终结果、token 用量、执行时间
- `AgentError`：自定义异常类

#### `base_agent.py` - 抽象基类代理
`BaseAgent` 是代理的核心抽象基类，提供：
- 工具初始化（从注册表加载配置的工具）
- LLM 客户端初始化
- Docker 环境集成（`DockerToolExecutor` 包装）
- 轨迹记录器设置
- 核心执行循环（`execute_task()` 方法）
- 消息历史管理
- CKG 数据库清理

#### `trae_agent.py` - TraeAgent 实现
`TraeAgent` 继承 `BaseAgent`，专为软件工程任务设计：
- 默认工具集：`str_replace_based_edit_tool`, `sequentialthinking`, `json_edit_tool`, `task_done`, `bash`
- 系统提示词（来自 `prompt/agent_prompt.py`）
- MCP 服务器集成
- `new_task()` 方法：设置任务上下文（项目路径、base commit、patch 路径等）
- `execute_task()` 方法：实现具体的执行循环（调用 LLM -> 解析工具调用 -> 执行工具 -> 循环）

#### `docker_manager.py` - Docker 容器管理
`DockerManager` 管理 Docker 容器生命周期：
- 支持从镜像名、容器 ID、Dockerfile 或 tar 镜像文件启动
- 工作区挂载
- 工具脚本复制到容器
- 交互式/非交互式 shell 执行
- 容器启动/停止/清理

### 5.3 工具模块 (`tools/`)

#### `base.py` - 工具基类
核心抽象类和数据结构：
- `ToolError`：工具错误异常
- `ToolExecResult`：工具执行中间结果（output, error, error_code）
- `ToolResult`：工具执行结果（call_id, name, success, result, error, id）
- `ToolCall`：工具调用定义（name, call_id, arguments, id）
- `ToolParameter`：工具参数定义（name, type, description, enum, items, required）
- `Tool`（抽象基类）：所有工具的基类，定义 `get_name()`, `get_description()`, `get_parameters()`, `execute()`, `json_definition()` 等方法
- `ToolExecutor`：工具执行器，管理工具注册和执行，支持并行和串行执行

#### 内置工具

| 工具类 | 注册名 | 功能 |
|--------|--------|------|
| `BashTool` | `bash` | 在 bash shell 中执行命令，支持持久会话 |
| `TextEditorTool` | `str_replace_based_edit_tool` | 文件查看、创建、字符串替换、插入 |
| `JSONEditTool` | `json_edit_tool` | 使用 JSONPath 表达式编辑 JSON 文件 |
| `SequentialThinkingTool` | `sequentialthinking` | 顺序思考，辅助复杂问题分解 |
| `TaskDoneTool` | `task_done` | 标记任务完成 |
| `CKGTool` | `ckg` | 查询代码知识图谱（搜索函数、类、方法） |
| `MCPTool` | 动态注册 | MCP 协议工具适配器 |

#### 其他工具文件
- `run.py`：异步命令执行工具，支持超时和输出截断
- `docker_tool_executor.py`：Docker 工具执行器，将工具调用路由到 Docker 容器中执行
- `ckg/`：代码知识图谱子模块
  - `base.py`：定义 `FunctionEntry` 和 `ClassEntry` 数据类
  - `ckg_database.py`：基于 tree-sitter 和 SQLite 的代码库解析和查询

### 5.4 基础设施模块 (`utils/`)

#### 配置系统 (`config.py`)
双层配置结构：
- `ModelProvider`：模型提供商配置（api_key, provider, base_url, api_version）
- `ModelConfig`：模型参数（model, temperature, top_p, top_k 等）
- `TraeAgentConfig`：TraeAgent 配置（工具列表、最大步数、MCP 服务器配置等）
- `AgentConfig`：通用代理配置
- `Config`：顶层配置，包含 lakeview 和多种 agent 配置

#### LLM 客户端 (`utils/llm_clients/`)

- `base_client.py`：`BaseLLMClient` 抽象基类，定义 `chat()` 接口
- `llm_basics.py`：`LLMMessage`, `LLMUsage`, `LLMResponse` 数据类
- `llm_client.py`：`LLMClient` 门面，根据 provider 类型选择具体实现
- 各提供商客户端实现：
  - `anthropic_client.py` - Anthropic Claude
  - `openai_client.py` - OpenAI
  - `azure_client.py` - Azure OpenAI
  - `openrouter_client.py` - OpenRouter
  - `doubao_client.py` - 豆包
  - `google_client.py` - Google Gemini
  - `ollama_client.py` - Ollama（本地模型）

#### MCP 客户端 (`mcp_client.py`)
- 支持 stdio 传输的 MCP 协议客户端
- 服务器状态管理（DISCONNECTED, CONNECTING, CONNECTED）
- 工具发现和调用

#### Lakeview (`lake_view.py`)
- 使用 LLM 对代理执行步骤进行自动摘要和标签化
- `extract_task_in_step()`：提取步骤的任务描述
- `extract_tag_in_step()`：对步骤进行标签分类（WRITE_TEST, EXAMINE_CODE, WRITE_FIX 等）
- `create_lakeview_step()`：创建步骤摘要

#### 轨迹记录器 (`trajectory_recorder.py`)
- 完整的 JSON 轨迹记录，包括：
  - 任务信息
  - LLM 交互记录（输入消息、响应、token 用量）
  - 代理步骤记录（状态、工具调用、结果）
  - 执行统计（时间、成功状态）
- 实时保存到文件

#### CLI 控制台 (`utils/cli/`)
- `cli_console.py`：`CLIConsole` 抽象基类，定义控制台接口
- `console_factory.py`：`ConsoleFactory` 工厂类
- `rich_console.py`：基于 Textual 的 TUI 实现
- `simple_console.py`：简单文本控制台实现

### 5.5 提示词模块 (`prompt/`)

- `agent_prompt.py`：包含 `TRAE_AGENT_SYSTEM_PROMPT`，定义了：
  - 系统行为指南（文件路径规则、问题解决步骤）
  - 7 步方法论：理解问题 -> 探索定位 -> 复现 bug -> 调试诊断 -> 实现修复 -> 验证测试 -> 工作总结
  - `sequential_thinking` 工具使用指南
  - 基本原则：像高级软件工程师一样工作

---

## 6. 关键类与函数说明

### 6.1 Agent 体系

#### `Agent` (trae_agent/agent/agent.py)
```
Agent.__init__(agent_type, config, trajectory_file, cli_console, docker_config, docker_keep)
  ├── 根据 agent_type 选择具体的 Agent 实现
  ├── 初始化轨迹记录器
  ├── 设置 CLI 控制台和 Lakeview
  └── 设置 MCP 客户端

Agent.run(task, extra_args, tool_names)
  ├── 初始化 MCP 工具
  ├── 打印任务详细信息
  ├── 启动 CLI 控制台异步任务
  ├── 执行 agent.execute_task()
  └── 清理 MCP 客户端
```

#### `BaseAgent` (trae_agent/agent/base_agent.py)
```
BaseAgent.__init__(agent_config, docker_config, docker_keep)
  ├── 创建 LLM 客户端
  ├── 从注册表加载工具列表
  ├── 可选：Docker 环境集成
  └── 创建 ToolExecutor 或 DockerToolExecutor

BaseAgent.execute_task() [核心执行循环]
  while step < max_steps:
    1. 构建消息列表（系统提示 + 历史 + 当前步骤）
    2. 调用 LLM（支持工具调用）
    3. 记录 LLM 交互
    4. 解析 LLM 响应
       - 如果有工具调用 → 执行工具
       - 如果完成 → 返回结果
       - 如果继续 → 循环
    5. 记录代理步骤
    6. 更新 Lakeview 摘要
    7. step++
```

#### `TraeAgent` (trae_agent/agent/trae_agent.py)
```
TraeAgent.__init__(trae_agent_config, docker_config, docker_keep)
  ├── 设置项目路径、base commit
  ├── 配置 MCP 服务器
  ├── 设置默认工具集
  └── 调用 BaseAgent.__init__

TraeAgent.new_task(task, extra_args, tool_names)
  ├── 解析任务参数（project_path, base_commit, must_patch）
  └── 初始化系统提示消息

TraeAgent.execute_task()
  └── 调用 BaseAgent.execute_task() 执行主循环
```

### 6.2 工具体系

#### `Tool` (trae_agent/tools/base.py)
```
Tool (ABC)
  ├── get_name() -> str              # 工具名称
  ├── get_description() -> str        # 工具描述
  ├── get_parameters() -> list[ToolParameter]  # 参数定义
  ├── get_input_schema() -> dict      # 获取 JSON Schema 格式的输入定义
  ├── json_definition() -> dict       # 获取完整的工具定义（用于 LLM）
  ├── execute(arguments) -> ToolExecResult  # 执行工具
  └── close()                         # 资源清理
```

#### `ToolExecutor` (trae_agent/tools/base.py)
```
ToolExecutor.__init__(tools)
  ├── 构建工具名称到实例的映射（不区分大小写和下划线）
  ├── execute_tool_call(tool_call) -> ToolResult
  ├── parallel_tool_call(tool_calls) -> list[ToolResult]  # 并行执行
  └── sequential_tool_call(tool_calls) -> list[ToolResult]  # 串行执行
```

### 6.3 LLM 客户端体系

#### `LLMClient` (trae_agent/utils/llm_clients/llm_client.py)
```
LLMClient.__init__(model_config)
  └── 根据 provider 类型选择具体客户端：
      OPENAI  → OpenAIClient
      ANTHROPIC → AnthropicClient
      AZURE    → AzureClient
      ...

LLMClient.chat(messages, model_config, tools, reuse_history)
  └── 委托给具体客户端

LLMClient.set_trajectory_recorder(recorder)
LLMClient.set_chat_history(messages)
```

#### `BaseLLMClient` (trae_agent/utils/llm_clients/base_client.py)
```
BaseLLMClient (ABC)
  ├── set_chat_history(messages)       # 设置对话历史
  ├── chat(messages, model_config, tools, reuse_history) -> LLMResponse  # 发送消息
  └── supports_tool_calling(model_config) -> bool  # 是否支持工具调用
```

### 6.4 数据类

#### `LLMMessage` (trae_agent/utils/llm_clients/llm_basics.py)
```python
@dataclass
class LLMMessage:
    role: str                          # user / assistant / system
    content: str | None = None
    tool_call: ToolCall | None = None
    tool_result: ToolResult | None = None
```

#### `LLMResponse` (trae_agent/utils/llm_clients/llm_basics.py)
```python
@dataclass
class LLMResponse:
    content: str
    usage: LLMUsage | None = None
    model: str | None = None
    finish_reason: str | None = None
    tool_calls: list[ToolCall] | None = None
```

#### `AgentStep` (trae_agent/agent/agent_basics.py)
```python
@dataclass
class AgentStep:
    step_number: int
    state: AgentStepState
    thought: str | None = None
    tool_calls: list[ToolCall] | None = None
    tool_results: list[ToolResult] | None = None
    llm_response: LLMResponse | None = None
    reflection: str | None = None
    error: str | None = None
    extra: dict[str, object] | None = None
    llm_usage: LLMUsage | None = None
```

### 6.5 控制台体系

#### `CLIConsole` (trae_agent/utils/cli/cli_console.py)
```
CLIConsole (ABC)
  ├── start()                           # 启动控制台显示
  ├── update_status(agent_step, agent_execution)  # 更新状态
  ├── print_task_details(details)       # 打印任务配置
  ├── print(message, color, bold)       # 打印消息
  ├── get_task_input() -> str           # 获取用户输入（交互模式）
  ├── get_working_dir_input() -> str    # 获取工作目录输入
  └── stop()                            # 停止控制台
```

#### 控制台实现
- **`SimpleConsole`**：简单文本输出，适用于 CI/非交互环境
- **`RichConsole`**：基于 Textual 的 TUI，提供更丰富的可视化界面

### 6.6 MCP 工具

#### `MCPClient` (trae_agent/utils/mcp_client.py)
```
MCPClient
  ├── connect_and_discover(mcp_server_name, config, tools_container, model_provider)
  │   ├── 建立 stdio 传输连接
  │   ├── 初始化 MCP 会话
  │   └── 发现工具并创建 MCPTool 实例
  ├── call_tool(name, args)            # 调用 MCP 工具
  ├── list_tools()                     # 列出可用工具
  └── cleanup(mcp_server_name)         # 清理资源
```

#### `MCPTool` (trae_agent/tools/mcp_tool.py)
将 MCP 协议的工具适配为本地 `Tool` 接口，实现 `get_name()`、`get_description()`、`get_parameters()`、`execute()` 方法。

### 6.7 Docker 管理

#### `DockerManager` (trae_agent/agent/docker_manager.py)
```
DockerManager.__init__(image, container_id, dockerfile_path, docker_image_file, ...)
DockerManager.start()
  ├── 构建镜像（如果提供 Dockerfile）
  ├── 加载镜像（如果提供 tar 文件）
  ├── 创建/附加容器
  ├── 挂载工作区
  ├── 复制工具脚本
  └── 启动 shell

DockerManager.execute(command)          # 在容器中执行命令
DockerManager.stop()                    # 停止容器
DockerManager.close()                   # 关闭并清理
```

#### `DockerToolExecutor` (trae_agent/tools/docker_tool_executor.py)
将工具调用路由到 Docker 容器中执行，支持路径翻译。

---

## 7. 依赖关系

### 模块依赖图

```
cli.py
  ├── agent/agent.py (Agent)
  │     ├── agent/agent_basics.py (AgentStep, AgentExecution...)
  │     ├── agent/trae_agent.py (TraeAgent)
  │     │     ├── agent/base_agent.py (BaseAgent)
  │     │     │     ├── agent/agent_basics.py
  │     │     │     ├── agent/docker_manager.py
  │     │     │     ├── tools/__init__.py (tools_registry)
  │     │     │     │     ├── tools/base.py (Tool, ToolExecutor)
  │     │     │     │     ├── tools/bash_tool.py
  │     │     │     │     ├── tools/edit_tool.py
  │     │     │     │     ├── tools/json_edit_tool.py
  │     │     │     │     ├── tools/sequential_thinking_tool.py
  │     │     │     │     ├── tools/task_done_tool.py
  │     │     │     │     ├── tools/ckg_tool.py
  │     │     │     │     │     └── tools/ckg/ckg_database.py
  │     │     │     │     │           └── tools/ckg/base.py
  │     │     │     │     └── tools/mcp_tool.py
  │     │     │     ├── tools/docker_tool_executor.py
  │     │     │     ├── utils/llm_clients/llm_client.py
  │     │     │     │     ├── utils/llm_clients/base_client.py
  │     │     │     │     ├── utils/llm_clients/llm_basics.py
  │     │     │     │     ├── utils/llm_clients/anthropic_client.py
  │     │     │     │     ├── utils/llm_clients/openai_client.py
  │     │     │     │     ├── utils/llm_clients/azure_client.py
  │     │     │     │     ├── utils/llm_clients/google_client.py
  │     │     │     │     ├── utils/llm_clients/ollama_client.py
  │     │     │     │     ├── utils/llm_clients/doubao_client.py
  │     │     │     │     └── utils/llm_clients/openrouter_client.py
  │     │     │     ├── utils/config.py
  │     │     │     ├── utils/trajectory_recorder.py
  │     │     │     └── utils/cli/cli_console.py
  │     │     ├── utils/mcp_client.py
  │     │     │     └── tools/mcp_tool.py
  │     │     └── prompt/agent_prompt.py
  │     ├── utils/trajectory_recorder.py
  │     └── utils/cli/__init__.py
  │           ├── utils/cli/cli_console.py
  │           ├── utils/cli/console_factory.py
  │           ├── utils/cli/rich_console.py
  │           └── utils/cli/simple_console.py
  └── utils/config.py
        └── utils/legacy_config.py
```

### 核心依赖流向

```
CLI 依赖 Agent → Agent 依赖 BaseAgent, TrajectoryRecorder
BaseAgent 依赖 ToolExecutor, LLMClient, DockerManager, TrajectoryRecorder
TraeAgent 依赖 BaseAgent, MCPClient, AgentPrompt
ToolExecutor 依赖 Tool（各工具实现）
LLMClient 依赖 BaseLLMClient（各提供商客户端）
MCPClient 依赖 MCPTool
Console 依赖 AgentStep, Config, LakeView
Config 依赖 LegacyConfig
```

---

## 8. 配置系统

### 8.1 配置文件格式（YAML/JSON）

Trae Agent 支持 YAML 和 JSON 两种配置文件格式，YAML 优先。

### 8.2 配置结构

```yaml
# 主要配置
default_provider: "anthropic"              # 默认 LLM 提供商
max_steps: 20                              # 最大执行步数
enable_lakeview: true                      # 是否启用 Lakeview

# 模型提供商配置
model_providers:
  anthropic:
    model: "claude-sonnet-4-20250514"
    api_key: "${ANTHROPIC_API_KEY}"        # 支持环境变量引用
    base_url: "https://api.anthropic.com"
    max_tokens: 4096
    temperature: 0.5
    top_p: 1
    top_k: 0
    parallel_tool_calls: false
    max_retries: 10
  openai:
    model: "gpt-4o"
    api_key: "${OPENAI_API_KEY}"
    # ...

# MCP 服务器配置
mcp_servers:
  my-server:
    command: "node"
    args: ["path/to/server.js"]
    env:
      KEY: "value"
    # 或使用 URL
    # url: "http://localhost:3000"

# 允许的 MCP 服务器列表
allow_mcp_servers: ["my-server"]

# Tools 配置
tools:
  - "bash"
  - "str_replace_based_edit_tool"
  - "json_edit_tool"
  - "sequentialthinking"
  - "task_done"

# Lakeview 配置
lakeview_config:
  model_provider: "openai"
  model_name: "gpt-4o-mini"
```

### 8.3 配置类层次

```
Config
  ├── lakeview: LakeviewConfig
  │     ├── model: ModelConfig
  │     └── ...
  ├── trae_agent: TraeAgentConfig
  │     ├── model: ModelConfig
  │     ├── tools: list[str]
  │     ├── max_steps: int
  │     ├── mcp_servers_config: dict[str, MCPServerConfig]
  │     ├── allow_mcp_servers: list[str]
  │     └── enable_lakeview: bool
  └── ... (未来可扩展其他 agent 类型)

ModelConfig
  ├── model: str
  ├── model_provider: ModelProvider
  │     ├── api_key: str
  │     ├── provider: str
  │     ├── base_url: str | None
  │     └── api_version: str | None
  ├── temperature: float
  ├── top_p: float
  ├── top_k: int
  ├── parallel_tool_calls: bool
  ├── max_retries: int
  └── ...
```

---

## 9. 项目运行方式

### 9.1 安装

```bash
# 克隆仓库
git clone https://github.com/bytedance/trae-agent.git
cd trae-agent

# 安装开发环境
make install-dev
make pre-commit-install
```

### 9.2 配置

在项目根目录创建 `trae_config.yaml` 或 `trae_config.json` 配置文件，配置 LLM API 密钥和模型参数。

### 9.3 CLI 使用

```bash
# 运行单个任务
trae-cli run "请修复 src/main.py 中的 bug" --project-path /path/to/project

# 交互式模式
trae-cli interactive

# 指定配置文件
trae-cli run --config-file trae_config.yaml "请添加新功能"

# 使用 Docker 运行
trae-cli run --docker --docker-image ubuntu:22.04 "请修复 bug"

# 重放轨迹
trae-cli replay --trajectory-file trajectories/trajectory_20250101_120000.json

# 启用 Lakeview
trae-cli run --enable-lakeview "请修复 bug"

# 指定控制台类型
trae-cli run --console-type simple "请修复 bug"
```

### 9.4 CLI 参数

```
run 命令：
  TASK                       任务描述
  --project-path TEXT        项目路径
  --config-file TEXT         配置文件路径
  --docker                   启用 Docker
  --docker-image TEXT        Docker 镜像
  --docker-container TEXT    Docker 容器 ID
  --dockerfile-path TEXT     Dockerfile 路径
  --docker-keep              Docker 容器执行后保留
  --trajectory-file TEXT     轨迹文件路径
  --enable-lakeview          启用 Lakeview
  --console-type [simple|rich]  控制台类型
  --tool-names TEXT          使用的工具列表（逗号分隔）
  -extra TEXT                额外参数
```

### 9.5 运行测试

```bash
make test
```

或使用 pytest 直接运行：

```bash
pytest tests/ -v
```

### 9.6 打包构建

```bash
# 构建 wheel 包
pip install build
python -m build

# 使用 pyinstaller 打包为独立可执行文件
pyinstaller trae-agent.spec
```

---

## 10. 数据流与执行流程

### 10.1 完整执行流程

```
用户输入任务
    │
    ▼
trae-cli run "修复 bug"
    │
    ├── 解析 CLI 参数
    ├── 加载配置文件 (config.py)
    ├── 检查 Docker 环境 (check_docker)
    │
    ▼
Agent.run(task)
    │
    ├── 创建 TraeAgent 实例
    ├── 创建 TrajectoryRecorder 实例
    ├── 创建 CLIConsole 实例
    │
    ▼
Agent.new_task(task)
    │
    ├── 解析任务参数（project_path, base_commit 等）
    ├── 构建系统提示消息
    │     └── TRAE_AGENT_SYSTEM_PROMPT
    └── 初始化消息历史
    │
    ▼
Agent.initialise_mcp()  （如果配置了 MCP 服务器）
    │
    ├── 为每个 MCP 服务器创建 MCPClient
    ├── connect_and_discover()
    │     ├── 建立 stdio 连接
    │     ├── 初始化 MCP 会话
    │     └── 发现工具列表
    └── 将 MCPTool 添加到工具列表
    │
    ▼
BaseAgent.execute_task()  [主循环]
    │
    ┌─────────────────────────────────────────────────────┐
    │  while step < max_steps:                            │
    │     │                                               │
    │     ├── 构建消息列表                                │
    │     │   ├── 系统提示                                │
    │     │   ├── 初始消息                                │
    │     │   └── 历史消息 + 当前消息                     │
    │     │                                               │
    │     ├── LLMClient.chat(messages, tools)              │
    │     │     └── 具体客户端发送请求并获取响应            │
    │     │                                               │
    │     ├── TrajectoryRecorder.record_llm_interaction()  │
    │     │                                               │
    │     ├── 解析 LLM 响应                               │
    │     │   ├── 有 tool_calls → 执行工具                │
    │     │   │     ├── ToolExecutor.execute_tool_call()   │
    │     │   │     │     ├── BashTool.execute()          │
    │     │   │     │     ├── TextEditorTool.execute()    │
    │     │   │     │     ├── JSONEditTool.execute()      │
    │     │   │     │     ├── SequentialThinkingTool.execute()│
    │     │   │     │     ├── CKGTool.execute()           │
    │     │   │     │     └── MCPTool.execute()           │
    │     │   │     │           └── MCPClient.call_tool() │
    │     │   │     └── 收集工具结果                      │
    │     │   │                                               │
    │     │   ├── 内容包含完成标记 → 结束循环              │
    │     │   │                                               │
    │     │   └── 需要继续思考 → 继续循环                  │
    │     │                                               │
    │     ├── 记录 AgentStep                               │
    │     ├── Lakeview 更新（如果启用）                    │
    │     │     ├── extract_task_in_step()                 │
    │     │     └── extract_tag_in_step()                  │
    │     │                                               │
    │     └── step++                                      │
    └─────────────────────────────────────────────────────┘
    │
    ▼
AgentExecution 返回
    │
    ├── 清理 MCP 客户端
    ├── Finalize TrajectoryRecorder
    ├── CLIConsole 停止
    └── 输出执行结果
```

### 10.2 工具调用数据流

```
LLM 响应中包含 tool_calls
    │
    ▼
解析 ToolCall 列表
    │
    ▼
ToolExecutor 路由工具调用
    │
    ├── 名称归一化（小写 + 去下划线）
    ├── 查找工具注册表
    │
    ▼
执行工具
    │
    ├── BashTool: 创建/复用 bash 会话 → 写入命令 → 读取输出
    ├── TextEditorTool: view/create/str_replace/insert
    ├── JSONEditTool: view/set/add/remove（JSONPath）
    ├── SequentialThinkingTool: 记录思考链
    ├── TaskDoneTool: 标记完成
    ├── CKGTool: 查询 SQLite 代码知识图谱
    └── MCPTool: 通过 MCP 协议调用外部工具
    │
    ▼
返回 ToolResult
    │
    ▼
将结果作为新消息加入消息历史
    │
    ▼
继续下一次 LLM 调用
```

### 10.3 Docker 集成数据流

```
配置了 docker_config
    │
    ▼
BaseAgent.__init__
    ├── 创建 DockerManager
    │     ├── 构建/拉取镜像
    │     ├── 创建容器（挂载工作区）
    │     └── 复制工具脚本到容器
    │
    └── 创建 DockerToolExecutor
          ├── 包装原始 ToolExecutor
          └── 对指定工具（bash, edit, json_edit）进行路径翻译
                │
                ▼
          DockerToolExecutor._execute_in_docker()
                ├── 翻译文件路径（主机路径 → 容器路径）
                ├── 构建 Docker 命令
                └── DockerManager.execute(command)
```

---

## 附录 A：Lakeview 标签系统

| 标签 | 含义 | 图标 |
|------|------|------|
| `WRITE_TEST` | 编写测试脚本 | ☑️ |
| `VERIFY_TEST` | 运行测试验证环境 | ✅ |
| `EXAMINE_CODE` | 查看/搜索代码 | 👁️ |
| `WRITE_FIX` | 修改源代码修复 bug | 📝 |
| `VERIFY_FIX` | 运行测试验证修复 | 🔥 |
| `REPORT` | 报告进度/结果 | 📣 |
| `THINK` | 分析思考 | 🧠 |
| `OUTLIER` | 其他操作 | ⁉️ |

## 附录 B：CKG 支持的文件类型

| 扩展名 | 语言 |
|--------|------|
| `.py` | Python |
| `.java` | Java |
| `.cpp`, `.hpp`, `.c++`, `.cxx`, `.cc` | C++ |
| `.c`, `.h` | C |
| `.ts`, `.tsx` | TypeScript |
| `.js`, `.jsx` | JavaScript |

## 附录 C：环境变量

| 变量 | 用途 |
|------|------|
| `OPENAI_API_KEY` | OpenAI API 密钥 |
| `ANTHROPIC_API_KEY` | Anthropic API 密钥 |
| `GOOGLE_API_KEY` | Google Gemini API 密钥 |
| `AZURE_API_KEY` | Azure OpenAI API 密钥 |
| 其他 | 对应各 LLM 提供商的 API 密钥 |

所有环境变量也可以通过 `.env` 文件加载（由 `python-dotenv` 提供支持）。