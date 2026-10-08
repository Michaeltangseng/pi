# pi agent 架构文档 · 任务路线图

> **状态：全部完成（2026-10-08）。** T00–T19 共 20 篇文档已全部写入 `../docs/`，进度表全绿。后续如需维护，按「写作规范」增量修订即可。

本目录是 pi monorepo 本地副本的**中文源码级架构文档体系**的总纲与进度表。目标：通过一套可导航的文档建立对 pi 架构的完整认知，支撑二次开发。

- 仓库：`../`（pi monorepo，14 个 workspace 包，约 17.8 万行 TypeScript）
- 成品文档位置：`../docs/`
- 任务详情：见 [tasks.md](tasks.md)（每个任务含：目标、必读源码、内容大纲、验收标准）

## 背景速览（2026-10 调研结论）

写作时以源码现状为准，以下结论用于快速定位方向：

- **生产主链路**：`coding-agent`（CLI/产品编排，8.6 万行）→ `pi-agent-core`（agent 循环，仅 6 个文件）→ `pi-ai`（多 provider 模型调用，2.7 万行）。`pi-tui` 是 coding-agent 专属 UI 框架；`pi-mcp`（MCP 客户端）与 `pi-codemode`（QuickJS-WASM 沙箱）在使用 MCP 时构成条件性关键路径。
- **实验轨**：`chord`（应用组合运行时：facets/services/replicated state/delta，零 Pi 依赖）→ `durable`（持久化 agent 运行时：entry/task/document + 提交管线）→ `env`（SSH 远端执行 + Rust daemon）。三者只在 `coding-agent/src/experimental/` 或独立产品中使用，**不在默认运行路径**。
- **各自独立的子系统**：`protocol`/`client`/`server`（CBOR 远程会话 C/S，实验）、`telemetry`（仅有契约、零调用点）、`evals`（评测工具，不进运行时）。
- **易混淆点（文档中必须逐一澄清）**：
  1. 两套 agent loop：agent-core 的 `Agent`（生产）vs durable 的 task 化 loop（实验）。`AgentTool.replay` 字段在 agent 包中定义但未被消费，在 durable 的 ToolTask 中才真正生效。
  2. 两套远程机制：JSONL over stdio 的 **RPC 模式**（`coding-agent/src/modes/rpc/`，成熟）vs CBOR 层封装的 **C/S 体系**（`protocol/client/server` + `coding-agent/src/experimental/`，实验）。
  3. 两套 telemetry：`pi-telemetry` 契约包（无调用点）vs coding-agent 的安装遥测（`src/core/telemetry.ts` + `PI_TELEMETRY`，与之无关）。
  4. 会话存储：coding-agent 生产使用自研 append-only JSONL 会话树（`src/core/session-manager.ts`），**不是** durable。
  5. 部分包的 README 描述的是目标形态而非现状（chord 缺对称 RPC、telemetry 无消费端），文档需标注「已实现 / 未实现」。

## 目标文档体系

```
docs/
├── index.md                          # 总索引与阅读路径            [T00 已完成]
├── glossary.md                       # 核心概念术语表              [T01 已完成]
├── architecture.md                   # 主体架构总览（分层/依赖/双轨）[T02 已完成]
├── data-flows.md                     # 关键数据流时序图（Mermaid）  [T17 已完成]
├── development.md                    # 二次开发指南（构建/调试/扩展点地图）[T18 已完成]
├── recipes.md                        # 改造指引（常见需求"改哪里"）  [T19 已完成]
└── packages/
    ├── coding-agent-startup.md       # 启动、参数、四运行模式、RPC  [T06 已完成]
    ├── coding-agent-runtime.md       # AgentSession、loop、工具、会话存储、压缩 [T07 已完成]
    ├── coding-agent-extensions.md    # 扩展系统、内置扩展、资源加载、SDK [T08 已完成]
    ├── ai.md                         # 多 provider LLM API         [T03 已完成]
    ├── agent.md                      # agent 运行时（agent loop）   [T04 已完成]
    ├── tui.md                        # 终端 UI 与差分渲染           [T05 已完成]
    ├── mcp.md                        # MCP 客户端                  [T09 已完成]
    ├── codemode.md                   # QuickJS-WASM 沙箱           [T10 已完成]
    ├── chord.md                      # 组合运行时（实验轨基座）      [T11 已完成]
    ├── durable.md                    # 持久化 agent 运行时（实验轨） [T12 已完成]
    ├── env.md                        # 远端执行环境（实验轨）        [T13 已完成]
    ├── remote-sessions.md            # protocol/client/server 三包合一 [T14 已完成]
    ├── telemetry.md                  # 遥测契约包                  [T15 已完成]
    └── evals.md                      # 评测工具                    [T16 已完成]
```

## 写作规范

### 定位与语言

- 中文正文；代码标识符、文件名、专有名词保留英文；术语首次出现时用一句话解释并链接 `docs/glossary.md` 对应条目。
- 定位为**源码级架构文档**，与现有英文文档互补而非重复：`packages/*/README.md`、`packages/coding-agent/docs/`（约 40 篇用户/行为文档）在每篇文末「相关文档」中引用链接（相对路径 `../../packages/...`）。
- 深度要求：关键流程必须给出**调用链**（入口 → 中间层 → 出口），每跳带 `file:line` 引用；核心数据结构给出定义位置与关键字段。

### 引用格式

- 源码路径统一**相对仓库根**书写，如 `packages/coding-agent/src/core/agent-session.ts`；行号引用写成 `` `packages/.../agent-session.ts:123` ``。
- 行号必须经过 Read/Grep 验证（引用类/函数定义处）；无法核实的行号不要写，只写文件路径。
- 代码片段 ≤ 10 行，只贴关键类型/签名，不贴大段实现。

### 每篇包文档统一结构

1. **定位**：一句话职责 + 在架构中的位置（小依赖图，Mermaid 可选）
2. **目录结构与模块地图**：文件/目录表，每项一句话职责
3. **核心数据结构**：定义位置 + 关键字段
4. **关键运行流程**：逐条「触发 → 调用链（带 file:line）→ 结果」；重要流程配 Mermaid 时序图
5. **对外接口与扩展点**：公开 API、被谁消费、二次开发的接入点
6. **现状与陷阱**：已实现/未实现标注、易混淆点、已知限制
7. **二次开发落点**：想做 X 应该改哪些文件
8. **相关文档**：本篇引用的现有英文文档 + 本套文档其他篇（含返回索引链接）

### 篇幅建议

| 级别 | 包 | 篇幅 |
|------|----|------|
| 小 | telemetry、evals、protocol 系（合并后）、codemode | 150–300 行 |
| 中 | ai、agent、tui、mcp、chord、durable、env | 300–600 行 |
| 大 | coding-agent 每篇 | 400–800 行 |
| 总览 | architecture、data-flows | 300–500 行 |

### 硬性规则

- 文档任务**只读源码、只写 `docs/` 与 `roadmap/`**；不修改 `packages/` 下任何文件，不运行构建/测试（文档改动无需 `npm run check`）。
- Mermaid 图仅用于结构关系与关键时序；语法需有效。
- 现有 README 描述与源码冲突时，以源码为准，并在「现状与陷阱」中说明。
- 完成任务后：勾选本文件进度表；将 `docs/index.md` 对应条目状态改为「已写」；若引入新术语，追加到 `docs/glossary.md`。

## 执行方式

- 每个任务产出 1 篇文档，T01 起按序执行；T17–T19 汇总类文档必须最后写（依赖前序任务）。
- 在新会话中执行任务时，对 Claude Code 说：**「执行 roadmap/tasks.md 的 T0X，遵循 roadmap/README.md 的写作规范」**。
- 任务间依赖见进度表「依赖」列：标注的依赖任务应先完成；其余任务可任意顺序执行。

## 进度总表

| 任务 | 产出 | 内容 | 状态 | 依赖 |
|------|------|------|------|------|
| T00 | `docs/index.md` | 总索引与阅读路径 | 已完成 | — |
| T01 | `docs/glossary.md` | 核心概念术语表 | 已完成 | — |
| T02 | `docs/architecture.md` | 主体架构总览 | 已完成 | — |
| T03 | `docs/packages/ai.md` | 多 provider LLM API | 已完成 | T01, T02 |
| T04 | `docs/packages/agent.md` | agent 运行时 | 已完成 | T01, T02 |
| T05 | `docs/packages/tui.md` | 终端 UI 与差分渲染 | 已完成 | T01, T02 |
| T06 | `docs/packages/coding-agent-startup.md` | 启动与四运行模式 | 已完成 | T01, T02 |
| T07 | `docs/packages/coding-agent-runtime.md` | 核心运行时与会话 | 已完成 | T01, T02 |
| T08 | `docs/packages/coding-agent-extensions.md` | 扩展系统与 SDK | 已完成 | T01, T02 |
| T09 | `docs/packages/mcp.md` | MCP 客户端 | 已完成 | T01, T02 |
| T10 | `docs/packages/codemode.md` | QuickJS-WASM 沙箱 | 已完成 | T01, T02 |
| T11 | `docs/packages/chord.md` | 组合运行时 | 已完成 | T01, T02 |
| T12 | `docs/packages/durable.md` | 持久化 agent 运行时 | 已完成 | T01, T02, T11 |
| T13 | `docs/packages/env.md` | 远端执行环境 | 已完成 | T12 |
| T14 | `docs/packages/remote-sessions.md` | CBOR C/S 远程会话体系 | 已完成 | T01, T02 |
| T15 | `docs/packages/telemetry.md` | 遥测契约包 | 已完成 | T01, T02 |
| T16 | `docs/packages/evals.md` | 评测工具 | 已完成 | T01, T02 |
| T17 | `docs/data-flows.md` | 关键数据流时序图 | 已完成 | T03, T04, T07 |
| T18 | `docs/development.md` | 二次开发指南 | 已完成 | T06–T08, T03 |
| T19 | `docs/recipes.md` | 改造指引 | 已完成 | T18 |

## 每篇文档通用自检清单

- [ ] 结构完整：8 个规定小节齐全（包文档）或大纲要求的内容齐备（总览类）
- [ ] 所有 `file:line` 引用经 Read/Grep 验证存在且指向正确
- [ ] 关键流程有调用链，重要流程配 Mermaid 图且语法有效
- [ ] 术语与 `docs/glossary.md` 一致；新术语已补录
- [ ] 现有英文文档以相对链接引用，无大段重复
- [ ] 生产/实验归属、已实现/未实现标注明确
- [ ] `docs/index.md` 与 `roadmap/README.md` 状态已更新
