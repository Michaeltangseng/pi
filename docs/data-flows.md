# 关键数据流时序图

汇总篇：把前面各包文档里的流程串成**端到端链路**。每条流给出触发点、参与文件（`file:line`）与**数据形态的每一次转换**。所有引用都来自本套文档各篇（均已按源码核实）。

[返回索引](index.md) · 术语见 [glossary](glossary.md) · 所属任务：T17（见 [roadmap](../roadmap/tasks.md)）

阅读建议：先看每张图的「形态」列（数据在每一跳变成了什么），再顺着参与者看代码位置。

## 一、启动链路：shell → 交互模式就绪

**触发**：用户在终端敲下 `pi`。

```mermaid
sequenceDiagram
    autonumber
    participant SH as shell
    participant CLI as cli.ts / setup.ts
    participant MAIN as main.ts
    participant BOOT as 装配（sdk.ts / services）
    participant TUI as InteractiveMode（pi-tui）

    SH->>CLI: argv
    CLI->>CLI: setupCli()：进程标题 / PI_CODING_AGENT 标记 / undici（cli/setup.ts:4-13）
    CLI->>MAIN: main(argv)（main.ts:573）
    MAIN->>MAIN: 子命令分流 install/config/mcp/auth（main.ts:582-618）
    MAIN->>MAIN: parseArgs（cli/args.ts:72）→ resolveAppMode（main.ts:112）
    MAIN->>MAIN: runMigrations（main.ts:666 → migrations.ts:305）
    MAIN->>MAIN: createSessionManager（main.ts:358：--session/--resume/--fork/--continue）
    MAIN->>BOOT: createRuntime 工厂（main.ts:730）：services → 会话创建
    BOOT->>BOOT: SDK 装配（core/sdk.ts:191-461）：<br/>ModelRuntime.create(:198) → reload 扩展(:205) → 恢复模型(:219) →<br/>初始模型(:236) → new Agent(:411) → new AgentSession(:461)
    MAIN->>TUI: new InteractiveMode(runtime).run()（main.ts:951-983）
    TUI->>TUI: init() 挂载组件树（interactive-mode.ts:943）→ mountInteractiveTui(:873)
    TUI->>TUI: subscribeToAgent() 订阅会话事件（:3400）
    Note over TUI: 后台并行：模型目录刷新(:1152)、版本检查(:1159)、<br/>包更新检查(:1166)、tmux 键盘检查(:1181)
```

关键形态转换：`argv: string[]` → `Args`（`cli/args.ts:13`）→ `AppMode`（`main.ts:112`）→ `CreateAgentSessionOptions`（`core/sdk.ts`）→ `AgentSession`（持 `Agent`）→ TUI 组件树就绪，进入事件驱动。

## 二、一次用户消息的全链路（生产轨）

这是 pi 的主动脉。**触发**：用户在编辑器里输入并回车。

```mermaid
sequenceDiagram
    autonumber
    participant TUI as InteractiveMode（pi-tui）
    participant AS as AgentSession
    participant AG as Agent（agent-core loop）
    participant MR as ModelRuntime / pi-ai
    participant WIRE as wire API（如 anthropic-messages）
    participant PROV as 模型服务
    participant SM as SessionManager（JSONL）

    TUI->>AS: 编辑器提交 → prompt(text)（interactive-mode.ts:3391 → agent-session.ts:1968）
    Note over AS: 扩展命令(:1977) → input 钩子(:1993) → skill/模板展开(:2007)
    AS->>AS: 校验模型与认证(:2032-2050) → 预检压缩(:2054)
    AS->>AS: before_agent_start 钩子(:2062) → 图片归一化(:2075)
    AS->>AG: 组装 AgentMessage[] 后 _runAgentPrompt(:2110)
    Note over AG: Agent.prompt(agent.ts:373) → runWithLifecycle(:507) → runLoop(agent-loop.ts:163)
    AG->>AG: declareToolChanges(:211) → prepareRequest(:219)
    AG->>AG: streamAssistantResponse(:381)：<br/>transformContext(:390) → convertToLlm(:395, core/messages.ts:148)
    AG->>MR: normalizeContext(pi-ai transcript.ts:30) → streamFn(:403, sdk.ts:420)
    MR->>MR: Models.stream(models.ts:877) → lazyStream(api/lazy.ts:46) → applyAuth(:843)
    MR->>WIRE: dispatch 按 model.api(models.ts:1072) → stream(anthropic-messages.ts:571)
    WIRE->>PROV: buildParams(:1124) → HTTP SSE（retryProviderRequest :649）
    PROV-->>WIRE: 原始 SSE 事件
    loop 流式
        WIRE-->>AG: AssistantMessageEvent（EventStream，event-stream.ts:97）
        AG-->>AS: AgentEvent：message_start / message_update / message_end（agent-loop.ts:414-468）
        AS--)SM: message_end 时 appendMessage → JSONL 落盘（agent-session.ts:1144-1165；session-manager.ts:1204/1172）
        AS-->>TUI: AgentSessionEvent（_handleAgentEvent 转发，agent-session.ts:1139-1141）
        TUI->>TUI: handleEvent 更新组件（interactive-mode.ts:3406）→ requestRender
        TUI->>TUI: TuiMainScreen.doRender 差分重绘（tui-main-screen.ts:247）
    end
    alt 有工具调用
        AG->>AG: executeToolCalls(agent-loop.ts:508)：<br/>prepare(:708：校验+beforeToolCall) → execute(:821) → finalize(:859：afterToolCall)
        Note over AG: 工具实现例：写文件经 file-mutation-queue 串行（file-mutation-queue.ts:32）
        AG-->>AS: tool_execution_start/update/end + tool-result 消息
        AG->>AG: 回到 runLoop 下一轮（工具结果进转录）
    else 模型停止
        AG->>AG: turn_end → finishTurn 决策（agent-loop.ts:286）
    end
    AG-->>AS: agent_end（agent-loop.ts:320）
    AS->>AS: _handlePostAgentRun(agent-session.ts:1853)：<br/>可重试错误→自动重试(:1864)；否则 _checkCompaction(:1884)
    AS-->>TUI: agent_settled（:1078）→ waitForIdle 解析
```

**形态转换链**（一条消息的每一跳）：

| 形态 | 产生点 | 说明 |
|------|--------|------|
| `string` / 图片块 | 编辑器提交 | 用户可见的输入 |
| `AgentMessage`（user，含自定义消息类型） | `AgentSession.prompt` 组装（`agent-session.ts:2079-2104`） | 可含 `bashExecution`/`custom` 等扩展消息 |
| `Message[]` | `convertToLlm`（`core/messages.ts:148`） | 扩展消息转 user；`!!` 与通知被过滤 |
| `TranscriptContext` | `normalizeContext`（pi-ai `utils/transcript.ts:30`） | system prompt/工具折叠进首条 system 消息（branded 类型） |
| provider 请求体 | wire API `buildParams`（`anthropic-messages.ts:1124`） | 厂商协议形态（消息/工具/思考参数） |
| `AssistantMessageEvent` | provider SSE → wire 适配 | `start → *_delta → *_end → done` |
| `AssistantMessage` | `EventStream.result()` | 终值消息（usage/stopReason） |
| `AgentEvent` | `streamAssistantResponse` 桥接 | `message_*` / `tool_execution_*` / `turn_*` / `agent_*` |
| `AgentSessionEvent` | `_handleAgentEvent` 转发 | 加 `queue_update`/`entry_appended`/`compaction_*`/`auto_retry_*` |
| `SessionEntry`（JSONL 一行） | `message_end` 时持久化 | `message` 条目（含 `id`/`parentId` 树结构） |
| 终端字节 | pi-tui 差分渲染 | 只写变化行（CSI 2026 帧包裹） |

## 三、压缩（compaction）：阈值与 overflow 两条触发

**触发 A（阈值）**：一轮响应完成后的 post-run 检查。
**触发 B（overflow）**：provider 报上下文溢出错误 / 可恢复的 length 截断。

```mermaid
sequenceDiagram
    autonumber
    participant AG as agent loop
    participant AS as AgentSession
    participant CP as compaction（prepare + 摘要 LLM）
    participant SM as SessionManager
    participant AG2 as agent.continue（重试）

    Note over AS: 共同入口：_checkCompaction(agent-session.ts:2947)
    alt 触发 A：阈值（case 3，:3049）
        AG-->>AS: 响应完成（usage 或估算越过阈值）
        AS->>CP: _runAutoCompaction("threshold")(:3097)
    else 触发 B：overflow（case 1/2，:3010）
        AG-->>AS: stopReason=error（isContextOverflow）或 length 截断
        AS->>AS: _omitRecoveryAttempt：追加 context_edit(null) 撤下失败响应(:3043)
        AS->>CP: _runAutoCompaction("overflow", willRetry)(:3044)
        Note over AS: 一次性闸门 _overflowRecoveryAttempt(:3019-3038)
    end
    CP->>CP: prepareCompaction(compaction/compaction.ts:872) → findCutPoint(:446，4096 op/keepRecentTokens)
    CP->>PROV: compact()(:965)：LLM 生成摘要（含轮中拆分的前缀摘要）
    CP->>SM: appendCompaction(session-manager.ts:1261)：<br/>summary + firstKeptEntryId + tokensBefore + systemMessage 快照(:1270-1283)
    AS-->>AS: compaction_end 事件（:2877 / 失败路径 :2890）
    alt willRetry（overflow 且响应未完成）
        AS->>AG2: _failedResponse 作路由载荷 → agent.continue(:1841)
        AG2->>AS: 重跑本轮（上下文已是压缩后的）
    end
    Note over SM: 之后每次 buildSessionContext(session-manager.ts:576)：<br/>buildContextEntries(:476) 从 firstKeptEntryId 起算 + 摘要消息（messages.ts:11-24）
```

**手动触发**：`/compact` → `session.compact`（`agent-session.ts:2764`）：先 `abort()`（`:2765`）→ `session_before_compact` 钩子（可取消/自定义摘要，`:2792-2812`）→ `appendCompaction`（`:2847`）→ `compaction_end`（`:2877`）；手动路径**永不自动重试**。

关键语义：压缩单元是**一次原子提交**（一个 compaction 条目 + 边界系统提示快照）；重放时模型看到的上下文 = 摘要 + `firstKeptEntryId` 之后的保留段。

## 四、扩展生命周期：从磁盘到每个钩子

**触发**：启动装配、`/reload`、或会话替换后的重绑定。

```mermaid
sequenceDiagram
    autonumber
    participant RL as ResourceLoader
    participant LD as extensions/loader.ts
    participant JI as jiti
    participant RU as ExtensionRunner
    participant FLOW as 上方各链路的钩子位点

    RL->>RL: reload(resource-loader.ts:509)：信任探测 → packageManager.resolve(:526)
    RL->>LD: discoverAndLoadExtensions(loader.ts:828)：<br/>项目 .pi/extensions → 全局 agentDir/extensions → 配置路径(:849-873)
    LD->>LD: discoverExtensionsInDir(:791)：直接文件 / index.ts / pi.extensions 清单(:749)
    LD->>JI: loadExtensionModule(:559)：jiti 直接 import TS（虚拟模块/tsconfig paths/别名表）
    LD->>LD: initializeExtension(:613)：factory(api) 注册 → commit(:532)
    Note over LD: 注册项写入 Extension 对象：tools/commands/flags/shortcuts/renderers/handlers
    AS->>RU: AgentSession.bindExtensions(agent-session.ts:3258) → runner
    RU->>RU: bindCore(runner.ts:410)：装真实动作 → 冲洗 pending provider/虚拟模型注册(:463-514)
    Note over RU: 此后注册立即生效(:516-543)
    FLOW->>RU: 运行期事件分发（快照 handler 列表，按加载序，单个失败隔离上报）
```

**钩子在主动脉上的位点**（回看第二节的编号）：

| 位点 | 事件 | 分发实现 |
|------|------|---------|
| `prompt` 入口（`agent-session.ts:1977/1993`） | 扩展命令、`input` | `runner` 命令表 / `emitInput` |
| 请求前（`:2062`） | `before_agent_start`（可改提示/工具/模型） | `emitBeforeAgentStart`（`runner.ts:1420`） |
| 上下文（`sdk.ts:436-440` 的 `transformContext`） | `context` / `context_with_system` | `emitContext`（`runner.ts:1298`，两阶段） |
| provider 请求（`sdk.ts:383-395`） | `before_provider_request` / `before_provider_headers` / `after_provider_response` | `emitBeforeProviderRequest`（`:1361`）等 |
| 工具调用（`agent-session.ts:652-655`） | `tool_call`（可 block）/ `tool_result`（可改结果） | `emitToolCall`（`:1242`，短路）/ `emitToolResult`（`:1183`） |
| 消息与轮（`:1103` 事件泵） | `message_*` / `turn_*` / `agent_*` / `agent_before_settle` | 各 `emit*` |
| `!` 命令与 RPC bash（`rpc-mode.ts:561`） | `user_bash`（可接管执行） | `emitUserBash`（`:1262`，异常 rethrow） |

## 五、实验轨对照：durable 的一次问答

**触发**：`Conversation.submit()`（[durable](packages/durable.md)）。与第二节逐段对照阅读——同样的语义（提示、流式、工具、压缩），不同的持久化纪律（**每一步先提交**）。

```mermaid
sequenceDiagram
    autonumber
    participant CV as Conversation 门面
    participant SUB as Submissions
    participant SCH as TaskScheduler
    participant GEN as GenerationTask
    participant AI as pi-ai（streamFn）
    participant TOOL as ToolTask
    participant SES as Session（提交管线）
    participant WATCH as 观察面（view/events）

    CV->>SUB: submit(draft)（harness.ts:100 → submissions.ts:148）
    SUB->>SES: commit：pi.user entry + submission 记录（requestId 去重 :155-163）
    SUB->>SCH: resume
    SCH->>SCH: #reserve(scheduler.ts:771)：pending→running 先落盘(:793)
    SCH->>GEN: 调度 pi.generation（phase=prepare）
    GEN->>GEN: planSystemEntries(prompt.ts:66) 差量化系统提示 → 位置式 pi.system entry
    GEN->>SES: commit checkpoint（cutoff=最新 entry，generation.ts:166-170）
    GEN->>AI: request phase：流式调用
    loop 流式（节流提交）
        AI-->>GEN: 增量
        GEN->>SES: 每 ~100ms 一次 partial commit（generation.ts:364-416，同时仅一 commit 在飞）
    end
    GEN->>SES: 终值 → AssistantEntry 提交
    GEN->>TOOL: startToolRound(generation.ts:545)：一次 commit 建 tool 任务
    TOOL->>TOOL: call(tool.ts:55)：解析→校验→beforeTool(:67-77)
    TOOL->>SES: 单次 commit 写 intent {phase:"execute", replay}(tool.ts:85-90)
    TOOL->>TOOL: 执行(:94-111)；崩溃恢复时按 replay 判定重跑或 interrupted(:98-110)
    TOOL->>SES: tool-result entry 提交
    GEN->>GEN: finishToolRound(:597)：inbox 边界 → 下一轮或 yield
    GEN->>SES: terminal → 任务终态（completing→terminal，:1063-1065）
    SUB-->>CV: submission done（wait() 解析）
    SES-->>WATCH: CommitPublication（session.ts:443 publish）
    WATCH->>WATCH: CommittedStateSource/Watch(observation.ts) → view/events 派生（events.ts:195）
```

**崩溃恢复对照**（实验轨独有）：`Harness.open`（`harness.ts:411`）→ `openTasks`（`scheduler.ts:239`）：`running → pending` 保留 checkpoint 重放（`:252-253`）→ reconcile 补推 abort 标记、failFast、撤回排队输入（`:527-540`）→ 续跑。判据：phase 必须产生持久进展，否则 fault（`:963-966`）。

**两轨的关键差异一句话**：生产轨「执行 → 出错时用记忆中的状态继续」；实验轨「**提交意图 → 执行 → 用提交的记录对账**」。

## 六、（可选）MCP 工具经 codemode 沙箱执行

**触发**：模型调用 `codemode` 工具，脚本里 `await tools.mcp__server__search(...)`（MCP 默认 exposure 就是 codemode，见 [MCP](packages/mcp.md) 流程 7）。

```mermaid
sequenceDiagram
    autonumber
    participant M as 模型
    participant CO as codemode（extensions/codemode/execute.ts）
    participant VM as QuickJS VM（worker）
    participant HC as 宿主 handleCall
    participant AG as agent 工具管线（runToolCall）
    participant MCP as McpClient / MCP 服务器

    M->>CO: codemode({code})（execute.ts:416）
    CO->>CO: parseCodemodeSource(source.ts:100) → new CodemodeSandbox(host.ts:394)
    CO->>VM: worker 启动 + prelude 求值（worker.ts:52-116）
    VM->>HC: bridge("call", mcp__server__search, argsJson)（host.ts:265）
    HC->>AG: ctx.executeTool(execute.ts:462) → runToolCall(agent-loop.ts:811)：<br/>校验 + tool_call/tool_result 钩子与直接调用一致
    AG->>MCP: McpClient.callTool(client.ts:376) → transport → 服务器（含进度通知）
    MCP-->>AG: CallToolResult（toLlmContent 转换由集成层做）
    AG-->>HC: outcome
    HC-->>VM: result（JSON 字符串）→ 脚本 Promise 解析
    Note over VM: 嵌套结果不进 LLM 上下文——只回脚本
    VM-->>CO: done{ok, valueJson, writesJson} → finish：interrupt 标志 + terminate(host.ts:323-326)
    CO-->>M: 仅脚本输出/返回值（截断/落盘后）
```

## 七、一页速查：文件与职责对照

| 形态/关注点 | 生产者 | 消费者 |
|-------------|--------|--------|
| `AgentSessionEvent` | `agent-session.ts:1039`（`_emit`） | TUI（`:3406`）、RPC（`rpc-mode.ts:355`）、print/json（`print-mode.ts:108`） |
| `CommitPublication`（durable） | `session.ts:447`（`#publish`） | view/events/taskGraph（`view.ts`/`events.ts`/`task-graph.ts`） |
| `AssistantMessageEvent` | wire API | agent loop 桥接（`agent-loop.ts:414-468`） |
| JSONL 会话文件 | `session-manager.ts:1172`（`_persist`） | `--resume` 选择器、`--export`、fork |
| `codemode-store` entry | `execute.ts:502-507` | 下次脚本执行的 `readCodemodeStore`（`:233-243`） |
| 协议帧（CBOR） | `protocol` 编解码 | `client`/`server`（[远程会话](packages/remote-sessions.md)） |

## 八、相关文档

- 链条上的各篇（按出现顺序）：[启动与运行模式](packages/coding-agent-startup.md)、[coding-agent 核心运行时](packages/coding-agent-runtime.md)、[agent 运行时](packages/agent.md)、[pi-ai](packages/ai.md)、[pi-tui](packages/tui.md)、[扩展系统与 SDK](packages/coding-agent-extensions.md)、[MCP](packages/mcp.md)、[codemode](packages/codemode.md)、[durable](packages/durable.md)、[chord](packages/chord.md)、[远程会话](packages/remote-sessions.md)
- 下一步：[二次开发指南](development.md)（怎么构建/调试/扩展点地图）· [改造指引](recipes.md)（改哪里速查）

---

[返回索引](index.md) · 任务与进度：[roadmap](../roadmap/README.md)
