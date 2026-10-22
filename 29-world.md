# 第 29 章 world：检查点存在哪、怎么恢复

> 一句话结论：world 是 eve 对 workflow 引擎的存储抽象。安装时做三件事：校验版本兼容、包一层 stranded 保护、启动。部署交接时最危险的不是崩溃，是新版本重放旧版本留下的 run。

## 一次部署引发的故障

周五下午发版，v2 上线。周一早上有人报告：上周五聊到一半的那个会话，现在一打开就报错，`CORRUPTED_EVENT_LOG`。

你查了 v2 的代码，没动过事件格式。查了数据库，那条 run 的记录还在。

真正的原因是：**v2 的进程把 v1 留下的活跃 run 重新捡起来跑了。**

local world 和 Postgres world 在启动时会重新入队所有活跃的 run。v1 的检查点是 v1 的格式，v2 的代码解析不了，于是报「事件日志损坏」。

这一章讲 eve 怎么处理这个场景，以及 world 在代码里怎么被装上。第 10 章讲过行为和选型，这一章讲实现。

## 29.1 installWorkflowWorld：19 行

```ts
// packages/eve/src/execution/install-workflow-world.ts
import { getWorld, setWorld } from "#internal/workflow/runtime.js";
import { guardStrandedSessionReplay } from "#execution/session-inbox/stranded-replay-guard.js";
import { validateWorkflowWorld, type ValidateWorkflowWorldInput } from "#internal/workflow/validate-world.js";

/**
 * Validates a Workflow World, installs it as the runtime singleton, and
 * starts it. Generated World plugins call this so eve owns what wraps the
 * World before its queue begins delivering.
 */
export async function installWorkflowWorld(input: ValidateWorkflowWorldInput): Promise<void> {
  validateWorkflowWorld(input);
  const world = guardStrandedSessionReplay(input.world);
  setWorld(world);
  await getWorld();
  await world.start?.();
}
```

四步，顺序不能换：

1. **`validateWorkflowWorld`** —— 版本兼容和形状检查。不通过就抛错，不让一个有问题的 world 进到运行时。
2. **`guardStrandedSessionReplay`** —— 包装 `createQueueHandler`，在队列投递前拦截「不属于这个版本的 run」。
3. **`setWorld`** —— 设为全局单例。
4. **`world.start?.()`** —— 启动。

注释里那句「eve owns what wraps the World before its queue begins delivering」是重点：**包装必须在 start 之前**。一旦 start 了，队列可能立刻开始投递，那时候再包就晚了。

## 29.2 world 必须长什么样

```ts
// packages/eve/src/internal/workflow/validate-world.ts:23
export function validateWorkflowWorld(input: ValidateWorkflowWorldInput): asserts input is {
  readonly packageName?: string;
  readonly world: World;
} {
  assertConfiguredWorldCompatibility(input.packageName);

  if (!isWorkflowWorld(input.world)) {
    throw new Error("Configured Workflow world factory did not return a valid World.");
  }
}
```

`asserts` 关键字让它成为类型守卫——校验通过后，TypeScript 知道 `input.world` 是 `World`。

`isWorkflowWorld` 做形状检查（有哪些必需方法）。具体的接口定义在 `@workflow/world` 里，eve 不重新定义。

## 29.3 版本兼容：只比 major 和预发布标签

```ts
// packages/eve/src/internal/workflow/world-compatibility.ts
const WORKFLOW_COMPATIBILITY_PACKAGES = ["@workflow/core", "@workflow/world"] as const;

export interface WorkflowWorldManifest {
  readonly dependencies?: Readonly<Record<string, string>>;
  readonly peerDependencies?: Readonly<Record<string, string>>;
}

function parseVersionLine(value: string): VersionLine | undefined {
  const match = /(\d+)\.(?:\d+|x|\*)(?:\.(?:\d+|x|\*))?(?:-([0-9A-Za-z.-]+))?/.exec(value.trim());
  if (match === null) return undefined;

  const major = Number(match[1]);
  if (!Number.isInteger(major)) return undefined;

  const prerelease = match[2];
  const prereleaseTag = prerelease === undefined ? undefined : prerelease.split(".")[0];

  return { major, prereleaseTag };
}
```

检查逻辑：

1. 读 world 包的 `package.json`。
2. 找它声明的 `@workflow/core` 或 `@workflow/world` 依赖（按顺序，先找到先用）。
3. 解析版本，只取 **major 号**和**预发布标签**。
4. 和 eve 内置的 `@workflow/core` 版本比对。

为什么不用完整 semver 解析？注释写得很直白：

> This is intentionally tiny: eve only needs the major number and any prerelease tag to detect a definite line mismatch, so we avoid pulling in a full semver parser.

只检测「明确不兼容」，不做「可能兼容」的推断。major 不同 = 一定不兼容；预发布标签不同（比如 `1.0.0-alpha` vs `1.0.0-beta`）= 一定不兼容。其余情况放行。

这个策略的好处是不会因为一个补丁版本差异就拒绝启动。代价是真正的不兼容可能漏过去——所以 `installWorkflowWorld` 之后还有运行时的错误处理兜底。

## 29.4 stranded：部署交接的真正风险

这是开头那个故障的答案，也是整章最重要的部分。

```ts
/**
 * Makes the World acknowledge, without replaying, queue deliveries for
 * top-level eve session runs that another eve version started. Worlds that
 * re-enqueue their active runs at startup (local, Postgres) would otherwise
 * replay such a run on this build's code and fail it as `CORRUPTED_EVENT_LOG`.
 * A top-level session stays active until ingress replaces it or its timeout
 * ends it. Subagent sessions and workflow tool runs are cancelled instead:
 * no later user message can replace their interrupted work. Ingress refuses
 * the same owners by the same verdict, so preserved owners cannot admit work.
 * Timeout workflows remain runnable so they can end preserved sessions.
 *
 * Mutates and returns `world`; only its `createQueueHandler` changes. Worlds
 * that replay each run on its own deployment are returned untouched.
 */
export function guardStrandedSessionReplay<TWorld extends World>(world: TWorld): TWorld {
  if (routesToOwnerDeployment(world)) return world;
  const { name } = resolveInstalledPackageInfo();
  const guardedWorkflows = new Map<string, GuardedWorkflow>([
    [`workflow//${name}//${WORKFLOW_ENTRY_NAME}`, "session"],
    [`workflow//${name}//${WORKFLOW_TOOL_RUN_WORKFLOW_NAME}`, "tool"],
  ]);
  const createQueueHandler = world.createQueueHandler.bind(world);
  world.createQueueHandler = (queueNamePrefix, handler) =>
    createQueueHandler(queueNamePrefix, async (message, meta) => {
      const workflow = guardedWorkflows.get(meta.queueName.slice(queueNamePrefix.length));
      const runId = readRunId(message);
      if (workflow !== undefined && runId !== undefined && (await skipRetiredRun(world, runId, workflow))) {
        return undefined;
      }
      return await handler(message, meta);
    });
  return world;
}
```

先把问题说清楚。

**场景**：你在跑 v1，一个会话正在等审批。这时候部署 v2。

- v1 的进程被替换，但那个会话的 run 还在 world 的队列里，状态是「活跃」。
- local world 和 Postgres world 在启动时会**重新入队所有活跃的 run**。
- v2 的进程拿到这个 run，用 v2 的代码重放 v1 的检查点。
- 检查点里的事件格式是 v1 的，v2 的代码解析不了，报 `CORRUPTED_EVENT_LOG`。

**解决**：在队列投递前检查这个 run 是不是「别的版本创建的」。是就**确认收到但不重放**（acknowledge without replaying）。

三类 run 的处理不一样：

| run 类型 | 处理 | 理由 |
| --- | --- | --- |
| 顶层 session（`workflowEntry`） | 确认不重放 | 用户的下一条消息会替换它，或者超时结束它 |
| 子代理 session | **取消** | 没有后续用户消息能替换它，留着就是死锁 |
| workflow tool run | **取消** | 同上 |
| timeout workflow | 保持可运行 | 它要负责结束那些被保留的 session |

「确认不重放」是什么意思：队列消息被 ack，不再投递，但 run 本身不取消。它处于「保留」状态，等外部事件（新消息、超时）来处理。

`skipRetiredRun` 判断一个 run 是否属于当前版本：

```ts
async function skipRetiredRun(world: World, runId: string, workflow: GuardedWorkflow): Promise<boolean> {
  const run = await readRetiredRun(world, runId);
  if (run === undefined) return false;
  switch (strandedReplayAction(run, workflow)) {
  // ...
}
```

`readRetiredRun` 读 run 的属性，比对 `EVE_VERSION_ATTRIBUTE`（run 创建时写入的 eve 版本）。版本不匹配就是「retired」。

这个机制解释了第 10 章的一个行为：**部署交接期间消息立即接受、地址始终有主**。因为旧 run 被保留而不是取消，新消息进来时可以接管它。

`routesToOwnerDeployment(world)` 那个提前返回也值得注意：有些 world 实现能把每个 run 路由到它原来的部署（比如 Vercel 的 world）。这种 world 不需要 guard，因为旧 run 会被路由回旧部署，不会在新代码上重放。

## 29.5 dev 下的 world

`.eve/.workflow-data/` 里只有一个 `version.txt`：

```text
.eve/.workflow-data/version.txt
```

会话数据不在这里。dev 模式用的是 `development-world-server`：CLI 父进程里跑一个 world 服务，子进程通过 HTTP 或 IPC 访问它。

这么设计的原因：dev 下每次重建都产生新快照（第 22 章），如果把数据放在快照目录里，每次重建都会丢会话。放在父进程里，重建不影响数据。

对应地，`eve dev` 退出时 world 服务也退出。想保留会话跨重启，要用 `--resume`（第 20 章的 `existing` 参数），或者改用 Postgres world。

## 29.6 回到那次部署故障

现在能说清整个过程了。

v1 的会话在等审批，run 处于活跃状态。部署 v2 后，v2 的 world 启动，队列重新入队这个 run。

`guardStrandedSessionReplay` 的包装生效：`createQueueHandler` 先查这个 run 的 `EVE_VERSION_ATTRIBUTE`，发现是 v1 的，于是 ack 但不重放。

那个会话在新版本里变成「保留」状态。用户下次发消息，ingress 用同样的判定拒绝旧 owner，然后创建一个新 run 接管。

所以正确的观察是：**会话看起来「卡住」了几秒，然后恢复**，而不是报 `CORRUPTED_EVENT_LOG`。

如果你看到了 `CORRUPTED_EVENT_LOG`，说明 guard 没生效。三种可能：

1. 你用的 world 不支持 `createQueueHandler` 包装（自定义 world 常见）。
2. 你用了 `routesToOwnerDeployment` 返回 true 的 world，但实际路由没生效。
3. 版本判定读不到 `EVE_VERSION_ATTRIBUTE`（run 是很老的版本创建的，没写这个属性）。

## 29.7 五个坑

1. **包装必须在 `world.start()` 之前。** 顺序错了，队列可能已经投递了 retired run。
2. **版本检查只比 major 和预发布标签。** 补丁版本不兼容不会被拦，只能靠运行时错误暴露。
3. **子代理 run 会被取消，不是保留。** 部署交接时正在跑的子代理任务会丢失，不会恢复。
4. **`.eve/.workflow-data` 里没有会话数据。** 想备份 dev 会话，备份那个目录没用。
5. **`routesToOwnerDeployment` 为真的 world 不走 guard。** 如果你自己实现 world 且支持按 run 路由到原部署，记得实现这个能力，否则会走保守路径。

## 29.8 练习

1. 打开 `.eve/.workflow-data/version.txt`，看里面是什么。再 `ls .eve/` 找会话数据实际存在哪。
2. 在 `agent/agent.ts` 里配一个不存在的 world 包名，跑 `eve dev`，记录报错信息。
3. 读 `strandedReplayAction` 的实现，列出它对三类 run 分别返回什么动作。
4. `eve dev` 起来后跑一轮对话，`kill -9` 父进程，重启后看会话是否还在。再用 `--resume` 试一次。

## 收尾：一条完整的链路

到这里，源码解析部分走完了。

```text
CLI（20）→ discover（21）→ compiler（22）→ runtime（23）
  → workflow（24）→ harness（25）→ eve channel（26）
  → channel 抽象（27）→ 鉴权与协议（28）→ world（29）
```

回顾一下这条链路上每一环回答的问题：

| 章 | 问题 |
| --- | --- |
| 19 | 1816 个文件，从哪读起 |
| 20 | 敲下 `eve dev` 之后发生了什么 |
| 21 | 文件名怎么变成能力名，为什么有些文件没反应 |
| 22 | 声明怎么变成产物，为什么改代码老会话不变 |
| 23 | 产物怎么变成活对象，模块级状态为什么串会话 |
| 24 | 会话为什么是一串 step，为什么工具会重跑 |
| 25 | 一个 turn 里模型和工具怎么循环 |
| 26 | HTTP 表面长什么样，401 和断流怎么排查 |
| 27 | 平台通道怎么复用同一套抽象 |
| 28 | 请求凭什么被信任，事件协议怎么演进 |
| 29 | 检查点存在哪，部署交接为什么不丢消息 |

每一章都回答同一个问题的不同侧面：**这段代码为什么必须这么写，出问题时该看哪里**。

源码会变，函数名和目录结构大体稳定。读的方法（第 19 章）比具体结论更耐用。

---

返回：[教程目录](README.md) · 上一章：[28 鉴权与协议](28-auth-and-protocol.md)
