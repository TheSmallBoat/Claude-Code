# Runtime Core 分层设计

## 文档状态

- 状态：Draft
- 日期：2026-04-03
- 作用：把 `01-07` 模块规格收束为可实现的 Rust 分层蓝图

## 1. 设计目标

本设计的目标不是复刻 TS 工程目录，而是回答四个工程问题：

1. Rust workspace 应拆成哪些中层模块。
2. 模块之间允许怎样的依赖方向。
3. 哪类状态应该归属于哪一层。
4. 背景任务、压缩、记忆、coding adapters 如何挂接到同一 query 内核上。

## 2. 总体分层

建议把基线版分成六层：

1. 数据与持久化层
2. 契约与治理层
3. 查询执行层
4. 长上下文治理层
5. 多执行体运行时层
6. 宿主能力与领域工作流层

对应关系如下：

```text
L6 宿主能力与领域工作流
   - agent-code-adapters
   - agent-code-workflows
   - CLI / TUI / SDK / Remote hosts

L5 多执行体运行时
   - agent-tasks
   - agent-subagents

L4 长上下文治理
   - agent-compact
   - agent-memory

L3 查询执行
   - agent-query

L2 契约与治理
   - agent-tools
   - agent-permissions
   - agent-context

L1 数据与持久化
   - agent-message
   - agent-session-store
```

## 3. 各层职责

### 3.1 L1 数据与持久化层

这一层只负责稳定事实模型：

1. message IR
2. transcript / sidechain transcript 读写
3. compact boundary / preserved segment relink
4. session-scoped append/load primitives

这一层不得直接依赖：

1. shell
2. git
3. LSP
4. skills
5. query loop

### 3.2 L2 契约与治理层

这一层负责把执行内核需要的治理面统一出来：

1. tool descriptor 与 scheduler contract
2. permission mode / rules / decision model
3. context section / cache-safe prefix / attachment planning

这层可以知道“要调用某种 adapter”，但不应直接持有具体 coding host 实现。

### 3.3 L3 查询执行层

`agent-query` 是主线程与所有子线程共享的唯一回合执行内核，负责：

1. submit message
2. turn state machine
3. model sampling
4. tool dispatch
5. stop hooks
6. transcript append

所有可恢复执行体最终都应回到这同一层运行，而不是复制一份专用子代理状态机。

### 3.4 L4 长上下文治理层

`agent-compact` 与 `agent-memory` 负责：

1. tool result budget shrinking
2. microcompact
3. proactive compact
4. reactive overflow recovery
5. session memory extraction / consumption

它们服务于 query 层，但不应拥有自己的消息事实源。

### 3.5 L5 多执行体运行时层

`agent-tasks` 与 `agent-subagents` 负责：

1. background task control plane
2. subagent lifecycle
3. backgrounded main-session query
4. sidechain transcript + metadata
5. resume

这层的关键职责不是“再造一个 query engine”，而是复用 L3 的 query 内核，并提供多执行体包装。

### 3.6 L6 宿主能力与领域工作流层

这一层才放：

1. 文件读写
2. grep / glob / shell / LSP / git / IDE bridge
3. skills / workflows / coding commands
4. CLI / TUI / SDK / Remote host assembly

这意味着“编程能力”是上层装配包，而不是 runtime core 本体。

## 4. 模块依赖方向

建议遵守如下依赖图：

```text
agent-message
    -> agent-session-store

agent-message + agent-session-store
    -> agent-context

agent-permissions
    -> agent-tools

agent-message + agent-session-store + agent-context + agent-tools + agent-permissions
    -> agent-query

agent-query + agent-context + agent-session-store
    -> agent-compact
    -> agent-memory

agent-query + agent-session-store
    -> agent-tasks
    -> agent-subagents

agent-tools + agent-permissions + agent-tasks
    -> agent-code-adapters

agent-code-adapters + agent-subagents + agent-tools
    -> agent-code-workflows
```

额外约束：

1. `agent-message` 不能反向依赖任何更高层。
2. `agent-query` 不得直接依赖 CLI/TUI/SDK host。
3. `agent-code-adapters` 不得定义新的 transcript 事实模型。
4. `agent-code-workflows` 只能组合 adapter / subagent，不得改写 query 主状态机。

## 5. 状态归属矩阵

### 5.1 `agent-query` 持有的状态

1. 当前 conversation messages
2. 当前 turn usage
3. abort controller
4. turn-scoped discovered skills / nested memory tracking

### 5.2 `agent-session-store` 持有的状态

1. transcript append cursor
2. compact boundary metadata
3. preserved segment relink 能力

### 5.3 `agent-context` 持有的状态

1. cache-safe params snapshot
2. system/user context contributors
3. attachment planning state

### 5.4 `agent-memory` 持有的状态

1. session memory thresholds
2. last summarized message id
3. relevant memory surfacing budget

### 5.5 `agent-tasks` 持有的状态

1. task records
2. output file path / offset
3. notified / eviction tracking

### 5.6 `agent-subagents` 持有的状态

1. agent identity scope
2. sidechain transcript metadata
3. foreground/background mode
4. resume context reconstruction state

### 5.7 `agent-code-adapters` 持有的状态

1. read-file cache mirror needed by coding tools
2. sandbox / shell runtime state
3. LSP client/session state
4. IDE bridge connection state

## 6. 一次主线程回合的标准流

建议把主线程一次 turn 明确定义为：

1. `agent-context` 组装 system/user context。
2. `agent-query` 构造 request working set。
3. `agent-query` 进入 streaming sampling。
4. `agent-tools` 调度具体 tool call。
5. 若命中 coding tool，则落到 `agent-code-adapters`。
6. tool result 回写 message IR 与 transcript。
7. `agent-compact` / `agent-memory` 根据预算和阈值介入。
8. `agent-query` 执行 stop hooks，决定继续还是终止。

这样 query 层始终是回合闭环中心，而不是让 tools、tasks、memory 互相直接驱动。

## 7. 子代理与后台执行的标准流

建议把多执行体流程定义为：

1. `agent-subagents` 申请新的 agent identity。
2. `agent-tasks` 注册 task record。
3. `agent-subagents` 建 sidechain transcript / metadata。
4. 在新的 context snapshot 上调用同一个 `agent-query`。
5. 若进入后台，则由 `agent-tasks` 接管可观察状态。
6. 终止时先更新 terminal task state，再做 cleanup / notification。

这保证主线程 query 和 background agent 共享同一个内核，只是拥有不同的控制面与持久化容器。

## 8. compaction 与 subagent 的层间关系

这里必须避免两个常见错误：

1. 不要让 `agent-compact` 直接知道 task registry。
2. 不要让 `agent-subagents` 自己发明另一套 compact/memory 机制。

正确做法是：

1. compact/memory 依旧是 query 的标准子服务。
2. subagent 只是给 query 提供新的 context、tools、transcript、task envelope。
3. session memory / summarize 这类内部 fork 可以走轻量 fork runner，但仍复用相同的 message/context contracts。

## 9. 宿主层与核心层的边界

中层设计里必须明确以下边界：

1. core 不知道 `git`, `rg`, `bash`, `.ipynb`, `LSP`。
2. core 只知道存在一些 tool providers / context contributors / workflow providers。
3. coding host adapters 不能改写 core 的 message、task、transcript 契约。
4. CLI/TUI/SDK/Remote 只在最外层装配，不得向内层渗漏展示状态结构。

## 10. 失败域划分

建议划分四类失败域：

### 10.1 Query failure domain

1. 模型 API error
2. tool execution error
3. stop hook error

由 `agent-query` 统一收敛成 turn result。

### 10.2 Context governance failure domain

1. autocompact failure
2. reactive compact failure
3. session memory extraction failure

由 `agent-compact` / `agent-memory` 局部熔断，不应直接破坏 transcript 主链。

### 10.3 Background runtime failure domain

1. async agent failure
2. resume failure
3. sidechain corruption

由 `agent-subagents` / `agent-tasks` 收敛成 terminal task state。

### 10.4 Host adapter failure domain

1. shell sandbox failure
2. LSP unavailable
3. git snapshot failure
4. IDE bridge notification failure

这些失败原则上 fail-open 到 query/tool result，而不应让 core 层状态失真。

## 11. 建议的 Rust workspace 拆分

最小可行 workspace 可以是：

1. `agent-message`
2. `agent-session-store`
3. `agent-permissions`
4. `agent-tools`
5. `agent-context`
6. `agent-query`
7. `agent-compact`
8. `agent-memory`
9. `agent-tasks`
10. `agent-subagents`
11. `agent-code-adapters`
12. `agent-code-workflows`

若需要进一步压缩首版规模，可以把 `agent-compact` 与 `agent-memory` 先合并，把 `agent-code-workflows` 暂时放在 host crate 中。

## 12. 实现顺序建议

建议按以下顺序开工：

1. `agent-message`
2. `agent-session-store`
3. `agent-permissions`
4. `agent-tools`
5. `agent-context`
6. `agent-query`
7. `agent-compact` + `agent-memory`
8. `agent-tasks` + `agent-subagents`
9. `agent-code-adapters`
10. `agent-code-workflows`

原因很简单：只有先把消息、转录、query 内核和上下文治理做稳，后面的背景任务和 coding host adapters 才不会反复返工。

## 13. 设计验收标准

这个中层设计只有在以下条件都满足时才算成立：

1. 任一层都不需要反向依赖更高层。
2. 主线程 query 与 subagent query 共用同一个执行内核。
3. transcript、task、context、memory 各自有清晰 owner。
4. coding host adapters 可以整体替换而不破坏 core。
5. CLI/TUI/SDK/Remote host 可以只通过装配层接入，而不修改 core。
