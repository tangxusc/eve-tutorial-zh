# eve 中文教程

本仓库是 [eve](https://eve.dev) 的中文教程，只有文档，没有可执行代码。18 章正文加 3 个附录，从 `eve init` 讲到上线。

写给要动手做事的人。每一章都能跑起来，讲清楚「为什么这样设计」，也讲清楚哪里会踩坑。

- 依据：[eve.dev](https://eve.dev) 官方文档与 [github.com/vercel/eve](https://github.com/vercel/eve) 源码，逐条对照 `eve@0.81.1` 的类型定义（`dist/src/**/*.d.ts`）与本地 `node_modules/eve/docs/`
- 对照版本：`eve@0.81.x`（beta）。API 仍在变，升级后先看 `node_modules/eve/CHANGELOG.md` 和本地 `node_modules/eve/docs/`，那份文档跟安装版本严格对齐。
- 示例项目：全教程共用一个「值班助手 opsbot」，从一句话提示词长到带工具、沙箱、审批、子代理、Slack、定时任务和评测的完整 agent。

### 相对 0.79 的行为变化

如果你手上的项目还钉在 0.79，下面这些差异会影响教程里的代码和结论。已在对应章节标注。

| 变化 | 出现在 | 章节 |
| --- | --- | --- |
| `ctx.approval.responder` 可用：能读审批人身份，也能用 `responder.getToken()` 取他的凭据 | 0.79.1 | 4、11 |
| 审批的第一个答案生效，重复提交被丢弃（双击不会再覆盖已做的决定） | 0.81 | 11 |
| 登录中断的调用在回调后自己续跑，不再多一次模型调用；`retryable: false` 失败不再在同一 turn 重复弹登录 | 0.81 | 13 |
| MCP 凭据每次请求前解析，`expiresAt` 过期即换；服务端 401 会驱逐缓存并重试一次，403 不重试 | 0.81 | 13 |
| MCP 连接可以订阅服务端事件（`experimental_events`） | 0.80 | 13 |
| 部署/压缩交接期间消息立即接受、地址始终有主，不再返回 `409 session_not_ready`；需要 world 具备 `hookForceClaim` / `hookRetention` | 0.81 | 10 |
| run 数据保留默认由追踪策略推导；私有会话在开发环境外默认结束即删 | 0.80 | 10、18 |
| 读超出保留期的会话流返回 `410 session_expired` | 0.79.1 | 9、附录 A |
| `useEveAgent` 新增 `followIdle: false`，空闲时关流 | 0.79.1 | 17 |
| 模型支持 AI Gateway 的会话亲和，可用 `sessionAffinity: false` 关掉 | 0.81 | 2 |
| OpenAPI path 参数拒绝 `.` 和 `..` | 0.80.1 | 13 |
| 评测可以断言推理内容（`reasoning.completed`） | 文档 0.81 | 16 |
| 扩展的 git 包支持 `github.authorize`（0.80）/ `github.attribution`（0.81） | 0.80、0.81 | 18 |

核对方法：`node_modules/eve/dist/src/**/*.d.ts` 是最终事实来源；文档与类型冲突时以类型为准，再翻 CHANGELOG 确认是不是版本差异。教程里所有 `import ... from "eve/..."` 都在包的 `exports` 里存在，具名导出都能在对应 `.d.ts` 里找到。

## 先读这三段

**eve 是什么。** 一个「文件系统即配置」的 agent 框架。`agent/` 目录下的文件名就是能力名：`tools/read_logs.ts` 就是 `read_logs` 工具，`skills/triage.md` 就是 `triage` 技能。没有注册表，没有 DSL，没有手写的 agent 循环。你只写能力，循环由框架跑。

**它跟常见 agent 框架的差别。** 大多数框架给你一个循环，持久化、沙箱、审批、多渠道要自己接。eve 把这四件事做成默认行为：每个 session 是一条持久 workflow，每个 step 有检查点，崩溃后从最后一个完成的 step 续跑；每个 agent 有独立沙箱；工具可以挂人工审批并停在那里等人；同一份代码能同时挂到 HTTP、Slack、Discord、定时任务上。

**学它的正确顺序。** 先建立 session / turn / step 的心智模型（第 1、10 章），再学工具和沙箱（第 4、5 章），然后才是上下文分层（第 6～8 章）。跳过心智模型直接抄代码，会在「为什么我的工具被重跑了两次」「为什么审批之后上下文没了」这类问题上卡很久。

## 目录

### 第一部分：入门

| 章节 | 讲什么 | 读完能做 |
| --- | --- | --- |
| [01 认识 eve](01-what-is-eve.md) | 设计哲学、三层执行模型、与其它框架的取舍 | 说清 eve 适合什么、不适合什么 |
| [02 第一个 agent](02-hello-agent.md) | `eve init`、`instructions.md`、模型配置（含 DeepSeek 等国产模型）、dev TUI | 跑起来并改掉它的行为 |
| [03 agent 目录解剖](03-project-anatomy.md) | 每个槽位、命名规则、默认值覆盖、构建产物 | 知道什么该放哪、什么能改 |

### 第二部分：核心能力

| 章节 | 讲什么 | 读完能做 |
| --- | --- | --- |
| [04 工具](04-tools.md) | `defineTool` 全部字段、schema、输出整形、流式、幂等 | 写出生产可用的工具 |
| [05 沙箱](05-sandbox.md) | workspace 种子、`prepare`、provider、`ctx.getSandbox()` | 让 agent 安全地跑命令和算数据 |
| [06 上下文控制](06-context-control.md) | 指令 / 技能 / 工具 / 工作区 / 记忆怎么分工，压缩与清空 | 控制 token 成本和行为稳定性 |
| [07 技能](07-skills.md) | `SKILL.md`、打包技能、deferred、动态技能 | 把长流程从系统提示里搬出去 |
| [08 状态与长期记忆](08-state-and-memory.md) | `defineState`、memory slot、scope、自建 provider | 分清「本轮记忆」和「跨会话记忆」 |
| [09 会话协议](09-sessions-and-streaming.md) | HTTP 路由、NDJSON 事件、Client SDK、steering | 用任何语言接上 agent |
| [10 持久化执行](10-durability.md) | 检查点、重放、停泊、workflow world（local / Postgres / 自定义）、部署交接、stranded session | 正确设计幂等和恢复策略 |
| [11 人机协同](11-human-in-the-loop.md) | approval 的两种形态、四种策略、response policy、`ask_question`、`defineWorkflowTool` 三个入口 | 让危险动作必须过人，长流程能挂起 |

### 第三部分：扩展与上线

| 章节 | 讲什么 | 读完能做 |
| --- | --- | --- |
| [12 子代理](12-subagents.md) | 内置 `agent` 工具 vs 声明式子代理、隔离边界、task | 拆分复杂任务而不污染主上下文 |
| [13 连接](13-connections.md) | MCP / OpenAPI、app 与 user 凭据、OAuth、凭据刷新、MCP 事件订阅 | 接第三方服务且不泄漏 token |
| [14 通道](14-channels.md) | eve HTTP 通道、鉴权链、Slack、自定义通道 | 让 agent 出现在用户所在的地方 |
| [15 定时与钩子](15-schedules-and-hooks.md) | cron 任务、事件钩子、事件落库 | 做日报、巡检、审计 |
| [16 评测](16-evals.md) | `defineEval`、断言、judge、tool stub | 改提示词前先有回归网 |
| [17 前端与部署](17-frontend-and-deploy.md) | `useEveAgent`、`withEve`、Vercel / 自托管 | 交付一个真实可用的应用 |
| [18 进阶与上线检查](18-advanced-and-production.md) | 动态能力、扩展包、远程 agent、安全与成本 | 上线前逐条过一遍 |

### 附录

- [A 术语与速查](appendix-a-glossary.md)：概念对照、目录速查、常见错误码
- [B 事件与 CLI](appendix-b-events-and-cli.md)：完整事件表、CLI 命令、HTTP 路由
- [C 环境变量与密钥](appendix-c-env-and-secrets.md)：`process.env` 在哪个进程里存在、`.env` 什么时候被读、编译期求值的坑、沙箱边界、排查表

## 怎么用这份教程

**第一遍：按顺序读 01 → 05。** 这五章是地基，后面的所有内容都在复用它们的概念。

**第二遍：按需跳。** 要做审批就看 11，要接 Slack 就看 14，要上线就看 17、18。每章开头有一句话结论，结尾有练习。

**动手建议。** 开一个空目录，跟着第 2 章建项目，之后每章的代码直接往里加。教程里的示例是连续的，最后会得到一个能跑的完整 agent。

## 约定

- 代码块标题里的路径都是相对项目根目录，例如 `agent/tools/read_logs.ts`。
- 命令默认在项目根目录执行。
- 流程图用 Mermaid 写（代码块语言标 `mermaid`），GitHub、VS Code、Obsidian 都能直接渲染；纯文本终端里会显示成代码。
- 「坑」标记的是官方文档里散落各处、但实际会让人卡住的点，不是猜测。
- 涉及钱的、涉及删除的、涉及对外发消息的例子，一律标了审批。这不是啰嗦，是 eve 的设计意图：默认不信任模型。

## 一句话版本

如果你只有五分钟：读第 1 章的表格，跑第 2 章的 `npx eve@latest init`，然后把第 4 章的 `defineTool` 例子抄进去。剩下的等遇到问题再查。
