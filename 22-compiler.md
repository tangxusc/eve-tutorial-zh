# 第 22 章 compiler：manifest 怎么变成产物

> 一句话结论：编译不是「把文件转成另一种格式」，是四层候选按优先级合并——赢家写进产物，输家记作 `shadowed`。你的 `tools/bash.ts` 能覆盖框架默认工具，靠的就是这张优先级表。

## 两个反直觉的现象

第一个：你在 `agent/tools/` 下写了一个 `bash.ts`，心里没底——会不会和内置的 bash 冲突？会不会报错？

答案是都不。它**覆盖**了内置的 bash，被换掉的那个悄悄记在编译产物的某个字段里，没有任何提示。

第二个：你改了一行工具代码，`eve dev` 的 HMR 提示更新了，但在 TUI 里接着聊，行为还是旧的。开一个新会话，新代码才生效。

这两个现象都出在编译这一层。第 21 章结束时，discover 交出一份 `AgentSourceManifest`，里面只有「作者写了什么」——纯数据，没有函数、没有默认能力。这一章讲它怎么变成 `.eve/compile/` 下能跑的东西，以及上面两个现象为什么会发生。

## 22.1 一次编译做了什么

入口很短：

```ts
// packages/eve/src/compiler/compile-agent.ts:79
export async function compileAgent(input: CompileAgentInput = {}): Promise<CompileAgentResult> {
  const discovered = await discoverAgentForCompilation(input);
  const artifactsRoot = join(discovered.project.appRoot, ".eve");
  const result = await writeAgentCompilation(discovered, {
    publishedRoot: artifactsRoot,
    writeRoot: artifactsRoot,
  });

  return finishAgentCompilation(result, CompileAgentError.fromDurableArtifacts);
}
```

三步：discover → 写产物 → 检查诊断（有 error 就抛）。

产物路径集中在一个函数里：

```ts
// packages/eve/src/compiler/artifacts.ts:130
function resolveCompilerArtifactPathsAt(appRoot, artifactsRoot): CompilerArtifactPaths {
  const discoveryDirectoryPath = join(resolvedArtifactsRoot, "discovery");
  const compileDirectoryPath = join(resolvedArtifactsRoot, "compile");

  return {
    appRoot: resolvedAppRoot,
    compiledManifestPath: join(compileDirectoryPath, "compiled-agent-manifest.json"),
    compileDirectoryPath,
    compileMetadataPath: join(compileDirectoryPath, "compile-metadata.json"),
    diagnosticsPath: join(discoveryDirectoryPath, "diagnostics.json"),
    discoveryManifestPath: join(discoveryDirectoryPath, "agent-discovery-manifest.json"),
    discoveryDirectoryPath,
    moduleMapPath: join(compileDirectoryPath, "module-map.mjs"),
    sandboxPreparedArtifactsPath: join(compileDirectoryPath, "sandbox-prepared-artifacts.json"),
  };
}
```

六个文件，分两个目录：

| 文件 | 内容 | 谁读 |
| --- | --- | --- |
| `discovery/agent-discovery-manifest.json` | 作者写了什么（第 21 章） | `eve info`、调试 |
| `discovery/diagnostics.json` | 诊断列表 | `eve info --json` |
| `compile/compiled-agent-manifest.json` | 编译后的完整配置，含默认能力 | runtime（第 23 章） |
| `compile/module-map.mjs` | 源码模块的导入表 | Node ESM loader |
| `compile/compile-metadata.json` | 四个产物的 sha256 | dev HMR |
| `compile/sandbox-prepared-artifacts.json` | 沙箱预热产物清单 | 沙箱启动 |

前两个是 discover 的产物，后四个才是编译的产物。分开的原因：discovery 只关心「文件在哪」，编译关心「代码是什么」。

## 22.2 九个步骤

```ts
// packages/eve/src/compiler/artifacts.ts:221
export async function writeCompilerArtifacts(
  input: WriteCompilerArtifactsInput,
): Promise<WriteCompilerArtifactsResult> {
  const paths = resolveCompilerArtifactPathsAt(input.appRoot, input.artifactLocations.writeRoot);
  const publishedPaths = resolveCompilerArtifactPathsAt(
    input.appRoot,
    input.artifactLocations.publishedRoot,
  );
  const diagnostics = input.diagnostics.map((diagnostic) =>
    projectDiscoverDiagnostic(diagnostic, ROOT_COMPILED_AGENT_NODE_ID),
  );
  const compiledManifest = await materializeWorkspaceResources({
    compileDirectoryPath: paths.compileDirectoryPath,
    manifest: await compileAgentManifest(input.manifest, {
      developmentExtensions: input.developmentExtensions,
      diagnostics,
    }),
  });
  const diagnosticsArtifact = createCompilerDiagnosticsArtifact(diagnostics);
  // ... 序列化、算 hash、写文件
}
```

展开成步骤：

1. 解析写入路径和发布路径。dev 下两者相同；构建到临时目录时不同（写进去的是临时目录，metadata 里记的是发布位置）。
2. 把 discover 诊断转成 compiler 诊断，挂上 node id（根节点是 `__root__`）。
3. `compileAgentManifest` —— 核心，下一节展开。
4. `materializeWorkspaceResources` —— 把沙箱 workspace 和技能文件复制到 `.eve/compile/workspace-resources/<node>/`。
5. 生成 diagnostics artifact。
6. 序列化三份 JSON（compiled manifest、discovery manifest、diagnostics）。
7. `createCompiledModuleMapSource` —— 生成 `module-map.mjs`。
8. `createCompileMetadata` —— 算四个 sha256 和 `sourceGraphHash`。
9. 并发写六个文件。

第 1 步的两个 root 值得注意：`writeRoot` 和 `publishedRoot`。`compileAgentInWorkspace` 用它们把产物写到一个地方、却记录另一个地方的路径。这是给「先在临时目录构建，再整体搬到输出目录」的场景用的。

## 22.3 为什么需要「合并」

如果编译只是「把 manifest 转成另一种 JSON」，那 207 行就够了。真正复杂的是：**同一个 slot 可以有多个来源**。

比如 `tools/bash`，至少有两份：

- `eve:defaults:tools/bash.ts` —— 框架自带的
- `tools/bash.ts` —— 你写的

还有可能来自扩展包。编译器的任务是选一个，并且让被选中的那个能跑。

选法是一张优先级表：

```ts
// packages/eve/src/compiler/source-graph.ts:262
const LAYER_PRECEDENCE: Readonly<Record<AgentSourceLayer, number>> = {
  "framework-default": 0,
  "extension-package": 1,
  "extension-override": 2,
  application: 3,
};

const FORM_PRECEDENCE: Readonly<Record<AgentSourceForm, number>> = {
  derived: 0,
  direct: 1,
};
```

| 层 | 谁提供 | 例子 |
| --- | --- | --- |
| `framework-default` | eve 自己 | `eve:defaults:tools/bash.ts` |
| `extension-package` | 安装的扩展包 | 扩展自带的工具 |
| `extension-override` | 扩展的覆盖声明 | 扩展替换另一个扩展的槽位 |
| `application` | 你的 `agent/` 目录 | `tools/bash.ts` |

数字大的赢。所以你的 `tools/bash.ts`（3）覆盖框架默认的 bash（0）——这就是本章开头第一个现象的答案。

同层同槽位有两个候选会怎样？直接报错：

```ts
for (const candidate of slotCandidates) {
  const precedence = `${candidate.layer}:${candidate.form}`;
  if (seenPrecedence.has(precedence)) {
    throw new Error(`Agent source slot "${slot}" has more than one ${candidate.layer} ${candidate.form} candidate.`);
  }
  seenPrecedence.add(precedence);
}
```

这是 discovery 阶段那个 `slot-collision` 之外的**第二道防线**。discover 抓的是同一目录下的同名文件，这里抓的是跨层同优先级冲突。

选完赢家，输家不会被丢掉，而是记一条 `shadowed`：

```ts
const winner = ordered[0]!;
selected.set(slot, winner);

for (const loser of ordered.slice(1)) {
  entries.push({
    kind: "shadowed",
    source: describeAgentSourceCandidate(loser),
    winnerSourceId: winner.sourceId,
  });
}
```

所以「我覆盖了默认 bash，怎么确认」的答案是：在 `compiled-agent-manifest.json` 里搜 `shadowed`，看被换掉的是哪个 sourceId。

## 22.4 每个节点怎么编译

一个 agent 加上它的所有子代理，构成一棵树。编译是按节点递归的：

```ts
// packages/eve/src/compiler/normalize-manifest.ts:172
async compileStaticNode(input: NodeCompileInput): Promise<CompiledLocalNodeResult> {
  const phaseOne = await this.createPhaseOneNodeSourceState(input, input.inheritedExternalDependencies);
  let config = await compileAgentConfig(input.manifest, this.context, {
    binding: phaseOne.selectedConfig.binding,
    definition: phaseOne.selectedConfig.definition,
    source: phaseOne.selectedConfig.source,
  });
  assertRootOnlyConfig(config, input.isRoot, input.manifest.agentId);
  applyAgentToolPolicy(phaseOne, config);
  applyDefaultToolPolicy(phaseOne, config);

  const externalDependencies = mergeExternalDependencies(...);
  const state = finalizeNodeSourceState(phaseOne, externalDependencies);
  markConfigRuntimeEntries(config, state.evaluation);
  const resources = await this.compileResources(input, state);
  const children = await this.compileChildren(input, state, externalDependencies);
  const manifest = createCompiledAgentNodeManifest({ ...resources, config, remoteAgents: children.remoteAgents });
  return { descendants: children.nodes, manifest };
}
```

七个动作，顺序不能换：

1. **phase one**：收集这个节点的全部候选（作者的 + 框架默认的 + 扩展的）。这时还没求值任何模块。
2. **`compileAgentConfig`**：求值 `agent.ts`，拿到 `defineAgent` 的返回值。这是唯一在编译期执行作者代码的地方。
3. **`assertRootOnlyConfig`**：检查只有根节点能用的配置（比如 `channels`），子代理写了就报错。
4. **两个 policy**：`tool: false` 时移除 `tools/agent`；`defaultTools: false` 时移除所有框架默认工具，但保留被作者覆盖过的 slot。
5. **`finalizeNodeSourceState`**：按优先级选赢家，产出 `selected` 和 `shadowed`。
6. **`compileResources`**：逐类能力归一化（tool / channel / schedule / skill / subagent / memory / sandbox）。
7. **`compileChildren`**：递归子代理，每个子代理一个 node id。

第 6 步里每个 `normalize-*.ts` 负责一类能力。它们的共同职责：把作者写的定义（可能是 `defineTool` 返回值，也可能是 markdown）转成产物里的稳定结构，并在形状不对时给出可读的报错。

举个例子，`normalize-tool.ts` 会把 `defineTool` 的结果拆成「模型看的 schema」和「运行时用的 handler 引用」两部分。前者进 manifest，后者进 module map 的绑定表。这就是为什么 `eve info --json` 里 `toolInputSchemas` 和你在 `defineTool` 里写的不完全一样——中间经过了这一步转换。

## 22.5 module map：两种加载方式

`module-map.mjs` 是编译产物里最特殊的一个：它不是 JSON，是可执行的 ESM。

```ts
// packages/eve/src/compiler/module-map.ts:60
export function createCompiledModuleMapSource(input: CreateCompiledModuleMapSourceInput): string {
  const importSpecifierStyle = input.importSpecifierStyle ?? "relative";
  let index = 0;
  let usesProgrammaticLoader = false;
  const scopes: RenderedNodeScope[] = collectBoundNodeScopes(input.manifest).map((scope) => {
    // ... 每个 binding 生成一个 module_N
  });
```

生成规则只有两条分支：

```ts
if (binding.backing.kind === "filesystem") {
  const specifier = /* 相对路径 */;
  // → import * as imported_module_N from "../../agent/tools/read_logs.ts"
} else {
  // backing.kind === "programmatic"
  usesProgrammaticLoader = true;
  // → const module_N = await loadProgrammaticModule({ kind: "programmatic", ... })
}
```

真实产物长这样：

```js
// .eve/compile/module-map.mjs
import { loadFrameworkProgrammaticModule as loadProgrammaticModule, memoizeModuleNamespaceFactories } from "eve/internal/programmatic-source-loader";
import * as imported_module_0 from "../../agent/channels/eve.ts";

const module_0 = memoizeModuleNamespaceFactories(imported_module_0);
const module_1 = await loadProgrammaticModule({"kind":"programmatic","moduleId":"sandbox.ts","registryId":"eve:defaults","revision":"eve@0.81.1:compiled-manifest-v53","semanticRevision":"eve:default-sandbox:v1"}, Object.freeze({}));
const module_2 = await loadProgrammaticModule({"kind":"programmatic","moduleId":"tools/bash.ts","registryId":"eve:defaults","revision":"eve@0.81.1:compiled-manifest-v53"}, Object.freeze({}));

export const moduleMap = Object.freeze({
  "nodes": Object.freeze({
    "__root__": Object.freeze({
      "modules": Object.freeze({
        "channels/eve.ts": module_0,
        "eve:defaults:sandbox.ts": module_1,
        "eve:defaults:tools/bash.ts": module_2,
      })
    })
  })
});
```

两种加载方式的理由：

- **你的代码走静态 import。** 打包器能分析、能 tree-shake、能给你精确的报错行号。
- **框架默认能力走 `loadProgrammaticModule`。** 它们不在项目目录里，没有稳定的相对路径可写。loader 按 `registryId` + `moduleId` 从 eve 包里取。

注意 `imported_module_0` 指向 `../../agent/channels/eve.ts`——**带 `.ts` 后缀**。这是 Node 的类型剥离在起作用，也是为什么 `engines.node` 要求 `>=24`。

`memoizeModuleNamespaceFactories` 包了一层，但它做的事情比名字听起来窄：

```ts
// packages/eve/src/compiler/source-graph.ts:14
export function memoizeModuleNamespaceFactories(namespace): ProgrammaticModuleNamespace {
  return Object.freeze(
    Object.fromEntries(
      Object.entries(namespace).map(([exportName, exportValue]) => {
        if (typeof exportValue !== "function") return [exportName, exportValue];
        let invocation: Promise<unknown> | undefined;
        const memoized = new Proxy(exportValue, {
          apply(target, thisArgument, argumentsList) {
            if (argumentsList.length > 0) {
              return Reflect.apply(target, thisArgument, argumentsList);
            }
            invocation ??= Promise.resolve().then(() =>
              Reflect.apply(target, thisArgument, argumentsList),
            );
            return invocation;
          },
        });
        return [exportName, memoized];
      }),
    ),
  );
}
```

它缓存的是**无参工厂函数的调用结果**，不是模块求值。模块只执行一次是 ESM 的天然行为，跟这个函数无关。

这个区别重要：`defineTool({...})` 在模块顶层调用一次，返回的对象被所有 session 共享。工具 handler 里的闭包状态是共享的，不是每个 session 一份。想按 session 隔离，状态得放进 `ctx`（第 8 章，第 23 章会再讲一次）。

## 22.6 compile-metadata：hash 决定重建

```ts
// packages/eve/src/compiler/artifacts.ts:170
export function createCompileMetadata(input): CompileMetadata {
  const generator = resolveInstalledPackageInfo();
  const compiledManifestHash = createContentHash(input.compiledManifestJson);
  const discoveryManifestHash = createContentHash(input.discoveryManifestJson);
  const diagnosticsHash = createContentHash(input.diagnosticsArtifactJson);
  const moduleMapHash = createContentHash(input.moduleMapSource);

  return {
    compile: {
      manifest: { path: ..., sha256: compiledManifestHash },
      moduleMap: { path: ..., sha256: moduleMapHash },
    },
    discovery: {
      diagnostics: { path: ..., sha256: diagnosticsHash },
      manifest: { path: ..., sha256: discoveryManifestHash },
      sourceGraphHash: createContentHash(
        `${compiledManifestHash}:${discoveryManifestHash}:${diagnosticsHash}:${moduleMapHash}`,
      ),
      summary: input.diagnosticsSummary,
    },
    generator: { name: generator.name, version: generator.version },
    kind: "eve-compile-metadata",
    status: input.diagnosticsSummary.errors > 0 ? "failed" : "ready",
    version: 6,
  };
}
```

真实文件：

```json
{
  "compile": {
    "manifest": { "path": ".eve/compile/compiled-agent-manifest.json", "sha256": "b42dbf2a..." },
    "moduleMap": { "path": ".eve/compile/module-map.mjs", "sha256": "6c5eae09..." }
  },
  "discovery": {
    "diagnostics": { "path": ".eve/discovery/diagnostics.json", "sha256": "da80f365..." },
    "manifest": { "path": ".eve/discovery/agent-discovery-manifest.json", "sha256": "2aba6069..." },
    "sourceGraphHash": "fba3817b...",
    "summary": { "errors": 0, "warnings": 0 }
  },
  "generator": { "name": "eve", "version": "0.81.1" },
  "kind": "eve-compile-metadata",
  "status": "ready",
  "version": 6
}
```

三个用途：

1. **`sourceGraphHash` 是重建判断依据。** 四个 hash 拼起来再 hash 一次。任何一个产物变了，这个值就变。
2. **`status` 记录这次编译是否成功。** `failed` 时产物仍然写盘了——这是故意的，让你能在编译失败后检查产物。
3. **`generator.version` 记录编译用的 eve 版本。** 升级 eve 后即使源码没变，这个字段也会变，触发重建。

一个容易误解的点：hash 算的是**序列化后的 JSON 文本**，不是语义。改一个字段的顺序、加一个空格，hash 就变。所以不要手工编辑 `.eve/compile/*.json`——下次编译会被覆盖，而且中间这段时间的 hash 是对不上的。

## 22.7 dev 下的快照：为什么改代码不影响运行中的会话

生产环境只有一份产物。dev 环境每次重建都生成一个完整快照：

```ts
// packages/eve/src/internal/nitro/dev-runtime-artifacts.ts:82
export async function stageDevelopmentRuntimeArtifactsSnapshot(
  compileResult: CompileAgentResult,
): Promise<DevelopmentRuntimeArtifactsSnapshot> {
  const snapshotRoot = join(
    resolveDevelopmentRuntimeArtifactsSnapshotsDirectory(compileResult.project.appRoot),
    `${Date.now().toString(36)}-${randomUUID()}`,
  );
  const sourceSnapshotPlan = await createDevelopmentSourceSnapshotPlan({ ... });

  await copyDevelopmentSourceSnapshot(sourceSnapshotPlan);
  await mountFrameworkPackage(sourceSnapshotPlan.runtimeAppRoot);
  await mountExtensionExternalDependencies({ ... });
  await cp(compileResult.paths.compileDirectoryPath, join(sourceSnapshotPlan.runtimeAppRoot, ".eve", "compile"), { recursive: true });
  await rewriteSnapshotCompiledManifest({ ... });
  await validateSnapshotCompiledManifestRoots({ ... });
  // ...
}
```

一次快照包含：

```text
.eve/dev-runtime/
├── current.json                                    ← 指针，指向当前活跃快照
└── snapshots/
    └── mvgazeec-d0df0027-7e5a-49d6-b772-5bf1b5f4b1a3/
        ├── source/                                 ← 你的源码副本
        │   ├── agent/
        │   ├── node_modules/eve -> ...             ← 软链到框架包
        │   └── .eve/compile/                       ← 编译产物副本
        └── ...generation metadata
```

`current.json`：

```json
{
  "appRoot": "/path/to/app",
  "kind": "eve-dev-runtime-artifacts-pointer",
  "runtimeAppRoot": "/path/to/app/.eve/dev-runtime/snapshots/mvgb5tkl-.../source",
  "snapshotRoot": "/path/to/app/.eve/dev-runtime/snapshots/mvgb5tkl-...",
  "version": 2
}
```

为什么这么做：**运行中的 session 需要一份不会被下一次编辑改掉的源码。**

你在 TUI 里聊到一半，改了一行工具代码，这个 session 的 module map 仍然指向旧快照。新开的 session 才用新快照。这就是本章开头第二个现象的答案。

代价是磁盘。一个快照约 960 KB（不含 `node_modules` 软链）。`pruneDevelopmentRuntimeArtifactsSnapshots` 按 `retainCount` 和 `gracePeriodMs` 清理，`current.json` 指向的那个永远保护。

`.eve/dev-runtime/snapshots/` 堆到几十个，说明 prune 没跑起来——通常是 dev server 被 `kill -9`，清理逻辑没机会执行。

## 22.8 回到开头那两个现象

**写 `tools/bash.ts` 会不会冲突？** 不会。`application` 层（3）高于 `framework-default` 层（0），你的实现直接覆盖内置的。被换掉的那个记作 `shadowed`，在 `compiled-agent-manifest.json` 里能查到。想确认覆盖生效了，搜这个字段。

**为什么改了代码老会话不变？** dev 下每次重建生成新快照，老会话绑定的是旧快照。新会话用新快照。想看新代码，开个新会话，或者看 `current.json` 指向的 snapshot 时间戳。

## 22.9 五个注意事项

1. **`compiled-agent-manifest.json` 不是给人读的。** 它包含完整的内联 schema、每个工具的输出整形配置、每个通道的路由表。看结构可以，别指望通读。
2. **改 `agent.ts` 会触发全量重编译，改 `instructions.md` 也会。** 因为 markdown 内容被内联进 discovery manifest，hash 变了。
3. **`shadowed` 条目只在有覆盖时出现。** 没覆盖就没有这个字段，不是空数组。
4. **dev 快照会让「改了但没生效」变得难判断。** 老 session 用老快照，行为不变是正常的。判断是否生效要看新 session，或者看 `current.json` 指向的 snapshot 时间戳。
5. **`compile-metadata.json` 的 `status: "failed"` 时产物仍然存在。** 别看到文件在就以为编译成功了。

## 22.10 练习

1. 在 `agent/tools/` 下写一个 `bash.ts`，导出任意 `defineTool`。跑 `eve build`，然后在 `compiled-agent-manifest.json` 里搜 `shadowed`，找到被覆盖的 `eve:defaults:tools/bash.ts`。
2. 跑一次 `eve dev`，记下 `current.json` 的 `snapshotRoot`。改一行 `agent/instructions.md`，等 HMR 完成后再看 `current.json`，确认指针变了、旧快照还在。
3. 打开 `module-map.mjs`，数一下有几个 `import * as`，几个 `loadProgrammaticModule`。对照 `agent/` 下的文件数解释这个比例。
4. 删掉 `.eve/compile/compile-metadata.json`，跑 `eve info`。看它是否重新生成，以及 `sourceGraphHash` 是否和之前一样。
5. 在 `agent/agent.ts` 里加 `defaultTools: false`，重新编译，看 `compiled-agent-manifest.json` 里还剩几个工具。

## 下一章

产物有了：一份 JSON 描述「有什么」，一个 ESM 模块提供「怎么加载」。

但 JSON 里没有函数。工具的执行逻辑、通道的 handler、指令的渲染函数，都还没挂上去。

第 23 章讲 runtime 怎么把这些声明变成活的 agent 对象，以及为什么工具里的模块级变量会在所有会话之间共享。

---

返回：[教程目录](README.md) · 上一章：[21 discover](21-discover.md) · 下一章：[23 runtime](23-runtime.md)
