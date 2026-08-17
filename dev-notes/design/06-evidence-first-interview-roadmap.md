---
title: "06 — Evidence-first Tau interview and development roadmap"
---

# Evidence-first Tau interview and development roadmap

## 结论先行

Tau 不应该被改造成 Pico V3 的 Python 复刻版。更合适的项目定位是：

> **一个 provider-neutral、可恢复、可分支、可压缩，并通过运行证据和分层评测持续验证的本地 coding-agent harness。**

Pico V3 最值得借鉴的不是功能清单，而是下面这条证据链：

```text
真实问题
  → 设计取舍
  → 明确的模块 seam
  → 固定任务合同 / 对照实验
  → trace、task state、report
  → 指标与失败分类
  → 有边界的面试结论
```

Tau 已经有比“基础 Agent 循环”更强的底座：provider-neutral loop、stateful harness、事件流、append-only JSONL session tree、resume、branch、compaction、skills、extensions 和 Textual adapter。当前真正缺的不是再堆一套 Agent 功能，而是把这些能力组织成**可验证、可复盘、可对比、可在面试里讲清楚的工程闭环**。

因此优先级应是：

1. 先完成评测合同和运行证据，让现有能力“可被证明”。
2. 再做 Tau 自己最有优势的恢复正确性实验，让 append-only session tree 形成独特叙事。
3. 然后做上下文成本 A/B；先埋点，后改策略。
4. 结构化长期记忆放到后面，不为了复刻 Pico 而提前引入。

## 调研基线

### Pico 材料中最值得迁移的方法

本路线参考了以下飞书材料：

- [关于 v3 及 v3 文档说明](https://icnoljnkix43.feishu.cn/wiki/RMf1wodbeiyAI1kUEFucwTxgnHh)
- [从 Novel Agent Harness 到 Pico：上下文压缩策略](https://icnoljnkix43.feishu.cn/wiki/KfLIwcxIdivWLkkCKvncaxg0neb)
- [08 — 评测框架与实验方法设计](https://icnoljnkix43.feishu.cn/wiki/AdIRwcIaFirI4gkd4NqcagKFnye)
- [90 — 针对项目描述的逐字面试话术](https://icnoljnkix43.feishu.cn/wiki/EYDzwj67Nilgb4kLdMZc3Z9RnPf)
- [0424 字节 & 腾讯一面，Agent 开发](https://icnoljnkix43.feishu.cn/wiki/M5Enw7jn3iwUvFkRmVBc7edVnx3)

提炼出的核心方法是：

- Harness 的价值不能只靠功能数说明，要回答“同一个模型为什么在这个 Harness 下更稳”。
- deterministic scripted baseline 先验证运行时合同；real provider 实验再验证模型参与后的效果。
- 一条 benchmark task 是合同，而不只是一句 prompt。它至少要有 fixture、允许工具、预算、目标产物和 verifier。
- “通过”是多个外部条件的交集，不能由模型自己说 `Done` 决定。
- 不同机制使用不同指标：上下文看成本和质量，恢复看错误恢复/误信率，安全看拦截，Harness regression 看合同稳定性。
- synthetic、estimated、actual、real-provider 数据必须分开标注；没有质量验证的数据只能叫方向性证据。

### 本地代码和学习进度

实际目录是：

- Pico：`/Users/leonard/codeRepo/pico`
- Pico V3：`/Users/leonard/codeRepo/pico-v3/pico`
- Tau：`/Users/leonard/codeRepo/tau`

当前 `/teach` 学习记录已经覆盖：

- message、provider、tool 和 event 协议
- simple turn、tool cycle、四类 loop 退出路径
- stateless loop / stateful harness 分工
- interrupted tool-call repair
- tool-safety seam
- provider → loop → harness → CodingSession → TUI 的本地事件链和背压
- append-only JSONL、replay、resume、branch、compaction
- harness message append 与 durable persistence 之间的 append-lag

所以后续不应回到“什么是 Agent loop”重新学习。下一个学习阶段应直接进入**评测、运行证据和恢复可信度**。

Tau 工作区还已有未提交的：

- `src/tau_coding/benchmark.py`
- `tests/test_benchmark.py`

它已经实现了最小 benchmark adapter：复制 fixture、工具白名单、步数预算、外部 verifier、`trace.jsonl` / `task_state.json` / `report.json`，以及 SWE-bench prediction JSONL 输出。当前验证结果是：

```text
pytest: 3 passed
ruff:   passed
mypy:   passed
```

这说明 Phase A 已经起步，但还不能据此声称 Tau 有正式 benchmark 成绩或 SWE-bench 分数。

## Tau 与 Pico V3 的能力映射

| 主题 | Tau 当前事实 | Pico V3 可借鉴点 | 决策 |
| --- | --- | --- | --- |
| Agent 核心 | `tau_ai` / `tau_agent` / `tau_coding` 分层，loop 与 harness 分开 | 用一条端到端运行链路组织讲解 | 不改核心定位，先把现有链路说清楚 |
| 会话持久化 | append-only JSONL、parent pointer、leaf、replay、branch、compaction | 区分恢复状态与复盘证据 | 把 Tau 的 session tree 作为主亮点 |
| 上下文 | token 估算、自动/溢出压缩、保留最近 20k、结构化增量摘要 | 分层预算、压力等级、成本/质量配对实验 | 先测现状，再决定是否加 section budget |
| 工具治理 | core 有 `before_tool_call` / `after_tool_call` seam；extension 可 block/rewrite；内置文件工具本身不限制绝对路径 | 风险分级、工作区策略、审计 | 策略留在 `tau_coding`/adapter，不塞进 `tau_agent` |
| 运行证据 | 有事件流、session JSONL、session export；没有统一的单次 run artifact 规范 | task state / trace / report 及聚合 | 作为第一优先级补齐 |
| 评测 | 已有最小未提交 adapter；缺 task suite、失败分类、聚合和可复现元数据 | contract → evaluator → metrics | 第一阶段完成成闭环 |
| 恢复可信度 | 能 replay/resume/repair，但没有 workspace freshness / runtime identity 判定 | checkpoint、漂移识别、false-trust 指标 | 第二阶段做成 Tau 的独特实验 |
| 长期记忆 | 当前 `SessionState` 是 replay state，不是语义长期记忆 | 分层记忆、freshness、ablation | 后置；避免把两个 “memory” 概念混在一起 |

## 面试定位

### 一句话版本

> 我在二次开发 Tau，一个 Python 的 Pi-style coding-agent harness。我的重点不是再包一层模型调用，而是利用 provider-neutral loop、事件流和 append-only session tree，把本地 code agent 做成可恢复、可分支、可压缩，并能通过固定 benchmark 和运行工件复盘的执行系统。

### 30 秒版本（当前可以真实使用）

> Tau 的核心分成 provider、portable harness 和 coding application 三层。我先沿真实代码把模型流式事件、工具执行、消息持久化、resume、branch 和 compaction 路径吃透，然后从评测 seam 开始二次开发。现在最小 adapter 已经能在隔离 fixture 中运行固定任务，用工具白名单、步数预算和外部 verifier 判定结果，并落 `trace`、`task_state`、`report` 三类工件。下一步不是直接宣称模型效果，而是先补任务合同、失败分类和复现元数据，再用 deterministic baseline 与真实 provider 实验分开验证。

### 90 秒版本（完成 Phase B 后使用）

> 我把这个项目定义成 Harness，而不只是 Agent，是因为模型只负责提议下一步，Harness 要负责它在什么边界内执行、状态怎么延续、失败怎么解释、什么条件下才算完成。Tau 的 `tau_agent` 保持 provider 和 UI 无关，`CodingSession` 在应用层负责本地工具、资源和 session 环境；所有前端消费同一事件流。它的 session 不是覆盖写聊天历史，而是 append-only tree：分支只移动 active leaf，压缩也不删除原始消息，而是在 replay 时用 `CompactionEntry` 替换上下文视图，所以历史仍可审计。我的二次开发在 `tau_coding` 增加 evidence-first evaluation：任务先被定义成 fixture、工具、预算、产物、verifier 的合同，运行后保留 trace 和 report，再按失败类型与机制指标聚合。这样我能区分模型波动、Harness 回归和恢复错误，而不是把所有东西压成一个成功率。

### 白板主图

```text
Provider stream
    ↓
run_agent_loop ── tool hooks ── coding tools
    ↓ events                 ↓ results
AgentHarness ───────── transcript state
    ↓ events                 ↓
CodingSession ───── append-only session tree
    ↓                         ├─ resume / branch
frontend adapters            └─ compaction replay

EvaluationRunner (tau_coding)
    ├─ isolated fixture + task contract
    ├─ deterministic / real-provider executor
    ├─ external verifier
    └─ trace + task_state + report → metrics
```

## 当前可以说与不能说的证据边界

### 已验证、可以说

- Tau 的三层架构和 UI/app policy-free core 是代码与路线图中的明确约束。
- loop、harness、events、tool hook、session tree、resume、branch、compaction 都有实际实现和测试，不是简历设想。
- compaction 不删除旧 entry；它通过 `replaces_entry_ids` 在 replay 时替换 active context。
- 当前最小 benchmark adapter 的 3 个测试、Ruff 和 Mypy 已通过。
- adapter 已能生成 SWE-bench 所需 prediction schema，但这只是互换格式能力。

### 暂时不能说

- 不能把 3 个单元测试说成“benchmark 100% 通过率”。
- 不能把 prediction JSONL 兼容说成“已跑通 SWE-bench”或给出 SWE-bench 分数。
- `expected_artifact` 当前只是字段，还没有进入 pass 判定，不能说已有完整四条件合同。
- 还没有 failure category、fixture snapshot、commit/model/decoding 元数据和聚合报告。
- 还没有 Tau 的上下文 A/B 成本结论。
- 还没有 workspace 漂移识别和 checkpoint false-trust 实验。
- 还没有结构化长期记忆，更不能复用 Pico 的“重复读取从 60 到 0”数字。

面试表达统一使用以下证据标签：

```text
implemented + tested     已实现且测试覆盖
deterministic evidence   固定模型输出下的运行时证据
estimated                启发式估算
actual                   provider 返回的真实 usage
real-provider result     真实模型重复实验
planned                  尚未实现，不写进简历成果
```

## 代码改造路线

### Phase A — 完成 Harness regression 合同（2–3 天）

目标：把现有 `benchmark.py` 从“能跑一个任务”变成最小、可信的回归层。

交付：

1. 用 typed model 定义并验证任务合同：`id`、`prompt`、`fixture_repo`、`allowed_tools`、`step_budget`、`expected_artifact`、`verifier`、`category`；安全类任务另带显式的 expected policy outcome，避免把“成功拦截”误判为任务失败。
2. 把 `expected_artifact_exists` 纳入 pass 条件。
3. 增加失败分类：`missing_artifact`、`budget_exceeded`、`verifier_failed`、`failure_stop_reason`、`unexpected_policy_decision`。
4. 写 6–8 个 deterministic regression tasks，覆盖：read/edit、原子 edit 失败、预算超限、工具白名单、工作区越界、interrupted tool repair、正常 stop。
5. artifact 带上 commit、branch、fixture snapshot hash、model、provider、task schema version 和运行时间。
6. 聚合 pass rate、budget rate、verifier rate、失败类别和平均工具步数。
7. 增加 `tau benchmark ...` 或等价独立入口；不要把 benchmark 分支塞进正常 TUI 主流程。

完成条件：同一 commit、同一 fixture、FakeProvider 下可重复得到相同 artifact；单条失败能定位到明确类别。

设计 seam：这层属于 `tau_coding`。它是应用策略，不进入 `tau_agent`。

评测要明确分成两条执行通道：

- `HarnessRegressionExecutor` 直接跑 `AgentHarness`，验证 provider/tool loop、hook、预算和事件合同。
- `CodingSessionExecutor` 跑完整 `CodingSession`，验证 persistence、resume、branch、compaction、skills 和 extensions。

只有这两个真实 adapter 都出现后，才值得提取共同的 executor interface。report 必须标明执行通道，不能用 direct-harness 结果证明完整 coding application。

### Phase B — 统一单次 Run Evidence（2–3 天）

目标：让正常 Tau run 和 benchmark run 使用同一份工件语义，而不是复制两套事件解析代码。

建议做一个深模块，接口保持小：

```python
recorder = RunRecorder(metadata=...)
recorder.observe(event)
evidence = recorder.finish(outcome=...)
```

模块内部负责：

- event → trace JSONL
- tool step / model turn / stop reason 统计
- provider usage、duration、compaction/retry 事件归一化
- `task_state.json` 和 `report.json`
- schema version 与原子写入

不要把 recorder 方法扩成每种事件一个接口；调用者只需要知道 `observe` 和 `finish`。测试也只跨这个 interface 验证 artifact。

完成条件：normal coding run 和 benchmark run 产出的共同字段语义一致；可以从一份 report 反查对应 trace 和 session。

### Phase C — 恢复可信度与 workspace drift（3–5 天）

目标：把 Tau 已有的 resume/branch/compaction 优势升级成“恢复正确性”亮点。

先不新增 `tau_agent` 核心 entry 类型。利用现有 `CustomEntry` seam，在 `tau_coding` 写入版本化 checkpoint：

```text
namespace: tau.checkpoint.v1
goal
progress
blockers
next_steps
key_files: path + digest
workspace_identity: cwd + git head + dirty fingerprint
runtime_identity: provider + model + policy
```

再设计一个纯 `ResumeAssessor`：

```python
assessment = assessor.assess(checkpoint, current_workspace, current_runtime)
# trusted | partially_stale | workspace_mismatch | incompatible | missing
```

恢复时永远重新发现当前 `AGENTS.md`、skills、tools 和 system prompt；checkpoint 只作为一段带可信度的短状态进入上下文，不能直接复用旧 prompt。

实验按风险而不是按功能数量设计：

- 未变化 checkpoint
- 单个关键文件变化
- 多个关键文件变化
- git HEAD / dirty state 变化
- cwd 变化
- checkpoint schema 不兼容
- provider/model/policy 变化
- 取消后的 dangling tool repair
- 工具报错但工作区已变化
- compaction 后 branch 到旧节点再恢复

核心指标：resume success rate、drift detection rate、recalibration rate、false-trust count。最重要的是 false-trust，而不是强行追求所有场景都恢复成功。

### Phase D — 上下文成本与质量成对实验（3–5 天）

目标：先证明 Tau 当前 compaction 策略的成本/质量，再考虑 Pico 式 section budget。

第一步只加观测：

- system / tools / messages / compaction summary 的估算 token
- provider actual usage（如果有）
- compaction call 自身 token
- overflow retry 与 task quality

第二步做 paired variants：

- `no_compaction`
- `tau_keep_recent_compaction`
- 可选 `section_budget_experiment`

scripted mode 用相同工具轨迹隔离 prompt 成本；real-provider mode 重复运行，用 verifier 保证质量。净收益按下面计算：

```text
baseline_input - optimized_input - compaction_call_total
```

只有净收益为正、verifier 无回归、没有 unknown pair 时，才把结果写成可认领结论。

不要直接照搬 Pico 的 50% 利用率和六 section 比例。Tau 当前的 system/tool/message 组织与 Pico 不同，必须先用自己的 trace 找压力来源。

### Phase E — 可选：workspace-anchored memory（后置）

只有 Phase A–D 形成闭环后再做。

第一版不需要向量库。用 path/tag/keyword/recency 和文件 digest 做可解释召回，并明确区分：

- `SessionState`：从 durable log replay 出来的当前会话状态
- `checkpoint`：恢复时必须评估可信度的短状态
- `working memory`：当前任务下一轮可能复用的信息
- `durable knowledge`：跨 session 的稳定项目事实

指标优先看重复读文件、额外工具步数、stale-memory rejection 和错误召回，不只看回答成功率。

## 后续学习路线

保持现有 `/teach` 节奏：每节约 15 分钟，先预测，再读关键代码，最后做一个 retrieval/exercise。不要把开发任务和学习任务拆成两套；每张卡都产出一个可测试的小增量或一段面试证据。

| Card | 核心问题 | 重点代码/产物 | 理解检查 |
| --- | --- | --- | --- |
| 18 | benchmark task 为什么是合同，不是一句 prompt？ | 当前 `benchmark.py`、Pico evaluator | 手写一条任务并指出所有不确定性 |
| 19 | deterministic baseline 能证明什么、不能证明什么？ | FakeProvider、scripted task | 区分 Harness 回归与模型能力 |
| 20 | “完成”应由哪些条件共同决定？ | artifact、budget、verifier、stop | 解释为什么模型说 Done 不够 |
| 21 | event stream 怎样变成 run evidence？ | `AgentEvent`、`CodingSessionEvent`、RunRecorder | 从一条 trace 还原一次工具调用 |
| 22 | 怎样设计 failure taxonomy？ | report aggregator | 给 5 个失败样本分类，说明优先级 |
| 23 | 怎样保证 benchmark 可复现？ | fixture hash、git/model metadata | 解释相同 pass rate 为什么仍可能不可比 |
| 24 | session replay 与 checkpoint resume 有什么不同？ | entries、memory.py、CustomEntry | 区分 durable truth、active view、resume hint |
| 25 | workspace 漂移为什么比恢复失败更危险？ | ResumeAssessor | 解释 false-trust 并设计两个反例 |
| 26 | compaction 怎么设计 paired A/B？ | context_window、report usage | 计算包含 compact call 的净收益 |
| 27 | 什么数据能写进简历？ | evidence registry | 把 estimated、synthetic、actual 分开表达 |

每完成一张卡，学习记录增加四项：

1. 我现在能解释的机制。
2. 我能指出的代码 seam。
3. 我亲自跑过的测试或 artifact。
4. 我不能声称的边界。

## 面试准备清单

### 必须能讲清的十个问题

1. 为什么叫 Harness，而不是 Agent framework？
2. 同一个模型在不同 Harness 下为什么表现不同？
3. `run_agent_loop` 与 `AgentHarness` 为什么分开？
4. 一次输入怎样经过 provider、tool、event、session 和 frontend？
5. 为什么 session 用 append-only tree，而不是覆盖写一份 JSON？
6. branch 与 compaction 的本质区别是什么？
7. 中断工具调用为什么需要 protocol compensation？
8. benchmark 如何区分 runtime bug、模型波动和 verifier bug？
9. 为什么恢复错比恢复失败危险，怎么测 false-trust？
10. 上下文优化为什么必须同时看成本和质量？

### 每个回答都用同一结构

```text
问题场景
→ 设计目标
→ Tau 的 seam 和实现
→ 关键 trade-off
→ 测试 / artifact / 指标
→ 当前边界和下一步
```

### 面试官深挖时的关键 trade-off

- **core policy-free vs 默认安全**：tool hook 在 core，具体 workspace/approval policy 在 coding app；可移植性更高，但应用层必须真的装配策略。
- **append-only audit vs 存储膨胀**：保留原始 entry 便于 branch/replay/审计，代价是磁盘增长和 replay 成本，需要索引或 snapshot 演进。
- **完整历史 vs compaction**：完整历史保真但贵；摘要省 token 但可能丢信息，因此需要 recent protection、结构化摘要和 quality verifier。
- **checkpoint vs history**：checkpoint 便于继续工作，但可能过期；history 可审计但噪音大。两者不能互相替代。
- **scripted vs real provider**：scripted 可复现、能隔离 Harness；real provider 能测真实质量但有随机性和成本。结论必须标数据来源。
- **一个总分 vs 分层指标**：总分容易比较但定位差；机制指标更可解释，但要防止挑指标讲故事。

## 建议的简历项目描述

### 当前版本（只写已验证事实）

> **Tau 本地 Coding Agent Harness 二次开发**：基于 Python/AnyIO/Pydantic 的 provider-neutral agent loop、事件流和 append-only session tree，完成模型流式输出、工具执行、resume/branch/compaction 机制的源码级学习与验证；在 `tau_coding` 侧实现最小 benchmark adapter，支持隔离 fixture、工具白名单、步数预算、外部 verifier、运行 trace/report 落盘和 SWE-bench prediction 格式导出，并以 FakeProvider 测试保证执行链路可复现。

### Phase C 完成后的目标版本（必须替换成真实实验数字）

> **可恢复、可评测的本地 Coding Agent Harness**：在 Tau 的 provider-neutral loop 与 append-only session tree 上构建 evidence-first evaluation 和恢复可信度机制；将任务定义为 fixture、工具、预算、产物、verifier 的显式合同，统一产出 trace/task state/report，并通过 deterministic regression 与真实 provider 对照区分 Harness 回归和模型波动；基于文件摘要和 workspace/runtime identity 评估 checkpoint 可信度，在 **[N]** 类恢复风险场景中实现漂移识别率 **[X%]**、false-trust **[Y]**。

方括号只能由真实 artifact 填充，不能预写 Pico 的数字。

## 第一周执行顺序

### Day 1

- 把当前 benchmark adapter 拆成 task contract、runner、artifact 三个内部职责。
- 写清 pass 条件和失败分类，先补测试再改实现。
- 产出 Card 18–20 的 retrieval answer。

### Day 2

- 增加 6–8 个 deterministic regression tasks 和 fixture snapshot。
- 生成第一份可重复 benchmark artifact。
- 写出 30 秒和 90 秒项目讲解的录音稿。

### Day 3

- 实现 benchmark aggregation 和最小命令入口。
- 从一个失败 task 反向定位到 trace 事件，形成调试案例。
- 完成 Card 21–23。

### Day 4

- 设计 `RunRecorder` interface，让 normal run 与 benchmark 共用字段语义。
- 用 fake events 做 artifact contract tests。
- 画一次白板主图并口述十个必问题中的前六个。

### Day 5

- 写 ResumeAssessor 的 ADR 和风险矩阵，暂不急着改 session core。
- 用 `CustomEntry` 做一个最小 checkpoint prototype。
- 完成 Card 24–25，更新简历描述中的“当前版本”，不填尚未产生的数字。

## 暂不做

- 不把 Pico V3 的目录和类名搬到 Tau。
- 不先做多 Agent、向量库或复杂 Memory UI 来制造功能感。
- 不把评测逻辑放进 `tau_agent`。
- 不同时重写 context、memory、checkpoint 和 tool policy。
- 不用单元测试数量包装 benchmark 指标。
- 不用 synthetic/estimated 数据冒充真实 provider 结论。
- 不为了漂亮数字复制 Pico 的比例、任务数或实验结果。

## 成功标准

这轮二次开发完成时，应该同时具备四种证据：

1. **代码证据**：每个机制能指到明确 interface、implementation 和 seam。
2. **测试证据**：FakeProvider 和 fake tools 下可重复验证运行时合同。
3. **运行证据**：每次 run 有可追溯的 trace、task state 和 report。
4. **实验事实**：真实 provider 与 deterministic baseline 分开，指标有分母、有失败样本、有不可声明边界。

项目最有说服力的状态不是“功能和 Pico 一样多”，而是：面试官问任何一个数字或设计，都能从结论一路追到 task contract、代码 seam、测试和 artifact。
