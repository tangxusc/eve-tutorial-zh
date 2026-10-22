# 第 26 章 默认 eve channel：23 条路由的源码

> 一句话结论：`eveChannel()` 返回的不是一个 handler，是一张 23 条路由的表加一层统一的鉴权包装。其中 12 条给调用方用，11 条是内部回调和兼容路径。

## 两个 curl 引发的问题

第一个：你照着文档写了 `curl -X POST http://localhost:2000/eve/v1/session`，返回 401，响应体是一段 JSON，`WWW-Authenticate` 头里写着 `Bearer`。你明明在 `agent/channels/eve.ts` 里配了 `localDev()`。

第二个：你写了个客户端订阅事件流，网络抖了一下连接断了。重连时带 `?startIndex=N`，结果收到了重复的 `message.appended`。

这两个问题一个在鉴权，一个在流式协议，都出在同一个文件里——`eve-channel/index.ts`，750 行，注册了 23 条路由。

第 14 章讲过 `eveChannel` 有哪些配置字段。这一章讲这些字段落到哪一行代码，以及上面两个问题怎么排查。

## 26.1 它返回什么

```ts
// packages/eve/src/eve-channel/index.ts:114
export function eveChannel(input: EveChannelInput): EveChannel {
  const uploadPolicy = mergeUploadPolicy(input.uploadPolicy);

  return defineChannel<undefined, EveEventContext>({
    cors: normalizeEveCors(input.cors),
    turnPolicy: input.turnPolicy,
    audience: (classifierInput) => {
      const audience = input.audience ?? defaultEveAudience;
      return typeof audience === "function" ? audience(classifierInput) : audience;
    },
    routes: [
      GET(EVE_HEALTH_ROUTE_PATH, async () => healthResponse()),
      HEAD(EVE_HEALTH_ROUTE_PATH, async () => healthResponse()),
      // ... 还有 20 多个条目
    ],
    events: input.events,
    fetchFile: input.fetchFile,
    uploadPolicy: input.uploadPolicy,
  });
}
```

一个函数，一个返回值。没有 class，没有中间件栈。

`EveChannelInput` 的字段去向值得对一遍：

| 输入字段 | 去处 | 影响 |
| --- | --- | --- |
| `cors` | `normalizeEveCors()` → `defineChannel.cors` | 预检和响应头 |
| `turnPolicy` | 原样传给 `defineChannel` | turn 进行中收到新消息怎么办 |
| `audience` | 包一层默认值 | 追踪可见性 |
| `events` | 原样传给 `defineChannel` | 事件钩子 |
| `fetchFile` | 原样传给 `defineChannel` | 附件下载 |
| `uploadPolicy` | `mergeUploadPolicy()` 后用于**两个地方**：`defineChannel.uploadPolicy` 和每个 handler 里的 `checkUploadPolicy` | 附件大小和类型限制 |
| `auth` | 不传给 `defineChannel` | **每条路由自己调用** `routeAuth` |
| `trustedForwarders` | 不传给 `defineChannel` | 每个 handler 自己判断 |

最后两行是关键设计：**鉴权不是中间件，是每个 handler 的第一行代码。** 这解释了开头第一个问题的一部分——`localDev()` 到底有没有生效，要看请求走到哪条路由、那条路由的 `routeAuth` 怎么判断。

## 26.2 23 条路由

按用途分三类。

**公开（无鉴权）**

| 方法 | 路径 | handler |
| --- | --- | --- |
| `GET` | `/eve/v1/health` | `healthResponse()` |
| `HEAD` | `/eve/v1/health` | 同上 |

`health` 是唯一不调 `routeAuth` 的。它返回 `{ ok, status, workflowId }`，给负载均衡和健康检查用。

**会话（需要鉴权）**

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/eve/v1/info` | 编译后的 agent 快照 |
| `POST` | `/eve/v1/session` | 建会话 |
| `POST` | `/eve/v1/session/:sessionId` | 追问或提交 `inputResponses` |
| `POST` | `/eve/v1/session/:sessionId/cancel` | 取消当前 turn |
| `POST` | `/eve/v1/session/:sessionId/compact` | 强制压缩 |
| `POST` | `/eve/v1/session/:sessionId/clear` | 清空历史 |
| `POST` | `/eve/v1/session/:sessionId/reset` | 终止会话 |
| `GET` | `/eve/v1/session/:sessionId/stubs` | 评测用：查 tool stub 命中情况 |
| `GET` | `/eve/v1/session/:sessionId/stream` | NDJSON 事件流 |
| `GET` | `/eve/v1/session/:parentId/subagents/:callId/:childId/stream` | 代理子会话的流 |

**内部（框架自用）**

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET`/`POST` | `/eve/v1/connections/:name/callback/:token` | 旧版 OAuth 回调，统一返回过期错误 |
| `GET`/`POST` | `/eve/v1/connections/:name/callback/:attemptId/:token` | 新版连接回调 |
| `POST` | `/eve/v1/callback/:token` | 子代理/远程 agent 回调 |
| `POST` | `/eve/v1/task-input/:token` | 旧版任务输入（兼容） |
| `GET`/`POST`/`PUT`/`PATCH`/`DELETE` | `/.well-known/workflow/v1/webhook/:token` | workflow webhook，五种方法同一个 handler |

表格按 handler 归并，一行可能对应多个方法，加起来是 11 条。

最后一行值得注意：**workflow webhook 不在 `/eve/v1` 下**，走的是 workflow 引擎的标准路径 `/.well-known/workflow/v1/webhook/:token`。它注册在 eve 的通道表里，但路径归属是引擎。

内部路由占了将近一半。它们不进公开文档，但都在同一张表里注册，所以也受同一套 CORS 和 instrumentation 约束。

## 26.3 鉴权不是中间件

每条会话路由的第一行都一样：

```ts
GET(EVE_INFO_ROUTE_PATH, async (req, args) => {
  const authResult = await routeAuth(req, input.auth);
  if (authResult instanceof Response) return authResult;
  // ... 业务逻辑
}),
```

`routeAuth` 的返回值只有两种：`Response`（拒绝，直接返回）或鉴权上下文（继续）。

这个模式的好处是**拒绝路径统一**：所有 401/403 的响应体、`WWW-Authenticate` 头都由 `routeAuth` 生成，不会出现「这个路由返回 401，那个路由返回 400」的不一致。

代价是**没有中间件复用**：每个 handler 都要写这两行。加一条新路由忘了写，这条路由就是匿名可访问的。

开头那个 401 的问题，排查顺序是：

1. 请求打到哪条路由？`curl -v` 看路径。
2. 那条路由有没有调 `routeAuth`？`health` 没有，其它都有。
3. `auth` 数组里每一项的返回值是什么？`localDev()` 只在 `eve dev` 进程里放行，看的是部署环境，不是请求头。

第 28 章会拆开 `routeAuth` 看它怎么遍历 `auth` 数组。

## 26.4 建会话要过十一道关

建会话的 handler 是全文件最长的一段，从第 160 行到 330 行，中间有十几道检查。按顺序：

| # | 检查 | 失败响应 |
| --- | --- | --- |
| 1 | `routeAuth` | 401 / 403 |
| 2 | body 是合法 JSON | 400 |
| 3 | `stubs` 需要 `allowToolStubs` | 403 |
| 4 | body 里不能有 `continuationToken` | 400 |
| 5 | `resolveForwardedPrincipal`：转发身份是否可信 | 403 |
| 6 | `parseCreateBody`：字段类型 | 400 |
| 7 | 附件上传策略 | 413 / 415 |
| 8 | `operationId` 需要已认证身份 | 400 |
| 9 | `operationId` 幂等：已有会话直接返回 | 202 |
| 10 | `resolveOnMessage`：钩子可以改写消息或拒绝 | 任意 |
| 11 | 建会话（可能抛 stranded 错误） | 503 |

第 4 条值得单独说。客户端不能自己传 `continuationToken`：

```ts
const tokenRejection = rejectSessionContinuationToken(payload);
if (tokenRejection !== null) return tokenRejection;
```

`continuationToken` 是服务端生成的会话续跑凭据，客户端传进来等于可以劫持任意会话。所以直接拒绝，不是忽略。

第 9 条是幂等：

```ts
const operationToken =
  body.operationId === undefined
    ? undefined
    : await deriveOperationContinuationToken({ auth: forwarded.auth, operationId: body.operationId });
if (operationToken !== undefined) {
  const owner = await args.resolveSession(operationToken);
  if (owner !== undefined) {
    return Response.json(createdSessionBody(owner.id, body), {
      headers: { "cache-control": "no-store", [EVE_SESSION_ID_HEADER]: owner.id },
      status: 202,
    });
  }
}
```

`operationId` 由客户端生成，服务端用「身份 + operationId」推导出一个 continuation token，再用它查已有会话。查到就返回那个会话，不新建。

这是给「网络超时后重试」用的：同一个 operationId 重发，不会创建两个会话。

`deriveOperationContinuationToken` 里带了 `auth`，所以不同用户用同一个 operationId 不会互相命中。

## 26.5 流：startIndex 与 tail index

这是开头第二个问题的所在。

```ts
// packages/eve/src/eve-channel/request.ts:299
export async function createSessionStreamResponse(request: Request, session: Session): Promise<Response> {
  const startIndex = parseStartIndex(request);
  if (startIndex instanceof Response) return startIndex;
  const follow = parseFollow(request);
  const includeTailIndex = !follow || parseIncludeTailIndex(request);

  try {
    // Resolve the tail before committing a response; otherwise an unknown or
    // unreachable session would answer 200 and fail mid-body. Open the event
    // stream alongside the tail lookup to avoid an extra round trip.
    const eventsPromise = session.getEventStream({ follow, startIndex });
    eventsPromise.catch(() => {});
    let tailIndex: number;
    try {
      tailIndex = await session.getStreamTailIndex();
    } catch (error) {
      void eventsPromise.then((events) => events.cancel()).catch(() => {});
      throw error;
    }
    const events = await eventsPromise;
    // ... 构造响应头
    return new Response(
      serializeAsNdjson(
        events,
        request.signal,
        includeTailIndex ? streamEventLimit(startIndex, tailIndex) : undefined,
        controlVersion !== undefined,
      ),
      { headers },
    );
  } catch (error) {
    // stranded / expired / missing 三种错误映射
  }
}
```

四个设计点：

**一、先查 tail index，再提交响应。**

注释写明了原因：如果先返回 200 再查，遇到不存在的会话会在流中途失败，客户端拿到的是「200 加一个半截流」，很难处理。

**二、两个查询并行。**

`eventsPromise` 先发起，再 `await getStreamTailIndex()`。tail 查询失败时取消事件流，避免泄漏。

**三、`includeTailIndex` 的默认值取决于 `follow`。**

`?follow=false`（读历史）时默认带 tail index；`follow` 模式（默认）要显式 `?includeTailIndex=1` 才带。

因为 follow 模式下 tail index 会不断变化，返回一个固定值会误导客户端。

**四、`streamEventLimit(startIndex, tailIndex)` 限制读取范围。**

只读到请求那一刻的 tail，后面新产生的事件由 follow 继续推。这保证了「历史部分和实时部分」的边界清晰。

响应头：

| 头 | 值 | 用途 |
| --- | --- | --- |
| `content-type` | `EVE_MESSAGE_STREAM_CONTENT_TYPE` | NDJSON |
| `cache-control` | `no-store, no-transform` | 禁止缓存和压缩 |
| `x-accel-buffering` | `no` | 关掉 nginx 缓冲 |
| `EVE_SESSION_ID_HEADER` | 会话 id | 客户端确认 |
| `EVE_STREAM_FORMAT_HEADER` | 格式标识 | 版本协商 |
| `EVE_STREAM_VERSION_HEADER` | 版本号 | 版本协商 |
| `EVE_STREAM_TAIL_INDEX_HEADER` | 数字 | 下次续传的起点 |

错误映射：

| 错误 | 响应 |
| --- | --- |
| `StrandedSessionOwnerError` | `strandedSessionResponse()` 生成的 503 |
| 会话超出保留期 | `410` + `{ code: "session_expired" }` |
| workflow run 不存在 | `404` + `Session not found.` |
| 其他 | `500` + `Session stream unavailable.` |

### 为什么重连会收到重复事件

`startIndex` 是「从第 N 条开始读」，不是「跳过 N 条」。如果你上一次收到第 10 条，重连时应该传 `startIndex=11`（下一条）。传 `startIndex=10` 就会把第 10 条再收一遍。

`EVE_STREAM_TAIL_INDEX_HEADER` 给的是「当前最后一条的序号」，客户端应该记成 `lastSeen + 1` 而不是直接用。

## 26.6 turnPolicy 落在哪

`turnPolicy` 在 `eveChannel` 里只是透传，真正用它的是会话句柄：

```ts
// packages/eve/src/channel/session.ts:132
const commandWithoutCaller = {
  auth: options.auth,
  delivery,
  kind: "send" as const,
  payload,
  requestId: metadata.requestId,
  turnPolicy: options.turnPolicy ?? metadata.turnPolicy ?? DEFAULT_TURN_POLICY,
  title: options.title,
};
return await runtime.dispatchSession({
  command: caller === undefined ? commandWithoutCaller : { ...commandWithoutCaller, caller },
  sessionId: id,
});
```

默认值在 `channel/types.ts:188`：

```ts
export const DEFAULT_TURN_POLICY: TurnPolicy = "steer";
```

所以：

- **不配 `turnPolicy`** = `steer`。turn 进行中收到新消息，打断当前 turn。
- **配 `queue`** = 新消息排队，等当前 turn 结束后处理。

`turnPolicy` 随 command 传进 `runtime.dispatchSession`，实际的分支在 session inbox 层（`execution/session-inbox/`）。它决定的是「这条消息进哪个队列」，不是「HTTP 层怎么响应」。

HTTP 层两种策略都返回 202，客户端无法从状态码区分。

## 26.7 回到开头那两个 curl

**401 那个。** `localDev()` 只在 `eve dev` 进程里放行，它看的是部署环境标记，不是请求头。你如果是在 `eve start`（生产模式）或者构建后的产物里打这个请求，`localDev()` 不会放行，最后落到 401。想本地调试，用 `eve dev`；想生产可用，配 `httpBasic` 或自己的鉴权。

**重复事件那个。** `startIndex` 是「从第 N 条开始读」，重连时要传 `lastSeen + 1`。`tail index` 头给的是最后一条的序号，不是下一条。

## 26.8 五个注意事项

1. **`health` 不鉴权。** 它会把 `workflowId` 暴露给任何人。介意的话在前面挂一层网关。
2. **客户端不能传 `continuationToken`。** 传了是 400，不是忽略。
3. **`operationId` 幂等需要已认证身份。** 匿名调用带 `operationId` 会 400，因为没法安全地推导 token。
4. **`follow` 模式下默认不返回 tail index。** 想要就加 `?includeTailIndex=1`。
5. **`turnPolicy` 在 HTTP 层没有区别。** 两种策略都返回 202，想知道消息是打断还是排队，只能看后续事件。

## 26.9 练习

1. 用 `curl` 打一遍 10 条会话路由（不带任何 header），记录每条的状态码。对照 26.2 的表，确认哪些返回 401、哪些返回 400。
2. 用同一个 `operationId` 连续两次 `POST /session`，比较返回的 `sessionId` 和状态码。
3. `GET /session/:id/stream?follow=false` 和 `?follow=false&includeTailIndex=1` 各请求一次，对比响应头。
4. 在 `agent/channels/eve.ts` 里把 `turnPolicy` 改成 `"queue"`，在 TUI 里连续快速发两条消息，观察事件流里 `turn.started` 出现的次数和顺序。

## 下一章

默认通道讲完了。但 eve 能接的不止 HTTP——Slack、Discord、Telegram、自定义 webhook，都走同一套抽象。

第 27 章讲 `defineChannel` 到底做了什么，以及 Slack 那 1529 行是怎么组织的。你会看到从 12 行到 1529 行的差距，几乎全是平台协议。

---

返回：[教程目录](README.md) · 上一章：[25 harness](25-harness.md) · 下一章：[27 channel 抽象](27-channel-abstraction.md)
