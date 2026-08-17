# Understands the loop / harness architectural split

The learner completed Card 10 (Loop vs Harness). They retrieved the central responsibility boundary accurately: `run_agent_loop` owns the algorithm for one run, while `AgentHarness` owns state and lifecycle across runs. They also identified that a slow listener delays production of the next event after correction on the exact propagation order: loop `yield` → harness `await _notify` → harness `yield` → downstream requests the next event.

During the interview exercise, the learner initially treated listeners as a deployment mechanism. This was corrected: listeners are in-process event consumers, while deployment reuse means embedding the same provider/tool loop inside different hosts such as a CLI, WebSocket worker, background job consumer, or test driver. They also learned that event observability assists testing, but the deeper testability benefit comes from injectable provider/tools/messages and isolation from long-lived host state.

Implications: the learner can now state the mechanism/policy and single-run/cross-run boundary. In later streaming and platform-design lessons, revisit the distinction between an in-process subscriber and a distributed event transport, and require an applied design answer before claiming mastery of distributed deployment concerns.
