# Bun `bun:bundle` 特性开关（`feature()`）

本仓库里通过 `import { feature } from 'bun:bundle'` 使用的**编译期/打包期**开关。Bun 在运行或 `bun build` 时根据命令行上的 `--feature=NAME`（可重复）把 `feature('NAME')` 折叠成字面量 `true`/`false`，并做死代码消除；**不是** `process.env`（虽然部分功能仍会再读环境变量）。

**启用示例：**

```bash
bun run --feature=KAIROS --feature=DUMP_SYSTEM_PROMPT ./src/bootstrap-entry.ts
```

下列名称来自对 `src/**/*.ts(x)` 的扫描；若与上游有出入，以源码为准。

---

## 按领域分组

### CLI 入口与子命令（`entrypoints/cli.tsx`、`commands.ts` 等）

| 标志 | 作用（源码语义） |
|------|------------------|
| `ABLATION_BASELINE` | 与 `CLAUDE_CODE_ABLATION_BASELINE` 等配合，.harness 消融实验基线（顶层即设多类 env）。 |
| `DUMP_SYSTEM_PROMPT` | `--dump-system-prompt`：打印当前渲染的 system prompt 后退出（prompt 敏感度评测等）。 |
| `CHICAGO_MCP` | `--computer-use-mcp` 等计算机使用 / MCP 相关入口。 |
| `DAEMON` | `daemon`、`--daemon-worker` 子命令与守护进程路径。 |
| `BRIDGE_MODE` | `remote-control` / `rc` / `remote` / `sync` / `bridge` 等远程控制 CLI。 |
| `BG_SESSIONS` | `ps` / `logs` / `attach` / `kill` / `--bg` / `--background` 等后台会话命令。 |
| `TEMPLATES` | `new` / `list` / `reply` 等模板相关子命令。 |
| `BYOC_ENVIRONMENT_RUNNER` | `environment-runner` 子命令。 |
| `SELF_HOSTED_RUNNER` | `self-hosted-runner` 子命令。 |
| `TORCH` | 注册 `torch` 命令模块。 |
| `ULTRAPLAN` | Ultraplan 会话/启动对话框、输入区触发等（与 companion 流程相关 UI）。 |
| `BUDDY` | Companion / Buddy 精灵、通知与全屏布局等 UI。 |
| `NEW_INIT` | `init` 命令的新实现路径。 |

### 远程 / Bridge / Claude Code Remote（`bridge/`、`remoteBridgeCore` 等）

| 标志 | 作用 |
|------|------|
| `BRIDGE_MODE` | 是否编译进 bridge/远程控制相关逻辑（与 GrowthBook 字符串并存；见 `bridgeEnabled.ts` 注释）。 |
| `CCR_AUTO_CONNECT` | CCR 自动连接行为。 |
| `CCR_MIRROR` | 出站仅镜像等镜像模式（与 `outboundOnly` 等配合）。 |
| `CCR_REMOTE_SETUP` | 远程安装/引导流程。 |
| `DIRECT_CONNECT` | 通过 URL 等直接连接 pending 状态与 REPL 路径。 |
| `SSH_REMOTE` | SSH 远程 pending 与会话路径。 |

### Assistant / Kairos 家族（`main.tsx`、`bridgeMain.ts`、`assistant/` 等）

| 标志 | 作用 |
|------|------|
| `KAIROS` | 助手模式主开关：模块加载、`--session-id`/`--continue`、Brief、团队上下文、多类工具与 UI。 |
| `KAIROS_BRIEF` | Brief-only 流程与文案（与 `KAIROS` 部分并列）。 |
| `KAIROS_CHANNELS` | 频道相关 UI（如 `EnterPlanModeTool`、interactive helpers）。 |
| `KAIROS_DREAM` | 捆绑技能里与 “dream” 相关的路径。 |
| `KAIROS_GITHUB_WEBHOOKS` | `SubscribePRTool` 等 GitHub webhook 工具。 |
| `KAIROS_PUSH_NOTIFICATION` | 推送通知类工具。 |
| `PROACTIVE` | 主动发用户消息 / SendUserMessage 提示等与 `KAIROS` 并列的 proactive 路径。 |

### 协调 / 多智能体 / 工作流

| 标志 | 作用 |
|------|------|
| `COORDINATOR_MODE` | 协调器模式模块、CLI 与权限 handler。 |
| `WORKFLOW_SCRIPTS` | 本地工作流任务与 `WorkflowTool`。 |
| `AGENT_TRIGGERS` | 定时/触发器类工具（如 `ScheduleCronTool` 路径）。 |
| `AGENT_TRIGGERS_REMOTE` | 远程触发器工具。 |
| `MONITOR_TOOL` | 监控类工具与任务（`MonitorMcpTask` 等）。 |
| `FORK_SUBAGENT` | 子代理 fork 行为。 |
| `VERIFICATION_AGENT` | `TaskUpdateTool` 等与验证代理相关的门控。 |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | 内置 explore/plan 类 agent 定义。 |

### 上下文、查询与压缩（`query.ts`、`QueryEngine`、`compact/` 等）

| 标志 | 作用 |
|------|------|
| `REACTIVE_COMPACT` | 响应式压缩相关逻辑。 |
| `CONTEXT_COLLAPSE` | 上下文折叠、与 `CtxInspectTool` 等。 |
| `EXPERIMENTAL_SKILL_SEARCH` | 技能预取（skill prefetch）。 |
| `TEMPLATES` | 任务分类器 / job classifier 模块（与 stop hooks 中 `TEMPLATES` 同名复用）。 |
| `HISTORY_SNIP` | 历史片段 Snip 工具与查询路径。 |
| `BG_SESSIONS` | 后台会话任务摘要等（与 CLI `ps` 等同名复用）。 |
| `TOKEN_BUDGET` | 令牌预算追踪器。 |
| `CHICAGO_MCP` | 主线程非 agent 时的 MCP 相关 stop hook 行为。 |
| `CACHED_MICROCOMPACT` | 待处理缓存编辑与 micro-compact 路径。 |
| `PROMPT_CACHE_BREAK_DETECTION` | 检测 prompt cache 失效/断裂，影响 compact、API、子代理等。 |
| `COMPACTION_REMINDERS` | 压缩提醒（附件/会话元数据侧）。 |

### 权限、分类器与 Bash（`bashPermissions`、`permissions` 等）

| 标志 | 作用 |
|------|------|
| `TRANSCRIPT_CLASSIFIER` | 自动模式、`auto` 权限模式、迁移、多处工具与 REPL 门控。 |
| `BASH_CLASSIFIER` | Bash 命令分类/审批路径（与 tree-sitter 等配合）。 |
| `TREE_SITTER_BASH` / `TREE_SITTER_BASH_SHADOW` | Bash 解析与 shadow 分类实验路径。 |
| `POWERSHELL_AUTO_MODE` | PowerShell 自动模式相关。 |
| `EXTRACT_MEMORIES` | 停止钩子中抽取记忆模块。 |
| `CHICAGO_MCP` | 与 agent 上下文相关的 MCP 钩子（`stopHooks`）。 |

### Memory、Team、提取（`memdir/`、`teamMemorySync/` 等）

| 标志 | 作用 |
|------|------|
| `TEAMMEM` | 团队记忆路径、提示与密钥守卫。 |
| `MEMORY_SHAPE_TELEMETRY` | 记忆形状相关遥测。 |
| `AGENT_MEMORY_SNAPSHOT` | 主线程 agent 记忆快照更新路径。 |
| `BREAK_CACHE_COMMAND` | 注入 break-cache 命令上下文。 |

### 工具与 Skills（`tools.ts`、`skills/bundled`、`constants/tools.ts`）

| 标志 | 作用 |
|------|------|
| `WORKFLOW_SCRIPTS` | 见上。 |
| `OVERFLOW_TEST_TOOL` | 溢出测试工具。 |
| `TERMINAL_PANEL` | 终端面板与 `TerminalCaptureTool`。 |
| `WEB_BROWSER_TOOL` | 内置 WebView 时的浏览器工具与 Chrome 技能提示。 |
| `MESSAGE_ACTIONS` | 消息快捷操作与键位。 |
| `VOICE_MODE` | 语音模式键位与相关模块。 |
| `QUICK_SEARCH` | 快速搜索键位。 |
| `REVIEW_ARTIFACT` | 捆绑技能：评审产物。 |
| `BUILDING_CLAUDE_APPS` | 捆绑技能：构建 Claude Apps。 |
| `RUN_SKILL_GENERATOR` | 捆绑技能：技能生成器。 |
| `MCP_SKILLS` | 从 MCP 拉取 skills、资源列表等客户端逻辑。 |
| `MCP_RICH_OUTPUT` | MCP 工具输出用富文本组件折叠长输出。 |
| `CONNECTOR_TEXT` | 连接器文本 beta header（`constants/betas.ts`）。 |

（`CtxInspectTool` 的注册与 **`CONTEXT_COLLAPSE`** 共用同一标志，见「上下文、查询与压缩」一节。）

### UI / REPL / 主题（`REPL.tsx`、`PromptInput`、`keybindings` 等）

| 标志 | 作用 |
|------|------|
| `LODESTONE` | 交互辅助与主循环中的 “lodestone” 路径。 |
| `HISTORY_PICKER` | 历史选择器 UI 与搜索焦点行为。 |
| `AUTO_THEME` | 主题选择器中的自动主题。 |
| `STREAMLINED_OUTPUT` | 与 `CLAUDE_CODE_STREAMLINED_OUTPUT` 联用的精简流式输出（`cli/print`）。 |
| `ULTRATHINK` | `thinking.ts` 中的扩展思考路径。 |

### 遥测、日志、统计（`metadata.ts`、`stats.ts`、`telemetry/` 等）

| 标志 | 作用 |
|------|------|
| `CHICAGO_MCP` | 元数据里 MCP 相关字段。 |
| `COWORKER_TYPE_TELEMETRY` | coworker 类型遥测。 |
| `KAIROS` | 元数据中与 Kairos 活动相关字段。 |
| `ENHANCED_TELEMETRY_BETA` | 增强遥测 beta（再读 `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA` 等 env）。 |
| `PERFETTO_TRACING` | Perfetto 跟踪集成。 |
| `SHOT_STATS` | 统计中的 “shot” 分布与展示。 |
| `SLOW_OPERATION_LOGGING` | 慢操作日志。 |
| `ANTI_DISTILLATION_CC` | API 层反蒸馏相关请求头/行为。 |

### 设置同步、持久化、安装（`settingsSync`、`nativeInstaller`、`filePersistence` 等）

| 标志 | 作用 |
|------|------|
| `UPLOAD_USER_SETTINGS` | 向服务端上传用户设置（与 `main.tsx` 初始化等）。 |
| `DOWNLOAD_USER_SETTINGS` | 下载/合并远程设置（`settingsSync`、`cli/print`）。 |
| `FILE_PERSISTENCE` | 文件持久化路径。 |
| `ALLOW_TEST_VERSIONS` | 原生安装包允许测试版本（`download.ts`）。 |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | 选择对应 libc 的原生二进制（musl vs glibc）。 |
| `NATIVE_CLIENT_ATTESTATION` | 原生客户端认证相关 cookie 片段（`constants/system.ts`）。 |

### `setup.ts` 与其它零散门控

| 标志 | 作用 |
|------|------|
| `UDS_INBOX` | Unix domain socket inbox；`tools.ts` 中同时门控 `ListPeersTool`。 |
| `CONTEXT_COLLAPSE` | 安装/引导流程中的上下文折叠相关逻辑（与查询侧同标志）。 |
| `COMMIT_ATTRIBUTION` | 提交归因。 |
| `TEAMMEM` | 团队记忆相关 setup。 |
| `HARD_FAIL` | `main.tsx` 中硬失败/错误路径。 |
| `AWAY_SUMMARY` | 离开摘要 hook。 |
| `SKILL_IMPROVEMENT` | 技能改进 hook。 |
| `UNATTENDED_RETRY` | API 重试策略（`withRetry.ts`）。 |
| `SKIP_DETECTION_WHEN_AUTOUPDATES_DISABLED` | 自动更新关闭时跳过某类检测（`AutoUpdaterWrapper`）。 |
| `NATIVE_CLIPBOARD_IMAGE` | 剪贴板图片粘贴路径。 |

---

## 按字母排序的完整列表（简要）

| 标志 | 说明摘要 |
|------|-----------|
| `ABLATION_BASELINE` | Harness 消融基线 env 注入。 |
| `AGENT_MEMORY_SNAPSHOT` | Agent 记忆快照更新。 |
| `AGENT_TRIGGERS` / `AGENT_TRIGGERS_REMOTE` | 定时/远程触发器工具。 |
| `ALLOW_TEST_VERSIONS` | 安装器测试版本通道。 |
| `ANTI_DISTILLATION_CC` | API 反蒸馏。 |
| `AUTO_THEME` | 自动主题。 |
| `AWAY_SUMMARY` | 离开摘要。 |
| `BASH_CLASSIFIER` | Bash 分类审批。 |
| `BG_SESSIONS` | 后台会话 CLI + 查询/工具。 |
| `BREAK_CACHE_COMMAND` | Break-cache 命令注入。 |
| `BRIDGE_MODE` | Bridge/远程控制总开关。 |
| `BUDDY` | Companion UI。 |
| `BUILDING_CLAUDE_APPS` | 捆绑技能。 |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | 内置 explore/plan agents。 |
| `BYOC_ENVIRONMENT_RUNNER` | BYOC environment-runner CLI。 |
| `CACHED_MICROCOMPACT` | 缓存的微压缩编辑。 |
| `CCR_AUTO_CONNECT` / `CCR_MIRROR` / `CCR_REMOTE_SETUP` | CCR 连接/镜像/远程安装。 |
| `CHICAGO_MCP` | 计算机使用 MCP、钩子与元数据。 |
| `COMMIT_ATTRIBUTION` | 提交归因。 |
| `COMPACTION_REMINDERS` | 压缩提醒。 |
| `CONNECTOR_TEXT` | 连接器文本 beta。 |
| `CONTEXT_COLLAPSE` | 上下文折叠与相关工具。 |
| `COORDINATOR_MODE` | 协调器模式。 |
| `COWORKER_TYPE_TELEMETRY` | Coworker 遥测。 |
| `DAEMON` | 守护进程 CLI。 |
| `DIRECT_CONNECT` | URL 直连会话。 |
| `DOWNLOAD_USER_SETTINGS` | 下载远程设置。 |
| `DUMP_SYSTEM_PROMPT` | 导出 system prompt。 |
| `ENHANCED_TELEMETRY_BETA` | 增强遥测 beta。 |
| `EXPERIMENTAL_SKILL_SEARCH` | 实验性技能搜索预取。 |
| `EXTRACT_MEMORIES` | 停止时抽取记忆。 |
| `FILE_PERSISTENCE` | 文件持久化。 |
| `FORK_SUBAGENT` | Fork 子代理。 |
| `HARD_FAIL` | 硬失败路径。 |
| `HISTORY_PICKER` | 历史选择器 UI。 |
| `HISTORY_SNIP` | 历史 Snip 工具。 |
| `HOOK_PROMPTS` | Hook 请求 prompt 注入 REPL。 |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | 原生包 libc 变体。 |
| `KAIROS` 系列 | 助手模式及衍生能力。 |
| `LODESTONE` | Lodestone 交互路径。 |
| `MCP_RICH_OUTPUT` | MCP 富文本输出。 |
| `MCP_SKILLS` | MCP skills 拉取与 UI。 |
| `MEMORY_SHAPE_TELEMETRY` | 记忆形状遥测。 |
| `MESSAGE_ACTIONS` | 消息操作快捷键。 |
| `MONITOR_TOOL` | 监控工具。 |
| `NATIVE_CLIENT_ATTESTATION` | 客户端认证 header。 |
| `NATIVE_CLIPBOARD_IMAGE` | 剪贴板图片。 |
| `NEW_INIT` | 新 init 实现。 |
| `OVERFLOW_TEST_TOOL` | 测试工具。 |
| `PERFETTO_TRACING` | Perfetto。 |
| `POWERSHELL_AUTO_MODE` | PowerShell 自动模式。 |
| `PROACTIVE` | 主动用户消息路径。 |
| `PROMPT_CACHE_BREAK_DETECTION` | Prompt cache 断裂检测。 |
| `QUICK_SEARCH` | 快速搜索。 |
| `REACTIVE_COMPACT` | 响应式压缩。 |
| `REVIEW_ARTIFACT` | 评审产物技能。 |
| `RUN_SKILL_GENERATOR` | 技能生成器技能。 |
| `SELF_HOSTED_RUNNER` | self-hosted-runner CLI。 |
| `SHOT_STATS` | Shot 分布统计。 |
| `SKILL_IMPROVEMENT` | 技能改进 hook。 |
| `SKIP_DETECTION_WHEN_AUTOUPDATES_DISABLED` | 与自动更新互斥的检测跳过。 |
| `SLOW_OPERATION_LOGGING` | 慢操作日志。 |
| `SSH_REMOTE` | SSH 远程会话。 |
| `STREAMLINED_OUTPUT` | 精简流式 CLI 输出。 |
| `TEAMMEM` | 团队记忆。 |
| `TEMPLATES` | 模板 CLI + 任务分类器。 |
| `TERMINAL_PANEL` | 终端面板与捕获工具。 |
| `TOKEN_BUDGET` | 令牌预算。 |
| `TORCH` | torch 命令。 |
| `TRANSCRIPT_CLASSIFIER` | 自动模式/分类器总开关。 |
| `TREE_SITTER_BASH` / `TREE_SITTER_BASH_SHADOW` | Bash 解析与 shadow。 |
| `UDS_INBOX` | UDS inbox 与 peers。 |
| `ULTRAPLAN` | Ultraplan UI。 |
| `ULTRATHINK` | 扩展思考。 |
| `UNATTENDED_RETRY` | 无人值守重试。 |
| `UPLOAD_USER_SETTINGS` | 上传用户设置。 |
| `VERIFICATION_AGENT` | 验证代理。 |
| `VOICE_MODE` | 语音模式。 |
| `WEB_BROWSER_TOOL` | 浏览器工具。 |
| `WORKFLOW_SCRIPTS` | 工作流脚本工具。 |

---

## 维护说明

- 新增或重命名开关时，可在此文档同步更新；也可用下面命令重新枚举：

  `grep -rhoE "feature\\(['\"][A-Z0-9_]+['\"]\\)" src --include='*.ts' --include='*.tsx' | sed -E "s/feature\\(['\"]([A-Z0-9_]+)['\"]\\)/\\1/" | sort -u`

- 若要为 `feature()` 增加类型约束，可在例如 `env.d.ts` 中扩展 `declare module 'bun:bundle' { interface Registry { features: 'FLAG1' | 'FLAG2' | ... } }`（见 `node_modules/bun-types/bundle.d.ts`）。
