# Mission: Master Tau's agent runtime for platform development and interviews

## Why
Use Tau's real implementation to quickly build a practical mental model of a local coding-agent runtime, with primary emphasis on recovery, log-based observability, and resumable/branchable/compactable sessions, so the learner can make well-placed secondary-development changes and clearly explain agent backend/platform design choices in interviews.

## Success looks like
- Explain how messages, provider/tool protocols, events, `run_agent_loop`, and `AgentHarness` fit together.
- Trace a prompt through model streaming, tool execution, steering/follow-up queues, cancellation, and recovery.
- Given a small feature or bug, identify the correct extension seam and verify the prediction in source and tests.
- Complete one interview-relevant secondary-development exercise around local recovery or log observability without coupling the core to a vendor or UI.
- Trace and explain how Tau reconstructs, branches, and compacts durable local sessions while preserving auditability.
- Design and defend a durable local coding-agent runtime first, then optionally explain how its seams could scale into an Agent backend/platform.

## Constraints
- Use the repository's actual Python 3.12+ implementation; do not introduce unrelated Rust material.
- Use approximately 15-minute micro-lessons, each answering one narrow question.
- Follow a core-first path through `tau_agent`, with special emphasis on `run_agent_loop` and `AgentHarness`.
- After understanding the harness, prioritize local coding-agent recovery, event/log observability, and durable session mechanics before optional distributed-platform extrapolation.
- Prefer guided questions, prediction, retrieval, and small exercises over large code dumps.
- Assume backend/system development experience, but do not assume familiarity with this codebase.

## Out of scope
- Exhaustive reading of every Tau module.
- Concrete Anthropic, OpenAI, Google, or other `tau_ai` vendor adapters.
- Deep Textual/TUI rendering internals and broad CLI product-shell coverage.
- Provider-specific transport details that do not clarify the provider-neutral runtime design.
- Remote-client disconnect/replay and multi-tenant distributed transport as a primary learning track; these are optional interview extrapolations after the local runtime is understood.
