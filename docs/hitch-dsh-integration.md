# Hitch ↔ DSH 对接改动

- 状态：Draft v0.1
- 目的：落实 [DSH 自进化 harness spec](dsh-self-evolving-harness-spec.md) §7 的三件交付物——`dsh-evolving` adapter、DSH stdout NDJSON 事件输出模式、eval 本地源守卫的处理。本文件是这三件事的具体改动设计；spec 只保留结论。
- 代码基线：agent-hitch `src/`、DSH `packages/bundle/headless/`（2026-08-19 源码核查）
- 更新：2026-08-19 — 初版

## 1. 为什么 Hitch 不能"装好直接用"

Hitch 认识哪些 harness 硬编码在 `src/adapters.js` 的模块级 `definitions` 对象里（现有 codex/claude/pi/opencode）。`hitch run --harness <id>@...` 的第一步是 `getAdapter(reference.harness_id)`（`engine.js:190`），查不到抛 `harness_not_found`（`adapters.js:316-322`）。没有插件目录、没有配置注册面、没有 CLI 参数能声明第五个 harness——`dsh-evolving` 必须是对 agent-hitch 仓库的源码提交。

唯一不改源码的路是协议冒充（`HITCH_CODEX_PATH` 指向包装脚本、让 dsh 说 codex 的 `exec --json` 方言），不可行：codex 的 `revision_sources` 解析不了 harness repo 的 commit（只能 `codex@installed` 可执行指纹），不可变解析、prepared artifact、eval 全部失效——这恰恰是需要 Hitch 的全部理由。

这是 pre-alpha 阶段的刻意取舍：adapter 的核心是 `translate()`——把各家 harness 的原生 JSON 事件方言翻译成 Hitch 归一化词表，inherently 每个 harness 一份定制代码。若未来需要通用化，演进方向是配置化 adapter 注册面；V1 一个硬编码定义比设计插件协议便宜一个数量级。

## 2. 改动一：`dsh-evolving` adapter（agent-hitch 源码提交）

照 pi 适配器模板（约 60 行）：

```js
"dsh-evolving": {
  id: "dsh-evolving",
  display_name: "DSH Evolving Harness",
  command: "dsh",
  path_env: "HITCH_DSH_PATH",
  version_args: ["--version"],
  revision_sources: {
    commit: {
      type: "git",
      url: "<harness overlay repo 远端 URL>",
      commands: [ /* 见下 */ ],
      entrypoint: "<launcher 脚本路径>",
    },
  },
  capabilities: {
    non_interactive: true,
    streaming: true,
    structured_messages: true,
    structured_tool_events: true,
    sessions: true,
    resume: false,
    model_selection: false,   // V1：模型由固定 dsh revision 的 settings 决定，符合 spec"模型固定"
    graceful_cancel: false,   // DSH headless 无优雅取消协议；Hitch terminateProcess 杀进程树
  },
  process(request, executable) { /* 见下 */ },
  translate(event, state) { /* 见 §3.2 映射表 */ },
}
```

### 2.1 revision_sources.commit 的构建命令

Hitch 对 `commit:` 选择器的 prepare 流程是 clone 该 URL 的指定 sha、跑 `commands`、以 `entrypoint` 为入口（`artifacts.js`）。构建命令的职责：

1. 安装/检出**固定版本的 dsh**（`manifest.dshRevision`，从 overlay 的 manifest.json 读出）；
2. 物化 overlay 为可执行入口——一个包装脚本，等价于：

```sh
#!/bin/sh
exec <pinned-dsh> --profile headless --events jsonl \
  --patch "<overlay>/harness.cordis.yml" "$@"
```

版本隔离由这里保证：同一 harness repo commit 重复 prepare 得到相同 identity（dsh 版本 + overlay 内容都钉死在 commit 里）。

注意：`commit:` 走注册远端 URL；`git+file://<path>#<sha>` 显式本地源（`harness-reference.js:33-55`）可用于 `hitch run`，本地开发循环不必推远端（见 §4）。

### 2.2 process()

```js
process(request, executable) {
  const args = ["--profile", "headless", "--events", "jsonl"];
  if (request.model) args.push("--model", request.model);  // V1 capabilities.model_selection=false 时不出现
  args.push(...request.agent_args);
  return { executable, args, input: request.prompt };
}
```

prompt 经 stdin 传入（Hitch 引擎统一 `child.stdin.end(specification.input)`）；workspace、timeout、取消由 Hitch 引擎管理，adapter 不管。

## 3. 改动二：DSH stdout NDJSON 事件输出模式（DSH 侧前置 PR）

### 3.1 设计

Hitch run 引擎对子进程只做三件事：stdin 喂 prompt、stdout 逐行 `JSON.parse`、非 JSON 行落入 `process.stdout` 兜底事件（`engine.js:221-235`）。**Hitch 不看你的磁盘，只看你的 stdout。**

DSH 侧改动：headless 增加 `--events jsonl`（命名以 DSH CLI 约定为准）：

- 订阅 `session/event`，把每个 **durable session event** 序列化为一行 JSON 写 stdout；
- run 开始时先输出一行含 session id 的 JSON（供 adapter 翻译成 `session.created`）；
- **互斥模式**：启用时不再输出最终纯文本行——最终文本已含于 `message.completed`，双份输出会造成两个真相源。最终文本在 `summarize()` 的提取逻辑不变，只是出口从 `io.stdout.write(text)` 改为事件投影；
- 投影 durable 事件集（`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*`）与"model-visible ⟺ logged"一致：投影的就是日志里有的，session log 照写不误，NDJSON 只是对外投影。

### 3.2 事件映射表（`translate()` 的依据）

| DSH session event | Hitch 归一化事件 | 说明 |
| --- | --- | --- |
| （run 开始时输出的 session id 行） | `session.created` | session_id 为 DSH SessionId（`session-<uuid>`） |
| `assistant/message` | `message.completed`（+`usage.updated`，若事件携带 usage） | 文本从 `event.data.message.content` 的 text 块拼接（headless `summarize()` 已验证该字段） |
| `tool/call` | `tool.started` | call_id / name / input 从 event.data 映射；**具体字段名以 `SessionEventMap`（`packages/core/session/src/types.ts`）为准，实现时逐字段核对** |
| `tool/result` | `tool.completed` | status 映射 succeeded/failed |
| `turn/end`（reason.kind === 'error'） | `diagnostic`（level: error） | error.code/message 已在 headless stderr 路径验证 |
| `turn/start` / `turn/end`（正常）/ `step/*` / `user/message` | 不映射 | Hitch 词表无对应概念；原始行保留在 events.jsonl 的 `provider.event` 兜底里 |

兜底：未识别的 JSON 行 → `provider.event`（Hitch 引擎对所有 translate 返回统一处理，四家适配器同款行为），原始事件不丢。

### 3.3 为什么不走"事后解析 session log"

技术上 DSH 有 `session-persistence-jsonl`，run 完磁盘上确实有完整事件流。此路不通的原因，按重要度：

1. **关联 key 不出进程门**。session id 在 runner 内部铸造（`headless/src/index.ts:112`，`session-${randomUUID()}`），整个进程生命周期 stdout 只有一次写入——最终文本（`index.ts:129`）；headless 与 session-persistence 中 grep `sessionRoot/sessionDir/session-dir` 零命中，无路径覆写机制。"哪份 log 对应哪次 run"没有机器可读答案，只剩 mtime 启发式——而 Hitch daemon `max_concurrent` 默认 4，baseline/candidate/相邻 round 交错执行是评测常态，猜错是现实风险而非理论风险。
2. **悖论：让事后解析可行的最小改动本身就是 NDJSON**。为解决关联至少要往 stdout 打一行含 session id/root 的 JSON——而这行 JSON 会被 Hitch 解析并翻成 `session.created`。分叉点不存在：同一个 `session/event` 订阅回调里写一行和写 N 行是同一量级改动，证据量差一个数量级。
3. **超时场景丢尾部**。Hitch 超时 `terminateProcess` 杀进程树；DSH 的 `sessions.flush()` 只在 happy path（`whenIdle` 之后）。评测最有分析价值的恰是超时/卡死 case——stdout 已流出的事件行此时已被 Hitch 落盘，未 flush 的 session log 尾部丢失。
4. **权威记录空心化**。spec 契约"Hitch run 文件是权威、plugin 只记引用位置"。证据在别处意味着 `result.json.output` 为空、`events.jsonl` 只有文本噪声，`RefinementRecord` 引用的权威记录里没有内容，审计/重放跨两套存储。
5. **DSH 特例化**。所有 harness 轨迹本应同构（归一化词表）；事后解析要为 DSH 的 `SessionEventMap` 写专门读取器，且 overlay 换 persistence 后端（jsonl ↔ sqlite）时读取器需分支。

结论：事后解析不是不可能（补一行"打印 session root"即可运行），而是不划算——DSH 侧改动位置与成本相同，换来无实时性、超时丢证据、权威记录空心。

## 4. 改动三：eval 本地源守卫的处理

现状（`evals.js`）两个硬守卫：

- `:34-36` harness_ref 必须为不可变 ref（`version:<exact>`/`commit:<sha>`）——可满足；
- `:38-39`、`:80-81` **拒绝 local `git+file` 源，要求 registered remote Git source**——自进化 harness repo 是本地仓库，candidate commit 不在 adapter 注册的远端 URL 上，直接撞墙。

**V1 处理：对照评测不经 `hitch eval`。** 用 `hitch run --harness dsh-evolving@git+file://<harness-repo>#<sha>`（显式本地源，`harness-reference.js:33-55` 支持）执行参数完全一致的 baseline/candidate 两次 run，`RefineService` 读取各自 run 目录 `events.jsonl` 计算分数差。与 V1 只自动接受 declarative mutation 的安全边界一致（无需进程隔离）。

**executable candidate 阶段启用 Harbor eval 时**，三选一：

1. 放宽守卫（agent-hitch 为自有仓库，`evals.js` 两处判定，改动数行；可加 config 开关而非删除）；
2. 为 harness repo 挂真实远端，adapter 注册该 URL；
3. 维持本地 bare mirror + 注册 file URL（若 Hitch 后续放开）。

倾向 1：改动最小且不引入网络依赖；Harbor 镜像构建（dsh + Node 22 + kernel Python 栈）另行解决，与本文件解耦。

## 5. 实施顺序与验收

依赖顺序（1 是 2 的前置，2 是 3 的前置）：

1. **DSH `--events jsonl` PR**：headless 事件投影 + 互斥输出；验收——同一 task 跑两遍，stdout JSONL 行集一致（除时间戳/id）；含 keyless snapshot 测试（DSH 仓库规范：模型可见行为变更必须带 keyless snapshot）；
2. **adapter 源码提交**：验收——`hitch list --json` 出现 `dsh-evolving`；`hitch run --harness dsh-evolving@git+file://…#<sha>` 的 `events.jsonl` 含完整归一化事件（session.created、≥1 message.completed、tool.started/completed 成对）；
3. **对照评测闭环**：同一 seed task 的 baseline/candidate 两次 run，参数除 harness ref 外逐字段一致（request.json diff 验证），RefineService 能从两个 run 目录算出分数差。

## 6. 参考

- [DSH 自进化 harness spec §7](dsh-self-evolving-harness-spec.md)
- [Hitch adapters（硬编码注册表）](../../agent-hitch/src/adapters.js)
- [Hitch run engine（stdout 消费模型）](../../agent-hitch/src/engine.js)
- [Hitch harness-reference（git+file 显式源）](../../agent-hitch/src/harness-reference.js)
- [Hitch evals（本地源守卫）](../../agent-hitch/src/evals.js)
- [DSH headless runner（现状输出）](../deepseek-harness/packages/bundle/headless/src/index.ts)
- [DSH SessionEventMap（映射字段权威）](../deepseek-harness/packages/core/session/src/types.ts)
