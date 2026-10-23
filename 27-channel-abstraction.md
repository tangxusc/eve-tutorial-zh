# 第 27 章 channel 抽象：平台通道怎么复用同一套

> 一句话结论：`defineChannel` 把「路由表 + adapter + 事件处理器」打成一个带哨兵的对象。平台通道不特殊，只是路由更多、事件处理器更厚、状态更复杂。

## 一个纠结

公司内部有个工单系统，需要在创建工单时通知 agent。你可以在现有的 HTTP 通道里加一条路由，也可以写一个新的自定义通道。

加路由最省事，但工单系统需要自己的鉴权、自己的会话映射、自己的回复格式。全塞进 eve 通道会让那个文件越来越难读。

写新通道听起来重，但 eve 的通道抽象只有一个函数 `defineChannel`。问题变成：从零写一个通道，到底要写多少东西？

这一章回答这个问题，然后拿 Slack 做对照——它是 eve 里最完整的平台通道实现，1529 行。看完你会知道哪些复杂度是 eve 要求的，哪些是平台协议带来的。

## 27.1 defineChannel 只做三件事

```ts
// packages/eve/src/public/definitions/channel.ts:296
export function defineChannel<TState = undefined, TCtx = void, ...>(
  definition: ChannelDefinition<TState, TCtx, TReceiveTarget, TMetadata>,
): Channel<TState, TReceiveTarget, TMetadata> {
  // 两个产物：adapter 是事件处理器集合，cors 是规范化后的配置
  const adapter = buildAdapter(definition);
  const cors = normalizeChannelCors(definition.cors);

  const compiled: CompiledChannel<TState, TReceiveTarget, TMetadata> = {
    __kind: CHANNEL_SENTINEL,   // 运行时判断「这个对象是不是通道」
    routes: definition.routes,
    adapter,
    cors,
    receive: definition.receive,
    turnPolicy: definition.turnPolicy,
  };

  return compiled;
}
```

三件事：把事件处理器打包成 adapter、归一化 CORS、贴上 `__kind` 哨兵。

`__kind: CHANNEL_SENTINEL` 是运行时识别用的。`isCompiledChannel()` 检查这个字段，用来区分「用户写的通道」和「框架内部的 fetch-only 通道」。后者没有这个哨兵，不能作为 `ctx.to()` 的目标。

返回对象里的 `receive` 和 `turnPolicy` 是给跨通道投递用的（27.4）。

注意 `routes` 是原样透传的——`defineChannel` 不解析、不排序、不校验。路由的顺序和内容完全由你决定，冲突要等编译期才检查。

## 27.2 路由 handler 的十一个参数

每个 handler 签名是 `(request, args) => Response`。`args` 里有十一个东西：

| 字段 | 作用 | 有 I/O 吗 |
| --- | --- | --- |
| `params` | 路径参数 | 否 |
| `requestIp` | 客户端 IP | 否 |
| `waitUntil` | 让后台任务活过响应 | 否 |
| `from` | 绑定通道本地地址到操作 | 是（首次调用时） |
| `resolveSession` | 快照当前拥有某地址的会话 | 是 |
| `attachSession` | 绑定固定 session id，不做 I/O | 否 |
| `to` | 选另一个通道的投递目标 | 否 |
| `describe()` | 读编译产物，描述 agent | 是（读 manifest） |
| `listSkillFiles()` / `readSkill()` | 读技能文件 | 是 |
| `invokeTool()` | 在会话外跑一个工具 | 是 |

`attachSession` 和 `resolveSession` 的区别值得记：

```ts
// attachSession: 你已经有 session id，直接拿句柄
const session = attachSession(sessionId);

// resolveSession: 你只有通道本地地址（比如 Slack 频道 id），问它现在归哪个会话
const session = await resolveSession({ channel: "slack", address: channelId });
```

`attachSession` 不做 I/O，所以可以在 handler 里随便调。`resolveSession` 要查存储，是异步的。

`invokeTool` 是给「在对话之外跑工具」用的——比如工单系统的 webhook 直接触发一个查询工具，不建会话。这也是 `describe()` 里 `tools` 字段的用途：先告诉调用方有哪些工具能这么跑。

## 27.3 adapter：跨 step 的持久状态

`ChannelAdapter` 的契约是这样的：

```ts
export type ChannelAdapter<TCtx extends ChannelAdapterContext<any> = ChannelAdapterContext> = {
  readonly kind: string;
  // state 每个 step 边界做一次 JSON 快照：只能放可序列化的东西
  readonly state?: Record<string, unknown>;
  // 把 agent 的输出投影成输入；返回 void 表示用默认投影
  deliver?(payload: DeliverPayload, ctx: TCtx): StepInput | void | Promise<StepInput | void>;
  createAdapterContext?(base: ChannelAdapterContext<StateOf<TCtx>>): TCtx;
  readonly fetchFile?: FetchFileFunction;
  readonly uploadPolicy?: UploadPolicy;
  readonly instrumentation?: {
    readonly metadata?: ChannelInstrumentationMetadataProjector;
    readonly audience?: ChannelAudienceProjector;
  };
} & ChannelEventHandlers<TCtx>;   // 事件处理器：键是事件类型
```

三个要点：

**一、`state` 在每个 step 边界自动 JSON 快照。**

注释写得很直接：

> Mutable adapter state. Auto-serialized at step boundaries via JSON snapshot. The adapter reads and writes this directly.

所以 adapter state 里放的东西必须能过 JSON。放 `Map`、`Date`、类实例，恢复时会丢或变形。

**二、`deliver` 可以改写 harness 看到的输入。**

```ts
// 三种返回：StepInput 表示自己投影、void 表示走默认、Promise 支持异步
deliver?(payload: DeliverPayload, ctx: TCtx): StepInput | void | Promise<StepInput | void>;
```

返回 `StepInput` 就覆盖，返回 `void` 就用默认投影（`defaultDeliverResult`）。Slack 用这个钩子把「@ 了机器人」的消息去掉 @ 前缀，把附件转成 `FilePart`。

**三、事件处理器抛错会被吞掉。**

```ts
export async function callAdapterEventHandler(
  adapter: ChannelAdapter,
  event: UnstampedMessageStreamEvent,
  ctx: ChannelAdapterContext,
): Promise<UnstampedMessageStreamEvent> {
  // 需要等待时把 continuation token 挂进事件，适配器才能续跑
  const eventForHandler = withWaitingContinuationToken(event, ctx);
  const handler = adapter[event.type] as ((data, ctx) => void | Promise<void>) | undefined;

  if (handler !== undefined) {
    try {
      await handler("data" in eventForHandler ? eventForHandler.data : undefined, ctx);
    } catch (error) {
      // 适配器抛错只记日志：事件流不能因为某个平台出错就中断
      log.error("adapter event handler threw — event swallowed", {
        adapterKind: getAdapterKind(adapter),
        eventType: event.type,
        error,
      });
    }
  }

  return withWaitingContinuationToken(eventForHandler, ctx);
}
```

吞掉的理由：事件处理器是副作用（发 Slack 消息、更新状态），它失败不应该让事件流写入失败。否则一个 Slack API 超时会让整个会话的检查点写不进去。

代价是：**适配器的错误只能从日志里找**，不会出现在 API 响应里。`log.error` 的命名空间是 `channel.adapter`，`EVE_LOG_LEVEL=debug` 能看到。

`withWaitingContinuationToken` 那两行也值得注意：`session.waiting` 事件在交给处理器之前会补上 `continuationToken`，返回时再补一次。因为处理器可能把它改掉。

## 27.4 跨通道投递

`ctx.to()` 让一个通道往另一个通道发消息：

```ts
// 跨通道投递的类型：目标地址由通道自己声明，调用方传错会在编译期报错
export type CrossChannelToFn = <TChannel extends ChannelReference<unknown>>(
  channel: TChannel,
  target: InferReceiveTarget<TChannel>,   // 平台地址，由通道自己定义
) => CrossChannelTargetHandle;

export interface CrossChannelTargetHandle {
  send(message: string | UserContent, options: CrossChannelSendOptions): Promise<Session>;
}
```

用法是「选通道 + 选目标 + 发消息」：

```ts
// 在 HTTP 通道的某个路由里，把结果推到 Slack
const slack = to(slackChannel, { channelId: "C0123" });
// 跨通道投递要显式给 auth：这里的调用者不是终端用户
await slack.send("巡检完成，3 个实例异常", { auth: null });
```

`InferReceiveTarget` 从通道定义的 `receive` 签名里推出目标类型，所以传错字段会在编译期报错。

`toCrossChannelTargets` 会把 agent 的所有通道投影成可投递目标，并且**过滤掉没有 `definition` 的通道**：

```ts
export function toCrossChannelTargets(
  channels: readonly ResolvedChannelDefinition[],
): readonly CrossChannelTarget[] {
  return channels.flatMap((channel) =>
    // 没有 definition 的通道（框架内部通道）不暴露给 to()
    channel.definition === undefined ? [] : [{ ... }],
  );
}
```

框架内部的 fetch-only 通道（比如连接回调）不能被 `to()` 选中。只有用 `defineChannel` 写的通道才有 `definition`。

这个功能的实际用途：定时任务在 Slack 里发日报（第 15 章），或者子代理把结果推到另一个通道。没有它，跨通道只能靠外部系统转发。

## 27.5 Slack 的 1529 行怎么组织

`public/channels/slack/` 目录下：

| 文件 | 行数 | 职责 |
| --- | --- | --- |
| `slackChannel.ts` | 1529 | 主文件：定义、路由、事件分发 |
| `api.ts` | 771 | Slack Web API 封装 |
| `interactions.ts` | 776 | 按钮、菜单、模态框的交互处理 |
| `transport.ts` | 187 | HTTP 传输与重试 |
| `verify.ts` | 52 | 请求签名校验 |

主文件里的函数按调用顺序：

| 行 | 函数 | 职责 |
| --- | --- | --- |
| 831 | `slackChannel()` | 定义通道：路由、adapter、事件处理器 |
| 1420 | `verifyInbound()` | 校验 Slack 签名和时间戳 |
| 1066 | `handleEventPost()` | 处理 Events API 的 POST |
| 1210 | `dispatchSlackMessage()` | 把 Slack 消息转成 turn |
| 1437 | `deliverSlackMessage()` | 把 agent 回复转成 Slack 消息 |
| 968 | `receiveOnSlack()` | 主动投递入口（对应 `receive`） |

一次消息的路径：

```text
Slack → POST /slack/events
  → verifyInbound()          签名不对就 401
  → handleEventPost()        区分 event_callback / url_verification
  → dispatchSlackMessage()   解析 @、线程、附件，转成 payload
  → session.send()           进入 eve 的会话层
  → ...（模型、工具）
  → deliverSlackMessage()    回复写回 Slack
```

`verify.ts` 只有 52 行，但它不自己算签名——真正的校验在 vendored 的 `@chat-adapter/slack/webhook` 里。这一层做两件事：检查配置齐全（`signingSecret` 或自定义 verifier 至少有一个），把底层错误归一化成可读消息。

从错误分支能反推底层校验了什么：

| 底层消息 | 含义 |
| --- | --- |
| `signature headers` | 缺 `X-Slack-Signature` 或 `X-Slack-Request-Timestamp` |
| `timestamp is invalid` | 时间戳格式错 |
| `timestamp is too old` | 超出允许偏差，防重放 |
| `signature is invalid` | 签名不匹配 |
| `verifier rejected` | 自定义 verifier 拒绝 |

52 行里没有一行密码学代码，但它是整条链路的安全边界。

**平台通道的复杂度不在 eve 的抽象上，在平台协议上。** 1529 行里真正跟 eve 相关的可能不到 300 行，其余是 Slack 的事件类型、线程语义、富文本格式、限流重试。

## 27.6 一个最小的自定义通道

回到开头那个工单系统的需求。答案是：不需要写 1529 行。

```ts
// agent/channels/webhook.ts
import { defineChannel, POST } from "eve/channels";

export default defineChannel({
  routes: [
    POST("/webhook", async (request, { attachSession }) => {
      const { text, sessionId } = (await request.json()) as { text: string; sessionId?: string };
      // 没带 sessionId 就新建一个：attachSession 不校验会话是否存在
      const session = attachSession(sessionId ?? crypto.randomUUID());
      await session.send(text, { auth: null });
      // 202 表示已受理，回复稍后从事件流来
      return Response.json({ ok: true, sessionId: session.id }, { status: 202 });
    }),
  ],
});
```

这 12 行就是一个能用的通道。它没有鉴权（`auth: null`）、没有事件处理器、没有 adapter state。

要加鉴权，在 handler 第一行判断 header；要加回复投递，在 adapter 里实现 `message.completed`；要保存状态，在 `defineChannel` 里加 `state` 和 adapter 字段。

从 12 行到 1529 行的差距，全是平台协议。理解这一点，就知道什么时候该自己写通道，什么时候该用现成的：

| 你的场景 | 建议 |
| --- | --- |
| 内部 webhook，格式自己定 | 12 行自定义通道 |
| 已有平台（Slack、Discord、Teams） | 用 `channel/slack` 之类 |
| 平台没有官方通道，但协议简单 | 自定义通道 + 平台文档 |
| 平台有复杂交互（按钮、模态框、线程） | 参考 Slack 的实现，别从零写 |

## 27.7 五个注意事项

1. **adapter state 必须能过 JSON。** `Map`、`Date`、类实例在 step 边界会变形。要存复杂结构，先序列化成普通对象。
2. **事件处理器抛错被吞。** 只能从 `channel.adapter` 命名空间的日志里找。API 响应永远是成功的。
3. **`attachSession` 不校验会话是否存在。** 它只是构造一个句柄，真正报错发生在 `send()` 时。
4. **`ctx.to()` 只能选有 `definition` 的通道。** 框架内部通道（连接回调之类）不行。
5. **`deliver` 返回 `void` 和返回 `undefined` 是一回事。** 都走默认投影，不会「清空输入」。

## 27.8 练习

1. 写 27.6 那个 12 行通道，`curl` 打一次，确认返回 202 并且会话能继续。
2. 在自定义通道里加一个 `message.completed` 事件处理器，故意 `throw new Error("test")`。确认请求仍然成功，并在 `EVE_LOG_LEVEL=debug` 的输出里找到那条被吞掉的错误。
3. 在 adapter 的 `state` 里放一个 `new Date()`，连续两轮对话后读出来，看它变成了什么。
4. 读 `slackChannel.ts` 的 `verifyInbound` 和 `verify.ts`，说出它防的是哪三种攻击。

## 下一章

通道写好了，请求能进来了。但进来的请求凭什么被信任？

第 28 章讲鉴权和协议：`routeAuth` 怎么遍历 auth 数组，401 的 `WWW-Authenticate` 谁生成，`forwardedPrincipal` 什么时候可信，以及 `protocol/message.ts` 那 2080 行的事件类型为什么这么设计。

---

返回：[教程目录](README.md) · 上一章：[26 默认 eve channel](26-eve-channel.md) · 下一章：[28 鉴权与协议](28-auth-and-protocol.md)
