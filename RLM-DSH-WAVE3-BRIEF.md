# RLM → dsh Stage 4 共享 Brief（全能力对齐）

先完整读 `/Users/bytedance/codes/open-source/prime-agent/RLM-DSH-STAGE2-HANDOFF.md`（尤其 §3 权威参考、§7 边界、§9 环境坑），再动手。

## 仓库

- 工作仓库（改动处）：`/Users/bytedance/codes/open-source/deepseek-harness`（下称 dsh），分支 main。
- 参考仓库（只读）：`/Users/bytedance/codes/open-source/prime-agent`（下称 PA）。
- 已建成的 RLM 包在 `dsh/packages/rlm/`：`rlm-kernel`（`ctx.rlmKernel` seam，含 `registerHostRequestHandlers`）、`rlm-kernel-python`（CPython provider，`py/rlm/` 运行时）、`tool-python`、`rlm-harness`（`ctx.rlmHarness`）、`rlm-bindings`（Stage 2/3 的 9 条 host wire）。

## 铁律

1. **只新建自己名下的文件**，不改任何共享文件：`rlm-bindings/src/index.ts`、`package.json`、`tsconfig*.json`、`cordis.patch.yml`、`locale/`、`README*`、`docs/`、`packages/preset/**`。需要新依赖时用**局部结构化 interface**（结构子集 typing），不 import 新包；确实绕不过的记在回执里，集成阶段统一处理。
2. 工厂模式：每个适配层导出 `create*HostHandlers(deps): HostRequestHandlers` 形态，与 `rlm-bindings/src/subagents.ts` 的 `createRlmHostHandlers` 对齐；wire handler 签名看 `rlm-kernel/src/index.ts` 的 `HostRequestHandler`。
3. 禁 `pnpm run`（会触发 install 检查失败）。命令一律：
   - 测试：`node node_modules/vitest/vitest.mjs run <path>`（在 dsh 根目录跑）
   - 覆盖率：加 `--coverage --coverage.reporter=json --coverage.reportsDirectory=/tmp/rlm-cov-<你的名字>`，用 `node -e` 解析 `coverage-final.json` 只筛自己文件，**逐文件 100%**
   - lint：`node --import tsx/esm scripts/run-oxlint.ts <path>`
   - 类型：`node node_modules/typescript/bin/tsc -b packages/rlm/rlm-bindings`（src 会被 include，必须 clean；测试文件不在 tsconfig 里，由 oxlint/tsgolint 管）
4. `exactOptionalPropertyTypes` + `noUncheckedIndexedAccess`：条件展开 `...(x === undefined ? {} : { x })`；**禁 `as unknown`**；schemastery 无 `.parse()`。
5. 禁 `git add -A` / `reset --hard` / `checkout .` / `clean -fd` / `stash`。**不要 commit**，改动留工作区。
6. 测试写法参照 `rlm-bindings/tests/` 现有 spec（fake/纯单测惯例，faux provider 见 `packages/agent-loop-testkit`）。
7. 回执（`agent_message.send` 给 parent）：建了哪些文件、每条 wire 的语义决策、测试/覆盖率/lint/tsc 结果、偏差与待集成事项（需要哪些 inject/依赖）。中文、简洁、无表情。

## PA 侧权威参考（handler 语义逐条对齐，不要重新发明）

| wire | PA 实现 | Python 侧契约 |
|---|---|---|
| `goal.get/create/complete` | `packages/coding-agent/src/core/agent-session.ts:3670` `handleGoalHostRequest` | PA skill 源：`~/.local/share/prime-agent/releases/0.9.6-darwin-arm64-97aff54310fde0c7a1d2a596a362db526f553395348a81696ec12c7c339052e1/skills/goal/src/` |
| `compact.run/status` | `agent-session.ts:3698` | 同目录 `compact/src/` |
| `refine.run/status` | `agent-session.ts:3752` + `core/refinement/refinement.ts` | 同目录 `refine/src/` |
| `rlm_heartbeat.*` | `agent-session.ts:3817` | 同目录 `rlm-heartbeat/src/` |
| `agent_message.*` | `agent-session.ts:3906` + `core/agent-messages.ts:530` `createAgentMessageHostHandlers` | 同目录 `agent-message/src/` |
| `agent_observe.*` | `agent-session.ts:3931` + `core/agent-observe.ts:77` `createAgentObserveHostHandlers` | 同目录 `agent-observe/src/` |
| `mcp.config/refresh/begin_login` | `core/mcp/mcp-manager.ts:181-220` `hostHandlers()` | dsh `packages/rlm/rlm-kernel-python/py/rlm/mcp.py:501,538`、`mcp_base.py:176` |
| `model.info` | `agent-session.ts` handler map 里内联（`:10418` 附近） | PA Python 不发，dsh 也不必发；纯 host 补齐 |

## dsh 侧可复用 seam（已核实）

- `ctx.goals`（`GoalService`，`packages/goal/goal/src/index.ts:240`）：`get/create/edit/pause/resume/complete/block/clear/disarm`。
- `ctx.compaction`（`CompactionEngine` 抽象，`packages/compaction/compaction/src/index.ts:119`）：`compactIfNeeded/compactNow/compactRegion`；无 status 查询。
- `ctx.subagents` + `ctx.sessionQuery`：rlm-bindings 已注入并用过（`src/subagents.ts`、`src/read.ts`）。
- steer 路线：`bash.ts` 的 `BashNoticeBoard` / Stage 3 的 `agent.steer(createUserMessage)` 先例。
- schedule：`packages/schedule/schedule/src/runtime.ts` `ScheduleRuntime`（注意：未暴露为 ctx service，`inject = ['agents','sessions','tools','sessionPersistence']`）。
- mcp-client（`packages/mcp/mcp-client`）：静态 per-server 配置插件，无 auth/oauth 代码。
