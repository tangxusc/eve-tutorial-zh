# 源码解析部分规划（第 19 章起）

> 状态：**10 章全部完成**（2026-10-22）。对照源码：`github.com/vercel/eve` tag `eve@0.81.1`，与正文一致。
> 本地路径参考：`packages/eve/src/**`（TypeScript 源），发布包里是 `dist/src/**`（编译产物 + `.d.ts`）。

## 一、为什么要加，加在哪

现有 18 章回答「怎么用」和「为什么这么设计」。缺的是第三层：**这些结论在代码里的哪一行成立，出问题时该看哪个文件**。

源码解析部分定位为第五部分，接在第 18 章之后。它不是「逐行翻译源码」，只做三件事：

1. **指出决策点**：一段代码为什么必须这么写，换一种写法会破坏什么不变量。
2. **给出定位路径**：某个现象（工具被重跑、审批后上下文没了、消息 409）对应哪个函数。
3. **暴露失败路径**：正常流程之外的错误分支、降级分支、兼容分支。

不写的内容：内部工具函数逐行解释、测试文件、Vercel 平台专属逻辑、TUI 渲染细节。

## 二、主线：一个 prompt 的完整旅程

章节顺序不按目录结构排，按「一次请求经过哪些层」排。读者跟着一条消息从终端走到模型再走回来，每章都有前因后果。

```text
npx eve dev
  └─ 20 CLI            进程、子进程、TUI、命令注册
       └─ 21 discover  扫描 agent/ 目录 → manifest
            └─ 22 compiler  manifest → module map → .eve/ 产物
                 └─ 23 runtime  module map → 活的 agent 对象
                      └─ 24 workflow  会话被切成 step，每步落检查点
                           └─ 25 harness  turn 内模型 ↔ 工具循环
                                └─ 26 eve channel  HTTP 表面：11 条路由
                                     └─ 27 channel 抽象  平台通道如何复用
                                          └─ 28 鉴权与协议  谁能调、事件怎么编码
                                               └─ 29 world  检查点存在哪、怎么恢复
```

第 19 章是方法论，不进主线。

## 三、章节清单

### 19 怎么读 eve 的源码

**回答**：源码在哪、`src` 和 `dist` 怎么对应、`#cli/xxx.js` 这种导入怎么解析、怎么确认一段代码真的被执行了。

| 源码 | 行数 | 讲什么 |
| --- | --- | --- |
| `package.json` | — | `imports` 的 `#*` 别名映射、`exports` 的公开面 |
| `bin/eve.js` | 250 | 入口脚本做了什么（详见 20 章） |
| `src/internal/logging.ts` | — | `createLogger` 的命名空间、日志级别从哪来 |
| `src/index.ts` | — | 对外导出的边界：什么算公开 API |

**关键结论**（已实测验证）：
- 发布包 `dist/src/**/*.js` **是压缩过的单行代码**（`dist/src/cli/run.js` 全文 1 行 9895 字节，变量名缩成 `e`/`t`/`n`），且没有 `.map`。**不能用来读实现**。
- 发布包 `dist/src/**/*.d.ts` 保留完整 JSDoc 和类型签名，**可读性最好**，且与安装版本严格一致。
- 结论：读实现去 GitHub 对应 tag，读契约读本地 `.d.ts`，两者对照。
- `#compiled/*` 是 vendored 依赖（commander、semver、opentelemetry），不是 eve 自己的代码。
- 判断「这段代码是否在跑」的三个手段：`EVE_LOG_LEVEL=debug`、`.eve/logs/*.dump`、`node --inspect` 断点。

**验证实验**：把 `src/internal/logging.ts` 的默认级别改成 `debug`，重跑 `eve dev`，对比输出量。

---

### 20 CLI：从 `npx eve` 到 dev server

**回答**：`eve` 不带参数时为什么有时 `init` 有时 `dev`？`eve dev` 为什么起子进程？TUI 和 server 怎么通信？`eve build` 的产物落在哪？

| 源码 | 行数 | 讲什么 |
| --- | --- | --- |
| `bin/eve.js` | 250 | Node 版本门禁、`ensureBuiltCli` 按需自编译、`runCli` 调用、错误 cause 链打印、退出前 flush |
| `src/cli/run.ts` | 449 | commander 注册、全部命令懒加载、`argv` 为空时的分支判断 |
| `src/cli/dev/command.ts` | 344 | `eve dev` 的参数、UI 模式选择、启动顺序 |
| `src/cli/dev/local-server-process.ts` | 254 | 子进程拉起、端口分配、就绪探测、清理 |
| `src/cli/commands/init.ts` | 623 | 脚手架生成哪些文件、哪些步骤可恢复 |
| `src/cli/commands/build.ts` | 101 | 构建调用了谁 |
| `src/cli/shutdown.ts` | 89 | 信号处理、强制退出兜底 |

**关键片段**：
- `runCli` 里 `input.length === 0` 的三分支（找不到 app root → workspace 走 `dev`，否则 `init`）。
- `registerProjectCommands` 的 `.action(async () => { const { runX } = await import(...) })` 模式：为什么每个命令都动态导入。
- `bin/eve.js` 末尾的 `flushWritable` + `process.exit`：为什么不能直接 return。

**注意**：
- `ensureBuiltCli` 会在 `src` 比 `dist` 新时**重新编译整个包**，这是 workspace 开发时的行为，装 npm 包时不会触发。
- 子进程崩溃时父进程的表现，以及 `.eve/logs/` 里留下什么。

**验证实验**：`eve dev` 起两个终端，一个 `ps -ef | grep eve` 看进程树；`kill -9` 子进程，观察父进程退出码。

---

### 21 discover：文件系统怎么变成 manifest

**回答**：哪些文件会被扫？文件名怎么变成能力名？`tools/read_logs.ts` → `read_logs` 的规则写在哪？未知文件为什么被忽略？

| 源码 | 行数 | 讲什么 |
| --- | --- | --- |
| `src/discover/slots.ts` | 120 | 槽位表：目录名 → 能力类型 |
| `src/discover/grammar.ts` | 624 | 文件名语法、命名推导、保留名 |
| `src/discover/discover-agent.ts` | 559 | 扫描主流程、默认值注入、诊断收集 |
| `src/discover/manifest.ts` | — | 内存中的 manifest 结构 |
| `src/discover/diagnostics.ts` | — | 哪些问题只警告、哪些直接失败 |

**关键片段**：
- `slots.ts` 的槽位常量表（`tools`、`skills`、`channels`、`schedules`…）。
- `grammar.ts` 里从路径推导逻辑名的函数：为什么 `tools/foo/bar.ts` 不合法。
- `discover-agent.ts` 注入 `eve:defaults` 的位置——**这就是「不写 `agent/tools/bash.ts` 也有 bash 工具」的出处**。

**产物对照**：打开 `.eve/discovery/agent-discovery-manifest.json`，逐字段对回代码。

**验证实验**：把 `agent/tools/read_logs.ts` 改名成 `Read-Logs.ts`，看 manifest 里的名字和 `eve info` 的输出。

---

### 22 compiler：manifest 怎么变成可执行产物

**回答**：`.eve/compile/` 下四个文件各是什么？`module-map.mjs` 为什么一半静态 import 一半 programmatic loader？dev 下改文件后哪些会重建？

| 源码 | 行数 | 讲什么 |
| --- | --- | --- |
| `src/compiler/compile-agent.ts` | 207 | 编译主流程 |
| `src/compiler/manifest.ts` | 1387 | `bindings` / `owner` / `usage` / `revision` 的生成 |
| `src/compiler/module-map.ts` | 387 | module map 生成、`eve:defaults` 与 authored 模块的合并 |
| `src/compiler/source-graph.ts` | 668 | 依赖图、哪些文件算「源码输入」 |
| `src/compiler/normalize-*.ts` | 各 100-400 | 每类能力的归一化（tool / channel / schedule / skill / subagent） |
| `src/internal/nitro/dev-runtime-artifacts.ts` | — | HMR：snapshot、revision、suspend lease |

**关键片段**：
- `.eve/compile/module-map.mjs` 的真实内容（已在本机验证）：
  ```js
  import * as imported_module_0 from "../../agent/channels/eve.ts";
  const module_1 = await loadProgrammaticModule({"kind":"programmatic","moduleId":"sandbox.ts","registryId":"eve:defaults",...});
  ```
  解释：用户代码走静态 import（享受打包器优化），框架默认能力走 programmatic loader（不在用户目录里）。
- `manifest.ts` 里 `owner.kind === "framework"` 与 `"authored"` 的区别，以及它决定了什么。

**产物解剖**（表格，逐文件）：

| 文件 | 谁写 | 谁读 | 什么时候失效 |
| --- | --- | --- | --- |
| `compile/compiled-agent-manifest.json` | compiler | runtime | 源码变更 |
| `compile/module-map.mjs` | compiler | Node ESM loader | 同上 |
| `compile/compile-metadata.json` | compiler | dev HMR | 同上 |
| `discovery/agent-discovery-manifest.json` | discover | `eve info` | 目录结构变更 |
| `dev-runtime/current.json` | dev host | 客户端 | 每次重建 |

**注意**：`module-map.mjs` 里 `import "../../agent/channels/eve.ts"` 直接引 `.ts`，靠的是 Node 的类型剥离 + 自定义 loader；所以 Node 版本低于要求时会先挂在 `bin/eve.js` 的版本检查上。

**验证实验**：手动删 `.eve/compile/`，跑 `eve info`，看它是否自动重建。

---

### 23 runtime：module map 怎么变成活的 agent

**回答**：`resolve-agent.ts` 拿到 module map 后做了什么？工具模块顶层的代码执行几次？`ctx` 里的 key 是谁定义的？

| 源码 | 行数 | 讲什么 |
| --- | --- | --- |
| `src/runtime/resolve-agent.ts` | 307 | 装配主流程 |
| `src/runtime/resolve-tool.ts` | 207 | 单个工具怎么被实例化 |
| `src/runtime/resolve-channel.ts` | 126 | 通道怎么被挂上路由 |
| `src/compiler/module-lifecycle.ts` | 122 | 模块求值时机与缓存 |
| `src/context/key.ts` | — | durable context 的键空间 |
| `src/execution/effective-agent-config.ts` | — | manifest + 运行时覆盖 → 最终配置 |

**关键片段**：
- `module-lifecycle.ts` 里「模块只求值一次」的实现，以及它为什么必须如此（工具里的闭包状态）。
- `resolve-tool.ts` 里 `defineTool` 的返回值怎么被包成可调用对象：schema 校验在哪一层做。

**注意**：工具模块顶层的 `Date.now()` 或随机数会在**进程启动时**固定，不是每次调用重新算。这是「为什么我的工具总是拿到同一个时间戳」的根因。

**验证实验**：在工具模块顶层 `console.log(Math.random())`，连续调两次同一工具，看日志出现几次。

---

### 24 workflow：会话为什么是一串 step

**回答**：session 和 workflow 是什么关系？step 边界谁划的？检查点里存了什么？崩溃恢复时哪些代码会重跑？

| 源码 | 行数 | 讲什么 |
| --- | --- | --- |
| `src/execution/session.ts` | 325 | `createSession` / `projectToDurableSession` / `hydrateDurableSession` |
| `src/execution/node-step.ts` | 306 | 单个 step 的执行包装 |
| `src/execution/create-session-step.ts` | 89 | 建会话这一步 |
| `src/execution/publish-session-events.ts` | 600 | 事件怎么落库并广播 |
| `src/execution/workflow-registry.ts` | — | workflow 名字与版本 |
| `src/execution/stable-workflow-names.ts` | — | 为什么名字不能改 |

**关键片段**：
- `projectToDurableSession` / `hydrateDurableSession` 这对函数：序列化边界在哪，哪些字段故意不存。
- `node-step.ts` 里 step 开始/结束的钩子，以及失败时写什么。

**注意**：step 内的副作用（发邮件、写数据库）在重放时会**再执行一次**。这不是 bug，是 step 语义。第 10 章讲过结论，这里给代码证据。

**验证实验**：在 step 中间 `process.exit(1)`，重启后看 `.eve/logs` 里的重放记录和事件序号。

---

### 25 harness：一个 turn 里的模型 ↔ 工具循环

**回答**：一次 turn 最多几轮模型调用？工具并行还是串行？工具报错怎么回灌给模型？压缩在哪一步触发？

| 源码 | 行数 | 讲什么 |
| --- | --- | --- |
| `src/harness/tool-loop.ts` | 173 | 循环主体 |
| `src/harness/session-machine/transitions.ts` | 863 | 状态机：哪些状态能转到哪些 |
| `src/harness/execute-tool.ts` | 43 | 工具执行入口 |
| `src/harness/model-call/` | — | 请求组装、流式解析、usage 记账 |
| `src/harness/compaction/` | — | 压缩触发与摘要生成 |
| `src/harness/step-hooks.ts` | — | 每步前后的钩子 |

**关键片段**：
- `tool-loop.ts` 的循环条件：什么情况下退出（`finishReason` 的哪个值）。
- 并行工具调用的调度点：`Promise.all` 在哪，失败一个会不会取消其余。
- 工具错误进 `action.result` 时 `isError: true` 的构造位置。

**注意**：工具返回大对象会直接进上下文，`tool-output-serialization.ts` 只做截断不做摘要。

**验证实验**：写一个总是抛错的工具，观察模型收到什么文本、重试几次后放弃。

---

### 26 默认 eve channel：11 条路由的源码

**回答**：`eveChannel()` 返回的到底是什么？每条路由做什么？`startIndex` 怎么实现重放？`turnPolicy` 的 `steer` 和 `queue` 差在哪一行？

| 源码 | 行数 | 讲什么 |
| --- | --- | --- |
| `src/eve-channel/index.ts` | 750 | `eveChannel()` 构造、路由注册、鉴权包装 |
| `src/eve-channel/request.ts` | 724 | body 解析、continuation token、`startIndex` / `includeTailIndex` |
| `src/eve-channel/support.ts` | 240 | health、CORS、`onMessage`、stranded session 响应 |
| `src/eve-channel/types.ts` | — | `EveChannelInput` 全部字段 |
| `src/eve-channel/audience.ts` | — | audience 投影 |
| `src/protocol/routes.ts` | — | 路由常量（已在附录 B 列表，这里给源码位置） |

**关键片段**：
- `eveChannel` 里 `defineChannel({...})` 的完整参数，逐个对回第 14 章的字段表。
- 每个路由 handler 的第一行：`routeAuth` 怎么包住它。
- `request.ts` 里 `parseSessionMessageBody` 对 `inputResponses` 的处理。
- `prompt-queue.ts`（72 行）——`turnPolicy` 的真正实现，`steer` 是中断当前 turn，`queue` 是排队。

**注意**：`POST /eve/v1/session/:id` 在 turn 进行中返回的不是错误，取决于 `turnPolicy`；默认 `steer` 会打断。

**验证实验**：`curl` 打一遍 11 条路由，记录每条的状态码和响应体；把 `turnPolicy` 改成 `queue` 再打一次。

---

### 27 channel 抽象：平台通道怎么复用同一套

**回答**：`defineChannel` 和 `eveChannel` 什么关系？adapter 的 `deliver` 钩子在哪被调用？Slack 的 1529 行怎么组织？

| 源码 | 行数 | 讲什么 |
| --- | --- | --- |
| `src/channel/adapter.ts` | 303 | adapter 契约：`state` / `deliver` / 事件处理器 / `fetchFile` |
| `src/channel/routes.ts` | 304 | 通道路由注册 |
| `src/channel/compiled-channel.ts` | — | 编译后的通道形态 |
| `src/channel/cross-channel-receive.ts` | 200 | 跨通道投递 |
| `src/channel/session-callback.ts` | 103 | 平台回调入口 |
| `src/channel/websocket-upgrade-server.ts` | — | WebSocket 升级 |
| `src/public/channels/slack/slackChannel.ts` | 1529 | 对照案例：真实平台通道的结构 |

**关键片段**：
- `adapter.ts` 的 `deliver?(payload, ctx): StepInput | void`——返回值怎么覆盖 harness 看到的输入。
- `callAdapterEventHandler` 的 try/catch：为什么事件处理器抛错被吞掉。
- `slackChannel.ts` 的目录切分：`api.ts` / `interactions.ts` / 主文件各管什么。

**注意**：adapter 的 `state` 在每个 step 边界自动 JSON 快照，放非可序列化对象会在恢复时丢掉。

**验证实验**：写一个 20 行的自定义通道，只实现 `deliver`，把入站消息改写成大写，观察模型收到什么。

---

### 28 鉴权、身份与协议

**回答**：`auth` 数组在哪遍历？401 的 `WWW-Authenticate` 谁生成？`forwardedPrincipal` 什么时候可信？事件类型为什么是 2080 行的判别联合？

| 源码 | 行数 | 讲什么 |
| --- | --- | --- |
| `src/public/channels/auth.ts` | 1280 | 全部 helper 的实现 |
| `src/channel/auth/http-basic.ts` | 98 | 一个 helper 的完整实现（当模板读） |
| `src/channel/auth/result.ts` | — | `AuthFn` 返回值 → `SessionAuthContext` |
| `src/channel/forwarded-principal.ts` | — | 转发身份的信任条件 |
| `src/protocol/message.ts` | 2080 | 事件类型的判别联合、版本演进 |
| `src/execution/ndjson-stream.ts` | 83 | NDJSON 编解码 |
| `src/execution/session-event-stream.ts` | 136 | 订阅与重放 |

**关键片段**：
- `routeAuth` 的遍历循环：三种返回值（接受 / 跳过 / 抛错）分别怎么处理。
- `http-basic.ts` 全文 98 行——最短的完整 `AuthFn`，适合当写自定义鉴权的模板。
- `message.ts` 里事件类型的定义方式：为什么加字段是兼容的、改字段名是破坏性的。

**注意**：`none()` 放在数组中间会让后面的鉴权永远不执行。

**验证实验**：配一个只有 `none()` 的通道，`curl` 不带任何 header，看是否 200；再换成 `httpBasic()` 看 401 响应头。

---

### 29 world：检查点存在哪、怎么恢复

**回答**：`.eve/.workflow-data` 里有什么？local world 和 Postgres world 的接口差别？stranded session 怎么被检测出来？

| 源码 | 行数 | 讲什么 |
| --- | --- | --- |
| `src/execution/install-workflow-world.ts` | 19 | 安装 world 的三步（校验 → 包装 → start） |
| `src/internal/workflow/validate-world.ts` | — | world 必须实现什么 |
| `src/internal/workflow/world-compatibility.ts` | — | 版本兼容检查 |
| `src/internal/workflow/development-world-server.ts` | — | dev 下的 world 服务端 |
| `src/internal/workflow/development-world-protocol.ts` | — | 父子进程通信协议 |
| `src/execution/session-inbox/stranded-replay-guard.ts` | — | stranded 检测与重放保护 |
| `src/internal/workflow/local-world-data-directory.ts` | 7 | 数据目录常量 |

**关键片段**：
- `installWorkflowWorld` 全文 19 行——最短的一章，但解释了「为什么 eve 要自己包一层 world」。
- `guardStrandedSessionReplay` 的判定条件。

**注意**：dev 下 world 数据不在 `.eve/.workflow-data`（那里只有 `version.txt`），实际跑在 development world server 里；生产用 Postgres world 才是真的落库。

**验证实验**：`eve dev` 中途 `kill`，重启后看会话能否续上；再删 `.eve/dev-runtime/` 对比。

## 四、新旧章节的分工

避免重复，明确边界：

| 现有章节 | 讲行为 | 新章节 | 讲实现 |
| --- | --- | --- | --- |
| 03 agent 目录解剖 | 什么放哪 | 21 discover | 扫描规则与 manifest 结构 |
| 10 持久化执行 | 检查点语义、world 选型 | 24 workflow、29 world | step 包装、序列化边界、world 安装 |
| 04 工具 | `defineTool` 字段 | 25 harness | 工具循环、错误回灌、并行调度 |
| 14 通道 | `eveChannel` 配置字段 | 26、27 | 路由 handler、adapter 生命周期 |
| 13 连接 | OAuth 流程 | 28 | 鉴权链遍历、forwarded principal |
| 09 会话协议 | 事件表、HTTP 路由 | 28 | 事件类型的源码定义 |

处理方式：现有章节末尾加一行「实现见第 N 章」，新章节开头加一行「用法见第 N 章」。不改动现有正文。

## 五、写作规范

1. **引用格式**：`packages/eve/src/eve-channel/index.ts · eveChannel()`。行号只在首次出现时给，格式 `:L120`，并在章首标注 `@eve@0.81.1`。行号会漂移，函数名不会。
2. **代码片段**：单段 ≤ 40 行，保留英文原注释，中文解读放在片段外。每段前一句话说「这段在解决什么」，后一句话说「为什么不能换个写法」。
3. **每章固定结构**：一句话结论 → 要回答的问题 → 源码地图表 → 主流程走读 → 关键数据结构 → 注意事项 → 验证实验 → 交叉引用。
4. **不贴测试代码**。测试文件只在「这个行为有回归测试」时提一句文件名。
5. **不确定的地方标 `TODO(verify)`**，不用推测填充。

## 六、交付节奏（已完成）

| 批次 | 章节 | 目的 |
| --- | --- | --- |
| 第一批（试点） | 19、20、26 | 覆盖你点名的两端：CLI + eve channel。写完先看深度够不够，再决定后面怎么写 |
| 第二批 | 21、22、23 | 编译与装配链路 |
| 第三批 | 24、25 | 执行与 harness |
| 第四批 | 27、28、29 | 通道抽象、鉴权协议、world |

预估篇幅：19 约 300 行，其余每章 500-700 行，合计约 5300 行，现有 10383 行 → 约 15700 行。

## 七、已定（2026-10-22）

1. **规模**：10 章完整链路（19-29）。
2. **顺序**：按链路顺序推进，19 → 20 → 21 → …
3. **代码密度**：每章 8-12 段真实源码，单段 ≤ 40 行。
