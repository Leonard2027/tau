# Refines the course mission toward local coding-agent recoverability

The learner explicitly refined the desired emphasis after Card 13 initially moved too quickly into remote-client disconnect and replay. The primary learning target is Tau as a local coding agent: recovery after cancellation/crash, log-based observability, and sessions that are resumable, branchable, and compactable. Distributed transport, WebSocket/SSE replay, and multi-tenant platform design should appear only as clearly labeled optional interview extrapolations.

The learner also requested that harness event propagation not be artificially separated from its actual local consumers. Future lessons must first trace the concrete product path — provider → loop → harness → ExtensionRuntime and CodingSession → TUI/CLI — before introducing a generalized platform boundary.

Implications: Card 13 has been rewritten around Tau's real local event path. Card 14 should deeply trace session entries, storage, tree/path replay, branches, compaction, and recovery. Card 15 should prefer a local recovery or log-observability secondary-development exercise. Card 16 should begin with a durable local coding-agent design and only then optionally scale the seams outward.
