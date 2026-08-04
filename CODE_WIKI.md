# Pi Agent Harness — Code Wiki

> 本文档是对 `pi-mono` 项目仓库的完整代码分析，涵盖技术栈、整体架构、各模块职责、关键类/函数、依赖关系及项目运行方式。

---

## 1. 项目概述

**Pi Agent Harness** 是一个可扩展的编码代理（coding agent）CLI 工具，核心功能是与 LLM（大语言模型）交互，通过工具调用（Tool Calling）实现自动化的代码编写、文件操作、命令执行等开发任务。

项目采用 **monorepo** 架构，使用 npm workspaces 管理多个子包，所有包共享统一的版本号（当前 v0.0.3）。

- 官网: [https://pi.dev](https://pi.dev)
- 文档: [https://pi.dev/docs/latest](https://pi.dev/docs/latest)
- 许可证: MIT

---

## 2. 技术栈

| 类别 | 技术 |
|------|------|
| **语言** | TypeScript 5.9+ (strict mode, NodeNext 模块解析) |
| **运行时** | Node.js >= 22.19.0, 也支持 Bun 打包为独立二进制 |
| **包管理** | npm workspaces (monorepo) |
| **构建工具** | esbuild (打包), TypeScript (类型检查) |
| **代码检查** | Biome (lint + format), 自定义检查脚本 |
| **序列化** | CBOR (用于协议层), typebox (运行时类型验证) |
| **终端 UI** | 自研 TUI 库 (差分渲染, 键盘事件, 图片支持) |
| **AI 提供商** | OpenAI, Anthropic, Google, Mistral, Amazon Bedrock, Azure, OpenRouter 等 30+ |
| **测试** | Vitest (各包独立测试) |
| **版本控制** | Git + GitHub Actions (CI/CD) |

---

## 3. 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    pi (coding-agent CLI)                     │
│  InteractiveMode / PrintMode / RpcMode                      │
├─────────────────────────────────────────────────────────────┤
│                   @pi-coding-agent                           │
│  AgentSession · SDK · Extensions · Compaction · Tools       │
├─────────────────────────────────────────────────────────────┤
│   @pi-agent-core          @pi-tui         @pi-ai             │
│   Agent · AgentLoop      Terminal UI     Models · Providers  │
│   Harness · Session      Components      Auth · Streaming    │
│   Tools · Skills         Keybindings     API Adapters        │
├──────────────────────┬──────────────────────────────────────┤
│  @pi-protocol        │  @pi-client / @pi-server              │
│  CBOR · Framing      │  RPC 客户端-服务端通信                │
│  Schemas · Codec     │  会话代理、连接管理                   │
├──────────────────────┴──────────────────────────────────────┤
│  @pi-agent-sqlite-node                                       │
│  SQLite 会话持久化存储 (Node.js 内置 sqlite 模块)            │
└─────────────────────────────────────────────────────────────┘
```

### 架构分层说明

1. **用户界面层** — `@pi-coding-agent` 的 `InteractiveMode` / `PrintMode` / `RpcMode`
2. **编码代理层** — `@pi-coding-agent` 的 SDK、会话管理、扩展系统、工具系统
3. **核心代理层** — `@pi-agent-core` 的 Agent、AgentLoop、Harness、Session 抽象
4. **AI 层** — `@pi-ai` 的多提供商 LLM 统一 API
5. **TUI 层** — `@pi-tui` 的终端 UI 组件库
6. **通信协议层** — `@pi-protocol` 的客户端-服务端协议
7. **服务端/客户端层** — `@pi-server` / `@pi-client` 的 RPC 通信
8. **存储层** — `@pi-agent-sqlite-node` 的会话持久化

---

## 4. 包依赖关系

```
@pi-ai                           (无内部依赖)
  ↑
@pi-tui                          (仅依赖外部库)
  ↑
@pi-agent-core                   (依赖 @pi-ai)
  ↑
@pi-protocol                     (无内部依赖)
  ↑
@pi-client / @pi-server          (依赖 @pi-protocol)
  ↑
@pi-agent-sqlite-node            (依赖 @pi-ai, @pi-agent-core)
  ↑
@pi-coding-agent                 (依赖所有上层包)
```

### 包列表

| npm 包名 | 目录 | 说明 |
|----------|------|------|
| `@earendil-works/pi-ai` | `packages/ai` | 统一多提供商 LLM API |
| `@earendil-works/pi-agent-core` | `packages/agent` | 代理运行时、工具调用、状态管理 |
| `@earendil-works/pi-coding-agent` | `packages/coding-agent` | 交互式编码代理 CLI |
| `@earendil-works/pi-tui` | `packages/tui` | 终端 UI 组件库 |
| `@earendil-works/pi-protocol` | `packages/protocol` | 客户端-服务端通信协议 |
| `@earendil-works/pi-client` | `packages/client` | RPC 客户端 |
| `@earendil-works/pi-server` | `packages/server` | RPC 服务端 |
| `@earendil-works/pi-agent-sqlite-node` | `packages/storage/sqlite-node` | SQLite 会话存储 |

---

## 5. 各模块详细说明

### 5.1 `@earendil-works/pi-ai` — AI 抽象层

**职责**: 提供统一的 LLM API 接口，屏蔽不同 AI 提供商（OpenAI、Anthropic、Google 等）的差异。

**核心目录结构**:
```
src/
├── index.ts               # 入口，导出所有核心类型
├── types.ts               # 核心类型定义 (Model, Message, Context, Tool, Usage 等)
├── models.ts              # Model 接口、Provider 接口、Models 集合
├── models-store.ts        # 模型元数据持久化存储
├── api/                   # 各 API 实现
│   ├── openai-responses.ts
│   ├── anthropic-messages.ts
│   ├── google-generative-ai.ts
│   ├── bedrock-converse-stream.ts
│   ├── mistral-conversations.ts
│   ├── lazy.ts            # 懒加载流包装
│   └── ...
├── providers/             # 提供商工厂函数
│   ├── openai.ts
│   ├── anthropic.ts
│   └── all.ts             # 聚合导出所有提供商
├── auth/                  # 认证系统
│   ├── context.ts
│   ├── credential-store.ts
│   ├── resolve.ts
│   └── types.ts
└── utils/                 # 工具函数
    ├── event-stream.ts    # EventStream 实现 (流式事件处理)
    ├── retry.ts           # 重试策略
    └── validation.ts      # 工具参数验证
```

**关键类型**:

- **`Model<TApi>`** — 模型描述，包含 id、name、provider、api、baseUrl、cost、contextWindow、maxTokens 等
- **`Provider<TApi>`** — 提供商运行时，拥有 auth、getModels()、stream()、streamSimple() 方法
- **`Models`** — 提供商集合，负责 auth 解析、模型刷新、流式请求分发
- **`Message`** — 统一消息类型（`UserMessage | AssistantMessage | ToolResultMessage`）
- **`Context`** — LLM 请求上下文（systemPrompt + messages + tools）
- **`Tool<TParameters>`** — 工具定义（name + description + parameters + constrainedSampling）
- **`AssistantMessageEventStream`** — 流式响应的事件流
- **`Usage`** — token 用量和成本计算

**支持提供商**: 30+ 个已知提供商，包括 OpenAI、Anthropic、Google、Amazon Bedrock、Azure、Mistral、DeepSeek、xAI、Groq、OpenRouter、GitHub Copilot、MoonshotAI 等。

**`Models` 类** (`models.ts`):
- `getProviders()` / `getProvider(id)` — 获取提供商
- `getModels(provider?)` — 获取模型列表
- `getModel(provider, id)` — 按 ID 查找模型
- `refresh()` — 刷新所有动态提供商的模型列表
- `checkAuth(providerId)` — 检查认证状态
- `getAuth(model)` — 解析认证信息
- `stream(model, context, options)` — 流式请求
- `complete(model, context, options)` — 完整请求
- `login(providerId, type, interaction)` — 执行登录流程
- `createModels(options)` — 工厂函数创建 Models 实例

---

### 5.2 `@earendil-works/pi-agent-core` — 代理核心

**职责**: 提供通用的 Agent 运行时，包括 Agent 类、事件循环、Harness 编排、会话管理、工具系统、技能系统。

**核心目录结构**:
```
src/
├── index.ts               # 入口，导出所有模块
├── agent.ts               # Agent 类 (状态管理、事件、队列)
├── agent-loop.ts          # Agent 事件循环 (LLM 调用、工具执行、消息处理)
├── types.ts               # 核心类型 (AgentState, AgentEvent, AgentTool 等)
├── node.ts                # Node.js 环境入口
├── stream-fn.ts           # 默认流函数配置
├── proxy.ts               # 代理流函数 (通过服务端代理 LLM 请求)
├── harness/
│   ├── agent-harness.ts   # AgentHarness 类 (编排层)
│   ├── types.ts           # 所有 Harness 类型 (FileSystem, Shell, Session 等)
│   ├── messages.ts        # 自定义消息类型 (BashExecution, Custom, BranchSummary 等)
│   ├── skills.ts          # 技能加载系统 (SKILL.md 文件解析)
│   ├── system-prompt.ts   # 系统提示词格式化
│   ├── prompt-templates.ts # 提示词模板系统
│   ├── compaction/        # 会话压缩系统
│   │   ├── compaction.ts  # 压缩逻辑
│   │   └── branch-summarization.ts # 分支摘要
│   ├── session/
│   │   ├── session.ts     # Session 类 (树状会话存储)
│   │   ├── repository.ts  # 会话仓库接口
│   │   ├── jsonl-repo.ts  # JSONL 文件仓库
│   │   └── memory-repo.ts # 内存仓库
│   └── tools/             # 内置工具
│       ├── index.ts       # 工具导出
│       ├── bash.ts        # Bash 执行工具
│       ├── read.ts        # 文件读取工具
│       ├── write.ts       # 文件写入工具
│       └── edit.ts        # 文件编辑工具
```

**`Agent` 类** (`agent.ts`):

核心的状态管理类，封装了低层 Agent 循环。

| 方法/属性 | 说明 |
|-----------|------|
| `constructor(options)` | 初始化 Agent，配置 streamFn、hooks、队列模式等 |
| `state` | 当前代理状态 (systemPrompt, model, messages, isStreaming 等) |
| `prompt(input)` | 发起新的提示，支持字符串或消息数组 |
| `continue()` | 从当前对话继续 |
| `steer(message)` | 在当前助手回复完成后注入消息 |
| `followUp(message)` | 在代理停止后注入消息 |
| `subscribe(listener)` | 订阅生命周期事件 |
| `abort()` | 中止当前运行 |
| `reset()` | 重置所有状态 |
| `waitForIdle()` | 等待当前运行完成 |

**`AgentHarness` 类** (`harness/agent-harness.ts`):

高层次的编排层，管理会话、工具、模型、资源等。

| 方法/属性 | 说明 |
|-----------|------|
| `prompt(text, options?)` | 发起提示并等待回复 |
| `skill(name, additionalInstructions?)` | 执行技能 |
| `steer(text)` | 在运行中注入消息 |
| `followUp(text)` | 在运行后注入消息 |
| `compact(customInstructions?)` | 压缩会话历史 |
| `navigateTree(targetId, options?)` | 导航到会话树的指定节点 |
| `setModel(model)` | 切换模型 |
| `setThinkingLevel(level)` | 设置推理级别 |
| `setTools(tools, activeToolNames?)` | 设置可用工具 |
| `subscribe(listener)` | 订阅事件 (包括 Agent 事件和 Harness 事件) |
| `on(type, handler)` | 订阅特定类型的事件钩子 |
| `abort()` | 中止 |
| `requestShutdown()` | 请求关闭 |

**Agent 事件循环** (`agent-loop.ts`):

核心执行流程:
1. 接收用户消息 → 2. 调用 LLM stream → 3. 解析工具调用 → 4. 执行工具 → 5. 返回结果到 LLM → 6. 重复直到停止

支持两种工具执行模式:
- `parallel` (默认): 工具调用并行执行
- `sequential`: 工具调用依次执行

**会话系统** (`harness/session/session.ts`):

`Session` 类使用树状结构存储会话条目，支持:
- 消息追加 (`appendMessage`)
- 模型变更 (`appendModelChange`)
- 推理级别变更 (`appendThinkingLevelChange`)
- 压缩条目 (`appendCompaction`)
- 分支摘要 (`appendBranchSummary`)
- 树导航 (`moveTo`)
- 上下文构建 (`buildContext`)

**技能系统** (`harness/skills.ts`):

从目录递归加载 `SKILL.md` 文件，支持 YAML frontmatter 元数据。技能文件包含名称、描述和具体指令，在系统提示词中以 XML 格式呈现。

**消息系统** (`harness/messages.ts`):

定义了扩展的消息类型:
- `BashExecutionMessage` — bash 执行结果
- `CustomMessage` — 自定义消息
- `BranchSummaryMessage` — 分支摘要
- `CompactionSummaryMessage` — 压缩摘要

`convertToLlm()` 函数将所有自定义消息类型转换为 LLM 可理解的 `Message` 类型。

---

### 5.3 `@earendil-works/pi-coding-agent` — 编码代理 CLI

**职责**: 提供完整的交互式编码代理 CLI，包括会话管理、扩展系统、工具系统、模型管理、配置文件管理。

**核心目录结构**:
```
src/
├── index.ts               # 入口，导出所有公共 API
├── main.ts                # CLI 主入口 (main 函数)
├── config.ts              # 配置路径
├── cli/                   # CLI 参数解析
│   ├── args.ts            # 命令行参数定义
│   ├── list-models.ts     # 列出模型
│   ├── session-picker.ts  # 会话选择器
│   └── startup-ui.ts      # 首次启动 UI
├── core/
│   ├── agent-session.ts   # AgentSession 类
│   ├── sdk.ts             # SDK (createAgentSession, 工具工厂等)
│   ├── model-runtime.ts   # ModelRuntime (模型 + 认证运行时)
│   ├── model-registry.ts  # 模型注册表
│   ├── model-resolver.ts  # 模型解析
│   ├── session-manager.ts # 会话管理器
│   ├── settings-manager.ts# 设置管理器
│   ├── skills.ts          # 技能加载
│   ├── trust-manager.ts   # 项目信任管理
│   ├── extensions/        # 扩展系统
│   │   ├── types.ts       # 扩展类型定义
│   │   └── index.ts       # 扩展运行时
│   ├── tools/             # 工具定义
│   │   ├── index.ts       # 工具导出
│   │   ├── bash.ts        # Bash 工具定义
│   │   ├── read.ts        # Read 工具定义
│   │   ├── write.ts       # Write 工具定义
│   │   ├── edit.ts        # Edit 工具定义
│   │   ├── grep.ts        # Grep 工具定义
│   │   ├── find.ts        # Find 工具定义
│   │   └── ls.ts          # Ls 工具定义
│   └── compaction/        # 压缩 (re-export from agent-core)
├── modes/                 # 运行模式
│   ├── index.ts           # 模式导出
│   ├── interactive/       # 交互模式 (TUI)
│   ├── print.ts           # 打印模式 (非交互)
│   └── rpc.ts             # RPC 模式
└── extensions/            # 内置扩展
```

**`main()` 函数** (`main.ts`):

CLI 的入口点，流程:
1. 解析命令行参数 (`parseArgs`)
2. 初始化配置和设置
3. 加载扩展
4. 解析模型
5. 创建会话
6. 进入交互模式 / 打印模式 / RPC 模式

**`AgentSession` 类** (`core/agent-session.ts`):

编码代理的核心会话类，封装了 Agent + AgentHarness + 各种管理器。

**`createAgentSession()`** (`core/sdk.ts`):

SDK 入口，创建完整的代理会话，配置:
- 内置工具 (Read, Bash, Edit, Write, Grep, Find, Ls)
- 自定义工具扩展
- 模型运行时
- 会话管理器
- 资源加载器
- 扩展系统

**扩展系统** (`core/extensions/`):

支持通过扩展钩子（hooks）自定义代理行为，包括:
- `before_agent_start` — 代理启动前
- `before_provider_request` — 请求前修改请求选项
- `before_provider_payload` — 请求前修改请求体
- `tool_call` — 工具调用前拦截
- `tool_result` — 工具结果后处理
- `session_before_compact` / `session_compact` — 压缩事件
- `session_before_tree` / `session_tree` — 树导航事件
- 自定义工具注册
- 自定义命令注册

**运行模式** (`modes/`):

- **InteractiveMode**: 完整的 TUI 交互模式，包含编辑器、消息列表、模型选择器、设置界面等
- **PrintMode**: 非交互模式，接收提示后输出结果
- **RpcMode**: 作为 RPC 服务运行，接收外部命令

---

### 5.4 `@earendil-works/pi-tui` — 终端 UI 库

**职责**: 提供高性能的终端 UI 组件库，支持差分渲染、键盘事件、图片显示等。

**核心组件**:
- `Box` — 盒子容器
- `Text` — 文本渲染
- `Input` — 输入框
- `Editor` — 编辑器
- `Markdown` — Markdown 渲染
- `SelectList` — 选择列表
- `SettingsList` — 设置列表
- `ScrollView` — 滚动视图
- `VStack` / `HStack` — 垂直/水平堆叠
- `Spacer` — 间距填充
- `Image` — 图片渲染 (Kitty / iTerm2 协议)
- `Loader` / `CancellableLoader` — 加载动画

**核心基础设施**:
- `TUI` / `Container` — TUI 容器和组件基础
- `TuiAltScreen` / `TuiMainScreen` — 备用/主屏幕
- `KeybindingsManager` — 键盘绑定管理
- `ProcessTerminal` — 终端接口实现
- `TerminalCapabilities` — 终端能力检测 (Kitty 协议、图片支持等)
- `StdinBuffer` — 输入缓冲

---

### 5.5 `@earendil-works/pi-protocol` — 通信协议

**职责**: 定义客户端和服务端之间的通信协议，使用 CBOR 二进制编码。

**核心文件**:
- `schemas.ts` — 所有协议消息的 typebox schema
- `codec.ts` — 消息编解码
- `framing.ts` — 消息帧封装
- `cbor/` — CBOR 编解码器

**协议消息类型** (`schemas.ts`):

- **客户端消息**: `ClientHello` (版本协商), `RequestEnvelope` (请求命令)
- **服务端消息**: `ServerHello`, `ResponseEnvelope`, `EventEnvelope` (事件推送)
- **命令**: `list`, `create`, `attach`, `detach`, `prompt`, `steer`, `abort`, `set_model`, `set_thinking`
- **事件**: `server_snapshot`, `session_snapshot`, `session_progress`, `session_removed`
- **会话快照**: 包含 transcript、model、thinkingLevel、queued messages 等

---

### 5.6 `@earendil-works/pi-client` — RPC 客户端

**职责**: 实现 RPC 客户端，通过传输层连接到服务端，管理会话生命周期。

**核心类**:
- `PiClient` — 客户端主类，管理连接、会话租赁、请求路由
- `Connection` — 连接管理
- `SessionHandle` — 会话句柄，提供对会话的读写接口

**功能**:
- 连接管理 (建立/断开/重连)
- 会话租赁 (shared/exclusive 模式)
- 命令发送和响应处理
- 服务端事件监听和处理

---

### 5.7 `@earendil-works/pi-server` — RPC 服务端

**职责**: 实现 RPC 服务端，管理多个客户端连接和会话。

**核心类**:
- `PiServer` — 服务端主类
- `LiveSessionManager` — 活跃会话管理
- `ServerSnapshotPublisher` — 快照发布

**功能**:
- 多传输层支持 (Unix socket)
- 会话创建、管理、锁定
- 命令路由和执行
- 快照广播
- 进度事件推送

---

### 5.8 `@earendil-works/pi-agent-sqlite-node` — SQLite 存储

**职责**: 提供基于 Node.js 内置 `sqlite` 模块的会话持久化存储。

**核心文件**:
- `index.ts` — 包装 Node.js 的 `DatabaseSync`，提供 `SqliteDatabase` 接口
- `sqlite/index.ts` — 数据库工厂和查询抽象
- `sqlite/storage/sessions.ts` — 会话存储
- `sqlite/storage/session-entries.ts` — 会话条目存储
- `sqlite/storage/session-materialized.ts` — 物化视图

---

## 6. 关键数据流

### 6.1 代理执行流程

```
用户输入 (CLI/TUI)
    │
    ▼
AgentSession.prompt()
    │
    ▼
AgentHarness.prompt()
    │
    ├─► 创建 TurnState (上下文、工具、模型)
    ├─► 处理 before_agent_start 钩子
    │
    ▼
Agent.runPromptMessages()
    │
    ▼
runAgentLoop() [agent-loop.ts]
    │
    ├─► 用户消息 → 事件分发
    ├─► streamAssistantResponse()
    │   ├─► transformContext() → 上下文转换
    │   ├─► convertToLlm() → 消息转换
    │   ├─► Models.streamSimple() → LLM 调用
    │   └─► 事件流 (text/toolcall deltas)
    │
    ├─► executeToolCalls()
    │   ├─► beforeToolCall 钩子
    │   ├─► 工具执行 (parallel/sequential)
    │   ├─► afterToolCall 钩子
    │   └─► 工具结果返回上下文
    │
    ├─► prepareNextTurn() → 更新上下文
    ├─► shouldStopAfterTurn() → 判断是否停止
    └─► 循环直到停止
    │
    ▼
    agent_end 事件 → 清理
```

### 6.2 会话树结构

```
Session Tree:
  root
   ├── message (user: "实现一个排序函数")
   ├── message (assistant: 思考中...)
   │   ├── tool_call (read)
   │   └── tool_result
   ├── message (assistant: "这是实现")
   ├── model_change (切换到新模型)
   ├── thinking_level_change (切换到 high)
   ├── compaction (历史压缩摘要)
   ├── branch_summary (分支摘要)
   └── leaf (当前活动叶子节点)
```

### 6.3 协议通信流程

```
Client                              Server
  │                                    │
  ├─ ClientHello ───────────────────►  │
  │                                    ├─ ServerHello (含快照)
  │◄───────────────────────────────────┤
  │                                    │
  ├─ RequestEnvelope (create) ──────►  │
  │                                    ├─ 创建会话
  │◄─ ResponseEnvelope (session) ─────┤
  │                                    │
  ├─ RequestEnvelope (prompt) ──────►  │
  │                                    ├─ 执行代理循环
  │◄─ EventEnvelope (progress) ────────┤
  │◄─ EventEnvelope (progress) ────────┤
  │◄─ ResponseEnvelope (result) ───────┤
```

---

## 7. 关键设计模式

### 7.1 事件流 (EventStream)

整个系统大量使用 `EventStream` 模式处理异步事件流。`AssistantMessageEventStream` 是 LLM 响应流的标准接口，支持 `start` → `text_delta` / `toolcall_delta` → `done` / `error` 的事件序列。

### 7.2 适配器模式 (Provider Adapter)

每个 AI 提供商实现统一的 `ProviderStreams` 接口 (`stream` + `streamSimple`)，通过适配器模式屏蔽 API 差异。

### 7.3 钩子系统 (Hook System)

`AgentHarness` 和扩展系统使用事件钩子机制，允许在代理执行的关键节点注入自定义行为，包括请求前修改、工具调用拦截、会话管理事件等。

### 7.4 消息队列 (Message Queue)

`Agent` 类实现了 steering 和 follow-up 两个消息队列，支持 `all` 和 `one-at-a-time` 两种消费模式，用于在代理运行中注入消息。

### 7.5 会话树 (Session Tree)

会话以树状结构存储，每个条目包含 id、parentId、timestamp，支持分支、导航、压缩和摘要生成。

---

## 8. 内置工具

| 工具 | 说明 | 定义位置 |
|------|------|----------|
| **Read** | 读取文件内容 (支持文本和图片) | `packages/agent/src/harness/tools/read.ts` |
| **Write** | 创建或覆盖文件 | `packages/agent/src/harness/tools/write.ts` |
| **Edit** | 编辑文件 (精确替换) | `packages/agent/src/harness/tools/edit.ts` |
| **Bash** | 执行 shell 命令 | `packages/agent/src/harness/tools/bash.ts` |
| **Grep** | 搜索文件内容 (ripgrep) | `packages/coding-agent/src/core/tools/grep.ts` |
| **Find** | 查找文件 (glob) | `packages/coding-agent/src/core/tools/find.ts` |
| **Ls** | 列出目录内容 | `packages/coding-agent/src/core/tools/ls.ts` |

---

## 9. 配置文件

| 文件 | 说明 |
|------|------|
| `~/.pi/agent/auth.json` | 提供商认证凭据 |
| `~/.pi/agent/models.json` | 缓存的模型元数据 |
| `~/.pi/agent/settings.json` | 用户设置 |
| `~/.pi/agent/trust.json` | 项目信任设置 |
| `~/.pi/agent/sessions/` | 会话数据目录 (JSONL 格式) |

---

## 10. 项目运行方式

### 10.1 开发环境

```bash
# 安装依赖 (不运行生命周期脚本)
npm install --ignore-scripts

# 构建所有包
npm run build

# 离线构建 (使用已有模型数据)
npm run build:offline

# 代码检查 (lint + format + 类型检查)
npm run check

# 运行测试
./test.sh

# 从源码运行 pi
./pi-test.sh
```

### 10.2 生成模型数据

```bash
# 刷新模型数据 (从提供商 API)
npm run generate:models

# 水合模型数据
npm run hydrate:model-data
```

### 10.3 发布

```bash
# 补丁版本
npm run release:patch

# 次要版本 (breaking changes)
npm run release:minor

# 本地发布测试
npm run release:local -- --out /tmp/pi-local-release --force
```

### 10.4 构建独立二进制

```bash
VERSION="<release-version>"
tar -xzf "pi-${VERSION}-source.tar.gz"
cd "pi-${VERSION}"
./scripts/build-binaries.sh --offline-model-data --platform linux-x64 --out "$PWD/out"
```

---

## 11. 安全与权限

- Pi 默认运行在启动用户/进程的权限下，不包含内置权限系统
- 使用 `--no-tools` 或 `--no-tools=builtin` 限制工具访问
- 项目信任机制 (`ProjectTrustStore`) 管理项目级别的访问控制
- 容器化方案: Gondolin (微 VM)、Docker、OpenShell

---

## 12. 扩展开发

Pi 支持通过扩展机制自定义行为:

```typescript
// 扩展示例
import { defineTool, createExtensionRuntime } from "@earendil-works/pi-coding-agent";

const myExtension = {
  name: "my-extension",
  tools: [
    defineTool({
      name: "my_tool",
      description: "My custom tool",
      parameters: { ... },
      execute: async (params) => { ... },
    }),
  ],
  hooks: {
    before_agent_start: async (event) => { ... },
    tool_call: async (event) => { ... },
  },
};
```

---

## 13. 供应链安全

- 外部直接依赖锁定到精确版本
- `package-lock.json` 是依赖的 ground truth
- CI 使用 `npm ci --ignore-scripts`
- 定期运行 `npm audit --omit=dev`
- 发布前进行本地 smoke test
- shrinkwrap 生成有生命周期脚本的白名单机制

---

*本文档由 AI 基于代码分析自动生成，最后更新于 2026-08-04。*