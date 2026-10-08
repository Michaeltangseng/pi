# 二次开发指南

操作手册：**把架构认知变成动手能力**。三部分——环境与工作流（构建/检查/调试）、**扩展点地图**（想做 X 该动哪里）、打包与测试设施。各机制的写法细节引用现有英文文档，不重复教程。

[返回索引](index.md) · 所属任务：T18（见 [roadmap](../roadmap/tasks.md)）· 前序阅读：[architecture](architecture.md)、[data-flows](data-flows.md)

## 一、开发环境与工作流

### 安装与构建

```bash
npm install --ignore-scripts   # 全仓依赖；安装器按供应链策略禁生命周期脚本
npm run build                  # 刷新模型数据（网络）后按依赖序构建全部包
npm run build:offline          # 模型数据已就绪时的离线重建
```

构建顺序即依赖序（根 `package.json` 的 `build` 脚本）：`chord → tui → telemetry → codemode → mcp → ai → durable → env → agent → protocol → client → server → coding-agent`。改了下层包要重建它；只改 coding-agent 时仍要求其依赖的 dist 存在（做过一次全量 build 即可）。

Node 要求 **≥ 22.19**；外部依赖全部**钉死精确版本**（`save-exact` + `min-release-age=2`），`package-lock.json` 是依赖唯一事实源，改依赖属于"需要审查的代码变更"。

### 质量门禁：`npm run check`

链式执行（`package.json`），改代码（非文档）后必须全绿：

| 步骤 | 检查什么 |
|------|---------|
| `biome check --write --error-on-warnings` | 格式化 + 静态检查（warning 即失败） |
| `check:pinned-deps` | 直接外部依赖必须是精确版本 |
| `check:runtime-deps` | 运行时依赖边界（谁可以 import 谁） |
| `check:ts-imports` | TS 相对导入规范（含 erasable 语法约束的导入形态） |
| `check:entry-graphs` | 入口图（如发布入口不得导入 experimental） |
| `check:install-lock:coding-agent` | 安装锁与根 lockfile 一致 |
| `tsc --noEmit` | 全量类型检查 |
| `check:browser-smoke` | 浏览器可加载冒烟 |

### 跑测试：`./test.sh`（不要直接跑 vitest 全量）

- 全量非 e2e：**仓库根 `./test.sh`**（自动跳过需要 API key 的 LLM 测试）。
- 单包/单文件：vitest 包直接 `node node_modules/vitest/dist/cli.js --run test/xxx.test.ts`；`packages/tui` 用 `node:test`（`node --test test/xxx.test.ts`）。
- **不要直接跑完整 vitest 套件**：存在环境变量触发即启用的 e2e 测试，会打真实端点。
- coding-agent 的 `test/suite/`：用 `test/suite/harness.ts` + **faux provider**——**禁止真实 provider API/key/付费 token**（见「注意事项」）。

### 从源码运行 pi

```bash
./pi-test.sh                 # 任意目录可跑；从源码启动 pi 本体
./pi-test.sh --help          # 走真实 CLI 参数路径
```

`pi-test.sh` 是调试一切 CLI/TUI 行为的首选——区别于 `npm run build` 后的产品二进制。

## 二、本地运行与调试

| 手段 | 用途 |
|------|------|
| `./pi-test.sh` | 源码版 pi；验证启动/参数/交互行为 |
| `PI_EXPERIMENTAL=1 ./pi-test.sh server …` / `client …` | 进入实验轨（远程 C/S、实验子命令；见 [远程会话](packages/remote-sessions.md)） |
| tmux 交互测试 | 受控终端里测 TUI：仓库内的操作指引见 [`.pi/skills/interactive-testing.md`](../.pi/skills/interactive-testing.md)（`tmux new-session -d -s pi-test -x 80 -y 24` → `send-keys` → `capture-pane`；发布冒烟用 release 二进制并等模型回复，仅启动不算通过） |
| `PI_TUI_WRITE_LOG=<file\|dir>` | 记录 pi-tui 的每一次终端写入（排查渲染问题；见 [pi-tui](packages/tui.md)） |
| `PI_TUI_DEBUG_REDRAW=1` / `PI_TUI_DEBUG=1` | 记录全量重绘原因 / 单帧渲染快照（`/tmp/tui/`） |
| `PI_OFFLINE=1` | 关闭启动期网络操作（版本检查/目录刷新） |
| `PI_STARTUP_BENCHMARK=1` | 跑完 interactive `init()` 即退出并打印 `printTimings()` 阶段耗时（启动性能） |
| 性能剖析 | `npm run profile:tui` / `npm run profile:rpc`（`scripts/profile-coding-agent-node.mjs`） |
| bug-report | 交互模式内置崩溃记录与 `/bug` 提示（`interactive-mode.ts` 的 `recordCrash`/`suggestBugReport`）；`session.summarizeForBugReport()` 可程序化生成摘要 |

排查渲染类问题的顺序建议：`PI_TUI_WRITE_LOG` 看输出 → `PI_TUI_DEBUG_REDRAW=1` 看是否异常全量重绘 → `PI_TUI_DEBUG=1` 对比单帧前后行。

## 三、扩展点地图（核心速查）

「想做什么 → 用哪个机制 → 动哪些文件 → 看哪篇文档」一站式表格。**优先用扩展机制，而不是改核心**（[architecture](architecture.md) 第八节：pi 有意保持核心最小）。

### 生产链路（coding-agent）

| 需求 | 机制 / 落点 | 现有英文文档 | 本套文档 |
|------|------------|-------------|---------|
| 新增一个工具 | 扩展 `pi.registerTool`（首选）或源码 `core/tools/<name>.ts` + `tools/index.ts` 注册 | [extensions.md](../packages/coding-agent/docs/extensions.md) | [T07](packages/coding-agent-runtime.md) 流程 6 |
| 新增斜杠命令 | `pi.registerCommand`；源码在 `core/slash-commands.ts` | [slash-commands.md](../packages/coding-agent/docs/slash-commands.md) | [T08](packages/coding-agent-extensions.md) |
| 快捷键 | `pi.registerShortcut`；默认值放 `DEFAULT_EDITOR_/APP_KEYBINDINGS`（禁止硬编码按键） | [keybindings.md](../packages/coding-agent/docs/keybindings.md) | [T05](packages/tui.md) 流程 5 |
| CLI flag | `pi.registerFlag` + `getFlag`（走 `unknownFlags` 通道） | extensions.md | [T06](packages/coding-agent-startup.md) |
| 拦截/改写行为（工具审批、输入、bash、上下文、provider 请求） | 事件钩子：`tool_call`/`input`/`user_bash`/`context`/`before_provider_request`…（约 40 个事件） | extensions.md | [T08](packages/coding-agent-extensions.md) 事件表 |
| 自定义模型服务 | `pi.registerProvider(name, config)`（支持自定义 `streamSimple`/OAuth） | [custom-provider.md](../packages/coding-agent/docs/custom-provider.md) | [T08](packages/coding-agent-extensions.md)、[T03](packages/ai.md) |
| pi-ai 内置新增 provider | `packages/ai/src/providers/<id>.ts` + `providers/all.ts` 注册 + 生成器数据源 | [models.md](../packages/coding-agent/docs/models.md) | [T03](packages/ai.md) 第七节 |
| 虚拟模型（命名路由） | `pi.registerVirtualModel` / `core/virtual-models.ts` | [virtual-models.md](../packages/coding-agent/docs/virtual-models.md) | [T07](packages/coding-agent-runtime.md) |
| 改系统提示 | 源码 `core/system-prompt.ts`（sections 机制）；扩展用 `before_agent_start` | — | [T07](packages/coding-agent-runtime.md) 流程 3 |
| Skill | `.pi/skills/*.md`（frontmatter：name/description/disable-model-invocation） | [skills.md](../packages/coding-agent/docs/skills.md) | [T08](packages/coding-agent-extensions.md) 流程 3 |
| Prompt 模板 | `.pi/prompts/*.md`（`$1`/`$@`/`${N:-default}` 替换） | [prompt-templates.md](../packages/coding-agent/docs/prompt-templates.md) | [T08](packages/coding-agent-extensions.md) |
| 主题 | 主题文件 + `pi config`；实现 `modes/interactive/theme/` | [themes.md](../packages/coding-agent/docs/themes.md) | [T08](packages/coding-agent-extensions.md) |
| 改会话存储/格式 | `core/session-manager.ts`（先读 [T07](packages/coding-agent-runtime.md) 流程 4 的树语义）+ bump 版本与迁移 | [session-format.md](../packages/coding-agent/docs/session-format.md) | T07 |
| 新增 RPC 命令 | `modes/rpc/rpc-types.ts` + `rpc-mode.ts`（两端同步） | [rpc-commands.md](../packages/coding-agent/docs/rpc-commands.md) | [T06](packages/coding-agent-startup.md) 流程 4 |
| SDK 嵌入应用 | `createAgentSession`（`core/sdk.ts:191`） | [sdk.md](../packages/coding-agent/docs/sdk.md) | [T08](packages/coding-agent-extensions.md) 流程 8、`examples/sdk/` |
| 接 MCP 服务器 | `mcp.json` / `pi.registerMcpServer`；注意默认 exposure=codemode | [mcp.md](../packages/coding-agent/docs/mcp.md) | [T09](packages/mcp.md) |
| 改压缩策略 | `core/compaction/`；拦截用 `session_before_compact` | [compaction.md](../packages/coding-agent/docs/compaction.md) | [T07](packages/coding-agent-runtime.md) 流程 5 |

### 下层与实验轨

| 需求 | 落点 | 本套文档 |
|------|------|---------|
| 改 agent 循环语义（钩子时序、工具批、steering） | `packages/agent/src/`（6 个文件，逐行可控） | [T04](packages/agent.md) |
| 改模型调用/认证/事件协议 | `packages/ai/src/`（models.ts 生命周期、auth/、api/ 适配） | [T03](packages/ai.md) |
| TUI 组件/渲染 | `packages/tui/src/`（Component/[LAYOUT_NODE]/renderer） | [T05](packages/tui.md) |
| MCP 客户端行为 | `packages/mcp/src/`（client/transports/oauth） | [T09](packages/mcp.md) |
| 沙箱能力（codemode） | `packages/codemode/src/`（globals/声明渲染/中断模型） | [T10](packages/codemode.md) |
| durable 自定义 task/doc（实验） | `defineTask`/`defineDoc` + registry；先读 `docs/spec.md` §1 不变式 | [T12](packages/durable.md) |
| chord 插件/facet（实验） | `defineFacet`/`defineService`/`replicatedState`（规格在 PLANNING.md） | [T11](packages/chord.md) |
| env 新增 op（实验，四处联动） | protocol.md → daemon → 客户端 → RemoteExecutionEnv | [T13](packages/env.md) |
| 远程 C/S 协议（实验） | `packages/protocol` → client/server 两端同步 + 版本递增 | [T14](packages/remote-sessions.md) |
| 遥测接入 | 实现 `TelemetryContext` + conformance 套件（现状零调用点） | [T15](packages/telemetry.md) |

## 四、示例与文档索引

- **`packages/coding-agent/examples/sdk/`**：14 个渐进示例（`01-minimal` … `14-codemode-mcp`）——SDK 嵌入从最简会话到自定义 provider/扩展/压缩钩子/codemode+MCP。`examples/sdk/README.md` 有逐例索引。
- **`packages/coding-agent/examples/extensions/`**：约 80 个样例，按主题覆盖：事件钩子（`confirm-destructive.ts`、`dirty-repo-guard.ts`）、自定义工具（`dynamic-tools.ts`、subagent）、命令与快捷键、自定义 UI（`custom-footer.ts`、`doom-overlay`）、git 集成（`auto-commit-on-exit.ts`）、系统提示改造（`claude-rules.ts`）、自定义 provider（`custom-provider-anthropic/`、`custom-provider-gitlab-duo/`）、SSH/sandbox 等。**写扩展先在这里找最近似样例**。
- **`examples/rpc-client.ts`**：`RpcClient` 子进程集成的最小示例；`examples/plugins/pi-example-plugin/`：pi 包（插件）样例。
- 英文文档总入口：`packages/coding-agent/docs/index.md`（约 40 篇）；本套文档索引：[docs/index.md](index.md)。

## 五、打包与分发

| 形态 | 方式 |
|------|------|
| 发布 npm | `npm run build && npm run check` 后走发布脚本（tarball 打包与 npm 发布同源，发布已验证产物）；包间引用用 workspace 版本范围 |
| **本地包给外部项目用** | `npm run pack:packages -- --out .artifacts/pi-packages`（先刷新模型数据；`--offline-model-data` 跳过）→ `node scripts/use-local-packages.mjs --manifest … --consumer ../my-project --package @earendil-works/pi-agent-core [--package-manager pnpm]` → 在消费者项目里 install。产物是**内容寻址的 `file:` 引用** + 传递依赖 overrides；改源码后两个命令都要重跑（根 README「Using local packages outside the monorepo」） |
| Standalone 二进制 | `scripts/build-binaries.sh`（bun 编译；release 源码包亦可复现，`--offline-model-data`/`--skip-install` 选项见根 README） |
| pi.dev 安装器 | 走 `packages/coding-agent/install-lock/`（从根 lockfile 生成的锁定集合；改依赖后 `npm run install-lock:coding-agent` 重新生成，`check` 会校验） |
| Nix | `nix run github:earendil-works/pi/stable`；模型目录 pin 在 `nix/model-catalog.json`（`npm run update:model-catalog-pin` 刷新） |

发布/版本：`release.mjs`（patch/minor/major）与 `sync-versions.js` 统一 workspace 版本；CHANGELOG 规则见 AGENTS.md（只动 `Unreleased` 段）。

## 六、测试设施地图

| 层面 | 设施 | 位置 |
|------|------|------|
| 全仓入口 | `./test.sh`（跳过 LLM 依赖测试） | 仓库根 |
| 单元/集成（vitest；tui 用 node:test） | 各包 `test/` | 各包 |
| coding-agent 运行时套件 | `test/suite/harness.ts` + **faux provider**（禁真实 API） | `packages/coding-agent/test/suite/` |
| TUI 渲染 | `test/virtual-terminal.ts`（@xterm/headless 真终端语义） | `packages/tui/test/` |
| 契约一致性 | telemetry adapter conformance；durable storage/env conformance；mcp 内存传输对端 | 各包 `src/testing/` |
| 行为评测 | host/docs 两种 eval；Docker 双镜像 lift 对比 | [T16](packages/evals.md) |
| 入口/边界静态检查 | `check:entry-graphs`、`check:runtime-deps`、boundary 测试（chord 零 Pi 依赖） | `scripts/`、各包 |

写测试的实用约定（AGENTS.md）：测试用自己的 harness 与 faux provider；修 GitHub issue 的回归测试在测试旁注明 issue 号；新测试先跑通再提交。

## 七、注意事项（踩坑高频区）

这些来自 AGENTS.md 与本次文档梳理，**动代码前先过一遍**：

1. **生成产物不可手改**：`packages/ai/src/models.generated.ts` 及 `providers/*.models.ts`、`data/*.json` 都是生成产物——改 `scripts/generate-models.ts` 后重新生成（[T03](packages/ai.md) 六.1）。
2. **erasable TypeScript 语法**：`packages/*/src|test` 与 `coding-agent/examples` 只允许 Node strip-only 模式能处理的语法——**禁止** `enum`/`namespace`/参数属性/`import =`；无 inline imports（动态 `await import()` 与内联类型导入都不行）；`any` 除非绝对必要。
3. **资产路径走 helper**：coding-agent 内访问包内资产（docs/examples/主题）必须经 `src/config.ts` 的 helper（`getDocsPath()` 等），不要 `__dirname`（源码/安装/二进制三种形态路径不同）。
4. **按键不得硬编码**：新快捷键加到默认键位表（`DEFAULT_EDITOR_KEYBINDINGS`/`DEFAULT_APP_KEYBINDINGS`/`TUI_KEYBINDINGS`），保持可配置。
5. **依赖变更 = 代码审查**：新依赖要精确版本 + 说明理由；生命周期脚本依赖需要显式 allowlist（install-lock 生成器里）。
6. **不跑 `npm run build`/`npm test` 除非必要**（AGENTS.md 命令约定）；文档改动无需任何检查。
7. **lockfile 提交被 pre-commit 拦截**（除非显式放行）；CI 用 `npm ci --ignore-scripts`。
8. **类型报错不要用降级代码"修"**：升级依赖而不是删类型（或改生成器）。
9. **实验轨的进入点**：`PI_EXPERIMENTAL=1`（server/client 子命令）；改实验代码前先确认没有被发布入口导入（`check:entry-graphs` 守着这条线）。
10. **提交信息格式**：`{feat,fix,docs}[(ai,tui,agent,coding-agent)]: …`；只 stage 自己改的文件（多 agent 同目录工作的约定）。

## 八、相关文档

- 根 [README.md](../README.md) 的 Development 节（构建/测试命令原文）；[AGENTS.md](../AGENTS.md)（规则全文，本篇只摘要）；[CONTRIBUTING.md](../CONTRIBUTING.md)（贡献流程）
- 写法教程（英文）：`packages/coding-agent/docs/` 的 `sdk.md`、`extensions.md`、`skills.md`、`prompt-templates.md`、`themes.md`、`packages.md`、`custom-provider.md`、`virtual-models.md`、`cli-integration.md`
- 本套：扩展点地图里的每篇 T0x 文档；下一步 [改造指引](recipes.md)（常见任务的"改哪里+怎么验证"清单）

---

[返回索引](index.md) · 任务与进度：[roadmap](../roadmap/README.md)
