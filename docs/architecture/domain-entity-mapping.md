# Domain 到 Entity 使用映射

本文基于 `docs/architecture/domain-entity-analysis.md` 中已定义的 Domain 与 Entity（含等价核心对象），梳理各 Domain 实际使用了哪些 Entity，以及使用方式（读 / 写 / 执行 / 约束 / 注入 / 同步）。

## 2.1 交互壳层（Interaction Shell）

| Entity | 如何使用 | 说明 |
|---|---|---|
| `AppState` | 读 / 写 / 同步 | 启动与入口流程读取全局设置、会话信息，并将初始化结果同步到状态中枢。 |
| `Command` | 读 / 执行 | 接收 slash command 输入并路由到对应命令协议。 |
| `RemoteSessionConfig` / `RemotePermissionResponse` | 读 / 同步 | 在 bridge/remote 路径下读取远程会话配置并同步权限交互结果。 |
| `MCPServerConnection` / `ScopedMcpServerConfig` | 读 / 同步 / 注入 | 启动阶段加载 MCP 配置并把可用连接能力注入运行上下文。 |

## 2.2 对话运行时（Conversation Runtime）

| Entity | 如何使用 | 说明 |
|---|---|---|
| `Message`（及其子类型） | 读 / 写 / 执行 | 作为多轮会话主载体，读取历史、追加新消息、驱动流式输出与工具回传。 |
| `ToolUseContext` | 注入 / 执行 | 为单轮执行注入 tools/commands/abort/state getter-setter 等上下文能力。 |
| `Tool` / `Tools` | 读 / 执行 | 在模型 tool-use 阶段选择并执行具体工具协议。 |
| `ToolPermissionContext` | 约束 / 读 | 在每次能力调用前后应用 allow/deny/ask 规则。 |
| Query 会话聚合（`QueryEngine` 内部态） | 读 / 写 / 约束 | 维护 `mutableMessages`、`permissionDenials`、`totalUsage`、abort 等会话内聚状态。 |
| `AppState` | 读 / 写 / 同步 | 同步运行期关键事实（权限、任务、通知、会话派生状态）。 |

## 2.3 能力编排域（Capability Orchestration）

| Entity | 如何使用 | 说明 |
|---|---|---|
| `Tool` / `Tools` | 读 / 执行 / 注入 | 构建并维护工具池，向运行时与输入处理暴露能力。 |
| `Command` | 读 / 执行 / 注入 | 聚合 slash command 能力并按协议分发。 |
| `ToolUseContext` | 注入 | 将能力集合与运行控制句柄注入到执行链路。 |
| `LoadedPlugin` | 注入 / 同步 | 把插件带来的命令/hooks/能力增量并入编排结果。 |
| `MCPServerConnection` / `ScopedMcpServerConfig` | 注入 / 同步 | 将 MCP 侧能力映射为内部可调用工具。 |
| `ToolPermissionContext` | 约束 | 对能力池中的执行入口施加统一权限策略。 |

## 2.4 权限与策略治理域（Permission & Policy Governance）

| Entity | 如何使用 | 说明 |
|---|---|---|
| `ToolPermissionContext` | 读 / 写 / 约束 | 权限核心对象，维护模式、规则与自动化标记。 |
| `ToolUseContext` | 约束 / 注入 | 在工具执行上下文中挂载权限回调与审计点。 |
| `AppState` | 读 / 写 / 同步 | 持久化权限状态、规则变更与策略决策结果。 |
| `RemotePermissionResponse` | 同步 / 约束 | 在远程链路中同步权限请求响应并约束后续执行。 |

## 2.5 上下文与记忆域（Context & Memory）

| Entity | 如何使用 | 说明 |
|---|---|---|
| `Message`（及其子类型） | 读 / 写 | 读取对话转录用于上下文构造，并写入摘要/提炼结果。 |
| Session Memory 核心对象 | 读 / 写 / 约束 | 管理初始化阈值、更新阈值、摘要位置与提炼状态。 |
| Query 会话聚合（`QueryEngine` 内部态） | 读 / 约束 | 依据会话内聚状态决定压缩、截断与记忆更新时机。 |
| `AppState` | 读 / 同步 | 从全局状态读取会话与配置事实，回写记忆相关状态。 |

## 2.6 集成织层（Integration Fabric：MCP / Remote / Bridge）

| Entity | 如何使用 | 说明 |
|---|---|---|
| `MCPServerConnection` / `ScopedMcpServerConfig` | 读 / 写 / 同步 / 注入 | 管理 MCP server 配置与连接生命周期，并把外部能力注入内部工具协议。 |
| `Tool` / `Tools` | 注入 / 执行 | 将外部能力适配为内部工具并参与执行。 |
| `ToolUseContext` | 注入 / 执行 | 把集成能力装配进单轮执行上下文。 |
| `RemoteSessionConfig` / `RemotePermissionResponse` | 读 / 写 / 同步 | 管理远程会话参数与权限往返响应。 |
| `AppState` | 读 / 写 / 同步 | 持久化连接态、远程态与集成运行态。 |

## 2.7 插件生态域（Plugin Ecosystem）

| Entity | 如何使用 | 说明 |
|---|---|---|
| `LoadedPlugin` | 读 / 写 / 注入 / 同步 | 完成插件发现、校验、装载、缓存，并把插件能力同步到系统。 |
| `Command` | 注入 / 执行 | 插件扩展命令集合并进入命令分发链路。 |
| `Tool` / `Tools` | 注入 / 执行 | 插件扩展工具能力并进入 tool-use 能力池。 |
| `MCPServerConnection` / `ScopedMcpServerConfig` | 注入 / 同步 | 插件作为 MCP 配置来源时向集成层增量注入。 |
| `AppState` | 写 / 同步 | 记录插件加载状态、错误与可用能力快照。 |

## 2.8 状态与持久化域（State & Persistence）

| Entity | 如何使用 | 说明 |
|---|---|---|
| `AppState` | 读 / 写 / 同步 | 全局事实源与状态中枢。 |
| `TaskState` | 读 / 写 / 同步 | 统一承载后台任务生命周期。 |
| `MCPServerConnection` / `ScopedMcpServerConfig` | 写 / 同步 | 持久化 MCP 配置与连接状态。 |
| `LoadedPlugin` | 写 / 同步 | 持久化插件装载结果与错误信息。 |
| `Message`（及其子类型） | 读 / 同步 | 作为会话相关状态衍生输入（如转录、摘要触发）。 |
| `ToolPermissionContext` | 读 / 同步 | 反映权限策略状态到全局状态变化流。 |

## 2.9 观测与控制平面（Observability & Control Plane）

| Entity | 如何使用 | 说明 |
|---|---|---|
| `AppState` | 读 / 同步 | 读取运行态用于埋点、剖析和特性开关观测。 |
| Query 会话聚合（`QueryEngine` 内部态） | 读 | 提供 usage、拒绝计数、流式阶段等观测信号。 |
| `TaskState` | 读 / 同步 | 统计后台任务执行行为与性能数据。 |
| `Message`（及其子类型） | 读 | 分析会话事件流与交互质量指标。 |

## 2.10 平台与原生适配域（Native/Platform Adapters）

| Entity | 如何使用 | 说明 |
|---|---|---|
| `Tool` / `Tools` | 执行 / 注入 | 为工具域提供底层平台能力实现（音频、系统交互等）。 |
| `ToolUseContext` | 注入 / 执行 | 通过统一上下文接入原生能力调用。 |
| `AppState` | 同步 | 同步平台能力状态与适配结果。 |
| `MCPServerConnection` / `ScopedMcpServerConfig` | 注入（间接） | 通过 shim/vendor 提供兼容能力时参与集成注入链路。 |

## 高频跨域 Entity

| 高频跨域 Entity | 高频原因 | 典型跨域范围 |
|---|---|---|
| `AppState` | 全局事实源，几乎所有 Domain 都会读写或同步。 | 交互壳层、运行时、权限、集成、插件、状态域、观测平面 |
| `Message`（及其子类型） | 会话执行与上下文构造统一载体。 | 对话运行时、上下文与记忆域、状态域、观测平面 |
| `Tool` / `Tools` | 能力协议核心，连接编排、执行、集成与插件扩展。 | 能力编排域、对话运行时、集成织层、插件生态、平台适配 |
| `ToolUseContext` | 单轮执行上下文枢纽，承接能力注入与权限约束。 | 对话运行时、能力编排域、权限治理、集成织层、平台适配 |
| `ToolPermissionContext` | 所有可执行能力的统一风险边界对象。 | 权限治理、运行时、能力编排、状态同步、远程权限链路 |
| `MCPServerConnection` / `ScopedMcpServerConfig` | 外部能力接入基础连接实体，跨入口/集成/编排/状态流动。 | 集成织层、能力编排域、状态域、交互壳层、插件生态 |

## 备注

- 本文术语严格对齐 `docs/architecture/domain-entity-analysis.md`。
- `Query 会话聚合（QueryEngine 内部态）`与`Session Memory 核心对象`按原文作为“等价核心对象”处理。
