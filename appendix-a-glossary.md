# 附录 A 术语与速查

## A.1 概念对照

| 术语 | 一句话 | 详细 |
| --- | --- | --- |
| session | 一整段持久对话，默认 30 天 | 第 1、10 章 |
| turn | 一条用户消息引发的全部工作 | 第 1、9 章 |
| step | turn 内的检查点，重放单位 | 第 10 章 |
| principal | 被认证的调用者，`auth.current` | 第 14 章 |
| scope | 记忆的分区依据 | 第 8 章 |
| namespace | 记忆的应用级域，scope 之前 | 第 8 章 |
| slot | 一个记忆槽（provider + scope + namespace） | 第 8 章 |
| park / held | 工作流挂起，不占算力。`held` 是等人或等任务，`park` 是还有待派发的工作 | 第 10、11 章 |
| steer | 打断生成、并入当前 turn | 第 9 章 |
| stranded | 被别的 eve 版本构建的会话，跑不了 | 第 10 章 |
| task | 一次异步调用，返回收据，结果后到 | 第 12 章 |
| compaction | 上下文压缩，摘要老历史 | 第 6 章 |
| connection | 外部 MCP / OpenAPI 服务 | 第 13 章 |
| channel | 对外的入口（HTTP、Slack…） | 第 14 章 |
| extension | 可复用的能力包，挂载后带命名空间 | 第 18 章 |
| deferred | 不进模型工具/技能列表，靠搜索发现 | 第 4、7 章 |

## A.2 目录速查

```text
agent/
├── agent.ts                    defineAgent：model / reasoning / compaction / limits / build
├── instructions.md             系统提示（或 instructions.ts、instructions/）
├── instrumentation/            遥测 provider，一文件一个
├── channels/                   eve.ts / slack.ts / discord.ts / web.ts / 自定义
├── connections/                MCP / OpenAPI，文件名 = 连接名
├── extensions/                 挂载扩展
├── hooks/                      事件订阅
├── skills/                     *.md 或 <name>/SKILL.md
├── lib/                        辅助代码，不进沙箱
├── memory/                     <slot>.ts，defineMemory
├── sandbox/
│   ├── sandbox.ts              defineSandbox
│   └── workspace/              → /workspace
├── tools/                      文件名 = 工具名
├── schedules/                  cron 任务
└── subagents/<name>/           完整 agent 根目录
evals/
├── evals.config.ts             必需
└── **/*.eval.ts
```

## A.3 import 速查

| 要什么 | 从哪 import |
| --- | --- |
| `defineAgent` | `eve` |
| `defineTool`、`defineWorkflowTool`、`disableTool`、`toolResultFrom`、`toolOutput`、`toolOutputPart` | `eve/tools` |
| `always` / `once` / `never` / `auto`，以及 `ApprovalPolicy`、`ApprovalResponsePolicy` 等类型 | `eve/tools/approval` |
| `defineState`、`SessionAuthContext` 等类型 | `eve/context` |
| `defineMemory` | `eve/memory` |
| `fileMemory`、`inMemory`、`MemoryDocumentConflictError` | `eve/memory/file` |
| `vercelBlob` | `eve/memory/file/vercel` |
| `byPrincipal` | `eve/memory/scope` |
| `defineMemoryProvider` | `eve/memory` |
| `defineSkill`、`defineDynamic`（技能） | `eve/skills` |
| `defineInstructions`、`defineDynamic`（指令） | `eve/instructions` |
| `defineHook` | `eve/hooks` |
| `defineSchedule`、`isScheduleAuth` | `eve/schedules` |
| `defineSandbox`、`DefaultSandbox`、`defineParentSandbox` | `eve/sandbox` |
| `DockerSandbox` | `eve/sandbox/docker` |
| `JustBashSandbox` | `eve/sandbox/just-bash` |
| `VercelSandbox` | `eve/sandbox/vercel` |
| `defineSandboxProvider` | `eve/sandbox/provider` |
| `eveChannel`、`defaultEveAuth` | `eve/channels/eve` |
| `localDev`、`vercelOidc`、`none`、`httpBasic`、`jwtHmac`、`jwtEcdsa`、`oidc`、`placeholderAuth`、`verifyHttpBasic`、`withAuthChallenges`、`routeAuth`、`ForbiddenError`、`UnauthenticatedError` | `eve/channels/auth` |
| `defineChannel`、`disableRoute`、`GET`、`POST`、`WS`、`SessionStrandedError` | `eve/channels` |
| `slackChannel` | `eve/channels/slack` |
| `defineMcpClientConnection`、`defineOpenAPIConnection`、`defineInteractiveAuthorization` | `eve/connections` |
| `defineRemoteAgent`、`defineWorkspaceAgent` | `eve` |
| `vercelOidc`、`bearer`（出站鉴权） | `eve/agents/auth` |
| `defineEval`、`defineEvalConfig`、`mockModel` | `eve/evals` |
| `includes`、`equals`、`matches`、`satisfies`、`similarity` | `eve/evals/expect` |
| `Braintrust`、`JUnit` | `eve/evals/reporters` |
| `Client`、`transcriptReducer` | `eve/client` |
| `sessions` | `eve/server` |
| `useEveAgent`、`openConversationInputs` | `eve/react` |
| `withEve` | `eve/next` |
| `defineExtension` | `eve/extension` |
| `workflow`（运行时程序工具） | `eve/tools/workflow` |
| `askQuestion` | `eve/tools/ask_question` |
| `noReply` | `eve/tools/no_reply` |
| `sleep` | `eve/tools/sleep` |
| `anthropic` | `eve/models/anthropic` |
| `openai`、`chatgpt` | `eve/models/openai` |
| `sleep`、`createHook`、`createWebhook`、`FatalError`（workflow 主体内） | `workflow` |
| `start`、`getRun`、`resumeHook`（只能在 step 里） | `workflow/api` |

## A.4 常用代码片段

**读当前调用者**

```ts
const user = ctx.session.auth.current;
if (user?.principalType !== "user") throw new Error("需要登录用户");
const tenant = user.attributes.tenantId;
```

**按输入决定审批**

```ts
approval: ({ toolInput }) =>
  (toolInput?.amount ?? 0) > 1000 ? "user-approval" : "not-applicable",
```

**沙箱跑命令并检查退出码**

```ts
const r = await (await ctx.getSandbox()).run({ command: "python3 x.py" });
if (r.exitCode !== 0) throw new Error(r.stderr);
```

**幂等键**

```ts
`${ctx.session.id}:${ctx.callId}`
```

**把大输出收窄给模型**

```ts
toModelOutput(o) { return { type: "text", value: `共 ${o.total} 条，示例：${o.sample}` }; },
```

**每轮重置状态**

```ts
export default defineHook({
  events: { async "turn.started"() { state.update(() => initialState()); } },
});
```

**只允许发起人审批**

```ts
response: ({ request, response }) =>
  request.principal && response.principal.principalId === request.principal.principalId
    ? { status: "allowed" }
    : { status: "rejected" },
```

## A.5 常见错误与含义

| 错误 / 状态 | 含义 | 怎么办 |
| --- | --- | --- |
| `409 session_not_ready` | 收件箱还在启动 | 有界退避重试 |
| `409 session_not_active` | 会话已终止 | 新建会话 |
| `409 session_stranded` | 别的 eve 版本构建的会话 | reset 或走渠道地址重置 |
| `410 session_expired` | 超出保留期 | 数据已删，无法恢复 |
| `UNKNOWN_TASK` | `taskId` 不存在或已结束 | 不带 `taskId` 重新开 |
| `principal_required` | 用户作用域连接没有用户身份 | 通道认证用户，或改 app 作用域 |
| `SESSION_TOKEN_LIMIT_REACHED` | 无人值守会话超出 token 额度 | 调大 `limits` 或缩短任务 |
| `SESSION_TOKEN_COST_LIMIT_REACHED` | 超出成本额度 | 同上 |
| `MODEL_CALL_FAILED` + `content-filter` | 模型输出被内容过滤 | 不重试，改提示词 |
| `413` / `415` | 附件超限 / 类型不允许 | 调 `uploadPolicy` 或改走 URL |
| `401` on `/eve/v1/*` | 通道鉴权拒绝 | 配 `channels/eve.ts` 或带凭据 |

## A.6 三张对照表

### 技能 / 工具 / 子代理

| | 技能 | 工具 | 子代理 |
| --- | --- | --- | --- |
| 改变什么 | 模型的想法 | 世界 | 上下文 |
| 加执行能力 | 否 | 是 | 是（独立上下文） |
| 何时进上下文 | 按需加载 | schema 常驻 | 只回传结论 |
| 写在哪 | `skills/` | `tools/` | `subagents/` |

### state / memory / connection

| | defineState | memory 槽 | connection |
| --- | --- | --- | --- |
| 跨会话 | 否 | 是 | 不适用 |
| 模型可见 | 否 | 是（自动召回） | 是（工具） |
| 谁写 | 你的代码 | provider | 外部服务 |
| 适合 | 计数器、计划 | 偏好、事实 | 查询外部数据 |

### 通道 / 连接

| | 通道 | 连接 |
| --- | --- | --- |
| 方向 | 用户 → agent | agent → 服务 |
| 模型感知 | 否 | 是 |
| 鉴权对象 | 入站调用者 | 出站凭据 |

## A.7 版本与文档

- 本文档对照 `eve@0.81.x`。eve 处于 beta，API 会变。
- 升级后先看 `node_modules/eve/CHANGELOG.md`。
- 本地文档在 `node_modules/eve/docs/`，与安装版本严格对齐，比 eve.dev 更可信。
- eve.dev 的 `/llms.txt` 是索引，`/llms-full.txt` 是全量。

### 文档与类型不一致时

| 项 | 现象 | 怎么处理 |
| --- | --- | --- |
| `sandbox.readTextFile()` | 文档的方法表没列全，类型里有，签名 `{ path, encoding?, startLine?, endLine? }` | 以类型为准，可以直接用 |
| `onMessage(ctx, message)` | 文档示例传两个参数，`EveMessageContext` 类型只声明了第一个 | 两个都行，第二个参数是消息文本 |

核对方法：`node_modules/eve/dist/src/**/*.d.ts` 是最终事实来源。文档与类型冲突时，以类型为准，然后去 CHANGELOG 找是不是版本差异。

---

返回：[教程目录](README.md) · 附录 B：[事件与 CLI](appendix-b-events-and-cli.md)
