# RLM → dsh 插件化：Stage 4 总结（全能力对齐）

状态日期：2026-09-27。前置：RLM-DSH-STAGE2-HANDOFF.md（Stage 0/1/2/3）。全部改动未 commit。

## 一句话现状

prime-agent RLM host handler map 的全部缺口（goal.*/compact.*/model.info/mcp.*/agent_message.*/agent_observe.*/rlm_heartbeat.*/refine.*）已在 dsh `rlm-bindings` 落地：**28 条 handler**（9 旧 + 19 新），283 tests 全过、15 个 src 文件逐文件 100% 覆盖、§8.1 全部门禁 EXIT=0、真实 CPython 内核 e2e smoke 8/8。另有 `apps/cli/config/examples/rlm-full/cordis.yml` overlay 一条 `--patch` 启用全部能力插件。

## 新文件（packages/rlm/rlm-bindings/）

src: goal.ts, compact.ts, model-info.ts, mcp.ts, refine.ts, message.ts, observe.ts, heartbeat.ts + 8 个 spec。index.ts 重写装配。

## 关键架构决策

1. **compaction 不 inject**：Web 组合把引擎挂在 agent 隔离 preset 里，宿主 inject 会永久 pending。改为请求时解析：live agent → `live.ctx.get('compaction')` → 回落 `ctx.agentPresets.serviceFor(live, 'compaction')`（新增 dep dsh-agent-preset-registry）。
2. **inject 最终**：rlmKernel, subagents, llm, sessionQuery, agents, goals, tokenMeter。
3. **refine 无独立 planner**：turn-stopping 消费 pending → steer refinement notice → agent 自精炼（设计笔记 outputs/refine-dsh-design.md）。
4. **heartbeat 自建**：ScheduleRuntime 字段缺一半（无 pause/cron/label/delivery_mode）且非 ctx service，故 HeartbeatStore(JSON 原子写, <dshHome>/rlm/heartbeats.json) + HeartbeatScheduler(单 timer)。
5. **mcp 安全默认**：createMcpHostHandlers({}) 三 dep 全省略——config 走 Python KeyError、refresh 恒抛 PA 原文、begin_login 不注册。

## Known Limitations（已写进 README，汇总）

mcp.begin_login 无 OAuth 底座不注册；mcp 无凭证存储；goal token_budget 只校验不执行（dsh 按轮数预算）、无 token/time 记账；compact 不透传 customInstructions、无预检、不自动 resume；refine 无独立 planner、pending 进程内易失；heartbeat 私有 JSON、重启 overdue 立即补拍；agent_message target 必须 live（无 cold resume）；顶层 siblings 只含 live roots；rlm-harness 只有抽象 seam 无具体 provider（read/refine/rollback 缺实现，待后续 stage）；Python 侧 mcp list_tools/call_tool 需真实配置源（dsh 无用户级 mcpServers settings）。

## 遗留

1. compact.run/refine.run 的 scheduled:true 路径只有单测与 index.spec 的 ctx.emit 覆盖，smoke 走的是拒绝路径。
2. heartbeat 到点投递只有单测覆盖。
3. pnpm-lock.yaml 4 条 devDep link 系手工补写（禁网），下次联网 `pnpm install` 复核。
4. message.ts/bash.ts 的 'rlm-bindings' MessageSourceMap augmentation 重复声明未收敛（tsc 兼容）。

## 过程资产

并行生产方式与坑见 prime-agent/RLM-DSH-WAVE3-BRIEF.md（worker 铁律：只新建名下文件、结构化 deps interface、逐文件 100% 覆盖）。
