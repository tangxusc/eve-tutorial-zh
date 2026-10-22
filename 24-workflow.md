# 第 24 章 workflow：会话为什么是一串 step

> 一句话结论：step 不是「函数调用」，是「检查点边界」。step 内的代码在崩溃恢复时会重跑，step 之间的不会。这是 eve 里唯一必须记住的语义。

## 一封发了两遍的邮件

你写了个工具，调用外部 API 发邮件。本地测试没问题，上线后偶尔有用户投诉：同一封邮件收到两封。

查日志，两次调用间隔几秒，参数完全一样。你没有重试逻辑。

原因在 eve 的执行模型里：**崩溃恢复会把没走完的 step 从头再跑一遍**。如果进程恰好在「邮件发出去了、检查点还没写」这个窗口里挂掉，恢复时整个 step 重来，邮件就发了两遍。

这不是 bug，是 workflow 引擎的语义。第 10 章讲过行为，这一章讲代码，讲完之后你会知道为什么必须给这类工具加幂等。

## 24.1 一个会话就是一个 workflow

eve 在 workflow 引擎上注册了几个固定名字：

```ts
// packages/eve/src/execution/stable-workflow-names.ts
const TURN_WORKFLOW_NAME = "turnWorkflow";
export const WORKFLOW_ENTRY_NAME = "workflowEntry";
export const SESSION_TIMEOUT_WORKFLOW_NAME = "sessionTimeoutWorkflow";
export const WORKFLOW_TOOL_RUN_WORKFLOW_NAME = "workflowToolRunWorkflow";
export const CONNECTION_EVENT_WORKFLOW_NAME = "connectionEventWorkflow";
export const CONNECTION_EVENT_DELIVERY_WORKFLOW_NAME = "connectionEventDeliveryWorkflow";

export const STABLE_WORKFLOW_NAMES: ReadonlySet<string> = new Set([
  WORKFLOW_ENTRY_NAME,
  TURN_WORKFLOW_NAME,
  SESSION_TIMEOUT_WORKFLOW_NAME,
  WORKFLOW_TOOL_RUN_WORKFLOW_NAME,
  CONNECTION_EVENT_WORKFLOW_NAME,
  CONNECTION_EVENT_DELIVERY_WORKFLOW_NAME,
]);
```

文件头的注释是关键：

> eve workflow functions whose bundled id carries no `@<version>` stamp, so an explicitly stamped deployment can find them across eve versions.

这些名字**不带版本戳**，所以升级 eve 之后，旧会话仍然能找到对应的 workflow 函数。反过来，如果你自己写 workflow 时改了名字，跑了一半的会话就找不回来了。

实际承载会话的是 `workflowEntry`。`workflow-runtime.ts` 里定义了它的稳定引用：

```ts
// packages/eve/src/execution/workflow-runtime.ts:107
export const workflowEntryReference = {
  workflowId: `workflow//${STABLE_ID_BASE}//${WORKFLOW_ENTRY_NAME}`,
};
```

`start()` 用这个 id 定位入口。注释说明了为什么 id 里没有版本戳：

> The id omits the package version stamp so the long-lived owner can rotate across deployments without rewriting the registry key.

一个会话就是一个 `workflowEntry` 的 run，每个 turn 是其中的一次 step。

`turnWorkflow` 虽然也在 `STABLE_WORKFLOW_NAMES` 里，但当前源码没有引用它——它是保留的名字，不是活跃入口。读源码时别被名字骗了。

## 24.2 一个 step 做什么

step 的类型定义只有一行：

```ts
// packages/eve/src/harness/types.ts:245
export type StepFn = (session: HarnessSession, input?: StepInput) => Promise<StepResult>;
```

输入是完整会话状态，输出是新的会话状态。看起来像纯函数，实际上不是——它会在里面调用模型、跑工具、写沙箱。

`createExecutionNodeStep` 负责把这个函数装配出来：

```ts
// packages/eve/src/execution/node-step.ts
export function createExecutionNodeStep(input: CreateExecutionNodeStepInput): StepFn {
  const resolveModel = createRuntimeModelResolver(input.modelResolutionScope);
  const tools = createNodeHarnessTools({ node: input.node });
  const sessionInstrumentation = input.instrumentation?.prepareExecution();
  const step = createToolLoopHarness({
    steeringSignal: input.steeringSignal,
    abortSignal: input.abortSignal,
    resolveModel,
    runtimeIdentity: buildRuntimeIdentity(input.node),
    tools,
    // ... 还有十几个参数
  });
  if (instrumentation === undefined) return step;
  return async (session, stepInput) => {
    try {
      return await step(session, stepInput);
    } finally {
      await instrumentation.flush();
    }
  };
}
```

它没有包一层业务逻辑——`createToolLoopHarness` 返回什么就交出去，只在有 instrumentation 时加一个 flush。装配（工具表、模型解析器、动态工具解析器）都发生在 `createExecutionNodeStep` 被调用的那一刻，不在 step 执行时。

一个 step 里发生的事：

1. 结算上一个 step 停泊的工作（`settleRuntimeWork`）。
2. 接受人类输入（`acceptHumanInput`）：审批、问题回答、排队消息。
3. 准备这一轮的输入，必要时开一个新 turn（`openTurn`）。
4. 执行已被批准的工作（`admitApprovedWork`）。
5. 调一次模型，流式接收（`runModelStep`）。
6. 模型要调工具就执行，结果写进会话。
7. 返回新的 `session`，step 结束。

**第 5 步只调一次模型。** 不是「一直循环到模型不再调工具」，是「调一次，执行它的工具，然后结束」。下一轮模型调用是下一个 step。

证据在 `model-call/call.ts`：

```ts
const agent = new ToolLoopAgent({
  // ...
  stopWhen: isStepCount(1),
  tools,
});
```

eve 用的是 AI SDK 的 `ToolLoopAgent`，但把它的循环限制成一步。所以：

| 模型行为 | step 数 |
| --- | --- |
| 直接回答 | 1 |
| 调 1 个工具，再回答 | 2 |
| 调工具 A，再调工具 B，再回答 | 3 |

这个设计的意义：**每次模型调用都是一个检查点**。崩溃恢复时最多重跑一次模型调用，而不是整个 turn。

`experimental.workflow.modelCallsPerStep` 这个配置项控制的就是这里的 `isStepCount(1)`（附录 B 里有它，默认 1）。

## 24.3 状态机：decide 是纯函数

每次 step 结束，状态变更走同一个入口：

```ts
// packages/eve/src/execution/publish-session-events.ts:545
export async function commitSessionStep(
  target: SessionStepState | RestoredSessionStep,
  decide: (view: SessionView) => readonly Transition[],
  options: { readonly origin: SessionEventOrigin; readonly inputSource?: string },
): Promise<PublishedSessionEvents> {
  const { state } = "ctx" in target ? target.durableSession : readDurableSession(target.sessionState);
  const view = sessionView(storedProjection(state), state);
  const transitions = decide(view);
  if (
    (transitions.length > 1 && transitions.some((transition) => transition.turn !== view.turn)) ||
    transitions.filter((transition) => transition.signIns !== undefined).length > 1
  ) {
    throw new Error("A transition batch must not overlap turn or sign-in snapshots.");
  }
  if ("ctx" in target) return await commit(target, transitions, options);
  if (transitions.every((transition) => changesNothing(view, transition))) {
    return { serializedContext: target.serializedContext, sessionState: target.sessionState };
  }
  return await commit(await restoreSessionStep(target), transitions, options);
}
```

设计要点：

**一、`decide` 是纯函数。**

输入 `SessionView`（当前状态的只读投影），输出 `Transition[]`（要做的事）。它不写数据库、不发事件、不改状态。所有副作用在 `commit` 里。

这让状态迁移可以单独测试。`harness/session-machine/transitions.ts` 那 863 行，大部分是这种纯函数。

**二、一批 transition 不能跨 turn。**

上面的检查：如果一批里有多条 transition 且 turn 不同，直接抛错。这是为了防止「一次提交里既结束了旧 turn 又开了新 turn」这种中间状态。

**三、无变化直接返回。**

`changesNothing` 检查 turn 没变、没有事件、没有登录、没有转发、没有提交标记、没有清空历史——全满足就不写盘。

这个优化很重要：一个 turn 里可能有很多次「检查有没有新输入」的 step，每次都写检查点会让存储爆掉。

## 24.4 检查点里存什么

`projectToDurableSession` 定义序列化边界：

```ts
// packages/eve/src/execution/session.ts:180
export function projectToDurableSession(session: HarnessSessionBase): DurableSession {
  const durable: {...} = {
    agent: { system: session.agent.system },
    continuationToken: session.continuationToken,
    sessionId: session.sessionId,
  };

  if (
    session.compaction.lastKnownInputTokens !== undefined ||
    session.compaction.lastKnownPromptMessageCount !== undefined
  ) {
    durable.compaction = {
      lastKnownInputTokens: session.compaction.lastKnownInputTokens,
      lastKnownPromptMessageCount: session.compaction.lastKnownPromptMessageCount,
    };
  }
  if (session.rootSessionId !== undefined) {
    durable.rootSessionId = session.rootSessionId;
  }
  if (session.limits !== undefined) {
    durable.limits = session.limits;
  }
  if (session.outputSchema !== undefined) {
    durable.outputSchema = session.outputSchema;
  }
  if (session.sandboxState !== undefined) {
    durable.sandboxState = session.sandboxState;
  }
  if (session.state !== undefined) {
    durable.state = session.state;
  }
  return durable;
}
```

| 存进检查点 | 不存，恢复时重建 |
| --- | --- |
| `agent.system`（系统提示词全文） | `agent.modelReference` |
| `continuationToken` | `agent.tools`（工具列表） |
| `sessionId` | `agent.reasoning` |
| `compaction.lastKnown*` | `agent.compactionModelReference` |
| `rootSessionId` | `history`（在事件流里） |
| `limits` | `sandboxState` 之外的沙箱运行时 |
| `outputSchema` | |
| `sandboxState` | |
| `state`（`defineState` 的值） | |

这个划分带来一个不明显的行为：**恢复老会话时，系统提示词用旧的，工具列表用新的。**

因为 `agent.system` 被存下来了，而 `agent.tools` 没有。恢复时 `hydrateDurableSession` 从当前的 `turnAgent` 重新生成工具列表。

实际影响：你改了 `instructions.md`，老会话仍然用旧提示词；你加了一个工具，老会话立刻能用上。这两件事的不对称经常让人困惑。

`history` 也不在检查点里——它在事件流里，通过 `startIndex` 重放（第 9 章）。

## 24.5 恢复时的一个细节

```ts
// packages/eve/src/execution/session.ts:232
export function hydrateDurableSession(input: {
  readonly durable: DurableSession;
  readonly turnAgent: RuntimeTurnAgent;
  readonly compactionOverrides?: { readonly thresholdPercent?: number };
}): HarnessSessionBase {
  const session: {...} = {
    agent: createSessionAgent(turnAgent, durable.agent.system, tools),
    compaction: createCompactionConfig({
      contextWindowTokens: turnAgent.model?.contextWindowTokens,
      lastKnownInputTokens: durable.compaction?.lastKnownInputTokens,
      lastKnownPromptMessageCount: durable.compaction?.lastKnownPromptMessageCount,
      thresholdPercent: input.compactionOverrides?.thresholdPercent,
    }),
    continuationToken: durable.continuationToken,
    sessionId: durable.sessionId,
  };
  // ...
  if (durable.limits !== undefined) {
    session.limits = durable.limits;
  }
  // ...
}
```

注意 `limits` 那里有一行注释：

> Persisted limits are already resolved (defaults, `false`, and any inherited parent budget applied at creation). Rehydrating verbatim keeps an uncapped session uncapped instead of re-applying the root default.

翻译：检查点里的 `limits` 已经解析过了——默认值、`false`（不限制）、从父会话继承的预算，在创建时就定好了。恢复时原样使用，不要重新套用根会话的默认值。

不这么做的后果：一个显式设成「不限 token」的子代理会话，恢复后会被重新套上 4000 万 token 的默认上限。

`agent` 的构造也值得看一眼：

```ts
function createSessionAgent(turnAgent, system, tools): SessionAgent {
  const base = {
    compactionModelReference: turnAgent.compactionModel,
    reasoning: turnAgent.reasoning,
    system,
    tools,
  };

  if (turnAgent.model !== undefined) return { ...base, modelReference: turnAgent.model };
  if (turnAgent.dynamicModel !== undefined) return { ...base, dynamicModel: true };
  throw new Error("Cannot create a session before dynamic subagent config is selected.");
}
```

`system` 是参数传进来的（旧提示词），`model` 和 `tools` 来自当前的 `turnAgent`。这一行代码就是上面那个不对称行为的出处。

## 24.6 事件与提交的顺序

一次 step 结束时，事件先发还是状态先写？

```ts
// packages/eve/src/execution/publish-session-events.ts · commit()
const { published } = await publishFromSessionStep(restored, {
  inputSource: options.inputSource,
  origin: options.origin,
  async publish(emit) {
    for (const transition of transitions) await publishTransition(transition, emit);
  },
  updateSession: (session) => ({
    session: transitions.reduce((next, transition) => saveTransition(next, transition), session),
  }),
});
```

在同一个作用域里：先 `publish` 事件，再 `updateSession` 写状态。两者在同一个 step 内完成，所以要么都成功，要么都失败。

但事件投递和事件钩子有先后：

```ts
/**
 * Delivers and writes one event the step produced, and returns it
 * as written for its hooks. Delivery comes first so the channel adapter's
 * handler shapes what is written.
 */
emit(event: UnstampedMessageStreamEvent): Promise<MessageStreamEvent>;
```

注释说的是：**先投递给通道适配器，再写入。** 因为适配器的 handler 会改变最终写进流里的内容——比如给事件补上 `continuationToken`，或者按平台格式改写。钩子拿到的必须是改写后的版本。

`callAdapterEventHandler` 里那个 try/catch 是配套的：适配器的事件处理器抛错会被记录并吞掉，不会中断事件流写入（第 27 章展开）。

## 24.7 回到那封发了两遍的邮件

step 的边界决定了重放的边界。发邮件的工具在某个 step 里执行，如果崩溃发生在「工具返回、检查点写入」之间，恢复时这个 step 从头再来：

- 模型调用重跑一次（多花一次钱）
- 工具执行重跑一次（多发一封邮件）

两个办法避免：

1. **让工具幂等。** 用业务上的唯一键去重。比如邮件用 `sessionId + turnId + callId` 做幂等键，服务端第二次收到直接返回第一次的结果。
2. **把副作用挪到 step 之外。** 比如工具只写一个待发队列，由外部 worker 消费。这样重放只会重复写队列，不会重复发邮件。

eve 没有提供「跳过已执行的工具」的机制，因为引擎的检查点粒度就是 step。第 10 章讲过的所有幂等策略，落点都在这里。

## 24.8 五个坑

1. **step 内的模型调用和工具执行都会重跑。** 崩溃发生在检查点写入前，这个 step 从头再来——包括已经花钱调过的模型和已经执行过的 `execute`。工具必须幂等，或者用 `ctx` 里的事务 id 去重。
2. **`agent.system` 存旧的，`agent.tools` 用新的。** 改提示词不影响老会话，加工具影响。见 24.4。
3. **workflow 名字是持久化契约。** 改名等于让所有跑了一半的会话失联。`STABLE_WORKFLOW_NAMES` 里的名字永远不要动。
4. **一批 transition 不能跨 turn。** 想在同一个提交里结束旧 turn 并开新 turn，会被 `A transition batch must not overlap turn` 挡住。
5. **`changesNothing` 会跳过写盘。** 如果你在 step 里只改了 `ctx` 但没有产生任何 transition，这个改动不会进检查点。改状态要通过 `commitSessionStep` 的 transition 走。

## 24.9 练习

1. 在一个工具里 `console.log(Date.now())`，然后 `kill -9` dev server 的**子进程**，重启后让会话继续。数一下这行日志出现了几次。
2. 改 `agent/instructions.md` 的一句话，不重启 dev，在已有会话里追问。看模型是否知道新内容。再开一个新会话对比。
3. 在 `compiled-agent-manifest.json` 里找 `sessionTimeoutWorkflow`，确认它的注册位置。想想为什么超时要用一个独立 workflow 而不是定时器。
4. 读 `harness/session-machine/transitions.ts` 里任意一个 `decide` 函数，确认它没有任何 I/O 调用。

## 下一章

step 是检查点边界，这个语义清楚了。但一个 step 内部还有一层：模型调用、工具执行、压缩、停泊。

第 25 章拆开这一层。你会看到为什么一次简单问答的事件流里会有好几个 `step.started`，以及工具参数写错时为什么 turn 不会失败。

---

返回：[教程目录](README.md) · 上一章：[23 runtime](23-runtime.md) · 下一章：[25 harness](25-harness.md)
