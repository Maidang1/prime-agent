# RLM → dsh 插件化：Stage 2 交接文档

状态日期：2026-09-26（Stage 2 + Stage 3 完成）。接手的 agent 请先完整读本文，再动手。

---

## 1. 一句话现状

把 prime-agent 的 RLM 核心（持久 Python REPL + Continual Harness）移植成 deepseek-harness（下称 **dsh**）的插件。**Stage 0/1/2 全部收尾并通过全部门禁**。Stage 2（`rlm-bindings`，即 Python 运行时 `host_request` 的宿主侧实现）已按 §6 设计落地：6 个 src 文件 + 6 个 spec、91 个测试全过、逐文件 100% 覆盖，并经真实 CPython 内核做了端到端 smoke（`rlm.run` / `rlm.list_subagents` / `rlm.find_models` / `rlm.collect` 全链路通过 Python 侧强校验）。两个仓库均无 commit，全部改动未提交。

### Stage 2 完成时的偏差与决定（替代 §10 的拍板项）

1. `session_dir` / `session_file` 只做展示路径、不建目录 — 已按此实现并写进 README Known Limitations。
2. ~~`bash.completed` / `bash.consumed` 只 ack~~ — **Stage 3 已完成**：completed 向所属会话 steer `[bash-done pid:N exit:X]` + `Command: <json>` 文本（dsh 无 custom role，走 `agent.steer(createUserMessage)`，source kind `rlm-bindings`）；consumed 经 `agent.inbox.remove(messageId)` 撤回仍 pending 的通知（`src/bash.ts` 的 `BashNoticeBoard` 按 pid+command 取最早一条）。已被 step 领取的通知无法撤回，写入 Known Limitations。dsh 提交 `feat(rlm): steer background bash completion notices into the owning session`。
3. `rlm.create_session` 与 `rlm.run` 共用 `startContinuable` 一条路径 — 已实现。
4. changelog fragment：五个 rlm 包均已补齐（`rlm-bindings` 新包一份；`rlm-kernel` 一份两 bullet，含 seam 本体与 `registerHostRequestHandlers`；tool-python / rlm-kernel-python / rlm-harness 各补一份 Stage 0/1 交付说明）。
6. **Stage 0/1 移植缺口（web 实测发现，已修）**：dsh 的 `rlm-kernel-python` 从不执行 bootstrap cell，用户命名空间里没有 `bash` / `rlm` / `mcp` 绑定（裸 `bash()` 报 NameError）。prime-agent 在 `ipython.ts` 的 `buildRlmBootstrapCode` 里做这件事。修复：provider 在 ready 握手后、acquire 返回前跑 bootstrap cell（预留请求 id `'0'`，不影响消费者 cell 序号；失败则 acquire 失败）。`py/rlm/repl.py` 的 `_ALWAYS_SKIP` 本来就为这三个名字设计（list_names/snapshot 均跳过）。dsh 提交 `fix(rlm): bind runtime conveniences into the kernel namespace at startup`。
5. 修正了交接期未发现的缺陷：`rlm-kernel/src/index.ts` 的 seam 新方法用了 `RlmHostRequestHandler(s)` 却漏了 type import（只有 `tsc -b` 整树才暴露）；`tsc -b tsconfig.host.json` 会类型检查所有包的 tests（与 §9 的旧说法相反），Stage 0/1 的 5 个测试文件共 10 处类型错误已一并修复。

---

## 2. 仓库与文件状态

### 2.1 dsh：`/Users/bytedance/codes/open-source/deepseek-harness`

分支 `main`，`git log -1` = `46a7f68`（rel/dsh-0.1.7-rc.1）。以下为**全部**未提交改动：

```
 M docs/config-catalog.i18n.yaml
 M docs/config-catalog.md
 M docs/config-catalog.zh.md
 M docs/tool-catalog.i18n.yaml
 M docs/tool-catalog.md
 M docs/tool-catalog.zh.md
 M packages/preset/agent-preset/skills/cordis-composition-reference/references/packages.md
 M scripts/check-workspace-constraints.ts
 M scripts/gen-tool-catalog.ts
 M scripts/verify-package-readme-model-experience.ts
 M tsconfig.base.json
 M tsconfig.host.json
?? packages/rlm/
```

`packages/rlm/` 下 4 个已建成包 + 1 个空壳：

| 包 | 状态 |
|---|---|
| `packages/rlm/rlm-kernel` | 完成。`ctx.rlmKernel` 能力 seam（`RlmKernel` 抽象类）。 |
| `packages/rlm/rlm-kernel-python` | 完成。CPython provider，`py/rlm/` 运行时随包发布。 |
| `packages/rlm/tool-python` | 完成。模型面 `python` 工具。 |
| `packages/rlm/rlm-harness` | 完成。`ctx.rlmHarness` harness 状态 seam（与 rlm-kernel-python 同目录装配）。 |
| `packages/rlm/rlm-bindings` | **空壳**。只有 `package.json` + 三个空目录 `locale/ src/ tests/`。 |

`packages/rlm/rlm-bindings/package.json` 是上一个 turn 的草稿，内容基本可用（name / exports / files 顺序 / `dsh.bundle.patch` / peer+dev 依赖），**但 peerDependencies 的清单是拍脑袋写的**，动手前请对照 §7 的依赖表校正。注意：`locale/`、`src/`、`tests/` 三个空目录 git 不跟踪，删掉重建无副作用。

### 2.2 prime-agent：`/Users/bytedance/codes/open-source/prime-agent`

未提交：`RLM-DSH-PLUGIN-PLAN.md`（340 行，Stage 0/1 的定稿计划）、`RLM-DSH-STAGE2-HANDOFF.md`（本文）。这两个文件故意放在 prime-agent 仓库里而不是 dsh 仓库——dsh 有 `doc-budgets` / `md-wrap` / `translation-pairing` / `md-links` 门禁，会lint 这些散文档。

### 2.3 Stage 0/1 已完成的具体事项

- `packages/rlm/` 四包：8 files / 109 tests 全过，逐文件 100% 覆盖。
- 修过 3 个真实缺陷：`rlm-kernel-python/src/index.ts` spawn 行的 `const child: ChildProcess = spawn(...)` 类型收窄问题；`lifecycle.spec.ts` 的 inline import 改为静态 import（否则 tsgolint 报 190+ 条 `no-unsafe-*`）；`tool-python/tests/python.spec.ts` 去 `as unknown` cast、`render` 内联调用、`oxlint-disable-next-line typescript/prefer-promise-reject-errors`（仓库先例见 `packages/llm/llm/tests/service.spec.ts:466`）。
- `tool-python/cordis.patch.yml` 新建（`insert` tool-python 行，`inject: [tools, rlmKernel]`，`maxCodeChars: 100000`）；`rlm-kernel-python/cordis.patch.yml` 新建。
- Stage 0 验收：`dsh --profile rlm --from-default-profile sdk-minimal` + 自写 profile patch 插 3 行，`--dump-config` 看到 rlm 行；再写脚本调 `runProfile()` 实测 `ctx.rlmKernel` / `ctx.rlmHarness` / `ctx.tools.get('python')` 均已挂载。
  - **坑**：`sdk-minimal` 树在 boot 返回约 50ms 后自动 dispose（与是否打 rlm 行无关，不是 RLM 缺陷）。脚本用 `ctx.effect(() => new Promise<void>(() => {}))` 续命也无效，只有真实 surface 才留得住。写 e2e 验证脚本时注意这点。
- Stage 2 的两处 seam 改动**已落码且验证 109 tests 仍全过**：
  - `packages/rlm/rlm-kernel/src/index.ts`：`RlmKernel` 新增具体方法 `registerHostRequestHandlers(handlers): () => void`（`private readonly hostRequestRegistrations` 数组 + 幂等 disposer）和 `protected hostRequestHandler(own, type)`（先查 per-acquire `entry.hostRequests`，再按注册顺序查全局注册）。因为是**具体方法而非 abstract**，现有测试里的 `StubKernel extends RlmKernel` 无需改动。
  - `packages/rlm/rlm-kernel-python/src/index.ts:551`：`const handler = type === undefined ? undefined : this.hostRequestHandler(entry.hostRequests, type)`（原为 `entry.hostRequests[type]`）。

---

## 3. 权威参考实现（最重要，先读这个）

Python 运行时是从 prime-agent 搬过来的，所以 **prime-agent 侧有这套 host handler 的原始实现**，移植时逐条对齐，不要重新发明：

| 作用 | 路径 | 行数 |
|---|---|---|
| wire 层校验 + 错误消息 | `packages/coding-agent/src/core/rlm-runtime.ts` | 501 |
| 语义层（run/list/collect/progress/delete） | `packages/coding-agent/src/core/agent-session.ts` | 13758（只需看下述区间） |

`agent-session.ts` 关键区间：

- `_createKernelHostHandlers()` — `:10372-10430`，完整的 handler map 注册处，是本次移植的**总纲**。
- `noteRlmProgress()` — `:10833`。
- `_rlmRegistryExtrasFromSnapshot()` — `:10853`。
- `_buildRlmSubagentList()` — `:10888`。
- `collectRlmChildren()` / `_rlmCollectEntryForRun()` / `_rlmChildRunMatchesTarget()` — `:10977-11075`。
- `_resolveDirectRlmSubagent()` / `_rlmSubagentMatchesTarget()` — `:11077-11120`。
- `deleteInactiveRlmSubagent()` — `:11122`。
- `_rlmChildSnapshotForRun()` — `:11442`；`_rlmChildSnapshotForSession()` — `:11521`。
- 常量与纯函数 — `:1078`（`RLM_PROGRESS_NOTE_MIN_INTERVAL_MS = 10_000`）、`:1080`（`RLM_CHILD_PROGRESS_NOTE_RING_MAX = 5`）、`:1081`（`RLM_CHILD_STALE_ACTIVITY_THRESHOLD_MS = 10 * 60_000`）、`:1084`（`RLM_REGISTRY_ANSWER_PREVIEW_MAX_LENGTH = 200`）、`:1086`（`RLM_REGISTRY_LABEL_MAX_LENGTH = 200`）、`:1257`（`compactRlmText`，默认 160）、`:1268`（`rlmChildLabel`）、`:1277`（`touchRlmChildActivity`）、`:1283`（`rlmActivityStaleMs`）、`:1305`（`readAssistantText`）。
- `rlm-runtime.ts` 的常量与纯函数：`:132`（`RLM_SUBAGENT_SESSION_NAME_MAX_LENGTH = 64`）、`:133-134`（`DEFAULT_RLM_MODEL_SEARCH_LIMIT = 8` / `MAX_RLM_MODEL_SEARCH_LIMIT = 20`）、`:135`（`RLM_MODEL_ERROR_SUGGESTION_LIMIT = 3`）、`:137`（`normalizeRequestedRlmSubagentSessionName`）、`:150`（`normalizeRequestedRlmSubagentThinkingLevel`）、`:162`（`normalizeRequestedRlmSubagentModel`）、`:181`（`createDefaultRlmSubagentSessionName`）、`:209`（`normalizeModelSearchText`）、`:213`（`findRlmModelMatches`）、`:245`（`findRlmShortFormModelMatches`）、`:259`（`findUniqueRlmShortFormModelMatch`）、`:280`（`formatRlmModelUnavailableError`）、`:290`（`createRlmCreateSessionHostHandler`）、`:303`（`createRlmRunHostHandler`）、`:322`（`createAsyncBashCompletionHostHandler`）、`:336`（`createAsyncBashConsumedHostHandler`）、`:350`（`createRlmFindModelsHostHandler`）、`:362`（`createRlmListSubagentsHostHandler`）、`:370`（`createRlmDeleteSubagentHostHandler`）、`:387`（`createRlmCollectHostHandler`）、`:434`（`createRlmProgressNoteHostHandler`）。

prime-agent 的 handler map 里还有 `model.info`、`goal.get` / `goal.create` / `goal.complete`、`compact.run` / `compact.status`：
- `model.info` 是 prime-agent skill 用的，**dsh 移植的 Python 运行时不发**，Stage 1 已确认无需实现。
- `goal.*` 与 `compact.*` 是 prime-agent 的 goal / compact skill 专用，**同样不进 Stage 2**，列进 Known Limitations。

---

## 4. Python 侧 host_request 契约（逐字，来自 `py/rlm/`）

载荷统一为 `{...payload, type}`（`type` 最后注入，payload 里的 `type` 键不能改路由，见 `py/rlm/__init__.py:150`）。Python 对 reply 做**强校验**，字段缺失或类型不对直接 `RuntimeError`。

| wire type | 入参 | reply 载荷（Python 强校验） |
|---|---|---|
| `rlm.run`（spawn 走这条，wire 兼容） | `{prompt: str, kwargs: {name, model?, thinking?}}` | `{rlm_child_id, name, session_dir, model}` 全非空 str |
| `rlm.create_session` | `{prompt, kwargs: {name?, model?, thinking?, cwd?}}` | `{active_session_id, session_id, name, session_file, model}` 全非空 |
| `rlm.find_models` | `{query, limit}` | `{models: [{provider, id, name, selector}]}` 四字段全非空 str |
| `rlm.list_subagents` | — | `{subagents: [{rlm_child_id, active_session_id?, session_id?, session_name, session_dir, status ∈ {running,completed,error}, activity?, tool_use_count?, duration_ms?, answer_preview?, replied_since_task?, progress_note?, label?, last_activity_at?, activity_stale_ms?}]}` |
| `rlm.collect` | `{targets: str[], timeout_ms}` | `{results: [{rlm_child_id, session_name?, session_dir?, status ∈ {queued,running,done,error,cancelled}, settled: bool, answer_preview?, error?, duration_ms?, tool_use_count?, replied_since_task?}]}` |
| `rlm.progress.note` | `{message}` | `{accepted: bool, retry_after_ms?}`（retry_after_ms 必须是 int 或缺失，不能是 null/float） |
| `rlm.delete_subagent` | `{target}` | `{subagent: <list_subagents 条目>}` |
| `bash.completed` | `{pid, command, exitCode}` | 仅需 `{status:'ok'}`；非 ok 时 Python 往 stderr 写一行提示然后继续 |
| `bash.consumed` | `{pid, command}` | 同上 |
| `mcp.config` / `mcp.refresh` | `{server}` | — **不实现**，见 §7 边界 |

校验细节（`py/rlm/__init__.py`）：
- `_spawn_handle_from_payload` `:82`；`_create_session_handle_from_payload` `:101`；`_model_from_payload` `:176`；`_subagent_from_payload` `:281`；`_child_result_from_payload` `:337`。
- `activity.kind` 必须 ∈ `{waiting, writing, executing}`，`tool_name` 缺省或 str。
- `activity_stale_ms` / `last_activity_at` / `tool_use_count` / `duration_ms` 走 `_optional_int_field`：`None` 或 int，**`bool` 会被拒**。
- `progress_note` 消息上限 512 UTF-16 code units（Python 侧 `:441` 自己先校验）。
- `collect` 的 `timeout_ms` 必须非负 int；`targets` 支持 handle / row / str / list 混合，归一化成 str 数组后再发给宿主。
- `repl.md`（`py/rlm/repl.md`）是写给模型看的运行时说明，`harness.py:821` 有 spawn 的示例文案。

---

## 5. dsh 侧 API 事实表（已逐一核实，含文件:行号）

> 更完整的摘录在 `/tmp/rlm-bindings-api-notes.md`（由 kimi 子代理产出，共 8 节，逐字签名 + 行号）。该文件在 `/tmp`，**重启即失**，需要就重新生成或按本文的表核对。

### 5.1 subagent

- 类名是 `SubagentRuntime`（**不是** `SubagentService`），`packages/subagent/subagent/src/index.ts:200`，`export default` 在 `:663`，注册为 cordis 服务 `ctx.subagents`。
- `startContinuable(spec: ContinuableStartSpec): Promise<ContinuableStart>` — `:261`
- `interrupt(targetSessionId: SessionId, authority: SubagentInterruptAuthority): void` — `:328`
- `drainContinuableChildren(parent: Agent, childIds: readonly SessionId[]): Promise<void>` — `:359`
- `listChildren(parentSessionId: SessionId, signal?: AbortSignal): Promise<SubagentCatalogEntry[]>` — `:374`
- `sendMessage(sender, targetId, content, options)` — `:279`；`listDescendants(rootSessionId, signal?)` — `:393`；`start(name, request)` — `:559`。
- `ContinuableStartSpec`（`src/types.ts:32`）= `{ provider: string; label: string; childId?: SessionId; request: Omit<SubagentStartRequest,'label'|'signal'|'outputSchema'>; signal: AbortSignal }`。
- `ContinuableStart`（`types.ts:53`）= `{ childId: SessionId; messageId: MessageId }`。
- `SubagentStartRequest`（`types.ts:145`）= `{ label?, prompt: ContentBlock[], parent: Agent, signal, agentOptions?, outputSchema?, maxDepth?, toolFilter?, persona? }`。
- `SubagentInterruptAuthority` = `{kind:'user', parentSessionId}` | `{kind:'ancestor', agent}`。
- `SubagentCatalogEntry`（`src/projection-types.ts:16`）= `{id, createdAt} & ({mode:'one-shot',label?} | {mode:'continuable',label} | {mode:'unknown',label?})`。
- `ContinuableStartSpec` 里 **request 的 label 被 Omit 掉了**：label 只从 spec 的顶层 `label` 进子会话 descriptor。
- `listChildren` 内部用 `ctx.get('sessionQuery').observeSession(parentSessionId)` 读 `subagentCatalog` 投影；缺 service 或投影会抛 `SubagentError`（`src/list-children.ts:78`）。
- spawn provider：`packages/subagent/subagent-spawn-in-process/src/index.ts`，`providerName` 默认 `'spawn'`（`:31`），capabilities 全 true，`inheritsParentContext = false`（子会话不带父历史）。**provider 名是可配的，所以 bindings 的 provider 名也要做成 Config，默认 `'spawn'`。**

### 5.2 llm

- `packages/llm/llm/src/index.ts`，类 `LlmRuntime`，服务 `ctx.llm`。
- `listProviders(): LlmProviderInfo[]` — `:473`（同步，无参数）。
- `listModels(provider: string): Promise<LlmModelInfo[]>` — `:696`；**会抛 `LlmError('INVALID_CATALOG')`**（adapter 元数据非法或重复）。
- `resolveModelInfo(provider, model, signal?): Promise<LlmResolvedModelInfo>` — `:734`。
- `LlmProviderInfo` = `{id, name}`；`LlmModelInfo` = `{provider, id, name, description?, inputModalities?}`。
- `LlmResolvedModelInfo` 多出 `context?:{contextWindow}` / `defaultMaxTokens?` / `reasoning?:{efforts, defaultEffort?}` / `systemPromptUpdate?`。
- `ReasoningEffortId`：`packages/llm/llm/src/brand.ts:67`（类型）+ `:74`（同名 brand 函数）。用法 `ReasoningEffortId('high')`。真实用例见 `packages/subagent/tool-subagent/src/model-selection.ts:124`。
- 模型发现标准写法：`packages/subagent/tool-subagent/src/list-models.ts:36`（`const llm = ctx.get('llm')`，缺服务抛错；selector 格式 `${provider}/${model.id}`）。

### 5.3 session-query / 投影

- `packages/session-query/session-query/src/index.ts:140`：`observeSession(sessionId, options: SessionObservationOptions = {}): Promise<SessionObservation>`；`_static inject = ['sessions']`。
- `SessionObservationOptions`（`src/observation.ts:43`）= `{signal?, projectionMode?: 'all'|'none'}`。**默认不计算投影**，要 `subagentTiming` 必须显式传 `projectionMode: 'all'`。
- `SessionObservation`（`observation.ts:16`）extends `Disposable`：`{source, header, inheritedEventCount, cursor, events, projections?}`。**用完要 dispose**（`using` 或手动 `[Symbol.dispose]()`）。
- `SubagentTimingProjection`（`packages/subagent/subagent/src/projection-types.ts:22`）= `{settledMs: number; active?: {since, through}; lastTurnCompleted?: boolean}`。
  - `settledMs` = descriptor 之后所有已完成 turn 的累计耗时。
  - `active` 存在 ⟺ 有未闭合 turn（**running 的权威信号**）。
  - `lastTurnCompleted === true` ⟺ 最近一个已闭合 turn 正常结束；`false` ⟺ 非正常结束（error/aborted/max-tokens/interrupted/forked，投影不区分）；新 `turn/start` 会清掉它（`packages/subagent/subagent/src/projection.ts:73-77`）。
- 会话事件词汇（`packages/core/session/src/types.ts:281` `SessionEventMap`）：`turn/start`、`turn/end`（`data.reason: TurnEndReason`）、`step/start`、`step/end`、`user/message`、`developer/message`、`system/message`、`assistant/message`（`{turn,step,message,stream,usage?,interrupted?}`）、`assistant/attempt`、`tool/call`（`{turn,step,callId,name,arguments}`）、`tool/result`、`request/header`、`request/context`、`session/end-seed`，外加 subagent 插件注册的 `subagent/descriptor`、`subagent/catalog`。
- `TurnEndReasonMap`（`src/types.ts:196`）：`completed` / `aborted` / `blocked` / `error`（`{error: LlmFailure}`）/ `max-tokens` / `interrupted` / `forked`。崩溃遗留的 open turn 在冷读时被合成为 `interrupted`（`packages/core/session/src/repair.ts:157-163`）。
- 权威的 "child 是否 settled / 完成 / 出错" 判定法（dsh UI 自己的用法，`packages/client/ui-subagent/src/client/SubagentHeaderLineage.tsx:243`）：`activity === 'inactive' && subagentTiming?.lastTurnCompleted === true`。

### 5.4 路径 / 其他

- `AgentOptions`（`packages/core/agent/src/runtime-types.ts:25`）= `{provider?, model?, reasoningEffort?, maxTokens?}`，**没有 `cwd`**。cwd 只能在 session 创建时经 `CreateAgentOptions.meta.cwd` / `CreateSessionOptions.meta.cwd` 传入，而 `startContinuable` 不透传 meta。→ **`rlm.create_session(cwd=...)` 在 dsh 上无法实现，必须显式报错。**
- 路径帮助：`@deepseek-ai/dsh-home-paths`（`packages/util/home-paths/src/index.ts`）：`dshHomePath(...segments)` `:98`、`resolveDshHome(configured?, env?)` `:87`、`dshCachePath({dshHome}, ...segments)` `:108`、`DSH_HOME_ENV = 'DSH_HOME'` `:18`。**`packages/rlm/` 现有 4 个包都没用 dsh home 路径**，rlm-bindings 是第一个，属于新增用法。

---

## 6. Stage 2 设计（已定）

### 6.1 包与文件

新包 `@deepseek-ai/dsh-rlm-bindings`，位于 `packages/rlm/rlm-bindings/`。src 拆 6 个文件，**每个文件必须 100% 覆盖**（dsh 硬门禁），所以保持文件小、分支少、纯函数优先：

| 文件 | 职责 |
|---|---|
| `src/index.ts` | 插件入口：`export const name = 'rlm-bindings'`、`export const inject = ['rlmKernel', 'subagents', 'llm', 'sessionQuery']`、`Config` + `Config` schema、`apply(ctx, config)`。在 `apply` 里组装 9 个 handler 并 `ctx.rlmKernel.registerHostRequestHandlers(map)`，返回的 disposer 交给 fiber。 |
| `src/read.ts` | 纯函数：payload 取值助手（`stringField` / `optionalStringField` / `stringArrayField` / `nonNegativeIntField` / `recordField`）+ reply 构造（`ok(result)` / `fail(message)`）。对齐 prime-agent 的错误消息文案。 |
| `src/child-facts.ts` | **纯函数** `foldChildFacts(observation: SessionObservation): ChildFacts`。从子会话事件 + `subagentTiming` 派生：status（queued/running/done/error）、settled、durationMs、toolUseCount（数 `tool/call` 事件）、answerPreview（最后一个非空 `assistant/message` 的 text 拼接后 compact）、repliedSinceTask（最后 assistant seq > 最后 user seq）、lastActivityAt、activityStaleMs、error（非正常 `turn/end` 的原因文案）。构造合成 `SessionObservation` 做单测，不碰真实 IO。 |
| `src/roster.ts` | 每 parent session 一个 `Roster`：`childId → {name, model, createdAt, sessionDir, progressNote?, lastNoteAt}` + `name → childId` 索引。`spawn` 时**同步预留 name**（`await` 前占位），保证 "name 在兄弟间唯一" 与 `collect`/`delete` 的 O(1) 解析。progress note 的 10s 节流也在这里。Roster 是进程内易失状态，重启即失；durable 事实一律从投影派生，不在这里复制。 |
| `src/subagents.ts` | 5 个子代理 handler + 2 个 bash 通知 handler 的 glue。按 §7 的契约表实现。 |
| `src/models.ts` | `rlm.find_models`：遍历 `llm.listProviders()` → 逐个 `llm.listModels()` → 按 query 打分排序切片。移植 `findRlmModelMatches` 的打分（exact=0 / prefix=3+i / partial=6+i，无 query 时 score=0，非有限分被过滤，同分按 selector `localeCompare`）。 |

配套：`package.json`、`tsconfig.json`（照 `rlm-kernel-python/tsconfig.json`，references 加 `packages/subagent/subagent`、`packages/llm/llm`、`packages/session-query/session-query`、`packages/core/agent`）、`cordis.patch.yml`、`locale/en.json` + `locale/zh.json`、`README.md` / `README.zh.md` / `README.i18n.yaml`、`tests/*.spec.ts`（`read` / `child-facts` / `roster` / `models` / `subagents` / `index` 六个 spec）。

**不写 `tsdown.config.ts`**（tool-python 也没有）。

### 6.2 handler 注册点

`ctx.rlmKernel.registerHostRequestHandlers({...})`（seam 已在 Stage 2 前半落码，见 §2.2）。handler 从 `context.agent`（`RlmHostRequestContext`）拿 parent，`agent.id` 即 SessionId。`context.signal` 是 handle 被 dispose 时 abort 的信号，作为 `startContinuable` 的 `signal` 传入。

### 6.3 9 条 wire 的映射契约

| wire type | dsh 调用 | reply | 备注 |
|---|---|---|---|
| `rlm.run` | `startContinuable({provider: config.providerName, label: name, request: {prompt: [{type:'text',text: prompt}], parent: agent, signal, ...agentOptions}, signal})` | `{rlm_child_id: childId, name, session_dir, model}` | `model` 缺省时取 parent 的 `provider/model`；parent 也没配则报错 |
| `rlm.create_session` | 同上，但显式传 mint 的 `childId: SessionId(uuid)` | `{active_session_id, session_id, name, session_file, model}` | `cwd` 非空 → **报错**（无法透传）；name 缺省时用 `createDefaultRlmSubagentSessionName` |
| `rlm.find_models` | `llm.listProviders()` + 每 provider `llm.listModels()` | `{models:[{provider,id,name,selector}]}` | limit 缺省 8，范围 1..20 |
| `rlm.list_subagents` | `subagents.listChildren(agent.id, signal)` + 每个 child `sessionQuery.observeSession(id, {signal, projectionMode:'all'})` | `{subagents:[...]}` | 见 §6.4 的字段派生 |
| `rlm.collect` | 同上的读 + 按 `timeout_ms` 等 child 收敛 | `{results:[...]}` | timeout=0 → 立即快照；超时返回当前快照，**绝不抛错**；target 无匹配或歧义 → 抛错 |
| `rlm.progress.note` | 写 roster + 节流 | `{accepted: bool, retry_after_ms?}` | 节流 10s，被拒返回 `retry_after_ms`，**不抛错** |
| `rlm.delete_subagent` | 解析行 → `subagents.drainContinuableChildren(agent, [childId])` → 从 roster 移除 | `{subagent: <行>}` | 目标仍在跑 → `outcome: 'skipped_running'`，保留 roster |
| `bash.completed` | 无 | `{}` | Stage 2 只 ack；prime-agent 会往父会话 steer 一条消息，留 Stage 3 |
| `bash.consumed` | 无 | `{}` | 同上 |

### 6.4 `list_subagents` 行的字段派生

- `rlm_child_id` ← catalog entry `id`
- `active_session_id` / `session_id` ← 同一个 childId（dsh 的 continuable child 只有一个 durable session id）
- `session_name` ← roster name，回落 catalog `label`
- `session_dir` ← `dshHomePath('rlm', 'children', childId)`
- `status`：`active !== undefined` → `running`；`lastTurnCompleted === true` → `completed`；`lastTurnCompleted === false` → `error`；两者都缺 → `running`（已准入未开 turn）
- `tool_use_count` / `duration_ms` / `answer_preview` / `replied_since_task` / `last_activity_at` / `activity_stale_ms` / `error` ← `foldChildFacts`
- `progress_note` ← roster（ring buffer 上限 5，取最新）
- `label` ← roster name，截到 200
- `activity` ← **不产出**。dsh 没有对应投影，Python 侧是 optional。写进 Known Limitations。
- `activity_stale_ms` ← `rlmActivityStaleMs` 语义：仅 running、且 wall 与 monotonic 取小、≥ 600_000 才给。

`collect` 的 status 映射：无观察 / 无 descriptor → `queued`；`active !== undefined` → `running`；`lastTurnCompleted === true` → `done`；`false` → `error`。`cancelled` **不产出**（dsh 没有单独暴露给 bindings 的取消信号）。`session_dir` 同 `list_subagents`。

---

## 7. 边界：明确不做（写进 README 的 Known Limitations）

1. `cwd` kwarg 不支持（`AgentOptions` 无此字段，`startContinuable` 不透传 session meta）→ 显式报错。
2. `session_dir` 是**稳定的展示路径，不创建目录、不写文件**。prime-agent 的 `sessionDir` 是真目录；dsh 的 JSONL 布局是 provider 私有的（`join(project, encodeSegment(id))` + generation 文件名），无公开 API。→ 需要你拍板：是否接受"只读路径"语义。
3. `activity` 不产出。
4. `bash.completed` / `bash.consumed` 只 ack，不 steer 父会话。
5. `model.info`、`goal.*`、`compact.*` 不实现（prime-agent 的 goal/compact skill 专用）。
6. `mcp.config` / `mcp.refresh` 不实现 → Python 侧 `RuntimeError` 冒泡到模型，够响。
7. roster 是进程内状态，宿主重启后 `progress_note` / name→childId 预留丢失；name 会回落 catalog label。注意 `spawn` 的唯一性校验在重启后可能放行重名（真实 catalog 不会重名，但 roster 空窗期可被绕过）。**这条要写进 README。**
8. `list_subagents` / `collect` 对每个 child 做一次 `observeSession`，N 个 child = N 次读 + N 次事件展开（数 `tool/call`）。child 数大时会慢。prime-agent 的内存快照没这个成本。→ 可考虑加配置上限，或接受并在 README 说明。

---

## 8. 待办清单（按依赖顺序）

1. 校正 `packages/rlm/rlm-bindings/package.json`：peerDeps 只保留真实 import 的（预计 `dsh-agent` / `dsh-llm` / `dsh-rlm-kernel` / `dsh-session` / `dsh-session-query` / `dsh-subagent` / `dsh-home-paths` / `cordis`），devDeps 同步加齐；`dependencies` 只有 `@deepseek-ai/schemastery`。
2. `tsconfig.json`（照 `rlm-kernel-python/tsconfig.json`，references 增补 subagent / llm / session-query / core/agent / util/home-paths / util/brand 视实际 import 而定）。
3. `cordis.patch.yml`：`insert` 一行 `rlm-bindings`，`inject: [rlmKernel, subagents, llm, sessionQuery]`，config 带默认值。注释要说明"这行依赖 subagents 与 spawn provider，缺了就在启动时大声失败而不是首次调用时失败"。
4. `locale/en.json` + `locale/zh.json`（`meta.title` / `meta.description`）。
5. `src/` 六个文件，顺序：`read.ts` → `child-facts.ts` → `roster.ts` → `models.ts` → `subagents.ts` → `index.ts`。
6. `tests/` 六个 spec，**逐文件 100% 覆盖**。`read` / `child-facts` / `roster` / `models` 用纯单测；`subagents` / `index` 用 fake（dsh 有 faux provider 惯例，见 `packages/agent-loop-testkit`，tool-python 的 devDeps 就依赖它）。
7. `tsconfig.base.json:423` 后（`@deepseek-ai/dsh-rlm-kernel-python` 之后、`@deepseek-ai/dsh-sandbox` 之前）插一行 paths；`tsconfig.host.json:321` 后插一行 references。
8. 重新生成 + 手工补译：
   - `node --import tsx/esm scripts/gen-tool-catalog.ts`
   - `node --import tsx/esm scripts/gen-config-catalog.ts`
   - `node --import tsx/esm scripts/gen-plugin-packages.ts`
   - 然后跑 `--check` 变体验证
   - `docs/tool-catalog.zh.md` / `docs/config-catalog.zh.md` 的**新节**要手工补：fence 内必须与英文逐字节相同，只译 prose 与"来源/需要/依赖"行。
   - `packages/preset/agent-preset/skills/cordis-composition-reference/references/packages.md` 的 `## rlm` 表加一行。
9. README 三件套。**硬性格式要求**（有专门门禁 `verify-package-readme-model-experience`）：
   - `Known Limitations and Deferred Work` 必须是逐字的这个 h2。
   - `Model Experience` 与 `Further Exploration` 是最后两个 H2。
   - `Summary` ≤ 100 词。
   - prose 不硬换行。
   - 参照 `packages/rlm/tool-python/README.md` 的结构逐节写。
   - `README.i18n.yaml` 用 `pnpm run verify-translation-pairing --write <path>` 重录 hash。
10. changelog fragment：`packages/rlm/rlm-bindings/.changes/<slug>.md`，一行 `- ...`，过去式动词开头。**注意 rlm 四个已建包都没有 `.changes/`**，说明 Stage 0/1 没建 fragment——接手后确认是否要补。

### 8.1 门禁清单（全部要绿）

`constraints`、`package-invariants`、`package-meta`、`package-dependencies`、`npm-install-layout`、`cordis-config`、`runtime-closure`、`type-equiv`、`doc-budgets`、`md-links`、`md-wrap`、`translation-pairing`、`export-jsdoc`、`no-unknown-casts`、`gen-tool-catalog --check`、`gen-config-catalog --check`、`gen-plugin-packages --check`、3 个 README 门禁。加上 `tsc -b`、vitest、oxlint 全 clean。

---

## 9. 环境与命令速查（全是坑）

- **`pnpm run <script>` 会触发 install 检查并失败**（连不上 `bnpm.byted.org`）。必须 `node --import tsx/esm scripts/<name>.ts`。tsx 会读 tsconfig paths，裸包名可解析。
- vitest：`node node_modules/vitest/vitest.mjs run <path>`。覆盖率加 `--coverage --coverage.reporter=json --coverage.reportsDirectory=/tmp/rlm-covN`，再用 `node -e` 解析 `coverage-final.json` 只筛 `/packages/rlm/`。
- oxlint：`node --import tsx/esm scripts/run-oxlint.ts <path>`。
- tsdown：`node node_modules/tsdown/dist/run.mjs`（`dist/run.js` 不存在）；包内无本地 `tsdown.config.ts` 时会在仓库根配置下以 workspace 模式找包，单包构建需 CLI 传 entry。
- tsc：`node node_modules/typescript/bin/tsc -b packages/rlm/<pkg>`。**测试文件不在任何 tsconfig include 里**，只有 oxlint 的 tsgolint 会检查它们。
- cordis：`ctx.dispose()` 不存在，用 `await ctx.fiber.dispose()`；`ctx.emit('agent/disposed', { agent })` 可用。
- schemastery Schema 没有 `.parse()`。`exactOptionalPropertyTypes` + `noUncheckedIndexedAccess` 开启 → 条件展开 `...x === undefined ? {} : { x }`；禁 `as unknown`。
- `expectedDshPackageFiles()` 的 files 顺序：`[icon?]`、`locale/*.json`(sorted)、`lib/index.js`、`lib/invariant.js`(若 export)、`lib/bin.js`、worker/loader/client/store/startup 特例、`<bundlePatch>`、`<packageFileExtras>`、`lib/types/**/*.js`(若 usesEmittedTreeDefaults)、`lib/types/**/*.d.ts`、typert 特例。rlm-bindings 用 `["locale/*.json","lib/index.js","cordis.patch.yml","lib/types/**/*.d.ts"]`。
- python3 3.14.6 可用但无 dill → snapshot e2e 用 `hasDill()` 守卫跳过；`~/.prime/agent/kernel-venv/bin/python` 有 dill 0.4.1。
- `__pycache__` 和 `lib/` 已被 `.gitignore` 覆盖。
- 用户偏好：中文、简洁技术性、无表情符号；只用 `apply_patch` 改文件（**本环境没有 `apply_patch` 工具**，实际一直用 `python3 - <<'PY'` 或 `node /tmp/x.cjs` 脚本落盘；注意 heredoc 里 Python 三引号不能出现在 `.cjs` 里）；禁 `git add -A` / `git reset --hard` / `git checkout .` / `git clean -fd` / `git stash`；不跑 `npm run dev/build/test`。

---

## 10. 需要用户拍板的 4 件事

1. `session_dir` 只做展示路径、不建目录 —— 接受吗？
2. `bash.completed` / `bash.consumed` 只 ack —— Stage 2 接受吗，还是本轮就把父会话 steer 做掉？
3. `rlm.create_session` 与 `rlm.run` 在 dsh 上是否就**共用 `startContinuable` 一条路径**（差别只是 mint childId + name 可缺省），还是需要更接近 prime-agent 的"resident daemon session"语义？
4. Stage 0/1 未建 changelog fragment，本 stage 是否补一个（含不含前三个包的）？
