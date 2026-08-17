# Pico / Pico v3 崩溃恢复源码取证与面试准备

> 研究范围：只读飞书 Pico 设计/面试文档，以及本地 `/Users/leonard/codeRepo/pico` 与 `/Users/leonard/codeRepo/pico-v3/pico` 的源码、测试和 release 学习文档。所有行号以本次读取时的工作树为准。
>
> 快照信息：`pico` 分支 `main`，HEAD `f54ad9c`（`fix: add project provider config and deepseek support`；比本地 `origin/main` 落后 13 个提交，工作树有用户改动）；`pico-v3/pico` 分支 `v3`，HEAD `8361861`（`release: sync v3 public source`，与本地 `origin/v3` 一致，工作树在测试前干净）。两者都包含共同祖先/相关实现提交 `295e4b7`（`!1 memory: add durable memory, checkpointed resume, and regression coverage`）。

## 先给结论

两个版本都实现了“会话级恢复（session continuity）”，但不能把它们描述成已经达到 exactly-once 的“进程级崩溃恢复”。共同的核心是：把会话历史、工作记忆、checkpoint 和运行证据落盘；重启后从 session JSON 重建一个新的 runtime，把 checkpoint 摘要放回下一次 prompt。

两者都没有把正在运行的 Python 协程、provider 请求、工具进程或工具副作用做成可重放的事务。工具的安全处理是工具返回后做工作区前后快照和 diff；如果进程死在工具执行中，通常只能看到一个已经落盘的 `tool_started`/较早 checkpoint，而没有工具完成记录，不能安全地自动重跑风险工具。

`pico-v3` 比原始 `pico` 更完整：session JSON 用临时文件 + `os.replace()`，另有 session-level JSONL event bus；`Runtime` 持有状态、`Engine` 推进 turn；工具事件、span、artifact graph、verifier suggestion、provider error 和协作式 abort 都有更细的证据。但 v3 自己的学习文档明确写了：中途持久化仍没有 Claude Code 的 transcript-first 那么细，run artifacts 与 session transcript 的恢复语义还应该继续打通。

因此面试应说“恢复到最后一个 durable checkpoint，并识别/保守处理未完成动作”，不要说“从崩溃点无损续跑”或“工具 exactly-once”。

## 0. 飞书文档怎样定义和包装“恢复”

飞书材料有两个层次，不能只读其中一篇就把口径混在一起：

1. [《07-会话状态、运行工件与恢复机制设计》](https://icnoljnkix43.feishu.cn/wiki/Ge7lwrznSiSj90ksOpic9FSpn1e)先建立基础分层：session 回答“以后还能不能继续干”，run artifacts 回答“刚才到底发生了什么”。session 放在 `.pico/sessions/<session_id>.json`；每次 ask 的 `.pico/runs/<run_id>/` 里有 `task_state.json`、`trace.jsonl`、`report.json`。`--resume <session_id|latest>` 加载旧状态、重新创建带当前 model/workspace 的 runtime；文档明确说恢复的是 session state，不是原进程。
2. [《00-全局总览》](https://icnoljnkix43.feishu.cn/wiki/L7FiwIhcEiGQE8kiHJJcEXa7nrg)和[《90-针对项目描述的逐字面试话术》](https://icnoljnkix43.feishu.cn/wiki/EYDzwj67Nilgb4kLdMZc3Z9RnPf)把它升级为“可信恢复”：working memory 用于当前推理，durable memory 用于跨轮复用，checkpoint 是恢复真相，完整 history 主要用于审计与解释。恢复时优先投影 checkpoint + working memory，并检查 freshness/runtime identity，而不是把整段旧历史重新塞回模型。

面试文档最值得借鉴的不是名词，而是风险定义：**最危险的失败不是“恢复不了”，而是“接受了已经过期的状态并继续写”。** 因此 `evaluate_resume_state()` 不只返回成功/失败，而区分 `full-valid`、`partial-stale`、`workspace-mismatch`、`schema-mismatch`、`no-checkpoint`。checkpoint 只保存最小、可校验的信息：目标、完成项、阻塞点、下一步、关键文件、freshness anchors 和 runtime identity。

文档把恢复评测拆成 5 类、每类 2 个场景：

| 场景 | 两个代表用例 | 想证明什么 |
|---|---|---|
| 基础 checkpoint | 恢复 goal；恢复 key files | 新 runtime 能找回任务方向 |
| 局部过期 | 单文件 stale；多文件 stale | 能重新锚定，而不是盲信摘要 |
| workspace drift | workspace fingerprint；runtime 条件变化 | 环境变了会降级恢复 |
| schema 不兼容 | 旧 schema；无 checkpoint | 不会把不可解释状态判成有效 |
| 工具部分成功 | shell 改动后失败；普通工具改动后失败 | 能看到副作用，先检查再重试 |

飞书话术宣称的验证口径是 workspace drift 识别率 100%、过期状态 false accept 为 0。这里要加上源码事实的限定：这是 scripted recovery ablation 的结果，不是 kill -9、断电或外部 API exactly-once 的证明。

[《当前 pico 版本关系》](https://icnoljnkix43.feishu.cn/wiki/FE2xwXm0YiZTlDkInx8cWBannro)把 main 定位为稳定学习/使用版，把 v3 定位为二三面架构设计的本地 Agent Runtime 参考；[《关于 v3 及 v3 文档说明》](https://icnoljnkix43.feishu.cn/wiki/RMf1wodbeiyAI1kUEFucwTxgnHh)也强调面试重点是模块设计、取舍和 benchmark 证据，不是背代码行号。这个定位与本地源码吻合：v3 是恢复工程边界的加固，不是把语义 checkpoint 变成了进程快照。

## 1. 原始 pico：怎样持久化和恢复

### 1.1 持久化分成 session 与 run 两层

原始 `SessionStore` 只有一个 session JSON 文件：`path()` 指向 `<session_id>.json`，`save()` 直接把完整 session JSON 写回，`load()` 读回，`latest()` 按 mtime 选最近 session。实现见 `/Users/leonard/codeRepo/pico/pico/runtime.py:66-84`。`Pico.from_session()` 在 `/Users/leonard/codeRepo/pico/pico/runtime.py:153-161` 只是加载这个 JSON 并构造一个新的 `Pico`。

每次 history 发生变化，`Pico.record()` 追加 item 后立即调用 `SessionStore.save()`（`/Users/leonard/codeRepo/pico/pico/runtime.py:440-446`）。`ask()` 一开始先写用户消息（`/Users/leonard/codeRepo/pico/pico/runtime.py:776-783`），所以正常情况下 provider 请求开始前，用户输入已经进入 session JSON；但这不是流式 transcript，也不是事务日志。

单次请求由 `RunStore` 保存审计工件，与长期 session 分开：

```text
.pico/runs/<run_id>/
  task_state.json
  trace.jsonl
  report.json
```

`task_state.json` 和 `report.json` 使用临时文件写完再 `replace`，`trace.jsonl` 逐条追加；实现见 `/Users/leonard/codeRepo/pico/pico/run_store.py:19-70`、`/Users/leonard/codeRepo/pico/pico/run_store.py:98-112`。README 也把这三种文件列为运行产物（`/Users/leonard/codeRepo/pico/README.md:166-180`）。注意：原始 session JSON 自己仍然是直接覆盖写，不享受 RunStore 的原子写保护。

### 1.2 checkpoint 的内容与恢复判定

初始化时 `_ensure_session_shape()` 给 session 增加 `checkpoints.current_id/items`、`runtime_identity`、`resume_state`（`/Users/leonard/codeRepo/pico/pico/runtime.py:163-177`）。checkpoint 不是完整内存快照，而是一份可读的任务锚点：

- `checkpoint_id` / `parent_checkpoint_id`
- `schema_version`（当前 `phase1-v1`）
- `current_goal`、`completed`、`current_blocker`、`next_step`
- 最近关键文件及其 freshness
- 当前 runtime identity（cwd、model、审批、步数、feature flags、workspace fingerprint、tool signature）

字段由 `/Users/leonard/codeRepo/pico/pico/runtime.py:601-631` 生成并保存；每次工具执行返回后（无论成功、普通错误或 partial success）、上下文压缩、freshness/workspace mismatch、最终完成或 stop 都会创建 checkpoint。工具路径的调用顺序是“执行工具 -> 写 history/task_state/trace -> 建 checkpoint”，见 `/Users/leonard/codeRepo/pico/pico/runtime.py:897-935`；final 和受限 stop 的收口见 `/Users/leonard/codeRepo/pico/pico/runtime.py:942-998`。

重新启动时 `evaluate_resume_state()` 会：

1. 让工作记忆里的文件摘要失效检查先跑一遍；
2. 校验 checkpoint schema；
3. 用文件 freshness 对比关键文件是否被外部改动；
4. 用 runtime identity 对比 cwd、model、审批模式、read-only、预算、feature flags、环境变量白名单、workspace fingerprint 和 tool signature；
5. 得到 `no-checkpoint`、`full-valid`、`partial-stale`、`workspace-mismatch` 或 `schema-mismatch`。

对应源码是 `/Users/leonard/codeRepo/pico/pico/runtime.py:206-271`。恢复 prompt 不是只依赖 history，而是把 `Resume status`、当前目标、卡点、下一步、关键文件和 stale paths 渲染成 `Task checkpoint:`，见 `/Users/leonard/codeRepo/pico/pico/runtime.py:273-295`。

CLI 通过 `--resume <id|latest>` 选择 session，再走 `Pico.from_session()`（`/Users/leonard/codeRepo/pico/pico/cli.py:223-241`、`/Users/leonard/codeRepo/pico/pico/cli.py:253-271`）。这是“加载旧 session，接着发一个新 user request”，不是“把旧的 in-flight turn 自动重放”。

### 1.3 tool call 出错、部分成功和副作用

风险工具在执行前后做 workspace snapshot，完成后计算 `affected_paths`、`diff_summary` 和 `workspace_changed`。`run_shell` 非零退出且工作区已经改变时被标记为 `partial_success`；无改动的失败标记为 `error`；异常路径也会做一次 after snapshot，并记录 `tool_partial_success` 或 `tool_failed`。实现见 `/Users/leonard/codeRepo/pico/pico/runtime.py:1078-1127`。

这类普通工具异常不会让整个 turn 直接崩掉，而是返回 `error: ...` 文本给模型，并在 memory 里写“检查 diff 后再 retry”的 process note（`/Users/leonard/codeRepo/pico/pico/runtime.py:676-693`）。风险工具的历史、task state、trace 和 checkpoint 会在工具函数返回后写入，见 `/Users/leonard/codeRepo/pico/pico/runtime.py:897-935`。

但它不是回滚或 exactly-once：原始工具本身直接写文件/执行 shell（`/Users/leonard/codeRepo/pico/pico/tools.py:207-258`），没有操作 ID、意图日志、幂等键或撤销记录。若进程在 `tool.run()` 中间被 kill，after snapshot、history、trace 和 checkpoint 都可能来不及写；下次 resume 也没有代码根据“未完成 tool”决定是否重放。原始 CLI 在 agent 运行期间只捕获 `RuntimeError`，输入阶段才捕获 `KeyboardInterrupt`（`/Users/leonard/codeRepo/pico/pico/cli.py:294-313`、`/Users/leonard/codeRepo/pico/pico/cli.py:333-337`），因此不能把 Ctrl-C/kill -9 说成已实现的崩溃恢复。

### 1.4 原始 pico 的关键测试与评测

源码测试覆盖的是“恢复合同”和“正常异常路径”，不是实际 kill -9 故障注入：

- session history 能保存并用 `from_session()` 继续：`/Users/leonard/codeRepo/pico/tests/test_pico.py:194-207`。
- checkpoint 在 context reduction 后生成，task state/report 只保存 checkpoint id，完整 checkpoint 留在 session：`/Users/leonard/codeRepo/pico/tests/test_pico.py:981-1026`。
- prompt 使用 checkpoint 的 goal/blocker/next step，而不只是 history：`/Users/leonard/codeRepo/pico/tests/test_pico.py:1029-1067`。
- 外部改文件后，旧摘要失效，恢复状态为 `partial-stale`：`/Users/leonard/codeRepo/pico/tests/test_pico.py:1070-1111`。
- shell 非零退出但改了文件会被标为 `partial_success`，包含具体 changed path：`/Users/leonard/codeRepo/pico/tests/test_pico.py:1114-1128`。
- workspace runtime identity 变化、schema 不兼容、无 checkpoint 都有单测：`/Users/leonard/codeRepo/pico/tests/test_pico.py:1131-1244`。
- freshness mismatch 会在下一次 model completion 前先创建新的 checkpoint：`/Users/leonard/codeRepo/pico/tests/test_pico.py:1247-1285`。
- RunStore 专门有“有 run_started/trace 但没有最终 report”的测试，说明 report 是最后收口工件而不是恢复依据：`/Users/leonard/codeRepo/pico/tests/test_run_store.py:58-66`。
- recovery ablation 用固定模型检查 checkpoint、stale、workspace mismatch、schema mismatch 和 partial-success 场景，并比较 `resume_enabled`/`resume_disabled`：`/Users/leonard/codeRepo/pico/pico/metrics.py:1287-1348`、`/Users/leonard/codeRepo/pico/pico/metrics.py:1516-1561`、`/Users/leonard/codeRepo/pico/pico/metrics.py:1592-1612`。

本次本地验证（使用已有 venv，`UV_CACHE_DIR=/private/tmp/pico-uv-cache uv run --no-sync pytest ...`）结果为 `15 passed, 47 deselected`；pytest 仅报告因为受限环境无法写 `.pytest_cache` 的 warning。

另外实际执行了 main 的 recovery ablation（10 个 scripted 场景，1 次重复）：`resume_enabled` 的 `resume_success_rate=0.9`、`stale_reanchor_rate=1.0`、`workspace_drift_detection_rate=1.0`、`resume_false_accept_rate=0.0`；`resume_disabled` 除 false accept 外均为 0。`0.9` 而不是 `1.0` 的原因是“无 checkpoint”场景被有意判定为不可恢复。

## 2. pico-v3：怎样持久化和恢复

### 2.1 Runtime / Engine / event bus 的分层

v3 把运行现场和控制循环拆开：`Pico` 持有 workspace、session、memory、tools、workers、permissions、checkpoint 等；`Engine` 推进一次 turn。模块职责表见 `/Users/leonard/codeRepo/pico-v3/pico/release/v3/learning/09-module-map.md:20-30`。

SessionStore 仍以 `<id>.json` 保存 session，但比原始版本加强了线程锁和原子替换：先写同目录临时文件，再 `os.replace()`；实现见 `/Users/leonard/codeRepo/pico-v3/pico/pico/core/session_store.py:12-41`。session JSON 可包含 history、memory、checkpoints、runtime mode、workers、todos、workspace root 等，release 文档明确列出这些边界（`/Users/leonard/codeRepo/pico-v3/pico/release/v3/learning/08-session-run-evaluation.md:7-21`）。

v3 新增 `SessionEventBus`，把会话级 timeline 追加到 `<id>.events.jsonl`，每条事件带 event/session_id/created_at 并做 redaction（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/session_events.py:13-29`）。engine 在 turn 开始先建 run 和发 `turn_started`，随后把 user message 记录到 session，再发 `user_message`/`run_started`（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/engine.py:73-115`）。`Pico.record()` 仍然是每条 history item 写回 session（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/runtime.py:620-623`）。

单次 run 仍然由 RunStore 分开管理，但 v3 多了长工具输出 `artifacts/`；task state 在运行中不断覆盖，trace 是 append-only，report 是结束摘要（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/run_store.py:1-61`、`/Users/leonard/codeRepo/pico-v3/pico/pico/core/run_store.py:63-112`）。engine 每轮 prompt 前写 task state，工具执行后也写，终态时再统一写 checkpoint、trace、report，见 `/Users/leonard/codeRepo/pico-v3/pico/pico/core/engine.py:123-225`、`/Users/leonard/codeRepo/pico-v3/pico/pico/core/engine_helpers.py:15-97`、`/Users/leonard/codeRepo/pico-v3/pico/pico/core/completion_governance.py:134-200`。

v3 的 trace 事件额外有稳定的 trace/turn/phase/span/parent_span/status/error_type 字段，`emit_trace()` 先追加 trace，再跑 artifact graph、verifier、reminder、evidence consumers，并把更新后的 task state 写回（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/runtime_events.py:28-47`、`/Users/leonard/codeRepo/pico-v3/pico/pico/core/runtime.py:675-697`）。这让审计和排障比原始 pico 更强，但它仍是证据层，不是可重放的事务日志。

### 2.2 checkpoint 和 session resume 仍是同一类语义

v3 的 `RuntimeCheckpointsMixin` 基本保留原始实现：checkpoint schema 仍是 `phase1-v1`，字段仍是 goal/completed/blocker/next/key files/freshness/runtime identity，并在保存 session 后返回 checkpoint（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/runtime_checkpoints.py:9-74`）。`Pico.evaluate_resume_state()` 仍按 schema、key-file freshness、runtime identity 判定 `partial-stale`、`workspace-mismatch`、`full-valid` 等（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/runtime.py:300-369`）。

启动参数 `--resume latest` 仍是从 session JSON 装配新的 `Pico`（`/Users/leonard/codeRepo/pico-v3/pico/pico/cli.py:153-230`）；REPL 的 `/resume` 调用 `resume_runtime_session()`。该函数会加载目标 session、重新绑定 event bus/memory/todo/worker/runtime profile，并把 `current_turn_id/current_run_id/current_run_dir/current_task_state` 清空（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/session_lifecycle.py:14-18`、`/Users/leonard/codeRepo/pico-v3/pico/pico/core/session_lifecycle.py:34-67`）。因此它明确是“用旧 session 开始下一轮”，不是把旧 run 的 provider/tool 协程恢复起来。

checkpoint 在下一次 prompt 中仍被渲染为 goal/blocker/next/key files；v3 的学习文档把 session、run、task state 的分工说得很清楚：session 负责下次怎么继续，run 负责这次发生了什么（`/Users/leonard/codeRepo/pico-v3/pico/release/v3/learning/08-session-run-evaluation.md:148-185`）。

### 2.3 tool call 中断和副作用处理

v3 的工具边界更完整：先做 registry/schema/path 校验、permission、policy 和重复调用检查，再对 risky tool 做 before/after workspace snapshot，记录 `tool_status`、`tool_error_code`、`affected_paths`、`workspace_changed`、`diff_summary`。普通异常会被捕获并返回错误文本；shell 非零且 workspace 有变化会标为 `partial_success`。实现见 `/Users/leonard/codeRepo/pico-v3/pico/pico/core/tool_executor.py:10-122`。

工具调用前先写 session event `tool_started`，工具函数返回后才写 `tool_finished`、history、run trace 和 checkpoint；顺序见 `/Users/leonard/codeRepo/pico-v3/pico/pico/core/engine_helpers.py:15-97`。所以在硬崩溃场景中，`tool_started` 可以是最后一个 durable signal，但没有 `tool_finished` 并不等价于“工具没执行”：工具可能已经产生了外部副作用。v3 没有 operation id、工具 commit marker、幂等键、undo log 或启动时 unmatched-tool reconciliation，也不会自动重跑风险工具。

v3 增加的是“协作式中断”，不是 crash recovery：`abort_current_turn()` 设置 `abort_requested` 并尝试调用 provider 的 `abort()`（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/runtime.py:783-793`）；Engine 在 provider 前后、工具批次后检查标志，并通过 `finish_stopped_run()` 写 stop、checkpoint、trace、report（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/engine.py:123-134`、`/Users/leonard/codeRepo/pico-v3/pico/pico/core/engine.py:247-373`、`/Users/leonard/codeRepo/pico-v3/pico/pico/core/completion_governance.py:88-131`）。provider 的普通错误则会重试特定的空响应，或写 `model_error`、checkpoint 和 report（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/model_errors.py:8-84`）。

子 agent/worker 的 session item 会落盘，但正在运行的 thread、child runtime 和 `_tasks` 映射是进程内对象；v3 在 resume/clear 时会先 shutdown 当前 worker（`/Users/leonard/codeRepo/pico-v3/pico/pico/core/worker_manager.py:26-36`、`/Users/leonard/codeRepo/pico-v3/pico/pico/core/session_lifecycle.py:69-73`）。因此不能宣称进程重启后能无损续接一个正在执行的 worker。

### 2.4 v3 自己承认的恢复边界

这不是推测，而是仓库 release 文档的明确口径：

- session JSON 使用临时文件 + `os.replace()`，但文档紧接着说 v3 的中途持久化还没有 Claude Code transcript-first 那么细（`/Users/leonard/codeRepo/pico-v3/pico/release/v3/learning/08-session-run-evaluation.md:7-21`）。
- 当前取舍明确说下一步应把 run artifacts 与 session transcript 的恢复语义再打通，让中断恢复更稳（`/Users/leonard/codeRepo/pico-v3/pico/release/v3/learning/08-session-run-evaluation.md:127-131`）。
- 失败模式表把“进程死在 provider 请求中”列为当前只有“session/run 部分落盘”，改进方向是“用户消息进入模型前先写 transcript”；同时建议 report-from-trace 校验、failure taxonomy、细化 runtime signature（`/Users/leonard/codeRepo/pico-v3/pico/release/v3/learning/08-session-run-evaluation.md:247-265`）。
- runtime 学习文档说明成熟 query loop 还会有 streaming partial、tool-use block、tool result budget 和 watchdog，而 v3 当前是 blocking 的完整文本调用，对长输出、部分失败和 provider stall 仍不够强（`/Users/leonard/codeRepo/pico-v3/pico/release/v3/learning/02-runtime-engine.md:73-97`、`/Users/leonard/codeRepo/pico-v3/pico/release/v3/learning/02-runtime-engine.md:174-205`）。

### 2.5 v3 关键测试、验收和指标

- `test_v3_runtime.py` 证明 session event timeline 会落盘，并检查 `session_started -> turn_started -> user_message -> model_requested -> model_parsed -> assistant_message -> turn_finished`；工具场景检查 `tool_started`、`tool_finished`、status 和 workspace_changed，见 `/Users/leonard/codeRepo/pico-v3/pico/tests/test_v3_runtime.py:34-81`。
- v3 继承并扩展 checkpoint 测试：context reduction 的 checkpoint/artifact 关系、prompt 使用 checkpoint、stale summary、partial success、workspace mismatch、schema mismatch、no checkpoint、freshness mismatch、runtime identity mismatch，见 `/Users/leonard/codeRepo/pico-v3/pico/tests/test_pico.py:1117-1162`、`/Users/leonard/codeRepo/pico-v3/pico/tests/test_pico.py:1165-1247`、`/Users/leonard/codeRepo/pico-v3/pico/tests/test_pico.py:1250-1422`、`/Users/leonard/codeRepo/pico-v3/pico/tests/test_pico.py:1450-1522`。
- `test_run_store.py` 同样验证了只有 trace/state、尚未生成 report 的中间态是可读的（`/Users/leonard/codeRepo/pico-v3/pico/tests/test_run_store.py:58-66`）。
- 真实 session acceptance 检查进程级 CLI 场景的 session event、trace、report、resume continuation 和 provider error；resume 只断言 `full-valid` 或 `workspace-mismatch`，provider error 断言 `model_error`，见 `/Users/leonard/codeRepo/pico-v3/pico/tests/test_real_session_acceptance.py:14-58`、`/Users/leonard/codeRepo/pico-v3/pico/tests/test_real_session_acceptance.py:79-108`。
- v3 的 recovery ablation 从原始 checkpoint/stale/mismatch/schema/partial-success 扩到 `memory_continuity_fact_todo`，共 11 类；评估 `resume_success_rate`、`stale_reanchor_rate`、`workspace_drift_detection_rate`、`resume_false_accept_rate`、`resumption_success_rate`、`first_action_correctness`、`todo_continuity_rate`，见 `/Users/leonard/codeRepo/pico-v3/pico/pico/evaluation/metrics.py:1465-1532`、`/Users/leonard/codeRepo/pico-v3/pico/pico/evaluation/metrics.py:1700-1828`。测试要求 resume-enabled 的连续性指标至少达到 0.8（`/Users/leonard/codeRepo/pico-v3/pico/tests/test_metrics.py:149-173`）。这些是 scripted/synthetic runtime harness 指标，不是 kill -9 后 exactly-once 的证明。

本次本地验证（`UV_CACHE_DIR=/private/tmp/pico-v3-uv-cache uv run --no-sync pytest ...`，覆盖 run store、v3 runtime、pico checkpoint/resume、recovery metrics）结果为 `20 passed, 75 deselected`；有 pytest cache 权限 warning 和 `datetime.utcnow()` deprecation warning，没有测试失败。

实际执行 v3 recovery ablation（增加 memory/todo continuity，共 11 个 scripted 场景，1 次重复）的结果是：`resume_success_rate=0.9`、`stale_reanchor_rate=1.0`、`workspace_drift_detection_rate=1.0`、`resume_false_accept_rate=0.0`、`resumption_success_rate=1.0`、`first_action_correctness=1.0`、`todo_continuity_rate=1.0`。关闭恢复后这些成功/连续性指标均为 0。它证明的是 fixture 中的分类与首动作合同，仍不等于操作系统级崩溃测试。

## 3. pico 与 pico-v3 的差异

| 维度 | 原始 pico | pico-v3 | 面试应如何表述 |
|---|---|---|---|
| 运行结构 | 一个 `Pico` 里同时放状态和 ask loop | `Pico`/Runtime 持状态，`Engine` 推进 turn | v3 的职责边界更清楚 |
| session 写入 | JSON 直接覆盖；history item 触发 save | 临时文件 + `os.replace()`，带锁；history 仍逐条 save | v3 降低半截 JSON 风险，但不是 WAL |
| session event | 没有独立 session event bus | `<id>.events.jsonl`，session/turn/model/tool/permission/worker timeline | v3 可审计性更强 |
| run 证据 | task_state/trace/report | 同样三件套 + artifacts、span、consumer-derived evidence | v3 更适合排障和评测 |
| checkpoint | `phase1-v1` goal/blocker/next/freshness/identity | 基本同一套 mixin/语义 | v3 不是恢复模型的根本升级 |
| 普通工具异常 | catch 后返回 error，after diff，partial-success | 同样，但多了 schema/permission/policy、tool event、artifact metadata | “保守告诉模型检查 diff” |
| 用户主动中断 | 主要依赖 CLI/异常边界，没有 turn abort API | `abort_requested` + provider abort + stopped checkpoint/report | v3 有协作式 stop，不等于崩溃恢复 |
| worker | 同步 read-only `delegate` | 可持续的 Explore/Worker，但 child thread/runtime 仍是进程内 | 进程重启不能声称 worker 无损续接 |
| 测试证据 | 单元/固定模型 recovery ablation | 继承这些，再加 event timeline、真实 CLI acceptance、memory/todo continuity | 证明 runtime 合同，不证明 kill -9 |

## 4. 面试建议：怎样讲得准确且有亮点

### 4.1 60 秒版本

> 我把 agent 的恢复拆成 session continuity 和 run evidence 两层。每次用户请求先把 user message 和 session history 落盘；一次 run 维护 `task_state`、append-only trace 和最终 report。模型每执行完工具、触发上下文压缩或结束 turn，就建立一个 checkpoint，里面不是 Python 内存快照，而是当前目标、已完成项、阻塞原因、下一步、关键文件 freshness 和 runtime identity。重启用 `--resume latest` 加载 session，重新构造 runtime，校验 checkpoint 的 schema、关键文件 hash 和模型/工具/工作区身份，再把 checkpoint 摘要放入下一次 prompt。风险工具执行前后做工作区快照，返回后把 changed paths、diff 和 `partial_success` 写进证据；所以工具失败不会静默吞掉，模型会被要求先检查现场再决定是否 retry。这里的边界是：我恢复的是最后一个 durable checkpoint，不是恢复原来的协程；如果进程死在 provider 或 tool 中间，我会把动作标成未决/需要人工或模型检查，不能安全地自动重放风险工具。

如果讲 v3，再补一句：

> v3 把 Runtime 和 Engine 分开，并增加 session event bus、tool started/finished、permission/policy、span 和 evidence consumers；还支持协作式 abort、provider error 收口和真实 CLI acceptance。但仓库文档也明确承认 transcript-first 和 in-flight recovery 仍是后续增强方向。

### 4.2 可讲亮点

1. **恢复状态不是一句“把历史读回来”**：checkpoint 把目标、阻塞和下一步结构化，文件 freshness 与 runtime identity 防止拿过期上下文继续写。
2. **session 与 run 分层**：session 解决下一轮怎么继续；run 的 task state/trace/report 解决这次发生了什么。v3 还把 session event timeline 独立出来，方便排障和 `/session`/`tail -f`。
3. **副作用是可观察的**：风险工具前后 snapshot，报告 changed paths、diff summary、tool status/error code；shell “退出失败但已经改文件”不是简单 failed，而是 `partial_success`。
4. **状态机比裸 while 更能解释失败**：`attempts`（模型调用轮次）、`tool_steps`（实际工具动作数）、`status`、`stop_reason` 分开，便于预算控制和分类评测。
5. **可验证而不是只展示 demo**：固定模型/fixture 的 recovery ablation、运行证据测试和 v3 真实 CLI acceptance 都能给出可复现的 runtime 合同。

### 4.3 不能夸大的边界

- 不能说已经支持 kill -9 后从 tool call 中点自动续跑；没有 in-flight tool reconciliation。
- 不能说工具 exactly-once、事务回滚或外部 API 幂等；工具写文件和 shell 都可能有真实副作用。
- 不能把 checkpoint 说成完整 snapshot；它是结构化导航摘要 + freshness/identity 校验。
- 不能把 `trace.jsonl`/`events.jsonl` 说成已经有 WAL、fsync、checksum 或坏尾行修复；当前是 append 写入。
- 不能把 “report 缺失但 trace 存在”说成恢复成功；它只说明中间证据可能已落盘。
- 不能把 `partial_success` 说成自动修复；它只让模型/人知道“先检查 diff，再决定 retry”。
- 不能把 scripted recovery ablation 的 `resume_success_rate` 当作真实生产崩溃恢复率；测试没有证明 OS kill、磁盘断电或 provider 请求重复语义。
- 不能说 v3 的 worker 可跨进程恢复；session 中持久化的是 worker metadata，child runtime/thread/active task map 仍在内存。

## 5. Tau 现在已经做到了什么

Tau 当前的强项与 Pico 不一样：它已经有 append-only 的 session entry tree，而不是反复覆盖一个 session JSON。

- `JsonlSessionStorage` 逐 entry 追加 JSONL（`/Users/leonard/codeRepo/tau/src/tau_agent/session/storage.py:24-42`）；`MessageEntry` 保存 canonical message，`LeafEntry` 是 active branch 指针，`CustomEntry` 给上层扩展保留命名空间（`/Users/leonard/codeRepo/tau/src/tau_agent/session/entries.py:35-103`）。
- `CodingSession.load()` 读取 entries，选择最新 leaf，按 root-to-leaf 路径 reducer replay，然后用恢复出的 messages 创建新的 `AgentHarness`（`/Users/leonard/codeRepo/tau/src/tau_coding/session.py:297-389`）。它还保留分支、compaction、模型/思考级别变更；这一点比 Pico 的单 JSON session 更适合解释“可重放状态”。
- 如果持久化 transcript 中出现 assistant tool call 但没有对应 tool result，加载时会补一个 synthetic error `Tool call interrupted by user`，把修复后的 suffix 和新 leaf 再落盘，使 provider 能接受这段历史（`/Users/leonard/codeRepo/tau/src/tau_coding/session.py:1728-1763`、`/Users/leonard/codeRepo/tau/src/tau_coding/session.py:2616-2653`）。

但当前只能准确称为**对话树恢复 + 已持久化悬空 tool call 修复**，还不能称为完整任务崩溃恢复，原因有三个：

1. **assistant/tool intent 有持久化窗口。** `tau_agent` 在 assistant 的 `MessageEndEvent` 已经向外 yield 后，才把 assistant 加进 harness messages（`/Users/leonard/codeRepo/tau/src/tau_agent/loop.py:110-136`）；`CodingSession` 在收到该事件时调用 `_persist_messages_since()`，此刻通常还看不到这条 assistant（`/Users/leonard/codeRepo/tau/src/tau_coding/session.py:1577-1606`）。最终 sweep 会补写，但若硬崩溃发生在 assistant 已展示、工具正在执行的窗口，tool-call intent 可能根本没有 durable，因此上面的悬空调用修复也发现不了它。
2. **JSONL 不是完整 WAL。** 每条 message 和 leaf 分两次 append，未组成一个事务；append 没有显式 flush/fsync，最后一行被截断会让整个读取报错，而不是只丢弃未提交尾部。实现边界见 `/Users/leonard/codeRepo/tau/src/tau_agent/session/storage.py:30-42`。
3. **没有语义 checkpoint/trust evaluation。** Tau 会重建 messages，但目前没有 Pico 式 goal/blocker/next step、关键文件 freshness、Git/workspace/runtime identity，也没有对“工具可能已产生副作用但 completion 未落盘”的 `unknown_outcome` 状态。

因此如果今天去面试，当前口径应该是：

> Tau 用 append-only session tree 和 active leaf 做 canonical transcript replay，支持分支/压缩，并会持久化修复已经记录但未完成的 tool call。它恢复的是 durable conversation state，不是 Python 协程或工具 exactly-once；硬崩溃时尚有 assistant/tool intent 与 JSONL 尾部窗口，语义 checkpoint 和副作用 reconciliation 是下一阶段。

## 6. Tau 面试版恢复协议应该怎样设计

### 6.1 模块边界

把 coding-specific 恢复做成 `tau_coding` 中的深模块，例如 `TaskRecovery`，不要让 `tau_agent` 知道 Git、workspace、approval 或本地文件 freshness。外部只依赖一个窄接口：

```python
plan = recovery.prepare(session_state, environment)

# RecoveryPlan
# status: valid | stale_files | environment_mismatch |
#         schema_mismatch | unknown_tool_outcome | no_checkpoint
# action: continue | reanchor | confirm | discard
# checkpoint: goal / completed / blocker / next_step / key_files
# reasons: 可展示、可评测的判定证据
```

checkpoint 可先复用现有 `CustomEntry(namespace="tau_coding.recovery", data=...)` 落入 session tree。这样 `tau_agent` 继续只负责可移植的消息、事件和重放机制；`tau_coding` 独占 workspace trust、prompt projection 与恢复策略。接口本身就是主要测试面，内部的 hash、Git、文件系统适配器可用 fake 替换。

runtime identity 不应机械照搬 Pico。建议把 `resolved cwd/repository identity`、Git HEAD + dirty diff digest、关键文件 content hash、tool schema hash、sandbox/approval/write scope 作为会影响信任的字段；provider/model/thinking level 作为 advisory 或单独兼容策略，避免仅换模型就无谓地否定任务 checkpoint。

### 6.2 实施顺序

1. **先修 durable boundary。** 在向 UI yield 确认完成事件前，让 assistant message 进入可持久化状态；在任何 tool 执行前必须先落 `ToolIntentEntry(prepared, operation_id, args_hash, pre_fingerprint, risk)`，返回后落 `ToolOutcomeEntry(committed, result_ref, affected_paths, post_fingerprint)`。
2. **处理 unknown outcome。** 启动扫描发现 `prepared` 没有 `committed` 时标为 `unknown_tool_outcome`。明确只读且幂等的工具可按策略重试；文件 patch、shell、网络写等 mutation 不自动重试，先重新扫描现场并请求确认。
3. **让 JSONL 尾部可恢复。** 以 leaf/commit entry 为提交标记；允许隔离或丢弃唯一的坏尾行，但中间损坏仍报硬错误；把 message + leaf 做成逻辑 batch，并提供可配置 flush/fsync durable mode。必要时从完整 entry 重建索引/leaf。
4. **加入语义 checkpoint。** 在 committed tool result、context compaction、controlled abort、provider error 和 final 等安全边界写 checkpoint；resume 时用 freshness/runtime identity 得出 `RecoveryPlan`，TUI 明确展示是直接继续、重新读取、确认未知副作用，还是丢弃旧状态。
5. **run evidence 与恢复真相分离。** 用现有 agent event stream 接一个 `CodingRunRecorder`，保存 versioned run event log，并由 reducer 物化 status/report。trace 用于解释“发生了什么”，checkpoint/commit journal 决定“能否继续”，不要互相冒充。

### 6.3 必须有的故障注入与指标

至少用真实子进程 failpoint/`os._exit()` 覆盖：assistant 流式中断、assistant/tool intent 提交后、只读工具中、写工具产生副作用后但 outcome 前、MessageEntry 与 LeafEntry 之间、JSONL 坏尾行、关键文件 drift、Git/runtime policy drift、tool schema mismatch、checkpoint schema mismatch。

建议同时跑 recovery-enabled/disabled ablation，并报告：

- `committed_message_recovery_rate`
- `workspace_drift_detection_rate`
- `stale_state_false_accept_rate`（最关键）
- `interrupted_tool_detection_rate`
- `unknown_mutating_tool_auto_retry_rate`（目标为 0）
- `duplicate_side_effect_rate`（目标为 0）
- `first_action_correctness`
- resume prompt token cost / full-history replay token cost

这些测试通过后，Tau 才适合把口径升级成“有恢复协议”，但仍不能对不可事务化的外部 API 承诺 exactly-once。

## 7. Tau 的 90 秒面试话术

> Tau 最初已经有 append-only session tree：message 是事实记录，leaf 是 active branch 指针，重启后沿 root-to-leaf replay，分支和 compaction 也能保留；已落盘但缺 tool result 的调用会补 interrupted result，让 provider-safe。做故障分析后我发现，这只能恢复对话，不等于恢复正在执行的任务：assistant 的 tool intent 存在一个 append-lag，写工具崩在中间时也无法判断副作用是否发生。于是我把恢复协议放在 `tau_coding` 边界：工具执行前后分别提交 intent/outcome，安全边界写最小语义 checkpoint，里面是 goal、blocker、next step、key files；启动时再用 Git/workspace、文件 freshness、tool schema 和权限策略做 trust evaluation。恢复结果不是一个布尔值，而是 continue、reanchor、confirm 或 discard；未完成的 mutating tool 一律视为 unknown outcome，绝不自动重跑。运行 trace 只负责审计，checkpoint 和 commit journal 才是恢复真相。最后我用真实子进程 kill-point 测试 drift detection、false accept、重复副作用和 first-action correctness。边界也很明确：这是恢复到最后 durable commit，不是恢复 Python 栈；外部工具没有幂等键时不承诺 exactly-once。
