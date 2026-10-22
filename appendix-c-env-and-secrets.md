# 附录 C 环境变量与密钥

> `process.env` 只在**应用运行时**存在且可信。它会在三个地方骗你：编译期就被读掉、沙箱里根本没有、浏览器里根本不存在。

## C.1 四个位置

| 位置 | 有 `process.env` 吗 | 谁在读 | 里面该放什么 |
| --- | --- | --- | --- |
| 编译期（`eve build` 的进程） | 有，但那是**构建机**的环境 | 静态定义模块的求值 | 被烘进编译产物的值 |
| 应用运行时（Node / Vercel Function） | 有 | 工具、连接、钩子、通道、动态解析器 | API key、数据库串、上游地址 |
| 沙箱 | **没有** | 模型跑的命令 | 什么都不该放 |
| 浏览器 | **没有** | React 组件 | 只有 `NEXT_PUBLIC_*` |

前两行是同一个变量名在两个进程里的两种答案，第三行干脆没有，第四行连 `process` 对象都不存在。教程里反复出现的「密钥只在应用运行时」，说的就是第一行和第三行不能混。

## C.2 `.env` 文件什么时候被读

eve 自己实现了 env 文件加载，不走 dotenv。读取顺序是 `.env` → `.env.development` → `.env.local` → `.env.development.local`，**后者覆盖前者**，所以优先级最高的是 `.env.development.local`，最低的是 `.env`。

解析用 Node 的 `util.parseEnv`，所以：**不做 `${VAR}` 展开**，带引号的值会去引号，`#` 开头是注释。要拼接就自己拼。

哪些命令会读：

| 命令 | 读 `.env*` | 说明 |
| --- | --- | --- |
| `eve dev` | 是 | 每次重新编译时重读 |
| `eve start` | 是 | 起生产服务前读一次 |
| `eve eval` | 是 | 跑评测前读 |
| `eve invoke` | 是 | 一次性调用前读 |
| `eve build` | **否** | 只用 shell 里已经 export 的变量 |
| `eve info` | **否** | 同上 |
| `eve deploy` | 不适用 | 在 Vercel 构建，用项目环境变量 |

`eve build` 不读 `.env*` 是**设计如此**，也是最容易被忽略的一条：本地 `eve dev` 一切正常，`eve build` 却报某个变量缺失，就是因为构建进程没拿到它。

还有一个反直觉的地方：**`eve start` 用的是同一个加载器，不分环境**。它会读 `.env.development.local`，也就是说你为开发写在那里的值，自托管的生产进程照样会读到并覆盖 `.env.local`。自托管时要么别用 `.env.development*`，要么把变量交给容器/进程管理器注入。

改完 `.env.local` 后最稳的做法是重启 `eve dev`。eve 只在下一次重新编译时重读这些文件，光改文件不一定触发重编译。

## C.3 编译期求值：最容易出错的地方

eve 在编译时就会**执行**你写的定义模块，用来校验和归一化。所以模块顶层的代码会在构建机上跑一遍。官方把模块分成三类：

| 生命周期 | 哪些定义 |
| --- | --- |
| 只编译 | 静态 instructions、静态技能、prompt 形式的 TS 定时任务、静态的 Gateway / 默认 agent 配置、provider 托管搜索、被完全覆盖的通道 |
| 编译 + 运行时 | 动态 instructions / 技能 / 工具 / 模型 / 子代理、可执行工具、生效中的通道、连接、钩子、memory、handler 形式的定时任务、直连 provider 模型、独立沙箱、远程子代理 |
| 只在运行时 | instrumentation、扩展挂载的初始化 |

「只编译」意味着**运行时不 import 它**。静态 `instructions.ts` 求值出来的内容会写进编译清单，部署后改环境变量不会改变它。

三个具体反例：

```ts
// 反例 1：静态指令在编译期读 env
// agent/instructions.ts（没有 defineDynamic 就是静态的）
import { defineInstructions } from "eve/instructions";

export default defineInstructions({
  content: `当前环境是 ${process.env.DEPLOY_ENV}。`,   // 编译期求值，随产物固定
});
```

```ts
// 反例 2：Gateway 模型字符串在编译期读 env
import { defineAgent } from "eve";

export default defineAgent({
  model: process.env.PRIMARY_MODEL!,   // 静态 Gateway 配置，编译期求值
});
```

```ts
// 反例 3：模块顶层做校验
const key = z.string().min(1).parse(process.env.OPS_TOKEN);  // 构建机上没这个变量 → eve build 直接失败
```

正确做法只有两条：

1. **把读取推迟到函数里**——工具 `execute`、连接 `getToken`、钩子 handler、`defineDynamic` 的解析器、通道的 `auth` 函数。这些都在运行时执行。
2. 如果确实要编译期求值（比如静态指令里写死一段文案），就接受「值属于构建产物」，并且把变量配到**构建环境**，不只是运行环境。

## C.4 集中校验

散落各处的 `process.env.X!` 有两个问题：类型是 `string | undefined`，以及拼错名字不会报错。用一个模块收口：

```ts
// agent/lib/env.ts
import { z } from "zod";

const schema = z.object({
  OPS_API: z.string().url(),
  LOG_TOKEN: z.string().min(1),
  LOG_API: z.string().url(),
  REDIS_URL: z.string().url().optional(),
});

let cached: z.infer<typeof schema> | undefined;

/** 在工具、连接、钩子的函数体里调用，不要在模块顶层调用。 */
export function env() {
  cached ??= schema.parse(process.env);
  return cached;
}
```

- 用函数而不是导出常量：常量在模块顶层求值，会被编译期执行；函数只在真正调用时读。
- 缓存是安全的：一个进程生命周期内 env 不会变。
- 报错信息里会带上变量名，比 `undefined is not a string` 好查得多。

需要「缺失就关闭某个能力」而不是启动失败时，在函数里返回 `null` 并记一条日志，别回退到默认值——那正是 ch14 里 `fail closed` 的做法。

## C.5 沙箱里没有 `process.env`

这是安全模型的核心，不是配置项。

- 内置的 `bash`、`read_file`、`write_file` 都跑在**应用运行时**，只是把操作代理进沙箱。所以它们能读 env，沙箱里的命令不能。
- eve 唯一主动注入的是 `AI_AGENT=eve`，让 `vercel` 这类 CLI 跳过交互提示。provider 自己还会设 `HOME`、`PATH` 这类基础变量，那是运行环境，不是你的配置。
- `environment({ env })` 把变量**烘进模板/镜像**，这个环境的所有会话共享。放密钥等于给每个会话发一份。
- `run({ env })` / `spawn({ env })` 只对这一次调用生效，但变量仍然出现在沙箱进程里——Docker、microsandbox、Vercel Sandbox 上模型都能从进程环境读到它。**单次注入不等于安全**。
- 沙箱里需要访问带鉴权的服务时，用凭据代理：密钥留在应用侧，由平台在防火墙上注入请求头，沙箱只看到响应。

判断标准很简单：**只要一个值写进沙箱，就当成模型能看到。**

## C.6 连接与通道的密钥

| 场景 | 密钥在哪读 | 会不会进编译产物 |
| --- | --- | --- |
| 连接的 `auth.getToken()` | 运行时，每次请求前 | 不会 |
| 连接的 `headers` 回调 | 运行时 | 不会 |
| 通道的 `auth` 函数 | 运行时，boot 时从源码重新取 | 不会（官方明确：route-auth 密钥不进编译产物） |
| `agent.ts` 里的字面量 | 编译期 | **会** |
| 技能正文、工具 `description` | 编译期 | **会**，而且会发给模型供应商 |

连接 token 还有两个性质值得记住：**每个 step 缓存**，以及**从不序列化进持久状态**。所以它不会出现在检查点、事件流或 `eve info` 里。

反面例子是把 token 塞进 `toolCall.providedArguments` 的字面量——那个值虽然对模型隐藏，但会写进编译后的连接定义。

## C.7 部署时的三套环境

**Vercel。** Preview 和 Production 是两套独立的变量。`.env.local` 不会随部署上传，`eve link` 只是把 AI Gateway 凭据拉到本地 `.env.local` 给你开发用。上线前逐个环境配。

**自托管。** `eve start` 会读 `.env*`，所以「文件放项目根 + 进程管理器启动」是可行的；也可以完全靠容器/系统注入，文件不存在就跳过。两条都行，别混着用——出问题时不知道值是哪来的。

**构建机与运行机分离时。** 编译期求值的那些定义（C.3）在构建机上就要拿到变量。CI 里常见的失败是「运行环境配好了，构建步骤没配」。

## C.8 会泄漏密钥的六个地方

| 位置 | 怎么漏 | 怎么防 |
| --- | --- | --- |
| 日志与 trace | `console.log(process.env)`、把整个 config 打进日志 | 只打字段名和长度；生产关掉 `EVE_TRACES_CONTENT` |
| 工具返回值 | 把上游错误体原样返回，里面带 token | 错误信息只保留状态码和业务原因 |
| 错误信息 | 连接串带密码，抛错时整条打出来 | 在 `lib/env.ts` 里就做脱敏 |
| `clientContext` | 它是客户端发来的、会进模型上下文 | 永远不要在里面放密钥；它也不是可信输入 |
| 技能 / 指令正文 | 正文进模型上下文 | 密钥不进 markdown，只进 env |
| 沙箱 env | 见 C.5 | 不放 |

## C.9 排查表

| 症状 | 原因 |
| --- | --- |
| 本地 `eve dev` 正常，`eve build` 报变量缺失 | `eve build` 不读 `.env*`，变量要 export 到构建进程 |
| 改了 `.env.local` 没生效 | 没重启 dev server，或该模块是「只编译」的，值已经写进清单 |
| 部署后模型调用 401 | 目标环境没配模型 key，或配到了另一个环境（Preview vs Production） |
| 沙箱里 `echo $OPS_TOKEN` 是空的 | 设计如此，沙箱没有应用 env |
| 浏览器里 `process is not defined` | 客户端代码读了 `process.env`；只有 `NEXT_PUBLIC_*` 会进浏览器包 |
| `eve info` 显示的能力和线上不一致 | `eve info` 是本地编译结果，不读 `.env*`，也不代表线上环境 |

## C.10 框架自己读的变量

这些是 eve 或它依赖的包会读的，别用同样的名字存自己的东西，尤其别占用 `EVE_` 前缀。

| 变量 | 谁读 | 作用 |
| --- | --- | --- |
| `AI_AGENT` | eve | 沙箱命令里自动设为 `eve`；可覆盖 |
| `EVE_TELEMETRY_DISABLED` | CLI | 关掉 CLI 遥测（单次命令） |
| `EVE_TRACES_CONTENT` | 本地 trace | `off` 时不记录 prompt、工具参数与结果 |
| `EVE_TRACES` | 本地 trace | `off` 关掉本地 trace 存储 |
| `EVE_TRACES_MAX_AGE_MS` / `EVE_TRACES_MAX_TOTAL_BYTES` / `EVE_TRACES_RETAIN_COUNT` | 本地 trace | 保留上限，默认 7 天 / 512 MiB / 20 条，各接受 `off` |
| `EVE_PUBLIC_ROUTE_PREFIX` | eve | 把 agent 路由挂到自定义前缀下 |
| `EVE_NEXT_PRODUCTION_ORIGIN` | `eve/next` | 非 Vercel 生产环境下 eve 服务的地址 |
| `EVE_MEMORY_BLOB_STORE_ID` / `EVE_MEMORY_BLOB_WEBHOOK_PUBLIC_KEY` | `fileMemory` | `eve add memory/file` 配好的 Blob 存储 |
| `WORKFLOW_LOCAL_DATA_DIR` | 本地 world | workflow 数据目录，默认 `.eve/.workflow-data` |
| `WORKFLOW_LOCAL_RECOVER_ACTIVE_RUNS` | 本地 world | 启动时是否恢复活跃 run |
| `WORKFLOW_POSTGRES_URL` | `world-postgres` | 连接串；没有时退回 `DATABASE_URL` |
| `EVE_MODEL_CONNECTION` / `EVE_MODEL_TEAM` / `EVE_MODEL_TEAM_NAME` / `EVE_MODEL_KEY_SOURCE` | dev server | TUI 登录后自动注入，不用你写进 `.env` |

---

返回：[教程目录](README.md) · 附录 A：[术语与速查](appendix-a-glossary.md) · 附录 B：[事件与 CLI](appendix-b-events-and-cli.md)
