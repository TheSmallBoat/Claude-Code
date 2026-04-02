# 07. 分布式与自治：Bridge、KAIROS、Coordinator

## 本章核心判断

Claude Code 最值得重视的，不只是它会调工具，而是它已经显露出“分布式 agent 平台”的雏形。Bridge、Remote、KAIROS、Coordinator 并不是几个离散 feature，而是系统向更高阶 agent 形态演进的四个方向：

1. 远程接入
2. 持续运行
3. 多角色分工
4. 跨会话自治

对下一代 Rust 项目而言，这一章决定它是否只是“本地命令代理”，还是“可持续运行的 agent 平台”。

## Bridge：把本地运行时变成远程可控执行宿主

### Bridge 的本质

从 [src/bridge/bridgeMain.ts](../src/bridge/bridgeMain.ts)、[src/bridge/replBridge.ts](../src/bridge/replBridge.ts)、[src/bridge/remoteBridgeCore.ts](../src/bridge/remoteBridgeCore.ts) 可以看出，Bridge 的本质不是“同步终端输出”，而是：

> 将本地 Claude Code 运行时注册为远程可访问的执行环境，并让远端成为其输入、控制和观察端。

### 两种形态

1. 独立 bridge server
2. REPL 内嵌 bridge

这两种形态共用大量消息与传输逻辑，但宿主边界不同：

- 独立 server 适合作为环境宿主
- REPL bridge 适合把现有本地会话镜像到远端

### 为什么它重要

Bridge 证明 Claude Code 的设计并不假设“用户必须坐在本机终端前”。这对新一代系统极其关键，因为真正成熟的 Code Agent 必须支持：

- 本地执行
- 远程观察
- 远程控制
- 权限回传
- 会话恢复

### 协议抽象价值

Bridge 同时支持环境层方案和 env-less v2 方案，说明其 transport layer 与 execution layer 已经开始解耦。Rust 版应该把这一点进一步做实。

## Remote Session：远端会话的消费与镜像

[src/remote/RemoteSessionManager.ts](../src/remote/RemoteSessionManager.ts) 与相关 WebSocket 代码说明，系统还能把远端 session 当作一等运行对象管理。

RemoteSessionManager 负责：

- WebSocket 订阅
- 远端消息转本地 SDKMessage
- 远端权限请求回路
- reconnect 与 viewer-only 行为差异

### 设计意义

Claude Code 并不把“本地会话”和“远端会话”做成完全不同的产品，而是尽量在消息协议层统一它们。这是非常正确的方向。

## KAIROS：让 agent 从会话工具进化为持续运行体

[docs/02-kairos.md](../docs/02-kairos.md)、[src/assistant/index.ts](../src/assistant/index.ts)、[src/proactive/index.ts](../src/proactive/index.ts)、[src/services/autoDream/autoDream.ts](../src/services/autoDream/autoDream.ts) 共同展示了 KAIROS 的系统意义。

### KAIROS 真正解决的问题

它不是“后台运行”那么简单，而是在解决：

- agent 如何跨终端生命周期存在
- agent 如何在无人交互时继续推进工作
- agent 如何通过 cron 和睡眠节奏管理自治行为
- agent 如何对多会话历史进行记忆整理

### 工程层面最有价值的点

1. 助手模式是 runtime state，而不是简单 CLI flag。
2. 主动模式维护 `active/paused/contextBlocked/nextTickAt` 等显式状态。
3. Dream 被建模为后台任务，而不是隐藏子线程。
4. 计划性和长期性通过 persistent task / logs / memory dir 组合体现。

### 对 Rust 的启示

如果未来要做持续运行代理，应该把自治调度器、睡眠语义、定时任务、记忆整理器做成内核能力，而不是后期“加个 daemon”。

## Coordinator：多 Agent 编排模式

[src/coordinator/coordinatorMode.ts](../src/coordinator/coordinatorMode.ts) 与 [docs/04-coordinator.md](../docs/04-coordinator.md) 表明，Coordinator 模式的关键不在于“能起多个 agent”，而在于：

> 系统开始显式区分协调者与执行者的职责边界。

### Coordinator 模式的四个关键设计点

1. Coordinator 拥有受限工具集，只做分派、追踪、综合。
2. Worker 拥有执行能力，但看不到 coordinator 的完整上下文。
3. Worker 结果以结构化 task notification 回注。
4. Prompt 明确要求 coordinator 自己综合，不允许甩锅式委派。

### 这说明什么

Claude Code 已经意识到，多代理系统的难点不是“并行调用”，而是：

- 上下文边界
- 角色责任
- 结果回流
- 避免委派链失真

这是下一代 multi-agent runtime 设计时必须重点继承的思想。

## 子代理与后台代理是多代理平台的基建

如果只看 Bridge、KAIROS、Coordinator，会误以为高阶能力都在 feature docs 里；但真正的基建其实在：

- [src/utils/forkedAgent.ts](../src/utils/forkedAgent.ts)
- [src/tools/AgentTool/runAgent.ts](../src/tools/AgentTool/runAgent.ts)
- [src/tasks/LocalAgentTask/LocalAgentTask.tsx](../src/tasks/LocalAgentTask/LocalAgentTask.tsx)

它们共同提供了：

- 上下文派生
- agent 执行包装
- transcript 侧链记录
- task notification 回流
- background lifecycle 管理

这说明高阶自治能力不是凭空出现的，而是建立在统一 agent kernel 之上的。

## 分布式与自治的统一视角

可以用下面的视角统一看待这些能力。

```text
+---------------------------+
| Main Local Agent          |
+---------------------------+
   |
   +--> +-------------------+
   |    | Forked Subagent   |
   |    +-------------------+
   |
   +--> +-------------------+
   |    | Background Task   |
   |    | Agent             |
   |    +-------------------+
   |
   +--> +-------------------+
   |    | Coordinator       |
   |    | Worker            |
   |    +-------------------+
   |
   +--> +-------------------+      +-----------------------+
   |    | Bridge Session    | ---> | Remote User / Web     |
   |    +-------------------+      +-----------------------+
   |
   +--> +---------------------------+      +---------------------------+
        | Assistant / KAIROS        | ---> | Dream / Cron / Proactive |
        | Runtime                   |      | Loop                      |
        +---------------------------+      +---------------------------+
```

这张图想表达的不是拓扑，而是一个更重要的结论：

> 这些高阶能力都不是独立产品，而是同一 agent kernel 在不同生命周期、不同控制权和不同传输边界下的投影。

## Rust 重构建议

### 建议模块化方式

1. `agent-kernel`
2. `agent-spawn`
3. `background-runtime`
4. `bridge-transport`
5. `remote-session`
6. `coordinator-runtime`
7. `assistant-runtime`
8. `scheduler-and-dream`

### 建议先后顺序

1. 先统一本地 agent 与 forked agent 内核
2. 再引入 background task runtime
3. 再引入 coordinator role model
4. 最后接入 remote/bridge transport

### 为什么不应先做 Bridge

如果底层 agent kernel 与 task model 没稳定，Bridge 只会放大不稳定性。正确顺序是先内核，后分布式宿主。

## 本章结论

Bridge、KAIROS、Coordinator 证明 Claude Code 的终局不是单终端命令行助手，而是可远控、可持续运行、可多角色协同的 agent 平台。新一代 Rust 项目如果只复刻主循环和工具系统，而忽略这一层，最终只会得到一个“更快的单机会话代理”，而不是完整平台。

## 代码证据

- [src/bridge/bridgeMain.ts](../src/bridge/bridgeMain.ts)
- [src/bridge/replBridge.ts](../src/bridge/replBridge.ts)
- [src/bridge/remoteBridgeCore.ts](../src/bridge/remoteBridgeCore.ts)
- [src/bridge/bridgeMessaging.ts](../src/bridge/bridgeMessaging.ts)
- [src/remote/RemoteSessionManager.ts](../src/remote/RemoteSessionManager.ts)
- [src/coordinator/coordinatorMode.ts](../src/coordinator/coordinatorMode.ts)
- [src/utils/forkedAgent.ts](../src/utils/forkedAgent.ts)
- [src/tools/AgentTool/runAgent.ts](../src/tools/AgentTool/runAgent.ts)
- [src/tasks/LocalAgentTask/LocalAgentTask.tsx](../src/tasks/LocalAgentTask/LocalAgentTask.tsx)
- [src/assistant/index.ts](../src/assistant/index.ts)
- [src/proactive/index.ts](../src/proactive/index.ts)
- [src/services/autoDream/autoDream.ts](../src/services/autoDream/autoDream.ts)
- [docs/02-kairos.md](../docs/02-kairos.md)
- [docs/04-coordinator.md](../docs/04-coordinator.md)
- [docs/06-bridge.md](../docs/06-bridge.md)
