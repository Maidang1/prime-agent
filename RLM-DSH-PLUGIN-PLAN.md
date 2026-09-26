# RLM Core → DeepSeek Harness Plugin：实施计划

状态：待评审。实施仓库：`/Users/bytedance/codes/open-source/deepseek-harness`（下称 dsh）。

## 0. 目标与非目标

目标：把 Prime Agent 的两大核心——**RLM（持久 Python REPL 作为模型工具面）** 与 **Continual Harness（可精炼的 harness 状态）**——抽成一个 Cordis 插件，安装到 dsh 上，使 dsh 获得 prime-agent 的编程式 agent 体验。

非目标（明确不搬）：
- 不搬 `packages/coding-agent/src/core/agent-session.ts`（13k 行）：compaction、goals、scheduling、daemon、session 存储在 dsh 均有等价物。
- 不搬 `packages/tui`、`packages/ai`、daemon 协议。
- 不改 dsh 自身仓库代码（`agent-loop` 等）。一切通过插件挂载。若确需改 `agent-loop`，按 dsh AGENTS.md 需同步更新 `docs/architecture.md`，单独评估。

## 1. 差距分析（为什么是这个形状）

| prime-agent 核心 | dsh 现状 | 结论 |
|---|---|---|
| 持久 Python kernel（跨 turn/压缩保留变量、句柄） | `ctx.ptcRuntime` 的 Python 后端是**每次 run 全新子进程**，且 seam 契约明确要求 "program state separate between runs" | 不能复用 ptcRuntime seam，须自建服务 |
| RLM 编程模型（`bash()` handle、`rlm.spawn`、skill 导入、host request） | 能力以独立 tool 散落暴露（bash/subagent/skill/goal） | 做成 bindings 桥接，复用 dsh 服务 |
| Continual Harness（supplemental prompt/memory/`/refine` 精炼 + 快照回滚） | 无对应物 | 移植 |

## 2. 插件架构（按 dsh capability-seam 规范拆分包）

dsh 规范：一个能力缝 = Service Definition / Service Provider / Consumer 三角，只在独立演进时拆分。RLM kernel 的 provider 与 definition 会独立演进（未来可能有 sandbox provider），故拆开。

新增 workspace 包（group：`packages/rlm/`）：

| 包 | 角色 | ctx key |
|---|---|---|
| `rlm-kernel` | Service Definition：持久 kernel 生命周期、run/cancel/interrupt/snapshot 契约、host handler 类型 | `ctx.rlmKernel` |
| `rlm-kernel-python` | Service Provider：CPython 持久子进程实现（协议客户端 + 进程管理 + 随包发布 `py/`） | 注册到 `ctx.rlmKernel` |
| `tool-python` | Consumer：模型面 `python` 工具（`ctx.tools.register`），把 kernel run 包装成 tool，abst 信号接 `exec.signal` | 注册到 `ctx.tools` |
| `rlm-harness` | Service + Consumer：harness 状态存储与 `/refine` 精炼 | `ctx.rlmHarness` |

bindings（host request → dsh 服务）放在 `rlm-kernel-python` 内的 bridge 模块，因为它持有 kernel、需要读 ctx；也可在 stage 3 视体量拆出 `rlm-bindings`。

## 3. 从 prime-agent 移植的文件映射

### TypeScript → dsh 插件

| prime-agent 文件 | 行数 | 去向 | 裁剪说明 |
|---|---|---|---|
| `packages/coding-agent/src/core/kernel/repl-manager.ts` | 1663 | `rlm-kernel-python/src/` | 移植协议客户端（stdin/stdout JSON-lines v3，规格见 `prime-agent-runtime/src/rlm/repl.md`）、进程管理、dill 快照、SIGINT 中断。外部耦合仅 3 处：`spawnHidden`（内联）、`orphan-process-journal`（删除，改 effect 析构即 dispose）、`bootstrap.ensureKernelPython`（换 dsh Config 解释器） |
| `packages/coding-agent/src/core/kernel/bootstrap.ts` | 961 | `rlm-kernel-python/src/bootstrap.ts` | **只移植三小块**：`RUNTIME_READY_CHECK`（改断言集）、解释器探测（uv venv/锁/缓存/自动安装 ~700 行全部删除，改 dsh Config fail-loud）、Windows batch shim。另注意其会往 venv 装一整套数据科学包（requests/httpx/pyyaml/pandas/numpy/scipy/bs4/lxml/pydantic/tyro 共 12 个 `DEFAULT_RLM_EXTRA_PACKAGES`）——dsh 侧不自动装，降级为 README 记录的可选 extras |
| `packages/coding-agent/src/core/kernel/shared.ts` | 338 | `rlm-kernel/src/types.ts` | 基本原样（`HostRequestHandler(s)`、快照配置等） |
| `packages/coding-agent/src/core/kernel/state-snapshot.ts` | 47 | `rlm-kernel-python/src/` | 原样 |
| `packages/coding-agent/src/core/rlm-runtime.ts` | 501 | `rlm-kernel/src/handlers.ts` | 原样（纯 handler 工厂，无 session 依赖） |
| `packages/coding-agent/src/core/prompts/rlm.ts` | 241 | `tool-python/src/prompt.ts`（section 文案源） | 移植四个文本块 + `buildRlmPrompt`/`buildChildAgentDoctrine`/`buildSubagentGuidance`。适配点：① 工具名 `ipython`→`python`；② `uv pip install` 与 "Pre-installed Python packages" 两行删除/改写（dsh 不自动装 extras，见 §3 bootstrap 行）；③ `messagesPath`→dsh session JSONL 路径；④ `skillsDir`/`installedSkills`→`ctx.skills` catalog；⑤ depth 文案对 `ctx.subagents` 的深度策略 |
| `packages/coding-agent/src/core/refinement/refinement.ts` | 1214 | `rlm-harness/src/` | 大部分是纯函数（HarnessState CRUD/merge、proposal 规范化与应用、prompt 格式化、query 打分、历史合并），原样移植；仅 `planRefinement`/`refineHarness` 两个入口有 4 个适配点，见 §6 |
| `agent-session.ts:10373` `_createKernelHostHandlers()` | ~90 行 handler 注册 | `rlm-kernel-python/src/bridges.ts` | 按 §4 表逐条重写到 dsh 服务 |

### Skills → 插件自带 dsh skill 包（此前遗漏的一类移植件，本轮补齐）

`packages/coding-agent/skills/` 共 13 个 skill，每个 = `SKILL.md`（22–87 行，合计 661 行）+ 可选 `src/<pkg>/` python 包（18 个 py 文件）+ 可选 `references/`。**python 包不移植**（能力改走 §4 桥接），**SKILL.md 层要移植**：dsh 的 skill 模型与 prime-agent 同构（磁盘目录 bundle + SKILL.md frontmatter），插件通过 `ctx.skills.register(...)`（embedded）或自带 skill 目录提供，逐个按 dsh API 改写措辞：

| skill | 处置 |
|---|---|
| `agent-message`（60 行） | 改写为 dsh `agent_message` 桥接的用法（§4 `ctx.subagents.sendMessage`） |
| `refine`（42 行） | 改写为 `ctx.rlmHarness` + `refine` 工具的用法（stage 4） |
| `goal`（46 行） | dsh 已有 goal 工具/skill，**不移植**，避免双份 catalog |
| `skill-creator`（81 行 + references/python-skills.md） | 移植；"Python-backed skill"一节改写为"插件如何把 skill 目录注入 kernel sys.path"的 dsh 版说明 |
| `agent-observe`（55 行） | 映射到 dsh agents 视图/查询能力，措辞改写后移植（stage 2） |
| `attach-image`（49 行） | 改写为 dsh `ctx.attachments` 用法（§5 图片通路） |
| `compact`（41 行）、`rlm-heartbeat`（54 行） | dsh 自有 compaction/schedule，**不移植** python 包；SKILL.md 若保留需改写为 dsh 语义，否则放弃 |
| `edit`（22 行） | dsh 自有 fs 编辑工具，**不移植** |
| `websearch`（30 行）、`linear`（45 行）、`notion`（49 行）、`prime-intellect`（87 行） | 走 dsh MCP/工具生态，**不移植**（stage 6 评估 prime-intellect 平台相关部分） |

### Python → 随包发布（`py/**/*.py`，仿 `dsh-experimental-ptc-runtime-python` 的 `files` 写法）

| prime-agent 文件 | 行数 | 处理 |
|---|---|---|
| `prime-agent-runtime/src/rlm/repl.py` | 1196 | 原样移植（stdin/stdout JSON-lines 协议 v3 核心） |
| `prime-agent-runtime/src/rlm/bash.py` | 1204 | 原样移植（host request 协议不变，TS 侧换桥接目标） |
| `prime-agent-runtime/src/rlm/harness.py` | 957 | 原样移植 |
| `prime-agent-runtime/src/rlm/skill.py` + `__init__.py` | 669 | 原样移植；skill 目录来源改为 dsh `ctx.skills` |
| `prime-agent-runtime/src/rlm/mcp.py` + `mcp_base.py` | 995 | 暂缓（见 §7 开放问题 4）；stage 6 可选 |
| `prime-agent-runtime/src/rlm/_winjob.py` | 378 | 随 Windows 支持决策处理（开放问题 5） |
| `prime-agent-runtime/test/*.py` | — | 移植为插件测试 |

### 协议决策（重要）

沿用 prime-agent 的 **stdin/stdout JSON-lines 协议 v3**，不适配 dsh 的 fd-3 协议。理由：fd-3 协议服务于"无状态单次 run"，持久 REPL 还需要 out-of-band 中断、快照、跨 cell 流式输出，语义不匹配；整体移植 Python 运行时 + 其测试的成本远低于适配。

## 4. Host request → dsh 服务映射（API 已逐一核实）

`ctx.subagents` = `SubagentRuntime`（`packages/subagent/subagent/src/index.ts`，providers: `spawn`/`fork`/`acp`/...）：

| host request（Python 侧） | dsh 目标（已核实 API） | 语义吻合点 |
|---|---|---|
| `rlm.spawn` | `ctx.subagents.startContinuable({ provider: 'spawn', request, signal })` → `{ childId, messageId }` | 与 admission handle 完全一致：inbox 接受即返回，不等子 agent 回答 |
| `rlm.collect(targets, timeout)` | 续存子：订阅 `subagent-settled` 通知（携带子最终 assistant 文本）+ 读子 session 尾部；一次性 run：`SubagentRun.result`。超时回退快照、不 reject | 与 prime-agent collect 契约一致 |
| `agent_message.send` | `ctx.subagents.sendMessage(sender: Agent, targetId, content, options)` → messageId；向上发父级、向下发直接子 | 官方 messaging 面；运行中目标在步边界接收，空闲目标被唤醒开新 turn |
| `rlm.list_subagents` | `ctx.subagents.listChildren(parentSessionId, signal)` → catalog entries | 目录即 roster |
| `rlm.delete_subagent` | `ctx.subagents.drainContinuableChildren(parent, [childId])`（+ `interrupt` 处理 running） | settle/释放子 |
| `rlm.progress.note` | 插件本地 note 存储，合并进 listChildren 派生的 roster 快照 | dsh 无等价物，自己持有；pull-based 语义不变 |
| `bash()` | `ctx.shell.execute(ctx.shell.resolve({ command, cwd }))` | 一个 execution handle 同时支持前台结果与后台读 |
| `bash.completed` | 载荷已从 `bash.py:712` 核实：`{type, pid, command（超长截断）, exitCode}`。触发条件：后台进程已结束并被 reap、创建 cell 已过完成屏障、句柄未被创建 cell await 且结果未被读取。桥接：把完成通知经 waking-delivery 送达父 agent，回 `{status:"ok"}`；python 收到 ok 后才会武装撤消通知 | 接收/拒绝协议明确 |
| `bash.consumed` | 载荷已核实：`{type, pid, command}`。仅在 `bash.completed` 被接受后武装，读取结果时触发，撤消父端排队的通知 | "撤消永不追过通知"由 python 侧武装时序保证 |
| bash 无宿主时 | `bash.py` 对 OSError/RuntimeError 全静默（standalone 运行时无 host handler 是合法场景），降级为 stderr 提示 | 插件桥接缺失时 kernel 不崩 |
| `goal.get/create/complete` | `ctx.goals`（`GoalRef { id, revision }`，stale 拒绝；依赖 `ctx.sessionProjections`，base/sdk 系 profile 均含） | — |
| `refine.run/status` | `ctx.rlmHarness`（新服务）；人类入口 `ctx.commands.register({ name: 'refine' })`（`packages/interaction/commands`，无前导斜杠的小写名） | 模型面可再选一个 `refine` 工具 |
| `model.info` | `ctx.llm` + agent 默认模型 | — |
| skill 注入 kernel | `ctx.skills` catalog 条目带 resolved path；skill 是目录 bundle（`SKILL.md`，可含 python 文件）→ 把 bundle 目录加入 kernel `sys.path`（移植 `KernelManagerOptions.pythonSkills` 通道） | 注意：dsh skill 是 markdown 指令包，prime-agent skill 是可导入 python 包——桥接通透传目录，两者都受益 |
| `compact.run/status`、`rlm_heartbeat.*` | 不移植 / 映射 `ctx.schedule`（stage 5 评估） | dsh 自有 compaction；schedule 已有模型面工具 |

## 5. Stage 1 设计草图：kernel 服务 + `python` 工具

### `ctx.rlmKernel` 服务接口（rlm-kernel 定缝，形状待实现期定稿）

```ts
interface RlmKernelService {
  /** One persistent kernel per agent session, created lazily on first use. */
  acquire(agent: Agent, options?: KernelAcquireOptions): Promise<KernelHandle>
  dispose(key: SessionId): Promise<void>
}
interface KernelHandle {
  execute(code: string, options: ExecuteOptions): Promise<ExecuteResult>  // 协议 v3 execute
  interrupt(): void                                                        // SIGINT 语义按 repl.md
  snapshot(): Promise<SnapshotResult>; restore(): Promise<RestoreResult>   // dill，懒加载
  dispose(): Promise<void>
}
```

生命周期（已核实 dsh 机制）：`Agent.id` 即 `SessionId`（session-backed 身份），`Agent.ctx` 是 agent 作用域 context；`ctx.agents` 事件含 `agent/disposed`。provider 维护 `Map<SessionId, KernelEntry>`（键 = `exec.agent.id`），`python` 工具首次执行时惰性创建，`agent/disposed` 时 dispose 对应 kernel，插件卸载时全部 dispose——全部走 `ctx.on()`/`ctx.effect()`，天然可重载。快照（snapshot/restore）在 session resume 时恢复变量，与 dsh 的 `ctx.agents.resume()` 对齐；dill 缺失时降级为"重启丢状态 + 结果里报告"。

**子 agent 的 kernel 策略**：`ctx.subagents.startContinuable` 的子 agent 继承父工具集（含 `python`），会各自惰性创建 kernel。每个 kernel 是一个独立 CPython 进程（内存/启动成本真实存在）——需要：① 递归深度用 dsh 自己的机制（`ctx.subagents.resolveMaxDepth(configured)`，用户设置解析），不移植 prime-agent 的 `rlm-max-depth.ts`；② Config 给一个并发 kernel 数上限（默认按 agent 树自然收敛，多数子 agent 不会真调 python）；③ 只对实际调用过 `python` 的 agent 保留 kernel，`agent/disposed` 即回收。**skill 热更新**：dsh skill-filesystem 会 watch 目录，新 skill 出现时已在运行的 kernel 不会自动获得新 sys.path 条目—— bridge 在 session 内把新 skill 目录追加进目标 kernel（记为 session 事件）。

### `python` 工具（tool-python 包）

ToolDefinition 对照 prime-agent `ipython` 工具（`packages/coding-agent/src/core/tools/ipython.ts`）逐项搬运：
- `name: 'python'`（不复用 `ipython` 名，避免与未来 dsh 官方工具撞名）；单参数 `code: string`。
- 描述涵盖：持久 REPL、top-level await、变量跨调用保留、`bash('cmd')`、优先走项目自身环境。
- 调度：dsh 工具分类器 `executionMode` 返回 `{ kind: 'exclusive' }`（kernel 单线程，对应 prime-agent 的 `executionMode: 'sequential'`）。
- 结果装配与 prime-agent 一致：stdout + stderr + trailing-expression repr + unattributed 后台输出 + kernel 重启提示；`isError` 当 status 为 error/aborted；details 带 durationMs/status/diffs 等（dsh 侧落 `presentationMeta`）。
- 中断后 kernel 仍忙（`KernelBusyAfterInterruptError`）：默许有界等待（`KERNEL_BUSY_REUSE_WAIT_MS`），超时则 kill+重启并在结果前缀 `<kernel_reset>` 通知（prime-agent 的交互二选一在 dsh 降级为内部策略 + 通知，避免阻塞工具执行；README limitations 记录）。
- 输出截断 65536 字符（`DEFAULT_MAX_OUTPUT_CHARS`）。
- 图片/产物通路：kernel 的 display 事件携带图片（prime-agent 的 attachment MIME）→ 插件按 dsh 规矩"先持久化后引用"：字节写入 `ctx.attachments.saveImages()` 得 `ImageAttachmentRef`（内容寻址，存 `<DSH_HOME>/attachments/v1`），工具结果只带引用不带 base64。
- 输出卡片：结果 view 首选 `{ card: 'terminal' }`（stdout/stderr + 退出态）或 generic；diff 类 display 用 `{ card: 'diff' }`。

### 与 dsh PTC 展示模式的相互作用（新发现，影响激活层）

已核实（`packages/core/tools/README.md` + `agent-tool-presentation` README + web-app presets）：工具展示模式是 **per-agent preset** 的三值——`native`（每个工具一个 function definition）/ `ptc`（只有 `run_code` + 生成的 SDK）/ `both`；`ptc`/`both` 需要挂了 PTC runtime，且**同一 agent 内所有工具一起投影，单个工具无法退出 SDK**。sdk-minimal **不挂** ptc-runtime-node，即只有 native；base/web 挂了 `dsh-ptc-runtime-node`，且 web-app 自带 `ptc` preset（`mode: ptc`）。

对本插件的含义：
1. **推荐激活形态固定 `mode: native`**（RLM 教义的前提就是"模型直接面对 REPL 工具"）。RLM profile 的 patch 里对 `tools` 行/agent preset 显式设 native，避免落到 ptc/both。
2. 若用户选 `both`：`python` 会作为 SDK binding 出现在 `run_code` 程序里——语义仍自洽（kernel 跨 run 持久），但 §3 的 prompt 文案需要一个 binding 风味变体（`await python({code: ...})` 而非直接 `python` 调用）。
3. 若用户选 `ptc`-only：模型永远不直接调 `python`，RLM 退化为"run_code 程序里的一个持久绑定"，与 dsh 无状态 PTC 的差异只剩状态保持——此模式下应重新评估插件价值，建议在 README 明确不支持 ptc-only。
4. `run_code`（无状态一次性）与 `python`（持久控制环境）在 `both` 下并存不冲突：教义 prompt 需各自说明适用场景。
- bootstrap 进度通过 tool working message / onUpdate 上报（dsh 工具流式语义待实现期确认）。

## 6. Harness 精炼设计

- `ctx.rlmHarness`：harness 状态（supplemental prompt、memory、skill 描述、subagent 规格）的 CRUD + 版本快照 + 回滚。
- 存储：优先 `ctx.sessions` 事件追加（满足 dsh "model-visible ⟺ logged" 铁律）；工作区级共享状态用 workspace 存储（读 `packages/workspace` README 后定）。
- **待定设计点**：kernel 快照（dill 载荷 + manifest）的落盘位置。影响 session resume 时的 kernel 恢复；候选：session 持久化目录旁（$DSH_HOME/sessions/<id>/ 同级 artifacts）或 `$DSH_HOME/rlm-kernels/<sessionId>/`。因恢复出的变量会影响后续模型请求，快照的存在与版本需以 session 事件记录（可重放性），Stage 1 定。
- 模型面与人类面分离（dsh 惯例）：模型面一个 `refine` 工具；人类入口 `ctx.commands.register({ name: 'refine', ... })`。两者共用同一 `ctx.rlmHarness` 逻辑。
- prompt 贡献：`ctx.systemPrompt.section({ name, text, order })`——runtime-context 贡献者用 `getContextOrder(name)` 分配 order；scoped 注册（`agent.ctx`）只影响单 agent，可做 per-agent 精炼。文案源即 prime-agent 的 `prompts/rlm.ts`（RLM 编程模型教义：REPL 控制、bash handle 语义、harness CRUD、native call contract），按 §3 表的 5 个适配点改造。
- 绝不改 base persona/官方 section：只新增自有 section 行，官方 spine 不改。

### 精炼数据流（已核实 prime-agent 侧实现）

`refineHarness(messages, state, history, model, ...)` = `planRefinement()`（auxiliary LLM 调用产出 `RefinementProposal`）→ `applyRefinementProposal()`（纯数据应用 + `rollbackOf` 记录）。数据模型：`HarnessState { entries: {prompt|memory|skill|subagent → id→HarnessEntry}, refinements: RefinementEvent[] }`，落盘为一个 JSON + `refinements.jsonl` 历史；`HarnessEntry` 带 `reference`（skill 要求 python import + callable）。

四个适配点（移植时唯一需要改的地方）：
1. `completeSimple`/`completeWithProviderRetry`（pi-ai）→ dsh `ctx.llm.stream()` 收集式调用（单次 attempt，重试由插件显式做或依赖 llm-retry 策略）。
2. `serializeConversation(convertToLlm(messages))` 的 trajectory 序列化 → 插件基于 `ctx.sessions`/session-query 自行实现等价序列化（dsh 无同名函数）。
3. `CustomEntry`（prime-agent session 自定义条目）承载 refine 结果 → dsh 用 session 自定义事件（需在 stage 4 定事件名与 `@mode`）。
4. `getAuxiliaryThinkingLevel` → dsh 的 reasoning effort 解析。

状态目录映射：global → `$DSH_HOME/harness/`；local → 当前 session artifact 目录。`harness.py` 已支持 `RLM_HARNESS_STATE_DIR`/`RLM_SESSION_DIR` 环境变量覆盖，kernel 侧由 TS 宿主按会话注入，无需改 Python。

## 7. 形态决策：fork 二开 vs 插件组合（结论：无需 fork）

**结论："只保留 dsh 核心"不必 fork。** dsh 的 profile/bundle 组合机制就是为此设计的，并有官方核心-only 基线 `sdk-minimal` 可抄。已核实的事实：

- `packages/bundle/sdk-minimal/cordis.patch.yml`（158 行）即"核心-only"完整树：agent-loop、agent、session、session-persistence-jsonl、tools、system-prompt、llm(+deepseek adapter)、sandbox-policy、subprocess、terminal(bash/pwsh)、jobs、mcp-resources、invariants——赤裸裸一条 shell工具，无 Web/设置/credential/telemetry/compaction/fs 工具/workspace 指令/skill/jobs 控制台/subagent。
- 自定义 profile：`dsh --profile rlm --from-default-profile sdk-minimal` 从官方模板初始化（新 manifest，空依赖、空 patch，盒内 bundle 从当前安装解析）；`dsh plugin --profile rlm add <pkg>` 用 pnpm 装包并自动把声明了 `dsh.bundle.patch` 的依赖并入层栈。
- 层序：profile bundles → profile `cordis.patch.yml` → 家庭级 `$DSH_HOME/cordis.patch.yml` → `--patch` 覆盖层；后层整份替换目标 row 的 config 或插入新 row，`disabled` 可关行。
- `@deepseek-ai/dsh`（CLI 包）依赖 `dsh-base`，后者依赖全套核心包（`dsh-subagent`、`dsh-subagent-spawn-in-process`、`dsh-skill`、`dsh-goal`、`dsh-jobs-local`、compaction 三件套等，共 92 个依赖）。均为已发布 npm 包，任何 dsh 安装都可被 patch row 直接引用挂载——所以 sdk-minimal 排除的 subagent/skill/goal 可以按需加回，而不是默认全带。
- 约束：外挂插件**不能**自带完整 Cordis 树（"custom plugin composition remains a profile plus ordered patch files"），只能以 bundle/patch 行叠加。

### 推荐形态（无 fork）

1. 插件 npm 包（RLM 四包合并发布或分四个 scope 包）+ 自带 `dsh.bundle.patch` 行（kernel 服务、python 工具、harness 服务、prompt section）。
2. 用户侧三行命令得到"核心+RLM"运行时（**已在真机验证**）：
   ```sh
   dsh --profile rlm --from-default-profile sdk-minimal   # 已验证：生成 profiles/rlm/
   dsh plugin --profile rlm add @<scope>/dsh-plugin-rlm    # profile 目录即 pnpm workspace
   dsh --profile rlm --patch ./rlm-rows.yml               # 追加 subagent/skill/goal 行
   ```
   实测细节：`--from-default-profile sdk-minimal` 生成的 `profiles/rlm/package.json` 为 `dsh-profile-rlm`，`dsh.profile.bundles: ["@deepseek-ai/dsh-sdk-minimal"]`、空 `dependencies`（pnpm 直接可用）；`cordis.yml` 为 `[]` 空树，层序 = bundle patch → profile `cordis.patch.yml` → `--patch`。
3. 若要 prime-agent 式单工具模式，patch 里 `disabled` 掉 `persistent-bash`（sdk-minimal 的 row id，见其 patch yml），只留 `python`。
4. 实施环境已验证就绪：`node apps/cli/lib/bin.js --profile sdk-minimal --dump-config` 在本地 checkout 直接跑通（产物已构建）。

### fork 二开的适用条件与代价

仅当需要**改 spine 包内部行为**（`agent-loop`/`system-prompt`/`agent`，而非走扩展点）时才 fork。当前设计不需要：kernel 是新缝、prompt 走 section 贡献、工具走注册。fork 代价：删 workspace 包须同时通过 `pnpm run constraints`、`verify-application-entrypoints`、publint、snapshot、i18n 全部门禁，且永久承担 upstream 同步；换来的仅是更小的安装体积——而"只用核心"在运行时已由 profile 实现。

## 8. 阶段计划与验收标准

**Stage 0 — 骨架**
- 建 group + 四包。每包按 dsh `docs/cookbook/adding-a-package.md` 清单：`package.json`（private/version 对齐根/type module/main+types+exports/`@deepseek-ai/cordis` 双列 peer+dev/`files` 精确列表，RLM 包加 `py/**/*.py`）、`tsconfig.json`（extends base、rootDir src、outDir lib/types、references vendor+cordis+组内依赖）、`src/index.ts`（`export default` 插件对象，`inject` 声明服务依赖，`apply(ctx)` 内 `ctx.effect()` 注册）、`README.md`（必需 Model Experience 段 + limitations 段）、`locale/en.json`+`zh.json`。
- `tsconfig.host.json` 加 references；`rlm-kernel-python` 的 `dsh.bundle.patch` 指向 `cordis.patch.yml`（插入 kernel/python 工具/harness/prompt section 行）。
- 验收：`dsh --profile web --dump-config` 能看到插件行；`pnpm run constraints` 通过。

**Stage 1 — 持久 kernel + `python` 工具**
- 移植 Python runtime（repl/bash/harness/skill）+ repl-manager 协议客户端 + bootstrap（Config 化解释器）。
- bridge：`model.info`、`bash`。
- 验收：真实 `python3` 子进程冒烟（跨两次 execute 变量保留、中断、stdout 截断）；单测 + e2e；snapshot（模型可见变更，按 dsh testing 政策）。

**Stage 2 — subagent bindings**
- `rlm.spawn/collect/list/delete/progress.note`、`agent_message`。
- 验收：python 内 spawn 出 dsh 子 agent 并回传结果；移植 `test_subagent_registry.py`、`test_create_session.py` 等效用例。

**Stage 3 — skills + goals**
- `ctx.skills` → kernel 注入；`ctx.goals` 三个 host request。
- 验收：skill 在 python 内 `import` 可用；goal 读写走 dsh 投影。

**Stage 4 — harness 精炼**
- `ctx.rlmHarness` + `refine` 工具 + systemPrompt section。
- 验收：一次 refine 改动能从 session log 重放恢复；快照回滚用例。

**Stage 5 — profile/patch 与打包**
- `rlm` profile 模板、`rlm.overlay.yml`（单工具模式）、README 文档、npm 打包 dry-run。
- 验收：`dsh --profile rlm` 全新会话可跑通一次 python 循环 spawn 子任务。

**Stage 6 — 可选**：MCP 桥（mcp.py）、heartbeat→schedule、Windows（_winjob.py）。

## 9. 风险

- R1：`ctx.ptcRuntime` 无状态契约不可违；自建 `ctx.rlmKernel` 是与 dsh 架构者的"新缝"，评审时需按 capability-seam 文档写清三角角色。
- R2：dill 快照依赖 `dill`——已实测可用（0.4.1，跨进程恢复通过）；缺 dill 时降级为重启丢状态，README limitations 写明 + Config 开关即可，风险低。
- R3：`bash.completed` 的"唤醒空闲 agent"语义：载荷契约已从 `bash.py` 核实（见 §4），python 侧有完整的接受/拒绝/降级协议；剩余唯一未知是 dsh 侧的送达通路选型（subagent 管理器同款 waking-delivery 为首选，`agent.inject` 不唤醒已文档化）。stage 1 验证。
- R4：dsh AGENTS.md 强制：README Model Experience 段 + limitations 段、snapshot 测试、effect-based 注册、Config 化可调参数、branded id、无 `unknown` cast。全部按此执行。
- R5：Windows 支持未定（dsh 的 python PTC 仅 Unix）。
- R6：模型可见性=可重放：kernel bootstrap code、skill 注入内容若影响模型请求，需产生 session 事件。
- R7：单工具模式会与 dsh preset/plan-mode 的固定文案冲突：base patch 的 plan-mode 描述点名了 `todo_write` 等工具（见 `packages/bundle/base/cordis.patch.yml`），禁用默认工具后这些文案悬空。单工具 profile 需自带精简 preset，或放弃单工具模式（回到开放问题 2 的“并存”建议）。
- R8：**dsh 是 developer preview（0.1.7-rc.1，README 明示 "THERE WILL BE COMPATIBILITY-BREAKING CHANGES"）**。插件依赖的缝（`ctx.subagents` 的 continuable API、`ctx.shell` resolve/execute、`ctx.systemPrompt.section`、`ctx.commands`）都可能随版本变动。对策：只依赖 core 组稳定缝（tools/agents/sessions/system-prompt/commands/subagents/skills/goals/shell），不依赖 experimental 缝（ptc-runtime 家族）；peerDependencies 锁 `~0.1.x`；每个 dsh 升级跑一次插件契约测试。
- R9：PTC 展示模式是 per-agent 全量投影，`python` 工具无法单独退出 SDK（详见 §5）。RLM profile 若被叠加到 ptc/both preset 上，教义前提被破坏。对策：RLM profile 自带 `mode: native` 的 patch/ preset，README 注明不支持 ptc-only。

## 10. 待你确认的开放问题

1. ~~fork vs 插件？~~ 已定：插件 + `sdk-minimal` 系自定义 profile（§7）。
2. **工具面**：`python` 与 dsh 默认工具（bash/fs 等）并存，还是 patch 成 prime-agent 式单 `python` 工具模式？（建议：默认并存，patch 层提供单工具 profile 供选择）
3. **Python 解释器供给**：已定方向为 dsh 惯例的 load 时 fail-loud Config 字段；细化点：插件随包发布 `py/`（rlm 包），启动时以 `PYTHONPATH` 注入而非要求用户预装 `prime-agent-runtime`——用户只需自备 CPython 3.10+（含 `mcp`，缺 `dill` 则快照降级）。uv 自动 venv（含 12 个数据科学 extras）不搬。若无异议即按此实施。
4. **MCP 桥**（`mcp.py`+`mcp_base.py` 近 1000 行）：首发还是 stage 6？（dsh 自有 `ctx.mcp` 生态，python 侧 MCP 集成可能与 dsh MCP 工具重复——需要时再评估）
5. **Windows 首发**：dsh 客户端支持 Windows，但其 python PTC 仅 Unix。`_winjob.py`（378 行）是否要移植？
6. **开发落点（新）**：dsh 工作区内开发→外发 npm（须过 `packages/*/*/src` 逐文件 100% 覆盖门禁，对移植代码是硬成本），还是直接以 out-of-tree 包开发（自带测试标准，但拿不到 repo 的 snapshot/expected 测试设施，且 §7 的分发流程不变）。建议：workspace 内开发到 Stage 2，用门禁逼出测试债，再外发。

## 11. 实施期 API 速查（已核实）

| 用途 | API |
|---|---|
| 注册工具 | `ctx.tools.register(defineTool({...}))`，返回 disposer；args 由 schema 推导；`exec.signal` 必须遵循；`exec.agent` 是通知归属 |
| bash 执行 | `ctx.shell.resolve(request)` → spec；`ctx.shell.execute(spec)` → `ShellExecution` handle（`done` promise、`exitCode`、消费式 read、`sandbox` facts）；`stdoutMaxBytes`/`onExpiry`/`stdin`/`env` 对进程内可信消费者开放（模型面工具不暴露） |
| 沙箱策略 | `ctx.sandboxPolicy.resolve(session, mode?)`，每次执行前调用（tool-bash 同款） |
| 子 agent | `ctx.subagents.startContinuable(spec)` → `{ childId, messageId }`；`.sendMessage(sender, targetId, content, options)`；`.interrupt(targetSessionId, authority)`；`.drainContinuableChildren(parent, ids)`；`.listChildren(parentSessionId, signal)`；一次性 `.start(name, request)` → `SubagentRun`（`.result` 为终态 promise） |
| settle 通知 | `subagent-settled` 事件自动送达子的直接父（携带子最终 assistant 文本块）——`rlm.collect` 与结果回传的底层机制 |
| prompt | `ctx.systemPrompt.section({ name, text, order })`；runtime 贡献用 `getContextOrder(name)`；`agent.ctx` 作用域只影响单 agent |
| 人类命令 | `ctx.commands.register({ name, description, input?, handler })`（无前导斜杠） |
| skills | `ctx.skills` registry；catalog 条目带 resolved path；skill = 磁盘目录 bundle（`SKILL.md` frontmatter + 任意附带文件） |
| LLM 调用 | `ctx.llm.stream({ provider, model, ... }): AsyncIterable<StreamChunk>`（单 attempt；adapter 自带重试禁用） |
| session 事件 | `session.append(intent, ...)`（SurfaceIntent），事件 JSDoc 需 `@mode` + payload `@param` |

## 12. 测试与合规

### 许可证

- prime-agent 为 MIT（`LICENSE`：Mario Zechner 2025 / Prime Intellect 2026），dsh 同为 MIT（DeepSeek 2026）。搬运约 4k 行 TS + 5.4k 行 Python 合法，但需保留版权声明；分发的 npm 插件 `LICENSE` 需含 prime-agent 归属段，dsh 侧 `THIRD_PARTY_NOTICES.md` 加对应条目。

### 可移植的测试资产（prime-agent 侧盘点）

| 类别 | 文件 | 去向 |
|---|---|---|
| 协议/进程 | `packages/coding-agent/test/repl-kernel-{startup,execute,abort,shutdown,protocol-corruption,pipe-errors,parent-watchdog,mcp-shutdown,state-roundtrip}.test.ts` | `rlm-kernel-python/tests/`（移植为 dsh 单测，子进程直接用真 python3） |
| 工具层 | `ipython-{bootstrap,attachments,cell-diff,cell-preview,cell-wrap,cell-background-output,provisioner}.test.ts` | `tool-python/tests/` |
| 桥接 | `kernel-{bash-shell,goal-skill,agent-message-skill,agent-observe-skill,attach-image-skill,rlm-heartbeat-skill}.test.ts`、`rlm-collect`、`rlm-progress-notes` | 按 stage 2/3 对应包的 tests |
| 精炼 | `refinement.test.ts`、`refinement-outcome-message.test.ts` | `rlm-harness/tests/` |
| Python | `prime-agent-runtime/test/test_{repl,bash,harness,create_session,subagent_registry,agent_message_skill,mcp,mcp_base,winjob}.py` | 随 py/ 发布，dsh 侧以子进程方式驱动 |

### dsh 测试铁律（实施硬约束，来自 docs/testing.md）

- **Coverage gate**：`packages/*/*/src` 逐文件 100% 行覆盖——移植代码不允许留死行，gate 会把未覆盖行当死代码标记。四包在 workspace 开发期间必须过此线。
- **HMR 安全**：每个 registry（tool/kernel/prompt section/command）须有"dispose fiber 后断言清理"的测试。
- **Snapshot**：`python` 工具、refine 结果、prompt section 均属 model/protocol/human-visible 变更，每个 stage 须补 keyless 录制会话场景（headless 档 `snapshots/session/`）。
- **e2e 留 key**：真模型 e2e 自门控 keyless 跳过；kernel 测试不依赖模型（纯子进程协议），不受影响。

## 13. 运行时实证（已在真机执行，非纸面分析）

**协议握手 spike**（2026-09-26，用 prime-agent 现成的 `~/.prime/agent/kernel-venv`，CPython 3.11.13）：

```
$ PYTHONPATH=<prime-agent>/prime-agent-runtime/src python -m rlm.repl
→ ready {protocol: 3, python: "3.11.13"}
← execute {id: e1, code: "x = 40 + 2\nprint('hello')\nx"}
→ stdout "hello from kernel" / result "42" / done {status: ok}
← list_names {id: l1}
→ done {status: ok, names: ["x"]}        # 会话内状态保留确认
← interrupt（空闲时）→ 无事件（按规格忽略）
← shutdown → done ok，exit 0
```

**结论（对计划的影响）**：
1. Python 运行时**零构建**可移植：源码目录 + `PYTHONPATH` 即可启动；dsh 插件只需把 `py/`（rlm 包）随包发布并注入 `PYTHONPATH`，或打进 wheel。
2. 运行时依赖极小：`mcp>=2,<3`（2.2.0 实测）+ `tyro`；`dill` 可选（懒加载，缺了快照降级）。与 dsh ptc-runtime-python 的"CPython 3.10+ 探测"惯例一致，Config fail-loud 方案（开放问题 3 建议项）可行。
3. prime-agent 的 kernel skills 是**独立 python 包**（editable 装入 venv：`agent-message`/`agent-observe`/`compact`/`edit`/`goal`/`refine`/`rlm-heartbeat`/`attach-image`/`websearch` 等，仓库位于 `packages/coding-agent/skills/`）。dsh 侧**不移植这些 python 包**，能力改走 host request 桥接：`refine→ctx.rlmHarness`、`goal→ctx.goals`、`agent-message→ctx.subagents.sendMessage`、`edit/fs→ctx.tools 的 fs 工具`、`websearch/linear/notion→dsh MCP 或工具`。python 侧只保留 `rlm` 核心包，`skill.py` 改为从 dsh skill 目录装配 import 路径。
4. Stage 1 验收标准中的"真实 python3 子进程冒烟"已提前以本 spike 形式验证通过；实施时把该 spike 固化为 `rlm-kernel-python/tests/` 的 e2e。

**追加 spike 2（同一 venv，均已通过）**：
- **host_request 桥接往返**：python 内 `await rlm.host_request('model.info')` → stdout 收到 `{"event":"host_request","id":"<minted>","data":{"type":"model.info"}}` → host 回 `{"type":"host_reply","id":"<minted>","data":{"status":"ok","result":{...}}}` → python 原样拿到 dict 并继续执行（`bridge reply: {'id': 'stub-model', ...}`）。**插件的全部 bindings 都建立在这一机制上，现已实证。**
- **dill 快照跨进程恢复**：进程 A `snapshot`（`saved: ["secret"]`, 49 bytes + manifest）→ 进程 B `restore`（`restored: ["secret"]`）→ 执行 `print(secret)` 得 `[1, 2, 3]`。session resume 恢复 kernel 状态的路径可行；R2 风险降级为"dill 缺失时降级"的纯文档事项。

**追加 spike 3（均通过，Stage 1 验收项全部提前验证）**：
- **运行中 cell 的中断**：`time.sleep` 循环执行中收到 `interrupt` → `error {ename: "KeyboardInterrupt"}` + traceback（指向 `<cell-1>` 源码行，linecache 生效）+ `done status: error`，kernel 随后正常服务。工具 abort（`exec.signal`）路径成立。
- **无宿主桥接的 `bash()`**：python 内 `await bash('echo ...')` 直接 spawn 真实进程并返回 `exit_code/output`——**前台 bash 不需要 host 往返**，`bash.completed/consumed` 只服务于后台句柄场景（与 R3 的范围判断一致：桥接只做后台通知，不做执行代理）。
- **display 事件词表**（UI 素材通路）：bash 活动以 `display` 事件承载，MIME `application/vnd.prime-agent.bash-activity+json`，载荷 `{id, pid, active}`，开始/结束各一条，且可异步到达并归属回原 cell id。dsh `python` 工具需把 display 事件映射为 tool 的 streaming 更新 / `presentationMeta`（prime-agent 侧另有 diff、attachment、agent-message 等 MIME，实现时从 `ipython.ts` 的 details 结构反推）。



## 14. 工作量估算（按已核实的移植面推导）

| 项 | 量 | 性质 |
|---|---|---|
| Python 运行时（rlm 包） | 5.4k 行原样搬运 + `skill.py` 装配改造 | 机械，spike 已验证可运行 |
| TS 协议客户端（repl-manager 等） | ~2.9k 行中 ~1.1k 实际移植（bootstrap 的 venv 机制不搬，ipython.ts 仅取 bootstrap code 模板 + 结果装配），3 处耦合手术 | 中等，协议 spike 已覆盖主要路径 |
| host request 桥接（~15 个 handler） | 新写 ~1.2k 行 | 主要设计工作量，§4 已逐条定目标 API |
| `python` 工具 + 呈现 | ~500 行 | 参照 `ipython.ts` 和 `tool-bash` |
| harness 精炼（refinement） | ~1.2k 行移植 + 4 处适配 ~400 行 | 中等 |
| 测试移植 + 补齐 100% 覆盖 | 套件已盘点（§12） | 大项：dsh 逐文件 100% 覆盖是硬门禁 |
| profile/patch/文档/打包 | 小 | Stage 5 |

结论：主体是"搬运 + 适配"而非新算法；最大不确定性在 dsh 侧测试门禁（100% 覆盖 + snapshot）而非 prime-agent 侧运行时。

**追加 spike 4（插件分发链，真机通过）**：造了一个最小包 `/tmp/rlm-plugin-spike`（仅 `package.json` 声明 `dsh.bundle.patch` + `cordis.patch.yml`，无代码——与 sdk-minimal 同为纯 patch 载体），然后：
```
DSH_HOME=/tmp/dsh-spike-home dsh --profile spike --from-default-profile sdk-minimal --dump-config   # ok
DSH_HOME=/tmp/dsh-spike-home dsh plugin --profile spike add /tmp/rlm-plugin-spike                   # pnpm link, rc=0
DSH_HOME=/tmp/dsh-spike-home dsh --profile spike --dump-config                                       # patch 已生效
```
结果：① `dsh.profile.bundles` 自动并入 `@spike/dsh-rlm-probe`（bundle 声明自动发现机制生效）；② patch 的**行配置整份替换**生效（`llm-deepseek` 的 `defaultContextWindow: 123456`、`streamIdleTimeoutMs` 被替换——印证"替换整个 config 而非深合并"的语义）；③ **行禁用**生效（`persistent-pwsh: disabled: true`）——R7 的单工具模式缓解路径得到实证；④ dump 按行标注 patch 来源（`# == @deepseek-ai/dsh-sdk-minimal, patched by @spike/dsh-rlm-probe`）。唯一未测的是带 `src/index.ts` 代码包的构建装载（需 tsdown/tsconfig 基建，属 Stage 0 工作；其机制与仓库内 60+ 现有包相同，风险低）。
