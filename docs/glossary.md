# 核心概念术语表

本表是全套文档的术语基准。每条约 2-3 行：定义、源码位置（已验证）、相关文档链接。

术语在写作过程中持续补充；源码位置以本仓库当前版本（v1.1.0）为准。

[返回索引](index.md)

## 先看：易混淆概念对照

pi 里有多组「同名不同物」的概念，先建立区分：

| 说法 | A | B |
|------|---|---|
| agent loop | `pi-agent-core` 的 `Agent`（生产链路唯一 loop） | `pi-durable` 的 task 化持久 loop（实验，概念平行演化） |
| 远程机制 | JSONL over stdio 的 **RPC 模式**（成熟，`modes/rpc/`） | CBOR 分帧的 **C/S 体系**（实验，`protocol/client/server`） |
| telemetry | `pi-telemetry` 契约包（无调用点） | coding-agent 的**安装遥测**（`core/telemetry.ts`，无关） |
| 会话存储 | coding-agent 自研 JSONL 会话树（生产） | durable 的 Storage（memory/SQLite/JSONL，实验） |
| Harness | pi 整体自我定位：agent harness | durable 的 `Harness` 对象（存储/调度入口） |
| provider | `pi-ai` 的模型服务提供方（40+） | 扩展 API 的 `registerProvider`（向 coding-agent 注册模型） |

## 通用概念

### harness（运行外壳）
对「把模型、工具、上下文、会话装配成一个可运行 agent」的统称。pi 的 harness 就是整个 coding-agent；durable 里的 `Harness` 是另一个具体对象（见下文）。

### agent loop（agent 循环）
模型调用与工具执行的往复循环：每轮把转录（transcript）发给模型；模型返回工具调用就执行、把结果追加回转录，直到模型不再调用工具或者被中止。
- 位置：`packages/agent/src/agent-loop.ts:38`（`agentLoop`）· 详见 [agent 运行时](packages/agent.md)

### turn（轮）
agent loop 中的一次「模型响应 + 其引发的工具执行」。`turn_end` 事件标志一轮结束，随后决定继续下一轮还是停止。参见 [agent 运行时](packages/agent.md)。

### session（会话）
一次持续交互的完整记录与状态载体。生产链路由 coding-agent 的 `SessionManager` 以 JSONL 文件持久化（见下）；实验链路由 durable 以记录 + 文档持久化。两者不共享格式。

### transcript / context（转录 / 上下文）
transcript 是历史消息的完整记录；context 是**实际发给模型**的窗口——经过压缩（compaction）、reset、分支选择之后的可见部分。压缩正是对两者差异的管理。

### tool call / tool result（工具调用与结果）
模型请求执行工具的结构（名称 + 参数）与执行回填的结果消息。工具契约是 `AgentTool`（`packages/agent/src/types.ts:466`），coding-agent 的内置工具（bash/edit/read 等）都实现它。

### steering（引导消息）
agent 运行过程中用户插入的引导：在每轮工具执行后被轮询消费，立即影响后续行为。与 follow-up 相对。

### follow-up（后续消息）
排队的用户消息，只在 agent「将要停止」时才交付，从而开启下一段工作。steering 与 follow-up 的注入点见 [agent 运行时](packages/agent.md)。

### compaction（上下文压缩）
当上下文接近 token 上限（或 provider 报 overflow）时，把较早的对话摘要成一条摘要消息、腾出窗口，会话得以继续。
- 位置：`packages/coding-agent/src/core/compaction/`（生产）· `packages/durable/src/harness/compaction.ts`（实验）

### token 预算 / overflow
每次请求可用 token 的约束（预算）与超限错误（overflow）。pi 的处理：先压缩再重试（生产与 durable 均如此）。

### harness 无关概念：faux provider
测试用的假 provider（`packages/ai/src/providers/faux.ts`），让测试不依赖真实 API。coding-agent 的测试套件（`test/suite/`）要求使用它。

## coding-agent 域（产品层）

### AgentSession
coding-agent 的核心会话对象：包装 `pi-agent-core` 的 `Agent`，负责会话文件、模型解析、工具集、扩展、压缩、事件转发（到 TUI/RPC/print）。
- 位置：`packages/coding-agent/src/core/agent-session.ts:378` · 详见 [核心运行时](packages/coding-agent-runtime.md)

### SessionManager / 会话树 / 分支 / fork
会话以 **append-only JSONL** 存储为**树**（可分支、可回退），`SessionManager` 负责读写、投影出某条路径的上下文。
- 位置：`packages/coding-agent/src/core/session-manager.ts:987`

### 四种运行模式
- **interactive**：默认全屏 TUI 交互模式。
- **print**（`-p` 或非 TTY）：单次执行、文本/JSON 输出。
- **json**：单次执行、机器可读事件流。
- **rpc**：JSONL over stdio 被外部进程驱动（IDE 集成等）。
- 分派位置：`packages/coding-agent/src/main.ts:112`（`resolveAppMode`）· 详见 [启动与运行模式](packages/coding-agent-startup.md)

### project trust（项目信任）
控制是否加载项目内的资源（扩展、skills 等）的开关。**它不构成沙箱**：工具仍以进程权限运行（见 `docs/security.md`）。

### extension（扩展）
通过 `ExtensionAPI` 注入的自定义能力：事件钩子、工具、命令、快捷键、provider、渲染器等。由 jiti 直接加载 TS，`builtin:<name>` 指内置扩展。
- 位置：`packages/coding-agent/src/core/extensions/types.ts:1564`（`ExtensionAPI`）、同文件 `:2017`（`ExtensionFactory`）· 详见 [扩展系统与 SDK](packages/coding-agent-extensions.md)

### skill（技能）
给模型的按需指令文件（如 `.pi/skills/*.md`），在合适时机注入提示，指导特定任务的做法。

### prompt template（提示模板）
可展开为提示文本的模板（`/命令` 或参数化片段），经 `core/prompt-templates.ts` 加载。

### theme（主题）
终端配色方案，经 `modes/interactive/theme/theme.ts` 加载。

### Pi package（Pi 包）
把扩展/skills/模板/主题打包经 npm 分发、用 `pi install` 安装的单元；扩展清单声明在包的 `pi.extensions` 字段。

### slash command（斜杠命令）
交互模式中 `/xxx` 触发的命令（`core/slash-commands.ts`）；扩展可注册自定义命令。

### install telemetry（安装遥测）
coding-agent 自带的匿名版本统计（`core/telemetry.ts`，`PI_TELEMETRY` 环境变量）。与 `pi-telemetry` 契约包**无关**。

## agent 包（pi-agent-core）

### Agent 类
有状态的运行时对象：持有消息与工具、驱动循环、管理队列（steering/follow-up）、发出事件。
- 位置：`packages/agent/src/agent.ts:188`

### AgentMessage
agent 层的消息联合类型；下游可用 TypeScript declaration merging 扩展自定义消息类型（扩展点）。
- 位置：`packages/agent/src/types.ts:374`

### AgentEvent
agent 运行时事件的联合类型（message_start/update/end、tool_execution_*、turn_end、agent_end 等），是 UI 层的唯一数据源。
- 位置：`packages/agent/src/types.ts:516`

### StreamFn
模型流式调用的注入契约：宿主（coding-agent）把「怎么调模型」注入 agent 包，因此 agent 包不依赖任何 provider。约定**不得 throw**，失败编码进流。
- 位置：`packages/agent/src/types.ts:33`、默认注入机制 `packages/agent/src/stream-fn.ts`

### replay（重放策略）
`AgentTool.replay?: "never" | "safe"`，声明工具在崩溃恢复后可否安全重跑。**注意：该字段在 agent 包中定义但生产 loop 不消费**；真正生效的是 durable 的 ToolTask。
- 位置：`packages/agent/src/types.ts:490`

### batch terminate（批次终止）
工具批执行中的提前退出语义：某工具结果要求终止时，本批剩余工具不再执行。agent 包与 durable 在此语义上保持一致。

## ai 包（pi-ai）

### provider / model（提供方 / 模型）
provider 是模型服务提供方（anthropic、openai、google 等 40+，规约 `<id>.ts` 工厂 + `<id>.models.ts` 生成包装）；model 是具体的模型条目（来自生成式模型目录）。

### wire API（线协议适配）
对不同厂商 HTTP 协议的适配层（15 个：anthropic-messages、openai-completions、google-generative-ai 等），provider 组合 wire API + 认证 + 模型数据。
- 位置：`packages/ai/src/api/`（每个 API 有 `.ts` 实现与 `.lazy.ts` 惰性包装）

### KnownApi / KnownProvider
已知 wire API 与 provider 的字符串联合类型（编译期/运行期收窄依据）。
- 位置：`packages/ai/src/types.ts:17`、`:43`

### Models 集合
统一模型调用入口：`stream`/`complete`/`generateImages`/`classify`/`getAuth`/`login` 等。
- 位置：`packages/ai/src/models.ts:250`（接口）、`:992`（`createModels`）、`:1041`（`createProvider`）

### AssistantMessageEvent
模型流式事件联合：start → text/thinking/toolcall 增量 → done/error。
- 位置：`packages/ai/src/types.ts:788`

### EventStream
事件流原语：push/end/result 语义，`AssistantMessageEventStream` 是其特化（终结为完整 AssistantMessage）。
- 位置：`packages/ai/src/utils/event-stream.ts:26`、`:97`

### 模型目录（models.generated）
从外部数据源（models.dev、OpenRouter、AI Gateway 等）生成的模型元数据产物；**不可手改**，改 `scripts/generate-models.ts` 后重新生成。

### virtual model（虚拟模型）
coding-agent 层的命名模型组合（自定义 provider + 模型 + 参数），扩展可注册。
- 位置：`packages/coding-agent/src/core/virtual-models.ts`

### OAuth 认证 / 凭据存储
订阅制 provider 的登录流程（PKCE、设备码等，`packages/ai/src/auth/oauth/`）与凭据持久化（`auth/resolve.ts`、`credential-store.ts`，刷新在锁内进行）。

## tui 包（pi-tui）

### Component / Focusable
UI 树的节点接口（渲染为字符串行数组、可失效缓存）与可聚焦扩展。
- 位置：`packages/tui/src/tui.ts:117`、`:180`

### TuiBase
渲染调度的抽象基类：焦点管理、overlay 栈、输入分发、`requestRender` 节流。
- 位置：`packages/tui/src/tui.ts:506`

### main-screen / alt-screen
两种 renderer：主屏模式（内容留在滚动历史中，逐行差分）与备用屏模式（全屏视口 + 约束布局）。

### 差分渲染（differential rendering）
只重绘变化区域：主屏比较首/末变更行（首变更在视口上方则全量），备用屏按行 diff。这是 pi-tui 的性能核心。

### overlay
叠在界面之上的浮动层（对话框等），有独立栈与焦点恢复语义。

### keybinding（键位）
可配置按键绑定，默认值在 `DEFAULT_EDITOR_KEYBINDINGS` / `DEFAULT_APP_KEYBINDINGS`（禁止在代码中硬编码按键判断）。

### native 模块
pi-tui 的平台原生扩展（prebuilds 中的 `.node`）：剪贴板、修饰键状态、Windows VT 输入。加载失败返回 undefined，调用方降级到命令行工具。

## mcp / codemode

### MCP（Model Context Protocol）
连接外部工具服务器的协议。pi-mcp 是**客户端**（另有有限的 server→client 请求支持）。
- 位置：`packages/mcp/src/client.ts:153`（`McpClient`）· 详见 [MCP 客户端](packages/mcp.md)

### transport（传输）
MCP 的物理通道：stdio（子进程）与 streamable HTTP（含 SSE 重连）。协议逻辑在客户端层，transport 只做 framing/IO。

### exposure（暴露方式）
MCP 工具进入 pi 的方式：直接作为工具（`tools`）或经 codemode 脚本调用（`codemode`，默认值）。决定了「MCP 使用时默认经过沙箱」这一关键行为。

### codemode 沙箱
QuickJS-WASM 隔离沙箱：模型写 JS 脚本，脚本内 `tools.<name>(args)` 调用工具；**嵌套调用结果不进 LLM 上下文**，只回注脚本输出。
- 位置：`packages/codemode/src/runtime/host.ts:340`（`CodemodeSandbox`）· 详见 [codemode](packages/codemode.md)

### prelude
沙箱 VM 内预注入的 JS 模板（构建 `tools`/`console`/`store` 等全局），以字符串模板形式存在。
- 位置：`packages/codemode/src/runtime/prelude-source.ts`

### host-call bridge
脚本内工具调用与宿主之间的消息桥（worker 线程 postMessage + 宿主 pending 表），宿主侧执行真实工具。

### store
codemode 脚本间持久化的键值状态，随 session 以 `codemode-store` entry 落盘。

## chord（实验轨基座）

### facet（插件单元）
chord 的可组合插件：声明依赖（requires）与提供（provides），有完整生命周期状态机。
- 位置：`packages/chord/src/api.ts:69`（`defineFacet`）

### service（服务）
带稳定 ID 的能力单元（`defineService`），分 singleton 与 keyed 实例；对消费者呈现为稳定 facade。
- 位置：`packages/chord/src/api.ts:73`

### replicated state（复制状态）
可在 provider/consumer 间同步的 JSON 状态：快照 + delta 更新 + 溢出 reset 语义。
- 位置：`packages/chord/src/api.ts:91`（`replicatedState`）

### delta（状态增量）
chord 独立的 JSON 变更引擎：`track` 接管根对象，draft 写入，`prepare` 生成 ops，`apply`/`applyImmutable` 重放。超量（4096 ops）折叠为全量替换。

### context（chord 上下文）
不可变的调用上下文链（Go 风格）：值查找沿父链、可派生取消。子路径导出 `chord/context`。

### facet bundle（插件包）
用 esbuild 打包为 content-addressed `.cjs` + manifest，经 `node:vm` 编译加载（绕过 Node 模块缓存），实现不重启替换。

### FacetKernel
chord 的插件宿主内核：装配、拓扑校序、激活、reload 切割、错误聚合。
- 位置：`packages/chord/src/facets/host.ts:340`

## durable（实验轨核心）

### entry（持久化条目）
不可变的转录记录（conversation 的四类记录之一），如 UserEntry/AssistantEntry/ToolResultEntry。定义工具：`defineEntry`（`packages/durable/src/entries.ts:6`）。

### document（文档）
带版本与迁移的可观察状态单元（如 `pi.live`、`pi.inbox`、`pi.agent`）。scope/history/fork 语义由 `defineDoc` 声明。
- 位置：`packages/durable/src/documents.ts:39`

### task（任务）
可恢复状态机单元：phase/checkpoint 显式建模，崩溃后可从 checkpoint 续跑。定义工具：`defineTask`（`packages/durable/src/tasks.ts:4`）。内置：GenerationTask / ToolTask / CompactionTask。

### 提交管线（commit pipeline）
durable 的一切写入路径：单写线串行队列 → 事务收集 writes → `storage.commit()` 原子落盘（返回 seq）→ 内存 tracker adopt → publish 通知观察者。**先提交、后可见**是其核心不变式。

### Tx / Session / Storage
durable 的三个核心契约：`Tx` 事务接口（`types.ts:766`）、`Session` 会话内核（`:921`）、`Storage` 存储后端接口（`:1015`）；后端实现有 memory / SQLite / JSONL / Cloudflare。

### submission（提交受理）
用户消息的受理记录：requestId 去重，经 queued → placed → done/unanswered 生命周期。

### ownership 树
任务与对话构成的归属树：中止自底向上级联，恢复时按树对账（running → pending）。

### Harness（durable）
durable 的入口对象：`Harness.open(storage)` 打开存储、创建/恢复会话、调度任务。
- 位置：`packages/durable/src/harness/harness.ts:411`

## env（实验轨）

### ExecutionEnv
执行环境契约（文件系统 + Shell 抽象），durable 通过它执行工具，因此可以本机（NodeExecutionEnv）或远端（RemoteExecutionEnv）。
- 位置：`packages/durable/src/env/index.ts:324`

### RemoteExecutionEnv / pi-env daemon
远端实现：TS 客户端经 SSH 与部署在对端的 Rust daemon 以帧协议通信，把工具执行搬到远端。
- 位置：`packages/env/src/remote-env.ts:351` · 详见 [远端执行环境](packages/env.md)

### 帧协议（frame protocol）
pi-env 的线协议：4 字节长度前缀 + 类型/ID/JSON + 二进制载荷；含取消、ping 判死、断线语义（见 `packages/env/docs/protocol.md`）。

## 远程 C/S 体系（protocol / client / server）

### PROTOCOL_VERSION / envelope（信封）
协议版本常量（当前 8）与消息信封（hello/request/cancel/response/service_event/attachment；CBOR 编码 + 4 字节大端分帧）。
- 位置：`packages/protocol/src/protocol.ts:5`

### hello 握手
连接建立：客户端首帧发版本，服务端校验后回 `ServerHello`（携带 serverId），超时 5 秒。

### attachment / presentation（附着/呈现）
同一 server 上某个 session 被某客户端「附着」的凭据与句柄；带外 `attachment` 消息发布路由变更（null 表示清除）。

### subscription（订阅）
服务事件流：先返回快照，再按 subscriptionId 推送增量（Chord 复制状态语义）。

### serverId / sessionId / attachmentId
三级路由围栏：server 级调用只需 serverId；session 级调用需三者齐备。

## telemetry / evals

### TelemetryContext / TelemetrySpan
回调式遥测契约：`startSpan(options, cb)` 恰好调用一次回调，以返回值/异常结算；span 结算后所有记录方法惰性化。
- 位置：`packages/telemetry/src/index.ts:14`、`:18`

### conformance 套件
第三方 adapter 必须通过的契约一致性测试集（`packages/telemetry/src/testing/conformance.ts`；durable/env 也有各自的 conformance）。

### host eval / docs eval / lift
pi-evals 的两种评测模式：host 在本机 in-process 跑 coding-agent；docs 用 Docker 双镜像（`without_docs`/`with_docs`）配对对比，量化文档带来的能力提升（lift）。

---

[返回索引](index.md) · 任务与进度：[roadmap](../roadmap/README.md)
