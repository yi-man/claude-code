# 仓库 Domain 与 Entity 分析

本文基于当前仓库代码结构与关键模块实现，提炼主要业务领域（Domain）与核心实体（Entity/等价核心对象），用于架构理解、模块拆分与后续重构参考。

## 1. 项目结构速览

- 入口与启动：`src/entrypoints/`、`src/main.tsx`
- 对话执行引擎：`src/query.ts`、`src/QueryEngine.ts`
- 命令与工具体系：`src/commands.ts`、`src/commands/**`、`src/tools.ts`、`src/tools/**`
- 状态管理：`src/state/**`
- 服务层（MCP/插件/记忆等）：`src/services/**`
- 记忆目录机制：`src/memdir/**`
- 文档：`docs/architecture/domain-architecture.md`
- 平台适配与恢复兼容：`vendor/**`、`shims/**`

该仓库整体形态是一个 **CLI/TUI 驱动的 Agent Runtime**，主链路可概括为：输入处理 -> 对话执行 -> 能力调用 -> 权限治理 -> 外部集成 -> 状态持久化。

## 2. 主要 Domain（业务领域/子系统）

### 2.1 交互壳层（Interaction Shell）

- 定位：接收用户输入并分流到 REPL、headless、daemon、bridge、MCP 等运行路径。
- 作用：承接启动参数、初始化上下文、触发后续执行链路。
- 关键模块：
  - `src/entrypoints/cli.tsx`
  - `src/main.tsx`
  - `src/replLauncher.tsx`
  - `src/commands.ts`
- 对外关系：向下驱动对话运行时、状态域、能力域与权限域。

### 2.2 对话运行时（Conversation Runtime）

- 定位：会话执行核心，维护多轮消息循环。
- 作用：处理模型流式输出、工具调用、错误恢复、压缩与停止逻辑。
- 关键模块：
  - `src/query.ts`
  - `src/QueryEngine.ts`
- 对外关系：依赖工具/命令能力，读写状态并受权限策略约束。

### 2.3 能力编排域（Capability Orchestration）

- 定位：统一管理可用能力（命令、工具、技能、插件能力）。
- 作用：构建能力池并暴露给用户输入处理与模型 tool-use。
- 关键模块：
  - `src/commands.ts`
  - `src/tools.ts`
  - `src/Tool.ts`
  - `src/skills/**`
- 对外关系：被运行时调用，受权限域拦截，与 MCP/插件域联动扩展。

### 2.4 权限与策略治理域（Permission & Policy Governance）

- 定位：决定能力调用是否允许执行（allow/deny/ask）。
- 作用：维护权限模式与规则集，控制执行风险边界。
- 关键模块：
  - `src/Tool.ts`（`ToolPermissionContext`）
  - `src/utils/permissions/**`
  - `src/state/onChangeAppState.ts`
  - `src/services/policyLimits/**`
- 对外关系：对工具调用进行硬约束，并与远程/桥接链路同步权限状态。

### 2.5 上下文与记忆域（Context & Memory）

- 定位：构造模型上下文并维护长期记忆。
- 作用：注入系统上下文、用户上下文、MEMORY 语义与会话摘要。
- 关键模块：
  - `src/context.ts`
  - `src/memdir/memdir.ts`
  - `src/services/SessionMemory/sessionMemory.ts`
- 对外关系：为运行时提供 prompt 语境，与状态域共同管理消息阈值与提炼策略。

### 2.6 集成织层（Integration Fabric：MCP / Remote / Bridge）

- 定位：连接外部系统与内部能力协议。
- 作用：把 MCP server、远程会话能力映射为内部工具/命令。
- 关键模块：
  - `src/services/mcp/client.ts`
  - `src/services/mcp/types.ts`
  - `src/remote/RemoteSessionManager.ts`
  - `src/entrypoints/mcp.ts`
- 对外关系：与能力域共同构建可调用能力集合，并受权限域治理。

### 2.7 插件生态域（Plugin Ecosystem）

- 定位：插件发现、装载、校验、缓存与挂载。
- 作用：动态扩展命令、技能与 MCP 配置来源。
- 关键模块：
  - `src/utils/plugins/pluginLoader.ts`
  - `src/services/plugins/**`
  - `src/plugins/**`
- 对外关系：对能力域与集成域进行增量注入，状态结果落到 `AppState`。

### 2.8 状态与持久化域（State & Persistence）

- 定位：全局事实源，承载运行期核心状态。
- 作用：统一维护会话、任务、插件、MCP 与通知等状态。
- 关键模块：
  - `src/state/AppStateStore.ts`
  - `src/state/AppState.tsx`
  - `src/state/onChangeAppState.ts`
  - `src/utils/sessionStorage.ts`
- 对外关系：几乎所有 Domain 都对其读写，是跨模块协同的状态中枢。

### 2.9 观测与控制平面（Observability & Control Plane）

- 定位：系统可观测性与特性开关治理。
- 作用：埋点、剖析、性能计数、实验开关。
- 关键模块：
  - `src/entrypoints/init.ts`
  - `src/services/analytics/**`
  - `src/utils/*Profiler*`
- 对外关系：横切所有 Domain，提供行为追踪与运行分析。

### 2.10 平台与原生适配域（Native/Platform Adapters）

- 定位：平台能力接入与恢复兼容层。
- 作用：提供音频、URL handler 等原生能力，补齐恢复仓库中的缺失依赖。
- 关键模块：
  - `vendor/audio-capture-src/index.ts`
  - `vendor/**`
  - `shims/ant-computer-use-mcp/index.ts`
  - `shims/**`
- 对外关系：为工具域与集成域提供底层能力实现或兜底。

## 3. 主要 Entity（含等价核心对象）

说明：该仓库不是传统 DDD 实体类风格，很多“实体”以 `type/interface + runtime aggregate` 形式存在。以下采用“等价核心对象”提炼。

### 3.1 `AppState`

- 定义位置：`src/state/AppStateStore.ts`
- 承载内容：全局会话状态（settings、permissions、tasks、mcp、plugins、notifications 等）
- 主要消费者：
  - `src/main.tsx`
  - `src/query.ts`
  - `src/state/AppState.tsx`
  - `src/state/onChangeAppState.ts`

### 3.2 `Message`（及其子类型）

- 定义位置：`src/types/message.ts`
- 承载内容：会话转录核心单元（用户消息、助手消息、系统消息、tool 输出等）
- 主要消费者：
  - `src/query.ts`
  - `src/QueryEngine.ts`
  - `src/utils/messages.ts`

### 3.3 `ToolUseContext`

- 定义位置：`src/Tool.ts`
- 承载内容：单轮执行内的工具上下文（可用 tools/commands、abort、app state getter/setter、permission 回调等）
- 主要消费者：
  - `src/query.ts`
  - `src/QueryEngine.ts`
  - `src/tools/**`

### 3.4 `Tool` / `Tools`

- 定义位置：`src/Tool.ts`
- 承载内容：工具协议（schema、权限、执行函数、渲染元数据）
- 主要消费者：
  - `src/tools.ts`
  - `src/query.ts`
  - `src/services/mcp/client.ts`

### 3.5 `Command`

- 定义位置：`src/types/command.ts`
- 承载内容：slash command 协议（prompt/local/local-jsx）
- 主要消费者：
  - `src/commands.ts`
  - `src/main.tsx`
  - `src/utils/processUserInput/**`

### 3.6 `TaskState`

- 定义位置：`src/tasks/types.ts`
- 承载内容：后台任务统一状态（本地 shell、agent、workflow、remote monitor 等）
- 主要消费者：
  - `src/state/AppStateStore.ts`
  - `src/tasks/**`
  - `src/screens/**`

### 3.7 `ToolPermissionContext`

- 定义位置：`src/Tool.ts`
- 承载内容：权限模式、allow/deny 规则、额外工作目录、自动化标记等
- 主要消费者：
  - `src/query.ts`
  - `src/utils/permissions/**`
  - `src/tools.ts`

### 3.8 `MCPServerConnection` / `ScopedMcpServerConfig`

- 定义位置：`src/services/mcp/types.ts`
- 承载内容：MCP server 配置与连接生命周期状态
- 主要消费者：
  - `src/services/mcp/client.ts`
  - `src/main.tsx`
  - `src/state/AppStateStore.ts`

### 3.9 `LoadedPlugin`

- 定义位置：`src/types/plugin.ts`（由 `pluginLoader.ts` 使用）
- 承载内容：插件来源、manifest、命令/hooks、加载错误信息
- 主要消费者：
  - `src/utils/plugins/pluginLoader.ts`
  - `src/commands.ts`
  - `src/state/AppStateStore.ts`

### 3.10 `RemoteSessionConfig` / `RemotePermissionResponse`

- 定义位置：`src/remote/RemoteSessionManager.ts`
- 承载内容：远程会话配置、viewerOnly、远端权限请求响应
- 主要消费者：
  - `src/remote/RemoteSessionManager.ts`
  - `src/main.tsx`

### 3.11 Query 会话聚合（`QueryEngine` 内部态）

- 定位：等价 Session Aggregate
- 定义位置：`src/QueryEngine.ts`
- 承载内容：`mutableMessages`、`readFileState`、`permissionDenials`、`totalUsage`、abort controller
- 主要消费者：
  - `QueryEngine.ask()` 以及 `query.ts` 执行链路

### 3.12 Session Memory 核心对象

- 定义位置：`src/services/SessionMemory/sessionMemory.ts`
- 承载内容：记忆初始化阈值、更新阈值、摘要位置、提炼状态
- 主要消费者：
  - post-sampling hook
  - memory 文件更新流程

## 4. Domain 关系图（文字版）

- 交互壳层创建运行上下文并进入对话运行时。
- 对话运行时驱动消息循环并触发能力调用。
- 能力调用受权限治理域约束。
- 上下文与记忆域持续补充 prompt 语境。
- MCP/插件/远程集成将外部能力注入能力池。
- 状态域记录全链路事实并向 UI/流程分发。
- 观测平面横切全流程输出 telemetry/profiling 数据。
- 原生适配域提供平台能力与恢复兜底。

## 5. 结论

该仓库的架构核心不是“面向实体类”，而是“面向运行时协议与状态聚合”。  
因此在后续扩展中，建议优先关注三类稳定边界：

- 协议边界：`Tool`、`Command`、`Message`、MCP types
- 状态边界：`AppState`、`TaskState`、会话聚合内部态
- 编排边界：`main.tsx` + `query.ts` + `QueryEngine.ts`

这三类边界共同决定系统可扩展性与可维护性。

Domain 决定“做什么、由谁负责”，Entity 决定“在系统里传递什么关键对象/契约”
