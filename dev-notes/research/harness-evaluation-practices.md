---
title: "Harness 评测：scripted model、回归测试与对外 benchmark 的边界"
---

# Harness 评测：scripted model、回归测试与对外 benchmark 的边界

**结论。** 用 scripted/fake provider（或 canned model response）驱动 agent
harness 是常见且正当的工程做法；它的目的，是把模型随机性和网络依赖拿掉，稳定地验证
loop、工具协议、错误处理和状态恢复。它本身**不能**证明真实模型的 agent 成功率，更不能
作为对外的模型/agent benchmark 成绩。最准确的命名通常是“确定性 harness 回归套件”或
“harness conformance suite”；其中单条按实际隔离范围可同时是 unit test 或
component/integration test，整套按用途是 regression suite。

本结论只使用官方文档、官方仓库/参考文档；记录日期为 2026-08-14。

## 这个 Pico 案例到底测了什么

根据交接材料 [Pico evaluation handoff](/Users/leonard/.claude/jobs/694e3cdf/tmp/pico-evaluation-handoff.md)，
固定的 12 条任务由 `ScriptedModelClient` 给出预定模型输出，验收是
`within_budget AND verifier_passed AND expected_artifact_exists AND
stop_reason == final_answer_returned`。它们刻意覆盖非法 patch 参数、路径逃逸、重复读取、
context reduction、resume freshness/workspace mismatch、durable-memory promotion 等。

因此这 12 条的被测对象是：**当既定模型轨迹到达 harness 时，harness 是否正确执行或拒绝
动作、继续状态机并留下可验证工件**。预定轨迹已经替模型作出了“下一步调用什么工具”的决定，
所以它们不能测量模型的任务理解、工具选择、规划、修复质量或在新任务上的泛化。尤其要注意，
scripted client 在错误后吐出下一条预定动作，只能证明 runtime 没有中断；除非测试还断言下一轮
prompt 或 trace 确实含有错误信息，否则不能据此证明错误已正确回灌，更不能证明模型理解错误后
完成了恢复。

这种拆分不是特例：OpenAI Agents SDK 的官方测试目录包含一个按 turn 排队输出和异常的
`FakeModel`；LangChain 的官方参考也直接提供 `FakeMessagesListChatModel`、
`FakeListChatModel` 和 `GenericFakeChatModel`，并将它们标为“for testing purposes”；
Inspect Evals 的官方贡献规范明确要求用 `mockllm/model` 模拟模型输出以保证确定性，且建议
unit test 不启动真实容器，而 mock sandbox 输出/exit code 来验证 solver logic。
[OpenAI Agents SDK `FakeModel`](https://github.com/openai/openai-agents-python/blob/main/tests/fake_model.py)；
[LangChain fake chat models](https://reference.langchain.com/python/langchain-core/language_models/fake_chat_models/)
；[Inspect Evals：mocking and sandboxes](https://ukgovernmentbeis.github.io/inspect_evals/contributing/index.html#mocking-and-sandboxes)。

## 分类：不是四选一，而是两条轴

“unit / integration / regression / benchmark”混合了**隔离范围**和**使用目的**，因此不应强行四选一。

| 问题 | 适用名称 | Pico fixed 12 的对应方式 |
| --- | --- | --- |
| 只测一个纯函数或一个 adapter，所有 I/O 都替身化？ | 单元测试 | 例如只测 tool-call gate 或 report 聚合。 |
| loop + scripted provider + 真实临时 fixture + 工具 adapter + verifier 一起跑？ | 组件/集成测试 | 大多数 fixed tasks 更接近这里；模型是 fake，但多个真实 harness 组件在交互。 |
| 将上述固定场景长期 pin 住、每次变更重复运行以发现行为倒退？ | 回归测试（用途） | 这是 12 条最恰当的主标签。 |
| 有版本化任务集、冻结协议/环境、真实系统输出、独立评分，并用于可比较的能力或成本测量？ | benchmark / evaluation（用途） | scripted 12 条单独不满足“真实系统输出”；最多称内部 **harness benchmark**，须明确其仅测机制合同。 |

建议 Tau 对其命名为：**deterministic harness regression suite**。若需要一个总称，可称
**harness conformance benchmark（内部）**，但所有文档和输出都应紧跟限定语“scripted model
trajectory；不代表 real-model task success”。不要把它简写为“Tau benchmark score”。

Inspect 的正式实践也把层次分开：它要求非平凡逻辑有 unit tests、相关 heavy/end-to-end tests
通过，并对少量真实样本手工运行/审阅 transcript；开发时先跑代表性小子集，再逐步扩大到完整
数据集。[Inspect Evals：testing standards 与 manual testing](https://ukgovernmentbeis.github.io/inspect_evals/contributing/index.html#testing-standards)。

## 对外声称效果时，应该如何测

应保留三层不能互相替代的证据：

1. **确定性机制回归（CI gate）。** 模型输出、外部 API 和必要的 sandbox 结果均可替身化；报告
   场景 ID、Tau commit、fixture hash、脚本版本、verifier、trace/report。可声称：
   “在版本 X 的 12 个 scripted harness 合同场景全部通过”。不可声称“模型能解决 12 个任务”。
2. **真实模型的端到端 eval。** 固定任务合同、workspace/容器、可用工具、预算、verifier；用真实
   provider 运行。报告模型和 provider 的精确 ID/版本、prompt/agent/harness commit、采样参数、
   工具权限、环境镜像/fixture 版本、任务 split、每任务结果和 logs/traces；随机模型应报告试次数、
   聚合方式与波动，而非只给最好一次。
3. **外部 benchmark。** 使用该 benchmark 的官方任务和官方 evaluator，不把自制 scripted
   场景混进分数。以 SWE-bench 为例，官方入口接收 patch predictions，支持 `--gold` 参考补丁，
   生成 Docker build/run logs 和最终结果；同一 `run_id`/instance 会复用缓存，重新比较不同 patch
   必须换 run ID。[SWE-bench 官方 README：evaluation harness](https://github.com/SWE-bench/SWE-bench/blob/main/README.md#-usage)。

真实模型 eval 也不应只看最终自然语言“完成了”。Anthropic 对 agent eval 的定义明确指出：评价
“agent”时评价的是 harness 与模型的组合；由于模型输出会变化，同一任务需要多次 trial，并应检查
最终环境 outcome 和完整 transcript，而不是只相信 agent 的完成声明。
[Anthropic：Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)。
Anthropic 的官方 tool-evaluation cookbook 实际
运行 agent loop 和工具，并按任务记录 expected/actual、exact score、时长、工具调用数，再汇总
accuracy、平均时长和工具调用数。
[Anthropic：Tool evaluation cookbook](https://platform.claude.com/cookbook/tool-evaluation-tool-evaluation)。
Anthropic 还要求 Skills 的上线评测覆盖应触发、不应触发和模糊边界，并跨所用模型测试；上线前要
检查 isolation、coexistence 与 regression。[Anthropic：Evaluating Skills before deployment](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/enterprise#evaluating-skills-before-deployment)。

OpenAI 对第三方评测的官方建议同样要求披露模型、reasoning setting、工具、harness、预算和有效性
检查，并说明被测系统与所作主张之间的对应关系。也就是说，harness 机制证据与真实系统效果证据
应分别留存，不能把一次 scripted replay 当作模型或 agent 能力证据。
[OpenAI：A shared playbook for trustworthy third-party evaluations](https://openai.com/index/trustworthy-third-party-evaluations-foundations/)。

## 对 Tau 的可执行表述

推荐在 `tau benchmark` 或报告中显式写入 `evaluation_mode`：

| `evaluation_mode` | 它回答的问题 | 可对外表述 |
| --- | --- | --- |
| `scripted_regression` | 给定轨迹时，runtime/工具/恢复/verifier 合同有没有退化？ | “12/12 harness regression scenarios passed”。 |
| `real_provider_acceptance` | 指定模型、权限、预算和任务合同下能否完成工作？ | “模型 M 在协议 P 的 N 次/任务上 verifier pass 为 …”；附运行工件。 |
| `external_benchmark` | 在外部冻结数据集和官方评分器下，系统表现如何？ | “按 benchmark B、split S、harness version H 的结果为 …”。 |

同时把 `task_id`、fixture/environment hash、tool policy、budget、model/provider、随机种子（若可用）、
harness commit、verifier 版本、trace/report 路径写入每条结果。这样 deterministic suite 能迅速定位
机制回归，而 real-provider 与外部 benchmark 的数字才有可解释、可复跑的比较基础。

## 一手来源索引

1. [Inspect Evals — Contributing: Testing standards, manual testing, mocking and sandboxes](https://ukgovernmentbeis.github.io/inspect_evals/contributing/index.html)（UK AI Security Institute 官方项目）。
2. [LangChain Core reference — fake chat models](https://reference.langchain.com/python/langchain-core/language_models/fake_chat_models/)（LangChain 官方 API 参考）。
3. [SWE-bench official repository README — evaluation harness usage](https://github.com/SWE-bench/SWE-bench/blob/main/README.md#-usage)（SWE-bench 官方仓库）。
4. [Anthropic — Tool evaluation cookbook](https://platform.claude.com/cookbook/tool-evaluation-tool-evaluation)；[Evaluating Skills before deployment](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/enterprise#evaluating-skills-before-deployment)（Anthropic 官方）。
5. [OpenAI Agents SDK — `FakeModel`](https://github.com/openai/openai-agents-python/blob/main/tests/fake_model.py)；[OpenAI — trustworthy third-party evaluations](https://openai.com/index/trustworthy-third-party-evaluations-foundations/)（OpenAI 官方）。
