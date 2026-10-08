# pi agent 架构文档

一套**源码级中文架构文档**，用于学习 pi 的架构并进行二次开发。

- 仓库：pi monorepo（14 个 workspace 包，约 17.8 万行 TypeScript）
- 定位：关键流程给出调用链与源码位置（`file:line`）；与 `packages/*/README.md`、`packages/coding-agent/docs/` 的英文用户文档互补，不重复
- 任务路线图、写作规范与进度：见 [roadmap/README.md](../roadmap/README.md)

## 阅读路径

### 路径 A：产品视角（推荐首次阅读）

1. [主体架构总览](architecture.md)——先建立全景：分层、双轨（生产/实验）、依赖关系
2. [coding-agent 启动与运行模式](packages/coding-agent-startup.md)——从进程入口进入
3. [coding-agent 核心运行时](packages/coding-agent-runtime.md)——会话、工具、压缩
4. [agent 运行时](packages/agent.md)——生产链路的 agent 循环
5. [pi-ai](packages/ai.md)——模型调用与 provider 体系
6. [pi-tui](packages/tui.md)——终端 UI 与差分渲染
7. [coding-agent 扩展系统与 SDK](packages/coding-agent-extensions.md)——二次开发主入口
8. [关键数据流时序图](data-flows.md)——把以上串成端到端链路
9. [二次开发指南](development.md) → [改造指引](recipes.md)

### 路径 B：依赖视角（自底向上）

[架构总览](architecture.md) → [pi-ai](packages/ai.md) → [agent 运行时](packages/agent.md) → [pi-tui](packages/tui.md) → coding-agent 三篇（[启动](packages/coding-agent-startup.md) / [运行时](packages/coding-agent-runtime.md) / [扩展与 SDK](packages/coding-agent-extensions.md)）→ 周边（MCP、codemode、实验轨、评测）

### 按需查阅

- 术语不认识：[术语表](glossary.md)
- 想改东西：[二次开发指南](development.md)、[改造指引](recipes.md)
- 实验性组件（chord/durable/env/远程 C/S）：见「实验轨」分组

## 文档清单

状态：全部**已写**（2026-10-08 完成，共 20 篇）。

### 总览

| 文档 | 内容 | 状态 |
|------|------|------|
| 本页 | 总索引与阅读路径 | 已写 |
| [glossary.md](glossary.md) | 核心概念术语表（含易混淆概念对照） | 已写 |
| [architecture.md](architecture.md) | 主体架构：分层、依赖矩阵、生产/实验双轨 | 已写 |
| [data-flows.md](data-flows.md) | 关键数据流时序图（启动/消息全链路/压缩/实验轨） | 已写 |

### 生产主链路

| 文档 | 内容 | 状态 |
|------|------|------|
| [packages/coding-agent-startup.md](packages/coding-agent-startup.md) | 启动流程、参数、interactive/print/json/rpc 四模式 | 已写 |
| [packages/coding-agent-runtime.md](packages/coding-agent-runtime.md) | AgentSession、工具系统、JSONL 会话树、压缩、模型层 | 已写 |
| [packages/coding-agent-extensions.md](packages/coding-agent-extensions.md) | 扩展系统、内置扩展、资源加载、Pi packages、SDK | 已写 |
| [packages/agent.md](packages/agent.md) | agent 运行时（pi-agent-core）：agent loop、工具执行、事件 | 已写 |
| [packages/ai.md](packages/ai.md) | 多 provider LLM API：类型、provider 工厂、wire 适配、认证 | 已写 |
| [packages/tui.md](packages/tui.md) | 终端 UI：差分渲染、布局、输入管线、native 模块 | 已写 |

### 关键周边

| 文档 | 内容 | 状态 |
|------|------|------|
| [packages/mcp.md](packages/mcp.md) | MCP 客户端：transport、OAuth、与 coding-agent 集成 | 已写 |
| [packages/codemode.md](packages/codemode.md) | QuickJS-WASM 沙箱：执行链路、中断模型、store | 已写 |

### 实验轨（不在默认运行路径）

| 文档 | 内容 | 状态 |
|------|------|------|
| [packages/chord.md](packages/chord.md) | 应用组合运行时：facets、services、replicated state、delta | 已写 |
| [packages/durable.md](packages/durable.md) | 持久化 agent 运行时：entry/task/document、提交管线、崩溃恢复 | 已写 |
| [packages/env.md](packages/env.md) | 远端执行环境：TS 客户端 + Rust daemon、SSH 安全链条 | 已写 |
| [packages/remote-sessions.md](packages/remote-sessions.md) | CBOR 远程会话 C/S 体系（protocol/client/server 三包合一） | 已写 |

### 横向与基建

| 文档 | 内容 | 状态 |
|------|------|------|
| [packages/telemetry.md](packages/telemetry.md) | 遥测契约包：契约语义、参考实现、conformance | 已写 |
| [packages/evals.md](packages/evals.md) | 评测工具：host/docs 两种模式、harness、Docker 链 | 已写 |

### 二次开发

| 文档 | 内容 | 状态 |
|------|------|------|
| [development.md](development.md) | 构建/调试/测试工作流、扩展点地图、打包分发 | 已写 |
| [recipes.md](recipes.md) | 常见改造需求的「改哪里、怎么验证」清单 | 已写 |

## 与现有英文文档的关系

- `packages/coding-agent/docs/`（约 40 篇）与各包 `README.md` 是**用户/行为级**文档：怎么用、有什么能力。
- 本套文档是**源码级架构**文档：实现在哪、怎么运转、改动会波及什么。每篇文末「相关文档」会引用对应英文文档。
