# Pi Code Wiki

> 本文档为 [`pi-mono`](https://github.com/earendil-works/pi-mono) 仓库的结构化代码 Wiki，涵盖项目整体架构、各包模块职责、关键类与函数、依赖关系以及运行方式。

---

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 整体架构](#2-整体架构)
- [3. 包结构与依赖关系](#3-包结构与依赖关系)
- [4. 核心包详解](#4-核心包详解)
  - [4.1 `@earendil-works/pi-ai`](#41-earendil-workspi-ai)
  - [4.2 `@earendil-works/pi-agent-core`](#42-earendil-workspi-agent-core)
  - [4.3 `@earendil-works/pi-coding-agent`](#43-earendil-workspi-coding-agent)
  - [4.4 `@earendil-works/pi-tui`](#44-earendil-workspi-tui)
  - [4.5 `@earendil-works/pi-orchestrator`](#45-earendil-workspi-orchestrator)
- [5. 关键类与函数参考](#5-关键类与函数参考)
- [6. 核心数据流](#6-核心数据流)
- [7. 配置系统](#7-配置系统)
- [8. 扩展系统](#8-扩展系统)
- [9. 认证与 Provider 体系](#9-认证与-provider-体系)
- [10. 运行方式](#10-运行方式)
- [11. 测试与构建](#11-测试与构建)
- [12. 开发约定](#12-开发约定)

---

## 1. 项目概述

**Pi** 是一个自扩展的极简终端 Coding Agent Harness，由 [Earendil Works](https://github.com/earendil-works) 维护。它的设计哲学是「核心极简，所有功能皆可通过扩展注入」——没有内置 MCP、子 Agent、计划模式、权限弹窗或 TODO 列表，这些都可由扩展实现。

仓库是一个基于 npm workspaces 的 TypeScript Monorepo，包含 5 个包：

| 包名 | 职责 |
|------|------|
| [`@earendil-works/pi-ai`](packages/ai) | 统一多 Provider LLM API（OpenAI/Anthropic/Google/Bedrock 等） |
| [`@earendil-works/pi-agent-core`](packages/agent) | 带 Tool 调用与会话状态机的 Agent 运行时 |
| [`@earendil-works/pi-coding-agent`](packages/coding-agent) | 交互式 Coding Agent CLI（核心产品） |
| [`@earendil-works/pi-tui`](packages/tui) | 基于差分渲染的终端 UI 库 |
| [`@earendil-works/pi-orchestrator`](packages/orchestrator) | 实验性多进程编排器（pre-release） |

- **语言**: TypeScript（严格 ESM，Node ≥ 22.19.0）
- **许可证**: MIT
- **当前版本**: 锁步版本号 `0.80.10`（所有包共享同一版本）
- **主仓库入口**: [README.md](README.md)、[AGENTS.md](AGENTS.md)、[CONTRIBUTING.md](CONTRIBUTING.md)

---

## 2. 整体架构

Pi 的分层非常清晰，从下到上：

```
┌────────────────────────────────────────────────────────────────┐
│            用户 / CLI / RPC Client / SDK 调用方                  │
└─────────────────────────────┬──────────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────────┐
│  packages/coding-agent  (pi CLI)                                │
│  ─ AgentSession / AgentSessionRuntime / Services                │
│  ─ 内置 Tools: read/bash/edit/write/grep/find/ls                │
│  ─ Extension Runner / Loader / Wrapper                          │
│  ─ ModelRuntime / ModelRegistry / SessionManager                │
│  ─ 四种模式: Interactive TUI / Print / JSON / RPC                │
│  ─ Settings / Keybindings / Prompt Templates / Skills            │
└──────────────┬─────────────────────────┬────────────────────────┘
               │                         │
┌──────────────▼─────────────┐ ┌─────────▼───────────────────────┐
│  packages/agent             │ │  packages/tui                   │
│  (pi-agent-core)            │ │  (差分渲染 TUI、Editor、组件库)   │
│  ─ Agent / agentLoop        │ └─────────────────────────────────┘
│  ─ AgentHarness (Session/   │
│      Compaction/Skills)     │
└──────────────┬──────────────┘
               │
┌──────────────▼─────────────────────────────────────────────────┐
│  packages/ai  (pi-ai)                                           │
│  ─ Models / Provider / ImagesModels                             │
│  ─ Auth (CredentialStore / OAuth / env vars)                    │
│  ─ 10 个 API 实现 (anthropic-messages, openai-responses, …)     │
│  ─ 39 个内置 Provider 工厂                                      │
│  ─ 动态模型目录 / 跨 Provider handoff / 上下文溢出检测            │
└────────────────────────────────────────────────────────────────┘
               ▲
               │ (实验性)
┌──────────────┴─────────────────────────────────────────────────┐
│  packages/orchestrator  (pi-orchestrator)                        │
│  ─ 多 pi 进程编排 (supervisor / IPC / RPC)                       │
└────────────────────────────────────────────────────────────────┘
```

### 分层原则

1. **pi-ai** 只关心 LLM 协议、Provider 认证、消息格式转换，不依赖任何上层包。
2. **pi-agent-core** 依赖 pi-ai，提供有状态 Agent 循环与持久化 Harness，但不感知 CLI 或 TUI。
3. **pi-tui** 是纯 TUI 库，不依赖业务逻辑，可独立使用。
4. **pi-coding-agent** 是终端产品，组合前三者并加上扩展、Tools、会话管理、设置等。
5. **pi-orchestrator** 是实验性的多进程编排器，依赖 coding-agent。

---

## 3. 包结构与依赖关系

### 3.1 内部依赖图

```
pi-ai            ← (无内部依赖)
pi-agent-core    → pi-ai
pi-tui           ← (无内部依赖)
pi-coding-agent  → pi-ai, pi-agent-core, pi-tui
pi-orchestrator  → pi-coding-agent
```

### 3.2 关键外部依赖

| 包 | 主要外部依赖 |
|----|-------------|
| pi-ai | `@anthropic-ai/sdk`, `openai`, `@google/genai`, `@mistralai/mistralai`, `@aws-sdk/client-bedrock-runtime`, `typebox`, `partial-json`, `https-proxy-agent` |
| pi-agent-core | `typebox`, `ignore`, `yaml` |
| pi-coding-agent | `chalk`, `cross-spawn`, `diff`, `glob`, `highlight.js`, `hosted-git-info`, `jiti`, `minimatch`, `proper-lockfile`, `semver`, `undici`, `yaml`, `@silvia-oddywer/photon-node` |
| pi-tui | `marked`, `get-east-asian-width` |
| pi-orchestrator | (仅依赖 coding-agent) |

### 3.3 Workspace 配置

根 [package.json](package.json) 使用 npm workspaces，所有包共享 `package-lock.json`；`pi-coding-agent` 额外有 `npm-shrinkwrap.json` 用于发布时锁定传递依赖。所有直接外部依赖均被 pin 到精确版本（`save-exact=true`，`min-release-age=2`）。

---

## 4. 核心包详解

### 4.1 `@earendil-works/pi-ai`

**职责**: 跨 39+ LLM Provider 的统一抽象层，提供流式协议、Tool 调用、Token/Cost 跟踪、自动认证解析、跨 Provider handoff、图像生成。

**入口**: [packages/ai/src/index.ts](packages/ai/src/index.ts) — 副作用自由的根入口，仅导出核心类型与 `Models`/`createModels`/`createProvider` 等核心 API。

#### 核心抽象

1. **`Models` 集合** ([packages/ai/src/models.ts](packages/ai/src/models.ts)) — Provider 注册中心，所有请求经过它路由。
2. **`Provider<TApi>`** — 每个 Provider 拥有 `id`、`auth`、`getModels()`、`stream()`/`streamSimple()`。
3. **API 实现** ([packages/ai/src/api/](packages/ai/src/api/)) — 10 个 wire-protocol 适配器，每个导出 `stream`/`streamSimple` 函数。
4. **Auth** ([packages/ai/src/auth/](packages/ai/src/auth/)) — `CredentialStore`、`OAuthAuth`、`ApiKeyAuth`、`AuthInteraction` 协议。
5. **`ImagesModels`** ([packages/ai/src/images-models.ts](packages/ai/src/images-models.ts)) — 图像生成并行 API 表面。

#### 支持的 Provider

OpenAI、Anthropic、Google Gemini、Vertex AI、Azure OpenAI、Amazon Bedrock、Mistral、Groq、Cerebras、xAI、OpenRouter、Cloudflare、DeepSeek、NVIDIA NIM、Together AI、Hugging Face、Fireworks、GitHub Copilot（OAuth）、OpenAI Codex（OAuth）、MiniMax、Moonshot AI、Kimi、Qwen、Xiaomi MiMo、ZAI、OpenCode Zen/Go、Vercel AI Gateway、Ant Ling 等，以及任意 OpenAI 兼容端点（Ollama、vLLM 等）。

#### 请求处理流程

```
models.stream(model, ctx, opts)
  → lazyStream (同步返回，异步解析)
    → applyAuth:
        ├─ CredentialStore 读 (stored OAuth/api_key)
        │   └─ OAuth 过期则 store.modify 锁内刷新
        ├─ 无 stored credential → envApiKeyAuth.resolve(env vars)
        └─ 合并 headers: auth → options → transformHeaders
    → provider.stream (lazyApi 动态导入对应 .lazy.ts)
        → API module (e.g. anthropic-messages.ts)
            → SDK client + message 转换 + 流事件发射
```

**关键特性**:
- 流式协议永不抛错：所有错误以 `error` 事件 + `stopReason: "error" | "aborted"` 形式返回。
- TypeBox Schema 用于 Tool 参数定义与验证（`validateToolCall`）。
- 跨 Provider 消息转换（thinking 块自动转为 `<thinking>` 文本）。
- Context overflow 自动检测（[packages/ai/src/utils/overflow.ts](packages/ai/src/utils/overflow.ts)，覆盖 23+ Provider 错误模式）。
- 浏览器可用（无 env var 时必须显式传 `apiKey` 或注入 `CredentialStore`）。
- OAuth 自动刷新（`anthropic.ts` 用 `node:http` 回调服务器；其他用 device-code 或 PKCE）。

---

### 4.2 `@earendil-works/pi-agent-core`

**职责**: 在 pi-ai 之上构建有状态 Agent 运行时，支持 Tool 调用、消息转换、流事件、并行/顺序 Tool 执行、Steering/Follow-up 队列、持久化会话树、Compaction、Skills、Prompt Templates、Hooks。

**入口**:
- [packages/agent/src/index.ts](packages/agent/src/index.ts) — 环境无关的主入口。
- [packages/agent/src/node.ts](packages/agent/src/node.ts) — 额外导出 `NodeExecutionEnv`。

#### 两层架构

**底层（bare runtime）**: `Agent` / `agentLoop` — 单次 in-memory 循环。

**上层（harness）**: `AgentHarness` — 包装 `runAgentLoop`，增加 JSONL 会话树持久化、Compaction、Branch Summary、Skills、Prompt Templates、Hook 系统、Provider 请求钩子、三队列模型。

#### `Agent` 类（[packages/agent/src/agent.ts](packages/agent/src/agent.ts)）

有状态 Agent 包装器。核心 API：

| 方法 | 用途 |
|------|------|
| `subscribe(listener)` | 订阅 `AgentEvent`，按注册顺序 await |
| `prompt(text \| message, images?)` | 触发一轮对话 |
| `continue()` | 从当前上下文继续（最后一条须为 user/toolResult） |
| `steer(message)` | 工具执行中插入 steering 消息 |
| `followUp(message)` | 工具完成后排队 follow-up |
| `clearAllQueues()` | 清空所有队列 |
| `abort()` | 取消当前操作 |
| `waitForIdle()` | 等待运行完成 |
| `reset()` | 清空 transcript 与队列 |

`AgentOptions` 支持 `initialState`、`convertToLlm`、`transformContext`、`streamFn`、`getApiKey`、`beforeToolCall`、`afterToolCall`、`prepareNextTurn`、`steeringMode`、`followUpMode`、`sessionId`、`thinkingBudgets`、`toolExecution`（`"parallel"` 默认 / `"sequential"`）等。

#### `agentLoop` 算法（[packages/agent/src/agent-loop.ts](packages/agent/src/agent-loop.ts)）

```
1. 注入 prompts 到 context，发射 agent_start + 每个 prompt 的 message_start/end
2. 主循环:
   a. 检查 steering 队列（每轮一次）
   b. 发射 turn_start
   c. 注入 pending steering 消息
   d. streamAssistantResponse():
      - transformContext → convertToLlm → streamFn
      - 映射 pi-ai 事件到 message_start/update/end
   e. 若 stopReason ∈ {error, aborted}: 发射 turn_end + agent_end 并返回
   f. 收集 toolCalls
   g. 若 stopReason === "length": 用 failToolCallsFromTruncatedMessage 标记所有 tool 失败
      否则: executeToolCalls (parallel 或 sequential)
   h. 注入 toolResult 消息，发射 turn_end
   i. 调用 prepareNextTurn / shouldStopAfterTurn
3. 检查 followUp 队列，若有则注入并 goto 2
4. 发射 agent_end(newMessages)
```

**Tool 执行细节**:
- **Preflight** (`prepareToolCall`): 查找 tool → `prepareArguments` → TypeBox 验证 → `beforeToolCall` hook（可阻止）。
- **并行模式**: preflight 顺序执行，execute 并发 `Promise.all`，`tool_execution_end` 按完成序发射，`toolResult` 消息按源序发射。
- **顺序模式**: 逐个执行，可中途 abort。
- **`terminate: true` 提示**: 仅当批次中所有 tool 都设置才生效，跳过自动 follow-up LLM 调用。

#### `AgentHarness`（[packages/agent/src/harness/agent-harness.ts](packages/agent/src/harness/agent-harness.ts)）

**不使用 `Agent` 类**，而是直接调用 `runAgentLoop` 并提供：

- **会话树持久化** — 所有消息、模型变更、思考级别变更、Tool 列表变更、Label、Compaction、Branch Summary、Leaf 指针都是 append-only JSONL 条目。
- **Compaction** — `prepareCompaction` 找切割点 → `compact()` 调用 LLM 生成结构化摘要 → 写入 `CompactionEntry`，后续 `buildContext` 自动折叠历史。
- **Branch 导航** — `navigateTree()` 走到目标 entry，可选 `generateBranchSummary()` 摘要被放弃的分支，`session.moveTo()` 切换 leaf。
- **Skills & Prompt Templates** — 从磁盘加载，附在 system prompt 中（`formatSkillsForSystemPrompt`），可显式 `/skill:name` 或 `/templatename` 调用。
- **Hook 系统** — `on(type, handler)` 注册类型特定 hook，返回值反馈到循环（如 `before_agent_start` 注入消息、`tool_result` 覆盖结果）。
- **三队列模型** — `steerQueue`（轮次中）、`followUpQueue`（停止后）、`nextTurnQueue`（下个 prompt 前），各有 `QueueMode`。
- **Provider Hooks** — `before_provider_request`、`before_provider_payload`、`after_provider_response` 用于在不替换 streamFn 的情况下注入/审计请求。
- **Phase 状态机** — `idle | turn | compaction | branch_summary | retry`，非 idle 阶段的变更延迟到下个 flush 点。

#### 会话树结构

```
SessionHeader { type: "session", version: 3, id, timestamp, cwd, metadata? }
SessionTreeEntry (line 2+) union of:
  MessageEntry | ThinkingLevelChangeEntry | ModelChangeEntry |
  ActiveToolsChangeEntry | CompactionEntry | BranchSummaryEntry |
  CustomEntry | CustomMessageEntry | LabelEntry | SessionInfoEntry |
  LeafEntry    # targetId 指向当前 leaf，实现 in-place 分支
```

实现:
- [packages/agent/src/harness/session/jsonl-storage.ts](packages/agent/src/harness/session/jsonl-storage.ts) — 文件存储（`uuidv7().slice(-8)` 作为 entry id）。
- [packages/agent/src/harness/session/jsonl-repo.ts](packages/agent/src/harness/session/jsonl-repo.ts) — cwd 编码为目录，session 文件名 `{timestamp}_{sessionId}.jsonl`。
- [packages/agent/src/harness/session/memory-storage.ts](packages/agent/src/harness/session/memory-storage.ts) — 测试用 in-memory。

#### Compaction 内部

[packages/agent/src/harness/compaction/compaction.ts](packages/agent/src/harness/compaction/compaction.ts):

- `DEFAULT_COMPACTION_SETTINGS = { enabled: true, reserveTokens: 16384, keepRecentTokens: 20000 }`
- `estimateTokens(message)` — 保守 chars/4 启发式，图像估 4800 chars。
- `shouldCompact(contextTokens, contextWindow, settings)` — `> contextWindow - reserveTokens`
- `findCutPoint` — 从后向前累计 token，找到一个 user/assistant/custom/branch_summary entry，永不切在 toolResult 中；若切在 turn 中间则标记 `isSplitTurn` 并额外生成 turn-prefix summary。
- `compact()` — 调用 `models.completeSimple`，使用 `SUMMARIZATION_PROMPT` 或 `UPDATE_SUMMARIZATION_PROMPT`（增量），追加 `<read-files>` / `<modified-files>` 块（基于 `extractFileOpsFromMessage`）。

---

### 4.3 `@earendil-works/pi-coding-agent`

**职责**: 终端 Coding Agent CLI（即用户运行的 `pi` 命令）。组合 pi-ai/pi-agent-core/pi-tui，加上扩展、Tools、会话管理、设置、信任、Telemetry。

**入口**:
- [packages/coding-agent/src/cli.ts](packages/coding-agent/src/cli.ts) — Node/Bun 启动入口（设 `process.title`，调用 `main(argv.slice(2))`）。
- [packages/coding-agent/src/main.ts](packages/coding-agent/src/main.ts) — 主流程编排。
- [packages/coding-agent/src/index.ts](packages/coding-agent/src/index.ts) — SDK 公共导出。

#### 启动流程

```
cli.ts → main(argv)
  ├─ runMigrations(cwd)              # oauth.json→auth.json, sessions-per-cwd 修复等
  ├─ parseArgs(argv)                 # 解析 CLI flags
  ├─ resolveAppMode()                # → "rpc"|"json"|"print"|"interactive"
  ├─ resolveSession (continue/resume/session/fork)
  └─ createAgentSessionRuntime(createRuntime, {...})  # 每会话工厂
       │
       └─ createRuntime (per-cwd):
          ├─ resolveProjectTrusted()  # 信任流：扩展可拦截
          ├─ createAgentSessionServices(cwd, agentDir, ...)
          │   ├─ ModelRuntime.create()
          │   ├─ SettingsManager.create(cwd, agentDir, ...)
          │   ├─ DefaultResourceLoader (extensions/skills/prompts/themes/context-files)
          │   └─ 注册 extension 期间排队的 providers
          ├─ createAgentSessionFromServices(...)
          └─ applyRuntimeApiKey (--api-key 注入)

  → appMode === "rpc"        ? runRpcMode(runtime)
  : appMode === "interactive" ? new InteractiveMode(runtime).run()
  :                            runPrintMode(runtime, {mode: ...})
```

#### `AgentSession` 类（[packages/coding-agent/src/core/agent-session.ts](packages/coding-agent/src/core/agent-session.ts)）

约 3270 行，是核心协调器。注入 `agent`（pi-agent-core）、`sessionManager`、`settingsManager`、`modelRuntime`、`resourceLoader`、`extensionRunner`。

**关键方法**:
- `prompt(message, options)` — 发射 `input` 事件 → 扩展 skill/template → 验证 model+auth → `_runAgentPrompt`。
- `_runAgentPrompt` — 设 `_isAgentRunActive=true`，调 `agent.prompt(...)`，循环 `_handlePostAgentRun`（重试/Compaction/排队消息）。
- `compact(options)` — 手动 Compaction；触发 `session_before_compact` 扩展钩子（可取消）。
- `navigateTree(targetId, options)` — 移动 leaf 指针；可选写 `BranchSummaryEntry`；触发 `session_before_tree`。
- `executeBash(...)` — 流式 bash 执行（用于交互/RPC 模式）。
- `reload()` — 发射 `session_shutdown`，重载 settings+resources，重建 runtime。
- `_buildRuntime` — 创建 `ExtensionRunner`，调 `bindCore(...)` 把扩展动作接到 runtime。

**生命周期钩子**（构造时安装）:
- `_installAgentToolHooks` — `agent.beforeToolCall`/`afterToolCall` 路由到 `extensionRunner.emitToolCall`/`emitToolResult`。
- `_installAgentNextTurnRefresh` — 重写 `agent.prepareNextTurnWithContext`，每轮重建 system prompt（含 skills/tools/context files）并刷新 tool 注册表。

#### Services Pattern

[packages/coding-agent/src/core/agent-session-services.ts](packages/coding-agent/src/core/agent-session-services.ts) — 把 cwd-bound 服务（`modelRuntime`、`settingsManager`、`resourceLoader`、`diagnostics`）打包成 `AgentSessionServices`，便于切换会话时重建。

#### Runtime Replacement

[packages/coding-agent/src/core/agent-session-runtime.ts](packages/coding-agent/src/core/agent-session-runtime.ts) — `AgentSessionRuntime` 持有当前 `AgentSession` 及其服务；`switchSession`/`newSession`/`fork`/`importFromJsonl` 每个都发射 `session_before_*` 钩子（可取消），拆卸当前会话（`teardownCurrent` → `session_shutdown` + `dispose`），用新 cwd/sessionManager 调用工厂，应用新 runtime，重绑扩展。

**关键设计**: 切换前捕获的扩展 `ExtensionContext` 在 `runner.invalidate()` 后所有方法抛错（`assertActive()` 守卫）。

#### 内置 Tools

[packages/coding-agent/src/core/tools/](packages/coding-agent/src/core/tools/) — `ToolName = "read" | "bash" | "edit" | "write" | "grep" | "find" | "ls"`。

每个 Tool 遵循相同模式：
1. `*Operations` 接口定义可插拔的远程操作（用于 RPC/SSH/Container）：
   - `ReadOperations` (`readFile`/`access`/`detectImageMimeType`)
   - `BashOperations.exec(command, cwd, {onData, signal, timeout, env})`
   - `EditOperations` (`readFile`/`writeFile`/`access`)
   - `WriteOperations` (`writeFile`/`mkdir`)
   - `LsOperations` (`exists`/`stat`/`readdir`)
2. `createLocal*Operations(...)` — 默认基于 `node:fs`/`node:child_process` 的本地实现。
3. TypeBox Schema 声明参数。
4. `create*Tool(operations, options)` 工厂返回 `ToolDefinition`。

**ToolDefinition** 字段: `name`、`label`、`description`、`promptSnippet`、`promptGuidelines`、`parameters`、`prepareArguments`、`execute(args, ctx, signal, onUpdate)`、`renderCall(toolCall, ctx)`、`renderResult(toolCall, result, ctx)`、`renderShell`、`executionMode`。

**Wrapper Pattern** ([packages/coding-agent/src/core/tools/tool-definition-wrapper.ts](packages/coding-agent/src/core/tools/tool-definition-wrapper.ts)):
- `wrapToolDefinition(definition, ctxFactory)` — 把富 `ToolDefinition` 适配为 pi-agent-core 的薄 `AgentTool`，每次执行时通过 `ctxFactory()` 注入新鲜 `ExtensionContext`。
- `wrapRegisteredTool(registeredTool, runner)` — 把扩展注册的 Tool 包装为 `AgentTool`，捕获执行中新增的 Tool（返回 `addedToolNames`）。

**支持模块**:
- `edit-diff.ts` — `detectLineEnding`、`normalizeToLF`、`normalizeForFuzzyMatch`（智能引号/破折号/空格）。
- `output-accumulator.ts` — 流式 UTF-8 解码器，有界 tail 缓冲 + 临时文件溢出（`DEFAULT_MAX_BYTES = 50KB`）。
- `truncate.ts` — `truncateHead`/`truncateTail`/`truncateLine`，返回富 `TruncationResult`。
- `path-utils.ts` — `resolveReadPathAsync` 带 macOS Unicode-folding 变体（NFD、AM/PM 窄空格、卷曲引号）。
- `file-mutation-queue.ts` — `withFileMutationQueue` 按 realpath 序列化同文件 mutation，跨文件并行。
- `render-utils.ts` — `shortenPath`（home→`~`）、`linkPath`（OSC 8 超链接）。

#### 模型运行时层

[packages/coding-agent/src/core/model-runtime.ts](packages/coding-agent/src/core/model-runtime.ts) — `ModelRuntime` 实现 pi-ai 的 `Models` 接口。

**三层组合**:
1. `builtins` — 来自 `@earendil-works/pi-ai/providers/all` 的内置 Provider（每个经 `withRemoteCatalog` 叠加 pi.dev 目录更新，除 `radius`）。
2. `nativeExtensionProviders` — 扩展直接注册的 `Provider` 对象。
3. `extensionProviders` — 扩展注册的 `ProviderConfigInput`，叠加在内置 Provider 上。

`recomposeProvider(id)` 调用 [packages/coding-agent/src/core/provider-composer.ts](packages/coding-agent/src/core/provider-composer.ts) 的 `composeModelProvider`，组合时失败则回退 base provider 并记录 `compositionErrors`。

**关键方法**: `getAuth`、`stream`/`streamSimple`、`refresh`、`registerProvider`/`registerNativeProvider`/`unregisterProvider`、`reloadConfig`、`setRuntimeApiKey`/`removeRuntimeApiKey`（运行时临时 key）、`getAvailable`、`getProviderAuthStatus`、`getError`。

**Snapshot** 模型: `{all, available, configuredProviders, storedProviders, auth}`，写操作强制刷新可用性，读操作合并到 pending refresh。

#### 会话管理

[packages/coding-agent/src/core/session-manager.ts](packages/coding-agent/src/core/session-manager.ts) — Append-only JSONL 树存储，`CURRENT_SESSION_VERSION = 3`。

**Entry 类型**: `SessionMessageEntry`、`ThinkingLevelChangeEntry`、`ModelChangeEntry`、`CompactionEntry`、`BranchSummaryEntry`、`CustomEntry`、`CustomMessageEntry`、`LabelEntry`、`SessionInfoEntry`。

**Migration**: `migrateV1ToV2`（线性→树，加 id/parentId）、`migrateV2ToV3`（`hookMessage`→`custom` role）。

**关键 API**:
- `appendMessage`/`appendThinkingLevelChange`/`appendModelChange`/`appendCompaction`/`appendCustomEntry`/`appendLabel`
- `branch(branchFromId)` — 仅移动 leaf 指针，无文件复制
- `branchWithSummary(...)` — 先写 `BranchSummaryEntry` 再移 leaf
- `createBranchedSession(leafId, options)` — 写新 JSONL，仅含 path + labels
- 静态工厂: `create`/`open`/`continueRecent`/`inMemory`/`forkFrom`
- `list(cwd, sessionDir)` / `listAll(sessionDir)` — 并发读 header（`MAX_CONCURRENT_SESSION_INFO_LOADS = 10`）

**Privacy 优化**: `_persist()` 延迟到首条 assistant 消息到达，失败 prompt 不留痕迹。

#### 设置与 Keybindings

[packages/coding-agent/src/core/settings-manager.ts](packages/coding-agent/src/core/settings-manager.ts):
- **`Settings`** 字段: `provider`、`model`、`theme`、`transport`、`steeringMode`、`followUpMode`、`compaction`、`retry`、`packages`、`extensions`、`skills`、`prompts`、`themes`、`terminal`、`images`、`thinkingBudgets`、`enableInstallTelemetry`、`defaultProjectTrust`、`httpProxy`、`httpIdleTimeoutMs` 等。
- 加载顺序: `~/.pi/agent/settings.json`（global）→ `<cwd>/.pi/settings.json`（project，仅信任时加载），`deepMergeSettings` 合并。
- `migrateSettings`: `queueMode`→`steeringMode`、`websockets`→`transport`、`retry.maxDelayMs`→`retry.provider.maxRetryDelayMs`。

[packages/coding-agent/src/core/keybindings.ts](packages/coding-agent/src/core/keybindings.ts):
- `AppKeybindings` 扩展 `TuiKeybindings` 加 `"app.*"` 命名空间（`app.interrupt`、`app.exit`、`app.thinking.cycle`、`app.model.cycleForward` 等）。
- `KEYBINDING_NAME_MIGRATIONS` 把旧名（`quit`、`toggle-thinking`）映射到新名。
- 不允许硬编码按键检查（必须加到 `DEFAULT_EDITOR_KEYBINDINGS` 或 `DEFAULT_APP_KEYBINDINGS`）。

#### Skills / Prompt Templates / Slash Commands

- **Skills** ([packages/coding-agent/src/core/skills.ts](packages/coding-agent/src/core/skills.ts)) — 遵循 [agentskills.io](https://agentskills.io) 标准，从 `~/.pi/agent/skills/`、`~/.agents/skills/`、`<cwd>/.pi/skills/`、`<cwd>/.agents/skills/`（向上遍历父目录）加载。Name 必须匹配 `^[a-z0-9-]+$`，最多 64 字符。`disableModelInvocation: true` 的 skill 仅 `/skill:name` 调用，不出现在 system prompt。
- **Prompt Templates** ([packages/coding-agent/src/core/prompt-templates.ts](packages/coding-agent/src/core/prompt-templates.ts)) — Markdown 文件，支持 YAML frontmatter（`description`、`argument-hint`），参数替换 `$1`、`$@`、`$ARGUMENTS`、`${@:N}`、`${@:N:L}`、`${N:-default}`。
- **Slash Commands** ([packages/coding-agent/src/core/slash-commands.ts](packages/coding-agent/src/core/slash-commands.ts)) — `BUILTIN_SLASH_COMMANDS`: `settings`、`model`、`scoped-models`、`export`、`import`、`share`、`copy`、`name`、`session`、`changelog`、`hotkeys`、`fork`、`clone`、`tree`、`trust`、`login`、`logout`、`new`、`compact`、`resume`、`reload`、`quit`。扩展注册命令通过 `/name` 触发。

#### 事件总线与执行

- [packages/coding-agent/src/core/event-bus.ts](packages/coding-agent/src/core/event-bus.ts) — `createEventBus()` 基于 `EventEmitter`，handler 包裹 try/catch 不让错误冒泡。
- [packages/coding-agent/src/core/exec.ts](packages/coding-agent/src/core/exec.ts) — `execCommand` 用 `spawn`（`shell: false`，须可执行路径），支持 abort + timeout。
- [packages/coding-agent/src/core/bash-executor.ts](packages/coding-agent/src/core/bash-executor.ts) — 流式 bash 执行，滚动缓冲 + 临时文件溢出。
- [packages/coding-agent/src/core/http-dispatcher.ts](packages/coding-agent/src/core/http-dispatcher.ts) — 安装 `undici.EnvHttpProxyAgent`，对齐 `globalThis.fetch`（避开 Node 26.0 bundled-fetch 解压 bug）。

#### 四种运行模式

| 模式 | 触发 | 入口 | 输出 |
|------|------|------|------|
| Interactive TUI | (default) | `new InteractiveMode(runtime).run()` | 全 TUI 渲染 |
| Print | `-p`/`--print` | `runPrintMode(runtime, {mode: "text"})` | 最终 assistant 文本 |
| JSON | `--mode json` | `runPrintMode(runtime, {mode: "json"})` | 所有事件 NDJSON |
| RPC | `--mode rpc` | `runRpcMode(runtime)` | JSONL 双向协议 |

**RPC 模式** ([packages/coding-agent/src/modes/rpc/rpc-mode.ts](packages/coding-agent/src/modes/rpc/rpc-mode.ts)):
- `takeOverStdout()` 把 `console.*` 重定向到 stderr，RPC 消息走原始 stdout。
- 严格 LF 分隔 JSONL（[packages/coding-agent/src/modes/rpc/jsonl.ts](packages/coding-agent/src/modes/rpc/jsonl.ts)，故意不用 Node `readline` 因其按 U+2028/U+2029 分隔会破坏 JSON 字符串）。
- 命令 (`RpcCommand`): `prompt`、`steer`、`follow_up`、`abort`、`new_session`、`get_state`、`set_model`、`cycle_model`、`get_available_models`、`set_thinking_level`、`compact`、`bash`、`switch_session`、`fork`、`clone`、`get_entries`、`get_tree` 等。
- 扩展 UI 请求通过 `RpcExtensionUIRequest`/`Response` 往返（按 `crypto.randomUUID()` 关联，有 timeout/abort）。

#### 信任与 Telemetry

- [packages/coding-agent/src/core/trust-manager.ts](packages/coding-agent/src/core/trust-manager.ts) — `ProjectTrustStore` 文件 JSON（`~/.pi/agent/project-trust.json`），canonicalize 后 walking-up 查找最近祖先决策。
- [packages/coding-agent/src/core/project-trust.ts](packages/coding-agent/src/core/project-trust.ts) — `resolveProjectTrusted` 流程: trustOverride → 无 trust-requiring 资源即自动 trust → 扩展 `project_trust` 事件（首个非 undecided 胜）→ trustStore 查 → `defaultProjectTrust` fallback → TUI 提示。
- Bootstrap: `DefaultResourceLoader.reload()` 先加载 user/global + CLI 扩展（强制 `setProjectTrusted(false)`），再 `resolveProjectTrust`，再加载 project-local 扩展。
- [packages/coding-agent/src/core/telemetry.ts](packages/coding-agent/src/core/telemetry.ts) — `PI_TELEMETRY=1` 或 settings `enableInstallTelemetry: true` 启用。
- [packages/coding-agent/src/core/provider-attribution.ts](packages/coding-agent/src/core/provider-attribution.ts) — 对 OpenRouter/NVIDIA/Cloudflare/OpenCode 附加 attribution headers（仅在 telemetry 启用时）。

---

### 4.4 `@earendil-works/pi-tui`

**职责**: 基于差分渲染的终端 UI 库，可独立用于任何 TUI 应用。

**入口**: [packages/tui/src/index.ts](packages/tui/src/index.ts)。

**核心模块**:
- [packages/tui/src/tui.ts](packages/tui/src/tui.ts) — `TUI` 主类，差分渲染循环。
- [packages/tui/src/terminal.ts](packages/tui/src/terminal.ts) — `ProcessTerminal` 抽象 raw stdin/stdout。
- [packages/tui/src/keybindings.ts](packages/tui/src/keybindings.ts) — `TuiKeybindingsManager`、`matchesKey`、`DEFAULT_EDITOR_KEYBINDINGS`、`DEFAULT_APP_KEYBINDINGS`。
- [packages/tui/src/editor-component.ts](packages/tui/src/editor-component.ts) + `components/editor.ts` — 多行编辑器，支持 `@` 文件 fuzzy 搜索、Tab 补全、Shift+Enter 多行、Ctrl+G 外部编辑器。
- `components/` 目录: `box`、`cancellable-loader`、`input`、`loader`、`markdown`（基于 `marked`）、`select-list`、`settings-list`、`spacer`、`text`、`truncated-text`、`image`（终端图像渲染）。
- [packages/tui/src/fuzzy.ts](packages/tui/src/fuzzy.ts) — `fuzzyFilter` 模糊匹配。
- [packages/tui/src/kill-ring.ts](packages/tui/src/kill-ring.ts) + `undo-stack.ts` — Emacs 风格剪贴环与撤销栈。
- [packages/tui/src/terminal-colors.ts](packages/tui/src/terminal-colors.ts) + `terminal-image.ts` — 终端颜色与图像协议（含 Sixel/iTerm2 支持）。

**关键外部依赖**: `marked`（Markdown 渲染）、`get-east-asian-width`（CJK 宽度计算）、`@xterm/headless`（仅测试时用）。

---

### 4.5 `@earendil-works/pi-orchestrator`

**职责**: 实验性多进程编排器，pre-release，API 不稳定。

**入口**:
- [packages/orchestrator/src/index.ts](packages/orchestrator/src/index.ts) — 导出全部模块。
- [packages/orchestrator/src/cli.ts](packages/orchestrator/src/cli.ts) — CLI 入口。

**模块**:
- [packages/orchestrator/src/supervisor.ts](packages/orchestrator/src/supervisor.ts) — 进程 supervisor。
- [packages/orchestrator/src/rpc-process.ts](packages/orchestrator/src/rpc-process.ts) — 子 pi 进程的 RPC 包装。
- `ipc/` — `server.ts`、`client.ts`、`protocol.ts` IPC 协议。
- [packages/orchestrator/src/storage.ts](packages/orchestrator/src/storage.ts) — 状态存储。
- [packages/orchestrator/src/handler.ts](packages/orchestrator/src/handler.ts) — 命令处理器。

**依赖**: 仅依赖 `pi-coding-agent`。用于在多 pi 进程间协调工作（参见 `pi-orchestrator --help`）。

---

## 5. 关键类与函数参考

### 5.1 pi-ai

| 符号 | 位置 | 说明 |
|------|------|------|
| `Models` | [models.ts](packages/ai/src/models.ts) | Provider 集合接口，所有请求路由中心 |
| `MutableModels` | [models.ts](packages/ai/src/models.ts) | 增加 `setProvider`/`deleteProvider` |
| `ModelsImpl` | [models.ts](packages/ai/src/models.ts) | 默认实现，构造时注入 `CredentialStore`、`ModelsStore`、`AuthContext` |
| `Provider<TApi>` | [models.ts](packages/ai/src/models.ts) | Provider 接口，含 `id`、`auth`、`getModels()`、`stream()`/`streamSimple()` |
| `createModels(options)` | [models.ts](packages/ai/src/models.ts) | 工厂 |
| `createProvider(input)` | [models.ts](packages/ai/src/models.ts) | 从 parts 构建 Provider（id/auth/models/api） |
| `envApiKeyAuth(name, envVars)` | [auth/helpers.ts](packages/ai/src/auth/helpers.ts) | 标准 env-var api-key 认证 helper |
| `lazyOAuth({ name, load })` | [auth/helpers.ts](packages/ai/src/auth/helpers.ts) | 延迟加载 OAuth 实现 |
| `CredentialStore` | [auth/types.ts](packages/ai/src/auth/types.ts) | 凭证存储接口，`modify()` 是唯一写路径 |
| `InMemoryCredentialStore` | [auth/credential-store.ts](packages/ai/src/auth/credential-store.ts) | 默认 in-memory 实现，按 provider id 序列化 |
| `resolveProviderAuth(...)` | [auth/resolve.ts](packages/ai/src/auth/resolve.ts) | 共享 auth 解析器（OAuth 锁内刷新 + api_key fallback） |
| `validateToolCall` / `validateToolArguments` | [utils/validation.ts](packages/ai/src/utils/validation.ts) | TypeBox 验证 Tool 参数 |
| `isContextOverflow(message, ctxWindow)` | [utils/overflow.ts](packages/ai/src/utils/overflow.ts) | 跨 Provider 溢出检测 |
| `isRetryableAssistantError(message)` | [utils/retry.ts](packages/ai/src/utils/retry.ts) | 可重试错误分类 |
| `combineAbortSignals(signals)` | [utils/abort-signals.ts](packages/ai/src/utils/abort-signals.ts) | 多信号合并 |
| `parseStreamingJson` / `parseJsonWithRepair` | [utils/json-parse.ts](packages/ai/src/utils/json-parse.ts) | 流式 JSON 容错解析 |
| `EventStream<T, R>` | [utils/event-stream.ts](packages/ai/src/utils/event-stream.ts) | 通用 push-based 异步迭代 |
| `AssistantMessageEventStream` | [utils/event-stream.ts](packages/ai/src/utils/event-stream.ts) | pi-ai 流协议实现 |
| `lazyStream(model, setup)` | [api/lazy.ts](packages/ai/src/api/lazy.ts) | 同步返回 stream，异步 setup 在后 |
| `lazyApi(load)` | [api/lazy.ts](packages/ai/src/api/lazy.ts) | 包装动态导入的 API 模块 |
| `builtinModels()` / `builtinProviders()` | [providers/all.ts](packages/ai/src/providers/all.ts) | 注册所有内置 Provider |
| `getBuiltinModel`/`getBuiltinModels`/`getBuiltinProviders` | [providers/all.ts](packages/ai/src/providers/all.ts) | 静态 catalog 类型化读取 |
| `ImagesModels` / `ImagesModelsImpl` | [images-models.ts](packages/ai/src/images-models.ts) | 图像生成 collection（不抛错，返回 `stopReason: "error"`） |
| `hasApi(model, api)` | [models.ts](packages/ai/src/models.ts) | 动态模型 API 类型缩窄 |

### 5.2 pi-agent-core

| 符号 | 位置 | 说明 |
|------|------|------|
| `Agent` | [agent.ts](packages/agent/src/agent.ts) | 有状态 Agent 包装器 |
| `AgentOptions` | [agent.ts](packages/agent/src/agent.ts) | 构造选项 |
| `MutableAgentState` | [agent.ts](packages/agent/src/agent.ts) | copy-on-write 内部状态 |
| `agentLoop(prompts, ctx, cfg, signal, streamFn)` | [agent-loop.ts](packages/agent/src/agent-loop.ts) | 低层循环流 |
| `agentLoopContinue(ctx, cfg, signal, streamFn)` | [agent-loop.ts](packages/agent/src/agent-loop.ts) | 从已有上下文继续 |
| `runAgentLoop` / `runAgentLoopContinue` | [agent-loop.ts](packages/agent/src/agent-loop.ts) | 内部驱动 |
| `streamProxy(model, ctx, options)` | [proxy.ts](packages/agent/src/proxy.ts) | 代理后端 streamFn |
| `AgentMessage` | [types.ts](packages/agent/src/types.ts) | `Message | CustomAgentMessages[keyof ...]` |
| `AgentState` | [types.ts](packages/agent/src/types.ts) | systemPrompt/model/thinkingLevel/tools/messages + readonly 流式状态 |
| `AgentTool<TParams, TDetails>` | [types.ts](packages/agent/src/types.ts) | 扩展 pi-ai `Tool`，加 `execute`、`executionMode` |
| `AgentToolResult<T>` | [types.ts](packages/agent/src/types.ts) | `{ content, details, usage?, addedToolNames?, terminate? }` |
| `AgentContext` | [types.ts](packages/agent/src/types.ts) | `{ systemPrompt, messages, tools? }` |
| `AgentLoopConfig` | [types.ts](packages/agent/src/types.ts) | 循环配置 + 所有 hook |
| `AgentEvent` | [types.ts](packages/agent/src/types.ts) | 生命周期/turn/message/tool 执行事件 union |
| `AgentHarness<TSkill, TPT, TTool>` | [harness/agent-harness.ts](packages/agent/src/harness/agent-harness.ts) | 上层 orchestrator（持久化 + hooks） |
| `AgentHarnessOptions` | [harness/types.ts](packages/agent/src/harness/types.ts) | 构造选项 |
| `Session<TMetadata>` | [harness/session/session.ts](packages/agent/src/harness/session/session.ts) | Tree-aware session 包装 |
| `SessionStorage<TMetadata>` | [harness/types.ts](packages/agent/src/harness/types.ts) | 低层存储接口 |
| `SessionRepo<TMetadata, TCreate, TList>` | [harness/types.ts](packages/agent/src/harness/types.ts) | Repo 接口（create/open/list/delete/fork） |
| `JsonlSessionStorage` | [harness/session/jsonl-storage.ts](packages/agent/src/harness/session/jsonl-storage.ts) | Append-only JSONL 文件存储 |
| `JsonlSessionRepo` | [harness/session/jsonl-repo.ts](packages/agent/src/harness/session/jsonl-repo.ts) | 文件系统 repo |
| `InMemorySessionStorage`/`Repo` | [harness/session/memory-*.ts](packages/agent/src/harness/session/) | 测试用 |
| `NodeExecutionEnv` | [harness/env/nodejs.ts](packages/agent/src/harness/env/nodejs.ts) | Node `fs`+`child_process` 实现 `ExecutionEnv` |
| `ExecutionEnv` | [harness/types.ts](packages/agent/src/harness/types.ts) | `FileSystem & Shell` |
| `compact(preparation, models, model, ...)` | [harness/compaction/compaction.ts](packages/agent/src/harness/compaction/compaction.ts) | 执行 Compaction |
| `prepareCompaction(entries, settings)` | [harness/compaction/compaction.ts](packages/agent/src/harness/compaction/compaction.ts) | 找切割点 + 收集待摘要消息 |
| `estimateContextTokens(messages)` | [harness/compaction/compaction.ts](packages/agent/src/harness/compaction/compaction.ts) | 估算 token 数（含 usage baseline） |
| `generateBranchSummary(entries, options)` | [harness/compaction/branch-summarization.ts](packages/agent/src/harness/compaction/branch-summarization.ts) | 分支摘要 |
| `loadSkills(env, dirs)` | [harness/skills.ts](packages/agent/src/harness/skills.ts) | 递归加载 SKILL.md |
| `loadPromptTemplates(env, paths)` | [harness/prompt-templates.ts](packages/agent/src/harness/prompt-templates.ts) | 加载 .md 模板 |
| `convertToLlm(messages)` | [harness/messages.ts](packages/agent/src/harness/messages.ts) | harness 版 LLM 消息转换 |
| `executeShellWithCapture(env, cmd, options)` | [harness/utils/shell-output.ts](packages/agent/src/harness/utils/shell-output.ts) | Bash 输出捕获（含临时文件溢出） |
| `truncateHead`/`truncateTail` | [harness/utils/truncate.ts](packages/agent/src/harness/utils/truncate.ts) | 输出截断 |

### 5.3 pi-coding-agent

| 符号 | 位置 | 说明 |
|------|------|------|
| `AgentSession` | [core/agent-session.ts](packages/coding-agent/src/core/agent-session.ts) | 核心协调器 |
| `AgentSessionServices` | [core/agent-session-services.ts](packages/coding-agent/src/core/agent-session-services.ts) | cwd-bound 服务集合 |
| `createAgentSessionServices(...)` | [core/agent-session-services.ts](packages/coding-agent/src/core/agent-session-services.ts) | 服务构建 |
| `createAgentSessionFromServices(...)` | [core/agent-session-services.ts](packages/coding-agent/src/core/agent-session-services.ts) | 从服务构建 session |
| `AgentSessionRuntime` | [core/agent-session-runtime.ts](packages/coding-agent/src/core/agent-session-runtime.ts) | 运行时持有者，支持切换 |
| `createAgentSession(options)` | [core/sdk.ts](packages/coding-agent/src/core/sdk.ts) | 公共 SDK 工厂 |
| `SessionManager` | [core/session-manager.ts](packages/coding-agent/src/core/session-manager.ts) | JSONL 会话树管理 |
| `ModelRuntime` | [core/model-runtime.ts](packages/coding-agent/src/core/model-runtime.ts) | pi-ai `Models` 实现，三层组合 |
| `ModelRegistry` | [core/model-registry.ts](packages/coding-agent/src/core/model-registry.ts) | 同步 facade 暴露给扩展 |
| `ModelConfig` | [core/model-config.ts](packages/coding-agent/src/core/model-config.ts) | `models.json` 不可变快照 |
| `FileModelsStore` | [core/models-store.ts](packages/coding-agent/src/core/models-store.ts) | 文件 `ModelsStore` |
| `composeModelProvider(...)` | [core/provider-composer.ts](packages/coding-agent/src/core/provider-composer.ts) | 三层组合 Provider |
| `withRemoteCatalog(provider, baseUrl)` | [core/remote-catalog-provider.ts](packages/coding-agent/src/core/remote-catalog-provider.ts) | pi.dev catalog 叠加（4h 缓存） |
| `resolveCliModel(...)` / `findInitialModel(...)` | [core/model-resolver.ts](packages/coding-agent/src/core/model-resolver.ts) | CLI 模型解析 |
| `resolveModelScope(patterns, runtime)` | [core/model-resolver.ts](packages/coding-agent/src/core/model-resolver.ts) | Glob 模型 scope 解析 |
| `SettingsManager` | [core/settings-manager.ts](packages/coding-agent/src/core/settings-manager.ts) | 设置加载/合并 |
| `KeybindingsManager` | [core/keybindings.ts](packages/coding-agent/src/core/keybindings.ts) | 快捷键管理 |
| `DefaultResourceLoader` | [core/resource-loader.ts](packages/coding-agent/src/core/resource-loader.ts) | 资源加载协调（extensions/skills/prompts/themes/context-files） |
| `ExtensionRunner` | [core/extensions/runner.ts](packages/coding-agent/src/core/extensions/runner.ts) | 扩展运行器 |
| `createExtensionAPI(runtime, options)` | [core/extensions/loader.ts](packages/coding-agent/src/core/extensions/loader.ts) | 扩展 API 表面 |
| `loadExtensionsCached(...)` | [core/extensions/loader.ts](packages/coding-agent/src/core/extensions/loader.ts) | 缓存式扩展加载（按 cwd） |
| `ToolDefinition` | [core/extensions/types.ts](packages/coding-agent/src/core/extensions/types.ts) | 富 Tool 类型 |
| `wrapToolDefinition(...)` | [core/tools/tool-definition-wrapper.ts](packages/coding-agent/src/core/tools/tool-definition-wrapper.ts) | Tool 适配 |
| `create*Tool(operations, options)` | [core/tools/*.ts](packages/coding-agent/src/core/tools/) | 内置 Tool 工厂 |
| `createCodingToolDefinitions(cwd, options)` | [core/tools/index.ts](packages/coding-agent/src/core/tools/index.ts) | 全部内置 Tool |
| `createReadOnlyToolDefinitions(...)` | [core/tools/index.ts](packages/coding-agent/src/core/tools/index.ts) | 仅 read/grep/find/ls |
| `ProjectTrustStore` | [core/trust-manager.ts](packages/coding-agent/src/core/trust-manager.ts) | 信任存储 |
| `resolveProjectTrusted(...)` | [core/project-trust.ts](packages/coding-agent/src/core/project-trust.ts) | 信任解析流程 |
| `AuthStorage` / `FileAuthStorageBackend` | [core/auth-storage.ts](packages/coding-agent/src/core/auth-storage.ts) | `auth.json` 凭证存储（`proper-lockfile` 锁） |
| `RuntimeCredentials` | [core/runtime-credentials.ts](packages/coding-agent/src/core/runtime-credentials.ts) | 运行时临时 key overlay |
| `resolveConfigValue(...)` | [core/resolve-config-value.ts](packages/coding-agent/src/core/resolve-config-value.ts) | 解析 `$(...)`/`$VAR`/字面量配置 |
| `DefaultPackageManager` | [core/package-manager.ts](packages/coding-agent/src/core/package-manager.ts) | npm/git 包管理 |
| `InteractiveMode` | [modes/interactive/interactive-mode.ts](packages/coding-agent/src/modes/interactive/interactive-mode.ts) | TUI 主类 |
| `runPrintMode(runtime, options)` | [modes/print-mode.ts](packages/coding-agent/src/modes/print-mode.ts) | Print/JSON 模式 |
| `runRpcMode(runtimeHost)` | [modes/rpc/rpc-mode.ts](packages/coding-agent/src/modes/rpc/rpc-mode.ts) | RPC 模式 |
| `RpcClient` | [modes/rpc/rpc-client.ts](packages/coding-agent/src/modes/rpc/rpc-client.ts) | RPC 客户端 |
| `attachJsonlLineReader(stream, onLine)` | [modes/rpc/jsonl.ts](packages/coding-agent/src/modes/rpc/jsonl.ts) | 严格 LF JSONL 读取 |
| `mergeProviderAttributionHeaders(...)` | [core/provider-attribution.ts](packages/coding-agent/src/core/provider-attribution.ts) | Provider attribution headers |
| `UsageTotals` / `addUsageToTotals` | [core/usage-totals.ts](packages/coding-agent/src/core/usage-totals.ts) | Token/Cost 累计 |
| `detectMiss(prev, message, models)` | [core/cache-stats.ts](packages/coding-agent/src/core/cache-stats.ts) | 缓存 miss 检测 |
| `FooterDataProvider` | [core/footer-data-provider.ts](packages/coding-agent/src/core/footer-data-provider.ts) | Footer 数据（git 分支、扩展状态） |
| `execCommand(command, args, cwd, options)` | [core/exec.ts](packages/coding-agent/src/core/exec.ts) | 子进程执行 |
| `executeBashWithOperations(...)` | [core/bash-executor.ts](packages/coding-agent/src/core/bash-executor.ts) | 流式 bash |
| `configureHttpDispatcher(timeoutMs)` | [core/http-dispatcher.ts](packages/coding-agent/src/core/http-dispatcher.ts) | undici 全局 dispatcher |
| `takeOverStdout()` / `writeRawStdout(text)` | [core/output-guard.ts](packages/coding-agent/src/core/output-guard.ts) | 输出守卫（避免污染 RPC 协议流） |
| `parseArgs(argv)` / `Args` | [cli/args.ts](packages/coding-agent/src/cli/args.ts) | CLI 解析 |
| `runMigrations(cwd)` | [migrations.ts](packages/coding-agent/src/migrations.ts) | 配置迁移 |

---

## 6. 核心数据流

### 6.1 单次 Prompt 完整流程

```
用户输入 → InteractiveMode 编辑器
  │
  ├─ Enter 提交 → AgentSession.prompt(text)
  │   ├─ 发射 `input` 事件（扩展可修改）
  │   ├─ 解析 `/skill:name`、`/templatename`、`!command`
  │   ├─ 区分 steer / followUp / 普通排队
  │   ├─ 验证 model + auth（getAuth 失败时引导 /login）
  │   └─ _runAgentPrompt():
  │       ├─ _isAgentRunActive = true
  │       ├─ agent.prompt(messages)
  │       │   ├─ BeforeAgentStart 扩展事件（可注入消息 / 覆盖 systemPrompt）
  │       │   ├─ 每轮:
  │       │   │   ├─ prepareNextTurnWithContext (重建 systemPrompt + tool 注册表)
  │       │   │   ├─ streamFn (含 transformHeaders 合并 provider attribution)
  │       │   │   │   └─ modelRuntime.streamSimple → pi-ai provider stream
  │       │   │   ├─ beforeToolCall hook → extensionRunner.emitToolCall
  │       │   │   ├─ tool.execute → emitToolCall 事件
  │       │   │   ├─ afterToolCall hook → extensionRunner.emitToolResult
  │       │   │   └─ turn_end (扩展可见)
  │       │   └─ agent_end
  │       └─ _handlePostAgentRun:
  │           ├─ 检查重试（错误时按 retry 配置）
  │           ├─ 检查 Compaction（溢出恢复 / 阈值主动）
  │           └─ 处理 queued steering/followUp
  │
  ├─ SessionManager.appendMessage（每条 message_end 时）
  └─ _persist（延迟到首条 assistant 消息）
```

### 6.2 Session 切换流程

```
switchSession(newPath) / newSession() / fork(entryId) / importFromJsonl(path)
  │
  ├─ emitBeforeSwitch/Fork/Tree (扩展可 cancel)
  ├─ teardownCurrent:
  │   ├─ emit session_shutdown
  │   ├─ session.dispose()
  │   └─ services.dispose()
  ├─ createRuntime(newCwd, newSessionManager)  # 重建 services + 加载扩展
  ├─ 应用新 runtime
  ├─ 扩展重绑 (bindCore)
  └─ 旧 ExtensionContext 失效 (runner.invalidate → assertActive 抛错)
```

### 6.3 Compaction 流程

```
触发: /compact 命令 / 自动溢出 / 阈值主动
  │
  ├─ 检查 phase === "idle"
  ├─ session_before_compact 扩展钩子（可 cancel / 提供预计算结果）
  ├─ prepareCompaction(branchEntries, settings):
  │   ├─ estimateContextTokens
  │   ├─ findCutPoint (后向累计，找 user/assistant 边界)
  │   └─ 收集 messagesToSummarize + previousSummary (增量)
  ├─ compact():
  │   ├─ generateSummary (SUMMARIZATION_PROMPT 或 UPDATE_SUMMARIZATION_PROMPT)
  │   ├─ 若 isSplitTurn: 额外 generateTurnPrefixSummary
  │   └─ 追加 <read-files>/<modified-files> 块
  ├─ session.appendCompaction(CompactionEntry)
  └─ 发射 session_compact 事件
```

### 6.4 跨 Provider 请求流程

```
models.stream(model, context, options)
  → lazyStream 同步返回 AssistantMessageEventStream
     异步:
       1. applyAuth:
          - getAuth(model) → resolveProviderAuth
            ├─ CredentialStore.read(providerId)
            │   └─ OAuth 过期 → credentials.modify 锁内 oauth.refresh
            │       失败 → ModelsError("oauth")，凭证保留
            ├─ 无 stored → envApiKeyAuth.resolve(env vars)
            └─ 无 auth → undefined (ModelsError("auth") on call)
          - 合并: auth.headers → model.headers → options.headers → transformHeaders
       2. provider.stream → lazyApi → 动态 import API module
       3. API module (e.g. anthropic-messages.ts):
          - 构造 SDK client (apiKey 或 OAuth)
          - transformMessages (跨 Provider 消息转换，如 thinking → text)
          - 流式发射 AssistantMessageEvent
       4. 失败永不抛错 → error 事件 + stopReason
```

---

## 7. 配置系统

### 7.1 配置文件位置

| 路径 | 作用域 |
|------|--------|
| `~/.pi/agent/settings.json` | 全局设置 |
| `<cwd>/.pi/settings.json` | 项目设置（仅信任时加载） |
| `~/.pi/agent/auth.json` | Provider 凭证（含 OAuth tokens） |
| `~/.pi/agent/models.json` | 自定义 Provider/模型配置 |
| `~/.pi/agent/keybindings.json` | 自定义快捷键 |
| `~/.pi/agent/project-trust.json` | 项目信任决策 |
| `~/.pi/agent/models-store.json` | 动态模型目录缓存 |
| `~/.pi/agent/sessions/` | 会话 JSONL 文件 |
| `~/.pi/agent/packages/` | npm/git 安装的 pi 包 |
| `~/.pi/agent/extensions/` | 用户扩展 |
| `~/.pi/agent/skills/` | 用户 skills |
| `~/.pi/agent/prompts/` | 用户 prompt 模板 |
| `~/.pi/agent/themes/` | 用户主题 |
| `<cwd>/.pi/{extensions,skills,prompts,themes}/` | 项目资源（仅信任时加载） |
| `<cwd>/AGENTS.md` 或 `CLAUDE.md` | 上下文文件（向上遍历） |
| `<cwd>/.pi/SYSTEM.md` / `APPEND_SYSTEM.md` | 系统提示覆盖/追加 |

### 7.2 关键环境变量

| 变量 | 说明 |
|------|------|
| `PI_CODING_AGENT_DIR` | 覆盖配置目录（默认 `~/.pi/agent`） |
| `PI_CODING_AGENT_SESSION_DIR` | 覆盖会话目录 |
| `PI_PACKAGE_DIR` | 覆盖包目录 |
| `PI_OFFLINE` / `PI_OFFLINE=1` | 禁用启动时所有网络操作 |
| `PI_SKIP_VERSION_CHECK` | 跳过版本更新检查 |
| `PI_TELEMETRY` | 覆盖 telemetry 与 attribution headers |
| `PI_CACHE_RETENTION` | 设为 `long` 启用扩展 prompt cache |
| `PI_EXPERIMENTAL=1` | 启用实验特性 |
| `PI_TIMING=1` | 启用启动计时输出到 stderr |
| `PI_ALLOW_LOCKFILE_CHANGE=1` | 允许提交 lockfile（CI 守卫） |
| `VISUAL` / `EDITOR` | Ctrl+G 外部编辑器回退 |
| `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / 等 | Provider API key |
| `HTTP_PROXY` / `HTTPS_PROXY` | HTTP 代理 |

### 7.3 Provider 认证解析顺序

每个 Provider 的 `auth` 通过 `resolveProviderAuth` 解析：

1. `overrides.apiKey`（显式传入）→ 立即解析。
2. `CredentialStore.read(providerId)`:
   - 若是 OAuth + Provider 有 `oauth` → 锁内刷新过期 token → `oauth.toAuth(credential)`。
   - 若是 api_key + Provider 有 `apiKey` → `apiKey.resolve({ ctx, credential })`。
   - 类型不匹配 → 返回 `undefined`。
3. 无 stored credential → `provider.auth.apiKey.resolve({ ctx })`（如 `envApiKeyAuth` 查 env vars）。
4. 仍无 → `undefined`（请求时报 `ModelsError("auth")`）。

**关键**: 失败的 OAuth 刷新不会静默回退到 env key（凭证保留以供重新登录）。

---

## 8. 扩展系统

### 8.1 Extension API 表面

扩展通过 `createExtensionAPI` 暴露的 `ExtensionAPI` 注入能力：

```typescript
export default function (pi: ExtensionAPI) {
  pi.registerTool({ name: "deploy", ... });      // 注册 Tool
  pi.registerCommand("stats", { ... });            // 注册 / 命令
  pi.registerShortcut("ctrl+x", handler);          // 注册快捷键
  pi.registerFlag({ name: "...", description });  // 注册 CLI flag
  pi.registerMessageRenderer(customType, renderer);
  pi.registerEntryRenderer(customType, renderer);
  pi.on("tool_call", async (event, ctx) => { ... });
  pi.on("before_agent_start", async (event) => { ... });
  pi.on("before_provider_request", async (event) => { ... });
  pi.registerProvider(providerOrName, config?);   // 注册 LLM Provider
  pi.registerNativeProvider(provider);            // 直接注册 Provider 对象
  pi.unregisterProvider(name);
  pi.events.on("...", handler);
  // 会话控制:
  pi.sendMessage(text, options);
  pi.setModel(provider, modelId);
  pi.compact(options);
  pi.newSession(options);
  pi.fork(entryId, options);
  pi.navigateTree(targetId, options);
  pi.switchSession(sessionPath, options);
  pi.reload();
  pi.shutdown();
}
```

默认导出可为 `async`，pi 会等待其初始化完成。

### 8.2 Extension 事件类型

- **会话事件**: `session_start`、`session_before_switch`、`session_before_fork`、`session_before_compact`、`session_compact`、`session_shutdown`、`session_before_tree`、`session_tree`。
- **Agent 事件**: `before_agent_start`、`context`、`before_provider_request`、`before_provider_headers`、`before_provider_payload`、`after_provider_response`。
- **Tool 事件**: `tool_call`（可阻止）、`tool_result`（可覆盖 content/details/usage/terminate）。
- **输入事件**: `input`（可修改用户输入）。
- **项目信任**: `project_trust`（首个非 undecided 胜，可 `remember: true`）。
- **消息渲染**: `message_renderer` / `entry_renderer`（自定义消息/条目类型）。

### 8.3 Extension 加载

[packages/coding-agent/src/core/extensions/loader.ts](packages/coding-agent/src/core/extensions/loader.ts):

- **`VIRTUAL_MODULES`**: 把 `typebox`、`@earendil-works/*` 等模块预导入，让 Bun 能打包进 binary；运行时通过 `virtualModules` 解析。
- **`getAliases()`**: Node 开发模式同等物，解析到 workspace `dist/*.js`。
- **`loadExtensionModule`**: 用 `jiti.import` 加载 TypeScript/JS（无需预编译）。
- **`loadExtensionsCached`**: 按 cwd 缓存加载结果，`clearExtensionCache()` 失效。
- **`createExtensionRuntime`**: 返回带 throwing stubs（动作方法）+ queueing stubs（`registerProvider`/`registerNativeProvider` 缓冲到 `bindCore`）的 `ExtensionRuntime`。

### 8.4 Extension 资源发现

`package.json` 的 `pi` 字段声明扩展资源：

```json
{
  "name": "my-pi-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

无 `pi` manifest 时按约定目录自动发现。

### 8.5 Pi Packages

通过 `pi install` 安装 npm 或 git 包，支持版本 pin、项目本地（`-l`）、更新检查：

```bash
pi install npm:@foo/pi-tools
pi install git:github.com/user/repo@v1
pi install https://github.com/user/repo@v1
pi install ssh://git@github.com/user/repo@v1
pi update --all        # 更新 pi 和包
pi update --extensions  # 仅更新包
pi update --self        # 仅更新 pi
pi config              # 启用/禁用资源
```

Git 包用 `npm install --omit=dev` 安装依赖（runtime 依赖必须在 `dependencies`）。

---

## 9. 认证与 Provider 体系

### 9.1 三层 Provider 组合

`ModelRuntime` 通过 `composeModelProvider` 组合三层：

1. **base** — 来自 `@earendil-works/pi-ai/providers/all` 的内置 Provider（每个经 `withRemoteCatalog` 叠加 pi.dev `/api/models/providers/<id>` catalog，4h 缓存，并发去重）。
2. **models.json** — 用户在 `~/.pi/agent/models.json` 配置的 Provider（`api`/`baseUrl`/`apiKey`/`headers`/`compat`/`oauth: "radius"`/`models[]`/`modelOverrides`）。
3. **extension** — 扩展通过 `pi.registerProvider` 注册的 `ProviderConfigInput`，叠加在 base 上。

`composeModelProvider` 不读凭证，仅组合 catalog 与 auth 元数据：
- `getModels()`: `applyModelsJson` (upsert/override) → `applyExtension` (扩展模型列表) → `applyModelOverride` (per-model 字段覆盖) → `extension.oauth.modifyModels`（OAuth 登录后）。
- `auth.apiKey`: `composeApiKeyAuth`，继承 base 的 login/check/resolve 或从 `models.json` 合成。
- `auth.oauth`: `composeOAuthAuth`，适配 `ExtensionOAuthConfig` 到 `OAuthAuth`。
- `stream`/`streamSimple`: 优先 `extension.streamSimple`（若 `model.api === extension.api`），否则 base provider（若支持该 api），否则 `getApiProvider(model.api)` 从 pi-ai compat 兜底。

### 9.2 Provider 列表（39+）

完整列表见 [packages/ai/README.md](packages/ai/README.md) 的 Supported Providers 表，包括：

- **OAuth Provider**: Anthropic（Claude Pro/Max）、OpenAI Codex（ChatGPT Plus/Pro）、GitHub Copilot、xAI、Radius。
- **API key Provider**: OpenAI、Ant Ling、Azure OpenAI、DeepSeek、NVIDIA NIM、Google Gemini、Vertex AI、Mistral、Groq、Cerebras、Cloudflare、xAI、OpenRouter、Vercel AI Gateway、ZAI、MiniMax、Together AI、Hugging Face、Fireworks、Moonshot AI、Kimi、Qwen、Xiaomi MiMo、OpenCode Zen/Go 等。
- **Ambient Auth**: Amazon Bedrock（AWS profiles / `AWS_BEARER_TOKEN_BEDROCK` / ECS 任务角色 / web identity）、Vertex AI（ADC + project/location）。
- **OpenAI 兼容**: Ollama、vLLM、LM Studio 等通过 `createProvider` + `openAICompletionsApi` 自定义。
- **Local Inference**: llama.cpp（通过内置扩展，`/llama` 命令管理模型下载/加载）。

### 9.3 `models.json` 配置示例

```json
{
  "version": 1,
  "providers": {
    "my-proxy": {
      "name": "My Proxy",
      "baseUrl": "https://proxy.example.com/v1",
      "apiKey": "sk-...",
      "api": "openai-completions",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        {
          "id": "custom-model",
          "name": "Custom Model",
          "reasoning": false,
          "input": ["text"],
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 128000,
          "maxTokens": 32000
        }
      ]
    }
  }
}
```

`apiKey` 字段支持 `$(command)`、`$VAR`、`${VAR}` 形式（通过 `resolveConfigValue` 解析，命令结果进程内缓存）。

---

## 10. 运行方式

### 10.1 安装

```bash
# npm 全局安装
npm install -g --ignore-scripts @earendil-works/pi-coding-agent

# 或安装脚本
curl -fsSL https://pi.dev/install.sh | sh

# 从源码运行（开发）
git clone https://github.com/earendil-works/pi-mono
cd pi-mono
npm install --ignore-scripts
./pi-test.sh  # 直接从源码运行 pi
```

### 10.2 认证

```bash
# API key
export ANTHROPIC_API_KEY=sk-ant-...
pi

# 或订阅登录
pi
/login              # 交互式选择 Provider
/login anthropic    # 直接登录指定 Provider

# CLI 直接传入
pi --api-key sk-... -p "Say hello"
```

### 10.3 命令行用法

```bash
pi [options] [@files...] [messages...]
```

**模式标志**:

| Flag | 说明 |
|------|------|
| (default) | Interactive TUI 模式 |
| `-p` / `--print` | Print 模式，输出最终响应 |
| `--mode json` | JSON 模式，所有事件 NDJSON |
| `--mode rpc` | RPC 模式（进程集成） |
| `--export <in> [out]` | 导出会话为 HTML |

**常用选项**:

| 选项 | 说明 |
|------|------|
| `--provider <name>` | 指定 Provider |
| `--model <pattern>` | 模型 pattern 或 ID（支持 `provider/id` 与 `:thinking`） |
| `--thinking <level>` | `off`/`minimal`/`low`/`medium`/`high`/`xhigh`/`max` |
| `--models <patterns>` | Ctrl+P 循环的逗号分隔 patterns |
| `--list-models [search]` | 列出可用模型 |
| `-c` / `--continue` | 继续最近会话 |
| `-r` / `--resume` | 浏览选择会话 |
| `--session <path\|id>` | 指定会话文件或 UUID |
| `--fork <path\|id>` | Fork 已有会话 |
| `--no-session` | 临时模式（不保存） |
| `--name <name>` / `-n` | 设置会话显示名 |
| `--tools <list>` / `-t` | Tool 白名单 |
| `--exclude-tools <list>` / `-xt` | Tool 黑名单 |
| `--no-builtin-tools` / `-nbt` | 禁用内置 Tool |
| `--no-tools` / `-nt` | 禁用所有 Tool |
| `-e` / `--extension <source>` | 加载扩展（可重复） |
| `--no-extensions` | 禁用扩展发现 |
| `--skill <path>` / `--no-skills` | Skill 控制 |
| `--prompt-template <path>` / `--no-prompt-templates` | 模板控制 |
| `--theme <path>` / `--no-themes` | 主题控制 |
| `--no-context-files` / `-nc` | 禁用 AGENTS.md/CLAUDE.md |
| `--system-prompt <text>` | 替换默认系统提示 |
| `--append-system-prompt <text>` | 追加系统提示 |
| `-a` / `--approve` | 信任项目本地文件 |
| `-na` / `--no-approve` | 忽略项目本地文件 |
| `--verbose` | 详细启动 |
| `--offline` | 离线模式 |
| `-h` / `--help` | 帮助 |
| `-v` / `--version` | 版本 |

**文件参数**（`@` 前缀）:

```bash
pi @prompt.md "Answer this"
pi -p @screenshot.png "What's in this image?"
pi @code.ts @test.ts "Review these files"
```

**Stdin 合并**（Print 模式）:

```bash
cat README.md | pi -p "Summarize this text"
```

### 10.4 包管理命令

```bash
pi install <source> [-l]         # 安装包，-l 项目本地
pi remove <source> [-l]          # 移除
pi uninstall <source> [-l]       # remove 别名
pi update [source|self|pi]       # 更新
pi update --all                  # 更新 pi + 包
pi update --extensions           # 仅更新包
pi update --models               # 刷新模型目录
pi update --self                 # 仅更新 pi
pi update --self --force         # 强制重装 pi
pi list                          # 列出已安装包
pi config                        # 启用/禁用资源
```

### 10.5 SDK 编程用法

```typescript
import { createAgentSession, ModelRuntime, SessionManager } from "@earendil-works/pi-coding-agent";

const modelRuntime = await ModelRuntime.create();
const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  modelRuntime,
});

await session.prompt("What files are in the current directory?");
```

高级多会话运行时替换用 `createAgentSessionRuntime()` 与 `AgentSessionRuntime`。详见 [packages/coding-agent/docs/sdk.md](packages/coding-agent/docs/sdk.md) 与 [packages/coding-agent/examples/sdk/](packages/coding-agent/examples/sdk/)。

### 10.6 RPC 集成

```bash
pi --mode rpc
```

使用严格 LF 分隔 JSONL 协议（不要用 Node `readline`，会按 U+2028/U+2029 分隔破坏 JSON 字符串）。客户端用 `RpcClient` 类：

```typescript
import { RpcClient } from "@earendil-works/pi-coding-agent";

const client = new RpcClient({ /* args */ });
await client.start();
const result = await client.prompt("Hello");
```

详见 [packages/coding-agent/docs/rpc.md](packages/coding-agent/docs/rpc.md)。

### 10.7 Tmux 测试（开发）

```bash
tmux new-session -d -s pi-test -x 80 -y 24
tmux send-keys -t pi-test "./pi-test.sh" Enter
sleep 3 && tmux capture-pane -t pi-test -p
tmux send-keys -t pi-test "your prompt here" Enter
tmux send-keys -t pi-test Escape       # 特殊键
tmux kill-session -t pi-test
```

---

## 11. 测试与构建

### 11.1 构建命令

```bash
# 全量构建（按依赖顺序）
npm run build
# 等价于:
cd packages/tui && npm run build
cd packages/ai && npm run build       # 含 generate-models
cd packages/agent && npm run build
cd packages/coding-agent && npm run build  # 含 copy-assets
cd packages/orchestrator && npm run build

# 检查（lint + format + type check）
npm run check
# 等价于:
biome check --write --error-on-warnings .
npm run check:pinned-deps       # 验证直接依赖 pin
npm run check:ts-imports        # 验证 TS 相对导入
npm run check:shrinkwrap        # 验证 coding-agent shrinkwrap
npm run check:install-lock:coding-agent
tsgo --noEmit
npm run check:browser-smoke
```

### 11.2 测试命令

```bash
# 全部非 e2e 测试（推荐，跳过有 endpoint/auth env var 时激活的 e2e）
./test.sh

# 运行特定测试（从包根目录）
node ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts

# coding-agent 回归测试
# 位置: packages/coding-agent/test/suite/regressions/<issue>-<slug>.test.ts
# 用 test/suite/harness.ts + faux provider，禁止真实 API/key/token

# 单包测试
cd packages/agent && npm run test
cd packages/ai && npm run test
cd packages/coding-agent && npm run test
cd packages/tui && npm run test  # 用 node --test

# pi-agent-core harness 测试
cd packages/agent && npm run test:harness
```

**关键规则**（来自 [AGENTS.md](AGENTS.md)）:
- 不要直接跑全 vitest suite（含 e2e，有 endpoint/auth env var 时激活）。
- 改了代码后跑 `npm run check`（不跑测试），修复所有 errors/warnings/infos 后才提交。
- 修改测试文件后必须跑该测试直到通过。
- 不要跑 `npm run build` 或 `npm test` 除非用户要求。

### 11.3 发布流程

**锁步版本**: 所有包共享版本号，每次发布全部更新。`patch` = 修复+新增，`minor` = 破坏性变更，无 major。

```bash
# 1. 确保 CHANGELOG 已更新（用户应先跑 /cl prompt）
# 2. 本地 smoke test
npm run release:local -- --out /tmp/pi-local-release --force
cd /tmp
# Node 包安装测试
/tmp/pi-local-release/node/pi --help
/tmp/pi-local-release/node/pi --version
/tmp/pi-local-release/node/pi --list-models
/tmp/pi-local-release/node/pi -p "Say exactly: ok"
# Bun binary 测试
/tmp/pi-local-release/bun/pi --help
# ...

# 3. 发布
PI_ALLOW_LOCKFILE_CHANGE=1 npm_config_min_release_age=0 npm run release:patch  # 修复+新增
PI_ALLOW_LOCKFILE_CHANGE=1 npm_config_min_release_age=0 npm run release:minor  # 破坏性
```

发布脚本: bump 所有包版本 → 更新 CHANGELOG → 重新生成发布产物 → 跑 `npm run check` → commit `Release vX.Y.Z` → tag `vX.Y.Z` → 加新 `## [Unreleased]` 段 → commit → push `main` 与 tag。

CI（`.github/workflows/build-binaries.yml`）由 tag push 触发，`publish-npm` job 用 GitHub Actions OIDC trusted publishing，无需本地 `npm publish`/OTP。

### 11.4 CI 工作流

[.github/workflows/](.github/workflows/) 包含:
- `ci.yml` — 主 CI（check + test）
- `build-binaries.yml` — 构建二进制 + 发布 npm（由 tag 触发）
- `pr-gate.yml` / `issue-gate.yml` — 贡献者门控（auto-close 新贡献者 PR/issue）
- `issue-triage-labels.yml` / `issue-analysis.yml` — issue 自动分类
- `npm-audit.yml` — 定期 `npm audit` + `npm audit signatures`
- `approve-contributor.yml` — 维护者审批流程
- `publish-model-catalog.yml` — 模型目录发布

---

## 12. 开发约定

### 12.1 代码风格

来自 [AGENTS.md](AGENTS.md) 的关键规则：

- **TypeScript**: 严格 ESM，仅用 erasable 语法（Node strip-only mode），禁用 `enum`、`namespace`/`module`、`import =`、`export =`、parameter properties。
- **导入**: 顶层导入，禁用 `await import()`、`import("pkg").Type` 等内联导入。
- **类型**: 禁用 `any` 除非必要；单行单调用点的 helper 内联。
- **键绑定**: 禁止硬编码按键检查，必须加到 `DEFAULT_EDITOR_KEYBINDINGS` 或 `DEFAULT_APP_KEYBINDINGS`。
- **生成文件**: 禁止直接改 `packages/ai/src/models.generated.ts`，改 `scripts/generate-models.ts` 后重新生成。
- **向后兼容**: 不保留除非用户要求。
- **删除**: 删除功能前必须先问，除非确认无用。
- **依赖**: 不为修过时依赖的类型错误而删/降级代码，应升级依赖。

### 12.2 Git 规则

- **多 session 并发**: 同 cwd 可能有多个 pi session 各改不同文件。git 操作只动自己改的文件。
- **提交**: 仅提交本 session 改的文件，显式 `git add <path1> <path2>`，禁止 `git add -A`/`git add .`。
- **禁止**: `git reset --hard`、`git checkout .`、`git clean -fd`、`git stash`、`git commit --no-verify`。
- **冲突**: 仅解决自己改的文件冲突，他文件冲突则 abort 并问用户。
- **不 force push**，不 push 到 main/master。
- **Commit 格式**: `{feat,fix,docs}[(ai,tui,agent,coding-agent)]: <message>`。
- **关闭 issue**: commit message 含 `fixes #<n>` 或 `closes #<n>`（多个 issue 重复 keyword，`closes #1, closes #2` 而非 `closes #1, #2`）。
- **未要求不 commit**。

### 12.3 依赖安全

- 直接外部依赖 pin 到精确版本，workspace 包用范围版本。
- `.npmrc`: `save-exact=true`、`min-release-age=2`（避免同日发布）。
- 本地装: `npm install --ignore-scripts`；CI 装: `npm ci --ignore-scripts`；不跑生命周期脚本除非用户要求。
- 元数据变化: `npm install --package-lock-only --ignore-scripts` 刷新 lockfile。
- coding-agent shrinkwrap 重生成: `node scripts/generate-coding-agent-shrinkwrap.mjs`（验证用 `--check`）。
- Pre-commit 阻止 lockfile 提交除非 `PI_ALLOW_LOCKFILE_CHANGE=1`。
- 新生命周期脚本依赖需要 review 与显式 allowlist entry。

### 12.4 Changelog 规则

- 位置: `packages/*/CHANGELOG.md`（每包一个）。
- 新条目放 `## [Unreleased]` 下，分段 `### Breaking Changes` / `### Added` / `### Changed` / `### Fixed` / `### Removed`。
- 已发布段（如 `## [0.12.2]`）不可变。
- 内部 attribution: `Fixed foo bar ([#123](link))`。
- 外部贡献: `Added feature X ([#456](link) by [@username](link))`。

### 12.5 Issue/PR 规则

- 新贡献者 issue/PR 默认 auto-close，维护者每日审查。
- `lgtmi` → 未来 issue 不 auto-close；`lgtm` → 未来 issue+PR 都不 auto-close。
- 用 `pkg:*` 标签（`pkg:agent`、`pkg:ai`、`pkg:coding-agent`、`pkg:tui`）。
- PR review 不 `gh pr checkout`，用 `gh pr view`/`gh pr diff`/`gh api`/`git show <ref>:<path>`。
- 评论写到临时文件用 `gh issue/pr comment --body-file`，不用 `--body`。
- 评论结尾必须含 AI-generated 免责声明（如 `This comment is AI-generated by /wr`）。

### 12.6 容器化与沙箱

Pi 不内置权限系统，默认以启动用户权限运行。需要更强边界时容器化或沙箱，参见 [packages/coding-agent/docs/containerization.md](packages/coding-agent/docs/containerization.md) 三种模式:

- **Gondolin 扩展**: 把 pi 和 Provider 认证留在宿主，把内置 Tool 与 `!` 命令路由到本地 Linux micro-VM。
- **Plain Docker**: 整个 pi 进程跑在本地容器。
- **OpenShell**: 整个 pi 进程跑在策略控制沙箱。

---

## 附录: 关键文档索引

- 项目主页: <https://pi.dev>
- 在线文档: <https://pi.dev/docs/latest>
- RFCs: <https://rfc.earendil.com/keyword/pi/>
- Discord: <https://discord.com/invite/3cU7Bz4UPx>

### 包内文档

- [packages/ai/README.md](packages/ai/README.md) — pi-ai 完整 API 文档
- [packages/agent/README.md](packages/agent/README.md) — pi-agent-core 完整 API 文档
- [packages/agent/docs/](packages/agent/docs/) — agent 内部文档（agent-harness、durable-harness、hooks、models、observability）
- [packages/coding-agent/README.md](packages/coding-agent/README.md) — pi CLI 用户文档
- [packages/coding-agent/docs/](packages/coding-agent/docs/) — 详细文档（compaction、containerization、custom-provider、development、extensions、json、keybindings、llama-cpp、models、packages、prompt-templates、providers、quickstart、rpc、sdk、security、session-format、sessions、settings、shell-aliases、skills、terminal-setup、termux、themes、tmux、tui、usage、windows）
- [packages/coding-agent/examples/](packages/coding-agent/examples/) — 扩展示例（70+ 个）与 SDK 示例（13 个）
- [AGENTS.md](AGENTS.md) — 项目规则（人与 Agent 共用）
- [CONTRIBUTING.md](CONTRIBUTING.md) — 贡献指南
- [SECURITY.md](SECURITY.md) — 安全策略

---

*本 Wiki 基于仓库当前状态生成，反映代码现状（不含版本历史）。如需查阅特定模块的最新实现，请直接阅读对应源码并对照其 CHANGELOG。*
