# 第 3 章 agent 目录解剖

> 路径决定名字，文件名决定能力。这一章把 `agent/` 下每个槽位的语义、命名规则、覆盖机制讲清楚。

## 3.1 全貌

```text
agent/
├── agent.ts                 # 运行时配置：模型、推理强度、压缩、限制
├── instructions.md          # 系统提示（或 instructions.ts / instructions/）
├── instrumentation/         # 遥测 provider 与出口
├── channels/                # HTTP 与消息平台入口
├── connections/             # 外部 MCP / OpenAPI 服务
├── extensions/              # 挂载的复用能力包
├── hooks/                   # 事件订阅
├── skills/                  # 按需加载的流程
├── lib/                     # 仅供 agent 代码 import 的辅助模块
├── memory/                  # 跨会话记忆槽
├── sandbox/                 # 沙箱定义 + workspace 种子
├── tools/                   # 类型化动作
├── schedules/               # cron 任务
└── subagents/               # 专家子代理
```

只写你需要的。框架默认值占同一批槽位，所以**同路径的文件会替换默认实现**。

## 3.2 命名规则

| 路径 | 解析成 |
| --- | --- |
| `agent/tools/read_logs.ts` | 工具 `read_logs` |
| `agent/connections/linear.ts` | 连接 `linear` |
| `agent/skills/triage.md` | 技能 `triage` |
| `agent/skills/runbook/SKILL.md` | 技能 `runbook` |
| `agent/subagents/log-analyst/agent.ts` | 子代理 `log-analyst` |
| `agent/schedules/daily/summary.ts` | 定时任务 `daily/summary` |
| `agent/hooks/audit.ts` | 钩子 `audit` |

注意哪些槽位支持子目录：`schedules/` 和 `hooks/` 支持递归嵌套（名字带路径），`skills/` 支持一个目录当一个技能，`subagents/` 每个目录是一个 agent。**`tools/` 和 `connections/` 是平铺的**——工具名就是文件名，不要指望 `tools/ops/read_logs.ts` 能注册成 `ops/read_logs`；要在工具名里分组就用下划线（`ops_read_logs`）。

命名限制：

- 工具名和技能名只用 ASCII 字母、数字、下划线、短横线，以字母或数字开头，最长 64 字符。
- `eve` 是保留命名空间。任何以 `eve__` 开头的名字都不能用（`eve__search`、`eve__tool`、`eve__skill`、`eve__task_wait` 等由框架占用）。
- 连接名会占据 `<name>__*` 整个前缀。有个叫 `linear` 的连接，就不能再有叫 `linear__foo` 的工具。
- 子代理名 `agent` 保留给内置的根副本工具。要声明式做，目录换个名字。

## 3.3 每个槽位的语义

### `agent.ts` —— 运行时配置

```ts
// agent/agent.ts
import { defineAgent } from "eve";

export default defineAgent({
  model: "anthropic/claude-sonnet-5",
  reasoning: "high",
  compaction: { thresholdPercent: 0.75 },
  limits: { maxInputTokensPerSession: 200_000 },
  build: { externalDependencies: ["sql.js"] },
});
```

不写这个文件，框架用默认模型（`openai/gpt-6-luna-fast`，`reasoning: "high"`）。一旦写了，`model` 就是必填。

字段的完整释义在第 2 章；`compaction` / `limits` 的子字段在第 6 章，`experimental.workflow` 在第 10 章。

### `instructions.md` / `instructions.ts` / `instructions/`

三种形态，优先级：根目录的 `instructions.md` 或 `instructions.ts`（二选一，同时存在是编译错误）在前，然后按文件名 `localeCompare` 排序拼接 `instructions/` 目录里的条目。

什么时候升级成 TypeScript：

- 需要拼装（从 `lib/` 取数据、按环境分支）
- 需要 `role: "user"`（把内容写进对话历史，而不是系统提示）
- 需要 import 一个 `.md` 文件用 `?raw`

```ts
// agent/instructions.ts
import { defineInstructions } from "eve/instructions";
import { buildSystemPrompt } from "./lib/prompts";

export default defineInstructions({ content: buildSystemPrompt() });
```

### `tools/` —— 类型化动作

文件名即工具名。详见第 4 章。

### `sandbox/` —— 沙箱

两种写法：`agent/sandbox.ts` 单文件，或 `agent/sandbox/sandbox.ts` 加 `agent/sandbox/workspace/` 种子目录。后者更常用，因为要带参考文件。

```text
agent/sandbox/
├── sandbox.ts
└── workspace/
    ├── schema.sql          → /workspace/schema.sql
    └── notes/grain.md      → /workspace/notes/grain.md
```

### `skills/` —— 按需流程

详见第 7 章。

### `memory/` —— 跨会话记忆

`agent/memory.ts`（单槽，名字是 `memory`）或 `agent/memory/<slot>.ts`（多槽）。两者互斥。详见第 8 章。

### `subagents/` —— 专家

`agent/subagents/<name>/` 是一个完整的 agent 根目录，用自己的 `agent.ts`、`instructions.md`、`tools/`、`sandbox/`。**不继承**父级任何手写能力。详见第 12 章。

### `channels/` —— 入口

`channels/eve.ts` 是内置 HTTP API 的配置（鉴权、CORS、上传策略）。`channels/slack.ts`、`channels/discord.ts` 等是平台通道。详见第 14 章。

### `connections/` —— 外部服务

MCP 或 OpenAPI。模型通过 `eve__search` / `eve__tool` 访问，不直接看到 URL 和 token。详见第 13 章。

### `schedules/` —— 定时

`.ts`（`defineSchedule`）或带 `cron` frontmatter 的 `.md`。详见第 15 章。

### `hooks/` —— 事件订阅

只观察，不能往上下文里注入内容。要注入内容用 `defineDynamic` + `defineInstructions`。详见第 15 章。

### `lib/` —— 辅助代码

只能被 agent 代码 `import`，**不会**被复制进沙箱。想让模型在沙箱里读到文件，放 `agent/sandbox/workspace/`。

这是新手最常见的混淆点：

| 我想…… | 放哪 |
| --- | --- |
| 让工具 `execute` 里 import 一个函数 | `agent/lib/` |
| 让模型用 `read_file` 读到一份规范文档 | `agent/sandbox/workspace/` |
| 让模型按一份长流程做事 | `agent/skills/` |

### `instrumentation/` —— 遥测

每个文件一个 provider，按路径命名。第 18 章简单提。

## 3.4 覆盖与禁用默认工具

框架的默认工具占用同一批槽位。三种处理方式：

```ts
// 1. 完全禁用
// agent/tools/bash.ts
import { disableTool } from "eve/tools";
export default disableTool();
```

```ts
// 2. 改写描述或包一层
// agent/tools/bash.ts
import { defineTool } from "eve/tools";
import { bash } from "eve/tools/bash";

export default defineTool({
  ...bash,
  description: "运行已批准的项目维护命令。",
  async execute(input, ctx) {
    console.info("sandbox command", input.command);
    return bash.execute(input, ctx);
  },
});
```

```ts
// 3. 全局关掉所有可选默认工具
// agent/agent.ts
export default defineAgent({
  model: "anthropic/claude-sonnet-5",
  defaultTools: false,
});
```

`defaultTools: false` 关掉的是**可选**默认工具（`bash`、`read_file`、`write_file`、`web_fetch`、`web_search`、`agent` 等）。你自己写的文件不受影响，`eve__*` 目录工具也不受影响——它们按会话能触达的东西自动挂载。

生产环境的建议：不用 `defaultTools: false` 一刀切，而是逐个审查。`bash` 和 `write_file` 给了模型在沙箱里任意读写的能力，如果你的 agent 根本不需要，直接 `disableTool()`。

## 3.5 单 agent、多 agent、独立包

| 需求 | 结构 |
| --- | --- |
| 一个 agent，可能带前端 | 根目录 `agent/`，前端放 `app/` |
| 一个 agent 带专家 | `agent/subagents/<name>/` |
| 多个可独立寻址的 agent，共享部署 | `agents/<name>/agent/`（eve agent workspace） |
| 独立依赖、独立发版 | 拆成独立包或独立项目 |

workspace 形态：

```text
operations/
├── package.json
├── agents/
│   ├── support/
│   │   ├── agent/
│   │   └── evals/
│   └── research/
│       ├── agent/
│       └── evals/
└── apps/web/
```

```bash
npx eve@latest init operations --agents support,research
npx eve dev --agent support
npx eve info --agent research
```

关键约束：成员目录里**不能有 `package.json`**，有就被排除在发现之外。根目录有 `agent/` 时项目按单 agent 处理，即使同时存在 `agents/`。

## 3.6 构建产物

`eve dev` 和 `eve build` 会在 `.eve/` 下写东西。真实结构（`eve@0.79.0` 上跑过一次抓的；这些目录名在 0.81 的源码里仍然存在）：

```text
.eve/
├── .workflow-data/          # 本地 workflow 状态，注意目录名带前导点
│   ├── version.txt          # 写它的 world 版本，例如 @workflow/world-local@5.1.0
│   ├── steps/               # 每个 step 的检查点：输入、输出、attempt、状态
│   ├── runs/                # run 元数据：部署 ID、spec 版本、执行上下文
│   ├── events/  streams/    # 事件流与游标
│   ├── hooks/  waits/       # 停泊中的等待与回调
│   └── .locks/
├── dev-runtime/snapshots/   # 本地开发的代码代次（进行中的 turn 钉在某一代）
├── dev-hosts/<uuid>/        # 每次 dev server 的编译产物、artifacts、nitro
├── sandbox-cache/           # 沙箱模板缓存
├── discovery/  compile/     # 发现与编译的诊断产物
└── traces/                  # 本地 trace
```

两个运维含义：

- **删掉 `.eve/.workflow-data` 会丢掉进行中的会话上下文**，包括多轮消息历史。你自己的外部存储（Redis、Postgres）里的数据不受影响。
- 部署到 Vercel 时这份状态由 Vercel Workflow 承载；自托管时要持久化这个目录或换外部 world。

## 3.7 被忽略的文件

自动发现会跳过：

- `*.test.ts` / `*.spec.ts`
- `__tests__/` 目录

所以你可以把工具的单测放在旁边：

```text
agent/tools/
├── read_logs.ts            # 注册为 read_logs
├── read_logs.test.ts       # 忽略
└── __tests__/
    └── fixtures.json       # 忽略
```

显式 `import`、沙箱 workspace 文件、打包技能的附属文件不受这条规则影响。

## 3.8 常见注意事项

**注意：在 `agent/` 根目录同时放了 `instructions.md` 和 `instructions.ts`。**
编译错误。二选一。

**注意：把技能写成了工具期望的行为。**
技能只加文本，不加执行能力。技能里写「调用 `read_logs` 工具」是可以的（工具本来就在），写「运行这段 Python」不行——除非有 `bash` 工具且沙箱可用。

**注意：`lib/` 里的文件在沙箱里找不到。**
设计如此。要共享给模型看，走 workspace 种子。

**注意：workspace 成员目录里放了 `package.json`。**
`eve info` 里看不到这个 agent，且没有明显报错。

**注意：改了目录名以为只是重命名。**
工具名、技能名、评测断言、审批策略里引用的名字全都跟着变。改名是 breaking change。

## 3.9 练习

1. 给 opsbot 加一个 `agent/lib/` 模块（比如格式化时间），确认工具能 import 它。
2. 把一个参考文件放进 `agent/sandbox/workspace/`，重启 dev server，让模型 `read_file` 读它。
3. 跑 `npx eve info`，对照输出检查你建的每个槽位是否被正确发现。
4. 用 `disableTool()` 关掉 `web_search`，确认模型不再声称能搜网页。

## 延伸阅读

- [Agent Files reference](https://eve.dev/docs/reference/agent-files)
- [Project Structure](https://eve.dev/docs/concepts/project-structure)
- [Built-in Tools](https://eve.dev/docs/concepts/built-in-tools)

---

上一章：[02 第一个 agent](02-hello-agent.md) · 下一章：[04 工具](04-tools.md)
