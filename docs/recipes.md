# 改造指引（Recipes）

常见二次开发需求的「**改哪里 → 步骤 → 怎么验证**」速查。前置：[development.md](development.md) 的扩展点地图（本篇是其操作细化版）；每条只列最小改动集，深入语义跳对应 T0x 文档。

[返回索引](index.md) · 所属任务：T19（见 [roadmap](../roadmap/tasks.md)）

### 通用验证循环

任何改造都走这条链（细节见 [development.md](development.md) 第一、二节）：

1. `npm run check` —— 格式化/依赖边界/类型全绿（改代码后必须）；
2. 有针对性的测试：`./test.sh` 或单文件 vitest（[T18](development.md) 第一节）；
3. 手动验证：`./pi-test.sh`（源码版 pi）；TUI 类改动走 tmux 交互测试（`.pi/skills/interactive-testing.md`）；
4. 行为改动找 `examples/` 里最近似的样例改跑一遍；协议/契约类跑对应 conformance 套件。

---

## A. 产品层（coding-agent）

### 1. 新增一个内置工具

- **涉及**：`packages/coding-agent/src/core/tools/<name>.ts`（实现）+ `tools/index.ts`（工厂与 `allToolNames`）、`tools/renderers/`（渲染）、默认激活清单（`AgentSession` 构造的 `initialActiveToolNames` 注释默认 `[read, bash, edit, write]`）。
- **步骤**：
  1. 抄最近似的工具（如 `read.ts`/`grep.ts`）建文件；实现 `ToolDefinition`（name/description/parameters[typebox]/execute；大输出走 `truncate.ts` 的截断策略、>限额落盘）；
  2. 在 `tools/index.ts` 导出工厂并登记；写工具声明进系统提示靠 `promptSnippet`/`promptGuidelines`（[T07](packages/coding-agent-runtime.md) 流程 6）；
  3. 需要与其它工具一致的执行语义（校验/钩子）就什么额外都不用做——`AgentTool` 管线自动包住（[T04](packages/agent.md) 流程 4）；
  4. 若工具**默认关闭**（像 grep/find/ls），别加进默认清单，让用户经 `--tools` 开。
- **验证**：给工具写 `test/` 单测（同目录工具的既有测试为模板）；`./pi-test.sh --tools <name> "使用该工具完成 X"` 端到端；`npm run check` 的 entry-graphs 确认没引入新依赖边。

### 2. 新增 slash command / 快捷键 / CLI flag

- **涉及**：命令 `core/slash-commands.ts`（内置）/ 扩展注册面 `core/extensions/types.ts:1652`；快捷键 `DEFAULT_EDITOR_KEYBINDINGS`/`DEFAULT_APP_KEYBINDINGS`（**禁止在代码里硬编码按键**，AGENTS.md）；flag `core/extensions/types.ts:1664`（`parseArgs` 的 `unknownFlags` 通道）。
- **步骤**：优先做成**扩展**（`pi.registerCommand`/`registerShortcut`/`registerFlag`，[T08](packages/coding-agent-extensions.md) 第五节）；只有核心功能才进 `slash-commands.ts`/键位表。flag 的消费点在启动装配（`main.ts:754` 透传 `extensionFlagValues`）。
- **验证**：`./pi-test.sh` 里 `/命令`、按键、`--flag` 各试一次；`args.test.ts`/键位相关既有测试对照补用例；`--help` 确认扩展 flag 出现在 "Extension CLI Flags" 段。

### 3. 编写一个扩展（工具/命令/渲染/事件）

- **涉及**：API 面 `core/extensions/types.ts`（`ExtensionAPI:1564`）；加载规则 `core/extensions/loader.ts:828`；样板 `examples/extensions/`（约 80 个）。
- **步骤**：
  1. 在 `.pi/extensions/`（项目）或 `~/.pi/agent/extensions/`（全局）建 `foo.ts`，默认导出 `(pi) => {...}`；
  2. 按需注册：`registerTool`（参数必须是 object schema）/`registerCommand`/`on(事件)`/渲染器（[T08](packages/coding-agent-extensions.md) 事件表）；
  3. 注意加载期限制：`setup` 阶段只能注册；动作方法（`sendMessage` 等）在 `bindCore` 前会抛错（[T08](packages/coding-agent-extensions.md) 六.1）；
  4. 要分发就打成 pi 包（`package.json` 的 `pi` 字段，见 recipe 16）。
- **验证**：`./pi-test.sh` 加载（启动清单里应出现）；`-ne` 确认没有它时行为回退；用 `test/suite/` 的 harness + faux provider 写行为测试（**禁真实 API**）。

### 4. 修改 system prompt / skills / templates

- **涉及**：`core/system-prompt.ts`（sections 机制：`buildSystemPromptSections:128`、差量 `diffSystemPromptSections:217`）、`core/resource-loader.ts`（发现与优先级）、`core/skills.ts`、`core/prompt-templates.ts`。
- **步骤**：
  1. **内容改动**：直接写 `.pi/skills/*.md`、`.pi/prompts/*.md` 或项目 `AGENTS.md`——不需要碰代码；
  2. **结构改动**（新 section）：在 `system-prompt.ts` 加段落（保持「命名 + 可独立替换」的性质）；提示更新会自动以 `SystemMessage.sections` 差量进转录（[T07](packages/coding-agent-runtime.md) 流程 3；pi-ai 的重放语义 [T03](packages/ai.md)）；
  3. 扩展要动态改提示走 `before_agent_start`（生产路径，`examples/extensions/claude-rules.ts`）。
- **验证**：`./pi-test.sh` 里 `/context`（或 TUI 的展开视图）看提示渲染；evals 的 docs 对比可量化改动效果（[T16](packages/evals.md)）。

### 5. 主题定制

- **涉及**：`modes/interactive/theme/theme.ts`（1160 行）；主题加载与校验 `theme-json.ts`；`pi config` 管理。
- **步骤**：写主题 JSON（照着内置主题）放全局/项目主题目录 → `--use-theme` 或设置里选；代码级定制改 `theme/`（颜色 token 有校验器，`setThemeJsonValidator`，`main.ts:904`）。
- **验证**：`./pi-test.sh --use-theme <name>`；系统主题在终端色彩上报前后的两种状态都看一眼（[T05](packages/tui.md) 流程 8 的颜色查询）。

### 6. 改会话存储 / 会话格式

- **涉及**：`core/session-manager.ts`（JSONL 树，[T07](packages/coding-agent-runtime.md) 流程 4）；迁移 `migrations.ts:305`；消费方：会话选择器、`--export`、fork、compaction。
- **步骤**：
  1. 新字段优先加在**条目可选属性**上（向后兼容，无需迁移）；
  2. 破坏性变化：bump `CURRENT_SESSION_VERSION`(:41) + 写迁移（参考 v1→v2 的 `migrateV1ToV2:287`）——迁移在 `migrations.ts` 的 `runMigrations` 链里静默执行（[T06](packages/coding-agent-startup.md) 流程 8）；
  3. 动 `buildSessionContext`/`buildContextEntries` 前先读 T07 的投影语义——压缩感知裁剪是核心不变量。
- **验证**：既有会话文件加载（旧版本兼容）、`--export`、`--resume` 选择器、fork 各跑一遍；`session-manager` 相关单测全绿。

### 7. 新增 RPC 命令

- **涉及**：`modes/rpc/rpc-types.ts`（`RpcCommand`/`RpcResponse` 联合）+ `modes/rpc/rpc-mode.ts`（`handleCommand` switch）+ 必要时 `rpc-client.ts` 加方法。
- **步骤**：联合类型加 command/response（带 `id`）→ `handleCommand` 实现 →（可选）client 封装 → 文档 `docs/rpc-commands.md` 同步。
- **验证**：`examples/rpc-client.ts` 改跑新命令；注意 `prompt` 类命令的**异步双段应答**语义（[T06](packages/coding-agent-startup.md) 六.4）。

### 8. SDK 嵌入到应用

- **涉及**：入口 `core/sdk.ts:191`（`createAgentSession`）；公开面 `src/index.ts`（494 行的完整契约）；示例 `examples/sdk/01-minimal` 起。
- **步骤**：`createAgentSession`（或 services/runtime 三件套做多会话）→ `session.subscribe` 收事件 → `prompt`/`steer`/`abort` 驱动；自定义工具传 `customTools`；内联扩展传 `extensionFactories`。
- **验证**：先跑 `examples/sdk/01-minimal` 确认环境；再按需往 02…14 逐步对齐；类型上只用 `src/index.ts` 导出的面（不深引内部路径）。

### 9. 改 compaction 策略

- **涉及**：`core/compaction/`（切点 `findCutPoint:446`、估算 `estimateTokens:298`、主流程 `compact():965`、分支摘要 `branch-summarization.ts`）；触发点在 `agent-session.ts` 的 `_checkCompaction:2947`/`_runAutoCompaction:3097`。
- **步骤**：先确认需求属于哪层——阈值/保留量调 **settings**（`compaction` 设置，默认 keepRecentTokens 20000）；拦截或替换摘要用扩展 `session_before_compact`（可给自定义摘要，[T07](packages/coding-agent-runtime.md) 流程 5）；改算法才动源码（保守：先加测试再动切点）。
- **验证**：`agent-session-compaction.test.ts` 系列；`examples/extensions/custom-compaction.ts`；手动长时间会话观察 `/compact` 与自动触发。

### 10. 新增 virtual model

- **涉及**：`core/virtual-models.ts`（定义与 `model_change` 记录）；注册面 `pi.registerVirtualModel`（[T08](packages/coding-agent-extensions.md)）。
- **步骤**：写路由函数（选物理模型 + thinking）→ 注册 → 用 `virtual-models.md` 的语义核对「选择记录在会话里的是虚拟名、响应里是物理名」（[T07](packages/coding-agent-runtime.md) 六.9）。
- **验证**：`./pi-test.sh` 选择该模型跑一轮，检查 `model_change` 条目与恢复行为；相关测试对照 `docs/virtual-models.md` 示例。

---

## B. 模型层（pi-ai）

### 11. 自定义 provider（不改 pi-ai，应用侧）

- **涉及**：`pi.registerProvider(name, config)`（`core/extensions/types.ts:1830`；config 支持 baseUrl/环境变量插值/apiKey/`streamSimple`/OAuth/模型清单）。
- **步骤**：优先级：① OpenAI 兼容端点直接配 `baseUrl`+`apiKey`；② 特殊协议给自定义 `streamSimple`；③ 要 `/login` 就补 `oauth` 三项（login/refreshToken/getApiKey）（[T08](packages/coding-agent-extensions.md) 示例块）。
- **验证**：`./pi-test.sh --provider <name> --model <id> "hi"`；模型选择器里出现；`docs/custom-provider.md` 与 `examples/extensions/custom-provider-*` 对照。
- 另：`models.json` 也可声明自定义 provider（`docs/models.md`）。

### 12. 把新 provider 做进 pi-ai（内置）

- **涉及**：`packages/ai/src/providers/<id>.ts`（工厂：auth + models + `api`）+ `providers/all.ts:136` 注册 +（动态目录）`fetchModels`/`refreshModels` + 生成器的数据源接入。
- **步骤**：抄 `providers/anthropic.ts:74` 的骨架 → 选中已有 wire API（[T03](packages/ai.md) 五：40+ provider 只复用 15 个 API）或新写 `api/<name>.ts`（实现 `ProviderStreams`，`types.ts:292`）→ `all.ts` 加一行 → `package.json` 依赖如新增。
- **验证**：`packages/ai` 单测；`npm run check` 的 pinned-deps；`./pi-test.sh --list-models` 看目录；真实密钥或 faux provider 冒烟。

### 13. 新增 wire API（新协议适配）

- **涉及**：`packages/ai/src/api/<name>.ts`（`stream`/`streamSimple`）+ `<name>.lazy.ts` 惰性包装 + `KnownApi`/`ApiOptionsMap`（`types.ts:17/:263`）+ 对应 `*Compat` 接口。
- **步骤**：以 `anthropic-messages.ts:571` 为模板（`resolveTranscript` → `createClient` → `buildParams` → SSE 循环 → push 事件）；遵守事件协议（start → *_delta → done/error，失败**编码进流**不抛，`types.ts:373-377`）；注册进 `compat.ts` 的 `BUILTIN_APIS`（可选）与 provider 工厂。
- **验证**：复用 `packages/ai/test/` 的流事件断言模式；`faux provider` 做协议单测；契约要点见 [T03](packages/ai.md) 流程 1 与六.3。

### 14. 模型目录数据源调整（价格/新模型/思考映射）

- **涉及**：`packages/ai/scripts/generate-models.ts`（3684 行；数据源 models.dev/OpenRouter/AI Gateway/Radius + 静态修正表）。
- **步骤**：**改生成器，永不手改产物**（`models.generated.ts`/`*.models.ts`/`data/*.json`）→ `npm run generate-models`（或包内 `npm run hydrate-model-data` 仅数据）→ 需要新数据源时按既有 loader 模式加。
- **验证**：`npm run check:model-data`（build 会跑）；`--list-models` 看结果；生成 diff 即产物 diff（可随提交带入，AGENTS.md 允许）。

---

## C. 集成与周边

### 15. 接入 / 定制 MCP

- **涉及**：`mcp.json`（项目/全局）或 `pi.registerMcpServer`；集成层 `extensions/mcp/`；默认 `exposure: "codemode"`（[T09](packages/mcp.md) 流程 7）。
- **步骤**：先纯配置接入（url 或 command+args）→ 想直接暴露给模型就调 `exposure: "direct"`（含 resources 三工具的 per-server exposure）→ OAuth 服务器用 `pi mcp login`（token 存 `mcp-auth.json`，needs-auth 会自动重连）。
- **验证**：`pi mcp` 子命令检查连接；`./pi-test.sh` 里让模型用 `mcp__server__tool` 或经 codemode 调用；手工杀服务器进程看 needs-auth/disconnected 流转。

### 16. 把扩展打包成 pi 包分发

- **涉及**：`package.json` 的 `pi` 字段（`{extensions, skills, prompts, themes}`，支持 glob）+ `pi install` 流程（`package-manager-cli.ts`）。
- **步骤**：建包（`examples/plugins/pi-example-plugin/` 为样例）→ 声明 `pi` 字段（peerDependencies 里放 `@earendil-works/chord` 或写作依赖按需；chord 类包会被宿主 external 化）→ 本地 `pi install <path|npm:<pkg>|git:…>` 验证 → 发布。
- **验证**：`pi list` 出现；`pi config` 里可启停；`-ne`/`-builtin:*` 开关生效（[T08](packages/coding-agent-extensions.md)）。

### 17. TUI 组件 / 渲染改动

- **涉及**：`packages/tui/src/`（组件、`[LAYOUT_NODE]`、两个 renderer）；coding-agent 侧组件在 `modes/interactive/components/`。
- **步骤**：写组件遵守两条硬约束——`render(width)` **每行不超宽**、`invalidate()` 清缓存（[T05](packages/tui.md) 六.2）；要布局参与就实现 `[LAYOUT_NODE]`（stack/scroll）；键位走 `getKeybindings()`。
- **验证**：`packages/tui/test/` 的 `VirtualTerminal` 快照测试；改动渲染算法时用 `PI_TUI_DEBUG_REDRAW=1`/`PI_TUI_DEBUG=1`；tmux 里目测（`.pi/skills/interactive-testing.md`）。

### 18. 工具输出/渲染定制（产品侧）

- **涉及**：`core/tools/renderers/`（`createAllToolRenderers`）+ 扩展 `registerToolRenderer`（[T08](packages/coding-agent-extensions.md)）。
- **步骤**：只改显示就写扩展渲染器（`examples/extensions/built-in-tool-renderer.ts`）；改截断/落盘行为才碰 `truncate.ts`/`output-accumulator.ts`（[T07](packages/coding-agent-runtime.md) 流程 6）。
- **验证**：`./pi-test.sh` 里观察工具卡片展开/折叠；大输出用例核对落盘路径提示。

---

## D. 实验轨（PI_EXPERIMENTAL=1）

### 19. durable：自定义 task / document

- **涉及**：`packages/durable/src/harness/define.ts`（`defineTool/section/hook` 糖）+ `types.ts` 的 `defineTask:233`/`defineDoc:39` + `registry.ts:112`。示例 `test/examples/12-tasks.ts`、`01-documents.ts`。
- **步骤**：
  1. `defineTask`：写 `initial`/`phases`（**每个 phase 必须 commit 变化或终态**，否则 fault，[T12](packages/durable.md) 流程 3）/`abort`/可选 `migrate`；
  2. `defineDoc`：声明 scope/history/fork/version，`checkpointWhen` 控 delta 链长；
  3. `createRegistry()` 注册（重名抛错）；子任务用 `ownership: {kind:"task"}`。
- **验证**：抄 `test/examples/00–31` 对应样例；崩溃恢复类行为用 13/31 的"中途 kill 再 open"模式验证（[T12](packages/durable.md) 流程 2）。

### 20. env：新增一个 op（四处联动）

- **涉及**：`packages/env/docs/protocol.md`（规范）→ `daemon/src/main.rs`（分发 + 实现）→ `src/connection.ts`（客户端调用）→ `src/remote-env.ts`（语义方法）+ `docs/semantics.md`（分工表）。
- **步骤**：协议先行（帧/字段/错误码）→ daemon 实现（文件类进线程池、进程类独立线程）→ 客户端 mock 起来 → 方法实现**对齐 NodeExecutionEnv 语义**（路径/错误映射/abort 检查点在客户端）。
- **验证**：`packages/durable/src/testing/env-conformance.ts` 套件必须过；`packages/env/test/differential.test.ts` 的随机对拍（60 种子 × 25 操作）扩到新方法。

### 21. chord：写一个 facet 插件（实验）

- **涉及**：`packages/chord/src/api.ts`（`defineFacet:69`/`defineService:73`）+ `facets/host.ts` 的生命周期规则；示例 `test/facets.test.ts`。
- **步骤**：`setup(env)` 只做同步声明（provide/use/observe/own/onActivate）；跨进程拆分别 facet；远程服务经 `RemoteServiceSource` 接入。
- **验证**：`facets.test.ts` 模式（含 shape-preserving reload 用例）；`boundary.test.ts` 确认没引入 Pi 依赖。

### 22. 远程 C/S：协议变更（实验）

- **涉及**：`packages/protocol/src/protocol.ts`（消息联合）+ 两端 codec 校验 + `PROTOCOL_VERSION:5` 递增（**严格相等匹配**，[T14](packages/remote-sessions.md) 六.3）。
- **步骤**：改 schema（strict object，记得两端同时改）→ 版本递增 → 更新 `codec`/测试；服务端行为改动注意两条纪律：响应先于更新（订阅）、响应后失败即断连。
- **验证**：`packages/server/src/testing/` 的内存 host/client 对端；Unix 发现 + 握手探测跑一遍。

---

## 附：验证方式速查

| 想验证什么 | 用什么 |
|-----------|--------|
| 类型/依赖边界/格式 | `npm run check` |
| 非 e2e 测试全量 | `./test.sh` |
| 单文件测试 | vitest（或 tui 的 `node --test`） |
| 源码版 pi 手动跑 | `./pi-test.sh`（TUI 走 tmux skill） |
| agent 行为断言（假模型） | `test/suite/harness.ts` + faux provider |
| 契约一致性 | 各包 `src/testing/` 的 conformance 套件 |
| 文档效果量化 | evals 的 docs 对比（[T16](packages/evals.md)） |
| 渲染问题 | `PI_TUI_WRITE_LOG` / `PI_TUI_DEBUG_REDRAW` / `PI_TUI_DEBUG` |
| 启动性能 | `PI_STARTUP_BENCHMARK=1`、`npm run profile:tui` |
| 崩溃现场 | bug-report（`recordCrash`/`/bug`、`summarizeForBugReport`） |

## 相关文档

- [development.md](development.md)（环境/调试/扩展点地图/打包）· [data-flows.md](data-flows.md)（改动处于哪条链路）· [architecture.md](architecture.md)（决策背景）
- 各 recipe 对应的 T0x 包文档：[索引](index.md)

---

[返回索引](index.md) · 任务与进度：[roadmap](../roadmap/README.md)
