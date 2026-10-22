# 附录 B 事件与 CLI

## B.1 完整事件表

事件流是 NDJSON，一行一个。`data` 是载荷，`meta` 是信封（`id`、`at`，有时有 `deliveryIds`）。

### 会话与 turn

| 事件 | 何时发 | 关键载荷 |
| --- | --- | --- |
| `session.started` | 会话建立 | `trace?`、`predecessor?` |
| `turn.started` | 新 turn | `turnId`、`trace?` |
| `message.received` | 输入到达 | 扁平文本 + 结构化 parts |
| `turn.waiting` | turn 停泊 | `turnId`、`on: "input" \| "tasks"` |
| `turn.completed` | turn 完成 | — |
| `turn.failed` | turn 失败 | `{ code, message, details?, terminal? }` |
| `turn.cancelled` | turn 被取消 | 后面必跟 `session.waiting` |
| `session.waiting` | 会话空闲 | `usage` |
| `session.failed` | 会话失败 | — |
| `session.completed` | 会话终结 | — |

### 模型步骤

| 事件 | 何时发 | 关键载荷 |
| --- | --- | --- |
| `step.started` | 模型调用开始 | `modelId`、`stepIndex` |
| `step.completed` | 模型调用结束 | `finishReason`、`usage` |
| `step.failed` | 模型调用失败 | `{ code, message, details? }` |
| `reasoning.appended` | 推理增量 | `reasoningDelta` |
| `reasoning.completed` | 推理定稿 | 全文 |
| `message.appended` | 文本增量 | `messageDelta` |
| `message.completed` | 文本定稿 | `message`、`finishReason`、`sequence` |
| `result.completed` | 结构化结果 | `result`（请求了 outputSchema 时） |

### 动作与工具

| 事件 | 何时发 | 关键载荷 |
| --- | --- | --- |
| `actions.requested` | 模型请求动作 | `actions[]`，执行前就流出 |
| `action.input.appended` | 工具输入流 | `inputTextDelta`、`callId` |
| `action.partial` | 生成器工具中间快照 | 完整快照，后写覆盖前写 |
| `action.result` | 工具返回 | `result`（含 `toolName`、`isError`、`output`） |
| `input.requested` | 停下来等人 | `requests[]`（`kind`、`requestId`、`options`） |
| `input.resolved` | 人已应答并被接受 | `resolutions[]`（`outcome`：`answered` / `approved` / `cancelled` / `denied` / `ignored` / `invalid`） |
| `approval.candidate` | 有人提交了审批，服务端开始校验 | `candidateId`、`requestId`、`outcome`（`pending` / `rejected` / `failed` / `timed-out` / `stale`）、`responderPrincipalId`、`reason?` |
| `approval.settled` | 审批最终落定 | `requestId`、`outcome`（`approved` / `cancelled`）、`responderPrincipalId` |
| `authorization.required` | 连接需要 OAuth | `authorization`（`url` / `userCode` / `instructions`） |
| `authorization.completed` | 授权结束 | `outcome`：`authorized` / `declined` / `failed` / `timed-out` |

### 任务与子代理

| 事件 | 何时发 | 关键载荷 |
| --- | --- | --- |
| `task.started` | 调用开始一个任务 | `taskId`、`callId`、`turnId`、`name`、`kind` |
| `task.settled` | 任务结束 | `taskId`、`status`、`output` 或 `error.message` |
| `agent.started` | 子会话打开 | `sessionId`、`streamPath`、`taskId?`、`callId`、`turnId` |

### 上下文

| 事件 | 何时发 | 关键载荷 |
| --- | --- | --- |
| `compaction.requested` | 压缩开始 | `modelId`、`usageInputTokens` |
| `compaction.completed` | 检查点写入 | 同上 |
| `context.cleared` | 清空历史完成 | — |

## B.2 事件顺序：一次带审批的调用

```text
session.started
turn.started              turn_0
message.received
step.started
actions.requested          restart_service
input.requested            tool-approval, req_A
turn.waiting               on: "input"
                          ← 人点批准
approval.candidate         req_A, outcome: pending      ← 只有配了 response 策略才有
approval.settled           req_A, outcome: approved
input.resolved             req_A → approve
step.started               （同一个 turn_0，续跑）
action.result              restart_service 完成
step.completed
message.completed          finishReason: "stop"
turn.completed
session.waiting
```

没有 `response` 策略时，审批直接落定，不会出现 `approval.candidate`——只看到 `input.resolved`。

## B.3 HTTP 路由

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/eve/v1/health` | 公开，`{ ok, status, workflowId }` |
| `GET` | `/eve/v1/info` | 编译后 agent 快照（agent-info v6） |
| `POST` | `/eve/v1/session` | 建会话，可带 `message` / `operationId` |
| `POST` | `/eve/v1/session/:id` | 追问或 `inputResponses` |
| `POST` | `/eve/v1/session/:id/cancel` | 取消当前 turn |
| `POST` | `/eve/v1/session/:id/clear` | 清历史 |
| `POST` | `/eve/v1/session/:id/compact` | 压缩 |
| `POST` | `/eve/v1/session/:id/reset` | 终止 |
| `GET` | `/eve/v1/session/:id/stream` | NDJSON，`startIndex` / `follow` / `includeTailIndex` |
| `POST` | `/eve/v1/dev/schedules/:id` | **仅 dev**，手动触发定时任务 |
| `POST` | `/eve/v1/slack` 等 | 平台通道的 webhook |

## B.4 CLI

| 命令 | 作用 |
| --- | --- |
| `eve init [target]` | 建项目 / 给已有项目加 agent |
| `eve dev` | 本地服务 + 终端 UI |
| `eve build` / `eve start` | 构建 / 启动生产服务 |
| `eve info` | 检查本地应用发现结果 |
| `eve set model [id] [--reasoning x]` | 改模型 |
| `eve eval [pattern]` | 跑评测 |
| `eve add [item]` | 装通道 / 工具 / 技能 / 连接 |
| `eve registry search <q>` | 搜注册表 |
| `eve link` / `eve deploy` | 关联 / 部署 Vercel |
| `eve remote connect --url <u>` | 连远端 agent 的 TUI |
| `eve remote invoke --url <u> [prompt]` | 一次性调用远端 |
| `eve remote info --url <u>` | 检查远端 |
| `eve logs list` / `eve logs show [id]` | 本地日志 |
| `eve traces list` / `eve traces show [id]` | 本地 trace |
| `eve extension init` / `build` | 扩展包 |
| `eve acp [url]` | 通过 ACP over stdio 暴露 agent |
| `eve telemetry disable` | 关 CLI 遥测 |

`eve` 不带子命令时：不在项目里就跑 `init`，在项目里就跑 `dev`。

### 常用 `eve add` 目标

```bash
eve add channel/web
eve add channel/slack
eve add tool/glob
eve add tool/grep
eve add tool/ask_question
eve add tool/no_reply
eve add tool/sleep
eve add memory/supermemory
```

## B.5 环境变量

常用几个列在这里，完整规则（哪些命令读 `.env*`、编译期求值、沙箱边界、泄漏面）见[附录 C](appendix-c-env-and-secrets.md)。

| 变量 | 用途 |
| --- | --- |
| `AI_GATEWAY_API_KEY` | AI Gateway 模型 |
| `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` | 直连 provider |
| `EVE_TELEMETRY_DISABLED=1` | 单次命令关遥测 |
| `AI_AGENT=eve` | 沙箱命令里自动注入，标识 agent 环境 |

## B.6 模型配置速查

```ts
// Gateway 字符串
model: "anthropic/claude-sonnet-5"

// 直连（内置 helper，读对应 API key）
model: anthropic()            // claude-sonnet-5
model: openai()               // gpt-6-luna-fast
model: chatgpt()              // 本地订阅，不能部署

// 本地 OpenAI 兼容服务
const ollama = createOpenAICompatible({ name: "ollama", baseURL: "http://localhost:11434/v1", includeUsage: true });
model: ollama("gpt-oss:20b"),
modelContextWindowTokens: 64_000,
```

`reasoning` 可选值：`"provider-default"`、`"none"`、`"minimal"`、`"low"`、`"medium"`、`"high"`、`"xhigh"`。

## B.7 限制字段速查

| 字段 | 默认 |
| --- | --- |
| `maxInputTokensPerSession` | 根会话 40,000,000 |
| `maxOutputTokensPerSession` | 不设 |
| `maxTokenCostUsdPerSession` | 不设 |
| `sessionTimeoutMs` | 30 天 |
| `compaction.thresholdPercent` | 0.9 |
| `experimental.workflow.modelCallsPerStep` | 1 |
| `experimental.workflow.retention` | 由追踪策略推导 |

任何限制设成 `false` 表示不设限。

---

返回：[教程目录](README.md) · 附录 A：[术语与速查](appendix-a-glossary.md)
