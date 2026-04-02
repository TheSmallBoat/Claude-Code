# QueryEngine 与 query 主循环笔记

## 文档状态

- 状态：Working Notes
- 日期：2026-04-03
- 范围：围绕 `QueryEngine.ts`、`query.ts` 及其直接依赖，梳理 Runtime Core 的执行主链

## 1. 证据源

本笔记主要基于以下代码：

1. `TS-Source/QueryEngine.ts`
2. `TS-Source/query.ts`
3. `TS-Source/query/config.ts`
4. `TS-Source/query/deps.ts`
5. `TS-Source/query/stopHooks.ts`
6. `TS-Source/context.ts`
7. `TS-Source/utils/queryContext.ts`
8. `TS-Source/bootstrap/state.ts`
9. `TS-Source/services/tools/toolOrchestration.ts`
10. `TS-Source/services/compact/autoCompact.ts`
11. `TS-Source/services/compact/microCompact.ts`

## 2. 第一判断：系统不是“请求一次模型”而是“驱动一个回合机”

`QueryEngine.submitMessage()` 并不直接等价于一次模型请求。它更像是：

1. 组装会话级上下文。
2. 处理用户输入及本地命令。
3. 持久化用户侧事实。
4. 构造 `ToolUseContext`。
5. 调用 `query()` 进入单回合执行机。
6. 持续消费 `query()` 产生的消息流并写回会话状态。

因此可以先作一个关键拆分：

1. `QueryEngine` 是会话级 orchestrator。
2. `query()` 是单回合 turn executor。

这两层拆分对 Rust 设计非常重要，意味着后续不应该只实现一个“大而全”的 runtime 对象。

## 3. QueryEngine 的职责判断

从 `submitMessage()` 的调用链看，`QueryEngine` 至少承担以下职责：

1. 持有会话级消息数组 `mutableMessages`。
2. 持有会话级累计 usage、permission denials、read file cache。
3. 组装基础 prompt：`fetchSystemPromptParts()` + custom prompt + memory mechanics prompt。
4. 调用 `processUserInput()` 处理 slash command、附件、工具允许列表、模型切换等前置逻辑。
5. 在进入 `query()` 前先把用户消息写入 transcript，以保证中途进程退出时仍可 resume。
6. 作为 `query()` 的上层，消费流式事件并同步更新本地消息状态、transcript、SDK 输出。

这说明在 Rust 中，`ConversationEngine` 不能只是 `Vec<Message>` 的包装器，它还要同时维护：

1. 会话缓存。
2. 转录写入策略。
3. 会话级累计 usage。
4. 当前回合的宿主接口句柄。

## 4. ToolUseContext 的本质判断

`ToolUseContext` 不是普通参数对象，而是运行时环境句柄集合。其字段包括：

1. 工具与命令注册表。
2. `getAppState()` / `setAppState()`。
3. `abortController`。
4. read file cache。
5. 进度、通知、SDK 状态输出接口。
6. 当前消息数组。
7. 权限上下文、查询链追踪、toolUseId、agentId 等环境信息。

因此，Rust 中不应把它设计成纯只读配置，而应拆成两部分：

1. `ExecutionContext`：稳定的环境句柄与依赖访问器。
2. `TurnMutableState`：消息、追踪、上下文突变等每回合可变部分。

## 5. query() 主循环的阶段顺序

`query.ts` 中的 while-loop 不是杂糅逻辑，而是一个非常明确的阶段机。按当前代码证据，可以归纳为：

### 5.1 初始化阶段

1. 解构不可变输入：`systemPrompt`、`userContext`、`systemContext`、`canUseTool`、`fallbackModel` 等。
2. 建立可变 `State`，字段包括：
   - `messages`
   - `toolUseContext`
   - `autoCompactTracking`
   - `maxOutputTokensRecoveryCount`
   - `hasAttemptedReactiveCompact`
   - `maxOutputTokensOverride`
   - `pendingToolUseSummary`
   - `stopHookActive`
   - `turnCount`
   - `transition`
3. 只在回合入口构造一次 `QueryConfig`，对部分 gate 做快照。
4. 启动相关记忆预取 `startRelevantMemoryPrefetch()`。

判断：这是明显的“回合局部状态”模型，适合在 Rust 中做 `TurnState` 结构体。

### 5.2 预处理上下文阶段

进入每次迭代后，消息会按以下顺序变换：

1. 从 compact boundary 后截取有效消息视图。
2. 应用 tool result budget，对过大的工具结果做内容替换。
3. 运行 snip compact。
4. 运行 microcompact。
5. 运行 context collapse。
6. 运行 autocompact。

顺序非常关键，不能随意交换，因为每一步针对的目标不同：

1. tool result budget 先做体积治理。
2. snip / microcompact 做轻量化压缩。
3. context collapse 尝试保留更细粒度上下文。
4. autocompact 作为较重的总结型压缩。

这说明 Rust 版需要一个明确的 `CompactionPipeline`，而不是把 compact 分散到模型调用前后各处。

## 6. 模型采样阶段

模型调用由 `deps.callModel()` 完成，`query()` 本身不耦合具体 API 实现；`query/deps.ts` 暴露了有限依赖注入点：

1. `callModel`
2. `microcompact`
3. `autocompact`
4. `uuid`

这说明作者已经在往“可测试 turn executor”方向收拢。Rust 版本应进一步做实：

1. `ModelDriver`
2. `CompactionDriver`
3. `IdGenerator`
4. `TimeSource`

### 6.1 流式采样的重要行为

流式采样过程中，`query()` 不是单纯转发 stream event，而是实时执行以下逻辑：

1. 追踪 assistant message。
2. 收集 `tool_use` block。
3. 可选地启用 `StreamingToolExecutor`，在模型仍在流式输出时并发跑工具。
4. 对可恢复错误做“暂不对外暴露”的 withheld 处理。

这意味着 Rust 版必须把“模型流式事件处理”和“执行状态转移”放在一个统一 reducer 中，而不是简单的事件回调拼接。

## 7. 恢复路径的顺序性是内核竞争力之一

从 `query.ts` 的错误与恢复分支看，当前实现不是“报错就结束”，而是有层级分明的恢复链：

### 7.1 Prompt-too-long 恢复链

顺序是：

1. 如果启用了 context collapse，先尝试 drain staged collapses。
2. 如果仍不行，再尝试 reactive compact。
3. 如果还不行，才把 withheld 的错误真正抛给上层。

这是一个很值得继承的设计：

1. 先尝试最保守、最便宜的恢复。
2. 再做更重的总结型恢复。
3. 最后才失败返回。

### 7.2 max_output_tokens 恢复链

顺序是：

1. 若使用默认 8k 上限，则先尝试提升到更高输出上限再重试。
2. 之后再通过 meta user message 提示模型“直接续写，不要道歉和回顾”。
3. 最多恢复三次。

判断：这是明显的 continuation policy，不应埋在宿主层。

### 7.3 token budget 不是硬停止，而是软引导继续

`query/tokenBudget.ts` 的逻辑显示：

1. 对主线程有效，对 subagent 无效。
2. 在预算尚未接近上限时，自动插入 meta nudge message，促使模型继续。
3. 连续多次继续且增量收益过低时，提前停止。

这不是普通的 token counter，而是“完成度驱动的软控制器”。

## 8. 工具执行阶段的关键语义

`services/tools/toolOrchestration.ts` 暴露出两个强约束：

1. 先按 `isConcurrencySafe()` 对工具调用分批。
2. 并发批只允许读类工具，且上下文突变会被延后到批次完成后统一应用。
3. 非并发安全工具严格串行执行。

这意味着工具系统真正重要的，不是“调用一个函数”，而是：

1. 工具副作用分类。
2. 调度计划生成。
3. 上下文突变提交点。
4. 中断与收尾策略。

这部分将直接影响 Rust 的 scheduler 设计。

## 9. 停止钩子不是外围插件，而是 query 内核的标准阶段

`query/stopHooks.ts` 表明 stop hooks 的地位很高：

1. 在模型完成且没有后续工具时执行。
2. 可写入 blocking errors，反向把回合送入下一次循环。
3. 可阻止 continuation。
4. 同时还承载 prompt suggestion、extract memories、auto dream 等后台行为触发。

这说明 stop hooks 不应在 Rust 中被建模为“可有可无的事件监听器”，而应是：

1. 一个有阻塞能力的生命周期阶段。
2. 一个会向消息流注入新消息的执行器。
3. 一个承载 turn-end 自动化的挂载点。

## 10. 会话持久化策略的要点

从 `QueryEngine.ts` 看，当前系统对 transcript 的处理有几个关键不变量：

1. 用户输入在首次模型调用前就要落盘。
2. assistant 消息的写入可 fire-and-forget，但顺序必须保持。
3. compact boundary 写入前要先把相关尾段补齐，否则 resume 会错。
4. progress/attachment 也需要进入 transcript，以保证下一次恢复与去重逻辑一致。

因此 Rust 基线版至少要支持：

1. 顺序追加。
2. 可显式 flush。
3. compact 边界前的补写。
4. resume 时基于 transcript 的一致性恢复。

## 11. 当前对 Rust 设计的抽象结论

目前可以先把 TS 实现还原成如下结构：

1. `ConversationEngine`
   - 维护会话历史、usage、cache、transcript sink。
2. `TurnExecutor`
   - 维护当前 turn state 与 while-loop。
3. `ContextAssembler`
   - 负责 prompt、user/system context、memory、cache-safe prefix。
4. `CompactionPipeline`
   - 负责 snip、microcompact、collapse、autocompact、reactive compact。
5. `ToolScheduler`
   - 负责工具分批、并发/串行、上下文突变提交。
6. `StopHookRuntime`
   - 负责 stop-hook 执行与 turn-end 自动化。
7. `TranscriptStore`
   - 负责持久化和恢复。

## 12. 暂未覆盖但必须继续补齐的点

当前还没有完全吃透以下内容：

1. `types/message.ts` 中消息 IR 的完整结构。
2. `utils/permissions/**` 的权限评估细则。
3. `utils/forkedAgent.ts` 的子代理隔离与 cache sharing 机制。
4. `services/SessionMemory/**` 与 `services/compact/**` 的更细粒度耦合关系。
5. 编程领域工具的统一副作用分类和 host contract。

因此本笔记可以支撑第一份查询执行规格书，但还不足以单独支撑完整 Runtime Core 实现。
