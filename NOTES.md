# Teaching notes

- Learner background: backend/system development.
- Target role: Agent backend/platform engineering.
- Primary goals: quickly understand the agent runtime architecture, gain enough seam-level knowledge for secondary development, and prepare clear interview explanations.
- Learner already understands introductory command entry and the three-layer overview; do not restart from basic CLI mapping.
- Preferred order: learn the portable `tau_agent` core through `run_agent_loop` and `AgentHarness`, then pivot directly into interview-oriented reliability, streaming, state, tool-safety, and system-design topics.
- Concrete `tau_ai` vendor adapters and deep Textual/TUI internals are deferred unless a later development task specifically requires them.
- Preferred pacing: approximately 15 minutes per micro-lesson; each lesson should answer one small question and be independently completable.
- Teaching style: guided discovery; ask for predictions before explanations; avoid large code dumps.
- When an abstract protocol feels abrupt, first trace one concrete object end to end across product assembly, harness/loop dispatch, execution, and message conversion; introduce optional metadata only afterward.
- Default lesson rhythm: 2–3 minutes prediction, 6–8 minutes focused source reading, 3–4 minutes exercise, 1–2 minutes recap.
- Every micro-lesson needs one comprehension check or small exercise.
- Do not record mastery from coverage alone; require an explicit retrieval answer or applied exercise.
- Secondary-development exercises should reuse existing protocols, hooks, event streams, or harness seams and should double as interview evidence.
- Verify the repository's actual language and structure before introducing language-specific material. Tau is a Python 3.12+ project.
- For Tau's local coding-agent architecture, trace the actual provider → loop → harness → CodingSession → TUI/CLI/extension path before introducing remote-client concepts. Never call an in-process consumer a network client; label WebSocket/SSE, disconnect, and replay explicitly as later platform-interview extrapolations rather than current behavior.
- Before teaching a multi-module mechanism, first provide a concise code map: the relevant files, each file's responsibility, exact functions/line ranges, and a recommended reading order. For local session durability, do this before asking the learner to reason about JSONL recovery, branching, or compaction.
