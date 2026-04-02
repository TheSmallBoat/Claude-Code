# 基线版 Code Agent Runtime Core 总规范

## 文档状态

- 状态：Draft
- 日期：2026-04-03
- 作用：作为 Rust 基线实现的顶层蓝图，统一 `01-07` 模块规格与两份中层设计

## 1. 总目标

本总规范面向的不是 TS 代码复用，也不是 TS 到 Rust 的逐文件翻译，而是从现有工程中提炼一个：

1. 轻量化
2. 极精简
3. 高稳定
4. 高性能
5. 高质量
6. 高自主性

的 Code Agent Runtime Core 基线版本。

这个基线应明确拆成两部分：

1. 与编程无关的 Runtime Core
2. 与编程强相关的领域能力适配层

## 2. 萃取边界

### 2.1 纳入基线的内容

1. query / conversation 执行内核
2. message IR 与 transcript store
3. tool / permission / context contracts
4. compact / memory / overflow recovery 主机制
5. task / subagent / background session runtime
6. coding host adapters 的稳定边界

### 2.2 明确排除的内容

1. TUI 组件与展示层细节
2. 宠物、陪伴、装饰性产品特征
3. 特定商业实验功能的 prompt 文案
4. 各类产品命令的 UX 壳层

## 3. 顶层架构

整个系统建议分成三大块：

```text
A. Runtime Core
   - agent-message
   - agent-session-store
   - agent-permissions
   - agent-tools
   - agent-context
   - agent-query
   - agent-compact
   - agent-memory
   - agent-tasks
   - agent-subagents

B. Programming Domain Adapters
   - agent-code-adapters
   - agent-code-workflows

C. Host Assembly
   - agent-host-assembly
   - agent-host-cli / sdk / remote
```

其中：

1. A 是真正的可复用 runtime 内核。
2. B 是 coding-specific 能力包。
3. C 只负责装配、transport、UI 和策略差异。

## 4. 核心设计断言

### 4.1 消息数组与 transcript 是统一事实源

任何会话状态最终都应落回：

1. message IR
2. transcript append / load
3. compact boundary metadata

不能把真正语义散落到 UI 状态或临时对象中。

### 4.2 只有一个查询执行内核

主线程 query、subagent query、background main-session query 都必须复用同一个 `agent-query` 内核，只允许外层 envelope 不同。

### 4.3 task 是控制面，agent 是执行面

`agent-tasks` 只负责：

1. 状态可见性
2. 输出文件
3. 通知与回收

真正的 query 执行仍然归 `agent-query` 与 `agent-subagents`。

### 4.4 compact 是多阶段治理，不是单一 summary 函数

长上下文治理必须至少区分：

1. tool-result budget shrinking
2. microcompact
3. proactive compaction
4. reactive overflow recovery
5. session memory extraction / consumption

### 4.5 子代理恢复依赖 sidechain transcript，而不是 task state hydration

可恢复执行体必须拥有：

1. sidechain transcript
2. metadata
3. agent identity scope
4. resume reconstruction path

### 4.6 编程能力永远在 core 外侧

`git`、`bash`、`rg`、`.ipynb`、LSP、VSCode 都不得下沉进 runtime core 本体。

## 5. 标准执行主线

### 5.1 主线程回合主线

一次标准 turn 应按以下顺序运行：

1. context assembly
2. request working-set construction
3. model streaming
4. tool dispatch
5. tool result append
6. context governance / compaction if needed
7. stop hooks / completion

### 5.2 后台子代理主线

后台 agent 应按以下顺序运行：

1. task register
2. sidechain transcript init
3. isolated context snapshot
4. shared query kernel execution
5. terminal task state update
6. cleanup + notification

### 5.3 背景主会话主线

main-session backgrounding 的本质是：

1. 把当前主 query 降格为独立 sidechain task
2. 用隔离 transcript 继续执行
3. 保持主 transcript 不被 `/clear` 等动作污染

## 6. 状态与持久化模型

系统至少存在四类持久化对象：

1. 主 transcript
2. sidechain transcripts
3. session memory files
4. task output files

它们的职责不能混用：

1. transcript 保存消息事实。
2. sidechain transcript 保存可恢复 agent 事实。
3. session memory 保存长期会话摘要账本。
4. task output 保存大输出与增量消费接口。

## 7. 扩展模型

所有扩展必须通过正式 registry 接入：

1. Tool registry
2. Command / workflow catalog
3. Agent catalog
4. Hook registry
5. MCP registry
6. Context contributors

任何新能力若不能通过这些边界接入，就说明它还没有被抽象成熟，不应进入基线版。

## 8. 编程领域适配层的定位

编程领域适配层负责：

1. workspace reader / writer
2. search adapters
3. shell execution adapter
4. repo adapter
5. code-intel adapter
6. IDE bridge
7. workflow / skill catalog

它的作用是让 runtime core 获得“可在代码库中工作”的能力，而不是改变 runtime core 的基本控制流。

## 9. 建议的首版 Rust crate 集

首版建议最少包含：

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
13. `agent-host-assembly`

若要极限压缩首版，也可以先把：

1. `agent-compact` + `agent-memory` 合并
2. `agent-tasks` + `agent-subagents` 合并
3. `agent-code-workflows` 临时放入 host crate

但不建议把 `agent-message`、`agent-session-store`、`agent-query` 混并成一个单体 crate。

## 10. 首版实现顺序

建议按三个阶段推进：

### Phase A: 内核主链

1. `agent-message`
2. `agent-session-store`
3. `agent-permissions`
4. `agent-tools`
5. `agent-context`
6. `agent-query`

### Phase B: 长上下文与多执行体

1. `agent-compact`
2. `agent-memory`
3. `agent-tasks`
4. `agent-subagents`

### Phase C: 编程宿主与工作流

1. `agent-code-adapters`
2. `agent-code-workflows`
3. `agent-host-assembly`

## 11. 基线版必须保留的不变量

首版实现无论如何裁剪，都必须保留：

1. query / transcript 统一事实源
2. compact boundary + preserved segment 语义
3. read-before-write 与 stale detection
4. task control plane 与 execution plane 分离
5. sidechain transcript resume 机制
6. coding adapters 与 core 的分层边界

如果这些不变量被打破，即使功能还在，也已经不是同一个 runtime 架构。

## 12. 当前已知缺口

截至 `08-11` 补遗落地，原先的文本资产、task 扩展类型、高阶 coding workflows、reactive compact 等缺口已经补齐。

当前文档集已经足够支撑“基线主链 + 近邻补遗”实现。

如果继续外扩，仍建议把以下主题视作后续专题，而不是基线阻塞项：

1. context collapse 的完整恢复协议。
2. Bridge / Remote / Coordinator 的分布式运行时规格。
3. browser / container / computer-use 等平台化宿主能力。

## 13. 评审结论

在评审者视角下，这份总规范与前置文档组合后，已经能够直接回答：

1. 模块怎么拆。
2. 依赖怎么走。
3. 状态归谁管。
4. 哪些能力属于 core，哪些属于 coding adapters，哪些属于 host。

因此它已经足以作为 Rust 基线版本的开工蓝图。

但如果继续向完整扩展版外推，仍应把第 12 节列出的后续专题作为明确 backlog，而不能假设它们已经自然消失。
