# 第 25 章 harness：一个 turn 里的模型与工具循环

> 一句话结论：eve 不用 AI SDK 的自动循环。每次模型调用是一个 step，工具执行完就结束，下一轮模型调用是新 step。这就是「每步一次模型调用」的全部含义。

## 三个 step.started

你问 agent 一句「现在几点了」。看事件流，`step.started` 出现了三次：

```text
step.started          ← 第 1 次
actions.requested     get_current_time
action.result         ...
step.completed
step.started          ← 第 2 次
message.appended      "现在是下午三点二十。"
step.completed
```

问题：一次问答，为什么要调三次模型？

答案是两次。第三次是最后那个空转的 step（用来收尾）。但重点是：**每调一次模型就是一个 step**，不是「一个 turn 一个 step」。

第 24 章讲过 step 是检查点边界。这一章拆开 step 内部，看它到底跑了什么，以及为什么 eve 要放弃 AI SDK 的自动循环，自己管这件事。

## 25.1 一个 step 的七个阶段

```ts
// packages/eve/src/harness/tool-loop.ts · executeStepBody()
async function executeStepBody(initialSession, generation, live, input, instrumentation) {
  const prepareHistory = createHistoryViewPreparer({ previous: config.historyView, projector: config.historyProjector });
  prepareHistory(initialSession.history, initialSession.state);
  // ...
  if (config.clearOnly === true) return clearContext(step);
  if (config.compactOnly === true) return compactHistory(step);

  const runtime = await settleRuntimeWork(step, input);
  if (runtime === undefined) return { next: null, session: step.session };
  const intake = await acceptHumanInput(step, runtime.input, { takeQueued: !runtime.waited });
  if (intake.kind === "stop") return intake.result;
  const turn = await prepareTurnInput(step, intake.input, { consumedMessage: intake.consumedMessage });
  if (intake.opensTurn) {
    const failed = await openTurn(step, { input: [...turn.ephemeral, ...turn.messages], message: intake.message });
    if (failed !== undefined) return failed;
  }
  await admitApprovedWork(step, intake.approved);

  return runModelStep(step, {
    approved: intake.approved,
    onResponse: (response) => handleStepResult(step, response),
    generation,
    hidesHeldText: step.hasDelegatedCaller || ctx?.get(ScheduleIdKey) !== undefined,
    setAttemptScope: (scope) => { attemptScope = scope; },
    turn,
  });
}
```

| 阶段 | 函数 | 做什么 | 可能提前返回 |
| --- | --- | --- | --- |
| 0 | `clearContext` / `compactHistory` | `session.clear()` / `compact()` 的专用路径 | 是 |
| 1 | `settleRuntimeWork` | 结算上一个 step 停泊的工作 | 是（返回 `undefined` 表示无事可做） |
| 2 | `acceptHumanInput` | 接收审批、问题回答、排队消息 | 是（`kind === "stop"`） |
| 3 | `prepareTurnInput` | 把输入转成消息，处理临时消息 | 否 |
| 4 | `openTurn` | 需要时开新 turn | 是（失败时返回错误结果） |
| 5 | `admitApprovedWork` | 执行已批准的工作 | 否 |
| 6 | `runModelStep` | 调一次模型，处理响应 | 否 |

第 0 阶段的短路很有意思：`clearOnly` 和 `compactOnly` 是**独立的 step**，不夹在正常流程里。所以 `POST /session/:id/clear` 不会消耗一次模型调用。

## 25.2 为什么每步只调一次模型

AI SDK 的 `ToolLoopAgent` 默认会一直循环：调模型 → 执行工具 → 再调模型 → …… 直到模型不再要求调工具。

eve 把它关掉了：

```ts
// packages/eve/src/harness/model-call/call.ts:393
const agent = new ToolLoopAgent({
  // ...
  onStepEnd: hooks.onStepEnd,
  onStepStart: hooks.onStepStart,
  prepareStep: hooks.prepareStep,
  reasoning: step.session.agent.modelReference?.reasoning ?? step.session.agent.reasoning,
  runtimeContext,
  stopWhen: isStepCount(1),
  telemetry: toEntryTelemetry(attempt?.telemetry, catalog.resolve),
  timeout: { chunkMs: MODEL_OUTPUT_TIMEOUT_MS, firstChunkMs: MODEL_OUTPUT_TIMEOUT_MS },
  toolApproval: buildToolApproval({ ... }),
  tools,
});
```

`stopWhen: isStepCount(1)` —— 跑一步就停。

对比一下两种设计：

| | AI SDK 默认 | eve |
| --- | --- | --- |
| 一次 `stream()` 调用 | 循环到结束 | 只跑一步 |
| 模型调用次数 | N | 每步 1 次 |
| 检查点 | 无（在内存里） | 每步一个 |
| 崩溃恢复 | 整个 turn 重来 | 最多重跑一个模型调用 |
| 事件粒度 | 只有最终结果 | 每次调用都有 `step.started` / `step.completed` |

eve 用「更多 step」换「更细的恢复粒度」。代价是每个 step 都要序列化一次会话状态——第 24 章的 `projectToDurableSession` 每步都跑。

这个交换在长任务上划算：一个跑了 20 轮的 turn，崩溃后最多损失一轮。在短任务上不划算：一次问答也要写一次检查点。

`experimental.workflow.modelCallsPerStep` 就是给这个交换调参的（附录 B，默认 1）。改成 2 会让一次 `stream()` 跑两步，检查点变少，恢复变粗。

## 25.3 工具出错时会发生什么

模型返回的响应不一定合法。工具参数不符合 schema 是常事。

eve 的处理是：**不抛异常，合成一条 tool error 消息塞回历史，让模型自己纠正。**

```ts
// packages/eve/src/harness/step/after-model.ts:73
export async function handleStepResult(step: Step, input: ModelResponse): Promise<StepResult> {
  const { promptMessages, result } = input;
  const stepOutput = resolveAssistantStepText(result.response.messages, result.text);
  const invalidInputToolErrors = getInvalidToolCallInputErrors({
    toolCalls: result.toolCalls as TypedToolCall<ToolSet>[],
  });
  // Unions every invalid-input signal: SDK-marked invalid calls (which get
  // SDK-synthesized tool errors), non-object inputs caught by
  // getInvalidToolCallInputErrors, and ids the stream consumer observed.
  const invalidInputToolCallIds = new Set([
    ...(result.invalidInputToolCallIds ?? []),
    ...result.toolCalls.filter(isInvalidToolCall).map((toolCall) => toolCall.toolCallId),
    ...invalidInputToolErrors.map((toolError) => toolError.toolCallId),
  ]);
  const rawResponseMessages = appendMissingToolResultMessages({
    append: invalidInputToolErrors.map((toolError) => createToolResultMessagePartFromToolError(toolError)),
    responseMessages: result.response.messages,
  });
  // ...
}
```

三个来源的无效调用被合并：

1. AI SDK 自己标记的（`result.invalidInputToolCallIds`）
2. `getInvalidToolCallInputErrors` 检查出的（输入不是对象之类）
3. 流式消费时观察到的

合并后统一补一条 tool result 消息，`isError: true`。模型下一轮看到这条错误，自己改参数重试。

这意味着：**工具参数错误不会让 turn 失败。** 它会被模型当成反馈消化掉。你在日志里看到的是 `action.result` 带 `isError: true`，不是 `turn.failed`。

这个设计有代价：如果模型一直改不对，它会反复重试，消耗 token。所以工具的描述和 schema 要写清楚，别指望模型从错误里猜出正确用法。

另一类工具是 `providerExecuted`：

```ts
const providerExecutedOutcomeIds = new Set<string>();
for (const part of [...(result.content ?? []), ...(result.toolResults ?? [])]) {
  if ((part.type === "tool-result" || part.type === "tool-error") && part.providerExecuted === true) {
    providerExecutedOutcomeIds.add(part.toolCallId);
  }
}
```

这类工具（web search 之类）由模型厂商执行，结果直接从响应里来，eve 不执行也不记录耗时。它们对应第 23 章的 `hasExecute: false`。

## 25.4 压缩发生在模型调用之前

不是「历史太长就压缩」，是「调模型前量一下，超了就压缩，然后重新构造请求」：

```ts
// packages/eve/src/harness/model-call/call.ts
/** Compacts the prompt when it's over the threshold, then rebuilds what depends on it. */
private async compact(options: ModelCallOptions, tools: ToolSet): Promise<ToolSet> {
  const { step, prompt } = this;
  const { config } = step;
  let compaction: Awaited<ReturnType<typeof maybeCompact>>;
  try {
    compaction = await maybeCompact({
      abortSignal: config.abortSignal,
      auth: step.ctx?.get(AuthKey) ?? null,
      emissionState: step.position(),
      messages: [...prompt.messages],
      model: this.input.model,
      promptMessages: withClientContext(prompt),
      publish: step.publish,
      requestEnvelopeTokens: this.requestEnvelopeTokens,
      // ...
    });
    if (compaction.failure !== undefined) throw compaction.failure.error;
  } catch (error) {
    this.compactionFailure = { error };
    throw error;
  }
  step.session = compaction.session;
  if (!compaction.compacted) return tools;
  compactPrompt(step, prompt, compaction.messages);
  // ... 重建工具集
}
```

判定函数在 `compaction/engine.ts`：

```ts
export function shouldCompact(
  messages: readonly ModelMessage[],
  config: CompactionConfig,
  requestEnvelopeTokens = 0,
  previousEnvelopeTokens = 0,
): boolean {
  return (
    messages.length > 0 &&
    getInputTokenCount(messages, config, requestEnvelopeTokens, previousEnvelopeTokens) +
      COMPACTION_PROMPT_OVERHEAD_TOKENS >
      config.threshold
  );
}
```

注意 `+ COMPACTION_PROMPT_OVERHEAD_TOKENS`：压缩本身要消耗 token（要调压缩模型生成摘要），所以阈值判断里预留了这部分。不预留的话，会在「刚好卡在阈值下」时反复触发压缩但压不下去。

`config.threshold` 由 `thresholdPercent` 乘上下文窗口得到，默认 0.9（`session.ts` 里的 `DEFAULT_COMPACTION_THRESHOLD_PERCENT`，附录 B 也有）。

压缩完成后，`compactPrompt` 替换 prompt 里的消息，`tools` 也要重建——因为工具列表里可能包含依赖历史的 deferred 工具。

**压缩发生在模型调用之前，不是之后。** 所以事件顺序是：

```text
step.started
compaction.requested
compaction.completed
message.appended      ← 压缩后的第一次模型输出
step.completed
```

## 25.5 停泊与续跑

`turn.waiting` 不是「模型在思考」，是「这个 turn 主动停下了，等外部输入」：

```ts
const runtime = await settleRuntimeWork(step, input);
if (runtime === undefined) return { next: null, session: step.session };
const intake = await acceptHumanInput(step, runtime.input, { takeQueued: !runtime.waited });
if (intake.kind === "stop") return intake.result;
```

`settleRuntimeWork` 返回 `undefined` 表示这个 step 没有可做的事——比如会话还在等审批，而这次 step 是定时器触发的。这时 step 直接返回，不消耗模型调用。

`acceptHumanInput` 的 `takeQueued: !runtime.waited` 决定要不要消费排队的消息：如果上一次是「等输入」，这次有新消息就取；如果不是，就跳过队列。

停泊状态存在会话的 `turn` 字段里，`turn.waiting` 事件的 `on` 字段说明在等什么：

- `on: "input"` —— 等审批或问题回答
- `on: "tasks"` —— 等子代理任务

第 11 章讲的四种审批策略，最终都落在这里：策略决定的是「停下时状态怎么记」，续跑路径只有这一条。

## 25.6 回到那三个 step.started

现在能解释了。

「现在几点了」需要调一个工具，所以：

1. **第一个 step**：调模型，模型说「我要用 `get_current_time`」。工具执行完，step 结束，写检查点。
2. **第二个 step**：调模型，模型看到工具结果，说「现在是下午三点二十」。没有更多工具调用，step 结束。
3. **第三个 step**：收尾（发 `session.waiting`、结算停泊工作），不调模型或只调一次空转。

如果模型一次就回答，只有两个 step。如果它连续调三个工具，就是四个。

所以「step 数」约等于「模型调用数 + 收尾」。看到事件流里 step 很多，说明模型在反复调工具，不是框架有问题。

## 25.7 五个坑

1. **工具参数错误不会让 turn 失败。** 它变成 `isError: true` 的 tool result，模型自己重试。想看真实失败，查 `turn.failed`，不是 `action.result`。
2. **一个 turn 的 step 数 = 模型调用数 + 辅助 step。** 20 轮工具调用就是 20 个 step，每个都写检查点。长任务的开销主要在序列化上。
3. **压缩在模型调用前，且会重建工具集。** 自定义工具的 schema 如果依赖历史，压缩后要能重新生成。
4. **`providerExecuted` 的工具没有本地耗时。** 追踪里看不到它们的执行时间，因为执行发生在厂商那边。
5. **`settleRuntimeWork` 返回 `undefined` 时 step 不消耗模型调用。** 如果你的定时任务看起来「跑了但没反应」，先确认它是不是走了这条空转路径。

## 25.8 练习

1. 让模型连续调三次同一个工具（提示词里明确要求），数一下流里有几个 `step.started`。对照 25.2 的表解释。
2. 故意给一个工具传错参数（在提示词里指定一个不存在的字段），观察事件流里出现的是 `turn.failed` 还是 `action.result` 带 `isError`。
3. 把 `thresholdPercent` 调到 0.1，跑几轮对话，看 `compaction.requested` 出现在哪个事件之后。
4. 读 `harness/step/after-model.ts` 的 `appendMissingToolResultMessages`，说出它解决的是什么问题——什么情况下模型会返回一个没有对应结果的工具调用。

## 下一章

模型和工具的循环讲完了。但这一切都发生在服务端内部，外部世界怎么进来、怎么出去？

第 26 章进到 HTTP 表面：默认 eve channel 的 750 行怎么组织 23 条路由，流式响应的 `startIndex` 怎么实现重放，以及 `turnPolicy` 的 `steer` 和 `queue` 差在哪一行。

---

返回：[教程目录](README.md) · 上一章：[24 workflow](24-workflow.md) · 下一章：[26 eve channel](26-eve-channel.md)
