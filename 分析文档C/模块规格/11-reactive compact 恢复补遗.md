# reactive compact 恢复补遗

## 文档状态

- 状态：Supplement Draft
- 日期：2026-04-03
- 对应 Rust 目标模块：`agent-compact`、`agent-query`、`agent-session-store`

## 1. 目标

补足 `05-压缩与记忆子系统规格` 中仍停留在接口级的 reactive compact 部分，明确 Rust 基线版必须继承的恢复协议：

1. 真实 overflow 的触发契约。
2. context collapse 与 reactive compact 的交接顺序。
3. reactive compact 成功后的 query state、transcript 与 resume 不变量。
4. 在内部实现缺失的情况下，如何为 Rust 版定义稳定接口而不过拟合恢复树中的 stub。

## 2. 证据源

本补遗主要基于以下代码：

1. `TS-Source/query.ts`
2. `TS-Source/commands/compact/compact.ts`
3. `TS-Source/services/compact/autoCompact.ts`
4. `TS-Source/services/compact/compact.ts`
5. `TS-Source/services/compact/postCompactCleanup.ts`
6. `TS-Source/services/compact/reactiveCompact.ts`
7. `TS-Source/utils/sessionStorage.ts`
8. `TS-Source/QueryEngine.ts`
9. `TS-Source/remote/sdkMessageAdapter.ts`
10. `TS-Source/remote/SessionsWebSocket.ts`
11. `TS-Source/entrypoints/sdk/coreSchemas.ts`
12. `TS-Source/services/contextCollapse/index.ts`

## 3. 第一判断：已恢复语义主要分布在 query 与 transcript 协议层

当前恢复树里，`services/compact/reactiveCompact.ts` 只有 stub，本身并不能提供实现级细节。但外围调用点已经恢复出足够多的稳定语义：

1. `query.ts` 给出了 withheld overflow 的识别、恢复顺序、单次尝试限制与 retry transition。
2. `/compact` 命令给出了 reactive-only 模式的人工入口、failure reason 映射与 hook 责任分工。
3. `compact.ts` 给出了 `CompactionResult`、boundary metadata 与 post-compact message 顺序。
4. `QueryEngine.ts` 与 `sessionStorage.ts` 给出了 preserved segment 的落盘、relink 与 resume 约束。

因此，Rust 版不应把 reactive compact 当成“某个尚未恢复出的摘要算法”，而应把它视作一个跨 query、compact result、transcript persistence、resume 的 overflow recovery protocol。

## 4. 触发契约

### 4.1 只接受真实 API overflow

reactive compact 不应响应本地的 token 预阻断，而应只在真实模型调用失败后启动。

这从 `query.ts` 的注释可以明确看出：

1. 当 reactive compact 或 context collapse 需要接管 overflow recovery 时，query loop 会跳过 synthetic preempt。
2. 这样真实 API 的 `prompt-too-long` 才能返回到 streaming loop。
3. streaming loop 再把该错误暂时 withheld，留给 overflow recovery coordinator 决定是否恢复。

### 4.2 至少存在两类可恢复溢出

当前证据明确区分了两类 recoverable overflow：

1. `prompt-too-long` / 413。
2. media-size rejection，例如 oversized image、PDF、过多图片等。

这两类错误都不是普通业务错误，而是上下文治理子系统要尝试接管的 overflow。

### 4.3 withholding 与 recovery gate 必须一致

`query.ts` 把 `mediaRecoveryEnabled` 在 turn 开始阶段 hoist 一次，并要求 streaming loop 的 withholding 和 post-stream recovery 共用同一值。

这条约束必须保留，因为如果：

1. 错误被 withheld。
2. 恢复 gate 在 5-30 秒的 streaming 期间翻转。
3. recovery 分支不再执行。

就会出现“错误消息被吞掉且没有恢复动作”的状态破坏。

### 4.4 manual `/compact` 是独立入口

reactive compact 不只服务于 query loop。`commands/compact/compact.ts` 说明：

1. 当 `isReactiveOnlyMode()` 为真时，手动 `/compact` 会直接走 reactive path。
2. 该路径不依赖主 query loop 中的 withheld 机制。
3. 但它仍然必须返回标准 `CompactionResult`，并执行统一 cleanup。

## 5. 恢复顺序

### 5.1 prompt-too-long：先 collapse drain，再 reactive compact

对 413 类错误，`query.ts` 的真实顺序是：

1. 先判断是否存在被 withheld 的 prompt-too-long assistant error。
2. 若开启 context collapse，且前一跳 transition 不是 `collapse_drain_retry`，先调用 `recoverFromOverflow()`。
3. 如果 collapse drain 确实提交了 staged collapses，则直接把新的 `messages` 写回 state，并用 `collapse_drain_retry` 作为 transition reason 继续主循环。
4. 只有在 drain 没有效果，或者 drain 后重试仍然 413 时，才进入 reactive compact。

这说明 reactive compact 不是第一道恢复手段，而是 cheaper recovery path 失败后的第二层恢复。

### 5.2 media-size：跳过 collapse，直接走 reactive path

源码注释已明确说明：context collapse 不会 strip image，因此 media overflow 不该先走 collapse drain。

因此必须保持：

1. media-size error 不经过 collapse drain。
2. 直接交给 reactive compact 做 strip-retry 或宣告不可恢复。

### 5.3 proactive 与 reactive 不能互相饿死

`autoCompact.ts` 的 gating 证明 reactive compact 与 proactive autocompact 必须解耦：

1. reactive-only mode 下，`shouldAutoCompact()` 会 suppress proactive autocompact。
2. context collapse mode 下，也会 suppress proactive autocompact，避免在 90% commit 与 95% blocking 之间和 collapse 竞争。
3. 这样 query loop 才能等到真实 413，再由 reactive recovery 接管。

## 6. 单次尝试与反死循环语义

`query.ts` 已经把反死循环语义编码进 query state：

1. state 必须携带 `hasAttemptedReactiveCompact`。
2. 同一轮 overflow recovery 至多进行一次 reactive compact 重试。
3. 成功后必须把该标志置为 `true`，防止 preserved tail 仍含 oversized media 时无限重试。
4. 如果 recovery 失败，则直接 surface withheld error 并结束该 turn。
5. 失败路径不能再回到常规 stop-hook blocking / retry 分支，否则会形成“overflow error -> hook 注入更多 tokens -> 再次 overflow”的死循环。
6. 可以执行 failure hook side effect，但不能把失败 turn 当成正常 assistant response 继续推进状态机。

## 7. 驱动接口契约

### 7.1 query path 需要的最小能力

从 `query.ts` 可还原出 reactive driver 的最低接口：

1. `isWithheldPromptTooLong(message) -> bool`
2. `isWithheldMediaSizeError(message) -> bool`
3. `tryReactiveCompact(request) -> Option<CompactionResult>`
4. `isReactiveOnlyMode() -> bool`

### 7.2 `tryReactiveCompact()` 的请求形状

`query.ts` 传给 `tryReactiveCompact()` 的参数说明，请求对象至少必须包含：

1. `hasAttempted`
2. `querySource`
3. `aborted`
4. `messages`
5. `cacheSafeParams { systemPrompt, userContext, systemContext, toolUseContext, forkContextMessages }`

其中 `cacheSafeParams` 不是附属优化，而是 reactive compact 与主 system prompt / user context / current tool registry 保持一致的必要输入。

### 7.3 manual `/compact` 入口还需要 failure reason

`reactiveCompactOnPromptTooLong()` 的调用说明 reactive driver 在手动 compaction 路径中还需要：

1. 接收 merged custom instructions。
2. 区分 `trigger: 'manual'`。
3. 通过 `ok/result` 或 `ok=false/reason` 报告结果。

从该调用点至少可以确认以下 failure reason 需要保留：

1. `too_few_groups`
2. `aborted`
3. `exhausted`
4. `error`
5. `media_unstrippable`

这说明 Rust 版不能把 reactive compact 的失败统一压扁成单个 `false`，至少在手动入口需要保留分类原因。

## 8. 成功后的统一行为

reactive compact 一旦成功，必须与其他 compaction 路径一样，进入统一的 post-compact 语义：

1. 使用 `buildPostCompactMessages()` 产出稳定顺序：boundary、summary、messagesToKeep、attachments、hookResults。
2. query path 必须把这些 post-compact messages 全量 yield 回上层。
3. state 必须更新为 `messages = postCompactMessages`。
4. `autoCompactTracking` 必须清空，因为这已经不是原先那条 proactive compact tracking 链。
5. `hasAttemptedReactiveCompact` 必须置为 `true`。
6. `transition.reason` 必须记录为 `reactive_compact_retry`。
7. `maxOutputTokensOverride`、`pendingToolUseSummary`、`stopHookActive` 都必须清空。
8. 如果启用了 `taskBudget`，还必须把 pre-compact final context 从 remaining budget 中扣除。

### 8.1 cleanup 与状态清理必须复用统一通道

从 `/compact` 的 reactive-only 路径和其他 compact 路径可确认，reactive compact 成功后至少要做：

1. `setLastSummarizedMessageId(undefined)`
2. `runPostCompactCleanup(querySource?)`
3. `suppressCompactWarning()`
4. `getUserContext.cache.clear?.()`

也就是说，reactive compact 不是可以绕开 cleanup 的旁路。只要成功替换了消息数组，它就必须进入与 full compact、session-memory compact 相同的 shared-state cleanup 体系。

### 8.2 PreCompact / PostCompact hook 的责任边界是分裂的

`commands/compact/compact.ts` 给出一条很重要的实现线索：

1. 外层调用者负责运行 PreCompact hooks，并与 custom instructions 合并。
2. reactive driver 负责完成 reactive compaction 本身，并产出 PostCompact hooks 已生效后的 `CompactionResult`。
3. 外层再把两侧的 userDisplayMessage 合并。

这意味着 Rust 版可以把“预处理上下文”和“overflow recovery driver”拆成两个协作对象，而不必把所有 hook 逻辑都塞进 reactive driver 内部。

## 9. transcript 与 resume 契约

### 9.1 compact boundary 是恢复协议的一部分

reactive compact 不是只要换掉内存里的消息数组就结束。若存在 preserved tail，boundary metadata 就成为恢复协议的一部分。

必须保留：

1. `preservedSegment.headUuid`
2. `preservedSegment.anchorUuid`
3. `preservedSegment.tailUuid`

loader 再基于这些字段把 preserved tail 在内存里 splice 回正确位置。

### 9.2 落盘顺序必须先 flush `tailUuid`

`QueryEngine.ts` 的注释明确说明：在写 compact boundary 之前，必须先把 `tailUuid` 之前尚未持久化的消息全部写入 transcript。

否则一旦：

1. SDK 子进程在 boundary 写完后、下一轮开始前被杀掉。
2. transcript 中缺少 preserved tail 的实际消息。
3. `applyPreservedSegmentRelinks()` 的 tail-to-head walk 断裂。

resume 就会回退为“加载完整 pre-compact history”，而不是正确恢复 compact 后的工作集。

### 9.3 relink 只能对最后一个 live seg-boundary 生效

`sessionStorage.ts` 的 `applyPreservedSegmentRelinks()` 说明：

1. 只对 absolute-last 且仍 live 的 seg-boundary 做 relink。
2. 如果后面又出现了不带 preserved segment 的 boundary，旧 seg 会被视为 stale，不再 relink。
3. relink 前必须先验证 tail-to-head walk 完整，验证失败时直接 no-op。
4. 这种 no-op 会跳过 prune，让 resume 加载完整 pre-compact history，而不是产生一条破损链。

这是一条非常关键的容错语义：metadata 损坏时，宁可退回“历史更多”，也不能生成损坏 transcript。

### 9.4 有 preserved segment 时不能做 pre-parse chain-walk 优化

`sessionStorage.ts` 还明确禁用了一个优化：当 transcript 含 preserved segment 时，不能在 parse 前用简单 chain walk 丢弃 pre-boundary 内容。

原因是：

1. preserved messages 在磁盘上仍然保留旧的 `parentUuid`。
2. 它们必须先完整 parse 进内存。
3. 再由 `applyPreservedSegmentRelinks()` 在内存中修补链关系。

因此 preserved segment 不只是 boundary metadata 字段，它会直接改变 transcript loader 的优化策略。

### 9.5 compact boundary 必须被所有宿主视为一等消息

当前代码已经把 compact boundary 作为一等消息类型处理：

1. SDK schema 单列 `compact_boundary`。
2. remote adapter 会把 SDK compact boundary 转回本地 `system` message，并保留 compact metadata。
3. `SessionsWebSocket` 对 compaction 期间短暂的 `4001 session not found` 有专门容错。

这说明 reactive compact 的恢复协议不能只存在于 CLI 内部，还必须能通过 SDK / remote transport 稳定传播。

## 10. reactive-only mode 的语义

当前代码明确区分了 proactive compaction 与 reactive overflow recovery：

1. reactive-only mode 不是“禁用 compact”，而是“不要预先 compact，等真实 overflow 再恢复”。
2. 这会 suppress 主 query loop 中的 proactive autocompact，也就不会先尝试 session-memory-first 那条 autocompact 路径。
3. manual `/compact` 仍然保留，并可内部路由到 reactive path。

所以 Rust 版应把下面两层拆开，而不是合并成一个 `compact_if_needed()`：

1. `ProactiveCompactor`
2. `ReactiveOverflowRecovery`

## 11. Rust 建模建议

建议把 reactive compact 建模为一个明确定义输入、输出和失败原因的 driver：

```rust
pub enum OverflowKind {
    PromptTooLong,
    MediaTooLarge,
}

pub struct ReactiveCompactRequest {
    pub kind: OverflowKind,
    pub has_attempted: bool,
    pub query_source: QuerySource,
    pub aborted: bool,
    pub messages: Vec<RuntimeMessage>,
    pub cache_safe_params: CacheSafePromptParams,
}

pub enum ReactiveCompactFailure {
    TooFewGroups,
    Aborted,
    Exhausted,
    Error,
    MediaUnstrippable,
}

pub enum ReactiveCompactOutcome {
    Recovered(CompactionResult),
    Skipped,
    Failed(ReactiveCompactFailure),
}

pub trait ReactiveCompactDriver {
    fn detect_withheld(&self, msg: &AssistantMessage) -> Option<OverflowKind>;
    fn reactive_only_mode(&self) -> bool;
    async fn recover(
        &self,
        req: ReactiveCompactRequest,
    ) -> ReactiveCompactOutcome;
}
```

同时建议把 query 层与 transcript 层的职责拆开：

1. `agent-query` 负责检测 overflow、协调 collapse drain、调用 reactive driver、设置 retry transition。
2. `agent-compact` 负责生成标准 `CompactionResult`。
3. `agent-session-store` 负责 compact boundary、preserved segment relink 与 resume 容错。

### 11.1 对缺失实现的正确态度

由于恢复树没有提供真实 `reactiveCompact.ts` 实现，Rust 版最合理的策略是：

1. 先完整实现这里已经恢复出的协议级不变量。
2. 允许 `ReactiveCompactDriver` 的首版实现委托到保守的 full compact fallback。
3. 后续如果补齐了更接近原版的 strip-retry / group-peeling 算法，再作为 driver 的内部策略替换。

这样既不会阻塞基线实现，也不会把一个缺失的 TS 细节硬编码进架构层。

## 12. 最小测试矩阵

至少应覆盖以下测试：

1. 真实 prompt-too-long 先走 collapse drain，失败后才进入 reactive compact。
2. media-size overflow 不会尝试 collapse drain，而是直接进入 reactive recovery。
3. 同一轮最多只允许一次 reactive compact 重试。
4. reactive compact 失败时会 surface withheld error，并绕开常规 stop-hook retry 链。
5. reactive compact 成功后会产出标准 post-compact message 顺序，并设置 `reactive_compact_retry` transition。
6. compact boundary 写入前会先 flush 到 `tailUuid`，避免 preserved segment relink 断裂。
7. tail-to-head walk 断裂时，loader 会放弃 prune，退回加载完整 pre-compact history。
8. reactive-only mode 会 suppress proactive autocompact，但 manual `/compact` 仍然可用。
9. SDK / remote transport 对 `compact_boundary` 的编码与解码保持字段完整。

## 13. 结论

当前恢复树已经足够证明：reactive compact 不是一个孤立的 compact helper，而是一条跨 query、compaction、transcript persistence、resume 的 overflow recovery 主协议。

Rust 基线版真正要复现的，不是某个缺失的 TS 内部实现细节，而是这条协议的稳定不变量：

1. 只对真实 overflow 生效。
2. 与 context collapse 有清晰交接顺序。
3. 只允许单次重试并避免死循环。
4. 成功后必须回到统一的 `CompactionResult` 与 boundary/resume 契约。

做到这四点，Reactive Compact 就已经从“接口级占位”提升为“可实现的 Rust 基线恢复模块”。
