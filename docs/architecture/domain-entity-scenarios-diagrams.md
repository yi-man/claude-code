# 基于 Domain / Entity 的典型场景架构图

本文基于以下文档整理：
- `docs/architecture/domain-entity-analysis.md`
- `docs/architecture/domain-entity-mapping.md`

目标是围绕 Domain 与 Entity，提炼可落地的典型场景，并为每个场景给出三类 Mermaid 图：
- component designs
- data flows
- build sequences

## 一、典型场景（5 个）

1. 场景 A：CLI 启动并进入会话
2. 场景 B：一次 Tool-Use 执行（含权限治理）
3. 场景 C：MCP 能力注入到可调用工具池
4. 场景 D：插件加载并扩展 Command/Tools
5. 场景 E：会话记忆更新（Session Memory）

## 场景 A：CLI 启动并进入会话

### 1) Component Designs

```mermaid
flowchart TB
  subgraph D1["Interaction Shell"]
    CLI["cli.tsx / main.tsx"]
    Repl["replLauncher.tsx"]
  end

  subgraph D2["State & Persistence"]
    AppState["Entity: AppState"]
    AppStateStore["AppStateStore"]
  end

  subgraph D3["Capability Orchestration"]
    Commands["Entity: Command"]
    Tools["Entity: Tool / Tools"]
    ToolUseContext["Entity: ToolUseContext"]
  end

  subgraph D4["Integration Fabric"]
    MCPConn["Entity: MCPServerConnection / ScopedMcpServerConfig"]
    RemoteCfg["Entity: RemoteSessionConfig / RemotePermissionResponse"]
  end

  subgraph D5["Conversation Runtime"]
    Query["query.ts"]
    QueryEngine["Entity Aggregate: QueryEngine内部态"]
    Message["Entity: Message"]
  end

  CLI --> Repl
  CLI --> AppStateStore --> AppState
  CLI --> Commands
  CLI --> Tools
  CLI --> MCPConn
  CLI --> RemoteCfg
  Commands --> ToolUseContext
  Tools --> ToolUseContext
  AppState --> Query
  ToolUseContext --> Query
  Query --> QueryEngine --> Message
```

### 2) Data Flows

```mermaid
flowchart LR
  Input["用户输入 / 启动参数"] --> Shell["Interaction Shell"]
  Shell --> S1["读取/初始化 AppState"]
  Shell --> C1["装配 Command 与 Tool"]
  Shell --> I1["同步 MCPServerConnection / RemoteSessionConfig"]
  S1 --> Runtime["Conversation Runtime(query.ts)"]
  C1 --> Runtime
  I1 --> Runtime
  Runtime --> QAgg["QueryEngine内部态"]
  QAgg --> Msg["Message 流"]
  Msg --> S2["回写 AppState（会话事实）"]
```

### 3) Build Sequences

```mermaid
sequenceDiagram
  participant User as User
  participant Shell as Interaction Shell
  participant State as AppStateStore
  participant Cap as Capability Orchestration
  participant Intg as Integration Fabric
  participant Runtime as Conversation Runtime
  participant QE as QueryEngine Aggregate

  User->>Shell: 启动 CLI
  Shell->>State: 初始化/读取 AppState
  State-->>Shell: AppState
  Shell->>Cap: 装配 Command 与 Tool
  Cap-->>Shell: ToolUseContext 基础能力
  Shell->>Intg: 加载 MCP/Remote 配置
  Intg-->>Shell: MCPServerConnection / RemoteSessionConfig
  Shell->>Runtime: 进入 query 执行
  Runtime->>QE: 创建会话聚合并注入 Message
  QE-->>Runtime: 首轮可执行上下文
```

## 场景 B：一次 Tool-Use 执行（含权限治理）

### 1) Component Designs

```mermaid
flowchart TB
  subgraph Runtime["Conversation Runtime"]
    Query["query.ts"]
    QE["QueryEngine"]
    Msg["Entity: Message"]
  end

  subgraph Cap["Capability Orchestration"]
    Tools["Entity: Tool / Tools"]
    TUC["Entity: ToolUseContext"]
  end

  subgraph Gov["Permission & Policy Governance"]
    TPC["Entity: ToolPermissionContext"]
    Policy["permissions / policyLimits"]
  end

  subgraph State["State & Persistence"]
    AppState["Entity: AppState"]
    TaskState["Entity: TaskState"]
  end

  Query --> QE --> Msg
  QE --> TUC
  TUC --> Tools
  Tools --> TPC --> Policy
  Policy -->|allow/deny/ask| Tools
  Tools --> Msg
  Msg --> AppState
  Tools --> TaskState
```

### 2) Data Flows

```mermaid
flowchart LR
  M1["Message(assistant: tool-call)"] --> Q["QueryEngine"]
  Q --> Ctx["ToolUseContext"]
  Ctx --> P["ToolPermissionContext 检查"]
  P -->|allow| T["Tool 执行"]
  P -->|deny| D["拒绝记录(permissionDenials)"]
  P -->|ask| A["用户确认后继续"]
  A --> T
  T --> O["tool output Message"]
  D --> O2["deny Message"]
  O --> S["AppState/TaskState 同步"]
  O2 --> S
```

### 3) Build Sequences

```mermaid
sequenceDiagram
  participant Runtime as Conversation Runtime
  participant QE as QueryEngine Aggregate
  participant TUC as ToolUseContext
  participant TPC as ToolPermissionContext
  participant Tool as Tool
  participant State as AppState

  Runtime->>QE: 解析 Message 中 tool-use 意图
  QE->>TUC: 构建本轮执行上下文
  TUC->>TPC: 发起权限判定
  TPC-->>TUC: allow / deny / ask
  alt allow
    TUC->>Tool: 执行 Tool
    Tool-->>QE: 工具输出
  else ask
    TUC-->>Runtime: 请求用户确认
    Runtime->>Tool: 用户同意后执行
    Tool-->>QE: 工具输出
  else deny
    TUC-->>QE: 拒绝结果
  end
  QE->>State: 回写 Message/权限/任务状态
```

## 场景 C：MCP 能力注入到可调用工具池

### 1) Component Designs

```mermaid
flowchart TB
  subgraph Intg["Integration Fabric (MCP)"]
    MCPTypes["Entity: ScopedMcpServerConfig / MCPServerConnection"]
    MCPClient["mcp/client.ts"]
  end

  subgraph Cap["Capability Orchestration"]
    Tools["Entity: Tool / Tools"]
    TUC["Entity: ToolUseContext"]
  end

  subgraph State["State & Persistence"]
    AppState["Entity: AppState"]
  end

  subgraph Runtime["Conversation Runtime"]
    Query["query.ts"]
  end

  MCPTypes --> MCPClient --> Tools
  Tools --> TUC --> Query
  MCPClient --> AppState
  AppState --> Query
```

### 2) Data Flows

```mermaid
flowchart LR
  Config["ScopedMcpServerConfig"] --> Conn["建立 MCPServerConnection"]
  Conn --> Discover["发现 MCP tools/resources"]
  Discover --> Adapt["适配为内部 Tool 协议"]
  Adapt --> Pool["Tools 能力池"]
  Pool --> Ctx["ToolUseContext 注入"]
  Ctx --> Runtime["Conversation Runtime 可调用"]
  Conn --> State["AppState 同步连接态"]
```

### 3) Build Sequences

```mermaid
sequenceDiagram
  participant Shell as Interaction Shell
  participant MCP as Integration Fabric(MCP Client)
  participant Cap as Capability Orchestration
  participant State as AppStateStore
  participant Runtime as Conversation Runtime

  Shell->>MCP: 读取 ScopedMcpServerConfig
  MCP->>MCP: 建立/维护 MCPServerConnection
  MCP->>Cap: 将外部能力映射为 Tool
  Cap-->>MCP: 更新 Tools 池
  MCP->>State: 写入连接状态与可用能力快照
  Shell->>Runtime: 启动会话并注入 ToolUseContext
  Runtime->>Cap: 在 tool-use 时调用 MCP 注入能力
```

## 场景 D：插件加载并扩展 Command/Tools

### 1) Component Designs

```mermaid
flowchart TB
  subgraph Plugin["Plugin Ecosystem"]
    Loader["pluginLoader.ts"]
    Loaded["Entity: LoadedPlugin"]
  end

  subgraph Cap["Capability Orchestration"]
    Command["Entity: Command"]
    Tools["Entity: Tool / Tools"]
  end

  subgraph Intg["Integration Fabric"]
    MCP["Entity: ScopedMcpServerConfig / MCPServerConnection"]
  end

  subgraph State["State & Persistence"]
    AppState["Entity: AppState"]
  end

  Loader --> Loaded
  Loaded --> Command
  Loaded --> Tools
  Loaded --> MCP
  Command --> AppState
  Tools --> AppState
  MCP --> AppState
```

### 2) Data Flows

```mermaid
flowchart LR
  Source["插件源(本地/配置)"] --> Discover["发现与校验"]
  Discover --> Loaded["LoadedPlugin"]
  Loaded --> ExtCmd["注入 Command 扩展"]
  Loaded --> ExtTool["注入 Tool 扩展"]
  Loaded --> ExtMCP["注入 MCP 配置增量"]
  ExtCmd --> Pool["统一能力池"]
  ExtTool --> Pool
  ExtMCP --> Pool
  Pool --> State["AppState 同步可用能力与错误信息"]
```

### 3) Build Sequences

```mermaid
sequenceDiagram
  participant Shell as Interaction Shell
  participant Plugin as Plugin Loader
  participant Cap as Capability Orchestration
  participant Intg as Integration Fabric
  participant State as AppState

  Shell->>Plugin: 启动时触发插件加载
  Plugin->>Plugin: 发现/校验/解析 manifest
  Plugin-->>Cap: 注入 Command 扩展
  Plugin-->>Cap: 注入 Tool 扩展
  Plugin-->>Intg: 注入 MCP 配置增量
  Cap->>State: 写入能力池快照
  Intg->>State: 写入连接配置变化
  Plugin->>State: 写入 LoadedPlugin 状态/错误
```

## 场景 E：会话记忆更新（Session Memory）

### 1) Component Designs

```mermaid
flowchart TB
  subgraph Runtime["Conversation Runtime"]
    QE["Entity Aggregate: QueryEngine内部态"]
    Msg["Entity: Message"]
  end

  subgraph Memory["Context & Memory"]
    SessionMem["Entity: Session Memory 核心对象"]
    MemDir["memdir / sessionMemory service"]
  end

  subgraph State["State & Persistence"]
    AppState["Entity: AppState"]
  end

  QE --> Msg
  Msg --> SessionMem
  SessionMem --> MemDir
  SessionMem --> AppState
  AppState --> QE
```

### 2) Data Flows

```mermaid
flowchart LR
  Transcript["Message 转录流"] --> Eval["阈值判定(初始化/更新)"]
  Eval -->|达到阈值| Summarize["生成摘要/提炼记忆"]
  Eval -->|未达到阈值| Skip["保持现状"]
  Summarize --> Persist["写入 memdir / session memory"]
  Persist --> State["AppState 同步记忆状态"]
  State --> Runtime["后续 QueryEngine 上下文注入"]
```

### 3) Build Sequences

```mermaid
sequenceDiagram
  participant Runtime as Conversation Runtime
  participant QE as QueryEngine Aggregate
  participant Mem as Session Memory Service
  participant Store as memdir
  participant State as AppState

  Runtime->>QE: 完成一轮消息循环
  QE->>Mem: 提交最新 Message 与 usage
  Mem->>Mem: 按阈值判定是否更新记忆
  alt 需要更新
    Mem->>Store: 写入摘要/提炼结果
    Mem->>State: 更新记忆相关状态
  else 无需更新
    Mem-->>QE: 返回 no-op
  end
  State-->>Runtime: 下轮上下文包含最新记忆
```

## 二、统一建模约束（落地建议）

- 节点命名优先保持 `Domain + Entity` 术语一致：`AppState`、`Message`、`ToolUseContext`、`ToolPermissionContext`、`MCPServerConnection`、`LoadedPlugin`。
- `Conversation Runtime` 与 `QueryEngine内部态`建议持续分层：前者编排流程，后者维护会话聚合状态。
- 所有可执行能力（Tool/Command/MCP注入/插件注入）统一进入能力池与 `ToolUseContext`，避免旁路执行。
- 权限判定统一经 `ToolPermissionContext`，减少策略逻辑分散。
- 记忆更新以 `Message` 与会话聚合信号为输入，状态回写统一走 `AppState`。
