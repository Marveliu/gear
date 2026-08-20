# Hitch ↔ DSH 对接改动

- 状态：Draft v0.3
- 目的：落实 [DSH 自进化 harness spec](dsh-self-evolving-harness-spec.md) §7 的四件交付物——`dsh-evolving` adapter、DSH stdout NDJSON 事件输出模式、V1 本地对照评测与 verifier 记录闭环、executable candidate 的 Harbor 路线。本文件是这些改动的具体设计；spec 只保留边界和结论。
- 已核查代码基线：agent-hitch `dev@c42ef93`、DeepSeek Harness `master@99f6f02fec`（2026-08-20）。实现前必须重新核对并把实际依赖 commit 写入 PR；不得把未提交工作区能力当作现有接口。
- 更新：2026-08-19 — 初版
- 更新：2026-08-19 — v0.2：记录当时工作区中的锁定解析、Harbor bootstrap、内存限制与 eval 判据设想
- 更新：2026-08-20 — v0.3：按已提交源码重新核查并修正启动协议、现有 DeepSeek adapter、`DSH_HOME`、NDJSON wire schema、事件字段、verifier 权威记录与 Harbor 依赖状态

## 1. 为什么仍需要 `dsh-evolving`

Hitch 的 adapter 注册表硬编码在 `src/adapters.js` 的模块级 `definitions` 对象里，没有插件目录、配置注册面或 CLI 动态注册参数。新增 harness id 仍然必须是 agent-hitch 的源码提交。

当前 Hitch **已经存在** `deepseek` adapter；它把 `@deepseek-ai/dsh` npm 包或 `deepseek-harness` 源码 commit 当作 revision，使用 headless profile，把最终纯文本翻译成 `message.delta`。它不能直接承载本设计的 revision 语义：自进化系统把 **overlay repo commit** 当作 Harness revision，而每个 overlay commit 又在 manifest 中固定 DSH base revision。对 `deepseek@git+file://<overlay-repo>#<sha>` 使用现有 adapter 时，Hitch 会按 DeepSeek 源码仓库的 recipe 构建 overlay repo，入口和构建命令均不匹配。

因此 V1 使用独立的 `dsh-evolving` definition 是合理的：

- `deepseek`：revision 是 DSH 本体；
- `dsh-evolving`：revision 是完整 Harness overlay，DSH 本体是其被钉死的依赖。

不采用协议冒充。把包装脚本挂到其他 adapter 的 `path_env` 只能为 installed executable 建指纹，不能让该 adapter 的 Git recipe 正确构建 overlay，也不能给 overlay commit 建立准确的 revision identity。

## 2. 改动一：`dsh-evolving` adapter（agent-hitch 源码提交）

定义骨架：

```js
"dsh-evolving": {
  id: "dsh-evolving",
  display_name: "DSH Evolving Harness",
  command: "dsh-evolving",
  path_env: "HITCH_DSH_EVOLVING_PATH",
  version_args: ["--version"],
  revision_sources: {
    commit: {
      type: "git",
      url: "<registered harness overlay repo URL>",
      commands: [ /* 见 §2.1 */ ],
      entrypoint: "<generated launcher path>",
    },
  },
  capabilities: {
    non_interactive: true,
    streaming: true,
    structured_messages: true,
    structured_tool_events: true,
    sessions: true,
    resume: false,
    model_selection: false,
    graceful_cancel: false,
  },
  process(request, executable, runtime) { /* 见 §2.2 */ },
  translate(record, state) { /* 见 §3.2 */ },
}
```

`model_selection: false` 表示模型由 overlay 所固定的 DSH settings 决定。由于当前 Hitch 尚未统一强制 capability，adapter 的 `process()` 仍须在收到非空 `request.model` 时以 `capability_unsupported` 失败，不能静默覆盖固定模型。

`graceful_cancel: false` 表示 DSH headless 没有应用层取消/flush 协议；Hitch 仍可对进程组先发 SIGTERM、超时后 SIGKILL，但不得把这描述成保证 session 尾部落盘的优雅取消。

### 2.1 构建产物与 launcher 合同

Hitch 对 `commit:` 和显式 `git+file` commit 的 prepare 流程是：检出 exact commit、执行 definition 中的固定 `commands`、验证 `entrypoint`、计算 artifact integrity 并原子提升缓存。

构建命令必须：

1. 校验 `harness/manifest.json` 的 schema、canonical digest 和 artifact 清单；
2. 按 manifest 中的 exact `dshRevision` 安装/检出 DSH，并验证 lockfile/integrity；禁止 branch、tag、range 和 `latest`；
3. 把该 commit 的完整 overlay 物化为只读运行目录；
4. 生成一个 launcher。launcher 是可信构建基础设施，不属于自动 mutation 的路径白名单。

launcher 的语义等价于：

```sh
#!/bin/sh
set -eu

if [ "${1-}" = "--version" ]; then
  exec <pinned-dsh> --version
fi

# DSH launcher flags 必须出现在第一个 headless app flag 之前。
exec <pinned-dsh> \
  --profile headless \
  --patch "<artifact>/harness.cordis.yml" \
  --events jsonl \
  "$@"
```

顺序是协议的一部分：DSH 顶层 launcher 在遇到第一个不认识的 app flag 后停止解析顶层参数，因此 `--patch` 必须位于 headless 自有的 `--events` 之前。

resolution identity 由 overlay source URL + full commit 决定；由于 exact DSH revision 和 overlay 内容都进入该 commit，它们被传递性钉死。prepared artifact identity 还应覆盖 dependency lock、toolchain 和最终 artifact integrity。相同 resolution identity 不代表跨平台 artifact 字节相同；artifact cache 仍按 platform/architecture/toolchain 区分。

### 2.2 `process()`

artifact launcher 已经负责 `--profile`、`--patch` 和 `--events`，adapter 不得重复添加这些参数。当前 DSH headless 的 task 合同是**位置参数**，不是 stdin：

```js
process(request, executable, runtime = {}) {
  if (request.model) {
    throw new HitchError("dsh-evolving fixes the rollout model in its overlay", {
      code: "capability_unsupported",
      exitCode: 10,
    });
  }
  assertAllowedHeadlessArgs(request.agent_args);
  return {
    executable,
    args: [...request.agent_args, request.prompt],
    input: "",
    env: runtime.runtime_home ? { DSH_HOME: runtime.runtime_home } : {},
  };
}
```

`agent_args` 只允许明确列入白名单的 headless app 参数；必须拒绝 `--profile`、`--patch`、`--events`、模型/provider、credential 和权限相关参数，避免绕过 artifact 固定配置。

每个 Hitch run 使用独立的 `runtime_home`/`DSH_HOME`。这避免用户级 settings、credential 文件、profile 和 session store 污染评测，也是并发运行时磁盘记录天然按 run 隔离的关联边界。

workspace、timeout、取消和进程树由 Hitch 引擎管理；adapter 只负责命令、环境和事件翻译。

### 2.3 resolution 锁定：当前 V1 不依赖未提交 CLI

已核查的 agent-hitch 基线没有 `hitch prepare/run --resolved-revision-file`。因此 V1 不把它写成前置能力：

1. RefineService 对 champion 和 candidate 分别使用 full 40-hex commit ref 执行 `hitch resolve --json`；
2. 把两份 resolution JSON 和各自 expected identity 记录进 round；champion 和 candidate 应当是**不同 identity**；
3. 每个后续 run 仍由当前 Hitch CLI resolve，但 RefineService 必须比较 run 的 `revision_identity` 与对应 expected identity；不一致则整轮失败；
4. champion 未变化时可复用此前 baseline run refs 和 expected identity，不需要重新运行 baseline。

未来若 `--resolved-revision-file` 以已合入 commit 交付，可作为优化，减少容器内/外重复 resolve；它不能代替 full commit、不变 source 和 identity 校验，也不能把 baseline 与 candidate 锁成同一个 identity。

## 3. 改动二：DSH stdout NDJSON 事件模式（DSH 侧前置 PR）

### 3.1 CLI 与 wire schema

headless app 新增内层参数 `--events jsonl`。成功运行时，stdout 成为严格 framing channel：每一行必须是一个完整 JSON object，禁止最终纯文本和其他日志写入 stdout；诊断与人类可读错误写 stderr。

V1 wire schema：

```ts
type DshHeadlessRecord =
  | {
      schema_version: 1
      kind: 'session'
      session_id: string
    }
  | {
      schema_version: 1
      kind: 'event'
      session_id: string
      event: SessionEvent
    }
```

要求：

- 使用 DSH 已有 `session/created` 与 scoped `session/event` 生命周期，在第一个该 session 的 event 之前输出 `kind: 'session'`；
- 每个 event record 都带 `session_id`，adapter 不依赖进程内隐式状态猜关联；
- 输出本次 fresh headless session 的全部 live `SessionEvent`，包括未被 Hitch 归一化词表理解的类型；
- 保留 event 原始 `seq`、`time`、`type`、`data` 和 surface metadata；
- `JSON.stringify(record) + '\n'` 保证内容中的换行被转义为单行；V1 不截断或另造 spill schema；
- 普通非 `--events jsonl` 模式继续保持现有“最终文本 stdout”兼容行为。

session log 的 persistence 逻辑不改变。stdout NDJSON 是对已接受 session event 的实时外部投影；事件被投影时不承诺 persistence backend 已完成 flush。

### 3.2 `translate()` 的确定映射

| DSH record/event | Hitch 归一化事件 | 精确字段与语义 |
| --- | --- | --- |
| `kind: 'session'` | `session.created` | `session_id = record.session_id` |
| `assistant/message` 且 text blocks 拼接非空 | `message.completed` | `text = data.message.content` 中所有 `type === 'text'` 的 block 按序拼接；空文本不得覆盖此前最终文本 |
| `assistant/message` 且有 usage | `usage.updated` | `usage = data.usage` |
| `tool/call` | `tool.started` | `call_id = data.callId`，`name = data.name`，`input = data.arguments`；V1 保留原始 JSON 字符串，不静默 parse/修复 |
| `tool/result` | `tool.completed` | `call_id = data.message.source.callId`；`status = failed` 当 `data.error` 存在或唯一 tool-result block 的 `isError === true`，否则 `succeeded`；`output` 来自该 block 的 content |
| `turn/end`，`reason.kind === 'error'` | `diagnostic` | `level = error`，message 包含 `reason.error.code/message` |
| 其他事件 | `provider.event` | `provider_type = event.type`，`native = record` |

对已归一化的事件也应在 Hitch event 的 `native` 字段中保留原始 record，便于审计字段损失。Hitch `result.json.output` 必须与 DSH 现有 `summarize()` 一致：最后一个**非空** assistant text 是最终输出。实现应带多 step、tool-call-only assistant message、空 text block 和 error turn 的单测。

### 3.3 为什么仍优先 stdout，而不是事后读取 session log

独立 `DSH_HOME` 已使“哪份磁盘 session 属于哪次 run”可确定，因此不能再把事后解析描述成只能依赖 mtime。该路线仍不作为 V1 主路径，原因是：

1. Hitch run engine 当前只有 stdout/stderr 翻译接口，没有 adapter post-run 磁盘采集阶段；采用磁盘读取需要修改通用引擎或让 wrapper 二次解析 DSH persistence backend；
2. stdout 是实时流，timeout/SIGKILL 前已经投影的事件会被 Hitch 保存；磁盘尾部仍取决于 DSH checkpoint/flush 是否完成；
3. Hitch run record 是 rollout 轨迹的权威；若结构化证据只留在 DSH backend，`events.jsonl` 与 `result.json.output` 会退化；
4. stdout wire schema 与 persistence backend 解耦，不需要为 JSONL/SQLite 分别实现采集器。

事后 session log 可以作为一致性诊断：happy path 测试可比较 stdout 投影与持久化 session 的 `(type, seq)`，但不能成为运行结果的唯一数据源。

## 4. V1 本地对照评测与 verifier 记录

### 4.1 当前 Hitch eval 限制

当前 `hitch eval`：

- 要求 immutable `version:<exact>` 或 `commit:<sha>`；
- 拒绝显式 `git+file` ref；resolve 后还要求 Git source 被 adapter 注册；
- Harbor trial 容器内会重新 prepare/run exact registered ref。

代码只用 `source.registered` 标志表达“registered”，但 registered `file://` URL 通常在容器中不可访问。因此仅删除输入守卫或把本地 bare repo 写进 adapter 都不足以构成 Harbor 闭环，还必须解决容器内 source/artifact 可达性。

### 4.2 V1：Hitch run 负责 rollout，RefineService 负责 verifier

只允许 §5 所定义 safe-declarative candidate 的 V1，对每个 seed task 分别运行：

```text
hitch run \
  --harness dsh-evolving@git+file://<harness-repo>#<full-sha> \
  --workspace-mode worktree \
  --cwd <immutable-seed-workspace> \
  --prompt-file <task-prompt> \
  --output jsonl
```

Hitch 是以下内容的权威：resolution、prepared artifact、workspace snapshot、过程事件、terminal result。Hitch run **不执行 verifier**，`events.jsonl` 也不能单独算出通过率。

RefineService 在成功或失败的 rollout 结束后，对 retained isolated workspace 执行 seed repo 固定的 verifier，并写自己的不可变记录：

```ts
interface EvaluationRecord {
  schemaVersion: 1
  id: EvaluationId
  roundId: RoundId
  role: 'baseline' | 'candidate' | 'held-out-baseline' | 'held-out-candidate'
  harnessRef: HarnessRef
  expectedRevisionIdentity: string
  hitchRunRef: string
  taskRef: string
  seedRevision: string
  workspaceSnapshotDigest: string
  evaluationContextDigest: string
  verifier: {
    digest: string
    argv: string[]
    timeoutMs: number
    exitCode?: number
    status: 'passed' | 'failed' | 'timed-out' | 'not-run'
    stdoutRef?: string
    stderrRef?: string
  }
  createdAt: string
}
```

边界：

- EvaluationRecord、verifier stdout/stderr 和 round lineage 由 RefineService 持久化；不得写入 Hitch 独占的 root；
- `verifier.argv` 是 argv 数组，不是交给 shell 的任意 command string；seed source 必须来自部署配置的可信 registry，并钉 full commit；
- `cwd` 必须是 seed workspace 内经 realpath 校验的相对路径，拒绝绝对路径、`..` 和 symlink escape；
- rollout 未成功时 verifier 默认为 `not-run`，该 task 计失败但保留 Hitch failure evidence；
- baseline/candidate 只有在 `evaluationContextDigest` 完全一致且 role/harness fields 合法时可比较；digest 至少覆盖 seed revision/task order、workspace snapshot、model/provider/sampling、DSH base revision、permission、环境镜像、timeout/token policy、verifier digest 和评分公式；
- V1 score 为 EvaluationRecord 的 `passedTrials / totalTrials`，不是从 Hitch `events.jsonl` 推断。

“safe-declarative candidate 不进 Harbor”只表示 candidate 没有新增可执行 artifact，**不表示没有进程风险**。rollout agent 的工具执行仍必须受固定 DSH sandbox/permission 配置约束；Hitch workspace 不是安全 sandbox。

### 4.3 executable candidate 的 Harbor 路线

修改 hook、tool、plugin row、workflow/verifier code 或其他可执行配置的 candidate 不自动 promotion。启用 Harbor 前必须交付并钉住一种完整路径：

1. 把 candidate 推到 adapter 注册的真实远端，让容器按 exact commit resolve/prepare；或
2. 为 Hitch eval 实现受校验的离线 bundle/bootstrap，把 exact resolution、source/artifact 和对应平台运行时上传进容器；或
3. 扩展 Harbor agent，使其接收宿主已准备 artifact，而不是在容器内按 ref 重新 resolve。

`--resolved-revision-file`、`HITCH_EVAL_BOOTSTRAP_DIR`、`memory_mb` 目前只可列为待合入 Hitch PR，不得作为既有能力。无论 Hitch 自身如何判终态，RefineService 都必须把任一 trial errored/cancelled、reward 缺失或 revision identity 不匹配视为 `decision: 'failed'`，不得当成零分 candidate 或消费部分分数。

## 5. 实施顺序与验收

1. **钉代码基线与 fixtures**
   - 在两仓 PR 中写明 agent-hitch/DSH full commit；
   - 提供 deterministic mock model、固定 tool fixture 和最小 overlay repo；
   - 所有文档源码链接在目标 checkout 中可解析。

2. **DSH `--events jsonl` PR**
   - headless inner CLI 正确解析 `--events jsonl` 与位置 task；
   - stdout 仅含符合 schema 的单行 JSON，stderr 承载诊断；
   - 用 deterministic mock 跑同一 fixture 两次，去除 id/time 后 snapshot 一致；不对真实在线模型要求字节级重复；
   - happy path 比较 stdout `(type, seq)` 与持久化 session；timeout fixture 验证已输出事件仍在 Hitch 侧可消费。

3. **`dsh-evolving` adapter PR**
   - `hitch list --json` 同时出现现有 `deepseek` 和新 `dsh-evolving`；
   - launcher 的 `--version`、顶层 flag 顺序、独立 `DSH_HOME` 和位置 task 有测试；
   - controlled task 的 Hitch `events.jsonl` 含一个 `session.created`、至少一个非空 `message.completed`，并对 fixture 中已知 tool call 产生严格配对的 `tool.started/tool.completed`；不要求任意自然语言 task 必然调用工具；
   - `result.json.output` 等于 DSH summarize 的最后非空 assistant text；
   - run 记录的 revision identity 与 RefineService expected identity 相同。

4. **本地对照评测闭环**
   - baseline/candidate 为不同 harness identity，但 `evaluationContextDigest` 相同；
   - workspace snapshot、DSH base revision、模型配置、permission 和 verifier digest 逐项相同；只允许 overlay digest 不同；
   - EvaluationRecord 能反查 Hitch run、verifier 输出和 task/seed revision；
   - score 由 EvaluationRecord 计算，失败/超时/identity mismatch 不产生可接受 candidate；
   - champion 不变时仅在满足 §6 的复用条件下复用 baseline。

5. **Harbor executable 路线（后续独立里程碑）**
   - 以已合入 Hitch commit 为依赖；
   - container 内无需隐式访问宿主路径即可 prepare exact candidate；
   - baseline/candidate 使用同一 container image/resource limits；
   - 任一 errored/cancelled trial 使 eval 不可比较。

## 6. 对照评测的随机性与 baseline 复用

相同请求参数不等于相同模型输出。`score = passed / total` 是对已观察结果的确定函数，但 rollout 本身可能随机。V1 必须在 seed manifest 中声明评测模式：

- `deterministic`：provider/model/sampling/seed 已验证可重复，允许一次 attempt，并允许 champion 未变时跨轮复用 baseline；
- `stochastic`：baseline/candidate 使用相同 attempts 和配对 task 顺序；promotion 依据预先配置的保守规则（至少报告样本数、均值/通过率和差异），不得复用旧 baseline 与新 candidate 直接比较。

held-out 也遵守同一规则。任何 seed revision、held-out revision/划分、evaluation context 或评分策略变化都会使 baseline cache key 变化并强制重跑。

## 7. 参考

- [DSH 自进化 harness spec §7](dsh-self-evolving-harness-spec.md)
- [Hitch adapters（含现有 DeepSeek adapter）](../../agent-hitch/src/adapters.js)
- [Hitch run engine（stdout 消费模型）](../../agent-hitch/src/engine.js)
- [Hitch harness-reference（git+file 显式源）](../../agent-hitch/src/harness-reference.js)
- [Hitch evals（当前本地源守卫与成功判据）](../../agent-hitch/src/evals.js)
- [Hitch Harbor agent（当前容器内重新 prepare/run）](../../agent-hitch/integrations/harbor/hitch_harbor_agent.py)
- [DSH headless runner（现状最终文本输出）](../../agentfw/deepseek-harness/packages/bundle/headless/src/index.ts)
- [DSH headless startup（现状位置 task）](../../agentfw/deepseek-harness/packages/bundle/headless/src/startup.ts)
- [DSH CLI launcher 参数分层](../../agentfw/deepseek-harness/apps/cli/src/args.ts)
- [DSH SessionEventMap](../../agentfw/deepseek-harness/packages/core/session/src/types.ts)
