# 08. Rust 重构蓝图

## 重构目标不是“换语言”，而是“重建运行时内核”

如果把这次工程定义为“把 TypeScript 改写成 Rust”，那么大概率会失败。真正正确的目标应该是：

> 基于现有系统中已经被证明有效的工程机制，重建一个更强类型、更高性能、更易测试、更易恢复的 Code Agent runtime。

因此，本章给出的不是语法迁移计划，而是架构重建路线。

## 一条总原则

### 保留语义，重写实现

必须保留的，是以下语义不变量：

1. 会话化消息流
2. query loop 的多 iteration 闭环
3. 工具与任务的统一执行语义
4. 上下文工程与压缩体系
5. 权限、并发、隔离、恢复机制
6. 扩展协议与分布式宿主能力

可以重写甚至应当推倒重来的，是以下具体实现：

1. 目前的模块依赖图
2. Node/Bun 特定加载模式
3. React/Ink 强耦合的局部状态形态
4. 部分为了 sourcemap 恢复或历史兼容而存在的临时结构

## 推荐总体架构

建议把新一代 Rust 项目拆成九个核心模块或 crate。

| 模块 | 职责 | 对应现有能力 |
| --- | --- | --- |
| `agent-message` | 消息 IR、转录、规范化、工具结果配对 | messages、session storage、API normalization |
| `agent-context` | system prompt、context layers、memory injection、cache-safe prefix | context、queryContext、memdir |
| `agent-query` | query state machine、event reducer、continuation logic | QueryEngine、query.ts |
| `agent-tools` | tool registry、tool scheduler、tool context、tool result budget | Tool.ts、tools.ts、tool orchestration |
| `agent-permissions` | permission context、rule evaluation、danger analysis | permissionSetup、permissions |
| `agent-tasks` | background tasks、agent tasks、task events、durable output | Task.ts、tasks、LocalAgentTask |
| `agent-extensions` | MCP、skills、plugins、workflows | mcp/client、skills、plugin loader |
| `agent-transport` | remote session、bridge、auth refresh、transport abstraction | bridge、remote |
| `agent-runtime` | bootstrap、config、policy、scheduler、observability、host adapters | main/init/setup/assistant |

## 应优先 Rust 化的部分

### 第一优先级：消息与 query 内核

优先重写：

- message normalization
- tool result pairing
- query state machine
- continuation conditions
- stream event reduction

原因：

- 类型收益最大
- 测试收益最大
- 最能形成跨宿主复用内核

### 第二优先级：权限与并发调度内核

优先重写：

- permission rule evaluation
- dangerous rule analysis
- concurrency-safe tool batching
- background/task isolation policy

原因：

- 这是最需要正确性的部分
- 适合 Rust 的强类型与枚举建模
- 最适合用 property-based testing 保证边界

### 第三优先级：上下文与压缩引擎

优先重写：

- token budget
- prompt section assembly
- compaction policy framework
- session memory / auto dream coordination

原因：

- 这是长期成本与性能的关键
- 需要更强的状态机和数据模型控制

### 第四优先级：transport 与分布式宿主

优先重写：

- bridge transport abstraction
- remote session subscription
- reconnect / JWT refresh / keepalive

原因：

- 边界复杂但相对独立
- 一旦 query 内核稳定，这层适合被抽象出来

## 不建议优先 Rust 化的部分

1. TUI 组件树
2. 复杂命令 UI 对话框
3. 插件管理界面
4. 各类非核心展示组件

原因不是这些不重要，而是它们不能决定新架构能否成立。

## 推荐技术路线

### 路线一：先做“Rust 内核 + TS 宿主”

这是最稳妥的路线。

做法：

1. 用 Rust 实现 message/query/permission/context 核心模块
2. 通过 FFI、N-API、WASM 或 sidecar 方式暴露给现有 TS 宿主
3. 逐步让现有 CLI/Bridge/插件系统调用 Rust 内核
4. 最后再决定哪些宿主能力继续保留 TS，哪些进一步 Rust 化

优势：

- 渐进迁移
- 风险低
- 可复用现有 UI 与插件生态

### 路线二：从一开始做纯 Rust 独立宿主

不建议一开始走这条路，除非团队已经明确放弃现有插件/技能生态和 Node 集成路径。

风险：

- 需要同时重做内核与宿主
- 迁移半径过大
- 很难快速验证行为一致性

## 建议的内部抽象

### 1. Message IR

建议使用枚举和结构化块：

- `User`
- `Assistant`
- `System`
- `Progress`
- `TaskNotification`
- `CompactBoundary`

每种消息都应能参与：

- transcript 持久化
- replay
- normalization
- cache-safe prefix 计算

### 2. Query State Machine

建议显式建模：

- `Prepare`
- `Streaming`
- `ToolDispatch`
- `AwaitToolResults`
- `Compacting`
- `Stopping`
- `Completed`
- `Failed`

### 3. Tool Scheduler

建议引入：

- `ToolCapability`
- `SideEffectClass`
- `ConcurrencyClass`
- `ToolBatch`
- `SchedulePlan`

### 4. Permission Kernel

建议引入：

- `PermissionMode`
- `PermissionRule`
- `PermissionContext`
- `PermissionDecision`
- `RuleRiskLevel`

### 5. Context Layering

建议引入：

- `PromptSection`
- `ContextLayer`
- `MemoryArtifact`
- `CompactionArtifact`
- `CacheSafePrefix`

## 迁移阶段建议

### Phase 0：建立行为基线

在现有仓库上建立黄金数据集：

- query replay fixtures
- tool orchestration fixtures
- permission rule fixtures
- compaction fixtures
- bridge transport fixtures

目的：让 Rust 实现有明确等价目标。

### Phase 1：重建消息与 query reducer

交付物：

- Rust message IR
- Rust query reducer
- TS 宿主桥接层

### Phase 2：重建 permission + tool scheduler

交付物：

- Rust permission evaluator
- Rust dangerous rule analyzer
- Rust tool batching planner

### Phase 3：重建 context/memory/compaction

交付物：

- Rust context builder
- Rust compaction engine
- Rust session memory engine

### Phase 4：重建 task/runtime/transport

交付物：

- Rust task runtime
- Rust bridge/remote transport core
- assistant/coordinator 调度框架

## 测试策略建议

### 必须具备的测试类型

1. Golden replay tests
2. Property-based tests
3. Stateful model tests
4. Failure injection tests
5. Recovery tests
6. Cache-key stability tests

### 最需要做 golden replay 的部分

1. tool result pairing
2. compact 触发与输出
3. permission decision
4. task notification 回流
5. bridge reconnect 行为

## 风险清单

### 风险一：只重构“快”的部分，忽略“难”的部分

结果会得到一个性能更好但能力更弱的系统。

### 风险二：过早放弃现有扩展生态

结果会导致新系统虽然内核更干净，但短期失去 MCP、skills、plugins 的平台价值。

### 风险三：没有先建立行为基线

结果是无法判断 Rust 实现到底是“更好”还是“语义偏移”。

### 风险四：先做 Bridge/多代理，后做内核

结果会把不稳定内核的问题放大到分布式层。

## 最终建议

新一代 Rust Code Agent 项目，最合理的定位应是：

> 一个以 Rust 为内核、以结构化消息和状态机为基础、以权限和上下文工程为核心竞争力、以 MCP/插件/远程能力为外壳的本地优先 agent runtime。

换句话说，最值得重构的不是“CLI 工具”，而是“agent operating system kernel”。

## 代码证据

- [src/QueryEngine.ts](../src/QueryEngine.ts)
- [src/query.ts](../src/query.ts)
- [src/Tool.ts](../src/Tool.ts)
- [src/Task.ts](../src/Task.ts)
- [src/services/tools/toolOrchestration.ts](../src/services/tools/toolOrchestration.ts)
- [src/utils/permissions/permissionSetup.ts](../src/utils/permissions/permissionSetup.ts)
- [src/context.ts](../src/context.ts)
- [src/services/SessionMemory/sessionMemory.ts](../src/services/SessionMemory/sessionMemory.ts)
- [src/services/autoDream/autoDream.ts](../src/services/autoDream/autoDream.ts)
- [src/services/mcp/client.ts](../src/services/mcp/client.ts)
- [src/bridge/bridgeMain.ts](../src/bridge/bridgeMain.ts)
- [src/coordinator/coordinatorMode.ts](../src/coordinator/coordinatorMode.ts)
- [src/utils/forkedAgent.ts](../src/utils/forkedAgent.ts)
