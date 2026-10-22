# 第 21 章 discover：文件系统怎么变成 manifest

> 一句话结论：文件名不是「约定」，是一张精确的分类表加一组正则。文件没被识别，九成是撞了表里的某一条——而且失败方式是静默的。

## 两个让人抓狂的瞬间

第一个：你在 `agent/tools/` 下新建了 `read-logs.ts`，写得好好的。跑 `eve dev`，问 agent「你有 read_logs 工具吗」，它说没有。文件明明在那儿。

第二个：你想给 agent 加点背景资料，建了个 `agent/prompts/` 目录，放了几个 md 文件。完全没反应，连个警告都没有。

这两件事的答案都在同一个地方：eve 启动时会扫一遍 `agent/` 目录，但**只认它认识的目录名和文件名**。不认识的东西不会报错，只会被忽略。

第 3 章讲过「什么该放哪」。这一章讲那些规则在代码里长什么样，以及它们怎么失败。搞懂之后，上面两个问题你会一眼看出原因。

## 21.1 扫描器长什么样

整个扫描的入口只有三个输入：

```ts
// packages/eve/src/discover/discover-agent.ts
export async function discoverAgent(input: DiscoverAgentInput): Promise<DiscoverAgentResult> {
  const source = input.source ?? createDiskProjectSource();
  const appRoot = resolve(input.appRoot);
  const agentRoot = resolve(input.agentRoot);
  const role = input.role ?? "agent";
  // ...
}
```

`agentRoot`、`appRoot`、`role`。`role` 是 `"agent"` 或 `"extension"`——扩展的源树用同一套扫描器，但禁用一部分槽位。

输出是 `{ manifest, diagnostics }`。manifest 是一份纯数据的清单：

```ts
// packages/eve/src/discover/manifest.ts
export interface AgentSourceManifest {
  agentId: string;
  agentRoot: string;
  appRoot: string;
  channels: ChannelSourceRef[];
  connections: ConnectionSourceRef[];
  configModule?: ModuleSourceRef;
  extensions: ExtensionSourceRef[];
  hooks: ModuleSourceRef[];
  instructions: InstructionsSourceRef[];
  lib: LibSourceRef[];
  memories: MemorySourceRef[];
  sandbox: SandboxSourceRef | null;
  schedules: ScheduleSourceRef[];
  skills: SkillSourceRef[];
  subagents: SubagentSourceRef[];
  tools: ModuleSourceRef[];
  diagnosticsSummary: { errors: number; warnings: number };
  kind: "eve-agent-discovery-manifest";
  version: 15;
}
```

注意这里**没有** `eve:defaults` 里的 bash、read_file、write_file。discovery 只记录**作者写的**东西。框架默认能力在编译阶段才注入，那是第 22 章的事。

所以「扫描」这件事本身只做一件事：把目录里的文件，变成上面这张表里的一行。下面五个问题，就是它做判断的全部依据。

## 21.2 第一个问题：哪些文件算数

所有判断集中在一个函数里，一张巨大的 if 表：

```ts
// packages/eve/src/discover/filesystem.ts:145
export function classifyAgentRootEntry(
  name: string,
  entryType: DirectoryEntryType,
): AgentRootEntryKind {
  if (entryType === "file") {
    if (matchesSupportedModuleBaseName(name, "agent")) {
      return "agent-config-module";
    }

    if (name.toLowerCase() === "instructions.md") {
      return "instructions-markdown";
    }
    // ... 其余文件槽位

    return "unknown";
  }

  if (entryType === "directory") {
    if (GENERATED_AGENT_DIRECTORY_NAMES.has(name)) {
      return "ignored-directory";
    }

    if (name === "channels") {
      return "channels-directory";
    }
    // ... 其余目录槽位，完整列表见下表

  }

  return "unknown";
}
```

展开成完整清单：

| 条目 | 类型 | 识别为 |
| --- | --- | --- |
| `agent.{ts,js,cts,mts,cjs,mjs}` | 文件 | 根配置模块 |
| `instructions.md` | 文件 | 指令（markdown） |
| `instructions.{ts,...}` | 文件 | 指令（模块） |
| `instructions/` | 目录 | 多条指令，文件名排序 |
| `memory.{ts,...}` / `memory/` | 文件/目录 | 记忆 provider |
| `system.md` / `system.{ts,...}` | 文件 | 旧版指令，**已废弃** |
| `tools/` `channels/` `connections/` `hooks/` `schedules/` `skills/` `subagents/` `sandbox/` `extensions/` | 目录 | 各自的能力槽位 |
| `lib/` | 目录 | 共享代码，不算能力 |
| `instrumentation/` | 目录 | 追踪埋点 |
| `.devtools/` `.eve/` `.next/` `.output/` `.vercel/` `node_modules/` | 目录 | 静默忽略 |

不在表里的目录不会报错，只会生成一条 warning：

```text
Ignoring unsupported directory "foo/" in the agent root.
```

这条 warning 默认不显示——`reportDiscoverWarnings` 只在编译时 `console.warn`，而 `eve dev` 的启动输出会把它刷过去。所以「我建了个 `prompts/` 目录，为什么没生效」的答案永远是：它被忽略了，没有任何东西读它。

### 一处大小写不一致

`instructions.md` 和 `system.md` 用 `toLowerCase()` 比较，`agent.ts`、`memory.ts` 用精确匹配：

| 文件名 | 是否识别 |
| --- | --- |
| `instructions.md` / `INSTRUCTIONS.MD` / `Instructions.Md` | 都识别 |
| `agent.ts` | 识别 |
| `Agent.ts` / `AGENT.ts` | **不识别**，落到 unknown |

这不是笔误，是历史遗留：markdown 槽位早期为了兼容 Windows 大小写不敏感的文件系统放宽了比较，模块槽位没跟着改。写文件名时统一用小写，能绕开这个不一致。

## 21.3 第二个问题：哪些扩展名算模块

```ts
// packages/eve/src/discover/filesystem.ts:7
export const SUPPORTED_AUTHORED_MODULE_FILE_EXTENSIONS = [
  ".cts",
  ".mts",
  ".cjs",
  ".mjs",
  ".ts",
  ".js",
] as const;
```

六种扩展名。注意 `.tsx` 和 `.jsx` **不在里面**——eve 的 agent 目录不处理 JSX，通道的 UI 在别处（第 17 章的前端）。

光有扩展名还不够，还有一套排除规则：

```ts
export function getSupportedModuleBaseName(name: string): string | null {
  if (isTypeScriptDeclarationFileName(name)) return null;   // .d.ts / .d.cts / .d.mts
  for (const extension of SUPPORTED_AUTHORED_MODULE_FILE_EXTENSIONS) {
    if (name.endsWith(extension) && name.length > extension.length) {
      return name.slice(0, -extension.length);
    }
  }
  return null;
}

export function isAuthoredTestPath(path: string): boolean {
  return /(?:^|[/\\])__tests__(?:[/\\]|$)/.test(path) || /\.(?:test|spec)\.[cm]?[jt]s$/.test(path);
}
```

三类文件被跳过：

- `.d.ts`：类型声明不是实现。想在 `tools/` 里放类型定义可以，但别指望它变成工具。
- `*.test.ts` / `*.spec.ts` / `__tests__/`：测试文件不进 agent。
- `*.map`：sourcemap 忽略。

最后一条有个细节：`isGeneratedSourceMapFileName` 会检查 `.map` 前面的文件名是不是合法模块或声明，所以 `tools/foo.ts.map` 被忽略，但 `tools/foo.map` 不会——后者会被当成一个未识别文件。

## 21.4 第三个问题：目录怎么往里走

所有 `xxx/` 目录都走同一个 walker：

```ts
// packages/eve/src/discover/named-source-directory.ts:109
// 省略了两个重载签名，只留实现
export async function discoverNamedSourceDirectory(input): Promise<{
  diagnostics: DiscoverDiagnostic[];
  sources: (MarkdownSourceRef<TDefinition> | ModuleSourceRef)[];
}> {
  const directoryPath = join(input.rootPath, input.directoryName);
  const directoryEntry = input.rootEntries.find((entry) => entry.name === input.directoryName);

  if (directoryEntry === undefined) {
    return { diagnostics: [], sources: [] };
  }

  if (!directoryEntry.isDirectory()) {
    return {
      diagnostics: [
        createDiscoverErrorDiagnostic({
          code: input.invalidDirectoryCode,
          message: input.invalidDirectoryMessage,
          sourcePath: directoryPath,
        }),
      ],
      sources: [],
    };
  }
  // ... 递归或平铺遍历
}
```

行为由三个参数决定：`recursive`、`allowMarkdown`、`validateSegment`。

| 槽位 | recursive | markdown | 名字校验 |
| --- | --- | --- | --- |
| `tools/` | 是 | 否 | `TOOL_SLUG_PATTERN` |
| `channels/` | 是 | 否 | `CHANNEL_SLUG_PATTERN` |
| `hooks/` | 是 | 否 | `HOOK_SLUG_PATTERN` |
| `extensions/` | **否** | 否 | `EXTENSION_SLUG_PATTERN` |
| `lib/` | 是 | 否 | 无 |
| `instructions/` | **否** | 是 | 无 |
| `schedules/` | 是 | 是 | 无 |
| `connections/` | 是 | 否 | `CONNECTION_SLUG_PATTERN` |

`tools/` 是递归的，这一点常被误解。`named-source-directory.ts` 的注释里写「Used by the `tools/` (flat)」，但调用点传的是 `recursive: true`：

```ts
// packages/eve/src/discover/discover-agent.ts
const toolsResult = await discoverNamedSourceDirectory({
  directoryName: "tools",
  invalidDirectoryCode: DISCOVER_TOOLS_DIRECTORY_INVALID,
  recursive: true,
  rootEntries,
  rootPath: agentRoot,
  source,
  validateSegment: createToolNameDiagnostic,
});
```

注释过期了，调用点是事实。这是第 19 章那条「不信注释信调用点」的现成案例。

递归带来一个后果：`tools/` 下可以建子目录，但每个路径段都要过 `createToolNameDiagnostic` 的校验。`tools/redis/overview.ts` 合法，`tools/redis cache/overview.ts` 不合法——`redis cache` 里的空格不在允许字符里。

## 21.5 第四个问题：名字怎么算

文件名去掉扩展名就是 slot 名：

```ts
// packages/eve/src/discover/grammar.ts
export const TOOL_SLUG_PATTERN = /^[a-zA-Z][a-zA-Z0-9_-]{0,63}$/;
```

规则：字母开头，后面是字母、数字、下划线、连字符，总长不超过 64。`read-logs.ts` 合法，`read logs.ts`、`1read.ts`、`read.logs.ts` 都不合法——最后一个会得到 `read.logs` 这个名字，点号不在允许字符里。

递归目录下，名字用连字符压平：

```ts
// packages/eve/src/compiler/normalize-tool.ts:53
const toolName = stripLogicalPathExtension(source.logicalPath)
  .replace(/^tools\//, "")
  .replaceAll("/", "-");
```

所以：

| 文件 | logicalPath | 工具名 |
| --- | --- | --- |
| `tools/read_logs.ts` | `tools/read_logs.ts` | `read_logs` |
| `tools/redis/overview.ts` | `tools/redis/overview.ts` | `redis-overview` |
| `tools/redis/overview/info.ts` | `tools/redis/overview/info.ts` | `redis-overview-info` |

目录层级被压平成一维名字。这带来一个隐蔽的冲突：`tools/redis-overview.ts` 和 `tools/redis/overview.ts` 都会得到 `redis-overview`。discovery 阶段不会报错，因为它们的 logicalPath 不同；冲突要等到编译时按名字去重才暴露。

### 保留的名字

同一个 `normalize-tool.ts` 里还有命名空间检查：

```ts
const reservation = eveNamespaceReservation(toolName);
if (reservation !== undefined) {
  throw new Error(`Tool "${source.logicalPath}" uses the reserved name "${toolName}". ...`);
}
```

`eveNamespaceReservation` 只保留两样东西：

```ts
// packages/eve/src/protocol/runtime-tools.ts:5
const EVE_NAMESPACE = "eve";

export function eveNamespaceReservation(name: string): string | undefined {
  return name === EVE_NAMESPACE || name.startsWith(`${EVE_NAMESPACE}__`)
    ? `eve reserves the "${EVE_NAMESPACE}" namespace for its built-in tools`
    : undefined;
}
```

所以 `tools/eve.ts` 和 `tools/eve__debug.ts` 会编译失败，其它名字都可以用——包括 `bash`、`read_file` 这些默认工具的名字。

**用默认工具的名字不会报错，会覆盖它。** 覆盖规则在 `source-graph.ts` 的层级优先级里：

```ts
const LAYER_PRECEDENCE: Readonly<Record<AgentSourceLayer, number>> = {
  "framework-default": 0,
  "extension-package": 1,
  "extension-override": 2,
  application: 3,
};
```

`application`（你写的）优先级最高。写一个 `tools/bash.ts` 就把框架默认的 bash 换掉了，被换掉的那个记作 `shadowed`，在编译产物里能查到。这个机制在第 22 章展开。

## 21.6 第五个问题：撞名了怎么办

同名文件同时存在时，**不是二选一，是整个 slot 消失**：

```ts
// packages/eve/src/discover/named-source-directory.ts · collectLeafSources()
if (candidates.markdownFileName !== undefined && candidates.moduleFileNames.length > 0) {
  input.diagnostics.push(
    createSlotCollisionDiagnostic(absoluteDirectory, slotLogicalPath, [
      candidates.markdownFileName,
      ...candidates.moduleFileNames,
    ]),
  );
  continue;   // ← 这个 slot 没了
}

if (candidates.moduleFileNames.length > 1) {
  input.diagnostics.push(
    createModuleSlotCollisionDiagnostic(absoluteDirectory, slotLogicalPath, candidates.moduleFileNames),
  );
  continue;
}
```

两种碰撞：

1. `schedules/daily.md` 和 `schedules/daily.ts` 同时存在 → `discover/slot-collision`
2. `tools/read_logs.ts` 和 `tools/read_logs.js` 同时存在 → `discover/module-slot-collision`

第二种最常见：编辑器自动生成 `.js`（比如 `tsc` 没配 `noEmit`），或者从别处拷文件时带了一份编译产物。

诊断信息里会列出所有冲突文件名，但**不会告诉你要保留哪个**。它只说「这几个文件撞了」。

`continue` 的后果是：这个工具/技能/定时任务完全不存在。不是用了其中一个，是一个都没有。

## 21.7 失败时怎么告诉你

诊断有两种级别，后果完全不同：

```ts
// packages/eve/src/compiler/compile-agent.ts:152
function finishAgentCompilation(result, createError): CompileAgentResult {
  if (result.diagnostics.some((diagnostic) => diagnostic.severity === "error")) {
    throw createError(result);
  }
  reportDiscoverWarnings(result.diagnostics);
  return result;
}
```

| severity | 后果 | 什么时候出现 |
| --- | --- | --- |
| `error` | 编译抛错，`eve dev` 起不来 | 名字非法、槽位冲突、目录该是目录却是文件、缺 instructions |
| `warning` | 打印一行，继续编译 | 未知根目录、`system.*` 废弃提醒 |

常见 error 码：

| 码 | 含义 | 修法 |
| --- | --- | --- |
| `discover/tool-name-invalid` | 工具文件名不合法 | 改成字母开头、只含 `[a-zA-Z0-9_-]` |
| `discover/slot-collision` | 同名 md 和模块 | 删掉一个 |
| `discover/module-slot-collision` | 同名多扩展名模块 | 删掉编译产物 |
| `discover/tools-directory-invalid` | `tools` 是文件不是目录 | 改名 |
| `discover/required-instructions-missing` | 没有 instructions | 加 `instructions.md` |
| `discover/unsupported-directory` | 未知根目录 | warning，不是 error |

诊断结果落在 `.eve/discovery/diagnostics.json`：

```json
{
  "diagnostics": [],
  "kind": "eve-compiler-diagnostics",
  "summary": { "errors": 0, "warnings": 0 },
  "version": 2
}
```

`eve info --json` 的 `diagnostics` 字段就是这里的 `summary`。排查「为什么我的能力没出现」时，先看这个文件，比翻终端快。

## 21.8 产物长什么样

打开一个真实项目的 `.eve/discovery/agent-discovery-manifest.json`（`instructions.content` 为可读性截断）：

```json
{
  "agentId": "opsbot",
  "agentRoot": "/path/to/app/agent",
  "appRoot": "/path/to/app",
  "channels": [
    { "sourceKind": "module", "logicalPath": "channels/eve.ts", "sourceId": "channels/eve.ts" }
  ],
  "connections": [],
  "diagnosticsSummary": { "errors": 0, "warnings": 0 },
  "extensions": [],
  "resolvedExtensions": [],
  "hooks": [],
  "memories": [],
  "instructions": [
    {
      "definition": {
        "content": "# Identity\n\nYou are a general-purpose AI agent powered by eve...",
        "role": "system"
      },
      "sourceKind": "markdown",
      "logicalPath": "instructions.md",
      "sourceId": "instructions.md"
    }
  ],
  "lib": [],
  "kind": "eve-agent-discovery-manifest",
  "sandbox": null,
  "sandboxWorkspaces": [],
  "schedules": [],
  "skills": [],
  "tools": [],
  "version": 15,
  "subagents": [],
  "configModule": { "sourceKind": "module", "logicalPath": "agent.ts", "sourceId": "agent.ts" }
}
```

四个值得注意的点：

1. **`tools: []` 不代表没有工具。** 这个项目确实没写工具，但即使写了，这里也只列作者的工具。默认的 bash、read_file 在编译产物里才出现。
2. **markdown 的内容被内联了。** `instructions` 的 `definition.content` 是全文，不是路径引用。所以改 `instructions.md` 会改变 manifest 的哈希，触发重新编译。
3. **模块只存路径。** `configModule` 只有 `logicalPath`，没有内容——模块要等 runtime 阶段才求值（第 23 章）。
4. **`sourceId` 和 `logicalPath` 大多数时候相同。** 扩展挂载时会不同，`sourceId` 带命名空间前缀。

## 21.9 子代理的规则更严

子代理用另一张分类表：

```ts
// packages/eve/src/discover/filesystem.ts:237
export function classifyLocalSubagentEntry(
  name: string,
  entryType: DirectoryEntryType,
): LocalSubagentEntryKind {
  // ... 文件槽位与 classifyAgentRootEntry 相同
  // ... 目录槽位少了 channels 和 instrumentation
  if (name === "schedules") {
    return "invalid-schedules-directory";
  }
  // ...
}
```

对比：

| 槽位 | 根 agent | 子代理 |
| --- | --- | --- |
| `tools/` `hooks/` `skills/` `connections/` `memory/` `sandbox/` `lib/` `subagents/` | 有 | 有 |
| `channels/` | 有 | **没有** |
| `instrumentation/` | 有 | **没有** |
| `schedules/` | 有 | **报错** |

为什么少这三个？因为子代理不是一个独立部署单元——它只在父 agent 的会话里被调用。通道需要独立的 HTTP 入口，定时任务需要一个能独立启动的入口，两者都不成立。埋点则统一由根 agent 负责。

写 `subagents/researcher/schedules/daily.ts` 会得到 `discover/schedules-directory-invalid` 的 error，不是静默忽略。

## 21.10 回到开头那两个瞬间

现在能解释了。

**`read-logs.ts` 为什么没被识别？** 三种可能，按概率排：

1. 名字不合法——`read-logs` 其实合法，但如果叫 `read.logs.ts` 就会得到 `read.logs`，点号过不了正则。
2. 同名文件撞了——比如 `read_logs.ts` 和 `read_logs.js` 同时存在，整个 slot 消失。
3. 文件写在了 `tools/` 之外的目录，被当成未知目录忽略了。

排查顺序：看 `.eve/discovery/diagnostics.json` 的 `summary.errors`，非 0 就有明确原因；为 0 就看 manifest 的 `tools` 数组里有没有它。

**`agent/prompts/` 为什么完全没反应？** 它不在分类表里，被当成未知目录，只生成一条 warning。而 warning 在 `eve dev` 的启动日志里会被刷过去。

## 21.11 五个注意事项

1. **未知目录静默忽略。** 建了 `prompts/`、`context/`、`docs/` 都不会报错，只是没人读。唯一的线索是编译时的 warning，而它默认被启动日志刷掉。
2. **`agent.ts` 大小写敏感，`instructions.md` 不敏感。** 见 21.2 的表。
3. **`tools/` 递归，工具名用连字符压平。** `tools/a/b.ts` 叫 `a-b`，不是 `b`。两个不同路径可能撞同一个名字，discovery 不报，编译才报。
4. **同名文件导致整个 slot 消失。** 不是取其一，是一个都没有。`.js` 和 `.ts` 同时存在是最常见的触发方式。
5. **`.d.ts` 不算模块。** 想在 `tools/` 里放类型定义可以，但别指望它变成工具。

## 21.12 练习

1. 在 `agent/tools/` 下建 `read-logs.ts` 和 `read_logs.ts`，跑 `eve info --json`，看 `tools` 数组里有几个。再改成 `read.logs.ts`，看诊断码变成什么。
2. 建 `agent/tools/redis/overview.ts`，导出任意 `defineTool`。用 `eve info --json` 查 `toolInputSchemas.root` 的 key，确认工具名是 `redis-overview`。
3. 把 `agent/instructions.md` 复制一份叫 `INSTRUCTIONS.MD`，跑一次 `eve dev`。看 `.eve/discovery/agent-discovery-manifest.json` 里 `instructions` 的 `logicalPath` 是哪个文件名——两个都匹配同一个槽位时不会报错，最终用哪个由 `localeCompare` 的排序结果决定，没有确定性保证。
4. 建 `agent/prompts/system.md`，跑 `eve build`，然后在 `.eve/discovery/diagnostics.json` 里找那条 warning。
5. 在 `agent/subagents/helper/` 下建 `schedules/daily.ts`，确认编译报 `discover/schedules-directory-invalid`。

## 下一章

扫描完成，`AgentSourceManifest` 交出去了。接下来要把它变成能跑的东西。

第 22 章讲编译。你会看到同一份 manifest 怎么和框架默认能力合并、为什么写 `tools/bash.ts` 能覆盖内置的 bash、以及 dev 下为什么改一行代码不影响正在跑的会话。

---

返回：[教程目录](README.md) · 上一章：[20 CLI 启动链路](20-cli-internals.md) · 下一章：[22 compiler](22-compiler.md)
