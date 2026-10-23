# 第 28 章 鉴权、身份与协议

> 一句话结论：`auth` 不是中间件，是一个按顺序试的函数数组。全部返回空才 401，抛错则按错误里带的 Response 拒绝。

## 一个说不清的 401

你给 agent 配了 `httpBasic`：

```ts
export default eveChannel({
  // 数组里的策略按顺序尝试，第一个返回真值的赢
  auth: [httpBasic({ username: "ops", password: process.env.OPS_PASSWORD! })],
});
```

客户端请求，拿到 401。问题是：密码错了？没带 header？还是服务端没读到环境变量？

响应体只有一行 `{"error":"Unauthorized"}`，`WWW-Authenticate: Basic realm="opsbot"`。看不出区别。

这一章讲 `routeAuth` 怎么遍历、401 怎么生成、以及客户端能从响应头里读到什么。顺便讲事件协议为什么是现在这个样子——那部分决定了你的客户端怎么处理版本升级。

## 28.1 routeAuth：遍历和兜底

```ts
// packages/eve/src/public/channels/auth.ts:712
export async function routeAuth(
  request: Request,
  auth: AuthFn<Request> | readonly AuthFn<Request>[],
): Promise<AuthResult | Response> {
  // 单个函数也接受，统一转成数组
  const list: readonly AuthFn<Request>[] = Array.isArray(auth)
    ? (auth as readonly AuthFn<Request>[])
    : [auth as AuthFn<Request>];

  try {
    // 按顺序试，第一个返回真值的赢；返回 null/undefined 交给下一个
    for (const fn of list) {
      const result = await fn(request);
      if (result) return result;
    }
  } catch (error) {
    // 抛错时带 response 字段：策略可以自定义拒绝响应（429、400……）
    if (
      typeof error === "object" &&
      error !== null &&
      "response" in error &&
      error.response instanceof Response
    ) {
      return error.response;
    }
    throw error;
  }

  // 全都没通过：用策略声明过的 challenge 拼 WWW-Authenticate
  const declaredChallenges = collectDeclaredChallenges(list);
  const challenges =
    declaredChallenges.length > 0
      ? addInvalidBearerTokenError(request, declaredChallenges)
      : [{ scheme: "Bearer" } satisfies UnauthorizedChallenge];
  return createUnauthorizedResponse({ challenges });
}
```

这三十几行定义了全部鉴权语义：

| AuthFn 返回 | 结果 |
| --- | --- |
| 一个 `AuthResult` | 接受，停止遍历 |
| `null` / `undefined` | 跳过，试下一个 |
| 抛出的错误带 `response` 字段 | 用那个 Response 拒绝 |
| 抛出的错误不带 `response` | 继续往上抛，变成 500 |

全部跳过后的 401 有两种情况：

- **配了带 challenge 的 helper**（`httpBasic()` 声明 `Basic`，token 类声明 `Bearer`），用收集到的 challenge。
- **什么都没声明**，默认 `Bearer`。

`addInvalidBearerTokenError` 是一个细节优化：

```ts
function addInvalidBearerTokenError(request, challenges) {
  const authorization = request.headers.get("authorization");
  // 没带 Bearer，或策略没声明 Bearer：不加 error，客户端会去登录
  if (authorization === null || !/^Bearer(?:\s|$)/i.test(authorization.trim())) return challenges;
  if (!challenges.some((challenge) => challenge.scheme === "Bearer")) return challenges;

  // 带了 Bearer 还被拒：加 error="invalid_token"，客户端据此去刷新
  return challenges.map((challenge) =>
    challenge.scheme !== "Bearer" || challenge.parameters?.error !== undefined
      ? challenge
      : { ...challenge, parameters: { ...challenge.parameters, error: "invalid_token" } },
  );
}
```

区分「没带 token」和「带了但无效」：客户端带了 Bearer token，但所有 Bearer 策略都拒了，响应里加 `error="invalid_token"`（RFC 6750）。客户端据此知道该刷新 token 而不是重新登录。

注意注释里的一句话：**要等整个 auth 数组走完才加这个标记**。因为后面的策略（比如 `localDev()`）可能接受这个请求，提前标记会让合法请求拿到误导性的错误。

回到开头那个 401：`httpBasic` 声明的是 `Basic`，不是 `Bearer`，所以不会加 `error="invalid_token"`。你只能从 `WWW-Authenticate: Basic realm="opsbot"` 判断是 Basic 认证失败。想区分「没带」和「密码错」，得自己写 AuthFn（28.2 有模板）。

## 28.2 一个完整 AuthFn 的解剖

`http-basic.ts` 是 98 行的完整实现，适合当模板：

```ts
// packages/eve/src/channel/auth/http-basic.ts:11
export function authenticateHttpBasicStrategy(input: {
  readonly authorization: string;
  readonly strategy: ResolvedHttpBasicAuthStrategy;
}): RouteStrategyAuthenticationResult {
  // 头解析不出来（没带、不是 Basic、base64 坏了）都算「未认证」，交给下一个策略
  const credentials = parseBasicAuthorizationHeader(input.authorization);

  if (credentials === null) {
    return {
      kind: "not-authenticated",
    };
  }

  // 用户名先比：不匹配也只说 not-authenticated，不区分「用户不存在」和「密码错」
  if (normalizeCredential(credentials.username) !== normalizeCredential(input.strategy.username)) {
    return {
      kind: "not-authenticated",
    };
  }

  // 密码用常数时间比较：普通 === 会因为提前返回泄漏前缀
  if (
    !timingSafeStringEquals(
      normalizeCredential(credentials.password),
      normalizeCredential(input.strategy.password),
    )
  ) {
    return {
      kind: "not-authenticated",
    };
  }

  // 通过：principalId 用配置里的用户名，不用请求里的
  return {
    kind: "authenticated",
    principal: {
      attributes: Object.freeze({}),
      authenticator: "http-basic",
      claims: Object.freeze({}),
      principalId: input.strategy.username,
      principalType: "user",
    },
  };
}
```

三个细节：

**一、用户名用普通比较，密码用 timing-safe。**

用户名不是秘密，泄露不了什么；密码比较必须常数时间，否则可以通过响应时间逐字符猜。

**二、`timingSafeStringEquals` 先 hash 再比。**

```ts
// 先 sha256 再比：长度不等时 timingSafeEqual 会抛错，哈希后长度固定
function timingSafeStringEquals(left: string, right: string): boolean {
  return timingSafeEqual(hashString(left), hashString(right));
}

function hashString(value: string): Buffer<ArrayBuffer> {
  return createHash("sha256").update(value, "utf8").digest();
}
```

Node 的 `timingSafeEqual` 要求两个 Buffer 等长，不等长直接抛错。hash 成 32 字节解决长度问题，同时保留常数时间。

**三、principal 的四个字段。**

| 字段 | 含义 | 用在哪 |
| --- | --- | --- |
| `principalId` | 稳定身份标识 | 记忆隔离、审批人记录 |
| `principalType` | `user` / `anonymous` / 其他 | `operationId` 幂等要求非匿名 |
| `authenticator` | 哪个策略认的 | 审计、调试 |
| `claims` / `attributes` | 自定义声明 | 工具里读 `ctx.auth` |

`claims` 和 `attributes` 类型相同（都是 `RouteAuthAttributes`，一组可序列化的键值对），eve 不规定里面放什么，由策略决定。两者都冻结，防止下游修改。工具里通过 `ctx.auth` 读。

### 想区分「没带」和「密码错」

自己写一个 AuthFn，抛出带 `response` 的错误：

```ts
const auth = async (request: Request) => {
  const header = request.headers.get("authorization");
  if (header === null) return null;          // 没带，交给下一个策略

  // 带了但不是 Basic：不是「没认证」，是请求错了，直接 400
  if (!header.startsWith("Basic ")) {
    throw Object.assign(new Error("wrong scheme"), {
      response: Response.json({ error: "expected Basic" }, { status: 400 }),
    });
  }

  const ok = verifyBasic(header);
  if (!ok) {
    throw Object.assign(new Error("bad credentials"), {
      response: Response.json({ error: "invalid_credentials" }, { status: 401 }),
    });
  }

  return { /* AuthResult */ };
};
```

`routeAuth` 看到 `error.response` 就直接返回它，不会走到默认的 401。这样客户端能区分三种情况。

## 28.3 转发身份：什么时候可信

`forwardedPrincipal` 解决的是「上游服务代用户调用」的场景。它不能无条件信任，所以有一个显式开关：

```ts
// 只有 trustedForwarders 里列出的来源，转发来的身份才被采纳
const forwarded = await resolveForwardedPrincipal({
  trustedForwarders: input.trustedForwarders,
  forwarder: auth,
  payload,
});
if (forwarded instanceof Response) return forwarded;
```

`trustedForwarders` 是一个谓词，签名类似 `(forwarder, context) => boolean`。不配就是「一律不信任」，任何转发身份都被忽略（不是拒绝，是忽略）。

为什么是「忽略」而不是「拒绝」：转发身份是可选的增强。一个不带转发身份的正常请求应该照常处理，不应该因为「你没配 trustedForwarders」而失败。

追踪上下文走另一条路：

```ts
// 只有回调请求才解析 traceparent：普通请求不该被上游的 trace 影响
const transportParentTraceContext =
  body.callback === undefined ? undefined : parseTraceparent(req.headers.get("traceparent"));
const parsedParentTraceContext =
  body.callback === undefined
    ? undefined
    : (readAgentDispatchTraceContext(req.headers.get("tracestate"), transportParentTraceContext) ??
        transportParentTraceContext);
```

`traceparent` 是 W3C 标准头，`tracestate` 里带 eve 自己的扩展（`readAgentDispatchTraceContext`）。两者只在 `body.callback !== undefined` 时才读——也就是只有「远程 agent 回调」场景才接受追踪上下文，普通请求里的 `traceparent` 被忽略。

这是一条重要的安全边界：**追踪头只在已声明的调用链里生效**，否则任何人都能伪造 trace 关系，污染可观测性数据。

## 28.4 事件协议：为什么是判别联合

`protocol/message.ts` 有 2080 行，大部分是事件类型定义。顶层是一个 37 个成员的联合：

```ts
// 判别联合：每个成员自带 type 字面量，switch 之后 data 的字段就确定了
export type UnstampedMessageStreamEvent =
  | ActionInputAppendedStreamEvent
  | AgentStartedStreamEvent
  | ApprovalCandidateStreamEvent
  | ApprovalSettledStreamEvent
  | ContextClearedStreamEvent
  | CompactionCompletedStreamEvent
  | CompactionRequestedStreamEvent
  | AuthorizationCompletedStreamEvent
  | AuthorizationRequiredStreamEvent
  | MessageAppendedStreamEvent
  | MessageCompletedStreamEvent
  | MessageReceivedStreamEvent
  | ReasoningAppendedStreamEvent
  | SessionCompletedStreamEvent
  | SessionFailedStreamEvent
  | SessionStartedStreamEvent
  | SessionWaitingStreamEvent
  // ... 还有十几个
```

每个成员都有一个字面量 `type` 字段做判别。这样客户端可以：

```ts
// 按 type 分支之后，TypeScript 知道 data 有哪些字段
switch (event.type) {
  case "message.appended":
    // TypeScript 在这里知道 event.data 有 messageDelta
    break;
  case "action.result":
    // 这里 event.data 有 toolName、isError、output
    break;
}
```

用联合而不是「一个通用事件加 payload 字段」，好处是**新增事件类型是纯增量**。加一个 `reasoning.appended` 不会影响任何现有客户端——它们的 `switch` 没有这个分支，走 default 忽略。

反过来，**改字段名是破坏性的**。把 `messageDelta` 改成 `delta`，所有读这个字段的客户端都会拿到 `undefined`，而且不会报错。

信封：

```ts
// id 从 stream version 20 起才有：回放老会话时这个字段可能缺失
export interface MessageStreamEventMeta {
  /** Server-issued message delivery identities, retained across the turn's workflow steps. */
  readonly deliveryIds?: readonly string[];
  /** ISO-8601 emission time. */
  readonly at: string;
  /**
   * Unique, lexicographically sortable identifier for this event.
   *
   * Stamped from stream version 20 on: rewinding into a session that started
   * before the upgrade yields events whose `id` is absent despite this type,
   * and those cannot be deduplicated.
   */
  readonly id: string;
}
```

`id` 的注释信息量很大：**从 stream version 20 起才有 `id`**。回放一个升级前创建的会话，拿到的事件没有 `id`，没法去重。

`id` 是「字典序可排序」的，所以客户端可以按字符串比较判断先后，不需要解析时间。

当前版本：

```ts
// 协议版本号：加字段会升版本，客户端据此判断支持哪些能力
export const EVE_MESSAGE_STREAM_VERSION = "26";
```

26 次协议演进。这个数字在响应头 `x-eve-stream-version` 里，客户端可以据此判断服务端支持哪些事件。

## 28.5 NDJSON 的取消语义

流式响应用 NDJSON（一行一个 JSON），编码在 `execution/session-event-stream.ts`，解码在 `execution/ndjson-stream.ts`。

解码器只有 83 行，但注释解释了为什么不能省事：

```ts
// 取消必须往上游传：否则停泊的会话会让文件轮询一直挂着
/**
 * Cancellation is forwarded to the source. When the returned stream is
 * cancelled — e.g. an SSE client disconnects and the server cancels the
 * response body — the underlying reader is cancelled too. This matters for
 * runs that never reach EOF: a parked (`session.waiting`) durable run keeps
 * its event stream open indefinitely, and the world-local streamer runs a
 * filesystem poll until its reader is cancelled. Without forwarding the
 * cancel, a pending `reader.read()` would block forever and that poll would
 * leak for the life of the process, degrading streaming throughput for every
 * other session.
 */
```

翻译：一个停泊的会话（`session.waiting`）永远不会 EOF，它的流一直开着。dev 环境的 world 实现靠**轮询文件系统**来推新事件。如果客户端断开后服务端不取消读取，那个轮询会一直跑，泄漏到进程结束，拖慢所有会话。

所以 `cancel(reason)` 里必须做两件事：

```ts
async cancel(reason) {
  cancelled = true;
  // 关键在这一行：把取消传给底层 reader，否则 pending 的 read() 永不返回
  await reader?.cancel(reason);
}
```

先置标志位，再取消底层 reader。标志位是为了让挂起的 `pull()` 在 `read()` 返回后立刻退出，不去碰已经关闭的 controller。

这类 bug 只在「客户端频繁断线 + 长会话」的组合下出现，本地开发几乎测不出来。

## 28.6 回到那个 401

现在能说清了。

`httpBasic` 走的是 `routeAuth` 的「全部跳过」路径，生成 401 + `WWW-Authenticate: Basic realm="..."`。它不区分「没带」和「密码错」，因为 Basic 协议本身不区分。

想区分，自己写 AuthFn，在「没带」时返回 `null`（交给下一个策略），在「密码错」时抛一个带 `response` 的错误。`routeAuth` 会用你给的 Response。

想看是哪个策略拒的，开 `EVE_LOG_LEVEL=debug`，日志里有 `authenticator` 字段。

## 28.7 五个注意事项

1. **`trustedForwarders` 不配等于「忽略转发身份」，不是「拒绝」。** 请求照常处理，只是没有转发来的身份。
2. **`none()` 放在数组中间会让后面的策略永远不执行。** 它返回 truthy，遍历立即停止。
3. **401 的 `error="invalid_token"` 只在带 Bearer 且全部 Bearer 策略都拒时才加。** 客户端据此区分「该刷新」还是「该登录」。
4. **事件 `id` 在升级前的会话里可能缺失。** 不能假设它一定存在。
5. **不转发流取消会泄漏轮询。** 自己实现流式端点时，`cancel` 里一定要取消底层 reader。

## 28.8 练习

1. 配一个 `auth: [none(), httpBasic(...)]` 的通道，用 `curl` 不带 header 请求，确认 200。再调换顺序，确认变成 401。
2. 用 `curl -H "Authorization: Bearer bad"` 请求一个只配了 `httpBasic()` 的通道，看 `WWW-Authenticate` 头里有没有 `error="invalid_token"`。
3. 读 `protocol/message.ts` 里 `MessageReceivedStreamEvent` 的定义，列出它的 `data` 有哪些字段，和附录 B 的事件表对照。
4. 写一个自定义 `AuthFn`，抛出一个带 `response` 字段的错误（比如 429 + `Retry-After`），确认 `routeAuth` 用的是你给的那个 Response 而不是默认 401。

## 下一章

鉴权和协议讲完了。请求能进来，能被信任，事件能流出去。

最后一个问题：这些东西存在哪？崩溃了怎么恢复？部署新版本时，跑了一半的会话怎么办？

第 29 章讲 world——eve 对 workflow 引擎的存储抽象，以及部署交接时最危险的那个场景。

---

返回：[教程目录](README.md) · 上一章：[27 channel 抽象](27-channel-abstraction.md) · 下一章：[29 world](29-world.md)
