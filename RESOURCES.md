# Tau Architecture Resources

## Knowledge

- [Tau README](README.md)
  The project's primary architectural overview and public behavior description. Use for: the three-layer model, core boundaries, capabilities, and local development commands.
- [Tau contributing guide](CONTRIBUTING.md)
  The repository's explicit design constraints and dependency rules. Use for: deciding which layer should own a change.
- [Tau project configuration](pyproject.toml)
  Authoritative source for Python version, dependencies, console entry point, and development tools. Use for: confirming runtime and tooling facts.
- [Typer documentation](https://typer.tiangolo.com/)
  Official documentation for Tau's type-annotated CLI framework. Use for: understanding how `tau_coding.cli:app` becomes the `tau` command.
- [AnyIO documentation](https://anyio.readthedocs.io/en/stable/)
  Official documentation for the asynchronous concurrency layer used to enter Tau's async runtime. Use for: understanding `anyio.run(...)`, cancellation, and bounded memory object streams for explicit backpressure.
- [PEP 525 — Asynchronous Generators](https://peps.python.org/pep-0525/)
  Python's authoritative async-generator specification. Use for: explaining Tau's pull-based, consumer-driven event pipeline.
- [WHATWG Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
  The browser-platform specification for SSE reconnect behavior, event `id`, and `Last-Event-ID`. Use for: client resume and replay contracts.
- [RFC 6455 — The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
  The WebSocket standard. Use for: close/abnormal-disconnect semantics and explaining why replay must be added at the application layer.
- [Apache Kafka introduction](https://kafka.apache.org/intro)
  Official description of durable partition logs and retention. Use for: event-log durability and replay vocabulary.
- [KafkaConsumer API](https://kafka.apache.org/23/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html)
  Official consumer offset, committed-position, and `seek()` semantics. Use for: durable cursors and restart/replay design.
- [Card 13 primary-source findings](reference/card13-streaming-primary-sources.md)
  Curated mapping of Tau's real local provider → loop → harness event path, with remote transport material retained only as optional interview reference.
- [Card 14 durable-session findings](reference/card14-durable-session-log-primary-sources.md)
  Local-first source map for Tau's append-only JSONL log, replay recovery, parent-pointer branches, compaction, and the current no-`fsync` durability gap.
- [JSON Lines specification](https://jsonlines.org/)
  The format-level source for one JSON value per line and append-friendly structured logs.
- [SQLite Write-Ahead Logging](https://www.sqlite.org/wal.html)
  Primary source for commit-by-append, reopen-and-recover, and explicit durability levels.
- [SQLite Atomic Commit](https://www.sqlite.org/atomiccommit.html)
  Primary source for crash atomicity and why `fsync` defines power-loss durability.
- [Pro Git — Branches in a Nutshell](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell)
  Official Git book chapter for parent-linked history and branches as movable pointers; useful for understanding Tau's `parent_id` and `LeafEntry` model.
- [Claude compaction documentation](https://platform.claude.com/docs/en/build-with-claude/compaction)
  Official context-compaction mechanics and trade-offs; use as comparison material for Tau's local `CompactionEntry` replay semantics.
- [Anthropic context engineering for agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
  First-party guidance on context rot, compaction, structured external memory, and preserving decisions and implementation state.
- [Pydantic documentation](https://pydantic.dev/docs/validation/latest/get-started/)
  Official documentation for validation and serialization models. Use for: understanding Tau's typed, discriminated event and session records.
- [Textual documentation](https://textual.textualize.io/)
  Official documentation for the terminal UI framework. Deferred from the main Agent backend/platform track; use only if a later development task requires TUI behavior.

## Wisdom (Communities)

- [Tau GitHub repository](https://github.com/Leonard2027/tau)
  Use issues, pull requests, and commit discussions to compare the architectural model with real maintenance decisions.

## Gaps

- Streaming/backpressure sources are curated in `reference/card13-streaming-primary-sources.md`, and context-management/durable-session sources are curated in `reference/card14-durable-session-log-primary-sources.md`. Production multi-tenant agent-system design is intentionally deferred under `MISSION.md`; it is not a current learning-track gap.
- No community recommendation beyond the project repository has been selected yet; add one only if real-world contribution feedback becomes part of the mission.
