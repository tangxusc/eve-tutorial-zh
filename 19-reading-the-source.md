# 第 19 章 怎么读 eve 的源码

> 一句话结论：eve 的代码按「执行时机」分层，不按功能分层。找代码先问「这段什么时候跑」，比按目录翻快十倍。

## 一个具体的困惑

假设你遇到这么一件事：给 `agent/tools/read_logs.ts` 加了一行日志，`eve dev` 重启了，但日志没出现。你确认文件路径没错，`eve info` 里也能看到这个工具。

问题出在哪？可能是文件名被识别成了别的名字，可能是模块没被加载，也可能这行代码根本没执行到。三种原因在三个不同的层。

想搞清楚，只能读源码。

打开 `github.com/vercel/eve`，`packages/eve/src` 下有 1816 个源文件、约 28 万行，另有 1064 个测试文件。通读不现实。

这一章不教任何具体机制，只教方法：**读哪一份、怎么看懂一个 import、从哪个目录找起、怎么确认代码真的在跑**。后面十章都建立在这套方法上，方法不对，读十遍也记不住。

## 19.1 先解决「读哪一份」

源码有三个来源，用途完全不同。选错来源，会浪费几个小时。

| 来源 | 路径 | 能回答什么 | 不能回答什么 |
| --- | --- | --- | --- |
| GitHub 源码 | `packages/eve/src/**/*.ts` | 实现怎么写、为什么这么写 | 与安装版本的差异 |
| 本地类型声明 | `node_modules/eve/dist/src/**/*.d.ts` | 安装版本的确切契约、字段、默认值 | 函数体 |
| 本地编译产物 | `node_modules/eve/dist/src/**/*.js` | 导出名是否存在 | **几乎什么都回答不了** |

第三条反直觉，值得展开。

很多人以为 `dist` 是「编译后的 TS」，可以当源码读。eve 不是。发布前做了压缩：

```js
// node_modules/eve/dist/src/cli/run.js —— 全文 1 行，9895 字节
import{Command,CommanderError}from"#compiled/commander/index.js";import{agentCommand}from"#cli/agent-command.js";import{parseDevelopmentServerUrl}from"#cli/dev/url.js";...
```

变量名缩成 `e`、`t`、`n`，换行和空格全没了，`find dist -name "*.map"` 返回 0——发布包里没有 sourcemap。这份代码只能执行，不能阅读。

但同一目录下的 `.d.ts` 是完整的：

```ts
// node_modules/eve/dist/src/internal/logging.d.ts
// 只有类型和 JSDoc，没有函数体——发布包里可读性最好的一层
/**
 * Framework-internal structured logger.
 *
 * Not public API; authors should use their own logger or Vercel
 * observability primitives.
 */
export type LogLevel = "debug" | "info" | "warn" | "error";

/**
 * Builds a logger bound to a stable namespace (e.g. `"slack.route"` or
 * `"harness.tool-loop"`). The namespace appears in every rendered line
 * and is threaded through to OTel span events so a single grep can
 * correlate structured logs with traces.
 */
export function createLogger(namespace: string): Logger;
```

注释、类型、默认值都在，还和安装的版本严格一致。

所以规则是：**读实现去 GitHub，读契约读本地 `.d.ts`**。两者对照，能同时确认「代码想干什么」和「安装版本实际是什么」。

本教程基于 `eve@0.81.1`。把 GitHub 源码对齐到该版本，三条命令：

```bash
git clone --depth 1 https://github.com/vercel/eve.git
cd eve
git fetch --depth 1 origin refs/tags/eve@0.81.1:refs/tags/eve@0.81.1
git checkout eve@0.81.1
```

小版本之间函数名基本稳定，行号会漂——所以教程引用一律用「文件 + 函数名」，行号只作参考。

## 19.2 再解决「看懂一个 import」

选好来源，打开任意一个源文件。第一屏几乎全是这种导入：

```ts
// packages/eve/src/eve-channel/index.ts:1
// 五个 import 对应五个子系统：工具桩、workflow 运行时、流式响应头、路由前缀。
// 读一个模块的 import，就知道它跨了哪些层。
import { InvalidToolStubTargetError } from "#tool-stubs/validate-targets.js";
import { readStubFailure, readMatchedStubRules } from "#execution/tool-stubs/steps.js";
import { getRun } from "#internal/workflow/runtime.js";
import { proxyStreamHeaders } from "#eve-channel/stream-headers.js";
import { EVE_ROUTE_PREFIX } from "#protocol/routes.js";
```

不搞懂这几行，后面每读一个文件都要卡一次。拆开看，每一行说了三件事。

**一、`#` 开头 = 包内私有路径。**

这不是公开 API。你在自己的 `agent/` 里写 `import ... from "#internal/logging.js"` 会失败，因为 `#` 只在包内部解析。对外暴露的入口走 `package.json` 的 `exports`，也就是 `eve/channels/eve`、`eve/tools/bash` 这种。

**二、写 `.js`，文件却是 `.ts`。**

靠 `package.json` 的 `imports` 字段做映射：

```json
{
  "imports": {
    "#compiled/*": {
      "eve-source": "./.generated/compiled/*",
      "default": "./dist/src/compiled/*"
    },
    "#*.js": {
      "eve-source": "./src/*.ts",
      "default": "./dist/src/*.js"
    }
  }
}
```

`#cli/run.js` 里的 `*` 匹配 `cli/run`，在 `eve-source` 条件下解析到 `./src/cli/run.ts`，否则解析到 `./dist/src/cli/run.js`。同一个 import 说明符，在 eve 仓库里指向 TS，在发布包里指向编译产物。

`eve-source` 是自定义条件，由 `packages/eve/tsconfig.json` 声明：

```json
{
  "compilerOptions": {
    "outDir": "./dist",
    "allowImportingTsExtensions": true,
    "rewriteRelativeImportExtensions": true,
    "customConditions": ["eve-source"]
  }
}
```

`rewriteRelativeImportExtensions` 负责把源码里的 `.ts` 后缀在编译时改成 `.js`。所以你看到的源文件里两种写法都有，都是对的。

**三、前缀就是分层。**

`#cli/`、`#internal/`、`#execution/`、`#harness/` 各管一段，下一节展开。

顺带说一个例外：`#compiled/*` 不是 eve 自己的代码，是 vendored 的第三方依赖（commander、semver、opentelemetry）。看到 `#compiled/commander/index.js` 直接当 npm 包理解，不用读。

## 19.3 然后解决「从哪找」

这是本章最有用的部分。回到开头那个困惑：日志没出现，该去哪个目录找？

答案是：先问这段代码**什么时候执行**。

eve 的目录不是按功能分的，是按执行时机分的：

| 目录 | 执行时机 | 触发频率 | 改这里会影响 |
| --- | --- | --- | --- |
| `discover/` | 编译期：扫描 `agent/` 目录 | 源码或目录结构变更 | 哪些文件被认成能力 |
| `compiler/` | 编译期：生成 manifest 与 module map | 同上 | 产物结构、能力归一化 |
| `cli/` | 进程启动：命令解析、起服务 | 每条命令一次 | 命令行为、dev server 生命周期 |
| `runtime/` | 进程启动后：module map → 活对象 | 每个产物版本一次 | 模块求值时机、配置合并 |
| `execution/` | 每个 workflow step | 每个 step | 检查点、事件落库、恢复 |
| `harness/` | 每次模型调用 / 每次工具调用 | 每个 turn 内多次 | 循环、工具调度、压缩 |
| `channel/` `eve-channel/` | 每次 HTTP 请求 | 每个请求 | 路由、鉴权、流 |
| `public/` | 作者代码 import 时 | 编译期 + 运行时 | 对外 API 的形状 |
| `internal/` | 跨层：宿主、日志、workflow、打包 | 视调用方 | 基础设施 |

有了这张表，开头的困惑就有了排查顺序：

1. 文件被识别了吗？→ `discover/`
2. 编译产物里有它吗？→ `compiler/` 和 `.eve/compile/`
3. 运行期加载了吗？→ `runtime/`
4. 代码真的执行到了吗？→ 下一节的三个手段

这张表还解释了几个常见现象：

- 改 `compiler/` 里的东西，要等 dev server 重建产物。改完立刻发请求可能还是旧代码。
- 改 `harness/` 里的东西，只影响**新开的 session**。已经在跑的 session 用它启动时的产物快照（第 22 章讲这个机制）。
- 改 `public/definitions/*` 里的类型，影响的是作者代码的编译，不是运行时行为。

## 19.4 最后解决「怎么确认」

读源码最容易犯的错是「以为这段在跑，其实没走这条分支」。三个办法验证，从轻到重。

**一、`EVE_LOG_LEVEL=debug`。**

日志阈值在 `src/internal/logging.ts:257` 的 `resolveThreshold()`：

```ts
// 每次写日志都现读环境变量：改了级别不用重启，代价是每条日志多一次 getenv
function resolveThreshold(): number {
  const configured = process.env.EVE_LOG_LEVEL?.toLowerCase();

  // 只认这四个值。拼错（比如写成 EVE_LOG=debug）不报错，静默落回 info
  if (
    configured === "debug" ||
    configured === "info" ||
    configured === "warn" ||
    configured === "error"
  ) {
    return LEVEL_SEVERITY[configured];
  }

  // 未设置时的默认值：info 及以上才输出，debug 默认关
  return LEVEL_SEVERITY.info;
}
```

默认 `info`，`debug` 被吞掉。设成 `debug` 后，`write()` 走 `console.log`：

```ts
// src/internal/logging.ts:308
function write(level: LogLevel, namespace: string, message: string, fields?: LogFields): void {
  // 低于阈值直接返回：过滤发生在拼字符串之前，被关掉的日志几乎不花钱
  if (LEVEL_SEVERITY[level] < resolveThreshold()) {
    return;
  }
  // ...
  // 按级别选输出流：error 走 stderr，其余走 stdout，方便分开重定向
  const sink = level === "error" ? console.error : level === "warn" ? console.warn : console.log;

  // 固定前缀 [eve:命名空间]，所以 grep "eve:channel" 能捞出整条链路
  const line = `[eve:${namespace}] ${message}`;

  // 没有结构化字段就只打一行；有字段时作为第二个参数交给 console
  if (fields === undefined) {
    sink(line);
    return;
  }
  sink(line, renderFields(fields));
}
```

输出长这样，方括号里是命名空间，可以直接 grep（字段部分省略）：

```text
[eve:eve.channel] serving a remote agent protocol 1 caller
[eve:channel.adapter] adapter event handler threw — event swallowed
```

命名空间就是源码里 `createLogger("xxx")` 的参数。想知道某个模块有没有跑，先搜它的命名空间字符串。

**二、`.eve/logs/*.dump`。**

每次 `eve dev` 在项目里留一对文件：`*.log` 记录终端输出，`*.dump` 是诊断快照。没输出时 `.log` 是 0 字节，`.dump` 一定有内容。

```json
{
  "updatedAt": "2026-10-20T06:30:24.622Z",
  "durationMs": 58469,
  "environment": {
    "eveVersion": "0.81.1",
    "nodeVersion": "v24.14.0",
    "platform": "darwin arm64"
  },
  "session": null
}
```

`session` 为 `null` 的那次 dev 进程没跑过会话。跑过对话之后再来看这个字段，结构会不一样。

**三、`node --inspect`。**

CLI 是普通 Node 进程，可以直接断点：

```bash
node --inspect-brk node_modules/eve/bin/eve.js dev
```

然后 Chrome 打开 `chrome://inspect`，在 `src`（本地开发）或 `dist/src`（发布包）里下断。压缩过的 dist 断点会落在整行上，变量名是 `e`/`t`，只能看调用栈，不能看逻辑——这也是为什么读实现必须用 GitHub 源码。

## 19.5 一张随时能查的索引

前面讲的是方法，这一节给工具。下面这张表按「你遇到的现象」组织，遇到问题直接查，不用从头想。

| 现象 / 需求 | 起点文件 | 下一层 |
| --- | --- | --- |
| 我加的文件没被识别成工具 | `discover/grammar.ts` | `discover/slots.ts` → `discover-agent.ts` |
| `eve info` 显示的能力和预期不符 | `cli/commands/info.ts` | `services/inspect-application.ts` |
| 工具参数校验失败，想看 schema 怎么生成 | `runtime/resolve-tool.ts` | `harness/provider-tool-schemas.ts` |
| 工具被调用了两次 | `execution/node-step.ts` | `execution/session.ts`（重放语义） |
| 审批之后上下文丢了 | `harness/hitl/` | `execution/session.ts` |
| 会话恢复失败、报 stranded | `execution/session-inbox/stranded-replay-guard.ts` | `internal/workflow/` |
| 路由 404 | `protocol/routes.ts` | `eve-channel/index.ts` |
| 路由 401，想知道为什么 | `public/channels/auth.ts` | `channel/auth/result.ts` |
| 流断了，`startIndex` 没生效 | `eve-channel/request.ts` | `execution/session-event-stream.ts` |
| 事件少了一个字段 | `protocol/message.ts` | 对应 `publish-session-events.ts` |
| `turnPolicy` 行为不对 | `channel/session.ts` | `execution/session-inbox/` |
| dev 改代码不生效 | `internal/nitro/dev-runtime-artifacts.ts` | `compiler/module-map.ts` |
| 部署后行为和本地不一样 | `internal/workflow/world-compatibility.ts` | `execution/install-workflow-world.ts` |

## 19.6 三条经验

方法之外，还有三条经验。它们不是规则，是反复排查中养成的习惯。

**一、先读类型，再读实现。**

`src/public/**/types.ts` 或 `.d.ts` 是契约，实现只是契约的一种满足方式。先看契约能省掉一半阅读量：很多分支是在处理契约允许、但你没想过的输入。

**二、先读测试，再读实现。**

1064 个测试文件、28 万行，和源码行数几乎一比一。这是 eve 最诚实的行为文档。测试名字直接描述行为：

```ts
// src/channel/adapter.test.ts
// 测试名就是行为契约：可选字段、返回值必须是 Buffer。
describe("ChannelAdapter (fetchFile field)", () => {
  it("treats the fetchFile field as optional", () => {
    const adapter: ChannelAdapter = { kind: "noop" };
    expect(adapter.fetchFile).toBeUndefined();
  });

  // 自定义通道实现 fetchFile 时，入参只有 url，返回值必须是 Buffer 而不是流
  it("accepts a fetchFile function that returns a Buffer", async () => {
    const adapter: ChannelAdapter = {
      kind: "custom",
      async fetchFile(url) {
        return Buffer.from(`url=${url}`);
      },
    };
    // ...
  });
});
```

想知道「`fetchFile` 返回 `null` 会怎样」，搜 `fetchFile` 的测试比读 300 行实现快。文档会过期，测试不会——测试挂了 CI 就红。

**三、不信注释，信调用点。**

源码注释描述的是作者的意图，不是当前行为。判断一段代码有没有用，搜它的调用点：

```bash
# 这个函数到底谁在调
rg "callAdapterEventHandler" packages/eve/src --type ts -g '!*.test.ts'
```

零调用点 + 只有测试在用 = 兼容层或已废弃。这种情况在 eve 里不少，因为它在 beta 期要保持向后兼容。

第 21 章会给你一个现成的例子：`named-source-directory.ts` 的注释说 `tools/` 是 flat 的，调用点传的是 `recursive: true`。注释过期了，代码没改。

## 19.7 练习

1. 把 `EVE_LOG_LEVEL` 设成 `debug`，跑 `eve dev`，找出输出里出现过的所有命名空间。对照 `src/**/` 里 `createLogger(` 的参数，看哪些模块这次没跑。
2. 打开 `node_modules/eve/dist/src/internal/logging.d.ts`，找到 `Logger` 接口。再去 GitHub 找 `src/internal/logging.ts` 的同名接口，对比两者是否一致。
3. 打开 `node_modules/eve/package.json` 的 `exports`，找到 `./channels/eve` 指向哪个文件。再顺着那个文件读下去，看 `eveChannel` 经过几层才到真正的实现。这条路径就是「公开 API 怎么包一层」的样板。
4. 打开 `.eve/logs/` 里最新的一份 `.dump`，说出这次 dev 进程跑了多久、用的哪个 eve 版本。

## 下一章

方法有了，从哪开始走？

第 20 章从你在终端敲下的第一行命令 `eve dev` 开始。你会看到它在进入命令行解析之前先检查 Node 版本、按需重编译自己，然后 fork 一个子进程跑服务器——以及为什么必须 fork。

---

返回：[教程目录](README.md) · 下一章：[20 CLI 启动链路](20-cli-internals.md)
