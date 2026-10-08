# 任务详情

每个任务产出一篇文档。执行规则、写作规范、通用自检清单见 [README.md](README.md)。路径均相对仓库根（`pi/`）。

---

## T01 · 核心概念术语表 → `docs/glossary.md`

**目标**: 建立全文档体系的术语基准。每条术语给出：中文一句话定义、归属包/机制、定义位置（源码路径，经 Grep 验证）、相关文档链接。按主题分组，标注混淆对（如「两套 agent loop」「两套 telemetry」）。

**术语清单（草稿，写作时增补与验证）**:

- 通用：harness、agent、agent loop、turn、session、transcript、context、tool call、tool result、steering、follow-up、compaction、token 预算/overflow
- coding-agent：project trust、extension、skill、prompt template、theme、Pi package、slash command、会话树/分支/fork、print/JSON/RPC/interactive 模式
- agent 包：`AgentMessage`、`AgentEvent`、`StreamFn`、`prepareRequest`/`prepareNextTurn` 钩子、declaration merging、batch terminate
- ai 包：provider、model、wire API（`KnownApi`）、virtual model、OAuth 认证、`AssistantMessageEvent`、模型目录（models.generated）
- tui：`Component`、`TuiBase`、main-screen/alt-screen、差分渲染、overlay、keybinding、native 模块
- mcp：MCP、transport（stdio/streamable-http）、OAuth PKCE、exposure（tools/codemode）
- codemode：sandbox、prelude、host-call bridge、store
- chord：facet、service、provider/consumer、replicated state、delta op、context、facet bundle
- durable：entry、document、task、phase/checkpoint、submission、commit 管线、ownership 树、replay 策略
- env：`ExecutionEnv`、`RemoteExecutionEnv`、daemon、帧协议
- 远程 C/S：envelope、hello 握手、attachment/presentation、subscription、serverId/sessionId/attachmentId
- telemetry：`TelemetryContext`/`TelemetrySpan`、schema、conformance 套件
- evals：host eval、docs eval、lift

**验收（补充通用清单）**: 每个术语的定义位置已验证；易混淆对在条目内互链说明；与各篇文档实际使用保持一致。

---

## T02 · 主体架构总览 → `docs/architecture.md`

**目标**: 一套文档的入口篇。讲清「pi 由什么组成、怎么分层、数据怎么流、生产与实验的边界」。

**内容大纲**:

1. 一句话概览 + 全景分层图（Mermaid）：用户 → coding-agent（入口/模式）→ core runtime（会话/工具/扩展）→ agent-core loop → ai（provider）→ 外部模型；tui 层；mcp/codemode；实验轨 chord/durable/env；远程 C/S；telemetry/evals 横切
2. 14 包依赖矩阵表：依赖/被依赖/生产或实验/规模（行数）
3. 生产链路解剖：一条用户输入从入口到模型到工具到落盘的骨架（细节指向各篇文档）
4. 实验轨解剖：与生产轨的差异对照表（内存 loop vs task 化持久 loop）
5. 两条远程路径对比：JSONL RPC 模式 vs CBOR C/S（表格）
6. 横切子系统：telemetry 契约现状、evals、install telemetry 的区分
7. monorepo 工程：workspaces、构建顺序、固定依赖/lockfile 策略、check 体系、二进制打包
8. 关键架构决策解读：tui 独立成包；agent-core 无 provider 依赖（靠 `StreamFn` 注入）；chord 零 Pi 依赖；durable 不依赖 agent 包；会话存储放在 coding-agent 而非 durable
9. 阅读地图：想了解 X → 读哪篇（链接本套文档）

**必读源码**:

- `package.json`（workspaces / build 顺序 / check 脚本）
- 各包 `packages/*/package.json`（依赖关系，抽查核对）
- `AGENTS.md`（工程规则）、`README.md`（顶层说明）
- `packages/coding-agent/src/index.ts`（SDK 公开面概览）、`packages/coding-agent/src/main.ts`（启动骨架）
- `packages/coding-agent/docs/index.md`（现有英文文档地图，用于阅读地图互链）

**验收（补充）**: 依赖矩阵与实际 `package.json` 抽查一致；双轨/混淆点全部覆盖；阅读地图链接有效。

---

## T03 · pi-ai → `docs/packages/ai.md`

**目标**: 讲清多 provider 统一 LLM API 的内部分层：类型与事件模型、provider 工厂、wire API 适配、认证、模型目录生成。

**必读源码**:

- `packages/ai/src/types.ts`（核心类型：消息/内容块 `AssistantMessageEvent`/`StreamOptions`/`KnownApi`/`KnownProvider`）
- `packages/ai/src/models.ts`（`createModels`/`createProvider`/`Models` 接口/`stream` 生命周期）
- `packages/ai/src/index.ts`（纯 core barrel 的边界）、`src/providers/all.ts`（`builtinModels`/`builtinProviders`）
- `packages/ai/src/utils/event-stream.ts`（`EventStream`/`AssistantMessageEventStream`）
- `packages/ai/src/api/lazy.ts`（惰性加载与 setup 失败转 error 事件）
- `packages/ai/src/api/anthropic-messages.ts`、`src/api/openai-completions.ts`（两个代表性 wire 适配）
- `packages/ai/src/auth/resolve.ts`、`src/auth/types.ts`、`src/auth/helpers.ts`、`src/auth/oauth/`（抽样 1-2 个流程）
- `packages/ai/src/providers/anthropic.ts` + `src/providers/anthropic.models.ts`（工厂与生成包装的规约）
- `packages/ai/src/utils/transcript.ts`（`normalizeContext`）、`src/utils/validation.ts`（`validateToolCall`）
- `packages/ai/scripts/generate-models.ts`（生成链路，读框架即可）、`src/models.generated.ts`（产物形态）
- `packages/ai/src/compat.ts`、`src/cli.ts`（login）、`src/models-store.ts`

**内容大纲**: ①概览与分层（core barrel / providers / api / auth 的边界与打包）②核心类型与事件模型 ③`Models` 集合与一次流式请求的生命周期 ④Provider 工厂与 wire API 适配层（15 个 API、42 个 provider 的规约）⑤认证与 OAuth（凭据存储、锁内刷新）⑥工具调用表示与校验 ⑦流式事件与惰性加载 ⑧模型目录生成链路（models.dev/OpenRouter/AI Gateway → data json → 生成代码）⑨扩展指南：新增 provider 的完整改动清单

**现有英文文档**: `packages/ai/README.md`（用户向 API 手册，引用不重复）。

---

## T04 · pi-agent-core → `docs/packages/agent.md`

**目标**: 讲清生产链路的 agent 循环：只有 6 个文件，可逐行级精读，是理解 pi 运行时语义的关键。

**必读源码**:

- `packages/agent/src/agent.ts`（`Agent` 类：`MutableAgentState`、`PendingMessageQueue` drain 模式、`ActiveRun` 生命周期、`processEvents` 状态归约与「await 全部监听者」屏障语义）
- `packages/agent/src/agent-loop.ts`（`runLoop` 内外双循环、三个钩子插入点、`declareToolChanges` 差量 system message、`streamAssistantResponse` 转换边界、`executeToolCallsSequential/Parallel` 的串行 preflight、`failToolCallsFromTruncatedMessage`、`runToolCall` 嵌套复用）
- `packages/agent/src/types.ts`（`StreamFn` 契约「不得 throw」、`AgentMessage` declaration merging、`AgentTool`、`AgentEvent`；注意 `replay` 字段定义但未被 loop 消费）
- `packages/agent/src/stream-fn.ts`（宿主注入默认模型运行时）、`src/proxy.ts`（SSE 代理，抽样）
- `packages/agent/test/agent-loop.test.ts`、`test/agent.test.ts`（行为规格）
- `packages/agent/examples/mcp-codemode/tools.ts`（把 MCP/codemode 包成 AgentTool）
- 下游装配参照：`packages/coding-agent/src/core/agent-session.ts`（只看 `Agent` 实例化与钩子接线部分，其余归 T07）

**内容大纲**: ①定位：与 pi-ai 的边界、与 coding-agent 的关系（生产唯一 loop）、与 durable 的平行演化 ②消息模型与两道转换边界（`convertToLlm`/`transformContext`）③agent loop 控制流：双循环、turn 生命周期、钩子时序（配 Mermaid 时序图）④工具系统：契约、校验与 hook 管线、串并行、batch terminate、截断防护、嵌套调用 ⑤`Agent` 类：状态归约、事件屏障、队列模型（steering/followUp）、失败路径（`handleRunFailure` 事件合成）⑥事件协议全谱与 UI 消费 ⑦传输层：`StreamFn` 契约与 proxy 模式

**现有英文文档**: `packages/agent/README.md`（API 文档）。

---

## T05 · pi-tui → `docs/packages/tui.md`

**目标**: 讲清终端 UI 框架的渲染管线与输入管线，以及 native 模块的边界。

**必读源码**:

- `packages/tui/src/tui.ts`（`Component`/`Container`/`TUI`/`TuiBase`；`requestRender` 节流；overlay 栈与焦点）
- `packages/tui/src/tui-main-screen.ts`、`src/tui-alt-screen.ts`（两种 renderer 的差分算法与全量重绘条件）
- `packages/tui/src/terminal.ts`（`Terminal` 接口与 `ProcessTerminal`：raw mode、Kitty 协议协商、resize）
- `packages/tui/src/stdin-buffer.ts`（转义序列超时、bracketed paste、Kitty 事件）
- `packages/tui/src/keys.ts`、`src/keybindings.ts`（按键解析与全局键位注册表）
- `packages/tui/src/layout.ts`、`src/layout-node.ts`、`src/components/stack.ts`（备用屏约束布局：basis/grow/shrink）
- `packages/tui/src/components/editor.ts`、`src/components/scroll-view.ts`（抽样两个复杂组件）
- `packages/tui/src/native-platform.ts`、`src/native-module-path.ts`（加载失败降级路径）；`packages/tui/native/`（构建脚本与 prebuilds）
- `packages/tui/test/virtual-terminal.ts`（测试设施，@xterm/headless）

**内容大纲**: ①总览与三层分层（TuiBase–renderer–组件）②`Component`/`TUI` 抽象与渲染调度 ③主屏差分渲染算法（firstChanged/lastChanged、全量重绘触发条件）④备用屏视口与约束布局求解 ⑤输入管线（stdin → buffer → keys → keybindings → focused component；鼠标/paste）⑥焦点与 overlay ⑦组件库与缓存失效约定 ⑧终端能力探测/图片/色彩（oklab）⑨native 模块：用途、构建流程、降级 ⑩测试与调试（virtual-terminal、`PI_TUI_WRITE_LOG`）

**现有英文文档**: `packages/tui/README.md`（用户 API，34KB）。

---

## T06 · coding-agent 启动与运行模式 → `docs/packages/coding-agent-startup.md`

**目标**: 讲清 CLI 从进程启动到进入某个运行模式的完整路径，以及四种模式各自的架构。

**必读源码**:

- `packages/coding-agent/src/cli.ts` → `src/cli/setup.ts`；`src/main.ts`（总调度：`resolveAppMode` 与模式分派终点）
- `packages/coding-agent/src/cli/args.ts`（参数解析）、`cli/startup-ui.ts`、`cli/session-picker.ts`、`cli/auth-check.ts`、`cli/file-processor.ts`
- `packages/coding-agent/src/modes/print-mode.ts`、`src/modes/json-event.ts`
- `packages/coding-agent/src/modes/interactive/interactive-mode.ts`（分批读，250KB：与 pi-tui 的接线、组件树、事件渲染）
- `packages/coding-agent/src/modes/rpc/`（`rpc-mode.ts`、`rpc-types.ts`、`rpc-client.ts`）——JSONL 协议栈
- `packages/coding-agent/src/rpc-entry.ts`（`package.json` exports `./rpc-entry`）
- `packages/coding-agent/src/bun/`（二进制发行入口链 `cli.ts` → `runtime-setup.ts`）
- `packages/coding-agent/src/config.ts`（资产路径 helpers，AGENTS.md 规定的唯一合法方式）、`src/migrations.ts`

**内容大纲**: ①入口与启动时序（含 Mermaid：cli.ts → main.ts → args → auth/session 选择 → 模式）②四模式分派规则与对比表（interactive/print/json/rpc，含 TTY 判定）③Interactive 模式架构：主类与 TUI 组件树、事件订阅 ④RPC 协议栈：JSONL 命令集、与「CBOR C/S 体系」（T14）的区分 ⑤bun 二进制发行链 ⑥首次启动的数据迁移（migrations）

**现有英文文档**: `packages/coding-agent/docs/` 下 `cli.md`、`json.md`、`rpc.md`、`rpc-commands.md`、`rpc-extension-ui.md`、`cli-integration.md`、`usage.md`、`keybindings.md`。

---

## T07 · coding-agent 核心运行时 → `docs/packages/coding-agent-runtime.md`

**目标**: 本套文档的重头。讲清 AgentSession 的生命周期、工具系统、JSONL 会话树、上下文组装与压缩、模型层。

**必读源码**:

- `packages/coding-agent/src/core/agent-session.ts`（4398 行，分批精读：prompt/abort/compact/模型切换/树导航）
- `packages/coding-agent/src/core/agent-session-runtime.ts`、`core/agent-session-services.ts`（cwd 绑定服务装配）
- `packages/coding-agent/src/core/session-manager.ts`（2013 行：JSONL 会话树读写、投影、`buildSessionContext`、文件格式）
- `packages/coding-agent/src/core/sdk.ts`（`createAgentSession*` 装配链）
- `packages/coding-agent/src/core/compaction/compaction.ts`、`core/compaction/branch-summarization.ts`
- `packages/coding-agent/src/core/tools/index.ts`、`core/tools/tool-definition-wrapper.ts`、`core/tools/file-mutation-queue.ts`、`core/tools/truncate.ts`、`core/tools/output-accumulator.ts`、`core/tools/renderers/`（抽样）
- `packages/coding-agent/src/core/system-prompt.ts`、`core/messages.ts`（`convertToLlm`）、`core/nested-tool-calls.ts`
- `packages/coding-agent/src/core/model-runtime.ts`、`model-registry.ts`、`model-resolver.ts`、`virtual-models.ts`、`auth-storage.ts`
- `packages/coding-agent/src/core/settings-manager.ts`、`core/trust-manager.ts`、`core/project-trust.ts`
- 运行保障抽样：`core/bash-executor.ts`、`core/event-bus.ts`、`core/telemetry.ts`（install telemetry）、`core/cache-warmer.ts`、`core/usage-totals.ts`

**内容大纲**: ①AgentSession 定位：与 agent-core `Agent` 的层次关系（装配与事件转发）②生命周期：创建（SDK/模式两条路径）、prompt/abort/切换模型/树导航 ③工具系统：内置工具清单、AgentTool 实现规约、并发与文件写串行化、输出截断与渲染分工 ④会话持久化：JSONL 文件格式与 entry 类型、会话树与分支/fork、`buildSessionContext` ⑤上下文组装：system prompt 拼装、`convertToLlm`、skills/templates 注入点 ⑥压缩：阈值/切点/摘要、分支摘要、cache warming ⑦模型层：解析/切换/凭据存储/虚拟模型 ⑧trust 与 settings ⑨SDK 装配链（`createAgentSession` → services → runtime）

**现有英文文档**: `packages/coding-agent/docs/` 下 `how-pi-works.md`、`sessions.md`、`session-format.md`、`compaction.md`、`models.md`、`providers.md`、`virtual-models.md`、`settings.md`、`security.md`、`environment-variables.md`。

---

## T08 · coding-agent 扩展系统与 SDK → `docs/packages/coding-agent-extensions.md`

**目标**: 二次开发的第一落点。讲清扩展如何被加载与执行、能注册什么、资源加载体系、Pi package 机制、SDK 面。

**必读源码**:

- `packages/coding-agent/src/core/extensions/types.ts`（84KB，分批读：`ExtensionAPI` 定义、`ExtensionFactory`、约 40 个事件）
- `packages/coding-agent/src/core/extensions/loader.ts`（jiti 加载、别名映射、`builtin:<name>`）、`core/extensions/runner.ts`（执行器）
- `packages/coding-agent/src/core/package-manager.ts`（扩展发现路径：全局/项目目录 + npm 包 `pi.extensions` 清单）、`core/pi-manifest.ts`、`src/package-manager-cli.ts`
- `packages/coding-agent/src/core/resource-loader.ts`、`core/skills.ts`、`core/prompt-templates.ts`、`src/modes/interactive/theme/theme.ts`
- `packages/coding-agent/src/extensions/index.ts`（`builtInExtensions`）、`src/extensions/mcp/`、`src/extensions/codemode/index.ts`、`src/extensions/llama/`、`src/extensions/tool-search/`（抽样）
- `packages/coding-agent/src/index.ts`（公开导出面）、`core/sdk.ts`（SDK 入口）
- `packages/coding-agent/examples/`（sdk 示例、扩展样例目录结构）

**内容大纲**: ①扩展体系总览：发现 → 加载（jiti）→ 注册 → 执行（runner 事件分发）完整时序（Mermaid）②`ExtensionAPI` 能力清单：事件钩子分类、`registerTool/Command/Shortcut/Flag/Provider/VirtualModel/Renderer`、消息与 UI 上下文 ③Loader 细节：TS 直接加载、别名解析、内置扩展机制 ④资源加载体系：skills/prompt templates/themes 的发现与优先级 ⑤Pi package：manifest 格式、安装/更新/卸载（`pi install`）⑥内置扩展设计：mcp/codemode/llama/tool-search 各自激活条件 ⑦SDK：`createAgentSession` 用法与导出面 ⑧二次开发落点：写扩展/包/主题分别带动哪些文件

**现有英文文档**: `packages/coding-agent/docs/` 下 `extensions.md`、`skills.md`、`prompt-templates.md`、`themes.md`、`packages.md`、`slash-commands.md`、`sdk.md`、`custom-provider.md`。

---

## T09 · pi-mcp → `docs/packages/mcp.md`

**目标**: 讲清 MCP 客户端的实现与在 coding-agent 中的集成层。

**必读源码**:

- `packages/mcp/src/client.ts`（615 行：状态机、initialize 版本协商、分页 `tools/list`、`tools/call` 进度/超时/取消、roots/ping handler）
- `packages/mcp/src/protocol/jsonrpc.ts`、`protocol/types.ts`（协议版本常量与兼容范围）、`protocol/content.ts`（`toLlmContent`）
- `packages/mcp/src/transports/transport.ts`（分层契约）、`transports/stdio.ts`（进程组 kill、Windows taskkill）、`transports/streamable-http.ts`（GET 流重连、`Last-Event-ID`、session 过期）
- `packages/mcp/src/oauth/`（discovery/flow/callback/provider，PKCE 全流程，抽样）
- `packages/mcp/src/index.ts`、`./testing`（`createInMemoryTransportPair`）与 `./oauth` 子路径
- 集成层：`packages/coding-agent/src/extensions/mcp/runtime.ts`、`tools.ts`、`resources.ts`、`config.ts`；exposure 默认值在 `packages/coding-agent/src/core/mcp-servers.ts`（`"codemode"`）

**内容大纲**: ①分层架构：client core / transport / oauth 各层职责 ②状态机与错误矩阵（重连、session 过期、stdio 关闭协议）③三种 transport 的选型与参数 ④OAuth 子流程逐步骤（发现 → 动态注册 → 授权 → 换 token → step-up）⑤与 pi 的接入：`toLlmContent`、工具包装、`exposure` 机制（tools vs codemode）、needs-auth 状态 ⑥测试：内存传输对端 ⑦明确的不支持清单（采样/tasks/server 端等）

**现有英文文档**: `packages/mcp/README.md`、`packages/coding-agent/docs/mcp.md`、`packages/agent/examples/mcp-codemode/`。

---

## T10 · pi-codemode → `docs/packages/codemode.md`

**目标**: 讲清 QuickJS-WASM 沙箱的执行链路与在 coding-agent 中的集成。

**必读源码**:

- `packages/codemode/src/runtime/host.ts`（宿主核心：`CodemodeSandbox`/`Execution`、worker 生命周期、超时/abort、`SharedArrayBuffer` interrupt）
- `packages/codemode/src/runtime/worker.ts`（VM 实例化、WASI shim、host-call 注册）
- `packages/codemode/src/runtime/protocol.ts`（宿主↔worker 消息联合与校验）
- `packages/codemode/src/runtime/prelude-source.ts`（VM 内 prelude 字符串模板与容量常量）
- `packages/codemode/src/source.ts`（`@options` 解析、`CODEMODE_SOURCE_GRAMMAR`）、`src/declarations.ts`（`renderDeclarations`、MCP preamble、`$defs` 展开）
- `packages/codemode/src/wasm.ts`（`loadQuickJSWasm` 缓存与打包注入）、`src/identifier.ts`、`src/types.ts`
- 集成层：`packages/coding-agent/src/extensions/codemode/tool.ts`、`execute.ts`（`models.*` 命名空间、store → session entry）、`execute.lazy.ts`（懒加载）、`index.ts`（`defaultActive: false`）

**内容大纲**: ①定位与安全模型：唯一能力=调工具、嵌套调用不进 LLM context、wasm 隔离与内存/输出上限 ②模块地图：宿主侧 vs VM 内 ③执行全链路（Mermaid：execute → worker → VM/prelude → bridge → 结算）④中断与失败模型：timeout/abort/interrupt flag/terminate 四态错误分类 ⑤store/load 持久化与 session entry 衔接 ⑥集成指南：独立库用法（wasm/workerUrl 注入、Bun 打包）与 pi 扩展用法（声明渲染、lazy、MCP 联动）

**现有英文文档**: `packages/codemode/README.md`、`packages/coding-agent/docs/codemode.md`。

---

## T11 · chord → `docs/packages/chord.md`

**目标**: 实验轨基座的实现解剖。概念密度最高，篇幅按「中-大」对待。**开篇必须声明：当前不在默认运行路径，是下一代宿主基座。**

**必读源码**:

- `packages/chord/README.md`、`packages/chord/PLANNING.md`（规格 + 已实现清单；文档应做「规格 → 实现 → 现状差异」映射）
- `packages/chord/src/index.ts` + `src/api.ts`（公开 API 全貌）、`src/types.ts`（全部契约类型）
- `packages/chord/src/facets/host.ts`（`FacetKernel` 生命周期状态机、拓扑排序、reload 切割）、`src/facets/loader.ts`
- `packages/chord/src/services/provider.ts`、`consumer.ts`、`state.ts`、`state-codec.ts`、`handle.ts`、`instances.ts`、`wire.ts`（抽样重点：provider/consumer/state）
- `packages/chord/src/delta/README.md`、`src/delta/tracker.ts`（overlay 写入模型）、`src/delta/diff.ts`（抽样）
- `packages/chord/src/context/index.ts`（小而完整）、`src/node/bundle.ts`、`src/node/bundle-loader.ts`
- `packages/chord/test/`（行为契约抽样：`state-fuzz.test.ts` 等）

**内容大纲**: ①定位与边界：为什么 chord 不属于 Pi（零 Pi 依赖铁律）、当前消费格局与 experimental 门控 ②分层架构与模块地图 ③核心概念模型：facet/service/replicated state/delta/context 关系图 ④FacetKernel 生命周期深潜（activate 各阶段、shape-preserving reload）⑤服务与状态传输：provider↔consumer 管线、快照/更新/reset 协议、codec 与 cursor ⑥delta 引擎原理：overlay 写入、diff 策略、所有权契约 ⑦Node 侧打包/加载：manifest、vm 加载、原子替换 ⑧测试体系与扩展点 ⑨现状标注：对称 RPC 未实现（`src/rpc/` 不存在）

**现有英文文档**: `packages/chord/README.md`（规格视角）、`PLANNING.md`。

---

## T12 · durable → `docs/packages/durable.md`

**目标**: 实验轨核心：持久化 agent 运行时的完整解剖。**开篇声明 Experimental 状态与「与 agent 包平行演化、不互相依赖」的关系。**

**必读源码**:

- `packages/durable/src/index.ts`、`src/types.ts`（全部契约：Storage/Session/Tx/TaskDefinition 等）
- `packages/durable/src/session/session.ts`（单写线 `#enqueue`、提交流水线 commit → adopt → publish）、`src/session/transaction.ts`（`StorageWrite` 生成与 adopt）
- `packages/durable/src/documents.ts`（`defineDoc`：scope/history/fork/version 语义）、`src/entries.ts`（内置 entry kind）
- `packages/durable/src/harness/harness.ts`（`Harness.open`、Conversation 门面、5 个内置文档）、`harness/scheduler.ts`（ownership 树、状态机、resume 对账、abort 级联）、`harness/generation.ts`（phase 机、占位式系统提示、overflow 重试）、`harness/tool.ts`（intent 先提交、`replay:"safe"`）、`harness/compaction.ts`
- `packages/durable/src/harness/submissions.ts`、`inbox.ts`、`live.ts`、`context.ts`、`prompt.ts`、`events.ts`、`view.ts`（抽样重点：submissions/context/events）
- `packages/durable/src/storage/memory.ts`、`storage/sqlite/migrations.ts`（schema 全文）、`storage/jsonl/storage.ts`（提交标记 + sidecar，抽样）
- `packages/durable/src/env/index.ts`（`ExecutionEnv` 契约，与 T13 衔接）
- `packages/durable/src/tools/`（CodingTools 与 file-mutation-queue，抽样）
- `packages/durable/docs/spec.md`（索引与差异标注）；`test/examples/00..31`（选择 2-3 个典型例子读）
- 实验集成参照：`packages/coding-agent/src/experimental/durable/`（只看装配部分）

**内容大纲**: ①定位与依赖：为什么需要 durable（对比 agent 包的内存 loop）；chord 与 pi-ai 各提供什么 ②内核分层总览（Storage 契约 → Session 提交管线 → 文档模型 → Harness → Scheduler 调用关系图）③持久化数据模型：四类记录 + document；SQLite schema 与 JSONL 布局；ID/seq 与迁移 ④文档系统：defineDoc 语义、tracker、delta、replicatedState、snapshotAsOf ⑤任务系统：phase/checkpoint、Generation/Tool/Compaction 三个内置 task 拆解、ownership 树与 abort 级联、replay 策略（与 agent 包 `AgentTool.replay` 未消费的对照）⑥对话协议：submit/submission 生命周期、reset、fork、子 agent、usage ⑦观察面：view/watch/watchEvents/taskGraph ⑧崩溃恢复流程（open 对账 → resume → 续跑）⑨二次开发：自定义 task/doc/tool/extension、storage 后端开发

**现有英文文档**: `packages/durable/README.md`（功能手册）、`packages/durable/docs/spec.md`。

---

## T13 · pi-env → `docs/packages/env.md`

**目标**: 讲清「远端执行环境」的 TS 客户端 + Rust daemon 双端架构与线协议。

**必读源码**:

- `packages/env/src/index.ts`、`src/connection.ts`（帧协议客户端、daemon 懒启动、同步行握手、ping 判死、`lost` 语义）
- `packages/env/src/remote-env.ts`（`RemoteExecutionEnv` 实现 durable `ExecutionEnv` 全操作；分块流水线）
- `packages/env/src/ssh.ts`（host key 流程、`sshArguments` 硬化、`deployDaemon` 校验与懒连接）
- `packages/env/src/watch.ts`、`src/errors.ts`（错误码映射）
- `packages/env/docs/protocol.md`（线协议 v1 全文）、`docs/semantics.md`（与 NodeExecutionEnv 等价规范）
- `packages/env/daemon/src/main.rs`（serve 循环、token、超时退出）、`frame.rs`、`exec.rs`（进程组/超时/spill）、`watch.rs`（抽样）
- `packages/durable/src/env/index.ts`（契约对照）
- 构建发布：`.github/workflows/env-daemons.yml`、`env.yml`（对等性测试）

**内容大纲**: ①定位：为什么需要远端 `ExecutionEnv`；契约来源（durable）与本地对照实现 ②总体架构：控制面/数据面划分、生命周期（探测/部署/启动/退出）③线协议：帧格式、同步行、取消、liveness、错误码（与 `protocol.md` 的索引关系）④SSH 集成与安全：host key、硬化参数、daemon 部署与校验、断线重连语义 ⑤daemon 内部：线程模型、exec、输出 window、watch、平台差异（Windows Job Objects）⑥语义等价验证策略 ⑦二次开发：新增一个 op 的端到端改动清单；构建/发布链路（checkout 无 `bin/` 的说明）

**现有英文文档**: `packages/env/README.md`（较短）、`docs/protocol.md`、`docs/semantics.md`。

---

## T14 · 远程会话 C/S 体系（protocol + client + server）→ `docs/packages/remote-sessions.md`

**目标**: 三包合一，讲清 CBOR 远程会话协议体系，并用对比表与 JSONL RPC 模式明确区分。

**必读源码**:

- `packages/protocol/src/protocol.ts`（`PROTOCOL_VERSION`、消息联合、三级路由）、`src/codec.ts`、`src/framing.ts`、`src/cbor/encoder.ts`、`src/cbor/decoder.ts`、`src/cbor/options.ts`（限制常量）
- `packages/client/src/client.ts`（`Client`：请求表、订阅生命周期、attachment 状态）、`src/connection.ts`（代次、握手、背压语义）、`src/unix.ts`（传输工厂与 `discoverUnixServers`）、`src/transport.ts`、`src/errors.ts`
- `packages/server/src/server.ts`（accept/dispatch、握手超时、Chord 增量编码器、failProtocol）、`src/session-router.ts`（attach/detach、attachmentId、per-client 串行化）、`src/types.ts`（`ServerHost` 契约）、`src/transports/unix/listener.ts`、`src/testing/host.ts`
- 集成层：`packages/coding-agent/src/experimental/server.ts`、`client-runtime.ts`、`session-worker-manager.ts`（抽样）
- 对比参照：`packages/coding-agent/src/modes/rpc/`（JSONL RPC，只用于对比表，不展开）

**内容大纲**: ①体系定位：实验性「远程 pi session」；与 JSONL RPC 模式的对比表（协议、传输、会话模型、成熟度、使用方）②protocol：分层（framing → CBOR → envelope → Chord payload）、消息词汇表、路由模型（serverId/sessionId/attachmentId 三级围栏）、握手与错误码 ③client：Connection/Client 两层职责、状态机、订阅水合、断线语义（不自动重连）④server：`ServerHost` 契约、attachment 状态机、请求/订阅处理流水线 ⑤Unix 传输与 serverId 约定 ⑥用 `./testing` 写协议测试

**现有英文文档**: `packages/protocol/README.md`、`packages/client/README.md`、`packages/server/README.md`。

---

## T15 · pi-telemetry → `docs/packages/telemetry.md`

**目标**: 短篇。澄清「契约包、零调用点」的现状，与 coding-agent 安装遥测区分。

**必读源码**:

- `packages/telemetry/src/index.ts`（契约与类型层）、`src/noop.ts`、`src/memory.ts`（参考实现设计取舍）
- `packages/telemetry/src/testing/conformance.ts`（分组用例清单化）、`src/testing/types.ts`
- 消费者与现状证据：`packages/ai/src/types.ts`（`telemetryContext?`）、`packages/ai/src/api/simple-options.ts`（透传）、`packages/ai/test/telemetry-options.test.ts`
- 区分对象：`packages/coding-agent/src/core/telemetry.ts`（install telemetry，与契约包无关）

**内容大纲**: ①定位：为什么是契约包而非 SDK；与 install telemetry 的区别（加粗提醒）②契约语义：`TelemetryContext`/`TelemetrySpan` 的结算与惰性语义 ③实现解剖：noop 与 InMemory 的取舍 ④类型层：schema → 精确推导 → typed starter，运行时零成本 ⑤现状与接入路径：ai 仅类型透传、agent schema 未实现；adapter 自备 exporter + conformance 套件用法

**现有英文文档**: `packages/telemetry/README.md`（详尽但描述目标态，需标注差异）。

---

## T16 · pi-evals → `docs/packages/evals.md`

**目标**: 短篇。讲清两种评测模式与 harness API，面向「以后要写 eval」的读者。

**必读源码**:

- `packages/evals/src/cli.ts`（docs runner 全流程）、`src/harness.ts`（`createPiCodingAgentHarness`/`createPiDocumentationEvalHarness`/`applyIsolatedEnvironment`/`verifySystemPrompt`）
- `packages/evals/src/plan.ts`（变体常量、任务规划、奇偶轮交替）、`src/docker.ts`（镜像构建与容器执行）、`src/report.ts`（配对统计 lift、blocked pair）
- `packages/evals/docker/entrypoint.ts`、`docker/install-runtime.mjs`（抽样）
- `packages/evals/evals/`（`smoke.eval.ts`、`documentation-audit.eval.ts`）、`vitest.evals.config.ts`

**内容大纲**: ①两种模式对比与选择（host vs docs）②harness API 参考 + 写 eval 模板（host 与 docs 各一）③Docker 隔离与镜像构建链 ④任务规划与配对统计（lift、blocked pair 处理）⑤artifact schema 与排障 ⑥运行成本与重复次数

**现有英文文档**: `packages/evals/README.md`。

---

## T17 · 关键数据流时序图 → `docs/data-flows.md`

**目标**: 汇总篇。用 Mermaid 时序图把散在各包文档里的关键流程串成端到端的完整链路。每条流：触发点、参与文件（file:line）、数据形态转换。

**必含流程**:

1. 启动链路：shell → `cli.ts` → `main.ts` → `resolveAppMode` → InteractiveMode
2. 一次用户消息全链路（生产轨）：interactive-mode 提交 → `AgentSession.prompt` → agent-core loop → pi-ai `Models.stream` → provider SSE → 流事件 → tool calls → 工具执行（含 file-mutation-queue）→ 事件 → TUI 重渲染 → JSONL 落盘
3. compaction 触发与恢复（阈值/overflow 两种触发）
4. 扩展生命周期：发现 → 加载 → initialize → 事件钩子参与以上链路的位置
5. 实验轨对照：durable submit → scheduler → generation phase → pi-ai stream（节流提交）→ commit → watchEvents

**可选**: MCP 工具经 codemode 沙箱执行的链路。

**依赖**: T03、T04、T07 完成后再写（流程引用其 file:line）；流程 5 参考 T12。

---

## T18 · 二次开发指南 → `docs/development.md`

**目标**: 从「学习架构」过渡到「动手改」的操作手册。

**内容大纲**:

1. 开发环境：`npm install --ignore-scripts`、构建顺序（含 `build:offline`）、`test.sh`、`pi-test.sh`（从任何目录跑源码版 pi）、`check` 体系逐项说明
2. 本地运行与调试：`PI_EXPERIMENTAL`、`PI_TUI_WRITE_LOG`、profile 脚本、tmux 交互测试（`.pi/skills/interactive-testing.md`）、bug-report 机制
3. **扩展点地图**（核心表格）：需求 → 机制 → 包/文件 → 现有英文文档。覆盖：工具、slash command、快捷键、flag、事件钩子、provider、virtual model、system prompt、skill、template、theme、会话存储、RPC 命令、SDK 嵌入、MCP、codemode、durable task（实验）、env op（实验）
4. 各机制最小示例索引（指向 `packages/coding-agent/examples/` 与 docs/，不重写教程）
5. 打包与分发：`pack:packages`、`use-local-packages.mjs`、二进制构建（bun / `scripts/build-binaries.sh`）、install-lock 机制
6. 测试设施：`test.sh`、coding-agent `test/suite/`（harness + faux provider，禁止真实 API）、tui virtual-terminal、各包 conformance 套件
7. 注意事项：固定依赖规则、`models.generated.ts` 不可手改（改生成器）、erasable TS 语法限制、AGENTS.md 关键规则摘要

**现有英文文档**: `packages/coding-agent/docs/` 下 `sdk.md`、`custom-provider.md`、`extensions.md` 等 + 根 `README.md` 的 Development 节。

**依赖**: T06–T08、T03 完成后写（保证文件引用准确）。

---

## T19 · 改造指引 → `docs/recipes.md`

**目标**: 常见二次开发需求的「改哪里、怎么验证」速查清单。每条：需求 → 涉及包/文件 → 步骤 → 验证方式。**依赖 T18 的扩展点地图。**

**任务清单（每条一小节）**:

| 需求 | 落点提示 |
|------|----------|
| 新增内置工具 | `coding-agent/src/core/tools/` + `AgentTool` 规约（T07/T04） |
| 新增 slash command / 快捷键 / flag | `core/slash-commands.ts`、`DEFAULT_*_KEYBINDINGS`、`core/extensions/types.ts` 注册面 |
| 编写扩展（工具/命令/渲染/事件） | T08 loader/runner/API；参考 `examples/` |
| 自定义 provider / 新 provider 进 pi-ai | `packages/ai/src/providers/` 规约 + `api/` 适配（T03） |
| 新增 virtual model | `coding-agent/src/core/virtual-models.ts` |
| 修改 system prompt / skills / templates | `core/system-prompt.ts`、`core/resource-loader.ts` |
| 主题定制 | `modes/interactive/theme/theme.ts` |
| 改会话存储/格式 | `core/session-manager.ts` + `migrations.ts`（注意兼容策略） |
| 新增 RPC 命令 | `modes/rpc/rpc-types.ts` + `rpc-mode.ts` |
| SDK 嵌入应用 | `core/sdk.ts`、`src/index.ts`（T08） |
| TUI 组件/渲染改动 | `packages/tui/`（T05） |
| 接入 MCP server | `extensions/mcp/`（T09） |
| durable 自定义 task/doc（实验） | `harness/define.ts`、`registry.ts`（T12） |
| env 新增 op（实验） | protocol.md → daemon → client 四处联动（T13） |
| 改 compaction 策略 | `core/compaction/`（T07） |
| 模型目录数据源调整 | `packages/ai/scripts/generate-models.ts`（T03） |

**依赖**: T18 完成后写。
