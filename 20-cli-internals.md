# 第 20 章 CLI 启动链路

> 一句话结论：`eve` 不是薄封装。它在进 commander 之前先检查 Node 版本、按需重编译自己；`eve dev` 会 fork 一个 detached 子进程跑服务器，然后用 IPC 而不是信号管理这个子进程的生死。

## 三个日常困惑

用 eve 久了，会遇到三个说不清的现象：

1. 第一次在 clone 下来的 eve 仓库里跑命令，卡十几秒才出结果。第二次就快了。
2. 在项目目录里敲 `eve`，它起 dev；在空目录里敲 `eve`，它跑 init。没人告诉你它怎么判断。
3. `eve dev` 按一次 Ctrl-C 有时退不掉，要按两次。`ps` 里还能看到两个 eve 进程。

这三个现象对应同一条链路上的三段代码。这一章从终端敲下命令开始，一路跟到 TUI 出现，把这三段都讲清楚。

## 20.1 命令进来之前，先过两道关

`package.json` 的 `bin` 字段指向 `bin/eve.js`。这个文件不包含任何业务逻辑，只做四件事。

```js
// packages/eve/bin/eve.js · runEveCli()
export async function runEveCli(argv = process.argv.slice(2), overrides = {}, dependencies = {}) {
  const options = createBootstrapOptions(overrides);
  await assertSupportedNodeVersion(
    dependencies.nodeVersion,
    dependencies.nodeEngineRequirement,
    options,
    dependencies,
  );

  const cliEntrypointPath = await ensureBuiltCli(options, dependencies);
  const importModule = dependencies.importModule ?? ((specifier) => import(specifier));
  const cliModule = await importModule(pathToFileURL(cliEntrypointPath).href);

  if (typeof cliModule.runCli !== "function") {
    throw new Error(`The eve CLI module at ${cliEntrypointPath} does not export runCli().`);
  }

  await cliModule.runCli(argv);
}
```

**第一关：Node 版本。**

`assertSupportedNodeVersion` 读 `package.json` 的 `engines.node`，用 vendored 的 semver 比对。不满足就直接抛错，不进 commander。

为什么放在最前面？因为代码里有硬性的版本依赖。`engines.node` 要求 `>=24`，实际用到的 API 也在这个量级——`shutdown.ts` 里的 `Promise.withResolvers()` 是 Node 22 才有的。版本不够时，你希望看到的是 `eve requires Node.js >=24.`，而不是运行到一半报 `Promise.withResolvers is not a function`。

**第二关：按需重编译自己。**

这一关解释了本章开头的第一个困惑。

```js
// packages/eve/bin/eve.js · ensureBuiltCli()
if (await exists(options.cliEntrypointPath)) {
  if (!packageCanBuildCli) {
    return options.cliEntrypointPath;
  }

  const [latestBuildInputMtimeMs, cliEntrypointMtimeMs] = await Promise.all([
    getBuildInputMtimeMs({ packageRoot: options.packageRoot }),
    getEntrypointMtimeMs(options.cliEntrypointPath),
  ]);

  if (cliEntrypointMtimeMs !== undefined && cliEntrypointMtimeMs >= latestBuildInputMtimeMs) {
    return options.cliEntrypointPath;
  }
}
// ... 否则依次跑 vendor-compiled、tsc、四个 post-build 脚本
```

逻辑是：`dist/src/cli/run.js` 存在，并且比 `src`、`bin`、`scripts` 里最新的文件还新，就直接用；否则重新编译整个包。

`packageCanBuildCli` 会检查 `bin`、`tsconfig.json`、`src`、`scripts/vendor-compiled.mjs` 是否存在。**发布包里没有 `src`**，所以这段分支永远不会在 npm 安装的项目里触发。只有从源码跑（workspace 开发、`git clone` 后 `npm link`）才会。

这就是「改了 eve 源码，第一次跑命令会卡十几秒」的原因——它在跑 tsc。第二次跑，`dist` 已经比 `src` 新，直接跳过。

**后面两件事：导入和退出。**

用 `pathToFileURL` 动态导入 `runCli`，因为路径是运行时算出来的。

命令跑完后，先 flush 再退出：

```js
// packages/eve/bin/eve.js 末尾
} finally {
  // Pipes and sockets can accept writes asynchronously, and `process.exit`
  // drops anything still queued, truncating large output such as
  // `eve info --json` read by a parent Node process.
  await Promise.all([flushWritable(process.stdout), flushWritable(process.stderr)]);
  // The CLI bootstraps build/dev toolchains that can leave native service
  // handles alive after the command has completed. Once the top-level
  // command resolves, terminate the bin process explicitly so commands like
  // `eve dev` + `/exit` do not hang on leaked implementation details.
  process.exit(process.exitCode ?? 0);
}
```

两段注释就是全部理由。前者解释为什么不能直接 `process.exit`（会截断管道里的 JSON），后者解释为什么必须 `process.exit`（构建工具链会留下 native handle，不主动退进程就挂着）。

顺序不能反：先 flush 再 exit。

**注意**：`flushWritable` 只对 stdout/stderr 的 `write` 回调做等待，不覆盖 `fs.writeSync` 之类的同步写。如果你在命令里用 `process.stdout.fd` 直接写，这部分不受保护。

## 20.2 命令怎么被分发

过了两道关，进入 `src/cli/run.ts`。这个文件建 commander 实例，按顺序注册全部命令。

所有命令的 action 都是同一个形状：

```ts
// packages/eve/src/cli/run.ts · createCliProgram()
agentCommand(program.command("start"), applicationContext)
  .description("Start a built eve application.")
  .option("--host <host>", "Host interface to bind")
  .option("--port <port>", "Port to listen on (defaults to $PORT, then 3000)", parsePortOption)
  .action(async (options: ProductionCliOptions) => {
    const { loadDevelopmentEnvironmentFiles } = await import("#cli/dev/environment.js");
    await loadDevelopmentEnvironmentFiles(applicationContext.root);
    // ... 起生产服务器
  });
```

`await import()` 写在 action 里面，不在模块顶层。这样 `eve --version` 只加载 commander 和注册代码，不会把 Vercel SDK、TUI 渲染器、eval runner 一起拉起来。启动时间差在这里，不在内存。

### `eve` 不带参数时跑什么

这是本章开头的第二个困惑。

```ts
// packages/eve/src/cli/run.ts:385 · runCli()
let input = argv;
if (input.length === 0) {
  const findApplicationRoot = runtime.findApplicationRoot ?? findCliApplicationRoot;
  const appRoot = await findApplicationRoot(applicationContext.root);
  if (appRoot === undefined) {
    const projectContext = await findEveProjectContext(applicationContext.root);
    input = projectContext?.kind === "workspace" ? ["dev"] : ["init"];
  } else {
    applicationContext.root = appRoot;
    input = ["dev"];
  }
}
```

翻译成行为：

| 当前目录 | `eve` 等价于 | 为什么 |
| --- | --- | --- |
| 有 `agent/` 或 `agents/` 的 eve 项目内 | `eve dev` | 已经初始化过 |
| 在 `agents/` 工作区根目录 | `eve dev` | 工作区也是项目 |
| 什么都没有的空目录 | `eve init` | 第一次用 |

`findCliApplicationRoot` 会向上找最近的 eve 应用根，所以你在 `agent/tools/` 子目录里敲 `eve` 也能起 dev。

### 工作区里选哪个 agent

`dev`、`build`、`start`、`info`、`eval`、`logs`、`traces` 都经过 `agentCommand` 包一层：

```ts
// packages/eve/src/cli/agent-command.ts · agentCommand()
export function agentCommand(
  command: Command,
  applicationContext: CliApplicationContext,
  requirement: AgentCommandRequirement = () => true,
): Command {
  command.option("--agent <name>", "Select an agent from an agents/ workspace");
  return command.hook("preAction", async (_command, actionCommand) => {
    const requestedName = actionCommand.opts<{ agent?: string }>().agent;
    if (!requirement(actionCommand)) { /* ... */ return; }

    const initialSelection = await applicationContext.resolveAgent();
    if (initialSelection.kind === "workspace") {
      applicationContext.root = await selectWorkspaceAgent(initialSelection.workspace, requestedName);
      await applicationContext.resolve();
      return;
    }
    // ... 非 workspace 时校验 --agent 只能指向工作区成员
  });
}
```

它做两件事：给命令加 `--agent` 选项；在 action 执行前把 `applicationContext.root` 改写成具体 agent 的目录。

`selectWorkspaceAgent` 的选择顺序值得记住：

1. 传了 `--agent <name>` → 按名字找，找不到直接报错并列出可选项。
2. 工作区只有一个成员 → 用它。
3. 多个成员且终端是 TTY → 弹交互选择。
4. 多个成员且不是 TTY（CI、管道）→ 报错，要求显式传 `--agent`。

第 4 条是给 CI 用的。在流水线里跑 `eve build` 而不指定 agent，不会静默选第一个，会失败。

### 参数写错时会发生什么

commander 默认在参数错误时自己 `process.exit(1)`。eve 关掉了这个行为：

```ts
// packages/eve/src/cli/run.ts · createCliProgram()
program
  .name("eve")
  .version(packageVersion)
  .showHelpAfterError()
  .exitOverride()
  .configureOutput({
    writeErr: (message) => {
      logger.error(message.trimEnd());
    },
    writeOut: (message) => {
      logger.log(message.trimEnd());
    },
  });
```

`.exitOverride()` 让 commander 抛 `CommanderError` 而不是直接退出。这样 `runCli` 的 try/catch 能接住，做三件事：记遥测（`usage_error` 还是 `error`）、决定是否补打帮助、最后统一抛出。

`.showHelpAfterError()` 是「参数写错时顺便打印用法」。`.configureOutput` 把 commander 的输出接到 eve 自己的 logger 上，保证 `--json` 模式下 stdout 只有 JSON，不会被用法说明污染。

catch 里有个专门的补丁：

```ts
if (error instanceof CommanderError) {
  const detectCodingAgentLaunch = runtime.isCodingAgentLaunch ?? isCodingAgentLaunch;
  const agentLaunched = await detectCodingAgentLaunch();
  if (input[0] === "init" && agentLaunched) {
    const { initAgentInstructions } = await import("#cli/commands/agent-instructions.js");
    logger.log(initAgentInstructions());
  }
  throw new Error(error.message);
}
```

`eve init` 因为参数错误被 commander 拒掉时，如果调用方是 AI agent，会补一段「你应该怎么初始化这个项目」的说明。因为 agent 拿到的错误信息如果只有 `unknown option`，它没法自己纠偏。

## 20.3 `eve dev`：为什么是两个进程

这是本章开头第三个困惑的答案，也是整章最核心的一段。

`eve dev` 不是「在一个进程里起个 HTTP server」，它 fork 一个子进程。原因写在 `local-server-process.ts` 的文件头：

```ts
/**
 * The `eve dev` server child: forked `detached` (its own process group) and
 * bound to the CLI's lifetime through the IPC channel, never through the OS.
 *
 * Why the child is not bound to the parent at the OS level (shared process
 * group, inherited terminal signals):
 *
 * - The TUI puts stdin in raw mode, which disables ISIG: Ctrl+C never becomes
 *   a terminal-generated SIGINT, so same-group signal delivery does nothing in
 *   the mode where Ctrl+C matters most.
 * - In headless mode, same-group delivery would signal parent and child
 *   simultaneously, racing the parent's graceful IPC shutdown against the
 *   child's own signal handler.
 * - macOS has no parent-death signal. IPC `disconnect` is the portable
 *   equivalent: the child self-stops whenever this process exits, for any
 *   reason including SIGKILL (see local-server-child.ts).
 */
```

三条理由，每条都对应一个真实约束：

1. **TUI 把 stdin 设成 raw mode，禁用了 ISIG。** Ctrl+C 不再产生 SIGINT，靠进程组投递信号在 TUI 里完全无效。这是最反直觉的一条：你以为 Ctrl+C 会杀进程组，实际上在 raw mode 下它只是一个字节 `\x03`。
2. **headless 模式下同组投递会同时打到父子进程。** 父进程正在走优雅关闭，子进程自己的 handler 也在跑，两边竞争。
3. **macOS 没有 parent-death signal。** Linux 有 `PR_SET_PDEATHSIG`，macOS 没有。可移植的替代是 IPC channel 断开。

`detached: true` 让子进程成为进程组组长。这样 `close()` 可以用 `process.kill(-pid, ...)` 一次性收掉整棵进程树——包括子进程再拉起的沙箱进程。

### 启动时的握手

`start()` 用 `fork` 拉起子进程，stdio 是 `["ignore", "pipe", "pipe", "ipc"]`，然后等一条 IPC 消息：

```ts
// packages/eve/src/cli/dev/local-server-process.ts · start()
spawned.on("message", (message: ChildMessage) => {
  if (message.type === "progress") options.onBootProgress?.(message.event);
  if (message.type === "started" && !settled) {
    settled = true;
    resolve(message.handle);
    if (message.handle.kind === "existing") {
      expectedExit = true;
      release();
    }
  }
  if (message.type === "failed" && !settled) {
    settled = true;
    reject(new Error(message.message));
  }
});
```

三种消息：`progress`（启动进度，喂给 TUI 的进度行）、`started`（带 handle，含 url）、`failed`。

`handle.kind === "existing"` 时子进程会主动断开 IPC 退出——因为已经有另一个 server 在跑，这个子进程只是去探测了一下。这对应 TUI 模式的 `existing: "attach-if-unconfigured"`。

实际效果和直觉相反，值得记清楚：

| 第二个终端执行的命令 | 结果 |
| --- | --- |
| `eve dev` | 附着到已有实例，TUI 连过去，不再起 server |
| `eve dev --port 3001` | **报错**：已有实例在跑，并打印连接命令 |
| `eve dev --no-ui` | 报错（`existing: "reject"`） |

判定在 `start-development-server.ts`：

```ts
const hasExplicitServerConfiguration =
  options.resume === true ||
  options.developmentExtensions !== undefined ||
  options.host !== undefined ||
  options.port !== undefined ||
  environmentPort !== undefined;

if (options.existing === "attach-if-unconfigured" && !hasExplicitServerConfiguration) {
  return { handle: { kind: "existing", appRoot: project.appRoot, url: existingServerUrl }, close: undefined };
}
throw await createDevelopmentServerAlreadyRunningError(project.appRoot, existingServerUrl);
```

只要传了 `--port`、`--host`、`--resume` 或设了 `PORT`，就被当成「你想控制 server 配置」，不再自动附着。想同时跑两个实例，得让它们各自有独立的项目目录。

### 关闭时的三级升级

```ts
// packages/eve/src/cli/dev/local-server-process.ts:47
const IPC_SHUTDOWN_GRACE_MS = 550;
const SIGTERM_GRACE_MS = 150;
const SIGKILL_REAP_MS = 100;

export const DEV_SERVER_CLOSE_BUDGET_MS =
  IPC_SHUTDOWN_GRACE_MS + SIGTERM_GRACE_MS + SIGKILL_REAP_MS;
```

`close()` 里的三级升级：

```ts
// packages/eve/src/cli/dev/local-server-process.ts · close()
if (active.connected) active.send({ type: "shutdown" });
if (
  await terminatesWithin(active, terminated.promise, () => childExited, IPC_SHUTDOWN_GRACE_MS)
)
  return;
signalProcessGroup(active, "SIGTERM");
if (await terminatesWithin(active, terminated.promise, () => childExited, SIGTERM_GRACE_MS))
  return;
signalProcessGroup(active, "SIGKILL");
await terminatesWithin(active, terminated.promise, () => childExited, SIGKILL_REAP_MS);
```

先礼后兵：IPC `shutdown` 消息 → 整个进程组 SIGTERM → 整个进程组 SIGKILL。每级都有超时，总和 800ms。

这个 800 不是随便定的。`shutdown.ts` 里的 `FORCED_EXIT_BACKSTOP_MS = 900`，注释写明：

> Must stay below the forced-exit backstop the dev command arms on its lifecycle, so a graceful close always beats the backstop and `eve dev` exits 0 after Ctrl+C.

优雅关闭必须跑在兜底之前，否则 Ctrl+C 后 `eve dev` 会以非 0 退出。这个约束有测试守着（`local-server-process.test.ts`）。

### 子进程这一侧

```ts
// packages/eve/src/cli/dev/local-server-child.ts
const stop = () => {
  if (stopping) return;
  stopping = true;
  console.log = console.warn = console.error = () => {};
  process.stdout.on("error", () => {});
  process.stderr.on("error", () => {});
  const cleanup = server.close();
  if (process.connected) process.disconnect?.();
  void cleanup.then(
    async () => {
      await intentReady.catch(() => undefined);
      await removeCleanupIntent?.().catch(() => undefined);
      process.exit(0);
    },
    () => process.exit(1),
  );
};

process.on("message", (message) => { if (message.type === "shutdown") stop(); });
// IPC disconnect is the parent-death signal: it fires however the parent
// exits (including SIGKILL), which no portable OS mechanism covers.
process.once("disconnect", stop);
process.once("SIGINT", stop);
process.once("SIGTERM", stop);
```

四个细节：

1. **先静音 console。** 关闭过程中 server 还在写日志，父进程可能已经关了管道，继续写会抛 EPIPE。先换成空函数，再给 stdout/stderr 挂 error 监听吞掉。
2. **先 `server.close()` 再 `disconnect`。** 顺序反了父进程会以为子进程已死，但其实资源还没释放。
3. **`disconnect` 是父进程死亡的唯一可靠信号。** 父进程被 `kill -9` 时没有信号发给子进程，只有 IPC channel 关闭。这是 macOS 上唯一可移植的方案。
4. **`recordCleanupIntent`。** 启动成功后写一个清理意图文件，正常退出时删掉。如果进程被 SIGKILL（连 `disconnect` 都来不及处理），文件留着，下次启动时 `reconcileCleanupIntents` 会发现并清理沙箱资源。

第 4 条解释了「为什么我的沙箱容器在异常退出后还留着」——它在等下一次 `eve dev` 来收尸。

## 20.4 为什么 Ctrl-C 要按两次

```ts
// packages/eve/src/cli/shutdown.ts · installShutdownSignal()
const handleSignal = (signal: NodeJS.Signals) => {
  if (terminalLeaseDepth > 0) return;
  signalCount += 1;
  if (signalCount > 1) process.exit(signal === "SIGINT" ? 130 : 143);
  stop(signal);
};
```

第一次收到 SIGINT：`stop()` 里 abort controller、resolve `stopped`、挂 900ms 兜底定时器。第二次：立刻 `process.exit(130)`，不等任何清理。

设计意图是「第一次给优雅关闭机会，第二次你说了算」。130 和 143 是 shell 约定的退出码（128 + 信号编号），`2` 是 SIGINT，`15` 是 SIGTERM。

`withTerminalLease` 是配套机制：

```ts
async withTerminalLease<T>(task: () => Promise<T>): Promise<T> {
  terminalLeaseDepth += 1;
  try {
    return await task();
  } finally {
    terminalLeaseDepth -= 1;
  }
}
```

在 lease 期间，SIGINT 被忽略（`terminalLeaseDepth > 0` 直接 return）。用途是把终端临时交给一个继承 stdio 的子进程——比如 `eve init` 装依赖时，包管理器的交互提示需要独占 Ctrl-C。这时按 Ctrl-C 应该中断 npm，而不是中断 eve。

## 20.5 `eve init`：生成什么，为什么自动进 dev

`eve init` 的结尾有一段容易误读的逻辑：

```ts
// packages/eve/src/cli/commands/init.ts · runInitCommand()
const agentLaunched = await dependencies.isCodingAgentLaunch();
const interactive = dependencies.hasInteractiveTerminal();
const startDevelopment = interactive && !agentLaunched && !options.nonInteractive;
```

三个条件同时成立才会在脚手架完成后直接拉起 dev：

- 终端是 TTY；
- **不是在 AI coding agent 里启动的**（`isCodingAgentLaunch()` 调用 vendored 的 `@vercel/detect-agent`）；
- 没传 `-n` / `--non-interactive`。

第二条是关键。你在 Claude Code、Cursor、Codex 的终端里跑 `eve init`，它不会去起一个交互式 TUI（那会卡住调用方），而是打印一段给 agent 看的指引，包含下一步该读哪些文档、该改哪个文件。同一段文字也写进了 `AGENTS.md`。

`agent/agent.ts` 来自 `BASE_AGENT_TEMPLATE`（选网关模型）或 `BYOK_AGENT_TEMPLATE`（自带 key，多一层 `modelOptions.providerOptions.gateway.byok`）。其余文件在 `src/setup/scaffold/create/project.ts` 的 `SHARED_TEMPLATE_FILES` 里，是一张 `Record<string, string>`：

| 文件 | 内容要点 |
| --- | --- |
| `agent/agent.ts` | `defineAgent({ model })`，模型由 init 时的选择决定 |
| `agent/instructions.md` | 身份与行为，从模板生成 |
| `agent/channels/eve.ts` | 默认 HTTP 通道 |
| `package.json` | 依赖 `eve`、`ai`、`zod`、`@vercel/connect`，脚本 `dev`/`build`/`start`/`eval` |
| `tsconfig.json` | `moduleResolution: "bundler"`，`types: ["node", "eve/workflow-modules"]` |
| `AGENTS.md` | 给 AI coding agent 的项目说明 |
| `CLAUDE.md` | 一行 `@AGENTS.md`，复用同一份内容 |
| `.gitignore` | 包含 `.eve`、`.vercel`、`.output`、`.nitro` |
| `.vercelignore` | 单独一份，因为 Vercel CLI 默认不忽略裸 `.env` |

`package.json` 里还有一段容易和 eve 内部别名混淆的配置：

```json
{
  "imports": {
    "#*": "./agent/*",
    "#evals/*": "./evals/*"
  }
}
```

这是**项目侧**的别名：`#tools/read_logs.js` 指向 `agent/tools/read_logs.ts`。和 eve 包内部的 `#cli/`、`#internal/` 是两套独立的映射，只是语法相同。读源码时看到 `#execution/`，那是 eve 自己的；看到 `#tools/`，那是项目作者写的。

最后一条 `.vercelignore` 有注释解释：Vercel CLI 默认忽略 `.env.local` 和 `.env.*.local`，但**不忽略裸 `.env`**。不写进 `.vercelignore`，源码部署会把 `.env` 一起上传。

`package.json` 里的版本号是占位符（`__EVE_INIT_PACKAGE_VERSION__` 之类），在生成时替换成执行 `eve init` 的那个 CLI 的版本。所以脚手架建出来的项目，钉的版本和生成它的 CLI 一致。

## 20.6 build / start / info

**`build`** 有两条路径，按项目形态分：

```ts
// packages/eve/src/cli/commands/build.ts
const projectContext = await input.applicationContext.resolveAgent();
if (projectContext.kind === "workspace") {
  const { buildAgentWorkspace } = await import("#internal/vercel/build-agent-workspace.js");
  const outputDir = await buildAgentWorkspace(projectContext.workspace, {
    skipSandboxPrewarm: options.skipSandboxPrewarm,
  });
  // ...
  return;
}
const buildHost =
  input.buildHost ?? (await import("#internal/nitro/host.js")).buildApplication;
```

`agents/` 工作区走 `buildAgentWorkspace`，单个 agent 走 `buildApplication`（Nitro 宿主）。工作区构建不支持 `--profile`，会直接抛错——因为一次要构建多个 agent，单份 profile 没有意义。

`--skip-sandbox-prewarm` 会跳过沙箱预热。产物仍然能构建，但可能不可部署——部署环境缺少预热好的沙箱镜像。

**`start`** 是生产模式：

```ts
const server = await startProductionHost(applicationContext.root, {
  host: options.host,
  port: options.port,
});
logger.log(renderCliTaggedLine(theme, { message: `server listening at ${server.url}`, tag: "start" }));
await waitForShutdownSignal({ close: () => server.close(), wait: () => server.wait() });
```

注意它用的是 `waitForShutdownSignal`（不带 `exitAfterMs`），没有 900ms 兜底。生产进程退出时宁可慢，也不能截断正在写的会话检查点。

**`info`** 输出 `ApplicationInfoJson`，字段在 `src/cli/commands/info.ts` 的接口里定义。几个容易忽略的：

- `toolInputSchemas.root` / `toolInputSchemas.subagents`：每个工具**发给模型的** schema，不是你在 `defineTool` 里写的那个。中间经过了归一化和 JSON Schema 转换。
- `artifacts`：五个产物的绝对路径（compiledManifest、discoveryManifest、diagnostics、moduleMap、metadata）。排查产物问题时直接 `cat` 这些路径。
- `channels[].urlPath`：每个通道注册的实际路由。默认 eve 通道是 `/eve/v1`。

## 20.7 回到开头那三个困惑

现在可以完整回答了。

**第一次跑命令为什么慢？** `ensureBuiltCli` 发现 `src` 比 `dist` 新，跑了 tsc 和四个 post-build 脚本。改完源码第一次执行卡十几秒是正常的，不是卡死。

**`eve` 怎么判断跑 init 还是 dev？** 向上找最近的 eve 应用根。找到就 dev，找不到就看是不是工作区，都不是就跑 init。

**为什么 Ctrl-C 要按两次？** 第一次给优雅关闭 900ms，第二次直接 `exit(130)`。子进程那侧靠 IPC `disconnect` 感知父进程死亡，即使父进程被 `kill -9` 也能退。

另外三处需要注意：

1. **`eve dev --no-ui` 和 `eve dev` 对「已有 server」的处理不同。** 前者传 `existing: "reject"`，端口被占直接失败；后者传 `existing: "attach-if-unconfigured"`，会尝试附着到已有实例。CI 里用 `--no-ui` 时要自己保证端口空闲。
2. **`kill -9` 父进程后，子进程靠 `disconnect` 退出，但沙箱资源要等下次启动才清理。** `recordCleanupIntent` 写的 `.eve/dev-cleanup-intent.<runId>.json` 留在磁盘上，`reconcileCleanupIntents` 在下一次 `eve dev` 启动时处理。
3. **Ctrl-C 第二次是硬退。** 如果第一次之后 900ms 内没关完，兜底会 `process.exit(0)`；如果连兜底都没触发（比如子进程卡在 uninterruptible sleep），第二次 Ctrl-C 直接 `exit(130)`，会话检查点可能没写完。

## 20.8 练习

1. 在 eve 源码目录里 `touch packages/eve/src/cli/run.ts`，再跑 `eve --version`。用 `time` 量一下耗时，对照 `ensureBuiltCli` 的分支解释为什么慢。
2. `eve dev` 起来后，另开终端 `ps -ef | grep local-server-child`。记下 PID，然后 `kill -9` 那个父进程（不是子进程），看子进程多久消失。对照 `disconnect` 的处理逻辑。
3. 在 `agents/` 工作区里建两个成员，用 `eve info` 不带 `--agent` 跑一次（管道重定向到文件，模拟非 TTY）。看报错内容。
4. 读 `eve info --json` 的输出，找出 `toolInputSchemas.root` 里 `bash` 工具的 schema。和你印象里 `defineTool` 的写法对比，看多了哪些字段。

## 下一章

命令跑起来了，dev server 在跑，TUI 连上了。但服务器启动时会先做一件事：扫描 `agent/` 目录。

第 21 章进到这一步。你会看到「文件名即能力名」这条规则到底是怎么实现的，以及为什么有些文件放进去毫无反应——连报错都没有。

---

返回：[教程目录](README.md) · 上一章：[19 怎么读 eve 的源码](19-reading-the-source.md) · 下一章：[21 discover](21-discover.md)
