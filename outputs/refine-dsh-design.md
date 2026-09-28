# refine.run / refine.status dsh host wires — 设计笔记（C-refine）

## 结论（TL;DR）

- dsh **有**回合结束钩子：`agent/turn-stopping`（`packages/core/agent/src/runtime-types.ts:381`，serial 事件，
  在 turn 边界提交前被 await；listener 调 `agent.steer(...)` 可让机器重读 inbox、turn 不关闭继续跑）。
  这正是 PA `_consumePendingRequestedRefine`（agent-session.ts:8623，在 agent_end / shouldStopAfterTurn 边界
  消费 `_pendingRequestedRefine`）的对应物。
- dsh **没有**精炼规划器：PA 的 `core/refinement/refinement.ts` 是一次独立 LLM pass（序列化轨迹 →
  精炼 system prompt → 产出 edits → apply），dsh 侧 `ctx.rlmHarness.refine(proposal, scope)` 只接受
  已成形的 proposal（`packages/rlm/rlm-harness/src/index.ts` `HarnessRefiner`），没有"从轨迹生成 proposal"的组件。
- 因此机制决策：**refine.run 记录 per-agent pending（进程内），turn-stopping 处消费并以 steer 方式把
  "精炼请求"作为 refinement notice 注入会话，由 agent 自己在下一步里经 harness CRUD 完成精炼**。
  语义对齐点：scheduled 立即返回；精炼发生在 turn 边界、绝不 mid-cell；agent 自动 resume（steer 使
  turn 继续）；重复调用只合并 instructions/global。做不到的机制见末节"偏差"。

## PA 侧契约（逐字来源）

- Python skill：`skills/refine/src/refine/__init__.py`
  - `refine.run(instructions=None, global_=False)` → `host_request("refine.run", payload)`；
    payload 仅在非 None/True 时带 `instructions`/`global`。
  - `refine.status()` → `host_request("refine.status")`，返回 `{pending, in_flight}`。
  - Python 侧对该 reply 无强校验（通用 bridge），dict 原样回给模型。
- Host：`agent-session.ts:3752` `handleRefineHostRequest`
  - 校验错误文案（逐字保留）：
    - `refine.run instructions must be a string when provided`
    - `refine.run global must be a boolean when provided`
  - 无活跃 turn：`{scheduled: false, reason: "no active turn; refine can only be requested while a turn is running"}`
  - 合并语义：`{instructions: instructions ?? previous?.instructions, global: globalFlag ?? previous?.global}`
  - 成功：`{scheduled: true, note}`；note 原文见下（dsh 改写了中句，见偏差）。

## dsh 映射

| wire | dsh 行为 | reply |
|---|---|---|
| `refine.run` | 校验 payload（PA 文案）；`agent.status !== 'running'` → scheduled:false（PA 的 `isStreaming` 对应 `AgentStatus`，`runtime-types.ts:172`）；否则 `RefineRequests.schedule(agentId, update)` 合并写入 | `{scheduled: true, note}` |
| `refine.status` | 查 `RefineRequests` | `{pending: bool, in_flight: bool}` |
| （边界消费） | `createRefineTurnStopping` listener：有 pending → consume（置 inFlight）+ `agent.steer(createRefineRequestMessage(...))`；无 pending → settle（清 inFlight） | — |

### in_flight 语义

PA：规划或应用进行中。dsh：pending 在 turn-stopping 被消费、steer 已注入，到下一次 turn-stopping
（精炼 step 跑完、turn 再次要关闭）之间为 true；下一次 turn-stopping 无 pending 时清掉。
turn 异常结束时 inFlight 可能滞留到下一个 turn 边界才清（README/receipt 列明）。

### steer 消息

- source kind 复用 `rlm-bindings`（bash.ts 已注册同名 MessageSourceMap 增强；refine.ts 重复一份同型增强，
  保证单文件被独立 import 进测试 program 时也能类型检查；interface merge 要求同型，逐字相同即合法）。
- 文本：`[refine-requested scope:local|global]` 头 + 一段说明（本 host 无独立规划器，请用 `rlm.harness`
  自做精炼、聚焦小编辑）+ 可选 `Focus: <instructions>`。

## 文件

- `src/refine.ts`：`RefineRequests`（per-agent pending/inFlight 状态类，进程内易失）、
  `formatRefineRequestNotice` / `createRefineRequestMessage`（纯函数）、
  `createRefineHostHandlers(deps): RlmHostRequestHandlers`（两条 wire）、
  `createRefineTurnStopping(deps)`（边界 listener 工厂）。
- `tests/refine.spec.ts`：纯单测 + fake agent（参照 subagents.spec.ts 惯例）。
- 注意：payload 读取助手放进 refine.ts 内部（read.ts 是共享文件，本轮不许改）。

## 集成时待办（本轮不做，index.ts 共享文件由集成阶段改）

1. `apply()` 里 `new RefineRequests()` 一份，把 `createRefineHostHandlers({requests})` 合并进
   `registerHostRequestHandlers` 的 map。
2. `ctx.on('agent/turn-stopping', createRefineTurnStopping({requests}))`（agent-scoped serial 事件）。
3. `ctx.on('agent/disposed', ({agent}) => requests.forget(String(agent.id)))` 防泄漏。

## 偏差（做不到/改写的机制）

1. **无独立精炼规划器**：PA 是旁路 LLM pass 产 edits 并 apply；dsh 退化为 steer 请求、由 agent 自行精炼。
   结果语义（harness 被精炼、agent 自动 resume）保持，产出质量依赖模型自觉。
2. **note 文案改写中句**：PA 原文 "applied edits are appended to your context as a refinement notice"
   与 dsh 机制不符；dsh 版改为 "the request is then steered into your context as a refinement notice"。
   首尾句（turn 结束才跑 / 自动 resume / 继续正常工作）保留。
3. **in_flight 粒度**：无 plan/apply 阶段区分；turn 异常结束时可能滞留到下一边界（见上）。
4. **global scope**：仅作为 steer 文本中的 scope 指令传递，由 agent 在调用 harness CRUD 时选择 global store；
   host 侧不强制执行（PA 是 planner 直接写 global store）。
5. 进程内状态：重启即丢（与 roster 同策略）。
