# Understands Tau's local streaming event path

The learner completed the revised Card 13 after correcting the course's framing. They explicitly rejected treating Tau's in-process harness consumer as a network client and requested that the lesson stay grounded in the local coding-agent product path. The lesson was rewritten accordingly.

They correctly ordered the full path of a text delta without assistance: provider `TextDeltaEvent` → loop `MessageUpdateEvent` → ExtensionRuntime listener → harness main `yield` → CodingSession processing/re-yield → TUI adapter. They also correctly retrieved the backpressure consequence: an extension handler that takes five seconds delays TUI delivery by at least five seconds because `AgentHarness._run` awaits `_notify` before yielding to the main iterator.

Implications: the learner distinguishes the harness subscription path from the main iterator path while understanding that both are local parts of one product event chain. They are ready to move from transient runtime events to durable local session logs, recovery, branches, and compaction. Remote transport replay should remain optional interview extrapolation only.
