# 第 23 章 runtime：产物怎么变成活的 agent

> 一句话结论：编译产物是名词，runtime 负责把名词变成动词——给每个声明挂回真实的函数。挂不回去的，就是运行时错误。

## 一个诡异的计数器

你在工具里写了个计数器，想统计调用次数：

```ts
// agent/tools/counter.ts
let count = 0;

export default defineTool({
  async execute() {
    count += 1;
    return { count };
  },
});
```

测试的时候一切正常：第一次返回 1，第二次 2，第三次 3。

上线后有人报告：他第一次用这个工具，返回的是 47。

这不是 bug，是你把状态放错了地方。`count` 是模块级变量，而模块在整个进程里只执行一次。所有用户、所有会话共享同一个 `count`。

这一章要回答的就是这类问题：**编译产物里的声明，怎么变成能跑的对象，以及这个对象的生命周期有多长**。搞清楚了，你会知道状态该放哪。

## 23.1 两个输入，一个输出

第 22 章留下了两份东西：一份纯 JSON 的 `compiled-agent-manifest.json`，一份可执行的 `module-map.mjs`。

runtime 把它们合成一个对象：

```ts
// packages/eve/src/runtime/resolve-agent.ts
interface ResolveAgentInput {
  manifest: CompiledAgentNodeManifest | CompiledAgentResources;
  moduleMap: CompiledModuleMap;
  nodeId?: string;
}

export async function resolveAgent(input: ResolveAgentInput): Promise<ResolvedAgent> {
  const resolvedSkills = input.manifest.skills.map((skill) => ({ ... }));
  const resolvedChannels = await Promise.all(
    input.manifest.channelRoutes.effective.map((channelEntry) =>
      resolveChannelDefinition(channelEntry, input.moduleMap, input.nodeId),
    ),
  );
  const resolvedTools = await Promise.all(
    input.manifest.tools.map((toolDefinition) =>
      resolveToolDefinition(toolDefinition, input.moduleMap, input.nodeId, ...),
    ),
  );
  // ... 连接、动态解析器、钩子、记忆、沙箱、指令
}
```

一个很直白的函数：遍历 manifest 的每一类资源，调用对应的 `resolveXxxDefinition`，把结果拼成 `ResolvedAgent`。

`Promise.all` 意味着各类资源**并行解析**，但每类内部的顺序被保留（`map` 保序）。这个顺序不是随机的——它来自 discovery 阶段的字母序，第 21 章提过。

## 23.2 声明与实现分离

为什么不能一步到位？因为编译产物要能存进 JSON，而 JSON 存不了函数。

对比同一个工具在两份产物里的样子：

| | `compiled-agent-manifest.json` | `ResolvedToolDefinition` |
| --- | --- | --- |
| 形态 | JSON，可序列化 | 对象，含函数 |
| `inputSchema` | 序列化的 JSON Schema | 活的 zod/JSON Schema 校验器 |
| `execute` | `hasExecute: true`（只是一个布尔） | 真实的 `execute` 函数 |
| `approval` | 归一化后的配置 | 加上 `approvalKey` 函数 |
| `toModelOutput` | 标记存在 | 真实函数 |

manifest 只能存能过 JSON 的东西。函数、类实例、Symbol 都不行。所以编译期把「有哪些函数」记成布尔标志，运行期再按标志去 module map 里取。

这个拆分还有一个副作用：**编译产物可以跨进程传输**。检查点、快照、部署产物都是 JSON，运行时在另一个进程里重新挂函数。这是 eve 能做持久化执行的前提。

## 23.3 一个工具怎么被装回来

```ts
// packages/eve/src/runtime/resolve-tool.ts:23
export async function resolveToolDefinition(
  definition: CompiledToolDefinition,
  moduleMap: CompiledModuleMap,
  nodeId: string | undefined,
  owner: AgentSourceOwner,
): Promise<ResolvedToolDefinition> {
  if (!definition.hasExecute) {
    const resolved: ResolvedToolDefinition = {
      availableInSubagents: definition.availableInSubagents,
      behavior: definition.behavior,
      deferred: definition.deferred,
      description: definition.description,
      inputSchema: toInputSchema(definition.inputSchema),
      logicalPath: definition.logicalPath,
      name: definition.name,
      outputSchema: toOutputSchema(definition.outputSchema),
      owner,
      sourceId: definition.sourceId,
      sourceKind: "module",
    };
    return definition.behavior?.handling?.kind === "provider-tool"
      ? frameworkTool(resolved)
      : resolved;
  }

  const resolvedExportValue = await loadResolvedModuleExport({
    definition,
    kindLabel: "tool",
    moduleMap,
    nodeId,
  });
  const resolvedRecord = expectObjectRecord(resolvedExportValue, describe(definition, "to return an object"));

  const execute =
    definition.behavior?.handling?.kind === "workflow-tool"
      ? undefined
      : (expectFunction(resolvedRecord.execute, describe(definition, "to provide an execute function")) as ...);

  const inputSchema = isToolSchema(resolvedRecord.inputSchema)
    ? resolvedRecord.inputSchema
    : toInputSchema(definition.inputSchema);
  // ...
}
```

三个分支值得注意：

**一、`hasExecute: false` 直接返回声明。**

这类工具没有本地执行函数。典型是 provider tool——模型厂商托管的工具（比如 web search），执行发生在厂商那边，eve 只负责把声明发给模型。

顺带解释了一个现象：你在这类工具的模块顶层写的副作用代码**不会执行**，因为整个模块都没被加载。

**二、`workflow-tool` 的 `execute` 是 `undefined`。**

它的入口在 workflow registry 里，不在进程内。这是第 11 章 `defineWorkflowTool` 的实现方式：编译期把它注册成一个 workflow，运行期这里故意不挂 execute。

**三、schema 有两条来源。**

```ts
const inputSchema = isToolSchema(resolvedRecord.inputSchema)
  ? resolvedRecord.inputSchema
  : toInputSchema(definition.inputSchema);
```

模块导出的 `inputSchema` 如果是活的 `ToolSchema`，优先用它；否则用 manifest 里序列化的那份重建。

为什么要两条？因为 schema 可能是动态的（依赖运行时值），也可能是纯静态的。静态的走 manifest 更快，动态的只能重新求值。

## 23.4 module map 的查找路径

```ts
// packages/eve/src/runtime/resolve-helpers.ts:65
export async function loadResolvedModuleExport(input: {
  readonly definition: {
    readonly exportName?: string;
    readonly logicalPath: string;
    readonly sourceId: string;
  };
  readonly kindLabel: string;
  readonly moduleMap: CompiledModuleMap;
  readonly nodeId: string | undefined;
}): Promise<unknown> {
  const resolvedNodeId = input.nodeId ?? ROOT_COMPILED_AGENT_NODE_ID;
  const moduleNamespace = input.moduleMap.nodes[resolvedNodeId]?.modules[input.definition.sourceId];

  if (moduleNamespace === undefined) {
    throw new ResolveAgentError(
      `Missing compiled module namespace for ${input.kindLabel} source "${input.definition.sourceId}" in node "${resolvedNodeId}".`,
      { logicalPath: input.definition.logicalPath, sourceId: input.definition.sourceId },
    );
  }

  const moduleRecord = expectObjectRecord(moduleNamespace, ...);
  const exportValue = getAuthoredModuleExport(moduleRecord, {
    exportName: input.definition.exportName,
    logicalPath: input.definition.logicalPath,
  });

  return await materializeAuthoredModuleExport(exportValue);
}
```

两级查找：`nodeId` 再 `sourceId`。

`nodeId` 是 `__root__` 或子代理的 id。子代理有独立的 module map scope，所以同名工具在不同子代理里可以是不同的实现。

`sourceId` 是 manifest 里的稳定标识，大多数时候等于 `logicalPath`，扩展挂载时带命名空间前缀。

查不到就抛 `Missing compiled module namespace`。这个错误几乎总是意味着产物和源码不一致——改了文件但没重新编译，或者手工删了 `module-map.mjs` 的一部分。

## 23.5 模块什么时候求值，求值几次

现在回到开头那个计数器。

编译产物里每个模块都带 `usage` 标记：

```json
{
  "backing": { "kind": "programmatic", "moduleId": "tools/bash.ts", "registryId": "eve:defaults" },
  "logicalPath": "tools/bash.ts",
  "owner": { "feature": "eve:defaults", "kind": "framework" },
  "usage": { "compile": true, "runtimeEntry": true }
}
```

两个标志来自 `NodeModuleLifecycle`：

```ts
// packages/eve/src/compiler/module-lifecycle.ts
class NodeModuleLifecycle {
  readonly #compileLoads = new Set<string>();
  readonly #runtimeRoots = new Set<string>();

  recordCompileLoad(sourceId: string): void {
    this.#compileLoads.add(sourceId);
  }

  requireRuntimeEntry(sourceId: string): void {
    this.#runtimeRoots.add(sourceId);
  }

  finalize(
    bindings: Readonly<Record<string, AgentModuleBinding>>,
  ): Record<string, CompiledModuleBinding> {
    const runtimeEntries = collectRuntimeEntryClosure(bindings, this.#runtimeRoots);
    const compiled: Record<string, CompiledModuleBinding> = {};

    for (const [sourceId, binding] of Object.entries(bindings)) {
      const usage = {
        compile: this.#compileLoads.has(sourceId),
        runtimeEntry: runtimeEntries.has(sourceId),
      };
      if (!usage.compile && !usage.runtimeEntry) {
        throw new Error(`Selected module binding "${sourceId}" has no compile or runtime usage.`);
      }
      compiled[sourceId] = Object.freeze({ ...binding, usage: Object.freeze(usage) });
    }

    return compiled;
  }
}
```

- `compile: true`：编译期求值过（比如 `agent.ts`，要读它的配置）。
- `runtimeEntry: true`：运行期还要再求值一次（比如工具的 `execute`，编译期只是检查形状）。

`collectRuntimeEntryClosure` 会把依赖闭包一起标上——一个 runtime entry 依赖的模块也必须能在运行期加载。

注意那个 `throw`：两个标志都为 `false` 的 binding 会让编译直接失败。所以产物里每个模块至少有一个 `usage` 为 `true`，不存在「编译期用过、运行期不用、但还留在产物里」的僵尸条目。

**关键推论：模块只求值一次，全局共享。**

ESM 保证一个模块在同一个进程里只执行一次。所以开头那个 `count`：

- 不是「每个会话一份」
- 不是「每个用户一份」
- 是「整个进程一份」

dev 下重启进程会归零，生产下所有用户共享同一个计数器。

要按会话隔离，状态必须放进 `ctx`（第 8 章的 `defineState`）或者沙箱文件系统。模块级变量在 eve 里是最容易误用的东西，因为它看起来像「文件私有」，实际是「进程全局」。

## 23.6 状态该放哪：context key

`ctx` 的键不是字符串，是 `ContextKey` 实例：

```ts
// packages/eve/src/context/key.ts:16
const KEY_REGISTRY_GLOBAL_KEY = Symbol.for("eve.context-key-registry");
const globalKeyRegistryContainer = globalThis as KeyRegistryGlobal;

if (globalKeyRegistryContainer[KEY_REGISTRY_GLOBAL_KEY] === undefined) {
  globalKeyRegistryContainer[KEY_REGISTRY_GLOBAL_KEY] = new Map<string, ContextKey<unknown>>();
}

const keyRegistry = globalKeyRegistryContainer[KEY_REGISTRY_GLOBAL_KEY];

export class ContextKey<T> {
  readonly name: string;
  readonly codec?: ContextKeyCodec<T>;

  constructor(name: string, options: ContextKeyOptions<T> = {}) {
    this.name = name;
    this.codec = options.codec;

    const existing = keyRegistry.get(name);
    if (existing !== undefined && (existing.codec === undefined) !== (this.codec === undefined)) {
      throw new Error(
        `ContextKey name collision: "${name}" is already registered ${
          existing.codec ? "with" : "without"
        } a codec, but a key ${this.codec ? "with" : "without"} a codec is being registered under the same name. This silently breaks context serialization — use a distinct name.`,
      );
    }

    keyRegistry.set(name, this as ContextKey<unknown>);
  }
}
```

两个设计点：

**一、注册表挂在 `globalThis` 上，用 `Symbol.for` 做键。**

注释解释了原因：Nitro 会把 eve 内联进构建产物，磁盘导入的 eve 是另一份。两份代码各有一份 `key.ts` 的模块实例，但 `Symbol.for` 是全局符号表，两边拿到同一个 Map。不这么做，A chunk 注册的 key 在 B chunk 里 `resolveKey` 找不到，序列化时会静默丢数据。

**二、同名 key 但 codec 不一致直接抛错。**

这是防「静默损坏」：如果带 codec 的 key 先注册，然后一个不带 codec 的同名 key 覆盖了注册表，序列化时值会以原始形式存储，读回来就不是原来的类型了。这种 bug 极难查，所以选择在注册时就炸。

写自定义 context key 时，名字要足够独特。`"state"`、`"config"` 这种一定会撞。

## 23.7 有效配置：manifest 之上还有一层

`resolveAgent` 产出的是「编译时确定的配置」。运行时还有一层覆盖：

```ts
// packages/eve/src/execution/effective-agent-config.ts
export function resolveEffectiveAgentRuntimeFromConfig(
  bundle: Pick<CompiledBundle, "resolvedAgent" | "turnAgent">,
  config: DynamicSubagentAgentConfig | undefined,
): EffectiveAgentRuntime {
  if (config === undefined) {
    if (bundle.turnAgent.configResolver === true) {
      throw new Error("Dynamic subagent execution requires a selected concrete agent config.");
    }
    return {
      limits: bundle.resolvedAgent.config?.limits,
      thresholdPercent: bundle.resolvedAgent.config?.compaction?.thresholdPercent,
      turnAgent: bundle.turnAgent,
    };
  }

  const { compactionModel: _c, configResolver: _r, dynamicModel: _d, model: _m, ...turnAgent } = bundle.turnAgent;
  return {
    limits: config.limits,
    thresholdPercent: config.compaction?.thresholdPercent,
    turnAgent: {
      ...turnAgent,
      compactionModel: config.compaction?.model,
      model: config.model,
      reasoning: config.reasoning,
    },
  };
}
```

配置来源从 context 的 `DynamicSubagentAgentConfigKey` 读。动态子代理（第 18 章）在运行时决定用哪个模型，覆盖就发生在这里。

`configResolver === true` 却没有具体配置时抛错——这是防止「声明了动态配置但没选具体值」的中间状态被当成正常情况跑下去。

## 23.8 回到那个计数器

`count` 的问题出在 23.5：模块只求值一次，`let count = 0` 只执行一次，之后所有调用共享同一个闭包。

三种改法，按推荐顺序：

1. **放 `ctx`。** 用 `defineState` 声明会话级状态，eve 负责在 step 边界序列化、恢复。这是标准做法。
2. **放沙箱文件。** 如果状态需要跨进程共享（比如多个副本），写进沙箱文件系统。
3. **放外部存储。** Redis、数据库。适合真正需要全局共享的计数。

模块级变量只在一种情况下合理：**只读的常量**。比如预编译的正则、静态配置表。任何可变的都不要放在模块顶层。

## 23.9 五个注意事项

1. **模块级变量全进程共享。** 见 23.5。这是 eve 里最常见的状态隔离 bug。
2. **`hasExecute: false` 的工具不会加载模块。** 你在模块顶层写的副作用代码不会执行。判断依据是编译产物里的这个布尔值，不是你写了什么。
3. **`workflow-tool` 的 `execute` 永远是 `undefined`。** 别在代码里判断 `tool.execute === undefined` 就认为是配置错误——先看 `behavior.handling.kind`。
4. **同名 context key 且 codec 不一致会直接抛错。** 名字起得越通用，越容易撞。
5. **`Missing compiled module namespace` 通常意味着产物过期。** 删掉 `.eve/compile/` 重新编译，比读代码快。

## 23.10 练习

1. 在 `agent/tools/` 下写一个带模块级计数器的工具，连续调用三次，看返回值。再开一个新会话调用，看计数器是否重置。
2. 在 `compiled-agent-manifest.json` 里搜 `"kind": "provider-tool"`，找到对应的工具，确认它的 `hasExecute` 是 `false`。再解释为什么这类工具不需要 module map 里的绑定。
3. 在 `agent/lib/` 下定义两个同名但 codec 不同的 `ContextKey`，跑 `eve dev`，看报错信息。
4. 打开 `compiled-agent-manifest.json`，找出 `usage.compile` 和 `usage.runtimeEntry` 都为 `true` 的模块，解释为什么它两边都要加载。

## 下一章

对象装好了，能跑了。但「跑」这件事在 eve 里不是一次函数调用，而是一串 step。

第 24 章讲 workflow：一个会话为什么被切成很多步，每步结束时存了什么，以及为什么崩溃之后你的工具可能被执行两次。

---

返回：[教程目录](README.md) · 上一章：[22 compiler](22-compiler.md) · 下一章：[24 workflow](24-workflow.md)
