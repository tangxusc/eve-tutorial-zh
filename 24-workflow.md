# 第 24 章 workflow：eve 和其他 agent 框架的分水岭

> 一句话结论：eve 把「一个会话」建模成「一个 workflow run」，step 是这个 run 的记账单位。step 内的代码崩溃后会重跑，step 之间的不会。这一层决定了 eve 的恢复、停泊、部署交接和事件流长什么样。

## 一封发了两遍的邮件

你写了个工具，调用外部 API 发邮件。本地测试没问题，上线后偶尔有用户投诉：同一封邮件收到两封。

查日志，两次调用间隔几秒，参数完全一样。你没有重试逻辑。

原因在 eve 的执行模型里：**崩溃恢复会把没走完的 step 从头再跑一遍**。如果进程恰好在「邮件发出去了、检查点还没写」这个窗口里挂掉，恢复时整个 step 重来，邮件就发了两遍。

这不是 bug，也不是你哪里写错了。它是 eve 的执行模型决定的：**eve 把会话建模成一个 workflow，而不是一个跑在进程里的循环。**

这一章先把这个概念讲清楚——它也是 eve 和大多数 agent 框架最根本的区别——再进代码。第 10 章讲过行为，这一章讲它为什么长这样。

## 24.1 普通 agent 的循环，死在哪

先看大多数人熟悉的写法。一个 turn 的核心，简化后就是这样：

```ts
// 大多数 agent 框架的核心，简化后
async function runTurn(history, userMessage) {
  const messages = [...history, userMessage];

  while (true) {
    // 调模型，拿回文本或工具调用
    const response = await model.generate(messages);
    if (response.toolCalls.length === 0) return response.text;

    // 有工具调用就执行，结果塞回消息列表，再进下一轮
    for (const call of response.toolCalls) {
      const result = await tools[call.name].execute(call.args);
      messages.push(toolResultMessage(call, result));
    }
  }
}
```

这段代码能跑，本地测试也永远跑得通。因为它默认了三件事：

1. 进程在这个 turn 结束前不会死；
2. 一个 turn 几分钟内能跑完；
3. 只有这一个进程在动这份 `messages`。

生产环境里，这三条全都不成立：

| 现实 | 结果 |
| --- | --- |
| 滚动部署，旧进程收到 SIGTERM | turn 做到一半没了，用户重发，前面花掉的模型调用白花 |
| serverless 函数超时 | 同上 |
| OOM、panic、机器回收 | 同上 |
| 等审批，人两小时后才点 | 进程得一直挂着，或者你自己实现挂起和恢复 |
| 用户关了页面，第二天回来 | 会话在内存里，没了 |

问题不在代码写得对不对，在**状态放错了地方**。`messages` 在内存里，而内存跟着进程走。进程一没，跑到哪、花了多少钱、工具执行到第几个，全丢。

## 24.2 workflow 的解法：记账，不是凭记忆

一句话：**每做完一步，把「做了什么、结果是什么、做完之后是什么状态」写进存储。进程死了，新进程照账本接着做。**

一个更直白的类比：凭记忆管钱，中途被打断就全乱；记账的话，谁接手都能从最后一行往下写。

这套做法在业界叫 durable execution（持久化执行）。Temporal、Azure Durable Functions 是同一思路的不同实现；eve 用的是 Vercel 的 `@workflow/*`，编译产物里会校验 `@workflow/core`、`@workflow/world` 的版本。名字不重要，重要的是它在做什么。

具体到 eve，账本是 workflow 引擎的存储，记账单位是 step。一个 turn 的账本大概长这样：

```text
run: workflow//eve//workflowEntry/<sessionId>
  step 1   in: 用户消息              out: 模型要求调用 query_metrics
  step 2   in: query_metrics 参数     out: 三个服务的错误率
  step 3   in: 工具结果               out: 模型要求调用 send_email
  step 4   ← 进程在这里被 kill，没有记录
```

重启之后，引擎做三件事：

1. 按 run id 找到账本；
2. step 1-3 有记录，直接取记录里的输出，**不重新执行**；
3. 从 step 4 开始跑。

这里有一个关键区分，很多讲 durable execution 的文章会含糊过去：

- **已记账的 step 不重做。** 引擎保证。
- **没记账的 step 会重做。** 引擎也保证，而且必然如此。

所以 durable 不等于 exactly-once。开头那封发了两遍的邮件，就发生在 step 4 里：邮件发出去了，但 step 4 还没记账。恢复时 step 4 重跑，邮件又发一次。

workflow 做的是把重跑范围从「整个 turn」缩到「一个 step」，不是消灭重跑。副作用要自己做幂等，24.10 会回到这件事。

### 为什么记账单位不能再细

你可能会想：每行代码记一笔不就行了。

不行。记账的前提是「重放时能得到一样的结果」，而大部分代码不满足：

- `Date.now()`、`Math.random()`、`crypto.randomUUID()` 每次调用都不一样；
- `fetch()` 打的外部服务，几分钟后可能已经变了；
- 读文件的内容可能被别的进程改了。

只有边界清晰、输入输出都能序列化的操作才适合记账。模型调用适合（输入是 messages，输出是 response），工具调用适合（输入是参数，输出是结果），沙箱里跑的一段任意脚本不适合。

所以账本只能记在**明确的边界**上。workflow 引擎把这个边界叫 step，eve 把 step 定在「一次模型调用 + 它要求的工具」。

这一句是整章的地基：**step 是记账单位，不是函数调用单位。** 后面所有看起来奇怪的行为，都能从这里推出来。

## 24.3 和其他 agent 框架比，差在哪

先给结论：**别的框架里，workflow 是你可以加的能力；eve 里，workflow 是会话本身的存在形式。**

| 关注点 | 典型做法 | eve |
| --- | --- | --- |
| 会话状态放哪 | 进程内存，可选 checkpointer | 每个 step 都写存储，内存只是当前 step 的缓存 |
| 崩溃恢复 | 从上一个 checkpoint 重来，可能整个 turn | 从上一个 step 继续 |
| 等审批几小时 | 进程挂着，或自己实现挂起/恢复 | `session.waiting`，进程可以退出，事件流不断 |
| 部署交接 | 在跑的会话丢失，或返回 409 | 新进程接管老 run，接着跑 |
| 重放历史 | 自己存消息、自己拼 | 事件流就是账本，`startIndex` 直接重放 |
| 一次崩溃的成本 | 整个 turn 的模型调用重花一遍 | 最多一个 step |
| 子代理 | 同一进程里递归调用 | 另一个 run，独立检查点、独立超时 |

按你可能会用到的几类东西，分开说：

**库，不是运行时**（Vercel AI SDK、OpenAI Agents SDK）。它们提供「自动多轮工具调用」的循环（AI SDK 叫 `maxSteps`，Agents SDK 叫 `maxTurns`）和工具调用协议，循环跑在你的进程里。进程没了，循环就没了。要在外面加持久化，你得自己再套一层——而那一层就是 workflow。

**图 + checkpointer**（LangGraph）。能力上最接近：有节点、有检查点、能中断恢复。差别在默认值和覆盖面。它的检查点粒度由你画的图决定，节点怎么切、状态 schema 怎么定，都是你的事。eve 把粒度定死在 step、把状态 schema 定死在 `DurableSession`，换来的是「不设计也能恢复」。代价是你不能自定义恢复粒度。

**CLI agent**（Claude Code 这类）。会话存在本地文件里，重启能续。但它没有跨进程的队列和检查点：另一个进程不会接管你正在跑的 turn，也没有「部署之后老会话继续」这回事。

**eve**。会话 = 一个 workflow run，从第一行代码起就这么建模。通道、审批、子代理、定时任务全部建在这层上：审批等待 = run 停在 step 边界；子代理 = 另一个 run；定时任务 = 一个带超时的 run。

这个建模有三个直接后果，后面各节都会碰到：

1. **会话有持久 id，跨进程、跨部署。** 它不是「进程里的一个对象」，是存储里的一条记录。
2. **事件流是 run 的副产品。** 不是额外写的日志，是账本可读的那一面。
3. **重放范围由 step 决定。** 不由你的函数调用栈决定。

## 24.4 一个会话就是一个 workflow

概念说完了，看 eve 把它落成了什么。第一步是给 workflow 起名字——这一步最容易被跳过，但它决定了整个恢复机制能不能成立。

### 名字不是给人看的，是给存储用的

启动一个 run，引擎往存储里写一条记录：run id、`workflowName`（就是那个字符串）、输入。它不写函数——函数是代码，代码在磁盘上，换一版就变了。

恢复时反过来：新进程读到一条 run 记录，拿到 `workflow//eve//workflowEntry` 这个字符串，得把它变回一个能调用的函数。它手上没有你的 import 图，只有一张构建时生成的注册表。

注册表只有十行：

```ts
// packages/eve/src/execution/workflow-registry.ts
declare global {
  // The Workflow SDK's generated entrypoint resolves a run's function from this
  // map; eve's transform creates it and registers every `"use workflow"` function.
  var __private_workflows: Map<string, unknown> | undefined;
}

export const WORKFLOW_REGISTRY_GLOBAL = "__private_workflows";

// 按 id 查函数：查不到返回 undefined，由调用方决定报什么错
export function readRegisteredWorkflow(workflowId: string): unknown {
  return globalThis[WORKFLOW_REGISTRY_GLOBAL]?.get(workflowId);
}
```

谁往里写？构建期的 transform。每个带 `"use workflow"` 的函数都会被改写成「删掉指令 + 挂上 workflowId + 注册自己」：

```ts
// packages/eve/src/internal/workflow-bundle/workflow-transformer.ts:183
// 命中稳定名单的用不带版本的基础名，其余用带版本的
const isStable = input.stableWorkflowNames?.has(fn.name) === true;
const workflowId = createWorkflowId(isStable ? stableIdBase : defaultIdBase, fn.name);
manifest.workflows ??= {};
const workflowsForFile = (manifest.workflows[input.filename] ??= {});
workflowsForFile[fn.name] = { workflowId };

if (input.mode === "workflow") {
  // 删掉 "use workflow" 指令本身
  replacements.push({ end: fn.directiveEnd, start: fn.directiveStart, text: "" });
  // 把 id 挂到函数对象上：序列化和 start() 都读它
  suffixes.push(`${fn.name}.workflowId = ${JSON.stringify(workflowId)};`);
  // 注册进全局表：这是「字符串 → 函数」唯一的一条路
  suffixes.push(
    `globalThis.${WORKFLOW_REGISTRY_GLOBAL}.set(${JSON.stringify(workflowId)}, ${fn.name});`,
  );
}
```

所以名字在构建期就烧进了产物，运行期只是查表。它不是注释，是主键。

### id 长什么样

```ts
// packages/eve/src/internal/workflow-bundle/workflow-transformer.ts:686
export function createWorkflowId(idBase: string, functionName: string): string {
  return `workflow//${idBase}//${functionName}`;
}
```

三段：`workflow//<模块>//<函数名>`。最后一段是你写的函数名，中间段决定跨版本还能不能找到。三种真实的 id：

| id | 谁 | 中间段哪来的 |
| --- | --- | --- |
| `workflow//eve//workflowEntry` | eve 的会话入口 | 包名，不带版本 |
| `workflow//eve@0.81.1//executeSleepTool` | eve 自带的普通 workflow 工具 | 包名 + 版本 |
| `workflow//./agent/tools/deploy//execute` | 你写的 workflow 工具 | 相对 app 根的文件路径 |

中间段的两种形态来自 `resolveModuleSpecifier`：

```ts
// packages/eve/src/internal/workflow-bundle/workflow-builders.ts:259
moduleSpecifier: `${base}@${pkg.version}`,   // 默认：带版本
stableModuleSpecifier: base,                 // 稳定名单：只留包名或路径
```

### 为什么只有这六个不带版本

默认带版本戳，是因为大多数 workflow 只是「这次部署内部的实现细节」：谁启动、谁执行都在同一个版本里，换版本之后由新部署重新拉起就行，不需要被老 run 找到。

只有这六个是跨部署的路由键。会话 run 可能活几天、定时器可能跨版本、工具 run 可能卡在审批上——这些 run 必须能被**另一个版本**的进程找到并继续。

如果入口 id 写成 `workflow//eve@0.81.1//workflowEntry`，升级到 0.82 之后新进程注册的是 `eve@0.82.0`，老 run 就永远恢复不了。

所以 eve 挑出六个入口，让它们不带版本戳：

```ts
// packages/eve/src/execution/stable-workflow-names.ts
/**
 * eve workflow functions whose bundled id carries no `@<version>` stamp, so
 * an explicitly stamped deployment can find them across eve versions.
 * Dependency-free so the bundler can import it.
 */
const TURN_WORKFLOW_NAME = "turnWorkflow";
export const WORKFLOW_ENTRY_NAME = "workflowEntry";
export const SESSION_TIMEOUT_WORKFLOW_NAME = "sessionTimeoutWorkflow";
export const WORKFLOW_TOOL_RUN_WORKFLOW_NAME = "workflowToolRunWorkflow";
export const CONNECTION_EVENT_WORKFLOW_NAME = "connectionEventWorkflow";
export const CONNECTION_EVENT_DELIVERY_WORKFLOW_NAME = "connectionEventDeliveryWorkflow";

// 白名单：只有这里面的名字允许作为稳定入口注册
export const STABLE_WORKFLOW_NAMES: ReadonlySet<string> = new Set([
  WORKFLOW_ENTRY_NAME,
  TURN_WORKFLOW_NAME,
  SESSION_TIMEOUT_WORKFLOW_NAME,
  WORKFLOW_TOOL_RUN_WORKFLOW_NAME,
  CONNECTION_EVENT_WORKFLOW_NAME,
  CONNECTION_EVENT_DELIVERY_WORKFLOW_NAME,
]);
```

运行期构造 id 的代码必须和构建期算出同一个字符串，所以它是手写的，不从注册表反查：

```ts
// packages/eve/src/execution/workflow-runtime.ts:103
/**
 * Stable workflow reference used by `start()` to locate the workflow
 * entrypoint registered by the Workflow DevKit builder. The id omits
 * the package version stamp so the long-lived owner can rotate across
 * deployments without rewriting the registry key.
 */
export const workflowEntryReference = {
  workflowId: `workflow//${STABLE_ID_BASE}//${WORKFLOW_ENTRY_NAME}`,
};
```

`STABLE_ID_BASE` 就是包名（`resolveInstalledPackageInfo().name`）。两边对不上时，报错是「找不到这个 workflow」，不是「版本不对」——所以这种错通常表现为启动失败，而不是类型错误。

格式有测试守着，`@` 和源码路径都不许出现：

```ts
// packages/eve/src/execution/workflow-runtime.test.ts:86
// The runtime references intentionally omit the `@<pkg.version>` stamp
// so an explicitly targeted deployment finds the same workflow even when
// eve itself has been upgraded.
expect(workflowEntryReference.workflowId).toBe(`workflow//${packageInfo.name}//workflowEntry`);
expect(workflowEntryReference.workflowId).not.toContain("/src/execution/");
expect(workflowEntryReference.workflowId).not.toContain("@");
```

这两条断言把「不能带版本戳、不能带源码路径」固定下来。谁把构建路径拼进 id，或者顺手加了版本，测试立刻红。

### 六个名字各管什么

| 名字 | 谁启动 | run 里干什么 | 活多久 |
| --- | --- | --- | --- |
| `workflowEntry` | `POST /session`，即 `workflow-runtime.ts` 的 `startSession` | 加载检查点、跑 turn、处理 owner 交接；run 本身是公开事件流的锚点 | 和会话一样长，可能几天 |
| `turnWorkflow` | 旧版 session driver | 把旧执行模型下的会话导入当前 owner 程序 | 只服务历史会话 |
| `sessionTimeoutWorkflow` | 建会话时按 `sessionTimeoutMs` 启动（`session/timeout-steps.ts`） | `sleep` 到截止时间，再给当前 owner 发超时信号 | 一个定时器，可能几小时 |
| `workflowToolRunWorkflow` | 调用 workflow 工具时（`tools/workflow/start.ts`） | 跑工具的 workflow body，处理 `ctx.ask`、`ctx.agent` 的中途交互 | 一次工具调用 |
| `connectionEventWorkflow` | 连接开始监听事件时（`connections/events/owner.ts`） | 持有 hook 等外部事件，每条事件起一个 delivery run | 和订阅一样长 |
| `connectionEventDeliveryWorkflow` | 上面那个 run 收到事件后 | 跑一次投递回调 | 很短 |

三处容易误解：

- **`workflowEntry` 不是「一个 turn 一个 run」**，是「一个会话一个 run」。turn 是这个 run 里的 step，24.5 展开。
- **`sessionTimeoutWorkflow` 是独立 run，不是 `setTimeout`。** 进程重启后定时器还在，因为 `sleep` 也在 workflow 里。
- **`turnWorkflow` 不是废弃名字。** 它在 `execution/legacy-session/turn-workflow.ts`，注释写着 "Historical dispatch name. Drivers from the former driver/turn execution model start this workflow for each turn."——给旧驱动用的兼容入口。这恰好是「名字不能改」的活证据：执行模型换了一代，旧名字还得留着，因为存储里还有指向它的 run。

### 名字还出现在哪

不止注册表。改名等于同时改四个地方：

**一、run 记录。** 存的就是这个字符串，改了就找不到函数。

**二、队列名。** 名字本身就是队列名的一部分，stranded 守卫按它判断这是会话还是工具 run：

```ts
// packages/eve/src/execution/session-inbox/stranded-replay-guard.ts:39
const guardedWorkflows = new Map<string, GuardedWorkflow>([
  [`workflow//${name}//${WORKFLOW_ENTRY_NAME}`, "session"],
  [`workflow//${name}//${WORKFLOW_TOOL_RUN_WORKFLOW_NAME}`, "tool"],
]);
```

**三、health 响应和 `eve info`。** `GET /eve/v1/health` 把入口 id 直接返回给客户端，`eve info` 也打印一行 `Workflow ID`：

```ts
// packages/eve/src/eve-channel/support.ts:24
export function healthResponse(): Response {
  return Response.json({
    ok: true,
    status: "ready",
    workflowId: workflowEntryReference.workflowId,
  });
}
```

**四、报错文案。** 工具改名后，正在跑的 run 恢复时拿到这句：

```ts
// packages/eve/src/execution/tools/workflow/body.ts:257
export function resolveWorkflowEntryPoint<TEntryPoint>(input: WorkflowBodyInput): TEntryPoint {
  const entryPoint = readRegisteredWorkflow(input.workflowId);
  if (typeof entryPoint !== "function") {
    throw new Error(
      `Tool "${input.toolName}" is not registered as a workflow in this deployment (${input.workflowId}). The tool was renamed or removed after this run started.`,
    );
  }
  return entryPoint as TEntryPoint;
}
```

最后一句原文就是改名的代价：**renamed or removed after this run started**。

### 所以那句话是字面意思

「一个会话就是一个 workflow」不是比喻。`POST /session` 创建的不是一个内存对象，是一个 `workflowEntry` 的 run：workflowId 固定，run id 每次不同。恢复、交接、事件流全挂在这条记录上。

## 24.5 一个 step 做什么

run 是账本，step 是账本里的一行。这一行到底记了什么，取决于 step 里执行了什么。

step 的类型定义只有一行：

```ts
// packages/eve/src/harness/types.ts:245
// step 就是一个函数：进 session、出 session。没有隐藏状态，所以能重放
export type StepFn = (session: HarnessSession, input?: StepInput) => Promise<StepResult>;
```

输入是完整会话状态，输出是新的会话状态。看起来像纯函数，实际上不是——它会在里面调用模型、跑工具、写沙箱。

`createExecutionNodeStep` 负责把这个函数装配出来：

```ts
// packages/eve/src/execution/node-step.ts
export function createExecutionNodeStep(input: CreateExecutionNodeStepInput): StepFn {
  // 这几样在 step 之外创建一次，每次调用复用
  const resolveModel = createRuntimeModelResolver(input.modelResolutionScope);
  const tools = createNodeHarnessTools({ node: input.node });
  const instrumentation = input.instrumentation;
  const sessionInstrumentation = instrumentation?.prepareExecution();

  // 真正的循环交给 harness，这里只做装配
  const step = createToolLoopHarness({
    steeringSignal: input.steeringSignal,
    abortSignal: input.abortSignal,
    resolveModel,
    runtimeIdentity: buildRuntimeIdentity(input.node),
    tools,
    // ... 还有十几个参数
  });
  if (instrumentation === undefined) return step;

  // 有 instrumentation 时包一层：成功失败都要在退出前 flush span
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
  // 关键：关掉 SDK 的自动循环，一次调用只走一步。多轮工具调用由外层 step 驱动
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

## 24.6 状态机：decide 是纯函数

每次 step 结束，状态变更走同一个入口：

```ts
// packages/eve/src/execution/publish-session-events.ts:545
export async function commitSessionStep(
  target: SessionStepState | RestoredSessionStep,
  decide: (view: SessionView) => readonly Transition[],
  options: { readonly origin: SessionEventOrigin; readonly inputSource?: string },
): Promise<PublishedSessionEvents> {
  const { state } =
    "ctx" in target ? target.durableSession : readDurableSession(target.sessionState);
  const view = sessionView(storedProjection(state), state);
  // decide 是纯函数：只读视图、返回要写的事件，不碰 IO
  const transitions = decide(view);

  // 一批 transition 只能属于同一个 turn，否则检查点会自相矛盾
  if (
    (transitions.length > 1 && transitions.some((transition) => transition.turn !== view.turn)) ||
    transitions.filter((transition) => transition.signIns !== undefined).length > 1
  ) {
    throw new Error("A transition batch must not overlap turn or sign-in snapshots.");
  }

  // 已经在 step 上下文里：直接提交
  if ("ctx" in target) return await commit(target, transitions, options);

  // 什么都没变就跳过写盘：重放时最省的一步
  if (transitions.every((transition) => changesNothing(view, transition))) {
    return { serializedContext: target.serializedContext, sessionState: target.sessionState };
  }

  // 否则先把 step 上下文恢复出来再提交
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

## 24.7 检查点里存什么

`projectToDurableSession` 定义序列化边界：

```ts
// packages/eve/src/execution/session.ts:180
export function projectToDurableSession(session: HarnessSessionBase): DurableSession {
  // 只挑要跨 step 存活的字段：history 走事件日志，工具定义走 manifest
  const durable: {...} = {
    agent: { system: session.agent.system },
    continuationToken: session.continuationToken,
    sessionId: session.sessionId,
  };

  // 可选字段逐个判断：undefined 不写进去，产物更小，恢复时也不会覆盖默认值
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

## 24.8 恢复时的一个细节

```ts
// packages/eve/src/execution/session.ts:232
export function hydrateDurableSession(input: {
  readonly durable: DurableSession;
  readonly turnAgent: RuntimeTurnAgent;
  readonly compactionOverrides?: {
    readonly thresholdPercent?: number;
  };
}): HarnessSessionBase {
  // 反方向：存下来的 system + 当前的 turnAgent + 当前的工具集，重新拼出会话
  const session: {...} = {
    // system 用旧的（老会话行为不变），工具集用新的（加工具立刻生效）
    agent: createSessionAgent(turnAgent, durable.agent.system, tools),
    // compaction 配置不存，按当前模型重算，只把上次的用量带回来
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

  // 静态模型：直接引用
  if (turnAgent.model !== undefined) return { ...base, modelReference: turnAgent.model };
  // 动态模型：只打标记，等运行时再解析
  if (turnAgent.dynamicModel !== undefined) return { ...base, dynamicModel: true };
  // 两者都没有，说明动态子代理还没选配置，这时候不该建会话
  throw new Error("Cannot create a session before dynamic subagent config is selected.");
}
```

`system` 是参数传进来的（旧提示词），`model` 和 `tools` 来自当前的 `turnAgent`。这一行代码就是上面那个不对称行为的出处。

## 24.9 事件与提交的顺序

一次 step 结束时，事件先发还是状态先写？

```ts
// packages/eve/src/execution/publish-session-events.ts · commit()
const { published } = await publishFromSessionStep(restored, {
  inputSource: options.inputSource,
  origin: options.origin,
  // 逐条发布：顺序就是数组顺序，事件 id 在这里分配
  async publish(emit) {
    for (const transition of transitions) await publishTransition(transition, emit);
  },
  // 发布成功后统一把 transition 落到会话状态上
  updateSession: (session) => ({
    session: transitions.reduce((next, transition) => saveTransition(next, transition), session),
  }),
});
```

在同一个作用域里：先 `publish` 事件，再 `updateSession` 写状态。两者在同一个 step 内完成，所以要么都成功，要么都失败。

但事件投递和事件钩子有先后：

```ts
// 投递先于写入：通道适配器看到的事件，就是最终写进去的那条
/**
 * Delivers and writes one event the step produced, and returns it
 * as written for its hooks. Delivery comes first so the channel adapter's
 * handler shapes what is written.
 */
emit(event: UnstampedMessageStreamEvent): Promise<MessageStreamEvent>;
```

注释说的是：**先投递给通道适配器，再写入。** 因为适配器的 handler 会改变最终写进流里的内容——比如给事件补上 `continuationToken`，或者按平台格式改写。钩子拿到的必须是改写后的版本。

`callAdapterEventHandler` 里那个 try/catch 是配套的：适配器的事件处理器抛错会被记录并吞掉，不会中断事件流写入（第 27 章展开）。

## 24.10 回到那封发了两遍的邮件

机制在 24.2 已经说清楚了：崩溃落在「工具返回、检查点写入」之间，这个 step 会整体重跑，邮件就发第二遍。

这件事躲不掉，只能防。两个办法：

1. **让工具幂等。** 用业务上的唯一键去重。比如邮件用 `sessionId + turnId + callId` 做幂等键，服务端第二次收到直接返回第一次的结果。
2. **把副作用挪到 step 之外。** 比如工具只写一个待发队列，由外部 worker 消费。这样重放只会重复写队列，不会重复发邮件。

eve 没有提供「跳过已执行的工具」的机制，因为引擎的检查点粒度就是 step。第 10 章讲过的所有幂等策略，落点都在这里。

## 24.11 五个注意事项

1. **step 内的模型调用和工具执行都会重跑。** 崩溃发生在检查点写入前，这个 step 从头再来——包括已经花钱调过的模型和已经执行过的 `execute`。工具必须幂等，或者用 `ctx` 里的事务 id 去重。
2. **`agent.system` 存旧的，`agent.tools` 用新的。** 改提示词不影响老会话，加工具影响。见 24.7。
3. **workflow 名字是持久化契约。** 改名等于让所有跑了一半的会话失联。`STABLE_WORKFLOW_NAMES` 里的名字永远不要动。
4. **一批 transition 不能跨 turn。** 想在同一个提交里结束旧 turn 并开新 turn，会被 `A transition batch must not overlap turn` 挡住。
5. **`changesNothing` 会跳过写盘。** 如果你在 step 里只改了 `ctx` 但没有产生任何 transition，这个改动不会进检查点。改状态要通过 `commitSessionStep` 的 transition 走。

## 24.12 练习

1. 在一个工具里 `console.log(Date.now())`，然后 `kill -9` dev server 的**子进程**，重启后让会话继续。数一下这行日志出现了几次。
2. 改 `agent/instructions.md` 的一句话，不重启 dev，在已有会话里追问。看模型是否知道新内容。再开一个新会话对比。
3. 在 `compiled-agent-manifest.json` 里找 `sessionTimeoutWorkflow`，确认它的注册位置。想想为什么超时要用一个独立 workflow 而不是定时器。
4. 读 `harness/session-machine/transitions.ts` 里任意一个 `decide` 函数，确认它没有任何 I/O 调用。
5. 跑 `eve info`，找到 `Workflow ID` 那一行；再 `curl` 一次 `/eve/v1/health`，对比两个值。它们必须一模一样——一边是运行期手写的，一边是构建期算出来的。

## 下一章

step 是检查点边界，这个语义清楚了。但一个 step 内部还有一层：模型调用、工具执行、压缩、停泊。

第 25 章拆开这一层。你会看到为什么一次简单问答的事件流里会有好几个 `step.started`，以及工具参数写错时为什么 turn 不会失败。

---

返回：[教程目录](README.md) · 上一章：[23 runtime](23-runtime.md) · 下一章：[25 harness](25-harness.md)
